# Domain Pitfalls

**Project:** DevReal -- BeReal-style dev accountability app
**Stack:** Next.js 15 (App Router) + Supabase + Drizzle ORM + Vercel
**Researched:** 2026-02-20
**Confidence basis:** Training knowledge (as of mid-2025) applied to the specific project context. WebSearch was unavailable, so findings rely on deep experience with this stack and domain. Confidence levels reflect this constraint -- items flagged LOW should be validated against current documentation before implementation.

---

## Critical Pitfalls

Mistakes that cause rewrites, data loss, or app-breaking failures.

---

### Pitfall 1: Supabase RLS Policies That Silently Return Empty Results Instead of Errors

**Severity:** CRITICAL
**Affects phase:** Phase 1 (MVP Core) -- Auth + Data Layer
**Confidence:** HIGH (well-documented PostgreSQL behavior)

**What goes wrong:** Row Level Security in PostgreSQL does not throw errors when a user queries rows they cannot access. It silently returns zero rows. This means a misconfigured RLS policy produces bugs that look like "no data" rather than "permission denied." Developers build features, test with the service role key (which bypasses RLS), ship to production, and discover users see empty feeds, empty crew lists, or empty leaderboards.

**Why it happens:**
- Development often uses the Supabase `service_role` key, which bypasses RLS entirely
- RLS policies are additive (permissive by default) -- if no policy matches, zero rows are returned
- Policies reference `auth.uid()` which is null in server-side contexts unless the session is explicitly passed
- Complex policies with JOINs (e.g., "user is a member of this crew") are easy to get subtly wrong

**Specific DevReal risks:**
- Crew feed query: `SELECT * FROM declarations WHERE crew_id IN (SELECT crew_id FROM crew_members WHERE user_id = auth.uid())` -- if the JOIN fails silently, the feed is blank
- Proof reviews: a reviewer must be in the same crew as the proof submitter -- getting this policy wrong means reviews silently fail
- League standings: querying across all users in a league requires careful "public read" policies

**Warning signs:**
- Features work in development but show empty data in production
- Features work for one user role but not another
- Data appears in Supabase dashboard but not in the app

**Prevention:**
1. **Never develop with the service role key for data queries.** Use `anon` key with proper user session from day one
2. **Write RLS policies before writing application code.** Define who can read/write each table first
3. **Create a test matrix:** For every table, test as (a) anonymous user, (b) authenticated user not in the crew, (c) crew member, (d) crew owner. Assert expected results AND assert empty results where appropriate
4. **Use Supabase's RLS debugging:** `SET LOCAL role = 'authenticated'; SET LOCAL request.jwt.claims = '{"sub":"user-id"}';` in SQL editor to test policies as specific users
5. **Enable RLS on ALL tables, including lookup tables.** Forgetting to enable RLS on a table means it is fully public by default
6. **Log when queries return zero rows unexpectedly** -- add application-level assertions for "this query should never return empty for an authenticated crew member"

**Detection:** Automated integration tests that run queries as different user roles and assert row counts. If your crew feed test returns 0 rows for a crew member, that is a test failure, not an empty state.

---

### Pitfall 2: Supabase Realtime + RLS Performance Cliff

**Severity:** CRITICAL
**Affects phase:** Phase 1 (MVP Core) -- Crew Feed
**Confidence:** MEDIUM (based on Supabase architecture knowledge; verify current Realtime docs)

**What goes wrong:** Supabase Realtime's `postgres_changes` feature evaluates RLS policies for every subscribed client on every database change. With 50 crews of 6 members each (300 concurrent subscribers), every `INSERT` into `declarations` or `ship_checks` triggers 300 RLS policy evaluations. If those policies involve JOINs (e.g., checking crew membership), this creates a multiplicative query load on the database.

**Why it happens:**
- Realtime broadcasts database changes to all subscribers on a channel
- Each subscriber's RLS policy is evaluated separately to determine if they should see the change
- Complex RLS policies (multi-table JOINs) multiply the per-event cost
- The free tier has 200 concurrent connections; Pro has 500 -- these are hard limits

**Specific DevReal risks:**
- Crew feed uses `postgres_changes` filtered by `crew_id` -- every declaration triggers RLS evaluation for all crew members
- If 10 crews are active simultaneously and each has a complex membership-check RLS policy, a single declaration INSERT could trigger 60 RLS evaluations
- At scale (500+ concurrent connections on Pro), this becomes a significant fraction of database capacity

**Warning signs:**
- Supabase dashboard shows high replication slot lag
- Realtime events arrive with increasing delay (seconds, then tens of seconds)
- Database CPU spikes correlate with realtime subscription count, not query count
- `pg_stat_activity` shows many `supabase_realtime` connections

**Prevention:**
1. **Use Broadcast channels instead of postgres_changes for crew feeds.** When a declaration is inserted, have the API route explicitly broadcast to the crew channel. This skips RLS evaluation entirely for realtime delivery
2. **Keep RLS policies on the underlying tables simple.** The simpler the policy, the cheaper the per-subscriber evaluation
3. **Use the `filter` parameter on postgres_changes subscriptions** to narrow the scope: `.on('postgres_changes', { filter: 'crew_id=eq.${crewId}' })` -- this reduces evaluation to subscribers on that specific crew
4. **Monitor Supabase Realtime metrics** from launch day: connection count, message throughput, replication lag
5. **Set a connection budget:** With 200 free-tier connections, allocate carefully. A user with 3 crews open has 3 connections. Budget for maximum concurrent users = connections / average_channels_per_user
6. **Consider hybrid approach:** Use Broadcast for high-frequency events (reactions, typing), postgres_changes only for critical state changes (proof_reviews verdict)

**Detection:** Monitor Supabase dashboard's Realtime metrics. If replication lag exceeds 5 seconds or database CPU exceeds 80% during normal usage, the RLS-on-Realtime pattern is likely the cause.

---

### Pitfall 3: Timezone Bugs That Corrupt Streaks, Leagues, and Ship Check Windows

**Severity:** CRITICAL
**Affects phase:** Phase 1 (MVP Core) -- Streak Tracking, Ship Check Timing
**Confidence:** HIGH (timezone bugs are the most common source of production incidents in global apps)

**What goes wrong:** DevReal's core mechanics depend on "what day is it for this user?" Streaks reset at midnight local time. Ship check windows are within the user's active hours. League resets happen Monday 00:00 UTC. Getting any of these wrong means users lose streaks unfairly, receive ship checks at 3 AM, or get placed in the wrong league week.

**Why it happens:**
- JavaScript `Date` objects use the server's timezone, not the user's
- Storing dates as `DATE` type in PostgreSQL strips timezone information
- "Midnight" means different things for different users -- midnight in Tokyo is 3 PM UTC
- Daylight Saving Time shifts break fixed UTC offset calculations twice per year
- Vercel serverless functions run in a single region (default: us-east-1) -- their local time is irrelevant to user timezones
- The `daily_activity` table uses `date` as a key -- but "today" depends on who is asking

**Specific DevReal risks:**
- **Streak corruption:** User in UTC+12 (New Zealand) submits at 11:30 PM local time (11:30 AM UTC). If the server counts this as "today" in UTC, but the user's "today" ends in 30 minutes, the daily_activity row might be attributed to the wrong date, either granting a double-day or missing a day
- **Ship check timing:** A user sets active hours 9 AM - 6 PM in UTC+9 (Tokyo). The cron job must fire the notification at a random time between 00:00 UTC and 09:00 UTC. If the cron runs hourly, the granularity is too coarse. If it runs per-user, the cost scales linearly
- **League reset edge case:** League resets Monday 00:00 UTC. A user in UTC-12 (Baker Island) could submit Sunday 11 PM local time, which is Monday 11 AM UTC -- after the reset. Their submission counts toward the NEW week, not the week they intended
- **DST transitions:** A user in US Eastern time switches from UTC-5 to UTC-4 in March. Their "midnight" shifts by an hour. If the system cached the offset, the ship check window and streak boundary are wrong for hours or days

**Warning signs:**
- Users in Australia/NZ/Japan report lost streaks
- Ship check notifications arrive outside active hours for users far from UTC
- League scores look wrong on Monday mornings for users near the date line
- Bug reports cluster around DST transition dates (March and November in the US)

**Prevention:**
1. **Store the user's IANA timezone string (e.g., `America/New_York`), never a UTC offset.** Offsets change with DST; IANA timezones handle this automatically
2. **All streak/day calculations must happen in the user's timezone.** Use a library like `date-fns-tz` or `luxon` to convert UTC timestamps to the user's timezone before determining "which day"
3. **The `daily_activity` table should use `(user_id, date)` where `date` is the date IN THE USER'S TIMEZONE**, not the UTC date. Calculate this at write time: `INSERT INTO daily_activity (user_id, date) VALUES ($1, (NOW() AT TIME ZONE $2)::date)`
4. **Ship check scheduling:** Store the next ship check time as a UTC timestamp computed from the user's active hours in their timezone. Recompute whenever the user changes timezone or active hours. Use a cron job that checks "which users have a ship check time <= NOW()" every minute, not per-user cron jobs
5. **League reset:** Clearly document that leagues reset Monday 00:00 UTC globally. This is a design decision, not a bug -- but communicate it to users so they understand late-Sunday-in-UTC submissions count toward the new week
6. **DST protection:** Never cache timezone offsets. Always compute the offset at the moment of use from the IANA timezone string. Test with users in timezones that observe DST and those that do not
7. **Add a "timezone sanity check" on login:** If the browser's detected timezone differs from the stored timezone, prompt the user to update. Travelers and people who move create stale timezone data

**Detection:** Automated tests with mocked timezones covering: UTC+12, UTC-12, UTC+0, UTC+5:30 (India, half-hour offset), UTC+5:45 (Nepal, 45-min offset), and DST transitions for US Eastern and Australian Eastern. Any streak/league test that does not specify a timezone is a bug in the test itself.

---

### Pitfall 4: Majority Vote Race Condition Corrupts Verification Status

**Severity:** CRITICAL
**Affects phase:** Phase 1 (MVP Core) -- Crew Verification
**Confidence:** HIGH (classic concurrent write problem)

**What goes wrong:** Two crew members review a proof simultaneously. Both read that 1 approval exists (out of 3 needed for majority in a crew of 5). Both insert their approval. A trigger or application code checks "is count >= majority?" and both see count = 2 (their own + the previous one). Neither triggers the "approved" status change because neither sees count = 3. Or worse: both trigger the status change and downstream effects (scoring, notifications) fire twice.

**Why it happens:**
- The spec says "Majority crew approval required within 48 hours"
- "Optimistic concurrency with conflict resolution" is mentioned but not specified
- Default PostgreSQL transaction isolation (`READ COMMITTED`) allows both transactions to see the same pre-existing count
- Application-level "read count, then decide" is inherently racy without serialization

**Specific DevReal risks:**
- Proof gets stuck in "pending" forever despite having enough votes (votes were counted before the last vote was committed)
- Proof gets approved twice, sending duplicate notifications and double-counting points
- In a 3-person crew (majority = 2), a single concurrent review window is enough to hit this

**Warning signs:**
- Proofs with enough reviews stay in "pending" status
- Users report "I approved but nothing happened"
- Duplicate "proof approved" notifications
- Score calculations show doubled verification points

**Prevention:**
1. **Use a database-level approach, not application-level counting.** Create a PostgreSQL function that atomically inserts the vote and checks the count:
```sql
CREATE OR REPLACE FUNCTION submit_review(
  p_ship_check_id UUID,
  p_reviewer_id UUID,
  p_verdict TEXT,
  p_reason TEXT DEFAULT NULL
) RETURNS TEXT AS $$
DECLARE
  v_crew_size INT;
  v_approve_count INT;
  v_majority INT;
  v_current_status TEXT;
BEGIN
  -- Lock the ship_check row to prevent concurrent status changes
  SELECT verification_status INTO v_current_status
  FROM ship_checks
  WHERE id = p_ship_check_id
  FOR UPDATE;

  IF v_current_status != 'pending' THEN
    RETURN v_current_status; -- Already resolved
  END IF;

  -- Insert the review (unique constraint on ship_check_id + reviewer_id prevents duplicates)
  INSERT INTO proof_reviews (ship_check_id, reviewer_id, verdict, reason)
  VALUES (p_ship_check_id, p_reviewer_id, p_verdict, p_reason);

  -- Count approvals
  SELECT COUNT(*) INTO v_approve_count
  FROM proof_reviews
  WHERE ship_check_id = p_ship_check_id AND verdict = 'approve';

  -- Get crew size for majority calculation
  SELECT COUNT(*) INTO v_crew_size
  FROM crew_members cm
  JOIN ship_checks sc ON sc.crew_id = cm.crew_id
  WHERE sc.id = p_ship_check_id AND cm.user_id != sc.user_id;

  v_majority := (v_crew_size / 2) + 1;

  IF v_approve_count >= v_majority THEN
    UPDATE ship_checks SET verification_status = 'approved' WHERE id = p_ship_check_id;
    RETURN 'approved';
  END IF;

  RETURN 'pending';
END;
$$ LANGUAGE plpgsql;
```
2. **`FOR UPDATE` lock on the ship_check row** serializes concurrent reviews. The second reviewer waits for the first to commit before reading the count
3. **Unique constraint on `(ship_check_id, reviewer_id)`** prevents double-voting at the database level
4. **Return the current status** so the application layer knows whether to trigger notifications -- only trigger on the transition from 'pending' to 'approved'
5. **Handle the 48-hour auto-approve** with a separate cron job, not inline with voting. The cron job should also use `FOR UPDATE` to prevent racing with a last-second vote

**Detection:** Integration test that submits two reviews concurrently (using `Promise.all` in a test) and asserts that the final status is correct and only one notification was sent.

---

### Pitfall 5: GitHub Webhook Security -- HMAC Verification Bypass

**Severity:** CRITICAL
**Affects phase:** Phase 2 (Gamification + GitHub Integration)
**Confidence:** HIGH (well-documented security requirement)

**What goes wrong:** The GitHub App sends webhooks to `/api/webhooks/github`. If the endpoint does not verify the `X-Hub-Signature-256` header, anyone who discovers the URL can forge webhook payloads. An attacker could fake PR merges, releases, or commits to inflate their trust score, trigger auto-verification on fraudulent proofs, and game the leaderboard.

**Why it happens:**
- Webhook endpoints are public URLs -- they have to be, for GitHub to reach them
- During development, signature verification is often skipped "for now"
- The verification code requires the webhook secret, which may not be set up in all environments
- If verification is implemented but the error is swallowed (returns 200 instead of 401), the endpoint processes forged payloads

**Specific DevReal risks:**
- Forged `pull_request.closed` (with `merged: true`) webhook auto-verifies a proof without crew review
- Forged `release.published` webhook inflates shipping velocity and trust score
- An attacker could auto-verify every proof submission by replaying webhook events with modified payload

**Warning signs:**
- Unexplained auto-verifications for users who did not actually push code
- Webhook events arriving for repositories the app is not installed on
- Trust scores inflating suspiciously for specific users

**Prevention:**
1. **Verify HMAC signature on EVERY webhook request before processing:**
```typescript
import { createHmac, timingSafeEqual } from 'crypto';

function verifyGitHubWebhook(payload: string, signature: string, secret: string): boolean {
  const expected = `sha256=${createHmac('sha256', secret).update(payload).digest('hex')}`;
  return timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}
```
2. **Use `timingSafeEqual`**, not `===`, to prevent timing attacks
3. **Return 401 immediately if verification fails** -- do not process the payload
4. **Store the webhook secret in environment variables**, never in code
5. **Validate the `X-GitHub-Event` header** matches expected events -- reject unknown event types
6. **Verify the `installation.id` in the payload** matches a known installation for your app -- this prevents cross-app replay attacks
7. **Idempotency:** Store processed webhook delivery IDs (`X-GitHub-Delivery` header) and reject duplicates. GitHub retries on failure, so you will receive duplicates
8. **Rate limit the webhook endpoint** -- even with signature verification, protect against replay floods

**Detection:** Canary test: send a request to the webhook endpoint with an invalid signature. If it returns anything other than 401, the verification is broken.

---

### Pitfall 6: File Upload Vulnerabilities via Supabase Storage

**Severity:** CRITICAL
**Affects phase:** Phase 1 (MVP Core) -- Proof Submission
**Confidence:** HIGH (standard web security concern)

**What goes wrong:** Users upload "screenshots" as proof of shipping. Without validation, they can upload executable files (`.html` with JavaScript, `.svg` with embedded scripts), enormous files (100 MB images that cost storage and bandwidth), or EXIF-laden images that leak GPS location, device info, and other metadata.

**Why it happens:**
- Supabase Storage accepts any file type by default
- Client-side validation is trivially bypassed
- MIME type headers can be spoofed (`Content-Type: image/png` on an HTML file)
- Browsers will execute JavaScript in SVG files served with the correct MIME type
- EXIF data in JPEG/PNG files often contains GPS coordinates, camera model, and timestamps

**Specific DevReal risks:**
- An uploaded SVG containing `<script>` tags could execute JavaScript when rendered in the crew feed (stored XSS)
- Signed URLs are time-limited but still serve the raw file -- if the file is malicious, the signed URL is a delivery mechanism
- Users upload high-resolution screenshots (5+ MB each). With 300 daily active users uploading daily, that is 1.5 GB/day, exhausting Supabase free tier (1 GB) in less than a day
- EXIF GPS data in screenshots could reveal users' home/office locations

**Warning signs:**
- Storage costs spike unexpectedly
- Supabase free tier storage fills up within weeks
- User reports of "weird behavior" when viewing proof images
- Privacy-conscious users notice location data in their uploaded images

**Prevention:**
1. **Validate file type on the server side** by checking magic bytes (file header), not just the MIME type or extension:
```typescript
// Check file magic bytes
const ALLOWED_MAGIC = {
  'image/png': [0x89, 0x50, 0x4E, 0x47],
  'image/jpeg': [0xFF, 0xD8, 0xFF],
  'image/webp': [0x52, 0x49, 0x46, 0x46], // RIFF header
};
```
2. **Set maximum file size to 2 MB** for proof screenshots. This is large enough for a high-quality screenshot and small enough to control costs. Enforce at the Supabase Storage bucket level (`fileSizeLimit` in bucket configuration) AND at the API route level
3. **Explicitly block SVG uploads.** SVGs can contain arbitrary JavaScript. If SVG is needed later, serve it through a sanitization pipeline
4. **Strip EXIF data on upload** using a library like `sharp` (already ideal for image processing in Node.js):
```typescript
import sharp from 'sharp';
const sanitized = await sharp(buffer)
  .rotate() // Apply EXIF orientation before stripping
  .resize(1920, 1080, { fit: 'inside', withoutEnlargement: true })
  .jpeg({ quality: 85 })
  .toBuffer();
```
5. **Serve uploaded images from a separate domain** (e.g., `cdn.devreal.dev`) or with `Content-Disposition: attachment` to prevent browser execution. Supabase signed URLs serve from the Supabase domain by default, which provides some isolation
6. **Set Supabase Storage bucket to private** and use signed URLs with short expiry (15-30 minutes) for viewing. Never make proof screenshots publicly accessible via permanent URLs
7. **Implement client-side image compression** before upload using browser canvas or a library like `browser-image-compression` to keep uploads under 1 MB

**Detection:** Upload test files with wrong extensions (`.png` file that is actually HTML) in integration tests. Verify the server rejects them. Scan uploaded files periodically for non-image content.

---

## High Pitfalls

Mistakes that cause major UX impact, significant rework, or user trust damage.

---

### Pitfall 7: Cold Start Death Spiral -- Crews of 1-2 Kill Engagement

**Severity:** HIGH
**Affects phase:** Phase 1 (MVP Core) -- Onboarding, Phase 3 (Growth)
**Confidence:** HIGH (every social app faces this; competitor post-mortems confirm)

**What goes wrong:** DevReal requires crews of 3-6. A user signs up, creates a crew, invites friends. One friend joins. Now it is a crew of 2 -- below the minimum for meaningful peer verification (majority of 1 = just yourself). The user submits proof, nobody reviews it, it auto-approves after 48 hours. The experience feels empty. Both users churn within a week.

The spec says "Allow solo start with crew nudges" and "7-day grace period when crew drops below 3." But the first-run experience IS the grace period for most users. If their first week is a dead crew, there is no second week.

**Why it happens:**
- The invite-to-join conversion rate is typically 20-40% for consumer apps
- A user must invite 5-10 people to get 3 crew members
- Early adopters' friends are not early adopters -- they wait to see if the app sticks
- There is no immediate value from the app before the crew reaches critical mass
- WIP.co, Makerlog, and Buildspace all suffered from this -- WIP defaulted to a giant chat, Makerlog to a feed, Buildspace to cohorts with assigned groups

**Warning signs:**
- Average crew size at day 7 is below 3
- More than 50% of crews never reach 3 members
- Retention drops sharply in the first week
- Users submit proofs that are never reviewed by a human

**Prevention:**
1. **Matchmaking over creation for the first 500 users.** Instead of "Create a crew," offer "Join an open crew" as the primary onboarding path. Seed 10-15 open crews with 2-3 founding members each (early testers, beta users, the DevReal team itself). New users join existing crews rather than creating empty ones
2. **"Crew of interest" system:** During onboarding, ask "What are you building?" (web app / mobile app / open source / design / content). Match users into crews by interest area. This gives crews something in common beyond "we all signed up the same day"
3. **Solo mode is genuinely useful:** The spec mentions solo mode with limited features. Make sure solo mode includes: personal streak tracking, the ship check mechanic, a public shipping log. Users should see value BEFORE they have a crew -- the crew amplifies it, not enables it
4. **Aggressively fill undersize crews:** If a crew has 2 members, surface it in "Looking for crew members" listings. Allow the crew to opt into auto-filling from matched users
5. **Launch strategy: simultaneous cohort.** Instead of a slow trickle of signups, batch the first 100+ users into a "Founding Class" launch event. Everyone starts the same week, crews are pre-seeded, and there is critical mass from day one. Buildspace did this well with their seasons
6. **Measure crew health, not just user signups.** The north star for launch is not "500 users" -- it is "75 crews with 4+ active members." Track and optimize for crew health from the start

**Detection:** Track "time to crew of 3" as a key metric. If the median is above 7 days, the cold start is failing. Track "proof reviews by a human" -- if most proofs are auto-approved (48h timeout), crews are too small or inactive.

---

### Pitfall 8: Gamification Burnout and Streak Anxiety

**Severity:** HIGH
**Affects phase:** Phase 2 (Gamification) -- Streaks, Leagues, Scoring
**Confidence:** HIGH (extensively documented in Duolingo research, Snapchat streak studies)

**What goes wrong:** Users build a 45-day streak. It becomes the most important thing in the app. They submit low-quality "proofs" (a screenshot of VS Code with one line changed) just to maintain the streak. They skip weekends with family to avoid breaking it. When they inevitably do break it, they feel crushed and churn immediately. The streak was not motivating shipping -- it was motivating streak maintenance.

**Why it happens:**
- Loss aversion is 2x stronger than gain-seeking (Kahneman & Tversky) -- losing a 45-day streak feels worse than gaining a new 1-day streak
- Snapchat research shows streaks create anxiety, sleep disruption, and unhealthy attachment patterns
- The spec says "miss 1 day = streak resets" -- this is the harshest possible reset policy
- "2 free freezes/month" helps but is insufficient for a 30-day streak (you need zero bad days in any 30-day window)
- Duolingo found that streak freezes reduced churn by 21%, but DevReal's core loop is more demanding than a 5-minute language lesson

**Specific DevReal risks:**
- Users gaming the system with trivial declarations and low-effort proofs just to maintain streaks
- Streak maintenance becomes the goal, not shipping quality work
- Users who break a long streak churn at a much higher rate than users who never built one
- Weekend/holiday submissions are lower quality because they are obligation, not genuine progress
- The "Late" tag on late submissions adds shame without adding accountability

**Warning signs:**
- Proof complexity ratings (peer-rated) decline as streak length increases
- Weekend proof submissions have notably lower crew approval rates
- Churn rate for users who break a 30+ day streak is 3-5x higher than average
- Users complain about "feeling like a chore" in feedback

**Prevention:**
1. **Reduce the streak penalty.** Instead of "miss 1 day = reset to 0," use a degrading streak: missing 1 day reduces the streak multiplier by 50% (not to zero). Missing 2 consecutive days resets. This preserves some investment for a single bad day
2. **Increase freeze availability.** 2 per month is too few. Consider 4 per month (1 per week effectively), or earn freezes through consistent shipping (7-day streak = earn a freeze)
3. **Weekend mode (borrow from ShipStreaks).** Allow users to opt out of Saturday/Sunday without affecting their streak. Many developers do not ship on weekends, and forcing them to creates resentment
4. **Quality gate:** Track average proof complexity alongside streak length. If a user's 30-day rolling average complexity drops below 2 while their streak is above 14, trigger a "Challenge Yourself" prompt. Do not automatically penalize -- just surface the pattern
5. **"Streak insurance" from crew activity.** If a user misses a day but their crew had 100% participation that day, the crew's activity "covers" the miss once per month. This reinforces crew bonds
6. **Celebrate the comeback, not just the streak.** When a user returns after a broken streak, give them a "Returning Shipper" 7-day XP boost (spec already mentions this). Make the return feel rewarding, not shameful
7. **Never show "0-day streak" after a break.** Show "Last streak: 45 days" prominently. The history matters. A user who has shipped for 45 consecutive days is not a beginner, even if their current streak is 1

**Detection:** Segmented churn analysis: compare churn rates for users who break streaks at different lengths (7, 14, 30, 60, 100 days). If breaking a 30+ day streak predicts churn, the system is creating unhealthy attachment.

---

### Pitfall 9: Vercel Cron Job Limitations Break Ship Check Scheduling

**Severity:** HIGH
**Affects phase:** Phase 1 (MVP Core) -- Random Ship Check Timing
**Confidence:** MEDIUM (verify current Vercel cron limits; these may have changed since mid-2025)

**What goes wrong:** DevReal needs to send a "Ship Check" notification at a random time during each user's active hours. The naive approach is a cron job that runs every minute, checks which users need a notification, and sends them. Vercel's free tier limits cron jobs to 1 per day. Pro tier allows cron jobs down to every minute, but each invocation is a serverless function call that counts against invocation limits (100K/month on free, 1M on Pro).

**Why it happens:**
- Vercel Cron is designed for periodic tasks (daily reports, weekly cleanups), not per-user scheduling
- Per-user random timing requires either (a) a per-minute cron that scans all users, or (b) pre-scheduling exact times and firing at those times
- 300 daily active users with per-minute cron = 1,440 cron invocations/day = 43,200/month -- 43% of the free tier's total invocation budget just for cron
- Each cron invocation must read from the database (which users need notification NOW?), adding to Supabase query load

**Specific DevReal risks:**
- Free tier: 1 cron job per day is completely insufficient. You cannot do random-time ship checks with a daily cron
- Pro tier ($20/month): Cron runs every minute, 1M invocations/month. The cron alone consumes ~43K invocations/month. At 5,000 DAU, the database query per invocation becomes the bottleneck
- Serverless cold starts add 200-500ms to each cron invocation. If the function takes >10s to process (querying users, sending notifications), it may overlap with the next invocation
- Vercel serverless functions have a 10-second timeout on the free tier and 60-second on Pro

**Warning signs:**
- Ship check notifications consistently arrive at the same time (the cron is firing but not randomizing)
- Notifications are late or missing for some users
- Vercel dashboard shows function timeout errors on the cron route
- Invocation count approaches plan limits mid-month

**Prevention:**
1. **Pre-schedule ship check times daily, not per-minute.** Run a single daily cron job at midnight UTC that:
   - For each active user, generates a random UTC timestamp within their active hours (converting from their timezone)
   - Stores this timestamp in a `ship_check_schedule` table
   - Example: User in UTC+9 with active hours 9 AM - 6 PM = random time between 00:00 UTC and 09:00 UTC
2. **Use a per-minute cron (or external scheduler) to dispatch.** Every minute, query `SELECT * FROM ship_check_schedule WHERE scheduled_at <= NOW() AND sent = false`. Send notifications for matching rows and mark them sent. This is a simple indexed query, not a full user scan
3. **Alternative: Use Supabase `pg_cron` instead of Vercel Cron.** `pg_cron` runs inside the database, can execute every minute with no serverless overhead, and can directly call a Supabase Edge Function to send notifications. This avoids Vercel invocation limits entirely
4. **Alternative: Use an external job scheduler** (e.g., Trigger.dev, Inngest, or QStash by Upstash) that supports delayed/scheduled jobs natively. Schedule each notification as an individual job at the pre-computed time
5. **Batch notification sending.** Group users whose ship check time falls within the same minute. Send batch push notifications via FCM's batch API (up to 500 per request) rather than individual sends
6. **Monitor invocation budget.** Set up alerts at 50% and 80% of monthly invocation limits. If hit, reduce cron frequency or move to `pg_cron`

**Detection:** Track "ship check notification delivery rate" -- what percentage of scheduled notifications were actually sent within 5 minutes of the scheduled time? If below 95%, the scheduling system is failing.

---

### Pitfall 10: OAuth Redirect and Session Handling Broken on Vercel Preview Deployments

**Severity:** HIGH
**Affects phase:** Phase 1 (MVP Core) -- Authentication
**Confidence:** MEDIUM (common with Supabase Auth + Vercel but specific behavior may have been fixed)

**What goes wrong:** GitHub OAuth requires a callback URL registered in the GitHub App settings. Vercel creates preview deployments on unique URLs (e.g., `devreal-git-feature-auth-kurtjallo.vercel.app`). The callback URL does not match the preview deployment URL, so OAuth redirects fail on every preview deployment. Developers cannot test auth flows on feature branches.

Additionally, Supabase Auth's `@supabase/ssr` package requires specific cookie handling in the Next.js App Router middleware. If the middleware is not correctly configured, the auth session is lost between server components and client components, causing infinite redirect loops or phantom logouts.

**Why it happens:**
- GitHub Apps only allow specific callback URLs (not wildcards)
- Vercel preview URLs are dynamic and unpredictable
- Supabase Auth stores session tokens in cookies, and the cookie domain must match the deployment URL
- Next.js App Router's server/client component boundary creates two separate execution contexts that must share the auth session via cookies
- The `createServerClient` and `createBrowserClient` functions from `@supabase/ssr` have different initialization patterns

**Specific DevReal risks:**
- Cannot test auth flows on preview deployments without manual GitHub App configuration
- Users experience "signed out" state when navigating between pages (cookie not properly refreshed)
- Auth state desync between server-rendered and client-rendered components
- Google OAuth has the same callback URL restriction

**Warning signs:**
- "Redirect URI mismatch" errors on preview deployments
- Users are randomly logged out when navigating
- Auth works on initial login but breaks on page refresh
- Server components show unauthenticated state while client components show authenticated state

**Prevention:**
1. **For preview deployments:** Use a separate GitHub App for development with `localhost:3000` and a wildcard Vercel preview URL pattern. Or use Supabase's built-in auth UI for preview testing and only test full GitHub OAuth on the production URL
2. **For Supabase Auth session handling**, follow the official `@supabase/ssr` pattern exactly:
   - Create a middleware that refreshes the session on every request
   - Use `createServerClient` in server components with cookie access
   - Use `createBrowserClient` (singleton) in client components
   - Never create the Supabase client inside a `useEffect` -- it must be created at module level or in a provider
3. **Set cookie options explicitly:**
```typescript
cookieOptions: {
  domain: process.env.NODE_ENV === 'production' ? '.devreal.dev' : 'localhost',
  sameSite: 'lax',
  secure: process.env.NODE_ENV === 'production',
  httpOnly: true,
  path: '/',
  maxAge: 60 * 60 * 24 * 7, // 7 days
}
```
4. **Test auth flow end-to-end** with Playwright or Cypress, including: sign in, navigate between pages, refresh, sign out, sign back in, open multiple tabs. All of these should maintain consistent auth state
5. **Handle the auth callback route correctly.** The `/auth/callback` route must exchange the authorization code for a session AND redirect to the intended page. If this route is a server action (not an API route), ensure it sets cookies before redirecting

**Detection:** Automated E2E test that: signs in with a test account, navigates to 5 different pages, refreshes each page, and asserts that the user remains authenticated throughout. Run on every PR.

---

### Pitfall 11: Notification Fatigue Causes User Churn

**Severity:** HIGH
**Affects phase:** Phase 2-3 (Gamification, Growth) -- Notification System
**Confidence:** HIGH (extensively studied in consumer app research)

**What goes wrong:** DevReal has many notification triggers: morning declaration reminder, random ship check, crew member declared, crew member submitted proof, proof needs review, streak at risk, league promotion/demotion, nudge received, evening reflection reminder. If all are enabled by default, users receive 5-8 notifications per day. Notification fatigue sets in within 2 weeks, users disable all notifications, and without the daily nudge (which IS the product), they stop opening the app.

**Why it happens:**
- The spec mentions "Maximum 3 notifications per day" as a principle, but the notification types listed exceed 3 easily
- Product teams add notifications incrementally ("just one more trigger"), and each seems reasonable in isolation
- The urgency of ship checks requires notifications, creating pressure to keep notification volume high
- Different users have different tolerance levels, and a one-size-fits-all approach fails

**Specific DevReal risks:**
- The daily ship check is the most important notification in the app. If users disable all notifications because of fatigue from lower-priority ones, they miss ship checks and their engagement drops
- Crew activity notifications (someone declared, someone submitted proof) are high-frequency in active crews -- a 6-person crew generates 12+ events per day
- Review request notifications create obligation pressure -- "your crew needs you to review" feels like work
- Multiple notification channels (push, email, in-app) multiply the perceived volume

**Warning signs:**
- Notification permission revocation rate exceeds 10% within 30 days
- Open rates for push notifications decline week over week
- Users explicitly mention "too many notifications" in feedback
- DAU drops correlate with notification volume increases

**Prevention:**
1. **Enforce a hard cap of 3 push notifications per day per user.** Priority order: (1) ship check, (2) one crew activity batch, (3) one time-sensitive alert (streak at risk, review needed). Everything else goes to the in-app notification center only
2. **Batch crew activity notifications.** Instead of individual pushes for each declaration/proof, send a single daily digest: "3 crewmates declared today. 2 submitted proof." Send this at a fixed time (e.g., noon local time)
3. **Make notification categories independently toggleable** from the start. The spec already mentions notification preferences -- implement this in Phase 1, not Phase 3
4. **The ship check notification is sacred.** It must always be delivered even if all other notifications are muted. Frame it as the one essential notification during onboarding: "This is the one notification that matters"
5. **Use in-app badges instead of push notifications** for low-urgency events (reactions, league standings, crew milestones). The badge creates curiosity without the interrupt
6. **A/B test notification frequency** from launch. Segment users into 2-notification and 3-notification groups. Measure 30-day retention for each group. Optimize for retention, not "engagement" (opens)
7. **Smart suppression:** If a user has already opened the app today, suppress the morning declaration reminder. If they already submitted proof, suppress the evening reminder. Only notify about things they have not done

**Detection:** Track notification permission status (granted, denied, revoked) as a first-class metric. If revocation rate exceeds 5% in any 7-day period, notification volume is too high.

---

### Pitfall 12: Drizzle ORM + Supabase Auth ID Type Mismatch

**Severity:** HIGH
**Affects phase:** Phase 1 (MVP Core) -- Data Layer
**Confidence:** MEDIUM (depends on current Drizzle + Supabase compatibility; verify)

**What goes wrong:** Supabase Auth generates user IDs as UUIDs in the `auth.users` table. Drizzle ORM schemas define columns with specific types. If the Drizzle schema defines `userId` as `text()` instead of `uuid()`, or uses a different UUID format, foreign key constraints and RLS policies that reference `auth.uid()` silently fail. The user is authenticated but their queries return zero rows because `auth.uid()::uuid != users.id::text`.

**Why it happens:**
- Drizzle schemas are defined in TypeScript, separate from the Supabase Auth schema
- The `users` table in the app's public schema must link to `auth.users` with a matching type
- PostgreSQL is strict about type comparison -- `uuid` and `text` are not implicitly comparable
- Copy-paste errors in schema definitions can use the wrong type
- Some tutorials use `text('id')` for simplicity, which breaks RLS policies that cast `auth.uid()` as UUID

**Warning signs:**
- RLS policies that work in SQL editor fail through the application
- `auth.uid()` returns a value but queries using it return no rows
- Type errors in Drizzle migrations mentioning UUID/text mismatches
- Foreign key constraints fail with "type mismatch" errors

**Prevention:**
1. **Use `uuid()` type consistently for all user ID columns in Drizzle:**
```typescript
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  // ... other columns
});
```
2. **Create a database trigger that syncs `auth.users` to your public `users` table** on signup. This is the standard Supabase pattern:
```sql
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger AS $$
BEGIN
  INSERT INTO public.users (id, username, avatar_url, email)
  VALUES (NEW.id, NEW.raw_user_meta_data->>'user_name', NEW.raw_user_meta_data->>'avatar_url', NEW.email);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```
3. **Test RLS policies with Drizzle queries, not just raw SQL.** A policy that works in SQL editor might fail through Drizzle if the types do not match
4. **In RLS policies, always cast explicitly:** `auth.uid() = user_id` (both UUID) rather than mixing types

**Detection:** Write a test that creates a test user via Supabase Auth, then queries the public `users` table via Drizzle with RLS enabled. If the query returns the user, types are aligned. If it returns nothing, there is a type mismatch.

---

## Moderate Pitfalls

Mistakes that cause technical debt, performance degradation, or delayed timelines.

---

### Pitfall 13: N+1 Queries on Crew Feed and Leaderboard

**Severity:** MEDIUM
**Affects phase:** Phase 1 (MVP Core) -- Crew Feed, Phase 2 -- Leaderboard
**Confidence:** HIGH (standard ORM pitfall)

**What goes wrong:** The crew feed shows declarations, proofs, reactions, and user profiles. A naive implementation fetches the crew, then loops over members to fetch declarations, then loops over declarations to fetch reactions. For a 6-person crew with 6 declarations and 4 reactions each, this generates 1 + 6 + 6 + 24 = 37 queries instead of 2-3 JOINed queries. The leaderboard is worse -- aggregating scores across users, goals, proofs, and reviews can generate hundreds of queries.

**Why it happens:**
- Drizzle ORM's relational queries (`db.query.users.findMany({ with: { declarations: true } })`) generate efficient JOINs, but mixing imperative code with queries creates N+1 patterns
- React Server Components make it easy to fetch data in each component independently -- each component issues its own query
- The leaderboard requires aggregating across multiple tables (users, declarations, ship_checks, proof_reviews, daily_activity) which tempts sequential fetching

**Prevention:**
1. **Use Drizzle's relational query API** for nested data:
```typescript
const crewFeed = await db.query.declarations.findMany({
  where: eq(declarations.crewId, crewId),
  with: {
    user: true,
    shipCheck: { with: { proofReviews: true } },
    reactions: { with: { user: true } },
  },
  orderBy: desc(declarations.createdAt),
  limit: 50,
});
```
2. **For the leaderboard, use a database view or materialized view** that pre-computes composite scores. The `daily_activity` summary table in the spec is the right idea -- extend it to include pre-computed league scores
3. **Use `Promise.all` for independent queries** in server components rather than sequential awaits
4. **Add query logging in development** to catch N+1 patterns early. Drizzle supports a `logger` option:
```typescript
const db = drizzle(client, { schema, logger: true });
```
5. **Set a query budget per page load:** Crew feed should use <= 3 queries. Leaderboard should use <= 2 queries (one for standings, one for user details). Profile page should use <= 4 queries

**Detection:** Enable Drizzle query logging in development. If any page load generates more than 10 queries, investigate.

---

### Pitfall 14: Vercel Serverless Function Cold Starts Degrade Ship Check UX

**Severity:** MEDIUM
**Affects phase:** Phase 1 (MVP Core)
**Confidence:** MEDIUM (cold start behavior depends on runtime and deployment configuration)

**What goes wrong:** The user taps the ship check notification. The app loads a server component that creates a Supabase client, checks auth, and fetches crew data. If this is the first request to this function in ~5-15 minutes, the serverless function cold starts: initializing the runtime, importing dependencies, establishing database connections. This adds 500ms-2s to the response time. For a feature with a 5-minute countdown timer, 2 seconds of loading feels like an eternity.

**Why it happens:**
- Vercel serverless functions are stateless -- they spin down after inactivity
- Drizzle ORM is lightweight (~7KB), but the PostgreSQL client still needs to establish a TCP connection
- Supabase's connection pooler (Supavisor) helps but does not eliminate the connection setup time
- Next.js App Router server components re-initialize on every request in serverless mode

**Prevention:**
1. **Use Supabase connection pooler URL** (port 6543 with `?pgbouncer=true`) for serverless connections, not the direct connection URL. This eliminates per-request TCP handshakes
2. **Optimize function bundle size.** Use Drizzle over Prisma (already chosen). Avoid importing heavy libraries in API routes -- tree-shake aggressively
3. **Pre-warm critical paths.** Use a cron job to hit the ship check page every 5 minutes to keep the function warm. This costs ~8,640 invocations/month (well within limits)
4. **Client-side optimistic rendering.** Show the ship check UI immediately with cached data from the last session. Fetch fresh data in the background. The countdown timer, proof tabs, and submit button do not need server data to render
5. **Use Edge Runtime for latency-critical routes.** Next.js supports `export const runtime = 'edge'` on individual routes. Edge functions have near-zero cold starts (~50ms). However, Edge Runtime does not support Node.js APIs (no `crypto` module, no `fs`), so only use it for read-heavy routes
6. **Connection caching.** In the serverless function, cache the database connection at the module level:
```typescript
let db: ReturnType<typeof drizzle> | null = null;
function getDb() {
  if (!db) {
    db = drizzle(new Pool({ connectionString: process.env.DATABASE_URL }));
  }
  return db;
}
```
This reuses the connection across invocations within the same function instance.

**Detection:** Monitor Time to First Byte (TTFB) for the ship check route. If P95 TTFB exceeds 2 seconds, cold starts are impacting UX.

---

### Pitfall 15: Composite Scoring Formula Creates Perverse Incentives

**Severity:** MEDIUM
**Affects phase:** Phase 2 (Gamification) -- Scoring System
**Confidence:** HIGH (game design principle; extensively analyzed in docs/research/03-gamification-mechanics.md)

**What goes wrong:** The composite scoring formula (Consistency 35%, Completion Rate 25%, Shipping Velocity 20%, Ambition Score 20%) creates optimization targets. Users will find the highest-ROI metric to game. For example:
- Consistency (35%) rewards showing up every day regardless of quality -- this has the highest weight, so users will prioritize daily presence over meaningful work
- Completion Rate (25%) rewards finishing what you declare -- so users declare trivially easy goals they can certainly complete
- Ambition Score (20%) is supposed to prevent trivial goals, but if peer-rated complexity is gamed (friends rate each other's trivial work as complex), this defense fails

**Why it happens:**
- Goodhart's Law: "When a measure becomes a target, it ceases to be a good measure"
- Users optimize for the metric, not the behavior the metric is supposed to represent
- In a small crew of friends, social pressure to rate honestly is counterbalanced by social pressure to help each other's scores
- The formula's weights are arbitrary -- there is no empirical basis for 35/25/20/20

**Warning signs:**
- Average declared goal complexity trends downward over time
- Completion rates approach 100% (everyone is completing everything -- suspicious)
- Users with the highest scores have the lowest-quality proofs
- Crew complexity ratings cluster at 5 or 8 (everyone picks the same "respectable" number)

**Prevention:**
1. **Launch with simpler scoring and iterate.** Phase 2 should use only Consistency + Completion Rate. Add Velocity and Ambition in a later iteration after observing real user behavior. Do not ship a 4-factor formula on day one
2. **Cross-crew complexity validation.** Randomly show goals + proofs to users outside the crew for independent complexity rating. Compare cross-crew ratings with in-crew ratings. Flag discrepancies
3. **Sliding windows, not all-time averages.** Use a 4-week rolling window for all metrics. This prevents early gaming from permanently inflating scores
4. **Transparency over secrecy.** Show users how their score is calculated. Hidden formulas invite conspiracy theories and frustration. Visible formulas invite optimization, but at least the optimization is directed
5. **Qualitative reviews.** Periodically ask crew members "Is [user]'s score accurate?" as a sentiment check. This cannot be gamed easily and provides ground truth
6. **Monitor the distribution.** Scores should follow a roughly normal distribution. If they cluster at the top (everyone has 90+), the system is not differentiating. If they cluster at extremes (everyone is either 95 or 20), the system is too binary

**Detection:** Track the correlation between composite score and proof complexity (peer-rated). If the correlation is negative or zero (high scores do not predict high-quality work), the scoring system is being gamed.

---

### Pitfall 16: 48-Hour Auto-Approve Window Enables Gaming

**Severity:** MEDIUM
**Affects phase:** Phase 1 (MVP Core) -- Verification
**Confidence:** HIGH (logical analysis of the specified rules)

**What goes wrong:** The spec says proofs auto-approve if no majority is reached within 48 hours. In a 3-person crew where one member submits proof, only 1 other member needs to approve (majority of 2 = 1 vote). If that one member is inactive or on vacation, the proof auto-approves without any human review. A colluding pair in a 4-person crew can game this: one submits low-quality proof, the other approves it instantly, and the remaining two members never review.

**Why it happens:**
- The auto-approve timeout is a safety net against blocking, which is correct
- But combined with small crews and majority voting, the bar for approval is very low
- A 3-person crew has the weakest verification: 1 vote approves
- The spec allows "auto-verified proofs (GitHub PRs, deploys) skip crew review entirely" -- this is fine for auto-verified, but the auto-approve timeout creates a similar bypass for manual proofs

**Prevention:**
1. **Minimum review count regardless of majority.** Require at least 2 reviews (regardless of crew size) before auto-approve kicks in. In a 3-person crew, this means both other members must weigh in, or the 48-hour timeout auto-approves
2. **Extend auto-approve timeout for small crews.** 48 hours for crews of 5-6. 72 hours for crews of 3-4. Smaller crews need more time because each reviewer's absence has a larger impact
3. **Track auto-approve rate per crew.** If more than 50% of a crew's proofs auto-approve, surface a warning: "Your crew has unreviewed proofs. Review your crewmates' work to keep accountability strong"
4. **Auto-approved proofs earn lower trust than peer-approved proofs.** This is already implied by the 3-tier trust system but should be explicit: auto-approved = Level 1 (self-reported equivalent), not Level 2 (peer-verified)
5. **Notification escalation.** At 24 hours without review, send a second notification to reviewers. At 36 hours, send a crew-wide "Proof needs review" notification

**Detection:** Track the percentage of proofs that are auto-approved vs. explicitly approved. If auto-approve rate exceeds 30%, crews are not actively reviewing.

---

### Pitfall 17: Database Performance Cliff with League Calculations

**Severity:** MEDIUM
**Affects phase:** Phase 2 (Gamification) -- Weekly Leagues
**Confidence:** MEDIUM (depends on query patterns and data volume)

**What goes wrong:** Weekly league calculations require aggregating composite scores across all users, ranking them, determining promotions/demotions, and creating new league_standings rows. The spec says "Full league history preserved (one row per user per week)." At 5,000 users after 52 weeks, that is 260,000 rows. The Monday reset query must: read the current week's scores for all users, compute rankings within each league, determine top 3 and bottom 3 per league, insert new rows for the next week, and update user records. If this runs as a single transaction, it locks the league_standings table for seconds.

**Why it happens:**
- League calculation is a batch operation, not a transactional one
- Running it as a single SQL query or transaction is natural but creates lock contention
- The composite score calculation itself may require JOINs across declarations, ship_checks, proof_reviews, and daily_activity for each user
- If the calculation runs during Monday morning when users are actively checking their standings, read queries contend with the write transaction

**Prevention:**
1. **Pre-compute league scores continuously, not in a Monday batch.** Maintain a `current_league_score` column on the users table that updates incrementally on each scoring event (declaration, proof, review). The Monday reset only needs to read pre-computed scores, not recalculate from raw data
2. **Run league reset during a low-traffic window.** Monday 00:00 UTC is the spec, but if most users are in US timezones, 00:00 UTC (7-8 PM Eastern Sunday) is peak time. Consider 06:00 UTC (1-2 AM Eastern Monday) instead
3. **Use `INSERT ... SELECT` for batch league row creation** instead of application-level loops. Let PostgreSQL handle the set operation
4. **Partition league_standings by week or month** using PostgreSQL table partitioning. This keeps queries fast as the table grows -- each week's data is in its own partition
5. **Add appropriate indexes:** `(week_start, league, weekly_score DESC)` for the leaderboard query and `(user_id, week_start)` for per-user history
6. **Use a materialized view for the "current league standings"** page. Refresh it at reset time and periodically (every 15 minutes) during the week. This prevents every leaderboard page load from running the aggregation query

**Detection:** Monitor query execution time for the league standings page. If it exceeds 500ms at any point, optimization is needed. Monitor the Monday reset job duration -- it should complete in under 30 seconds even at 10,000 users.

---

### Pitfall 18: Supabase Storage Signed URL Expiry Creates Broken Images

**Severity:** MEDIUM
**Affects phase:** Phase 1 (MVP Core) -- Proof Screenshots
**Confidence:** HIGH (inherent to signed URL architecture)

**What goes wrong:** Supabase Storage generates signed URLs with a configurable expiry (default: 1 hour). Users submit proof with a screenshot. The signed URL is stored in the `ship_checks` table. Two hours later, a crew member opens the crew feed to review the proof. The image URL has expired. They see a broken image placeholder. The proof appears invalid even though the image exists in storage.

**Why it happens:**
- Signed URLs are designed for temporary access, not permanent references
- The naive implementation stores the signed URL in the database instead of the storage path
- Regenerating signed URLs on every feed load adds latency and Supabase API calls
- The crew has 48 hours to review -- any fixed expiry shorter than 48 hours will cause broken images

**Prevention:**
1. **Store the storage path, not the signed URL.** The `ship_checks.proof_data` column should store `proofs/user-123/2026-02-20/screenshot.jpg`, not `https://xyz.supabase.co/storage/v1/object/sign/proofs/...?token=...`
2. **Generate signed URLs on demand** when rendering the feed:
```typescript
const { data } = await supabase.storage
  .from('proofs')
  .createSignedUrl(shipCheck.proofData, 3600); // 1 hour expiry
```
3. **Cache signed URLs client-side** with their expiry time. Regenerate before expiry instead of on every render
4. **Set a reasonable expiry.** For crew feed viewing, 1 hour is fine (users do not leave the feed open for hours). For profile pages and historical viewing, generate on demand with longer expiry (24 hours)
5. **Alternative: Use Supabase's public bucket with auth middleware.** If the proof images do not contain sensitive information (they are shared with the crew anyway), a public bucket with an unpredictable path (`/proofs/{uuid}/{uuid}.jpg`) provides permanent URLs without expiry concerns. However, this sacrifices access control
6. **Consider Supabase Storage's transform feature** for thumbnails. Generate thumbnails on the fly for feed display and serve full-resolution only when the user taps to view. This reduces bandwidth

**Detection:** Automated test that uploads an image, waits for the signed URL to expire, then loads the crew feed and verifies the image still renders (by regenerating the URL).

---

### Pitfall 19: Next.js App Router Caching Serves Stale Crew Feed Data

**Severity:** MEDIUM
**Affects phase:** Phase 1 (MVP Core) -- Crew Feed
**Confidence:** MEDIUM (Next.js caching behavior has changed across versions; verify with Next.js 15 docs)

**What goes wrong:** Next.js App Router aggressively caches server component output by default. The crew feed is rendered as a server component. User A submits a declaration. User B navigates to the crew feed. The cached version is served -- User A's declaration is missing. User B refreshes the page and still sees stale data because the router cache has not been invalidated.

**Why it happens:**
- Next.js has multiple caching layers: Data Cache, Full Route Cache, Router Cache (client-side)
- The Data Cache caches `fetch()` responses indefinitely by default in App Router
- The Router Cache caches prefetched and visited routes on the client for 30 seconds (dynamic pages) to 5 minutes (static pages)
- Supabase client calls using `@supabase/ssr` do not go through `fetch()` by default, so they may bypass the Data Cache but still be affected by the Router Cache
- Drizzle ORM queries bypass Next.js caching entirely (they use direct PostgreSQL connections), but the component output that renders the query result is still cached

**Specific DevReal risks:**
- Crew feed shows stale data -- user submits declaration but crewmates do not see it for 30+ seconds
- Leaderboard shows stale scores after a proof is approved
- User profile shows old streak count after a new day is completed
- The ship check countdown timer starts from a cached timestamp, showing incorrect remaining time

**Prevention:**
1. **Mark dynamic routes explicitly.** Add `export const dynamic = 'force-dynamic'` to the crew feed page layout. This opts out of full route caching:
```typescript
// app/crew/[crewId]/page.tsx
export const dynamic = 'force-dynamic';
```
2. **Use `revalidatePath` or `revalidateTag` after mutations.** When a declaration is submitted, call `revalidatePath('/crew/[crewId]')` in the server action to invalidate the cached feed
3. **For real-time data, use client components with Supabase Realtime** instead of server-cached data. The crew feed should be a client component that subscribes to realtime updates, with server-rendered initial data:
```typescript
// Server: Fetch initial data
const declarations = await db.query.declarations.findMany({ ... });
// Client: Subscribe to realtime updates
<CrewFeed initialData={declarations} crewId={crewId} />
```
4. **Avoid the Next.js Data Cache for Supabase queries.** If using `fetch()` with the Supabase REST API (instead of Drizzle), set `{ cache: 'no-store' }` or `{ next: { revalidate: 0 } }` on every request
5. **Use `router.refresh()` after client-side mutations** to force the router to re-fetch server components:
```typescript
import { useRouter } from 'next/navigation';
const router = useRouter();
await submitDeclaration(data);
router.refresh();
```

**Detection:** Manual testing: submit a declaration in one browser, check the crew feed in another browser within 5 seconds. If the declaration does not appear, caching is stale. Automate this with E2E tests that use two browser contexts.

---

## Minor Pitfalls

Mistakes that cause annoyance, minor UX issues, or easily fixable technical debt.

---

### Pitfall 20: Invite Link SEO and Social Preview Failures

**Severity:** LOW
**Affects phase:** Phase 1 (MVP Core) -- Crew Invites
**Confidence:** HIGH (standard web development issue)

**What goes wrong:** Users share crew invite links (`devreal.dev/crew/abc123`) on Twitter, Discord, and Slack. The link preview shows a generic "DevReal" title with no crew-specific information. The link looks suspicious or uninteresting. Conversion rate from link share to crew join is low.

**Prevention:**
1. **Generate dynamic OG metadata for invite pages** using Next.js `generateMetadata`:
```typescript
export async function generateMetadata({ params }: { params: { code: string } }) {
  const crew = await getCrew(params.code);
  return {
    title: `Join ${crew.name} on DevReal`,
    description: `${crew.memberCount} developers shipping daily. Join the crew.`,
    openGraph: {
      title: `Join ${crew.name} on DevReal`,
      description: `Ship with ${crew.memberCount} developers.`,
      images: [{ url: `/api/og/crew/${params.code}` }],
    },
  };
}
```
2. **Generate dynamic OG images** using `@vercel/og` (or `next/og`) that show the crew name, member count, and a visual of the crew's shipping activity
3. **Test link previews** using the Twitter Card Validator, Facebook Sharing Debugger, and Slack's unfurl preview before launch

**Detection:** Share an invite link in Slack and Twitter. If the preview is generic or missing, fix the metadata.

---

### Pitfall 21: Missing Database Indexes on High-Query Columns

**Severity:** LOW (initially) / MEDIUM (at scale)
**Affects phase:** Phase 1 (MVP Core) -- All queries
**Confidence:** HIGH (standard database optimization)

**What goes wrong:** Drizzle ORM generates correct SQL but does not automatically create indexes. Queries that are fast with 100 rows become slow with 100,000 rows. The crew feed query filtering by `crew_id` and ordering by `created_at` performs a sequential scan on the declarations table.

**Prevention:** Add indexes proactively in the Drizzle schema for known query patterns:
```typescript
// Indexes to add from day one
export const declarationsCrewIdx = index('declarations_crew_created_idx')
  .on(declarations.crewId, declarations.createdAt);

export const shipChecksCrewIdx = index('ship_checks_crew_submitted_idx')
  .on(shipChecks.crewId, shipChecks.submittedAt);

export const proofReviewsCheckIdx = index('proof_reviews_check_idx')
  .on(proofReviews.shipCheckId);

export const leagueStandingsWeekIdx = index('league_standings_week_league_idx')
  .on(leagueStandings.weekStart, leagueStandings.league, leagueStandings.weeklyScore);

export const dailyActivityUserDateIdx = index('daily_activity_user_date_idx')
  .on(dailyActivity.userId, dailyActivity.date);

export const nudgesCrewDateIdx = index('nudges_crew_date_idx')
  .on(nudges.crewId, nudges.createdAt);
```

**Detection:** Enable `pg_stat_user_tables` monitoring. If `seq_scan` count is high relative to `idx_scan` for any table with more than 10,000 rows, missing indexes are the cause.

---

### Pitfall 22: Crew Invite Code Collisions and Brute-Force

**Severity:** LOW
**Affects phase:** Phase 1 (MVP Core) -- Crew Creation
**Confidence:** HIGH (basic security consideration)

**What goes wrong:** The spec says crew invite codes are 6-character alphanumeric codes (`ABC123`). With 36 characters (A-Z, 0-9), there are 36^6 = ~2.2 billion possible codes. This is sufficient to avoid collisions. However, a 6-character code can be brute-forced: an attacker can try all 2.2 billion codes to discover and join private crews. At 100 requests/second, the entire space is covered in ~250 days. At 10,000 requests/second, 2.5 days.

**Prevention:**
1. **Rate limit the join-by-code endpoint** to 5 attempts per IP per minute. This makes brute-force infeasible (2.2B / 5 per minute = 835,000 years)
2. **Use a longer code or include mixed case.** 8-character alphanumeric (36^8 = ~2.8 trillion) or 6-character case-sensitive (62^6 = ~57 billion) provides more entropy
3. **Track failed join attempts.** If a code receives more than 20 failed attempts, temporarily disable it and alert the crew owner
4. **Require crew owner approval for code joins** as an optional setting. The invite link can auto-approve; the code requires a confirmation step

**Detection:** Monitor the rate of failed join-by-code attempts. Spikes indicate brute-force attempts.

---

### Pitfall 23: Soft Delete and Account Anonymization Complexity

**Severity:** LOW
**Affects phase:** Phase 3 (Growth) -- Account Deletion
**Confidence:** HIGH (GDPR/privacy compliance is well-understood)

**What goes wrong:** The spec says "Soft delete + anonymize for account deletion (30-day reversible grace period)." This sounds simple but touches every table: user rows, declarations, ship_checks, proof_reviews, reactions, nudges, league_standings, daily_activity, crew_members, and uploaded files in Supabase Storage. Missing any table means personal data persists after "deletion."

**Prevention:**
1. **Create a comprehensive deletion checklist** that maps every table containing user data. Maintain this checklist as tables are added
2. **Use a database function for anonymization** that handles all tables atomically:
```sql
CREATE OR REPLACE FUNCTION anonymize_user(p_user_id UUID) RETURNS void AS $$
BEGIN
  UPDATE users SET username = '[deleted]', display_name = '[deleted user]',
    avatar_url = NULL, email = NULL, github_id = NULL, deleted_at = NOW()
  WHERE id = p_user_id;
  -- Cascade to other tables...
  DELETE FROM notification_preferences WHERE user_id = p_user_id;
  -- Delete uploaded files via storage API (cannot be done in SQL)
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```
3. **Separate "soft delete" from "anonymize."** Soft delete = set `deleted_at`, hide from UI. Anonymize = strip PII. These happen at different times (immediately vs. after 30-day grace period)
4. **Do NOT delete league_standings or daily_activity rows** -- anonymize them. These contribute to crew history and aggregate statistics. Show as "[deleted user]" in historical views
5. **Delete Supabase Storage files** (proof screenshots) as part of the anonymization job. This requires a separate API call since storage is not in the database
6. **Test with a full user lifecycle:** Create user, join crew, submit declarations, upload proofs, get reviews, earn league standing, then delete. Verify no PII remains after anonymization

**Detection:** Quarterly audit query: `SELECT * FROM users WHERE deleted_at IS NOT NULL` then check all related tables for remaining PII.

---

## Phase-Specific Warnings

| Phase | Likely Pitfall | ID | Mitigation |
|-------|---------------|----|------------|
| Phase 1: MVP Core | RLS silent empty results | P1 | Write RLS policies first, test with multiple user roles |
| Phase 1: MVP Core | Timezone streak corruption | P3 | Store IANA timezone, compute dates in user TZ, test with UTC+12 and UTC-12 |
| Phase 1: MVP Core | Vote race condition | P4 | Use PostgreSQL function with FOR UPDATE lock |
| Phase 1: MVP Core | File upload vulnerabilities | P6 | Server-side magic byte validation, EXIF stripping, 2 MB limit |
| Phase 1: MVP Core | OAuth redirect on preview deployments | P10 | Separate dev GitHub App, test auth E2E |
| Phase 1: MVP Core | Auth ID type mismatch | P12 | UUID type consistently, auth trigger for user sync |
| Phase 1: MVP Core | Stale crew feed from caching | P19 | `force-dynamic`, Realtime for live updates |
| Phase 1: MVP Core | Cold start death spiral | P7 | Matchmaking over creation, solo mode has standalone value |
| Phase 2: Gamification | Realtime + RLS performance cliff | P2 | Use Broadcast channels, not postgres_changes for feeds |
| Phase 2: Gamification | GitHub webhook security | P5 | HMAC verification with timingSafeEqual |
| Phase 2: Gamification | Streak anxiety and burnout | P8 | Degrading streak, weekend mode, 4 freezes/month |
| Phase 2: Gamification | Scoring perverse incentives | P15 | Launch with 2 factors, add more after observing behavior |
| Phase 2: Gamification | Cron job limitations | P9 | Pre-schedule daily, dispatch per-minute, or use pg_cron |
| Phase 2: Gamification | League calculation performance | P17 | Pre-compute scores, partition standings table |
| Phase 2: Gamification | Auto-approve gaming | P16 | Minimum 2 reviews, extended timeout for small crews |
| Phase 3: Growth | Notification fatigue | P11 | Hard cap of 3 push/day, batch crew activity |
| Phase 3: Growth | Account deletion complexity | P23 | Comprehensive table checklist, separate soft delete from anonymize |

---

## Sources and Confidence Notes

**HIGH confidence items** (P1, P3, P4, P5, P6, P7, P8, P11, P13, P15, P16, P18, P20, P21, P22, P23):
- Based on well-established PostgreSQL behavior, web security standards, consumer app research, and game design principles
- These patterns are consistent across versions and unlikely to have changed

**MEDIUM confidence items** (P2, P9, P10, P12, P14, P17, P19):
- Based on training data as of mid-2025. These relate to specific Supabase Realtime behavior, Vercel cron limits, Next.js caching defaults, and Drizzle/Supabase type interactions that may have changed in recent releases
- **Recommendation:** Validate these against current documentation before implementing mitigations:
  - Supabase Realtime docs: https://supabase.com/docs/guides/realtime
  - Vercel Cron docs: https://vercel.com/docs/cron-jobs
  - Next.js 15 caching docs: https://nextjs.org/docs/app/building-your-application/caching
  - `@supabase/ssr` docs: https://supabase.com/docs/guides/auth/server-side
  - Drizzle ORM + Supabase: https://orm.drizzle.team/docs/get-started-postgresql#supabase

**Items that could NOT be verified** (WebSearch unavailable):
- Current Vercel cron job limits for free/Pro tier (P9) -- the specific numbers (1/day free, 1/minute Pro) are from training data and should be confirmed
- Current Next.js 15 default caching behavior (P19) -- Next.js caching defaults have changed across minor versions
- Current Supabase Realtime connection limits (P2) -- the 200 free / 500 Pro numbers should be confirmed
- Current `@supabase/ssr` initialization patterns (P10) -- the package has been updated since training data
