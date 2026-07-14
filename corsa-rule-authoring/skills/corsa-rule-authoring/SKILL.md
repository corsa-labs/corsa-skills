---
name: corsa-rule-authoring
description: >
  Create and manage Corsa compliance rules for transaction monitoring.
  Use when writing rules, configuring rule templates, setting up conditions
  and thresholds, managing the rule lifecycle, evaluating transactions,
  or setting up automated alert generation based on transaction patterns.
---

# Corsa Rule Authoring

You are a Corsa rule authoring specialist. Help developers create, manage, and test transaction monitoring rules using the Corsa compliance API and `@corsa-labs/sdk`.

For complete SDK method signatures, condition schemas, action configurations, and evaluation response formats, see [references/REFERENCE.md](references/REFERENCE.md).

## Rule Lifecycle

Rules follow a strict lifecycle: **Draft** → **Active** → **Disabled**.

```
Create (POST /v1/rules)          → Draft
Activate (POST /v1/rules/{id}/activate) → Active (evaluates incoming transactions)
Disable (POST /v1/rules/{id}/disable)   → Disabled (stops evaluating)
Delete (DELETE /v1/rules/{id})   → Soft-deleted (only non-active rules)
```

**Key behaviors:**
- Only **active** rules evaluate incoming transactions automatically
- Updating an **active** rule creates a new **version** (old version disabled, new version active) — this is atomic and audited
- Updating a **draft/disabled** rule modifies it in place
- Deleting an **active** rule fails — disable it first
- `activate` and `disable` accept an optional `reason` for audit trail

## Quick Start: From Template

The fastest way to create a rule is to copy a pre-built template:

```typescript
// 1. Browse available templates
const templates = await client.ruleTemplates.listRuleTemplates(1, 20);

// 2. Copy a template to your workspace (creates a draft rule)
const { ruleId } = await client.ruleTemplates.copyRuleTemplate("template-uuid");

// 3. Customize the draft
const updated = await client.rules.updateRule(ruleId, {
  name: "My customized rule",
  description: "Adjusted thresholds for our risk profile",
  conditions: {
    all: [
      {
        label: "Large withdrawal",
        entity: "transaction",
        property: "amount",
        operator: "greaterThanInclusive",
        value: 25000,
      },
    ],
  },
});

// 4. Activate when ready
const active = await client.rules.activateRule(ruleId);
```

## Quick Start: Custom Rule

```typescript
const rule = await client.rules.createRule({
  name: "High-risk withdrawal detection",
  description: "Alert when high-risk customers make large withdrawals exceeding $50,000 in 24 hours",
  conditions: {
    all: [
      {
        label: "High-risk customer",
        entity: "client",
        property: "riskTier",
        operator: "equal",
        value: "HIGH",
      },
      {
        label: "Large withdrawal amount",
        entity: "transaction",
        aggregationProperty: "amount",
        aggregationOperator: "sum",
        aggregationTimeType: "in_the_last",
        aggregationTimeValue: 1,
        aggregationTimePeriod: "days",
        aggregationFilters: [
          { property: "type", operator: "equal", value: "WITHDRAW" },
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

// Activate to start evaluating transactions
await client.rules.activateRule(rule.id);
```

## Rule Structure

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Human-readable rule name |
| `description` | No | Detailed purpose of the rule |
| `conditions` | Yes | Conditions using `all` (AND) / `any` (OR) logic |
| `actions` | Yes | Actions to execute when conditions match |

## Conditions

Conditions define **when** a rule matches. They use boolean logic with `all` (AND) and `any` (OR) groups that can be nested.

### Condition Fields

| Field | Description |
|-------|-------------|
| `label` | Human-readable name (appears in evaluation results and narratives) |
| `entity` | What to check: `transaction`, `client`, `wallet`, `bankAccount` |
| `entityRelationship` | For client conditions: `all`, `sender`, or `receiver` |
| `property` | Field to evaluate (e.g., `amount`, `riskTier`, `country`) |
| `operator` | Comparison operator |
| `value` | Value to compare against |
| `path` | Optional JSONPath for nested fields |

### Operators

| Operator | Description |
|----------|-------------|
| `equal` | Exact match |
| `notEqual` | Not equal |
| `greaterThan` | Greater than (exclusive) |
| `greaterThanInclusive` | Greater than or equal |
| `lessThan` | Less than (exclusive) |
| `lessThanInclusive` | Less than or equal |
| `contains` | String/array contains |
| `doesNotContain` | String/array does not contain |
| `in` | Value is in a list |
| `notIn` | Value is not in a list |
| `between` | Value is between two bounds |

### Aggregation Conditions (Rolling Thresholds)

For time-windowed checks like "total withdrawals in the last 24 hours":

| Field | Description |
|-------|-------------|
| `aggregationProperty` | Field to aggregate (e.g., `amount`) |
| `aggregationOperator` | `sum`, `count`, `avg`, `min`, `max`, `median`, `stddev`, `percentile`, `countDistinct`, `first`, `last` |
| `aggregationTimeType` | `all_time`, `in_the_last`, `after`, `before`, `between` |
| `aggregationTimeValue` | Number (e.g., `1`, `24`, `7`) |
| `aggregationTimePeriod` | `minutes`, `hours`, `days`, `weeks`, `months`, `years` |
| `aggregationFilters` | Additional filters on the aggregated transactions |
| `aggregationPercentile` | Percentile value (when using `percentile` operator) |

### Examples

**Simple property check:**

```typescript
{
  label: "High-risk customer",
  entity: "client",
  property: "riskTier",
  operator: "equal",
  value: "HIGH",
}
```

**Aggregation with time window:**

```typescript
{
  label: "Total withdrawals exceed $50K in 24h",
  entity: "transaction",
  aggregationProperty: "amount",
  aggregationOperator: "sum",
  aggregationTimeType: "in_the_last",
  aggregationTimeValue: 1,
  aggregationTimePeriod: "days",
  aggregationFilters: [
    { property: "type", operator: "equal", value: "WITHDRAW" },
  ],
  operator: "greaterThanInclusive",
  value: 50000,
}
```

**Nested boolean logic:**

```typescript
{
  all: [
    {
      label: "Large amount",
      entity: "transaction",
      property: "amount",
      operator: "greaterThanInclusive",
      value: 10000,
    },
    {
      any: [
        {
          label: "High-risk country sender",
          entity: "client",
          entityRelationship: "sender",
          property: "country",
          operator: "in",
          value: ["IR", "KP", "SY"],
        },
        {
          label: "High-risk country receiver",
          entity: "client",
          entityRelationship: "receiver",
          property: "country",
          operator: "in",
          value: ["IR", "KP", "SY"],
        },
      ],
    },
  ],
}
```

## Actions

Actions define **what happens** when a rule matches.

| Action Type | Description |
|-------------|-------------|
| `CREATE_ALERT` | Creates a compliance alert with the specified category, priority, and status |
| `HALT_TRANSACTION` | Marks the transaction for halting (contributes to `FREEZE` decision) |

### CREATE_ALERT Config

```typescript
{
  type: "CREATE_ALERT",
  config: {
    category: "TRANSACTION_MONITORING",
    priority: "HIGH",      // Highest priority wins when multiple rules match
    status: "NEW",
    // Optional routing fields:
    subCategory: "Structuring",          // Granular classification within the category
    description: "Custom alert message", // Override the default alert description
    assigneeId: "analyst-uuid",          // Assign to a specific analyst on creation
    dueDateHours: 24,                     // Due date offset in hours from alert creation time
  },
}
```

When multiple rules match the same transaction, alerts are **consolidated** — one alert is created with the highest priority across all matching rules.

### HALT_TRANSACTION

```typescript
{
  type: "HALT_TRANSACTION",
  config: {},
}
```

Adds `shouldHalt: true` to the alert's metadata and contributes to a `FREEZE` evaluation decision. Typically combined with `CREATE_ALERT`.

## Evaluation

### Automatic Evaluation

Active rules are automatically evaluated when transactions are ingested through the normal flow (deposits, withdrawals, trades). Matched rules trigger their configured actions (alert creation, halt).

### On-Demand Evaluation (API)

Test rules or evaluate specific transactions without triggering actions:

```typescript
const result = await client.evaluation.evaluate({
  transactionId: "txn-uuid-123",
  transactionData: {
    amount: 75000,
    currency: "USD",
    type: "WITHDRAW",
    clientId: "client-uuid-123",
  },
});

console.log(result.decision);       // "ALLOW" or "FREEZE"
console.log(result.triggeredRuleIds); // IDs of matched rules
console.log(result.matches);         // Detailed match info per rule
console.log(result.latencyMs);       // Evaluation latency
```

**Important:** The on-demand `evaluate` endpoint checks rules and returns results but does **not** create alerts or execute actions. Alerts are only created through the normal transaction ingestion pipeline.

### View Evaluation History

```typescript
// By transaction — which rules evaluated this transaction?
const byTx = await client.evaluation.getTransactionEvaluations("txn-uuid-123", 1, 20);

// By rule — which transactions did this rule evaluate?
const byRule = await client.evaluation.getRuleEvaluations("rule-uuid", 1, 20);
```

## SDK Methods Reference

### Rules (`client.rules`)

| Method | Description |
|--------|-------------|
| `createRule(requestBody)` | Create a rule (draft) |
| `listRules(page?, limit?, ...)` | List rules with filtering and pagination |
| `getRule(id, version?)` | Get a rule by ID (optionally a specific version) |
| `updateRule(id, requestBody)` | Update a rule (versioned if active) |
| `activateRule(id, requestBody?)` | Activate a rule (optional `reason` for audit) |
| `disableRule(id, requestBody?)` | Disable a rule (optional `reason` for audit) |
| `deleteRule(id, reason?)` | Soft delete a non-active rule |

### Rule Templates (`client.ruleTemplates`)

| Method | Description |
|--------|-------------|
| `listRuleTemplates(page?, limit?, ...)` | List templates with filtering |
| `getRuleTemplate(id)` | Get a template by ID |
| `copyRuleTemplate(id)` | Copy a template as a draft rule in your workspace |

### Evaluation (`client.evaluation`)

| Method | Description |
|--------|-------------|
| `evaluate(requestBody)` | Evaluate transaction data against active rules |
| `getTransactionEvaluations(transactionId, page?, pageSize?)` | Get evaluation results for a transaction |
| `getRuleEvaluations(ruleId, page?, pageSize?)` | Get evaluation results for a rule |

## Best Practices

1. **Start from templates** — browse and copy pre-built templates, then customize thresholds for your risk profile
2. **Use clear labels** on conditions — they appear in evaluation results and help with audit narratives
3. **Test before activating** — use the on-demand evaluate endpoint to verify rule behavior with sample data
4. **Provide reasons** when activating, disabling, or deleting rules — these are recorded in the audit trail
5. **Use aggregations for velocity checks** — sum/count over time windows catches structuring and rapid-fire patterns
6. **Combine `CREATE_ALERT` with `HALT_TRANSACTION`** only when the rule should freeze the transaction, not just alert
7. **Version awareness** — updating an active rule creates a new version atomically; the old version is preserved for audit

## Common Pitfalls

| Mistake | Fix |
|---------|-----|
| Expecting `evaluate` API to create alerts | On-demand evaluation only returns match results — alerts come from the transaction ingestion pipeline |
| Trying to delete an active rule | Disable it first, then delete |
| Forgetting to activate after creating | Rules start as drafts — call `activateRule` when ready |
| No labels on conditions | Always add labels — they make evaluation results readable |
| Overly broad conditions | Use `aggregationFilters` to narrow what gets aggregated |

## Links

- [Rules & Evaluation Guide](https://docs.corsa.finance/transaction-monitoring/rules-api)
- [API Reference](https://api.corsa.finance/api-spec.json)
