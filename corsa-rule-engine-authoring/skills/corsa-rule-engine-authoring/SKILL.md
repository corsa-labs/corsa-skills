---
name: corsa-rule-engine-authoring
description: >
  Deep transaction-monitoring rule authoring for the Corsa rule engine.
  Use when translating a compliance policy into a valid rule JSON payload,
  building complex conditions with nested AND/OR logic, aggregation/velocity
  time windows, configuring CREATE_ALERT or HALT_TRANSACTION actions,
  testing rules against transaction history, or managing rule lifecycle
  (draft → active → disabled). Works at the REST API level for maximum
  precision — pair with corsa-rule-spec-drafting for the policy-translation step.
---

# Corsa Rule Engine Authoring

You are a Corsa rule engine specialist. Help build, test, and manage transaction-monitoring
rules via the Corsa REST API. Rules are evaluated **per transaction** — a rule inspects
the transaction and its participants (sender/receiver client, wallet, bank account) and
fires configured actions when conditions match.

## Rule Lifecycle

```
Create   POST /v1/rules                 → status: "draft"
Activate POST /v1/rules/{id}/activate   → status: "active"  (evaluates all new transactions)
Disable  POST /v1/rules/{id}/disable    → status: "disabled"
Delete   DELETE /v1/rules/{id}          → soft-delete (non-active rules only)
```

Key lifecycle behaviours:
- Only **active** rules evaluate incoming transactions.
- Updating an **active** rule atomically creates a new version (old version is disabled).
- Updating a **draft** or **disabled** rule modifies it in place.
- Disable a rule before deleting it.
- `activate` and `disable` accept an optional `{ "reason": "..." }` body for the audit trail.

---

## Rule Structure

```json
POST /v1/rules
{
  "name": "Human-readable rule name",
  "description": "Why this rule exists",
  "conditions": { ... },
  "actions": [ ... ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique, descriptive name |
| `description` | No | Narrative purpose — appears in evaluation results |
| `conditions` | Yes | Boolean tree of condition objects |
| `actions` | Yes | Array of action objects to fire on match |

---

## Conditions

Conditions are a recursive boolean tree using `all` (AND) and `any` (OR).

```json
{
  "all": [
    { <condition> },
    {
      "any": [
        { <condition> },
        { <condition> }
      ]
    }
  ]
}
```

### Condition Fields

| Field | Description |
|-------|-------------|
| `label` | Human-readable name — appears in evaluation results and alert narratives |
| `entity` | What to inspect: `transaction`, `client`, `wallet`, `bankAccount` |
| `entityRelationship` | For `client`/`wallet`/`bankAccount`: `all`, `sender`, `receiver` |
| `property` | Field to compare (see catalogs below) |
| `operator` | Comparison operator (see table below) |
| `value` | Value to compare against |
| `path` | Optional JSONPath for nested fields on the entity |

### Operators

| Operator | Types | Notes |
|----------|-------|-------|
| `equal` | any | Exact match |
| `notEqual` | any | |
| `greaterThan` | number | Exclusive |
| `greaterThanInclusive` | number | |
| `lessThan` | number | Exclusive |
| `lessThanInclusive` | number | |
| `contains` | string / array | Substring or array membership |
| `doesNotContain` | string / array | |
| `in` | any | Value in provided list |
| `notIn` | any | |
| `between` | number | Inclusive bounds: `value: [low, high]` |
| `isNull` | any | Field is absent / null |
| `isNotNull` | any | Field is present and non-null |

No regex, fuzzy, or semantic matching — only the operators above are supported.

### Entity Property Catalog

**`transaction`**

| Property | Type | Description |
|----------|------|-------------|
| `amount` | number | Transaction amount in base currency |
| `convertedAmount` | number | Amount converted to platform currency |
| `currency` | string | ISO 4217 currency code (e.g. `USD`, `BTC`) |
| `type` | string | `DEPOSIT`, `WITHDRAW`, `TRADE`, `TRANSFER` |
| `status` | string | `SUCCESS`, `PENDING`, `FAILED`, `CANCELLED` |
| `paymentMethod` | string | e.g. `CRYPTO`, `BANK_TRANSFER`, `CARD` |
| `blockchainNetworkId` | string | e.g. `ethereum-mainnet`, `bitcoin-mainnet` |
| `country` | string | ISO 3166-1 alpha-2 country code |
| `riskLevel` | string | `LOW`, `MEDIUM`, `HIGH`, `VERY_HIGH` |
| `riskScore` | number | 0–100 |
| `createdAt` | string | ISO 8601 timestamp |

**`client`** (individual or corporate)

| Property | Type | Description |
|----------|------|-------------|
| `riskTier` | string | `LOW`, `MEDIUM`, `HIGH`, `VERY_HIGH` |
| `riskScore` | number | 0–100 |
| `kycStatus` | string | `APPROVED`, `PENDING`, `REJECTED`, `EXPIRED` |
| `accountStatus` | string | `APPROVED`, `SUSPENDED`, `CLOSED` |
| `activityStatus` | string | `ACTIVE`, `DORMANT` |
| `sanctionsStatus` | string | `CLEAR`, `MATCH`, `POTENTIAL_MATCH` |
| `pepStatus` | string | `CLEAR`, `MATCH`, `POTENTIAL_MATCH` |
| `adverseMediaStatus` | string | `CLEAR`, `MATCH`, `POTENTIAL_MATCH` |
| `country` | string | Client's country of residence/registration |
| `jurisdiction` | string | Regulatory jurisdiction |
| `type` | string | `INDIVIDUAL`, `CORPORATE` |
| `kycTier` | string | Platform-defined KYC tier (e.g. `TIER_1`, `TIER_2`) |
| `dailyLimit` | number | Client's configured daily transaction limit |
| `monthlyLimit` | number | Client's configured monthly transaction limit |
| `tags` | array | Custom tags assigned to the client |
| `daysSinceOnboarding` | number | Days since `onboardedAt` |

**`wallet`** (blockchain wallet)

| Property | Type | Description |
|----------|------|-------------|
| `riskLevel` | string | `LOW`, `MEDIUM`, `HIGH`, `VERY_HIGH` |
| `riskScore` | number | 0–100 |
| `address` | string | Blockchain address |
| `blockchainNetworkId` | string | e.g. `ethereum-mainnet` |

**`bankAccount`**

| Property | Type | Description |
|----------|------|-------------|
| `riskLevel` | string | `LOW`, `MEDIUM`, `HIGH`, `VERY_HIGH` |
| `riskScore` | number | 0–100 |
| `country` | string | Bank country |
| `currency` | string | Account currency |
| `bankName` | string | Bank name |

---

## Aggregation Conditions (Velocity & Rolling Thresholds)

For time-windowed checks ("total sender deposits in the last 7 days", "distinct counterparties in 24h"):

| Field | Description |
|-------|-------------|
| `aggregationProperty` | Field to aggregate — `amount`, `convertedAmount`, `txHash`, `clientId`, or any `transaction` property |
| `aggregationOperator` | See table below |
| `aggregationTimeType` | `all_time`, `in_the_last`, `after`, `before`, `between` |
| `aggregationTimeValue` | Numeric value (e.g. `1`, `24`, `7`) |
| `aggregationTimePeriod` | `minutes`, `hours`, `days`, `weeks`, `months`, `years` |
| `aggregationFilters` | Array of simple conditions applied to the aggregated transactions |
| `aggregationPercentile` | Percentile value 0–100 (only for `percentile` operator) |
| `entityRelationship` | `sender` or `receiver` — which side of the transaction to aggregate |

### Aggregation Operators

| Operator | Description |
|----------|-------------|
| `sum` | Total of the property |
| `count` | Number of transactions |
| `countDistinct` | Unique values of the property |
| `avg` | Mean |
| `min` | Minimum |
| `max` | Maximum |
| `median` | Median |
| `stddev` | Standard deviation |
| `percentile` | Given percentile (requires `aggregationPercentile`) |
| `velocityChange` | Rate of change over time |
| `movingAverage` | Rolling mean |
| `deviationFromAverage` | How far current period deviates from historical average |
| `baselineComparison` | Compares current period to a longer baseline period |

### Aggregation Filter Fields

Each filter in `aggregationFilters` is a simple `{ property, operator, value }` — same operators as simple conditions.

```json
{
  "label": "Deposits from sender exceeding $50K in 24h",
  "entity": "transaction",
  "entityRelationship": "sender",
  "aggregationProperty": "convertedAmount",
  "aggregationOperator": "sum",
  "aggregationTimeType": "in_the_last",
  "aggregationTimeValue": 24,
  "aggregationTimePeriod": "hours",
  "aggregationFilters": [
    { "property": "type", "operator": "equal", "value": "DEPOSIT" }
  ],
  "operator": "greaterThanInclusive",
  "value": 50000
}
```

### Dynamic Thresholds (compare against client limits)

Use `path` to reference a per-client configured limit instead of a hardcoded value:

```json
{
  "label": "Deposit exceeds client daily limit",
  "entity": "transaction",
  "aggregationProperty": "convertedAmount",
  "aggregationOperator": "sum",
  "aggregationTimeType": "in_the_last",
  "aggregationTimeValue": 1,
  "aggregationTimePeriod": "days",
  "aggregationFilters": [
    { "property": "type", "operator": "equal", "value": "DEPOSIT" }
  ],
  "operator": "greaterThanInclusive",
  "path": "client.sender.dailyLimit"
}
```

---

## Actions

```json
"actions": [
  {
    "type": "CREATE_ALERT",
    "config": {
      "category": "TRANSACTION_MONITORING",
      "priority": "HIGH",
      "status": "NEW",
      "subCategory": "Structuring",
      "description": "Custom alert narrative",
      "assigneeId": "analyst-uuid",
      "dueDateHours": 48
    }
  },
  {
    "type": "HALT_TRANSACTION",
    "config": {}
  }
]
```

| Action | Effect |
|--------|--------|
| `CREATE_ALERT` | Creates a compliance alert. When multiple rules match the same transaction, alerts are **consolidated** — one alert is created at the highest priority across matching rules. |
| `HALT_TRANSACTION` | Contributes `FREEZE` to the evaluation decision. Use alongside `CREATE_ALERT`. |

**Priority values** (highest wins on consolidation): `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`

**Category values**: `TRANSACTION_MONITORING`, `CLIENT_RISK`, `SANCTIONS`, `FRAUD`

---

## Common Rule Patterns

### Structuring — multiple deposits just below threshold

```json
{
  "name": "Structuring — 3+ deposits $9K–$9,999 in 24h",
  "conditions": {
    "all": [
      {
        "label": "3 or more deposits in range",
        "entity": "transaction",
        "entityRelationship": "receiver",
        "aggregationProperty": "amount",
        "aggregationOperator": "count",
        "aggregationTimeType": "in_the_last",
        "aggregationTimeValue": 24,
        "aggregationTimePeriod": "hours",
        "aggregationFilters": [
          { "property": "type", "operator": "equal", "value": "DEPOSIT" },
          { "property": "amount", "operator": "greaterThanInclusive", "value": 9000 },
          { "property": "amount", "operator": "lessThan", "value": 10000 }
        ],
        "operator": "greaterThanInclusive",
        "value": 3
      }
    ]
  },
  "actions": [{ "type": "CREATE_ALERT", "config": { "category": "TRANSACTION_MONITORING", "priority": "HIGH", "status": "NEW" } }]
}
```

### High-risk country + large amount

```json
{
  "conditions": {
    "all": [
      {
        "label": "Transaction to/from high-risk country",
        "entity": "client",
        "entityRelationship": "all",
        "property": "country",
        "operator": "in",
        "value": ["IR", "KP", "SY", "CU"]
      },
      {
        "label": "Amount above $10,000",
        "entity": "transaction",
        "property": "convertedAmount",
        "operator": "greaterThanInclusive",
        "value": 10000
      }
    ]
  }
}
```

### Dormant account reactivation with large deposit

```json
{
  "conditions": {
    "all": [
      {
        "label": "Client was dormant",
        "entity": "client",
        "property": "activityStatus",
        "operator": "equal",
        "value": "DORMANT"
      },
      {
        "label": "Large deposit",
        "entity": "transaction",
        "property": "type",
        "operator": "equal",
        "value": "DEPOSIT"
      },
      {
        "label": "Amount over $5,000",
        "entity": "transaction",
        "property": "convertedAmount",
        "operator": "greaterThanInclusive",
        "value": 5000
      }
    ]
  }
}
```

---

## Evaluation API

### On-Demand Testing (no alert side-effects)

```bash
POST /v1/evaluation/evaluate
{
  "transactionId": "txn-uuid",
  "transactionData": {
    "amount": 75000,
    "currency": "USD",
    "type": "WITHDRAW"
  }
}
```

Response:
```json
{
  "decision": "FREEZE",
  "triggeredRuleIds": ["rule-uuid-1"],
  "matches": [ ... ],
  "latencyMs": 12
}
```

**Important:** The `evaluate` endpoint does **not** create alerts or execute actions — it only returns what *would* happen. Alerts are created only through the normal transaction ingestion pipeline.

### View Evaluation History

```bash
GET /v1/evaluation/transaction/{transactionId}/results?page=1&pageSize=20
GET /v1/evaluation/rule/{ruleId}/results?page=1&pageSize=20
```

---

## Rule Templates

Templates are pre-built rule definitions that can be copied and customized:

```bash
GET /v1/rule-templates?page=1&limit=20   # browse available templates
GET /v1/rule-templates/{id}              # get a specific template
POST /v1/rule-templates/{id}/copy        # copy as a draft rule → returns { ruleId }
```

After copying, update the draft rule via `PUT /v1/rules/{id}` and activate it.

---

## Key API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/rules` | Create a rule (starts as draft) |
| `GET` | `/v1/rules` | List rules (filter by status, name, page) |
| `GET` | `/v1/rules/{id}` | Get a rule (optionally `?version=N`) |
| `PUT` | `/v1/rules/{id}` | Update a rule |
| `POST` | `/v1/rules/{id}/activate` | Activate — `{ "reason": "..." }` |
| `POST` | `/v1/rules/{id}/disable` | Disable — `{ "reason": "..." }` |
| `DELETE` | `/v1/rules/{id}` | Soft-delete (non-active only) |
| `POST` | `/v1/evaluation/evaluate` | On-demand evaluation (no side-effects) |
| `GET` | `/v1/evaluation/transaction/{id}/results` | Evaluation history for a transaction |
| `GET` | `/v1/evaluation/rule/{id}/results` | Evaluation history for a rule |
| `GET` | `/v1/rule-templates` | Browse templates |
| `POST` | `/v1/rule-templates/{id}/copy` | Copy template as draft rule |

All requests require `Authorization: Bearer <API_TOKEN>:<API_SECRET>`.

---

## What Rules Cannot Do

| Request | What to do instead |
|---------|-------------------|
| Regex or fuzzy matching on text | Use `equal`, `in`, or `contains` on defined enum fields only |
| Live OFAC / PEP lookups inside a rule | Sanctions/PEP are pre-computed flags on the `client` entity |
| ML models or custom anomaly scoring | Use the built-in statistical aggregation operators (`stddev`, `deviationFromAverage`) |
| Graph / multi-hop fund tracing | Only direct transaction participants are available |
| Periodic / scheduled rules | Rules fire on transaction ingestion only — use Workflows for scheduled checks |
| Emails, webhooks, case creation | Use `CREATE_ALERT`; Workflows can react to the resulting alert |

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Calling `evaluate` and expecting alerts | `evaluate` is test-only; alerts come from the ingestion pipeline |
| Deleting an active rule | Disable it first (`POST /v1/rules/{id}/disable`), then delete |
| Forgetting to activate after creating | Rules start as `draft` — call `POST /v1/rules/{id}/activate` |
| No labels on conditions | Always add `label` — required for readable evaluation results |
| Hardcoding thresholds that should scale per client | Use `path: "client.sender.dailyLimit"` for dynamic thresholds |
| Combining `HALT_TRANSACTION` without `CREATE_ALERT` | Add `CREATE_ALERT` so analysts see why the transaction was halted |

---

## Works Best With

- **corsa-rule-spec-drafting** — turn a compliance policy clause into a grounded rule description first, then pass it here for implementation.
- **corsa-rule-authoring** — for SDK-based rule creation (TypeScript `@corsa-labs/sdk`).

## Links

- [Rules & Evaluation Guide](https://docs.corsa.finance/transaction-monitoring/rules-api)
- [Testing Rules](https://docs.corsa.finance/transaction-monitoring/testing-rules)
- [External Rules](https://docs.corsa.finance/transaction-monitoring/external-rules)
- [API Reference](https://api.corsa.finance/api-spec.json)
