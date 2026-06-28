MUST ISSUE LOG
prompt: Missing or ambiguous consent flows, unenforceable terms implied by API surface, missing user agreement checkpoints
path: must/TC-ISSUE-2026Q2.md
target: toifood/-ts-toifood-web

INSTRUCTION FOR AI MODEL:

YOU MAY READ AND UPDATE EXISTING ENTRIES AS REQUIREMENTS EVOLVE.
ADD NEW ENTRIES AT THE TOP FOR NEW TOPICS; UPDATE IN PLACE FOR EXISTING ONES.

FORMAT: ## ISSUE:{NAME} {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD OR UPDATE ENTRIES DIRECTLY BELOW THIS LINE -->## ISSUE:TC 2026-06-28 13:12 ▸ Terms incomplete: no pricing detail, no governing law, no age gate, stale date
## ISSUE:TC 2026-06-29 06:30 ▸ Terms incomplete: no pricing, no governing law, no age gate; dual privacy-doc risk added

Terms.jsx (Last updated: April 2026) four gaps persist:
1. **Pricing opacity** — Premium section says "Contact us for details" with no rates disclosed; FAQ contradicts by listing specific hourly limits.
2. **No governing law clause** — No NZ jurisdiction or dispute-resolution statement present.
3. **No age requirement** — Privacy.jsx states 13+ but Terms.jsx is silent; Apple/Google require explicit age gates in published T&Cs.
4. **Passive acceptance only** — "Continued use constitutes acceptance" with no affirmative consent or in-app update notification. Unenforceable for material changes under NZ Consumer Guarantees Act 1993.
New: `frontend/public/privacy.html` is a static HTML file separate from `Privacy.jsx` at /privacy. If these diverge, users may see different disclosures depending on Cloudflare routing — dual-source compliance risk.

Terms.jsx (Last updated: April 2026) has four material gaps:
1. **Pricing opacity** — Premium section says "Contact us for details" with no rates disclosed; FAQ contradicts by listing specific hourly limits. Rates must appear in T&Cs.
2. **No governing law clause** — No NZ jurisdiction or dispute-resolution statement present.
3. **No age requirement** — Privacy says 13+ but Terms is silent; Apple/Google require explicit age gates in published T&Cs.
4. **Passive acceptance only** — "Continued use constitutes acceptance" with no affirmative consent flow or in-app update notification. Unenforceable for material changes under NZ Consumer Guarantees Act 1993.
