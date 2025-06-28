# Lead Sync Integration: DomIQ ↔ ResMan

## Feature Overview
Bidirectional synchronization of leads between DomIQ AI chatbot and ResMan Lead Management module.

## What It Delivers for On-site Teams
- **Unified Lead View**: All chatbot inquiries appear in ResMan Lead Management
- **Real-time Updates**: Lead status changes sync across both systems
- **Complete Lead History**: Full conversation context available in ResMan
- **Automated Lead Creation**: No manual data entry required
- **Status Tracking**: Lead progression tracked from initial contact to lease
- **Bidirectional Sync**: Changes in ResMan automatically update chatbot context

## Data Flow Architecture

### 1. Lead Creation Flow (DomIQ → ResMan)
```
Chatbot Lead Created → Data Transformation → ResMan API → Lead Management Module
```

### 2. Status Update Flow (ResMan → DomIQ)
```
ResMan Status Change → Webhook → Data Transformation → Chatbot Context Update
```

### 3. Lead Pull Flow (DomIQ ← ResMan)
```
Scheduled Sync → ResMan API → Lead Data → Chatbot Context Enrichment
```

### 4. Real-time Sync Flow (Bidirectional)
```
WebSocket Connection → Real-time Updates → Bidirectional Sync → Context Consistency
```

## Implementation Requirements

### Database Schema Updates
```sql
-- Add ResMan integration tables to existing schema
CREATE TABLE resman_integration (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id),
    resman_property_id VARCHAR(100),
    api_credentials JSONB,
    webhook_url VARCHAR(255),
    sync_enabled BOOLEAN DEFAULT TRUE,
    sync_direction VARCHAR(20) DEFAULT 'bidirectional', -- 'to_resman', 'from_resman', 'bidirectional'
    last_sync_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE lead_sync_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id),
    resman_lead_id VARCHAR(100),
    sync_direction VARCHAR(20), -- 'to_resman', 'from_resman', 'bidirectional'
    sync_status VARCHAR(50), -- 'pending', 'success', 'failed', 'conflict'
    sync_type VARCHAR(50), -- 'create', 'update', 'status_change', 'enrichment'
    error_message TEXT,
    conflict_resolution VARCHAR(50), -- 'resman_wins', 'domiq_wins', 'manual_review'
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE lead_sync_queue (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id),
    resman_lead_id VARCHAR(100),
    sync_operation VARCHAR(50), -- 'push', 'pull', 'merge'
    sync_data JSONB,
    priority INTEGER DEFAULT 1, -- 1=low, 2=medium, 3=high
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    next_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### API Endpoints to Create

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
  };
}

interface CreateLeadResponse {
  success: boolean;
  resman_lead_id: string;
  sync_status: string;
  created_at: string;
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
  }>;
  total_count: number;
  sync_summary: {
    new_leads: number;
    updated_leads: number;
    conflicts_resolved: number;
  };
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
}
```

#### 6. Real-time Sync API
```typescript
// WebSocket endpoint for real-time bidirectional sync
// WS /api/resman/sync/websocket
interface RealTimeSyncMessage {
  type: 'lead_update' | 'status_change' | 'new_lead' | 'sync_conflict';
  data: any;
  timestamp: string;
  source: 'resman' | 'domiq';
}
```

### Data Transformation Layer

#### DomIQ → ResMan Mapping
```typescript
interface DomIQToResManMapper {
  // Map DomIQ conversation data to ResMan lead format
  mapConversationToLead(conversation: Conversation): ResManLead;
  
  // Map DomIQ user data to ResMan contact format
  mapUserToContact(user: User): ResManContact;
  
  // Map DomIQ tour booking to ResMan appointment
  mapTourToAppointment(tour: TourBooking): ResManAppointment;
  
  // Map DomIQ lead score to ResMan lead score
  mapLeadScore(domiqScore: number): number;
}
```

#### ResMan → DomIQ Mapping
```typescript
interface ResManToDomIQMapper {
  // Map ResMan lead status to DomIQ conversation context
  mapLeadStatusToContext(status: string): ConversationContext;
  
  // Map ResMan notes to DomIQ conversation updates
  mapNotesToUpdates(notes: string[]): ConversationUpdate[];
  
  // Map ResMan lead data to DomIQ user profile
  mapLeadToUserProfile(lead: ResManLead): UserProfile;
  
  // Map ResMan custom fields to DomIQ context
  mapCustomFieldsToContext(customFields: Record<string, any>): ConversationContext;
}
```

## Integration Points

### 1. Chatbot Lead Creation Hook (Push)
```typescript
// In ChatModal.tsx - after lead qualification
const handleLeadQualification = async (leadData: LeadData) => {
  // Create lead in local database
  const conversation = await createConversation(leadData);
  
  // Push to ResMan
  await syncLeadToResMan(conversation);
  
  // Update conversation context
  updateConversationContext(conversation);
};
```

### 2. Scheduled Lead Pull (Pull)
```typescript
// Scheduled job to pull leads from ResMan
const scheduledLeadPull = async () => {
  try {
    const leads = await pullLeadsFromResMan({
      date_from: getLastPullDate(),
      include_conversations: true
    });
    
    for (const lead of leads) {
      await processResManLead(lead);
    }
    
    await updateLastPullDate();
  } catch (error) {
    console.error('Error pulling leads from ResMan:', error);
  }
};
```

### 3. ResMan Webhook Handler (Pull)
```typescript
// In Next.js API route
export async function POST(request: Request) {
  const webhookData = await request.json();
  
  // Validate webhook signature
  if (!validateWebhookSignature(request)) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  // Process lead update from ResMan
  await processResManLeadUpdate(webhookData);
  
  return new Response('OK', { status: 200 });
}
```

### 4. Real-time Bidirectional Sync
```typescript
// WebSocket connection for real-time bidirectional updates
class RealTimeLeadSync {
  private resmanWebSocket: WebSocket;
  private domiqWebSocket: WebSocket;
  
  constructor() {
    this.initializeWebSockets();
  }
  
  private initializeWebSockets() {
    // Connect to ResMan WebSocket
    this.resmanWebSocket = new WebSocket(RESMAN_WS_URL);
    this.resmanWebSocket.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.handleResManUpdate(update);
    };
    
    // Connect to DomIQ WebSocket
    this.domiqWebSocket = new WebSocket(DOMIQ_WS_URL);
    this.domiqWebSocket.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.handleDomIQUpdate(update);
    };
  }
  
  private async handleResManUpdate(update: any) {
    // Process update from ResMan
    await this.processResManUpdate(update);
    
    // Update DomIQ context
    await this.updateDomIQContext(update);
  }
  
  private async handleDomIQUpdate(update: any) {
    // Process update from DomIQ
    await this.processDomIQUpdate(update);
    
    // Push to ResMan
    await this.pushToResMan(update);
  }
}
```

### 5. Conflict Resolution Service
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
}
```

## Error Handling & Retry Logic

### Retry Strategy
```typescript
interface RetryConfig {
  maxAttempts: number;
  backoffDelay: number;
  exponentialBackoff: boolean;
  retryableErrors: string[];
}

const retryLeadSync = async (leadData: any, config: RetryConfig) => {
  for (let attempt = 1; attempt <= config.maxAttempts; attempt++) {
    try {
      return await syncLeadToResMan(leadData);
    } catch (error) {
      if (attempt === config.maxAttempts) {
        await logSyncFailure(leadData, error);
        throw error;
      }
      
      if (!config.retryableErrors.includes(error.code)) {
        throw error;
      }
      
      await delay(config.backoffDelay * Math.pow(2, attempt - 1));
    }
  }
};
```

### Conflict Resolution
```typescript
interface ConflictResolution {
  // Handle duplicate leads
  handleDuplicateLead(existingLead: ResManLead, newLead: DomIQLead): ResManLead;
  
  // Handle conflicting status updates
  resolveStatusConflict(resmanStatus: string, domiqStatus: string): string;
  
  // Handle data inconsistencies
  resolveDataInconsistency(field: string, resmanValue: any, domiqValue: any): any;
  
  // Handle sync conflicts
  resolveSyncConflict(conflict: SyncConflict): ConflictResolution;
}
```

## Monitoring & Alerting

### Key Metrics to Track
- Lead sync success rate (push and pull)
- Sync latency (time from creation to ResMan)
- Failed sync attempts
- Data consistency checks
- API response times
- Conflict resolution rates
- Bidirectional sync performance

### Alerting Rules
```typescript
const alertingRules = {
  syncFailureRate: {
    threshold: 0.05, // 5% failure rate
    window: '5m',
    action: 'email_team'
  },
  syncLatency: {
    threshold: 30000, // 30 seconds
    window: '1m',
    action: 'slack_alert'
  },
  apiErrors: {
    threshold: 10,
    window: '1m',
    action: 'pagerduty'
  },
  conflictRate: {
    threshold: 0.1, // 10% conflict rate
    window: '10m',
    action: 'slack_alert'
  },
  bidirectionalSyncLag: {
    threshold: 60000, // 1 minute lag
    window: '5m',
    action: 'email_team'
  }
};
```

## Testing Strategy

### Unit Tests
- Data transformation functions (both directions)
- API endpoint handlers
- Error handling logic
- Conflict resolution rules
- Bidirectional sync logic

### Integration Tests
- End-to-end lead sync flow (push and pull)
- Webhook processing
- Real-time updates
- Error scenarios
- Conflict resolution scenarios

### Load Tests
- High-volume lead creation
- Concurrent status updates
- API rate limiting
- Database performance
- Bidirectional sync under load

## Deployment Checklist

### Pre-deployment
- [ ] ResMan API credentials configured
- [ ] Webhook endpoints registered
- [ ] Data mapping validated (both directions)
- [ ] Error handling tested
- [ ] Monitoring configured
- [ ] Conflict resolution rules defined
- [ ] Bidirectional sync tested

### Post-deployment
- [ ] Verify lead sync functionality (push and pull)
- [ ] Test status update flow (both directions)
- [ ] Monitor error rates
- [ ] Validate data consistency
- [ ] Test conflict resolution
- [ ] Verify real-time sync
- [ ] Train onsite teams

## Rollback Plan
1. Disable ResMan integration via feature flag
2. Continue local lead storage
3. Manually sync critical leads
4. Investigate and fix issues
5. Re-enable integration

## Success Criteria
- [ ] 99%+ lead sync success rate (both directions)
- [ ] <30 second sync latency
- [ ] Zero data loss during sync
- [ ] <5% conflict rate
- [ ] Real-time bidirectional sync
- [ ] Onsite team adoption >80%
- [ ] Reduced manual data entry by 90% 