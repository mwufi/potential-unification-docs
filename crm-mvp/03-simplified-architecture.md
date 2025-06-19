# Simplified Architecture (No Event Sourcing)

## Overview

Removing event sourcing complexity while keeping simple event tracking for future AI features like "you never read emails from X, should I auto-archive?"

## Simplified Event Tracking

### 1. Basic Events Table

```sql
-- Super simple events table
CREATE TABLE user_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  data JSONB NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Index for quick lookups
CREATE INDEX idx_user_events_user_type ON user_events(user_id, type, created_at DESC);
```

### 2. What We Track (Only What's Useful)

```typescript
// Just track high-level actions for AI insights
type SimpleEvent = 
  | { type: 'email_received', data: { from: string, subject: string } }
  | { type: 'email_opened', data: { emailId: string, from: string } }
  | { type: 'email_archived', data: { from: string } }
  | { type: 'contact_viewed', data: { contactId: string } }
  | { type: 'ai_query', data: { query: string, helpful: boolean } }

// Dead simple tracking
async function trackEvent(userId: string, event: SimpleEvent) {
  await db.insert(userEvents).values({
    userId,
    type: event.type,
    data: event.data
  })
}
```

### 3. Using Events for AI Insights

```typescript
// Example: Detect unread newsletter patterns
async function findUnreadNewsletters(userId: string) {
  const result = await db.execute(sql`
    WITH email_stats AS (
      SELECT 
        data->>'from' as sender,
        COUNT(*) FILTER (WHERE type = 'email_received') as received_count,
        COUNT(*) FILTER (WHERE type = 'email_opened') as opened_count
      FROM user_events
      WHERE user_id = ${userId}
        AND created_at > NOW() - INTERVAL '30 days'
        AND type IN ('email_received', 'email_opened')
      GROUP BY data->>'from'
    )
    SELECT sender, received_count, opened_count
    FROM email_stats
    WHERE opened_count = 0 
      AND received_count > 3
      AND sender LIKE '%newsletter%' OR sender LIKE '%.substack.com'
  `)
  
  return result.rows
}

// Use in AI suggestions
async function getAISuggestions(userId: string) {
  const unreadNewsletters = await findUnreadNewsletters(userId)
  
  const suggestions = []
  for (const newsletter of unreadNewsletters) {
    suggestions.push({
      type: 'auto_archive',
      message: `You haven't read any of the ${newsletter.received_count} emails from ${newsletter.sender}. Archive future emails?`,
      action: () => createEmailFilter(userId, newsletter.sender, 'archive')
    })
  }
  
  return suggestions
}
```

## Simplified Background Jobs

### 1. Simple Job Queue (No Complex State Machine)

```typescript
// Just use pg-boss or bull instead of custom implementation
import PgBoss from 'pg-boss'

const boss = new PgBoss(DATABASE_URL)

// Define job handlers
boss.work('sync-emails', async (job) => {
  const { userId } = job.data
  await syncUserEmails(userId)
})

boss.work('enrich-contact', async (job) => {
  const { contactId } = job.data
  await enrichContact(contactId)
})

// Queue jobs
await boss.send('sync-emails', { userId })
await boss.send('enrich-contact', { contactId }, { 
  retryLimit: 3,
  retryDelay: 60 
})
```

### 2. Cron Jobs for Periodic Tasks

```typescript
// Simple cron using node-cron or Vercel cron
import { cron } from '@vercel/cron'

// Sync all users' emails every hour
export const syncEmails = cron('0 * * * *', async () => {
  const users = await getActiveUsers()
  for (const user of users) {
    await boss.send('sync-emails', { userId: user.id })
  }
})

// Calculate analytics daily
export const calculateStats = cron('0 2 * * *', async () => {
  await calculateDailyStats()
})
```

## Simplified Architecture Benefits

### What We Keep:
1. **Core CRM Features**: Contact management, email sync
2. **AI Queries**: Natural language search
3. **Basic Analytics**: For AI suggestions
4. **Background Processing**: For heavy tasks

### What We Remove:
1. ❌ Complex event sourcing
2. ❌ Event replay mechanisms  
3. ❌ Event projections
4. ❌ CQRS patterns
5. ❌ Custom job queue implementation

### New Tech Stack:
```javascript
// Before (Complex)
- Custom EventBus with PostgreSQL LISTEN/NOTIFY
- Custom job queue with state machines
- Event sourcing with projections
- Complex retry mechanisms

// After (Simple)
- Basic events table for analytics
- pg-boss or BullMQ for jobs
- Direct database queries
- Simple retry with exponential backoff
```

## Updated Data Flow

### Email Sync Flow (Simplified)
```typescript
async function syncUserEmails(userId: string) {
  // 1. Get new emails
  const emails = await gmail.getNewEmails(lastSyncId)
  
  // 2. Process each email
  for (const email of emails) {
    // Store email
    await db.insert(emails).values(email)
    
    // Extract contacts (simple)
    const contacts = extractContactsFromEmail(email)
    await upsertContacts(contacts)
    
    // Track event (simple)
    await trackEvent(userId, {
      type: 'email_received',
      data: { from: email.from, subject: email.subject }
    })
  }
  
  // 3. Update sync state
  await updateLastSyncId(userId, emails[emails.length - 1].id)
}
```

### AI Query Flow (Simplified)
```typescript
async function handleAIQuery(query: string, userId: string) {
  // 1. Understand query (no complex intent parsing)
  const { sql, params } = await llm.generateSQL(query)
  
  // 2. Execute query
  const results = await db.execute(sql, params)
  
  // 3. Format response
  const response = await llm.formatResponse(query, results)
  
  // 4. Track for learning (simple)
  await trackEvent(userId, {
    type: 'ai_query',
    data: { query, helpful: true }
  })
  
  return response
}
```

## Implementation Shortcuts

### 1. Use Existing Services
- **Jobs**: pg-boss or BullMQ instead of custom
- **Search**: PostgreSQL full-text instead of vector DB initially
- **Caching**: Simple in-memory cache instead of Redis
- **Real-time**: Polling instead of websockets

### 2. Simpler Patterns
```typescript
// Instead of complex event handlers
eventBus.on('contact.created', ComplexHandler)

// Just do this
async function createContact(data) {
  const contact = await db.insert(contacts).values(data)
  await trackEvent(userId, { type: 'contact_created', data: { id: contact.id } })
  await queue.send('enrich-contact', { contactId: contact.id })
  return contact
}
```

### 3. Progressive Enhancement
Start with:
- Basic email sync
- Simple contact extraction  
- Pre-defined AI queries

Add later:
- Smart suggestions from events
- Auto-archive/filters
- Complex AI insights

## New Timeline (Even Faster!)

### Week 1-2: Core
- Gmail sync
- Contact extraction
- Basic UI

### Week 3-4: AI
- Natural language queries
- Contact insights
- Email search

### Week 5: Polish
- Performance
- Error handling
- Beta launch

**Total: 5 weeks instead of 8!**

The key insight: Events are just analytics data, not the source of truth. This makes everything MUCH simpler while still enabling cool AI features later.