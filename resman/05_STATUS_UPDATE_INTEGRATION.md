# Status Update Integration: DomIQ ↔ ResMan

## Feature Overview
Real-time synchronization of lead status changes between DomIQ AI chatbot and ResMan Lead Management system.

## What It Delivers for On-site Teams
- **Real-time Status Updates**: Instant visibility of lead progression across both systems
- **Automated Status Transitions**: Chatbot automatically updates status based on conversation outcomes
- **Staff Notifications**: Immediate alerts when leads change status
- **Status History**: Complete audit trail of all status changes
- **Contextual Updates**: Status changes include relevant conversation context
- **Workflow Automation**: Trigger automated actions based on status changes

## Data Flow Architecture

### 1. Status Update Flow (DomIQ → ResMan)
```
Chatbot Status Change → Status Validation → ResMan API → Lead Management Update → Staff Notification
```

### 2. Status Update Flow (ResMan → DomIQ)
```
ResMan Status Change → Webhook → Chatbot Context Update → Conversation Enhancement → Prospect Notification
```

## Implementation Requirements

### Database Schema Updates
```sql
-- Add status update integration tables
CREATE TABLE status_sync_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id),
    resman_lead_id VARCHAR(100),
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    status_reason TEXT,
    updated_by VARCHAR(100), -- 'chatbot', 'resman_user', 'system'
    sync_direction VARCHAR(20), -- 'to_resman', 'from_resman'
    sync_status VARCHAR(50), -- 'pending', 'success', 'failed'
    context_data JSONB, -- Additional context for the status change
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE status_workflow_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id),
    trigger_status VARCHAR(50),
    target_status VARCHAR(50),
    conditions JSONB, -- Conditions that must be met
    actions JSONB, -- Actions to take when triggered
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE status_notification_preferences (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id),
    status_type VARCHAR(50),
    notification_method VARCHAR(50), -- 'email', 'sms', 'slack', 'in_app'
    recipients JSONB, -- Array of recipient identifiers
    template_id VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### API Endpoints to Create

#### 1. Status Update API
```typescript
// PUT /api/resman/leads/{lead_id}/status
interface UpdateStatusRequest {
  new_status: 'new' | 'contacted' | 'qualified' | 'tour_scheduled' | 'application' | 'leased' | 'lost';
  status_reason?: string;
  context_data?: {
    conversation_summary?: string;
    key_insights?: string[];
    next_steps?: string;
    urgency_level?: 'low' | 'medium' | 'high';
  };
  updated_by: 'chatbot' | 'resman_user' | 'system';
  trigger_notifications?: boolean;
}

interface UpdateStatusResponse {
  success: boolean;
  new_status: string;
  updated_at: string;
  notifications_sent: number;
  workflow_actions_triggered: string[];
}
```

#### 2. Status History API
```typescript
// GET /api/resman/leads/{lead_id}/status-history
interface StatusHistoryRequest {
  lead_id: string;
  date_from?: string;
  date_to?: string;
  limit?: number;
}

interface StatusHistoryResponse {
  status_changes: Array<{
    id: string;
    old_status: string;
    new_status: string;
    status_reason?: string;
    updated_by: string;
    context_data?: Record<string, any>;
    created_at: string;
  }>;
  total_count: number;
}
```

#### 3. Status Workflow API
```typescript
// POST /api/resman/status-workflows
interface CreateWorkflowRequest {
  property_id: string;
  trigger_status: string;
  target_status: string;
  conditions: {
    conversation_duration?: number; // minutes
    message_count?: number;
    lead_score?: number;
    time_of_day?: string;
  };
  actions: Array<{
    type: 'send_notification' | 'schedule_follow_up' | 'update_lead_score' | 'create_task';
    config: Record<string, any>;
  }>;
}

// GET /api/resman/status-workflows/{property_id}
interface GetWorkflowsResponse {
  workflows: Array<{
    id: string;
    trigger_status: string;
    target_status: string;
    conditions: Record<string, any>;
    actions: Array<Record<string, any>>;
    is_active: boolean;
  }>;
}
```

#### 4. Status Webhook Receiver
```typescript
// POST /api/resman/webhooks/status-update
interface StatusUpdateWebhook {
  lead_id: string;
  old_status: string;
  new_status: string;
  status_reason?: string;
  updated_by: string;
  updated_at: string;
  context_data?: Record<string, any>;
}
```

## Integration Points

### 1. Chatbot Status Update Logic
```typescript
// In ChatModal.tsx - when conversation reaches certain milestones
const handleStatusUpdate = async (newStatus: LeadStatus, context?: StatusContext) => {
  try {
    const statusData = {
      new_status: newStatus,
      status_reason: context?.reason,
      context_data: {
        conversation_summary: await generateConversationSummary(),
        key_insights: await extractKeyInsights(),
        next_steps: context?.nextSteps,
        urgency_level: context?.urgencyLevel
      },
      updated_by: 'chatbot'
    };
    
    // Update status in ResMan
    const result = await updateResManLeadStatus(conversation.resman_lead_id, statusData);
    
    // Log status change
    await logStatusChange({
      conversation_id: conversation.id,
      resman_lead_id: conversation.resman_lead_id,
      old_status: conversation.current_status,
      new_status: newStatus,
      status_reason: context?.reason,
      updated_by: 'chatbot',
      sync_direction: 'to_resman',
      sync_status: 'success',
      context_data: statusData.context_data
    });
    
    // Update local conversation status
    await updateConversationStatus(conversation.id, newStatus);
    
    // Trigger workflow actions
    await triggerStatusWorkflows(newStatus, context);
    
  } catch (error) {
    handleStatusUpdateError(error);
  }
};
```

### 2. Status Workflow Engine
```typescript
class StatusWorkflowEngine {
  async triggerStatusWorkflows(newStatus: string, context: StatusContext): Promise<void> {
    const workflows = await this.getActiveWorkflows(context.property_id);
    
    for (const workflow of workflows) {
      if (workflow.trigger_status === newStatus && 
          this.evaluateConditions(workflow.conditions, context)) {
        
        await this.executeWorkflowActions(workflow.actions, context);
      }
    }
  }
  
  private async executeWorkflowActions(actions: WorkflowAction[], context: StatusContext): Promise<void> {
    for (const action of actions) {
      switch (action.type) {
        case 'send_notification':
          await this.sendStatusNotification(action.config, context);
          break;
        case 'schedule_follow_up':
          await this.scheduleFollowUp(action.config, context);
          break;
        case 'update_lead_score':
          await this.updateLeadScore(action.config, context);
          break;
        case 'create_task':
          await this.createTask(action.config, context);
          break;
      }
    }
  }
  
  private async sendStatusNotification(config: NotificationConfig, context: StatusContext): Promise<void> {
    const notification = {
      type: 'status_update',
      recipients: config.recipients,
      template: config.template_id,
      data: {
        lead_name: context.lead_name,
        old_status: context.old_status,
        new_status: context.new_status,
        status_reason: context.status_reason,
        conversation_summary: context.conversation_summary,
        urgency_level: context.urgency_level
      }
    };
    
    await this.notificationService.send(notification);
  }
}
```

### 3. ResMan Status Webhook Handler
```typescript
// Handle ResMan status changes
const handleResManStatusUpdate = async (webhookData: StatusUpdateWebhook) => {
  const { lead_id, old_status, new_status, status_reason, updated_by, context_data } = webhookData;
  
  try {
    // Find conversation by ResMan lead ID
    const conversation = await getConversationByResManLeadId(lead_id);
    
    if (conversation) {
      // Update conversation status
      await updateConversationStatus(conversation.id, new_status);
      
      // Log status change
      await logStatusChange({
        conversation_id: conversation.id,
        resman_lead_id: lead_id,
        old_status,
        new_status,
        status_reason,
        updated_by,
        sync_direction: 'from_resman',
        sync_status: 'success',
        context_data
      });
      
      // Update conversation context with new status
      await updateConversationContext(conversation.id, {
        current_status: new_status,
        status_reason,
        last_status_update: new Date(),
        status_updated_by: updated_by
      });
      
      // Send prospect notification if appropriate
      if (shouldNotifyProspect(new_status)) {
        await sendProspectStatusNotification(conversation, new_status, status_reason);
      }
    }
    
  } catch (error) {
    console.error('Error processing ResMan status update:', error);
    await logStatusSyncError(lead_id, error);
  }
};
```

## Status Mapping

### DomIQ → ResMan Status Mapping
```typescript
interface StatusMapper {
  mapChatbotStatusToResMan(chatbotStatus: ChatbotStatus): ResManStatus {
    const statusMap = {
      'initial_contact': 'new',
      'engaged': 'contacted',
      'qualified': 'qualified',
      'tour_booked': 'tour_scheduled',
      'application_started': 'application',
      'leased': 'leased',
      'lost': 'lost',
      'follow_up_needed': 'contacted'
    };
    
    return statusMap[chatbotStatus] || 'new';
  }
  
  mapResManStatusToChatbot(resmanStatus: ResManStatus): ChatbotStatus {
    const reverseMap = {
      'new': 'initial_contact',
      'contacted': 'engaged',
      'qualified': 'qualified',
      'tour_scheduled': 'tour_booked',
      'application': 'application_started',
      'leased': 'leased',
      'lost': 'lost'
    };
    
    return reverseMap[resmanStatus] || 'initial_contact';
  }
}
```

## Status Validation

### Status Transition Rules
```typescript
class StatusValidationService {
  private validTransitions = {
    'new': ['contacted', 'qualified', 'lost'],
    'contacted': ['qualified', 'tour_scheduled', 'lost'],
    'qualified': ['tour_scheduled', 'application', 'lost'],
    'tour_scheduled': ['application', 'leased', 'lost'],
    'application': ['leased', 'lost'],
    'leased': [], // Terminal state
    'lost': [] // Terminal state
  };
  
  validateStatusTransition(oldStatus: string, newStatus: string): boolean {
    const allowedTransitions = this.validTransitions[oldStatus] || [];
    return allowedTransitions.includes(newStatus);
  }
  
  getRequiredConditions(newStatus: string): StatusConditions {
    const conditions: StatusConditions = {};
    
    switch (newStatus) {
      case 'qualified':
        conditions.minLeadScore = 70;
        conditions.requiredFields = ['email', 'phone'];
        break;
      case 'tour_scheduled':
        conditions.requiredFields = ['email', 'phone', 'tour_datetime'];
        break;
      case 'application':
        conditions.requiredFields = ['email', 'phone', 'income_verification'];
        break;
    }
    
    return conditions;
  }
}
```

## Notification System

### Status Notification Templates
```typescript
interface StatusNotificationTemplates {
  'new_to_contacted': {
    subject: 'New Lead Contacted - {lead_name}',
    body: 'A new lead has been contacted via chatbot. Lead score: {lead_score}',
    recipients: ['property_manager', 'leasing_agent']
  },
  
  'contacted_to_qualified': {
    subject: 'Lead Qualified - {lead_name}',
    body: 'Lead has been qualified and is ready for tour scheduling.',
    recipients: ['leasing_agent']
  },
  
  'qualified_to_tour_scheduled': {
    subject: 'Tour Scheduled - {lead_name}',
    body: 'Tour has been scheduled for {tour_datetime}. Please prepare.',
    recipients: ['leasing_agent', 'property_manager']
  },
  
  'tour_scheduled_to_application': {
    subject: 'Application Started - {lead_name}',
    body: 'Lead has started the application process. Follow up required.',
    recipients: ['leasing_agent']
  },
  
  'application_to_leased': {
    subject: 'Lead Leased - {lead_name}',
    body: 'Congratulations! Lead has successfully leased.',
    recipients: ['property_manager', 'leasing_agent', 'management']
  }
}
```

### Notification Service
```typescript
class StatusNotificationService {
  async sendStatusNotification(statusChange: StatusChange): Promise<void> {
    const template = this.getNotificationTemplate(statusChange.old_status, statusChange.new_status);
    
    if (!template) return;
    
    const notificationData = {
      subject: this.interpolateTemplate(template.subject, statusChange),
      body: this.interpolateTemplate(template.body, statusChange),
      recipients: await this.resolveRecipients(template.recipients, statusChange),
      priority: this.getNotificationPriority(statusChange.new_status)
    };
    
    await this.notificationService.send(notificationData);
  }
  
  private getNotificationPriority(status: string): 'low' | 'medium' | 'high' | 'urgent' {
    const priorities = {
      'new': 'medium',
      'contacted': 'medium',
      'qualified': 'high',
      'tour_scheduled': 'high',
      'application': 'urgent',
      'leased': 'high',
      'lost': 'low'
    };
    
    return priorities[status] || 'medium';
  }
}
```

## Monitoring & Analytics

### Status Analytics
```typescript
interface StatusAnalytics {
  getStatusMetrics(propertyId: string, dateRange: DateRange): Promise<{
    total_status_changes: number;
    average_time_in_status: Record<string, number>;
    status_conversion_rates: Record<string, number>;
    most_common_transitions: Array<{from: string, to: string, count: number}>;
  }>;
  
  getStatusTrends(propertyId: string): Promise<{
    daily_status_changes: Array<{date: string, count: number}>;
    status_distribution: Record<string, number>;
    conversion_funnel: Array<{status: string, count: number, conversion_rate: number}>;
  }>;
  
  getStatusPerformanceMetrics(propertyId: string): Promise<{
    average_response_time: number;
    status_update_accuracy: number;
    workflow_trigger_rate: number;
    notification_delivery_rate: number;
  }>;
}
```

## Testing Strategy

### Unit Tests
- Status validation logic
- Status mapping functions
- Workflow condition evaluation
- Notification template processing

### Integration Tests
- End-to-end status update flow
- Webhook processing
- Workflow execution
- Notification delivery

### User Acceptance Tests
- Status update workflow
- Notification delivery
- Workflow automation
- Status history tracking

## Deployment Checklist

### Pre-deployment
- [ ] ResMan status API access configured
- [ ] Status workflow rules defined
- [ ] Webhook endpoints registered
- [ ] Notification templates created
- [ ] Status validation rules configured

### Post-deployment
- [ ] Verify status update functionality
- [ ] Test workflow automation
- [ ] Validate notification delivery
- [ ] Check status history tracking
- [ ] Monitor status sync performance

## Success Criteria
- [ ] 99%+ status sync success rate
- [ ] <10 second status update time
- [ ] 100% notification delivery rate
- [ ] Real-time status synchronization
- [ ] Complete status history preservation
- [ ] Zero data loss during status operations 