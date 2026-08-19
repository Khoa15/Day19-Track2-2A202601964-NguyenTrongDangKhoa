# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a VinUniversity AI course lab (Day 19, Track 2) building a **hybrid search API** combining BM25 keyword search with vector similarity (Qdrant) and RRF fusion, plus a Feast feature store. The codebase supports two deployment paths: **Lite** (in-process, no Docker) and **Docker** (full production stack).

## Common Commands

```bash
# Lite path (default — no Docker, ~700MB RAM)
bash setup-lite.sh              # First-time setup (~60s)
make api                         # FastAPI on :8000
make benchmark                   # Precision@10 + P99 latency
make lab                         # Jupyter Lab on :8888
make test                        # pytest (~2s, 34 tests)

# Docker path (Qdrant + Redis + Postgres, ~6GB RAM)
bash setup-docker.sh             # Full stack setup (~3-8min)
make docker-up                   # Start services only

# Shared
make seed                        # Regenerate data/corpus_vn.jsonl + golden_set
make notebooks                   # Execute ALL notebooks headless (grader runs this)
make clean-lite                  # Wipe venv + data + Feast registry
```

## Architecture

### Dual-Path Design

The system is designed around two interchangeable paths controlled by `.env`:

| Path | Vector Store | BM25 | Feature Store | Embedding |
|------|-------------|------|---------------|-----------|
| Lite (default) | Qdrant in-memory | rank-bm25 | SQLite (Feast) | fastembed (384d) |
| Docker | Qdrant server (:6333) | rank-bm25 | Redis + Postgres | bge-m3 (1024d) |

Key env vars: `QDRANT_MODE` (memory|server), `EMBEDDING_BACKEND` (fastembed|multilingual|bge-m3|openai), `FEAST_ONLINE_STORE`, `FEAST_OFFLINE_STORE`.

**Switching `EMBEDDING_BACKEND` changes vector dimension — must re-index.** Run `make seed` after switching.

### Core Search Architecture (`app/search.py`)

- **Searcher** class: loads corpus, builds BM25 + vector index at startup
- Three search modes: `keyword` (BM25), `semantic` (vector), `hybrid` (RRF with k=60)
- RRF is **1-based** ranking: `score = 1/(k + rank)`, not 0-based
- Lite mode: `QdrantClient(":memory:")` — stateless each run
- Docker mode: `QdrantClient(url="http://localhost:6333")` — persists across runs

### Embedding Abstraction (`app/embeddings.py`)

`Embedder` class provides a uniform `embed(list[str]) -> Iterator[np.ndarray]` across backends. Dimension is always `self.dim` (384/1024/1536 based on model), never a constant.

### FastAPI Service (`app/main.py`)

- Single `Searcher` instance loaded at startup via lifespan context manager
- First request is slow (~30s) while embedding model loads; subsequent requests are fast
- P99 latency target: < 50ms after warmup (rubric threshold)

### Feast Feature Store (`app/feast_repo/`)

- `feature_store.yaml` configured per-path (SQLite lite vs Redis+Postgres docker)
- `feature_views.py` defines 3 feature views
- Run `feast apply` then `feast materialize` to populate online store
- If `feast apply` fails: delete `registry.db` and retry

### Notebook-Driven Curriculum (NB1-NB8)

Source of truth is Jupytext `.py` files in `notebooks/`. Edit `.py` directly or `.ipynb` in Jupyter — they're kept in sync by `make lab`.

| Notebook | Focus | Pass Criteria |
|----------|-------|---------------|
| NB1 | Embeddings + Qdrant index | 1000 vectors indexed, paraphrase query returns correct cluster |
| NB2 | Hybrid search + RRF | Hybrid Precision@10 > keyword AND semantic |
| NB3 | FastAPI + latency | P99 < 50ms server-side |
| NB4 | Feast feature store | materialize succeeds, online lookup < 10ms |

NB5-NB8 are advanced missions (filtered search, agentic retrieval, semantic cache, feature engineering).

## Testing

Tests use a **mini corpus** (60 docs, ~1s to index) to keep `make test` fast. See `tests/conftest.py` for the `mini_corpus` and `index` fixtures. Run a single test file: `python -m pytest tests/test_search.py -v`.

## Data Flow

```
scripts/seed_corpus.py → data/corpus_vn.jsonl (1000 VN docs)
                        → data/golden_set.jsonl (50 queries + expected doc_ids)

app/search.py ← corpus_vn.jsonl
             → Qdrant collection "lab19_corpus" (vectors)
             → BM25 index (in-memory)

app/main.py ← Searcher singleton
           → FastAPI /search endpoint → JSON response
```

## Key Gotchas

1. **Embedding dimension changes**: Switching `EMBEDDING_BACKEND` requires re-running notebooks
2. **RRF rank is 1-based**: Common bug is using 0-based index in the fusion formula
3. **Feast registry conflicts**: If `feast apply` errors, delete `app/feast_repo/registry.db`
4. **Notebook sync**: Both `.py` and `.ipynb` exist — edits to one don't auto-sync to the other unless using `make lab`
5. **Corpus not committed**: `data/corpus_vn.jsonl` is generated, not git-tracked. Run `make seed` first.
