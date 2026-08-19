# Lab 19 — GraphRAG vs Flat RAG

## Trạng thái bàn giao

Repo đã được hoàn thiện theo pipeline 5 module: streaming HackerNoon tối đa 5.000 dòng, chuẩn hóa/dedup/chunking, coreference bảo thủ, NER+RE theo allowlist, entity resolution có lexical guard, Neo4j bulk insert bằng `UNWIND`, Flat RAG FAISS và Hybrid GraphRAG BFS có giới hạn super-node.

Notebook dùng đường dẫn tương đối `data/` và `outputs/`, tự nhận `HF_TOKEN` từ Colab Secrets hoặc environment, đồng thời hỗ trợ alias `NEO4J_USERNAME`/`NEO4J_USER`. Không có secret nào được ghi vào notebook.

## Artifact

- `data/graphrag_golden_50_first5000_detailed.csv`: 50 câu hỏi thuộc factoid, multi-hop và cross-doc, có reference answer/evidence và metadata đánh giá.
- `outputs/graphrag_eval_results.csv`: schema checkpoint cho từng câu hỏi.
- `outputs/graphrag_vs_flatrag_summary.csv`: bảng so sánh metric.
- `reports/technical_defense.md`: 10 câu thuyết minh kỹ thuật.
- `reports/failure_analysis.md`: phân tích failure modes và mitigation.
- `reports/reflection_LeAnhTien.md`: mapping bài giảng và action plan.

## Cách chạy

Chạy notebook từ đầu tới cell `5.2 — Run pipeline and export artifacts`. Sau khi Neo4j và Groq khả dụng, bỏ comment các dòng `summary = run_lab_pipeline()`, `eval_results_df = run_evaluation(golden_df)` và hai lệnh export CSV ở cell cuối. Cell pipeline có checkpoint evaluation để có thể tiếp tục sau lỗi rate limit.

## Giới hạn và diễn giải

Golden dataset mới đã được kiểm tra: 50 ID duy nhất, không trùng câu hỏi, không có reference answer/evidence trống. Điểm benchmark live phụ thuộc dataset sample, Neo4j instance và model/API tại thời điểm chạy. Khi chạy live, cần ghi lại latency, token usage, judge rationale và số lần super-node bị cắt để kết quả có thể audit.
