# Multi-Channel Communications Database Component: DomIQ ↔ ResMan

## Database Schema for Multi-Channel Communications Logging Integration

### Core Tables

#### Communication Log Sync Table
```sql
CREATE TABLE communication_log_sync (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    resman_prospect_id VARCHAR(100),
    communication_type VARCHAR(50) CHECK (communication_type IN ('chat', 'email', 'sms', 'voice')),
    message_id VARCHAR(100),
    sender_type VARCHAR(50) CHECK (sender_type IN ('ai', 'prospect', 'staff')),
    content TEXT,
    content_summary TEXT,
    sentiment_score DECIMAL(3,2),
    key_topics JSONB,
    timestamp TIMESTAMPTZ,
    sync_direction VARCHAR(20) CHECK (sync_direction IN ('to_resman', 'from_resman', 'bidirectional')),
    sync_status VARCHAR(50) CHECK (sync_status IN ('pending', 'success', 'failed', 'conflict')),
    sync_type VARCHAR(50) CHECK (sync_type IN ('create', 'update', 'enrichment', 'compliance')),
    retry_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE communication_log_sync IS 'Tracks all communications and sync status between DomIQ and ResMan';
```

#### Communication Metadata Table
```sql
CREATE TABLE communication_metadata (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    communication_log_id UUID REFERENCES communication_log_sync(id) ON DELETE CASCADE,
    metadata_type VARCHAR(50) CHECK (metadata_type IN ('attachments', 'links', 'emotions', 'intents')),
    metadata_value JSONB,
    sync_status VARCHAR(50) CHECK (sync_status IN ('pending', 'success', 'failed')),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE communication_metadata IS 'Stores metadata for communications (attachments, links, emotions, intents)';
```

#### Communication Compliance Flags Table
```sql
CREATE TABLE communication_compliance_flags (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    communication_log_id UUID REFERENCES communication_log_sync(id) ON DELETE CASCADE,
    flag_type VARCHAR(50) CHECK (flag_type IN ('gdpr_consent', 'opt_out', 'sensitive_data', 'regulatory')),
    flag_description TEXT,
    flag_severity VARCHAR(20) CHECK (flag_severity IN ('low', 'medium', 'high', 'critical')),
    is_resolved BOOLEAN DEFAULT FALSE,
    resolved_by VARCHAR(100),
    resolved_at TIMESTAMPTZ,
    sync_status VARCHAR(50) CHECK (sync_status IN ('pending', 'success', 'failed')),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE communication_compliance_flags IS 'Tracks compliance flags for communications';
```

#### Communication Sync Queue Table
```sql
CREATE TABLE communication_sync_queue (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    communication_log_id UUID REFERENCES communication_log_sync(id) ON DELETE CASCADE,
    sync_operation VARCHAR(50) CHECK (sync_operation IN ('push', 'pull', 'merge', 'compliance_check')),
    sync_data JSONB,
    priority INTEGER DEFAULT 1 CHECK (priority IN (1, 2, 3)),
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    next_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE communication_sync_queue IS 'Queue for managing communication sync operations with retry logic';
```

### Indexes for Performance
```sql
CREATE INDEX idx_communication_log_sync_status ON communication_log_sync(sync_status, sync_direction, created_at);
CREATE INDEX idx_communication_log_sync_type ON communication_log_sync(communication_type, sender_type, timestamp);
CREATE INDEX idx_communication_log_sync_prospect ON communication_log_sync(resman_prospect_id, timestamp);
CREATE INDEX idx_communication_metadata_type ON communication_metadata(metadata_type, sync_status);
CREATE INDEX idx_communication_compliance_flags_severity ON communication_compliance_flags(flag_severity, is_resolved);
CREATE INDEX idx_communication_sync_queue_priority ON communication_sync_queue(priority, next_retry_at) WHERE next_retry_at IS NOT NULL;
```

### Data Consistency Constraints
```sql
ALTER TABLE communication_log_sync ADD CONSTRAINT chk_sentiment_score
    CHECK (sentiment_score >= -1 AND sentiment_score <= 1);
ALTER TABLE communication_log_sync ADD CONSTRAINT chk_retry_count
    CHECK (retry_count >= 0);
ALTER TABLE communication_sync_queue ADD CONSTRAINT chk_priority_range
    CHECK (priority >= 1 AND priority <= 3);
ALTER TABLE communication_sync_queue ADD CONSTRAINT chk_retry_count_limit
    CHECK (retry_count <= max_retries);
```

### Monitoring Views
```sql
CREATE VIEW communication_sync_success_rate AS
SELECT 
    communication_type,
    sync_direction,
    COUNT(*) as total_communications,
    COUNT(*) FILTER (WHERE sync_status = 'success') as successful_syncs,
    ROUND(
        COUNT(*) FILTER (WHERE sync_status = 'success')::DECIMAL / COUNT(*) * 100, 2
    ) as success_rate
FROM communication_log_sync
WHERE created_at >= NOW() - INTERVAL '24 hours'
GROUP BY communication_type, sync_direction;

CREATE VIEW compliance_flags_summary AS
SELECT 
    flag_type,
    flag_severity,
    COUNT(*) as total_flags,
    COUNT(*) FILTER (WHERE is_resolved = true) as resolved_flags,
    ROUND(
        COUNT(*) FILTER (WHERE is_resolved = true)::DECIMAL / COUNT(*) * 100, 2
    ) as resolution_rate
FROM communication_compliance_flags
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY flag_type, flag_severity;
``` 