# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Cá nhân — Nguyễn Công Hùng  
**Thành viên:** Nguyễn Công Hùng → M1–M5

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------:|-----------:|---:|
| Faithfulness | 0.8238 | 0.5167 | -0.3071 |
| Answer Relevancy | 0.7408 | 0.5367 | -0.2042 |
| Context Precision | 0.9250 | 0.9458 | +0.0208 |
| Context Recall | 0.9000 | 0.7833 | -0.1167 |

**Nhận xét nhanh:** Production retrieval tăng nhẹ Context Precision (+0.0208), nhưng Faithfulness, Answer Relevancy và Context Recall đều giảm so với naive baseline. Điều này cho thấy việc lấy được context “sạch” hơn chưa đủ để bảo đảm câu trả lời cuối đúng và bám sát bằng chứng.

**Giới hạn của report hiện tại:** `ragas_report.json` chỉ lưu aggregate và failure summary, không lưu answer/context theo từng câu. Vì vậy các mục **Got** và nhánh “Context đúng?” bên dưới được ghi là *chưa được persist*, thay vì suy đoán câu trả lời mà hệ thống đã sinh ra.

## Bottom-5 Failures

### #1
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** Không được lưu trong `ragas_report.json`; RAGAS ghi nhận `faithfulness = 0.0`.
- **Worst metric:** Faithfulness = 0.0.
- **Error Tree:** Output sai/không được support → Context đúng? **Chưa xác minh vì report không lưu context** → Query là câu hỏi ngưỡng số tiền → cần kiểm tra retrieval có lấy đúng rule “trên 50 triệu” hay lấy nhầm rule 5–50 triệu.
- **Root cause:** Theo Diagnostic Tree, lỗi trực tiếp là generation không bám bằng chứng. Một rủi ro cụ thể cần kiểm tra là xử lý ngưỡng số 55 triệu: nếu retriever đưa cả nhiều mức phê duyệt, LLM có thể chọn nhầm Director thay vì CEO.
- **Suggested fix:** Lưu trace `answer + contexts + source` theo từng query; thêm regression test cho các boundary 50/55 triệu; trong prompt yêu cầu trích đúng điều kiện số tiền trước khi kết luận người phê duyệt; dùng temperature thấp.

### #2
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Chính sách hiện hành v2.0 yêu cầu đổi mật khẩu mỗi **120 ngày**; v1.0 cũ là 90 ngày và đã bị thay thế.
- **Got:** Không được lưu trong `ragas_report.json`; RAGAS ghi nhận `faithfulness = 0.0`.
- **Worst metric:** Faithfulness = 0.0.
- **Error Tree:** Output sai/không được support → Context đúng? **Chưa persist** → Corpus có cả policy cũ và mới → kiểm tra M2/M5 có giữ `source/version/status` và ranking có ưu tiên v2.0 không.
- **Root cause:** Đây là failure có khả năng cao liên quan **version conflict**: corpus chứa cả rule 90 ngày và 120 ngày. Retrieval có thể lấy đúng chủ đề nhưng generation chọn thông tin từ phiên bản đã bị thay thế.
- **Suggested fix:** Parse/preserve metadata version + trạng thái “hiện hành/đã thay thế”, ưu tiên/filter policy hiện hành trước rerank hoặc generation; thêm test bắt buộc query này phải trả 120 ngày và không được dùng v1.0.

### #3
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo v2024: 15 ngày cơ bản + 3 ngày thâm niên = **18 ngày phép**; lương Senior (P3–P4) là **20–35 triệu VNĐ/tháng**.
- **Got:** Không được lưu trong `ragas_report.json`; RAGAS ghi nhận `faithfulness = 0.0`.
- **Worst metric:** Faithfulness = 0.0.
- **Error Tree:** Output sai/không được support → Context đúng? **Chưa persist** → Query cần ghép ít nhất hai mảnh bằng chứng: phép năm/thâm niên và bảng lương → kiểm tra top-k sau rerank có giữ đủ cả hai nguồn hay không.
- **Root cause:** Câu hỏi **multi-hop**. Chỉ cần một trong hai tài liệu bị rơi khỏi top-3 rerank hoặc model trộn policy cũ/mới là answer không còn faithful.
- **Suggested fix:** Với query đa ý, tách retrieval thành sub-query hoặc bảo đảm top-k chứa evidence từ cả leave policy và salary table; prompt yêu cầu trả từng phần dựa trên source tương ứng; thêm regression test kiểm tra cả `18 ngày` và `20–35 triệu`.

### #4
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Nghỉ 16–30 ngày cần **Giám đốc điều hành (CEO)** phê duyệt; nghỉ trên 14 ngày không lương thì nhân viên phải tự đóng phần bảo hiểm của mình.
- **Got:** Không được lưu trong `ragas_report.json`; RAGAS ghi nhận `faithfulness = 0.0`.
- **Worst metric:** Faithfulness = 0.0.
- **Error Tree:** Output sai/không được support → Context đúng? **Chưa persist** → Query là bài toán match khoảng `20 ∈ [16,30]` → kiểm tra chunk/rerank có giữ nguyên bảng/range và có đưa đúng rule vào top context không.
- **Root cause:** Khả năng lỗi ở việc diễn giải **numeric range** hoặc generation không bám đúng rule phê duyệt sau khi nhận nhiều policy nghỉ phép gần nhau.
- **Suggested fix:** Giữ bảng/range nguyên khối khi chunk; thêm test cho 15/16/20/30/31 ngày; prompt yêu cầu xác định khoảng trước rồi mới trả người phê duyệt; lưu retrieved context để xác nhận lỗi nằm ở retrieval hay generation.

### #5
- **Question:** Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?
- **Expected:** Theo policy v2024 hiện hành, từ **3 năm** thâm niên được cộng thêm 1 ngày cho mỗi 3 năm; policy v2023 cũ dùng mốc 5 năm.
- **Got:** Không được lưu trong `ragas_report.json`; RAGAS ghi nhận `faithfulness = 0.0`.
- **Worst metric:** Faithfulness = 0.0.
- **Error Tree:** Output sai/không được support → Context đúng? **Chưa persist** → Corpus có rule cũ 5 năm và rule mới 3 năm → cần kiểm tra source/version ở retrieval.
- **Root cause:** Tương tự failure #2, đây là **version conflict** rõ ràng. Semantic similarity cao giữa hai phiên bản khiến cả hai có thể cùng xuất hiện trong candidate list.
- **Suggested fix:** Version-aware retrieval/filtering; ưu tiên tài liệu có metadata “hiện hành”; reranker/generation prompt phải xử lý “đã thay thế”; thêm regression test bắt buộc trả mốc 3 năm.

## Case Study (cho presentation)

**Question chọn phân tích:** Bao lâu phải đổi mật khẩu một lần?

**Error Tree walkthrough:**
1. **Output đúng?** → RAGAS cho Faithfulness = 0.0, nên answer không được chứng minh bởi evidence mà evaluator nhận được.
2. **Context đúng?** → Chưa thể kết luận từ report hiện tại vì `save_report()` không persist per-question context. Corpus chắc chắn có cả v1.0 (90 ngày) và v2.0 hiện hành (120 ngày), nên đây là điểm cần trace đầu tiên.
3. **Query rewrite / retrieval OK?** → Query rất ngắn và đúng chủ đề; vấn đề đáng kiểm tra hơn là M2 có retrieve cả hai phiên bản và pipeline có cơ chế ưu tiên version hiện hành hay chưa.
4. **Fix ở bước:** Trước hết thêm trace per-query; sau đó thêm metadata/version-aware filtering hoặc reranking rule, và ràng buộc prompt “chỉ dùng policy hiện hành; nếu có tài liệu đã thay thế thì không dùng làm kết luận”.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Persist `question`, `answer`, top contexts, `source`, score BM25/dense/RRF/rerank cho từng query để Error Tree có bằng chứng thay vì suy đoán.
- Làm ablation `M1+M2+M3` với/không M5 để xác định enrichment có tạo noise hay không.
- Thêm version-aware metadata rule cho các cặp policy cũ/mới.
- Thêm regression tests cho numeric threshold/range và multi-hop questions.
