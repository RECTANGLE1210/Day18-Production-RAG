# Individual Reflection — Lab 18

**Tên:** Nguyễn Công Hùng  
**Module phụ trách:** M1–M5 (bài cá nhân, triển khai và tích hợp end-to-end)

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** M1 Advanced Chunking, M2 Hybrid Search, M3 Cross-Encoder Reranking, M4 RAGAS Evaluation/Failure Analysis và M5 Enrichment.
- **Các hàm/class chính đã viết:**
  - **M1:** `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()`; giữ `parent_id`, `section` và source metadata.
  - **M2:** `segment_vietnamese()`, `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()`; dùng BGE-M3 + Qdrant và giữ payload text/metadata.
  - **M3:** `CrossEncoderReranker._load_model()` và `rerank()`; lazy-load/cache `BAAI/bge-reranker-v2-m3`, batch predict và sort theo rerank score.
  - **M4:** RAGAS 4 metrics, OpenRouter judge LLM + local BGE embeddings, safe fallback và Diagnostic Tree cho bottom failures.
  - **M5:** summarize, HyQA, contextual prepend, metadata extraction và combined enrichment 1 call/chunk qua OpenRouter; giữ `original_text` và bảo vệ provenance metadata.
- **Số tests pass:** Chưa có log `pytest tests/ -v` tổng hợp trong output hiện tại; tuy nhiên `python main.py` đã chạy end-to-end thành công qua baseline, production, RAGAS và report generation. Cần chạy final test suite trước khi nộp.

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Một production RAG không chỉ là embedding + vector search. Chất lượng phụ thuộc vào cả chunking, sparse+dense retrieval, rank fusion, cross-encoder reranking, provenance/version metadata, generation prompt và evaluation theo từng failure mode.
- **Điều bất ngờ nhất:** Production pipeline không tốt hơn baseline ở mọi metric. Context Precision tăng **0.9250 → 0.9458**, nhưng Faithfulness giảm **0.8238 → 0.5167**, Answer Relevancy giảm **0.7408 → 0.5367** và Context Recall giảm **0.9000 → 0.7833**. Điều này cho thấy tăng độ phức tạp không bảo đảm chất lượng nếu source version, recall hoặc generation chưa được kiểm soát.
- **Kết nối với bài giảng:**
  - M1 — “Chunk đúng ngữ cảnh, không chỉ cắt đủ ký tự”: semantic/hierarchical/structure-aware chunking.
  - M2 — “Hybrid Search: BM25, Dense và RRF”: dùng rank fusion thay vì cộng trực tiếp hai loại score.
  - M3 — “Rerank top ứng viên”: retriever lấy rộng, cross-encoder đọc query-document kỹ hơn để chọn context cuối.
  - M4 — RAGAS + Failure Analysis: aggregate score chỉ là điểm bắt đầu; cần Error Tree theo từng câu.
  - M5 — Enrichment có kiểm soát: enrichment phải giữ raw text/provenance và cân bằng chất lượng với API cost/latency.

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:**
  1. Thiết lập dependency trong môi trường dùng chung có conflict version.
  2. Docker/Qdrant từng không pull được image do lỗi lấy OAuth token từ Docker Hub.
  3. Download model Hugging Face lớn (`bge-m3`, `bge-reranker-v2-m3`) bị timeout/connection reset và vấn đề compatibility quanh tokenizer/processor.
  4. Refactor LLM sang OpenRouter nhưng vẫn phải giữ cùng model và tương thích với OpenAI-compatible client/RAGAS.
  5. Kết quả production cuối cùng kém baseline ở generation metrics dù retrieval precision tốt hơn.
- **Cách giải quyết:**
  - Tách lỗi environment khỏi lỗi algorithm, kiểm tra từng layer trước khi sửa code.
  - Dùng cache/resume và cấu hình timeout phù hợp cho Hugging Face thay vì xóa cache/tải lại từ đầu.
  - Giữ Qdrant làm dense store theo scaffold và chỉ debug network/container riêng.
  - Tạo một configuration authority cho OpenRouter (`OPENROUTER_API_KEY`, base URL, model, `get_llm_client()`), không reintroduce OpenAI key trực tiếp.
  - Với kết quả RAGAS, không “tối ưu theo cảm giác”; dùng bottom failures để nhận ra các pattern version conflict, numeric range và multi-hop.
- **Thời gian debug:** Không đo riêng. Run end-to-end cuối mất **914.9s (~15.2 phút)**; riêng M5 enrichment mất **503.5s**, cho thấy latency/cost của enrichment là phần cần quan sát kỹ.

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:**
  1. Thiết lập virtual environment sạch ngay từ đầu để tránh dependency conflict giữa các lab.
  2. Thêm logging/tracing per query từ đầu: BM25 results, dense results, RRF rank, rerank score, final contexts, final answer và source/version.
  3. Chạy ablation sau mỗi module (`baseline → +M1 → +M2 → +M3 → +M5`) thay vì chỉ so baseline với full production. Như vậy sẽ xác định chính xác module nào làm metric tăng/giảm.
  4. Thiết kế metadata version/status như first-class retrieval signal vì corpus cố tình có policy cũ/mới.
  5. Chỉ chạy full enrichment sau smoke sample vì 125 calls mất hơn 8 phút.
- **Module nào muốn thử tiếp:** M2/M4. Tôi muốn thêm version-aware retrieval/reranking và mở rộng report để lưu per-question traces, sau đó dùng failure analysis để kiểm chứng từng thay đổi bằng RAGAS thay vì chỉ nhìn aggregate.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Teamwork | 4 *(bài cá nhân: đánh giá khả năng giữ contract và tích hợp các module)* |
| Problem solving | 5 |

### Action plan

- Chạy `pytest tests/ -v` và `python check_lab.py` trước submission.
- Bổ sung per-question answer/context/source vào report phục vụ Error Tree.
- Thực hiện ablation M5 và version-aware retrieval.
- Chỉ giữ thay đổi nếu metric trên test set tăng và không làm regression các câu policy-version/boundary.
