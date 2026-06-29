ASSET LOG - ANALYSIS
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ASSET ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES.

REQUIRED FORMAT FOR EACH ASSET ENTRY:

## ASSET:analysis {YYYY-MM-DD HH:MM} â†’ {CONTENT}


CUSTOM PROMPT:
Well-structured patterns, test coverage, production-ready components

PATHS:
would/

####### <!-- ANCHOR MARKER - ADD ALL NEW ASSET ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES-->
## ASSET:analysis 2026-06-30 06:41 → atomic Lua rate-limit, Redis NX cooldown, and structured per-request logging are production-solid patterns

**Finding — `src/middleware/rateLimit.ts` (Lua INCR+EXPIRE)**
The rate limiter uses an inline Lua script for atomic increment-and-expire, which correctly prevents the race condition where two concurrent requests both observe count=1 and the Redis key never receives an expiry. This is the correct Redis pattern for sliding-window counters — non-atomic alternatives (GET then SET EX) would leak uncapped keys under load.

**Finding — `src/services/ai/insights.ts` (Redis NX cooldown)**
`runInsightAnalysis` enforces its once-per-week limit with `redis.set(key, "1", "EX", ..., "NX")`. The NX flag makes the gate atomic — parallel post-save triggers from concurrent recipe saves cannot double-fire the insight pipeline.

**Finding — `src/index.ts` (request logging)**
The global `res.on("finish")` listener logs method, path, status code, duration, and userId for every request in a consistent structured format (`[req] METHOD /path STATUS Xms userId=Y`). This is grep-friendly and provides basic observability without an external APM dependency.

**Finding — `src/middleware/auth.ts`**
`requireAuth` is single-responsibility and minimal: it extracts the bearer token, verifies it, attaches `userId` to the request, and does nothing else. Token errors map to 401 without leaking internal failure reasons to the response body.
## ASSET:analysis 2026-06-29 12:37 → Discover CTE with LATERAL pantry join, irregular pluralStem dictionary in cookRecords, OG generate-then-store pattern, and res.on(finish) request logger

**Finding — `src/routes/recipes.ts` GET /recipes/discover**
The discover query uses a PostgreSQL CTE with a LATERAL subquery to compute pantry match counts in the database. The LATERAL join iterates `r.pantryUsed` via `UNNEST` and matches against the requesting user's pantry items with a `LOWER()` comparison — no N+1, no in-Node aggregation. The outer query filters, groups, and limits to 20, keeping the result set small regardless of total shared recipe count. This is the correct approach for a multi-user recipe-matching query.

**Finding — `src/routes/cookRecords.ts` `pluralStem`**
The cook records implementation includes a 14-entry irregular dictionary covering non-obvious food-relevant forms (loaves/loaf, leaves/leaf, geese/goose) plus an `ee$` guard. This level of care in ingredient stem-matching prevents false negatives in pantry coverage calculations for the cook session flow.

**Finding — `src/routes/recipes.ts` OG image pipeline**
OG images are generated at recipe-generation time (`generateOgImage(emoji)`), returned in the API response as base64, and re-submitted by the client on save where they are stored on the Recipe row. This avoids synchronous canvas rendering on every share-link page load — the heavy computation happens once at generation time and is served from the database thereafter.

**Finding — `src/index.ts` request logger**
`res.on("finish")` captures `userId` from `req` after auth middleware has run, producing structured `[req] METHOD path STATUS ms userId=` lines for every request. This provides basic APM-style observability without an external SDK and is grep-parseable for per-user request rate analysis.
## ASSET:analysis 2026-06-29 12:28 → 13 inline SVG icon components in SharedRecipe and a Wasm SVG-to-PNG OG pipeline

**Finding — `frontend/src/pages/SharedRecipe.jsx`**
13 inline SVG icon components (IcoBasket, IcoRestaurant, IcoSparkles, IcoLeaf, IcoFlower, IcoWater, IcoAlert, IcoMoon, IcoFlash, IcoBread, IcoPot, IcoFastFood, IcoChefHat) are defined at the top of the file, mirroring the mobile app Ionicons set. These are a candidate for extraction to a shared `src/icons/` module.

**Finding — `og-worker/src/index.js`**
The OG worker implements a full SVG-to-PNG pipeline: fetches Twemoji PNGs, composes an SVG with radial-gradient background + title text + emoji, and renders to a 1200×630 PNG via `@resvg/resvg-wasm` (WebAssembly in a Cloudflare Worker). Output is cached at `max-age=86400`.
## ASSET:analysis 2026-06-21 19:41 → Redis Lua atomics, Apple JWKS caching, and no-enumeration auth patterns are well-engineered

Three production-ready patterns found in the authentication and rate-limiting layer:

**1. rateLimit.ts — Lua script prevents expiry-race on first increment**
The INCR+EXPIRE Lua atom in src/middleware/rateLimit.ts correctly closes the window where two concurrent requests both see count===1 and the key never receives an expiry. The comment explains the invariant. This is the correct approach for Redis-backed rate limiting.

**2. auth.ts — Apple JWKS cached with 1hr TTL**
getCachedAppleKeys() caches Apple public keys in module memory with a proper expiry check, avoiding a live HTTPS fetch on every Apple auth call. Keys rotate infrequently so 1hr is safe and matches Apple's documented guidance.

**3. auth.ts — no-enumeration on password reset and resend-verification**
Both POST /auth/forgot-password and POST /auth/resend-verification always return 200 regardless of whether the email exists. This correctly prevents account enumeration attacks, and the pattern is consistent across both flows.
## ASSET:backend 2026-06-20 11:39 -> Codebase health snapshot update — new feature surface, GitHub auth-metric push, storeReport path bug confirmed

**Changes since 2026-06-13 entry:**

| Area | Status |
|---|---|
| `routes/cookRecords.ts` | Active in production — tracks STARTED/COMPLETED/ABANDONED cook sessions |
| `routes/flows.ts` | Active — first-login and manual flows, UserFlowView recorded per user |
| `routes/insights.ts` | Active — UserInsight CRUD, `runInsightAnalysis()` fires after every recipe save |
| `routes/admin.ts` | Active — admin-only endpoints for managing roles, flows, insights |
| AUTH-METRIC GitHub push | New — every external auth event pushed to `would/AUTH-METRIC.csv` in `ts-toifood-dev` via TOIFOOD_CROSS_REPO_TOKEN |
| `storeReport.ts` path | Broken — writes to `-ARCHIVE/-WOULD/` which does not exist; all store KPI writes silently fail |
| `getAIProvider()` factory | Still dead code — routes still instantiate providers directly |
| OG image in BYTEA | Unchanged — `Recipe.ogImage Bytes?` still in PostgreSQL; no CDN migration |
| Recipe CSV metrics | Still local-only flat files; no DB or platform migration |

**Active AI provider usage (production):**
- Ollama: default for `POST /recipes/generate` (local `qwen2.5:7b`)
- Claude: optional, selected via `provider: "claude"` in request body; falls back to Ollama on error
- Insights analysis: Ollama only (`src/services/ai/insights.ts`) — Claude and OpenAI not used
- OpenAI: still present (`src/services/ai/openai.ts`) but not wired into any route
## ASSET:analysis 2026-06-19 16:46 → Architecture topology, dependency map, and request flow reference

**Runtime topology:**
```
Mobile App (iOS/Android)
  ↓ HTTPS
Cloudflare Tunnel → toifood.co.nz
  ↓
Mac mini M4 (jayreck)
  ├── Node.js :3000  — this repo, PM2 managed
  ├── PostgreSQL :5432  — local
  └── Redis :6379  — local

Mac mini M4 (jayagent, same host)
  └── Ollama :11434  — qwen2.5:7b, reached via 127.0.0.1
```

**POST /recipes/generate request flow:**
1. `requireAuth` (JWT verify) → `recipeGenerateRateLimit` (Redis INCR)
2. `prisma.user.findUnique` (role, recipeStyle, continentPreferences) + `prisma.recipe.findMany` (last 20 titles)
3. Build prompt → send to Ollama (serial queue) or Claude (30s timeout)
4. On Claude failure → fallback to Ollama
5. `pluralStem` pantry matching on recipe ingredients
6. In parallel: `findRecipeVideo(title)` + `generateOgImage(emoji)` (Twemoji CDN fetch)
7. `appendMetric(...)` → recipe-metrics.csv
8. Return JSON

**External dependencies:**
- `https://appleid.apple.com/auth/keys` — Apple JWKS (per Apple auth request, no cache)
- `https://cdnjs.cloudflare.com/ajax/libs/twemoji/14.0.2/72x72/*.png` — emoji PNGs (cached in-process)
- `GOOGLE_CHAT_WEBHOOK_URL` — operational alerts
- YouTube Data API — `src/services/youtube.ts`
- Gmail SMTP — `GMAIL_USER` / `GMAIL_APP_PASSWORD`
- `https://api.anthropic.com/v1/messages` — Claude (when `AI_PROVIDER=claude` or `provider=claude`)

**Active data models:** User, Recipe, RecipeReview, SavedList, SavedListItem, DietaryPreference, PantryItem, Flow, FlowStep, UserFlowView, PasswordResetToken, EmailVerificationToken
## ASSET:analysis 2026-06-19 16:05 → Flow system is cleanly extensible with typed step enum and per-user completion tracking; admin guard is DB-confirmed not JWT-claimed

Two well-designed structural patterns: (1) The `Flow` + `FlowStep` + `UserFlowView` system in the Prisma schema provides a content-addressable onboarding framework: steps are ordered integers with a `FlowStepType` enum (`preferences | tip`), flows have `trigger` (`first_login | manual`), `priority`, `adminOnly`, and `isActive` flags. `UserFlowView` records per-user completion with `skippedSteps`, `responses`, and `completedAt`. Adding a new step type (e.g., `rating`) requires only extending the enum and the response handler in `src/routes/flows.ts`. (2) `requireAdmin` middleware in `src/routes/admin.ts` calls `prisma.user.findUnique({ select: { role: true } })` at request time rather than reading the role from the JWT payload. This means a role downgrade takes effect immediately without requiring token re-issue — correct for an admin escalation model where the JWT's 7-day lifetime is too long to trust for role claims.
## ASSET:analysis 2026-06-19 14:28 → Three-context architecture cleanly separates session, server data, and error state; registerUnauthenticatedHandler decouples API from React lifecycle

Context separation is correct: `AuthContext` owns only JWT token + user identity + tab-bar visibility, `UserDataContext` owns server-fetched preferences and usage data, `AppStateContext` owns the error overlay — no context is responsible for two unrelated domains. The `registerUnauthenticatedHandler` pattern in `src/services/api.ts` keeps the axios instance framework-agnostic while still routing 401 events into React state: the API module doesn't import React or reference any hook, making it independently testable. `pantryMatch.ts` exports named pure functions (`computeMatchStats`, `pantryMatch`, `ingredientContains`) with no side effects and typed interfaces, making it the strongest candidate for unit tests in the codebase.
## ASSET:analysis 2026-06-15 09:12 → AIProvider interface, discover SQL query, and extractFoodEmoji pipeline are well-abstracted and production-ready

Three strong structural patterns: (1) `src/services/ai/index.ts` exposes `getAIProvider()` — a clean factory that switches between `OllamaProvider`, `ClaudeProvider`, and `OpenAIProvider` via `AI_PROVIDER` env var. All providers implement the `AIProvider` interface with a single `generateRecipe()` method, making future provider addition a contained change. (2) `GET /recipes/discover` uses a single raw SQL query with LATERAL join to compute pantry match counts and grocery percentages in one DB round-trip — correct and efficient, avoiding N+1 fetch patterns. (3) `extractFoodEmoji()` implements a deterministic four-stage pipeline: AI emoji gate → title keyword inference → ingredient inference → hardcoded fallback. Each stage has a clear fallthrough contract. The `FOOD_DRINK_SET` gate intentionally excludes dishware and drink emojis, which is documented inline.
## ASSET:analysis 2026-06-14 23:03 → metric columns available for cross-provider recipe quality analysis

From `logs/recipe-metrics.csv`:
- **Provider**: `requestedProvider`, `usedProvider`, `fallback`
- **Quality**: `pantryPct` (selected items used / selected items), `groceryPct` (pantry items / total ingredients), `steps`, `ingredientCount`
- **Performance**: `responseMs`
- **Context**: `style` (classic/creative), `filters`, `continent`, `promptVersion`

From `logs/discover-metrics.csv`:
- `avgPantryPct`, `avgGroceryPct` across discover feed results per user session

Note: `groceryPct` in metrics CSV is currently miscalculated (see BUG-ISSUE-2026Q2) — cross-provider analysis should use `pantryPct` as the reliable signal until the bug is fixed.
## ASSET:analysis 2026-06-13 18:11 → Clean AIProvider abstraction, atomic Redis patterns, and production-grade auth flows

The `AIProvider` interface with three concrete implementations (Ollama, OpenAI, Claude) behind a factory function is a clean extension point — adding a new provider requires only a new class and a factory switch case. The Redis Lua INCR+EXPIRE script is a production-correct atomic concurrency pattern. `pluralStem` + `stemMatch` in `cookRecords.ts` handles 15 irregular plurals (leaf/leaves, knife/knives, etc.) with an explicit invariant table. Apple Sign In uses native Node.js `crypto.createPublicKey({ format: 'jwk' })` without extra dependencies. The `@@unique([userId, flowId])` constraint on `UserFlowView` prevents duplicate flow records at the DB layer regardless of concurrent requests.
## ASSET:analysis 2026-06-13 17:04 → Discover SQL, insight thresholds, and continent pool

**Discover SQL entry point:** `src/routes/recipes.ts:645-696`
- Filter: `shareToken IS NOT NULL`, `userId != requester`, `groceryPct >= 20`
- Order: `matchCount DESC, createdAt DESC` — LIMIT 20
- Includes: avgStars, reviewCount, myStars, author profileVisibility
- Pantry match is computed via `UNNEST(r.pantryUsed)` cross-referenced against the requesting user's `PantryItem` table

**Insight analysis thresholds** (`src/services/ai/insights.ts`):
| Category | Threshold | Min data |
|---|---|---|
| dietary | ≥30% of last 50 recipes match tag not in prefs | 5 recipes |
| cuisine | ≥40% of continent-tagged recipes from one continent | 3 with continent |
| style | ≥70% of styled recipes use one style ≠ current | 10 recipes, 5 with style |
| pantry | ingredient in ≥25% of recipes, not in pantry | 5 recipes |
| mealType | one type ≥50% of typed recipes | 4 with mealType |

**Continent pool:** `src/services/ai/provider.ts:383-450` — 71 [country, continent] pairs across 7 continents

**Ollama does not filter by continentPreferences:** `src/services/ai/ollama.ts:190` — `pickRegion()` called with no args
## ASSET:analysis 2026-06-09 18:16 â†’ Express middleware stack is minimal and explicit; no framework magic; Prisma-generated types make DB schema the single source of truth; ownership-guard pattern is applied uniformly across all mutable routes

**Explicit, readable middleware stack:**
- The Express middleware chain in `src/index.ts` is short and explicit: CORS â†’ JSON body parser â†’ request logger â†’ routes. There is no hidden middleware injected by a framework, no auto-discovery, no decorator-based registration. Any developer reading `src/index.ts` top-to-bottom sees exactly what executes on every request, in order. This makes debugging request-level issues straightforward.

**Prisma schema as single source of truth:**
- `prisma/schema.prisma` is the authoritative definition of the data model. Prisma-generated TypeScript types flow into all route handlers via `@prisma/client` â€” there is no separate model layer, no manual type-to-table mapping, and no ORM configuration file. Adding a column to the schema automatically makes it available and typed throughout the application after `prisma generate`. The data model and the types are structurally in sync by construction.

**Uniform ownership-guard pattern:**
- Every route that mutates user-owned data (`recipes`, `lists`, `pantry`, `records`, `insights`) uses the same `{ id, userId: req.userId! }` where-clause pattern. The consistency of this pattern means ownership enforcement is auditable by grep: any route that queries by id alone is immediately identifiable as a potential IDOR risk. This regularity is a meaningful security asset â€” it is not accidental.

## ASSET:analysis 2026-06-09 18:03 â†’ Codebase is ~2000 LOC across ~20 files; TypeScript strict typing throughout; domain-aligned route structure; operational tooling (pm2, Slack, Redis, health) production-ready for current scale

**Codebase size and shape:**
- ~2,000 lines of TypeScript across ~20 source files â€” small enough for any developer to hold the full architecture in context, with no hidden complexity in deep dependency trees
- Routes are domain-aligned (one file per entity: recipes, users, auth, pantry, lists, records, insights, flows, admin, chat) â€” finding the handler for any endpoint is predictable
- No over-engineering: middleware is minimal (auth, rate-limit, admin check), no abstract base classes, no decorator magic. The complexity is proportional to the problem.

**TypeScript coverage:**
- Full TypeScript across all source files â€” no `any` escape hatches visible in public-facing route handlers
- Shared types in `shared/src/index` create a typed contract between backend and mobile client â€” breaking changes surface at compile time, not runtime
- Prisma-generated types make all DB queries type-safe â€” column name typos are compile errors

**Operational readiness at current scale:**
- pm2 process management with auto-restart â€” zero-downtime handling of uncaught exceptions
- Slack alerting for auth failures and recipe generation errors via `chatAlert()`
- Redis-backed rate limiting with exponential backoff â€” handles Redis instability gracefully
- `/health` endpoint suitable for load balancer or uptime monitoring
- In-memory stats cache with stale fallback â€” public-facing metrics survive DB hiccups

**AI architecture extensibility:**
- `AIProvider` interface + `getAIProvider()` factory â€” adding a fourth AI backend requires implementing one interface and registering the provider key. No other code changes needed.
- Runtime `AI_PROVIDER` env var switching â€” can hot-swap providers without redeployment (after pm2 restart with updated env)
## ASSET:analysis 2026-06-07 16:30 â†’ Production-grade auth stack, typed throughout, multi-provider AI, solid input validation; codebase healthy for current scale

**Tech stack:** Node.js 18+ / TypeScript / Express / Prisma ORM / PostgreSQL / Redis / Ollama (local AI) / JWT + bcrypt / Passport (Local + Google + Apple)

**Strengths:**
- **Full TypeScript** â€” all route handlers typed, shared types in `shared/src/index`, no implicit any visible
- **Multi-provider AI architecture** â€” `getAIProvider()` factory with Ollama/OpenAI/Claude backends, switchable at runtime via env var. Clean `AIProvider` interface in `provider.ts`.
- **Auth is production-ready** â€” bcrypt/12, JWT/7d, rate-limited, Apple JWKS verified, Google OAuth via Passport, email verification token flow, password reset token flow all implemented
- **Prisma ORM** â€” type-safe queries, migrations tracked, cascade deletes configured correctly for most models
- **Role-based access** â€” `requireAdmin` middleware, role-gated rate limits, `isPremium` computed server-side
- **Operational tooling** â€” Slack alerts via `chatAlert()`, structured console logs per request, `/health` endpoint, `process.on` error handlers, Redis retry strategy, in-memory `/stats` cache with TTL
- **API versioning started** â€” 1-1-1 prefix established with explicit comment about legacy deprecation plan

**Scale assessment:**
- Appropriate for current user base on a Mac mini M4. The main bottleneck at scale will be OG image canvas generation (CPU-bound, synchronous) and Prisma connection pool defaults.
- Codebase is ~2,000 lines of TypeScript across 20 files â€” manageable, readable, no premature abstraction.

`useReveal` disconnects the IntersectionObserver after the first intersection (`observer.disconnect()` inside the callback), so elements that have animated in never incur further observer overhead. The FAQ component implements per-question deep linking via `id` attributes and `window.location.hash` comparison — users can share URLs like `/faq#premium` that auto-open and scroll to that specific question. The `scrollMarginTop: 80` on category sections correctly accounts for the fixed navbar height (64px) when category pills trigger smooth scroll.
