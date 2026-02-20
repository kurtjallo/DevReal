# DevReal

## What This Is

DevReal is a BeReal-style accountability app for developers. Developers join small crews (3-6 people), make daily declarations about what they're shipping, and prove it with time-boxed proof submissions that their crew verifies. It combines the urgency of BeReal's random check-ins with developer-native verification (GitHub PRs, deploy URLs, screenshots) and Duolingo-style weekly leagues to create genuine shipping accountability — not just performative build-in-public tweets.

## Core Value

Proof over promises. The gap between "announcing what you're building" and "proving you built it" is where DevReal lives. Everything else — gamification, leagues, profiles — exists to reinforce this core loop.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Core Loop**
- [ ] Morning declaration: "What are you shipping today?" — single text field, 280 char max, quick tags (Feature/Bug fix/Refactor/Design/Docs), shows crewmates who've already declared
- [ ] Random ship check: notification at random time during user's active hours, 5-minute response window, "Late" tag after window but can submit until midnight, fewer points for late submissions
- [ ] Multi-format proof submission: screenshot upload, URL with liveness check, GitHub link (auto-pull PR/commit details), text description — tabbed interface with countdown timer
- [ ] Optional evening reflection: mood/progress indicator (shipped/progress/stuck/break), crew daily summary, not required
- [ ] 1 declaration + 1 ship check per crew per day; proof without declaration is allowed (lower consistency score)

**Crews**
- [ ] Crew size: 3-6 members
- [ ] Crew creation with shareable invite link (devreal.dev/crew/abc123) and short code (ABC123)
- [ ] Crew roles: owner and member; owner must transfer ownership before leaving
- [ ] Crew feed: chronological activity (declarations, proofs, milestones) with status indicators (green/yellow/gray)
- [ ] Crew reactions: "Ship it!", "LGTM", "On it", "Call out"
- [ ] Nudge system: 1 nudge per person per day, separate nudge entity for tracking/limiting
- [ ] Stall detection: neutral visual indicator when crewmate hasn't declared

**Verification**
- [ ] Majority crew approval required within 48 hours; auto-approves on timeout
- [ ] Push notification + feed badge for review requests; one-tap approve, long-press reject with reason
- [ ] Rejection allows resubmission with better proof (max 2 resubmissions, restarts 48h window)
- [ ] Auto-verified proofs (GitHub PRs, deploys) skip crew review entirely
- [ ] 3-tier trust system: Level 1 self-reported, Level 2 peer-verified, Level 3 auto-verified
- [ ] Trust score builds over time; high trust = less review needed; decays 5%/week inactive

**Integrations (MVP)**
- [ ] GitHub App (read-only): PR merges, releases, commit activity
- [ ] URL liveness checks: HTTP check + content change detection

**Gamification**
- [ ] Weekly leagues: Bronze > Silver > Gold > Platinum > Diamond; reset Monday; top 3 promote, bottom 3 demote
- [ ] Permanent tiers: Starter (0-99) > Builder (100-499) > Shipper (500-1,499) > Architect (1,500-4,999) > Legend (5,000+)
- [ ] Composite scoring: Consistency 35%, Completion Rate 25%, Shipping Velocity 20%, Ambition Score 20%
- [ ] Streaks: miss 1 day = reset; 2 free freezes/month; 1 vacation week/quarter; crew-validated legitimate absence
- [ ] Celebration animations: checkmark (declaration), card glow (proof), confetti (7-day), full-screen (30-day), special (100-day), trophy (league promotion)
- [ ] Shipping heat map: daily activity summary table (declared/proof_submitted/proof_verified/milestone_progress)

**Profiles & UX**
- [ ] GitHub OAuth (primary auth), Google OAuth (secondary)
- [ ] Auto-pull name + avatar from GitHub; user picks username
- [ ] Onboarding: landing > GitHub OAuth > profile setup > create/join crew (skippable with nudges) > set first goal > crew feed
- [ ] Left sidebar navigation: Declare, Feed, Leaderboard, Profile (collapsible)
- [ ] Dark mode default, light mode available
- [ ] Profile page: shipping heat map, stats, badges, verified portfolio
- [ ] League history fully preserved (one row per user per week)

**Data & Privacy**
- [ ] User-set timezone; day resets at midnight local time; all timestamps stored UTC
- [ ] Per-user notification preferences: active hours, morning declaration time, quiet hours, notification type toggles
- [ ] Proof file storage: Supabase Storage with signed URLs
- [ ] Declarations scoped per-crew (separate declarations per crew for Pro users)
- [ ] Soft delete + anonymize for account deletion (30-day reversible grace period)
- [ ] Optimistic concurrency with conflict resolution for crew reviews

**Monetization**
- [ ] Months 0-6: 100% free
- [ ] Pro tier ($9.99/mo or $79.99/yr): up to 3 crews, advanced analytics, extended proof history, verified badge, custom domain
- [ ] Crew Pro ($4.99/mo per crew): larger crews, crew analytics
- [ ] Free tier forever: core proof submission, 1 crew, basic leaderboard, streaks, social features

**Edge Cases**
- [ ] Crew drops below 3: 7-day grace period, verification switches to Level 1, merge suggestion after grace
- [ ] Mid-week crew join: league standing starts next Monday
- [ ] Dead crew (30 days all-inactive): soft archive with reactivation option
- [ ] Owner leaving: must transfer ownership first; last member = archive
- [ ] Concurrent reviews: optimistic with atomic vote counting; late reviewers see "already approved"

### Out of Scope

- Native mobile app (iOS/Android) — web app first; revisit after product-market fit
- PWA / offline support — unnecessary complexity for MVP; standard web app
- Monetary stakes / commitment contracts — deferred to Phase 2+ due to payment and legal complexity
- Recruiter platform — requires 10K+ profiles; Phase 3+ (months 12+)
- GitLab / Bitbucket integration — GitHub-first for MVP; expand later
- Vercel / Netlify deploy detection — Phase 2+ integration
- npm / PyPI package publish detection — Phase 2+ integration
- Figma API integration — Phase 2+ for design work
- Challenge sponsorships — requires scale; Phase 3+
- API access / job board — platform features; Phase 3+
- Video proof uploads — screenshot + URL + GitHub + text covers MVP needs
- Anonymous usage — GitHub OAuth required from day one

## Context

- Competitive landscape is clearing: Buildspace shut down (Aug 2024), Polywork shut down (Jan 2025), Makerlog uncertain
- #buildinpublic Twitter community is the primary early adopter channel (~100-200 signups expected)
- Crew requirement (3-6 members) creates a natural viral loop — users must invite others
- Meta-strategy: build DevReal in public using DevReal
- Target: 500 users in 50-75 crews within first 2 months
- Market sizing: SAM of 3-5M developers globally; SOM of 100K-200K users by Year 3
- 7 research documents completed covering competitive landscape, tech stack, gamification, target audience, monetization, UX patterns, and verification mechanisms (in `research/`)

## Constraints

- **Tech stack**: Next.js 15 (App Router) + Supabase + Drizzle ORM + Vercel — chosen for speed-to-ship, cost efficiency ($0/month at MVP), and developer familiarity
- **Auth**: GitHub OAuth primary (developer identity), Google OAuth secondary (broad reach)
- **Storage**: Supabase Storage (MVP) with migration path to Cloudflare R2 at scale
- **Notifications**: Firebase Cloud Messaging for push notifications
- **Cost**: Infrastructure must stay under $45/month up to ~5K users
- **Privacy**: Read-only GitHub permissions only; no code storage, only metadata (hashes, line counts, timestamps)
- **Daily interaction budget**: All daily touchpoints must complete in < 30 seconds

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Web app, not PWA/native | Ship faster, single codebase, iterate quickly; native can come later if needed | — Pending |
| 5-minute ship check window | Short enough for urgency/authenticity, long enough for devs to screenshot; late submissions allowed until midnight | — Pending |
| Morning declaration + random ship check + optional evening reflection | Three touchpoints create accountability bookends; evening is optional to prevent fatigue | — Pending |
| Per-crew declarations (not global) | Multi-crew Pro users need context-specific declarations; side-project crew ≠ work crew | — Pending |
| Allow solo start with crew nudges | Reduces onboarding friction; some users want to try before committing to a group | — Pending |
| Auto-verified proofs skip crew review | Reduces review burden, rewards using integrations; crew still sees in feed | — Pending |
| Majority approval (not unanimous) | Prevents single inactive member from blocking verification; 48h auto-approve as safety net | — Pending |
| Supabase Storage with signed URLs | Industry standard S3-compatible pattern; time-limited URLs for privacy; clean migration to R2 | — Pending |
| User-set timezone with midnight local reset | Fair across global users; all data stored UTC; active hours for ship check timing | — Pending |
| Soft delete + anonymize (30-day grace) | GDPR-friendly; preserves crew history with '[deleted user]'; reversible | — Pending |
| Separate nudge table | Enforces 1/person/day limit at DB level; enables nudge analytics; cleaner than overloading reactions | — Pending |
| Daily activity summary table for heat map | Pre-aggregated for fast rendering; avoids expensive cross-table queries across 365 days | — Pending |
| Full league history preserved | Enables trends, season summaries, "Gold for 8 weeks" messaging; linear growth manageable | — Pending |
| 7-day grace period for small crews | Gives time to recruit without disrupting active members; merge suggestion after grace | — Pending |
| Mid-week join = league starts next Monday | Prevents gaming (joining winning crew late); fair to existing members | — Pending |

---
*Last updated: 2026-02-20 after initialization*
