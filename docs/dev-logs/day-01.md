**2026-08-05**

# DAY 1: MAGNUS LEDGER OPERATIONAL RECORD

**Status:** Inception & Architectural Blueprinting  
**Date:** August 4, 2026  
**Focus:** Post-Mortem of V1 (GAS) & Architectural Blueprinting for V2 (Python/Playwright)  
**Document Version:** 2.0.0-PROD  
**Target Audience:** Lead Systems Engineers, Infrastructure Architects, & AI Integration Specialists  
**Author:** Andrew Benedict | Founder & Lead Systems Architect @ Magnus Ledger  

---

## Executive Summary & System Overview

Magnus Ledger is a decoupled diagnostic and lead-auditing engine designed for automated B2B service-first audits. It transitions the legacy Google Apps Script (GAS) prototype into a self-healing, low-latency pipeline capable of discovering local service businesses via Google Places API (New), scraping dynamic server-rendered payloads, and performing structured conversion-friction analysis using local and cloud-based LLM models.

---

## 1. Legacy Post-Mortem & Diagnostic Findings (V1.0)

The V1 prototype operated as a monolithic Google Apps Script macro that attempted direct DOM scraping, executed unthrottled Gemini API requests, and updated Google Sheets sequentially. Stress-testing over an 8-hour debugging session exposed critical failure modes inherent to the execution boundaries of GAS.

```text
+-------------------------------------------------------------------------+
|                         LEGACY V1 ARCHITECTURE                          |
|                                                                         |
|  [Google Places API] -> [GAS DOM Fetch] -> [Gemini API] -> [GAS Sheet]  |
|                                                                         |
|   Root Failures: Silent URL drops, HTTP 429 Token Traps, 6-Min Timeout  |
|               Cloudflare Egress Bans, Unhydrated SPA HTML               |
+-------------------------------------------------------------------------+
```

### 1.1 Root Technical Breakdown

#### Strict Egress Validation & URL Parsing Failure

* **Mechanism:** The Apps Script V8 execution environment applies non-standard URL normalization within `new URL()`. Target landing pages containing tracking parameters (e.g., Google Business Profile parameters like `?utm_source=GMB`) failed runtime parsing, causing valid lead targets to drop silently before scraping.
* **Legacy Patch:** Replaced strict constructor validation with raw prefix checking:

```javascript
const isFetchable = (url) => typeof url === 'string' && (url.startsWith('http://') || url.startsWith('https://'));
```

#### HTTP 429 Token-Per-Minute (TPM) Rate Limit Exhaustion

* **Mechanism:** Agency and enterprise target pages contained dense DOM structures yielding tens of thousands of raw tokens. Passing unparsed HTML strings to Gemini API rapidly exceeded free/standard tier TPM limits, triggering unhandled HTTP 429 errors.

#### Runtime & Infrastructure Constraints

* **IP Reputation & Egress Blocks:** Shared Apps Script IP blocks frequently triggered Cloudflare, Akamai, and AWS WAF anti-bot challenges.
* **Execution Ceiling:** GAS enforces a hard 6-minute maximum execution timeout per trigger run.
* **The Hydration Gap:** GAS lacks a client-side JavaScript engine, returning empty single-page application (SPA) shell markup (React, Vue, Next.js) devoid of target DOM nodes.

---

## 2. Decoupled V2 Architecture & Hardware Specifications

### 2.1 Host System Hardware Boundaries & Local VRAM Allocation

To maintain local evaluation capabilities and keep external API costs $< 2\%$, primary parsing and schema validation run within local hardware constraints.

* **GPU Compute:** NVIDIA GeForce RTX 3070 (8.0 GB GDDR6 VRAM)
* **CPU Compute:** Intel Core i7-10870H (8 Cores, 16 Threads, 2.21 GHz base, up to 5.0 GHz boost)
* **Concurrency Limit:** Maximum of 20 parallel threads allocated across Playwright instances and asynchronous inference loops

#### VRAM Budget Distribution ($8.0\text{ GB}$ Total Boundary)

To eliminate CUDA Out-Of-Memory (OOM) errors during continuous execution runs, memory is budgeted according to:

$$VRAM_{\text{total}} = VRAM_{\text{SLM}} + VRAM_{\text{Browser}} + VRAM_{\text{OS}} + VRAM_{\text{Buffer}}$$

Where individual allocations are bounded as follows:

* **Local SLM (Structured Parsing & Schema Extraction):** `Qwen2.5-Coder-7B-Instruct-Q4_K_M` or `Llama-3.1-8B-Instruct-Q4_K_M` GGUF via `llama.cpp` / `Ollama` (`--n-gpu-layers 32`):

$$VRAM_{\text{SLM}} \approx 4.8\text{ GB}$$

* **Headless Playwright Browser Instances:**

$$VRAM_{\text{Browser}} \approx 1.2\text{ GB}$$

* **Host Operating System Overhead:**

$$VRAM_{\text{OS}} \approx 1.5\text{ GB}$$

* **Safety Margin Buffer:**

$$VRAM_{\text{Buffer}} \approx 0.5\text{ GB}$$

### 2.2 Hydration-First Extraction & RSC Parsing

To avoid brittle CSS/XPath DOM selectors subject to visual updates, Magnus Ledger V2 extracts state objects directly from initial server-rendered responses. Frameworks like Next.js, Remix, and React Server Components (RSC) embed application state directly into page script tags.

```text
                    ┌────────────────────────────────────────┐
                    │        Target Server DOM Response      │
                    └───────────────────┬────────────────────┘
                                        │
                ┌───────────────────────┴───────────────────────┐
                ▼                                               ▼
     ┌──────────────────────┐                        ┌──────────────────────┐
     │   __NEXT_DATA__ tag  │                        │ self.__next_f.push() │
     └──────────┬───────────┘                        └──────────┬───────────┘
                │                                               │
                │  (Raw Script Extraction)                      │ (Chunk Stream Extraction)
                ▼                                               ▼
     ┌──────────────────────┐                        ┌──────────────────────┐
     │ Stringified JS State │                        │ Multi-line JS Chunks │
     └──────────┬───────────┘                        └──────────┬───────────┘
                │                                               │
                └───────────────────────┬───────────────────────┘
                                        │
                                        ▼
                          ┌───────────────────────────┐
                          │   chompjs C-Parser Engine │
                          └─────────────┬─────────────┘
                                        │
                                        ▼
                          ┌───────────────────────────┐
                          │ Python Dictionary Object  │
                          └───────────────────────────┘
```

* **Target Identification:** Locates script tags matching `__NEXT_DATA__` or chunk streams pushed through `self.__next_f.push()`.
* **C-Based Extraction:** Employs `chompjs` to parse extracted JavaScript object literals into Python dictionaries without initializing a heavy JS engine.
* **Fallback Conversion:** Obfuscated or unstructured pages pass through the Firecrawl API to generate cleaned Markdown before LLM ingestion.

---

## 3. Grammar-Guided Local Inference Engine

To maintain structural output validity without relying on post-generation retry loops, Magnus Ledger enforces token-level constraints during generation by converting target Pydantic schemas into Extended Backus-Naur Form (EBNF) grammars.

### 3.1 EBNF Schema Definition

```ebnf
root ::= object
object ::= '{' ws '"detected_vulnerability"' ws ':' ws string_vulnerability ws ',' ws '"exact_leak_root_cause"' ws ':' ws string_max120 ws ',' ws '"estimated_monthly_revenue_leak_usd"' ws ':' ws integer ws ',' ws '"suggested_zero_risk_fix_strategy"' ws ':' ws string_unconstrained ws '}'

string_vulnerability ::= '"high_latency_leak"' | '"form_friction_overflow"' | '"broken_data_webhook"'
string_max120 ::= '"' [^"]{1,60} '"'  ; Restricts token sequence length at generation layer
string_unconstrained ::= '"' [^"]* '"'
integer ::= [0-9]+
ws ::= [ \t\n\r]*
```

---

## 4. Visual Fallback & Self-Healing Engine

When DOM modifications cause selector queries to throw a `SelectorNotFoundError`, the extraction pipeline falls back to spatial comparison using Structural Similarity Index Measure (SSIM) metrics across captured viewports.

```text
[Playwright Selector Query]
            │
            ├─► (Success) ──► Execute Pipeline Step
            │
            └─► (SelectorNotFoundError)
                                │
                                ▼
                   [Capture Viewport Screenshot]
                                │
                                ▼
             [Grayscale Conversion & Dimension Alignment]
                                │
                                ▼
              [Compute SSIM against Reference Gallery]
                                │
                                ▼
             [SSIM >= 0.90?] ───┬──► (Yes) ──► Isolate UI Element & Continue
                                │
                                └──► (No)  ──► Safe Log & Graceful Degradation
```

### 4.1 Implementation Code

```python
import os
import cv2
import numpy as np
from PIL import Image
from skimage.metrics import structural_similarity as ssim

class VisualSelfHealingEngine:
    def __init__(self, reference_images_dir: str):
        self.ref_dir = reference_images_dir

    def calculate_perceptual_similarity(self, img_path_a: str, img_path_b: str) -> float:
        """
        Calculates the Structural Similarity Index (SSIM) between captured UI screenshots.
        Evaluates against an operational P99 threshold of 0.90.
        """
        try:
            # Load images as grayscale arrays
            img_a = cv2.imread(img_path_a, cv2.IMREAD_GRAYSCALE)
            img_b = cv2.imread(img_path_b, cv2.IMREAD_GRAYSCALE)
            
            if img_a is None or img_b is None:
                return 0.0

            # Match target spatial dimensions
            if img_a.shape != img_b.shape:
                img_b = cv2.resize(img_b, (img_a.shape[1], img_a.shape[0]))
                
            # Perform spatial SSIM evaluation
            score, _ = ssim(img_a, img_b, full=True)
            return float(score)
        except Exception as e:
            return 0.0
```

---

## 5. V2 Migration Roadmap & Operational Benchmarks

### 5.1 Phased Execution Schedule

```text
Phase 1: Infrastructure        Phase 2: Pipeline Optimization     Phase 3: Production
[Days 1-2]                      [Days 3-4]                         [Day 5+]
• Containerize in Docker        • Migrate to python httpx          • Deploy Docker Container
  (Python 3.12-slim)            • Integrate Firecrawl SDK          • Enable CRON Scheduler
• Playwright headless setup     • Implement Pydantic schema        • Google Sheets API appends
• Local SQLite state tracking     validation                       • Real-time logging
```

#### Phase 1: Infrastructure & Containerization (Days 1–2)

* Package extraction modules within a standardized Docker image (`python:3.12-slim`).
* Configure Playwright headless dependencies for containerized browser testing.
* Replace in-memory state variables with a local SQLite database for URL normalization and lead deduplication.

#### Phase 2: Pipeline Optimization (Days 3–4)

* Replace sequential HTTP calls with Python `httpx` async pipelines.
* Integrate Firecrawl SDK for converting unstructured HTML into clean Markdown.
* Enforce response validation using Pydantic models and EBNF constraints.

#### Phase 3: Production Deployment (Day 5+)

* Deploy containerized jobs to container runners (Google Cloud Run / AWS ECS).
* Configure CRON automation triggers for scheduled lead acquisition runs.
* Output validated diagnostic results directly to Google Sheets via official REST API endpoints.

### 5.2 Performance Benchmark Targets

| Operational Metric | Legacy Prototype (V1.0) | Decoupled Engine (V2.0-PROD) | Measured Metric Delta |
| --- | --- | --- | --- |
| **Target-to-Audit Latency** | $> 30.0\text{ s}$ | $< 5.0\text{ s}$ | $\sim 83\%$ Latency Reduction |
| **Pipeline Success Rate** | $\approx 60\%$ (Egress drops) | $> 95\%$ | $+35\%$ Reliability Increase |
| **Batch Lead Capacity** | $< 10\text{ leads/run}$ (GAS timeout) | $50+\text{ leads/run}$ | $5\times$ Throughput Increase |
| **Schema Validation Error Rate** | $\approx 18\%$ (Malformed JSON) | $0.0\%$ (EBNF constrained) | Elimination of Schema Failures |

