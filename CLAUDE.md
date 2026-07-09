# WBusinessMarketingReport (B&R + Marketing Report) — Claude Memory

Weekly "B&R + Marketing Report" (Business & Revenue + App Growth + Meta Ads) for Wasal.
Sibling of the WSocialReport repo (same design system, same Vercel-on-push pattern), but a
separate report family, separate repo, separate deploy.

## Repo structure & deploy

- Publish directory: `bnr-marketing-reports/`. Root `vercel.json` rewrites `/` and `/*` into it.
- Static site, no build step. Vercel deploys straight from `main` on every push.
- Week folders are lowercase: `bnr-marketing-reports/w27/`, `w28/`, ... Each contains a single
  `index.html` (the week's report, renamed from whatever filename it arrived as).
- `bnr-marketing-reports/index.html` is the landing page: a card per week, **newest first**,
  each linking to `/wNN/`.

## Adding a new week

1. `mkdir bnr-marketing-reports/wNN`
2. Copy the incoming `wasal_week[NN]_bnr_marketing_report.html` into that folder as `index.html`.
3. Add a new card to `bnr-marketing-reports/index.html` at the top of `.reports-grid`, mark it
   `latest`, and demote the previous latest card to a plain card.
4. Commit and push to `main` (this repo has no PR workflow by default — push directly unless
   told otherwise). Vercel redeploys automatically.

## Report page structure (per week's index.html)

Fixed page order, each `<section class="page-section pb" id="...">`:
1. Summary / cover (`#p-summary`)
2. Business & Revenue (`#p-br`) — GMV, SKU/brand/seller growth, sales goals, top sellers, category
   pies. Sales-team qualitative notes ("Les bonnes nouvelles" / "A Fixer") go here as an
   `.insight-grid` at the bottom.
3. App Growth (`#p-growth`) — AppsFlyer KPI tiles (`.growth-cards.kpi-row`, flat single-metric
   cards) + Conversion Funnel Overview (`.funnel-split`: funnel SVG left, "Weekly Highlights ·
   Performance Marketing" list right).
4. Overview / Outside Digital Performance (`#p-overview`) — Weekly Highlights, Plan of Action
   (`.plan-grid`), Blocking Points (`.insight-grid.cols-3`, red accent).
5. Top 5 Ads (`#p-topads`) — Meta Ads ranking table + clicks/CTR charts.

Update the masthead nav links and the `Page 0N` chip in each `.page-label` if you add/reorder
pages. Keep the sticky masthead nav order in sync with section order.

## Conversion funnel data rules

The funnel is a hand-built SVG (not a charting library), rebuilt per week from whatever stage
figures are given. **Stages must be monotonically decreasing** (each stage's count must be ≤ the
one above it) — that's what makes the shape and the "% of previous stage" rate labels honest.

- If new data only covers some stages, do not silently keep old figures for the others: check
  whether the mix still decreases top-to-bottom. If it doesn't, say so and ask for the missing
  figures rather than shipping a funnel that implies more sign-ups than carts, etc.
- Rate label color thresholds (already in CSS via `var(--good)/var(--warn)/var(--bad)`):
  `>=25%` good (green), `10-24.9%` warn (amber), `<10%` bad (red).
- Geometry: half-widths follow a sqrt scale relative to the top stage (`sqrt(value / value0)`),
  centered in the viewBox. See the current funnel SVG in any `wNN/index.html` for the worked
  example (stage y-positions evenly spaced, `maxHalfWidth` ~130, dividers between all but the
  final stage).

## Export PDF / print

Chrome's print engine **cannot split a CSS grid container across a page break** — if a grid
doesn't fit the remaining space on a page, the whole grid jumps to the next page and leaves the
current one mostly blank. This bit us on the funnel and the Business & Revenue cards.

Fix already in place (`@media print` block in each report's `<style>`), keep it if you copy this
template forward:
- Multi-card row grids (`.br-row1/2/3`, `.br-pies`, `.growth-cards`, `.plan-grid`,
  `.insight-grid`, `.two-col*`, `.three-col`) are switched to `display: flex !important;
  flex-wrap: wrap;` for print, since flex children *can* wrap across a page break.
- Single-card components (`.card`, `.br-card`, `.growth-card`, `.funnel-card`, `.insight`,
  `.ranking`) keep `page-break-inside: avoid` so they still print intact.
- Print-only compact sizing (smaller fonts/padding, capped funnel SVG width ~240px, shorter chart
  heights) so blocks actually fit together instead of relying on avoiding breaks alone.
- If you add a new grid-based section, add it to both lists above, then verify by actually
  rendering the exported PDF (Playwright `page.pdf()` + `pypdfium2` to render pages to images) —
  don't just eyeball the screen view, print pagination is a separate layout pass.

## Mobile responsiveness gotcha

CSS specificity ties are broken by **source order**, not by which media query "sounds" more
specific. A mobile override declared *before* a component's base rule in the stylesheet will
silently lose to that base rule even inside a matching `@media (max-width: ...)` block, because
the base rule appears later in the file. This caused real (invisible until screenshotted) bugs
here for `.br-row1/2/3`, `.funnel-split`, and `.growth-cards.kpi-row`.

Rule of thumb: put mobile/print overrides for a component **after** that component's own CSS
block in the file, not in an early shared media-query block. When adding a new responsive
override, grep for the component's base rule and confirm the override comes after it.

## Style rules

- Brand system: Bricolage Grotesque font, teal-to-violet gradient (`--gradient` /
  `--gradient-h`), light mode, card-based layout with a 3px gradient top-bar accent on every
  card (`::before` pseudo-element). Match this exactly for new sections — don't introduce a new
  visual language.
- **Never use em dashes (`—` / `&mdash;`)** anywhere in report content or copy. Use a comma, a
  period, or "and" instead. En dashes (`–` / `&ndash;`) are fine for date ranges and are used
  throughout ("Jun 29 – Jul 5").
- Reuse existing CSS classes/components before inventing new ones: `.insight` (good/warn/bad
  accent cards), `.ops-list` / `.ops-list-sm` (bulleted highlight lists), `.growth-card` /
  `.growth-card.compact` (KPI tiles), `.br-card`, `.plan-grid`. Check the current week's
  `index.html` for the full class vocabulary before adding new CSS.
