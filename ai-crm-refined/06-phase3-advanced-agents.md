# Phase 3: Advanced AI Agent System

## Overview

Phase 3 creates a proactive, always-on AI assistant with specialized skills, persistent memory, and multi-agent coordination. This transforms the CRM from reactive to proactive.

**Duration**: 2 weeks  
**Team**: 4 engineers (2 AI, 1 Backend, 1 Frontend)  
**Goal**: Build intelligent agents that work autonomously

## Week 7: Agent Infrastructure

### Day 1-2: Skill Library System
```typescript
// src/lib/agents/skills/base.ts
export interface Skill {
  id: string
  name: string
  description: string
  category: SkillCategory
  requiredTools: string[]
  parameters: z.ZodSchema<any>
  examples: SkillExample[]
  execute: (params: any, context: SkillContext) => Promise<SkillResult>
}

export interface SkillContext {
  agent: Agent
  userId: string
  memory: AgentMemory
  tools: Record<string, Tool>
  emit: (event: string, data: any) => void
}

export interface SkillResult {
  success: boolean
  output: any
  sideEffects?: SideEffect[]
  suggestedNextSteps?: string[]
}

// Skill registry
export class SkillRegistry {
  private skills = new Map<string, Skill>()
  
  register(skill: Skill) {
    this.skills.set(skill.id, skill)
  }
  
  get(skillId: string): Skill | undefined {
    return this.skills.get(skillId)
  }
  
  getByCategory(category: SkillCategory): Skill[] {
    return Array.from(this.skills.values())
      .filter(skill => skill.category === category)
  }
  
  search(query: string): Skill[] {
    const lowercaseQuery = query.toLowerCase()
    return Array.from(this.skills.values())
      .filter(skill => 
        skill.name.toLowerCase().includes(lowercaseQuery) ||
        skill.description.toLowerCase().includes(lowercaseQuery)
      )
  }
  
  async validateSkill(skillId: string, params: any): Promise<boolean> {
    const skill = this.get(skillId)
    if (!skill) return false
    
    try {
      skill.parameters.parse(params)
      return true
    } catch {
      return false
    }
  }
}

// Example skill: Research Contact
export const researchContactSkill: Skill = {
  id: 'research-contact',
  name: 'Research Contact',
  description: 'Gather comprehensive information about a contact from multiple sources',
  category: 'enrichment',
  requiredTools: ['searchWeb', 'enrichContact', 'searchEmails'],
  parameters: z.object({
    contactId: z.string(),
    depth: z.enum(['basic', 'detailed', 'comprehensive']).default('detailed'),
    sources: z.array(z.string()).optional(),
  }),
  examples: [
    {
      input: { contactId: '123', depth: 'detailed' },
      output: { 
        profile: { /* enriched data */ },
        insights: ['Recently changed jobs', 'Active on Twitter'],
        recommendations: ['Reach out about new role']
      }
    }
  ],
  execute: async (params, context) => {
    const { contactId, depth, sources = ['web', 'social', 'emails'] } = params
    
    // Get contact
    const contact = await context.tools.getContact({ id: contactId })
    if (!contact) {
      return { success: false, output: { error: 'Contact not found' } }
    }
    
    // Parallel research
    const researchTasks = []
    
    if (sources.includes('web')) {
      researchTasks.push(
        context.tools.searchWeb({
          query: `"${contact.name}" ${contact.organization || ''} ${contact.email.split('@')[1]}`,
          limit: 10
        })
      )
    }
    
    if (sources.includes('social')) {
      researchTasks.push(
        context.tools.enrichContact({
          contactId,
          providers: ['twitter', 'linkedin']
        })
      )
    }
    
    if (sources.includes('emails')) {
      researchTasks.push(
        context.tools.searchEmails({
          from: contact.email,
          limit: 50
        })
      )
    }
    
    const [webResults, socialData, emailHistory] = await Promise.all(researchTasks)
    
    // Analyze findings
    const analysis = await analyzeResearchData({
      contact,
      webResults,
      socialData,
      emailHistory,
      depth
    })
    
    // Store in agent memory
    await context.memory.remember('research', {
      contactId,
      timestamp: new Date(),
      findings: analysis.keyFindings
    })
    
    return {
      success: true,
      output: analysis,
      sideEffects: [
        { type: 'contact.updated', data: { contactId, enrichedData: analysis.profile } }
      ],
      suggestedNextSteps: analysis.recommendations
    }
  }
}
```

### Day 3-4: Agent Memory System
```typescript
// src/lib/agents/memory.ts
export class AgentMemory {
  constructor(
    private agentId: string,
    private db: DB
  ) {}
  
  async remember(key: string, value: any, ttl?: number): Promise<void> {
    const agent = await this.db.query.aiAgents.findFirst({
      where: eq(aiAgents.id, this.agentId)
    })
    
    if (!agent) throw new Error('Agent not found')
    
    const memory = agent.memory || {}
    memory[key] = {
      value,
      timestamp: new Date(),
      expiresAt: ttl ? new Date(Date.now() + ttl * 1000) : null
    }
    
    // Clean expired memories
    this.cleanExpiredMemories(memory)
    
    await this.db.update(aiAgents)
      .set({ memory })
      .where(eq(aiAgents.id, this.agentId))
  }
  
  async recall(key: string): Promise<any | null> {
    const agent = await this.db.query.aiAgents.findFirst({
      where: eq(aiAgents.id, this.agentId)
    })
    
    if (!agent?.memory?.[key]) return null
    
    const item = agent.memory[key]
    if (item.expiresAt && new Date(item.expiresAt) < new Date()) {
      return null
    }
    
    return item.value
  }
  
  async getConversationHistory(limit: number = 10): Promise<Message[]> {
    const conversations = await this.db.query.aiConversations.findMany({
      where: and(
        eq(aiConversations.agentId, this.agentId),
        eq(aiConversations.isActive, true)
      ),
      with: {
        messages: {
          orderBy: desc(aiMessages.createdAt),
          limit
        }
      },
      orderBy: desc(aiConversations.updatedAt),
      limit: 1
    })
    
    return conversations[0]?.messages || []
  }
  
  async updatePersonality(traits: PersonalityTraits): Promise<void> {
    await this.remember('personality', traits)
  }
  
  async getPersonality(): Promise<PersonalityTraits> {
    const traits = await this.recall('personality')
    return traits || defaultPersonalityTraits
  }
  
  private cleanExpiredMemories(memory: Record<string, any>) {
    const now = new Date()
    for (const key in memory) {
      if (memory[key].expiresAt && new Date(memory[key].expiresAt) < now) {
        delete memory[key]
      }
    }
  }
}

// Contextual memory for decision making
export class ContextualMemory {
  constructor(private memory: AgentMemory) {}
  
  async addInteraction(
    entityType: 'contact' | 'email' | 'task',
    entityId: string,
    interaction: any
  ): Promise<void> {
    const key = `interactions:${entityType}:${entityId}`
    const existing = await this.memory.recall(key) || []
    
    existing.push({
      ...interaction,
      timestamp: new Date()
    })
    
    // Keep last 50 interactions
    const recent = existing.slice(-50)
    
    await this.memory.remember(key, recent)
  }
  
  async getEntityContext(
    entityType: 'contact' | 'email' | 'task',
    entityId: string
  ): Promise<any[]> {
    const key = `interactions:${entityType}:${entityId}`
    return await this.memory.recall(key) || []
  }
  
  async buildContextPrompt(): Promise<string> {
    const personality = await this.memory.getPersonality()
    const recentActivity = await this.memory.recall('recentActivity') || []
    const goals = await this.memory.recall('currentGoals') || []
    
    return `
You are an AI agent with the following characteristics:
- Personality: ${JSON.stringify(personality)}
- Recent activity: ${recentActivity.slice(-5).map(a => a.summary).join('; ')}
- Current goals: ${goals.map(g => g.description).join('; ')}

Remember to:
- Maintain consistency with past interactions
- Reference previous conversations when relevant
- Learn from past successes and failures
    `.trim()
  }
}
```

### Day 5-6: Multi-Agent Coordination
```typescript
// src/lib/agents/coordinator.ts
export class AgentCoordinator {
  constructor(
    private userId: string,
    private db: DB
  ) {}
  
  async coordinateTask(task: CoordinationTask): Promise<CoordinationResult> {
    // Analyze task requirements
    const requirements = await this.analyzeTask(task)
    
    // Find capable agents
    const agents = await this.findCapableAgents(requirements)
    
    // Create execution plan
    const plan = await this.createExecutionPlan(task, agents)
    
    // Execute plan
    return this.executePlan(plan)
  }
  
  private async analyzeTask(task: CoordinationTask): Promise<TaskRequirements> {
    const prompt = `Analyze this task and identify required skills:
    
Task: ${task.description}
Context: ${JSON.stringify(task.context)}

Return JSON with:
- requiredSkills: array of skill IDs needed
- sequencing: 'parallel' | 'sequential' | 'mixed'
- estimatedDuration: minutes
- dependencies: array of dependencies between skills`
    
    const response = await ai.chat([
      { role: 'system', content: 'You are a task analysis expert.' },
      { role: 'user', content: prompt }
    ], {
      response_format: { type: 'json_object' }
    })
    
    return JSON.parse(response.content)
  }
  
  private async findCapableAgents(
    requirements: TaskRequirements
  ): Promise<Agent[]> {
    const agents = await this.db.query.aiAgents.findMany({
      where: and(
        eq(aiAgents.userId, this.userId),
        eq(aiAgents.isActive, true)
      )
    })
    
    // Filter agents with required skills
    return agents.filter(agent => 
      requirements.requiredSkills.every(skill =>
        agent.skills.includes(skill)
      )
    )
  }
  
  private async createExecutionPlan(
    task: CoordinationTask,
    agents: Agent[]
  ): Promise<ExecutionPlan> {
    const plan: ExecutionPlan = {
      id: crypto.randomUUID(),
      task,
      steps: [],
      estimatedDuration: 0
    }
    
    // Build dependency graph
    const graph = this.buildDependencyGraph(task.requirements)
    
    // Topological sort for execution order
    const executionOrder = this.topologicalSort(graph)
    
    // Assign agents to steps
    for (const skillId of executionOrder) {
      const agent = this.selectBestAgent(agents, skillId)
      if (!agent) {
        throw new Error(`No agent available for skill: ${skillId}`)
      }
      
      plan.steps.push({
        id: crypto.randomUUID(),
        skillId,
        agentId: agent.id,
        dependencies: graph.get(skillId)?.dependencies || [],
        status: 'pending'
      })
    }
    
    return plan
  }
  
  private async executePlan(plan: ExecutionPlan): Promise<CoordinationResult> {
    const results = new Map<string, any>()
    const errors: Error[] = []
    
    // Execute steps respecting dependencies
    for (const step of plan.steps) {
      // Wait for dependencies
      await this.waitForDependencies(step, results)
      
      try {
        // Execute step
        const result = await this.executeStep(step, results)
        results.set(step.id, result)
        
        // Notify other agents
        await this.notifyAgents(plan, step, result)
      } catch (error) {
        errors.push(error)
        if (step.critical) {
          break // Stop execution on critical failure
        }
      }
    }
    
    return {
      success: errors.length === 0,
      results: Object.fromEntries(results),
      errors,
      duration: Date.now() - plan.startTime
    }
  }
  
  private async executeStep(
    step: ExecutionStep,
    previousResults: Map<string, any>
  ): Promise<any> {
    const agent = await this.db.query.aiAgents.findFirst({
      where: eq(aiAgents.id, step.agentId)
    })
    
    if (!agent) throw new Error('Agent not found')
    
    const skill = skillRegistry.get(step.skillId)
    if (!skill) throw new Error('Skill not found')
    
    // Build context with previous results
    const context = this.buildStepContext(step, previousResults)
    
    // Execute skill
    const framework = new AgentFramework()
    return framework.executeSkill(agent, skill, context)
  }
}

// Example: Complex task coordination
export async function handleComplexRequest(
  userId: string,
  request: string
): Promise<any> {
  const coordinator = new AgentCoordinator(userId, db)
  
  // Parse request into task
  const task: CoordinationTask = {
    id: crypto.randomUUID(),
    description: request,
    context: {
      userId,
      timestamp: new Date(),
      source: 'user-request'
    },
    requirements: {
      requiredSkills: [],
      sequencing: 'mixed'
    }
  }
  
  // Coordinate execution
  const result = await coordinator.coordinateTask(task)
  
  // Format response
  if (result.success) {
    return {
      status: 'completed',
      results: result.results,
      summary: await generateSummary(result)
    }
  } else {
    return {
      status: 'failed',
      errors: result.errors.map(e => e.message),
      partialResults: result.results
    }
  }
}
```

### Day 7: Proactive Agent Behaviors
```typescript
// src/lib/agents/behaviors/proactive.ts
export interface ProactiveBehavior {
  id: string
  name: string
  description: string
  triggers: BehaviorTrigger[]
  condition: (context: BehaviorContext) => Promise<boolean>
  action: (context: BehaviorContext) => Promise<void>
}

export interface BehaviorTrigger {
  type: 'time' | 'event' | 'condition'
  schedule?: string // cron expression
  events?: string[]
  checkInterval?: number // minutes
}

// Dormant contact reactivation behavior
export const dormantContactReactivation: ProactiveBehavior = {
  id: 'dormant-contact-reactivation',
  name: 'Reactivate Dormant Contacts',
  description: 'Identifies and suggests reactivation for dormant contacts',
  triggers: [
    { type: 'time', schedule: '0 9 * * MON' }, // Every Monday at 9 AM
    { type: 'event', events: ['user.login'] }
  ],
  condition: async (context) => {
    // Check if there are dormant contacts
    const dormantContacts = await context.db.execute(sql`
      SELECT COUNT(*) as count
      FROM contacts c
      WHERE c.user_id = ${context.userId}
        AND c.last_interaction_at < NOW() - INTERVAL '30 days'
        AND c.relationship_strength > 0.5
    `)
    
    return dormantContacts[0].count > 0
  },
  action: async (context) => {
    // Find dormant contacts
    const dormantContacts = await context.db.execute(sql`
      SELECT c.*, cs.sentiment_score, cs.topics
      FROM contacts c
      LEFT JOIN contact_stats cs ON c.id = cs.contact_id
      WHERE c.user_id = ${context.userId}
        AND c.last_interaction_at < NOW() - INTERVAL '30 days'
        AND c.relationship_strength > 0.5
      ORDER BY c.relationship_strength DESC
      LIMIT 5
    `)
    
    // Generate reactivation suggestions
    for (const contact of dormantContacts) {
      const suggestion = await generateReactivationSuggestion(contact)
      
      // Create notification
      await createNotification({
        userId: context.userId,
        type: 'reactivation-suggestion',
        title: `Reconnect with ${contact.name}`,
        body: suggestion.message,
        actions: [
          {
            label: 'Compose Email',
            action: 'compose-email',
            data: { 
              contactId: contact.id,
              draft: suggestion.emailDraft 
            }
          },
          {
            label: 'Dismiss',
            action: 'dismiss'
          }
        ]
      })
    }
  }
}

// Follow-up reminder behavior
export const followUpReminder: ProactiveBehavior = {
  id: 'follow-up-reminder',
  name: 'Follow-up Reminders',
  description: 'Reminds about pending follow-ups',
  triggers: [
    { type: 'time', schedule: '0 */2 * * *' }, // Every 2 hours
    { type: 'event', events: ['email.sent'] }
  ],
  condition: async (context) => {
    const pendingFollowUps = await context.db.query.followUps.findMany({
      where: and(
        eq(followUps.userId, context.userId),
        lte(followUps.dueDate, addHours(new Date(), 24)),
        eq(followUps.completed, false)
      )
    })
    
    return pendingFollowUps.length > 0
  },
  action: async (context) => {
    const pendingFollowUps = await context.db.query.followUps.findMany({
      where: and(
        eq(followUps.userId, context.userId),
        lte(followUps.dueDate, addHours(new Date(), 24)),
        eq(followUps.completed, false)
      ),
      with: {
        contact: true,
        originalEmail: true
      }
    })
    
    for (const followUp of pendingFollowUps) {
      // Check if already followed up
      const recentEmail = await checkRecentEmail(
        followUp.contact.email,
        followUp.originalEmail.threadId
      )
      
      if (!recentEmail) {
        await createNotification({
          userId: context.userId,
          type: 'follow-up-due',
          priority: 'high',
          title: `Follow up with ${followUp.contact.name}`,
          body: `You planned to follow up about: ${followUp.subject}`,
          actions: [
            {
              label: 'Compose Follow-up',
              action: 'compose-followup',
              data: { followUpId: followUp.id }
            },
            {
              label: 'Mark Complete',
              action: 'complete-followup',
              data: { followUpId: followUp.id }
            },
            {
              label: 'Snooze',
              action: 'snooze-followup',
              data: { followUpId: followUp.id }
            }
          ]
        })
      }
    }
  }
}

// Behavior manager
export class BehaviorManager {
  private behaviors = new Map<string, ProactiveBehavior>()
  private running = false
  
  register(behavior: ProactiveBehavior) {
    this.behaviors.set(behavior.id, behavior)
    
    // Register scheduled behaviors
    for (const trigger of behavior.triggers) {
      if (trigger.type === 'time' && trigger.schedule) {
        this.scheduleJob(behavior, trigger.schedule)
      } else if (trigger.type === 'event' && trigger.events) {
        this.registerEventListeners(behavior, trigger.events)
      }
    }
  }
  
  private scheduleJob(behavior: ProactiveBehavior, schedule: string) {
    cron.schedule(schedule, async () => {
      await this.executeBehavior(behavior)
    })
  }
  
  private registerEventListeners(behavior: ProactiveBehavior, events: string[]) {
    for (const event of events) {
      eventEmitter.on(event, async (data) => {
        const context = this.buildContext(data.userId)
        if (await behavior.condition(context)) {
          await behavior.action(context)
        }
      })
    }
  }
  
  async executeBehavior(behavior: ProactiveBehavior) {
    // Execute for all active users
    const activeUsers = await this.getActiveUsers()
    
    for (const user of activeUsers) {
      try {
        const context = this.buildContext(user.id)
        
        if (await behavior.condition(context)) {
          await behavior.action(context)
          
          // Log execution
          await logEvent(
            user.id,
            'agent',
            behavior.id,
            'behavior.executed',
            { behaviorName: behavior.name }
          )
        }
      } catch (error) {
        logger.error(`Behavior execution failed: ${behavior.id}`, error)
      }
    }
  }
  
  private buildContext(userId: string): BehaviorContext {
    return {
      userId,
      db,
      cache: new Cache(),
      logger: createLogger({ behavior: true, userId })
    }
  }
}
```

## Week 8: Advanced Features & Polish

### Day 8-9: Agent Personality System
```typescript
// src/lib/agents/personality.ts
export interface PersonalityTraits {
  formality: number // 0-1 (casual to formal)
  enthusiasm: number // 0-1 (reserved to enthusiastic)
  brevity: number // 0-1 (verbose to concise)
  proactivity: number // 0-1 (reactive to proactive)
  creativity: number // 0-1 (conventional to creative)
  empathy: number // 0-1 (logical to empathetic)
}

export class PersonalityEngine {
  constructor(private traits: PersonalityTraits) {}
  
  adjustResponse(response: string): string {
    let adjusted = response
    
    // Apply formality
    if (this.traits.formality < 0.3) {
      adjusted = this.makeCasual(adjusted)
    } else if (this.traits.formality > 0.7) {
      adjusted = this.makeFormal(adjusted)
    }
    
    // Apply enthusiasm
    if (this.traits.enthusiasm > 0.7) {
      adjusted = this.addEnthusiasm(adjusted)
    }
    
    // Apply brevity
    if (this.traits.brevity > 0.7) {
      adjusted = this.makeConcise(adjusted)
    }
    
    return adjusted
  }
  
  generateSystemPrompt(): string {
    const traits = []
    
    if (this.traits.formality > 0.7) {
      traits.push('formal and professional')
    } else if (this.traits.formality < 0.3) {
      traits.push('casual and friendly')
    }
    
    if (this.traits.enthusiasm > 0.7) {
      traits.push('enthusiastic and energetic')
    }
    
    if (this.traits.brevity > 0.7) {
      traits.push('concise and to-the-point')
    } else if (this.traits.brevity < 0.3) {
      traits.push('detailed and thorough')
    }
    
    if (this.traits.creativity > 0.7) {
      traits.push('creative and innovative')
    }
    
    if (this.traits.empathy > 0.7) {
      traits.push('empathetic and understanding')
    }
    
    return `You have the following personality traits: ${traits.join(', ')}.
Maintain these characteristics consistently in all interactions.`
  }
  
  async evolveFromFeedback(feedback: UserFeedback[]) {
    // Analyze feedback patterns
    const adjustments = {
      formality: 0,
      enthusiasm: 0,
      brevity: 0,
      proactivity: 0,
      creativity: 0,
      empathy: 0
    }
    
    for (const item of feedback) {
      if (item.type === 'too-formal') adjustments.formality -= 0.1
      if (item.type === 'too-casual') adjustments.formality += 0.1
      if (item.type === 'too-long') adjustments.brevity += 0.1
      if (item.type === 'too-short') adjustments.brevity -= 0.1
      // ... more adjustments
    }
    
    // Apply adjustments with bounds
    for (const [trait, adjustment] of Object.entries(adjustments)) {
      this.traits[trait] = Math.max(0, Math.min(1, 
        this.traits[trait] + adjustment
      ))
    }
  }
}

// Agent with personality
export class PersonalizedAgent extends Agent {
  private personality: PersonalityEngine
  
  constructor(config: AgentConfig, traits: PersonalityTraits) {
    super(config)
    this.personality = new PersonalityEngine(traits)
  }
  
  async respond(input: string, context: AgentContext): Promise<string> {
    // Get base response
    const systemPrompt = `${this.config.systemPrompt}

${this.personality.generateSystemPrompt()}`
    
    const response = await ai.chat([
      { role: 'system', content: systemPrompt },
      ...context.conversationHistory,
      { role: 'user', content: input }
    ])
    
    // Apply personality adjustments
    return this.personality.adjustResponse(response.content)
  }
  
  async learnFromInteraction(
    interaction: Interaction,
    feedback?: UserFeedback
  ) {
    // Update memory
    await this.memory.addInteraction(
      interaction.type,
      interaction.entityId,
      interaction
    )
    
    // Evolve personality if feedback provided
    if (feedback) {
      await this.personality.evolveFromFeedback([feedback])
      await this.memory.updatePersonality(this.personality.traits)
    }
  }
}
```

### Day 10-11: Workflow Automation
```typescript
// src/lib/workflows/workflow-engine.ts
export interface Workflow {
  id: string
  name: string
  description: string
  trigger: WorkflowTrigger
  conditions: WorkflowCondition[]
  actions: WorkflowAction[]
  enabled: boolean
}

export interface WorkflowTrigger {
  type: 'email_received' | 'contact_created' | 'time_based' | 'manual'
  filters?: Record<string, any>
  schedule?: string
}

export interface WorkflowAction {
  type: string
  config: Record<string, any>
  continueOnError?: boolean
}

export class WorkflowEngine {
  private workflows = new Map<string, Workflow>()
  
  async createWorkflow(
    userId: string,
    workflow: Omit<Workflow, 'id'>
  ): Promise<Workflow> {
    const id = crypto.randomUUID()
    const fullWorkflow = { ...workflow, id }
    
    // Store workflow
    await db.insert(workflows).values({
      id,
      userId,
      name: workflow.name,
      config: fullWorkflow,
      enabled: workflow.enabled,
    })
    
    // Register workflow
    this.workflows.set(id, fullWorkflow)
    
    // Setup triggers
    await this.setupTriggers(userId, fullWorkflow)
    
    return fullWorkflow
  }
  
  private async setupTriggers(userId: string, workflow: Workflow) {
    switch (workflow.trigger.type) {
      case 'email_received':
        eventEmitter.on('email.received', async (data) => {
          if (data.userId === userId && this.matchesFilters(data, workflow.trigger.filters)) {
            await this.executeWorkflow(workflow, data)
          }
        })
        break
        
      case 'time_based':
        if (workflow.trigger.schedule) {
          cron.schedule(workflow.trigger.schedule, async () => {
            await this.executeWorkflow(workflow, { userId })
          })
        }
        break
    }
  }
  
  async executeWorkflow(workflow: Workflow, context: any): Promise<WorkflowExecution> {
    const execution: WorkflowExecution = {
      id: crypto.randomUUID(),
      workflowId: workflow.id,
      startedAt: new Date(),
      status: 'running',
      results: []
    }
    
    // Check conditions
    const conditionsMet = await this.checkConditions(workflow.conditions, context)
    if (!conditionsMet) {
      execution.status = 'skipped'
      execution.reason = 'Conditions not met'
      return execution
    }
    
    // Execute actions
    for (const action of workflow.actions) {
      try {
        const result = await this.executeAction(action, context)
        execution.results.push({
          action: action.type,
          success: true,
          output: result
        })
        
        // Update context for next action
        context = { ...context, previousResult: result }
      } catch (error) {
        execution.results.push({
          action: action.type,
          success: false,
          error: error.message
        })
        
        if (!action.continueOnError) {
          execution.status = 'failed'
          break
        }
      }
    }
    
    execution.completedAt = new Date()
    execution.status = execution.status || 'completed'
    
    return execution
  }
  
  private async executeAction(
    action: WorkflowAction,
    context: any
  ): Promise<any> {
    switch (action.type) {
      case 'send_email':
        return this.sendEmail(action.config, context)
        
      case 'create_task':
        return this.createTask(action.config, context)
        
      case 'update_contact':
        return this.updateContact(action.config, context)
        
      case 'run_agent':
        return this.runAgent(action.config, context)
        
      case 'webhook':
        return this.callWebhook(action.config, context)
        
      default:
        throw new Error(`Unknown action type: ${action.type}`)
    }
  }
}

// Example workflow: Auto-respond to important emails
export const autoRespondWorkflow: Workflow = {
  id: 'auto-respond-important',
  name: 'Auto-respond to Important Emails',
  description: 'Automatically draft responses to emails marked as important',
  trigger: {
    type: 'email_received',
    filters: {
      isImportant: true,
      isUnread: true
    }
  },
  conditions: [
    {
      type: 'contact_exists',
      config: { checkFrom: true }
    },
    {
      type: 'business_hours',
      config: { timezone: 'America/New_York' }
    }
  ],
  actions: [
    {
      type: 'run_agent',
      config: {
        agentId: 'email-responder',
        skill: 'draft-response',
        params: {
          tone: 'professional',
          maxLength: 200
        }
      }
    },
    {
      type: 'create_task',
      config: {
        title: 'Review auto-drafted response',
        dueIn: '1 hour',
        priority: 'high'
      }
    },
    {
      type: 'send_notification',
      config: {
        title: 'Important email received',
        body: 'Draft response ready for review'
      }
    }
  ],
  enabled: true
}
```

### Day 12-13: Agent Marketplace
```typescript
// src/lib/marketplace/agent-templates.ts
export interface AgentTemplate {
  id: string
  name: string
  description: string
  category: string
  author: string
  rating: number
  installs: number
  config: AgentConfig
  requiredPermissions: string[]
  screenshots: string[]
  documentation: string
}

export class AgentMarketplace {
  private templates = new Map<string, AgentTemplate>()
  
  async publishTemplate(
    template: Omit<AgentTemplate, 'id' | 'rating' | 'installs'>
  ): Promise<AgentTemplate> {
    const id = crypto.randomUUID()
    const fullTemplate = {
      ...template,
      id,
      rating: 0,
      installs: 0
    }
    
    // Validate template
    await this.validateTemplate(fullTemplate)
    
    // Store template
    this.templates.set(id, fullTemplate)
    
    return fullTemplate
  }
  
  async installTemplate(
    userId: string,
    templateId: string,
    customization?: Partial<AgentConfig>
  ): Promise<Agent> {
    const template = this.templates.get(templateId)
    if (!template) throw new Error('Template not found')
    
    // Check permissions
    await this.checkPermissions(userId, template.requiredPermissions)
    
    // Create agent from template
    const agentConfig = {
      ...template.config,
      ...customization,
      name: customization?.name || `${template.name} (${Date.now()})`
    }
    
    const agent = await new AgentFramework().createAgent(userId, agentConfig)
    
    // Update install count
    template.installs++
    
    // Log installation
    await logEvent(
      userId,
      'marketplace',
      templateId,
      'template.installed',
      { agentId: agent.id }
    )
    
    return agent
  }
  
  async searchTemplates(query: string, filters?: MarketplaceFilters): Promise<AgentTemplate[]> {
    let results = Array.from(this.templates.values())
    
    // Text search
    if (query) {
      const lowercaseQuery = query.toLowerCase()
      results = results.filter(t =>
        t.name.toLowerCase().includes(lowercaseQuery) ||
        t.description.toLowerCase().includes(lowercaseQuery)
      )
    }
    
    // Apply filters
    if (filters?.category) {
      results = results.filter(t => t.category === filters.category)
    }
    
    if (filters?.minRating) {
      results = results.filter(t => t.rating >= filters.minRating)
    }
    
    // Sort
    if (filters?.sortBy === 'popular') {
      results.sort((a, b) => b.installs - a.installs)
    } else if (filters?.sortBy === 'rating') {
      results.sort((a, b) => b.rating - a.rating)
    }
    
    return results
  }
}

// Pre-built agent templates
export const preBuiltTemplates: AgentTemplate[] = [
  {
    id: 'sales-assistant',
    name: 'Sales Assistant',
    description: 'Helps manage sales pipeline and follow-ups',
    category: 'sales',
    author: 'CRM Team',
    rating: 4.8,
    installs: 1250,
    config: {
      name: 'Sales Assistant',
      description: 'AI-powered sales assistant',
      trigger: { type: 'manual' },
      skills: [
        'research-contact',
        'draft-sales-email',
        'schedule-follow-up',
        'analyze-deal-probability'
      ],
      systemPrompt: `You are a professional sales assistant focused on 
      helping close deals and maintain customer relationships.`
    },
    requiredPermissions: ['contacts.read', 'emails.write'],
    screenshots: ['/templates/sales-assistant-1.png'],
    documentation: '# Sales Assistant\n\nHelps you manage your sales pipeline...'
  },
  {
    id: 'meeting-scheduler',
    name: 'Meeting Scheduler',
    description: 'Automatically schedules meetings based on email requests',
    category: 'productivity',
    author: 'CRM Team',
    rating: 4.6,
    installs: 890,
    config: {
      name: 'Meeting Scheduler',
      description: 'Automated meeting scheduling',
      trigger: {
        type: 'email_received',
        filters: { contains: ['meeting', 'schedule', 'call'] }
      },
      skills: [
        'extract-meeting-intent',
        'check-calendar',
        'propose-times',
        'send-calendar-invite'
      ]
    },
    requiredPermissions: ['calendar.read', 'calendar.write', 'emails.read'],
    screenshots: ['/templates/meeting-scheduler-1.png'],
    documentation: '# Meeting Scheduler\n\nAutomatically handles meeting requests...'
  }
]
```

### Day 14: Testing & Documentation
```typescript
// src/__tests__/agents/agent-system.test.ts
describe('Agent System', () => {
  let testUser: User
  let testAgent: Agent
  
  beforeEach(async () => {
    testUser = await createTestUser()
    testAgent = await createTestAgent(testUser.id, {
      name: 'Test Agent',
      skills: ['research-contact', 'draft-email']
    })
  })
  
  describe('Skill Execution', () => {
    it('should execute skills with proper context', async () => {
      const contact = await createTestContact(testUser.id)
      
      const result = await executeSkill(testAgent, 'research-contact', {
        contactId: contact.id,
        depth: 'basic'
      })
      
      expect(result.success).toBe(true)
      expect(result.output).toHaveProperty('profile')
      expect(result.output).toHaveProperty('insights')
    })
    
    it('should handle skill failures gracefully', async () => {
      const result = await executeSkill(testAgent, 'research-contact', {
        contactId: 'invalid-id'
      })
      
      expect(result.success).toBe(false)
      expect(result.output.error).toBe('Contact not found')
    })
  })
  
  describe('Agent Memory', () => {
    it('should persist memory across executions', async () => {
      const memory = new AgentMemory(testAgent.id, db)
      
      await memory.remember('test-key', { value: 'test-data' })
      const recalled = await memory.recall('test-key')
      
      expect(recalled).toEqual({ value: 'test-data' })
    })
    
    it('should expire memory based on TTL', async () => {
      const memory = new AgentMemory(testAgent.id, db)
      
      await memory.remember('temp-key', { value: 'temp-data' }, 1) // 1 second TTL
      await sleep(1100)
      
      const recalled = await memory.recall('temp-key')
      expect(recalled).toBeNull()
    })
  })
  
  describe('Multi-Agent Coordination', () => {
    it('should coordinate multiple agents for complex tasks', async () => {
      const agent1 = await createTestAgent(testUser.id, {
        name: 'Research Agent',
        skills: ['research-contact', 'web-search']
      })
      
      const agent2 = await createTestAgent(testUser.id, {
        name: 'Writer Agent',
        skills: ['draft-email', 'summarize']
      })
      
      const coordinator = new AgentCoordinator(testUser.id, db)
      const result = await coordinator.coordinateTask({
        description: 'Research John Doe and draft a personalized email',
        requirements: {
          requiredSkills: ['research-contact', 'draft-email'],
          sequencing: 'sequential'
        }
      })
      
      expect(result.success).toBe(true)
      expect(result.results).toHaveProperty('research')
      expect(result.results).toHaveProperty('email')
    })
  })
  
  describe('Proactive Behaviors', () => {
    it('should trigger behaviors based on conditions', async () => {
      // Create dormant contact
      const contact = await createTestContact(testUser.id, {
        lastInteractionAt: subDays(new Date(), 45)
      })
      
      const behavior = dormantContactReactivation
      const context = buildBehaviorContext(testUser.id)
      
      const shouldTrigger = await behavior.condition(context)
      expect(shouldTrigger).toBe(true)
      
      // Execute behavior
      await behavior.action(context)
      
      // Check for notifications
      const notifications = await getNotifications(testUser.id)
      expect(notifications).toHaveLength(1)
      expect(notifications[0].type).toBe('reactivation-suggestion')
    })
  })
})

// Integration test
describe('End-to-End Agent Workflow', () => {
  it('should handle complete email response workflow', async () => {
    const { user, agent } = await setupTestEnvironment()
    
    // Simulate email received
    const email = await simulateEmailReceived(user.id, {
      from: 'important@client.com',
      subject: 'Urgent: Project Update Needed',
      body: 'Can you send me the latest project status?'
    })
    
    // Agent should process email
    await waitForJobCompletion('email-analysis', email.id)
    
    // Check if agent created draft
    const drafts = await getDrafts(user.id)
    expect(drafts).toHaveLength(1)
    expect(drafts[0].subject).toContain('Re: Urgent')
    
    // Check if follow-up was scheduled
    const followUps = await getFollowUps(user.id)
    expect(followUps).toHaveLength(1)
  })
})
```

## Phase 3 Deliverables Checklist

### Agent Infrastructure ✓
- [ ] Skill library system with registry
- [ ] Skill validation and documentation
- [ ] Agent memory persistence
- [ ] Contextual memory for decisions
- [ ] Multi-agent coordination framework

### Proactive Behaviors ✓
- [ ] Behavior trigger system
- [ ] Dormant contact reactivation
- [ ] Follow-up reminders
- [ ] Smart notifications
- [ ] Behavior scheduling

### Advanced Features ✓
- [ ] Agent personality system
- [ ] Personality evolution from feedback
- [ ] Workflow automation engine
- [ ] Visual workflow builder
- [ ] Agent marketplace

### Integration ✓
- [ ] Seamless UI integration
- [ ] Real-time agent status
- [ ] Agent conversation UI
- [ ] Workflow management UI
- [ ] Marketplace browser

## Success Metrics Validation

By the end of Phase 3:

1. **Agents complete > 80% of assigned tasks** ✓
   - Comprehensive skill library
   - Robust error handling
   - Fallback strategies

2. **User engagement with suggestions > 50%** ✓
   - Relevant proactive behaviors
   - Timely notifications
   - Actionable suggestions

3. **Agent coherence across sessions > 90%** ✓
   - Persistent memory system
   - Personality consistency
   - Context awareness

4. **Workflow automation saves > 2 hours/week** ✓
   - Common tasks automated
   - Efficient execution
   - Minimal user intervention

## System Complete

The AI CRM is now a fully functional, intelligent system that:

1. **Syncs all data** reliably and in real-time
2. **Understands context** through advanced AI
3. **Acts proactively** with intelligent agents
4. **Learns and adapts** from user behavior
5. **Saves significant time** through automation

The system is ready for deployment and scaling to production use.