# Database Schema Component: ResMan Integration

## Database Hierarchy Overview

```
company (Greystar)
├── property (company_id FK) ## (Property ID as chatbot ID) <-> Sync with Milvus --> Automatic Milvus addup. 
│   ├── conversation (property_id FK) 
│   │   ├── message (conversation_id FK) (Delete) 
│   │   └── lead_notification (conversation_id FK)
│   ├── faq (property_id FK)
│   └── website_integration (property_id FK)
└── property_manager (company_id FK)
    └── property_manager_assignment (property_manager_id FK, property_id FK)
        └── lead_notification (property_manager_id FK)

user (user persistence frontend --> Each browser unique id, can use it to track the user)
└── conversation (user_id FK)
    ├── message (conversation_id FK)
    └── lead_notification (conversation_id FK)
```

## Core Database Tables

### Company Table
```sql
CREATE TABLE company (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL UNIQUE,
    logo_url VARCHAR(255),
    contact_email VARCHAR(255),
    contact_phone VARCHAR(20),
    hubspot_company_id VARCHAR(100),
    -- Integration API Keys
    calendly_org_uri VARCHAR(255),
    sendgrid_api_key VARCHAR(255),
    amplitude_api_key VARCHAR(255),
    openai_api_key VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Property Table
```sql
CREATE TABLE property (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID REFERENCES company(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    address VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50) NOT NULL,
    zip_code VARCHAR(20) NOT NULL,
    property_type VARCHAR(50),
    units_count INTEGER,
    amenities JSONB,
    features JSONB,
    website_url VARCHAR(255),
    hubspot_property_id VARCHAR(100),
    -- Chatbot Configuration
    chatbot_avatar_url VARCHAR(255),
    chatbot_is_active BOOLEAN DEFAULT TRUE,
    chatbot_welcome_message TEXT,
    chatbot_embed_code TEXT,
    chatbot_widget_settings JSONB,
    chatbot_theme_color VARCHAR(7) DEFAULT '#7c3aed',
    chatbot_position VARCHAR(20) DEFAULT 'bottom-right',
    chatbot_auto_open BOOLEAN DEFAULT FALSE,
    chatbot_business_hours JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Property Manager Table
```sql
CREATE TABLE property_manager (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id UUID REFERENCES company(id) ON DELETE CASCADE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL UNIQUE,
    role VARCHAR(100),
    access_level VARCHAR(50) NOT NULL DEFAULT 'read'
        CHECK (access_level IN ('write','read')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Property Manager Assignment Table
```sql
CREATE TABLE property_manager_assignment (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    property_manager_id UUID REFERENCES property_manager(id) ON DELETE CASCADE,
    is_primary BOOLEAN DEFAULT FALSE,
    start_date DATE NOT NULL,
    end_date DATE,
    permissions JSONB,
    notification_preferences JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT chk_assignment_dates 
        CHECK (end_date IS NULL OR end_date >= start_date)
);
```

### Chatbot Table
```sql
CREATE TABLE chatbot (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    avatar_url VARCHAR(255),
    is_active BOOLEAN DEFAULT TRUE,
    welcome_message TEXT,
    embed_code TEXT,
    widget_settings JSONB,
    theme_color VARCHAR(7) DEFAULT '#7c3aed',
    position VARCHAR(20) DEFAULT 'bottom-right',
    auto_open BOOLEAN DEFAULT FALSE,
    business_hours JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT uq_chatbot_property UNIQUE (property_id) 
);
```

### FAQ Table (Enhanced for Human-in-the-Loop)
```sql
CREATE TABLE faq (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    question TEXT NOT NULL,
    answer TEXT, -- DROP NOT NULL for human-in-loop workflow
    category VARCHAR(100),
    source_type VARCHAR(50),
    status VARCHAR(20) DEFAULT 'answered' 
        CHECK (status IN ('pending','answered','needs_review')),
    assigned_to UUID REFERENCES property_manager(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### User Table (Enhanced for Browser Persistence)
```sql
CREATE TABLE "user" (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    first_name VARCHAR(100), -- DROP NOT NULL for anonymous users
    last_name VARCHAR(100),  -- DROP NOT NULL for anonymous users
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20) UNIQUE,
    age INTEGER CHECK (age >= 0),
    lead_source VARCHAR(100),
    hubspot_contact_id VARCHAR(100),
    -- Browser Persistence
    browser_id VARCHAR(255),
    session_count INTEGER DEFAULT 1,
    last_seen TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Conversation Table (Enhanced Status Tracking)
```sql
CREATE TABLE conversation (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    chatbot_id UUID REFERENCES chatbot(id) ON DELETE CASCADE,
    user_id UUID REFERENCES "user"(id) ON DELETE SET NULL,
    start_time TIMESTAMPTZ DEFAULT NOW(),
    end_time TIMESTAMPTZ,
    CONSTRAINT chk_end_after_start  
        CHECK (end_time IS NULL OR end_time > start_time),
    is_qualified BOOLEAN DEFAULT FALSE,
    is_book_tour BOOLEAN DEFAULT FALSE,
    tour_type VARCHAR(50),
    tour_datetime TIMESTAMP,
    ai_intent_summary TEXT,
    apartment_size_preference VARCHAR(50),
    move_in_date DATE,
    price_range_min DECIMAL(10,2) CHECK (price_range_min >= 0),
    price_range_max DECIMAL(10,2) CHECK (price_range_max >= price_range_min),
    occupants_count INTEGER CHECK (occupants_count >= 0),
    has_pets BOOLEAN,
    pet_details JSONB,
    desired_features JSONB,
    work_location VARCHAR(255),
    reason_for_moving TEXT,
    pre_qualified BOOLEAN DEFAULT FALSE,
    source VARCHAR(100),
    status VARCHAR(50) DEFAULT 'chat_initiated'
        CHECK (status IN ('chat_initiated', 'info_collected', 'tour_scheduled', 
                         'tour_completed', 'handed_off', 'active', 'completed', 'abandoned')),
    notification_status JSONB,
    lead_score INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Message Table
```sql
CREATE TABLE message (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    sender_type VARCHAR(20) NOT NULL CHECK (sender_type IN ('user','bot')),
    message_text TEXT NOT NULL,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    message_type VARCHAR(50),
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Lead Notification Table
```sql
CREATE TABLE lead_notification (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    property_manager_id UUID REFERENCES property_manager(id),
    notification_type VARCHAR(50),
    status VARCHAR(50),
    sent_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    response_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Website Integration Table
```sql
CREATE TABLE website_integration (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    website_url VARCHAR(255) NOT NULL,
    chatbot_id UUID REFERENCES chatbot(id),
    integration_type VARCHAR(50),
    configuration JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    tracking_id VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

## New Tables for Enhanced Features

### URL Property Mapping Table
```sql
CREATE TABLE url_property_mapping (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    url VARCHAR(255) NOT NULL,
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(url, property_id)
);
```

### Amplitude Analytics Table
```sql
CREATE TABLE amplitude_analytics (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    user_id UUID REFERENCES "user"(id) ON DELETE SET NULL,

    -- Core Engagement Metrics (18 variables used in frontend)
    chat_session_started BOOLEAN DEFAULT FALSE,
    user_messages_sent INTEGER DEFAULT 0,
    bot_messages_received INTEGER DEFAULT 0,
    answer_button_clicks INTEGER DEFAULT 0,
    contact_captured BOOLEAN DEFAULT FALSE,
    contact_method VARCHAR(20),
    tour_booked BOOLEAN DEFAULT FALSE,
    tour_type VARCHAR(20),
    email_office_clicked INTEGER DEFAULT 0,
    phone_call_clicked INTEGER DEFAULT 0,
    incentive_offered BOOLEAN DEFAULT FALSE,
    incentive_accepted BOOLEAN DEFAULT FALSE,
    incentive_expired BOOLEAN DEFAULT FALSE,
    admin_handoff_triggered BOOLEAN DEFAULT FALSE,
    customer_service_escalated BOOLEAN DEFAULT FALSE,
    conversation_abandoned BOOLEAN DEFAULT FALSE,
    widget_session_ended BOOLEAN DEFAULT FALSE,
    widget_minimized INTEGER DEFAULT 0,

    -- Calculated Metrics
    session_duration INTEGER,
    engagement_score VARCHAR(5),
    qualified BOOLEAN DEFAULT FALSE,
    pre_lease BOOLEAN DEFAULT FALSE,
    tour_intent BOOLEAN DEFAULT FALSE,
    hot BOOLEAN DEFAULT FALSE,
    signed BOOLEAN DEFAULT FALSE,

    -- Session Tracking
    device_id VARCHAR(255),
    session_id VARCHAR(255),
    page_url TEXT,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Email Management Tables
```sql
CREATE TABLE email_drafts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lead_id UUID REFERENCES "user"(id) ON DELETE CASCADE,
    conversation_id UUID REFERENCES conversation(id) ON DELETE CASCADE,
    subject TEXT NOT NULL,
    content TEXT NOT NULL,
    preview TEXT,
    status VARCHAR(20) DEFAULT 'draft',
    priority VARCHAR(10) DEFAULT 'medium',
    tags JSONB,
    scheduled_for TIMESTAMPTZ,
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE email_campaigns (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    template_content TEXT,
    sent_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Task Management Table
```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    conversation_id UUID REFERENCES conversation(id) ON DELETE SET NULL,
    assigned_to UUID REFERENCES property_manager(id) ON DELETE SET NULL,
    question TEXT NOT NULL,
    answer TEXT,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Tours Management Table
```sql
CREATE TABLE tours (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversation(id) ON DELETE SET NULL,
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    property_manager_id UUID REFERENCES property_manager(id) ON DELETE SET NULL,
    prospect_name VARCHAR(255) NOT NULL,
    prospect_email VARCHAR(255),
    prospect_phone VARCHAR(20),
    unit_interest VARCHAR(100),
    tour_datetime TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER DEFAULT 60,
    tour_type VARCHAR(20) DEFAULT 'in_person',
    status VARCHAR(20) DEFAULT 'scheduled',
    source VARCHAR(20) DEFAULT 'chat',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Calendar Integration Table
```sql
CREATE TABLE calendar_integrations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID REFERENCES property(id) ON DELETE CASCADE,
    property_manager_id UUID REFERENCES property_manager(id) ON DELETE CASCADE,
    calendly_uri VARCHAR(255),
    ical_url VARCHAR(255),
    webhook_url VARCHAR(255),
    last_sync TIMESTAMPTZ,
    sync_status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Required Schema Modifications

### Update FAQ Table for Human-in-the-Loop Workflow
```sql
ALTER TABLE faq ALTER COLUMN answer DROP NOT NULL;
ALTER TABLE faq ADD COLUMN status VARCHAR(20) DEFAULT 'answered';
ALTER TABLE faq ADD COLUMN assigned_to UUID REFERENCES property_manager(id);
ALTER TABLE faq ADD CONSTRAINT faq_status_check 
    CHECK (status IN ('pending','answered','needs_review'));
```

### Enhance User Table for Browser Persistence
```sql
ALTER TABLE "user" ADD COLUMN browser_id VARCHAR(255);
ALTER TABLE "user" ADD COLUMN session_count INTEGER DEFAULT 1;
ALTER TABLE "user" ADD COLUMN last_seen TIMESTAMPTZ;
ALTER TABLE "user" ALTER COLUMN first_name DROP NOT NULL;
ALTER TABLE "user" ALTER COLUMN last_name DROP NOT NULL;
```

### Update Conversation Status for Frontend Stages
```sql
ALTER TABLE conversation ADD CONSTRAINT conversation_status_check 
    CHECK (status IN ('chat_initiated', 'info_collected', 'tour_scheduled', 
                     'tour_completed', 'handed_off', 'active', 'completed', 'abandoned'));
```

### Enhance Chatbot Table for Widget Configuration
```sql
ALTER TABLE chatbot ADD COLUMN theme_color VARCHAR(7) DEFAULT '#7c3aed';
ALTER TABLE chatbot ADD COLUMN position VARCHAR(20) DEFAULT 'bottom-right';
ALTER TABLE chatbot ADD COLUMN auto_open BOOLEAN DEFAULT FALSE;
ALTER TABLE chatbot ADD COLUMN business_hours JSONB;
```

### Environment Variables to Database Migration
```sql
ALTER TABLE company ADD COLUMN calendly_org_uri VARCHAR(255);
ALTER TABLE company ADD COLUMN sendgrid_api_key VARCHAR(255);
ALTER TABLE company ADD COLUMN amplitude_api_key VARCHAR(255);
ALTER TABLE company ADD COLUMN openai_api_key VARCHAR(255);
```

## Performance Indexes

### High-Traffic Lookup Indexes
```sql
-- Conversation analytics queries
CREATE INDEX idx_conversation_chatbot_started
  ON conversation (chatbot_id, start_time DESC);

-- Message clustering for conversation joins
CREATE INDEX idx_message_conversation_timestamp
  ON message (conversation_id, timestamp DESC);

-- User session tracking
CREATE INDEX idx_user_browser_id
  ON "user" (browser_id);

-- Property manager assignments
CREATE INDEX idx_property_manager_assignment_property
  ON property_manager_assignment (property_id, is_primary);

-- FAQ status tracking
CREATE INDEX idx_faq_property_status
  ON faq (property_id, status);

-- Amplitude analytics
CREATE INDEX idx_amplitude_conversation_created
  ON amplitude_analytics (conversation_id, created_at DESC);

-- Tours scheduling
CREATE INDEX idx_tours_property_datetime
  ON tours (property_id, tour_datetime);

-- Email drafts
CREATE INDEX idx_email_drafts_conversation_status
  ON email_drafts (conversation_id, status);
```

## Database Relationships Summary

### Foreign Key Relationships
- **company** → **property** (1:many)
- **company** → **property_manager** (1:many)
- **property** → **conversation** (1:many)
- **property** → **faq** (1:many)
- **property** → **website_integration** (1:many)
- **property** → **tours** (1:many)
- **property** → **tasks** (1:many)
- **property** → **email_campaigns** (1:many)
- **property_manager** → **property_manager_assignment** (1:many)
- **property_manager** → **lead_notification** (1:many)
- **property_manager** → **tasks** (1:many)
- **property_manager** → **tours** (1:many)
- **user** → **conversation** (1:many)
- **conversation** → **message** (1:many)
- **conversation** → **lead_notification** (1:many)
- **conversation** → **amplitude_analytics** (1:1)
- **conversation** → **tours** (1:many)
- **conversation** → **email_drafts** (1:many)

### Cascade Delete Rules
- **ON DELETE CASCADE**: property, property_manager, conversation, message, lead_notification, faq, website_integration, tours, tasks, email_campaigns
- **ON DELETE SET NULL**: conversation.user_id, conversation.chatbot_id, tasks.conversation_id, tours.conversation_id, email_drafts.conversation_id

### Unique Constraints
- **company.name**: Unique company names
- **property_manager.email**: Unique email addresses
- **property_manager.phone**: Unique phone numbers
- **user.email**: Unique email addresses (nullable)
- **user.phone**: Unique phone numbers (nullable)
- **chatbot.property_id**: One chatbot per property
- **url_property_mapping.url**: Unique URL mappings

## Data Flow Patterns

### RAG Integration Points
- **FAQ Table**: Stores property-specific knowledge base
- **URL Property Mapping**: Maps URLs to property IDs for RAG context
- **Property ID**: Used as chatbot ID for Milvus integration

### Analytics Integration Points
- **Amplitude Analytics**: Tracks 18 core engagement metrics
- **Conversation Status**: Tracks lead progression stages
- **User Browser Persistence**: Tracks anonymous user sessions

### Integration API Points
- **Company API Keys**: Store integration credentials
- **Property Manager Assignments**: Route leads to appropriate managers
- **Lead Notifications**: Real-time manager alerts
- **Tour Management**: Calendar integration support
- **Email Management**: Campaign and draft tracking 