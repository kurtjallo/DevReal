# DevReal Feature Spec

> BeReal-style accountability app for developers with crew-based verification and shipping leaderboards.

---

## Core Loop

### Morning Declaration (daily, user-configured time)
- "What are you shipping today?" — single text field, 280 char max
- Quick tags: Feature, Bug fix, Refactor, Design, Docs
- Shows which crewmates have already declared (social proof)
- 1 declaration per crew per day
- Scoped per-crew — multi-crew Pro users declare separately in each crew

### Random Ship Check (once daily, random time)
- Notification at a random time during user's active hours (respects timezone + quiet hours)
- **5-minute response window** — timer at top of screen, turns yellow at 2 min, red at 1 min
- Can submit late until midnight local time — gets "Late" tag, fewer points
- Missing entirely (no submission by midnight) = missed day, breaks streak unless frozen
- Proof without a morning declaration is allowed (lower consistency score)
- 1 ship check per crew per day

### Proof Submission (Ship Check UI)
- Tabbed interface below countdown timer: Screenshot | URL | GitHub | Text
- **Screenshot:** Upload from device, no filters, raw authenticity
- **URL:** Auto-previewed with OpenGraph + liveness check
- **GitHub:** Auto-pulls recent PRs/commits from connected repos; shows diff stats
- **Text:** Freeform description for non-visual work
- Submit button prominent; minimal UI, no distractions

### Optional Evening Reflection
- Notification at user-configured time (or skip)
- Quick mood/progress indicator: Shipped it / Made progress / Stuck / Took a break
- Optional note field
- Shows crew's daily summary
- Not required — just a bookend

---

## User Flows

### Onboarding (4 screens max)
1. **Landing page** — "Prove you ship." CTA: "Sign in with GitHub"
2. **GitHub OAuth** — read-only permissions (profile, email)
3. **Profile setup** — auto-pull name + avatar from GitHub, user picks a username
4. **Create or Join Crew** — skippable with persistent nudges
   - **Create:** Name your crew → get shareable invite link (devreal.dev/crew/abc123) + short code (ABC123)
   - **Join:** Enter invite code or open an invite link
5. **Set first goal** — "What are you building?" + deadline picker (This week / 2 weeks / This month / Custom)
6. **Land on crew feed** (or solo dashboard if crew skipped)

### Daily Loop (returning user)
1. Morning: Open app → Declaration prompt → Type + tag → Submit → See crew feed
2. Random time: Ship check notification → Open → 5-min countdown + proof tabs → Submit
3. Evening (optional): Reflection prompt → Quick mood tap → See crew summary
4. Throughout day: Check crew feed, react to crewmates, review pending proofs

### Crew Verification Flow
1. User submits proof → Crewmates get push notification + feed badge
2. Reviewer taps proof card → One-tap approve OR long-press reject with reason
3. Majority of crew must approve within 48 hours
4. Auto-approves if no majority reached within 48h
5. **If rejected:** Submitter notified with reason → Can resubmit (max 2 resubmissions, restarts 48h window)
6. **Auto-verified proofs** (GitHub PRs, deploys) skip crew review entirely; shown in feed with "Auto-verified via GitHub" badge

---

## Crews

- **Size:** 3-6 members
- **Invite mechanism:** Shareable link (devreal.dev/crew/abc123) + short code (ABC123)
- **Roles:** Owner and Member; owner must transfer ownership before leaving
- **Membership:** 1 free crew per user; Pro unlocks up to 3 crews
- **Crew feed:** Chronological activity (declarations, proofs, milestones) with status indicators
  - Green: declared + shipped
  - Yellow: declared only
  - Gray: inactive today
- **Crew reactions:** "Ship it!", "LGTM", "On it", "Call out"
- **Stall detection:** Neutral indicator when crewmate hasn't declared ("Haven't heard from @alex today")
- **Nudge:** 1 nudge per person per day; separate nudge entity for tracking/enforcement

---

## Gamification & Progression

### Weekly Leagues (Duolingo-style)
- Bronze > Silver > Gold > Platinum > Diamond
- Reset every Monday
- Top 3 promote, bottom 3 demote
- Full league history preserved (one row per user per week)
- Mid-week crew joiners: league starts next Monday

### Permanent Tiers
- Starter (0-99) > Builder (100-499) > Shipper (500-1,499) > Architect (1,500-4,999) > Legend (5,000+)
- Never reset

### Scoring (composite)
- Consistency (35%): active_days / expected_days x streak_multiplier (1.0 + current_streak x 0.02, capped at 1.5)
- Completion Rate (25%): completed_goals / declared_goals (on-time 1.0x, late 0.8x, pivoted 0.7x, abandoned 0.0x)
- Shipping Velocity (20%): sum(complexity_points x speed_factor) / time_period (Fibonacci scale 1-13)
- Ambition Score (20%): weighted_avg_complexity x frequency_factor

### Streaks
- Miss 1 day = streak resets
- 2 free streak freezes per month
- 1 vacation week per quarter (pauses all decay)
- Crew can validate "legitimate absence"

### Celebrations
| Event | Animation |
|-------|-----------|
| Daily declaration | Subtle checkmark |
| Proof submitted | Card glow + "Shipped!" label |
| 7-day streak | Confetti burst + badge |
| 30-day streak | Full-screen celebration + badge |
| 100-day streak | Special animation + permanent badge |
| League promotion | Trophy animation + new badge |

### Stakes (Phase 2+)
- Users stake $5-$100 on goals
- Success: money returned
- Failure: distributed to crew/charity (DevReal takes 10-15%)

---

## Verification & Integrations

### MVP Integrations
- **GitHub App** (read-only): PR merges, releases, commit activity
- **URL liveness checks:** HTTP check + content change detection

### 3-Tier Trust System
| Level | Trust | Description |
|-------|-------|-------------|
| 1 | Self-reported | User claims they shipped (lowest trust) |
| 2 | Peer-verified | Crew reviews and approves proof (medium trust) |
| 3 | Auto-verified | Automated signal from GitHub/deploy (highest trust) |

### Trust Score
- Builds over time based on verification history
- High trust = less crew review needed
- Low trust = additional proof required
- Decays with inactivity (5%/week), minimum floor at "new user" level
- New users start at medium trust

### Private/Closed-Source Projects
- Redacted screenshots (blurred sensitive info)
- Text descriptions reviewed by crew for plausibility
- No code access required — works for NDA-bound devs

### Anti-Gaming
- Peer-rated complexity (Fibonacci scale 1-13)
- Crew validation required before counting toward metrics
- Pattern detection for suspiciously low-effort submissions
- Random audit checks from users outside the crew

### Phase 2+ Integrations
- Vercel/Netlify deploy detection
- npm/PyPI package publish detection
- GitLab and Bitbucket support
- Figma API for design work

---

## Data Model

### Core Entities

**users** — id, username, display_name, avatar_url, github_id, email, trust_score, current_streak, longest_streak, composite_score, tier, timezone, created_at, deleted_at (soft delete)

**crews** — id, name, invite_code, invite_link, created_by, member_count, status (active/archived), created_at

**crew_members** — id, crew_id, user_id, role (owner/member), joined_at

**goals** — id, user_id, title, description, deadline, status (active/completed/abandoned), complexity (1-13 fibonacci), created_at

**declarations** — id, user_id, crew_id, text, tag (feature/bugfix/refactor/design/docs), created_at

**ship_checks** — id, user_id, crew_id, declaration_id (nullable), proof_type (screenshot/url/github/text), proof_data (url/text/path), is_late, verification_status (pending/approved/rejected), resubmission_count, submitted_at

**proof_reviews** — id, ship_check_id, reviewer_id, verdict (approve/reject), reason, reviewed_at

**reactions** — id, target_type (declaration/ship_check), target_id, user_id, type (ship_it/lgtm/on_it/call_out), created_at

**nudges** — id, nudger_id, nudged_id, crew_id, created_at

**league_standings** — id, user_id, league (bronze/silver/gold/platinum/diamond), weekly_score, rank, week_start, promoted (bool), demoted (bool)

**daily_activity** — user_id, date, declared (bool), proof_submitted (bool), proof_verified (bool), milestone_progress (bool)

**reflections** — id, user_id, mood (shipped/progress/stuck/break), note, created_at

**notification_preferences** — user_id, active_hours_start, active_hours_end, morning_declaration_time, quiet_hours_start, quiet_hours_end, declaration_reminders (bool), ship_check (bool), crew_activity (bool), review_requests (bool)

### Key Relationships
- User has many Crews (through crew_members); Crew has many Users
- Declaration belongs to User + Crew (scoped per-crew)
- Ship Check belongs to User + Crew; optionally linked to Declaration
- Proof Reviews belong to Ship Check + Reviewer (User)
- League Standings: one row per user per week (full history)
- Daily Activity: one row per user per day (pre-aggregated for heat map)

### Storage
- Proof files (screenshots): Supabase Storage with signed URLs
- All timestamps stored as UTC; displayed in user's local timezone

---

## Edge Cases

### Crew Lifecycle
- **Crew drops below 3 members:** 7-day grace period; verification switches to Level 1 (self-report); merge suggestion shown after grace period; never force-dissolve
- **Dead crew (30 days all-inactive):** Soft archive; "Your crew has gone quiet" notification with options: wake up crew, leave and find new one, or let it rest; reactivatable anytime
- **Owner leaving:** Must transfer ownership to another member first; if last member, crew is archived
- **Mid-week crew join:** New member sees feed immediately; league standing starts next Monday

### User Lifecycle
- **Solo start (no crew):** App functional with limited features; persistent but non-aggressive nudges to create/join crew
- **Account deletion:** Soft delete + anonymize; profile data wiped; username released after 30 days; activity shows as "[deleted user]" in crew feeds; reversible within 30 days
- **Returning after inactivity:** Trust score decayed (5%/week, minimum floor); streak at 0; "Returning Shipper" boost for 7 days

### Daily Loop Edge Cases
- **Missed ship check entirely:** No submission by midnight = missed day; breaks streak unless frozen
- **Proof without declaration:** Allowed; counts toward shipping but lowers consistency score
- **Concurrent reviews:** Optimistic with atomic vote counting; both reviews accepted; late reviewer sees "already approved" if majority reached
- **Rejection + resubmission:** Max 2 resubmissions per proof; each restarts the 48h review window

### Timezone & Timing
- **Day boundary:** Midnight in user's local timezone
- **Ship check window:** Random time within user's active hours (local time)
- **League reset:** Monday 00:00 UTC (global)
- **All timestamps stored UTC**, displayed in user's local timezone

---

## Monetization

### Phase 1 (Months 0-6): 100% Free
- Build community, gather feedback, iterate

### Phase 2 (Months 6-12): Freemium
- **Pro:** $9.99/month or $79.99/year (~33% discount)
- **Crew Pro:** $4.99/month per crew (larger crews, crew analytics)
- Stakes feature introduced

### Phase 3 (Months 12-24+): Platform Revenue
- Recruiter platform: $299-999/month per seat
- Challenge sponsorships: $5K-50K per challenge
- API access: tiered pricing
- Job board: $299/posting

### Free Tier (forever free)
- Core shipping proof submission
- Basic profile and public track record
- 1 crew membership
- Basic leaderboard visibility
- Weekly challenges
- Streak tracking
- Social features (reactions, comments, follows)

### Pro Tier ($9.99/mo)
- Up to 3 crews
- Advanced analytics
- Extended proof history
- Verified Shipper badge
- Custom profile domain (username.devreal.dev)

---

## UX & Platform

- **Platform:** Web app (Next.js 15)
- **Theme:** Dark mode default, light mode available
- **Navigation:** Left sidebar — Declare, Feed, Leaderboard, Profile (collapsible)

### Key Screens
1. **Landing page:** "Prove you ship." + GitHub OAuth CTA
2. **Onboarding:** Profile setup > Create/Join Crew (skippable) > Set First Goal
3. **Daily Declaration:** Single text field + tags + crew status indicators
4. **Ship Check:** Countdown timer (yellow at 2min, red at 1min) + tabbed proof submission
5. **Crew Feed:** Chronological activity with green/yellow/gray status indicators
6. **Leaderboard:** Weekly leagues + permanent tiers + category boards
7. **Profile:** Shipping heat map (52-week grid), stats, badges, verified portfolio
8. **Evening Reflection:** Quick mood tap + optional note + crew daily summary

### Design Principles
- Speed is the feature: all daily interactions < 30 seconds
- Developer-native: contribution graphs, markdown, git integration, dark mode
- Small groups, big accountability
- Celebrate shipping, not engagement
- Healthy competition (consistency > volume)
- Proof over promises
- Notification is the product: daily nudge drives core loop

---

## Tech Stack

| Layer | Choice |
|-------|--------|
| Frontend | Next.js 15 (App Router) |
| Backend/DB | Supabase (PostgreSQL + Auth + Realtime + Storage) |
| ORM | Drizzle |
| Hosting | Vercel (frontend) + Supabase Cloud (backend) |
| Auth | GitHub OAuth (primary), Google OAuth (secondary) |
| Notifications | Firebase Cloud Messaging (future) |
| Storage | Supabase Storage with signed URLs (MVP) > Cloudflare R2 (scale) |

---

## Implementation Phases

### Phase 1: MVP Core (Months 1-2)
- Auth (GitHub OAuth)
- User profiles with timezone settings
- Crew creation/joining with invite links + codes
- Daily declaration (per-crew scoped)
- Proof submission (screenshot + URL + text) with 5-min countdown
- Crew feed + reactions
- Basic streak tracking
- Notification preferences

### Phase 2: Gamification (Months 2-3)
- Weekly leagues with promotion/demotion
- Permanent tiers + composite scoring system
- Shipping heat map (daily activity summary table)
- Leaderboard page
- Celebration animations
- GitHub App integration (auto-verification)
- Evening reflection

### Phase 3: Growth (Months 3-6)
- Trust score system with 3 tiers
- Nudge system (1/person/day)
- Stall detection
- Shareable proof cards (for Twitter/social)
- Public profiles / portfolio mode
- Push notification system (FCM)
- Anti-gaming measures
- Crew lifecycle (grace periods, archiving, merging)

### Phase 4: Monetization (Months 6-12)
- Pro tier + Crew Pro
- Stakes / commitment contracts
- Deploy verification (Vercel/Netlify)
- Additional integrations (GitLab, npm, etc.)

### Phase 5: Platform (Months 12+)
- Recruiter platform
- Challenge sponsorships
- API access
- Job board
