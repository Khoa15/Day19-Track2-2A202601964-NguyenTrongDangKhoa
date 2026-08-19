# Plan: Day 19 Lab — Lite → Plus Implementation

## Context

Day 19 is a Vector Store + Feature Store lab for VinUniversity AICB-P2T2 (Track 2).
Two tiers:
- **Lite (Phase 2)**: NB1–NB4 — 100 pts core
- **Plus (Phase 3)**: NB5–NB8 — 50 pts advanced
- **Bonus (Phase 4)**: Architecture + agent — 20 pts optional

Repo is cloned, `.venv` active, `setup-lite.sh` completed, notebooks converted to `.ipynb`.
Goal: execute all notebooks, capture screenshots, complete reflection, submit public GitHub URL.

---

## Phase 1: Verify Environment

```bash
make verify-lite   # smoke test: Qdrant memory + BM25 + Feast SQLite
make test           # pytest — all tests must pass
```

**Verify output:** All checks pass. Qdrant in-memory, BM25, Feast SQLite online lookup green.

---

## Phase 2: Core — NB1–NB4 (100 pts)

### NB1: Embeddings & Vector Indexing (`01_embeddings_index.ipynb`)

**Stack:** `fastembed` (BAAI/bge-small-en-v1.5, 384d, ONNX CPU) + Qdrant in-memory.

**What it does:**
- Loads 1000 docs from `data/corpus_vn.jsonl`
- Batch-embed (64 docs/batch): `title + " " + text`
- Upsert to Qdrant collection `lab19`
- Test: keyword query + paraphrase query (no "cloud" literal)

**Rubric (20 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| 1a | `client.count("lab19").count == 1000` | 5 |
| 1b | Top-5 visible for keyword query | 5 |
| 1c | Paraphrase query returns cloud topic docs (no "cloud" in query) | 10 |

**Expected output:**
```
Indexed: 1000 vectors
Top-5 with cloud topic dominance on paraphrase query
```

**Vibe-coding:**
- **Delegate:** batch embed + upsert loop (mechanical, spec: batch=64, payload schema)
- **Think hard:** model choice. `bge-small-en` is English-trained — weak on Vietnamese. `bge-m3` (multilingual) is better but heavier. This is an architecture decision — specify corpus language, latency budget, re-index cost before asking AI.

---

### NB2: Hybrid Search RRF (`02_hybrid_search_rrf.ipynb`)

**Stack:** `rank-bm25` (BM25Okapi) + Qdrant vector + Reciprocal Rank Fusion.

**What it does:**
- Build BM25 index on tokenized corpus
- Build vector index (same embedder as NB1)
- Implement RRF: `score(d) = Σ 1/(k + rank_r(d))`, **rank is 1-based**, k=60
- Pull top-50 from each retriever, fuse, return top-10
- Evaluate Precision@10 on 50 golden queries for 3 modes
- Slice by query type: exact / paraphrase / mixed

**Rubric (25 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| 2a | RRF formula: 1/(k+rank), rank 1-based (not 0-based) | 10 |
| 2b | Hybrid avg Precision@10 > keyword AND > semantic | 10 |
| 2c | Slice table: hybrid wins on mixed, vector on paraphrase, BM25 on exact | 5 |

**Expected output:**
```
Precision@10 (avg over 50 queries):
  Keyword (BM25)   :  XX.X%
  Semantic (vector):  XX.X%
  Hybrid  (RRF=60) :  XX.X%

  type         n    kw      sem     hyb
  exact       XX   XX.X%   XX.X%   XX.X%
  paraphrase   XX   XX.X%   XX.X%   XX.X%
  mixed        XX   XX.X%   XX.X%   XX.X%
```

**Teaching moment:** On English-trained `bge-small-en`, semantic recall on Vietnamese paraphrases is weak (24–32%). Switching to `bge-m3` (full Docker path) helps semantic win on paraphrase queries.

**Vibe-coding:**
- **Delegate:** per-mode search wrappers, table formatting
- **Think hard:** RRF formula. Cross-check with deck §3 before delegating. If AI writes rank starting at 0 or `1/rank` instead of `1/(k+rank)`, quality silently degrades.

---

### NB3: Search API Benchmark (`03_search_api_benchmark.ipynb`)

**Stack:** FastAPI `/search?q=...&mode=kw|sem|hyb` + `time.perf_counter()` server-side.

**What it does:**
- Start FastAPI server
- 10 warmup queries (cold start skews P99 — must warm up before measuring)
- 100 iterations per mode, measure server-side latency
- Report P50/P95/P99 per mode

**Rubric (25 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| 3a | `/search` returns valid SearchResponse with `latency_ms` field | 5 |
| 3b | P50/P95/P99 latency table for 3 modes printed | 10 |
| 3c | Hybrid P99 server-side < 50 ms after warmup | 10 |

**Commands:**
```bash
make api &      # FastAPI on :8000 (background)
make benchmark  # Precision@10 + latency table
```

**Vibe-coding:**
- **Delegate:** latency table formatting, SearchResponse schema
- **Think hard:** what to measure. Server-side `time.perf_counter()` excludes network. P99 after warmup, not cold start.

---

### NB4: Feast Feature Store (`04_feast_feature_store.ipynb`)

**Stack:** Feast + SQLite (lite) / Redis (docker) online store.

**Actual feature views (from `app/feast_repo/feature_views.py`):**

| Feature View | Entity | Features |
|---|---|---|
| `user_profile_features` | user | `reading_speed_wpm`, `preferred_language`, `topic_affinity` |
| `item_popularity_features` | item | `click_count_24h`, `ctr_7d`, `avg_dwell_seconds` |
| `query_velocity_features` | user | `queries_last_hour`, `distinct_topics_24h` |

**What it does:**
- `feast apply` — register 3 feature views
- `feast materialize-incremental` — populate online store
- `get_online_features()` — P99 < 10 ms lookup
- `get_historical_features()` — PIT join (point-in-time, avoids leakage)

**Rubric (30 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| 4a | `feast apply` succeeds — 3 feature views registered | 5 |
| 4b | `materialize-incremental` succeeds — rows materialized | 5 |
| 4c | `get_online_features()` returns valid dict for `u_001` | 5 |
| 4d | 100-call online lookup P99 < 10 ms | 5 |
| 4e | PIT join via `get_historical_features()` returns 3 rows × N features | 5 |
| REPRO | Reproducible from clean `bash setup-lite.sh && make benchmark` | 5 |

**Fix:** If `feast apply` fails on retry, delete `app/feast_repo/registry.db` and re-apply.

**Vibe-coding:**
- **Delegate:** feature view boilerplate, online lookup code
- **Think hard:** entity/ttl/source design per feature. PIT join correctness.

---

## Phase 3: Advanced — NB5–NB8 (50 pts)

### NB5: Filtered Search (`05_filtered_search.ipynb`)

**Stack:** `app.filters.FilteredIndex` + brute-force ground truth.

**Three strategies:**

| Strategy | How | Failure mode |
|---|---|---|
| **post-filter** | ANN first → discard non-matching | recall crashes at tight selectivity |
| **pre-filter** | filter first → exact scan | always correct but loses index |
| **filtered-ANN** | filter inside index loop | correct + fast |

**Rubric (10 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| 5a | Recall table: post-filter drops sharply at tight filters, filtered-ANN holds 1.00 | 5 |
| 5b | Over-fetch ladder: fetch_k ≈ 50% corpus needed to recover recall | 5 |

**Expected output:** Two tables — recall cliff at selectivity ~4%, over-fetch ladder.

**Vibe-coding:**
- **Delegate:** table loops, format, Filter model construction
- **Think hard:** ground truth definition. Must be top-K *within filtered subset*, not top-K of whole corpus. Wrong ground truth → post-filter looks fine.

---

### NB6: Agentic Retrieval (`06_agent_retrieval.ipynb`)

**Stack:** `app.agent` — rule-based planner (zero LLM), retrieval-as-a-tool.

**What it does:**
- Plan: split multi-part question into sub-queries
- Execute: same budget (16 docs total) for single-shot vs agentic
- Measure: recall + **balance** (coverage of both parts of compound question)
- Reflect: retry with relaxed filter when too few results

**Rubric (12 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| 6a | Agentic > single-shot on recall AND balance at same 16-doc budget | 5 |
| 6b | Explanation: why `agentic (+filter)` is lower than `agentic (no filter)` | 4 |
| 6c | `build_context()` runs, outputs both Feast features and doc_ids | 3 |

**Expected output:** Table with Δrecall, Δbalance, calls, ms per strategy.

**Vibe-coding:**
- **Delegate:** JSON schema, eval loop, table printing
- **Think hard:** budget comparison. Agent must NOT get 32 docs vs single-shot 16 — equal budget is the only valid comparison.

---

### NB7: Semantic Cache (`07_semantic_cache.ipynb`)

**Stack:** `app.cache.SemanticCache` — Qdrant collection as vector store.

**Three parameters, three failure modes:**

| Param | Set wrong | Consequence |
|---|---|---|
| `threshold` | too low | **false hit** — answer from different question |
| `ttl_s` | missing | **stale hit** — old answer persists |
| `namespaced` | missing | **tenant leak** — OWASP LLM08:2025 |

**Sweep must measure TWO columns:** hit rate (savings) + false-hit rate (wrong answers).

**Rubric (12 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| 7a | Sweep table has BOTH columns: savings + wrong answers | 5 |
| 7b | Threshold selection with explanation why 0.75 is insufficient | 4 |
| 7c | Cross-tenant leak demo: GLOBEX reads ACME data when namespaced=False | 3 |

**Expected output:** Three tables (threshold sweep, TTL, leak demo).

**Vibe-coding:**
- **Delegate:** wrapper put/get, sweep loop, TTL virtual clock
- **Think hard:** false-hit definition. Must measure both positive (should hit) and negative (should miss) sides. Hit rate alone is meaningless.

---

### NB8: Feature Engineering (`08_feature_engineering.ipynb`)

**Stack:** `app.features` (pandas/numpy) + Feast on-demand feature view.

**Six feature families (from notebook, not 4 as initially planned):**

| # | Family | Example |
|---|---|---|
| 1 | Window aggregates | searches_1h, searches_7d |
| 2 | Ratios / normalization | query_len / user_avg |
| 3 | Lag & delta | prev_query_len, query_len_delta |
| 4 | Recency | seconds_since_last |
| 5 | Categorical encoding | frequency / target encoding |
| 6 | Embedding as feature | user/item vector |

**What it does:**
- Generate event log: 200 users, 30 days, search + click
- Measure: AUC train vs holdout (feature signal fidelity)
- Leakage experiment: target-naive vs target-in-fold on session_id vs user_id
- PIT join vs latest-value join: AUC difference
- On-demand feature view: `amount_vs_avg` computed at request time

**Rubric (16 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| 8a | AUC table: target-naive gap > 0.30 on session_id, in-fold ≈ 0 | 4 |
| 8b | PIT vs latest join: % leaked rows + AUC difference | 4 |
| 8c | On-demand feature view: same user, two amounts → two different ratios | 4 |
| TESTS | `make test` and `make verify-lite` pass | 4 |

**Fix:** feast 0.65 — must use `from feast.on_demand_feature_view import on_demand_feature_view`, not `from feast import on_demand_feature_view`.

**Vibe-coding:**
- **Delegate:** groupby/rolling, describe(), event log generator
- **Think hard:** split → encode order. If you encode before splitting, holdout labels leak into encoding — the gap disappears and you conclude target encoding is safe when it is not.

---

## Phase 4: Bonus Challenge (20 pts optional)

### `bonus/ARCHITECTURE.md` (~600–1000 words)

- Architecture diagram (Mermaid / ASCII / hand-drawn)
- 3 architecture decisions with explicit tradeoff (X vs Y, why X)
- Rejected alternative with reason
- Vietnamese-context awareness (tokenizer, code-switching, Decree-13)

### `bonus/agent.py` (~80–150 lines)

```python
class HybridMemoryAgent:
    def remember(self, text: str, user_id: str = "u_001") -> None: ...
    def recall(self, query: str, user_id: str = "u_001") -> str: ...
```

### `bonus/demo.py` (5 queries)

1. Simple lookup: "Kubernetes documents?"
2. Profile-needed: "What to read next?" (uses `topic_affinity`)
3. Fresh activity: "What am I focused on?" (uses `queries_last_hour`)
4. Paraphrase: "Scale infrastructure docs?" (vector wins)
5. Mixed: "Cloud security summary" (hybrid + profile)

**Rubric (20 pts):**

| # | Criterion | Pts |
|---|-----------|----:|
| B1 | ARCHITECTURE.md exists, ≥ 600 words, diagram | 3 |
| B2 | 3 decisions with explicit tradeoff | 6 |
| B3 | At least 1 decision with Vietnamese-context awareness | 2 |
| B4 | Rejected alternative with reason | 2 |
| B5 | agent.py runs (remember + recall) | 4 |
| B6 | demo.py exits 0 with 5 query outputs | 3 |

---

## Phase 5: Screenshot, Reflection & Submission

### Screenshots (`submission/screenshots/`)

Capture minimum 1 screenshot per notebook showing key deliverable:

| Notebook | Screenshot |
|----------|------------|
| NB1 | 1000 vectors indexed + paraphrase query top-5 cloud topic |
| NB2 | Precision@10 table (hybrid > kw, hybrid > sem) + slice table |
| NB3 | API response sample + P50/P95/P99 latency table |
| NB4 | feast apply STDOUT + online lookup result + PIT join |
| NB5 | Recall cliff table + over-fetch ladder |
| NB6 | Strategy comparison table (agentic vs single-shot) |
| NB7 | Threshold sweep (savings + wrong) + leak demo |
| NB8 | Leakage table + PIT vs latest + on-demand output |

### Reflection (`submission/REFLECTION.md`)

Answer ≤ 200 words:

> **Which mode wins on which query type and why? When would you NOT use hybrid?**

Draft answer:
- **exact**: BM25 wins — verbatim technical terms in corpus
- **paraphrase**: semantic/vector wins — semantic rephrasing without exact terms
- **mixed**: hybrid wins — captures both semantic intent and keyword signals

**Do NOT use hybrid when:**
1. Latency is critical and BM25 alone meets quality bar
2. Corpus is small where hybrid overhead exceeds benefit
3. All queries are pure exact term matches where BM25 is sufficient
4. Embedding model is wrong for the language (e.g., English model on Vietnamese corpus)

### Git & Submit

```bash
git add -A
git commit -m "Day 19 submission — <Họ Tên>"
git push -u origin main
```

Submit public GitHub URL to VinUni LMS Day-19 box.

**Keep repo public until grades released.**

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `python3: command not found` | Install Python 3.10+ |
| NB1: `expected 1000 indexed, got X` | Run `make seed` |
| NB4: `feast apply` fails on retry | Delete `app/feast_repo/registry.db` |
| NB3: P99 > 50ms | Warm up 10 queries before measuring |
| Docker: Qdrant port conflict | `docker compose down` then `docker compose up -d` |
| Docker: Qdrant timeout on first run | Wait 60s; image first pull ~200MB |
| NB8: feast `on_demand_feature_view` import error | Use full path: `from feast.on_demand_feature_view import ...` |
| Python 3.14: pyarrow / dill crash | `setup-lite.sh` auto-detects and applies override |
| Embedding model change | Must re-index corpus (dimension changes) |

---

## Verification Checklist

- [ ] `make verify-lite` → all checks pass
- [ ] `make test` → all tests pass
- [ ] NB1: 1000 vectors indexed; paraphrase query returns cloud topic
- [ ] NB2: Hybrid avg P@10 > keyword AND > semantic
- [ ] NB3: `/search` returns latency_ms; Hybrid P99 < 50ms after warmup
- [ ] NB4: feast apply + materialize + online P99 < 10ms + PIT join
- [ ] NB5: Post-filter recall cliff visible; filtered-ANN holds 1.00
- [ ] NB6: Agentic > single-shot at same 16-doc budget
- [ ] NB7: Sweep table with savings + wrong columns; leak demo works
- [ ] NB8: Leakage gap > 0.30 on session_id; on-demand feature view works
- [ ] Screenshots in `submission/screenshots/`
- [ ] `submission/REFLECTION.md` completed (≤ 200 words)
- [ ] Bonus: `bonus/ARCHITECTURE.md` + `bonus/agent.py` + `bonus/demo.py`
- [ ] Repo is public
- [ ] GitHub URL submitted to LMS
