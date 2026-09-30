<div align="center">

# 🛡️ Vangaurd OpsSheild

### Self-RAG Incident Response Copilot for Cloud Operations

An AI-powered copilot that helps on-call engineers diagnose production incidents in real-time by grounding every troubleshooting decision in **private runbook evidence** — and falling back to verified internet context only when internal documentation falls short.

Built with **Self-Reflective Retrieval-Augmented Generation (Self-RAG)** using LangGraph, Pinecone, and Groq.

[Features](#-features) · [Architecture](#-architecture) · [Self-RAG Pipeline](#-self-rag-pipeline) · [Tech Stack](#-tech-stack) · [Getting Started](#-getting-started) · [API Reference](#-api-reference)

---

</div>

## 🔍 The Problem

During a production incident, on-call engineers face immense pressure to diagnose and resolve issues quickly. They often need to:

- Sift through scattered runbooks, SOPs, and postmortem documents
- Decide whether to trust an LLM response that might hallucinate critical operational steps
- Maintain context across multiple follow-up questions during a triage session
- Know when internal docs are insufficient and external sources are needed

Generic LLM chatbots **cannot be trusted** for incident response because they lack grounding in an organization's private operational knowledge and have no mechanism to self-verify their outputs.

## 💡 The Solution

**Vangaurd OpsSheild** implements a full **Self-RAG (Self-Reflective Retrieval-Augmented Generation)** pipeline that:

1. **Retrieves** — Searches private runbooks first via semantic vector search (Pinecone)
2. **Grades** — LLM-as-judge evaluates each retrieved document for relevance
3. **Generates** — Produces grounded incident response steps strictly from evidence
4. **Verifies** — Runs two self-reflection checks:
   - `IsSUP` (Is Supported) — Are the generated steps grounded in the retrieved evidence?
   - `IsUSE` (Is Useful) — Does the response actually address the engineer's question?
5. **Self-corrects** — Rewrites queries and retries retrieval if verification fails
6. **Escalates to Web** — Falls back to Tavily internet search only when internal evidence is exhausted
7. **Remembers** — Maintains multi-turn conversation memory via SQLite checkpointing

> Every answer is either **grounded in evidence** or the system explicitly tells you it cannot provide a reliable response — no hallucinated runbook steps, ever.

---

## ✨ Features

| Feature | Description |
|---|---|
| **Self-RAG Pipeline** | 13-node LangGraph state machine with retrieval → grading → generation → support verification → usefulness checking → self-correction loops |
| **Private Knowledge Base** | Upload runbooks (PDF, TXT, MD, DOCX) that get chunked and indexed into Pinecone for semantic retrieval |
| **Multi-Turn Memory** | SQLite-backed conversation persistence — follow-up questions maintain incident context across the session |
| **Adaptive Query Rewriting** | Automatically rewrites search queries when initial retrieval returns no relevant documents |
| **Web Search Fallback** | Tavily-powered internet search activates only after internal retrieval is exhausted |
| **Full Audit Trail** | Every query, response, Self-RAG trace, source citations, and verification status is logged to SQLite |
| **Transparent Decision Trace** | Expandable Self-RAG workflow trace shows every node the pipeline traversed for each response |
| **Source Attribution** | Every answer displays citations — internal runbook chunks or web sources with page numbers and URLs |
| **Real-Time UI** | Dark-themed incident console with starter prompts, session management, and live Self-RAG status indicators |
| **Containerized** | Production-ready Docker image with configurable environment variables |

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Vangaurd OpsSheild                          │
│                                                                    │
│  ┌──────────────┐    ┌──────────────────────────────────────────┐  │
│  │   Frontend    │    │            FastAPI Backend               │  │
│  │              │    │                                          │  │
│  │  index.html  │◄──►│  /api/chat    → Self-RAG Pipeline       │  │
│  │  app.js      │    │  /api/upload  → Document Ingestion      │  │
│  │  styles.css  │    │  /api/audits  → Audit Trail             │  │
│  │              │    │  /api/health  → Health Check             │  │
│  └──────────────┘    └──────────┬───────────────────────────────┘  │
│                                 │                                   │
│          ┌──────────────────────┼──────────────────────┐           │
│          │                      │                      │           │
│          ▼                      ▼                      ▼           │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐     │
│  │   Pinecone   │    │    SQLite DBs    │    │    Tavily    │     │
│  │ Vector Store │    │                  │    │  Web Search  │     │
│  │              │    │  audit.db        │    │              │     │
│  │  Embeddings: │    │  (query logs)    │    │  Fallback    │     │
│  │  MiniLM-L6   │    │                  │    │  when docs   │     │
│  │  (384-dim)   │    │  langgraph_      │    │  insufficient│     │
│  │              │    │  memory.sqlite   │    │              │     │
│  │  Namespace:  │    │  (conversation   │    └──────────────┘     │
│  │  incident-   │    │   checkpoints)   │                         │
│  │  runbooks    │    └──────────────────┘                         │
│  └──────────────┘                                                  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    LLM: Groq (GPT-OSS-120B)                 │  │
│  │  Structured outputs for: retrieval decisions, relevance     │  │
│  │  grading, support verification, and usefulness checking     │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Self-RAG Pipeline

The core of Vangaurd OpsSheild is a **13-node LangGraph state machine** that implements the Self-RAG paper's reflective retrieval approach, adapted for operational incident response:

```
                              ┌──────────────┐
                              │    START      │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │ Contextualize │  ← Rewrites follow-ups using
                              │   Question    │    SQLite conversation memory
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │   Decide      │  ← LLM decides: does this need
                              │  Retrieval    │    evidence or general knowledge?
                              └──┬────────┬──┘
                       ┌─────────┘        └──────────┐
                       ▼                              ▼
              ┌────────────────┐           ┌──────────────────┐
              │    Retrieve    │           │  Generate Direct  │
              │   (Pinecone)   │           │  (No Retrieval)   │
              └───────┬────────┘           └────────┬─────────┘
                      │                             │
              ┌───────▼────────┐                    │
              │ Grade Relevance│                    │
              │ (LLM-as-Judge) │                    │
              └──┬──────┬──┬──┘                    │
        relevant │      │  │ no relevant            │
                 │      │  │ docs left              │
       ┌─────────┘      │  └──────────┐             │
       ▼                │             ▼             │
┌──────────────┐  ┌─────▼──────┐  ┌──────────┐    │
│   Generate   │  │  Rewrite   │  │  Rewrite  │    │
│ from Context │  │ Internal Q │  │  Web Q    │    │
└──────┬───────┘  └─────┬──────┘  └─────┬─────┘    │
       │                │               │           │
       │            ┌───▼───┐    ┌──────▼──────┐    │
       │            │Retrieve│   │ Web Search  │    │
       │            │ Again  │   │  (Tavily)   │    │
       │            └───┬───┘    └──────┬──────┘    │
       │                │               │           │
       │                └──────┬────────┘           │
       │                       │                    │
       ▼                 (back to Grade)            │
┌──────────────┐                                    │
│ Check Support│  ← IsSUP: Is the answer            │
│  (IsSUP)     │    grounded in evidence?           │
└──┬────────┬──┘                                    │
   │        │                                       │
   │ fully  │ not supported                         │
   │        ▼                                       │
   │  ┌──────────┐                                  │
   │  │  Revise  │ ← Rewrites answer to stay        │
   │  │  Answer  │   within evidence bounds          │
   │  └────┬─────┘                                  │
   │       └──────► (back to IsSUP)                 │
   ▼                                                │
┌──────────────┐                                    │
│   Check      │  ← IsUSE: Does the answer          │
│ Usefulness   │    address the question?           │
│  (IsUSE)     │                                    │
└──┬────────┬──┘                                    │
   │        │                                       │
   │ useful │ not useful → retry or no_answer       │
   │        │                                       │
   ▼        ▼                                       │
┌──────────────┐                                    │
│   Commit     │◄───────────────────────────────────┘
│   Memory     │  ← Saves Q&A pair to SQLite thread
└──────┬───────┘
       │
┌──────▼───────┐
│     END      │
└──────────────┘
```

### Key Self-RAG Reflection Tokens

| Token | Full Name | Purpose |
|---|---|---|
| **Retrieve** | Retrieval Decision | Determines if the question requires evidence retrieval or can be answered from general knowledge |
| **IsREL** | Is Relevant | Grades each retrieved document chunk for relevance to the operational question |
| **IsSUP** | Is Supported | Verifies the generated answer is grounded in the retrieved evidence (hallucination guard) |
| **IsUSE** | Is Useful | Checks if the final response actually addresses the engineer's question |

---

## 🛠 Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Backend** | FastAPI + Uvicorn | Async REST API server |
| **LLM** | Groq (GPT-OSS-120B) | Inference for all Self-RAG nodes via `langchain-groq` |
| **Orchestration** | LangGraph | State machine for the Self-RAG pipeline with conditional edges |
| **Vector Store** | Pinecone (Serverless) | Semantic search over private runbooks (cosine similarity) |
| **Embeddings** | HuggingFace `all-MiniLM-L6-v2` | Local 384-dim embeddings (no API key required) |
| **Web Search** | Tavily API | Advanced web search fallback when internal docs are insufficient |
| **Memory** | LangGraph SQLite Checkpointer | Multi-turn conversation persistence per incident thread |
| **Audit DB** | SQLite (`audit.db`) | Logs all queries, answers, traces, sources, and verification status |
| **Document Parsing** | PyPDF, python-docx, LangChain loaders | PDF, DOCX, TXT, and Markdown ingestion |
| **Text Splitting** | LangChain `RecursiveCharacterTextSplitter` | 800-char chunks with 160-char overlap |
| **Frontend** | Vanilla HTML/CSS/JS + Jinja2 | Dark-themed incident response console |
| **Observability** | LangSmith | LLM tracing and debugging |
| **Containerization** | Docker | Production-ready deployment |

---

## 📁 Project Structure

```
vangaurd-opssheild/
├── app.py                      # FastAPI application — routes, startup, middleware
├── data_ingestion.py           # CLI script to batch-ingest documents into Pinecone
├── template.py                 # Project scaffolding utility
├── requirements.txt            # Python dependencies
├── Dockerfile                  # Production container (Python 3.11-slim + Uvicorn)
├── .env                        # Environment variables (API keys, model config)
│
├── src/                        # Core application modules
│   ├── __init__.py
│   ├── config.py               # Pydantic Settings — all env vars centralized
│   ├── self_rag.py             # ⭐ Self-RAG pipeline — 13-node LangGraph state machine
│   ├── vectorstore.py          # Pinecone client, index management, retriever factory
│   ├── ingestion.py            # Document loading, chunking, and vector upsert
│   ├── models.py               # Pydantic request/response schemas
│   └── db.py                   # SQLite audit trail — init, save, query
│
├── documents/                  # Sample incident runbooks (ingested into Pinecone)
│   ├── checkout-api-runbook.md
│   ├── deployment-rollback-sop.md
│   └── payments-high-cpu-runbook.md
│
├── templates/                  # Jinja2 HTML templates
│   └── index.html              # Incident Response Console UI
│
├── static/                     # Frontend assets
│   ├── styles.css              # Dark-themed UI with JetBrains Mono + DM Sans
│   └── app.js                  # Chat client, file upload, session management
│
├── data/                       # Runtime databases (gitignored)
│   ├── audit.db                # RAG audit trail
│   └── langgraph_memory.sqlite # Conversation checkpoints
│
├── uploads/                    # User-uploaded runbooks (runtime)
│
└── Self-Rag-Code/              # Research & experimentation
    └── self_rag.ipynb          # Jupyter notebook — Self-RAG prototyping
```

---

## 🗄 Database Schema

### Audit Trail (`data/audit.db`)

Logs every Self-RAG interaction for observability and debugging.

```sql
CREATE TABLE rag_audit (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    created_at      TEXT NOT NULL,        -- ISO 8601 UTC timestamp
    question        TEXT NOT NULL,        -- User's incident query
    answer          TEXT NOT NULL,        -- Generated response
    route           TEXT,                 -- "Private Runbooks" | "Internet Search" | "General Knowledge" | "No Reliable Evidence"
    used_web        INTEGER DEFAULT 0,   -- 1 if Tavily web search was activated
    support_status  TEXT,                 -- IsSUP result: "fully_supported" | "partially_supported" | "no_support"
    usefulness      TEXT,                 -- IsUSE result: "useful" | "not_useful"
    trace_json      TEXT,                 -- JSON array of Self-RAG pipeline trace steps
    sources_json    TEXT                  -- JSON array of source citations
);
```

### Conversation Memory (`data/langgraph_memory.sqlite`)

LangGraph's built-in SQLite checkpointer stores per-thread conversation state, enabling multi-turn incident triage without losing context.

---

## 📡 API Reference

### `POST /api/chat`

Run the Self-RAG pipeline on an incident query.

**Request:**
```json
{
  "question": "Our checkout API is returning 502 errors after deployment",
  "thread_id": "incident-abc123"
}
```

**Response:**
```json
{
  "answer": "1. Compare the incident start time with the latest deployment...",
  "route": "Private Runbooks",
  "used_web_search": false,
  "support_status": "fully_supported",
  "usefulness": "useful",
  "sources": [
    {
      "type": "internal",
      "title": "checkout-api-runbook.md",
      "source": "documents/checkout-api-runbook.md",
      "page": 1
    }
  ],
  "trace": [
    "Memory: new incident session",
    "Retrieval decision: true",
    "Internal retrieval: 5 chunks",
    "Relevance grade (internal): 3/5 relevant",
    "Generated answer from internal evidence",
    "Support check: fully_supported",
    "Usefulness check: useful",
    "SQLite memory checkpoint updated"
  ],
  "thread_id": "incident-abc123",
  "memory_turns": 1
}
```

### `POST /api/upload`

Upload a runbook document to expand the private knowledge base.

**Request:** `multipart/form-data` with file field (PDF, TXT, MD, or DOCX)

**Response:**
```json
{
  "filename": "kubernetes-troubleshooting.pdf",
  "chunks_indexed": 42,
  "namespace": "incident-runbooks"
}
```

### `GET /api/audits?limit=20`

Retrieve recent Self-RAG audit trail entries.

### `GET /api/health`

Health check endpoint for container orchestration.

---

## 🚀 Getting Started

### Prerequisites

- Python 3.11+
- [Pinecone](https://www.pinecone.io/) account (free tier works)
- [Groq](https://groq.com/) API key
- [Tavily](https://tavily.com/) API key (optional — for web search fallback)

### 1. Clone & Setup

```bash
git clone https://github.com/<your-username>/vangaurd-opssheild.git
cd vangaurd-opssheild
python -m venv .venv
source .venv/bin/activate        # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure Environment

Create a `.env` file in the project root:

```env
# LLM
GROQ_API_KEY=your_groq_api_key
GROQ_MODEL=openai/gpt-oss-120b

# Embeddings (local — no API key needed)
EMBEDDING_MODEL=all-MiniLM-L6-v2
EMBEDDING_DIMENSION=384

# Vector Store
PINECONE_API_KEY=your_pinecone_api_key
PINECONE_INDEX_NAME=vangaurd-opssheild-self-rag
PINECONE_NAMESPACE=incident-runbooks
PINECONE_CLOUD=aws
PINECONE_REGION=us-east-1

# Web Search (optional)
TAVILY_API_KEY=your_tavily_api_key

# Self-RAG Tuning
TOP_K=5
MAX_SUPPORT_RETRIES=2
MAX_RETRIEVAL_REWRITES=2
MAX_WEB_REWRITES=2
DATABASE_PATH=data/audit.db

# Observability (optional)
LANGSMITH_API_KEY=your_langsmith_key
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=CloudOps-Sentinel
```

### 3. Ingest Runbooks

```bash
# Ingest the sample runbooks into Pinecone
python data_ingestion.py
```

### 4. Run the Application

```bash
python app.py
# Or directly:
uvicorn app:app --host 0.0.0.0 --port 8080 --reload
```

Open **http://localhost:8080** in your browser.

### 5. Docker Deployment

```bash
docker build -t vangaurd-opssheild .
docker run -p 8080:8080 --env-file .env vangaurd-opssheild
```

---

## 🧪 Sample Queries

Try these to see the Self-RAG pipeline in action:

| Query | Expected Route |
|---|---|
| *"Our checkout API is returning 502 errors after deployment. What should the on-call engineer check first?"* | Private Runbooks |
| *"CPU on the payments service is above 90%. What is our recommended response procedure?"* | Private Runbooks |
| *"What should I verify when a Kubernetes pod keeps restarting with CrashLoopBackOff?"* | Internet Search (fallback) |
| *"What is the rollback procedure?"* → *"What should I verify after rolling back?"* | Private Runbooks + Memory |

---

## 📊 How Self-RAG Differs from Standard RAG

| Aspect | Standard RAG | Self-RAG (Vangaurd OpsSheild) |
|---|---|---|
| Retrieval | Always retrieves | Decides if retrieval is needed |
| Relevance | Uses all retrieved docs | LLM grades each doc individually |
| Hallucination | No grounding check | **IsSUP** verifies every answer against evidence |
| Quality | No output evaluation | **IsUSE** checks if the answer is actually helpful |
| Self-Correction | None | Rewrites queries and regenerates if checks fail |
| Fallback | None | Escalates to web search, then gracefully admits uncertainty |
| Memory | Stateless | Multi-turn SQLite thread persistence |

---

## ⚡ Key Engineering Decisions

| Decision | Rationale |
|---|---|
| **Structured JSON-mode evaluators** | All Self-RAG reflection nodes use Pydantic schemas with `json_mode` — no fragile regex parsing |
| **SHA-256 idempotent chunk IDs** | Re-uploading a runbook doesn't create duplicate vectors in Pinecone |
| **Local HuggingFace embeddings** | Zero API cost for embeddings; reduces latency and external dependency |
| **ThreadPool offloading** | Synchronous LangGraph execution runs in `run_in_threadpool` so FastAPI's async loop stays responsive |
| **Safe failure mode** | When no evidence exists, the system refuses to speculate rather than risk dangerous production commands |
| **Deterministic inference** | `temperature=0` on all LLM calls for consistent, reproducible incident triage |

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -m 'Add new feature'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

<div align="center">

**Built with** LangGraph · Pinecone · Groq · Tavily · FastAPI

</div>