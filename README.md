# AI-Native Practice Operations Platform

> **Architecture Case Study — Principal AI Architect | AI Platforms & Distributed Systems**

This repository presents an **anonymized enterprise architecture case study** for a multi-tenant, HIPAA-aligned practice operations platform. The platform sits beside the clinical record and runs capture, response, matching, intake, scheduling, referral management, capacity planning, and analytics for multi-clinician behavioural-health practices.

The material is intentionally architecture-first. It focuses on **why architectural decisions were made, what constraints shaped them, how the platform was decomposed, and how the design was validated against production requirements**.

The staff console in this repository is presented as **Northline**. That name is an anonymized demonstration label for the coordinator workspace. It is not a customer, employer, or practice name. The console and the narrated walkthrough show how the architecture is operated. They do not embed the production Temporal, model-serving, or data-plane runtimes.

**Walkthrough:** [`public/northline-walkthrough.mp4`](public/northline-walkthrough.mp4) — 5 minutes 35 seconds, voiceover synced to the clicks.

---

## 1. Executive Context

Multi-clinician behavioural-health practices run the clinical record in an EHR. Everything that happens before the first session, and most of what keeps a person engaged after it, ran on inboxes, spreadsheets, voicemail, and memory. Model use, if present at all, was a drafting aid inside one channel. It did not own safety, tenancy, matching outcomes, or cost.

The architectural objective was to establish a reusable **AI execution and control boundary** for the operational lifecycle around the session: a supervised agent fabric inside durable workflows, a feature platform with point-in-time correctness, a provider-independent model gateway, and governance that does not depend on the model behaving.

### Architecture focus

- Multi-tenant operational AI runtime beside the EHR
- Supervised multi-agent orchestration with typed handoffs
- Durable workflow execution for lifecycles measured in days
- Practice-voice generation with per-tenant adapters
- Safety sentinel as a hard barrier, not a prompt
- Graded-outcome matching and a feature platform
- Graph and vector retrieval with identifier rehydration
- Session, graph, and operational state kept apart
- Model and provider abstraction by capability class
- Autonomy as a policy with a live quality term
- Policy enforcement and tool authorisation outside the LLM
- AI guardrails and indirect prompt-injection defence
- Observability across agent steps, model calls, tools, and cost
- Reliability, failure containment, and graceful degradation
- AI FinOps under scale-based pricing
- Kubernetes deployment with a cross-cloud inference failover path
- Architecture governance and ADR-driven decision making

---

## 2. Business / Engineering Problem

The platform needed to support practice operations while addressing several architectural concerns simultaneously:

1. **Multi-tenancy** — isolate tenant identity, messages, features, caches, retrieval namespaces, adapters, and model artifacts. A practice ranges from five clinicians to a 25+ clinician, multi-location network. The architecture must serve both without a fork.
2. **Stateful execution** — an inquiry lives for days: first reply, follow-up cadence, intake chase, slot hold, referral loop, and a human who may take over in the middle.
3. **Provider abstraction** — business workflows must not bind to a model name. Frontier capability is bought. High-volume narrow models are self-hosted. Either side can fail.
4. **Retrieval quality and privacy** — practice knowledge, payer rules, and eligibility paths must be retrievable without placing raw identifiers in a prompt.
5. **Operational resilience** — contain failure across model providers, calendars, EHR connectors, telephony, and the primary region.
6. **Safety and governance** — a small fraction of messages contain crisis, harm, or abuse, often indirectly and in a second language. The platform is operational, not clinical. It must not diagnose, recommend treatment, interpret a screener, or counsel.
7. **Observability** — correlate an inquiry, a workflow run, an agent step, a model call, a tool call, a human decision, and a cost record.
8. **Cost control** — pricing follows practice scale, not token volume. The platform, not the tenant, carries inference-cost risk. Cost per inquiry lifecycle has to stay inside the plan.

---

## 3. Architectural Objectives

The target architecture was designed around the following objectives:

- Establish one operating layer for capture, response, match, intake, schedule, refer, plan, and measure, rather than a channel-specific assistant.
- Separate **control-plane and governance concerns** from **request-path execution**.
- Keep the EHR as the clinical system of record. Synchronise and reconcile. Do not overwrite.
- Put anything that must survive a deploy, a wait, or a compensation into a durable workflow. Put anything that reasons into a checkpointed graph.
- Make safety, consent, and authorisation explicit architectural boundaries. Do not leave them to model refusal.
- Treat retrieval, features, memory, model invocation, and tool execution as independently governed capabilities.
- Learn from outcomes that arrive days to months later, using the same feature definitions the online decision used.
- Build production observability, eval gates, and cost attribution into the runtime rather than after deployment.
- Hold measurable controls for first-reply latency, crisis recall, isolation, and unit cost.

### Measurable objectives

| Objective | Target |
|---|---|
| First on-brand reply, every day of the week | p50 ≤ 90s, p95 ≤ 150s; median first reply under two minutes |
| Voicemail transcribed, tagged, and routed | p95 ≤ 60s from the end of the recording |
| Language detection | < 1s, 30+ languages; the thread continues in the detected language |
| Crisis-signal recall | ≥ 0.99 per supported language on an adjudicated golden set |
| Match quality | Right-fit better than first-available, measured by attended and retained sessions, not by bookings alone |
| Isolation | Zero cross-tenant exposure across data, caches, retrieval, features, and model artifacts |
| Unit economics | Cost per inquiry lifecycle inside plan at fleet load, attributable per tenant |

---

## 4. Production Evidence

Figures below are aggregate outcomes across pilot practices, measured on the same event-sourced funnel the platform runs. Individual practices vary. They are not fleet-wide capacity claims.

| Metric | Before | After |
|---|---:|---:|
| Average response time to a new inquiry | 9h 12m | **2h 14m** (68% faster) |
| Median first reply | — | **1m 42s** |
| Same-day reply rate | baseline | **94%** |
| Inquiry-to-session conversion | baseline | **+38%** |
| Right-fit match versus first-available | 1.0× | **2.4×** |
| Time to first session | baseline | **6 days sooner** |
| In-network match rate | baseline | **+41%** |
| 90-day retention | baseline | **+27%** |
| Administrative time | baseline | **12 hours saved per clinician per week** |
| Booked sessions per month, after 12 months | baseline | **+162%** average lift |
| Inquiries lost in the inbox | untracked | **0** |
| Onboarding | — | **94%** onboarded in under 30 days |

Supporting production characteristics, not capacity targets:

| Characteristic | Evidence |
|---|---|
| Programme | Greenfield. 18 months to full capability. First production tenants at month 7. |
| Hot-path model mix | ~70% of invocations served by encoders, rankers, and tenant adapters rather than frontier models |
| Billable input tokens | 45–60% reduction from prefix caching and context discipline |
| Safety detection | Fine-tuned multilingual ensemble met the ≥ 0.99 recall floor on the adjudicated set. LLM-only moderation did not, on indirect Spanish-language ideation. |
| Auto-send compute budget | ~27s p95, leaving headroom for one full provider retry inside the reply objective |

> **Important:** The planning load model in section 15 is an architectural sizing assumption. It is not presented as achieved production scale. Pilot outcomes above are the validated evidence.

---

## 5. Architecture Principles

### 5.1 Platform over channel duplication
Capture, tagging, voice, matching, intake, and referral behaviour belong in one lifecycle, not in a template sequence per inbox.

### 5.2 Policy outside the model
The model may propose a draft, a rank, or a slot. Authorisation, autonomy, consent, and side effects are decided outside the model.

### 5.3 Safety is a barrier, not a filter
The sentinel runs on the critical path, fail closed. An indeterminate verdict is not a clear. No generative reply is produced on the crisis path.

### 5.4 Explicit execution boundaries
Channel ingest, the staff workspace, orchestration, intelligence, and data each have a tier boundary. The BFF contains no business logic.

### 5.5 Two runtimes, one rule
Anything that must survive a deploy, wait for a person, wait for time, or compensate is a workflow concern. Anything that reasons over context to produce a proposal is a graph concern.

### 5.6 Typed handoffs
A specialist receives a validated payload, not another agent's prose. Free-form inter-agent chat is rejected.

### 5.7 Human decisions are training rows
Approve, edit, reject, and override are events. They train autonomy, voice adapters, and the ranker. The review schema is part of the training pipeline.

### 5.8 Autonomy is earned
A rising rejection rate for a tenant and intent demotes that intent to approval until quality recovers. Autonomy is not a static setting.

### 5.9 Point-in-time features
Online decisions and offline training read the same definitions. A feature that did not exist at decision time cannot appear in the training row.

### 5.10 Minimum necessary context
Facts hold codes and span references. Identifiers are rehydrated after generation. They are not placed in the prompt.

### 5.11 Evidence-driven architecture
Decisions are connected to a quality-attribute scenario, a spike or a benchmark, a trade-off that was accepted, and a production or eval result.

---

## 6. Target Architecture

Five tiers own request flow. Two planes own control and governance.

```text
Channels (web, SMS, email, voice, voicemail, referral, directory)
                          |
              T1  Channel and edge
              consent, language ID, WAF
                          |
              T2  Experience and API
              staff workspace, GraphQL BFF
                          |
              T3  Orchestration
         Temporal lifecycle  +  LangGraph step
              /            |            \
        Safety        Specialists      Verifier
         barrier       (typed           (other
                        handoff)         model family)
                          |
              T4  Intelligence
         gateway, router, Triton, vLLM, retrieval, speech
                          |
              T5  Data and knowledge
     OLTP, event log, feature store, vector, lexical, graph
                          |
        Control plane (policy, autonomy, personas, kills)
        Governance plane (factory, eval, red team, audit, cost)
```

| Tier or plane | Owns | Non-negotiable property |
|---|---|---|
| T1 Channel and edge | Web, SMS, email, telephony, voicemail, directories, referrals. WAF, rate limits, consent, edge language identification. | Idempotent ingestion. Opt-out before any outbound byte. |
| T2 Experience and API | Staff workspace, clinician and leadership views, GraphQL BFF, REST and gRPC. | Tenant claim in every token. No business logic in the BFF. |
| T3 Orchestration | Durable lifecycles, supervisor and specialist agents, human interrupts. | Bounded autonomy. Every side effect is a typed, authorised, idempotent tool call. |
| T4 Intelligence | Model gateway and router, inference, retrieval, speech, feature serving. | Single egress for every model call. Policy evaluated per call. |
| T5 Data and knowledge | OLTP, event log, lakehouse, feature store, vector, lexical, and graph stores. | Isolation by construction. Point-in-time correctness for every training row. |
| Control plane | Policy engine, autonomy policy, persona profiles, kill switches, configuration. | Declarative, versioned, evaluated at runtime. |
| Governance plane | Model factory, evaluation, red teaming, AI inventory, audit ledger, cost ledger. | Nothing reaches production without lineage, an eval scorecard, and a rollback path. |

### Scope delivered

| Domain | Capability |
|---|---|
| Capture | Unified inbox across web, chat, SMS, email, calls, voicemail, referrals, and directory leads. Missed-call capture. Cross-channel entity resolution. |
| Understand | Insurance, urgency, concern, modality, geography, clinician traits. Call and voicemail transcription. |
| Respond | Practice-voice drafts and follow-up sequences. Human takeover pauses automation immediately. Full-thread localisation. |
| Match | Hard eligibility, then a learned rank on fit, availability, and outcomes. Overridable scores. Waitlist. |
| Intake | Forms, consent, screeners, pre-fill, reminders, reconciled EHR handoff. Screener scoring is deterministic. The model does not interpret it. |
| Schedule | Live availability, transactional holds, confirmations, reschedules. |
| Refer | Attribution to source. Closed-loop status to the referring provider. No clinical detail in that draft. |
| Plan | Caseload, utilisation, demand by specialty, waitlist, hiring signals, multi-location forecast. |
| Measure | Source-to-session funnel, keyword attribution, grounded weekly digest. |

---

## 7. Control Plane vs Data Plane

### Control plane

Responsible for platform-wide configuration and governance:

- Agent registry and versioned agent artifacts
- Tool manifests and capability-token policy
- Autonomy policy and persona profiles
- Model and provider routing by capability class
- Tenant configuration, including autonomy opt-ins and business hours
- Evaluation configuration and release gates
- Kill switches
- Operational metadata

### Data plane

Responsible for runtime execution:

- Channel ingest and the canonical inquiry envelope
- Workflow execution across the inquiry lifecycle
- Agent-graph steps inside a workflow activity
- Retrieval and feature reads
- Model invocation through the gateway
- Tool execution through the gateway
- Human interrupts and takeover
- Runtime telemetry and the cost ledger

### Governance plane

Responsible for what is allowed to ship and what must be provable later:

- Model factory and adapter lifecycle
- Node, trajectory, and outcome evaluation
- Trace replay of safety-relevant production traces
- Red teaming
- Audit ledger and decision records
- Per-tenant cost ledger

This separation lets governance change without coupling it to the first-reply path, and lets a kill switch or an autonomy demotion take effect without a model deploy.

---

## 8. Agent Runtime

The agent layer is a supervised hierarchy executing inside durable workflows. It is not an open group chat. Free-form multi-agent conversation was measured, on the same inputs, at 3.1× token spend and non-reproducible tool sequences.

| Runtime | Responsibility | Horizon | Technology |
|---|---|---|---|
| Durable workflow | Inquiry, follow-up cadences, intake, holds, referral loops, sagas, timers, human waits | Seconds to 90 days | Temporal |
| Agent graph | Supervisor routing, specialist execution, parallel fan-out, reflection, checkpointed state, interrupts | Milliseconds to minutes | LangGraph |

Graph invocations run as Temporal activities with heartbeats, deterministic retries, and idempotency keys. A timed-out step is retried or rerouted without a duplicate send or a duplicate hold.

Three structural patterns are used, and only where they belong:

1. **Parallel fan-out with a safety barrier.** On every inbound message the supervisor fans out to Safety, Language, and Understanding. Safety is a hard barrier and short-circuits the graph on a positive or indeterminate verdict.
2. **Typed handoff.** `handoff(target, reason, payload_schema)` is a state transition. The supervisor cannot pass arbitrary text.
3. **Generate, verify, repair.** Every outbound artifact passes a verifier from a different model family. At most two repairs. Defects are structured: `UNSUPPORTED_CLAIM`, `POLICY_CONFLICT`, `PHI_EGRESS`, `TONE_DRIFT`, `LANGUAGE_MISMATCH`. The third failure degrades to an approved template or a human queue.

### Graph state

Agents are stateless between invocations. State lives in a versioned contract (`AssistantState`) or in the operational store. Agents read and write only the fields their manifest declares. Parallel branches merge through reducers. Raw message text is not stored in facts. Facts hold normalised codes and span references into the encrypted message store. `human.takeover` is checked before every side-effecting node.

### Budgets

Enforced by the runtime, not by the prompt.

| Guard | Default |
|---|---|
| Max graph steps | 12, then the workflow routes to a human |
| Max repair iterations | 2 |
| Token ceiling | 24k input / 2k output. The gateway rejects the call. |
| Wall-clock deadline | Derived from the phase objective |
| Cost ceiling | Per tenant, per intent. Downgrade the tier before rejecting. |
| Cycle detection | The same node and state hash visited twice is a hard stop. |

### Agent inventory

Each agent is a versioned artifact: charter, output schema, tool allow-list, routing policy, eval suite, autonomy ceiling, data classes, and a degraded-mode behaviour.

| Agent | Charter | Ceiling |
|---|---|---|
| Assistant Supervisor | Classifies phase, plans the node sequence, routes, merges proposals. | A1. Cannot act. |
| Safety Sentinel | Crisis, self-harm, harm to others, abuse, medical emergency, minors at risk. | Fail closed. Overrides every other agent. |
| Language and Localisation | Language, script, terminology locks, back-translation. | A3 for templates. A1 for free text in low-resource languages. |
| Inquiry Understanding | Concern, payer, modality, availability, traits. Facts, not actions. | A0. |
| Communications Drafter | Practice-voice triage, follow-ups, reminders, scheduling messages. | A3 on approved intents. A1 otherwise. |
| Matching | Eligibility, rank, explanation, waitlist. | A2. A person or policy commits. |
| Scheduling | Slot proposals and short-lived holds. | A3 for holds. A2 to confirm a first session. |
| Benefits and Eligibility | Payer normalisation, panel check, eligibility preparation. | A2. |
| Intake Orchestrator | Packet selection, pre-fill, missing-item chase. | A3 for reminders. |
| Referral Loop | Referrer link and closed-loop status. Clinical detail excluded. | A1 toward providers. |
| Directory Content | Directory copy from an approved clinician profile. | A1. Clinician approves. |
| Capacity and Growth Planner | Demand, waitlist, and hiring signals from forecast features. | A1. |
| Insight Narrator | Weekly digest strictly from computed metrics. | A1. Numbers must come from a tool. |
| RCM and Credentialing Copilot | Payer correspondence and denial categorisation for staff. | A1. Never submits. |
| Verifier | Factual consistency, policy, PHI egress, tone, promises. | Veto only. |

### Evaluation of the fabric

| Level | What is measured | Method |
|---|---|---|
| Node | Extraction F1, safety recall and precision, translation adequacy, draft policy conformance | Golden sets per node and language |
| Trajectory | Routing, tool-call correctness, unnecessary steps, budget use, handoff validity | Replay of recorded traces through a candidate graph |
| Outcome | Time to first reply, edit distance, approval rate, booked-and-showed rate, complaint rate | Online, by tenant, intent, and language |

A graph change ships through trace replay. Divergence in the tool-call sequence on a safety-relevant trace blocks release.

---

## 9. Retrieval Architecture

Retrieval is hybrid and tenant-filtered. The knowledge graph is derived. It is never the system of record.

```text
Operational store
      |
 CDC (Debezium) + event log
      |
 Practice knowledge graph     Embeddings (version-stamped)
      |                              |
 Neo4j, path-shaped questions    pgvector (long tail)
                                 OpenSearch k-NN + BM25 (large tenants)
      \                              /
       +-------- hybrid fusion -----+
                     |
                Rerank / hop bound
                     |
           Context assembly
           (phi class on every ref)
                     |
              Agent / gateway
                     |
        Identifier rehydration after generation
```

GraphRAG is scoped to path-shaped questions: eligibility explanations, referral-network insight, and multi-hop operational analysis. Every hop is recorded in the context manifest. Semantic caching is disabled for any PHI-bearing prompt. A near-duplicate prompt across tenants produced cross-tenant cache hits in a naive design. That design was rejected.

---

## 10. State, Memory, and Features

The platform separates state that has different lifecycles:

| State | Where it lives | Why it is separate |
|---|---|---|
| Business lifecycle | Temporal event history | Must survive deploys and multi-day waits |
| Reasoning state | LangGraph checkpoint | Inspectable proposal path. Not the system of record. |
| Facts | Graph state, codes and span refs | Keeps raw PHI out of checkpoints and downstream prompts |
| Messages | Encrypted message store | Source of the spans. Not copied into prompts wholesale. |
| Features | Online store and point-in-time offline store | Same definition for the decision and the training row |
| Review decisions | Event log | Labels for autonomy, adapters, and the ranker |
| Metrics | Semantic layer | The digest narrator may query metrics. It may not write SQL. |

### Feature platform

The ranker does not ship on ad-hoc SQL. Offline-online comparison showed that ad-hoc features overstated offline ranking quality relative to shadow traffic. Feature views cover inquiry, clinician capacity, payer panel, and demand. Streaming compute maintains live availability: a clinician blocking a calendar slot stops being offered within the scenario budget (30 seconds). Training rows are point-in-time correct.

### Matching ranker

Hard constraints first: license, panel, modality, age band, language, caseload. Then a learned rank on fit, live availability, and graded outcomes — booked, attended, retained at 90 days. A booking-optimised ranker increased bookings and reduced 90-day retention in shadow analysis. That objective was rejected. Match overrides are stored with a reason and used as propensity-corrected training signal.

### Practice voice

Per-tenant LoRA adapters on an open-weight model, with direct preference optimisation on staff edit pairs. Frontier models are reserved for complex or sensitive threads. Deleting a tenant removes that tenant's adapter. Shared weights are unaffected. This is the deletion property required when a tenant offboards.

---

## 11. Model Gateway

Agents bind to a capability class, never to a model name. The gateway is the only egress for a model call.

Primary responsibilities:

- Provider-neutral request normalisation
- Routing by intent class, with a quality floor, a latency budget, a data-class eligibility rule, and a cost ceiling
- Refusal of calls that would exceed the remaining graph budget
- Circuit breaking per provider
- Failover to an equivalent class on the secondary cloud
- Prefix-cache eligibility
- Token and latency telemetry
- Cost attribution on every call: tenant, persona, intent, agent, endpoint, workflow run

```text
Agent step
    |
Capability class + data class + remaining budget
    |
Gateway policy
    |
 +-----------+----------------+
 |           |                |
Triton     vLLM            Frontier
encoders   tenant          (primary cloud,
rankers    adapters        secondary cloud
sentinel                   as failover)
```

About 70% of invocations stay on the self-hosted narrow tier. Managed frontier capacity is bought where the task is open-ended. TensorRT-LLM was evaluated for generative adapters and not adopted: multi-LoRA hot-swap mattered more than the throughput gain at this load.

### First-reply budget

The sub-two-minute reply is a budget. Compute for the auto-send path is ~27s at p95.

| Stage | Budget | Degraded mode |
|---|---:|---|
| Ingest, dedup, envelope | 1.5s | Queue and retry. Never drop. |
| Language identification | 0.2s | Practice primary language, flagged for review. |
| Safety sentinel | 1.8s | Fail closed. Crisis-resource template. No generative reply. |
| Structured extraction | 2.5s | Partial tags. A person completes them. |
| Policy and consent | 0.1s | Block outbound. |
| Draft | 6–12s | Pre-approved localised template. |
| Verifier | 2.0s | Template, or a human queue. |
| Autonomy and dispatch | 1.0s | Queue for approval. |

Product pressure to take safety off the critical path was refused. The accepted concession is a plainer first sentence on the slowest percentile, not an unsafe one.

---

## 12. Policy Enforcement and Autonomy

Authorisation and autonomy are different decisions, and neither is made by the agent that produced the proposal.

### Authorisation

```text
Tool call
    |
Capability token (entity + verb + expiry)
    |
Gateway: allow-list, data class, rate limit, idempotency
    |
Allow / Deny
    |
Side effect only if a workflow activity is the caller
```

A capability token is short-lived and scoped. An injected instruction cannot escalate past the token it was minted, and it cannot address an entity outside the current inquiry. Messaging is not callable by the model. Only a workflow activity may send, and only after the autonomy decision. EHR payloads are constructed deterministically. Credentials never enter a prompt.

Internal tools are exposed as MCP servers because the protocol gives a uniform, schema-described contract. The gateway adds what the protocol does not: per-call authorisation, capability tokens, egress policy, and audit. The Agent2Agent protocol is used at the organisational boundary — a partner billing agent, a referring system's care-coordination agent — not inside the platform.

### Autonomy

```text
mode = autonomy_policy(
  intent_class,
  risk_class,
  calibrated_confidence,
  tenant_autonomy_config,
  channel,
  language_tier,
  recent_quality_signal
) -> AUTO_SEND | STAGE_FOR_APPROVAL | ESCALATE | BLOCK
```

`recent_quality_signal` is the control loop. If the rejection rate for a tenant and intent breaches its limit, that intent is demoted until quality recovers. Low-resource languages are capped at human review. A static per-tenant autonomy setting was rejected because it kept auto-sending through a quality regression.

Human review is an interrupt in the graph and a signal-wait in the workflow. The decision form is approve, edit-and-approve, reject with a reason code, or take over. Takeover flips a flag and cancels pending timers atomically.

---

## 13. AI Guardrails and Prompt-Injection Defence

Traditional authorisation and AI safety are separate questions.

### Authorisation question

> Is this actor allowed to perform this action against this entity?

### AI safety question

> Is this interaction a crisis, an unsafe clinical claim, a PHI egress, or an injected instruction — including one that arrived inside a referral PDF?

Controls that do not depend on the model behaving:

- Safety sentinel as a fine-tuned multilingual ensemble on the critical path, with an LLM judge only in the ambiguous band, fail closed
- Verifier from a different model family than the drafter
- Placeholder rehydration, so identifiers are not in the prompt to be extracted
- Untrusted-content isolation for documents and retrieved chunks
- Intent-scoped tool allow-lists
- Mesh egress gateway with a destination allow-list
- Denied topics, content filters, and grounding checks on managed-model routes
- PII and PHI detection on egress
- Low-resource languages capped at A1

The red-team lifecycle:

```text
Threat model
    |
Attack scenarios (direct and indirect, every channel and document type)
    |
Controlled execution (PyRIT, Giskard, garak, promptfoo)
    |
Detection
    |
Mitigation as a boundary, not a prompt edit
    |
Regression tests on the golden and adversarial sets
    |
Production monitoring and automatic autonomy demotion
```

A release artifact for every new integration is an exfiltration-path analysis: every path from a PHI source to an external sink, each with a control and a test.

---

## 14. Durable and Event-Driven Coordination

Two coordination mechanisms, used for different failure modes.

**Temporal** owns the business lifecycle: multi-day follow-up, human waits, slot-hold expiry, referral loops, compensations, and the continuity runbook. A deploy in the middle of a four-day cadence must not double-send. A spike against a single agent framework failed that test.

**The event log** (Kafka, schema registry, outbox, change data capture) owns facts that other planes must observe: calendar changes into the feature store, graph materialisation, analytics, and audit. Consumers are idempotent. The inquiry envelope is delivered at least once and deduplicated.

Long-running work is not forced through the synchronous first-reply path. The first reply has a budget. The rest of the lifecycle has timers.

---

## 15. Scalability Strategy

The workload is not throughput-hard. It is latency-hard, correctness-hard, and isolation-hard, with a variable-latency model on the critical path. That single observation drives durable workflows rather than synchronous chains, and a small-model tier in front of frontier models.

> **Important:** The table below is the **planning load model**, extrapolated conservatively from observed practice volumes. It is not a claim of achieved fleet scale.

| Dimension | Planning assumption | Derived load |
|---|---|---|
| Tenants | 2,500 active practices | Long tail. Top 5% of tenants generate ~30% of volume. |
| Inquiries | ~300 per tenant per month | ~750k inquiries/month. ~0.3/s mean. 8–12/s Monday peak, burst factor 30×. |
| Conversation events | ~18 messages per inquiry lifecycle | ~13.5M messages/month |
| Model invocations | 9–14 per inquiry | ~9M calls/month. ~70% small or distilled. |
| Tokens | ~38k input and ~3k output per lifecycle before caching | ~28B input tokens/month before a 45–60% prefix-cache reduction |
| Voice | ~25% of inquiries include audio, mean 2.1 minutes | ~400k audio minutes/month |
| Calendar and EHR sync | ~40k clinicians × ~12 events/day | ~15M sync events/month |

Horizontal scaling boundaries are per tier: channel adapters, BFF, workflow workers, graph workers, Triton pools, vLLM pools, Flink, and feature serving. GPU quotas are per tenant tier so one tenant's backfill cannot starve first-reply inference. The validated production evidence remains the pilot-practice outcomes in section 4.

---

## 16. Reliability and Failure Containment

| Failure | Detection | Response |
|---|---|---|
| Model timeout or 5xx | Gateway circuit breaker per provider | Equivalent class on the alternate provider. If the class is exhausted, template path. |
| Schema-invalid output | Constrained decoding plus validator | One retry with the error. Then fallback model. Then template. |
| Verifier veto twice | Graph edge | Human queue with the defects attached |
| Tool call rejected | Tool gateway | Security event. Re-plan without that tool, or escalate. |
| Duplicate side effect | Idempotency key on workflow, activity, and attempt | Gateway returns the original result |
| Human takeover mid-graph | Workflow signal | Cancel the scope, release holds, suppress pending sends |
| Safety indeterminate | Sentinel verdict | Treat as elevated. Human inside the page objective. No generative outbound. |
| Primary model provider at ~40% errors | Breaker | No reply-latency breach beyond five minutes. No unsafe output. |
| Loss of the primary region | Regional failure | Core lifecycle RTO ≤ 60 minutes. Operational-state RPO ≤ 5 minutes. Reply path degraded but live within 15 minutes. |

Recovery is cross-region for state and cross-cloud for inference only. A symmetric active-active multi-cloud data plane was rejected: it roughly doubled the compliance surface for PHI at rest, outside the risk appetite. Continuity itself is a Temporal workflow with approval gates. Failover is rehearsed with fault injection, not asserted from a diagram.

Degraded modes are explicit: localised template, human queue, lexical-only retrieval, and fail-closed safety. A brown-out must not become a silent quality drop.

---

## 17. Observability and Real-Time Monitoring

Observability is correlated on one trace from channel ingest to the human decision.

### Platform telemetry

- Service health, queue depth, workflow task latency
- Kafka lag and feature freshness
- Regional and dependency health

### AI runtime telemetry

- Time to first reply, by tenant and channel
- Per-stage latency against the budget in section 11
- Safety verdict and sentinel version
- Model, prompt, and adapter versions on the span
- Token consumption and cache hit
- Tool-call sequence and handoff payload validity
- Verifier defects
- Human edit distance, approval, rejection, override

### Security telemetry

- Policy denials and rejected tool calls
- Prompt-injection attempts, including indirect injection in documents
- Egress blocks
- Tenant-boundary violations
- Takeover and automation lock

### FinOps telemetry

- Cost per inquiry lifecycle
- Cost per tenant, intent, agent, and endpoint
- Spend velocity against the tenant ceiling
- Share of calls served by the narrow tier

OpenTelemetry GenAI semantic conventions cover model, tool, and agent spans. The secondary cloud's telemetry is correlated on the same trace identifiers. Langfuse is used for LLM trace exploration. Lineage from a training row back to a decision record is an OpenLineage concern, not a log search.

---

## 18. AI FinOps

Cost is an engineering property because the commercial model is scale-based. Token volume is the platform's risk.

```text
Inquiry
  |
Tenant + intent + agent + workflow run
  |
Gateway pre-flight estimate
  |
Route or downgrade
  |
Tokens, cache outcome, endpoint
  |
Cost ledger
  |
Showback by tenant and tier
```

Mechanisms that are part of the production design:

- Small-model-first routing, with frontier models only where the capability class requires them
- Per-tenant LoRA adapters instead of a full fine-tune per practice
- Prefix caching and context discipline (45–60% of billable input removed)
- Budget objects on the graph, enforced at the gateway
- Spend-velocity anomaly detection with automatic tier downgrade
- Three-year TCO, including GPU reservations and on-call, as an input to every managed-versus-self-hosted decision
- Showback by tenant and tier. No per-query chargeback to the practice.

Semantic caching is not a FinOps lever on the inquiry path. It is limited to non-PHI knowledge and staff-copilot queries.

---

## 19. Cloud and Deployment Architecture

AWS is the primary cloud because its HIPAA-eligible managed surface covers the most of this stack with the least undifferentiated operations. Azure is attached for inference failover and disaster recovery, not as a second full estate.

| Concern | Decision |
|---|---|
| Tenancy topology | Multi-account landing zone. One account per cell for the largest networks. Shared-services account for the control plane and model factory. Separate audit account. |
| Compute | EKS per cell. Karpenter for CPU, GPU, and inference pools. KEDA on queue depth. |
| Mesh | Istio, strict mTLS, default-deny network policy, egress only through an allow-listed gateway |
| Workflow | Temporal |
| Agent graph | LangGraph, PostgreSQL checkpointer |
| Services | TypeScript staff workspace and GraphQL BFF. Python and FastAPI for AI services, with Pydantic contracts shared by REST, events, and tool schemas. Go for the tool gateway and channel adapters. gRPC internally. |
| Narrow inference | NVIDIA Triton for encoders, rankers, and the sentinel. vLLM with multi-LoRA for tenant voice adapters. |
| Frontier inference | Managed models on the primary cloud, under a BAA. Azure AI Foundry as the failover target. |
| Features and data | Feast, Flink, Iceberg on object storage, Aurora PostgreSQL with row-level security and pgvector, a multi-region online feature store, OpenSearch, Neo4j materialised by CDC |
| Training | Managed pipelines, Ray Train, PyTorch, LoRA, preference optimisation, gradient-boosted ranking. Model registry with signed artifacts. |
| Policy | OPA / Cedar for attribute-based policy, persona bundles, and autonomy |
| Delivery | Terraform in both clouds. One CI system, OIDC into both. GitOps and progressive delivery. Signed images admitted only if verified. |
| Observability | OpenTelemetry, Grafana, Langfuse, OpenLineage |

Deliberately not adopted: a second Kubernetes estate on the failover cloud, a second CI system, a globally distributed document database for state, semantic caching on the inquiry path, and the graph as a system of record.

Portability is preserved at four layers only: the model gateway, workflow and agent orchestration, feature definitions, and model artifact formats. Coupling below those layers is accepted, and the inference exit path is tested with live failover.

---

## 20. Architecture Decision Records

Decisions below were owned end to end: options framed, evidence commissioned, the call made, the cost accepted. Each follows **Context → Constraint → Options → Evidence → Decision → Trade-off**.

| ADR | Decision | Rejected | Evidence | Cost accepted |
|---|---|---|---|---|
| ADR-01 | Deterministic services own business state. Models only propose. | Agents with direct write access to scheduling and messaging | One injected instruction could double-book or message the wrong person | More boundary code. Slower feature velocity in the first two quarters. |
| ADR-02 | Temporal for the lifecycle, LangGraph for reasoning. Graphs run as activities. | One framework for both | A four-day cadence with a mid-flight deploy, a human takeover, and a provider outage double-sent on the single-runtime design | Two runtimes to operate |
| ADR-03 | Supervisor, typed handoffs, no inter-agent chat | Group chat | 3.1× token spend and non-reproducible tool sequences on the same inputs | Less emergent flexibility |
| ADR-04 | Fine-tuned safety ensemble, fail closed, on the critical path | Provider moderation, safety run asynchronously | LLM-only detection missed indirect ideation in Spanish below the recall floor. The ensemble met ≥ 0.99. | Labelling cost. A plainer first sentence at the tail. |
| ADR-05 | Self-host high-volume narrow models. Buy frontier capability. | All-managed, or all self-hosted | Three-year TCO plus p95 and p99 per capability class | GPU planning and serving on-call |
| ADR-06 | Per-tenant LoRA plus preference optimisation on edit pairs | Prompt-only style. A shared voice model. Full fine-tune per tenant. | Blind preference test, edit distance, and the tenant-deletion requirement | Adapter lifecycle at hundreds of tenants |
| ADR-07 | Feature platform with point-in-time correctness before the ranker ships | Ad-hoc SQL features. Ship the ranker first. | Ad-hoc features overstated offline ranking quality versus shadow traffic | A quarter of platform work before the ranking gain is visible |
| ADR-08 | Graded outcome labels: booked, attended, retained | Optimise for the booking | Booking-optimised ranker increased bookings and lowered 90-day retention in shadow | Slower labels |
| ADR-09 | Placeholder rehydration. Identifiers stay out of prompts. | Instruction-based PHI avoidance | Red-team campaigns extracted identifiers from instruction-only designs | A template step on the dispatch path |
| ADR-10 | Verifier from a different model family | Self-critique by the drafter's family | Same-family judging approved a large share of its own factual errors | A second provider |
| ADR-11 | Autonomy as a policy with a live quality term | Static per-tenant autonomy | Static autonomy kept auto-sending through a multi-day quality regression | Automatic demotions that have to be explained |
| ADR-12 | Cross-region recovery for state. Cross-cloud failover for inference only. | Symmetric multi-cloud for data at rest | Full multi-cloud data plane roughly doubled the compliance surface | Dependence on one cloud for state-plane recovery |
| ADR-13 | Knowledge graph derived by CDC, never authoritative | Graph as the system of record | Drift analysis of authoritative-graph designs | A second query path to keep honest |
| ADR-14 | Semantic caching only for non-PHI classes | Semantic cache on the inquiry path | Near-duplicate prompts across tenants produced cross-tenant hits | Lower cache hit rate |
| ADR-15 | Egress allow-list at the mesh, not only in application code | Application checks alone | Exfiltration analysis found agent-reachable paths that application checks did not cover | Mesh operational overhead |
| ADR-16 | Low-resource languages capped at human-review autonomy | Uniform autonomy across languages | Per-language recall and translation adequacy below the floor on the long tail | Slower replies in those languages |

### The three hardest calls

**Latency versus safety on the first reply.** The generative reply was wanted inside 30 seconds with safety asynchronous. Safety stayed on the path. The sentinel's first layers moved onto the narrow serving tier, and language and extraction ran in parallel around it. The reply objective was met with the barrier intact.

**Managed versus self-hosted inference.** The split was settled per capability class with a two-week spike on p95, p99, and cost per 1,000 calls, plus three-year TCO including reservations and on-call. Narrow and high-volume is self-hosted. Frontier capability is bought.

**Autonomy that can be taken away.** A system that demotes itself looks less capable. Autonomy is still withdrawn automatically when rejection rate breaches its control limit. That cost uncomfortable conversations. It also meant the platform did not auto-send through a quality regression.

---

## 21. Demonstration Catalog

The narrated walkthrough and the Northline console are a staff-workspace demonstration of the decisions above. They are synthetic data. They are not a practice's records, and they do not execute the production model gateway.

| Scene | What it is designed to show | Architectural property |
|---|---|---|
| Command | Open inquiries, safety holds, median first reply against the two-minute objective, morning cost | Outcome and cost on one screen |
| Inbox | Web, SMS, voicemail, referral, directory, and reschedule in one queue. A Sunday 21:42 Spanish inquiry inside the reply budget. A separate SMS already on the crisis path. | Unified capture. Fail closed with no draft. |
| Inquiry | Language, sentinel verdict and version, facts as codes and span references, staged draft, verifier defects, in-language edit, approve, takeover | Generate-verify-repair. Human accountable for the send. |
| Matching | Eligible clinicians only. Override requires a reason. Hold is short-lived and confirmed by a person. | Hard constraints, then rank. Override as a future label. |
| Intake | Deterministic screener scoring. Typed handoff to the clinical record. | The model does not interpret the instrument. |
| Referrals | Closed-loop status staged for the referring provider | Clinical detail excluded from the outbound draft |
| Agent studio | Add an agent as a full artifact: charter, tier, capability class, tool allow-list, autonomy ceiling, persona, data classes, identifier exclusion, schema, eval suite, degraded mode, side-effect class, cost ceiling | An agent is a governed artifact, not a prompt |
| Orchestration | Supervisor, safety barrier, verifier, autonomy decision. Budget remaining. | Typed handoffs and a visible ceiling |
| Governance | Router mix, kill switches, audit ledger of the actions just taken | The review UI writes the evidence store |
| Capacity | Demand against available hours, hiring signal, digest | Numbers come from the metric tool |

The agent published in the walkthrough is a Waitlist Narrator at A1. It may explain position and likely time to a first session from forecast features. It may not promise a date the schedule cannot keep.

---

## 22. Repository Structure

```text
.
├── README.md
├── public/
│   └── northline-walkthrough.mp4
├── screenshots/
│   └── 01-command.png ... 11-capacity.png
└── src/
    ├── routes/index.tsx
    └── components/northline/
        ├── shell.tsx
        ├── screens.tsx
        ├── data.ts
        └── store.ts
```

| Path | What it is |
|---|---|
| `README.md` | This case study |
| `public/northline-walkthrough.mp4` | Narrated walkthrough of the golden path, including agent creation |
| `screenshots/` | Still frames of the same console |
| `src/components/northline/` | The console: shell, screens, synthetic seed data, client state |

### Run the console

```bash
npm install
npm run dev
```

Open `http://localhost:8080`.

The console is a local, single-tenant demonstration. Publishing an agent, approving a send, overriding a match, and placing a hold write to the on-screen audit ledger. There is no network call to a model provider.

---

## 23. Principal AI Architect Perspective

Role on the programme: sole principal AI architect, accountable for platform architecture across the lifecycle — control plane and data plane, technology selection, trust and safety release gates, cost governance, and resilience. Peak team of 28 across AI, agent engineering, ML platform, data platform, SRE, DevOps, QA, clinical safety, and product.

The case study is structured to show architecture ownership across:

- Operational problem to architecture
- Non-functional requirements and quality-attribute scenarios
- Distributed-systems decomposition
- Agent runtime architecture under a clinical-safety constraint
- Retrieval, features, and delayed labels
- Cloud architecture and a deliberate limit on multi-cloud scope
- Security, tenancy, and governance
- Reliability engineering
- Latency budgets
- AI observability
- FinOps under a pricing model the platform cannot pass through
- Architecture governance
- Production validation on pilot practices
- Technical leadership, including three decisions that were refused

The emphasis is not on listing technologies. It is on the reasoning chain:

```text
Operational requirement
        |
Constraint (safety, tenancy, latency, deletion, unit cost)
        |
Architectural problem
        |
Options
        |
Evaluation criteria or spike
        |
Architectural decision
        |
Trade-off accepted
        |
Implementation boundary
        |
Production or eval evidence
```

The defensible position of this platform is not the language model. The same models are available to anyone. The durable advantage is the closed loop: a decision is made with reproducible features, a person edits or overrides it, the outcome arrives days later, the three are joined into a labelled record, and the next ranker, adapter, sentinel, and autonomy policy is measurably better for that tenant. None of the hardest risks — low-resource-language safety, feedback-loop bias, tenant memorisation, cost exposure, provider outages — was solved by a better prompt. Each was solved by an architectural boundary, a measurement, and a control loop.

---

## 24. Confidentiality and Anonymization

This repository is an **anonymized architecture portfolio artifact**.

Practice names, customer names, employer names, credentials, secrets, internal URLs, and confidential implementation details are intentionally excluded. The staff console uses the demonstration name Northline and synthetic inquiries.

The narrative is intended to communicate architectural thinking, system design, engineering trade-offs, and production-oriented reasoning without exposing confidential information.

---

## 25. Author

**Jyotirmoy Bardhan**  
Principal AI Architect — AI Platforms & Distributed Systems

Architecture interests represented in this case study:

`Agentic AI` · `Distributed Systems` · `Enterprise AI Platforms` · `HIPAA-aligned AI` · `GraphRAG` · `AI Governance` · `AI Security` · `SRE` · `LLMOps` · `FinOps` · `Architecture Governance`
