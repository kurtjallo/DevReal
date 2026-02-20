# Requirements: DevReal

**Defined:** 2026-02-20
**Core Value:** Proof over promises. The gap between "announcing what you're building" and "proving you built it" is where DevReal lives.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Authentication

- [ ] **AUTH-01**: User can sign up and log in with GitHub OAuth (auto-pull name + avatar)
- [ ] **AUTH-02**: User can pick a unique username during signup
- [ ] **AUTH-03**: User can sign up and log in with Google OAuth
- [ ] **AUTH-04**: User session persists across browser refresh
- [ ] **AUTH-05**: User sets timezone during account setup

### Onboarding

- [ ] **ONBD-01**: New user completes onboarding in 4 screens max (landing > OAuth > profile > crew > goal > feed)
- [ ] **ONBD-02**: User can skip crew creation/joining during onboarding (with recurring nudges)
- [ ] **ONBD-03**: User sets first daily goal during onboarding

### Crews

- [ ] **CREW-01**: User can create a crew with a name (3-6 member capacity)
- [ ] **CREW-02**: User can join a crew via shareable invite link or short code
- [ ] **CREW-03**: Crew owner can transfer ownership; must transfer before leaving
- [ ] **CREW-04**: User can nudge a crewmate once per day (neutral framing)
- [ ] **CREW-05**: Crew feed shows stall indicator when a crewmate hasn't declared today
- [ ] **CREW-06**: Crew feed shows chronological activity with status indicators (green/yellow/gray)

### Core Loop

- [ ] **LOOP-01**: User can make a daily declaration ("What are you shipping today?") with 280 char max
- [ ] **LOOP-02**: User can tag declaration with category (Feature/Bug fix/Refactor/Design/Docs)
- [ ] **LOOP-03**: User sees which crewmates have already declared today
- [ ] **LOOP-04**: User can submit proof via screenshot upload
- [ ] **LOOP-05**: User can submit proof via URL with automatic liveness check
- [ ] **LOOP-06**: User can submit proof via GitHub link (auto-pulls PR/commit details)
- [ ] **LOOP-07**: User can submit proof via text description
- [ ] **LOOP-08**: User receives random ship check notification during active hours with 5-minute response window
- [ ] **LOOP-09**: User can submit late proof until midnight with "Late" tag and reduced points
- [ ] **LOOP-10**: User can complete optional evening reflection with mood indicator and crew daily summary
- [ ] **LOOP-11**: Declarations are scoped per-crew (separate declarations per crew)
- [ ] **LOOP-12**: Proof without prior declaration is allowed (lower consistency score)

### Verification

- [ ] **VERF-01**: Majority crew approval required within 48 hours (auto-approves on timeout)
- [ ] **VERF-02**: User can one-tap approve or reject (with reason) a crewmate's proof
- [ ] **VERF-03**: Rejected proof allows resubmission with better evidence (max 2, restarts 48h window)
- [ ] **VERF-04**: GitHub-verified proofs (merged PRs, releases) skip crew review entirely
- [ ] **VERF-05**: Auto-verified proofs display verification badge in feed
- [ ] **VERF-06**: Proofs have 3 trust levels: self-reported, peer-verified, auto-verified
- [ ] **VERF-07**: User trust score builds over time and decays 5% per week of inactivity

### GitHub Integration

- [ ] **GHUB-01**: User can connect GitHub account via read-only GitHub App
- [ ] **GHUB-02**: System auto-detects PR merges and releases as verified proofs
- [ ] **GHUB-03**: URL proof submissions include automatic liveness check with content change detection

### Feed & Notifications

- [ ] **FEED-01**: User sees crew feed with declarations, proofs, and milestones (chronological)
- [ ] **FEED-02**: User can react to feed entries with "Ship it!", "LGTM", "On it", or "Call out"
- [ ] **FEED-03**: User receives push notifications for ship check, declaration reminder, and crew activity
- [ ] **FEED-04**: Maximum 3 notifications per day enforced
- [ ] **FEED-05**: User can configure quiet hours (no notifications during this window)

### Gamification

- [ ] **GAME-01**: User streak increments daily with at least one declaration or proof submitted
- [ ] **GAME-02**: Missing one day without a freeze resets streak to zero
- [ ] **GAME-03**: User has 2 streak freezes per month and 1 vacation week per quarter
- [ ] **GAME-04**: Crew can validate legitimate absence for unplanned breaks
- [ ] **GAME-05**: Composite score: Consistency 35%, Completion 25%, Velocity 20%, Ambition 20%
- [ ] **GAME-06**: Weekly leagues: Bronze > Silver > Gold > Platinum > Diamond
- [ ] **GAME-07**: League resets Monday; top 3 promote, bottom 3 demote
- [ ] **GAME-08**: Mid-week crew joiners start league competition next Monday
- [ ] **GAME-09**: Full league history preserved (one row per user per week)
- [ ] **GAME-10**: Permanent tiers: Starter (0-99) > Builder (100-499) > Shipper (500-1,499) > Architect (1,500-4,999) > Legend (5,000+)
- [ ] **GAME-11**: Celebration animations for declarations, proofs, streak milestones (7/30/100-day), and league promotions
- [ ] **GAME-12**: Shipping heat map on profile (52-week grid, color-coded by activity, tap for details)

### Profiles & UX

- [ ] **PROF-01**: User has public profile with streak counter, stats, and recent activity
- [ ] **PROF-02**: User can view other users' profiles
- [ ] **PROF-03**: Dark mode is default theme; user can switch to light mode
- [ ] **PROF-04**: Left sidebar navigation: Declare, Feed, Leaderboard, Profile (collapsible)
- [ ] **PROF-05**: All daily interactions complete in under 30 seconds

### Monetization

- [ ] **MNTZ-01**: Core features are 100% free for months 0-6
- [ ] **MNTZ-02**: Free tier forever: core proof submission, 1 crew, basic leaderboard, streaks, social features
- [ ] **MNTZ-03**: Pro tier scaffolding ($9.99/mo or $79.99/yr): up to 3 crews, advanced analytics, extended proof history, verified badge

### Privacy & Data

- [ ] **PRIV-01**: User-set timezone with midnight local reset (all timestamps stored UTC)
- [ ] **PRIV-02**: Per-user notification preferences (active hours, declaration time, quiet hours, toggles)
- [ ] **PRIV-03**: Proof files stored with signed URLs (time-limited access)
- [ ] **PRIV-04**: User can delete account with 30-day reversible grace period (soft delete + anonymize)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Crew Lifecycle

- **CLIF-01**: 7-day grace period when crew drops below 3 members
- **CLIF-02**: Dead crew auto-archive after 30 days all-inactive with reactivation option
- **CLIF-03**: Crew merge suggestion after grace period expires

### Monetization (Extended)

- **MNTZ-04**: Crew Pro tier ($4.99/mo per crew): larger crews, crew analytics
- **MNTZ-05**: Custom domain for Pro users

### Growth

- **GROW-01**: Shareable proof cards for social media
- **GROW-02**: Public portfolio mode for profiles
- **GROW-03**: Advanced analytics dashboard (Pro feature)

### Anti-Gaming

- **ANTI-01**: Pattern detection for trivial task gaming
- **ANTI-02**: Random audit checks on self-reported proofs

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Native mobile app (iOS/Android) | Web-first; revisit after product-market fit |
| PWA / offline support | Unnecessary complexity for MVP; standard web app |
| Monetary stakes / commitment contracts | Payment + legal complexity; Phase 2+ |
| Recruiter platform | Requires 10K+ profiles; Phase 3+ (months 12+) |
| GitLab / Bitbucket integration | GitHub-first for MVP; expand based on demand |
| Vercel / Netlify deploy detection | Phase 2+ integration |
| npm / PyPI publish detection | Phase 2+ integration |
| Figma API integration | Phase 2+ for design work |
| Challenge sponsorships | Requires scale; Phase 3+ |
| API access / job board | Platform features; Phase 3+ |
| Video proof uploads | Screenshot + URL + GitHub + text covers MVP needs |
| Anonymous usage | GitHub OAuth required for developer identity |
| Direct messaging | Crew feed + reactions cover social needs; DMs add moderation burden |
| Algorithmic feed | Small groups (3-6) don't need algorithmic ranking; chronological is simpler and fairer |

## Traceability

Which phases cover which requirements. Updated by create-roadmap.

| Requirement | Phase | Status |
|-------------|-------|--------|
| AUTH-01 | Phase 2 | Pending |
| AUTH-02 | Phase 2 | Pending |
| AUTH-03 | Phase 2 | Pending |
| AUTH-04 | Phase 2 | Pending |
| AUTH-05 | Phase 2 | Pending |
| ONBD-01 | Phase 3 | Pending |
| ONBD-02 | Phase 3 | Pending |
| ONBD-03 | Phase 3 | Pending |
| CREW-01 | Phase 3 | Pending |
| CREW-02 | Phase 3 | Pending |
| CREW-03 | Phase 3 | Pending |
| CREW-04 | Phase 5 | Pending |
| CREW-05 | Phase 5 | Pending |
| CREW-06 | Phase 5 | Pending |
| LOOP-01 | Phase 4 | Pending |
| LOOP-02 | Phase 4 | Pending |
| LOOP-03 | Phase 4 | Pending |
| LOOP-04 | Phase 4 | Pending |
| LOOP-05 | Phase 4 | Pending |
| LOOP-06 | Phase 4 | Pending |
| LOOP-07 | Phase 4 | Pending |
| LOOP-08 | Phase 6 | Pending |
| LOOP-09 | Phase 6 | Pending |
| LOOP-10 | Phase 6 | Pending |
| LOOP-11 | Phase 4 | Pending |
| LOOP-12 | Phase 4 | Pending |
| VERF-01 | Phase 7 | Pending |
| VERF-02 | Phase 7 | Pending |
| VERF-03 | Phase 7 | Pending |
| VERF-04 | Phase 9 | Pending |
| VERF-05 | Phase 9 | Pending |
| VERF-06 | Phase 7 | Pending |
| VERF-07 | Phase 7 | Pending |
| GHUB-01 | Phase 9 | Pending |
| GHUB-02 | Phase 9 | Pending |
| GHUB-03 | Phase 9 | Pending |
| FEED-01 | Phase 5 | Pending |
| FEED-02 | Phase 5 | Pending |
| FEED-03 | Phase 6 | Pending |
| FEED-04 | Phase 6 | Pending |
| FEED-05 | Phase 6 | Pending |
| GAME-01 | Phase 8 | Pending |
| GAME-02 | Phase 8 | Pending |
| GAME-03 | Phase 8 | Pending |
| GAME-04 | Phase 8 | Pending |
| GAME-05 | Phase 8 | Pending |
| GAME-06 | Phase 8 | Pending |
| GAME-07 | Phase 8 | Pending |
| GAME-08 | Phase 8 | Pending |
| GAME-09 | Phase 8 | Pending |
| GAME-10 | Phase 8 | Pending |
| GAME-11 | Phase 8 | Pending |
| GAME-12 | Phase 9 | Pending |
| PROF-01 | Phase 10 | Pending |
| PROF-02 | Phase 10 | Pending |
| PROF-03 | Phase 1 | Pending |
| PROF-04 | Phase 1 | Pending |
| PROF-05 | Phase 10 | Pending |
| MNTZ-01 | Phase 10 | Pending |
| MNTZ-02 | Phase 10 | Pending |
| MNTZ-03 | Phase 10 | Pending |
| PRIV-01 | Phase 2 | Pending |
| PRIV-02 | Phase 6 | Pending |
| PRIV-03 | Phase 10 | Pending |
| PRIV-04 | Phase 10 | Pending |

**Coverage:**
- v1 requirements: 65 total
- Mapped to phases: 65
- Unmapped: 0 ✓

---
*Requirements defined: 2026-02-20*
*Last updated: 2026-02-20 after roadmap creation*
