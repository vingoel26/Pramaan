# Pramaan (Claim-to-Clause) — Product Requirements Document

|  |  | 
| ----- | ----- | 
| **Event** | Forge Gen AI Buildathon (IIT / NIT / IIIT / BITS) | 
| **Theme** | Theme 1: Civic Pulse, Digital Governance & Public Information | 
| **Problem statements covered** | #1 Multilingual Policy, Rights & Scheme Explainer + #4 Real-Time Fact-Verification Engine | 
| **Team** | 2 people (Person A, Person B) | 
| **Build window** | 10 hours | 
| **Working name** | Pramaan (Hindi: "proof / evidence") | 
| **Data platform** | MongoDB Atlas for everything (documents, vectors, keyword search, hybrid search, logs) | 
| **Doc version** | 1.2 (Hardened Architecture + Original Work Split) | 

> **How to use this document with AI:** save it as `docs/PRD.md` in the repo. Every AI prompt in Section 16 says "read docs/PRD.md section X". Keep this file in your AI coding tool's context so schemas and contracts stay consistent between both people.

## 1. Summary

Pramaan takes a viral claim about a government policy, scheme, or rule (text or a screenshot), finds the **exact clause** in live official sources that supports or contradicts it, and explains the result in plain Hindi (text + audio).

**One-line pitch:** *"Don't just tell people a forward is fake. Show them the exact line of the government notification, in their language."*

**What makes it different from a chatbot or a normal fact-checker**

* Every verdict carries a **quoted clause + source link**, and the quote is verified in code (with deterministic span-snapping repair) to actually exist in the fetched document (no LLM-only claims).

* Verdicts are richer than true/false: `SUPPORTED`, `CONTRADICTED`, `OUTDATED`, `PARTIALLY_TRUE`, `UNVERIFIED`, `NOT_CHECKABLE`.

* If no official source is found, it says **"Unverified"** instead of guessing.

* It bridges fact-checking (problem 4) and plain-language policy explanation (problem 1) in one flow.

* An agent loop (LangGraph) retries with reformulated queries, uses live web-search discovery (`site:pib.gov.in OR site:myscheme.gov.in`) when the index is thin, and streams every step to the UI.

* One MongoDB Atlas cluster powers vector search, keyword search, concurrent RRF hybrid ranking, documents, and the ingestion log.

## 2. Problem and users

**Problem.** Citizens receive viral forwards about subsidies, rules, deadlines, and rights. Official documents exist but are long, legalistic, and mostly in English or formal Hindi. Fact-checks are slow, binary, and rarely point to the exact clause.

**Primary users**

* Everyday citizens receiving WhatsApp-style forwards (farmers, shopkeepers, students, gig workers).

* Small business owners checking rumoured compliance changes.

**Secondary users**

* Journalists and local volunteers who want a fast first-pass verification with citations.

## 3. Goals, non-goals, success metrics

### Goals

1. A **live, deployed** web app that verifies a claim end to end in front of judges.

2. A visible agent trace (plan, retrieve, live_fetch, rerank, judge, verify, retry, abstain).

3. Real data only: ingested from live official sources into Atlas, no mock datasets.

4. Hindi explanation with audio.

5. Graceful handling of missing sources, vague claims, API failures, and NIC/WAF blocks.

### Non-goals (for the 10-hour build)

* Deepfake / audio forensics (roadmap slide only).

* Telegram / WhatsApp bots.

* e-Gazette direct scraping (too fragile for 10 hours; PIB + MyScheme via RSS + Tavily/Serper first).

* User accounts, history, payments.

* More than one output language beyond Hindi (stretch only).

* Legal advice. The product explains official text; it is not a lawyer.

### Success metrics (for the demo)

| Metric | Target | 
| ----- | ----- | 
| End-to-end verification works live | 100% of demo runs | 
| Time to first trace event | under 2 s | 
| Total time for one claim | under 18 s (max 2 attempts) | 
| Verified-quote rate on test claims where an official source exists | 80%+ (with span-repair) | 
| False "verified" (quote not in source) | 0 (enforced by code) | 
| Abstain instead of guess when no source exists | 100% of unverifiable test claims | 

## 4. Hackathon compliance checklist (hard rules)

| Rule | How we comply | 
| ----- | ----- | 
| No dummy data (disqualifier) | App only reads from live PIB/official fetches and the Atlas collections built from them. Test fixtures live only in `/tests` and are never imported by the app. | 
| Core logic built during the window | Only accounts, API keys, and the empty Atlas cluster are created beforehand. Repo is created at start. All code is written in the window. | 
| Live demo, not pre-recorded | Demo recorded from a real live run; judging done on the deployed URL. | 
| Everyone can explain the code | Section 16.6: 15-minute swap session before judging. | 
| Real-time APIs / live DBs | PIB RSS + seeded search backfill + `live_fetch` via Tavily/Jina on demand; MongoDB Atlas as the live DB. | 
| Beyond a "frontend wrapper" | Custom ingestion, concurrent hybrid RRF retrieval, stateful LangGraph agent, deterministic verbatim span-verification. | 

If a government site blocks cloud data center IPs during judging, `live_fetch` uses Tavily (`include_raw_content=True`) / Jina Reader (`https://r.jina.ai/`) as a clean proxy fallback, and previously fetched documents in Atlas are served with "cached on `<date>`".

## 5. User scenarios

| ID | Scenario | Expected result | 
| ----- | ----- | ----- | 
| US-1 | User pastes "Govt has stopped the X subsidy from next month" | Agent finds the notification; verdict + quoted clause + source link + Hindi explanation | 
| US-2 | User pastes a claim that was true but later amended | Verdict `OUTDATED` with both clauses/dates when found | 
| US-3 | User pastes a claim with no official source | After 2 attempts (including `live_fetch`), verdict `UNVERIFIED` with an honest explanation | 
| US-4 | User pastes an opinion ("this scheme is useless") | `NOT_CHECKABLE`, with a prompt to ask a factual question | 
| US-5 | User uploads a WhatsApp screenshot | (Could) Vision LLM extracts the text, then normal flow | 
| US-6 | User watches the trace panel | Sees each agent step, retries, and failures live | 
| US-7 | Judge opens the ingestion panel | Sees newly ingested official documents with real timestamps | 

## 6. Functional requirements

Priority: **M** = must, **S** = should, **C** = could.

| ID | Requirement | Priority | 
| ----- | ----- | ----- | 
| FR-1 | Accept a text claim (up to \~1,500 chars), Hindi, English, or Hinglish | M | 
| FR-2 | Extract atomic checkable claims as structured JSON; mark opinions as not checkable | M | 
| FR-3 | Plan a search query + source filters per claim | M | 
| FR-4 | Hybrid retrieval in MongoDB Atlas: concurrent `asyncio.gather` of Atlas Vector Search (`$vectorSearch`) + Atlas Search (`$search` BM25), fused with app-side Reciprocal Rank Fusion (`rrf`) | M | 
| FR-5 | Cohere rerank of fused candidates, keep top 3 | M | 
| FR-6 | LLM judge returns label + exact quote + chunk id + reason | M | 
| FR-7 | Code-level verbatim check against in-memory `state["reranked"]` with deterministic prefix/suffix span repair (fixes LLM `...` ellipses without rerunning retrieval) | M | 
| FR-8 | Decide: verified, then explain; quote-only failure, repair or re-prompt judge once; retrieval thin and attempts < 2, retry with `live_fetch`/reformulated query; otherwise abstain | M | 
| FR-9 | Stream every node event to the UI (SSE) | M | 
| FR-10 | Evidence card: verdict badge, quote, source link, doc date, confidence, "limits of this check" | M | 
| FR-11 | Scheduled + seeded ingestion of PIB/scheme documents into Atlas with a timestamped log collection | M | 
| FR-12 | Ingestion panel in UI (latest documents + timestamps) | S | 
| FR-13 | Plain-language "what this means for you" explanation in English + Hindi (generated in a single structured LLM call with digit-preservation check) | M | 
| FR-14 | Hindi audio (Sarvam TTS) player; fallback to browser speech synthesis | S | 
| FR-15 | `live_fetch`: if retrieval is thin, query Tavily (`site:pib.gov.in OR site:myscheme.gov.in`), ingest top 3 URLs on demand, and inject chunks directly into `candidates` | M | 
| FR-16 | Screenshot input via vision LLM | C | 
| FR-17 | Second source (Google Fact Check API) | C | 
| FR-18 | Second language | C | 
| FR-19 | Run log collection (`runs`) with trace events for debugging and demo evidence | S | 

## 7. Non-functional requirements

* **Honesty over fluency:** no verdict without a verified quote; default to `UNVERIFIED`.

* **Robustness:** every external call has a timeout (10-15 s), 1-2 retries, and a user-visible fallback. App never shows a blank screen or a stack trace.

* **Degradation order:** vector search fails, keyword-only; keyword search fails, vector-only; both fail, trigger `live_fetch` or clean error; reranker down, use RRF fused order; direct PIB fetch blocked by WAF (403/timeout), fall back to Tavily `raw_content` or `https://r.jina.ai/<url>`; TTS down, text + browser `speechSynthesis`.

* **Rate-limit isolation:** Use **two separate Cohere API keys** (`COHERE_INGEST_KEY` for Person B's embeddings/retrieval, `COHERE_QUERY_KEY` for Person A's live rerank) so background ingestion never triggers `429` during a live demo.

* **Politeness:** max 1 request/sec to PIB, identify with a descriptive Browser-like User-Agent.

* **Security basics:** API keys and the Atlas connection string only in env vars; dedicated Atlas DB user with a strong password and least privilege; CORS limited to the frontend origin; input length limits; strip HTML from inputs; no secrets in the repo.

* **Observability:** structured logs per request with a `run_id`; trace events saved in the `runs` collection.

* **Free-tier discipline:** stay within Atlas M0 limits (512 MB storage, 3 search/vector indexes): cap the backfill at **1,200-1,500 high-value chunks**, use bulk writes, never return embeddings in queries.

## 8. Architecture and tech stack

```
Next.js UI --POST /verify--> FastAPI --invoke--> LangGraph agent
   ^                            |                    |
   |<------ SSE events ---------+                    +--> MongoDB Atlas
                                                    |      - documents, chunks (+ embeddings)
                                                    |      - Atlas Vector Search + Atlas Search
                                                    |      - Concurrent asyncio.gather + Python RRF
                                                    |      - ingestion_log, runs
                                                    +--> Cohere (INGEST_KEY: embed | QUERY_KEY: rerank)
                                                    +--> Tavily API / Jina Reader (live_fetch discovery)
                                                    +--> LLM API (extract, plan, judge, bilingual explain)
                                                    +--> Sarvam TTS (Hindi audio; fallback: Web Speech API)

Ingestion worker (Laptop seed + FastAPI APScheduler) --> PIB RSS + Seeded Queries -> parse -> chunk -> Cohere (INGEST_KEY) -> Atlas (bulk upsert)
Agent live_fetch --------------------------------------> Tavily search (site:pib.gov.in) -> ingest_url() (returns fresh chunks directly into state)
```

| Layer | Choice | Notes | 
| ----- | ----- | ----- | 
| Frontend | Next.js (App Router) + TypeScript + Tailwind | Deploy on Vercel (Person B) | 
| Backend | FastAPI (Python 3.11+) + Pydantic v2 | Deploy on Render or Railway (Person A) | 
| Orchestration | LangGraph `StateGraph` | Streamed with `astream(stream_mode="updates")` (Person A) | 
| LLM | One fast frontier model with structured JSON output (e.g., Gemini 2.5 Flash or GPT-4o) | Single `llm.py` wrapper; handles both EN + HI explanation in one call (Person A) | 
| Database (everything) | **MongoDB Atlas** (M0 free cluster) | Documents, chunks, vectors, keyword index, logs (Person B) | 
| DB driver | PyMongo (sync wrapped in `asyncio.to_thread` or Motor/PyMongo Async) | Keep consistent across `mongo.py` and `hybrid.py` (Person B) | 
| Vector search | Atlas Vector Search index `chunks_vector` on `chunks.embedding` | Cosine similarity, 1024 dims (`embed-multilingual-v3.0`) (Person B) | 
| Keyword search | Atlas Search index `chunks_text` on `chunks.text` and `chunks.title` | BM25-style text ranking (Person B) | 
| Hybrid fusion | Concurrent `asyncio.gather(vector_search, text_search)` + app-side RRF | Works on all MongoDB Atlas versions (7.0+ and 8.0+) with zero preview syntax bugs (Person B) | 
| Embeddings & Rerank | Cohere `embed-multilingual-v3.0` (B) + `rerank-multilingual-v3.0` (A) | **Two API keys**: `COHERE_INGEST_KEY` and `COHERE_QUERY_KEY` | 
| Live Discovery | **Tavily Search API** (`site:pib.gov.in`, `site:myscheme.gov.in`) | Bypasses broken PIB ASP.NET search and NIC data-center IP blocks | 
| Language & Audio | Bilingual LLM prompt (EN + HI text) + **Sarvam AI** (`bulbul:v1`) for TTS | Browser `speechSynthesis` (`hi-IN`) as automatic fallback (Person A) | 
| Fetch/parse | httpx, BeautifulSoup4, feedparser, PyMuPDF, Jina Reader fallback | (Person B) | 
| Scheduler | APScheduler (inside FastAPI process) | Polls RSS every 20 min (Person B) | 

## 9. Data model and ingestion spec (MongoDB Atlas)

### 9.1 Sources & Seed Strategy

| Source | Use | Priority | 
| ----- | ----- | ----- | 
| PIB RSS feeds | Live recent announcements (polled every 20 min) | M | 
| Seeded Flagship Schemes (via Tavily/PIB URLs) | 30-40 historical PIB/MyScheme pages on PM-KISAN, Ayushman Bharat, Ration/ONORC, UPI/RBI rules, LPG subsidies, EPFO | M | 
| On-demand `live_fetch` (Tavily `site:pib.gov.in`) | Real-time retrieval for any claim missing from the pre-indexed corpus | M | 

> **Why seeded URLs + Tavily matter:** PIB's RSS feed only holds today's \~25 releases. Running `scripts/backfill.py` locally from Person B's laptop with both the RSS feed AND 30 seeded flagship scheme searches ensures judges can test real viral forwards from the past 2 years and get instant hits, while avoiding `.gov.in` cloud IP blocks.

### 9.2 Collections

Database: `pramaan`

**`documents`**

```json
{ "_id": "<uuid5(url)>", "url": "...", "title": "...", "ministry": "...",
  "published_at": "2026-10-28", "fetched_at": "ISO", "content_hash": "sha256",
  "status": "ok" }
```

**`chunks`** (the search collection)

```json
{ "_id": "<uuid5(url + idx)>", "doc_id": "...", "idx": 3,
  "text": "exact chunk text, never modified",
  "title": "...", "ministry": "...", "url": "...", "published_at": "2026-10-28",
  "embedding": [ ...1024 floats... ], "ingested_at": "ISO" }
```

**`ingestion_log`**

```json
{ "url": "...", "title": "...", "chunks_added": 12, "status": "ok|skipped|error",
  "message": "", "created_at": "ISO" }
```

**`runs`**

```json
{ "run_id": "...", "claim": "...", "label": "SUPPORTED", "attempts": 1,
  "latency_ms": 11200, "events": [ ... ], "created_at": "ISO" }
```

### 9.3 Regular indexes

* `documents`: unique on `url`

* `chunks`: `doc_id`; unique compound `(doc_id, idx)`

* `ingestion_log`: `created_at` descending

* `runs`: `created_at` descending

### 9.4 Atlas Search indexes (free tier allows 3 total, we use 2)

**Vector index `chunks_vector`** on `chunks`:

```json
{
  "fields": [
    { "type": "vector", "path": "embedding", "numDimensions": 1024, "similarity": "cosine" },
    { "type": "filter", "path": "ministry" },
    { "type": "filter", "path": "doc_id" }
  ]
}
```

**Text index `chunks_text`** on `chunks`:

```json
{ "mappings": { "dynamic": false, "fields": {
    "text": { "type": "string" },
    "title": { "type": "string" } } } }
```

Create both with `scripts/setup_indexes.py` in Hour 1 and poll until `queryable`.

### 9.5 Ingestion pipeline & WAF fallback

1. **Fetch with resilience:** Attempt direct `httpx.get(url)` with a standard Chrome User-Agent (timeout 8s). If status is 403/5xx or timeout (common from Render/Railway IPs hitting `.gov.in`), immediately fall back to `https://r.jina.ai/{url}` or Tavily's `raw_content`.

2. **Skip** URLs already in `documents` with the same `content_hash`.

3. **Parse** HTML/text to clean body text; extract `title`, `ministry`, `published_at`, `url`.

4. **Chunk**: split by paragraph/section, target 200-400 tokens, \~30-token overlap; keep chunk text **unmodified**. Assign deterministic `id = str(uuid5(NAMESPACE_URL, f"{url}#{idx}"))` immediately in memory.

5. **Embed** in batches (up to 90 texts/call) using `COHERE_INGEST_KEY` (`input_type="search_document"`).

6. **Upsert** documents and chunks with `bulk_write` of `ReplaceOne(..., upsert=True)`.

7. **Return** the list of in-memory `Chunk` objects (without embeddings) from `ingest_url()` so `live_fetch` can use them immediately.

### 9.6 Storage & Rate-Limit Budget

* Cap total chunks at **1,500 chunks** (`MAX_CHUNKS=1500`). At \~10 KB/chunk, that uses \~15 MB (well inside the 512 MB M0 limit) and takes <18 Cohere batch API calls to embed.

* Never return `embedding` in query results (`{"$project": {"embedding": 0}}`).

### 9.7 Index lag bypass (critical)

Atlas Search and Vector Search indexes take several seconds to index newly written documents. Therefore:

1. `live_fetch` calls `ingest_url()`, which writes to Atlas in the background AND returns the freshly built `list[Chunk]` (with deterministic `id`s) directly into `state["candidates"]`.

2. `verbatim_check` **always looks up `chunk_id` inside `state["reranked"]` in memory**, never by querying MongoDB.

## 10. Agent specification (LangGraph)

### 10.1 Verdict labels

`SUPPORTED`, `CONTRADICTED`, `OUTDATED`, `PARTIALLY_TRUE`, `UNVERIFIED`, `NOT_CHECKABLE`

### 10.2 State

```python
from typing import TypedDict

class GraphState(TypedDict, total=False):
    run_id: str
    raw_input: str
    input_lang: str
    claims: list[dict]          # [{id, text, type, entities, checkable}]
    current_claim: dict
    plan: dict                  # {query, filters, source_hint}
    failed_queries: list[str]
    candidates: list[dict]      # Chunk dicts after RRF hybrid fusion (+ live_fetch chunks)
    reranked: list[dict]        # top 3 Chunk dicts after Cohere rerank (kept in memory!)
    top_rerank_score: float
    judgement: dict             # {label, quote, chunk_id, reason, confidence}
    quote_ok: bool
    attempts: int
    live_fetch_done: bool
    verdict: dict               # final Verdict object
    explanation_en: str
    explanation_hi: str
    audio_url: str | None
```

### 10.3 Nodes & Ownership

| Node | Type | Input to Output | Owner | 
| ----- | ----- | ----- | ----- | 
| `extract_claims` | LLM (JSON) | `raw_input` to `claims`, `current_claim` | A | 
| `plan_sources` | LLM (JSON) | claim (+ failed queries) to `plan` | A | 
| `retrieve` | Code (Atlas) | `plan` to `candidates` (Concurrent Vector + Text + Python RRF) | B | 
| `live_fetch` | Code (Tavily + B's ingest) | thin results: search Tavily (`site:pib.gov.in`), call `ingest_url()`, add chunks directly to `candidates` | A (calls B's `ingest_url`) | 
| `rerank` | Cohere (`QUERY_KEY`) | `candidates` to `reranked` (top 3) + `top_rerank_score` | A | 
| `judge` | LLM (JSON) | claim + `reranked` to `judgement` | A | 
| `verbatim_check` | Code (In-Memory + Repair) | checks `judgement.quote` vs `state["reranked"]`; auto-repairs ellipsis `...` spans | A | 
| `decide` | Conditional edge | routes to `explain_translate`, `plan_sources` (max 2 attempts), or `abstain` | A | 
| `explain_translate` | LLM (Bilingual JSON) + Sarvam TTS | `judgement` to `explanation_en`, `explanation_hi`, `audio_url` | A | 
| `abstain` | Code | builds `UNVERIFIED` or `NOT_CHECKABLE` verdict | A | 

### 10.4 Edges & Fast-Retry Logic

```
START -> extract_claims
extract_claims -> (not checkable) abstain | (checkable) plan_sources
plan_sources -> retrieve
retrieve -> (len(candidates) < 3 and not live_fetch_done) live_fetch -> rerank
retrieve -> (len(candidates) >= 3) rerank
rerank -> (top_rerank_score < 0.25 and not live_fetch_done) live_fetch -> rerank
rerank -> judge -> verbatim_check -> decide
decide -> explain_translate -> END      (quote_ok and label in {SUPPORTED, CONTRADICTED, OUTDATED, PARTIALLY_TRUE})
decide -> plan_sources                  (otherwise and attempts < 2)
decide -> abstain -> END                (otherwise and attempts >= 2)
```

*Latency optimization:* Capping `attempts` at 2 (initial pass + 1 reformulated query with `live_fetch`) plus single-call bilingual explanation keeps worst-case latency under 18 seconds.

### 10.5 Guardrails (enforced in code, not just prompts)

1. **In-memory verbatim check + span repair:** `verbatim_check` finds the cited chunk in `state["reranked"]`. If exact normalized substring fails (e.g., LLM inserted `...` or dropped punctuation), it checks if the first 3 words and last 3 words of the quote exist in the chunk within 450 chars and snaps `judgement["quote"]` to the exact raw slice from the chunk. If that still fails, `quote_ok = False`.

2. Default outcome is `UNVERIFIED`.

3. Explanations are generated strictly from the verified quote and metadata in one bilingual JSON call, verified by a digit-preservation check.

4. Always display "Limits of this check" and the source date.

## 11. API specification

### 11.1 Endpoints

| Method | Path | Purpose | 
| ----- | ----- | ----- | 
| POST | `/verify` | Start a run; returns an SSE stream (`text/event-stream`) | 
| GET | `/ingestion/recent?limit=20` | Latest `ingestion_log` entries | 
| GET | `/health` | Liveness + Atlas connectivity and index-ready status | 
| POST | `/admin/ingest` | (Protected by `X-Admin-Token`) trigger a poll/backfill | 

### 11.2 Request

```json
{ "text": "Govt has stopped the ... subsidy from next month", "lang": "hi" }
```

### 11.3 SSE events

```json
{ "type": "step", "run_id": "r_123", "node": "retrieve", "status": "start|done|error",
  "message": "Searching indexed documents", "data": { "hits": 14 }, "ts": "2026-11-01T10:00:00Z" }
```

```json
{ "type": "final", "run_id": "r_123", "verdict": { } }
```

```json
{ "type": "error", "run_id": "r_123", "message": "Retrieval is unavailable. Please try again." }
```

### 11.4 Shared types

```python
# backend/app/schemas.py
from pydantic import BaseModel
from typing import Literal, Optional

Label = Literal["SUPPORTED","CONTRADICTED","OUTDATED","PARTIALLY_TRUE","UNVERIFIED","NOT_CHECKABLE"]

class Chunk(BaseModel):
    id: str
    text: str
    url: str
    title: str
    ministry: Optional[str] = None
    published_at: Optional[str] = None
    score: float = 0.0

class Verdict(BaseModel):
    label: Label
    claim: str
    quote: Optional[str] = None
    source_url: Optional[str] = None
    source_title: Optional[str] = None
    source_date: Optional[str] = None
    confidence: float = 0.0
    reason: str = ""
    explanation_en: str = ""
    explanation_hi: str = ""
    audio_url: Optional[str] = None
    attempts: int = 0
    limits_note: str = "Based only on the official documents indexed; verify critical decisions with the issuing department."
```

```ts
// frontend/src/types.ts (mirror of the above)
export type Label = "SUPPORTED"|"CONTRADICTED"|"OUTDATED"|"PARTIALLY_TRUE"|"UNVERIFIED"|"NOT_CHECKABLE";
export interface Verdict { label: Label; claim: string; quote?: string; source_url?: string;
  source_title?: string; source_date?: string; confidence: number; reason: string;
  explanation_en: string; explanation_hi: string; audio_url?: string; attempts: number; limits_note: string; }
export interface StepEvent { type: "step"; run_id: string; node: string;
  status: "start"|"done"|"error"; message: string; data?: Record<string, unknown>; ts: string; }
```

## 12. Frontend specification (Next.js)

**Single page** with two columns (stacked on mobile).

| Component | Behaviour | 
| ----- | ----- | 
| `ClaimInput` | Textarea + "Verify" button + example-claim chips built from recently ingested PIB titles (**not hardcoded**) | 
| `TracePanel` | Live list of step events with icons for start/done/error/retry; node name, message, elapsed time | 
| `EvidenceCard` | Verdict badge (colour per label), claim, quoted clause (blockquote), source link, doc date, confidence, reason, "Limits of this check" | 
| `ExplanationBlock` | English + Hindi toggle | 
| `AudioPlayer` | Plays `audio_url`; falls back to `window.speechSynthesis` (`hi-IN`) | 
| `IngestionPanel` | Polls `/ingestion/recent` every 30 s; shows title, time, chunks added | 
| `ErrorBanner` | Friendly messages for timeouts / API errors | 

## 13. Repository layout

```
pramaan/
├── docs/PRD.md
├── README.md                      # setup, architecture diagram, demo steps        [B]
├── backend/
│   ├── Dockerfile                                                                  [A]
│   ├── requirements.txt
│   ├── .env.example
│   ├── app/
│   │   ├── main.py                # FastAPI app, CORS, routes, scheduler startup   [A]
│   │   ├── schemas.py             # shared Pydantic types                          [A]
│   │   ├── events.py              # SSE helpers                                    [A]
│   │   ├── llm.py                 # LLM wrapper (JSON mode, retries, timeouts)     [A]
│   │   ├── prompts.py             # all prompts (including bilingual explain)      [A]
│   │   ├── graph/
│   │   │   ├── state.py                                                            [A]
│   │   │   ├── nodes.py                                                            [A]
│   │   │   ├── verify.py          # verbatim check + deterministic span repair     [A]
│   │   │   └── build.py           # graph wiring                                   [A]
│   │   ├── rerank.py              # Cohere rerank wrapper (COHERE_QUERY_KEY)       [A]
│   │   ├── language.py            # Sarvam TTS + digit check                       [A]
│   │   ├── runs.py                # save run logs to Atlas                         [A]
│   │   ├── ingest/
│   │   │   ├── sources.py         # PIB RSS + Tavily discovery + Jina fallback     [B]
│   │   │   ├── parse.py           # BS4 + PyMuPDF parser                           [B]
│   │   │   ├── chunk.py           # paragraph chunker (exact substrings)           [B]
│   │   │   ├── embed.py           # Cohere embed wrapper (COHERE_INGEST_KEY)       [B]
│   │   │   ├── pipeline.py        # ingest_url(), poll_feeds(), backfill()         [B]
│   │   │   ├── routes.py          # /ingestion/recent, /admin/ingest               [B]
│   │   │   └── scheduler.py       # APScheduler periodic ingestion                 [B]
│   │   └── retrieval/
│   │       ├── mongo.py           # client, collections, regular indexes, health   [B]
│   │       ├── indexes.py         # SearchIndexModel defs + wait-until-queryable   [B]
│   │       └── hybrid.py          # retrieve(): asyncio.gather(vec, text) + RRF    [B]
│   ├── scripts/
│   │   ├── setup_indexes.py                                                        [B]
│   │   └── backfill.py            # RSS + 30 seeded scheme queries (run locally!)  [B]
│   └── tests/
├── frontend/                                                                       [B]
│   ├── src/app/page.tsx
│   ├── src/components/*.tsx
│   ├── src/lib/stream.ts          # POST + SSE reader
│   └── src/types.ts
```

**Rule:** each person edits only their own directories. Shared files (`schemas.py`, `types.ts`) change only after a quick message to the other person.

## 14. Work split

### 14.1 Ownership

#### Person A: Verification Engine (Agent + Backend Core)
In simple terms, Person A builds the **verification engine** and is responsible for:
* **FastAPI setup** (`main.py`, CORS, `/health`, `/verify`).
* **SSE streaming** (`events.py`, streaming step and final events).
* **LangGraph workflow** (`graph/state.py`, `graph/nodes.py`, `graph/build.py`).
* **Claim extraction and search planning** (`extract_claims`, `plan_sources`, `llm.py`, `prompts.py`).
* **LLM judgement** (`judge` node).
* **Verbatim verification** (`graph/verify.py` with deterministic span-repair).
* **Retry and abstention logic** (`decide` conditional routing, `abstain` node).
* **Cohere reranking** (`rerank.py` using `COHERE_QUERY_KEY`).
* **Hindi explanation and TTS** (`explain_translate` bilingual LLM generation + `language.py` Sarvam TTS).
* **Live fetching** (`live_fetch` node using Tavily search + calling Person B's `ingest_url`).
* **Run logging** (`runs.py` saving trace logs to MongoDB).
* **Backend deployment** (Dockerfile + Render/Railway deployment).

#### Person B: Data, Retrieval, and Frontend
Person B is responsible for:
* **MongoDB Atlas setup** (`retrieval/mongo.py`, collections, regular indexes, health check).
* **PIB ingestion** (`ingest/sources.py` with PIB RSS + Tavily seed discovery + Jina Reader fallback).
* **Document parsing** (`ingest/parse.py` using BeautifulSoup4 + PyMuPDF).
* **Chunking** (`ingest/chunk.py` exact unmodified paragraph substrings).
* **Embedding generation** (`ingest/embed.py` using `COHERE_INGEST_KEY` + `ingest/pipeline.py` + `scripts/backfill.py`).
* **Search-index creation** (`retrieval/indexes.py` + `scripts/setup_indexes.py` for Vector and BM25 Search indexes).
* **Hybrid retrieval** (`retrieval/hybrid.py` running concurrent Vector + BM25 search merged with Python RRF).
* **Periodic ingestion** (`ingest/scheduler.py` with APScheduler + `ingest/routes.py` for `/ingestion/recent` and `/admin/ingest`).
* **Next.js frontend** (`frontend/` components, `stream.ts` SSE reader, state management).
* **Frontend deployment** (Vercel deployment + connecting to backend).
* **README and pitch deck** (`README.md`, Mermaid architecture diagram, 5-7 slide deck).

|  | **Person A: Verification Engine** | **Person B: Data, Retrieval & Frontend** | 
| ----- | ----- | ----- | 
| Core | FastAPI setup, SSE streaming, LangGraph workflow, claim extraction, search planning, LLM judgement, verbatim verification, retry & abstention logic, run logging | MongoDB Atlas setup, PIB ingestion, document parsing, chunking, embedding generation, search-index creation, hybrid retrieval, periodic ingestion | 
| Integrations | LLM, Cohere reranking (`COHERE_QUERY_KEY`), Hindi explanation & Sarvam TTS, live fetching (Tavily + calling `ingest_url`) | Cohere embeddings (`COHERE_INGEST_KEY`), APScheduler | 
| UI | none (provides `docs/sample_sse.txt` at Hour 2) | Entire Next.js frontend | 
| Deploy | Backend deployment (Render/Railway) | Frontend deployment (Vercel), Atlas cluster & network access | 
| Docs | Test runs + demo video recording | README, architecture diagram, pitch deck | 

### 14.2 Interface contract (agree in the first 30 minutes)

Person B delivers (in `backend/app/`):

```python
# retrieval/hybrid.py
async def retrieve(query: str, filters: dict | None = None, k: int = 12) -> list[Chunk]: ...

# ingest/pipeline.py
async def ingest_url(url: str, fallback_text: str | None = None, fallback_title: str | None = None) -> dict:
    # returns {"url":..., "status": "ok|skipped|error", "message": str,
    #          "chunks_added": int, "chunks": list[Chunk]}
    # CRITICAL: Each Chunk in "chunks" MUST have its deterministic id = uuid5(url#idx) populated!
async def poll_feeds() -> list[dict]: ...

# ingest/routes.py & ingest/scheduler.py
# exposes router (GET /ingestion/recent, POST /admin/ingest) and start_scheduler() for main.py
```

Person A calls these. Until B's real functions exist, A develops against a **dev-only stub** in `tests/` (never imported by the app) and switches to the real functions at Gate 2 (Hour 4).

Person A delivers (for B's frontend): the `/verify` SSE stream and `/ingestion/recent` shapes in Section 11, plus a sample SSE transcript copied from a real run in `docs/sample_sse.txt` at Hour 2.

## 15. Timeline and gates (10 hours)

| Hour | Person A (Verification Engine) | Person B (Data, Retrieval & Frontend) | Joint | 
| ----- | ----- | ----- | ----- | 
| 0-0.5 |  |  | Repo, `.env` (with 2 Cohere keys + Tavily key), Atlas cluster, schemas, split confirmed | 
| 0.5-2 | FastAPI skeleton + SSE; LangGraph skeleton with stubs; `extract_claims` + `plan_sources`; export `docs/sample_sse.txt` | `mongo.py` + `setup_indexes.py` (**create both search indexes in Hour 1**); `sources.py`, `parse.py`, `chunk.py` | **Gate 1 (h2):** graph runs end to end with stubs and outputs `sample_sse.txt` | 
| 2-4 | `judge`, `verbatim_check` (with span repair), `decide`, `rerank.py` | `embed.py`, `pipeline.py`, `hybrid.py` (concurrent Vector + BM25 + RRF), `scheduler.py`, `routes.py`, run `scripts/backfill.py` locally | **Gate 2 (h4):** real chunks come back from `retrieve()` against Atlas | 
| 4-5.5 | `explain_translate` (bilingual LLM + Sarvam TTS), `live_fetch` (Tavily + `ingest_url`), wire B's real `retrieve()` and `routes.py` | Next.js app: scaffold, `stream.ts`, `ClaimInput`, `TracePanel`, `EvidenceCard`, `ExplanationBlock` |  | 
| 5.5-6.5 | Deploy backend (Render/Railway); env + CORS | Finish `IngestionPanel` + `AudioPlayer`; deploy frontend (Vercel); connect to backend | **Gate 3 (h6.5): deployed app verifies a claim end to end** | 
| 6.5-8 | `runs.py` logging, error boundaries, test 10+ real viral claims, tune prompts | Mobile UI polish, error states, start README & architecture diagram | **Feature freeze at h8** | 
| 8-9.5 | Bugfixes from test runs; record backup live run | Pitch deck (5-7 slides) + finish README |  | 
| 9.5-10 | Record final 2-3 min live demo video | Fill submission form | **15-min swap-explanation session** | 

**If a gate is missed, cut in this order:** Sarvam TTS audio (use browser `speechSynthesis` only), `IngestionPanel` UI component (show Atlas UI or `/ingestion/recent` JSON), Cohere rerank (fallback to RRF order). **Never cut:** verbatim check, abstain path, `live_fetch`, the hour-6.5 deployment.

## 16. Step-by-step implementation plan (with AI prompts)

### 16.0 Master context prompt (paste at the start of every AI session)

```
You are helping build "Pramaan", a hackathon project. Read docs/PRD.md fully.
Constraints: Python 3.11, FastAPI, Pydantic v2, LangGraph, MongoDB Atlas (PyMongo) for ALL data
(documents, chunks with embeddings, Atlas Vector Search, Atlas Search, ingestion log, run logs),
Cohere (embed-multilingual-v3.0 + rerank-multilingual-v3.0 with separate COHERE_INGEST_KEY and COHERE_QUERY_KEY),
Tavily API for live_fetch discovery, Next.js App Router + TypeScript + Tailwind.
Rules: no mock or hardcoded data in app code (fixtures only in /tests);
every external call needs a timeout, retry, and a graceful fallback;
never return the embedding field from MongoDB queries; keep functions small and typed;
follow the repo layout and ownership in Section 13 and 14; do not change shared types in
schemas.py / types.ts without telling me. I am Person [A|B]. Only touch files I own.
```

### 16.1 Phase 0: Shared setup (both, 30 min)

**Env vars (`.env.example`)**

```
LLM_API_KEY=
LLM_MODEL=
COHERE_INGEST_KEY=
COHERE_QUERY_KEY=
COHERE_EMBED_MODEL=embed-multilingual-v3.0
COHERE_RERANK_MODEL=rerank-multilingual-v3.0
TAVILY_API_KEY=
MONGODB_URI=
MONGODB_DB=pramaan
SARVAM_API_KEY=
FRONTEND_ORIGIN=http://localhost:3000
ADMIN_TOKEN=
MAX_CHUNKS=1500
```

---

### 16.2 Person A: Verification Engine (Agent + Backend Core)

#### A1. FastAPI setup + SSE streaming (40 min) \[M\]

**AI prompt:**

```
Read PRD sections 7, 11. Create backend/app/main.py, events.py, using schemas.py.
Implement GET /health and POST /verify (VerifyRequest {text, lang}) which returns an SSE stream.
Stream three placeholder step events and a final event using the Verdict schema, with 0.3s delays.
Add CORS using FRONTEND_ORIGIN. Add a pytest that parses the SSE output.
```

#### A2. LLM wrapper (25 min) \[M\]

**AI prompt:**

```
Create backend/app/llm.py: async def complete_json(system: str, user: str, model: type[BaseModel]) -> BaseModel.
Use my LLM provider's async SDK with JSON/structured output and temperature=0. 15s timeout, 2 retries with backoff,
strip code fences, validate with the Pydantic model, raise LLMError on failure. Add tests with a fake client.
```

#### A3. LangGraph workflow skeleton with stubs (45 min) \[M\] **(Gate 1)**

**AI prompt:**

```
Read PRD section 10. Implement graph/state.py (GraphState), graph/nodes.py (stubs for extract_claims,
plan_sources, retrieve, live_fetch, rerank, judge, verbatim_check, decide, explain_translate, abstain),
graph/build.py (StateGraph wiring per 10.4).
In main.py, stream node updates as SSE step events and save a real sample output to docs/sample_sse.txt for Person B.
Add a test that runs the graph and asserts it reaches END in both the verified path and the abstain path.
```

#### A4. Claim extraction and search planning (`extract_claims` + `plan_sources`) (35 min) \[M\]

**Prompts (`prompts.py`)**

```
EXTRACT_SYSTEM:
You extract checkable factual claims about Indian government policy, schemes, rules, deadlines or rights.
Input may be Hindi, English or Hinglish. Return JSON:
{"claims":[{"id":"c1","text":"<claim in user's wording>","type":"new_rule|cancellation|eligibility|deadline|amount|other",
"entities":{"scheme":"","ministry":"","amount":"","date":"","state":""},"checkable":true|false}]}
Opinions, predictions and vague statements must have checkable=false. Do not invent details.

PLAN_SYSTEM:
You write a search plan to verify one claim against official Indian government documents (PIB, MyScheme, ministries).
Return JSON: {"query":"<concise English search query with official scheme name, ministry, and key terms>",
"filters":{"ministry":"<optional>"}, "source_hint":"pib"}.
If previous queries failed, produce a broader query using official English scheme names and synonyms.
```

**AI prompt:**

```
Implement extract_claims and plan_sources nodes using llm.complete_json with the prompts in PRD 16.2 A4.
Pick the first checkable claim as current_claim; if none, route to abstain with NOT_CHECKABLE.
plan_sources receives failed_queries from state to avoid repeating. Add unit tests with a fake LLM.
```

#### A5. LLM judgement, Verbatim verification (with Span Repair) & Retry/Abstention logic (60 min) \[M\]

**Judge prompt**

```
JUDGE_SYSTEM:
You are a strict fact-verification judge. Given a CLAIM and up to 3 CHUNKS of official text, decide whether
any chunk directly supports or contradicts the claim.
Rules:
- The quote MUST be copied character-for-character as a continuous substring from ONE chunk. Do NOT use "..." ellipses inside the quote.
- Choose label: SUPPORTED, CONTRADICTED, OUTDATED (was true but a later chunk amends it), PARTIALLY_TRUE, or SILENT (chunks do not address the claim).
- If unsure, use SILENT.
Return JSON: {"label":"...","chunk_id":"...","quote":"...","reason":"<one sentence>","confidence":0.0-1.0}
```

**Verbatim check with deterministic span repair (`graph/verify.py`)**

```python
import re, unicodedata

def _norm(s: str) -> str:
    s = unicodedata.normalize("NFKC", s)
    s = s.replace("\u2018", "'").replace("\u2019", "'").replace("\u201c", '"').replace("\u201d", '"')
    return re.sub(r"\s+", " ", s).strip().lower()

def verify_and_repair_quote(quote: str, chunk_text: str, min_len: int = 20) -> tuple[bool, str]:
    """Returns (is_valid, verified_raw_quote). Looks up in memory, repairs '...' if prefix+suffix exist in chunk."""
    if not quote or not chunk_text:
        return False, ""
    norm_q = _norm(quote)
    norm_c = _norm(chunk_text)
    if len(norm_q) >= min_len and norm_q in norm_c:
        return True, quote.strip()
    # Deterministic span repair if LLM used '...' or altered minor mid-sentence punctuation
    words = [w for w in re.split(r"\W+", norm_q) if len(w) > 2]
    if len(words) >= 6:
        prefix = r"\b" + r"\W+".join(re.escape(w) for w in words[:3])
        suffix = r"\W+".join(re.escape(w) for w in words[-3:]) + r"\b"
        pattern = re.compile(f"({prefix}.{{1,450}}?{suffix})", re.IGNORECASE | re.DOTALL)
        m = pattern.search(chunk_text)
        if m and len(_norm(m.group(1))) >= min_len:
            return True, m.group(1).strip()
    return False, ""
```

**Decide**

```python
def route_after_check(state) -> str:
    j = state.get("judgement") or {}
    good = state.get("quote_ok") and j.get("label") in {"SUPPORTED","CONTRADICTED","OUTDATED","PARTIALLY_TRUE"}
    if good:
        return "explain_translate"
    return "plan_sources" if state.get("attempts", 0) < 2 else "abstain"
```

**AI prompt:**

```
Implement judge, verbatim_check and the decide router per PRD 10 and 16.2 A5.
CRITICAL: verbatim_check MUST look up chunk_id inside state["reranked"] (or state["candidates"]) in memory, NEVER by querying MongoDB.
Use verify_and_repair_quote() so if a quote is repaired, state["judgement"]["quote"] is updated to the exact substring from chunk.text.
Write unit tests for verify_and_repair_quote (exact match, smart quotes, '...' ellipsis repair, fake quote rejection) and route_after_check (max 2 attempts).
```

#### A6. Cohere reranking (`rerank.py`) (30 min) \[M\]

**AI prompt:**

```
Create backend/app/rerank.py using the Cohere SDK rerank endpoint (COHERE_QUERY_KEY and COHERE_RERANK_MODEL env vars).
Input: query, list[Chunk]. Output: (top 3 list[Chunk] with updated relevance score, top_score float).
10s timeout, one retry, and on failure return the first 3 inputs unchanged with top_score=0.5. Add tests with a fake client.
```

#### A7. Hindi explanation and TTS (`explain_translate` + `language.py`) (45 min) \[M explain, S TTS\]

**Explain prompt (Single LLM call for both English & Hindi)**

```
EXPLAIN_SYSTEM:
Explain in simple, friendly language (Class 8 reading level) what the verified official text means for an ordinary citizen.
Use ONLY the verified QUOTE and metadata provided. Do not add facts, numbers, or dates not in the quote.
Keep all numbers/digits identical between English and Hindi.
Return JSON:
{
  "explanation_en": "<3-4 sentences in plain English: verdict, what the clause says, what it means for the citizen>",
  "explanation_hi": "<3-4 sentences in natural, conversational Hindi (Devanagari) with exact same numbers/dates>"
}
```

**AI prompt:**

```
Implement backend/app/language.py and the explain_translate node per PRD 16.2 A7.
1. Call llm.complete_json with EXPLAIN_SYSTEM to get both explanation_en and explanation_hi in a single LLM call.
2. Run a digit-preservation check (verify Arabic digits in explanation_en also appear in explanation_hi; if missing, append the English numbers in parentheses).
3. Call Sarvam AI TTS API (https://api.sarvam.ai/text-to-speech, model "bulbul:v1", target_language_code "hi-IN") with an 8s timeout to generate a data:audio/wav;base64,... URL for audio_url. On any TTS error, set audio_url=None so the frontend uses browser speechSynthesis.
```

#### A8. Live fetching (`live_fetch` via Tavily) (45 min) \[M\]

**How:** When `retrieve` returns `< 3` chunks or `top_rerank_score < 0.25` (and `not state.get("live_fetch_done")`), query Tavily API (`search_depth="basic"`, `include_domains=["pib.gov.in", "myscheme.gov.in", "indiacode.nic.in"]`, `max_results=3`), call B's `ingest_url(url)` concurrently on the returned URLs, and append the returned `Chunk` objects directly into `state["candidates"]`.
**AI prompt:**

```
Implement the live_fetch node in graph/nodes.py.
Query Tavily API (https://api.tavily.com/search) with TAVILY_API_KEY, query=state["plan"]["query"],
include_domains=["pib.gov.in", "myscheme.gov.in"], max_results=3, include_raw_content=True.
For each result, call ingest.pipeline.ingest_url(url, fallback_text=result.get("raw_content"), fallback_title=result.get("title")).
Append all returned Chunk objects directly into state["candidates"], set live_fetch_done=True, and route to rerank.
```

#### A9. Run logging, error handling & Backend deployment (45 min) \[M\]

**AI prompt:**

```
Add node-level error handling and a 45s run timeout to the graph runner. Errors should emit a step event with
status "error" and return an UNVERIFIED verdict. Add runs.py to save run logs to the MongoDB "runs" collection.
Include Person B's ingest router and start_scheduler() in main.py startup.
Write a Dockerfile for the backend (Python 3.11 slim, PORT env) and deploy configuration for Render/Railway.
```

---

### 16.3 Person B: Data, Retrieval, and Frontend

#### B1. MongoDB Atlas setup + Search-index creation (45 min) \[M\]

**Steps:** Run `scripts/setup_indexes.py` in Hour 1 so `chunks_vector` (1024 dims) and `chunks_text` build while you write the parser.
**AI prompt:**

```
Read PRD section 9. Create backend/app/retrieval/mongo.py (PyMongo client, collections documents/chunks/
ingestion_log/runs, regular indexes from 9.3, health()) and retrieval/indexes.py plus scripts/setup_indexes.py
which create the Atlas Vector Search index "chunks_vector" (1024 dimensions, cosine) and Atlas Search index "chunks_text"
exactly as in 9.4 using SearchIndexModel and poll until both are queryable.
```

#### B2. PIB ingestion, Document parsing & Chunking (60 min) \[M\]

**How:** `.gov.in` sites often block cloud IPs. In `sources.py`, try direct `httpx.get(url)` first (with Chrome User-Agent, 8s timeout); if it fails or returns 403/empty body, automatically fetch `https://r.jina.ai/{url}` (or use `fallback_text` passed from Tavily in `live_fetch`).
**AI prompt:**

```
Implement ingest/sources.py, ingest/parse.py, and ingest/chunk.py per PRD 9.5.
1. sources.py: fetch_page(url) tries direct httpx GET with a browser User-Agent (8s timeout); if status != 200 or body < 200 chars, falls back to fetching https://r.jina.ai/{url}. Also implement fetch_pib_rss() polling PIB RSS feeds.
2. parse.py: extract title, ministry, published_at, and clean body text from HTML (BeautifulSoup) or markdown/text (from Jina/Tavily fallback).
3. chunk.py: split text into paragraph-aware chunks of ~250-350 words with ~30-word overlap. Chunk text MUST be an exact unmodified substring of the parsed body text.
Add unit tests for chunk.py and parse.py.
```

#### B3. Embedding generation + Seeded Laptop Backfill (60 min) \[M\]

**How:** Use `COHERE_INGEST_KEY` in `embed.py`. In `scripts/backfill.py`, ingest both the live PIB RSS feed AND run 25-30 seeded Tavily searches (`site:pib.gov.in` for PM-KISAN, Ayushman Bharat, Ration Card, Free LPG, EPFO, RBI UPI rules, MSP, etc.) **from your local laptop** to populate \~1,000-1,200 high-value chunks in Atlas.
**AI prompt:**

```
Implement ingest/embed.py (using COHERE_INGEST_KEY, model COHERE_EMBED_MODEL, batch size 90, 429 exponential backoff)
and ingest/pipeline.py with:
- async def ingest_url(url: str, fallback_text: str | None = None, fallback_title: str | None = None) -> dict
  Must assign deterministic uuid5 _id to every Chunk (uuid5(NAMESPACE_URL, f"{url}#{idx}")), bulk_write to MongoDB, log to ingestion_log, and return {"url": url, "status": "ok", "chunks_added": len(chunks), "chunks": chunks} (without embeddings).
- async def poll_feeds() -> list[dict]
- async def backfill(limit_chunks: int = 1200)
Also create scripts/backfill.py which polls PIB RSS plus uses Tavily (site:pib.gov.in) across 20 major Indian government schemes (PM-KISAN, Ayushman Bharat, PM Awas, Ujjwala, Ration card, UPI, etc.) to seed Atlas with ~1,000 real chunks.
```

#### B4. Hybrid retrieval (`retrieval/hybrid.py`) + Periodic ingestion (`scheduler.py`, `routes.py`) (60 min) \[M\] **(Gate 2)**

**How:** Run `$vectorSearch` and `$search` concurrently using `asyncio.gather` (via `asyncio.to_thread` on PyMongo), then merge with `rrf()`. Always project `{"embedding": 0}`.
```python
def rrf(rank_lists: list[list[str]], k: int = 60) -> list[tuple[str, float]]:
    scores: dict[str, float] = {}
    for lst in rank_lists:
        for rank, cid in enumerate(lst):
            scores[cid] = scores.get(cid, 0.0) + 1.0 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```
**AI prompt:**

```
Implement backend/app/retrieval/hybrid.py, ingest/scheduler.py, and ingest/routes.py.
In hybrid.py: async def retrieve(query: str, filters: dict | None = None, k: int = 12) -> list[Chunk].
1. Embed query with Cohere (input_type="search_query" using embed_texts).
2. Run two MongoDB aggregations concurrently using asyncio.gather:
   - Vector search: [{"$vectorSearch": {"index": "chunks_vector", "path": "embedding", "queryVector": qvec, "numCandidates": 100, "limit": 20}}, {"$project": {"embedding": 0}}]
   - Text search: [{"$search": {"index": "chunks_text", "text": {"query": query, "path": ["text", "title"]}}}, {"$limit": 20}, {"$project": {"embedding": 0}}]
3. If one search fails, log a warning and use the other. Merge results using rrf() and return top k Chunk objects.
In scheduler.py and routes.py: implement APScheduler periodic polling of poll_feeds() every 20 min, GET /ingestion/recent, and POST /admin/ingest.
```

#### B5-B7. Next.js frontend & Frontend deployment (3 hours total) \[M\]

**B5. Scaffold + Streaming Client (`stream.ts`) (45 min)**

```
AI prompt:
Read PRD sections 11 and 12. Create frontend/src/types.ts (exact mirror of schemas.py),
src/lib/stream.ts (async generator over SSE from POST /verify using fetch + ReadableStream reader with proper line buffering),
and a clean layout in src/app/page.tsx. Test against docs/sample_sse.txt.
```

**B6. Core UI Components (90 min)**

```
AI prompt:
Build the components in PRD section 12 with Tailwind CSS:
- ClaimInput (textarea, Verify button, dynamic example chips fetched from /ingestion/recent titles)
- TracePanel (live animated timeline of step events showing node, status badge, message, and elapsed ms)
- EvidenceCard (verdict badge with distinct color+icon for SUPPORTED/CONTRADICTED/OUTDATED/PARTIALLY_TRUE/UNVERIFIED/NOT_CHECKABLE, verbatim quote in a highlighted blockquote, source link + date, confidence meter, limits note)
- ExplanationBlock (English / Hindi toggle with Noto Sans Devanagari font)
- AudioPlayer (plays audio_url if present; otherwise uses window.speechSynthesis with lang="hi-IN")
- IngestionPanel (polls /ingestion/recent every 30s showing live indexed PIB docs)
```

**B7. Frontend deployment + README and pitch deck (90 min, Hours 5.5-9.5)**

* Deploy Next.js on Vercel at Hour 5.5 (`Gate 3`), pointing `NEXT_PUBLIC_API_URL` to Person A's Render/Railway URL.

* Hours 8-9.5: Write `README.md` with Mermaid architecture diagram and build the 6-slide pitch deck.

## 17. Demo script and submission checklist

### Demo (2-3 minutes, live)

1. **(20 s)** Problem: show a real WhatsApp-style viral forward about a government scheme and explain why binary fact-checkers fail.

2. **(60 s)** Live Run 1 (Verified): click/paste a real claim, watch the `TracePanel` stream each node (`extract_claims -> plan_sources -> retrieve -> rerank -> judge -> verbatim_check -> explain_translate`), show the exact highlighted government clause, source link, and play the Hindi audio explanation.

3. **(35 s)** Live Run 2 (Honest Abstain): paste a fabricated rumour, watch the agent trigger `live_fetch` and reformulate its query, and cleanly abstain with **`UNVERIFIED`** instead of hallucinating.

4. **(20 s)** Show the live `IngestionPanel` with real PIB documents and timestamps stored in MongoDB Atlas.

5. **(20 s)** Architecture slide: single MongoDB Atlas hybrid engine, deterministic span-repair verbatim check, and live discovery.

### Submission checklist

* \[ \] **Problem statement & Solution description** completed

* \[ \] **Prototype link:** live deployed Vercel URL connected to live Render/Railway backend

* \[ \] **Demo video:** 2-3 min live end-to-end run

* \[ \] **GitHub repo:** public, clean `README.md` with Mermaid diagram, zero secrets committed

* \[ \] **Pitch deck:** 5-7 slides (PDF/link)

* \[ \] **15-minute swap session completed:** Person A can explain ingestion/Atlas indexes/UI; Person B can explain LangGraph nodes/RRF/verbatim span-repair.

## 18. Risks and mitigations

| Risk | Likelihood | Mitigation | 
| ----- | ----- | ----- | 
| PIB structure/WAF blocks cloud IPs | High | Run initial backfill from laptop; use Jina Reader (`r.jina.ai`) and Tavily `raw_content` as automatic fallback | 
| Cohere trial rate limits during backfill | High | Separate `COHERE_INGEST_KEY` and `COHERE_QUERY_KEY`; cap backfill at 1,500 chunks | 
| LLM judge outputs `...` inside quote | High | Deterministic span-repair in `verify_and_repair_quote` snaps prefix+suffix to raw chunk text | 
| Atlas M0 limits (512 MB storage, 3 indexes) | Medium | Use only 2 indexes; cap chunks at 1,500 (\~15 MB); never return embeddings | 
| Search index lag after writes | High | `live_fetch` injects fresh chunks directly into `state["candidates"]`; `verbatim_check` checks in-memory `state["reranked"]` | 
| `$rankFusion` preview errors on M0 | High | Replaced with concurrent `asyncio.gather($vectorSearch, $search)` + Python `rrf()` | 
| Hindi translation alters digits or adds latency | Medium | Single bilingual LLM JSON output + regex digit-preservation check | 
| Free-tier backend sleeps/cold start | Medium | Ping `/health` 5 minutes before judging and recording | 

## 19. How this maps to the judging rubric

| Criterion (points) | Where we score | 
| ----- | ----- | 
| Technical architecture and depth (20) | Custom resilient ingestion, Atlas concurrent hybrid retrieval (vector + BM25 + RRF), Cohere rerank, stateful LangGraph agent, single-platform data design | 
| AI reasoning and agentic execution (25) | Plan, retrieve, live_fetch, judge, verify/repair, retry, abstain loop; zero-hallucination verbatim check; visible trace | 
| Problem relevance, impact and USP (25) | Claim-to-clause bridge between fact-checking and plain-language policy explanation; Hindi voice; honest abstain | 
| Live demo and robustness (30) | Deployed app, live ingestion, edge cases shown on purpose, graceful WAF & TTS fallbacks | 

## 20. Roadmap (for the deck)

* Deepfake and audio forensics module with calibrated confidence.

* e-Gazette and state gazette ingestion; India Code section lookups.

* More Indian languages and voice input.

* WhatsApp/Telegram bot to match how forwards actually travel.

* Provenance tracking: cluster claim variants and show how a rumour mutates.