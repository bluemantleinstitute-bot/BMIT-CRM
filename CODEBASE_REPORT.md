# WACRM Codebase Analysis Report

**Project:** wacrm v0.8.0
**Author:** Arnas Donauskas
**License:** MIT
**Generated:** 2026-07-31

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Technology Stack](#2-technology-stack)
3. [Directory Structure](#3-directory-structure)
4. [Application Architecture](#4-application-architecture)
5. [Database Schema](#5-database-schema)
6. [API Routes](#6-api-routes)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [Real-time Features](#8-real-time-features)
9. [AI Integration](#9-ai-integration)
10. [MCP Server](#10-mcp-server)
11. [Security Analysis](#11-security-analysis)
12. [Testing](#12-testing)
13. [Configuration & Build](#13-configuration--build)
14. [Key Metrics](#14-key-metrics)

---

## 1. Project Overview

WACRM is a **self-hostable WhatsApp CRM template** built on Next.js and Supabase. It provides a shared inbox, contact management, sales pipelines, broadcast messaging, no-code automations, conversational flows (chatbot builder), and an AI reply assistant — all integrated with the Meta WhatsApp Business API.

### Core Features

- **Shared Inbox** — Real-time WhatsApp conversation management with message threading, reactions, reply-with-quote, and quick replies
- **Contact Management** — CRUD with tags, custom fields, CSV import, bulk operations, and deduplication
- **Sales Pipelines** — Kanban-style deal boards with stages, drag-and-drop, analytics, and deal value tracking
- **Broadcast Messaging** — Template-based mass messaging with audience segmentation (tags, custom fields, CSV), scheduling, and delivery tracking
- **No-Code Automations** — Trigger-action workflows with branching conditions, delays, webhook actions, and execution logs
- **Conversational Flows** — Visual chatbot builder with node-based graph editor, keyword triggers, and run history
- **AI Assistant** — BYO provider key (OpenAI/Anthropic), knowledge base with hybrid RAG retrieval (FTS + pgvector), auto-reply bot, draft generation, per-conversation caps, and handoff to human agents
- **Multi-User Accounts** — Role-based access control (owner > admin > agent > viewer), team invitations, and ownership transfer
- **Public REST API** — API-key authenticated `/api/v1` endpoints for contacts, conversations, messages, broadcasts, and webhooks
- **MCP Server** — Model Context Protocol server for driving the CRM from Claude, Cursor, and other AI clients
- **Internationalization** — Full i18n via next-intl with locale-based dictionary loading

---

## 2. Technology Stack

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| Next.js | 16.2.6 | App Router, server/client components, API routes |
| React | 19.2.4 | UI framework |
| TypeScript | ^6.0 | Type safety |
| Tailwind CSS | v4 | Utility-first styling (via @tailwindcss/postcss) |
| shadcn/ui | base-nova style | Component library (24 UI primitives) |
| Recharts | ^3.8.1 | Dashboard charts |
| @xyflow/react | ^12.11.0 | Flow visual editor (node graph) |
| @dnd-kit | ^6.3.1 / ^10.0.0 | Drag-and-drop (pipeline board, flow builder) |
| lucide-react | ^1.22.0 | Icon library |
| sonner | ^2.0.7 | Toast notifications |
| next-intl | ^4.13.1 | Internationalization |
| opus-recorder | ^8.0.5 | Audio recording (Ogg/Opus) |
| date-fns | ^4.4.0 | Date formatting |
| class-variance-authority | ^0.7.1 | Variant-based class composition |

### Backend
| Technology | Version | Purpose |
|---|---|---|
| Supabase SSR | ^0.12.0 | Server-side auth with cookie handling |
| Supabase JS | ^2.107.0 | Database, auth, storage, realtime client |
| PostgreSQL | (via Supabase) | Primary database with RLS |
| pgvector | (via Supabase) | Vector embeddings for semantic search |
| AES-256-GCM | (built-in) | Encryption for sensitive tokens |

### DevOps & Quality
| Technology | Version | Purpose |
|---|---|---|
| Vitest | ^4.1.9 | Unit testing |
| ESLint | ^9 | Linting (next/core-web-vitals + typescript) |
| Prettier | ^3.9.1 | Code formatting (with tailwindcss plugin) |
| @dagrejs/dagre | ^3.0.0 | Graph layout algorithm for flow editor |

### MCP Server
| Technology | Version | Purpose |
|---|---|---|
| @modelcontextprotocol/sdk | ^1.18.0 | MCP protocol implementation |
| zod | ^3.25.0 | Schema validation for tool inputs |

---

## 3. Directory Structure

```
CRM/
├── .github/                    # GitHub config (CI, templates, security policy)
├── docs/                       # Documentation (public-api.md, mcp.md)
├── messages/                   # i18n dictionaries (en.json)
├── mcp-server/                 # Standalone MCP server package
│   └── src/                    # config, client, tools (read/write/broadcast)
├── public/                     # Static assets (SVGs, opus encoder)
├── src/
│   ├── app/                    # Next.js App Router (70 files)
│   │   ├── (auth)/             # Auth pages (login, signup, forgot-password)
│   │   ├── (dashboard)/        # Authenticated pages (16 pages + 4 layouts)
│   │   ├── api/                # API route handlers (42 endpoints + 1 test)
│   │   ├── join/               # Invitation redemption (hybrid auth)
│   │   ├── globals.css         # Global Tailwind styles
│   │   ├── layout.tsx          # Root layout (i18n, theme, boot script)
│   │   └── page.tsx            # Root redirect to /dashboard
│   ├── components/             # React components (109 files)
│   │   ├── agents/             # AI playground, usage
│   │   ├── auth/               # Role-based gating
│   │   ├── automations/        # Automation builder
│   │   ├── broadcasts/         # 4-step wizard
│   │   ├── contacts/           # Contact forms, import, custom fields
│   │   ├── dashboard/          # Metrics, charts, activity feed
│   │   ├── flows/              # Flow editor (11 files)
│   │   ├── inbox/              # Conversation list, message thread (11 files)
│   │   ├── interactive/        # Interactive message builder/preview
│   │   ├── layout/             # Sidebar, header, mode toggle
│   │   ├── pipelines/          # Kanban board, deal cards, analytics
│   │   ├── presence/           # Online/offline indicators
│   │   ├── settings/           # 23 settings components
│   │   ├── tremor/             # Chart utilities
│   │   └── ui/                 # 24 shadcn/ui primitives
│   ├── hooks/                  # React hooks (8 files)
│   ├── i18n/                   # next-intl request config
│   ├── lib/                    # Business logic (110 files)
│   │   ├── account/            # Member management
│   │   ├── ai/                 # AI engine (24 files, 12 test files)
│   │   ├── api/                # Public API helpers (7 files)
│   │   ├── api-keys/           # Key hashing, scopes
│   │   ├── auth/               # Account, roles, invitations
│   │   ├── automations/        # Engine, triggers, templates
│   │   ├── contacts/           # Dedup, CSV parsing
│   │   ├── dashboard/          # Queries, date utils
│   │   ├── flows/              # Engine, validation, templates (14 files)
│   │   ├── inbox/              # Conversation queries
│   │   ├── storage/            # Media upload
│   │   ├── supabase/           # Client (browser) + server factories
│   │   ├── webhooks/           # Delivery, signing, SSRF protection
│   │   └── whatsapp/           # Meta API, encryption, templates (33 files)
│   ├── middleware.ts           # Auth gating + cookie refresh
│   └── types/                  # TypeScript type definitions (648 lines)
├── supabase/
│   └── migrations/             # 35 SQL migrations
├── next.config.ts              # Next.js 16 + next-intl + security headers
├── tsconfig.json               # TypeScript config (strict, @/* alias)
├── vitest.config.ts            # Test runner config
├── eslint.config.mjs           # ESLint flat config
├── postcss.config.mjs          # Tailwind v4 PostCSS
├── components.json             # shadcn/ui config
├── package.json                # Project manifest
├── .env.local.example          # Environment variable template
├── AGENTS.md                   # AI agent instructions
├── CHANGELOG.md                # Version history (v0.1.0 → v0.8.0)
├── CONTRIBUTING.md             # Contribution guide (template repo)
├── SECURITY_AUDIT_REPORT.md    # Security audit findings
└── README.md                   # Project overview
```

### File Counts

| Category | Count |
|---|---|
| **Pages (UI routes)** | 20 |
| **Layouts** | 4 |
| **API Route Handlers** | 42 |
| **API Test Files** | 1 |
| **Components** | 109 |
| **Hooks** | 8 |
| **Lib Modules** | 55 source + 55 test |
| **Migrations** | 35 |
| **MCP Server Files** | 8 source + 7 config |
| **Total Source Files** | ~390 |

---

## 4. Application Architecture

### Routing Structure

The app uses **Next.js App Router** with route groups for layout segmentation:

```
/ (root) → redirects to /dashboard
/login, /signup, /forgot-password  → (auth) layout — no sidebar
/dashboard, /inbox, /contacts, ... → (dashboard) layout — with Sidebar + Header + PresenceHeartbeat
/join/[token]                      → standalone layout — invitation redemption
```

### Component Architecture

- **Layouts** are server components exporting metadata (`robots: noindex`)
- **All pages** are client components (`"use client"`) using Supabase client-side SDK
- **DashboardShell** (`dashboard-shell.tsx:1`) is the authenticated wrapper:
  - Auth gating (redirect to `/login` if unauthenticated)
  - Renders `<AuthProvider>` → `<Sidebar>` → `<Header>` → `<main>{children}</main>`
  - Includes `<PresenceHeartbeat>` for online/away status

### Data Flow

1. **Client-side fetch**: Pages use `createClient()` (browser Supabase client) to query data directly
2. **API routes**: Server-side handlers use `createClient()` (server Supabase client with cookies) or service-role client for privileged operations
3. **Real-time**: Supabase Realtime channels subscribe to `postgres_changes` on `messages`, `conversations`, `member_presence`, and `notifications` tables
4. **Middleware**: Handles auth cookie refresh, redirects, and protects `/api/whatsapp/*` routes

### State Management

- **React Context**: `AuthContext` (user/profile/account/role), `ThemeContext` (theme/mode)
- **Custom Hooks**: `useAuth`, `useCan`, `usePresence`, `useRealtime`, `useTotalUnread`, `useUnreadNotifications`, `useTheme`
- **Local State**: Each page manages its own data fetching with `useState`/`useEffect` or `useCallback`
- **Optimistic Updates**: Used in contacts (bulk delete), pipelines (drag-drop), broadcasts (toggle), automations (active toggle)

---

## 5. Database Schema

### Core Tables (35 migrations)

#### Accounts & Users
| Table | Purpose | Key Columns |
|---|---|---|
| `accounts` | Multi-tenant workspace | `id`, `name`, `owner_user_id`, `default_currency` |
| `profiles` | User profile + role | `user_id`, `account_id`, `account_role` (owner/admin/agent/viewer), `beta_features[]` |

#### Contacts & Messaging
| Table | Purpose | Key Columns |
|---|---|---|
| `contacts` | Customer records | `phone`, `phone_normalized` (generated), `name`, `email`, `company`, `account_id` |
| `tags` | Contact labels | `name`, `color`, `account_id` |
| `contact_tags` | Many-to-many | `contact_id`, `tag_id` |
| `custom_fields` | User-defined fields | `field_name`, `field_type`, `account_id` |
| `contact_custom_values` | Field values | `contact_id`, `custom_field_id`, `value` |
| `contact_notes` | Free-text notes | `contact_id`, `note_text` |
| `conversations` | Chat threads | `status` (open/pending/closed), `assigned_agent_id`, `unread_count`, `ai_*` fields |
| `messages` | Individual messages | `sender_type`, `content_type`, `status`, `reply_to_message_id`, `interactive_payload`, `ai_generated` |
| `message_reactions` | Emoji reactions | `message_id`, `actor_type`, `emoji` |
| `quick_replies` | Reusable snippets | `title`, `kind` (text/interactive), `content_text`, `interactive_payload` |

#### WhatsApp Integration
| Table | Purpose | Key Columns |
|---|---|---|
| `whatsapp_config` | API credentials | `phone_number_id`, `waba_id`, `access_token` (encrypted), `verify_token` |
| `message_templates` | Meta templates | `name`, `language`, `category`, `status`, `meta_template_id`, `quality_score` |

#### Sales Pipeline
| Table | Purpose | Key Columns |
|---|---|---|
| `pipelines` | Named pipelines | `name`, `account_id` |
| `pipeline_stages` | Ordered stages | `name`, `position`, `color` |
| `deals` | Deal records | `title`, `value`, `status` (open/won/lost), `stage_id`, `contact_id` |

#### Broadcasting
| Table | Purpose | Key Columns |
|---|---|---|
| `broadcasts` | Campaign header | `template_name`, `status`, aggregate counts |
| `broadcast_recipients` | Per-recipient status | `contact_id`, `status`, `whatsapp_message_id` |

#### Automations
| Table | Purpose | Key Columns |
|---|---|---|
| `automations` | Workflow definitions | `trigger_type`, `trigger_config`, `is_active` |
| `automation_steps` | Ordered steps | `step_type`, `step_config`, `parent_step_id` (branching) |
| `automation_logs` | Execution audit | `status`, `steps_executed[]`, `trigger_event` |
| `automation_pending_executions` | Delay queue | `run_at`, `next_step_position`, `context` |

#### Conversational Flows (Chatbot)
| Table | Purpose | Key Columns |
|---|---|---|
| `flows` | Flow definitions | `trigger_type`, `entry_node_id`, `fallback_policy` |
| `flow_nodes` | Graph nodes | `node_type`, `node_key`, `config` (edges + params) |
| `flow_runs` | Runtime state machine | `status`, `vars` (collected values), UNIQUE on (account_id, contact_id) WHERE active |
| `flow_run_events` | Append-only audit | `event_type`, `node_key`, `payload` |

#### AI Assistant
| Table | Purpose | Key Columns |
|---|---|---|
| `ai_configs` | Provider config | `provider`, `model`, `api_key` (encrypted), `auto_reply_enabled` |
| `ai_knowledge_documents` | Knowledge base | `title`, `content` |
| `ai_knowledge_chunks` | RAG units | `fts` (tsvector), `embedding` (vector(1536)), HNSW index |
| `ai_usage_log` | Token tracking | `prompt_tokens`, `completion_tokens`, `total_tokens` |

#### API & Integrations
| Table | Purpose | Key Columns |
|---|---|---|
| `api_keys` | Bearer credentials | `key_hash` (SHA-256), `key_prefix`, `scopes[]`, `expires_at`, `revoked_at` |
| `webhook_endpoints` | Outbound events | `url`, `secret` (encrypted), `events[]`, `failure_count` |

#### Collaboration
| Table | Purpose | Key Columns |
|---|---|---|
| `account_invitations` | Invite tokens | `token_hash`, `role`, `expires_at` |
| `notifications` | In-app alerts | `type`, `conversation_id`, `read_at` |
| `member_presence` | Online/away | `status`, `last_seen_at` |

### Storage Buckets

| Bucket | Max Size | Access |
|---|---|---|
| `avatars` | 2 MB | Public reads, per-user writes |
| `flow-media` | 16 MB | Public reads, account-scoped writes |
| `chat-media` | 16 MB | Public reads, account-scoped writes |

### Key Database Functions (RPCs)

| Function | Security | Purpose |
|---|---|---|
| `handle_new_user()` | DEFINER | Creates account + profile on signup |
| `is_account_member()` | DEFINER | Checks membership + role hierarchy |
| `set_member_role()` | DEFINER | Admin changes teammate role |
| `remove_account_member()` | DEFINER | Removes member, creates fresh personal account |
| `transfer_account_ownership()` | DEFINER | Owner → admin demotion + target promotion |
| `peek_invitation()` | DEFINER | Anonymous invitation preview |
| `redeem_invitation()` | DEFINER | Joins account via invitation |
| `increment_automation_execution_count()` | DEFINER | Atomic counter (service_role only) |
| `increment_flow_execution_count()` | DEFINER | Atomic counter (service_role only) |
| `claim_ai_reply_slot()` | DEFINER | Atomic per-conversation AI reply cap |
| `match_ai_knowledge_fts()` | INVOKER | Lexical RAG retrieval |
| `match_ai_knowledge_semantic()` | INVOKER | Semantic RAG retrieval (pgvector) |
| `filter_contacts_by_tags()` | INVOKER | Server-side tag filtering with pagination |
| `touch_presence()` | DEFINER | Online/away heartbeat |
| `record_webhook_failure()` | DEFINER | Atomic consecutive-failure counter |
| `merge_duplicate_contacts()` | DEFINER | Phone deduplication merge |

---

## 6. API Routes

### Internal API (42 endpoints)

#### Account Management (`/api/account/*`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/account` | any member | Get account + role |
| PATCH | `/api/account` | admin+ | Rename account |
| GET | `/api/account/members` | any member | List members |
| PATCH | `/api/account/members/[userId]` | admin+ | Change member role |
| DELETE | `/api/account/members/[userId]` | admin+ | Remove member |
| GET | `/api/account/invitations` | admin+ | List invitations |
| POST | `/api/account/invitations` | admin+ | Create invitation |
| DELETE | `/api/account/invitations/[id]` | admin+ | Revoke invitation |
| GET | `/api/account/api-keys` | any member | List API keys |
| POST | `/api/account/api-keys` | admin+ | Mint API key |
| DELETE | `/api/account/api-keys/[id]` | admin+ | Revoke API key |
| POST | `/api/account/transfer-ownership` | owner | Transfer ownership |

#### WhatsApp Integration (`/api/whatsapp/*`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET/POST | `/api/whatsapp/webhook` | verify_token / HMAC | Meta inbound webhook |
| POST | `/api/whatsapp/send` | authenticated | Dashboard outbound send |
| POST | `/api/whatsapp/broadcast` | authenticated | Dashboard broadcast |
| POST | `/api/whatsapp/react` | authenticated | Emoji reaction |
| GET/POST | `/api/whatsapp/config` | authenticated | WhatsApp connection |
| GET | `/api/whatsapp/config/verify-registration` | authenticated | Diagnostic check |
| GET | `/api/whatsapp/media/[mediaId]` | authenticated | Proxy media download |
| PATCH/DELETE | `/api/whatsapp/templates/[id]` | authenticated | Edit/delete Meta template |
| POST | `/api/whatsapp/templates/sync` | authenticated | Sync Meta → local |
| POST | `/api/whatsapp/templates/submit` | authenticated | Submit to Meta |

#### AI Assistant (`/api/ai/*`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/ai/config` | any member | Read AI config |
| POST | `/api/ai/config` | admin+ | Save AI config |
| POST | `/api/ai/test` | admin+ | Validate AI credentials |
| POST | `/api/ai/draft` | agent+ | Generate draft reply |
| POST | `/api/ai/playground` | agent+ | Test-chat with AI |
| GET | `/api/ai/usage` | admin+ | Token usage stats |
| POST | `/api/ai/autoreply/[conversationId]` | agent+ | Toggle auto-reply |
| GET/POST | `/api/ai/knowledge` | varies | Knowledge base CRUD |
| POST | `/api/ai/knowledge/reindex` | admin+ | Re-embed all docs |
| GET/PATCH/DELETE | `/api/ai/knowledge/[id]` | varies | Document CRUD |

#### Automations (`/api/automations/*`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET/POST | `/api/automations` | varies | List/create |
| GET/PATCH/DELETE | `/api/automations/[id]` | varies | CRUD |
| POST | `/api/automations/[id]/duplicate` | agent+ | Clone automation |
| POST | `/api/automations/engine` | agent+ | Manual trigger |
| GET | `/api/automations/cron` | x-cron-secret | Drain pending queue |

#### Flows (`/api/flows/*`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET/POST | `/api/flows` | varies | List/create |
| GET/PUT/DELETE | `/api/flows/[id]` | varies | CRUD |
| POST | `/api/flows/[id]/activate` | agent+ | Toggle status |
| GET | `/api/flows/[id]/runs` | authenticated | Run history |
| GET | `/api/flows/cron` | x-cron-secret | Sweep abandoned runs |
| GET | `/api/flows/templates` | authenticated | Template gallery |

#### Invitations (`/api/invitations/*`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/invitations/[token]/peek` | public | Preview invitation |
| POST | `/api/invitations/[token]/redeem` | authenticated | Accept invitation |

#### Quick Replies (`/api/quick-replies/*`)
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET/POST | `/api/quick-replies` | varies | List/create |
| PATCH/DELETE | `/api/quick-replies/[id]` | agent+ | Update/delete |

### Public REST API v1 (`/api/v1/*`) — API-Key Authenticated

| Method | Endpoint | Scope | Description |
|---|---|---|---|
| GET | `/api/v1/me` | any key | Identity probe |
| GET/POST | `/api/v1/contacts` | read/write | Contact CRUD |
| GET/PATCH | `/api/v1/contacts/[id]` | read/write | Single contact |
| GET | `/api/v1/conversations` | read | List conversations |
| GET | `/api/v1/conversations/[id]` | read | Single conversation |
| GET | `/api/v1/conversations/[id]/messages` | read | List messages |
| POST | `/api/v1/messages` | send | Send message |
| POST | `/api/v1/broadcasts` | send | Launch broadcast |
| GET | `/api/v1/broadcasts/[id]` | send | Broadcast status |
| GET/POST | `/api/v1/webhooks` | manage | Webhook CRUD |
| GET/PATCH/DELETE | `/api/v1/webhooks/[id]` | manage | Single webhook |

---

## 7. Authentication & Authorization

### Authentication Flow

1. **Supabase Auth** — Email/password authentication with `signInWithPassword` / `signUp`
2. **Middleware** (`middleware.ts`) — Refreshes tokens on every request, redirects unauthenticated users from protected paths
3. **Client-side** — `AuthProvider` wraps the dashboard, manages auth state via `onAuthStateChange`
4. **Server-side** — `createServerClient` from `@supabase/ssr` reads cookies for API route handlers

### Role-Based Access Control

**Hierarchy:** `owner > admin > agent > viewer`

| Capability | owner | admin | agent | viewer |
|---|---|---|---|---|
| View dashboard | ✓ | ✓ | ✓ | ✓ |
| Send messages | ✓ | ✓ | ✓ | ✗ |
| Edit settings | ✓ | ✓ | ✗ | ✗ |
| Manage members | ✓ | ✓ | ✗ | ✗ |
| Delete account | ✓ | ✗ | ✗ | ✗ |
| Transfer ownership | ✓ | ✗ | ✗ | ✗ |

### API Key Authentication (Public API)

- SHA-256 hashed keys with `key_prefix` for display
- Scoped permissions: `contacts:read`, `contacts:write`, `conversations:read`, `messages:read`, `messages:send`, `broadcasts:send`, `webhooks:manage`
- Optional expiry (up to 365 days), revocation support

### Middleware Protection

- Protected pages: `/dashboard`, `/inbox`, `/contacts`, `/pipelines`, `/broadcasts`, `/automations`, `/settings`
- Protected API routes: `/api/whatsapp/*` (except `/webhook`)
- Cookie refresh: Copies rotated cookies to all responses to prevent session wedge (issue #288)

---

## 8. Real-time Features

Supabase Realtime is used for live updates on:

| Table | Usage |
|---|---|
| `messages` | New messages in inbox, delivery status updates |
| `conversations` | New conversations, unread count changes, status changes |
| `member_presence` | Online/away status for team members |
| `notifications` | New assignment notifications |
| `message_reactions` | Emoji reaction additions/removals |

### Hooks

- **`useRealtime`** — Generic subscription to `messages` and `conversations` postgres_changes
- **`usePresence`** — Tracks online/away status with periodic re-derivation (15s interval)
- **`useTotalUnread`** — Computes total unread conversation count across all conversations
- **`useUnreadNotifications`** — Tracks unread notification count

### Resync Mechanism

The inbox page (`inbox/page.tsx`) implements a resync mechanism:
- Bumps a `resyncToken` on WebSocket reconnect and tab visibility change
- Refetches missed events that occurred during disconnection
- Handles conversation hydration when realtime events arrive without contact data

---

## 9. AI Integration

### Provider Support

- **OpenAI** — GPT models via OpenAI API
- **Anthropic** — Claude models via Anthropic API
- BYO (Bring Your Own) API key, encrypted with AES-256-GCM

### Knowledge Base (RAG)

- **Hybrid retrieval**: PostgreSQL full-text search (`tsvector` with `'simple'` config) + pgvector semantic search (1536-dimensional embeddings)
- **HNSW index** for approximate nearest neighbor search
- **Document chunking**: Documents are split into chunks for retrieval
- **Reindex endpoint**: Re-embeds all documents after adding an embeddings key

### Auto-Reply Bot

- Per-conversation toggle with configurable max replies per conversation
- Handoff to human agent (assigns conversation, sets `ai_handoff_summary`)
- Take over/resume: Agents can pause the bot (claim conversation) or resume it (clear assignment, reset reply count)
- Atomic claim slot via `claim_ai_reply_slot()` RPC prevents race conditions

### Usage Tracking

- Token usage logged per LLM call (prompt_tokens, completion_tokens, total_tokens)
- Admin dashboard with daily aggregation, per-mode breakdown, zero-filled series

---

## 10. MCP Server

A standalone **Model Context Protocol** server (`mcp-server/`) that exposes the wacrm public API as MCP tools for AI clients (Claude Desktop, Cursor, etc.).

### Safety Model

1. **Read-only by default** — Write/broadcast tools are not registered unless `WACRM_ENABLE_WRITES` / `WACRM_ENABLE_BROADCASTS` are set
2. **API-key scopes** — Server-side enforcement regardless of tool visibility
3. **Explicit broadcast confirmation** — `send_broadcast` requires `confirm: true` and is marked `destructive`

### Available Tools

| Tool | Group | Scope | Description |
|---|---|---|---|
| `whoami` | read | any | Show account + scopes |
| `list_contacts` | read | contacts:read | List/search contacts |
| `get_contact` | read | contacts:read | Read one contact |
| `list_conversations` | read | conversations:read | List conversations |
| `get_conversation` | read | conversations:read | Read one conversation |
| `list_messages` | read | messages:read | List messages |
| `get_broadcast` | read | broadcasts:send | Poll broadcast status |
| `send_message` | write | messages:send | Send WhatsApp message |
| `create_contact` | write | contacts:write | Create/find contact |
| `update_contact` | write | contacts:write | Update contact |
| `send_broadcast` | broadcast | broadcasts:send | Launch broadcast (requires confirm) |

---

## 11. Security Analysis

### Security Audit Findings (July 25, 2026)

| Severity | Count | Summary |
|---|---|---|
| **CRITICAL** | 2 | No CSRF on state-changing endpoints; Production secrets in .env.local |
| **HIGH** | 4 | Middleware only protects /api/whatsapp/*; DB errors leaked to clients; CSP report-only; 16 npm vulnerabilities |
| **MEDIUM** | 8 | javascript: URL injection; cookie security flags; invitation URL host spoofing; PostgREST filter interpolation; cursor interpolation; Meta API error leakage; inconsistent cron auth |
| **LOW** | 11 | Legacy CBC decryption; ILIKE wildcard injection; client-side file validation; DNS rebinding; TOCTOU races; verify token timing; schema exposure; DRY_RUN in prod; console.log PII; mediaId sanitization; ENCRYPTION_KEY non-null assertion |
| **INFORMATIONAL** | 7 | No eval/exec, no server actions, no innerHTML, no postMessage issues |

### Positive Security Patterns

- AES-256-GCM encryption for sensitive tokens (WhatsApp access_token, AI api_key, webhook secrets)
- SHA-256 hashed API keys and invitation tokens
- Comprehensive Row Level Security (RLS) on every table
- Webhook HMAC-SHA256 signature verification
- SSRF protection on webhook delivery
- Security headers (HSTS, X-Frame-Options: DENY, CSP report-only)
- UUID validation on all path parameters
- Rate limiting (per-user, per-account, per-IP)
- Centralized RBAC with role hierarchy
- Constant-time comparisons for token verification
- Privilege escalation prevention via BEFORE UPDATE trigger on profiles

---

## 12. Testing

### Test Framework

- **Vitest** ^4.1.9 with `tsconfigPaths` for `@/*` alias resolution
- Environment: node
- Pattern: `src/**/*.test.ts` and `src/**/*.test.tsx`

### Test Coverage by Module

| Module | Test Files | Coverage |
|---|---|---|
| `src/lib/ai/` | 10 test files | auto-reply, chunk, config, context, embeddings, generate, handoff, knowledge, query, usage |
| `src/lib/whatsapp/` | 12 test files | broadcast-core, encryption, interactive, meta-api (3), phone-utils, resolve-conversation, send-message, template-components, template-header-handle, template-send-builder, template-status-normalize, template-validators, template-webhook, webhook-signature |
| `src/lib/api/v1/` | 3 test files | contacts, conversations, pagination |
| `src/lib/api-keys/` | 2 test files | keys, scopes |
| `src/lib/auth/` | 3 test files | account, api-context, invitations, roles |
| `src/lib/automations/` | 3 test files | engine, validate |
| `src/lib/contacts/` | 2 test files | dedupe, parse-contact-csv |
| `src/lib/flows/` | 5 test files | edges, engine, fallback, layout, validate |
| `src/lib/inbox/` | 1 test file | conversations |
| `src/lib/storage/` | 1 test file | upload-media |
| `src/lib/webhooks/` | 5 test files | deliver, endpoints, events, sign, ssrf |
| `src/lib/` (root) | 4 test files | broadcast-status, currency, presence, rate-limit |
| `src/app/api/whatsapp/send/` | 1 test file | route.test.ts |
| `src/components/flows/` | 2 test files | flow-editor-state, shared |
| `src/components/ui/` | 1 test file | dropdown-menu-group-label |
| `src/middleware.test.ts` | 1 test file | middleware |

### Total: 55 test files

---

## 13. Configuration & Build

### Environment Variables

**Required (app won't start without):**
- `NEXT_PUBLIC_SUPABASE_URL` — Supabase project URL
- `NEXT_PUBLIC_SUPABASE_ANON_KEY` — Supabase anonymous key
- `SUPABASE_SERVICE_ROLE_KEY` — Server-side only, bypasses RLS
- `ENCRYPTION_KEY` — 64 hex chars (32 bytes, AES-256-GCM)
- `META_APP_SECRET` — Meta App Secret for webhook HMAC

**Recommended:**
- `NEXT_PUBLIC_SITE_URL` — Canonical deployment URL
- `NEXT_PUBLIC_APP_LOCALE` — Default locale

**Optional:**
- `ALLOWED_INVITE_HOSTS` — Hostname allowlist for invitations
- `AUTOMATION_CRON_SECRET` — Shared secret for cron endpoints
- `META_APP_ID` — Required for image-header template submission
- `WHATSAPP_TEMPLATES_DRY_RUN` — Bypass Meta API in dev/CI
- `AI_REQUEST_TIMEOUT_MS` — AI provider timeout (default 30000)
- `AI_CONTEXT_MESSAGE_LIMIT` — Recent messages for AI context (default 20)

### Build & Dev Commands

```bash
npm run dev          # Next.js dev server
npm run build        # Production build
npm run start        # Production server
npm run lint         # ESLint
npm run typecheck    # TypeScript check (--noEmit)
npm run format       # Prettier write
npm run format:check # Prettier check
npm run test         # Vitest run
npm run test:watch   # Vitest watch
```

### Next.js Configuration

- **next-intl** plugin for i18n
- **Security Headers**: HSTS (63072000s), X-Content-Type-Options (nosniff), X-Frame-Options (DENY), Referrer-Policy (strict-origin-when-cross-origin), Permissions-Policy (camera denied, microphone self-only)
- **CSP Report-Only**: Full policy with directives for all resource types
- **Cache-Control**: API routes `no-store`; static assets to Next defaults; everything else `s-maxage=300, stale-while-revalidate=86400`

---

## 14. Key Metrics

| Metric | Value |
|---|---|
| **Total source files** | ~390 |
| **Total lines of code** | ~15,000+ (excluding generated/node_modules) |
| **Pages** | 20 |
| **API endpoints** | 50 (42 internal + 8 public v1) |
| **React components** | 109 |
| **Database tables** | 25+ |
| **Database migrations** | 35 |
| **Database RPCs** | 16 |
| **Test files** | 55 |
| **Storage buckets** | 3 |
| **MCP tools** | 11 |
| **npm dependencies** | 20 production, 8 dev |
| **npm vulnerabilities (known)** | 16 (noted in security audit) |
| **Version history** | 12 releases (v0.1.0 → v0.8.0) |

### Page Load & Performance Notes

- Theme boot script runs before React hydration to prevent flash of unstyled content (FOUC)
- Supabase browser client is a singleton to prevent auth-lock contention
- Broadcast sending uses batched inserts (200/batch) and batched API calls (10/batch with 1s delay)
- Inbox implements a resync mechanism for WebSocket reconnection
- Flow editor uses @dagrejs/dagre for automatic graph layout
- All API routes return `Cache-Control: no-store` to prevent stale data

---

*Report generated by analyzing all source files in the WACRM codebase. The `.env.local` file was intentionally excluded from analysis per security policy.*
