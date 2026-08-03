# Evaluation and guardrails detail — Resolution Intelligence

Supporting detail for the [main case study](README.md). This document goes one level deeper into how the team evaluated quality and what guardrails the pipeline enforces. Verified against the team's current evaluation artifacts in [github.com/venkat1596/resolution-intelligence-capstone](https://github.com/venkat1596/resolution-intelligence-capstone) — treat every number below as a prototype evaluation result against a public dataset, not a production outcome.

## Data tiering as a guardrail

Before retrieval quality or generation quality was ever measured, the team made a product decision about what counts as recommendable knowledge. `FIXED` + `WORKSFORME` together (~174,428 of 301,464 records) form the broader answer-eligible pool, but the team split that pool further into a strict default tier and weaker supporting tiers, implemented as an explicit `solution_gold` / `solution_silver` / `weak_resolution` role enum in code ([`src/resolution_intelligence/retrieval/knowledge_filters.py`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/src/resolution_intelligence/retrieval/knowledge_filters.py)):

| Tier | Approx. size | Treatment |
|---|---|---|
| `solution_gold` | 109,278 | Default recommend-eligible solution index |
| `solution_silver` (archived / text-thin fixed issues) | ~41,756 | Lower-weighted secondary solution index |
| `weak_resolution` (`WORKSFORME`) | 23,394 | Weak-resolution index, lower confidence |
| Duplicate-linked | 40,946 | Retrieval/clustering supervision only |
| `WONTFIX` | 41,646 | Policy/limitation evidence only, never a recommended fix |
| Open / unresolved | 21,589 | Situational awareness only, never recommended |

Source: [`reports/data_quality_indexing_audit/data_quality_indexing_audit.md`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/data_quality_indexing_audit/data_quality_indexing_audit.md) ("Stricter Gold/Silver Split" section).

An open or `WONTFIX` issue can look highly relevant to a retrieval system and still be actively wrong to recommend as a solution — and even within "solved," a well-documented gold fix carries more confidence than a sparse `WORKSFORME` workaround. Keeping these tiers separate is a guardrail that happens before any model is involved.

## Retrieval evaluation

- **Ground truth:** 40,946 duplicate-link relationships in the Eclipse dataset were used as retrieval supervision — if issue A is a known duplicate of solved issue B, a good retriever should surface B for A.
- **Metrics:** recall@k, MRR, and nDCG against that duplicate-link ground truth, evaluated separately across query types (natural/clean, synthetic duplicate-closure, and technical-independent queries).
- **Candidate-depth audit:** a targeted study asking whether recall failures were a *coverage* problem (correct answer never retrieved at all) or a *ranking* problem (correct answer retrieved but not promoted to the top-k shown to the model). On the technical-independent query set, Recall@200 (0.701) was close to the oracle Recall@10 achievable by reranking within that same 200-candidate pool (0.693) — meaning the correct answer was usually already in the retrieved pool; the gap was in promotion, not coverage. That finding shaped where the team spent optimization effort (reranking and field-aware indexing) rather than just widening the candidate pool. Source: [`reports/evaluation/candidate_depth_audit/technical_qwen_issue/summary.json`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/evaluation/candidate_depth_audit/technical_qwen_issue/summary.json).

## End-to-end (RAGAS-style) evaluation

The canonical end-to-end suite runs 26 cases — 13 expected to route autonomously (high confidence) and 13 expected to route to human review (low confidence) — through the live pipeline (`/api/analyze`) and scores each on:

- **Faithfulness** — do generated claims trace back to cited evidence?
- **Answer relevance** — does the recommendation actually address the ticket?
- **Context precision** — how much of the retrieved evidence was actually relevant?
- **Context recall** — how much of the relevant evidence was actually retrieved?

Latest results (from the team's most recent post-optimization regression run), averaged from per-case scores in [`reports/evaluation/canonical_ragas/queries_26_ragas.json`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/evaluation/canonical_ragas/queries_26_ragas.json):

| Metric | Autonomous bucket (13 cases) | Human-review bucket (13 cases) |
|---|---|---|
| Faithfulness | 0.977 | 0.952 |
| Answer relevance | 0.969 | — |
| Context precision | 0.991 | ~0.0 (by design, on the low-confidence demo case) |
| Context recall | 0.992 | ~0.0 (by design, on the low-confidence demo case) |
| Routing accuracy | 13/13 | 13/13 |

Routing accuracy (26/26 overall) is independently confirmed in a separate consistency run comparing each case's expected route against the live pipeline's actual output (`verdict_ok: true` for all 26 rows) — see [`reports/workflow/optimization/consistency_results.json`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/workflow/optimization/consistency_results.json).

Near-zero context precision/recall on the low-confidence bucket isn't a failure — it's the intended behavior: when there's no real matching evidence, the system reports that honestly instead of forcing a citation to something loosely related, and the classifier routes the case to a human instead of resolving it without review.

*Note: an earlier evaluation report in the repository shows lower, slightly different numbers from before a later optimization and cache-consistency pass. This case study uses the most recently regenerated results, consistent with the precedence the team's own repository gives to current artifacts over earlier ones.*

## Latency and quality tradeoff

The team's evaluation write-up reports that a quality-gated optimization pass cut end-to-end synthesis latency by roughly 41% (46.7s → 27.4s on the autonomous path; 50.3s → 32.7s on the human-review path) while holding faithfulness, other RAGAS-style metrics, and routing accuracy steady, evaluated as a before/after regression on the same 26-case suite. Source: [`reports/evaluation/canonical_ragas/canonical_ragas_report.tex`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/evaluation/canonical_ragas/canonical_ragas_report.tex). This figure exists only in that narrative report — no separate generated benchmark data file reproduces the exact before/after numbers at this commit, so treat it as the team's reported result rather than one independently re-derived from raw data here.

## Guardrails vs. evaluation metrics

These are two different mechanisms and worth keeping distinct. **Runtime guardrails** change what the pipeline does; **RAGAS-style evaluation** (above) measures quality afterward and, in the current iteration, doesn't feed back into runtime behavior.

Runtime guardrails:

- **Citation validity checks** run after generation: every claim is checked against real evidence IDs from the pack, with automatic sanitization and conditional repair when a citation doesn't resolve — a deterministic check rather than trusting the model's self-reported citations. ([`src/resolution_intelligence/guardrails/reasoning.py`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/src/resolution_intelligence/guardrails/reasoning.py))
- **Confidence/risk-based routing policy** is the primary safety mechanism: the core workflow's classifier (`classify_and_route`) assigns cases to a fully autonomous, semi-autonomous (human approval), or human-led path using confidence, severity, and evidence gaps — this is an enforced policy gate, not a metric the model can talk its way around. ([`src/resolution_intelligence/workflow/stages.py`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/src/resolution_intelligence/workflow/stages.py)) The canonical evaluation set and the lightweight UI demo path exercise a simplified autonomous/human-review split rather than all three routes.
- **Gated, low-trust external evidence** — the StackOverflow fallback (via a Qwen-cached, write-through ChromaDB store, populated from Tavily web search) only activates when internal evidence is thin on an exact identifier, and is treated as lower-trust than internal ticket history rather than blended in as equivalent. ([`src/resolution_intelligence/retrieval/stackoverflow_cache.py`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/src/resolution_intelligence/retrieval/stackoverflow_cache.py), [`src/resolution_intelligence/adapters/tavily_web.py`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/src/resolution_intelligence/adapters/tavily_web.py))

Observability, not a behavior-changing guardrail:

- **Tracing** — most pipeline stages (validation/redaction, retrieval, evidence assembly, aggregation, reasoning, routing) carry Arize/Phoenix spans with tokens, latency, and failures visible per stage, so a quality regression can usually be isolated to a specific stage instead of showing up only as a worse final answer. Feedback capture is not currently instrumented with its own span. ([`src/resolution_intelligence/observability/arize_phoenix.py`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/src/resolution_intelligence/observability/arize_phoenix.py))

## Known evaluation gap

The team identified — but has not yet fully implemented — a point-in-time evaluation methodology to guard against resolution leakage: because ticket histories in the dataset include fields that changed after resolution, a retrieval or evaluation setup that isn't careful about "as-of" cutoffs could inadvertently use information that wouldn't have existed at ticket-open time. Current evaluation favors immutable fields (summary/description) where possible, but this isn't yet a strictly enforced, leakage-safe methodology. It's on the [future roadmap](README.md#future-product-roadmap).
