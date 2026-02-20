# Technology Stack

**Project:** DevReal
**Researched:** 2026-02-20
**Research method:** npm registry version checks (live), existing project research docs, training data (cutoff May 2025)

---

## Critical Update: Next.js 16 Is Stable

The project spec says "Next.js 15" but **Next.js 16 is now the `latest` stable release** (16.1.6, released Oct 2025). Next.js 15 is on a `backport` tag (15.5.12). The recommendation below reflects this discovery.

**Recommendation: Use Next.js 15 (latest 15.x backport) for launch, not Next.js 16.**

Rationale:
- Next.js 16 shipped Oct 2025 and is at 16.1.6. It requires React 19 (which is at 19.2.4 and stable).
- However, Next.js 16 is only ~4 months old. The ecosystem (shadcn/ui, middleware patterns, Supabase SSR helpers, tutorials) has more battle-tested coverage for 15.x.
- Next.js 15.5.12 is actively maintained on the `backport` tag with security and bug fixes.
- Upgrading from 15.x to 16.x later is a minor lift since both support React 19 and App Router.
- **MEDIUM confidence:** I cannot verify Next.js 16 breaking changes without web access. If the team is comfortable with a newer release, 16.x is a valid choice -- just verify shadcn/ui and @supabase/ssr compatibility first.

**Alternative: If going with Next.js 16**, pin to `16.1.6` and verify all key integrations before committing. The peer dependencies show it accepts React 18.2+ or 19.x, so there is no blocker.

---

## Recommended Stack

### Core Framework

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Next.js | 15.5.12 (backport) | Full-stack React framework | Largest ecosystem, best Vercel integration, SSR/SSG, Server Components, API routes. Backport channel is actively maintained. | HIGH |
| React | 19.2.4 | UI library | Stable, Server Components support, `use` hook, Actions. Next.js 15.5.x supports React 19. | HIGH |
| React DOM | 19.2.4 | DOM rendering | Must match React version | HIGH |
| TypeScript | 5.9.3 | Type safety | Non-negotiable for a project this size. Drizzle's type inference requires it. | HIGH |

### Backend / BaaS

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Supabase JS | 2.97.0 | BaaS client (DB, Auth, Storage, Realtime) | Single SDK for all backend needs. PostgreSQL relational model fits crews/users/goals perfectly. $0 at MVP, $25/mo Pro. | HIGH |
| @supabase/ssr | 0.8.0 | Server-side auth for Next.js App Router | Cookie-based session management for RSC/middleware. Official package. | HIGH |
| Drizzle ORM | 0.45.1 | Type-safe SQL query builder | ~7KB bundle, fast serverless cold starts, SQL-transparent (critical for leaderboard aggregation queries), no codegen step. Works directly with Supabase's PostgreSQL. | HIGH |
| drizzle-kit | 0.31.9 | Schema migrations CLI | Generates SQL migrations from TypeScript schema. `drizzle-kit push` for dev, `drizzle-kit generate` + `drizzle-kit migrate` for prod. | HIGH |
| postgres | 3.4.8 | PostgreSQL driver | postgres.js is the recommended driver for Drizzle + Supabase. Faster than `pg`, supports prepared statements, connection pooling. | HIGH |

### UI / Styling

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Tailwind CSS | 4.2.0 | Utility-first CSS | Industry standard for rapid UI development. v4 has CSS-first config, no more tailwind.config.js. | HIGH |
| shadcn/ui | latest (CLI) | Component library | Not a package -- copies components into your project. Accessible, customizable, built on Radix UI. Dark mode via CSS variables. | HIGH |
| Radix UI | (via shadcn) | Accessible primitives | Dropdown menus, dialogs, tooltips, tabs -- all used in DevReal's proof submission UI. Unstyled, composable. | HIGH |
| Lucide React | 0.575.0 | Icons | Tree-shakeable, consistent with shadcn/ui default icon set. | HIGH |
| tailwind-merge | 3.5.0 | Merge Tailwind classes | Resolves class conflicts when composing components (e.g., `cn()` utility). | HIGH |
| clsx | 2.1.1 | Conditional classnames | Lightweight classname builder, used with tailwind-merge in `cn()`. | HIGH |
| class-variance-authority | 0.7.1 | Component variants | Type-safe variant props for components (button sizes, card states, etc.). | HIGH |
| next-themes | 0.4.6 | Dark/light mode toggle | Works with App Router, supports system preference detection, SSR-safe. | HIGH |

### State Management / Data Fetching

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| TanStack Query | 5.90.21 | Server state management | Caching, background refetching, optimistic updates for crew feed. Pairs with Supabase queries. NOT needed for simple RSC data fetching -- use for client-side real-time feed state. | HIGH |
| Zustand | 5.0.11 | Client state | Lightweight (1KB), for UI state like countdown timers, proof submission drafts, sidebar collapse state. Do NOT put server data here. | HIGH |
| nuqs | 2.8.8 | URL state management | Type-safe URL search params. Useful for leaderboard filters, feed pagination, league week selection. | MEDIUM |

### Forms / Validation

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Zod | 4.3.6 | Schema validation | Runtime validation for API inputs, form data, env vars. Drizzle has `drizzle-zod` for auto-generating Zod schemas from DB tables. | HIGH |
| React Hook Form | 7.71.2 | Form state management | Performant (uncontrolled inputs), minimal re-renders. Declaration form, profile setup, proof submission all need forms. | HIGH |
| @hookform/resolvers | 5.2.2 | RHF + Zod bridge | Connects Zod schemas to React Hook Form validation. | HIGH |

### Authentication

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Supabase Auth (via supabase-js) | 2.97.0 | GitHub + Google OAuth | Built into Supabase, 50K MAU free, RLS integration. Configure GitHub and Google OAuth providers in Supabase dashboard. | HIGH |
| @octokit/rest | 22.0.1 | GitHub API client | For GitHub App integration: fetching PRs, commits, releases for auto-verification. REST API is simpler than GraphQL for read-only use. | HIGH |
| @octokit/auth-app | 8.2.0 | GitHub App auth | Authenticate as a GitHub App (installation tokens) for read-only repo access. Required for the GitHub integration feature. | HIGH |

### Push Notifications

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| firebase (client) | 12.9.0 | FCM web push client | Register service worker, get push token, handle foreground messages. Free at any scale. Only import `firebase/messaging` -- do NOT use Firebase for auth or DB. | HIGH |
| firebase-admin | (check latest) | FCM server-side sending | Send push notifications from Vercel API routes or Supabase Edge Functions. Topic-based messaging for crew notifications. | HIGH |

**FCM architecture note:** The `firebase` client SDK is large (~100KB). Only import the messaging module. The service worker (`firebase-messaging-sw.js`) runs independently from the Next.js app and handles background notifications.

### File Storage / Image Processing

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Supabase Storage (via supabase-js) | 2.97.0 | Screenshot uploads with signed URLs | Upload proof screenshots directly from client via signed upload URLs. RLS policies control access per-crew. 1GB free, 2GB bandwidth. | HIGH |
| sharp | 0.34.5 | Server-side image processing | Resize/compress screenshots before storage (save bandwidth/storage costs). Use in API route, NOT client-side. Max 1MB upload target. | HIGH |
| browser-image-compression | (check) | Client-side image compression | Compress images in the browser BEFORE upload to reduce bandwidth. Targets < 1MB. | MEDIUM |

**Upload flow:**
1. Client compresses image with browser-image-compression (< 1MB target)
2. Client requests signed upload URL from API route
3. Client uploads directly to Supabase Storage (no server middleman for the actual upload)
4. API route stores the file path reference in the database
5. Serve via Supabase Storage CDN with image transformations

### Animations / Celebrations

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Motion (framer-motion) | 12.34.3 | UI animations | Layout animations for feed items, card glow effects, page transitions, league promotion animations. The package is now `motion` (rebranded from framer-motion). | HIGH |
| canvas-confetti | 1.9.4 | Confetti effects | Lightweight (6KB), zero-dependency canvas-based confetti for streak milestones (7-day, 30-day). No React wrapper needed. | HIGH |

**Note:** `framer-motion` and `motion` are now the same package (rebranded). Use `motion` as the import. Both npm packages resolve to the same code at v12+.

### Heat Map / Data Visualization

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Custom SVG component | N/A | Shipping heat map (GitHub contribution graph style) | Build as a custom React component rendering an SVG grid. The GitHub contribution graph is a simple 52x7 grid of colored squares. No charting library needed for this. | HIGH |
| Recharts | 3.7.0 | Optional: analytics charts (Pro tier) | If you need line/bar charts for analytics dashboards later. Tree-shakeable, React-native, composable. Defer to Phase 3+. | MEDIUM |

**Heat map recommendation:** Do NOT use a charting library for the heat map. It is a 52-column x 7-row grid of `<rect>` elements in an SVG. Build a custom `<ShippingHeatMap />` component that takes the pre-aggregated `daily_activity` data. This is ~50 lines of code and renders faster than any charting library.

### Cron / Scheduled Jobs

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Vercel Cron Jobs | (built-in) | Scheduled triggers | Define cron schedules in `vercel.json`. Triggers API routes on schedule. Free tier: 2 cron jobs, Pro: 40. Use for: league reset (weekly), random ship check scheduling (daily), stale crew detection. | HIGH |
| Supabase pg_cron | (built-in) | Database-level scheduled jobs | Available on Supabase Pro ($25/mo). SQL-level cron for: trust score decay, streak resets, daily activity aggregation. | MEDIUM |

**Random ship check scheduling pattern:**
1. Vercel Cron runs daily at midnight UTC
2. API route queries all users with their active hours + timezone
3. For each user, generates a random time within their active window
4. Stores the scheduled time in a `scheduled_checks` table
5. A second Vercel Cron runs every minute (or every 5 minutes) and fires FCM notifications for checks whose scheduled time has arrived

**Cost note:** Vercel free tier allows 2 cron jobs. You need at minimum: (1) daily scheduling job, (2) per-minute notification dispatcher. This fits free tier exactly. If you need more crons (weekly league reset, daily aggregation), you will need Vercel Pro ($20/mo) or handle them via Supabase pg_cron.

### Date / Time

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| date-fns | 4.1.0 | Date manipulation + timezone | Tree-shakeable (only import what you use), timezone support via `date-fns-tz`. Lighter than dayjs for tree-shaking. | HIGH |

**Timezone handling:** All timestamps stored as UTC in PostgreSQL. Use `date-fns-tz` to convert to user's local timezone for display. User's timezone is stored in the `users` table and set during onboarding.

### Environment / Configuration

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| @t3-oss/env-nextjs | 0.13.10 | Type-safe environment variables | Validates env vars at build time with Zod schemas. Catches missing SUPABASE_URL, GITHUB_CLIENT_ID, etc. before runtime. | HIGH |

### Toasts / Notifications (In-App)

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Sonner | 2.0.7 | Toast notifications | Beautiful, accessible toasts. Integrates with shadcn/ui. "Proof approved!", "Streak milestone!", ship check countdown warnings. | HIGH |

---

## Testing Stack

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Vitest | 4.0.18 | Unit + integration tests | Fast, ESM-native, Jest-compatible API. Works with React Testing Library. Faster than Jest for Next.js projects. | HIGH |
| @testing-library/react | 16.3.2 | Component testing | Test React components by user behavior, not implementation details. Industry standard. | HIGH |
| Playwright | 1.58.2 | E2E testing | Cross-browser E2E tests. Next.js has first-class Playwright support (peer dependency in Next.js 16). Test critical flows: onboarding, declaration, proof submission, crew joining. | HIGH |
| MSW (Mock Service Worker) | 2.12.10 | API mocking | Mock Supabase API responses in tests. Intercepts at the network level, works with both server and client components. | HIGH |

**Testing strategy:**
- **Unit tests (Vitest):** Scoring calculations, timezone logic, streak logic, trust score decay, Zod schemas
- **Component tests (Vitest + RTL):** Proof submission form, countdown timer, heat map rendering, crew feed items
- **Integration tests (Vitest + MSW):** Supabase auth flow, GitHub OAuth callback, file upload flow
- **E2E tests (Playwright):** Full user journeys -- onboarding, daily declaration, ship check submission, crew verification

**Test file location:** Co-locate test files next to source files (`component.tsx` + `component.test.tsx`), except E2E tests which go in `/e2e/`.

---

## Dev Tooling

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Biome | 2.4.4 | Linting + formatting | Single tool replaces ESLint + Prettier. 10-100x faster. Opinionated defaults reduce bikeshedding. | HIGH |
| husky | 9.1.7 | Git hooks | Run linting/formatting on pre-commit. Lightweight, works with lint-staged. | HIGH |
| lint-staged | 16.2.7 | Staged file linting | Only lint/format files that are being committed. Fast pre-commit checks. | HIGH |

**Why Biome over ESLint + Prettier:**
- ESLint 10.0.1 + Prettier 3.8.1 work fine but require separate configs and are slower.
- Biome handles both linting and formatting in one tool with one config file (`biome.json`).
- Biome is written in Rust and is 10-100x faster for large codebases.
- **Tradeoff:** Biome has fewer lint rules than ESLint. For this project's needs (React, TypeScript, accessibility), Biome coverage is sufficient.
- **If you prefer ESLint:** Use ESLint 10.x flat config + `eslint-config-next` + Prettier.

---

## Monitoring / Observability

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| Sentry | 10.39.0 (@sentry/nextjs) | Error tracking + performance monitoring | Catches unhandled errors, slow API routes, failed Supabase queries. Free tier: 5K errors/mo, 10K transactions/mo. Set up from day one. | HIGH |
| Vercel Analytics | 1.6.1 (@vercel/analytics) | Web analytics | Privacy-friendly, built into Vercel. Track page views, user flows, without GDPR cookie banners. Free on Vercel hobby. | HIGH |
| Vercel Speed Insights | 1.3.1 (@vercel/speed-insights) | Core Web Vitals | Real user metrics (LCP, FID, CLS). Catch performance regressions automatically. Free on Vercel hobby. | HIGH |

**What to set up from day one:**
1. **Sentry** -- non-negotiable. Catches crashes before users report them. Configure source maps upload in build pipeline.
2. **Vercel Analytics** -- one-line setup, zero config.
3. **Vercel Speed Insights** -- one-line setup.

**What to add later (Phase 2+):**
- PostHog or Mixpanel for product analytics (feature usage, funnel analysis, retention)
- Upstash for rate limiting on API routes (@upstash/ratelimit 2.0.8)

---

## CI/CD Pipeline

### Recommended Setup (GitHub Actions + Vercel)

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npx biome check .
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npx vitest run --coverage

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test

  # Vercel handles deployment automatically via Git integration
  # No deployment step needed in CI
```

### Deployment Strategy

| Environment | Trigger | URL | Purpose |
|-------------|---------|-----|---------|
| Production | Push to `main` | devreal.dev | Live users |
| Preview | Pull request | `*.vercel.app` | Review before merge |
| Local | `npm run dev` | localhost:3000 | Development |

**Supabase environment strategy:**
- One Supabase project for production
- One Supabase project for development/preview (link to Vercel preview deployments)
- Use Supabase branching (if available on your plan) or maintain two projects
- Migrations run via `drizzle-kit migrate` in a GitHub Action before Vercel deploy

**Database migration workflow:**
1. Edit Drizzle schema in `src/db/schema/`
2. Run `npx drizzle-kit generate` to create SQL migration
3. Review the generated SQL
4. Commit migration file
5. CI runs `npx drizzle-kit migrate` against the target database
6. Vercel deploys the new code

---

## Alternatives Considered and Rejected

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Framework | Next.js 15.5 | SvelteKit | Smaller ecosystem, fewer Supabase examples, team familiarity with React |
| Framework | Next.js 15.5 | Next.js 16 | Too new (4 months old), ecosystem compatibility unverified. Upgrade path is easy later. |
| ORM | Drizzle | Prisma 7 | Larger bundle (~200KB vs 7KB), codegen step, slower cold starts on serverless |
| ORM | Drizzle | Kysely | Less ergonomic, no migration tooling built-in |
| Auth | Supabase Auth | Clerk | Adds another service ($0.02/MAU after 10K), Supabase Auth is bundled and sufficient |
| Auth | Supabase Auth | Auth.js (NextAuth) | More setup work, no RLS integration, need to manage sessions yourself |
| Push | FCM | OneSignal | Paid plans needed for advanced features; FCM is free at any scale |
| Push | FCM | Novu | More complex setup for what is fundamentally just push notifications |
| Styling | Tailwind 4 + shadcn | Chakra UI | Heavier runtime, less customizable, falling out of favor in 2025 |
| Styling | Tailwind 4 + shadcn | Material UI (MUI) | Opinionated design language, heavy bundle, not appropriate for a dev tool aesthetic |
| State | TanStack Query + Zustand | Redux Toolkit | Overkill for this app's state needs, more boilerplate |
| State | TanStack Query | SWR | TanStack Query has better devtools, mutation support, optimistic update patterns |
| Testing | Vitest | Jest | Slower, requires more config for ESM/TypeScript, Vitest is the modern standard |
| Testing | Playwright | Cypress | Playwright is faster, has better multi-browser support, is a Next.js peer dependency |
| Linting | Biome | ESLint + Prettier | Two tools to configure vs one, 10-100x slower |
| Animation | Motion | React Spring | Motion has better layout animations and a larger community |
| DB Driver | postgres.js | pg (node-postgres) | postgres.js is faster, supports connection pooling natively, recommended by Drizzle docs |
| Heat Map | Custom SVG | D3.js / Recharts | Massive overkill for a colored grid. Custom SVG is ~50 lines and renders faster. |

---

## What NOT to Use

| Technology | Why Not |
|------------|---------|
| Firebase (for DB/Auth) | NoSQL doesn't fit relational data model (crews, users, goals), unpredictable pricing, vendor lock-in |
| MongoDB | Same NoSQL mismatch. Leaderboard aggregation queries are painful without JOINs |
| tRPC | Adds complexity layer between Next.js API routes and Supabase. Server Actions + direct Drizzle queries are simpler for this project size |
| GraphQL | Over-engineering for this use case. REST + direct DB queries via Drizzle are faster to build and maintain |
| Prisma | Codegen step, larger bundle, slower cold starts. Drizzle is the right choice for serverless. |
| Redux | Overkill state management. TanStack Query handles server state, Zustand handles minimal client state |
| Styled Components / Emotion | CSS-in-JS has performance overhead and is falling out of favor in the RSC era |
| next-pwa | Deprecated/unmaintained. Use @serwist/next (9.5.6) if PWA features are needed later (out of scope for MVP) |
| Moment.js | Deprecated. Use date-fns. |
| Axios | Unnecessary when `fetch` is built into the browser and Next.js. |

---

## Full Installation Commands

```bash
# Initialize Next.js 15 with TypeScript and Tailwind
npx create-next-app@15 devreal --typescript --tailwind --eslint --app --src-dir

# Core dependencies
npm install @supabase/supabase-js @supabase/ssr drizzle-orm postgres zod

# UI
npx shadcn@latest init
npm install next-themes sonner lucide-react tailwind-merge clsx class-variance-authority

# State management + data fetching
npm install @tanstack/react-query zustand

# Forms
npm install react-hook-form @hookform/resolvers

# Dates
npm install date-fns

# GitHub integration
npm install @octokit/rest @octokit/auth-app

# Push notifications (FCM)
npm install firebase firebase-admin

# Animation + celebrations
npm install motion canvas-confetti

# Image processing (server-side)
npm install sharp

# Environment validation
npm install @t3-oss/env-nextjs

# Monitoring
npm install @sentry/nextjs @vercel/analytics @vercel/speed-insights

# Dev dependencies
npm install -D drizzle-kit typescript @types/node @types/react @types/react-dom
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
npm install -D playwright @playwright/test
npm install -D msw
npm install -D @biomejs/biome
npm install -D husky lint-staged
```

---

## Cost Analysis

### MVP (0-500 users): $0/month

| Service | Tier | Cost |
|---------|------|------|
| Vercel | Hobby | $0 |
| Supabase | Free | $0 |
| Firebase (FCM) | Free | $0 |
| Cloudflare (DNS) | Free | $0 |
| GitHub Actions | Free (2,000 min/mo) | $0 |
| Sentry | Free (5K errors) | $0 |
| **Total** | | **$0** |

### Growth (500-5,000 users): ~$45/month

| Service | Tier | Cost |
|---------|------|------|
| Vercel | Pro | $20/mo |
| Supabase | Pro | $25/mo |
| Firebase (FCM) | Free | $0 |
| Cloudflare (DNS) | Free | $0 |
| GitHub Actions | Free (3,000 min/mo on Pro) | $0 |
| Sentry | Free or Team ($26/mo) | $0-26 |
| **Total** | | **$45-71** |

### Scale trigger points:
- **Supabase Pro needed when:** >500MB database, >50K auth users, >200 realtime connections, or when you need pg_cron
- **Vercel Pro needed when:** >100GB bandwidth/mo, >100K serverless invocations, or need >2 cron jobs
- **Storage upgrade (to R2) when:** Supabase Storage egress exceeds 2GB/mo consistently

---

## Project File Structure (Recommended)

```
devreal/
  src/
    app/                     # Next.js App Router pages
      (auth)/                # Auth-related routes (login, callback)
      (dashboard)/           # Authenticated routes
        feed/
        declare/
        leaderboard/
        profile/
      api/                   # API routes
        webhooks/github/     # GitHub App webhook handler
        cron/                # Vercel cron endpoints
        upload/              # Signed URL generation
    components/              # Shared React components
      ui/                    # shadcn/ui components
      feed/                  # Feed-specific components
      proof/                 # Proof submission components
      heatmap/               # Shipping heat map
    db/
      schema/                # Drizzle schema definitions
      migrations/            # Generated SQL migrations
      queries/               # Reusable query functions
    lib/
      supabase/              # Supabase client (server + browser)
      github/                # GitHub API helpers
      scoring/               # Composite score calculations
      notifications/         # FCM helpers
      utils.ts               # cn(), formatDate, etc.
    hooks/                   # Custom React hooks
    types/                   # Shared TypeScript types
  e2e/                       # Playwright E2E tests
  public/
    firebase-messaging-sw.js # FCM service worker
  drizzle.config.ts          # Drizzle kit configuration
  biome.json                 # Biome linting/formatting config
  vercel.json                # Cron job definitions
```

---

## Version Summary (All Verified via npm Registry)

| Package | Version | Verified |
|---------|---------|----------|
| next | 15.5.12 (backport) | Yes (npm) |
| react | 19.2.4 | Yes (npm) |
| typescript | 5.9.3 | Yes (npm) |
| @supabase/supabase-js | 2.97.0 | Yes (npm) |
| @supabase/ssr | 0.8.0 | Yes (npm) |
| drizzle-orm | 0.45.1 | Yes (npm) |
| drizzle-kit | 0.31.9 | Yes (npm) |
| postgres | 3.4.8 | Yes (npm) |
| tailwindcss | 4.2.0 | Yes (npm) |
| lucide-react | 0.575.0 | Yes (npm) |
| tailwind-merge | 3.5.0 | Yes (npm) |
| clsx | 2.1.1 | Yes (npm) |
| class-variance-authority | 0.7.1 | Yes (npm) |
| next-themes | 0.4.6 | Yes (npm) |
| @tanstack/react-query | 5.90.21 | Yes (npm) |
| zustand | 5.0.11 | Yes (npm) |
| nuqs | 2.8.8 | Yes (npm) |
| zod | 4.3.6 | Yes (npm) |
| react-hook-form | 7.71.2 | Yes (npm) |
| @hookform/resolvers | 5.2.2 | Yes (npm) |
| @octokit/rest | 22.0.1 | Yes (npm) |
| @octokit/auth-app | 8.2.0 | Yes (npm) |
| firebase | 12.9.0 | Yes (npm) |
| sharp | 0.34.5 | Yes (npm) |
| motion | 12.34.3 | Yes (npm) |
| canvas-confetti | 1.9.4 | Yes (npm) |
| date-fns | 4.1.0 | Yes (npm) |
| sonner | 2.0.7 | Yes (npm) |
| recharts | 3.7.0 | Yes (npm) |
| @t3-oss/env-nextjs | 0.13.10 | Yes (npm) |
| @sentry/nextjs | 10.39.0 | Yes (npm) |
| @vercel/analytics | 1.6.1 | Yes (npm) |
| @vercel/speed-insights | 1.3.1 | Yes (npm) |
| vitest | 4.0.18 | Yes (npm) |
| @testing-library/react | 16.3.2 | Yes (npm) |
| playwright | 1.58.2 | Yes (npm) |
| msw | 2.12.10 | Yes (npm) |
| @biomejs/biome | 2.4.4 | Yes (npm) |
| husky | 9.1.7 | Yes (npm) |
| lint-staged | 16.2.7 | Yes (npm) |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Core framework (Next.js + React) | HIGH | Versions verified via npm. Recommendation to use 15.5.x is conservative but safe. |
| Backend (Supabase + Drizzle) | HIGH | Versions verified. Well-documented integration pattern. |
| UI (Tailwind + shadcn) | HIGH | Industry standard stack, versions verified. |
| Auth (Supabase Auth) | HIGH | Built-in, well-documented for GitHub + Google OAuth. |
| Push notifications (FCM) | HIGH | Architecture pattern is well-established. Firebase SDK version verified. |
| File storage (Supabase Storage) | HIGH | Signed URL pattern is standard S3-compatible. |
| Testing (Vitest + Playwright) | HIGH | Versions verified. Standard modern testing stack. |
| Monitoring (Sentry) | HIGH | Version verified. Next.js SDK is mature. |
| Cron / scheduling | MEDIUM | Vercel cron free tier limit (2 jobs) is tight. May need Pro sooner than expected. |
| Heat map | HIGH | Custom SVG is the right approach. No library needed. |
| Animations | HIGH | Motion (framer-motion) is the dominant React animation library. |
| Cost projections | MEDIUM | Based on published pricing pages. Cannot verify via web access. Supabase and Vercel pricing may have changed. |

---

## Open Questions / Flags for Phase Research

1. **Next.js 15 vs 16:** The team should make a conscious decision. If starting fresh, Next.js 16 may be fine -- but verify shadcn/ui, @supabase/ssr, and @sentry/nextjs compatibility first. This research recommends 15.5.x as the safe default.

2. **Vercel Cron limit:** 2 free cron jobs may not be enough. The random ship check scheduler needs at least 2 (daily schedule + minutely dispatch). Weekly league reset and daily aggregation would require Vercel Pro ($20/mo) or offloading to Supabase pg_cron.

3. **Supabase Realtime connection limits:** Free tier is 200 concurrent connections. With 3-6 members per crew, each with a WebSocket connection, you hit the limit at ~33-66 active crews. Plan for Supabase Pro ($25/mo, 500 connections) before public launch.

4. **FCM service worker + Next.js:** The Firebase messaging service worker (`firebase-messaging-sw.js`) must be placed in `/public/` and registered manually. It does not go through Next.js routing. This is a common integration gotcha.

5. **Tailwind CSS v4 migration:** Tailwind 4 uses CSS-first configuration (no `tailwind.config.js`). If using `create-next-app@15`, it may scaffold Tailwind v3 config. Plan to either use Tailwind v4 from the start or accept v3 scaffolding and upgrade later.

6. **Drizzle + Supabase RLS interaction:** Drizzle queries bypass RLS when using the `postgres` driver directly (since it connects as the `postgres` role). For RLS enforcement, use the Supabase JS client for data access or configure Drizzle to connect with the `anon` or `authenticated` role. This is an architectural decision with security implications.

---

## Sources

- npm registry (live queries, 2026-02-20) for all version numbers
- Existing project research: `docs/research/02-tech-stack.md` for rationale and alternatives analysis
- Training data (cutoff May 2025) for architecture patterns and library comparison -- flagged as MEDIUM confidence where applicable
