# FundAdvisor: Architecture for Interactive Investment Guidance

---

## 1. Introduction

We present FundAdvisor, an AI system designed to guide customers toward suitable fund selections through interactive conversation. The system addresses the three core challenges posed in the task brief: iterative data collection under uncertainty, principled stopping criteria, and tractable recommendation with auditable decision-making.

A live demo is available at:

> **https://cuneatic-unflurried-lily.ngrok-free.dev/c/bebc913e-3568-4dcd-b600-90c71d51a99b**

The demo shows the system's output for the query *"I have 10000 gbp to invest for 1 year"*, demonstrating how FundAdvisor's questionnaire-driven approach outperforms a naive embedding retrieval baseline.

I built an end-to-end pipeline using the design decisions described below. The four key contributions are:

1. **LLM-driven questionnaire generation**: the foundation of the entire system. By eliciting structured investor preferences through adaptive, conversational questions, the system acquires the precise information needed for the deterministic scorer to select funds that match the user's needs. Without this questionnaire, the system would be reduced to keyword matching or generic retrieval which is very likely unable to distinguish a capital-preservation investor from a growth-seeking one.
2. **RAG-powered news article retrieval**: relevant market news articles are retrieved per recommended fund and presented alongside the Investment Policy Statement, giving the investor current market context to refer to when reviewing their personalised recommendations.
3. **Naive retrieval baseline**: a second, independent fund-selection method using semantic similarity, serving as a baseline comparison to demonstrate the effectiveness of the questionnaire-driven approach.
4. **LLM-as-a-Judge evaluation**: an automated pairwise quality assessment drawing from the AutoMetrics framework (Ryan et al., 2025), comparing the primary method against the baseline in real time. As shown in the live demo, our method constantly scores higher than the baseline.

---

## 2. Iterative Data Collection: LLM-Driven Questionnaire Generation

### 2.1 The Problem

Retail investors arrive with incomplete, uncertain information about their own financial situation. A rigid, form-based intake process risks either overwhelming users with irrelevant questions or under-collecting critical data. The system must ask the right question at the right moment.

The LLM-driven questionnaire is not merely a data-collection convenience, it is the mechanism that makes accurate fund selection possible by proactively asking user necessary questions.

### 2.2 Approach: Adaptive LLM-Generated Questions

FundAdvisor models the investor profile as a 7-dimension state vector (`DimensionState`):

| Dimension | Type | Example Values |
|---|---|---|
| Risk tolerance | float 0–1 | 0.2 (conservative) → 0.85 (aggressive) |
| Time horizon | integer (years) | 1, 5, 10, 20 |
| Goal type | enum | growth, income, balanced, preservation |
| ESG constraints | list of keywords | ["fossil fuels", "solar energy"], or [] |
| Currency preference | enum | GBP, USD, either |
| Liquidity needs | enum | low, medium, high |
| Tax wrapper | enum | ISA, SIPP, LISA, GIA, unsure |

On every conversational turn, the system performs two LLM calls:

1. **Extraction**: A structured extraction prompt parses all prior user messages and populates whichever dimensions it can infer. Critically, this is stateless and re-reads the full conversation history on each turn, which means a single rich user message (e.g., "I have £10k, 1 year, high risk, ISA, GBP") can fill 4–5 dimensions at once. There is no fixed question ordering.

2. **Question generation**: If any dimensions remain `null`, the LLM generates a single, natural follow-up question targeting the first missing field. The generation prompt is parameterised by a field description (e.g., "their risk tolerance means asking what percentage portfolio drop they could stomach in a bad year without panic-selling") and constrained to be conversational and concise (maximum 2 sentences). Fallback questions are hardcoded for robustness if the LLM call fails.

### 2.3 Why LLM-Driven, Not Template-Based

- **Adaptive ordering**: unlike a fixed questionnaire, the system adapts to whatever the user volunteers. If a user's opening message reveals their goal, risk tolerance, and horizon, the system skips those questions entirely.
- **Natural conversation**: each question is generated in the context of the conversation history, producing a coherent dialogue rather than a clinical checklist.

Noted that if the LLM is unavailable, hardcoded fallback questions ensure the system never stalls.

### 2.4 Design Considerations for Iterative Data Collection

- **Latency**: two LLM calls per turn (extraction + question generation) adds latency compared with a pure rule-based approach. Mitigated by using a lightweight model (aka Gemini 3.1 Flash Lite).
- **Extraction reliability**: natural language is ambiguous, some answer such as "I'm fine with some risk" could map to 0.5 or 0.7. The extractor prompt includes some calibration anchors (e.g., "comfortable with 30% drop" will be explicitly mapped to 0.6 in prompt). There might be biases. In production, a confirmation step before finalising the profile should be performed to calibrate this.
- **Auditability**: every dimension is extracted as a structured JSON field from the user's own words, stored in a dataclass. The full conversation is the audit trail; the structured profile is the decision input. No hidden state, so the whole process is reproducible.

---

## 3. Stopping Criteria

The stopping criterion is deterministic: the system proceeds to recommendation **if and only if all 7 dimensions are non-null**. This avoids the unstability/unauditability of a learned or heuristic stopping condition. So the user always knows how many questions remain. Also edge cases are covered: `esg_constraints = []` (user explicitly has no exclusions) and `tax_wrapper = "unsure"` both count as filled. `None` is reserved for "not yet discussed."

---

## 4. Tractable Recommendation: Deterministic Weighted Scoring

FundAdvisor uses a deterministic weighted-sum scorer that evaluates every fund in the catalog against the investor's profile.

Each sub-score is computed from scoring functions. They are:

- **Risk**: User's preferred risk rating (1–7).
- **Goal**: Investment goal (growth/income/balanced/preservation) against fund category (equity/thematic/fixed-income/cash).
- **Horizon**: ratio of user's horizon to fund's stated horizon where perfect match yields 1.0.
- **Currency**: preference matching (GBP user -> GBP fund 1.0, USD 0.6).
- **Liquidity**: categorical scoring based on fund type and user's access needs.
- **ESG**: hard gate: any ESG keyword match in the fund's objective scores the fund at 0.0 (eliminated).

This scoring function is what transforms the questionnaire's structured output into a fund ranking. This also shows the merit of questionaire: without the questionnaire, the scorer has nothing to compute with.

After scoring, the system selects the top 5 funds by score.

### 4.1 LLM-Assisted Allocation (Optional)

The LLM can also be used for allocation percentages and rationale prose for a more fine-grained investment. It receives the pre-selected funds and is explicitly instructed not to change the selection. This preserves auditability (the fund choice is deterministic and replayable from the profile) while leveraging the LLM's ability to allocate fund percentages.

If the LLM fails or returns invalid JSON, the system falls back to proportional allocation by hard-coded suitability score.

### 4.2 Investment Policy Statement (IPS) Generation

A second LLM call generates a structured markdown IPS with six mandated sections (Investor Profile, Asset Allocation table, Selected Instruments, Rebalancing Strategy, ESG Criteria, Caveats). This gives the user a formal, auditable document alongside the recommendation, and future rebalancing guidance.

### 4.3 Design Trade-offs

- **Weight selection**: the weights currently are hand-tuned. In production, these could be calibrated against historical suitability assessments or compliance officer judgments, and can be validated by back-testing.
- **No cross-fund interactions**: each fund is scored independently. The scorer does not model portfolio-level properties like correlation or sector concentration.

---

## 5. News Article Retrieval for Informed Decision-Making

A recommendation and its Investment Policy Statement are more useful if the investor can contextualise them with current market information. FundAdvisor enriches each recommended fund with relevant news articles retrieved from a RAG backend, so the user can refer to current market commentary while reading their personalised investment profile and fund recommendations.

This design choice reflects the insight that investors do not make decisions out of vaccum. Even for retail investors, they could benefit from knowing, for example, that a recommended bond fund's sector has recently faced volatility, or that a recommended equity fund's market has seen positive momentum. The news articles provide this temporal context and can be useful in the cross-check.

---

## 6. Evaluation: LLM-as-a-Judge with Baseline Comparison

### 6.1 Motivation: Why Automated Evaluation Matters

Evaluating recommendation quality is hard and subjective. There is no single "correct" portfolio for a given retail investor's profile. Human expert evaluation is the gold standard but does not scale, i.e., are expensive.

My approach to automated evaluation is motivated by the AutoMetrics framework (Ryan et al., 2026), which demonstrates that automatically generated LLM-based evaluation metrics can closely approximate human judgments when given well-calibrated rubrics and explicit scoring anchors. The AutoMetrics methodology proposes a multi-stage pipeline for discovering and calibrating task-specific evaluation criteria, showing that structured LLM-as-a-judge evaluation can achieve high correlation with human assessments across diverse NLP tasks. FundAdvisor adapts this principle to the financial recommendation domain by defining domain-specific evaluation dimensions and strict score calibration guidelines.

### 6.2 Baseline: Embedding Retrieval

To demonstrate the value of the questionnaire-driven approach, FundAdvisor implements a second, independent fund selection method as a **baseline**:

- **Method**: embed the Investment Policy Statement text using `qwen/qwen3-embedding-8b` via OpenRouter, embed all fund profiles (name + category + risk + currency + objective), retrieve the top-5 funds by cosine similarity.
- **Allocation**: equal-weight 20% per fund (no suitability-aware allocation).
- **Rationale**: this baseline represents what a naive semantic-retrieval system would select: "find funds whose descriptions are closest to the investment brief." It directly tests whether structured questionnaire-driven profiling adds value over raw text similarity.

The baseline is relatively simple, whose purpose is not to be a strong competitor but to serve as a **controlled comparison** that shows the contribution of the questionnaire. The key insight is: without the structured profile elicited by the questionnaire, a system can only match text, so it cannot reason about risk tolerance, goal alignment, or constraint satisfaction in a principled way.

### 6.3 Pairwise LLM-as-a-Judge Evaluation

An independent evaluator LLM (`google/gemini-3-flash-preview` which is more powerful than the `google/gemini-3.1-flash-lite-preview` and good enough in judging winners) receives:

- The full investor profile (all 7 dimensions)
- Method A's selections with suitability scores and allocations (Our Method: User Questionnaire + weighted suitability scoring)
- Method B's selections (Baseline: embedding retrieval, equal weight)

And evaluates both on four dimensions:

| Dimension | What It Judges |
|---|---|
| **Suitability** | Do the selected funds match risk tolerance, goal type, and time horizon? |
| **Diversification** | Is the portfolio spread across asset classes, sectors, and currencies? |
| **Constraint compliance** | Are ESG, currency, liquidity, and tax wrapper requirements respected? |
| **Risk calibration** | Is the overall portfolio risk appropriate for this investor? |

The evaluator prompt uses strict score calibration adapted from the AutoMetrics approach of well-defined rubrics with anchored scoring:

- 9–10: exceptional, reserved for truly outstanding portfolios
- 5–6: adequate but with notable gaps
- 1–2: fundamentally unsuitable
- "DO NOT default to high scores. Most portfolios are 5–8, not 9–10."

### 6.4 Live Demo Results

The live demo (https://cuneatic-unflurried-lily.ngrok-free.dev/c/bebc913e-3568-4dcd-b600-90c71d51a99b) shows a complete interaction starting from the query *"I have 10000 gbp to invest for 1 year"*. After the questionnaire completes the investor profile, the LLM-as-a-Judge evaluation confirms:

> **Quality Evaluation:** Our Method (6.2) **>** Baseline (3.8)

---

## 7. System Design

![System Design](./SysDesign.svg)

---

## 8. Regulatory Considerations

- **Auditability**: fund selection is fully deterministic. Given the same profile and catalog, the scorer always produces the same top-5. The LLM only narrates (allocation + IPS), never chooses.
- **Traceability**: the DimensionState is a structured JSON record extracted from the user's own words. The conversation itself is the audit trail.

---

## 9. References

[1] Ryan, M. J., Zhang, Y., Salunkhe, A., Chu, Y., Xu, D., & Yang, D. (2025). *AutoMetrics: Approximate Human Judgments with Automatically Generated Evaluators* [Software, v1.0.0]. GitHub. https://github.com/XenonMolecule/autometrics. To appear in *Proceedings of the International Conference on Learning Representations (ICLR 2026)*. https://openreview.net/forum?id=ymJuBifPUy

---

## Appendix A: Technology Stack

| Component | Technology |
|---|---|
| UI Framework | OpenWebUI (Pipe interface) |
| LLM Provider | OpenRouter API (AsyncOpenAI client) |
| Recommendation Model | google/gemini-3.1-flash-lite-preview |
| Evaluator Model | google/gemini-3-flash-preview |
| Embedding Model | qwen/qwen3-embedding-8b |
| News Retrieval | Custom RAG backend and dataset |
| KIID Documents | FastAPI static file server |
| Language | Python 3.11 |

