# Lab 19 — GraphRAG vs Flat RAG

## Trạng thái bàn giao

Pipeline đã chạy live trên 5.000 dòng HackerNoon: 2.675 dòng có text, còn 2.105 bài sau exact dedup; tạo 2.105 chunks, ưu tiên toàn bộ 51 source rows được golden dataset dẫn chiếu. Coreference và NER+RE chạy không có batch error, sinh 53 triples, 64 nodes và 53 edges trong Neo4j.

Notebook dùng `data/` và `outputs/`, tự nhận secrets từ Colab hoặc `.env`, hỗ trợ `NEO4J_USERNAME`/`NEO4J_USER`, và có cell điều phối `5.2 — Run pipeline and export artifacts`. Model Groq live dùng `openai/gpt-oss-120b`; judge dùng OpenAI `gpt-4o-mini`.

## Benchmark 50 câu

| Metric trung bình | Flat RAG | GraphRAG |
|---|---:|---:|
| Comprehensiveness | 3.460 | 3.680 |
| Faithfulness | 3.660 | 3.940 |
| Multi-hop reasoning | 3.460 | 3.680 |
| Latency (s) | 2.968 | 6.469 |
| Token usage | 962.56 | 1170.56 |

GraphRAG cải thiện cả ba điểm chất lượng, đặc biệt ở nhóm multi-hop, nhưng latency cao hơn khoảng 2,18 lần và token usage cao hơn khoảng 22%. Theo nhóm, GraphRAG đạt 5.00/5.00/5.00 cho factoid, 3.59/3.91/3.59 cho cross-doc và 3.48/3.74/3.48 cho multi-hop.

## Kiểm tra integrity

- Neo4j: 64 nodes, 53 edges.
- Edges thiếu `source_chunk_id`, `published_date` hoặc `evidence`: **0**.
- Top degree: ServiceNow 9, Microsoft 8, NVIDIA 4; không có super-node >100 nên policy không phải cắt cạnh trong run này.
- Evaluation: 50/50 rows, không có answer/judge score null.

## Artifact

- `data/graphrag_golden_50_first5000_detailed.csv`: 50 câu hỏi có reference answer/evidence.
- `outputs/graphrag_eval_results.csv`: 50 kết quả chi tiết và rationale.
- `outputs/graphrag_vs_flatrag_summary.csv`: 15 dòng tổng hợp theo nhóm/metric.
- `reports/technical_defense.md`, `reports/failure_analysis.md`, `reports/reflection_LeAnhTien.md`.

## Kết luận

Flat RAG phù hợp cho factoid đơn giản khi ưu tiên chi phí và latency. Hybrid GraphRAG đáng dùng cho câu hỏi truy vết quan hệ, timeline và cross-document; cần giữ giới hạn BFS, cache embedding và batch LLM để kiểm soát chi phí production.
