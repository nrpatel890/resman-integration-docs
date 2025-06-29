# Tour Booking Frontend Component: DomIQ ↔ ResMan

## Overview
This file describes the frontend architecture, UI components, and state management for seamless tour booking and calendar sync between DomIQ and ResMan.

---

## 1. Tour Booking UI

### Tour Booking Modal/Drawer
```typescript
// UI for booking a tour (from chatbot or admin panel)
interface TourBookingModalProps {
  isOpen: boolean;
  onClose: () => void;
  onSubmit: (tourData: TourData) => Promise<void>;
  loading: boolean;
  error?: string;
  availableSlots: TimeSlot[];
  onCheckAvailability: (date: string, tourType: string) => Promise<void>;
}

interface TourData {
  prospectName: string;
  prospectEmail: string;
  prospectPhone: string;
  tourType: 'in_person' | 'virtual' | 'self_guided';
  preferredDate: string;
  preferredTime: string;
  propertyId: string;
  unitPreferences?: string[];
  specialRequests?: string;
}

interface TimeSlot {
  time: string;
  staffMember: string;
  duration: number;
}
```

### Tour Booking Status Indicator
```typescript
// Shows booking/sync status for each tour
interface TourBookingStatusProps {
  status: 'scheduled' | 'confirmed' | 'completed' | 'cancelled' | 'no_show';
  syncStatus: 'pending' | 'success' | 'failed';
  lastUpdated?: string;
  onRetry?: () => void;
  onViewDetails?: () => void;
}
```

### Tour Confirmation Banner
```typescript
// Banner to show confirmation details to prospect
interface TourConfirmationBannerProps {
  appointmentId: string;
  confirmedDatetime: string;
  staffMember: string;
  confirmationDetails: {
    location?: string;
    virtualLink?: string;
    instructions: string;
  };
  onClose: () => void;
}
```

---

## 2. Availability & Calendar UI

### Availability Picker
```typescript
// UI for picking available slots
interface AvailabilityPickerProps {
  propertyId: string;
  date: string;
  tourType: 'in_person' | 'virtual' | 'self_guided';
  availableSlots: TimeSlot[];
  onSelectSlot: (slot: TimeSlot) => void;
  onRefresh: () => void;
  loading: boolean;
}
```

### Calendar View
```typescript
// Calendar view for staff/admins
interface TourCalendarViewProps {
  propertyId: string;
  tours: TourEvent[];
  onDateChange: (date: string) => void;
  onTourClick: (appointmentId: string) => void;
  loading: boolean;
}

interface TourEvent {
  appointmentId: string;
  prospectName: string;
  tourType: string;
  scheduledDatetime: string;
  status: string;
  staffMember: string;
}
```

---

## 3. Tour Status & Notifications

### Tour Status Update Modal
```typescript
// UI for updating tour status (staff/admin)
interface TourStatusUpdateModalProps {
  isOpen: boolean;
  appointmentId: string;
  currentStatus: string;
  onUpdateStatus: (status: string, notes?: string) => Promise<void>;
  loading: boolean;
  error?: string;
}
```

### Staff Notification Banner
```typescript
// Banner to notify staff of new/updated tours
interface StaffNotificationBannerProps {
  notifications: StaffTourNotification[];
  onAcknowledge: (notificationId: string) => void;
}

interface StaffTourNotification {
  id: string;
  appointmentId: string;
  prospectName: string;
  tourType: string;
  scheduledDatetime: string;
  status: string;
  staffMember: string;
}
```

---

## 4. State Management & Hooks

### Tour Booking State Slice (Redux or Zustand)
```typescript
interface TourBookingState {
  tours: TourEvent[];
  availability: Record<string, TimeSlot[]>;
  bookingStatus: Record<string, string>;
  confirmations: Record<string, TourConfirmationBannerProps>;
  notifications: StaffTourNotification[];
  loading: boolean;
  error?: string;
}
```

### Tour Booking Hooks
```typescript
// Hook to manage tour booking
function useTourBooking() {
  // bookTour, checkAvailability, updateTourStatus, fetchTours, etc.
}

// Hook for calendar availability
function useCalendarAvailability() {
  // getAvailability, refreshAvailability, invalidateCache, etc.
}

// Hook for staff notifications
function useStaffTourNotifications() {
  // fetchNotifications, acknowledgeNotification, etc.
}
```

---

## 5. Error Handling & Alerts

### Booking Error Alert
```typescript
// Alert for booking/sync errors
interface BookingErrorAlertProps {
  error: string;
  onRetry?: () => void;
  onClose: () => void;
}
```

### Alerting Integration
- Integrate with notification system (toast, snackbar, or modal)
- Show alerts for:
  - Booking failures
  - Double-booking detected
  - Confirmation delivery issues
  - Calendar sync errors

---

## 6. Integration Points

- **TourBookingModal.tsx**: Trigger booking, check availability, show confirmation
- **TourCalendarView**: Show all tours, allow status updates
- **StaffNotificationBanner**: Notify staff of new/updated tours
- **AvailabilityPicker**: Let users select from available slots
- **TourStatusUpdateModal**: Allow staff to update/correct tour status

---

## 7. Monitoring & Analytics

### Tour Analytics Widget
```typescript
// Widget for displaying tour metrics
interface TourAnalyticsWidgetProps {
  metrics: TourMetrics;
  loading: boolean;
  onRefresh: () => void;
}

interface TourMetrics {
  totalBookings: number;
  confirmedTours: number;
  completedTours: number;
  averageBookingTime: number;
  staffUtilization: number;
  conversionRate: number;
}
```

---

## 8. Testing & QA

- Unit tests for all hooks and UI components
- Integration tests for booking, availability, and notification flows
- UI tests for error and alert handling

---

This frontend component provides a complete UI and state management solution for tour booking, calendar sync, confirmation, and monitoring between DomIQ and ResMan. 