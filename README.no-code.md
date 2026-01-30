# Decommerce MCP + AI Micro-Apps — Technical Specification (No-Code Version)

This is a **plain-English** version of the technical specification in `MCP-integration-technical.md`. It explains the **same system and requirements**, but **without any code snippets**.

---

## Table of Contents

1. Executive Summary
2. Vision & Goals
3. What Micro-Apps Can and Cannot Do *(Limitations & Boundaries)*
   - Enhanced Capabilities Details
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
22. Troubleshooting *(NEW)*
23. Key Reference Files

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
- Apps requiring new database tables or custom-coded UI components.
  - *Note: Pre-built configurable widgets (dashboard cards, charts) ARE supported - see Enhanced Capabilities.*
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
| **Loop over items** | Process up to 5,000 users/posts/orders per execution |
| **Apply conditions** | Only act when specific criteria are met |
| **Display UI widgets** | Dashboard cards, stat widgets, charts, tables in designated areas |
| **Trigger other apps** | App A emits event → App B listens and runs |
| **Store app state** | Key-value storage for tracking data between executions |
| **Use pre-built integrations** | Slack, Discord, Mailchimp via platform connectors |
| **Process files (basic)** | Generate PDF from template, resize images via pre-built actions |
| **Request human approval** | Pause execution, await admin approval, then continue |
| **Call external URLs** | Webhooks to pre-approved domains |

### Enhanced Capabilities (Configuration-Based)

These powerful features work through configuration, not custom code:

| Feature | How It Works | Example |
|---------|--------------|---------|
| **UI Widgets** | Select from pre-built widget types, configure data source | Show "Top 10 Users" leaderboard card on dashboard |
| **App Chaining** | Define events apps can emit; other apps listen | "Order Shipped" app triggers "Send Review Request" app |
| **State Storage** | Key-value store per app for persistence | Track "last_processed_date" between runs |
| **Approval Queues** | Action pauses at approval step until admin approves | Email blast waits for admin OK before sending |
| **Compensating Actions** | Define "undo" action if later steps fail | If reward fails, don't send the notification |
| **Pre-built Integrations** | Platform provides connectors for popular services | Send to Slack channel via `slack.send_message` action |
| **Template-based Files** | Generate documents from templates | PDF receipt from order data |

### How Enhanced Capabilities Work

#### State Storage
- **Actions**: `set_state` (save value), `get_state` (retrieve value), `delete_state` (remove key)
- **Scope**: Per-app key-value store (isolated per app)
- **Limits**: 100 keys per app, 10KB max per value
- **Use case**: Store "last_processed_id" to resume where you left off

#### Approval Queues

<pre>
┌─────────┐    ┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   App   │───▶│  Execution  │───▶│   request_   │───▶│   PAUSED    │
│ Trigger │    │   starts    │    │   approval   │    │  (waiting)  │
└─────────┘    └─────────────┘    └──────────────┘    └──────┬──────┘
                                                              │
                                         ┌────────────────────┼────────────────────┐
                                         │                    │                    │
                                         ▼                    ▼                    ▼
                                  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
                                  │   APPROVE   │      │   REJECT    │      │   TIMEOUT   │
                                  │ (continue)  │      │  (cancel)   │      │ (7 days)    │
                                  └──────┬──────┘      └──────┬──────┘      └──────┬──────┘
                                         │                    │                    │
                                         ▼                    ▼                    ▼
                                  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
                                  │  Continue   │      │   Execution │      │   Execution │
                                  │  execution  │      │   cancelled │      │   cancelled │
                                  └─────────────┘      └─────────────┘      └─────────────┘
</pre>

- **Action**: `request_approval` pauses execution
- **Admin sees**: Pending approval in Apps UI with action preview
- **Options**: Approve (continue), Reject (cancel), Modify (edit pending action)
- **Timeout**: 7 days, then auto-cancel

#### App Chaining (Event Emission)

<pre>
┌─────────────────────────────────────────────────────────────────────┐
│                         TENANT SCOPE                                 │
│                                                                     │
│  ┌─────────────┐         emit_event          ┌─────────────┐       │
│  │    APP A    │────────────────────────────▶│  Event Bus  │       │
│  │ "Order      │     "order_processed"       │             │       │
│  │  Processor" │                             └──────┬──────┘       │
│  └─────────────┘                                    │              │
│                                                     │ matches      │
│                          ┌──────────────────────────┼──────────────┐
│                          │                          │              │
│                          ▼                          ▼              │
│                   ┌─────────────┐           ┌─────────────┐       │
│                   │    APP B    │           │    APP C    │       │
│                   │ "Send Review│           │ "Update     │       │
│                   │  Request"   │           │  Analytics" │       │
│                   └─────────────┘           └─────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
</pre>

- **Action**: `emit_event` with custom event name and payload
- **Other apps**: Listen via event trigger with matching event name
- **Scope**: Events are tenant-scoped (cannot cross tenants)
- **Use case**: App A emits "order_processed" → App B listens for "order_processed"

#### Pre-built Integrations

| Integration | Actions Available |
|-------------|-------------------|
| Slack | send_message, send_dm |
| Discord | send_message, send_embed |
| Mailchimp | add_subscriber, update_tags |
| Twilio | send_sms |

#### UI Widgets

| Widget Type | Description | Placement |
|-------------|-------------|-----------|
| stat_card | Single metric with label | Dashboard |
| leaderboard | Top N users table | Dashboard |
| chart_line | Time series chart | Dashboard |
| chart_bar | Bar chart | Dashboard |
| table | Data table with columns | Dashboard |

#### File Processing

| Action | Description |
|--------|-------------|
| generate_pdf | Create PDF from template + data |
| resize_image | Resize image to specified dimensions |

**Templates**: Platform provides built-in templates; custom templates can be uploaded via admin panel.

#### Compensating Actions
- **Definition**: Add `on_failure` block to any action
- **Behavior**: If the action fails, run the compensating action automatically
- **Use case**: If `award_xp` fails, run `send_notification` to alert admin

### What Micro-Apps CANNOT Do (Hard Limits)

These are architectural boundaries that cannot be overcome through configuration:

| Limitation | Reason |
|------------|--------|
| **Write custom code** | Cannot run JavaScript, Python, SQL, or any executable code |
| **Create new database tables** | Cannot define custom schemas or add columns |
| **Build custom UI components** | Cannot create new React/Vue components from scratch |
| **Access external APIs directly** | Must use pre-built platform connectors; no raw HTTP calls |
| **Run indefinitely** | 5-minute maximum execution timeout per run |
| **Bypass tenant isolation** | Cannot access data from other tenants |
| **Operate in real-time** | Apps are triggered (batch), not live/interactive |
| **Run complex nested logic** | No nested if-else chains beyond 2 levels or recursive functions |
| **Custom calculations** | Limited to built-in aggregations (sum, avg, count, min, max) |

### Volume & Performance Boundaries

| Boundary | Limit | Impact |
|----------|-------|--------|
| **Items per loop** | 5,000 | For larger datasets, split across multiple scheduled runs |
| **Execution timeout** | 5 minutes | Long-running tasks should be broken into smaller apps |
| **AI calls per execution** | 500 | Use batch AI processing for high-volume personalization |
| **Emails per execution** | 2,000 | High-volume campaigns can use multiple scheduled runs |
| **Apps per tenant** | 100 | Organize apps by category; archive unused apps |

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

### When to Request a Core Platform Feature

Contact the development team only if you need:

- **New integrations** not yet available (e.g., a new CRM or payment provider)
- **Custom UI components** beyond the pre-built widget library
- **Processing 100K+ records** in a single operation
- **Custom database schemas** for specialized data storage
- **A/B testing or experimentation** frameworks
- **Real-time interactive features** (chat, live updates)

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

#### Building Blocks Relationship

<pre>
┌─────────────────────────────────────────────────────────────────────┐
│                         MICRO-APP                                    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                        TRIGGER                               │   │
│  │    (scheduled | event | manual | delayed)                    │   │
│  │                                                              │   │
│  │    "WHEN should this app run?"                               │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                              │ fires                                │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      CONNECTORS                              │   │
│  │    (users | posts | orders | metrics | ...)                  │   │
│  │                                                              │   │
│  │    "WHAT data does this app need?"                           │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                              │ provides data                        │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                       ACTIONS                                │   │
│  │    (send_email | create_post | award_xp | ai_generate | ...) │   │
│  │                                                              │   │
│  │    "WHAT should this app do with the data?"                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
</pre>

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

### App Lifecycle States

<pre>
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    ▼                                         │
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐     │
│  DRAFT  │───▶│ ACTIVE  │───▶│ PAUSED  │───▶│ ACTIVE  │     │
└─────────┘    └────┬────┘    └────┬────┘    └─────────┘     │
     │              │              │                          │
     │              ▼              │                          │
     │         ┌─────────┐        │                          │
     │         │  ERROR  │◀───────┘                          │
     │         └────┬────┘                                   │
     │              │                                         │
     │              ▼                                         │
     │         ┌─────────┐                                   │
     └────────▶│ DELETED │◀──────────────────────────────────┘
               └─────────┘

State Transitions:
- DRAFT → ACTIVE: Admin enables app
- ACTIVE → PAUSED: Admin pauses or auto-pause on errors
- ACTIVE → ERROR: Consecutive failures exceed threshold
- PAUSED → ACTIVE: Admin resumes
- ERROR → ACTIVE: Admin fixes and resumes
- Any → DELETED: Admin deletes app
</pre>

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

#### Data Pipeline Visualization

<pre>
┌──────────────────────────────────────────────────────────────────────┐
│                        DATA PIPELINE                                  │
│                                                                      │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐   │
│  │ Connector │───▶│ Transform │───▶│ Transform │───▶│  Output   │   │
│  │  (fetch)  │    │ (filter)  │    │  (sort)   │    │  (data)   │   │
│  └───────────┘    └───────────┘    └───────────┘    └─────┬─────┘   │
│       │                                                    │         │
│       │ Example:                                           │         │
│       │ users connector                                    │         │
│       │ → filter: status = "active"                        ▼         │
│       │ → sort: engagement_score DESC              ┌───────────┐    │
│       │ → limit: 10                                │  ACTIONS  │    │
│       │                                            │ (execute) │    │
│       └────────────────────────────────────────────┴───────────┘    │
└──────────────────────────────────────────────────────────────────────┘
</pre>

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

Enforces platform constraints:

| Limit | Value |
|-------|-------|
| Max pipeline steps | 10 |
| Max actions per app | 20 |
| Max nested actions (in loops) | 50 |
| Max loop nesting depth | 2 |
| Max apps per tenant | Per tier (10-unlimited) |
| Max items per loop | 5,000 |

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

### Available Connectors

| Connector | Description | Key Filters |
|-----------|-------------|-------------|
| users | Platform users | status, role, created_at, engagement_score |
| posts | User-generated content | room_id, author_id, created_at, status |
| rooms | Community spaces | status, type, member_count |
| room_members | Room membership data | room_id, user_id, role |
| missions | Available missions | status, type, room_id |
| user_missions | User mission progress | user_id, mission_id, status |
| contributions | User contributions | user_id, type, created_at |
| rewards | Reward claims | user_id, reward_type, claimed_at |
| shopify_orders | Shopify orders | customer_email, status, created_at |
| shopify_products | Shopify products | status, type, collection |
| engagement_metrics | Aggregated metrics | user_id, period, metric_type |
| notifications | Sent notifications | user_id, type, status |
| app_state | Current app's state storage | key |

Each connector also:
- validates filters against allowed operators
- provides field metadata for documentation and AI guidance

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

### Available Actions

| Category | Actions |
|----------|---------|
| **Communication** | send_email, send_notification, send_push |
| **Content** | create_post, update_post, flag_post, delete_post, create_comment |
| **Users** | update_user, award_xp, award_badge, update_engagement_score |
| **Missions** | create_mission, complete_mission, assign_mission |
| **State** | set_state, get_state, delete_state |
| **Flow Control** | for_each, condition, request_approval, emit_event |
| **AI** | ai_analyze, ai_generate, ai_classify |
| **External** | webhook_call |
| **Integrations** | slack.send_message, discord.send_message, mailchimp.add_subscriber, twilio.send_sms |
| **Files** | generate_pdf, resize_image |
| **Compensate** | on_failure (wrapper for any action) |

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

### Available Events

| Category | Event Names |
|----------|-------------|
| **User Lifecycle** | user.created, user.updated, user.deleted, user.login |
| **Content** | post.created, post.updated, post.deleted, post.flagged, comment.created |
| **Missions** | mission.created, mission.completed, mission.expired |
| **Rewards** | reward.claimed, xp.awarded, badge.earned |
| **Rooms** | room.created, room.member_joined, room.member_left |
| **Shopify** | shopify.order.created, shopify.order.fulfilled, shopify.customer.created |
| **Custom** | Any event emitted by another app via `emit_event` action |

### Trigger Filtering

Event triggers can include filters to restrict which events qualify.

#### Filter Operators

| Operator | Description | Example |
|----------|-------------|---------|
| eq | Equals | status eq "active" |
| ne | Not equals | role ne "admin" |
| gt, gte | Greater than (or equal) | age gt 18 |
| lt, lte | Less than (or equal) | score lte 100 |
| in | In list | status in ["active", "pending"] |
| nin | Not in list | type nin ["spam", "deleted"] |
| contains | String contains | name contains "john" |
| starts_with | String starts with | email starts_with "admin" |
| ends_with | String ends with | email ends_with "@company.com" |
| regex | Regex match | username regex "^user_[0-9]+$" |
| exists | Field exists | profile_picture exists true |
| and, or | Logical combinations | (status eq "active") and (role eq "user") |

### Shopify Integration

Shopify webhooks are mapped into normalized platform events that can trigger apps.

#### Customer Matching

When a Shopify event arrives:
1. Platform looks up community user by **email** (primary key)
2. If found: Event includes `user_id` for targeting
3. If NOT found: Event includes `shopify_customer_email` only
4. App can choose to:
   - Skip non-community customers (add filter: `user_id exists true`)
   - Create community user (action: `create_user_from_shopify`)
   - Send email anyway (use `shopify_customer_email` as recipient)

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

### Per-Tenant Rate Limits by Tier

| Resource | Free | Starter | Pro | Enterprise |
|----------|------|---------|-----|------------|
| Apps | 10 | 25 | 100 | Unlimited |
| Executions/hour | 50 | 200 | 1,000 | 5,000 |
| AI calls/day | 100 | 500 | 2,000 | 10,000 |
| Emails/day | 100 | 1,000 | 10,000 | 100,000 |
| Webhooks/hour | 50 | 200 | 1,000 | 5,000 |
| State storage keys | 50 | 200 | 1,000 | 10,000 |
| Chat messages/hour | 20 | 50 | 200 | 1,000 |

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

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| App not running | Status is "paused" or "error" | Check status in Apps UI; resume or fix errors |
| Missing data in actions | Variable not in scope | Check pipeline step IDs match variable references |
| Email not sent | Rate limit exceeded | Check quota in tenant settings; wait or upgrade |
| Event trigger not firing | Event name mismatch | Verify exact event name spelling |
| Loop processing partial | Timeout reached | Reduce items per run or split into multiple apps |
| AI action failing | Token limit exceeded | Reduce prompt size or batch items |
| Webhook failing | Domain not allowlisted | Add domain to tenant webhook allowlist |
| State not persisting | Key limit exceeded | Delete unused keys or upgrade tier |

### Debugging Steps

1. **Check Execution Logs**: Apps UI → App Details → Execution History
2. **Review Error Messages**: Logs show exact failure point and error
3. **Validate Configuration**: Use "Test Run" to validate without side effects
4. **Check Rate Limits**: Tenant Settings → Usage shows current quota usage
5. **Inspect Variables**: Execution logs show variable values at each step
6. **Test with Fewer Items**: Reduce loop items to isolate issues

### Error Codes

| Code | Meaning | Resolution |
|------|---------|------------|
| E001 | Invalid configuration | Fix schema errors shown in validation |
| E002 | Rate limit exceeded | Wait or upgrade tier |
| E003 | Execution timeout | Break into smaller apps |
| E004 | Connector error | Check filter syntax and data availability |
| E005 | Action failed | Review action parameters and permissions |
| E006 | Variable not found | Ensure variable is defined in scope |
| E007 | External service error | Check webhook URL and service status |

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

