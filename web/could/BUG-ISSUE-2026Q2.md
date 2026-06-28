ISSUE LOG
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ISSUE ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ISSUE ENTRIES.

REQUIRED FORMAT FOR EACH ISSUE ENTRY:

## ISSUE:{NAME OF ENVIRONMENT} {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD ALL NEW ISSUE ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ISSUE ENTRIES-->## ISSUE:bug 2026-06-14 08:29 → `wrapTitle` in OG worker silently drops trailing words when a title fills both lines exactly
## ISSUE:bug 2026-06-29 06:25 → og-worker fires wasted 404 CDN fetch for emoji-less recipes; EST. cook time display ignores hours conversion

**Finding — `og-worker/src/index.js` (empty emoji guard missing)**
When `recipe.emoji` is absent or empty, `emoji` is set to `''`. `emojiCodepoint('')` spreads the empty string to an empty array, maps nothing, and returns `''`. The Twemoji URL becomes `https://cdnjs.cloudflare.com/ajax/libs/twemoji/14.0.2/72x72/.png` — a guaranteed 404. The worker still fires this fetch unconditionally before the AbortController timeout (3 s). `imgRes.ok` is false so `emojiImgTag` stays `''` and the SVG falls back to an empty `<text>` element. Net result: every OG image render for a recipe with no emoji wastes up to 3 seconds on a doomed CDN request. Fix: add `if (!emoji)` guard before the Twemoji fetch block.

**Finding — `frontend/src/pages/SharedRecipe.jsx` (line 402–403)**
The EST. cook time label renders `{recipe.cookTime} MIN` unconditionally. A 90-minute recipe shows "90 MIN"; a 120-minute recipe shows "120 MIN". The desktop timer card (line 582) correctly applies hours-and-minutes formatting for `recipe.cookTime >= 60` — `${Math.floor(recipe.cookTime / 60)} hr ${recipe.cookTime % 60} min` — but the EST. display does not use the same logic. A user cooking a 90-minute recipe sees "90 MIN" at the top of the page and "1 hr 30 min" on the timer in the same session.
## ISSUE:bug 2026-06-28 06:36 → Timer display reverts to original cook time when paused + og-worker API URL hardcoded to production

**Finding — `frontend/src/pages/SharedRecipe.jsx` (~line 581)**
The desktop timer card renders `timerActive ? formatTime(timeLeft) : (recipe.cookTime >= 60 ? ... : `${recipe.cookTime} min`)`. When `timerActive` is false — including when the user has paused mid-countdown — the display always shows the original `recipe.cookTime`, not the remaining `timeLeft`. A user who starts a 30-minute timer and pauses it with 8 minutes remaining sees the display snap back to "30 min". The `timeLeft` state is correctly decremented in the background, so the timer resumes from the right point, but the paused UI gives no indication of time remaining. Every prior timer log addressed the "cannot restart after zero" issue; this display regression at any pause point is distinct and unlogged.

**Finding — `og-worker/src/index.js` and `frontend/functions/recipe/[token].js`**
Both files hardcode `https://api.toifood.co.nz` as the recipe API base with no environment variable or `wrangler.toml` binding. Cloudflare Workers support `env` vars and `wrangler.toml` environments (production/staging), but neither the og-worker nor the Pages function uses them. Any change to OG image generation or meta-tag injection must be validated against production — there is no way to point either worker at a staging API without a code change. Combined with no automated tests (see TEST-ISSUE), OG regressions can only be caught by manually inspecting live social previews after deployment.
## ISSUE:bug 2026-06-27 10:55 → Timer cannot restart after countdown + og-worker silently truncates long recipe titles

**Finding — `frontend/src/pages/SharedRecipe.jsx`**
The cooking timer cannot be restarted once the countdown reaches zero. When `timeLeft` hits 0, `setTimerActive(false)` fires but `timeLeft` is never reset to `recipe.cookTime * 60`. Re-pressing the play button sets `timerActive = true`, but the `useEffect` guard `timerActive && timeLeft > 0` immediately evaluates false — the interval never starts. Users must reload the page to reuse the timer.

**Finding — `og-worker/src/index.js` (`wrapTitle`)**
`wrapTitle(title, maxChars = 26)` silently drops words that overflow two lines. When the second line is full and more words remain, `current = ''` is set and the loop breaks — surplus words are discarded with no ellipsis appended. A title such as "Slow-Cooked Moroccan Lamb Shoulder with Apricots" renders visibly truncated on OG images with no indication text was cut.

**Finding — `og-worker/src/index.js` (cache mismatch)**
The Worker fetches recipe data with `cf: { cacheTtl: 300 }` (5 min) but returns the generated PNG with `Cache-Control: public, max-age=86400` (24 hr). If a recipe title is updated, the OG image can remain stale in CDN/browser caches for up to 24 hours while the API reflects the change within 5 minutes.
## ISSUE:bug 2026-06-24 19:10 → og:image regex regressed from `[^>]*` to `[^]*`, destroying HTML from that tag to end-of-file

`frontend/functions/recipe/[token].js` line:
```js
html = html.replace(/<meta property="og:image"[^]*/, `<meta property="og:image" content="${img}" /`);
```

In JavaScript, `[^]` inside a character class is a negated *empty* set — it matches **any character including newlines**, making `[^]*` equivalent to `[\s\S]*`. Because `.replace()` without the `g` flag is greedy, this pattern matches from the first `<meta property="og:image"` to the **end of the document string**. The replacement value ends with `/` (no `>`), so the produced HTML has:
1. An unclosed `<meta property="og:image" content="..." /` with no closing `>`.
2. Everything after the og:image tag — `</head>`, `<body>`, the Ionicons scripts, `</html>` — is gone.

The subsequent `html.replace('</head>', ...)` silently no-ops (no `</head>` left to match), so the `og:image:width`, `og:image:height`, and `og:image:type` dimension tags are never appended. Crawlers receiving this response see a headless document fragment.

The ASSET entry from 2026-06-15 documented this regex as `[^>]*` and proved its correctness. The current file has `[^]*` — either a copy-paste slip or an editor substitution that silently changed one character. The `[^>]*` form (stop before `>`) is the correct pattern: it consumes `content="/logo.png" /` but leaves the trailing `>` in place, which then closes the replacement tag.

Separately: `AnnouncementNote.jsx` declares `const COLLAPSED_LINES = 2` at module scope but never references it. The component went straight to a modal pattern, leaving this constant as dead code that implies a planned collapse feature that was never implemented.
## ISSUE:bug 2026-06-24 09:32 → Nested profile fetch in SharedRecipe is not cancelled on unmount

In `frontend/src/pages/SharedRecipe.jsx` the initial recipe fetch chains a second `fetch` for the author profile (line ~219–224) inside a `.then()` callback. Unlike the outer fetch, this inner call has no `AbortController` and is not cancelled when the component unmounts. If a user navigates away before the profile resolves, React receives a `setFullProfile` call on an unmounted component and logs a warning; in edge cases where the token changes via client-side navigation the stale profile could flash into view for the new recipe.

The `formatMemberSince` function (line 136–139) is defined at module scope but never called anywhere in the component — the component uses inline `authorDur` logic and `formatCookDuration` instead. Dead code that will mislead future contributors.

In `og-worker/src/index.js`, `emojiCodepoint('')` on an empty emoji string returns `''`, constructing the URL `https://cdnjs.cloudflare.com/ajax/libs/twemoji/14.0.2/72x72/.png`. A network request is still dispatched to that invalid path before the `try/catch` absorbs the failure, adding unnecessary latency on every OG render for recipes with no emoji.
## ISSUE:bug 2026-06-24 09:00 → Three runtime bugs in SharedRecipe and og-worker need fixing

**1. SharedRecipe.jsx — fetch calls have no AbortController (lines 212–244)**
The recipe fetch and the subsequent author-profile fetch both run without an `AbortController`. If the user navigates away before either resolves, `setRecipe`, `setAuthor`, `setFullProfile`, `setLoading`, and `setError` are called on an unmounted component. React 18 silences the warning but the state update still runs, can interfere on remount, and wastes network bandwidth.

**2. SharedRecipe.jsx — `isPremium` badge flashes FREE → PREMIUM on load (line 296)**
`fullProfile` starts as `null`, so `authorRole` falls back to `author?.role`. If `author.role` is `'free'` but the profile endpoint returns a paid role, the badge visibly flips from FREE to PREMIUM during the second render. This is a UX regression visible on every shared-recipe page load for premium users.

**3. og-worker/src/index.js — Twemoji CDN failure produces a blank emoji in the OG image**
When the `cdnjs.cloudflare.com` PNG fetch fails or times out, the fallback is an SVG `<text>` element with a font emoji. `@resvg/resvg-wasm` renders SVG without a bundled emoji font, so the fallback renders as an empty glyph — the OG image shows no emoji at all. The WASM init (`wasmInitPromise`) is also awaited *after* all async work is complete, meaning cold-start concurrent requests each burn two round-trips before hitting a potential init failure.
## ISSUE:bug 2026-06-23 22:01 → Three confirmed bugs: dead utility function, broken emoji OG fallback, and inconsistent Twitter meta attribute rewrite

**1. `formatMemberSince` is dead code** (`frontend/src/pages/SharedRecipe.jsx:136`)
Defined at the top of the file but never called anywhere in the JSX. `formatCookDuration` is used, but `formatMemberSince` is not. Causes confusion and will drift out of sync with any future date-formatting changes.

**2. OG-image emoji fallback silently broken** (`og-worker/src/index.js:60–75`)
When the Twemoji CDN fetch fails or returns non-OK (e.g. for compound/unknown emoji), the SVG falls back to a native `<text>` element containing the raw emoji character. `resvg-wasm` is a Rust-based SVG renderer with no system emoji font; it will render the glyph as a blank box or tofu. The catch block swallows the error silently, so the OG image is delivered with no emoji and no signal to the caller.

**3. Cloudflare Pages function rewrites Twitter `property=` to `name=`** (`frontend/functions/recipe/[token].js:27–32`)
`index.html` declares all Twitter card tags with `property=` (matching the OG convention used throughout the file). The Pages function regexes correctly match on `property=`, but the replacement strings switch to `name=` for `twitter:title`, `twitter:description`, and `twitter:image`. The page therefore leaves the request with a mixed-attribute set. Both attributes are accepted by Twitter's crawler in practice, but the inconsistency creates a maintenance trap — the next developer who aligns to spec by switching `index.html` to `name=` will break the Pages function match silently.
## ISSUE:web 2026-06-23 21:00 → Four bugs — timer no-reset, OG tag regex fragility, window.location in render, og-worker twemoji CDN age

**Bug 1 — Timer cannot be restarted after reaching zero** (`frontend/src/pages/SharedRecipe.jsx` ~L441-451)
When `timeLeft` reaches 0, `setTimerActive(false)` is called but `timeLeft` stays at 0. The play button reappears but clicking it starts an interval that immediately fires `setTimerActive(false)` again. No reset mechanism exists — the timer is permanently broken after one full countdown.

**Bug 2 — Fragile OG tag regex in Pages Function** (`frontend/functions/recipe/[token].js` ~L35)
`html.replace(/<meta property="og:image"[^>]*/` strips everything from the opening tag up to the last char before `>`, then appends `/`. This works only if `index.html` contains exactly one such tag with no nested quotes. Any change to the base HTML structure will silently corrupt the injected tag.

**Bug 3 — `window.location` read during render** (`frontend/src/pages/FAQ.jsx` L1059)
`const deepLink = window.location.origin + '/faq#' + id;` is computed in the function body of `FAQItem` on every render. While this works in the current browser-only SPA, it will throw if any SSR or prerendering is ever added.

**Bug 4 — Twemoji CDN URL pinned to 14.0.2** (`og-worker/src/index.js` ~L55)
The OG worker fetches emoji PNGs from `cdnjs.cloudflare.com/ajax/libs/twemoji/14.0.2/72x72/`. Newer emoji codepoints (Unicode 15+) added to the app will produce 404 responses, silently falling back to SVG text rendering which may render as empty boxes in rsvg.
## ISSUE:bug 2026-06-20 19:23 → Three live bugs: og:image URL regression, 2× dimension mismatch, and wrapTitle word-drop still unfixed

**Bug 1 (HIGH) — og:image URL regression** `frontend/functions/recipe/[token].js`
`ogImage` is currently set to `https://api.toifood.co.nz/recipes/public/${token}/og-image`, but the 2026-04-27 ASSET entry confirmed the intent was `https://og.toifood.co.nz/${token}` (the dedicated resvg-wasm Worker). The og-worker is still deployed at `og.toifood.co.nz` (wrangler.toml: `name = "toifood-og"`) and is not being referenced by the Pages Function. Either a regression occurred or the backend added its own `/og-image` route — currently unverified. Until resolved, social crawlers may receive a different (or missing) OG image than the Worker generates.

**Bug 2 (MEDIUM) — og:image dimensions 2× actual** `frontend/functions/recipe/[token].js`
The injected meta tags declare `og:image:width=2400` and `og:image:height=1260`, but the og-worker renders at `W=1200, H=630` (`og-worker/src/index.js` lines `const W = 1200, H = 630`). Declared dimensions are 2× the actual PNG output. Some social platforms (e.g. LinkedIn) reject or distort images where declared and actual dimensions disagree.

**Bug 3 (MEDIUM) — wrapTitle word-drop still unfixed** `og-worker/src/index.js`
Previously logged 2026-06-14. When a title's second line fills to exactly ≤26 chars and the next word triggers `lines.length === 2`, the code sets `current = ''` and breaks — that word is silently discarded with no ellipsis. Example: `"Slow Roasted Lamb Shoulder With Herb Crust"` — the word `"Crust"` is dropped from the OG image text. Fix: replace `current = ''; break;` with `lines.push(current); break;` before resetting.

**Bug 4 (LOW) — twitter meta attribute convention flip** `frontend/functions/recipe/[token].js`
`index.html` uses `property="twitter:title"` / `property="twitter:description"` / `property="twitter:image"`. The Pages Function regex correctly matches these but replaces them with `name="twitter:..."`. After processing, the served HTML has a mix: `og:*` tags use `property=`, twitter tags use `name=`. No functional breakage (both are accepted by crawlers), but the inconsistency with the source convention is confusing.
## ISSUE:bug 2026-06-15 20:02 → Three bugs: empty utility crash, twitter meta regex mismatch, FAQ scroll timing

Bug 1 (CRITICAL) — frontend/src/utils/announcementNote.js is empty. useAnnouncementNoteManager imports resolveNote, resolveDietaryAllergyNote, resolveDietaryInfoNote from it. These exports do not exist. SharedRecipe throws TypeError on any recipe that has announcement note fields.

Bug 2 (MEDIUM) — In frontend/functions/recipe/[token].js, the regex replacements for twitter:title and twitter:description target the attribute 'property=' but then emit 'name=' in the replacement. The source index.html also uses 'property=' for twitter tags. This means the regex will match index.html's tags, but after replacement the new tags use 'name=' — creating an inconsistency. More critically, if the regex targets 'property="twitter:title"' but index.html uses 'name="twitter:title"', the replacement silently does nothing and leaves stale default tags.

Bug 3 (MINOR) — FAQ.jsx FAQItem deeplink scroll uses setTimeout(..., 300) which may fire before the component's layout is fully painted on slow connections, causing misaligned scroll position.

In `og-worker/src/index.js`, the `wrapTitle` function builds lines word-by-word. When `lines.length === 2` is reached mid-word, the code sets `current = ''` and breaks — the word that triggered the second-line overflow is discarded without being pushed to `lines`. For a title like `"Slow Roasted Lamb Shoulder With Herb Crust"` where the second line fills to exactly 26 chars and the next word triggers the break, that next word is lost from the OG image text with no ellipsis or truncation indicator. The fix is to push `current` before breaking when `lines.length === 2`.
