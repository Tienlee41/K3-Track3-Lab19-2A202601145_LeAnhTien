# Reflection and Action Plan

**Học viên:** Lê Anh Tiến

**Lab:** Production-Grade GraphRAG vs Flat RAG

## 1. Mapping bài giảng vào code

| Lecture concept | Module | Notebook implementation | Quan sát thực nghiệm |
|---|---|---|---|
| Streaming, exact dedup, chunk overlap | Module 1 | `load_news()`, `standardize_news()`, `build_chunks()` | 5.000 raw rows → 2.675 usable rows → 2.105 articles/chunks sau exact dedup trong live run. |
| Conservative coreference | Module 1 | `resolve_coref_batch()`, `run_coref()` | Ambiguous mention được giữ nguyên và log, ưu tiên precision hơn recall. |
| Strict NER/RE schema | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `extract_triples_batch()` | 53 triples hợp lệ được giữ lại; output ngoài allowlist bị loại. |
| Bulk Cypher ingestion | Module 2 | `setup_graph_schema()`, `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` batch 1.000; live Neo4j có 64 nodes, 53 edges. |
| Entity Resolution + Union-Find | Module 3 | `build_resolution_map()`, `UF`, `merge_guard()` | 128 audit decisions; cấu hình 0.90 rất bảo thủ và bỏ lỡ alias Dell. |
| Flat RAG FAISS | Module 4 | `build_flat_index()`, `retrieve_flat_context()`, `answer_flat_rag()` | Nhanh hơn; mạnh ở factoid và khi graph coverage thiếu. |
| Hybrid GraphRAG | Module 4 | `match_seeds()`, `retrieve_graph_context()`, `textualize()`, `answer_graph_rag()` | Mạnh ở `G5000-06`, nhưng phụ thuộc extraction và seed coverage. |
| Super-node mitigation | Module 4 | `effective_edge_limit()`, `test_supernode_policy()`, `GLOBAL_EDGE_CAP` | Live graph chưa có super-node; boundary test xác minh degree 101 → cap 50. |
| LLM-as-a-Judge | Module 5 | `judge_answer()`, `run_evaluation()`, `comparison_table()` | 50/50 câu có score, rationale, latency và token. |

## 2. Debugging và bài học

Lỗi khó nhất là giữ cho đường dẫn local/Colab, secret loading và artifact resume nhất quán. Notebook ban đầu có kết quả CSV nhưng trạng thái execution của nhiều cell không được lưu vào file `.ipynb`, khiến người chấm không thể phân biệt kết quả live với số liệu được mô tả lại.

Giải pháp là:

- Resolve đường dẫn từ `Path.cwd()` và chỉ fallback `/content` trong Colab.
- Đọc secret từ Colab userdata hoặc `.env`, không hard-code.
- Tách `run_lab_pipeline()` khỏi `validate_submission_artifacts()`.
- Mặc định Run All xác minh golden/evaluation/audit/Neo4j đã tồn tại; chỉ tái sinh LLM artifacts khi đặt `RUN_FULL_PIPELINE=1`.
- Bổ sung assertion cho score range, missing values, ID coverage, provenance và super-node boundary.

Bài học chính: artifact tồn tại chưa đủ; một pipeline production cần checkpoint, validation có thể lặp lại và bằng chứng failure rõ ràng.

## 3. Action Plan cho đồ án thực tế

**Tên đề xuất:** Internal Engineering & Technology Intelligence Assistant.

**Khi nào dùng GraphRAG:**

- Flat/Hybrid RAG cho summary, factoid và tra cứu theo tài liệu.
- GraphRAG cho câu hỏi ownership, partnership, investment, employment, product lineage và timeline xuyên tài liệu.
- Query router chọn đường đi dựa trên intent; không dùng GraphRAG cho mọi câu hỏi.

**Node dự kiến:**

- `Company`, `Person`, `Product`, `Technology`, `Document`, `Event`.

**Relation dự kiến:**

- `ACQUIRED`, `DEVELOPED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`, `USES`, `LEADS`, `MENTIONED_IN`, `PARTICIPATED_IN`.

**Entity Resolution:**

- Manual alias cho ticker/tên pháp nhân quan trọng.
- Blocking theo entity type và token signature.
- HNSW top-k candidate generation, sau đó lexical/domain guard.
- Human-review queue cho cặp ở vùng xám; không giảm threshold toàn cục để sửa một alias.

**Super-node:**

- Degree-aware cap và global edge cap.
- Filter theo time window/relation intent trước khi sort theo recency.
- Với node cực lớn, partition theo community và sinh community report có provenance.

**Đo lường:**

- Theo dõi answer quality, retrieval coverage, seed-match rate, invalid provenance, latency và token theo từng question group.
- Duy trì regression set gồm cả ca GraphRAG thắng và GraphRAG thua như `G5000-06`/`G5000-26`.

## 4. Tự đánh giá

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Hiểu GraphRAG | 4 | Hiểu rõ giá trị multi-hop và dependency vào graph coverage. |
| Kiểm soát AI Coding Agent | 4 | Giữ schema guard, provenance, bounded traversal; từ chối all-pairs/BFS vô hạn. |
| Chất lượng Knowledge Graph | 4 | Provenance đầy đủ; cần tăng extraction coverage và reviewed aliases. |
| Phân tích/debug | 4 | Có benchmark và root-cause cases; cần thêm telemetry seed/edge coverage. |
