# Technical Defense — Lab 19 GraphRAG vs Flat RAG

**Học viên:** Lê Anh Tiến

**Phạm vi thực nghiệm:** 5.000 source rows, 50 golden questions

**Ngày chốt kết quả:** 20/08/2026

## 1. Conservative Coreference Resolution

Một ca khó được spot-check ở source row `1811`, bài *What Big Tech CEOs are saying about their companies' layoffs*. Đoạn dữ liệu nhắc `Clarke`, `Dell` và `Phillips`, sau đó dùng đại từ `it`. Nếu chọn thực thể gần nhất một cách máy móc, `it` có thể bị gán sang Dell dù mệnh đề đang chuyển sang Phillips, từ đó tạo false edge về hoạt động cắt giảm nhân sự.

Pipeline chỉ rewrite khi antecedent xuất hiện rõ trong cùng chunk. Trường hợp mơ hồ được giữ nguyên và ghi vào `unresolved_mentions`. Đây là lựa chọn ưu tiên precision: bỏ sót một edge có thể bổ sung ở lần extraction sau, nhưng false edge sẽ làm BFS khuếch đại thông tin sai sang nhiều câu trả lời.

## 2. Entity Resolution Threshold và Lexical Guard

Ngưỡng vector được giữ ở `0.90`; lexical guard yêu cầu `SequenceMatcher >= 0.72` sau khi bỏ hậu tố pháp nhân. Audit live có 128 candidate decisions và toàn bộ là `REJECT_GUARD`, vì không cặp nào đạt ngưỡng merge. Cặp gần nhất là `Dell Technologies Inc.` và `Dell Technologies`, cosine `0.8266`, bị từ chối vì `COSINE_BELOW_THRESHOLD`.

Dataset live không sinh cặp nào vừa có cosine `>0.85` vừa bị lexical guard chặn; vì vậy báo cáo không gán nhầm một ví dụ threshold reject thành lexical reject. Nhánh lexical guard vẫn được thực thi sau vector threshold trong `build_resolution_map()`. Kết quả này cho thấy cấu hình hiện tại thiên về precision nhưng bỏ lỡ alias Dell; cách sửa an toàn là thêm alias đã review, không hạ global threshold.

## 3. Union-Find và tính kiểm toán được

Manual aliases được xử lý trước, vector candidates sau đó mới đi qua lexical guard. Những merge được chấp nhận được đưa vào Disjoint-Set Union để bảo đảm tính bắc cầu và deterministic. Canonical name được chọn từ surface form phổ biến, còn mọi quyết định được giữ trong `entity_resolution_audit_df` và xuất thành `outputs/entity_resolution_audit.csv`.

Không dùng all-pairs cosine vì độ phức tạp `O(N²)`. ANN top-k giới hạn số candidate, phù hợp hơn khi tăng số entity.

## 4. Schema và Allowlist Guard

Mọi node có base label `Entity` và một trong ba type `Company`, `Person`, `Technology`. Relation chỉ được nhận nếu thuộc:

`ACQUIRED`, `DEVELOPED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`, `USES`, `LEADS`.

Output ngoài allowlist bị loại trước khi persistence. Cách này ngăn LLM tự mở rộng schema bằng relation tùy ý và giúp Cypher, index, retrieval giữ được contract ổn định.

## 5. Bulk Neo4j Ingestion

`bulk_insert_nodes()` và `bulk_insert_edges()` dùng `UNWIND $rows AS row`, batch size mặc định 1.000. Unique constraint `entity_id` và các range index `name_norm` cho Entity/Company/Person/Technology đã được xác minh ở trạng thái `ONLINE`.

So với một transaction cho mỗi triple, batch UNWIND giảm round trip, vẫn giới hạn kích thước transaction và cho phép retry theo batch.

## 6. Provenance Integrity

Mỗi edge lưu `source_chunk_id`, `published_date`, `evidence`, `confidence`. Kiểm tra trực tiếp Neo4j ngày 20/08/2026 cho kết quả:

- 64 nodes.
- 53 edges.
- 0 edge thiếu `source_chunk_id`, `published_date` hoặc `evidence`.
- 0 edge thiếu `confidence` hoặc có evidence rỗng.

Provenance được đưa vào graph context dưới dạng `date`, `chunk`, `evidence`, cho phép truy ngược câu trả lời về nguồn.

## 7. Super-node Mitigation

Ngưỡng super-node là degree `>100`; node đó chỉ lấy tối đa 50 cạnh mới nhất. Toàn traversal còn bị chặn bởi `GLOBAL_EDGE_CAP=250` và `MAX_GRAPH_CONTEXT_CHARS=14000`.

Live graph chưa kích hoạt cap: ServiceNow degree 9, Microsoft 8, NVIDIA 4. Notebook có boundary test deterministic: degree 100 giữ requested limit, degree 101 bị hạ còn 50. Ưu điểm của ưu tiên cạnh mới là kiểm soát context và phù hợp câu hỏi hiện thời; rủi ro là làm mất sự kiện lịch sử. Production nên thêm date window/query intent thay vì luôn dùng recency.

## 8. Flat RAG và Hybrid GraphRAG

Flat RAG dùng normalized MiniLM embeddings và FAISS `IndexFlatIP`, lấy top-6 chunks. Hybrid GraphRAG thực hiện seed extraction, exact/alias match, vector fallback, BFS tối đa 2 hops, textualize graph có provenance rồi bổ sung top-4 vector chunks.

Graph context giúp nối các mốc quan hệ rải qua nhiều tài liệu; vector context giữ độ bao phủ khi extraction thiếu edge. Nếu không tìm thấy seed, hệ thống trả diagnostics `NO_SEED` thay vì âm thầm dựng graph context không liên quan.

## 9. Benchmark và Failure Cases

LLM Judge đã chấm đủ 50/50 câu trên ba tiêu chí, có rationale, latency và token usage.

| Metric | Flat RAG | GraphRAG | Delta Graph - Flat |
|---|---:|---:|---:|
| Comprehensiveness | 3.460 | 3.680 | +0.220 |
| Faithfulness | 3.660 | 3.940 | +0.280 |
| Multi-hop reasoning | 3.460 | 3.680 | +0.220 |
| Latency trung bình | 2.968 s | 6.469 s | +3.501 s |
| Token trung bình | 962.56 | 1170.56 | +208.00 |

Ở `G5000-06`, Flat RAG đạt 1/1/1 còn GraphRAG đạt 5/5/5 nhờ nối được ba mốc ServiceNow–NVIDIA–AI Lighthouse. Ngược lại, ở `G5000-26`, Flat đạt 5/5/5 còn GraphRAG đạt 1/1/1 vì graph context thiếu edge/provider Cohere và hybrid prompt kết luận không có bằng chứng.

## 10. Trade-offs, Agent Control và Scale 350MB

GraphRAG tăng ba metric chất lượng nhưng latency cao gấp khoảng `2.18×` và token tăng khoảng `21.6%`. Flat RAG hợp lý cho factoid/summary; GraphRAG nên được route cho multi-hop, timeline và cross-document.

Đề xuất bị từ chối là pairwise cosine toàn bộ entity và mở BFS không giới hạn. Cả hai đều dễ gây OOM/context explosion và không tạo được đường kiểm toán rõ ràng.

Khi scale lên 350MB, bottleneck đầu tiên là LLM extraction, sau đó là entity resolution và graph ingestion. Kế hoạch production: queue bất đồng bộ có rate limit, checkpoint/idempotency theo chunk hash, persistent HNSW/FAISS index, blocking theo entity type, UNWIND theo partition, cache embeddings, query router và temporal filter. Community reports chỉ nên bật sau khi có evaluation riêng cho global questions.
