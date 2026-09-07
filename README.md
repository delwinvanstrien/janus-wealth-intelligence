# Janus — Relationship Manager Wealth Intelligence System

> **An auditable, deterministic intelligence platform for Private Banking Relationship Managers that turns complex portfolio movements, mandate breaches, and market shocks into defensible, client-ready actions.**

*Originally created for the Julius Baer Wealth Intelligence Challenge (SingHacks 2026).*

---

## Overview

In private wealth management, relationship managers (RMs) oversee books of dozens of high-net-worth clients, navigating hundreds of positions, strict regulatory mandates, and sudden market volatility. Synthesizing these movements manually across core banking files is time-prohibitive, while relying on generic generative AI introduces hallucination risks that violate banking compliance.

**Janus** bridges the gap between raw portfolio accounting and high-conviction client communications. It replaces black-box summaries with a **zero-hallucination, deterministic computation engine** paired with strictly guardrailed narrative synthesis.

### Key Capabilities

- **Exact Mathematical Decomposition**: Every dollar of portfolio change is decomposed into exact trading, price, and FX effects that sum *identically* back to observed moves, asserted in code.
- **Explainable Client Triage**: Proactively ranks an RM's entire client book by urgency, surfacing who to call first with transparent, auditable rationales.
- **Look-Through Concentration & Mandate Tracking**: Traverses structured products and funds to uncover true underlying asset exposure, distinguishing authorized client waivers from silent mandate drift.
- **Strictly Guardrailed LLM Narration**: A language model only narrates pre-computed financial facts and citations—it is structurally blocked from fabricating numbers or causes.
- **Quarantined External Market Color**: Live financial press search (Reuters, Bloomberg, FT, etc.) is isolated in a sandboxed lane, preventing external commentary from contaminating verified portfolio numbers.
- **Dual Presentation Surfaces**: An interactive full-featured workbench (NiceGUI) and a decoupled headless REST API (FastAPI) with a lightweight static client.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| **Core Engine & Data Processing** | Python 3.13, Pandas, NumPy, Pydantic |
| **Primary Web Workbench** | NiceGUI (reactive Python UI with Tailwind CSS) |
| **API & Microservice Layer** | FastAPI, Uvicorn (REST JSON endpoints, CORS enabled) |
| **LLM Orchestration & Search** | OpenAI GPT-4o (scoped tool-calling loop, strict guardrail prompts), DuckDuckGo Search |
| **Testing & Quality Assurance** | Pytest (golden regression tests, exact arithmetic reconciliation) |

---

## The One Principle Everything Obeys

> **A deterministic engine reasons. A language model only narrates. Nothing on screen is a number or a cause the engine didn't compute and cite. When the data can't explain a move, the system explicitly reports "unexplained" — it never free-associates a cause.**

Every dollar of portfolio change is decomposed by exact arithmetic (`trading_effect + price_effect + fx_effect = total_delta_usd`), verified by automated test assertions. Every cause cited alongside it comes from a transparent, word-boundary match against the bank's own verified event log—never from a model's latent memory. 

An LLM (`gpt-4o`) is used solely downstream of that computation to transform already-cited structured facts into clear prose or answer client-scoped questions. It is structurally unable to introduce external figures or ungrounded claims.

---

## Quickstart

### 1. Installation

```bash
git clone https://github.com/delwinvanstrien/janus-wealth-intelligence.git
cd janus-wealth-intelligence
pip install -r requirements.txt
```

### 2. Run Automated Regression Tests

Verify exact mathematical reconciliation and zero-drift attribution across portfolio holdings:

```bash
pytest tests/
```

### 3. Run the CLI Smoke Test

Quickly inspect the deterministic engine output and client triage ranking directly in your terminal (no browser required):

```bash
python run.py            # default client (CL-0012) + top triage rankings
python run.py CL-0002    # inspect any specific client
```

### 4. Launch the Interactive RM Workbench

Start the full interactive three-column workbench on `http://localhost:8080`:

```bash
python app.py
```

### Optional: Enable Live LLM Features

Janus works completely offline without API keys (falling back to deterministic templating). To enable GPT-4o narration, the interactive "Ask Why" client chat, and live market press search:

```bash
export OPENAI_API_KEY=sk-...
```

### Alternative: Headless FastAPI Service

A decoupled REST API and static frontend are also available:

```bash
uvicorn api.main:app --reload
# Open frontend/index.html in your browser
```

---

## System Architecture

```
data/*.csv + rm_notes.json   (12 institutional source files, immutable)
        │
        ▼
   src/ingest.py             loads everything into an in-memory DataStore
        │
        ▼
┌─── src/engine/ ────────────────────────────────────────────────────────────┐
│  DETERMINISTIC ENGINES. No LLM. Returns findings, never prose.             │
│                                                                            │
│  attribution.py    exact trading + price + FX decomposition                │
│  mandate.py        allocation-band & single-position breaches              │
│  concentration.py  cross-portfolio look-through concentration              │
│  collateral.py     loan-to-value trajectory vs. margin call                │
│  liquidity.py      commitments & cash needs vs. what's sellable            │
│  ground.py         links moves to event_log.csv taxonomy                   │
└────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
   src/schema.py              the Insight contract (structured schema)
   src/build.py               assembles engine findings into Insights
   src/engine/prioritise.py   computes explainable urgency score across book
        │
        ▼
┌─── src/engine/narrate/ ────────────────────────────────────────────────────┐
│  template.py        deterministic prose from Insight. Zero external deps.  │
│  __init__.py        narrate(): template or GPT-4o with strict guardrails   │
│  chat.py            "Ask Why": tool-calling loop closed over client_id     │
│  market_context.py  QUARANTINED live web search for external press color   │
└────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
   api/main.py     FastAPI JSON layer  ──►  frontend/index.html
   app.py          NiceGUI workbench   ──►  data/decisions.json (persistent audit log)
```

---

## The Insight Contract

Every finding from every engine is normalized into an immutable `Insight` object ([src/schema.py](file:///Users/delwinvanstrien/Documents/GitHub/janus-wealth-intelligence/src/schema.py)) before reaching the UI or narration layer:

| Field | Purpose | Guarantee |
|---|---|---|
| `headline` | Concise one-line finding | Strict character limit, factual summary |
| `contributions[]` | Computed numeric drivers | Renders as inspectable interactive chips |
| `evidence` | Event refs, holding IDs, mandate rule IDs, RM notes | Traceable back to raw bank logs |
| `confidence` | `high` / `medium` / `low` | Based on data completeness and grounding |
| `caveats[]` | Limitations of the observation | Explicit transparency on unobserved factors |
| `suggested_action` | Actionable proposal for the RM | Proposal only; human RM retains final decision |
| `explained_pct` | Share of move tied to real events | Attribution only; remainder labeled "unexplained" |
| `narrative` | Human-readable explanation | Generated strictly from the fields above |

*If a fact cannot be traced to a structured field in this schema, it is structurally barred from being generated.*

---

## The Five Quantitative Engines

1. **Attribution Engine** ([src/engine/attribution.py](file:///Users/delwinvanstrien/Documents/GitHub/janus-wealth-intelligence/src/engine/attribution.py))
   Decomposes portfolio delta between dated snapshots into `trading_effect + price_effect + fx_effect`. Reconciles identically to total change with a mathematical assertion. Grounds moves to events in `event_log.csv` and transparently flags ungrounded delta as unexplained.
2. **Mandate Engine** ([src/engine/mandate.py](file:///Users/delwinvanstrien/Documents/GitHub/janus-wealth-intelligence/src/engine/mandate.py))
   Evaluates portfolios against strategic asset allocation bands and single-instrument exposure limits. Cross-references RM notes to differentiate authorized client waivers from passive drift.
3. **Concentration Engine** ([src/engine/concentration.py](file:///Users/delwinvanstrien/Documents/GitHub/janus-wealth-intelligence/src/engine/concentration.py))
   Performs look-through analysis across direct holdings, funds, and structured products across a client's entire book to expose hidden single-name or single-sector concentration risks.
4. **Collateral & Margin Engine** ([src/engine/collateral.py](file:///Users/delwinvanstrien/Documents/GitHub/janus-wealth-intelligence/src/engine/collateral.py))
   Tracks multi-period loan-to-value (LTV) paths for credit facilities against maintenance covenants and margin call triggers.
5. **Liquidity Engine** ([src/engine/liquidity.py](file:///Users/delwinvanstrien/Documents/GitHub/janus-wealth-intelligence/src/engine/liquidity.py))
   Stress-tests upcoming capital calls and planned cash withdrawals against asset liquidity tiers (T0 to T30+), flagging cash shortfall horizons.

---

## Guardrailed AI Architecture

Janus implements a three-tier architecture for language model integration:

1. **Narration Synthesis** (`narrate/__init__.py`):
   Transforms finished `Insight` objects into 2–4 concise sentences for RM client prep. Guaranteed fallback to deterministic templates upon missing API keys, network disruption, or format mismatch.
2. **"Ask Why" Client-Scoped Agent** (`narrate/chat.py`):
   An interactive conversational assistant. Each of its analytical tools (`get_portfolio_attribution`, `get_rm_notes`, `get_insights`, `check_mandate_bands`, `check_liquidity`) is **cryptographically bound to a single `client_id`** upon instantiation, preventing cross-client data leakage. Capped at 5 execution rounds.
3. **Quarantined Market Context** (`narrate/market_context.py`):
   Runs live web searches strictly restricted to eight reputable institutional financial press outlets (*Reuters, Bloomberg, The Wall Street Journal, Financial Times, CNBC, AP, MarketWatch, The Economist*). **Completely quarantined** from the core engine—its findings are explicitly watermarked and cannot alter calculated numbers or suggested actions.

---

## Primary User Interface: RM Workbench (`app.py`)

A responsive, three-column decision workspace built with NiceGUI:

- **Left Rail — Triage Queue**: All clients across the RM's book ranked by composite urgency score, with primary drivers and life stage indicators visible at a glance.
- **Center Canvas — Insight Intelligence**: Interactive cards detailing computed metrics, cited event log entries, and the RM's notes on file. Features **Accept / Modify / Reject** actions and an on-demand **Market Context Enquiry** drawer.
- **Right Rail — Decision & Audit Log**: Real-time record of all RM actions persisted to `data/decisions.json`. Functions as a compliant audit trail and a client meeting prep sheet.

---

## Repository Layout

```
janus-wealth-intelligence/
├── api/
│   └── main.py              # FastAPI REST endpoints (/clients, /triage, /insights)
├── config/
│   └── grounding.py         # Tag taxonomy mapping event logs to instruments
├── data/                    # Institutional CSV data + RM notes + decision audit trail
├── fixtures/                # Pre-generated Insight/triage JSON snapshots for demo clients
├── frontend/
│   └── index.html           # Standalone static client interface consuming FastAPI
├── scripts/
│   └── gen_fixtures.py      # Utility to re-sync static fixtures with engine output
├── src/
│   ├── build.py             # Assembles raw engine findings into Insight schemas
│   ├── ingest.py            # In-memory high-speed data store
│   ├── schema.py            # The Insight contract definition
│   └── engine/              # Five deterministic quantitative engines & triage prioritizer
│       └── narrate/         # Guardrailed LLM narrative generation & Ask Why chat
├── tests/
│   └── test_golden_cl0012.py# Pytest mathematical reconciliation regression test
├── app.py                   # Primary NiceGUI interactive workbench
├── run.py                   # Zero-dependency CLI smoke test
├── requirements.txt         # Project dependencies
└── README.md                # System documentation
```
