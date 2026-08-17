# Báo cáo Thực hành Lab 17: Multi-Memory Agent với Zep

## 1. Câu hỏi Lý thuyết & Thiết kế

* **Tầng bộ nhớ quan trọng nhất:** Trong bộ test này, **Long-term Memory** quan trọng nhất vì chiếm 4/11 case cốt lõi (**E02, E03, E08, E09**), đóng vai trò quyết định trong việc duy trì sở thích ngôn ngữ (Python), theo dõi open-loop task (16:00), xử lý cập nhật xung đột (recency E08) và cách ly dữ liệu người dùng (E09).
* **Trade-off Zep vs. Redis + Qdrant:** Zep Cloud V3 tự động hóa graph extraction, entity resolution, temporal validity (recency) và context assembly theo relevance; giảm mạnh công sức phát triển. Ngược lại, tự triển khai Redis/Qdrant cho phép toàn quyền kiểm soát độ trễ, lưu trữ và embedding, nhưng đòi hỏi tự xây dựng toàn bộ pipeline trích xuất tri thức, chống rò rỉ dữ liệu và giải quyết xung đột.
* **Guardrail chống Memory Poisoning:** (1) Xác thực provenance và quyền sở hữu (`user_id` namespace isolation); (2) Lọc/redact PII và kiểm tra consent trước khi ingest; (3) Áp dụng schema validation và rate limit ngăn prompt injection ghi đè dữ liệu sai lệch vào durable memory.

## 2. Phân tích Benchmark

* **Hit rate các tầng:** Ở cấu hình `student` chuẩn, cả 4 tầng đều đạt 100% hit rate trên practice set (11/11). Ngược lại, baseline `no_memory` thất bại ở toàn bộ các tầng durable do thiếu bộ nhớ cross-session.
* **Query tiêu tốn token nhất:** Case **E07** (mixed) và **E08** (recency/facts dày đặc) tiêu tốn nhiều token nhất do cần nạp đa nguồn dữ liệu (long-term facts + semantic rules).
* **Case Mixed (E07):** Kết hợp **Long-term** (lấy user preference) và **Semantic** (lấy payment rules). Evidence bắt buộc: `"Python"` và `"Idempotency-Key"`.
* **Token Reduction:** Đạt mức giảm token 70–85% so với raw transcript nhờ Sliding Window, Compaction và Context Budget (10/4/3/3). `no_memory` có reduction cao nhưng hit rate = 0 vì mất hoàn toàn ngữ cảnh cần thiết.

## 3. Phân tích Recency (E08) & Compaction (E10)

* **E08 (Recency):** Zep gán validity range theo dòng thời gian, ưu tiên quyết định mới nhất (`BLUEBIRD-42`, `TypeScript`, `NestJS`) thay thế cấu hình cũ.
* **E10 (Compaction):** Khi vượt ngưỡng áp lực token, cơ chế sliding window cô đọng turn cũ thành durable notes, bảo tồn nguyên vẹn deadline `REVIEW-DEADLINE-1600` (Friday, 16:00) dù tin nhắn gốc đã bị evict.
