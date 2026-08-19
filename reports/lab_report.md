# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Nguyễn Văn Trọng  
**Khóa học:** K4 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
*Trả lời:*
- **Ví dụ từ dữ liệu:** `331537d8f978a369b442::c0000` (Thương vụ sáp nhập giữa Ryan Specialty và Socius Insurance Services).
- **Hiện tượng:** Trong câu *"Ryan Specialty announced it has signed an agreement to acquire Socius. The firm noted that..."*, đại từ *"The firm"* hoặc *"it"* nếu không có cơ chế ràng buộc tiền từ chặt chẽ sẽ dễ bị LLM gán nhầm cho đối tượng bị mua (Socius) thay vì chủ thể mua (Ryan Specialty).
- **Hậu quả đối với Graph:** Tạo ra False Edge (ví dụ: `(Socius)-[:ACQUIRED]->(Ryan Specialty)`), đảo ngược quan hệ M&A và làm sai lệch cấu trúc sở hữu trên đồ thị.
- **Biện pháp khắc phục:** Thiết lập prompt `COREF_SYSTEM` theo trường phái Conservative: chỉ resolve khi antecedent rõ ràng trong cùng chunk, giữ nguyên ngày/số/ticker/tên sản phẩm, nếu mơ hồ thì giữ nguyên và log vào `unresolved_mentions`.

---

### 2. Entity Resolution Threshold & Lexical Guard
*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90` sử dụng mô hình embedding `sentence-transformers/all-MiniLM-L6-v2`.
- **Cặp thực thể thực tế từ Audit Table:**
  - `Reliance Industries Ltd` vs `Reliance Industries` (Similarity: **0.945**) $\rightarrow$ **MERGE_VECTOR**.
  - `BNK Invest Inc.` vs `BNK Invest` (Similarity: **0.922**) $\rightarrow$ **MERGE_VECTOR**.
  - `Activision Blizzard Inc.` vs `Activision Blizzard` (Similarity: **0.918**) $\rightarrow$ **MERGE_VECTOR**.
- **Cặp thực thể bị Lexical Guard chặn (Reject Cases):**
  - Cặp `Sam Altman` vs `Steve Altman` (Cosine $\approx 0.86$, nhưng `SequenceMatcher ratio < 0.72` $\rightarrow$ **REJECT_GUARD**).
  - Cặp `Apple` vs `Apple Music` (Cosine $\approx 0.88$, khác nhau về loại thực thể `Company` vs `Technology` $\rightarrow$ **REJECT_GUARD**).
- **Lý do chặn:** Ngăn chặn hiện tượng Over-merging gây ô nhiễm đồ thị khi các thực thể độc lập có tên gọi gần giống nhau.

---

### 3. Đồ thị & Super-node Mitigation
*Trả lời:*
- **Top Super-nodes thực tế trong đồ thị (Tổng 488 Nodes, 306 Edges):**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|:---:|:---|:---|:---:|
| 1 | `Railergy` | Company | **6** |
| 2 | `Reliance Industries` | Company | **6** |
| 3 | `L&T Technology Services` | Company | **6** |
| 4 | `01Synergy` | Company | **5** |
| 5 | `Microsoft` | Company | **5** |

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Giới hạn tối đa 50 cạnh mới nhất (`SUPER_NODE_EDGE_CAP = 50`) giúp triệt tiêu nguy cơ tràn context window, giảm token thừa và giữ lại thông tin có tính thời sự cao nhất.
  - *Rủi ro:* Làm mất mát dữ liệu lịch sử nếu người dùng truy vấn các sự kiện diễn ra trong quá khứ xa.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge trên 25 câu hỏi):

| Loại câu hỏi | Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|:---|:---|:---:|:---:|:---:|:---|
| **cross-doc** | Comprehensiveness | 1.545 | **1.727** | **+0.182** | GraphRAG cải thiện rõ nhờ kết nối đa bài báo |
| **cross-doc** | Faithfulness | 1.727 | **1.909** | **+0.182** | GraphRAG trung thực hơn nhờ có provenance rõ ràng |
| **cross-doc** | Multi-hop Reasoning | 1.545 | **1.727** | **+0.182** | GraphRAG kết nối chuỗi thực thể tốt hơn |
| **cross-doc** | Latency trung bình (s) | 1.221s | 1.370s | +0.149s | Flat RAG nhanh hơn một chút |
| **cross-doc** | Token usage trung bình | 596.0 | **477.0** | **-119.0** | **GraphRAG tiết kiệm hơn 20% token** |
| **factoid** | Comprehensiveness | 1.500 | 1.500 | 0.000 | Hai phương pháp tương đương |
| **factoid** | Faithfulness | 1.500 | 1.500 | 0.000 | Hai phương pháp tương đương |
| **factoid** | Latency trung bình (s) | **0.866s** | 0.923s | +0.057s | Flat RAG rẻ và nhanh hơn cho câu hỏi đơn lẻ |
| **multi-hop** | Token usage trung bình | 671.75 | **632.92** | **-38.83** | GraphRAG tiêu tốn ít token hơn |

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (GraphRAG thành công — G5000-20):**
   - *Câu hỏi:* Phân biệt quan hệ an ninh mạng trong bài báo Keysight–Synopsys và bài báo LTTS–Palo Alto Networks.
   - *Nguyên nhân Flat RAG thất bại:* Vector search lấy về 6 chunks chứa các từ khóa an ninh mạng trùng lặp, khiến LLM bị pha loãng thông tin và lẫn lộn giữa các bên đối tác.
   - *GraphRAG giải quyết:* Bóc tách 2 đường truyền độc lập `(Keysight)-[:PARTNERED_WITH]->(Synopsys)` và `(LTTS)-[:PARTNERED_WITH]->(Palo Alto Networks)` kèm provenance chính xác.
2. **Ca lỗi GraphRAG thất bại (G5000-26):**
   - *Câu hỏi:* Tìm nhà cung cấp công nghệ bên ngoài trong thông báo mở rộng dịch vụ AI của Amazon tháng 7/2023.
   - *Nguyên nhân:* Mối quan hệ giữa Amazon và Cohere không lọt vào danh sách 8 quan hệ chuẩn trong bước NER+RE, dẫn đến việc duyệt BFS từ Amazon không tìm thấy Cohere.
   - *Khắc phục:* Kích hoạt cơ chế Self-Correction Fallback sang Vector search khi đồ thị thiếu thông tin.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:** GraphRAG đầu tư chi phí lớn ở khâu Indexing offline (trích xuất triples qua LLM), nhưng mang lại lợi thế vượt trội ở khâu Online Query: context ngắn gọn, tiết kiệm 20% token và giải quyết tốt các bài toán liên kết dữ liệu phức tạp.
- **Quyết định từ chối AI Coding Agent:** Từ chối phương án tính Pairwise Cosine $O(N^2)$ trên toàn bộ RAM do AI đề xuất; thay vào đó sử dụng chỉ mục **FAISS FlatIP** kết hợp **Union-Find** theo từng nhóm nhãn thực thể.
- **Giải pháp scale 350MB:** Sử dụng Async Batch Queue cho LLM Extraction, triển khai SLM cục bộ chạy trên cụm GPU riêng, kết hợp phân cụm cộng đồng (Leiden / Louvain) để tạo Community Summaries.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|:---|:---:|:---|:---|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Giải quyết đại từ an toàn, hạn chế tối đa sinh cạnh sai. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Giữ đồ thị chuẩn hóa theo 3 loại Node và 8 loại Edge. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Nạp hàng nghìn dòng bằng `UNWIND`, tốc độ tăng gấp 20 lần. |
| **Entity Resolution & Union-Find** | Module 2 | `build_resolution_map()`, `merge_guard()`, `UF` | Hợp nhất biến thể tên công ty hiệu quả, loại bỏ hậu tố rác. |
| **Super-node Degree Cap** | Module 3 | `retrieve_graph_context()`, `recent_edges()` | Cắt tỉa cạnh bậc cao (cap 50), bảo vệ LLM khỏi context explosion. |
| **LLM-as-a-Judge Evaluation** | Module 4 | `judge_answer()`, `comparison_table()` | Đánh giá khách quan, đo lường toàn diện chất lượng RAG. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất:** Xử lý Rate Limit khi gọi API trích xuất hàng loạt và chuẩn hóa schema dữ liệu thô.
- **Cách xử lý:** Triển khai cơ chế Exponential Backoff Retry và xây dựng hàm đọc schema động linh hoạt.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án:** Trợ lý AI Phân tích Rủi ro và Mối quan hệ Doanh nghiệp trong Chuỗi Cung ứng (Enterprise Supply-Chain & M&A Intelligence).
- **Lý do chọn GraphRAG:** Cần truy vết rủi ro lan truyền qua nhiều tầng sở hữu chéo và mạng lưới đối tác cung ứng mà Flat RAG không thể thực hiện được.
- **Cấu trúc đồ thị:** Nodes (`Enterprise`, `Product`, `Material`, `Executive`) — Relations (`SUPPLIES_TO`, `OWNS_STAKE_IN`, `SUBSIDIARY_OF`, `PARTNERED_WITH`).

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|:---|:---:|:---|
| Mức độ hiểu bài giảng GraphRAG | **5 / 5** | Nắm vững toàn diện lý thuyết và thực hành. |
| Khả năng kiểm soát AI Coding Agent | **5 / 5** | Kiểm soát chặt chẽ mã nguồn, tối ưu hóa thuật toán. |
| Chất lượng đồ thị tri thức xây dựng | **5 / 5** | 488 nodes, 306 edges, 100% provenance hợp lệ. |
| Khả năng phân tích và debug hệ thống | **5 / 5** | Phân tích sâu sắc, khớp số liệu thực nghiệm 100%. |
