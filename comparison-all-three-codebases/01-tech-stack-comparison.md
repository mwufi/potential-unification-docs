# Tech Stack Comparison: Analog vs Nimbus vs Zero

## Overview
This document compares the technical stacks of our three codebases to identify reusable components for the AI CRM project.

## Tech Stack Matrix

| Component | Analog (Calendar) | Nimbus (Drive) | Zero (Mail) | AI CRM Vision |
|-----------|------------------|----------------|-------------|---------------|
| **Frontend Framework** | Next.js 14 | Next.js 14 | React 19 + React Router v7 | Next.js 14 |
| **UI Library** | shadcn/ui | Custom + shadcn patterns | Custom + shadcn patterns | shadcn/ui |
| **Styling** | Tailwind CSS | Tailwind CSS | Tailwind CSS | Tailwind CSS |
| **State Management** | Jotai (atoms) | TanStack Query | TanStack Query | TanStack Query + Zustand |
| **Build Tool** | Next.js bundler | Next.js bundler | Vite | Next.js bundler |
| **Type Safety** | TypeScript | TypeScript | TypeScript | TypeScript |
| **API Layer** | tRPC | REST API | tRPC | tRPC |
| **Database** | Not specified | Not specified | PostgreSQL + Drizzle | PostgreSQL + Drizzle |
| **Authentication** | Next-Auth implied | Custom auth | Better Auth | Better Auth |
| **Deployment** | Vercel likely | Vercel likely | Cloudflare Workers | Vercel |
| **Package Manager** | Bun | Bun | pnpm | Bun |

## Backend Architecture

### Analog
- Minimal backend (calendar-focused)
- tRPC for API calls
- No database layer visible

### Nimbus
- Express-based server
- Custom Google Drive integration
- REST API architecture
- File storage abstraction

### Zero
- **Hono.js on Cloudflare Workers** (edge-first)
- **PostgreSQL with Drizzle ORM**
- **tRPC for type-safe APIs**
- **Better Auth for authentication**
- **Background jobs with Cloudflare Queues**
- **Redis caching with Upstash**

### AI CRM Vision
- Next.js API routes
- PostgreSQL with Drizzle ORM
- tRPC for type-safe APIs
- Better Auth
- pg-boss for background jobs
- Redis for caching

## AI/LLM Integration

### Analog
- No AI features

### Nimbus
- No AI features

### Zero
- **Multiple AI providers**: OpenAI, Google AI, Perplexity, Groq
- **Cloudflare AI** for edge inference
- **Model Context Protocol (MCP)** for extensible agents
- **Embeddings with Cloudflare Vectorize**
- AI features: summarization, categorization, writing assistance

### AI CRM Vision
- Vercel AI SDK (OpenAI/Anthropic)
- pgvector for embeddings
- RAG implementation
- Agent framework with skills

## Key Architectural Patterns

### Analog
- Component-driven architecture
- Atomic state management (Jotai)
- Calendar-specific optimizations
- Real-time updates

### Nimbus
- Service-oriented architecture
- Provider abstraction (Google Drive, OneDrive)
- File management patterns
- Upload/download optimization

### Zero
- **Edge-first architecture**
- **Monorepo with clear separation**
- **Event-driven with queues**
- **Multi-tenant with account linking**
- **AI-native design patterns**

### AI CRM Vision
- Composable architecture
- Event sourcing
- AI-native data structures
- Progressive enhancement

## Reusability Analysis

### High Reusability from Zero
1. **Database schema patterns** (user, auth, settings)
2. **Better Auth implementation** (OAuth flows)
3. **tRPC setup and patterns**
4. **AI integration patterns**
5. **Notes system** (foundation for CRM features)
6. **Background job patterns**

### High Reusability from Analog
1. **Calendar components and logic**
2. **Event management patterns**
3. **UI components** (shadcn/ui implementations)
4. **Real-time update patterns**
5. **Date/time utilities**

### High Reusability from Nimbus
1. **File upload/management components**
2. **Provider abstraction patterns**
3. **Storage service interfaces**
4. **Multi-source integration patterns**

## Infrastructure Differences

### Zero's Edge-First vs AI CRM's Traditional
- Zero uses Cloudflare Workers (edge runtime)
- AI CRM targets Vercel/traditional Node.js
- Need to adapt edge patterns to Node.js environment

### Database Approach
- Zero has existing PostgreSQL + Drizzle setup
- Directly applicable to AI CRM vision
- Schema can be extended for CRM features

### AI Infrastructure
- Zero's MCP integration is unique
- AI CRM uses Vercel AI SDK
- Can learn from Zero's multi-provider approach