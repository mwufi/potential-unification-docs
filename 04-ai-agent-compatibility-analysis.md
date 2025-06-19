# AI Agent Compatibility Analysis

## Overview

An AI agent that's aware of both Drive files and Calendar events would need to interact with structured data from both systems. This analysis examines what components from each codebase would be necessary and how to architect such an integration.

## Required Components from Each Codebase

### From Analog (Calendar)

#### 1. Data Models & Types
```typescript
// Essential interfaces
interface Calendar {
  id: string
  name: string
  timeZone?: string
  primary: boolean
  color?: string
}

interface CalendarEvent {
  id: string
  title: string
  description?: string
  start: Temporal.ZonedDateTime
  end: Temporal.ZonedDateTime
  allDay?: boolean
  location?: string
  status?: string
}
```

#### 2. Provider Layer
- `GoogleCalendarProvider` class
- `MicrosoftCalendarProvider` class
- Base `CalendarProvider` interface
- Error handling utilities

#### 3. API Endpoints (tRPC)
- `events.list` - Query events by time range
- `events.create` - Create new events
- `events.update` - Modify events
- `events.delete` - Remove events
- `calendars.list` - Get user calendars

#### 4. Temporal Handling
- Temporal polyfill integration
- Date/time utilities
- Timezone management

### From Nimbus (Drive)

#### 1. Data Models & Types
```typescript
interface File {
  id: string
  name: string
  mimeType: string
  size?: string
  createdTime?: string
  modifiedTime?: string
  parents?: string[]
  trashed?: boolean
}
```

#### 2. Provider Layer
- `GoogleDriveProvider` class
- Future multi-provider support structure

#### 3. API Endpoints (REST/Hono)
- `GET /files` - List all files
- `GET /files/:id` - Get specific file
- `POST /files` - Create file/folder
- `PUT /files/:id` - Update file
- `DELETE /files/:id` - Delete file

#### 4. File Operations
- File metadata handling
- Folder structure navigation
- MIME type management

## AI Agent Architecture

### 1. Unified Data Layer

```typescript
interface UnifiedContext {
  // Calendar context
  upcomingEvents: CalendarEvent[]
  calendars: Calendar[]
  
  // Drive context
  recentFiles: File[]
  folders: File[]
  
  // Cross-context
  eventAttachments: Map<string, File[]>
  fileCalendarReferences: Map<string, CalendarEvent[]>
}
```

### 2. Agent Capabilities

#### Calendar Operations
```typescript
interface CalendarCapabilities {
  // Query
  findEvents(query: string, timeRange?: TimeRange): CalendarEvent[]
  getAvailability(timeRange: TimeRange): TimeSlot[]
  searchEventsByParticipant(email: string): CalendarEvent[]
  
  // Actions
  scheduleEvent(params: EventParams): CalendarEvent
  rescheduleEvent(eventId: string, newTime: TimeRange): CalendarEvent
  addEventAttachment(eventId: string, fileId: string): void
  
  // Intelligence
  suggestMeetingTimes(participants: string[], duration: number): TimeSlot[]
  detectSchedulingConflicts(event: CalendarEvent): Conflict[]
}
```

#### Drive Operations
```typescript
interface DriveCapabilities {
  // Query
  searchFiles(query: string): File[]
  findRelatedFiles(context: string): File[]
  getFilesByType(mimeType: string): File[]
  
  // Actions
  createDocument(title: string, content: string): File
  organizeFiles(strategy: OrganizationStrategy): void
  shareFile(fileId: string, recipients: string[]): void
  
  // Intelligence
  suggestRelevantFiles(eventContext: CalendarEvent): File[]
  categorizeFiles(files: File[]): FileCategories
}
```

#### Cross-Domain Intelligence
```typescript
interface CrossDomainCapabilities {
  // Meeting preparation
  prepareMeetingMaterials(event: CalendarEvent): {
    agenda: File
    relevantDocs: File[]
    actionItems: string[]
  }
  
  // Document scheduling
  scheduleDocumentReview(file: File, reviewers: string[]): CalendarEvent
  
  // Workflow automation
  createProjectWorkflow(projectName: string): {
    folder: File
    kickoffMeeting: CalendarEvent
    timeline: CalendarEvent[]
  }
  
  // Context awareness
  getContextForTimeRange(range: TimeRange): {
    events: CalendarEvent[]
    activeDocuments: File[]
    suggestions: string[]
  }
}
```

### 3. Implementation Requirements

#### Authentication & Permissions
```typescript
interface AgentAuth {
  // Unified token management
  googleTokens: {
    calendar: OAuthToken
    drive: OAuthToken
  }
  microsoftTokens?: {
    calendar: OAuthToken
    onedrive: OAuthToken
  }
  
  // Permission scopes needed
  requiredScopes: {
    calendar: ['calendar.read', 'calendar.write']
    drive: ['drive.read', 'drive.write', 'drive.file']
  }
}
```

#### Natural Language Processing
```typescript
interface NLPRequirements {
  // Intent detection
  detectIntent(query: string): {
    domain: 'calendar' | 'drive' | 'both'
    action: string
    entities: Entity[]
  }
  
  // Entity extraction
  extractTimeReferences(text: string): Temporal.ZonedDateTime[]
  extractFileReferences(text: string): string[]
  extractPeople(text: string): string[]
}
```

## Integration Architecture

### Option 1: Unified API Gateway
```
┌─────────────────┐
│   AI Agent      │
└────────┬────────┘
         │
┌────────▼────────┐
│  Unified API    │
│    Gateway      │
├─────────────────┤
│ - Auth Manager  │
│ - Rate Limiter  │
│ - Cache Layer   │
└────┬───────┬────┘
     │       │
┌────▼───┐ ┌─▼─────┐
│ Analog │ │Nimbus │
│  API   │ │ API   │
└────────┘ └───────┘
```

### Option 2: Direct Integration
```
┌─────────────────┐
│   AI Agent      │
├─────────────────┤
│ Calendar Module │──► Analog tRPC API
│ Drive Module    │──► Nimbus REST API
│ Cross Module    │──► Both APIs
└─────────────────┘
```

### Option 3: Event-Driven Architecture
```
┌──────────────┐     ┌──────────────┐
│ Calendar API │────►│ Event Bus    │
└──────────────┘     └──────┬───────┘
                            │
┌──────────────┐           ▼
│  Drive API   │────►┌──────────────┐
└──────────────┘     │  AI Agent    │
                     │  Processor   │
                     └──────────────┘
```

## Required Enhancements

### For Analog
1. **Event Search API**: Full-text search across events
2. **Bulk Operations**: Batch create/update events
3. **Webhooks**: Real-time event notifications
4. **Attachment Support**: Link files to events
5. **Advanced Queries**: Complex date/participant filters

### For Nimbus
1. **Content Search**: Search within file contents
2. **Metadata API**: Custom file properties
3. **Batch Operations**: Multi-file operations
4. **Change Notifications**: File change webhooks
5. **Permission APIs**: Share and permission management

### For Both
1. **Unified Auth**: Single token for both services
2. **Rate Limit Coordination**: Shared quota management
3. **Caching Strategy**: Coordinated cache invalidation
4. **Error Handling**: Consistent error formats
5. **Audit Logging**: Track AI agent actions

## Recommended Architecture

### Phase 1: Adapter Pattern
Create adapters for each service that normalize data:

```typescript
class CalendarAdapter {
  constructor(private provider: CalendarProvider) {}
  
  async getEventsWithContext(timeRange: TimeRange): EventWithContext[] {
    const events = await this.provider.events(...)
    return this.enrichWithContext(events)
  }
}

class DriveAdapter {
  constructor(private provider: DriveProvider) {}
  
  async getFilesWithMetadata(query: string): FileWithMetadata[] {
    const files = await this.provider.search(query)
    return this.enrichWithMetadata(files)
  }
}
```

### Phase 2: Context Engine
Build a context engine that maintains state:

```typescript
class ContextEngine {
  private calendarAdapter: CalendarAdapter
  private driveAdapter: DriveAdapter
  private cache: ContextCache
  
  async buildContext(userId: string): UnifiedContext {
    // Parallel fetch
    const [events, files] = await Promise.all([
      this.calendarAdapter.getUpcomingEvents(),
      this.driveAdapter.getRecentFiles()
    ])
    
    // Cross-reference
    const enriched = this.crossReference(events, files)
    
    return this.cache.set(userId, enriched)
  }
}
```

### Phase 3: AI Agent Interface
Implement the agent with natural language understanding:

```typescript
class FluffyMailAgent {
  async process(query: string, context: UnifiedContext): AgentResponse {
    const intent = this.nlp.detectIntent(query)
    
    switch(intent.domain) {
      case 'calendar':
        return this.calendarOps.handle(intent, context)
      case 'drive':
        return this.driveOps.handle(intent, context)
      case 'both':
        return this.crossOps.handle(intent, context)
    }
  }
}
```

## Conclusion

Building an AI agent aware of both Drive and Calendar requires:

1. **From existing codebases**: Provider classes, data models, API endpoints
2. **New components**: Unified context, adapters, NLP processing
3. **Enhancements**: Search APIs, webhooks, batch operations
4. **Architecture**: Adapter pattern with context engine

The modular architecture of both codebases makes them well-suited for AI agent integration. The main challenges are:
- Coordinating rate limits across services
- Building efficient cross-domain queries
- Maintaining context consistency
- Handling authentication across providers

Estimated effort: 4-6 weeks for basic agent, 8-12 weeks for full-featured implementation.