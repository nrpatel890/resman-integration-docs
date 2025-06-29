# Tour Booking Backend Component: DomIQ ↔ ResMan

## Backend API Endpoints

### 1. Book Tour API
```typescript
// POST /api/resman/tours/book
interface BookTourRequest {
  conversation_id: string;
  tour_data: {
    prospect_name: string;
    prospect_email: string;
    prospect_phone: string;
    tour_type: 'in_person' | 'virtual' | 'self_guided';
    preferred_date: string;
    preferred_time: string;
    property_id: string;
    unit_preferences?: string[];
    special_requests?: string;
  };
}

interface BookTourResponse {
  appointment_id: string;
  confirmed_datetime: string;
  staff_member: string;
  confirmation_details: {
    location?: string;
    virtual_link?: string;
    instructions: string;
  };
}
```

### 2. Availability Check API
```typescript
// GET /api/resman/tours/availability
interface AvailabilityRequest {
  property_id: string;
  date: string;
  tour_type?: 'in_person' | 'virtual' | 'self_guided';
}

interface AvailabilityResponse {
  available_slots: Array<{
    time: string;
    staff_member: string;
    duration: number;
  }>;
  next_available_date?: string;
}
```

### 3. Update Tour API
```typescript
// PUT /api/resman/tours/{appointment_id}
interface UpdateTourRequest {
  status?: 'confirmed' | 'cancelled' | 'completed' | 'no_show';
  rescheduled_datetime?: string;
  staff_notes?: string;
  outcome?: 'interested' | 'not_interested' | 'follow_up_needed';
}
```

### 4. Tour Webhook Receiver
```typescript
// POST /api/resman/webhooks/tour-update
interface TourUpdateWebhook {
  appointment_id: string;
  status: string;
  updated_fields: Record<string, any>;
  staff_member?: string;
  timestamp: string;
}
```

## Services & Logic

### Tour Booking Service
- Check availability (cache, then ResMan API)
- Book tour in ResMan
- Update local DB and sync status
- Send confirmation to prospect
- Notify staff
- Handle errors and retry logic

### Calendar Availability Service
- getAvailability(propertyId, date)
- invalidateCache(propertyId, date)
- fetchFromResMan(propertyId, date)
- cacheAvailability(propertyId, date, slots)

### Tour Status Update Service
- handleTourStatusUpdate(webhookData)
- updateTourStatus(appointment_id, status, staff_notes)
- updateConversationContext(conversation, updates)
- send notifications (confirmation, cancellation, follow-up)

### Data Transformation
- mapChatbotTourToResMan(tourData)
- mapResManStatusToChatbot(resmanStatus)

### Error Handling
- handleAvailabilityError
- handleBookingError
- handleConfirmationError
- retryTourBooking

### Notification System
- sendConfirmation(appointment)
- notifyNewTour(appointment)

### Monitoring & Analytics
- getTourMetrics(propertyId, dateRange)
- getTourTrends(propertyId)

## Testing & QA
- Unit tests for transformation, booking, and notification
- Integration tests for booking flow and webhook
- User acceptance tests for staff and prospect flows 