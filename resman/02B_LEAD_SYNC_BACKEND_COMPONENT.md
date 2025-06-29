# Lead Sync Backend Component: DomIQ ↔ ResMan

## Backend API Architecture for Lead Sync Integration

### Core Lead Sync API Endpoints

#### 1. Lead Creation API (Push to ResMan)
```typescript
// POST /api/resman/leads
interface CreateLeadRequest {
  conversation_id: string;
  lead_data: {
    first_name: string;
    last_name: string;
    email?: string;
    phone?: string;
    property_id: string;
    source: 'chatbot';
    initial_message: string;
    lead_score: number;
    move_in_date?: string;
    budget_range?: {
      min: number;
      max: number;
    };
    apartment_size_preference?: string;
    occupants_count?: number;
    has_pets?: boolean;
    pet_details?: PetDetails;
    desired_features?: string[];
    work_location?: string;
    reason_for_moving?: string;
  };
}

interface CreateLeadResponse {
  success: boolean;
  resman_lead_id: string;
  sync_status: string;
  created_at: string;
  sync_log_id: string;
}

// Implementation
export async function POST(request: Request) {
  try {
    const { conversation_id, lead_data } = await request.json();
    
    // Validate conversation exists
    const conversation = await getConversationById(conversation_id);
    if (!conversation) {
      return new Response(JSON.stringify({ error: 'Conversation not found' }), {
        status: 404,
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    // Transform DomIQ data to ResMan format
    const resmanLeadData = await transformToResManFormat(conversation, lead_data);
    
    // Push to ResMan
    const resmanResponse = await pushLeadToResMan(resmanLeadData);
    
    // Log sync operation
    const syncLog = await logSyncOperation({
      conversation_id,
      resman_lead_id: resmanResponse.lead_id,
      sync_direction: 'to_resman',
      sync_status: 'success',
      sync_type: 'create',
      sync_data: resmanLeadData,
      response_data: resmanResponse
    });
    
    // Update conversation with ResMan lead ID
    await updateConversationResManId(conversation_id, resmanResponse.lead_id);
    
    return new Response(JSON.stringify({
      success: true,
      resman_lead_id: resmanResponse.lead_id,
      sync_status: 'success',
      created_at: new Date().toISOString(),
      sync_log_id: syncLog.id
    }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });
    
  } catch (error) {
    console.error('Error creating lead in ResMan:', error);
    return new Response(JSON.stringify({ error: 'Failed to create lead' }), {
      status: 500,
      headers: { 'Content-Type': 'application/json' }
    });
  }
}
```

#### 2. Lead Pull API (Pull from ResMan)
```typescript
// GET /api/resman/leads/pull
interface PullLeadsRequest {
  property_id?: string;
  date_from?: string;
  date_to?: string;
  status_filter?: string[];
  limit?: number;
  include_conversations?: boolean;
  sync_type?: 'full' | 'incremental';
}

interface PullLeadsResponse {
  leads: Array<{
    resman_lead_id: string;
    first_name: string;
    last_name: string;
    email?: string;
    phone?: string;
    status: string;
    lead_score: number;
    last_activity: string;
    notes?: string;
    custom_fields?: Record<string, any>;
    property_id: string;
    created_at: string;
    updated_at: string;
  }>;
  total_count: number;
  sync_summary: {
    new_leads: number;
    updated_leads: number;
    conflicts_resolved: number;
    failed_syncs: number;
  };
  next_sync_token?: string;
}

// Implementation
export async function GET(request: Request) {
  try {
    const { searchParams } = new URL(request.url);
    const property_id = searchParams.get('property_id');
    const date_from = searchParams.get('date_from');
    const date_to = searchParams.get('date_to');
    const status_filter = searchParams.get('status_filter')?.split(',');
    const limit = parseInt(searchParams.get('limit') || '100');
    const include_conversations = searchParams.get('include_conversations') === 'true';
    const sync_type = searchParams.get('sync_type') || 'incremental';
    
    // Get ResMan leads
    const resmanLeads = await pullLeadsFromResMan({
      property_id,
      date_from,
      date_to,
      status_filter,
      limit,
      sync_type
    });
    
    // Process each lead
    const processedLeads = [];
    const syncSummary = {
      new_leads: 0,
      updated_leads: 0,
      conflicts_resolved: 0,
      failed_syncs: 0
    };
    
    for (const lead of resmanLeads) {
      try {
        const result = await processResManLead(lead, include_conversations);
        processedLeads.push(result.lead);
        
        if (result.action === 'created') syncSummary.new_leads++;
        else if (result.action === 'updated') syncSummary.updated_leads++;
        else if (result.action === 'conflict_resolved') syncSummary.conflicts_resolved++;
        
      } catch (error) {
        console.error(`Error processing lead ${lead.resman_lead_id}:`, error);
        syncSummary.failed_syncs++;
      }
    }
    
    return new Response(JSON.stringify({
      leads: processedLeads,
      total_count: processedLeads.length,
      sync_summary: syncSummary,
      next_sync_token: resmanLeads.next_sync_token
    }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });
    
  } catch (error) {
    console.error('Error pulling leads from ResMan:', error);
    return new Response(JSON.stringify({ error: 'Failed to pull leads' }), {
      status: 500,
      headers: { 'Content-Type': 'application/json' }
    });
  }
}
```

#### 3. Lead Status Update API (Bidirectional)
```typescript
// PUT /api/resman/leads/{lead_id}/status
interface UpdateLeadStatusRequest {
  status: 'new' | 'contacted' | 'qualified' | 'tour_scheduled' | 'application' | 'leased' | 'lost';
  notes?: string;
  next_follow_up?: string;
  updated_by: 'chatbot' | 'resman_user' | 'system';
  sync_direction: 'to_resman' | 'from_resman';
  custom_fields?: Record<string, any>;
}

interface UpdateLeadStatusResponse {
  success: boolean;
  new_status: string;
  updated_at: string;
  sync_conflicts?: Array<{
    field: string;
    resman_value: any;
    domiq_value: any;
    resolution: string;
  }>;
  conversation_updated: boolean;
}

// Implementation
export async function PUT(
  request: Request,
  { params }: { params: { lead_id: string } }
) {
  try {
    const { lead_id } = params;
    const { status, notes, next_follow_up, updated_by, sync_direction, custom_fields } = await request.json();
    
    // Validate lead exists in ResMan
    const resmanLead = await getResManLead(lead_id);
    if (!resmanLead) {
      return new Response(JSON.stringify({ error: 'Lead not found in ResMan' }), {
        status: 404,
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    // Get corresponding conversation
    const conversation = await getConversationByResManLeadId(lead_id);
    
    if (sync_direction === 'to_resman') {
      // Update ResMan lead status
      const resmanResponse = await updateResManLeadStatus(lead_id, {
        status,
        notes,
        next_follow_up,
        custom_fields
      });
      
      // Log sync operation
      await logSyncOperation({
        conversation_id: conversation?.id,
        resman_lead_id: lead_id,
        sync_direction: 'to_resman',
        sync_status: 'success',
        sync_type: 'status_update',
        sync_data: { status, notes, next_follow_up, custom_fields },
        response_data: resmanResponse
      });
      
    } else if (sync_direction === 'from_resman') {
      // Update DomIQ conversation status
      if (conversation) {
        const conflicts = await updateConversationFromResMan(conversation.id, {
          status,
          notes,
          next_follow_up,
          custom_fields
        });
        
        // Log sync operation
        await logSyncOperation({
          conversation_id: conversation.id,
          resman_lead_id: lead_id,
          sync_direction: 'from_resman',
          sync_status: conflicts.length > 0 ? 'conflict' : 'success',
          sync_type: 'status_update',
          sync_data: { status, notes, next_follow_up, custom_fields },
          conflict_details: conflicts.length > 0 ? { conflicts } : undefined
        });
        
        return new Response(JSON.stringify({
          success: true,
          new_status: status,
          updated_at: new Date().toISOString(),
          sync_conflicts: conflicts,
          conversation_updated: true
        }), {
          status: 200,
          headers: { 'Content-Type': 'application/json' }
        });
      }
    }
    
    return new Response(JSON.stringify({
      success: true,
      new_status: status,
      updated_at: new Date().toISOString(),
      conversation_updated: !!conversation
    }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });
    
  } catch (error) {
    console.error('Error updating lead status:', error);
    return new Response(JSON.stringify({ error: 'Failed to update lead status' }), {
      status: 500,
      headers: { 'Content-Type': 'application/json' }
    });
  }
}
```

#### 4. Lead Merge API (Conflict Resolution)
```typescript
// POST /api/resman/leads/merge
interface MergeLeadsRequest {
  resman_lead_id: string;
  domiq_conversation_id: string;
  merge_strategy: 'resman_wins' | 'domiq_wins' | 'manual_review';
  merge_fields: string[];
  conflict_resolution?: Record<string, any>;
  notes?: string;
}

interface MergeLeadsResponse {
  success: boolean;
  merged_lead_id: string;
  conflicts_resolved: number;
  merge_summary: {
    fields_merged: string[];
    fields_conflicted: string[];
    final_values: Record<string, any>;
  };
  sync_log_id: string;
}

// Implementation
export async function POST(request: Request) {
  try {
    const { resman_lead_id, domiq_conversation_id, merge_strategy, merge_fields, conflict_resolution, notes } = await request.json();
    
    // Get both lead records
    const resmanLead = await getResManLead(resman_lead_id);
    const conversation = await getConversationById(domiq_conversation_id);
    
    if (!resmanLead || !conversation) {
      return new Response(JSON.stringify({ error: 'Lead or conversation not found' }), {
        status: 404,
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    // Perform merge operation
    const mergeResult = await performLeadMerge({
      resmanLead,
      conversation,
      merge_strategy,
      merge_fields,
      conflict_resolution
    });
    
    // Log merge operation
    const syncLog = await logSyncOperation({
      conversation_id: conversation.id,
      resman_lead_id: resman_lead_id,
      sync_direction: 'bidirectional',
      sync_status: 'success',
      sync_type: 'merge',
      sync_data: {
        merge_strategy,
        merge_fields,
        conflict_resolution,
        notes
      },
      response_data: mergeResult
    });
    
    // Update conflict resolution status
    await updateConflictResolutionStatus(resman_lead_id, conversation.id, 'resolved');
    
    return new Response(JSON.stringify({
      success: true,
      merged_lead_id: mergeResult.merged_lead_id,
      conflicts_resolved: mergeResult.conflicts_resolved,
      merge_summary: mergeResult.merge_summary,
      sync_log_id: syncLog.id
    }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });
    
  } catch (error) {
    console.error('Error merging leads:', error);
    return new Response(JSON.stringify({ error: 'Failed to merge leads' }), {
      status: 500,
      headers: { 'Content-Type': 'application/json' }
    });
  }
}
```

#### 5. Webhook Receiver (ResMan → DomIQ)
```typescript
// POST /api/resman/webhooks/lead-update
interface LeadUpdateWebhook {
  lead_id: string;
  status: string;
  updated_fields: Record<string, any>;
  timestamp: string;
  updated_by: string;
  sync_direction: 'from_resman';
  webhook_signature?: string;
}

// Implementation
export async function POST(request: Request) {
  try {
    const webhookData: LeadUpdateWebhook = await request.json();
    
    // Validate webhook signature
    if (!validateWebhookSignature(request, webhookData)) {
      return new Response('Unauthorized', { status: 401 });
    }
    
    // Process lead update from ResMan
    const result = await processResManLeadUpdate(webhookData);
    
    // Log webhook processing
    await logSyncOperation({
      conversation_id: result.conversation_id,
      resman_lead_id: webhookData.lead_id,
      sync_direction: 'from_resman',
      sync_status: result.success ? 'success' : 'failed',
      sync_type: 'webhook_update',
      sync_data: webhookData,
      error_message: result.error
    });
    
    return new Response('OK', { status: 200 });
    
  } catch (error) {
    console.error('Error processing ResMan webhook:', error);
    return new Response('Internal Server Error', { status: 500 });
  }
}

// Webhook signature validation
function validateWebhookSignature(request: Request, webhookData: LeadUpdateWebhook): boolean {
  const signature = request.headers.get('x-resman-signature');
  const expectedSignature = generateWebhookSignature(webhookData);
  return signature === expectedSignature;
}
```

#### 6. Real-time Sync API (WebSocket)
```typescript
// WebSocket endpoint for real-time bidirectional sync
// WS /api/resman/sync/websocket

interface RealTimeSyncMessage {
  type: 'lead_update' | 'status_change' | 'new_lead' | 'sync_conflict' | 'heartbeat';
  data: any;
  timestamp: string;
  source: 'resman' | 'domiq';
  property_id?: string;
}

// WebSocket connection handler
export function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const property_id = searchParams.get('property_id');
  const auth_token = searchParams.get('token');
  
  // Validate authentication
  if (!validateWebSocketAuth(auth_token)) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  // Upgrade to WebSocket
  const { socket, response } = Deno.upgradeWebSocket(request);
  
  // Initialize real-time sync handler
  const syncHandler = new RealTimeLeadSync(socket, property_id);
  
  socket.onopen = () => {
    console.log('WebSocket connection established for real-time sync');
    syncHandler.onConnect();
  };
  
  socket.onmessage = (event) => {
    const message: RealTimeSyncMessage = JSON.parse(event.data);
    syncHandler.handleMessage(message);
  };
  
  socket.onclose = () => {
    console.log('WebSocket connection closed');
    syncHandler.onDisconnect();
  };
  
  socket.onerror = (error) => {
    console.error('WebSocket error:', error);
    syncHandler.onError(error);
  };
  
  return response;
}

// Real-time sync handler class
class RealTimeLeadSync {
  private socket: WebSocket;
  private property_id?: string;
  private resmanWebSocket?: WebSocket;
  private heartbeatInterval?: number;
  
  constructor(socket: WebSocket, property_id?: string) {
    this.socket = socket;
    this.property_id = property_id;
  }
  
  async onConnect() {
    // Connect to ResMan WebSocket if property specified
    if (this.property_id) {
      await this.connectToResMan();
    }
    
    // Start heartbeat
    this.startHeartbeat();
  }
  
  async handleMessage(message: RealTimeSyncMessage) {
    try {
      switch (message.type) {
        case 'lead_update':
          await this.handleLeadUpdate(message.data);
          break;
        case 'status_change':
          await this.handleStatusChange(message.data);
          break;
        case 'new_lead':
          await this.handleNewLead(message.data);
          break;
        case 'sync_conflict':
          await this.handleSyncConflict(message.data);
          break;
        case 'heartbeat':
          await this.handleHeartbeat(message.data);
          break;
        default:
          console.warn('Unknown message type:', message.type);
      }
    } catch (error) {
      console.error('Error handling real-time sync message:', error);
      this.sendError(error);
    }
  }
  
  private async handleLeadUpdate(data: any) {
    // Process lead update from DomIQ
    const result = await processDomIQLeadUpdate(data);
    
    // Forward to ResMan if connected
    if (this.resmanWebSocket) {
      this.resmanWebSocket.send(JSON.stringify({
        type: 'lead_update',
        data: result,
        timestamp: new Date().toISOString(),
        source: 'domiq'
      }));
    }
  }
  
  private async handleStatusChange(data: any) {
    // Process status change from DomIQ
    const result = await processDomIQStatusChange(data);
    
    // Forward to ResMan if connected
    if (this.resmanWebSocket) {
      this.resmanWebSocket.send(JSON.stringify({
        type: 'status_change',
        data: result,
        timestamp: new Date().toISOString(),
        source: 'domiq'
      }));
    }
  }
  
  private async handleNewLead(data: any) {
    // Process new lead from DomIQ
    const result = await processDomIQNewLead(data);
    
    // Forward to ResMan if connected
    if (this.resmanWebSocket) {
      this.resmanWebSocket.send(JSON.stringify({
        type: 'new_lead',
        data: result,
        timestamp: new Date().toISOString(),
        source: 'domiq'
      }));
    }
  }
  
  private async handleSyncConflict(data: any) {
    // Handle sync conflict
    const resolution = await resolveSyncConflict(data);
    
    // Notify both systems of resolution
    this.sendMessage({
      type: 'sync_conflict_resolved',
      data: resolution,
      timestamp: new Date().toISOString(),
      source: 'system'
    });
  }
  
  private async handleHeartbeat(data: any) {
    // Respond to heartbeat
    this.sendMessage({
      type: 'heartbeat',
      data: { status: 'alive' },
      timestamp: new Date().toISOString(),
      source: 'system'
    });
  }
  
  private async connectToResMan() {
    try {
      const resmanConfig = await getResManIntegrationConfig(this.property_id!);
      this.resmanWebSocket = new WebSocket(resmanConfig.websocket_url);
      
      this.resmanWebSocket.onmessage = (event) => {
        const resmanMessage = JSON.parse(event.data);
        this.handleResManMessage(resmanMessage);
      };
      
      this.resmanWebSocket.onerror = (error) => {
        console.error('ResMan WebSocket error:', error);
      };
      
    } catch (error) {
      console.error('Error connecting to ResMan WebSocket:', error);
    }
  }
  
  private async handleResManMessage(message: any) {
    // Process message from ResMan
    const result = await processResManMessage(message);
    
    // Forward to DomIQ client
    this.sendMessage({
      type: message.type,
      data: result,
      timestamp: new Date().toISOString(),
      source: 'resman'
    });
  }
  
  private sendMessage(message: RealTimeSyncMessage) {
    if (this.socket.readyState === WebSocket.OPEN) {
      this.socket.send(JSON.stringify(message));
    }
  }
  
  private sendError(error: any) {
    this.sendMessage({
      type: 'error',
      data: { message: error.message },
      timestamp: new Date().toISOString(),
      source: 'system'
    });
  }
  
  private startHeartbeat() {
    this.heartbeatInterval = setInterval(() => {
      this.sendMessage({
        type: 'heartbeat',
        data: { status: 'ping' },
        timestamp: new Date().toISOString(),
        source: 'system'
      });
    }, 30000); // 30 seconds
  }
  
  onDisconnect() {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
    }
    if (this.resmanWebSocket) {
      this.resmanWebSocket.close();
    }
  }
  
  onError(error: any) {
    console.error('Real-time sync error:', error);
  }
}
```

### Data Transformation Services

#### DomIQ → ResMan Mapper
```typescript
class DomIQToResManMapper {
  // Map DomIQ conversation data to ResMan lead format
  async mapConversationToLead(conversation: Conversation): Promise<ResManLead> {
    const user = await getUserById(conversation.user_id);
    
    return {
      first_name: user?.first_name || 'Anonymous',
      last_name: user?.last_name || 'User',
      email: user?.email,
      phone: user?.phone,
      property_id: conversation.property_id,
      source: 'chatbot',
      initial_message: conversation.ai_intent_summary || 'Chatbot interaction',
      lead_score: this.mapLeadScore(conversation.lead_score || 0),
      move_in_date: conversation.move_in_date,
      budget_range: {
        min: conversation.price_range_min || 0,
        max: conversation.price_range_max || 0
      },
      apartment_size_preference: conversation.apartment_size_preference,
      occupants_count: conversation.occupants_count,
      has_pets: conversation.has_pets,
      pet_details: conversation.pet_details,
      desired_features: conversation.desired_features,
      work_location: conversation.work_location,
      reason_for_moving: conversation.reason_for_moving,
      custom_fields: {
        domiq_conversation_id: conversation.id,
        chatbot_session_start: conversation.start_time,
        lead_qualification_status: conversation.is_qualified ? 'qualified' : 'unqualified',
        tour_booking_status: conversation.is_book_tour ? 'scheduled' : 'not_scheduled'
      }
    };
  }
  
  // Map DomIQ user data to ResMan contact format
  async mapUserToContact(user: User): Promise<ResManContact> {
    return {
      first_name: user.first_name || 'Anonymous',
      last_name: user.last_name || 'User',
      email: user.email,
      phone: user.phone,
      age: user.age,
      lead_source: user.lead_source || 'chatbot',
      custom_fields: {
        domiq_user_id: user.id,
        browser_id: user.browser_id,
        session_count: user.session_count,
        last_seen: user.last_seen
      }
    };
  }
  
  // Map DomIQ tour booking to ResMan appointment
  mapTourToAppointment(tour: TourBooking): ResManAppointment {
    return {
      prospect_name: tour.prospect_name,
      prospect_email: tour.prospect_email,
      prospect_phone: tour.prospect_phone,
      unit_interest: tour.unit_interest,
      tour_datetime: tour.tour_datetime,
      duration_minutes: tour.duration_minutes,
      tour_type: this.mapTourType(tour.tour_type),
      source: 'chatbot',
      notes: tour.notes,
      custom_fields: {
        domiq_tour_id: tour.id,
        conversation_id: tour.conversation_id
      }
    };
  }
  
  // Map DomIQ lead score to ResMan lead score
  mapLeadScore(domiqScore: number): number {
    // Convert DomIQ 0-100 score to ResMan 0-100 score
    return Math.min(Math.max(domiqScore, 0), 100);
  }
  
  // Map DomIQ tour type to ResMan tour type
  mapTourType(domiqType: string): string {
    const tourTypeMap: Record<string, string> = {
      'in_person': 'in_person',
      'virtual': 'virtual',
      'self_guided': 'self_guided'
    };
    return tourTypeMap[domiqType] || 'in_person';
  }
}
```

#### ResMan → DomIQ Mapper
```typescript
class ResManToDomIQMapper {
  // Map ResMan lead status to DomIQ conversation context
  mapLeadStatusToContext(status: string): ConversationContext {
    const statusContextMap: Record<string, ConversationContext> = {
      'new': { stage: 'initial_contact', priority: 'high' },
      'contacted': { stage: 'contacted', priority: 'high' },
      'qualified': { stage: 'qualified', priority: 'medium' },
      'tour_scheduled': { stage: 'tour_scheduled', priority: 'medium' },
      'application': { stage: 'application', priority: 'low' },
      'leased': { stage: 'leased', priority: 'low' },
      'lost': { stage: 'lost', priority: 'low' }
    };
    
    return statusContextMap[status] || { stage: 'unknown', priority: 'medium' };
  }
  
  // Map ResMan notes to DomIQ conversation updates
  mapNotesToUpdates(notes: string[]): ConversationUpdate[] {
    return notes.map(note => ({
      type: 'note',
      content: note,
      timestamp: new Date(),
      source: 'resman'
    }));
  }
  
  // Map ResMan lead data to DomIQ user profile
  mapLeadToUserProfile(lead: ResManLead): UserProfile {
    return {
      first_name: lead.first_name,
      last_name: lead.last_name,
      email: lead.email,
      phone: lead.phone,
      lead_source: lead.source || 'resman',
      custom_fields: {
        resman_lead_id: lead.id,
        resman_status: lead.status,
        resman_lead_score: lead.lead_score
      }
    };
  }
  
  // Map ResMan custom fields to DomIQ context
  mapCustomFieldsToContext(customFields: Record<string, any>): ConversationContext {
    return {
      stage: customFields.domiq_stage || 'unknown',
      priority: customFields.domiq_priority || 'medium',
      tags: customFields.domiq_tags || [],
      metadata: customFields
    };
  }
}
```

### Conflict Resolution Service

```typescript
class LeadSyncConflictResolver {
  async resolveConflict(conflict: LeadConflict): Promise<ConflictResolution> {
    const { field, resman_value, domiq_value, conflict_type } = conflict;
    
    switch (conflict_type) {
      case 'status_conflict':
        return this.resolveStatusConflict(resman_value, domiq_value);
      
      case 'data_conflict':
        return this.resolveDataConflict(field, resman_value, domiq_value);
      
      case 'duplicate_lead':
        return this.resolveDuplicateLead(resman_value, domiq_value);
      
      default:
        return this.escalateToManualReview(conflict);
    }
  }
  
  private async resolveStatusConflict(resmanStatus: string, domiqStatus: string): Promise<ConflictResolution> {
    // Business logic for status conflict resolution
    const statusPriority = {
      'leased': 1,
      'application': 2,
      'tour_scheduled': 3,
      'qualified': 4,
      'contacted': 5,
      'new': 6,
      'lost': 7
    };
    
    const resmanPriority = statusPriority[resmanStatus] || 999;
    const domiqPriority = statusPriority[domiqStatus] || 999;
    
    return {
      resolution: resmanPriority <= domiqPriority ? 'resman_wins' : 'domiq_wins',
      final_value: resmanPriority <= domiqPriority ? resmanStatus : domiqStatus,
      reason: 'Status priority resolution'
    };
  }
  
  private async resolveDataConflict(field: string, resmanValue: any, domiqValue: any): Promise<ConflictResolution> {
    // Field-specific conflict resolution rules
    const fieldRules: Record<string, (resman: any, domiq: any) => any> = {
      'email': (resman, domiq) => resman || domiq, // Prefer non-null email
      'phone': (resman, domiq) => resman || domiq, // Prefer non-null phone
      'lead_score': (resman, domiq) => Math.max(resman || 0, domiq || 0), // Take higher score
      'move_in_date': (resman, domiq) => resman || domiq, // Prefer non-null date
      'budget_range': (resman, domiq) => {
        // Merge budget ranges
        const resmanRange = resman || { min: 0, max: 0 };
        const domiqRange = domiq || { min: 0, max: 0 };
        return {
          min: Math.min(resmanRange.min, domiqRange.min),
          max: Math.max(resmanRange.max, domiqRange.max)
        };
      }
    };
    
    const resolver = fieldRules[field];
    if (resolver) {
      const finalValue = resolver(resmanValue, domiqValue);
      return {
        resolution: 'merged',
        final_value: finalValue,
        reason: `Field-specific resolution for ${field}`
      };
    }
    
    // Default: ResMan wins for most fields
    return {
      resolution: 'resman_wins',
      final_value: resmanValue,
      reason: 'Default ResMan priority'
    };
  }
  
  private async resolveDuplicateLead(resmanLead: ResManLead, domiqLead: DomIQLead): Promise<ConflictResolution> {
    // Check if leads are actually duplicates
    const similarityScore = this.calculateLeadSimilarity(resmanLead, domiqLead);
    
    if (similarityScore > 0.8) {
      // High similarity - merge leads
      const mergedLead = await this.mergeLeads(resmanLead, domiqLead);
      return {
        resolution: 'merged',
        final_value: mergedLead,
        reason: `High similarity score: ${similarityScore}`
      };
    } else {
      // Low similarity - escalate to manual review
      return this.escalateToManualReview({
        field: 'duplicate_lead',
        resman_value: resmanLead,
        domiq_value: domiqLead,
        conflict_type: 'duplicate_lead'
      });
    }
  }
  
  private calculateLeadSimilarity(resmanLead: ResManLead, domiqLead: DomIQLead): number {
    let score = 0;
    let totalFields = 0;
    
    // Compare name similarity
    if (resmanLead.first_name && domiqLead.first_name) {
      score += this.stringSimilarity(resmanLead.first_name, domiqLead.first_name);
      totalFields++;
    }
    
    if (resmanLead.last_name && domiqLead.last_name) {
      score += this.stringSimilarity(resmanLead.last_name, domiqLead.last_name);
      totalFields++;
    }
    
    // Compare email
    if (resmanLead.email && domiqLead.email) {
      score += resmanLead.email.toLowerCase() === domiqLead.email.toLowerCase() ? 1 : 0;
      totalFields++;
    }
    
    // Compare phone
    if (resmanLead.phone && domiqLead.phone) {
      score += this.phoneSimilarity(resmanLead.phone, domiqLead.phone);
      totalFields++;
    }
    
    return totalFields > 0 ? score / totalFields : 0;
  }
  
  private stringSimilarity(str1: string, str2: string): number {
    const longer = str1.length > str2.length ? str1 : str2;
    const shorter = str1.length > str2.length ? str2 : str1;
    
    if (longer.length === 0) return 1.0;
    
    const editDistance = this.levenshteinDistance(longer, shorter);
    return (longer.length - editDistance) / longer.length;
  }
  
  private phoneSimilarity(phone1: string, phone2: string): number {
    const clean1 = phone1.replace(/\D/g, '');
    const clean2 = phone2.replace(/\D/g, '');
    
    if (clean1 === clean2) return 1.0;
    
    // Check if one is a subset of the other (e.g., area code vs full number)
    if (clean1.includes(clean2) || clean2.includes(clean1)) return 0.8;
    
    return 0;
  }
  
  private levenshteinDistance(str1: string, str2: string): number {
    const matrix = [];
    
    for (let i = 0; i <= str2.length; i++) {
      matrix[i] = [i];
    }
    
    for (let j = 0; j <= str1.length; j++) {
      matrix[0][j] = j;
    }
    
    for (let i = 1; i <= str2.length; i++) {
      for (let j = 1; j <= str1.length; j++) {
        if (str2.charAt(i - 1) === str1.charAt(j - 1)) {
          matrix[i][j] = matrix[i - 1][j - 1];
        } else {
          matrix[i][j] = Math.min(
            matrix[i - 1][j - 1] + 1,
            matrix[i][j - 1] + 1,
            matrix[i - 1][j] + 1
          );
        }
      }
    }
    
    return matrix[str2.length][str1.length];
  }
  
  private async escalateToManualReview(conflict: LeadConflict): Promise<ConflictResolution> {
    // Create manual review task
    await createManualReviewTask(conflict);
    
    return {
      resolution: 'manual_review',
      final_value: null,
      reason: 'Escalated to manual review'
    };
  }
}
```

### Error Handling & Retry Logic

```typescript
interface RetryConfig {
  maxAttempts: number;
  backoffDelay: number;
  exponentialBackoff: boolean;
  retryableErrors: string[];
}

class LeadSyncRetryHandler {
  private config: RetryConfig;
  
  constructor(config: RetryConfig) {
    this.config = config;
  }
  
  async retryLeadSync(leadData: any, operation: string): Promise<any> {
    for (let attempt = 1; attempt <= this.config.maxAttempts; attempt++) {
      try {
        switch (operation) {
          case 'push':
            return await this.pushLeadToResMan(leadData);
          case 'pull':
            return await this.pullLeadFromResMan(leadData);
          case 'update':
            return await this.updateLeadInResMan(leadData);
          default:
            throw new Error(`Unknown operation: ${operation}`);
        }
      } catch (error) {
        if (attempt === this.config.maxAttempts) {
          await this.logSyncFailure(leadData, error, operation);
          throw error;
        }
        
        if (!this.config.retryableErrors.includes(error.code)) {
          throw error;
        }
        
        const delay = this.config.exponentialBackoff 
          ? this.config.backoffDelay * Math.pow(2, attempt - 1)
          : this.config.backoffDelay;
        
        await this.delay(delay);
        
        // Log retry attempt
        await this.logRetryAttempt(leadData, error, attempt, operation);
      }
    }
  }
  
  private async delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  private async logSyncFailure(leadData: any, error: any, operation: string): Promise<void> {
    await logSyncOperation({
      conversation_id: leadData.conversation_id,
      resman_lead_id: leadData.resman_lead_id,
      sync_direction: operation === 'push' ? 'to_resman' : 'from_resman',
      sync_status: 'failed',
      sync_type: operation,
      sync_data: leadData,
      error_message: error.message,
      error_code: error.code
    });
  }
  
  private async logRetryAttempt(leadData: any, error: any, attempt: number, operation: string): Promise<void> {
    console.log(`Retry attempt ${attempt} for ${operation} operation:`, {
      conversation_id: leadData.conversation_id,
      error: error.message,
      attempt
    });
  }
}

// Default retry configuration
const defaultRetryConfig: RetryConfig = {
  maxAttempts: 3,
  backoffDelay: 1000, // 1 second
  exponentialBackoff: true,
  retryableErrors: [
    'NETWORK_ERROR',
    'TIMEOUT_ERROR',
    'RATE_LIMIT_ERROR',
    'TEMPORARY_ERROR'
  ]
};
```

### Monitoring & Alerting Service

```typescript
class LeadSyncMonitor {
  private alertingRules: AlertingRules;
  
  constructor(alertingRules: AlertingRules) {
    this.alertingRules = alertingRules;
  }
  
  async trackSyncMetrics(metrics: SyncMetrics): Promise<void> {
    // Store metrics in database
    await this.storeSyncMetrics(metrics);
    
    // Check alerting rules
    await this.checkAlertingRules(metrics);
  }
  
  private async checkAlertingRules(metrics: SyncMetrics): Promise<void> {
    // Check sync failure rate
    if (metrics.syncFailureRate > this.alertingRules.syncFailureRate.threshold) {
      await this.triggerAlert('syncFailureRate', {
        current: metrics.syncFailureRate,
        threshold: this.alertingRules.syncFailureRate.threshold
      });
    }
    
    // Check sync latency
    if (metrics.avgSyncLatency > this.alertingRules.syncLatency.threshold) {
      await this.triggerAlert('syncLatency', {
        current: metrics.avgSyncLatency,
        threshold: this.alertingRules.syncLatency.threshold
      });
    }
    
    // Check API errors
    if (metrics.apiErrorCount > this.alertingRules.apiErrors.threshold) {
      await this.triggerAlert('apiErrors', {
        current: metrics.apiErrorCount,
        threshold: this.alertingRules.apiErrors.threshold
      });
    }
    
    // Check conflict rate
    if (metrics.conflictRate > this.alertingRules.conflictRate.threshold) {
      await this.triggerAlert('conflictRate', {
        current: metrics.conflictRate,
        threshold: this.alertingRules.conflictRate.threshold
      });
    }
  }
  
  private async triggerAlert(alertType: string, data: any): Promise<void> {
    const rule = this.alertingRules[alertType as keyof AlertingRules];
    
    switch (rule.action) {
      case 'email_team':
        await this.sendEmailAlert(alertType, data);
        break;
      case 'slack_alert':
        await this.sendSlackAlert(alertType, data);
        break;
      case 'pagerduty':
        await this.sendPagerDutyAlert(alertType, data);
        break;
      default:
        console.warn(`Unknown alert action: ${rule.action}`);
    }
  }
  
  private async sendEmailAlert(alertType: string, data: any): Promise<void> {
    // Implementation for email alerts
    console.log(`Email alert: ${alertType}`, data);
  }
  
  private async sendSlackAlert(alertType: string, data: any): Promise<void> {
    // Implementation for Slack alerts
    console.log(`Slack alert: ${alertType}`, data);
  }
  
  private async sendPagerDutyAlert(alertType: string, data: any): Promise<void> {
    // Implementation for PagerDuty alerts
    console.log(`PagerDuty alert: ${alertType}`, data);
  }
}

interface AlertingRules {
  syncFailureRate: {
    threshold: number;
    window: string;
    action: string;
  };
  syncLatency: {
    threshold: number;
    window: string;
    action: string;
  };
  apiErrors: {
    threshold: number;
    window: string;
    action: string;
  };
  conflictRate: {
    threshold: number;
    window: string;
    action: string;
  };
  bidirectionalSyncLag: {
    threshold: number;
    window: string;
    action: string;
  };
}

interface SyncMetrics {
  syncFailureRate: number;
  avgSyncLatency: number;
  apiErrorCount: number;
  conflictRate: number;
  bidirectionalSyncLag: number;
  totalOperations: number;
  successfulOperations: number;
  failedOperations: number;
}
```

This backend component provides comprehensive API endpoints and services for bidirectional lead synchronization between DomIQ and ResMan, including data transformation, conflict resolution, error handling, and monitoring capabilities. 