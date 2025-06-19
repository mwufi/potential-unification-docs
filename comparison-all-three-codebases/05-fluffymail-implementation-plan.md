# FluffyMail AI CRM Implementation Plan

## Executive Summary

Starting FluffyMail as a greenfield project while leveraging 70% of existing code from Zero, Analog, and Nimbus. This plan gets us to MVP in 4 weeks with a team of 2-4 developers.

## Week 0: Setup & Architecture (2-3 days)

### Day 1: Project Bootstrap
```bash
# Initialize FluffyMail
bun create next-app fluffymail --typescript --tailwind --app
cd fluffymail

# Core dependencies (match AI CRM vision)
bun add @trpc/server @trpc/client @trpc/react-query @trpc/next
bun add drizzle-orm postgres @vercel/postgres
bun add @ai-sdk/openai @ai-sdk/anthropic ai
bun add better-auth @better-auth/client
bun add @tanstack/react-query zustand
bun add pg-boss ioredis
bun add zod react-hook-form
bun add -D drizzle-kit @types/pg tsx
```

### Day 2-3: Core Infrastructure
1. **Copy from Zero:**
   - `/lib/auth-client.ts` and `/lib/auth-proxy.ts`
   - `/components/ui/*` (entire UI library)
   - `/providers/query-provider.tsx`
   - Database schema (auth tables)
   - tRPC setup

2. **Adapt for Next.js:**
   - Convert Hono routes → Next.js API routes
   - Cloudflare Queues → pg-boss
   - Edge runtime → Node.js

3. **Setup essentials:**
   - Environment variables
   - Database connections
   - Authentication flow
   - Basic layout

## Week 1: Foundation & Gmail Sync

### Goals
- ✅ Working authentication with Google OAuth
- ✅ Initial Gmail sync (emails + contacts)
- ✅ Basic UI showing synced data
- ✅ Background job system

### Implementation Steps

#### Days 1-2: Authentication & Database
```typescript
// 1. Set up database schema (extend Zero's)
// schema/index.ts
export const users = pgTable('users', { /* from Zero */ });
export const accounts = pgTable('accounts', { /* from Zero */ });
export const sessions = pgTable('sessions', { /* from Zero */ });

// Add CRM tables
export const contacts = pgTable('contacts', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id),
  email: varchar('email', { length: 255 }).notNull(),
  name: varchar('name', { length: 255 }),
  company: varchar('company', { length: 255 }),
  enrichedData: jsonb('enriched_data'),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});

// 2. Copy & adapt auth from Zero
// app/api/auth/[...all]/route.ts
```

#### Days 3-4: Gmail Integration
```typescript
// 1. Copy email sync patterns from Zero
// 2. Adapt for full Gmail API access
// lib/gmail/sync.ts
export async function syncGmailAccount(accountId: string) {
  // Initial sync logic
  // Use pg-boss for background processing
}

// 3. Extract contacts during sync
// lib/extractors/contact-extractor.ts
export function extractContactsFromEmail(email: GmailMessage) {
  // Smart extraction logic
  // Return normalized contact data
}
```

#### Day 5: Basic UI
```typescript
// 1. Copy mail list components from Zero
// 2. Create contact list view
// app/(dashboard)/contacts/page.tsx
export default function ContactsPage() {
  // Display synced contacts
  // Use Zero's table components
}
```

### Deliverables
- Working OAuth login
- Gmail emails syncing to database
- Contacts extracted and displayed
- Background sync every 5 minutes

## Week 2: Calendar Integration & Enhanced Sync

### Goals
- ✅ Google Calendar sync
- ✅ Meeting → Contact association
- ✅ Real-time updates via webhooks
- ✅ Unified timeline view

### Implementation Steps

#### Days 1-2: Calendar Integration
```typescript
// 1. Copy calendar components from Analog
// components/calendar/*

// 2. Add Google Calendar API
// lib/google/calendar-sync.ts
export async function syncCalendarEvents(accountId: string) {
  // Sync calendar events
  // Extract meeting attendees → contacts
}
```

#### Days 3-4: Webhook System
```typescript
// app/api/webhooks/google/route.ts
export async function POST(req: Request) {
  // Handle Gmail push notifications
  // Handle Calendar updates
  // Queue processing jobs
}
```

#### Day 5: Timeline View
```typescript
// 1. Create unified activity timeline
// components/timeline/activity-timeline.tsx
// Combine emails, meetings, notes
```

### Deliverables
- Calendar events syncing
- Email + Calendar in one timeline
- Real-time updates working
- Contact profiles with activities

## Week 3: AI Features & CRM Core

### Goals
- ✅ AI-powered contact enrichment
- ✅ Email composition with context
- ✅ Basic deal pipeline
- ✅ Notes and tasks

### Implementation Steps

#### Days 1-2: AI Infrastructure
```typescript
// 1. Set up Vercel AI SDK (adapt from Zero)
// lib/ai/providers.ts
export const ai = createAI({
  model: 'gpt-4-turbo',
  apiKey: process.env.OPENAI_API_KEY,
});

// 2. Contact enrichment
// lib/ai/enrichment.ts
export async function enrichContact(email: string) {
  // Search multiple sources
  // Return structured data
}
```

#### Days 3-4: CRM Features
```typescript
// 1. Extend Zero's notes to full CRM
// - Contact notes
// - Deal tracking
// - Task management

// 2. Create deal pipeline
// components/deals/pipeline.tsx
```

#### Day 5: AI Email Composition
```typescript
// 1. Adapt Zero's email composer
// 2. Add CRM context awareness
// components/compose/ai-composer.tsx
```

### Deliverables
- Contacts auto-enriched from web
- AI helps write contextual emails
- Basic deal tracking
- Notes on contacts/deals

## Week 4: Analytics & Agent System

### Goals
- ✅ Relationship analytics
- ✅ Basic agent framework
- ✅ Workflow automation
- ✅ Polish and optimization

### Implementation Steps

#### Days 1-2: Analytics Dashboard
```typescript
// 1. Create analytics components
// app/(dashboard)/analytics/page.tsx
// - Relationship strength scores
// - Communication patterns
// - Deal pipeline metrics
```

#### Days 3-4: Agent System
```typescript
// 1. Build agent framework
// lib/agents/base-agent.ts
export abstract class BaseAgent {
  abstract skills: Skill[];
  abstract execute(context: Context): Promise<Result>;
}

// 2. Create first agents
// - FollowUpAgent
// - LeadScoringAgent
// - MeetingPrepAgent
```

#### Day 5: Polish & Deploy
- Performance optimization
- Error handling
- Deploy to Vercel
- Set up monitoring

### Deliverables
- Analytics dashboard live
- 3 working AI agents
- Deployed to production
- Documentation complete

## Technical Decisions

### Architecture Choices
1. **Next.js 14 App Router** - Full-stack, great DX
2. **PostgreSQL + Drizzle** - Type-safe, performant
3. **tRPC** - End-to-end type safety
4. **Vercel** - Easy deployment, AI SDK
5. **pg-boss** - Reliable job queue

### Code Organization
```
fluffymail/
├── app/
│   ├── (auth)/
│   ├── (dashboard)/
│   ├── api/
│   └── layout.tsx
├── components/
│   ├── ui/ (from Zero)
│   ├── calendar/ (from Analog)
│   ├── contacts/
│   ├── deals/
│   └── ai/
├── lib/
│   ├── db/
│   ├── ai/
│   ├── gmail/
│   ├── agents/
│   └── utils/
├── server/
│   ├── routers/
│   └── trpc.ts
└── jobs/
    ├── sync/
    └── enrichment/
```

### Migration Strategy

1. **Start with Zero's structure** but use Next.js
2. **Copy UI components** wholesale
3. **Adapt API layer** from Hono to Next.js
4. **Reuse schemas** with CRM extensions
5. **Keep AI patterns** but use Vercel AI SDK

## Risk Mitigation

### Technical Risks
1. **Gmail API rate limits**
   - Solution: Implement exponential backoff
   - Use batch operations

2. **Large data volumes**
   - Solution: Pagination everywhere
   - Background processing
   - Database partitioning

3. **AI costs**
   - Solution: Aggressive caching
   - User quotas
   - Cheaper models for simple tasks

### Timeline Risks
1. **Week 1 overrun**
   - Mitigation: Can ship without calendar
   - Focus on email + contacts only

2. **AI enrichment complexity**
   - Mitigation: Start with simple sources
   - Add more providers later

3. **Agent system complexity**
   - Mitigation: Ship with simple rules first
   - Enhance with AI gradually

## Success Metrics

### Week 1
- [ ] 100% of emails synced
- [ ] 90%+ contact extraction accuracy
- [ ] <5s page load times

### Week 2  
- [ ] Calendar events synced
- [ ] Real-time updates working
- [ ] Unified timeline shipped

### Week 3
- [ ] 50%+ contacts enriched
- [ ] AI email drafts rated 4+/5
- [ ] Deal pipeline functional

### Week 4
- [ ] 3 agents deployed
- [ ] Analytics dashboard live
- [ ] <2s AI response time

## Team Structure

### Option 1: 2 Developers
- Dev 1: Backend, sync, AI
- Dev 2: Frontend, UI, UX
- Timeline: 4 weeks

### Option 2: 4 Developers
- Dev 1: Infrastructure, auth, database
- Dev 2: Sync engines, webhooks
- Dev 3: Frontend, components
- Dev 4: AI, agents, analytics
- Timeline: 2.5 weeks

## Conclusion

By leveraging existing code from Zero, Analog, and Nimbus, we can build FluffyMail's AI CRM in 4 weeks with 2 developers or 2.5 weeks with 4 developers. The key is to:

1. Start with Zero's foundation
2. Add calendar from Analog
3. Enhance with CRM features
4. Layer on AI capabilities
5. Ship iteratively

This approach saves 6+ weeks versus starting from scratch while delivering a production-ready AI CRM.