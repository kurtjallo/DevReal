# Roadmap: DevReal

## Overview

DevReal's roadmap builds from infrastructure through the complete accountability loop. Phase 1 scaffolds the project and design system. Phases 2-5 establish the core daily flow (auth > crews > declarations/proof > feed). Phase 6 adds the BeReal-style time-pressure mechanic. Phase 7 introduces peer verification. Phase 8 layers on the full gamification engine. Phase 9 connects GitHub for auto-verification. Phase 10 wraps with profiles, monetization scaffolding, and privacy features. Each phase delivers a complete, verifiable capability that builds on the previous.

## Phases

- [ ] **Phase 1: Foundation & Design System** - Project scaffolding, database schema, dark/light mode, navigation shell
- [ ] **Phase 2: Authentication** - GitHub/Google OAuth, sessions, timezone, username
- [ ] **Phase 3: Onboarding & Crews** - Onboarding flow, crew CRUD, invite links/codes
- [ ] **Phase 4: Core Loop** - Daily declarations, multi-format proof submission
- [ ] **Phase 5: Crew Feed & Social** - Chronological feed, reactions, nudges, stall detection
- [ ] **Phase 6: Ship Check & Notifications** - BeReal-style timer, push notifications, quiet hours
- [ ] **Phase 7: Peer Verification & Trust** - Majority approval, reject/resubmit, trust tiers
- [ ] **Phase 8: Gamification Engine** - Streaks, scoring, leagues, tiers, celebrations
- [ ] **Phase 9: GitHub Integration & Auto-Verification** - GitHub App, auto-verify, heat map
- [ ] **Phase 10: Profiles, Monetization & Privacy** - Public profiles, Pro tier, account management

## Phase Details

### Phase 1: Foundation & Design System
**Goal**: Working dev server with database, design system, and app shell
**Depends on**: Nothing (first phase)
**Requirements**: PROF-03, PROF-04
**Research needed**: Unlikely (standard Next.js + Supabase setup)
**Success Criteria** (what must be TRUE):
  1. Next.js 15 dev server runs with Supabase connected
  2. Drizzle ORM configured with database schema (all tables, indexes, RLS policies)
  3. Dark mode default works with light mode toggle
  4. Sidebar navigation shell renders (Declare, Feed, Leaderboard, Profile — collapsible)
  5. Tailwind 4 + shadcn component library configured
**Plans**: TBD

### Phase 2: Authentication
**Goal**: Users can sign up, log in, and maintain sessions with timezone configured
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, PRIV-01
**Research needed**: Unlikely (Supabase Auth handles OAuth flow)
**Success Criteria** (what must be TRUE):
  1. User can sign up and log in with GitHub OAuth (name + avatar auto-pulled)
  2. User can sign up and log in with Google OAuth
  3. User can pick a unique username during signup
  4. User session persists across browser refresh
  5. User timezone is set during account setup and stored for all time-based features
**Plans**: TBD

### Phase 3: Onboarding & Crews
**Goal**: New users complete onboarding and can create or join crews
**Depends on**: Phase 2
**Requirements**: ONBD-01, ONBD-02, ONBD-03, CREW-01, CREW-02, CREW-03
**Research needed**: Unlikely
**Success Criteria** (what must be TRUE):
  1. New user completes onboarding in 4 screens max
  2. User can skip crew creation/joining (with recurring nudges later)
  3. User can set first daily goal during onboarding
  4. User can create a crew with a name (3-6 member capacity)
  5. User can join a crew via shareable invite link or short code
  6. Crew owner can transfer ownership; must transfer before leaving
**Plans**: TBD

### Phase 4: Core Loop
**Goal**: Users can make daily declarations and submit proof in multiple formats
**Depends on**: Phase 3
**Requirements**: LOOP-01, LOOP-02, LOOP-03, LOOP-04, LOOP-05, LOOP-06, LOOP-07, LOOP-11, LOOP-12
**Research needed**: Likely (URL liveness check implementation, GitHub link auto-pull)
**Research topics**: URL preview/liveness checking libraries, GitHub API for PR/commit detail fetching, file upload to Supabase Storage
**Success Criteria** (what must be TRUE):
  1. User can declare what they're shipping today (280 chars, category tags)
  2. User sees which crewmates have already declared
  3. User can submit proof via screenshot upload
  4. User can submit proof via URL with liveness check
  5. User can submit proof via GitHub link (auto-pulls PR/commit details)
  6. User can submit proof via text description
  7. Declarations scoped per-crew; proof without declaration allowed
**Plans**: TBD

### Phase 5: Crew Feed & Social
**Goal**: Crew feed shows all activity with reactions, nudges, and stall detection
**Depends on**: Phase 4
**Requirements**: FEED-01, FEED-02, CREW-04, CREW-05, CREW-06
**Research needed**: Likely (Supabase Realtime for live feed updates)
**Research topics**: Supabase Realtime channels per crew, optimistic UI updates, real-time subscription patterns
**Success Criteria** (what must be TRUE):
  1. User sees chronological crew feed with declarations, proofs, and milestones
  2. Feed shows status indicators (green/yellow/gray) per crewmate
  3. User can react with "Ship it!", "LGTM", "On it", "Call out"
  4. User can nudge a crewmate once per day (neutral framing)
  5. Stall indicator shows when crewmate hasn't declared today
**Plans**: TBD

### Phase 6: Ship Check & Notifications
**Goal**: BeReal-style random ship check with push notifications and user preferences
**Depends on**: Phase 5
**Requirements**: LOOP-08, LOOP-09, LOOP-10, FEED-03, FEED-04, FEED-05, PRIV-02
**Research needed**: Likely (FCM push notifications, Vercel cron scheduling, ship check timer mechanics)
**Research topics**: Firebase Cloud Messaging setup with Next.js, Vercel cron job limits and scheduling patterns, per-user random time generation, 5-minute window enforcement server-side
**Success Criteria** (what must be TRUE):
  1. User receives random ship check notification during their active hours
  2. Ship check has 5-minute response window with countdown timer
  3. Late proof accepted until midnight with "Late" tag and reduced points
  4. User can complete optional evening reflection with mood indicator
  5. Push notifications work for ship check, declaration reminder, and crew activity
  6. Max 3 notifications per day enforced with configurable quiet hours
  7. User can configure notification preferences (active hours, quiet hours, toggles)
**Plans**: TBD

### Phase 7: Peer Verification & Trust
**Goal**: Crew-based proof verification with trust tiers that build over time
**Depends on**: Phase 5
**Requirements**: VERF-01, VERF-02, VERF-03, VERF-06, VERF-07
**Research needed**: Unlikely (standard approval workflow + trust score calculation)
**Success Criteria** (what must be TRUE):
  1. Majority crew approval required within 48h (auto-approves on timeout)
  2. User can one-tap approve or reject (with reason) a crewmate's proof
  3. Rejected proof allows resubmission with better evidence (max 2, restarts 48h window)
  4. Proofs have trust levels: self-reported and peer-verified
  5. Trust score builds over time and decays 5% per week of inactivity
**Plans**: TBD

### Phase 8: Gamification Engine
**Goal**: Full gamification with streaks, composite scoring, weekly leagues, tiers, and celebrations
**Depends on**: Phase 6, Phase 7
**Requirements**: GAME-01, GAME-02, GAME-03, GAME-04, GAME-05, GAME-06, GAME-07, GAME-08, GAME-09, GAME-10, GAME-11
**Research needed**: Likely (league bracket assignment algorithm, composite scoring formula, animation libraries)
**Research topics**: League bracket pooling strategy (20-30 users per bracket), Monday cron for league resets, canvas-confetti or Lottie for celebration animations, Framer Motion for micro-interactions
**Success Criteria** (what must be TRUE):
  1. Streak increments daily and resets on missed day without freeze
  2. Streak freezes (2/month) and vacation mode (1 week/quarter) work correctly
  3. Composite score calculates from Consistency/Completion/Velocity/Ambition
  4. Weekly leagues with Monday reset, top-3 promote, bottom-3 demote
  5. Permanent tiers progress from Starter to Legend (never reset)
  6. Celebration animations trigger for declarations, proofs, streak milestones, and league promotions
**Plans**: TBD

### Phase 9: GitHub Integration & Auto-Verification
**Goal**: GitHub App connected for auto-verified proofs; shipping heat map on profiles
**Depends on**: Phase 7
**Requirements**: GHUB-01, GHUB-02, GHUB-03, VERF-04, VERF-05, GAME-12
**Research needed**: Likely (GitHub App creation, webhook security, heat map rendering)
**Research topics**: GitHub App registration and webhook setup, HMAC signature verification, @octokit/rest for PR/release detection, SVG heat map component (52-week grid)
**Success Criteria** (what must be TRUE):
  1. User can connect GitHub account via read-only GitHub App
  2. System auto-detects PR merges and releases as verified proofs
  3. Auto-verified proofs skip crew review and display verification badge
  4. URL proof submissions include liveness check with content change detection
  5. Profile shows 52-week shipping heat map (color-coded, tap for details)
**Plans**: TBD

### Phase 10: Profiles, Monetization & Privacy
**Goal**: Public profiles, Pro tier scaffolding, and account management
**Depends on**: Phase 8, Phase 9
**Requirements**: PROF-01, PROF-02, PROF-05, MNTZ-01, MNTZ-02, MNTZ-03, PRIV-03, PRIV-04
**Research needed**: Likely (Stripe integration for Pro tier, signed URL implementation)
**Research topics**: Stripe Checkout / Customer Portal integration, Supabase Storage signed URLs, account deletion with data anonymization, performance optimization for 30-second interaction budget
**Success Criteria** (what must be TRUE):
  1. User has public profile with streak counter, stats, and recent activity
  2. User can view other users' profiles
  3. Free tier and Pro tier clearly delineated with feature gates
  4. Proof files stored with time-limited signed URLs
  5. User can delete account with 30-day reversible grace period (soft delete + anonymize)
  6. All daily interactions complete in under 30 seconds
**Plans**: TBD

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation & Design System | 0/TBD | Not started | - |
| 2. Authentication | 0/TBD | Not started | - |
| 3. Onboarding & Crews | 0/TBD | Not started | - |
| 4. Core Loop | 0/TBD | Not started | - |
| 5. Crew Feed & Social | 0/TBD | Not started | - |
| 6. Ship Check & Notifications | 0/TBD | Not started | - |
| 7. Peer Verification & Trust | 0/TBD | Not started | - |
| 8. Gamification Engine | 0/TBD | Not started | - |
| 9. GitHub Integration & Auto-Verification | 0/TBD | Not started | - |
| 10. Profiles, Monetization & Privacy | 0/TBD | Not started | - |
