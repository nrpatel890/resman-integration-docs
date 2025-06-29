# Contact Management Database Component: DomIQ ↔ ResMan

## Database Schema for Contact Management Integration

### Core Tables

#### Contact Sync Table
```sql
CREATE TABLE contact_sync (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES "user"(id) ON DELETE CASCADE,
    resman_contact_id VARCHAR(100),
    sync_status VARCHAR(50) CHECK (sync_status IN ('pending', 'success', 'failed', 'merged')),
    last_sync_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE contact_sync IS 'Tracks contact sync status between DomIQ and ResMan';
```

#### Contact Matching Rules Table
```sql
CREATE TABLE contact_matching_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    matching_criteria JSONB NOT NULL, -- Email, phone, name combinations
    merge_strategy VARCHAR(50) CHECK (merge_strategy IN ('resman_wins', 'domiq_wins', 'manual_review')),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE contact_matching_rules IS 'Defines rules for matching and merging contacts';
```

#### Contact Enrichment Log Table
```sql
CREATE TABLE contact_enrichment_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    contact_id UUID REFERENCES "user"(id) ON DELETE CASCADE,
    enrichment_type VARCHAR(50) CHECK (enrichment_type IN ('conversation_data', 'preferences', 'interaction_history')),
    enriched_data JSONB,
    source VARCHAR(50) CHECK (source IN ('chatbot', 'resman', 'manual')),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE contact_enrichment_log IS 'Logs enrichment data added to contacts';
```

### Indexes for Performance
```sql
CREATE INDEX idx_contact_sync_status ON contact_sync(sync_status, last_sync_at);
CREATE INDEX idx_contact_matching_rules_property ON contact_matching_rules(property_id);
CREATE INDEX idx_contact_enrichment_log_contact ON contact_enrichment_log(contact_id, enrichment_type);
```

### Data Consistency Constraints
```sql
ALTER TABLE contact_sync ADD CONSTRAINT chk_contact_sync_status
    CHECK (sync_status IN ('pending', 'success', 'failed', 'merged'));
ALTER TABLE contact_matching_rules ADD CONSTRAINT chk_merge_strategy
    CHECK (merge_strategy IN ('resman_wins', 'domiq_wins', 'manual_review'));
ALTER TABLE contact_enrichment_log ADD CONSTRAINT chk_enrichment_type
    CHECK (enrichment_type IN ('conversation_data', 'preferences', 'interaction_history'));
```

### Monitoring Views
```sql
CREATE VIEW contact_sync_success_rate AS
SELECT 
    property_id,
    COUNT(*) FILTER (WHERE sync_status = 'success')::float / NULLIF(COUNT(*),0) * 100 AS sync_success_rate,
    COUNT(*) FILTER (WHERE sync_status = 'merged') AS merged_contacts
FROM contact_sync cs
JOIN "user" u ON cs.user_id = u.id
JOIN property p ON u.id = p.id
GROUP BY property_id;
``` 