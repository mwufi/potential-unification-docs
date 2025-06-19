# Testing Strategy & Quality Gates

## Overview

Comprehensive testing strategy ensuring reliability, performance, and maintainability across all phases of development.

## Testing Pyramid

```
         /\
        /  \      E2E Tests (10%)
       /    \     - Critical user journeys
      /      \    - Cross-browser testing
     /--------\   
    /          \  Integration Tests (30%)
   /            \ - API contract tests
  /              \- Service integration
 /----------------\
/                  \ Unit Tests (60%)
                     - Business logic
                     - Data transformations
                     - Utility functions
```

## Testing Phases by Development Stage

### Phase 1: Foundation Testing

#### Unit Tests
```typescript
// src/__tests__/services/email-sync.test.ts
describe('EmailSyncService', () => {
  describe('parseGmailMessage', () => {
    it('should extract all email fields correctly', () => {
      const gmailMessage = createMockGmailMessage()
      const parsed = parseGmailMessage(gmailMessage)
      
      expect(parsed).toMatchObject({
        messageId: expect.any(String),
        subject: expect.any(String),
        from: { email: expect.any(String), name: expect.any(String) },
        to: expect.arrayContaining([expect.any(String)]),
        bodyText: expect.any(String),
        internalDate: expect.any(Date),
      })
    })
    
    it('should handle missing optional fields', () => {
      const minimalMessage = { id: '123', threadId: '456' }
      const parsed = parseGmailMessage(minimalMessage)
      
      expect(parsed.cc).toEqual([])
      expect(parsed.attachments).toEqual([])
    })
  })
})
```

#### Integration Tests
```typescript
// src/__tests__/integration/gmail-sync.test.ts
describe('Gmail Sync Integration', () => {
  let container: PostgreSqlContainer
  let gmailMock: GmailAPIMock
  
  beforeAll(async () => {
    container = await createTestDatabase()
    gmailMock = new GmailAPIMock()
  })
  
  it('should sync emails from Gmail to database', async () => {
    // Setup
    const user = await createTestUser()
    const account = await createTestAccount(user.id)
    
    gmailMock.addMessages([
      createMockEmail({ subject: 'Test Email 1' }),
      createMockEmail({ subject: 'Test Email 2' }),
    ])
    
    // Execute
    const syncService = new EmailSyncService()
    const result = await syncService.syncAccount(account.id)
    
    // Verify
    expect(result.synced).toBe(2)
    expect(result.errors).toHaveLength(0)
    
    const emails = await db.query.emails.findMany({
      where: eq(emails.userId, user.id)
    })
    
    expect(emails).toHaveLength(2)
    expect(emails[0].subject).toBe('Test Email 1')
  })
})
```

### Phase 2: AI Testing

#### AI Response Testing
```typescript
// src/__tests__/ai/response-quality.test.ts
describe('AI Response Quality', () => {
  const testCases = [
    {
      query: 'Show me emails from John',
      expectedTools: ['searchEmails'],
      expectedParams: { from: 'John' },
    },
    {
      query: 'Draft a follow-up to my last email with Sarah',
      expectedTools: ['searchEmails', 'composeEmail'],
      minConfidence: 0.8,
    },
  ]
  
  testCases.forEach(({ query, expectedTools, expectedParams, minConfidence }) => {
    it(`should handle query: "${query}"`, async () => {
      const response = await aiService.query(query, { userId: testUser.id })
      
      // Check tool usage
      if (expectedTools) {
        const usedTools = response.toolCalls.map(c => c.tool)
        expect(usedTools).toEqual(expect.arrayContaining(expectedTools))
      }
      
      // Check parameters
      if (expectedParams) {
        const toolCall = response.toolCalls.find(c => c.tool === expectedTools[0])
        expect(toolCall.params).toMatchObject(expectedParams)
      }
      
      // Check confidence
      if (minConfidence) {
        expect(response.confidence).toBeGreaterThanOrEqual(minConfidence)
      }
    })
  })
})
```

#### Prompt Testing
```typescript
// src/__tests__/ai/prompts.test.ts
describe('Prompt Engineering Tests', () => {
  it('should maintain consistent persona across responses', async () => {
    const queries = [
      'Hello, how are you?',
      'Can you help me find some emails?',
      'Thanks for your help!',
    ]
    
    const responses = await Promise.all(
      queries.map(q => aiService.query(q, { userId: testUser.id }))
    )
    
    // Analyze tone consistency
    const toneAnalysis = responses.map(r => analyzeTone(r.content))
    const toneVariance = calculateVariance(toneAnalysis)
    
    expect(toneVariance).toBeLessThan(0.2) // Low variance = consistent
  })
})
```

### Phase 3: Agent Testing

#### Agent Behavior Testing
```typescript
// src/__tests__/agents/behaviors.test.ts
describe('Agent Behaviors', () => {
  describe('Proactive Behaviors', () => {
    it('should trigger dormant contact reactivation', async () => {
      // Create scenario
      await createTestContact(user.id, {
        name: 'John Doe',
        lastInteractionAt: subDays(new Date(), 45),
        relationshipStrength: 0.8,
      })
      
      // Run behavior
      const behavior = new DormantContactBehavior()
      await behavior.execute(user.id)
      
      // Check results
      const notifications = await getNotifications(user.id)
      expect(notifications).toContainEqual(
        expect.objectContaining({
          type: 'reactivation-suggestion',
          title: expect.stringContaining('John Doe'),
        })
      )
    })
  })
  
  describe('Agent Coordination', () => {
    it('should coordinate multiple agents for complex tasks', async () => {
      const task = {
        description: 'Research competitor and draft comparison email',
        requiredSkills: ['web-search', 'analyze-data', 'compose-email'],
      }
      
      const result = await agentCoordinator.executeTask(task)
      
      expect(result.steps).toHaveLength(3)
      expect(result.success).toBe(true)
      expect(result.output).toHaveProperty('email')
      expect(result.output).toHaveProperty('research')
    })
  })
})
```

## Test Data Management

### Test Data Factories
```typescript
// src/test/factories/index.ts
import { Factory } from 'fishery'
import { faker } from '@faker-js/faker'

export const userFactory = Factory.define<User>(() => ({
  id: faker.datatype.uuid(),
  email: faker.internet.email(),
  name: faker.name.fullName(),
  createdAt: faker.date.past(),
}))

export const emailFactory = Factory.define<Email>(({ associations }) => ({
  id: faker.datatype.uuid(),
  userId: associations.user?.id || faker.datatype.uuid(),
  messageId: faker.datatype.uuid(),
  subject: faker.lorem.sentence(),
  from: faker.internet.email(),
  bodyText: faker.lorem.paragraphs(3),
  internalDate: faker.date.recent(),
}))

export const contactFactory = Factory.define<Contact>(({ associations }) => ({
  id: faker.datatype.uuid(),
  userId: associations.user?.id || faker.datatype.uuid(),
  email: faker.internet.email(),
  name: faker.name.fullName(),
  relationshipStrength: faker.datatype.float({ min: 0, max: 1 }),
}))
```

### Scenario Builders
```typescript
// src/test/scenarios.ts
export async function createActiveUserScenario() {
  const user = await userFactory.create()
  
  // Create contacts with varying relationship strengths
  const contacts = await contactFactory.createList(20, {
    user,
    relationshipStrength: () => faker.datatype.float({ min: 0.3, max: 1 })
  })
  
  // Create emails with those contacts
  for (const contact of contacts) {
    const emailCount = Math.floor(contact.relationshipStrength * 10)
    await emailFactory.createList(emailCount, {
      user,
      from: contact.email,
    })
  }
  
  return { user, contacts }
}
```

## Performance Testing

### Load Testing
```typescript
// src/__tests__/performance/load.test.ts
import autocannon from 'autocannon'

describe('Load Tests', () => {
  it('should handle 100 concurrent email syncs', async () => {
    const result = await autocannon({
      url: 'http://localhost:3000/api/trpc',
      connections: 100,
      duration: 30,
      requests: [
        {
          method: 'POST',
          path: '/email.sync',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({ userId: testUser.id }),
        }
      ]
    })
    
    expect(result.errors).toBe(0)
    expect(result.timeouts).toBe(0)
    expect(result.requests.average).toBeGreaterThan(10) // 10+ req/sec
  })
})
```

### Query Performance
```typescript
// src/__tests__/performance/queries.test.ts
describe('Query Performance', () => {
  beforeAll(async () => {
    // Create large dataset
    await seedDatabase({
      users: 10,
      emailsPerUser: 10000,
      contactsPerUser: 1000,
    })
  })
  
  it('should search emails in < 100ms', async () => {
    const start = performance.now()
    
    const results = await emailService.search({
      userId: testUser.id,
      query: 'meeting tomorrow',
      limit: 50,
    })
    
    const duration = performance.now() - start
    
    expect(duration).toBeLessThan(100)
    expect(results.length).toBeGreaterThan(0)
  })
})
```

## Security Testing

### Authentication Tests
```typescript
// src/__tests__/security/auth.test.ts
describe('Authentication Security', () => {
  it('should reject invalid tokens', async () => {
    const response = await request(app)
      .get('/api/emails')
      .set('Authorization', 'Bearer invalid-token')
    
    expect(response.status).toBe(401)
  })
  
  it('should prevent access to other users data', async () => {
    const user1 = await createTestUser()
    const user2 = await createTestUser()
    const user2Email = await emailFactory.create({ user: user2 })
    
    const response = await authenticatedRequest(user1)
      .get(`/api/emails/${user2Email.id}`)
    
    expect(response.status).toBe(404) // Not found, not forbidden
  })
})
```

### Input Validation
```typescript
// src/__tests__/security/validation.test.ts
describe('Input Validation', () => {
  const injectionTests = [
    { field: 'email', value: 'test@example.com; DROP TABLE users;--' },
    { field: 'name', value: '<script>alert("XSS")</script>' },
    { field: 'query', value: "' OR '1'='1" },
  ]
  
  injectionTests.forEach(({ field, value }) => {
    it(`should sanitize ${field} field`, async () => {
      const response = await request(app)
        .post('/api/contacts')
        .send({ [field]: value })
      
      expect(response.status).not.toBe(500)
      
      if (response.status === 200) {
        expect(response.body[field]).not.toContain('script')
        expect(response.body[field]).not.toContain('DROP')
      }
    })
  })
})
```

## Continuous Integration

### GitHub Actions Workflow
```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Bun
        uses: oven-sh/setup-bun@v1
        
      - name: Install dependencies
        run: bun install
        
      - name: Run unit tests
        run: bun test:unit
        
      - name: Run integration tests
        run: bun test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          
      - name: Run E2E tests
        run: bun test:e2e
        
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## Quality Gates

### Pre-Commit Hooks
```json
// .husky/pre-commit
{
  "hooks": {
    "pre-commit": "lint-staged",
    "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
  }
}

// lint-staged.config.js
module.exports = {
  '*.{ts,tsx}': [
    'eslint --fix',
    'prettier --write',
    'bun test --findRelatedTests',
  ],
}
```

### Pull Request Checks
Required checks before merge:
- ✅ All tests passing
- ✅ Code coverage > 80%
- ✅ No TypeScript errors
- ✅ No ESLint errors
- ✅ Performance benchmarks pass
- ✅ Security scan clean

### Definition of Done
A feature is considered complete when:
1. **Code Complete**
   - Feature implemented
   - Unit tests written (>80% coverage)
   - Integration tests written
   - Code reviewed

2. **Quality Assured**
   - Manual testing passed
   - Performance validated
   - Security reviewed
   - Accessibility checked

3. **Documentation**
   - API documented
   - User guide updated
   - Changelog updated

## Test Environments

### Local Development
```bash
# Run all tests
bun test

# Run specific test suites
bun test:unit
bun test:integration
bun test:e2e

# Run in watch mode
bun test --watch

# Run with coverage
bun test --coverage
```

### CI Environment
- Runs on every commit
- Parallel test execution
- Automatic retries for flaky tests
- Test result artifacts

### Staging Environment
- Mirrors production
- Smoke tests after deployment
- Performance monitoring
- Weekly security scans

## Monitoring & Alerts

### Test Metrics Dashboard
Track key metrics:
- Test execution time trends
- Flaky test identification
- Coverage trends
- Test failure patterns

### Alerts
- Test suite fails on main branch
- Coverage drops below 80%
- Performance regression detected
- Security vulnerability found

## Best Practices

### Writing Good Tests
1. **Descriptive names**: Test names should explain what and why
2. **Arrange-Act-Assert**: Clear test structure
3. **Independent tests**: No shared state between tests
4. **Fast tests**: Mock external dependencies
5. **Deterministic**: Same result every time

### Test Maintenance
1. **Regular cleanup**: Remove obsolete tests
2. **Refactor tests**: Keep tests DRY
3. **Update scenarios**: Match real-world usage
4. **Performance tuning**: Optimize slow tests

## Conclusion

This comprehensive testing strategy ensures:
- **Reliability**: Catch bugs before production
- **Confidence**: Safe refactoring and updates
- **Performance**: Maintain speed at scale
- **Security**: Protect user data
- **Quality**: Consistent user experience

By following these guidelines, the team can maintain high quality while moving fast.