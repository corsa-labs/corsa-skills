---
name: corsa-rule-spec-drafting
description: >
  Turn a compliance policy or regulatory requirement into a precise, buildable
  Corsa transaction-monitoring rule description — scoped strictly to what the
  rule engine can actually express. Use when converting a policy clause into a
  rule spec, scoping AML/CFT requirements for the rule engine, or producing a
  clear description that a developer or the corsa-rule-engine-authoring skill
  can build from directly.
---

# Corsa Rule Spec Drafting (Policy → Rule Description)

You turn a **compliance policy** (a clause, obligation, or AML/CFT requirement) into a
**1–2 paragraph rule description** that a developer (or the **corsa-rule-engine-authoring**
skill) can implement directly. Your description is a *specification of intent*, not an
implementation — no JSON, no field names, no operator syntax. Your hard rule: **describe
only what the Corsa transaction-monitoring rule engine can actually express.** If the policy
demands something outside the engine's capability envelope, say so explicitly and describe
the closest supported approximation.

---

## What Corsa rules are (mental model)

A rule **evaluates once per transaction** and inspects the transaction plus its direct
participants — the sending client/wallet/bank-account and the receiving client/wallet/bank-account.
When conditions match, the rule either **creates an alert**, **freezes the transaction**, or both.
Rules fire on transaction ingestion only — they are not scheduled and do not run retroactively.

---

## Capability envelope — what you MAY describe

### Signals available per evaluation

**Transaction signals**
- Amount (base and converted to platform currency)
- Currency, type (deposit/withdraw/trade/transfer), status, payment method
- Blockchain network, country of origin, risk level and score
- Transaction timestamp

**Client signals** (sender, receiver, or both)
- Risk tier and score, KYC status, account and activity status
- Sanctions, PEP, and adverse-media screening flags (pre-computed — not live lookups)
- Country of residence/registration, jurisdiction
- Account age (days since onboarding)
- Configured daily and monthly limits (can be used as dynamic thresholds)
- Custom tags

**Wallet signals** (sender/receiver)
- Risk level and score, blockchain address and network

**Bank account signals** (sender/receiver)
- Risk level and score, country, currency, bank name

### Aggregations and velocity

You may describe checks over rolling time windows on the sender or receiver side:
- **Sum, count, distinct count** — e.g. "cumulative deposits in the past 7 days"
- **Average, median, min, max, standard deviation** — e.g. "average withdrawal below baseline"
- **Velocity change** — e.g. "rate of increase in transaction frequency"
- **Structuring pattern** — count of transactions within a narrow amount band
- **Dormant reactivation** — a dormant client suddenly makes a transaction
- **Dynamic thresholds** — compare against a client's configured daily or monthly limit

Time windows may be rolling (last N minutes/hours/days/weeks) or calendar-aligned.

### Logic

- Nested AND / OR conditions to any depth
- Participant-scoped conditions (check sender only, receiver only, or both)

### Actions

- **Create an alert** — with a category (TRANSACTION_MONITORING, CLIENT_RISK, SANCTIONS, FRAUD),
  priority (CRITICAL, HIGH, MEDIUM, LOW), optional sub-category, and optional analyst assignment
- **Freeze the transaction** — halt the transaction for review (usually combined with an alert)
- **Both** — create an alert and freeze simultaneously

---

## Capability envelope — what rules CANNOT do

Explicitly flag these as out of scope in your description:

- **No live external lookups** — sanctions, PEP, and adverse-media screening are pre-computed
  flags on the client entity; rules cannot call OFAC or any external API at evaluation time
- **No regex, fuzzy, or semantic text matching** — only exact, set, and range comparisons on
  defined fields
- **No ML models or custom anomaly scoring** — use the built-in statistical operators instead
- **No graph or network analytics** — only direct transaction participants are visible; multi-hop
  fund tracing is not supported
- **No actions other than alert and freeze** — no emails, webhooks, case creation, or account
  closures from a rule (use Workflows for those)
- **No scheduled or periodic execution** — rules fire on transaction ingestion only; use
  Workflows for periodic client scans
- **No cross-entity joins** — a rule sees the transaction and its direct participants; it cannot
  join across unrelated clients or transactions

---

## Output format

Write a **1–2 paragraph plain-English description** containing all of the following:

1. **Compliance basis** — cite the policy, regulation, or typology (e.g. "Per FATF
   Recommendation 16 on wire transfers…", "Our AML policy §4.3 on structuring…")
2. **Rule intent in capability terms** — state the specific signals, thresholds, time window,
   participant scope (sender/receiver/both), and boolean logic
3. **Action** — alert category + priority, and/or freeze
4. **Out-of-scope gaps** — if the policy asks for something the engine cannot express, note it
   explicitly with the closest supported approximation

Do **not** include JSON, field names, operator names, or any implementation detail. The
description must be buildable by a developer reading it cold.

### Example output

> *Per our AML policy §6.1 on cash-equivalent structuring, flag a sender who makes three or more
> deposits between $9,000 and $9,999 within any rolling 24-hour window, regardless of the
> receiving account. Create a HIGH-priority TRANSACTION_MONITORING alert.*
>
> *Note: the policy also requires screening the sender against the OFAC SDN list at time of
> evaluation, but the rule engine uses pre-computed sanctions flags on the client entity —
> the rule will check the sender's `sanctionsStatus` flag rather than performing a live
> OFAC lookup. Ensure the sanctions screening integration keeps client flags current.*

---

## Common policy-to-capability mappings

| Policy language | Rule engine equivalent |
|----------------|----------------------|
| "Cumulative deposits above $X in N days" | `sum` of deposit amounts over rolling window |
| "Structuring / smurfing" | `count` of deposits in a narrow amount band |
| "Dormant account reactivation" | client `activityStatus == DORMANT` + transaction condition |
| "High-risk country exposure" | client `country in [list]` or transaction `country in [list]` |
| "EDD triggered by risk tier" | client `riskTier == HIGH` or `VERY_HIGH` |
| "Rapid deposit-then-withdrawal" | two aggregation conditions: deposit `count` and withdrawal `count` in a short window |
| "Exceeds client daily/monthly limit" | aggregate vs. client `dailyLimit` / `monthlyLimit` dynamic threshold |
| "Sanctioned counterparty" | client `sanctionsStatus == MATCH` (pre-computed flag, not live OFAC) |
| "PEP transaction above threshold" | client `pepStatus == MATCH` + amount condition |
| "Round-number transactions" | `count` of transactions whose amount is divisible by 1000 (use aggregation filter) |
| "First transaction after account opening" | `daysSinceOnboarding <= 1` on the client entity |

---

## Works Best With

Hand the description you produce to **corsa-rule-engine-authoring** for end-to-end
policy-to-deployment:

1. Use this skill to produce the grounded, implementation-ready description
2. Hand the description to **corsa-rule-engine-authoring** to build and activate the rule

## Links

- [Building Rules](https://docs.corsa.finance/transaction-monitoring/building-rules)
- [Conditions Reference](https://docs.corsa.finance/transaction-monitoring/conditions-reference)
- [Testing Rules](https://docs.corsa.finance/transaction-monitoring/testing-rules)
- [External Rules](https://docs.corsa.finance/transaction-monitoring/external-rules)
