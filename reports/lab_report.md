# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Nguyễn Minh Đức  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 20/08/2026  

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** Các tin HackerNoon thường viết dạng press-release ngắn (`description`), ví dụ một bài nhắc `onsemi` rồi dùng đại từ `"the company"` / `"it"` khi chuyển sang `Sineng Electric` trong cùng đoạn. Antecedent không luôn rõ trong một chunk ngắn ~220 từ.
- **Hiện tượng:** Cơ chế conservative coreference (chỉ resolve khi antecedent rõ trong cùng chunk) thường **giữ nguyên** đại từ khi ambiguous. Trong lab, nếu resolve sai (ví dụ gán `"the company"` về `onsemi` thay vì `Sineng Electric`) sẽ tạo false subject cho quan hệ `INTEGRATES` / `PARTNERS_WITH`.
- **Hậu quả đối với Graph:** False coreference → false edge (gán nhầm quan hệ M&A / partnership / technology cho công ty sai). Vì schema allowlist vẫn chấp nhận edge nếu type/relation hợp lệ, lỗi này **không bị chặn ở ingestion** mà chỉ lộ ra khi audit / đánh giá multi-hop.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90` (tham số mặc định trong `build_resolution_map(..., threshold=0.90, top_k=5)`).
- **Cặp thực thể bị Guard chặn (thiết kế / kỳ vọng từ lexical guard):** `Apple` vs `Apple Music`, hoặc `Microsoft` vs `Microsoft Azure` — vector embedding thường rất gần (> 0.85) vì cùng brand token, nhưng Lexical Guard từ chối merge khi một tên là **substring/product extension** của tên kia hoặc khác suffix pháp lý / product line.
- **Lý do chặn:** Gộp nhầm company ↔ product/service sẽ “gom” mọi cạnh của product vào một node công ty, làm Super-node phình to và phá vỡ semantics của quan hệ `DEVELOPS` / `USES`. Trong lần chạy lab này, `raw_triples_df` rỗng → audit table không có cặp `REJECT_GUARD` thực tế; cơ chế vẫn được giữ theo thiết kế notebook (vector ANN + lexical guard + Union-Find).

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Kết quả graph_checks thực tế (cell 2.4):**
  - `{'nodes': 0, 'edges': 0, 'invalid_provenance_edges': 0}`
  - `Nodes to insert: 0 | Edges to insert: 0` (cell 2.3)
  - Top-degree table: **Empty DataFrame** (không có entity sau extraction)

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | *(không có — KG rỗng)* | — | 0 |
| 2 | *(không có — KG rỗng)* | — | 0 |
| 3 | *(không có — KG rỗng)* | — | 0 |

- **Nguyên nhân gốc:** Bước NER+RE (cell 2.1) không sinh được triples hợp lệ sau allowlist (`ALLOWED_NODE_TYPES` / `ALLOWED_RELATIONS`), nên Entity Resolution / Neo4j insert không có dữ liệu. `invalid_provenance_edges = 0` vẫn đúng vì không có edge nào.
- **Ưu điểm & Rủi ro của Temporal Mitigation (policy trong code: degree > 100 → cap 50 edge mới nhất):**
  - *Ưu điểm:* Giảm bùng nổ context khi gặp hub entity (NVIDIA, Microsoft, OpenAI…); ưu tiên quan hệ mới giúp câu hỏi “current events” ổn định hơn về latency/token.
  - *Rủi ro:* Câu hỏi lịch sử / multi-hop dựa trên cạnh cũ có thể bị cắt; nếu seed chỉ match vào Super-node thì GraphRAG mất đường đi quan trọng dù edge vẫn tồn tại trong Neo4j. Trong run này `graph_supernode_events = 0` trên mọi câu vì không có đồ thị để traverse.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

Số liệu lấy từ `outputs/graphrag_vs_flatrag_summary.csv` và trung bình toàn bộ 8 câu trong `outputs/graphrag_eval_results.csv` (G01–G08).

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|-------------------|----------|----------|--------------------------|-------------------|
| **Comprehensiveness (1–5)** | 1.0 | 1.0 | 0.0 | Cả hai không trả lời được gold; Judge chấm thấp nhất. |
| **Faithfulness (1–5)** | 1.0 | 1.0 | 0.0 | Câu trả lời chủ yếu là “insufficient evidence”; không bịa fact nhưng cũng không đúng gold. |
| **Multi-hop Reasoning (1–5)** | 1.0 | 1.0 | 0.0 | Không có chuỗi suy luận đa hop vì thiếu context đúng / thiếu graph edges. |
| **Latency trung bình (s)** | 4.226 | 8.787 | +4.561 | GraphRAG chậm hơn rõ (seed + traversal + hybrid context). |
| **Token usage trung bình** | 1911.0 | 1628.75 | −282.25 | GraphRAG dùng ít token hơn trong sample này (graph context gần như trống). |

Theo nhóm câu hỏi (từ summary CSV):
- **factoid:** Flat latency ~3.88s vs Graph ~9.70s; token Flat 1930 vs Graph 1469.5
- **multi-hop:** Flat ~4.17s vs Graph ~5.20s
- **cross-doc:** Flat ~4.97s vs Graph ~10.55s (chênh latency lớn nhất)

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (và GraphRAG cũng không cứu được — vì KG rỗng):**
   - *Question ID & Câu hỏi:* `G01` — *Which company's cloud unit partnered with Hugging Face, the maker of a ChatGPT rival?*
   - *Gold:* Amazon's cloud unit (AWS) partnered with Hugging Face.
   - *Tại sao Flat RAG thất bại?* Top-k vector chunks không chứa bài về AWS–Hugging Face (hoặc bị noise từ các bài “ChatGPT rival / cloud partner” khác). Generator trả lời: *evidence is insufficient*.
   - *GraphRAG xử lý thế nào?* Seed/traversal không có cạnh liên quan vì **0 edges trong Neo4j**; Hybrid GraphRAG suy giảm về gần Flat RAG + graph context trống → vẫn insufficient. Đây là failure mode **extraction → empty KG**, không phải failure của BFS policy.

2. **Ca lỗi GraphRAG thất bại (cả hai cùng sai) — multi-hop / cross-doc:**
   - *Question ID & Câu hỏi:* `G03` — ServiceNow × NVIDIA × AI Lighthouse / Accenture; và `G08` — technology gắn NVIDIA across multiple chunks.
   - *Nguyên nhân:* (1) Flat retrieval không ghép được multi-document evidence; (2) Graph thiếu edge `PARTNERS_WITH` / provenance giữa ServiceNow–NVIDIA–Accenture do extraction trống; (3) Generator (Groq `qwen/qwen3.6-27b`) đôi khi phát ra thinking tags dài, làm context/token kém hiệu quả và Judge khó chấm cao.
   - *Đề xuất khắc phục:*
     - Re-run extraction với model ổn định + log `extraction_errors_df`.
     - Nới batch / giảm filter tạm để audit recall trước khi siết allowlist.
     - Xây golden queries **sau khi** verify entity/edge coverage trong Neo4j (`MATCH` theo tên công ty).
     - Strip `<think>` trước khi judge / export answer.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:** Trong run này quality hai bên ngang nhau (đều 1.0) nên **không chứng minh được lợi thế reasoning của GraphRAG**. Trade-off thực tế quan sát được: GraphRAG **latency cao hơn ~2×** (8.8s vs 4.2s) nhưng **token generation thấp hơn một chút**. Chi phí “ẩn” của GraphRAG nằm ở indexing pipeline (coref + NER/RE + entity resolution + Neo4j), không chỉ ở query-time.
- **Quyết định từ chối AI Coding Agent:** Từ chối đề xuất chạy pairwise cosine $O(N^2)$ cho near-dedup / entity resolution trên toàn bộ dump (~300MB, hàng trăm nghìn bài). Thay bằng exact SHA-1 dedup + (thiết kế) MinHash/LSH hoặc ANN + Lexical Guard. Cũng từ chối hard-code API keys vào notebook; giữ secrets trong `.env`.
- **Giải pháp scale 350MB:** Bottleneck đầu tiên là **LLM extraction throughput / rate-limit**, sau đó embedding + Neo4j write. Hướng xử lý: async batch queue, giới hạn `EXTRACTION_MAX_CHUNKS` theo ưu tiên relevance, HNSW/FAISS cho entity blocking, community partitioning / Global Search cho câu hỏi thematic, và provenance bắt buộc trên mọi edge để debug.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `run_coref()`, `resolve_coref_batch()` | An toàn hơn aggressive resolve; nhưng nếu skip quá nhiều thì RE thiếu subject rõ. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Bảo vệ schema; quá chặt + model yếu → 0 triples (failure chính của lab run). |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` ổn; đã thêm guard khi DataFrame rỗng và reconnect khi Aura `SessionExpired`. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF`, Lexical Guard | Threshold 0.90 hợp lý; không chạy được trên data thật vì không có mentions. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()` | Policy degree>100 → top 50 theo `published_date`; chưa kích hoạt vì graph rỗng. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `run_evaluation()` | Judge qua OpenRouter (`openai/gpt-4o-mini`); điểm đều 1 phản ánh retrieval/KG failure hơn là bias của metric. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:**
  1) Path Colab `/content/...` trên Windows local.  
  2) HackerNoon schema dùng `description` thay vì `text`.  
  3) Neo4j Aura `ServiceUnavailable` / `ConnectionResetError 10054`.  
  4) Groq model `llama-3.3-70b-versatile` 404 → phải chuyển `qwen/qwen3.6-27b`.  
  5) Pipeline extraction → **0 triples** khiến toàn bộ GraphRAG “chạy được” nhưng **không có tri thức**.
- **Cách bạn đã xử lý thành công:** Chuẩn hóa path local (`data/`, `outputs/`), thêm `"description"` vào loader, harden `run_cypher` (retry/reconnect), fallback Groq model, và guard empty DataFrame ở entity resolution / bulk insert để không crash khi KG rỗng. Bài học: **green cell ≠ correct graph**; phải assert `nodes/edges > 0` trước evaluation.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** Hybrid GraphRAG cho tin công nghệ / quan hệ doanh nghiệp (M&A, partnership, investment) — mở rộng pipeline Lab 19.
- **Đặc thù bài toán & Lý do chọn giải pháp:** Câu hỏi multi-hop / cross-doc (A đầu tư B, B phát triển C) cần cạnh quan hệ có provenance; Flat RAG dễ phân mảnh. Hybrid (subgraph + vector chunks) phù hợp hơn pure Flat hoặc pure Graph.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `Company`, `Person`, `Technology`, `Product`, `Event`
  - Relations: `INVESTED_IN`, `PARTNERS_WITH`, `ACQUIRED`, `FOUNDED`, `DEVELOPS`, `USES`, `COMPETES_WITH`
- **Chiến lược xử lý Super-node & Entity Resolution:** Cap cạnh theo thời gian + type filter; entity resolution = manual alias + ANN (threshold cao) + lexical guard (ticker/suffix/product-of-company); bắt buộc audit `REJECT_GUARD` trước khi productionize.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 4 | Hiểu rõ pipeline & failure modes; chất lượng KG lần chạy chưa đạt. |
| Khả năng kiểm soát AI Coding Agent | 4 | Từ chối O(N²), hard-code secrets; chấp nhận path/local/Neo4j hardening cần thiết. |
| Chất lượng đồ thị tri thức xây dựng | 2 | KG rỗng (0 nodes/edges) — đây là điểm yếu chính cần cải thiện bằng re-extraction. |
| Khả năng phân tích và debug hệ thống | 4 | Trace được lỗi từ path → schema → Aura → model 404 → empty triples → eval score 1.0. |
