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
- `index.html` — the entire app (HTML + CSS + JS, ~1,700 lines)
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
- `parseMarriottRates(text, nights)` — OCR text → `{cashOnly, cpCash, cpPoints, pointsOnly}`. Uses Japanese+English markers: `¥` / `JPY` / `円` / `合計` / `金額` for cash; `ポイント` / `points` / `pt` for points; `獲得` signals earned-points noise (skip). For multi-night stays, picks "total" price over per-night when both appear (ratio match within ±2 %).
- `calculate()` + `RECOMMEND_THRESHOLD = 1.5` — only recommends point redemption when yen-per-point ≥ 1.5. Below that, recommends "現金払い推奨".
- Tesseract language is `"jpn+eng"` (see `runOcr()`). Japanese dictionary is ~12 MB on first load.

## Quirks — don't fight these
- **Hotels data pipeline is not in the repo.** The original `build_hotels_js.py` / `scrape_hotels.py` / `hotels-raw.json` live only on the author's Dropbox, and Python isn't installed on his machine. Never try to regenerate `hotels.js`. Always apply display-time fixes in `index.html` (`CITY_DISPLAY`, `GENERIC_CITY_TOKENS`, `prettifyCity`, `mergeCityGroups`).
- City keys in `hotels.js` include ~726 three-letter marsha fallbacks and ~40 generic English tokens. These are already bucketed to "（その他）" at display time — don't try to clean the data.
- Marriott deep-link must use `availabilitySearch.mi` with all secondary params kept (`isRateCalendar`, `costView`, `clusterCode`) even though they look redundant — removing them breaks guest count.
- Color scheme: **Marriott Bonvoy corporate — black `#141414` + peach-orange `#F28E5B`.** Defined in `:root` CSS custom properties. Don't reintroduce the old burgundy/gold/red palette.
- BEST badge / point-redemption recommendation only fires when value ≥ 1.5 yen per point. Below that the user prefers preserving points and paying cash.
- Logo is a real PNG (`marriott-bonvoy.png`), not an SVG recreation. Don't replace it with inline SVG.

## Local development
- No `npm install` or build step. Open `index.html` in a browser.
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
