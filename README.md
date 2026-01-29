# Decommerce MCP + AI Micro-Apps — Technical Specification

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Vision & Goals](#vision--goals)
3. [Why MCP + Micro-Apps?](#why-mcp--micro-apps)
4. [System Architecture](#system-architecture)
5. [Part 1: MCP Server](#part-1-mcp-server)
6. [Part 2: App Framework](#part-2-app-framework)
7. [Part 3: Admin Panel AI Chatbox](#part-3-admin-panel-ai-chatbox)
8. [Micro-App Catalog](#micro-app-catalog)
9. [App Configuration Schema](#app-configuration-schema)
10. [Variable Interpolation](#variable-interpolation)
11. [App Configuration Validation](#app-configuration-validation)
12. [Data Connectors](#data-connectors)
13. [Logic Transformations](#logic-transformations)
14. [Action Executors](#action-executors)
15. [Triggers System](#triggers-system)
16. [Execution Safeguards](#execution-safeguards)
17. [Error Recovery & Retry](#error-recovery--retry)
18. [Rate Limits & Quotas](#rate-limits--quotas)
19. [Implementation Phases](#implementation-phases)
20. [Testing & Verification](#testing--verification)
21. [Key Reference Files](#key-reference-files)

---

## Executive Summary

This document specifies a system that allows **AI to create micro-apps on the Decommerce platform**. Unlike Shopify where human developers build apps, here **AI generates apps on the fly** based on admin requests via a chatbox interface.

**The system has three parts:**

| Part | Description | Location |
|------|-------------|----------|
| **MCP Server** | Protocol layer allowing AI to interact with Decommerce | Backend (NestJS) |
| **App Framework** | Engine that stores, runs, and manages micro-apps | Backend (NestJS) |
| **AI Chatbox** | Interface where admins describe apps in natural language | Admin Panel (React) |

**Example flow:**
1. Admin types: *"Create an app that detects low-quality posts and flags them for review"*
2. AI understands the request and generates an app configuration
3. App Framework saves and activates the app
4. App runs automatically, flagging posts as they're created

---

## Vision & Goals

### Vision

> "Unlike Shopify where human developers build apps, here the AI itself creates these micro-apps. Admin asks AI → AI builds the app on the fly → app is ready to use."

### Goals

- Enable tenant admins to create automation apps via natural language
- AI generates app configurations (not code) that the framework executes
- Support 30+ app types covering email, content, engagement, analytics, moderation
- Maintain multi-tenant isolation — each tenant has their own apps
- Build on existing Decommerce features (users, rooms, posts, missions, rewards, Shopify)

### Non-Goals (Out of Scope)

- AI writing actual code (React components, NestJS modules)
- Apps that require new database tables or UI components
- Features like DM, video chat, payments — these need core development

---

## Why MCP + Micro-Apps?

### Zapier vs MCP vs Micro-Apps

| Aspect | Zapier | MCP (External) | MCP + Micro-Apps |
|--------|--------|----------------|------------------|
| **User** | Business users | Developers with AI tools | Tenant admins |
| **Interface** | Zapier UI | Claude Desktop/ChatGPT | Admin panel chatbox |
| **Creates** | Zaps (trigger → action) | One-off operations | Persistent apps |
| **Intelligence** | None | AI-powered | AI-powered |
| **Runs** | On Zapier's servers | On-demand | On Decommerce servers |

### Key Benefits

1. **No-Code App Creation** — Admins describe apps in plain English
2. **AI-Powered** — Apps can use AI for content generation, analysis, decisions
3. **Persistent Automation** — Apps run continuously (scheduled, event-driven)
4. **Platform Differentiator** — No other community platform offers this

### What Apps Can Be Created?

| Category | Examples | Count |
|----------|----------|-------|
| Email/Communication | Weekly digests, win-back campaigns, welcome series | 7+ |
| Content Automation | AI blog posts, welcome posts, recaps | 6+ |
| Engagement/Gamification | Campaigns, leaderboards, streaks | 6+ |
| Analytics/Reporting | Admin reports, churn alerts, performance reports | 6+ |
| Moderation/Management | Post detection, cleanup, scoring | 5+ |
| Shopify Integration | Thank you posts, VIP rewards, review requests | 5+ |
| **Total** | | **35+** |

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ADMIN PANEL (React)                              │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      AI Chatbox Component                          │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │ Admin: "Create an app that sends weekly engagement emails"  │  │  │
│  │  │                                                              │  │  │
│  │  │ AI: "I'll create a Weekly Engagement Digest app..."         │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      Apps Management UI                            │  │
│  │  [Weekly Digest ✓] [Post Detector ✓] [Welcome Series ○]           │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      DECOMMERCE BACKEND (NestJS)                         │
│                                                                          │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │
│  │   MCP Server    │   │  App Framework  │   │   AI Service        │   │
│  │                 │   │                 │   │   (Claude API)      │   │
│  │  Tools:         │   │  - App Storage  │   │                     │   │
│  │  - create_app   │──▶│  - Triggers     │◀──│  - Content Gen      │   │
│  │  - list_apps    │   │  - Connectors   │   │  - Analysis         │   │
│  │  - run_app      │   │  - Actions      │   │  - Decisions        │   │
│  │  - delete_app   │   │  - Scheduler    │   │                     │   │
│  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │
│                                    │                                     │
│                                    ▼                                     │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    Existing Decommerce Services                    │  │
│  │  [Users] [Rooms] [Posts] [Missions] [Rewards] [Shopify] [Email]   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                    │                                     │
│                                    ▼                                     │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    Tenant Database (PostgreSQL)                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Request Flow: Creating an App

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Admin   │     │  Admin   │     │  Claude  │     │   MCP    │     │   App    │
│  Panel   │     │  Backend │     │   API    │     │  Server  │     │Framework │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │                │                │
     │ "Create weekly │                │                │                │
     │  email app"    │                │                │                │
     │───────────────▶│                │                │                │
     │                │                │                │                │
     │                │  Send to Claude│                │                │
     │                │  with MCP tools│                │                │
     │                │───────────────▶│                │                │
     │                │                │                │                │
     │                │                │  AI decides to │                │
     │                │                │  call create_app                │
     │                │                │───────────────▶│                │
     │                │                │                │                │
     │                │                │                │  Validate +    │
     │                │                │                │  Store config  │
     │                │                │                │───────────────▶│
     │                │                │                │                │
     │                │                │                │  { success }   │
     │                │                │                │◀───────────────│
     │                │                │                │                │
     │                │                │  { app created}│                │
     │                │                │◀───────────────│                │
     │                │                │                │                │
     │                │  "App created!"│                │                │
     │                │◀───────────────│                │                │
     │                │                │                │                │
     │ "✓ Weekly      │                │                │                │
     │  Digest created"                │                │                │
     │◀───────────────│                │                │                │
```

### Request Flow: App Execution

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│Scheduler │     │   App    │     │  Data    │     │  Claude  │     │  Action  │
│ (Cron)   │     │Framework │     │Connectors│     │   API    │     │Executors │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │                │                │
     │ Trigger:       │                │                │                │
     │ Monday 9am     │                │                │                │
     │───────────────▶│                │                │                │
     │                │                │                │                │
     │                │ Load app config│                │                │
     │                │ "Weekly Digest"│                │                │
     │                │                │                │                │
     │                │ Fetch users    │                │                │
     │                │───────────────▶│                │                │
     │                │                │                │                │
     │                │  [1000 users]  │                │                │
     │                │◀───────────────│                │                │
     │                │                │                │                │
     │                │ Fetch engagement                │                │
     │                │───────────────▶│                │                │
     │                │                │                │                │
     │                │  [user stats]  │                │                │
     │                │◀───────────────│                │                │
     │                │                │                │                │
     │                │ For each user: │                │                │
     │                │ Generate email │                │                │
     │                │───────────────────────────────▶│                │
     │                │                │                │                │
     │                │                │  [email body]  │                │
     │                │◀───────────────────────────────│                │
     │                │                │                │                │
     │                │ Send email     │                │                │
     │                │───────────────────────────────────────────────▶│
     │                │                │                │                │
     │                │                │                │     { sent }   │
     │                │◀───────────────────────────────────────────────│
```

---

## Part 1: MCP Server

The MCP Server exposes tools that AI can call to interact with Decommerce.

### MCP Server Module Structure

<details>
<summary>View MCP Server Module Structure</summary>

```
src/mcp/
├── mcp.module.ts                    # NestJS module
├── mcp.controller.ts                # HTTP endpoints for MCP transport
├── mcp-server.factory.ts            # Creates MCP Server instance
├── mcp-context.service.ts           # Per-request tenant context
│
├── auth/
│   └── mcp-api-key.guard.ts         # API key validation
│
├── tools/
│   ├── index.ts                     # Registers all tools
│   ├── apps.tools.ts                # create_app, list_apps, update_app, delete_app, run_app
│   ├── users.tools.ts               # list_users, get_user, etc.
│   ├── rooms.tools.ts               # list_rooms, get_room, etc.
│   ├── posts.tools.ts               # list_posts, create_post, etc.
│   ├── missions.tools.ts            # list_missions, create_mission, etc.
│   ├── rewards.tools.ts             # list_contributions, etc.
│   ├── notifications.tools.ts       # send_notification
│   └── analytics.tools.ts           # get_stats, get_metrics
│
├── resources/
│   ├── index.ts
│   ├── platform-schema.resource.ts  # Entity documentation
│   ├── app-templates.resource.ts    # Available app building blocks
│   └── enums.resource.ts            # Platform enums
│
└── guards/
    └── mcp-rate-limit.guard.ts
```

</details>

### MCP Tools for App Management

| Tool | Description | Input |
|------|-------------|-------|
| `create_app` | Create a new micro-app | `{ name, description, trigger, data_pipeline, actions }` |
| `list_apps` | List all apps for this tenant | `{ status?, limit?, offset? }` |
| `get_app` | Get app details and run history | `{ app_id }` |
| `update_app` | Update app configuration | `{ app_id, ...updates }` |
| `delete_app` | Delete an app | `{ app_id }` |
| `run_app` | Manually trigger an app | `{ app_id }` |
| `pause_app` | Pause an app | `{ app_id }` |
| `resume_app` | Resume a paused app | `{ app_id }` |
| `get_app_logs` | Get execution logs | `{ app_id, limit? }` |

### MCP Tools for Data Access

| Tool | Description |
|------|-------------|
| `list_users` | List users with filters |
| `get_user` | Get user profile + stats |
| `list_rooms` | List rooms |
| `get_room` | Get room details |
| `list_posts` | List posts with filters |
| `create_post` | Create a post |
| `list_missions` | List missions |
| `create_mission` | Create a mission |
| `list_contributions` | List reward contributions |
| `get_community_stats` | Get overview stats |
| `get_engagement_metrics` | Get engagement over time |
| `send_notification` | Send push/email notification |

---

## Part 2: App Framework

The App Framework is the **runtime engine** that stores, schedules, executes, and monitors micro-apps. While the MCP Server lets AI create apps, the App Framework is what actually runs them.

### What the App Framework Does

Think of the App Framework as a **workflow automation engine** with four core responsibilities:

1. **Storage** — Saves app configurations as JSON in the database (not code)
2. **Triggering** — Knows when to run each app (scheduled time, event fired, manual click)
3. **Execution** — Runs the app's data pipeline and actions
4. **Monitoring** — Logs every execution for debugging and auditing

### The Building Blocks Concept

Apps are built from **three types of building blocks** that AI combines:

| Building Block | What It Does | Examples |
|----------------|--------------|----------|
| **Connectors** | Fetch data from the platform | Get users, get posts, get Shopify orders |
| **Actions** | Do something with the data | Send email, create post, award XP, flag content |
| **Triggers** | Decide when the app runs | Every Monday 9am, when post is created, when admin clicks Run |

**The key insight:** AI doesn't write code — it assembles these pre-built blocks into a configuration. The framework then executes that configuration safely and predictably.

### How an App Executes

When an app runs (triggered by schedule, event, or manual click), here's what happens:

```
1. TRIGGER fires (e.g., Monday 9am)
         ↓
2. Load app configuration from database
         ↓
3. DATA PIPELINE runs:
   - Connector 1: Fetch active users → [1000 users]
   - Connector 2: Fetch engagement stats → [user stats]
   - Connector 3: Fetch Shopify orders → [recent orders]
   - Merge/filter/transform data as needed
         ↓
4. ACTIONS execute:
   - For each user:
     - AI generates personalized email content
     - Send email via SendGrid
         ↓
5. LOG execution results (success/failure, items processed, errors)
```

### Why Configuration-Based Apps Matter

| Approach | Risk | Benefit |
|----------|------|---------|
| AI writes actual code | High — code could do anything | Maximum flexibility |
| AI generates configuration | Low — framework controls what's possible | Safe, predictable, auditable |

We chose **configuration-based apps** because:
- AI can only use pre-built connectors and actions (no arbitrary code execution)
- Every app is inspectable — admins can see exactly what it does
- Execution is sandboxed — apps can't access anything outside defined connectors
- Easy to pause, modify, or delete without code changes

### App Framework Module Structure

<details>
<summary>View App Framework Module Structure</summary>

```
src/app-framework/
├── app-framework.module.ts          # NestJS module
│
├── entities/
│   ├── app.entity.ts                # App configuration storage
│   ├── app-execution.entity.ts      # Execution history
│   └── app-log.entity.ts            # Detailed logs
│
├── services/
│   ├── app.service.ts               # CRUD operations for apps
│   ├── app-executor.service.ts      # Runs app logic
│   ├── app-scheduler.service.ts     # Cron/scheduled triggers
│   └── app-event-listener.service.ts # Event-based triggers
│
├── connectors/                      # Data source connectors
│   ├── connector.interface.ts       # Base interface
│   ├── users.connector.ts           # Fetch user data
│   ├── engagement.connector.ts      # Fetch engagement metrics
│   ├── posts.connector.ts           # Fetch posts
│   ├── rooms.connector.ts           # Fetch rooms
│   ├── missions.connector.ts        # Fetch missions
│   ├── rewards.connector.ts         # Fetch rewards/contributions
│   ├── shopify.connector.ts         # Fetch Shopify orders
│   └── analytics.connector.ts       # Fetch aggregated stats
│
├── actions/                         # Action executors
│   ├── action.interface.ts          # Base interface
│   ├── send-email.action.ts         # Send email via SendGrid
│   ├── create-post.action.ts        # Create a post
│   ├── send-notification.action.ts  # Push notification
│   ├── update-user.action.ts        # Update user fields
│   ├── create-mission.action.ts     # Create a mission
│   ├── award-xp.action.ts           # Give XP to users
│   ├── flag-content.action.ts       # Flag post for review
│   └── webhook.action.ts            # Call external URL
│
├── logic/                           # Data transformation
│   ├── filter.logic.ts              # Filter data
│   ├── sort.logic.ts                # Sort data
│   ├── group.logic.ts               # Group data
│   ├── aggregate.logic.ts           # Sum, count, average
│   ├── merge.logic.ts               # Combine data sources
│   └── ai-generate.logic.ts         # AI content generation
│
├── triggers/                        # Trigger types
│   ├── trigger.interface.ts         # Base interface
│   ├── scheduled.trigger.ts         # Cron-based
│   ├── event.trigger.ts             # On user signup, post created, etc.
│   └── manual.trigger.ts            # Admin clicks "Run"
│
└── dto/
    ├── app-config.dto.ts            # App configuration schema
    └── execution-result.dto.ts      # Execution result format
```

</details>

### Understanding the Components

#### Services (The Brain)

| Service | Responsibility |
|---------|----------------|
| `app.service.ts` | CRUD operations — create, read, update, delete apps |
| `app-executor.service.ts` | The execution engine — runs data pipeline → actions in sequence |
| `app-scheduler.service.ts` | Manages cron jobs — loads scheduled apps on startup, creates/removes jobs |
| `app-event-listener.service.ts` | Listens to platform events (user.created, post.created) and triggers matching apps |

#### Connectors (Data In)

Connectors are **read-only data sources**. Each connector knows how to fetch one type of data with filters:

| Connector | What It Fetches | Example Filters |
|-----------|-----------------|-----------------|
| `users` | User records | status=Active, created_after=7d ago |
| `user_engagement` | Calculated stats | XP this week, posts count, missions completed |
| `posts` | Post records | room_id=5, status=Published |
| `rooms` | Room records | room_type_id=3 |
| `missions` | Mission records | is_active=true, mission_type=Quiz |
| `shopify_orders` | Shopify order data | created_after=7d ago, total_gte=100 |
| `analytics` | Aggregated metrics | Period=last_30_days |

**Why separate connectors?** Each connector handles its own query logic, validation, and error handling. AI just specifies which connector to use and what filters to apply.

#### Actions (Data Out)

Actions are **operations that change state**. Each action knows how to do one thing:

| Action | What It Does | Required Params |
|--------|--------------|-----------------|
| `send_email` | Send email via SendGrid | to, subject, body |
| `create_post` | Create a post in a room | room_id, content, author_id |
| `send_notification` | Push notification to users | user_ids, title, message |
| `update_user` | Update user fields | user_id, fields |
| `award_xp` | Give XP to a user | user_id, amount, reason |
| `flag_content` | Flag post for review | content_id, reason |
| `ai_generate` | Generate content using Claude | prompt, output_var |
| `ai_analyze` | Analyze content using Claude | input, prompt, output_var |
| `for_each` | Loop over items and run nested actions | items, as, do |
| `webhook` | Call external URL | url, method, body |

**Why separate actions?** Each action handles validation, error handling, and rollback. The executor just calls actions in sequence.

#### Triggers (When to Run)

| Trigger Type | How It Works | Use Case |
|--------------|--------------|----------|
| `scheduled` | Cron job runs at specified time | Weekly digest every Monday 9am |
| `event` | Platform emits event, listener checks for matching apps | Moderate new posts, welcome new users |
| `manual` | Admin clicks "Run" button | One-time reports, testing |

### Database Schema

The App Framework uses two main tables to store apps and track their execution history.

#### App Entity

The `apps` table stores the app configuration and status. Each row is one micro-app.

**Key fields:**
- `tenant_id` — Tenant identifier for multi-tenant isolation. Each tenant only sees their own apps.
- `config` (JSONB) — The full app configuration: trigger, data pipeline, actions. This is what AI generates.
- `status` — ACTIVE (running), PAUSED (stopped), or ERROR (failed too many times)
- `version` — App version for tracking changes and supporting rollback
- `created_by` — Admin user who created the app (for audit and permissions)
- `run_count` / `error_count` — Execution statistics for monitoring
- `last_run_at` — When the app last executed (for debugging)

<details>
<summary>View App Entity Code</summary>

```typescript
@Entity('apps')
export class App extends EntityHelper {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  tenant_id: string;  // Multi-tenant isolation - CRITICAL for security

  @Column()
  name: string;

  @Column({ type: 'text', nullable: true })
  description: string;

  @Column({ type: 'jsonb' })
  config: AppConfig;  // Full app configuration

  @Column({ type: 'int', default: 1 })
  version: number;  // Incremented on each update

  @Column({ type: 'jsonb', nullable: true })
  previous_configs: AppConfig[];  // Version history for rollback (last 5 versions)

  @Column({ type: 'enum', enum: AppStatus, default: AppStatus.ACTIVE })
  status: AppStatus;  // ACTIVE, PAUSED, ERROR

  @Column()
  created_by: number;  // Admin user ID who created this app

  @Column({ nullable: true })
  updated_by: number;  // Admin user ID who last modified this app

  @Column({ type: 'timestamp', nullable: true })
  last_run_at: Date;

  @Column({ type: 'int', default: 0 })
  run_count: number;

  @Column({ type: 'int', default: 0 })
  error_count: number;

  @Column({ type: 'int', default: 0 })
  consecutive_errors: number;  // Reset on success, used for auto-pause

  @CreateDateColumn()
  created_at: Date;

  @UpdateDateColumn()
  updated_at: Date;
}

// All queries MUST include tenant_id to ensure multi-tenant isolation
// Example: appRepository.find({ where: { tenant_id: currentTenantId } })
```

</details>

#### App Execution Entity

The `app_executions` table logs every time an app runs. This provides:
- **Audit trail** — See exactly when each app ran and what it did
- **Debugging** — If something fails, check the error message and result
- **Monitoring** — Track success rates, processing times, items handled

<details>
<summary>View App Execution Entity Code</summary>

```typescript
@Entity('app_executions')
export class AppExecution extends EntityHelper {
  @PrimaryGeneratedColumn()
  id: number;

  @ManyToOne(() => App)
  app: App;

  @Column({ type: 'enum', enum: ExecutionStatus })
  status: ExecutionStatus;  // RUNNING, SUCCESS, FAILED

  @Column({ type: 'timestamp' })
  started_at: Date;

  @Column({ type: 'timestamp', nullable: true })
  completed_at: Date;

  @Column({ type: 'int', nullable: true })
  items_processed: number;

  @Column({ type: 'jsonb', nullable: true })
  result: any;

  @Column({ type: 'text', nullable: true })
  error_message: string;
}
```

</details>

---

## Part 3: Admin Panel AI Chatbox

The AI Chatbox is the **user interface** where admins interact with AI to create and manage micro-apps. It's a React component embedded in the admin panel.

### How the Chatbox Works

1. **Admin types a request** in natural language (e.g., "Create an app that sends weekly emails")
2. **Frontend sends request** to backend API endpoint
3. **Backend forwards to Claude API** with MCP tools available
4. **Claude decides what to do** — might call `create_app` tool, or ask clarifying questions
5. **Response streams back** to the chatbox in real-time
6. **If app is created**, it appears in the Apps List immediately

### The Conversation Flow

```
Admin: "Create an app that flags low-quality posts"
                    ↓
Backend: Sends to Claude with system prompt + MCP tools
                    ↓
Claude: Understands intent, generates app config, calls create_app tool
                    ↓
MCP Server: Validates config, stores in database
                    ↓
Claude: "I've created a Low-Quality Post Detector app. It will..."
                    ↓
Frontend: Shows response + new app in list
```

### Frontend Components

```
admin-panel/src/
├── components/
│   └── ai-apps/
│       ├── AIChatbox.tsx            # Main chat interface
│       ├── ChatMessage.tsx          # Individual message component
│       ├── AppConfirmation.tsx      # "Create this app?" dialog
│       ├── AppsList.tsx             # List of created apps
│       ├── AppCard.tsx              # Individual app card
│       ├── AppDetails.tsx           # App detail view
│       └── AppLogs.tsx              # Execution logs view
│
├── pages/
│   └── ai-apps/
│       ├── index.tsx                # Main AI Apps page
│       └── [appId].tsx              # App detail page
│
├── hooks/
│   └── useAIChat.ts                 # Chat state management
│
└── services/
    └── ai-apps.service.ts           # API calls to backend
```

### Component Responsibilities

| Component | What It Does |
|-----------|--------------|
| `AIChatbox.tsx` | Main chat UI — input box, message list, streaming response display |
| `ChatMessage.tsx` | Renders individual messages (user vs AI styling, markdown support) |
| `AppConfirmation.tsx` | Modal that shows "AI wants to create this app — confirm?" before creation |
| `AppsList.tsx` | Grid/list of all apps with status badges, last run time, action buttons |
| `AppCard.tsx` | Individual app card with name, description, status, quick actions |
| `AppDetails.tsx` | Full app detail view — configuration, execution history, logs |
| `AppLogs.tsx` | Execution log viewer with filtering, error highlighting |

### Chat Interface Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│  🤖 AI App Builder                                              [?] [×] │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 🤖 AI                                                               │ │
│  │ Hi! I can help you create automation apps for your community.      │ │
│  │ Try asking me to:                                                   │ │
│  │ • "Create a weekly email digest for active users"                  │ │
│  │ • "Build an app that detects low-quality posts"                    │ │
│  │ • "Set up a welcome series for new members"                        │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 👤 You                                                              │ │
│  │ Create an app that sends personalized emails every Monday based    │ │
│  │ on user's activity and Shopify purchases                           │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 🤖 AI                                                               │ │
│  │ I'll create a **Weekly Personalized Email** app for you:           │ │
│  │                                                                     │ │
│  │ **Trigger:** Every Monday at 9:00 AM                               │ │
│  │                                                                     │ │
│  │ **Data Sources:**                                                   │ │
│  │ • Active users with email                                          │ │
│  │ • User engagement (XP, posts, missions)                            │ │
│  │ • Shopify orders (last 7 days)                                     │ │
│  │                                                                     │ │
│  │ **Action:**                                                         │ │
│  │ • Generate personalized email using AI                             │ │
│  │ • Send via SendGrid                                                │ │
│  │                                                                     │ │
│  │ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐        │ │
│  │ │ ✓ Create App    │ │ ✎ Modify        │ │ ✗ Cancel        │        │ │
│  │ └─────────────────┘ └─────────────────┘ └─────────────────┘        │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ Type your request...                                      [Send]│    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Apps Management UI

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Your Micro-Apps                                           [+ Create New]│
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 📧 Weekly Personalized Email                        [Active] ⚙️ ▶️  │ │
│  │ Sends personalized emails every Monday                              │ │
│  │ ────────────────────────────────────────────────────────────────── │ │
│  │ Trigger: Every Monday 9am  │  Last run: 2 days ago  │  Sent: 1,247 │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 🔍 Low-Quality Post Detector                        [Active] ⚙️ ▶️  │ │
│  │ Flags suspicious posts for review                                   │ │
│  │ ────────────────────────────────────────────────────────────────── │ │
│  │ Trigger: On new post  │  Last run: 5 min ago  │  Flagged: 3 today  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ 👋 New User Welcome Series                          [Paused] ⚙️ ▶️  │ │
│  │ 3-email series for new users (Day 1, 3, 7)                          │ │
│  │ ────────────────────────────────────────────────────────────────── │ │
│  │ Trigger: On user signup  │  Last run: 1 day ago  │  Sent: 89       │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Chatbox Backend API

The chatbox communicates with the backend through a REST API that handles AI conversations and streaming responses.

#### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/ai-apps/chat` | Send a message and get AI response (streaming) |
| GET | `/api/v1/ai-apps/conversations` | List conversation history |
| GET | `/api/v1/ai-apps/conversations/:id` | Get specific conversation |
| DELETE | `/api/v1/ai-apps/conversations/:id` | Delete conversation |

#### Chat Endpoint

**Request:**

```typescript
POST /api/v1/ai-apps/chat
Authorization: Bearer {admin_jwt_token}
Content-Type: application/json

{
  "message": "Create an app that sends weekly engagement emails",
  "conversation_id": "conv_abc123",  // Optional: continue existing conversation
  "require_confirmation": true       // Optional: require user confirm before app creation
}
```

**Response (Server-Sent Events for streaming):**

```typescript
// Stream response using SSE
Content-Type: text/event-stream

event: message_start
data: {"conversation_id": "conv_abc123", "message_id": "msg_xyz789"}

event: content_delta
data: {"delta": "I'll create a **Weekly Engagement"}

event: content_delta
data: {"delta": " Digest** app for you."}

event: tool_use
data: {"tool": "create_app", "status": "pending", "app_preview": {...}}

event: tool_result
data: {"tool": "create_app", "status": "success", "app_id": 42}

event: content_delta
data: {"delta": "\n\n✓ App created successfully!"}

event: message_complete
data: {"message_id": "msg_xyz789", "usage": {"input_tokens": 150, "output_tokens": 320}}
```

#### Chat Controller Implementation

<details>
<summary>View Chat Controller Implementation</summary>

```typescript
@Controller('api/v1/ai-apps')
@UseGuards(JwtAuthGuard, AdminGuard)
export class AiAppsController {
  @Post('chat')
  @Sse()
  async chat(
    @Body() body: ChatRequestDto,
    @Request() req: AuthenticatedRequest,
  ): Promise<Observable<MessageEvent>> {
    const { tenant_id, user_id } = req;

    // Rate limit check
    const rateCheck = await this.rateLimitService.checkLimit(tenant_id, 'chatbox');
    if (!rateCheck.allowed) {
      throw new TooManyRequestsException(
        `Rate limit exceeded. Try again in ${rateCheck.reset_in_seconds}s`
      );
    }

    // Create or get conversation
    const conversation = body.conversation_id
      ? await this.conversationService.get(body.conversation_id)
      : await this.conversationService.create(tenant_id, user_id);

    // Build Claude request with MCP tools
    const claudeRequest = {
      model: process.env.ANTHROPIC_MODEL,
      max_tokens: 4096,
      system: this.buildSystemPrompt(tenant_id),
      messages: [
        ...conversation.messages,
        { role: 'user', content: body.message },
      ],
      tools: this.mcpToolService.getAvailableTools(tenant_id),
      stream: true,
    };

    // Return streaming response
    return this.claudeService.streamWithToolExecution(
      claudeRequest,
      {
        tenant_id,
        user_id,
        conversation_id: conversation.id,
        require_confirmation: body.require_confirmation,
      }
    );
  }

  private buildSystemPrompt(tenant_id: string): string {
    return `You are an AI assistant that helps create micro-apps for the Decommerce platform.

Tenant: ${tenant_id}
Available tools: create_app, list_apps, update_app, delete_app, run_app, pause_app

When creating apps:
1. Ask clarifying questions if the request is ambiguous
2. Generate a complete app configuration
3. Explain what the app will do before creating it
4. Use the create_app tool to save the app

Always be helpful and explain your actions clearly.`;
  }
}
```

</details>

#### Tool Confirmation Flow

When `require_confirmation: true`, the API pauses before executing destructive tools:

```typescript
event: tool_use
data: {
  "tool": "create_app",
  "status": "awaiting_confirmation",
  "app_preview": {
    "name": "Weekly Engagement Digest",
    "trigger": "scheduled",
    "actions": ["fetch users", "generate email", "send email"]
  },
  "confirmation_id": "confirm_abc123"
}

// Frontend shows confirmation dialog
// User clicks "Confirm"

POST /api/v1/ai-apps/confirm
{
  "confirmation_id": "confirm_abc123",
  "approved": true
}

// Stream continues with tool execution
event: tool_result
data: {"tool": "create_app", "status": "success", "app_id": 42}
```

#### Conversation Storage

```typescript
interface Conversation {
  id: string;
  tenant_id: string;
  user_id: number;
  messages: Message[];
  created_at: Date;
  updated_at: Date;
}

interface Message {
  role: 'user' | 'assistant';
  content: string;
  tool_calls?: ToolCall[];
  timestamp: Date;
}
```

---

## Micro-App Catalog

### Category 1: Email & Communication Apps

| App Name | Trigger | Data Sources | Actions |
|----------|---------|--------------|---------|
| Weekly Engagement Digest | Scheduled (weekly) | Users, Engagement, Missions | AI generate email → Send email |
| Win-Back Campaign | Scheduled (daily) | Users (inactive 14+ days) | AI generate email → Send email |
| New User Welcome Series | On user signup | Users | Send email (Day 1, 3, 7) |
| Shopify + Community Email | Scheduled (weekly) | Users, Engagement, Shopify Orders | AI generate email → Send email |
| Mission Reminder | Scheduled (daily) | Users, UserMissions (incomplete) | Send notification |
| Reward Available Alert | On reward qualified | Users, Contributions | Send email |
| Top Contributor Spotlight | Scheduled (weekly) | Users, Engagement (top 10) | AI generate email → Send to all |

### Category 2: Content Automation Apps

| App Name | Trigger | Data Sources | Actions |
|----------|---------|--------------|---------|
| AI Weekly Recap Post | Scheduled (weekly) | Analytics, Posts (top) | AI generate content → Create post |
| Auto Welcome Post | On user joins room | Users, Rooms | AI generate welcome → Create post |
| Product Announcement | On Shopify product added | Shopify Products | AI generate announcement → Create post |
| Event Reminder Post | Scheduled (before event) | Missions (upcoming) | Create reminder post |
| User Milestone Celebration | On XP milestone | Users | AI generate congrats → Create post |
| Community Highlights | Scheduled (monthly) | Posts, Users, Analytics | AI generate summary → Create post |

### Category 3: Engagement & Gamification Apps

| App Name | Trigger | Data Sources | Actions |
|----------|---------|--------------|---------|
| Engagement Campaign Builder | Manual / Scheduled | Missions, Rewards | Create missions + Create rewards |
| Weekly Leaderboard Rewards | Scheduled (weekly) | Users (top 10 by XP) | Award XP + Send notification |
| Streak Bonus System | Daily check | Users, Engagement (daily activity) | Award bonus XP |
| Referral Booster Campaign | Manual | Invitations, Users | Create referral missions |
| Room Activity Challenge | Manual | Rooms, Posts | Create challenge mission |
| New User Onboarding | On user signup | Users, Missions | Assign kick-off missions |

### Category 4: Analytics & Reporting Apps

| App Name | Trigger | Data Sources | Actions |
|----------|---------|--------------|---------|
| Weekly Admin Report | Scheduled (weekly) | Analytics, Users, Posts | AI generate report → Email admins |
| Churn Risk Alert | Scheduled (daily) | Users, Engagement | AI analyze → Notification to admins |
| Content Performance Report | Scheduled (weekly) | Posts, Votes, Comments | AI generate report → Email admins |
| Mission Effectiveness Report | Scheduled (monthly) | Missions, UserMissions | AI analyze → Create report post |
| Shopify-Community Correlation | Scheduled (monthly) | Shopify, Engagement | AI analyze → Email admins |
| Room Health Dashboard | Scheduled (weekly) | Rooms, Posts, Users | Generate stats → Email admins |

### Category 5: Moderation & Management Apps

| App Name | Trigger | Data Sources | Actions |
|----------|---------|--------------|---------|
| Low-Quality Post Detector | On new post | Posts | AI analyze → Flag post / Notify admins |
| Inactive User Cleanup | Scheduled (weekly) | Users (inactive 30+ days) | Flag users / Send win-back email |
| New User Watchlist | On new user's first post | Users, Posts | AI analyze → Flag if suspicious |
| Engagement Score Calculator | Scheduled (daily) | Users, Posts, Missions, Votes | Calculate score → Update user |
| Auto-Archive Old Rooms | Scheduled (monthly) | Rooms (no activity 60 days) | Archive room / Notify admins |

### Category 6: Shopify Integration Apps

| App Name | Trigger | Data Sources | Actions |
|----------|---------|--------------|---------|
| Purchase Thank You | On Shopify order | Shopify Orders, Users | Create thank you post + Mission |
| VIP Buyer Rewards | On order above threshold | Shopify Orders, Users | Award bonus XP + Notification |
| Product Review Request | 7 days after purchase | Shopify Orders, Users | Send email with review mission |
| First Purchase Celebration | On first order | Shopify Orders, Users | Create celebration post + Reward |
| Abandoned Cart Recovery | On cart abandoned | Shopify Carts, Users | Send notification |

---

## App Configuration Schema

### Full Schema

<details>
<summary>View <code>AppConfig</code> schema (TypeScript)</summary>

```typescript
interface AppConfig {
  // App metadata
  app: {
    name: string;
    description?: string;
    version: string;
    icon?: string;
  };

  // When the app runs
  trigger: TriggerConfig;

  // Data to fetch
  data_pipeline: DataStepConfig[];

  // What to do with the data
  actions: ActionConfig[];

  // Optional settings
  settings?: {
    enabled: boolean;
    retry_on_error: boolean;
    max_retries: number;
    notify_on_error: boolean;
  };
}

// Trigger configurations
type TriggerConfig =
  | { type: 'scheduled'; cron: string; timezone?: string }
  | { type: 'event'; event: EventType; filter?: Record<string, any> }
  | { type: 'manual' };

type EventType =
  | 'user.created'
  | 'user.verified'
  | 'post.created'
  | 'post.reported'
  | 'mission.completed'
  | 'reward.claimed'
  | 'room.joined'
  | 'shopify.order.created'
  | 'shopify.order.fulfilled';

// Data pipeline step
interface DataStepConfig {
  id: string;
  connector: ConnectorType;
  filter?: Record<string, any>;
  fields?: string[];
  join_on?: string;
  limit?: number;
}

type ConnectorType =
  | 'users'
  | 'user_engagement'
  | 'posts'
  | 'rooms'
  | 'room_members'
  | 'missions'
  | 'user_missions'
  | 'contributions'
  | 'reward_claims'
  | 'shopify_orders'
  | 'shopify_products'
  | 'analytics'
  | 'notifications';

// Action configuration
interface ActionConfig {
  type: ActionType;
  condition?: ConditionConfig;
  params: Record<string, any>;
}

type ActionType =
  | 'send_email'
  | 'create_post'
  | 'send_notification'
  | 'update_user'
  | 'create_mission'
  | 'award_xp'
  | 'flag_content'
  | 'webhook'
  | 'for_each'
  | 'ai_analyze'
  | 'ai_generate';

// Condition configuration for conditional actions
interface ConditionConfig {
  field: string;           // Field path (supports dot notation: "user.status")
  operator: ConditionOperator;
  value: any;              // Value to compare against
  // Optional: combine multiple conditions
  and?: ConditionConfig[];
  or?: ConditionConfig[];
}

type ConditionOperator =
  | 'equals'           // field === value
  | 'not_equals'       // field !== value
  | 'contains'         // field.includes(value) - for strings/arrays
  | 'not_contains'     // !field.includes(value)
  | 'starts_with'      // field.startsWith(value)
  | 'ends_with'        // field.endsWith(value)
  | 'greater_than'     // field > value
  | 'greater_than_or_equals' // field >= value
  | 'less_than'        // field < value
  | 'less_than_or_equals'    // field <= value
  | 'is_null'          // field === null || field === undefined
  | 'is_not_null'      // field !== null && field !== undefined
  | 'is_empty'         // field.length === 0 (for strings/arrays)
  | 'is_not_empty'     // field.length > 0
  | 'in'               // value.includes(field) - field is in array
  | 'not_in'           // !value.includes(field)
  | 'matches'          // RegExp(value).test(field)
  | 'between';         // value[0] <= field <= value[1]
```

</details>

### Example: Low-Quality Post Detector

<details>
<summary>View Low-Quality Post Detector Config</summary>

```json
{
  "app": {
    "name": "Low-Quality Post Detector",
    "description": "Automatically flags suspicious posts for review",
    "version": "1.0"
  },

  "trigger": {
    "type": "event",
    "event": "post.created"
  },

  "data_pipeline": [
    {
      "id": "new_post",
      "connector": "posts",
      "filter": { "id": "{{trigger.post_id}}" }
    },
    {
      "id": "author",
      "connector": "users",
      "filter": { "id": "{{new_post.author_id}}" }
    },
    {
      "id": "author_history",
      "connector": "posts",
      "filter": { "author_id": "{{author.id}}" },
      "limit": 10
    }
  ],

  "actions": [
    {
      "type": "ai_analyze",
      "params": {
        "input": "{{new_post}}",
        "prompt": "Analyze this post for quality issues: spam patterns, very short content (<20 chars), repeated characters, inappropriate language. Return { is_low_quality: boolean, reasons: string[], confidence: number }",
        "output_var": "analysis"
      }
    },
    {
      "type": "flag_content",
      "condition": {
        "field": "analysis.is_low_quality",
        "operator": "equals",
        "value": true
      },
      "params": {
        "content_type": "post",
        "content_id": "{{new_post.id}}",
        "reason": "{{analysis.reasons}}",
        "set_status": "PENDING"
      }
    },
    {
      "type": "send_notification",
      "condition": {
        "field": "analysis.is_low_quality",
        "operator": "equals",
        "value": true
      },
      "params": {
        "to": "admins",
        "title": "Post flagged for review",
        "message": "Post by {{author.email}} flagged: {{analysis.reasons}}"
      }
    }
  ]
}
```

</details>

### Example: Weekly Engagement Digest

<details>
<summary>View Weekly Engagement Digest Config</summary>

```json
{
  "app": {
    "name": "Weekly Engagement Digest",
    "description": "Personalized weekly email to active users",
    "version": "1.0"
  },

  "trigger": {
    "type": "scheduled",
    "cron": "0 9 * * MON",
    "timezone": "UTC"
  },

  "data_pipeline": [
    {
      "id": "active_users",
      "connector": "users",
      "filter": {
        "status": "Active",
        "has_email": true,
        "last_active": { "$gte": "-30d" }
      }
    },
    {
      "id": "user_stats",
      "connector": "user_engagement",
      "join_on": "active_users.id",
      "fields": ["xp_this_week", "posts_this_week", "missions_completed_this_week"]
    },
    {
      "id": "available_missions",
      "connector": "missions",
      "filter": { "is_active": true }
    }
  ],

  "actions": [
    {
      "type": "for_each",
      "params": {
        "items": "active_users",
        "as": "user",
        "do": [
          {
            "type": "ai_generate",
            "params": {
              "prompt": "Write a friendly weekly digest email for {{user.first_name}}. This week they earned {{user.xp_this_week}} XP, made {{user.posts_this_week}} posts. Suggest 2-3 missions from: {{available_missions}}. Keep it under 150 words, encouraging tone.",
              "output_var": "email_body"
            }
          },
          {
            "type": "send_email",
            "params": {
              "to": "{{user.email}}",
              "subject": "Your weekly community update, {{user.first_name}}!",
              "body": "{{email_body}}",
              "template": "digest"
            }
          }
        ]
      }
    }
  ]
}
```

</details>

---

## Variable Interpolation

Apps use **variable interpolation** to reference data from the data pipeline, trigger events, and action outputs. The syntax is `{{variable}}` using double curly braces.

### Basic Syntax

```
{{variable_name}}           → Simple variable reference
{{object.property}}         → Dot notation for nested access
{{array[0]}}                → Array index access
{{object.items[0].name}}    → Combined nested + array access
```

### Variable Sources

| Source | Syntax | Example |
|--------|--------|---------|
| **Data pipeline step** | `{{step_id.field}}` | `{{active_users.email}}` |
| **Trigger payload** | `{{trigger.field}}` | `{{trigger.post_id}}` |
| **Loop variable** | `{{loop_var.field}}` | `{{user.first_name}}` (inside for_each) |
| **Action output** | `{{output_var}}` | `{{email_body}}` (from ai_generate) |
| **App metadata** | `{{app.field}}` | `{{app.name}}` |
| **Current context** | `{{now}}`, `{{tenant_id}}` | Execution timestamp, current tenant |

### Default Values

Use the `|` pipe operator to provide fallback values:

```
{{user.nickname | user.first_name}}      → Use nickname, fallback to first_name
{{user.avatar | "default.png"}}          → Use avatar, fallback to literal string
{{stats.posts_count | 0}}                → Use posts_count, fallback to 0
```

### Filters & Transformations

Apply transformations using the `:` colon operator:

```
{{user.first_name:uppercase}}            → JOHN
{{user.email:lowercase}}                 → john@example.com
{{created_at:date("YYYY-MM-DD")}}        → 2025-01-29
{{amount:number("0.00")}}                → 1234.56
{{items:count}}                          → 5 (array length)
{{content:truncate(100)}}                → First 100 chars...
{{list:join(", ")}}                      → item1, item2, item3
{{value:json}}                           → JSON stringify
```

### Available Filters

| Filter | Description | Example |
|--------|-------------|---------|
| `uppercase` | Convert to uppercase | `{{name:uppercase}}` → "JOHN" |
| `lowercase` | Convert to lowercase | `{{email:lowercase}}` → "john@example.com" |
| `capitalize` | Capitalize first letter | `{{name:capitalize}}` → "John" |
| `trim` | Remove whitespace | `{{input:trim}}` |
| `date(format)` | Format date | `{{created_at:date("MMM D, YYYY")}}` → "Jan 29, 2025" |
| `number(format)` | Format number | `{{price:number("0,0.00")}}` → "1,234.56" |
| `count` | Array length | `{{users:count}}` → 42 |
| `first` | First array element | `{{items:first}}` |
| `last` | Last array element | `{{items:last}}` |
| `join(sep)` | Join array | `{{tags:join(", ")}}` → "a, b, c" |
| `truncate(n)` | Truncate string | `{{content:truncate(50)}}` → "First 50 chars..." |
| `json` | JSON stringify | `{{object:json}}` |
| `default(val)` | Default if null | `{{name:default("Anonymous")}}` |

### Escaping

To output literal `{{` without interpolation, use double brackets:

```
{{{{variable}}}}  → Outputs: {{variable}}
```

### Interpolation in Different Contexts

**In filter values:**
```json
{
  "filter": { "user_id": "{{trigger.user_id}}" }
}
```

**In action params:**
```json
{
  "type": "send_email",
  "params": {
    "to": "{{user.email}}",
    "subject": "Hello {{user.first_name}}!"
  }
}
```

**In AI prompts:**
```json
{
  "type": "ai_generate",
  "params": {
    "prompt": "Write a welcome message for {{user.first_name}} who joined {{user.created_at:date('MMMM YYYY')}}."
  }
}
```

### Variable Scope

Variables are scoped hierarchically:

1. **Global scope**: `trigger.*`, `app.*`, `now`, `tenant_id`
2. **Pipeline scope**: Each data step adds its `step_id` to scope
3. **Loop scope**: `for_each` adds its `as` variable (shadows outer variables)
4. **Action scope**: `output_var` from previous actions

```json
{
  "data_pipeline": [
    { "id": "users", "connector": "users" }
  ],
  "actions": [
    {
      "type": "for_each",
      "params": {
        "items": "users",
        "as": "user",
        "do": [
          {
            "type": "ai_generate",
            "params": {
              "prompt": "Welcome {{user.first_name}}",
              "output_var": "welcome_msg"
            }
          },
          {
            "type": "send_email",
            "params": {
              "to": "{{user.email}}",
              "body": "{{welcome_msg}}"
            }
          }
        ]
      }
    }
  ]
}
```

---

## App Configuration Validation

Before an AI-generated app configuration is saved, it must pass validation. This ensures apps are safe, well-formed, and within limits.

### Validation Pipeline

```
AI generates config → Schema Validation → Semantic Validation → Limit Checks → Save
```

### Schema Validation

Validates that the configuration matches the expected TypeScript interface:

<details>
<summary>View schema validator (<code>AppConfigValidator</code>)</summary>

```typescript
class AppConfigValidator {
  validateSchema(config: unknown): ValidationResult {
    // 1. Required fields
    if (!config.trigger) return { valid: false, errors: ['Missing trigger'] };
    if (!config.actions || config.actions.length === 0) {
      return { valid: false, errors: ['At least one action required'] };
    }

    // 2. Trigger type validation
    const validTriggers = ['scheduled', 'event', 'delayed', 'manual'];
    if (!validTriggers.includes(config.trigger.type)) {
      return { valid: false, errors: [`Invalid trigger type: ${config.trigger.type}`] };
    }

    // 3. Action type validation
    for (const action of config.actions) {
      if (!VALID_ACTION_TYPES.includes(action.type)) {
        return { valid: false, errors: [`Invalid action type: ${action.type}`] };
      }
    }

    // 4. Connector type validation
    for (const step of config.data_pipeline || []) {
      if (!VALID_CONNECTOR_TYPES.includes(step.connector)) {
        return { valid: false, errors: [`Invalid connector: ${step.connector}`] };
      }
    }

    return { valid: true, errors: [] };
  }
}
```

</details>

### Semantic Validation

Validates that the configuration makes logical sense:

<details>
<summary>View semantic validator (<code>SemanticValidator</code>)</summary>

```typescript
class SemanticValidator {
  validate(config: AppConfig): ValidationResult {
    const errors: string[] = [];

    // 1. Variable references exist
    const definedVariables = this.collectDefinedVariables(config);
    const usedVariables = this.extractUsedVariables(config);

    for (const variable of usedVariables) {
      if (!definedVariables.includes(variable) && !this.isBuiltIn(variable)) {
        errors.push(`Undefined variable: {{${variable}}}`);
      }
    }

    // 2. Cron expression is valid (for scheduled triggers)
    if (config.trigger.type === 'scheduled') {
      if (!this.isValidCron(config.trigger.cron)) {
        errors.push(`Invalid cron expression: ${config.trigger.cron}`);
      }
    }

    // 3. Event exists (for event triggers)
    if (config.trigger.type === 'event') {
      if (!VALID_EVENTS.includes(config.trigger.event)) {
        errors.push(`Unknown event: ${config.trigger.event}`);
      }
    }

    // 4. Email has required fields
    for (const action of this.findActions(config, 'send_email')) {
      if (!action.params.to) errors.push('send_email missing "to" parameter');
      if (!action.params.subject) errors.push('send_email missing "subject" parameter');
      if (!action.params.body) errors.push('send_email missing "body" parameter');
    }

    // 5. for_each has items and nested actions
    for (const action of this.findActions(config, 'for_each')) {
      if (!action.params.items) errors.push('for_each missing "items" parameter');
      if (!action.params.as) errors.push('for_each missing "as" parameter');
      if (!action.params.do || action.params.do.length === 0) {
        errors.push('for_each missing nested actions in "do"');
      }
    }

    return { valid: errors.length === 0, errors };
  }
}
```

</details>

### Limit Validation

Ensures the app is within resource limits:

<details>
<summary>View limit validator (<code>LimitValidator</code>)</summary>

```typescript
class LimitValidator {
  async validate(config: AppConfig, tenant_id: string): Promise<ValidationResult> {
    const errors: string[] = [];

    // 1. Pipeline steps limit
    if ((config.data_pipeline?.length || 0) > MAX_PIPELINE_STEPS) {
      errors.push(`Too many data pipeline steps (max: ${MAX_PIPELINE_STEPS})`);
    }

    // 2. Actions limit
    const actionCount = this.countActions(config.actions);
    if (actionCount > MAX_ACTIONS_PER_APP) {
      errors.push(`Too many actions (max: ${MAX_ACTIONS_PER_APP})`);
    }

    // 3. Nesting depth limit
    const depth = this.maxNestingDepth(config.actions);
    if (depth > MAX_NESTED_LOOPS) {
      errors.push(`Nesting too deep (max: ${MAX_NESTED_LOOPS} levels)`);
    }

    // 4. Apps per tenant limit
    const existingCount = await this.appService.count({ where: { tenant_id } });
    if (existingCount >= MAX_APPS_PER_TENANT) {
      errors.push(`App limit reached (max: ${MAX_APPS_PER_TENANT} per tenant)`);
    }

    return { valid: errors.length === 0, errors };
  }
}
```

</details>

### Validation Response

When validation fails, AI receives detailed error messages:

<details>
<summary>View <code>ValidationResult</code> format + example</summary>

```typescript
interface ValidationResult {
  valid: boolean;
  errors: string[];
  warnings?: string[];  // Non-fatal issues
  suggestions?: string[]; // Improvement suggestions
}

// Example failure response to AI:
{
  "valid": false,
  "errors": [
    "Undefined variable: {{user.nickname}} - did you mean {{user.first_name}}?",
    "send_email missing 'subject' parameter"
  ],
  "warnings": [
    "for_each loop may process many items - consider adding a limit filter"
  ]
}
```

</details>

---

## Data Connectors

### Available Connectors

| Connector | Source | Available Fields |
|-----------|--------|------------------|
| `users` | User table | id, email, first_name, last_name, status, created_at, last_active |
| `user_engagement` | Calculated | total_xp, xp_this_week, posts_count, missions_completed, votes_given |
| `posts` | Post table | id, content, title, status, author_id, room_id, created_at, votes_count |
| `rooms` | Room table | id, name, description, member_count, post_count |
| `room_members` | UserRoom table | user_id, room_id, joined_at, is_admin |
| `missions` | Mission table | id, title, description, xp, mission_type, is_active |
| `user_missions` | UserMission table | user_id, mission_id, is_completed, completed_at |
| `contributions` | Contribution table | id, title, reward_type, is_active |
| `reward_claims` | ContributionRewardHistory | user_id, contribution_id, claimed_at, status |
| `shopify_orders` | Shopify API | order_id, customer_email, total, items, created_at |
| `shopify_products` | Shopify API | product_id, title, price, inventory |
| `analytics` | BigQuery/Calculated | daily_active_users, posts_per_day, engagement_rate |
| `notifications` | Notification table | id, user_id, title, read_at |

### Connector Interface

```typescript
interface DataConnector {
  name: string;

  // Fetch data with filters
  fetch(
    connection: Connection,
    filter?: Record<string, any>,
    options?: { limit?: number; offset?: number; fields?: string[] }
  ): Promise<any[]>;

  // Get available fields
  getFields(): FieldDefinition[];

  // Validate filter
  validateFilter(filter: Record<string, any>): ValidationResult;
}
```

---

## Logic Transformations

The data pipeline supports transformations to filter, sort, group, and aggregate data between connector fetches and action execution.

### Available Logic Operations

| Operation | Description | Example |
|-----------|-------------|---------|
| `filter` | Keep items matching criteria | Keep users with XP > 100 |
| `sort` | Order items by field | Sort by created_at descending |
| `group` | Group items by field | Group posts by room_id |
| `aggregate` | Calculate stats | Count, sum, average, min, max |
| `merge` | Combine data sources | Join users with their stats |
| `limit` | Take first N items | First 100 users |
| `map` | Transform each item | Extract specific fields |

### Pipeline Step Configuration

```typescript
interface DataStepConfig {
  id: string;
  connector: ConnectorType;
  filter?: FilterConfig;       // Filter at connector level
  fields?: string[];           // Select specific fields
  limit?: number;

  // Post-fetch transformations
  transform?: TransformConfig[];
}

interface TransformConfig {
  operation: 'filter' | 'sort' | 'group' | 'aggregate' | 'merge' | 'limit' | 'map';
  params: Record<string, any>;
}
```

### Filter Syntax

Filters support comparison operators and logical combinations:

```typescript
interface FilterConfig {
  // Simple equality
  field?: any;
  // { "status": "Active" } → WHERE status = 'Active'

  // Comparison operators
  $gt?: any;   // Greater than
  $gte?: any;  // Greater than or equal
  $lt?: any;   // Less than
  $lte?: any;  // Less than or equal
  $ne?: any;   // Not equal
  $in?: any[]; // In array
  $nin?: any[]; // Not in array
  $like?: string; // LIKE pattern
  $ilike?: string; // Case-insensitive LIKE

  // Logical operators
  $and?: FilterConfig[];
  $or?: FilterConfig[];
  $not?: FilterConfig;
}

// Examples:
{
  "filter": {
    "status": "Active",
    "xp": { "$gte": 100 },
    "created_at": { "$gte": "-30d" }  // Relative date
  }
}

{
  "filter": {
    "$or": [
      { "role": "admin" },
      { "xp": { "$gte": 1000 } }
    ]
  }
}
```

### Sort Syntax

```typescript
interface SortConfig {
  field: string;
  order: 'asc' | 'desc';
}

// In transform:
{
  "operation": "sort",
  "params": {
    "by": [
      { "field": "xp", "order": "desc" },
      { "field": "created_at", "order": "asc" }
    ]
  }
}
```

### Group & Aggregate

```typescript
// Group users by room and count
{
  "operation": "group",
  "params": {
    "by": "room_id",
    "aggregations": [
      { "field": "id", "function": "count", "as": "user_count" },
      { "field": "xp", "function": "sum", "as": "total_xp" },
      { "field": "xp", "function": "avg", "as": "avg_xp" }
    ]
  }
}

// Available aggregation functions:
// count, sum, avg, min, max, first, last
```

### Merge (Join) Data Sources

```typescript
// Join user stats with users
{
  "data_pipeline": [
    {
      "id": "users",
      "connector": "users",
      "filter": { "status": "Active" }
    },
    {
      "id": "stats",
      "connector": "user_engagement",
      "transform": [
        {
          "operation": "merge",
          "params": {
            "with": "users",
            "on": { "left": "user_id", "right": "id" },
            "type": "left"  // left, inner, outer
          }
        }
      ]
    }
  ]
}
```

### Map (Transform)

```typescript
// Extract and rename fields
{
  "operation": "map",
  "params": {
    "fields": {
      "name": "{{first_name}} {{last_name}}",
      "email": "email",
      "engagement_score": "{{xp * 0.1 + posts_count * 5}}"
    }
  }
}
```

### Example: Complex Data Pipeline

```json
{
  "data_pipeline": [
    {
      "id": "active_users",
      "connector": "users",
      "filter": {
        "status": "Active",
        "created_at": { "$gte": "-30d" }
      },
      "transform": [
        {
          "operation": "sort",
          "params": { "by": [{ "field": "xp", "order": "desc" }] }
        },
        {
          "operation": "limit",
          "params": { "count": 100 }
        }
      ]
    },
    {
      "id": "user_stats",
      "connector": "user_engagement",
      "transform": [
        {
          "operation": "merge",
          "params": {
            "with": "active_users",
            "on": { "left": "user_id", "right": "id" }
          }
        },
        {
          "operation": "filter",
          "params": {
            "posts_this_week": { "$gte": 1 }
          }
        }
      ]
    }
  ]
}
```

---

## Action Executors

### Available Actions

| Action | Description | Parameters |
|--------|-------------|------------|
| `send_email` | Send email via SendGrid | to, subject, body, template? |
| `create_post` | Create a post in a room | room_id, content, author_id, title?, is_pinned? |
| `send_notification` | Push notification | to (user_ids or "admins"), title, message |
| `update_user` | Update user fields | user_id, fields to update |
| `create_mission` | Create a mission | title, description, xp, mission_type |
| `award_xp` | Give XP to user | user_id, amount, reason |
| `flag_content` | Flag post/comment | content_type, content_id, reason, set_status |
| `webhook` | Call external URL | url, method, body, headers |
| `for_each` | Loop over items | items, as, do (nested actions) |
| `ai_analyze` | AI analysis | input, prompt, output_var |
| `ai_generate` | AI content generation | prompt, output_var, max_tokens? |

### Action Interface

<details>
<summary>View <code>ActionExecutor</code> interface</summary>

```typescript
interface ActionExecutor {
  type: string;

  // Execute the action
  execute(
    params: Record<string, any>,
    context: ExecutionContext
  ): Promise<ActionResult>;

  // Validate parameters
  validateParams(params: Record<string, any>): ValidationResult;
}

interface ExecutionContext {
  connection: Connection;       // Tenant database
  variables: Record<string, any>; // Data from pipeline
  trigger_data: any;            // Data from trigger event
  app: App;                     // App configuration
}
```

</details>

### Webhook Security

The `webhook` action calls external URLs, which requires strict security controls to prevent SSRF (Server-Side Request Forgery) and other attacks.

#### URL Allowlist

Webhooks can only call pre-approved domains:

<details>
<summary>View webhook allowlist / blocklist config</summary>

```typescript
interface WebhookConfig {
  // Tenant-level allowlist (set by admin)
  allowed_domains: string[];  // e.g., ["api.slack.com", "hooks.zapier.com"]

  // Global blocklist (always enforced)
  blocked_patterns: string[]; // Internal IPs, localhost, cloud metadata
}

// Default blocked patterns (cannot be overridden):
const BLOCKED_PATTERNS = [
  '127.0.0.1', 'localhost', '0.0.0.0',
  '10.*', '172.16.*', '172.17.*', '172.18.*', '172.19.*',
  '172.20.*', '172.21.*', '172.22.*', '172.23.*', '172.24.*',
  '172.25.*', '172.26.*', '172.27.*', '172.28.*', '172.29.*',
  '172.30.*', '172.31.*', '192.168.*',
  '169.254.169.254',  // AWS metadata
  'metadata.google.internal',  // GCP metadata
];
```

</details>

#### Webhook Action Security

<details>
<summary>View Webhook Security Implementation</summary>

```typescript
class WebhookAction implements ActionExecutor {
  async execute(params: WebhookParams, context: ExecutionContext) {
    const { url, method, body, headers } = params;

    // 1. Validate URL against allowlist
    const domain = new URL(url).hostname;
    if (!this.isAllowed(domain, context.tenant_id)) {
      throw new AppExecutionError(
        `Domain "${domain}" not in webhook allowlist. ` +
        `Add it in Settings > Webhook Domains.`
      );
    }

    // 2. Block internal/private IPs
    if (this.isBlocked(url)) {
      throw new AppExecutionError('URL blocked for security reasons');
    }

    // 3. Enforce timeout
    const response = await this.httpClient.request({
      url,
      method: method || 'POST',
      data: body,
      headers: {
        'Content-Type': 'application/json',
        'User-Agent': 'Decommerce-App-Framework/1.0',
        ...headers,
      },
      timeout: WEBHOOK_TIMEOUT_MS,  // 30 seconds max
    });

    // 4. Don't follow redirects to blocked URLs
    if (response.redirected && this.isBlocked(response.url)) {
      throw new AppExecutionError('Redirect to blocked URL');
    }

    return { status: response.status, body: response.data };
  }
}
```

</details>

#### Webhook Limits

| Limit | Value | Description |
|-------|-------|-------------|
| `WEBHOOK_TIMEOUT_MS` | 30,000 | Max time to wait for response |
| `WEBHOOK_MAX_RETRIES` | 2 | Retry attempts on failure |
| `WEBHOOK_MAX_BODY_SIZE` | 1 MB | Maximum request body size |
| `WEBHOOK_RATE_LIMIT` | 100/hour | Max webhook calls per tenant per hour |

#### Webhook Logging

All webhook calls are logged for audit:

<details>
<summary>View <code>WebhookLog</code> interface</summary>

```typescript
interface WebhookLog {
  app_id: number;
  execution_id: number;
  url: string;
  method: string;
  status_code: number;
  response_time_ms: number;
  success: boolean;
  error?: string;
  timestamp: Date;
}
```

</details>

---

## Triggers System

### Trigger Types

| Type | Description | Configuration |
|------|-------------|---------------|
| `scheduled` | Cron-based | `{ cron: "0 9 * * MON", timezone: "UTC" }` |
| `event` | On platform event | `{ event: "user.created", filter?: {...} }` |
| `delayed` | Event + delay | `{ event: "user.created", delays: ["1d", "3d", "7d"] }` |
| `manual` | Admin clicks "Run" | `{}` |

### Delayed Trigger (for Welcome Series)

The `delayed` trigger is specifically designed for multi-step sequences like welcome email series. It fires at specified intervals after an event occurs.

**Use case:** Send welcome emails on Day 1, Day 3, and Day 7 after signup.

<details>
<summary>View <code>DelayedTriggerConfig</code> schema</summary>

```typescript
interface DelayedTriggerConfig {
  type: 'delayed';
  event: EventType;           // Base event to trigger the sequence
  delays: DelaySpec[];        // When to execute after the event
  filter?: Record<string, any>; // Optional filter on the event
}

type DelaySpec = string | {
  delay: string;              // e.g., "1d", "3d", "7d", "24h", "30m"
  condition?: ConditionConfig; // Optional: only run if condition met
};

// Examples:
// "1d" = 1 day after event
// "3d" = 3 days after event
// "7d" = 7 days after event
// "24h" = 24 hours after event
// "30m" = 30 minutes after event
```

</details>

**Example: Welcome Email Series**

```json
{
  "app": {
    "name": "New User Welcome Series",
    "description": "3-email series for new users"
  },
  "trigger": {
    "type": "delayed",
    "event": "user.created",
    "delays": [
      { "delay": "1d", "condition": null },
      { "delay": "3d", "condition": { "field": "user.is_active", "operator": "equals", "value": true } },
      { "delay": "7d", "condition": { "field": "user.missions_completed", "operator": "less_than", "value": 3 } }
    ]
  },
  "data_pipeline": [
    {
      "id": "user",
      "connector": "users",
      "filter": { "id": "{{trigger.user_id}}" }
    }
  ],
  "actions": [
    {
      "type": "ai_generate",
      "params": {
        "prompt": "Write welcome email #{{trigger.delay_index}} for {{user.first_name}}",
        "output_var": "email_body"
      }
    },
    {
      "type": "send_email",
      "params": {
        "to": "{{user.email}}",
        "subject": "{{trigger.delay_index == 0 ? 'Welcome!' : trigger.delay_index == 1 ? 'How are you doing?' : 'We miss you!'}}",
        "body": "{{email_body}}"
      }
    }
  ]
}
```

**How Delayed Triggers Work:**

<details>
<summary>View Delayed Trigger Implementation</summary>

```typescript
@Entity('delayed_executions')
export class DelayedExecution {
  @PrimaryGeneratedColumn()
  id: number;

  @ManyToOne(() => App)
  app: App;

  @Column({ type: 'jsonb' })
  trigger_data: any;  // Original event data

  @Column({ type: 'int' })
  delay_index: number;  // Which delay in the sequence (0, 1, 2...)

  @Column({ type: 'timestamp' })
  scheduled_for: Date;  // When to execute

  @Column({ type: 'enum', enum: DelayedStatus })
  status: DelayedStatus;  // PENDING, EXECUTED, CANCELLED, SKIPPED
}

@Injectable()
export class DelayedTriggerService {
  // Called when base event fires
  async scheduleDelayedExecutions(app: App, eventPayload: any) {
    const { delays } = app.config.trigger;
    const baseTime = new Date();

    for (let i = 0; i < delays.length; i++) {
      const delay = delays[i];
      const scheduledFor = this.addDelay(baseTime, delay.delay || delay);

      await this.delayedExecutionRepo.save({
        app,
        trigger_data: { ...eventPayload, delay_index: i },
        delay_index: i,
        scheduled_for: scheduledFor,
        status: DelayedStatus.PENDING,
      });
    }
  }

  // Cron job runs every minute to check for due executions
  @Cron('* * * * *')
  async processDelayedExecutions() {
    const dueExecutions = await this.delayedExecutionRepo.find({
      where: {
        scheduled_for: LessThanOrEqual(new Date()),
        status: DelayedStatus.PENDING,
      },
    });

    for (const execution of dueExecutions) {
      // Check condition if specified
      const delay = execution.app.config.trigger.delays[execution.delay_index];
      if (delay.condition) {
        const conditionMet = await this.evaluateCondition(
          delay.condition,
          execution.trigger_data
        );
        if (!conditionMet) {
          execution.status = DelayedStatus.SKIPPED;
          await this.delayedExecutionRepo.save(execution);
          continue;
        }
      }

      // Execute the app
      await this.appExecutorService.execute(execution.app, {
        trigger_data: execution.trigger_data,
      });

      execution.status = DelayedStatus.EXECUTED;
      await this.delayedExecutionRepo.save(execution);
    }
  }
}
```

</details>

**Cancellation:** If the user completes a specific action (like making a purchase), you can cancel remaining delayed executions:

```typescript
// Cancel remaining welcome emails if user becomes highly engaged
await delayedTriggerService.cancelPendingExecutions(app.id, user_id);
```

### Available Events

| Event | Fires When | Payload |
|-------|------------|---------|
| `user.created` | New user signs up | `{ user_id, email }` |
| `user.verified` | Email verified | `{ user_id }` |
| `post.created` | New post created | `{ post_id, author_id, room_id }` |
| `post.reported` | Post reported | `{ post_id, reporter_id }` |
| `mission.completed` | User completes mission | `{ user_id, mission_id, xp_earned }` |
| `reward.claimed` | User claims reward | `{ user_id, contribution_id }` |
| `room.joined` | User joins room | `{ user_id, room_id }` |
| `shopify.order.created` | New Shopify order | `{ order_id, customer_email, total }` |
| `shopify.order.fulfilled` | Order fulfilled | `{ order_id, customer_email }` |

### Trigger Filter Syntax

Event triggers can include filters to only fire for specific events. This prevents apps from running on every event.

#### Basic Filter Syntax

<details>
<summary>View event trigger filter schema</summary>

```typescript
interface EventTriggerConfig {
  type: 'event';
  event: EventType;
  filter?: TriggerFilterConfig;
}

interface TriggerFilterConfig {
  // Match payload fields
  [field: string]: any | FilterOperator;
}
```

</details>

#### Examples

**1. Only trigger for posts in a specific room:**

```json
{
  "trigger": {
    "type": "event",
    "event": "post.created",
    "filter": {
      "room_id": 42
    }
  }
}
```

**2. Only trigger for high-value Shopify orders:**

```json
{
  "trigger": {
    "type": "event",
    "event": "shopify.order.created",
    "filter": {
      "total": { "$gte": 100 }
    }
  }
}
```

**3. Only trigger for users with verified email domain:**

```json
{
  "trigger": {
    "type": "event",
    "event": "user.created",
    "filter": {
      "email": { "$like": "%@company.com" }
    }
  }
}
```

**4. Complex filter with OR condition:**

```json
{
  "trigger": {
    "type": "event",
    "event": "mission.completed",
    "filter": {
      "$or": [
        { "xp_earned": { "$gte": 100 } },
        { "mission_id": { "$in": [1, 2, 3] } }
      ]
    }
  }
}
```

#### Available Filter Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `$eq` | Equals (default) | `{ "status": "Active" }` |
| `$ne` | Not equals | `{ "status": { "$ne": "Deleted" } }` |
| `$gt` | Greater than | `{ "total": { "$gt": 50 } }` |
| `$gte` | Greater than or equal | `{ "xp": { "$gte": 100 } }` |
| `$lt` | Less than | `{ "age": { "$lt": 18 } }` |
| `$lte` | Less than or equal | `{ "items": { "$lte": 5 } }` |
| `$in` | In array | `{ "room_id": { "$in": [1, 2, 3] } }` |
| `$nin` | Not in array | `{ "status": { "$nin": ["Banned", "Deleted"] } }` |
| `$like` | SQL LIKE | `{ "email": { "$like": "%@gmail.com" } }` |
| `$exists` | Field exists | `{ "shopify_id": { "$exists": true } }` |
| `$and` | All conditions match | `{ "$and": [{...}, {...}] }` |
| `$or` | Any condition matches | `{ "$or": [{...}, {...}] }` |

#### Filter Evaluation

<details>
<summary>View Filter Evaluation Code</summary>

```typescript
class TriggerFilterEvaluator {
  matches(filter: TriggerFilterConfig, payload: any): boolean {
    for (const [field, condition] of Object.entries(filter)) {
      // Handle logical operators
      if (field === '$and') {
        return condition.every(c => this.matches(c, payload));
      }
      if (field === '$or') {
        return condition.some(c => this.matches(c, payload));
      }

      // Get payload value
      const value = this.getNestedValue(payload, field);

      // Simple equality
      if (typeof condition !== 'object' || condition === null) {
        if (value !== condition) return false;
        continue;
      }

      // Operator-based comparison
      if (!this.evaluateOperator(value, condition)) return false;
    }
    return true;
  }

  private evaluateOperator(value: any, operators: Record<string, any>): boolean {
    for (const [op, expected] of Object.entries(operators)) {
      switch (op) {
        case '$eq': if (value !== expected) return false; break;
        case '$ne': if (value === expected) return false; break;
        case '$gt': if (value <= expected) return false; break;
        case '$gte': if (value < expected) return false; break;
        case '$lt': if (value >= expected) return false; break;
        case '$lte': if (value > expected) return false; break;
        case '$in': if (!expected.includes(value)) return false; break;
        case '$nin': if (expected.includes(value)) return false; break;
        case '$like': if (!this.matchLike(value, expected)) return false; break;
        case '$exists': if ((value !== undefined) !== expected) return false; break;
      }
    }
    return true;
  }
}
```

</details>

### Shopify Webhook Integration

Shopify events reach the App Framework through the existing Shopify webhook integration. Here's how it works:

```
Shopify Store → Webhook → Decommerce Backend → App Framework → Micro-App
```

#### Shopify Webhook Registration

When a tenant connects their Shopify store, Decommerce automatically registers webhooks:

```typescript
// Registered Shopify webhooks:
const SHOPIFY_WEBHOOKS = [
  'orders/create',       // → shopify.order.created
  'orders/fulfilled',    // → shopify.order.fulfilled
  'orders/cancelled',    // → shopify.order.cancelled
  'products/create',     // → shopify.product.created
  'products/update',     // → shopify.product.updated
  'customers/create',    // → shopify.customer.created
  'carts/create',        // → shopify.cart.created (for abandoned cart)
  'carts/update',        // → shopify.cart.updated
];
```

#### Webhook Handler

The existing Shopify controller receives webhooks and translates them to platform events:

<details>
<summary>View Shopify Webhook Handler</summary>

```typescript
@Controller('webhooks/shopify')
export class ShopifyWebhookController {
  @Post(':tenant_id')
  @ShopifyHmacVerified()  // Verify Shopify signature
  async handleWebhook(
    @Param('tenant_id') tenant_id: string,
    @Headers('X-Shopify-Topic') topic: string,
    @Body() payload: any,
  ) {
    // Map Shopify topic to platform event
    const eventMap = {
      'orders/create': 'shopify.order.created',
      'orders/fulfilled': 'shopify.order.fulfilled',
      // ...
    };

    const eventType = eventMap[topic];
    if (!eventType) return;

    // Transform payload to normalized format
    const eventPayload = this.transformPayload(topic, payload, tenant_id);

    // Emit platform event (picked up by App Framework)
    this.eventEmitter.emit(eventType, eventPayload);
  }

  private transformPayload(topic: string, shopifyPayload: any, tenant_id: string) {
    // Normalize Shopify data to platform format
    if (topic === 'orders/create') {
      return {
        tenant_id,
        order_id: shopifyPayload.id,
        customer_email: shopifyPayload.customer?.email,
        customer_name: shopifyPayload.customer?.first_name,
        total: parseFloat(shopifyPayload.total_price),
        currency: shopifyPayload.currency,
        items: shopifyPayload.line_items.map(item => ({
          product_id: item.product_id,
          title: item.title,
          quantity: item.quantity,
          price: parseFloat(item.price),
        })),
        created_at: new Date(shopifyPayload.created_at),
      };
    }
    // ... other transformations
  }
}
```

</details>

#### Matching Shopify Customers to Platform Users

To link Shopify orders to platform users:

```typescript
class ShopifyConnector {
  async matchUserByEmail(email: string, tenant_id: string): Promise<User | null> {
    // Find user by Shopify email
    return this.userRepository.findOne({
      where: { email, tenant_id },
    });
  }
}

// In app execution context, the matched user is available:
{
  "trigger_data": {
    "order_id": 12345,
    "customer_email": "john@example.com",
    "matched_user": {
      "id": 789,
      "first_name": "John",
      "email": "john@example.com"
    }
  }
}
```

#### Shopify-Specific Connectors

| Connector | Description | Filters |
|-----------|-------------|---------|
| `shopify_orders` | Fetch order history | customer_email, created_after, total_gte |
| `shopify_products` | Fetch products | product_id, title, collection_id |
| `shopify_customers` | Fetch Shopify customers | email, created_after |

### Scheduler Implementation

<details>
<summary>View Scheduler Implementation</summary>

```typescript
@Injectable()
export class AppSchedulerService implements OnModuleInit {
  private scheduledJobs: Map<number, CronJob> = new Map();

  async onModuleInit() {
    // Load all active scheduled apps
    const apps = await this.appService.findByTriggerType('scheduled');

    for (const app of apps) {
      this.scheduleApp(app);
    }
  }

  scheduleApp(app: App) {
    const { cron, timezone } = app.config.trigger;

    const job = new CronJob(cron, async () => {
      await this.appExecutorService.execute(app);
    }, null, true, timezone);

    this.scheduledJobs.set(app.id, job);
  }

  unscheduleApp(appId: number) {
    const job = this.scheduledJobs.get(appId);
    if (job) {
      job.stop();
      this.scheduledJobs.delete(appId);
    }
  }
}
```

</details>

### Event Listener Implementation

<details>
<summary>View Event Listener Implementation</summary>

```typescript
@Injectable()
export class AppEventListenerService {
  @OnEvent('user.created')
  async handleUserCreated(payload: UserCreatedEvent) {
    const apps = await this.appService.findByEvent('user.created');

    for (const app of apps) {
      // Check if filter matches
      if (this.matchesFilter(app.config.trigger.filter, payload)) {
        await this.appExecutorService.execute(app, { trigger_data: payload });
      }
    }
  }

  @OnEvent('post.created')
  async handlePostCreated(payload: PostCreatedEvent) {
    const apps = await this.appService.findByEvent('post.created');

    for (const app of apps) {
      if (this.matchesFilter(app.config.trigger.filter, payload)) {
        await this.appExecutorService.execute(app, { trigger_data: payload });
      }
    }
  }

  // ... handlers for other events
}
```

</details>

---

## Execution Safeguards

This section defines safeguards to prevent runaway apps, resource exhaustion, and concurrent execution issues.

### Concurrent Execution Prevention

An app should not run multiple times simultaneously. This prevents data corruption and resource contention.

**Implementation: Execution Lock**

<details>
<summary>View Execution Lock Implementation</summary>

```typescript
interface ExecutionLock {
  app_id: number;
  tenant_id: string;
  locked_at: Date;
  expires_at: Date;  // Auto-expire after max execution time
  execution_id: number;
}

class AppExecutorService {
  async execute(app: App, context?: ExecutionContext): Promise<void> {
    // 1. Try to acquire lock
    const lock = await this.acquireLock(app.id, app.tenant_id);
    if (!lock) {
      console.log(`App ${app.id} is already running, skipping execution`);
      return;
    }

    try {
      // 2. Run the app
      await this.runAppPipeline(app, context);
    } finally {
      // 3. Always release lock
      await this.releaseLock(app.id, app.tenant_id);
    }
  }

  private async acquireLock(appId: number, tenantId: string): Promise<boolean> {
    // Use database advisory lock or Redis SETNX
    // Returns false if app is already locked and lock hasn't expired
  }
}
```

</details>

**Lock Configuration:**
| Setting | Value | Description |
|---------|-------|-------------|
| Lock timeout | 5 minutes | Max time before lock auto-expires |
| Stale lock recovery | On startup | Clean up any orphaned locks |

### Maximum Limits

These limits prevent resource exhaustion:

| Limit | Value | Description |
|-------|-------|-------------|
| `MAX_APPS_PER_TENANT` | 50 | Maximum apps a tenant can create |
| `MAX_PIPELINE_STEPS` | 10 | Maximum data connector steps per app |
| `MAX_ACTIONS_PER_APP` | 20 | Maximum actions per app |
| `MAX_FOR_EACH_ITEMS` | 1000 | Maximum items in a for_each loop |
| `MAX_NESTED_LOOPS` | 2 | Maximum nesting depth for for_each |
| `MAX_EXECUTION_TIME` | 5 minutes | Maximum time for single execution |
| `MAX_AI_CALLS_PER_EXECUTION` | 100 | Maximum AI API calls per execution |
| `MAX_EMAILS_PER_EXECUTION` | 500 | Maximum emails per single execution |
| `MAX_CONSECUTIVE_ERRORS` | 5 | Auto-pause app after consecutive failures |

### Enforcement

<details>
<summary>View Enforcement Code</summary>

```typescript
class AppValidator {
  validate(config: AppConfig): ValidationResult {
    const errors: string[] = [];

    // Check pipeline steps
    if (config.data_pipeline.length > MAX_PIPELINE_STEPS) {
      errors.push(`Too many data pipeline steps (max: ${MAX_PIPELINE_STEPS})`);
    }

    // Check actions count
    const actionCount = this.countActions(config.actions);
    if (actionCount > MAX_ACTIONS_PER_APP) {
      errors.push(`Too many actions (max: ${MAX_ACTIONS_PER_APP})`);
    }

    // Check for_each nesting depth
    const nestingDepth = this.checkNestingDepth(config.actions);
    if (nestingDepth > MAX_NESTED_LOOPS) {
      errors.push(`for_each nesting too deep (max: ${MAX_NESTED_LOOPS})`);
    }

    return { valid: errors.length === 0, errors };
  }
}
```

</details>

### Runtime Safeguards

**For-each loop protection:**

<details>
<summary>View ForEach Protection Code</summary>

```typescript
class ForEachAction {
  async execute(params: ForEachParams, context: ExecutionContext) {
    const items = this.resolveItems(params.items, context);

    // Enforce limit
    if (items.length > MAX_FOR_EACH_ITEMS) {
      throw new AppExecutionError(
        `for_each items exceed limit (${items.length} > ${MAX_FOR_EACH_ITEMS})`
      );
    }

    // Process with progress tracking
    for (let i = 0; i < items.length; i++) {
      // Check execution timeout
      if (context.hasTimedOut()) {
        throw new AppExecutionError('Execution timeout reached');
      }

      await this.executeNestedActions(params.do, {
        ...context,
        [params.as]: items[i],
        loopIndex: i,
      });
    }
  }
}
```

</details>

**Auto-pause on repeated failures:**

<details>
<summary>View Auto-Pause Code</summary>

```typescript
class AppExecutorService {
  async handleExecutionResult(app: App, success: boolean) {
    if (success) {
      // Reset consecutive error count
      await this.appService.update(app.id, { consecutive_errors: 0 });
    } else {
      // Increment consecutive error count
      const newCount = app.consecutive_errors + 1;

      if (newCount >= MAX_CONSECUTIVE_ERRORS) {
        // Auto-pause the app
        await this.appService.update(app.id, {
          status: AppStatus.ERROR,
          consecutive_errors: newCount,
        });

        // Notify admins
        await this.notifyAdmins(app.tenant_id, {
          title: 'App auto-paused',
          message: `App "${app.name}" paused after ${newCount} consecutive failures`,
        });
      } else {
        await this.appService.update(app.id, { consecutive_errors: newCount });
      }
    }
  }
}
```

</details>

---

## Error Recovery & Retry

### Retry Strategy

Failed executions can be automatically retried with exponential backoff:

<details>
<summary>View <code>RetryConfig</code></summary>

```typescript
interface RetryConfig {
  enabled: boolean;
  max_retries: number;       // Default: 3
  initial_delay_ms: number;  // Default: 1000 (1 second)
  max_delay_ms: number;      // Default: 60000 (1 minute)
  backoff_multiplier: number; // Default: 2
}

// Delay calculation: min(initial_delay * (multiplier ^ attempt), max_delay)
// Attempt 1: 1s, Attempt 2: 2s, Attempt 3: 4s, ...
```

</details>

### Partial Execution Handling

When an execution fails mid-way through a for_each loop:

<details>
<summary>View <code>ExecutionCheckpoint</code></summary>

```typescript
interface ExecutionCheckpoint {
  execution_id: number;
  last_completed_action: string;
  last_processed_index: number;
  partial_results: any;
  can_resume: boolean;
}
```

</details>

**Resume behavior:**
- If `can_resume: true`, retry continues from `last_processed_index + 1`
- If `can_resume: false`, entire execution restarts (actions are not idempotent)

### Idempotency Markers

For actions that should not be repeated (like sending emails):

<details>
<summary>View <code>IdempotencyKey</code> + usage example</summary>

```typescript
interface IdempotencyKey {
  execution_id: number;
  action_index: number;
  item_id: string;  // e.g., user_id for emails
}

class SendEmailAction {
  async execute(params: SendEmailParams, context: ExecutionContext) {
    const idempotencyKey = this.generateKey(context, params.to);

    // Check if already sent in this execution
    if (await this.wasSent(idempotencyKey)) {
      return { skipped: true, reason: 'Already sent in this execution' };
    }

    await this.sendEmail(params);
    await this.markSent(idempotencyKey);
  }
}
```

</details>

---

## Implementation Phases

### Phase 1: Foundation (Week 1-2)

**Goal:** MCP Server + Basic App Framework

| Task | Files | Description |
|------|-------|-------------|
| MCP Server setup | `src/mcp/*` | HTTP transport, auth guard, basic tools |
| App entity | `src/app-framework/entities/*` | App, AppExecution tables |
| App service | `src/app-framework/services/app.service.ts` | CRUD for apps |
| App MCP tools | `src/mcp/tools/apps.tools.ts` | create_app, list_apps, etc. |
| Migration | `src/database/migrations/*` | Add apps tables |

**Deliverables:**
- MCP server running at `/v1/mcp`
- Can create/list/delete apps via MCP tools
- Apps stored in database

### Phase 2: Connectors & Actions (Week 2-3)

**Goal:** Data fetching + Action execution

| Task | Files | Description |
|------|-------|-------------|
| Data connectors | `src/app-framework/connectors/*` | users, posts, rooms, missions, etc. |
| Action executors | `src/app-framework/actions/*` | send_email, create_post, etc. |
| App executor | `src/app-framework/services/app-executor.service.ts` | Run app pipeline |
| AI integration | `src/app-framework/logic/ai-generate.logic.ts` | Claude API for content gen |

**Deliverables:**
- Apps can fetch data from all connectors
- Apps can execute all actions
- AI content generation working

### Phase 3: Triggers System (Week 3-4)

**Goal:** Scheduled + Event-based triggers

| Task | Files | Description |
|------|-------|-------------|
| Scheduler | `src/app-framework/services/app-scheduler.service.ts` | Cron-based triggers |
| Event listener | `src/app-framework/services/app-event-listener.service.ts` | Event-based triggers |
| Event emission | Various services | Emit events for app triggers |

**Deliverables:**
- Scheduled apps run on time
- Event-triggered apps run on events
- Manual trigger working

### Phase 4: Admin Panel UI (Week 4-5)

**Goal:** Chat interface + App management

| Task | Files | Description |
|------|-------|-------------|
| AI Chatbox | `admin-panel/src/components/ai-apps/AIChatbox.tsx` | Chat interface |
| App list | `admin-panel/src/components/ai-apps/AppsList.tsx` | View/manage apps |
| App details | `admin-panel/src/components/ai-apps/AppDetails.tsx` | View app config + logs |
| API service | `admin-panel/src/services/ai-apps.service.ts` | Backend API calls |

**Deliverables:**
- Admin can chat with AI to create apps
- Admin can view/manage/pause apps
- Admin can view execution logs

### Phase 5: Testing & Polish (Week 5-6)

**Goal:** Stability + Documentation

| Task | Description |
|------|-------------|
| Unit tests | Test connectors, actions, executor |
| Integration tests | Test full app execution flow |
| Error handling | Graceful failures, retries, alerts |
| Documentation | Admin guide, app examples |
| Performance | Optimize for scale |

---

## Testing & Verification

### Test Cases

| Test | Description | Expected Result |
|------|-------------|-----------------|
| Create app via MCP | AI calls `create_app` tool | App saved to database |
| Scheduled trigger | App with cron trigger | Runs at scheduled time |
| Event trigger | App with `post.created` trigger | Runs when post created |
| Data connector | Fetch users with filter | Returns filtered users |
| Send email action | App sends email | Email delivered via SendGrid |
| AI generate | App generates content | AI returns text |
| For each loop | App processes 100 users | All 100 processed |
| Error handling | Connector fails | Error logged, admin notified |
| App pause/resume | Admin pauses app | App stops running |
| Execution logs | App runs | Logs saved with details |

### Manual Testing Checklist

- [ ] Create "Weekly Digest" app via chatbox
- [ ] Verify app appears in app list
- [ ] Manually trigger app, verify emails sent
- [ ] Create "Post Detector" app
- [ ] Create a test post, verify it's analyzed
- [ ] Pause an app, verify it stops running
- [ ] View execution logs
- [ ] Delete an app

---

## Key Reference Files

### Backend (NestJS)

| File | Purpose |
|------|---------|
| `src/zapier/zapier.module.ts` | Module pattern to follow |
| `src/zapier/zapier.service.ts` | Data query patterns |
| `src/zapier/guards/api-token.guard.ts` | Auth guard pattern |
| `src/klaviyo/klaviyo.service.ts` | External API integration pattern |
| `src/tenancy/tenancy.module.ts` | Multi-tenant connection pattern |
| `src/mail/mail.service.ts` | SendGrid email sending |
| `src/setting/entities/setting.entity.ts` | Add mcp_api_token column |

### Admin Panel (React)

| File | Purpose |
|------|---------|
| `src/components/*` | Component patterns |
| `src/services/*` | API service patterns |
| `src/pages/*` | Page structure |

---

## Rate Limits & Quotas

### Global Rate Limits (Environment Variables)

<details>
<summary>View global rate limit env vars</summary>

```bash
# MCP Server
MCP_ENABLED=true
MCP_RATE_LIMIT_READ=500     # Read operations per hour (global)
MCP_RATE_LIMIT_WRITE=100    # Write operations per hour (global)

# App Framework
APP_FRAMEWORK_ENABLED=true
APP_MAX_PER_TENANT=50
APP_EXECUTION_TIMEOUT=300000  # 5 minutes

# AI Service (Claude)
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-5-sonnet-20241022
AI_MAX_TOKENS=2000
AI_RATE_LIMIT_PER_MINUTE=60  # Global rate limit
```

</details>

### Per-Tenant Rate Limits

Each tenant has individual rate limits to ensure fair resource allocation:

<details>
<summary>View <code>TenantRateLimits</code> interface</summary>

```typescript
interface TenantRateLimits {
  tenant_id: string;

  // AI API limits
  ai_calls_per_hour: number;        // Default: 500
  ai_calls_per_day: number;         // Default: 5000
  ai_tokens_per_day: number;        // Default: 1,000,000

  // App execution limits
  app_executions_per_hour: number;  // Default: 100
  emails_per_day: number;           // Default: 10,000

  // Chatbox limits
  chatbox_messages_per_hour: number; // Default: 50

  // Override timestamp (for temporary increases)
  override_until?: Date;
}
```

</details>

### Rate Limit Tiers

| Tier | AI Calls/Hour | AI Calls/Day | Tokens/Day | App Executions/Hour |
|------|---------------|--------------|------------|---------------------|
| **Free** | 100 | 500 | 100,000 | 20 |
| **Starter** | 300 | 2,000 | 500,000 | 50 |
| **Pro** | 500 | 5,000 | 1,000,000 | 100 |
| **Enterprise** | Custom | Custom | Custom | Custom |

### Rate Limit Enforcement

<details>
<summary>View Rate Limit Service</summary>

```typescript
class RateLimitService {
  async checkLimit(
    tenant_id: string,
    limit_type: 'ai_calls' | 'executions' | 'emails' | 'chatbox'
  ): Promise<{ allowed: boolean; remaining: number; reset_at: Date }> {
    const limits = await this.getTenantLimits(tenant_id);
    const usage = await this.getUsage(tenant_id, limit_type);

    const limit = limits[`${limit_type}_per_hour`];
    const remaining = Math.max(0, limit - usage.count);

    return {
      allowed: remaining > 0,
      remaining,
      reset_at: usage.window_end,
    };
  }

  async incrementUsage(tenant_id: string, limit_type: string): Promise<void> {
    // Use Redis INCR with TTL for sliding window
    const key = `rate_limit:${tenant_id}:${limit_type}:${this.getCurrentWindow()}`;
    await this.redis.incr(key);
    await this.redis.expire(key, 3600);  // 1 hour TTL
  }
}
```

</details>

### Rate Limit Response Headers

API responses include rate limit info:

```
X-RateLimit-Limit: 500
X-RateLimit-Remaining: 342
X-RateLimit-Reset: 1706540400
```

### Quota Exceeded Handling

When a tenant exceeds their quota:

<details>
<summary>View Quota Exceeded Handler</summary>

```typescript
class AppExecutorService {
  async execute(app: App, context: ExecutionContext) {
    const rateCheck = await this.rateLimitService.checkLimit(
      app.tenant_id,
      'ai_calls'
    );

    if (!rateCheck.allowed) {
      // Log the skip
      await this.logExecution(app, {
        status: 'RATE_LIMITED',
        message: `Rate limit exceeded. Resets at ${rateCheck.reset_at}`,
      });

      // Optionally reschedule for later
      if (app.config.settings?.reschedule_on_rate_limit) {
        await this.rescheduleExecution(app, rateCheck.reset_at);
      }

      return;
    }

    // Proceed with execution
    await this.runAppPipeline(app, context);
  }
}
```

</details>

---

## Summary

This system enables **AI to create micro-apps on the fly** for Decommerce tenants:

1. **Admin describes** what they want in natural language via chatbox
2. **AI generates** an app configuration (JSON)
3. **App Framework** stores and runs the app automatically
4. **35+ app types** possible across email, content, engagement, analytics, moderation, Shopify

**Key architectural decisions:**

- **Configuration-based apps** — AI generates JSON config, not code (safe, predictable)
- **Building blocks approach** — Connectors + Actions that AI combines
- **Event-driven architecture** — Leverages existing NestJS EventEmitter
- **Multi-tenant isolation** — Apps scoped to tenant database

**Implementation:** 5-6 weeks across MCP Server, App Framework, Triggers, Admin UI, and Testing.

---

**Document Version**: 2.0
**Last Updated**: January 2025
**Major Changes**:
- v2.0: Complete rewrite for AI-generated micro-apps architecture
- v1.0: Initial MCP connector specification
