# Marriott Point Value Calculator

A single-page web app that helps Japanese travelers decide whether to pay cash, use points, or combine both for a Marriott Bonvoy stay.

- **Live site:** https://shingo1983.github.io/marriott-point-calc/
- **Repo:** https://github.com/Shingo1983/marriott-point-calc (public)
- **Audience:** Japanese users. UI text is 日本語 only. Overseas city names should be 日本語 wherever practical.

## Tech stack
- Pure static site: `index.html` + `hotels.js` + `marriott-bonvoy.png`. No bundler, no framework, no backend.
- Deployed via GitHub Pages on every push to `main` — goes live in ~1–2 minutes.
- External libs loaded from CDN at runtime: Tesseract.js (OCR, `jpn+eng`).

## Tracked files
- `index.html` — the entire app (HTML + CSS + JS, ~2,300 lines)
- `hotels.js` — `window.HOTELS_DATA` (~1.2 MB, 4-level nested: Region → Country → City → `[name, marsha, urlPath]`). **Do not try to regenerate** — see Quirks below.
- `marriott-bonvoy.png` — header logo displayed in `.logo-mark`
- `.gitignore` — excludes raw data, Python build scripts, temp files, `.claude/`
- `CLAUDE.md` — this file

## Key code locations (all in `index.html`)
- `updateMarriottLink()` — builds the Marriott reservation deep-link. Endpoint MUST be `/reservation/availabilitySearch.mi` (not `availability.mi`). Required params: `numberOfAdults`, `numberOfChildren`, `childrenAges` (CSV), `numberOfRooms`, `fromDate`/`toDate` in `MM/DD/YYYY`, `useRewardsPoints=true`, `isRateCalendar=false`, `costView=AVG`, `clusterCode=none`. Removing any of these breaks guest count rendering on the Marriott page.
- `CITY_DISPLAY` (~300 entries) — maps English city keys to 日本語
- `GENERIC_CITY_TOKENS` (Set) — 3-letter marsha codes (TYO, LAX…) and generic noise tokens ("University", "Historic") → bucketed into "（その他）"
- `prettifyCity(key)` — translates/buckets city keys at display time
- `mergeCityGroups(rawCities)` — dedupes hotels by marsha across keys that map to the same display name
- `HOTELS_SUPPLEMENT` + `mergeSupplement()` — hotels missing from `hotels.js`, merged into `REGIONS` at load (currently the 7 Ritz-Carlton properties in Japan; marsha codes verified against official URLs). Add new missing hotels here, never in `hotels.js`.
- `showHotelList()` / `applyHotelFilter()` / `visibleHotels` — hotel dropdown with a name filter (`#hotel-filter`, shown when a group has > `HOTEL_FILTER_MIN` = 12 hotels). `onHotelChange` indexes into `visibleHotels`, not `currentHotels`.
- `loadPrefs()` / `savePrefs()` (`PREFS_KEY`) — persists adults / children / ages / currency in localStorage.
- `fetchLiveFx()` / `onCurrencyChange()` / `manualFx` — live FX from open.er-api.com (8 s timeout). Without live rates, switching currency swaps in the per-currency manual rate (or blank) instead of carrying the previous currency's rate over.
- `parseMarriottRates(text, nights)` — OCR text → `{cashOnly, cpCash, cpPoints, pointsOnly}`. First collapses spaces Tesseract inserts between Japanese characters ("ポイ ント" → "ポイント"), strips JP/EN dates, then classifies numbers: strong markers (`¥`/`\`/`$`/`€` right before, `円`/currency code right after → cash; `ポイント`/`points`/`pts` right after → points) beat weak context words (`合計`/`金額`/`total` vs `ポイント`), chosen by nearest distance. `獲得`/`earn` on the same line = earned-points noise (skip). Handles `1,234` / `1.234` / `1,234.56` / `1.234,56`; unseparated numbers (`784`) count only with an adjacent marker. "A + B" combo decides which side is cash by markers. Multi-night: picks the total over per-night (ratio ±2 %); member-rate labels win over non-member.
- `detectCurrency(text)` — picks the currency symbol/code that appears most in the OCR text; `applyOcrText()` switches `#currency-select` to it so yen screenshots aren't multiplied by a EUR/USD rate.
- `getOcrWorker()` / `runOcr()` — one Tesseract worker (`jpn+eng`, ~12 MB dictionary on first load) created once and reused; status strings are mapped to Japanese via `OCR_STATUS_JA`. Images can also be pasted (Ctrl+V) or dropped anywhere on the page.
- `calculate()` + `RECOMMEND_THRESHOLD = 1.5` — only recommends point redemption when yen-per-point ≥ 1.5. Below that, recommends "現金払い推奨".

## Quirks — don't fight these
- **Hotels data pipeline is not in the repo.** The original `build_hotels_js.py` / `scrape_hotels.py` / `hotels-raw.json` live only on the author's Dropbox, and Python isn't installed on his machine. Never try to regenerate `hotels.js`. Always apply display-time fixes in `index.html` (`CITY_DISPLAY`, `GENERIC_CITY_TOKENS`, `prettifyCity`, `mergeCityGroups`).
- City keys in `hotels.js` include ~726 three-letter marsha fallbacks and ~40 generic English tokens. These are already bucketed to "（その他）" at display time — don't try to clean the data.
- **`hotels.js` has zero Ritz-Carlton and zero Bulgari properties worldwide** (the scrape never covered ritzcarlton.com / bulgarihotels.com). Ritz-Carlton Japan is patched in via `HOTELS_SUPPLEMENT`; Bulgari is deliberately left out (not bookable with Bonvoy points). Other Ritz-Carltons are still missing.
- Elements styled with an explicit `display:` (e.g. `.ocr-actions { display:flex }`) ignore the `hidden` attribute unless there is a matching `[hidden] { display:none }` rule. Several already exist (`.help-modal`, `.nights-badge`, `.ocr-actions`, `.hotel-filter-wrap`) — add one whenever you give a `hidden`-toggled element its own `display`.
- STEP 4 history is a read-only sample hard-coded in `HISTORY_DEFAULT`. `loadHistory()` deletes the old `marriott-history-v1` localStorage key on purpose so stale saved copies never shadow new sample rows.
- Marriott deep-link must use `availabilitySearch.mi` with all secondary params kept (`isRateCalendar`, `costView`, `clusterCode`) even though they look redundant — removing them breaks guest count.
- Color scheme: **Marriott Bonvoy corporate — black `#141414` + peach-orange `#F28E5B`.** Defined in `:root` CSS custom properties. Don't reintroduce the old burgundy/gold/red palette.
- BEST badge / point-redemption recommendation only fires when value ≥ 1.5 yen per point. Below that the user prefers preserving points and paying cash.
- Logo is a real PNG (`marriott-bonvoy.png`), not an SVG recreation. Don't replace it with inline SVG.

## Local development
- No `npm install` or build step. Open `index.html` in a browser.
- The Claude desktop app's browser pane renders `file://` pages as a static snapshot **without** `hotels.js` (only 2 regions appear). For a real preview there, serve the folder over HTTP: an untracked `.claude/serve.ps1` (PowerShell `TcpListener` static server on port 8765) plus `.claude/launch.json` config `static` exist on the author's machine; recreate them if missing (PowerShell 5.1 only, no Node/Python).
- Author's shell is Git Bash (MINGW64) on Windows. Use forward-slash paths. No Node / Python / Bun runtimes available locally.
- For one-off JSON inspection: `powershell.exe -Command "..."` works (PowerShell 5.1 is present). But `ConvertFrom-Json` chokes on case-insensitive duplicate keys in the raw data — fall back to regex if needed.

## Deployment workflow
- `origin` remote has a PAT embedded in the URL (no expiry). `git push` works without credential prompts.
- Every push to `main` auto-deploys to GitHub Pages within 1–2 minutes.
- Commit message prefix convention: `feat:` / `fix:` / `chore:` / `style:` / `docs:`.
- Before pushing a non-trivial change, sanity-check by opening `index.html` in a browser — date + guest count → dropdowns → Marriott link → OCR → rate calculation.

## When the user asks for changes
- Japanese UI stays Japanese — no English strings in user-visible text.
- Prefer runtime fixes in `index.html` over touching `hotels.js`.
- Do not add build tooling, frameworks, or a backend.
- Dropdown / city edits → `CITY_DISPLAY` / `GENERIC_CITY_TOKENS` / `prettifyCity` / `mergeCityGroups`.
- OCR edits → `parseMarriottRates`; pass `daysBetween(state.checkin, state.checkout)` as `nights`.
- Marriott URL edits → `updateMarriottLink`; keep the mandatory param set.
- Calculator / recommendation edits → `calculate()` and `RECOMMEND_THRESHOLD`.
- Commit + push at natural task-complete points. The live deploy appears ~1 minute later.
