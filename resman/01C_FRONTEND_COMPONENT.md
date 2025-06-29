# Frontend Component: ResMan Integration

## Frontend Architecture Overview

The frontend component provides a comprehensive user interface for managing the ResMan integration system. It includes dashboard views, configuration panels, analytics displays, and real-time monitoring capabilities for property managers and administrators.

## Core Dashboard Components

### Main Dashboard Layout
```typescript
// Dashboard layout structure
interface DashboardLayout {
  header: DashboardHeader;
  sidebar: DashboardSidebar;
  main: DashboardMain;
  footer: DashboardFooter;
}

// Dashboard header with navigation and user info
interface DashboardHeader {
  logo: string;
  companyName: string;
  userProfile: UserProfile;
  notifications: NotificationCenter;
  search: GlobalSearch;
}

// Dashboard sidebar with navigation menu
interface DashboardSidebar {
  navigation: NavigationMenu;
  quickActions: QuickActionPanel;
  propertySelector: PropertySelector;
}
```

### Property Management Dashboard
```typescript
// Property overview dashboard
interface PropertyDashboard {
  // Property information
  propertyInfo: {
    name: string;
    address: string;
    city: string;
    state: string;
    zipCode: string;
    propertyType: string;
    unitsCount: number;
    amenities: string[];
    features: string[];
    websiteUrl: string;
  };
  
  // Chatbot configuration
  chatbotConfig: {
    isActive: boolean;
    avatarUrl: string;
    welcomeMessage: string;
    themeColor: string;
    position: 'bottom-right' | 'bottom-left' | 'top-right' | 'top-left';
    autoOpen: boolean;
    businessHours: BusinessHours;
    widgetSettings: WidgetSettings;
  };
  
  // Integration status
  integrations: {
    resman: IntegrationStatus;
    hubspot: IntegrationStatus;
    calendly: IntegrationStatus;
    sendgrid: IntegrationStatus;
    amplitude: IntegrationStatus;
  };
}
```

## Lead Management Components

### Lead Overview Dashboard
```typescript
// Lead management main component
interface LeadOverview {
  // Lead statistics
  stats: {
    totalLeads: number;
    qualifiedLeads: number;
    toursBooked: number;
    conversions: number;
    averageResponseTime: number;
  };
  
  // Lead funnel visualization
  funnel: LeadFunnel;
  
  // Recent leads table
  recentLeads: LeadTable;
  
  // Lead status distribution
  statusDistribution: StatusChart;
}

// Lead funnel component
interface LeadFunnel {
  stages: {
    initial: number;
    qualified: number;
    tourScheduled: number;
    tourCompleted: number;
    application: number;
    lease: number;
  };
  
  conversionRates: {
    initialToQualified: number;
    qualifiedToTour: number;
    tourToApplication: number;
    applicationToLease: number;
  };
}

// Lead table component
interface LeadTable {
  columns: {
    name: string;
    email: string;
    phone: string;
    source: string;
    status: string;
    leadScore: number;
    lastActivity: string;
    assignedTo: string;
    actions: string;
  };
  
  data: LeadRecord[];
  pagination: Pagination;
  filters: LeadFilters;
  sorting: SortConfig;
}

interface LeadRecord {
  id: string;
  name: string;
  email: string;
  phone: string;
  source: string;
  status: LeadStatus;
  leadScore: number;
  lastActivity: Date;
  assignedTo: string;
  conversationId: string;
  propertyId: string;
}
```

### Individual Lead Management
```typescript
// Individual lead detail view
interface LeadDetail {
  // Lead information
  leadInfo: {
    id: string;
    name: string;
    email: string;
    phone: string;
    age?: number;
    leadSource: string;
    createdAt: Date;
    lastSeen: Date;
    sessionCount: number;
  };
  
  // Conversation history
  conversation: {
    messages: ChatMessage[];
    aiIntentSummary: string;
    apartmentSizePreference: string;
    moveInDate: Date;
    priceRange: { min: number; max: number };
    occupantsCount: number;
    hasPets: boolean;
    petDetails: PetDetails;
    desiredFeatures: string[];
    workLocation: string;
    reasonForMoving: string;
  };
  
  // Lead qualification data
  qualification: {
    isQualified: boolean;
    isBookTour: boolean;
    tourType: string;
    tourDateTime: Date;
    preQualified: boolean;
    leadScore: number;
  };
  
  // Activity timeline
  timeline: ActivityTimeline;
  
  // Actions panel
  actions: LeadActions;
}

// Activity timeline component
interface ActivityTimeline {
  events: TimelineEvent[];
  filters: TimelineFilters;
}

interface TimelineEvent {
  id: string;
  type: 'message' | 'tour_booked' | 'status_change' | 'notification' | 'email_sent';
  timestamp: Date;
  description: string;
  metadata: any;
  user: string;
}
```

## Conversation Management Components

### Chat Interface
```typescript
// Chat interface for managing conversations
interface ChatInterface {
  // Conversation list
  conversationList: {
    conversations: ConversationItem[];
    filters: ConversationFilters;
    search: string;
    sortBy: 'latest' | 'oldest' | 'status' | 'leadScore';
  };
  
  // Active conversation
  activeConversation: {
    id: string;
    user: UserInfo;
    messages: ChatMessage[];
    status: ConversationStatus;
    leadData: LeadQualificationData;
    actions: ConversationActions;
  };
  
  // Message composition
  messageComposer: {
    text: string;
    attachments: File[];
    quickReplies: QuickReply[];
    send: () => void;
  };
}

interface ChatMessage {
  id: string;
  senderType: 'user' | 'bot';
  messageText: string;
  timestamp: Date;
  messageType: 'text' | 'image' | 'file' | 'quick_reply';
  metadata: any;
}

interface ConversationActions {
  qualify: () => void;
  bookTour: () => void;
  assignToManager: (managerId: string) => void;
  sendEmail: () => void;
  escalate: () => void;
  close: () => void;
}
```

### FAQ Management Interface
```typescript
// FAQ management component
interface FAQManagement {
  // FAQ list
  faqList: {
    faqs: FAQItem[];
    filters: FAQFilters;
    search: string;
    categories: string[];
  };
  
  // FAQ editor
  faqEditor: {
    question: string;
    answer: string;
    category: string;
    sourceType: string;
    status: 'pending' | 'answered' | 'needs_review';
    assignedTo: string;
    save: () => void;
    publish: () => void;
  };
  
  // Human-in-loop workflow
  workflow: {
    pendingQuestions: FAQItem[];
    assignedQuestions: FAQItem[];
    needsReview: FAQItem[];
    assignQuestion: (faqId: string, managerId: string) => void;
    markAnswered: (faqId: string, answer: string) => void;
    markForReview: (faqId: string, reason: string) => void;
  };
}

interface FAQItem {
  id: string;
  propertyId: string;
  question: string;
  answer?: string;
  category: string;
  sourceType: string;
  status: 'pending' | 'answered' | 'needs_review';
  assignedTo?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

## Analytics & Reporting Components

### Analytics Dashboard
```typescript
// Analytics main dashboard
interface AnalyticsDashboard {
  // Key performance indicators
  kpis: {
    totalConversations: number;
    qualifiedLeads: number;
    tourBookings: number;
    conversionRate: number;
    averageSessionDuration: number;
    engagementScore: number;
  };
  
  // Engagement metrics
  engagement: {
    chatSessionStarted: number;
    userMessagesSent: number;
    botMessagesReceived: number;
    answerButtonClicks: number;
    contactCaptured: number;
    tourBooked: number;
    emailOfficeClicked: number;
    phoneCallClicked: number;
    incentiveOffered: number;
    incentiveAccepted: number;
    adminHandoffTriggered: number;
    customerServiceEscalated: number;
    conversationAbandoned: number;
    widgetSessionEnded: number;
    widgetMinimized: number;
  };
  
  // Charts and visualizations
  charts: {
    conversationTrends: LineChart;
    leadQualificationRates: BarChart;
    tourBookingTrends: LineChart;
    engagementScores: Heatmap;
    sourceDistribution: PieChart;
    timeOfDayActivity: BarChart;
  };
}

// Chart components
interface LineChart {
  data: ChartDataPoint[];
  xAxis: string;
  yAxis: string;
  title: string;
  color: string;
}

interface BarChart {
  data: ChartDataPoint[];
  categories: string[];
  values: number[];
  title: string;
  colors: string[];
}

interface PieChart {
  data: { label: string; value: number; color: string }[];
  title: string;
}

interface Heatmap {
  data: { x: string; y: string; value: number }[];
  xCategories: string[];
  yCategories: string[];
  title: string;
}
```

### Deep Insights Component
```typescript
// Deep insights and AI analysis
interface DeepInsights {
  // AI-generated insights
  insights: {
    leadBehavior: LeadBehaviorInsights;
    conversationPatterns: ConversationPatternInsights;
    conversionOptimization: ConversionOptimizationInsights;
    performanceTrends: PerformanceTrendInsights;
  };
  
  // Predictive analytics
  predictions: {
    leadScoring: LeadScoringPredictions;
    conversionProbability: ConversionProbabilityPredictions;
    churnRisk: ChurnRiskPredictions;
    revenueForecast: RevenueForecastPredictions;
  };
  
  // Recommendations
  recommendations: {
    conversationImprovements: ConversationRecommendation[];
    leadQualification: QualificationRecommendation[];
    tourScheduling: SchedulingRecommendation[];
    followUpStrategies: FollowUpRecommendation[];
  };
}

interface LeadBehaviorInsights {
  peakActivityHours: string[];
  preferredContactMethods: { method: string; percentage: number }[];
  commonQuestions: { question: string; frequency: number }[];
  decisionFactors: { factor: string; importance: number }[];
}

interface ConversationPatternInsights {
  averageMessagesPerSession: number;
  commonConversationFlows: ConversationFlow[];
  responseTimeAnalysis: ResponseTimeAnalysis;
  escalationTriggers: EscalationTrigger[];
}
```

## Email Management Components

### Email Dashboard
```typescript
// Email management interface
interface EmailDashboard {
  // Email drafts
  drafts: {
    drafts: EmailDraft[];
    filters: DraftFilters;
    search: string;
    sortBy: 'created' | 'updated' | 'priority' | 'status';
  };
  
  // Email campaigns
  campaigns: {
    campaigns: EmailCampaign[];
    templates: EmailTemplate[];
    analytics: CampaignAnalytics;
  };
  
  // Email composer
  composer: EmailComposer;
}

interface EmailDraft {
  id: string;
  leadId: string;
  conversationId: string;
  subject: string;
  content: string;
  preview: string;
  status: 'draft' | 'scheduled' | 'sent' | 'failed';
  priority: 'low' | 'medium' | 'high';
  tags: string[];
  scheduledFor?: Date;
  sentAt?: Date;
  createdAt: Date;
}

interface EmailComposer {
  to: string;
  subject: string;
  content: string;
  template?: EmailTemplate;
  attachments: File[];
  scheduledFor?: Date;
  priority: 'low' | 'medium' | 'high';
  tags: string[];
  preview: () => void;
  save: () => void;
  send: () => void;
  schedule: () => void;
}
```

## Tour Management Components

### Tour Calendar Interface
```typescript
// Tour management interface
interface TourCalendar {
  // Calendar view
  calendar: {
    view: 'day' | 'week' | 'month';
    events: TourEvent[];
    availability: AvailabilitySlot[];
    selectedDate: Date;
  };
  
  // Tour list
  tours: {
    tours: TourItem[];
    filters: TourFilters;
    search: string;
    sortBy: 'datetime' | 'status' | 'prospect' | 'manager';
  };
  
  // Tour booking form
  bookingForm: TourBookingForm;
}

interface TourEvent {
  id: string;
  title: string;
  start: Date;
  end: Date;
  type: 'scheduled' | 'confirmed' | 'completed' | 'cancelled';
  prospect: string;
  manager: string;
  unit: string;
  color: string;
}

interface TourBookingForm {
  prospectName: string;
  prospectEmail: string;
  prospectPhone: string;
  unitInterest: string;
  tourDateTime: Date;
  durationMinutes: number;
  tourType: 'in_person' | 'virtual' | 'self_guided';
  assignedManager: string;
  notes: string;
  book: () => void;
  reschedule: () => void;
  cancel: () => void;
}
```

## Task Management Components

### Task Management Interface
```typescript
// Task management interface
interface TaskManagement {
  // Task list
  tasks: {
    tasks: TaskItem[];
    filters: TaskFilters;
    search: string;
    sortBy: 'created' | 'due' | 'priority' | 'status';
  };
  
  // Task board (Kanban)
  board: {
    columns: {
      pending: TaskItem[];
      inProgress: TaskItem[];
      completed: TaskItem[];
      cancelled: TaskItem[];
    };
    dragAndDrop: boolean;
  };
  
  // Task form
  taskForm: TaskForm;
}

interface TaskItem {
  id: string;
  propertyId: string;
  conversationId?: string;
  assignedTo?: string;
  question: string;
  answer?: string;
  status: 'pending' | 'in_progress' | 'completed' | 'cancelled';
  priority: 'low' | 'medium' | 'high';
  dueDate?: Date;
  createdAt: Date;
  updatedAt: Date;
}

interface TaskForm {
  question: string;
  answer?: string;
  assignedTo?: string;
  priority: 'low' | 'medium' | 'high';
  dueDate?: Date;
  save: () => void;
  assign: () => void;
  complete: () => void;
}
```

## Settings & Configuration Components

### Integration Settings
```typescript
// Integration configuration interface
interface IntegrationSettings {
  // ResMan integration
  resman: {
    enabled: boolean;
    apiKey: string;
    baseUrl: string;
    webhookUrl: string;
    syncFrequency: 'realtime' | 'hourly' | 'daily';
    testConnection: () => void;
    save: () => void;
  };
  
  // HubSpot integration
  hubspot: {
    enabled: boolean;
    apiKey: string;
    portalId: string;
    syncContacts: boolean;
    syncCompanies: boolean;
    syncDeals: boolean;
    testConnection: () => void;
    save: () => void;
  };
  
  // Calendly integration
  calendly: {
    enabled: boolean;
    orgUri: string;
    webhookUrl: string;
    syncAvailability: boolean;
    testConnection: () => void;
    save: () => void;
  };
  
  // SendGrid integration
  sendgrid: {
    enabled: boolean;
    apiKey: string;
    fromEmail: string;
    fromName: string;
    testConnection: () => void;
    save: () => void;
  };
  
  // Amplitude integration
  amplitude: {
    enabled: boolean;
    apiKey: string;
    secretKey: string;
    projectId: string;
    trackEvents: boolean;
    testConnection: () => void;
    save: () => void;
  };
}
```

### Chatbot Configuration
```typescript
// Chatbot settings interface
interface ChatbotSettings {
  // Basic settings
  basic: {
    name: string;
    avatarUrl: string;
    welcomeMessage: string;
    isActive: boolean;
  };
  
  // Appearance settings
  appearance: {
    themeColor: string;
    position: 'bottom-right' | 'bottom-left' | 'top-right' | 'top-left';
    autoOpen: boolean;
    showAvatar: boolean;
    showWelcomeMessage: boolean;
  };
  
  // Business hours
  businessHours: {
    enabled: boolean;
    timezone: string;
    hours: {
      monday: { start: string; end: string; enabled: boolean };
      tuesday: { start: string; end: string; enabled: boolean };
      wednesday: { start: string; end: string; enabled: boolean };
      thursday: { start: string; end: string; enabled: boolean };
      friday: { start: string; end: string; enabled: boolean };
      saturday: { start: string; end: string; enabled: boolean };
      sunday: { start: string; end: string; enabled: boolean };
    };
    offlineMessage: string;
  };
  
  // Widget settings
  widget: {
    embedCode: string;
    trackingId: string;
    customCSS: string;
    customJS: string;
  };
}
```

## Real-time Components

### Real-time Notifications
```typescript
// Real-time notification system
interface RealTimeNotifications {
  // Notification center
  notificationCenter: {
    notifications: Notification[];
    unreadCount: number;
    markAsRead: (id: string) => void;
    markAllAsRead: () => void;
    clear: () => void;
  };
  
  // Live updates
  liveUpdates: {
    newLeads: LeadNotification[];
    tourBookings: TourNotification[];
    messageAlerts: MessageNotification[];
    systemAlerts: SystemNotification[];
  };
  
  // WebSocket connection
  websocket: {
    connected: boolean;
    reconnect: () => void;
    subscribe: (channel: string) => void;
    unsubscribe: (channel: string) => void;
  };
}

interface Notification {
  id: string;
  type: 'lead' | 'tour' | 'message' | 'system';
  title: string;
  message: string;
  timestamp: Date;
  read: boolean;
  actionUrl?: string;
  metadata?: any;
}
```

### Live Chat Widget
```typescript
// Live chat widget component
interface LiveChatWidget {
  // Widget state
  state: {
    isOpen: boolean;
    isMinimized: boolean;
    isTyping: boolean;
    connectionStatus: 'connected' | 'connecting' | 'disconnected';
  };
  
  // Chat interface
  chat: {
    messages: ChatMessage[];
    inputValue: string;
    sendMessage: (message: string) => void;
    sendQuickReply: (reply: string) => void;
    uploadFile: (file: File) => void;
  };
  
  // Widget controls
  controls: {
    open: () => void;
    close: () => void;
    minimize: () => void;
    maximize: () => void;
  };
  
  // Widget configuration
  config: {
    themeColor: string;
    position: string;
    avatarUrl: string;
    welcomeMessage: string;
    businessHours: BusinessHours;
  };
}
```

## State Management

### Redux Store Structure
```typescript
// Main store structure
interface AppState {
  // Authentication
  auth: {
    user: User | null;
    isAuthenticated: boolean;
    loading: boolean;
    error: string | null;
  };
  
  // Properties
  properties: {
    list: Property[];
    current: Property | null;
    loading: boolean;
    error: string | null;
  };
  
  // Leads
  leads: {
    list: Lead[];
    current: Lead | null;
    filters: LeadFilters;
    loading: boolean;
    error: string | null;
  };
  
  // Conversations
  conversations: {
    list: Conversation[];
    current: Conversation | null;
    messages: ChatMessage[];
    loading: boolean;
    error: string | null;
  };
  
  // Analytics
  analytics: {
    data: AnalyticsData;
    loading: boolean;
    error: string | null;
  };
  
  // Settings
  settings: {
    integrations: IntegrationSettings;
    chatbot: ChatbotSettings;
    loading: boolean;
    error: string | null;
  };
  
  // UI state
  ui: {
    sidebarOpen: boolean;
    theme: 'light' | 'dark';
    notifications: Notification[];
    modals: ModalState[];
  };
}
```

### Context Providers
```typescript
// Context providers for state management
interface ContextProviders {
  // Property context
  PropertyProvider: {
    currentProperty: Property | null;
    setCurrentProperty: (property: Property) => void;
    properties: Property[];
    loading: boolean;
  };
  
  // User context
  UserProvider: {
    currentUser: User | null;
    setCurrentUser: (user: User) => void;
    loading: boolean;
  };
  
  // Theme context
  ThemeProvider: {
    theme: 'light' | 'dark';
    toggleTheme: () => void;
  };
  
  // Notification context
  NotificationProvider: {
    notifications: Notification[];
    addNotification: (notification: Notification) => void;
    removeNotification: (id: string) => void;
    clearNotifications: () => void;
  };
}
```

## Responsive Design & Accessibility

### Responsive Breakpoints
```typescript
// Responsive design configuration
interface ResponsiveConfig {
  breakpoints: {
    xs: '0px';
    sm: '576px';
    md: '768px';
    lg: '992px';
    xl: '1200px';
    xxl: '1400px';
  };
  
  // Mobile-first approach
  mobile: {
    sidebarCollapsed: boolean;
    tableScrollable: boolean;
    chartResponsive: boolean;
  };
  
  // Tablet optimizations
  tablet: {
    sidebarCollapsible: boolean;
    tablePagination: boolean;
    chartLegend: boolean;
  };
  
  // Desktop enhancements
  desktop: {
    sidebarExpanded: boolean;
    tableFull: boolean;
    chartDetailed: boolean;
  };
}
```

### Accessibility Features
```typescript
// Accessibility configuration
interface AccessibilityConfig {
  // ARIA labels
  ariaLabels: {
    navigation: string;
    search: string;
    notifications: string;
    userMenu: string;
  };
  
  // Keyboard navigation
  keyboardNav: {
    enabled: boolean;
    shortcuts: KeyboardShortcut[];
  };
  
  // Screen reader support
  screenReader: {
    announcements: string[];
    landmarks: Landmark[];
  };
  
  // Color contrast
  colorContrast: {
    enabled: boolean;
    minimumRatio: number;
  };
  
  // Focus management
  focusManagement: {
    trapFocus: boolean;
    restoreFocus: boolean;
    focusIndicator: boolean;
  };
}

interface KeyboardShortcut {
  key: string;
  action: string;
  description: string;
  global: boolean;
}
```

## Performance Optimization

### Component Optimization
```typescript
// Performance optimization strategies
interface PerformanceOptimization {
  // Code splitting
  codeSplitting: {
    routes: Route[];
    components: Component[];
    libraries: Library[];
  };
  
  // Lazy loading
  lazyLoading: {
    images: boolean;
    components: boolean;
    data: boolean;
  };
  
  // Caching
  caching: {
    apiResponses: boolean;
    staticAssets: boolean;
    userPreferences: boolean;
  };
  
  // Virtual scrolling
  virtualScrolling: {
    tables: boolean;
    lists: boolean;
    charts: boolean;
  };
  
  // Memoization
  memoization: {
    expensiveCalculations: boolean;
    componentRenders: boolean;
    dataTransformations: boolean;
  };
}
```

### Bundle Optimization
```typescript
// Bundle optimization configuration
interface BundleOptimization {
  // Tree shaking
  treeShaking: {
    enabled: boolean;
    libraries: string[];
  };
  
  // Minification
  minification: {
    enabled: boolean;
    removeComments: boolean;
    removeConsole: boolean;
  };
  
  // Compression
  compression: {
    gzip: boolean;
    brotli: boolean;
  };
  
  // CDN
  cdn: {
    enabled: boolean;
    domain: string;
    assets: string[];
  };
}
```

This frontend component provides a comprehensive user interface for managing all aspects of the ResMan integration, including lead management, conversation handling, analytics, email management, tour scheduling, and system configuration. 