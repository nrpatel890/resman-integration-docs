# Tour Booking Integration: DomIQ ↔ ResMan

## Feature Overview
Seamless tour scheduling between DomIQ AI chatbot and ResMan calendar/appointment system.

## What It Delivers for On-site Teams
- **Automated Tour Scheduling**: Chatbot books tours directly in ResMan calendar
- **Real-time Availability**: Live calendar sync prevents double-booking
- **Tour Confirmation**: Automated confirmations sent to prospects
- **Staff Notifications**: Instant alerts when tours are booked
- **Tour Follow-up**: Automated reminders and follow-up sequences
- **Tour Outcomes**: Track tour completion and next steps

## Data Flow Architecture

### 1. Tour Booking Flow (DomIQ → ResMan)
```
Chatbot Tour Request → Availability Check → ResMan Calendar → Appointment Creation → Confirmation
```

### 2. Tour Update Flow (ResMan → DomIQ)
```
ResMan Tour Change → Webhook → Chatbot Context Update → Prospect Notification
```

## Implementation Requirements

### Database Schema Updates
```sql
-- Add tour booking integration tables
CREATE TABLE tour_booking_sync (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id),
    resman_appointment_id VARCHAR(100),
    tour_type VARCHAR(50), -- 'in_person', 'virtual', 'self_guided'
    scheduled_datetime TIMESTAMPTZ,
    status VARCHAR(50), -- 'scheduled', 'confirmed', 'completed', 'cancelled', 'no_show'
    prospect_info JSONB,
    staff_notes TEXT,
    sync_status VARCHAR(50), -- 'pending', 'success', 'failed'
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE calendar_availability_cache (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id),
    date DATE,
    available_slots JSONB, -- Array of available time slots
    last_updated TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(property_id, date)
);
```

### API Endpoints to Create

#### 1. Tour Booking API
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

#### 2. Availability Check API
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
    duration: number; // minutes
  }>;
  next_available_date?: string;
}
```

#### 3. Tour Update API
```typescript
// PUT /api/resman/tours/{appointment_id}
interface UpdateTourRequest {
  status?: 'confirmed' | 'cancelled' | 'completed' | 'no_show';
  rescheduled_datetime?: string;
  staff_notes?: string;
  outcome?: 'interested' | 'not_interested' | 'follow_up_needed';
}
```

#### 4. Tour Webhook Receiver
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

## Integration Points

### 1. Chatbot Tour Booking Flow
```typescript
// In TourBookingModal.tsx
const handleTourBooking = async (tourData: TourData) => {
  try {
    // Check availability first
    const availability = await checkResManAvailability(tourData);
    
    if (!availability.available_slots.length) {
      // Suggest alternative dates/times
      return suggestAlternativeSlots(tourData);
    }
    
    // Book tour in ResMan
    const appointment = await bookTourInResMan(tourData);
    
    // Update conversation context
    await updateConversationWithTour(appointment);
    
    // Send confirmation to prospect
    await sendTourConfirmation(appointment);
    
    // Notify staff
    await notifyStaffOfNewTour(appointment);
    
  } catch (error) {
    handleTourBookingError(error);
  }
};
```

### 2. Real-time Calendar Sync
```typescript
// Calendar availability cache service
class CalendarAvailabilityService {
  async getAvailability(propertyId: string, date: string): Promise<TimeSlot[]> {
    // Check cache first
    const cached = await this.getCachedAvailability(propertyId, date);
    if (cached && this.isCacheValid(cached)) {
      return cached.available_slots;
    }
    
    // Fetch from ResMan API
    const availability = await this.fetchFromResMan(propertyId, date);
    
    // Cache the result
    await this.cacheAvailability(propertyId, date, availability);
    
    return availability;
  }
  
  async invalidateCache(propertyId: string, date: string) {
    await this.clearCachedAvailability(propertyId, date);
  }
}
```

### 3. Tour Status Updates
```typescript
// Handle ResMan tour status changes
const handleTourStatusUpdate = async (webhookData: TourUpdateWebhook) => {
  const { appointment_id, status, staff_notes } = webhookData;
  
  // Update local tour record
  await updateTourStatus(appointment_id, status, staff_notes);
  
  // Update conversation context
  const conversation = await getConversationByTourId(appointment_id);
  if (conversation) {
    await updateConversationContext(conversation, {
      tour_status: status,
      staff_notes,
      last_updated: new Date()
    });
  }
  
  // Send notifications based on status
  switch (status) {
    case 'confirmed':
      await sendTourConfirmation(appointment_id);
      break;
    case 'cancelled':
      await sendTourCancellation(appointment_id);
      break;
    case 'completed':
      await sendTourFollowUp(appointment_id);
      break;
  }
};
```

## Data Transformation

### DomIQ → ResMan Tour Mapping
```typescript
interface TourDataMapper {
  mapChatbotTourToResMan(tourData: ChatbotTourData): ResManAppointment {
    return {
      prospect_name: `${tourData.first_name} ${tourData.last_name}`,
      prospect_email: tourData.email,
      prospect_phone: tourData.phone,
      appointment_type: this.mapTourType(tourData.tour_type),
      scheduled_datetime: tourData.preferred_datetime,
      duration: this.getTourDuration(tourData.tour_type),
      property_id: tourData.property_id,
      source: 'chatbot',
      notes: this.formatTourNotes(tourData)
    };
  }
  
  private mapTourType(chatbotType: string): string {
    const typeMap = {
      'in_person': 'In-Person Tour',
      'virtual': 'Virtual Tour',
      'self_guided': 'Self-Guided Tour'
    };
    return typeMap[chatbotType] || 'In-Person Tour';
  }
}
```

### ResMan → DomIQ Status Mapping
```typescript
interface StatusMapper {
  mapResManStatusToChatbot(resmanStatus: string): ChatbotTourStatus {
    const statusMap = {
      'Scheduled': 'scheduled',
      'Confirmed': 'confirmed',
      'Completed': 'completed',
      'Cancelled': 'cancelled',
      'No Show': 'no_show'
    };
    return statusMap[resmanStatus] || 'scheduled';
  }
}
```

## Error Handling

### Tour Booking Errors
```typescript
interface TourBookingErrorHandler {
  handleAvailabilityError(error: Error): Promise<AlternativeSlots> {
    // Suggest alternative dates/times
    return this.suggestAlternatives();
  }
  
  handleBookingError(error: Error): Promise<BookingFallback> {
    // Create local booking and queue for manual sync
    return this.createLocalBooking();
  }
  
  handleConfirmationError(error: Error): Promise<void> {
    // Queue confirmation for retry
    return this.queueConfirmationRetry();
  }
}
```

### Retry Logic
```typescript
const retryTourBooking = async (tourData: TourData, maxAttempts = 3) => {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await bookTourInResMan(tourData);
    } catch (error) {
      if (attempt === maxAttempts) {
        // Create local booking as fallback
        return await createLocalTourBooking(tourData);
      }
      
      // Wait before retry
      await delay(1000 * attempt);
    }
  }
};
```

## Notification System

### Tour Confirmations
```typescript
interface TourConfirmationService {
  async sendConfirmation(appointment: ResManAppointment): Promise<void> {
    const emailData = {
      to: appointment.prospect_email,
      subject: `Tour Confirmed - ${appointment.property_name}`,
      template: 'tour_confirmation',
      data: {
        prospect_name: appointment.prospect_name,
        tour_datetime: appointment.scheduled_datetime,
        location: appointment.location,
        virtual_link: appointment.virtual_link,
        staff_member: appointment.staff_member,
        instructions: appointment.instructions
      }
    };
    
    await this.emailService.send(emailData);
  }
}
```

### Staff Notifications
```typescript
interface StaffNotificationService {
  async notifyNewTour(appointment: ResManAppointment): Promise<void> {
    const notification = {
      type: 'new_tour_booking',
      staff_member: appointment.staff_member,
      data: {
        prospect_name: appointment.prospect_name,
        tour_datetime: appointment.scheduled_datetime,
        tour_type: appointment.appointment_type,
        property_name: appointment.property_name
      }
    };
    
    await this.sendStaffNotification(notification);
  }
}
```

## Monitoring & Analytics

### Key Metrics
- Tour booking success rate
- Average booking completion time
- Tour confirmation delivery rate
- Staff response time to new tours
- Tour completion rate
- Prospect satisfaction scores

### Dashboard Widgets
```typescript
interface TourAnalytics {
  getTourMetrics(propertyId: string, dateRange: DateRange): Promise<{
    total_bookings: number;
    confirmed_tours: number;
    completed_tours: number;
    average_booking_time: number;
    staff_utilization: number;
  }>;
  
  getTourTrends(propertyId: string): Promise<{
    daily_bookings: Array<{date: string, count: number}>;
    tour_type_distribution: Record<string, number>;
    conversion_rate: number;
  }>;
}
```

## Testing Strategy

### Unit Tests
- Tour data transformation functions
- Availability calculation logic
- Notification sending
- Error handling scenarios

### Integration Tests
- End-to-end tour booking flow
- Calendar availability sync
- Webhook processing
- Email confirmation delivery

### User Acceptance Tests
- Staff booking workflow
- Prospect tour experience
- Calendar management
- Notification delivery

## Deployment Checklist

### Pre-deployment
- [ ] ResMan calendar API access configured
- [ ] Webhook endpoints registered
- [ ] Email templates created
- [ ] Staff notification preferences set
- [ ] Calendar availability cache tested

### Post-deployment
- [ ] Verify tour booking functionality
- [ ] Test calendar sync accuracy
- [ ] Validate email confirmations
- [ ] Check staff notifications
- [ ] Monitor booking success rates

## Success Criteria
- [ ] 95%+ tour booking success rate
- [ ] <2 minute booking completion time
- [ ] 100% confirmation email delivery
- [ ] Real-time calendar availability
- [ ] Zero double-bookings
- [ ] Staff satisfaction >90% 