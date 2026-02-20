# Proof-of-Ship Verification Mechanisms

> Research for DevReal -- a BeReal-style accountability app for developers

## Table of Contents

1. [Automated Verification Methods](#1-automated-verification-methods)
2. [Manual/Social Verification](#2-manualsocial-verification)
3. [Hybrid Approaches](#3-hybrid-approaches)
4. [Handling Different Project Types](#4-handling-different-project-types)
5. [Anti-Gaming Measures](#5-anti-gaming-measures)
6. [Technical Implementation](#6-technical-implementation)
7. [Recommended Verification Architecture](#7-recommended-verification-architecture)

---

## 1. Automated Verification Methods

### 1.1 GitHub Integration

GitHub is the richest source of automated ship-proof for open source and many private projects.

**What can be detected:**

| Signal | API Endpoint / Webhook Event | Reliability | Notes |
|--------|------------------------------|-------------|-------|
| Commits pushed | `push` webhook / REST `GET /repos/{owner}/{repo}/commits` | High | Easiest signal but also easiest to game |
| PRs merged | `pull_request` webhook (action: `closed`, `merged: true`) | High | Stronger signal than raw commits -- implies review |
| Releases published | `release` webhook / REST `GET /repos/{owner}/{repo}/releases` | Very High | Strong ship signal -- implies a versioned artifact |
| Deploy status | `deployment_status` webhook | High | Direct evidence of deployment activity |
| CI checks passing | `check_run` / `check_suite` webhooks | Medium-High | Green builds indicate working code |
| Issues closed | `issues` webhook (action: `closed`) | Medium | Weaker signal, but useful for milestone tracking |
| Tags created | `create` webhook (ref_type: `tag`) | High | Often correlates with releases |

**Failure modes:**
- Commits can be trivially faked (empty commits, whitespace changes, commit bots like [TheSpeedX/commit-bot](https://github.com/TheSpeedX/commit-bot))
- Commit metadata can be spoofed -- anyone can set a git username/email to impersonate another contributor
- Private repos require elevated OAuth scopes, raising privacy concerns
- API rate limits can throttle polling-based detection
- Webhook delivery is not guaranteed (GitHub retries but can fail)

**Reliability ranking:** Releases > PRs merged > Deploy status > CI passing > Commits pushed

### 1.2 Deploy Verification

Verifying that something is actually live on the internet is one of the strongest proof-of-ship signals.

**Platform-specific approaches:**

| Platform | Method | API Endpoint | Notes |
|----------|--------|-------------|-------|
| Vercel | List deployments | `GET /v6/deployments` | Returns deployment state (READY, ERROR, BUILDING, etc.) |
| Vercel | Deploy hooks | `POST https://api.vercel.com/v1/integrations/deploy/{id}` | Trigger + verify deploys |
| Netlify | List site deploys | `GET /api/v1/sites/{site_id}/deploys` | Returns deploy status with `summary.status` field |
| Netlify | Build hooks | `POST https://api.netlify.com/build_hooks/{id}` | Trigger builds programmatically |
| Railway | Deployment API | REST API available | Less documented than Vercel/Netlify |

**URL liveness checking:**
- Simple HTTP HEAD/GET request to verify a URL returns 200
- Can compare response hashes over time to detect actual content changes (not just uptime)
- Useful for "prove your site is live" verification
- Limitation: static sites that haven't changed will still return 200

**Failure modes:**
- Sites can be deployed with no meaningful changes
- URL might be live but the new feature might not be visible
- DNS propagation delays can cause false negatives
- Sites behind auth walls can't be easily verified

### 1.3 CI/CD Integration

**GitHub Actions:**
- REST API: `GET /repos/{owner}/{repo}/actions/runs` -- lists workflow runs with status
- Webhook: `workflow_run` event fires on completion
- The workflow dispatch API now returns run IDs directly in responses
- Can verify successful builds, test passes, and deployments

**CircleCI:**
- Integrates with GitHub Checks API -- status visible on PRs
- API available for querying build status
- Can be combined with GitHub status checks for unified view

**General CI/CD signals:**
- Green builds are a moderate ship signal (code compiles, tests pass)
- Deploy pipelines completing are a stronger signal
- Can track build frequency as a proxy for development activity

### 1.4 Package Registry Detection

| Registry | API | Detection Method | Reliability |
|----------|-----|-----------------|-------------|
| npm | `GET https://registry.npmjs.org/{package}` | Check `time` field for latest publish date | Very High |
| PyPI | `GET https://pypi.org/pypi/{package}/json` | Check `releases` object for new versions | Very High |
| crates.io | `GET https://crates.io/api/v1/crates/{name}` | Check versions list for new entries | Very High |
| RubyGems | `GET https://rubygems.org/api/v1/versions/{gem}.json` | Check for new version entries | Very High |

**Strengths:**
- Package publishes are very strong ship signals -- you actually released something
- Public APIs, no auth required for public packages
- Immutable records (published versions can't be easily faked)

**Limitations:**
- Only applies to library/package authors
- PyPI lacks a search API (package name must be known)
- Not all projects produce publishable packages

### 1.5 App Store Detection

**Apple App Store Connect API:**
- Full REST API for querying app status
- Statuses include: "Prepare for Submission", "Waiting for Review", "In Review", "Ready for Sale"
- Supports webhooks for status change notifications
- Requires API key with appropriate permissions

**Google Play Developer API:**
- Publishing API supports checking release status across tracks (alpha, beta, production)
- Known limitation: API status can be unreliable during review -- status changes to "in progress" immediately on submission even if the app is pulled for review
- Requires service account with Google Play Console access

**Reliability:** Medium-High for detecting submissions; the actual "live on store" verification is the strongest signal but has delays due to review processes.

---

## 2. Manual/Social Verification

### 2.1 Peer Review by Crew Members

This is the core social verification mechanism. Crew members (3-8 people) vote on whether a submission counts as "shipped."

**How it works:**
1. Developer submits proof (link, screenshot, description)
2. Crew members review and vote (thumbs up/down)
3. Majority or unanimous approval = verified

**Design considerations:**
- **Threshold:** Majority (>50%) vs. supermajority (>66%) vs. unanimous
- **Time window:** Crew has 24-48 hours to review before auto-approval
- **Abstention:** Members who don't vote within the window default to approval (reduces friction) or are excluded from the count
- **Dispute:** If rejected, submitter can appeal or provide additional proof

**Precedent from existing platforms:**
- **Stack Overflow:** Community voting determines answer quality; reputation gates voting privileges
- **Wikipedia:** Peer editing and review with tiered trust levels
- **Reddit:** Upvote/downvote with karma systems
- **eBay/Amazon:** Buyer-seller review systems with verified purchase badges

### 2.2 Screenshot + Description Submission

**Implementation:**
- Developer uploads a screenshot showing their work
- Adds a text description of what was accomplished
- Crew reviews for plausibility

**Strengths:**
- Works for any project type (code, design, content)
- Low friction -- anyone can take a screenshot
- Visual proof is intuitive to review

**Weaknesses:**
- Screenshots can be fabricated or recycled
- No way to automatically verify screenshot authenticity
- Reviewing screenshots takes crew time and effort

**Mitigation:** Combine with metadata (timestamp, URL visited) or require screenshots to include identifiable elements (date, specific UI state).

### 2.3 Demo Video/GIF Proof

**Implementation:**
- Short video (15-60 seconds) or animated GIF showing the shipped feature
- Can be recorded via screen capture
- Uploaded directly or linked from external hosting

**Strengths:**
- Harder to fake than screenshots
- Shows functionality in action, not just appearance
- Engaging for crew members to review
- Creates a library of progress over time (motivating)

**Weaknesses:**
- Higher friction to create than screenshots
- Larger file sizes, storage costs
- Still possible to fake with screen recordings of other people's work

**Precedent:** Dribbble uses short video/GIF uploads for design showcases. Loom popularized async video for work communication.

### 2.4 Link Submission with Crew Validation

**Implementation:**
- Developer shares a URL (live site, PR, blog post, etc.)
- Crew members visit the link and confirm it's real
- System can auto-check that the URL is reachable

**Strengths:**
- Verifiable -- crew can click and see for themselves
- Combines automated (URL check) and manual (crew review) verification
- Works for many project types

**Weaknesses:**
- Links can break after verification
- Private/authenticated content can't be verified by everyone
- Crew may not have context to evaluate if the link represents real progress

### 2.5 How Existing Platforms Handle Verification

**Product Hunt:**
- Ship feature focused on pre-launch audience building rather than proof of shipping
- Product launches are community-voted but not technically verified
- Relies on social reputation -- fake launches get called out by the community

**Hacker News (Show HN):**
- No formal verification -- you post a link and the community evaluates
- Social accountability via public scrutiny
- Comments quickly identify fake or low-effort submissions
- Reputation is implicit (account age, karma)

**WIP.co:**
- Streak-based accountability -- log completed to-dos daily
- No automated verification; relies on honor system + community
- "Every post is a changelog entry" -- completed work, not plans
- Social pressure from the streak system itself drives honesty

**Makerlog:**
- Similar to WIP.co -- daily task logging with streaks
- Community-driven accountability
- Free alternative to WIP.co with similar mechanics
- GitHub-style contribution graphs for visual motivation

---

## 3. Hybrid Approaches

### 3.1 Automated Signals + Peer Confirmation

The most robust approach combines both automated and social verification.

**Tiered verification model:**

```
Level 1: Self-Reported (lowest trust)
  - Developer claims they shipped
  - No automated or peer verification
  - Useful for: closed-source work, non-code milestones

Level 2: Peer-Verified (medium trust)
  - Developer submits proof
  - Crew reviews and approves
  - Useful for: design work, content, private projects

Level 3: Auto-Verified (highest trust)
  - Automated signal detected (PR merged, deploy live, package published)
  - No crew review needed (but crew is notified)
  - Useful for: open source, public deploys, package publishes
```

**Recommended flow:**
1. Developer declares a goal with a verification type
2. System monitors for automated signals (if applicable)
3. If auto-signal detected, verification is instant
4. If no auto-signal, developer submits manual proof
5. Crew reviews and votes on manual submissions
6. Unverified claims decay trust score over time

### 3.2 Trust Scores

A dynamic trust score tracks each developer's verification history.

**Trust score factors:**
- Number of auto-verified completions (highest weight)
- Number of peer-verified completions (medium weight)
- Number of self-reported completions (low weight)
- Crew endorsement ratio (how often crew approves your submissions)
- Streak consistency (long streaks boost trust)
- Dispute history (disputes lower trust)

**Trust score effects:**
- High trust: self-reported claims are accepted with minimal review
- Medium trust: peer review required for most claims
- Low trust: additional proof required, crew scrutiny increased
- New users: start at medium trust, build up through verified completions

**Trust decay:**
- Trust scores should decay slowly over time if not maintained
- Extended inactivity resets trust toward the baseline
- Prevents gaming by building trust and then submitting low-quality proof

### 3.3 Handling Closed-Source Projects

Closed-source projects present the biggest verification challenge.

**Approaches:**
1. **Anonymized proof:** Developer can share redacted screenshots or videos showing progress without revealing proprietary code
2. **Trusted witness:** A colleague or manager who can vouch (like a reference check)
3. **Deploy verification:** Even if code is private, the deployed result is often public
4. **Milestone-based:** Focus on outcomes (site launched, feature demo'd) rather than code activity
5. **Internal tooling integration:** Companies could connect private repos via OAuth with appropriate scoping -- the system only checks activity volume, not content
6. **NDA-friendly proof templates:** Provide structured templates that allow proof without revealing sensitive details ("Merged 3 PRs this week related to the authentication module")

---

## 4. Handling Different Project Types

### 4.1 Open Source Projects

**Verification methods (easiest):**
- GitHub/GitLab webhook integration for commits, PRs, releases
- Package registry monitoring (npm, PyPI, crates.io)
- Deploy status from Vercel/Netlify/Railway
- CI/CD pipeline status from GitHub Actions

**Confidence level:** Very high -- all activity is public and auditable.

### 4.2 Closed Source / Startup Work

**Verification methods:**
- Deploy verification (check if a URL is live and has changed)
- App store submission status (via App Store Connect / Google Play APIs)
- Anonymized activity summaries (connected via OAuth, report counts not content)
- Peer review with crew members who may be co-workers
- Demo videos showing the shipped feature

**Confidence level:** Medium -- relies more on social trust and deploy checks.

### 4.3 Design Work

**Verification methods:**
- Figma API integration -- detect file changes, version history updates
  - Figma REST API supports webhooks for file change notifications
  - Can track version history and comment activity
  - Dev Mode integration available for handoff tracking
- Screenshot/video comparisons (before/after)
- Link to Figma file with crew validation
- Dribbble/Behance post links

**Confidence level:** Medium -- Figma API can automate some checks, but design "completion" is subjective.

### 4.4 Content Creation

**Verification methods:**
- Blog posts: URL check (is the post live?) + content hash to detect changes
- Videos: YouTube/Vimeo API to detect new uploads
- Courses: Platform-specific APIs (Udemy, Teachable) or URL verification
- Podcasts: RSS feed monitoring for new episodes
- Newsletter: Substack/Buttondown API or screenshot of sent email

**Confidence level:** Medium-High -- published content is publicly verifiable.

### 4.5 Non-Code Milestones

**Verification methods:**

| Milestone | Verification Approach |
|-----------|-----------------------|
| Landing page live | URL liveness check + content change detection |
| First user signup | Screenshot of analytics dashboard or user count |
| Revenue milestone | Screenshot of Stripe/payment dashboard (redacted) |
| User feedback collected | Screenshot of survey results or testimonials |
| Partnerships formed | Screenshot of agreement/communication (redacted) |
| Product Hunt launch | Link to PH page + crew verification |

**Confidence level:** Low-Medium -- mostly relies on self-reported proof with peer review.

---

## 5. Anti-Gaming Measures

### 5.1 Detecting Fake Commits

**The problem:** Tools like [commit-bot](https://github.com/TheSpeedX/commit-bot) exist specifically to generate fake GitHub contributions. Additionally, trivial changes (whitespace, empty commits, auto-generated files) can inflate activity metrics.

**Detection heuristics:**

| Heuristic | What It Catches | Implementation |
|-----------|----------------|----------------|
| Minimum lines changed | Empty/whitespace-only commits | Check diff stats via GitHub API |
| File type filtering | Auto-generated files (lock files, build artifacts) | Exclude known generated file patterns |
| Commit message analysis | Meaningless or auto-generated messages | NLP/pattern matching on commit messages |
| Commit timing patterns | Bot-like regular intervals | Statistical analysis of commit timestamps |
| Author verification | Spoofed commits | Require GPG-signed commits or match to authenticated user |
| Code complexity scoring | Trivial changes padded as meaningful | AST-level analysis (expensive, but possible) |

**Practical approach for DevReal:**
- Don't try to be a code quality tool -- that's a rabbit hole
- Focus on PRs merged (not raw commits) as the primary signal
- Require commit signature verification for highest trust tier
- Use crew review as the human check on automated signals
- Flag suspicious patterns (e.g., 50 commits in 1 minute) for crew review

### 5.2 Minimum Meaningful Contribution Thresholds

**Configurable per-goal thresholds:**
- "Ship a feature" = at least 1 PR merged with >10 lines changed
- "Deploy an update" = deployment detected + URL content change
- "Publish a package" = new version on registry
- "Write a blog post" = URL live with >500 words of new content

**Let crews set their own standards:**
- Some crews might be strict (only auto-verified counts)
- Some might be lenient (self-reported is fine)
- Crew culture should drive verification norms
- DevReal provides the tools; crews decide the rules

### 5.3 Crew-Based Dispute Resolution

**Dispute flow:**
1. Crew member flags a submission as suspicious
2. Submitter is notified and can provide additional proof
3. Crew votes on whether the submission is legitimate
4. If rejected, the submission doesn't count toward the goal
5. Repeated disputed submissions lower the submitter's trust score

**Escalation:**
- If a crew can't resolve a dispute, it can be escalated to DevReal moderators
- Moderators review the evidence and make a final call
- Repeat offenders can be flagged or restricted

### 5.4 Reputation Systems and Trust Decay

**Reputation model:**
```
reputation = (auto_verified * 3) + (peer_verified * 2) + (self_reported * 1) - (disputed * 5)
```

**Trust decay mechanics:**
- Trust score decays by 5% per week of inactivity
- Minimum floor at "new user" level (never goes to zero)
- Rebuilding trust after a dispute takes 2x the effort
- Crews can see each member's trust score and history

**Social incentives:**
- High-trust members get a visual badge/indicator
- Trust history is visible to potential crew members when joining
- Crews can set minimum trust requirements for membership

---

## 6. Technical Implementation

### 6.1 OAuth Scopes Needed

#### GitHub (Recommended: GitHub App, not OAuth App)

**Critical insight:** GitHub Apps support fine-grained, read-only permissions -- OAuth Apps do not. GitHub's OAuth scopes (`repo`, `public_repo`) grant both read AND write access. For DevReal, a GitHub App is strongly recommended.

**GitHub App permissions needed:**
| Permission | Access Level | Purpose |
|-----------|-------------|---------|
| Repository contents | Read-only | Detect commits, code changes |
| Pull requests | Read-only | Detect PR merges |
| Deployments | Read-only | Detect deploy activity |
| Actions | Read-only | Detect CI/CD workflow runs |
| Metadata | Read-only | Basic repo info (always granted) |

**OAuth App scopes (if GitHub App is not feasible):**
- `public_repo` -- read/write access to public repos (no read-only option exists)
- `repo` -- read/write access to all repos including private (if needed)
- This is a significant limitation -- users may be uncomfortable granting write access

#### GitLab

| Scope | Purpose |
|-------|---------|
| `read_repository` | Read-only access to repository content via Git-over-HTTP |
| `read_api` | Read-only access to the API (includes commits, MRs, pipelines) |

GitLab has better read-only scope granularity than GitHub OAuth Apps.

#### Bitbucket

| Scope | Purpose |
|-------|---------|
| `repository:read` | Read access to all repos the user can access |
| `pullrequest:read` | Read access to pull requests |
| `pipeline:read` | Read access to pipeline/CI results |

Bitbucket supports fine-grained read-only scopes, similar to GitLab.

### 6.2 Webhooks vs Polling

**Webhooks (recommended primary approach):**

| Aspect | Details |
|--------|---------|
| Latency | Real-time (seconds) |
| Rate limits | Don't count against API rate limits |
| Reliability | GitHub retries failed deliveries, but delivery is not guaranteed |
| Setup cost | Requires public HTTPS endpoint to receive POST requests |
| Maintenance | Need to handle webhook signature verification, retries, deduplication |
| Scaling | Each user's repos send webhooks to your server; can be high volume |

**Polling (recommended as fallback/sync mechanism):**

| Aspect | Details |
|--------|---------|
| Latency | Depends on poll interval (minutes to hours) |
| Rate limits | Counts against API limits (5,000/hr for authenticated GitHub requests) |
| Reliability | Deterministic -- you control when and what you fetch |
| Setup cost | Simple -- just make API calls on a schedule |
| Maintenance | Need to handle pagination, ETags for conditional requests |
| Scaling | Linear with number of users; can hit rate limits at scale |

**Recommended strategy:**
1. **Webhooks as primary** -- subscribe to relevant events (push, pull_request, release, deployment_status) for connected repos
2. **Periodic sync as backup** -- poll once every 6-12 hours using conditional requests (ETags) to catch missed webhooks
3. **On-demand check** -- when a user submits proof, immediately verify via API call

**GitHub best practices:**
- Use conditional requests (ETags, `If-None-Match` headers) -- 304 responses don't count against rate limits
- Make serial requests, not concurrent, when bulk-fetching
- Respect `retry-after` and `x-ratelimit-reset` headers
- Consider GitHub's GraphQL API for batching multiple queries

### 6.3 Rate Limits and API Considerations

| Platform | Rate Limit | Auth Method | Notes |
|----------|-----------|-------------|-------|
| GitHub REST | 5,000 req/hr (authenticated) | OAuth token or GitHub App installation token | Conditional requests (304) don't count |
| GitHub GraphQL | 5,000 points/hr | Same | More efficient for batched queries |
| GitLab | 2,000 req/min (authenticated) | OAuth token or personal access token | Higher limit than GitHub |
| Bitbucket | 1,000 req/hr | OAuth token | Lowest limit of the three |
| Vercel | 100 req/min | Bearer token | Moderate limit |
| Netlify | Varies by plan | OAuth token | Less documented limits |
| npm registry | No auth required for public | None | Generous limits for read-only |
| PyPI | No auth required for public | None | No search API available |

**Scaling strategies:**
- Cache API responses aggressively (GitHub's ETags help)
- Use webhooks to eliminate most polling needs
- Batch operations using GraphQL where available
- Queue and throttle API calls per user/platform
- Consider a dedicated token pool for high-volume operations

### 6.4 Privacy Concerns

**What data to access:**

| Data | Access? | Justification |
|------|---------|---------------|
| Commit hashes and timestamps | Yes | Verify activity timing and volume |
| Commit messages | Yes | Display in proof submissions |
| Diff stats (lines added/removed) | Yes | Determine if changes are meaningful |
| File names changed | Yes, but store minimally | Detect auto-generated vs. meaningful files |
| Actual code content | No | Not needed for verification; major privacy risk |
| PR titles and descriptions | Yes | Display in proof submissions |
| Deploy URLs and status | Yes | Verify deployments |
| Repository names | Yes | Display which repo activity relates to |
| Branch names | Minimal | Only needed for deploy branch detection |

**Privacy principles:**
1. **Minimum viable access:** Only request the scopes/permissions actually needed
2. **No code storage:** Never store source code; only store metadata (commit hashes, line counts, timestamps)
3. **User control:** Let users choose which repos to connect and what data to share
4. **Transparency:** Show users exactly what data the app has accessed
5. **Data retention:** Delete activity data after the goal period ends (configurable)
6. **Opt-in granularity:** Users can connect specific repos, not their entire account

**GitHub App advantage:** Users can select exactly which repositories the app can access during installation, providing built-in granularity that OAuth Apps lack.

---

## 7. Recommended Verification Architecture

### 7.1 Verification Flow

```
Developer declares goal
         |
         v
  [Goal has automated signal?]
         |
    Yes /   \ No
       /     \
      v       v
 Connect    Manual proof
 integration  submission
      |        |
      v        v
  Webhook/    Screenshot,
  API poll    video, link,
      |       description
      |        |
      v        v
 Auto-verify  Crew review
  (Level 3)   (Level 2)
      |        |
      v        v
   SHIPPED!   Vote result
              |
         Pass / Fail
         /       \
        v         v
    SHIPPED!   Dispute?
                  |
             Provide more
               proof
```

### 7.2 Verification Priority Matrix

| Project Type | Primary Verification | Secondary Verification | Fallback |
|-------------|---------------------|----------------------|----------|
| Open source app | GitHub PR merged + deploy detected | Auto-verify | Crew review |
| npm/PyPI package | Registry publish detected | Auto-verify | Crew review |
| Mobile app | App store submission status | Deploy API check | Screenshot + crew review |
| Closed source | Deploy URL change detected | Crew review | Self-report + trust score |
| Design project | Figma API activity | Screenshot comparison | Crew review |
| Blog/content | URL liveness + content check | Link submission | Crew review |
| Revenue milestone | N/A | Screenshot of dashboard | Crew review |

### 7.3 MVP Recommendation

For the initial release, prioritize these verification methods:

**Phase 1 (MVP):**
1. Link submission with crew review (works for everything)
2. Screenshot/video upload with crew voting
3. GitHub App integration for auto-detecting PR merges and releases

**Phase 2:**
4. Deploy verification (Vercel/Netlify URL checks)
5. npm/PyPI package publish detection
6. Trust scores based on verification history

**Phase 3:**
7. GitLab and Bitbucket integration
8. Figma API for design work
9. App store submission detection
10. Advanced anti-gaming heuristics

This phased approach starts with the simplest universal method (crew review), adds the highest-value automation (GitHub), and progressively expands to more platforms and project types.

---

## Sources

- [GitHub Webhook Events and Payloads](https://docs.github.com/en/webhooks/webhook-events-and-payloads)
- [GitHub OAuth Scopes](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps)
- [GitHub Apps vs OAuth Apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)
- [GitHub REST API Best Practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api)
- [GitHub App Permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app)
- [Vercel Deployments API](https://vercel.com/docs/rest-api/endpoints/deployments)
- [Vercel Deploy Hooks](https://vercel.com/docs/deploy-hooks)
- [Netlify Deploy Hooks](https://www.contentful.com/developers/docs/tutorials/general/automate-site-builds-with-webhooks/)
- [npm Registry API](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md)
- [Apple App Store Connect API](https://developer.apple.com/documentation/appstoreconnectapi)
- [Google Play Developer API](https://developers.google.com/android-publisher)
- [Bitbucket Cloud OAuth Scopes](https://developer.atlassian.com/cloud/bitbucket/bitbucket-cloud-rest-api-scopes/)
- [GitLab OAuth2 API](https://docs.gitlab.com/api/oauth2/)
- [Figma REST API](https://developers.figma.com/docs/rest-api/)
- [RepoSense Contribution Analysis](https://github.com/reposense/RepoSense)
- [WIP.co](https://wip.co/help)
- [Reputation Systems (Wikipedia)](https://en.wikipedia.org/wiki/Reputation_system)
- [Fake GitHub Commits and Security](https://www.cybersecuritydive.com/news/github-commits-malicious-code/627466/)
- [Unverified Commits Security Risks](https://checkmarx.com/blog/unverified-commits-are-you-unknowingly-trusting-attackers-code/)
- [GitHub App vs OAuth -- Nango Blog](https://nango.dev/blog/github-app-vs-github-oauth)
