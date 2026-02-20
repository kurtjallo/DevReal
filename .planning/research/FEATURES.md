# Feature Landscape

**Domain:** Developer accountability / social productivity (BeReal-style)
**Project:** DevReal
**Researched:** 2026-02-20
**Overall confidence:** HIGH (informed by 7 existing research documents, competitive analysis, gamification psychology research, and UX pattern analysis)

---

## Table Stakes

Features users expect from a developer accountability/social productivity app. Missing any of these means users leave or never adopt. These are validated against what WIP.co, Makerlog, ShipStreaks, and general habit/accountability apps provide.

| # | Feature | Why Expected | Complexity | Dependencies | Notes |
|---|---------|-------------|------------|--------------|-------|
| T1 | **User authentication (GitHub OAuth)** | Developers expect GitHub as identity; establishes credibility and enables integrations | S | None | GitHub primary, Google secondary. Auto-pull name + avatar. Must be fast (<10s onboarding). |
| T2 | **Daily declaration ("What are you shipping today?")** | Core loop entry point; every competitor has daily task/goal logging | S | T1 | Single text field, 280 char max, quick tags. Must complete in <15 seconds. Per-crew scoped. |
| T3 | **Proof submission (multi-format)** | ShipStreaks proved proof-based check-ins are valued; text-only is the #1 failure mode of WIP/Makerlog | M | T1, T2 | Screenshot upload, URL with liveness check, GitHub link (auto-pull PR/commit), text description. Tabbed UI. |
| T4 | **Crew system (3-6 members)** | Small groups are THE differentiator from all competitors; no existing platform has cracked this for devs | M | T1 | Create/join via shareable link + short code. Roles: owner + member. This is the core social unit. |
| T5 | **Crew feed (chronological activity)** | Users need to see what crewmates are doing; drives Hawthorne effect and social proof | M | T4, T2, T3 | Declarations, proofs, milestones. Status indicators (green/yellow/gray). Must feel like a group chat, not a dashboard. |
| T6 | **Streak tracking** | Universal in accountability apps; Duolingo shows 7-day streakers are 3.6x more likely to retain | S | T2 | Miss 1 day = reset. Must be prominently displayed. Streak counter is the #1 engagement metric. |
| T7 | **Basic notifications (ship check + reminders)** | "The notification IS the product" -- BeReal proved daily push drives the core loop | M | T1, T2 | Morning declaration reminder, random ship check (5-min window), crew activity. Max 3/day. Configurable quiet hours. |
| T8 | **User profiles with shipping history** | GitHub contribution graph set the expectation; devs want a visible track record | S | T1, T6 | Streak counter, basic stats, recent activity. Public by default. |
| T9 | **Onboarding flow (4 screens max)** | Every extra screen loses users; competitors with clunky onboarding (WIP invite-only) struggle with growth | S | T1, T4 | Landing > GitHub OAuth > profile setup > create/join crew > set first goal > crew feed. Crew step skippable with nudges. |
| T10 | **Timezone handling** | Global user base; midnight reset must be fair per-user | S | T1 | User-set timezone, all timestamps stored UTC, displayed local. Day resets at local midnight. |
| T11 | **Dark mode** | Developer audience overwhelmingly expects dark mode; Linear, GitHub, VS Code all default dark | S | None | Dark mode default, light mode available. Developer-native visual language. |
| T12 | **Crew reactions** | Low-friction social engagement; Strava's kudos proved one-tap reactions drive return visits | S | T5 | "Ship it!", "LGTM", "On it", "Call out" -- developer-native vocabulary, not generic emoji. |
| T13 | **Invite system (links + codes)** | Crew model requires inviting others; this IS the viral loop | S | T4 | Shareable link (devreal.dev/crew/abc123) + short code (ABC123). Must work via text, Slack, Discord. |

### Table Stakes Summary

These 13 features form the minimum viable product. Without them, DevReal is just another todo app. The combination of **proof-based check-ins** (T3) + **crew accountability** (T4) + **daily notification loop** (T7) is what no competitor has assembled together.

**Estimated total complexity for table stakes: 3-4 weeks of focused development.**

---

## Differentiators

Features that set DevReal apart from WIP.co, Makerlog, ShipStreaks, and general accountability apps. These are competitive advantages -- not expected, but highly valued when present.

| # | Feature | Value Proposition | Complexity | Dependencies | Notes |
|---|---------|-------------------|------------|--------------|-------|
| D1 | **Random ship check with 5-min window (BeReal mechanic)** | No competitor has time-pressure verification. Creates urgency, prevents staging, drives authentic proof. BeReal proved this mechanic drives 47.7M DAU at Duolingo-scale engagement. | M | T7, T3 | Random time during user's active hours. Timer UI (yellow at 2min, red at 1min). "Late" tag after window. Can still submit until midnight with reduced points. |
| D2 | **Crew-based peer verification (majority approval)** | No platform has structured peer verification for devs. ShipStreaks has optional verification; WIP/Makerlog have zero. This is the core moat. | M | T4, T3 | Majority crew approval within 48h. Auto-approve on timeout. One-tap approve, long-press reject with reason. Max 2 resubmissions. |
| D3 | **Weekly leagues (Duolingo-style)** | Duolingo's leagues drive 40% more engagement. Weekly resets prevent despair. No developer accountability platform has leagues. | M | T6, scoring system | Bronze > Silver > Gold > Platinum > Diamond. Reset Monday. Top 3 promote, bottom 3 demote. Show 5 above/5 below user. |
| D4 | **Composite scoring (4 metrics)** | Avoids GitHub contribution graph problem of rewarding noise. Multi-metric scoring rewards QUALITY shipping, not just volume. | M | T6, T2, T3 | Consistency 35%, Completion 25%, Velocity 20%, Ambition 20%. Prevents gaming with trivial tasks. |
| D5 | **Permanent tiers (Starter to Legend)** | Long-term progression that never resets. Creates investment and identity. Weekly leagues handle short-term competition; tiers handle long-term status. | S | D4 | Starter (0-99) > Builder (100-499) > Shipper (500-1,499) > Architect (1,500-4,999) > Legend (5,000+). |
| D6 | **GitHub App integration (auto-verification)** | Developer-native proof that skips crew review entirely. Reduces verification burden. No competitor auto-verifies from GitHub at this level. | L | T1, T3, D2 | Read-only GitHub App: PR merges, releases, commit activity. Auto-verified proofs show "Auto-verified via GitHub" badge. Webhooks primary, polling fallback. |
| D7 | **3-tier trust system** | Nuanced verification: self-reported < peer-verified < auto-verified. Rewards using integrations. Trust builds over time, decays with inactivity. | M | D2, D6 | Level 1: self-reported. Level 2: peer-verified. Level 3: auto-verified. High trust = less review needed. Decays 5%/week inactive. |
| D8 | **Shipping heat map (GitHub-style)** | Developers already think in "green squares." Visual progress over 52 weeks is the most compelling profile element. Neither WIP nor Makerlog have this done well. | M | T8, daily_activity table | 52-week grid, color-coded by activity level. Tap any day for details. Pre-aggregated daily_activity table for performance. |
| D9 | **Streak freezes + vacation mode** | Anti-burnout by design. ShipStreaks has weekend mode; Duolingo has streak freezes that reduced churn by 21%. Buildspace died partly from founder burnout culture. | S | T6 | 2 free freezes/month. 1 vacation week/quarter. Crew-validated legitimate absence. Makes streaks sustainable, not anxiety-inducing. |
| D10 | **Celebration animations** | Duolingo proved micro-celebrations drive habit formation. Asana's flying unicorn became iconic. Every shipping milestone should feel rewarding. | S | T6, D3, D5 | Hierarchy: subtle checkmark (declaration) < card glow (proof) < confetti (7-day) < full-screen (30-day) < special (100-day) < trophy (league promotion). CSS/Lottie animations. |
| D11 | **Nudge system (1/person/day)** | Gentle peer pressure without harassment. Limited to 1 nudge/person/day to prevent pile-ons. Separate nudge entity for tracking/enforcement. | S | T4, T5 | "Haven't heard from @alex today" framing. Neutral, not accusatory. Tracked in separate nudges table for analytics and rate limiting. |
| D12 | **Evening reflection (optional)** | Bookends the day without adding burden. Shows crew daily summary. Mood tracking (shipped/progress/stuck/break) provides sentiment data. | S | T2, T5 | Optional -- never required. Quick mood tap + optional note. Not tied to streaks or scoring. Pure user value. |
| D13 | **Stall detection** | Visual indicator when crewmate hasn't declared. Neutral framing. Creates ambient awareness without shaming. | S | T5, T4 | Gray status indicator. "Haven't heard from @alex today." Triggers nudge availability. No aggressive visual treatment (no red, no warning icons). |

### Differentiator Summary

The combination of **D1 (BeReal time-pressure) + D2 (crew verification) + D3 (weekly leagues) + D4 (composite scoring)** is DevReal's competitive moat. No existing platform combines time-pressured proof submission, structured peer verification, and multi-metric competitive leagues. This combination is fundamentally impossible on Twitter, Slack, or existing maker tools.

**Estimated total complexity for all differentiators: 5-7 weeks of focused development.**

---

## Anti-Features

Features to deliberately NOT build. Common mistakes in this domain. Each anti-feature is informed by a specific competitor failure or research finding.

| # | Anti-Feature | Why Avoid | What to Do Instead | Source/Lesson |
|---|-------------|-----------|-------------------|---------------|
| A1 | **Public shaming / aggressive call-outs** | Creates toxic culture. Research shows hybrid competition + support outperforms pure competition. Buildspace's culture contributed to founder burnout. | Frame inactivity as neutral data ("Haven't heard from @alex"), not judgment ("@alex is slacking"). Limit nudges to 1/day. | Buildspace shutdown, Snapchat streak anxiety research |
| A2 | **Unlimited/unstructured feed (global social network)** | WIP.co's "shouting into the void" problem. Scale kills intimacy. Indie Hackers' 100K+ members can't form accountability bonds. | Keep the crew (3-6) as the primary social unit. Global feed is secondary/optional. | WIP.co user complaints, IH community feedback |
| A3 | **Self-reported only (no proof requirement)** | WIP and Makerlog's #1 failure: "shipped feature X" with zero verification. Undermines the entire value proposition. | Require proof for ship checks. Multi-format (screenshot, URL, GitHub, text) to reduce friction. | WIP.co, Makerlog competitive analysis |
| A4 | **Single-metric leaderboard (streak length or points only)** | GitHub's contribution graph rewards noise over signal. Pure streak counting encourages trivial tasks. Single metrics are gameable. | Composite scoring: Consistency 35%, Completion 25%, Velocity 20%, Ambition 20%. Fibonacci complexity ratings. | GitHub contribution graph gaming, gamification research |
| A5 | **Monetization before community** | Makerlog has no business model. Buildspace died with $1.5M revenue at $100M valuation. WIP's $199/yr Pro is overpriced for weak value. But monetizing too early kills network effects. | 100% free for months 0-6. Build community first, then introduce Pro tier with clear value at $9.99/mo. | Buildspace, WIP.co, Makerlog monetization analysis |
| A6 | **Founder-dependent community** | Buildspace died when Farza burned out. WIP is Marc's side project. Community energy must be structural (crews), not personality-driven. | Design crews to generate their own energy. Product drives engagement, not a single personality. | Buildspace shutdown post-mortem |
| A7 | **Native mobile app (at launch)** | Doubles development surface. Web app reaches all platforms. PWA can be added later if needed. Product-market fit matters more than platform. | Web app (Next.js 15) first. Responsive design for mobile browsers. Native only after PMF. | Scope management, FEATURE_SPEC.md constraints |
| A8 | **Video proof uploads** | Higher friction than screenshots. Larger storage costs. Harder to review quickly. Screenshot + URL + GitHub + text covers 95% of proof needs. | Screenshot upload + URL with liveness check + GitHub link + text description. Four formats is sufficient. | UX research on submission friction |
| A9 | **Anonymous usage / no-auth mode** | GitHub OAuth is essential for developer identity, auto-verification, and trust. Anonymous usage undercuts accountability by design. | GitHub OAuth required from day one. Google OAuth as secondary. | Core product philosophy: "proof over promises" |
| A10 | **Algorithmic feed** | Small groups (3-6) don't need algorithmic ranking. Chronological is simpler, fairer, and transparent. BeReal uses chronological successfully. | Chronological crew feed. Small group size makes algorithmic ranking unnecessary and potentially harmful. | BeReal UX teardown |
| A11 | **Direct messaging between users** | Adds moderation burden, abuse potential, and scope creep. Crew feed + reactions cover social needs. DMs distract from the core loop. | Crew feed with reactions and nudges is the social layer. External messaging (Slack, Discord) handles deeper conversation. | Scope management |
| A12 | **Complex goal hierarchies / project management** | DevReal is accountability, not project management. Linear, Jira, and Notion already solve PM. Feature creep into PM dilutes the core value. | Simple: one declaration per day per crew. Optional milestone with deadline. No subtasks, no sprints, no boards. | Competitive positioning against PM tools |
| A13 | **Pay-to-win on leaderboards** | Premium should never give unfair competitive advantages. Leaderboards must reflect actual shipping, not spending. Destroys trust in the system. | Premium = quality of life (more crews, analytics, badges). Never bonus XP, league advantages, or scoring boosts for paying users. | Monetization anti-patterns research |
| A14 | **Guilt-trip notifications** | "You're falling behind @sarah" creates anxiety. Direct comparison notifications are toxic. Snapchat streak research shows unbounded loss aversion causes emotional distress. | Encouraging tone: "Your crew is on fire!" not "You're the only one who hasn't shipped." Celebrate group progress, don't shame individuals. | Snapchat streak psychology, notification UX research |
| A15 | **Multi-platform fragmentation** | Buildspace used Discord + Twitter + custom platform and lost cohesion. WIP uses Telegram + website. Fragmented experiences dilute engagement. | DevReal is a single, self-contained web experience. No requirement to be on Discord, Slack, or Twitter to use the product. | Buildspace post-mortem |

### Anti-Features Summary

The anti-features fall into three categories:
1. **Toxicity prevention** (A1, A14): Accountability must feel supportive, not punishing
2. **Scope discipline** (A7, A8, A11, A12, A15): Build one thing well before expanding
3. **Design philosophy** (A2, A3, A4, A6, A9, A10, A13): Structural decisions that prevent competitor failure modes

---

## Feature Dependencies

Understanding what must exist before what is critical for phasing.

```
FOUNDATION LAYER (must exist first):
  T1 (Auth) ──────────────> Everything
  T10 (Timezone) ─────────> T2, T7, D1 (all time-based features)
  T11 (Dark mode) ────────> All UI (set up design system early)

CORE LOOP (build in this order):
  T1 (Auth)
   └──> T4 (Crews) + T13 (Invites)
         └──> T2 (Declaration)
               └──> T3 (Proof submission)
                     └──> T5 (Crew feed) + T12 (Reactions)
                           └──> T7 (Notifications)

GAMIFICATION (requires core loop):
  T6 (Streaks) ───────────> D9 (Freezes/vacation)
  D4 (Composite scoring) ─> D3 (Weekly leagues)
  D4 (Composite scoring) ─> D5 (Permanent tiers)
  D3 + D5 ────────────────> D10 (Celebrations)

VERIFICATION (requires proof submission):
  T3 (Proof) ─────────────> D2 (Crew verification)
  D2 (Verification) ──────> D7 (Trust system)
  D6 (GitHub App) ─────────> D7 (Trust system, auto-verified tier)

SOCIAL POLISH (requires crew feed):
  T5 (Crew feed) ─────────> D11 (Nudges)
  T5 (Crew feed) ─────────> D13 (Stall detection)
  T5 (Crew feed) ─────────> D12 (Evening reflection)

PROFILE (requires history):
  T8 (Profile) ───────────> D8 (Shipping heat map)
```

### Critical Path

The critical path for MVP is:
```
Auth > Crews > Declaration > Proof Submission > Crew Feed > Notifications > Streaks
```
This is 7 features in sequence. Everything else can be built in parallel once this chain exists.

---

## How Key Mechanics Work (Technical Deep Dive)

### BeReal-Style Time-Pressure Loop

**How BeReal does it:**
- Single daily notification sent to ALL users simultaneously at a random time
- 2-minute capture window (front + back camera)
- "Late" label if posted after window; retake counter shows if staged
- Reciprocity gate: can't see friends' posts until you post yours

**How DevReal should adapt this:**

| Aspect | BeReal | DevReal Adaptation | Rationale |
|--------|--------|-------------------|-----------|
| Timing | Same time for all users globally | Random time within each user's configured active hours | Devs work different schedules; global sync doesn't work across timezones |
| Window | 2 minutes | 5 minutes | Code screenshots take longer than selfies; need time to switch contexts |
| Late policy | "Late" label visible to all | "Late" tag with reduced points, submit until midnight | Still allow contribution, just reward promptness |
| Reciprocity gate | Must post to see feed | Consider for MVP; could gate crew feed behind daily declaration | Powerful engagement driver but risks frustrating users who just want to browse |
| Notification | Push notification | Push notification + in-app timer when opened | Must support quiet hours and user-configured active hours |

**Technical implementation approach:**

1. **Scheduling:** Per-user cron job or scheduled function that picks a random time within the user's active hours window each day. Store next_ship_check_time in the user record or a separate schedule table. Regenerate daily at midnight local time.

2. **Notification delivery:** When ship_check_time arrives, trigger push notification via FCM (Firebase Cloud Messaging). Store the notification timestamp. Start the 5-minute window server-side.

3. **Window enforcement:** The 5-minute window is tracked server-side (ship_check.window_start). Client shows countdown timer. Submissions after window_start + 5min get is_late=true. Submissions after midnight local time are rejected.

4. **Randomness quality:** Use cryptographic randomness to prevent patterns. Vary by at least 1 hour from previous day's time. Never schedule during quiet hours. Distribution should feel genuinely random to the user.

5. **Edge cases:** What if the user has the app open when the check fires? Show in-app alert with timer. What if notification delivery fails? Allow manual check-in via the app (slightly delayed). What if user crosses timezone during the day? Use timezone at time of scheduling, not at time of notification.

**Confidence: HIGH** -- BeReal's mechanic is well-documented. The adaptation for developers (longer window, per-user timing, multi-format proof) is straightforward.

### Duolingo-Style League/Streak System

**How Duolingo does it:**

- **Leagues:** 30 users per league bracket. Bronze through Diamond (10 tiers). Weekly cycle (Monday-Sunday). Top N promote, bottom N demote. XP earned during the week determines ranking.
- **Streaks:** Consecutive days of completing at least one lesson. Streak freeze costs in-app currency (gems). Visual: flame icon with day count. iOS widget showing streak increased DAU by 60%.
- **Engagement data:** Users with 7-day streaks are 3.6x more likely to remain engaged. Streak freezes reduced churn by 21% for at-risk users. Leagues drive 40% more engagement.

**How DevReal should implement this:**

**League mechanics:**
- Pool users into brackets of ~20-30 for weekly competition
- Bracket assignment: new users start in Bronze. Subsequent weeks, users placed in their earned league.
- Score for the week = composite score (Consistency 35%, Completion 25%, Velocity 20%, Ambition 20%) calculated from that week's activity only
- Promotion: Top 3 in bracket promote to next league
- Demotion: Bottom 3 in bracket demote to previous league
- Mid-range: Stay in current league
- Mid-week joiners: League starts next Monday (prevents gaming)
- Full league history preserved: one row per user per week in league_standings table

**Streak mechanics:**
- Streak increments when user submits at least one declaration OR proof in a day
- Day boundary: midnight in user's local timezone
- Miss one day without a freeze = streak resets to 0
- 2 freezes per month (free, auto-allocated on the 1st)
- 1 vacation week per quarter (pauses all scoring and streaks)
- Crew can validate "legitimate absence" for unplanned breaks
- "Returning Shipper" boost: 7 days of bonus XP after extended absence to ease re-engagement

**Display:**
- Streak counter: prominent in navigation, profile, and crew feed
- League badge: visible on profile and in crew context
- Weekly league results: notification on Monday morning with promotion/demotion result

**Confidence: HIGH** -- Duolingo's system is extensively documented and analyzed in gamification literature.

### Peer Verification Systems (Cross-Domain Analysis)

How other domains handle peer verification, and what DevReal can learn:

| Domain | System | What Works | What Fails | DevReal Lesson |
|--------|--------|-----------|------------|----------------|
| **Code review (GitHub)** | PR approval gates: require N approvals before merge. Reviewers comment, request changes, or approve. CODEOWNERS auto-assign reviewers. | Structured flow, clear approve/reject states, conversation thread for feedback | Review fatigue at scale, rubber-stamping, slow turnaround | Use majority approval (not unanimous). Auto-approve on 48h timeout to prevent blocking. Keep review lightweight (one-tap approve). |
| **Stack Overflow** | Community voting with reputation gates. Upvote/downvote answers. Reputation unlocks privileges (edit at 2K, close-vote at 3K). | Scales to millions of users, self-moderating, quality floats up | Gaming (reputation farming), harsh culture toward newcomers, answer quality varies | Trust score that builds over time and gates capabilities (high trust = less review needed). |
| **Wikipedia** | Tiered trust: anonymous < registered < autoconfirmed < admin. Edit history determines trust level. Revert capability based on trust. | Massive scale peer review. Trust earned through demonstrated good behavior. | Edit wars, bureaucracy, high barrier for new editors | 3-tier trust model (self-reported < peer-verified < auto-verified) matches this pattern well. Decay prevents trust from becoming stale. |
| **eBay/Amazon reviews** | Buyer-seller feedback with verified purchase badges. Aggregate ratings. Seller can respond to reviews. | Verified purchase adds credibility. Response mechanism prevents one-sided narratives. | Fake reviews, review bombing, pay-for-review schemes | Resubmission mechanic (rejected proof can be resubmitted with better evidence) mirrors seller response. Auto-verification badge = verified purchase badge. |
| **Strava segments** | Automatic leaderboard creation from GPS data. Flag suspicious activities. Community can flag impossible times. | Automated verification (GPS) + community policing. No manual review needed for most activities. | GPS spoofing, Strava "snipers" (drivers on cycling segments) | GitHub auto-verification = Strava GPS verification. Crew review = community flagging. Hybrid is strongest. |
| **Academic peer review** | Double-blind review. 2-3 reviewers per paper. Accept/revise/reject decisions. Resubmission allowed after revision. | Structured evaluation criteria. Multiple independent reviewers reduce bias. | Slow (months), reviewer fatigue, inconsistent standards | Majority approval from crew. Rejection with reason + resubmission mirrors academic revise-and-resubmit. |

**Key insight:** The most successful peer verification systems combine automated signals (verified purchase, GPS, commit data) with human judgment (reviews, votes, flags). DevReal's 3-tier trust system (self-reported < peer-verified < auto-verified) directly mirrors this proven pattern.

**Confidence: HIGH** -- Peer verification patterns are well-established across many domains.

### Social Features That Drive Retention in Small-Group Apps

Research from Habitica (party system), Squad (joinsquad.co), Focusmate, Strava clubs, and fitness accountability apps identifies these retention drivers:

**1. Shared stakes / mutual dependency (HIGHEST IMPACT)**
- Habitica's "boss battles" where one member's failure damages the whole group
- Research shows 20-40% greater adherence when peer accountability is present
- DevReal equivalent: crew's collective activity affects crew ranking. One stalling member drags down the crew score.

**2. Visible reciprocity (HIGH IMPACT)**
- BeReal's reciprocity gate (post to see friends' posts)
- Strava's kudos create obligation to reciprocate
- DevReal equivalent: seeing who declared creates social pressure to declare too ("4/6 declared"). Reactions create reciprocal engagement loops.

**3. Small group identity (HIGH IMPACT)**
- Buildspace's "houses" created tribal identity
- Strava clubs have names, logos, shared challenges
- DevReal equivalent: crew names, shared streaks, crew-level badges ("High Ambition crew"), collective celebrations when all members ship.

**4. Low-friction social touch (MEDIUM IMPACT)**
- Strava kudos (one tap)
- Duolingo friend quests
- DevReal equivalent: one-tap reactions ("Ship it!", "LGTM"), nudge button (1/day limit), crew feed as ambient awareness.

**5. Celebration of group milestones (MEDIUM IMPACT)**
- Group celebration when everyone in crew ships on the same day
- Weekly crew summary ("Your crew shipped 14 times this week")
- Monthly crew awards ("Crew MVP", "Most Improved")

**6. Re-engagement mechanics (MEDIUM IMPACT)**
- "Returning Shipper" boost after absence
- Crew sends "welcome back" when member returns
- Streak freeze prevents catastrophic loss, keeping the door open for return

**Confidence: HIGH** -- Small group dynamics are well-studied in psychology and validated by multiple successful products.

### Notification Patterns: What Works vs. What Annoys

**Patterns that WORK (drive engagement without fatigue):**

| Pattern | Example | Why It Works |
|---------|---------|-------------|
| Time-sensitive action | "Ship Check! 5 minutes to submit proof." | Clear urgency, specific action, natural deadline |
| Social proof trigger | "3 of your crewmates already declared today." | FOMO + social norm, shows you're behind peers |
| Milestone proximity | "One more ship and you hit a 30-day streak!" | Loss aversion near a goal; near-miss psychology |
| Earned celebration | "You hit Gold league! Trophy animation." | Positive reinforcement at moment of achievement |
| Peer action | "@sarah just shipped! Your crew is on fire." | Social proof, creates awareness without pressure |
| At-risk warning | "Your 15-day streak ends at midnight." | Loss aversion is 2x more motivating than gain-seeking |

**Patterns that ANNOY (cause notification fatigue and app deletion):**

| Pattern | Example | Why It Fails |
|---------|---------|-------------|
| Guilt-tripping | "You're the only one who hasn't shipped today." | Feels shaming, not motivating. Creates negative association with the app. |
| Direct comparison | "You're falling behind @sarah." | Competitive anxiety, especially for less active users. |
| Too frequent | More than 3 notifications/day | Notification fatigue. Users mute or delete the app. |
| Generic/irrelevant | "Check out what's new on DevReal!" | No specific action or value. Feels like spam. |
| Non-actionable | "Your crew activity was low this week." | What am I supposed to do about it? No clear next step. |
| Off-hours | Notifications during quiet hours or sleep | Disrespectful of user boundaries. Instant mute/uninstall. |

**DevReal notification budget (max 3/day):**
1. Morning declaration reminder (configurable time)
2. Random ship check (within active hours)
3. One contextual notification (crew activity, streak warning, or celebration -- whichever is most relevant)

**Smart notification rules:**
- Never send more than 3 notifications per day
- Respect user-configured quiet hours absolutely
- Tone is always encouraging, never guilt-tripping
- Every notification must have a clear action the user can take
- Celebration notifications should feel rewarding, not transactional
- If user hasn't opened app by noon, send ONE gentle reminder (not repeated reminders)

**Confidence: HIGH** -- Notification psychology is well-studied. The 3-notification-max rule is validated by multiple consumer app studies.

---

## MVP Recommendation

For MVP, prioritize in this order:

### Phase 1: Core Loop (Weeks 1-3)
Build the daily habit chain. Without this, nothing else matters.

1. **T1** - Auth (GitHub OAuth) -- S
2. **T10** - Timezone handling -- S
3. **T11** - Dark mode / design system -- S
4. **T4** - Crew system -- M
5. **T13** - Invite system -- S
6. **T9** - Onboarding flow -- S
7. **T2** - Daily declaration -- S
8. **T3** - Proof submission (screenshot + URL + text; GitHub auto-pull deferred) -- M
9. **T5** - Crew feed -- M
10. **T12** - Reactions -- S
11. **T6** - Basic streak tracking -- S
12. **T8** - Basic profile -- S

### Phase 2: Engagement Engine (Weeks 3-5)
Add the mechanics that drive daily return and competition.

1. **D1** - Random ship check with 5-min window -- M
2. **T7** - Notifications (declaration reminder + ship check + crew activity) -- M
3. **D2** - Crew verification (majority approval, 48h timeout) -- M
4. **D4** - Composite scoring system -- M
5. **D3** - Weekly leagues -- M
6. **D5** - Permanent tiers -- S
7. **D9** - Streak freezes + vacation mode -- S

### Phase 3: Polish and Growth (Weeks 5-7)
Features that improve retention and reduce friction.

1. **D6** - GitHub App integration (auto-verification) -- L
2. **D7** - Trust system (3 tiers) -- M
3. **D8** - Shipping heat map -- M
4. **D10** - Celebration animations -- S
5. **D11** - Nudge system -- S
6. **D12** - Evening reflection -- S
7. **D13** - Stall detection -- S

### Defer to Post-MVP

These features have value but should not distract from the core loop:

- **Monetary stakes / commitment contracts** -- Legal complexity, payment integration; Phase 2+ (months 6+)
- **Shareable proof cards for social media** -- Nice for growth but not needed for core loop
- **Public profiles / portfolio mode** -- Needs enough history to be useful; launch after 1-2 months of data
- **Advanced analytics dashboard** -- Pro tier feature for monetization phase
- **Push notifications via FCM** -- Can launch with in-app notifications first; push requires FCM setup and PWA service worker
- **Crew lifecycle management** -- Grace periods, archiving, merge suggestions; only needed after crews start dying (months 2+)
- **Anti-gaming measures** -- Pattern detection, random audit checks; only needed if gaming becomes a real problem
- **Recruiter platform / job board** -- Requires 10K+ profiles; Phase 3+ (months 12+)
- **Additional integrations** -- GitLab, Bitbucket, Vercel, Netlify, npm, Figma; expand based on user demand

---

## Complexity Estimates Reference

| Size | Definition | Approximate Time |
|------|-----------|-----------------|
| S | Single component, straightforward logic, no external dependencies | 1-2 days |
| M | Multiple components, some integration work, moderate business logic | 3-5 days |
| L | Complex integration, external APIs, significant business logic, edge cases | 1-2 weeks |
| XL | Major subsystem, multiple external integrations, complex state management | 2-4 weeks |

---

## Validation Against Competitor Landscape

| DevReal Feature | WIP.co | Makerlog | ShipStreaks | Buildspace | Indie Hackers |
|----------------|--------|----------|------------|------------|---------------|
| Daily declaration | Yes (Telegram) | Yes (web) | Yes (web) | Weekly | No |
| Proof-of-ship | No | No | Yes (optional) | Weekly update | No |
| Structured peer verification | No | No | Loose/optional | No | No |
| Small group (3-6) accountability | No | No | No | Houses (large) | Weak groups |
| Time-pressure mechanic (BeReal) | No | No | No | No | No |
| Weekly leagues | No | No | No | Points/leaderboard | No |
| Composite scoring (multi-metric) | No | Basic | No | Single metric | No |
| Auto-verification (GitHub) | No | GitHub integration (task logging) | No | No | No |
| Streak freezes / anti-burnout | Purchasable | No | Weekend mode | No | N/A |
| Trust system (tiered) | No | No | No | No | No |

**Gap DevReal fills:** No existing platform combines ALL of: mandatory proof + structured small-group verification + time-pressure mechanics + multi-metric leagues + auto-verification. Each competitor has at most 1-2 of these. DevReal is the first to assemble the complete stack.

---

## Sources

All findings in this document are derived from:

- `/Users/kurt/Documents/git-repos/BeShip/research/01-competitive-landscape.md` -- Competitor analysis of WIP.co, Makerlog, ShipStreaks, Buildspace, Indie Hackers, and 7 adjacent platforms
- `/Users/kurt/Documents/git-repos/BeShip/research/03-gamification-mechanics.md` -- Duolingo, BeReal, Strava, GitHub, Snapchat gamification research; psychology of accountability; scoring system design; anti-gaming mechanisms
- `/Users/kurt/Documents/git-repos/BeShip/research/04-target-audience.md` -- Build-in-public movement, indie hacker community, developer demographics, personas, cold start strategy
- `/Users/kurt/Documents/git-repos/BeShip/research/05-monetization.md` -- Revenue models (Discord, Strava, Duolingo, GitHub, LinkedIn), pricing research, unit economics
- `/Users/kurt/Documents/git-repos/BeShip/research/06-ux-patterns.md` -- BeReal UX teardown, social app patterns (Strava, Duolingo, Linear, GitHub, Notion), notification strategy, celebration animations
- `/Users/kurt/Documents/git-repos/BeShip/research/07-verification-mechanisms.md` -- GitHub/GitLab/Bitbucket APIs, deploy verification, peer review systems, trust scores, anti-gaming measures
- `/Users/kurt/Documents/git-repos/BeShip/FEATURE_SPEC.md` -- Existing feature specification and data model
- `/Users/kurt/Documents/git-repos/BeShip/.planning/PROJECT.md` -- Project definition, requirements, constraints, key decisions

**Confidence assessment:**
- Table stakes: HIGH -- validated against 5+ competitors and general accountability app patterns
- Differentiators: HIGH -- each is grounded in psychology research and competitor gap analysis
- Anti-features: HIGH -- each is tied to a specific competitor failure mode
- Technical mechanics: HIGH for BeReal/Duolingo (well-documented), MEDIUM for notification optimization (requires A/B testing in practice)
- Complexity estimates: MEDIUM -- estimates are educated guesses without having built the actual codebase; actual complexity depends on implementation details
