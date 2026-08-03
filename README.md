# magnus-ledger-audit-engine

 Magnus Ledger Audit Engine v3.0

Decoupled, serverless lead discovery & conversion-friction diagnostic pipeline built for automated B2B service-first audits.

 Overview

Magnus Ledger Audit Engine is an enterprise-grade lead generation and technical diagnostic tool. It discovers local service businesses via Google Places API (New), measures real-time DOM/ingestion latency, and feeds scraped website structure directly into Gemini 2.5 Flash to generate math-validated conversion friction analyses, personalized cold emails, and 60-second Loom video scripts.

 System Architecture

┌───────────────────────────────────────────────────────┐
│ 1. Google Places API                                  │
│    • Paginated Search (Up to 60 leads/city)           │
└───────────────────────────┬───────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────┐
│ 2. Web Latency Ingestion                              │
│    • DOM Scraper & Latency Benchmarker (ms)           │
└───────────────────────────┬───────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────┐
│ 3. Gemini 2.5 Flash Core                              │
│    • Structural Audit + CRO Math Models               │
└───────────────────────────┬───────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────┐
│ 4. Google Workspace Matrix                            │
│    • Live Leads Feed Auto-Logging & Deduplication     │
└───────────────────────────────────────────────────────┘

1. Google Places API
- Executes paginated searches across chosen cities.

- Returns up to 60 business leads per search run.

2. Web Latency Ingestion
Scrapes DOM elements from lead websites.

Measures response times in milliseconds to locate site slowdowns.

3. Gemini 2.5 Flash Core
- Audits site structure against extracted page code.

- Calculates conversion loss scores using statistical models.

- Drafts custom sales emails and Loom video scripts.

4. Google Workspace Matrix
- Removes duplicate records from lead lists automatically.

- Writes clean lead data into Google Sheets.

Key Features
- Paginated Discovery: Processes nextPageToken tags to find HVAC, roofing, and plumbing contractors.

- Automatic Deduplication: Saves normalized URLs in cache. This step prevents repeated scraping and API rate limits.

- Speed Benchmarking: Records page load speed. It flags sites with load times over two seconds.

- Structured JSON Audits: Enforces strict schemas in Gemini 2.5 Flash to generate:

  - Business vertical classification

  - Conversion barrier reports (Form bloat, broken mobile views)

  - Cold outreach emails (Under 100 words)

  - Timed Loom video scripts (60 seconds)

Security & Environment
This system stores API keys inside script properties to block public access.

- Places API Key: Saved under PLACES_API_KEY in PropertiesService.

- Gemini API Key: Saved under AI_STUDIO_API_KEY in PropertiesService.

- No Hardcoded Secrets: Scripts pull active keys at runtime.

Setup & Usage
- Configure Sheets: Create two tabs named Configuration and Live Leads Feed. Add keywords to cell A2 and cities to column B.

- Save API Keys: Open Project Settings in Google Apps Script. Add PLACES_API_KEY and AI_STUDIO_API_KEY to Script Properties.

- Run Pipeline: Refresh your sheet menu. Select Magnus Ledger Agent -> Run Lead Audit Pipeline.

Architected by Andrew Benedict | Founder & Lead Systems Architect at Magnus Ledger
