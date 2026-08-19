# Thuyết Minh Kỹ Thuật (Technical Defense) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Hoàng Đức Anh  
**MSSV:** 2A202601223  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  

---

### 1. Coreference Resolution (Phân giải đại từ)
- **Tình huống thực tế:** Trong các bài báo công nghệ có cấu trúc câu phức tạp (ví dụ thương vụ hợp tác giữa *ServiceNow*, *NVIDIA* và *Accenture* hoặc bài báo *Keysight–Synopsys* trích dẫn báo cáo từ *Palo Alto Networks*):
  - *Câu 1:* "Keysight and Synopsys announced a new IoT cybersecurity solution."
  - *Câu 2:* "According to a recent threat report by Palo Alto Networks, they warned about severe vulnerabilities in industrial devices."
- **Hiện tượng phân giải sai:** Đại từ *"they"* ở câu 2 rất dễ bị mô hình hiểu lầm là trỏ tới *Keysight và Synopsys* thay vì *Palo Alto Networks*.
- **Hậu quả đối với Knowledge Graph:** Khi phân giải sai antecedent, bước Relation Extraction sẽ sinh ra một cạnh sai (**False Edge**), ví dụ: `(Keysight)-[:DEVELOPED]->(IoT Threat Report)` hoặc `(Synopsys)-[:PARTNERED_WITH]->(Palo Alto Networks)`. Cạnh sai này làm ô nhiễm đồ thị tri thức, dẫn đến việc duyệt đồ thị (Graph Traversal) đưa ra các kết luận sai lệch nghiêm trọng.
- **Giải pháp an toàn:** Áp dụng **Conservative Coreference Resolution Prompt** — Chỉ phân giải khi thực thể được xác định với độ chắc chắn tuyệt đối trong cùng một chunk; nếu có yếu tố mơ hồ, giữ nguyên văn bản gốc và log vào `unresolved_mentions`.

---

### 2. Entity Resolution Threshold & Lexical Guard
- **Ngưỡng cosine similarity đã chọn:** `threshold = 0.90` kết hợp mô hình embedding `sentence-transformers/all-MiniLM-L6-v2` và `FAISS IndexFlatIP`.
- **Cặp thực thể điển hình bị Lexical Guard chặn:** 
  - `Sam Altman` vs `Steve Altman` (hoặc `Apple` vs `Apple Music`, `Dell` vs `Dell NativeEdge`).
  - *Độ tương đồng Vector:* $\text{Cosine Similarity} \approx 0.88 - 0.92$ (rất cao do cùng chia sẻ ngữ cảnh công nghệ và họ/tiền tố).
- **Lý do Lexical Guard ra quyết định REJECT_GUARD:**
  - Sau khi chuẩn hóa và loại bỏ hậu tố pháp lý (`strip_suffix`), hàm `merge_guard()` thực hiện so khớp chuỗi ký tự bằng `SequenceMatcher(None, na, nb).ratio()`.
  - Mặc dù vector embedding có độ tương đồng cao do cùng không gian ngữ nghĩa, sự khác biệt giữa hai tên riêng (*Sam* vs *Steve*) hoặc giữa *Tên công ty* và *Dòng sản phẩm/dịch vụ* khiến tỷ lệ so khớp từ vựng không đạt ngưỡng tối thiểu ($< 0.72$).
  - Cơ chế này ngăn chặn triệt để hiện tượng **False Merge** (gộp nhầm hai con người khác nhau thành 1 node duy nhất, hoặc gộp sản phẩm vào thực thể công ty).

---

### 3. Đồ thị & Super-node Mitigation
- **Top 3 Super-nodes trong đồ thị:** `ServiceNow` (degree 5), `Youtility` (degree 4), `Squeeze` (degree 4) (và các tập đoàn lớn `Microsoft`, `Google`, `Amazon` trong đồ thị mở rộng).
- **Ưu điểm & Rủi ro của Temporal Mitigation (Cắt tỉa theo thời gian mới nhất):**
  - **Ưu điểm:**
    1. *Kiểm soát bùng nổ ngữ cảnh:* Giới hạn tối đa 50 cạnh mới nhất và `GLOBAL_EDGE_CAP = 250` giúp độ dài context đồ thị luôn nằm trong ngưỡng an toàn ($\le 14.000$ ký tự), giảm độ trễ và chi phí Token cho LLM Generator.
    2. *Ưu tiên tính thời sự:* Đảm bảo hệ thống luôn phản ánh tình trạng công nghệ, quan hệ hợp tác và lãnh đạo mới nhất của doanh nghiệp.
  - **Rủi ro tiềm ẩn:**
    1. *Mất mát tri thức lịch sử:* Nếu người dùng hỏi về sự kiện lịch sử (ví dụ: *"Ai là người sáng lập ban đầu của công ty vào năm 2015?"*), chính sách cắt tỉa chỉ giữ lại 50 cạnh mới nhất sẽ loại bỏ các cạnh lịch sử này, khiến Graph Traversal không tìm thấy đường đi tới câu trả lời chính xác.

---

### 4. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
- **Đánh đổi Quality vs Cost vs Latency:**
  - *Flat RAG:* Indexing nhanh, chi phí thấp, nhưng không giải quyết được các câu hỏi suy luận bắc cầu nhiều bước (Multi-hop) khi thông tin nằm rải rác ở nhiều tài liệu.
  - *GraphRAG:* Chi phí đầu tư ban đầu cao hơn (gọi LLM để giải quyết Coreference, trích xuất NER/RE, Entity Resolution và nạp vào Neo4j). Bù lại, GraphRAG cung cấp khả năng suy luận logic minh bạch và truy vết nguồn gốc (Data Provenance) tuyệt đối.
- **Quyết định kiểm soát AI Coding Agent:**
  - AI Agent từng đề xuất tính toán toàn bộ ma trận tương đồng $O(N^2)$ Pairwise Cosine Similarity giữa tất cả các cặp thực thể trên RAM.
  - **Lý do từ chối:** Gây tràn bộ nhớ (OOM) khi số lượng thực thể lớn. Thay vào đó yêu cầu phân nhóm theo nhãn (`ALLOWED_NODE_TYPES`) và dùng FAISS IndexFlatIP kết hợp `Lexical Guard`.
- **Giải pháp kiến trúc khi Scale lên toàn bộ 350MB (~100.000 bài báo):**
  - Dùng mô hình mã nguồn mở tự host (`vLLM` chạy `Qwen2.5-7B-Instruct` hoặc `Llama-3.1-8B-Instruct`) kết hợp hàng đợi tác vụ bất đồng bộ (Celery + Redis).
  - Tối ưu Cypher `UNWIND` với batch 5.000 records, index BTREE trên ID và name.
  - Áp dụng kỹ thuật Blocking / LSH cho Entity Resolution.
