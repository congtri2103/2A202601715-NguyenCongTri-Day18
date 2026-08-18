# Group Report — Lab 18: Production RAG

**Nhóm:** Nguyễn Công Trí
**Ngày:** 18/08/2026

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| [Tên bạn] | M1: Chunking | ☑ | 13/13 |
| [Tên bạn] | M2: Hybrid Search | ☑ | 5/5 |
| [Tên bạn] | M3: Reranking | ☑ | 5/5 |
| [Tên bạn] | M4: Evaluation | ☑ | 4/4 |
| [Tên bạn] | M5: Enrichment | ☑ | 10/10 |

*(Bài làm cá nhân — 1 người phụ trách toàn bộ 5 module. `pytest tests/ -v` → **37/37 test pass, 0 fail**. `python main.py` đã chạy thành công end-to-end, sinh cả `naive_baseline_report.json` và `ragas_report.json`.)*

## Kết quả RAGAS

| Metric | Naive | Production | Δ |
|--------|-------|-----------|---|
| Faithfulness | 0.7905 | 0.6167 | -0.1738 |
| Answer Relevancy | 0.7510 | 0.6866 | -0.0644 |
| Context Precision | 0.9250 | 0.9667 | +0.0417 |
| Context Recall | 0.9250 | 0.8333 | -0.0917 |

## Key Findings

1. **Biggest improvement:** Context Precision tăng +0.0417 (0.925 → 0.967) nhờ M3 Reranking — CrossEncoder lọc bỏ context không liên quan trước khi đưa vào prompt, giúp tỷ lệ context "đúng trọng tâm" cao hơn hẳn so với dense-only baseline.

2. **Biggest challenge:** Faithfulness giảm mạnh nhất (-0.1738), nghịch lý vì context đã sạch hơn (precision cao hơn) nhưng model lại "trung thành" với context kém hơn. Nguyên nhân chính tìm được qua failure analysis: **corpus có các cặp tài liệu 2 phiên bản** (password v1/v2, nghỉ phép v2023/v2024), và pipeline production hiện chưa có cơ chế ưu tiên phiên bản hiện hành — khi cả 2 version cùng được retrieve, model dễ trộn lẫn số liệu cũ/mới, bị RAGAS chấm là hallucinating dù thực chất là lỗi thiếu metadata filtering, không phải model bịa thông tin từ hư không.

3. **Surprise finding:** 3/5 câu trong bottom-5 failures đều rơi đúng vào các cặp tài liệu có phiên bản song song — không phải lỗi ngẫu nhiên rải rác, mà là 1 root cause hệ thống lặp lại nhiều lần. Điều này cho thấy retrieval (M2) và reranking (M3) đang hoạt động đúng thiết kế (context_precision/recall vẫn ở mức cao 0.83–0.97), nhưng thiếu 1 tầng xử lý version ở giữa retrieval và generation — đúng như cảnh báo ban đầu trong README của lab.

## Presentation Notes (5 phút)

1. **RAGAS scores (naive vs production):** Production thắng ở Context Precision (+0.04) nhờ reranking, nhưng thua ở 3/4 metric còn lại — đặc biệt Faithfulness (-0.17). Đây không phải "production luôn tốt hơn" một cách hiển nhiên — cần phân tích sâu để biết tại sao.

2. **Biggest win — module nào, tại sao:** M3 Reranking là điểm sáng rõ nhất — CrossEncoder (`bge-reranker-v2-m3`) đọc kỹ cặp query–document, loại bỏ context không liên quan mà dense/BM25 retrieval ban đầu lẫn vào, thể hiện qua context_precision tăng lên gần như tối đa (0.9667).

3. **Case study — 1 failure, Error Tree walkthrough:** Câu "Bao lâu phải đổi mật khẩu một lần?" (faithfulness = 0.0, tệ nhất). Output sai → Context nhiều khả năng đúng (context_precision toàn cục 0.97, cả 2 chunk password v1/v2 khả năng đều được retrieve) → Query không mơ hồ, không cần rewrite → Root cause nằm ở generation: model không có rule ưu tiên phiên bản hiện hành khi context chứa cả 2 version xung đột. Fix: thêm metadata `status: current/superseded` + prompt rule chọn bản mới nhất.

4. **Next optimization nếu có thêm 1 giờ:** (1) Sửa M4 lưu `answer` + `contexts` đầy đủ cho từng câu trong `failures` để debug chính xác hơn thay vì suy luận gián tiếp qua ground_truth. (2) Thêm metadata version/status vào M1/M5, dùng để filter ở M2 — loại chunk phiên bản cũ trừ khi câu hỏi hỏi về lịch sử/so sánh. (3) Viết thêm 3-5 câu test_set dạng "so sánh chính sách cũ và mới" để đo riêng khả năng version-handling, tách bạch khỏi lỗi retrieval thông thường.