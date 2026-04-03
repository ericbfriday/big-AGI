# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

```bash
# Development
npm run dev                           # Start dev server (Next.js + Turbopack)
npm run build                         # Production build
npm run start                         # Start production server

# Code Quality (safe while dev server runs)
npx tsc --noEmit                      # Type check without building
npx eslint src/path/to/file.ts        # Lint specific file
npm run lint                          # Lint entire project

# Database (requires POSTGRES_PRISMA_URL)
npm run db:push                       # Push Prisma schema to database
npm run db:studio                     # Open Prisma Studio GUI
```

## Architecture Overview

Big-AGI is a Next.js 15 application with a modular architecture for advanced AI interactions. The codebase follows a three-layer structure with distinct separation of concerns.

### Core Directory Structure

```
app/api/               # Next.js App Router (API routes only)
  edge/[trpc]/         #   Edge runtime tRPC endpoint
  cloud/[trpc]/        #   Node.js runtime tRPC endpoint
pages/                 # Next.js Pages Router (file-based routing -> src/apps/)
src/
  apps/                # Feature applications (self-contained modules)
  modules/             # Reusable business logic and integrations
  common/              # Shared infrastructure, stores, layout, utilities
  server/              # Backend: tRPC routers, Prisma, environment config
kb/                    # Knowledge base documentation (see @kb/KB.md)
tools/                 # Development tooling and scripts
```

### Key Technologies

- **Frontend**: Next.js 15, React 18, MUI Joy (beta), Emotion CSS-in-JS
- **State Management**: Zustand 5 with localStorage/IndexedDB persistence
- **API Layer**: tRPC 11 with React Query for type-safe client-server communication
- **Database**: Prisma 5 with PostgreSQL (optional, for link sharing)
- **Validation**: Zod 4 for runtime schema validation
- **Runtime**: Edge Runtime for AI streaming, Node.js for data processing
- **Analytics**: PostHog (optional)

### Path Aliases (tsconfig)

```
~/common/*  -> src/common/*
~/modules/* -> src/modules/*
~/server/*  -> src/server/*
```

## Apps (`src/apps/`)

Each app is a self-contained feature module with a main `App*.tsx` component. Some have local state stores (`store-app-*.ts`).

**Apps**: `beam/`, `call/`, `chat/`, `diff/`, `draw/`, `link-chat/`, `news/`, `personas/`, `settings-modal/`, `tokens/`

Pages in `/pages/` map to these apps (e.g., `pages/index.tsx` -> `AppChat`, `pages/draw.tsx` -> `AppDraw`).

## Modules (`src/modules/`)

Modules provide reusable business logic and integrations:

| Module | Purpose |
|--------|---------|
| `aix/` | AI communication framework - streaming, provider abstraction |
| `beam/` | Multi-model reasoning (scatter/gather pattern) |
| `blocks/` | Content rendering (markdown, code, images, etc.) |
| `llms/` | Language model abstraction - 17 vendor integrations |
| `aifn/` | AI functions (code fixup, image captioning, follow-ups) |
| `browse/` | Web scraping and content extraction |
| `dblobs/` | Binary large object storage |
| `elevenlabs/` | Text-to-speech integration |
| `google/` | Google Search integration |
| `persona/` | Persona/character system |
| `t2i/` | Text-to-image generation |
| `trade/` | Import/export (ChatGPT, markdown, JSON) |
| `youtube/` | YouTube transcript extraction |
| `3rdparty/` | Third-party integrations |
| `backend/` | Backend utility services |

## Key Subsystems

### AIX - AI Communication (`src/modules/aix/`)

Client-server streaming architecture with provider abstraction:
- **Client** -> tRPC -> **Server** -> **AI Providers**
- Particle-based streaming: `AixWire_Particles` -> `ContentReassembler` -> `DMessage`
- Provider-agnostic through adapter pattern (OpenAI, Anthropic, Gemini protocols)
- Primary entry: `aixChatGenerateContent_DMessage()` in `aix.client.ts`

### Beam - Multi-Model Reasoning (`src/modules/beam/`)

Scatter/Gather pattern for parallel AI processing:
- **Scatter**: Multiple models (rays) process input in parallel
- **Gather**: Fusion algorithms combine outputs
- State managed via `store-beam_vanilla.ts` (vanilla Zustand, no React integration)
- BeamStore per conversation via `ConversationHandler`

### Conversation Management

- **`ConversationHandler`** (`src/common/chat-overlay/ConversationHandler.ts`): Singleton per conversation, orchestrates chat, beam, and ephemerals
- **Per-chat stores**: `store-perchat_vanilla.ts` with slices for composer, ephemerals, variform
- **Message structure**: `DMessage` -> `DMessageFragment[]`
- **Multi-pane support** with independent conversation states

### LLM Vendor System (`src/modules/llms/`)

17 vendors registered in `vendors.registry.ts`:
`alibaba`, `anthropic`, `azure`, `deepseek`, `googleai` (Gemini), `groq`, `lmstudio`, `localai`, `mistral`, `moonshot`, `ollama`, `openai`, `openpipe`, `openrouter`, `perplexity`, `togetherai`, `xai`

Each vendor implements the `IModelVendor` interface (`IModelVendor.ts`).

## State Management

### Global Stores (`src/common/stores/`)

| Store | Persistence | Purpose |
|-------|------------|---------|
| `chat/store-chats.ts` | IndexedDB | Conversations and messages |
| `llms/store-llms.ts` | localStorage | Model configurations |
| `store-ux-labs.ts` | localStorage | UI preferences and labs features |
| `store-ui.ts` | localStorage | General UI state |
| `store-client.ts` | localStorage | Client configuration |
| `folders/store-chat-folders.ts` | localStorage | Chat folder organization |
| `metrics/store-metrics.ts` | localStorage | Usage metrics |
| `workspace/store-client-workspace.ts` | localStorage | Workspace state |

### Per-Instance Stores (Vanilla Zustand)

- `store-beam_vanilla.ts`: Beam scatter/gather state (high-performance, no React)
- `store-perchat_vanilla.ts`: Chat overlay state with composer/ephemeral/variform slices

### Storage Patterns

- Stores use `createIDBPersistStorage()` for IndexedDB persistence (`src/common/util/idbUtils.ts`)
- Version-based migrations handle data structure changes
- Partialize/merge functions control what gets persisted
- Rehydration logic repairs and upgrades data on load

## Layout System ("Optima") (`src/common/layout/optima/`)

Responsive layout composition system:
- `OptimaLayout.tsx` - Main layout with desktop/mobile adaptation
- `bar/` - Top bar with dropdown
- `nav/` - Desktop/mobile navigation
- `drawer/` - Side drawer (conversation list)
- `panel/` - Side panel (settings, details)
- `portals/` - React portal management for flexible component placement
- `overlays/` - Modal/overlay system
- `scratchclip/` - Clipboard features

## Server Architecture (`src/server/`)

Split architecture with two tRPC routers:

### Edge Runtime (`trpc.router-edge.ts`)
Low-latency AI operations at `/api/edge`:
- `aixRouter` - AI streaming and communication
- `llmAnthropicRouter`, `llmGeminiRouter`, `llmOllamaRouter`, `llmOpenAIRouter` - Vendor integrations
- `elevenlabsRouter`, `googleSearchRouter`, `youtubeRouter` - External services
- `backendRouter` - Backend services

### Cloud Runtime (`trpc.router-cloud.ts`)
Node.js data processing at `/api/cloud`:
- `browseRouter` - Web scraping (Puppeteer-based)
- `tradeRouter` - Import/export functionality

### Database (Optional)
Prisma schema at `src/server/prisma/schema.prisma`:
- PostgreSQL with connection pooling
- Single model: `LinkStorage` for chat link sharing (visibility, voting, expiration)
- Not required for core functionality - app is local-first

### Environment Configuration
- Server env vars defined and validated in `src/server/env.ts`
- Full variable reference: `docs/environment-variables.md`
- HTTP Basic Auth via `middleware.ts` (optional)

## Security Considerations

- API keys stored client-side in localStorage (user-provided)
- Server-side API keys in environment variables only
- XSS protection through proper content escaping
- No credential transmission to third parties
- Optional HTTP Basic Auth middleware for deployment protection

## Common Development Tasks

### Adding a New LLM Vendor
1. Create vendor directory in `src/modules/llms/vendors/[vendor]/`
2. Implement `IModelVendor` interface (see `IModelVendor.ts`)
3. Register in `vendors.registry.ts` (add to `ModelVendorId` type and `MODEL_VENDOR_REGISTRY`)
4. Add server-side env vars to `src/server/env.ts` (if needed)

### Debugging Storage Issues
- Check IndexedDB: DevTools -> Application -> IndexedDB -> `app-chats`
- Monitor Zustand state via React DevTools
- Check migration logs in console during rehydration

## Knowledge Base

Architecture and system documentation is available in the `/kb/` knowledge base:

@kb/KB.md
