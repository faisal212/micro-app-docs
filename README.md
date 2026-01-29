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
10. [Data Connectors](#data-connectors)
11. [Action Executors](#action-executors)
12. [Triggers System](#triggers-system)
13. [Implementation Phases](#implementation-phases)
14. [Testing & Verification](#testing--verification)
15. [Key Reference Files](#key-reference-files)

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

The App Framework is the engine that stores, runs, and manages micro-apps.

### App Framework Module Structure

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

### Database Schema

#### App Entity

```typescript
@Entity('apps')
export class App extends EntityHelper {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column({ type: 'text', nullable: true })
  description: string;

  @Column({ type: 'jsonb' })
  config: AppConfig;  // Full app configuration

  @Column({ type: 'enum', enum: AppStatus, default: AppStatus.ACTIVE })
  status: AppStatus;  // ACTIVE, PAUSED, ERROR

  @Column({ type: 'timestamp', nullable: true })
  last_run_at: Date;

  @Column({ type: 'int', default: 0 })
  run_count: number;

  @Column({ type: 'int', default: 0 })
  error_count: number;

  @CreateDateColumn()
  created_at: Date;

  @UpdateDateColumn()
  updated_at: Date;
}
```

#### App Execution Entity

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

---

## Part 3: Admin Panel AI Chatbox

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
```

### Example: Low-Quality Post Detector

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

### Example: Weekly Engagement Digest

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

---

## Triggers System

### Trigger Types

| Type | Description | Configuration |
|------|-------------|---------------|
| `scheduled` | Cron-based | `{ cron: "0 9 * * MON", timezone: "UTC" }` |
| `event` | On platform event | `{ event: "user.created", filter?: {...} }` |
| `manual` | Admin clicks "Run" | `{}` |

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

### Scheduler Implementation

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

### Event Listener Implementation

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

## Environment Variables

```bash
# MCP Server
MCP_ENABLED=true
MCP_RATE_LIMIT_READ=500
MCP_RATE_LIMIT_WRITE=100

# App Framework
APP_FRAMEWORK_ENABLED=true
APP_MAX_PER_TENANT=50
APP_EXECUTION_TIMEOUT=300000  # 5 minutes

# AI Service (Claude)
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-5-sonnet-20241022
AI_MAX_TOKENS=2000
AI_RATE_LIMIT_PER_MINUTE=60
```

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
