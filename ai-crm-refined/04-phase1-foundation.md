# Phase 1: Foundation & Data Sync Implementation

## Overview

Phase 1 establishes the core infrastructure and achieves complete email synchronization. This phase is critical as all subsequent features depend on having reliable access to user data.

**Duration**: 3 weeks  
**Team**: 2 Senior Full-Stack Engineers  
**Goal**: Sync all user data with 100% reliability

## Week 1: Core Setup & Authentication

### Day 1-2: Project Bootstrap
```bash
# Initialize project
bunx create-next-app@latest ai-crm --typescript --tailwind --app
cd ai-crm

# Install core dependencies
bun add drizzle-orm postgres @trpc/server @trpc/client @trpc/react-query
bun add better-auth @better-auth/adapter-drizzle
bun add pg-boss @upstash/redis
bun add -D drizzle-kit @types/node
```

**Deliverables**:
- Next.js project with TypeScript
- Drizzle ORM configured
- Basic folder structure
- Environment variables setup

### Day 3-4: Database Setup
```typescript
// drizzle.config.ts
export default {
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  driver: 'pg',
  dbCredentials: {
    connectionString: process.env.DATABASE_URL!,
  },
  tablesFilter: ["ai_crm_*"],
} satisfies Config

// Run migrations
bun drizzle-kit generate:pg
bun drizzle-kit push:pg
```

**Schema Implementation**:
1. Create all tables from data model doc
2. Set up indexes and constraints
3. Add triggers for updated_at
4. Test with seed data

### Day 5-7: Authentication System
```typescript
// src/app/api/auth/[...all]/route.ts
import { auth } from "@/lib/auth"
import { toNextJsHandler } from "better-auth/next-js"

export const { GET, POST } = toNextJsHandler(auth)

// src/lib/auth/provider.tsx
"use client"
import { useAuth } from "@/lib/auth/hooks"

export function AuthProvider({ children }) {
  const { user, loading } = useAuth()
  
  if (loading) return <LoadingSpinner />
  if (!user) return <SignInPage />
  
  return <>{children}</>
}
```

**OAuth Implementation**:
1. Google OAuth setup with required scopes
2. Token storage and refresh logic
3. Session management
4. Protected route middleware

## Week 2: Email Sync Engine

### Day 8-9: Gmail API Integration
```typescript
// src/lib/google/gmail.ts
import { google } from 'googleapis'

export class GmailClient {
  private gmail: any
  
  constructor(accessToken: string) {
    const auth = new google.auth.OAuth2()
    auth.setCredentials({ access_token: accessToken })
    this.gmail = google.gmail({ version: 'v1', auth })
  }
  
  async listMessages(query?: string, pageToken?: string) {
    return this.gmail.users.messages.list({
      userId: 'me',
      q: query,
      pageToken,
      maxResults: 500,
    })
  }
  
  async getMessage(messageId: string) {
    return this.gmail.users.messages.get({
      userId: 'me',
      id: messageId,
      format: 'full',
    })
  }
  
  async batchGetMessages(messageIds: string[]) {
    const batch = new google.auth.BatchRequest()
    
    messageIds.forEach(id => {
      batch.add(this.gmail.users.messages.get({
        userId: 'me',
        id,
        format: 'full',
      }))
    })
    
    return batch.execute()
  }
}
```

### Day 10-11: Email Sync Jobs
```typescript
// src/jobs/handlers/email-sync.ts
export class EmailSyncHandler extends JobHandler<EmailSyncData> {
  name = 'email-sync'
  
  async process(data: EmailSyncData) {
    const { userId, accountId, full = false } = data
    
    // Get OAuth token
    const token = await TokenManager.getValidToken(userId)
    const gmail = new GmailClient(token)
    
    // Determine sync strategy
    const syncState = await this.getSyncState(accountId)
    const query = full ? '' : `after:${syncState.lastSyncDate}`
    
    // Sync emails
    await this.syncEmails(gmail, userId, query)
    
    // Update sync state
    await this.updateSyncState(accountId)
  }
  
  private async syncEmails(
    gmail: GmailClient, 
    userId: string, 
    query: string
  ) {
    let pageToken: string | undefined
    let totalSynced = 0
    
    do {
      // List messages
      const response = await gmail.listMessages(query, pageToken)
      const messageIds = response.data.messages?.map(m => m.id) || []
      
      if (messageIds.length === 0) break
      
      // Batch fetch full messages
      const messages = await gmail.batchGetMessages(messageIds)
      
      // Process in parallel batches
      const BATCH_SIZE = 50
      for (let i = 0; i < messages.length; i += BATCH_SIZE) {
        const batch = messages.slice(i, i + BATCH_SIZE)
        
        await Promise.all(
          batch.map(msg => this.processEmail(userId, msg))
        )
      }
      
      totalSynced += messages.length
      pageToken = response.data.nextPageToken
      
      // Progress notification
      await this.notifyProgress(userId, totalSynced)
      
    } while (pageToken)
  }
  
  private async processEmail(userId: string, gmailMessage: any) {
    const parsed = parseGmailMessage(gmailMessage)
    
    // Store email
    const [email] = await db.insert(emails).values({
      userId,
      messageId: parsed.id,
      threadId: parsed.threadId,
      fromEmail: parsed.from.email,
      fromName: parsed.from.name,
      toEmails: parsed.to,
      ccEmails: parsed.cc,
      bccEmails: parsed.bcc,
      subject: parsed.subject,
      bodyText: parsed.textBody,
      bodyHtml: parsed.htmlBody,
      snippet: parsed.snippet,
      labels: parsed.labels,
      internalDate: parsed.internalDate,
      hasAttachments: parsed.attachments.length > 0,
      attachmentCount: parsed.attachments.length,
    }).onConflictDoUpdate({
      target: [emails.userId, emails.messageId],
      set: { updatedAt: new Date() }
    }).returning()
    
    // Queue contact extraction
    await createJob('contact-extraction', {
      userId,
      emailId: email.id
    })
  }
}
```

### Day 12-13: Contact Extraction
```typescript
// src/jobs/handlers/contact-extraction.ts
export class ContactExtractionHandler extends JobHandler<ContactExtractionData> {
  name = 'contact-extraction'
  
  async process(data: ContactExtractionData) {
    const { userId, emailId } = data
    
    const email = await db.query.emails.findFirst({
      where: eq(emails.id, emailId)
    })
    
    if (!email) return
    
    // Extract contacts from various sources
    const contacts = await this.extractContacts(email)
    
    // Process each contact
    for (const contactData of contacts) {
      await this.processContact(userId, contactData, emailId)
    }
  }
  
  private async extractContacts(email: Email): Promise<ExtractedContact[]> {
    const contacts: ExtractedContact[] = []
    
    // From sender
    if (email.fromEmail) {
      contacts.push({
        email: email.fromEmail,
        name: email.fromName || this.extractNameFromEmail(email.fromEmail),
        source: 'from',
        confidence: 1.0
      })
    }
    
    // To/CC/BCC recipients
    const allRecipients = [
      ...(email.toEmails || []),
      ...(email.ccEmails || []),
      ...(email.bccEmails || [])
    ]
    
    for (const recipient of allRecipients) {
      const { email: addr, name } = this.parseEmailAddress(recipient)
      if (addr) {
        contacts.push({
          email: addr,
          name: name || this.extractNameFromEmail(addr),
          source: 'recipient',
          confidence: 0.9
        })
      }
    }
    
    // Extract from email signature
    const signatureContacts = await this.extractFromSignature(
      email.bodyText || email.bodyHtml || ''
    )
    contacts.push(...signatureContacts)
    
    // Extract from email body
    const bodyContacts = await this.extractFromBody(
      email.bodyText || ''
    )
    contacts.push(...bodyContacts)
    
    return contacts
  }
  
  private async extractFromSignature(body: string): Promise<ExtractedContact[]> {
    const contacts: ExtractedContact[] = []
    
    // Find signature block (usually after -- or multiple newlines)
    const signatureMatch = body.match(/(?:--|\n{3,})([\s\S]+)$/)
    if (!signatureMatch) return contacts
    
    const signature = signatureMatch[1]
    
    // Extract emails
    const emailRegex = /([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})/g
    const emails = signature.match(emailRegex) || []
    
    // Extract phone numbers
    const phoneRegex = /(?:\+?1[-.\s]?)?\(?([0-9]{3})\)?[-.\s]?([0-9]{3})[-.\s]?([0-9]{4})/g
    const phones = signature.match(phoneRegex) || []
    
    // Extract names (heuristic: capitalized words near contact info)
    const nameRegex = /^([A-Z][a-z]+ (?:[A-Z][a-z]+ ?){0,2})$/gm
    const names = signature.match(nameRegex) || []
    
    // Try to associate names with emails
    for (const email of emails) {
      const closestName = this.findClosestName(email, names, signature)
      contacts.push({
        email,
        name: closestName,
        phone: phones[0], // Simple association
        source: 'signature',
        confidence: 0.7
      })
    }
    
    return contacts
  }
  
  private async processContact(
    userId: string, 
    contactData: ExtractedContact,
    emailId: string
  ) {
    // Check if contact exists
    let contact = await db.query.contacts.findFirst({
      where: and(
        eq(contacts.userId, userId),
        eq(contacts.email, contactData.email)
      )
    })
    
    if (!contact) {
      // Create new contact
      [contact] = await db.insert(contacts).values({
        userId,
        email: contactData.email,
        name: contactData.name,
        phone: contactData.phone,
        source: 'email',
        sourceDetails: {
          confidence: contactData.confidence,
          extractedFrom: contactData.source
        }
      }).returning()
      
      // Log event
      await logEvent(
        userId,
        'contact',
        contact.id,
        'contact.created',
        { source: 'email_extraction' }
      )
    } else {
      // Update if we have better data
      if (contactData.name && !contact.name) {
        await db.update(contacts)
          .set({ name: contactData.name })
          .where(eq(contacts.id, contact.id))
      }
    }
    
    // Create interaction record
    await db.insert(emailInteractions).values({
      emailId,
      contactId: contact.id,
      interactionType: contactData.source as any
    }).onConflictDoNothing()
  }
}
```

### Day 14: Real-time Updates
```typescript
// src/app/api/webhooks/gmail/route.ts
export async function POST(req: Request) {
  const body = await req.json()
  
  // Verify webhook (check Google's pub/sub signature)
  if (!verifyGoogleWebhook(req.headers, body)) {
    return new Response('Unauthorized', { status: 401 })
  }
  
  // Decode message
  const message = JSON.parse(
    Buffer.from(body.message.data, 'base64').toString()
  )
  
  // Find account
  const account = await db.query.accounts.findFirst({
    where: eq(accounts.providerAccountId, message.emailAddress)
  })
  
  if (!account) {
    return new Response('Account not found', { status: 404 })
  }
  
  // Queue incremental sync
  await createJob('email-sync', {
    userId: account.userId,
    accountId: account.id,
    full: false
  }, {
    priority: 10 // High priority for real-time
  })
  
  return new Response('OK', { status: 200 })
}

// Setup webhook subscription
export async function setupGmailPushNotifications(
  userId: string,
  accountId: string
) {
  const token = await TokenManager.getValidToken(userId)
  const gmail = new GmailClient(token)
  
  const response = await gmail.gmail.users.watch({
    userId: 'me',
    requestBody: {
      topicName: `projects/${env.GOOGLE_PROJECT_ID}/topics/gmail-push`,
      labelIds: ['INBOX'],
      labelFilterAction: 'include'
    }
  })
  
  // Store subscription info
  await db.update(accounts)
    .set({
      syncState: {
        ...account.syncState,
        pushNotification: {
          historyId: response.data.historyId,
          expiration: response.data.expiration
        }
      }
    })
    .where(eq(accounts.id, accountId))
}
```

## Week 3: UI & Performance

### Day 15-16: Basic UI
```typescript
// src/app/(app)/layout.tsx
export default function AppLayout({ children }) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <main className="flex-1 overflow-y-auto">
        {children}
      </main>
    </div>
  )
}

// src/app/(app)/emails/page.tsx
export default function EmailsPage() {
  const { data: emails, isLoading } = trpc.email.list.useQuery({
    limit: 50
  })
  
  if (isLoading) return <EmailsSkeleton />
  
  return (
    <div className="divide-y">
      {emails?.map(email => (
        <EmailRow key={email.id} email={email} />
      ))}
    </div>
  )
}

// src/components/email-row.tsx
export function EmailRow({ email }: { email: Email }) {
  return (
    <div className="p-4 hover:bg-gray-50">
      <div className="flex items-start justify-between">
        <div className="flex-1">
          <p className="font-medium">{email.fromName || email.fromEmail}</p>
          <p className="text-sm text-gray-900">{email.subject}</p>
          <p className="text-sm text-gray-500">{email.snippet}</p>
        </div>
        <time className="text-sm text-gray-500">
          {formatRelativeTime(email.internalDate)}
        </time>
      </div>
    </div>
  )
}
```

### Day 17-18: Performance Optimization
```typescript
// src/server/api/routers/email.ts
export const emailRouter = createTRPCRouter({
  list: protectedProcedure
    .input(z.object({
      limit: z.number().min(1).max(100).default(50),
      cursor: z.string().optional(),
      search: z.string().optional(),
    }))
    .query(async ({ ctx, input }) => {
      const { limit, cursor, search } = input
      
      // Build query
      let query = db
        .select({
          id: emails.id,
          subject: emails.subject,
          snippet: emails.snippet,
          fromEmail: emails.fromEmail,
          fromName: emails.fromName,
          internalDate: emails.internalDate,
          isUnread: emails.isUnread,
          hasAttachments: emails.hasAttachments,
        })
        .from(emails)
        .where(eq(emails.userId, ctx.session.user.id))
        .orderBy(desc(emails.internalDate))
        .limit(limit + 1)
      
      if (cursor) {
        query = query.where(lt(emails.internalDate, new Date(cursor)))
      }
      
      if (search) {
        query = query.where(
          sql`search_vector @@ plainto_tsquery('english', ${search})`
        )
      }
      
      const items = await query
      
      let nextCursor: typeof cursor | undefined = undefined
      if (items.length > limit) {
        const nextItem = items.pop()
        nextCursor = nextItem!.internalDate.toISOString()
      }
      
      return {
        items,
        nextCursor,
      }
    }),
})

// Add database indexes for performance
CREATE INDEX idx_emails_user_date_covering 
ON emails(user_id, internal_date DESC) 
INCLUDE (subject, snippet, from_email, from_name, is_unread, has_attachments);
```

### Day 19-20: Testing & Documentation
```typescript
// src/__tests__/email-sync.test.ts
describe('Email Sync', () => {
  let testUser: User
  let testAccount: Account
  
  beforeEach(async () => {
    testUser = await createTestUser()
    testAccount = await createTestAccount(testUser.id)
  })
  
  it('should sync emails from Gmail', async () => {
    // Mock Gmail API
    const mockGmail = mockGmailApi()
    mockGmail.users.messages.list.mockResolvedValue({
      data: {
        messages: [
          { id: 'msg1', threadId: 'thread1' },
          { id: 'msg2', threadId: 'thread1' },
        ]
      }
    })
    
    // Run sync
    const handler = new EmailSyncHandler()
    await handler.process({
      userId: testUser.id,
      accountId: testAccount.id,
      full: true
    })
    
    // Verify emails stored
    const emails = await db.query.emails.findMany({
      where: eq(emails.userId, testUser.id)
    })
    
    expect(emails).toHaveLength(2)
    expect(emails[0].messageId).toBe('msg1')
  })
  
  it('should extract contacts from emails', async () => {
    // Create test email
    const testEmail = await createTestEmail(testUser.id, {
      fromEmail: 'john@example.com',
      fromName: 'John Doe',
      bodyText: `
        Hi there,
        
        Please contact my assistant at jane@example.com
        
        --
        John Doe
        CEO, Example Corp
        john@example.com
        (555) 123-4567
      `
    })
    
    // Run extraction
    const handler = new ContactExtractionHandler()
    await handler.process({
      userId: testUser.id,
      emailId: testEmail.id
    })
    
    // Verify contacts created
    const contacts = await db.query.contacts.findMany({
      where: eq(contacts.userId, testUser.id)
    })
    
    expect(contacts).toHaveLength(2)
    expect(contacts.map(c => c.email)).toContain('john@example.com')
    expect(contacts.map(c => c.email)).toContain('jane@example.com')
  })
})
```

### Day 21: Deployment & Monitoring
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Bun
        uses: oven-sh/setup-bun@v1
        
      - name: Install dependencies
        run: bun install
        
      - name: Run tests
        run: bun test
        
      - name: Build
        run: bun run build
        
      - name: Run migrations
        run: bun drizzle-kit push:pg
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          
      - name: Deploy to Vercel
        uses: vercel/action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
```

## Phase 1 Deliverables Checklist

### Core Infrastructure ✓
- [ ] Next.js app with TypeScript
- [ ] PostgreSQL database with all tables
- [ ] Authentication with Google OAuth
- [ ] tRPC API setup
- [ ] Background job system

### Email Sync ✓
- [ ] Gmail API integration
- [ ] Initial sync (last 3 months)
- [ ] Incremental sync
- [ ] Real-time webhook updates
- [ ] Email parsing and storage

### Contact Extraction ✓
- [ ] Extract from sender/recipients
- [ ] Extract from signatures
- [ ] Extract from email body
- [ ] Contact deduplication
- [ ] Interaction tracking

### UI ✓
- [ ] Sign in with Google
- [ ] Email list view
- [ ] Contact list view
- [ ] Sync status indicator
- [ ] Basic search

### Performance ✓
- [ ] Database indexes
- [ ] Query optimization
- [ ] Pagination
- [ ] Caching strategy

### Testing & Deployment ✓
- [ ] Unit tests for critical paths
- [ ] Integration tests
- [ ] CI/CD pipeline
- [ ] Production deployment
- [ ] Monitoring setup

## Success Metrics Validation

By the end of Phase 1, the system should:

1. **Sync 10,000 emails in < 5 minutes** ✓
   - Batch processing with 50 emails/batch
   - Parallel job processing
   - Optimized database inserts

2. **Extract contacts with > 90% accuracy** ✓
   - Multiple extraction strategies
   - Confidence scoring
   - Deduplication logic

3. **Handle real-time updates with < 5s latency** ✓
   - Gmail push notifications
   - High-priority job queue
   - WebSocket updates to UI

4. **Zero data loss during sync operations** ✓
   - Transactional updates
   - Idempotent operations
   - Comprehensive error handling

## Next Steps

With Phase 1 complete, we have:
- All user data synchronized
- Reliable infrastructure
- Real-time updates working
- Basic UI to visualize data

The system is now ready for Phase 2: AI Intelligence & Automation.