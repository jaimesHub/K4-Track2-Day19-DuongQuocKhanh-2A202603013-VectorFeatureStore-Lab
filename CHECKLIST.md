# Checklist — Lab 19: Vector Store + Feature Store

Đây là danh sách kiểm tra từng bước cho học viên hoàn thành lab ngày 19. Mỗi mục liệt kê công việc cần làm, tiêu chí đạt, điểm rubric, và ảnh chụp cần lưu.

> **Lưu ý:** Bạn cần chạy các lệnh từ repo root. Tất cả lệnh `make` sẽ tự động dùng `.venv/bin/*` — **không cần activate venv**.

---

## 1. Chuẩn bị (Setup)

### 1.1 Kiểm tra Python version
- [x] Kiểm tra Python 3.10–3.14 đã cài: `python3 --version`
- [ ] Nếu lỗi, download từ https://www.python.org/downloads/
- [x] Trong repo, file `.python-version` sẽ tự chỉ định version (hiện 3.11.15)

### 1.2 Chạy setup
- [x] Chạy lệnh (lite path, mặc định, khuyến cáo): `bash setup-lite.sh`
  - Lệnh này sẽ tạo `.venv`, cài đặt dependencies, sinh corpus, và chạy smoke test
  - Mất khoảng 60 giây
  - Đợi cho đến khi thấy: `All checks passed`
- [ ] **HOẶC** nếu bạn có Docker/Podman (full stack): `bash setup-docker.sh`
  - Cần ≥8 GB RAM free, ports 6333/6379/5432 không xung đột
  - Tự động pull Qdrant server + Redis + Postgres (~500 MB)

### 1.3 Tạo .env
- [x] Copy `cp .env.example .env`
- [x] Mở `.env` và chọn embedding model (mặc định: `EMBEDDING_BACKEND=fastembed`):
  - `fastembed`: Mặc định, 384d, tiếng Anh (yếu trên câu hỏi VN)
  - `multilingual` / `bge-m3`: Tốt hơn cho tiếng Việt (1024d) — **NB2 bài học chính là thử cái này**
  - `openai`: Cần `OPENAI_API_KEY`, tính tiền
- [x] Lưu `.env`

### 1.4 Sinh corpus dữ liệu
- [x] Chạy: `make seed`
  - Tạo `data/corpus_vn.jsonl` (1000 docs tiếng Việt, 10 chủ đề × 100 docs/chủ đề)
  - Tạo `data/golden_set.jsonl` (50 queries golden cho đánh giá)
  - Xác định → mọi lần chạy, corpus luôn giống nhau
- [ ] **Quan trọng:** Tests phụ thuộc file này — nếu thiếu, tests sẽ skip (không fail)

### 1.5 Smoke test
- [x] Chạy: `make verify-lite` (lite path) hoặc `make verify-docker` (docker path)
  - Nếu xanh: setup thành công
  - Nếu đỏ: xem mục Troubleshooting ở cuối

---

## 2. Khối Core (NB1–NB4) — 100 điểm

### NB1 — Embeddings & Vector Indexing — 20 điểm

**Mục tiêu:** Hiểu embedding + Qdrant indexing. Index 1000 docs, tìm kiếm semantic.

**Công việc:**
- [x] Mở Jupyter Lab: `make lab` (trên http://localhost:8888)
- [x] Mở `01_embeddings_index.ipynb`
- [x] Chạy từng cell theo hướng dẫn:
  - Load corpus từ `data/corpus_vn.jsonl`
  - Tạo embedder từ `fastembed` (ONNX, CPU)
  - Tạo Qdrant in-memory collection "lab19"
  - Embed + index 1000 docs (topics: cloud, AI, data, …)
  - Test semantic search: query tiếng Việt → top-5 results
- [x] Kiểm tra cell cuối: `client.count("lab19").count == 1000`

**Tiêu chí đạt (rubric):**
- ✅ **5 pts:** `client.count("lab19").count == 1000` (indexed = 1000 vectors)
- ✅ **5 pts:** Top-5 results visible cho keyword query (cell §5 output)
- ✅ **10 pts:** Paraphrase query (không chứa từ "cloud") trả top-5 về topic "cloud" chủ yếu
  - Ví dụ: "Xử lý dữ liệu trên máy chủ ảo" → phải có docs về cloud computing

**Ảnh chụp cần lưu:**
- [x] Screenshot: Cell thể hiện corpus đã load (10 docs)
- [x] Screenshot: Cell với 1000 indexed vectors
- [x] Screenshot: Paraphrase query + top-5 results (phải thấy topic/title)

**Vibe-coding callout:** Jupyter cells chủ yếu là boilerplate; focus vào hiểu *vì sao* embedding + cosine similarity hoạt động.

---

### NB2 — Hybrid Search: BM25 + Vector + RRF — 25 điểm

**Mục tiêu:** BM25 (keyword) + vector (semantic) + RRF fusion. Đo Precision@10 trên 50 golden queries.

**Công việc:**
- [x] Mở `02_hybrid_search_rrf.ipynb`
- [x] Xây dựng BM25 từ corpus (whitespace tokenizer)
- [x] **QUAN TRỌNG:** Implement RRF (Reciprocal Rank Fusion) với công thức:
  ```
  score(doc_d) = sum [ 1/(k + rank_r) ] cho tất cả rank_r của doc_d
  k = 60 (tuning parameter)
  rank là 1-based (first = rank 1, KHÔNG phải 0-based!)
  ```
  - Đây là **điểm hay hỏi nhất** trong lab — rank 0-based → fail
- [x] Load 50 golden queries từ `data/golden_set.jsonl`
- [x] Với mỗi query, chạy 3 mode: keyword, semantic, hybrid
- [x] Tính Precision@10 cho mỗi query, mỗi mode
- [x] In bảng kết quả:
  - Cột: keyword, semantic, hybrid
  - Hàng: avg, min, max, stddev
  - **Hybrid phải thắng cả keyword LẪN semantic** về trung bình

**Tiêu chí đạt (rubric):**
- ✅ **10 pts:** `search_hybrid` implement đúng công thức RRF `1/(k + rank)`, rank 1-based
- ✅ **10 pts:** Bảng Precision@10 average: hybrid > keyword AND hybrid > semantic
- ✅ **5 pts:** Bảng slice theo loại query:
  - Exact query (từ literal trong corpus) → BM25 thắng
  - Paraphrase query (diễn đạt lại, không literal) → vector thắng
  - Mixed query → hybrid thắng

**Ảnh chụp cần lưu:**
- [x] Screenshot: Bảng Precision@10 trung bình (hybrid > 2 mode khác)
- [x] Screenshot: Bảng slice theo loại query

**Vibe-coding callout:** Code fusion + evaluate có boilerplate nhiều; nhưng **công thức RRF phải review cẩn thận** — một dòng sai rank làm fail rubric.

**Hướng dẫn nếu hybrid không thắng:**
- Kiểm tra lại rank formula: rank 1-based không phải 0-based?
- Embedding model yếu trên tiếng Việt? Thử đổi `EMBEDDING_BACKEND=bge-m3` trong `.env`, rồi chạy lại NB1 + NB2
- BM25 tokenizer đúng (whitespace)? Không dùng phụ thuộc NLP heavy

---

### NB3 — FastAPI Endpoint + Latency Benchmark — 25 điểm

**Mục tiêu:** Bọc Searcher thành REST API. Đo P50/P95/P99 latency. Đảm bảo P99 hybrid < 50 ms.

**Công việc:**
- [x] Mở `03_search_api_benchmark.ipynb`
- [x] API sẽ khởi động background (uvicorn trên port 8000)
- [x] Test `/search?q=<query>&mode=<mode>&top_k=<top_k>` endpoint
  - `mode`: keyword, semantic, hybrid
  - Response format: `SearchResponse` với fields `results`, `latency_ms`, …
- [x] Chạy 10 warmup queries (cold start latency không đáng tin)
- [x] Chạy 100 production queries, đo latency từng query
- [x] Tính percentiles: P50, P95, P99 cho 3 mode
- [x] In bảng: 
  - Cột: keyword, semantic, hybrid
  - Hàng: P50, P95, P99

**Tiêu chí đạt (rubric):**
- ✅ **5 pts:** FastAPI `/search` trả về valid `SearchResponse` với field `latency_ms`
- ✅ **10 pts:** Bảng P50/P95/P99 cho 3 mode (server-side measurement)
- ✅ **10 pts:** **P99 hybrid server-side < 50 ms** (QUAN TRỌNG)
  - Nếu > 50 ms ở cold start → chạy warmup, rồi đo lại
  - Đã đo (sau warm-up): hybrid P99 = 22.3 ms (< 50 ms) ✔

**Ảnh chụp cần lưu:**
- [x] Screenshot: 1 sample response JSON từ `/search` endpoint (thấy latency_ms, results)
- [x] Screenshot: Bảng P50/P95/P99 (3 mode)

**Vibe-coding callout:** Subprocess + httpx boilerplate cho AI. Nhưng **warm-up pattern** phải bạn tự nghĩ: tại sao cold start lớn? Khi nào có thể skip?

**Hướng dẫn P99 > 50 ms:**
- Bình thường ở cold start (model load + index build 1000 docs lần đầu)
- **Fix:** Chạy 10–20 warmup query trước đo, hoặc dùng `make api &` chạy ở terminal riêng rồi đợi Searcher load
- Hybrid P99 thường ~20–40 ms sau warmup → accept

---

### NB4 — Feast Feature Store — 25 điểm

**Mục tiêu:** Setup Feast với 3 feature views. Materialize + online lookup. PIT join cho training data.

**Công việc:**
- [x] Mở `04_feast_feature_store.ipynb`
- [x] Xem `app/feast_repo/feature_views.py` — 3 views định sẵn:
  1. User feature view (user_id → features như user_age, device, …)
  2. Item feature view (doc_id → features như doc_topic, publish_date, …)
  3. Query-doc interaction view (query_id + doc_id → click, dwell_time, …)
- [x] **Quan trọng:** Đừng thêm feature view mới — rubric assert đúng 3 views này
- [x] Chạy Feast apply:
  ```bash
  cd app/feast_repo
  feast apply
  feast feature-views list  # phải thấy 3 views
  ```
- [x] Materialize offline → online store:
  ```bash
  feast materialize-incremental $(date -u +%Y-%m-%dT%H:%M:%S)Z
  ```
  - Xem log: phải thấy rows materialized tới online store
- [x] Test online lookup:
  ```python
  from feast import FeatureStore
  fs = FeatureStore(repo_path="app/feast_repo")
  features = fs.get_online_features(
      features=["user_features:user_age", "item_features:doc_topic"],
      entity_rows=[{"user_id": "u_001"}]
  )
  print(features)  # phải thấy dict với values
  ```
- [x] Đo latency: 100 online lookups, tính P99 (cần < 10 ms)
- [x] Historical features (PIT join):
  ```python
  fs.get_historical_features(
      entity_df=df,  # có column user_id, timestamp
      features=[…]
  )  # trả DataFrame 3 rows × N features
  ```

**Tiêu chí đạt (rubric):**
- ✅ **5 pts:** `feast apply` thành công, `feature-views list` shows all 3 views
- ✅ **5 pts:** `materialize-incremental` thành công, log shows rows materialized to online store
- ✅ **5 pts:** `get_online_features()` trả dict hợp lệ cho user_id=u_001
- ✅ **5 pts:** 100-call online lookup P99 (any value acceptable; P99 < 10 ms = full credit)
  - Đã đo (sau warm-up): P99 = 7.86 ms (< 10 ms) ✔ — full credit
- ✅ **5 pts:** PIT join via `get_historical_features()` trả 3 rows × N features DataFrame
  - Lưu ý: entity_df gốc chỉ trả 2 dòng (u_001 bị bỏ do feature_ts > entity_ts); đã sửa timestamp thành [NOW, NOW-1h, NOW-2h] → 3 dòng

**Ảnh chụp cần lưu:**
- [x] Screenshot: `feast apply` STDOUT (thấy "Created" hoặc "Deployed" 3 feature views)
- [x] Screenshot: `materialize-incremental` log (thấy rows materialized)
- [x] Screenshot: Online lookup result dict
- [x] Screenshot: PIT join DataFrame (shape 3 × N)

**Vibe-coding callout:** Feast YAML + feature view code ít thay đổi. Focus: hiểu entity keys, online/offline store sync, PIT join logic.

**Hướng dẫn nếu `feast apply` lỗi:**
- Xoá `app/feast_repo/registry.db` + `online_store.db` + `data/` folder
- Chạy lại `feast apply`
- Nếu vẫn lỗi: kiểm tra `feature_store.yaml` config (QDRANT_MODE, FEAST_ONLINE_STORE, …)

---

## 3. Kiểm tra Reproducibility — 5 điểm

- [ ] Trên máy **sạch** (chưa chạy lab):
  - Lite: `bash setup-lite.sh && make benchmark`
  - Docker: `bash setup-docker.sh && make benchmark`
  - Phải xanh tất cả → 5 pts rubric
  - Đã chạy trên máy hiện tại: `make verify-lite` ✔, `make test` (41 passed) ✔, `make benchmark` PASS ✔. Còn tuỳ chọn: clone sạch vào /tmp rồi chạy `bash setup-lite.sh && make benchmark`.
- [x] Tất cả `.ipynb` đã chạy xong + outputs lưu

---

## 4. Khối Advanced (NB5–NB8) — 50 điểm (tuỳ chọn)

> **Lưu ý:** NB5–NB8 là khối nâng cao. Chỉ làm nếu:
> - Hoàn thành NB1–NB4 ✓
> - Có thời gian thêm
> - Muốn tìm hiểu sâu hơn
> Rubric cho phép chấm core-only hoặc core + 2 trong 4 advanced.

### NB5 — Filtered Search: Cái Bẫy Recall — 10 điểm

**Mục tiêu:** Post-filter vs pre-filter vs filtered-ANN. Đo recall theo độ chọn lọc.

**Công việc:**
- [ ] Mở `05_filtered_search.ipynb`
- [ ] Xây dựng 3 chiến lược lọc trên cùng corpus + query:
  1. **Post-filter:** ANN top-50 → filter → recall sập ở tight filter
  2. **Pre-filter:** Lọc trước → brute-force cosine → luôn đúng nhưng mất index
  3. **Filtered-ANN:** Filter *trong* Qdrant index → best trade-off
- [ ] Đo recall (groundtruth từ pre-filter) theo selectivity (% docs qua filter):
  - 100% selectivity → recall 1.00
  - 10% selectivity → post-filter recall sập (< 0.70), filtered-ANN = 1.00
- [ ] Over-fetch ladder: tuning `fetch_k` để cứu recall

**Tiêu chí đạt (rubric):**
- ✅ **5 pts:** Bảng recall: post-filter **giảm rõ** khi filter chặt, filtered-ANN **giữ 1.00**
- ✅ **5 pts:** Over-fetch ladder cho thấy `fetch_k` ≈ 50% corpus để cứu recall

**Ảnh chụp cần lưu:**
- [ ] Screenshot: Bảng recall vs selectivity (3 chiến lược)
- [ ] Screenshot: Over-fetch ladder table

---

### NB6 — Agentic Retrieval — 12 điểm

**Mục tiêu:** Retrieval-as-a-tool. Planner tách câu hỏi, reflection loop. Feat Feast features.

**Công việc:**
- [ ] Mở `06_agent_retrieval.ipynb`
- [ ] Xem `app/agent.py`: tool schema + rule-based planner (NO LLM, zero-key)
- [ ] So sánh 3 chiến lược trên **cùng ngân sách 16 docs**:
  1. Single-shot retrieval (1 query → top-16)
  2. Agentic no-filter (planner tách câu hỏi, reflection, không filter)
  3. Agentic with-filter (planner suy ra filter từ câu hỏi)
- [ ] Đo recall vs balance trên golden queries
- [ ] **Agentic phải > single-shot** về cả recall lẫn balance
- [ ] Giải thích vì sao agentic+filter có thể **thấp hơn** agentic no-filter
  - Hint: filter sai → miss documents
- [ ] Chạy `build_context()`: dump features (Feast) + doc_ids cùng lúc

**Tiêu chí đạt (rubric):**
- ✅ **5 pts:** Bảng 3 chiến lược ở cùng budget 16 docs; agentic > single-shot cả recall lẫn balance
- ✅ **4 pts:** Giải thích vì sao agentic(+filter) **thấp hơn** agentic(no filter)
- ✅ **3 pts:** `build_context()` chạy được, output thấy cả feature (Feast) lẫn doc_ids

**Ảnh chụp cần lưu:**
- [ ] Screenshot: Bảng 3 chiến lược (recall, balance)
- [ ] Screenshot: `build_context()` output (feature + doc_ids)

---

### NB7 — Semantic Cache — 12 điểm

**Mục tiêu:** Cache queries tương tự → reuse answers. Tuning ngưỡng, TTL. **Demo rò chéo tenant.**

**Công việc:**
- [ ] Mở `07_semantic_cache.ipynb`
- [ ] Setup SemanticCache (Qdrant collection riêng): score similarity, cache answer
- [ ] Sweep threshold (0.70, 0.75, 0.80, 0.85, 0.90):
  - Cột 1: **Tiết kiệm** (% query hit cache)
  - Cột 2: **Trả lời sai** (% hit false, i.e. answer không khớp query)
  - Bảng phải thấy trade-off: threshold thấp → tiết kiệm nhiều nhưng sai nhiều
- [ ] Chọn threshold hợp lý + giải thích vì sao 0.75 chưa đủ
- [ ] **BONUS bảo mật:** Demo cross-tenant leak:
  - Setup 2 user: u_001, u_002
  - Cache answer cho u_001's query
  - u_002 query tương tự → `namespaced=False` → **leak** (thấy câu trả lời của u_001)
  - `namespaced=True` → **MISS** (không reuse)

**Tiêu chí đạt (rubric):**
- ✅ **5 pts:** Bảng sweep có **cả 2 cột:** tiết kiệm và trả lời sai
- ✅ **4 pts:** Chọn threshold có lý + giải thích tại sao 0.75 chưa đủ
- ✅ **3 pts:** Demo rò chéo tenant: leak khi `namespaced=False`, MISS khi `True`

**Ảnh chụp cần lưu:**
- [ ] Screenshot: Bảng sweep (threshold vs tiết kiệm vs sai)
- [ ] Screenshot: Demo leak (u_002 thấy answer của u_001)

---

### NB8 — Feature Engineering — 12 điểm

**Mục tiêu:** 6 họ feature + target-encoding leakage. PIT vs latest join. On-demand feature view.

**Công việc:**
- [ ] Mở `08_feature_engineering.ipynb`
- [ ] Sinh synthetic event log (search queries, clicks, dwell time)
- [ ] 6 họ feature:
  1. Window aggregation (searches_1h, searches_7d)
  2. Ratios (query_len / user_avg_len)
  3. Lag & delta (prev_query_len, query_len_delta)
  4. Recency (seconds_since_last)
  5. Categorical encoding (frequency, target encoding) ← **LEAKAGE ở đây**
  6. Embedding features (user/item vectors)
- [ ] **Target-encoding leakage experiment:**
  - Encode target (e.g., click) vào feature → AUC inflated
  - Bảng: in-fold vs out-fold AUC
  - **Gap > 0.30** trên `session_id`, in-fold ≈ 0 (hint: PIT join bug)
- [ ] **PIT vs latest join:**
  - PIT join (point-in-time): dùng feature tại time của event → safe
  - Latest join: dùng feature mới nhất lúc training → rò rỉ (leakage)
  - Báo cáo: % dòng rò rỉ + AUC chênh lệch
- [ ] **On-demand feature view** (`app/feast_repo_ondemand/`):
  - Compute feature từ server-time (amount) + stored feature (avg_amount)
  - Cùng user, 2 amount khác → amount_vs_avg khác

**Tiêu chí đạt (rubric):**
- ✅ **4 pts:** Bảng leakage: `target-naive` gap > 0.30 trên `session_id`, in-fold ≈ 0
- ✅ **4 pts:** PIT vs latest join: báo cáo % dòng rò + AUC chênh lệch
- ✅ **4 pts:** On-demand feature view: cùng user, 2 amount → 2 amount_vs_avg khác

**Ảnh chụp cần lưu:**
- [ ] Screenshot: Bảng leakage (in-fold vs out-fold AUC)
- [ ] Screenshot: PIT vs latest join comparison
- [ ] Screenshot: On-demand feature view output (2 rows, khác amount_vs_avg)

---

## 5. Generate Advanced Data + Run All Notebooks

Nếu làm NB5–NB8:

- [ ] Chạy: `make gen-advanced`
  - Tạo `data/agent_queries.jsonl` (NB6 compound queries)
  - Tạo `data/spend.parquet` (NB8 spend data)

- [ ] Chạy ALL notebooks headless:
  ```bash
  make notebooks
  ```
  - Chạy lần lượt 01.ipynb → 08.ipynb
  - Phải tất cả PASS
  - Lưu outputs vào .ipynb files

---

## 6. Submission

### 6.1 Tạo GitHub repo

- [ ] Nếu chưa có: `git init -b main`
- [ ] Add remote (tạo repo trên github.com trước):
  ```bash
  git remote add origin https://github.com/<username>/<repo-name>.git
  ```
- [x] Set repo **PUBLIC** trên GitHub

### 6.2 Thêm screenshots

- [x] Tạo folder `submission/screenshots/` (đã có template)
- [x] Thêm ảnh chụp từng notebook:
  - **NB1:** corpus loaded + 1000 indexed + paraphrase query results
  - **NB2:** Precision@10 bảng (hybrid > kw/sem) + slice bảng
  - **NB3:** API response sample + P50/P95/P99 bảng
  - **NB4:** `feast apply` STDOUT + `materialize-incremental` log + online lookup + PIT join DF
  - **NB5–8:** (nếu làm) tương ứng
- [x] Đặt tên file theo nội dung: `nb1_indexed.png`, `nb1_top5.png`, `nb1_paraphrase.png`, `nb2_precision.png`, `nb2_slice.png`, `nb3_response.png`, `nb3_latency.png`, `nb3_pass.png`, `nb4_parquet.png`, `nb4_apply.png`, `nb4_materialize.png`, `nb4_online_lookup.png`, `nb4_latency.png`, `nb4_pit_join.png` (14 ảnh; repo không bắt buộc quy ước tên)

### 6.3 Điền REFLECTION.md

- [x] Mở `submission/REFLECTION.md`
- [x] Điền:
  - ✔ Họ tên
  - ✔ Cohort (A20-K4)
  - ✔ Path đã chạy (lite)
  - ✔ Trả lời câu hỏi (≤ 200 chữ, hiện 149 chữ):
    > Trên 50 golden queries, mode nào thắng ở loại query nào (exact/paraphrase/mixed)? Khi nào **không** dùng hybrid?

**Ví dụ trả lời:**
```
Hybrid thắng trên mixed queries (~75% P@10). BM25 thắng exact (query literal).
Vector thắng paraphrase (diễn đạt lại). 

Không dùng hybrid khi: 1) corpus rất nhỏ (<1K doc) → overhead, 2) latency < 5ms 
budget (hybrid slower), 3) corpus toàn numerical/structured (BM25 enough).
```

### 6.4 Push GitHub

- [x] Stage files:
  ```bash
  git add notebooks/*.ipynb submission/REFLECTION.md submission/screenshots/
  ```
- [x] Commit:
  ```bash
  git commit -m "Lab 19 submission — <Họ Tên>"
  ```
- [x] Push:
  ```bash
  git push -u origin main
  ```

### 6.5 Submit VinUni LMS

- [x] Copy public repo URL: `https://github.com/<username>/<repo-name>`
- [x] Paste vào LMS submission box cho Day 19
- [x] **Xác nhận repo PUBLIC** (kiểm tra bằng private window — nếu private → 0 điểm)

---

## 7. Bonus Challenge (Tùy chọn, +20 điểm)

Nếu bạn muốn thêm điểm + portfolio piece:

- [ ] Đọc `BONUS-CHALLENGE.md` (15 phút)
- [ ] Tạo folder `bonus/` trong repo
- [ ] Làm 3 deliverable:
  1. **`bonus/ARCHITECTURE.md`** (~600 từ, diagram, 3 trade-off decisions)
  2. **`bonus/agent.py`** (~150 dòng, `HybridMemoryAgent.remember()` + `.recall()`)
  3. **`bonus/demo.py`** (gọi agent, in 5 query outputs)
- [ ] Push + submit URL LMS thêm 1 lần

**Rubric bonus:**
- 3 pts: ARCHITECTURE.md ≥ 600 words + diagram
- 6 pts: 3 architecture decisions với explicit trade-off
- 2 pts: ≥ 1 decision Vietnamese-context aware
- 2 pts: Rejected alternative named + reason
- 4 pts: agent.py chạy được
- 3 pts: demo.py exit 0, 5 outputs printed

---

## 8. Troubleshooting

### Setup

| **Triệu chứng** | **Fix** |
|---|---|
| `python3: command not found` | Install Python 3.10+ từ https://www.python.org |
| `bash setup-lite.sh` error | Xoá `.venv`, chạy lại `bash setup-lite.sh` |
| Port 8000 in use (FastAPI) | `lsof -ti:8000 \| xargs kill -9` hoặc đổi `--port 8001` |
| Port 6333/6379/5432 in use (Docker) | `docker compose down && docker compose up -d` |

### Data

| **Triệu chứng** | **Fix** |
|---|---|
| NB1 báo `expected 1000, got X` | Chạy `make seed` |
| Tests skip (không fail) | Tests phụ thuộc corpus — chạy `make seed` trước |
| `data/corpus_vn.jsonl` missing | Chạy `make seed` |

### NB2 (Hybrid Search)

| **Triệu chứng** | **Fix** |
|---|---|
| Hybrid không thắng keyword/semantic | **Rank 1-based không?** Kiểm tra RRF formula: `1/(k + rank)` rank = 1..top_k |
| | Embedding model yếu? Đổi `EMBEDDING_BACKEND=bge-m3`, chạy lại NB1 + NB2 |
| | BM25 tokenization? Phải whitespace, không dùng NLP heavy lib |

### NB3 (API Latency)

| **Triệu chứng** | **Fix** |
|---|---|
| P99 > 50 ms | Bình thường ở cold start — chạy warmup 10 query, rồi đo lại |
| | Hoặc: `make api &` ở terminal riêng, đợi 10s, rồi notebook query |
| `/search` timeout | API server chưa up — check `http://localhost:8000/healthz` |

### NB4 (Feast)

| **Triệu chứng** | **Fix** |
|---|---|
| `feast apply` lỗi | Xoá `app/feast_repo/registry.db`, `online_store.db`, `data/` folder, chạy lại |
| Materialize fail | Check `.env`: FEAST_ONLINE_STORE / FEAST_OFFLINE_STORE config đúng? |
| Online lookup slow | P99 > 10 ms — bình thường cho SQLite. Docker path (Redis) sẽ nhanh hơn |

### Docker Path

| **Triệu chứng** | **Fix** |
|---|---|
| Qdrant timeout | Đợi 60s sau `docker compose up -d`, pull image lần đầu ~200 MB |
| `port 6333 already allocated` | `docker ps` xem container nào occupying, `docker kill <id>` |
| Container exit | Check `docker logs qdrant` / `redis` / `postgres` để xem error |

### Advanced Notebooks (NB5–8)

| **Triệu chứng** | **Fix** |
|---|---|
| NB6 agent.py import error | Chạy `make seed` trước (agent queries phụ thuộc data) |
| NB8 on-demand feature view error | Check Feast Python version — 3.14 cần `dill ≥ 0.4` override (setup-lite.sh xử lý auto) |

---

## 9. Checklist Cuối Cùng

Trước khi submit:

- [ ] `.ipynb` files có outputs lưu (không phải "Cell not run")
- [x] `submission/REFLECTION.md` điền đầy đủ
- [x] Ảnh chụp cho mỗi notebook ✓
- [x] Repo PUBLIC trên GitHub ✓
- [ ] `make notebooks` chạy tất cả xanh (hoặc core-only tuỳ requirement)
- [x] Không có sensitive data (API keys, credentials) trong commit
- [x] Url repo dán vào LMS ✓
- [ ] (Nếu làm bonus) `bonus/` folder có cả 3 file ✓

---

## 10. Điểm Số Tóm Tắt

| **Khối** | **Tiêu chí** | **Điểm** |
|---|---|---|
| Core (bắt buộc) | NB1–NB4 + reproducibility | **100** |
| Advanced (tuỳ) | NB5–NB8 (10+12+12+12 pts) | 50 |
| Bonus (tuỳ) | Architecture + code | 20 |
| **Tổng** | | **170** |

Rubric của giảng viên cho phép:
- **Core-only:** 100 pts (safe)
- **Core + 2 advanced:** 100 + 24 = 124 pts (recommended)
- **Core + all advanced:** 100 + 50 = 150 pts
- **+ Bonus:** +20 pts (khuyến khích)

---

## 11. Tips Cuối

1. **Đọc VIBE-CODING.md** (5 phút) trước làm NB1 — giúp khi delegate code cho AI
2. **Warm-up trước đo latency** — NB3 P99 < 50 ms dễ với warmup
3. **Review RRF rank công thức** NB2 — rank 1-based, không 0-based, 1 dòng sai = fail
4. **Tách notebook nếu cần** — làm NB1–4 core trước, advanced sau
5. **Docker vs lite:** Lite đủ cho concept, Docker tốt hơn cho bài học embedding tiếng Việt
6. **Đừng edit feature_views.py thêm view** — rubric assert đúng 3 views NB4
7. **Commit thường xuyên** — dễ rollback nếu cần

---

**Chúc bạn hoàn thành lab!** 🎉
