# Failure Analysis — Flat RAG vs GraphRAG

## Phương pháp truy vết

Mỗi ca được chọn trực tiếp từ `outputs/graphrag_eval_results.csv`. Quy trình phân tích gồm: so sánh answer với reference, đọc judge rationale, đối chiếu chunk citation, xác định lỗi ở retrieval/extraction/seed/traversal, rồi đề xuất biện pháp có thể kiểm thử.

## Case 1 — Flat RAG thất bại, GraphRAG thành công (`G5000-06`)

**Câu hỏi:** Chuỗi phát triển generative AI của ServiceNow từ tháng 5 đến cuối tháng 7/2023 gồm partnership, feature và adoption program nào?

| Hệ thống | Comp. | Faith. | Multi-hop |
|---|---:|---:|---:|
| Flat RAG | 1 | 1 | 1 |
| GraphRAG | 5 | 5 | 5 |

Flat RAG không lấy được mốc tháng 5, nhầm feature tháng 6 thành marketplace và nhầm chương trình tháng 7 thành Cohere. Root cause là top-k semantic search ưu tiên các chunk có từ khóa generative AI nhưng không giữ được ba mốc theo cùng entity/timeline.

GraphRAG nối đúng:

1. ServiceNow `PARTNERED_WITH` NVIDIA trong tháng 5.
2. ServiceNow `DEVELOPED` Now Assist for Virtual Agent trong tháng 6.
3. ServiceNow, NVIDIA và Accenture gắn với AI Lighthouse cuối tháng 7.

Graph answer dẫn các chunk `531b5c0beb405c822e6c::c0000`, `5d479b59c6d424d4a679::c0000`, `84b909fc526ba22d835f::c0000`, `a5a8b0ece135c638d4f0::c0000`. Đây là ca chứng minh lợi ích multi-hop/cross-document của graph traversal.

## Case 2 — GraphRAG thất bại, Flat RAG thành công (`G5000-26`)

**Câu hỏi:** Provider bên ngoài nào xuất hiện trong mở rộng AI-service của Amazon và capability nào đi kèm?

| Hệ thống | Comp. | Faith. | Multi-hop |
|---|---:|---:|---:|
| Flat RAG | 5 | 5 | 5 |
| GraphRAG | 1 | 1 | 1 |

Flat RAG lấy đúng chunk `6ec376fffd2c56fef630`, trả lời Cohere và chương trình xây dựng conversational customer-service agents. GraphRAG lại nói context không có thông tin.

Root cause có thể khoanh vùng ở graph coverage/seed resolution: hoặc extraction không tạo edge Amazon–Cohere/capability, hoặc seed `Amazon` không resolve tới node chứa edge đó. Đây không phải lỗi super-node vì live graph không có node degree >100.

**Khắc phục có thể kiểm thử:**

- Thêm coverage check: mọi `required_relations`/evidence row trong golden phải map được ít nhất một edge hoặc được đánh dấu `VECTOR_ONLY`.
- Log seed candidates và matched node ID vào evaluation output.
- Nếu graph context rỗng/không chứa seed, route sang vector fallback trước khi generate answer.
- Chạy self-correction hop 2 → hop 3 → vector, có stop condition.

## Case 3 — Thiếu temporal/event qualifier (`G5000-32`, `G5000-33`)

`G5000-32` yêu cầu phân biệt plug-in tháng 3 với app-store plan tháng 6. Flat đạt 4/4/4, Graph đạt 2/2/2 vì graph context chỉ giữ được sự kiện tháng 6. `G5000-33` yêu cầu phân biệt AP–OpenAI collaboration với voluntary White House commitment; Flat đạt 5/5/5, Graph đạt 2/3/2 vì thiếu sự kiện governance.

Hai ca cho thấy relation type tổng quát chưa đủ mô tả event state. Cần bổ sung qualifier như `event_type`, `announced_at`, `effective_at`, `status` và filter time window trước BFS. Một event node riêng cũng phù hợp hơn việc gắn nhiều mốc thời gian trực tiếp lên Company–Company edge.

## Super-node và Entity-resolution Observations

- Top degree live: ServiceNow 9, Microsoft 8, NVIDIA 4; vì vậy cap không kích hoạt trên sample.
- Boundary test trong notebook xác minh degree 100 chưa bị cap và degree 101 chỉ còn tối đa 50 cạnh.
- Entity audit có 128 quyết định, đều là reject. Cặp gần nhất `Dell Technologies Inc.` / `Dell Technologies` có cosine 0.8266 và bị threshold 0.90 từ chối. Đây là false-negative alias candidate; nên sửa qua reviewed alias map thay vì giảm threshold toàn cục.

## Kết luận

GraphRAG thắng khi graph extraction bao phủ đủ chuỗi quan hệ, nhưng thất bại mạnh khi edge hoặc seed bị thiếu. Vì vậy production router không nên mặc định tin graph context: retrieval diagnostics, coverage checks và vector fallback là một phần bắt buộc của kiến trúc hybrid.
