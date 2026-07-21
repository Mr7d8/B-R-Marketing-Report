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
4. Add the favicon link to the new week's `<head>`: `<link rel="icon" type="image/png"
   href="/favicon.png">`, right after `<title>`. The favicon itself lives once at
   `bnr-marketing-reports/favicon.png` (site root via `vercel.json`'s rewrites), no need to copy
   it per week, just link to it.
5. Commit and push to `main` (this repo has no PR workflow by default, push directly unless told
   otherwise). Vercel redeploys automatically.
   - If working on a feature branch (e.g. a Claude Code Remote session bound to one), a plain
     `git push origin <branch>` can still result in GitHub auto-opening *and auto-merging* a PR
     from it asynchronously, main can advance out from under you between pushes. Before pushing
     to `main`, always `git fetch origin main` first and check whether it already contains your
     latest branch commit. Then `git checkout -B main origin/main`, `git merge <branch>` (not a
     bare fast-forward assumption), and `git push origin main`, switching back to the feature
     branch afterward so the session's working branch stays what it was.

## Report page structure (per week's index.html)

Fixed page order, each `<section class="page-section pb" id="...">`:
1. Summary / cover (`#p-summary`)
2. Business & Revenue (`#p-br`): GMV, SKU/brand/seller growth, sales goals, top sellers, category
   pies. Sales-team qualitative notes ("Les bonnes nouvelles" / "A Fixer") go here as an
   `.insight-grid` at the bottom.
3. App Growth (`#p-growth`): AppsFlyer KPI tiles (`.growth-cards.kpi-row`, flat single-metric
   cards) plus Conversion Funnel Overview (`.funnel-split`: funnel SVG left, "Weekly Highlights ·
   Performance Marketing" list right). **Always write this list yourself** from the week's own
   AppsFlyer/Mixpanel numbers already on the same page (budget, active users, sessions, funnel
   rates, retention, etc.), don't leave it as a pending skeleton just because no one supplied
   prose bullets separately. This is a legitimate data-driven summary of figures already
   verified elsewhere on the page, not a fabrication. This does not extend to Page 03's Weekly
   Highlights below, see that section for why.
4. Overview / Outside Digital Performance (`#p-overview`): Weekly Highlights, Plan of Action
   (`.plan-grid`), Blocking Points (`.insight-grid.cols-3`, red accent). Unlike Page 02's
   highlights, this section covers business/ops initiatives (campaign launches, program
   approvals, workstream status) that this report has no data source for. Leave it pending if
   no one supplied it, writing something here means inventing specific business claims, not
   summarizing numbers already in the report.
5. Top 5 Ads (`#p-topads`): Meta Ads ranking table plus clicks/CTR charts. Ranking columns are
   Spend, Impr., Clicks, CTR, **Cost/Result** (its own column, not folded into the Results
   label), Results. When Meta returns duplicate/generic ad names (e.g. several ads literally
   named "CRSL - 1"), fetch each ad's ad set name (one extra `ads_get_ad_entities` call at
   `level=adset`) and use it as disambiguating context in the display name, e.g. "CRSL - 1 ·
   High-Tech & Audio", so the ranking table and charts don't show indistinguishable duplicates.

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
- Rate label placement and wording: right-aligned, stacked directly beneath that stage's value
  (same `x`, `y` of the value line + 15), not centered inside the funnel body. Phrase each label
  as "`X%` of `<previous stage, lowercase, short>`" (e.g. "8.5% of installs", "15.5% of carts",
  "29.6% of checkouts"), not the generic "X% of previous". This applies to both the on-screen
  funnel SVG and the print-view funnel SVG, which otherwise share identical markup.
- Geometry: half-widths follow a sqrt scale relative to the top stage (`sqrt(value / value0)`),
  centered in the viewBox. See the current funnel SVG in any `wNN/index.html` for the worked
  example (stage y-positions evenly spaced, `maxHalfWidth` ~130, dividers between all but the
  final stage).
- Blending two sources into one funnel (e.g. Mixpanel for the top stages, AppsFlyer's
  `af_purchase` for the final one because Mixpanel didn't report a purchase stage that week) is
  fine, it's better than shipping a funnel with a stage missing. But always add a small note
  under the funnel spelling out which stages came from which source and why, plus any side
  annotations that came with the data but aren't funnel stages themselves (a ratio between two
  stages, a context-only count like "/checkout" visits). Don't let a blended funnel look like a
  single clean source.

## Export PDF / print

**The exported PDF is a separate, purpose-built layout, not a reflow of the screen page.**
`window.print()` renders a hidden `#print-view` block (plain HTML/CSS, three `.pv-page` divs,
`page-break-after: always` between them) while the interactive sections (`.masthead`, `main`,
`footer`) get `display: none` for print. `#print-view` itself is `display: none` by default and
only switched to `display: block` inside `@media print`, so it never shows on screen.

This replaced an earlier approach that tried to make the *same* interactive DOM reflow for print
(shrinking cards, converting grids to flex, chasing Chart.js resize timing, etc). That fought
Chrome's print engine on multiple fronts at once and kept breaking in new ways with every fix:
CSS grid containers refuse to split across a page break, so an oversized one jumps whole to the
next page and leaves the current one blank; Chart.js canvases don't resize themselves when the
print stylesheet changes their container, so charts overflowed or rendered tiny depending on
which way the CSS was adjusted; and `margin-top` is silently discarded by Chrome on whatever box
happens to land first on a new page, so spacing fixes that looked right in one spot failed in
another. A dedicated print layout sidesteps all of it: no shared CSS to fight, no canvases to
resize, no page-break math, because every `.pv-page` is curated to fit on one page by construction.

**Fixed to 3 pages**, each a `.pv-page` inside `#print-view`. Content here should match the web
version as closely as the 3-page ceiling allows, don't pre-emptively cut sellers/rows/prose to
save space, only trim if a render actually overflows to a 4th page:
1. Cover + Business & Revenue: KPI grid, full top-sellers list as CSS bars (`.pv-bar-row` with
   the `.pv-bars.compact` modifier for tighter row height when the list runs to ~10 sellers,
   width is the only per-bar value that changes), category mix as two CSS `conic-gradient` pies
   (`.pv-pie`, see below), sales team notes with the full "Les bonnes nouvelles" / "A Fixer"
   text (not a trimmed subset).
2. App Growth + Top Ads: KPI grid, the conversion funnel (the same hand-built SVG used on
   screen, copy it as-is since it's pure vector and prints reliably with no JS involved), Top 5
   Ads as a full `<table class="pv-table pv-table-ads">` with the same columns as the screen
   ranking (Spend, Impr., Clicks, CTR, Cost/Res., Results, in that order, don't drop Results
   itself when adding Cost/Res.) plus a small thumbnail per row (reuse the Cloudinary URLs, see
   "Ad thumbnail images" below), and two CSS bar-chart blocks below the table ("Clicks by ad" /
   "CTR by ad", `.pv-bars.compact.small` inside a `.pv-pie-row`) mirroring the screen's ad
   performance charts.
3. Overview & Operations: Weekly Highlights as a 2-column bullet list (`.pv-hl-list`), full Plan
   of Action grid, Blocking Points.

**Pie charts use CSS `conic-gradient`, not Chart.js.** A `background: conic-gradient(...)` on a
round `div` (`.pv-pie`) renders identically in print and on screen, no canvas, no resize
dependency, nothing that can go wrong between the two contexts, which is the entire reason this
print rewrite exists. To build one: cumulative-sum the category values, divide each cumulative
point by the *sum of all values* (not literally 100, category breakdowns rarely round to exactly
100) to get gradient-stop percentages, e.g. for values `[51, 19.2, 15.8]` summing to 96.4, stop 1
ends at `51/96.4*100 = 52.9%`. Pair each slice with a legend chip (`.pv-pie-legend`) showing the
*raw* value (51%, not the renormalized 52.9%) since that's the number sales actually reported.
Reuse the same 7-color palette as the screen charts (teal `#00c4cc`, blue `#00a3ff`, indigo
`#5b5fff`, violet `#7b61ff`, purple `#8b3dff`, mint `#00d9a3`, amber `#ffb020`) in category order
so the print and screen versions of the same chart use matching colors per category.

**Updating for a new week:** edit the numbers directly inside `#print-view` (near the end of the
file, before `</body>`) using the small `.pv-*` component classes already defined in the
`@media print` block (`.pv-kpi-grid`/`.pv-kpi`, `.pv-bar-row`, `.pv-pie`, `.pv-table`,
`.pv-note-box`, `.pv-plan-grid`, `.pv-hl-list`). Don't try to make the screen sections themselves
print well, that's the trap that caused all the churn above. If a week's content doesn't fit 3
pages, cut prose rather than shrinking fonts/padding to the point of illegibility, print
pagination has no slack for guessing.

**Verify by rendering the actual PDF** (Playwright `page.pdf()` plus `pypdfium2` to render pages
to images), not by eyeballing the screen view or trusting that reserved container space matches
what a chart/table actually draws inside it, print is a separate layout pass with its own quirks.

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

**Cloudinary connector gotcha:** the connector can show as `connected: true` at the org level
(via `ListConnectors`) while still being `enabledInChat: false` for the current session, in
which case its tools never load no matter how many times you re-check. Ask the user to enable it
for the current chat specifically (not just connect it), and don't loop on re-checking, ask once
and let them confirm before retrying. If it truly isn't available this session, build the Top 5
Ads page with real ranking data and a placeholder thumbnail icon rather than shipping a raw
`fbcdn.net` URL, then re-host the thumbnails in a follow-up once the connector is live.

**Local dev sandbox network limitations:** the environment this report is typically built in
(Claude Code sandbox) blocks outbound requests to `cdn.jsdelivr.net` (Chart.js/datalabels),
`*.fbcdn.net`, and `res.cloudinary.com` at the proxy level (`CONNECT tunnel failed, response
403`). This means local Playwright screenshots/PDF renders will show broken Chart.js canvases
and broken thumbnail `<img>` tags even when the code and the uploaded assets are both correct,
production (Vercel, real browsers) has no such restriction. Don't mistake this for a real defect:
confirm a Cloudinary upload succeeded from its API response (it returns `secure_url`, `bytes`,
etc. on success) rather than from a local screenshot, and confirm Chart.js configs are correct by
pattern-matching against a previous week's already-deployed, working charts.

## Chart.js pie/doughnut datalabels must not overlap

The GMV/SKU category pies (`chart-gmv-cat`, `chart-sku-cat`) show permanent `%` labels on each
slice via chartjs-plugin-datalabels, per the report-wide "always show values, no hover-only
labels" rule. With several small categories clustered next to each other (e.g. Maison 7.1%,
Beaut&eacute; 6.3%, Fashion femme 9.4% all adjacent), their centroid labels land close enough to
visually overlap and become unreadable, this happened for real in W29.

Fix: always set `display: 'auto'` on the `datalabels` block for pie/doughnut charts, alongside
the existing `formatter` cutoff that hides labels below a minimum share:

```js
datalabels: {
  display: 'auto',
  color: '#fff', font: { weight: '700', size: 10 },
  formatter: function(v) { return v < 5 ? null : v + '%'; }
}
```

`display: 'auto'` is a chartjs-plugin-datalabels feature that hides a label if it would overlap
an already-placed one, evaluated in dataset order, so keep category data sorted largest-first
(as it already is) so the labels that survive are the biggest, most important slices. The full
breakdown is never lost since the legend below each pie always lists every category and its
percentage regardless of which inline labels got auto-hidden. This applies to every pie/doughnut
chart in the report, not just this week's two.

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
