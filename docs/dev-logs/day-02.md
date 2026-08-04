# Magnus Ledger — Progress & Technical Architecture Log (V2.5)

**Date:** August 5, 2026  
**Project:** Magnus Ledger Lead Audit & CRO Diagnostics Pipeline  
**Author:** Systems Engineering / Founder Desk  
**Status:** V2.5 Asynchronous Unified Pipeline Active ($0 Cost Guardrails Verified)

---

## 1. System Overview

Magnus Ledger is an automated auditing tool for local service contractors (Plumbing, Roofing, HVAC). It fetches site performance metrics, identifies conversion friction points (such as slow load speeds or excessively long quote forms), and generates custom audit summaries and outreach emails using Gemini 2.5 Flash—all running within free API tier limits.

### Primary Goals
* Replace subscription SaaS audit tools with a self-contained Python script.
* Combine headless DOM scraping, local deterministic scoring formulas, and LLM text generation.
* Automatically log qualified leads and audit metrics directly to Google Sheets.

---

## 2. V1 Prototype Post-Mortem

An 8-hour stress test of the original Google Apps Script (GAS) prototype highlighted several runtime bottlenecks, prompting a shift to a local CPython setup.

### Technical Failures in V1 (GAS)
* **URL Parsing Failures:** Native `new URL()` in GAS V8 threw unhandled exceptions on URLs containing tracking parameters (such as `?utm_source=GMB`), causing valid target domains to drop silently.
* **Token Cap Breaches (HTTP 429):** Passing unparsed HTML payloads into Gemini consumed tens of thousands of tokens per page (due to inline CSS, SVGs, and SPA bundles), quickly exceeding Google AI Studio's Free Tier Tokens-Per-Minute (TPM) ceiling.
* **Execution Timeouts:** GAS enforces a hard 6-minute (360-second) timeout per execution, blocking multi-page scraping or batch processing.
* **Shared IP Blocking:** Requests coming from shared Google Cloud IP ranges frequently triggered anti-bot challenges (Cloudflare / AWS WAF).

---

## 3. V2.5 Architecture (CPython Setup)

The system was rewritten in Python 3.14 (`magnus_ledger_unified_pipeline.py`) as an asynchronous pipeline to remove runtime limits and handle parallel scraping tasks efficiently.

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                      MAGNUS LEDGER V2.5 PIPELINE                        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
        ┌────────────────────────────────────────────────────────┐
        │ 1. Budget Guardrail Gate ($0 Cost Enforcement)         │
        └────────────────────────────┬───────────────────────────┘
                                     │
                                     ▼
        ┌────────────────────────────────────────────────────────┐
        │ 2. Environment (.env) & Session UUID Generation        │
        └────────────────────────────┬───────────────────────────┘
                                     │
                                     ▼
        ┌────────────────────────────────────────────────────────┐
        │ 3. Google Places API (New) Paginated Text Search       │
        └────────────────────────────┬───────────────────────────┘
                                     │
                                     ▼
        ┌────────────────────────────────────────────────────────┐
        │ 4. 7-Day Idempotency Check (Local JSON Cache)          │
        └────────────────────────────┬───────────────────────────┘
                                     │
                                     ▼
        ┌────────────────────────────────────────────────────────┐
        │ 5. Multi-Tier Scraper                                  │
        │    • Primary: Playwright Async (DOM Ingestion)         │
        │    • Fallback: Firecrawl Markdown                      │
        └────────────────────────────┬───────────────────────────┘
                                     │
                                     ▼
        ┌────────────────────────────────────────────────────────┐
        │ 6. Local Scoring Engine                                │
        │    • Form Field Penalty Calculation                    │
        │    • Page Speed Latency Penalty                        │
        │    • Risk Index (1-10)                                 │
        └────────────────────────────┬───────────────────────────┘
                                     │
                                     ▼
        ┌────────────────────────────────────────────────────────┐
        │ 7. Gemini Model Router                                 │
        │    • Primary: gemini-2.5-flash                         │
        │    • Fallback: gemini-2.5-flash-lite                   │
        └────────────────────────────┬───────────────────────────┘
                                     │
                                     ▼
        ┌────────────────────────────────────────────────────────┐
        │ 8. Async Google Sheets Writer                          │
        └────────────────────────────────────────────────────────┘
```
## 4. Scoring & Diagnostic Formulas

To keep results consistent and reduce token usage, the pipeline calculates performance metrics using deterministic formulas before passing summary data to Gemini.

### 1. Form Field Penalty ($P_{\text{form}}$)
Calculates potential conversion drop-off when forms exceed an 8-field baseline ($N$ represents total `<input>`, `<select>`, and `<textarea>` elements):

$$P_{\text{form}} = 1 - (0.95)^{N - 8} \quad \text{for } N > 8$$

### 2. Page Speed Latency Penalty ($T_{\text{latency}}$)
Applies a compounding 20% penalty for every second of load delay over $2.0\text{s}$:

$$T_{\text{latency}} = 1 - (0.80)^{\text{Latency} - 2.0} \quad \text{for } \text{Latency} > 2.0\text{s}$$

### 3. Vulnerability Risk Index ($V_{\text{index}}$)
Assigns a technical score from 1 to 10 based on specific page issues:

* Load latency $> 2000\text{ ms}$: $+2\text{ pts}$
* Load latency $> 4000\text{ ms}$: $+2\text{ pts}$
* Input fields $> 8$: $+2\text{ pts}$
* Input fields $> 12$: $+2\text{ pts}$
* Missing SSL certificate (`http://`): $+1\text{ pt}$

---

## 5. Execution Controls & Safety Guardrails

Hard limits in the script ensure execution stays strictly within free service tiers:

* **Max Total Leads:** `MAX_TOTAL_LEADS_PER_RUN = 5` halts execution after 5 target sites.
* **Per-City Cap:** `MAX_NEW_LEADS_PER_CITY = 3` limits over-scraping single areas.
* **Payload Truncation:** `MAX_TEXT_PAYLOAD_CHARS = 4000` truncates DOM/Markdown text before sending to the LLM to prevent TPM overflow.
* **Request Pacing:** `PACING_DELAY_SECONDS = 4.0` keeps overall request rates well under 15 RPM.
* **Budget Safety Guard:** The `BudgetSafetyGuard` class monitors throughput and shuts down execution if thresholds are reached.

---

## 6. Test Run Results (August 5, 2026)

A verification run targeting "Plumber" and "Roofing Contractor" businesses in Miami, FL recorded the following output:

* **Discovery:** The Places API returned 37 initial listings.
* **Scraping:** Playwright successfully ingested target sites, while Firecrawl handled minimal or JavaScript-heavy pages (such as Miami 24/7 Plumbing).
* **Metrics Calculated:**
  * **Miami Roof-Tech:** 48 form fields detected $\rightarrow$ $87.15\%$ form penalty score.
  * **Miami 24/7 Plumbing:** $6.62\text{s}$ load latency $\rightarrow$ $64.36\%$ latency penalty score.
* **Guardrail Trigger:** The Budget Safety Guard stopped execution after processing exactly 5 leads, keeping total API cost at $0.00.

---

## 7. Setup & Configuration

### Environment Variables (`.env`)
Store required keys in a local `.env` file loaded via `load_dotenv_if_present()`:

```env
GOOGLE_PLACES_API_KEY=your_places_api_key
AI_STUDIO_API_KEY=your_gemini_api_key
SPREADSHEET_ID=your_google_sheet_id
FIRECRAWL_API_KEY=your_firecrawl_api_key
GOOGLE_APPLICATION_CREDENTIALS=g_creds.json

```
## 8. Milestone Status

| Milestone | Status | Details |
| :--- | :--- | :--- |
| **V1.0 GAS Post-Mortem** | Completed | Documented script limits, token consumption, and IP blocks. |
| **V2.0 Migration** | Completed | Replaced GAS with a local Playwright scraper and Pydantic validation. |
| **V2.5 Unified Pipeline** | Verified | Integrated multi-tier scraping, math formulas, Gemini routing, and Sheets sync. |
| **Environment Fixes** | Verified | Ported to Windows CPython, replaced `ulid` with native `uuid`. |
| **Safety Guardrails** | Verified | Confirmed script halts after 5 leads with 4s pacing delay in Miami test. |
| **Google Sheets Sync** | Ready | Awaiting service account access and SDK installation. |
| **Daily Workflow** | Next | Set up daily 1-hour schedule: Run Audit $\rightarrow$ Record Video $\rightarrow$ Send Draft. |
