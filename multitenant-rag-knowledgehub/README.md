# Multi-Tenant RAG Knowledge Hub

**Role:** Product Manager · **Domain:** Insurance · Enterprise AI · **Stage:** 0→1 Platform Build

---

## TL;DR

Insurance organisations run on knowledge — but that knowledge is locked in silos. Underwriters, claims adjusters, sales teams, and customer support all operate on different document sets, with different access rights, and different interpretations of the same policies. I led the design and delivery of a **multi-tenant, RAG-based knowledge platform** that gave each team a secure, domain-specific AI assistant — without exposing cross-functional data or building five separate systems.

The result: knowledge retrieval time dropped by an estimated **~60%**, response consistency improved measurably across teams, and the platform onboarded 4 tenant groups within 3 months of launch with zero cross-tenant data incidents.

---

## Context & Background

A mid-to-large general insurer had a sprawling knowledge problem. Across five core functions — Underwriting, Claims, Sales & Distribution, Broker Operations, and Customer Support — teams were working from:

- Shared network drives with inconsistent folder structures
- Static SharePoint wikis that were months out of date
- Manual PDF search via email or internal portals
- Verbal SME validation for anything non-standard

The organisation had already piloted a **single shared chatbot** using a general-purpose LLM. It failed for two reasons:

1. **Data isolation**: Underwriting risk documents could not be exposed to sales agents or brokers. The shared-context model had no enforcement mechanism.
2. **Relevance failure**: A single knowledge base produced responses that were generically correct but domain-irrelevant. A claims adjudicator asking about coverage triggers would receive policy marketing copy. Trust in the tool collapsed within weeks.

The brief was clear: design a system that feels like one product but behaves like five — with strict tenant isolation, domain-relevant retrieval, and a consistent conversational interface.

---

## My Role

I was the product lead on this initiative, working with a team of 2 ML/data engineers, 1 backend engineer, 1 solutions architect, and business stakeholders from each of the five tenant functions. My responsibilities:

- Discovery, user research, and problem framing across all five persona groups
- Architecture decision input (isolation strategy, retrieval design, access control model)
- PRD, user stories, phased roadmap, and acceptance criteria
- Prioritisation of MVP scope under a fixed 12-week build timeline
- Stakeholder alignment across Compliance, IT Security, and Operations leadership
- Go-to-market, onboarding playbook, and adoption strategy per tenant

---

## Discovery & Problem Framing

### Research Approach

I ran **18 structured interviews** across the five tenant groups (3–4 per function), supplemented by workflow observation sessions and a document audit of each team's knowledge sources.

Key findings:

| Function | Avg. queries/day | Avg. time to resolve (manual) | Top friction |
|---|---|---|---|
| Claims Adjudicators | 30–45 | 14–18 min | Locating specific policy clauses for edge cases |
| Underwriters | 15–25 | 20–25 min | Cross-referencing risk rules with regulatory guidance |
| Customer Support | 50–70 | 8–12 min | Context-switching across 4+ systems per query |
| Sales / Brokers | 20–35 | 10–15 min | Finding product-specific exclusions and benefit tables |

Two findings shaped the entire product direction:

**Finding 1 — The problem was retrieval, not knowledge.** Every team had access to the right documents. The failure was in retrieving the right clause, from the right document, in under a minute. SMEs weren't bottlenecks because they knew more — they were bottlenecks because they knew *where* things were.

**Finding 2 — Data sensitivity was non-negotiable.** During discovery, a Claims SME explicitly flagged that underwriting loss ratios and case reserve data were commercially sensitive and could not be accessible to brokers or sales. Compliance confirmed: any system without tenant-level isolation would not be approved for rollout.

### Problem Statement

> *Insurance operations teams cannot quickly retrieve domain-specific, accurate information from their knowledge corpus — and a single shared AI assistant cannot satisfy both the relevance and data isolation requirements across functions.*

---

## Users & Personas

### Persona 1 — Claims Adjudicator (High Priority)

| Attribute | Detail |
|---|---|
| **Query type** | Coverage triggers, adjudication rules, historical case precedents |
| **Accuracy requirement** | Very high — errors have financial and regulatory consequences |
| **Latency tolerance** | Medium — willing to wait 30–60 sec for a reliable answer |
| **Key need** | Source citations alongside every response; no hallucination tolerance |

### Persona 2 — Underwriter (High Priority)

| Attribute | Detail |
|---|---|
| **Query type** | Risk appetite guidelines, reinsurance terms, product-specific clauses |
| **Accuracy requirement** | Very high — decisions directly affect portfolio risk |
| **Latency tolerance** | Medium — prefers precision over speed |
| **Key need** | Ability to query across multiple guideline versions with date context |

### Persona 3 — Customer Support Representative (High Volume)

| Attribute | Detail |
|---|---|
| **Query type** | Policy servicing, coverage FAQs, claim status, renewal terms |
| **Accuracy requirement** | High — incorrect answers affect NPS and regulatory compliance |
| **Latency tolerance** | Low — handling live customer calls, needs sub-10 sec responses |
| **Key need** | Simple, jargon-free outputs they can relay directly to customers |

### Persona 4 — Sales Agent / Broker (Secondary)

| Attribute | Detail |
|---|---|
| **Query type** | Product benefits, exclusions, pricing tiers, competitor comparison |
| **Accuracy requirement** | Medium — sales context, not binding decisions |
| **Latency tolerance** | Low — in-conversation lookups during client calls |
| **Key need** | Quick, confident answers; no access to internal risk or claims data |

---

## Solution Design

### Architecture Approach: Logical Multi-Tenancy

After evaluating three isolation models, I recommended **logical isolation using per-tenant vector collections** with metadata-based filtering — rather than physical infrastructure separation or a single shared index.

| Approach | Isolation strength | Cost | Complexity | Our choice |
|---|---|---|---|---|
| Separate vector DBs per tenant | Very high | Very high | High | ✗ Over-engineered for phase 1 |
| Single shared index, no isolation | None | Low | Low | ✗ Failed compliance gate |
| Per-tenant collections + metadata filtering | High | Medium | Medium | ✓ Selected |

The reasoning: separate infrastructure would have tripled cost and delayed launch by 6+ weeks. A shared index failed Compliance review immediately. Logical isolation — separate ChromaDB collections per tenant, with query-time metadata filters enforcing tenant context — gave us isolation guarantees without infrastructure bloat.

### How the System Works

```
1. User authenticates
       ↓ Role + tenant ID extracted from identity provider (SSO)

2. Query submitted via chat interface
       ↓ Query embedded into vector representation

3. Tenant-scoped retrieval
       ↓ Vector search restricted to tenant-specific collection
       ↓ Metadata filters applied (document type, effective date, access tier)

4. Top-K relevant chunks retrieved with relevance scores

5. Context injected into LLM prompt
       ↓ Prompt includes: tenant context, retrieved chunks, query

6. LLM generates structured response:
       • Direct answer
       • Reasoning (where applicable)
       • Source citations (document name, section, page)

7. Guardrails applied
       ↓ Output validated against compliance rules
       ↓ Cross-tenant data references blocked

8. Response returned to user
       ↓ Query + metadata logged for analytics and retrieval retraining
```

### Key Architectural Decisions

**Decision 1 — Shared LLM, isolated context**

Instead of separate fine-tuned models per tenant (expensive, slow to update), we used a single shared LLM (GPT-4) with tenant-specific context injection. The LLM itself holds no tenant data — all domain specificity comes from retrieval. This kept inference costs manageable and meant knowledge updates required only document re-ingestion, not model retraining.

**Decision 2 — Metadata tagging as a first-class engineering requirement**

Early ingestion tests showed that retrieval relevance within a tenant was poor when documents were ingested without structure. I pushed for a metadata schema upfront: `{tenant_id, function, document_type, effective_date, version, product_line}`. This added 2 weeks to the data engineering pipeline but reduced retrieval irrelevance by an estimated 40% in internal testing.

**Decision 3 — Citation-mandatory outputs**

Given the high-stakes nature of claims and underwriting decisions, I specified that every response must include a source citation. This was a product constraint, not a model capability — implemented via structured output prompting and a UI component that rendered the source alongside the answer. Adoption data later confirmed that citation visibility was the single biggest driver of agent trust.

**Decision 4 — MVP scope cut: no cross-tenant insights**

Business leadership initially wanted "cross-functional analytics" — the ability to see trends across all tenant queries. I deprioritised this to Phase 3 on two grounds: (1) Compliance had not yet defined the data governance framework for aggregated query logs, and (2) it would have introduced architectural complexity at the retrieval layer that risked delaying the MVP. The decision was documented and accepted by stakeholders.

---

## Phased Roadmap

### Phase 1 — MVP (Weeks 1–12)

- Tenant isolation architecture (4 tenants: Claims, Underwriting, Customer Support, Sales)
- Document ingestion pipeline (PDF, DOCX, policy wordings, SOPs)
- Metadata tagging schema and validation
- RAG retrieval with per-tenant vector collections
- Basic chat interface with source citations
- Role-based access control (SSO integration)
- Usage logging (query volume, latency, citation clicks)

### Phase 2 — Relevance & Trust (Months 4–6)

- Agent feedback loop (thumbs up/down per response — inline rating)
- Retrieval reranking based on feedback signals
- Improved chunking strategy (clause-level vs document-level)
- Analytics dashboard for tenant admins (query trends, unanswered queries)
- Expanded tenant onboarding: Broker Operations added

### Phase 3 — Scale & Intelligence (Months 7–12)

- Cross-tenant aggregated insights (under Compliance-approved governance framework)
- Personalisation layer (user query history, role-specific reranking)
- Automated document freshness detection (flag outdated chunks)
- API access for embedding into existing CRM and claims portals

---

## Metrics & Outcomes

| Metric | Baseline | Post-Launch (90 days) | Change |
|---|---|---|---|
| Avg. knowledge retrieval time (manual) | 14–25 min | ~5–8 min | **↓ ~60%** |
| First-contact resolution (Customer Support) | 64% | 79% | **↑ 15 pp** |
| SME escalations for knowledge queries | ~30% of queries | ~14% | **↓ 53%** |
| Cross-tenant data incidents | — | 0 | **Zero breaches** |
| Tenant onboarding time (new team) | N/A (manual) | ~2 weeks | **New capability** |
| Agent-reported response trust (survey) | — | 77% rated responses "reliable" | — |
| Platform DAU at 90 days | — | 82% of eligible users | — |

**Measurement approach:**
- Retrieval time: CRM timestamps + agent self-report survey (pre/post cohort)
- First-contact resolution: call centre system logs
- SME escalation: ticket system (tagged "knowledge query" escalations)
- Data incidents: IT Security audit log review
- Trust: biweekly pulse survey, 5-point Likert scale

---

## Challenges & What I Learned

### Retrieval quality varied dramatically by document type

Policy wordings and SOPs chunked well at clause level. But underwriting guidelines — often structured as decision trees or tabular risk matrices — produced incoherent chunks when split by character count. We had to build document-type-specific chunking logic, which wasn't in the original MVP scope. I should have audited document structure diversity earlier in discovery.

**Learning:** Document structure is a product input, not just an engineering detail. PMs building RAG systems need to understand the corpus before signing off on a chunking strategy.

### Access control implementation was underestimated

The initial plan assumed SSO role mapping would be a 3-day integration. It took 3 weeks due to inconsistent role naming conventions across the insurer's identity provider and the need to map hierarchical roles (e.g. "Senior Underwriter" inheriting "Underwriter" permissions). I should have flagged this as a risk dependency earlier.

**Learning:** In enterprise environments, identity and access management is almost always more complex than it looks. Scope it as a separate workstream with its own timeline.

### Agents needed to be taught to distrust the system (selectively)

Counterintuitively, the biggest adoption challenge was overconfidence. Some agents — particularly in Customer Support — were treating AI outputs as authoritative without checking citations. We introduced a "verify before acting" prompt in the UI and added training sessions framing the tool as a "research assistant, not a decision maker." Escalation rates for borderline queries actually increased post-training — which was the right behaviour.

**Learning:** In high-stakes domains, the risk of over-trust is as real as the risk of under-adoption. Build in mechanisms that preserve human accountability.

### Metadata tagging was a product problem, not just an engineering problem

Initial document ingestion had inconsistent tagging — some PDFs had no effective date metadata, policy versions were not consistently labelled, and product line tags were applied manually by the data team with no validation. I introduced a metadata validation checklist as a mandatory step in the ingestion pipeline, with rejected documents flagged for remediation before entering the index.

**Learning:** Data quality governance is product work. Without it, retrieval quality degrades invisibly — and agents blame the AI, not the data.

---

## What I'd Do Differently

- **Run a corpus audit before scoping the build.** I underestimated the document structure diversity across tenants. A 1-week audit upfront would have surfaced the chunking complexity earlier and prevented mid-sprint rework.
- **Define the metadata schema in week 1, collaboratively.** The schema was initially drafted by engineering alone. When business stakeholders reviewed it, two critical fields were missing (product line and regulatory jurisdiction). Earlier cross-functional input would have caught this.
- **Instrument retrieval quality from day one.** We didn't have Recall@K or MRR tracking until Phase 2. Running retrieval evals in parallel with the build would have given us earlier signal on where the index was failing.
- **Build the feedback loop in MVP, not Phase 2.** The inline thumbs up/down was deprioritised to Phase 2 as a "nice to have." In hindsight, that feedback data would have been invaluable for understanding retrieval failures in the first 90 days.

---


## Skills Demonstrated

`Multi-Tenant Architecture` · `RAG System Design` · `User Research` · `Compliance & Data Governance` · `0→1 Product Build` · `Insurance Domain` · `Access Control Design` · `Metrics Instrumentation` · `Stakeholder Alignment` · `Phased Roadmapping`
