# PrivateLedger

**On-device personal finance intelligence. Your money never leaves your machine.**

PrivateLedger reads your bank statements, categorises every transaction, answers questions about your spending, forecasts your cash flow, and scans receipts — and it does all of it **locally in your browser**. No account, no server, no upload. The language model itself runs on your device via WebGPU.

> Most finance apps send your statements to their servers. PrivateLedger has no server to breach, subpoena, or sell to — the whole pipeline runs on your machine.

## Live Demo: https://private-ledger-nishant.netlify.app/

---

## Table of contents

- [What it does](#what-it-does)
- [Why it's different](#why-its-different)
- [How it works](#how-it-works)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Running locally](#running-locally)
- [Deploying to Netlify](#deploying-to-netlify)
- [The Free / Pro model](#the-free--pro-model)
- [Configuration](#configuration)
- [Browser support](#browser-support)
- [Privacy](#privacy)
- [Roadmap](#roadmap)
- [Disclaimers](#disclaimers)

---

## What it does

| Feature | What it does | Free limit (per day) | Pro |
|---|---|---|---|
| **Import** | Reads `.csv` and `.pdf` bank statements, parsed entirely in the browser | Unlimited | Unlimited |
| **Auto-categorise** | Sorts every transaction into categories (Food, Transport, Bills…) | Unlimited | Unlimited |
| **Dashboard** | Money in / out / net, top category, a spend-by-category chart, full transaction table | Unlimited | Unlimited |
| **Ask** | Natural-language questions answered from your own numbers, by an on-device LLM | 5 | Unlimited |
| **Forecast** | Projects next month's spend per category from your recent averages | 2 | Unlimited |
| **Receipt scan** | On-device OCR of a receipt photo → extracts the total → adds a transaction | 3 | Unlimited |
| **Tax assistant** | General India tax guidance grounded in built-in reference notes | 3 | Unlimited |

Every feature is fully built and available in the free tier — Pro only lifts the daily caps and loads a larger model. Limits reset daily and are tracked in `localStorage`.

---

## Why it's different

Existing finance apps (Mint-style trackers and newer AI tools) upload your financial data to the cloud to analyse it. That creates a permanent privacy liability — a breach, a data-sharing deal, or a subpoena can expose your entire financial life.

PrivateLedger takes the opposite approach: **the intelligence comes to your data, not the other way around.** Parsing, categorisation, the language model, OCR, and forecasting all execute client-side. The privacy promise isn't a policy you have to trust — it's enforced by the architecture.

---

## How it works

```
  Your statement (CSV / PDF)
            │
            ▼
   ┌─────────────────┐     all in the browser
   │  Parse locally  │     (pdf.js / CSV parser)
   └───────┬─────────┘
           ▼
   ┌─────────────────┐
   │   Categorise    │     keyword rules → category per transaction
   └───────┬─────────┘
           ▼
   ┌─────────────────┐
   │   Dashboard     │     tiles + doughnut chart + table (Chart.js)
   └───────┬─────────┘
           ▼
   ┌───────────────────────────────────────────────┐
   │  Ask / Tax      →  on-device LLM (WebLLM/WebGPU) │
   │  Forecast       →  pure-JS projection            │
   │  Receipt scan   →  on-device OCR (Tesseract.js)  │
   └───────────────────────────────────────────────┘
```

**Retrieval for the chat:** when you ask a question, PrivateLedger computes a compact summary of your data locally (totals, per-category spend, largest transactions, date range) and passes only that summary to the on-device model as grounding. The model answers from your real numbers rather than guessing.

---

## Tech stack

- **Plain HTML / CSS / JavaScript** — no build step, no framework. Two static files.
- **[WebLLM](https://github.com/mlc-ai/web-llm)** (`@mlc-ai/web-llm`) — runs a quantized small language model in-browser via **WebGPU**. Default model: `Llama-3.2-1B-Instruct` (Pro uses `Llama-3.2-3B-Instruct`).
- **[pdf.js](https://mozilla.github.io/pdf.js/)** — extracts text from PDF statements.
- **[Chart.js](https://www.chartjs.org/)** — the spending chart.
- **[Tesseract.js](https://tesseract.projectnaptha.com/)** — on-device OCR for receipts.
- **Google Fonts** — Plus Jakarta Sans + Space Mono.

All libraries load from public CDNs. There is **no backend and no API key** anywhere in this project.

---

## Project structure

```
privateledger/
├── index.html    # Landing page (Tactile 3D design, animated hero, pricing)
├── app.html      # The full application (import, dashboard, ask, forecast, receipts, tax)
└── README.md
```

That's the whole app. Static files, nothing to compile.

---

## Running locally

Because the app uses ES modules and `fetch`, open it through a local server rather than double-clicking the file.

```bash
# from the project folder, pick any one:
python3 -m http.server 8000
# or
npx serve .
```

Then visit `http://localhost:8000` (landing) and `http://localhost:8000/app.html` (the app).

> The on-device model and OCR download model/engine files on first use. Give the first chat a minute — after that everything is cached in your browser.

---

## Deploying to Netlify

1. Push `index.html`, `app.html`, and `README.md` to a GitHub repo.
2. In Netlify: **Add new site → Import an existing project → GitHub** → select the repo.
3. Leave **build command** and **publish directory** empty — it's a static site.
4. Deploy. Your site is live at `https://<your-site>.netlify.app/`, and the app at `/app.html`.

Every push to the repo redeploys automatically.

---

## The Free / Pro model

The free tier includes every feature, each capped per day. **Pro** removes the caps and loads a larger model. Unlocking is done with a secret code (no payment integration in this version) — the "paid" features are fully present in the code and gated behind a flag, so a real payment step can be added later without touching the feature code.

- Enter the code via the **Unlock Pro** button.
- Default code: `LEDGER-PRO-2026` (change it — see below).
- The unlock state is stored in `localStorage` on the user's device.

---

## Configuration

All settings live in one `CONFIG` object near the top of the `<script>` in **`app.html`**:

```js
const CONFIG = {
  PRO_CODE: "LEDGER-PRO-2026",          // your secret Pro unlock code
  MODEL: "Llama-3.2-1B-Instruct-q4f16_1-MLC",   // free-tier model
  WEBLLM_URL: "https://cdn.jsdelivr.net/npm/@mlc-ai/web-llm@0.2.79/+esm",
  TESSERACT_URL: "https://cdnjs.cloudflare.com/ajax/libs/tesseract.js/5.1.1/tesseract.min.js",
  LIMITS: { ask: 5, forecast: 2, receipt: 3, tax: 3 }   // free daily caps
};
```

- **Change the Pro code** → edit `PRO_CODE`.
- **Change daily limits** → edit `LIMITS`.
- **Use a different / smaller model** → change `MODEL` to any ID from WebLLM's prebuilt list (smaller = faster download, less capable).

---

## Browser support

| Feature | Requirement |
|---|---|
| Import, categorise, dashboard, forecast | Any modern browser |
| Receipt OCR | Any modern browser (uses WebAssembly) |
| Ask + Tax assistant (the LLM) | **Chrome or Edge on desktop** with WebGPU |

The app detects missing WebGPU and shows a friendly "open in Chrome/Edge" message rather than failing silently. First model load downloads roughly 0.9 GB (free model), cached afterwards.

---

## Privacy

- Files are parsed in the browser and **never transmitted**.
- The language model runs locally — **no API calls, no logging**.
- OCR runs locally.
- The app works **offline** once the model and libraries are cached.
- There is no analytics, no tracking, and no server in this project.

---

## Roadmap

- Point the landing-page CTAs directly at `/app.html`.
- Persist imported transactions locally (IndexedDB) so they survive a refresh.
- Smarter PDF statement parsing for more bank formats.
- Optional Stripe **test-mode** checkout in place of the unlock code.
- Budgets and recurring-payment detection.

---

## Disclaimers

PrivateLedger is a demo project. It does **not** provide financial or investment advice. The tax assistant gives **general information only**, grounded in illustrative reference notes that may not reflect the current financial year — always verify with a qualified professional. Statement parsing is best-effort and should be checked against your actual statements.

---

*Built to run entirely on your device. Your data never leaves your machine.*
