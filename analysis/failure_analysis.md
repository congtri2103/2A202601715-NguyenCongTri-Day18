# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Nguyễn Công Trí
**Thành viên:** Nguyễn Công Trí

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.7905 | 0.6167 | -0.1738 |
| Answer Relevancy | 0.7510 | 0.6866 | -0.0644 |
| Context Precision | 0.9250 | 0.9667 | +0.0417 |
| Context Recall | 0.9250 | 0.8333 | -0.0917 |

**Nhận xét tổng quan:** Production pipeline cải thiện `context_precision` (+0.04, nhờ reranking lọc context sạch hơn) nhưng lại **giảm** ở 3/4 metric còn lại, đặc biệt `faithfulness` (-0.17). Đây là kết quả phản trực giác — retrieval tốt hơn nhưng generation lại kém trung thành hơn — được phân tích chi tiết ở phần Case Study bên dưới.

## Bottom-5 Failures

### #1
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0): 120 ngày. Chính sách cũ (v1.0) là 90 ngày nhưng đã bị thay thế.
- **Got:** Không có trong report hiện tại (M4 chưa lưu `answer` per-question ở bottom-5 — xem Root cause).
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng? *Không xác định được* (thiếu `contexts` trong report) → Query OK? Có khả năng cao — câu hỏi rõ ràng, không mơ hồ.
- **Root cause:** Ground truth yêu cầu **phân biệt rõ v1.0 (90 ngày, cũ) và v2.0 (120 ngày, hiện hành)**. `context_precision` của pipeline production ở mức cao (0.97 trung bình) nên nhiều khả năng cả 2 chunk (v1 và v2) đều được retrieve. Faithfulness = 0.0 gợi ý model có thể đã **trộn lẫn 2 phiên bản** hoặc chọn nhầm phiên bản cũ khi trả lời, khiến câu trả lời không khớp với bất kỳ đoạn context riêng lẻ nào theo cách RAGAS đánh giá.
- **Suggested fix:** (1) Thêm metadata `version` + `status: current/superseded` vào chunk từ M1/M5, dùng để filter hoặc rerank ưu tiên bản hiện hành. (2) Sửa prompt: bắt buộc model nêu rõ "theo chính sách hiện hành..." và chỉ dùng 1 phiên bản trừ khi câu hỏi yêu cầu so sánh. (3) Sửa M4 để lưu `answer` + `contexts` cho từng câu trong `failures`, không chỉ `diagnosis`, để lần debug sau xác nhận được đúng nguyên nhân thay vì suy luận gián tiếp.

### #2
- **Question:** Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?
- **Expected:** Theo v2024 (hiện hành): 3 năm/1 ngày phép thêm. v2023 (cũ): 5 năm.
- **Got:** Không có trong report hiện tại (cùng hạn chế như #1).
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng? Không xác định được → Query OK? Có, câu hỏi không mơ hồ.
- **Root cause:** Cùng pattern với #1 — đây là cặp tài liệu `nghi_phep_nam_v2023.md` / `nghi_phep_nam_v2024.md`. Khả năng cao model lấy nhầm con số "5 năm" (v2023) thay vì "3 năm" (v2024 hiện hành), hoặc nêu cả hai nhưng không rõ ràng đâu là bản đang áp dụng — bị RAGAS chấm là không trung thành với 1 nguồn context xác định.
- **Suggested fix:** Giống #1: gắn `version`/`status` metadata; cân nhắc **loại bỏ chunk phiên bản cũ khỏi index chính** (chỉ giữ lại nếu câu hỏi có từ khóa "trước đây", "cũ", "so sánh") thay vì để retriever tự chọn.

### #3
- **Question:** Mật khẩu phải có tối thiểu bao nhiêu ký tự?
- **Expected:** v2.0 (hiện hành): 12 ký tự. v1.0 (cũ): 8 ký tự, đã thay thế.
- **Got:** Không có trong report hiện tại.
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng? Không xác định được → Query OK? Có.
- **Root cause:** Lần thứ 3 liên tiếp rơi vào đúng cặp tài liệu password v1/v2 — củng cố giả thuyết version confusion là nguyên nhân hệ thống, không phải ngẫu nhiên. Đây là bằng chứng mạnh cho thấy **cả 3 case tệ nhất đều cùng 1 root cause**, không phải 3 lỗi riêng lẻ.
- **Suggested fix:** Bổ sung câu hỏi trong test_set kiểu "theo chính sách hiện hành..." để đo riêng khả năng chọn đúng version, tách biệt với lỗi retrieval thông thường.

### #4
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** Không có trong report hiện tại.
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng? Chưa xác định — cần xem contexts thực tế → Query OK? Có, câu hỏi rõ ràng, có số liệu cụ thể (55 triệu) cần model tự suy luận đúng ngưỡng phê duyệt (>50 triệu).
- **Root cause:** Khác 3 case trên — đây **không phải lỗi version**, mà là câu hỏi dạng numeric/reasoning (55 triệu > ngưỡng 50 triệu → CEO). Có khả năng: (a) model không thực hiện đúng phép so sánh số học đơn giản trên context, hoặc (b) context có nhiều ngưỡng phê duyệt khác nhau (theo cấp bậc) khiến model chọn nhầm người phê duyệt.
- **Suggested fix:** Cải thiện prompt với instruction rõ ràng hơn cho câu hỏi có điều kiện số học (VD: "so sánh số tiền với các ngưỡng trong context trước khi trả lời, nêu rõ ngưỡng nào được áp dụng"). Đây là vấn đề generation/prompt, không phải retrieval — đúng như `diagnosis` mà M4 đã gán.

### #5
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Nghỉ 16-30 ngày cần CEO phê duyệt. Lưu ý thêm: nghỉ trên 14 ngày không lương, nhân viên tự đóng bảo hiểm.
- **Got:** Không có trong report hiện tại.
- **Worst metric:** faithfulness (0.5 — cao hơn 4 câu trên, tức là chỉ sai một phần)
- **Error Tree:** Output sai một phần → Context đúng? Nhiều khả năng đúng (score 0.5 không phải 0.0) → Query OK? Có.
- **Root cause:** Ground truth có **2 phần thông tin**: (1) ai phê duyệt (CEO, theo mốc 16-30 ngày) và (2) lưu ý phụ về bảo hiểm khi nghỉ >14 ngày. Score 0.5 gợi ý model có thể chỉ trả lời đúng phần (1) mà bỏ sót phần (2), hoặc ngược lại — đây là lỗi **completeness** chứ không hoàn toàn là hallucination thuần tuý.
- **Suggested fix:** Prompt nên yêu cầu model liệt kê **tất cả điều khoản liên quan** tìm thấy trong context, không chỉ trả lời phần chính của câu hỏi — đặc biệt với câu hỏi có ràng buộc phụ (side conditions).

## Case Study (cho presentation)

**Question chọn phân tích:** "Bao lâu phải đổi mật khẩu một lần?" (#1 — điểm faithfulness thấp nhất, 0.0)

**Error Tree walkthrough:**
1. **Output đúng?** Không — faithfulness = 0.0 nghĩa là RAGAS đánh giá câu trả lời không được hỗ trợ đầy đủ bởi context đã cung cấp.
2. **Context đúng?** Chưa xác nhận được trực tiếp (hạn chế của report hiện tại), nhưng suy luận gián tiếp: `context_precision` trung bình toàn bộ pipeline là 0.97 (rất cao), nên khả năng cao **cả 2 chunk (mat_khau_v1.md và mat_khau_v2.md) đều nằm trong context được retrieve** — đây chính xác là tình huống mà README đã cảnh báo ngay từ đầu ("corpus có cả chính sách cũ và mới... đừng bỏ metadata version").
3. **Query rewrite OK?** Có — câu hỏi không mơ hồ, không cần rewrite.
4. **Fix ở bước:** Không phải M1 (chunking) hay M2 (retrieval) — vì context_precision cao chứng tỏ retrieval đang hoạt động tốt, thậm chí "quá tốt" khi trả về cả 2 phiên bản. Vấn đề nằm ở **generation/prompt**: pipeline hiện tại không có cơ chế ưu tiên "phiên bản hiện hành" khi có nhiều context version xung đột. Fix cần ở **prompt template** (thêm rule chọn version mới nhất) kết hợp **metadata filtering ở M2/M5** (đánh dấu chunk nào là `current` vs `superseded`).

**Nếu có thêm 1 giờ, sẽ optimize:**
- Sửa M4 để lưu `answer` và `contexts` đầy đủ cho từng câu trong `failures` (không chỉ `diagnosis` cấp cao) — hiện tại phải suy luận gián tiếp qua ground_truth vì thiếu dữ liệu answer thực tế, làm giảm độ chắc chắn của root cause analysis.
- Thêm field `version`/`status` vào metadata M1 chunking cho các cặp tài liệu cũ/mới, dùng metadata filter ở M2 để loại bớt chunk `superseded` trừ khi câu hỏi hỏi rõ về lịch sử/so sánh.
- Viết thêm 3-5 câu test_set dạng "so sánh chính sách cũ và mới" để đo riêng khả năng xử lý version, tách bạch với lỗi retrieval/generation thông thường.