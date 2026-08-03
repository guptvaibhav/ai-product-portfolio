# Resolution Intelligence for Customer Escalations

An evidence-grounded support copilot for Tier-1 engineers working already-escalated tickets — built as a team capstone for Maven's *Building Agentic AI Applications with a Problem-First Approach*.

> **Team project.** This case study documents the product thinking behind a capstone built with a team. See [Team attribution and my contribution](#team-attribution-and-my-contribution) before reading claims about "we" vs. "I."

**Original implementation:** [github.com/venkat1596/resolution-intelligence-capstone](https://github.com/venkat1596/resolution-intelligence-capstone) · **Supporting docs:** [architecture.md](architecture.md) · [evaluation-and-guardrails.md](evaluation-and-guardrails.md)

---

## Executive overview

When a support ticket gets escalated, a Tier-1 engineer usually has to reconstruct context by hand — searching past tickets, docs, and logs for anything that looks similar before they can even start diagnosing. Resolution Intelligence is a prototype copilot that does that reconstruction for them: it retrieves relevant historical issues, assembles them into a structured, cited evidence pack, and only then produces a resolution recommendation — one that has to point back to specific evidence or say it doesn't have enough. Low-confidence or high-risk cases are routed to a human instead of getting an answer anyway.

The team prototyped and evaluated this against a public dataset of **301,464 Eclipse issue-tracker records** as a stand-in for enterprise ticket history, since no enterprise support corpus was available for a course project.

## Target user

Tier-1 support engineers handling tickets that have already been escalated — past the point of a scripted first-response flow, where the engineer needs real technical context to move the case forward.

## User problem

Escalated tickets rarely come with the context needed to resolve them. The engineer has to manually search for prior similar issues, relevant documentation, and logs, then piece together whether anything applies before diagnosis can even begin. That reconstruction step competes with actually solving the ticket.

*(The team's problem framing assumed this manual reconstruction is a meaningful share of resolution time — that assumption motivated the project but isn't a measurement from this dataset or any live support desk, and the portfolio doesn't present it as one.)*

## Why AI and retrieval are appropriate

The underlying task — "has something like this happened before, and what worked?" — is fundamentally a search-and-synthesis problem, not a generation problem. A large, mostly-unread archive of historical tickets is exactly the kind of high-recall, hard-to-manually-search asset that retrieval-augmented generation is built for: dense embeddings surface issues that are semantically similar even when the wording differs, and exact-token search catches the stack traces and error strings that embeddings tend to miss.

## Why this is not simply a chatbot

A chat interface that answers questions about tickets would be easy to build and easy to get wrong in ways that erode trust. The product decisions here are specifically about *not* being a free-form chatbot:

- **Evidence-grounded generation, checked after the fact** — the reasoning step runs over a deterministically assembled evidence pack, and every generated claim is checked against real evidence IDs after generation, with citations sanitized or repaired when they don't resolve — a downstream control, not just an instruction to the model to "please cite sources."
- **Evidence IDs, not prompt-only citation requests** — each retrieved item gets a stable ID at assembly time, so citation checking is a lookup, not a judgment call.
- **Exact-token fallback** — stack traces, error codes, and identifiers are matched with SQLite FTS5 lexical search alongside dense retrieval, because semantic similarity alone misses exact strings that matter most for technical diagnosis.
- **Confidence-gated routing, not a universal answer** — the core workflow classifies cases into three routes (fully autonomous, semi-autonomous with human approval, human-led review); the system is designed to say "I don't have enough evidence" and escalate rather than produce a plausible-sounding guess.
- **Feedback as evaluation data, not silent retraining** — engineer feedback is captured and stored for evaluation; the first iteration deliberately does not auto-retrain on it.

## Product principles

1. Every recommendation must cite evidence by ID — no citation, no claim.
2. Retrieval quality is evaluated separately from generation quality, so failures can be traced to the right stage.
3. Solved knowledge, duplicate signals, and open/negative-outcome issues are different data tiers and are never treated as interchangeable (see [Data and retrieval strategy](#data-and-retrieval-strategy)).
4. Low confidence should route to a person, not produce a more confident-sounding answer.
5. External, lower-trust sources are gated and clearly separated from the internal evidence base.

## Scope and non-goals

**In scope:** a single-tenant, local prototype that ingests a structured ticket, retrieves and evaluates evidence from a public issue-tracker corpus, and produces a cited recommendation with routing and feedback capture.

**Explicitly out of scope:**

- Production deployment, multi-tenant isolation, or authentication.
- Real enterprise ticket data — the Eclipse public dataset is an evaluation stand-in, not a claim about performance on live support data.
- Automated model retraining from feedback (deliberately deferred past the first iteration).
- General-purpose chat or open-ended Q&A over the ticket corpus.

## Three-iteration product strategy

The team scoped the build as three iterations of increasing capability, each shipping something evaluable rather than building the full system before testing anything:

**Iteration 1 — Evidence retrieval and structured evidence pack.** Ingest a ticket, retrieve candidate historical issues with dense (Qwen) and lexical (FTS5) search, and assemble a structured, source-attributed evidence pack. No generation yet — this iteration is deliberately about proving retrieval quality before spending effort on synthesis.

**Iteration 2 — Grounded reasoning and cited resolution synthesis.** Add an aggregation stage that deduplicates and scores evidence, then a reasoning stage that produces a resolution recommendation, candidate solutions, and confidence — with every claim required to trace back to an evidence ID.

**Iteration 3 — Risk-aware routing, human review, feedback capture, and improvement.** Add a classifier/router that assigns cases to a fully autonomous, semi-autonomous (human approval), or human-led path based on confidence and severity, capture structured engineer feedback, layer in RAGAS-style automated quality evaluation, and add a gated, low-trust external fallback (StackOverflow, via a cache-first, write-through source) for cases where internal evidence is thin.

The current repository reflects the union of all three iterations — the architecture diagram below is the Iteration 3 state.

## Current architecture

![Resolution Intelligence system architecture — Iteration 3](assets/system-architecture.png)

*Diagram from the team's original repository (`reports/architecture/updated_system_diagram.png`), reproduced here for context.*

At a high level: a ticket is normalized and put through deterministic sensitive-data redaction, used to build both semantic and lexical retrieval queries, run against the Eclipse SQLite store (dense + FTS5 + typed-graph relationships, with a gated StackOverflow fallback), assembled into a scored evidence pack, aggregated, reasoned over by an LLM that must cite evidence, routed by a confidence-aware classifier, and closed with structured feedback capture. Runtime guardrails (citation validation, routing policy) enforce behavior inline; RAGAS-style evaluation and Arize/Phoenix tracing run alongside the pipeline as offline quality checks rather than inside it. Full stage-by-stage detail is in [architecture.md](architecture.md).

## Data and retrieval strategy

About 174,000 of the 301,464 Eclipse records (`FIXED` + `WORKSFORME`) form the broader pool of *answer-eligible* solved knowledge — but the team didn't treat that pool as one undifferentiated tier. It's split further into a strict default-recommend tier and weaker supporting tiers:

| Tier | Approx. size | Treatment |
|---|---|---|
| `solution_gold` | 109,278 | Default recommend-eligible solution index |
| `solution_silver` (archived / text-thin fixed issues) | ~41,756 | Lower-weighted secondary solution index |
| `weak_resolution` (`WORKSFORME`) | 23,394 | Weak-resolution index — supporting evidence, lower confidence |
| Duplicate-linked | 40,946 | Used as retrieval/clustering supervision, not recommended directly |
| `WONTFIX` | 41,646 | Kept as policy/limitation evidence only — never recommended as a fix |
| Open / unresolved | 21,589 | Shown for situational awareness only — never recommended |

This matters because a retrieval system that can't tell a fixed issue from an abandoned or still-open one will confidently recommend things that were never actually solutions — and even within "solved," a well-documented gold fix isn't the same reliability signal as a sparse `WORKSFORME` workaround. Retrieval itself is hybrid: **Qwen3-Embedding-0.6B** dense vectors for semantic similarity (with a documented `text-embedding-3-small` OpenAI-embedding path for CPU-only setups, selected via an alternate retrieval index rather than the default Docker Compose configuration), **SQLite FTS5** for exact-token matching on stack traces and identifiers, and typed graph relationships (`duplicate_of`, `depends_on`, `blocks`, `see_also`) for structured issue context beyond plain similarity.

## Evidence-pack concept

Rather than passing raw retrieved text into a prompt and hoping the model cites it correctly, the pipeline assembles a structured evidence pack first: each retrieved item gets a stable evidence ID, a relevance score, and provenance (which retrieval method surfaced it, and from where). The reasoning stage can only make claims that reference those IDs, and a downstream check validates that citations actually resolve to real evidence before a recommendation is returned — a deterministic control instead of trusting the model to self-report.

## Guardrails

Runtime guardrails change what the pipeline does; RAGAS-style evaluation measures quality afterward without changing pipeline behavior. Both matter, but they're different mechanisms (detail in [evaluation-and-guardrails.md](evaluation-and-guardrails.md)):

- **Citation validity checks (runtime)** — generated claims are checked against real evidence IDs, with sanitization and conditional repair when a citation doesn't resolve.
- **Confidence/risk-based routing policy (runtime)** — the classifier enforces which of three routes a case takes; this is a behavior-changing gate, not just a reported metric.
- **Gated, low-trust external fallback (runtime)** — the StackOverflow web source only engages when internal evidence is thin on an exact match, is cached write-through in a local vector store, and is treated as lower-trust than internal ticket history.
- **RAGAS-style automated evaluation (offline)** — faithfulness, answer relevance, context precision, and context recall are computed on saved workflow output for quality reporting; they don't feed back into runtime pipeline behavior in the current iteration.
- **Tracing and observability** — most pipeline stages are traced (validation/redaction, retrieval, evidence assembly, aggregation, reasoning, routing) with tokens, latency, and failures visible per stage (Arize/Phoenix); feedback capture itself is not currently instrumented with its own trace span.

## Human-in-the-loop design

The product concept defines three routing buckets — **fully autonomous** (low risk, high confidence), **semi-autonomous** (known issue, requires human approval before acting), and **human-led** (high risk or low confidence) — and the core workflow's classifier implements and tests all three. The canonical evaluation suite and the lightweight UI demo path currently exercise a simplified two-way split (autonomous vs. human-review); the semi-autonomous middle path exists and is unit-tested but isn't separately represented in the headline evaluation numbers below. In the team's demo cases, a deliberately low-confidence ticket returns **context precision and recall of 0.0** rather than a fabricated match — the system correctly recognizes it has no real evidence and escalates instead of guessing. Feedback (a yes/no signal plus human notes) is captured on every case and stored as evaluation data; the first iteration intentionally does not feed it back into automatic retraining, so trust in the feedback signal can be established before it's allowed to change behavior. Recommendations surface to the engineer working the ticket — nothing in this workflow sends a resolution directly to a customer.

## Evaluation strategy

The team evaluated retrieval and generation separately:

- **Retrieval evaluation** used recall@k, MRR, and nDCG against duplicate-link pairs as ground truth, plus a candidate-depth audit that specifically tested whether retrieval failures were a coverage problem (right answer never in the pool) or a ranking problem (right answer present but not promoted).
- **End-to-end evaluation** used a 26-case suite (13 high-confidence/autonomous, 13 low-confidence/human-review), each run through the full live pipeline and scored on RAGAS-style faithfulness, answer relevance, context precision, and context recall, plus whether routing matched the expected path.

Full methodology and caveats are in [evaluation-and-guardrails.md](evaluation-and-guardrails.md).

## Prototype results that can be verified

These are prototype evaluation numbers against the public Eclipse dataset — treat them as indicators of pipeline behavior, not production outcomes. Each links to the exact source file at the commit it was verified against (`520c38a`):

- **Routing accuracy:** 26/26 cases matched their expected route (`verdict_ok: true` for all rows) in the team's consistency regression run — [`reports/workflow/optimization/consistency_results.json`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/workflow/optimization/consistency_results.json).
- **Faithfulness:** 0.977 average on the 13 autonomous/high-confidence cases, 0.952 on the 13 human-review/low-confidence cases — averaged from per-case scores in [`reports/evaluation/canonical_ragas/queries_26_ragas.json`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/evaluation/canonical_ragas/queries_26_ragas.json).
- **Context precision/recall:** ~0.99 on the autonomous bucket (same file); deliberately near-zero on the low-confidence demo case, confirming the system doesn't fabricate evidence when there isn't any.
- **Candidate-depth audit:** on the technical-independent query set, Recall@200 (0.701) was close to the oracle Recall@10 achievable by reranking within that same 200-candidate pool (0.693) — meaning most retrieval misses were a *promotion/ranking* problem, not a *coverage* problem, which shaped where the team focused optimization effort. Source: [`reports/evaluation/candidate_depth_audit/technical_qwen_issue/summary.json`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/evaluation/candidate_depth_audit/technical_qwen_issue/summary.json).
- **Latency (reported, not independently re-derived):** the team's evaluation write-up reports a quality-gated optimization pass cutting end-to-end synthesis latency roughly 41% (46.7s → 27.4s on the autonomous path; 50.3s → 32.7s on the human-review path) without moving faithfulness or routing accuracy — [`reports/evaluation/canonical_ragas/canonical_ragas_report.tex`](https://github.com/venkat1596/resolution-intelligence-capstone/blob/520c38a2702db7b20598e49d8ddfc1f65d47c6c8/reports/evaluation/canonical_ragas/canonical_ragas_report.tex). This figure lives in the narrative report only — no separate generated benchmark file reproduces the exact numbers at this commit, so treat it as the team's reported result rather than an independently verified one.

## Product and technical tradeoffs

- **Hybrid retrieval over a single method** — dense embeddings alone miss exact identifiers; FTS5 alone misses semantic paraphrase. Running both adds pipeline complexity but each covers the other's blind spot.
- **Retrieval-then-generate over generate-then-cite** — deterministic evidence-pack assembly before reasoning adds latency and stages, but it's what makes citation validation possible at all.
- **SQLite over a managed vector/search service** — kept the prototype self-contained and reproducible for a course project; it isn't built for concurrent multi-user production load.
- **Gated external fallback over a fully open web search** — StackOverflow content only enters the evidence base when internal evidence is thin, and is marked lower-trust rather than blended in as equivalent to verified ticket history.
- **Feedback capture without auto-retraining** — slower to improve the model automatically, but avoids letting noisy early feedback silently change system behavior.

## Limitations

- Evaluated against a single public dataset (Eclipse issue tracker), not enterprise support data — results are prototype indicators, not a production benchmark.
- Runs locally via Docker Compose; there's no auth, multi-tenancy, or load testing behind these numbers.
- The team identified time-aware evaluation (making sure retrieval only uses information that would have existed before a ticket's real resolution date) as a risk worth controlling for, but a strict point-in-time evaluation methodology isn't yet implemented — current evaluation doesn't fully rule out resolution leakage.
- The original repository has no open-source license; it's linked here for demonstration and transparency, not reuse.
- This is a capstone prototype, not a shipped or commercially deployed product.

## Team attribution and my contribution

Resolution Intelligence was a **team capstone**, not a solo project. My contribution focused on product problem framing, target-user and workflow definition, iteration strategy, human-in-the-loop and feedback-loop design, guardrail and evaluation framing, architecture communication, and the product narrative in this case study. I collaborated with the team on the broader technical design and implementation, but I did not personally build the Qwen retrieval index, the Langflow implementation, the evaluation harness, the database, the Docker deployment, or the UI — those are team-built and documented in the [original repository](https://github.com/venkat1596/resolution-intelligence-capstone).

Where this write-up says **"we,"** it refers to team implementation. Where it says **"I,"** it refers to a specific, individually-owned contribution.

## Links to the original implementation

- Repository: [github.com/venkat1596/resolution-intelligence-capstone](https://github.com/venkat1596/resolution-intelligence-capstone)
- Architecture docs: `docs/architecture/`
- Evaluation docs: `docs/evaluation/`
- Setup guide: `TEAM_BUNDLE_README.md`

## How to explore or run the original project

The original repository includes a full setup guide (`README.md` and `TEAM_BUNDLE_README.md`) covering Python environment setup with `uv`, building the Eclipse SQLite database from the public Zenodo dataset, running the CLI workflow, wiring up the Langflow visual flow, and starting the local React + FastAPI UI with `docker compose up`. It requires an OpenAI API key for the aggregation and reasoning stages; retrieval and evidence-pack assembly work without one. Start with the repository's own README for current, authoritative setup steps.

## Future product roadmap

- A leakage-safe, point-in-time evaluation methodology, so retrieval and synthesis are tested strictly on information available before a ticket's real resolution.
- A path from the public Eclipse dataset toward a design compatible with real enterprise ticketing systems (Jira/ServiceNow-style data).
- Production-readiness work: authentication, multi-tenant isolation, and retrieval infrastructure tested at real support-desk scale.
- A structured path from captured feedback to model or ranking improvement, once enough human-reviewed data has accumulated to trust the signal.
- Deeper use of typed-graph evidence (dependency and blocking chains) inside synthesis, not just retrieval.
