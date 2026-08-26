# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Trương Minh Tâm — 2A202602005
**Ngày:** 2026-08-26

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~39.7ms P95)
[Presidio PII Scan]
    │ block if: VN_CCCD / VN_PHONE / EMAIL detected
    │ action:   return 400 + "PII detected in query"
    ▼ (~1259.7ms P95)
[NeMo Input Rail]
    │ block if: off-topic / jailbreak / prompt injection
    │ action:   return 503 + refuse message
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Search → M3 Rerank → GPT-4o-mini
    ▼
[NeMo Output Rail]
    │ flag if:  PII in response / sensitive content
    │ action:   replace with safe response
    ▼
User Response
```

---

## Latency Budget

*(Đo từ Task 12 — measure_p95_latency() trên 10 adversarial inputs, xem reports/guard_results.json — số liệu dưới đây là lần chạy chính thức cuối cùng, khớp với file JSON)*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | 25.17 | 39.68 | 39.68 | <10ms |
| NeMo Input Rail | 684.94 | 1259.73 | 1259.73 | <300ms |
| RAG Pipeline | (không đo trong lab — Day 18 pipeline không nằm trong scope Task 12) | | | <2000ms |
| NeMo Output Rail | (không đo riêng — Task 12 chỉ đo input rail theo skeleton) | | | <300ms |
| **Total Guard (Presidio + NeMo input)** | 710.10 | **1297.52** | 1297.52 | **<500ms** |

**Budget OK?** [x] No

**Comment:** Cả 2 layer đều vượt budget riêng lẻ đã đề ra trong bảng gốc (Presidio 39.7ms > 10ms; NeMo 1259.7ms >> 300ms), và tổng P95 (1297.5ms) vượt xa ngưỡng 500ms — dù chạy lại nhiều lần, con số dao động (đo được từ ~1300ms đến ~1830ms tùy lần) nhưng luôn vi phạm nghiêm trọng budget, nên kết luận "vượt budget" là ổn định, không phải do nhiễu đo đạc. Bottleneck rõ ràng là **NeMo input rail** — chiếm ~95% tổng latency vì mỗi request gọi `self_check_input` qua gpt-4o-mini (1 LLM round-trip), cộng thêm 1 round-trip sinh câu trả lời chính (`generate_bot_message`) khi input được allow. Presidio cũng cao hơn kỳ vọng ban đầu (<10ms) vì dùng NER-based `PERSON` recognizer (spaCy `en_core_web_lg`) chạy song song với các regex recognizer — có thể tối ưu bằng cách tắt các recognizer không cần thiết (chỉ giữ VN_CCCD/VN_PHONE/EMAIL, loại spaCy NLP entirely) để đưa Presidio về đúng tầm regex-only (<10ms). Với NeMo, cách giảm latency thực tế: dùng streaming để trả lời sớm phần không nhạy cảm, hoặc thay self_check_input bằng một classifier nhỏ/rẻ hơn (fine-tuned nhỏ, hoặc heuristic + fallback LLM chỉ khi heuristic không chắc) thay vì luôn gọi full LLM.

---

## CI/CD Gates (phải pass trước khi merge to main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải ≥ 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 total < 500ms
```

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale NeMo model |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | factual 0.886 / multi_hop 0.712 / adversarial 0.782 |
| Worst metric | faithfulness (dominant failure metric toàn bộ 50q) |
| Dominant failure distribution | factual |
| Cohen's κ | 0.2857 (fair — thấp hơn nhiều so với ngưỡng substantial 0.6; xem `analysis/bias_report.md`) |
| Adversarial pass rate | 20 / 20 (100%) |
| Guard P95 latency | 1297.52 ms (Presidio 39.68ms + NeMo 1259.73ms) |

---

## Nhận xét & Cải tiến

RAGAS cho thấy pipeline Day 18 yếu nhất ở **faithfulness** — mô hình có xu hướng bịa hoặc dùng nhầm chính sách cũ (v2023) khi corpus chứa cả bản hết hiệu lực, đúng như thiết kế bộ test 3-distribution dự đoán; cần thêm metadata filter theo phiên bản hiệu lực trước khi retrieve, không chỉ dựa vào reranking. LLM-as-judge với swap-and-average cho thấy judge khá ổn định về vị trí (0% position bias trên mẫu nhỏ) nhưng κ=0.2857 (fair, thấp hơn nhiều so với ngưỡng substantial 0.6) cho thấy judge KHÔNG đủ tin cậy để làm gate tự động không giám sát — phân tích chi tiết cho thấy judge có thiên hướng chấm dễ dãi khi thiếu ground_truth để đối chiếu số liệu cụ thể (xem `analysis/bias_report.md`); bắt buộc phải kết hợp spot-check người với judge, đặc biệt cho câu multi-hop/adversarial dễ gây tranh cãi. Sau khi sửa lỗi cấu hình gốc trong `rails.co` (dialog-flow kiểu `user ask X` không hoạt động như input rail — phải chuyển sang `self_check_input`/`self_check_output` action-based flow với prompt tùy chỉnh theo domain HR tiếng Việt), guard stack đạt 20/20 (100%) trên bộ adversarial, vượt cả mức bonus 18/20. Tuy nhiên P95 latency (1832ms) vượt xa ngân sách 500ms đề ra — đây là đánh đổi thực tế giữa độ chính xác (LLM-based self-check bắt được jailbreak/off-topic tinh vi tốt hơn heuristic) và tốc độ; nếu deploy production thật, tôi sẽ tách guard stack thành 2 tầng: heuristic/regex nhanh chặn các trường hợp rõ ràng trước, chỉ escalate lên LLM self-check khi heuristic không chắc, đồng thời cache kết quả check cho các câu hỏi lặp lại.
