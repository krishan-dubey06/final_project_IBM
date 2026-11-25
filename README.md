# FinIndex RAG — Vectorless Personal Finance Assistant

A full-stack AI chat interface for personal finance guidance, powered by **gemini-3** and a corpus of **28 personal finance books** using hierarchical PageIndex-style retrieval (no vector embeddings).

## Architecture

```
Query
  │
  ├─ Vagueness Check → Clarification Questions
  │
  ├─ Multi-Query Generation (3-5 sub-queries)
  │
  ├─ Level 0: Book Routing (all 28 book summaries → select 1-4 books)
  │
  ├─ Level 1: Chapter Selection (per book, lazy expansion)
  │
  ├─ Level 2: Section Selection (per chapter)
  │
  ├─ Level 3: Subsection Selection (if present)
  │
  ├─ Leaf Retrieval (raw text from selected pages)
  │
  ├─ Re-Ranking:
  │     Normal Agent  → LLM pointwise scoring (<20s latency)
  │     Deep Research → Cross-encoder (sentence-transformers)
  │
  ├─ Hard Relevance Gate (min score 6.0/10)
  │
  ├─ Tavily Web Search (if corpus fails gate or temporal query)
  │
  └─ Streaming Answer Synthesis (grok-3, with inline citations)
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| LLM | grok-3 (xAI, 131K context window) via OpenAI-compatible API |
| Backend | Python 3.11, FastAPI, uvicorn |
| Tree Storage | SQLite (no vector DB) |
| Re-Ranking | LLM pointwise (Normal) + Cross-Encoder (Deep Research) |
| Web Search | Tavily API |
| Frontend | React 18, Vite, vanilla CSS |
| Deployment | Docker, AWS EC2/ECS |

## Setup

### 1. Prerequisites

- Python 3.11+
- Node.js 20+
- API Keys: xAI Grok-3, Tavily (optional: Groq, Gemini, OpenAI as fallbacks)

### 2. Environment Variables

```bash
cp .env.example .env
# Edit .env and fill in your API keys:
# XAI_API_KEY=xai-...
# TAVILY_API_KEY=tvly-...
```

**Never commit `.env` to version control.**

### 3. Install Python Dependencies

```bash
# Using uv (recommended)
pip install uv
uv pip install -e .

# Or using pip
pip install -r requirements.txt
```

Required packages:
- `fastapi`, `uvicorn[standard]`
- `pymupdf` (PDF parsing)
- `langchain-openai` (xAI Grok client)
- `sentence-transformers` (cross-encoder reranking)
- `tavily-python` (web search)
- `pydantic-settings`

### 4. Install Frontend Dependencies

```bash
cd frontend
npm install
```

### 5. Ingest the Book Corpus (one-time)

Place your 28 PDF files in `data/pdf_files/`. Then run:

```bash
# From the project root
python -m backend.ingestion.seed_all_28
```

The ingestion pipeline:
- Parses each PDF into a hierarchical tree (Book → Chapters → Sections)
- Generates LLM summaries at each level using grok-3
- Stores everything in SQLite (`data/finance_corpus.db`)
- Exports a Corpus Index (`data/corpus_index.json`)
- **Idempotent**: re-running skips already-ingested books

### 6. Run Locally (Development)

**Terminal 1 — Backend:**
```bash
python -m backend.main
# FastAPI running at http://localhost:8000
```

**Terminal 2 — Frontend:**
```bash
cd frontend
npm run dev
# Vite dev server at http://localhost:3000
# All /api/* requests proxied to localhost:8000
```

### 7. Run with Docker (Production)

```bash
# Build and start
docker-compose up --build -d

# Or with dev frontend (hot reload)
docker-compose --profile dev up --build
```

Access at `http://localhost:8000`

### 8. Deploy to AWS

**EC2:**
```bash
# On EC2 instance (Ubuntu 22.04, t3.medium+ recommended)
sudo apt-get install docker.io docker-compose-plugin -y
git clone <your-repo> finindex-rag
cd finindex-rag
cp .env.example .env  # fill in API keys
# Copy your PDF files to data/pdf_files/
docker-compose up --build -d
```

**ECS (Fargate):**
```bash
# Build and push to ECR
aws ecr get-login-password --region us-east-1 | docker login ...
docker build -t finindex-rag .
docker tag finindex-rag:latest <ecr-uri>/finindex-rag:latest
docker push <ecr-uri>/finindex-rag:latest
# Then create ECS task definition with env vars from AWS Secrets Manager
```

## Configuration

All pipeline parameters are tunable via the UI (⚙ Settings button) or via `.env`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `NUM_SUBQUERIES` | 4 | Multi-query reformulations per question |
| `TOP_K_RERANK` | 8 | Chunks kept after reranking |
| `MAX_BOOKS_TO_ROUTE` | 4 | Max books selected at Level 0 |
| `TREE_EXPANSION_BREADTH` | 3 | Max nodes per tree level |
| `TOKEN_BUDGET_BOOK_ROUTING` | 8000 | Tokens for 28 book summaries |
| `TOKEN_BUDGET_ANSWER_SYNTHESIS` | 12000 | Tokens for answer synthesis |
| `CROSS_ENCODER_ENABLED` | false | Cross-encoder in Normal Agent |
| `DEEP_RESEARCH_CROSS_ENCODER` | true | Cross-encoder in Deep Research |
| `MIN_RELEVANCE_SCORE` | 6.0 | Minimum gate score (0-10) |
| `ENABLE_WEB_SEARCH_FALLBACK` | true | Tavily fallback on corpus miss |

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/health` | GET | Health check + configuration status |
| `/api/corpus` | GET | All 28 book summaries |
| `/api/chat` | POST | Streaming SSE chat endpoint |
| `/api/config` | GET/POST | Read/update pipeline config |
| `/api/ingest` | POST | Trigger corpus ingestion |
| `/api/ingest/status` | GET | Check ingestion status |

## Agent Modes

### Normal Agent
- Single-pass retrieval → LLM pointwise reranking → streaming answer
- Target latency: **< 20 seconds**
- Best for: direct personal finance questions

### Deep Research Agent  
- Decomposes into 2-4 sub-questions
- Independent full retrieval pipeline per sub-question
- Cross-encoder reranking (more accurate)
- Iterative gap detection → Tavily web search for outdated corpus data
- Synthesizes long-form report with sectioned analysis + full source list
- Best for: complex, multi-faceted financial questions

## Book Corpus (28 books)

| Title | Author | Key Topics |
|-------|--------|-----------|
| Rich Dad Poor Dad | R. Kiyosaki | Assets vs Liabilities, Cashflow |
| The Intelligent Investor | B. Graham | Value Investing, Margin of Safety |
| The Psychology of Money | M. Housel | Behavioral Finance, Compounding |
| The Simple Path to Wealth | JL Collins | VTSAX, FIRE, Index Funds |
| I Will Teach You To Be Rich | R. Sethi | Automation, Credit Cards |
| The Total Money Makeover | D. Ramsey | Debt Snowball, Baby Steps |
| One Up On Wall Street | P. Lynch | Stock Picking, Scuttlebutt |
| A Random Walk Down Wall Street | B. Malkiel | EMH, Index Funds |
| The Little Book of Common Sense Investing | J. Bogle | Passive Investing, Low Fees |
| ... | ... | ... |

_(26 more books — see UI Corpus Browser for full list)_
