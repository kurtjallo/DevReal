# Project Guidelines

## Rules

- For every front-end change, use the front-end skill
- Never include "Co-Authored-By: Claude" in any git commits, pushes, or PRs. No Claude attribution in the repo.
- After every git commit, update this CLAUDE.md file to reflect the current state of the project (new files, components, decisions, etc.).

## Plan Mode Rules

When entering plan mode, follow this protocol:

**Before starting:** Ask if I want one of two modes:

1. **BIG CHANGE** -- Work through interactively, one section at a time (Architecture > Code Quality > Tests > Performance) with at most 4 top issues per section.
2. **SMALL CHANGE** -- Work through interactively, ONE question per review section.

**My engineering preferences (use these to guide recommendations):**

- DRY is important -- flag repetition aggressively.
- Well-tested code is non-negotiable; I'd rather have too many tests than too few.
- Code should be "engineered enough" -- not under-engineered (fragile, hacky) and not over-engineered (premature abstraction, unnecessary complexity).
- Err on the side of handling more edge cases, not fewer; thoughtfulness > speed.
- Bias toward explicit over clever.

**Review sections (run in order):**

1. **Architecture review** -- Evaluate: system design and component boundaries, dependency graph and coupling, data flow patterns and bottlenecks, scaling characteristics and single points of failure, security architecture (auth, data access, API boundaries).

2. **Code quality review** -- Evaluate: code organization and module structure, DRY violations (be aggressive), error handling patterns and missing edge cases (call out explicitly), technical debt hotspots, areas that are over/under-engineered relative to my preferences.

3. **Test review** -- Evaluate: test coverage gaps (unit, integration, e2e), test quality and assertion strength, missing edge case coverage (be thorough), untested failure modes and error paths.

4. **Performance review** -- Evaluate: N+1 queries and database access patterns, memory-usage concerns, caching opportunities, slow or high-complexity code paths.

**For each issue found:**

- Describe the problem concretely, with file and line references.
- Present 2-3 options, including "do nothing" where reasonable.
- For each option: specify implementation effort, risk, impact on other code, and maintenance burden.
- Give an opinionated recommended option and why, mapped to my preferences above.
- Explicitly ask whether I agree or want a different direction before proceeding.

**Formatting rules for AskUserQuestion:**

- NUMBER issues (1, 2, 3...) and give LETTERS for options (A, B, C...).
- Each AskUserQuestion option must clearly label the issue NUMBER and option LETTER so I don't get confused.
- The recommended option is always the 1st option.

**Workflow:**

- Do not assume my priorities on timeline or scale.
- After each review section, pause and ask for feedback before moving on.

## Project State

- **Project name:** DevReal
- **Repo:** https://github.com/kurtjallo/DevReal.git
- **Description:** BeReal-style accountability app for developers with crew-based verification and shipping leaderboards
- **Tech stack (planned):** Next.js 15 + Supabase + Drizzle ORM + Vercel
- **Current phase:** Pre-development (research complete, defining feature spec)

### Files
- `research/01-competitive-landscape.md` — Analysis of WIP.co, Makerlog, ShipStreaks, Buildspace, Indie Hackers
- `research/02-tech-stack.md` — Full stack recommendation (Next.js 15, Supabase, Drizzle, Vercel)
- `research/03-gamification-mechanics.md` — Scoring, streaks, leagues, anti-gaming, engagement loops
- `research/04-target-audience.md` — Audience segments, personas, go-to-market, market sizing
- `research/05-monetization.md` — Freemium model, pricing tiers, unit economics, revenue projections
- `research/06-ux-patterns.md` — Screen designs, navigation, interactions, celebrations, design principles
- `research/07-verification-mechanisms.md` — GitHub integration, deploy checks, peer review, trust scores
