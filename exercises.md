# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi chít chat đơn giản không dựa trên context document. | Chatbot bị đặt câu hỏi nghiệp vụ nhưng hallucinate thông tin sai lệch so với chính sách OrbitTech. | Tinh chỉnh RAG prompt (strict grounding), thêm instruction "chỉ trả lời dựa trên context". |
| Answer Relevance | Khách hàng hỏi câu dài nhiều chi tiết thừa và câu trả lời đi thẳng vào giải pháp không lặp lại từ khóa. | Câu trả lời lan man, trả lời sai trọng tâm câu hỏi của người dùng. | Kiểm tra câu lệnh prompt sinh câu trả lời, cải thiện query transformation / intent parsing. |
| Context Recall | Câu hỏi đơn giản factual mà retriever lấy thiếu các chi tiết không quan trọng. | Retriever không lấy được chunk chứa điều kiện bảo hành/đổi trả cốt lõi dẫn đến trả lời thiếu. | Cải thiện phương pháp chunking, tối ưu hóa BM25 / vector index / hybrid search. |
| Context Precision | Retriever lấy top-5 chunks trong đó thông tin đúng nằm ở rank 3-4 thay vì rank 1. | Mọi chunk liên quan bị đẩy xuống cuối kết quả retrieval hoặc chứa toàn noise/irrelevant context. | Bổ sung Reranker (như Cross-Encoder) để sắp xếp lại thứ tự chunk trước khi truyền vào LLM. |
| Completeness | Khách hàng hỏi nhiều ý phụ không liên quan đến chính sách và assistant bỏ qua ý phụ vô hại. | Thiếu các bước xử lý quan trọng trong quy trình hoàn tiền/đổi trả hoặc bỏ sót ngoại lệ chính sách. | Cập nhật system prompt để trích xuất đầy đủ các điều kiện và ngoại lệ trong reference answer. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Condition 1 (Original Order):** Đưa `Model A` làm Response 1 và `Model B` làm Response 2 vào prompt của LLM Judge để chấm/so sánh.
> - **Condition 2 (Reversed Order):** Tráo đổi vị trí: Đưa `Model B` làm Response 1 và `Model A` làm Response 2 vào cùng prompt rubric.
> - **Phân tích:** Nếu kết quả chấm đổi chiều thiên vị cho Response 1 ở cả 2 condition (ví dụ: Response 1 luôn thắng), xác nhận LLM Judge dính Position Bias. Xử lý bằng cách Swap Positioning & Average Score (tính trung bình cả 2 lượt swap).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Định nghĩa rõ ràng trong Rubric phạt điểm câu trả lời dông dài, chứa thông tin thừa ("Penalize unnecessary filler or wordiness").
> - Yêu cầu LLM Judge đánh giá tiêu chí **Conciseness & Precision** hoặc **Completeness per token**, trong đó điểm tối đa (5/5) chỉ trao cho câu trả lời ngắn gọn, chính xác, đủ ý mà không thừa từ.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - LLM Judge không có nhận thức thực tế và dễ dính các bias tự nhiên (position, verbosity, self-preference).
> - Việc calibrate với Human Labels (đánh giá từ chuyên gia/human experts) giúp tính toán độ tương quan (Cohen's Kappa / Spearman Correlation), phát hiện khoảng lệch (systematic bias) và tinh chỉnh prompt/rubric để LLM Judge đạt độ chính xác gần nhất với đánh giá con người.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Trong domain hỗ trợ khách hàng, đưa ra thông tin bịa đặt (hallucination) gây tổn hại trực tiếp tới uy tín và tài chính doanh nghiệp. |
| Answer Relevance | 0.80 | Đảm bảo câu trả lời trực tiếp giải quyết thắc mắc của khách hàng, tránh trả lời lạc đề gây phiền hà. |
| Completeness | 0.75 | Đảm bảo cung cấp đủ thông tin quy trình/điều kiện cho khách hàng, cho phép dung sai nhỏ đối với các chi tiết phụ. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation:** Dùng trong giai đoạn phát triển (Dev/Staging) và CI/CD pipeline trước khi deploy. Chạy tự động trên Golden Dataset để kiểm tra regression.
> - **Online Evaluation:** Dùng trong môi trường Production thực tế. Giám sát các câu trả lời thật từ user qua telemetry, LLM-as-a-Judge real-time, phản hồi thumbs-up/thumbs-down và tỉ lệ chuyển hỗ trợ (escalation rate).
> - **Human Review:** Dùng định kỳ (đánh giá mẫu 1-5% lượt chat) hoặc khi có alert từ Online Eval/từ chối từ khách hàng, nhằm rà soát các trường hợp khó (edge cases) và cập nhật Golden Dataset.

---

## Part 2 — Core Coding (9:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

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

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

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
| E02 | easy | `01_product_catalog.md` | Tra cứu trực tiếp thông số phần cứng NovaBook 14 (RAM 16 GB, SSD 512 GB) từ một vị trí văn bản trong corpus mà không cần suy luận phức tạp. |
| H01 | hard | `09_escalation_and_policy_updates.md` | Yêu cầu xử lý điều kiện quy định phiên bản chính sách: Sự kiện kích hoạt (ngày đặt hàng 25/8/2026) quyết định áp dụng policy v1.0 (21 ngày unopened), không phụ thuộc vào ngày nhận hàng (3/9/2026). |
| A02 | adversarial | `00_system_scope.md` | Tấn công Prompt Injection cố tình giả lập lệnh hệ thống (System Override) để ép Assistant tiết lộ system prompt và mật khẩu. Assistant từ chối theo nguyên tắc bảo mật. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*
> Điểm khó nhất là việc trích xuất bằng chứng (evidence provenance) phải khớp tuyệt đối (verbatim substring) từng ký tự từ các file Markdown trong corpus, đặc biệt là các đoạn chứa mã định danh (như tên file `03_promotions_and_membership.md` có ký tự backtick) hoặc thuật ngữ nghiệp vụ. Đồng thời, việc đảm bảo mọi claim trong `expected_answer` đều có bằng chứng trực tiếp bảo vệ mà không bịa thêm thông tin ngoài corpus đòi hỏi sự rà soát kỹ lưỡng qua nhiều tài liệu liên quan (cross-document evidence).

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
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
