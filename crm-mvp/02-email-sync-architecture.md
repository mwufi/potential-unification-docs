# Email Sync & Contact Extraction Architecture

## Overview

This document details the email synchronization and contact extraction system, the core engine of our AI CRM. We'll use Gmail API with intelligent parsing to build a comprehensive contact database.

## Email Sync Architecture

### 1. Gmail API Integration

```typescript
// Using Gmail API v1 with better-auth tokens
interface GmailService {
  // Initial sync
  performInitialSync(userId: string): Promise<InitialSyncResult>
  
  // Incremental sync
  syncNewEmails(userId: string, historyId: string): Promise<SyncResult>
  
  // Real-time updates
  setupPushNotifications(userId: string): Promise<WatchResponse>
  
  // Thread operations
  getFullThread(threadId: string): Promise<EmailThread>
}

interface InitialSyncResult {
  emailCount: number
  oldestEmail: Date
  newestEmail: Date
  historyId: string
  extractedContacts: number
}
```

### 2. Sync Strategy

#### Initial Sync (First Time)
```typescript
class InitialSyncStrategy {
  async execute(userId: string) {
    // 1. Start with most recent emails
    const recentEmails = await gmail.messages.list({
      maxResults: 500,
      q: 'newer_than:3m' // Last 3 months
    })
    
    // 2. Process in batches to avoid timeouts
    for (const batch of chunk(recentEmails, 50)) {
      await jobQueue.enqueue({
        type: 'process-email-batch',
        data: { userId, messageIds: batch.map(m => m.id) }
      })
    }
    
    // 3. Set up incremental sync
    const { historyId } = await gmail.getProfile()
    await saveUserSyncState(userId, { historyId, lastSync: new Date() })
    
    // 4. Schedule deep sync for older emails
    await jobQueue.enqueue({
      type: 'deep-email-sync',
      data: { userId, before: '3m' },
      runAt: new Date(Date.now() + 3600000) // 1 hour later
    })
  }
}
```

#### Incremental Sync
```typescript
class IncrementalSyncStrategy {
  async execute(userId: string) {
    const { historyId } = await getUserSyncState(userId)
    
    // Get changes since last sync
    const history = await gmail.history.list({
      startHistoryId: historyId,
      historyTypes: ['messageAdded', 'messageDeleted']
    })
    
    // Process new messages
    for (const record of history.history || []) {
      if (record.messagesAdded) {
        await this.processNewMessages(userId, record.messagesAdded)
      }
    }
    
    // Update sync state
    await updateUserSyncState(userId, {
      historyId: history.historyId,
      lastSync: new Date()
    })
  }
}
```

#### Real-time Sync (Push Notifications)
```typescript
class RealtimeSyncSetup {
  async setupWatch(userId: string) {
    // Create Pub/Sub topic for this user
    const topicName = `projects/crm-project/topics/gmail-${userId}`
    
    const response = await gmail.users.watch({
      userId: 'me',
      requestBody: {
        topicName,
        labelIds: ['INBOX', 'SENT']
      }
    })
    
    // Store watch expiration
    await saveWatchExpiration(userId, response.expiration)
    
    // Schedule renewal before expiration
    await jobQueue.enqueue({
      type: 'renew-gmail-watch',
      data: { userId },
      runAt: new Date(parseInt(response.expiration) - 3600000)
    })
  }
}

// Webhook handler
export async function handleGmailWebhook(req: Request) {
  const message = parsePubSubMessage(req.body)
  const { userId, historyId } = message.data
  
  await jobQueue.enqueue({
    type: 'incremental-sync',
    data: { userId, historyId },
    priority: 'high'
  })
}
```

### 3. Email Processing Pipeline

```typescript
class EmailProcessor {
  async processEmail(userId: string, messageId: string) {
    // 1. Fetch full message
    const message = await gmail.messages.get({
      id: messageId,
      format: 'full'
    })
    
    // 2. Parse email data
    const email = this.parseGmailMessage(message)
    
    // 3. Store email
    await db.insert(emails).values({
      id: email.id,
      userId,
      threadId: email.threadId,
      from: email.from,
      to: email.to,
      cc: email.cc,
      subject: email.subject,
      bodyText: email.bodyText,
      bodyHtml: email.bodyHtml,
      sentAt: email.sentAt,
      labels: email.labels
    })
    
    // 4. Extract contacts
    const contacts = await this.extractContacts(email)
    
    // 5. Update interactions
    await this.recordInteractions(userId, email, contacts)
    
    // 6. Emit events
    await eventBus.emit('email.processed', {
      userId,
      emailId: email.id,
      contactsFound: contacts.length
    })
  }
  
  private parseGmailMessage(message: gmail.Message): ParsedEmail {
    const headers = message.payload.headers
    const getHeader = (name: string) => 
      headers.find(h => h.name.toLowerCase() === name.toLowerCase())?.value
    
    return {
      id: message.id,
      threadId: message.threadId,
      from: parseEmailAddress(getHeader('from')),
      to: parseEmailAddressList(getHeader('to')),
      cc: parseEmailAddressList(getHeader('cc')),
      subject: getHeader('subject'),
      sentAt: new Date(parseInt(message.internalDate)),
      bodyText: extractTextBody(message.payload),
      bodyHtml: extractHtmlBody(message.payload),
      labels: message.labelIds || []
    }
  }
}
```

## Contact Extraction System

### 1. Multi-Source Contact Extraction

```typescript
class ContactExtractor {
  private extractors: ContactExtractorStrategy[] = [
    new EmailHeaderExtractor(),
    new SignatureExtractor(),
    new BodyContentExtractor(),
    new DomainIntelligenceExtractor()
  ]
  
  async extractContacts(email: ParsedEmail): Promise<ExtractedContact[]> {
    const allContacts = new Map<string, ExtractedContact>()
    
    // Run all extractors
    for (const extractor of this.extractors) {
      const contacts = await extractor.extract(email)
      
      // Merge with existing
      for (const contact of contacts) {
        const existing = allContacts.get(contact.email)
        if (existing) {
          allContacts.set(contact.email, this.mergeContacts(existing, contact))
        } else {
          allContacts.set(contact.email, contact)
        }
      }
    }
    
    // Score confidence
    return Array.from(allContacts.values()).map(contact => ({
      ...contact,
      confidence: this.calculateConfidence(contact)
    }))
  }
}
```

### 2. Extraction Strategies

#### Email Header Extractor
```typescript
class EmailHeaderExtractor implements ContactExtractorStrategy {
  extract(email: ParsedEmail): ExtractedContact[] {
    const contacts: ExtractedContact[] = []
    
    // Extract from 'From' field
    if (email.from) {
      contacts.push({
        email: email.from.email,
        name: email.from.name,
        source: 'email_header_from',
        firstSeen: email.sentAt
      })
    }
    
    // Extract from 'To' and 'CC' fields
    for (const recipient of [...email.to, ...email.cc]) {
      contacts.push({
        email: recipient.email,
        name: recipient.name,
        source: 'email_header_recipient',
        firstSeen: email.sentAt
      })
    }
    
    return contacts
  }
}
```

#### Signature Extractor
```typescript
class SignatureExtractor implements ContactExtractorStrategy {
  private signaturePatterns = [
    /^--\s*$/m,  // Standard signature delimiter
    /^(Best|Regards|Sincerely|Thanks)/m,
    /^Sent from my/m
  ]
  
  extract(email: ParsedEmail): ExtractedContact[] {
    const signature = this.extractSignature(email.bodyText)
    if (!signature) return []
    
    const contact: Partial<ExtractedContact> = {
      source: 'email_signature'
    }
    
    // Extract phone
    const phone = signature.match(/[\+]?[(]?[0-9]{3}[)]?[-\s\.]?[(]?[0-9]{3}[)]?[-\s\.]?[0-9]{4,6}/)?.[0]
    if (phone) contact.phone = phone
    
    // Extract title and company
    const titleCompany = signature.match(/^(.+?)\s*[\|,@]\s*(.+?)$/m)
    if (titleCompany) {
      contact.title = titleCompany[1].trim()
      contact.company = titleCompany[2].trim()
    }
    
    // Extract social links
    const linkedin = signature.match(/linkedin\.com\/in\/([a-zA-Z0-9-]+)/)?.[1]
    if (linkedin) contact.linkedinUrl = `https://linkedin.com/in/${linkedin}`
    
    const twitter = signature.match(/(?:twitter\.com\/|@)([a-zA-Z0-9_]+)/)?.[1]
    if (twitter) contact.twitterHandle = twitter
    
    return contact.email ? [contact as ExtractedContact] : []
  }
  
  private extractSignature(text: string): string | null {
    // Find signature start
    for (const pattern of this.signaturePatterns) {
      const match = text.match(pattern)
      if (match) {
        return text.slice(match.index)
      }
    }
    
    // Fallback: last 10 lines
    const lines = text.split('\n')
    if (lines.length > 10) {
      return lines.slice(-10).join('\n')
    }
    
    return null
  }
}
```

#### Domain Intelligence Extractor
```typescript
class DomainIntelligenceExtractor implements ContactExtractorStrategy {
  private domainPatterns = {
    enterprise: /^.+@(?!gmail|yahoo|hotmail|outlook|icloud|me|mac)/,
    role: /^(info|contact|sales|support|admin|hello)@/
  }
  
  async extract(email: ParsedEmail): Promise<ExtractedContact[]> {
    const contacts: ExtractedContact[] = []
    
    for (const address of [email.from, ...email.to, ...email.cc]) {
      if (!address?.email) continue
      
      const domain = address.email.split('@')[1]
      if (!domain || !this.isEnterpriseDomain(address.email)) continue
      
      // Infer company from domain
      const company = await this.inferCompanyName(domain)
      
      contacts.push({
        email: address.email,
        name: address.name,
        company,
        source: 'domain_intelligence',
        metadata: {
          isEnterpriseEmail: true,
          isRoleEmail: this.domainPatterns.role.test(address.email)
        }
      })
    }
    
    return contacts
  }
  
  private async inferCompanyName(domain: string): Promise<string> {
    // Remove common suffixes
    let company = domain.replace(/\.(com|co|io|net|org|ai)$/, '')
    
    // Convert to title case
    company = company.split(/[-.]/)
      .map(word => word.charAt(0).toUpperCase() + word.slice(1))
      .join(' ')
    
    return company
  }
}
```

### 3. Contact Deduplication & Merging

```typescript
class ContactMerger {
  async mergeWithExisting(userId: string, extracted: ExtractedContact[]): Promise<Contact[]> {
    const merged: Contact[] = []
    
    for (const newContact of extracted) {
      // Find existing contact
      const existing = await db.query.contacts.findFirst({
        where: and(
          eq(contacts.userId, userId),
          eq(contacts.email, newContact.email)
        )
      })
      
      if (existing) {
        // Merge data (prefer user-edited data)
        const updated = await db.update(contacts)
          .set({
            name: existing.name || newContact.name,
            company: existing.company || newContact.company,
            title: existing.title || newContact.title,
            phone: existing.phone || newContact.phone,
            linkedinUrl: existing.linkedinUrl || newContact.linkedinUrl,
            twitterHandle: existing.twitterHandle || newContact.twitterHandle,
            lastInteractionAt: new Date(),
            updatedAt: new Date()
          })
          .where(eq(contacts.id, existing.id))
          .returning()
        
        merged.push(updated[0])
      } else {
        // Create new contact
        const created = await db.insert(contacts)
          .values({
            userId,
            ...newContact,
            firstSeenAt: newContact.firstSeen,
            lastInteractionAt: newContact.firstSeen,
            tags: this.inferTags(newContact)
          })
          .returning()
        
        merged.push(created[0])
        
        // Queue enrichment
        await jobQueue.enqueue({
          type: 'contact-enrichment',
          data: { contactId: created[0].id }
        })
      }
    }
    
    return merged
  }
  
  private inferTags(contact: ExtractedContact): string[] {
    const tags: string[] = []
    
    if (contact.metadata?.isEnterpriseEmail) tags.push('enterprise')
    if (contact.metadata?.isRoleEmail) tags.push('role-based')
    if (contact.company) tags.push('has-company')
    if (contact.linkedinUrl || contact.twitterHandle) tags.push('social-connected')
    
    return tags
  }
}
```

### 4. Interaction Tracking

```typescript
class InteractionTracker {
  async recordInteraction(
    userId: string, 
    email: ParsedEmail, 
    contacts: Contact[]
  ) {
    // Determine interaction type and direction
    const userEmail = await this.getUserEmail(userId)
    const isOutbound = email.from.email === userEmail
    
    for (const contact of contacts) {
      // Skip self
      if (contact.email === userEmail) continue
      
      // Record interaction
      await db.insert(interactions).values({
        userId,
        contactId: contact.id,
        type: 'email',
        emailId: email.id,
        direction: isOutbound ? 'outbound' : 'inbound',
        occurredAt: email.sentAt,
        metadata: {
          subject: email.subject,
          threadId: email.threadId,
          hasAttachments: email.hasAttachments
        }
      })
      
      // Update contact stats
      await this.updateContactStats(contact.id, isOutbound)
    }
  }
  
  private async updateContactStats(contactId: string, isOutbound: boolean) {
    // Increment counters
    const field = isOutbound ? 'totalEmailsSent' : 'totalEmailsReceived'
    const timeField = isOutbound ? 'lastEmailSentAt' : 'lastEmailReceivedAt'
    
    await db.update(contactStats)
      .set({
        [field]: sql`${field} + 1`,
        [timeField]: new Date(),
        updatedAt: new Date()
      })
      .where(eq(contactStats.contactId, contactId))
    
    // Queue recalculation of derived metrics
    await jobQueue.enqueue({
      type: 'calculate-contact-metrics',
      data: { contactId },
      delay: 60000 // 1 minute delay to batch updates
    })
  }
}
```

## Performance Optimizations

### 1. Batch Processing
```typescript
// Process emails in batches
const BATCH_SIZE = 50
const MAX_CONCURRENT = 5

async function processBatch(messageIds: string[]) {
  const chunks = chunk(messageIds, BATCH_SIZE)
  
  await pMap(chunks, async (chunk) => {
    const messages = await gmail.messages.batchGet({ ids: chunk })
    await Promise.all(messages.map(m => processEmail(m)))
  }, { concurrency: MAX_CONCURRENT })
}
```

### 2. Caching Strategy
```typescript
// Cache frequently accessed data
const contactCache = new LRUCache<string, Contact>({
  max: 10000,
  ttl: 1000 * 60 * 5 // 5 minutes
})

// Cache email threads
const threadCache = new LRUCache<string, EmailThread>({
  max: 1000,
  ttl: 1000 * 60 * 60 // 1 hour
})
```

### 3. Database Indexes
```sql
-- Optimize contact lookups
CREATE INDEX idx_contacts_user_email ON contacts(user_id, email);
CREATE INDEX idx_contacts_user_updated ON contacts(user_id, updated_at DESC);

-- Optimize email queries
CREATE INDEX idx_emails_user_thread ON emails(user_id, thread_id);
CREATE INDEX idx_emails_user_sent ON emails(user_id, sent_at DESC);

-- Optimize interaction queries
CREATE INDEX idx_interactions_contact_occurred ON interactions(contact_id, occurred_at DESC);
CREATE INDEX idx_interactions_user_type ON interactions(user_id, type, occurred_at DESC);
```

## Error Handling & Recovery

### 1. Sync Failure Recovery
```typescript
class SyncRecovery {
  async handleSyncFailure(userId: string, error: Error) {
    // Log error
    await logger.error('Sync failed', { userId, error })
    
    // Determine if retryable
    if (this.isRetryable(error)) {
      await jobQueue.enqueue({
        type: 'email-sync',
        data: { userId },
        attempts: 1,
        delay: this.calculateBackoff(1)
      })
    } else {
      // Notify user
      await notifyUser(userId, 'Email sync failed. Please reconnect your account.')
    }
  }
}
```

### 2. Partial Sync State
```typescript
// Track sync progress
interface SyncProgress {
  userId: string
  totalEmails: number
  processedEmails: number
  failedEmails: string[]
  lastProcessedId: string
  status: 'in_progress' | 'completed' | 'failed'
}

// Resume from last successful email
async function resumeSync(progress: SyncProgress) {
  const remaining = await getEmailsAfter(progress.lastProcessedId)
  await continueProcessing(remaining, progress)
}
```

## Monitoring & Metrics

```typescript
// Key metrics to track
interface SyncMetrics {
  syncDuration: Histogram
  emailsProcessed: Counter
  contactsExtracted: Counter
  extractionAccuracy: Gauge
  syncFailures: Counter
  apiQuotaUsage: Gauge
}

// Example metric collection
metrics.syncDuration.observe(
  { userId, syncType: 'incremental' },
  Date.now() - startTime
)
```