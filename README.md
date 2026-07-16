# 🕵️ Competitor Intelligence & Pricing Monitor

> Automated competitor price tracking and internal price benchmarking — built entirely on free-tier tools.

This project monitors competitor pricing for refurbished electronics across multiple product conditions (Like New, Excellent, Very Good, Good), compares it against internal catalog prices, and automatically alerts the team when the company's price is equal to or higher than a competitor's — a real risk in a market where price is the primary purchase driver.

> **Note on privacy**: all examples below use a placeholder domain (`competitor-site.com`) instead of the real target. The techniques are general-purpose; the specific competitor isn't the point.

## 📐 Architecture

```
Scenario 1 — Scraper
Google Sheets (competitor_urls)
        │  Search Rows → loops once per tracked URL
        ▼
HTTP (ScraperAPI, render=true)
        │
        ▼
Text Parser (Match Pattern — regex extraction)
        │
        ▼
JSON (Parse JSON — variant array)
        │
        ▼
Iterator (one row per product condition)
        │
        ▼
Google Sheets (market_intelligence_log) — Add a Row

Scenario 2 — Price Comparison & Alerting
Iterator (from market_intelligence_log)
        │
        ▼
Tools (Set multiple variables — normalize price fields)
        │
        ▼
Router
   ├── Google Sheets (Search Rows on source_product) → match by product + condition
   │         │
   │         ▼
   │   Google Sheets (price_compare_log) — Add a Row (price_difference %)
   │         │
   │         ▼
   │   Router → if our_price >= competitor_price → Gmail (Price Alert)
   └── Google Sheets (Add a Row) — fallback logging
```

## 🧱 Why this exists

Manually checking competitor prices across multiple products and conditions doesn't scale. This pipeline runs on a schedule, logs every price point over time, and only interrupts a human when something actually needs attention — a price that's no longer competitive.

## 🚧 The hard part: scraping a site with active anti-bot protection

This was the main technical challenge of the project, and worth documenting because it shaped every architectural decision below.

The target competitor's product pages are protected by a **proof-of-work JavaScript challenge** (a "Sloth VDF" — Verifiable Delay Function — running in a Web Worker). Instead of returning the product page, the server first returns a challenge page that must be *solved by executing JavaScript in a real browser* before a valid session is granted.

This ruled out plain HTTP requests entirely. Over the course of building this, the following approaches were tested, in order:

| Method | Result |
|---|---|
| Make.com `HTTP > Make a request` | Blocked — `429 Too Many Requests` / challenge page returned |
| Google Apps Script `UrlFetchApp` | Blocked — same challenge page, confirming it's not User-Agent-based |
| Google Sheets `IMPORTXML` | Partially worked (Google's crawler IP), but hit a **50,000-character result limit** on the product JSON blob, and became unreliable under repeated testing |
| Google Sheets `IMPORTDATA` | Blocked — returned the same challenge/monitoring script |
| **ScraperAPI with `render=true`** | ✅ **Works** — runs a real headless browser that solves the JS challenge before returning the rendered page |

**Key takeaway**: a challenge that requires executing JavaScript cannot be defeated by changing headers or switching HTTP clients — it requires an actual browser execution environment somewhere in the pipeline. This is the architectural reason `render=true` is non-negotiable here, even though it costs significantly more API credits (~10x) than a plain request.

## 🛠️ Scraping setup (ScraperAPI)

1. Sign up at [scraperapi.com](https://www.scraperapi.com) and grab your API key from the dashboard.
2. Free trial credits are one-time (not monthly), and `render=true` requests consume roughly **10x** the credits of a standard request — budget accordingly (a few thousand renders is enough to build and validate a pipeline, not to run it continuously forever).
3. Request format:
   ```
   https://api.scraperapi.com/?api_key=YOUR_KEY&url=<encoded_target_url>&render=true
   ```
4. In Make, the target URL should be URL-encoded before being appended:
   ```
   https://api.scraperapi.com/?api_key=YOUR_KEY&url={{urlencode(2.target_url)}}&render=true
   ```
5. Increase the HTTP module's timeout to 60–90 seconds — rendered requests take significantly longer than plain HTTP calls (5–15s+) because the headless browser has to load and solve the challenge first.
6. Set `Parse response` to **No** — the response is HTML, not JSON, at this stage.

## 🔍 Extracting the pricing data

The product's variant data (price, stock quantity, condition) is embedded as JSON inside a `<script>` tag in the server-rendered HTML — not exposed through a separate clean API endpoint. It's extracted using a Match Pattern (regex) module:

```regex
"variants":(\[[^\]]*\]),"has_variant_in_stock":(true|false)
```

The extracted array is then parsed with a `JSON > Parse JSON` module and iterated to produce one row per product condition (e.g. label `A0`–`A4` mapping to Like New / Excellent / Very Good / Good / Heavily Used), each carrying its own price and stock quantity — `quantity > 0` is used as the reliable in-stock signal, rather than relying on UI text.

## ⚠️ A note on regex robustness

Extracting from an embedded JSON blob via regex is inherently fragile — if the competitor changes field order or nesting, the pattern breaks silently. This is a known trade-off of the current version; a more resilient version would parse the full JSON payload and traverse it by key rather than pattern-matching a substring.

## 📊 Dashboard (Looker Studio)

Connected directly to the Google Sheets logs, with four views:
- **Competitor Price Trend** — line chart of competitor pricing over time, filterable by product/condition
- **Price Benchmark** — table comparing our price vs. competitor price vs. % difference, with conditional heatmap formatting
- **Competitor Overview** — raw scrape log: stock quantity, availability, condition, price drop flags
- **Alert Log** — record of every price-alert email triggered, for tracking how often the team was notified

## 🧪 Lessons for anyone building something similar

- Test anti-bot behavior *before* designing the rest of the pipeline — it changes which tools are even viable.
- Different Google services (`IMPORTXML`, `IMPORTDATA`, Apps Script `UrlFetchApp`) run on different underlying IP ranges and are **not interchangeable** for bypass purposes, even though they're all "Google."
- In Make, arithmetic must be written as a single expression inside one variable reference — chaining separate variable pills with an operator typed in between concatenates them as text instead of calculating. Always resolve the full expression as one unit, not as two separate references joined by a typed operator.
- Repeated testing against a rate-limited endpoint compounds the problem — space out test runs, and expect to occasionally wait out a temporary IP flag rather than debug further.

## 🚀 Setup

1. Import both Make scenarios (Scraper + Price Comparison).
2. Connect Google Sheets and point each module at your own spreadsheet (`competitor_urls`, `market_intelligence_log`, `source_product`, `price_compare_log`).
3. Add your ScraperAPI key to the HTTP module.
4. Connect Gmail (or swap for Slack/Telegram) for the alert step.
5. Connect the Google Sheets logs to a new Looker Studio report for the dashboard.
6. Start with `Run once` to validate each step before enabling a schedule.

## 🙏 Disclosure

This was built as a hands-on learning project with the help of AI assistants (Gemini, Claude) — not written from scratch solo. Sharing it as-is, warts and debugging detours included, because that's a more honest picture of how this kind of build actually goes.

## 🏷️ Tags

`#Automation` `#CompetitiveIntelligence` `#MakeCom` `#WebScraping` `#DataQuality` `#GoogleSheets` `#LookerStudio`
