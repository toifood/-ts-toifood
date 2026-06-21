ISSUE LOG - USAGE
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ISSUE ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ISSUE ENTRIES.

REQUIRED FORMAT FOR EACH ISSUE ENTRY:

## ISSUE:usage {YYYY-MM-DD HH:MM} â†’ {CONTENT}


CUSTOM PROMPT:
Performance bottlenecks, N+1 queries, memory leaks, resource exhaustion

PATHS:
would/

####### <!-- ANCHOR MARKER - ADD ALL NEW ISSUE ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ISSUE ENTRIES-->
## ISSUE:usage 2026-06-21 19:41 → AUTH-METRIC.csv grows unboundedly and 5-query insight fan-out will degrade with user growth

Three performance and resource concerns grounded in the current data and code:

**1. AUTH-METRIC.csv unbounded growth with full-file GitHub sync on every auth event**
The file currently has 46 rows (Apr–Jun 2026). Every login/register event triggers pushRowToGitHub which downloads the entire file, appends one row, and re-uploads it. As the user base grows, each PUT transfers O(rows) bytes. At 10k auth events the file approaches 500KB and each write downloads and re-uploads it in full. Introduce chunked log files by month, or replace GitHub file writes with an append-only DB table.

**2. GET /insights runs 5 parallel Prisma queries on every call**
src/routes/insights.ts calls Promise.all with 5 findFirst queries per request. With no caching, each page load that fetches insights hits Postgres 5 times. A single findMany grouped by category would reduce this to one round-trip.

**3. GET /users/:id/profile loads all shared recipes then aggregates in-process**
prisma.recipe.findMany({ where: { userId: id, shareToken: { not: null } } }) has no row limit. For a user with many shared recipes, all rows are returned to Node.js for in-process counting. Move the aggregation to the database using Prisma groupBy to avoid transferring rows that are immediately summed and discarded.
## ISSUE:backend 2026-06-20 11:39 -> Two new usage gaps — AUTH-METRIC pushed to GitHub but IPs stored unredacted, and storeReport output has been silently missing since introduction

**New finding 1 — AUTH-METRIC.csv in GitHub repo contains unredacted user IPs**
`pushRowToGitHub()` in `src/routes/auth.ts` pushes each auth event row to `toifood-dev/ts-toifood-dev/would/AUTH-METRIC.csv`. The `ip` field is the raw client IP (IPv4-mapped prefix stripped but otherwise unredacted). This file is now growing in `ts-toifood-dev` git history. If repo access is ever broadened or the file is referenced in analytics tooling, user IPs are exposed. The `would/AUTH-METRIC.csv` currently in `ts-toifood-dev` contains rows from production auth events. Retention of IPs in a Git-tracked file is a data minimisation concern.

**New finding 2 — storeReport output has been missing since introduction**
`storeReport.ts` writes to `-ARCHIVE/-WOULD/usage-issue-v1.md` and `-ARCHIVE/-WOULD/usage-asset-v1.md`. Neither path exists. Every scheduled store KPI report has been failing silently with `ENOENT`. The `USAGE-ISSUE-V1.md` entries showing iOS installs=0 (logged 2026-05-31) were likely written before the path was changed — or the cron was manually run from a different working directory. No store KPI data has been recorded in the intended location since the `-ARCHIVE/-WOULD/` path was introduced.

**Ongoing gaps (unchanged from June 13th):**
- Rate-limit hit rate (429 events) not tracked in any CSV or metric
- CookRecord abandonment rate not surfaced in digest or any report
- Insight generation/acceptance rate not reported
## ISSUE:usage 2026-06-19 16:46 → Metrics are flat-file CSV with no retention, funnel, or user-level aggregation; user IDs committed to git

**Issues found:**

1. **CSV metric files with user IDs are committed to the repo.** The file tree shows `logs/discover-metrics.csv`, `logs/recipe-metrics.csv`, `logs/memory-metrics.csv` as tracked git blobs. These files contain `userId` fields and generation timestamps. User PII in version history is a privacy risk. These files should be added to `.gitignore` immediately.

2. **No retention or funnel metrics.** The CSV tracks per-generation events but has no concept of: daily active users, retention (day 1/7/30 cohorts), generate → save conversion rate, or feature adoption (pantry, lists, flows). There is no way to determine if users are returning.

3. **Daily digest reads only today's data.** `src/digest.ts` filters CSV lines by today's ISO date prefix. No weekly or monthly rollup, no trend comparison, no alert when daily count drops significantly.

4. **Discover metrics not logged for zero-result queries.** `if (scored.length > 0)` gates the `appendDiscoverMetric` call. Users who get no discover results (empty pantry, no shared recipes) are invisible — missing a critical signal for onboarding friction.

5. **Rate limit counter resets on Redis restart.** `GET /recipes/usage` returns Redis counters. If Redis restarts mid-hour, the client sees max limits as if unused, while the actual usage is lost.

6. **`dump.rdb` is committed to the repo.** The file tree shows `dump.rdb` — a Redis RDB snapshot — as a tracked file. This contains rate limit state and should not be in version control.

**Recommendation:** Add `logs/` and `dump.rdb` to `.gitignore` immediately. Add discover logging for zero-result cases. Consider a weekly summary cron in the digest.
## ISSUE:usage 2026-06-19 16:05 → OllamaProvider queue is per-instance so concurrent requests bypass serialization; discover LATERAL join has no index on lowercased pantry ingredient

Two resource efficiency concerns: (1) `OllamaProvider` defines `private queue: Promise<unknown> = Promise.resolve()` to serialize Ollama calls, but `src/routes/recipes.ts` calls `new OllamaProvider()` on every request (`const { OllamaProvider } = await import(...); const ollama = new OllamaProvider()`). Each request gets a fresh instance with an empty queue, so concurrent recipe generations each get their own independent queue that never blocks the others. Under concurrent load, multiple Ollama `/api/generate` calls run in parallel against the same `qwen2.5:7b` model, which cannot pipeline requests — this causes latency spikes and potential OOM on the Mac mini M4. (2) The `GET /recipes/discover` LATERAL subquery (`FROM UNNEST(r."pantryUsed") pu WHERE LOWER(pu) IN (SELECT LOWER(ingredient) FROM "PantryItem" WHERE "userId" = ...)`) runs against `PantryItem` with no index on `LOWER(ingredient)`. As shared recipe and pantry counts grow, this becomes a sequential scan per recipe row.
## ISSUE:usage 2026-06-19 14:28 → UserDataContext.refresh() fires 4 API calls on auth change AND on every app foreground with no deduplication; recipeAPI.getMyRecipes() fetches full payloads just for a count

Two performance concerns: (1) `UserDataContext.refresh()` issues four parallel requests (`getPreferences`, `getUsage`, `pantryAPI.getAll`, `recipeAPI.getMyRecipes`) on every `isAuthenticated` change AND on every `AppState 'active'` event (via the second `useEffect`). If the user locks and unlocks their phone, all four calls fire. No debounce, no flag guard against concurrent in-flight calls, no cache TTL. (2) `recipeAPI.getMyRecipes()` (`GET /recipes`) returns up to 500 full recipe objects including `ingredients[]`, `steps[]`, and `pantryUsed[]` arrays — several MB of JSON — just to compute `setRecipeCount(recipes.length)`. A `/recipes/count` endpoint or a `_count` select would reduce this to a single integer.
## ISSUE:usage 2026-06-15 09:12 → GET /recipes fetches up to 500 full recipe objects with no cursor pagination; emojiCache has no size bound

`src/routes/recipes.ts` line 562–587: `prisma.recipe.findMany` with `take: 500` and no cursor pagination. Each row includes `ingredients` (string[]), `steps` (string[]), `pantryUsed` (string[]) — the full recipe payload. For a user with 400+ saved recipes, this single query returns several MB of JSON on every app load. There is no incremental loading. Combined with the `savedListItems: { select: { listId: true } }` join, this is an N-row lateral join. As usage grows, this endpoint will dominate response latency. Secondary: `emojiCache` Map (line 18) accumulates Twemoji PNG buffers (~3–8 KB each) indefinitely per process lifetime with no eviction — a minor but real memory growth vector over days.
## ISSUE:usage 2026-06-14 23:03 → recipe-metrics.csv is an unbounded flat file with no query layer

`logs/recipe-metrics.csv` is appended on every generation call (`src/routes/recipes.ts:844`). It has no rotation, no size limit, and no query interface. The file lives on the Mac mini M4 filesystem — a disk-full event would silently drop all future metrics writes (the catch block only logs a warning).

Migrating to a `RecipeMetric` Postgres table via Prisma would: enable SQL queries over generation history, tie into the existing backup story, and surface usage patterns (provider breakdown, pantry match rates, continent diversity) without manual CSV parsing. The existing CSV columns map cleanly to a Prisma model.
## ISSUE:usage 2026-06-13 18:11 → No pagination on recipe list; OG image generation blocks the request thread

`GET /recipes` returns all recipes for a user in a single query with no limit or cursor — as a user's recipe count grows this will become a memory and latency problem. `@napi-rs/canvas` OG image generation runs synchronously in the recipe-generate request handler: canvas creation, drawing, and PNG encoding block the Node.js event loop per recipe. `runInsightAnalysis` fetches up to 50 recipes and calls Ollama up to 5 times after each recipe generation; under concurrent load this compounds DB query and Ollama pressure. Append-only CSV log files (`RECIPE-METRIC.csv`, `DISCOVER-METRIC.csv`, `DIGEST-METRIC.csv`) have no rotation strategy and grow unboundedly on a long-running server.
## ISSUE:usage 2026-06-13 17:04 → MEMORY-METRIC.csv orphaned; cook records absent from digest; getRecipeUsage returns 0 on Redis failure

**1. MEMORY-METRIC.csv is orphaned.** `logs/MEMORY-METRIC.csv` appears in the git tree but no code in `src/` writes to it. It may be written by an external process under the `jayagent` account or is a leftover artifact. If external, it should be documented; if orphaned, it should be removed.

**2. Cook records not summarised in daily digest.** The `CookRecord` model (migration `20260531000000`) tracks STARTED/COMPLETED/ABANDONED cooking sessions. `src/digest.ts` does not read or report on cook records. Adding a daily completion rate (`COMPLETED / (COMPLETED + ABANDONED)`) to the digest would give visibility into a new engagement metric.

**3. `getRecipeUsage` returns zeroed quota on Redis failure.** `src/middleware/rateLimit.ts:40-57` returns `{used: 0, max: N, ttl: 0}` for both providers when Redis throws. This means `GET /recipes/usage` tells users they have full quota even if they've exhausted it, which could create UX confusion (generate button appears enabled, then hits a Redis-recovered 429).

**4. No retention policy on CSV metric files.** Log files grow indefinitely. On a Mac mini with limited SSD, `RECIPE-METRIC.csv` will accumulate all-time history. A cron rotation or line-count cap would prevent unbounded growth.
## ISSUE:usage 2026-06-09 18:16 â†’ digest.ts is the primary usage-review mechanism but its data queries and output format are invisible to this tracking system; Slack alerts are ad-hoc strings with no structured event taxonomy

**digest.ts as a black-box usage channel:**
- `src/digest.ts` is a separate process that generates periodic operational summaries, likely sent to Slack via `chatAlert()`. The data it queries (user counts, recipe counts, error rates), the aggregation window, and the output format are entirely undocumented. If the product owner's primary view into weekly usage is this digest, decisions are being made on an ad-hoc Slack message that has no retention, no searchability, and no versioned format.
- If digest is renamed, refactored, or crashes silently (no pm2 supervision), usage visibility disappears with no alert.

**chatAlert() event taxonomy is unstructured:**
- `chatAlert()` is called from multiple callsites (Apple auth failure, recipe generation error, likely digest summaries) with free-form message strings. There is no event taxonomy, no `event_type` field, and no Slack channel routing by severity or category. As call sites multiply, Slack becomes noisy without structured filtering. Automated alerting rules cannot be built on regex-matched free text.

**Historical usage irrecoverable after log rotation:**
- pm2 default log rotation discards logs after they reach a size threshold. Console request logs (`[req] METHOD PATH STATUS DURATIONms userId=X`) are the only persistent telemetry â€” once rotated, weekly active users, endpoint hit rates, and error rates for any previous time window are unrecoverable. No log export to a queryable sink (Datadog, Loki, CloudWatch) exists.

## ISSUE:usage 2026-06-09 18:03 â†’ Cook session funnel invisible; insight acceptance rate untracked; no per-provider cost attribution from DB; UserFlowView step-drop data collected but never queried

**Cook funnel not aggregated:**
- `CookRecord` stores STARTED/COMPLETED/ABANDONED per session, but no route or job computes aggregate funnel stats (startâ†’complete rate, avg session duration, most-abandoned recipes). The data is in DB but only accessible by raw SQL; there is no admin endpoint to surface it.

**Insight acceptance untracked:**
- `PATCH /insights/:id` accepts `action: "accept" | "dismiss"` but `UserInsight.acceptedAt` or a status field is not visible in the schema â€” it is unclear whether accept/dismiss is persisted at all, or just triggers some UI update. If insight acceptance is not stored, the ML feedback loop (which insights are useful) cannot be built.

**AI provider cost attribution gap:**
- `Recipe.provider` stores the AI provider per recipe. `CookRecord` links recipe to cook session. But there is no query that aggregates: "total recipes generated per provider per week per cohort." Cost-per-cohort analysis requires joining Recipe + User + CookRecord with a GROUP BY provider, which does not exist as an endpoint.

**UserFlowView step-drop data unused:**
- `UserFlowView.skippedSteps` (array) and `UserFlowView.responses` (JSON) are written on flow completion, but no admin or analytics endpoint queries them. Step-level drop-off rates (which onboarding step causes most users to skip) are derivable from this data but currently invisible.

**Request logs not persisted:**
- All HTTP request logs go to stdout only. pm2 rotates logs by default after a certain size. Historical usage trends (weekly active users, endpoint popularity, error rate over time) cannot be reconstructed once logs rotate.
## ISSUE:usage 2026-06-07 16:30 â†’ CookRecord data collected but no aggregation endpoint; insight trigger unknown; storeMetrics data only via admin route

**CookRecord analytics gap:**
- `CookRecord` now stores pantry vs. grocery ingredient breakdowns per cook session, but there is no aggregation endpoint to answer questions like "what % of ingredients users typically have in pantry" or "how often do cooks abandon vs. complete". The data is collected but not surfaced.

**Insight trigger opacity:**
- `runInsightAnalysis()` exists in `src/services/ai/insights.ts` but it is unclear from the codebase when/where it is triggered. If it is triggered only on recipe save, users with old recipe libraries will never receive insights. No scheduled job or batch runner is visible.

**Store metrics:**
- `GET /store-metrics` is behind requireAdmin â€” only accessible to admins. If the product team wants to see AppStore/PlayStore install counts, they need admin credentials. No read-only analytics role exists.

**Flow view tracking:**
- `UserFlowView` tracks flow completion, skipped steps, and JSON responses â€” useful data â€” but no admin endpoint exists to query aggregate flow completion rates or step drop-off.

**Continent/dietary data collected but unused in reporting:**
- `continentPreferences`, `dietaryTags`, `mealType` are stored per-recipe but only surfaced in the public profile endpoint. No internal dashboard or aggregate query exists.
## ISSUE:usage 2026-06-07 10:00 â†’ No analytics beyond console logs; recipe provider not tracked per-user in DB; UserFlowView written but never queried

**Analytics gap:**
- All request logging goes to console only: `[req] METHOD PATH STATUS DURATIONms userId=X`. No structured logging to a sink (e.g. Datadog, Loki, Seq). No way to query usage history after log rotation.
- `GET /stats` rounds all counts to nearest 10 for display â€” not useful for actual analytics. No admin dashboard endpoint that shows real counts.

**Recipe provider not persisted correctly:**
- `Recipe.provider` field exists in schema but it's unclear from routes whether the AI provider name (ollama/openai/claude) is consistently written on save. If missing, it's impossible to determine cost breakdown from DB alone.

**UserFlowView unused in queries:**
- `UserFlowView` table records onboarding flow completions, but no route returns aggregate completion rates. `GET /admin/flows` returns `_count: { views: true }` which gives total views but not completion vs. skip rates.

**Chat route undocumented:**
- `src/routes/chat.ts` exists but is not in README â€” unclear what it does or how it's used.

**YouTube integration:**
- `src/services/youtube.ts` is called per recipe generate â€” if it fails, it's unclear whether it's a blocking error or silent skip. No logging of how often video lookup succeeds.

**Action needed:** Add structured request logging. Consistently write `provider` on recipe save. Add analytics endpoints for admins. Document the chat route.

For a single `/recipe/:token` page load, `api.toifood.co.nz/recipes/public/${token}` is called by: (1) the Cloudflare Pages Function at request time for SSR meta injection, (2) the `toifood-og` Worker when the browser/crawler fetches the OG image, and (3) the React client on mount in `SharedRecipe.jsx`. All three calls are independent with no shared cache or coordination. The Pages Function and OG Worker use `cf: { cacheTtl: 300 }`, but the client-side fetch has no CDN caching, so each real user visit produces at minimum one uncached call to the origin.
