ASSET LOG - PRICE
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ASSET ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES.

REQUIRED FORMAT FOR EACH ASSET ENTRY:

## ASSET:price {YYYY-MM-DD HH:MM} â†’ {CONTENT}


CUSTOM PROMPT:
Pricing logic correctness, audit trails, webhook idempotency

PATHS:
would/

####### <!-- ANCHOR MARKER - ADD ALL NEW ASSET ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES-->
## ASSET:price 2026-06-30 06:41 → tier entitlements in one constant; per-provider per-user Redis keys are audit-friendly; admin bypass prevents quota exhaustion

**Finding — `src/middleware/rateLimit.ts` `LIMITS` constant**
All tier entitlements live in one object (`LIMITS = { free, premium, admin }`), which is the single source of truth for what each subscription tier can do per hour and per provider. Adding a new tier or adjusting a cap is a one-line change in one file with no scattered conditionals to update.

**Finding — `ratelimit:{userId}:{provider}` key pattern**
The Redis key includes both user ID and provider (ollama/claude) as segments. This means usage per-tier and per-model can be independently inspected (`KEYS ratelimit:*:claude`) for billing audits without a DB query, and a provider outage can be mitigated by resetting only the affected key segment.

**Finding — admin bypass in rate limiter**
Admins skip rate limiting entirely via an early return after role check, so internal testing and support work cannot accidentally exhaust a production quota or produce misleading usage telemetry.
## ASSET:price 2026-06-29 12:37 → Shared authLimiter instance prevents config drift; duplicate-save guard on recipe creation; appstore JWT correctly regenerated per call with 20m expiry

**Finding — `src/routes/auth.ts` `authLimiter`**
A single `rateLimit` instance (`windowMs: 15min, max: 10`) is shared across register, login, Apple auth, forgot-password, and resend-verification. Using one shared instance means the limit configuration cannot drift between endpoints — a change to the rate window applies everywhere simultaneously. `standardHeaders: true` correctly surfaces `RateLimit-*` headers per RFC 9110.

**Finding — `src/routes/recipes.ts` POST / duplicate guard**
Before creating a recipe, the save handler queries for an existing recipe with the same title created by the same user within the past 24 hours (`createdAt: { gte: oneDayAgo }`). The duplicate is logged (`[recipe:save] duplicate title=...`) but not rejected — the save proceeds. This provides an audit trail for double-tap saves (e.g. network retry) without blocking legitimate same-title saves across separate days.

**Finding — `src/services/appstore.ts` `makeToken()`**
`APPSTORE_PRIVATE_KEY!.replace(/\\n/g, "\n")` normalises the PEM key whether it was stored with literal `\n` escape sequences (common in `.env` files) or with real newlines. A fresh 20-minute JWT is signed per metrics call, matching Apple's maximum token lifetime requirement. No token is reused or cached across requests, eliminating the risk of serving requests with an expired App Store Connect token.
## ASSET:price 2026-06-29 12:28 → SharedRecipe.jsx surfaces Premium/Basic tier label and premium author badge as passive upgrade signals

**Finding — `frontend/src/pages/SharedRecipe.jsx` lines 421–427**
The recipe meta grid displays `recipe.provider === "claude" ? "Premium" : "Basic"` alongside a sparkles icon. This communicates the two-tier model to cold visitors arriving via shared links without a dedicated pricing page.

**Finding — author card (lines 678–684)**
Premium accounts display a lime sparkle badge on the author avatar, gated on `isPremium` derived from `fullProfile?.role ?? author?.role !== "free"`. Both signals reinforce status differentiation and serve as passive marketing for the paid tier.
## ASSET:price 2026-06-21 19:41 → AppStore JWT uses short-lived 20m tokens and admin gate protects store metrics endpoint

Two correct pricing/billing implementation details:

**1. appstore.ts — JWT uses 20m expiry as required by Apple**
makeToken() in src/services/appstore.ts signs with expiresIn: "20m". Apple's App Store Connect API requires tokens expire within 20 minutes; longer-lived tokens are rejected. The implementation is correct and the token is freshly generated per metrics call inside getAppStoreMetrics().

**2. storeMetrics.ts — admin guard prevents non-admin billing data exposure**
The GET / handler applies both requireAuth and requireAdmin middleware. Store metrics (crash rates, install counts, active device counts) are only accessible to admin-role users, correctly preventing regular or premium users from reading operational and commercial data.
## ASSET:backend 2026-06-20 11:39 -> Pricing infrastructure snapshot — unchanged limits, no billing integration, Redis dump in repo confirmed

**Rate limit table (current, unchanged since June 13th):**

| Role | `ollama`/hr | `claude`/hr |
|---|---|---|
| `free` | 3 | 2 |
| `premium` | 10 | 5 |
| `admin` | bypass | bypass |

**Role elevation paths (current):**

| Path | Mechanism |
|---|---|
| `free` → `premium` | Manual DB update by admin or `PATCH /admin/users/:id/role` (if such endpoint exists) |
| `free` → `admin` | Manual DB update only |
| Apple IAP | Not implemented — `appstore.ts` reads metrics only, no receipt validation |
| Google Play billing | Not implemented — `playstore.ts` reads crash/ANR rates only, no subscription API |

**Redis state for rate limiting:**
- Key pattern: `ratelimit:{userId}:{provider}` (TTL: 3600s)
- Insight cooldown: `insights:cooldown:{userId}` (TTL: 604800s / 1 week)
- `dump.rdb` present in repo root — Redis SAVE configured but persistence to repo working directory is not isolated from source code
- No Redis persistence confirmation on restore: counters reset to 0 on restart (free hourly reset for all users)

**Usage endpoint:**
`GET /1-1-1/api/recipes/usage` — returns `{ ollama: { used, max, ttl }, claude: { used, max, ttl } }` for the authenticated user. No admin aggregate view of system-wide usage.
## ASSET:price 2026-06-19 16:46 → Rate limit configuration and cost reference

**Rate limit table** (`src/middleware/rateLimit.ts`):

| Role | Ollama/hr | Claude/hr |
|---|---|---|
| `free` | 3 | 2 |
| `premium` | 10 | 5 |
| `admin` | 999 | 999 |

**Redis key pattern:** `ratelimit:{userId}:{provider}` (TTL: 3600s, set on first increment)

**AI model cost reference (2026Q2 pricing):**
- `claude-haiku-4-5-20251001`: ~$0.80/MTok input, ~$4.00/MTok output
- Typical recipe prompt: ~400–600 input tokens, ~300–500 output tokens
- Estimated cost per Claude call: ~$0.0002–$0.0005
- `qwen2.5:7b` via Ollama: self-hosted, compute cost only (Mac mini M4, no cloud spend)

**Upgrade path (manual):** Admin sets `User.role = 'premium'` directly via DB or admin route. No automated billing flow exists.

**Usage/cost logging:** `recipe-metrics.csv` logs `requestedProvider`, `usedProvider`, `fallback` per generation. Daily digest summarises counts by model. No per-user cost aggregation exists.

**Fallback cost note:** Claude failure → Ollama fallback is logged as `fallback: true` in CSV. The Claude API call still incurs input token cost even on failure (if the request reached Anthropic before erroring).

**`GET /recipes/usage` response shape:**
```json
{ "ollama": { "used": 1, "max": 3, "ttl": 3412 }, "claude": { "used": 0, "max": 2, "ttl": 0 } }
```
## ASSET:price 2026-06-19 16:05 → Redis rate limit correctly uses increment-then-expire pattern; fail-open on Redis outage prevents generation downtime

The rate limit implementation in `src/middleware/rateLimit.ts` uses `redis.incr(key)` followed by `redis.expire(key, 3600)` only when `count === 1`, which is the correct pattern to avoid resetting the TTL on every request. If `expire` is called on every increment, a sustained burst would never expire — the implementation avoids this correctly. When Redis is unavailable, the catch block logs a warning and calls `next()` (fail-open), ensuring a Redis outage never blocks recipe generation for users. The `getRecipeUsage()` export returns `{ used: 0, max: LIMITS.free.ollama, ttl: 0 }` on Redis failure, giving the client a graceful degraded state rather than an error response.
## ASSET:price 2026-06-19 14:28 → bannerUsage from recipeAPI.getUsage() provides server-validated quota display independent of client-side SecureStore counter

`UserDataContext` fetches `bannerUsage` via `recipeAPI.getUsage()` (`GET /recipes/usage`) which returns `{ ollama: { used, max, ttl }, claude: { used, max, ttl } }` from the backend's Redis rate-limit state. This is the authoritative quota source — it cannot be bypassed by reinstall. The client-side `claudeLimit.ts` counter in `AnnouncementBanner` serves as a fast local read, but `bannerUsage` provides the correct fallback for any discrepancy. `user.role` (free/premium/admin) is also returned by the backend and used throughout the UI for premium gating, keeping role state server-authoritative.
## ASSET:price 2026-06-15 09:12 → recipe-metrics.csv provides per-request cost attribution with requestedProvider, usedProvider, and fallback fields

`src/routes/recipes.ts` calls `appendMetric()` for every generation, writing `requestedProvider`, `usedProvider`, `fallback`, and `responseMs` to `logs/recipe-metrics.csv`. This means every Claude call is auditable: if `requestedProvider=claude` and `usedProvider=ollama` and `fallback=true`, the Anthropic call failed and incurred no cost. If `usedProvider=claude` and `fallback=false`, cost was incurred. The `promptVersion` field (`claude-v4`, `ollama-v4`) tracks which prompt template was used, enabling per-version cost regression analysis. The `getRecipeUsage()` middleware in `src/middleware/rateLimit.ts` enforces per-user daily generation caps, limiting worst-case billing exposure.
## ASSET:price 2026-06-14 23:03 → Claude Haiku 4.5 pricing reference and cost projection

Model: `claude-haiku-4-5-20251001`
- Input: $0.80 / 1M tokens ($0.0008 / 1K)
- Output: $4.00 / 1M tokens ($0.004 / 1K)
- Typical recipe call: ~200 in + ~300 out = ~$0.0003/call
- Prompt cache not currently used (would reduce input cost by 90% on repeated system prompt)

Ollama (`qwen2.5:7b`) cost: $0 (local compute). The fallback path (Claude → Ollama) already de-risks cost spikes. Free tier users default to Ollama; premium users request Claude via `provider=claude` param (`src/routes/recipes.ts:930`).
## ASSET:price 2026-06-13 18:11 → Role-tiered rate limiting correctly implemented with Redis atomicity

Rate limits are cleanly tiered by role in `src/middleware/rateLimit.ts`: `free` (3 ollama / 2 claude per hour), `premium` (10 / 5), `admin` (unlimited). A Redis Lua script (`INCR` + `EXPIRE` in one atomic call) prevents the race condition where two concurrent requests both see count=1 and the key never gets an expiry set. `getRecipeUsage()` exposes current used/max/TTL to clients so the frontend can display accurate quota status. The `isPremium` and `isAdmin` flags in `/users/me` are derived cleanly from `role` without separate boolean columns.
## ASSET:price 2026-06-13 17:04 → Claude call site, rate-limit config, and token response surface

**Model:** `claude-haiku-4-5-20251001` — hardcoded at `src/services/ai/claude.ts:59`

**Prompt version:** `claude-v4` (`src/services/ai/claude.ts:3`)

**max_tokens:** 1024 per call (`src/services/ai/claude.ts:60`)

**Timeout:** 30s AbortSignal (`src/services/ai/claude.ts:74`)

**Rate-limit config** (`src/middleware/rateLimit.ts:18-21`):
```
free:    { ollama: 3, claude: 2 }   // per hour
premium: { ollama: 10, claude: 5 }
admin:   { ollama: 999, claude: 999 }
```
Redis keys: `ratelimit:{userId}:claude`, `ratelimit:{userId}:ollama` — 1hr TTL

**Token usage surface:** `src/services/ai/claude.ts:85` — `data` from `response.json()` includes `usage` field; currently unused

**Caching opportunity:** add `anthropic-beta: prompt-caching-2024-07-31` header and wrap system string in `{ type: 'text', text: '...', cache_control: { type: 'ephemeral' } }`
## ASSET:price 2026-06-09 18:16 â†’ Provider isolation via AI_PROVIDER env var prevents accidental paid-tier usage in dev; AppStore ES256 JWT is short-lived; role-tier limits are in-memory constants with no DB round-trip

**Provider isolation at runtime:**
- `AI_PROVIDER` env var selects the backend at process startup â€” switching from Ollama to Claude or OpenAI requires an explicit env change and pm2 restart, not an API call. This means dev environments running without `AI_PROVIDER` set default to free Ollama automatically, preventing accidental charged API usage in development or test runs.

**No DB round-trip for tier enforcement:**
- Rate limit tier values (`free: 3 ollama / 2 claude`, `premium: 10 / 5`, `admin: unlimited`) are stored as in-code constants, not DB rows. Each rate-limit check reads the user's role from the JWT payload (already in memory) and the Redis counter â€” no `SELECT` on the users table per request. This keeps the quota enforcement path fast and immune to DB latency spikes.

**AppStore credential safety:**
- App Store Connect API uses ES256 JWTs with a 20-minute expiry, generated fresh per request from the private key env var. No long-lived token is stored in memory or DB â€” credential theft window is limited to the 20-minute JWT lifetime.
- `PLAY_SERVICE_ACCOUNT_JSON` loaded from env var at startup; if absent, the PlayStore service returns `null` gracefully â€” no crash or 500 on unconfigured environments.

## ASSET:price 2026-06-09 18:03 â†’ Ollama (qwen2.5:7b) as zero-cost default carries 100% of free-tier load; Redis rate limit counters expose live quota state to clients; isPremium computed server-side

**Cost-free default path:**
- Ollama running locally on the Mac mini M4 (`http://127.0.0.1:11434`) is the default AI provider â€” all free-tier recipe generation incurs zero marginal API cost
- `AI_PROVIDER` env var controls provider selection; switching from Ollama to paid providers requires a deliberate environment change, preventing accidental cost runaway in dev

**Rate limit as cost guardrail:**
- Redis counters `ratelimit:{userId}:{provider}` with 1-hour TTL enforce hard caps before any paid API call is made
- `getRecipeUsage()` returns real-time quota state â€” clients can surface "X of Y AI calls used" without a round-trip to the AI provider
- `isPremium` flag computed server-side as `role !== "free"` â€” no client-side role spoofing can unlock higher quota tiers

**Tier structure clarity:**
- Role tiers (free/premium/admin) map directly to quota limits stored in code constants â€” no database lookup needed per request, making quota checks fast and deterministic
- `express-rate-limit` on auth endpoints (10 req/15 min) prevents credential stuffing that could create premium accounts fraudulently

**AppStore / PlayStore credential isolation:**
- App Store Connect API uses short-lived ES256 JWT tokens (20-min expiry) â€” no long-lived credential stored in memory
- PlayStore service account JSON is loaded from env var â€” absent credentials cause graceful null return, not a crash
## ASSET:price 2026-06-07 16:30 â†’ Redis-backed hourly rate limits per role/provider; free=3 Ollama/2 Claude, premium=10/5, admin=unlimited

**Rate limit implementation (`src/middleware/rateLimit.ts`):**
- Per-user, per-provider (ollama/claude) Redis counters with 1-hour TTL
- Role-based limits: `free` â†’ 3 Ollama / 2 Claude per hour; `premium` â†’ 10 / 5; `admin` â†’ bypass
- `getRecipeUsage()` function exposes current usage counters to the client (visible in UI)
- `express-rate-limit` package used on all auth endpoints (10 req/15 min window) to prevent brute force

**Cost-control infrastructure:**
- Ollama (local `qwen2.5:7b`) is the default provider â€” zero marginal API cost for the majority of usage
- AI provider selection is runtime-configurable via `AI_PROVIDER` env var â€” can switch away from paid providers without deployment
- AppStore/PlayStore metric polling uses ES256 JWT tokens with 20-minute expiry â€” scoped API access

**UserRole enum:** `free` / `premium` / `admin` â€” role gates rate limits and premium feature access
**`isPremium` flag** computed server-side as `role !== "free"`, so no client-side bypass possible
## ASSET:price 2026-06-07 10:00 â†’ Role-based rate limits via Redis; 3 AI providers with cost profile

**Rate limit architecture** (`src/middleware/rateLimit.ts`):

| Role | Ollama/hr | Claude/hr |
|---|---|---|
| free | 3 | 2 |
| premium | 10 | 5 |
| admin | unlimited | unlimited |

- Limits tracked in Redis: key `ratelimit:{userId}:{provider}`, TTL 1 hour
- Per-request: checks user role from DB, increments Redis counter, returns 429 with `retryAfter` seconds if exceeded
- Redis: `ioredis` with offline queue disabled, exponential retry backoff `min(times*200, 2000)`

**AI provider cost profile:**

| Provider | Cost | Selection |
|---|---|---|
| Ollama (`qwen2.5:7b`) | Free â€” runs locally on Mac mini M4 (jayagent account, :11434) | Default |
| OpenAI | API cost per call | `AI_PROVIDER=openai` or per-request body |
| Claude (Anthropic) | API cost per call | `AI_PROVIDER=claude` or per-request body |

**Usage endpoint:** `GET /recipes/usage` â€” returns per-user quota state (used/max/ttl) for both providers

The FAQ accurately documents the free/premium limits (free: 2 Claude + 3 Basic/hr; premium: 5 Claude + 10 Basic/hr) and the SharedRecipe page correctly uses `recipe.provider === 'claude'` to display `Premium` vs `Basic` labels in the recipe meta grid. The `isPremium` badge on the author card (`authorRole !== 'free'`) is also consistent with the FAQ's description of the role model. No contradictions were found between the FAQ copy and the live UI rendering logic.
