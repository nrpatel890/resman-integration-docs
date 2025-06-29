# Contact Management Frontend Component: DomIQ ↔ ResMan

## Overview
This file describes the frontend architecture, UI components, and state management for unified contact management and sync between DomIQ and ResMan.

---

## 1. Contact Creation & Search UI

### Contact Creation Modal/Drawer
```typescript
// UI for creating or updating a contact
interface ContactCreationModalProps {
  isOpen: boolean;
  onClose: () => void;
  onSubmit: (contactData: ContactData) => Promise<void>;
  loading: boolean;
  error?: string;
}

interface ContactData {
  firstName: string;
  lastName: string;
  email?: string;
  phone?: string;
  propertyId: string;
  preferences?: ContactPreferences;
}

interface ContactPreferences {
  communicationMethod?: 'email' | 'phone' | 'text';
  contactFrequency?: 'immediate' | 'daily' | 'weekly';
  optOut?: boolean;
}
```

### Contact Search Bar & Results
```typescript
// UI for searching contacts
interface ContactSearchBarProps {
  onSearch: (query: ContactSearchQuery) => void;
  loading: boolean;
}

interface ContactSearchQuery {
  email?: string;
  phone?: string;
  name?: string;
  propertyId?: string;
  includeInactive?: boolean;
}

interface ContactSearchResultsProps {
  contacts: ContactSummary[];
  loading: boolean;
  onSelect: (contactId: string) => void;
}

interface ContactSummary {
  contactId: string;
  firstName: string;
  lastName: string;
  email?: string;
  phone?: string;
  status: string;
  lastInteraction: string;
  leadScore?: number;
}
```

---

## 2. Contact Profile & Enrichment

### Contact Profile View
```typescript
// UI for displaying/enriching a contact profile
interface ContactProfileProps {
  contactId: string;
  contact: ContactDetails;
  enrichmentData?: ContactEnrichmentData;
  onUpdate: (updates: Partial<ContactDetails>) => Promise<void>;
  loading: boolean;
  error?: string;
}

interface ContactDetails {
  contactId: string;
  firstName: string;
  lastName: string;
  email?: string;
  phone?: string;
  status: string;
  preferences?: ContactPreferences;
  notes?: string;
  enrichmentData?: ContactEnrichmentData;
}

interface ContactEnrichmentData {
  conversationSummary?: string;
  keyTopics?: string[];
  sentiment?: string;
  interactionPatterns?: string;
  preferencesExtracted?: ContactPreferences;
  leadScore?: number;
  lastInteraction?: string;
}
```

### Enrichment Log Viewer
```typescript
// UI for viewing enrichment history
interface EnrichmentLogViewerProps {
  contactId: string;
  logs: EnrichmentLog[];
  loading: boolean;
  onRefresh: () => void;
}

interface EnrichmentLog {
  id: string;
  enrichmentType: string;
  enrichedData: any;
  source: string;
  createdAt: string;
}
```

---

## 3. Deduplication & Matching

### Duplicate Contact Alert
```typescript
// Banner/modal to alert user of potential duplicates
interface DuplicateContactAlertProps {
  potentialMatches: ContactSummary[];
  onMerge: (primaryId: string, secondaryId: string) => Promise<void>;
  onManualReview: () => void;
  onDismiss: () => void;
}
```

### Contact Merge Modal
```typescript
// UI for merging two contacts
interface ContactMergeModalProps {
  primaryContact: ContactDetails;
  secondaryContact: ContactDetails;
  onMerge: () => Promise<void>;
  onCancel: () => void;
  loading: boolean;
}
```

---

## 4. Preferences & Opt-Out

### Preferences Editor
```typescript
// UI for editing contact preferences
interface PreferencesEditorProps {
  contactId: string;
  preferences: ContactPreferences;
  onSave: (prefs: ContactPreferences) => Promise<void>;
  loading: boolean;
  error?: string;
}
```

### Opt-Out Banner
```typescript
// Banner to show opt-out status and allow opt-in/out
interface OptOutBannerProps {
  contactId: string;
  preferences: ContactPreferences;
  onOptOut: (type: 'email' | 'phone' | 'all') => Promise<void>;
  onOptIn: (type: 'email' | 'phone' | 'all') => Promise<void>;
  loading: boolean;
}
```

---

## 5. State Management & Hooks

### Contact State Slice (Redux or Zustand)
```typescript
interface ContactState {
  contacts: ContactSummary[];
  selectedContact?: ContactDetails;
  enrichmentLogs: Record<string, EnrichmentLog[]>;
  duplicateMatches: ContactSummary[];
  loading: boolean;
  error?: string;
}
```

### Contact Management Hooks
```typescript
// Hook to manage contacts
function useContactManagement() {
  // createContact, searchContacts, updateContact, fetchContactProfile, etc.
}

// Hook for enrichment
function useContactEnrichment() {
  // fetchEnrichmentLogs, enrichContact, etc.
}

// Hook for deduplication
function useContactDeduplication() {
  // findDuplicates, mergeContacts, manualReview, etc.
}

// Hook for preferences
function useContactPreferences() {
  // updatePreferences, handleOptOut, etc.
}
```

---

## 6. Error Handling & Alerts

### Contact Error Alert
```typescript
// Alert for contact sync/errors
interface ContactErrorAlertProps {
  error: string;
  onRetry?: () => void;
  onClose: () => void;
}
```

### Alerting Integration
- Integrate with notification system (toast, snackbar, or modal)
- Show alerts for:
  - Duplicate detected
  - Merge required
  - Enrichment failed
  - Preference update failed

---

## 7. Integration Points

- **ChatModal.tsx**: Trigger contact creation and enrichment
- **ContactSearchBar**: Search and select contacts
- **ContactProfile**: View and update contact details
- **DuplicateContactAlert**: Handle deduplication and merging
- **PreferencesEditor**: Edit and save communication preferences

---

## 8. Monitoring & Analytics

### Contact Analytics Widget
```typescript
// Widget for displaying contact metrics
interface ContactAnalyticsWidgetProps {
  metrics: ContactMetrics;
  loading: boolean;
  onRefresh: () => void;
}

interface ContactMetrics {
  totalContacts: number;
  newContacts: number;
  activeContacts: number;
  contactGrowthRate: number;
  averageLeadScore: number;
  conversionRate: number;
}
```

---

## 9. Testing & QA

- Unit tests for all hooks and UI components
- Integration tests for contact creation, search, enrichment, and deduplication flows
- UI tests for error and alert handling

---

This frontend component provides a complete UI and state management solution for contact creation, enrichment, deduplication, and preference management between DomIQ and ResMan. 