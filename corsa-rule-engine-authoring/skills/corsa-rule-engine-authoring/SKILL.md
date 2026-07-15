---
name: corsa-rule-engine-authoring
description: >
  Author, test, and manage Corsa transaction-monitoring rules directly against
  the rule-engine-service — the deep, engine-accurate companion to the SDK-only
  corsa-rule-authoring skill. Use when writing or editing compliance rules,
  translating a natural-language policy into rule JSON, building conditions,
  thresholds, or aggregation/velocity time-windows, wiring CREATE_ALERT /
  HALT_TRANSACTION actions, choosing entity properties and comparison operators,
  evaluating or back-testing a rule before activation, managing the
  draft/active/disabled lifecycle and versioning, or working with rule
  templates, external (vendor) rules, dynamic per-client thresholds, or the
  evaluation/facts and evaluate-and-execute endpoints.
---

# Corsa Rule Engine Authoring (Deep)

You are a Corsa transaction-monitoring rule specialist working at the level of
the real engine (`rule-engine-service`), not just the SDK surface. Your job is
to turn a natural-language compliance policy ("alert when a high-risk customer
withdraws more than $50k in 24h") into a valid, deployable rule, test it against
history, and activate it safely.

This file is self-sufficient: every enum, operator, field, endpoint, and limit
below is transcribed from source. It supersedes the public `corsa-rule-authoring`
skill where they disagree — the drift table says where and why.

---

## Table of contents

1. Mental model
2. Source of truth & known drift
3. Interfaces (REST v1, SDK, tenancy, the AI "agents" surface)
4. Rule anatomy — the only four fields
5. Lifecycle & versioning
6. Creating rules — template / custom / external
7. Conditions — complete reference (nesting, fields, operators, aggregation, valueRef)
8. Actions — CREATE_ALERT, HALT_TRANSACTION, consolidation
9. Evaluation & testing — sync vs on-demand, decisions, back-tests, fact discovery
10. Entity / property catalog
11. Validation rules & limits
12. Worked examples (complete JSON payloads)
13. Best practices
14. Common pitfalls
15. Error handling
16. Source map & doc links

---

## 1. Mental model

A **rule** is `conditions` (the *when*) plus `actions` (the *what*), evaluated
against a **transaction** in the context of its participants (client, wallet,
bank account) on both the sending and receiving side.

```
transaction ─▶ [ rule engine ] ─▶ conditions match?  ── yes ─▶ run actions ─▶ decision (ALLOW | FREEZE)
                                                       ── no  ─▶ (record non-match)
```

Under the hood the engine is [`json-rules-engine`](https://www.npmjs.com/package/json-rules-engine).
You author conditions in a friendly shape (`entity` + `property` + `operator` +
`value`, or an `aggregation*` block). On save, the service **normalizes** that
into engine-native facts: each `entity/property` pair is hashed into a `fact`
id (`fact_<32hex>`, `agg_<32hex>`, `threshold_<32hex>`) and compiled into a
stored `jreRule`. **You never write fact hashes by hand** — write `entity` +
`property` and let the normalizer do it (`condition-normalizer.ts`).

Key consequences of the engine being fact-based:
- A **direct** condition reads one property of one entity/participant.
- An **aggregation** condition rolls up *many* historical transactions
  (velocity, structuring, dormancy, ratios) and compares the aggregate to a
  static `value` or a dynamic client-field `valueRef`.
- A **null/undefined fact** makes comparison operators return `false` (the rule
  simply doesn't fire) rather than throwing — safe by default.

---

## 2. Source of truth & known drift

Source code in `rule-engine-service` wins over the README and the public skill.
Facts below are transcribed from `src/` on commit `485e1a9` (internal-docs
snapshot) and verified live. Drift you should know about:

| Claim in README / public `corsa-rule-authoring` | Reality in code | Where |
|---|---|---|
| Rules have a `priority` and a `scope` field | **Removed.** Rule DTO is only `{ name, description?, conditions, actions }`. `priority` now exists **only** inside `CREATE_ALERT` config. | migration `1765550000000-RemovePriorityAndScope`; `create-rule.dto.ts` |
| Lifecycle is "Draft → Active → Disabled" (+ a `deleted` state) | Statuses are exactly `draft`/`active`/`disabled`. Delete is a **soft-delete** (`deletedAt`), not a status. You can activate from **draft or disabled**. | `rule.enum.ts` `RuleStatus`; `rule.service.ts` |
| Aggregation operators = 11 (sum…last) | **27** operators, incl. velocity/behavioral (`velocityChange`, `dormantReactivation`, `directionalRatio`, …). | `rule-condition.interface.ts` `AggregationOperatorType` |
| Aggregation periods = minutes…years (6) | **9** — adds `calendar_weeks`, `calendar_months`, `calendar_years`. | `AggregationTimePeriod` |
| Rule-test routes are `POST /v1/rules/:ruleId/tests`, `GET /v1/rules/:ruleId/tests/:testId` | Actual routes are `POST /v1/rule-tests/:ruleId`, `GET /v1/rule-tests/:testId`. README is **stale**. | `rule-test.controller.ts` |
| No dynamic thresholds | `valueRef` lets an aggregation compare against a **client field** (e.g. `> 60% of dailyTransactionLimit`). | `create-rule.dto.ts` `EntityFieldThresholdRefDto` |
| No external rules; single evaluate endpoint | There is a full **external (vendor) rules** surface, a separate **`evaluate-and-execute`** path, and an AI **narratives** endpoint. | `external-rule.controller.ts`, `sync-evaluation.controller.ts`, `evaluation.controller.ts` |
| `src/evaluation/agents` is AI rule generation | It is **AI narrative generation** (evaluation → prose). There is **no** natural-language-to-rule endpoint anywhere. | `narrative.agent.ts` |

---

## 3. Interfaces

### 3.1 REST v1 (authoritative)

All routes are versioned `v1`. `platformId` is derived from the request context
(auth/gateway), **never** from the body or a query param — this is the tenant
boundary; every stored row and query is scoped to it.

**Rules — `rule.controller.ts`**

| Verb & path | Purpose | Body / query |
|---|---|---|
| `POST /v1/rules` | Create rule (starts as `draft`) | `CreateRuleDto` |
| `PUT /v1/rules/:id` | Update rule (versioned if active) | `UpdateRuleDto` |
| `POST /v1/rules/:id/activate` | Activate (draft/disabled → active) | `{ reason? }` |
| `POST /v1/rules/:id/disable` | Disable (active → disabled) | `{ reason? }` |
| `DELETE /v1/rules/:id` | Soft-delete (non-active only) | `?reason=` (query) |
| `GET /v1/rules/:id` | Get one (optional `?version=`) | — |
| `GET /v1/rules` | List (paginated, filterable) | — |
| `GET /v1/rules/:id/audit` | Audit trail for the rule | — |

**External (vendor) rules — `external-rule.controller.ts`** *(informational only; not evaluated)*

| Verb & path | Purpose |
|---|---|
| `POST /v1/external-rules` | Register a vendor rule (created **active**) |
| `PUT /v1/external-rules/:id` | Update in place |
| `DELETE /v1/external-rules/:id` | Soft-delete (`?reason=`) |
| `GET /v1/external-rules` | List (paginated) |
| `GET /v1/external-rules/:id` | Get one |
| `GET /v1/external-rules/vendors` | Distinct vendor names |
| `GET /v1/external-rules/name-exists?name=` | `{ exists: boolean }` |

**Rule templates — `rule-template.controller.ts`**

| Verb & path | Purpose |
|---|---|
| `GET /v1/rule-templates` | List templates (paginated) |
| `GET /v1/rule-templates/:id` | Template detail |
| `POST /v1/rule-templates/:id/copy` | Copy template into your workspace as a **draft** rule |

**Evaluation — `evaluation.controller.ts` / `sync-evaluation.controller.ts`**

| Verb & path | Purpose | Executes actions? |
|---|---|---|
| `POST /v1/evaluation/evaluate` | On-demand: evaluate a transaction against active rules | **No** — returns matches + decision only |
| `POST /v1/evaluation/evaluate-and-execute` | Synchronous inline evaluation during ingestion | **Yes** — creates alerts / freezes |
| `GET /v1/evaluation/facts` | Discover the full fact catalog + operator/aggregation option lists | — |
| `GET /v1/evaluation/transaction/:transactionId/results` | Evaluation history for a transaction (paginated) | — |
| `GET /v1/evaluation/rule/:ruleId/results` | Evaluation history for a rule (paginated) | — |
| `POST /v1/evaluation/narratives` | AI narratives for prior evaluations (explanation, not authoring) | — |

**Rule testing (back-tests) — `rule-test.controller.ts`**

| Verb & path | Purpose |
|---|---|
| `POST /v1/rule-tests/:ruleId` | Start an async back-test of a rule (or an unsaved candidate) against historical transactions |
| `GET /v1/rule-tests/:testId` | Test results with path analysis (query: `page`, `pageSize`, `pathNumber`, `matchedOnly`) |
| `GET /v1/rule-tests` | List test runs (paginated) |
| `PATCH /v1/rule-tests/:testId` | Rename a test run |
| `DELETE /v1/rule-tests/:testId` | Soft-delete a test run |

### 3.2 SDK (`@corsa-labs/sdk`) — convenience wrapper

The SDK wraps the core rule/template/evaluation endpoints. Method names below
are the documented surface; when in doubt, the REST contract above is
authoritative, and newer surfaces (external rules, rule-tests,
evaluate-and-execute, narratives) may be REST-only in your SDK version.

```typescript
client.rules.createRule(body)                 // POST /v1/rules
client.rules.listRules(page?, limit?, ...)    // GET  /v1/rules
client.rules.getRule(id, version?)            // GET  /v1/rules/:id
client.rules.updateRule(id, body)             // PUT  /v1/rules/:id
client.rules.activateRule(id, { reason? })    // POST /v1/rules/:id/activate
client.rules.disableRule(id, { reason? })     // POST /v1/rules/:id/disable
client.rules.deleteRule(id, reason?)          // DELETE /v1/rules/:id

client.ruleTemplates.listRuleTemplates(page?, limit?)
client.ruleTemplates.getRuleTemplate(id)
client.ruleTemplates.copyRuleTemplate(id)     // → { ruleId } (draft)

client.evaluation.evaluate(body)              // POST /v1/evaluation/evaluate
client.evaluation.getTransactionEvaluations(txId, page?, pageSize?)
client.evaluation.getRuleEvaluations(ruleId, page?, pageSize?)
```

### 3.3 The AI "agents" surface — narratives, not authoring

`src/evaluation/agents/narrative.agent.ts` is Corsa's only LLM surface in this
service. It is a **compliance-narrative generator**: given already-computed
`conditionResults` for an evaluation, it emits one plain-English sentence per
evaluation for the review UI (`POST /v1/evaluation/narratives`). It runs
`claude-sonnet-4-5` (temp 0.1, maxTokens 2000) via `@corsa-labs/agent-orchestrator`
→ Corsa `ai-service`, is cache-first, and soft-fails to `[]` on error.

It **does not** author, generate, or modify rules — the prompt forbids it from
even naming rules or outcomes. There is **no** natural-language-to-rule endpoint
in the service (a dormant `agentConfig` fact stub exists but is unused). So: to
"generate a rule from a prompt," *you* (the coding agent) produce the
`CreateRuleDto` JSON using this document — the engine has no NL authoring API.

---

## 4. Rule anatomy — the only four fields

`CreateRuleDto` (`create-rule.dto.ts`). There is **no** `priority`, `scope`,
`enabled`, or `status` field on input — status is controlled by the lifecycle
endpoints, not the body.

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | `string` | ✅ | Non-empty (trimmed). No max-length at DTO layer; keep it short and human-readable. |
| `description` | `string` | ❌ | Free text; helps audit narratives. |
| `conditions` | `ConditionsDto` | ✅ | `{ all?: [], any?: [] }`; must have at least one non-empty condition. |
| `actions` | `ActionConfigDto[]` | ✅ | Array of `{ type, config }`. At least one. |

```jsonc
{
  "name": "High-risk withdrawal detection",
  "description": "Alert when high-risk customers withdraw > $50k in 24h",
  "conditions": { "all": [ /* ...conditions... */ ] },
  "actions": [ { "type": "CREATE_ALERT", "config": { "category": "TRANSACTION_MONITORING", "priority": "HIGH", "status": "NEW" } } ]
}
```

`UpdateRuleDto` is the same fields, all optional, plus `reason?` (recorded in
the audit trail when you edit an active rule).

---

## 5. Lifecycle & versioning

Status enum `RuleStatus` = `draft` | `active` | `disabled` (`rule.enum.ts`).
There is no `deleted` status; deletion sets `deletedAt` (soft-delete).

```
                 activate                     disable
   ┌────────┐  ───────────▶  ┌────────┐  ───────────▶  ┌──────────┐
   │ draft  │                │ active │                │ disabled │
   └────────┘  ◀───────────  └────────┘  ◀───────────  └──────────┘
        │        (edit active = new version)   activate      │
        │                                                    │
        └──────────────── delete (soft) ◀────────────────────┘
              (delete allowed only when NOT active)
```

- **Only `active` rules evaluate** incoming transactions automatically.
- **Activate**: allowed from `draft` **or** `disabled`. No-op if already active.
- **Disable**: only from `active` (else `400 "Only active rules can be disabled"`).
- **Delete**: blocked while `active` (`400 "Active rules cannot be deleted. Disable the rule first"`); otherwise soft-deletes.
- **Editing a `draft`/`disabled` rule**: mutates the same row in place, same version.
- **Editing an `active` rule = automatic versioning** (`updateActiveRuleInternal`):
  in one DB transaction the old version row flips to `disabled` and a **new row**
  with `version + 1` is inserted as `active`; then the platform engine reloads.
  The old version is preserved for audit and retrievable via `GET /v1/rules/:id?version=N`.
- **Audit**: every change writes a `RuleAudit` row with an `AuditAction`
  (`create`, `edit`, `activate`, `disable`, `update_active`, `delete`), the actor,
  a JSON `diff`, and the optional `reason`. Always pass `reason` on
  activate/disable/delete/active-edit.

---

## 6. Creating rules

### 6.1 From a template (fastest)

The service seeds a template library on startup. Current templates
(`templateKey`): `threshold-avoidance`, `interaction-with-high-risk-wallet`,
`rapid-account-activity-shortly-after-onboarding`, `risk-based-volume-thresholds`.

```typescript
const templates = await client.ruleTemplates.listRuleTemplates(1, 20);
const { ruleId } = await client.ruleTemplates.copyRuleTemplate("<template-uuid>"); // → draft
await client.rules.updateRule(ruleId, { conditions: { all: [ /* tuned thresholds */ ] } });
await client.rules.activateRule(ruleId, { reason: "Tuned for our risk profile" });
```

### 6.2 Custom rule

Create → **back-test** (§9.4) → activate. Never activate blind.

```typescript
const rule = await client.rules.createRule({ name, description, conditions, actions });
// POST /v1/rule-tests/{rule.id}  → inspect alertRatio & paths
await client.rules.activateRule(rule.id, { reason: "Back-test alert ratio 0.4%, approved" });
```

### 6.3 External (vendor) rules

For recording rules that run in a **third-party** system so Corsa can attribute
their alerts. `CreateExternalRuleDto`: `{ name (≤255), description? (≤2000),
vendorName (≤255, required, must not be "corsa"), externalRuleId? (≤255) }`.
They are created **active**, carry **no conditions or actions**, are **never
evaluated** by the Corsa engine, and are managed only through `/v1/external-rules/*`
(the normal `/v1/rules/*` mutators reject them).

---

## 7. Conditions — complete reference

### 7.1 Boolean nesting

Top-level `conditions` is `{ all?: Condition[], any?: Condition[] }`.
- `all` = AND (every child true). `any` = OR (at least one true).
- Any node may itself be a **group** (`{ label?, all?, any? }`) — nesting is
  **recursive with no depth cap**. There is **no `not`** operator/group.
- A group must contain at least one non-empty `all`/`any`; empty groups are dropped.

### 7.2 Leaf condition fields

A leaf compares one fact to a value. Fields (`ConditionDto`):

| Field | Type | Req | Meaning |
|---|---|---|---|
| `label` | string | ❌ | Human label; surfaces in evaluation results, path analysis, narratives. **Always set it.** |
| `entity` | enum | ✅ (leaf) | `transaction` \| `client` \| `wallet` \| `bankAccount`. |
| `entityRelationship` | enum | ❌ | `all` \| `sender` \| `receiver` (default `all`). Which participant's field to read, or which side an aggregation is scoped to. |
| `property` | string | ✅ (leaf) | Fact key for the entity (see §10). For aggregations, doubles as the aggregation *source* (default `"transactions"`). |
| `operator` | enum | ✅ (leaf) | One of the 11 comparison operators (§7.3). |
| `value` | any | ✅* | Static value. `in`/`notIn` → array; `between` → `[min, max]`. *Required unless `valueRef` is used on an aggregation. |
| `path` | string | ❌ | JSONPath into a nested fact value, e.g. `"$.kyc.status"`. |
| `valueRef` | object | ❌ | Dynamic threshold from a client field (aggregation only) — see §7.6. |
| `all` / `any` | Condition[] | ❌ | Nested groups. |

> `fact`, `factPath`, `thresholdRef`, `aggregation` also exist on the DTO but are
> **backend-generated during normalization** — do not set them.

### 7.3 Comparison operators (complete — 11)

`Operator` enum + AJV schema. These are the **only** allowed operators; there is
no regex/geo/sanctions operator (those are expressed via properties + `in`).

| Operator | Value shape | Meaning / semantics |
|---|---|---|
| `equal` | scalar | Exact match. |
| `notEqual` | scalar | Not equal. |
| `greaterThan` | number/date | `fact > value`; null/undefined fact → `false`. |
| `greaterThanInclusive` | number/date | `fact >= value`; null → `false`. |
| `lessThan` | number/date | `fact < value`; null → `false`. |
| `lessThanInclusive` | number/date | `fact <= value`; null → `false`. |
| `in` | array | `fact ∈ value`. |
| `notIn` | array | `fact ∉ value`. |
| `contains` | scalar/array | String/array contains value. |
| `doesNotContain` | scalar/array | Negation of `contains`. |
| `between` | `[min, max]` | `min <= fact <= max`; **date strings are compared as timestamps**; non-length-2 array → `false`. |

Notes from the custom operators (`custom-operators.ts`): comparison and `between`
operators treat a boolean fact value as a pre-computed match flag (returned
directly), and return `false` for null/undefined facts (e.g. an unresolved
threshold) instead of coercing.

**Simple condition example**

```jsonc
{ "label": "High-risk customer", "entity": "client", "property": "riskTier", "operator": "equal", "value": "HIGH" }
```

### 7.4 Aggregation / velocity conditions

Set `entity: "transaction"` and an `aggregation*` block to roll up historical
transactions and compare the aggregate to `value` (or `valueRef`). Scope the
window to a participant with `entityRelationship` (`sender` = the source client's
transactions, `receiver` = destination's, `all` = either side).

Core fields:

| Field | Type | Meaning |
|---|---|---|
| `aggregationOperator` | enum (§7.5) | How to aggregate. Default `sum`. |
| `aggregationProperty` | string | Numeric field to aggregate (e.g. `amount`, `net_amount`, `converted_amount`, `riskScore`). Ignored by count-like operators. Default `amount`. |
| `aggregationTimeType` | enum | Window type: `all_time` \| `in_the_last` \| `after` \| `before` \| `between`. |
| `aggregationTimeValue` | number \| string \| `{start,end}` | Window magnitude (`in_the_last: 24`), a timestamp (`after`/`before`), or a range (`between`). |
| `aggregationTimePeriod` | enum | Unit for `in_the_last`: `minutes` \| `hours` \| `days` \| `weeks` \| `months` \| `years` \| `calendar_weeks` \| `calendar_months` \| `calendar_years`. |
| `aggregationFilters` | array | Narrow which transactions are aggregated (§7.7). |
| `aggregationPercentile` | int 0–100 | Only for `percentile` (default 50). |
| `aggregationIncludeCurrentTransaction` | boolean | Include the triggering txn in the aggregate. Only for `sum`/`count`/`min`/`max`/`countDistinct`/`percentage` on `transactions`. |
| `operator` + `value` | — | Compares the aggregate result (e.g. `greaterThanInclusive` `50000`). |

Behavioral operators (velocity, dormancy, ratios, baselines) accept additional
fields (`aggregationWindowSize`, `aggregationBaselineDays`, `aggregationDormancyDays`,
`aggregationNumeratorDirection`, `aggregationCounterpartyField`, etc.). Discover
the full option set — and which operators hide `aggregationProperty` — at runtime
via `GET /v1/evaluation/facts` (`aggregationOperators[]`).

**Aggregation example — total withdrawals > $50k in the last 24h, this sender**

```jsonc
{
  "label": "Withdrawals > $50k in 24h",
  "entity": "transaction",
  "entityRelationship": "sender",
  "aggregationOperator": "sum",
  "aggregationProperty": "amount",
  "aggregationTimeType": "in_the_last",
  "aggregationTimeValue": 24,
  "aggregationTimePeriod": "hours",
  "aggregationFilters": [ { "property": "type", "operator": "equal", "value": "WITHDRAW" } ],
  "operator": "greaterThanInclusive",
  "value": 50000
}
```

### 7.5 Aggregation operators (complete — 27)

`AggregationOperatorType` (string values). **Count-like** operators ignore
`aggregationProperty`.

| Category | Operators (value strings) |
|---|---|
| Statistical | `sum`, `avg`, `min`, `max`, `median`, `stddev`, `percentile` |
| Counting | `count` *(count-like)*, `countDistinct`, `first`, `last`, `percentage` *(count-like)* |
| First/temporal | `isFirstTransaction` *(count-like)*, `timeSinceLastTransaction` *(count-like)* |
| Velocity / trend | `movingAverage`, `velocityChange`, `deviationFromAverage`, `baselineComparison` |
| Dormancy / behavior | `dormantReactivation` *(count-like)*, `dormantReactivationActivity` *(count-like)*, `rapidDepositWithdrawal` *(count-like)* |
| Relationship | `newCounterparty` *(count-like)*, `sharedWalletUsage` *(count-like)* |
| Pattern | `repeatingAmountCount` *(count-like)*, `consecutiveStatusCount` *(count-like)*, `minPeriodicCount` *(count-like)*, `countRoundTransactions` *(count-like)*, `directionalRatio` |

**Aggregation target-field rules** (`aggregation-validator.ts`):
- Count-like operators need no `aggregationProperty`.
- `countDistinct` target must be one of the distinct-able fields (curated set:
  `amount`, `net_amount`, `converted_amount`, `to_client_id`, `from_client_id`,
  `currency`, `from_country`, `to_country`, `to_wallet_id`, `from_wallet_id`,
  `to_bank_account_id`, `from_bank_account_id`, `payment_method`).
- All other operators require a **numeric** target: `amount`, `converted_amount`
  (`convertedAmount`), `net_amount` (`netAmount`), `risk_score` (`riskScore`).

### 7.6 Dynamic thresholds (`valueRef`)

Compare an **aggregation** against a numeric **client** field instead of a
constant. `valueRef` overrides `value`.

```jsonc
"valueRef": { "entity": "client", "role": "sender", "property": "dailyTransactionLimit", "multiplier": 0.6 }
```

- `entity` must be `"client"`. `role` must be `"sender"` or `"receiver"` (not `all`).
- `property` must be a numeric client field: `offrampDailyLimit`,
  `dailyTransactionLimit`, `annualTransactionLimit`, `riskScore`,
  `adverseMediaScore`, `pepScore`, `annualDepositEstimate`, `transactionCount`,
  `alertCount`, `caseCount`, `monthlyTransactionVolume`, `annualTransactionVolume`.
- `multiplier` (≥ 0, default 1.0) scales the field, e.g. `0.6` = 60% of the limit.
- Only valid on aggregation conditions (not plain leaves).

### 7.7 Aggregation filters

`aggregationFilters[]` narrows the aggregated set. Each item is
`{ property?, operator?, value? }` **or** `{ property?, operator?, matchSide? }`:

- `matchSide` (`"source"` | `"destination"`) resolves the comparison value from
  the *triggering* transaction's side (e.g. "same destination country as this
  txn") and is **mutually exclusive** with `value`.
- Each filter's `value` is validated/cast against its own `operator`, so
  `between`/`in` work inside filters too.

### 7.8 Nested boolean example

```jsonc
{
  "all": [
    { "label": "Large transfer", "entity": "transaction", "property": "amount", "operator": "greaterThanInclusive", "value": 10000 },
    {
      "any": [
        { "label": "Sender in high-risk country", "entity": "client", "entityRelationship": "sender", "property": "country", "operator": "in", "value": ["IR", "KP", "SY"] },
        { "label": "Receiver in high-risk country", "entity": "client", "entityRelationship": "receiver", "property": "country", "operator": "in", "value": ["IR", "KP", "SY"] }
      ]
    }
  ]
}
```

---

## 8. Actions

An action is `{ type, config }` (`ActionConfigDto`). `type` ∈ `ActionType` =
`CREATE_ALERT` | `HALT_TRANSACTION` (the only two).

### 8.1 CREATE_ALERT

Creates a compliance alert (HTTP call to compliance-entity-service). `config`
fields (all optional; the handler applies defaults):

| `config` field | Type | Default | Notes |
|---|---|---|---|
| `category` | enum | `TRANSACTION_MONITORING` | see full list below |
| `priority` | `LOW`\|`MEDIUM`\|`HIGH` | `MEDIUM` | **highest priority wins on consolidation** |
| `status` | `NEW`\|`IN_REVIEW`\|`ESCALATED`\|`RESOLVED` | `NEW` | initial alert status |
| `subCategory` | string | — | granular classification (e.g. `"Structuring"`) |
| `description` | string (≤2000) | auto-generated from rule name(s) | override the alert text |
| `assigneeId` | string | — | assign to an analyst on creation |
| `dueDateHours` | number | — | due date = alert time + N hours |

`category` values: `KYC`, `KYB`, `TRANSACTION_MONITORING`,
`ONCHAIN_TRANSACTION_MONITORING`, `SCREENING_SANCTIONS`, `SCREENING_PEP`,
`SCREENING_ADVERSE_MEDIA`, `SCREENING_REGULATORY`, `SCREENING_OTHER`, `FRAUD`,
`PERIODIC_REVIEW`, `EDD`, `OTHER`.

The handler auto-populates the rest of the alert: `associatedRuleIds` (**all**
matched rule ids), `associatedClients` (deduped from/to), `triggeringTransaction`,
`associatedTransactions`, and `source = { alertSource: TRANSACTION_MONITORING,
vendor: "Corsa", vendorData: { evaluatedAt, shouldHalt, totalMatches } }`.

> Enum values are only defaulted when **missing**. An invalid non-empty string is
> passed through to entity-service, which may reject it — use exact values above.

### 8.2 HALT_TRANSACTION

```jsonc
{ "type": "HALT_TRANSACTION", "config": {} }
```

`config` is **ignored**. The handler sets each triggering transaction's status to
**`FROZEN`** in compliance-entity-service (reason = `"Frozen by rule(s): <names>"`),
and any matched rule carrying `HALT_TRANSACTION` drives the transaction-level
decision to `FREEZE`. Typically paired with `CREATE_ALERT` — a freeze with no
alert leaves analysts blind.

### 8.3 Multi-rule consolidation

When several active rules match one transaction in a single evaluation
(`action-executor.service.ts`):
- **One consolidated `CREATE_ALERT`** is emitted, not one per rule. The config of
  the **highest-priority** matching rule is used wholesale (ties keep the first;
  missing/invalid priority is treated as `MEDIUM`). `associatedRuleIds` still lists
  **every** matched rule; `vendorData.totalMatches` = match count.
- **HALT_TRANSACTION** runs once and freezes the transaction if **any** matched
  rule carries it ("any halt → freeze").
- There is **no cross-evaluation idempotency** on alerts; dedup is handled by
  skipping duplicate evaluations within a 180s sync/async window, not by alert state.

---

## 9. Evaluation & testing

### 9.1 The three evaluation paths

| Path | Trigger | Source | Actions run? |
|---|---|---|---|
| Automatic (async) | RabbitMQ `transaction.created` event | `EVENT` | ✅ |
| Synchronous inline | `POST /v1/evaluation/evaluate-and-execute` during ingestion | `SYNC_API` | ✅ |
| On-demand (test) | `POST /v1/evaluation/evaluate` | `API` | ❌ (returns matches only) |

Sync and async are deduplicated against each other within a **180s** window so a
transaction isn't alerted twice. `TriggerSource` =
`TRANSACTION_CREATED_EVENT` | `TRANSACTION_CREATED_SYNC` | `TRANSACTION_AUDIT` |
`CLIENT_AUDIT`.

### 9.2 On-demand evaluate (safe dry-run)

```typescript
const result = await client.evaluation.evaluate({
  transactionId: "txn-123",
  transactionData: { amount: 75000, currency: "USD", type: "WITHDRAW", clientId: "client-123" }
});
result.decision;          // "ALLOW" | "FREEZE"
result.triggeredRuleIds;  // string[]
result.matches;           // RuleMatchDto[] with per-condition results & explain
result.latencyMs;
```

`POST /v1/evaluation/evaluate` returns matches + decision and persists an
evaluation record, but **does not create alerts or freeze anything**. Use it to
sanity-check a single transaction shape.

### 9.3 Decisions

`EvaluationDecision` has exactly two values: **`ALLOW`** and **`FREEZE`**. A rule's
decision is `FREEZE` iff it has a `HALT_TRANSACTION` action; otherwise `ALLOW`
(a `CREATE_ALERT`-only rule is still `ALLOW`). The transaction-level decision is
`FREEZE` if **any** matched rule is `FREEZE`, else `ALLOW`. (A response DTO lists
`UNKNOWN` in its Swagger enum, but the code never emits it.)

### 9.4 Back-testing before activation (the right way to validate)

`POST /v1/rule-tests/:ruleId` runs a rule against **historical** transactions
(from the graph) and returns analytics. It **never** creates alerts or freezes —
results are theoretical (`wouldCreateAlert`).

`CreateRuleTestDto`:

| Field | Type | Notes |
|---|---|---|
| `name?` | string | Test-run label. |
| `dateFrom?` / `dateTo?` | date | Historical window. |
| `count?` | int 1–1000 | Sample size (default 100). |
| `transactionIds?` | string[] | Explicit ids — **overrides** `dateFrom`/`dateTo`/`count`. |
| `timestampField?` | `"initiatedAt"` \| `"createdAt"` | Which timestamp to filter by (default `initiatedAt`). |
| `rule?` | `CreateRuleDto` | **Test an unsaved candidate rule** without persisting it — ideal for iterating thresholds. |

The run is async (`PENDING` → `RUNNING` → `COMPLETED`/`FAILED`; one active run per
rule at a time). `GET /v1/rule-tests/:testId` returns:
- `overview`: `alertRatio` (matched / evaluated, e.g. `0.009` = 0.9%),
  `totalAlertsTriggered`, `totalPaths`, and per-path breakdown.
- **Paths**: a *path* is a unique combination of the conditions that passed — one
  distinct route through the rule. Path analysis shows which condition
  combinations drive matches and each one's share of alerts, so you can spot an
  over-broad clause before activating.

Rule of thumb: iterate `rule?` overrides in a back-test until `alertRatio` is
sane for your volume, then create + activate.

### 9.5 Discovering facts at runtime

`GET /v1/evaluation/facts` returns the live catalog so you never guess a property:

```jsonc
{
  "facts": [ { "id": "fact_…", "name": "riskTier", "category": "client", "type": "ENUM",
               "supportedOperators": ["in","notIn"], "possibleValues": ["LOW","MEDIUM","HIGH"],
               "path": { "entity": "customer", "property": "riskTier", "participant": {…} },
               "dataLayer": { "aggregatable": true, "supportedAggregations": ["COUNT"] } } ],
  "categories": ["transaction","client","wallet","bankAccount"],
  "operatorLabels": { … },
  "aggregationOperators": [ … ], "aggregationTimeTypes": [ … ],
  "aggregationTimePeriods": [ … ], "entityRelationships": [ … ]
}
```

Use `name` as `property`, `supportedOperators` to pick an operator, and
`possibleValues` for enum inputs.

---

## 10. Entity / property catalog

Author-facing `entity` values: `transaction`, `client`, `wallet`, `bankAccount`
(internally `client` maps to `customer` — transparent to you). Properties are
**camelCase fact keys** for direct conditions; aggregation target/filter fields
use the transaction aggregation field names (mostly snake_case, e.g. `net_amount`,
`from_country`). When unsure, call `GET /v1/evaluation/facts`.

`entityRelationship`: `all` (default) | `sender` | `receiver`. Transaction
properties have no participant; client/wallet/bankAccount conditions read the
sender's, the receiver's, or either side's entity.

### transaction (`entity: "transaction"`)

| property | type | values / notes |
|---|---|---|
| `amount` | number | transaction amount |
| `convertedAmount` | number | amount in platform currency |
| `convertedCurrency` | fiat | `USD`, `EUR` |
| `netAmount` | number | |
| `currency` | string (enum-like) | e.g. `BTC,ETH,USDT,USDC,SOL,MATIC,AVAX` (free-form) |
| `type` | enum | `DEPOSIT`, `WITHDRAW`, `TRADE`, `TRANSFER` |
| `status` | enum | `SUCCESS`, `PENDING`, `CANCELLED`, `FAILED`, `FROZEN` |
| `riskScore` | number | 0–100 |
| `riskLevel` | enum | `LOW`, `MEDIUM`, `HIGH`, `UNKNOWN` |
| `paymentMethod` | string (enum-like) | e.g. `CRYPTO,CARD,BANK_TRANSFER,WIRE,ACH,SEPA` |
| `chain` | string (enum-like) | e.g. `ethereum,bitcoin,polygon,solana,base,…` |
| `fromCountry` / `toCountry` | string (enum-like) | ISO country (free-form, not enforced) |
| `referenceId`, `txHash` | string | |
| `initiatedAt` | date | |

*(Aggregation field names for the same data use snake_case: `net_amount`,
`converted_amount`, `from_country`, `to_country`, `from_client_id`, `to_client_id`,
`from_wallet_id`, `to_wallet_id`, `payment_method`, `side_in_trade`, etc.)*

### client (`entity: "client"`)

Common (individual + corporate):

| property | type | values / notes |
|---|---|---|
| `riskTier` | enum | `LOW`, `MEDIUM`, `HIGH` |
| `riskScore` | number | 0–100 |
| `type` | enum | `INDIVIDUAL`, `CORPORATE` |
| `country`, `incorporationCountry`, `jurisdictionCountry` | string (enum-like) | |
| `kycStatus` | enum | `APPROVED`, `WAITING_FOR_REVIEW`, `IN_REVIEW`, `REJECTED`, `OFF_BOARDED`, `FROZEN`, `PENDING_DOCUMENTS`, `CLOSED_BY_CLIENT`, `APPLICATION_IN_PROGRESS`, `PENDING_IDV`, `PENDING_EDD`, `PENDING_SCREENING` |
| `kycLevel` | enum | `TIER_1`, `TIER_2`, `TIER_3` |
| `activityStatus` | enum | `ACTIVE`, `NOT_ACTIVE` |
| `isPep` | boolean | |
| `isSanctioned` | boolean | |
| `isAdverseMedia` | boolean | |
| `pepScore`, `adverseMediaScore` | number | 0–100 |
| `accountAge` | number | days (not aggregatable — computed) |
| `age` | number | years (computed) |
| `dateOfBirth`, `onboardedAt`, `riskCalculatedAt` | date | |
| `alertCount`, `caseCount` | number | |
| `dailyTransactionLimit`, `offrampDailyLimit`, `annualTransactionLimit`, `annualDepositEstimate` | number | usable as `valueRef` thresholds |
| `declaredAssets`, `clientTags` | array | use `contains`/`doesNotContain` |
| `tagCount` | number | |

Corporate-only: `businessType`, `businessSubType`, `sourceOfFunds`,
`incorporationType`, `ownershipType`, `ownershipComplexity`, `adverseMediaRiskLevel`
(`NONE,LOW,MEDIUM,HIGH`), `pepTier` (`NO_PEP,TIER_4..TIER_1`),
`monthlyTransactionVolume`, `annualTransactionVolume`, `hasSubpoena`, `hasSAR`.
(For the exact `businessType`/`businessSubType`/`sourceOfFunds` value sets — ~60
sub-types — read `possibleValues` from `GET /v1/evaluation/facts`.)

Session (device/IP, under `client`): `loginIp`, `ipCountry`, `ipCountryCode`,
`geoCity`, `isVpn`, `isProxy`, `isTor`, `isDatacenter`, `deviceFingerprint`,
`deviceType`, `browser`, `os`, `sessionStartedAt`.

### wallet (`entity: "wallet"`)

| property | type | values / notes |
|---|---|---|
| `address` | string | |
| `chain` | string (enum-like) | |
| `riskLevel` | enum | `LOW`, `MEDIUM`, `HIGH`, `UNKNOWN` |
| `riskScore` | number | 0–100 |
| `associatedClientIds` | array | |
| `clientId`, `referenceId`, `riskReason` | string | |
| `riskCalculatedAt`, `createdAt` | date | |

### bankAccount (`entity: "bankAccount"`)

| property | type | values / notes |
|---|---|---|
| `bankName` | string | |
| `currency` | string (enum-like) | e.g. `USD,EUR,GBP,CAD,AUD` |
| `country` | string (enum-like) | |
| `riskLevel` | enum | `LOW`, `MEDIUM`, `HIGH`, `UNKNOWN` |
| `riskScore` | number | 0–100 |
| `balance` | number | |
| `accountNumber`, `routingNumber`, `accountHolderName` | string (sensitive) | |

**Aggregatability:** every fact is aggregatable **except** computed ones
(`accountAge`, `age`, and the internal `transaction_direction`/`source`/`destination`).
Numeric aggregation targets are limited to `amount`, `converted_amount`,
`net_amount`, `risk_score` (+ camelCase aliases). Some raw fields (e.g.
transaction `alertCount`, `hasAlerts`, `parentOperationId`) are intentionally not
exposed as conditionable facts — if a property isn't in `GET /v1/evaluation/facts`,
it isn't usable.

---

## 11. Validation rules & limits

- **Rule**: `name` required/non-empty; `conditions` must have ≥1 non-empty
  condition (`all` or `any`); `actions` array required. No rule-level `priority`/`scope`.
- **External rule**: `name`/`vendorName`/`externalRuleId` ≤ 255; `description` ≤ 2000;
  `vendorName` may not be "corsa" (reserved).
- **Alert description**: ≤ 2000 chars (auto-truncated with a `… (+N more)` suffix on consolidation).
- **Back-test `count`**: 1–1000 (default 100); one active run per rule.
- **Condition value**: required for every leaf operator; `in`/`notIn` need arrays;
  `between` needs a length-2 array; a scalar operator given a multi-element array errors.
- **Aggregation**: `percentile` 0–100; numeric target required for statistical ops;
  `countDistinct` target must be distinct-able; `includeCurrentTransaction` only on
  the supported operators; behavioral-operator/period-type combos are validated
  (e.g. `YEAR` period is rejected for `velocityChange`/`minPeriodicCount`).
- **valueRef**: `entity="client"`, `role`≠`all`, numeric client property, `multiplier`≥0.
- **Fact/operator**: the chosen `operator` must be in the fact's `supportedOperators`;
  unknown properties (facts not in the registry) are rejected at save.
- **Nesting depth**: no hard cap, but a rule that fails JRE-schema validation at load
  time is **silently skipped** by the engine — keep rules well-formed and test them.

---

## 12. Worked examples (complete `POST /v1/rules` payloads)

### 12.1 Structuring / threshold avoidance

≥ 3 cash-in deposits each just under the $10k reporting line within 24h.

```json
{
  "name": "Structuring - sub-threshold deposits",
  "description": "3+ deposits each in [9,000, 9,999] within 24h for the same sender",
  "conditions": {
    "all": [
      {
        "label": "3+ near-threshold deposits in 24h",
        "entity": "transaction",
        "entityRelationship": "sender",
        "aggregationOperator": "count",
        "aggregationTimeType": "in_the_last",
        "aggregationTimeValue": 24,
        "aggregationTimePeriod": "hours",
        "aggregationFilters": [
          { "property": "type", "operator": "equal", "value": "DEPOSIT" },
          { "property": "amount", "operator": "between", "value": [9000, 9999] }
        ],
        "operator": "greaterThanInclusive",
        "value": 3
      }
    ]
  },
  "actions": [
    { "type": "CREATE_ALERT", "config": { "category": "TRANSACTION_MONITORING", "subCategory": "Structuring", "priority": "HIGH", "status": "NEW" } }
  ]
}
```

### 12.2 Velocity — rapid withdrawals

5+ withdrawals in a rolling 1-hour window.

```json
{
  "name": "Rapid withdrawal velocity",
  "description": "5+ withdrawals in the last hour for one sender",
  "conditions": {
    "all": [
      {
        "label": "5+ withdrawals in 1h",
        "entity": "transaction",
        "entityRelationship": "sender",
        "aggregationOperator": "count",
        "aggregationTimeType": "in_the_last",
        "aggregationTimeValue": 1,
        "aggregationTimePeriod": "hours",
        "aggregationFilters": [ { "property": "type", "operator": "equal", "value": "WITHDRAW" } ],
        "operator": "greaterThanInclusive",
        "value": 5
      }
    ]
  },
  "actions": [
    { "type": "CREATE_ALERT", "config": { "category": "TRANSACTION_MONITORING", "subCategory": "Velocity", "priority": "MEDIUM" } }
  ]
}
```

### 12.3 High-risk jurisdiction + large amount (nested all/any)

Large transfer where either participant sits in a high-risk country.

```json
{
  "name": "Large transfer to/from high-risk jurisdiction",
  "description": "Transfer >= $10k with sender or receiver in IR/KP/SY",
  "conditions": {
    "all": [
      { "label": "Amount >= $10k", "entity": "transaction", "property": "amount", "operator": "greaterThanInclusive", "value": 10000 },
      {
        "any": [
          { "label": "Sender high-risk country", "entity": "client", "entityRelationship": "sender", "property": "country", "operator": "in", "value": ["IR", "KP", "SY"] },
          { "label": "Receiver high-risk country", "entity": "client", "entityRelationship": "receiver", "property": "country", "operator": "in", "value": ["IR", "KP", "SY"] }
        ]
      }
    ]
  },
  "actions": [
    { "type": "CREATE_ALERT", "config": { "category": "SCREENING_REGULATORY", "subCategory": "Geographic risk", "priority": "HIGH" } }
  ]
}
```

### 12.4 Sanctions / PEP exposure → alert **and** freeze

```json
{
  "name": "Sanctioned or PEP counterparty - freeze",
  "description": "Freeze and alert when either participant is sanctioned or a PEP",
  "conditions": {
    "any": [
      { "label": "Sender sanctioned", "entity": "client", "entityRelationship": "sender", "property": "isSanctioned", "operator": "equal", "value": true },
      { "label": "Receiver sanctioned", "entity": "client", "entityRelationship": "receiver", "property": "isSanctioned", "operator": "equal", "value": true },
      { "label": "Sender is PEP", "entity": "client", "entityRelationship": "sender", "property": "isPep", "operator": "equal", "value": true }
    ]
  },
  "actions": [
    { "type": "CREATE_ALERT", "config": { "category": "SCREENING_SANCTIONS", "priority": "HIGH", "status": "ESCALATED" } },
    { "type": "HALT_TRANSACTION", "config": {} }
  ]
}
```

### 12.5 Dynamic per-client limit (`valueRef`)

24h withdrawal total exceeds 100% of the client's own `dailyTransactionLimit`.

```json
{
  "name": "Exceeds client daily withdrawal limit",
  "description": "Sum of a sender's withdrawals in 24h exceeds their configured daily limit",
  "conditions": {
    "all": [
      {
        "label": "24h withdrawals > client daily limit",
        "entity": "transaction",
        "entityRelationship": "sender",
        "aggregationOperator": "sum",
        "aggregationProperty": "amount",
        "aggregationTimeType": "in_the_last",
        "aggregationTimeValue": 24,
        "aggregationTimePeriod": "hours",
        "aggregationFilters": [ { "property": "type", "operator": "equal", "value": "WITHDRAW" } ],
        "operator": "greaterThan",
        "valueRef": { "entity": "client", "role": "sender", "property": "dailyTransactionLimit", "multiplier": 1.0 }
      }
    ]
  },
  "actions": [
    { "type": "CREATE_ALERT", "config": { "category": "TRANSACTION_MONITORING", "subCategory": "Limit breach", "priority": "HIGH" } }
  ]
}
```

---

## 13. Best practices

1. **Author from `GET /v1/evaluation/facts`, not memory.** Property keys, enum
   values, and per-fact supported operators are all there — it's the contract.
2. **Always label conditions.** Labels drive evaluation results, path analysis,
   and analyst-facing narratives; unlabeled rules are hard to audit.
3. **Back-test before activating.** Use `POST /v1/rule-tests/:ruleId` (with a
   `rule?` override to iterate) and watch `alertRatio` + path breakdown. An
   alert ratio near 100% means the rule is too broad.
4. **Narrow aggregations with `aggregationFilters`.** Aggregating *all*
   transactions is almost never what you want — filter by `type`, `status`, etc.
5. **Scope velocity rules with `entityRelationship`.** `sender` vs `receiver`
   changes whose history you sum; `all` doubles the surface.
6. **Freeze deliberately.** Pair `HALT_TRANSACTION` with `CREATE_ALERT`, and
   remember any single halting rule freezes the whole transaction.
7. **Provide `reason`** on activate/disable/delete and on active-rule edits — it
   lands in the audit trail.
8. **Mind versioning.** Editing an active rule creates a new version atomically
   and reloads the engine; the prior version stays queryable for audit.
9. **Prefer `valueRef` over hard-coded limits** when the threshold is really a
   per-client attribute — it keeps one rule instead of one-per-tier.

---

## 14. Common pitfalls

| Pitfall | Fix |
|---|---|
| Expecting `POST /v1/evaluation/evaluate` to create alerts | It's a dry-run — only `evaluate-and-execute` and the async ingestion path run actions. |
| Setting a rule-level `priority`/`scope` | Removed. Priority lives in `CREATE_ALERT.config.priority`; there is no scope field. |
| Deleting an active rule | Disable first, then delete. |
| Forgetting to activate | New rules are `draft` and don't evaluate until activated. |
| Writing a `fact` hash by hand | Write `entity` + `property`; the normalizer builds the fact. |
| Using `converted_amount` (snake) as a **direct** `property` | Direct properties are camelCase fact keys (`convertedAmount`). Snake_case is for aggregation target/filter fields. |
| `in`/`notIn` with a scalar, or `between` without `[min,max]` | `in`/`notIn` take arrays; `between` takes exactly `[min, max]`. |
| `valueRef` on a non-aggregation leaf, or `role: "all"` | `valueRef` is aggregation-only; role must be `sender`/`receiver`; property must be a numeric client field. |
| Aggregating without filters | Add `aggregationFilters` so you sum/count the right subset. |
| Assuming rule-test routes are `/v1/rules/:id/tests` | Real routes are `/v1/rule-tests/:ruleId` and `/v1/rule-tests/:testId` (README is stale). |
| Expecting the AI "agents" surface to generate rules | It generates evaluation **narratives**, not rules — build the JSON yourself. |
| A malformed rule "just doesn't fire" | The engine silently skips rules that fail JRE-schema validation at load — back-test to confirm it actually matches. |

---

## 15. Error handling

Validation runs at create/update (`BadRequestException`, HTTP 400) with explicit
messages, e.g.:
- `"Rule name is required"`, `"Rule must have at least one condition"`
- `Condition value is required for operator "<op>"`
- `Operator "between" requires [min, max] array of length 2, received: …`
- `Operator "<op>" expects a single value, not an array. Use "in" or "notIn" …`
- `Unsupported targetProperty "<x>" for <OP> operator. Supported values: …`
- `valueRef.property "<x>" is not a supported numeric client field. Supported values: …`
- `valueRef.role "all" is ambiguous … Use "sender" or "receiver"`
- `Only active rules can be disabled` / `Active rules cannot be deleted. Disable the rule first`

Lifecycle guardrails also 400 when you try to mutate an external rule through the
normal `/v1/rules/*` endpoints (use `/v1/external-rules/*`). At **evaluation**
time, a rule that fails JRE-schema validation is skipped (logged, not thrown), so
"no alert" can mean "invalid rule" — always back-test.

---

## 16. Source map & doc links

Key files in `rule-engine-service` (read these when this doc is insufficient):

| Area | File |
|---|---|
| Create/Update DTOs, `ConditionDto`, `ActionConfigDto`, `valueRef` | `src/rule/dto/create-rule.dto.ts`, `update-rule.dto.ts` |
| Lifecycle, versioning, activate/disable/delete | `src/rule/rule.service.ts`, `rule.controller.ts` |
| Statuses & audit actions | `src/common/enums/rule.enum.ts` |
| External rules | `src/rule/external-rule.controller.ts`, `dto/create-external-rule.dto.ts` |
| Condition model, operators, aggregation enums | `src/common/interfaces/rule-condition.interface.ts` |
| JRE schema (runtime allowed operators) | `src/rule/schemas/rule.schema.ts` |
| Normalizer (entity/property → fact) | `src/rule/utils/condition-normalizer.ts`, `fact-path-builder.ts` |
| Aggregation validation | `src/rule/utils/aggregation-validator.ts`, `src/common/constants/aggregation-fields.ts` |
| Custom operators | `src/evaluation/engine/operators/custom-operators.ts` |
| Fact catalog (properties per entity) | `src/evaluation/facts/definitions/**` (properties, registry, blocklist, builders) |
| Actions & consolidation | `src/actions/**`, `src/common/enums/action.enum.ts` |
| Evaluation, decisions | `src/evaluation/evaluation.service.ts`, `dto/evaluation-result.dto.ts` |
| Sync/async orchestration | `src/triggers/trigger-orchestrator.service.ts`, `sync-evaluation.controller.ts`, `message-consumer.service.ts` |
| Rule testing & path analysis | `src/rule-testing/**` |
| AI narrative (not authoring) | `src/evaluation/agents/narrative.agent.ts`, `src/evaluation/narrative.service.ts` |

Docs:
- Internal deep-dives (MCP `internal-docs`): `rule-engine-service` →
  `architectural-patterns/json-rules-engine-as-core-evaluator`,
  `.../fact-provider-pattern`, `.../unified-sync-async-evaluation-*`,
  `.../ajv-schema-gate`, `key-modules/src-rule-`, `key-modules/src-evaluation-facts-`.
- Public API: https://docs.corsa.finance/transaction-monitoring/rules-api
- Sibling skill (SDK-only baseline): `corsa-rule-authoring`.
