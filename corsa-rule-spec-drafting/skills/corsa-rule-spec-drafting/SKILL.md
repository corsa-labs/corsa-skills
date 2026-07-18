---
name: corsa-rule-spec-drafting
description: >
  Turn a compliance policy into an accurate, buildable description of a Corsa
  transaction-monitoring rule, bounded by what the rule engine can actually
  express. Use when drafting a transaction-monitoring rule requirement or spec
  from a compliance policy, deciding whether a policy control is expressible as
  a rule, or writing the 1-2 paragraph rule description that a coding agent will
  later implement with the corsa-rule-engine-authoring skill. It exists to stop
  the description from hallucinating engine features that don't exist (regex /
  free-text matching, ML or custom anomaly scoring, live sanctions screening,
  graph/network tracing, scheduled or client-triggered runs, or any action
  other than create-alert and halt/freeze).
---

# Corsa Rule Spec Drafting

You translate a **compliance policy** into a short, precise **description of the
transaction-monitoring rule that should be built**. You do **not** write the
rule itself — a downstream coding agent does that using the
`corsa-rule-engine-authoring` skill. Your job is to hand that agent a
requirement that (a) captures the policy's compliance intent and (b) stays
strictly inside what the Corsa rule engine can actually do.

> The failure mode you exist to prevent: a description that asks for a feature
> the engine doesn't have (fuzzy name matching, an ML risk model, a live OFAC
> call, "trace funds three hops out", "run nightly", "email the customer").
> The coding agent would then either fail or silently build something else.
> Describe rules **only** in the capability terms below.

## Your output — the contract

Produce **1–2 paragraphs** of prose. It must contain:

1. **Compliance basis** — the policy/regulatory driver and the AML/fraud
   typology it targets (cite the policy section/clause if one is given, e.g.
   "BSA structuring", "OFAC sanctions", "EDD monitoring").
2. **Rule intent, in capability terms** — what should trigger the rule: which
   signals, what thresholds, what time window, which participant (sender /
   receiver / either), and how conditions combine (AND/OR).
3. **Action** — whether it raises an alert (with a category and priority),
   halts/freezes the transaction, or both.
4. **Gaps, if any** — if the policy asks for something out of envelope, state
   the closest supported approximation and explicitly flag the residual gap for
   a human.

Keep it **declarative** ("flag a sender whose deposits total …"). Do **not**
write JSON, field/fact names, operators, or thresholds-as-code — the coding
agent maps your prose onto the schema. Be specific about numbers and windows
(the policy usually supplies them); stay vague about implementation.

## Capability envelope — what a rule CAN express

**Trigger & scope.** A rule is evaluated **per transaction** (when a transaction
is created). It can inspect the transaction and, for both the **sending** and
**receiving** side, that party's **client, wallet, and bank account**. Client /
wallet / bank conditions are always evaluated in the context of the triggering
transaction.

**Signals available** (reference these, not invented fields):

| Group | Examples of usable signals |
|---|---|
| Transaction | amount, converted amount, net amount, currency, type (deposit / withdrawal / trade / transfer), status (success/pending/cancelled/failed/frozen), payment method, blockchain/chain, source & destination country, risk score/level, timestamps |
| Client (sender/receiver) | risk tier & score, KYC status & tier, individual vs corporate, country / incorporation / jurisdiction, **PEP / sanctions / adverse-media flags & scores** (pre-screened), account age, activity status, configured limits (daily / annual / offramp), alert & case counts, tags, corporate attributes (business type, source of funds, ownership, expected volumes) |
| Client session/device | IP country, VPN / proxy / Tor / datacenter flags, device type, browser, OS |
| Wallet (sender/receiver) | risk level & score, address, chain, associated clients |
| Bank account (sender/receiver) | bank name, country, currency, risk level & score, balance |

**Comparisons.** Equals / not-equals; numeric & date thresholds (greater /
less, inclusive or exclusive); ranges (between two bounds); set membership (in /
not-in a list, e.g. a high-risk-country list); string/array contains /
not-contains.

**Aggregation & velocity** over a **time window**, scoped to the sender's or
receiver's transaction history:

- Totals & statistics: sum, count, distinct count, average, min, max, median,
  standard deviation, percentile.
- Velocity & trend: rate-of-change (velocity change), moving average, deviation
  from average, baseline comparison (current period vs historical), time since
  last transaction, first-transaction detection.
- Patterns: structuring-style counts, repeating amounts, round-number counts,
  consecutive status streaks, minimum activity per period, dormant-account
  reactivation, rapid deposit-then-withdrawal.
- Relationship: new counterparty, shared-wallet usage.
- Windows: rolling ("in the last N minutes/hours/days/weeks/months/years"),
  calendar-aligned (calendar weeks/months/years), before/after/between
  timestamps, or all-time. Aggregations can be filtered (e.g. withdrawals only)
  and can include or exclude the current transaction.
- **Dynamic thresholds**: compare an aggregate to the client's *own* configured
  limit (e.g. "more than 100% of their daily limit"), optionally scaled (e.g.
  60% of it).

**Logic.** Any number of conditions combined with nested **AND / OR** to any
depth.

**Actions when a rule fires.**

- **Create a compliance alert** — with a category (KYC, KYB, transaction
  monitoring, on-chain monitoring, sanctions / PEP / adverse-media / regulatory
  screening, fraud, periodic review, EDD, other), a priority (low / medium /
  high), an initial status, and optionally a sub-category, assignee, due date,
  and description.
- **Halt (freeze) the transaction** — sets it to FROZEN; the only decisions are
  **ALLOW** or **FREEZE**.
- Alert and freeze can be combined. If several rules fire on one transaction,
  one consolidated alert is raised (highest priority) and any freezing rule
  freezes it.

## Out of scope — do NOT ask for these

If the policy implies any of these, do not put it in the rule description as if
the engine does it — approximate and flag instead (see below).

- **Regex / free-text / semantic matching** — no memo-text mining, no fuzzy name
  matching. Only exact / set / contains on defined fields.
- **Live screening or external lookups inside the rule** — sanctions, PEP, and
  adverse-media are read as **pre-computed flags/scores** on the client; the rule
  does not call OFAC or any vendor. Describe it as "fires when the client is
  flagged sanctioned", never "screens the name against the OFAC list".
- **ML / custom anomaly models / bespoke scoring** — only the built-in statistical
  aggregations above. The engine *consumes* existing risk scores; it does not
  train or compute new ones.
- **Graph / network analytics** beyond the transaction's direct participants and
  the built-in new-counterparty / shared-wallet signals — no arbitrary N-hop
  fund tracing, no "counterparty of my counterparty".
- **Actions other than alert and halt/freeze** — no emailing, blocking logins,
  closing/suspending accounts, auto-filing SARs, messaging customers, or calling
  webhooks. (Assignment, priority, due date, and category on the alert are the
  only routing levers.)
- **Scheduled / periodic / client-triggered rules** — there is no "review every
  client every 90 days" rule. Rules fire on transactions. A client attribute can
  be a *condition*, but the *trigger* is always a transaction.
- **Fields not in the catalog** — you cannot invent a transaction attribute that
  the platform doesn't ingest.
- **Cross-transaction correlation** beyond time-window aggregation of the *same
  participant* — no matching an unrelated deposit by a different client to this
  transfer.

## Policy concept → engine lever

| Policy language | Express it as |
|---|---|
| Large / high-value transaction | transaction amount threshold (raw or converted) |
| Cumulative value over a period | sum aggregation + time window |
| Multiple / repeated transactions | count aggregation + window |
| Structuring / just under a reporting threshold | count of transactions with amount between `[threshold−δ, threshold)` in a window |
| Velocity / sudden spike | velocity-change / deviation-from-average / baseline comparison, or count/sum in a short window |
| High-risk customer / wallet / jurisdiction | risk tier/score, wallet risk, or country in a high-risk list |
| Sanctioned / PEP / adverse media | the client's pre-computed sanctions / PEP / adverse-media flag or score |
| Rapid activity shortly after onboarding | account age + activity count/sum in a window |
| Dormant account reactivation | dormant-reactivation aggregation |
| Exceeds the customer's own expected profile/limit | aggregate compared to the client's configured limit (dynamic threshold) |
| Round-number or repeated amounts | round-number / repeating-amount pattern counts |
| Cross-border / specific corridors | source / destination country conditions |
| Trade- / deposit- / withdrawal-specific control | transaction type filter |
| "Generate a case/alert for review" | create-alert (choose category + priority) |
| "Block / hold pending review" | halt/freeze (+ alert) |

## When the policy needs something out of envelope

Describe the **closest supported approximation** and name the gap explicitly, so
a human can route the missing part elsewhere. Example: a policy wanting "fuzzy
match the beneficiary name against our internal watchlist" → describe a rule that
fires on the pre-computed sanctions/adverse-media flag, and add: "the fuzzy
name-matching itself is out of the rule engine's scope and is assumed to be
performed by the screening pipeline that sets the flag."

## Template

> To satisfy **[policy / regulation + typology]**, build a transaction-monitoring
> rule that **[triggering condition: signal(s), threshold(s), time window,
> sender/receiver scope, AND/OR logic]**. On a match, **[action: raise an alert
> with category X at priority Y / and-or halt the transaction]**. *[Optional:
> the rule approximates [policy element] via [supported signal]; [residual gap]
> is out of the rule engine's scope.]*

## Worked examples (grounded — safe to imitate)

**Structuring (BSA / CTR evasion)**
> To satisfy the AML program's requirement to detect structuring and evasion of
> the $10,000 CTR reporting threshold under the BSA, build a transaction-
> monitoring rule that flags a sending customer who makes three or more deposits,
> each between $9,000 and $9,999, within a rolling 24-hour window. The rule
> should look only at deposit-type transactions on the sender side, count the
> qualifying deposits over the trailing 24 hours, and fire when the count reaches
> three. On a match, raise a transaction-monitoring alert with a "Structuring"
> sub-category at high priority for analyst review; freezing is not required at
> this stage.

**Sanctions exposure (OFAC) — alert + freeze**
> To meet OFAC sanctions-compliance obligations, build a rule that fires whenever
> the sending or receiving customer is flagged as sanctioned. Because sanctions
> status is screened upstream, the rule reads the customer's pre-computed
> sanctions flag rather than performing any list matching itself. On a match,
> both halt (freeze) the transaction and raise a high-priority
> sanctions-screening alert in an escalated status. Note: name/fuzzy watchlist
> matching is out of the rule engine's scope and is assumed to be handled by the
> screening pipeline that sets the flag.

**Activity beyond expected profile (EDD) — dynamic threshold**
> To support enhanced due-diligence monitoring of customers who transact beyond
> their declared profile, build a rule that flags a sending customer whose total
> withdrawals over a rolling 24-hour window exceed their own configured daily
> transaction limit. The rule aggregates withdrawal amounts for the sender across
> the trailing day and compares the total to that customer's daily limit (a
> per-customer dynamic threshold, not a fixed number). On a match, raise a
> high-priority transaction-monitoring alert ("Limit breach") for EDD review.

## Checklist before you hand off

- [ ] Every trigger uses a real signal from the catalog above.
- [ ] Thresholds, time windows, and participant scope (sender/receiver) are explicit.
- [ ] The action is limited to create-alert and/or halt/freeze, with a category and priority chosen.
- [ ] No regex, ML, live-lookup, graph-tracing, scheduled-run, or non-alert/freeze action was requested.
- [ ] The policy driver is cited and any out-of-scope portion is flagged.
- [ ] Output is 1–2 declarative paragraphs — no JSON, no field/fact names.

---

*Downstream:* hand the description to an agent running **`corsa-rule-engine-authoring`**,
which owns the full schema, operators, entity/property catalog, and deployable
JSON payloads.
