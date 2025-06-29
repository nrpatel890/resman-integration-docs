# Status Update Backend Component: DomIQ ↔ ResMan

## Backend API Endpoints

### 1. Status Update API
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

### 2. Status History API
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

### 3. Status Workflow API
```typescript
// POST /api/resman/status-workflows
interface CreateWorkflowRequest {
  property_id: string;
  trigger_status: string;
  target_status: string;
  conditions: {
    conversation_duration?: number;
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

### 4. Status Webhook Receiver
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

## Services & Logic

### Status Update Service
- handleStatusUpdate(newStatus, context)
- updateResManLeadStatus(lead_id, statusData)
- logStatusChange(logData)
- updateConversationStatus(conversationId, newStatus)
- triggerStatusWorkflows(newStatus, context)

### Status Workflow Engine
- getActiveWorkflows(propertyId)
- evaluateConditions(conditions, context)
- executeWorkflowActions(actions, context)
- sendStatusNotification(config, context)
- scheduleFollowUp(config, context)
- updateLeadScore(config, context)
- createTask(config, context)

### Status Mapping
- mapChatbotStatusToResMan(chatbotStatus)
- mapResManStatusToChatbot(resmanStatus)

### Status Validation
- validateStatusTransition(oldStatus, newStatus)
- getRequiredConditions(newStatus)

### Notification Service
- sendStatusNotification(statusChange)
- getNotificationTemplate(oldStatus, newStatus)
- interpolateTemplate(template, statusChange)
- resolveRecipients(recipients, statusChange)
- getNotificationPriority(status)

### Monitoring & Analytics
- getStatusMetrics(propertyId, dateRange)
- getStatusTrends(propertyId)
- getStatusPerformanceMetrics(propertyId)

## Testing & QA
- Unit tests for validation, mapping, workflow, and notifications
- Integration tests for status update, webhook, and workflow execution 