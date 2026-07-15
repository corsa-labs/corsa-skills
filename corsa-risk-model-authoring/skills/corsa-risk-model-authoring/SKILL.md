---
name: corsa-risk-model-authoring
description: >
  Author, edit, simulate, deploy, activate, and run Corsa risk-rating models
  (risk formulas) for individual and corporate clients. Use when creating or
  changing a risk formula, defining scoring factors / pipelines / thresholds /
  override conditions, discovering referenceable factor fields and operators,
  testing a formula via simulation before activation, running risk assessments
  by entity data or entity id, or resolving the active formula for an entity type.
---

# Corsa Risk Model Authoring

You are a Corsa risk-rating specialist. Help engineers author, test, and ship **risk models** (a.k.a. **risk formulas**) in the `risk-rating-service` — the service that scores a client into a risk **level** (`LOW`/`MEDIUM`/`HIGH`) and **score** (0–100) from weighted factors.

The interface is a **REST v1 API** plus the generated **`@corsa-labs/risk-rating-service-rest-client`** SDK. There is no visual builder surface to script — you author formulas as JSON payloads and POST them. Everything you need is in this file.

> The repo's `README-RISK-RATING.md` is **stale** — it documents a `corporate-clients` module, `/v1/corporate-clients/risk/...` routes, a `clientData` request field, and three `calculationMethod`s (custom/lookup/rules). None of that is true anymore. Trust this skill and the source; see [README discrepancies](#readme-discrepancies) at the end.

## Table of Contents

1. [Mental model](#mental-model)
2. [Vocabulary](#vocabulary)
3. [Minimal valid formula](#minimal-valid-formula)
4. [Formula structure](#formula-structure)
5. [Scoring math & risk levels](#scoring-math--risk-levels)
6. [Operators reference](#operators-reference)
7. [Factor field catalog](#factor-field-catalog)
8. [Condition templates & country lists](#condition-templates--country-lists)
9. [Custom fields](#custom-fields)
10. [Lifecycle, versioning & active-formula resolution](#lifecycle-versioning--active-formula-resolution)
11. [REST v1 endpoints](#rest-v1-endpoints)
12. [SDK client](#sdk-client)
13. [Simulation: test before activation](#simulation-test-before-activation)
14. [Running assessments](#running-assessments)
15. [Validation & limits](#validation--limits)
16. [Worked example — individual-client KYC model](#worked-example--individual-client-kyc-model)
17. [Worked example — corporate-client KYB model](#worked-example--corporate-client-kyb-model)
18. [Best practices](#best-practices)
19. [Common pitfalls](#common-pitfalls)
20. [Error handling](#error-handling)
21. [Source map & doc links](#source-map--doc-links)
22. [README discrepancies](#readme-discrepancies)

---

## Mental model

A risk formula is **four layers**, evaluated top to bottom:

```
entityData (client snapshot)
     │
     ▼
1. FACTORS (pipelines)   each factor resolves a source field, evaluates its
     │                   scoreConditions, and assigns points (max-score-wins)
     ▼
   totalScore = plain SUM of all factor scores        (no weighting step)
     │
     ▼
2. THRESHOLDS            totalScore → model band: LOW | MEDIUM | HIGH
     │
     ▼
3. OVERRIDE CONDITIONS   optional. First matching rule forces the *effective*
     │                   band (e.g. sanctioned → HIGH). Does NOT change score.
     ▼
4. RULES (json-rules-engine)   optional. Produces recommendations / triggered
                         events only. NEVER changes scores or bands.
```

Key truths that trip people up:

- **There is no `weight` field.** A factor's "weight" is simply its `maxScore` — its share of the 100-point budget. A factor with `maxScore: 25` can contribute up to 25 of 100 points. Design weights by choosing `maxScore` values.
- **Total is a plain sum**, not an average or weighted mean. The sum of all factor `maxScore`s must be **≤ 100** (validated at save).
- **Model band vs effective band.** Scoring produces the *model* `riskLevel` from thresholds. An override can set a different `effectiveRiskLevel` — but the persisted assessment's `riskLevel` stays the model band; the override is reported separately in `overrideApplied` / `effectiveRiskLevel`.
- **Only `CORPORATE_CLIENT` and `INDIVIDUAL_CLIENT` are scorable.** `TRANSACTION` and `BLOCKCHAIN_WALLET` are valid `EntityType`s but have **no referenceable fields**, so you cannot build a meaningful formula for them today.
- **Every factor must be `calculationMethod: 'pipeline'`.** Legacy `custom`/`lookup`/`rules` factor methods are rejected at validation.

---

## Vocabulary

| Term | Meaning |
|------|---------|
| **Formula / model** | A named, versioned scoring configuration scoped to one `platformId` + `entityType`. |
| **Factor** | One risk component (`RiskFactorDefinition`) with a name, `minScore`/`maxScore` bounds, and a `pipeline`. |
| **Pipeline** | How a factor scores: a `sourceField` + ordered `scoreConditions` + optional `defaultScore`. |
| **Score condition** | `{ conditions: { field?, operator, value?/min?/max? }, score }` — award `score` points when the condition matches. |
| **Thresholds** | `{ low, medium, high }`, each `{ min, max }` — map `totalScore` to a band. |
| **Override condition** | Forces the effective band when its conditions match (first match wins). |
| **Rules** | `RuleProperties[]` from `json-rules-engine`; post-score recommendations only. **Required field** — pass `[]` if unused. |
| **`EntityType`** | `CORPORATE_CLIENT` \| `INDIVIDUAL_CLIENT` \| `TRANSACTION` \| `BLOCKCHAIN_WALLET`. |
| **`FormulaStatus`** | `DRAFT` \| `ACTIVE` \| `INACTIVE`. Max one `ACTIVE` per `platformId` + `entityType`. |
| **`formulaGroupId`** | Logical lineage id shared by all versions of a formula. |
| **`version`** | String label like `"0.1"`, `"1.2"`; minor increments on each deploy/activate. |

---

## Minimal valid formula

The smallest formula that passes validation — one pipeline factor, contiguous non-overlapping bands, and the mandatory (possibly empty) `rules` array:

```json
{
  "name": "Minimal individual model",
  "entityType": "INDIVIDUAL_CLIENT",
  "factors": [
    {
      "name": "sanctionsScreen",
      "description": "Sanctions hit contributes the full budget",
      "minScore": 0,
      "maxScore": 100,
      "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "sanctions.isSanctioned",
        "scoreConditions": [
          { "conditions": { "operator": "equals", "value": true }, "score": 100 }
        ],
        "defaultScore": 0
      }
    }
  ],
  "thresholds": {
    "low":    { "min": 0,  "max": 33 },
    "medium": { "min": 34, "max": 66 },
    "high":   { "min": 67, "max": 100 }
  },
  "rules": []
}
```

`rules: []` is **not optional** in the create payload — omit it and validation fails. `overrideConditions` is optional. Created formulas start as `DRAFT`; you must `activate` or `deploy` before they score anything.

---

## Formula structure

`CreateRiskFormulaDto` (`src/risk-rating/dto/risk-formula.dto.ts:268`):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | `string` | ✅ | Human-readable model name. |
| `description` | `string` | — | Optional. |
| `entityType` | `EntityType` | ✅ | `CORPORATE_CLIENT` or `INDIVIDUAL_CLIENT` (others have no fields). Immutable after create. |
| `factors` | `RiskFactorDefinition[]` | ✅ | ≥1 factor; sum of `maxScore` ≤ 100. |
| `thresholds` | `RiskThresholds` | ✅ | `{ low, medium, high }`, each `{ min, max }`. |
| `rules` | `RuleProperties[]` | ✅ | `json-rules-engine` rules for recommendations. Pass `[]` if none. |
| `overrideConditions` | `OverrideCondition[]` | — | Optional band overrides. |

### Factor — `RiskFactorDefinition` (`risk-formula.entity.ts:18`, DTO `:121`)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | `string` (UUID) | — | Stable id within a formula lineage. Omit on create; the service assigns one. |
| `name` | `string` | ✅ | Factor name (shown in the per-factor breakdown). |
| `description` | `string` | ✅ | Required — not optional. |
| `minScore` | `number` | ✅ | Must be `< maxScore`. Lower clamp of the factor's output. |
| `maxScore` | `number` | ✅ | Upper clamp **and** the factor's weight in the 100-point budget. |
| `calculationMethod` | `'pipeline'` | ✅ | Only `'pipeline'` is accepted. |
| `groupName` | `string` | — | UI theme group. If omitted, derived from the capitalized first path segment of `sourceField` (e.g. `general.*` → `General`). |
| `pipeline` | `Pipeline` | ✅* | Required in practice — validation rejects a factor without one. |

### Pipeline — `Pipeline` (DTO `:86`)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `sourceField` | `string \| string[]` | ✅ | Dot-path into the entity (see [catalog](#factor-field-catalog)). An array = multi-source (max-score-wins across sources). Always stored as an array. |
| `scoreConditions` | `ScoreCondition[]` | ✅ | ≥1 condition. Evaluated with max-score-wins. |
| `defaultScore` | `number` | — | Awarded when **no** condition matches. Falsy/undefined → `0`. |
| `customFieldMapping` | `{ key, fieldType }` | — | Required only when `sourceField` is a `customFields.*` path. See [custom fields](#custom-fields). |

### Score condition — `ScoreCondition` (`risk-formula.entity.ts:7`)

```jsonc
{
  "conditions": {
    "field": "some.other.path",   // optional; defaults to the pipeline sourceField value
    "operator": "in",             // one of the 10 scoring operators
    "value": ["A", "B"],          // for equals/contains/in/comparisons
    "min": 1,                     // for "between" only (number OR "YYYY-MM-DD")
    "max": 3                      // for "between" only
  },
  "score": 5                      // points awarded when this condition matches
}
```

- `between` uses **`min`/`max`**, never `value`. Every other operator uses `value`.
- Date fields compare by ISO **day-key** (`YYYY-MM-DD`) for `equals`, and by timestamp for comparisons. Pass dates as `YYYY-MM-DD` strings.

### Thresholds — `RiskThresholds` (DTO `:170`)

```json
{ "low": {"min":0,"max":33}, "medium": {"min":34,"max":66}, "high": {"min":67,"max":100} }
```

- **Must not overlap**: `low.max < medium.min` and `medium.max < high.min` (validated; gaps are allowed but pointless).
- At scoring time only `high.min` and `medium.min` are read (see [scoring math](#scoring-math--risk-levels)). Keep `high.max` at your top of range (usually 100) for clarity even though it isn't consulted.

### Override condition — `OverrideCondition` (DTO `:213`)

Two shapes — single-field (legacy) or compound (AND across `conditions[]`):

```jsonc
// Single field
{ "field": "sanctions.isSanctioned", "operator": "equals", "value": true,
  "overrideLevel": "HIGH", "reason": "Sanctioned entity" }

// Compound (ALL conditions must match)
{ "conditions": [
    { "field": "politicalExposure.isPoliticallyExposed", "operator": "equals", "value": true },
    { "field": "general.countryOfIncorporation", "operator": "in", "value": ["IRN","PRK"] }
  ],
  "overrideLevel": "HIGH", "reason": "PEP in prohibited jurisdiction" }
```

- `overrideLevel` ∈ `LOW`/`MEDIUM`/`HIGH`; `reason` is **required**.
- Overrides use a **different operator set** than scoring (see below) — notably `containsAny` is available, `notExists` is not.
- **First matching override wins** (array order). Put the most severe / most specific first.

---

## Scoring math & risk levels

How a single factor produces a score (`calculateFactorScores` / `evaluatePipeline`, `risk-formula.service.ts:956+`):

1. **Resolve the source value** at `sourceField` from `entityData` (calculated fields are computed on the fly — see catalog).
2. **Max-score-wins at three levels:**
   - **Multi-source** (`sourceField` is an array): score each populated source; the **highest** matched score wins.
   - **Array value** (the field holds an array, e.g. `members.memberJurisdictions`): score each element; the **highest** element score wins.
   - **Multiple matching conditions**: when several `scoreConditions` match one value, the **highest** `score` wins.
3. **No match anywhere** → `defaultScore` (or `0`).
4. **Clamp** the result to `[minScore, maxScore]`.

Then across the whole formula (`runAssessment`, `:733+`):

```
totalScore = Σ factor.score            // plain sum, no weighting
```

Band selection (`determineRiskLevel`, `:1545`):

```
if (totalScore >= thresholds.high.min)        → "HIGH"
else if (totalScore >= thresholds.medium.min) → "MEDIUM"
else                                          → "LOW"
```

- `>=` is inclusive; ties resolve **upward** (a score exactly at `high.min` is `HIGH`).
- `low.min`, `low.max`, `medium.max`, `high.max` are **not read** at scoring time — only `high.min` and `medium.min` matter. Anything below `medium.min` is `LOW`.
- Score range is **0–100** by construction: each factor is clamped to its `maxScore`, and the sum of `maxScore`s is validated ≤ 100, so the total can never exceed 100.

Override and rules layers:

- `checkOverrides` runs **after** banding; the first fully-matching override sets `effectiveRiskLevel`. The stored `riskLevel` remains the model band.
- `runRulesEngine` runs last, purely for recommendations/`triggeredRules`. If a rule throws, it's logged as a warning and the assessment still succeeds with empty events. **Rules never change scores.**

---

## Operators reference

### Scoring operators (score conditions) — 10

Used inside `pipeline.scoreConditions[].conditions.operator` (`risk-formula.service.ts:1248`):

| Operator | Semantics |
|----------|-----------|
| `equals` | Type-aware equality (date day-key, boolean, numeric coercion, else strict `===`). |
| `contains` | Array-contains (element/intersection) or substring for strings. |
| `in` | `value` **must be an array**; true if the field value (or any array element) is in it. |
| `greaterThan` | Numeric/date `>`. |
| `lessThan` | Numeric/date `<`. |
| `greaterThanOrEqual` | Numeric/date `>=`. |
| `lessThanOrEqual` | Numeric/date `<=`. |
| `between` | Inclusive range using **`min`/`max`** (numbers, or `YYYY-MM-DD` for dates). |
| `exists` | Value is not `null`/`undefined`. |
| `notExists` | Value is `null`/`undefined`. |

### Override operators — 10 (a *different* set)

Used in `overrideConditions` (`OVERRIDE_OPERATORS`, `risk-formula.entity.ts:41`):

`equals`, `in`, `contains`, **`containsAny`**, `greaterThan`, `greaterThanOrEqual`, `lessThan`, `lessThanOrEqual`, `between`, `exists`.

> ⚠️ The two sets diverge: scoring has **`notExists`** (overrides don't); overrides have **`containsAny`** (scoring doesn't). Don't copy an operator from one layer to the other without checking this table.

### Allowed operators per field value type (enforced on activate/deploy)

Registry validation (`getAllowedOperators`, `risk-factor-field.registry.ts:646`) restricts which operators a factor may use given the field's `valueType`. A factor that violates this passes `create` but **fails `activate`/`deploy`**:

| `valueType` | Allowed operators |
|-------------|-------------------|
| `number` | `equals`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `between`, `exists`, `notExists` |
| `boolean` | `equals` **only** |
| `selection` (single) | `equals`, `in`, `exists`, `notExists` |
| `selection` (multiple) | `contains`, `in`, `exists`, `notExists` |
| `date` | `equals`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `between`, `exists`, `notExists` |
| `custom` | all 10 scoring operators |

Custom-field-type operator sets (`getAllowedOperatorsForCustomFieldType`, `:668`): `text`/`country_list` → `equals`,`contains`,`in`,`exists`,`notExists`; `boolean` → `equals`; `number`/`date` → comparison set as above.

> A single-select `selection` field does **not** allow `contains`; a multi-select does **not** allow `equals`. A `boolean` field allows only `equals` (use `equals: false`, not `notExists`).

---

## Factor field catalog

These are the **referenceable `sourceField` dot-paths** per entity type (`risk-factor-field.registry.ts`). Fetch them live per entity type with `GET /v1/risk/factor-fields?entityType=...` before authoring — the registry is the source of truth and may grow. `TRANSACTION` and `BLOCKCHAIN_WALLET` return **no fields**.

**Calculated fields** (⚙) are computed at scoring time from a date/relation dependency; reference them directly.

### CORPORATE_CLIENT fields

| Path | Value type | Notes / options |
|------|-----------|-----------------|
| `business.incorporationType` | selection (single) | `CORPORATION`, `LIMITED_LIABILITY_COMPANY`, `LIMITED_LIABILITY_PARTNERSHIP`, `LIMITED_PARTNERSHIP`, `GENERAL_PARTNERSHIP`, `SOLE_PROPRIETORSHIP`, `TRUST`, `UNINCORPORATED_ASSOCIATION`, `NOT_FOR_PROFIT_CORPORATION_ORGANIZATION`, `DECENTRALIZED_AUTONOMOUS_ORGANIZATION` |
| `business.ownershipType` | selection (single) | `PUBLICLY_TRADED`, `PRIVATELY_HELD` |
| `business.ownershipComplexity` | selection (single) | `SIMPLE_OWNERSHIP`, `COMPLEX_OWNERSHIP` |
| `business.businessType` | selection (single) | 13 industries: `ASSET_MANAGER`, `BANK`, `BLOCKCHAIN_NATIVE`, `COMMODITIES`, `EXCHANGE`, `FINANCIAL_INSTITUTION`, `FMI`, `GOVERNMENT`, `INSTITUTIONAL_TRADING`, `NON_FINANCIAL_CORPORATE`, `PAYMENTS`, `RETAIL_TRADING`, `WEALTH_MANAGER` |
| `business.businessSubType` | selection (single) | 55 values (e.g. `US_CRYPTO_EXCHANGE`, `FOREIGN_CRYPTO_EXCHANGE`, `DEFI_EXCHANGE`, `NFT_MARKETPLACE`, `VENTURE_CAPITAL_FIRM`, `FAMILY_OFFICE`, …). Fetch the full list from the API. |
| `general.timeSinceIncorporation` ⚙ | number | Decimal years since `general.dateOfIncorporation`. |
| `general.dateOfIncorporation` | date | |
| `general.jurisdiction` | selection (single) | country codes |
| `general.countryOfIncorporation` | selection (single) | country codes |
| `application.relationshipAge` ⚙ | number | Decimal years since `application.onboardedAt`. |
| `application.onboardedAt` | date | |
| `sourceOfFundsInfo.sourceOfFunds` | selection (single) | `OPERATING_FUNDS`, `INVESTOR_FUNDS`, `PROPRIETARY_FUNDS`, `CUSTOMER_FUNDS`, `TOKEN_GENERATION_EVENT`, `OTHER` |
| `sourceOfFundsInfo.sourceOfFundsJurisdictions` | selection (multiple) | country codes |
| `sourceOfFundsInfo.monthlyNetIncome` | number | |
| `sourceOfFundsInfo.monthlyTransactionVolume` | number | |
| `sourceOfFundsInfo.annualTransactionVolume` | number | |
| `members.memberJurisdictions` ⚙ | selection (multiple) | Countries across all beneficial owners/members. Array value → max-score-wins per element. |
| `address.registrationAddress.country` | selection (single) | country codes |
| `address.businessAddress.country` | selection (single) | country codes |
| `adverseMedia.adverseMediaRecency` ⚙ | number | Decimal years since `adverseMedia.adverseMediaDate`. |
| `adverseMedia.adverseMediaDate` | date | |
| `adverseMedia.adverseMediaRiskLevel` | selection (single) | `NONE`, `LOW`, `MEDIUM`, `HIGH` |
| `adverseMedia.adverseMediaScore` | number | |
| `adverseMedia.adverseMediaResolved` | boolean | |
| `adverseMedia.adverseMediaFines` | number | |
| `politicalExposure.isPoliticallyExposed` | boolean | |
| `politicalExposure.pepTier` | selection (single) | `NO_PEP`, `TIER_1`, `TIER_2`, `TIER_3`, `TIER_4` |
| `politicalExposure.pepJurisdiction` | selection (multiple) | country codes |
| `politicalExposure.pepScore` | number | |
| `currentRisk.level` | selection (single) | risk levels |
| `currentRisk.score` | number | |
| `sanctions.isSanctioned` | boolean | |
| `screening.hasSubpoena` | boolean | |
| `screening.hasSAR` | boolean | |
| `tags` | selection (multiple) | free tag set (no fixed options) |

### INDIVIDUAL_CLIENT fields

| Path | Value type | Notes / options |
|------|-----------|-----------------|
| `general.citizenship` | selection (single) | country codes |
| `general.jurisdiction` | selection (single) | country codes |
| `general.age` ⚙ | number | Decimal years since `general.dateOfBirth`. |
| `address.country` | selection (single) | country codes |
| `financial.monthlyNetIncome` | number | |
| `financial.expectedMonthlyTransactionVolume` | number | |
| `work.occupation` | selection (single) | 21 values: `Lawyer`, `Notary`, `Accountant`, `Auditor`, `Merchant`, `Investor`, `Independent professional`, `Real estate`, `Construction`, `Virtual Asset Service Providers`, `VASP`, `Student`, `Retired`, `Real estate developer`, `Consultant`, `Retired / unemployed`, `Business owner`, `Public employee`, `Banker`, `Employee`, `Other` |
| `application.relationshipAge` ⚙ | number | Decimal years since onboarding. |
| `politicalExposure.isPoliticallyExposed` | boolean | |
| `politicalExposure.pepTier` | selection (single) | `NO_PEP`, `TIER_1`, `TIER_2`, `TIER_3`, `TIER_4` |
| `politicalExposure.pepJurisdiction` | selection (multiple) | country codes |
| `adverseMedia.isAdverseMedia` | boolean | |
| `adverseMedia.adverseMediaRecency` ⚙ | number | Decimal years since most recent finding. |
| `adverseMedia.adverseMediaRiskLevel` | selection (single) | `NONE`, `LOW`, `MEDIUM`, `HIGH` |
| `sourceOfFundsInfo.sourceOfFunds` | selection (single) | same 6 values as corporate |
| `sourceOfFundsInfo.sourceOfFundsJurisdictions` | selection (multiple) | country codes |
| `currentRisk.level` | selection (single) | risk levels |
| `currentRisk.score` | number | |
| `sanctions.isSanctioned` | boolean | |
| `tags` | selection (multiple) | free tag set |

> Country-code selection fields expect **ISO codes** (the built-in examples and templates use **ISO3**, e.g. `USA`, `IRN`, `PRK`). Be consistent with whatever your platform's entity snapshots store.

---

## Condition templates & country lists

Instead of hand-listing every country, use the built-in **condition templates** — pre-authored regulatory `scoreConditions` you can drop into a factor's pipeline. Fetch them with `GET /v1/risk/factor-condition-templates?entityType=...` (`risk-factor-condition-template.registry.ts`).

| Template id | Label | Category | Score mapping (`operator: in`) | Supported source field |
|-------------|-------|----------|-------------------------------|------------------------|
| `cpi_country_risk` | Corruption Perceptions Index (2025) | `countryRisk` | low-risk tier → **1.3**, medium → **2**, high → **2** (`defaultScore 2`) | corporate `general.countryOfIncorporation` / individual `address.country` |
| `basel_aml_country_risk` | Basel AML Index (2025) | `countryRisk` | low → **0.7**, medium → **1.3**, high → **2** (`defaultScore 2`) | same |

Each template's `scoreConditions` are `{ operator: "in", value: [ISO3…] }` arrays already populated with the tier membership — copy the `scoreConditions` (and `defaultScore`) straight into your factor's `pipeline`. (Note: for CPI, low and medium/high map to 1.3 vs 2, so it effectively distinguishes low-risk from the rest.)

If you prefer to hand-roll country factors, the service ships curated ISO3 lists in `src/risk-rating/utils/country-risk-data.ts` (referenced by the built-in Fireblocks model), including:

- `HIGH_JURISDICTION_COUNTRY_CODES` — `BLR, CUB, IRN, MMR, PRK, RUS`
- `FATF_GREY_JURISDICTION_COUNTRY_CODES` — 22 FATF grey-list countries
- `LOW_JURISDICTION_COUNTRY_CODES` — US, Canada, UK, Australia, NZ, Japan, Singapore, HK, Korea, Switzerland, Norway, Iceland + 27 EU
- `*_JURISDICTION_COUNTRY_CODE_ALIASES` — Fireblocks 2026 low/medium/high buckets (also match ISO2 + names)

These are server constants, not API inputs — to use them in your own model, inline the concrete ISO3 arrays you need into `scoreConditions[].conditions.value`.

---

## Custom fields

Reference a platform-defined custom field at path **`customFields.<key>.value`** and declare a `customFieldMapping` on the pipeline (`custom-risk-factor-field-types.ts`, `risk-factor-field.registry.ts:576`):

```json
{
  "name": "productRisk",
  "description": "Risk by product/service type",
  "minScore": 0,
  "maxScore": 10,
  "calculationMethod": "pipeline",
  "pipeline": {
    "sourceField": "customFields.product_service_type.value",
    "customFieldMapping": { "key": "product_service_type", "fieldType": "text" },
    "scoreConditions": [
      { "conditions": { "operator": "in", "value": ["Cross-border payments","Digital asset custody"] }, "score": 10 }
    ],
    "defaultScore": 3
  }
}
```

Rules (enforced on activate/deploy):

- `fieldType` ∈ `text` \| `boolean` \| `country_list` \| `number` \| `date`.
- `key` must match `/^[A-Za-z][A-Za-z0-9_]*$/`.
- The pipeline must use **exactly one** source field, and it must equal `customFields.<key>.value`.
- Operators allowed follow the custom-field-type table above (`text`/`country_list` → equals/contains/in/exists/notExists; `number`/`date` → comparisons; `boolean` → equals).

---

## Lifecycle, versioning & active-formula resolution

```
create ──► DRAFT ──activate──► ACTIVE ──deactivate──► INACTIVE
              │                   │
              │                   └──deploy──► new ACTIVE version (old → INACTIVE)
              └──duplicate──► new DRAFT (fresh formulaGroupId)
                              delete ──► soft-deleted (deletedAt set)
```

| Operation | Endpoint | Effect |
|-----------|----------|--------|
| **Create** | `POST /v1/risk/formulas` | New `DRAFT`, fresh `formulaGroupId`, `version: null`. |
| **Update** | `PUT /v1/risk/formulas/:id` | Edits a `DRAFT`/`INACTIVE` in place, no version bump. On an `ACTIVE` formula, **only `name`/`description`** may change — structural edits must go through **deploy**. |
| **Activate** | `POST /v1/risk/formulas/:id/activate` | Flips a formula to `ACTIVE`. Only the **latest** version in the group can be activated. Fails with `409` if a *different* formula is already `ACTIVE` for that `entityType`. |
| **Deploy** | `POST /v1/risk/formulas/:id/deploy` | The versioning path for a live model: validates, **deactivates** the current active row, creates a **new `ACTIVE` row** with an incremented `version` and `basedOnFormulaId` lineage, writes a `DEPLOYED` history event. |
| **Duplicate** | `POST /v1/risk/formulas/:id/duplicate` | New `DRAFT` with a **fresh `formulaGroupId`** and regenerated factor ids. |
| **Deactivate** | `POST /v1/risk/formulas/:id/deactivate` | `ACTIVE` → `INACTIVE`. |
| **Delete** | `DELETE /v1/risk/formulas/:id` | **Soft delete** (`deletedAt`), preserving historical assessments. Deactivates first. |

Key rules:

- **One `ACTIVE` per `platformId` + `entityType`** — enforced at three layers (activate pre-check, deploy's guarded swap, and a DB partial-unique index). Activating/deploying a second formula for the same entity type either swaps it (deploy) or `409`s (activate a different one).
- **Optimistic concurrency on deploy**: pass `expectedVersion`; if the active formula has moved past it, you get `409 Conflict`.
- **`version`** starts at `"0.1"` and bumps the **minor** each deploy/activate (`"0.1"` → `"0.2"` → … `"1.0"`).
- **`rescorePolicy`** on deploy (`DeployRiskFormulaRescorePolicy`): `RESCORE_EXISTING_CLIENTS` enqueues a portfolio rescore of in-scope clients; `FUTURE_CHANGES_ONLY` (**default**) applies only to new assessments.

**Active-formula resolution** — `getActiveFormula` (`risk-formula.service.ts:663`) / `GET /v1/risk/formulas/active`:

```
findOne({ platformId, status: ACTIVE, entityType, deletedAt: null })
```

`platformId` always comes from the authenticated context, never from the request. If you call `/formulas/active` without `entityType` and multiple active formulas exist, you get a `400` telling you to specify one.

---

## REST v1 endpoints

All routes are served under **`/v1/risk`** (controller `@Controller({ path: "risk", version: "1" })`, URI versioning, **no** `corporate-clients` segment) and **`/v1/risk/simulation`**. Every route requires a **bearer token** (global auth guard); `platformId` is derived from that token's context — never sent in params or body.

### `risk` controller (`risk-rating.controller.ts`)

| Method | Path | Body / Query | Returns |
|--------|------|--------------|---------|
| `POST` | `/v1/risk/formulas` | `CreateRiskFormulaDto` | `RiskFormulaDto` (201) |
| `GET` | `/v1/risk/formulas` | `?status=&entityType=` | `RiskFormulaDto[]` |
| `GET` | `/v1/risk/formulas/paginated` | `?status=&entityType=&search=&page=&pageSize=` | `PaginatedRiskFormulasDto` |
| `GET` | `/v1/risk/formulas/counts` | — | `RiskFormulaCountsDto` |
| `GET` | `/v1/risk/formulas/active` | `?entityType=` | `RiskFormulaDto` (404 if none) |
| `GET` | `/v1/risk/formulas/:formulaGroupId/history` | `?page=&pageSize=` | `PaginatedRiskFormulaHistoryDto` |
| `GET` | `/v1/risk/formulas/:id` | — | `RiskFormulaDto` |
| `PUT` | `/v1/risk/formulas/:id` | `UpdateRiskFormulaDto` | `RiskFormulaDto` |
| `POST` | `/v1/risk/formulas/:id/deploy` | `DeployRiskFormulaDto` | `RiskFormulaDto` (200) |
| `POST` | `/v1/risk/formulas/:id/duplicate` | `DuplicateRiskFormulaDto` | `RiskFormulaDto` (201) |
| `POST` | `/v1/risk/formulas/:id/activate` | — | `RiskFormulaDto` (200) |
| `POST` | `/v1/risk/formulas/:id/deactivate` | — | `RiskFormulaDto` (200) |
| `DELETE` | `/v1/risk/formulas/:id` | — | `204 No Content` |
| `POST` | `/v1/risk/assess` | `RunRiskAssessmentDto` + `?dryRun=` | `RiskAssessmentResultDto` |
| `POST` | `/v1/risk/assess/:entityType/:entityId` | `?formulaId=` (**required**) `&dryRun=` | `RiskAssessmentResultDto` |
| `POST` | `/v1/risk/assess/batch` | `BatchRiskAssessmentDto` | `BatchRiskAssessmentResultDto` |
| `GET` | `/v1/risk/factor-fields` | `?entityType=` (**required**) | `RiskFactorFieldsResponseDto` |
| `GET` | `/v1/risk/factor-condition-templates` | `?entityType=` | `RiskFactorConditionTemplatesResponseDto` |
| `GET` | `/v1/risk/formulas/examples/fireblocks` | — | `CreateRiskFormulaDto` |
| `GET` | `/v1/risk/formulas/examples/monetae` | `?entityType=` | `CreateRiskFormulaDto` |
| `GET` | `/v1/risk/assessments/history/:entityId` | `?entityType=&limit=&offset=&riskLevel=&formulaId=&fromDate=&toDate=` | `RiskAssessmentHistoryDto[]` |
| `GET` | `/v1/risk/assessments/comparison/:entityId` | `?entityType=` | `AssessmentComparisonSummaryDto \| null` |
| `PUT` | `/v1/risk/assessments/:assessmentId/notes` | `UpdateAssessmentNotesDto` | `RiskAssessmentHistoryDto` |

> The `:entityType` path param on `assess/:entityType/:entityId` is **lowercase** (`corporate_client`, `individual_client`, `transaction`, `blockchain_wallet`) — unlike the UPPER `EntityType` enum used in bodies/queries.

### `risk/simulation` controller (`risk-formula-simulation.controller.ts`)

| Method | Path | Body / Query | Returns |
|--------|------|--------------|---------|
| `POST` | `/v1/risk/simulation/runs` | `TriggerSimulationRunDto` | `SimulationRunDto` |
| `GET` | `/v1/risk/simulation/runs` | `?formulaGroupId=` | `SimulationRunDto[]` |
| `PUT` | `/v1/risk/simulation/runs/:runId` | `TriggerSimulationRunDto` | `SimulationRunDto` (rerun in place) |
| `PATCH` | `/v1/risk/simulation/runs/:runId/name` | `RenameSimulationRunDto` | `SimulationRunDto` |
| `GET` | `/v1/risk/simulation/runs/:runId` | — | `SimulationRunDto` |
| `GET` | `/v1/risk/simulation/runs/:runId/clients` | `?page=&pageSize=&bandChangedOnly=&projectedBand=` | `PaginatedSimulationRunClientsDto` |

---

## SDK client

`@corsa-labs/risk-rating-service-rest-client` — a generated client (1:1 with the routes above). Every method takes an optional trailing `context?` (Tweed context propagation) and returns `Promise<AxiosResponse<T>>`.

```typescript
import { RiskRatingServiceClient } from "@corsa-labs/risk-rating-service-rest-client";

const client = new RiskRatingServiceClient({
  BASE: config.integrations.riskRatingServiceUrl,
  TOKEN: async () => getBearerToken(),   // or a string
});

// Discover fields & templates for the entity type first
const { data: fields }    = await client.riskRating.getRiskFactorFields("INDIVIDUAL_CLIENT");
const { data: templates } = await client.riskRating.getRiskFactorConditionTemplates("INDIVIDUAL_CLIENT");

// Author → simulate → activate
const { data: draft } = await client.riskRating.createRiskFormula(myFormula);
const { data: run }   = await client.riskRatingSimulation.triggerSimulationRun({
  name: "pre-activation check", formulaGroupId: draft.formulaGroupId, formulaId: draft.id,
  datasetType: "FULL_PORTFOLIO", sampling: 2000,
});
// …poll getSimulationRun(run.id) until COMPLETED, inspect aggregates…
await client.riskRating.activateRiskFormula(draft.id);
```

Sub-services: **`client.riskRating`**, **`client.riskRatingSimulation`**, **`client.health`**.

Selected `client.riskRating` methods:

| Method | Signature |
|--------|-----------|
| `createRiskFormula` | `(requestBody: CreateRiskFormulaDto, context?)` → `RiskFormulaDto` |
| `updateRiskFormula` | `(id, requestBody: UpdateRiskFormulaDto, context?)` → `RiskFormulaDto` |
| `deployRiskFormula` | `(id, requestBody: DeployRiskFormulaDto, context?)` → `RiskFormulaDto` |
| `duplicateRiskFormula` | `(id, requestBody: DuplicateRiskFormulaDto, context?)` → `RiskFormulaDto` |
| `activateRiskFormula` | `(id, context?)` → `RiskFormulaDto` |
| `deactivateRiskFormula` | `(id, context?)` → `RiskFormulaDto` |
| `deleteRiskFormula` | `(id, context?)` → `void` |
| `getActiveRiskFormula` | `(entityType?, context?)` → `RiskFormulaDto` |
| `getRiskFormulas` | `(status?, entityType?, context?)` → `RiskFormulaDto[]` |
| `getRiskFormulasPaginated` | `(status?, entityType?, search?, page=1, pageSize=100, context?)` |
| `getRiskFormulaHistory` | `(formulaGroupId, status?, entityType?, search?, page, pageSize, context?)` |
| `getRiskFactorFields` | `(entityType, context?)` → `RiskFactorFieldsResponseDto` |
| `getRiskFactorConditionTemplates` | `(entityType?, context?)` → `RiskFactorConditionTemplatesResponseDto` |
| `runRiskAssessment` | `(requestBody: RunRiskAssessmentDto, dryRun?, context?)` → `RiskAssessmentResultDto` |
| `runRiskAssessmentByEntityId` | `(entityType, entityId, formulaId, dryRun?, context?)` → `RiskAssessmentResultDto` |
| `runBatchRiskAssessment` | `(requestBody: BatchRiskAssessmentDto, context?)` → `BatchRiskAssessmentResultDto` |

`client.riskRatingSimulation`: `triggerSimulationRun`, `listSimulationRuns(formulaGroupId)`, `rerunSimulationRun(runId, body)`, `getSimulationRun(runId)`, `renameSimulationRun(runId, body)`, `getSimulationRunClients(runId, page=1, pageSize=25, bandChangedOnly?, projectedBand?)`.

---

## Simulation: test before activation

**Always simulate a draft against the real portfolio before you activate it.** Simulation reuses the exact production scoring engine (`evaluateFormulaForSimulation`) but is completely **non-mutating** — it reads each client's current risk as the "before", computes the draft's projection as the "after", and writes results only into simulation tables. Activating anything is a separate, deliberate call.

### Trigger — `TriggerSimulationRunDto` (`risk-formula-simulation.dto.ts:101`)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | `string` | ✅ | Max 120 chars. |
| `formulaGroupId` | `string` | ✅ | The draft's group. |
| `formulaId` | `string` (UUID) | ✅ | The persisted formula anchoring the run. |
| `datasetType` | `"FULL_PORTFOLIO" \| "CLIENT_LIST"` | — | Default `FULL_PORTFOLIO`. |
| `clientIds` | `string[]` | — (required for `CLIENT_LIST`) | Max **1,000** (`SIMULATION_CLIENT_LIST_MAX`). |
| `creationDateFilter` | `{ from?, to? }` (ISO-8601) | — | Restrict portfolio by client creation date. |
| `sampling` | `number` | — | Most-recent-N clients to score, `0–10,000` (`MAX_SIMULATION_SAMPLING_COUNT`), default `1000`. |
| `formulaOverride` | `FormulaOverrideDto` | — | Simulate an **unsaved** candidate without persisting it. `factors`, `thresholds`, `overrideConditions` are passed as **JSON-encoded strings** (GraphQL Mesh compatibility). |

```json
{
  "name": "Q3 tightening test",
  "formulaGroupId": "grp-abc",
  "formulaId": "0f3c…",
  "datasetType": "FULL_PORTFOLIO",
  "sampling": 2000,
  "creationDateFilter": { "from": "2025-01-01T00:00:00.000Z" }
}
```

The run executes asynchronously (`PENDING` → `RUNNING` → `COMPLETED`/`FAILED`). **Poll `GET /runs/:runId`** until the status is terminal.

### Reading results — `SimulationRunDto`

- **`aggregates`** (`SimulationAggregatesDto`): `evaluated`, `unscorable` (clients that errored), `movedHigher`/`movedLower`, `totalBandChanges`, `publishedDistribution` vs `draftDistribution` (`{LOW,MEDIUM,HIGH}` counts), `transitionMatrix` (before→after grid), `scoreHistogram` (20 buckets of width 5).
- **Per-client** via `GET /runs/:runId/clients` (`SimulationRunClientDto`): `previousScore`/`projectedScore`, `scoreDelta`, `previousBand`/`projectedBand`, `bandChanged`, `topFactorDeltas`. Filter with `bandChangedOnly=true` or `projectedBand=`.
- **Staleness flags** (computed from the scoring **signature**, so re-publishing the exact tested changes still counts as a match):
  - `isStale` — the latest/draft formula's scoring content changed since this run; re-run.
  - `baseVersionSuperseded` — a different formula was activated since; the baseline moved.
  - `matchesActiveVersion` — the simulated config equals the currently active formula.

### Workflow

1. `create` / `update` a **DRAFT** (or edit an existing group's draft).
2. `POST /simulation/runs` against it — `FULL_PORTFOLIO` (optionally sampled/date-filtered) or a `CLIENT_LIST` of known ids. Or pass `formulaOverride` to test a fully unsaved candidate.
3. Poll `GET /runs/:runId`; inspect `aggregates` (how many clients move band, in which direction) and drill into impacted clients.
4. Iterate: edit the draft, `PUT /runs/:runId` to rerun in place. Watch `isStale`/`baseVersionSuperseded`.
5. When satisfied, **`deploy`** (to push changes onto the live model with a new version) or **`activate`** (to make this the group's active formula).

---

## Running assessments

### By entity data — `POST /v1/risk/assess`

`RunRiskAssessmentDto` (`risk-assessment.dto.ts:21`):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `formulaId` | `string` (UUID) | ✅ | Must reference a formula that is currently `ACTIVE` on your platform. |
| `entityType` | `EntityType` | ✅ | Must match the formula's `entityType`. |
| `entityData` | `object` | ✅ | The full entity snapshot (from compliance-entity-service). **The field is `entityData`, not `clientData`.** |
| `context` | `object` | — | Optional derived-fact overrides. |
| `dryRun` | `boolean` | — | Default `false`. `true` computes without persisting or applying to the entity. |

### By entity id — `POST /v1/risk/assess/:entityType/:entityId?formulaId=…`

Fetches the entity snapshot from compliance-entity-service by id, then scores it. **`formulaId` is a required query param** — this path does **not** auto-resolve the active formula. To score by id with the active model, first call `GET /v1/risk/formulas/active?entityType=…`, then pass that `id`.

### Batch — `POST /v1/risk/assess/batch`

`BatchRiskAssessmentDto`: `{ formulaId, entityType, entities: object[] }` → `BatchRiskAssessmentResultDto` with `results[]`, `errors[]`, and `statistics` (average score, band counts, overrides applied).

### Result — `RiskAssessmentResultDto` (`risk-assessment.dto.ts:61`)

| Field | Notes |
|-------|-------|
| `assessmentId`, `clientId` | Present only on persisted (non-dry-run) assessments. |
| `formulaId`, `formulaName` | Which formula scored. |
| `totalScore` | 0–100 sum. |
| `riskLevel` | **Model** band (`LOW`/`MEDIUM`/`HIGH`). |
| `effectiveRiskLevel` | After overrides; equals `riskLevel` when no override applied. |
| `factors[]` | Per-factor `AssessmentFactorResult`: `name`, `description`, `groupName?`, `score`, `maxScore`, `percentage`, `details?`. |
| `triggeredRules[]` | Rules-engine events (recommendations). |
| `overrideApplied` | `{ applied, level?, reason? }`. |
| `breakdown[]`, `recommendations[]`, `riskExplanation` | Human-readable explanation. |
| `assessedAt`, `executionTime` | Timing. |

> There is **no top-level `maxPossibleScore`** field — it's folded into `riskExplanation`; per-factor caps are on `factors[].maxScore`.

### Auto-apply to the entity's current risk

After a non-dry-run assessment, `RiskApplicationService.applyModelRiskIfCurrent` may write the result back to the entity's `currentRisk` (via a dedicated `/internal/current-risk/model` endpoint). It is **gated and safe**:

- **Never clobbers a manual override**: if the entity's current risk source is `MANUAL_OVERRIDE`, the write is skipped — regardless of anything else.
- **Fail-closed feature gate**: automatic writes happen only when the platform config has `autoRiskRating === true`. Deploy/activation portfolio rescores use an explicit opt-in that bypasses the gate.
- **No-op suppression**: unchanged level+score → skip.
- **Never rolls back the assessment**: apply failures are logged, not thrown.
- `dryRun: true` skips persistence and apply entirely — use it to preview a score.

---

## Validation & limits

Enforced across create/update/activate/deploy (`risk-formula.service.ts`):

**Structural (`validateFormulaStructure`, every write):**

- Thresholds **must not overlap**: `low.max < medium.min` and `medium.max < high.min` (`400` "Risk thresholds must not overlap").
- Sum of factor `maxScore` **≤ 100** (`400` "…maximum score must not exceed 100").
- Each factor: `minScore < maxScore` (`400` "Invalid score range").
- Each factor: `calculationMethod === 'pipeline'` (`400` "Unsupported calculation method").
- Each factor: non-empty `pipeline.sourceField` and non-empty `scoreConditions` (`400` "…required").
- Custom field mapping: valid key, **exactly one** source field equal to `customFields.<key>.value`.

**Registry-aware (`validateFactorsAgainstRegistry`, only on activate & deploy):**

- Every `sourceField` must be a **registered** path for the `entityType` (or a `customFields.*` path with a mapping).
- Multi-source fields must be **compatible** (same `valueType` + option source); custom paths can't mix with registered ones.
- Every operator must be **allowed** for the field's value type (see the [operator table](#operators-reference)).

**Numeric limits:**

| Limit | Value |
|-------|-------|
| Total formula `maxScore` sum | ≤ **100** |
| Formula list `pageSize` | default 100, max **200** |
| Simulation `clientIds` (CLIENT_LIST) | max **1,000** |
| Simulation `sampling` | `0`–**10,000**, default 1,000 |
| Simulation run `name` | ≤ **120** chars |
| Simulation clients `pageSize` | default 25, max **200** |

> A factor that uses an operator disallowed for its field type will **create fine but fail to activate**. Validate against the registry (or just try activating in a dev environment) before shipping.

---

## Worked example — individual-client KYC model

A complete, self-consistent `CreateRiskFormulaDto`. Seven factors summing to exactly **100**, non-overlapping bands, a sanctions override to `HIGH`, empty `rules`. Copy, adjust the country lists to your policy, POST to `/v1/risk/formulas`.

```json
{
  "name": "Individual KYC Risk Model",
  "description": "Onboarding risk score for individual clients",
  "entityType": "INDIVIDUAL_CLIENT",
  "factors": [
    {
      "name": "geographicRisk",
      "description": "Country of residence risk",
      "minScore": 0, "maxScore": 25, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "address.country",
        "scoreConditions": [
          { "conditions": { "operator": "in", "value": ["IRN","PRK","MMR","SYR","CUB"] }, "score": 25 },
          { "conditions": { "operator": "in", "value": ["USA","GBR","CAN","DEU","FRA","AUS","SGP","JPN","CHE"] }, "score": 0 }
        ],
        "defaultScore": 12
      }
    },
    {
      "name": "pepRisk",
      "description": "Political exposure by PEP tier",
      "minScore": 0, "maxScore": 20, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "politicalExposure.pepTier",
        "scoreConditions": [
          { "conditions": { "operator": "in", "value": ["TIER_1"] }, "score": 20 },
          { "conditions": { "operator": "in", "value": ["TIER_2"] }, "score": 15 },
          { "conditions": { "operator": "in", "value": ["TIER_3"] }, "score": 10 },
          { "conditions": { "operator": "in", "value": ["TIER_4"] }, "score": 5 },
          { "conditions": { "operator": "in", "value": ["NO_PEP"] }, "score": 0 }
        ],
        "defaultScore": 0
      }
    },
    {
      "name": "occupationRisk",
      "description": "Occupation / economic activity risk",
      "minScore": 0, "maxScore": 15, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "work.occupation",
        "scoreConditions": [
          { "conditions": { "operator": "in", "value": ["Virtual Asset Service Providers","VASP","Real estate developer"] }, "score": 15 },
          { "conditions": { "operator": "in", "value": ["Lawyer","Notary","Accountant","Auditor","Real estate"] }, "score": 10 },
          { "conditions": { "operator": "in", "value": ["Employee","Public employee","Student","Retired"] }, "score": 3 }
        ],
        "defaultScore": 7
      }
    },
    {
      "name": "adverseMedia",
      "description": "Adverse media severity",
      "minScore": 0, "maxScore": 15, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "adverseMedia.adverseMediaRiskLevel",
        "scoreConditions": [
          { "conditions": { "operator": "equals", "value": "HIGH" }, "score": 15 },
          { "conditions": { "operator": "equals", "value": "MEDIUM" }, "score": 10 },
          { "conditions": { "operator": "equals", "value": "LOW" }, "score": 5 },
          { "conditions": { "operator": "equals", "value": "NONE" }, "score": 0 }
        ],
        "defaultScore": 0
      }
    },
    {
      "name": "sourceOfFunds",
      "description": "Source of funds risk",
      "minScore": 0, "maxScore": 10, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "sourceOfFundsInfo.sourceOfFunds",
        "scoreConditions": [
          { "conditions": { "operator": "in", "value": ["OTHER","TOKEN_GENERATION_EVENT"] }, "score": 10 },
          { "conditions": { "operator": "in", "value": ["INVESTOR_FUNDS","PROPRIETARY_FUNDS"] }, "score": 6 },
          { "conditions": { "operator": "in", "value": ["OPERATING_FUNDS"] }, "score": 2 }
        ],
        "defaultScore": 5
      }
    },
    {
      "name": "expectedVolume",
      "description": "Expected monthly transaction volume",
      "minScore": 0, "maxScore": 10, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "financial.expectedMonthlyTransactionVolume",
        "scoreConditions": [
          { "conditions": { "operator": "greaterThanOrEqual", "value": 100000 }, "score": 10 },
          { "conditions": { "operator": "between", "min": 25000, "max": 99999 }, "score": 6 },
          { "conditions": { "operator": "between", "min": 5000, "max": 24999 }, "score": 3 },
          { "conditions": { "operator": "lessThan", "value": 5000 }, "score": 1 }
        ],
        "defaultScore": 1
      }
    },
    {
      "name": "sanctions",
      "description": "Sanctions screening flag",
      "minScore": 0, "maxScore": 5, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "sanctions.isSanctioned",
        "scoreConditions": [
          { "conditions": { "operator": "equals", "value": true }, "score": 5 },
          { "conditions": { "operator": "equals", "value": false }, "score": 0 }
        ],
        "defaultScore": 0
      }
    }
  ],
  "thresholds": {
    "low":    { "min": 0,  "max": 33 },
    "medium": { "min": 34, "max": 66 },
    "high":   { "min": 67, "max": 100 }
  },
  "overrideConditions": [
    { "field": "sanctions.isSanctioned", "operator": "equals", "value": true,
      "overrideLevel": "HIGH", "reason": "Sanctioned individual — mandatory high risk" }
  ],
  "rules": []
}
```

Self-consistency check: `25+20+15+15+10+10+5 = 100`; bands `0–33 / 34–66 / 67–100` don't overlap. ✔

---

## Worked example — corporate-client KYB model

Eight factors summing to **100**, including a **multi-source** geography factor (max-score-wins across three country fields) and both a single-field and a compound override. Modeled on the built-in Fireblocks/Monetae corporate models but trimmed to a clean, copyable payload.

```json
{
  "name": "Corporate KYB Risk Model",
  "description": "Onboarding risk score for corporate clients",
  "entityType": "CORPORATE_CLIENT",
  "factors": [
    {
      "name": "entityGeography",
      "description": "Highest-risk country across incorporation & addresses",
      "minScore": 5, "maxScore": 20, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": ["general.countryOfIncorporation","address.registrationAddress.country","address.businessAddress.country"],
        "scoreConditions": [
          { "conditions": { "operator": "in", "value": ["IRN","PRK","MMR","SYR","RUS","CUB","BLR"] }, "score": 20 },
          { "conditions": { "operator": "in", "value": ["USA","GBR","DEU","FRA","CAN","AUS","SGP","JPN","CHE","NLD"] }, "score": 5 }
        ],
        "defaultScore": 12
      }
    },
    {
      "name": "beneficialOwnerJurisdiction",
      "description": "Risk of beneficial-owner jurisdictions (array field)",
      "minScore": 3, "maxScore": 15, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "members.memberJurisdictions",
        "scoreConditions": [
          { "conditions": { "operator": "in", "value": ["IRN","PRK","MMR","SYR","RUS"] }, "score": 15 },
          { "conditions": { "operator": "in", "value": ["PAN","CYM","VGB"] }, "score": 10 },
          { "conditions": { "operator": "in", "value": ["USA","GBR","DEU"] }, "score": 3 }
        ],
        "defaultScore": 8
      }
    },
    {
      "name": "businessType",
      "description": "Inherent risk of the business line",
      "minScore": 5, "maxScore": 20, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "business.businessType",
        "scoreConditions": [
          { "conditions": { "operator": "in", "value": ["EXCHANGE","BLOCKCHAIN_NATIVE"] }, "score": 20 },
          { "conditions": { "operator": "in", "value": ["PAYMENTS","RETAIL_TRADING","INSTITUTIONAL_TRADING"] }, "score": 12 },
          { "conditions": { "operator": "in", "value": ["BANK","FINANCIAL_INSTITUTION","ASSET_MANAGER","WEALTH_MANAGER"] }, "score": 8 },
          { "conditions": { "operator": "in", "value": ["GOVERNMENT","NON_FINANCIAL_CORPORATE"] }, "score": 5 }
        ],
        "defaultScore": 10
      }
    },
    {
      "name": "incorporationType",
      "description": "Legal structure risk",
      "minScore": 2, "maxScore": 10, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "business.incorporationType",
        "scoreConditions": [
          { "conditions": { "operator": "in", "value": ["DECENTRALIZED_AUTONOMOUS_ORGANIZATION","TRUST"] }, "score": 10 },
          { "conditions": { "operator": "in", "value": ["UNINCORPORATED_ASSOCIATION","LIMITED_PARTNERSHIP"] }, "score": 7 },
          { "conditions": { "operator": "in", "value": ["CORPORATION","LIMITED_LIABILITY_COMPANY"] }, "score": 4 },
          { "conditions": { "operator": "in", "value": ["SOLE_PROPRIETORSHIP","GENERAL_PARTNERSHIP"] }, "score": 2 }
        ],
        "defaultScore": 6
      }
    },
    {
      "name": "ownershipComplexity",
      "description": "Ownership structure complexity",
      "minScore": 2, "maxScore": 10, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "business.ownershipComplexity",
        "scoreConditions": [
          { "conditions": { "operator": "equals", "value": "COMPLEX_OWNERSHIP" }, "score": 10 },
          { "conditions": { "operator": "equals", "value": "SIMPLE_OWNERSHIP" }, "score": 2 }
        ],
        "defaultScore": 6
      }
    },
    {
      "name": "entityAge",
      "description": "Years since incorporation (younger = riskier)",
      "minScore": 1, "maxScore": 10, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "general.timeSinceIncorporation",
        "scoreConditions": [
          { "conditions": { "operator": "lessThan", "value": 1 }, "score": 10 },
          { "conditions": { "operator": "between", "min": 1, "max": 3 }, "score": 6 },
          { "conditions": { "operator": "greaterThan", "value": 3 }, "score": 1 }
        ],
        "defaultScore": 10
      }
    },
    {
      "name": "relationshipAge",
      "description": "Years since onboarding (newer = riskier)",
      "minScore": 1, "maxScore": 5, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "application.relationshipAge",
        "scoreConditions": [
          { "conditions": { "operator": "lessThan", "value": 1 }, "score": 5 },
          { "conditions": { "operator": "between", "min": 1, "max": 3 }, "score": 3 },
          { "conditions": { "operator": "greaterThan", "value": 3 }, "score": 1 }
        ],
        "defaultScore": 5
      }
    },
    {
      "name": "pep",
      "description": "Any beneficial owner politically exposed",
      "minScore": 0, "maxScore": 10, "calculationMethod": "pipeline",
      "pipeline": {
        "sourceField": "politicalExposure.isPoliticallyExposed",
        "scoreConditions": [
          { "conditions": { "operator": "equals", "value": true }, "score": 10 },
          { "conditions": { "operator": "equals", "value": false }, "score": 0 }
        ],
        "defaultScore": 0
      }
    }
  ],
  "thresholds": {
    "low":    { "min": 0,  "max": 33 },
    "medium": { "min": 34, "max": 66 },
    "high":   { "min": 67, "max": 100 }
  },
  "overrideConditions": [
    { "field": "sanctions.isSanctioned", "operator": "equals", "value": true,
      "overrideLevel": "HIGH", "reason": "Sanctioned entity" },
    { "conditions": [
        { "field": "politicalExposure.isPoliticallyExposed", "operator": "equals", "value": true },
        { "field": "general.countryOfIncorporation", "operator": "in", "value": ["IRN","PRK","MMR","SYR"] }
      ],
      "overrideLevel": "HIGH", "reason": "PEP incorporated in a prohibited jurisdiction" }
  ],
  "rules": []
}
```

Self-consistency check: `20+15+20+10+10+10+5+10 = 100`. ✔ For a ready-made starting point, `GET /v1/risk/formulas/examples/fireblocks` (15-factor corporate model, sums to 100) and `GET /v1/risk/formulas/examples/monetae?entityType=…` return full `CreateRiskFormulaDto` payloads you can duplicate and tweak.

---

## Best practices

1. **Discover before you author.** Call `GET /factor-fields?entityType=` and `GET /factor-condition-templates?entityType=` first — don't guess field paths or hand-list countries when a CPI/Basel template exists.
2. **Make `maxScore`s add up to your budget.** Design weights by picking `maxScore` per factor so the sum is exactly 100 (or your chosen ceiling). The relative `maxScore` *is* the weight.
3. **Always set `defaultScore`.** Real client data has gaps; the no-match fallback should reflect your risk appetite for "unknown" (often mid-range, not 0).
4. **Use `notExists`/`defaultScore` for missing data deliberately.** Treating missing PEP/geography as low risk is a classic under-scoring bug.
5. **Simulate against the full portfolio before activating.** Check the transition matrix and `movedHigher`/`movedLower` counts — a formula change that silently re-bands hundreds of clients is a compliance event, not a deploy.
6. **Use `deploy` (not `update`) to change a live model.** Deploy versions and audits the change and can trigger a rescore; `update` on an `ACTIVE` formula only lets you touch name/description anyway.
7. **Put the most severe overrides first.** First match wins — a `HIGH` sanctions override should precede softer ones.
8. **Prefer `dryRun: true` when testing assessment calls** so you don't persist noise or mutate `currentRisk`.
9. **Keep factor `name`s stable across versions.** Simulation diffs and factor-delta reporting key off names; renaming churns the history.

---

## Common pitfalls

| Mistake | Fix |
|---------|-----|
| Omitting `rules` on create | `rules` is required — pass `[]` if you have no recommendation rules. |
| Using `clientData` in the assess body | The field is **`entityData`**. `clientData` is stale README terminology. |
| `calculationMethod: "custom"` / `"lookup"` / `"rules"` | Only **`"pipeline"`** is accepted; anything else is rejected. |
| Factor `maxScore`s summing to > 100 | Validation fails. Rebalance so the sum ≤ 100. |
| Overlapping thresholds | Ensure `low.max < medium.min` and `medium.max < high.min`. |
| `between` with a `value` | `between` uses **`min`/`max`**; `value` is ignored. |
| `contains` on a single-select field, or `equals` on a multi-select | Single-select allows `equals`/`in`; multi-select allows `contains`/`in`. Check the [operator table](#operators-reference). |
| `exists`/`notExists` on a boolean field | Boolean fields allow **only `equals`** at registry validation. Use `equals: false`. |
| `notExists` in an override, or `containsAny` in a score condition | The two operator sets differ — overrides lack `notExists`, scoring lacks `containsAny`. |
| Expecting overrides to change the score | Overrides set `effectiveRiskLevel` only; `totalScore` and stored `riskLevel` are unchanged. |
| Assess-by-id without `formulaId` | `POST /assess/:entityType/:entityId` requires `?formulaId=`; it does not auto-pick the active formula. Fetch it via `/formulas/active` first. |
| Formula created but never scores | It's a `DRAFT`. Activate or deploy it. |
| Building a formula for `TRANSACTION`/`BLOCKCHAIN_WALLET` | No fields are registered for those types — only `CORPORATE_CLIENT`/`INDIVIDUAL_CLIENT` are scorable today. |
| Two active formulas for one entity type | Not allowed — activate `409`s; deploy swaps. Deactivate the old one or deploy over it. |
| Expecting activation to overwrite a client's manual risk | `MANUAL_OVERRIDE` current-risk is never clobbered by model risk. |
| Passing `platformId` in params/body | It's derived from the auth context; request-supplied values are ignored. |

## Error handling

| Symptom | Cause / status | Fix |
|---------|----------------|-----|
| `400 Risk thresholds must not overlap` | Bands overlap | Fix threshold boundaries. |
| `400 …maximum score must not exceed 100` | Σ `maxScore` > 100 | Rebalance factor caps. |
| `400 Invalid score range for factor: X` | `minScore >= maxScore` | Ensure `minScore < maxScore`. |
| `400 Unsupported calculation method for factor: X` | Non-pipeline method | Use `calculationMethod: "pipeline"`. |
| `400 Pipeline sourceField/scoreConditions required` | Empty pipeline | Provide a source field and ≥1 condition. |
| `400 operator "…" is not allowed for field type "…"` | Registry op check (activate/deploy) | Pick an operator valid for that value type. |
| `400 custom field mapping must use exactly one source field` | Custom field with multi-source | One source field equal to `customFields.<key>.value`. |
| `404 Active risk formula … not found` | Assessment `formulaId` not `ACTIVE` on this platform | Activate the formula, or pass an active id. |
| `400 entityType is required when multiple active formulas exist` | `/formulas/active` without `entityType` | Pass `entityType`. |
| `409 Conflict` (activate) | Another formula already `ACTIVE` for the entity type | Deactivate it first, or use `deploy`. |
| `409 Conflict` (deploy) | `expectedVersion` stale / concurrent deploy | Re-fetch the active version and retry. |
| `400 Risk assessment failed: …` | Runtime pipeline error (fail-loud) | The engine aborts rather than mis-scoring — fix the offending factor/field. |

All validation/runtime failures are `BadRequestException` (400); lookups are `NotFoundException` (404); concurrency/uniqueness are `ConflictException` (409). The rules engine is the one resilient layer — a rule error is logged as a warning and the assessment still succeeds.

---

## Source map & doc links

Key files (`risk-rating-service`, `src/risk-rating/`):

| File | What's in it |
|------|--------------|
| `dto/risk-formula.dto.ts` | `CreateRiskFormulaDto`, `Update/Deploy/Duplicate`, `RiskFactorDefinitionDto`, `PipelineDto`, `ScoreConditionDto`, `RiskThresholdsDto`, `OverrideConditionDto`, `DeployRiskFormulaRescorePolicy`. |
| `dto/risk-assessment.dto.ts` | `RunRiskAssessmentDto`, `BatchRiskAssessmentDto`, result DTOs. |
| `dto/risk-formula-simulation.dto.ts` | `TriggerSimulationRunDto`, `SimulationRunDto`, aggregates, limits. |
| `entities/risk-formula.entity.ts` | `RiskFactorDefinition`/`ScoreCondition`/`RiskThresholds` interfaces, `OVERRIDE_OPERATORS`, columns. |
| `fields/risk-factor-field.registry.ts` | Field catalog per entity type; `getAllowedOperators`. |
| `fields/risk-factor-condition-template.registry.ts` | CPI / Basel templates + country tiers. |
| `fields/custom-risk-factor-field-types.ts` | Custom field types + mapping. |
| `utils/country-risk-data.ts` | Curated ISO3 country lists (FATF, EU, jurisdiction buckets). |
| `risk-formula.service.ts` | The engine: `runAssessment`, `calculateFactorScores`, `evaluatePipeline`, `doesConditionMatch`, `determineRiskLevel`, `checkOverrides`, validation, lifecycle, `getActiveFormula`. |
| `risk-formula-simulation.service.ts` | Simulation execution, aggregates, staleness signatures. |
| `risk-rating.controller.ts` | The `risk` REST controller (all routes above) + built-in example builders. |
| `risk-formula-simulation.controller.ts` | The `risk/simulation` REST controller. |
| `risk-application.service.ts` | Auto-apply to entity `currentRisk` (autoRiskRating gate, manual-override protection). |
| `entities/example-production-risk-modules.ts` | Ground-truth Fireblocks & Monetae factor arrays. |

Internal deep-dive docs (`internal-docs` MCP, repo `risk-rating-service`):

- `architectural-patterns/pipeline-based-scoring-with-custom-operators`
- `architectural-patterns/formula-versioning-with-formulagroupid-and-version-history`
- `architectural-patterns/simulation-before-deploy-workflow-with-scoring-signature-staleness-detection`
- `architectural-patterns/manual-override-protection-model-risk-never-clobbers-manual-risk-source`
- `key-modules/src-risk-rating-dto`, `key-modules/src-risk-rating-fields`, `key-modules/src-risk-rating-risk-formula-service`

---

## README discrepancies

The service's `README-RISK-RATING.md` predates the pipeline rewrite. When it disagrees with this skill or the source, **trust the source**. Confirmed stale claims:

| README says | Reality |
|-------------|---------|
| Module is `corporate-clients`; routes under `/v1/corporate-clients/risk/...` | Module is `src/risk-rating/`; routes under **`/v1/risk/...`**. The service is entity-type-generic. |
| Three calculation methods: custom (sandboxed JS), lookup, rules | Only **`pipeline`**. No sandboxed-JS execution exists. `rules` is a separate top-level field, not a factor method. |
| Assessment body field `clientData` | Field is **`entityData`**. |
| `DELETE` = "deactivate" | `DELETE` is a **soft delete** (`deletedAt`); deactivate is a separate call. |
| Overrides "force specific risk levels" (rewrite the score) | Overrides set `effectiveRiskLevel` only; the stored model `riskLevel`/`totalScore` are unchanged. |
| Only ~8 endpoints | The controller exposes 23, plus a separate simulation controller. |
