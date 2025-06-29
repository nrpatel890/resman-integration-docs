# Lead Sync Frontend Component: DomIQ ↔ ResMan

## Overview
This file describes the frontend architecture, UI components, and state management for bidirectional lead sync between DomIQ and ResMan. It covers lead creation, sync status, conflict resolution, real-time updates, and error handling.

---

## 1. Lead Creation & Sync UI

### Lead Creation Modal/Drawer
```typescript
// UI for creating a new lead (from chatbot or admin panel)
interface LeadCreationModalProps {
  isOpen: boolean;
  onClose: () => void;
  onSubmit: (leadData: LeadData) => Promise<void>;
  loading: boolean;
  error?: string;
}

// Usage: <LeadCreationModal ... />
```

### Lead Sync Status Indicator
```typescript
// Shows sync status for each lead (conversation)
interface LeadSyncStatusProps {
  resmanSyncStatus: 'not_synced' | 'syncing' | 'synced' | 'failed' | 'conflict';
  lastSyncAt?: string;
  onRetry?: () => void;
  onViewLog?: () => void;
}

// Usage: <LeadSyncStatus ... />
```

### Lead Sync Log Viewer
```typescript
// Shows sync log history for a lead
interface LeadSyncLogViewerProps {
  conversationId: string;
  logs: LeadSyncLog[];
  loading: boolean;
  onRefresh: () => void;
}

interface LeadSyncLog {
  id: string;
  syncDirection: string;
  syncStatus: string;
  syncType: string;
  createdAt: string;
  errorMessage?: string;
  conflictResolution?: string;
}
```

---

## 2. Conflict Resolution UI

### Conflict Resolution Modal
```typescript
// UI for resolving sync conflicts
interface ConflictResolutionModalProps {
  isOpen: boolean;
  conflict: LeadSyncConflict | null;
  onResolve: (resolution: ConflictResolution) => Promise<void>;
  onEscalate: () => void;
  loading: boolean;
}

interface LeadSyncConflict {
  id: string;
  field: string;
  resmanValue: any;
  domiqValue: any;
  conflictType: string;
  conflictSeverity: string;
  resolutionStatus: string;
}

interface ConflictResolution {
  resolution: 'resman_wins' | 'domiq_wins' | 'manual_review' | 'merged';
  finalValue: any;
  reason: string;
}
```

### Conflict Notification Banner
```typescript
// Banner to alert user of unresolved conflicts
interface ConflictNotificationBannerProps {
  conflicts: LeadSyncConflict[];
  onClick: (conflictId: string) => void;
}
```

---

## 3. Real-time Sync & Monitoring

### Real-time Sync Indicator
```typescript
// Shows real-time sync connection status
interface RealTimeSyncIndicatorProps {
  connected: boolean;
  lastPing: string;
  onReconnect: () => void;
}
```

### Sync Metrics Dashboard Widget
```typescript
// Widget for displaying sync metrics (success rate, latency, etc.)
interface SyncMetricsWidgetProps {
  metrics: SyncMetrics;
  loading: boolean;
  onRefresh: () => void;
}

interface SyncMetrics {
  syncSuccessRate: number;
  avgSyncLatency: number;
  conflictRate: number;
  failedSyncs: number;
  pendingQueue: number;
}
```

---

## 4. State Management & Hooks

### Lead Sync State Slice (Redux or Zustand)
```typescript
interface LeadSyncState {
  leads: Lead[];
  syncLogs: Record<string, LeadSyncLog[]>;
  conflicts: LeadSyncConflict[];
  metrics: SyncMetrics;
  realTimeConnected: boolean;
  loading: boolean;
  error?: string;
}
```

### Lead Sync Hooks
```typescript
// Hook to fetch and sync leads
function useLeadSync() {
  // fetchLeads, pushLead, pullLeads, updateLeadStatus, mergeLeads, etc.
}

// Hook for real-time sync connection
function useRealTimeLeadSync() {
  // connect, disconnect, subscribe, handleMessage, etc.
}

// Hook for conflict resolution
function useLeadSyncConflicts() {
  // fetchConflicts, resolveConflict, escalateConflict, etc.
}
```

---

## 5. Error Handling & Alerts

### Sync Error Alert
```typescript
// Alert for sync errors
interface SyncErrorAlertProps {
  error: string;
  onRetry?: () => void;
  onClose: () => void;
}
```

### Alerting Integration
- Integrate with notification system (toast, snackbar, or modal)
- Show alerts for:
  - Sync failures
  - Conflict detected
  - Real-time connection lost
  - Manual review required

---

## 6. Integration Points

- **ChatModal.tsx**: Trigger lead creation and sync after qualification
- **LeadFunnel/LeadsTable**: Show sync status and allow manual retry
- **ConflictResolutionModal**: Allow managers to resolve or escalate conflicts
- **RealTimeSyncIndicator**: Show live sync status in dashboard header/footer
- **SyncMetricsWidget**: Display sync health in admin dashboard

---

## 7. Testing & QA

- Unit tests for all hooks and UI components
- Integration tests for lead creation, sync, and conflict resolution flows
- Mock WebSocket for real-time sync tests
- UI tests for error and alert handling

---

This frontend component provides a complete UI and state management solution for bidirectional lead sync, conflict resolution, and real-time monitoring between DomIQ and ResMan. 