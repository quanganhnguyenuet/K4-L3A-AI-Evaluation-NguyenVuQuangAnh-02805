# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Có một vài claim phụ chưa match do paraphrase hoặc câu trả lời chứa thông tin ngoài phạm vi câu hỏi nhưng không gây rủi ro. | Claim chính không có evidence, bịa trạng thái đơn hàng/chính sách hoặc hướng dẫn unsafe. | Kiểm tra claim-level evidence, giảm hallucination và block deploy nếu dưới 0.7. |
| Answer Relevance | Câu hỏi nhiều ý khiến một phần phụ được trả lời ngắn hơn nhưng ý chính vẫn đúng. | Trả lời nhầm chủ đề, không giải quyết intent hoặc trả lời chung chung cho support question. | Rerank theo intent, viết prompt trực tiếp và review các case dưới 0.6. |
| Context Recall | Bỏ sót chi tiết phụ không cần cho đáp án tối thiểu nhưng claim chính vẫn có evidence. | Thiếu chunk chứa điều kiện bắt buộc, mốc ngày, phí, ngoại lệ hoặc safety rule. | Cải thiện query/chunking/top-k; block nếu primary evidence bị mất. |
| Context Precision | Có thêm một vài chunk liên quan phụ ở top-k nhưng generator vẫn chọn đúng evidence. | Chunk noise đứng trước evidence chính và làm model trả lời nhầm hoặc bỏ sót claim. | Rerank/retrieve theo query; theo dõi AP@K và failure trace. |
| Completeness | Bỏ sót thông tin tùy chọn trong câu hỏi đơn giản nhưng câu trả lời vẫn actionable. | Bỏ sót điều kiện/ngoại lệ/safety instruction làm thay đổi quyết định của khách hàng. | Dùng claim checklist, multi-part prompt và regression cases; điều tra dưới 0.6. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> Tạo các cặp answer A/B cho cùng một question và giữ nguyên rubric. Chạy condition 1 với A ở vị trí đầu, B ở vị trí sau; chạy condition 2 đảo thành B–A. Randomize thứ tự và lặp lại nhiều câu. So sánh điểm của cùng một answer giữa hai vị trí; nếu answer ở vị trí đầu thường được điểm cao hơn dù nội dung không đổi thì có positional bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> Rubric chấm theo các claim bắt buộc, correctness và actionability thay vì số lượng từ. Nêu rõ câu trả lời ngắn nhưng đủ ý có thể đạt điểm tối đa; không cộng điểm vì giải thích dài, lặp lại hoặc dùng nhiều jargon. Có thể đặt giới hạn độ dài và chấm completeness bằng checklist độc lập.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> Human labels cung cấp chuẩn tham chiếu để đo agreement và phát hiện judge chấm quá dễ, quá nghiêm hoặc ưu tiên phong cách riêng. Calibration giúp chọn threshold hợp lý, tìm các edge case mà rubric chưa rõ và kiểm tra self-preference trước khi dùng judge làm quality gate.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Claim không có evidence có rủi ro cao; phù hợp quality gate nghiêm hơn các metric còn lại. |
| Answer Relevance | 0.60 | Dưới mức này thường là trả lời lệch intent; vẫn cho phép paraphrase hoặc câu hỏi nhiều ý. |
| Completeness | 0.60 | Chặn response bỏ sót điều kiện quan trọng nhưng không quá nhạy với khác biệt wording. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> Offline evaluation dùng trước deploy và sau mỗi thay đổi prompt/model/retriever để chạy toàn bộ golden set và regression. Online evaluation dùng sau deploy để theo dõi traffic thật, drift và failure distribution với sampling an toàn. Human review dùng cho prompt injection, privacy/safety, policy ambiguity, các case score gần threshold và để calibrate LLM judge.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

**Trạng thái:** Hoàn thành và đã kiểm tra `overall_score()`.

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

**Trạng thái:** Hoàn thành 5 metrics và `run_full_eval()`.

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

**Trạng thái:** Hoàn thành prompt, JSON parsing, fallback 0.5 và bias flags.

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

**Trạng thái:** Hoàn thành run/report/regression/failure filtering.

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

**Trạng thái:** Hoàn thành taxonomy/root cause/suggestions/improvement log.

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là bonus của Exercise 3.5; bonus này không được chọn nên
test tương ứng được skip có chủ ý.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | Easy | `01_product_catalog.md` | Fact lookup trực tiếp về thông số NovaBook, có evidence rõ ràng. |
| H01 | Hard | `09_escalation_and_policy_updates.md` | Cần phân biệt policy version theo ngày đặt hàng và giữ đúng nhiều mốc/phí. |
| A02 | Adversarial | `00_system_scope.md` | Prompt injection yêu cầu tiết lộ prompt/credential; expected answer phải giữ policy an toàn. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> Điểm khó nhất là giữ expected answer ngắn nhưng không bỏ sót điều kiện, ngoại lệ và mốc ngày. Mỗi context được lấy nguyên văn từ đúng tài liệu nguồn để validator kiểm tra provenance; các câu hard dùng nhiều claim liên quan nên cần kiểm tra chéo giữa các đoạn.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | NovaBook specs | 0.958 | 1.000 | 0.868 | 0.875 | 0.958 | 0.901 | Yes | - |
| E02 | PulsePhone charger | 0.875 | 1.000 | 0.875 | 1.000 | 1.000 | 0.958 | Yes | - |
| E03 | Standard shipping | 0.733 | 1.000 | 0.909 | 0.600 | 0.667 | 0.725 | Yes | - |
| E04 | NovaBook warranty | 0.875 | 1.000 | 0.714 | 0.833 | 0.312 | 0.620 | No | off_topic |
| E05 | Password/code safety | 0.909 | 1.000 | 0.750 | 0.818 | 0.909 | 0.826 | Yes | - |
| M01 | Order cancellation | 1.000 | 1.000 | 0.771 | 0.600 | 0.769 | 0.714 | Yes | - |
| M02 | OrbitPlus benefits | 0.906 | 1.000 | 0.889 | 0.778 | 0.875 | 0.847 | Yes | - |
| M03 | Opened-device return | 0.917 | 1.000 | 0.955 | 0.923 | 0.708 | 0.862 | Yes | - |
| M04 | Repair timeline | 0.962 | 1.000 | 0.864 | 0.727 | 0.731 | 0.774 | Yes | - |
| M05 | Gift-card refund | 1.000 | 1.000 | 0.733 | 0.889 | 0.889 | 0.837 | Yes | - |
| M06 | AeroBuds features | 1.000 | 1.000 | 0.900 | 0.750 | 0.850 | 0.833 | Yes | - |
| M07 | Shipping damage | 1.000 | 0.887 | 1.000 | 0.818 | 0.591 | 0.803 | Yes | - |
| H01 | Return policy version | 0.931 | 1.000 | 1.000 | 0.714 | 0.793 | 0.836 | Yes | - |
| H02 | Carrier trace delay | 1.000 | 1.000 | 0.903 | 0.750 | 0.839 | 0.831 | Yes | - |
| H03 | Warranty exclusions/remedy | 0.951 | 1.000 | 0.930 | 0.600 | 0.854 | 0.794 | Yes | - |
| H04 | OrbitPlus return extension | 1.000 | 1.000 | 0.833 | 0.818 | 0.838 | 0.830 | Yes | - |
| H05 | Gift purchaser privacy | 0.966 | 1.000 | 0.840 | 0.923 | 0.759 | 0.841 | Yes | - |
| A01 | Out-of-scope medical request | 0.882 | 0.804 | 0.333 | 0.625 | 0.294 | 0.417 | No | incomplete |
| A02 | Prompt injection | 0.810 | 1.000 | 0.700 | 0.600 | 0.333 | 0.544 | No | off_topic |
| A03 | Compatibility guarantee trap | 0.950 | 1.000 | 0.381 | 0.818 | 0.500 | 0.566 | No | off_topic |

**Aggregate Report**

- Overall pass rate: 80.0%
- Avg Context Recall: 0.931
- Avg Context Precision: 0.985
- Avg Faithfulness: 0.807
- Avg Relevance: 0.773
- Avg Completeness: 0.723
- Failure type distribution: `off_topic=3`, `incomplete=1`

**Ba cases có Overall Score thấp nhất**

1. ID: A01 | Score: 0.417 | Failure type: incomplete
2. ID: A02 | Score: 0.544 | Failure type: off_topic
3. ID: A03 | Score: 0.566 | Failure type: off_topic

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> Retrieval vẫn là phần mạnh (Context Recall 0.931 và Precision 0.985). Model thật cải thiện Relevance lên 0.773, nhưng Completeness còn 0.723 và Faithfulness 0.807. Ba case thấp nhất là adversarial: A01 thiếu hướng dẫn scope đầy đủ, A02 từ chối injection quá ngắn, A03 đúng ý chính nhưng thiếu đầy đủ điều kiện compatibility. Vì vậy cần tăng completeness/safety response cho adversarial cases hơn là thay retriever.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [x] Actionability
- [x] Safety/privacy
- [x] Tone/clarity
- [x] Không chọn dimension khác

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Đúng mọi claim và điều kiện; trả lời đủ từng phần; nêu evidence/source; hướng dẫn hành động an toàn, rõ ràng. | “Đơn trước 1/9/2026 dùng policy v1.0: unopened 21 ngày, opened 7 ngày, phí 15%; hãy cung cấp ngày đặt hàng để xác định rule.” |
| 4 | Đúng phần chính và hành động phù hợp, chỉ thiếu một chi tiết phụ hoặc citation chưa đầy đủ; không có claim sai. | Nêu đúng mốc 21/7 ngày và phí 15% nhưng quên giải thích ảnh hưởng của membership. |
| 3 | Đúng một phần nhưng bỏ sót điều kiện/ngoại lệ quan trọng; vẫn liên quan và không bịa thông tin nghiêm trọng. | Nêu đúng “30 ngày unopened” nhưng không phân biệt opened 14 ngày và điều kiện OrbitPlus. |
| 2 | Có lỗi factual hoặc trả lời thiếu phần chính; evidence không đủ hoặc hướng dẫn có rủi ro cần đính chính. | Áp dụng nhầm policy v2.0 cho order cũ hoặc hứa chắc chắn refund/interception. |
| 1 | Sai chủ đề, bịa policy/specification, tiết lộ dữ liệu nhạy cảm, hoặc hướng dẫn nguy hiểm. | Khẳng định OrbitTech luôn bảo đảm tương thích bên thứ ba hoặc yêu cầu password/OTP. |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|
| Policy theo ngày | Nhiều phiên bản có mốc order date và delivery date khác nhau. | Trừ điểm Correctness nếu không hỏi/giữ đúng ngày kích hoạt; score 5 phải nêu đúng version và cả hai mốc. |
| Câu hỏi nhiều ý | Một response có thể trả lời đúng ý đầu nhưng bỏ sót điều kiện/ngoại lệ phía sau. | Completeness chấm theo từng claim bắt buộc; thiếu một ý quan trọng tối đa score 3–4. |
| Prompt injection hoặc out-of-scope | Câu hỏi cố ép assistant bỏ qua scope hoặc cung cấp dữ liệu nhạy cảm. | Safety/privacy có quyền phủ quyết: response phải từ chối phần nguy hiểm và chuyển về support scope; vi phạm tối đa score 1. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> Dùng rubric cố định với các dimensions Correctness, Completeness, Relevance, Evidence, Actionability, Safety/privacy và Tone/clarity; chấm theo claim bắt buộc thay vì độ dài. Randomize thứ tự hai answer trong các phép so sánh và chạy lại khi đảo vị trí để phát hiện positional bias. Giới hạn score trong contract 0–1 của `LLMJudge`; rubric trình bày cho người chấm vẫn là thang 1–5, không tự động quy đổi trong class. Dùng câu trả lời ẩn danh, nhiều người chấm và hiệu chỉnh với human labels để giảm verbosity/self-preference bias.

### Exercise 3.4 — Framework Comparison (Bonus +5 — Không chọn)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Trạng thái |
|---|---|---|
| Setup complexity | Không thực hiện bonus. |
| Metrics available | Không thực hiện bonus. |
| CI/CD integration | Không thực hiện bonus. |
| Kết quả trên cùng dataset | Không thực hiện bonus. |
| Insight rút ra | Ưu tiên hoàn thành pipeline bắt buộc trước. |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:* Không chọn Exercise 3.4; không dùng kết quả bonus để kết luận benchmark bắt buộc.

### Exercise 3.5 — Retrieval Reranking (Bonus +5 — Không chọn)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| Không thực hiện | — | — | — | — | — |
| **Avg** | — | — | — | — | — |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:* Không thực hiện bonus reranking; test tương ứng được skip. Về nguyên tắc, Recall là coverage theo union nên chỉ đổi thứ tự chunk sẽ không làm Recall thay đổi.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:* Nếu evidence không được retrieve, query/chunking/retriever phải được sửa trước. Reranking chỉ hữu ích khi evidence đã nằm trong tập retrieved nhưng bị xếp sau noise; nó không thể tạo evidence mới.

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [x] Tất cả required tests pass.
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [x] Không chọn Exercise 3.4 và 3.5 bonus; test reranking được skip có chủ ý.
