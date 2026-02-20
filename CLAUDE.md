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

## Workflow Orchestration

### 1. Plan Mode Default
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- If something goes sideways, STOP and re-plan immediately - don't keep pushing
- Use plan mode for verification steps, not just building
- Write detailed specs upfront to reduce ambiguity

### 2. Subagent Strategy
- Use subagents liberally to keep main context window clean
- Offload research, exploration, and parallel analysis to subagents
- For complex problems, throw more compute at it via subagents
- One task per subagent for focused execution

### 3. Self-Improvement Loop
- After ANY correction from the user: update tasks/lessons.md with the pattern
- Write rules for yourself that prevent the same mistake
- Ruthlessly iterate on these lessons until mistake rate drops
- Review lessons at session start for relevant project

### 4. Verification Before Done
- Never mark a task complete without proving it works
- Diff behavior between main and your changes when relevant
- Ask yourself: "Would a staff engineer approve this?"
- Run tests, check logs, demonstrate correctness

### 5. Demand Elegance (Balanced)
- For non-trivial changes: pause and ask "is there a more elegant way?"
- If a fix feels hacky: "Knowing everything I know now, implement the elegant solution"
- Skip this for simple, obvious fixes - don't over-engineer
- Challenge your own work before presenting it

### 6. Autonomous Bug Fixing
- When given a bug report: just fix it. Don't ask for hand-holding
- Point at logs, errors, failing tests - then resolve them
- Zero context switching required from the user
- Go fix failing CI tests without being told how

## Task Management

1. **Plan First**: Write plan to tasks/todo.md with checkable items
2. **Verify Plan**: Check in before starting implementation
3. **Track Progress**: Mark items complete as you go
4. **Explain Changes**: High-level summary at each step
5. **Document Results**: Add review section to tasks/todo.md
6. **Capture Lessons**: Update tasks/lessons.md after corrections

## Core Principles

- **Simplicity First**: Make every change as simple as possible. Impact minimal code.
- **No Laziness**: Find root causes. No temporary fixes. Senior developer standards.
- **Minimal Impact**: Changes should only touch what's necessary. Avoid introducing bugs.

## Project State

- **Project name:** DevReal
- **Repo:** https://github.com/kurtjallo/DevReal.git
- **Description:** BeReal-style accountability app for developers with crew-based verification and shipping leaderboards
- **Tech stack (planned):** Next.js 15 + Supabase + Drizzle ORM + Vercel
- **Current phase:** Pre-development (research complete, requirements defined, ready for roadmap)
- **GSD config:** YOLO mode, comprehensive depth, parallel execution

### File Structure

```
.planning/                          # GSD workflow artifacts
├── config.json                     # GSD config (yolo, comprehensive, parallel)
├── PROJECT.md                      # Project definition, constraints, key decisions
├── REQUIREMENTS.md                 # v1 requirements (65 across 11 categories)
└── research/                       # Synthesized research for planning
    ├── ARCHITECTURE.md             # Database schema, API routes, real-time patterns
    ├── FEATURES.md                 # Feature landscape (table stakes, differentiators, anti-features)
    ├── PITFALLS.md                 # 23 domain pitfalls with prevention strategies
    ├── STACK.md                    # Technology stack (validated versions, cost analysis)
    └── SUMMARY.md                  # Research executive summary

docs/                               # Reference documentation
├── feature-spec.md                 # Detailed feature spec (user flows, data model, edge cases)
└── research/                       # Original deep-dive research
    ├── 01-competitive-landscape.md # WIP.co, Makerlog, ShipStreaks, Buildspace, Indie Hackers
    ├── 02-tech-stack.md            # Full stack recommendation and alternatives
    ├── 03-gamification-mechanics.md # Scoring, streaks, leagues, anti-gaming
    ├── 04-target-audience.md       # Audience segments, personas, go-to-market
    ├── 05-monetization.md          # Freemium model, pricing tiers, unit economics
    ├── 06-ux-patterns.md           # Screen designs, navigation, celebrations
    └── 07-verification-mechanisms.md # GitHub integration, peer review, trust scores
```
