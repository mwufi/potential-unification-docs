# Tech Stack Analysis: Analog vs Nimbus

## Overview

This document compares the technical architectures and technology choices between Analog (Calendar app) and Nimbus (Drive/Files app).

## Analog (Calendar App)

### Core Framework & Build Tools
- **Monorepo**: Turborepo
- **Framework**: Next.js 15.3.2 with React 19
- **Package Manager**: Bun 1.2.13
- **Build Pipeline**: Turbo tasks for build, dev, lint, db operations

### Frontend Stack
- **Styling**: Tailwind CSS 4
- **UI Components**: Radix UI (dialogs, dropdowns, tooltips, etc.)
- **State Management**: Jotai
- **Date/Time**: date-fns, temporal-polyfill
- **Drag & Drop**: @dnd-kit/core
- **Animations**: Motion (formerly Framer Motion)
- **Forms**: React Hook Form with Zod resolvers
- **Hotkeys**: react-hotkeys-hook

### Backend & Data
- **API**: tRPC (v11)
- **Authentication**: Better Auth (v1.2.9-beta.8) with Google & Microsoft OAuth
- **Database**: PostgreSQL (inferred from DATABASE_URL)
- **Cache**: Upstash Redis
- **Query Client**: TanStack Query

### Development Tools
- **ESLint**: Custom config with Next.js support
- **Prettier**: With import sorting and Tailwind plugins
- **TypeScript**: Version 5

### Notable Features
- Temporal polyfill for advanced date/time handling
- Analytics integration (Simple Analytics)
- Docker compose for local development

## Nimbus (Drive/Files App)

### Core Framework & Build Tools
- **Monorepo**: Native Bun workspaces (no Turborepo)
- **Framework**: Next.js 15.2.4 with React 19 (frontend) + Hono (backend)
- **Package Manager**: Bun (latest)
- **Build Pipeline**: Direct bun commands with concurrently for parallel dev

### Frontend Stack
- **Styling**: Tailwind CSS 4
- **UI Components**: Radix UI (similar components as Analog)
- **State Management**: React Query (TanStack Query)
- **File Handling**: react-dropzone
- **Animations**: Motion
- **Forms**: React Hook Form with Zod
- **HTTP Client**: Axios

### Backend & Data
- **Backend Framework**: Hono (separate server app)
- **Authentication**: Better Auth (v1.2.8)
- **Database**: PostgreSQL with Drizzle ORM
- **Validation**: Zod with Hono validators
- **Email**: Resend

### Development Tools
- **ESLint**: Custom config with unused imports plugin
- **Prettier**: With import sorting and Tailwind plugins
- **TypeScript**: Version 5
- **Git Hooks**: Husky with lint-staged

### Notable Features
- Separate backend server (Hono) for API
- File upload capabilities with react-dropzone
- Rate limiting (next-rate-limit)
- Vercel Analytics integration

## Key Differences

1. **Architecture Pattern**:
   - Analog: Integrated Next.js app with tRPC
   - Nimbus: Separated frontend (Next.js) and backend (Hono)

2. **Build System**:
   - Analog: Turborepo for orchestration
   - Nimbus: Native Bun workspaces with concurrently

3. **API Layer**:
   - Analog: tRPC for type-safe APIs
   - Nimbus: REST APIs with Hono + Zod validation

4. **Database/ORM**:
   - Analog: Direct database access (ORM not visible yet)
   - Nimbus: Drizzle ORM

5. **State Management**:
   - Analog: Jotai (atomic state management)
   - Nimbus: React Query only

6. **Specialized Libraries**:
   - Analog: DnD Kit, temporal-polyfill, date-fns
   - Nimbus: react-dropzone, axios

## Common Technologies

- Next.js 15.x with React 19
- Better Auth for authentication
- PostgreSQL database
- Radix UI components
- Tailwind CSS 4
- React Hook Form + Zod
- Motion for animations
- TypeScript 5
- Bun as package manager