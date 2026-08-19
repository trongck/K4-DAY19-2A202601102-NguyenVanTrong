# BÁO CÁO SUY NGẪM CÁ NHÂN & KẾ HOẠCH HÀNH ĐỘNG (REFLECTION & ACTION PLAN)
## Lab 19: Production-Grade GraphRAG vs Flat RAG

**Học viên:** Nguyễn Văn Trọng  
**Khóa học:** K4 · Track 3: GraphRAG  
**Ngày hoàn thành:** 19/08/2026  

---

## 🗺️ I. MAPPING BÀI GIẢNG VÀO CODE THỰC THI

| # | Khái niệm trọng tâm trong bài giảng | Module tương ứng | Khối code / Hàm cụ thể | Quan sát thực tế & Đánh giá hiệu quả |
|---|-------------------------------------|------------------|------------------------|--------------------------------------|
| 1 | **Conservative Coreference Resolution** | Module 1 (Tiền xử lý) | `resolve_coref_batch()`, `run_coref()` | Giải quyết đại từ chỉ khi có tiền từ (antecedent) rõ ràng trong chunk. Tránh sinh cạnh ảo (false edges) khi nạp vào đồ thị. |
| 2 | **Schema & Allowlist Guard** | Module 2 (Trích xuất) | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `EXTRACT_SYSTEM` | Ràng buộc cấu trúc đồ thị theo 3 loại Node (`Company`, `Person`, `Technology`) và 8 loại Edge cho phép, loại bỏ triệt để thực thể rác. |
| 3 | **Entity Resolution (Vector ANN + Lexical Guard)** | Module 2 (Hợp nhất thực thể) | `build_resolution_map()`, `merge_guard()`, `UF` | Kết hợp FAISS Cosine Similarity (ngưỡng 0.90) với Union-Find và kiểm tra chuỗi loại bỏ hậu tố (`Inc.`, `Corp.`, `Ltd.`), ngăn chặn Over-merging. |
| 4 | **Production Bulk Cypher Ingestion** | Module 2 (Nạp dữ liệu) | `bulk_insert_nodes()`, `bulk_insert_edges()` bằng `UNWIND` | Nạp hàng nghìn Node và Edge theo từng Batch 1000 dòng, tăng tốc độ nạp dữ liệu lên gấp 20 lần so với câu lệnh đơn lẻ từng dòng. |
| 5 | **Super-node Mitigation & Provenance** | Module 3 & 5 (Duyệt đồ thị) | `retrieve_graph_context()`, `recent_edges()`, `graph_checks()` | Giới hạn tối đa 50 cạnh mới nhất đối với các node có bậc $> 100$; đảm bảo 100% các cạnh có `source_chunk_id` và `published_date` truy vết. |
| 6 | **LLM-as-a-Judge Evaluation Framework** | Module 4 (Đánh giá) | `judge_answer()`, `judge_json()`, `comparison_table()` | Tự động chấm điểm 1–5 theo 3 tiêu chí: Comprehensiveness, Faithfulness, Multi-hop Reasoning với Reference Answer làm mỏ neo chính xác. |

---

## 🛠️ II. QUÁ TRÌNH DEBUGGING & BÀI HỌC KỸ THUẬT

1. **Vấn đề kỹ thuật phức tạp nhất gặp phải:**
   - **Xử lý cấu trúc dữ liệu không đồng nhất & Quản lý Rate Limit:** Khi stream tập dữ liệu `HackerNoon`, cấu trúc cột có thể biến thiên giữa các schema (ví dụ: `text` vs `content` vs `description`). Ngoài ra, khi gọi LLM theo batch cho 400 chunks, nếu gọi tuần tự từng request đơn lẻ sẽ dễ bị lỗi nghẽn hoặc chạm giới hạn Rate Limit của Groq/OpenAI.
2. **Cách xử lý & Bài học đúc kết:**
   - Đã chuẩn hóa hàm `pick_col()` và `standardize_news()` linh hoạt tự động dò tìm các cột văn bản hợp lệ.
   - Thiết kế cơ chế gọi LLM theo batch (batch size 4–5 chunks) kết hợp với thuật toán **Exponential Backoff Retry** (`time.sleep(2**attempt + random.random())`), giúp pipeline chạy mượt mà 100% mà không bị gián đoạn giữa chừng.

---

## 🎯 III. KẾ HOẠCH ÁP DỤNG VÀO ĐỒ ÁN THỰC TẾ (ACTION PLAN)

* **Tên đồ án / Dự án dự kiến:** Hệ thống Trợ lý AI Phân tích Rủi ro và Mối quan hệ Doanh nghiệp trong Chuỗi Cung ứng (Enterprise Supply-Chain & M&A Intelligence Assistant).
* **Lý do lựa chọn GraphRAG:**
  - Bài toán chuỗi cung ứng và đầu tư sở hữu mang tính chất mạng lưới phức tạp (đa tầng, đa quan hệ, nhiều công ty con và cổ đông sở hữu chéo).
  - Flat RAG thông thường chỉ trả lời được các câu hỏi đơn lẻ trong 1 bài báo, hoàn toàn bất lực trước câu hỏi: *"Nếu nhà cung cấp A bị đình chỉ hoạt động, những doanh nghiệp sản xuất nào ở tầng 2 và tầng 3 trong đồ thị sẽ bị ảnh hưởng?"*.
* **Cấu trúc Schema dự kiến:**
  - **Nodes:** `Enterprise`, `Product`, `Material`, `Executive`, `Geographic_Location`.
  - **Relations:** `SUPPLIES_TO`, `OWNS_STAKE_IN`, `SUBSIDIARY_OF`, `LOCATED_IN`, `PARTNERED_WITH`.
* **Chiến lược kiểm soát đồ thị:**
  - Áp dụng cơ chế **Temporal Super-node Mitigation** để lọc các sự kiện hợp tác cung ứng trong 12 tháng gần nhất.
  - Sử dụng **Entity Resolution với Lexical Guard** chặt chẽ để phân biệt các công ty con cùng tập đoàn nhưng có mã số thuế / pháp nhân độc lập.

---

## 📋 IV. TỰ ĐÁNH GIÁ (SELF-ASSESSMENT)

| Tiêu chí đánh giá | Điểm tự chấm (1–5) | Ghi chú minh chứng |
| :--- | :---: | :--- |
| **Mức độ thấu hiểu kiến trúc GraphRAG** | **5 / 5** | Nắm vững toàn bộ pipeline từ Preprocessing, Ingestion UNWIND đến Hybrid Traversal và LLM-as-a-Judge. |
| **Khả năng kiểm soát AI Coding Agent** | **5 / 5** | Tự tay kiểm soát toàn bộ logic, từ chối các đề xuất gây tràn bộ nhớ ($O(N^2)$), đảm bảo an toàn bí mật API Key. |
| **Chất lượng đồ thị tri thức xây dựng** | **5 / 5** | Đồ thị 488 nodes, 306 edges với 100% provenance hợp lệ (`invalid_provenance_edges = 0`), kiểm soát super-nodes chặt chẽ. |
| **Khả năng phân tích lỗi & Thuyết minh** | **5 / 5** | Trả lời đầy đủ, sâu sắc 10 câu hỏi bảo vệ kiến trúc, khớp số liệu thực nghiệm 100%. |
