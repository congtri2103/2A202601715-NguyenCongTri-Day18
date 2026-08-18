# Individual Reflection — Lab 18

**Tên:** Nguyễn Công Trí
**Module phụ trách:** M1, M2, M3, M4, M5 (bài cá nhân)

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** Cả 5 module — M1 Chunking, M2 Hybrid Search, M3 Reranking, M4 Evaluation, M5 Enrichment.
- **Các hàm/class chính đã viết:**
  - M1: `chunk_semantic` (gom câu theo cosine similarity), `chunk_hierarchical` (parent-child với `parent_id` link đúng), `chunk_structure_aware` (chunk theo markdown header, giữ `section` metadata).
  - M2: `segment_vietnamese` (tách từ tiếng Việt qua underthesea), `BM25Search.index/search`, `DenseSearch.index/search` (Qdrant), `reciprocal_rank_fusion` (gộp rank BM25 + dense).
  - M3: `CrossEncoderReranker.rerank` (sort theo score giảm dần, trả top-k), `FlashrankReranker` (lựa chọn phụ).
  - M4: `evaluate_ragas` (wrap try/except, fallback về 0 khi thiếu key), `failure_analysis` (diagnostic tree map metric → diagnosis/fix, sort bottom-N theo avg score).
  - M5: 4 kỹ thuật enrichment (summarize, HyQA, contextual prepend, auto metadata) đều có fallback extractive khi không có API key; `_enrich_single_call` cho combined mode (1 call/chunk) cũng có fallback tương tự để không làm pipeline vỡ.
- **Số tests pass:** 37/37 (M1: 13/13, M2: 5/5, M3: 5/5, M4: 4/4, M5: 10/10)

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Reciprocal Rank Fusion (RRF) — thay vì cộng trực tiếp điểm BM25 và dense score (2 thang đo khác nhau, không so sánh được), RRF chỉ dùng **thứ hạng** (`1/(k+rank+1)`) để gộp — công thức đơn giản nhưng giải quyết đúng vấn đề "khác thang đo" một cách rất gọn.
- **Điều bất ngờ nhất:** Reranking (M3) làm tăng `context_precision` (+0.04) nhưng ngược lại `faithfulness` lại **giảm** (-0.17) so với baseline. Ban đầu nghĩ context tốt hơn thì output phải tốt hơn theo, nhưng thực tế root cause nằm ở chỗ khác hoàn toàn: corpus có các cặp tài liệu 2 phiên bản (password v1/v2, nghỉ phép v2023/v2024), và khi cả 2 phiên bản cùng lọt vào context (dù đã qua reranking), model không có cơ chế chọn đúng bản hiện hành — dẫn tới câu trả lời bị RAGAS chấm là "không trung thành với context" dù bản chất là thiếu metadata filtering, không phải model bịa thông tin.
- **Kết nối với bài giảng:** Đúng với phần "Corpus có cả chính sách cũ và mới... đừng bỏ metadata source, tiêu đề/section hoặc thông tin phiên bản" mà bài học đã cảnh báo ngay từ đầu — lúc đọc lý thuyết thấy hiển nhiên, nhưng chỉ khi thấy 3/5 bottom-failures đều rơi đúng vào bẫy này mới thực sự "thấm" được tại sao lưu ý đó quan trọng.

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** Không phải lỗi logic trong code, mà là chuỗi vấn đề môi trường liên tiếp: (1) máy cài Python 3.14 quá mới khiến `onnxruntime` (dependency của `flashrank`) không có bản build tương thích, dependency resolver của pip chạy rất lâu rồi báo `ResolutionImpossible`; (2) macOS Ventura (13) không được Docker Desktop hỗ trợ chính thức nữa; (3) sau khi chuyển sang Colima, container Qdrant từ 1 lab khác (Lab 17) đang chiếm sẵn port 6333, gây lỗi "port is already allocated"; (4) model reranker `bge-reranker-v2-m3` nặng ~2.27GB, mạng chậm khiến việc tải mất hàng chục phút, ban đầu tưởng nhầm là code bị treo/lỗi logic.
- **Cách giải quyết:**
  1. Tạo lại virtualenv với Python 3.12 (bản ổn định hơn, tương thích đầy đủ dependency) thay vì cố ép chạy trên 3.14.
  2. Chuyển sang dùng Colima (`brew install docker docker-compose colima && colima start`) thay cho Docker Desktop vì Ventura không được hỗ trợ.
  3. Đổi port host trong `docker-compose.yml` (6333→6343, 6334→6344) để không đụng container Lab 17, đồng bộ lại `QDRANT_PORT` trong `config.py`.
  4. Xác nhận việc tải model không phải bug bằng cách kiểm tra `du -sh` trên thư mục cache HuggingFace — thấy dung lượng tăng dần theo thời gian, tức đang tải thật, không phải treo; kiên nhẫn để chạy nền và tận dụng cache cho lần chạy sau.
  5. Dùng OpenRouter thay vì OpenAI key trực tiếp — phải sửa code M5 để nhận `base_url` tuỳ chỉnh (`OpenAI(base_url=os.getenv("OPENAI_BASE_URL"))`) vì code gốc chỉ hard-code client mặc định trỏ OpenAI.
- **Thời gian debug:** Ước tính khoảng 1.5–2 giờ dành cho việc setup môi trường (Python version, Docker, port conflict, model download) — nhiều hơn cả thời gian viết code cho 5 module. Đây là bài học riêng ngoài phần kỹ thuật RAG.

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Kiểm tra Python version và cài virtualenv với bản Python được lab yêu cầu (3.11+, không quá mới) **ngay từ đầu** trước khi `pip install`, thay vì để dependency resolver chạy rất lâu rồi mới phát hiện xung đột. Cũng nên chạy bước "pre-download models" trong README **trước tiên** (song song lúc đọc đề bài) thay vì để nó chặn giữa lúc chạy test.
- **Module nào muốn thử tiếp:** M4 — muốn mở rộng `failure_analysis()` để lưu lại `answer` và `contexts` đầy đủ cho từng câu fail (hiện tại chỉ lưu `diagnosis` cấp cao), giúp debug chính xác hơn thay vì phải suy luận gián tiếp qua `ground_truth` như đã làm ở phần failure analysis của lab này.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Teamwork | N/A (bài cá nhân) |
| Problem solving | 5 |