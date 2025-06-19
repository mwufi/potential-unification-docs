# Reusable Components Inventory

## Component Reusability Matrix

### From Zero (Mail) - Primary Source

#### 🟢 Direct Copy (Minimal Changes)
```typescript
// Authentication
- /lib/auth-client.ts - Better Auth client setup
- /lib/auth-proxy.ts - OAuth proxy implementation
- /hooks/use-connections.ts - Multi-account management
- /components/user/user-button.tsx - User profile UI

// UI Components
- /components/ui/* - All UI primitives
- /components/responsive-modal.tsx - Modal system
- /components/mail/attachments-accordion.tsx - File display
- /components/theme/* - Theme system
- /components/icons/* - Icon library

// Utilities
- /lib/utils.ts - Common utilities
- /lib/schemas.ts - Validation schemas
- /hooks/use-debounce.ts - Debounce hook
- /hooks/use-media-query.ts - Responsive hooks
```

#### 🟡 Adapt & Extend
```typescript
// Email → Communication System
- /hooks/use-threads.ts → use-communications.ts
- /components/mail/mail-list.tsx → communication-list.tsx
- /lib/email-utils.ts → communication-utils.ts

// Notes → CRM Features
- /hooks/use-notes.tsx → use-crm-notes.ts
- Note model → Extended for contacts/deals
- Note UI → CRM activity timeline

// AI Features
- /lib/ai-client.ts → Vercel AI SDK wrapper
- /components/create/ai-chat.tsx → AI assistant UI
- Writing style analysis → Communication insights

// Search
- Email search → Universal search
- /hooks/use-search-value.ts → Enhanced search
```

#### 🔴 Reference Only
```typescript
// Edge-specific
- Cloudflare Workers setup
- Cloudflare AI integration
- Hyperdrive connection

// Different Architecture
- React Router v7 setup
- Hono.js API structure
```

### From Analog (Calendar) - Calendar Features

#### 🟢 Direct Copy
```typescript
// Calendar Components
- /components/event-calendar/* - Entire calendar system
- /components/date-picker.tsx - Date selection
- /atoms/calendar-settings.ts - Calendar state

// Calendar Utilities
- /event-calendar/utils/date-time.ts - Date helpers
- /event-calendar/hooks/use-calendar-navigation.ts
- Calendar view components (day/week/month)

// Drag & Drop
- DnD context and providers
- Draggable event implementation
```

#### 🟡 Adapt & Extend
```typescript
// Event Management → Activity Tracking
- Event types → CRM activities
- Event creation → Activity logging
- Recurring events → Follow-up scheduling

// Calendar Integration
- Add email integration
- Add meeting insights
- Connect to contacts
```

### From Nimbus (Drive) - File Management

#### 🟢 Direct Copy
```typescript
// File Components
- Upload interfaces
- File browser UI
- Folder navigation

// Provider Patterns
- /providers/google/google-drive.ts - API patterns
- Multi-provider abstraction
- Error handling patterns
```

#### 🟡 Adapt & Extend
```typescript
// File Management → Document Tracking
- File metadata → Document relationships
- Upload flow → Attachment system
- Provider auth → Extended OAuth
```

## Shared Patterns Across All Three

### UI/UX Patterns
```typescript
// Common UI Elements
- Sidebar navigation (all three)
- Settings pages (Zero + Nimbus)
- Search interfaces (Zero + Nimbus)
- Modal systems (all three)
- Theme switching (all three)

// Interaction Patterns
- Bulk operations (Zero + Nimbus)
- Drag and drop (Analog + Zero)
- Real-time updates (Zero + Analog)
- Keyboard shortcuts (Zero)
```

### Technical Patterns
```typescript
// API Design
- tRPC usage (Zero + Analog)
- Error handling (all three)
- Loading states (all three)
- Optimistic updates (Zero)

// State Management  
- TanStack Query (Zero + Nimbus)
- Atom-based state (Analog)
- Context providers (all three)

// Infrastructure
- TypeScript patterns (all three)
- Monorepo structure (all three)
- Build configurations (all three)
```

## Specific File Recommendations

### Week 1-2: Foundation
**Copy these files directly:**
```
From Zero:
- /apps/mail/lib/auth-client.ts
- /apps/mail/lib/auth-proxy.ts
- /apps/mail/components/ui/*
- /apps/mail/hooks/use-connections.ts
- /apps/mail/providers/query-provider.tsx
- /apps/server/src/db/schema.ts (auth tables)
- /apps/server/src/trpc/trpc.ts

From Analog:
- /apps/web/src/lib/utils.ts
- /apps/web/components.json
- /apps/web/src/components/ui/* (fill gaps)
```

### Week 3-4: Core Features
**Adapt these components:**
```
From Zero:
- Email sync → Contact sync
- Thread display → Contact timeline
- Notes system → CRM notes
- Categories → Tags/segments

From Analog:
- Calendar components → Meeting tracking
- Event creation → Activity creation
- Calendar sync → Full integration

From Nimbus:
- File upload → Document management
- Folder structure → Project organization
```

### Week 5-6: AI Integration
**Build upon Zero's patterns:**
```
- AI provider abstraction
- Streaming responses
- Tool/function calling
- Context management
- Prompt templates
```

## Migration Utilities Needed

### 1. Runtime Compatibility Layer
```typescript
// edge-compat.ts
export const createEdgeCompatibleAPI = (handler) => {
  // Convert Hono handlers to Next.js API routes
  // Handle environment differences
  // Manage request/response formats
}
```

### 2. Database Migration Scripts
```typescript
// migrate-zero-schema.ts
- Convert Zero schema to CRM schema
- Add contact, company, deal tables
- Extend existing tables
- Preserve relationships
```

### 3. Component Adapter
```typescript
// react-compat.ts
- React 19 → React 18 compatibility
- React Router → Next.js routing
- Handle hydration differences
```

## Time Estimates

### Direct Reuse: 2-3 days
- Copy UI components
- Set up authentication
- Configure build tools

### Adaptation: 5-7 days  
- Modify Zero components for CRM
- Integrate calendar from Analog
- Adapt file handling from Nimbus

### New Development: 10-15 days
- CRM-specific features
- Advanced AI capabilities
- Analytics and reporting
- Workflow automation

## Total Estimated Savings

By reusing components:
- **UI Components**: Save 1 week
- **Authentication**: Save 1 week  
- **Email Infrastructure**: Save 2 weeks
- **Calendar Features**: Save 1 week
- **AI Foundation**: Save 1 week

**Total: 6 weeks saved** vs building from scratch