# Background Jobs & Event System Architecture

## Overview

This document outlines the background job processing and event-driven architecture that powers async operations, enrichment, and real-time features in our AI CRM.

## Background Job System

### 1. Job Queue Architecture

We'll use PostgreSQL as our job queue for simplicity (can migrate to Redis/BullMQ later).

```typescript
// Job definitions
interface Job<T = any> {
  id: string
  type: JobType
  data: T
  status: JobStatus
  priority: 'low' | 'normal' | 'high' | 'critical'
  attempts: number
  maxAttempts: number
  createdAt: Date
  scheduledFor: Date
  startedAt?: Date
  completedAt?: Date
  failedAt?: Date
  error?: string
  result?: any
  metadata?: {
    userId?: string
    correlationId?: string
    parentJobId?: string
  }
}

type JobStatus = 
  | 'pending'
  | 'scheduled'
  | 'processing'
  | 'completed'
  | 'failed'
  | 'cancelled'
  | 'dead'

type JobType =
  // Email sync jobs
  | 'email-sync-initial'
  | 'email-sync-incremental'
  | 'email-sync-deep'
  | 'process-email-batch'
  
  // Contact jobs
  | 'contact-extraction'
  | 'contact-enrichment'
  | 'contact-stats-calculation'
  
  // Enrichment jobs
  | 'enrich-twitter'
  | 'enrich-linkedin'
  | 'enrich-clearbit'
  
  // Analytics jobs
  | 'calculate-relationship-scores'
  | 'generate-contact-insights'
  
  // Maintenance jobs
  | 'cleanup-old-events'
  | 'renew-gmail-watch'
  | 'refresh-oauth-tokens'
```

### 2. Job Queue Implementation

```typescript
class JobQueue {
  private workers: Map<JobType, JobWorker> = new Map()
  private isRunning = false
  private pollInterval: NodeJS.Timer | null = null
  
  // Register a worker for a job type
  register<T>(type: JobType, worker: JobWorker<T>) {
    this.workers.set(type, worker)
  }
  
  // Enqueue a job
  async enqueue<T>(params: {
    type: JobType
    data: T
    options?: JobOptions
  }): Promise<string> {
    const job = await db.insert(backgroundJobs).values({
      id: generateId(),
      type: params.type,
      data: params.data,
      status: 'pending',
      priority: params.options?.priority || 'normal',
      attempts: 0,
      maxAttempts: params.options?.maxAttempts || 3,
      scheduledFor: params.options?.runAt || new Date(),
      metadata: params.options?.metadata
    }).returning()
    
    // Emit event
    await eventBus.emit('job.enqueued', {
      jobId: job[0].id,
      type: params.type
    })
    
    return job[0].id
  }
  
  // Start processing jobs
  async start() {
    this.isRunning = true
    this.pollInterval = setInterval(() => this.poll(), 1000)
    console.log('Job queue started')
  }
  
  // Poll for jobs
  private async poll() {
    if (!this.isRunning) return
    
    // Use advisory lock to prevent double processing
    const job = await db.transaction(async (tx) => {
      const result = await tx.execute(sql`
        SELECT * FROM background_jobs
        WHERE status = 'pending'
          AND scheduled_for <= NOW()
        ORDER BY
          CASE priority
            WHEN 'critical' THEN 1
            WHEN 'high' THEN 2
            WHEN 'normal' THEN 3
            WHEN 'low' THEN 4
          END,
          created_at
        LIMIT 1
        FOR UPDATE SKIP LOCKED
      `)
      
      if (result.rows.length === 0) return null
      
      const job = result.rows[0]
      
      // Mark as processing
      await tx.update(backgroundJobs)
        .set({
          status: 'processing',
          startedAt: new Date(),
          attempts: job.attempts + 1
        })
        .where(eq(backgroundJobs.id, job.id))
      
      return job
    })
    
    if (job) {
      await this.processJob(job)
    }
  }
  
  // Process a single job
  private async processJob(job: Job) {
    const worker = this.workers.get(job.type)
    if (!worker) {
      await this.failJob(job, new Error(`No worker registered for ${job.type}`))
      return
    }
    
    try {
      // Set job context for tracing
      const context = {
        jobId: job.id,
        userId: job.metadata?.userId,
        correlationId: job.metadata?.correlationId
      }
      
      // Execute worker
      const result = await withContext(context, () => worker.process(job))
      
      // Mark complete
      await db.update(backgroundJobs)
        .set({
          status: 'completed',
          completedAt: new Date(),
          result
        })
        .where(eq(backgroundJobs.id, job.id))
      
      // Emit event
      await eventBus.emit('job.completed', {
        jobId: job.id,
        type: job.type,
        duration: Date.now() - job.startedAt.getTime()
      })
      
    } catch (error) {
      await this.handleJobError(job, error)
    }
  }
  
  // Handle job failure
  private async handleJobError(job: Job, error: Error) {
    const shouldRetry = job.attempts < job.maxAttempts && this.isRetryable(error)
    
    if (shouldRetry) {
      // Calculate backoff
      const delay = this.calculateBackoff(job.attempts)
      
      await db.update(backgroundJobs)
        .set({
          status: 'scheduled',
          scheduledFor: new Date(Date.now() + delay),
          error: error.message
        })
        .where(eq(backgroundJobs.id, job.id))
      
      await eventBus.emit('job.retrying', {
        jobId: job.id,
        attempt: job.attempts,
        nextRunAt: new Date(Date.now() + delay)
      })
    } else {
      await this.failJob(job, error)
    }
  }
  
  private calculateBackoff(attempt: number): number {
    // Exponential backoff with jitter
    const base = Math.pow(2, attempt) * 1000
    const jitter = Math.random() * 1000
    return Math.min(base + jitter, 300000) // Max 5 minutes
  }
}
```

### 3. Job Workers

#### Email Sync Worker
```typescript
class EmailSyncWorker implements JobWorker<EmailSyncData> {
  async process(job: Job<EmailSyncData>) {
    const { userId, syncType } = job.data
    
    // Set up progress tracking
    const progress = new SyncProgress(job.id)
    
    try {
      switch (syncType) {
        case 'initial':
          await this.performInitialSync(userId, progress)
          break
        case 'incremental':
          await this.performIncrementalSync(userId, progress)
          break
        case 'deep':
          await this.performDeepSync(userId, progress)
          break
      }
      
      return progress.getSummary()
    } finally {
      await progress.complete()
    }
  }
  
  private async performInitialSync(userId: string, progress: SyncProgress) {
    // Get recent emails (last 3 months)
    const emails = await gmail.listMessages({
      q: 'newer_than:3m',
      maxResults: 500
    })
    
    progress.setTotal(emails.length)
    
    // Process in chunks
    for (const chunk of chunks(emails, 50)) {
      await jobQueue.enqueue({
        type: 'process-email-batch',
        data: {
          userId,
          messageIds: chunk.map(e => e.id)
        },
        options: {
          metadata: { parentJobId: progress.jobId }
        }
      })
      
      progress.incrementQueued(chunk.length)
    }
  }
}
```

#### Contact Enrichment Worker
```typescript
class ContactEnrichmentWorker implements JobWorker<ContactEnrichmentData> {
  private enrichers: Map<string, Enricher> = new Map([
    ['twitter', new TwitterEnricher()],
    ['linkedin', new LinkedInEnricher()],
    ['clearbit', new ClearbitEnricher()]
  ])
  
  async process(job: Job<ContactEnrichmentData>) {
    const { contactId, sources = ['twitter'] } = job.data
    
    // Get contact
    const contact = await db.query.contacts.findFirst({
      where: eq(contacts.id, contactId)
    })
    
    if (!contact) throw new Error('Contact not found')
    
    const results: EnrichmentResult[] = []
    
    // Run enrichments in parallel
    const enrichmentPromises = sources.map(async (source) => {
      const enricher = this.enrichers.get(source)
      if (!enricher) return null
      
      try {
        const data = await enricher.enrich(contact)
        
        // Store enrichment
        await db.insert(enrichments).values({
          contactId,
          source,
          data,
          fetchedAt: new Date()
        })
        
        results.push({ source, success: true, data })
      } catch (error) {
        results.push({ source, success: false, error: error.message })
      }
    })
    
    await Promise.all(enrichmentPromises)
    
    // Update contact with enriched data
    await this.updateContactWithEnrichments(contactId, results)
    
    // Emit event
    await eventBus.emit('contact.enriched', {
      contactId,
      sources: results.filter(r => r.success).map(r => r.source)
    })
    
    return results
  }
}
```

#### Twitter Enricher
```typescript
class TwitterEnricher implements Enricher {
  async enrich(contact: Contact): Promise<EnrichmentData> {
    // Find Twitter handle
    let handle = contact.twitterHandle
    
    if (!handle && contact.email) {
      // Try to find handle from email/name
      handle = await this.findTwitterHandle(contact.email, contact.name)
    }
    
    if (!handle) throw new Error('No Twitter handle found')
    
    // Fetch recent tweets
    const tweets = await this.fetchRecentTweets(handle)
    
    // Analyze activity
    const analysis = {
      handle,
      followerCount: tweets.user.followersCount,
      tweetCount: tweets.user.tweetsCount,
      recentTweets: tweets.data.slice(0, 10).map(t => ({
        id: t.id,
        text: t.text,
        createdAt: t.createdAt,
        likes: t.publicMetrics.likeCount,
        retweets: t.publicMetrics.retweetCount
      })),
      topics: this.extractTopics(tweets.data),
      activityLevel: this.calculateActivityLevel(tweets.data),
      lastActiveAt: tweets.data[0]?.createdAt
    }
    
    return analysis
  }
  
  private async fetchRecentTweets(handle: string) {
    // Use Twitter API v2
    const response = await fetch(`https://api.twitter.com/2/users/by/username/${handle}/tweets`, {
      headers: {
        'Authorization': `Bearer ${process.env.TWITTER_BEARER_TOKEN}`
      },
      params: {
        'max_results': 100,
        'tweet.fields': 'created_at,public_metrics',
        'user.fields': 'public_metrics'
      }
    })
    
    return response.json()
  }
}
```

## Event System Architecture

### 1. Event Store Design

```typescript
interface Event {
  id: string
  type: EventType
  version: number
  userId: string
  aggregateId?: string
  aggregateType?: string
  data: JsonValue
  metadata: {
    source: string
    correlationId?: string
    causationId?: string
    timestamp: Date
    ipAddress?: string
    userAgent?: string
  }
  createdAt: Date
}

// Event types organized by domain
type EventType =
  // User events
  | 'user.signed_up'
  | 'user.signed_in'
  | 'user.signed_out'
  | 'user.profile_updated'
  
  // Account events
  | 'account.linked'
  | 'account.unlinked'
  | 'account.token_refreshed'
  
  // Email events
  | 'email.synced'
  | 'email.sent'
  | 'email.received'
  | 'email.thread_updated'
  
  // Contact events
  | 'contact.created'
  | 'contact.updated'
  | 'contact.enriched'
  | 'contact.merged'
  | 'contact.tagged'
  
  // Interaction events
  | 'interaction.recorded'
  | 'interaction.email_sent'
  | 'interaction.email_received'
  
  // AI events
  | 'ai.query_executed'
  | 'ai.suggestion_generated'
  | 'ai.action_taken'
  
  // System events
  | 'job.enqueued'
  | 'job.completed'
  | 'job.failed'
  | 'sync.started'
  | 'sync.completed'
```

### 2. Event Bus Implementation

```typescript
class EventBus {
  private handlers: Map<EventType, Set<EventHandler>> = new Map()
  private middleware: EventMiddleware[] = []
  
  // Emit an event
  async emit(type: EventType, data: any, metadata?: Partial<EventMetadata>) {
    const event: Event = {
      id: generateId(),
      type,
      version: 1,
      userId: getCurrentUserId(),
      data,
      metadata: {
        source: 'crm-app',
        timestamp: new Date(),
        correlationId: getCorrelationId(),
        ...metadata
      },
      createdAt: new Date()
    }
    
    // Apply middleware
    for (const mw of this.middleware) {
      await mw.before?.(event)
    }
    
    // Store event
    await db.insert(events).values(event)
    
    // Notify listeners via PostgreSQL
    await db.execute(sql`
      NOTIFY events, ${JSON.stringify({
        type: event.type,
        eventId: event.id
      })}
    `)
    
    // Process synchronous handlers
    const handlers = this.handlers.get(type) || new Set()
    for (const handler of handlers) {
      if (handler.sync) {
        try {
          await handler.handle(event)
        } catch (error) {
          console.error(`Sync handler failed for ${type}:`, error)
        }
      }
    }
    
    // Queue async handlers
    for (const handler of handlers) {
      if (!handler.sync) {
        await jobQueue.enqueue({
          type: 'process-event',
          data: { eventId: event.id, handlerName: handler.name }
        })
      }
    }
    
    // Apply middleware
    for (const mw of this.middleware) {
      await mw.after?.(event)
    }
    
    return event.id
  }
  
  // Subscribe to events
  on(type: EventType | EventType[], handler: EventHandler) {
    const types = Array.isArray(type) ? type : [type]
    
    for (const t of types) {
      if (!this.handlers.has(t)) {
        this.handlers.set(t, new Set())
      }
      this.handlers.get(t)!.add(handler)
    }
    
    // Set up PostgreSQL listener if needed
    this.setupListener(types)
  }
  
  // Set up PostgreSQL LISTEN
  private async setupListener(types: EventType[]) {
    const client = await pgPool.connect()
    
    client.on('notification', async (msg) => {
      if (msg.channel !== 'events') return
      
      const { type, eventId } = JSON.parse(msg.payload)
      if (!types.includes(type)) return
      
      // Fetch and process event
      const event = await db.query.events.findFirst({
        where: eq(events.id, eventId)
      })
      
      if (event) {
        const handlers = this.handlers.get(type) || new Set()
        for (const handler of handlers) {
          await handler.handle(event)
        }
      }
    })
    
    await client.query('LISTEN events')
  }
}
```

### 3. Event Handlers

#### Contact Stats Calculator
```typescript
class ContactStatsHandler implements EventHandler {
  name = 'contact-stats-calculator'
  sync = false
  
  async handle(event: Event) {
    switch (event.type) {
      case 'interaction.email_sent':
      case 'interaction.email_received':
        await this.updateEmailStats(event)
        break
      case 'contact.enriched':
        await this.updateEnrichmentStats(event)
        break
    }
  }
  
  private async updateEmailStats(event: Event) {
    const { contactId, direction } = event.data
    
    // Update counters
    const field = direction === 'outbound' 
      ? 'totalEmailsSent' 
      : 'totalEmailsReceived'
    
    await db.update(contactStats)
      .set({
        [field]: sql`${field} + 1`,
        lastInteractionAt: new Date()
      })
      .where(eq(contactStats.contactId, contactId))
    
    // Recalculate relationship score
    await jobQueue.enqueue({
      type: 'calculate-relationship-scores',
      data: { contactId }
    })
  }
}
```

#### AI Training Data Collector
```typescript
class AITrainingDataHandler implements EventHandler {
  name = 'ai-training-collector'
  sync = true // Process immediately for real-time learning
  
  async handle(event: Event) {
    // Collect events that are useful for AI training
    const trainingEvents = [
      'contact.created',
      'contact.enriched',
      'interaction.recorded',
      'ai.query_executed'
    ]
    
    if (!trainingEvents.includes(event.type)) return
    
    // Transform event into training data
    const trainingData = this.transformForTraining(event)
    
    // Store in vector database for RAG
    await vectorDB.upsert({
      id: event.id,
      vector: await this.generateEmbedding(trainingData),
      metadata: {
        type: event.type,
        userId: event.userId,
        timestamp: event.createdAt
      },
      content: trainingData
    })
  }
  
  private transformForTraining(event: Event): string {
    switch (event.type) {
      case 'contact.created':
        return `New contact added: ${event.data.name} from ${event.data.company}`
      case 'interaction.recorded':
        return `Email ${event.data.direction} with ${event.data.contactName} about "${event.data.subject}"`
      default:
        return JSON.stringify(event.data)
    }
  }
}
```

### 4. Event Sourcing Patterns

#### Event Replay
```typescript
class EventReplayer {
  async replayEvents(
    filter: EventFilter,
    handler: (event: Event) => Promise<void>
  ) {
    const events = await db.query.events.findMany({
      where: and(
        filter.userId ? eq(events.userId, filter.userId) : undefined,
        filter.type ? eq(events.type, filter.type) : undefined,
        filter.after ? gte(events.createdAt, filter.after) : undefined,
        filter.before ? lte(events.createdAt, filter.before) : undefined
      ),
      orderBy: asc(events.createdAt)
    })
    
    for (const event of events) {
      await handler(event)
    }
  }
  
  // Rebuild contact stats from events
  async rebuildContactStats(contactId: string) {
    const stats = {
      totalEmailsSent: 0,
      totalEmailsReceived: 0,
      interactions: []
    }
    
    await this.replayEvents(
      {
        type: ['interaction.email_sent', 'interaction.email_received'],
        aggregateId: contactId
      },
      async (event) => {
        if (event.data.direction === 'outbound') {
          stats.totalEmailsSent++
        } else {
          stats.totalEmailsReceived++
        }
        stats.interactions.push(event.data)
      }
    )
    
    // Calculate derived metrics
    const metrics = this.calculateMetrics(stats)
    
    // Update database
    await db.update(contactStats)
      .set(metrics)
      .where(eq(contactStats.contactId, contactId))
  }
}
```

#### Event Projections
```typescript
class EventProjector {
  // Project events into read models
  async projectUserActivity(userId: string): Promise<UserActivitySummary> {
    const summary = {
      totalEmails: 0,
      totalContacts: 0,
      totalInteractions: 0,
      recentActivity: []
    }
    
    await eventReplayer.replayEvents(
      { userId, after: daysAgo(30) },
      async (event) => {
        switch (event.type) {
          case 'email.synced':
            summary.totalEmails += event.data.count
            break
          case 'contact.created':
            summary.totalContacts++
            break
          case 'interaction.recorded':
            summary.totalInteractions++
            summary.recentActivity.push({
              type: event.type,
              timestamp: event.createdAt,
              description: this.describeEvent(event)
            })
            break
        }
      }
    )
    
    return summary
  }
}
```

## Integration with Background Jobs

### Job-Event Coordination
```typescript
// Jobs emit events
class JobEventEmitter {
  async onJobComplete(job: Job, result: any) {
    await eventBus.emit('job.completed', {
      jobId: job.id,
      type: job.type,
      duration: job.completedAt.getTime() - job.startedAt.getTime(),
      result
    })
  }
}

// Events trigger jobs
eventBus.on('contact.created', async (event) => {
  // Queue enrichment
  await jobQueue.enqueue({
    type: 'contact-enrichment',
    data: { contactId: event.data.id },
    options: {
      metadata: { causationId: event.id }
    }
  })
})

eventBus.on('email.synced', async (event) => {
  // Queue analytics calculation
  await jobQueue.enqueue({
    type: 'calculate-email-analytics',
    data: { userId: event.userId },
    options: { delay: 60000 } // 1 minute delay
  })
})
```

## Monitoring and Observability

```typescript
// Job metrics
interface JobMetrics {
  jobsEnqueued: Counter
  jobsCompleted: Counter
  jobsFailed: Counter
  jobDuration: Histogram
  queueDepth: Gauge
}

// Event metrics
interface EventMetrics {
  eventsEmitted: Counter
  eventProcessingTime: Histogram
  eventHandlerErrors: Counter
}

// Dashboard queries
const getDashboardStats = async () => {
  const [jobStats, eventStats] = await Promise.all([
    db.select({
      total: count(),
      pending: count(eq(backgroundJobs.status, 'pending')),
      failed: count(eq(backgroundJobs.status, 'failed'))
    }).from(backgroundJobs),
    
    db.select({
      total: count(),
      lastHour: count(gte(events.createdAt, hoursAgo(1))),
      byType: sql`json_object_agg(type, count)`
    }).from(events)
  ])
  
  return { jobs: jobStats, events: eventStats }
}
```