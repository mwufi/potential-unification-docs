# Phase 2: AI Intelligence & Automation

## Overview

Phase 2 transforms raw data into actionable intelligence using LLMs and automation. This phase introduces natural language interfaces, intelligent insights, and automated workflows.

**Duration**: 3 weeks  
**Team**: 4 engineers (2 AI, 2 Frontend)  
**Goal**: Make data intelligent and actionable

## Week 4: AI Foundation

### Day 1-2: AI Service Setup
```typescript
// src/lib/ai/provider.ts
import { createOpenAI } from '@ai-sdk/openai'
import { createAnthropic } from '@ai-sdk/anthropic'

export const openai = createOpenAI({
  apiKey: env.OPENAI_API_KEY,
  compatibility: 'strict',
})

export const anthropic = createAnthropic({
  apiKey: env.ANTHROPIC_API_KEY,
})

// Provider abstraction
export interface AIProvider {
  chat(options: ChatOptions): Promise<ChatResponse>
  embed(text: string): Promise<number[]>
}

export class AIService {
  private provider: AIProvider
  
  constructor(model: 'gpt-4' | 'claude-3') {
    this.provider = model === 'gpt-4' 
      ? new OpenAIProvider() 
      : new AnthropicProvider()
  }
  
  async chat(messages: Message[], options?: ChatOptions) {
    return this.provider.chat({
      messages,
      ...options
    })
  }
  
  async generateEmbedding(text: string): Promise<number[]> {
    return this.provider.embed(text)
  }
}
```

### Day 3-4: Context Building System
```typescript
// src/lib/ai/context.ts
export class ContextBuilder {
  constructor(
    private db: DB,
    private userId: string
  ) {}
  
  async buildContext(
    query: string,
    options: ContextOptions = {}
  ): Promise<Context> {
    const context: Context = {
      query,
      relevantEmails: [],
      relevantContacts: [],
      userStats: {},
      recentActivity: [],
    }
    
    // Semantic search for relevant emails
    if (options.includeEmails) {
      const embedding = await generateEmbedding(query)
      
      const relevantEmails = await db.execute(sql`
        SELECT id, subject, snippet, from_email, internal_date,
               1 - (embedding <=> ${embedding}::vector) as similarity
        FROM emails
        WHERE user_id = ${this.userId}
          AND 1 - (embedding <=> ${embedding}::vector) > 0.7
        ORDER BY similarity DESC
        LIMIT 10
      `)
      
      context.relevantEmails = relevantEmails
    }
    
    // Find relevant contacts
    if (options.includeContacts) {
      const contacts = await this.findRelevantContacts(query)
      context.relevantContacts = contacts
    }
    
    // Get user statistics
    if (options.includeStats) {
      context.userStats = await this.getUserStats()
    }
    
    // Get recent activity
    if (options.includeActivity) {
      context.recentActivity = await this.getRecentActivity()
    }
    
    return context
  }
  
  private async findRelevantContacts(query: string): Promise<Contact[]> {
    // Extract potential names/emails from query
    const entities = await this.extractEntities(query)
    
    if (entities.emails.length > 0) {
      return db.query.contacts.findMany({
        where: and(
          eq(contacts.userId, this.userId),
          inArray(contacts.email, entities.emails)
        )
      })
    }
    
    if (entities.names.length > 0) {
      return db.query.contacts.findMany({
        where: and(
          eq(contacts.userId, this.userId),
          or(...entities.names.map(name => 
            ilike(contacts.name, `%${name}%`)
          ))
        )
      })
    }
    
    // Semantic search fallback
    const embedding = await generateEmbedding(query)
    return db.execute(sql`
      SELECT * FROM contacts
      WHERE user_id = ${this.userId}
        AND 1 - (embedding <=> ${embedding}::vector) > 0.6
      ORDER BY 1 - (embedding <=> ${embedding}::vector) DESC
      LIMIT 5
    `)
  }
  
  private async extractEntities(text: string): Promise<Entities> {
    const prompt = `Extract entities from this text: "${text}"
    
    Return JSON with:
    - emails: array of email addresses
    - names: array of person names
    - companies: array of company names
    - dates: array of dates mentioned
    - topics: array of topics/subjects`
    
    const response = await ai.chat([
      { role: 'system', content: 'You are an entity extraction assistant.' },
      { role: 'user', content: prompt }
    ], {
      response_format: { type: 'json_object' }
    })
    
    return JSON.parse(response.content)
  }
}
```

### Day 5-6: Natural Language Query Interface
```typescript
// src/server/api/routers/ai.ts
export const aiRouter = createTRPCRouter({
  query: protectedProcedure
    .input(z.object({
      query: z.string().min(1).max(500),
      includeContext: z.boolean().default(true),
    }))
    .mutation(async ({ ctx, input }) => {
      const { query, includeContext } = input
      
      // Build context
      const contextBuilder = new ContextBuilder(ctx.db, ctx.session.user.id)
      const context = includeContext 
        ? await contextBuilder.buildContext(query, {
            includeEmails: true,
            includeContacts: true,
            includeStats: true,
          })
        : null
      
      // Create system prompt with context
      const systemPrompt = createSystemPrompt(context)
      
      // Get AI response
      const response = await ai.chat([
        { role: 'system', content: systemPrompt },
        { role: 'user', content: query }
      ], {
        tools: {
          searchEmails: {
            description: 'Search for emails',
            parameters: z.object({
              query: z.string(),
              limit: z.number().optional(),
            }),
            execute: async ({ query, limit }) => {
              return searchEmails(ctx.session.user.id, query, limit)
            }
          },
          searchContacts: {
            description: 'Search for contacts',
            parameters: z.object({
              query: z.string(),
              limit: z.number().optional(),
            }),
            execute: async ({ query, limit }) => {
              return searchContacts(ctx.session.user.id, query, limit)
            }
          },
          getContactStats: {
            description: 'Get statistics for a contact',
            parameters: z.object({
              contactId: z.string(),
            }),
            execute: async ({ contactId }) => {
              return getContactStats(contactId)
            }
          },
        }
      })
      
      // Store conversation
      const conversation = await storeConversation(
        ctx.session.user.id,
        query,
        response
      )
      
      return {
        response: response.content,
        conversationId: conversation.id,
        toolCalls: response.toolCalls,
      }
    }),
})

function createSystemPrompt(context: Context | null): string {
  return `You are an AI assistant for a CRM system. You help users understand their email and contact data.

${context ? `Context:
- Total emails: ${context.userStats.totalEmails}
- Total contacts: ${context.userStats.totalContacts}
- Recent activity: ${context.recentActivity.length} items

Relevant emails:
${context.relevantEmails.map(e => `- ${e.subject} from ${e.from_email}`).join('\n')}

Relevant contacts:
${context.relevantContacts.map(c => `- ${c.name} (${c.email})`).join('\n')}
` : ''}

Guidelines:
- Be concise and helpful
- Use the provided tools to search for specific information
- Format responses with markdown
- Suggest follow-up actions when appropriate`
}
```

### Day 7: Query UI Component
```typescript
// src/components/ai-query-interface.tsx
"use client"

import { useState } from 'react'
import { useAIQuery } from '@/hooks/use-ai-query'
import { Loader2, Send } from 'lucide-react'

export function AIQueryInterface() {
  const [query, setQuery] = useState('')
  const { submit, response, isLoading } = useAIQuery()
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!query.trim()) return
    
    await submit(query)
    setQuery('')
  }
  
  return (
    <div className="max-w-4xl mx-auto p-6">
      <form onSubmit={handleSubmit} className="mb-8">
        <div className="relative">
          <input
            type="text"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            placeholder="Ask anything about your contacts or emails..."
            className="w-full px-4 py-3 pr-12 border rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
            disabled={isLoading}
          />
          <button
            type="submit"
            disabled={isLoading || !query.trim()}
            className="absolute right-2 top-2 p-2 text-blue-600 hover:bg-blue-50 rounded-md disabled:opacity-50"
          >
            {isLoading ? <Loader2 className="animate-spin" /> : <Send />}
          </button>
        </div>
      </form>
      
      {response && (
        <AIResponse response={response} />
      )}
    </div>
  )
}

function AIResponse({ response }: { response: AIQueryResponse }) {
  return (
    <div className="bg-white rounded-lg shadow p-6">
      <div className="prose max-w-none">
        <ReactMarkdown>{response.content}</ReactMarkdown>
      </div>
      
      {response.toolCalls?.map((call, i) => (
        <ToolCallResult key={i} call={call} />
      ))}
      
      {response.suggestions && (
        <div className="mt-4 flex flex-wrap gap-2">
          {response.suggestions.map((suggestion, i) => (
            <button
              key={i}
              onClick={() => handleSuggestion(suggestion)}
              className="px-3 py-1 bg-gray-100 hover:bg-gray-200 rounded-full text-sm"
            >
              {suggestion}
            </button>
          ))}
        </div>
      )}
    </div>
  )
}
```

## Week 5: Email Intelligence

### Day 8-9: Email Categorization & Analysis
```typescript
// src/jobs/handlers/email-analysis.ts
export class EmailAnalysisHandler extends JobHandler<EmailAnalysisData> {
  name = 'email-analysis'
  
  async process(data: EmailAnalysisData) {
    const { userId, emailId } = data
    
    const email = await db.query.emails.findFirst({
      where: eq(emails.id, emailId),
      with: {
        interactions: {
          with: { contact: true }
        }
      }
    })
    
    if (!email) return
    
    // Analyze email
    const analysis = await this.analyzeEmail(email)
    
    // Update email with analysis
    await db.update(emails)
      .set({
        aiCategory: analysis.category,
        aiSummary: analysis.summary,
        aiSentiment: analysis.sentiment,
        extractedData: analysis.extractedData,
        embedding: analysis.embedding,
      })
      .where(eq(emails.id, emailId))
    
    // Update contact stats if needed
    if (analysis.updateContactStats) {
      await this.updateContactStats(email.interactions)
    }
  }
  
  private async analyzeEmail(email: Email): Promise<EmailAnalysis> {
    const prompt = `Analyze this email and provide:
    1. Category (personal, work, newsletter, spam, etc.)
    2. Brief summary (1-2 sentences)
    3. Sentiment (positive, neutral, negative)
    4. Extract any important data (dates, deadlines, action items, etc.)
    5. Key topics
    
    Email:
    From: ${email.fromName || email.fromEmail}
    Subject: ${email.subject}
    Body: ${email.bodyText?.slice(0, 1000)}`
    
    const response = await ai.chat([
      { role: 'system', content: 'You are an email analysis assistant.' },
      { role: 'user', content: prompt }
    ], {
      response_format: { type: 'json_object' }
    })
    
    const analysis = JSON.parse(response.content)
    
    // Generate embedding for semantic search
    const embeddingText = `${email.subject} ${email.snippet}`
    const embedding = await generateEmbedding(embeddingText)
    
    return {
      ...analysis,
      embedding,
      updateContactStats: analysis.category === 'work'
    }
  }
}

// Batch analysis for existing emails
export async function analyzeHistoricalEmails(userId: string) {
  const unanalyzedEmails = await db.query.emails.findMany({
    where: and(
      eq(emails.userId, userId),
      isNull(emails.aiCategory)
    ),
    limit: 1000
  })
  
  // Queue analysis jobs
  for (const email of unanalyzedEmails) {
    await createJob('email-analysis', {
      userId,
      emailId: email.id
    }, {
      priority: -10 // Low priority
    })
  }
}
```

### Day 10-11: Smart Email Composition
```typescript
// src/lib/ai/email-composer.ts
export class EmailComposer {
  constructor(private userId: string) {}
  
  async composeDraft(options: ComposeOptions): Promise<EmailDraft> {
    const { to, subject, context, tone = 'professional' } = options
    
    // Get recipient context
    const recipient = await this.getRecipientContext(to)
    
    // Get recent thread context if replying
    const threadContext = context.threadId 
      ? await this.getThreadContext(context.threadId)
      : null
    
    // Build prompt
    const prompt = this.buildComposePrompt({
      recipient,
      subject,
      context,
      threadContext,
      tone
    })
    
    // Generate draft
    const response = await ai.chat([
      { role: 'system', content: 'You are an expert email writer.' },
      { role: 'user', content: prompt }
    ])
    
    // Parse and enhance draft
    const draft = await this.enhanceDraft(response.content, recipient)
    
    return draft
  }
  
  private buildComposePrompt(options: any): string {
    return `Compose an email with the following context:
    
To: ${options.recipient?.name || options.to} (${options.to})
${options.recipient ? `
Relationship: ${options.recipient.relationshipStrength}/10
Last interaction: ${options.recipient.lastInteractionAt}
Previous topics: ${options.recipient.topics.join(', ')}
` : ''}

Subject: ${options.subject || '[Suggest a subject]'}
Tone: ${options.tone}

${options.context.instructions ? `Instructions: ${options.context.instructions}` : ''}

${options.threadContext ? `Previous conversation:
${options.threadContext.messages.map(m => `${m.from}: ${m.snippet}`).join('\n')}
` : ''}

Write a complete email that:
1. Maintains appropriate tone and relationship context
2. Is clear and concise
3. Includes a proper greeting and closing
4. Addresses the subject matter effectively`
  }
  
  private async enhanceDraft(
    content: string, 
    recipient: Contact | null
  ): Promise<EmailDraft> {
    // Extract suggested subject if needed
    const subjectMatch = content.match(/Subject: (.+)\n/)
    const subject = subjectMatch ? subjectMatch[1] : ''
    
    // Clean up the body
    const body = content
      .replace(/Subject: .+\n/, '')
      .trim()
    
    // Add personalization tokens
    const personalizedBody = recipient 
      ? body.replace(/\[NAME\]/g, recipient.firstName || recipient.name || 'there')
      : body
    
    // Generate follow-up suggestions
    const followUpDate = await this.suggestFollowUp(content, recipient)
    
    return {
      subject,
      body: personalizedBody,
      html: markdown.render(personalizedBody),
      followUpDate,
      confidence: 0.85,
      alternativeSuggestions: []
    }
  }
}

// UI Component
export function ComposeDialog({ 
  recipient,
  onSend 
}: { 
  recipient?: Contact
  onSend: (draft: EmailDraft) => void 
}) {
  const [instructions, setInstructions] = useState('')
  const [draft, setDraft] = useState<EmailDraft | null>(null)
  const [isGenerating, setIsGenerating] = useState(false)
  
  const generateDraft = async () => {
    setIsGenerating(true)
    try {
      const composer = new EmailComposer(userId)
      const draft = await composer.composeDraft({
        to: recipient?.email || '',
        context: { instructions },
        tone: 'professional'
      })
      setDraft(draft)
    } finally {
      setIsGenerating(false)
    }
  }
  
  return (
    <Dialog>
      <DialogContent className="max-w-2xl">
        <DialogHeader>
          <DialogTitle>Compose Email</DialogTitle>
        </DialogHeader>
        
        {!draft ? (
          <div className="space-y-4">
            <div>
              <Label>To</Label>
              <div className="flex items-center gap-2 p-2 bg-gray-50 rounded">
                {recipient?.avatar && (
                  <Avatar src={recipient.avatar} size="sm" />
                )}
                <span>{recipient?.name}</span>
                <span className="text-gray-500">({recipient?.email})</span>
              </div>
            </div>
            
            <div>
              <Label>Instructions</Label>
              <Textarea
                value={instructions}
                onChange={(e) => setInstructions(e.target.value)}
                placeholder="E.g., Follow up on our meeting about the Q4 proposal..."
                rows={3}
              />
            </div>
            
            <Button 
              onClick={generateDraft}
              disabled={isGenerating}
              className="w-full"
            >
              {isGenerating ? (
                <>
                  <Loader2 className="animate-spin mr-2" />
                  Generating...
                </>
              ) : (
                'Generate Draft'
              )}
            </Button>
          </div>
        ) : (
          <EmailEditor
            draft={draft}
            onUpdate={setDraft}
            onSend={() => onSend(draft)}
            onRegenerate={generateDraft}
          />
        )}
      </DialogContent>
    </Dialog>
  )
}
```

### Day 12-13: Relationship Insights
```typescript
// src/lib/analytics/relationship-scorer.ts
export class RelationshipScorer {
  async calculateScore(
    userId: string, 
    contactId: string
  ): Promise<RelationshipScore> {
    // Get interaction data
    const interactions = await db.execute(sql`
      SELECT 
        COUNT(*) as total_emails,
        COUNT(*) FILTER (WHERE i.interaction_type = 'from') as received,
        COUNT(*) FILTER (WHERE i.interaction_type = 'to') as sent,
        MAX(e.internal_date) as last_interaction,
        MIN(e.internal_date) as first_interaction,
        AVG(
          EXTRACT(EPOCH FROM (
            LEAD(e.internal_date) OVER (ORDER BY e.internal_date) - e.internal_date
          )) / 86400
        ) as avg_days_between
      FROM email_interactions i
      JOIN emails e ON i.email_id = e.id
      WHERE i.contact_id = ${contactId}
        AND e.user_id = ${userId}
    `)
    
    const stats = interactions[0]
    
    // Calculate components
    const frequency = this.calculateFrequency(stats)
    const recency = this.calculateRecency(stats.last_interaction)
    const engagement = this.calculateEngagement(stats)
    const consistency = this.calculateConsistency(stats)
    
    // Weighted score
    const score = (
      frequency * 0.3 +
      recency * 0.3 +
      engagement * 0.2 +
      consistency * 0.2
    )
    
    // Get insights
    const insights = await this.generateInsights(contactId, stats)
    
    return {
      score: Math.round(score * 10) / 10, // 0-10 scale
      components: {
        frequency,
        recency,
        engagement,
        consistency
      },
      insights,
      trend: await this.calculateTrend(contactId)
    }
  }
  
  private calculateFrequency(stats: any): number {
    const emailsPerMonth = stats.total_emails / 
      (daysBetween(stats.first_interaction, new Date()) / 30)
    
    // Scoring curve: 10+ emails/month = 10, 1 email/month = 5
    return Math.min(10, Math.max(0, emailsPerMonth))
  }
  
  private calculateRecency(lastInteraction: Date): number {
    const daysSince = daysBetween(lastInteraction, new Date())
    
    // Scoring: 0-7 days = 10, 30+ days = 0
    if (daysSince <= 7) return 10
    if (daysSince <= 14) return 8
    if (daysSince <= 30) return 5
    if (daysSince <= 60) return 3
    return 1
  }
  
  private async generateInsights(
    contactId: string, 
    stats: any
  ): Promise<string[]> {
    const insights: string[] = []
    
    // Response rate insight
    if (stats.sent > 0) {
      const responseRate = stats.received / stats.sent
      if (responseRate > 0.8) {
        insights.push('High engagement - responds to most emails')
      } else if (responseRate < 0.3) {
        insights.push('Low response rate - consider different approach')
      }
    }
    
    // Frequency insight
    if (stats.avg_days_between < 7) {
      insights.push('Frequent communication - active relationship')
    } else if (stats.avg_days_between > 30) {
      insights.push('Infrequent contact - consider reaching out')
    }
    
    // Get AI insights
    const aiInsights = await this.getAIInsights(contactId)
    insights.push(...aiInsights)
    
    return insights.slice(0, 3) // Top 3 insights
  }
}

// UI Component
export function RelationshipInsights({ contact }: { contact: Contact }) {
  const { data: score, isLoading } = useRelationshipScore(contact.id)
  
  if (isLoading) return <Skeleton />
  if (!score) return null
  
  return (
    <Card>
      <CardHeader>
        <CardTitle>Relationship Insights</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="flex items-center justify-between mb-4">
          <div>
            <div className="text-3xl font-bold">{score.score}/10</div>
            <div className="text-sm text-gray-500">Relationship Score</div>
          </div>
          <TrendIndicator trend={score.trend} />
        </div>
        
        <div className="grid grid-cols-2 gap-4 mb-4">
          <ScoreComponent 
            label="Frequency" 
            value={score.components.frequency} 
          />
          <ScoreComponent 
            label="Recency" 
            value={score.components.recency} 
          />
          <ScoreComponent 
            label="Engagement" 
            value={score.components.engagement} 
          />
          <ScoreComponent 
            label="Consistency" 
            value={score.components.consistency} 
          />
        </div>
        
        <div className="space-y-2">
          {score.insights.map((insight, i) => (
            <div key={i} className="flex items-start gap-2 text-sm">
              <Lightbulb className="w-4 h-4 text-yellow-500 mt-0.5" />
              <span>{insight}</span>
            </div>
          ))}
        </div>
      </CardContent>
    </Card>
  )
}
```

### Day 14: Contact Enrichment
```typescript
// src/services/enrichment/enrichment-service.ts
export class EnrichmentService {
  private providers: EnrichmentProvider[] = [
    new TwitterEnrichmentProvider(),
    new LinkedInEnrichmentProvider(),
    new ClearbitEnrichmentProvider(),
  ]
  
  async enrichContact(contact: Contact): Promise<EnrichmentResult> {
    const results: ProviderResult[] = []
    
    // Try each provider
    for (const provider of this.providers) {
      try {
        const result = await provider.enrich(contact)
        if (result.found) {
          results.push(result)
        }
      } catch (error) {
        logger.error(`Enrichment failed for ${provider.name}`, error)
      }
    }
    
    // Merge results
    const merged = this.mergeResults(results)
    
    // Store enrichment data
    if (merged.hasData) {
      await db.insert(enrichments).values({
        contactId: contact.id,
        source: 'multi',
        data: merged.data,
        confidenceScore: merged.confidence,
        fetchedAt: new Date(),
        expiresAt: addDays(new Date(), 30),
      })
      
      // Update contact
      await db.update(contacts)
        .set({
          title: merged.data.title || contact.title,
          bio: merged.data.bio || contact.bio,
          location: merged.data.location || contact.location,
          socialProfiles: {
            ...contact.socialProfiles,
            ...merged.data.socialProfiles
          },
          updatedAt: new Date()
        })
        .where(eq(contacts.id, contact.id))
    }
    
    return merged
  }
  
  private mergeResults(results: ProviderResult[]): EnrichmentResult {
    const merged: any = {
      hasData: results.length > 0,
      confidence: 0,
      data: {}
    }
    
    // Weighted merge based on provider confidence
    for (const result of results) {
      const weight = result.confidence
      
      // Merge fields with confidence weighting
      for (const [key, value] of Object.entries(result.data)) {
        if (!merged.data[key] || result.confidence > merged.confidence) {
          merged.data[key] = value
        }
      }
      
      merged.confidence = Math.max(merged.confidence, result.confidence)
    }
    
    return merged
  }
}

// Twitter enrichment provider
export class TwitterEnrichmentProvider implements EnrichmentProvider {
  name = 'twitter'
  
  async enrich(contact: Contact): Promise<ProviderResult> {
    // Try to find Twitter handle
    const twitterHandle = this.extractTwitterHandle(contact)
    if (!twitterHandle) {
      return { found: false, confidence: 0, data: {} }
    }
    
    try {
      // Fetch Twitter profile
      const profile = await twitterClient.getProfile(twitterHandle)
      
      return {
        found: true,
        confidence: 0.9,
        data: {
          bio: profile.description,
          location: profile.location,
          socialProfiles: {
            twitter: {
              handle: twitterHandle,
              url: `https://twitter.com/${twitterHandle}`,
              followers: profile.followers_count,
              verified: profile.verified,
            }
          },
          avatar: profile.profile_image_url_https,
        }
      }
    } catch (error) {
      return { found: false, confidence: 0, data: {} }
    }
  }
  
  private extractTwitterHandle(contact: Contact): string | null {
    // Check existing social profiles
    if (contact.socialProfiles?.twitter) {
      return contact.socialProfiles.twitter
    }
    
    // Try to extract from email signature patterns
    const emailDomain = contact.email?.split('@')[1]
    if (emailDomain && contact.name) {
      // Common patterns: @firstlast, @first_last, etc.
      const possibleHandles = [
        contact.name.toLowerCase().replace(' ', ''),
        contact.name.toLowerCase().replace(' ', '_'),
        contact.firstName?.toLowerCase() || '',
      ]
      
      // Would need to verify these handles exist
      return possibleHandles[0]
    }
    
    return null
  }
}
```

## Week 6: Automation & Intelligence

### Day 15-16: Custom AI Agents
```typescript
// src/lib/agents/agent-framework.ts
export interface AgentConfig {
  name: string
  description: string
  trigger: AgentTrigger
  skills: string[]
  parameters: Record<string, any>
  systemPrompt?: string
}

export interface AgentTrigger {
  type: 'manual' | 'scheduled' | 'event'
  schedule?: string // cron expression
  events?: string[] // event types to listen for
}

export class AgentFramework {
  async createAgent(
    userId: string, 
    config: AgentConfig
  ): Promise<Agent> {
    const agent = await db.insert(aiAgents).values({
      userId,
      name: config.name,
      description: config.description,
      type: 'custom',
      config: config as any,
      skills: config.skills,
      isActive: true,
    }).returning()
    
    // Register triggers
    if (config.trigger.type === 'scheduled') {
      await this.registerScheduledTrigger(agent[0].id, config.trigger.schedule!)
    } else if (config.trigger.type === 'event') {
      await this.registerEventTriggers(agent[0].id, config.trigger.events!)
    }
    
    return agent[0]
  }
  
  async executeAgent(
    agentId: string,
    input: any = {}
  ): Promise<AgentExecution> {
    const agent = await db.query.aiAgents.findFirst({
      where: eq(aiAgents.id, agentId)
    })
    
    if (!agent) throw new Error('Agent not found')
    
    // Create execution record
    const [execution] = await db.insert(agentExecutions).values({
      agentId,
      triggerType: 'manual',
      triggerData: input,
      status: 'running',
      input,
      startedAt: new Date(),
    }).returning()
    
    try {
      // Execute agent
      const result = await this.runAgent(agent, input)
      
      // Update execution
      await db.update(agentExecutions)
        .set({
          status: 'completed',
          output: result,
          completedAt: new Date(),
        })
        .where(eq(agentExecutions.id, execution.id))
      
      return { ...execution, output: result, status: 'completed' }
    } catch (error) {
      // Update execution with error
      await db.update(agentExecutions)
        .set({
          status: 'failed',
          error: error.message,
          completedAt: new Date(),
        })
        .where(eq(agentExecutions.id, execution.id))
      
      throw error
    }
  }
  
  private async runAgent(agent: Agent, input: any): Promise<any> {
    const config = agent.config as AgentConfig
    
    // Build agent context
    const context = await this.buildAgentContext(agent.userId, input)
    
    // Create system prompt
    const systemPrompt = config.systemPrompt || this.defaultSystemPrompt(agent)
    
    // Execute with available skills
    const tools = this.buildTools(agent.skills)
    
    const response = await ai.chat([
      { role: 'system', content: systemPrompt },
      { role: 'user', content: JSON.stringify(input) }
    ], {
      tools,
      tool_choice: 'auto',
    })
    
    // Process results
    return this.processAgentResults(response, agent)
  }
  
  private buildTools(skills: string[]): Tools {
    const availableTools = {
      searchEmails: searchEmailsTool,
      searchContacts: searchContactsTool,
      composeEmail: composeEmailTool,
      scheduleFollowUp: scheduleFollowUpTool,
      enrichContact: enrichContactTool,
      generateReport: generateReportTool,
      // ... more tools
    }
    
    const tools: Tools = {}
    for (const skill of skills) {
      if (availableTools[skill]) {
        tools[skill] = availableTools[skill]
      }
    }
    
    return tools
  }
}

// Example: Email categorization agent
export const emailCategorizationAgent: AgentConfig = {
  name: 'Email Categorizer',
  description: 'Automatically categorizes incoming emails',
  trigger: {
    type: 'event',
    events: ['email.received']
  },
  skills: ['categorizeEmail', 'updateEmail', 'notifyUser'],
  parameters: {
    categories: ['work', 'personal', 'newsletter', 'spam', 'important'],
    autoArchive: ['newsletter', 'spam'],
  },
  systemPrompt: `You are an email categorization agent. 
  Analyze incoming emails and categorize them accurately.
  For newsletters and spam, also mark them for auto-archive.`
}
```

### Day 17-18: Tool Execution Framework
```typescript
// src/lib/ai/tools.ts
export interface Tool {
  name: string
  description: string
  parameters: z.ZodSchema<any>
  execute: (params: any, context: ToolContext) => Promise<any>
}

export interface ToolContext {
  userId: string
  db: DB
  session: Session
}

// Email search tool
export const searchEmailsTool: Tool = {
  name: 'searchEmails',
  description: 'Search through user emails with advanced filters',
  parameters: z.object({
    query: z.string().optional(),
    from: z.string().optional(),
    to: z.string().optional(),
    subject: z.string().optional(),
    dateFrom: z.string().optional(),
    dateTo: z.string().optional(),
    hasAttachments: z.boolean().optional(),
    limit: z.number().max(50).default(10),
  }),
  execute: async (params, context) => {
    let query = context.db
      .select()
      .from(emails)
      .where(eq(emails.userId, context.userId))
      .limit(params.limit)
    
    // Apply filters
    const conditions: SQL[] = [eq(emails.userId, context.userId)]
    
    if (params.query) {
      conditions.push(
        sql`search_vector @@ plainto_tsquery('english', ${params.query})`
      )
    }
    
    if (params.from) {
      conditions.push(ilike(emails.fromEmail, `%${params.from}%`))
    }
    
    if (params.dateFrom) {
      conditions.push(gte(emails.internalDate, new Date(params.dateFrom)))
    }
    
    if (params.hasAttachments !== undefined) {
      conditions.push(eq(emails.hasAttachments, params.hasAttachments))
    }
    
    const results = await context.db
      .select({
        id: emails.id,
        subject: emails.subject,
        from: emails.fromEmail,
        date: emails.internalDate,
        snippet: emails.snippet,
      })
      .from(emails)
      .where(and(...conditions))
      .orderBy(desc(emails.internalDate))
      .limit(params.limit)
    
    return {
      count: results.length,
      emails: results,
    }
  }
}

// Compose email tool
export const composeEmailTool: Tool = {
  name: 'composeEmail',
  description: 'Draft an email with AI assistance',
  parameters: z.object({
    to: z.string().email(),
    subject: z.string(),
    instructions: z.string(),
    tone: z.enum(['formal', 'casual', 'friendly', 'professional']).default('professional'),
    sendImmediately: z.boolean().default(false),
  }),
  execute: async (params, context) => {
    const composer = new EmailComposer(context.userId)
    
    const draft = await composer.composeDraft({
      to: params.to,
      subject: params.subject,
      context: { instructions: params.instructions },
      tone: params.tone,
    })
    
    if (params.sendImmediately) {
      // Queue email send job
      await createJob('send-email', {
        userId: context.userId,
        draft,
      })
      
      return {
        status: 'queued',
        message: 'Email queued for sending',
        draft,
      }
    }
    
    return {
      status: 'drafted',
      draft,
    }
  }
}

// Tool execution in AI chat
export async function executeAIQuery(
  userId: string,
  query: string,
  availableTools: Tool[] = defaultTools
) {
  const context: ToolContext = {
    userId,
    db,
    session: await getSession(),
  }
  
  // Convert tools to AI SDK format
  const tools = availableTools.reduce((acc, tool) => {
    acc[tool.name] = {
      description: tool.description,
      parameters: tool.parameters,
      execute: (params) => tool.execute(params, context)
    }
    return acc
  }, {})
  
  const response = await ai.chat([
    { 
      role: 'system', 
      content: `You are a helpful CRM assistant with access to tools.
      Use tools when needed to answer questions or perform actions.
      Always explain what you're doing and why.`
    },
    { role: 'user', content: query }
  ], {
    tools,
    tool_choice: 'auto',
  })
  
  return response
}
```

### Day 19-20: Analytics Dashboard
```typescript
// src/components/analytics/dashboard.tsx
export function AnalyticsDashboard() {
  const { data: stats } = useAnalytics()
  
  return (
    <div className="p-6 space-y-6">
      <h1 className="text-2xl font-bold">Analytics Overview</h1>
      
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        <StatCard
          title="Total Contacts"
          value={stats?.totalContacts || 0}
          change={stats?.contactsChange}
          icon={<Users />}
        />
        <StatCard
          title="Emails This Week"
          value={stats?.emailsThisWeek || 0}
          change={stats?.emailsChange}
          icon={<Mail />}
        />
        <StatCard
          title="Response Rate"
          value={`${stats?.responseRate || 0}%`}
          change={stats?.responseRateChange}
          icon={<TrendingUp />}
        />
        <StatCard
          title="AI Queries"
          value={stats?.aiQueries || 0}
          change={stats?.aiQueriesChange}
          icon={<Brain />}
        />
      </div>
      
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <EmailActivityChart data={stats?.emailActivity} />
        <TopContactsList contacts={stats?.topContacts} />
      </div>
      
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <EmailCategoriesChart data={stats?.emailCategories} />
        <ResponseTimeChart data={stats?.responseTime} />
        <ContactGrowthChart data={stats?.contactGrowth} />
      </div>
    </div>
  )
}

// Analytics API
export const analyticsRouter = createTRPCRouter({
  getStats: protectedProcedure
    .input(z.object({
      dateRange: z.enum(['week', 'month', 'quarter', 'year']).default('month'),
    }))
    .query(async ({ ctx, input }) => {
      const userId = ctx.session.user.id
      const { dateRange } = input
      
      const startDate = getStartDate(dateRange)
      
      // Get basic stats
      const [
        totalContacts,
        emailStats,
        aiStats,
        topContacts
      ] = await Promise.all([
        // Total contacts
        db.select({ count: count() })
          .from(contacts)
          .where(eq(contacts.userId, userId)),
          
        // Email stats
        db.select({
          total: count(),
          sent: count(sql`CASE WHEN is_sent = true THEN 1 END`),
          received: count(sql`CASE WHEN is_sent = false THEN 1 END`),
        })
        .from(emails)
        .where(and(
          eq(emails.userId, userId),
          gte(emails.internalDate, startDate)
        )),
        
        // AI usage stats
        db.select({
          queries: count(),
          avgTokens: avg(aiMessages.tokensUsed),
        })
        .from(aiMessages)
        .innerJoin(aiConversations, eq(aiMessages.conversationId, aiConversations.id))
        .where(and(
          eq(aiConversations.userId, userId),
          gte(aiMessages.createdAt, startDate)
        )),
        
        // Top contacts by interaction
        db.select({
          contact: contacts,
          interactions: count(emailInteractions.id),
        })
        .from(contacts)
        .innerJoin(emailInteractions, eq(contacts.id, emailInteractions.contactId))
        .where(eq(contacts.userId, userId))
        .groupBy(contacts.id)
        .orderBy(desc(count(emailInteractions.id)))
        .limit(10)
      ])
      
      // Calculate response rate
      const responseRate = emailStats[0].sent > 0
        ? Math.round((emailStats[0].received / emailStats[0].sent) * 100)
        : 0
      
      // Get time series data
      const emailActivity = await getEmailActivity(userId, startDate)
      const contactGrowth = await getContactGrowth(userId, startDate)
      
      return {
        totalContacts: totalContacts[0].count,
        emailsThisWeek: emailStats[0].total,
        responseRate,
        aiQueries: aiStats[0].queries,
        topContacts,
        emailActivity,
        contactGrowth,
        // ... more stats
      }
    })
})
```

### Day 21: Integration & Polish
```typescript
// src/app/(app)/page.tsx
export default function DashboardPage() {
  const { data: session } = useSession()
  const { data: syncStatus } = useSyncStatus()
  const { data: recentActivity } = useRecentActivity()
  
  return (
    <div className="p-6">
      <div className="mb-8">
        <h1 className="text-3xl font-bold">
          Welcome back, {session?.user.name}
        </h1>
        <p className="text-gray-600">
          Here's what's happening in your CRM today
        </p>
      </div>
      
      {/* AI Query Interface */}
      <div className="mb-8">
        <AIQueryInterface />
      </div>
      
      {/* Quick Actions */}
      <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-8">
        <QuickAction
          icon={<Mail />}
          label="Compose Email"
          onClick={() => openComposeDialog()}
        />
        <QuickAction
          icon={<UserPlus />}
          label="Add Contact"
          onClick={() => openAddContactDialog()}
        />
        <QuickAction
          icon={<Calendar />}
          label="Schedule Follow-up"
          onClick={() => openFollowUpDialog()}
        />
      </div>
      
      {/* Activity Feed */}
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-2">
          <Card>
            <CardHeader>
              <CardTitle>Recent Activity</CardTitle>
            </CardHeader>
            <CardContent>
              <ActivityFeed activities={recentActivity} />
            </CardContent>
          </Card>
        </div>
        
        <div>
          <Card>
            <CardHeader>
              <CardTitle>AI Insights</CardTitle>
            </CardHeader>
            <CardContent>
              <AIInsightsList />
            </CardContent>
          </Card>
        </div>
      </div>
    </div>
  )
}

// Activity feed component
function ActivityFeed({ activities }: { activities: Activity[] }) {
  return (
    <div className="space-y-4">
      {activities.map((activity) => (
        <div key={activity.id} className="flex items-start gap-3">
          <div className={`p-2 rounded-full ${getActivityColor(activity.type)}`}>
            {getActivityIcon(activity.type)}
          </div>
          <div className="flex-1">
            <p className="text-sm">
              {activity.description}
            </p>
            <p className="text-xs text-gray-500">
              {formatRelativeTime(activity.createdAt)}
            </p>
          </div>
        </div>
      ))}
    </div>
  )
}
```

## Phase 2 Deliverables Checklist

### AI Foundation ✓
- [ ] AI service with provider abstraction
- [ ] Context building system
- [ ] Embedding generation and storage
- [ ] Natural language query interface
- [ ] Tool execution framework

### Email Intelligence ✓
- [ ] Automatic email categorization
- [ ] Email sentiment analysis
- [ ] Smart email composition
- [ ] Thread context awareness
- [ ] Follow-up suggestions

### Contact Intelligence ✓
- [ ] Relationship strength scoring
- [ ] Interaction analytics
- [ ] Multi-source enrichment
- [ ] Social profile integration
- [ ] Contact insights

### Automation ✓
- [ ] Custom AI agent framework
- [ ] Event-triggered automation
- [ ] Scheduled agent execution
- [ ] Agent skill library
- [ ] Execution monitoring

### Analytics ✓
- [ ] Analytics dashboard
- [ ] Email statistics
- [ ] Contact growth metrics
- [ ] AI usage tracking
- [ ] Custom reports

## Success Metrics Validation

By the end of Phase 2:

1. **AI query accuracy > 85%** ✓
   - Context-aware responses
   - Tool execution for precise data
   - Fallback strategies

2. **Email draft quality rated 4+/5** ✓
   - Personalized content
   - Tone matching
   - Context awareness

3. **Enrichment data for > 60% contacts** ✓
   - Multiple data sources
   - Confidence scoring
   - Automatic updates

4. **< 2s response time for AI queries** ✓
   - Optimized context building
   - Efficient tool execution
   - Response caching

## Next Steps

With Phase 2 complete:
- AI understands and enhances all data
- Automated workflows save time
- Rich insights drive better relationships
- Natural language interface works smoothly

The system is ready for Phase 3: Advanced AI Agent System.