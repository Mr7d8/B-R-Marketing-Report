# WBusinessMarketingReport (B&R + Marketing Report), Claude Memory

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
4. Commit and push to `main` (this repo has no PR workflow by default, push directly unless told
   otherwise). Vercel redeploys automatically.

## Report page structure (per week's index.html)

Fixed page order, each `<section class="page-section pb" id="...">`:
1. Summary / cover (`#p-summary`)
2. Business & Revenue (`#p-br`): GMV, SKU/brand/seller growth, sales goals, top sellers, category
   pies. Sales-team qualitative notes ("Les bonnes nouvelles" / "A Fixer") go here as an
   `.insight-grid` at the bottom.
3. App Growth (`#p-growth`): AppsFlyer KPI tiles (`.growth-cards.kpi-row`, flat single-metric
   cards) plus Conversion Funnel Overview (`.funnel-split`: funnel SVG left, "Weekly Highlights ·
   Performance Marketing" list right).
4. Overview / Outside Digital Performance (`#p-overview`): Weekly Highlights, Plan of Action
   (`.plan-grid`), Blocking Points (`.insight-grid.cols-3`, red accent).
5. Top 5 Ads (`#p-topads`): Meta Ads ranking table plus clicks/CTR charts.

Update the masthead nav links and the `Page 0N` chip in each `.page-label` if you add or reorder
pages. Keep the sticky masthead nav order in sync with section order.

## Conversion funnel data rules

The funnel is a hand-built SVG (not a charting library), rebuilt per week from whatever stage
figures are given. **Stages must be monotonically decreasing**: each stage's count must be at or
below the one above it. That's what makes the shape and the "% of previous stage" rate labels
honest.

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

Chrome's print engine **cannot split a CSS grid container across a page break**. If a grid
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
  rendering the exported PDF (Playwright `page.pdf()` plus `pypdfium2` to render pages to
  images). Don't just eyeball the screen view, print pagination is a separate layout pass.

**Chart.js canvases do not resize themselves for print.** Chart.js sizes each canvas to its
container once, at creation time, and never re-measures unless something calls `.resize()`. The
print stylesheet changes container widths (grid to flex, narrower columns), but the canvas keeps
its on-screen pixel size, so charts overflow their card or clip axis labels in the exported PDF.
Fix: every chart created via `safeChart()` is pushed into a global `allCharts` array, and
`beforeprint`/`afterprint` listeners call `.resize()` on all of them. Keep this pattern (the
`allCharts` array and both listeners) in the script block of every future week's report, and add
any new chart creation through `safeChart()` so it gets registered automatically.

**The ad ranking table's desktop column widths overflow the print page.** `.ads-ranking
.ranking-row`'s fixed-pixel columns add up to ~668px before the flexible "Ad" column gets any
room at all, more than A4's printable width. It also inherits `.card`'s generous desktop padding
and the 74px thumbnail, making it far too tall to share a page with anything else (this is why
"Clicks by ad" and "CTR by ad" used to each get shoved onto their own near-empty page). Fixed with
print-only overrides: a much narrower `grid-template-columns`, smaller row padding/thumb/fonts,
and a compact `.card { padding: 14px 16px; }` plus smaller `.chart-wrap` height so the ranking
table and both chart cards fit on one page. If you touch the ranking table's screen layout, check
whether the print column widths still add up under the (fixed, ~718px usable) A4 content width.

**Print header band.** The screen masthead (logo, nav links, Export PDF button) is hidden on
print and replaced by a slim, single-line, repeating header: `Wasal · WNN B&R + Marketing Report
· <date range>`, pinned with `position: fixed; top: 0;` so Chrome repeats it on every printed
page. The date range lives in a dedicated `<span class="print-date-inline">` inside
`.brand-lock` (hidden on screen via `display: none`, shown only in print) so it can be updated
independently of the visible brand text. Update that span's date range each week alongside the
other per-week date mentions (cover-kicker, cover-meta, footer, page-label).

**The fixed header overlaps content unless every card gets extra top spacing, and it must be
padding, not margin.** Two things stacked here, both confirmed by rendering the actual PDF and
inspecting it (pypdfium2 to page images), not by guessing:

1. A bigger `@page` top margin does *not* create clearance under the fixed header: `position:
   fixed` print elements and normal flowing content are both anchored to the page's *content box*
   (inside the margin), so growing the margin pushes the header and the content it would overlap
   down together, netting zero gap.
2. `margin-top` on the card itself doesn't work either, even though that seems like the obvious
   fix. Chrome's print engine discards a box's top margin by default whenever that box happens to
   be the first fragment on a new page (CSS Fragmentation's `margin-break: discard`, which
   Chromium doesn't let you opt out of since it never implemented the `margin-break` property).
   That's exactly the situation this needed to protect, so `margin-top` silently did nothing for
   whichever card started a page, while looking like it worked for cards that weren't at a page
   boundary.

The fix that actually works: `padding-top` instead of `margin-top`, since padding is not
discarded the same way. Apply `padding-top: 1.3cm` in print to every `page-break-inside: avoid`
card (`.card`, `.br-card`, `.growth-card`, `.funnel-card`, `.insight`, `.ranking`, `.kpi-card`,
`.key-finding`). This rule must be the *last* thing touching these classes' padding in the
stylesheet (equal specificity, source order decides), since several compact-sizing rules earlier
in the print block set the full `padding` shorthand for the same classes and would otherwise wipe
it out.

For the `.page-label` / `.cover-kicker` pills, don't put `padding-top` directly on the pill
itself (it distorts the rounded pill shape). Put `padding-top: 1.3cm` on `.page-section` instead:
a fragmented box's own padding-top only ever renders on its *first* fragment, never on later
continuation pages, which is exactly where these pills sit (once per section, right at the
start), so it reaches them without needing to touch the pill's own box.

Side effect: a bit of extra vertical gap wherever this padding lands, whether or not that
specific box turned out to be at a page boundary. That's a fair trade for guaranteed correctness,
since CSS genuinely cannot detect "is this element about to start a printed page" ahead of time.
If you add a new print-avoid-break component, add it to the `padding-top` rule too (at the end of
the print block), not just the `page-break-inside: avoid` one.

## Mobile responsiveness gotcha

CSS specificity ties are broken by **source order**, not by which media query "sounds" more
specific. A mobile override declared *before* a component's base rule in the stylesheet will
silently lose to that base rule even inside a matching `@media (max-width: ...)` block, because
the base rule appears later in the file. This caused real (invisible until screenshotted) bugs
here for `.br-row1/2/3`, `.funnel-split`, and `.growth-cards.kpi-row`.

Rule of thumb: put mobile/print overrides for a component **after** that component's own CSS
block in the file, not in an early shared media-query block. When adding a new responsive
override, grep for the component's base rule and confirm the override comes after it.

## Ad thumbnail images (Top 5 Ads page)

Meta Ads creative thumbnails often arrive as direct `scontent-*.fbcdn.net` URLs copied from Ads
Manager. These are signed, temporary CDN links (the `oh=`/`oe=` query params encode an expiry)
and go dead within days, which is why old weeks' thumbnails eventually show as blank/broken
placeholders even though the report itself still loads fine.

Fix: re-host each thumbnail on Cloudinary before it expires, right when building that week's
report. Use `upload-asset` with the `fbcdn.net` URL directly as `file`, since the upload happens
from Cloudinary's own servers, not wherever the report is being edited from, so it isn't subject
to whatever network policy blocks that host locally. Upload each ad's image once (folder
`wasal-bnr-report/wNN/`, one `public_id` per ad), then point both the `<img src>` and the
matching `data-lb-src` (they're the same URL twice per ranking row, once for the inline
thumbnail, once for the lightbox) at the returned `secure_url`. Do this before the original
signed URL expires, not after, since an already-expired URL can no longer be fetched by anyone.

## Style rules

- Brand system: Bricolage Grotesque font, teal-to-violet gradient (`--gradient` /
  `--gradient-h`), light mode, card-based layout with a 3px gradient top-bar accent on every
  card (`::before` pseudo-element). Match this exactly for new sections, don't introduce a new
  visual language.
- **Never use em dashes (`—` / `&mdash;`)** anywhere in report content, copy, or this file. Use a
  comma, a period, or "and" instead. En dashes (`–` / `&ndash;`) are fine for date ranges and are
  used throughout ("Jun 29 – Jul 5").
- Reuse existing CSS classes/components before inventing new ones: `.insight` (good/warn/bad
  accent cards), `.ops-list` / `.ops-list-sm` (bulleted highlight lists), `.growth-card` /
  `.growth-card.compact` (KPI tiles), `.br-card`, `.plan-grid`. Check the current week's
  `index.html` for the full class vocabulary before adding new CSS.
