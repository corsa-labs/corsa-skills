---
name: corsa-workflow-authoring
description: >
  Author Corsa no-code compliance workflows — build a workflow from a natural-language
  prompt, configure event or scheduled triggers, assemble the node tree (ACTION, RECORD,
  NOTIFY, BRANCH, AI/Copilot), wire branches and conditions, interpolate variables, and
  validate/deploy. Use when generating or editing a workflow definition, driving the
  workflow-builder MCP tools, or producing a valid workflow-definitions payload.
---

# Corsa Workflow Authoring

You are a Corsa workflow authoring specialist. Help build valid, deployable compliance
workflows for the Corsa no-code workflow engine.
Workflows react to compliance events (or run on a schedule), walk a tree of nodes, and call
Corsa microservices to screen clients, run risk assessments, create alerts/cases, notify
analysts, request documents, send emails, call webhooks, and run AI/Copilot analysis.

For the exhaustive catalogs (every trigger filter field, every node `data` field, per-handler
runtime behavior, MCP tool schemas, the normalizer alias maps, and more worked examples), see
[references/REFERENCE.md](references/REFERENCE.md).

> The service README predates the current schema in two places. **Trust this skill:** triggers
> are a single `triggerConfig` object keyed by a dot-form `event` (not a `triggerConfigs` array
> with separate `entityType`/`eventType`), and `workflow.started` is **not** a valid trigger.

## Mental model

A workflow definition is **one trigger + a parent→child tree of nodes**. It is *not* a free-form
DAG and there is *no* `edges` array — every node except the trigger carries a `parentNodeId`, and
the tree is derived from those pointers.

```
        TRIGGER  (exactly one, the root — no parentNodeId)
           │
        ┌──┴───┐          children of the same parent run in PARALLEL
     ACTION   RECORD      (siblings can't see each other's output)
        │
      BRANCH               conditional fan-out
     ┌──┴──┐
 BRANCH_PATH BRANCH_PATH   each path is a node; conditions live on the path
     │           │
   NOTIFY       AI
```

Two ways a workflow starts:

- **Event-driven** (`triggerConfig`): an audit event (e.g. `individual_client.updated`) arrives
  via RabbitMQ, is matched against active definitions, and starts one execution for that entity.
- **Scheduled** (`scheduleConfig`): a Temporal Schedule fires on an interval, queries the entity
  service for matching clients, and processes them in a batch through the same node tree.

A definition has **exactly one** of `triggerConfig` or `scheduleConfig` — never both, never neither.

## Lifecycle

```
DRAFT ──deploy──▶ ACTIVE ──deactivate──▶ INACTIVE ──archive──▶ ARCHIVED (terminal)
  ▲                  │                       │
  └── create         └── edit = new version  └── re-deploy ▶ ACTIVE
```

- **DRAFT** — created here; editable; not listening.
- **ACTIVE** — listening for its trigger (or schedule registered). Editing an ACTIVE definition is
  allowed and creates a new version snapshot.
- **INACTIVE** — paused; schedule removed; editable; can re-deploy or archive.
- **ARCHIVED** — soft-deleted, terminal, immutable.

`deploy` runs the full validation gate (graph, cycles, deploy-strict field checks, scheduled-entity
count) and, if `scheduleConfig.interval` is present, registers the Temporal Schedule.

## Two ways to build

### A. workflow-builder MCP tools — recommended for an agent

The service exposes an authenticated MCP endpoint (`POST /mcp/workflow-builder`, `Authorization:
Bearer <JWT>`) with **11 tools** and **2 introspection resources**. You drive a **server-side draft**
(revisioned, optimistically locked) through incremental mutations, each of which returns the updated
draft plus a validation result with **machine-readable repair hints** (`suggestedFix`, `allowedValues`).
This is the path designed for LLM agents — prefer it.

Tools: `create_workflow_draft`, `set_workflow_trigger`, `add_step_after_node`, `add_branch_after_node`,
`add_branch_path`, `update_node_config`, `move_node_after`, `remove_node`, `validate_workflow_draft`,
`save_workflow_definition`, `publish_workflow_definition`.

Resources (read these **first**, every session — they are the source of truth for allowed values):
- `workflow-builder://trigger-catalog` — supported events, filter fields per entity, operators, value types.
- `workflow-builder://variable-catalog` — supported `{{variable}}` keys and where each is allowed.

### B. workflow-definitions REST API — one-shot

`POST /v1/workflow-definitions` with the whole definition, then `POST /v1/workflow-definitions/:id/deploy`.
Same schema, same validator. Use when you already have a complete, correct definition and don't need the
incremental repair loop. (`tenant`/`platformId` comes from the JWT context — never put it in the body.)

Both paths produce and validate the **same** definition shape (below), so the schema knowledge in this
skill applies regardless of interface.

## The build loop (canonical recipe)

```
1. Read workflow-builder://trigger-catalog and workflow-builder://variable-catalog.
2. create_workflow_draft { name, description }                         → { draft, validation }
3. set_workflow_trigger { draftId, expectedRevision, triggerConfig }   (or scheduleConfig)
4. add_step_after_node / add_branch_after_node / add_branch_path ...    (build the tree)
5. update_node_config to fill in each node's data                       (prompt, payload, recipients…)
6. validate_workflow_draft { draftId, mode: "executable" }             → fix issues, repeat 4–6
7. publish_workflow_definition { draftId, expectedRevision }            → creates + deploys (ACTIVE)
      (or save_workflow_definition to persist as DRAFT without deploying)
```

**Revision discipline (critical):** every mutation returns `draft.revision`. Pass the latest value back
as `expectedRevision` on the next mutating call. A stale value returns a `REVISION_MISMATCH` error — re-read
the draft and retry with the current revision. Never guess it.

**Partial vs executable validation:** mutations validate in `partial` mode (structure/shape only — a
half-built tree is fine). `executable` mode (also run by save/publish) adds the deploy-grade checks: a
complete trigger, required per-node fields (AI `prompt`, ACTION `description`, RECORD CREATE `payload`),
resolvable `{{variables}}`, depth/limit and cycle checks. Always run `executable` before publish.

## Workflow definition anatomy

Top-level payload (`POST/PUT /v1/workflow-definitions`, and what the draft promotes to):

| Field | Required | Notes |
|---|---|---|
| `name` | Yes | Non-empty string. |
| `description` | No | String. |
| `triggerConfig` | one of | Event trigger. **Mutually exclusive** with `scheduleConfig`. |
| `scheduleConfig` | one of | Scheduled trigger. **Mutually exclusive** with `triggerConfig`. |
| `nodes` | Yes | Array of node objects (below). Must contain exactly one TRIGGER. |

Every **node** object:

| Field | Required | Notes |
|---|---|---|
| `id` | Yes | Unique within `nodes`. You assign it (e.g. `notify_1`). |
| `type` | Yes | `TRIGGER \| ACTION \| RECORD \| NOTIFY \| BRANCH \| BRANCH_PATH \| AI`. (`CODE` is rejected.) |
| `name` | Yes | Non-empty human label. |
| `parentNodeId` | Yes except TRIGGER | The node this runs after. TRIGGER omits it (or `null`); every other node **must** set it to an existing node id. |
| `data` | per type | Type-specific config (see Node types). |

**Graph rules (enforced):**
- Exactly **one** TRIGGER node, and it is the root (no parent).
- The graph is connected: every node must be reachable from the trigger via `parentNodeId`.
- **No cycles.** Duplicate ids rejected.
- At least one non-trigger node.
- Limits: ≤ **200** nodes, depth ≤ **50**, BRANCH fan-out ≤ **20** paths.

## Triggers

### Event trigger — `triggerConfig`

A single object: `{ event, filters[] }`. `event` is a dot-form string; `filters` is ANDed (all must
pass). Empty `filters` = fire on every event of that type. There is no OR across filters — for OR
semantics, create separate definitions.

**Events (the only valid values):** `alert.created`, `alert.updated`, `case.created`, `case.updated`,
`individual_client.created`, `individual_client.updated`, `corporate_client.created`,
`corporate_client.updated`, `transaction.created`, `transaction.updated`.

**Filter object:** `{ field, operator, value?, fromValue?, toValue? }`. `field` must be an allowed field
for that event's entity (dot-notation for nested paths, e.g. `currentRisk.level`). The allowed field
catalog per entity is large — read `workflow-builder://trigger-catalog` or REFERENCE.md.

**Operators** (each field type allows a subset — strings/enums get equality+list ops; numbers add
comparisons+`between`; dates add `relative_date`; booleans get only `equals`/`not_equals`):

| Operator | Requires | Notes |
|---|---|---|
| `equals` / `not_equals` | `value` | scalar |
| `in` / `not_in` | `value` | **non-empty array** |
| `greater_than` / `less_than` / `greater_than_or_equal` / `less_than_or_equal` | `value` | numeric/date compare |
| `between` | `fromValue`+`toValue` (or `value:[a,b]`) | inclusive range |
| `changed_from_to` | **both** `fromValue` and `toValue` | field transition (needs previous value) |
| `relative_date` | `value: { amount, unit, direction, comparison?, tolerance? }` | date fields only |

```json
{
  "triggerConfig": {
    "event": "individual_client.updated",
    "filters": [
      { "field": "currentRisk.level", "operator": "changed_from_to", "fromValue": ["LOW", "MEDIUM"], "toValue": "HIGH" }
    ]
  }
}
```

### Scheduled trigger — `scheduleConfig`

```json
{
  "scheduleConfig": {
    "interval": { "amount": 1, "unit": "days" },
    "entityType": "individual_client",
    "filters": [ { "field": "activityStatus", "operator": "equals", "value": "NOT_ACTIVE" } ]
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `interval` | Yes | `{ amount, unit }`, `unit ∈ minutes\|hours\|days\|weeks`. Resolved duration must be **≥ 5 minutes and ≤ 1 year**. |
| `entityType` | Yes | **Only `individual_client` or `corporate_client`** are queryable. (alert/case/transaction are not yet supported and return zero entities.) |
| `filters` | No | Same operators as event filters; fields must match the chosen client type. |
| `startDate` / `endDate` / `timezone` | No | Accepted; note the current scheduler applies a fixed interval and does **not** honor start/end/timezone as a calendar (see REFERENCE). |

A scheduled run caps at **5,000** entities (100/page × 50 pages).

## Node types

Each non-trigger node needs a `parentNodeId` and a `data` object shaped by its type. Below is the
authoring-critical shape of each; full field tables and runtime behavior are in REFERENCE.md.

### TRIGGER
Exactly one; the root. `data` mirrors the trigger mode for the UI: `{ "triggerMode": "EVENT" }` or
`{ "triggerMode": "SCHEDULED" }`. The canonical config is the top-level `triggerConfig`/`scheduleConfig`;
`triggerMode` must agree with which one you set.

### ACTION — `data.actionType` selects the operation
One flat `data` shape; `actionType` is required and drives which fields are needed.

| `actionType` | What it does | Key `data` fields |
|---|---|---|
| `SCREEN_CLIENT` | Trigger an integration screening sync for the client | `clientId?`, `integrationId?`, `syncType?` |
| `SCREEN_PEP_SANCTIONS` | PEP/sanctions screening | **`screeningType`** (`PERSON\|COMPANY\|ENTITY`), `threshold?`, `dataset?`, `createAlertOnMatch?`, `alertPriority?`, `alertCategory?` |
| `RUN_RISK_ASSESSMENT` | Run a risk formula | `formulaId?`, `entityType?`, `entityId?` |
| `INITIATE_DEEP_RESEARCH` | Kick off deep-research agent | `speed?` (`FAST\|SLOW`), `clientId?` |
| `DOCUMENT_REQUEST` | Create a KYC case requesting docs | **`description`** (at deploy), `caseCategory?`, `casePriority?` |
| `INFORMATION_REQUEST` | Create a case requesting info | **`description`** (at deploy) |
| `CLIENT_PERIODIC_REVIEW` | Create a periodic-review alert | **`description`** (at deploy), `reviewInterval?` (e.g. `30d`), `priority?` |
| `CREATE_INTERACTION` | Log an interaction/note case | `interactionType?`, `notes?`, `category?`, `priority?` |
| `SEND_EMAIL` | Client-facing email via messaging-service | **`templateId` OR `body`**; if `body` then **`subject`**; `recipient?`, `clientId?`, `variableOverrides?` |
| `WEBHOOK` | Call an external URL | **`url`** (HTTPS, no private/internal hosts), `method?` (`POST\|PUT\|PATCH`), `headers?`, `payloadTemplate?`, `retries?` (≤10) |

> Do **not** invent entity-mutation action types (`CREATE_ALERT`, `UPDATE_CASE`, …). Mutating an entity
> is a **RECORD** node. (The normalizer will rewrite such ACTIONs, but author them as RECORD directly.)

### RECORD — create or update a Corsa entity
`data`: `{ entityType, operation, payload, conditions?, fromNodeId?, missingAssociationBehavior? }`.

- `entityType ∈ ALERT | CASE | CLIENT | TRANSACTION` (UPPERCASE).
- `operation ∈ CREATE | UPDATE` (UPPERCASE). **CREATE is only valid for `ALERT` and `CASE`** (CLIENT and
  TRANSACTION are UPDATE-only; CASE CREATE via RECORD is additionally unsupported at runtime — use the
  `INFORMATION_REQUEST`/`DOCUMENT_REQUEST`/`CREATE_INTERACTION` actions to open cases).
- `payload` is the entity fields to set (required non-empty for CREATE at deploy). String values support
  `{{variables}}`; enum-ish values (`status`, `priority`, `category`, …) are auto-uppercased.
- `conditions` (UPDATE only): gate the update on the target entity's current state (see Conditions).

```json
{ "id": "rec_1", "type": "RECORD", "name": "Escalate risk", "parentNodeId": "trigger_1",
  "data": { "entityType": "CLIENT", "operation": "UPDATE",
            "payload": { "currentRisk": { "level": "HIGH" } } } }
```

### NOTIFY — internal notification to analysts
`data`: `{ types | type, message, subject?, recipients?, userIds?, fromNodeId? }`.

- Channels: `types: ["IN_APP" | "EMAIL" | "SLACK"]` (preferred, multi-channel) or legacy single `type`.
- `message` required (supports `{{variables}}`).
- **IN_APP** needs `userIds` (UUIDs). **EMAIL** needs `recipients` (emails) and usually `subject`.
  **SLACK** needs `recipients` (user emails).

### BRANCH / BRANCH_PATH — conditional fan-out
- A `BRANCH` node holds no conditions itself (only optional `repeatConfig`). Its children must be **≥2**
  `BRANCH_PATH` nodes (≤20).
- Each `BRANCH_PATH` has `data.conditions[]`. A path with **empty/omitted conditions is the default**
  (taken only when no other path matches). Author **at most one** default path.
- The matched path(s)' children continue execution.

```json
{ "id": "br_1", "type": "BRANCH", "name": "By risk", "parentNodeId": "ai_1", "data": {} },
{ "id": "bp_high", "type": "BRANCH_PATH", "name": "High risk", "parentNodeId": "br_1",
  "data": { "conditions": [ { "source": "previousNode", "nodeId": "ai_1", "field": "isSuspicious", "operator": "eq", "value": true } ] } },
{ "id": "bp_default", "type": "BRANCH_PATH", "name": "Otherwise", "parentNodeId": "br_1", "data": { "conditions": [] } }
```

### AI — Copilot analysis (feature-gated `ENABLE_WORKFLOW_AI_NODE`)
`data`: `{ prompt, modelId?, outputJsonSchema?, toolGroups?, maxTokens?, temperature?, systemInstructions?, contextNodeIds? }`.

- `prompt` (required at deploy) — supports `{{variables}}`. The trigger entity's data and any
  `contextNodeIds` upstream outputs are appended automatically.
- `modelId` default `claude-sonnet-4-6`. `outputJsonSchema` forces structured output (object or JSON string).
- `toolGroups` — read-only groups only: `entity_query`, `metrics`, `screening`, `web_search`.
- `maxTokens` 1–32000; `temperature` 0–1; `contextNodeIds` inject named upstream node outputs into the prompt.
- Output is the model's result spread at top level plus a `_meta` block (`taskId`, `tokenUsage`, …). The
  `_meta` is stripped when passed downstream as context.

```json
{ "id": "ai_1", "type": "AI", "name": "Assess transaction", "parentNodeId": "trigger_1",
  "data": { "modelId": "claude-sonnet-4-6",
            "prompt": "Assess the transaction for {{client.name}} for AML red flags.",
            "outputJsonSchema": { "type": "object",
              "properties": { "isSuspicious": {"type":"boolean"}, "reasoning": {"type":"string"} },
              "required": ["isSuspicious","reasoning"] },
            "toolGroups": ["entity_query"], "temperature": 0.1 } }
```

### CODE — not supported
`CODE` nodes are rejected at every ingress. Use `AI`, `ACTION`, `RECORD`, or `NOTIFY` instead.

### repeatConfig — any executable node
`data.repeatConfig = { interval, maxRepetitions? }`. `interval` is `<number><unit>` with unit
`s|m|h|d|w|M|y` (e.g. `30d`, `6h`). `maxRepetitions` 1–100. If `interval` is missing/unparseable the node
runs exactly once regardless of `maxRepetitions`.

## Variable interpolation

`{{variable}}` placeholders resolve at runtime in NOTIFY `message`/`subject`, ACTION `body`/`subject`,
AI `prompt`, and RECORD `payload` string values. Read `workflow-builder://variable-catalog` for the
authoritative list. Core variables:

| Variable | Resolves to |
|---|---|
| `{{trigger.entityType}}` / `{{trigger.eventType}}` / `{{trigger.entityId}}` | the trigger event |
| `{{client.id}}` / `{{client.name}}` / `{{client.type}}` | the resolved client |
| `{{workflow.instanceId}}` | this execution |
| `{{entity.<path>}}` | any field of the trigger entity (dot-notation, e.g. `{{entity.currentRisk.level}}`) |
| `{{node.<nodeId>.<path>}}` | a field from a prior node's output |

For entity fields you can also use the entity-typed prefix (e.g. `{{alert.status}}`, `{{transaction.txAmount}}`)
in addition to `{{entity.status}}`. In a RECORD `payload`, a value that is exactly one unresolved
`{{var}}` is **dropped** (so DTO defaults apply); mixed text is kept as-is.

## Conditions (branch & record)

Condition object: `{ field, operator, value?, source?, nodeId? }`.

- `source: "triggerEvent"` (default-ish) evaluates against the trigger entity; `source: "previousNode"`
  evaluates against a prior node's output and **requires `nodeId`** referencing an ancestor node.
- **Operators** (distinct from trigger-filter operators): `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`,
  `not_in`, `exists`. `gt/gte/lt/lte` are numeric-only. All conditions in an array are ANDed.

## Worked example — high-risk client → AI review → branch → alert + notify

Event-driven, single definition. Publishing this creates it as DRAFT and deploys to ACTIVE.

```json
{
  "name": "High-risk individual: AI review and escalate",
  "description": "When an individual client moves to HIGH risk, run an AI review; if suspicious, open an alert and notify compliance.",
  "triggerConfig": {
    "event": "individual_client.updated",
    "filters": [ { "field": "currentRisk.level", "operator": "changed_from_to", "fromValue": ["LOW","MEDIUM"], "toValue": "HIGH" } ]
  },
  "nodes": [
    { "id": "trigger_1", "type": "TRIGGER", "name": "Risk elevated to HIGH", "data": { "triggerMode": "EVENT" } },

    { "id": "ai_1", "type": "AI", "name": "AI AML review", "parentNodeId": "trigger_1",
      "data": { "modelId": "claude-sonnet-4-6",
                "prompt": "Review client {{client.name}} ({{client.id}}) now at HIGH risk. Identify AML red flags.",
                "outputJsonSchema": { "type": "object", "properties": { "isSuspicious": {"type":"boolean"}, "reasoning": {"type":"string"} }, "required": ["isSuspicious","reasoning"] },
                "toolGroups": ["entity_query"], "temperature": 0.1 } },

    { "id": "br_1", "type": "BRANCH", "name": "Suspicious?", "parentNodeId": "ai_1", "data": {} },

    { "id": "bp_yes", "type": "BRANCH_PATH", "name": "Suspicious", "parentNodeId": "br_1",
      "data": { "conditions": [ { "source": "previousNode", "nodeId": "ai_1", "field": "isSuspicious", "operator": "eq", "value": true } ] } },
    { "id": "bp_no", "type": "BRANCH_PATH", "name": "Not suspicious", "parentNodeId": "br_1", "data": { "conditions": [] } },

    { "id": "rec_alert", "type": "RECORD", "name": "Open alert", "parentNodeId": "bp_yes",
      "data": { "entityType": "ALERT", "operation": "CREATE",
                "payload": { "category": "AML", "priority": "HIGH", "description": "AI-flagged: {{node.ai_1.reasoning}}" } } },

    { "id": "notify_1", "type": "NOTIFY", "name": "Notify compliance", "parentNodeId": "rec_alert",
      "data": { "types": ["IN_APP"], "userIds": ["<analyst-uuid>"], "message": "High-risk client {{client.name}} flagged by AI review." } }
  ]
}
```

## Validation & safety limits

The **same** validator runs on create, update, and deploy. Structural checks (shape, discriminators,
graph, cycles, branch rules, webhook SSRF safety) run always. Deploy-strict checks (`enforceActivationNodeData`)
require: complete trigger; AI `prompt` + `modelId`; ACTION `description` where required; RECORD CREATE
`payload` non-empty; all `{{variables}}` resolvable against the catalog.

Hard limits to respect (full table in REFERENCE.md):

| Limit | Value |
|---|---|
| Nodes per workflow / depth / branch fan-out | 200 / 50 / 20 |
| Active workflows per platform | 20 |
| Execution throttle (per definition) | 10 / minute |
| Scheduled entities per run | 5,000 |
| Schedule interval | 5 minutes – 1 year |
| Node `repeatConfig.maxRepetitions` | 1 – 100 |

**Cycle detection (at deploy):** a workflow may not trigger on an event it also *produces*. RECORD and
certain ACTION nodes produce audit events (e.g. `RECORD ALERT/CREATE` → `alert.created`;
`CLIENT_PERIODIC_REVIEW` → `alert.created`). Deploying is rejected if a definition self-triggers, or if
it closes a trigger loop across active workflows. Keep produced-event → trigger-event chains acyclic.

## The normalizer — your leeway (and its limits)

A normalizer runs before validation on create/update and on draft mutations. It coerces common near-misses
but **never invents required data**. It will: lowercase→canonical `event` and underscore→dot
(`alert_created`→`alert.created`); alias operators (`eq`/`==`→`equals`, `gte`→`greater_than_or_equal`);
alias filter fields (`riskLevel`→`currentRisk.level`, `amount`→`amount.value`); uppercase RECORD
`entityType`/`operation` and enum payload values; lift stray RECORD top-level fields into `payload`;
rewrite an entity-mutation ACTION into a RECORD node; infer NOTIFY channel from `recipients`/`userIds`.

Don't rely on it as a crutch — author the canonical shapes. Unknown fields not covered by the normalizer
are silently **stripped** by the validation pipe (not rejected), which can make a mistake look like it
"worked" until runtime. Put entity fields inside `payload`, never at the node-data top level.

## Handling validation errors

Every failed tool call / validation returns issues shaped:
`{ code, path, message, severity, suggestedFix?, allowedValues? }`. Loop on them:

1. Read `path` to find the offending node/field and `code` for the failure class
   (`INVALID_TRIGGER_CONFIG`, `MISSING_PARENT`, `GRAPH_CYCLE`, `BRANCH_CHILD_TYPE`,
   `EXECUTABLE_VALIDATION_FAILED`, `REVISION_MISMATCH`, …).
2. If `allowedValues` is present, pick from it. If `suggestedFix` is present, apply it.
3. Re-mutate with the current `expectedRevision`, then `validate_workflow_draft` again.
4. Only `publish` once `executable` validation is clean.

## Best practices

1. **Introspect first.** Read the trigger and variable catalogs before choosing fields/operators/variables —
   they are the ground truth and prevent deploy-time rejections.
2. **Build incrementally via the MCP tools** and validate after each step; let the `suggestedFix`/`allowedValues`
   hints drive corrections instead of guessing.
3. **Narrow the trigger with filters.** An unfiltered `*.updated` trigger fires on every update and will hit
   the 10/min throttle fast. Filter to the transition you care about (often `changed_from_to`).
4. **Mutate entities with RECORD, act with ACTION, alert humans with NOTIFY.** Don't overload one for another.
5. **Prefer structured AI output.** Give AI nodes an `outputJsonSchema` and reference results downstream via
   `{{node.<id>.<field>}}` or branch conditions with `source: "previousNode"`.
6. **Remember sibling isolation.** Parallel siblings can't see each other's output — chain sequentially
   (parent→child) when a later node needs an earlier node's result.
7. **Keep chains acyclic.** Watch produced-event → trigger-event loops across your active workflows.
8. **One trigger config only.** Set `triggerConfig` **or** `scheduleConfig`, and keep the TRIGGER node's
   `triggerMode` consistent with it.

## Common pitfalls

| Mistake | Fix |
|---|---|
| Using the README's `triggerConfigs` array / `entityType`+`eventType` | Use a single `triggerConfig` with a dot-form `event` (e.g. `individual_client.updated`). |
| Triggering on `workflow.started` | Not a valid trigger event — it's rejected. Chain via produced events instead. |
| A node without `parentNodeId` (or the trigger *with* one) | Every non-TRIGGER node needs a parent; the TRIGGER must have none. |
| Expecting an `edges` array | There is none — connect nodes only via `parentNodeId`. |
| `actionType: "CREATE_ALERT"` (or any entity mutation) | Use a RECORD node (`entityType`+`operation`). |
| RECORD `CREATE` on `CLIENT`/`TRANSACTION` | Only `ALERT`/`CASE` support CREATE; clients/transactions are UPDATE-only. |
| BRANCH with conditions on it, or with <2 paths | Conditions live on BRANCH_PATH; a BRANCH needs ≥2 paths and ≤1 default. |
| Entity fields at node-data top level | Put them in `payload` — stray keys are silently stripped. |
| Reusing a stale `expectedRevision` | Always send the latest `draft.revision`; retry on `REVISION_MISMATCH`. |
| Scheduling on `alert`/`case`/`transaction` | Only `individual_client`/`corporate_client` are queryable on a schedule. |
| A `CODE` node | Unsupported — use AI/ACTION/RECORD/NOTIFY. |

## Links

- Full reference (schemas, catalogs, runtime contracts, more examples): [references/REFERENCE.md](references/REFERENCE.md)
- Corsa docs: https://docs.corsa.finance/workflows
