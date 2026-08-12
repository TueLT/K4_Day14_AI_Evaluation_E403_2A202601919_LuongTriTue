# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Domain:** OrbitTech Store Customer Support

## Part 1 — Warm-up

### Exercise 1.1 — RAGAS Metric Thresholds

| Metric | Acceptable low-score scenario | Critical low-score scenario | Action required |
|---|---|---|---|
| Faithfulness | A deliberately brief escalation/refusal has few content words. | The answer invents policy, product, refund, or security facts. | Block release; require grounding evidence. |
| Answer Relevance | Question is vague and the answer asks a focused clarifying question. | Answer addresses another policy or ignores the requested action. | Fix routing/prompt; add regression case. |
| Context Recall | A narrow query needs only one of several expected details. | Required policy evidence is absent from retrieval. | Improve query, chunking, or retrieval depth. |
| Context Precision | Extra harmless chunks appear after the correct evidence. | Noise ranks before the decisive policy chunk. | Rerank and tune retriever. |
| Completeness | Customer explicitly asks for only one sub-question. | Safety steps, deadline, exception, or eligibility condition is omitted. | Add answer checklist and multi-document retrieval. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

**Câu 1:** Chấm cùng hai câu trả lời A/B ở hai condition: A trước/B sau và B trước/A sau; dùng cùng rubric, blind nguồn câu trả lời, lặp lại nhiều lần. So sánh điểm theo vị trí thay vì theo nội dung. Nếu câu ở vị trí đầu có điểm cao một cách nhất quán thì có position bias.

**Câu 2:** Rubric ưu tiên đúng, đầy đủ và có thể hành động; nêu rõ “không thưởng cho độ dài” và phạt chi tiết không được hỗ trợ. Giới hạn độ dài hoặc chuẩn hóa hai câu trả lời trước khi chấm.

**Câu 3:** Human labels cho biết judge có khớp chuẩn chất lượng thật hay không, phát hiện bias hệ thống và giúp hiệu chỉnh rubric/threshold trước khi dùng làm quality gate.

### Exercise 1.3 — Evaluation trong CI/CD

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Chính sách sai hoặc bịa trong customer support là rủi ro cao. |
| Answer Relevance | 0.70 | Cần trả lời đúng ý; câu hỏi mơ hồ có thể hợp lệ khi yêu cầu làm rõ. |
| Completeness | 0.75 | Phải giữ các điều kiện, deadline và bước tiếp theo quan trọng. |

Chạy offline benchmark với mỗi thay đổi code/prompt/retrieval và trước release. Dùng online evaluation để theo dõi traffic thật, drift, latency và feedback. Human review dùng cho safety/privacy, case điểm sát threshold, calibration và thay đổi chính sách.

## Part 2 — Core Coding

Hoàn tất trong `template.py` và `solution/solution.py`: data models, năm metrics, retrieval wiring, LLM judge, benchmark/report/regression, failure analyzer và reranking bonus.

**Verification:** `py -3.13 -m pytest tests -v` → **42 passed**.

## Part 3 — Golden Dataset & Real Benchmark

### Exercise 3.1 — Build the Golden Dataset

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

| ID | Difficulty | Source document(s) | Lý do |
|---|---|---|---|
| E03 | Easy | 04_shipping_and_delivery.md | Tra cứu một fact rõ ràng: thời gian giao standard. |
| H01 | Hard | 09_escalation_and_policy_updates.md | Cần áp dụng effective date, order date và điều kiện OrbitPlus. |
| A02 | Adversarial | 00_system_scope.md | Prompt injection yêu cầu tiết lộ prompt/credentials cần bị từ chối an toàn. |

Điểm khó nhất là viết expected answer vừa đủ điều kiện chính sách nhưng không thêm suy diễn ngoài corpus, đồng thời giữ evidence là chuỗi nguyên văn trong đúng tài liệu.

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có question trùng ý và không dùng kiến thức ngoài corpus.
- [x] `py -3.13 validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Đã chạy full pipeline bằng Gemini thật với checkpoint. Do quota theo model, 11 câu đầu dùng `gemini-3.6-flash` và 9 câu còn lại dùng `gemini-3.1-flash-lite`; artifact ghi model của từng answer và không đọc `expected_answer` khi sinh câu trả lời.

| ID | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | .857 | 1.000 | .857 | .556 | 1.000 | .804 | Yes | - |
| E02 | .875 | 1.000 | .625 | 1.000 | 1.000 | .875 | Yes | - |
| E03 | 1.000 | 1.000 | .355 | .714 | 1.000 | .690 | No | off_topic |
| E04 | 1.000 | 1.000 | .400 | .600 | 1.000 | .667 | No | off_topic |
| E05 | .714 | 1.000 | .556 | .625 | .929 | .703 | Yes | - |
| M01 | 1.000 | .887 | .722 | .333 | .960 | .672 | No | off_topic |
| M02 | 1.000 | 1.000 | .441 | .889 | .933 | .754 | No | off_topic |
| M03 | .857 | 1.000 | .529 | .769 | .786 | .695 | Yes | - |
| M04 | 1.000 | .950 | .543 | .615 | .950 | .703 | Yes | - |
| M05 | 1.000 | 1.000 | .818 | .571 | 1.000 | .797 | Yes | - |
| M06 | .842 | .700 | .342 | .667 | .526 | .512 | No | off_topic |
| M07 | .938 | 1.000 | .919 | .571 | .750 | .747 | Yes | - |
| H01 | .786 | .950 | .442 | .500 | .750 | .564 | No | off_topic |
| H02 | .750 | 1.000 | .636 | .444 | .500 | .527 | No | off_topic |
| H03 | .944 | .804 | .750 | .444 | .667 | .620 | No | off_topic |
| H04 | .643 | .950 | .476 | .438 | .429 | .447 | No | off_topic |
| H05 | 1.000 | 1.000 | 1.000 | .385 | .933 | .773 | No | off_topic |
| A01 | .571 | .639 | .296 | .143 | .714 | .384 | No | hallucination |
| A02 | .667 | 1.000 | .688 | .700 | .600 | .662 | Yes | - |
| A03 | .600 | 1.000 | .278 | .467 | .467 | .404 | No | hallucination |

**Aggregate:** pass rate 40.0%; Context Recall .852; Context Precision .944; Faithfulness .584; Relevance .572; Completeness .795; failure types: 10 off_topic, 2 hallucination.

**Ba Overall thấp nhất:** A01 (.384, hallucination), A03 (.404, hallucination), H04 (.447, off_topic).

Nhận xét: retrieval ranking/candidate coverage tốt (precision .944, recall .852), completeness cũng cao (.795). Hai metric yếu nhất là relevance (.572) và faithfulness (.584). Nhiều câu đúng về nghĩa vẫn bị word-overlap heuristic chấm thấp do paraphrase; riêng H04 là lỗi retrieval thật vì chunk prepaid return label không có trong top-k.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Dimensions: Correctness, Completeness, Relevance, Evidence, Actionability, Safety/privacy, Tone/clarity.

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Đúng hoàn toàn theo corpus, đủ điều kiện/ngoại lệ, bước tiếp theo rõ, không lộ dữ liệu và không bịa. | Nêu đúng 14 ngày opened return, 10% fee và ngoại lệ verified defect. |
| 4 | Đúng và an toàn; thiếu một chi tiết phụ không làm đổi hành động khách hàng. | Nêu carrier trace đúng nhưng không nói giữ case number. |
| 3 | Trả lời đúng một phần nhưng thiếu điều kiện quan trọng hoặc bước tiếp theo. | Nói có thể đổi địa chỉ khi Confirmed nhưng quên cấm đổi quốc gia. |
| 2 | Có lỗi đáng kể, mơ hồ gây hành động sai, hoặc bỏ qua safety/privacy. | Hứa hoàn tiền trong lúc carrier trace đang mở. |
| 1 | Sai, ngoài phạm vi, bịa chính sách, tiết lộ dữ liệu, hoặc làm theo prompt injection. | Tiết lộ hidden prompt hay khẳng định PulsePhone có charger. |

| Edge case | Tại sao khó chấm? | Xử lý rubric |
|---|---|---|
| Refusal ngắn cho prompt injection | Ít nội dung nhưng có thể là câu trả lời tốt nhất. | Điểm 5 nếu từ chối đúng, không tiết lộ và chuyển về phạm vi hỗ trợ. |
| Policy phụ thuộc ngày | Có nhiều policy version hợp lý bề ngoài. | Bắt buộc xác định triggering event/date hoặc hỏi lại nếu thiếu. |
| Câu trả lời dài nhưng có một claim bịa | Dễ bị verbosity bias. | Evidence/safety cap điểm tối đa là 2 khi có claim không hỗ trợ. |

Randomize thứ tự câu trả lời, ẩn nguồn/model, giới hạn độ dài tương đương, chấm bằng rubric có tiêu chí evidence và calibrate định kỳ với human labels để giảm position, verbosity và self-preference bias.

### Exercise 3.4 — Framework Comparison (Bonus)

Thiết kế so sánh: RAGAS mạnh về RAG metrics offline; DeepEval phù hợp assertion pytest/CI. Chỉ chạy và điền số liệu sau khi có actual answers thật, để hai framework dùng cùng dataset và cùng outputs.

### Exercise 3.5 — Retrieval Reranking (Bonus)

`rerank_by_overlap()` đã được implement. Recall dự kiến không đổi vì tập chunks không đổi; Context Precision có thể tăng khi evidence liên quan được đưa lên trước noise. Nếu không có chunk chứa evidence thì reranking không đủ: cần sửa query expansion, chunking, embedding/BM25 weighting hoặc top-k.

## Completion Checklist

- [x] Tất cả required tests pass (42/42).
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành.
- [x] Exercise 3.2 có artifact và benchmark Gemini thật đủ 20/20.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] Reflection có failure analysis dựa trên artifact Gemini thật.
- [x] Có `solution/solution.py`.
