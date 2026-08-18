# Group Report — Lab 18: Production RAG

**Nhóm:** Cá nhân — Nguyễn Công Hùng  
**Ngày:** 18/08/2026

## Thành viên & Phân công

Bài được thực hiện cá nhân, nên một người triển khai và tích hợp toàn bộ M1–M5.

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| Nguyễn Công Hùng | M1: Advanced Chunking | ☑ | Chưa có log `pytest tests/ -v` tổng hợp trong output gửi kèm |
| Nguyễn Công Hùng | M2: Hybrid Search (BM25 + Dense + RRF) | ☑ | Chưa có log tổng hợp |
| Nguyễn Công Hùng | M3: Cross-Encoder Reranking | ☑ | Chưa có log tổng hợp |
| Nguyễn Công Hùng | M4: RAGAS Evaluation + Failure Analysis | ☑ | End-to-end RAGAS chạy thành công trên 20 câu |
| Nguyễn Công Hùng | M5: Enrichment (OpenRouter, combined 1-call/chunk) | ☑ | End-to-end enrich thành công 125/125 chunks |

## Kết quả RAGAS

| Metric | Naive | Production | Δ |
|--------|------:|-----------:|---:|
| Faithfulness | 0.8238 | 0.5167 | -0.3071 |
| Answer Relevancy | 0.7408 | 0.5367 | -0.2042 |
| Context Precision | 0.9250 | 0.9458 | +0.0208 |
| Context Recall | 0.9000 | 0.7833 | -0.1167 |

- Số câu đánh giá: **20** cho cả baseline và production.
- Production tạo **125 chunks từ 26 documents**.
- M5 enrich **125/125 chunks**, mất **503.5s**.
- Dense indexing mất **28.3s**.
- RAGAS production mất **71.3s**.
- Tổng `python main.py`: **914.9s (~15.2 phút)**.
- Hai PDF scan không có text layer được bỏ qua đúng boundary của repo.

## Key Findings

1. **Biggest improvement:**  
   `Context Precision` tăng từ **0.9250 → 0.9458 (+0.0208)**. Hybrid retrieval + reranking giúp context được trả về có độ tập trung cao hơn baseline dense-only.

2. **Biggest challenge:**  
   `Faithfulness` giảm mạnh **-0.3071** và `Answer Relevancy` giảm **-0.2042**. Bottom failures chủ yếu được Diagnostic Tree gán vào `faithfulness`, cho thấy retrieval tốt hơn chưa chuyển thành answer đáng tin cậy. Các câu có policy version cũ/mới, numeric threshold/range và multi-hop là nhóm rủi ro rõ nhất.

3. **Surprise finding:**  
   Production pipeline phức tạp hơn nhưng **không tự động tốt hơn baseline**. Context Precision tăng, trong khi Context Recall giảm từ 0.9000 xuống 0.7833 và generation metrics giảm mạnh. Đây là bằng chứng cần ablation từng module thay vì giả định thêm chunking/rerank/enrichment luôn cải thiện chất lượng.

## Presentation Notes (5 phút)

1. **RAGAS scores (naive vs production):**  
   - Faithfulness: 0.8238 → 0.5167  
   - Answer Relevancy: 0.7408 → 0.5367  
   - Context Precision: 0.9250 → 0.9458  
   - Context Recall: 0.9000 → 0.7833  
   Kết luận: retrieval precision tốt hơn nhưng answer quality giảm.

2. **Biggest win — module nào, tại sao:**  
   M2 + M3 là phần có tín hiệu tích cực rõ nhất vì Context Precision tăng. BM25 tiếng Việt + BGE-M3 dense được fuse bằng RRF, sau đó CrossEncoder rerank top candidates trước khi trả context cho LLM.

3. **Case study — 1 failure, Error Tree walkthrough:**  
   Query: **“Bao lâu phải đổi mật khẩu một lần?”**  
   Ground truth hiện hành là **120 ngày (v2.0)**, trong khi corpus vẫn chứa policy cũ **90 ngày (v1.0, đã thay thế)**. Failure có Faithfulness = 0.0. Vì report chưa persist answer/context theo từng câu, bước tiếp theo là trace retrieved sources để xác định model lấy nhầm version ở retrieval hay generation.

4. **Next optimization nếu có thêm 1 giờ:**  
   - Persist full per-question trace (`answer`, contexts, sources, retrieval/rerank scores).  
   - Thêm version-aware filtering/priority cho policy hiện hành.  
   - Làm ablation M5 vì enrichment chiếm 503.5s và production vẫn giảm faithfulness/relevancy.  
   - Thêm regression tests cho version conflict, numeric boundaries/ranges và multi-hop query.

## Kết luận

Run end-to-end đã hoàn thành và tạo đủ hai report. Tuy nhiên kết quả hiện tại cho thấy production pipeline cần được **debug theo evidence**, không chỉ tiếp tục thêm module. Ưu tiên tiếp theo là quan sát per-query trace và xử lý version provenance trước khi tối ưu thêm retrieval/generation.
