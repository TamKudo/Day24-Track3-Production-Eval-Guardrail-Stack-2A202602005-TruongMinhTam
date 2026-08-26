# LLM Judge Bias Report — Phase B

**Sinh viên:** Trương Minh Tâm — 2A202602005
**Ngày:** 2026-08-26
**Judge model:** gpt-4o-mini

---

## 1. Pairwise Judge Results

*(Chạy pairwise_judge() trên 5 cặp answers, xem `reports/judge_results.json`)*

| # | Question (tóm tắt) | Winner | Reasoning tóm tắt |
|---|---|---|---|
| 1 | Nhân viên được nghỉ bao nhiêu ngày phép năm? | A | A đúng (15 ngày, v2024); B sai vì dùng số liệu v2023 đã hết hiệu lực |
| 2 | Thưởng Tết tối thiểu là bao nhiêu? | B | B nêu rõ "1 tháng lương"; A chỉ có 1 từ, không đủ thông tin |
| 3 | Nhân viên thử việc có được nghỉ phép năm không? | A | A đúng theo luật lao động (thử việc không có phép năm); B sai |
| 4 | Mua thiết bị 55 triệu cần ai phê duyệt? | B | B nêu đúng lý do (vượt ngưỡng 50 triệu → CEO); A thiếu căn cứ |
| 5 | Manager dùng VPN cá nhân khi WFH được không? | B | B đúng theo policy VPN v1.3 (cấm VPN cá nhân); A sai |

---

## 2. Swap-and-Average Results

| # | Pass 1 Winner | Pass 2 Winner | Final | Position Consistent? |
|---|---|---|---|---|
| 1 | A | A | A | True |
| 2 | B | B | B | True |
| 3 | A | A | A | True |
| 4 | B | B | B | True |
| 5 | B | B | B | True |

**Position bias rate:** 0% (0/5 case NOT consistent)

---

## 3. Cohen's κ Analysis

**Human labels:** `human_labels_10q.json` (10 câu, 5 label=1, 5 label=0)
**Judge labels:** chạy `judge_single_answer()` trên đúng 10 `model_answer` tương ứng (xem `reports/judge_results.json` — số liệu dưới đây khớp với file JSON hiện tại).

| Question ID | Human Label | Judge Label | Agree? |
|---|---|---|---|
| 1  | 1 | 1 | ✅ |
| 5  | 0 | 1 | ❌ |
| 12 | 1 | 1 | ✅ |
| 21 | 1 | 1 | ✅ |
| 23 | 1 | 1 | ✅ |
| 29 | 0 | 1 | ❌ |
| 33 | 1 | 1 | ✅ |
| 41 | 0 | 1 | ❌ |
| 46 | 1 | 1 | ✅ |
| 50 | 0 | 0 | ✅ |

**Cohen's κ:** 0.2857
**Interpretation:** fair (0.2–0.4 theo thang Landis-Koch — thấp hơn nhiều so với ngưỡng substantial 0.6)

Accuracy thô là 7/10 = 70%, nhưng κ chỉ 0.2857 vì phải trừ đi phần đồng thuận có thể xảy ra ngẫu nhiên — đúng như điểm RUBRIC lưu ý: "accuracy 70% vẫn có thể κ thấp nếu lớp nhãn lệch" (ở đây judge gần như luôn chấm 1 — 9/10 câu — nên phần lớn "đồng thuận" là ăn may trên các câu human cũng chấm 1, chứ không phải judge phân biệt tốt case sai). Ba điểm bất đồng, đều theo cùng một mẫu hình:
- **Q5** (mua thiết bị 55 triệu — ai phê duyệt): model trả lời "Giám đốc phòng ban", human chấm sai (đúng phải là CEO vì vượt ngưỡng 50 triệu) nhưng judge chấm đúng (1) — judge không có ground_truth về ngưỡng phê duyệt nên không đối chiếu được.
- **Q29** (tạm ứng 8 triệu, phạt trễ hạn): model trả lời thiếu (chỉ nêu Trưởng phòng, phạt sai công thức) nhưng judge vẫn chấm đúng (1) — judge không tính lại số học nên bỏ sót lỗi phạt pro-rata.
- **Q41** (phép năm — dùng số liệu v2023 cũ): judge chấm đúng (1) nhưng human chấm sai (0) vì phát hiện model dùng chính sách hết hiệu lực.

Cả 3 case bất đồng đều là **judge chấm "đúng" (1) nhầm cho câu trả lời mà human đã xác minh là sai** — nghĩa là judge có thiên hướng dễ dãi (lenient bias) khi không có ground_truth/policy gốc để đối chiếu số liệu cụ thể, đặc biệt với các câu cần tính toán (ngưỡng phê duyệt, công thức phạt) hoặc cần biết phiên bản chính sách nào đang hiệu lực. Đây là hạn chế cấu trúc của cách chấm `judge_single_answer()` hiện tại (chỉ đưa câu hỏi + answer, không có ground_truth) — nên xem là điểm cần cải thiện trước khi tin cậy con số κ này.

---

## 4. Verbosity Bias

Trong 5 case pairwise (mục 1), vị trí câu đúng/dài đã được trộn chủ động giữa A và B để tránh nhiễu do thiết kế bộ test:
- A thắng + A dài hơn B: 2/5 case
- B thắng + B dài hơn A: 3/5 case
- **Verbosity bias rate:** 1.0 (100%)

**Kết luận:** Trong bộ 5 cặp demo này, câu trả lời **đúng** luôn tình cờ cũng là câu **dài hơn** (vì câu đúng phải nêu đủ căn cứ/số liệu, còn câu sai thường cụt hoặc thiếu chi tiết). Vì vậy verbosity_bias=1.0 ở đây **không tách bạch được** "LLM thích câu dài" khỏi "câu dài thường đúng hơn vì đủ thông tin" — đây là hạn chế cỡ mẫu nhỏ (n=5) của lab, không phải bằng chứng chắc chắn về verbosity bias thực sự. Để kết luận đáng tin, cần bộ test lớn hơn có cả case "sai nhưng dài dòng" (ví dụ answer B trong ASSIGNMENT.md: "theo nhiều nguồn...") để tách 2 hiệu ứng.

---

## 5. Nhận xét chung

- κ = 0.2857 (fair) thấp hơn nhiều so với ngưỡng substantial 0.6 — judge gpt-4o-mini KHÔNG đủ tin cậy để dùng làm gate CI/CD tự động không giám sát ở dạng hiện tại. Nguyên nhân chính không phải judge "kém" một cách ngẫu nhiên, mà là **thiên hướng chấm dễ dãi có hệ thống** (9/10 câu được chấm 1) khi thiếu ground_truth để đối chiếu số liệu — cả 3 case sai đều là judge chấm "đúng" nhầm.
- Bắt buộc phải bổ sung ground_truth (đáp án chuẩn từ policy, không chỉ câu hỏi) và metadata ngày hiệu lực chính sách vào prompt của `judge_single_answer()` trước khi dùng κ này làm căn cứ tin cậy — đây là thay đổi ưu tiên cao nhất trước khi đưa vào CI/CD.
- Position bias 0% trên mẫu 5 cặp pairwise — không đáng lo, nhưng cỡ mẫu nhỏ nên chưa thể kết luận chắc chắn cho toàn bộ 50 câu.
- Swap-and-average có ích về mặt quy trình (bắt được nếu judge đổi ý khi đảo vị trí) dù trong lần chạy này cả 5 cặp đều nhất quán — nghĩa là chưa quan sát được vụ position bias thực tế, nhưng cơ chế double-pass vẫn nên giữ trong production vì rủi ro luôn tồn tại và chi phí chạy thêm 1 pass thấp hơn nhiều so với rủi ro một quyết định sai lệch do vị trí.
- Trong production, nên dùng LLM judge như **tín hiệu hỗ trợ**, không phải chân lý tuyệt đối: kết hợp κ định kỳ với human spot-check bắt buộc (không tùy chọn) cho mọi câu multi_hop/adversarial — đây là nhóm câu judge sai nhiều nhất (Q5, Q29, Q41 đều thuộc dạng cần tính toán hoặc kiểm tra phiên bản chính sách).
- **Lưu ý về reproducibility:** κ đo được dao động giữa các lần chạy (một lần chạy trước đó ra 0.583, lần này ra 0.2857) vì `judge_single_answer()` gọi LLM không có `temperature=0` cố định và không cache — đây tự nó là một phát hiện quan trọng: một judge có kết quả không ổn định giữa các lần chạy thì càng không nên dùng làm gate tự động cứng nhắc trong CI/CD.
