# Multi-Channel Communications Logging: DomIQ ↔ ResMan

## Feature Overview
Automatic logging of all AI-generated communications (emails, SMS, chat transcripts) to ResMan prospect records for complete communication history, compliance, and follow-up purposes.

## What It Delivers for On-site Teams
- **Complete Communication History**: All AI interactions automatically logged in ResMan
- **Compliance Documentation**: Full audit trail for regulatory requirements
- **Follow-up Context**: Complete conversation history for staff reference
- **Multi-channel Visibility**: Unified view of emails, SMS, and chat interactions
- **Automated Logging**: No manual data entry required
- **Searchable Records**: Easy retrieval of past communications
- **Bidirectional Sync**: Communication updates from ResMan enhance chatbot context
- **Real-time Updates**: Instant sync of communication changes across both systems

## Data Flow Architecture

### 1. Communication Capture Flow (DomIQ → ResMan)
```
AI Communication → Content Processing → ResMan API → Prospect Record → Communication Log
```

### 2. Real-time Logging Flow (DomIQ → ResMan)
```
Chat Message → Immediate Processing → ResMan Update → Real-time Log Entry
```

### 3. Communication Pull Flow (DomIQ ← ResMan)
```
Scheduled Sync → ResMan API → Communication History → Chatbot Context Enrichment
```

### 4. Bidirectional Sync Flow (Real-time)
```
WebSocket Connection → Real-time Updates → Bidirectional Sync → Context Consistency
```

## Implementation Requirements

### Database Schema Updates
```sql
-- Add communication logging integration tables
CREATE TABLE communication_log_sync (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id),
    resman_prospect_id VARCHAR(100),
    communication_type VARCHAR(50), -- 'chat', 'email', 'sms', 'voice'
    message_id VARCHAR(100),
    sender_type VARCHAR(50), -- 'ai', 'prospect', 'staff'
    content TEXT,
    content_summary TEXT,
    sentiment_score DECIMAL(3,2),
    key_topics JSONB,
    timestamp TIMESTAMPTZ,
    sync_direction VARCHAR(20), -- 'to_resman', 'from_resman', 'bidirectional'
    sync_status VARCHAR(50), -- 'pending', 'success', 'failed', 'conflict'
    sync_type VARCHAR(50), -- 'create', 'update', 'enrichment', 'compliance'
    retry_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE communication_metadata (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    communication_log_id UUID REFERENCES communication_log_sync(id),
    metadata_type VARCHAR(50), -- 'attachments', 'links', 'emotions', 'intents'
    metadata_value JSONB,
    sync_status VARCHAR(50), -- 'pending', 'success', 'failed'
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE communication_compliance_flags (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    communication_log_id UUID REFERENCES communication_log_sync(id),
    flag_type VARCHAR(50), -- 'gdpr_consent', 'opt_out', 'sensitive_data', 'regulatory'
    flag_description TEXT,
    flag_severity VARCHAR(20), -- 'low', 'medium', 'high', 'critical'
    is_resolved BOOLEAN DEFAULT FALSE,
    resolved_by VARCHAR(100),
    resolved_at TIMESTAMPTZ,
    sync_status VARCHAR(50), -- 'pending', 'success', 'failed'
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE communication_sync_queue (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    communication_log_id UUID REFERENCES communication_log_sync(id),
    sync_operation VARCHAR(50), -- 'push', 'pull', 'merge', 'compliance_check'
    sync_data JSONB,
    priority INTEGER DEFAULT 1, -- 1=low, 2=medium, 3=high
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    next_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### API Endpoints to Create

#### 1. Communication Logging API (Push to ResMan)
```typescript
// POST /api/resman/communications/log
interface LogCommunicationRequest {
  prospect_id: string;
  communication_data: {
    type: 'chat' | 'email' | 'sms' | 'voice';
    sender_type: 'ai' | 'prospect' | 'staff';
    content: string;
    content_summary?: string;
    sentiment_score?: number;
    key_topics?: string[];
    timestamp: string;
    metadata?: {
      attachments?: Array<{
        name: string;
        type: string;
        size: number;
        url?: string;
      }>;
      links?: string[];
      emotions?: Record<string, number>;
      intents?: string[];
    };
  };
  compliance_flags?: Array<{
    type: string;
    description: string;
    severity: 'low' | 'medium' | 'high' | 'critical';
  }>;
  sync_direction?: 'to_resman' | 'bidirectional';
}

interface LogCommunicationResponse {
  success: boolean;
  log_entry_id: string;
  resman_communication_id: string;
  compliance_flags_raised: number;
  processing_time_ms: number;
  sync_conflicts?: Array<{
    field: string;
    resman_value: any;
    domiq_value: any;
    resolution: string;
  }>;
}
```

#### 2. Communication Pull API (Pull from ResMan)
```typescript
// GET /api/resman/communications/pull
interface PullCommunicationsRequest {
  prospect_id?: string;
  date_from?: string;
  date_to?: string;
  communication_types?: string[];
  include_metadata?: boolean;
  include_compliance_flags?: boolean;
  limit?: number;
  offset?: number;
}

interface PullCommunicationsResponse {
  communications: Array<{
    resman_communication_id: string;
    prospect_id: string;
    type: string;
    sender_type: string;
    content: string;
    content_summary?: string;
    sentiment_score?: number;
    timestamp: string;
    metadata?: Record<string, any>;
    compliance_flags?: Array<{
      type: string;
      description: string;
      severity: string;
      is_resolved: boolean;
    }>;
  }>;
  total_count: number;
  sync_summary: {
    new_communications: number;
    updated_communications: number;
    conflicts_resolved: number;
  };
}
```

#### 3. Communication History API (Bidirectional)
```typescript
// GET /api/resman/communications/{prospect_id}/history
interface CommunicationHistoryRequest {
  prospect_id: string;
  date_from?: string;
  date_to?: string;
  communication_types?: string[];
  limit?: number;
  offset?: number;
  include_sync_status?: boolean;
}

interface CommunicationHistoryResponse {
  communications: Array<{
    id: string;
    type: string;
    sender_type: string;
    content: string;
    content_summary?: string;
    sentiment_score?: number;
    timestamp: string;
    metadata?: Record<string, any>;
    compliance_flags?: Array<{
      type: string;
      description: string;
      severity: string;
      is_resolved: boolean;
    }>;
    sync_status?: {
      direction: string;
      status: string;
      last_sync: string;
    };
  }>;
  total_count: number;
  date_range: {
    from: string;
    to: string;
  };
  sync_status: {
    last_pull: string;
    last_push: string;
    conflicts_pending: number;
  };
}
```

#### 4. Communication Search API (Enhanced)
```typescript
// GET /api/resman/communications/search
interface CommunicationSearchRequest {
  prospect_id?: string;
  search_query: string;
  date_from?: string;
  date_to?: string;
  communication_types?: string[];
  sentiment_filter?: 'positive' | 'negative' | 'neutral';
  compliance_flags?: string[];
  sync_status?: string[];
  limit?: number;
}

interface CommunicationSearchResponse {
  results: Array<{
    id: string;
    prospect_id: string;
    type: string;
    content: string;
    content_summary?: string;
    timestamp: string;
    relevance_score: number;
    matched_terms: string[];
    sync_status?: string;
  }>;
  total_count: number;
  search_metadata: {
    query: string;
    processing_time_ms: number;
    filters_applied: Record<string, any>;
    sync_coverage: {
      resman_only: number;
      domiq_only: number;
      both_systems: number;
    };
  };
}
```

#### 5. Communication Merge API (Conflict Resolution)
```typescript
// POST /api/resman/communications/merge
interface MergeCommunicationsRequest {
  resman_communication_id: string;
  domiq_communication_id: string;
  merge_strategy: 'resman_wins' | 'domiq_wins' | 'manual_review';
  merge_fields: string[];
  conflict_resolution?: Record<string, any>;
}

interface MergeCommunicationsResponse {
  success: boolean;
  merged_communication_id: string;
  conflicts_resolved: number;
  merge_summary: {
    fields_merged: string[];
    fields_conflicted: string[];
    final_values: Record<string, any>;
  };
}
```

#### 6. Compliance Flag Management API (Enhanced)
```typescript
// POST /api/resman/communications/{log_id}/compliance-flags
interface CreateComplianceFlagRequest {
  flag_type: string;
  description: string;
  severity: 'low' | 'medium' | 'high' | 'critical';
  evidence?: Record<string, any>;
  sync_direction?: 'to_resman' | 'from_resman';
}

// PUT /api/resman/compliance-flags/{flag_id}/resolve
interface ResolveComplianceFlagRequest {
  resolution_notes: string;
  resolved_by: string;
  resolution_action?: string;
  sync_to_resman?: boolean;
}

// GET /api/resman/compliance-flags/sync
interface SyncComplianceFlagsRequest {
  date_from?: string;
  date_to?: string;
  flag_types?: string[];
  include_resolved?: boolean;
}

interface SyncComplianceFlagsResponse {
  flags: Array<{
    id: string;
    communication_id: string;
    flag_type: string;
    description: string;
    severity: string;
    is_resolved: boolean;
    sync_status: string;
  }>;
  sync_summary: {
    new_flags: number;
    updated_flags: number;
    resolved_flags: number;
  };
}
```

#### 7. Real-time Communication Sync API
```typescript
// WebSocket endpoint for real-time bidirectional communication sync
// WS /api/resman/communications/sync/websocket
interface RealTimeCommunicationSyncMessage {
  type: 'communication_log' | 'compliance_flag' | 'metadata_update' | 'sync_conflict';
  data: any;
  timestamp: string;
  source: 'resman' | 'domiq';
  sync_direction: 'push' | 'pull' | 'bidirectional';
}
```

## Integration Points

### 1. Chat Message Logging (Push)
```typescript
// In ChatModal.tsx - after each message
const handleMessageLogging = async (message: ChatMessage) => {
  try {
    const logData = {
      prospect_id: conversation.resman_prospect_id,
      communication_data: {
        type: 'chat',
        sender_type: message.sender === 'user' ? 'prospect' : 'ai',
        content: message.content,
        content_summary: await generateMessageSummary(message.content),
        sentiment_score: await analyzeSentiment(message.content),
        key_topics: await extractKeyTopics(message.content),
        timestamp: message.timestamp,
        metadata: {
          message_id: message.id,
          conversation_id: conversation.id,
          message_length: message.content.length,
          response_time_ms: message.response_time,
          ai_model_used: message.ai_model,
          confidence_score: message.confidence
        }
      },
      sync_direction: 'to_resman'
    };

    // Check for compliance flags
    const complianceFlags = await checkComplianceFlags(message.content);
    if (complianceFlags.length > 0) {
      logData.compliance_flags = complianceFlags;
    }

    // Log to ResMan
    await logCommunicationToResMan(logData);

    // Update local sync status
    await updateCommunicationSyncStatus(message.id, 'success');

  } catch (error) {
    console.error('Error logging communication:', error);
    await updateCommunicationSyncStatus(message.id, 'failed');
    await queueCommunicationRetry(message.id);
  }
};
```

### 2. Scheduled Communication Pull (Pull)
```typescript
// Scheduled job to pull communications from ResMan
const scheduledCommunicationPull = async () => {
  try {
    const communications = await pullCommunicationsFromResMan({
      date_from: getLastCommunicationPullDate(),
      include_metadata: true,
      include_compliance_flags: true
    });
    
    for (const communication of communications.communications) {
      await processResManCommunication(communication);
    }
    
    await updateLastCommunicationPullDate();
    
    // Enrich chatbot context with new communications
    await enrichChatbotContextWithCommunications(communications);
    
  } catch (error) {
    console.error('Error pulling communications from ResMan:', error);
  }
};
```

### 3. Email Communication Logging (Push)
```typescript
// In email service - after sending emails
const handleEmailLogging = async (emailData: EmailData) => {
  try {
    const logData = {
      prospect_id: emailData.prospect_id,
      communication_data: {
        type: 'email',
        sender_type: 'ai',
        content: emailData.body,
        content_summary: await generateEmailSummary(emailData.body),
        sentiment_score: await analyzeSentiment(emailData.body),
        key_topics: await extractKeyTopics(emailData.body),
        timestamp: emailData.sent_at,
        metadata: {
          email_id: emailData.id,
          subject: emailData.subject,
          recipient: emailData.to,
          template_used: emailData.template_id,
          delivery_status: emailData.delivery_status,
          open_rate: emailData.open_rate,
          click_rate: emailData.click_rate
        }
      },
      sync_direction: 'to_resman'
    };

    // Log to ResMan
    await logCommunicationToResMan(logData);

    // Update email tracking
    await updateEmailTracking(emailData.id, 'logged_to_resman');

  } catch (error) {
    console.error('Error logging email communication:', error);
    await queueEmailLoggingRetry(emailData.id);
  }
};
```

### 4. SMS Communication Logging (Push)
```typescript
// In SMS service - after sending/receiving SMS
const handleSMSLogging = async (smsData: SMSData) => {
  try {
    const logData = {
      prospect_id: smsData.prospect_id,
      communication_data: {
        type: 'sms',
        sender_type: smsData.direction === 'outbound' ? 'ai' : 'prospect',
        content: smsData.message,
        content_summary: await generateSMSSummary(smsData.message),
        sentiment_score: await analyzeSentiment(smsData.message),
        key_topics: await extractKeyTopics(smsData.message),
        timestamp: smsData.timestamp,
        metadata: {
          sms_id: smsData.id,
          phone_number: smsData.phone_number,
          direction: smsData.direction,
          delivery_status: smsData.delivery_status,
          message_length: smsData.message.length,
          cost: smsData.cost
        }
      },
      sync_direction: 'to_resman'
    };

    // Log to ResMan
    await logCommunicationToResMan(logData);

    // Update SMS tracking
    await updateSMSTracking(smsData.id, 'logged_to_resman');

  } catch (error) {
    console.error('Error logging SMS communication:', error);
    await queueSMSLoggingRetry(smsData.id);
  }
};
```

### 5. Real-time Bidirectional Communication Sync
```typescript
// WebSocket connection for real-time bidirectional communication sync
class RealTimeCommunicationSync {
  private resmanWebSocket: WebSocket;
  private domiqWebSocket: WebSocket;
  
  constructor() {
    this.initializeWebSockets();
  }
  
  private initializeWebSockets() {
    // Connect to ResMan WebSocket
    this.resmanWebSocket = new WebSocket(RESMAN_COMMUNICATION_WS_URL);
    this.resmanWebSocket.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.handleResManCommunicationUpdate(update);
    };
    
    // Connect to DomIQ WebSocket
    this.domiqWebSocket = new WebSocket(DOMIQ_COMMUNICATION_WS_URL);
    this.domiqWebSocket.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.handleDomIQCommunicationUpdate(update);
    };
  }
  
  private async handleResManCommunicationUpdate(update: any) {
    // Process communication update from ResMan
    await this.processResManCommunicationUpdate(update);
    
    // Update DomIQ context with new communication data
    await this.updateDomIQContextWithCommunication(update);
    
    // Check for compliance flags
    if (update.compliance_flags) {
      await this.processComplianceFlags(update.compliance_flags);
    }
  }
  
  private async handleDomIQCommunicationUpdate(update: any) {
    // Process communication update from DomIQ
    await this.processDomIQCommunicationUpdate(update);
    
    // Push to ResMan
    await this.pushCommunicationToResMan(update);
  }
}
```

### 6. Communication Conflict Resolution Service
```typescript
class CommunicationSyncConflictResolver {
  async resolveConflict(conflict: CommunicationConflict): Promise<ConflictResolution> {
    const { field, resman_value, domiq_value, conflict_type } = conflict;
    
    switch (conflict_type) {
      case 'content_conflict':
        return this.resolveContentConflict(resman_value, domiq_value);
      
      case 'metadata_conflict':
        return this.resolveMetadataConflict(field, resman_value, domiq_value);
      
      case 'compliance_flag_conflict':
        return this.resolveComplianceFlagConflict(resman_value, domiq_value);
      
      case 'duplicate_communication':
        return this.resolveDuplicateCommunication(resman_value, domiq_value);
      
      default:
        return this.escalateToManualReview(conflict);
    }
  }
  
  private async resolveContentConflict(resmanContent: string, domiqContent: string): Promise<ConflictResolution> {
    // Business logic for content conflict resolution
    // Usually ResMan content is more authoritative for historical communications
    return {
      resolution: 'resman_wins',
      final_value: resmanContent,
      reason: 'ResMan content is authoritative for historical communications'
    };
  }
  
  private async resolveComplianceFlagConflict(resmanFlags: any[], domiqFlags: any[]): Promise<ConflictResolution> {
    // Merge compliance flags from both systems
    const mergedFlags = [...resmanFlags, ...domiqFlags];
    const uniqueFlags = this.deduplicateComplianceFlags(mergedFlags);
    
    return {
      resolution: 'merge',
      final_value: uniqueFlags,
      reason: 'Compliance flags should be merged from both systems'
    };
  }
}
```

## Content Processing

### 1. Message Summarization Service
```typescript
class MessageSummarizationService {
  async generateMessageSummary(content: string): Promise<string> {
    // Use AI to generate concise summary
    const summary = await this.aiService.summarize({
      text: content,
      max_length: 200,
      include_key_points: true
    });
    
    return summary;
  }

  async generateEmailSummary(emailBody: string): Promise<string> {
    // Extract key information from email
    const summary = await this.aiService.extractEmailSummary({
      body: emailBody,
      include_action_items: true,
      include_sentiment: true
    });
    
    return summary;
  }

  async generateSMSSummary(smsMessage: string): Promise<string> {
    // For SMS, often the message is short enough to use as-is
    if (smsMessage.length <= 100) {
      return smsMessage;
    }
    
    // Otherwise, summarize
    return await this.aiService.summarize({
      text: smsMessage,
      max_length: 100
    });
  }
}
```

### 2. Sentiment Analysis Service
```typescript
class SentimentAnalysisService {
  async analyzeSentiment(content: string): Promise<number> {
    const sentiment = await this.aiService.analyzeSentiment({
      text: content,
      return_score: true
    });
    
    // Return score between -1 (negative) and 1 (positive)
    return sentiment.score;
  }

  async analyzeConversationSentiment(messages: Message[]): Promise<{
    overall_sentiment: number;
    sentiment_trend: 'improving' | 'declining' | 'stable';
    key_emotions: Record<string, number>;
  }> {
    const analysis = await this.aiService.analyzeConversationSentiment({
      messages: messages.map(m => ({ content: m.content, sender: m.sender })),
      include_trend_analysis: true,
      include_emotion_detection: true
    });
    
    return analysis;
  }
}
```

### 3. Key Topics Extraction Service
```typescript
class KeyTopicsExtractionService {
  async extractKeyTopics(content: string): Promise<string[]> {
    const topics = await this.aiService.extractTopics({
      text: content,
      max_topics: 5,
      include_confidence: true
    });
    
    return topics.map(topic => topic.name);
  }

  async categorizeCommunication(content: string): Promise<{
    primary_category: string;
    secondary_categories: string[];
    urgency_level: 'low' | 'medium' | 'high';
    action_required: boolean;
  }> {
    const categorization = await this.aiService.categorizeCommunication({
      text: content,
      categories: [
        'property_inquiry',
        'tour_booking',
        'application_status',
        'maintenance_request',
        'general_question',
        'complaint',
        'feedback'
      ]
    });
    
    return categorization;
  }
}
```

## Compliance Management

### 1. Compliance Flag Detection
```typescript
class ComplianceFlagService {
  async checkComplianceFlags(content: string): Promise<ComplianceFlag[]> {
    const flags: ComplianceFlag[] = [];

    // Check for GDPR consent issues
    if (await this.checkGDPRConsent(content)) {
      flags.push({
        type: 'gdpr_consent',
        description: 'Potential GDPR consent issue detected',
        severity: 'high'
      });
    }

    // Check for opt-out requests
    if (await this.checkOptOutRequest(content)) {
      flags.push({
        type: 'opt_out',
        description: 'Opt-out request detected',
        severity: 'critical'
      });
    }

    // Check for sensitive data
    if (await this.checkSensitiveData(content)) {
      flags.push({
        type: 'sensitive_data',
        description: 'Sensitive personal data detected',
        severity: 'high'
      });
    }

    // Check for regulatory compliance
    if (await this.checkRegulatoryCompliance(content)) {
      flags.push({
        type: 'regulatory',
        description: 'Potential regulatory compliance issue',
        severity: 'medium'
      });
    }

    return flags;
  }

  private async checkGDPRConsent(content: string): Promise<boolean> {
    const gdprKeywords = [
      'consent', 'permission', 'agree', 'opt-in', 'opt-out',
      'data protection', 'privacy', 'personal information'
    ];
    
    return gdprKeywords.some(keyword => 
      content.toLowerCase().includes(keyword)
    );
  }

  private async checkOptOutRequest(content: string): Promise<boolean> {
    const optOutKeywords = [
      'stop', 'unsubscribe', 'opt out', 'no more', 'remove me',
      'don\'t contact', 'cease communication'
    ];
    
    return optOutKeywords.some(keyword => 
      content.toLowerCase().includes(keyword)
    );
  }

  private async checkSensitiveData(content: string): Promise<boolean> {
    // Check for SSN, credit card, etc.
    const sensitivePatterns = [
      /\b\d{3}-\d{2}-\d{4}\b/, // SSN
      /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/, // Credit card
      /\b\d{3}[\s-]?\d{3}[\s-]?\d{4}\b/ // Phone with area code
    ];
    
    return sensitivePatterns.some(pattern => pattern.test(content));
  }
}
```

## Real-time Logging

### 1. Real-time Communication Sync
```typescript
class RealTimeCommunicationSync {
  private syncQueue: Array<CommunicationLog> = [];
  private isProcessing = false;

  async addToSyncQueue(communication: CommunicationLog): Promise<void> {
    this.syncQueue.push(communication);
    
    if (!this.isProcessing) {
      this.processQueue();
    }
  }

  private async processQueue(): Promise<void> {
    this.isProcessing = true;
    
    while (this.syncQueue.length > 0) {
      const communication = this.syncQueue.shift();
      if (communication) {
        try {
          await this.syncToResMan(communication);
        } catch (error) {
          console.error('Error syncing communication:', error);
          // Re-queue for retry
          this.syncQueue.push(communication);
        }
      }
    }
    
    this.isProcessing = false;
  }

  private async syncToResMan(communication: CommunicationLog): Promise<void> {
    const response = await fetch('/api/resman/communications/log', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(communication)
    });

    if (!response.ok) {
      throw new Error(`Failed to sync communication: ${response.statusText}`);
    }
  }
}
```

## Monitoring & Analytics

### 1. Communication Analytics
```typescript
interface CommunicationAnalytics {
  getCommunicationMetrics(prospectId: string, dateRange: DateRange): Promise<{
    total_communications: number;
    communication_types: Record<string, number>;
    average_sentiment: number;
    response_times: {
      ai_response_time: number;
      staff_response_time: number;
    };
    compliance_flags_count: number;
    sync_coverage: {
      resman_only: number;
      domiq_only: number;
      both_systems: number;
    };
  }>;

  getCommunicationTrends(prospectId: string): Promise<{
    daily_communications: Array<{date: string, count: number}>;
    sentiment_trend: Array<{date: string, sentiment: number}>;
    communication_volume_by_type: Record<string, number>;
    sync_performance: {
      push_success_rate: number;
      pull_success_rate: number;
      conflict_resolution_rate: number;
    };
  }>;

  getComplianceMetrics(prospectId: string): Promise<{
    total_flags: number;
    flags_by_type: Record<string, number>;
    flags_by_severity: Record<string, number>;
    resolution_rate: number;
    average_resolution_time: number;
    sync_status: {
      flags_synced_to_resman: number;
      flags_pulled_from_resman: number;
      pending_sync: number;
    };
  }>;
}
```

## Testing Strategy

### Unit Tests
- Content processing functions (summarization, sentiment analysis)
- Compliance flag detection logic
- Data transformation functions (both directions)
- Error handling scenarios
- Conflict resolution logic

### Integration Tests
- End-to-end communication logging flow (push and pull)
- Real-time sync functionality
- Compliance flag processing
- Search and retrieval functionality
- Bidirectional sync scenarios

### User Acceptance Tests
- Communication history viewing
- Search functionality
- Compliance flag management
- Export and reporting capabilities
- Real-time sync performance

## Deployment Checklist

### Pre-deployment
- [ ] ResMan communication API access configured
- [ ] Content processing services deployed
- [ ] Compliance detection rules configured
- [ ] Real-time sync infrastructure ready
- [ ] Data retention policies defined
- [ ] Bidirectional sync tested
- [ ] Conflict resolution rules defined

### Post-deployment
- [ ] Verify communication logging functionality (push and pull)
- [ ] Test compliance flag detection
- [ ] Validate search and retrieval
- [ ] Check real-time sync performance
- [ ] Monitor compliance metrics
- [ ] Test bidirectional sync
- [ ] Verify conflict resolution

## Success Criteria
- [ ] 100% communication logging success rate (both directions)
- [ ] <5 second logging latency
- [ ] 95%+ compliance flag accuracy
- [ ] Real-time communication sync
- [ ] Complete audit trail preservation
- [ ] Zero data loss during logging operations
- [ ] <5% sync conflict rate
- [ ] Bidirectional sync performance <10 seconds 