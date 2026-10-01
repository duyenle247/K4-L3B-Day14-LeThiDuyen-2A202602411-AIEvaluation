# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 30.0% (6 / 20 passed)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.784 | 0.107 | 0.962 | Khả năng bao phủ tài liệu của retriever rất tốt, hầu hết các câu hỏi factual đều đạt > 0.85. |
| Context Precision | 0.916 | 0.000 | 1.000 | Xếp hạng chunk xuất sắc, chunk liên quan luôn được đưa lên rank 1 hoặc rank 2 trong top 5. |
| Faithfulness | 0.617 | 0.275 | 1.000 | Căn cứ tương đối tốt vào context, ít xuất hiện hallucination nghiêm trọng. |
| Relevance | 0.482 | 0.231 | 0.692 | Điểm yếu nhất, câu trả lời trích dẫn rộng làm loãng từ khóa cốt lõi đối với câu hỏi. |
| Completeness | 0.655 | 0.267 | 1.000 | Độ hoàn chỉnh khá tốt, đáp ứng được hầu hết ý chính của expected answer. |
| Overall Score | 0.584 | 0.321 | 0.744 | Điểm trung bình ở mức cần cải thiện (needs work), cần tối ưu prompt sinh câu trả lời. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Precision (0.916), các cases E01 (0.735), M06 (0.744), H02 (0.726).
- Metrics/cases ở mức Needs Work (0.6–0.8): Context Recall (0.784), Completeness (0.655), Faithfulness (0.617).
- Metrics/cases ở mức Significant Issues (<0.6): Relevance (0.482), Overall Score trung bình (0.584), các cases M01 (0.321), M05 (0.384).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 5.0% |
| irrelevant | 2 | 10.0% |
| incomplete | 0 | 0.0% |
| off_topic | 11 | 55.0% |
| refusal | 0 | 0.0% |

*(Lưu ý: 6/20 cases đạt Passed chiếm 30.0%)*

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **khâu Generation**, không phải Retrieval.
Dẫn chứng:
1. Retrieval metrics đạt mức rất cao: **Context Precision đạt 0.916** và **Context Recall đạt 0.784**, chứng tỏ BM25 retriever đã tìm đúng tài liệu cần thiết và đặt đúng các chunk chứa bằng chứng quan trọng lên đầu danh sách.
2. Ngược lại, **Relevance chỉ đạt 0.482** (kéo theo 11 ca bị đánh nhãn `off_topic` vì không vượt qua ngưỡng 0.5). Generator có xu hướng đưa thêm các câu ngữ cảnh phụ xung quanh câu trả lời chính, làm pha loãng tỷ lệ từ vựng trùng khớp trực tiếp với câu hỏi.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *Điền:* `M01` — "Can a customer return an opened AeroBuds Pro package if they decide they do not like the sound quality?"

**Expected answer:**

> *Điền:* "No. AeroBuds Pro are in-ear audio products and hygiene accessories; opened ear tips and in-ear audio products are non-returnable unless they are defective."

**Actual answer:**

> *Điền:* "The AeroBuds Pro are wireless earbuds supplied with a charging case and three ear-tip sizes. The AeroBuds Pro and separately purchased OrbitTech accessories have a 12-month warranty. OrbitTech does not authorize a carrier to leave a signature-required package unattended."

**Scores:** Context Recall: 0.600 | Context Precision: 1.000 | Faithfulness: 0.429 | Relevance: 0.267 | Completeness: 0.267 | Overall: 0.321

**Evidence inspection:** Retriever lấy đúng các chunks từ `01_product_catalog.md` và `06_warranty_policy.md`, nhưng đoạn văn về quy định vệ sinh không đổi trả từ `05_returns_and_exchanges.md` bị xếp sau các chunk mô tả tính năng sản phẩm do câu hỏi chứa nhiều keyword tên sản phẩm.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Actual answer liệt kê thông số kỹ thuật và bảo hành thay vì trả lời thẳng câu hỏi có được trả hàng hay không. |
| Why 1 | Tại sao symptom xảy ra? | Câu trả lời không trích dẫn quy định ngoại lệ về phụ kiện vệ sinh cá nhân/tai nghe in-ear đã mở seal. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Đoạn văn chứa điều khoản cấm đổi trả tai nghe in-ear có điểm lexical match thấp hơn đoạn mô tả sản phẩm AeroBuds Pro. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | BM25 chỉ khớp từ khóa bề mặt "AeroBuds Pro" thay vì hiểu ngữ nghĩa của ý định "muốn trả hàng vì không thích chất âm". |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống thiếu bộ phân tách ý định (intent classifier) và chưa áp dụng Hybrid Dense Retrieval. |
| Why 5 | Root cause có thể hành động được là gì? | Cần áp dụng query expansion/hybrid retrieval và bổ sung rule prompt: "Luôn bắt đầu câu trả lời bằng khẳng định Yes/No rõ ràng". |

**Root cause từ `find_root_cause()`:**

> *Paste output:* "Answer does not address the question — improve prompt clarity"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Đồng ý. Mặc dù retrieval có thể cải thiện thêm, nguyên nhân trực tiếp làm câu trả lời hỏng là do generator không trả lời câu hỏi đóng (Can a customer return...?) mà lại chép thông tin mô tả sản phẩm và bảo hành vào câu trả lời.

**Proposed fix cụ thể:**

> *Câu trả lời:* Cập nhật system prompt: "For yes/no questions, begin with an unambiguous 'Yes' or 'No' and directly state the controlling policy before providing supporting details."

---

### Failure 2

**ID và question:**

> *Điền:* `M05` — "If an order paid with both a credit card and an OrbitTech gift card is returned, how and when will the refund be issued?"

**Expected answer:**

> *Điền:* "After inspection, refunds are processed within 5 to 7 business days to the original payment methods. The card portion is refunded to the credit card, and the gift-card portion is issued as a replacement gift card."

**Actual answer:**

> *Điền:* "Customers may pay by supported credit or debit card, OrbitTech gift card, or bank transfer. Up to two gift cards may be combined with one card payment. OrbitTech cannot refund cash for a gift-card-funded portion; that amount returns to a replacement gift card."

**Scores:** Context Recall: 0.842 | Context Precision: 1.000 | Faithfulness: 0.480 | Relevance: 0.357 | Completeness: 0.316 | Overall: 0.384

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy rất tốt chunk từ `02_orders_and_payments.md` và `05_returns_and_exchanges.md`, nhưng generator lại chọn các câu nói về cách thanh toán ban đầu thay vì trích xuất vế thời gian "within 5 to 7 business days".

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời nói về cách phối hợp thẻ khi mua hàng và thiếu hẳn mốc thời gian hoàn tiền (5–7 business days). |
| Why 1 | Tại sao symptom xảy ra? | Generator tập trung vào quy tắc thanh toán thay vì quy trình xử lý sau khi trả hàng. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Chunk thanh toán `02_orders` có mật độ từ "credit card" và "gift card" cao nên chiếm ưu thế trong prompt. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt không nhắc nhở mô hình phải trả lời đầy đủ cả hai vế "how" (bằng cách nào) và "when" (khi nào). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống không có bước phân rã câu hỏi đa phần (multi-part question decomposition). |
| Why 5 | Root cause có thể hành động được là gì? | Cần thiết kế prompt hướng dẫn mô hình kiểm tra trả lời đủ các câu hỏi phụ (both mechanism and timeframe). |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause: "Answer does not address the question — improve prompt clarity". Proposed fix: Yêu cầu mô hình trả lời theo cấu trúc danh sách gồm 2 mục rõ ràng: "Phương thức hoàn trả (How)" và "Thời gian xử lý (When)".

---

### Failure 3

**ID và question:**

> *Điền:* `E02` — "What is the minimum purchase amount required for OrbitPay instalments, and what is the payment schedule?"

**Expected answer:**

> *Điền:* "OrbitPay instalments require an eligible device purchase of at least USD 300 after discounts, with 25% due at checkout followed by three equal monthly payments."

**Actual answer:**

> *Điền:* "OrbitPay instalments are available for eligible device purchases of at least USD 300 after discounts. OrbitTech cannot refund cash for a gift-card-funded portion; that amount returns to a replacement gift card. Work begins only after approval and required payment."

**Scores:** Context Recall: 0.842 | Context Precision: 0.804 | Faithfulness: 0.393 | Relevance: 0.556 | Completeness: 0.474 | Overall: 0.474

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy đúng chunk về OrbitPay trong `02_orders_and_payments.md`, nhưng generator lại bỏ sót câu lịch thanh toán 25% và ghép nhầm một câu về gift card refund và work quote từ chunk khác.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Trả lời được số tiền USD 300 nhưng thiếu lịch trả góp 25% + 3 tháng, đồng thời chèn câu không liên quan. |
| Why 1 | Tại sao symptom xảy ra? | Generator bị phân tâm bởi các câu chứa từ khóa thanh toán khác trong cùng ngữ cảnh. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Các câu trong chunk được nối liền nhau và mô hình cắt ghép thiếu chọn lọc. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa có bước kiểm tra chéo (self-consistency check) giữa câu hỏi và câu trả lời sinh ra. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Evaluator chạy offline sau khi hoàn thành, không có validation guardrail chặn câu trả lời lỗi ở runtime. |
| Why 5 | Root cause có thể hành động được là gì? | Thắt chặt system prompt để trích xuất đầy đủ toàn bộ mệnh đề đi kèm định lượng thay vì ngắt câu giữa chừng. |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause: "Answer is missing key information — increase context window or improve generation". Proposed fix: Tinh chỉnh generator để khi phát hiện câu hỏi về "schedule / timeline", phải trích xuất đầy đủ cả tỷ lệ phần trăm ban đầu và số kỳ thanh toán.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1. Diluted Relevance / Verbosity | Generator trích dẫn thêm các câu xung quanh không liên quan, làm loãng tỷ lệ từ khóa trực tiếp của câu hỏi. | E02, E03, M02, M03, M05, H01, H02, H03, H05, A02, A03 | High |
| 2. Multi-hop Synthesis Missing | Generator không kết hợp đủ thông tin từ 2 tài liệu khác nhau vào cùng một câu trả lời hoàn chỉnh. | M01, M05 | High |
| 3. Partial Hallucination / Sentence Mixing | Ghép các câu từ các chunk khác nhau gây hiểu lầm hoặc giảm faithfulness. | E04 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:* Chọn **Cluster 1 (Diluted Relevance / Verbosity)** vì cluster này chiếm tới 11 trên tổng số 14 ca thất bại (gần 79% tổng số lỗi). Chỉ cần tối ưu system prompt để generator trả lời ngắn gọn, trực diện, loại bỏ các câu ngữ cảnh phụ là có thể nâng pass rate từ 30% lên trên 80% ngay lập tức.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```markdown
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Tune retriever similarity threshold to filter irrelevant chunks | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Refine system prompt and add intent detection to stay on topic | Open |
| F004 | irrelevant | Multiple issues detected — review full pipeline | Improve prompt clarity to directly address user questions | Open |
| F005 | off_topic | Answer is missing key information — increase context window or improve generation | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F007 | off_topic | Answer is missing key information — increase context window or improve generation | Implement hallucination checker to filter unsupported claims | Open |
| F008 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F009 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F010 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F011 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F012 | irrelevant | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F013 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F014 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
```

**Ba improvement suggestions ưu tiên**

1. Tinh chỉnh system prompt để trả lời trực diện câu hỏi, áp dụng intent detection và loại bỏ các câu trích dẫn rườm rà.
2. Thêm guardrail kiểm tra hallucination để lọc các câu ghép không có căn cứ từ context trước khi trả về cho người dùng.
3. Tăng chunk size và tối ưu ranh giới ngữ nghĩa của chunk để tránh phân mảnh thông tin quy trình đa bước.

| Suggestion | Target metric | Verification method |
|---|---|---|
| 1. Refine system prompt & intent detection | Answer Relevance (kỳ vọng tăng từ 0.482 lên > 0.65) | Chạy lại `evaluate_answers.py` và theo dõi tỷ lệ lỗi `off_topic` giảm xuống < 3 cases. |
| 2. Implement Hallucination Guardrail | Faithfulness (kỳ vọng tăng từ 0.617 lên > 0.80) | Chạy lại `run_regression()` so sánh số ca có faithfulness < 0.5. |
| 3. Context Chunk Boundary Optimization | Completeness (kỳ vọng tăng từ 0.655 lên > 0.80) | Đánh giá lại trên nhóm câu hỏi Hard (H01–H05) trong Golden Dataset. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* Chạy `run_regression()` tự động trong CI/CD pipeline trước mỗi lần merge code (Pull Request), mỗi khi thay đổi system prompt, thay đổi mô hình LLM, hoặc cập nhật lại tài liệu corpus/retriever. Ngoài ra, cần chạy định kỳ hàng tuần để kiểm tra tính ổn định.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:* Rất phù hợp. Trong lĩnh vực hỗ trợ khách hàng và chính sách bảo hành/hoàn tiền, mức sụt giảm 5% (0.05) có thể dẫn đến việc hàng chục khách hàng nhận thông tin sai lệch về điều khoản bồi thường, gây thiệt hại tài chính và khiếu nại nghiêm trọng.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment:** Faithfulness bị giảm quá 0.05 hoặc dưới 0.70; xuất hiện lỗi `hallucination` trên các chính sách bảo hành; vi phạm an toàn trên các ca `adversarial` (A01–A03).
> - **Alert Only:** Context Recall hoặc Completeness bị giảm nhẹ (< 0.05) trên các câu hỏi mở, hoặc Relevance dao động nhẹ nhưng tổng thể vẫn đảm bảo độ chính xác cốt lõi.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit Testing (pytest)] → [Offline Evaluation (Golden Dataset)] → [Regression Gate (run_regression <= 0.05 drop)] → Deploy
```

> *Giải thích:* Thay đổi trước hết phải vượt qua Unit Tests để đảm bảo syntax/logic không lỗi, sau đó chạy toàn bộ 20 QA trong Golden Dataset để đo điểm 5 metrics, và cuối cùng chạy qua cổng Regression Gate so sánh với bản baseline đang chạy production; nếu đạt mới kích hoạt CD deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cải tiến system prompt: thêm cấu trúc trả lời trực diện và cấm trích dẫn thừa. | Relevance & Pass Rate | Giảm 80% lỗi `off_topic`, nâng pass rate từ 30% lên > 75%. |
| 2 | Nâng cấp Retriever sang Hybrid Search (BM25 + Semantic Embedding). | Context Recall | Cải thiện Context Recall từ 0.784 lên > 0.90 cho các câu hỏi phức tạp. |
| 3 | Tích hợp Few-shot examples vào prompt cho các trường hợp ngoại lệ. | Completeness | Đảm bảo không sót điều kiện 10% restocking fee hay quy định OrbitPlus 45 ngày. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Edge Case ngày chuyển tiếp chính sách:** Đơn hàng đặt đúng ngày 01/09/2026 (ngày v2.0 bắt đầu có hiệu lực) để kiểm tra mô hình có phân định chính xác ranh giới `>= Sept 1` hay không.
> 2. **Trường hợp kết hợp nhiều khuyến mãi:** Khách hàng dùng cả mã giảm giá phần trăm kết hợp với thẻ thành viên OrbitPlus trên phụ kiện để kiểm tra quy tắc không cộng dồn (non-stacking).
> 3. **Adversarial Jailbreak nâng cao:** Người dùng đóng vai nhân viên bảo hành OrbitTech nội bộ yêu cầu cung cấp tài liệu kỹ thuật mật.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Ban đầu tôi dự đoán khâu Retrieval sẽ là điểm nghẽn lớn nhất vì kho tài liệu chứa nhiều chính sách tương tự nhau. Tuy nhiên kết quả thực tế cho thấy BM25 Retriever hoạt động cực kỳ chính xác với Context Precision lên tới **0.916**. Bất ngờ lớn nhất là tỷ lệ pass rate chỉ đạt 30% chủ yếu do khâu Generation bị phạt điểm bởi metric **Relevance** (do mô hình trả lời quá dài và đưa thêm câu ngữ cảnh phụ).

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:* 
> - **Giới hạn của Word-overlap:** Quá phụ thuộc vào việc trùng lặp từ vựng chính xác, không hiểu được từ đồng nghĩa (synonyms), diễn đạt lại (paraphrasing), hoặc câu phủ định mang nghĩa đối lập nhưng chung nhiều từ khóa.
> - **Thay thế/bổ sung trong Production:**
>   1. Dùng **Embedding Semantic Similarity** (Cosine similarity qua mô hình embedding) để đo ngữ nghĩa thay vì đếm từ.
>   2. Dùng **LLM-as-a-Judge** với rubric phân cấp 1–5 đã thiết kế ở Exercise 3.3 để đánh giá đa chiều (Correctness, Completeness, Safety).
>   3. Sử dụng các framework đánh giá chuyên sâu như **RAGAS** hoặc **DeepEval** với các metric chuẩn hóa: Faithfulness (qua NLI / claim verification), Answer Relevancy và Hallucination Metric.
