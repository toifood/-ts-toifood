ISSUE LOG - PRICE
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ISSUE ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ISSUE ENTRIES.

REQUIRED FORMAT FOR EACH ISSUE ENTRY:

## ISSUE:price {YYYY-MM-DD HH:MM} â†’ {CONTENT}


CUSTOM PROMPT:
Billing edge cases, subscription state mismatches, failed charge handling

PATHS:
would/

####### <!-- ANCHOR MARKER - ADD ALL NEW ISSUE ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ISSUE ENTRIES-->
## ISSUE:backend 2026-06-20 11:39 -> No payment provider still — plus rate-limit hit rate remains unmeasured and Redis persistence gap confirmed

**Status since June 13th:**

1. **No payment provider (unchanged)**: `UserRole.premium` is still set by manual DB update or admin endpoint only. No Apple IAP validation, no Google Play billing, no Stripe. This has been the case since `price-issue-v1.md` (2026-05-07). No code in `src/services/appstore.ts` or `src/services/playstore.ts` touches `User.role` — those services only read metrics. Subscription state and role elevation are entirely disconnected.

2. **Rate-limit hit rate still unmeasured**: The `recipeGenerateRateLimit()` middleware returns 429 but writes no CSV row, no Redis metric, and sends no Chat/Slack alert. The only signal is in PM2 logs via the global request logger. No way to know: (a) how often free users exhaust limits, (b) whether the free limits (3 ollama/hr, 2 claude/hr) are driving churn, or (c) whether premium users ever exhaust limits.

3. **Redis persistence gap confirmed**: Rate-limit counters reset on any Redis restart. The `dump.rdb` file in the repo root suggests Redis `SAVE` is configured, but `dump.rdb` committed to the repo indicates the file is in the working directory (not a separate data volume). If the repo is cloned fresh or the process restarted, the `dump.rdb` may be stale or absent.

4. **Claude cost visibility**: Claude usage is rate-limited but there is no metric on Anthropic API spend. Token counts are not logged in RECIPE-METRIC.csv. Unexpected Claude usage spikes (e.g., if admin generates hundreds of recipes) produce no cost alert.
## ISSUE:price 2026-06-19 16:46 → Rate limits are Redis-ephemeral and the tier model has no automated billing enforcement

**Current model:** `free` = 3 Ollama / 2 Claude per hour; `premium` = 10 Ollama / 5 Claude per hour; `admin` = unlimited. Limits stored in Redis with 1-hour TTL keys.

**Issues:**

1. **Redis restart resets all counters.** Redis runs locally with no persistence configured (no RDB/AOF mentioned). If Redis restarts, all hourly rate-limit counts reset to zero — users can exhaust limits, trigger a restart, and repeat. For current scale this is acceptable risk, but worth noting.

2. **Claude limits are very tight (free: 2/hr, premium: 5/hr).** Claude handles dietary filters and continent preferences (Ollama does not). A 2/hr cap makes the higher-quality provider barely usable for free users. Premium at 5/hr is also modest if Claude is the main premium differentiator.

3. **No billing integration.** `User.role` is set manually by an admin. There is no Stripe, RevenueCat, or similar integration. Upgrading to premium requires out-of-band action, limiting monetisation automation.

4. **No per-user cost attribution.** Redis tracks call counts, not cost. If a small number of premium users dominate Claude usage, there is no alerting mechanism. `claude-haiku-4-5-20251001` costs ~$0.80/MTok input, ~$4.00/MTok output — at current scale (~$0.0003/call) this is negligible, but not tracked.

5. **Rate limit usage counter survives client disconnect.** A user's Redis counter increments even if the Claude/Ollama call later fails (fallback counts against Ollama quota too? — no, fallback uses Ollama key). Claude failure → Ollama fallback counts against Ollama quota, not Claude, which is correct.

**Recommendation:** Add Redis persistence (`appendonly yes`). Consider surfacing remaining Claude generations prominently in the app UI as a premium conversion nudge.
## ISSUE:price 2026-06-19 16:05 → Rate limit window is hourly not daily; client shows CLAUDE_DAILY_LIMIT=3 while server enforces 2 per hour creating a limit mismatch

The Redis rate limiter in `src/middleware/rateLimit.ts` sets TTL to `60 * 60` seconds (1 hour) per provider key (`ratelimit:{userId}:ollama`, `ratelimit:{userId}:claude`). Limits are 3 Ollama + 2 Claude for free users, resetting every hour. The mobile client's `src/utils/claudeLimit.ts` (referenced in `src/routes/recipes.ts` context and visible in PRICE-ISSUE entries) defines `CLAUDE_DAILY_LIMIT = 3` — a different cap on a different time window. A free user gets 2 Claude generations per hour from the server but the app UI signals a daily cap of 3. After waiting 1 hour, the server counter resets and they get another 2. There is no daily ceiling server-side. The premium tier's `{ claude: 5 }` per-hour limit is likewise never bounded at a daily level, making worst-case Anthropic cost theoretically unbounded within a 24-hour period.
## ISSUE:price 2026-06-19 14:28 → Claude daily limit is enforced client-side in SecureStore only — bypassable by reinstall; no IAP SDK present

Two billing gaps: (1) `src/utils/claudeLimit.ts` enforces `CLAUDE_DAILY_LIMIT = 3` by reading/writing to `expo-secure-store` key `'claude_daily_limit'`. Clearing app data, reinstalling, or running on an emulator with a fresh keychain gives any user unlimited Claude generations. The backend's server-side rate limiter (`getRecipeUsage`) is the real guard, but the client-side counter creates a false sense that the limit is enforced at the app layer. (2) `package.json` has no IAP SDK (`expo-in-app-purchases`, `react-native-purchases`, or `react-native-iap`). `app/partials/PremiumModal.tsx` exists in the file tree, implying a premium upsell flow, but there is no payment processing capability in the current mobile codebase. Premium upgrades cannot be completed inside the app.
## ISSUE:price 2026-06-15 09:12 → Claude premium tier is bypassable — any free user can request Claude by sending provider:"claude" in the request body

`src/routes/recipes.ts` line 235 computes `isPremium = user?.role !== "free"` but never uses this value before line 253: `if (provider === "claude" && process.env.ANTHROPIC_API_KEY)`. The only gate is whether the env var is set and whether the client sends `"provider": "claude"` in the JSON body. A free-tier user who calls `POST /recipes/generate` with `{"provider": "claude", ...}` directly against the API gets a full Claude Haiku generation, incurring Anthropic API cost, with no role check. The `isPremium` variable is assigned but dead — it is never referenced in a conditional guard on the Claude branch. This is an API-level billing bypass, not just a UI issue.
## ISSUE:price 2026-06-14 23:03 → add per-call cost tracking for Claude provider in metrics CSV

Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) is used for recipe generation. A typical call is ~200 tokens in + ~300 tokens out ≈ $0.0003/call (input $0.0008/1K, output $0.004/1K). No cost is currently captured in `logs/recipe-metrics.csv`.

Add `costUsd` column to `appendMetric()` in `src/routes/recipes.ts:844`. For Ollama calls, `costUsd = 0`. For Claude, derive from token counts in the Anthropic response `usage` field (already available on the response object — just not captured).

At 1,000 Claude generates/day the monthly cost would be ~$9 — low now, but knowing usage trends enables the premium tier pricing decision.
## ISSUE:price 2026-06-13 18:11 → No subscription webhook handling; role upgrades to premium have no automated trigger

The backend has a `UserRole` enum (`free`, `premium`, `admin`) with role-tiered rate limits, but there is no code path that upgrades a user from `free` to `premium` in response to a payment event. `src/services/appstore.ts` and `src/services/playstore.ts` only fetch analytics metrics — neither handles StoreKit 2 server notifications or Google Play Real-Time Developer Notifications. A user who pays via the App Store would remain on the `free` rate limit indefinitely. There is no `roleUpdatedAt` field, no log event on role change, and no idempotency guard, making manual upgrades unauditable and risky to double-apply.
## ISSUE:price 2026-06-13 17:04 → Claude spend untracked; prompt caching unused; OpenAI provider has no rate limiting

**1. No token/cost logging.** `RECIPE-METRIC.csv` records `usedProvider=claude` and `responseMs` but not input/output token counts. The Anthropic API returns `usage.input_tokens` and `usage.output_tokens` in every response (`src/services/ai/claude.ts:85`) but these are discarded. Without token logging there is no way to reconcile App Store revenue against AI spend as user count grows.

**2. Prompt caching unused.** `src/services/ai/claude.ts:63-74` sends the full request with no `cache_control` headers. The system prompt `"You are a professional chef. Always respond with valid JSON only."` is static on every call and eligible for prompt caching. At Haiku 4.5 pricing, cached input tokens cost 90% less. Adding `cache_control: { type: "ephemeral" }` to the system block requires the `anthropic-beta: prompt-caching-2024-07-31` header.

**3. OpenAI provider bypasses rate limiting.** `src/services/ai/openai.ts` uses `gpt-4o` but is not wired into `recipeGenerateRateLimit()` (`src/routes/recipes.ts:176`). The rate-limit middleware only checks `provider === 'claude'` vs ollama; a request with `provider=openai` skips the limit entirely. OpenAI is also not guarded by an API key check at the middleware layer.

**4. Rate limits have no daily cap.** The hourly Redis key (`ratelimit:{userId}:claude`, 3600s TTL) resets every hour. A premium user can make 5×24=120 Claude calls/day with no daily ceiling.
## ISSUE:price 2026-06-09 18:16 â†’ digest.ts and slack-bot.ts are unmetered auxiliary processes; heavy DB aggregation queries in digest compete with user traffic on single-node PostgreSQL; no read-replica strategy

**Auxiliary processes outside cost and query budgets:**
- `src/digest.ts` is a separate Node.js process that runs on a schedule (likely cron or setInterval). Its queries are not metered, rate-limited, or accounted for in any cost tracking. If it computes aggregate stats (`GROUP BY`, `COUNT(*)` on recipes, cook records, or users), it performs full table scans on the same single-node PostgreSQL instance serving user-facing API requests. No read-replica strategy exists.
- `src/slack-bot.ts` is another separate process that likely responds to Slack commands. If any Slack command triggers an AI provider call (e.g., "generate a test recipe"), that call is outside the rate-limit system entirely â€” no `ratelimit:{userId}:{provider}` key is checked.
- Neither process has documented query budgets, timeouts, or connection pool sizing. Three Node.js processes (main API + digest + slack-bot) share the same PostgreSQL instance with default Prisma connection pools â€” total connection count is `3 Ã— pool_size` with no configured limit.

**Token cost of digest AI calls:**
- If `digest.ts` generates a periodic Slack summary using an AI provider (e.g., summarising recipe trends), it uses Claude/OpenAI tokens with no `max_tokens` cap and no logging of token usage. These calls are invisible to the cost tracking system.

**Action needed:** Document the digest.ts query schedule and SQL profile. Set explicit Prisma `connection_limit` per process. If digest makes AI calls, add token usage logging. Gate slack-bot AI commands behind the same rate-limit middleware as the API.

## ISSUE:price 2026-06-09 18:03 â†’ Recipe save triggers up to NÃ—AI insight calls (one per category) outside rate limit scope; no per-call cost cap on OpenAI/Claude; store metrics polling unbounded

**Cost multiplication via insight pipeline:**
- Each recipe save can trigger `runInsightAnalysis()` which iterates over ALL insight categories for the user. If a user has 5 active categories, one recipe save = 5 AI provider calls. These calls are not counted against the per-user rate limit (which tracks `ratelimit:{userId}:{provider}` per recipe request, not per insight call). A user who generates 3 recipes/hr (at their free tier limit) could actually trigger 15+ AI calls per hour.

**No max_tokens cap on recipe generation prompts:**
- Neither the Claude provider (`src/services/ai/claude.ts`) nor the OpenAI provider include a `max_tokens` constraint in their API calls. A malformed or adversarial prompt could generate an unusually large completion, driving up token cost with no ceiling. Claude API allows up to 200K output tokens on some models.

**AppStore/PlayStore polling frequency uncontrolled:**
- `GET /store-metrics` is admin-only but calls live external APIs (App Store Connect, Google Play Developer Reporting) on each request. If an admin dashboard polls this endpoint frequently, it will exhaust Google's daily quota and trigger rate limits on Apple's API â€” neither of which alerts on failure.

**YouTube API quota invisible:**
- `findRecipeVideo()` runs on every recipe save. Google YouTube Data API v3 has a 10,000 unit daily quota. A day with 200+ recipe saves will exhaust the quota silently; subsequent recipes return no video with only a logged warning.

**Action needed:** Gate insight generation behind a job queue or debounce. Add `max_tokens` to all AI provider calls. Rate-limit the `/store-metrics` admin endpoint. Add YouTube quota exhaustion alerting.
## ISSUE:price 2026-06-07 16:30 â†’ Insight AI calls unbilled and unmetered; openai provider in recipe generation has no cost cap; Redis failure silently removes all rate limits

**Unmetered AI cost paths:**
- `runInsightAnalysis()` in `src/services/ai/insights.ts` calls the AI provider (Ollama/Claude/OpenAI) to generate insights for users. This runs server-side outside any rate limit middleware â€” no hourly cap, no per-user throttle. If the insights endpoint is triggered frequently, AI costs accumulate silently.
- The YouTube video search (`findRecipeVideo()`) runs on every recipe save â€” Google API quota is untracked.

**Rate limit bypass risk:**
- `rateLimit.ts` catches Redis errors and silently skips rate limiting (`console.warn` only). If Redis goes down, all users â€” including free tier â€” can generate unlimited recipes, directly driving OpenAI/Claude API costs.
- The `admin` role completely bypasses rate limits but there is no audit log of admin recipe generation volume.

**Pricing model gaps:**
- `premium` role is stored in DB but there is no subscription management, payment webhook, or Stripe integration. Role can only be set via direct DB update or admin API â€” no automated downgrade on payment failure.
- No metering of token usage per AI call â€” cost per recipe is unknown and untracked.
## ISSUE:price 2026-06-07 10:00 â†’ Claude/OpenAI API costs invisible; Redis outage silently disables rate limits; no billing integration for premium role

**Cost visibility gaps:**
- Claude provider (`src/services/ai/claude.ts`) has no usage logging â€” API call cost is invisible. No token counts, no per-user spend tracking in DB.
- OpenAI provider same issue â€” no usage logged, no cost alerting.
- Ollama is free (local), but Claude + OpenAI incur real API costs per recipe generation.

**Rate limit bypass risk:**
- When Redis is unavailable, `recipeGenerateRateLimit()` silently calls `next()` â€” rate limits are fully disabled. A Redis outage means unbounded Claude/OpenAI API calls, which could incur significant unexpected cost.
- No Slack alert is fired when Redis goes down and rate limiting is skipped.

**Premium tier without billing:**
- `UserRole` enum (free/premium/admin) exists and controls limits, but there is no payment processing, subscription management, or billing webhook in the codebase. The premium role can only be set by direct DB edit or admin tooling not visible in this repo.
- No endpoint to upgrade a user to premium â€” limits are enforced but there's no way for users to pay and self-upgrade.

**Action needed:** Add usage logging (tokens/cost) to Claude and OpenAI providers. Add Slack alert when Redis is unavailable and rate limiting falls back. Implement billing integration or document the manual upgrade process.

Pricing information exists only inside a single FAQ entry (`id: 'premium'` and `id: 'generation-limit'`). The Home page has no plan comparison, no premium CTA, and no mention of what premium unlocks. The SharedRecipe page labels recipes as `Premium` or `Basic` based on `recipe.provider === 'claude'` but provides no link or prompt to upgrade. Users arriving via shared recipe links — likely the highest-intent audience — have no in-page path to learn about or purchase premium.
