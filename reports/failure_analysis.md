# Failure Analysis

## Case 1: GraphRAG thắng ở G5000-06

Question yêu cầu nối timeline ServiceNow từ partnership tháng 5, feature tháng 6 tới adoption program tháng 7. GraphRAG tăng điểm so với Flat RAG vì graph context giữ các quan hệ và mốc thời gian từ nhiều chunks, sau đó vector context bổ sung văn bản nguồn.

## Case 2: GraphRAG thua ở G5000-26

Question hỏi provider công nghệ trong mở rộng AI-service của Amazon và capability đi kèm. GraphRAG bị điểm thấp hơn Flat RAG. Root cause có khả năng là extraction không tạo đủ edge cho provider/capability hoặc seed matcher chọn sai node; BFS khi đó làm nổi bật subgraph không liên quan. Mitigation: kiểm tra coverage relation theo evidence rows, thêm alias seed từ `seed_entities`, và dùng self-correction hop 3 khi graph context không đủ.

## Case 3: Cross-document ambiguity ở G5000-32/G5000-33

Các câu hỏi phân biệt sự kiện OpenAI theo tháng và loại hoạt động. GraphRAG có thể trả lời sai khi một entity có nhiều edge gần nhau về thời gian; temporal cap hiện ưu tiên ngày mới nhưng chưa hiểu loại sự kiện. Cần thêm relation qualifier/event type và filter theo published-date window trước BFS.

## Super-node check

Run thực tế có top degree ServiceNow=9, Microsoft=8, NVIDIA=4; không node nào vượt threshold 100. Policy degree >100 -> tối đa 50 cạnh vẫn được kiểm tra trong code, nhưng chưa được kích hoạt bởi sample này.
