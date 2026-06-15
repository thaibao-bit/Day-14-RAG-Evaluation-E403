# Day 14 — Exercises
## AI Evaluation & Benchmarking | Lab Worksheet

**Lab Duration:** 3 hours

---

## Part 1 — Warm-up (0:00–0:20)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng, score interpretation:
- 0.8–1.0: Good (Monitor, maintain)
- 0.6–0.8: Needs work (Analyze failures, iterate)
- < 0.6: Significant issues (Deep investigation)

Cho mỗi RAGAS metric, xác định khi nào score thấp là acceptable vs critical:

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|--------|------------------------------|-----------------------------|-----------------| 
| Faithfulness | | | |
| Answer Relevancy | | | |
| Context Recall | | | |
| Context Precision | | | |
| Completeness | | | |

---

### Exercise 1.2 — Position Bias in LLM-as-Judge

Từ bài giảng, 3 loại bias trong LLM-as-Judge:
- **Position Bias:** Judge ưu tiên answer xuất hiện trước
- **Verbosity Bias:** Judge cho điểm cao hơn answer dài hơn
- **Self-Preference:** GPT-4 judge ưu tiên GPT-4 output

**Câu 1: Thiết kế experiment phát hiện Position Bias**
> *Mô tả thí nghiệm với ít nhất 2 conditions:*

**Câu 2: Làm sao fix Verbosity Bias trong rubric design?**
> *Your answer:*

**Câu 3: Tại sao cần "calibrate against human" theo best practices?**
> *Your answer:*

---

### Exercise 1.3 — Evaluation trong CI/CD

Theo bài giảng: "Agent không pass eval = không được deploy, giống unit test."

**Câu 1: Bạn sẽ set threshold nào cho từng metric trong CI/CD pipeline?**

| Metric | Threshold (block deploy nếu dưới) | Lý do |
|--------|----------------------------------|-------|
| Faithfulness | | |
| Answer Relevancy | | |
| Completeness | | |

**Câu 2: Khi nào nên chạy offline eval vs online eval?**
> *Your answer (tham khảo bảng triggers trong bài giảng):*

---

## Part 2 — Core Coding (0:20–1:20)

Implement all TODOs in `template.py`. Focus on:

### Task 1: Data Models
- `QAPair` dataclass: question, expected_answer, context, metadata
- `EvalResult` dataclass: qa_pair, actual_answer, faithfulness, relevance, completeness, passed, failure_type
- `overall_score()` method: average of 3 metrics

### Task 2: RAGASEvaluator
- `evaluate_faithfulness(answer, context)` → word overlap heuristic
- `evaluate_relevance(answer, question)` → word overlap heuristic  
- `evaluate_completeness(answer, expected)` → word overlap heuristic
- `run_full_eval(...)` → combine all 3 + determine failure_type

### Task 3: LLMJudge
- `score_response(question, answer, rubric)` → build prompt, call judge, parse scores
- `detect_bias(scores_batch)` → check positional, leniency, severity bias

### Task 4: BenchmarkRunner
- `run(qa_pairs, agent_fn, evaluator)` → run all pairs through agent + eval
- `generate_report(results)` → aggregate stats
- `run_regression(new_results, baseline_results)` → detect drops > 0.05
- `identify_failures(results, threshold)` → filter below threshold

### Task 5: FailureAnalyzer
- `categorize_failures(failures)` → group by type
- `find_root_cause(failure)` → suggest cause based on lowest score
- `generate_improvement_suggestions(failures)` → prioritized fix list
- `generate_improvement_log(failures, suggestions)` → Markdown table output

**Verify:** `pytest tests/ -v`

---

## Part 3 — Extended Exercises (1:20–2:20)

### Exercise 3.1 — Build Your Golden Dataset (Stratified Sampling)

Theo bài giảng, golden dataset cần:
- Expert-written expected answers
- Stratified sampling theo difficulty
- Cover tất cả use cases chính
- Có edge cases và adversarial inputs

**Tạo 20 QA pairs cho domain của bạn (từ Day 2):**

#### Easy (5 pairs) — Factual lookup, single-doc
| ID | Question | Expected Answer | Context (1–2 sentences) | Source Doc |
|----|----------|-----------------|------------------------|------------|
| E01 | | | | |
| E02 | | | | |
| E03 | | | | |
| E04 | | | | |
| E05 | | | | |

#### Medium (7 pairs) — Multi-step reasoning, 2–3 docs
| ID | Question | Expected Answer | Context (1–2 sentences) | Source Doc |
|----|----------|-----------------|------------------------|------------|
| M01 | | | | |
| M02 | | | | |
| M03 | | | | |
| M04 | | | | |
| M05 | | | | |
| M06 | | | | |
| M07 | | | | |

#### Hard (5 pairs) — Complex/ambiguous, nhiều cách hiểu
| ID | Question | Expected Answer | Context (1–2 sentences) | Source Doc |
|----|----------|-----------------|------------------------|------------|
| H01 | | | | |
| H02 | | | | |
| H03 | | | | |
| H04 | | | | |
| H05 | | | | |

#### Adversarial (3 pairs) — Out-of-scope, cố tình phá
| ID | Question | Expected Answer | Context (1–2 sentences) | Attack Type |
|----|----------|-----------------|------------------------|-------------|
| A01 | | | | Out-of-scope |
| A02 | | | | Prompt injection |
| A03 | | | | Ambiguous/trap |

---

### Exercise 3.2 — Benchmark Run

Chạy `BenchmarkRunner` trên 20 QA pairs. Ghi lại kết quả:

| ID | Question (short) | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|----|-----------------|--------------|-----------|--------------|---------|---------|--------------|
| E01 | | | | | | | |
| E02 | | | | | | | |
| ... | | | | | | | |

**Aggregate Report:**
- Overall pass rate: ____%
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**3 câu hỏi scored thấp nhất:**
1. ID: ___ | Score: ___ | Failure type: ___
2. ID: ___ | Score: ___ | Failure type: ___
3. ID: ___ | Score: ___ | Failure type: ___

---

### Exercise 3.3 — LLM-as-Judge Rubric Design

Theo bài giảng, rubric scoring 1–5 cần tiêu chí CỤ THỂ cho mỗi mức.

**Thiết kế rubric cho domain của bạn:**

| Score | Tiêu chí (domain-specific) | Ví dụ response |
|-------|---------------------------|----------------|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Criteria dimensions (chọn 3–5 từ list hoặc tự thêm):**
- [ ] Correctness (đúng sự thật?)
- [ ] Completeness (đủ chi tiết?)
- [ ] Relevance (trả lời đúng câu hỏi?)
- [ ] Citation (trích nguồn?)
- [ ] Tone (giọng phù hợp context?)
- [ ] Actionability (có thể hành động theo?)
- [ ] Safety (không có harmful content?)

**3 edge cases khó score:**

| Edge Case | Tại sao khó score | Cách xử lý trong rubric |
|-----------|-------------------|------------------------|
| | | |
| | | |
| | | |

---

### Exercise 3.4 — Framework Comparison (Bonus)

Nếu đã hoàn thành 3.1–3.3, chọn 2 trong 3 frameworks để so sánh:

| Tiêu chí | Framework 1: _____ | Framework 2: _____ |
|----------|-------------------|-------------------|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Score cho cùng dataset | | |
| Insight rút ra | | |

**Câu hỏi phân tích:**
- Scores có consistent giữa 2 frameworks không?
- Framework nào strict hơn? Tại sao?
- Failure cases có giống nhau không?

---

## Part 4 — Reflection (2:20–2:50)
See `reflection.md`

---

## Submission Checklist
- [ ] All tests pass: `pytest tests/ -v`
- [ ] `overall_score` implemented
- [ ] `run_regression` implemented  
- [ ] `generate_improvement_log` implemented
- [ ] `exercises.md` completed: golden dataset 20 QA (stratified) + benchmark results + rubric
- [ ] `reflection.md` written: 3 failures with 5 Whys + improvement log + CI/CD strategy
- [ ] `solution/solution.py` copied
