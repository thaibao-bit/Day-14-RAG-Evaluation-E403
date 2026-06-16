# Day 14 - Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Paste results từ Exercise 3.2 và tóm tắt:

**Overall pass rate:** `15%`

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.33 | 0.00 | 0.64 | 0.17 |
| Relevance | 0.73 | 0.25 | 0.90 | 0.16 |
| Completeness | 0.67 | 0.32 | 1.00 | 0.18 |
| Overall Score | 0.58 | 0.40 | 0.76 | 0.09 |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metrics ở Good (0.8-1.0)? `0`
- Bao nhiêu metrics ở Needs Work (0.6-0.8)? `8`
- Bao nhiêu metrics ở Significant Issues (<0.6)? `12`

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 8 | 40% |
| irrelevant | 1 | 5% |
| incomplete | 0 | 0% |
| off_topic | 8 | 40% |
| refusal | 0 | 0% |

---

## 2. Top 3 Worst Failures - 5 Whys Analysis

Theo bài giảng: "Phân loại failure TRƯỚC KHI fix. Đừng fix từng failure riêng lẻ - CLUSTER rồi fix root cause."

### Failure 1

**Question:** *What is the CEO of a company not mentioned in the knowledge base?*

**Agent Answer:** *The knowledge base does not mention the CEO, so I do not know from the provided materials.*

**Scores:** Faithfulness: `0.08` | Relevance: `0.57` | Completeness: `0.53` | Overall: `0.40`

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Faithfulness rất thấp dù câu trả lời có vẻ an toàn |
| Why 1 | Tại sao xảy ra? | Câu trả lời đúng ý, nhưng wording không khớp nhiều với context nguồn |
| Why 2 | Tại sao wording không khớp? | Agent dựa vào generic refusal template thay vì trích sát từ context |
| Why 3 | Tại sao agent dùng template chung? | Prompt generation ưu tiên "safe response" hơn grounded response |
| Why 4 | Root cause là gì? | Chưa có cơ chế buộc refusal/out-of-scope responses vẫn phải bám vào evidence được cấp |

**Root cause (from `find_root_cause()`):**
> `Context is missing or irrelevant - improve retrieval`

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Đồng ý một phần. Faithfulness thấp thật sự cho thấy overlap với context rất kém, nhưng ở case này vấn đề lớn hơn là generation template hóa, không hẳn là hiệu quả retrieval hay grounding rule.

**Proposed fix (cụ thể, actionable):**
> Thêm mẫu refusal có điều kiện: "Based on the provided materials, I cannot identify...". Đồng thời thêm eval riêng cho out-of-scope cases để không phạt nhầm safe answer thành hallucination lexical.

---

### Failure 2

**Question:** *How would you investigate a sudden drop in pass rate after a prompt change?*

**Agent Answer:** *To investigate a sudden drop in pass rate after a prompt change, compare benchmark deltas, inspect regressions by failure type, and trace behavior changes.*

**Scores:** Faithfulness: `0.17` | Relevance: `0.73` | Completeness: `0.45` | Overall: `0.45`

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Answer khá đúng hướng nhưng thiếu grounding và chưa đầy đủ |
| Why 1 | Tại sao completeness dưới 0.5? | Bỏ sót bước "add new failure cases back into benchmark" |
| Why 2 | Tại sao bỏ sót? | Agent trả lời ngắn gọn hơn expected reference |
| Why 3 | Tại sao faithfulness cũng thấp? | Context chưa chứa nhiều cụm từ trùng exact wording với answer |
| Why 4 | Root cause là gì? | Prompt chưa ép answer phải quote lại các bước phân tích trong source workflow |

**Root cause:**
> Generation chưa được ràng buộc theo checklist, nên vừa thiếu ý quan trọng vừa không grounded sát context.

**Proposed fix:**
> Dùng answer template theo checklist: compare deltas -> cluster failures -> trace retrieval/generation -> add cases to benchmark. Cho model trích hoặc paraphrase sát hơn với context.

---

### Failure 3

**Question:** *Should a chatbot use RAG or fine-tuning for a policy handbook that changes weekly?*

**Agent Answer:** *For a policy handbook that changes weekly, a chatbot should use RAG because the knowledge changes frequently.*

**Scores:** Faithfulness: `0.18` | Relevance: `0.80` | Completeness: `0.43` | Overall: `0.47`

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Answer đúng conclusion nhưng quá ngắn và bỏ sót trade-off |
| Why 1 | Tại sao completeness thấp? | Chưa nêu rõ "không cần retraining model weights" |
| Why 2 | Tại sao faithfulness thấp? | Answer không lặp lại các term trong context như "fine-tuning", "stable behavior", "style adaptation" |
| Why 3 | Tại sao bỏ sót trade-off? | Agent ưu tiên trả lời một phía thay vì so sánh hai lựa chọn |
| Why 4 | Root cause là gì? | Prompt cho câu hỏi so sánh chưa bắt buộc format "recommendation + rationale + trade-off" |

**Root cause:**
> Prompting cho comparison questions quá ngắn, không buộc model cover cả recommendation và lý do đối lập.

**Proposed fix:**
> Thêm few-shot cho dạng "A hay B" và bắt answer theo khung: choice, reason, when the alternative is better.

---

## 3. Failure Clustering

Theo bài giảng: "Fix 1 root cause giải quyết nhiều failures cùng lúc."

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | Answers đúng hướng nhưng không grounded sát context wording | 8 | High |
| 2 | Comparison/process questions bị thiếu checklist quan trọng | 6 | High |
| 3 | Adversarial và out-of-scope responses chưa có template refusal grounded | 3 | Medium |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Chọn cluster 1. Đây là cụm lớn nhất và kéo faithfulness xuống trên nhiều nhóm câu hỏi khác nhau. Sửa grounding rule và retrieval-aware prompting sẽ tác động đến nhiều failure nhất.

---

## 4. Improvement Log (from `generate_improvement_log`)

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | irrelevant | Answer does not address the question - improve prompt clarity | Implement stricter grounding checks and improve retrieval quality before generation | Open |
| F002 | off_topic | Context is missing or irrelevant - improve retrieval | Refine prompts and routing so answers stay tightly aligned to the user question | Open |
| F003 | off_topic | Context is missing or irrelevant - improve retrieval | Add intent detection and topic validation before finalizing responses | Open |
| F004 | off_topic | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F005 | off_topic | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F006 | hallucination | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F007 | off_topic | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F008 | hallucination | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F009 | hallucination | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F010 | hallucination | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F011 | hallucination | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F012 | off_topic | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F013 | hallucination | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F014 | hallucination | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F015 | off_topic | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F016 | hallucination | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
| F017 | off_topic | Context is missing or irrelevant - improve retrieval | Investigate failure pattern | Open |
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Implement stricter grounding checks and improve retrieval quality before generation.
2. Refine prompts and routing so answers stay tightly aligned to the user question.
3. Add intent detection and topic validation before finalizing responses.

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> Trước mỗi merge vào `main`, sau mỗi prompt change, sau khi thay retriever/chunker/reranker, và trước release candidate. Đây là những điểm thay đổi dễ gây metric drift nhất.

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> Có. Với benchmark nhỏ 20 cases, `0.05` là mức để nhìn thấy sự giảm có ý nghĩa mà chưa quá nhạy. Nếu đây là support bot high-stakes, tôi sẽ strict hơn cho faithfulness, ví dụ `0.03`.

**Câu 3: Khi phát hiện regression - block deployment hay chỉ alert?**
> Block deployment nếu regression rơi vào faithfulness hoặc recall của nhóm high-risk cases. Chỉ alert nếu regression nhỏ ở completeness cho nhóm low-risk. Trade-off là block giúp an toàn hơn, nhưng alert giữ velocity cho thay đổi không quan trọng.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**

```text
Code change -> Unit tests -> Offline eval + regression check -> Human review for risky cases -> Deploy
```

---

## 6. Continuous Improvement Loop

Theo bài giảng: Evaluate -> Analyze -> Improve -> Augment (add to benchmark) -> lặp lại

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Buộc answer phải cite/paraphrase sát retrieved context | Faithfulness | Giảm failure hallucination/off_topic do wording lệch context |
| 2 | Thêm prompt templates cho comparison và process questions | Completeness | Giảm bỏ sót checklist ở hard questions |
| 3 | Bổ sung adversarial refusal templates grounded trong context | Faithfulness + Safety | Xử lý out-of-scope và prompt injection ổn định hơn |

**Bạn sẽ thêm failure cases nào vào benchmark cho sprint tiếp theo?**
> Thêm 3 nhóm: comparison questions cần nêu trade-off rõ ràng, out-of-scope questions cần refusal grounded, và regression cases sau prompt change có wording gần đúng nhưng thiếu 1 bước checklist.

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** `RAGAS-inspired heuristic`

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**
> Tôi sẽ chọn RAGAS cho lớp metric retrieval/grounding và kết hợp DeepEval cho gating trong CI/CD. Nếu buộc chọn 1 framework chính, tôi nghiêng về DeepEval vì nó vào pytest flow rất tự nhiên cho team Python.

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | Team cần vừa gate regression vừa viết test rõ ràng cho các tình huống nguy cơ |
| CI/CD integration vì... | Pytest-native để đưa vào GitHub Actions, Jenkins, hoặc pre-merge jobs rất dễ |
| Team workflow vì... | Dev quen test-driven flow, nên eval sẽ được coi như một phần của bộ test thay vì một script bên ngoài |
