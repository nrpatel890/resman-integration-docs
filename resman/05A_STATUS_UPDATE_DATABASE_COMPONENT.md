# Status Update Database Component: DomIQ ↔ ResMan

## Database Schema for Status Update Integration

### Core Tables

#### Status Sync Log Table
```sql
CREATE TABLE status_sync_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    resman_lead_id VARCHAR(100),
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    status_reason TEXT,
    updated_by VARCHAR(100) CHECK (updated_by IN ('chatbot', 'resman_user', 'system')),
    sync_direction VARCHAR(20) CHECK (sync_direction IN ('to_resman', 'from_resman')),
    sync_status VARCHAR(50) CHECK (sync_status IN ('pending', 'success', 'failed')),
    context_data JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE status_sync_log IS 'Audit log of all status changes and syncs between DomIQ and ResMan';
```

#### Status Workflow Rules Table
```sql
CREATE TABLE status_workflow_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    trigger_status VARCHAR(50),
    target_status VARCHAR(50),
    conditions JSONB,
    actions JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE status_workflow_rules IS 'Defines workflow rules for status transitions and automation';
```

#### Status Notification Preferences Table
```sql
CREATE TABLE status_notification_preferences (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    status_type VARCHAR(50),
    notification_method VARCHAR(50) CHECK (notification_method IN ('email', 'sms', 'slack', 'in_app')),
    recipients JSONB,
    template_id VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE status_notification_preferences IS 'Stores notification preferences for status changes';
```

### Indexes for Performance
```sql
CREATE INDEX idx_status_sync_log_status ON status_sync_log(new_status, sync_status, created_at);
CREATE INDEX idx_status_workflow_rules_property ON status_workflow_rules(property_id, is_active);
CREATE INDEX idx_status_notification_preferences_property ON status_notification_preferences(property_id, status_type);
```

### Data Consistency Constraints
```sql
ALTER TABLE status_sync_log ADD CONSTRAINT chk_status_sync_status
    CHECK (sync_status IN ('pending', 'success', 'failed'));
ALTER TABLE status_sync_log ADD CONSTRAINT chk_status_sync_direction
    CHECK (sync_direction IN ('to_resman', 'from_resman'));
ALTER TABLE status_sync_log ADD CONSTRAINT chk_status_updated_by
    CHECK (updated_by IN ('chatbot', 'resman_user', 'system'));
ALTER TABLE status_notification_preferences ADD CONSTRAINT chk_notification_method
    CHECK (notification_method IN ('email', 'sms', 'slack', 'in_app'));
```

### Monitoring Views
```sql
CREATE VIEW status_sync_success_rate AS
SELECT 
    property_id,
    COUNT(*) FILTER (WHERE sync_status = 'success')::float / NULLIF(COUNT(*),0) * 100 AS sync_success_rate,
    COUNT(*) FILTER (WHERE sync_status = 'failed') AS failed_status_updates
FROM status_sync_log ssl
JOIN conversation c ON ssl.conversation_id = c.id
GROUP BY property_id;
``` 