# Evaluation and guardrails detail — Resolution Intelligence

Supporting detail for the [main case study](README.md). This document goes one level deeper into how the team evaluated quality and what guardrails the pipeline enforces. Verified against the team's current evaluation artifacts in [github.com/venkat1596/resolution-intelligence-capstone](https://github.com/venkat1596/resolution-intelligence-capstone) — treat every number below as a prototype evaluation result against a public dataset, not a production outcome.

## Data tiering as a guardrail

Before retrieval quality or generation quality was ever measured, the team made a product decision about what counts as recommendable knowledge:

| Tier | Approx. size | Treatment |
|---|---|---|
| Gold solved (`FIXED`, `WORKSFORME`) | ~174,000 | Eligible as a recommended resolution |
| Duplicate-linked | 40,946 | Retrieval/clustering supervision only |
| `WONTFIX` | 41,646 | Policy/limitation evidence only, never a recommended fix |
| Open / unresolved | 21,589 | Situational awareness only, never recommended |

An open or `WONTFIX` issue can look highly relevant to a retrieval system and still be actively wrong to recommend as a solution. Keeping these tiers separate is a guardrail that happens before any model is involved.

## Retrieval evaluation

- **Ground truth:** 40,946 duplicate-link relationships in the Eclipse dataset were used as retrieval supervision — if issue A is a known duplicate of solved issue B, a good retriever should surface B for A.
- **Metrics:** recall@k, MRR, and nDCG against that duplicate-link ground truth, evaluated separately across query types (natural/clean, synthetic duplicate-closure, and technical-independent queries).
- **Candidate-depth audit:** a targeted study asking whether recall failures were a *coverage* problem (correct answer never retrieved at all) or a *ranking* problem (correct answer retrieved but not promoted to the top-k shown to the model). On the technical-independent query set, Recall@200 (0.701) was close to oracle Recall@10 (0.693) pulled from the same candidate pool — meaning the correct answer was usually already in the retrieved pool; the gap was in promotion, not coverage. That finding shaped where the team spent optimization effort (reranking and field-aware indexing) rather than just widening the candidate pool.

## End-to-end (RAGAS-style) evaluation

The canonical end-to-end suite runs 26 cases — 13 expected to route autonomously (high confidence) and 13 expected to route to human review (low confidence) — through the live pipeline (`/api/analyze`) and scores each on:

- **Faithfulness** — do generated claims trace back to cited evidence?
- **Answer relevance** — does the recommendation actually address the ticket?
- **Context precision** — how much of the retrieved evidence was actually relevant?
- **Context recall** — how much of the relevant evidence was actually retrieved?

Latest results (from the team's most recent post-optimization regression run):

| Metric | Autonomous bucket (13 cases) | Human-review bucket (13 cases) |
|---|---|---|
| Faithfulness | 0.977 | 0.952 |
| Answer relevance | 0.969 | — |
| Context precision | 0.991 | ~0.0 (by design, on the low-confidence demo case) |
| Context recall | 0.992 | ~0.0 (by design, on the low-confidence demo case) |
| Routing accuracy | 13/13 | 13/13 |

Near-zero context precision/recall on the low-confidence bucket isn't a failure — it's the intended behavior: when there's no real matching evidence, the system reports that honestly instead of forcing a citation to something loosely related, and the classifier routes the case to a human instead of auto-resolving it.

*Note: an earlier evaluation report in the repository shows lower, slightly different numbers from before a later optimization and cache-consistency pass. This case study uses the most recently regenerated results, consistent with the precedence the team's own repository gives to current artifacts over earlier ones.*

## Latency and quality tradeoff

A quality-gated optimization pass cut end-to-end synthesis latency by roughly 41% (46.7s → 27.4s on the autonomous path; 50.3s → 32.7s on the human-review path) while holding faithfulness, other RAGAS metrics, and routing accuracy steady — evaluated as a before/after regression on the same 26-case suite, not as separate uncontrolled runs.

## Guardrails in production terms

- **Citation validity checks** run after generation: every claim is checked against real evidence IDs from the pack, with automatic sanitization and conditional repair when a citation doesn't resolve — a deterministic check rather than trusting the model's self-reported citations.
- **Confidence-based routing** is the primary safety mechanism: cases are classified into autonomous vs. human-review paths using confidence, severity, and evidence gaps, not left to the model to decide when to hedge.
- **Gated, low-trust external evidence** — the StackOverflow fallback (via a Qwen-cached, write-through ChromaDB store, populated from Tavily web search) only activates when internal evidence is thin on an exact identifier, and is treated as lower-trust than internal ticket history rather than blended in as equivalent.
- **Tracing and observability** — every pipeline branch, token count, latency figure, and failure is traced (Arize/Phoenix), so a quality regression can be isolated to retrieval, aggregation, or reasoning instead of showing up only as a worse final answer.

## Known evaluation gap

The team identified — but has not yet fully implemented — a point-in-time evaluation methodology to guard against resolution leakage: because ticket histories in the dataset include fields that changed after resolution, a retrieval or evaluation setup that isn't careful about "as-of" cutoffs could inadvertently use information that wouldn't have existed at ticket-open time. Current evaluation favors immutable fields (summary/description) where possible, but this isn't yet a strictly enforced, leakage-safe methodology. It's on the [future roadmap](README.md#future-product-roadmap).
