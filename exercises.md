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
| Faithfulness | Khi câu hỏi là câu chào hỏi, cảm ơn xã giao, hoặc câu hỏi ngoài phạm vi (out-of-scope/adversarial) mà trợ lý lịch sự từ chối/disclaimer bằng mẫu chung không cần dựa vào context; hoặc khi câu trả lời dùng từ đồng nghĩa/paraphrase khác từ ngữ context dẫn tới word overlap thấp nhưng ngữ nghĩa chuẩn xác. | Khi câu trả lời bịa đặt (hallucination) các thông tin nghiệp vụ cốt lõi (chính sách bảo hành, hoàn tiền, giá bán, điều kiện đổi trả) mà tài liệu không có. Gây rủi ro pháp lý và tranh chấp với khách hàng. | Thắt chặt system prompt ("Chỉ trả lời dựa trên context được cung cấp, nếu không có hãy từ chối"), giảm temperature = 0.0, thêm hallucination filter/guardrail trước khi phản hồi người dùng. |
| Answer Relevance | Khi người dùng đặt câu hỏi quá ngắn, mơ hồ, hoặc câu hỏi cố tình tấn công (prompt injection/jailbreak) khiến trợ lý phải hỏi lại để làm rõ (clarifying question) hoặc từ chối theo quy định an toàn, dẫn đến word overlap thấp với câu hỏi. | Người dùng hỏi trực diện một vấn đề kỹ thuật/chính sách cụ thể (ví dụ: "Chính sách bảo hành tai nghe"), nhưng trợ lý trả lời sang chủ đề hoàn toàn khác (ví dụ: tư vấn mua laptop) hoặc lan man ngoài lề. | Cải tiến prompt hướng dẫn tập trung giải quyết đúng trọng tâm câu hỏi; bổ sung bộ phân loại ý định (Intent Classifier) hoặc Query Rewriting trước khi đưa vào RAG pipeline. |
| Context Recall | Khi câu hỏi mang tính giao tiếp thông thường, hoặc câu hỏi về thông tin không thuộc kho dữ liệu cửa hàng (out-of-domain) nên retriever không thể tìm thấy context tương ứng. | Người dùng hỏi về thông tin quan trọng có trong kho tài liệu (ví dụ: điều kiện đổi trả, thông số kỹ thuật), nhưng retriever bỏ sót chunk chứa bằng chứng quan trọng (gold evidence), khiến generator không có đủ dữ liệu trả lời. | Tăng số lượng top-k chunks lấy về; tối ưu hóa kích thước chunk (chunk size) và chunk overlap; kết hợp Hybrid Search (BM25 + Dense Vector Embeddings); áp dụng kỹ thuật HyDE (Hypothetical Document Embeddings). |
| Context Precision | Khi chỉ có 1-2 chunks được truy xuất và tất cả đều liên quan; hoặc tài liệu ngắn nên thứ tự xếp hạng chưa tối ưu vẫn không làm loãng thông tin đưa vào context của LLM. | Chunk chứa thông tin chính xác nhất bị xếp ở cuối danh sách (rank thấp), trong khi các chunks đầu là noise/nhiễu, khiến LLM bị hiện tượng "Lost in the Middle" hoặc sinh câu trả lời sai lệch theo chunk đầu. | Bổ sung bước Reranking (dùng Cross-Encoder Reranker hoặc lexical overlap reranker) để đưa chunk liên quan nhất lên đầu; lọc bỏ (prune) các chunks có similarity score thấp trước khi nhồi vào prompt. |
| Completeness | Khi người dùng chỉ hỏi xác nhận nhanh (quick answer / câu hỏi Yes-No) và trợ lý trả lời súc tích, ngắn gọn đúng trọng tâm thay vì liệt kê chi tiết dài dòng như expected answer chuẩn. | Người dùng hỏi về quy trình gồm nhiều bước/điều kiện bắt buộc (ví dụ: 4 điều kiện để được hoàn tiền), nhưng câu trả lời chỉ nêu 1 điều kiện và bỏ sót 3 điều kiện còn lại, khiến khách hàng hiểu sai quy trình. | Điều chỉnh prompt yêu cầu liệt kê đầy đủ tất cả các bước/điều kiện theo định dạng bullet points; tăng max output tokens; đảm bảo context retriever gom đủ các chunk liên quan đến toàn bộ các ý cần trả lời. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Mục tiêu:** Xác định xem LLM Judge có xu hướng thiên vị câu trả lời xuất hiện ở vị trí đầu tiên (Position 1) hay không.
> - **Condition 1 (Original Order):** Cung cấp cho LLM Judge cặp câu trả lời với Model A ở vị trí Candidate 1 và Model B ở vị trí Candidate 2. Ghi nhận model chiến thắng và điểm số.
> - **Condition 2 (Swapped Order):** Đảo ngược vị trí của cùng cặp câu trả lời: Model B ở vị trí Candidate 1 và Model A ở vị trí Candidate 2, giữ nguyên toàn bộ prompt, câu hỏi, context và rubric chấm điểm.
> - **Phân tích kết quả:** So sánh kết quả ở 2 điều kiện. Nếu Candidate ở vị trí 1 luôn nhận điểm cao hơn bất kể là Model A hay Model B (hoặc tỷ lệ thay đổi lựa chọn vượt quá 15-20%), chứng tỏ judge có Position bias rõ rệt. Khắc phục bằng cách đánh giá cả hai chiều rồi lấy trung bình hoặc ngẫu nhiên hóa vị trí ứng viên.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Quy định rõ trong rubric đánh giá: "Độ dài không đồng nghĩa với chất lượng. Đánh giá chất lượng dựa trên mật độ thông tin chính xác và khả năng trả lời đúng trọng tâm (fact density)."
> - Bổ sung tiêu chí tính súc tích (conciseness): phạt điểm những câu trả lời dài dòng, lan man, lặp từ hoặc chứa các đoạn văn sáo rỗng không cần thiết.
> - Cung cấp Few-shot examples minh họa rõ: câu trả lời ngắn gọn nhưng đầy đủ ý vẫn đạt điểm tối đa (5/5), trong khi câu trả lời dài nhưng rỗng thông tin hoặc thiếu ý chính sẽ bị trừ điểm (2-3/5).

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**
> *Câu trả lời:*
> - LLM Judge không có nhận thức thực sự và dễ mắc các bias nội tại (position, verbosity, self-preference, leniency/severity bias), có thể cho điểm lệch xa tiêu chuẩn thực tế của chuyên gia.
> - Hiệu chỉnh (calibrate) với nhãn của con người (human expert labels) bằng các hệ số tương quan (như Cohen’s Kappa, Spearman correlation):
>   1. Đảm bảo độ tin cậy và giá trị đo lường thực tế của pipeline đánh giá tự động trước khi triển khai trên diện rộng.
>   2. Phát hiện sớm các lỗ hổng trong prompt rubric để điều chỉnh tiêu chí chấm điểm và ví dụ mẫu (few-shot).
>   3. Xác định được mức độ tin cậy và biên độ sai số khi dùng LLM Judge thay thế con người nhằm tối ưu chi phí và tốc độ kiểm thử.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Đối với hệ thống hỗ trợ khách hàng OrbitTech, tính trung thực là tối quan trọng. Hallucination về giá cả, thời hạn bảo hành hay chính sách đổi trả có thể gây thiệt hại tài chính và tranh chấp pháp lý nghiêm trọng. |
| Answer Relevance | 0.80 | Đảm bảo câu trả lời giải quyết trực tiếp và chính xác thắc mắc của khách hàng, tránh gây ức chế do trả lời lạc đề, lảng tránh hoặc thông tin vô nghĩa. |
| Completeness | 0.75 | Đảm bảo khách hàng nhận được hướng dẫn đầy đủ tất cả các bước hoặc điều kiện cần thiết, tránh việc phải liên hệ lại nhiều lần (giảm repeat contact rate). |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment):**
>   - Sử dụng trong quá trình phát triển (development) và trong CI/CD pipeline trước khi release phiên bản mới.
>   - Đánh giá trên Golden Dataset cố định với chi phí thấp, tốc độ nhanh, giúp phát hiện sớm hiện tượng hồi quy (regression) mà không gây rủi ro cho người dùng thực.
> - **Online Evaluation (Post-deployment / Production):**
>   - Sử dụng liên tục khi hệ thống đang chạy phục vụ khách hàng thực tế (real-time telemetry).
>   - Thu thập tín hiệu phản hồi ngầm định (implicit: dwell time, click-through, escalation rate) và tường minh (explicit: thumbs up/down, CSAT), hoặc chạy A/B testing giữa các prompt/model để đo lường trải nghiệm thực tế.
> - **Human Review (Periodic & Targeted Auditing):**
>   - Sử dụng định kỳ để kiểm toán chất lượng hoặc đánh giá có chủ đích trên các trường hợp rủi ro cao: các phiên hội thoại bị khách hàng đánh giá tiêu cực, các câu trả lời có điểm tin cậy (confidence score) thấp, hoặc các ca biên (edge cases / adversarial attacks).
>   - Dùng để xây dựng, làm giàu Golden Dataset và hiệu chỉnh lại LLM-as-a-Judge.

---

## Part 2 — Core Coding (14:45–15:40)

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

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

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

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
