# Module 7 Integrated Evaluation Report — Fine-Tuning vs. Pre-Trained Inference

> The Module 7 deliverable. Synthesizes Lab 7A (fine-tuning), Integration 7A (domain shift), Lab 7B (QA), and Integration 7B (summarization).

---

## 1. Comparison Table

| Task | Approach | Model | Training cost | Inference cost | Quality metric | Value |
|---|---|---|---|---|---|---|
| Sentiment classification (Lab 7A) | Fine-tuning | DistilBERT (`distilbert-base-uncased`) | ~30 min CPU + 7,472 labeled reviews | ~50 ms / example | Macro-F1 | 0.62 |
| Domain transfer (Integration 7A) | Fine-tuned model out-of-domain | (same) | Already trained | ~50 ms / example | Domain-shift judgment | Confidence collapse: mean prob 0.43, max 0.66; neutral never predicted across 1,033 articles |
| Extractive QA (Lab 7B) | Pre-trained inference | distilbert-base-cased-distilled-squad | 0 | ~50 ms / example | EM / token-F1 | 0.34 / 0.46 |
| Summarization (Integration 7B) | Pre-trained inference | distilbart-cnn-6-6 | 0 | ~3 sec / example | ROUGE-1 / 2 / L F1 | 0.37 / 0.16 / 0.27 |

## 2. Findings

- **Fine-tuning produces the highest in-domain quality but transfers poorly.** The DistilBERT classifier achieved macro-F1 of 0.62 on app-review sentiment, but suffered complete confidence collapse on tech news (Integration 7A): mean predicted probability of 0.43, zero neutral predictions, and no confidence threshold above 0.66 — making the model unusable out-of-domain without retraining.

- **Pre-trained QA covers the right region but struggles with span boundaries.** The 12-point gap between EM (0.34) and token-F1 (0.46) on 1,000 tech-news QA examples indicates the model frequently extracts spans with correct semantic content but incorrect boundaries — it locates the answer region but not precisely enough for exact extraction tasks.

- **Summarization ROUGE numbers are moderate and mask faithfulness failures.** ROUGE-1 of 0.37 and ROUGE-L of 0.27 on 120 tech-news articles are usable for low-stakes summarization, but the lowest-ROUGE example (NEWS_0042, ROUGE-L: 0.08) shows the model summarizing a completely different topic — a failure ROUGE signals but cannot explain.

- **Pre-trained inference has zero training cost but unpredictable failure modes.** Both the QA and summarization models required no labeled data or training time. But their failure modes — distractor entity selection in QA, topic drift in summarization — are harder to characterize and fix than a fine-tuned model's errors, which are measurable against a known label distribution.

- **The fine-tuning vs. pre-trained tradeoff is a data availability question.** Where labeled data exists (app reviews), fine-tuning produced a calibrated, task-specific model. Where it does not (news QA, summarization), pre-trained inference is the practical starting point — but the domain-shift results are a warning that transfer requires validation, not assumption.

## 3. Faithfulness Check

### Example A — High ROUGE (NEWS_0079, ROUGE-L: 0.62)

> **Reference summary:** Heidi Pratt was rushed to a hospital in Costa Rica for undisclosed illness. Pratt is currently appearing on "I'm a Celebrity... Get Me Out of Here". Pratt had blamed "the devil" for wanting to leave show earlier in week.
>
> **Predicted summary:** Heidi Pratt was taken to a hospital for an undisclosed illness, her publicist says. Pratt has been competing on NBC's "I'm a Celebrity ... Get Me Out of Here". Pratt and her husband, Spencer, tried to quit the show earlier in the week.
>
> ROUGE-1: 0.64; ROUGE-2: 0.39; ROUGE-L: 0.62
>
> **Faithful? Yes.** Every claim in the predicted summary — the hospital visit, the show name, the quit attempt — is grounded in the article. ROUGE correctly identifies this as high quality: n-gram overlap reflects genuine semantic alignment, not coincidental vocabulary sharing.

### Example B — Mid ROUGE (NEWS_0083, ROUGE-L: 0.24)

> **Reference summary:** NEW: At concert, Michelle Obama urges young people to get involved now. Concert features the Jonas Brothers, Miley Cyrus, Bow Wow and Demi Lovato. Billy Ray Cyrus, Keke Palmer and Usher make introductions. Did you miss the show? It's available later on Disney.com.
>
> **Predicted summary:** Michelle Obama calls on young people to play a part in their country's future now. Thousands of kids danced and yelled in Washington's Verizon Center. Miley Cyrus, Demi Lovato, Bow Wow, the Jonas Brothers and others performed. Michelle Obama: "Are we ready to go?"
>
> ROUGE-1: 0.42; ROUGE-2: 0.18; ROUGE-L: 0.24
>
> **Faithful? Yes.** The predicted summary omits the Disney.com detail and the introductory speakers but does not hallucinate. ROUGE penalizes it for structural differences — bullet-style reference vs. flowing predicted prose — rather than factual error, illustrating how ROUGE conflates format mismatch with content mismatch.

### Example C — Low ROUGE (NEWS_0042, ROUGE-L: 0.08)

> **Reference summary:** One high-profile film producer has fired hundreds of personal assistants. Naomi Campbell's treatment of her employees has landed her in court. And would you really want to answer to Simon Cowell or Al Capone?
>
> **Predicted summary:** New grads are looking for jobs near their college or hometowns. Indianapolis, Philadelphia and Cincinnati are the top cities for new grads. The list is based on the ranking of the top U.S. cities with the highest concentration of young adults.
>
> ROUGE-1: 0.13; ROUGE-2: 0.00; ROUGE-L: 0.08
>
> **Faithful? No — complete topic drift.** The predicted summary describes a job-market article for new graduates; the reference article is about celebrity employers and workplace abuse. The model appears to have summarized a different portion of a concatenated or mis-tokenized document, likely due to input truncation at the model's 1,024-token limit. This is the canonical failure ROUGE catches indirectly (near-zero overlap) but cannot diagnose: a ROUGE-L of 0.08 signals catastrophic failure but not its cause.

## 4. Production Decision Matrix

| Scenario | Recommendation | Justification |
|---|---|---|
| Real-time app store review triage dashboard for a product team | **Fine-tune** | The Lab 7A classifier achieved macro-F1 of 0.62 on in-domain app reviews with ~30 min training; pre-trained inference has no equivalent in-domain calibration, and a product team's triage decisions require a measurable, domain-specific error rate to set alert thresholds. |
| Daily tech / entertainment news summary digest for an internal newsroom | **Pre-trained inference** | ROUGE-1 of 0.37 and ROUGE-L of 0.27 are sufficient for a low-stakes internal digest where editors scan summaries before acting; zero training cost and ~3s per article make distilBART immediately deployable without labeled summarization data. |
| Domain-expert QA on legal contracts | **Fine-tune** | The pre-trained QA model's 34% EM on news text would degrade further on legal language, and distractor entity errors (picking the wrong party name or date) carry direct legal risk; fine-tuning on labeled contract QA pairs with a no-answer abstention mechanism is the minimum viable production path. |

## 5. What You Would Do Differently

If a labeled summarization dataset for tech/entertainment news were available, the highest-value investment would be fine-tuning `sshleifer/distilbart-cnn-6-6` on domain-specific article-summary pairs rather than relying on CNN/DailyMail-trained weights. The topic-drift failure in NEWS_0042 (ROUGE-L: 0.08) suggests the model's input truncation behavior is misaligned with tech-news article structure — tech articles often bury the lede, whereas CNN/DM articles front-load key information. Fine-tuning on tech-news pairs would teach the model which portions of a tech article carry summary-worthy content. Beyond retraining, I would add a lightweight NLI-based faithfulness filter checking whether each predicted summary is entailed by the source article, routing low-entailment summaries to human review rather than publishing automatically — a practical guard against the topic-drift failure mode that ROUGE cannot catch.

## 6. Limits of the Evaluation

The two most important limits for the production scenarios in Section 4 are faithfulness measurement and latency under load. ROUGE measures n-gram overlap, not factual accuracy — the NEWS_0042 topic-drift failure scored ROUGE-L of 0.08 and was caught, but a summary that hallucinated a single entity name while reusing most of the article's vocabulary could score ROUGE-L above 0.5 and pass undetected; for the newsroom digest, this is the operative publishing risk. The second limit is that all inference times are single-request, warm-cache, CPU-only measurements: the ~3s per article figure for summarization does not reflect throughput under concurrent load, where batching strategy, hardware, and queue depth dominate the latency profile. Neither limit is addressable with the current evaluation setup; both require production-environment load testing and a faithfulness audit before deployment decisions in Section 4 can be treated as final.