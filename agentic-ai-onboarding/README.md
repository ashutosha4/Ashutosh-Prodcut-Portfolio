
# Agentic AI Platform for Group Insurance Onboarding

**Role:** Product Manager · **Domain:** Group Insurance · Enterprise Operations · **Stage:** 0→1 Platform Build  
**Tech:** LangGraph · CrewAI · GPT-4 · FastAPI · Azure · Python · REST APIs

---

## TL;DR

Group insurance onboarding for large employer accounts is one of the most operationally expensive processes in the insurance back-office — involving data handoffs across HR systems, eligibility validation, underwriting rule checks, and policy record creation, all managed through a fragile mix of spreadsheets, emails, and manual ops effort.

I led the design and delivery of a **multi-agent agentic AI platform** that replaced this linear, human-coordinated workflow with an intelligent orchestration layer — five specialised AI agents, each owning a distinct step of the onboarding journey, collaborating dynamically to handle real-world data variability, exceptions, and edge cases without constant human intervention.

Outcome: end-to-end onboarding time reduced from **~12 days to ~2.5 days** on average for batches of 200–500 employees, manual ops effort cut by **~70%**, and exception handling accuracy improved to **94%** against a human-review baseline.

---

## Context & Background

### The Group Insurance Onboarding Problem

When a mid-to-large employer takes out a group insurance policy — life, health, or group personal accident — the insurer must onboard every enrolled employee onto that policy. For a company with 500 employees, this means processing 500 individual records, each requiring:

- **Data ingestion** from the employer's HR system (SAP HR, Workday, or flat-file exports)
- **Data validation** — checking completeness, format, and consistency of fields (date of birth, salary band, designation, dependent details)
- **Eligibility determination** — applying the employer's plan rules (who qualifies for which coverage tier, based on grade, tenure, employment type)
- **Exception handling** — resolving records that fail validation or sit in eligibility edge cases (contract employees, employees on probation, missing NRI declarations)
- **Policy record creation** — generating the member certificate and updating the policy administration system
- **Confirmation dispatch** — notifying HR and the employee

The insurer we worked with was managing **40–60 group onboarding batches per month**, ranging from 50 to 2,000 employee records per batch. The operations team of 12 handled this entirely manually, with underwriting providing rule guidance via email and HR teams submitting corrections through a shared inbox.

### Why Existing Automation Failed

The team had previously implemented an RPA-based workflow that handled simple, well-formatted batches. But it broke down consistently on:

- **Incomplete records** — employees with missing salary bands or ambiguous employment type (contract vs permanent)
- **Rule variability** — each employer group policy had custom eligibility rules; the RPA could not dynamically interpret rule documents
- **Exception feedback loops** — unresolved exceptions were bounced back to HR via email, creating 3–5 day delays per iteration
- **No reasoning capability** — the system could check a field against a lookup table but could not interpret ambiguous cases or suggest corrections

The RPA was processing only ~55% of records straight-through. The remaining 45% required manual intervention, consuming the majority of the ops team's capacity.

---

## My Role

I was the product lead on this initiative, embedded with a cross-functional team of:

- 2 ML/AI engineers (agent design, LangGraph orchestration)
- 1 backend engineer (FastAPI, system integrations)
- 1 data engineer (ETL pipeline, HR system connectors)
- 1 solutions architect (Azure infrastructure, security design)
- Domain SMEs from Group Ops and Underwriting

My responsibilities:

- End-to-end product ownership: discovery, PRD, roadmap, acceptance criteria, delivery oversight
- Agent capability scoping — defining what each agent could and could not do autonomously
- Human-in-the-loop design: specifying escalation thresholds, review UI, and audit trail requirements
- Stakeholder alignment with Group Ops, IT Security, Underwriting, and Compliance
- Go-to-market and change management for the ops team adoption
- Post-launch metric tracking and Phase 2 prioritisation

---

## Discovery & Problem Framing

### Research Approach

I ran **14 structured interviews** with ops agents, underwriters, HR contacts from 3 client accounts, and the RPA support team. I also shadowed the ops team for 3 live onboarding batches end to end — one simple batch (clean data, standard plan), one complex batch (500+ employees, multiple plan tiers, ~30% exception rate), and one failure case (a batch that took 19 days to complete due to repeated HR corrections).

### Key Findings

**Finding 1 — The bottleneck was exception handling, not data processing.**

Straight-through records (clean data, standard eligibility) were already handled in ~2 hours by the RPA. The 12-day average was almost entirely driven by exception queues: records flagged for issues sat in shared inboxes, were manually reviewed, returned to HR, corrected, re-submitted, and re-processed. A single batch with 80 exceptions could consume 3–4 days of an ops agent's week.

**Finding 2 — Exceptions were largely predictable and correctable.**

Analysing 3 months of exception logs (approx. 4,200 flagged records), I found that ~68% of exceptions fell into 5 recurring categories:

| Exception type | Frequency | Typical resolution |
|---|---|---|
| Missing salary band | 22% | Derivable from job grade mapping table |
| Ambiguous employment type | 18% | Determinable from contract start date + designation |
| Duplicate member record | 14% | Resolvable via PAN/Aadhaar match |
| Missing nominee details | 8% | Default to statutory nominee rules (flagged for HR confirmation) |
| Eligibility edge case (probation, part-time) | 6% | Requires underwriting rule lookup + plan document check |

This meant an intelligent agent with access to the right reference data and rule documents could auto-resolve the majority of exceptions — without human involvement.

**Finding 3 — Ops agents were doing reasoning work, not just processing.**

When I shadowed agents resolving exceptions, they weren't following a script. They were cross-referencing the plan document, checking a salary grade mapping spreadsheet, interpreting a clause, and making a judgment call. This was exactly the kind of contextual, multi-source reasoning that a well-designed AI agent could handle.

### Problem Statement

> *Group insurance onboarding is throttled not by data volume but by exception handling — a reasoning-intensive task currently performed manually, causing multi-day delays, high ops cost, and inconsistent outcomes across batches.*

---

## Users & Personas

### Persona 1 — Group Ops Analyst (Primary)

| Attribute | Detail |
|---|---|
| **Core task** | Process onboarding batches: validate, resolve exceptions, create policy records |
| **Volume** | 3–5 batches/week, 50–500 records each |
| **Current pain** | 60–70% of time spent on exception queues; repetitive, low-value work |
| **Success metric** | Straight-through processing rate; hours saved per batch |
| **Key need** | System that handles the obvious exceptions automatically and surfaces only the genuinely ambiguous ones |

### Persona 2 — HR / Benefits Administrator at Employer (External)

| Attribute | Detail |
|---|---|
| **Core task** | Submit employee data, respond to correction requests, confirm onboarding completion |
| **Current pain** | Multiple back-and-forth cycles; no visibility into where their batch stands |
| **Success metric** | Number of correction cycles; time from submission to confirmation |
| **Key need** | Single submission with intelligent auto-correction where possible; clear status visibility |

### Persona 3 — Underwriting / Rules Owner (Secondary)

| Attribute | Detail |
|---|---|
| **Core task** | Define and maintain eligibility rules per group policy |
| **Current pain** | Ad hoc email queries from ops for rule interpretation; rules encoded nowhere machine-readable |
| **Success metric** | Reduction in rule interpretation queries from ops |
| **Key need** | Rules encoded in a queryable format; agent applies rules consistently without manual mediation |

---

## Solution Design

### Why Agentic AI — Not RPA, Not a Single LLM

I evaluated three architectural options before committing to the multi-agent approach:

| Option | Capability | Failure mode | Decision |
|---|---|---|---|
| Enhanced RPA | Fast for structured, rule-consistent records | Cannot reason over ambiguous data or interpret rule documents | ✗ Already failed at 45% exception rate |
| Single LLM pipeline | Can reason, interpret, generate | No task specialisation; context window fills fast on large batches; hard to audit individual decisions | ✗ Rejected |
| Multi-agent orchestration | Specialised reasoning per task; each agent auditable; dynamic routing | More complex to build and test; orchestration failures need explicit handling | ✓ Selected |

The key insight: group onboarding is not one task — it is five distinct cognitive tasks (ingest, validate, determine eligibility, resolve exceptions, generate records) that each require different reference data, different reasoning strategies, and different escalation rules. Specialised agents, each owning one task, are more reliable, more auditable, and more maintainable than a single model trying to do everything.

---

### The Multi-Agent Architecture

The platform is built on **LangGraph** for agent orchestration and **CrewAI** for agent role definition, running on **Azure** with a **FastAPI** backend exposing integration endpoints.

#### Agent Roles & Responsibilities

**Agent 1 — Ingestion Agent**

*Responsibility:* Connects to HR system data sources (REST API for Workday/SAP HR, or secure SFTP for flat-file uploads), ingests the employee roster, normalises field formats (date formats, name casing, phone number structure), and stages records for processing.

*Tools available:* HR system API connectors, data normalisation rules, field mapping config per employer account.

*Output:* Normalised employee record batch in a standardised schema, with ingestion audit log (record count, timestamp, source system).

---

**Agent 2 — Validation Agent**

*Responsibility:* Runs field-level and record-level validation against the plan's data requirements. Checks for: mandatory field completeness, format compliance, logical consistency (e.g. date of birth vs age eligibility), and cross-field dependencies (e.g. nominee details required if sum assured > ₹50L).

*Tools available:* Validation rule engine (per plan configuration), reference data tables (pincode lookup, IFSC codes, designation taxonomy).

*Output:* Clean records forwarded to Eligibility Agent. Records failing validation tagged with specific failure reason codes and routed to Exception Agent. Validation report generated per batch.

*Key design decision:* Validation rules are plan-configurable, not hardcoded. When a new group policy is onboarded, an ops admin inputs the plan's data requirements into a configuration layer — the Validation Agent reads these dynamically. This was a deliberate choice to avoid a brittle rules engine that required engineering changes for every new client.

---

**Agent 3 — Eligibility Agent**

*Responsibility:* Determines coverage eligibility and tier assignment for each validated employee, by applying the group policy's eligibility rules. These rules define: who qualifies (permanent employees only vs all, minimum tenure, grade bands), what they qualify for (sum assured tier by salary band or designation), and any exclusions (pre-existing condition waiting periods, probation restrictions).

*Tools available:* Plan eligibility rule document (retrieved via RAG over the policy PDF), salary grade mapping tables, employment type classification logic, LLM reasoning for ambiguous rule interpretation.

*Key capability:* The Eligibility Agent uses **RAG over the policy document** to answer eligibility questions that cannot be resolved by structured rules alone. For example: *"Does a contract employee on a 24-month fixed-term engagement qualify as a permanent employee under Section 3.2 of the plan document?"* — this requires reading and interpreting natural language policy text, not just checking a table.

*Output:* Eligibility determination per record (eligible / ineligible / conditional), coverage tier assigned, reasoning logged. Ineligible or conditional records routed to Exception Agent.

---

**Agent 4 — Exception Handling Agent**

*Responsibility:* The most complex agent. Receives records flagged by Validation or Eligibility agents and attempts autonomous resolution using a tiered approach:

**Tier 1 — Auto-resolve:** Exception type is known, resolution is deterministic.
- Missing salary band → derive from job grade mapping table
- Duplicate record → match via PAN number, merge or flag as update
- Format error → auto-correct and log

**Tier 2 — Reasoned resolve:** Exception requires contextual reasoning but can be resolved with available reference data.
- Ambiguous employment type → cross-reference contract start date, designation, and tenure to classify
- Eligibility edge case → query policy document via RAG, apply reasoning, log confidence score

**Tier 3 — Human escalation:** Exception cannot be confidently resolved autonomously.
- Missing information not derivable from any available source
- Conflicting data across systems with no resolution heuristic
- Low confidence score (< 70%) on eligibility determination
- Any record involving regulatory status flags (NRI, minor nominees)

For Tier 3 escalations, the agent generates a structured exception card: the specific issue, what was attempted, what information is needed, and a suggested resolution for the ops agent to validate. This replaces the current process of an ops agent reading a raw error log and figuring out what went wrong themselves.

*Output:* Auto-resolved records passed to Policy Generation Agent. Tier 3 escalations surfaced in the ops review UI with structured context cards.

---

**Agent 5 — Policy Generation Agent**

*Responsibility:* For all resolved records (auto-resolved + human-confirmed), generates the policy artefacts: member certificate, policy schedule update, and system records in the policy administration platform (via REST API). Triggers the confirmation workflow: member welcome email with certificate, HR confirmation summary, and audit entry in the compliance log.

*Tools available:* Policy administration system API, document generation templates (per plan), email dispatch service, audit log writer.

*Output:* Member certificates generated, policy system updated, confirmation dispatched, end-to-end audit trail written.

---

**The Orchestrator — LangGraph State Machine**

The Orchestrator is the backbone of the platform. It is not itself an LLM — it is a **LangGraph state machine** that manages the directed graph of agent tasks, tracks the processing state of every record in a batch, and handles routing decisions.

Key orchestrator responsibilities:

- **Parallel processing:** Runs Validation and Eligibility checks concurrently where records are independent, reducing total processing time
- **Dynamic routing:** Routes each record to the correct next agent based on its current state (validated → eligibility, flagged → exception, resolved → policy generation)
- **Failure handling:** If an agent returns an error or timeout, the orchestrator retries (up to 3 attempts), then escalates to human review with full context
- **Batch state tracking:** Maintains a real-time state for the entire batch — how many records are clean, in exception, escalated, resolved, completed — surfaced in the ops dashboard
- **Audit log integrity:** Every state transition is logged with timestamp, agent ID, action taken, and confidence score where applicable

---

### End-to-End Processing Flow

```
HR System / File Upload
        │
        ▼
┌───────────────────┐
│  Ingestion Agent  │  ← Connects to Workday / SAP HR / SFTP
│                   │    Normalises & stages records
└────────┬──────────┘
         │  Normalised batch
         ▼
┌───────────────────────────────────────────┐
│            LangGraph Orchestrator         │
│   (State machine — tracks every record)   │
└───┬───────────────────────────────────┬───┘
    │                                   │
    ▼ (parallel)                        ▼ (parallel)
┌──────────────────┐        ┌──────────────────────┐
│ Validation Agent │        │  Eligibility Agent   │
│                  │        │  (RAG over policy    │
│ Field checks     │        │   doc + rule tables) │
│ Format checks    │        │  Tier assignment     │
│ Logic checks     │        │                      │
└────────┬─────────┘        └──────────┬───────────┘
         │ Flagged records             │ Edge cases
         └──────────┬──────────────────┘
                    ▼
        ┌──────────────────────────┐
        │  Exception Handling      │
        │  Agent                   │
        │                          │
        │  Tier 1: Auto-resolve    │
        │  Tier 2: Reasoned resolve│
        │  Tier 3: Escalate ──────►│ Ops Review UI
        └──────────┬───────────────┘  (structured
                   │ Resolved records  exception cards)
                   │ + Human-confirmed
                   ▼
        ┌──────────────────────────┐
        │  Policy Generation       │
        │  Agent                   │
        │                          │
        │  Member certificate      │
        │  Policy system update    │
        │  Confirmation dispatch   │
        │  Audit log entry         │
        └──────────────────────────┘
```

---

### Human-in-the-Loop Design

Full automation was explicitly ruled out for two reasons: (1) group insurance policy records are legally binding documents — errors have regulatory consequences, and (2) early system confidence on edge cases was insufficient to warrant removing human oversight entirely.

The human-in-the-loop layer was designed carefully to avoid recreating the old bottleneck:

- **Ops agents only see Tier 3 escalations** — not the full exception queue. The agent has already resolved Tiers 1 and 2 autonomously.
- **Each escalation card pre-fills the resolution** — the agent proposed what it would do if it had more confidence. The ops agent either approves (one click) or overrides with a correction.
- **Approvals feed the learning loop** — every human-confirmed decision is logged and used to improve agent confidence calibration over time.
- **SLA enforcement** — escalated records older than 4 hours trigger an automatic Slack notification to the ops team lead. Nothing sits in a queue invisibly.

---

## Key Product Decisions & Trade-offs

### 1. LangGraph vs CrewAI for Orchestration

Both were evaluated. **CrewAI** was used for agent role definition and tool assignment (clean abstraction for specifying what each agent can do and what tools it has access to). **LangGraph** was used for the orchestration state machine — its directed graph model maps directly to the branching, conditional routing logic of the onboarding workflow. Using both in combination gave us role clarity from CrewAI and flow control rigour from LangGraph.

### 2. RAG for Rule Interpretation

A key design decision was giving the Eligibility Agent the ability to query the plan document directly via RAG, rather than requiring underwriting to pre-encode every rule into a structured table. The trade-off: RAG retrieval introduces latency (~2–4 sec per complex eligibility query) and requires high-quality document chunking. The benefit: the system can handle rule nuance and new plan configurations without engineering changes.

### 3. Confidence Thresholds

Every agent decision that involves reasoning (rather than a deterministic rule check) outputs a confidence score (0–100). The Exception Handling Agent escalates to human review below 70%. This threshold was calibrated over the first 4 weeks post-launch using human reviewer feedback — early thresholds were too conservative (escalating ~35% of records), final calibration settled at ~8% escalation rate.

### 4. Audit Trail as a First-Class Feature

Given the regulatory context (IRDAI compliance for group policy records), every agent action — including intermediate reasoning steps — is written to an immutable audit log. This was a non-negotiable requirement from Compliance, scoped into MVP. Each log entry records: timestamp, agent ID, input record hash, action taken, tools used, output, and confidence score. The audit log is queryable by ops leads and available for regulatory inspection.

---

## Phased Roadmap

### Phase 1 — MVP (Weeks 1–14)

- 5-agent orchestration pipeline (Ingestion, Validation, Eligibility, Exception, Policy Generation)
- LangGraph state machine with parallel processing and dynamic routing
- RAG integration over plan eligibility documents
- Ops review UI (Tier 3 escalation cards with one-click approval)
- Audit log (agent actions, confidence scores, state transitions)
- Integration: Workday API, policy administration system REST API, SFTP ingestion fallback
- Slack SLA alerting for aged escalations

### Phase 2 — Precision & Feedback (Months 4–6)

- Feedback loop: human approvals/overrides feed confidence recalibration
- Retrieval reranking for RAG (improving eligibility clause precision)
- Analytics dashboard (batch-level: STP rate, exception breakdown, processing time, agent accuracy)
- Self-service plan configuration UI for ops to onboard new group policies without engineering support
- Expanded HR system connectors (BambooHR, Oracle HCM)

### Phase 3 — Proactive Intelligence (Months 7–12)

- Predictive exception flagging: score incoming batches for exception likelihood before processing starts, based on historical patterns per employer account
- Proactive data quality nudges to HR portals: flag likely errors before submission
- Cross-batch analytics: exception trend reporting per employer, surfaced to account managers for client conversations
- API for HR platforms to trigger onboarding directly from Workday/SAP workflow events (event-driven architecture)

---

## Metrics & Outcomes

| Metric | Baseline | Post-Launch (90 days) | Change |
|---|---|---|---|
| End-to-end onboarding time (avg per batch) | ~12 days | ~2.5 days | **↓ 79%** |
| Straight-through processing rate | ~55% | ~88% | **↑ 33 pp** |
| Manual ops effort per batch (hours) | ~18 hrs | ~5.5 hrs | **↓ ~70%** |
| Exception auto-resolution rate | 0% (fully manual) | ~92% of Tier 1+2 exceptions | **New capability** |
| Escalation rate (Tier 3, to human) | ~45% of records | ~8% of records | **↓ 37 pp** |
| HR correction cycles per batch | 3–5 cycles avg | < 1 cycle avg | **↓ ~80%** |
| Policy record accuracy (vs audit) | 96.2% | 98.8% | **↑ 2.6 pp** |
| Onboarding batches processed/month | 40–60 | 80–100 (same team size) | **↑ ~67% throughput** |

**Measurement approach:**
- Onboarding time: policy administration system timestamps (batch open → all records confirmed)
- STP rate: orchestrator state logs (records completing all 5 agents without human touch)
- Ops effort: time-tracking data (ops team logged hours per batch pre and post)
- Exception auto-resolution: agent action logs (Tier 1+2 resolutions vs Tier 3 escalations)
- Policy accuracy: monthly audit sample (random 5% of records reviewed by compliance)

---

## Challenges & What I Learned

### Agent coordination failures were the hardest bugs to diagnose

When the system misbehaved, the failure was often not in a single agent but in the handoff between agents — a record that was marked "resolved" by the Exception Agent but carried a data field that the Policy Generation Agent couldn't process. We introduced a **schema contract** between agents: each agent's output must conform to a validated schema before the Orchestrator routes it forward. Schema validation failures are logged and surfaced as Orchestrator-level exceptions. This pattern — treating agent outputs as typed contracts — eliminated the majority of silent handoff failures.

**Learning:** In multi-agent systems, the interface between agents is as important as the agents themselves. Define output schemas early and enforce them at the orchestration layer.

### Confidence calibration took longer than expected

Our initial confidence thresholds were set based on engineering intuition. In the first 2 weeks of production, the Exception Agent was escalating ~35% of records to human review — much higher than the target 10%. I ran a calibration sprint: ops agents reviewed a sample of Tier 3 escalations and rated whether the agent's proposed resolution was actually correct. We found the agent was correct ~78% of the time at the threshold it was escalating at. We lowered the escalation threshold from 80% to 70% confidence, which dropped the escalation rate to ~8% without a material impact on accuracy.

**Learning:** Confidence thresholds are a product decision, not just an engineering parameter. They encode the acceptable trade-off between automation rate and accuracy risk — and they need to be tuned empirically against real data, not set theoretically.

### RPA decommissioning was a change management problem

The existing RPA had been in production for 3 years. The ops team was familiar with its failure modes and had workarounds. Switching to the new platform meant abandoning those workarounds and trusting a system they didn't understand. I ran a 4-week parallel-run period where both systems processed the same batches — ops agents could compare outputs. The agentic system outperformed on exception handling in every parallel run. By week 3, the team was voluntarily preferring the new system. By week 4, the RPA was decommissioned.

**Learning:** Never ask a team to trust a new system on faith. Parallel runs that generate direct comparisons are the most effective adoption mechanism when replacing a workflow people have lived with for years.

### Eligibility RAG required domain-specific chunking

Policy documents for group insurance are structured differently from standard prose: they contain tables, numbered clauses, cross-references ("subject to Section 4.2(b)"), and defined terms. Standard character-based chunking fragmented these documents in ways that destroyed meaning. We built a document parser that chunked at the clause boundary, preserved section numbering as metadata, and extracted defined terms into a separate lookup layer. Retrieval precision on eligibility queries improved from ~61% to ~89% after the parser was rebuilt.

**Learning:** Domain-specific document structure requires domain-specific ingestion logic. Never assume a general-purpose chunker will work on insurance, legal, or regulatory documents — audit retrieval quality on real queries before launch.

---

## What I'd Do Differently

- **Schema contracts between agents should be in scope 0, not discovered mid-build.** Define the input/output schema for every agent before a line of code is written. This would have eliminated 2 weeks of integration debugging.
- **Run retrieval quality evals before the eligibility agent is built.** We discovered the chunking problem late, after the agent was already written to depend on a retrieval layer that wasn't working well. A retrieval-first evaluation sprint (Recall@K, MRR, manual spot-checks on eligibility queries) would have surfaced this in week 2, not week 8.
- **Include ops agents in exception card UX design from the start.** The first version of the escalation card UI was designed by the engineering team. It was technically complete but ops agents found it confusing — the proposed resolution was buried below the error detail. A single co-design session with two ops agents would have caught this before launch.
- **Build the analytics dashboard earlier.** We had no batch-level visibility until Phase 2. During the first 90 days, I was manually querying the orchestrator logs to understand system performance. An ops dashboard should be MVP, not Phase 2.

---

## Architecture Summary

| Layer | Component | Technology |
|---|---|---|
| **Orchestration** | LangGraph state machine | LangGraph (Python) |
| **Agent framework** | Role and tool definition | CrewAI |
| **LLM** | Reasoning, eligibility interpretation, exception resolution | GPT-4 (Azure OpenAI) |
| **RAG** | Policy document retrieval for eligibility queries | ChromaDB + Azure Blob |
| **Backend** | API layer, integration endpoints | FastAPI (Python) |
| **Data pipeline** | HR data ingestion, ETL, normalisation | Python (Pandas), Azure Data Factory |
| **Integrations** | HR systems, policy admin platform | REST APIs, SFTP |
| **Cloud** | Hosting, compute, storage | Azure (AKS, Blob, OpenAI) |
| **Audit** | Immutable action log | Azure Cosmos DB |
| **Ops UI** | Escalation review, batch dashboard | React (internal tool) |

---

## Artefacts

- Product Requirements Document (PRD) — agentic onboarding platform
- Agent capability specifications (per-agent: role, tools, input schema, output schema, escalation logic)
- LangGraph orchestration design (state graph, routing conditions, failure handling)
- User research synthesis (14 interviews, 3 batch shadowing sessions)
- Exception taxonomy and auto-resolution playbook
- Confidence calibration log (threshold iterations and outcome data)
- Go-to-market plan + parallel-run protocol
- Compliance audit log specification

*Full artefacts available on request.*

---

## Skills Demonstrated

`Agentic AI System Design` · `Multi-Agent Orchestration` · `LangGraph` · `CrewAI` · `RAG Integration` · `Human-in-the-Loop Design` · `0→1 Product Build` · `Group Insurance Domain` · `Compliance & Auditability` · `User Research` · `Change Management` · `Metrics Instrumentation`
