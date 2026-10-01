# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 85.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.875 | 0.40 | 1.00 | Tương đối ổn, retriever gom được khá nhiều ý chính |
| Context Precision | 0.712 | 0.30 | 1.00 | Hơi lởm, toàn xếp chunk cần thiết xuống dưới |
| Faithfulness | 0.825 | 0.40 | 1.00 | Kéo được qua 0.8 nhưng vẫn có mấy case bịa chính sách (hallucination) |
| Relevance | 0.832 | 0.50 | 1.00 | Trả lời khá đúng trọng tâm, ít lan man |
| Completeness | 0.810 | 0.35 | 1.00 | Bị miss một vài ngoại lệ (exceptions) ở mấy câu khó |
| Overall Score | 0.822 | 0.41 | 1.00 | Kéo rate xuống chủ yếu do đám câu Hard |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Khoảng 14 cases (toàn bộ Easy và Medium)
- Metrics/cases ở mức Needs Work (0.6–0.8): Khoảng 3 cases
- Metrics/cases ở mức Significant Issues (<0.6): 3 cases (đều rơi vào phần Hard)

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 5% |
| irrelevant | 1 | 5% |
| incomplete | 1 | 5% |
| off_topic | 0 | 0% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:* Vấn đề nằm nhiều nhất ở khâu **retrieval ranking**. Nhìn vào Context Precision (0.712) thấp hơn hẳn so với Recall (0.875), chứng tỏ BM25 lấy được đủ tài liệu nhưng lại vứt các chunk chứa thông tin quan trọng xuống cuối rank. Hậu quả là sinh ra nhiễu, làm model bị rối dẫn đến completeness bị hụt và thỉnh thoảng hallucination. Generation cũng có chút vấn đề ở khoản suy luận logic thời gian.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *Điền:* H05 - The carrier left my USD 1,500 laptop unattended... can I escalate immediately?

**Expected answer:**

> *Điền:* You must start with routine support. Missing packages are not listed as immediate escalation reasons (like safety or fraud).

**Actual answer:**

> *Điền:* Yes, because the item is high value (USD 1,500) and was stolen, you can escalate immediately to Tier 2.

**Scores:** Context Recall: 0.40 | Context Precision: 0.30 | Faithfulness: 0.40 |
Relevance: 0.50 | Completeness: 0.35 | Overall: 0.41

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?[cite: 19]

> *Câu trả lời:* Retriever lấy thiếu cái chunk quy định "Safety, fraud mới được escalate luôn" từ file 09, mà lại gom nguyên một mớ chunk về shipping damage (file 04).

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | AI tự bịa ra rule là hàng giá trị cao thì được escalate luôn. |
| Why 1 | Tại sao symptom xảy ra? | Model ko thấy list các trường hợp được escalate ngay trong prompt. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Retriever (BM25) ko đưa chunk đó vào top 5. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | BM25 match chữ "stolen" và "laptop" quá mạnh vào file shipping, bỏ qua file policy escalation. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Đang xài lexical search thuần, ko hiểu semantic (ngữ nghĩa). |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu mô hình embedding / reranker để hiểu ý định câu hỏi. |

**Root cause từ `find_root_cause()`:**[cite: 19]

> *Paste output:* Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**[cite: 19]

> *Câu trả lời:* Hoàn toàn đồng ý. Score recall với precision lẹt đẹt thế kia thì model ko có data mà trả lời nên đành bịa (faithfulness rớt còn 0.4).

**Proposed fix cụ thể:**[cite: 19]

> *Câu trả lời:* Thay BM25 bằng vector search (dùng embedding) hoặc giữ BM25 nhưng lắp thêm Cross-Encoder Reranker để đẩy chunk policy lên top.

### Failure 2[cite: 19]

**ID và question:**[cite: 19]

> *Điền:* H01 - Order Nov 1st, repair Dec. Policy v1 or v2? Water damage covered?

**Expected answer:**[cite: 19]

> *Điền:* Policy v1 applies (based on purchase date). Water damage is excluded.

**Actual answer:**[cite: 19]

> *Điền:* Policy v2 applies because the repair request is in December. Water damage is covered under accidental damage.

**Scores:** Context Recall: 0.50 | Context Precision: 0.40 | Faithfulness: 0.45 |
Relevance: 0.60 | Completeness: 0.40 | Overall: 0.48[cite: 19]

**Evidence inspection:**[cite: 19]

> *Câu trả lời:* Chunk lấy về có cả policy v1 và v2, nhưng model chọn nhầm mốc thời gian áp dụng.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | AI xác định sai version chính sách áp dụng. |
| Why 1 | Tại sao symptom xẩy ra? | Model hiểu lầm ngày "request repair" là ngày trigger policy. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt không ép model đọc kỹ điều kiện "triggering event is the order-placement date". |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Câu hỏi dạng reasoning (suy luận logic) đang vượt quá khả năng zero-shot của prompt hiện tại. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Thiếu step-by-step thinking trong prompt. |
| Why 5 | Root cause có thể hành động được là gì? | Generation prompt base quá mỏng, thiếu Chain of Thought. |

**Root cause và proposed fix:**[cite: 19]

> *Câu trả lời:* Root cause là Generation logic yếu. Fix bằng cách thêm CoT vào prompt: "Let's think step by step: 1. Determine the order date. 2. Find the applicable policy version based on that date...".

### Failure 3[cite: 19]

**ID và question:**[cite: 19]

> *Điền:* H04 - Return laptop but keep bundled earbuds, full refund?

**Expected answer:**[cite: 19]

> *Điền:* No, the promotional value of the earbuds is deducted from the refund.

**Actual answer:**[cite: 19]

> *Điền:* Yes, you can return the laptop for a refund within the return window.

**Scores:** Context Recall: 0.55 | Context Precision: 0.45 | Faithfulness: 0.60 |
Relevance: 0.65 | Completeness: 0.50 | Overall: 0.58[cite: 19]

**Evidence inspection:**[cite: 19]

> *Câu trả lời:* Retriever bắt được chunk quy định return máy, nhưng lại miss mất chunk nói về "promotional bundle deduction" trong file 03.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời thiếu vế trừ tiền quà tặng (incomplete). |
| Why 1 | Tại sao symptom xảy ra? | Model không nhắc đến vụ trừ tiền. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Chunk chứa rule về bundle không xuất hiện trong context đưa cho model. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | `top_k = 5` là quá ít, không đủ chỗ chứa các điều khoản phụ. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa test thử các mức `top_k` lớn hơn (như 7 hay 10). |
| Why 5 | Root cause có thể hành động được là gì? | Context window bị cấu hình quá nhỏ (chỉ lấy 5 chunks). |

**Root cause và proposed fix:**[cite: 19]

> *Câu trả lời:* Tăng `top_k` lên 10. Các model xịn bây giờ context window rất lớn, không việc gì phải bóp `top_k` ở mức 5 cả.

---

## 3. Failure Clustering[cite: 19]

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | BM25 retrieval ranking quá tệ, thiếu ngữ nghĩa | H05, H04 | High |
| 2 | Prompt sinh văn bản thiếu Chain of Thought | H01 | Medium |
| 3 | Mức top_k=5 quá hẹp để nhét ngoại lệ | H04 | Low |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**[cite: 19]

> *Câu trả lời:* Sửa Cluster 1 (Ranking). Vì nếu hệ thống không lấy được text đúng (hoặc lấy được nhưng nhét tít xuống dưới cùng làm model bị nhiễu), thì dù có làm Prompt xịn đến mấy model cũng không có data mà nói.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001       | irrelevant | Context is missing or irrelevant — improve retrieval | Implement cross-encoder reranker | Open |
| F002       | hallucination | Answer is missing key information — improve generation | Update system prompt with Chain of Thought | Open |
| F003       | incomplete | Context is missing or irrelevant — improve retrieval | Increase top_k to 8-10 chunks | Open |

**Ba improvement suggestions ưu tiên**

1. Tích hợp thêm mô hình Reranker (vd: bge-reranker) ngay sau bước BM25 để xếp hạng lại chunk.
2. Cập nhật System Prompt, nhét thêm lệnh "Think step-by-step" để model suy luận logic tốt hơn.
3. Tăng tham số `top_k` từ 5 lên 8 (hoặc 10) để vớt thêm các ngoại lệ (exceptions) nằm rải rác.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Lắp Reranker | Context Precision | Chạy lại pipeline benchmark, expect Precision sẽ vọt lên > 0.85 |
| Thêm CoT vào Prompt | Faithfulness & Completeness | Chạy `evaluate_answers`, check xem mấy case bịa chính sách (hallucination) có biến mất ko |
| Tăng top_k lên 8 | Context Recall | Recall dự kiến sẽ tăng sát mức 0.95 do context window gom được nhiều ý hơn |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* Nên gắn thẳng vào CI pipeline (ví dụ Github Actions). Mỗi khi có dev mở Pull Request đòi sửa prompt, đổi embedding model hay chỉnh tham số retriever thì script này tự chạy để check xem điểm có bị tụt ko.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:* Hơi gắt nhưng cần thiết. Bot CSKH mà tư vấn sai chính sách bảo hành hay refund là công ty đền ốm. Tuy nhiên với metric kiểu Relevance thì có thể nới lỏng drop xuống 0.1, vì đôi khi bot nói hơi dài dòng xíu nhưng vẫn đúng ý thì cũng ko chết ai.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:* Phải block deploy ngay lập tức nếu `Faithfulness` bị tụt (vì đồng nghĩa với việc bot bắt đầu bịa chuyện). Còn nếu `Context Precision` tụt nhẹ thì chỉ bắn alert lên kênh Slack cho team review, vì miễn nó vẫn tìm ra câu trả lời thì khách hàng chưa bị ảnh hưởng ngay.

## 4. Improvement Log

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | irrelevant | Context is missing or irrelevant — improve retrieval | Implement cross-encoder reranker | Open |
| F002 | hallucination | Answer is missing key information — improve generation | Update system prompt with Chain of Thought | Open |
| F003 | incomplete | Context is missing or irrelevant — improve retrieval | Increase top_k to 8-10 chunks | Open |

### Ba improvement suggestions ưu tiên

1. Tích hợp mô hình Reranker (VD: bge-reranker) sau bước BM25.
2. Sửa lại System Prompt, chèn thêm câu `"Think step-by-step before answering"`.
3. Nới rộng top_k từ 5 lên 8.

### Metric dự kiến thay đổi và cách đo lại

| Suggestion | Target metric | Verification method |
|------------|---------------|----------------------|
| Thêm Reranker | Context Precision | Chạy lại pipeline, expect Precision tăng từ 0.71 lên > 0.85 |
| CoT Prompt | Completeness & Faithfulness | Chạy lại evaluate_answers, expect không còn lỗi hallucination |
| Tăng top_k lên 8 | Context Recall | Recall phải tăng lên sát mức 0.95 do gom được nhiều chunk hơn |

## 5. Regression Testing Strategy

### Câu 1: Khi nào chạy `run_regression()` trong production workflow?

>**Câu trả lời:** Chạy ngay trong CI pipeline (trên Github Actions / Gitlab CI) mỗi khi có ai đó mở Pull Request sửa đổi system prompt, đổi embedding model, hoặc tinh chỉnh thông số của retriever.

### Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?

>**Câu trả lời:** Hơi gắt nhưng hợp lý. Đối với CSKH, tư vấn sai chính sách bảo hành/hoàn tiền là ăn phạt và mất uy tín ngay. Tuy nhiên có thể nới lỏng relevance drop xuống 0.1 vì đôi khi AI trả lời hơi lan man chút xíu nhưng vẫn đúng thì chả sao.

### Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?

>**Câu trả lời:** Block deployment nếu Faithfulness bị drop (vì nó tương đương với việc bot bắt đầu bịa chuyện). Chỉ alert qua Slack nếu Context Precision tụt nhẹ, vì miễn model vẫn tìm thấy câu trả lời thì user ko bị ảnh hưởng ngay lập tức.

### Câu 4: Điền evaluation stages vào flow.

```text
Code/prompt/retrieval change → [Run Unit/Eval Tests] → [Run Regression vs Baseline] → [Manual Human Review for Edge Cases] → Deploy
```

>**Giải thích:** Unit tests để check code ko lỗi, Regression test để đảm bảo bộ metrics ko bị tụt điểm so với bản cũ. Cuối cùng, nếu pass hết thì lead vẫn liếc qua mấy cái edge cases trước khi ấn nút deploy thật.

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|----------|--------|---------------------------|-----------------|
| 1 | Lắp Reranker | Context Precision | Giảm hẳn nhiễu trong top-k, model trả lời dứt khoát hơn |
| 2 | Sửa Prompt CoT | Faithfulness | Khắc phục mấy cái lỗi nhầm lẫn ngày tháng / version policy |
| 3 | Tuning lại chunk size | Context Recall | Model ko bị miss các exception ở cuối file |

### Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?

>**Câu trả lời:** Cần thêm vài case về việc tính toán số ngày hoàn tiền (VD: hàng giao hôm thứ 6, tính thêm 3 business days thì có bị tính cả T7, CN ko). Và thêm case khách hàng nổi giận chửi bậy xem AI có bị jailbreak ko.

## 7. Final Reflection

### Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?

>**Câu trả lời:** Lúc đầu nghĩ Retriever BM25 kém lắm, recall chắc khá thấp. Nhưng Recall tận 0.87 (tức là nó tìm thấy khá đủ text), nhưng Precision lại cực tệ. Hóa ra vấn đề ko phải là tìm ko ra, mà là tìm ra rồi nhưng lại xếp nhầm thứ tự ưu tiên.

### Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?

>**Câu trả lời:** Quá cứng nhắc. Dùng word-overlap thì ví dụ khách dùng từ đồng nghĩa (VD: expected là "laptop", answer là "computer") thì hệ thống chấm điểm thấp vì không trùng token, dù ý nghĩa y hệt nhau. Lên production em sẽ giảm bớt word-overlap này và thay bằng LLM-as-a-Judge (như Ragas auth hoặc DeepEval) hoặc dùng Cosine Similarity của Embedding để chấm điểm ngữ nghĩa.

