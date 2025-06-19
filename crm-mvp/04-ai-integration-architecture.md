# AI Integration Architecture

## Overview

This document outlines the AI layer that powers intelligent features in our CRM: natural language queries, contact insights, relationship analysis, and predictive suggestions.

## AI System Architecture

### 1. Core AI Components

```typescript
// AI Service Architecture
interface AIService {
  // Natural language interface
  chat: AIChatService
  
  // Structured queries
  query: AIQueryService
  
  // Analytics & insights
  insights: AIInsightsService
  
  // Suggestions & automation
  suggestions: AISuggestionService
  
  // Email assistance
  email: AIEmailService
}
```

### 2. AI Chat Service (Natural Language Interface)

```typescript
class AIChatService {
  private llm: LLMProvider
  private contextBuilder: ContextBuilder
  private functionRegistry: FunctionRegistry
  
  async process(message: string, userId: string): Promise<AIResponse> {
    // 1. Build context
    const context = await this.contextBuilder.build(userId, message)
    
    // 2. Determine intent and extract parameters
    const analysis = await this.analyzeIntent(message, context)
    
    // 3. Execute appropriate function
    if (analysis.requiresFunction) {
      return await this.executeFunctions(analysis, context)
    }
    
    // 4. Generate conversational response
    return await this.generateResponse(message, context, analysis)
  }
  
  private async analyzeIntent(message: string, context: UserContext) {
    const prompt = `
      Analyze this user query about their CRM data:
      Query: "${message}"
      
      Available context:
      - Total contacts: ${context.stats.totalContacts}
      - Recent interactions: ${context.stats.recentInteractions}
      - Email accounts: ${context.accounts.map(a => a.email).join(', ')}
      
      Determine:
      1. Intent (one of: query_contacts, analyze_relationship, get_insights, send_email, general_question)
      2. Entities (contacts, time ranges, metrics)
      3. Required functions to call
      4. Any clarifications needed
      
      Respond in JSON format.
    `
    
    const response = await this.llm.complete(prompt, { 
      temperature: 0.1,
      responseFormat: 'json' 
    })
    
    return JSON.parse(response)
  }
}
```

### 3. Function Registry (Tool Use)

```typescript
class FunctionRegistry {
  private functions: Map<string, AIFunction> = new Map()
  
  constructor() {
    // Register all available functions
    this.register('search_contacts', new SearchContactsFunction())
    this.register('analyze_relationship', new AnalyzeRelationshipFunction())
    this.register('get_email_stats', new GetEmailStatsFunction())
    this.register('find_top_contacts', new FindTopContactsFunction())
    this.register('generate_email_draft', new GenerateEmailDraftFunction())
    this.register('schedule_followup', new ScheduleFollowupFunction())
  }
  
  async execute(name: string, params: any, context: UserContext) {
    const fn = this.functions.get(name)
    if (!fn) throw new Error(`Unknown function: ${name}`)
    
    return await fn.execute(params, context)
  }
}

// Example function implementation
class FindTopContactsFunction implements AIFunction {
  schema = {
    name: 'find_top_contacts',
    description: 'Find top contacts by various metrics',
    parameters: {
      metric: {
        type: 'string',
        enum: ['email_frequency', 'recent_activity', 'response_rate'],
        description: 'Metric to sort by'
      },
      limit: {
        type: 'number',
        default: 10,
        description: 'Number of contacts to return'
      },
      timeRange: {
        type: 'string',
        enum: ['week', 'month', 'quarter', 'year', 'all_time'],
        default: 'month'
      }
    }
  }
  
  async execute(params: any, context: UserContext) {
    const { metric, limit, timeRange } = params
    
    // Build query based on metric
    let query = db
      .select({
        contact: contacts,
        stats: contactStats,
        score: sql<number>`0`
      })
      .from(contacts)
      .leftJoin(contactStats, eq(contacts.id, contactStats.contactId))
      .where(eq(contacts.userId, context.userId))
    
    // Apply metric-specific sorting
    switch (metric) {
      case 'email_frequency':
        query = query.orderBy(
          desc(sql`${contactStats.totalEmailsSent} + ${contactStats.totalEmailsReceived}`)
        )
        break
      case 'recent_activity':
        query = query.orderBy(desc(contactStats.lastInteractionAt))
        break
      case 'response_rate':
        query = query.orderBy(desc(contactStats.responseRate))
        break
    }
    
    const results = await query.limit(limit)
    
    return {
      contacts: results,
      metric,
      timeRange,
      generated: new Date()
    }
  }
}
```

### 4. AI Query Service (Structured Queries)

```typescript
class AIQueryService {
  private queryParser: QueryParser
  private queryExecutor: QueryExecutor
  
  async query(input: string, userId: string): Promise<QueryResult> {
    // Parse natural language into structured query
    const structuredQuery = await this.queryParser.parse(input)
    
    // Execute query
    const results = await this.queryExecutor.execute(structuredQuery, userId)
    
    // Format results
    return this.formatResults(results, structuredQuery)
  }
}

class QueryParser {
  async parse(input: string): Promise<StructuredQuery> {
    const prompt = `
      Convert this natural language query into a structured database query:
      "${input}"
      
      Available fields:
      - contacts: name, email, company, title, tags
      - stats: totalEmails, lastInteraction, responseRate
      - timeRanges: today, week, month, quarter, year
      
      Output JSON with:
      - entity: 'contacts' | 'interactions' | 'emails'
      - filters: array of conditions
      - sort: field and direction
      - limit: number
    `
    
    const response = await llm.complete(prompt, { 
      temperature: 0,
      responseFormat: 'json'
    })
    
    return JSON.parse(response)
  }
}
```

### 5. AI Insights Service

```typescript
class AIInsightsService {
  async generateContactInsights(contactId: string): Promise<ContactInsights> {
    // Gather all data about contact
    const [contact, stats, interactions, enrichments] = await Promise.all([
      db.query.contacts.findFirst({ where: eq(contacts.id, contactId) }),
      db.query.contactStats.findFirst({ where: eq(contactStats.contactId, contactId) }),
      this.getRecentInteractions(contactId),
      this.getEnrichments(contactId)
    ])
    
    // Generate insights
    const insights = await this.llm.complete({
      messages: [{
        role: 'system',
        content: 'You are a CRM analyst. Generate actionable insights about this contact.'
      }, {
        role: 'user',
        content: `
          Contact: ${contact.name} (${contact.email})
          Company: ${contact.company}
          
          Stats:
          - Total emails: ${stats.totalEmailsSent + stats.totalEmailsReceived}
          - Response rate: ${stats.responseRate}%
          - Last interaction: ${stats.lastInteractionAt}
          
          Recent interactions:
          ${interactions.map(i => `- ${i.type}: ${i.metadata.subject}`).join('\n')}
          
          Social presence:
          ${enrichments.twitter ? `- Twitter: ${enrichments.twitter.followerCount} followers` : ''}
          
          Generate:
          1. Relationship summary (2-3 sentences)
          2. Key insights (3-5 bullet points)
          3. Suggested next actions (2-3 recommendations)
          4. Conversation starters based on their interests
        `
      }],
      temperature: 0.7
    })
    
    return this.parseInsights(insights)
  }
  
  async generateWeeklyReport(userId: string): Promise<WeeklyReport> {
    const weekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000)
    
    // Gather weekly data
    const [
      newContacts,
      totalInteractions,
      topContacts,
      emailStats
    ] = await Promise.all([
      this.getNewContacts(userId, weekAgo),
      this.getInteractionCount(userId, weekAgo),
      this.getTopContactsByActivity(userId, weekAgo),
      this.getEmailStats(userId, weekAgo)
    ])
    
    // Generate report
    const report = await this.llm.complete({
      messages: [{
        role: 'system',
        content: 'You are a CRM analyst creating a weekly summary report.'
      }, {
        role: 'user',
        content: `
          Create a weekly CRM report with these stats:
          
          New contacts: ${newContacts.length}
          Total interactions: ${totalInteractions}
          Emails sent: ${emailStats.sent}
          Emails received: ${emailStats.received}
          
          Top contacts by activity:
          ${topContacts.map(c => `- ${c.name}: ${c.interactionCount} interactions`).join('\n')}
          
          Generate:
          1. Executive summary (2-3 sentences)
          2. Key metrics and trends
          3. Notable relationships to nurture
          4. Recommended actions for the week ahead
        `
      }],
      temperature: 0.6
    })
    
    return {
      period: { start: weekAgo, end: new Date() },
      summary: report,
      metrics: { newContacts, totalInteractions, emailStats },
      recommendations: this.extractRecommendations(report)
    }
  }
}
```

### 6. AI Suggestion Service

```typescript
class AISuggestionService {
  async getSuggestions(userId: string): Promise<Suggestion[]> {
    const suggestions: Suggestion[] = []
    
    // Check for various suggestion triggers
    await Promise.all([
      this.checkStaleRelationships(userId, suggestions),
      this.checkMissingInfo(userId, suggestions),
      this.checkFollowUpNeeded(userId, suggestions),
      this.checkEngagementOpportunities(userId, suggestions)
    ])
    
    // Rank suggestions by priority
    return this.rankSuggestions(suggestions)
  }
  
  private async checkStaleRelationships(userId: string, suggestions: Suggestion[]) {
    // Find contacts with no interaction in 30+ days
    const staleContacts = await db.query.contactStats.findMany({
      where: and(
        eq(contacts.userId, userId),
        lt(contactStats.lastInteractionAt, daysAgo(30)),
        gt(contactStats.relationshipStrength, 50) // Only important relationships
      ),
      limit: 5
    })
    
    for (const contact of staleContacts) {
      suggestions.push({
        type: 'reconnect',
        priority: 'medium',
        title: `Reconnect with ${contact.name}`,
        description: `You haven't interacted with ${contact.name} in ${daysSince(contact.lastInteractionAt)} days`,
        action: {
          type: 'compose_email',
          params: { contactId: contact.id, template: 'reconnect' }
        }
      })
    }
  }
  
  private async checkFollowUpNeeded(userId: string, suggestions: Suggestion[]) {
    // Find unanswered emails
    const unansweredEmails = await db.execute(sql`
      SELECT DISTINCT ON (e.thread_id) 
        e.*, 
        c.name as contact_name,
        c.id as contact_id
      FROM emails e
      JOIN interactions i ON i.email_id = e.id
      JOIN contacts c ON c.id = i.contact_id
      WHERE e.user_id = ${userId}
        AND i.direction = 'inbound'
        AND e.sent_at > NOW() - INTERVAL '7 days'
        AND NOT EXISTS (
          SELECT 1 FROM interactions i2
          WHERE i2.contact_id = i.contact_id
            AND i2.direction = 'outbound'
            AND i2.occurred_at > i.occurred_at
        )
      ORDER BY e.thread_id, e.sent_at DESC
    `)
    
    for (const email of unansweredEmails.slice(0, 3)) {
      suggestions.push({
        type: 'follow_up',
        priority: 'high',
        title: `Follow up with ${email.contact_name}`,
        description: `Unanswered email: "${email.subject}"`,
        action: {
          type: 'compose_reply',
          params: { emailId: email.id, threadId: email.thread_id }
        }
      })
    }
  }
}
```

### 7. AI Email Service

```typescript
class AIEmailService {
  async generateDraft(params: EmailDraftParams): Promise<EmailDraft> {
    const { contactId, purpose, context } = params
    
    // Get contact information
    const contact = await this.getContactWithHistory(contactId)
    
    // Build prompt based on purpose
    const prompt = this.buildEmailPrompt(purpose, contact, context)
    
    // Generate email
    const draft = await this.llm.complete({
      messages: [{
        role: 'system',
        content: `You are drafting professional emails for a CRM user. 
                  Match the tone of previous conversations.
                  Be concise and actionable.`
      }, {
        role: 'user',
        content: prompt
      }],
      temperature: 0.7
    })
    
    return {
      subject: this.extractSubject(draft),
      body: this.extractBody(draft),
      suggestedSendTime: this.calculateOptimalSendTime(contact)
    }
  }
  
  private buildEmailPrompt(purpose: EmailPurpose, contact: Contact, context: any) {
    const prompts = {
      followUp: `
        Draft a follow-up email to ${contact.name} at ${contact.company}.
        Last interaction: ${contact.lastInteraction.subject}
        Context: ${context.notes}
        
        Keep it brief and reference our last conversation.
      `,
      
      introduction: `
        Draft an introduction email to ${contact.name} at ${contact.company}.
        Their role: ${contact.title}
        Connection context: ${context.via}
        
        Be warm but professional.
      `,
      
      meeting: `
        Draft an email to schedule a meeting with ${contact.name}.
        Purpose: ${context.purpose}
        Duration: ${context.duration}
        
        Suggest 2-3 time slots and be flexible.
      `
    }
    
    return prompts[purpose] || prompts.followUp
  }
  
  async enhanceEmail(draft: string): Promise<EnhancedEmail> {
    const analysis = await this.llm.complete({
      messages: [{
        role: 'system',
        content: 'You are an email writing coach. Analyze and enhance this email draft.'
      }, {
        role: 'user',
        content: `
          Analyze this email draft and provide:
          1. Tone analysis
          2. Clarity score (1-10)
          3. Suggested improvements
          4. Alternative phrasings for key sentences
          
          Draft:
          ${draft}
        `
      }],
      temperature: 0.3,
      responseFormat: 'json'
    })
    
    return {
      original: draft,
      enhanced: this.applyEnhancements(draft, analysis.suggestions),
      analysis: {
        tone: analysis.tone,
        clarity: analysis.clarity,
        suggestions: analysis.suggestions
      }
    }
  }
}
```

### 8. Context Building & RAG

```typescript
class ContextBuilder {
  private vectorStore: VectorStore
  private embedder: Embedder
  
  async build(userId: string, query: string): Promise<UserContext> {
    // Get user's basic stats
    const stats = await this.getUserStats(userId)
    
    // Find relevant context using RAG
    const relevantContext = await this.findRelevantContext(userId, query)
    
    // Get recent activity
    const recentActivity = await this.getRecentActivity(userId)
    
    return {
      userId,
      stats,
      relevantContext,
      recentActivity,
      accounts: await this.getUserAccounts(userId)
    }
  }
  
  private async findRelevantContext(userId: string, query: string) {
    // Generate embedding for query
    const queryEmbedding = await this.embedder.embed(query)
    
    // Search vector store
    const results = await this.vectorStore.search({
      vector: queryEmbedding,
      filter: { userId },
      limit: 10
    })
    
    // Group by type
    return {
      contacts: results.filter(r => r.metadata.type === 'contact'),
      interactions: results.filter(r => r.metadata.type === 'interaction'),
      insights: results.filter(r => r.metadata.type === 'insight')
    }
  }
}

class VectorStore {
  async upsert(doc: VectorDocument) {
    // Store in PostgreSQL with pgvector
    await db.insert(vectorDocuments).values({
      id: doc.id,
      userId: doc.metadata.userId,
      vector: doc.vector,
      content: doc.content,
      metadata: doc.metadata,
      createdAt: new Date()
    })
  }
  
  async search(params: SearchParams): Promise<VectorDocument[]> {
    const results = await db.execute(sql`
      SELECT *,
        1 - (vector <=> ${params.vector}) as similarity
      FROM vector_documents
      WHERE user_id = ${params.filter.userId}
      ORDER BY vector <=> ${params.vector}
      LIMIT ${params.limit}
    `)
    
    return results.rows
  }
}
```

### 9. Prompt Templates & Few-Shot Examples

```typescript
class PromptLibrary {
  private templates = {
    contactAnalysis: `
      Analyze this contact's engagement pattern:
      
      Contact: {name} ({email})
      Company: {company}
      Role: {title}
      
      Interaction History:
      - Total emails: {totalEmails}
      - Sent: {emailsSent}, Received: {emailsReceived}
      - Average response time: {avgResponseTime}
      - Last interaction: {lastInteraction}
      
      Recent email subjects:
      {recentSubjects}
      
      Provide:
      1. Engagement level (low/medium/high)
      2. Relationship stage (new/developing/established/at-risk)
      3. Communication style insights
      4. Topics of interest
      5. Best time to reach out
    `,
    
    queryInterpretation: `
      User query: "{query}"
      
      Interpret this as a CRM database query.
      
      Examples:
      - "top clients" → contacts ordered by email frequency, limit 10
      - "people from Google" → contacts where company contains 'Google'
      - "who haven't I talked to recently" → contacts with no interaction in 30 days
      
      Available fields: {availableFields}
      
      Return a structured query in JSON format.
    `
  }
  
  get(name: string, vars: Record<string, any>): string {
    let template = this.templates[name]
    
    // Replace variables
    for (const [key, value] of Object.entries(vars)) {
      template = template.replace(`{${key}}`, String(value))
    }
    
    return template
  }
}
```

## AI Model Configuration

```typescript
interface AIConfig {
  // Primary LLM for complex reasoning
  primary: {
    provider: 'anthropic' | 'openai'
    model: 'claude-3-opus' | 'gpt-4-turbo'
    temperature: 0.7
    maxTokens: 4096
  }
  
  // Fast LLM for simple tasks
  fast: {
    provider: 'anthropic' | 'openai'
    model: 'claude-3-haiku' | 'gpt-3.5-turbo'
    temperature: 0.3
    maxTokens: 1024
  }
  
  // Embedding model
  embeddings: {
    provider: 'openai' | 'cohere'
    model: 'text-embedding-3-small'
    dimensions: 1536
  }
  
  // Rate limiting
  rateLimits: {
    requestsPerMinute: 60
    tokensPerMinute: 90000
  }
}
```

## Security & Privacy

```typescript
class AISecurityLayer {
  // Sanitize data before sending to LLM
  async sanitize(data: any): Promise<any> {
    return {
      ...data,
      email: this.hashEmail(data.email),
      phone: data.phone ? 'REDACTED' : undefined,
      // Remove any sensitive fields
      accessToken: undefined,
      refreshToken: undefined
    }
  }
  
  // Validate AI responses
  validateResponse(response: any): boolean {
    // Check for PII leakage
    if (this.containsPII(response)) {
      logger.warn('AI response contains PII')
      return false
    }
    
    // Check for prompt injection attempts
    if (this.detectInjection(response)) {
      logger.error('Possible prompt injection detected')
      return false
    }
    
    return true
  }
}
```

## Performance Optimization

```typescript
class AICache {
  // Cache common queries
  private queryCache = new LRUCache({
    max: 1000,
    ttl: 1000 * 60 * 60 // 1 hour
  })
  
  // Cache embeddings
  private embeddingCache = new LRUCache({
    max: 10000,
    ttl: 1000 * 60 * 60 * 24 * 7 // 1 week
  })
  
  async getCachedOrGenerate<T>(
    key: string,
    generator: () => Promise<T>
  ): Promise<T> {
    const cached = this.queryCache.get(key)
    if (cached) return cached as T
    
    const result = await generator()
    this.queryCache.set(key, result)
    return result
  }
}
```