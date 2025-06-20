# Async Tasks for FluffyMail AI CRM

## Overview

This document outlines all asynchronous tasks needed for the AI CRM, organized by priority and implementation phase. Each task includes its purpose, trigger, frequency, and processing requirements.

## Task Categories

### 1. Data Sync Tasks
Tasks that fetch and synchronize data from external sources.

### 2. Data Processing Tasks
Tasks that analyze and transform existing data.

### 3. AI/ML Tasks
Tasks that use AI models for analysis and generation.

### 4. Notification Tasks
Tasks that monitor conditions and alert users.

### 5. Maintenance Tasks
Tasks that clean up and optimize the system.

## Phase 1: Foundation (Week 1)

### 1.1 Initial Email Sync
**Job Name:** `sync-emails-initial`
**Purpose:** Bulk import historical emails when user first connects
**Trigger:** User connects Gmail/Outlook account
**Processing:**
```typescript
interface InitialSyncJob {
  connectionId: string;
  userId: string;
  startDate: Date; // Usually 2 years ago
  batchSize: number; // 100-500 emails per batch
}

// Fetches all historical emails in batches
// Creates follow-up jobs for each batch
// Handles rate limiting and pagination
```
**Priority:** High
**Duration:** 5-60 minutes depending on mailbox size
**Retries:** 5 with exponential backoff

### 1.2 Incremental Email Sync
**Job Name:** `sync-emails-incremental`
**Purpose:** Fetch new emails since last sync
**Trigger:** 
- Webhook from Gmail/Outlook
- Scheduled every 15 minutes as fallback
**Processing:**
```typescript
interface IncrementalSyncJob {
  connectionId: string;
  userId: string;
  lastSyncToken?: string;
  lastSyncDate: Date;
}

// Fetches only new/modified emails
// Updates sync token for next run
// Triggers contact extraction for new emails
```
**Priority:** High
**Duration:** 10-60 seconds
**Retries:** 3

### 1.3 Contact Extraction
**Job Name:** `extract-contacts`
**Purpose:** Extract contact information from emails
**Trigger:** New email synced
**Processing:**
```typescript
interface ExtractContactsJob {
  emailId: string;
  userId: string;
  force?: boolean; // Re-extract even if done
}

// Parses email headers (from, to, cc)
// Extracts names from signatures
// Identifies company domains
// Creates or updates contact records
// Handles duplicates intelligently
```
**Priority:** Medium
**Duration:** 1-5 seconds per email
**Retries:** 3

### 1.4 Calendar Sync
**Job Name:** `sync-calendar-events`
**Purpose:** Sync Google Calendar meetings
**Trigger:** 
- Initial connection
- Webhook updates
- Daily scheduled sync
**Processing:**
```typescript
interface CalendarSyncJob {
  connectionId: string;
  userId: string;
  startDate: Date;
  endDate: Date;
}

// Fetches calendar events
// Extracts attendee information
// Links to existing contacts
// Creates meeting records
```
**Priority:** Medium
**Duration:** 30-120 seconds
**Retries:** 3

## Phase 2: AI Intelligence (Week 2-3)

### 2.1 Contact Enrichment
**Job Name:** `enrich-contact`
**Purpose:** Find additional information about contacts
**Trigger:** 
- New contact created
- Manual enrichment request
- Weekly batch enrichment
**Processing:**
```typescript
interface EnrichContactJob {
  contactId: string;
  userId: string;
  sources: ('linkedin' | 'twitter' | 'company' | 'web')[];
}

// Searches multiple data sources
// Validates found information
// Updates contact with:
//   - Title, company, location
//   - Social profiles
//   - Bio/description
//   - Profile photo
```
**Priority:** Low
**Duration:** 5-30 seconds per contact
**Retries:** 2
**Rate Limits:** Respect API limits

### 2.2 Email Summarization
**Job Name:** `summarize-email-thread`
**Purpose:** Generate AI summary of email threads
**Trigger:** 
- New email in thread
- Thread viewed by user
- Batch processing for historical
**Processing:**
```typescript
interface SummarizeThreadJob {
  threadId: string;
  userId: string;
  model?: 'fast' | 'accurate';
}

// Fetches all emails in thread
// Generates concise summary
// Extracts key points and action items
// Identifies sentiment and urgency
```
**Priority:** Medium
**Duration:** 2-10 seconds
**Retries:** 2

### 2.3 Relationship Analysis
**Job Name:** `analyze-relationships`
**Purpose:** Calculate relationship strength and patterns
**Trigger:** 
- Daily scheduled job
- After significant email activity
**Processing:**
```typescript
interface AnalyzeRelationshipsJob {
  userId: string;
  contactId?: string; // Specific or all
  timeframe: '30d' | '90d' | '1y';
}

// Analyzes communication patterns:
//   - Frequency of contact
//   - Response times
//   - Initiation patterns
//   - Meeting frequency
//   - Email sentiment
// Calculates relationship score
// Identifies at-risk relationships
```
**Priority:** Low
**Duration:** 1-5 minutes
**Retries:** 2

### 2.4 Smart Categorization
**Job Name:** `categorize-emails`
**Purpose:** AI-powered email categorization
**Trigger:** New emails synced
**Processing:**
```typescript
interface CategorizeEmailsJob {
  emailIds: string[];
  userId: string;
}

// Uses AI to categorize:
//   - Important vs FYI
//   - Requires response
//   - Project/topic tags
//   - Priority level
// Learns from user corrections
```
**Priority:** Medium
**Duration:** 1-3 seconds per email
**Retries:** 2

### 2.5 Writing Style Analysis
**Job Name:** `analyze-writing-style`
**Purpose:** Analyze user's writing patterns for AI assistance
**Trigger:** 
- Initial setup
- Monthly update
**Processing:**
```typescript
interface AnalyzeWritingStyleJob {
  userId: string;
  sampleSize?: number; // Default 100 sent emails
}

// Analyzes sent emails for:
//   - Tone and formality
//   - Common phrases
//   - Email structure
//   - Sign-offs
// Creates writing profile for AI
```
**Priority:** Low
**Duration:** 2-5 minutes
**Retries:** 2

## Phase 3: Proactive Agents (Week 3-4)

### 3.1 Follow-up Reminder Agent
**Job Name:** `check-follow-ups`
**Purpose:** Identify emails needing follow-up
**Trigger:** Daily at 9 AM user timezone
**Processing:**
```typescript
interface CheckFollowUpsJob {
  userId: string;
  lookbackDays: number; // Usually 3-7
}

// Identifies sent emails with no response
// Checks importance and context
// Creates reminder tasks
// Drafts follow-up templates
```
**Priority:** Medium
**Duration:** 30-60 seconds
**Retries:** 2

### 3.2 Meeting Prep Agent
**Job Name:** `prepare-meeting`
**Purpose:** Gather context before meetings
**Trigger:** 1 hour before calendar event
**Processing:**
```typescript
interface PrepareMeetingJob {
  meetingId: string;
  userId: string;
}

// Gathers:
//   - Recent emails with attendees
//   - Previous meeting notes
//   - Relevant documents
//   - Suggested talking points
// Sends prep email 30min before
```
**Priority:** High
**Duration:** 30-90 seconds
**Retries:** 1

### 3.3 Lead Scoring Agent
**Job Name:** `score-leads`
**Purpose:** Calculate deal probability
**Trigger:** 
- New contact activity
- Weekly batch scoring
**Processing:**
```typescript
interface ScoreLeadsJob {
  userId: string;
  contactIds?: string[];
}

// Analyzes:
//   - Email engagement patterns
//   - Meeting frequency
//   - Response times
//   - Email sentiment
//   - Company signals
// Assigns probability score
// Identifies hot leads
```
**Priority:** Low
**Duration:** 10-60 seconds
**Retries:** 2

### 3.4 Relationship Reactivation Agent
**Job Name:** `find-dormant-relationships`
**Purpose:** Identify relationships to reactivate
**Trigger:** Weekly on Mondays
**Processing:**
```typescript
interface FindDormantRelationshipsJob {
  userId: string;
  inactiveDays: number; // Default 90
  minRelationshipScore: number;
}

// Finds previously active contacts
// Suggests reactivation reason
// Drafts personalized outreach
// Creates reactivation tasks
```
**Priority:** Low
**Duration:** 1-3 minutes
**Retries:** 2

### 3.5 Email Draft Agent
**Job Name:** `generate-email-draft`
**Purpose:** Create AI-powered email drafts
**Trigger:** User requests draft
**Processing:**
```typescript
interface GenerateEmailDraftJob {
  userId: string;
  recipientId: string;
  context: {
    purpose: string;
    keyPoints: string[];
    tone?: string;
  };
}

// Analyzes recipient communication style
// Reviews recent context
// Generates personalized draft
// Applies user's writing style
```
**Priority:** High
**Duration:** 5-15 seconds
**Retries:** 2

## Phase 4: Advanced Features (Future)

### 4.1 Deal Pipeline Agent
**Job Name:** `update-deal-pipeline`
**Purpose:** Automatically move deals through stages
**Trigger:** Email/meeting activity
**Processing:**
```typescript
interface UpdateDealPipelineJob {
  dealId: string;
  activityType: 'email' | 'meeting' | 'signature';
}

// Analyzes activity for deal signals
// Updates deal stage
// Calculates close probability
// Alerts on important changes
```

### 4.2 Competitive Intelligence Agent
**Job Name:** `monitor-competitors`
**Purpose:** Track competitor mentions
**Trigger:** Daily
**Processing:**
```typescript
interface MonitorCompetitorsJob {
  userId: string;
  competitors: string[];
}

// Scans emails for competitor mentions
// Tracks win/loss patterns
// Identifies competitive situations
```

### 4.3 Contract Analysis Agent
**Job Name:** `analyze-contract`
**Purpose:** Extract key terms from attachments
**Trigger:** Contract attachment detected
**Processing:**
```typescript
interface AnalyzeContractJob {
  attachmentId: string;
  userId: string;
}

// Extracts contract text
// Identifies key terms
// Flags important dates
// Compares to templates
```

## Maintenance Tasks

### M.1 Data Cleanup
**Job Name:** `cleanup-old-data`
**Purpose:** Remove old temporary data
**Trigger:** Daily at 3 AM
**Processing:**
- Delete orphaned records
- Clean old job logs
- Archive old emails (optional)

### M.2 Sync Health Check
**Job Name:** `check-sync-health`
**Purpose:** Ensure all connections are working
**Trigger:** Every 6 hours
**Processing:**
- Test each connection
- Refresh tokens if needed
- Alert on failures

### M.3 Analytics Aggregation
**Job Name:** `aggregate-analytics`
**Purpose:** Pre-calculate analytics data
**Trigger:** Daily at 2 AM
**Processing:**
- Calculate daily/weekly/monthly stats
- Update relationship scores
- Generate trend data

## Implementation Priority

### Week 1: Core Sync
1. Initial email sync ⭐
2. Incremental email sync ⭐
3. Contact extraction ⭐
4. Calendar sync

### Week 2: Basic AI
1. Contact enrichment
2. Email summarization
3. Smart categorization
4. Relationship analysis

### Week 3: Proactive Features
1. Follow-up reminders
2. Meeting prep
3. Email draft generation
4. Writing style analysis

### Week 4: Advanced Agents
1. Lead scoring
2. Relationship reactivation
3. Deal pipeline updates
4. Maintenance tasks

## Technical Considerations

### Queue Configuration
```typescript
// High priority queues (< 1 min processing)
const fastQueues = [
  'extract-contacts',
  'categorize-emails',
  'generate-email-draft'
];

// Medium priority queues (1-5 min)
const normalQueues = [
  'sync-emails-incremental',
  'summarize-email-thread',
  'check-follow-ups'
];

// Low priority queues (> 5 min)
const slowQueues = [
  'sync-emails-initial',
  'analyze-relationships',
  'enrich-contact'
];
```

### Worker Scaling
```typescript
// Configure workers based on queue
const workerConfig = {
  'extract-contacts': { teamSize: 10, teamConcurrency: 5 },
  'sync-emails-initial': { teamSize: 2, teamConcurrency: 1 },
  'enrich-contact': { teamSize: 5, teamConcurrency: 2 },
  'generate-email-draft': { teamSize: 3, teamConcurrency: 3 }
};
```

### Error Handling
- All jobs should be idempotent
- Failed jobs retry with exponential backoff
- Dead letter queue for investigation
- Monitoring alerts for repeated failures

This comprehensive async task system powers the intelligence of the AI CRM, turning raw email data into actionable insights and automating routine tasks to save users 10+ hours per month.