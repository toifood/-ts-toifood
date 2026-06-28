SHOULD ISSUE LOG
prompt: review and update architecture patterns, service boundaries, infrastructure decisions, dependency graph
path: should/ARCH-ISSUE-2026Q2.md
target: ts-toifood-web

INSTRUCTION FOR AI MODEL:

YOU MAY READ AND UPDATE EXISTING ENTRIES AS THE SYSTEM EVOLVES.
ADD NEW ENTRIES AT THE TOP FOR NEW TOPICS; UPDATE IN PLACE FOR EXISTING ONES.

FORMAT: ## ISSUE:ARCH {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD OR UPDATE ENTRIES DIRECTLY BELOW THIS LINE -->## ISSUE:ARCH 2026-06-29 06:24 ▸ Dual Cloudflare deployment relationship is undocumented and og:image routing chain is implicit

The repo deploys two separate Cloudflare projects: `toifood-web` (Pages, `frontend/wrangler.toml`) and `toifood-og` (Worker, `og-worker/wrangler.toml`). The Pages Function at `frontend/functions/recipe/[token].js` injects `og:image` pointing to `https://api.toifood.co.nz/recipes/public/${token}/og-image` — this implies the backend proxies to the og-worker, but that chain is nowhere documented in this repo. If the backend changes that route, OG images silently break.

The `frontend/public/_redirects` file uses HTTP 301 (permanent) redirects for all API proxy routes (`/auth/*`, `/recipes/*`, `/pantry/*`, `/user/*`, `/lists/*`, `/flows/*`, `/stats`, `/health`, `/app-config`). Permanent redirects are cached indefinitely by browsers and CDNs — if `api.toifood.co.nz` ever changes, clients with cached 301s will be broken with no server-side way to fix it. These should be 200-proxies or 302s.

The API base URL `https://api.toifood.co.nz` is hardcoded in three separate files: `frontend/src/pages/SharedRecipe.jsx` (line `const API_URL`), `frontend/functions/recipe/[token].js`, and `og-worker/src/index.js` — no environment variable abstraction. Any domain change requires edits across all three.

No TypeScript configured — the entire codebase is plain JSX/JS. No type safety on the recipe API response shape; if the backend changes the `data.recipe` structure, runtime errors in `SharedRecipe.jsx` will be silent until users report them.
