# Research Summary

**Project:** DevReal
**Date:** 2026-02-20
**Dimensions:** Stack, Features, Architecture, Pitfalls (4 parallel agents)

---

## Executive Summary

DevReal's research phase is complete across all four dimensions. The stack is validated (Next.js 15.5 + Supabase + Drizzle + Vercel), the feature landscape is mapped (13 table stakes, 13 differentiators, 15 anti-features), the architecture is designed (13+ tables, RLS policies, cron scheduling, real-time patterns), and 23 domain-specific pitfalls are documented with prevention strategies.

**The single most important finding:** The competitive moat is the combination of BeReal time-pressure + crew verification + weekly leagues + composite scoring. No existing platform has all four. This combination must ship together to differentiate — shipping any subset just replicates what competitors already do.

---

## Key Findings by Dimension

### Stack (STACK.md)
- **Next.js 16 is now stable** (16.1.6) but recommend **15.5.x** for ecosystem safety; easy upgrade later
- **40+ packages verified** via npm registry with exact versions
- **Vercel free tier cron limit (2 jobs)** is tight — ship check scheduler alone needs 2; will need Pro ($20/mo) for league resets
- **Supabase Realtime 200-connection limit** hit at ~33-66 active crews; Pro ($25/mo) needed before public launch
- **Drizzle bypasses RLS** when using postgres.js directly — critical architectural decision needed

### Features (FEATURES.md)
- **13 table stakes** — the minimum viable product (auth, declarations, proofs, crews, feed, streaks, etc.)
- **13 differentiators** — the competitive moat (BeReal timer, crew verification, weekly leagues, composite scoring, trust system, etc.)
- **15 anti-features** — things to deliberately NOT build (public shaming, algorithmic feed, DMs, PM features, pay-to-win)
- **Critical path:** Auth > Crews > Declaration > Proof > Feed > Notifications > Streaks
- **Everything else can parallelize** once the critical path exists

### Architecture (ARCHITECTURE.md)
- **13+ database tables** with detailed schema, indexes, and RLS policies
- **Route groups:** `(auth)`, `(onboarding)`, `(app)` for three layout contexts
- **Real-time:** One Supabase channel per crew using Postgres Changes
- **Ship check scheduling:** Pre-compute random times at day-start, 5-min cron scans for due notifications
- **Scoring is batch, not real-time** — weekly cron calculates composite scores and league promotions
- **Build order is dictated by data dependencies:** Auth > Users > Crews > Declarations > Ship Checks > Feed

### Pitfalls (PITFALLS.md)
- **23 pitfalls:** 6 Critical, 6 High, 7 Moderate, 4 Minor
- **Most dangerous:** RLS policies silently returning empty results (develops with service key, deploys with anon key = broken app)
- **Most underestimated:** Timezone handling — every streak/league/ship-check depends on "what day is it for this user?"
- **Product-level:** Cold start death spiral — crews of 1-2 kill engagement immediately; need matchmaking-first onboarding
- **Gamification trap:** Streak anxiety + scoring perverse incentives need safety valves from day one

---

## Critical Decisions Needed

| # | Decision | Options | Research Recommendation |
|---|----------|---------|----------------------|
| 1 | Next.js 15.5 vs 16.1 | 15.5 (safe) vs 16.1 (latest) | 15.5 — verify shadcn/supabase/sentry compatibility before upgrading |
| 2 | Drizzle + RLS strategy | App-level auth vs DB-level RLS | Hybrid: Drizzle for writes (app-level auth), Supabase JS for reads (RLS enforced) |
| 3 | Onboarding flow | Create crew first vs Matchmaking | Matchmaking-first to avoid cold start death spiral (crew of 1-2) |
| 4 | Budget for Supabase Pro | Free tier vs Pro from start | Pro ($25/mo) — Realtime limits and pg_cron access are needed for core features |
| 5 | Vercel plan | Hobby vs Pro | Hobby for development, Pro ($20/mo) needed before launch for cron jobs |
| 6 | Initial scoring complexity | Full 4-factor vs Simple 2-factor | Start with 2-factor (Consistency + Completion), add Velocity + Ambition in Phase 2 |

---

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Foundation + Core Loop (Weeks 1-4)
Build the daily habit chain that everything depends on.

- **Addresses:** Auth, user profiles, crew system, daily declarations, proof submission, crew feed, basic streaks
- **Avoids:** Cold start pitfall (P7) via matchmaking onboarding; RLS pitfall (P1) via proper policy setup; timezone pitfall (P3) via IANA timezone handling from day one
- **Uses:** Next.js 15.5, Supabase Auth, Drizzle ORM, Tailwind 4 + shadcn, Sentry
- **Must include:** Database schema with indexes, RLS policies, timezone handling, file upload security, monitoring (Sentry + Vercel Analytics)

### Phase 2: Engagement Engine (Weeks 4-6)
Add mechanics that drive daily return and competition.

- **Implements:** Ship check with 5-min timer, crew verification (majority approval), basic scoring (2-factor), weekly leagues, streak freezes, celebration animations
- **Avoids:** Streak anxiety (P8) via freezes/vacation mode; scoring perverse incentives (P15) via starting simple
- **Uses:** Vercel cron jobs, FCM push notifications, Motion animations, canvas-confetti

### Phase 3: Verification + Polish (Weeks 6-8)
Deepen trust and improve retention.

- **Implements:** GitHub App integration (auto-verification), 3-tier trust system, nudge system, stall detection, shipping heat map, evening reflection
- **Avoids:** Webhook security pitfall (P4) via HMAC verification; notification fatigue (P12) via 3/day cap
- **Uses:** @octokit/rest, custom SVG heat map component

### Phase 4: Growth + Monetization (Months 3-6)
Features for scale and revenue.

- **Implements:** Public profiles, shareable proof cards, Pro tier, full 4-factor scoring, advanced crew lifecycle, anti-gaming
- **Avoids:** Feature creep — scope is explicitly bounded by anti-features list

### Phase 5: Platform (Months 6+)
- Recruiter platform, stakes/commitment contracts, additional integrations, API access

**Phase ordering rationale:**
- Phase 1 must establish auth + crews + declarations + proofs — everything else reads from this data
- Phase 2 gamification can only work once there's activity data to score and display
- Phase 3 GitHub integration is parallelizable with Phase 2 but logically follows (enhances verification)
- Phases 1-3 should ship in ~8 weeks for an MVP that has the full competitive moat

**Research flags for phases:**
- Phase 2: Ship check scheduler needs deeper research on Vercel cron limits and FCM integration
- Phase 3: GitHub App webhook security needs implementation-time verification
- Phase 4: Payment integration (Stripe) will need its own research phase

---

## Confidence Assessment

| Dimension | Confidence | Notes |
|-----------|------------|-------|
| Stack | HIGH | All 40+ versions verified via npm; architecture patterns well-established |
| Features | HIGH | Validated against 5+ competitors and gamification research |
| Architecture | HIGH | Schema, indexes, and patterns verified against official docs |
| Pitfalls | MEDIUM-HIGH | Stack-specific pitfalls from deep knowledge; some Supabase specifics may have evolved |

---

## Files

| File | Lines | Content |
|------|-------|---------|
| STACK.md | ~550 | Technology stack with versions, rationale, alternatives, cost analysis, CI/CD, project structure |
| FEATURES.md | ~800 | Table stakes, differentiators, anti-features, dependencies, complexity estimates, technical deep dives |
| ARCHITECTURE.md | ~700 | Folder structure, database schema, RLS, API routes, real-time, cron, file uploads, scoring engine |
| PITFALLS.md | ~600 | 23 pitfalls with severity, prevention, detection, phase mapping |
| SUMMARY.md | This file | Executive synthesis with roadmap implications |
