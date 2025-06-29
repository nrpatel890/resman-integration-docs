# Universal Tour Backend Component: Multi-Source → ResMan

## Backend API Implementation for Universal Tour Integration

### API Endpoints

#### 1. Universal Tour Submission API
```typescript
// POST /api/universal/tours/submit
interface UniversalTourSubmissionRequest {
  source: string; // 'website', 'phone', 'walkin', 'third_party', 'manual'
  source_id?: string; // Optional ID from source system
  tour_data: {
    // Flexible structure - can accept any format
    [key: string]: any;
  };
  metadata?: {
    submitted_by?: string;
    submission_method?: string;
    source_url?: string;
    user_agent?: string;
    ip_address?: string;
  };
}

interface UniversalTourSubmissionResponse {
  success: boolean;
  resman_tour_id?: string;
  processing_time_ms: number;
  validation_errors?: string[];
  warnings?: string[];
  resman_response?: any; // Full response from ResMan
}
```

#### 2. Tour Data Standardization API
```typescript
// POST /api/universal/tours/standardize
interface TourStandardizationRequest {
  source: string;
  raw_data: Record<string, any>;
}

interface TourStandardizationResponse {
  standardized_data: {
    prospect_info: {
      name: string;
      email?: string;
      phone?: string;
      resman_prospect_id?: string;
    };
    tour_info: {
      property_id: string;
      tour_type: 'self_guided' | 'virtual' | 'agent_led';
      scheduled_date: string;
      duration_minutes: number;
      timezone: string;
      agent_id?: string;
      meeting_location?: string;
      virtual_meeting_url?: string;
      special_instructions?: string;
    };
    preferences?: {
      preferred_contact_method?: 'email' | 'sms' | 'phone';
      accessibility_requirements?: string[];
      language_preference?: string;
      group_size?: number;
    };
  };
  confidence_score: number; // 0-1, how confident we are in the standardization
  missing_fields: string[];
  suggested_values: Record<string, any>;
}
```

#### 3. Health Check API
```typescript
// GET /api/universal/tours/health
interface HealthCheckResponse {
  status: 'healthy' | 'degraded' | 'unhealthy';
  resman_connection: 'connected' | 'disconnected';
  last_resman_check: string;
  uptime_seconds: number;
  total_requests_processed: number;
  average_processing_time_ms: number;
}
```

### Data Standardization Engine

#### 1. In-Memory Data Mapper
```typescript
class UniversalTourDataMapper {
  private sourceConfigs: Map<string, SourceConfig> = new Map();

  async standardizeTourData(source: string, rawData: any): Promise<StandardizedTourData> {
    const config = await this.getSourceConfig(source);
    const mapper = this.createMapper(config);
    
    return mapper.map(rawData);
  }

  private createMapper(config: SourceConfig): DataMapper {
    return {
      map: (rawData: any) => {
        const standardized = {
          prospect_info: this.mapProspectInfo(rawData, config.data_mapping.prospect),
          tour_info: this.mapTourInfo(rawData, config.data_mapping.tour),
          preferences: this.mapPreferences(rawData, config.data_mapping.preferences)
        };

        return {
          ...standardized,
          confidence_score: this.calculateConfidence(standardized),
          missing_fields: this.identifyMissingFields(standardized),
          suggested_values: await this.suggestMissingValues(standardized)
        };
      }
    };
  }

  private mapProspectInfo(rawData: any, mapping: any): ProspectInfo {
    return {
      name: this.extractValue(rawData, mapping.name),
      email: this.extractValue(rawData, mapping.email),
      phone: this.extractValue(rawData, mapping.phone),
      resman_prospect_id: this.extractValue(rawData, mapping.resman_prospect_id)
    };
  }

  private mapTourInfo(rawData: any, mapping: any): TourInfo {
    return {
      property_id: this.extractValue(rawData, mapping.property_id),
      tour_type: this.determineTourType(rawData, mapping.tour_type),
      scheduled_date: this.parseDateTime(rawData, mapping.scheduled_date),
      duration_minutes: this.extractDuration(rawData, mapping.duration),
      timezone: this.extractTimezone(rawData, mapping.timezone),
      agent_id: this.extractValue(rawData, mapping.agent_id),
      meeting_location: this.extractValue(rawData, mapping.meeting_location),
      virtual_meeting_url: this.extractValue(rawData, mapping.virtual_meeting_url),
      special_instructions: this.extractValue(rawData, mapping.special_instructions)
    };
  }

  private extractValue(rawData: any, mapping: any): any {
    if (typeof mapping === 'string') {
      return rawData[mapping];
    }
    
    if (mapping.path) {
      return this.getNestedValue(rawData, mapping.path);
    }
    
    if (mapping.transform) {
      const value = this.extractValue(rawData, mapping.source);
      return this.applyTransform(value, mapping.transform);
    }
    
    return undefined;
  }

  private determineTourType(rawData: any, mapping: any): string {
    const value = this.extractValue(rawData, mapping);
    
    // AI-powered tour type detection
    const tourTypeKeywords = {
      'self_guided': ['self', 'guided', 'walk', 'self-tour', 'unaccompanied'],
      'virtual': ['virtual', 'online', 'video', 'zoom', 'teams', 'remote'],
      'agent_led': ['agent', 'guided', 'accompanied', 'with agent', 'staff']
    };

    const text = JSON.stringify(rawData).toLowerCase();
    
    for (const [type, keywords] of Object.entries(tourTypeKeywords)) {
      if (keywords.some(keyword => text.includes(keyword))) {
        return type;
      }
    }

    return 'agent_led'; // Default
  }

  private parseDateTime(rawData: any, mapping: any): string {
    const value = this.extractValue(rawData, mapping);
    
    if (!value) return null;

    // Try multiple date formats
    const dateFormats = [
      'YYYY-MM-DD HH:mm:ss',
      'MM/DD/YYYY HH:mm',
      'DD/MM/YYYY HH:mm',
      'YYYY-MM-DDTHH:mm:ssZ',
      'MM/DD/YYYY',
      'DD/MM/YYYY'
    ];

    for (const format of dateFormats) {
      try {
        const parsed = moment(value, format, true);
        if (parsed.isValid()) {
          return parsed.toISOString();
        }
      } catch (e) {
        continue;
      }
    }

    // Fallback to AI date parsing
    return this.aiParseDateTime(value);
  }
}
```

#### 2. Source-Specific Handlers
```typescript
class SourceHandlerFactory {
  async createHandler(source: string): Promise<SourceHandler> {
    const config = await this.getSourceConfig(source);
    
    switch (config.source_type) {
      case 'api':
        return new APISourceHandler(config);
      case 'webhook':
        return new WebhookSourceHandler(config);
      case 'form':
        return new FormSourceHandler(config);
      case 'manual':
        return new ManualSourceHandler(config);
      default:
        throw new Error(`Unknown source type: ${config.source_type}`);
    }
  }
}

class APISourceHandler implements SourceHandler {
  constructor(private config: SourceConfig) {}

  async processTourData(rawData: any): Promise<ProcessedTourData> {
    // Handle API-specific data processing
    const processed = await this.validateAPIData(rawData);
    const standardized = await this.standardizeData(processed);
    
    return {
      ...standardized,
      source_metadata: {
        api_version: rawData.api_version,
        request_id: rawData.request_id,
        timestamp: rawData.timestamp
      }
    };
  }
}

class FormSourceHandler implements SourceHandler {
  constructor(private config: SourceConfig) {}

  async processTourData(rawData: any): Promise<ProcessedTourData> {
    // Handle form submission data
    const processed = await this.cleanFormData(rawData);
    const standardized = await this.standardizeData(processed);
    
    return {
      ...standardized,
      source_metadata: {
        form_id: rawData.form_id,
        submission_id: rawData.submission_id,
        user_agent: rawData.user_agent,
        ip_address: rawData.ip_address
      }
    };
  }
}
```

### Main Processing Logic

#### 1. Simple Universal Endpoint
```typescript
// Main endpoint - no database, just process and forward
app.post('/api/universal/tours/submit', async (req, res) => {
  const startTime = Date.now();
  
  try {
    const { source, source_id, tour_data, metadata } = req.body;
    
    // 1. Validate input
    if (!source || !tour_data) {
      return res.status(400).json({
        success: false,
        error: 'Missing required fields: source and tour_data'
      });
    }

    // 2. Standardize the data
    const standardized = await standardizeTourData(source, tour_data);
    
    // 3. Send directly to ResMan
    const resmanResult = await sendToResMan(standardized);
    
    // 4. Return success response
    res.json({
      success: true,
      resman_tour_id: resmanResult.id,
      processing_time_ms: Date.now() - startTime,
      warnings: standardized.warnings || [],
      resman_response: resmanResult
    });
    
  } catch (error) {
    console.error('Error processing tour submission:', error);
    
    res.status(500).json({
      success: false,
      error: error.message,
      processing_time_ms: Date.now() - startTime
    });
  }
});
```

#### 2. ResMan Integration
```typescript
class ResManIntegration {
  async sendToResMan(tourData: StandardizedTourData): Promise<any> {
    try {
      // 1. Check if prospect exists in ResMan
      const prospectId = await this.findOrCreateProspect(tourData.prospect_info);
      
      // 2. Validate property availability
      const availability = await this.checkAvailability(tourData.tour_info);
      
      if (!availability.available) {
        throw new Error(`Property not available for requested time. Conflicts: ${availability.conflicts.length}`);
      }

      // 3. Create tour in ResMan
      const resmanTour = await this.createResManTour({
        ...tourData,
        prospect_id: prospectId
      });

      // 4. Create calendar event
      const calendarEvent = await this.createCalendarEvent(resmanTour);

      return {
        id: resmanTour.id,
        calendar_event_id: calendarEvent.id,
        sync_details: {
          prospect_synced: true,
          tour_created: true,
          calendar_updated: true
        }
      };

    } catch (error) {
      throw new Error(`ResMan sync failed: ${error.message}`);
    }
  }

  private async findOrCreateProspect(prospectInfo: ProspectInfo): Promise<string> {
    // Try to find existing prospect
    let prospectId = prospectInfo.resman_prospect_id;
    
    if (!prospectId) {
      prospectId = await this.searchProspectByEmail(prospectInfo.email);
    }
    
    if (!prospectId) {
      prospectId = await this.searchProspectByPhone(prospectInfo.phone);
    }
    
    // Create new prospect if not found
    if (!prospectId) {
      prospectId = await this.createProspect(prospectInfo);
    }
    
    return prospectId;
  }

  private async checkAvailability(tourInfo: TourInfo): Promise<AvailabilityResult> {
    const conflicts = await this.resManAPI.checkCalendarConflicts({
      property_id: tourInfo.property_id,
      start_time: tourInfo.scheduled_date,
      end_time: this.calculateEndTime(tourInfo.scheduled_date, tourInfo.duration_minutes),
      agent_id: tourInfo.agent_id
    });

    return {
      available: conflicts.length === 0,
      conflicts: conflicts
    };
  }
}
```

### Integration Examples

#### 1. Website Form Integration
```typescript
// Example: Website booking form submission
const handleWebsiteFormSubmission = async (formData: any) => {
  const response = await fetch('/api/universal/tours/submit', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      source: 'website',
      source_id: formData.submission_id,
      tour_data: {
        prospect_name: formData.name,
        prospect_email: formData.email,
        prospect_phone: formData.phone,
        property_name: formData.property,
        preferred_date: formData.date,
        preferred_time: formData.time,
        tour_type: formData.tour_type,
        group_size: formData.group_size,
        special_requests: formData.requests
      },
      metadata: {
        submitted_by: formData.user_id,
        submission_method: 'website_form',
        source_url: formData.source_url,
        user_agent: formData.user_agent,
        ip_address: formData.ip_address
      }
    })
  });

  return response.json();
};
```

#### 2. Phone System Integration
```typescript
// Example: Phone booking system integration
const handlePhoneBooking = async (callData: any) => {
  const response = await fetch('/api/universal/tours/submit', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      source: 'phone',
      source_id: callData.call_id,
      tour_data: {
        caller_name: callData.caller_name,
        caller_phone: callData.caller_phone,
        property_interest: callData.property_interest,
        preferred_date: callData.preferred_date,
        preferred_time: callData.preferred_time,
        tour_type: callData.tour_type,
        agent_notes: callData.agent_notes,
        call_duration: callData.duration,
        call_recording_url: callData.recording_url
      },
      metadata: {
        submitted_by: callData.agent_id,
        submission_method: 'phone_call',
        call_id: callData.call_id,
        call_timestamp: callData.timestamp
      }
    })
  });

  return response.json();
};
```

#### 3. Walk-in App Integration
```typescript
// Example: Walk-in tablet app integration
const handleWalkInBooking = async (appData: any) => {
  const response = await fetch('/api/universal/tours/submit', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      source: 'walkin',
      source_id: appData.session_id,
      tour_data: {
        visitor_name: appData.visitor_name,
        visitor_email: appData.visitor_email,
        visitor_phone: appData.visitor_phone,
        property_id: appData.property_id,
        immediate_tour: appData.immediate_tour,
        preferred_date: appData.preferred_date,
        preferred_time: appData.preferred_time,
        tour_type: appData.tour_type,
        immediate_availability: appData.immediate_availability,
        device_id: appData.device_id
      },
      metadata: {
        submitted_by: appData.staff_id,
        submission_method: 'walkin_app',
        device_location: appData.device_location,
        session_duration: appData.session_duration
      }
    })
  });

  return response.json();
};
```

#### 4. Third-Party Platform Integration
```typescript
// Example: Third-party booking platform webhook
const handleThirdPartyWebhook = async (webhookData: any) => {
  const response = await fetch('/api/universal/tours/submit', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      source: 'third_party',
      source_id: webhookData.booking_id,
      tour_data: {
        platform_booking_id: webhookData.booking_id,
        platform_name: webhookData.platform,
        customer_name: webhookData.customer.name,
        customer_email: webhookData.customer.email,
        customer_phone: webhookData.customer.phone,
        property_id: webhookData.property_id,
        booking_date: webhookData.booking_date,
        booking_time: webhookData.booking_time,
        tour_type: webhookData.tour_type,
        platform_fee: webhookData.fee,
        platform_commission: webhookData.commission
      },
      metadata: {
        submitted_by: 'third_party_webhook',
        submission_method: 'webhook',
        platform: webhookData.platform,
        webhook_signature: webhookData.signature
      }
    })
  });

  return response.json();
};
```

### Simple Monitoring

#### 1. In-Memory Statistics
```typescript
class SimpleTourStats {
  private stats = {
    totalRequests: 0,
    successfulRequests: 0,
    failedRequests: 0,
    averageProcessingTime: 0,
    lastRequestTime: null,
    startTime: Date.now()
  };

  recordRequest(success: boolean, processingTime: number) {
    this.stats.totalRequests++;
    if (success) {
      this.stats.successfulRequests++;
    } else {
      this.stats.failedRequests++;
    }
    
    // Update average processing time
    const totalTime = this.stats.averageProcessingTime * (this.stats.totalRequests - 1) + processingTime;
    this.stats.averageProcessingTime = totalTime / this.stats.totalRequests;
    
    this.stats.lastRequestTime = new Date();
  }

  getStats() {
    return {
      ...this.stats,
      successRate: this.stats.totalRequests > 0 ? 
        (this.stats.successfulRequests / this.stats.totalRequests) * 100 : 0,
      uptimeSeconds: Math.floor((Date.now() - this.stats.startTime) / 1000)
    };
  }
}
```

#### 2. Health Check Endpoint
```typescript
app.get('/api/universal/tours/health', async (req, res) => {
  try {
    // Check ResMan connection
    const resmanStatus = await checkResManConnection();
    
    const stats = tourStats.getStats();
    
    res.json({
      status: resmanStatus.connected ? 'healthy' : 'degraded',
      resman_connection: resmanStatus.connected ? 'connected' : 'disconnected',
      last_resman_check: new Date().toISOString(),
      uptime_seconds: stats.uptimeSeconds,
      total_requests_processed: stats.totalRequests,
      average_processing_time_ms: Math.round(stats.averageProcessingTime),
      success_rate_percent: Math.round(stats.successRate)
    });
  } catch (error) {
    res.status(500).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});
```

### Error Handling

#### 1. Validation Errors
```typescript
class TourValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public value: any,
    public validationType: string
  ) {
    super(message);
    this.name = 'TourValidationError';
  }
}

const validateTourData = (tourData: any): ValidationResult => {
  const errors: ValidationError[] = [];
  
  // Required field validation
  if (!tourData.prospect_info?.name) {
    errors.push({
      field: 'prospect_info.name',
      message: 'Prospect name is required',
      type: 'required'
    });
  }
  
  if (!tourData.tour_info?.property_id) {
    errors.push({
      field: 'tour_info.property_id',
      message: 'Property ID is required',
      type: 'required'
    });
  }
  
  return {
    valid: errors.length === 0,
    errors
  };
};
```

#### 2. ResMan Integration Errors
```typescript
class ResManIntegrationError extends Error {
  constructor(
    message: string,
    public operation: string,
    public resmanError?: any
  ) {
    super(message);
    this.name = 'ResManIntegrationError';
  }
}

const handleResManError = (error: any, operation: string): never => {
  if (error.response?.status === 404) {
    throw new ResManIntegrationError(
      `Resource not found in ResMan: ${operation}`,
      operation,
      error.response.data
    );
  }
  
  if (error.response?.status === 409) {
    throw new ResManIntegrationError(
      `Conflict in ResMan: ${operation}`,
      operation,
      error.response.data
    );
  }
  
  throw new ResManIntegrationError(
    `ResMan API error during ${operation}: ${error.message}`,
    operation,
    error.response?.data
  );
};
```

### Testing Strategy

#### Unit Tests
- Data standardization functions
- Source-specific handlers
- ResMan integration logic
- Error handling scenarios

#### Integration Tests
- End-to-end tour submission flow
- Multi-source data processing
- ResMan API integration
- Error response handling

### Deployment Checklist

#### Pre-deployment
- [ ] Universal API endpoints deployed
- [ ] Data standardization engine configured
- [ ] ResMan API access configured
- [ ] Source configurations defined
- [ ] Health check endpoint ready

#### Post-deployment
- [ ] Verify universal endpoint functionality
- [ ] Test data standardization accuracy
- [ ] Validate ResMan sync process
- [ ] Check health monitoring
- [ ] Monitor performance metrics

### Success Criteria
- [ ] 100% tour data acceptance from any source
- [ ] 95%+ data standardization accuracy
- [ ] 99%+ ResMan sync success rate
- [ ] <3 second processing time per tour
- [ ] Zero local data storage required
- [ ] Support for unlimited booking sources 