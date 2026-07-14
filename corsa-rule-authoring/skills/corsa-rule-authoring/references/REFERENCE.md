# Corsa Rule Authoring — Full Reference

Complete SDK method signatures, condition schemas, action configs, and evaluation response formats for the Corsa rule engine.

## SDK Client Setup

```typescript
import { CorsaClient } from '@corsa-labs/sdk';

const corsa = new CorsaClient({
  BASE: "https://api.corsa.finance",
  HEADERS: {
    Authorization: `Bearer ${process.env.CORSA_API_TOKEN}:${process.env.CORSA_API_SECRET}`
  }
});
```

EU region: use `https://api.eu.corsa.finance`.

---

## Rules Service

Access: `corsa.rules`

| Method | Description |
|--------|-------------|
| `createRule(requestBody)` | Create a rule (starts as draft) |
| `listRules(page?, limit?, sortBy?, filter?)` | List rules with pagination and filtering |
| `getRule(id, version?)` | Get a rule by ID (optional specific version) |
| `updateRule(id, requestBody)` | Update a draft/disabled rule in place, or version an active rule |
| `activateRule(id, requestBody?)` | Activate a draft or disabled rule |
| `disableRule(id, requestBody?)` | Disable an active rule |
| `deleteRule(id, reason?)` | Soft-delete a draft or disabled rule |

### Rule Lifecycle

```
createRule()    → Draft
activateRule()  → Active   (evaluates every ingested transaction automatically)
disableRule()   → Disabled (no longer evaluates)
deleteRule()    → Soft-deleted (must be Draft or Disabled first)
```

Updating an **active** rule creates a new **version** atomically — the old version is disabled, the new version goes active. Updating a draft or disabled rule modifies in place.

### Create a Rule

```typescript
const rule = await corsa.rules.createRule({
  name: "Large withdrawal from high-risk client",
  description: "Alert when high-risk clients withdraw more than $50K in 24 hours",
  conditions: {
    all: [
      {
        label: "High-risk client",
        entity: "client",
        property: "riskTier",
        operator: "equal",
        value: "HIGH",
      },
      {
        label: "Large 24h withdrawal sum",
        entity: "transaction",
        aggregationOperator: "sum",
        aggregationProperty: "convertedAmount",
        aggregationTimeType: "in_the_last",
        aggregationTimeValue: 24,
        aggregationTimePeriod: "hours",
        aggregationFilters: [
          { property: "type", operator: "equal", value: "WITHDRAW" }
        ],
        operator: "greaterThanInclusive",
        value: 50000,
      },
    ],
  },
  actions: [
    {
      type: "CREATE_ALERT",
      config: {
        category: "TRANSACTION_MONITORING",
        priority: "HIGH",
        status: "NEW",
      },
    },
  ],
});
// rule.id — the new rule's ID (status: draft)
```

### Activate / Disable / Delete

```typescript
await corsa.rules.activateRule("rule-uuid", { reason: "Approved by compliance team" });
await corsa.rules.disableRule("rule-uuid", { reason: "Under review" });
await corsa.rules.deleteRule("rule-uuid", "No longer needed");
```

### Update an Active Rule (creates new version)

```typescript
const updated = await corsa.rules.updateRule("rule-uuid", {
  name: "Updated threshold",
  conditions: { /* ... */ },
  reason: "Raised threshold after risk review",
});
// updated.version — incremented version number
```

---

## Rule Templates Service

Access: `corsa.ruleTemplates`

| Method | Description |
|--------|-------------|
| `listRuleTemplates(page?, limit?, search?, filter?)` | Browse pre-built templates |
| `getRuleTemplate(id)` | Get a template by ID |
| `copyRuleTemplate(id)` | Copy template to workspace as a draft rule |

### Browse and Copy a Template

```typescript
const templates = await corsa.ruleTemplates.listRuleTemplates(1, 20);
// Filter: ?search=structuring&filter.products=$eq:crypto

const { ruleId } = await corsa.ruleTemplates.copyRuleTemplate("template-uuid");
// ruleId — draft rule you can customize before activating
```

---

## Evaluation Service

Access: `corsa.evaluation`

| Method | Description |
|--------|-------------|
| `evaluate(requestBody)` | Evaluate rules against a transaction on-demand |
| `getTransactionEvaluations(transactionId, page?, pageSize?)` | Get evaluation history for a transaction |
| `getRuleEvaluations(ruleId, page?, pageSize?)` | Get evaluation history for a rule |

### On-Demand Evaluate

```typescript
const result = await corsa.evaluation.evaluate({
  transactionId: "txn-uuid-123",
  transactionData: {
    amount: 75000,
    convertedAmount: 75000,
    currency: "USD",
    type: "WITHDRAW",
    clientId: "client-uuid-123",
  },
});
```

**Evaluation response:**

| Field | Description |
|-------|-------------|
| `decision` | `ALLOW` — no halt action triggered. `FREEZE` — at least one matched rule has `HALT_TRANSACTION`. |
| `triggeredRuleIds` | Array of rule IDs that matched. |
| `matches` | Per-rule match details including condition-level results. |
| `evaluatedAt` | ISO timestamp. |
| `latencyMs` | Processing time in milliseconds. |

> **Important:** `evaluate()` returns match results but **does not create alerts**. Alerts are only created when transactions flow through the normal ingestion pipeline (`createDeposit`, `createWithdrawal`, `createTrade`) and match active rules.

### Query Evaluation History

```typescript
const byTransaction = await corsa.evaluation.getTransactionEvaluations("txn-uuid", 1, 20);
const byRule = await corsa.evaluation.getRuleEvaluations("rule-uuid", 1, 20);
```

---

## Condition Schema

### Structure

Conditions use nested `all` (AND) / `any` (OR) groups:

```json
{
  "all": [
    { "entity": "transaction", "property": "amount", "operator": "greaterThan", "value": 50000 },
    {
      "any": [
        { "entity": "client", "property": "riskTier", "operator": "equal", "value": "HIGH" },
        { "entity": "client", "property": "country", "operator": "in", "value": ["IR", "KP", "SY"] }
      ]
    }
  ]
}
```

### Condition Fields

| Field | Required | Description |
|-------|----------|-------------|
| `entity` | Yes | `transaction`, `client`, `wallet`, `bankAccount` |
| `property` | Yes (non-aggregation) | The field to check on the entity |
| `operator` | Yes | Comparison operator (see below) |
| `value` | Yes | The threshold or target |
| `label` | No | Human-readable label shown in evaluation results |
| `entityRelationship` | No | `all`, `sender`, or `receiver` (for non-transaction entities) |
| `aggregationOperator` | Aggregation | `sum`, `count`, `avg`, `min`, `max`, `median`, `stddev`, `percentile`, `countDistinct` |
| `aggregationProperty` | Aggregation | Field to aggregate (e.g., `amount`, `convertedAmount`) |
| `aggregationTimeType` | Aggregation | `all_time`, `in_the_last`, `after`, `before`, `between` |
| `aggregationTimeValue` | Conditional | Numeric value for window length |
| `aggregationTimePeriod` | Conditional | `minutes`, `hours`, `days`, `weeks`, `months`, `years` |
| `aggregationFilters` | No | Sub-conditions narrowing which transactions to aggregate |
| `aggregationPercentile` | Conditional | Required when `aggregationOperator` is `percentile` |

### Comparison Operators

| Operator | Description |
|----------|-------------|
| `equal` | Exact match |
| `notEqual` | Not equal |
| `greaterThan` | Strictly greater |
| `greaterThanInclusive` | Greater or equal |
| `lessThan` | Strictly less |
| `lessThanInclusive` | Less or equal |
| `in` | Value in list: `value: ["USD", "EUR"]` |
| `notIn` | Value not in list |
| `contains` | Array field contains the value |
| `doesNotContain` | Array field does not contain the value |
| `between` | Within range (inclusive): `value: [1000, 50000]` |

### Entity Relationships

| Relationship | Description |
|--------------|-------------|
| `all` | Matches if condition is true for any participant (default) |
| `sender` | Evaluate only the sending party |
| `receiver` | Evaluate only the receiving party |

---

## Actions

### `CREATE_ALERT`

Creates a compliance alert when the rule matches.

```json
{
  "type": "CREATE_ALERT",
  "config": {
    "category": "TRANSACTION_MONITORING",
    "priority": "HIGH",
    "status": "NEW",
    "subCategory": "STRUCTURING",
    "assigneeId": "user-uuid",
    "dueDateHours": 48
  }
}
```

| Field | Required | Values |
|-------|----------|--------|
| `category` | Yes | `TRANSACTION_MONITORING` |
| `priority` | Yes | `LOW`, `MEDIUM`, `HIGH` |
| `status` | Yes | `NEW` |
| `subCategory` | No | Custom sub-category string |
| `assigneeId` | No | Corsa user UUID to auto-assign |
| `dueDateHours` | No | Hours from alert creation until due |

When multiple active rules match the same transaction, alerts are **consolidated** — one alert is created with the highest priority across all matching rule configs.

### `HALT_TRANSACTION`

Marks the transaction for halting (`FREEZE` decision) in the evaluation result.

```json
{
  "type": "HALT_TRANSACTION"
}
```

This action appears in the `evaluate()` response as `decision: "FREEZE"`. In the ingestion pipeline (with `evaluateSynchronously: true`), the transaction receives a `HALT` status and the ingestion response includes the evaluation result.

---

## Common Rule Patterns

### Velocity Check (count of transactions)

```typescript
{
  entity: "transaction",
  label: "More than 5 withdrawals in 1 hour",
  aggregationOperator: "count",
  aggregationProperty: "id",
  aggregationTimeType: "in_the_last",
  aggregationTimeValue: 1,
  aggregationTimePeriod: "hours",
  aggregationFilters: [
    { property: "type", operator: "equal", value: "WITHDRAW" }
  ],
  operator: "greaterThan",
  value: 5,
}
```

### Cumulative Threshold (structuring detection)

```typescript
{
  entity: "transaction",
  label: "Total deposits near $10K threshold in 7 days",
  aggregationOperator: "sum",
  aggregationProperty: "convertedAmount",
  aggregationTimeType: "in_the_last",
  aggregationTimeValue: 7,
  aggregationTimePeriod: "days",
  aggregationFilters: [
    { property: "type", operator: "equal", value: "DEPOSIT" }
  ],
  operator: "between",
  value: [8000, 10000],
}
```

### High-Risk Jurisdiction

```typescript
{
  entity: "client",
  label: "Client from sanctioned country",
  entityRelationship: "sender",
  property: "country",
  operator: "in",
  value: ["IR", "KP", "SY", "CU"],
}
```

### Large Single Transaction

```typescript
{
  entity: "transaction",
  label: "Single transaction over $100K",
  property: "convertedAmount",
  operator: "greaterThanInclusive",
  value: 100000,
}
```

---

## Synchronous Evaluation During Ingestion

Set `evaluateSynchronously: true` on the transaction object to evaluate rules inline during deposit, withdrawal, or trade creation:

```typescript
const deposit = await corsa.deposits.createDeposit({
  referenceId: "DEP-001",
  initiatedBy: "client-uuid",
  initiatedAt: "2024-01-15T08:30:00Z",
  depositTransaction: {
    referenceId: "TX-001",
    amount: { amount: 75000, currency: "USD", netAmount: 75000 },
    from: { walletAddress: "0xSource..." },
    to: { walletAddress: "0xDest..." },
    evaluateSynchronously: true,  // ← triggers inline evaluation
    statusHistory: [{ type: "SUCCESS", timestamp: "2024-01-15T08:30:00Z" }],
  },
});
// deposit.depositTransaction.evaluationResult contains the decision
```

The response `evaluationResult` follows the same structure as the `/v1/evaluation/evaluate` response.
