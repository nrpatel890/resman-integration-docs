# Contact Management Backend Component: DomIQ ↔ ResMan

## Backend API Endpoints

### 1. Contact Creation/Update API
```typescript
// POST /api/resman/contacts
interface CreateContactRequest {
  contact_data: {
    first_name: string;
    last_name: string;
    email?: string;
    phone?: string;
    property_id: string;
    source: 'chatbot';
    initial_interaction: {
      type: 'chat' | 'email' | 'phone';
      timestamp: string;
      message?: string;
    };
    preferences?: {
      communication_method: 'email' | 'phone' | 'text';
      contact_frequency: 'immediate' | 'daily' | 'weekly';
      opt_out: boolean;
    };
  };
}

interface CreateContactResponse {
  contact_id: string;
  is_new_contact: boolean;
  matched_contact_id?: string;
  enrichment_data?: Record<string, any>;
}
```

### 2. Contact Search API
```typescript
// GET /api/resman/contacts/search
interface ContactSearchRequest {
  email?: string;
  phone?: string;
  name?: string;
  property_id?: string;
  include_inactive?: boolean;
}

interface ContactSearchResponse {
  contacts: Array<{
    contact_id: string;
    first_name: string;
    last_name: string;
    email?: string;
    phone?: string;
    status: string;
    last_interaction: string;
    lead_score?: number;
  }>;
  total_count: number;
}
```

### 3. Contact Update API
```typescript
// PUT /api/resman/contacts/{contact_id}
interface UpdateContactRequest {
  contact_data: {
    first_name?: string;
    last_name?: string;
    email?: string;
    phone?: string;
    status?: string;
    preferences?: ContactPreferences;
    notes?: string;
  };
  update_source: 'chatbot' | 'resman' | 'manual';
}
```

### 4. Contact Webhook Receiver
```typescript
// POST /api/resman/webhooks/contact-update
interface ContactUpdateWebhook {
  contact_id: string;
  updated_fields: Record<string, any>;
  update_source: string;
  timestamp: string;
  user_id?: string;
}
```

## Services & Logic

### Contact Matching Service
- findMatchingContact(contactData)
- calculateMatchConfidence(resmanContact, chatbotContact)
- searchResManContact(criteria)

### Contact Enrichment Service
- enrichContactWithConversationData(contactId, conversationId)
- generateConversationSummary(messages)
- extractKeyTopics(messages)
- analyzeSentiment(messages)
- extractPreferences(messages)
- calculateLeadScore(conversation)

### Data Transformation
- mapChatbotContactToResMan(contactData)
- mapResManContactToChatbot(resmanContact)

### Contact Deduplication Service
- handleDuplicateContact(newContact)
- mergeContacts(primaryContactId, secondaryContactId)
- updateContactReferences(secondaryContactId, primaryContactId)
- markContactAsMerged(secondaryContactId, primaryContactId)

### Communication Preference Service
- updateContactPreferences(contactId, preferences)
- handleOptOut(contactId, optOutType)
- logPreferenceChange(contactId, preferences)

### Monitoring & Analytics
- getContactMetrics(propertyId, dateRange)
- getContactTrends(propertyId)
- getContactEnrichmentMetrics(propertyId)

## Testing & QA
- Unit tests for matching, transformation, deduplication, and preferences
- Integration tests for contact creation, update, webhook, and enrichment
- User acceptance tests for contact workflow and preference management 