# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Trần Văn Toàn  
**Ngày:** 30/06/2026

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~14.5ms P95)
[Presidio PII Scan]
    │ block if: VN_CCCD / VN_PHONE / EMAIL detected
    │ action:   return 400 + "PII detected in query"
    ▼ (~940.6ms P95)
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

*(Điền từ kết quả Task 12 — measure_p95_latency())*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | 13.50 | 14.48 | 14.48 | <10ms |
| NeMo Input Rail | 704.86 | 940.59 | 940.59 | <300ms |
| RAG Pipeline | 1200.00 | 1800.00 | 2200.00 | <2000ms |
| NeMo Output Rail | 700.00 | 940.00 | 940.00 | <300ms |
| **Total Guard** | 718.36 | **954.42** | 954.42 | **<500ms** |

**Budget OK?** [ ] Yes / [x] No  
**Comment:** Cả Presidio (14.48ms P95) và NeMo Input Rail (940.59ms P95) đều vượt ngân sách độ trễ cho phép (<10ms và <300ms tương ứng). NeMo Input Rail sử dụng mô hình LLM thông qua API gọi mạng (gpt-4o-mini) nên bị ảnh hưởng lớn bởi độ trễ mạng và thời gian sinh của mô hình. Để tối ưu hóa độ trễ xuống dưới 500ms, ta có thể thay thế bước gọi LLM bằng một mô hình phân loại nhỏ chạy trực tiếp local (ví dụ: DeBERTa fine-tuned) hoặc triển khai giải pháp vLLM nội bộ với kỹ thuật Speculative Decoding.

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
| RAGAS avg_score (50q) | 0.8056 |
| Worst metric | faithfulness |
| Dominant failure distribution | factual |
| Cohen's κ | 0.074 |
| Adversarial pass rate | 20 / 20 |
| Guard P95 latency | 954.42 ms |

---

## Nhận xét & Cải tiến

> Hệ thống Guardrail Stack hoạt động rất tốt trong việc ngăn chặn các hành vi tấn công mã độc, PII và các câu hỏi ngoài phạm vi (đạt tỉ lệ chặn tuyệt đối 20/20 trong Adversarial Suite).
> Tuy nhiên, hệ thống gặp điểm hạn chế nghiêm trọng về mặt Latency do cuộc gọi LLM qua mạng của NeMo Guardrails kéo dài tổng P95 lên đến hơn 950ms.
> Thêm vào đó, điểm số Cohen's κ đạt mức khá thấp (0.074) cho thấy sự lệch pha lớn giữa đánh giá từ nhãn người dùng (human labels) và nhãn tự động từ LLM Judge đối với tập dữ liệu nhỏ.
> Nếu đưa hệ thống này lên môi trường production thực tế, tôi sẽ thay thế NeMo Guardrails LLM bằng một mô hình phân loại cục bộ chạy trên GPU (như SetFit hoặc BERT fine-tuned) để nén độ trễ đầu vào xuống dưới 50ms, đồng thời xây dựng bộ dữ liệu đánh giá RAG chất lượng cao hơn để tối ưu độ tương thích của LLM Judge.
