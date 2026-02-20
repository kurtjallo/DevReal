# DevReal

**Proof over promises.** A BeReal-style accountability app for developers. Join a crew of 3-6 devs, declare what you're shipping each day, prove it with time-boxed evidence, and compete in weekly leagues.

## What Is This?

The gap between "announcing what you're building" and "proving you built it" is where DevReal lives. Every day:

1. **Declare** what you're shipping (280 chars, quick tags)
2. **Prove it** when a random ship check hits (5-minute window, BeReal-style)
3. **Your crew verifies** the proof (or GitHub auto-verifies it)
4. **Compete** in weekly leagues and climb permanent tiers

No more performative build-in-public tweets. Real proof, real accountability, real shipping.

## Core Features

### The Daily Loop
- Morning declaration: "What are you shipping today?"
- Random ship check during your active hours with a 5-minute response window
- Multi-format proof: screenshot, URL (with liveness check), GitHub link, or text
- Optional evening reflection with mood tracking

### Crews (3-6 Members)
- Small group accountability — not shouting into the void
- Shareable invite links and short codes
- Crew feed with reactions: "Ship it!", "LGTM", "On it", "Call out"
- Nudge system (1/person/day, neutral framing)

### Verification
- Crew-based peer verification (majority approval within 48h)
- GitHub App integration for auto-verified proofs (merged PRs, releases)
- 3-tier trust system: self-reported < peer-verified < auto-verified

### Gamification
- Streaks with freezes (2/month) and vacation mode (1 week/quarter)
- Composite scoring: Consistency 35%, Completion 25%, Velocity 20%, Ambition 20%
- Weekly leagues: Bronze > Silver > Gold > Platinum > Diamond
- Permanent tiers: Starter > Builder > Shipper > Architect > Legend
- Celebration animations at every milestone

### Profiles
- GitHub-style shipping heat map (52-week grid)
- Public profile with streak counter, stats, and shipping history
- Dark mode default

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 15 (App Router) |
| Database | Supabase (Postgres + Auth + Realtime + Storage) |
| ORM | Drizzle ORM |
| Styling | Tailwind CSS 4 + shadcn/ui |
| Hosting | Vercel |
| Auth | GitHub OAuth (primary) + Google OAuth (secondary) |
| Notifications | Firebase Cloud Messaging |
| Integrations | GitHub App (read-only) |

## Project Status

**Phase:** Pre-development (research and planning complete, ready to build)

### Roadmap

| # | Phase | Requirements | Status |
|---|-------|-------------|--------|
| 1 | Foundation & Design System | 2 | Not started |
| 2 | Authentication | 6 | Not started |
| 3 | Onboarding & Crews | 6 | Not started |
| 4 | Core Loop (Declarations & Proof) | 9 | Not started |
| 5 | Crew Feed & Social | 5 | Not started |
| 6 | Ship Check & Notifications | 7 | Not started |
| 7 | Peer Verification & Trust | 5 | Not started |
| 8 | Gamification Engine | 11 | Not started |
| 9 | GitHub Integration & Auto-Verify | 6 | Not started |
| 10 | Profiles, Monetization & Privacy | 8 | Not started |

**65 requirements** across 11 categories. Full details in `.planning/REQUIREMENTS.md`.

## Project Structure

```
.planning/              Roadmap, requirements, research, phase plans
docs/                   Feature spec and original research (7 deep-dives)
```

## Why DevReal?

The competitive landscape is clearing: Buildspace shut down (Aug 2024), Polywork shut down (Jan 2025), Makerlog is uncertain. No existing platform combines:

- Mandatory proof submission (not self-reported tasks)
- Structured small-group verification (not shouting into the void)
- Time-pressure mechanics (BeReal-style urgency)
- Multi-metric competitive leagues (not single-metric leaderboards)
- Auto-verification via GitHub (not trust-me-bro)

DevReal assembles the complete stack.

## License

TBD
