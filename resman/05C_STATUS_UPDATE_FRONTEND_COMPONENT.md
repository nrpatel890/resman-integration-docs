# Status Update Frontend Component: DomIQ ↔ ResMan

## Overview
This file describes the frontend architecture, UI components, and state management for real-time status update sync and workflow between DomIQ and ResMan.

---

## 1. Status Update UI

### Status Update Modal/Drawer
```typescript
// UI for updating lead status
interface StatusUpdateModalProps {
  isOpen: boolean;
  currentStatus: string;
  onUpdate: (newStatus: string, reason?: string, context?: StatusContext) => Promise<void>;
  loading: boolean;
  error?: string;
  allowedTransitions: string[];
}

interface StatusContext {
  conversationSummary?: string;
  keyInsights?: string[];
  nextSteps?: string;
  urgencyLevel?: 'low' | 'medium' | 'high';
}
```

### Status History Viewer
```typescript
// UI for viewing status change history
interface StatusHistoryViewerProps {
  leadId: string;
  statusChanges: StatusChangeLog[];
  loading: boolean;
  onRefresh: () => void;
}

interface StatusChangeLog {
  id: string;
  oldStatus: string;
  newStatus: string;
  statusReason?: string;
  updatedBy: string;
  contextData?: Record<string, any>;
  createdAt: string;
}
```

---

## 2. Workflow & Automation UI

### Workflow Rule Editor
```typescript
// UI for creating/editing status workflow rules
interface WorkflowRuleEditorProps {
  propertyId: string;
  workflows: WorkflowRule[];
  onCreate: (rule: WorkflowRule) => Promise<void>;
  onUpdate: (rule: WorkflowRule) => Promise<void>;
  onDelete: (ruleId: string) => Promise<void>;
  loading: boolean;
}

interface WorkflowRule {
  id?: string;
  triggerStatus: string;
  targetStatus: string;
  conditions: Record<string, any>;
  actions: Array<Record<string, any>>;
  isActive: boolean;
}
```

### Workflow Action Log Viewer
```typescript
// UI for viewing workflow action logs
interface WorkflowActionLogViewerProps {
  propertyId: string;
  logs: WorkflowActionLog[];
  loading: boolean;
  onRefresh: () => void;
}

interface WorkflowActionLog {
  id: string;
  workflowId: string;
  actionType: string;
  status: string;
  triggeredAt: string;
  details?: Record<string, any>;
}
```

---

## 3. Notification & Alerting UI

### Status Notification Banner
```typescript
// Banner to notify staff of status changes
interface StatusNotificationBannerProps {
  notifications: StatusNotification[];
  onAcknowledge: (notificationId: string) => void;
}

interface StatusNotification {
  id: string;
  leadId: string;
  oldStatus: string;
  newStatus: string;
  subject: string;
  body: string;
  recipients: string[];
  createdAt: string;
  priority: 'low' | 'medium' | 'high' | 'urgent';
}
```

### Prospect Notification Modal
```typescript
// Modal to show prospect status updates
interface ProspectNotificationModalProps {
  isOpen: boolean;
  notification: StatusNotification;
  onClose: () => void;
}
```

---

## 4. State Management & Hooks

### Status State Slice (Redux or Zustand)
```typescript
interface StatusState {
  statusHistory: Record<string, StatusChangeLog[]>;
  workflows: WorkflowRule[];
  notifications: StatusNotification[];
  workflowLogs: WorkflowActionLog[];
  loading: boolean;
  error?: string;
}
```

### Status Update Hooks
```typescript
// Hook to manage status updates
function useStatusUpdate() {
  // updateStatus, fetchStatusHistory, etc.
}

// Hook for workflow management
function useStatusWorkflow() {
  // fetchWorkflows, createWorkflow, updateWorkflow, deleteWorkflow, etc.
}

// Hook for notifications
function useStatusNotifications() {
  // fetchNotifications, acknowledgeNotification, etc.
}
```

---

## 5. Error Handling & Alerts

### Status Error Alert
```typescript
// Alert for status sync/errors
interface StatusErrorAlertProps {
  error: string;
  onRetry?: () => void;
  onClose: () => void;
}
```

### Alerting Integration
- Integrate with notification system (toast, snackbar, or modal)
- Show alerts for:
  - Status update failures
  - Workflow execution errors
  - Notification delivery issues

---

## 6. Integration Points

- **ChatModal.tsx**: Trigger status updates and show status history
- **StatusUpdateModal**: Allow staff to update lead status
- **WorkflowRuleEditor**: Manage status automation rules
- **StatusNotificationBanner**: Notify staff of status changes
- **ProspectNotificationModal**: Show status updates to prospects

---

## 7. Monitoring & Analytics

### Status Analytics Widget
```typescript
// Widget for displaying status metrics
interface StatusAnalyticsWidgetProps {
  metrics: StatusMetrics;
  loading: boolean;
  onRefresh: () => void;
}

interface StatusMetrics {
  totalStatusChanges: number;
  averageTimeInStatus: Record<string, number>;
  statusConversionRates: Record<string, number>;
  mostCommonTransitions: Array<{from: string, to: string, count: number}>;
}
```

---

## 8. Testing & QA

- Unit tests for all hooks and UI components
- Integration tests for status update, workflow, and notification flows
- UI tests for error and alert handling

---

This frontend component provides a complete UI and state management solution for real-time status update sync, workflow automation, and notification management between DomIQ and ResMan. 