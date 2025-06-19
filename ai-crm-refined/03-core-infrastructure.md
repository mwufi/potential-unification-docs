# Core Infrastructure & Shared Components

## Overview

This document defines the foundational components that all teams will build upon. These components must be implemented first and form the backbone of the entire system.

## 1. Configuration Management

### Environment Configuration
```typescript
// src/config/env.ts
export const env = {
  // Core
  NODE_ENV: process.env.NODE_ENV || 'development',
  DATABASE_URL: process.env.DATABASE_URL!,
  REDIS_URL: process.env.REDIS_URL,
  
  // Auth
  NEXTAUTH_URL: process.env.NEXTAUTH_URL!,
  NEXTAUTH_SECRET: process.env.NEXTAUTH_SECRET!,
  
  // Google
  GOOGLE_CLIENT_ID: process.env.GOOGLE_CLIENT_ID!,
  GOOGLE_CLIENT_SECRET: process.env.GOOGLE_CLIENT_SECRET!,
  GOOGLE_WEBHOOK_URL: process.env.GOOGLE_WEBHOOK_URL,
  
  // AI
  OPENAI_API_KEY: process.env.OPENAI_API_KEY,
  ANTHROPIC_API_KEY: process.env.ANTHROPIC_API_KEY,
  
  // Storage
  S3_BUCKET: process.env.S3_BUCKET,
  S3_REGION: process.env.S3_REGION,
  
  // Feature Flags
  FEATURES: {
    AI_AGENTS: process.env.FEATURE_AI_AGENTS === 'true',
    ENRICHMENT: process.env.FEATURE_ENRICHMENT === 'true',
    WEBHOOKS: process.env.FEATURE_WEBHOOKS === 'true',
  }
}

// Validation
const validateEnv = () => {
  const required = ['DATABASE_URL', 'NEXTAUTH_URL', 'GOOGLE_CLIENT_ID']
  const missing = required.filter(key => !process.env[key])
  if (missing.length) {
    throw new Error(`Missing required env vars: ${missing.join(', ')}`)
  }
}
```

### Feature Flags System
```typescript
// src/lib/features.ts
export class FeatureFlags {
  static async isEnabled(
    feature: string, 
    userId?: string
  ): Promise<boolean> {
    // Check env first
    if (env.FEATURES[feature] === false) return false
    
    // Check user-specific flags
    if (userId) {
      const user = await db.query.users.findFirst({
        where: eq(users.id, userId)
      })
      return user?.features?.[feature] === true
    }
    
    return true
  }
  
  static async enableForUser(
    userId: string, 
    feature: string
  ): Promise<void> {
    await db.update(users)
      .set({ 
        features: sql`features || jsonb_build_object(${feature}, true)` 
      })
      .where(eq(users.id, userId))
  }
}
```

## 2. Database Layer

### Drizzle ORM Setup
```typescript
// src/db/index.ts
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'
import * as schema from './schema'

const client = postgres(env.DATABASE_URL, {
  max: 20,
  idle_timeout: 20,
  connect_timeout: 10,
})

export const db = drizzle(client, { 
  schema,
  logger: env.NODE_ENV === 'development'
})

// Type exports
export type DB = typeof db
export type NewUser = typeof schema.users.$inferInsert
export type User = typeof schema.users.$inferSelect
export type NewContact = typeof schema.contacts.$inferInsert
export type Contact = typeof schema.contacts.$inferSelect
// ... etc
```

### Connection Pooling
```typescript
// src/db/pool.ts
export class DatabasePool {
  private static pools = new Map<string, postgres.Sql>()
  
  static getConnection(name: string = 'default'): postgres.Sql {
    if (!this.pools.has(name)) {
      this.pools.set(name, postgres(env.DATABASE_URL, {
        max: name === 'analytics' ? 5 : 20,
        idle_timeout: 20,
        connect_timeout: 10,
      }))
    }
    return this.pools.get(name)!
  }
  
  static async closeAll() {
    for (const [name, pool] of this.pools) {
      await pool.end()
    }
  }
}
```

### Query Patterns
```typescript
// src/db/queries/base.ts
export abstract class BaseRepository<T> {
  constructor(
    protected db: DB,
    protected table: any
  ) {}
  
  async findById(id: string): Promise<T | null> {
    const [result] = await this.db
      .select()
      .from(this.table)
      .where(eq(this.table.id, id))
      .limit(1)
    return result || null
  }
  
  async create(data: Partial<T>): Promise<T> {
    const [result] = await this.db
      .insert(this.table)
      .values(data)
      .returning()
    return result
  }
  
  async update(id: string, data: Partial<T>): Promise<T> {
    const [result] = await this.db
      .update(this.table)
      .set(data)
      .where(eq(this.table.id, id))
      .returning()
    return result
  }
  
  async delete(id: string): Promise<boolean> {
    const result = await this.db
      .delete(this.table)
      .where(eq(this.table.id, id))
    return result.rowCount > 0
  }
}
```

## 3. Authentication System

### Better Auth Configuration
```typescript
// src/lib/auth.ts
import { betterAuth } from "better-auth"
import { drizzleAdapter } from "better-auth/adapters/drizzle"
import { db } from "@/db"

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: "pg",
    schema: {
      users: "users",
      sessions: "sessions",
      accounts: "accounts",
      verificationTokens: "verification_tokens"
    }
  }),
  
  emailAndPassword: {
    enabled: false // OAuth only for now
  },
  
  socialProviders: {
    google: {
      clientId: env.GOOGLE_CLIENT_ID,
      clientSecret: env.GOOGLE_CLIENT_SECRET,
      scopes: [
        "openid",
        "email", 
        "profile",
        "https://www.googleapis.com/auth/gmail.readonly",
        "https://www.googleapis.com/auth/calendar.readonly",
        "https://www.googleapis.com/auth/contacts.readonly"
      ]
    }
  },
  
  callbacks: {
    session: async ({ session, token }) => {
      // Add custom fields to session
      return {
        ...session,
        user: {
          ...session.user,
          id: token.sub
        }
      }
    },
    
    jwt: async ({ token, account }) => {
      // Store OAuth tokens
      if (account) {
        token.accessToken = account.access_token
        token.refreshToken = account.refresh_token
      }
      return token
    }
  }
})

// Auth helpers
export const getServerSession = async () => {
  const session = await auth.api.getSession({
    headers: headers()
  })
  return session
}

export const requireAuth = async () => {
  const session = await getServerSession()
  if (!session) {
    throw new Error("Unauthorized")
  }
  return session
}
```

### OAuth Token Management
```typescript
// src/lib/auth/tokens.ts
export class TokenManager {
  static async getValidToken(userId: string): Promise<string> {
    const account = await db.query.accounts.findFirst({
      where: eq(accounts.userId, userId)
    })
    
    if (!account) throw new Error("No account found")
    
    // Check if token is expired
    if (account.expiresAt && account.expiresAt < new Date()) {
      return this.refreshToken(account)
    }
    
    return decrypt(account.accessToken)
  }
  
  static async refreshToken(account: Account): Promise<string> {
    const response = await fetch('https://oauth2.googleapis.com/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        client_id: env.GOOGLE_CLIENT_ID,
        client_secret: env.GOOGLE_CLIENT_SECRET,
        refresh_token: decrypt(account.refreshToken),
        grant_type: 'refresh_token'
      })
    })
    
    const data = await response.json()
    
    // Update stored tokens
    await db.update(accounts)
      .set({
        accessToken: encrypt(data.access_token),
        expiresAt: new Date(Date.now() + data.expires_in * 1000)
      })
      .where(eq(accounts.id, account.id))
    
    return data.access_token
  }
}
```

## 4. API Layer (tRPC)

### tRPC Setup
```typescript
// src/server/api/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server'
import { type NextRequest } from 'next/server'
import superjson from 'superjson'
import { ZodError } from 'zod'
import { getServerSession } from '@/lib/auth'
import { db } from '@/db'

export const createTRPCContext = async (opts: { req: NextRequest }) => {
  const session = await getServerSession()
  
  return {
    db,
    session,
    req: opts.req,
  }
}

const t = initTRPC.context<typeof createTRPCContext>().create({
  transformer: superjson,
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError:
          error.cause instanceof ZodError ? error.cause.flatten() : null,
      },
    }
  },
})

// Middleware
const enforceUserIsAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.session || !ctx.session.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' })
  }
  
  return next({
    ctx: {
      session: { ...ctx.session, user: ctx.session.user },
    },
  })
})

// Export reusable router and procedures
export const createTRPCRouter = t.router
export const publicProcedure = t.procedure
export const protectedProcedure = t.procedure.use(enforceUserIsAuthed)
```

### Root Router
```typescript
// src/server/api/root.ts
import { createTRPCRouter } from './trpc'
import { userRouter } from './routers/user'
import { contactRouter } from './routers/contact'
import { emailRouter } from './routers/email'
import { aiRouter } from './routers/ai'

export const appRouter = createTRPCRouter({
  user: userRouter,
  contact: contactRouter,
  email: emailRouter,
  ai: aiRouter,
})

export type AppRouter = typeof appRouter
```

## 5. Background Job System

### pg-boss Setup
```typescript
// src/jobs/index.ts
import PgBoss from 'pg-boss'
import { env } from '@/config/env'

let boss: PgBoss | null = null

export async function getJobQueue(): Promise<PgBoss> {
  if (!boss) {
    boss = new PgBoss({
      connectionString: env.DATABASE_URL,
      max: 20,
      retentionDays: 30,
      monitorStateIntervalSeconds: 30,
      maintenanceIntervalSeconds: 120,
    })
    
    boss.on('error', error => console.error('pg-boss error:', error))
    await boss.start()
  }
  
  return boss
}

// Job types
export interface JobData {
  'email-sync': { userId: string; accountId: string; full?: boolean }
  'contact-extraction': { userId: string; emailId: string }
  'enrichment': { userId: string; contactId: string; sources?: string[] }
  'ai-analysis': { userId: string; type: string; data: any }
}

// Type-safe job creation
export async function createJob<T extends keyof JobData>(
  name: T,
  data: JobData[T],
  options?: PgBoss.SendOptions
): Promise<string> {
  const boss = await getJobQueue()
  return boss.send(name, data, options)
}
```

### Job Handlers
```typescript
// src/jobs/handlers/base.ts
export abstract class JobHandler<T> {
  abstract name: string
  
  async handle(job: PgBoss.Job<T>): Promise<void> {
    try {
      await this.process(job.data)
    } catch (error) {
      console.error(`Job ${this.name} failed:`, error)
      throw error // Let pg-boss handle retry
    }
  }
  
  abstract process(data: T): Promise<void>
  
  async register(boss: PgBoss): Promise<void> {
    await boss.work(
      this.name,
      { teamSize: 5, teamConcurrency: 2 },
      job => this.handle(job)
    )
  }
}
```

## 6. Caching Layer

### Redis Cache
```typescript
// src/lib/cache.ts
import { Redis } from '@upstash/redis'

const redis = new Redis({
  url: env.REDIS_URL,
  token: env.REDIS_TOKEN,
})

export class Cache {
  static async get<T>(key: string): Promise<T | null> {
    try {
      return await redis.get(key)
    } catch (error) {
      console.error('Cache get error:', error)
      return null
    }
  }
  
  static async set(
    key: string, 
    value: any, 
    ttl?: number
  ): Promise<void> {
    try {
      if (ttl) {
        await redis.setex(key, ttl, value)
      } else {
        await redis.set(key, value)
      }
    } catch (error) {
      console.error('Cache set error:', error)
    }
  }
  
  static async delete(key: string): Promise<void> {
    try {
      await redis.del(key)
    } catch (error) {
      console.error('Cache delete error:', error)
    }
  }
  
  static async invalidatePattern(pattern: string): Promise<void> {
    // For Upstash Redis, we need to fetch keys first
    const keys = await redis.keys(pattern)
    if (keys.length > 0) {
      await redis.del(...keys)
    }
  }
}

// Cache decorators
export function Cached(ttl: number = 3600) {
  return function (
    target: any,
    propertyName: string,
    descriptor: PropertyDescriptor
  ) {
    const method = descriptor.value
    
    descriptor.value = async function (...args: any[]) {
      const key = `${target.constructor.name}:${propertyName}:${JSON.stringify(args)}`
      
      const cached = await Cache.get(key)
      if (cached) return cached
      
      const result = await method.apply(this, args)
      await Cache.set(key, result, ttl)
      
      return result
    }
  }
}
```

## 7. Error Handling

### Custom Errors
```typescript
// src/lib/errors.ts
export class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 500,
    public isOperational: boolean = true
  ) {
    super(message)
    Object.setPrototypeOf(this, AppError.prototype)
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 'VALIDATION_ERROR', 400)
  }
}

export class AuthenticationError extends AppError {
  constructor(message: string = 'Authentication required') {
    super(message, 'AUTHENTICATION_ERROR', 401)
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 'NOT_FOUND', 404)
  }
}

export class RateLimitError extends AppError {
  constructor(retryAfter?: number) {
    super('Rate limit exceeded', 'RATE_LIMIT', 429)
    if (retryAfter) {
      this.retryAfter = retryAfter
    }
  }
  retryAfter?: number
}
```

### Global Error Handler
```typescript
// src/lib/errors/handler.ts
export const errorHandler = (error: unknown): Response => {
  // Log error
  console.error('Global error:', error)
  
  // Handle known errors
  if (error instanceof AppError) {
    return NextResponse.json(
      {
        error: {
          code: error.code,
          message: error.message,
          ...(error instanceof RateLimitError && 
            error.retryAfter && { retryAfter: error.retryAfter })
        }
      },
      { status: error.statusCode }
    )
  }
  
  // Handle Zod errors
  if (error instanceof ZodError) {
    return NextResponse.json(
      {
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Invalid request data',
          errors: error.errors
        }
      },
      { status: 400 }
    )
  }
  
  // Default error
  return NextResponse.json(
    {
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred'
      }
    },
    { status: 500 }
  )
}
```

## 8. Logging & Monitoring

### Structured Logging
```typescript
// src/lib/logger.ts
import pino from 'pino'

export const logger = pino({
  level: env.LOG_LEVEL || 'info',
  transport: env.NODE_ENV === 'development' 
    ? { target: 'pino-pretty' }
    : undefined,
  base: {
    env: env.NODE_ENV,
  },
  redact: ['password', 'token', 'accessToken', 'refreshToken'],
})

// Context logger
export const createLogger = (context: Record<string, any>) => {
  return logger.child(context)
}

// Request logger
export const requestLogger = (req: NextRequest) => {
  return createLogger({
    requestId: crypto.randomUUID(),
    url: req.url,
    method: req.method,
    userAgent: req.headers.get('user-agent'),
  })
}
```

### Performance Monitoring
```typescript
// src/lib/monitoring.ts
export class Monitor {
  private static timers = new Map<string, number>()
  
  static startTimer(label: string): void {
    this.timers.set(label, performance.now())
  }
  
  static endTimer(label: string): number {
    const start = this.timers.get(label)
    if (!start) return 0
    
    const duration = performance.now() - start
    this.timers.delete(label)
    
    logger.info({
      type: 'performance',
      label,
      duration,
    })
    
    return duration
  }
  
  static async trackAsync<T>(
    label: string,
    fn: () => Promise<T>
  ): Promise<T> {
    this.startTimer(label)
    try {
      return await fn()
    } finally {
      this.endTimer(label)
    }
  }
}
```

## 9. Testing Infrastructure

### Test Database
```typescript
// src/test/db.ts
import { PostgreSqlContainer } from '@testcontainers/postgresql'
import { drizzle } from 'drizzle-orm/postgres-js'
import { migrate } from 'drizzle-orm/postgres-js/migrator'
import postgres from 'postgres'

let container: PostgreSqlContainer
let testDb: ReturnType<typeof drizzle>

export async function setupTestDb() {
  container = await new PostgreSqlContainer()
    .withDatabase('test_db')
    .withUsername('test_user')
    .withPassword('test_pass')
    .start()
  
  const client = postgres(container.getConnectionUri())
  testDb = drizzle(client)
  
  // Run migrations
  await migrate(testDb, { migrationsFolder: './drizzle' })
  
  return testDb
}

export async function teardownTestDb() {
  await container.stop()
}

export { testDb }
```

### Test Utilities
```typescript
// src/test/utils.ts
export const createTestUser = async (
  overrides?: Partial<NewUser>
): Promise<User> => {
  return db.insert(users).values({
    email: `test-${Date.now()}@example.com`,
    name: 'Test User',
    ...overrides
  }).returning()[0]
}

export const createTestContact = async (
  userId: string,
  overrides?: Partial<NewContact>
): Promise<Contact> => {
  return db.insert(contacts).values({
    userId,
    email: `contact-${Date.now()}@example.com`,
    name: 'Test Contact',
    ...overrides
  }).returning()[0]
}

// Mock external services
export const mockGmailApi = () => {
  return {
    users: {
      messages: {
        list: vi.fn().mockResolvedValue({ messages: [] }),
        get: vi.fn().mockResolvedValue({ /* mock message */ }),
      }
    }
  }
}
```

## 10. Deployment Configuration

### Docker Setup
```dockerfile
# Dockerfile
FROM node:20-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Dependencies
FROM base AS deps
COPY package.json bun.lock* ./
RUN bun install --frozen-lockfile

# Builder
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN bun run build

# Runner
FROM base AS runner
ENV NODE_ENV production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

### Health Checks
```typescript
// src/app/api/health/route.ts
export async function GET() {
  try {
    // Check database
    await db.execute(sql`SELECT 1`)
    
    // Check Redis
    await Cache.get('health-check')
    
    // Check job queue
    const boss = await getJobQueue()
    const stats = await boss.getQueueSize()
    
    return NextResponse.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      checks: {
        database: 'ok',
        cache: 'ok',
        jobs: 'ok',
        queueSize: stats
      }
    })
  } catch (error) {
    return NextResponse.json(
      {
        status: 'unhealthy',
        error: error.message
      },
      { status: 503 }
    )
  }
}
```