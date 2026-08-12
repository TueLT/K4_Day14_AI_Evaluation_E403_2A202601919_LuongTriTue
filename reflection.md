# Day 14 — Reflection

## Evaluation Report & Failure Analysis

## 1. Benchmark Results Summary

**Run mode: live Gemini RAG.** Checkpointing completed all 20 answers. Due to model-specific quota, E01–M06 use `gemini-3.6-flash` and M07–A03 use `gemini-3.1-flash-lite`. Both models received only the question and retrieved chunks, never the golden expected answer.

| Check | Result |
|---|---|
| Unit tests | 42/42 passed |
| Golden dataset validation | PASS |
| QA distribution | 5 easy, 7 medium, 5 hard, 3 adversarial |
| Corpus coverage | 10/10 documents |
| Gemini artifact / benchmark | 20 answers and 20 benchmark results saved |

| Metric | Average | Interpretation |
|---|---:|---|
| Context Recall | .852 | Good evidence coverage, with gaps on adversarial and multi-policy cases. |
| Context Precision | .944 | Relevant chunks usually rank early. |
| Faithfulness | .584 | Borderline; paraphrases and some unsupported wording reduce lexical overlap. |
| Relevance | .572 | Weakest average; concise policy answers often repeat few question tokens. |
| Completeness | .795 | Most expected conditions are covered. |
| Overall Score | .650 | Eight of twenty cases pass the strict all-three-metrics rule. |

Pass rate is **40.0%**. Among 12 failures: off_topic 10 (83.3%), hallucination 2 (16.7%). Retrieval is generally strong; answer-side lexical metrics reveal both paraphrase sensitivity and a few real routing/coverage gaps.

## 2. Top 3 Worst Failures — 5 Whys

### A01 — out-of-scope medical request (overall .384)

Gemini correctly refused medical diagnosis and offered supported OrbitTech topics. Scores: recall .571, precision .639, faithfulness .296, relevance .143, completeness .714.

1. Symptom: a semantically correct refusal receives the lowest overall score.
2. Why: the medical question and safe refusal intentionally share few content tokens.
3. Why: BM25 ranked the scope chunk third after incidental “diagnosis/plan” matches.
4. Why: relevance and faithfulness are lexical overlap heuristics, not refusal-aware metrics.
5. Root cause/fix: add pre-retrieval scope routing and a safety-aware judge for adversarial cases.

### A03 — false charger premise (overall .404)

Gemini correctly states that the PulsePhone X has no charger in the box. Scores: recall .600, precision 1.000, faithfulness .278, relevance .467, completeness .467.

1. Symptom: a correct false-premise correction scores below threshold.
2. Why: expected answer also mentions the order confirmation, which Gemini omitted.
3. Why: retrieval found the product fact but did not surface all expected supporting policy.
4. Why: the model considered the top-ranked fact sufficient for a concise response.
5. Root cause/fix: retrieve and verify every expected answer slot; add a false-premise completeness check.

### H04 — defective opened return (overall .447)

Gemini correctly removes the restocking fee but says return-shipping responsibility is unavailable. Scores: recall .643, precision .950, faithfulness .476, relevance .438, completeness .429.

1. Symptom: only one of two requested remedies is answered.
2. Why: top-k omitted the paragraph stating that OrbitTech supplies a prepaid label for verified defects.
3. Why: query terms emphasize “opened/14 days” more than “return shipping.”
4. Why: no coverage check verifies that each sub-question has evidence.
5. Root cause/fix: decompose multi-part queries and retrieve/rerank for each answer slot.

## 3. Failure Clustering

| Cluster | Root cause | Failure IDs | Priority |
|---|---|---|---|
| Scope and intent routing | Scope evidence not ranked first; lexical metric is refusal-unaware. | A01 | High |
| Multi-part evidence coverage | Required second-policy paragraph absent from top-k. | H04, M06 | High |
| Metric calibration/paraphrase | Correct concise answers receive low lexical overlap. | A03, E03, E04, H05 | Medium |

If only one cluster can be fixed, prioritize multi-part evidence coverage: unlike a metric-calibration false negative, missing evidence can cause genuinely incomplete customer actions.

## 4. Improvement Log

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|---|---|---|---|---|
| A01 | hallucination | Scope routing and refusal-unaware lexical scoring | Route out-of-scope requests before BM25 and add a safety judge | Open |
| A03 | hallucination | Expected supporting detail omitted | Add answer-slot coverage validation | Open |
| H04 | off_topic | Prepaid-label evidence missing from top-k | Decompose query and rerank per sub-question | Open |

| Suggestion | Target metric | Verification method |
|---|---|---|
| Scope/attack router | Relevance, safety pass rate | Re-run A01/A02 plus new adversarial variants. |
| Multi-part retrieval | Context Recall, Completeness | Confirm H04 retrieves both fee and shipping-label evidence. |
| Semantic judge calibration | Faithfulness, Relevance | Compare lexical and human/LLM labels on paraphrased correct answers. |

## 5. Regression Testing Strategy

Run `run_regression()` for every prompt, model, retrieval, chunking, corpus, or policy update and block when any answer-side average drops by more than 0.05. Safety/privacy failures and fabricated policy must block regardless of aggregate change. Low relevance on a human-verified correct paraphrase should alert and trigger calibration rather than automatically block.

```text
Code/prompt/retrieval change → Offline benchmark → Regression gate → Human review for safety/threshold cases → Deploy
```

## 6. Continuous Improvement Loop

| Priority | Action | Metric expected to improve | Expected impact |
|---:|---|---|---|
| 1 | Decompose multi-part questions and rerank per answer slot. | Context Recall, Completeness | Fewer omitted policy actions. |
| 2 | Route scope/injection requests before normal retrieval. | Relevance, safety | Reliable refusals and privacy behavior. |
| 3 | Add semantic judge/human calibration beside overlap metrics. | Faithfulness, Relevance validity | Fewer false-negative evaluations. |

Add benchmark variants for policy effective dates, account compromise at each order status, defective-return shipping, and prompt injection disguised as a support request.

## 7. Final Reflection

The most surprising result is that several concise, correct Gemini answers fail lexical metrics: A01 correctly refuses medical advice yet scores lowest. Word overlap misses paraphrases, negation, numerical/date logic, answer appropriateness, and safe refusals. Production evaluation should supplement it with claim-level citation checks, semantic/LLM judging calibrated against humans, policy/date assertions, safety classifiers, and online feedback.

The results and traces in this report come from `artifacts/actual_answers.json` and `artifacts/benchmark_results.json`. The artifact records each answer's model, so the mixed Gemini Flash run remains auditable and can serve as a frozen regression baseline.
