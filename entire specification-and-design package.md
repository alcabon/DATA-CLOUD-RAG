The architect and data-cloud-eng agents can draft essentially the entire specification-and-design package — the interesting question is which artifacts are *distinctive* to an agentic/Agentforce build versus the generic software ones. Here's the catalogue, grouped, with what each contains and which agent in our setup would own it.

## Requirements & specification

**Vision & scope / solution brief** — the problem, business goals and success metrics (containment rate, CSAT, average handle time), and explicit in/out-of-scope. This frames everything downstream.

**Functional / capability specification** — per topic (rebooking, loyalty, baggage, policy FAQ): what the agent must do, the inputs and outputs, and the business rules (rebooking eligibility, fare-rule logic, tier benefits, baggage SLA windows).

**User stories with acceptance criteria** — Gherkin-style given/when/then. These do double duty: they're the spec *and* the basis the `qa` agent turns into Testing Center scenarios, so generating them well pays off twice.

**Conversation design specification** — the artifact most specific to conversational agents: sample dialogues for happy paths and edge cases, the agent persona and tone, disambiguation flows, and the exact phrasing for refusals and human escalation. AI is genuinely strong here because it can generate dozens of realistic dialogue variants.

**Intent & utterance catalogue + topic taxonomy** — the mapped set of intents, example utterances per topic, and the routing boundaries between topics. This feeds the topic classification descriptions directly.

**Non-functional requirements** — latency, availability, peak-irrops throughput (a snowstorm is your load test), languages/localization, and accessibility.

## Architecture & design

**Solution Architecture Document (HLD)** — the layered view (channel → agent/topics → grounding/integration → governance), component responsibilities, and integration topology.

**ADRs (Architecture Decision Records)** — already in the pipeline; the per-decision records capturing the action-type mapping and why.

**Topic & action design spec** — per topic: classification description, scope instructions, the attached actions, and each action's contract (signature, inputs/outputs) plus its action-type decision (transactional vs structured-read vs live-read vs RAG).

**Integration / API design** — the gateway contracts (OpenAPI specs or references), sequence diagrams for the rebooking confirm-and-commit loop and the baggage lookup, idempotency design, and the error/timeout/Named-Credential plan.

**Prompt & instruction spec** — agent-level and topic-level instructions and guardrail prompts, written as a reviewable artifact rather than buried in metadata.

**Sequence & flow diagrams** — conversation flows, the commit flow, escalation-to-human, and Omni-Channel routing.

## Data & grounding design (owned by `data-cloud-eng`)

**Data model / DMO design** — the data dictionary, source-to-DMO mapping, and identity-resolution rules, with the structured-vs-unstructured matrix that decides what's a DMO lookup and what's RAG.

**Grounding & RAG design** — retriever inventory, chunking and indexing strategy, knowledge sources, and the citation-metadata mapping so answers stay traceable.

**Prompt template specs** — the Prompt Builder templates, with the sample query + grounded answer that proves each retriever works.

## Cross-cutting (security, test, ops, governance)

**Security, privacy & compliance design** — Trust Layer configuration, PII masking policy, RBAC/permission model, data residency, audit/logging design, and a threat model that must include prompt injection and data exfiltration. Passenger data (PNR, GDPR) raises the bar here.

**Test strategy & plan + requirements traceability matrix** — the scenario design including negative paths, and the requirement → topic/action → test matrix that proves coverage.

**Observability & evaluation design** — what to log, the KPI set, the eval rubric, the rollout plan (rainbow/canary), and drift monitoring.

**Environment, deployment & release design + runbook** — org strategy (scratch/sandbox/prod), CI/CD, rollback, and incident handling.

**Risk register, assumptions log, and a token/cost budget model** — the latter ties to our last conversation: the build-pipeline token budget plus a runtime consumption estimate (now in Agentic Work Units rather than raw tokens).

## Two things worth saying plainly

First, where AI adds the most unique value isn't authoring any single document — it's *cross-document consistency*. Keeping the intent catalogue, topic taxonomy, acceptance criteria, and test scenarios mutually aligned is tedious and error-prone by hand, and an AI that holds them together (and regenerates the traceability matrix on every change) is a real advantage over manual authoring.

Second, the validation boundary from my earlier self-assessment applies sharply here. AI can *draft* all of the above, but several are plausible-but-sometimes-wrong on domain specifics and must be SME-validated before they're authoritative: the data model against the real source systems, anything touching fare rules or aviation/passenger-data regulation, and any document that becomes a binding contract (API contracts, acceptance criteria, the compliance design). Treat AI output as a strong first draft that compresses weeks of blank-page work, not as signed-off truth.

One gap in our current roster: design-phase documents like the conversation spec, intent catalogue, and functional spec sit upstream of the `architect` and `data-cloud-eng` agents we defined. If these documents matter to you, the clean move is to add a **business-analyst / conversation-designer agent** (Opus or Sonnet) that owns requirements and dialogue design and feeds the architect. I can sketch that agent in the same `.claude/agents/` format, or draft a template for any one of these documents — the conversation design spec and the topic & action design spec are usually the highest-leverage starting points.
