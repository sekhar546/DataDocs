# DataPlumbr — Working Notes

> **Internal only. Not for external distribution.**
> Tracks open validation questions, risks, actions, and key decisions.
> The main documentation set (VISION, PRODUCT, ARCHITECTURE, etc.) is the outward-facing record.
> This file is the inward-facing one — what is unresolved, what is being tested, what has been decided and why.

Last updated: 2026-05

---

## 1. Validation Agenda

Three existence questions. Each must be answered against reality — not resolved by adding document sections. Status stays **Open** until a real-world finding closes it.

| # | Question | How to Answer | Deadline | Status | Finding |
|---|---|---|---|---|---|
| **V1** | Does the annotation moat survive frontier model improvement? | Run the frontier-model experiment: Facets schema + representative sample data → frontier model → score what fraction of Layer 2 annotations it recovers independently | Day 7 | Open | — |
| **V2** | Which segment has both unsolved pain and budget simultaneously? | 10 customer discovery conversations: 5 with Series B+ payer AI companies, 5 with seed/Series A. Specific question: does budget correlate with the pain being already solved? | Day 30 | Open | — |
| **V3** | What is the data bootstrap path? | Decide between: (a) synthetic Facets data from schema knowledge, (b) informal former-colleague validation, (c) structural-only validation first. Must be resolved before KB build begins. | Day 7 | Open | — |

### How to Close a Validation Item

A validation item is closed when you have a real-world finding — a number, a pattern across conversations, a decision with named rationale. Do not close with "seems likely" or a document argument. Write the actual finding in the Finding column.

**V1 scoring guide:**

| Model recovery rate | Interpretation | Implication |
|---|---|---|
| < 30% | Moat is large and durable | Proceed as planned — infrastructure company |
| 30–60% | Partial moat | Rebuild pitch around durable subset; reprice accordingly |
| > 60% | Moat is time-limited (~24–36 months) | Run as time-boxed arbitrage; maximize revenue now; plan for acquisition |

---

## 2. Risk Register

### Critical Risks — Existence-Level

| # | Risk | Severity | Current Status | Owner | Notes |
|---|---|---|---|---|---|
| **R1** | Frontier model improvement erodes annotation moat | Existential | **Unaddressed** | Raja | Entire business thesis rests on a capability gap that frontier labs are actively closing. Addressed only by V1 experiment. |
| **R2** | Data access chicken-and-egg | Blocking | **Unaddressed** | Raja | KB needs validation data → data needs client → client needs product. No named bootstrap path in current documents. Addressed by V3. |
| **R3** | Buyer contradiction | Strategic | **Unaddressed** | Raja | Series B+ buyers have budget because they already solved the domain grounding problem internally. Earlier-stage buyers have the pain but not the budget. Addressed by V2. |
| **R4** | Cross-client moat sequencing trap | High | Acknowledged / Deferred | Raja | Moat requires 20+ deployments to form. Vulnerable window (Phase 1–2) is exactly when a funded competitor could outbuild a solo founder. No survival strategy documented for pre-moat window. |
| **R5** | Capacity oversubscription | High | Acknowledged / Deferred | Raja | Runtime build + KB curation + sales + service delivery = 3–4 full-time jobs. "40% services cap" does not make the remaining 60% sufficient. No explicit sequencing of which jobs run when. |

### Smaller Risks — Non-Existential

| # | Risk | Severity | Status | Notes |
|---|---|---|---|---|
| **R6** | Pricing anchor is misleading | Low | Noted | "$40–80k vs $150–250k consultant" — consultant does the work; DataPlinth only improves client's AI. Real comparison includes client's own AI build cost. |
| **R7** | Kùzu is a young project | Low | Noted | "Migrate to Neo4j" fallback is non-trivial — graph query dialects differ. Migration would land at the worst time (Phase 3 scale pressure). |
| **R8** | WisdomAI competitive risk underweighted | Medium | Monitoring | $50M Series A (Kleiner Perkins + NVIDIA). Currently enterprise BI, not knowledge infrastructure — but that capital allows a fast pivot. Monitor for product direction signals. |

### Risk Status Definitions

| Status | Meaning |
|---|---|
| **Unaddressed** | Not confronted in documentation or planning. Needs active resolution. |
| **Acknowledged / Deferred** | Named and understood. Intentionally deferred — conditions for resolution not yet met. |
| **In Progress** | Active work underway to resolve. |
| **Closed** | Resolved with a specific finding or decision. Write the closing note. |
| **Monitoring** | External risk. No direct action; watching for triggers. |

---

## 3. Action Queue

Ordered by priority. An action is complete only when the expected output exists — not when effort has been expended.

### Immediate — Days 1–7

| # | Action | Priority | Deadline | Status | Expected Output |
|---|---|---|---|---|---|
| **A1** | Run frontier model experiment | 1 | Day 7 | Not started | Scored percentage of Layer 2 annotations derivable by frontier model from schema + data alone. Closes V1. |
| **A2** | Decide and document the data bootstrap path | 2 | Day 7 | Not started | Named path (synthetic / colleague validation / structural-first). Written rationale. Closes V3. |

### Near-Term — Days 8–30

| # | Action | Priority | Deadline | Status | Expected Output |
|---|---|---|---|---|---|
| **A3** | Customer discovery — 10 conversations | 1 | Day 30 | Not started | Pattern across conversations: which segment has both pain and budget. Closes V2. |
| **A4** | Set up competitive monitoring | 3 | Day 14 | Not started | Google Alerts for "domain knowledge infrastructure", "LLM grounding", "AI knowledge base", "WisdomAI". |

### Month 1–3

| # | Action | Priority | Target | Status | Expected Output |
|---|---|---|---|---|---|
| **A5** | Engage lawyer — license + no-distillation clause | 2 | Month 3 | Not started | Drafted license agreement with field-of-use and no-distillation clauses. |
| **A6** | Release Facets Layer 1 schema graph on GitHub | 2 | Month 3 | Not started | Public GitHub repo with structural-only schema graph (no annotations). Inbound discovery channel. |
| **A7** | Validate demand — 2–3 pilot conversations | 1 | Day 30 | Not started | 3 unprompted confirmations from technical leaders that they have experienced the domain grounding problem. |

### Deferred — Pending Validation Agenda Completion

| # | Action | Blocked By | Notes |
|---|---|---|---|
| **A8** | Begin Facets KB build (Layer 1 + Layer 2) | V1, V3 | Do not start until V1 tells you the moat is durable enough and V3 gives you a data path. |
| **A9** | Finalize pricing model | V2 | Buyer segment finding may require price repositioning. |
| **A10** | Resolve capacity / sequencing plan | V1, V2, V3 | Can only sequence properly once existence questions are answered. |
| **A11** | Domain expert recruiting (healthcare EHR) | Phase 2a milestone | Premature until Phase 1a generates revenue and validates model. |

---

## 4. Decision Log

Key decisions made during the founding phase. Captures what was decided, why, and what was considered and rejected. Useful when explaining choices to a future co-founder or investor.

| # | Date | Decision | Rationale | Alternatives Considered | Status |
|---|---|---|---|---|---|
| **D1** | 2026-05 | Organization name: **DataPlumbr** | Plumbers work with pipes and infrastructure; makes complex systems flow correctly. Distinctive. Domain available. | Multiple alternatives searched and eliminated | Final |
| **D2** | 2026-05 | Product name: **DataPlinth** | "Data" + "Plinth" (architectural foundation). Clean compound, no conflicts found in AI/data space. Coherent family with DataPlumbr. | Stratum (taken), Substrate (taken), Plinth (plinth.co healthcare conflict), Stratos (stratos.ai conflict), Sopharch, Mulagyan, Fondsage, Fortinth, Vajrinth | Final — pending formal trademark search |
| **D3** | 2026-05 | Starting scope: **Facets payer data for government-sponsored plans** | Highest founder depth (7/10), high willingness to pay, claims errors have direct revenue consequence, medium-high KB reusability across plans | FHIR R4 (low WTP), Claims 837/835 (less founder depth), HL7 v2.x (low WTP) | Final |
| **D4** | 2026-05 | Phase 1 delivery: **in-environment only, no hosted API** | HIPAA compliance requires it for target buyers. Avoids data residency complexity. Deferred KB-as-a-Service to Phase 2 evaluation. | Dual-mode (in-environment + hosted API) — rejected as Phase 1 scope overload | Final for Phase 1 |
| **D5** | 2026-05 | Service track: **funded R&D, not a business line** | Service engagements exist to generate product signal, not revenue. 40% time cap enforced. 40% of profit ring-fenced for product development. | Service-first (consulting) model — rejected; doesn't scale, margin is linear | Final |
| **D6** | 2026-05 | KB delivery: **two-component model** (runtime container + KB artifact) | Runtime and KB evolve at different cadences. Coupling them forces runtime updates for every KB change. Independent versioning enables safe KB skipping. | Single container baking — rejected; creates update coupling | Final |
| **D7** | 2026-05 | Retrieval: **GraphRAG hybrid** (Kùzu graph + local vector index) | Structural queries (join paths, cardinality) require deterministic graph traversal; vector similarity fails on multi-hop relational queries. Semantic queries (annotations, compliance) suited for vector search. | Pure RAG — rejected; fails on structural queries | Final |
| **D8** | 2026-05 | Embeddings: **annotation corpus ships as text, not pre-computed vectors** | Pre-computed vectors couple KB to a specific embedding model. Client's model may differ, causing silent retrieval degradation. Local compilation at startup eliminates model coupling. | Ship pre-computed — rejected for Phase 1 (with exception: one pre-computed index ships for text-embedding-3-small as startup-latency mitigation) | Final |
| **D9** | 2026-05 | Phase 3 expansion gate: **$3M ARR + domain expert hire required** | Founder has 1 year of banking experience — insufficient for Layer 2 annotations in financial services. Expansion without domain depth replicates the exact problem DataPlinth sells against. | Founder-led expansion — rejected; experience gap would compromise KB quality | Final |
| **D10** | 2026-05 | No freemium tier in Phase 1 | Free tier creates support burden that consumes the 40% service track time cap without product signal or revenue. Open-source Layer 1 schema graph achieves inbound discovery without the burden. | Freemium — deferred to Phase 2 when team capacity exists | Deferred |
| **D11** | 2026-05 | **Not** a model, SaaS API, consultancy, or data governance tool | Each alternative fails for a specific reason: model requires redistribution rights and goes stale; SaaS API fails regulated data residency; consultancy doesn't scale; governance catalogs client data rather than encoding domain knowledge for AI. | All four alternatives — explicitly rejected in VISION.md | Final |
| **D12** | 2026-05 | Founder scope claim: **Facets payer data (2011–2017), not "12 years healthcare DE"** | Honest representation of actual operational depth. Epic Clarity, Cerner, HL7 v2.x, FHIR require domain expert hires — founder cannot personally produce those annotations. Overstating depth would undermine trust in design partner conversations and investor due diligence. | Broader healthcare DE framing — rejected as inaccurate | Final |

---

## 5. Document Set Status

Reference for current state of the main documentation repo.

| Document | Version | Last Significant Change | Audience |
|---|---|---|---|
| README.md | 1.0 | Added Quick Start reading paths | All |
| VISION.md | 1.0 | Added Founding Asset + vintage knowledge rationale; universal failure examples | Investors, co-founders |
| PRODUCT.md | 1.0 | Added Validation Framework, Knowledge Currency, Client Configuration Variation, structural/semantic query table | Investors, co-founders, clients |
| ARCHITECTURE.md | 1.0 | Added Client Integration section, Schema Drift mechanism, Performance Considerations, Minimum System Requirements | Technical co-founders, clients |
| MARKET.md | 1.0 | Added Market Timing / 2026 demand validation framework; full sector analysis | Investors, co-founders |
| GTM.md | 1.0 | Added License Integrity, Integration Strategy, KB-specific pilot success metrics | Investors, co-founders |
| ROADMAP.md | 1.0 | Phase gates, moat evolution, decision gates | Investors, co-founders |
| TEAM.md | 1.0 | Honest founder profile, bus factor statement, co-founder/investor criteria | Investors, co-founders |

**External reviews processed:** 4 rounds. All valid tactical points actioned. Critical business-premise risks (V1–V3) identified in final review — tracked here, not addressed in documents.

---

*This file lives in a separate private repo. The main documentation set is in the public-facing DataPlumbr repo.*
