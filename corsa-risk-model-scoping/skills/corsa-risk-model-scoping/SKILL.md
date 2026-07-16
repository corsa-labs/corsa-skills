---
name: corsa-risk-model-scoping
description: >
  Translate a compliance policy into a buildable Corsa risk-rating model
  description (spec) for another agent to implement. Use when turning a
  regulatory/AML/KYC/KYB policy into a risk-model requirement, deciding what a
  risk model should score, or writing the description a coding agent will build
  from. Summarizes exactly what Corsa risk models can and cannot express so the
  description never promises features the platform lacks.
---

# Corsa Risk Model Scoping

Your job: read a **compliance policy** (or the relevant excerpt) and write a short **description of the risk-rating model to build** — a 1–2 paragraph spec that a coding agent will implement using the `corsa-risk-model-authoring` skill. You do **not** build the model. You produce the requirement.

The one rule that matters: **stay inside the capability envelope below.** Every capability you name in the description must map to something in this file. If the policy asks for something the platform can't express, **say so explicitly in the description** rather than inventing a feature — a hallucinated requirement becomes a coding agent that ships the wrong thing (or nothing).

## What a Corsa risk model is

A risk model scores **one client** — an **individual** or a **corporate** entity — into a **risk level** (`LOW` / `MEDIUM` / `HIGH`) and a **score from 0 to 100**, computed from a set of **weighted factors**. Each factor maps one client attribute (e.g. country, PEP tier, business type) to points via simple conditions; the factor scores are **summed** into the total; the total falls into a band via thresholds. Optional **override conditions** can force a band (e.g. sanctioned → HIGH) regardless of score. It is a deterministic, auditable **profile-scoring** model — think KYC/KYB onboarding and periodic-review risk rating.

## Capability envelope — what you CAN specify

| Capability | What it means for your description |
|------------|-------------------------------------|
| **Entity scope** | Score an **individual client** or a **corporate client**. One model per entity type. |
| **Weighted factors** | Any number of factors. A factor's **maximum points is its weight** — allocate the 100-point budget across factors to reflect policy priority. Total of all factor maxima must be ≤ 100. |
| **Scoring** | Each factor maps a client attribute to points using: equality, set-membership (`in` / `contains`), numeric or date comparisons and ranges (`between`), and presence checks. Factor scores are **summed** (no cross-factor math). |
| **Attribute catalog** | Factors reference a fixed catalog of client attributes (see [below](#attribute-catalog-summary)) plus platform-defined custom fields. |
| **Country risk** | Built-in country-tier scoring: **CPI** (Corruption Perceptions Index) and **Basel AML Index** templates, plus curated high-risk / FATF-grey / EU lists. Use these for any geographic factor. |
| **Bands / thresholds** | Exactly **three** bands — `LOW`, `MEDIUM`, `HIGH` — defined by score cutoffs. You may specify the cutoffs (e.g. HIGH ≥ 67). |
| **Overrides** | Force a band (`LOW`/`MEDIUM`/`HIGH`) when hard conditions match — independent of the computed score. Use for policy "must be classified High if…" clauses (sanctions, PEP in prohibited jurisdiction, etc.). |
| **Recommendations** | An optional rules layer can attach advisory narrative/flags to a result. Advisory only. |
| **Lifecycle** | Models are drafted, **simulated against the real client portfolio** to preview band movement, then activated; they are versioned, with one active model per entity type. You may note that the model should be simulated before activation. |

## Out of scope — do NOT put these in a description

These are the common hallucinations. If the policy implies one, name the boundary and route it elsewhere instead of specifying it as a risk-model feature.

- **No machine learning / behavioral / statistical scoring.** Scoring is deterministic weighted rules only. Don't describe "a model that learns", anomaly detection, or predicted-risk scores.
- **No transaction monitoring, velocity, or aggregation.** A risk model scores a client's **profile**, not their transaction stream. Anything like "flag > $10k in 24h", "detect structuring", "rolling 30-day volume", or "alert on rapid withdrawals" is **transaction-monitoring rules — a separate system** (`corsa-rule-authoring`), not a risk factor. You *may* use **declared / expected** figures that live on the KYC profile (e.g. expected monthly transaction volume, declared annual income); you may **not** use computed actual transaction activity.
- **No entity types beyond individual and corporate clients.** Transactions, wallets, bank accounts, and products cannot be scored as standalone entities.
- **No invented data.** Factors can only read attributes in the catalog (or custom fields that already exist on the platform's client records). If the policy needs a data point the platform doesn't hold, flag it.
- **No custom code / free-form formulas.** Only the fixed operator set over catalog attributes. No sandboxed scripts, no arbitrary expressions.
- **Exactly three bands, score capped at 100.** No custom number of tiers, no custom band names, no 0–1000 scale.
- **Overrides force a band only.** They cannot take action — no auto-offboarding, account freezing, SAR/CTR filing, or notifications. Those are downstream/other systems; describe them as *consequences of the HIGH band*, not model features.
- **No cross-factor weighting logic.** Each factor is scored independently then summed. No "if factor A matched, weight factor B differently", no multiplicative interactions.
- **No live external lookups at scoring time** beyond the client snapshot and a few derived fields (age, relationship tenure, aggregated beneficial-owner jurisdictions).

## Attribute catalog (summary)

Reference these by concept — the coding agent has the exact field paths, options, and operators. Naming a real attribute keeps the description grounded.

**Individual client:** citizenship; country of residence; jurisdiction; age; relationship tenure (time since onboarding); occupation / economic activity; PEP status, PEP tier, PEP jurisdiction; sanctions flag; adverse-media flag / severity / recency; source of funds (+ jurisdictions); declared monthly net income; expected monthly transaction volume; current risk level/score; tags; platform custom fields.

**Corporate client:** country of incorporation; jurisdiction; registration- and business-address country; incorporation type; business type and sub-type (industry); ownership type (public / private) and ownership complexity (simple / complex); time since incorporation; relationship tenure; beneficial-owner / member jurisdictions; source of funds (+ jurisdictions); declared monthly & annual transaction volume; monthly net income; PEP status / tier / jurisdiction; adverse-media severity / recency / score / fines; sanctions flag; subpoena flag; SAR flag; current risk level/score; tags; platform custom fields.

**Geography helpers:** CPI country tiers, Basel AML country tiers, curated high-risk / FATF-grey / EU country lists.

> Note: transaction-volume and income attributes above are **declared / expected KYC figures on the profile** — not live transaction analytics.

## How to write the description

Read the policy, then produce **1–2 paragraphs** that a coding agent can build from. Cover, in order:

1. **Entity type** — individual or corporate client.
2. **Factors** — the risk drivers to score, **each tied to the policy requirement it satisfies** (cite the clause / regulation), with a **relative weight** (heavy / medium / light, or approximate points) reflecting policy emphasis. Use only catalog attributes and the country-risk templates.
3. **Bands & cutoffs** — the three bands, plus any policy-mandated threshold (e.g. "any PEP is at least MEDIUM", "HIGH triggers EDD").
4. **Mandatory overrides** — every "must be classified High/… if…" clause in the policy, as a band-forcing override independent of score.
5. **Out-of-scope flags** — anything the policy asks for that the model can't express (transaction monitoring, actions, ML, missing data). State it plainly so a human handles it elsewhere; do not fold it into the model.

Keep it prose, tight, and traceable — every factor should be justifiable back to the policy. Do not include JSON or field paths (that is the coding agent's job); describe *what and why*, not *how*.

### Description template

> Build a **[INDIVIDUAL_CLIENT | CORPORATE_CLIENT]** risk model scoring 0–100 into LOW/MEDIUM/HIGH. Weight factors to reflect policy priority: **[factor]** (heaviest, ~X pts) per **[policy clause]**, **[factor]** (~Y pts) per **[clause]**, … Use the built-in **[CPI / Basel / FATF]** country tiers for geographic risk. Set bands so that **[policy cutoff, e.g. HIGH ≥ 67 triggers EDD]**. Apply mandatory **HIGH** overrides — independent of score — when **[sanctions hit / FATF-blacklisted jurisdiction / …]**, per **[clause / regulation]**. Simulate against the current portfolio before activation. *Out of scope for the model (handle separately): [transaction-monitoring / enforcement actions / unavailable data].*

## Worked examples

### Individual (policy → description)

**Policy excerpt:** *"All individual customers are risk-rated at onboarding. Geographic risk (residence and citizenship) and political exposure are the primary drivers. Any customer who is sanctioned, or who is resident or a citizen of a FATF call-for-action jurisdiction, must be rated High irrespective of other factors. Adverse media and occupation are secondary factors. High-risk customers undergo enhanced due diligence."*

**Description:**
> Build an **INDIVIDUAL_CLIENT** risk model scoring 0–100 into LOW/MEDIUM/HIGH. Reflect policy priority in the weights: **country of residence and citizenship** as the heaviest factors (≈25 and ≈15 pts) scored via the built-in CPI/Basel country tiers and the curated FATF/high-risk lists, and **PEP tier** next (≈20 pts), per the policy's "primary drivers" clause. Secondary factors — **adverse-media severity** (≈15 pts) and **occupation / economic-activity risk** (≈15 pts) — round out the profile, with **source of funds** and **declared expected transaction volume** as lighter contributors (≈5 pts each). Bands should place clearly clean profiles in LOW and cumulatively risky ones in HIGH; treat the HIGH band as the trigger for enhanced due diligence (EDD is a downstream action, not part of the model).
>
> Apply two **mandatory HIGH-risk overrides**, independent of the computed score: (1) the customer's **sanctions flag** is set, and (2) the customer's **residence or citizenship** is a **FATF call-for-action** country — both per the policy's "irrespective of other factors" clause. Simulate the model against the current individual-client portfolio to confirm the band distribution before activation. *Out of scope for the model: EDD workflow and any offboarding — these are downstream consequences of the HIGH rating, not model features.*

### Corporate (policy → description)

**Policy excerpt:** *"Corporate clients are rated on inherent business risk. Industry/business type, jurisdiction of incorporation, and beneficial-ownership transparency drive the score. Shell-like structures (complex ownership, recently incorporated) are elevated risk. Entities in sanctioned jurisdictions are prohibited (High). Money services and crypto businesses are inherently high risk."*

**Description:**
> Build a **CORPORATE_CLIENT** risk model scoring 0–100 into LOW/MEDIUM/HIGH. Weight the policy's stated drivers heaviest: **business type / sub-type (industry)** (≈20 pts, elevating money-services and crypto businesses per the "inherently high risk" clause), **country of incorporation** (≈20 pts via CPI/Basel tiers + FATF/high-risk lists), and **beneficial-owner / member jurisdictions** (≈15 pts) for ownership transparency. Add **ownership complexity** and **incorporation type** (≈10 pts each) plus **time since incorporation** and **relationship tenure** (≈10 and ≈5 pts) to capture the "shell-like structure" signal (complex ownership, recently incorporated), and a light **PEP** contributor (≈10 pts). Set bands so cumulatively risky profiles land in HIGH.
>
> Apply a **mandatory HIGH override** — independent of score — when the entity's **sanctions flag** is set or its **country of incorporation** is a prohibited/sanctioned jurisdiction, per the "prohibited (High)" clause. Simulate against the corporate portfolio before activation. *Out of scope for the model: the "prohibited" enforcement itself (blocking/offboarding) is a downstream action; the model only rates the entity High.*

## Pre-flight checklist

Before you emit the description, verify:

- [ ] Entity type is individual or corporate client (nothing else).
- [ ] Every factor maps to a **catalog attribute** or a country-risk template — no invented data.
- [ ] Weights are an allocation of a ≤100-point budget; no cross-factor or multiplicative logic.
- [ ] Exactly three bands; score 0–100.
- [ ] Every "must be High/…" policy clause is expressed as a **band-forcing override**, not baked into scoring.
- [ ] No transaction-monitoring / velocity / aggregation logic snuck in (declared/expected profile figures are fine; computed transaction activity is not).
- [ ] No enforcement **actions** described as model features (offboarding, freezing, SAR filing) — only band outputs.
- [ ] Any unsupported policy requirement is **flagged as out of scope**, not hallucinated.
- [ ] Each factor is traceable to a specific policy clause / regulation.
- [ ] Output is 1–2 paragraphs of prose — no JSON, no field paths.

The coding agent's `corsa-risk-model-authoring` skill turns a description that passes this checklist into a valid, activatable formula. A description that fails it turns into rework — or a compliance gap.
