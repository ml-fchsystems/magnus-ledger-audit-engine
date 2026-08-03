# magnus-ledger-audit-engine

 Magnus Ledger Audit Engine v3.0

Decoupled, serverless lead discovery & conversion-friction diagnostic pipeline built for automated B2B service-first audits.

 Overview

Magnus Ledger Audit Engine is an enterprise-grade lead generation and technical diagnostic tool. It discovers local service businesses via Google Places API (New), measures real-time DOM/ingestion latency, and feeds scraped website structure directly into Gemini 2.5 Flash to generate math-validated conversion friction analyses, personalized cold emails, and 60-second Loom video scripts.

 System Architecture

┌───────────────────────────┐
│ 1. Google Places API      │ ──> Paginated Search (Up to 60 leads/city)
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│ 2. Web Latency Ingestion  │ ──> DOM Scraper & Latency Benchmarker (ms)
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│ 3. Gemini 2.5 Flash Core  │ ──> Structural Audit + CRO Math Models
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│ 4. Google Workspace Matrix│ ──> Live Leads Feed Auto-Logging & Deduplication
└─────────────┬─────────────┘


Pipeline Stage Details

Google Places API (New)

Executes paginated search queries across targeted municipalities.

Filters and returns up to 60 qualified business leads per city run.

Web Latency Ingestion

Scrapes DOM elements and evaluates core site response metrics in milliseconds.

Identifies initial technical bottlenecks and rendering friction.

Gemini 2.5 Flash Core

Runs structural content audits against extracted DOM payloads.

Applies conversion rate optimization (CRO) scoring models to produce audit output, customized outreach emails, and video scripts.

Google Workspace Matrix

Automatically deduplicates incoming lead lists.

Records lead data and audit metrics directly to Google Sheets for real-time tracking.

 Key Features

Paginated Multi-City Discovery: Native nextPageToken processing targeting local service verticals (HVAC, Roofing, Plumbing).

Automated Deduplication Engine: Caches normalized URLs locally to prevent redundant scraping cycles and rate-limiting.

Ingestion Latency Benchmarking: Captures DOM load speeds to pinpoint performance drop-offs (>2.0s latency thresholds).

Gemini 2.5 Flash Structured JSON Output: Uses strict schema validation to generate:

Industry Vertical Identification

Conversion Friction Gap Analysis (Form proliferation, mobile viewport bugs)

Low-Pressure Cold Outreach Drafts (<100 words, high response hook)

Sequential 60-Second Loom Video Scripts with explicit timecodes

 Security & Environment Architecture

To protect API credentials from public exposure, this engine uses zero-trust script property bindings:

Google Cloud Places API Key: Stored under PLACES_API_KEY via PropertiesService.

Gemini AI Studio API Key: Stored under AI_STUDIO_API_KEY via PropertiesService.

No Hardcoded Keys: Code files only reference key parameters dynamically at runtime.

 Setup & Usage

Sheet Setup: Create two tabs: Configuration (Keywords in A2, Cities in Column B) and Live Leads Feed.

Key Storage: Open Project Settings (⚙️) in Google Apps Script and add PLACES_API_KEY and AI_STUDIO_API_KEY to Script Properties.

Execution: Refresh your Google Sheet and select 🤖 Magnus Ledger Agent -> ⚡ Run Lead Audit Pipeline.

Architected by Andrew Benedict | Founder & Lead Systems Architect @ Magnus Ledger
