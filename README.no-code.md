# Decommerce MCP + AI Micro-Apps — Plain-English Specification (No-Code)

This document is the plain-English companion to `MCP-integration-technical.md`. It describes the **same system and requirements**, but without code snippets. The goal is alignment: **what we’re building, what it can do, and where the hard edges are**.

**Audience**: product, engineering, ops, and anyone reviewing the micro-app/MCP direction.  
**Not included**: implementation details, exact endpoints/DTOs, or database schema diffs (those live in the technical doc and codebase).

This is a planning document. Unless a section explicitly says “we already do this today”, treat the statements below as **target behavior** we intend to implement.

---

## How to read this doc

- If you want the “why” and high-level shape, read **Executive Summary → Vision & Goals → System Architecture**.
- If you want to sanity-check safety and constraints, read **What Micro-Apps Can and Cannot Do → Execution Safeguards → Rate Limits & Quotas**.
- If you want to understand the building blocks an app config can use, read **App Configuration Schema → Connectors → Actions → Triggers**.

---

## Glossary (quick definitions)

- **Micro-app**: A tenant-scoped automation defined as **configuration** (trigger + data pipeline + actions) and executed by Decommerce. Not “custom code”.
- **MCP server**: The protocol layer exposing safe, tenant-scoped tools/resources that an AI agent can call.
- **App Framework**: The runtime that stores, validates, schedules, executes, and logs micro-apps.
- **Connector**: A read-only data fetch step (e.g., users, missions, shopify orders).
- **Action**: A side-effect or control-flow step (e.g., award XP, create post, request approval).
- **Trigger**: What causes execution (scheduled, event-driven, manual, delayed).
- **Tenant-scoped**: Every tool call / app execution runs inside a single tenant boundary; no cross-tenant access.

---

## Assumptions & open questions (read this before debating numbers)

This spec includes a few concrete limits (timeouts, loop sizes, quotas). Unless the codebase already enforces them, treat them as **target defaults** until we wire enforcement into configuration validation + runtime safeguards.

- **Numeric limits (target defaults)**: execution timeout (5 minutes), loop cap (5,000), “apps per tenant” (100), AI calls per execution (500), emails per execution (2,000).
- **Tier quotas (directional)**: the tier table is a placeholder unless your billing/plan system already defines these exact numbers.
- **Open questions**:
  - Do we want a single global execution timeout, or per-action/per-connector timeouts too?
  - Where do we enforce email/webhook quotas: at MCP tool-level, execution runtime, or both?
  - Which parts of “UI widgets” are v1 vs later (cards/leaderboards/charts)?

---

## Table of Contents

1. Executive Summary
2. How Micro-Apps Help Decommerce *(business value)*
3. Vision & Goals
4. What Micro-Apps Can and Cannot Do *(capabilities, boundaries, and enhanced capabilities)*
5. Why MCP + Micro-Apps?
6. System Architecture
7. Part 1: MCP Server
8. Part 2: App Framework
9. Part 3: Admin Panel AI Chatbox
10. Micro-App Catalog
11. Real-World Decommerce Scenarios *(end-to-end examples)*
12. App Configuration Schema
13. Variable Interpolation
14. App Configuration Validation
15. Data Connectors
16. Logic Transformations
17. Action Executors
18. Triggers System
19. Execution Safeguards
20. Error Recovery & Retry
21. Rate Limits & Quotas
22. Implementation Phases
23. Testing & Verification
24. Troubleshooting
25. Key Reference Files

---

## Executive Summary

This system is intended to let **tenant admins create micro-apps on the Decommerce platform** by describing what they want in plain English. Instead of building a bespoke integration for every workflow, we’ll standardize on a small set of **approved building blocks** (connectors, actions, triggers) and let the AI assemble those blocks into an app configuration.

Why this matters for Decommerce: we already have strong primitives (users, rooms, posts, missions, rewards, Shopify, notifications). Micro-apps are the layer that turns those primitives into “one-click operations” and recurring automation without waiting on a new backend deployment for every idea.

**The system has three parts:**

| Part | Description | Location |
|------|-------------|----------|
| MCP Server | Protocol layer allowing AI to interact with Decommerce | Backend (NestJS) |
| App Framework | Engine that stores, runs, and manages micro-apps | Backend (NestJS) |
| AI Chatbox | Interface where admins describe apps in natural language | Admin Panel (React) |

**Example flow:**
- Admin requests: "Create an app that awards 500 XP to users who complete their first mission and posts a welcome message in the #announcements room."
- AI interprets intent and produces an **app configuration** (not executable code).
- App Framework will validate, store, and activate the app configuration.
- The app will run based on its trigger (event-driven/scheduled/manual).

---

## How Micro-Apps Help Decommerce

This system directly addresses Decommerce's core business challenges:

### Community Engagement Automation

| Challenge | Micro-App Solution |
|-----------|-------------------|
| Users complete missions but don't return | Welcome series app sends personalized follow-ups at 1, 3, 7 days |
| Low engagement in rooms | Room Activity Booster app awards bonus XP for posting in quiet rooms |
| Users earn XP but never claim rewards | Reward Reminder app notifies users with unclaimed rewards |
| Leaderboard stagnation | Weekly Challenge app creates time-limited missions with bonus multipliers |

### Shopify-Community Bridge

| Challenge | Micro-App Solution |
|-----------|-------------------|
| Shopify customers don't join community | Purchase Thank You app invites buyers to community with bonus XP |
| No correlation between purchases and engagement | VIP Buyer Rewards app awards badges to high-value customers |
| Abandoned carts | Cart Recovery app sends community-exclusive discount codes |
| Post-purchase silence | Review Request app triggers 7 days after order fulfillment |

### Gamification Optimization

| Challenge | Micro-App Solution |
|-----------|-------------------|
| Users don't know about new missions | Mission Announcement app posts to relevant rooms when missions launch |
| Streak breaks discourage users | Streak Saver app sends reminder before streak expires |
| NFT badge distribution is manual | Badge Minter app automatically mints badges when milestones hit |
| Campaign end is anticlimactic | Campaign Finale app announces winners and distributes prizes |

### Admin Efficiency

| Challenge | Micro-App Solution |
|-----------|-------------------|
| Manual weekly reports | Weekly Digest app emails engagement metrics every Monday |
| Spam/low-quality posts | Content Moderator app uses AI to flag suspicious posts |
| Inactive users clutter analytics | Churn Alert app identifies users inactive for 30+ days |
| No visibility into mission performance | Mission Analytics app tracks completion rates by mission type |

---

## Vision & Goals

### Vision

AI can assemble platform capabilities into “micro-apps” on demand:
Admin asks AI → AI produces an app configuration → Decommerce runs it safely.

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

| Capability | Decommerce Examples |
|------------|---------------------|
| **Fetch platform data** | Users with completed missions, active room members, Shopify order history, XP leaderboards |
| **Send communications** | Mission completion congratulations, streak expiring warnings, reward available alerts |
| **Create content** | Auto-post mission announcements, weekly leaderboard updates, campaign kickoff posts |
| **Update records** | Award XP on mission completion, update engagement scores, flag posts for review |
| **Use AI** | Generate personalized mission recommendations, analyze post quality, create celebration messages |
| **Run on schedules** | Daily streak reminders, weekly XP digests, monthly campaign reports |
| **React to events** | On mission.completed → award bonus XP; on shopify.order.created → create welcome mission |
| **Loop over items** | Process up to 5,000 users for mass XP awards or notification campaigns |
| **Apply conditions** | Only award badge if user has 1000+ XP and completed 10+ missions |
| **Display UI widgets** | Mission completion rate chart, top contributors leaderboard, room activity heatmap |
| **Trigger other apps** | "Mission Completed" app emits event → "Badge Award" app mints NFT badge |
| **Store app state** | Track last leaderboard snapshot to detect rank changes between runs |
| **Use pre-built integrations** | Sync new members to Klaviyo, post achievements to Discord, notify via Slack |
| **Process files (basic)** | Generate PDF certificates for campaign winners, resize user-uploaded images |
| **Request human approval** | Mass XP adjustment waits for admin approval before executing |
| **Call external URLs** | Webhooks to external analytics platforms or CRM systems |

### Enhanced Capabilities (Configuration-Based)

These powerful features work through configuration, not custom code:

| Feature | How It Works | Decommerce Example |
|---------|--------------|-------------------|
| **UI Widgets** | Select from pre-built widget types, configure data source | "Top 10 Mission Completers This Week" leaderboard on community dashboard |
| **App Chaining** | Define events apps can emit; other apps listen | "Mission Milestone" app emits event → "NFT Badge Minter" app mints celebratory badge |
| **State Storage** | Key-value store per app for persistence | Track "last_leaderboard_positions" to detect when users climb or drop ranks |
| **Approval Queues** | Action pauses at approval step until admin approves | Mass XP adjustment (10,000+ XP) waits for admin approval before executing |
| **Compensating Actions** | Define "undo" action if later steps fail | If NFT mint fails, revert XP award and notify admin |
| **Pre-built Integrations** | Platform provides connectors for popular services | Sync new community members to Klaviyo list automatically |
| **Template-based Files** | Generate documents from templates | PDF certificate for campaign winners with XP stats and badges earned |

### How Enhanced Capabilities Work

#### State Storage
- **Actions**: `set_state` (save value), `get_state` (retrieve value), `delete_state` (remove key)
- **Scope**: Per-app key-value store (isolated per app)
- **Limits**: 100 keys per app, 10KB max per value
- **Use cases**:
  - Store "last_leaderboard_positions" to detect when users climb or drop ranks
  - Track "processed_mission_ids" to avoid duplicate notifications
  - Save "streak_reminder_sent" to prevent multiple reminders per day

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
│  │ "Mission    │   "milestone_reached"       │             │       │
│  │  Tracker"   │                             └──────┬──────┘       │
│  └─────────────┘                                    │              │
│                                                     │ matches      │
│                          ┌──────────────────────────┼──────────────┐
│                          │                          │              │
│                          ▼                          ▼              │
│                   ┌─────────────┐           ┌─────────────┐       │
│                   │    APP B    │           │    APP C    │       │
│                   │ "NFT Badge  │           │ "Klaviyo    │       │
│                   │  Minter"    │           │  Sync"      │       │
│                   └─────────────┘           └─────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
</pre>

- **Action**: `emit_event` with custom event name and payload
- **Other apps**: Listen via event trigger with matching event name
- **Scope**: Events are tenant-scoped (cannot cross tenants)
- **Use cases**:
  - "Mission Tracker" emits "milestone_reached" → "NFT Badge Minter" mints celebratory badge
  - "Shopify Order Handler" emits "vip_customer" → "Room Inviter" adds user to exclusive room
  - "Campaign Ended" emits "winners_selected" → "Prize Distributor" awards rewards

#### Pre-built Integrations

| Integration | Actions Available | Decommerce Use Case |
|-------------|-------------------|---------------------|
| Klaviyo | sync_profile, add_to_list, trigger_flow, track_event | Sync community members, trigger email flows on mission completion |
| Slack | send_message, send_dm | Alert team when VIP customers join, notify on campaign milestones |
| Discord | send_message, send_embed | Post leaderboard updates, announce new missions |
| Mailchimp | add_subscriber, update_tags | Segment users by engagement level, tag by badges earned |
| Twilio | send_sms | Send streak expiring warnings, reward claim confirmations |

#### UI Widgets

| Widget Type | Description | Decommerce Example |
|-------------|-------------|-------------------|
| stat_card | Single metric with label | "Missions Completed Today: 47" |
| leaderboard | Top N users table | "Top 10 XP Earners This Week" |
| chart_line | Time series chart | "Daily Active Users (30 days)" |
| chart_bar | Bar chart | "Missions by Type (Trivia, Hangman, Survey)" |
| table | Data table with columns | "Recent Reward Claims" |
| progress_ring | Circular progress indicator | "Campaign Progress: 73% Complete" |
| activity_feed | Recent events stream | "Live Mission Completions" |

#### File Processing

| Action | Description |
|--------|-------------|
| generate_pdf | Create PDF from template + data |
| resize_image | Resize image to specified dimensions |

**Templates**: Platform provides built-in templates; custom templates can be uploaded via admin panel.

#### Compensating Actions
- **Definition**: Add `on_failure` block to any action
- **Behavior**: If the action fails, run the compensating action automatically
- **Use cases**:
  - If `mint_nft_badge` fails → revert XP award and notify admin
  - If `send_email` fails → create in-app notification as fallback
  - If `create_mission` fails → log error and alert via Slack

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

### Operational limits (target defaults)

These are the “how big/how often” limits we use to keep the runtime predictable. If the runtime/validation layer doesn’t already enforce a value, treat it as a **target default** to implement (not a promise we’re already meeting).

| Area | Limit | Target default | Why it exists | Where enforced |
|------|-------|----------------|---------------|----------------|
| **Looping** | Items per loop | 5,000 | Prevent long-running mass operations from starving workers | Validation + runtime stop conditions |
| **Runtime** | Execution timeout per run | 5 minutes | Keep executions bounded; avoid “stuck” apps | Runtime watchdog/timeout |
| **AI** | AI calls per execution | 500 | Control cost/latency and provider pressure | Runtime counters + provider limits |
| **Email** | Emails per execution | 2,000 | Prevent accidental spam blasts; protect deliverability | Action executor + quota gates |
| **Tenant capacity** | Apps per tenant | 100 | Keep the system operable; encourage archiving | Validation + plan/tier limits |
| **Scheduling** | Minimum interval | 1 minute | Avoid “near-real-time” expectations; protect scheduler | Scheduler validation |
| **Scheduling** | Single timezone per scheduled app | 1 timezone | Simpler mental model; fewer edge cases | Trigger config validation |
| **Scheduling** | Business-day logic (skip weekends/holidays) | Not supported | Avoid hidden complexity in v1 | Product limitation (explicit) |

### Data access & security boundaries

This is the “what you can touch” boundary. We keep it explicit because it’s the main safety guarantee for tenant trust.

| Boundary | What it means in practice | Where enforced |
|----------|----------------------------|----------------|
| **Read via connectors only** | No raw SQL / arbitrary queries; only approved datasets with allowed filters | Connector layer |
| **Write via actions only** | All side effects go through typed actions (auditable, validateable) | Action executor |
| **Tenant isolation** | App can only access the current tenant’s data and events | Request context + data layer |
| **Current data only** | No historical “as-of” snapshots unless platform already stores them | Connector layer |
| **No stored secrets** | Micro-app configs don’t store API keys; integrations use tenant-managed settings | Settings/integration layer |
| **Webhook allowlist** | Webhooks can only target approved domains | Webhook executor |
| **No code execution** | No JS/Python/SQL snippets inside configs | Config validation |
| **No internal network access** | Block localhost, private IPs, metadata endpoints, redirects to them | Webhook executor |

### When to Request a Core Platform Feature

Contact the development team only if you need:

- **New game mechanics** not in the catalog (e.g., slot machine, scratch cards, poker)
- **New token/chain integrations** beyond supported chains (ETH, MATIC, BNB, Polygon)
- **Custom NFT contract deployment** for unique badge mechanics or special minting rules
- **Real-time interactive features** like live chat, collaborative games, or live streaming
- **Processing 100K+ items** in a single operation (split into smaller batches if possible)
- **Custom analytics dashboards** beyond the widget capabilities
- **New Shopify webhook types** not yet mapped to platform events
- **Custom reward calculation formulas** beyond standard XP/token awards
- **Third-party integrations** not yet supported (e.g., specific CRM, payment provider)

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

At this point we’re out of “what” and into “how the pieces work together”. The rest of the doc breaks the system into three parts that match how we’ll ship it: the MCP surface (tools/resources), the runtime (store + execute), and the admin UI (chat + control plane).

## Part 1: MCP Server

The MCP Server will expose **tool endpoints** and **resources** an AI agent can use to interact with Decommerce safely (and predictably). The MCP layer is where we plan to enforce tenant context, auth, and coarse-grained read/write policy before anything touches business data.

### MCP module responsibilities (responsibilities, not implementation detail)

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

**Proposed v1 scope (starting set):**
- `create_app`, `list_apps`, `get_app`, `update_app`, `delete_app`
- `run_app`, `pause_app`, `resume_app`
- `get_app_logs`

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

These are tenant-scoped tools for listing/fetching core entities and metrics. We should start narrow and expand as we learn what admins actually automate most:
- list/get users
- list/get rooms
- list/create posts (where appropriate)
- list/create missions
- list contributions/rewards data
- fetch community stats and engagement metrics
- send notifications (to admins or users)

---

## Part 2: App Framework

The App Framework will be the runtime engine that stores, validates, schedules, executes, and monitors micro-apps. If MCP is the “hands” an agent can use, the framework is the “operating system” that makes those hands safe at scale.

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

The admin panel will be the control plane for humans. It will provide:
- a chat interface to request apps in natural language,
- streaming responses,
- optional confirmation gates for tool execution,
- and an apps management UI (list/detail/logs/controls).

### Endpoints (shape)

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

This is a reference catalog of example apps by category. Not all of these need to ship on day one; the point is to show the kinds of automations the building blocks should enable.

**Proposed v1 starter pack (what we should be able to build early):**

| App | Trigger | Why it’s a good v1 test |
|-----|---------|--------------------------|
| Welcome XP Bonus | `user.created` | Exercises event triggers + awarding XP + posting |
| Room Welcome | `room.member_joined` | Simple event-driven messaging |
| Mission Completion Congrats | `mission.completed` | Common workflow; clear success criteria |
| Weekly XP Digest | scheduled | Exercises scheduling + aggregations + email |
| Reward Reminder | scheduled | Exercises filtering + notifications |
| Purchase Thank You | `shopify.order.created` | Exercises Shopify mapping + user matching |
| Content Moderator (basic) | `post.created` | Exercises AI step + moderation action |

### Email & Communication
| App Name | Trigger | What It Does |
|----------|---------|--------------|
| Weekly XP Digest | Scheduled: Monday 8 AM | Sends email with XP earned, missions completed, rank change |
| Streak Warning | Scheduled: Daily 6 PM | Alerts users whose streak expires in next 6 hours |
| Reward Reminder | Scheduled: Weekly | Notifies users with unclaimed rewards (500+ XP available) |
| Room Welcome | Event: room.member_joined | Sends personalized welcome message to new room members |
| Campaign Kickoff | Event: campaign.started | Emails opted-in users about new campaign with first mission link |
| Win-Back Email | Scheduled: Weekly | Targets users inactive 30+ days with special "comeback" mission |

### Content Automation
| App Name | Trigger | What It Does |
|----------|---------|--------------|
| Mission Announcer | Event: mission.created | Auto-posts mission announcement in relevant room |
| Leaderboard Update | Scheduled: Friday 5 PM | Posts weekly leaderboard to #announcements room |
| Milestone Celebration | Event: milestone.reached | Creates congratulations post when user hits XP milestone |
| Campaign Recap | Event: campaign.ended | Posts campaign results with top performers and stats |
| New Member Spotlight | Scheduled: Daily | Features users who joined in last 24 hours |

### Engagement & Gamification
| App Name | Trigger | What It Does |
|----------|---------|--------------|
| Streak Bonus | Event: contribution.completed | Awards +50% XP if user has 7+ day streak |
| Leaderboard Climber | Event: xp.awarded | Mints "Rising Star" badge when user enters top 10 |
| NFT Badge Minter | Event: milestone.reached | Auto-mints NFT badge when XP threshold hit |
| Room Activity Booster | Scheduled: Daily | Awards bonus XP for posting in rooms with <5 posts today |
| Referral Tracker | Event: user.created | Awards XP to referrer when referred user completes first mission |
| Hangman Helper | Scheduled: Daily | Sends hint to users stuck on hangman for 24+ hours |

### Analytics & Reporting
| App Name | Trigger | What It Does |
|----------|---------|--------------|
| Weekly Admin Digest | Scheduled: Monday 9 AM | Emails admin with engagement metrics, top performers |
| Churn Risk Alert | Scheduled: Daily | Flags users inactive 14+ days; creates re-engagement list |
| Mission Effectiveness | Scheduled: Weekly | Reports completion rates by mission type (trivia vs hangman) |
| Shopify Correlation | Scheduled: Monthly | Analyzes purchase behavior vs community engagement |
| Room Health Dashboard | Scheduled: Daily | Updates dashboard widgets with room activity metrics |

### Moderation & Management
| App Name | Trigger | What It Does |
|----------|---------|--------------|
| Content Moderator | Event: post.created | AI analyzes post quality; flags suspicious content |
| Inactive Cleanup | Scheduled: Monthly | Identifies users with 0 activity in 90 days |
| New User Watchlist | Event: user.created | Monitors first 7 days of activity; alerts if suspicious |
| Engagement Calculator | Scheduled: Daily | Recalculates engagement scores based on recent activity |
| Room Archiver | Scheduled: Monthly | Auto-archives rooms with <10 posts in 60 days |

### Shopify Integration
| App Name | Trigger | What It Does |
|----------|---------|--------------|
| Purchase Thank You | Event: shopify.order.created | Awards XP (1 per $1 spent) + "Customer" badge |
| VIP Buyer | Event: shopify.order.fulfilled | Mints "VIP" NFT badge at $500 lifetime purchases |
| Review Request | Delayed: 7 days after order.fulfilled | Creates "Share Experience" mission with 300 XP reward |
| First Purchase Celebration | Event: shopify.order.created | Awards "First Purchase" badge + room invite |
| Restock Alert | Event: shopify.product.restocked | Notifies previous buyers of product availability |

---

## Real-World Decommerce Scenarios

These end-to-end examples show how micro-apps work together to solve actual Decommerce use cases.

### Scenario 1: New User Onboarding Journey

**Goal**: Guide new users through their first week with progressively valuable touchpoints.

**Apps Created**:

1. **Welcome XP Bonus** (Event: user.created)
   - Awards 100 XP instantly to new user
   - Creates personalized welcome post in #introductions room
   - Syncs user profile to Klaviyo with "new_member" tag

2. **Day 1 Mission Nudge** (Delayed: 24h after user.created)
   - Checks if user has completed any mission
   - If not: sends push notification suggesting easiest mission (e.g., "Verify Email")
   - If yes: sends congratulations with next mission recommendation

3. **Day 3 Room Invite** (Delayed: 72h after user.created)
   - Analyzes user's profile interests and completed missions
   - AI generates personalized room recommendations
   - Sends email with top 3 room suggestions and invite links

4. **Week 1 Celebration** (Delayed: 7 days after user.created)
   - Calculates user's first-week stats (XP earned, missions completed, posts)
   - If engaged (5+ missions): awards "Active Newcomer" NFT badge
   - If inactive (<2 missions): sends win-back email with exclusive bonus mission

### Scenario 2: Shopify Purchase → Community Engagement

**Goal**: Convert Shopify customers into active community members.

**Apps Created**:

1. **Purchase Thank You** (Event: shopify.order.created)
   - Matches customer email to community user
   - If member: awards "Customer" badge + 200 XP + thank you notification
   - If not member: sends community invitation email with 500 XP signup bonus
   - Tracks purchase in Klaviyo for segmentation

2. **Product Fan Badge** (Event: shopify.order.fulfilled)
   - Counts user's total fulfilled orders
   - At 3 orders: mints "Loyal Customer" NFT badge
   - At 10 orders: mints "VIP" NFT badge + grants access to exclusive room
   - Posts achievement in #community-wins room

3. **Review Request** (Delayed: 7 days after shopify.order.fulfilled)
   - Creates personalized "Share Your Experience" mission for user
   - Mission rewards 300 XP + entry into monthly prize drawing
   - Links to product-specific discussion room
   - If no response: follows up at day 14

### Scenario 3: Weekly Engagement Boost Campaign

**Goal**: Maintain consistent community activity with weekly challenges.

**Apps Created**:

1. **Monday Challenge Launcher** (Scheduled: Monday 9 AM)
   - AI generates weekly theme based on trending topics and past performance
   - Creates time-limited missions (Trivia, Hangman, Survey) expiring Sunday
   - Posts announcement in #weekly-challenges room
   - Sends push notification to users who participated in previous weeks

2. **Mid-Week Reminder** (Scheduled: Wednesday 2 PM)
   - Fetches users who haven't participated yet this week
   - Calculates current participation rate
   - Sends personalized push: "72 members completed this week's challenge - join them!"
   - Posts progress update in #weekly-challenges room

3. **Friday Leaderboard Update** (Scheduled: Friday 5 PM)
   - Calculates current week's standings based on challenge XP
   - Posts leaderboard to #weekly-challenges room
   - Notifies top 10 users of their current position
   - Sends "You're almost in top 10!" to users ranked 11-15

4. **Sunday Winner Announcement** (Scheduled: Sunday 8 PM)
   - Finalizes weekly rankings
   - Awards bonus XP: 1st (500), 2nd (300), 3rd (200)
   - Mints "Weekly Champion" badge to #1
   - Posts celebration post with stats and next week preview
   - Resets weekly challenge state for next week

---

## App Configuration Schema

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

## Variable Interpolation

Configurations use template placeholders to reference values available at runtime.

### Syntax
- Variables are referenced using a double-brace placeholder format: `{{variable.name}}`

### Variable Sources

| Source | Available Variables |
|--------|---------------------|
| **User data** | `user.id`, `user.email`, `user.name`, `user.xp_total`, `user.web3_account`, `user.engagement_score` |
| **Mission data** | `mission.id`, `mission.name`, `mission.xp_reward`, `mission.action_type`, `mission.status` |
| **Contribution data** | `contribution.id`, `contribution.type`, `contribution.days_in_a_row`, `contribution.completed` |
| **Room data** | `room.id`, `room.name`, `room.member_count`, `room.type`, `room.is_exclusive` |
| **Campaign data** | `campaign.id`, `campaign.name`, `campaign.start_date`, `campaign.end_date` |
| **Shopify data** | `order.id`, `order.totalPriceAmount`, `order.currency`, `product.title`, `product.price` |
| **NFT Badge data** | `badge.id`, `badge.name`, `badge.contract_type`, `badge.chain_id` |
| **Event payload** | `event.user_id`, `event.mission_id`, `event.timestamp`, `event.type` |
| **Built-ins** | `now`, `tenant_id`, `app_id`, `execution_id` |
| **AI outputs** | `ai_result.text`, `ai_result.classification`, `ai_result.score` |
| **Pipeline outputs** | `steps.step_id.data`, `steps.step_id.count` |

### Example Interpolations

| Template | Result |
|----------|--------|
| `Congrats {{user.name}}! You earned {{mission.xp_reward}} XP!` | "Congrats Alex! You earned 500 XP!" |
| `Your {{contribution.days_in_a_row}}-day streak continues!` | "Your 7-day streak continues!" |
| `Thanks for your {{order.totalPriceAmount}} {{order.currency}} order!` | "Thanks for your 49.99 USD order!" |
| `Welcome to {{room.name}} ({{room.member_count}} members)` | "Welcome to Gaming Lounge (342 members)" |
| `You've earned the {{badge.name}} badge!` | "You've earned the VIP Customer badge!" |
| `Campaign {{campaign.name}} ends {{campaign.end_date \| date}}` | "Campaign Summer Challenge ends Jan 31" |

### Common features
- Dot notation for nested fields: `{{user.profile.avatar_url}}`
- Array indexing: `{{steps.users.data[0].name}}`
- Default/fallback values: `{{user.nickname \| default:user.name}}`
- Formatting filters: `uppercase`, `lowercase`, `date`, `number`, `truncate`, `join`, `json`
- Clear scoping rules: global → pipeline → loop → action outputs

---

## App Configuration Validation

Before saving an AI-generated configuration, we will require validation to pass:

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

## Data Connectors

Connectors are read-only data access building blocks. Each connector defines:
- what entity/metric it reads from
- allowed filter fields/operators
- allowed field selection
- limits/offsets

**Proposed v1 scope (starting set):**
- `users`, `rooms`, `posts`
- `missions`, `user_missions`, `contributions`
- `shopify_orders`

### Available Connectors

| Connector | Description | Key Filters |
|-----------|-------------|-------------|
| users | Platform users | status, role, created_at, engagement_score, web3_account |
| posts | User-generated content | room_id, author_id, created_at, status |
| rooms | Community spaces | status, type, member_count, is_exclusive |
| room_members | Room membership data | room_id, user_id, role, is_admin |
| missions | Available missions | status, type, room_id, action_type |
| user_missions | User mission progress | user_id, mission_id, status, completed_at |
| contributions | User contributions | user_id, type, created_at, days_in_a_row |
| rewards | Reward claims | user_id, reward_type, claimed_at, token_type |
| campaigns | Mission campaigns | status, type, start_date, end_date |
| nft_badges | NFT badge definitions | contract_type, room_id, campaign_id, chain_id |
| milestones | XP milestones | xp_threshold, reward_type |
| total_rewards | User accumulated rewards | user_id, token_type, amount |
| hangman_status | Hangman game progress | user_id, mission_id, completed, remain_count |
| trivia_status | Trivia game progress | user_id, mission_id, score, completed |
| shopify_orders | Shopify orders | customer_email, status, created_at, totalPriceAmount |
| shopify_products | Shopify products | status, type, collection |
| engagement_metrics | Aggregated metrics | user_id, period, metric_type |
| notifications | Sent notifications | user_id, type, status |
| app_state | Current app's state storage | key |

Each connector also:
- validates filters against allowed operators
- provides field metadata for documentation and AI guidance

---

## Logic Transformations

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

## Action Executors

Actions are side-effect operations or control-flow blocks. Each action type has:
- parameter validation
- execution logic
- structured result output (for downstream steps or logs)

**Proposed v1 scope (starting set):**
- Communication: `send_email`, `send_notification` (and/or `send_push` if already supported)
- Content: `create_post`
- Users: `award_xp`, `update_user`
- Flow control: `for_each`, `condition`, `request_approval`
- Integrations (only if configured): `klaviyo.sync_profile`, `klaviyo.add_to_list`

### Available Actions

| Category | Actions |
|----------|---------|
| **Communication** | send_email, send_notification, send_push |
| **Content** | create_post, update_post, flag_post, delete_post, create_comment |
| **Users** | update_user, award_xp, award_badge, update_engagement_score, connect_wallet |
| **Missions** | create_mission, complete_mission, assign_mission, expire_mission |
| **Contributions** | create_contribution, complete_contribution, award_contribution_xp |
| **Campaigns** | start_campaign, end_campaign, select_winners |
| **NFT/Web3** | mint_nft_badge, transfer_token, verify_wallet, check_token_balance |
| **Gamification** | start_streak, extend_streak, reset_streak, award_milestone |
| **Rooms** | add_to_room, remove_from_room, grant_room_access |
| **State** | set_state, get_state, delete_state |
| **Flow Control** | for_each, condition, request_approval, emit_event |
| **AI** | ai_analyze, ai_generate, ai_classify |
| **External** | webhook_call |
| **Klaviyo** | klaviyo.sync_profile, klaviyo.add_to_list, klaviyo.trigger_flow, klaviyo.track_event |
| **Integrations** | slack.send_message, discord.send_message, mailchimp.add_subscriber, twilio.send_sms |
| **Files** | generate_pdf, resize_image |
| **Compensate** | on_failure (wrapper for any action) |

### Webhook security requirements
Webhook calls will be constrained with:
- allowlisted domains (tenant-level)
- blocked internal/private networks and cloud metadata targets
- strict timeouts
- redirect safety checks
- request/response logging for audit
- call quotas to prevent abuse

---

## Triggers System

### Trigger types
- **scheduled**: cron-based scheduler triggers execution.
- **event**: platform emits events; event listener triggers matching apps.
- **manual**: admin requests run.
- **delayed**: on an event, schedule one or more future executions at specified delays, with optional conditions per delay.

**Proposed v1 scope (starting set):**
- Trigger types: `manual`, `scheduled`, `event` (delayed can be v1.5 if scheduling infra isn’t ready)
- Events: `user.created`, `mission.completed`, `post.created`, `room.member_joined`, `shopify.order.created`

### Available Events

| Category | Event Names |
|----------|-------------|
| **User Lifecycle** | user.created, user.updated, user.deleted, user.login, user.wallet_connected |
| **Content** | post.created, post.updated, post.deleted, post.flagged, comment.created |
| **Missions** | mission.created, mission.completed, mission.expired, mission.assigned |
| **Contributions** | contribution.created, contribution.completed, contribution.expired |
| **Rewards** | reward.claimed, xp.awarded, badge.earned, milestone.reached |
| **Campaigns** | campaign.started, campaign.ended, campaign.winner_selected |
| **NFT Badges** | nft_badge.minted, nft_badge.claimed, nft_badge.transferred |
| **Streaks** | streak.started, streak.extended, streak.broken |
| **Games** | hangman.completed, trivia.completed, spin_wheel.completed, survey.completed |
| **Rooms** | room.created, room.member_joined, room.member_left, room.access_requested |
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
- Each app execution will acquire an execution lock.
- If locked, a new run is skipped to prevent overlapping execution.
- Locks will expire to avoid stale lock deadlocks.

### Limits enforcement (how the system stays stable)

The specific limits live earlier in the doc under **Operational limits (target defaults)**. This section is about the mechanics that make those limits real at runtime:

- **Validation gates**: reject configs that exceed structural limits (pipeline steps, actions, nesting).
- **Runtime stop conditions**: stop loops/executions when item caps or timeouts are hit and log a clear reason.
- **Counters & quotas**: track AI calls/emails/webhooks during an execution and stop before exceeding caps.
- **Auto-pause on repeated failures**: after \(N\) consecutive failures, pause the app and notify admins (value for \(N\) should be configurable within platform bounds).

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
For actions that should not repeat (like sending emails), we’ll use idempotency keys so retries do not duplicate side effects.

---

## Rate Limits & Quotas

### Global rate limits
System-wide limits control:
- MCP read/write tool calls
- app execution duration and concurrency pressure
- AI tokens/calls and streaming capacity

### Per-Tenant Rate Limits by Tier

This table is about **plan-level quotas** (how much a tenant can do over time). It’s separate from the **per-execution operational limits** earlier in the doc.
If your billing/plan system doesn’t already enforce these exact numbers, treat them as **placeholder targets**.

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
- MCP server skeleton: transport + auth + tenant context + tool registry + basic rate limiting
- App storage: app + execution entities, persistence, and migrations
- CRUD + logs: create/list/get/update/delete apps + fetch execution logs
- “Hello world” end-to-end: create an app config and run it manually against a test tenant

### Phase 2: Connectors & Actions (Week 2–3)
- Implement v1 connector set (users/rooms/posts/missions/user_missions/contributions/shopify_orders)
- Implement v1 action set (award_xp, create_post, send_notification/send_email, basic flow control)
- Executor runtime: pipeline → transforms → actions, with logging + stop conditions
- AI step integration (guardrails + prompt templates + output stored in execution context)

### Phase 3: Triggers System (Week 3–4)
- Scheduler for cron-based apps (minimum: once per minute)
- Event listeners for platform events (user/mission/post/room/shopify v1 set)
- Manual run support (admin clicks “Run now”)
- Delayed triggers if feasible; otherwise document as v1.5

### Phase 4: Admin Panel UI (Week 4–5)
- Chatbox UI with streaming responses + tool confirmation UI
- Apps list + app details pages (config preview + status + last run)
- Execution history/log viewer (search/filter by status)
- Pause/resume/run controls
- Basic “dry run / validate-only” path if supported by backend

### Phase 5: Testing & Polish (Week 5–6)
- Unit tests for: validation, v1 connectors/actions, execution stop conditions
- Integration tests: event trigger → pipeline → action side effect (in staging)
- Error handling: retries, idempotency where needed, auto-pause + notifications
- Docs + tuning: performance, quotas, and operational dashboards

---

## Testing & Verification

### Test cases (what we should be able to prove)
- **Create app via chat flow**: AI proposes a config → confirmation (if enabled) → app is stored and appears in apps list
- **Manual run**: “Run now” executes the pipeline/actions and writes an execution record
- **Scheduled run**: cron trigger fires on time and doesn’t overlap if the prior run is still locked
- **Event run**: a known platform event triggers the correct app(s) with correct tenant scoping
- **Connector correctness**: filters/operators are validated and results are bounded/paginated
- **Action correctness**: side effects happen once (idempotency where required), failures are reported clearly
- **AI step persistence**: AI outputs are captured into the execution context and show up in logs
- **Limit enforcement**: loop caps/timeouts/quota counters stop execution and log “why” (not just “failed”)
- **Pause/resume**: paused apps don’t run; resuming restores scheduling/event triggers
- **Observability**: logs can be queried and rendered in the admin UI without manual DB access

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
| Event trigger not firing | Event name mismatch | Verify exact event name spelling (e.g., `mission.completed` not `missionCompleted`) |
| Loop processing partial | Timeout reached | Reduce items per run or split into multiple apps |
| AI action failing | Token limit exceeded | Reduce prompt size or batch items |
| Webhook failing | Domain not allowlisted | Add domain to tenant webhook allowlist |
| State not persisting | Key limit exceeded | Delete unused keys or upgrade tier |

### Decommerce-Specific Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| NFT mint failing | Wallet not connected or insufficient gas | Verify user has `web3_account`; check chain gas prices |
| Streak not updating | Contribution timestamp issue | Verify `contribution.created` fires before midnight UTC |
| Mission not completing | Action requirements not met | Check `mission.action_type` matches user's actual activity |
| Klaviyo sync failing | API key invalid or list not found | Verify Klaviyo settings in tenant configuration |
| XP not awarded | Duplicate contribution check | Each contribution type can only award XP once per period |
| Badge already minted | User already has badge | Check `nft_badges` connector for existing badge ownership |
| Room access denied | User not a member | Use `add_to_room` action before sending room-specific content |
| Campaign winner not selected | Campaign still active | Ensure `campaign.end_date` has passed before selecting winners |
| Trivia/Hangman stuck | Game state corrupted | Check `trivia_status` or `hangman_status` connector for state |
| Shopify order not matched | Email mismatch | Verify customer email in Shopify matches community user email |

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

