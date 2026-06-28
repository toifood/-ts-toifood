ASSET LOG
INSTRUCTION FOR AI MODEL:

ALWAYS ADD NEW ASSET ENTRIES AT THE TOP, DIRECTLY BELOW THIS HEADER.

NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES.

REQUIRED FORMAT FOR EACH ASSET ENTRY:

## ASSET:{NAME OF ENVIRONMENT} {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD ALL NEW ASSET ENTRIES DIRECTLY BELOW THIS LINE, NEVER DELETE OR EDIT PREVIOUS ASSET ENTRIES-->## ASSET:test 2026-06-14 08:29 → Utility functions and hooks are structured for easy unit testing
## ASSET:test 2026-06-29 06:25 → useAnnouncementNoteManager is fully synchronous and covers all six note branches in a single renderHook call

**`frontend/src/hooks/useAnnouncementNoteManager.js`**
All six returned fields (`descriptionNote`, `ingredientNote`, `methodNote`, `longRecipeNote`, `dietaryAllergyNote`, `dietaryInfoNote`) are computed synchronously from plain JS values — no async, no context, no DOM, no side effects. A single `renderHook(() => useAnnouncementNoteManager(recipe, DIETARY_INFO))` with Vitest + @testing-library/react produces all six fields simultaneously. A well-chosen fixture — `steps: Array(9)` for the `longRecipeNote` threshold, `dietaryTags: ['GlutenFree', 'NutFree']` for both allergen note types, `descriptionNote: 'text'` vs `''` for the null path — covers every branch in one test suite with no mocks required. The precise `useMemo` dep list (`recipe.descriptionNote`, `recipe.ingredientNote`, `recipe.methodNote`, `recipe.steps?.length`, `recipe.dietaryTags`, `dietaryInfo`) can be verified by updating each dep individually in `act()` and asserting only the relevant note field changes — confirming no stale closure and no unnecessary recomputation.
## ASSET:test 2026-06-28 06:36 → All critical paths are pure functions — ideal unit-test surface with no mocking complexity

**`frontend/src/utils/announcementNote.js`**
All three exports (`resolveNote`, `resolveDietaryAllergyNote`, `resolveDietaryInfoNote`) are pure functions: no side effects, no async, no DOM, no React. They take plain JS primitives and return plain objects or null. Vitest can be added to the project in a single `package.json` change and these functions tested with zero setup beyond import.

**`og-worker/src/index.js` — `wrapTitle` and `emojiCodepoint`**
`wrapTitle` and `emojiCodepoint` are both pure string utilities at module scope. A vitest or plain Node test suite can import and exercise them directly — no Cloudflare Worker runtime, no WASM, no network required. A property-based test over random ASCII titles would have caught the word-drop case (`lines.length === 2` mid-word) on first run.

**`frontend/functions/recipe/[token].js`**
The `onRequest` function is a deterministic HTML transform. A test can pass a mock `context` with `params.token`, a mock `env.ASSETS.fetch` that returns a known HTML string, and a mock `fetch` for the recipe API. The entire meta-tag injection logic — including the `og:image` regex, the twitter attribute replacements, and the dimension tag injection — can be exercised in isolation from any Cloudflare runtime. The `escapeHtml` function is also independently testable and covers a security-critical code path.
## ASSET:test 2026-06-27 10:55 → Test coverage state for ts-toifood-web

Both packages (`frontend/`, `og-worker/`) have zero test files and no test runner configured. Current coverage is 0%.

**Priority additions**
- `og-worker/src/index.js`: unit tests for `wrapTitle` (overflow truncation, single-word title, exact two-line fit), `emojiCodepoint` (multi-codepoint emoji, ZWJ sequences, regional flag pairs, variation selector stripping), `escapeXml` (ampersand, double-quote, angle brackets)
- `frontend/src/utils/announcementNote.js`: unit tests for `resolveDietaryAllergyNote` and `resolveDietaryInfoNote` with full `DIETARY_INFO` map coverage including edge cases (tag present with no `criticalAllergen` field, tag with no `dietaryGuideline` field, empty tag array)
- `frontend/src/hooks/useReveal.js`: hook test with mocked `IntersectionObserver`, plus guard test for environments where the API is absent

**Recommended tooling**
- `og-worker/`: Vitest in Node mode (no DOM needed for pure utility functions; wasm can be mocked)
- `frontend/`: Vitest + @testing-library/react
## ASSET:test 2026-06-24 19:10 → FAQ and AnnouncementNote are fully testable without network or timers

**`FAQ.jsx` is synchronous and router-mockable**
The entire FAQ page renders from the static `FAQS` array and `CATEGORIES` constant — no `useEffect`, no API calls, no async data. Wrapping `<FAQ />` in `<MemoryRouter initialEntries={['/faq#premium']}>` gives full control over hash-based deep linking. All interactive behaviour (accordion expand/collapse, category pill smooth-scroll, deep-link auto-open and scroll) is exercisable with `userEvent.click` and `waitFor` with no network mocks. The `window.location.origin` read in `FAQItem` is the only environment dependency; a `vi.stubGlobal` on `window.location` handles it in one line.

**`AnnouncementNote.jsx` state surface is minimal**
The component manages exactly one boolean (`modalVisible`). All visual variation is driven by the `TYPE_STYLES` lookup on `config.type`. A test matrix of six type values × [card rendered, modal opened, modal closed] covers the full component surface in under 20 assertions. The `if (!config || !config.text) return null` guard is a single `expect(container).toBeEmptyDOMElement()` call. Because the component has no side effects and no timers, tests run synchronously and do not require `act` wrapping beyond the initial render.
## ASSET:test 2026-06-24 09:32 → Core logic is isolated in pure functions and a single hook, making test adoption low-friction

`frontend/src/utils/announcementNote.js` exports three pure functions with no side effects and no DOM or network dependencies. Adding Vitest and writing unit tests for all branching paths (empty tags, mixed allergen/guideline tags, null text) would take under an hour and give high-confidence coverage of the allergy-warning path.

`frontend/src/hooks/useAnnouncementNoteManager.js` wraps all note resolution in a single `useMemo` with a flat dependency array of primitives. It is self-contained enough to test end-to-end with `@testing-library/react`'s `renderHook`, covering the full `descriptionNote → dietaryAllergyNote → longRecipeNote` output surface in one pass.

`og-worker/src/index.js` separates `wrapTitle`, `emojiCodepoint`, and `escapeXml` as named functions before the `fetch` handler. They can be imported and unit-tested with Node's built-in test runner (`node --test`) without spinning up a Worker runtime, since they have no Cloudflare-specific API dependencies.
## ASSET:test 2026-06-24 09:00 → Codebase structure is highly amenable to testing with minimal setup

**Pure utility functions are ready to unit-test today**
`resolveNote`, `resolveDietaryAllergyNote`, `resolveDietaryInfoNote` (in `announcementNote.js`) and `wrapTitle`, `escapeXml`, `emojiCodepoint` (in `og-worker/src/index.js`) take plain inputs and return plain outputs with no DOM or network dependency. A single Vitest config would cover all of them without mocks.

**Single API base URL makes fetch mocking straightforward**
`SharedRecipe.jsx` calls only one base URL (`https://api.toifood.co.nz`) for both recipe and profile endpoints. `vi.stubGlobal('fetch', ...)` or `msw` with a single handler intercepts the entire data layer — no complex dependency injection needed.

**`useReveal` hook is independently testable**
The hook takes a threshold, sets up an `IntersectionObserver`, and returns `[ref, visible]`. Because it has no coupling to application state, it can be tested standalone with a mock `IntersectionObserver` in a few lines.

**Component tree is shallow and co-located**
All pages import from a flat `components/` and `hooks/` layer with no circular dependencies or global stores. React Testing Library + Vitest can be added to `frontend/` with one dev-dependency block and no build changes beyond `vite.config.js` adding a `test` key.
## ASSET:test 2026-06-23 22:01 → Testable pure-function surface is large and dependency-free — high ROI for a first test pass

**Immediately unit-testable (no DOM, no network, no Cloudflare env):**

`og-worker/src/index.js`
- `escapeXml(str)` — 5 edge cases (ampersand, angle brackets, quote, empty string, already-escaped)
- `emojiCodepoint(emoji)` — single emoji, compound ZWJ sequence, emoji with variation selector (fe0f strip), empty string
- `wrapTitle(title, maxChars)` — one-word title, exactly-at-limit word, long single word exceeding maxChars, empty string, > 2 lines truncation
- `bufToBase64(buf)` — round-trip against `atob` reference

`frontend/src/utils/announcementNote.js`
- `resolveNote(text, type)` — null, undefined, whitespace-only, valid string
- `resolveDietaryAllergyNote(tags, info)` — empty tags, tags with no allergen, one allergen, multiple allergens
- `resolveDietaryInfoNote(tags, info)` — same matrix

`frontend/functions/recipe/[token].js` (testable as string transforms)
- HTML fixture containing all six meta tags → verify each regex replaces correctly and does not duplicate tags
- HTML fixture missing `og:image` tag → verify graceful no-op

**Integration-testable with lightweight harnesses:**
- `og-worker`: Cloudflare Miniflare v3 (`wrangler dev --local`) can exercise the full fetch handler with mock `ASSETS` binding
- `frontend/functions`: Cloudflare Pages local mode (`wrangler pages dev`) supports full function testing against the built dist

**Inline functions worth extracting for testability** (`SharedRecipe.jsx:136–151`):
`formatMemberSince` and `formatCookDuration` are date-formatting helpers defined inline. Moving them to a shared utils file makes them directly unit-testable without mounting the component.
## ASSET:test 2026-06-20 19:23 → Analysis only — no files changed

Zero test coverage confirmed, three priority test targets identified (see corresponding ISSUE entry). No test files were created in this run. To set up:
- `frontend/`: add Vitest to `devDependencies`, add `"test": "vitest"` script to `package.json`
- `og-worker/`: add a `test/wrapTitle.test.js` file (no extra dependencies needed for pure function tests)
- Pages Function: can be tested with a Vitest environment or a plain Node test runner against the exported `onRequest` function with a mocked `context`
## ASSET:test 2026-06-15 20:02 → Add Vitest unit tests for utilities and hooks + Playwright E2E for SharedRecipe golden path

Step 1 — Install Vitest in frontend/package.json devDependencies:
"vitest": "^2.x", "@testing-library/react": "^16.x", "@testing-library/user-event": "^14.x", "jsdom": "^25.x"
Add script: "test": "vitest"

Step 2 — Unit tests to write first (highest ROI):
- announcementNote.test.js: resolveNote, resolveDietaryAllergyNote, resolveDietaryInfoNote with null/empty/valid inputs (would have caught the empty-file bug)
- og-worker/src/index.test.js: wrapTitle with 0/1/2/3-word titles and 26-char boundary; emojiCodepoint with and without variation selectors; bufToBase64 round-trip
- SharedRecipe.test.js: formatCookDuration at 0, 11, 12, 36 months; formatMemberSince formatting
- cloudflare-function.test.js: meta tag injection for all 7 replaced tags including twitter property/name mismatch

Step 3 — Playwright E2E:
- Load /recipe/:token with a known public token; verify title, emoji, ingredients, method steps render
- Verify OG tags are set in SSR HTML response
- Verify /recipe/invalid-token shows error state, not blank screen

The utility layer (`src/utils/announcementNote.js`) is composed of three pure functions with no side effects: `resolveNote`, `resolveDietaryAllergyNote`, `resolveDietaryInfoNote`. The `wrapTitle` and `escapeXml` functions in the OG worker are also pure. The `useReveal` hook only wraps `IntersectionObserver`, which can be mocked. `useAnnouncementNoteManager` takes `recipe` and `dietaryInfo` as plain arguments and returns a plain object — trivial to test with Vitest + `renderHook`. Adding Vitest to `frontend` and a simple Node test runner to `og-worker` would provide coverage with minimal setup.
