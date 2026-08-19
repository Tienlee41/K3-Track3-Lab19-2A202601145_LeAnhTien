# Báo cáo thực hành — GraphRAG vs Flat RAG

**Học viên:** Lê Anh Tiến

**Khóa:** AICB-K34 · Track 3: GraphRAG

**Ngày hoàn thiện:** 20/08/2026

## Phần 1 — Thuyết minh kỹ thuật và phân tích ca lỗi

### 1. Coreference Resolution

Source row `1811` nhắc Clarke, Dell và Phillips rồi dùng `it`, tạo nguy cơ gán sai antecedent giữa hai công ty. Pipeline chỉ rewrite khi antecedent rõ trong cùng chunk; nếu mơ hồ thì giữ nguyên và log `unresolved_mentions`. False coreference có thể biến thành false employment/layoff edge, sau đó bị BFS khuếch đại sang answer.

### 2. Entity Resolution

- Vector threshold: `0.90`.
- Lexical threshold: `SequenceMatcher >= 0.72` sau khi bỏ hậu tố pháp nhân.
- Audit: 128 rows.
- Candidate gần nhất: `Dell Technologies Inc.` / `Dell Technologies`, cosine `0.8266`, quyết định `REJECT_GUARD`, reason `COSINE_BELOW_THRESHOLD`.

Live audit không có cặp `>0.85` bị lexical guard chặn; báo cáo giữ nguyên sự thật này thay vì gắn sai nguyên nhân. Cấu hình bảo thủ giúp tránh false merge nhưng gây false negative cho Dell. Biện pháp phù hợp là reviewed alias map.

### 3. Graph integrity và Super-node

Kiểm tra trực tiếp Neo4j:

| Kiểm tra | Kết quả |
|---|---:|
| Nodes | 64 |
| Edges | 53 |
| Edge thiếu source/date/evidence | 0 |
| Top degree 1 | ServiceNow — 9 |
| Top degree 2 | Microsoft — 8 |
| Top degree 3 | NVIDIA — 4 |

Live sample chưa có node degree >100. Notebook bổ sung boundary test deterministic cho policy: degree 100 không cap, degree 101 cap ở 50. Traversal còn bị giới hạn bởi `GLOBAL_EDGE_CAP=250` và context 14.000 ký tự. Recency cap giảm context explosion nhưng có thể loại sự kiện lịch sử; production cần time-window theo query.

### 4. Benchmark

| Metric | Flat RAG | GraphRAG | Delta |
|---|---:|---:|---:|
| Comprehensiveness | 3.460 | 3.680 | +0.220 |
| Faithfulness | 3.660 | 3.940 | +0.280 |
| Multi-hop reasoning | 3.460 | 3.680 | +0.220 |
| Latency | 2.968 s | 6.469 s | +3.501 s |
| Token usage | 962.56 | 1170.56 | +208.00 |

GraphRAG tăng chất lượng nhưng latency khoảng `2.18×` và token tăng `21.6%`.

**Flat RAG thất bại — `G5000-06`:** Flat đạt 1/1/1; Graph đạt 5/5/5. Graph nối đúng ServiceNow–NVIDIA tháng 5, Now Assist tháng 6 và AI Lighthouse cuối tháng 7.

**GraphRAG thất bại — `G5000-26`:** Flat đạt 5/5/5; Graph đạt 1/1/1. Vector retrieval lấy đúng Cohere và conversational agents, còn graph context thiếu provider/capability. Cần coverage check, seed diagnostics và vector fallback.

### 5. Trade-offs, Agent Control và Scale

Flat RAG thích hợp factoid và câu hỏi ưu tiên latency. GraphRAG thích hợp relational/timeline/cross-document nhưng có extraction/indexing overhead. Các đề xuất all-pairs entity similarity và BFS không giới hạn bị từ chối vì `O(N²)`, OOM và context explosion.

Scale 350MB cần async extraction queue, idempotent chunk hash, persistent ANN/HNSW, type blocking, UNWIND partition, embedding cache, query router và temporal filter. Chi tiết 10 câu bảo vệ nằm trong `technical_defense.md`; root-cause trace nằm trong `failure_analysis.md`.

## Phần 2 — Reflection và Action Plan

### Mapping bài giảng vào code

| Khái niệm | Code |
|---|---|
| Conservative coreference | `resolve_coref_batch()`, `run_coref()` |
| Schema allowlist | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` |
| Bulk ingestion | `bulk_insert_nodes()`, `bulk_insert_edges()` |
| Entity Resolution | `build_resolution_map()`, `UF`, `merge_guard()` |
| Flat/Hybrid retrieval | `retrieve_flat_context()`, `retrieve_graph_context()` |
| Super-node cap | `effective_edge_limit()`, `test_supernode_policy()` |
| LLM Judge | `judge_answer()`, `run_evaluation()` |
| Submission validation | `validate_submission_artifacts()` |

### Debugging và bài học

Vấn đề khó nhất là đồng bộ đường dẫn Colab/local, secrets và trạng thái artifact. Cách xử lý là resolve path từ repo root, secret fallback có kiểm soát, checkpoint evaluation và tách full regeneration khỏi validation. Bài học: CSV tồn tại chưa đủ; submission phải có assertions và execution evidence tái lập được.

### Action Plan

Đồ án tiếp theo là trợ lý engineering-news nội bộ. Nodes gồm Company, Person, Product, Technology, Document, Event; relations gồm ownership, partnership, investment, employment, usage và event participation. Flat RAG xử lý summary/factoid; GraphRAG chỉ được route cho multi-hop/timeline.

Entity resolution dùng reviewed aliases + HNSW candidates + lexical/domain guard + human review vùng xám. Super-node được xử lý bằng relation/time filter, degree cap, global cap và community partition. Chi tiết reflection đầy đủ nằm trong `reflection_LeAnhTien.md`.

## Artifact bàn giao

- `Day19_GraphRAG_vs_FlatRAG_Production_Lab_Guide.ipynb`
- `data/golden_dataset.csv` — 50 câu, đủ reference answer/evidence.
- `outputs/graphrag_eval_results.csv` — 50 kết quả có scores/rationales/latency/tokens.
- `outputs/graphrag_vs_flatrag_summary.csv` — 15 aggregate rows.
- `outputs/entity_resolution_audit.csv` — 128 audit rows.
- `reports/technical_defense.md`
- `reports/failure_analysis.md`
- `reports/reflection_LeAnhTien.md`
