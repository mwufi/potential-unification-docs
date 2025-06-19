# Data Model & Schema Design

## Core Design Principles

1. **AI-Optimized**: JSONB fields for flexible LLM consumption
2. **Event Sourced**: Complete audit trail for AI training
3. **Performance First**: Strategic indexes and partitioning
4. **Privacy Aware**: Encryption fields marked explicitly
5. **Extensible**: Custom fields without schema changes

## Database Schema

### User & Authentication

```sql
-- Users table (from Better Auth, extended)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    avatar_url TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    settings JSONB DEFAULT '{}',
    features JSONB DEFAULT '{}', -- feature flags
    onboarding_completed BOOLEAN DEFAULT FALSE,
    timezone VARCHAR(50) DEFAULT 'UTC'
);

-- User accounts (OAuth connections)
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL, -- 'google', 'microsoft'
    provider_account_id VARCHAR(255) NOT NULL,
    access_token TEXT ENCRYPTED, -- marked for encryption
    refresh_token TEXT ENCRYPTED,
    expires_at TIMESTAMPTZ,
    token_type VARCHAR(50),
    scope TEXT,
    id_token TEXT ENCRYPTED,
    session_state TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    sync_state JSONB DEFAULT '{}', -- track sync progress
    UNIQUE(provider, provider_account_id)
);

CREATE INDEX idx_accounts_user_id ON accounts(user_id);
CREATE INDEX idx_accounts_provider ON accounts(provider);
```

### Contacts & Organizations

```sql
-- Contacts table
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    email VARCHAR(255),
    name VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(50),
    avatar_url TEXT,
    organization_id UUID,
    title VARCHAR(255),
    bio TEXT,
    location JSONB, -- {city, state, country, timezone}
    social_profiles JSONB DEFAULT '{}', -- {twitter, linkedin, etc}
    custom_fields JSONB DEFAULT '{}',
    tags TEXT[] DEFAULT '{}',
    source VARCHAR(50), -- 'email', 'manual', 'import'
    source_details JSONB, -- extraction confidence, etc
    relationship_strength DECIMAL(3,2), -- 0.00 to 1.00
    last_interaction_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    is_deleted BOOLEAN DEFAULT FALSE,
    merged_into_id UUID, -- for deduplication
    search_vector tsvector, -- full text search
    embedding vector(1536) -- for semantic search
);

CREATE INDEX idx_contacts_user_id ON contacts(user_id);
CREATE INDEX idx_contacts_email ON contacts(email);
CREATE INDEX idx_contacts_organization ON contacts(organization_id);
CREATE INDEX idx_contacts_updated_at ON contacts(updated_at DESC);
CREATE INDEX idx_contacts_search ON contacts USING GIN(search_vector);
CREATE INDEX idx_contacts_embedding ON contacts USING ivfflat(embedding vector_cosine_ops);
CREATE INDEX idx_contacts_tags ON contacts USING GIN(tags);

-- Organizations table
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(255),
    website TEXT,
    description TEXT,
    industry VARCHAR(100),
    size VARCHAR(50), -- '1-10', '11-50', etc
    location JSONB,
    logo_url TEXT,
    social_profiles JSONB DEFAULT '{}',
    custom_fields JSONB DEFAULT '{}',
    enrichment_data JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_organizations_domain ON organizations(domain);
CREATE INDEX idx_organizations_user_id ON organizations(user_id);
```

### Email & Communication

```sql
-- Emails table (partitioned by user_id and date)
CREATE TABLE emails (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    account_id UUID REFERENCES accounts(id) ON DELETE CASCADE,
    message_id VARCHAR(255) NOT NULL, -- Gmail message ID
    thread_id VARCHAR(255), -- Gmail thread ID
    from_email VARCHAR(255),
    from_name VARCHAR(255),
    to_emails TEXT[], -- array of emails
    cc_emails TEXT[],
    bcc_emails TEXT[],
    subject TEXT,
    body_text TEXT,
    body_html TEXT,
    snippet TEXT, -- first 200 chars
    labels TEXT[],
    is_unread BOOLEAN DEFAULT TRUE,
    is_important BOOLEAN DEFAULT FALSE,
    is_starred BOOLEAN DEFAULT FALSE,
    is_draft BOOLEAN DEFAULT FALSE,
    is_sent BOOLEAN DEFAULT FALSE,
    has_attachments BOOLEAN DEFAULT FALSE,
    attachment_count INTEGER DEFAULT 0,
    internal_date TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    extracted_data JSONB DEFAULT '{}', -- phone numbers, addresses, etc
    ai_summary TEXT,
    ai_category VARCHAR(50),
    ai_sentiment VARCHAR(20),
    search_vector tsvector,
    embedding vector(1536),
    UNIQUE(user_id, message_id)
) PARTITION BY RANGE (internal_date);

-- Create monthly partitions
CREATE TABLE emails_2024_01 PARTITION OF emails FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
-- ... create partitions for each month

CREATE INDEX idx_emails_user_thread ON emails(user_id, thread_id);
CREATE INDEX idx_emails_internal_date ON emails(internal_date DESC);
CREATE INDEX idx_emails_from ON emails(from_email);
CREATE INDEX idx_emails_search ON emails USING GIN(search_vector);
CREATE INDEX idx_emails_labels ON emails USING GIN(labels);

-- Email interactions (link between emails and contacts)
CREATE TABLE email_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_id UUID REFERENCES emails(id) ON DELETE CASCADE,
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    interaction_type VARCHAR(20), -- 'from', 'to', 'cc', 'bcc'
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_interactions_email ON email_interactions(email_id);
CREATE INDEX idx_interactions_contact ON email_interactions(contact_id);
CREATE INDEX idx_interactions_type ON email_interactions(interaction_type);
```

### AI & Automation

```sql
-- AI conversations
CREATE TABLE ai_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    agent_id UUID,
    title VARCHAR(255),
    context JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE
);

-- AI messages
CREATE TABLE ai_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID REFERENCES ai_conversations(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL, -- 'user', 'assistant', 'system'
    content TEXT NOT NULL,
    function_calls JSONB, -- for tool use
    metadata JSONB DEFAULT '{}',
    tokens_used INTEGER,
    model VARCHAR(50),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ai_messages_conversation ON ai_messages(conversation_id, created_at);

-- AI agents
CREATE TABLE ai_agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    type VARCHAR(50), -- 'assistant', 'enrichment', 'email'
    config JSONB NOT NULL, -- prompts, parameters, etc
    skills TEXT[], -- available skills
    memory JSONB DEFAULT '{}', -- persistent memory
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Agent executions
CREATE TABLE agent_executions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID REFERENCES ai_agents(id) ON DELETE CASCADE,
    trigger_type VARCHAR(50), -- 'manual', 'scheduled', 'event'
    trigger_data JSONB,
    status VARCHAR(20), -- 'pending', 'running', 'completed', 'failed'
    input JSONB,
    output JSONB,
    error TEXT,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_executions_agent ON agent_executions(agent_id);
CREATE INDEX idx_executions_status ON agent_executions(status);
```

### Background Jobs & Events

```sql
-- Background jobs (pg-boss compatible)
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    data JSONB,
    priority INTEGER DEFAULT 0,
    retry_limit INTEGER DEFAULT 3,
    retry_count INTEGER DEFAULT 0,
    retry_delay INTEGER DEFAULT 0,
    retry_backoff BOOLEAN DEFAULT TRUE,
    start_after TIMESTAMPTZ DEFAULT NOW(),
    started_on TIMESTAMPTZ,
    singleton_key VARCHAR(100),
    expire_in INTERVAL DEFAULT '15 minutes',
    created_on TIMESTAMPTZ DEFAULT NOW(),
    completed_on TIMESTAMPTZ,
    keep_until TIMESTAMPTZ DEFAULT NOW() + INTERVAL '14 days',
    on_complete BOOLEAN DEFAULT FALSE,
    output JSONB,
    state VARCHAR(20) DEFAULT 'created'
);

CREATE INDEX idx_jobs_name_state ON jobs(name, state);
CREATE INDEX idx_jobs_singleton ON jobs(singleton_key);
CREATE INDEX idx_jobs_start_after ON jobs(start_after);

-- Event store
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    entity_type VARCHAR(50) NOT NULL, -- 'contact', 'email', 'agent'
    entity_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL, -- 'contact.created', 'email.received'
    payload JSONB NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions for events
CREATE TABLE events_2024_01 PARTITION OF events FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE INDEX idx_events_entity ON events(entity_type, entity_id);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_user ON events(user_id);
```

### Analytics & Metrics

```sql
-- Contact statistics
CREATE TABLE contact_stats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    total_emails_sent INTEGER DEFAULT 0,
    total_emails_received INTEGER DEFAULT 0,
    last_email_sent_at TIMESTAMPTZ,
    last_email_received_at TIMESTAMPTZ,
    avg_response_time INTERVAL,
    interaction_score DECIMAL(5,2),
    topics TEXT[], -- extracted topics
    sentiment_score DECIMAL(3,2), -- -1 to 1
    calculated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_contact_stats_unique ON contact_stats(contact_id);

-- User analytics
CREATE TABLE user_analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    emails_received INTEGER DEFAULT 0,
    emails_sent INTEGER DEFAULT 0,
    contacts_added INTEGER DEFAULT 0,
    ai_queries INTEGER DEFAULT 0,
    ai_tokens_used INTEGER DEFAULT 0,
    active_time_seconds INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_user_analytics_unique ON user_analytics(user_id, date);
```

### Enrichment & Cache

```sql
-- Enrichment results
CREATE TABLE enrichments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    source VARCHAR(50) NOT NULL, -- 'twitter', 'linkedin', 'clearbit'
    data JSONB NOT NULL,
    confidence_score DECIMAL(3,2),
    fetched_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    is_stale BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_enrichments_contact ON enrichments(contact_id);
CREATE INDEX idx_enrichments_source ON enrichments(source);

-- Cache table for expensive operations
CREATE TABLE cache (
    key VARCHAR(255) PRIMARY KEY,
    value JSONB NOT NULL,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_cache_expires ON cache(expires_at);
```

## Database Functions & Triggers

```sql
-- Update search vectors
CREATE OR REPLACE FUNCTION update_contact_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := 
        setweight(to_tsvector('english', COALESCE(NEW.name, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.email, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.organization_id::text, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(NEW.bio, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_contact_search_vector_trigger
BEFORE INSERT OR UPDATE ON contacts
FOR EACH ROW EXECUTE FUNCTION update_contact_search_vector();

-- Auto-update timestamps
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to all tables with updated_at
CREATE TRIGGER update_users_timestamp BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
-- ... repeat for other tables

-- Event logging function
CREATE OR REPLACE FUNCTION log_event(
    p_user_id UUID,
    p_entity_type VARCHAR,
    p_entity_id UUID,
    p_event_type VARCHAR,
    p_payload JSONB,
    p_metadata JSONB DEFAULT '{}'
) RETURNS UUID AS $$
DECLARE
    v_event_id UUID;
BEGIN
    INSERT INTO events (user_id, entity_type, entity_id, event_type, payload, metadata)
    VALUES (p_user_id, p_entity_type, p_entity_id, p_event_type, p_payload, p_metadata)
    RETURNING id INTO v_event_id;
    
    -- Notify listeners
    PERFORM pg_notify('events', json_build_object(
        'id', v_event_id,
        'type', p_event_type,
        'entity_id', p_entity_id
    )::text);
    
    RETURN v_event_id;
END;
$$ LANGUAGE plpgsql;
```

## Indexes Strategy

### Performance Indexes
- Primary keys: B-tree indexes by default
- Foreign keys: B-tree for lookups
- Timestamp fields: B-tree DESC for recent data
- Arrays: GIN for contains queries
- Full text: GIN with tsvector
- Vector similarity: IVFFlat for embeddings

### Covering Indexes
```sql
-- For common query patterns
CREATE INDEX idx_emails_user_date_covering 
ON emails(user_id, internal_date DESC) 
INCLUDE (subject, from_email, snippet);

CREATE INDEX idx_contacts_user_strength_covering
ON contacts(user_id, relationship_strength DESC)
INCLUDE (name, email, last_interaction_at);
```

## Partitioning Strategy

### Time-based Partitions
- `emails`: Monthly partitions by internal_date
- `events`: Monthly partitions by created_at
- Automatic partition creation via pg_partman

### Benefits
- Faster queries on recent data
- Easier data archival
- Parallel query execution
- Reduced index size

## Migration Strategy

1. **Schema versioning** with timestamps
2. **Forward-only migrations** (no down migrations)
3. **Zero-downtime deployments** with careful planning
4. **Data migrations** separate from schema migrations