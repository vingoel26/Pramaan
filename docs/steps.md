# Pramaan (Claim-to-Clause) — Step-by-Step Execution Plan

This execution plan operationalizes **docs/PRD.md** for a two-person team (**Person A** and **Person B**) across a 10-hour build window.

---

## 1. Team Allocation & Architectural Boundary

```
                     ┌────────────────────────────────────────────────────────┐
                     │                Person B (Frontend)                     │
                     │  Next.js (App Router) + Tailwind + SSE Stream Reader   │
                     │  ClaimInput ── TracePanel ── EvidenceCard ── Audio     │
                     └───────────────────────────┬────────────────────────────┘
                                                 │ POST /verify (SSE)
                                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Person A (Backend & Agent)                        │
│ FastAPI (main.py, events.py, schemas.py)                                    │
│ └── LangGraph (extract -> plan -> retrieve -> rerank -> judge -> repair)    │
│     ├── LLM (Gemini 2.5 Flash / GPT-4o JSON)                                │
│     ├── Cohere Rerank (COHERE_QUERY_KEY)                                    │
│     ├── Sarvam AI TTS (Hindi audio)                                         │
│     └── live_fetch (Tavily search) ──┐ calls                                │
└──────────────────────────────────────┼──────────────────────────────────────┘
                                       │ ingest_url() & retrieve()
┌──────────────────────────────────────▼──────────────────────────────────────┐
│                     Person B (Data Platform & Retrieval)                    │
│ MongoDB Atlas (M0 Cluster)                                                  │
│ ├── collections: documents, chunks, ingestion_log, runs                     │
│ ├── Search Indexes: chunks_vector (1024d) & chunks_text (BM25)              │
│ ├── Hybrid Retrieval: asyncio.gather(vector, text) + Python RRF             │
│ ├── Ingestion: httpx -> Jina fallback -> BS4/PyMuPDF -> Chunker             │
│ ├── Embeddings: Cohere embed-multilingual-v3.0 (COHERE_INGEST_KEY)          │
│ └── Seeded Backfill: Laptop script (PIB RSS + 30 Flagship Schemes)          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Strict Codebase Ownership

| Directory / File | Owner | Notes |
| :--- | :--- | :--- |
| `backend/Dockerfile`, `requirements.txt` | **Person A** | Base dependencies and containerization |
| `backend/app/main.py`, `events.py`, `schemas.py` | **Person A** | Core API, SSE streaming, shared models |
| `backend/app/llm.py`, `prompts.py` | **Person A** | Structured LLM output wrapper & system prompts |
| `backend/app/graph/*` | **Person A** | LangGraph state, nodes, verbatim check & wiring |
| `backend/app/rerank.py` | **Person A** | Cohere Rerank (`COHERE_QUERY_KEY`) |
| `backend/app/language.py` | **Person A** | Bilingual explain + Sarvam TTS |
| `backend/app/runs.py` | **Person A** | Trace logging to MongoDB `runs` |
| `backend/app/retrieval/*` | **Person B** | Mongo connection, search indexes, hybrid RRF |
| `backend/app/ingest/*` | **Person B** | Ingestion pipeline, parser, chunker, embedder, scheduler, routes |
| `backend/scripts/*` | **Person B** | `setup_indexes.py`, `backfill.py` |
| `frontend/*` | **Person B** | Complete Next.js application |
| `README.md`, pitch deck | **Person B** | Architecture diagram, slides, setup docs |

---

## 2. Shared Setup (Hour 0:00 – 0:30)

Both Person A and Person B collaborate on initial bootstrapping:

1. **Git Repository Setup**:
   - Clone repo, create `main` branch, establish feature branches (`feat/engine-a`, `feat/data-ui-b`).
2. **Environment File (`.env.example` & `.env`)**:
   - Create root/backend `.env` containing:
     ```env
     LLM_API_KEY=...
     LLM_MODEL=...
     COHERE_INGEST_KEY=...       # Person B embeddings
     COHERE_QUERY_KEY=...        # Person A reranking
     COHERE_EMBED_MODEL=embed-multilingual-v3.0
     COHERE_RERANK_MODEL=rerank-multilingual-v3.0
     TAVILY_API_KEY=...          # Person A live_fetch & Person B backfill
     MONGODB_URI=...             # Shared Atlas connection string
     MONGODB_DB=pramaan
     SARVAM_API_KEY=...          # Person A TTS
     FRONTEND_ORIGIN=http://localhost:3000
     ADMIN_TOKEN=secret_admin_token_123
     MAX_CHUNKS=1500
     ```
3. **Database & API Key Verification**:
   - Person B sets up MongoDB Atlas M0 cluster, creates database user, whitelists `0.0.0.0/0` (for hackathon mobility), and shares `MONGODB_URI`.
   - Ensure two separate Cohere keys are provisioned to avoid rate-limiting collisions during demos.
4. **Contract Sign-Off**:
   - Confirm shared schemas in `backend/app/schemas.py` and `frontend/src/types.ts`.

---

## 3. Person A: Verification Engine (Agent & Backend Core)

### Step A1: Environment & FastAPI Skeleton with SSE Streaming
- **Time**: Hour 0:30 – 1:10 (40 min)
- **Files**:
  - `backend/requirements.txt`
  - `backend/app/schemas.py`
  - `backend/app/events.py`
  - `backend/app/main.py`
- **Objective**: Establish the FastAPI server with CORS, health check, and a functional Server-Sent Events (SSE) generator.
- **Detailed Actions**:
  1. Set up `requirements.txt` with: `fastapi`, `uvicorn`, `pydantic>=2.0`, `langgraph`, `langchain-core`, `cohere`, `httpx`, `pymongo`, `apscheduler`, `pytest`, `pytest-asyncio`.
  2. Implement `backend/app/schemas.py` containing `Chunk`, `Verdict`, `VerifyRequest`, and `Label` exactly per PRD Section 11.4.
  3. Implement `backend/app/events.py` with an SSE event formatter:
     - `format_sse(event_type: str, data: dict) -> str` emitting standard `data: {...}\n\n`.
  4. Implement `backend/app/main.py`:
     - Configure FastAPI app with CORS middleware allowing `FRONTEND_ORIGIN`.
     - `GET /health`: Returns `{"status": "ok", "app": "pramaan"}`.
     - `POST /verify`: Accepts `VerifyRequest`, returns `StreamingResponse(..., media_type="text/event-stream")`.
     - Send dummy test steps (`extract_claims`, `retrieve`, `judge`) followed by a dummy `Verdict` to validate SSE line-buffering.
- **Verification**: Run `curl -N -X POST http://localhost:8000/verify -H "Content-Type: application/json" -d '{"text":"test claim","lang":"en"}'`. Ensure stream yields discrete events.

---

### Step A2: Robust Structured LLM Wrapper
- **Time**: Hour 1:10 – 1:35 (25 min)
- **Files**:
  - `backend/app/llm.py`
- **Objective**: Build a typed, resilient client for structured JSON generation.
- **Detailed Actions**:
  1. Create `backend/app/llm.py`:
     ```python
     async def complete_json(system: str, user: str, schema: type[BaseModel]) -> BaseModel
     ```
  2. Configure provider (Gemini 2.5 Flash / GPT-4o) using native JSON schema or strict mode at `temperature=0.0`.
  3. Include:
     - 15-second timeout via `asyncio.wait_for`.
     - Retry logic: 2 attempts with exponential backoff on `TimeoutError` or JSON decode errors.
     - Markdown fence stripper (in case the model wraps JSON in ````json ... ````).
     - Pydantic model validation (`schema.model_validate_json(raw_text)`).
     - Custom exception `LLMError`.
- **Verification**: Write a quick script in `backend/tests/test_llm.py` asserting that passing a sample schema produces a validated Pydantic instance.

---

### Step A3: LangGraph State & Workflow Skeleton with Stubs (**GATE 1**)
- **Time**: Hour 1:35 – 2:15 (40 min)
- **Files**:
  - `backend/app/graph/state.py`
  - `backend/app/graph/nodes.py` (stubs)
  - `backend/app/graph/build.py`
  - `docs/sample_sse.txt` (Deliverable for Person B)
- **Objective**: Wire the complete StateGraph flow using stub functions, connect to `/verify`, and export a sample SSE event stream.
- **Detailed Actions**:
  1. Define `GraphState` in `graph/state.py` exactly per PRD Section 10.2.
  2. In `graph/nodes.py`, write stub implementations for:
     - `extract_claims`, `plan_sources`, `retrieve` (stub returning 2 mock chunks), `live_fetch`, `rerank`, `judge`, `verbatim_check`, `decide`, `explain_translate`, `abstain`.
  3. In `graph/build.py`, build the graph per PRD Section 10.4:
     - Add nodes, standard edges, and conditional edges (`decide`).
     - Compile graph: `graph = workflow.compile()`.
  4. In `main.py`, replace placeholder SSE generator with `graph.astream(initial_state, stream_mode="updates")`:
     - As each node yields state updates, emit `type: "step"` events.
     - Emit `type: "final"` with the final `Verdict` when the graph reaches `END`.
  5. Run a live request through `curl` and pipe output to `docs/sample_sse.txt`.
- **GATE 1 CHECKPOINT**: Provide `docs/sample_sse.txt` to Person B so frontend SSE parsing can proceed with real event contracts.

---

### Step A4: Claim Extraction & Search Planning Nodes
- **Time**: Hour 2:15 – 2:50 (35 min)
- **Files**:
  - `backend/app/prompts.py`
  - `backend/app/graph/nodes.py`
- **Objective**: Extract checkable claims from multi-lingual user input and form targeted search queries.
- **Detailed Actions**:
  1. In `prompts.py`, write `EXTRACT_SYSTEM` and `PLAN_SYSTEM` per PRD Section 16.2 A4.
  2. Implement Pydantic extraction models:
     ```python
     class ClaimItem(BaseModel):
         id: str
         text: str
         type: str
         entities: dict
         checkable: bool

     class ExtractClaimsResponse(BaseModel):
         claims: list[ClaimItem]

     class PlanSourcesResponse(BaseModel):
         query: str
         filters: dict = {}
         source_hint: str = "pib"
     ```
  3. Implement node `extract_claims(state: GraphState) -> dict`:
     - Calls `complete_json(EXTRACT_SYSTEM, state["raw_input"], ExtractClaimsResponse)`.
     - Updates `claims`. Selects first checkable claim as `current_claim`.
     - If no claims are checkable, marks verdict as `NOT_CHECKABLE`.
  4. Implement node `plan_sources(state: GraphState) -> dict`:
     - Passes `current_claim` and `state.get("failed_queries", [])` to avoid re-generating futile keywords.
     - Produces concise English search terms targeting Indian government bodies (PIB, MyScheme).
- **Verification**: Run unit test with English, Hindi, and Hinglish inputs; confirm opinions ("this scheme is terrible") are flagged `checkable=False`.

---

### Step A5: LLM Judgement, Deterministic Verbatim Span Repair & Routing
- **Time**: Hour 2:50 – 3:50 (60 min)
- **Files**:
  - `backend/app/graph/verify.py`
  - `backend/app/prompts.py`
  - `backend/app/graph/nodes.py`
- **Objective**: Implement the core integrity mechanism: judge claim against retrieved chunks and enforce code-level verbatim quote verification.
- **Detailed Actions**:
  1. In `prompts.py`, write `JUDGE_SYSTEM` per PRD Section 16.2 A5 (instructing exact verbatim quote copy from one chunk, or `SILENT`).
  2. In `graph/verify.py`, implement `verify_and_repair_quote(quote: str, chunk_text: str, min_len: int = 20) -> tuple[bool, str]`:
     - Normalize Unicode (NFKC), strip curly quotes (`"“"`, `"”"`), and collapse whitespace.
     - Check direct normalized substring inclusion (`norm_q in norm_c`).
     - **Deterministic Span Repair**: If exact match fails (e.g. LLM inserted `...` or altered punctuation), extract the first 3 words and last 3 words. Regex search for `({prefix}.{{1,450}}?{suffix})` in `chunk_text`.
     - If matched, snap and return the true continuous slice from the source chunk.
  3. Implement node `judge(state: GraphState) -> dict`:
     - Format `state["reranked"]` (top 3 chunks) and `state["current_claim"]`.
     - Call `complete_json` with `JUDGE_SYSTEM`.
  4. Implement node `verbatim_check(state: GraphState) -> dict`:
     - **CRITICAL**: Search for `chunk_id` exclusively in memory inside `state["reranked"]` (never query MongoDB).
     - Run `verify_and_repair_quote`.
     - If repaired, overwrite `state["judgement"]["quote"]` with repaired text.
     - Set `state["quote_ok"] = True/False`.
  5. Implement conditional edge function `decide(state: GraphState) -> str`:
     - If `quote_ok` and label in `{SUPPORTED, CONTRADICTED, OUTDATED, PARTIALLY_TRUE}`: return `"explain_translate"`.
     - If attempts < 2: increment `attempts`, append query to `failed_queries`, return `"plan_sources"`.
     - Else: return `"abstain"`.
  6. Implement node `abstain(state: GraphState) -> dict`:
     - Constructs honest `UNVERIFIED` verdict explaining that no official clause was verified.
- **Verification**: Test with: (a) Exact quote, (b) Quote with ellipsis `...` that matches raw chunk ends, (c) Hallucinated quote (must fail to `quote_ok=False`).

---

### Step A6: Cohere Reranking Module
- **Time**: Hour 3:50 – 4:20 (30 min)
- **Files**:
  - `backend/app/rerank.py`
  - `backend/app/graph/nodes.py`
- **Objective**: Rerank fused candidates with `rerank-multilingual-v3.0` using dedicated `COHERE_QUERY_KEY`.
- **Detailed Actions**:
  1. Implement `backend/app/rerank.py`:
     ```python
     async def rerank_chunks(query: str, candidates: list[Chunk]) -> tuple[list[Chunk], float]
     ```
  2. Use `cohere.AsyncClient(api_key=os.getenv("COHERE_QUERY_KEY"))`.
  3. Call `cohere_client.rerank(model=os.getenv("COHERE_RERANK_MODEL"), query=query, documents=[c.text for c in candidates], top_n=3)`.
  4. Update each returned `Chunk.score` with Cohere relevance score.
  5. Resilience: 10s timeout; on API failure, fallback gracefully to top 3 candidates ordered by RRF score with default `top_score = 0.5`.
  6. In `graph/nodes.py`, update `rerank` node to populate `state["reranked"]` and `state["top_rerank_score"]`.
- **Verification**: Run with dummy candidates; verify top 3 returned and `state["reranked"]` holds valid objects.

---

### Step A7: Bilingual Plain Explanation & Sarvam Audio
- **Time**: Hour 4:20 – 5:05 (45 min)
- **Files**:
  - `backend/app/language.py`
  - `backend/app/prompts.py`
  - `backend/app/graph/nodes.py`
- **Objective**: Generate simultaneous English + conversational Hindi explanations and generate Hindi TTS audio.
- **Detailed Actions**:
  1. In `prompts.py`, write `EXPLAIN_SYSTEM` per PRD Section 16.2 A7.
  2. In `language.py`:
     - Implement digit preservation check: verify digits present in `explanation_en` appear in `explanation_hi`. If missing, append English numerals in parentheses.
     - Implement `generate_tts(text_hi: str) -> str | None`:
       - Call Sarvam AI TTS (`https://api.sarvam.ai/text-to-speech`, model `bulbul:v1`, language `hi-IN`).
       - Wrap in 8s timeout.
       - Return base64 audio URI (`data:audio/wav;base64,...`).
       - On any network/API failure, log warning and return `None` (frontend automatically falls back to Web Speech API).
  3. In `graph/nodes.py`, update `explain_translate`:
     - Run bilingual LLM completion.
     - Run digit check.
     - Trigger TTS.
     - Populate `verdict` with full metadata per `Verdict` schema.
- **Verification**: Test with Hindi input and verify both `explanation_en`, `explanation_hi`, and audio URL/fallback flags are populated.

---

### Step A8: Live Fetch Discovery Node (**Integration with Person B**)
- **Time**: Hour 5:05 – 5:50 (45 min)
- **Files**:
  - `backend/app/graph/nodes.py`
- **Objective**: On-demand real-time retrieval from Tavily (`site:pib.gov.in`) when database retrieval is thin, calling Person B's `ingest_url()`.
- **Detailed Actions**:
  1. In `graph/nodes.py`, implement `live_fetch(state: GraphState) -> dict`:
     - Triggered when `len(candidates) < 3` or `top_rerank_score < 0.25` (and `not live_fetch_done`).
     - Query Tavily API (`https://api.tavily.com/search`) with:
       - `query`: `state["plan"]["query"]`
       - `include_domains`: `["pib.gov.in", "myscheme.gov.in"]`
       - `max_results`: 3
       - `include_raw_content`: True
     - For each URL found, call Person B's `backend.app.ingest.pipeline.ingest_url(url, fallback_text=raw_content, fallback_title=title)`.
     - Collect returned `Chunk` objects (which have deterministic `id`s generated by B).
     - Append fresh chunks directly into `state["candidates"]` (bypassing Atlas search index lag!).
     - Set `state["live_fetch_done"] = True`.
- **Verification**: Test with a rare query; verify Tavily retrieves URLs, `ingest_url()` runs, and fresh chunks are injected into candidates.

---

### Step A9: Main App Wiring, Run Logging & Backend Deployment (**GATE 3**)
- **Time**: Hour 5:50 – 6:35 (45 min)
- **Files**:
  - `backend/app/runs.py`
  - `backend/app/main.py`
  - `backend/Dockerfile`
- **Objective**: Wire Person B's real `retrieve` and routes into `main.py`, record traces to MongoDB, containerize, and deploy.
- **Detailed Actions**:
  1. In `backend/app/runs.py`:
     - Save executed graph trace events, latency, claim, and verdict into MongoDB `runs` collection.
  2. In `backend/app/main.py`:
     - Include Person B's ingestion router (`app.include_router(ingest_router)`).
     - Attach Person B's APScheduler startup on FastAPI lifespan.
     - Replace retrieval stub with Person B's `from backend.app.retrieval.hybrid import retrieve`.
  3. Create `backend/Dockerfile`:
     ```dockerfile
     FROM python:3.11-slim
     WORKDIR /app
     COPY requirements.txt .
     RUN pip install --no-cache-dir -r requirements.txt
     COPY . .
     ENV PORT=8000
     CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT}"]
     ```
  4. Deploy backend to **Render** or **Railway**:
     - Configure environment variables.
     - Verify `https://<backend-url>/health` returns `{"status": "ok"}`.
- **GATE 3 CHECKPOINT**: Live backend accessible over HTTPS with CORS enabled for frontend origin.

---

### Step A10: Live Tuning, Edge Testing & Swap Rehearsal
- **Time**: Hour 6:35 – 10:00 (Iterative with Person B)
- **Files**: `backend/app/prompts.py`, `backend/tests/*`
- **Actions**:
  1. Run 10+ real test claims:
     - Flagship valid schemes (PM-KISAN, Ayushman Bharat, LPG subsidy).
     - Outdated rules (Ration card deadline extensions).
     - Pure fabrications / viral WhatsApp forwards (must cleanly abstain to `UNVERIFIED`).
     - Opinions / non-checkable statements (must return `NOT_CHECKABLE`).
  2. Tune prompts to guarantee strict verbatim quotes without ellipses.
  3. Conduct **15-minute code swap session** with Person B (learn Atlas index configs and frontend stream parser; explain graph routing, span repair, and retry limits).

---

## 4. Person B: Data, Retrieval & Frontend

### Step B1: MongoDB Atlas Setup & Search Index Automation
- **Time**: Hour 0:30 – 1:15 (45 min)
- **Files**:
  - `backend/app/retrieval/mongo.py`
  - `backend/app/retrieval/indexes.py`
  - `backend/scripts/setup_indexes.py`
- **Objective**: Initialize MongoDB Atlas client, regular indexes, and create Atlas Vector Search & Atlas Search indexes in Hour 1 (so they build in the background).
- **Detailed Actions**:
  1. In `backend/app/retrieval/mongo.py`:
     - Initialize `MongoClient` with `MONGODB_URI`.
     - Define collections: `documents`, `chunks`, `ingestion_log`, `runs`.
     - Create regular indexes:
       - `documents`: unique on `url`.
       - `chunks`: `doc_id`; unique compound on `(doc_id, idx)`.
       - `ingestion_log` & `runs`: descending on `created_at`.
     - Implement `check_health() -> dict` testing connection.
  2. In `backend/app/retrieval/indexes.py` and `backend/scripts/setup_indexes.py`:
     - Define `SearchIndexModel` for:
       - **`chunks_vector`** on `chunks`: 1024 dimensions, cosine similarity, filters on `ministry`, `doc_id`.
       - **`chunks_text`** on `chunks`: BM25 text index on `text` and `title`.
     - Run script to create both indexes and poll `list_search_indexes()` until status is `queryable`.
- **Verification**: Run `python backend/scripts/setup_indexes.py` and ensure indexes are recognized on Atlas dashboard.

---

### Step B2: Resilient PIB Ingestion, Parser & Paragraph Chunker
- **Time**: Hour 1:15 – 2:15 (60 min)
- **Files**:
  - `backend/app/ingest/sources.py`
  - `backend/app/ingest/parse.py`
  - `backend/app/ingest/chunk.py`
- **Objective**: Build a resilient scraper capable of bypassing government NIC/WAF IP blocks and extracting clean, verbatim paragraph chunks.
- **Detailed Actions**:
  1. In `ingest/sources.py`:
     - Implement `fetch_page(url: str) -> str`:
       - First attempt: `httpx.AsyncClient` with desktop Chrome User-Agent (8s timeout).
       - If status != 200, status is 403, or content < 200 chars (common on cloud hosting): **automatically fall back to `https://r.jina.ai/{url}`** (or Tavily `raw_content`).
     - Implement `fetch_pib_rss() -> list[dict]`:
       - Parse PIB RSS feeds using `feedparser` to extract fresh announcement URLs and titles.
  2. In `ingest/parse.py`:
     - Parse HTML via `BeautifulSoup` (cleaning scripts, styles, navigation bars).
     - If Jina markdown is received, extract structured body.
     - Extract metadata: `title`, `ministry`, `published_at`, `url`.
  3. In `ingest/chunk.py`:
     - Implement `chunk_text(text: str, target_words: int = 250, overlap_words: int = 30) -> list[str]`.
     - Split by double newlines/paragraphs.
     - **CRITICAL**: Every chunk must be an **exact, unmodified substring** of the document text so downstream verbatim checking succeeds.
- **Verification**: Write unit tests in `tests/test_chunk.py` asserting every chunk in `chunk_text(doc)` satisfies `chunk in doc`.

---

### Step B3: Embedding Generation & Ingestion Pipeline
- **Time**: Hour 2:15 – 3:15 (60 min)
- **Files**:
  - `backend/app/ingest/embed.py`
  - `backend/app/ingest/pipeline.py`
- **Objective**: Embed text with Cohere using `COHERE_INGEST_KEY`, generate deterministic UUID5 IDs, and bulk upsert into Atlas.
- **Detailed Actions**:
  1. In `ingest/embed.py`:
     - Initialize Cohere client using `COHERE_INGEST_KEY` and `embed-multilingual-v3.0`.
     - Implement batch embedding (up to 90 items per call).
     - Handle `input_type="search_document"`.
     - Implement backoff for 429 rate limits.
  2. In `ingest/pipeline.py`:
     - Implement `ingest_url(url: str, fallback_text: str = None, fallback_title: str = None) -> dict`:
       - Compute SHA256 of text. Skip if identical in `documents`.
       - Chunk text into paragraph slices.
       - Assign deterministic chunk IDs: `id = str(uuid.uuid5(uuid.NAMESPACE_URL, f"{url}#{idx}"))`.
       - Embed chunks with Cohere.
       - Bulk upsert into `documents` and `chunks` collections using `pymongo.ReplaceOne(..., upsert=True)`.
       - Log result to `ingestion_log`.
       - Return `{"url": url, "status": "ok", "chunks_added": len(chunks), "chunks": [Chunk(...)]}` (without embeddings).
- **Verification**: Test `ingest_url` on a single PIB URL; verify document and chunks appear in MongoDB Atlas with embeddings.

---

### Step B4: Seeded Backfill Script (Local Laptop Execution)
- **Time**: Hour 3:15 – 3:45 (30 min)
- **Files**:
  - `backend/scripts/backfill.py`
- **Objective**: Populate Atlas with 1,000–1,200 high-value chunks across 25–30 flagship Indian government schemes from your laptop (to avoid cloud IP blocks).
- **Detailed Actions**:
  1. In `backend/scripts/backfill.py`:
     - Define flagship schemes list: PM-KISAN, Ayushman Bharat (AB-PMJAY), PM Awas Yojana, Ujjwala LPG, Ration Card / One Nation One Ration Card, Atal Pension Yojana, EPFO withdrawal rules, RBI UPI transaction guidelines, MSP notifications.
     - For each scheme, run a Tavily search (`site:pib.gov.in <scheme name>`, max 3 results).
     - Combine with current PIB RSS URLs.
     - Call `ingest_url()` for each found page.
     - Stop once total chunks in collection reach \~1,200 (`MAX_CHUNKS=1500` budget limit).
  2. Execute `python backend/scripts/backfill.py` locally on your laptop.
- **Verification**: Check Atlas collection count: `chunks` has > 800 documents, and `ingestion_log` records successful ingests.

---

### Step B5: Concurrent Hybrid Retrieval Engine (**GATE 2**)
- **Time**: Hour 3:45 – 4:45 (60 min)
- **Files**:
  - `backend/app/retrieval/hybrid.py`
- **Objective**: Implement concurrent Vector + BM25 search combined with Reciprocal Rank Fusion (RRF), delivering the primary query contract to Person A.
- **Detailed Actions**:
  1. In `retrieval/hybrid.py`, implement `rrf(rank_lists: list[list[str]], k: int = 60) -> list[tuple[str, float]]`:
     ```python
     def rrf(rank_lists: list[list[str]], k: int = 60) -> list[tuple[str, float]]:
         scores = {}
         for lst in rank_lists:
             for rank, cid in enumerate(lst):
                 scores[cid] = scores.get(cid, 0.0) + 1.0 / (k + rank + 1)
         return sorted(scores.items(), key=lambda x: x[1], reverse=True)
     ```
  2. Implement `async def retrieve(query: str, filters: dict | None = None, k: int = 12) -> list[Chunk]`:
     - Embed `query` with Cohere (`input_type="search_query"`).
     - Run two MongoDB aggregations concurrently via `asyncio.gather`:
       - **Vector Search**:
         ```python
         [{"$vectorSearch": {"index": "chunks_vector", "path": "embedding", "queryVector": qvec, "numCandidates": 100, "limit": 20}},
          {"$project": {"embedding": 0}}]
         ```
       - **Text Search (BM25)**:
         ```python
         [{"$search": {"index": "chunks_text", "text": {"query": query, "path": ["text", "title"]}}},
          {"$limit": 20},
          {"$project": {"embedding": 0}}]
         ```
     - If either pipeline fails (e.g. index temporary lag), fallback gracefully to the other.
     - Fuse results via `rrf()`, map back to `Chunk` models, and return top `k`.
- **GATE 2 CHECKPOINT**: Person A can import `from backend.app.retrieval.hybrid import retrieve` and receive real government chunks from Atlas.

---

### Step B6: Periodic Ingestion Scheduler & Ingestion API
- **Time**: Hour 4:45 – 5:15 (30 min)
- **Files**:
  - `backend/app/ingest/scheduler.py`
  - `backend/app/ingest/routes.py`
- **Objective**: Expose the live ingestion log to the frontend and run a background 20-minute poller.
- **Detailed Actions**:
  1. In `ingest/scheduler.py`:
     - Configure `AsyncIOScheduler` (APScheduler) to run `poll_feeds()` every 20 minutes.
     - Export `start_scheduler()` and `shutdown_scheduler()`.
  2. In `ingest/routes.py`:
     - `GET /ingestion/recent?limit=20`: Query `ingestion_log` collection descending by `created_at`.
     - `POST /admin/ingest`: Trigger backfill or feed poll on demand (guarded by `X-Admin-Token` header).
- **Verification**: Hit `http://localhost:8000/ingestion/recent` and confirm recent items return JSON array.

---

### Step B7: Next.js Frontend Scaffold & SSE Stream Consumer
- **Time**: Hour 5:15 – 6:00 (45 min)
- **Files**:
  - `frontend/package.json`
  - `frontend/src/types.ts`
  - `frontend/src/lib/stream.ts`
- **Objective**: Scaffold Next.js application, declare mirrored TypeScript schemas, and implement line-buffered SSE stream reader.
- **Detailed Actions**:
  1. Scaffold Next.js (App Router, Tailwind CSS, TypeScript):
     - `npx create-next-app@latest frontend --typescript --tailwind --app --no-eslint`
  2. In `frontend/src/types.ts`:
     - Create types mirroring backend: `Label`, `Chunk`, `Verdict`, `StepEvent`, `VerifyRequest`.
  3. In `frontend/src/lib/stream.ts`:
     - Implement `async function* streamVerification(payload: VerifyRequest): AsyncGenerator<StepEvent | { type: "final", verdict: Verdict }>`
     - Use `fetch(..., { method: "POST", headers: { "Content-Type": "application/json" } })`.
     - Read response body with `response.body.getReader()`.
     - **Buffer chunks across chunks**: split on `\n\n`, strip `data: `, and `JSON.parse` each complete event cleanly without dropping partial messages.
- **Verification**: Test stream against `docs/sample_sse.txt` or local backend; log incoming events to browser console.

---

### Step B8: Core Frontend UI Components
- **Time**: Hour 6:00 – 7:15 (75 min)
- **Files**:
  - `frontend/src/components/ClaimInput.tsx`
  - `frontend/src/components/TracePanel.tsx`
  - `frontend/src/components/EvidenceCard.tsx`
  - `frontend/src/components/ExplanationBlock.tsx`
  - `frontend/src/components/AudioPlayer.tsx`
  - `frontend/src/components/IngestionPanel.tsx`
  - `frontend/src/app/page.tsx`
- **Objective**: Build the single-page responsive UI showcasing the live agent execution and verified clause evidence.
- **Detailed Actions**:
  1. `ClaimInput`:
     - Textarea for Hindi/English claim, character counter (\~1500 chars max), submit button with loading spinner.
     - Dynamic example chips derived from recently ingested titles fetched from `/ingestion/recent`.
  2. `TracePanel`:
     - Animated vertical timeline displaying real-time events (`extract_claims`, `retrieve`, `judge`, etc.).
     - Show status badges (start/done/error/retry) and elapsed time.
  3. `EvidenceCard`:
     - Large color-coded verdict badge:
       - `SUPPORTED`: Emerald green
       - `CONTRADICTED`: Crimson red
       - `OUTDATED`: Amber orange
       - `PARTIALLY_TRUE`: Yellow
       - `UNVERIFIED`: Slate gray
       - `NOT_CHECKABLE`: Blue-gray
     - **Blockquote**: Render the verbatim quoted clause prominently with quote marks and highlighted styling.
     - Metadata footer: clickable source link, publication date, confidence gauge, and "Limits of this check" notice.
  4. `ExplanationBlock`:
     - Tabs/Toggle for English and Hindi (Devanagari font).
     - Displays the plain-language citizen impact summary.
  5. `AudioPlayer`:
     - Plays `audio_url` from Sarvam AI.
     - Fallback: If `audio_url` is null, uses `window.speechSynthesis` with `SpeechSynthesisUtterance(..., lang="hi-IN")`.
  6. `IngestionPanel`:
     - Side/Bottom panel polling `/ingestion/recent` every 30s. Shows judges live documents entering Atlas.
- **Verification**: Test UI with mock states and real stream; verify layout remains responsive on mobile screens.

---

### Step B9: Frontend Deployment on Vercel (**GATE 3**)
- **Time**: Hour 7:15 – 7:45 (30 min)
- **Files**:
  - `frontend/.env.production`
- **Objective**: Deploy Next.js frontend to Vercel and connect it to Person A's deployed backend URL.
- **Detailed Actions**:
  1. Push code to GitHub repository.
  2. Import project into Vercel.
  3. Set environment variable: `NEXT_PUBLIC_API_URL=https://<person-a-backend-url>`.
  4. Trigger build and confirm zero TypeScript or bundling errors.
  5. Test live end-to-end verification through the public Vercel URL.
- **GATE 3 CHECKPOINT**: Complete live web app verifying claims in public production.

---

### Step B10: README, Architecture Diagram, Pitch Deck & Polish
- **Time**: Hour 7:45 – 10:00 (Iterative with Person A)
- **Files**:
  - `README.md`
  - Pitch Deck (PDF / Google Slides)
- **Objective**: Document project architecture, prepare 5–7 slide presentation deck, and rehearse live demonstration.
- **Detailed Actions**:
  1. In `README.md`:
     - Write project pitch, problem statement, and solution.
     - Embed Mermaid diagram illustrating Atlas hybrid engine, LangGraph agent, and verbatim repair.
     - Document setup instructions and local testing commands.
  2. Pitch Deck (6 Slides):
     - Slide 1: Title, Team, One-line hook ("Don't just tell people it's fake. Show the exact clause.").
     - Slide 2: Problem: The Gap between Fact-Checking and Policy Explanations.
     - Slide 3: Live Solution & Architecture (MongoDB Atlas single-platform design).
     - Slide 4: Integrity Engine: Verbatim Verification & Deterministic Span Repair.
     - Slide 5: Live Demo Highlights & Edge Cases (Honest Abstain, Bilingual Audio).
     - Slide 6: Roadmap (e-Gazette, Voice Input, WhatsApp bot).
  3. Conduct **15-minute swap session** with Person A.

---

## 5. Critical Checkpoints & Gates Summary

```
Hour 0.0          Hour 2.0             Hour 4.0            Hour 6.5           Hour 8.0          Hour 10.0
   │                  │                    │                   │                  │                 │
   ├── Repo & Env ────┤── Gate 1 ──────────┼── Gate 2 ─────────┼── Gate 3 ────────┼── Freeze ───────┤
   │   - Atlas M0     │   - A: LangGraph   │   - B: Real Atlas │   - Deployed BE  │   - No new      │ Demo Video &
   │   - 2 Cohere     │     runs w/ stubs  │     hybrid search │     (Render)     │     features    │ Submission
   │     keys         │   - A: sample_sse  │   - B: backfill   │   - Deployed FE  │   - Polish &    │ Form
   │   - Schemas      │   - B: Indexes     │     complete      │     (Vercel)     │     deck        │
   │                  │     queryable      │   - A: wires B    │   - Live run     │   - Swap test   │
```

- **Gate 1 (Hour 2:00)**: LangGraph agent runs end-to-end with stubs, producing `docs/sample_sse.txt`. Person B has created MongoDB search indexes (`chunks_vector`, `chunks_text`).
- **Gate 2 (Hour 4:00)**: `retrieve()` returns real official PIB chunks from Atlas. Seed backfill has populated >800 chunks. Person A switches from stubs to B's real retrieval and pipeline.
- **Gate 3 (Hour 6:30)**: Live backend deployed on Render/Railway and live frontend on Vercel. A user can paste a claim and receive an end-to-end verified clause on the public URL.
- **Feature Freeze (Hour 8:00)**: Zero new features. Focus exclusively on prompt tuning, edge testing, UI polish, deck creation, and backup video recording.
- **Swap Session (Hour 9:30)**: 15-minute reciprocal code review so both team members can confidently defend any part of the stack during judging.

---

## 6. Critical Engineering Guardrails (Both Must Follow)

1. **Two Cohere API Keys**: Person B uses `COHERE_INGEST_KEY` exclusively for document embeddings. Person A uses `COHERE_QUERY_KEY` exclusively for reranking. This eliminates `429 Too Many Requests` during live judge evaluation.
2. **Never Return Embeddings**: In PyMongo queries and aggregations, always include `{"$project": {"embedding": 0}}`. Returning 1024-float arrays kills network bandwidth and breaches memory limits.
3. **In-Memory Verbatim Checks**: `verbatim_check` looks up candidate text strictly in `state["reranked"]` in memory. It must **never** perform a roundtrip MongoDB lookup, guaranteeing zero index-lag race conditions.
4. **Jina Reader Fallback for Government Sites**: Any scrape targeting `.gov.in` that returns 403, 5xx, or empty strings must fall back immediately to `https://r.jina.ai/{url}` or Tavily `raw_content`.
5. **Deterministic Span Repair**: When the LLM judge inserts ellipses `...` into a quote, `verify_and_repair_quote()` automatically snaps to the prefix and suffix boundary in the source chunk rather than failing the check.
6. **Graceful Audio Fallback**: If Sarvam AI TTS times out or fails, set `audio_url = None` so the frontend seamlessly switches to browser `window.speechSynthesis`.
