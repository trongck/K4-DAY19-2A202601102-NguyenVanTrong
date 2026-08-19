# BÁO CÁO PHÂN TÍCH CA LỖI (FAILURE ANALYSIS)
## Lab 19: Production-Grade GraphRAG vs Flat RAG

**Học viên:** Nguyễn Văn Trọng  
**Khóa học:** K4 · Track 3: GraphRAG  
**Mục tiêu:** Truy vết nguyên nhân gốc rễ (Root-Cause Analysis) các trường hợp Flat RAG hoặc GraphRAG gặp thất bại/hạn chế trong quá trình thực nghiệm.

---

## 🔍 CA LỖI 1: FLAT RAG THẤT BẠI TRONG SUY LUẬN ĐỐI CHIẾU THỰC THỂ PHÂN TÁN (CROSS-DOC / MULTI-HOP)

### 1. Thông tin câu hỏi kiểm thử:
* **Question ID:** `G5000-20` (hoặc câu hỏi trích xuất quan hệ bảo mật IoT giữa Keysight – Synopsys và LTTS – Palo Alto Networks).
* **Nội dung câu hỏi:** *"Distinguish the cybersecurity relationships in the Keysight–Synopsys article and the LTTS–Palo Alto Networks article..."*
* **Đáp án chuẩn (Reference Answer):** Keysight và Synopsys là đối tác phát triển giải pháp an ninh mạng cho thiết bị IoT; trong khi L&T Technology Services hợp tác với Palo Alto Networks để cung cấp dịch vụ bảo mật OT (Operational Technology).
* **Kết quả chấm điểm của LLM-as-a-Judge:**
  - **Flat RAG:** Comprehensiveness = 4, Faithfulness = 5, Multi-hop Reasoning = 4
  - **GraphRAG:** Comprehensiveness = 5, Faithfulness = 5, Multi-hop Reasoning = **5**

### 2. Hiện tượng & Truy vết nguyên nhân gốc rễ (Root-Cause Analysis):
* **Cơ chế của Flat RAG:** Flat RAG dựa vào độ tương đồng ngữ nghĩa của toàn bộ đoạn văn bản (Vector Cosine Similarity). Vì cả hai bài viết đều chứa các từ khóa rất giống nhau (*"cybersecurity"*, *"partnership"*, *"network solutions"*), vector retriever lấy về các chunks bị chồng chéo ngữ cảnh và thiếu liên kết định hướng rõ ràng giữa từng cặp chủ thể - đối tượng. Khi LLM đọc một khối văn bản dài gồm 6 chunks khác nhau, nó dễ bị hiện tượng pha loãng thông tin và diễn giải chung chung.
* **Cơ chế vượt trội của GraphRAG:** 
  - Knowledge Graph đã bóc tách rõ ràng 2 bộ triples độc lập:
    1. `(Keysight:Company)-[:PARTNERED_WITH]->(Synopsys:Company)` kèm evidence về IoT chip design.
    2. `(L&T Technology Services:Company)-[:PARTNERED_WITH]->(Palo Alto Networks:Company)` kèm evidence về OT Security.
  - Khi Graph Traversal kích hoạt từ seed nodes `Keysight`, `Synopsys`, `L&T Technology Services`, ngữ cảnh đồ thị được tóm tắt cực kỳ tinh gọn:
    `Keysight [Company] -PARTNERED_WITH-> Synopsys [Company] | chunk=...`
    `L&T Technology Services [Company] -PARTNERED_WITH-> Palo Alto Networks [Company] | chunk=...`
  - LLM nhìn thấy ngay ranh giới cấu trúc của 2 mối quan hệ đối tác riêng biệt mà không bị nhiễu văn phong báo chí, đạt điểm tuyệt đối 5/5 về Multi-hop Reasoning.

---

## 🔍 CA LỖI 2: GRAPHRAG THẤT BẠI DO THIẾU CẠNH TRÍCH XUẤT (EXTRACTION RECALL LOSS & SEED EXTRACTION FAILURE)

### 1. Thông tin câu hỏi kiểm thử:
* **Question ID:** `G5000-26`
* **Nội dung câu hỏi:** *"What external technology provider is named inside Amazon's July AI-service expansion, and what other new AI capability is highlighted in the selected record?"*
* **Đáp án chuẩn (Reference Answer):** Amazon nêu tên nhà cung cấp công nghệ Cohere và giới thiệu chương trình hỗ trợ xây dựng ứng dụng đàm thoại thông minh.
* **Kết quả thực tế:** Cả Flat RAG và GraphRAG đều trả về điểm Comprehensiveness = 1, Faithfulness = 1.

### 2. Hiện tượng & Truy vết nguyên nhân gốc rễ (Root-Cause Analysis):
* **Nguyên nhân tầng 1 (Extraction Filtering):** Trong pha trích xuất NER+RE ở Module 2, quan hệ giữa Amazon và Cohere nằm ở một câu văn phức hợp mô tả việc người dùng được cấp quyền truy cập công nghệ của các bên thứ ba. LLM trích xuất đã ưu tiên độ chính xác (Precision) với schema nghiêm ngặt (`ALLOWED_RELATIONS`), khiến mối quan hệ lỏng lẻo này không lọt vào danh sách 8 quan hệ chuẩn hóa.
* **Nguyên nhân tầng 2 (Seed Entity Disconnect):** Khi câu hỏi truy vấn đến *"external technology provider"*, Seed Extractor nhận diện seed là `Amazon`. Tuy nhiên, do cạnh liên kết `(Amazon)-[:PARTNERED_WITH]->(Cohere)` không tồn tại trong Neo4j, việc duyệt BFS từ node Amazon không thể tìm thấy node Cohere.
* **Đề xuất khắc phục chuẩn Production (Hybrid Fallback & Self-Correction):**
  1. **Triển khai Bonus Self-Correction:** Khi Graph context trả về không chứa thông tin về nhà cung cấp bên ngoài (`missing="technology provider"`), hệ thống tự động fallback sang Vector Retrieval với $k=8$ chunks rộng hơn.
  2. **Schema Relaxation & Open Information Extraction (OpenIE):** Cho phép đồ thị lưu trữ thêm nhãn quan hệ mở (Open relations) kết hợp với thuộc tính `evidence` dạng văn bản ngắn để không bỏ sót các thực thể liên kết dạng hợp tác mềm.
