# DevReal Tech Stack Research

## Recommended Stack (TL;DR)

| Layer | Choice | Why |
|-------|--------|-----|
| **Frontend** | Next.js 15 (App Router) | Largest ecosystem, best Vercel integration, SSR/SSG flexibility, strong PWA support |
| **Backend/BaaS** | Supabase | PostgreSQL foundation, built-in auth/storage/realtime, generous free tier, open-source |
| **Database** | PostgreSQL (via Supabase) | Relational data model fits users/crews/goals, RLS for security, real-time via WAL |
| **ORM** | Drizzle ORM | Lightweight (~7kb), fast cold starts on serverless, SQL-transparent, no codegen step |
| **Auth** | Supabase Auth | Bundled with BaaS, 50K MAU free, supports GitHub/Google/Twitter OAuth, RLS integration |
| **Real-time** | Supabase Realtime | WebSocket-based broadcast, presence, and Postgres Changes — no extra infra needed |
| **Push Notifications** | Firebase Cloud Messaging (FCM) | Free at any scale, cross-browser support, industry standard for web push |
| **File Storage** | Supabase Storage (start) / Cloudflare R2 (scale) | Supabase Storage is bundled; migrate to R2 for zero egress fees at scale |
| **Hosting** | Vercel (frontend) + Supabase (backend) | Vercel free tier for Next.js is unbeatable; Supabase hosts DB/auth/realtime |
| **Styling** | Tailwind CSS + shadcn/ui | Rapid UI development, consistent design system, copy-paste components |

**Estimated monthly cost at MVP scale: $0** (all within free tiers)
**Cost at moderate scale (~5K users): ~$25-45/month** (Supabase Pro)

---

## Detailed Analysis by Layer

### 1. Frontend Framework

#### Candidates Evaluated
- **Next.js 15** (React, Vercel)
- **Remix / React Router v7** (React, Shopify)
- **SvelteKit** (Svelte, Vercel)
- **Nuxt 3** (Vue, community)

#### Decision: Next.js 15 with App Router

**Why Next.js wins for DevReal:**
- **Ecosystem dominance**: Largest library ecosystem, most tutorials, most hiring potential
- **Vercel integration**: Deploy with `git push`, automatic preview deployments, edge functions
- **Rendering flexibility**: SSR for dynamic pages (feed, crew activity), SSG for public pages (leaderboards, profiles)
- **PWA support**: `next-pwa` or `@serwist/next` packages add service worker, offline support, and install prompts
- **Server Components**: Reduce client-side JavaScript for faster page loads
- **API Routes**: Built-in serverless functions for webhooks (GitHub, Stripe) without a separate backend

**Trade-offs acknowledged:**
- SvelteKit produces 50% smaller bundles and scores ~90/100 on Lighthouse vs Next.js's ~75/100 average
- SvelteKit has a simpler mental model with less boilerplate
- However, Next.js's ecosystem advantage and Supabase's first-class Next.js support outweigh the performance gap for an MVP

**What about Remix?**
Remix merged into React Router v7 in late 2024. While its data-loading patterns are elegant, the community is smaller and the migration introduced some ecosystem confusion. Not ideal for a solo dev wanting maximum community support.

#### PWA Implementation
```
next-pwa or @serwist/next
  - Service worker generation
  - Offline caching strategies
  - Web app manifest
  - Install prompt handling
  - Background sync for offline goal submissions
```

**iOS caveat:** iOS 16.4+ supports PWA push notifications, but only when the app is installed to the home screen. No rich media or silent push support yet. This is acceptable for DevReal since the daily nudge is a simple text notification.

---

### 2. Backend / BaaS

#### Candidates Evaluated
- **Supabase** (open-source, PostgreSQL-based)
- **Firebase** (Google, proprietary, NoSQL)
- **Custom server** (Express/Fastify + managed DB)

#### Decision: Supabase

**Why Supabase wins for DevReal:**

| Factor | Supabase | Firebase | Custom Server |
|--------|----------|----------|---------------|
| Database | PostgreSQL (relational) | Firestore (NoSQL) | PostgreSQL |
| Data model fit | Excellent — users, crews, goals are relational | Poor — denormalization required | Excellent |
| Real-time | Built-in (Broadcast, Presence, DB Changes) | Built-in (superior for simple cases) | Manual WebSocket setup |
| Auth | Built-in, 50K MAU free | Built-in, 50K MAU free | Need separate solution |
| Storage | Built-in, 1GB free | Built-in, 5GB free | Need S3/R2 setup |
| Vendor lock-in | Low — standard PostgreSQL, can self-host | High — proprietary DB format | None |
| Pricing model | Predictable, resource-based | Per-read/write (can spike unexpectedly) | Fully controlled |
| Dev speed | Very fast — instant APIs, dashboard | Very fast — mature SDKs | Slow — build everything |
| SQL queries | Full SQL, complex joins, aggregations | Limited querying, no joins | Full SQL |

**Critical advantage for DevReal:** The data model is inherently relational. Users belong to crews, goals belong to users, proofs belong to goals, reactions belong to proofs. PostgreSQL handles this naturally. Firestore would require denormalization gymnastics and data duplication.

**Supabase provides out of the box:**
- Auto-generated REST and GraphQL APIs from your database schema
- Row Level Security (RLS) for fine-grained access control at the database level
- Edge Functions (Deno-based) for custom server logic
- Database webhooks for triggering side effects
- Built-in dashboard for data management

**Scaling path:** If DevReal outgrows Supabase's managed service, the entire stack is open-source and can be self-hosted on any infrastructure. PostgreSQL expertise is widely available.

---

### 3. Database

#### Decision: PostgreSQL (via Supabase)

**Why PostgreSQL over MongoDB or Firestore:**

DevReal's core data model is deeply relational:
```
users
  ├── goals (one-to-many)
  │     ├── proofs (one-to-many)
  │     └── reactions (many-to-many)
  ├── crew_memberships (many-to-many → crews)
  └── streaks / stats (one-to-one)

crews
  ├── members (many-to-many → users)
  ├── activity_feed (one-to-many)
  └── messages (one-to-many)

leaderboards
  └── computed from goals/proofs with SQL aggregations
```

PostgreSQL advantages for this model:
- **JOINs**: Leaderboard queries need to aggregate across users, goals, and proofs — trivial with SQL, painful with NoSQL
- **Transactions**: Creating a goal + updating a streak + posting to crew feed should be atomic
- **Indexes**: Composite indexes for efficient leaderboard sorting (e.g., `(period, score DESC)`)
- **Full-text search**: Search for users, crews, goals without a separate search service
- **JSON columns**: Store flexible data (goal metadata, proof details) when needed, best of both worlds
- **Row Level Security**: Database-enforced authorization — crew messages only visible to crew members

**Supabase free tier:** 500 MB database storage, more than enough for an MVP with thousands of users.

---

### 4. Real-time

#### Decision: Supabase Realtime

Supabase Realtime is built with Elixir/Phoenix and provides three mechanisms over WebSockets:

| Feature | Use Case in DevReal |
|---------|-------------------|
| **Broadcast** | Crew chat messages, reactions, live notifications |
| **Presence** | Show who's online in a crew, typing indicators |
| **Postgres Changes** | New goal declared → crew feed updates, proof submitted → leaderboard refresh |

**Why not roll your own WebSockets?**
- Supabase Realtime handles connection management, reconnection, and scaling
- No separate WebSocket server to deploy and maintain
- Authorization is handled via Supabase's RLS policies
- Channels map naturally to crews (one channel per crew)

**Implementation pattern for crew activity feed:**
```typescript
// Subscribe to crew activity
const channel = supabase
  .channel(`crew:${crewId}`)
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'activity_feed',
    filter: `crew_id=eq.${crewId}`
  }, (payload) => {
    // Update UI with new activity
  })
  .on('presence', { event: 'sync' }, () => {
    // Update online member count
  })
  .subscribe()
```

**Scaling note:** Supabase Realtime supports up to 200 concurrent connections on the free tier and 500 on Pro. For DevReal's crew model (3-8 members), this supports 25-60+ active crews simultaneously on free tier alone.

---

### 5. Authentication

#### Decision: Supabase Auth

#### Candidates Evaluated
| Provider | Free Tier | Best For | Trade-off |
|----------|-----------|----------|-----------|
| **Supabase Auth** | 50K MAU | BaaS integration, RLS | Less polished UI components |
| **Clerk** | 10K MAU | Pre-built UI, DX | $0.02/MAU after free tier, separate service |
| **NextAuth/Auth.js** | Unlimited (self-hosted) | Full control, data ownership | More setup work, maintain yourself |
| **Firebase Auth** | 50K MAU | Google ecosystem | Vendor lock-in to Firebase |

**Why Supabase Auth for DevReal:**
- **Bundled**: No additional service to manage — auth, database, and storage share one project
- **RLS integration**: Auth user IDs flow directly into Row Level Security policies
- **Social OAuth**: GitHub, Google, and Twitter/X login supported out of the box
- **50K MAU free**: More than enough for MVP and early growth
- **Session management**: `@supabase/ssr` handles cookie-based sessions in Next.js App Router

**Social login setup** (all three required for DevReal):
- **GitHub OAuth**: Primary login for developers, access to public repo data for proof verification
- **Google OAuth**: Broad reach for non-developer users
- **Twitter/X OAuth**: Enables sharing achievements to Twitter, social proof

**If Supabase Auth becomes limiting:** Migration to Clerk is straightforward since both use JWT-based auth. Clerk offers superior pre-built UI components and multi-factor auth if needed later.

---

### 6. Push Notifications

#### Decision: Firebase Cloud Messaging (FCM)

**Why FCM over OneSignal or raw Web Push API:**

| Factor | FCM | OneSignal | Raw Web Push |
|--------|-----|-----------|-------------|
| **Cost** | Free forever, unlimited | Free up to 10K subscribers | Free (your server) |
| **Setup complexity** | Moderate | Easy (dashboard-driven) | Complex |
| **Analytics** | Built-in | Built-in, superior | Build your own |
| **Cross-browser** | Chrome, Firefox, Edge, Safari | Same + abstraction layer | Manual per-browser |
| **Segmentation** | Topics, conditions | Advanced segments, A/B testing | Manual |
| **iOS PWA support** | Yes (16.4+, when installed) | Yes | Yes |

**Why FCM wins:**
- **Free at any scale**: No per-message or per-subscriber costs, ever
- **Reliable delivery**: Google's infrastructure ensures messages reach users
- **Topic-based**: Subscribe users to crew-specific topics for targeted notifications
- **Integration with Supabase**: Trigger FCM from Supabase Edge Functions or Database Webhooks

**DevReal notification types:**
1. **Daily goal nudge** (scheduled): "Time to declare today's goal!" — triggered by a cron job (Supabase pg_cron or Vercel Cron)
2. **Crew activity** (event-driven): "Alex just shipped their feature!" — triggered by proof submission
3. **Deadline approaching** (scheduled): "4 hours left on your goal" — triggered by cron checking upcoming deadlines
4. **Streak at risk** (event-driven): "Don't break your 7-day streak!" — triggered by end-of-day check

**Note:** Using FCM does not require using Firebase for anything else. It is a standalone service with its own SDK.

---

### 7. File Storage

#### Decision: Supabase Storage (MVP) → Cloudflare R2 (scale)

**Phase 1 — MVP: Supabase Storage**
- Bundled with Supabase, no additional setup
- 1 GB free storage, 2 GB egress
- Integrated with Supabase Auth for access control
- Image transformations via built-in CDN
- Sufficient for early proof-of-ship screenshots

**Phase 2 — Scale: Cloudflare R2**
- **Zero egress fees** — massive cost savings when serving many images
- S3-compatible API — minimal code changes to migrate
- At 10 TB monthly egress: R2 costs ~$15/month vs S3's ~$891/month
- Global CDN via Cloudflare's network
- Can be integrated with Supabase (self-hosted) as a storage backend

**Image handling for proof submissions:**
```
Upload flow:
  1. Client compresses image (browser-side, < 1MB)
  2. Request signed upload URL from Supabase Storage
  3. Direct upload to storage (no server middleman)
  4. Store URL reference in goals/proofs table
  5. Serve via CDN with automatic resizing
```

---

### 8. Hosting & Deployment

#### Decision: Vercel (frontend) + Supabase Cloud (backend)

**Why this split works:**

| Component | Host | Cost (MVP) | Cost (Growth) |
|-----------|------|------------|---------------|
| Next.js app | Vercel Free | $0 | $20/mo (Pro) |
| Database + Auth + Realtime | Supabase Free | $0 | $25/mo (Pro) |
| Push notifications | FCM | $0 | $0 |
| DNS + CDN | Cloudflare Free | $0 | $0 |
| **Total** | | **$0** | **$45/mo** |

**Vercel Free Tier includes:**
- 100 GB bandwidth/month
- 100K serverless function invocations/month
- Automatic HTTPS, preview deployments, CI/CD from Git
- Edge network for global performance

**Why not Railway or Fly.io?**
- No separate backend server is needed when using Supabase as BaaS
- Vercel + Supabase covers all requirements without managing containers
- If a dedicated backend is eventually needed (heavy compute, background jobs), Railway at $5/mo is the best upgrade path

**Deployment pipeline:**
```
GitHub repo
  → Push to main → Vercel auto-deploy (production)
  → Push to branch → Vercel preview deployment
  → Supabase migrations run via CLI or GitHub Action
```

---

### 9. ORM / Query Layer

#### Decision: Drizzle ORM

#### Candidates Evaluated
| ORM | Bundle Size | Cold Start Impact | Type Safety | Migration Style |
|-----|-------------|------------------|-------------|-----------------|
| **Drizzle** | ~7 KB | Negligible | Inferred from TS schema | SQL-based, explicit |
| **Prisma 7** | ~200 KB (improved from v6) | Noticeable on serverless | Generated from .prisma schema | Declarative, auto-generated |
| **Raw SQL** | 0 KB | None | Manual typing | Manual |

**Why Drizzle for DevReal:**
- **Serverless-optimized**: 7 KB bundle = fast cold starts on Vercel Functions
- **SQL-transparent**: You see the SQL being generated, important for complex leaderboard queries
- **No codegen step**: Types update instantly as you edit the schema (unlike Prisma's `prisma generate`)
- **PostgreSQL-native**: Full support for PostgreSQL features (arrays, JSON, enums, etc.)
- **Supabase compatible**: Works directly with Supabase's PostgreSQL connection string

**Note on Prisma 7:** Prisma 7 (late 2025) removed the Rust engine and went pure TypeScript, closing the performance gap significantly. It remains a valid choice if you prefer its declarative schema and migration system. However, Drizzle's lighter footprint and SQL-first approach better suit a serverless-deployed Next.js app.

**Example Drizzle schema for DevReal:**
```typescript
import { pgTable, text, timestamp, integer, uuid, boolean } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  username: text('username').unique().notNull(),
  avatarUrl: text('avatar_url'),
  githubId: text('github_id').unique(),
  currentStreak: integer('current_streak').default(0),
  longestStreak: integer('longest_streak').default(0),
  createdAt: timestamp('created_at').defaultNow(),
})

export const crews = pgTable('crews', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  description: text('description'),
  inviteCode: text('invite_code').unique().notNull(),
  createdAt: timestamp('created_at').defaultNow(),
})

export const goals = pgTable('goals', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  crewId: uuid('crew_id').references(() => crews.id),
  title: text('title').notNull(),
  deadline: timestamp('deadline').notNull(),
  status: text('status').default('active'), // active, completed, failed
  isPublic: boolean('is_public').default(true),
  createdAt: timestamp('created_at').defaultNow(),
})
```

---

## Full Stack Summary

```
┌─────────────────────────────────────────────┐
│                   Client                     │
│  Next.js 15 (App Router) + Tailwind/shadcn  │
│  PWA: @serwist/next (service worker)        │
│  State: React Server Components + TanStack   │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│              Vercel Edge                     │
│  SSR / API Routes / Cron Jobs               │
│  Middleware (auth checks, redirects)         │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│              Supabase                        │
│  ┌─────────┐ ┌──────┐ ┌─────────┐         │
│  │PostgreSQL│ │ Auth │ │ Storage │         │
│  │ + Drizzle│ │      │ │         │         │
│  └─────────┘ └──────┘ └─────────┘         │
│  ┌──────────────┐ ┌──────────────┐         │
│  │   Realtime   │ │Edge Functions│         │
│  │  (WebSocket) │ │   (Deno)     │         │
│  └──────────────┘ └──────────────┘         │
└─────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│           External Services                  │
│  FCM (push) │ GitHub API │ Cloudflare (CDN) │
└─────────────────────────────────────────────┘
```

---

## Why This Stack Optimizes for the Requirements

### Fast MVP Development
- Supabase eliminates building auth, storage, real-time, and API layers from scratch
- Next.js + shadcn/ui provides production-ready UI components
- Drizzle's schema-as-code approach means database changes are fast and type-safe
- Vercel's git-push deployment removes all DevOps overhead

### Solo Developer Friendly
- One language throughout: TypeScript
- One deployment target: `git push` to main
- One dashboard for backend: Supabase Studio
- Minimal infrastructure to monitor and maintain
- Strong community and documentation for all choices

### Room to Scale
- PostgreSQL is battle-tested at massive scale
- Supabase is open-source — can self-host if needed
- Vercel scales automatically with traffic
- Cloudflare R2 migration path for storage cost optimization
- Architecture supports splitting into microservices later if needed

### Cost-Effective at Low Scale
- **$0/month** total at MVP with free tiers across the board
- Supabase free: 500 MB DB, 50K auth users, 1 GB storage, 200 realtime connections
- Vercel free: 100 GB bandwidth, 100K function calls
- FCM: Free forever at any scale
- First upgrade needed around ~5K active users: Supabase Pro at $25/month

---

## Alternatives Considered but Not Recommended

| Alternative | Why Not |
|-------------|---------|
| **SvelteKit** | Smaller ecosystem, fewer Supabase examples, harder to hire for |
| **Firebase** | NoSQL doesn't fit relational data model, vendor lock-in, unpredictable pricing |
| **Prisma** | Larger bundle size, codegen step, less optimal for serverless cold starts |
| **Clerk** | Excellent DX but adds another service; Supabase Auth is sufficient and bundled |
| **OneSignal** | Great features but paid plans needed for advanced segmentation; FCM is free |
| **MongoDB** | NoSQL doesn't fit the relational crew/goal/user model |
| **Custom Express server** | Unnecessary complexity when Supabase provides APIs, auth, and realtime |
| **AWS S3** | Egress fees add up; R2 is cheaper; Supabase Storage is simpler to start |

---

## Getting Started

```bash
# 1. Create Next.js app with Supabase template
npx create-next-app -e with-supabase devreal

# 2. Add PWA support
npm install @serwist/next

# 3. Add Drizzle ORM
npm install drizzle-orm postgres
npm install -D drizzle-kit

# 4. Add UI components
npx shadcn@latest init

# 5. Add push notification support
npm install firebase

# 6. Deploy
vercel deploy
```

---

## Sources

- [Next.js vs Remix vs SvelteKit 2026 Comparison (NxCode)](https://www.nxcode.io/resources/news/nextjs-vs-remix-vs-sveltekit-2025-comparison)
- [Best PWA Frameworks 2026 (WebOsmotic)](https://webosmotic.com/blog/pwa-frameworks/)
- [Supabase vs Firebase Complete Comparison (Leanware)](https://www.leanware.co/insights/supabase-vs-firebase-complete-comparison-guide)
- [Supabase vs Firebase 2026 (ClickIT)](https://www.clickittech.com/software-development/supabase-vs-firebase/)
- [Clerk vs Supabase Auth vs NextAuth Production Reality (Medium)](https://medium.com/better-dev-nextjs-react/clerk-vs-supabase-auth-vs-nextauth-js-the-production-reality-nobody-tells-you-a4b8f0993e1b)
- [Auth Providers Compared 2026 (DesignRevision)](https://designrevision.com/blog/auth-providers-compared)
- [Best Push Notification Service 2026 (EngageLab)](https://www.engagelab.com/blog/best-push-notification-service)
- [PWA Push Notifications with FCM (Pretius)](https://pretius.com/blog/pwa-push-notifications)
- [Prisma vs Drizzle 2026 (DesignRevision)](https://designrevision.com/blog/prisma-vs-drizzle)
- [Drizzle vs Prisma 2026 (MakerKit)](https://makerkit.dev/blog/tutorials/drizzle-vs-prisma)
- [Deploying Full Stack Apps 2026 (Nucamp)](https://www.nucamp.co/blog/deploying-full-stack-apps-in-2026-vercel-netlify-railway-and-cloud-options)
- [Cloudflare R2 vs AWS S3 (Cloudflare)](https://www.cloudflare.com/pg-cloudflare-r2-vs-aws-s3/)
- [Supabase Pricing 2026 (UI Bakery)](https://uibakery.io/blog/supabase-pricing)
- [Supabase Realtime Docs](https://supabase.com/realtime)
- [Supabase Pricing](https://supabase.com/pricing)
