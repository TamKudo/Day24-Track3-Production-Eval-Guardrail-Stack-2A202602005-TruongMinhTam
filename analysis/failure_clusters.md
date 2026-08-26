# Failure Cluster Analysis — Phase A

**Sinh viên:** Trương Minh Tâm — 2A202602005
**Ngày:** 2026-08-26

---

## 1. Aggregate RAGAS Scores theo Distribution

| Metric | factual | multi_hop | adversarial |
|---|---|---|---|
| faithfulness | 0.9083 | 0.4596 | 0.8000 |
| answer_relevancy | 0.8021 | 0.5984 | 0.6862 |
| context_precision | 0.9458 | 0.9417 | 0.9583 |
| context_recall | 0.8750 | 0.8708 | 0.6833 |
| **avg_score** | **0.8828** | **0.7176** | **0.7820** |

---

## 2. Bottom 10 Questions

| Rank | Distribution | Question | avg_score | worst_metric |
|---|---|---|---|---|
| 1 | multi_hop | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài... | 0.2500 | faithfulness |
| 2 | adversarial | Bao lâu phải đổi mật khẩu một lần? | 0.3125 | faithfulness |
| 3 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng... | 0.3750 | faithfulness |
| 4 | multi_hop | Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai... | 0.4167 | faithfulness |
| 5 | multi_hop | Nhân viên tạm ứng 4 triệu và một nhân viên khác tạm ứng 7 triệu... | 0.4167 | faithfulness |
| 6 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi... | 0.4167 | faithfulness |
| 7 | multi_hop | Nhân viên thử việc tháng thứ 3 phát hiện vi phạm bảo mật. Họ... | 0.4583 | faithfulness |
| 8 | factual | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | 0.5000 | faithfulness |
| 9 | multi_hop | Nhân viên tạm ứng 8 triệu, chưa thanh toán sau 30 ngày (quá hạn)... | 0.5000 | faithfulness |
| 10 | factual | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | 0.5977 | context_recall |

---

## 3. Failure Cluster Matrix

*(Mỗi ô = số câu có worst_metric = row, thuộc distribution = col)*

| worst_metric | factual | multi_hop | adversarial | Total |
|---|---|---|---|---|
| faithfulness | 3 | 14 | 2 | 19 |
| answer_relevancy | 12 | 3 | 1 | 16 |
| context_precision | 2 | 0 | 0 | 2 |
| context_recall | 3 | 3 | 7 | 13 |
| **Total** | **20** | **20** | **10** | **50** |

---

## 4. Dominant Failure Analysis

**Dominant distribution (theo script, tổng số câu có worst_metric bất kỳ):** factual
**Dominant metric (toàn corpus):** faithfulness

**Lý do phân tích:**

Script `cluster_analysis()` chọn "dominant distribution" bằng tổng số câu trong cột đó (mỗi câu luôn có đúng 1 worst_metric, nên tổng cột = count của distribution: factual=20, multi_hop=20, adversarial=10) — vì vậy con số này chỉ phản ánh cỡ mẫu, không phải "distribution nào tệ nhất". Nhìn vào avg_score thực tế (mục 1) và bottom-10 (mục 2), bức tranh khác hẳn: **multi_hop mới là distribution yếu nhất** (avg_score 0.7176, thấp nhất trong 3 nhóm) và chiếm 6/10 vị trí bottom-10, vì các câu multi-hop đòi hỏi tính toán/kết hợp nhiều policy (VD: phụ cấp + phép năm, tạm ứng + phạt trễ hạn) khiến model dễ tính sai số liệu → faithfulness thấp (0.4596). `faithfulness` là metric yếu nhất toàn corpus (14/20 câu multi_hop có faithfulness là worst_metric) vì model có xu hướng bịa hoặc tính sai khi phải tổng hợp nhiều con số từ nhiều đoạn chính sách khác nhau, thay vì chỉ trích dẫn nguyên văn như câu factual đơn giản.

---

## 5. Suggested Fixes

| Metric yếu | Root cause | Suggested fix |
|---|---|---|
| faithfulness | LLM hallucinating (đặc biệt khi phải tính toán multi-hop từ nhiều con số chính sách) | Tighten system prompt yêu cầu show-work từng bước tính toán, lower temperature, thêm few-shot ví dụ tính đúng cho câu multi-hop |
| context_recall | Missing relevant chunks (đặc biệt adversarial — câu hỏi về policy có version conflict) | Thêm metadata filter theo phiên bản hiệu lực (v2024 > v2023) trước khi retrieve, hoặc BM25 hybrid để bắt đúng chunk hiện hành |
| context_precision | Too many irrelevant chunks | Add reranking mạnh hơn hoặc giảm top_k trước rerank (ít gặp trong lab này — chỉ 2/50 câu) |
| answer_relevancy | Answer doesn't match question (nhiều nhất ở factual — model trả lời đúng nhưng lạc trọng tâm câu hỏi) | Improve prompt template để bám sát câu hỏi gốc, tránh trả lời thừa/thiếu trọng tâm |

---

## 6. Nhận xét về Adversarial Distribution

So sánh avg_score: **factual (0.8828) > adversarial (0.7820) > multi_hop (0.7176)**. Điều thú vị là adversarial — vốn được thiết kế để "gài bẫy" (version conflict, negation trap) — lại KHÔNG phải distribution tệ nhất; multi_hop mới là distribution yếu nhất vì đòi hỏi tính toán nhiều bước. Tuy nhiên xét đúng mục tiêu bonus của Phase A (so sánh adversarial vs factual): **adversarial (0.7820) < factual (0.8828)** — đạt điều kiện bonus, xác nhận bộ test set 10 câu adversarial vẫn stress-test được pipeline tốt hơn baseline factual.

Về version conflict cụ thể: pipeline vẫn bị "nhầm" bởi v2023 vs v2024 — câu #2 trong bottom-10 ("Bao lâu phải đổi mật khẩu một lần?", adversarial, faithfulness=thấp) và câu #6 ("VPN cá nhân NordVPN khi WFH", adversarial) đều thuộc adversarial và rơi vào bottom-10 vì corpus chứa cả policy cũ/mới (mật khẩu v1/v2, VPN v1.3) khiến model có thể trích dẫn nhầm bản hết hiệu lực — context_recall của adversarial cũng thấp nhất trong 4 metric (0.6833), khớp với giả thuyết "thiếu chunk đúng phiên bản hiện hành" hơn là do model bịa hoàn toàn.
