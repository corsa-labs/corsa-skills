---
name: corsa-workflow-scoping
description: >
  Translate a compliance policy or requirement into a short, grounded workflow description
  that a downstream workflow-builder agent will implement. Use when scoping a workflow from a
  policy, deciding what a compliance workflow should do, or writing the requirement/spec that
  another agent will build. Keeps the description strictly within Corsa's real workflow
  capabilities so it never asks for features that do not exist.
---

# Corsa Workflow Scoping (Policy → Workflow Description)

You turn a **compliance policy** (or a clause/obligation from one) into a **1–2 paragraph workflow
description** that a downstream coding agent will build with the full workflow-builder skill. Your
description is a *specification of intent*, not an implementation — no JSON, no node ids, no field
schemas. Your one hard rule: **describe only what Corsa workflows can actually do.** Everything you
ask for must map to a capability below; if the policy needs something outside the envelope, say so
explicitly instead of inventing it.

## What Corsa workflows are (mental model)

A workflow **reacts to a compliance event, or runs on a schedule, and then performs an ordered set
of automated steps** — screening, risk assessment, opening/updating alerts and cases, notifying
analysts, emailing clients, AI analysis — with optional conditional branching. Workflows are for
**orchestration and follow-up**, not real-time transaction blocking (that is the transaction-monitoring
*rules* engine — see "Not supported"). Keep descriptions within this shape.

## Capability envelope (what you may ask for)

**How a workflow starts (pick one):**
- **On an event** — when an entity is *created* or *updated*: an **individual client**, **corporate
  client**, **alert**, **case**, or **transaction**. You can narrow it with filters on that entity's
  fields (e.g. risk level, status, category, country, amount, sanctions/PEP flags, dates). Common
  patterns: a client's risk rating rises to HIGH, a new high-risk client is onboarded, a new alert or
  case is opened, a large or high-risk transaction posts.
- **On a schedule** — periodically scan **individual or corporate clients** matching filters (e.g.
  dormant, overdue for periodic review) on an interval (minimum 5 minutes, up to yearly) and process
  the batch.

**Steps a workflow can perform:**
- **Screening & due diligence** — screen a client through integrations; run **PEP/sanctions screening**
  (person / company / entity, optionally auto-raising an alert on a match); **initiate deep research**
  on a client.
- **Risk** — **run a risk assessment** (a risk-rating formula) on the client/entity.
- **Analyst tasks & cases** — **request documents** from a client, **request information**, open a
  **periodic-review** task, or **create an interaction/note** (these create cases/alerts for analysts,
  with a due date).
- **Records** — **create alerts or cases**; **update** alerts, cases, clients, or transactions (e.g.
  set a client's risk level, change a status, **escalate** an alert to a case). Conditions can gate an
  update on the entity's current state.
- **Communications** — **send a templated email to the client**; **notify analysts** in-app, by email,
  or via Slack.
- **AI / Copilot analysis** — run an **AI step** that analyzes the entity (and prior steps' outputs)
  and returns a **structured result** (e.g. `isSuspicious` + reasoning, a classification, a narrative).
  It can consult **read-only** tools (entity lookup, screening data, web search). Use it for triage,
  narrative generation, or routing — but it decides, it does not act; any resulting action is a separate
  downstream step.
- **Control flow** — **branch** on a condition (an entity field, or a prior step's output such as an AI
  verdict); run steps in sequence or in parallel; optionally **repeat** a step on an interval.

## Not supported — do NOT put these in a description

- **No custom code / arbitrary scripts.** If a requirement needs bespoke logic, express it as AI
  analysis + branching, or flag it as a gap.
- **No real-time transaction blocking / hold / freeze.** Workflows *react after* an event; they cannot
  synchronously stop a transaction. Blocking/`FREEZE` is the **transaction-monitoring rules** engine —
  scope that as a rule, not a workflow.
- **Triggers are limited** to create/update of client, alert, case, and transaction. No arbitrary
  system events, no "when another workflow runs," no direct deposit/withdrawal triggers (those surface
  as transaction events).
- **Scheduled scans cover clients only** (individual/corporate) — not alerts, cases, or transactions.
- **Record creation is limited to alerts and cases.** Clients and transactions can be *updated* but not
  *created* by a workflow.
- **External reach is limited** to the Corsa services above plus an **outbound webhook** — no arbitrary
  third-party API integrations beyond a webhook call.
- **Scale is bounded.** Workflows are rate-limited per definition and scheduled runs are capped — don't
  describe high-frequency, high-fan-out, or "run on every transaction instantly" behavior.

If the policy demands something here, write it into the description as an explicit note (e.g.
*"Policy §4.2 requires blocking the payment; that is a transaction rule, not a workflow — out of scope
here"*) rather than pretending the workflow can do it.

## How to write the description

1. **Extract the obligations** from the policy: the *trigger condition*, the *required control/action*,
   any *timeline/SLA*, *escalation path*, *notification* duty, and *record-keeping* requirement.
2. **Map each obligation to a capability** in the envelope. Note gaps explicitly; never invent a feature.
3. **Write 1–2 paragraphs**, in plain English, that state:
   - the **trigger** (event + filter, or schedule + population);
   - the **ordered steps** and any **conditional branches** (with their criteria);
   - what **records** are created/updated and **who is notified**;
   - the **compliance driver** — cite the policy clause/obligation behind each part, including timelines,
     escalation, and record-keeping the policy mandates.
4. **Stay implementation-agnostic**: name capabilities in platform terms ("run a PEP/sanctions
   screening", "open a periodic-review task", "escalate the alert to a case") but do **not** specify JSON,
   node ids, or field names — that is the builder agent's job. Keep it to 1–2 paragraphs.

## Output contract

A single 1–2 paragraph description that a workflow-builder agent can implement directly. It must contain:
**trigger**, **steps/branching**, **records + notifications**, and the **explicit compliance requirement**
(with policy citation and any SLA/escalation/record-keeping). No implementation detail; no unsupported
features; flag any gap.

## Worked example

**Policy excerpt:** *"When an individual customer's risk rating is raised to High, the customer must
undergo enhanced due diligence: a sanctions and PEP re-screening and an analyst review completed within
3 business days. If a sanctions match is identified, a high-priority alert must be raised and the MLRO
notified. The screening outcome must be recorded on the customer's case." (AML Policy §5.3)*

**Workflow description (output):**
> Build an event-driven workflow that triggers when an **individual client's risk level changes to
> HIGH**. On trigger, run a **PEP/sanctions re-screening** of the client and, in parallel, open an
> **enhanced-due-diligence review task** for an analyst with a **3-day due date** (AML Policy §5.3 EDD
> obligation and review SLA — note: the policy specifies *business* days, which the builder should
> approximate with a calendar-day due date unless business-day handling is available). Then **branch on
> the screening result**: if a sanctions/PEP **match is found**, **create a HIGH-priority sanctions
> alert** and **notify the MLRO in-app and by email**; otherwise **record the clear screening outcome on
> the client's case**. This satisfies §5.3's EDD-on-risk-escalation trigger, the analyst-review SLA, the
> match-handling and MLRO-notification duties, and the screening-record-keeping requirement.

Note how it: stays inside the envelope; names capabilities in platform terms; embeds the policy citation
and the SLA/escalation/record-keeping obligations; flags the business-day nuance instead of inventing a
feature; and leaves all implementation (nodes, fields, JSON) to the builder agent.

## Guardrails

- Never describe a feature outside the envelope — map to a real capability or flag the gap in-line.
- Keep to **1–2 paragraphs**; be specific about *what* and *why* (the compliance driver), not *how*.
- Distinguish **workflow** (react + orchestrate) from **rule** (synchronous transaction decision) — send
  blocking/hold requirements to the rules track.
- When unsure whether a capability exists, describe the intent and add a short "builder to confirm"
  note rather than asserting a feature.
