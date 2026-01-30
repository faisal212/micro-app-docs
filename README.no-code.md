# Decommerce MCP + AI Micro-Apps — Technical Specification (No-Code Version)

This is a **plain-English** version of the technical specification in `MCP-integration-technical.md`. It explains the **same system and requirements**, but **without any code snippets**.

---

## Table of Contents

1. Executive Summary
2. Vision & Goals
3. What Micro-Apps Can and Cannot Do *(Limitations & Boundaries)*
4. Why MCP + Micro-Apps?
5. System Architecture
6. Part 1: MCP Server
7. Part 2: App Framework
8. Part 3: Admin Panel AI Chatbox
9. Micro-App Catalog
10. App Configuration Schema (Described)
11. Variable Interpolation (Described)
12. App Configuration Validation (Described)
13. Data Connectors (Described)
14. Logic Transformations (Described)
15. Action Executors (Described)
16. Triggers System (Described)
17. Execution Safeguards
18. Error Recovery & Retry
19. Rate Limits & Quotas
20. Implementation Phases
21. Testing & Verification
22. Key Reference Files

---

## Executive Summary

This system allows **AI to create micro-apps on the Decommerce platform**. Unlike Shopify where human developers build apps, here **AI generates apps on the fly** based on admin requests via a chatbox interface.

**The system has three parts:**

| Part | Description | Location |
|------|-------------|----------|
| MCP Server | Protocol layer allowing AI to interact with Decommerce | Backend (NestJS) |
| App Framework | Engine that stores, runs, and manages micro-apps | Backend (NestJS) |
| AI Chatbox | Interface where admins describe apps in natural language | Admin Panel (React) |

**Example flow:**
- Admin requests: “Create an app that detects low-quality posts and flags them for review.”
- AI interprets intent and produces an **app configuration** (not executable code).
- App Framework validates, stores, and activates the app.
- App runs continuously based on its trigger (event-driven/scheduled/manual).

---

## Vision & Goals

### Vision

AI can assemble platform capabilities into “micro-apps” on demand:
Admin asks AI → AI designs the app configuration → app becomes active and runs.

### Goals

- Enable tenant admins to create automation apps via natural language.
- AI generates **configurations** (not application code) that the platform runtime executes.
- Support 30+ app types across: email, content, engagement, analytics, moderation, Shopify.
- Maintain strict **multi-tenant isolation**: each tenant has its own apps and data boundaries.
- Build on existing Decommerce capabilities (users, rooms, posts, missions, rewards, Shopify, email).

### Non-Goals (Out of Scope)

- AI writing real application code (new React components, new backend modules).
- Apps requiring new database tables or new UI pages/components.
- Major platform features unrelated to automation (payments, DM, video chat, etc.).

---

## What Micro-Apps Can and Cannot Do

This section sets clear expectations about the capabilities and boundaries of the micro-app system.

### What Micro-Apps CAN Do

| Capability | Examples |
|------------|----------|
| **Fetch platform data** | Get users, posts, missions, Shopify orders, engagement metrics |
| **Send communications** | Emails, push notifications, in-app notifications |
| **Create content** | Posts, comments, missions, rewards |
| **Update records** | User fields, post status, content flags |
| **Use AI** | Generate personalized text, analyze content quality, make decisions |
| **Run on schedules** | Daily, weekly, monthly, custom cron expressions |
| **React to events** | User signup, post created, order placed, mission completed |
| **Loop over items** | Process up to 1,000 users/posts/orders per execution |
| **Apply conditions** | Only act when specific criteria are met |
| **Call external URLs** | Webhooks to pre-approved domains |

### What Micro-Apps CANNOT Do

| Limitation | Reason | Alternative |
|------------|--------|-------------|
| **Create custom UI** | Cannot add new pages, buttons, modals, or visual components | Request as core platform feature |
| **Create database tables** | Cannot add new tables or columns to store custom data | Request as core platform feature |
| **Connect to external services** | Cannot integrate with Slack, Discord, Stripe, Mailchimp directly (beyond webhooks) | Request as core platform feature |
| **Process files** | Cannot upload, download, or manipulate images, PDFs, videos | Request as core platform feature |
| **Run complex logic** | No nested if-else chains, custom functions, or programming constructs | Keep apps simple; split into multiple apps |
| **Perform custom calculations** | Limited to built-in aggregations (sum, avg, count, min, max) | Request as core platform feature |
| **Operate in real-time** | Apps are triggered (batch), not live/interactive | Use platform's real-time features |
| **Wait for human approval** | Cannot pause mid-execution for user input | Use confirmation before app creation instead |
| **Trigger other apps** | App A cannot start App B | Create combined app or use events |
| **Rollback on failure** | If action 3 fails, actions 1-2 are NOT undone | Design actions to be safe if incomplete |

### Volume & Performance Boundaries

| Boundary | Typical Limit | Impact |
|----------|---------------|--------|
| **Items per loop** | 1,000 | Cannot email 50,000 users in one run; split across multiple runs |
| **Execution timeout** | 5 minutes | Cannot process very large datasets in one execution |
| **AI calls per execution** | 100 | Cannot generate unique content for thousands of users at once |
| **Emails per execution** | 500 | High-volume campaigns need multiple scheduled runs |
| **Apps per tenant** | 50 | Focus on essential automations |

### Scheduling Boundaries

| Boundary | Impact |
|----------|--------|
| **Minimum interval** | Cannot run more frequently than once per minute |
| **Single timezone** | Each scheduled app uses one timezone; cannot auto-adjust per user |
| **No business-day logic** | Cannot automatically skip weekends/holidays |

### Data Access Boundaries

| Boundary | Impact |
|----------|--------|
| **Read via connectors only** | Cannot run arbitrary database queries |
| **Write via actions only** | Cannot directly UPDATE/DELETE records outside defined actions |
| **Tenant isolation** | Cannot access data from other tenants |
| **Current data only** | Cannot query historical snapshots ("users as of last month") |

### Security Boundaries

| Boundary | Reason |
|----------|--------|
| **No stored secrets** | Cannot save API keys for external services |
| **Webhook allowlist** | Can only call pre-approved external URLs |
| **No code execution** | Cannot run JavaScript, Python, or SQL snippets |
| **No internal network access** | Cannot call localhost, private IPs, or cloud metadata |

### When to Request a Core Feature Instead

If you need any of the following, contact the development team to build it as a platform feature:

- Custom integrations (Slack notifications, Stripe payments, social login)
- New UI components (custom dashboards, widgets, forms)
- Processing large volumes (100K+ records)
- Complex business rules (multi-step approvals, workflow engines)
- File handling (image resizing, PDF generation, imports/exports)
- A/B testing or experimentation
- Custom analytics or reporting dashboards

---

## Why MCP + Micro-Apps?

### Zapier vs MCP vs MCP + Micro-Apps

| Aspect | Zapier | MCP (External) | MCP + Micro-Apps |
|--------|--------|----------------|------------------|
| User | Business users | Developers with AI tools | Tenant admins |
| Interface | Zapier UI | Claude Desktop / ChatGPT | Decommerce admin panel chatbox |
| Creates | Zaps (trigger → action) | One-off operations | Persistent apps |
| Intelligence | None | AI-powered | AI-powered |
| Runs | Zapier servers | On-demand | Decommerce servers |

### Key benefits

- **No-code app creation**: admins describe what they want.
- **AI-powered**: apps can incorporate AI analysis/generation steps.
- **Persistent automation**: runs continuously on events/schedules.
- **Platform differentiation**: a unique, AI-native automation layer.

### What apps can be created?

The catalog includes 35+ examples across:
- Email/communication
- Content automation
- Engagement/gamification
- Analytics/reporting
- Moderation/management
- Shopify integration

---

## System Architecture

### High-level architecture

<pre>
┌─────────────────────────────────────────────────────────────────────────┐
│                         ADMIN PANEL (React)                              │
│                                                                         │
│  - AI Chatbox (streaming conversation)                                  │
│  - App Confirmation (optional “approve tool action?” gate)              │
│  - Apps Management UI (list / details / logs / run / pause / resume)    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      DECOMMERCE BACKEND (NestJS)                         │
│                                                                         │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │
│  │   MCP Server    │   │  App Framework  │   │     AI Service       │   │
│  │ (tools/resources)│  │ (runtime engine)│   │ (Claude or similar)  │   │
│  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │
│           │                     │                       │              │
│           └───────────────┬─────┴───────────────┬───────┘              │
│                           ▼                     ▼                      │
│        Existing Decommerce Services        Tenant DB (PostgreSQL)       │
│     (Users/Rooms/Posts/Missions/etc.)   (apps, executions, logs, data)  │
└─────────────────────────────────────────────────────────────────────────┘
</pre>

- **Admin Panel (React)**
  - AI chatbox for “describe an app” interactions.
  - Apps management UI (list, details, logs, pause/resume, run).
- **Decommerce Backend (NestJS)**
  - MCP Server (tools AI can call)
  - App Framework (storage, triggers, execution, monitoring)
  - AI Service (calls to the model provider, e.g., Claude)
- **Existing Decommerce services**
  - Users, Rooms, Posts, Missions, Rewards, Shopify, Email/Notifications
- **Tenant database (PostgreSQL)**
  - Stores apps, executions, logs, and tenant-scoped business data.

### Request flow: creating an app

<pre>
┌──────────┐   ┌───────────────┐   ┌─────────────┐   ┌──────────────┐   ┌──────────────┐
│  Admin   │   │ Admin Backend │   │ AI Provider │   │  MCP Server   │   │ App Framework │
│  Panel   │   │   (NestJS)    │   │ (e.g Claude)│   │ (tool layer)  │   │ (store+rules) │
└────┬─────┘   └──────┬────────┘   └──────┬──────┘   └──────┬───────┘   └──────┬───────┘
     │                 │                   │                 │                  │
     │ “Create app …”  │                   │                 │                  │
     ├────────────────▶│                   │                 │                  │
     │                 │  chat+tools list  │                 │                  │
     │                 ├──────────────────▶│                 │                  │
     │                 │                   │ decides to use  │                  │
     │                 │                   │ create_app tool │                  │
     │                 │                   ├────────────────▶│                  │
     │                 │                   │                 │ validate+store   │
     │                 │                   │                 ├─────────────────▶│
     │                 │                   │                 │  result (ok/err) │
     │                 │                   │◀────────────────┤                  │
     │      streamed response back to UI (and app appears in list if created)   │
     │◀────────────────────────────────────────────────────────────────────────┘
</pre>

1. Admin sends a natural-language request from the chatbox UI.
2. Backend forwards it to the AI provider with:
   - a system instruction (tenant context, allowed tools),
   - a tool list (MCP tools),
   - and streaming enabled for a good UX.
3. AI decides whether it needs clarifications or is ready to propose a full app.
4. If ready, AI uses an MCP tool to create/store the app configuration.
5. Backend returns success and the new app appears in the admin UI.

### Request flow: app execution

<pre>
┌───────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Trigger   │   │ App Framework │   │  Connectors  │   │   Actions    │
│ (cron/event│  │ (load+execute)│   │  (read data) │   │ (side effects)│
└─────┬─────┘   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
      │                 │                  │                  │
      │ fires            │                  │                  │
      ├────────────────▶│ load app config   │                  │
      │                 ├──────────────────▶│ fetch pipeline    │
      │                 │◀──────────────────┤ results           │
      │                 │ (optional transforms)                  │
      │                 ├───────────────────────────────────────▶│ run actions
      │                 │◀───────────────────────────────────────┤ results/errors
      │                 │ log execution + update counters/status │
</pre>

1. A trigger fires (scheduled time, event emitted, or manual run).
2. App Framework loads the app configuration from storage.
3. It runs the **data pipeline** (connectors + optional transformations).
4. It runs the **actions** (including possible AI calls).
5. It logs execution results for audit/debugging and updates status counters.

---

## Part 1: MCP Server

The MCP Server exposes **tool endpoints** and **resources** that AI can use to interact with Decommerce safely.

### MCP module responsibilities (conceptual)

- Accept MCP requests via HTTP transport.
- Authenticate requests (e.g., API key or tenant-scoped secret).
- Apply rate limiting for read vs write operations.
- Establish per-request tenant context (so every tool call is tenant-scoped).
- Register tools by domain: apps, users, rooms, posts, missions, rewards, notifications, analytics.
- Expose resources (documentation-like endpoints) such as:
  - platform schema overview,
  - available app templates/building blocks,
  - platform enums and event names.

### MCP tools for app management

| Tool | Purpose |
|------|---------|
| create_app | Create a micro-app by saving an app configuration |
| list_apps | List apps for the current tenant (filterable) |
| get_app | Fetch one app plus run history |
| update_app | Update app configuration or metadata (versioned) |
| delete_app | Delete an app |
| run_app | Manually trigger an app execution |
| pause_app | Pause an app |
| resume_app | Resume a paused app |
| get_app_logs | Fetch recent execution logs |

### MCP tools for data access

These are tenant-scoped tools for listing/fetching core entities and metrics, for example:
- list/get users
- list/get rooms
- list/create posts (where appropriate)
- list/create missions
- list contributions/rewards data
- fetch community stats and engagement metrics
- send notifications (to admins or users)

---

## Part 2: App Framework

The App Framework is the runtime engine that stores, schedules, executes, and monitors micro-apps.

### What the App Framework does

1. **Storage**: saves app configurations (as data) in the database.
2. **Triggering**: decides when to run apps (scheduled/event/manual/delayed).
3. **Execution**: runs data pipeline and actions in order.
4. **Monitoring**: logs every execution for audit and debugging.

### Building blocks

Apps are assembled from three building block types:

| Building block | Purpose | Examples |
|---|---|---|
| Connectors | Fetch data | users, posts, missions, shopify orders |
| Actions | Produce side effects | send email, create post, award XP, flag content |
| Triggers | Decide when to run | cron schedule, platform event, manual run |

### Why configuration-based apps

Generating configurations instead of arbitrary code is safer because:
- AI is constrained to approved connectors/actions.
- Every app is inspectable and auditable.
- The runtime can enforce limits and security consistently.
- Apps are easy to pause/modify/delete without redeploying code.

### Core stored entities (described)

- **App**
  - tenant identifier (required for isolation)
  - name/description
  - configuration payload (the full app definition)
  - status (active/paused/error)
  - versioning (current version + recent history)
  - counters and timestamps (run/error counts, last run, consecutive errors)
  - created_by / updated_by for audit

- **App Execution**
  - linked app reference
  - status (running/success/failed)
  - start/end timestamps
  - items processed
  - result payload and/or error message

---

## Part 3: Admin Panel AI Chatbox

The admin panel provides:
- a chat interface to request apps in natural language,
- streaming responses,
- optional confirmation gates for tool execution,
- and an apps management UI (list/detail/logs/controls).

### Endpoints (conceptual)

The backend provides endpoints to:
- send a chat message and stream AI responses,
- list conversations and fetch a single conversation,
- delete conversation history,
- confirm or reject a pending tool execution when confirmation is required.

### Tool confirmation flow (conceptual)

When confirmation is enabled:
- AI proposes an action that would create/update/delete an app.
- Backend pauses and emits a “needs confirmation” event with an app preview.
- Frontend shows a confirmation dialog.
- If admin approves, backend continues and executes the pending tool call.

---

## Micro-App Catalog

This is a reference catalog of example apps by category. Categories include:

### Email & Communication
- Weekly engagement digest
- Win-back campaigns
- Welcome series
- Shopify + community emails
- Mission reminders
- Reward available alerts
- Top contributor spotlights

### Content Automation
- Weekly recap posts
- Auto welcome posts
- Product announcements
- Event reminder posts
- Milestone celebration posts
- Monthly highlights

### Engagement & Gamification
- Campaign builder
- Weekly leaderboard rewards
- Streak bonus system
- Referral booster campaigns
- Room activity challenges
- New user onboarding missions

### Analytics & Reporting
- Weekly admin reports
- Churn risk alerts
- Content performance reports
- Mission effectiveness reports
- Shopify/community correlation
- Room health dashboards

### Moderation & Management
- Low-quality post detector
- Inactive user cleanup
- New user watchlist
- Engagement score calculator
- Auto-archive old rooms

### Shopify Integration
- Purchase thank-you automations
- VIP buyer rewards
- Review requests after purchase
- First purchase celebration
- Abandoned cart recovery

---

## App Configuration Schema (Described)

An app configuration contains:

### 1) App metadata
- name
- optional description
- version string
- optional icon

### 2) Trigger configuration
Supported trigger types:
- **scheduled**: cron expression + optional timezone
- **event**: platform event name + optional filter
- **manual**: run on admin request
- **delayed**: event-based sequence with one or more delays (used for multi-step sequences like welcome series)

### 3) Data pipeline
An ordered list of steps. Each step has:
- a unique step id
- a connector type (which dataset to fetch)
- optional filter criteria
- optional field selection
- optional join/merge behavior (depending on the chosen pipeline design)
- optional limits
- optional post-fetch transformations (see Logic Transformations)

### 4) Actions
An ordered list of actions. Each action has:
- an action type
- optional condition (execute only when condition matches)
- action parameters (inputs)

### 5) Optional settings
Examples of settings:
- enabled/disabled
- retry behavior and retry limits
- notify-on-error

---

## Variable Interpolation (Described)

Configurations use template placeholders to reference values available at runtime.

### Syntax
- Variables are referenced using a double-brace placeholder format.

### Variable sources
- Trigger payload values (e.g., identifiers from an event)
- Data pipeline step outputs (each step id becomes a variable scope)
- Loop variables introduced by for-each style actions
- Outputs produced by earlier actions (e.g., AI text outputs)
- Built-in values like current time and tenant id

### Common features
- Dot notation for nested fields.
- Array indexing where needed.
- Default/fallback values when a field is missing.
- Basic formatting filters (uppercase/lowercase/date/number/truncate/join/json/default).
- Clear scoping rules: global → pipeline → loop → action outputs.

---

## App Configuration Validation (Described)

Before saving an AI-generated configuration, validation must pass:

### 1) Schema validation
Ensures:
- trigger exists and is valid
- at least one action exists
- action types are recognized
- connector types are recognized

### 2) Semantic validation
Ensures:
- all referenced variables are defined in scope (or are built-ins)
- cron expressions are valid for scheduled triggers
- event names are valid for event triggers
- actions include required parameters (e.g., email requires recipient/subject/body)
- loop actions include required loop fields and nested actions

### 3) Limit validation
Enforces platform constraints such as:
- max pipeline steps
- max actions (including nested actions)
- max loop nesting depth
- max apps per tenant

### Validation feedback
If invalid, return:
- a clear list of errors
- optional warnings (non-fatal)
- optional suggestions (how to fix)

---

## Data Connectors (Described)

Connectors are read-only data access building blocks. Each connector defines:
- what entity/metric it reads from
- allowed filter fields/operators
- allowed field selection
- limits/offsets

Example connector families:
- users
- engagement metrics
- posts
- rooms and room members
- missions and user missions
- contributions and reward claims
- Shopify orders and products
- analytics metrics
- notifications

Each connector should also:
- validate filters
- provide field metadata for documentation and AI guidance

---

## Logic Transformations (Described)

Between data fetching and action execution, pipeline steps can apply transformations like:
- filter (keep items matching criteria)
- sort (order by fields)
- group (group by a key)
- aggregate (count/sum/avg/min/max/first/last)
- merge/join (combine sources using keys)
- limit (take first N)
- map (reshape/rename fields and derive new values)

The goal is to let AI compose robust pipelines without needing custom code.

---

## Action Executors (Described)

Actions are side-effect operations or control-flow blocks. Each action type has:
- parameter validation
- execution logic
- structured result output (for downstream steps or logs)

### Common action types
- send email
- create post
- send notification
- update user
- create mission
- award XP
- flag content
- webhook call to external URL
- for-each loop (iterate over items and execute nested actions)
- AI analyze
- AI generate

### Webhook security requirements
Webhook calls must be constrained with:
- allowlisted domains (tenant-level)
- blocked internal/private networks and cloud metadata targets
- strict timeouts
- redirect safety checks
- request/response logging for audit
- call quotas to prevent abuse

---

## Triggers System (Described)

### Trigger types
- **scheduled**: cron-based scheduler triggers execution.
- **event**: platform emits events; event listener triggers matching apps.
- **manual**: admin requests run.
- **delayed**: on an event, schedule one or more future executions at specified delays, with optional conditions per delay.

### Available events (conceptual examples)
Examples include user lifecycle events, content events, mission/reward events, room membership events, and Shopify order events.

### Trigger filtering
Event triggers can include filters to restrict which events qualify, including:
- simple equality matches
- comparison operators (greater/less than, etc.)
- list membership
- pattern matching for strings
- logical combinations (AND/OR)

### Shopify integration
Shopify webhooks are mapped into normalized platform events that can trigger apps. Customer identity matching can be done using a stable key such as email.

---

## Execution Safeguards

### Concurrent execution prevention
- Each app execution must acquire an execution lock.
- If locked, a new run is skipped to prevent overlapping execution.
- Locks must expire to avoid stale lock deadlocks.

### Maximum limits (conceptual)
Enforce limits such as:
- max apps per tenant
- max pipeline steps per app
- max actions per app (including nested)
- max items per loop
- max nested loop depth
- max execution duration
- max AI calls per execution
- max emails per execution
- auto-pause after repeated failures

### Runtime safeguards
- loop execution should track progress and stop when timeouts are reached
- repeated failures should increment counters and transition app to an error/paused state
- notify admins on auto-pauses and severe failures when configured

---

## Error Recovery & Retry

### Retry strategy
Failed executions can retry with exponential backoff. Retry settings are configurable per app (within platform constraints).

### Partial execution handling
When a loop fails mid-way:
- the framework may support checkpointing to resume at the next item
- or may restart from scratch depending on whether actions are idempotent

### Idempotency markers
For actions that must not repeat (like sending emails), use idempotency keys so retries do not duplicate side effects.

---

## Rate Limits & Quotas

### Global rate limits
System-wide limits control:
- MCP read/write tool calls
- app execution duration and concurrency pressure
- AI tokens/calls and streaming capacity

### Per-tenant limits
Tenant-specific quotas ensure fair usage, including:
- AI calls per hour/day and tokens per day
- app executions per hour
- emails per day
- chat messages per hour

### Quota exceeded handling
When quotas are exceeded:
- log that execution was skipped due to rate limiting
- optionally reschedule for later if configured

---

## Implementation Phases

### Phase 1: Foundation (Week 1–2)
- MCP server setup (transport, auth, tool registry, basic rate limiting)
- App entities and storage
- App CRUD service
- App MCP tools for create/list/update/delete/run
- Database migrations

### Phase 2: Connectors & Actions (Week 2–3)
- Implement connectors for core Decommerce data sources
- Implement action executors (email/post/notification/etc.)
- Implement the app executor (pipeline + actions)
- Integrate AI generation/analysis for AI steps

### Phase 3: Triggers System (Week 3–4)
- Scheduler for cron-based apps
- Event listeners for platform events
- Manual run support
- Delayed trigger scheduling and processing

### Phase 4: Admin Panel UI (Week 4–5)
- Chatbox UI with streaming
- Apps list and app details pages
- Execution logs viewer
- Pause/resume/run controls
- Confirmation flow for tool execution

### Phase 5: Testing & Polish (Week 5–6)
- Unit tests for connectors, actions, validation
- Integration tests for full execution flows
- Robust error handling and admin notifications
- Documentation and performance tuning

---

## Testing & Verification

### Test cases (conceptual)
- create app through the AI flow and confirm it is stored
- scheduled trigger runs at the expected time
- event trigger runs when the matching event occurs
- connectors return filtered data correctly
- email and notification actions perform side effects correctly
- AI steps produce outputs and are stored in execution context
- loop actions handle N items and enforce limits
- failures are logged and retries behave as configured
- pause/resume prevents and restores execution as expected
- logs can be queried and displayed in admin UI

### Manual testing checklist (conceptual)
- create a “Weekly Digest” via chatbox and verify it appears
- run it manually and verify outputs (emails/logs)
- create a “Post Detector” and test with a new post event
- pause an app and verify it stops running
- inspect execution logs
- delete an app and verify it disappears and stops executing

---

## Key Reference Files

These are the existing codebase patterns to follow (module, auth guard, multi-tenancy, email integration), plus the target directories where the new work will land:

- Backend (NestJS): module patterns, auth guard patterns, tenancy patterns, mail sending patterns, settings storage for tokens.
- Admin panel (React): component and service patterns for pages, API clients, and streaming UI.

---

## Summary

This system enables **AI-generated micro-apps** on Decommerce by:
- letting admins request automations in natural language,
- having AI output a structured app configuration,
- storing and running it in a controlled runtime with connectors/actions/triggers,
- and enforcing isolation, validation, execution limits, retries, and audit logs.

