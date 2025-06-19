# Compatibility and Shared Features Analysis

## Overview

Analog (Calendar) and Nimbus (Drive) share significant architectural patterns, technologies, and implementation approaches, making them highly compatible for potential unification.

## Shared Technical Foundation

### Core Technologies (Identical)
- **Frontend Framework**: Next.js 15.x with React 19
- **Package Manager**: Bun
- **Authentication**: Better Auth
- **Database**: PostgreSQL
- **UI Library**: shadcn/ui components
- **Styling**: Tailwind CSS 4
- **Form Handling**: React Hook Form + Zod
- **Animations**: Motion (formerly Framer Motion)
- **TypeScript**: Version 5

### Architectural Patterns
1. **Monorepo Structure**: Both use workspace-based monorepos
2. **API-First Design**: Direct integration with Google APIs
3. **Sync-less Architecture**: No local data storage, real-time API calls
4. **Provider Pattern**: Service classes wrapping Google APIs
5. **Type Safety**: Strong TypeScript types throughout

## Shared UI Components

### Identical Components (15+)
- animated-group, avatar, button, card, checkbox
- collapsible, dialog, dropdown-menu, input, label
- separator, sheet, sidebar, skeleton, tooltip

### Component Architecture
- Both use shadcn/ui as base
- Same variant system (CVA)
- Identical theming approach (CSS variables)
- Shared Radix UI primitives
- Same Lucide icons

## Authentication & User Management

### Identical Implementation
```typescript
// Same schema in both apps
account table:
- id, accountId, providerId
- userId (reference)
- accessToken, refreshToken, idToken
- accessTokenExpiresAt, refreshTokenExpiresAt
- scope

user table:
- id, name, email, emailVerified
- image, createdAt, updatedAt

session table:
- id, expiresAt, token
- userId, ipAddress, userAgent
```

### OAuth Providers
- Both support Google OAuth
- Analog adds Microsoft OAuth
- Same token refresh mechanism
- Identical session management

## Shared Design Patterns

### 1. Provider Classes
```typescript
// Analog
class GoogleCalendarProvider {
  constructor({ accessToken })
  async calendars()
  async events()
}

// Nimbus
class GoogleDriveProvider {
  constructor(accessToken)
  async listFiles()
  async createFile()
}
```

### 2. Error Handling
- Both wrap API calls in error handlers
- Similar error propagation patterns
- Consistent error types

### 3. State Management
- Client-side state with React Query
- Server state through API calls
- No complex state synchronization

## Key Differences

### 1. Build Systems
- **Analog**: Turborepo for orchestration
- **Nimbus**: Native Bun workspaces

### 2. API Architecture
- **Analog**: tRPC for type-safe APIs
- **Nimbus**: REST with Hono backend

### 3. State Libraries
- **Analog**: Jotai for atomic state
- **Nimbus**: React Query only

### 4. Specialized Features
- **Analog**: Calendar-specific (DnD Kit, temporal-polyfill)
- **Nimbus**: File-specific (react-dropzone, axios)

## Unification Potential

### High Compatibility Areas

1. **Shared Component Library**
   - Extract common UI components
   - Create @fluffymail/ui package
   - Maintain consistent design system

2. **Authentication Package**
   - Unified @fluffymail/auth
   - Support multiple providers
   - Shared session management

3. **Google Integration Core**
   - Base provider class
   - Token management
   - Error handling utilities

4. **Database Schema**
   - Unified user/account tables
   - Shared auth infrastructure
   - Common patterns for OAuth

### Unification Strategy

#### Phase 1: Shared Packages
```
packages/
  ui/          # Shared components
  auth/        # Authentication logic
  db/          # Database schemas
  google-core/ # Base Google integration
```

#### Phase 2: Unified API Layer
- Standardize on one approach (tRPC or REST)
- Create shared API utilities
- Common error handling

#### Phase 3: Merged Application
```
apps/
  fluffymail/  # Unified app
    features/
      calendar/  # From Analog
      drive/     # From Nimbus
      shared/    # Common features
```

### Benefits of Unification

1. **Code Reuse**: 70%+ shared code
2. **Maintenance**: Single codebase
3. **Consistency**: Unified UX
4. **Features**: Cross-product integration
5. **Performance**: Shared caching/optimization

### Challenges

1. **API Layer**: Different approaches need reconciliation
2. **Build System**: Choose between Turborepo vs native workspaces
3. **State Management**: Reconcile Jotai vs React Query only
4. **Feature Parity**: Complete missing features before merge

## Recommendation

The codebases are highly compatible and well-suited for unification. The shared foundation (Next.js, Better Auth, shadcn/ui) provides an excellent base. The main work would be:

1. Standardizing the API layer
2. Creating shared packages
3. Merging domain-specific features
4. Completing feature parity

Estimated effort: 2-4 weeks for a complete unification with proper testing.