# Worker Architecture for FluffyMail AI CRM

## The Challenge

Vercel (and most serverless platforms) don't support long-running background processes. You need separate worker infrastructure.

## Architecture Options

### Option 1: Separate Worker Service (Recommended) ✅

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Next.js   │     │ PostgreSQL  │     │   Worker    │
│   (Vercel)  │────▶│  + pg-boss  │◀────│  Service    │
│             │     │   (Queue)   │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
```

**Implementation:**
```typescript
// worker-service/index.ts
import PgBoss from 'pg-boss';
import { drizzle } from 'drizzle-orm/postgres-js';

const boss = new PgBoss(process.env.DATABASE_URL);
const db = drizzle(process.env.DATABASE_URL);

async function start() {
  await boss.start();
  
  // Email sync worker
  await boss.work('sync-emails', async (job) => {
    const { connectionId, userId } = job.data;
    await syncEmailsForConnection(connectionId, userId);
  });
  
  // Contact extraction worker
  await boss.work('extract-contacts', { teamSize: 5 }, async (job) => {
    const { emailId } = job.data;
    await extractContactsFromEmail(emailId);
  });
  
  // Enrichment worker
  await boss.work('enrich-contact', { teamSize: 3 }, async (job) => {
    const { contactId } = job.data;
    await enrichContactData(contactId);
  });
  
  // AI analysis worker
  await boss.work('analyze-relationships', async (job) => {
    const { userId } = job.data;
    await analyzeUserRelationships(userId);
  });
}

start();
```

**Deployment Options:**

1. **Railway/Render** (Easiest)
```dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["npm", "run", "worker"]
```

2. **AWS ECS/Fargate**
```typescript
// terraform/ecs.tf
resource "aws_ecs_service" "worker" {
  name            = "fluffymail-worker"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.worker.arn
  desired_count   = 2 // Scale as needed
}
```

3. **Fly.io** (Great DX)
```toml
# fly.toml
app = "fluffymail-worker"

[processes]
worker = "npm run worker"

[env]
  NODE_ENV = "production"

[[services]]
  processes = ["worker"]
  internal_port = 3000
```

### Option 2: Serverless Workers (Complex)

Using Cloudflare Queues or AWS SQS + Lambda:
```typescript
// Cloudflare Worker with Queues
export default {
  async queue(batch: MessageBatch, env: Env): Promise<void> {
    for (const message of batch.messages) {
      const { type, data } = message.body;
      
      switch (type) {
        case 'sync-emails':
          await syncEmails(data);
          break;
        case 'extract-contacts':
          await extractContacts(data);
          break;
      }
      
      message.ack();
    }
  },
};
```

### Option 3: Temporal/Inngest (Workflow Engines)

**Inngest Example** (Recommended for complex workflows):
```typescript
// inngest/functions/sync-email.ts
import { inngest } from './client';

export const syncEmailAccount = inngest.createFunction(
  { id: 'sync-email-account' },
  { event: 'email/sync.requested' },
  async ({ event, step }) => {
    // Step 1: Fetch emails
    const emails = await step.run('fetch-emails', async () => {
      return await fetchEmailsFromGmail(event.data.connectionId);
    });
    
    // Step 2: Store in database (parallel)
    await step.run('store-emails', async () => {
      return await Promise.all(
        emails.map(email => storeEmail(email))
      );
    });
    
    // Step 3: Extract contacts (fan-out)
    const contactJobs = emails.map(email => ({
      name: `extract-contacts-${email.id}`,
      data: { emailId: email.id }
    }));
    
    await step.sendEvents('extract-contacts', contactJobs);
    
    // Step 4: Trigger analysis
    await step.sendEvent('analyze-relationships', {
      name: 'relationships/analyze.requested',
      data: { userId: event.data.userId }
    });
  }
);
```

## Recommended Architecture for FluffyMail

### 1. **Hybrid Approach**
```
┌─────────────────┐
│   Next.js App   │
│   (Vercel)      │
├─────────────────┤
│ - Web UI        │
│ - API Routes    │
│ - Queue Jobs    │
└────────┬────────┘
         │ 
         ▼
┌─────────────────┐     ┌─────────────────┐
│   PostgreSQL    │     │  Worker Service │
│   + pg-boss     │◀────│   (Railway)     │
└─────────────────┘     ├─────────────────┤
                        │ - Email Sync    │
                        │ - Enrichment    │
                        │ - AI Analysis   │
                        └─────────────────┘
```

### 2. **Job Scheduling from Next.js**
```typescript
// app/api/connections/sync/route.ts
import PgBoss from 'pg-boss';

const boss = new PgBoss(process.env.DATABASE_URL);

export async function POST(req: Request) {
  const { connectionId } = await req.json();
  
  // Schedule immediate sync
  await boss.send('sync-emails', {
    connectionId,
    userId: session.user.id
  });
  
  // Schedule recurring sync
  await boss.schedule(
    'sync-emails',
    '*/15 * * * *', // Every 15 minutes
    { connectionId, userId: session.user.id }
  );
  
  return Response.json({ success: true });
}
```

### 3. **Worker Implementation**
```typescript
// workers/src/jobs/email-sync.ts
export async function syncEmails(job: Job) {
  const { connectionId, userId } = job.data;
  
  // 1. Get connection details
  const connection = await db
    .select()
    .from(connections)
    .where(eq(connections.id, connectionId))
    .limit(1);
  
  // 2. Initialize Gmail client
  const gmail = new GoogleGmail({
    accessToken: connection.accessToken,
    refreshToken: connection.refreshToken
  });
  
  // 3. Fetch emails (with pagination)
  let pageToken = null;
  do {
    const response = await gmail.messages.list({
      pageToken,
      maxResults: 100,
      q: 'newer_than:2d' // Last 2 days for incremental
    });
    
    // 4. Process batch
    for (const message of response.messages) {
      // Store email
      await db.insert(emails).values({
        messageId: message.id,
        userId,
        // ... other fields
      });
      
      // Queue contact extraction
      await boss.send('extract-contacts', {
        emailId: message.id
      });
    }
    
    pageToken = response.nextPageToken;
  } while (pageToken);
  
  // 5. Queue relationship analysis
  await boss.send('analyze-relationships', { userId });
}
```

## Development Setup

### Local Development
```bash
# Terminal 1: Next.js app
npm run dev

# Terminal 2: Worker service
npm run worker:dev
```

### Environment Variables
```env
# .env.local (Next.js)
DATABASE_URL=postgresql://...
REDIS_URL=redis://...

# .env.worker (Worker service)
DATABASE_URL=postgresql://... # Same DB
REDIS_URL=redis://...
OPENAI_API_KEY=sk-...
```

## Scaling Considerations

### 1. **Worker Pooling**
```typescript
// Control concurrency
await boss.work('extract-contacts', {
  teamSize: 10, // 10 concurrent workers
  teamConcurrency: 5 // 5 jobs per worker
}, handler);
```

### 2. **Priority Queues**
```typescript
// High priority for new emails
await boss.send('sync-emails', data, {
  priority: 10,
  retryLimit: 3,
  retryDelay: 60
});

// Low priority for historical analysis
await boss.send('analyze-history', data, {
  priority: 1,
  startAfter: '10 minutes'
});
```

### 3. **Monitoring**
```typescript
// Add job monitoring
boss.on('job-failed', (job) => {
  console.error(`Job ${job.name} failed:`, job.lastError);
  // Send to Sentry/monitoring
});

boss.on('job-complete', (job) => {
  console.log(`Job ${job.name} completed in ${job.completedIn}ms`);
  // Track metrics
});
```

## Key Decisions

1. **Use Railway/Render** for worker hosting (easy, affordable)
2. **Start with pg-boss** (simpler than Temporal/Inngest)
3. **Single worker service** that handles all job types
4. **Scale horizontally** by adding more worker instances
5. **Monitor everything** - jobs are critical for CRM

This gives you a production-ready worker architecture that can handle millions of emails and scale with your user base!