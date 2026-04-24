# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Deployment workflow

Every change follows this sequence — no exceptions:

```bash
wrangler deploy
git add <files> && git commit -m "..." && git push origin main
```

Never ask for confirmation before deploying or pushing. Both steps always happen together.

## Project overview

**FTM Trafik Ceza Rehberi** — a Turkish traffic fine lookup tool for Fahri Trafik Müfettişleri (honorary traffic inspectors). Users describe a traffic violation in plain Turkish; the app finds the matching KTK (Karayolları Trafik Kanunu) article(s) and whether FTM or only police (kolluk) can issue that fine.

Live URL: `https://ftm-madde-bulucu.mnuman.workers.dev/`

## Architecture

**Two-file application:**

- **`worker.js`** — Cloudflare Worker. Accepts POST `{query: string}`, sends it to Claude Haiku with a detailed system prompt containing the full article list, returns `{maddeler: [{no, pct, gerekce}]}`. Uses `env.ANTHROPIC_API_KEY` (set in Cloudflare dashboard). Model: `claude-haiku-4-5-20251001`.

- **`index.html`** — Single-file frontend (HTML + CSS + JS, no build step). Deployed separately via Cloudflare Pages (triggered by `git push`). The Worker only handles POST — GET returns 405. Contains:
  - `CEZA_DATA` object: all ~130 KTK articles with TL fine, ceza puanı, FTM eligibility, kanun metni
  - `CHIP_RESULTS` object: hardcoded AI results for the 12 quick-select chips (bypasses Worker)
  - `EGM` object: 4-month traffic statistics (Ara 2025–Mar 2026) for the hidden dashboard
  - `WORKER_URL` / `GITHUB_BASE`: the Worker endpoint and raw GitHub URL for fetching detail JSONs
  - `buildCard()`: renders a result card; `renderResults()`: splits results into FTM vs. kolluk sections
  - `openDrw()` / `renderDrw()`: bottom sheet drawer that fetches `data/<madde-no>.json` for detail view

**Supporting files:**

- `data/<madde-no>.json` — Detail JSON for each article (e.g. `data/47-1-b.json`). Slashes converted to hyphens in filename. Fetched on drawer open from GitHub raw. Top-level keys: `madde_no`, `baslik`, `kanun_metni`, `ozet`, `ceza`, `istatistikler`, `ilgili_maddeler`, `kaynaklar`. Application cases field varies by file: `uygulama_durumlari` (older files) or `uygulama_alani` (newer files) — `renderDrw()` handles both.
- `data/sozler.json` — 99 traffic slogans; one random slogan displayed in the logo area.
- `levhalar/` — Traffic sign PNGs referenced in detail JSONs.
- `istatistik.html` — Standalone statistics page (separate from the main app).
- `wrangler.toml` — Worker name: `ftm-madde-bulucu`, entry: `worker.js`.

## Key conventions

**Article no → filename:** slashes become hyphens. `47/1-b` → `data/47-1-b.json`.

**FTM vs. kolluk:** `CEZA_DATA[no].ftm === true` means FTM can issue the fine. Cards render with green stripe (FTM) or orange stripe (kolluk). The system prompt in `worker.js` is the authoritative source of FTM eligibility.

**CSS:** single `<style>` block, no preprocessor. All CSS variables defined in `:root`. Do not append duplicate rules — edit the existing rule instead.

**JS syntax checker** (use before deploy when editing JS in index.html):
```bash
node -e "
const fs=require('fs');
const s=fs.readFileSync('index.html','utf8').match(/<script>([\s\S]*?)<\/script>/)[1];
try{new Function(s.replace(/[\uD800-\uDFFF]/g,'_').replace(/async\s+function/g,'function').replace(/await\s+/g,''));console.log('OK');}catch(e){console.log('HATA:',e.message);}
"
```

**Editing index.html:** the file is large (~110KB). Use `node -e` scripts for complex replacements to avoid CRLF issues on Windows. For simple edits, the Edit tool works fine.

**`gerekce` field:** max 2 short sentences, plain citizen language, no bureaucratic phrases like "Bu madde kapsamında..." or "söz konusu fiil...". Defined in `worker.js` line ~188.
