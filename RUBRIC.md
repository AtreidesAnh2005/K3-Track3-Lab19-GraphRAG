# Rubric Đánh Giá — Lab 19: Production-Grade GraphRAG vs Flat RAG

**Tổng điểm:** 100 điểm + 10 điểm Bonus · Hình thức: Thực hành Cá nhân

---

## 📊 Bảng tổng hợp phân bổ điểm

| Phần | Trọng số | Tiêu chí chính |
|------|----------|---------------|
| **1. Implementation & Pipeline** | 40% (40 điểm) | Đầy đủ 5 modules; Code chạy thông suốt; Bulk insert chuẩn `UNWIND` |
| **2. Failure Modes & Mitigations** | 20% (20 điểm) | Kiểm soát Super-node, 100% Provenance, Audit Entity Resolution |
| **3. Golden Evaluation & Benchmarking** | 20% (20 điểm) | Đánh giá Factoid/Multi-hop/Cross-doc qua LLM Judge; Xuất báo cáo CSV |
| **4. Thuyết minh Kỹ thuật & Reflection** | 20% (20 điểm) | Trả lời 10 câu hỏi bảo vệ kiến trúc; Mapping bài giảng & Action Plan |
| **5. Bonus Challenges** | Tối đa +10 điểm | Community Detection / Self-Correction / Near-Dedup |

---

## 🔍 Chi tiết tiêu chí chấm điểm

### 1. Implementation & Pipeline (40 điểm)

| # | Tiêu chí | Điểm tối đa | Phương pháp kiểm tra |
|---|----------|-------------|---------------------|
| 1.1 | **Preprocessing & Coreference:** Stream dữ liệu thành công; Khử trùng lặp; Phân giải đại từ theo conservative prompt, log `unresolved_mentions` | 8 | Chạy cell M1; Kiểm tra output DataFrame không rỗng |
| 1.2 | **NER + RE Extraction:** Trích xuất quan hệ đúng Schema (Node labels & Relation types hợp lệ); Có bằng chứng `evidence` & `confidence` | 10 | Kiểm tra DataFrame triples trích xuất được; Schema allowlist hợp lệ |
| 1.3 | **Bulk Ingestion Neo4j:** Sử dụng cú pháp `UNWIND $rows AS row` theo batch; Tạo Constraint & Indexes trên database | 10 | Kiểm tra hàm `bulk_insert_nodes` và `bulk_insert_edges`; Database có dữ liệu |
| 1.4 | **Retrieval Architecture:** Xây dựng Flat RAG FAISS Index + Hybrid Graph Retrieval (Seed extraction + BFS traversal + Context linearization) | 12 | Chạy cell M4; Kiểm tra `answer_flat_rag()` và `answer_graph_rag()` sinh kết quả |

---

### 2. Failure Modes & Mitigations (20 điểm)

| # | Tiêu chí | Điểm tối đa | Phương pháp kiểm tra |
|---|----------|-------------|---------------------|
| 2.1 | **Super-node Mitigation:** Node có bậc $degree > 100$ được cắt tỉa tự động về tối đa $\le 50$ cạnh mới nhất; Tổng số cạnh thu thập không vượt `GLOBAL_EDGE_CAP` | 8 | Chạy hàm `test_supernode_policy()`; Không gặp hiện tượng bùng nổ token/context |
| 2.2 | **Edge Provenance Integrity:** Mọi cạnh trong Neo4j đều có đầy đủ `source_chunk_id` và `published_date` | 6 | Chạy Cypher check: `invalid_provenance_edges == 0` |
| 2.3 | **Entity Resolution & Audit:** Kết hợp Vector ANN + Lexical Guard; Tạo bảng audit phân loại rõ `MERGE_MANUAL`, `MERGE_VECTOR`, `REJECT_GUARD` | 6 | Xem bảng `entity_resolution_audit_df`; Có ít nhất 10+ dòng audit minh bạch |

---

### 3. Golden Evaluation & Benchmarking (20 điểm)

| # | Tiêu chí | Điểm tối đa | Phương pháp kiểm tra |
|---|----------|-------------|---------------------|
| 3.1 | **Golden Dataset Quality:** Bộ câu hỏi có đầy đủ 3 nhóm (`factoid`, `multi-hop`, `cross-doc`); Đã điền câu trả lời chuẩn (`reference_answer`) | 6 | Kiểm tra file dataset; Không để trống reference answers |
| 3.2 | **LLM-as-a-Judge Evaluation:** Chạy đánh giá toàn bộ câu hỏi trên 3 thang điểm (Comprehensiveness, Faithfulness, Multi-hop reasoning) kèm giải thích (Rationale) | 8 | Kiểm tra checkpoint và output evaluation |
| 3.3 | **Benchmark Comparison Reports:** Xuất đầy đủ 2 file CSV vào thư mục `reports/` và có nhận xét phân tích sự chênh lệch Quality vs Latency vs Token usage | 6 | Kiểm tra `graphrag_eval_results.csv` và `graphrag_vs_flatrag_summary.csv` |

---

### 4. Thuyết minh Kỹ thuật & Reflection (20 điểm)

| # | Tiêu chí | Điểm tối đa | Phương pháp kiểm tra |
|---|----------|-------------|---------------------|
| 4.1 | **10 Câu hỏi Thuyết minh Kỹ thuật:** Trả lời sâu sắc, có dẫn chứng cụ thể từ dữ liệu thực tế (Threshold, Supernodes, Failure case, Trade-offs, Scalability) | 10 | Đánh giá nội dung file `reports/technical_defense.md` |
| 4.2 | **Báo cáo Ca lỗi (Failure Analysis):** Phân tích ít nhất 2 ca lỗi của Flat RAG hoặc GraphRAG theo quy trình truy vết nguyên nhân gốc rễ (Root-cause analysis) | 5 | Đánh giá file `reports/failure_analysis.md` |
| 4.3 | **Reflection Cá nhân:** Hoàn thành bảng Mapping bài giảng, chia sẻ bài học debug và Action Plan áp dụng vào đồ án thực tế | 5 | Đánh giá file `reports/reflection_[HọTên].md` |

---

### 5. Bonus Challenges (+10 điểm tối đa)

| Bonus | Điểm | Yêu cầu kỹ thuật |
|-------|------|-----------------|
| **Global Search via Community Reports** | +5 | Áp dụng NetworkX / GDS phân cụm cộng đồng, nạp `community_id`, sinh tóm tắt cộng đồng và truy vấn câu hỏi vĩ mô |
| **Self-Correction Graph Retrieval** | +5 | Cơ chế tự động mở rộng bán kính (Hop 2 → Hop 3 → Vector fallback) dựa trên đánh giá tính đầy đủ của ngữ cảnh |
| **Near-Dedup Implementation** | +3 | Bổ sung cơ chế lọc trùng lặp gần (MinHash/LSH, SimHash hoặc Embedding ANN) thay vì chỉ dùng exact hash |

---

## 🚫 Lỗi trừ điểm thủ tục

| Lỗi vi phạm | Mức phạt |
|-------------|----------|
| Hard-code API Key hoặc Password vào notebook nộp bài | **-10 điểm** (Nguy cơ bảo mật) |
| Cạnh trong Knowledge Graph bị thiếu `source_chunk_id` hoặc `published_date` | **-5 điểm** |
| Không xuất được 2 file CSV báo cáo kết quả | **-5 điểm** |
| Nộp thiếu file thuyết minh hoặc file reflection cá nhân | **-5 điểm/file** |

---

## 🏁 Quy trình Nộp bài

1. Hoàn thiện tất cả các cell code trong [`Day19_GraphRAG_vs_FlatRAG_Production_Lab_Guide.ipynb`](Day19_GraphRAG_vs_FlatRAG_Production_Lab_Guide.ipynb) (Đảm bảo chạy `Restart & Run All` thành công).
2. Kiểm tra 2 file báo cáo trong thư mục `outputs/`:
   - `outputs/graphrag_eval_results.csv`
   - `outputs/graphrag_vs_flatrag_summary.csv`
3. Hoàn thành các file báo cáo trong thư mục `reports/`:
   - `reports/technical_defense.md`
   - `reports/failure_analysis.md`
   - `reports/reflection_[HọTên].md`
4. Commit và push toàn bộ lên GitHub cá nhân, gửi đường link repo cho Giảng viên / Trợ giảng.
