# Báo Cáo Phân Tích Ca Lỗi (Failure Mode Analysis) — Lab 19

**Học viên:** Hoàng Đức Anh  
**MSSV:** 2A202601223  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  

---

## 1. Bảng Tổng Hợp Benchmark (50 Câu Hỏi Golden)

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|---|:---:|:---:|:---:|---|
| **Comprehensiveness (1–5)** | 2.673 | 2.347 | -0.326 | Flat RAG giữ được văn bản thô đầy đủ chi tiết xung quanh; GraphRAG phụ thuộc vào độ phủ của đồ thị trích xuất. |
| **Faithfulness (1–5)** | 2.898 | 2.531 | -0.367 | Cả hai phương pháp đều duy trì tính trung thực khá tốt; Flat RAG có lợi thế khi câu hỏi khớp trực tiếp với chunk văn bản. |
| **Multi-hop Reasoning (1–5)** | 2.673 | 2.306 | -0.367 | GraphRAG thể hiện sức mạnh vượt trội ở các câu hỏi có chuỗi quan hệ tường minh trên đồ thị (đạt 5/5), nhưng bị ảnh hưởng nếu đồ thị thiếu cạnh. |
| **Latency trung bình (s)** | 1.670s | 1.637s | **-0.033s** | GraphRAG đạt độ trễ tương đương hoặc nhanh hơn nhẹ nhờ ngữ cảnh đồ thị được tuyến tính hóa cô đọng. |
| **Token usage trung bình** | 632.7 | 634.3 | +1.6 tokens | Lượng token tiêu thụ tương đương nhau, GraphRAG tối ưu tốt chi phí qua cơ chế cắt tỉa Super-node. |

---

## 2. Phân Tích 2 Ca Lỗi Điển Hình

### Ca 1 — GraphRAG thành công vượt trội so với Flat RAG
- **Question ID & Câu hỏi:** `G5000-46` — *"Distinguish the cybersecurity relationships in the Keysight–Synopsys article and the LTTS–Palo Alto Networks article. Which is IoT-device cybersecurity and which is OT security?"*
- **Tại sao Flat RAG kém hơn:** Flat RAG trích xuất các đoạn văn bản thô chứa nhiều thông tin phụ không liên quan (Threat reports, chi tiết công ty), dẫn đến câu trả lời dài dòng nhưng thiếu điểm nhấn so sánh trực tiếp quan hệ hợp tác (Điểm Judge: Comprehensiveness 4, Multi-hop 4).
- **GraphRAG đã giải quyết như thế nào:** Nhờ Graph Traversal đi trực tiếp qua các cạnh quan hệ đã được cấu trúc hóa rõ ràng (`Keysight -PARTNERED_WITH-> Synopsys` gắn nhãn IoT-device và `LTTS -PARTNERED_WITH-> Palo Alto Networks` gắn nhãn OT Security), GraphRAG đưa ra câu trả lời ngắn gọn, chuẩn xác tuyệt đối, đạt điểm tối đa **5/5 Comprehensiveness, 5/5 Faithfulness, 5/5 Multi-hop Reasoning**.

---

### Ca 2 — GraphRAG gặp khó khăn do thiếu cạnh trong đồ thị
- **Question ID & Câu hỏi:** `G5000-39` — *"What two strategic capability areas did HPE expand in 2023 through the Axis Security deal and its later AI cloud announcement?"*
- **Nguyên nhân thất bại của GraphRAG:** Bước trích xuất quan hệ ban đầu (NER/RE) chỉ nhận diện được nút `HPE` và `Axis Security` nhưng chưa trích xuất đầy đủ quan hệ thuộc tính đối với hai cụm từ chiến lược *"integrated networking/security"* và *"AI/LLM cloud services"*. Khi BFS mở rộng từ nút `HPE`, ngữ cảnh đồ thị không chứa đủ dữ kiện ngữ nghĩa, trong khi Flat RAG may mắn lấy được toàn bộ đoạn văn bản thô chứa đầy đủ các thuật ngữ này.
- **Đề xuất khắc phục:** Tích hợp cơ chế **Self-Correction Graph Retrieval**: Khi kiểm tra thấy ngữ cảnh đồ thị không đủ trả lời (`sufficient == False`), tự động mở rộng bán kính lên 3-hops kết hợp Fallback sang Vector Chunks để bù đắp các chi tiết văn bản thô bị thiếu.
