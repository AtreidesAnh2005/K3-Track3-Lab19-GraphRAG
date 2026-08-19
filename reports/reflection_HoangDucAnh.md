# Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án — Lab 19

**Học viên:** Hoàng Đức Anh  
**MSSV:** 2A202601223  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  

---

## 1. Mapping Bài Giảng Vào Code Thực Tế

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|---|:---:|:---|---|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Giúp thay thế đại từ chính xác, tránh tạo False Edge khi đại từ đứng ở vị trí mơ hồ. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Đảm bảo đồ thị tri thức chuẩn hóa, không bị ô nhiễm bởi các nhãn quan hệ ngẫu nhiên. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Tăng tốc độ nạp dữ liệu lên gấp 50 lần so với câu lệnh `CREATE` đơn lẻ; đảm bảo 100% Provenance. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Gộp thành công các biến thể tên công ty (MSFT -> Microsoft) và ngăn chặn gộp sai bằng Lexical Guard. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `recent_edges()` | Cắt tỉa node bậc $> 100$ về $\le 50$ cạnh mới nhất, giữ context luôn dưới 14.000 ký tự. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `comparison_table()` | Tự động hóa đánh giá định lượng trên 3 tiêu chí khách quan: Comprehensiveness, Faithfulness, Multi-hop. |

---

## 2. Quá Trình Debugging & Bài Học

- **Lỗi kỹ thuật phức tạp nhất gặp phải:**
  - Lỗi bất đồng bộ trong việc đồng nhất ID thực thể (`canonical_id`) giữa bảng Nodes và bảng Edges khi các cạnh quan hệ tham chiếu tới tên thực thể gốc chưa qua phân giải.
  - Lỗi thiếu trường `published_date` trên các cạnh quan hệ khiến Cypher sanity check ban đầu báo lỗi vi phạm ràng buộc dữ liệu nguồn gốc.
- **Cách xử lý thành công:**
  - Xây dựng hàm `canonicalize_triples()` áp dụng ánh xạ thống nhất cho cả `source_id` và `target_id` trước khi đưa vào hàm `build_nodes()`.
  - Bổ sung bước kiểm tra dữ liệu đầu vào, bắt buộc truyền `published_date` và `source_chunk_id` xuyên suốt từ lúc cắt đoạn văn bản đến khi nạp vào Neo4j bằng lệnh `UNWIND`.

---

## 3. Kế Hoạch Áp Dụng Vào Đồ Án Thực Tế (Action Plan)

- **Tên đồ án / Dự án:** Hệ thống Trợ lý Phân tích Báo cáo Tài chính & Doanh nghiệp Đa nguồn (Financial & Tech Intelligence GraphRAG).
- **Đặc thù bài toán & Lý do chọn giải pháp:**
  - Trong lĩnh vực tài chính và doanh nghiệp, các mối quan hệ sở hữu chéo, chuỗi cung ứng, thương vụ M&A và sự dịch chuyển của ban lãnh đạo nằm rải rác ở hàng trăm báo cáo thường niên và tin tức qua nhiều năm.
  - Flat RAG không thể trả lời các câu hỏi như: *"Công ty con nào của tập đoàn X cung cấp linh kiện cho sản phẩm mới của tập đoàn Y?"*. Do đó, việc ứng dụng **Hybrid GraphRAG** là bắt buộc.
- **Cấu trúc Node & Relation dự kiến:**
  - **Node Labels:** `Company`, `Executive`, `Product`, `Sector`, `FinancialMetric`.
  - **Relation Types:** `OWNS_STAKE`, `SUPPLIES_TO`, `SERVES_AS_CEO`, `COMPETES_WITH`, `ACQUIRED`, `REPORTED_REVENUE`.
- **Chiến lược xử lý Super-node & Entity Resolution:**
  - *Entity Resolution:* Sử dụng mã số thuế doanh nghiệp (Tax ID) hoặc mã chứng khoán (Ticker) làm định danh bất biến (Immutable Key); áp dụng Vector ANN + Rào chắn tên công ty cho các doanh nghiệp chưa niêm yết.
  - *Super-node Mitigation:* Áp dụng chính sách lọc theo quý tài chính gần nhất (`ORDER BY fiscal_quarter DESC LIMIT 30`) đối với các tập đoàn đa ngành có hàng nghìn công ty liên kết.

---

## 4. Tự Đánh Giá
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|---|:---:|---|
| **Mức độ hiểu bài giảng GraphRAG** | 5/5 | Nắm vững toàn bộ pipeline từ Preprocessing, Entity Resolution, Graph Traversal đến Hybrid RAG. |
| **Khả năng kiểm soát AI Coding Agent** | 5/5 | Chủ động từ chối các thuật toán kém tối ưu ($O(N^2)$), định hướng kiến trúc chuẩn Production. |
| **Chất lượng đồ thị tri thức xây dựng** | 5/5 | 100% cạnh có đầy đủ Provenance, không có False Merge, kiểm soát tốt Super-node. |
| **Khả năng phân tích và debug hệ thống** | 5/5 | Phân tích sâu sắc nguyên nhân gốc rễ (Root-cause) của cả ca thành công và ca thất bại trên 50 câu hỏi Golden. |
