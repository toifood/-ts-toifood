SHOULD ISSUE LOG
prompt: review and update disaster recovery procedures, rollback plans, backup strategies, incident response runbooks
path: should/RECOVERY-ISSUE-2026Q2.md
target: ts-toifood-web

INSTRUCTION FOR AI MODEL:

YOU MAY READ AND UPDATE EXISTING ENTRIES AS THE SYSTEM EVOLVES.
ADD NEW ENTRIES AT THE TOP FOR NEW TOPICS; UPDATE IN PLACE FOR EXISTING ONES.

FORMAT: ## ISSUE:RECOVERY {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD OR UPDATE ENTRIES DIRECTLY BELOW THIS LINE -->## ISSUE:RECOVERY 2026-06-29 06:24 ▸ og-worker has no fallback image on WASM init failure; 301 redirects are unrecoverable cache risk

The `og-worker/src/index.js` calls `await wasmInitPromise` synchronously in the request path — if the WASM module fails to initialise (e.g. cold start timeout, memory limit), the entire request throws with no try/catch around the WASM render path. The worker returns a 500 with no fallback PNG. Social share cards for all recipes would break silently with no alerting mechanism.

All API proxy routes in `frontend/public/_redirects` use HTTP 301 (permanent) redirects to `https://api.toifood.co.nz/...`. A 301 is cached permanently by clients. If the backend domain changes, or these routes need to be removed/changed, there is no server-side way to invalidate cached 301s — affected users would be permanently broken until their browser cache expires. This is a recovery anti-pattern for routes that may change.

No rollback runbook documented for Cloudflare Pages deployments. Cloudflare Pages supports instant rollback through the dashboard, but the procedure (which deployment to roll back to, how to verify, who is responsible) is not documented in this repo.

The `SharedRecipe.jsx` timer state (`setTimeLeft`, `setTimerActive`) is purely in-memory — if a user refreshes mid-cook, the timer resets. This is a minor UX recovery gap but worth noting as intentional or a known limitation.
