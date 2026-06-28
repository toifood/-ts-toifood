ASSET LOG
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ASSET ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES.

REQUIRED FORMAT FOR EACH ASSET ENTRY:

## ASSET:{NAME OF ENVIRONMENT} {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD ALL NEW ASSET ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES-->## ASSET:bug 2026-06-14 08:29 → Pages Function OG meta regex correctly exploits tag structure to avoid double-closing
## ASSET:bug 2026-06-29 06:25 → wasmInitPromise pattern is race-condition-safe; AnnouncementNote null guard is doubly enforced

**`og-worker/src/index.js` — WASM init is correctly structured**
`initWasm(resvgWasm)` is called once at module scope before any request handler executes. Each request `await`s `wasmInitPromise` before constructing `Resvg`. On a warm Worker instance this resolves immediately; on a cold start it waits. This is the correct Cloudflare Workers WASM pattern — calling `initWasm` inside the handler would throw `"already initialized"` on the second request in the same isolate. The module-level promise also ensures concurrent cold-start requests both await the single initialisation rather than racing to double-initialise.

**`frontend/src/components/AnnouncementNote.jsx` — null guard prevents crash**
The component returns `null` early when `!config || !config.text`. Every call site passes the result of `useAnnouncementNoteManager`, which returns `null` for unresolved notes. This means callers like `{notes.descriptionNote && <AnnouncementNote config={notes.descriptionNote} />}` are doubly protected — the conditional render in `SharedRecipe.jsx` and the guard inside the component itself. There is no path where a missing or empty note causes a render crash.
## ASSET:bug 2026-06-28 06:36 → announcementNote.js now fully implemented; og:image regex correctly uses [^>]*

**`frontend/src/utils/announcementNote.js` — critical crash resolved**
The June 15 ISSUE entry logged that `announcementNote.js` was empty, causing `useAnnouncementNoteManager` to throw `TypeError` on any recipe with announcement note fields. The file now exports all three required functions — `resolveNote`, `resolveDietaryAllergyNote`, and `resolveDietaryInfoNote` — with correct signatures matching the hook's import. `resolveDietaryAllergyNote` properly maps dietary tags to `criticalAllergen` values and builds a safety-critical allergy warning string. `resolveDietaryInfoNote` similarly produces dietary guideline notes. Both functions guard against empty input and return `null` rather than a broken object, which is the correct contract for the `AnnouncementNote` component.

**`frontend/functions/recipe/[token].js` — og:image regex is correct**
The June 24 ISSUE entry logged a regression from `[^>]*` to `[^]*` in the og:image replace call, which would have consumed the rest of the HTML document. Current code reads `/<meta property="og:image"[^>]*/` — the `[^>]*` form, which correctly stops before the closing `>` and does not destroy the document. The `escapeHtml` function on title, description, and image URL continues to protect against XSS via malicious recipe content in all injected tag attributes.
## ASSET:bug 2026-06-27 10:55 → Bug audit of ts-toifood-web (frontend + og-worker)

Three concrete bugs identified across the web layer.

**Timer reset (`frontend/src/pages/SharedRecipe.jsx`)**
`timeLeft` is initialised from `recipe.cookTime * 60` on data load but never re-initialised when the countdown expires. After expiry the play button is a no-op until page reload. Fix: when `timeLeft === 0` and the user presses play again, reset `timeLeft` to `recipe.cookTime * 60` before setting `timerActive = true`.

**Silent title truncation (`og-worker/src/index.js`)**
`wrapTitle` drops words beyond 2 lines with no ellipsis. Fix: after the loop, if the original title had more words than were consumed, append `…` to `lines[lines.length - 1]`.

**Cache TTL mismatch (`og-worker/src/index.js`)**
Upstream recipe cache is 300 s; downstream OG image `max-age` is 86400 s — a 288x gap. Fix: lower `max-age` to ~600 s, or derive a cache key from the recipe update timestamp to allow targeted invalidation.
## ASSET:bug 2026-06-24 19:10 → Injection guards and cross-browser icon loading are correctly implemented throughout

**`escapeHtml` applied to all injected values** (`frontend/functions/recipe/[token].js`)
All three dynamic values — `title`, `description`, and `ogImage` — pass through `escapeHtml` before interpolation into the HTML template. The function escapes `&`, `<`, `>`, and `"`, covering both attribute-value injection and HTML-entity encoding. A recipe title containing `"` or `<` cannot break the injected meta tag structure or escalate into script injection.

**Ionicons loaded with correct ESM + nomodule fallback** (`frontend/index.html`)
```html
<script type="module" src="https://unpkg.com/ionicons@7.2.1/dist/ionicons/ionicons.esm.js"></script>
<script nomodule src="https://unpkg.com/ionicons@7.2.1/dist/ionicons/ionicons.js"></script>
```
Both the ES-module (modern browsers) and the legacy UMD (non-module browsers) variants are loaded, ensuring `<ion-icon>` web components register and render in all environments the app targets. Because web components upgrade in place, icons that render before the script loads will backfill correctly with no layout shift.

**iOS Smart App Banner wired up** (`frontend/index.html`)
`<meta name="apple-itunes-app" content="app-id=6761888929, ...">` enables the iOS native app-install banner automatically on shared recipe pages. Users on Safari mobile who haven't installed the app see a native prompt without any JavaScript or A/B logic — a high-value conversion touch-point with zero maintenance cost.
## ASSET:bug 2026-06-24 09:32 → Defensive patterns in place reduce crash surface across fetch and rendering paths

`og-worker/src/index.js` wraps both the recipe API fetch and the Twemoji fetch in independent `try/catch` blocks with 3-second `AbortController` timeouts. Failures in either path fall back gracefully: the title defaults to `'A recipe from toifood'` and the emoji renders as an inline SVG `<text>` element rather than an embedded image. The worker always returns a valid PNG response regardless of upstream availability.

`frontend/src/hooks/useReveal.js` correctly calls `observer.disconnect()` both in the effect return (cleanup on unmount) and immediately after the element becomes visible, preventing IntersectionObserver callbacks from firing after the component has cleaned up.

`frontend/src/utils/announcementNote.js` exports pure functions with explicit null-guards (`text?.trim()`, `.filter(Boolean)`) that make the note resolution predictable and side-effect-free — any breakage is isolated to that module.
## ASSET:bug 2026-06-24 09:00 → Timer cleanup and OG edge caching are implemented correctly

**Timer interval cleanup is safe**
`useEffect` in `SharedRecipe.jsx` (lines 246–256) returns a cleanup that calls `clearInterval(timerRef.current)` on every dependency change — `timerActive` and `timeLeft` — so the interval is always torn down before a new one is started. There is no interval accumulation or leak across re-renders.

**OG worker uses Cloudflare fetch cache correctly**
The `fetch` to `https://api.toifood.co.nz/recipes/public/${token}` passes `cf: { cacheTtl: 300 }`, instructing the Cloudflare edge to cache the upstream recipe response for 5 minutes. Combined with the `Cache-Control: public, max-age=86400` header on the rendered PNG response, repeated social-preview requests for the same recipe token are served from edge cache with no origin hit — good for cost and latency.

**`escapeXml` in og-worker prevents SVG injection**
All dynamic strings (recipe title, emoji fallback) pass through `escapeXml` before being interpolated into the SVG template, correctly neutralising `&`, `<`, `>`, and `"` characters that would otherwise break the SVG or introduce content injection.
## ASSET:bug 2026-06-23 22:01 → Defensive patterns inventory — all external calls have timeouts, XSS guards, and graceful fallbacks

**Fetch safety** (`og-worker/src/index.js:50–65, 67–78`)
Both the recipe API fetch and the Twemoji image fetch are guarded with `AbortController` + `setTimeout(3000)`. Neither error propagates; the worker always returns a valid PNG response regardless of upstream failure.

**XSS / SVG-injection guards**
- `escapeHtml` in `frontend/functions/recipe/[token].js` wraps all user-supplied title, description, and image URL strings before injecting into HTML.
- `escapeXml` in `og-worker/src/index.js` escapes recipe title and emoji before embedding in SVG markup, preventing SVG injection through recipe content.

**Memory leak prevention** (`frontend/src/hooks/useReveal.js:9,13`)
`IntersectionObserver.disconnect()` is called both in the observed callback (on intersection) and in the `useEffect` cleanup. Elements are never left observed after unmount or after the reveal fires.

**Cloudflare CDN caching** (`og-worker/src/index.js:52`, `frontend/functions/recipe/[token].js:37`)
Both the og-worker and the Pages function set `cf: { cacheTtl: 300 }` on upstream recipe fetches, keeping backend load low under social-share bursts.

**Announcement note null safety** (`frontend/src/utils/announcementNote.js:11`)
`resolveNote` trims and null-checks the input string before returning; `AnnouncementNote.jsx` also guards on `!config || !config.text` before rendering, preventing blank note cards from empty AI-generated fields.
## ASSET:web 2026-06-23 21:00 → API error handling and OG worker fallbacks are solid throughout

**1. SharedRecipe error state** (`frontend/src/pages/SharedRecipe.jsx` ~L476-485)
The recipe fetch is wrapped in `.catch(() => setError(true))` with a friendly full-page error UI ("Recipe not found" with a back link), rather than crashing or showing a blank page.

**2. og-worker abort timeouts** (`og-worker/src/index.js`)
Both the recipe API call and the Twemoji PNG fetch use explicit `AbortController` with 3-second timeouts. If either fails, the worker continues gracefully — recipes without emoji fall back to SVG text rendering, and the response is always a valid PNG.

**3. Pages Function fallthrough** (`frontend/functions/recipe/[token].js`)
If the recipe API is unavailable at SSR time, the function falls through to safe default values (`title = 'toifood'`, `description = 'A recipe shared on toifood.'`) rather than returning a 500 error.

**4. AnnouncementNote null guard** (`frontend/src/components/AnnouncementNote.jsx` L494)
`if (!config || !config.text) return null;` prevents rendering an empty note card if the backend returns a note with no text, or if `resolveNote()` returns null.
## ASSET:bug 2026-06-20 19:23 → Analysis only — no files changed

Four bugs identified (see corresponding ISSUE entry). No code was modified in this run. All bugs are in existing files:
- `frontend/functions/recipe/[token].js` — og:image URL, dimensions, twitter attribute convention
- `og-worker/src/index.js` — wrapTitle word-drop
## ASSET:bug 2026-06-15 20:02 → Fix stubs for announcementNote, fix twitter meta regexes, fix FAQ scroll timing

Fix 1 — announcementNote.js minimum stubs to prevent crash:
export const resolveNote = (n) => (n?.text ? { text: n.text, type: n.type ?? 'note' } : null);
export const resolveDietaryAllergyNote = () => null;
export const resolveDietaryInfoNote = () => null;
(Full implementation in RECOVERY-ASSET)

Fix 2 — Cloudflare Function twitter meta regexes in frontend/functions/recipe/[token].js:
Change all three twitter tag replacements to match either 'property=' or 'name=' attribute:
html = html.replace(/<meta (?:property|name)="twitter:title"[^>]*>/, `<meta name="twitter:title" content="${t}" />`);
Apply same pattern to twitter:description and twitter:image.

Fix 3 — FAQ deeplink scroll timing in FAQ.jsx FAQItem:
Replace setTimeout(() => scrollIntoView(...), 300) with:
requestAnimationFrame(() => requestAnimationFrame(() => itemRef.current?.scrollIntoView({ behavior: 'smooth', block: 'center' })));

The `og:image` replacement in `frontend/functions/recipe/[token].js` uses `/<meta property="og:image"[^>]*/` — `[^>]*` stops before the `>` character, so for `<meta property="og:image" content="old" />` it matches `<meta property="og:image" content="old" /` (consuming the `/` but leaving the `>`). The replacement string ends with `/`, so the result is `<meta property="og:image" content="new" />` — the original `>` closes the tag. This is a subtle but correct use of the regex; the tag is well-formed in the output.
