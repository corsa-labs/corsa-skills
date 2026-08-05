# Corsa Workflow Authoring — Full Reference

Exhaustive reference for building workflow definitions in the **workflows-service**. The
[SKILL.md](../SKILL.md) is the operating manual; this document is the catalog. Where the two differ,
this document and the live introspection resources (`workflow-builder://trigger-catalog`,
`workflow-builder://variable-catalog`) win over the service README, which is stale on triggers.

Conventions: node graph is a **parent→child tree** via `parentNodeId` (no `edges`). Multi-tenant
`platformId` always comes from the auth context, never the request body.

---

## 1. Interfaces

### 1.1 workflow-builder MCP server

- **Endpoint:** `POST /mcp/workflow-builder` (no API version prefix). Streamable-HTTP MCP transport,
  **stateless** (a fresh server per request).
- **Auth:** required — `Authorization: Bearer <JWT>`. `platformId` + `userId` are decoded from the token
  and scope every draft. There is no `platformId` tool parameter.
- **Server identity:** `workflows-service-builder` v1.0.0.

#### Resources (read-only introspection)

| Resource | URI | Content |
|---|---|---|
| `workflow_trigger_catalog` | `workflow-builder://trigger-catalog` | `{ events: [{ event, fields: [{ field, valueType, enumValues?, operators[] }] }], schedules: [{ entityType, fields[] }], allFilterOperators[] }` |
| `workflow_variable_catalog` | `workflow-builder://variable-catalog` | `{ syntax: "{{variable.key}}", variables: [{ key, aliases?, label, description, valueType, source, entityTypes?, triggerEvents?, allowedIn[], examples?, resolutionPaths? }] }` |

`allowedIn` values: `NOTIFY.message`, `NOTIFY.subject`, `ACTION.body`, `ACTION.subject`, `AI.prompt`,
`RECORD.fields`.

#### Tools (11)

Shared fragment for mutating tools: `{ draftId: uuid, expectedRevision?: int≥1 }`. All tools return
`structuredContent`; mutations return `{ draft, validation }`, save/publish add `workflowDefinition`.
`workflowNode` = `{ id, type, name, parentNodeId?, data? }`.

| Tool | Input | Effect |
|---|---|---|
| `create_workflow_draft` | `{ name, description?, workflowDefinitionId? }` | Create a persistent draft (`revision: 1`, empty `nodes`). |
| `set_workflow_trigger` | `{ …rev, triggerConfig? , scheduleConfig? }` | Set exactly one of event/schedule; upserts the mirror TRIGGER node. |
| `add_step_after_node` | `{ …rev, parentNodeId, node }` | Add an ACTION/RECORD/NOTIFY/AI node (not TRIGGER/BRANCH/BRANCH_PATH). |
| `add_branch_after_node` | `{ …rev, parentNodeId, branchNodeId?, branchName?, paths: [{nodeId?, name?, conditions?}] (≥2) }` | Add a BRANCH + its BRANCH_PATH children (must include exactly one default path). |
| `add_branch_path` | `{ …rev, branchNodeId, pathNodeId?, name?, conditions? }` | Add a BRANCH_PATH under an existing BRANCH. |
| `update_node_config` | `{ …rev, nodeId, name?, data? }` | Shallow-merge name/data; re-normalizes and re-validates. |
| `move_node_after` | `{ …rev, nodeId, parentNodeId }` | Reparent a non-trigger node. |
| `remove_node` | `{ …rev, nodeId, mode: "NODE_ONLY"\|"SUBTREE" }` | Remove a node (NODE_ONLY reparents children to grandparent) or its whole subtree. |
| `validate_workflow_draft` | `{ draftId, mode?: "partial"\|"executable" }` | Structured issues; default `partial`. |
| `save_workflow_definition` | `{ …rev, changelog? }` | Validate `executable`, then create/update the real definition as **DRAFT**. |
| `publish_workflow_definition` | `{ …rev, changelog?, reason? }` | Save **and** deploy → **ACTIVE** (registers schedule if any). |

**Draft → definition promotion:** first save assigns and persists a `workflowDefinitionId` on the draft, so
later saves update the same definition. The draft is **not** deleted after publish — it stays linked for
future edits. Publish rolls back status if schedule registration fails.

**Tool error shape:** `isError: true`, `structuredContent = { success:false, error:{ code, message, issues[] } }`.
Issue: `{ code, path, message, severity: "error"|"warning", suggestedFix?, allowedValues? }`. Error codes
include `DTO_VALIDATION_FAILED`, `VALIDATION_FAILED`, `NOT_FOUND`, `TOOL_EXECUTION_FAILED`, `REVISION_MISMATCH`.

### 1.2 workflow-definitions REST API

Base: `/v1/workflow-definitions` (no global prefix).

| Method | Path | Description |
|---|---|---|
| POST | `/v1/workflow-definitions` | Create (DRAFT). Body = definition. |
| GET | `/v1/workflow-definitions` | List (filter/paginate). |
| GET | `/v1/workflow-definitions/:id` | Get one. |
| PUT | `/v1/workflow-definitions/:id` | Update (allowed in DRAFT/INACTIVE/ACTIVE; not ARCHIVED). Body adds `changelog?`. |
| PATCH | `/v1/workflow-definitions/:id/rename` | Rename without a new version. |
| DELETE | `/v1/workflow-definitions/:id` | Archive (soft delete). |
| POST | `/v1/workflow-definitions/:id/deploy` | Validate + set ACTIVE (+register schedule). Body `{ reason? }`. |
| POST | `/v1/workflow-definitions/:id/deactivate` | ACTIVE → INACTIVE (removes schedule). |
| GET | `/v1/workflow-definitions/:id/versions` / `/versions/:version` | Version snapshots. |
| GET | `/v1/workflows` | Metadata list: name, 30-day execution count, updatedAt/By, status. |
| GET | `/v1/workflow-definitions/trigger-catalog` / `/variable-catalog` | Same catalogs as the MCP resources. |

**Query params:** `filter.status` (`DRAFT|ACTIVE|INACTIVE|ARCHIVED`), `filter.name`, `filter.id`, `page`
(default 1), `limit` (default 10, max 100), `sortBy` (e.g. `updatedAt:DESC`).

Validation pipe is `{ whitelist: true, transform: true, skipUndefinedProperties: true }` — **unknown
fields are stripped, not rejected**. No Zod on the REST DTOs; validation is class-validator decorators +
imperative service checks. The MCP tool boundary additionally uses Zod.

---

## 2. Definition schema

Top-level (`CreateWorkflowDefinitionDto` / `UpdateWorkflowDefinitionDto`):

| Field | Create | Update | Rules |
|---|---|---|---|
| `name` | required | optional | non-empty string |
| `description` | optional | optional | string |
| `triggerConfig` | one-of | optional | discriminated union on `event`; XOR `scheduleConfig` |
| `scheduleConfig` | one-of | optional | XOR `triggerConfig` |
| `nodes` | required | optional | node array (discriminated on `type`) |
| `changelog` | — | optional | string (update/save only) |

Presence rule (`validateTriggerModeConfigConsistency`): exactly one of `triggerConfig`/`scheduleConfig`;
the TRIGGER node's `data.triggerMode` must match (`EVENT`→triggerConfig, `SCHEDULED`→scheduleConfig).

Server-set fields (never send): `id`, `platformId`, `status` (default DRAFT), `version` (default 1),
`createdBy`/`updatedBy`, timestamps.

**Node base:** `{ id (unique, non-empty), type, name (non-empty), parentNodeId? (string|null), data? }`.
Each node's `type` determines the shape of its `data` field.

**Accepted `type` values:** `TRIGGER, ACTION, RECORD, NOTIFY, BRANCH, BRANCH_PATH, AI`. `CODE` exists in
the runtime enum (legacy rows + a stub handler) but is rejected at the DTO and service layers.

**Graph validation:** ≥1 node, ≤200; exactly one TRIGGER (root, no parent); no duplicate ids; all nodes
reachable from the trigger (BFS); no cycles (DFS); ≥1 non-trigger node; depth ≤50; branch fan-out ≤20.
Edges are derived from `parentNodeId`.

---

## 3. Node `data` schemas & runtime contracts

Each node handler runs with the workflow execution context and returns a status of `SUCCESS`, `FAILED`, or `SKIPPED`. On FAILED the branch stops (children don't run); on SKIPPED children still run.

### 3.1 TRIGGER
`data.triggerMode ∈ EVENT|SCHEDULED`, plus optional `filters`/`repeatConfig` (UI mirror only). Runtime just
records `{ triggerMode, triggerEvent, entityCount }` and descends. No handler.

### 3.2 ACTION
Common fields: `actionType` (required enum), `description?`, `fromNodeId?` (upstream ancestor whose
output/entity this uses; defaults to `parentNodeId`), `payload?`, `conditions?`, `repeatConfig?`,
`missingAssociationBehavior?` (`FAIL|SKIP`). Per-actionType specifics and runtime:

| actionType | Required (deploy) | Other `data` | Output keys |
|---|---|---|---|
| `SCREEN_CLIENT` | — | `clientId?`, `integrationId?` (default `chainalysis`), `syncType?` (default `partial`) | `clientId, integrationId, syncType, integrationSyncTriggered, syncResult` |
| `SCREEN_PEP_SANCTIONS` | `screeningType` (`PERSON\|COMPANY\|ENTITY`) | `threshold?` (0–1), `dataset?`, `createAlertOnMatch?`, `alertPriority?`, `alertCategory?` | `screeningType, totalResults, matchCount, results[], alertCreated?, alertId?` (batch 10) |
| `RUN_RISK_ASSESSMENT` | — | `formulaId?`, `entityType?`, `entityId?`, `entityData?` | `entityType, entityId?, formulaId, assessmentResult` |
| `INITIATE_DEEP_RESEARCH` | — | `speed?` (`FAST\|SLOW`), `clientId?` | `clientId, clientType, speed, jobId, jobStatus` (batch 10) |
| `DOCUMENT_REQUEST` | `description` | `clientId?`, `caseCategory?` (default KYC), `casePriority?` (default MEDIUM), `reviewersIds?` | `clientId, caseCreated, caseResponse` |
| `INFORMATION_REQUEST` | `description` | same as DOCUMENT_REQUEST | same |
| `CLIENT_PERIODIC_REVIEW` | `description` | `clientId?`, `clientType?`, `reviewInterval?` (default `30d`), `priority?`, `assigneeId?` | `clientId, reviewInterval, dueDate, alertId, alertCreated` |
| `CREATE_INTERACTION` | — | `interactionType?` (default NOTE), `notes?`, `category?`, `priority?`, `assigneeId?`, `dueDays?`/`dueDate?` | `clientId, interactionType, caseId, caseCreated` |
| `SEND_EMAIL` | `templateId` OR `body` (+`subject` if `body`) | `recipient?` (`"client"` or email), `clientId?`, `variableOverrides?`, `conditions?` | `templateId?, clientId, recipient, subject, messageId, threadId` |
| `WEBHOOK` | `url` (HTTPS, no private/internal host) | `method?` (`POST\|PUT\|PATCH`, default POST), `headers?`, `payloadTemplate?`, `retries?` (≤10), `includeEventData?` (default true), `includePreviousResults?` (default false) | `url, method, statusCode` |

Allowed source-entity restrictions (`ACTION_REQUIRED_SOURCE_ENTITIES`): e.g. `DOCUMENT_REQUEST`/
`CLIENT_PERIODIC_REVIEW` accept only `individual_client`/`corporate_client`; `WEBHOOK` accepts any.
Entity-mutation actionTypes (`CREATE_ALERT`, `UPDATE_CASE`, …) are invalid — the `@IsEnum` rejects them and
the normalizer rewrites them to RECORD.

**Client resolution chain** (used by client-scoped actions and RECORD): `data.clientId` → trigger `entityId`
if the trigger is a client → entity `associatedClients`/`clients`/`clientsIds` → transaction `from.client`/
`to.client` → `node.data.clientId`. `fetchClientWithFallback` tries the preferred client type then the other
on 404.

### 3.3 RECORD (`WorkflowRecordNodeDataDto`)
`{ entityType, operation, payload?, fromNodeId?, conditions?, repeatConfig?, missingAssociationBehavior? }`.

- `entityType ∈ ALERT|CASE|CLIENT|TRANSACTION` (UPPERCASE). `operation ∈ CREATE|UPDATE`. Runtime also
  understands `ESCALATE` and `UPDATE_STATUS`. **CREATE forbidden for CLIENT & TRANSACTION.** CASE CREATE via
  RECORD is unsupported at runtime (use INFORMATION_REQUEST/DOCUMENT_REQUEST/CREATE_INTERACTION actions).
- `payload` (required non-empty for CREATE at deploy): entity fields. String values interpolated; enum
  keys (`status, priority, category, accountStatus, activityStatus`) auto-uppercased unless they contain a
  `{{template}}`. `dueDays` (relative) takes precedence over `dueDate`.
- Notable behavior: ALERT UPDATE re-assigns the alert to the executing user then applies payload;
  `status: "ESCALATED"` auto-routes to escalate-alert-to-case; alerts/cases created by workflows are stamped
  with source `vendor: "Corsa Workflows"` and a link back to the definition.
- `conditions` (UPDATE only): fetch the target entity's current state, evaluate (see §6); if not met →
  `{ success:true, skipped:true, reason:"Conditions not met" }`.
- Bulk (scheduled): groups identical interpolated payloads and bulk-updates in chunks of 100; output
  `{ entityType, operation, bulk:true, totalProcessed, succeeded, failed, errors?, createdEntityIds? }`.
- Single output: `{ entityType, operation, entityId, response }`.

(There is a separate `WorkflowRecordEventType` enum — `CLIENT_UPDATED`, `ALERT_CREATED`, … — used elsewhere;
RECORD nodes use `entityType`+`operation`, not that enum.)

### 3.4 NOTIFY
`{ types?: [IN_APP|EMAIL|SLACK] | type?, message, subject?, recipients?, userIds?, fromNodeId?, repeatConfig? }`.
`message` required (interpolated). Per channel:

| Channel | Requires | Notes |
|---|---|---|
| `IN_APP` | `userIds[]` (UUIDs) | Auto-links to the trigger entity's page when possible |
| `EMAIL` | `recipients[]` (emails) | `subject` interpolated (default "Workflow Notification"); appends an HTML summary of prior node outputs |
| `SLACK` | `recipients[]` (user emails) | Sends via configured Slack integration |

Multi-channel output: `{ notificationTypes, results:[…perChannel] }`. A channel fails only if **all** its
recipients fail. If the source node (`fromNodeId`/`parentNodeId`) was SKIPPED, NOTIFY returns SKIPPED.
Distinct from ACTION `SEND_EMAIL` (which sends **client-facing** templated email); NOTIFY EMAIL is an
**internal analyst** notification.

### 3.5 BRANCH / BRANCH_PATH
- BRANCH `data`: only `repeatConfig?`. Children must all be BRANCH_PATH; **2–20** of them; **≤1 default**
  (empty-conditions) path.
- BRANCH_PATH `data`: `{ conditions?: [...], repeatConfig? }`. Parent must be a BRANCH. Empty/omitted
  conditions ⇒ default path (taken only if no explicit path matches).
- Runtime: single-entity mode evaluates each path's conditions (AND) and runs matched paths' children in
  parallel; bulk/scheduled mode partitions the entity batch per path.

### 3.6 AI
`{ modelId?, prompt?, outputJsonSchema?, toolGroups?, maxTokens?, temperature?, systemInstructions?,
contextNodeIds?, context?, repeatConfig? }`.

- Deploy requires `prompt` and `modelId`. Default model `claude-sonnet-4-6`; supported ids include
  `claude-sonnet-4-5`, `claude-sonnet-4-6`, `claude-opus-4-6`, `gpt-5-2`, `gpt-5-5`, `gpt-5-5-mini`.
- `outputJsonSchema`: object or JSON string; when present the model is forced to structured output.
- `toolGroups` (read-only only): `entity_query`, `metrics`, `screening`, `web_search`.
- `maxTokens` 1–32000; `temperature` 0–1; `systemInstructions` appended to (not replacing) the default
  workflow AI instructions (which enforce clean machine-consumable text); `contextNodeIds` inject named
  upstream successful outputs.

**Execution — async, idempotent.** The AI node runs asynchronously and polls until the analysis completes or fails. Temporal retries are safe because idempotency is keyed on `${instanceId}:${nodeId}:${repetition}[:entity:${entityId}]` — stable across retries; unique per repetition and per bulk entity.
- **Prompt enrichment:** (1) `{{variable}}` interpolation; (2) append `--- TRIGGER ENTITY DATA ---` block;
  (3) if `contextNodeIds`, append `--- CONTEXT FROM PREVIOUS STEPS ---` (SUCCESS outputs, `_meta` stripped).
- **Output:** the model result spread at top level + `_meta: { nodeType:"AI", taskId, traceId, durationMs,
  tokenUsage:{input,output}, toolCalls }`. `_meta` is stripped when fed downstream.
- **Bulk:** cap **100 entities** per execution (else error), concurrency 5.
- **Temporal:** dedicated proxy — `startToCloseTimeout` 10 min, `heartbeatTimeout` 45s, max 2 attempts,
  backoff 5s×2. Non-retryable error types: `AI_SCHEMA_VALIDATION_ERROR`, `AI_PROMPT_TOO_LONG`.

### 3.7 CODE
Rejected at the API. Runtime stub returns `{ nodeType:"CODE", stub:true }` for legacy rows only.

### 3.8 repeatConfig (`WorkflowNodeRepeatConfig`) — ACTION/RECORD/NOTIFY/AI
`{ interval: string (required), maxRepetitions?: 1–100 }`. Interval grammar `^(\d+)(s|m|h|d|w|M|y)$`
(`M`≈30d, `y`≈365d). `totalExecutions = interval ? clamp(maxRepetitions ?? 1, 1, 100) : 1` — a missing/
unparseable interval means it runs exactly once. Each repetition sleeps `interval` first (for reps > 0),
sets `context.repetition`, and (for AI) varies the idempotency key. Only the **final** repetition's output
flows downstream.

### 3.9 node-dispatcher routing
`type` → handler: `NOTIFY`→Notify, `RECORD`→Record, `AI`→AI, `CODE`→stub. `ACTION` → by uppercased
`actionType` (§3.2). `TRIGGER`/`BRANCH`/`BRANCH_PATH` are handled in the workflow walk (no handler). Unknown
type or actionType → FAILED with a descriptive error.

---

## 4. Trigger event catalog & filter fields

### 4.1 Events (dot-form, the only valid `event` values)
`alert.created`, `alert.updated`, `case.created`, `case.updated`, `individual_client.created`,
`individual_client.updated`, `corporate_client.created`, `corporate_client.updated`, `transaction.created`,
`transaction.updated`. (`workflow.started` is **not** a valid trigger.)

### 4.2 Filter object & operators
`{ field, operator, value?, fromValue?, toValue? }`. All 11 operators:
`equals, not_equals, in, not_in, changed_from_to, greater_than, less_than, greater_than_or_equal,
less_than_or_equal, between, relative_date`.

Per field-type allow-lists:
- **string/enum:** `equals, not_equals, in, not_in, changed_from_to`
- **number:** the above (minus/plus) + `greater_than, less_than, greater_than_or_equal,
  less_than_or_equal, between` + `changed_from_to`
- **date:** `equals, not_equals, greater_than, less_than, greater_than_or_equal, less_than_or_equal,
  between, relative_date, changed_from_to`
- **boolean:** `equals, not_equals` only

Value shapes: `in`/`not_in` → non-empty array; `between` → `fromValue`+`toValue` or `value:{from,to}` or
`value:[a,b]`; `changed_from_to` → **both** `fromValue` and `toValue` at deploy (each scalar or array);
`relative_date` → `{ amount:number, unit:"s"|"m"|"h"|"d"|"w"|"M"|"y", direction:"past"|"future",
comparison?:"eq"|"gt"|"gte"|"lt"|"lte" (default eq), tolerance?:string (default "1d") }`.

Filters within a config are **ANDed**; empty `filters` = match every event of that type. There is no OR
across filters (the multi-config array was collapsed away).

### 4.3 Allowed filter fields per entity
(Authoritative source: `workflow-builder://trigger-catalog`. Nested via dot-notation.)

**ALERT:** `status, priority, category, truePositiveStatus, dueDate, source.alertSource, source.vendor,
hasAssociatedClient, hasAssociatedTransaction, description, subCategory, assigneeId, raisedAt, closedDate,
researchRiskLevel, associatedRuleIds`.

**CASE:** `status, priority, category, truePositiveStatus, source, dueDate, hasAssociatedClient,
hasAssociatedTransaction, description, subCategory, assigneeId, closedDate`.

**Shared client** (individual + corporate): `accountStatus, activityStatus, currentRisk.level,
currentRisk.score, application.onboardedAt, application.lastReviewedAt, application.submittedAt,
application.nextPeriodicReview, contact.emailAddress, sanctions.isSanctioned,
politicalExposure.isPoliticallyExposed, adverseMedia.isAdverseMedia, sanctionsStatus, pepStatus,
adverseMediaStatus`.

**Individual client adds:** `address.country, general.dateOfBirth, general.gender, general.jurisdiction,
general.citizenship, work.occupation, application.kycTier, financial.monthlyNetIncome,
financial.expectedMonthlyTransactionVolume, financial.annualDepositEstimate`.

**Corporate client adds:** `address.registrationAddress.country, general.legalEntityName,
general.dateOfIncorporation, general.countryOfIncorporation, business.industry, business.businessType,
business.businessSubType, business.ownershipType, business.ownershipComplexity, business.incorporationType,
sourceOfFundsInfo.sourceOfFunds, sourceOfFundsInfo.monthlyNetIncome,
sourceOfFundsInfo.monthlyTransactionVolume, sourceOfFundsInfo.annualTransactionVolume, screening.hasSubpoena,
screening.hasSAR, adverseMedia.adverseMediaRiskLevel, adverseMedia.adverseMediaResolved,
politicalExposure.pepTier`.

**TRANSACTION:** `type, status, amount.value, amount.currency, txAmount, txCurrency, convertedAmount,
convertedCurrency, currentRisk.level, currentRisk.score, paymentMethod, paymentRail, transferType,
initiatedAt`.

Enum-valued fields: `truePositiveStatus ∈ TRUE_POSITIVE, FALSE_POSITIVE, INCONCLUSIVE, UNKNOWN`;
`application.kycTier ∈ TIER_1, TIER_2, TIER_3`; `sanctionsStatus`/`pepStatus`/`adverseMediaStatus ∈
NOT_SCREENED, CLEAR, IN_REVIEW, MATCH_CONFIRMED`.

Virtual/computed fields available to the matcher: `hasAssociatedClient`, `hasAssociatedTransaction`
(boolean presence), `associatedRuleIds` (from `associatedRules[].ruleId`), `truePositiveStatus`.

---

## 5. Schedule config

`WorkflowScheduleConfigInputDto`:

| Field | Required | Rules |
|---|---|---|
| `interval` | Yes | `{ amount:int≥1 (≤524160), unit: minutes\|hours\|days\|weeks }`; resolved duration ∈ **[5 min, 1 year]**. |
| `entityType` | Yes | `individual_client` or `corporate_client` only (schedulable). alert/case/transaction not yet queryable. |
| `filters` | No | Same operators as event filters; fields validated against the chosen client type at deploy. |
| `startDate` / `endDate` | No | ISO 8601; `startDate < endDate`. |
| `timezone` | No | IANA (e.g. `America/New_York`). |

**Temporal mapping caveat:** the scheduler registers `scheduleId = sched-<definitionId>` with a fixed
`intervals:[{ every: <ms> }]` spec — it is **not** a cron/calendar. `startDate`/`endDate`/`timezone` are
captured on the DTO but **not currently applied** to the Temporal schedule. So "every day at 09:00 in
America/New_York" is not achievable today; it fires every N ms from registration. Registration is
delete-then-create; removed on deactivate/archive.

Runtime: one `workflow_instance` (parent) per tick; entities paginated 100/page, ≤50 pages, hard cap 5,000
(exceeding → non-retryable `ENTITY_LIMIT_EXCEEDED`). Each page runs `runWorkflow` as a child with the entity
batch. Deploy-time best-effort pre-check rejects if matching entities exceed 5,000 (skipped if entity
service is down).

---

## 6. Interpolation & conditions

### 6.1 `{{variable}}` catalog
Pattern `{{[\w.-]+}}` (dashes allowed so UUID node ids work). Allowed in NOTIFY `message`/`subject`, ACTION
`body`/`subject`, AI `prompt`, RECORD `payload` strings.

Static: `{{trigger.entityType}}`, `{{trigger.eventType}}`, `{{trigger.entityId}}`, `{{client.id}}`,
`{{client.name}}` (from `overview_name`→`firstName+lastName`→`fullName`→`companyName`→`legalEntityName`),
`{{client.type}}`, `{{workflow.instanceId}}`.

Dynamic entity fields: `{{entity.<path>}}` resolves case-insensitively against the trigger entity (dot
paths). Entity-typed prefixes also work depending on trigger type: `{{alert.*}}`, `{{case.*}}`,
`{{transaction.*}}`, `{{client.*}}`/`{{individual_client.*}}`/`{{corporate_client.*}}`. Objects are
JSON-stringified; null/undefined → left unresolved.

Prior-node outputs: `{{node.<nodeId>.<dotPath>}}` resolves against that node's `output` (case-insensitive
nodeId).

**Pure-variable drop (RECORD payloads):** a value that is *exactly* one `{{var}}` that fails to resolve is
**omitted** from the payload (so the DTO default applies, e.g. `category`→`OTHER`). Mixed text like
`"Flagged: {{x}}"` is always kept even if partially unresolved.

Example resolution catalog (see changelog-era field names): Alert/Case `{{entity.status}}`,
`{{entity.priority}}`, `{{entity.category}}`, `{{entity.dueDate}}`, `{{entity.assigneeId}}`; Transaction
`{{entity.txAmount}}`, `{{entity.txCurrency}}`, `{{entity.txHash}}`, `{{entity.convertedAmount}}`,
`{{entity.type}}`; Client `{{entity.overview_name}}`, `{{entity.activityStatus}}`, `{{entity.firstName}}`;
nested `{{entity.currentRisk.level}}`, `{{entity.currentStatus.type}}`.

### 6.2 Conditions (branch & record)
Object: `{ field, operator, value?, source?, nodeId? }`.
- `source: "triggerEvent"` → evaluate against the trigger entity; `source: "previousNode"` → against a prior
  node's output and **requires `nodeId`** (must be an ancestor). RECORD conditions fetch the target entity's
  current state instead.
- Operators (`getNestedValue` + `evaluateOperator`): `eq`, `neq`, `gt`, `gte`, `lt`, `lte` (numeric-only),
  `in`, `not_in` (`value` is an array), `exists`. Array-valued fields get set/membership semantics. All
  conditions in a path/record are ANDed.

---

## 7. Event matching, dedup & throttle (runtime background)

- The consumer loads all **ACTIVE** definitions for the platform and matches in memory: incoming event =
  `joinTriggerEvent(entityType, eventType)`; a definition matches iff `triggerConfig.event ===` that string
  and all filters pass.
- **Operation events** (`deposit/withdraw/withdrawal/trade .created/.updated`) are decomposed into synthetic
  `transaction.created/updated` events per embedded transaction, then matched as transactions.
- **Entity-level dedup:** Redis `SET NX EX` on key `wf:dedup:<definitionId>:<entityId>`, TTL **30s**,
  cross-pod; fail-open on Redis error. Prevents double-fires from overlapping events on the same entity.
- **Throttle:** in-memory sliding window, **10 executions / 60s per definition per pod** (effective cap
  scales with replicas). Applies to event-triggered and manual executions. Scheduled ticks are not throttled.
- Each execution emits a `workflow.started` audit event (definition id, instance id, trigger event) for audit
  trails and downstream chaining (subject to cycle detection).

---

## 8. Cycle detection (deploy-time)

A definition is rejected at deploy if it triggers on an event it also produces (self-cycle) or if it closes a
loop across active workflows (DFS 3-color over produced-event → trigger-event edges). Only RECORD and some
ACTION nodes "produce" events (static inference from `data`):

| Node | Condition | Produces |
|---|---|---|
| RECORD | `ALERT` + `CREATE` | `alert.created` |
| RECORD | `ALERT` + `UPDATE` | `alert.updated` |
| RECORD | `ALERT` + `ESCALATE` | `alert.updated` **and** `case.created` |
| RECORD | `CASE` + `CREATE`/`UPDATE` | `case.created` / `case.updated` |
| RECORD | `CLIENT` + `UPDATE` | `individual_client.updated` **and** `corporate_client.updated` |
| RECORD | `TRANSACTION` + `UPDATE`/`UPDATE_STATUS` | `transaction.updated` |
| ACTION | `CLIENT_PERIODIC_REVIEW` | `alert.created` |
| ACTION | `CREATE_INTERACTION` / `INFORMATION_REQUEST` / `DOCUMENT_REQUEST` | `case.created` |
| ACTION | `INITIATE_DEEP_RESEARCH` | `alert.created` |
| ACTION | `SCREEN_PEP_SANCTIONS` (only if `createAlertOnMatch: true`) | `alert.created` |

`RUN_RISK_ASSESSMENT`, `SCREEN_CLIENT`, `WEBHOOK`, `SEND_EMAIL`, `NOTIFY`, `AI`, `BRANCH*`, `TRIGGER`
produce nothing for cycle purposes. Example self-cycle to avoid: a workflow triggered on `alert.created`
that contains a `RECORD ALERT/CREATE` node.

---

## 9. Validation matrix

| Check | partial (draft mutation) | executable (validate/save) | deploy |
|---|---|---|---|
| trigger/schedule mutual exclusivity | ✓ | ✓ | ✓ |
| trigger/schedule catalog validity (field/operator/value) | ✓ | ✓ | ✓ |
| node DTO shape per type; CODE rejected | ✓ | ✓ | ✓ |
| node count ≤200; unique ids; ≤1 TRIGGER | ✓ | ✓ | ✓ |
| parent rules (trigger no parent; others have existing parent) | ✓ | ✓ | ✓ |
| reachability + no cycles (within definition) | ✓ | ✓ | ✓ |
| branch shape (≥2 paths, ≤1 default, path parent is BRANCH) | ✓ | ✓ | ✓ |
| webhook SSRF safety (HTTPS, no private hosts) | ✓ | ✓ | ✓ |
| trigger present & complete | | ✓ | ✓ |
| required per-node fields (AI prompt/modelId, ACTION description, RECORD CREATE payload) | | ✓ | ✓ |
| `{{variable}}` references resolve against catalog | | ✓ | ✓ |
| interval bounds, max depth (50) | | ✓ | ✓ |
| cross-workflow cycle detection | | | ✓ |
| active-workflows-per-platform ≤20 | | | ✓ |
| scheduled entity count ≤5,000 (best-effort) | | | ✓ |
| schedule registration (Temporal) | | | ✓ |

Deploy requires status DRAFT or INACTIVE.

---

## 10. Normalizer (`normalizeWorkflowInput`)

Runs before validation on create/update and on every draft mutation. **Coerces near-misses; never invents
required data.** Reference of what it fixes:

- **Trigger event:** lowercase + underscore→dot when the result is a known event (`alert_created` →
  `alert.created`).
- **Filter operators:** `eq`/`==`/`equal`→`equals`; `gte`/`>=`→`greater_than_or_equal`; `gt`/`>`→
  `greater_than` (and lt variants); `notin`→`not_in`; `changedfromto`→`changed_from_to`; `relativedate`→
  `relative_date`.
- **Filter fields:** `riskLevel`→`currentRisk.level`, `riskScore`→`currentRisk.score`, `amount`→
  `amount.value`, `currency`→`amount.currency`, `vendor`→`source.vendor`, `alertSource`→`source.alertSource`,
  `onboardedAt`→`application.onboardedAt`, and similar.
- **RECORD:** uppercase `entityType`/`operation`; `sourceEntity`→`fromNodeId` (`"TRIGGER"`→`"trigger"`);
  lift stray top-level fields into `payload` (anything outside `{entityType, operation, fromNodeId, conditions,
  payload, repeatConfig, missingAssociationBehavior}`); uppercase enum payload keys; convert relative
  `dueDate:{amount,unit,type:"relative"}` → `{ days:N }`.
- **ACTION→RECORD coercion:** `actionType` matching `^(CREATE|UPDATE)_(ALERT|CASE|CLIENT|TRANSACTION)$` is
  rewritten to a RECORD node with derived `entityType`/`operation`. Also uppercases `actionType`/`method`.
- **NOTIFY:** uppercase `type`; infer channel — `recipients`→`EMAIL`, `userIds`→`IN_APP`.

Not covered by the normalizer + not whitelisted ⇒ silently **stripped** by the validation pipe. Author the
canonical shapes; don't depend on coercion.

---

## 11. Consolidated safety limits

| Limit | Value | Scope |
|---|---|---|
| Nodes per workflow | 200 | definition |
| Tree depth | 50 | definition |
| Branch fan-out (paths per BRANCH) | 20 | definition |
| Active workflows | 20 | per platform |
| Execution throttle | 10 / 60s | per definition per pod |
| Entity dedup TTL | 30s | per (definition, entity) |
| Scheduled entities per run | 5,000 (100/page × 50 pages) | per tick |
| Schedule interval | 5 min – 1 year | schedule |
| `repeatConfig.maxRepetitions` | 1 – 100 | node |
| AI bulk entities per execution | 100 (concurrency 5) | AI node |
| Webhook retries | ≤ 10 | webhook node |
| Standard activity timeout / attempts | 1 min / 3 | non-AI node |
| AI activity timeout / heartbeat / attempts | 10 min / 45s / 2 | AI node |

---

## 12. Versioning & instances

**Versioning:** a mutable head row (`workflow_definition`) holds current name/status/version; append-only
`workflow_definition_version` rows snapshot `triggerConfig`/`nodes`/`scheduleConfig`. Create sets v1; update
increments the version via optimistic lock (`WHERE version = :expected`) and snapshots; deploy snapshots
without incrementing (records the reason). Execution always uses the **head's** current config, not a
historical snapshot; `workflow_instance.workflowDefinitionVersion` is a provenance label only.

**Instances API:**

| Method | Path | Description |
|---|---|---|
| POST | `/v1/workflow-instances/:definitionId/execute` | Manually execute (rate-limited; re-validates). |
| GET | `/v1/workflow-instances` | List (filter by `definitionId`, `status`); hides scheduled child instances. |
| GET | `/v1/workflow-instances/:id` | Detail (aggregates child node instances for scheduled runs). |
| POST | `/v1/workflow-instances/:id/cancel` | Best-effort Temporal cancel; marks CANCELLED. |

Instance status: `PENDING, RUNNING, COMPLETED, FAILED, CANCELLED, TIMED_OUT`. Per-node status: `PENDING,
RUNNING, COMPLETED, FAILED, SKIPPED, CANCELLED`. On any node FAILED the run finalizes FAILED and pending
nodes → CANCELLED; otherwise pending → SKIPPED and run COMPLETED.

---

## 13. More worked examples

### 13.1 Scheduled dormant-account periodic review (corporate)

```json
{
  "name": "Quarterly review of dormant corporates",
  "description": "Every 90 days, open a periodic-review alert for corporate clients that are not active.",
  "scheduleConfig": {
    "interval": { "amount": 90, "unit": "days" },
    "entityType": "corporate_client",
    "filters": [ { "field": "activityStatus", "operator": "equals", "value": "NOT_ACTIVE" } ]
  },
  "nodes": [
    { "id": "trigger_1", "type": "TRIGGER", "name": "Scheduled: Corporate Client", "data": { "triggerMode": "SCHEDULED" } },
    { "id": "act_review", "type": "ACTION", "name": "Open periodic review", "parentNodeId": "trigger_1",
      "data": { "actionType": "CLIENT_PERIODIC_REVIEW", "description": "Quarterly review for {{client.name}}", "reviewInterval": "90d", "priority": "MEDIUM" } }
  ]
}
```

### 13.2 New-client sanctions screening → conditional alert + email

```json
{
  "name": "Screen new individuals for sanctions",
  "description": "On individual client creation, run PEP/sanctions screening; if matched, alert and email the client.",
  "triggerConfig": { "event": "individual_client.created", "filters": [] },
  "nodes": [
    { "id": "trigger_1", "type": "TRIGGER", "name": "New individual client", "data": { "triggerMode": "EVENT" } },
    { "id": "act_screen", "type": "ACTION", "name": "PEP/sanctions screen", "parentNodeId": "trigger_1",
      "data": { "actionType": "SCREEN_PEP_SANCTIONS", "screeningType": "PERSON", "threshold": 0.85,
                "createAlertOnMatch": true, "alertPriority": "HIGH", "alertCategory": "SANCTIONS" } },
    { "id": "br_1", "type": "BRANCH", "name": "Any matches?", "parentNodeId": "act_screen", "data": {} },
    { "id": "bp_match", "type": "BRANCH_PATH", "name": "Matched", "parentNodeId": "br_1",
      "data": { "conditions": [ { "source": "previousNode", "nodeId": "act_screen", "field": "matchCount", "operator": "gt", "value": 0 } ] } },
    { "id": "bp_clear", "type": "BRANCH_PATH", "name": "Clear", "parentNodeId": "br_1", "data": { "conditions": [] } },
    { "id": "notify_match", "type": "NOTIFY", "name": "Notify MLRO", "parentNodeId": "bp_match",
      "data": { "types": ["IN_APP","EMAIL"], "userIds": ["<mlro-uuid>"], "recipients": ["mlro@example.com"],
                "subject": "Sanctions match: {{client.name}}", "message": "{{node.act_screen.matchCount}} potential match(es) for {{client.name}}." } }
  ]
}
```

