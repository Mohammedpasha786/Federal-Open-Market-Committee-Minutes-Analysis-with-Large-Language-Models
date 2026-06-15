# FOMC Minutes Analyzer — RAG Pipeline with LLMs

> **MathWorks Student Competition Project** | MATLAB · PostgreSQL · pgvector · OpenAI API

A Retrieval-Augmented Generation (RAG) system that fetches, embeds, and semantically queries Federal Open Market Committee (FOMC) meeting minutes, enabling natural-language analysis of U.S. monetary policy documents.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Modules](#modules)
- [Running Tests](#running-tests)
- [CI/CD](#cicd)
- [Workflow Diagram](#workflow-diagram)
- [References](#references)

---

## Overview

The pipeline automates five stages:

| Stage | Description |
|-------|-------------|
| **Fetch** | Scrapes FOMC calendar page and downloads meeting minutes as plain text |
| **Preprocess** | Cleans HTML, normalises whitespace, splits into overlapping word-window chunks |
| **Embed** | Encodes each chunk via `all-MiniLM-L6-v2` (local) or OpenAI `text-embedding-3-small` (fallback) |
| **Store** | Upserts chunk vectors into PostgreSQL with the `pgvector` extension |
| **Query** | Embeds a user question → cosine ANN search → top-K retrieval → LLM answer generation |

A sentiment scorer (`fomc_sentiment_analysis.m`) additionally labels each document Hawkish / Neutral / Dovish and plots tone over time.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         FOMC RAG Pipeline                        │
│                                                                  │
│  ┌─────────────┐    ┌────────────────┐    ┌──────────────────┐  │
│  │  Fed Reserve│───▶│ fetch_fomc_    │───▶│ preprocess_and_  │  │
│  │  Website    │    │ documents.m    │    │ chunk.m          │  │
│  └─────────────┘    └────────────────┘    └────────┬─────────┘  │
│                                                    │             │
│                                           ┌────────▼─────────┐  │
│                                           │  embed_chunks.m  │  │
│                                           │  (MiniLM/OpenAI) │  │
│                                           └────────┬─────────┘  │
│                                                    │             │
│                                           ┌────────▼─────────┐  │
│                                           │  PostgreSQL +    │  │
│                                           │  pgvector        │  │
│                                           └────────┬─────────┘  │
│                                                    │             │
│  ┌─────────────┐    ┌────────────────┐    ┌────────▼─────────┐  │
│  │  LLM Answer │◀───│  call_llm.m    │◀───│  rag_query.m     │  │
│  │  (GPT-4o)   │    │                │    │  (cosine ANN)    │  │
│  └─────────────┘    └────────────────┘    └──────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
fomc-rag/
├── src/
│   ├── fomc_rag_pipeline.m          # Main entry point
│   ├── fetch_fomc_documents.m       # Web scraper & HTML stripper
│   ├── preprocess_and_chunk.m       # Text cleaner & sliding-window chunker
│   ├── embed_chunks.m               # Sentence embedding (MiniLM / OpenAI)
│   ├── store_vectors.m              # pgvector upsert
│   ├── rag_query.m                  # Query → retrieve → prompt
│   ├── call_llm.m                   # OpenAI chat completion wrapper
│   └── fomc_sentiment_analysis.m    # Hawkish / Dovish scorer + plot
├── data/
│   └── init_db.sql                  # PostgreSQL schema + pgvector setup
├── tests/
│   └── test_pipeline.m              # MATLAB unit tests (matlab.unittest)
├── docs/
│   └── FOMC_RAG_Project_Report.docx # Full project report
├── .github/
│   └── workflows/
│       └── ci.yml                   # GitHub Actions CI
└── README.md
```

---

## Prerequisites

| Dependency | Version | Purpose |
|---|---|---|
| MATLAB | R2023b+ | Core runtime |
| Text Analytics Toolbox | – | `tokenizedDocument`, `bag of words` |
| Database Toolbox | – | `postgresql()`, `fetch()`, `exec()` |
| Deep Learning Toolbox | – | `sentenceEmbeddingModel` |
| Statistics & ML Toolbox | – | Similarity metrics |
| PostgreSQL | 14+ | Vector database host |
| pgvector extension | 0.5+ | `vector` type + ANN index |
| OpenAI API key | – | LLM inference (GPT-4o) |

---

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/fomc-rag.git
cd fomc-rag

# 2. Set up PostgreSQL database
psql -U postgres -f data/init_db.sql

# 3. Set environment variables
export OPENAI_API_KEY=sk-...
export PGUSER=postgres
export PGPASSWORD=yourpassword

# 4. (Optional) Install MiniLM add-on from MATLAB Add-On Explorer
#    Search: "all-MiniLM-L6-v2 sentence embedding"
```

---

## Configuration

All settings live at the top of `fomc_rag_pipeline.m`:

```matlab
config.llm_model    = 'gpt-4o';          % LLM model
config.embed_model  = 'all-MiniLM-L6-v2'; % Embedding model
config.top_k        = 5;                  % Retrieved chunks per query
config.chunk_size   = 512;                % Words per chunk
config.chunk_overlap= 64;                 % Overlap between chunks
config.db_host      = 'localhost';
config.db_port      = 5432;
config.db_name      = 'fomc_rag';
```

---

## Usage

### Full Pipeline (Fetch → Embed → Store → Query)

```matlab
cd src
fomc_rag_pipeline
```

The script enters an interactive Q&A loop after ingestion:

```
Query: What was the FOMC's stance on inflation in 2023?
Answer: Based on the March 2023 minutes, the Committee noted that...
```

### Sentiment Analysis Only

```matlab
docs = fetch_fomc_documents('https://www.federalreserve.gov/monetarypolicy/fomccalendars.htm');
results = fomc_sentiment_analysis(docs, config);
disp(results)
```

### One-shot Query (Database Already Populated)

```matlab
config = struct('llm_api_key', getenv('OPENAI_API_KEY'), ...
                'llm_model', 'gpt-4o', 'top_k', 5, ...
                'embed_model', 'all-MiniLM-L6-v2', ...
                'db_host', 'localhost', 'db_port', 5432, ...
                'db_name', 'fomc_rag', 'db_user', getenv('PGUSER'), ...
                'db_pass', getenv('PGPASSWORD'));

answer = rag_query('How did the Fed respond to rising unemployment?', config);
disp(answer)
```

---

## Modules

### `fetch_fomc_documents.m`
Scrapes the FOMC calendar page, extracts all minutes URLs matching `/fomcminutes<YYYYMMDD>.htm`, downloads each page, and returns a struct array with `.text`, `.date`, `.url`.

### `preprocess_and_chunk.m`
Lowercases, removes non-ASCII noise, splits text into word arrays, then slides a window of `chunk_size` words with `overlap` stride across each document.

### `embed_chunks.m`
Attempts local `sentenceEmbeddingModel('all-MiniLM-L6-v2')`. Falls back transparently to the OpenAI `text-embedding-3-small` endpoint in batches of 100.

### `store_vectors.m`
Connects to PostgreSQL via the Database Toolbox native interface, creates the `fomc_chunks` table and `ivfflat` index if absent, and bulk-inserts chunk embeddings as pgvector literals.

### `rag_query.m`
Embeds the user query, executes a cosine ANN search (`<=>` operator) against `fomc_chunks`, assembles a context block from the top-K results, and calls `call_llm.m` with a system prompt constraining the model to answer only from retrieved text.

### `call_llm.m`
Thin wrapper around `webwrite` sending a JSON chat-completion request to `https://api.openai.com/v1/chat/completions`.

### `fomc_sentiment_analysis.m`
Counts hawkish vs. dovish keyword hits per document, computes a normalised score, labels each meeting, calls the LLM for a one-sentence summary, and renders a bar chart of tone over time.

---

## Running Tests

```matlab
cd tests
results = runtests('test_pipeline');
disp(results)
```

Five unit tests cover: HTML stripping, chunk count sanity, chunk overlap verification, pgvector literal format, and empty document handling.

---

## CI/CD

GitHub Actions runs on every push and pull request to `main`. See [`.github/workflows/ci.yml`](.github/workflows/ci.yml) for full configuration.

---

## References

- [FOMC Calendars — Federal Reserve](https://www.federalreserve.gov/monetarypolicy/fomccalendars.htm)
- [LLMs with MATLAB — GitHub](https://github.com/matlab-deep-learning/llms-with-matlab)
- [RAG in Finance using MATLAB — GitHub](https://github.com/ydong9107/RAGinFinance)
- [PostgreSQL Native Interface — MathWorks](https://www.mathworks.com/help/database/postgresql-native-interface.html)
- [pgvector — GitHub](https://github.com/pgvector/pgvector)
- Pisaneschi, B. (2024). *RAG for Finance: Automating Document Analysis with LLMs*. CFA Institute.

---

*Mohammed Afreed Pasha,Ashraf · SR University, Warangal *
