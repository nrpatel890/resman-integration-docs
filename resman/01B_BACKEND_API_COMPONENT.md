# Backend API Component: ResMan Integration

## API Architecture Overview

The backend API component handles all data operations, external integrations, and business logic for the ResMan integration system. It provides RESTful endpoints for frontend consumption and manages bidirectional data flow with ResMan.

## Core API Endpoints

### Authentication & Authorization
```typescript
// Authentication endpoints
POST /api/auth/login
POST /api/auth/logout
POST /api/auth/refresh
GET /api/auth/verify
POST /api/auth/forgot-password
POST /api/auth/reset-password

// Authorization middleware
interface AuthMiddleware {
  verifyToken: (req: Request, res: Response, next: NextFunction) => void;
  requireRole: (roles: string[]) => (req: Request, res: Response, next: NextFunction) => void;
  requirePropertyAccess: (req: Request, res: Response, next: NextFunction) => void;
}
```

### Company Management APIs
```typescript
// Company CRUD operations
GET /api/companies
GET /api/companies/:id
POST /api/companies
PUT /api/companies/:id
DELETE /api/companies/:id

// Company integration settings
GET /api/companies/:id/integrations
PUT /api/companies/:id/integrations
POST /api/companies/:id/test-integration

interface CompanyAPI {
  // Core company data
  id: string;
  name: string;
  logo_url?: string;
  contact_email?: string;
  contact_phone?: string;
  hubspot_company_id?: string;
  
  // Integration API keys
  calendly_org_uri?: string;
  sendgrid_api_key?: string;
  amplitude_api_key?: string;
  openai_api_key?: string;
}
```

### Property Management APIs
```typescript
// Property CRUD operations
GET /api/properties
GET /api/properties/:id
POST /api/properties
PUT /api/properties/:id
DELETE /api/properties/:id

// Property chatbot configuration
GET /api/properties/:id/chatbot
PUT /api/properties/:id/chatbot
POST /api/properties/:id/chatbot/test
GET /api/properties/:id/embed-code

// Property manager assignments
GET /api/properties/:id/managers
POST /api/properties/:id/managers
DELETE /api/properties/:id/managers/:managerId

interface PropertyAPI {
  id: string;
  company_id: string;
  name: string;
  address: string;
  city: string;
  state: string;
  zip_code: string;
  property_type?: string;
  units_count?: number;
  amenities?: JSON;
  features?: JSON;
  website_url?: string;
  hubspot_property_id?: string;
  
  // Chatbot configuration
  chatbot_avatar_url?: string;
  chatbot_is_active: boolean;
  chatbot_welcome_message?: string;
  chatbot_embed_code?: string;
  chatbot_widget_settings?: JSON;
  chatbot_theme_color?: string;
  chatbot_position?: string;
  chatbot_auto_open?: boolean;
  chatbot_business_hours?: JSON;
}
```

### Property Manager APIs
```typescript
// Property manager CRUD operations
GET /api/property-managers
GET /api/property-managers/:id
POST /api/property-managers
PUT /api/property-managers/:id
DELETE /api/property-managers/:id

// Manager assignments
GET /api/property-managers/:id/assignments
POST /api/property-managers/:id/assignments
PUT /api/property-managers/:id/assignments/:assignmentId
DELETE /api/property-managers/:id/assignments/:assignmentId

// Manager notifications
GET /api/property-managers/:id/notifications
PUT /api/property-managers/:id/notifications/:notificationId
POST /api/property-managers/:id/notifications/mark-read

interface PropertyManagerAPI {
  id: string;
  company_id: string;
  first_name: string;
  last_name: string;
  email: string;
  phone: string;
  role?: string;
  access_level: 'read' | 'write';
  
  // Assignment data
  assignments?: PropertyManagerAssignment[];
  notifications?: LeadNotification[];
}
```

### User Management APIs
```typescript
// User CRUD operations
GET /api/users
GET /api/users/:id
POST /api/users
PUT /api/users/:id
DELETE /api/users/:id

// Browser persistence
GET /api/users/browser/:browserId
POST /api/users/browser/:browserId/session
PUT /api/users/browser/:browserId/last-seen

// User sessions
GET /api/users/:id/sessions
GET /api/users/:id/conversations

interface UserAPI {
  id: string;
  first_name?: string;
  last_name?: string;
  email?: string;
  phone?: string;
  age?: number;
  lead_source?: string;
  hubspot_contact_id?: string;
  
  // Browser persistence
  browser_id?: string;
  session_count: number;
  last_seen?: Date;
}
```

### Conversation Management APIs
```typescript
// Conversation CRUD operations
GET /api/conversations
GET /api/conversations/:id
POST /api/conversations
PUT /api/conversations/:id
DELETE /api/conversations/:id

// Conversation messages
GET /api/conversations/:id/messages
POST /api/conversations/:id/messages
PUT /api/conversations/:id/messages/:messageId
DELETE /api/conversations/:id/messages/:messageId

// Conversation status updates
PUT /api/conversations/:id/status
POST /api/conversations/:id/qualify
POST /api/conversations/:id/book-tour

// Conversation analytics
GET /api/conversations/:id/analytics
GET /api/conversations/analytics/summary

interface ConversationAPI {
  id: string;
  chatbot_id: string;
  user_id?: string;
  start_time: Date;
  end_time?: Date;
  
  // Lead qualification data
  is_qualified: boolean;
  is_book_tour: boolean;
  tour_type?: string;
  tour_datetime?: Date;
  ai_intent_summary?: string;
  apartment_size_preference?: string;
  move_in_date?: Date;
  price_range_min?: number;
  price_range_max?: number;
  occupants_count?: number;
  has_pets?: boolean;
  pet_details?: JSON;
  desired_features?: JSON;
  work_location?: string;
  reason_for_moving?: string;
  pre_qualified: boolean;
  source?: string;
  status: ConversationStatus;
  notification_status?: JSON;
  lead_score?: number;
}
```

### FAQ Management APIs
```typescript
// FAQ CRUD operations
GET /api/faqs
GET /api/faqs/:id
POST /api/faqs
PUT /api/faqs/:id
DELETE /api/faqs/:id

// FAQ human-in-loop workflow
GET /api/faqs/pending
GET /api/faqs/assigned/:managerId
PUT /api/faqs/:id/assign/:managerId
PUT /api/faqs/:id/answer
POST /api/faqs/:id/needs-review

// FAQ RAG integration
POST /api/faqs/rag/query
POST /api/faqs/rag/update
GET /api/faqs/rag/status

interface FAQAPI {
  id: string;
  property_id: string;
  question: string;
  answer?: string;
  category?: string;
  source_type?: string;
  status: 'pending' | 'answered' | 'needs_review';
  assigned_to?: string;
}
```

### Lead Notification APIs
```typescript
// Lead notification management
GET /api/lead-notifications
GET /api/lead-notifications/:id
POST /api/lead-notifications
PUT /api/lead-notifications/:id
DELETE /api/lead-notifications/:id

// Notification status updates
PUT /api/lead-notifications/:id/read
PUT /api/lead-notifications/:id/respond
POST /api/lead-notifications/:id/send

// Notification preferences
GET /api/lead-notifications/preferences/:managerId
PUT /api/lead-notifications/preferences/:managerId

interface LeadNotificationAPI {
  id: string;
  conversation_id: string;
  property_manager_id?: string;
  notification_type: string;
  status: string;
  sent_at?: Date;
  read_at?: Date;
  response_at?: Date;
}
```

## Enhanced Feature APIs

### Amplitude Analytics APIs
```typescript
// Amplitude data endpoints
POST /api/amplitude-data
GET /api/amplitude-data/conversation/:conversationId
GET /api/amplitude-data/property/:propertyId
GET /api/amplitude-data/summary

// Analytics calculations
POST /api/amplitude-data/calculate-engagement
GET /api/amplitude-data/engagement-scores
GET /api/amplitude-data/lead-qualification-rates

interface AmplitudeAnalyticsAPI {
  id: string;
  conversation_id: string;
  user_id?: string;
  
  // Core engagement metrics
  chat_session_started: boolean;
  user_messages_sent: number;
  bot_messages_received: number;
  answer_button_clicks: number;
  contact_captured: boolean;
  contact_method?: string;
  tour_booked: boolean;
  tour_type?: string;
  email_office_clicked: number;
  phone_call_clicked: number;
  incentive_offered: boolean;
  incentive_accepted: boolean;
  incentive_expired: boolean;
  admin_handoff_triggered: boolean;
  customer_service_escalated: boolean;
  conversation_abandoned: boolean;
  widget_session_ended: boolean;
  widget_minimized: number;
  
  // Calculated metrics
  session_duration?: number;
  engagement_score?: string;
  qualified: boolean;
  pre_lease: boolean;
  tour_intent: boolean;
  hot: boolean;
  signed: boolean;
  
  // Session tracking
  device_id?: string;
  session_id?: string;
  page_url?: string;
}
```

### Email Management APIs
```typescript
// Email draft management
GET /api/email/drafts
GET /api/email/drafts/:id
POST /api/email/drafts
PUT /api/email/drafts/:id
DELETE /api/email/drafts/:id

// Email sending
POST /api/email/send/:id
POST /api/email/send-test
POST /api/email/schedule/:id

// Email campaigns
GET /api/email/campaigns
GET /api/email/campaigns/:id
POST /api/email/campaigns
PUT /api/email/campaigns/:id
DELETE /api/email/campaigns/:id

// Email templates
GET /api/email/templates
POST /api/email/templates
PUT /api/email/templates/:id

interface EmailDraftAPI {
  id: string;
  lead_id: string;
  conversation_id: string;
  subject: string;
  content: string;
  preview?: string;
  status: 'draft' | 'scheduled' | 'sent' | 'failed';
  priority: 'low' | 'medium' | 'high';
  tags?: JSON;
  scheduled_for?: Date;
  sent_at?: Date;
}

interface EmailCampaignAPI {
  id: string;
  property_id: string;
  title: string;
  description?: string;
  template_content: string;
  sent_count: number;
}
```

### Task Management APIs
```typescript
// Task CRUD operations
GET /api/tasks
GET /api/tasks/:id
POST /api/tasks
PUT /api/tasks/:id
DELETE /api/tasks/:id

// Task assignment
PUT /api/tasks/:id/assign/:managerId
PUT /api/tasks/:id/unassign
GET /api/tasks/assigned/:managerId

// Task status updates
PUT /api/tasks/:id/status
PUT /api/tasks/:id/answer

interface TaskAPI {
  id: string;
  property_id: string;
  conversation_id?: string;
  assigned_to?: string;
  question: string;
  answer?: string;
  status: 'pending' | 'in_progress' | 'completed' | 'cancelled';
}
```

### Tour Management APIs
```typescript
// Tour CRUD operations
GET /api/tours
GET /api/tours/:id
POST /api/tours
PUT /api/tours/:id
DELETE /api/tours/:id

// Tour scheduling
POST /api/tours/schedule
PUT /api/tours/:id/reschedule
PUT /api/tours/:id/cancel
PUT /api/tours/:id/confirm

// Tour availability
GET /api/tours/availability/:propertyId
GET /api/tours/availability/:propertyId/:managerId

// Tour outcomes
PUT /api/tours/:id/outcome
GET /api/tours/outcomes/:propertyId

interface TourAPI {
  id: string;
  conversation_id?: string;
  property_id: string;
  property_manager_id?: string;
  prospect_name: string;
  prospect_email?: string;
  prospect_phone?: string;
  unit_interest?: string;
  tour_datetime: Date;
  duration_minutes: number;
  tour_type: 'in_person' | 'virtual' | 'self_guided';
  status: 'scheduled' | 'confirmed' | 'completed' | 'cancelled' | 'no_show';
  source: 'chat' | 'website' | 'phone' | 'walk_in';
}
```

### Calendar Integration APIs
```typescript
// Calendar integration management
GET /api/calendar/integrations
GET /api/calendar/integrations/:id
POST /api/calendar/integrations
PUT /api/calendar/integrations/:id
DELETE /api/calendar/integrations/:id

// Calendar sync
POST /api/calendar/sync/:integrationId
GET /api/calendar/sync/status/:integrationId
POST /api/calendar/sync/test/:integrationId

// Calendar availability
GET /api/calendar/availability/:propertyId
GET /api/calendar/availability/:propertyId/:managerId

interface CalendarIntegrationAPI {
  id: string;
  property_id: string;
  property_manager_id: string;
  calendly_uri?: string;
  ical_url?: string;
  webhook_url?: string;
  last_sync?: Date;
  sync_status: 'active' | 'inactive' | 'error';
}
```

## ResMan Integration APIs

### ResMan Lead Sync APIs
```typescript
// Lead sync operations
POST /api/resman/leads/sync
GET /api/resman/leads/:leadId
PUT /api/resman/leads/:leadId
POST /api/resman/leads/:leadId/status

// Lead conflict resolution
GET /api/resman/leads/conflicts
POST /api/resman/leads/resolve-conflict/:conflictId

// Lead webhook endpoints
POST /api/resman/webhooks/lead-created
POST /api/resman/webhooks/lead-updated
POST /api/resman/webhooks/lead-status-changed

interface ResManLeadSyncAPI {
  // Push to ResMan
  pushLead: (leadData: LeadData) => Promise<ResManResponse>;
  updateLead: (leadId: string, updates: LeadUpdates) => Promise<ResManResponse>;
  
  // Pull from ResMan
  pullLead: (leadId: string) => Promise<LeadData>;
  pullLeadUpdates: (since: Date) => Promise<LeadUpdate[]>;
  
  // Conflict resolution
  resolveConflict: (conflictId: string, resolution: ConflictResolution) => Promise<void>;
}
```

### ResMan Tour Booking APIs
```typescript
// Tour booking operations
POST /api/resman/tours/book
PUT /api/resman/tours/:tourId
DELETE /api/resman/tours/:tourId

// Tour availability sync
GET /api/resman/tours/availability/:propertyId
POST /api/resman/tours/sync-availability

// Tour webhook endpoints
POST /api/resman/webhooks/tour-created
POST /api/resman/webhooks/tour-updated
POST /api/resman/webhooks/tour-cancelled

interface ResManTourBookingAPI {
  // Push to ResMan
  bookTour: (tourData: TourBookingData) => Promise<ResManResponse>;
  updateTour: (tourId: string, updates: TourUpdates) => Promise<ResManResponse>;
  cancelTour: (tourId: string, reason: string) => Promise<ResManResponse>;
  
  // Pull from ResMan
  getTourAvailability: (propertyId: string, dateRange: DateRange) => Promise<AvailabilitySlot[]>;
  syncTourChanges: (since: Date) => Promise<TourChange[]>;
}
```

### ResMan Contact Management APIs
```typescript
// Contact sync operations
POST /api/resman/contacts/sync
GET /api/resman/contacts/:contactId
PUT /api/resman/contacts/:contactId
POST /api/resman/contacts/merge

// Contact webhook endpoints
POST /api/resman/webhooks/contact-created
POST /api/resman/webhooks/contact-updated
POST /api/resman/webhooks/contact-merged

interface ResManContactManagementAPI {
  // Push to ResMan
  syncContact: (contactData: ContactData) => Promise<ResManResponse>;
  updateContact: (contactId: string, updates: ContactUpdates) => Promise<ResManResponse>;
  mergeContacts: (primaryId: string, secondaryId: string) => Promise<ResManResponse>;
  
  // Pull from ResMan
  getContact: (contactId: string) => Promise<ContactData>;
  getContactUpdates: (since: Date) => Promise<ContactUpdate[]>;
}
```

### ResMan Unit Availability APIs
```typescript
// Unit availability operations
GET /api/resman/units/availability/:propertyId
GET /api/resman/units/pricing/:propertyId
POST /api/resman/units/sync

// Unit webhook endpoints
POST /api/resman/webhooks/unit-availability-changed
POST /api/resman/webhooks/unit-pricing-changed

interface ResManUnitAvailabilityAPI {
  // Pull from ResMan
  getUnitAvailability: (propertyId: string) => Promise<UnitAvailability[]>;
  getUnitPricing: (propertyId: string) => Promise<UnitPricing[]>;
  syncUnitData: (propertyId: string) => Promise<SyncResult>;
  
  // Scheduled sync
  scheduleSync: (propertyId: string, frequency: 'hourly' | 'daily') => Promise<void>;
}
```

## WebSocket APIs for Real-time Communication

### Real-time Chat APIs
```typescript
// WebSocket connection endpoints
WS /api/ws/chat/:conversationId
WS /api/ws/notifications/:managerId
WS /api/ws/analytics/:propertyId

interface WebSocketAPI {
  // Chat messages
  sendMessage: (conversationId: string, message: ChatMessage) => void;
  receiveMessage: (conversationId: string, callback: (message: ChatMessage) => void) => void;
  
  // Notifications
  sendNotification: (managerId: string, notification: LeadNotification) => void;
  receiveNotifications: (managerId: string, callback: (notification: LeadNotification) => void) => void;
  
  // Analytics
  sendAnalytics: (propertyId: string, analytics: AmplitudeAnalytics) => void;
  receiveAnalytics: (propertyId: string, callback: (analytics: AmplitudeAnalytics) => void) => void;
}
```

## Error Handling & Response Formats

### Standard API Response Format
```typescript
interface APIResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  meta?: {
    pagination?: {
      page: number;
      limit: number;
      total: number;
      totalPages: number;
    };
    timestamp: string;
    requestId: string;
  };
}
```

### Error Codes
```typescript
enum ErrorCodes {
  // Authentication errors
  UNAUTHORIZED = 'UNAUTHORIZED',
  INVALID_TOKEN = 'INVALID_TOKEN',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',
  
  // Authorization errors
  FORBIDDEN = 'FORBIDDEN',
  INSUFFICIENT_PERMISSIONS = 'INSUFFICIENT_PERMISSIONS',
  
  // Validation errors
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  MISSING_REQUIRED_FIELD = 'MISSING_REQUIRED_FIELD',
  INVALID_FORMAT = 'INVALID_FORMAT',
  
  // Database errors
  RECORD_NOT_FOUND = 'RECORD_NOT_FOUND',
  DUPLICATE_RECORD = 'DUPLICATE_RECORD',
  DATABASE_ERROR = 'DATABASE_ERROR',
  
  // Integration errors
  RESMAN_CONNECTION_ERROR = 'RESMAN_CONNECTION_ERROR',
  RESMAN_API_ERROR = 'RESMAN_API_ERROR',
  WEBHOOK_ERROR = 'WEBHOOK_ERROR',
  
  // Business logic errors
  INVALID_STATUS_TRANSITION = 'INVALID_STATUS_TRANSITION',
  CONFLICT_RESOLUTION_REQUIRED = 'CONFLICT_RESOLUTION_REQUIRED',
  TOUR_UNAVAILABLE = 'TOUR_UNAVAILABLE',
}
```

## API Rate Limiting & Security

### Rate Limiting Configuration
```typescript
interface RateLimitConfig {
  windowMs: number; // 15 minutes
  maxRequests: number; // 100 requests per window
  message: string;
  standardHeaders: boolean;
  legacyHeaders: boolean;
}

// Rate limiting by endpoint type
const rateLimits = {
  auth: { windowMs: 15 * 60 * 1000, maxRequests: 5 },
  chat: { windowMs: 60 * 1000, maxRequests: 30 },
  analytics: { windowMs: 60 * 1000, maxRequests: 100 },
  resman: { windowMs: 60 * 1000, maxRequests: 50 },
};
```

### Security Middleware
```typescript
interface SecurityMiddleware {
  // CORS configuration
  cors: {
    origin: string[];
    credentials: boolean;
    methods: string[];
  };
  
  // Helmet security headers
  helmet: {
    contentSecurityPolicy: boolean;
    crossOriginEmbedderPolicy: boolean;
    crossOriginOpenerPolicy: boolean;
    crossOriginResourcePolicy: boolean;
    dnsPrefetchControl: boolean;
    frameguard: boolean;
    hidePoweredBy: boolean;
    hsts: boolean;
    ieNoOpen: boolean;
    noSniff: boolean;
    permittedCrossDomainPolicies: boolean;
    referrerPolicy: boolean;
    xssFilter: boolean;
  };
  
  // API key validation
  validateApiKey: (req: Request, res: Response, next: NextFunction) => void;
  
  // Request sanitization
  sanitizeInput: (req: Request, res: Response, next: NextFunction) => void;
}
```

## API Documentation & Testing

### OpenAPI/Swagger Documentation
```yaml
openapi: 3.0.0
info:
  title: ResMan Integration API
  version: 1.0.0
  description: Comprehensive API for ResMan integration with DomIQ chatbot

servers:
  - url: https://api.domiq.com/v1
    description: Production server
  - url: https://staging-api.domiq.com/v1
    description: Staging server

paths:
  /companies:
    get:
      summary: Get all companies
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 10
      responses:
        '200':
          description: List of companies
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CompanyList'
```

### API Testing Endpoints
```typescript
// Health check endpoints
GET /api/health
GET /api/health/database
GET /api/health/resman
GET /api/health/external-services

// Testing endpoints (development only)
POST /api/test/resman-connection
POST /api/test/webhook-delivery
POST /api/test/email-sending
GET /api/test/analytics-calculation

interface HealthCheckAPI {
  status: 'healthy' | 'degraded' | 'unhealthy';
  checks: {
    database: HealthStatus;
    resman: HealthStatus;
    email: HealthStatus;
    analytics: HealthStatus;
  };
  timestamp: string;
  version: string;
}
```

## Performance Optimization

### Caching Strategy
```typescript
interface CacheConfig {
  // Redis configuration
  redis: {
    host: string;
    port: number;
    password?: string;
    db: number;
  };
  
  // Cache TTL settings
  ttl: {
    property: 3600; // 1 hour
    user: 1800; // 30 minutes
    conversation: 300; // 5 minutes
    faq: 7200; // 2 hours
    analytics: 600; // 10 minutes
  };
  
  // Cache invalidation patterns
  invalidation: {
    property: ['property:*', 'chatbot:*'];
    user: ['user:*', 'conversation:*'];
    conversation: ['conversation:*', 'message:*'];
  };
}
```

### Database Query Optimization
```typescript
interface QueryOptimization {
  // Connection pooling
  pool: {
    min: number;
    max: number;
    acquire: number;
    idle: number;
  };
  
  // Query timeouts
  timeout: {
    select: 5000; // 5 seconds
    insert: 10000; // 10 seconds
    update: 10000; // 10 seconds
    delete: 5000; // 5 seconds
  };
  
  // Batch operations
  batch: {
    maxSize: number;
    timeout: number;
  };
}
```

This backend API component provides comprehensive endpoints for all aspects of the ResMan integration, including data management, external integrations, real-time communication, and performance optimization. 