# Contact Management Integration: DomIQ ↔ ResMan

## Feature Overview
Unified contact profile management between DomIQ AI chatbot and ResMan contact database.

## What It Delivers for On-site Teams
- **Unified Contact Profiles**: Single source of truth for all prospect information
- **Real-time Contact Updates**: Changes sync across both systems instantly
- **Contact History**: Complete interaction history from chatbot conversations
- **Duplicate Prevention**: Smart contact matching and merging
- **Contact Enrichment**: Enhanced profiles with chatbot interaction data
- **Communication Preferences**: Centralized contact preferences and opt-outs

## Data Flow Architecture

### 1. Contact Creation Flow (DomIQ → ResMan)
```
Chatbot Contact → Contact Matching → ResMan Contact Creation/Update → Profile Enrichment
```

### 2. Contact Update Flow (ResMan → DomIQ)
```
ResMan Contact Change → Webhook → Chatbot Context Update → Conversation Enhancement
```

## Implementation Requirements

### Database Schema Updates
```sql
-- Add contact management integration tables
CREATE TABLE contact_sync (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES "user"(id),
    resman_contact_id VARCHAR(100),
    sync_status VARCHAR(50), -- 'pending', 'success', 'failed', 'merged'
    last_sync_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE contact_matching_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id),
    matching_criteria JSONB, -- Email, phone, name combinations
    merge_strategy VARCHAR(50), -- 'resman_wins', 'domiq_wins', 'manual_review'
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE contact_enrichment_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    contact_id UUID REFERENCES "user"(id),
    enrichment_type VARCHAR(50), -- 'conversation_data', 'preferences', 'interaction_history'
    enriched_data JSONB,
    source VARCHAR(50), -- 'chatbot', 'resman', 'manual'
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### API Endpoints to Create

#### 1. Contact Creation/Update API
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

#### 2. Contact Search API
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

#### 3. Contact Update API
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

#### 4. Contact Webhook Receiver
```typescript
// POST /api/resman/webhooks/contact-update
interface ContactUpdateWebhook {
  contact_id: string;
  updated_fields: Record<string, any>;
  update_source: string;
  timestamp: string;
  user_id?: string; // ResMan user who made the change
}
```

## Integration Points

### 1. Chatbot Contact Creation
```typescript
// In ChatModal.tsx - when user provides contact information
const handleContactSubmission = async (contactData: ContactData) => {
  try {
    // Search for existing contact in ResMan
    const existingContact = await searchResManContact(contactData);
    
    if (existingContact) {
      // Update existing contact with new interaction data
      await updateResManContact(existingContact.id, {
        ...contactData,
        last_interaction: new Date(),
        interaction_count: existingContact.interaction_count + 1
      });
      
      // Link to existing contact
      await linkContactToConversation(conversationId, existingContact.id);
      
    } else {
      // Create new contact in ResMan
      const newContact = await createResManContact({
        ...contactData,
        source: 'chatbot',
        initial_interaction: {
          type: 'chat',
          timestamp: new Date(),
          message: getInitialMessage()
        }
      });
      
      // Link to new contact
      await linkContactToConversation(conversationId, newContact.id);
    }
    
    // Enrich contact with conversation data
    await enrichContactWithConversationData(contactData, conversationId);
    
  } catch (error) {
    handleContactCreationError(error);
  }
};
```

### 2. Contact Matching Service
```typescript
class ContactMatchingService {
  async findMatchingContact(contactData: ContactData): Promise<ResManContact | null> {
    const matchingCriteria = [
      { email: contactData.email },
      { phone: contactData.phone },
      { 
        first_name: contactData.first_name,
        last_name: contactData.last_name 
      }
    ];
    
    for (const criteria of matchingCriteria) {
      const match = await this.searchResManContact(criteria);
      if (match && this.calculateMatchConfidence(match, contactData) > 0.8) {
        return match;
      }
    }
    
    return null;
  }
  
  private calculateMatchConfidence(resmanContact: ResManContact, chatbotContact: ContactData): number {
    let confidence = 0;
    
    // Email match: 40% weight
    if (resmanContact.email === chatbotContact.email) {
      confidence += 0.4;
    }
    
    // Phone match: 30% weight
    if (resmanContact.phone === chatbotContact.phone) {
      confidence += 0.3;
    }
    
    // Name match: 30% weight
    if (this.normalizeName(resmanContact.first_name) === this.normalizeName(chatbotContact.first_name) &&
        this.normalizeName(resmanContact.last_name) === this.normalizeName(chatbotContact.last_name)) {
      confidence += 0.3;
    }
    
    return confidence;
  }
}
```

### 3. Contact Enrichment Service
```typescript
class ContactEnrichmentService {
  async enrichContactWithConversationData(contactId: string, conversationId: string): Promise<void> {
    const conversation = await getConversationById(conversationId);
    const messages = await getConversationMessages(conversationId);
    
    const enrichmentData = {
      conversation_summary: await this.generateConversationSummary(messages),
      key_topics: await this.extractKeyTopics(messages),
      sentiment_analysis: await this.analyzeSentiment(messages),
      interaction_patterns: this.analyzeInteractionPatterns(messages),
      preferences_extracted: await this.extractPreferences(messages),
      lead_score: this.calculateLeadScore(conversation),
      last_interaction: conversation.end_time || new Date()
    };
    
    await this.updateResManContact(contactId, {
      enrichment_data: enrichmentData,
      last_enriched: new Date()
    });
  }
  
  private async extractPreferences(messages: Message[]): Promise<ContactPreferences> {
    const preferences: ContactPreferences = {};
    
    for (const message of messages) {
      if (message.sender === 'user') {
        // Extract move-in date preferences
        const moveInMatch = message.content.match(/move.?in.*?(\d{1,2}\/\d{1,2}\/\d{4})/i);
        if (moveInMatch) {
          preferences.move_in_date = moveInMatch[1];
        }
        
        // Extract budget preferences
        const budgetMatch = message.content.match(/\$?(\d{1,3}(?:,\d{3})*)/g);
        if (budgetMatch && budgetMatch.length >= 2) {
          preferences.budget_range = {
            min: parseInt(budgetMatch[0].replace(/[$,]/g, '')),
            max: parseInt(budgetMatch[1].replace(/[$,]/g, ''))
          };
        }
        
        // Extract unit type preferences
        const unitTypes = ['studio', '1 bedroom', '2 bedroom', '3 bedroom'];
        for (const unitType of unitTypes) {
          if (message.content.toLowerCase().includes(unitType)) {
            preferences.unit_preferences = preferences.unit_preferences || [];
            preferences.unit_preferences.push(unitType);
          }
        }
      }
    }
    
    return preferences;
  }
}
```

## Data Transformation

### DomIQ → ResMan Contact Mapping
```typescript
interface ContactDataMapper {
  mapChatbotContactToResMan(contactData: ChatbotContactData): ResManContact {
    return {
      first_name: contactData.first_name,
      last_name: contactData.last_name,
      email: contactData.email,
      phone: contactData.phone,
      source: 'chatbot',
      lead_source: 'website_chat',
      initial_contact_date: contactData.first_interaction,
      status: 'new',
      communication_preferences: {
        preferred_method: contactData.preferences?.communication_method || 'email',
        contact_frequency: contactData.preferences?.contact_frequency || 'immediate',
        opt_out: contactData.preferences?.opt_out || false
      },
      notes: this.formatContactNotes(contactData)
    };
  }
  
  private formatContactNotes(contactData: ChatbotContactData): string {
    return `Contact created via DomIQ chatbot on ${new Date().toLocaleDateString()}.
Initial message: "${contactData.initial_message}"
Lead score: ${contactData.lead_score}
Property interest: ${contactData.property_name}`;
  }
}
```

### ResMan → DomIQ Contact Mapping
```typescript
interface ResManToDomIQMapper {
  mapResManContactToChatbot(resmanContact: ResManContact): ChatbotContactContext {
    return {
      contact_id: resmanContact.id,
      full_name: `${resmanContact.first_name} ${resmanContact.last_name}`,
      email: resmanContact.email,
      phone: resmanContact.phone,
      status: resmanContact.status,
      lead_score: resmanContact.lead_score,
      last_interaction: resmanContact.last_interaction,
      preferences: resmanContact.communication_preferences,
      notes: resmanContact.notes,
      enrichment_data: resmanContact.enrichment_data
    };
  }
}
```

## Duplicate Prevention

### Contact Deduplication Service
```typescript
class ContactDeduplicationService {
  async handleDuplicateContact(newContact: ContactData): Promise<DeduplicationResult> {
    const matches = await this.findPotentialMatches(newContact);
    
    if (matches.length === 0) {
      return { action: 'create_new', confidence: 1.0 };
    }
    
    const bestMatch = matches[0];
    
    if (bestMatch.confidence > 0.9) {
      // High confidence match - auto-merge
      return {
        action: 'auto_merge',
        existing_contact_id: bestMatch.contact.id,
        confidence: bestMatch.confidence
      };
    } else if (bestMatch.confidence > 0.7) {
      // Medium confidence - flag for manual review
      return {
        action: 'manual_review',
        potential_matches: matches.slice(0, 3),
        confidence: bestMatch.confidence
      };
    } else {
      // Low confidence - create new contact
      return { action: 'create_new', confidence: bestMatch.confidence };
    }
  }
  
  async mergeContacts(primaryContactId: string, secondaryContactId: string): Promise<void> {
    // Merge conversation history
    await this.mergeConversationHistory(primaryContactId, secondaryContactId);
    
    // Merge contact preferences
    await this.mergeContactPreferences(primaryContactId, secondaryContactId);
    
    // Update all references to secondary contact
    await this.updateContactReferences(secondaryContactId, primaryContactId);
    
    // Mark secondary contact as merged
    await this.markContactAsMerged(secondaryContactId, primaryContactId);
  }
}
```

## Communication Preferences

### Preference Management Service
```typescript
class CommunicationPreferenceService {
  async updateContactPreferences(contactId: string, preferences: ContactPreferences): Promise<void> {
    // Update in ResMan
    await this.updateResManPreferences(contactId, preferences);
    
    // Update in DomIQ
    await this.updateDomIQPreferences(contactId, preferences);
    
    // Log preference change
    await this.logPreferenceChange(contactId, preferences);
  }
  
  async handleOptOut(contactId: string, optOutType: 'email' | 'phone' | 'all'): Promise<void> {
    const preferences = await this.getContactPreferences(contactId);
    
    switch (optOutType) {
      case 'email':
        preferences.email_opt_out = true;
        break;
      case 'phone':
        preferences.phone_opt_out = true;
        break;
      case 'all':
        preferences.email_opt_out = true;
        preferences.phone_opt_out = true;
        break;
    }
    
    await this.updateContactPreferences(contactId, preferences);
  }
}
```

## Monitoring & Analytics

### Contact Analytics
```typescript
interface ContactAnalytics {
  getContactMetrics(propertyId: string, dateRange: DateRange): Promise<{
    total_contacts: number;
    new_contacts: number;
    active_contacts: number;
    contact_growth_rate: number;
    average_lead_score: number;
    conversion_rate: number;
  }>;
  
  getContactTrends(propertyId: string): Promise<{
    daily_new_contacts: Array<{date: string, count: number}>;
    contact_source_distribution: Record<string, number>;
    lead_score_distribution: Record<string, number>;
  }>;
  
  getContactEnrichmentMetrics(propertyId: string): Promise<{
    enrichment_rate: number;
    average_enrichment_score: number;
    top_enrichment_sources: Array<{source: string, count: number}>;
  }>;
}
```

## Testing Strategy

### Unit Tests
- Contact matching algorithms
- Data transformation functions
- Duplicate detection logic
- Preference management

### Integration Tests
- End-to-end contact creation flow
- Contact update synchronization
- Webhook processing
- Data consistency validation

### User Acceptance Tests
- Contact creation workflow
- Duplicate handling
- Preference management
- Contact enrichment

## Deployment Checklist

### Pre-deployment
- [ ] ResMan contact API access configured
- [ ] Contact matching rules defined
- [ ] Webhook endpoints registered
- [ ] Duplicate handling strategy confirmed
- [ ] Communication preferences configured

### Post-deployment
- [ ] Verify contact creation functionality
- [ ] Test contact matching accuracy
- [ ] Validate duplicate prevention
- [ ] Check preference synchronization
- [ ] Monitor contact enrichment

## Success Criteria
- [ ] 99%+ contact sync success rate
- [ ] <5 second contact creation time
- [ ] 95%+ duplicate detection accuracy
- [ ] Real-time preference synchronization
- [ ] Complete contact history preservation
- [ ] Zero data loss during contact operations 