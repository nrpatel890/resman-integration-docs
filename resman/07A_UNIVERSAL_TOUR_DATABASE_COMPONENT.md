# Universal Tour Database Component: Multi-Source → ResMan

## Database Schema for Universal Tour Integration

### Overview
The Universal Tour Integration is designed to be lightweight with minimal database requirements. This system operates primarily in-memory with no persistent storage for tour data, as all data is processed and immediately forwarded to ResMan.

### Minimal Database Tables (Optional)

#### 1. Tour Processing Log Table (Optional - for debugging/monitoring)
```sql
CREATE TABLE tour_processing_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source VARCHAR(50) NOT NULL,
    source_id VARCHAR(100),
    processing_status VARCHAR(20) CHECK (processing_status IN ('processing', 'success', 'failed')),
    processing_time_ms INTEGER,
    resman_tour_id VARCHAR(100),
    resman_prospect_id VARCHAR(100),
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE tour_processing_log IS 'Optional logging table for debugging and monitoring tour processing';
```

#### 2. Source Configuration Table (Optional - for dynamic source management)
```sql
CREATE TABLE source_configurations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source_name VARCHAR(50) UNIQUE NOT NULL,
    source_type VARCHAR(20) CHECK (source_type IN ('api', 'webhook', 'form', 'manual')),
    is_active BOOLEAN DEFAULT TRUE,
    data_mapping JSONB,
    validation_rules JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE source_configurations IS 'Configuration for different tour booking sources';
```

#### 3. Processing Statistics Table (Optional - for monitoring)
```sql
CREATE TABLE tour_processing_stats (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    date DATE NOT NULL,
    source VARCHAR(50) NOT NULL,
    total_requests INTEGER DEFAULT 0,
    successful_requests INTEGER DEFAULT 0,
    failed_requests INTEGER DEFAULT 0,
    average_processing_time_ms INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(date, source)
);

COMMENT ON TABLE tour_processing_stats IS 'Daily statistics for tour processing by source';
```

### Indexes for Performance (if tables are used)
```sql
-- Only create indexes if the optional tables are implemented
CREATE INDEX idx_tour_processing_log_source ON tour_processing_log(source, created_at);
CREATE INDEX idx_tour_processing_log_status ON tour_processing_log(processing_status, created_at);
CREATE INDEX idx_source_configurations_active ON source_configurations(is_active, source_type);
CREATE INDEX idx_tour_processing_stats_date ON tour_processing_stats(date, source);
```

### In-Memory Data Structures

#### 1. Source Configuration Cache
```typescript
// In-memory source configurations (no database required)
interface SourceConfig {
  source_name: string;
  source_type: 'api' | 'webhook' | 'form' | 'manual';
  data_mapping: {
    prospect: {
      name: string | MappingRule;
      email: string | MappingRule;
      phone: string | MappingRule;
      resman_prospect_id?: string | MappingRule;
    };
    tour: {
      property_id: string | MappingRule;
      tour_type: string | MappingRule;
      scheduled_date: string | MappingRule;
      duration_minutes: number | MappingRule;
      timezone: string | MappingRule;
      agent_id?: string | MappingRule;
      meeting_location?: string | MappingRule;
      virtual_meeting_url?: string | MappingRule;
      special_instructions?: string | MappingRule;
    };
    preferences?: {
      preferred_contact_method?: string | MappingRule;
      accessibility_requirements?: string | MappingRule;
      language_preference?: string | MappingRule;
      group_size?: number | MappingRule;
    };
  };
  validation_rules?: ValidationRule[];
}

interface MappingRule {
  source: string;
  path?: string;
  transform?: string;
  default?: any;
}

interface ValidationRule {
  field: string;
  required: boolean;
  type: 'string' | 'email' | 'phone' | 'date' | 'number';
  min_length?: number;
  max_length?: number;
  pattern?: string;
}
```

#### 2. Processing Statistics (In-Memory)
```typescript
// In-memory statistics tracking
interface TourProcessingStats {
  totalRequests: number;
  successfulRequests: number;
  failedRequests: number;
  averageProcessingTime: number;
  lastRequestTime: Date | null;
  startTime: Date;
  sourceStats: Map<string, SourceStats>;
}

interface SourceStats {
  totalRequests: number;
  successfulRequests: number;
  failedRequests: number;
  averageProcessingTime: number;
  lastRequestTime: Date | null;
}
```

### Data Flow Architecture

#### 1. No-Persistence Flow
```
Tour Submission → In-Memory Processing → ResMan API → Response
```

#### 2. Optional Logging Flow
```
Tour Submission → In-Memory Processing → ResMan API → Optional Log → Response
```

### Configuration Management

#### 1. Source Configuration Loading
```typescript
class SourceConfigurationManager {
  private configs: Map<string, SourceConfig> = new Map();

  // Load configurations from environment variables or config files
  async loadConfigurations(): Promise<void> {
    // Default configurations for common sources
    const defaultConfigs: SourceConfig[] = [
      {
        source_name: 'website',
        source_type: 'form',
        data_mapping: {
          prospect: {
            name: 'prospect_name',
            email: 'prospect_email',
            phone: 'prospect_phone'
          },
          tour: {
            property_id: 'property_name',
            tour_type: 'tour_type',
            scheduled_date: 'preferred_date',
            duration_minutes: 60,
            timezone: 'UTC'
          }
        }
      },
      {
        source_name: 'phone',
        source_type: 'manual',
        data_mapping: {
          prospect: {
            name: 'caller_name',
            phone: 'caller_phone'
          },
          tour: {
            property_id: 'property_interest',
            tour_type: 'agent_led',
            scheduled_date: 'preferred_date',
            duration_minutes: 60,
            timezone: 'UTC'
          }
        }
      }
    ];

    defaultConfigs.forEach(config => {
      this.configs.set(config.source_name, config);
    });
  }

  getConfiguration(source: string): SourceConfig | undefined {
    return this.configs.get(source);
  }

  addConfiguration(config: SourceConfig): void {
    this.configs.set(config.source_name, config);
  }
}
```

### Monitoring Views (if optional tables are implemented)

#### 1. Processing Success Rate View
```sql
CREATE VIEW tour_processing_success_rate AS
SELECT 
    source,
    DATE(created_at) as processing_date,
    COUNT(*) as total_requests,
    COUNT(*) FILTER (WHERE processing_status = 'success') as successful_requests,
    ROUND(
        COUNT(*) FILTER (WHERE processing_status = 'success')::DECIMAL / COUNT(*) * 100, 2
    ) as success_rate,
    AVG(processing_time_ms) as avg_processing_time_ms
FROM tour_processing_log
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY source, DATE(created_at)
ORDER BY processing_date DESC, success_rate DESC;
```

#### 2. Source Performance View
```sql
CREATE VIEW source_performance_summary AS
SELECT 
    source,
    COUNT(*) as total_requests,
    COUNT(*) FILTER (WHERE processing_status = 'success') as successful_requests,
    COUNT(*) FILTER (WHERE processing_status = 'failed') as failed_requests,
    ROUND(
        COUNT(*) FILTER (WHERE processing_status = 'success')::DECIMAL / COUNT(*) * 100, 2
    ) as success_rate,
    AVG(processing_time_ms) as avg_processing_time_ms,
    MAX(created_at) as last_request_time
FROM tour_processing_log
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY source
ORDER BY total_requests DESC;
```

### Data Retention Policy

#### 1. Minimal Retention (if logging is implemented)
```sql
-- Optional: Clean up old processing logs (keep only 30 days)
CREATE OR REPLACE FUNCTION cleanup_old_tour_logs()
RETURNS void AS $$
BEGIN
    DELETE FROM tour_processing_log 
    WHERE created_at < NOW() - INTERVAL '30 days';
    
    DELETE FROM tour_processing_stats 
    WHERE date < CURRENT_DATE - INTERVAL '90 days';
END;
$$ LANGUAGE plpgsql;

-- Optional: Schedule cleanup (if using cron or similar)
-- 0 2 * * * psql -d your_database -c "SELECT cleanup_old_tour_logs();"
```

### Health Check Queries

#### 1. System Health Check
```sql
-- Check if optional logging is working (if implemented)
SELECT 
    COUNT(*) as recent_requests,
    COUNT(*) FILTER (WHERE processing_status = 'success') as recent_successes,
    COUNT(*) FILTER (WHERE processing_status = 'failed') as recent_failures,
    AVG(processing_time_ms) as avg_processing_time
FROM tour_processing_log
WHERE created_at >= NOW() - INTERVAL '1 hour';
```

#### 2. Source Configuration Check
```sql
-- Check source configurations (if implemented)
SELECT 
    source_name,
    source_type,
    is_active,
    updated_at
FROM source_configurations
WHERE is_active = true
ORDER BY source_name;
```

### Key Design Principles

1. **No Persistent Storage**: Tour data is not stored locally - it's processed and immediately forwarded to ResMan
2. **In-Memory Processing**: All data standardization and processing happens in memory
3. **Optional Logging**: Database tables are optional and only used for debugging/monitoring
4. **Lightweight**: Minimal resource usage and complexity
5. **Stateless**: Each request is processed independently without maintaining state
6. **Fast Processing**: Sub-second processing times with no database I/O for core functionality

### Migration Strategy

#### 1. Initial Setup (No Database Required)
```bash
# No database migration needed for core functionality
# The system can operate entirely without a database
```

#### 2. Optional Logging Setup (if monitoring is desired)
```sql
-- Only run if logging is needed
-- V1__Create_Optional_Tour_Logging.sql
CREATE TABLE IF NOT EXISTS tour_processing_log (
    -- ... table definition
);

CREATE TABLE IF NOT EXISTS source_configurations (
    -- ... table definition
);

CREATE TABLE IF NOT EXISTS tour_processing_stats (
    -- ... table definition
);
```

This database component emphasizes the lightweight, no-persistence approach of the Universal Tour Integration while providing optional structures for monitoring and debugging when needed. 