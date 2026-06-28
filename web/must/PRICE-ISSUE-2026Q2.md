MUST ISSUE LOG
prompt: Pricing inconsistencies, missing currency validation, discount calculation errors, billing edge cases, unguarded price mutation paths
path: must/PRICE-ISSUE-2026Q2.md
target: toifood/-ts-toifood-web

INSTRUCTION FOR AI MODEL:

YOU MAY READ AND UPDATE EXISTING ENTRIES AS REQUIREMENTS EVOLVE.
ADD NEW ENTRIES AT THE TOP FOR NEW TOPICS; UPDATE IN PLACE FOR EXISTING ONES.

FORMAT: ## ISSUE:{NAME} {YYYY-MM-DD HH:MM} → {CONTENT}

####### <!-- ANCHOR MARKER - ADD OR UPDATE ENTRIES DIRECTLY BELOW THIS LINE -->## ISSUE:PRICE 2026-06-28 13:12 ▸ No pricing page; premium cost undisclosed; no refund or billing terms anywhere on site
## ISSUE:PRICE 2026-06-29 06:30 ▸ No pricing page; premium cost undisclosed; FAQ free-tier description inconsistent

Three gaps from 2026-06-28 persist; one FAQ inconsistency added:
1. **No price disclosure** — Actual subscription cost is absent from the entire website. App Store/Google Play listings may carry pricing but no reference or link exists on toifood.co.nz.
2. **No refund/cancellation policy** — Required by Apple App Store Review Guidelines (3.1.2) and Google Play billing policy for subscription apps.
3. **No billing cycle disclosure** — Monthly vs annual cadence, trial periods, and auto-renewal terms absent from all web pages.
4. **FAQ rate inconsistency** — FAQ id 'premium' states "Free users get 2 Premium recipes per hour" (omitting the 3 Basic allowance), while FAQ id 'generation-limit' gives the full breakdown (2 Claude + 3 Basic / 5 Claude + 10 Basic). The partial statement in 'premium' could mislead users about their free tier entitlements.

The web frontend has no pricing page and Terms.jsx explicitly defers pricing to email ("Contact us for details on available premium tiers and pricing"). Critical gaps:
1. **No price disclosure** — Actual subscription cost is absent from the website entirely. App Store/Google Play listings may carry pricing but no reference or link exists on toifood.co.nz.
2. **No refund/cancellation policy** — Required by Apple App Store Review Guidelines (3.1.2) and Google Play billing policy for subscription apps.
3. **No billing cycle disclosure** — Monthly vs annual cadence, trial periods, and auto-renewal terms are absent from all web pages.
FAQ.jsx (id: generation-limit) documents usage limits but not cost — the only pricing-adjacent content on the site.
