# Omkara Data Room — Vite + React

A Vite + React port of the single-file **Daily Reading — Omkara Data Room** app.

## Setup

```bash
npm install      # install dependencies (not bundled in this zip)
npm run dev      # start the dev server at http://localhost:5173
npm run build    # production build -> dist/
npm run preview  # preview the production build
```

## How API calls work (proxy)

The app calls **relative** paths and lets Vite's dev-server proxy forward
them to the real upstreams, so the browser only ever makes **same-origin**
requests — no CORS preflight, no `Access-Control-*` requirements.

| App fetches      | Proxied to                          | Host          |
| ---------------- | ----------------------------------- | ------------- |
| `/api/*`         | `https://omkaradata.com/api/*`      | main data hub |
| `/occ-api/*`     | `https://omkaracapital.in/api/*`    | TV bytes      |

Two prefixes are used because both upstreams expose their endpoints under
`/api/...`; `/occ-api` keeps them unambiguous and is rewritten back to
`/api` on the way out (see `vite.config.js`).

### Production

The Vite proxy only runs in `dev`/`preview`. For a deployed build, add the
equivalent rewrite at your host. On **Vercel** (`vercel.json`):

```json
{
  "rewrites": [
    { "source": "/api/:path*",     "destination": "https://omkaradata.com/api/:path*" },
    { "source": "/occ-api/:path*", "destination": "https://omkaracapital.in/api/:path*" }
  ]
}
```

Or front the static build with Cloudflare / nginx doing the same path forwarding.

## Architecture

This is a pragmatic port that preserves the original behaviour 1:1. The
original is an imperative, DOM-owning SPA (~8,300 lines), so React acts as a
thin shell rather than a full component rewrite:

```
index.html          Vite entry — fonts, Chart.js CDN, #root, module script
src/
  main.jsx          Mounts <App/>, imports global styles (no StrictMode — see note)
  App.jsx           Renders the original markup once, then runs initLegacyApp()
  appMarkup.html    The original <body> markup (imported with Vite ?raw)
  legacyApp.js      The original <script>, wrapped in initLegacyApp() and exported
  styles.css        The original <style> block, verbatim
vite.config.js      React plugin + dev proxy
```

**Why a shell and not full components?** The original code owns its DOM
directly (`getElementById`, `insertAdjacentHTML`, manual render functions).
React mounts the markup exactly once via `dangerouslySetInnerHTML` and never
re-renders it, so the imperative layer keeps full control and behaviour is
identical to the original file. `initLegacyApp()` is idempotent, so it runs
its bootstrap (event listeners, modal injection, timers, first fetch) once.

### What changed from the original

1. API endpoint string constants rewritten to relative paths
   (`https://omkaradata.com/api/*` → `/api/*`,
   `https://omkaracapital.in/api/*` → `/occ-api/*`).
2. The `<script>` body wrapped in `initLegacyApp()` and exported.
3. `<style>` and `<body>` markup split into their own files.

Nothing else — the application logic is unchanged.

## Update — watchlist company listing (WatchList_AddCompany, input:4)

The `WatchList_AddCompany` endpoint is now also used in **list mode**
(`input:4`) to load the companies belonging to each watchlist. Previously the
app synced watchlist *names* from the server but read their *companies* only
from localStorage, so a watchlist populated elsewhere showed a stale/short
list (the reported Portfolio bug).

What fires now:

- **On page load / refresh** — after the watchlist names sync, one
  `WatchList_AddCompany` POST (`input:4`) is sent per server-backed watchlist,
  so each shows up in the Network tab. Payload:
  `{"ID":"","WatchListID":<id>,"AccordCode":"","CompanyName":"","status":false,"input":4,"UserID":"2"}`
- **On selecting a watchlist** (Settings -> Watchlists) or **activating its
  chip** (Daily Reading) — its companies are loaded from the server (cached
  per session) and shown.

Server rows map to the app's company shape (`BseCode` -> `BSECode`,
`AccordCode` used as the stable id, server row `ID` kept as the deletable
`watchlistEntryId`) and are **de-duped by AccordCode** (the server can return
the same company twice). Console helper: `window.listWatchlistCompanies(175)`.

### Update — one company call per selection on page load

Earlier the page-load sync fetched companies for *every* watchlist (one
`WatchList_AddCompany` input:4 POST each), so the Network tab showed several
on load. Now page load fetches companies for **only the selected watchlist**
— a single input:4 POST — and every other watchlist loads lazily when it is
selected (Settings list) or its chip is activated (Daily Reading).

The selected watchlist is persisted (`localStorage: omkara.wl.editing`) so a
reload keeps the same selection and the single call targets it. Tradeoff:
unselected watchlists' company counts come from the local cache (or show 0
until first selected after a cache clear), since accurate counts for all
watchlists would require one call per watchlist.

### Update — sidebar +1px and Daily Reading as the #home page

- The sidebar column width went from `200px` to `201px` in `.app`'s
  `grid-template-columns`. The sidebar, brand/heading, and nav all fill that
  grid column, so they widen by 1px with it; the `1fr` main content absorbs
  the remaining space. (Other `200px`/`1200px` values in the CSS are search
  inputs and company tables — unrelated to the sidebar.)
- The Daily Reading view is now the site's **home page**, reachable as
  `#home`. Clicking the **Daily Reading** nav item or the **branding/logo
  area** both navigate there. `#settings` deep-links to Settings. The view↔hash
  sync uses `history.replaceState` (no scroll jump, no routing loop).

### Fix — blank page on open (#home routing)

The initial hash-routing call ran too early in startup and invoked
`showView()` before the FAB element it touches was created, throwing a
temporal-dead-zone error that aborted the rest of init and left a blank page.
The initial route now runs at the very end of init (after the FAB and views
exist). Opening the app — or `#home` — shows the Daily Reading page as the
home page; `#settings` still deep-links to Settings. No separate landing page.

### Feature — Forensic page

A "Forensic" nav item now sits in the sidebar Quick Links, directly below
Daily Reading. Clicking it opens a Forensic landing (`#forensic`) that prompts
you to pick a company and focuses the top search bar. Selecting a company from
that search opens the existing Company page — with its header section (name,
NSE/BSE badges, sector/industry/ISIN, live-price panel) — directly on the
**Forensic** tab, and the Forensic nav stays highlighted. Navigating to Daily
Reading or Settings disarms the flow, so the normal search still opens
companies on Overview.

### Update — Forensic page is header-only

The Forensic page now shows ONLY the company name section. After picking a
company from the top search in the Forensic flow, the Company view opens with
just its `.co-header` card — name, NSE/BSE badges, sector · industry · ISIN,
description, and the live-price panel — with the tab bar and all panes hidden
(via a `cv-header-only` class on `#companyView`) and the
`Forensic_DetailedTables` API call skipped. The normal top-search company page
is unchanged: full tab bar, Overview rendering, and the Forensic tab + API.

### Feature — Forensic card enriched from companynote

On the Forensic page, selecting a company now calls `companynote`
(`POST /api/companynote { CompanyID }`, mapped from the search result's
`CompanyID`) and enriches the header card once it returns:

- **Company name** ← `companynote.CompanyName`
- **NSE / BSE chips** ← `companynote.NSEcode` / `BSEcode`, each a clickable
  deep-link (`NSELink` / `BSELink`) that opens the exchange page in a **new
  tab**. No link → plain chip. Existing chip colours are unchanged.
- **Sector · Industry · ISIN** ← from the search API (`SymbolMaster_WithCode`).
- The **description line is removed** on the Forensic card; the demo
  live-price panel is kept as-is.

The card renders instantly from the search result and is enriched when the
note lands (one call per selection, guarded by `CompanyID` against stale
responses). The normal top-search company page is unaffected — no companynote
call, plain (non-link) chips.

### Update — company website link on the Forensic card

The Forensic header meta line now leads with the company website, taken from
`companynote.WebSiteLink`. It renders as a globe icon + the address (e.g.
`www.sansera.in`) as the FIRST item, before Sector, in one line with
Sector · Industry · ISIN (dot-separated, wraps only if needed). It's a native
`<a target="_blank" rel="noopener noreferrer">` so it opens the company site
in a new tab. When `WebSiteLink` is absent the icon and link are omitted
entirely. Website link appears on the Forensic card only (companynote isn't
called on the normal company page).

### Feature — Forensic page "Single Page" tab (Forensic_DetailedTables)

Below the company header card on the Forensic page there's now a tab bar:
`Single Page | Ratios | Capital Structure | Directors and Auditor | Capital
History | Dividend History | ESOP`. Single Page is active; the other six are
greyed-out/disabled placeholders.

Single Page integrates `Forensic_DetailedTables` (`POST` with
`{ CompanyId, type }`, CompanyId mapped from the search result's `CompanyID`):
- A **Consolidated / Standalone** toggle. Opening Single Page loads `con` by
  default; `std` loads only when clicked. Each mode is cached, so toggling is
  instant after the first fetch (one request per type), with an in-flight
  abort + stale-response guard on company switch.
- All **10 tables stacked** one below the other, reusing the existing table
  renderers (Snapshot/Averages KPI grids; the 8 time-series tables with their
  green/red CAGR cells).
- A **sticky jump-chips bar** directly below the Snapshot table — one chip per
  table; clicking smooth-scrolls to that table and the bar stays pinned under
  the topbar while scrolling.
- `button_status` drives whether Standalone is available.

Scope: Forensic page only. The normal company page's Forensic tab is unchanged
(`#forensicPage` is hidden there).

### Polish — Single Page presentation redesign

The Single Page tables were rendering unstyled because the forensic table CSS
was scoped to the old `.cv-forensic` container; the new `#forensicPage` didn't
carry it. Fixed by scoping that proven styling into the page (the tables wrap
renders inside a `.fp-tables.cv-forensic` context), then layering:
- **Snapshot** → grouped metric cards (one card per API group, label → value
  rows; responsive grid).
- **Every table in its own white card** (border, rounded, padding, title).
- **Time-series tables** keep the pinned metric column + horizontal scroll,
  green/red CAGR tints, and metric tooltips; the (unneeded) vertical sticky
  header is dropped so it can't hide behind the topbar/chips.
- **Consolidated / Standalone** toggle restyled to a **white active pill** on a
  light track (was black).

No renderer rewrite and no extra API calls — pure CSS + a wrapper class, so
con/std stay cached and switching is still instant.

### Fixes — Single Page polish round 2

- Tab bar: removed the unused right-hand scroll affordance (tabs wrap instead).
- All tables: negative values now render in brackets, e.g. `(-45.90)` /
  `(-364.64)` (positives unchanged).
- Time-series summary-column group header is renamed per table: Fund Flow →
  "Cumulative", Working capital analysis → "Averages", Asset efficiency &
  Expense Analysis → "Cumulative/Average" (others keep "CAGR").
- Fixed the hidden first row + blank strip on the single-row-header tables
  (Capital structure, Du Pont, ShareHolding Pattern): the shared sticky-header
  offset (meant for two-row CAGR headers) was pushing single-row headers down
  30px. Headers are now static on the Single Page (tables are short and fully
  visible), and tables fit the card width with no horizontal scrollbar.
- Averages → Shareholding card heading now shows the latest filing period
  (e.g. "Shareholding (%) Mar-2026", the date in grey), sourced from the Single
  Page dataset.

### Fixes — Single Page polish round 3

- Fund Flow & Expense Analysis: removed the blanket green flood on the summary
  (3/5/10yr) columns. Cells are neutral/black by default; only the API-flagged
  cells (the condition rows — Cash from ops, CFO/EBITDA, FCF; and Income tax
  paid/expense) show green/red on number + background.
- Info ("i") tooltips now pop ABOVE the icon on the Single Page, so they no
  longer overflow the card bottom or force a scrollbar.
- Summary sub-column headers normalized to "3yrs / 5yrs / 10yrs" (fixes the
  "3Yrs" casing in Expense Analysis) across the CAGR / Cumulative / Average
  groups.
- ShareHolding Pattern: added a Quarterly / Yearly toggle (Quarterly default).
  Yearly filters to the March (…03) year-end columns only. Applies only to this
  table.

### Fixes — Single Page polish round 4

- Summary sub-column headers (3yrs / 5yrs / 10yrs) now render lowercase across
  all tables (the earlier rule lost a specificity tie to the shared uppercase
  header style; forced with !important). Group labels stay as-is.
- Negative values in the summary (3/5/10yr) columns show in red across the
  condition-coloured tables (Fund Flow, Expense Analysis, Asset efficiency).
  Rows carrying an "i"/API condition keep their API green/red instead.
- Asset efficiency joined the condition-coloured set: only Capex/EBIDTA(%)
  (the "i" row) keeps its colour; the other summary cells go neutral black.
- ShareHolding Pattern: the Quarterly / Yearly toggle now sits on the heading
  line, right-aligned.

### Fix — Working capital analysis 10yr neutral

- Working capital analysis joined the condition-coloured set. Its summary
  columns are no longer blanket-green: only the API-flagged cells carry colour.
  Since the API only flags the 3yr and 5yr averages (compared against the 10yr
  long-term baseline), the 10yrs column now renders neutral while 3yrs / 5yrs
  keep their green/red.
