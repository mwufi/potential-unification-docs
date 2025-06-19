# AI CRM MVP - System Architecture Overview

## Executive Summary

This document outlines the system architecture for an AI-powered CRM that leverages email data to provide intelligent contact insights and relationship management. The MVP focuses on Gmail integration, contact extraction, enrichment, and AI-powered querying.

## Core Principles

1. **Leverage Existing Code**: Reuse authentication, UI components, and Google integration from Analog/Nimbus
2. **Event-Driven Architecture**: Everything is an event for future AI training
3. **Background Processing**: Heavy lifting happens asynchronously
4. **Simple Data Model**: Start with contacts and interactions, expand later
5. **AI-First Design**: Structure data for LLM consumption from day one

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Frontend (Next.js)                   │
├─────────────────────────────────────────────────────────────┤
│                          API Layer (tRPC)                    │
├─────────────────────────────────────────────────────────────┤
│  Auth Service │ Contact Service │ Email Service │ AI Service │
├───────────────┴─────────────────┴───────────────┴───────────┤
│                    Event Bus (PostgreSQL LISTEN/NOTIFY)      │
├─────────────────────────────────────────────────────────────┤
│  Background Jobs  │  Email Sync  │  Enrichment  │  Analytics │
├───────────────────┴──────────────┴──────────────┴───────────┤
│                    Data Layer (PostgreSQL + Redis)           │
└─────────────────────────────────────────────────────────────┘
```

## Component Architecture

### 1. Frontend Layer
- **Framework**: Next.js 15 (from existing codebases)
- **UI Library**: shadcn/ui (already in use)
- **State Management**: TanStack Query + Jotai
- **Real-time Updates**: SSE for live data

### 2. API Layer
- **Framework**: tRPC (from Analog)
- **Authentication**: Better Auth (existing)
- **Rate Limiting**: Built-in (from Nimbus)
- **Validation**: Zod schemas

### 3. Core Services

#### Auth Service
```typescript
interface AuthService {
  // Reuse from existing
  signIn(): Promise<User>
  linkGoogleAccount(): Promise<Account>
  
  // New for CRM
  getGmailPermissions(): Promise<Permissions>
  refreshTokens(): Promise<void>
}
```

#### Contact Service
```typescript
interface ContactService {
  // Core CRUD
  listContacts(filters?: ContactFilters): Promise<Contact[]>
  getContact(id: string): Promise<Contact>
  updateContact(id: string, data: Partial<Contact>): Promise<Contact>
  
  // Analytics
  getTopContacts(metric: 'emails' | 'recent'): Promise<Contact[]>
  getContactStats(id: string): Promise<ContactStats>
  
  // Enrichment
  enrichContact(id: string): Promise<EnrichmentResult>
}
```

#### Email Service
```typescript
interface EmailService {
  // Sync operations
  syncEmails(since?: Date): Promise<SyncResult>
  getEmailThread(threadId: string): Promise<EmailThread>
  
  // Analysis
  extractContacts(emails: Email[]): Promise<ExtractedContact[]>
  analyzeInteractions(contactId: string): Promise<Interaction[]>
}
```

#### AI Service
```typescript
interface AIService {
  // Natural language queries
  query(prompt: string, context?: QueryContext): Promise<AIResponse>
  
  // Specific AI features
  summarizeRelationship(contactId: string): Promise<string>
  suggestNextActions(contactId: string): Promise<Action[]>
  generateEmailDraft(context: EmailContext): Promise<string>
}
```

### 4. Background Processing

#### Job Queue System
```typescript
interface JobQueue {
  // Job management
  enqueue<T>(job: Job<T>): Promise<string>
  process<T>(jobType: string, handler: JobHandler<T>): void
  
  // Monitoring
  getJobStatus(jobId: string): Promise<JobStatus>
  retryFailed(): Promise<number>
}

// Job types
type JobType = 
  | 'email-sync'
  | 'contact-extraction'
  | 'contact-enrichment'
  | 'social-media-sync'
  | 'analytics-calculation'
```

#### Email Sync Worker
```typescript
class EmailSyncWorker {
  async sync(userId: string, since?: Date) {
    // 1. Fetch emails from Gmail API
    const emails = await gmailAPI.getEmails({ since })
    
    // 2. Extract contacts
    const contacts = await this.extractContacts(emails)
    
    // 3. Store in database
    await this.storeEmails(emails)
    await this.upsertContacts(contacts)
    
    // 4. Emit events
    await eventBus.emit('emails.synced', { userId, count: emails.length })
    
    // 5. Trigger enrichment jobs
    for (const contact of contacts) {
      await jobQueue.enqueue({
        type: 'contact-enrichment',
        data: { contactId: contact.id }
      })
    }
  }
}
```

### 5. Event System

#### Event Store
```typescript
interface Event {
  id: string
  type: EventType
  userId: string
  timestamp: Date
  data: JsonValue
  metadata?: {
    source: string
    version: string
  }
}

type EventType = 
  | 'user.signed_in'
  | 'account.linked'
  | 'email.received'
  | 'email.sent'
  | 'contact.created'
  | 'contact.enriched'
  | 'contact.viewed'
  | 'ai.query_executed'
```

#### Event Bus Implementation
```typescript
class EventBus {
  // Using PostgreSQL LISTEN/NOTIFY for simplicity
  async emit(type: EventType, data: any) {
    await db.insert(events).values({
      type,
      userId: getCurrentUserId(),
      timestamp: new Date(),
      data
    })
    
    await db.execute(sql`NOTIFY ${type}, ${JSON.stringify(data)}`)
  }
  
  subscribe(type: EventType, handler: EventHandler) {
    // PostgreSQL LISTEN implementation
  }
}
```

## Data Model

### Core Tables

```sql
-- Reuse from existing
users (id, email, name, created_at)
accounts (id, user_id, provider, access_token, refresh_token)

-- New for CRM
contacts (
  id uuid PRIMARY KEY,
  user_id uuid REFERENCES users(id),
  email text UNIQUE,
  name text,
  company text,
  title text,
  phone text,
  linkedin_url text,
  twitter_handle text,
  notes text,
  tags text[],
  custom_fields jsonb,
  first_seen_at timestamp,
  last_interaction_at timestamp,
  created_at timestamp,
  updated_at timestamp
)

emails (
  id text PRIMARY KEY, -- Gmail message ID
  user_id uuid REFERENCES users(id),
  thread_id text,
  from_email text,
  to_emails text[],
  cc_emails text[],
  subject text,
  body_text text,
  body_html text,
  sent_at timestamp,
  labels text[],
  raw_headers jsonb,
  created_at timestamp
)

interactions (
  id uuid PRIMARY KEY,
  user_id uuid REFERENCES users(id),
  contact_id uuid REFERENCES contacts(id),
  type text, -- 'email_sent', 'email_received', 'meeting', etc
  email_id text REFERENCES emails(id),
  direction text, -- 'inbound', 'outbound'
  metadata jsonb,
  occurred_at timestamp
)

contact_stats (
  contact_id uuid PRIMARY KEY REFERENCES contacts(id),
  total_emails_sent integer DEFAULT 0,
  total_emails_received integer DEFAULT 0,
  avg_response_time_hours float,
  last_email_sent_at timestamp,
  last_email_received_at timestamp,
  email_frequency_score float, -- 0-100
  relationship_strength float, -- 0-100
  updated_at timestamp
)

enrichments (
  id uuid PRIMARY KEY,
  contact_id uuid REFERENCES contacts(id),
  source text, -- 'twitter', 'linkedin', 'clearbit', etc
  data jsonb,
  fetched_at timestamp
)

events (
  id uuid PRIMARY KEY,
  user_id uuid REFERENCES users(id),
  type text,
  data jsonb,
  metadata jsonb,
  created_at timestamp
)

background_jobs (
  id uuid PRIMARY KEY,
  type text,
  status text, -- 'pending', 'processing', 'completed', 'failed'
  data jsonb,
  result jsonb,
  error text,
  attempts integer DEFAULT 0,
  created_at timestamp,
  started_at timestamp,
  completed_at timestamp
)
```

## Reusable Components from Existing Codebases

### From Both:
1. **Authentication**: Better Auth setup with Google OAuth
2. **UI Components**: All shadcn/ui components
3. **Database Setup**: PostgreSQL with Drizzle ORM
4. **TypeScript Config**: Type safety setup
5. **Deployment Config**: Vercel/deployment setup

### From Analog:
1. **tRPC Setup**: API layer architecture
2. **Google Calendar Integration**: Reuse for Google Contacts API
3. **Provider Pattern**: Adapt for email providers
4. **Date Handling**: Temporal API usage

### From Nimbus:
1. **Rate Limiting**: For API calls
2. **Background Job Structure**: Adapt for email sync
3. **File Organization**: For email attachments
4. **Hono Server**: Could use for webhook endpoints

## New Components Needed

### 1. Gmail Integration
```typescript
class GmailProvider {
  constructor(private accessToken: string) {}
  
  async getMessages(query?: string): Promise<GmailMessage[]>
  async getThread(threadId: string): Promise<GmailThread>
  async watchMailbox(topicName: string): Promise<WatchResponse>
  async getAttachment(messageId: string, attachmentId: string): Promise<Attachment>
}
```

### 2. Contact Extraction Engine
```typescript
class ContactExtractor {
  extractFromEmail(email: Email): ExtractedContact[] {
    // Parse email headers
    // Extract names from signatures
    // Identify companies from domains
    // Parse phone numbers and social links
  }
  
  mergeContacts(existing: Contact, extracted: ExtractedContact): Contact {
    // Intelligent merging logic
    // Preserve user edits
    // Update statistics
  }
}
```

### 3. Enrichment Pipeline
```typescript
interface EnrichmentProvider {
  name: string
  enrich(email: string): Promise<EnrichmentData>
}

class EnrichmentPipeline {
  providers: EnrichmentProvider[] = [
    new TwitterEnricher(),
    new LinkedInEnricher(),
    new ClearbitEnricher()
  ]
  
  async enrichContact(contact: Contact): Promise<EnrichedContact> {
    const results = await Promise.allSettled(
      this.providers.map(p => p.enrich(contact.email))
    )
    return this.mergeEnrichments(contact, results)
  }
}
```

### 4. AI Query Engine
```typescript
class AIQueryEngine {
  async processQuery(query: string, userId: string): Promise<QueryResult> {
    // 1. Understand intent
    const intent = await this.classifyIntent(query)
    
    // 2. Extract parameters
    const params = await this.extractParameters(query, intent)
    
    // 3. Fetch relevant data
    const context = await this.gatherContext(userId, intent, params)
    
    // 4. Generate response
    const response = await this.generateResponse(query, context)
    
    // 5. Format for UI
    return this.formatResponse(response, intent)
  }
}
```

## Keeping It Simple

### MVP Scope Limits:
1. **Gmail Only**: No other email providers initially
2. **Basic Enrichment**: Just Twitter/X posts to start
3. **Simple AI**: Pre-defined queries first, then natural language
4. **No Team Features**: Single user only
5. **Basic Analytics**: Email frequency and response times only

### Progressive Enhancement Plan:
1. **Phase 1**: Auth + Email Sync + Contact List
2. **Phase 2**: Analytics + Basic AI Queries
3. **Phase 3**: Enrichment + Advanced AI
4. **Phase 4**: Team features + Multiple providers

### Technical Simplifications:
1. **Use PostgreSQL for Everything**: Events, queues, and data
2. **Server-Side Rendering**: Minimize client complexity
3. **Polling > WebSockets**: For MVP real-time updates
4. **Monolithic First**: Single Next.js app with API routes
5. **Direct AI Calls**: No complex RAG initially

## Success Metrics

1. **Performance**: Sync 1000 emails in < 30 seconds
2. **Accuracy**: 95% contact extraction accuracy
3. **AI Quality**: Useful responses to 80% of queries
4. **Reliability**: 99% job completion rate
5. **User Experience**: < 3 clicks to any feature

## Next Steps

1. Set up project structure reusing existing components
2. Implement Gmail OAuth and basic sync
3. Build contact extraction algorithm
4. Create simple UI for contact list
5. Add basic AI query interface
6. Implement background job system
7. Add enrichment capabilities
8. Polish and optimize