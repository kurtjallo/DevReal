# DevReal Feature Spec

> BeReal-style accountability app for developers with crew-based verification and shipping leaderboards.

---

## Core Loop

**Morning Declaration (daily, user-configured time)**
- "What are you shipping today?" — single text field, 280 char max
- Quick tags: Feature, Bug fix, Refactor, Design, Docs
- Shows which crewmates have already declared (social proof)

**Random Ship Check (once daily, random time)**
- Notification at a random time during user's active hours
- **5-minute response window** — posts after the window get a "Late" tag
- Multi-format proof submission:
  - Screenshot/photo upload
  - URL (auto-previewed with liveness check)
  - GitHub link (auto-pulls PR/commit details)
  - Text description
- Crew reviews and verifies the proof

---

## Crews

- **Size:** 3-6 members
- **Verification:** Majority approval required within 48 hours; auto-approves on timeout
- **Membership:** 1 free crew per user; Pro unlocks up to 3 crews
- **Crew feed:** Chronological activity feed showing declarations, proofs, milestones
- **Crew reactions:** "Ship it!", "LGTM", "On it", "Call out" (developer-native)
- **Stall detection:** Visual indicator when a crewmate hasn't declared (neutral, not aggressive)
- **Nudge:** Crewmates can nudge once per person per day

---

## Gamification & Progression

### Weekly Leagues (Duolingo-style)
- Bronze > Silver > Gold > Platinum > Diamond
- Reset every Monday
- Top 3 promote, bottom 3 demote

### Permanent Tiers
- Starter (0-99) > Builder (100-499) > Shipper (500-1,499) > Architect (1,500-4,999) > Legend (5,000+)
- Never reset

### Scoring (composite)
- Consistency (35%): active days / expected days x streak multiplier
- Completion Rate (25%): completed goals / declared goals
- Shipping Velocity (20%): complexity-weighted completions over time
- Ambition Score (20%): average goal complexity x frequency

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
1. **Onboarding:** Welcome > GitHub OAuth > Create/Join Crew > Set First Goal (4 screens max)
2. **Daily Declaration:** Single text field + tags + crew status
3. **Ship Check:** Multi-format proof submission with 5-min countdown
4. **Crew Feed:** Chronological activity with status indicators (green/yellow/gray)
5. **Leaderboard:** Weekly leagues + permanent tiers + category boards
6. **Profile:** Shipping heat map, stats, badges, verified portfolio

### Design Principles
- Speed is the feature: all daily interactions < 30 seconds
- Developer-native: contribution graphs, markdown, git integration, dark mode
- Small groups, big accountability
- Celebrate shipping, not engagement
- Healthy competition (consistency > volume)
- Proof over promises

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
| Storage | Supabase Storage (MVP) > Cloudflare R2 (scale) |

---

## Implementation Phases

### Phase 1: MVP Core (Months 1-2)
- Auth (GitHub OAuth)
- User profiles
- Crew creation/joining
- Daily declaration
- Proof submission (screenshot + URL + text)
- Crew feed + reactions
- Basic streak tracking

### Phase 2: Gamification (Months 2-3)
- Weekly leagues
- Permanent tiers + scoring system
- Shipping heat map
- Leaderboard
- Celebration animations
- GitHub App integration (auto-verification)

### Phase 3: Growth (Months 3-6)
- Trust score system
- Advanced crew features (nudge, stall detection)
- Shareable proof cards (for Twitter/social)
- Public profiles / portfolio mode
- Notification system
- Anti-gaming measures

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
