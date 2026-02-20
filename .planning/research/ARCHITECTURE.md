# Architecture Patterns

**Domain:** Real-time developer accountability / social productivity platform
**Project:** DevReal
**Researched:** 2026-02-20
**Overall confidence:** HIGH (Next.js App Router structure verified via official docs; Supabase/Drizzle/Vercel patterns from training data + project research)

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Project Folder Structure](#2-project-folder-structure)
3. [Database Schema & Indexing](#3-database-schema--indexing)
4. [Row Level Security Policies](#4-row-level-security-policies)
5. [API Route Structure & Middleware](#5-api-route-structure--middleware)
6. [Real-time Subscriptions](#6-real-time-subscriptions)
7. [GitHub App Webhook Pipeline](#7-github-app-webhook-pipeline)
8. [Scheduled Notifications (Cron)](#8-scheduled-notifications-cron)
9. [File Uploads & Signed URLs](#9-file-uploads--signed-urls)
10. [Scoring Engine](#10-scoring-engine)
11. [Weekly League Reset System](#11-weekly-league-reset-system)
12. [Rate Limiting](#12-rate-limiting)
13. [Component Boundaries & Data Flow](#13-component-boundaries--data-flow)
14. [Build Order & Dependencies](#14-build-order--dependencies)
15. [Anti-Patterns to Avoid](#15-anti-patterns-to-avoid)
16. [Scalability Considerations](#16-scalability-considerations)

---

## 1. System Overview

```
                              Vercel Edge Network
                    +-------------------------------------+
                    |                                     |
                    |   Next.js 15 (App Router)           |
                    |   +-------------------------------+ |
                    |   | Server Components (RSC)       | |
                    |   | Server Actions                | |
                    |   | API Routes (/api/*)            | |
                    |   | Middleware (auth, rate-limit)  | |
                    |   | Cron Routes (/api/cron/*)     | |
                    |   +-------------------------------+ |
                    |                                     |
                    +--+--------+--------+--------+------+
                       |        |        |        |
           +-----------+   +----+   +----+   +----+--------+
           |               |        |        |              |
           v               v        v        v              v
    +------+------+  +-----+--+  +--+---+  +-+----------+  +--------+
    |  Supabase   |  |Supabase|  |Supa- |  |  Supabase  |  | GitHub |
    |  PostgreSQL |  |  Auth  |  |base  |  |  Storage   |  |  API   |
    |  + Drizzle  |  |        |  |Real- |  |  (proofs)  |  |        |
    |             |  |        |  |time  |  |            |  |        |
    +------+------+  +--------+  +------+  +------------+  +--------+
           |
    +------+------+
    | pg_cron     |
    | (Supabase)  |
    | DB triggers |
    | DB functions |
    +-------------+
```

### Key Architectural Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Rendering strategy | Server Components default, Client Components for interactive | Minimize client JS; crew feed and declarations are data-driven |
| Data fetching | Server Actions + Drizzle queries | Type-safe, no REST layer to maintain for mutations |
| API routes | Only for webhooks and cron jobs | Server Actions handle user-facing mutations; API routes for external integrations |
| Real-time | Supabase Postgres Changes per crew channel | Small crews (3-6) mean low fan-out; Postgres Changes auto-sync with DB |
| Scoring | Batch calculation via pg_cron, not real-time | Scores don't need sub-second freshness; batch is simpler and cheaper |
| Auth flow | Supabase Auth with PKCE, `@supabase/ssr` middleware | Cookie-based sessions, RLS integration, zero custom auth code |

---

## 2. Project Folder Structure

Based on Next.js 15 App Router conventions (verified from official docs). Uses the "store project files outside of `app`" strategy with route groups for layout segmentation.

```
src/
+-- app/
|   +-- (auth)/                          # Auth route group (no sidebar layout)
|   |   +-- login/
|   |   |   +-- page.tsx
|   |   +-- callback/
|   |   |   +-- route.ts                 # OAuth callback handler
|   |   +-- layout.tsx                   # Minimal layout (centered, no nav)
|   |
|   +-- (onboarding)/                    # Onboarding route group
|   |   +-- setup/
|   |   |   +-- page.tsx                 # Profile setup after first auth
|   |   +-- crew/
|   |   |   +-- create/
|   |   |   |   +-- page.tsx
|   |   |   +-- join/
|   |   |   |   +-- [code]/
|   |   |   |       +-- page.tsx
|   |   +-- goal/
|   |   |   +-- page.tsx                 # Set first goal
|   |   +-- layout.tsx                   # Onboarding stepper layout
|   |
|   +-- (app)/                           # Main app route group (with sidebar)
|   |   +-- declare/
|   |   |   +-- page.tsx                 # Morning declaration
|   |   |   +-- loading.tsx
|   |   +-- feed/
|   |   |   +-- page.tsx                 # Crew feed (default home)
|   |   |   +-- loading.tsx
|   |   +-- ship-check/
|   |   |   +-- page.tsx                 # Proof submission with countdown
|   |   |   +-- loading.tsx
|   |   +-- leaderboard/
|   |   |   +-- page.tsx
|   |   |   +-- loading.tsx
|   |   +-- profile/
|   |   |   +-- page.tsx                 # Own profile
|   |   |   +-- settings/
|   |   |   |   +-- page.tsx
|   |   +-- crew/
|   |   |   +-- [crewId]/
|   |   |   |   +-- page.tsx             # Crew detail/management
|   |   |   |   +-- settings/
|   |   |   |       +-- page.tsx
|   |   +-- u/
|   |   |   +-- [username]/
|   |   |       +-- page.tsx             # Public profile
|   |   +-- layout.tsx                   # Sidebar + bottom nav layout
|   |   +-- error.tsx
|   |   +-- not-found.tsx
|   |
|   +-- api/
|   |   +-- webhooks/
|   |   |   +-- github/
|   |   |       +-- route.ts             # GitHub App webhook receiver
|   |   +-- cron/
|   |   |   +-- ship-check/
|   |   |   |   +-- route.ts             # Schedule random ship check notifications
|   |   |   +-- league-reset/
|   |   |   |   +-- route.ts             # Monday league promotion/demotion
|   |   |   +-- daily-summary/
|   |   |   |   +-- route.ts             # End-of-day activity rollup
|   |   |   +-- auto-approve/
|   |   |       +-- route.ts             # 48h timeout auto-approval
|   |   +-- upload/
|   |       +-- signed-url/
|   |           +-- route.ts             # Generate signed upload URLs
|   |
|   +-- layout.tsx                       # Root layout (html, body, providers)
|   +-- global-error.tsx
|   +-- not-found.tsx
|
+-- lib/
|   +-- db/
|   |   +-- index.ts                     # Drizzle client initialization
|   |   +-- schema/
|   |   |   +-- users.ts
|   |   |   +-- crews.ts
|   |   |   +-- declarations.ts
|   |   |   +-- ship-checks.ts
|   |   |   +-- proof-reviews.ts
|   |   |   +-- reactions.ts
|   |   |   +-- nudges.ts
|   |   |   +-- league-standings.ts
|   |   |   +-- daily-activity.ts
|   |   |   +-- reflections.ts
|   |   |   +-- notification-preferences.ts
|   |   |   +-- goals.ts
|   |   |   +-- github-installations.ts
|   |   |   +-- index.ts                 # Re-export all schemas
|   |   +-- queries/
|   |   |   +-- users.ts                 # User-related queries
|   |   |   +-- crews.ts                 # Crew-related queries
|   |   |   +-- declarations.ts
|   |   |   +-- ship-checks.ts
|   |   |   +-- leaderboard.ts           # Complex leaderboard queries
|   |   |   +-- heat-map.ts              # Daily activity aggregation queries
|   |   |   +-- scoring.ts               # Scoring engine queries
|   |   +-- migrations/                  # Drizzle migration files
|   |
|   +-- supabase/
|   |   +-- client.ts                    # Browser Supabase client
|   |   +-- server.ts                    # Server-side Supabase client (cookies)
|   |   +-- admin.ts                     # Service-role client (for cron/webhooks)
|   |   +-- middleware.ts                # Auth session refresh logic
|   |
|   +-- actions/
|   |   +-- declarations.ts              # Server Actions for declarations
|   |   +-- ship-checks.ts              # Server Actions for proof submission
|   |   +-- crews.ts                     # Server Actions for crew operations
|   |   +-- reviews.ts                   # Server Actions for proof reviews
|   |   +-- reactions.ts
|   |   +-- nudges.ts
|   |   +-- profile.ts
|   |
|   +-- services/
|   |   +-- github.ts                    # GitHub API client + webhook processing
|   |   +-- scoring.ts                   # Composite score calculation logic
|   |   +-- league.ts                    # League promotion/demotion logic
|   |   +-- notifications.ts            # FCM + in-app notification dispatch
|   |   +-- upload.ts                    # File upload + signed URL generation
|   |   +-- verification.ts             # Auto-verification logic
|   |
|   +-- hooks/
|   |   +-- use-realtime-feed.ts         # Crew feed real-time subscription
|   |   +-- use-crew-presence.ts         # Who's online in crew
|   |   +-- use-countdown.ts            # Ship check countdown timer
|   |   +-- use-optimistic-review.ts    # Optimistic concurrency for reviews
|   |
|   +-- utils/
|   |   +-- timezone.ts                  # Timezone conversion helpers
|   |   +-- dates.ts                     # Date manipulation (day boundaries, etc.)
|   |   +-- scoring.ts                   # Score calculation formulas
|   |   +-- rate-limit.ts               # Rate limiting utilities
|   |   +-- crypto.ts                    # Webhook signature verification
|   |
|   +-- types/
|   |   +-- index.ts                     # Shared TypeScript types
|   |   +-- database.ts                  # DB row types (inferred from Drizzle)
|   |   +-- api.ts                       # API request/response types
|   |
|   +-- constants/
|       +-- scoring.ts                   # Scoring weights, caps, thresholds
|       +-- leagues.ts                   # League names, promotion rules
|       +-- reactions.ts                 # Reaction types
|       +-- limits.ts                    # Rate limits, crew sizes, etc.
|
+-- components/
|   +-- ui/                              # shadcn/ui components (auto-generated)
|   +-- layout/
|   |   +-- sidebar.tsx
|   |   +-- bottom-nav.tsx
|   |   +-- header.tsx
|   +-- feed/
|   |   +-- feed-list.tsx
|   |   +-- feed-item.tsx
|   |   +-- declaration-card.tsx
|   |   +-- proof-card.tsx
|   |   +-- reaction-bar.tsx
|   +-- crew/
|   |   +-- crew-card.tsx
|   |   +-- member-list.tsx
|   |   +-- invite-dialog.tsx
|   |   +-- crew-status.tsx
|   +-- ship-check/
|   |   +-- countdown-timer.tsx
|   |   +-- proof-tabs.tsx
|   |   +-- screenshot-upload.tsx
|   |   +-- url-input.tsx
|   |   +-- github-picker.tsx
|   |   +-- text-proof.tsx
|   +-- leaderboard/
|   |   +-- league-table.tsx
|   |   +-- tier-badge.tsx
|   |   +-- weekly-standings.tsx
|   +-- profile/
|   |   +-- heat-map.tsx
|   |   +-- stats-bar.tsx
|   |   +-- badge-grid.tsx
|   |   +-- streak-counter.tsx
|   +-- celebrations/
|   |   +-- confetti.tsx
|   |   +-- streak-milestone.tsx
|   |   +-- league-promotion.tsx
|   +-- shared/
|       +-- countdown.tsx
|       +-- avatar.tsx
|       +-- status-dot.tsx
|       +-- empty-state.tsx
|
+-- supabase/
    +-- migrations/                      # SQL migration files
    +-- seed.sql                         # Development seed data
    +-- config.toml                      # Supabase local dev config
```

### Why This Structure

- **Route groups** `(auth)`, `(onboarding)`, `(app)` give three distinct layout contexts without affecting URLs. The main app has a sidebar; auth and onboarding do not.
- **`lib/` outside `app/`** keeps business logic separate from routing. Pages are thin: they fetch data and render components.
- **`lib/actions/`** holds Server Actions, co-located by domain. Server Actions are the primary mutation path for user-facing operations.
- **`lib/services/`** holds complex business logic that Server Actions and API routes both call. This prevents duplication between cron jobs and user-triggered actions.
- **`lib/db/queries/`** isolates database queries. Pages and actions import from here. Query functions return typed results via Drizzle inference.
- **`components/`** is organized by feature domain, not by type. This prevents the "100 files in components/" problem.

---

## 3. Database Schema & Indexing

### Complete Schema (Drizzle ORM)

**Confidence: HIGH** -- Based on the feature spec data model, adapted for implementation concerns.

```typescript
// lib/db/schema/users.ts
import { pgTable, uuid, text, integer, real, timestamp, boolean, pgEnum } from 'drizzle-orm/pg-core';

export const tierEnum = pgEnum('tier', ['starter', 'builder', 'shipper', 'architect', 'legend']);

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  authId: text('auth_id').unique().notNull(),          // Supabase auth.users.id
  username: text('username').unique().notNull(),
  displayName: text('display_name'),
  avatarUrl: text('avatar_url'),
  githubId: text('github_id').unique(),
  githubUsername: text('github_username'),
  email: text('email'),
  trustScore: real('trust_score').default(0.5).notNull(),
  currentStreak: integer('current_streak').default(0).notNull(),
  longestStreak: integer('longest_streak').default(0).notNull(),
  compositeScore: real('composite_score').default(0).notNull(),
  lifetimeXp: integer('lifetime_xp').default(0).notNull(),
  tier: tierEnum('tier').default('starter').notNull(),
  timezone: text('timezone').default('UTC').notNull(),    // IANA timezone string
  streakFreezes: integer('streak_freezes').default(2).notNull(),
  isOnboarded: boolean('is_onboarded').default(false).notNull(),
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});
```

```typescript
// lib/db/schema/crews.ts
export const crewStatusEnum = pgEnum('crew_status', ['active', 'grace_period', 'archived']);
export const crewRoleEnum = pgEnum('crew_role', ['owner', 'member']);

export const crews = pgTable('crews', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  inviteCode: text('invite_code').unique().notNull(),    // 6-char alphanumeric
  createdBy: uuid('created_by').references(() => users.id).notNull(),
  memberCount: integer('member_count').default(0).notNull(),
  status: crewStatusEnum('status').default('active').notNull(),
  graceDeadline: timestamp('grace_deadline', { withTimezone: true }),
  lastActivityAt: timestamp('last_activity_at', { withTimezone: true }).defaultNow(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

export const crewMembers = pgTable('crew_members', {
  id: uuid('id').primaryKey().defaultRandom(),
  crewId: uuid('crew_id').references(() => crews.id, { onDelete: 'cascade' }).notNull(),
  userId: uuid('user_id').references(() => users.id, { onDelete: 'cascade' }).notNull(),
  role: crewRoleEnum('role').default('member').notNull(),
  joinedAt: timestamp('joined_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  uniqueMember: unique().on(table.crewId, table.userId),
}));
```

```typescript
// lib/db/schema/declarations.ts
export const declarationTagEnum = pgEnum('declaration_tag', [
  'feature', 'bugfix', 'refactor', 'design', 'docs'
]);

export const declarations = pgTable('declarations', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  crewId: uuid('crew_id').references(() => crews.id).notNull(),
  text: text('text').notNull(),                           // max 280 chars enforced at app level
  tag: declarationTagEnum('tag'),
  date: text('date').notNull(),                           // 'YYYY-MM-DD' in user's local timezone
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  // One declaration per user per crew per day
  uniqueDaily: unique().on(table.userId, table.crewId, table.date),
  // Feed queries: get today's declarations for a crew
  crewDateIdx: index('idx_declarations_crew_date').on(table.crewId, table.date),
}));
```

```typescript
// lib/db/schema/ship-checks.ts
export const proofTypeEnum = pgEnum('proof_type', ['screenshot', 'url', 'github', 'text']);
export const verificationStatusEnum = pgEnum('verification_status', [
  'pending', 'approved', 'rejected', 'auto_approved', 'auto_verified'
]);

export const shipChecks = pgTable('ship_checks', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  crewId: uuid('crew_id').references(() => crews.id).notNull(),
  declarationId: uuid('declaration_id').references(() => declarations.id),
  proofType: proofTypeEnum('proof_type').notNull(),
  proofData: text('proof_data').notNull(),                // URL, storage path, or text content
  proofMetadata: jsonb('proof_metadata'),                 // GitHub PR details, URL check results, etc.
  isLate: boolean('is_late').default(false).notNull(),
  verificationStatus: verificationStatusEnum('verification_status').default('pending').notNull(),
  verificationLevel: integer('verification_level').default(1).notNull(), // 1=self, 2=peer, 3=auto
  resubmissionCount: integer('resubmission_count').default(0).notNull(),
  approvalCount: integer('approval_count').default(0).notNull(),
  rejectionCount: integer('rejection_count').default(0).notNull(),
  requiredApprovals: integer('required_approvals').notNull(), // calculated at submission: floor(crew_size/2)
  reviewDeadline: timestamp('review_deadline', { withTimezone: true }), // 48h from submission
  date: text('date').notNull(),                           // 'YYYY-MM-DD' in user's local timezone
  submittedAt: timestamp('submitted_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  // One ship check per user per crew per day
  uniqueDaily: unique().on(table.userId, table.crewId, table.date),
  // Feed queries
  crewDateIdx: index('idx_ship_checks_crew_date').on(table.crewId, table.date),
  // Auto-approve cron: find pending checks past deadline
  pendingDeadlineIdx: index('idx_ship_checks_pending_deadline')
    .on(table.verificationStatus, table.reviewDeadline),
  // User history for scoring
  userStatusIdx: index('idx_ship_checks_user_status')
    .on(table.userId, table.verificationStatus, table.date),
}));
```

```typescript
// lib/db/schema/proof-reviews.ts
export const verdictEnum = pgEnum('verdict', ['approve', 'reject']);

export const proofReviews = pgTable('proof_reviews', {
  id: uuid('id').primaryKey().defaultRandom(),
  shipCheckId: uuid('ship_check_id').references(() => shipChecks.id, { onDelete: 'cascade' }).notNull(),
  reviewerId: uuid('reviewer_id').references(() => users.id).notNull(),
  verdict: verdictEnum('verdict').notNull(),
  reason: text('reason'),                                 // Required for rejections
  version: integer('version').notNull(),                  // Optimistic concurrency version
  reviewedAt: timestamp('reviewed_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  // One review per reviewer per ship check
  uniqueReview: unique().on(table.shipCheckId, table.reviewerId),
}));
```

```typescript
// lib/db/schema/reactions.ts
export const reactionTypeEnum = pgEnum('reaction_type', ['ship_it', 'lgtm', 'on_it', 'call_out']);
export const targetTypeEnum = pgEnum('target_type', ['declaration', 'ship_check']);

export const reactions = pgTable('reactions', {
  id: uuid('id').primaryKey().defaultRandom(),
  targetType: targetTypeEnum('target_type').notNull(),
  targetId: uuid('target_id').notNull(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  type: reactionTypeEnum('type').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  // One reaction per user per target
  uniqueReaction: unique().on(table.targetType, table.targetId, table.userId),
  // Load reactions for a feed item
  targetIdx: index('idx_reactions_target').on(table.targetType, table.targetId),
}));
```

```typescript
// lib/db/schema/nudges.ts
export const nudges = pgTable('nudges', {
  id: uuid('id').primaryKey().defaultRandom(),
  nudgerId: uuid('nudger_id').references(() => users.id).notNull(),
  nudgedId: uuid('nudged_id').references(() => users.id).notNull(),
  crewId: uuid('crew_id').references(() => crews.id).notNull(),
  date: text('date').notNull(),                           // 'YYYY-MM-DD' for 1/day enforcement
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  // One nudge per nudger per nudged per day
  uniqueDaily: unique().on(table.nudgerId, table.nudgedId, table.date),
}));
```

```typescript
// lib/db/schema/league-standings.ts
export const leagueEnum = pgEnum('league', ['bronze', 'silver', 'gold', 'platinum', 'diamond']);

export const leagueStandings = pgTable('league_standings', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  league: leagueEnum('league').default('bronze').notNull(),
  weeklyScore: real('weekly_score').default(0).notNull(),
  rank: integer('rank'),
  weekStart: text('week_start').notNull(),                // 'YYYY-MM-DD' Monday of the week
  promoted: boolean('promoted').default(false).notNull(),
  demoted: boolean('demoted').default(false).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  // One row per user per week
  uniqueWeek: unique().on(table.userId, table.weekStart),
  // Leaderboard: sort by score within a league for a given week
  leaderboardIdx: index('idx_league_standings_leaderboard')
    .on(table.weekStart, table.league, table.weeklyScore),
  // User history
  userHistoryIdx: index('idx_league_standings_user').on(table.userId, table.weekStart),
}));
```

```typescript
// lib/db/schema/daily-activity.ts
export const dailyActivity = pgTable('daily_activity', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  date: text('date').notNull(),                           // 'YYYY-MM-DD'
  declared: boolean('declared').default(false).notNull(),
  proofSubmitted: boolean('proof_submitted').default(false).notNull(),
  proofVerified: boolean('proof_verified').default(false).notNull(),
  milestoneProgress: boolean('milestone_progress').default(false).notNull(),
  activityLevel: integer('activity_level').default(0).notNull(), // 0-4 for heat map intensity
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  // One row per user per day
  uniqueDay: unique().on(table.userId, table.date),
  // Heat map: get 365 days for a user, ordered by date
  heatMapIdx: index('idx_daily_activity_heatmap').on(table.userId, table.date),
}));
```

```typescript
// lib/db/schema/notification-preferences.ts
export const notificationPreferences = pgTable('notification_preferences', {
  userId: uuid('user_id').references(() => users.id).primaryKey(),
  activeHoursStart: text('active_hours_start').default('09:00').notNull(), // HH:MM local time
  activeHoursEnd: text('active_hours_end').default('18:00').notNull(),
  morningDeclarationTime: text('morning_declaration_time').default('09:00').notNull(),
  quietHoursStart: text('quiet_hours_start').default('22:00').notNull(),
  quietHoursEnd: text('quiet_hours_end').default('07:00').notNull(),
  declarationReminders: boolean('declaration_reminders').default(true).notNull(),
  shipCheck: boolean('ship_check').default(true).notNull(),
  crewActivity: boolean('crew_activity').default(true).notNull(),
  reviewRequests: boolean('review_requests').default(true).notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});
```

```typescript
// lib/db/schema/goals.ts
export const goalStatusEnum = pgEnum('goal_status', ['active', 'completed', 'abandoned', 'pivoted']);

export const goals = pgTable('goals', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  crewId: uuid('crew_id').references(() => crews.id),
  title: text('title').notNull(),
  description: text('description'),
  deadline: timestamp('deadline', { withTimezone: true }),
  status: goalStatusEnum('status').default('active').notNull(),
  complexity: integer('complexity'),                      // 1-13 Fibonacci scale
  peerComplexity: integer('peer_complexity'),             // Median of crew ratings
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  completedAt: timestamp('completed_at', { withTimezone: true }),
});
```

```typescript
// lib/db/schema/github-installations.ts
export const githubInstallations = pgTable('github_installations', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  installationId: integer('installation_id').unique().notNull(), // GitHub App installation ID
  accountLogin: text('account_login').notNull(),                  // GitHub username
  accessToken: text('access_token'),                              // Encrypted installation access token
  tokenExpiresAt: timestamp('token_expires_at', { withTimezone: true }),
  repos: jsonb('repos'),                                          // Selected repos [{id, name, fullName}]
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});
```

```typescript
// lib/db/schema/reflections.ts
export const moodEnum = pgEnum('mood', ['shipped', 'progress', 'stuck', 'break']);

export const reflections = pgTable('reflections', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  crewId: uuid('crew_id').references(() => crews.id),
  mood: moodEnum('mood').notNull(),
  note: text('note'),
  date: text('date').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  uniqueDaily: unique().on(table.userId, table.crewId, table.date),
}));
```

### Indexing Strategy Summary

| Query Pattern | Index | Table | Rationale |
|---------------|-------|-------|-----------|
| Crew feed (today's activity) | `(crew_id, date)` | declarations, ship_checks | Most frequent query; used every page load |
| Heat map (365 days per user) | `(user_id, date)` | daily_activity | Profile page; range scan on date |
| Leaderboard (weekly by league) | `(week_start, league, weekly_score)` | league_standings | Sorted scan within league |
| User league history | `(user_id, week_start)` | league_standings | Profile page history |
| Pending auto-approvals | `(verification_status, review_deadline)` | ship_checks | Cron job; partial index on 'pending' |
| Reactions for feed item | `(target_type, target_id)` | reactions | Loaded with each feed item |
| User scoring inputs | `(user_id, verification_status, date)` | ship_checks | Batch scoring engine |

**Partial index recommendation for the auto-approve cron:**

```sql
CREATE INDEX idx_ship_checks_auto_approve
  ON ship_checks (review_deadline)
  WHERE verification_status = 'pending';
```

This is far more efficient than a composite index since it only indexes the small subset of checks that are pending review.

---

## 4. Row Level Security Policies

RLS policies are the authorization layer. Every table that users can read or write gets RLS enabled.

**Confidence: HIGH** -- These are standard Supabase RLS patterns.

### Core Principle

Use `auth.uid()` to get the authenticated user's Supabase auth ID, then join to `users.auth_id` for application-level user ID. Create a helper function to avoid repeating this join:

```sql
-- Helper: get app user ID from auth ID
CREATE OR REPLACE FUNCTION public.current_user_id()
RETURNS uuid AS $$
  SELECT id FROM public.users WHERE auth_id = auth.uid()
$$ LANGUAGE sql STABLE SECURITY DEFINER;
```

### RLS Policies by Table

```sql
-- USERS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Anyone can read non-deleted users (public profiles)
CREATE POLICY "Users are publicly readable"
  ON users FOR SELECT
  USING (deleted_at IS NULL);

-- Users can only update their own profile
CREATE POLICY "Users can update own profile"
  ON users FOR UPDATE
  USING (auth_id = auth.uid())
  WITH CHECK (auth_id = auth.uid());

-- CREWS
ALTER TABLE crews ENABLE ROW LEVEL SECURITY;

-- Active crews are publicly readable (for invite links)
CREATE POLICY "Active crews are readable"
  ON crews FOR SELECT
  USING (status != 'archived');

-- Only authenticated users can create crews
CREATE POLICY "Authenticated users can create crews"
  ON crews FOR INSERT
  WITH CHECK (auth.uid() IS NOT NULL);

-- Only crew owner can update crew
CREATE POLICY "Crew owner can update"
  ON crews FOR UPDATE
  USING (created_by = public.current_user_id());

-- CREW_MEMBERS
ALTER TABLE crew_members ENABLE ROW LEVEL SECURITY;

-- Members can see their crew's members
CREATE POLICY "Crew members can see membership"
  ON crew_members FOR SELECT
  USING (
    crew_id IN (
      SELECT cm.crew_id FROM crew_members cm
      WHERE cm.user_id = public.current_user_id()
    )
  );

-- Users can join crews (insert themselves)
CREATE POLICY "Users can join crews"
  ON crew_members FOR INSERT
  WITH CHECK (user_id = public.current_user_id());

-- DECLARATIONS
ALTER TABLE declarations ENABLE ROW LEVEL SECURITY;

-- Crew members can see declarations in their crews
CREATE POLICY "Crew members can read declarations"
  ON declarations FOR SELECT
  USING (
    crew_id IN (
      SELECT cm.crew_id FROM crew_members cm
      WHERE cm.user_id = public.current_user_id()
    )
  );

-- Users can only create their own declarations
CREATE POLICY "Users create own declarations"
  ON declarations FOR INSERT
  WITH CHECK (user_id = public.current_user_id());

-- SHIP_CHECKS
ALTER TABLE ship_checks ENABLE ROW LEVEL SECURITY;

-- Crew members can see ship checks in their crews
CREATE POLICY "Crew members can read ship checks"
  ON ship_checks FOR SELECT
  USING (
    crew_id IN (
      SELECT cm.crew_id FROM crew_members cm
      WHERE cm.user_id = public.current_user_id()
    )
  );

-- Users can create their own ship checks
CREATE POLICY "Users create own ship checks"
  ON ship_checks FOR INSERT
  WITH CHECK (user_id = public.current_user_id());

-- System can update ship checks (for approval counts)
-- Handled via service role in Server Actions, not direct client updates

-- PROOF_REVIEWS
ALTER TABLE proof_reviews ENABLE ROW LEVEL SECURITY;

-- Crew members can see reviews for ship checks in their crews
CREATE POLICY "Crew members can read reviews"
  ON proof_reviews FOR SELECT
  USING (
    ship_check_id IN (
      SELECT sc.id FROM ship_checks sc
      WHERE sc.crew_id IN (
        SELECT cm.crew_id FROM crew_members cm
        WHERE cm.user_id = public.current_user_id()
      )
    )
  );

-- Users can create reviews (not for their own ship checks)
CREATE POLICY "Users create reviews for crewmates"
  ON proof_reviews FOR INSERT
  WITH CHECK (
    reviewer_id = public.current_user_id()
    AND ship_check_id NOT IN (
      SELECT id FROM ship_checks WHERE user_id = public.current_user_id()
    )
  );

-- DAILY_ACTIVITY
ALTER TABLE daily_activity ENABLE ROW LEVEL SECURITY;

-- Users can read their own activity (heat map)
CREATE POLICY "Users read own daily activity"
  ON daily_activity FOR SELECT
  USING (user_id = public.current_user_id());

-- Public profiles: allow reading any user's activity
CREATE POLICY "Public activity is readable"
  ON daily_activity FOR SELECT
  USING (
    user_id IN (SELECT id FROM users WHERE deleted_at IS NULL)
  );
```

### RLS Performance Notes

- The `current_user_id()` function is marked `STABLE` so PostgreSQL caches it within a single transaction.
- The `IN (SELECT crew_id FROM crew_members WHERE ...)` subquery pattern is standard for multi-tenant crew-scoping. For large crew counts, this stays efficient because users are in 1-3 crews max.
- **Do not use RLS for the cron/webhook API routes.** Those use the Supabase service role client which bypasses RLS entirely. This is intentional -- cron jobs and webhooks need cross-user access.

---

## 5. API Route Structure & Middleware

### Middleware (Authentication)

```typescript
// middleware.ts (root level)
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => request.cookies.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) => {
            request.cookies.set(name, value);
            response.cookies.set(name, value, options);
          });
        },
      },
    }
  );

  // Refresh session (important: must call getUser, not getSession)
  const { data: { user } } = await supabase.auth.getUser();

  // Protected routes: redirect to login if not authenticated
  if (!user && request.nextUrl.pathname.startsWith('/(app)')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // Onboarding redirect: send un-onboarded users to setup
  // (checked in (app) layout.tsx, not middleware, to avoid DB call in middleware)

  return response;
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

### API Route Organization

| Route | Method | Purpose | Auth |
|-------|--------|---------|------|
| `/api/webhooks/github` | POST | Receive GitHub App webhooks | Signature verification |
| `/api/cron/ship-check` | GET | Schedule random ship check notifications | Vercel cron secret |
| `/api/cron/league-reset` | GET | Weekly league promotion/demotion | Vercel cron secret |
| `/api/cron/daily-summary` | GET | Aggregate daily activity for heat map | Vercel cron secret |
| `/api/cron/auto-approve` | GET | Auto-approve 48h-expired pending reviews | Vercel cron secret |
| `/api/upload/signed-url` | POST | Generate signed upload URL for proof screenshots | Supabase auth |

### Cron Job Security

```typescript
// Shared cron auth helper: lib/utils/cron-auth.ts
export function verifyCronSecret(request: Request): boolean {
  const authHeader = request.headers.get('authorization');
  return authHeader === `Bearer ${process.env.CRON_SECRET}`;
}
```

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/ship-check",
      "schedule": "*/5 * * * *"
    },
    {
      "path": "/api/cron/auto-approve",
      "schedule": "*/30 * * * *"
    },
    {
      "path": "/api/cron/daily-summary",
      "schedule": "0 * * * *"
    },
    {
      "path": "/api/cron/league-reset",
      "schedule": "0 0 * * 1"
    }
  ]
}
```

### Server Actions vs API Routes Decision Matrix

| Operation | Use | Reason |
|-----------|-----|--------|
| Create declaration | Server Action | User-facing mutation, needs auth, benefits from form integration |
| Submit proof | Server Action | User-facing mutation, immediate UI feedback needed |
| Submit review | Server Action | User-facing mutation with optimistic concurrency |
| Send reaction | Server Action | User-facing, lightweight |
| Send nudge | Server Action | User-facing, needs rate limiting |
| GitHub webhook | API Route | External caller, needs signature verification, not form-based |
| Cron jobs | API Route | Triggered by Vercel scheduler, not user-initiated |
| Signed URL generation | API Route | Client-side upload flow needs a URL before the upload |

---

## 6. Real-time Subscriptions

### Channel Architecture

For DevReal's crew model (3-6 members per crew, users in 1-3 crews), Supabase Realtime Postgres Changes is the right approach. Each crew gets its own channel.

**Confidence: HIGH** -- Standard Supabase Realtime pattern.

```typescript
// lib/hooks/use-realtime-feed.ts
'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';
import type { Declaration, ShipCheck, Reaction } from '@/lib/types';

type FeedEvent =
  | { type: 'declaration'; data: Declaration }
  | { type: 'ship_check'; data: ShipCheck }
  | { type: 'reaction'; data: Reaction };

export function useRealtimeFeed(crewId: string, initialItems: FeedEvent[]) {
  const [items, setItems] = useState<FeedEvent[]>(initialItems);
  const supabase = createClient();

  useEffect(() => {
    const channel = supabase
      .channel(`crew:${crewId}`)
      // Listen for new declarations
      .on('postgres_changes', {
        event: 'INSERT',
        schema: 'public',
        table: 'declarations',
        filter: `crew_id=eq.${crewId}`,
      }, (payload) => {
        setItems(prev => [
          { type: 'declaration', data: payload.new as Declaration },
          ...prev,
        ]);
      })
      // Listen for new ship checks (proofs)
      .on('postgres_changes', {
        event: 'INSERT',
        schema: 'public',
        table: 'ship_checks',
        filter: `crew_id=eq.${crewId}`,
      }, (payload) => {
        setItems(prev => [
          { type: 'ship_check', data: payload.new as ShipCheck },
          ...prev,
        ]);
      })
      // Listen for verification status updates
      .on('postgres_changes', {
        event: 'UPDATE',
        schema: 'public',
        table: 'ship_checks',
        filter: `crew_id=eq.${crewId}`,
      }, (payload) => {
        setItems(prev =>
          prev.map(item =>
            item.type === 'ship_check' && item.data.id === (payload.new as ShipCheck).id
              ? { ...item, data: payload.new as ShipCheck }
              : item
          )
        );
      })
      // Listen for reactions
      .on('postgres_changes', {
        event: 'INSERT',
        schema: 'public',
        table: 'reactions',
        // Note: reactions use target_id, not crew_id.
        // We filter client-side for reactions on items in this crew.
      }, (payload) => {
        setItems(prev => [
          { type: 'reaction', data: payload.new as Reaction },
          ...prev,
        ]);
      })
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [crewId, supabase]);

  return items;
}
```

### Presence (Who's Online)

```typescript
// lib/hooks/use-crew-presence.ts
'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';

interface PresenceState {
  userId: string;
  username: string;
  onlineAt: string;
}

export function useCrewPresence(crewId: string, currentUser: { id: string; username: string }) {
  const [onlineMembers, setOnlineMembers] = useState<PresenceState[]>([]);
  const supabase = createClient();

  useEffect(() => {
    const channel = supabase
      .channel(`crew-presence:${crewId}`)
      .on('presence', { event: 'sync' }, () => {
        const state = channel.presenceState<PresenceState>();
        const members = Object.values(state).flat();
        setOnlineMembers(members);
      })
      .subscribe(async (status) => {
        if (status === 'SUBSCRIBED') {
          await channel.track({
            userId: currentUser.id,
            username: currentUser.username,
            onlineAt: new Date().toISOString(),
          });
        }
      });

    return () => {
      supabase.removeChannel(channel);
    };
  }, [crewId, currentUser, supabase]);

  return onlineMembers;
}
```

### Real-time Performance Considerations

| Concern | Approach |
|---------|----------|
| Connection limits (200 free, 500 Pro) | DevReal crews are small (3-6). Even 50 concurrent crews = ~250 connections. Free tier handles MVP. |
| Subscription fan-out | Filter by `crew_id` at the Postgres Changes level, not client-side. This uses Supabase's server-side filtering. |
| Reactions channel noise | Reactions on the `reactions` table don't filter by crew_id (polymorphic target). Accept this on MVP; at scale, use Broadcast for reactions instead of Postgres Changes. |
| Multi-crew users | Subscribe to 1-3 channels simultaneously (one per crew). Each is lightweight. |
| Reconnection | Supabase client handles reconnection automatically. The `subscribe()` call re-establishes on network recovery. |

### When to Use Broadcast vs Postgres Changes

| Use Postgres Changes for: | Use Broadcast for: |
|---|---|
| Declarations (persisted, need DB trigger) | Typing indicators |
| Ship checks (persisted) | "User is viewing proof" |
| Review status updates (persisted) | Ephemeral reactions (MVP: skip this) |
| Nudges (persisted) | Real-time countdown sync (not needed: each client runs its own timer) |

---

## 7. GitHub App Webhook Pipeline

### Architecture

```
GitHub (push/PR/release event)
        |
        v
  POST /api/webhooks/github
        |
  1. Verify webhook signature (HMAC-SHA256)
  2. Parse event type + action
  3. Look up GitHub installation -> DevReal user
  4. Process event based on type
  5. Update ship_check if relevant proof exists
  6. Return 200 immediately (process async if needed)
```

### Implementation

```typescript
// app/api/webhooks/github/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { verifyGitHubWebhook } from '@/lib/utils/crypto';
import { processGitHubEvent } from '@/lib/services/github';

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get('x-hub-signature-256');
  const event = request.headers.get('x-github-event');
  const deliveryId = request.headers.get('x-github-delivery');

  // 1. Verify signature
  if (!signature || !verifyGitHubWebhook(body, signature)) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 401 });
  }

  // 2. Parse payload
  const payload = JSON.parse(body);

  // 3. Acknowledge immediately (GitHub expects quick response)
  // Process the event -- for MVP, this is synchronous and fast enough.
  // At scale, push to a queue (Inngest, QStash, or Supabase Edge Function).
  try {
    await processGitHubEvent(event!, payload, deliveryId!);
  } catch (error) {
    console.error('GitHub webhook processing error:', error);
    // Still return 200 to prevent GitHub retries for app-level errors
    // Log the error for investigation
  }

  return NextResponse.json({ received: true });
}
```

```typescript
// lib/services/github.ts
import { db } from '@/lib/db';
import { githubInstallations, shipChecks } from '@/lib/db/schema';
import { eq, and } from 'drizzle-orm';

export async function processGitHubEvent(
  event: string,
  payload: any,
  deliveryId: string
) {
  // Only process events we care about
  const relevantEvents: Record<string, (payload: any) => Promise<void>> = {
    'pull_request': processPullRequest,
    'push': processPush,
    'release': processRelease,
  };

  const handler = relevantEvents[event];
  if (!handler) return;

  await handler(payload);
}

async function processPullRequest(payload: any) {
  // Only care about merged PRs
  if (payload.action !== 'closed' || !payload.pull_request.merged) return;

  const installationId = payload.installation.id;
  const repoFullName = payload.repository.full_name;
  const prTitle = payload.pull_request.title;
  const prNumber = payload.pull_request.number;
  const prUrl = payload.pull_request.html_url;
  const additions = payload.pull_request.additions;
  const deletions = payload.pull_request.deletions;
  const mergedAt = payload.pull_request.merged_at;

  // Find the DevReal user for this installation
  const [installation] = await db
    .select()
    .from(githubInstallations)
    .where(eq(githubInstallations.installationId, installationId));

  if (!installation) return;

  // Check if the user has a pending ship check for today
  // If so, auto-verify it
  const today = getUserLocalDate(installation.userId); // helper function
  const [pendingCheck] = await db
    .select()
    .from(shipChecks)
    .where(and(
      eq(shipChecks.userId, installation.userId),
      eq(shipChecks.date, today),
      eq(shipChecks.verificationStatus, 'pending'),
      eq(shipChecks.proofType, 'github'),
    ));

  if (pendingCheck) {
    await db
      .update(shipChecks)
      .set({
        verificationStatus: 'auto_verified',
        verificationLevel: 3,
        proofMetadata: {
          type: 'pr_merged',
          repo: repoFullName,
          prTitle,
          prNumber,
          prUrl,
          additions,
          deletions,
          mergedAt,
        },
      })
      .where(eq(shipChecks.id, pendingCheck.id));
  }

  // Store the event for the activity feed regardless
  // (User can reference it when submitting future proof)
}
```

### Webhook Signature Verification

```typescript
// lib/utils/crypto.ts
import { createHmac, timingSafeEqual } from 'crypto';

export function verifyGitHubWebhook(body: string, signature: string): boolean {
  const secret = process.env.GITHUB_WEBHOOK_SECRET!;
  const expected = 'sha256=' + createHmac('sha256', secret)
    .update(body)
    .digest('hex');

  // Timing-safe comparison to prevent timing attacks
  try {
    return timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expected)
    );
  } catch {
    return false;
  }
}
```

### GitHub App Setup Requirements

| Config | Value |
|--------|-------|
| Webhook URL | `https://devreal.dev/api/webhooks/github` |
| Webhook secret | Random 32+ char secret stored in `GITHUB_WEBHOOK_SECRET` |
| Permissions | Contents (read), Pull requests (read), Deployments (read), Metadata (read) |
| Events | `push`, `pull_request`, `release`, `deployment_status` |
| Installation scope | User-selected repositories |

---

## 8. Scheduled Notifications (Cron)

### The Ship Check Scheduling Problem

DevReal needs to send each user a random notification within their configured active hours. This is timezone-aware and per-user.

**Approach: Scan-and-schedule via Vercel cron running every 5 minutes.**

```
Every 5 minutes:
  1. Query users whose random ship check time falls in the current 5-min window
  2. For each such user, trigger notification
```

### Pre-computing Random Times

When the day starts (midnight in user's timezone), compute a random ship check time for that day and store it. The cron job then just finds users whose scheduled time is "now."

```typescript
// Database table for scheduled notifications
export const scheduledNotifications = pgTable('scheduled_notifications', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  type: text('type').notNull(),                           // 'ship_check', 'declaration_reminder'
  scheduledFor: timestamp('scheduled_for', { withTimezone: true }).notNull(),
  date: text('date').notNull(),                           // 'YYYY-MM-DD'
  sent: boolean('sent').default(false).notNull(),
  sentAt: timestamp('sent_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  uniqueDaily: unique().on(table.userId, table.type, table.date),
  // Cron query: unsent notifications scheduled before now
  pendingIdx: index('idx_scheduled_pending')
    .on(table.sent, table.scheduledFor),
}));
```

### Cron Implementation

```typescript
// app/api/cron/ship-check/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { verifyCronSecret } from '@/lib/utils/cron-auth';
import { db } from '@/lib/db';
import { scheduledNotifications, users, notificationPreferences } from '@/lib/db/schema';
import { eq, and, lte, sql } from 'drizzle-orm';

export async function GET(request: NextRequest) {
  if (!verifyCronSecret(request)) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const now = new Date();

  // Step 1: Schedule today's notifications for users who don't have one yet.
  // This runs on every invocation but the unique constraint prevents duplicates.
  await scheduleNewNotifications(now);

  // Step 2: Find and send notifications that are due.
  const dueNotifications = await db
    .select({
      notification: scheduledNotifications,
      user: users,
    })
    .from(scheduledNotifications)
    .innerJoin(users, eq(scheduledNotifications.userId, users.id))
    .where(and(
      eq(scheduledNotifications.sent, false),
      eq(scheduledNotifications.type, 'ship_check'),
      lte(scheduledNotifications.scheduledFor, now),
    ))
    .limit(100); // Process in batches

  // Step 3: Send notifications (FCM or in-app)
  for (const { notification, user } of dueNotifications) {
    try {
      await sendShipCheckNotification(user);
      await db
        .update(scheduledNotifications)
        .set({ sent: true, sentAt: now })
        .where(eq(scheduledNotifications.id, notification.id));
    } catch (error) {
      console.error(`Failed to send notification to ${user.id}:`, error);
    }
  }

  return NextResponse.json({
    scheduled: dueNotifications.length,
    timestamp: now.toISOString(),
  });
}

async function scheduleNewNotifications(now: Date) {
  // Find users who need a ship check scheduled for today
  // but don't have one yet.
  // "Today" is relative to each user's timezone.
  //
  // This query:
  // 1. Gets all active users with notification prefs
  // 2. Calculates their local date
  // 3. Checks if they already have a scheduled notification for that date
  // 4. If not, inserts one with a random time in their active hours

  await db.execute(sql`
    INSERT INTO scheduled_notifications (id, user_id, type, scheduled_for, date)
    SELECT
      gen_random_uuid(),
      u.id,
      'ship_check',
      -- Random time within active hours, converted to UTC
      (
        (CURRENT_DATE AT TIME ZONE u.timezone)
        + (np.active_hours_start || ' hours')::interval
        + (random() * (
          EXTRACT(EPOCH FROM (
            (np.active_hours_end || ':00')::time - (np.active_hours_start || ':00')::time
          )) / 3600.0
        ) || ' hours')::interval
      ) AT TIME ZONE u.timezone,
      TO_CHAR(NOW() AT TIME ZONE u.timezone, 'YYYY-MM-DD')
    FROM users u
    JOIN notification_preferences np ON np.user_id = u.id
    WHERE u.deleted_at IS NULL
      AND np.ship_check = true
      -- Only for users where "today" in their timezone doesn't have a notification yet
      AND NOT EXISTS (
        SELECT 1 FROM scheduled_notifications sn
        WHERE sn.user_id = u.id
          AND sn.type = 'ship_check'
          AND sn.date = TO_CHAR(NOW() AT TIME ZONE u.timezone, 'YYYY-MM-DD')
      )
    ON CONFLICT (user_id, type, date) DO NOTHING
  `);
}
```

### Cron Schedule Summary

| Cron Route | Schedule | Vercel Cron Expression | Purpose |
|------------|----------|----------------------|---------|
| `/api/cron/ship-check` | Every 5 min | `*/5 * * * *` | Schedule and send ship check notifications |
| `/api/cron/auto-approve` | Every 30 min | `*/30 * * * *` | Auto-approve pending reviews past 48h deadline |
| `/api/cron/daily-summary` | Every hour | `0 * * * *` | Update daily_activity for users whose day just ended |
| `/api/cron/league-reset` | Monday 00:00 UTC | `0 0 * * 1` | Calculate ranks, promotions/demotions, create new week |

**Vercel cron limits (Pro plan):** Up to 40 cron jobs. DevReal uses 4, well within limits.

**Vercel cron limits (Hobby plan):** 2 cron jobs, max daily execution. This is insufficient for DevReal. **The Pro plan ($20/month) is required for production.** For development, use `pg_cron` in Supabase or a local scheduler.

---

## 9. File Uploads & Signed URLs

### Upload Flow

```
Client                    API Route                  Supabase Storage
  |                          |                              |
  |  1. Request signed URL   |                              |
  |  POST /api/upload/       |                              |
  |     signed-url           |                              |
  |  {bucket, path, type}    |                              |
  |------------------------->|                              |
  |                          |  2. Create signed upload URL  |
  |                          |  storage.createSignedUploadUrl|
  |                          |------------------------------>|
  |                          |  3. Return {signedUrl, path}  |
  |                          |<------------------------------|
  |  4. Receive signed URL   |                              |
  |<-------------------------|                              |
  |                                                         |
  |  5. Upload file directly to Storage                     |
  |  PUT {signedUrl}                                        |
  |  (no server middleman)                                  |
  |-------------------------------------------------------->|
  |                                                         |
  |  6. Store path in ship_checks.proof_data               |
  |  (via Server Action)                                    |
  |                                                         |
```

### Signed URL API Route

```typescript
// app/api/upload/signed-url/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createServerClient } from '@/lib/supabase/server';

export async function POST(request: NextRequest) {
  const supabase = await createServerClient();

  // Verify auth
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { fileName, contentType } = await request.json();

  // Validate content type
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];
  if (!allowedTypes.includes(contentType)) {
    return NextResponse.json({ error: 'Invalid file type' }, { status: 400 });
  }

  // Generate unique path: proofs/{userId}/{date}/{uuid}.{ext}
  const ext = contentType.split('/')[1];
  const date = new Date().toISOString().split('T')[0];
  const path = `proofs/${user.id}/${date}/${crypto.randomUUID()}.${ext}`;

  // Create signed upload URL (expires in 5 minutes)
  const { data, error } = await supabase.storage
    .from('proof-screenshots')
    .createSignedUploadUrl(path);

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({
    signedUrl: data.signedUrl,
    path: data.path,
    token: data.token,
  });
}
```

### Reading Proof Images (Signed Download URLs)

When rendering proof images in the feed, generate short-lived signed download URLs:

```typescript
// lib/services/upload.ts
import { createAdminClient } from '@/lib/supabase/admin';

export async function getSignedProofUrl(path: string): Promise<string | null> {
  const supabase = createAdminClient();

  const { data, error } = await supabase.storage
    .from('proof-screenshots')
    .createSignedUrl(path, 3600); // 1 hour expiry

  if (error) return null;
  return data.signedUrl;
}
```

### Storage Bucket Configuration

```sql
-- Supabase Storage bucket (created via dashboard or migration)
-- Bucket: proof-screenshots
-- Public: false (require signed URLs)
-- File size limit: 5MB
-- Allowed MIME types: image/jpeg, image/png, image/webp, image/gif
```

### Client-Side Upload Component Pattern

```typescript
// components/ship-check/screenshot-upload.tsx (simplified)
'use client';

export function ScreenshotUpload({ onUploadComplete }: { onUploadComplete: (path: string) => void }) {
  const [uploading, setUploading] = useState(false);

  async function handleFileSelect(file: File) {
    setUploading(true);

    // 1. Client-side compression (optional, for large images)
    const compressed = await compressImage(file, { maxWidth: 1920, quality: 0.8 });

    // 2. Get signed URL from our API
    const res = await fetch('/api/upload/signed-url', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        fileName: file.name,
        contentType: compressed.type,
      }),
    });
    const { signedUrl, path, token } = await res.json();

    // 3. Upload directly to Supabase Storage (bypasses our server)
    await fetch(signedUrl, {
      method: 'PUT',
      headers: { 'Content-Type': compressed.type },
      body: compressed,
    });

    // 4. Return the path for storage in DB
    onUploadComplete(path);
    setUploading(false);
  }

  // ... render file input / drag-drop zone
}
```

---

## 10. Scoring Engine

### Architecture: Batch Calculation

**Decision: Batch, not real-time.**

Scores are recalculated on a schedule, not on every event. Rationale:
- Composite scores depend on aggregates (active_days / expected_days, completed_goals / declared_goals) that span weeks of data.
- Nobody needs sub-second score freshness. Updating every few hours is sufficient.
- Batch is simpler, debuggable, and avoids race conditions from concurrent events.

### Calculation Pipeline

```
Weekly league score update:
  Trigger: /api/cron/league-reset (Monday 00:00 UTC)

  1. For each user active this week:
     a. Calculate consistency_score (active_days / expected_days * streak_multiplier)
     b. Calculate completion_rate (completed / declared, weighted by on-time/late/pivoted)
     c. Calculate shipping_velocity (sum(complexity * speed_factor) / days)
     d. Calculate ambition_score (avg_complexity * frequency_factor)
     e. composite = 0.35*consistency + 0.25*completion + 0.20*velocity + 0.20*ambition

  2. Rank users within their league
  3. Determine promotions/demotions (top 3 promote, bottom 3 demote)
  4. Insert new league_standings row for next week
  5. Update users.composite_score and users.lifetime_xp
```

### Score Calculation Implementation

```typescript
// lib/services/scoring.ts

export interface ScoreComponents {
  consistency: number;    // 0-1
  completion: number;     // 0-1
  velocity: number;       // 0-1 (normalized)
  ambition: number;       // 0-1 (normalized)
  composite: number;      // 0-100 weighted sum
}

export function calculateCompositeScore(inputs: {
  activeDays: number;
  expectedDays: number;
  currentStreak: number;
  completedGoals: number;
  declaredGoals: number;
  onTimeCompletions: number;
  lateCompletions: number;
  pivotedGoals: number;
  totalComplexityPoints: number;
  totalSpeedFactoredPoints: number;
  periodDays: number;
  avgGoalComplexity: number;
  goalsPerWeek: number;
}): ScoreComponents {
  const {
    activeDays, expectedDays, currentStreak,
    completedGoals, declaredGoals,
    onTimeCompletions, lateCompletions, pivotedGoals,
    totalComplexityPoints, totalSpeedFactoredPoints,
    periodDays, avgGoalComplexity, goalsPerWeek,
  } = inputs;

  // Consistency: 35%
  const streakMultiplier = Math.min(1.0 + currentStreak * 0.02, 1.5);
  const consistency = expectedDays > 0
    ? Math.min((activeDays / expectedDays) * streakMultiplier, 1.0)
    : 0;

  // Completion Rate: 25%
  const weightedCompletions = declaredGoals > 0
    ? (onTimeCompletions * 1.0 + lateCompletions * 0.8 + pivotedGoals * 0.7) / declaredGoals
    : 0;
  const completion = Math.min(weightedCompletions, 1.0);

  // Shipping Velocity: 20% (normalized to 0-1)
  const rawVelocity = periodDays > 0 ? totalSpeedFactoredPoints / periodDays : 0;
  // Normalize: 5 complexity-adjusted points per day = top score
  const velocity = Math.min(rawVelocity / 5.0, 1.0);

  // Ambition Score: 20% (normalized to 0-1)
  const frequencyFactor = Math.min(goalsPerWeek / 2, 1.5);
  // Normalize: avg complexity of 8 with 2 goals/week = top score
  const rawAmbition = (avgGoalComplexity / 13.0) * frequencyFactor;
  const ambition = Math.min(rawAmbition, 1.0);

  // Composite: weighted sum, scaled to 0-100
  const composite = (
    consistency * 35 +
    completion * 25 +
    velocity * 20 +
    ambition * 20
  );

  return { consistency, completion, velocity, ambition, composite };
}
```

### Daily Activity Update (for Heat Map)

```typescript
// app/api/cron/daily-summary/route.ts
// Runs every hour, updates daily_activity for users whose day just ended

export async function GET(request: NextRequest) {
  if (!verifyCronSecret(request)) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Find users where it's just past midnight (00:00-00:59) in their timezone
  // These users need their previous day's activity summarized
  await db.execute(sql`
    INSERT INTO daily_activity (id, user_id, date, declared, proof_submitted, proof_verified, activity_level)
    SELECT
      gen_random_uuid(),
      u.id,
      TO_CHAR((NOW() - interval '1 hour') AT TIME ZONE u.timezone, 'YYYY-MM-DD'),
      EXISTS (
        SELECT 1 FROM declarations d
        WHERE d.user_id = u.id
          AND d.date = TO_CHAR((NOW() - interval '1 hour') AT TIME ZONE u.timezone, 'YYYY-MM-DD')
      ),
      EXISTS (
        SELECT 1 FROM ship_checks sc
        WHERE sc.user_id = u.id
          AND sc.date = TO_CHAR((NOW() - interval '1 hour') AT TIME ZONE u.timezone, 'YYYY-MM-DD')
      ),
      EXISTS (
        SELECT 1 FROM ship_checks sc
        WHERE sc.user_id = u.id
          AND sc.date = TO_CHAR((NOW() - interval '1 hour') AT TIME ZONE u.timezone, 'YYYY-MM-DD')
          AND sc.verification_status IN ('approved', 'auto_verified', 'auto_approved')
      ),
      false, -- milestone_progress: checked separately
      -- activity_level: 0=none, 1=declared, 2=declared+proof, 3=declared+verified, 4=milestone
      CASE
        WHEN EXISTS (SELECT 1 FROM ship_checks sc WHERE sc.user_id = u.id
          AND sc.date = TO_CHAR((NOW() - interval '1 hour') AT TIME ZONE u.timezone, 'YYYY-MM-DD')
          AND sc.verification_status IN ('approved', 'auto_verified', 'auto_approved'))
        THEN 3
        WHEN EXISTS (SELECT 1 FROM ship_checks sc WHERE sc.user_id = u.id
          AND sc.date = TO_CHAR((NOW() - interval '1 hour') AT TIME ZONE u.timezone, 'YYYY-MM-DD'))
        THEN 2
        WHEN EXISTS (SELECT 1 FROM declarations d WHERE d.user_id = u.id
          AND d.date = TO_CHAR((NOW() - interval '1 hour') AT TIME ZONE u.timezone, 'YYYY-MM-DD'))
        THEN 1
        ELSE 0
      END
    FROM users u
    WHERE u.deleted_at IS NULL
      -- Only process users where it's 00:xx in their timezone right now
      AND EXTRACT(HOUR FROM NOW() AT TIME ZONE u.timezone) = 0
    ON CONFLICT (user_id, date) DO UPDATE SET
      declared = EXCLUDED.declared,
      proof_submitted = EXCLUDED.proof_submitted,
      proof_verified = EXCLUDED.proof_verified,
      activity_level = EXCLUDED.activity_level
  `);

  return NextResponse.json({ success: true });
}
```

---

## 11. Weekly League Reset System

### Reset Flow (Monday 00:00 UTC)

```typescript
// app/api/cron/league-reset/route.ts

export async function GET(request: NextRequest) {
  if (!verifyCronSecret(request)) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const thisMonday = getThisMonday(); // 'YYYY-MM-DD'
  const lastMonday = getLastMonday(); // 'YYYY-MM-DD'

  // Step 1: Calculate final weekly scores for last week
  await calculateWeeklyScores(lastMonday);

  // Step 2: Rank users within each league
  await rankWithinLeagues(lastMonday);

  // Step 3: Determine promotions and demotions
  await determinePromotionsAndDemotions(lastMonday);

  // Step 4: Create new league_standings rows for this week
  await createNewWeekStandings(thisMonday, lastMonday);

  // Step 5: Update user composite scores and lifetime XP
  await updateUserAggregates();

  // Step 6: Handle mid-week joiners (they start fresh this week)
  // Already handled: users without a league_standings row for this week
  // get one created with their current league and score 0.

  return NextResponse.json({ success: true, week: thisMonday });
}

async function determinePromotionsAndDemotions(weekStart: string) {
  // For each league, find top 3 and bottom 3
  const leagues = ['bronze', 'silver', 'gold', 'platinum', 'diamond'];

  for (const league of leagues) {
    const standings = await db
      .select()
      .from(leagueStandings)
      .where(and(
        eq(leagueStandings.weekStart, weekStart),
        eq(leagueStandings.league, league),
      ))
      .orderBy(desc(leagueStandings.weeklyScore));

    if (standings.length === 0) continue;

    // Top 3 promote (except Diamond -- already at top)
    if (league !== 'diamond') {
      const promotees = standings.slice(0, 3);
      for (const s of promotees) {
        await db
          .update(leagueStandings)
          .set({ promoted: true })
          .where(eq(leagueStandings.id, s.id));
      }
    }

    // Bottom 3 demote (except Bronze -- already at bottom)
    if (league !== 'bronze') {
      const demotees = standings.slice(-3);
      for (const s of demotees) {
        await db
          .update(leagueStandings)
          .set({ demoted: true })
          .where(eq(leagueStandings.id, s.id));
      }
    }
  }
}

async function createNewWeekStandings(thisMonday: string, lastMonday: string) {
  // Create standings for the new week based on last week's promotions/demotions
  await db.execute(sql`
    INSERT INTO league_standings (id, user_id, league, weekly_score, week_start)
    SELECT
      gen_random_uuid(),
      ls.user_id,
      CASE
        WHEN ls.promoted AND ls.league = 'bronze' THEN 'silver'
        WHEN ls.promoted AND ls.league = 'silver' THEN 'gold'
        WHEN ls.promoted AND ls.league = 'gold' THEN 'platinum'
        WHEN ls.promoted AND ls.league = 'platinum' THEN 'diamond'
        WHEN ls.demoted AND ls.league = 'diamond' THEN 'platinum'
        WHEN ls.demoted AND ls.league = 'platinum' THEN 'gold'
        WHEN ls.demoted AND ls.league = 'gold' THEN 'silver'
        WHEN ls.demoted AND ls.league = 'silver' THEN 'bronze'
        ELSE ls.league
      END::league,
      0,
      ${thisMonday}
    FROM league_standings ls
    WHERE ls.week_start = ${lastMonday}
    ON CONFLICT (user_id, week_start) DO NOTHING
  `);
}
```

---

## 12. Rate Limiting

### Per-Crew Daily Limits (1 Declaration + 1 Proof Per Day)

**Primary enforcement: Database unique constraints.**

The `declarations` and `ship_checks` tables both have:
```sql
UNIQUE (user_id, crew_id, date)
```

This is the strongest possible enforcement -- the database itself rejects duplicates. No application-level race conditions are possible.

**Application-level check (for better UX -- show "already declared" before attempting insert):**

```typescript
// lib/actions/declarations.ts
'use server';

import { db } from '@/lib/db';
import { declarations } from '@/lib/db/schema';
import { eq, and } from 'drizzle-orm';

export async function createDeclaration(formData: FormData) {
  const userId = await getCurrentUserId();
  const crewId = formData.get('crewId') as string;
  const text = formData.get('text') as string;
  const tag = formData.get('tag') as string;
  const today = await getUserLocalDate(userId);

  // Pre-check (for UX, not security)
  const existing = await db
    .select({ id: declarations.id })
    .from(declarations)
    .where(and(
      eq(declarations.userId, userId),
      eq(declarations.crewId, crewId),
      eq(declarations.date, today),
    ));

  if (existing.length > 0) {
    return { error: 'You already declared in this crew today' };
  }

  // Insert (unique constraint is the real guard)
  try {
    const [declaration] = await db
      .insert(declarations)
      .values({ userId, crewId, text, tag, date: today })
      .returning();

    // Update daily_activity
    await upsertDailyActivity(userId, today, { declared: true });

    return { data: declaration };
  } catch (error: any) {
    if (error.code === '23505') { // Unique violation
      return { error: 'You already declared in this crew today' };
    }
    throw error;
  }
}
```

### Nudge Rate Limiting (1 Per Person Per Day)

Same pattern -- `UNIQUE (nudger_id, nudged_id, date)` constraint on the `nudges` table.

### API Rate Limiting (for webhooks and external-facing routes)

For the GitHub webhook endpoint, GitHub itself handles rate limiting. For the upload signed-url endpoint:

```typescript
// lib/utils/rate-limit.ts
// Simple in-memory rate limiter for serverless (resets on cold start, which is acceptable)

const rateLimitMap = new Map<string, { count: number; resetAt: number }>();

export function rateLimit(key: string, limit: number, windowMs: number): boolean {
  const now = Date.now();
  const entry = rateLimitMap.get(key);

  if (!entry || now > entry.resetAt) {
    rateLimitMap.set(key, { count: 1, resetAt: now + windowMs });
    return true; // allowed
  }

  if (entry.count >= limit) {
    return false; // rate limited
  }

  entry.count++;
  return true; // allowed
}
```

For production at scale, use Vercel's `@vercel/kv` or Upstash Redis for persistent rate limiting across serverless instances. For MVP, the in-memory approach plus database constraints is sufficient.

### Optimistic Concurrency for Reviews

```typescript
// lib/actions/reviews.ts
'use server';

export async function submitReview(shipCheckId: string, verdict: 'approve' | 'reject', reason?: string) {
  const reviewerId = await getCurrentUserId();

  // Start a transaction for atomic review + count update
  return await db.transaction(async (tx) => {
    // 1. Get current ship check with version
    const [check] = await tx
      .select()
      .from(shipChecks)
      .where(eq(shipChecks.id, shipCheckId))
      .for('update'); // Row-level lock

    if (!check) throw new Error('Ship check not found');
    if (check.verificationStatus !== 'pending') {
      return { error: 'This proof has already been reviewed' };
    }

    // 2. Insert review (unique constraint prevents double-review)
    try {
      await tx.insert(proofReviews).values({
        shipCheckId,
        reviewerId,
        verdict,
        reason,
        version: check.approvalCount + check.rejectionCount + 1,
      });
    } catch (error: any) {
      if (error.code === '23505') {
        return { error: 'You already reviewed this proof' };
      }
      throw error;
    }

    // 3. Update counts atomically
    const newApprovalCount = verdict === 'approve' ? check.approvalCount + 1 : check.approvalCount;
    const newRejectionCount = verdict === 'reject' ? check.rejectionCount + 1 : check.rejectionCount;

    // 4. Check if majority reached
    let newStatus = 'pending' as const;
    if (newApprovalCount >= check.requiredApprovals) {
      newStatus = 'approved';
    }
    // Rejection: if remaining possible approvals can't reach majority
    const totalReviews = newApprovalCount + newRejectionCount;
    const crewSize = await getCrewSize(check.crewId);
    const possibleRemainingApprovals = (crewSize - 1) - totalReviews; // -1 for submitter
    if (newApprovalCount + possibleRemainingApprovals < check.requiredApprovals) {
      newStatus = 'rejected';
    }

    await tx
      .update(shipChecks)
      .set({
        approvalCount: newApprovalCount,
        rejectionCount: newRejectionCount,
        verificationStatus: newStatus,
        verificationLevel: newStatus === 'approved' ? 2 : check.verificationLevel,
      })
      .where(eq(shipChecks.id, shipCheckId));

    return { data: { status: newStatus } };
  });
}
```

---

## 13. Component Boundaries & Data Flow

### Component Boundary Diagram

```
+------------------------------------------------------------------+
|                        PRESENTATION LAYER                         |
|                                                                   |
|  Pages (RSC)        Components (Client)      Hooks               |
|  - Fetch data       - Interactive UI         - Real-time subs    |
|  - Pass to          - Optimistic updates     - Presence          |
|    components       - Form handling          - Countdown timer   |
|  - Auth checks      - Animations             - Optimistic state  |
+------------------------------------------------------------------+
                              |
                     Server Actions / API Routes
                              |
+------------------------------------------------------------------+
|                        BUSINESS LOGIC LAYER                       |
|                                                                   |
|  Actions (lib/actions/)    Services (lib/services/)               |
|  - Input validation        - GitHub event processing              |
|  - Auth enforcement        - Score calculation                    |
|  - Call services           - League management                    |
|  - Return results          - Notification dispatch                |
|                            - Upload management                    |
+------------------------------------------------------------------+
                              |
                     Drizzle ORM Queries
                              |
+------------------------------------------------------------------+
|                        DATA LAYER                                 |
|                                                                   |
|  Schema (lib/db/schema/)   Queries (lib/db/queries/)              |
|  - Table definitions       - Named query functions                |
|  - Type inference          - Complex joins/aggregations           |
|  - Relations               - Parameterized, reusable              |
|                                                                   |
|  Supabase Services                                                |
|  - Auth (session mgmt)     - Storage (file uploads)               |
|  - Realtime (subscriptions) - RLS (authorization)                 |
+------------------------------------------------------------------+
```

### Data Flow: Declaration Submission

```
User types declaration
        |
        v
Client Component (form)
        |
        v
Server Action: createDeclaration()
  1. Validate input (280 chars, valid tag)
  2. Get user ID from session
  3. Calculate user's local date
  4. Check existing declaration (UX pre-check)
  5. INSERT into declarations table
  6. UPSERT daily_activity
  7. Return result
        |
        v
Supabase Postgres Changes fires
        |
        v
Other crew members' useRealtimeFeed hook
receives INSERT event
        |
        v
Feed UI updates in real-time
```

### Data Flow: Proof Review (with Optimistic Concurrency)

```
Reviewer taps "Approve"
        |
        v
Client Component: optimistic UI update
  (show approval immediately)
        |
        v
Server Action: submitReview()
  1. BEGIN transaction
  2. SELECT ship_check FOR UPDATE (row lock)
  3. Check status is still 'pending'
  4. INSERT proof_review (unique constraint guard)
  5. UPDATE approval_count atomically
  6. Check if majority reached
  7. UPDATE verification_status if resolved
  8. COMMIT
        |
    success? ----no----> Revert optimistic update
        |                show "already reviewed" message
       yes
        |
        v
Supabase Postgres Changes fires UPDATE on ship_checks
        |
        v
All crew members see updated verification status
```

---

## 14. Build Order & Dependencies

### Dependency Graph

```
Phase 1: Foundation (no dependencies)
  +-- Auth (Supabase Auth + middleware)
  +-- Database schema (Drizzle migrations)
  +-- User profiles
  +-- Basic UI shell (layout, sidebar, navigation)

Phase 2: Core Loop (depends on Phase 1)
  +-- Crew creation/joining (depends on: auth, users)
  +-- Declarations (depends on: auth, users, crews)
  +-- Crew feed with real-time (depends on: declarations, crews)

Phase 3: Proof & Verification (depends on Phase 2)
  +-- File upload + signed URLs (depends on: auth)
  +-- Ship check / proof submission (depends on: declarations, upload)
  +-- Proof review system (depends on: ship checks, crews)
  +-- Reactions (depends on: declarations, ship checks)

Phase 4: Gamification (depends on Phase 3)
  +-- Daily activity tracking (depends on: declarations, ship checks)
  +-- Streak system (depends on: daily activity)
  +-- Scoring engine (depends on: declarations, ship checks, goals)
  +-- Weekly leagues (depends on: scoring engine)
  +-- Leaderboard (depends on: leagues)
  +-- Heat map (depends on: daily activity)
  +-- Celebrations (depends on: streaks, leagues)

Phase 5: Integrations (can run parallel with Phase 4)
  +-- GitHub App setup
  +-- Webhook pipeline (depends on: ship checks)
  +-- Auto-verification (depends on: webhook pipeline, proof reviews)

Phase 6: Notifications & Polish (depends on Phase 4 + 5)
  +-- Ship check scheduling cron (depends on: notification prefs)
  +-- Push notifications / FCM
  +-- Nudge system (depends on: crews)
  +-- Auto-approve cron (depends on: proof reviews)
  +-- Evening reflections
```

### Build Order Implications for Roadmap

1. **Auth + DB + Users + UI shell MUST come first.** Everything depends on having authenticated users with profiles in the database.

2. **Crews before declarations.** Declarations are scoped per-crew. You cannot test declarations without at least one crew.

3. **Declarations before ship checks.** The daily flow is declare then prove. Ship checks reference declarations (optionally). Building ship checks first would leave an incomplete user flow.

4. **File uploads can be built independently** of the rest of Phase 3, but must be ready before ship check UI is complete.

5. **Scoring engine can lag behind.** Users can submit proofs and get peer reviews without scores being calculated. Scores are a read-only aggregate -- they don't block any write operations.

6. **GitHub integration is independent** of the core loop. It enhances verification but the system works without it. Build it in parallel with gamification.

7. **Notifications are the last mile.** The entire app works without push notifications (users just check the app manually). Adding cron-scheduled ship checks is polish, not foundation.

---

## 15. Anti-Patterns to Avoid

### Anti-Pattern 1: Client-Side Auth Checks Only

**What:** Checking `session` only in client components; skipping RLS or server-side auth validation.
**Why bad:** Anyone can call your API routes or Server Actions directly. Client-side checks are bypassable.
**Instead:** Always verify auth in Server Actions (`supabase.auth.getUser()`), enable RLS on every table, and use the `current_user_id()` helper in RLS policies.

### Anti-Pattern 2: Polling for Real-time Updates

**What:** Using `setInterval` to refetch crew feed data every N seconds.
**Why bad:** Wasteful of bandwidth and server resources. Creates janky UI with stale data between polls.
**Instead:** Use Supabase Postgres Changes subscriptions. Data arrives within seconds of the INSERT/UPDATE.

### Anti-Pattern 3: Storing Timezone Offsets Instead of IANA Names

**What:** Storing `timezone: 'UTC-5'` or `timezone_offset: -300`.
**Why bad:** Offsets don't account for daylight saving time. UTC-5 is EST, but the same location becomes UTC-4 during EDT. Your ship check scheduling will be wrong for 6 months of the year.
**Instead:** Store IANA timezone names (`America/New_York`). Use them with `AT TIME ZONE` in SQL or `Intl.DateTimeFormat` in JS.

### Anti-Pattern 4: Calculating Scores On Every Page Load

**What:** Running the composite scoring query in the leaderboard page's data fetcher.
**Why bad:** The scoring query aggregates across multiple tables with date ranges. Running it for every leaderboard view is expensive and slow.
**Instead:** Pre-calculate scores in the batch cron job. The leaderboard page just reads from `league_standings` -- a single-table sorted query.

### Anti-Pattern 5: Passing File Uploads Through the Server

**What:** Uploading proof screenshots to a Next.js API route, then forwarding to Supabase Storage.
**Why bad:** Doubles bandwidth usage, increases latency, and can hit Vercel's 4.5MB request body limit. On the Hobby plan, serverless function timeout is 10 seconds.
**Instead:** Use signed upload URLs. The client uploads directly to Supabase Storage.

### Anti-Pattern 6: Global Real-time Channels

**What:** One channel for all users, filtering events client-side by crew_id.
**Why bad:** Every user receives every event for every crew. At 100 active crews, each user processes 100x more events than needed.
**Instead:** One channel per crew (`crew:{crewId}`). Users subscribe to only their 1-3 crews.

### Anti-Pattern 7: No Optimistic Concurrency on Reviews

**What:** Two reviewers submit approvals at the same time, both read `approval_count = 1`, both write `approval_count = 2`, resulting in a lost update.
**Why bad:** The count is wrong, and majority detection fails.
**Instead:** Use `SELECT ... FOR UPDATE` inside a transaction, or use atomic increment (`SET approval_count = approval_count + 1`).

### Anti-Pattern 8: Using `getSession()` Instead of `getUser()` in Server Code

**What:** Calling `supabase.auth.getSession()` in middleware or Server Actions.
**Why bad:** `getSession()` reads from cookies without validating the JWT with the Supabase Auth server. A tampered JWT would pass. This is explicitly warned against in Supabase docs.
**Instead:** Always use `supabase.auth.getUser()` for server-side auth checks. It validates the JWT by calling the Auth server.

---

## 16. Scalability Considerations

| Concern | At 100 users | At 5K users | At 50K users |
|---------|--------------|-------------|--------------|
| Database | Supabase Free (500MB) | Supabase Pro ($25/mo) | Supabase Pro + read replicas |
| Realtime connections | ~30 concurrent (free tier: 200) | ~500 concurrent (Pro: 500) | Need connection pooling or Supabase Scale plan |
| Storage | Supabase Storage free (1GB) | Supabase Storage Pro | Migrate to Cloudflare R2 |
| Cron job load | Trivial | Scan ~5K users per run (fast) | Batch in chunks, consider pg_cron |
| Leaderboard queries | Instant | Instant (indexed) | Instant (indexed, materialized views if needed) |
| Heat map queries | 365 rows per user | Same | Same (pre-aggregated) |
| Webhook volume | Negligible | ~100/day | ~5K/day; add queue (Inngest/QStash) |
| Score calculation | < 1 second | < 10 seconds | Batch in chunks, ~5 minutes |
| Vercel functions | Free tier (100K/mo) | Pro ($20/mo) | Pro; monitor cold starts |
| Total monthly cost | $0 | ~$45 | ~$150+ |

### Migration Triggers

| Trigger | Action |
|---------|--------|
| >200 concurrent Realtime connections | Upgrade to Supabase Pro |
| >1GB proof screenshots | Migrate Storage to Cloudflare R2 |
| Webhook processing >10s | Add async queue (Inngest or QStash) |
| Score calculation >60s | Move to pg_cron database-level job |
| >100K Vercel function invocations/mo | Upgrade to Vercel Pro |
| Leaderboard queries >200ms | Add materialized views with refresh |

---

## Sources

- **Next.js 15 App Router project structure** -- Official docs (verified via direct fetch, v16.1.6 current): https://nextjs.org/docs/app/getting-started/project-structure
- **Supabase Realtime** -- Project research `02-tech-stack.md` (Broadcast, Presence, Postgres Changes patterns)
- **Supabase RLS** -- Training data patterns (MEDIUM confidence for exact syntax; verified patterns are standard Supabase)
- **Vercel Cron Jobs** -- Training data (MEDIUM confidence for exact limits; Pro plan required for >2 crons)
- **GitHub App Webhooks** -- Project research `07-verification-mechanisms.md` (webhook events, signature verification, permissions)
- **Drizzle ORM schema** -- Project research `02-tech-stack.md` (schema patterns, index syntax)
- **Scoring formulas** -- Project research `03-gamification-mechanics.md` (exact weights and formulas from feature spec)
- **File upload pattern** -- Training data (standard Supabase signed URL pattern, MEDIUM confidence)

### Confidence Notes

| Area | Confidence | Note |
|------|------------|------|
| Folder structure | HIGH | Verified against Next.js 15 official docs |
| Database schema | HIGH | Derived directly from feature spec data model |
| RLS policies | MEDIUM | Standard patterns; exact SQL syntax should be tested |
| Cron scheduling | MEDIUM | Vercel cron limits from training data; verify current Pro plan limits |
| Real-time subscriptions | HIGH | Standard Supabase Postgres Changes; patterns from official docs/research |
| GitHub webhooks | HIGH | Well-documented in project research and GitHub official docs |
| Scoring engine | HIGH | Formulas defined in feature spec; implementation is straightforward math |
| File upload flow | MEDIUM | Standard signed URL pattern; exact `createSignedUploadUrl` API should be verified |
| Optimistic concurrency | HIGH | Standard PostgreSQL `SELECT FOR UPDATE` pattern |
