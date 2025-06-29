# Tour Booking Database Component: DomIQ ↔ ResMan

## Database Schema for Tour Booking Integration

### Core Tables

#### Tour Booking Sync Table
```sql
CREATE TABLE tour_booking_sync (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    resman_appointment_id VARCHAR(100),
    tour_type VARCHAR(50) CHECK (tour_type IN ('in_person', 'virtual', 'self_guided')),
    scheduled_datetime TIMESTAMPTZ,
    status VARCHAR(50) CHECK (status IN ('scheduled', 'confirmed', 'completed', 'cancelled', 'no_show')),
    prospect_info JSONB,
    staff_notes TEXT,
    sync_status VARCHAR(50) CHECK (sync_status IN ('pending', 'success', 'failed')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE tour_booking_sync IS 'Tracks tour bookings and sync status with ResMan';
```

#### Calendar Availability Cache Table
```sql
CREATE TABLE calendar_availability_cache (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    available_slots JSONB NOT NULL, -- Array of available time slots
    last_updated TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(property_id, date)
);

COMMENT ON TABLE calendar_availability_cache IS 'Caches available time slots for each property/date';
```

### Indexes for Performance
```sql
CREATE INDEX idx_tour_booking_sync_status ON tour_booking_sync(status, scheduled_datetime);
CREATE INDEX idx_tour_booking_sync_conversation ON tour_booking_sync(conversation_id);
CREATE INDEX idx_calendar_availability_cache_property_date ON calendar_availability_cache(property_id, date);
```

### Data Consistency Constraints
```sql
ALTER TABLE tour_booking_sync ADD CONSTRAINT chk_tour_booking_status
    CHECK (status IN ('scheduled', 'confirmed', 'completed', 'cancelled', 'no_show'));
ALTER TABLE tour_booking_sync ADD CONSTRAINT chk_tour_booking_sync_status
    CHECK (sync_status IN ('pending', 'success', 'failed'));
```

### Triggers (Optional)
-- Add triggers for automatic status updates or cache invalidation if needed.

### Monitoring Views
```sql
CREATE VIEW tour_booking_success_rate AS
SELECT 
    property_id,
    COUNT(*) FILTER (WHERE status = 'completed')::float / NULLIF(COUNT(*),0) * 100 AS completion_rate,
    COUNT(*) FILTER (WHERE sync_status = 'success')::float / NULLIF(COUNT(*),0) * 100 AS sync_success_rate
FROM tour_booking_sync
GROUP BY property_id;
``` 