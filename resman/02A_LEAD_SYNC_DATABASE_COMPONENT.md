# Lead Sync Database Component: DomIQ ↔ ResMan

## Database Schema for Lead Sync Integration

### Core Lead Sync Tables

#### ResMan Integration Configuration Table
```sql
-- Configuration table for ResMan integration settings
CREATE TABLE resman_integration (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    resman_property_id VARCHAR(100) NOT NULL,
    api_credentials JSONB NOT NULL, -- Encrypted API keys and tokens
    webhook_url VARCHAR(255),
    sync_enabled BOOLEAN DEFAULT TRUE,
    sync_direction VARCHAR(20) DEFAULT 'bidirectional' 
        CHECK (sync_direction IN ('to_resman', 'from_resman', 'bidirectional')),
    sync_frequency VARCHAR(20) DEFAULT 'realtime' 
        CHECK (sync_frequency IN ('realtime', 'hourly', 'daily')),
    last_sync_at TIMESTAMPTZ DEFAULT NOW(),
    last_successful_sync_at TIMESTAMPTZ,
    sync_status VARCHAR(50) DEFAULT 'active' 
        CHECK (sync_status IN ('active', 'paused', 'error', 'disconnected')),
    error_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Ensure one integration per property
    CONSTRAINT uq_resman_integration_property UNIQUE (property_id)
);

COMMENT ON TABLE resman_integration IS 'Configuration for ResMan integration per property';
COMMENT ON COLUMN resman_integration.api_credentials IS 'Encrypted ResMan API credentials';
COMMENT ON COLUMN resman_integration.sync_direction IS 'Direction of data synchronization';
COMMENT ON COLUMN resman_integration.sync_frequency IS 'How often to sync data';
```

#### Lead Sync Log Table
```sql
-- Audit trail for all lead sync operations
CREATE TABLE lead_sync_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    resman_lead_id VARCHAR(100),
    sync_direction VARCHAR(20) NOT NULL 
        CHECK (sync_direction IN ('to_resman', 'from_resman', 'bidirectional')),
    sync_status VARCHAR(50) NOT NULL 
        CHECK (sync_status IN ('pending', 'success', 'failed', 'conflict', 'retry')),
    sync_type VARCHAR(50) NOT NULL 
        CHECK (sync_type IN ('create', 'update', 'status_change', 'enrichment', 'merge')),
    sync_data JSONB, -- Original data that was synced
    response_data JSONB, -- Response from ResMan API
    error_message TEXT,
    error_code VARCHAR(100),
    conflict_resolution VARCHAR(50) 
        CHECK (conflict_resolution IN ('resman_wins', 'domiq_wins', 'manual_review', 'merged')),
    conflict_details JSONB, -- Details about conflicts and resolutions
    retry_count INTEGER DEFAULT 0,
    processing_time_ms INTEGER, -- Time taken for sync operation
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Index for performance
    CONSTRAINT idx_lead_sync_log_conversation_created 
        UNIQUE (conversation_id, created_at)
);

COMMENT ON TABLE lead_sync_log IS 'Audit trail for all lead synchronization operations';
COMMENT ON COLUMN lead_sync_log.sync_data IS 'Original data sent/received during sync';
COMMENT ON COLUMN lead_sync_log.response_data IS 'Response data from ResMan API';
COMMENT ON COLUMN lead_sync_log.conflict_details IS 'Detailed conflict information and resolution';
```

#### Lead Sync Queue Table
```sql
-- Queue for managing sync operations with retry logic
CREATE TABLE lead_sync_queue (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    resman_lead_id VARCHAR(100),
    sync_operation VARCHAR(50) NOT NULL 
        CHECK (sync_operation IN ('push', 'pull', 'merge', 'status_update', 'enrichment')),
    sync_data JSONB NOT NULL, -- Data to be synced
    priority INTEGER DEFAULT 1 
        CHECK (priority IN (1, 2, 3)), -- 1=low, 2=medium, 3=high
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    next_retry_at TIMESTAMPTZ,
    status VARCHAR(50) DEFAULT 'pending' 
        CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'cancelled')),
    error_message TEXT,
    processed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE lead_sync_queue IS 'Queue for managing lead sync operations with retry logic';
COMMENT ON COLUMN lead_sync_queue.sync_data IS 'Data payload for sync operation';
COMMENT ON COLUMN lead_sync_queue.priority IS 'Priority level for processing order';
```

#### Lead Sync Conflicts Table
```sql
-- Track and manage sync conflicts between systems
CREATE TABLE lead_sync_conflicts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE SET NULL,
    resman_lead_id VARCHAR(100),
    conflict_type VARCHAR(50) NOT NULL 
        CHECK (conflict_type IN ('status_conflict', 'data_conflict', 'duplicate_lead', 'field_mismatch')),
    conflict_field VARCHAR(100),
    resman_value JSONB,
    domiq_value JSONB,
    conflict_severity VARCHAR(20) DEFAULT 'medium' 
        CHECK (conflict_severity IN ('low', 'medium', 'high', 'critical')),
    resolution_status VARCHAR(50) DEFAULT 'pending' 
        CHECK (resolution_status IN ('pending', 'auto_resolved', 'manual_review', 'resolved')),
    resolution_strategy VARCHAR(50) 
        CHECK (resolution_strategy IN ('resman_wins', 'domiq_wins', 'manual_review', 'merged')),
    resolved_value JSONB,
    resolved_by UUID REFERENCES property_manager(id),
    resolved_at TIMESTAMPTZ,
    notes TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE lead_sync_conflicts IS 'Track and manage conflicts between DomIQ and ResMan data';
COMMENT ON COLUMN lead_sync_conflicts.conflict_field IS 'Field name where conflict occurred';
COMMENT ON COLUMN lead_sync_conflicts.resman_value IS 'Value from ResMan system';
COMMENT ON COLUMN lead_sync_conflicts.domiq_value IS 'Value from DomIQ system';
```

#### Lead Sync Mapping Table
```sql
-- Map DomIQ conversations to ResMan leads
CREATE TABLE lead_sync_mapping (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    resman_lead_id VARCHAR(100) NOT NULL,
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    mapping_status VARCHAR(50) DEFAULT 'active' 
        CHECK (mapping_status IN ('active', 'inactive', 'merged', 'deleted')),
    last_sync_at TIMESTAMPTZ DEFAULT NOW(),
    sync_version INTEGER DEFAULT 1, -- Track sync version for conflict detection
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Ensure unique mapping
    CONSTRAINT uq_lead_sync_mapping_conversation UNIQUE (conversation_id),
    CONSTRAINT uq_lead_sync_mapping_resman_lead UNIQUE (resman_lead_id)
);

COMMENT ON TABLE lead_sync_mapping IS 'Maps DomIQ conversations to ResMan lead IDs';
COMMENT ON COLUMN lead_sync_mapping.sync_version IS 'Version number for conflict detection';
```

### Enhanced Existing Tables

#### Enhanced Conversation Table
```sql
-- Add ResMan-specific fields to conversation table
ALTER TABLE conversation ADD COLUMN resman_lead_id VARCHAR(100);
ALTER TABLE conversation ADD COLUMN resman_sync_status VARCHAR(50) DEFAULT 'not_synced' 
    CHECK (resman_sync_status IN ('not_synced', 'syncing', 'synced', 'failed', 'conflict'));
ALTER TABLE conversation ADD COLUMN resman_last_sync_at TIMESTAMPTZ;
ALTER TABLE conversation ADD COLUMN resman_sync_version INTEGER DEFAULT 0;
ALTER TABLE conversation ADD COLUMN resman_custom_fields JSONB;

-- Add indexes for ResMan sync performance
CREATE INDEX idx_conversation_resman_lead_id ON conversation(resman_lead_id);
CREATE INDEX idx_conversation_resman_sync_status ON conversation(resman_sync_status);
CREATE INDEX idx_conversation_resman_last_sync ON conversation(resman_last_sync_at);
```

#### Enhanced User Table
```sql
-- Add ResMan-specific fields to user table
ALTER TABLE "user" ADD COLUMN resman_contact_id VARCHAR(100);
ALTER TABLE "user" ADD COLUMN resman_sync_status VARCHAR(50) DEFAULT 'not_synced' 
    CHECK (resman_sync_status IN ('not_synced', 'syncing', 'synced', 'failed', 'conflict'));
ALTER TABLE "user" ADD COLUMN resman_last_sync_at TIMESTAMPTZ;
ALTER TABLE "user" ADD COLUMN resman_sync_version INTEGER DEFAULT 0;
ALTER TABLE "user" ADD COLUMN resman_custom_fields JSONB;

-- Add indexes for ResMan sync performance
CREATE INDEX idx_user_resman_contact_id ON "user"(resman_contact_id);
CREATE INDEX idx_user_resman_sync_status ON "user"(resman_sync_status);
```

#### Enhanced Lead Notification Table
```sql
-- Add ResMan-specific fields to lead notification table
ALTER TABLE lead_notification ADD COLUMN resman_notification_id VARCHAR(100);
ALTER TABLE lead_notification ADD COLUMN resman_sync_status VARCHAR(50) DEFAULT 'not_synced' 
    CHECK (resman_sync_status IN ('not_synced', 'syncing', 'synced', 'failed'));
ALTER TABLE lead_notification ADD COLUMN resman_last_sync_at TIMESTAMPTZ;

-- Add indexes for ResMan sync performance
CREATE INDEX idx_lead_notification_resman_id ON lead_notification(resman_notification_id);
CREATE INDEX idx_lead_notification_resman_sync ON lead_notification(resman_sync_status);
```

### Performance Indexes

#### High-Traffic Sync Indexes
```sql
-- Lead sync queue performance
CREATE INDEX idx_lead_sync_queue_status_priority ON lead_sync_queue(status, priority DESC, created_at);
CREATE INDEX idx_lead_sync_queue_next_retry ON lead_sync_queue(next_retry_at) WHERE status = 'pending';
CREATE INDEX idx_lead_sync_queue_conversation ON lead_sync_queue(conversation_id, created_at);

-- Lead sync log performance
CREATE INDEX idx_lead_sync_log_status_created ON lead_sync_log(sync_status, created_at DESC);
CREATE INDEX idx_lead_sync_log_resman_lead ON lead_sync_log(resman_lead_id, created_at DESC);
CREATE INDEX idx_lead_sync_log_direction_type ON lead_sync_log(sync_direction, sync_type, created_at);

-- Lead sync conflicts performance
CREATE INDEX idx_lead_sync_conflicts_status ON lead_sync_conflicts(resolution_status, conflict_severity);
CREATE INDEX idx_lead_sync_conflicts_type_field ON lead_sync_conflicts(conflict_type, conflict_field);
CREATE INDEX idx_lead_sync_conflicts_resolved ON lead_sync_conflicts(resolved_at) WHERE resolution_status = 'resolved';

-- Lead sync mapping performance
CREATE INDEX idx_lead_sync_mapping_status ON lead_sync_mapping(mapping_status, last_sync_at);
CREATE INDEX idx_lead_sync_mapping_property ON lead_sync_mapping(property_id, mapping_status);
```

#### Analytics and Monitoring Indexes
```sql
-- Sync performance analytics
CREATE INDEX idx_lead_sync_log_processing_time ON lead_sync_log(processing_time_ms) WHERE processing_time_ms IS NOT NULL;
CREATE INDEX idx_lead_sync_log_error_tracking ON lead_sync_log(error_code, created_at) WHERE error_code IS NOT NULL;

-- Conflict resolution analytics
CREATE INDEX idx_lead_sync_conflicts_resolution_time ON lead_sync_conflicts(resolved_at - created_at) WHERE resolved_at IS NOT NULL;
CREATE INDEX idx_lead_sync_conflicts_by_resolver ON lead_sync_conflicts(resolved_by, resolved_at) WHERE resolved_by IS NOT NULL;
```

### Data Consistency Constraints

#### Referential Integrity
```sql
-- Ensure conversation has valid ResMan mapping when synced
ALTER TABLE conversation ADD CONSTRAINT chk_resman_sync_mapping 
    CHECK (
        (resman_sync_status = 'synced' AND resman_lead_id IS NOT NULL) OR
        (resman_sync_status != 'synced')
    );

-- Ensure user has valid ResMan mapping when synced
ALTER TABLE "user" ADD CONSTRAINT chk_user_resman_sync_mapping 
    CHECK (
        (resman_sync_status = 'synced' AND resman_contact_id IS NOT NULL) OR
        (resman_sync_status != 'synced')
    );

-- Ensure lead notification has valid ResMan mapping when synced
ALTER TABLE lead_notification ADD CONSTRAINT chk_notification_resman_sync_mapping 
    CHECK (
        (resman_sync_status = 'synced' AND resman_notification_id IS NOT NULL) OR
        (resman_sync_status != 'synced')
    );
```

#### Business Logic Constraints
```sql
-- Ensure sync version increments properly
ALTER TABLE conversation ADD CONSTRAINT chk_resman_sync_version 
    CHECK (resman_sync_version >= 0);

ALTER TABLE "user" ADD CONSTRAINT chk_user_resman_sync_version 
    CHECK (resman_sync_version >= 0);

-- Ensure retry count doesn't exceed max retries
ALTER TABLE lead_sync_queue ADD CONSTRAINT chk_retry_count 
    CHECK (retry_count <= max_retries);

-- Ensure priority is valid
ALTER TABLE lead_sync_queue ADD CONSTRAINT chk_priority_range 
    CHECK (priority >= 1 AND priority <= 3);
```

### Data Archiving and Cleanup

#### Archive Tables for Historical Data
```sql
-- Archive table for old sync logs
CREATE TABLE lead_sync_log_archive (
    LIKE lead_sync_log INCLUDING ALL,
    archived_at TIMESTAMPTZ DEFAULT NOW()
);

-- Archive table for old sync conflicts
CREATE TABLE lead_sync_conflicts_archive (
    LIKE lead_sync_conflicts INCLUDING ALL,
    archived_at TIMESTAMPTZ DEFAULT NOW()
);

-- Archive table for old sync queue items
CREATE TABLE lead_sync_queue_archive (
    LIKE lead_sync_queue INCLUDING ALL,
    archived_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### Cleanup Procedures
```sql
-- Function to archive old sync logs (older than 90 days)
CREATE OR REPLACE FUNCTION archive_old_sync_logs()
RETURNS void AS $$
BEGIN
    INSERT INTO lead_sync_log_archive 
    SELECT *, NOW() FROM lead_sync_log 
    WHERE created_at < NOW() - INTERVAL '90 days';
    
    DELETE FROM lead_sync_log 
    WHERE created_at < NOW() - INTERVAL '90 days';
END;
$$ LANGUAGE plpgsql;

-- Function to archive resolved conflicts (older than 30 days)
CREATE OR REPLACE FUNCTION archive_resolved_conflicts()
RETURNS void AS $$
BEGIN
    INSERT INTO lead_sync_conflicts_archive 
    SELECT *, NOW() FROM lead_sync_conflicts 
    WHERE resolution_status = 'resolved' AND resolved_at < NOW() - INTERVAL '30 days';
    
    DELETE FROM lead_sync_conflicts 
    WHERE resolution_status = 'resolved' AND resolved_at < NOW() - INTERVAL '30 days';
END;
$$ LANGUAGE plpgsql;

-- Function to clean up old queue items (older than 7 days)
CREATE OR REPLACE FUNCTION cleanup_old_queue_items()
RETURNS void AS $$
BEGIN
    INSERT INTO lead_sync_queue_archive 
    SELECT *, NOW() FROM lead_sync_queue 
    WHERE status IN ('completed', 'failed', 'cancelled') AND created_at < NOW() - INTERVAL '7 days';
    
    DELETE FROM lead_sync_queue 
    WHERE status IN ('completed', 'failed', 'cancelled') AND created_at < NOW() - INTERVAL '7 days';
END;
$$ LANGUAGE plpgsql;
```

### Monitoring and Analytics Views

#### Sync Performance Views
```sql
-- View for sync success rates
CREATE VIEW sync_success_rates AS
SELECT 
    property_id,
    sync_direction,
    sync_type,
    COUNT(*) as total_operations,
    COUNT(CASE WHEN sync_status = 'success' THEN 1 END) as successful_operations,
    ROUND(
        COUNT(CASE WHEN sync_status = 'success' THEN 1 END)::DECIMAL / COUNT(*) * 100, 2
    ) as success_rate,
    AVG(processing_time_ms) as avg_processing_time,
    MAX(created_at) as last_operation
FROM lead_sync_log
WHERE created_at >= NOW() - INTERVAL '24 hours'
GROUP BY property_id, sync_direction, sync_type;

-- View for conflict resolution statistics
CREATE VIEW conflict_resolution_stats AS
SELECT 
    conflict_type,
    conflict_severity,
    resolution_status,
    COUNT(*) as total_conflicts,
    COUNT(CASE WHEN resolution_status = 'resolved' THEN 1 END) as resolved_conflicts,
    AVG(EXTRACT(EPOCH FROM (resolved_at - created_at))/3600) as avg_resolution_hours
FROM lead_sync_conflicts
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY conflict_type, conflict_severity, resolution_status;

-- View for queue performance metrics
CREATE VIEW queue_performance_metrics AS
SELECT 
    sync_operation,
    priority,
    status,
    COUNT(*) as queue_items,
    AVG(EXTRACT(EPOCH FROM (processed_at - created_at))) as avg_processing_seconds,
    MAX(retry_count) as max_retries,
    COUNT(CASE WHEN status = 'failed' THEN 1 END) as failed_items
FROM lead_sync_queue
WHERE created_at >= NOW() - INTERVAL '24 hours'
GROUP BY sync_operation, priority, status;
```

#### Real-time Sync Status Views
```sql
-- View for current sync status across properties
CREATE VIEW property_sync_status AS
SELECT 
    p.id as property_id,
    p.name as property_name,
    ri.sync_enabled,
    ri.sync_status,
    ri.last_successful_sync_at,
    ri.error_count,
    COUNT(lsq.id) as pending_queue_items,
    COUNT(lsc.id) as pending_conflicts
FROM property p
LEFT JOIN resman_integration ri ON p.id = ri.property_id
LEFT JOIN lead_sync_queue lsq ON p.id = (
    SELECT property_id FROM conversation WHERE id = lsq.conversation_id
) AND lsq.status = 'pending'
LEFT JOIN lead_sync_conflicts lsc ON p.id = (
    SELECT property_id FROM conversation WHERE id = lsc.conversation_id
) AND lsc.resolution_status = 'pending'
GROUP BY p.id, p.name, ri.sync_enabled, ri.sync_status, ri.last_successful_sync_at, ri.error_count;

-- View for sync health monitoring
CREATE VIEW sync_health_monitoring AS
SELECT 
    'sync_failures' as metric,
    COUNT(*) as count,
    MAX(created_at) as last_occurrence
FROM lead_sync_log 
WHERE sync_status = 'failed' AND created_at >= NOW() - INTERVAL '1 hour'
UNION ALL
SELECT 
    'pending_conflicts' as metric,
    COUNT(*) as count,
    MAX(created_at) as last_occurrence
FROM lead_sync_conflicts 
WHERE resolution_status = 'pending'
UNION ALL
SELECT 
    'queue_backlog' as metric,
    COUNT(*) as count,
    MAX(created_at) as last_occurrence
FROM lead_sync_queue 
WHERE status = 'pending';
```

### Database Triggers for Data Consistency

#### Automatic Sync Status Updates
```sql
-- Trigger to update sync status when conversation changes
CREATE OR REPLACE FUNCTION update_conversation_sync_status()
RETURNS TRIGGER AS $$
BEGIN
    -- Reset sync status when conversation data changes
    IF OLD.resman_sync_status = 'synced' AND 
       (NEW.ai_intent_summary != OLD.ai_intent_summary OR
        NEW.is_qualified != OLD.is_qualified OR
        NEW.is_book_tour != OLD.is_book_tour OR
        NEW.status != OLD.status) THEN
        NEW.resman_sync_status := 'not_synced';
        NEW.resman_sync_version := OLD.resman_sync_version + 1;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_conversation_sync_status
    BEFORE UPDATE ON conversation
    FOR EACH ROW
    EXECUTE FUNCTION update_conversation_sync_status();

-- Trigger to update sync status when user changes
CREATE OR REPLACE FUNCTION update_user_sync_status()
RETURNS TRIGGER AS $$
BEGIN
    -- Reset sync status when user data changes
    IF OLD.resman_sync_status = 'synced' AND 
       (NEW.first_name != OLD.first_name OR
        NEW.last_name != OLD.last_name OR
        NEW.email != OLD.email OR
        NEW.phone != OLD.phone) THEN
        NEW.resman_sync_status := 'not_synced';
        NEW.resman_sync_version := OLD.resman_sync_version + 1;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_user_sync_status
    BEFORE UPDATE ON "user"
    FOR EACH ROW
    EXECUTE FUNCTION update_user_sync_status();
```

#### Automatic Queue Management
```sql
-- Trigger to add sync queue item when conversation needs syncing
CREATE OR REPLACE FUNCTION add_sync_queue_item()
RETURNS TRIGGER AS $$
BEGIN
    -- Add to sync queue when conversation becomes unsynced
    IF NEW.resman_sync_status = 'not_synced' AND 
       (OLD.resman_sync_status IS NULL OR OLD.resman_sync_status != 'not_synced') THEN
        INSERT INTO lead_sync_queue (
            conversation_id,
            sync_operation,
            sync_data,
            priority
        ) VALUES (
            NEW.id,
            'push',
            jsonb_build_object(
                'conversation_id', NEW.id,
                'sync_version', NEW.resman_sync_version
            ),
            2 -- Medium priority
        );
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_add_sync_queue_item
    AFTER UPDATE ON conversation
    FOR EACH ROW
    EXECUTE FUNCTION add_sync_queue_item();
```

This database component provides a comprehensive foundation for bidirectional lead synchronization between DomIQ and ResMan, including data integrity, performance optimization, monitoring capabilities, and automated conflict resolution. 