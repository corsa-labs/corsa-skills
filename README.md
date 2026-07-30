# Corsa Skills

AI skills for integrating with the [Corsa](https://corsa.finance) compliance platform. Built on the [Agent Skills](https://agentskills.io) open standard — works with Cursor, Claude Code, VS Code / Copilot, OpenAI Codex CLI, Gemini CLI, and 40+ more AI tools.

## Available Skills

| Skill | Status | Description |
|-------|--------|-------------|
| **corsa-integration** | Available | Full API integration guide — SDK setup, authentication, data ingestion, webhooks |
| **corsa-webhook-debugging** | Available | Debug webhook delivery, signature verification, and event handling |
| **corsa-rule-authoring** | Available | Create and manage compliance rules via the SDK — conditions, aggregations, actions, evaluation |
| **corsa-rule-engine-authoring** | Available | Deep rule authoring via REST API — full property catalog, advanced aggregations, lifecycle management, and evaluation |
| **corsa-rule-spec-drafting** | Available | Turn a compliance policy into a precise, buildable rule description scoped to what the rule engine can express |
| **corsa-data-pipeline** | Available | Build production data pipelines — backfill, real-time sync, entity mapping |
| **corsa-workflow-authoring** | Available | Author no-code compliance workflows — triggers, node trees, conditions, variables, and deployment via workflow-builder MCP tools or REST API |
| **corsa-workflow-scoping** | Available | Translate a compliance policy into a workflow description grounded in real Corsa workflow capabilities |

## Installation

### Cursor (Marketplace)

Install directly from the [Cursor Marketplace](https://cursor.com/marketplace) — search for **Corsa**.

### Cursor (Manual)

```bash
git clone https://github.com/corsa-labs/corsa-skills.git
cp -r corsa-skills/corsa-integration/skills/corsa-integration ~/.cursor/skills/
cp -r corsa-skills/corsa-data-pipeline/skills/corsa-data-pipeline ~/.cursor/skills/
cp -r corsa-skills/corsa-webhook-debugging/skills/corsa-webhook-debugging ~/.cursor/skills/
cp -r corsa-skills/corsa-rule-authoring/skills/corsa-rule-authoring ~/.cursor/skills/
cp -r corsa-skills/corsa-rule-engine-authoring/skills/corsa-rule-engine-authoring ~/.cursor/skills/
cp -r corsa-skills/corsa-rule-spec-drafting/skills/corsa-rule-spec-drafting ~/.cursor/skills/
cp -r corsa-skills/corsa-workflow-authoring/skills/corsa-workflow-authoring ~/.cursor/skills/
cp -r corsa-skills/corsa-workflow-scoping/skills/corsa-workflow-scoping ~/.cursor/skills/
```

**Optional: Cursor rules for `corsa-integration`**

The `corsa-integration` package also ships a Cursor rules file with SDK coding conventions (correct auth format, `referenceId` patterns, deprecated aliases). To install it alongside the skill:

```bash
mkdir -p ~/.cursor/rules
cp corsa-skills/corsa-integration/rules/corsa-sdk-patterns.mdc ~/.cursor/rules/
```

This file has `alwaysApply: false` — Cursor will suggest it when you're working on files that match `**/*.ts`.

### Claude Code

```bash
git clone https://github.com/corsa-labs/corsa-skills.git
cp -r corsa-skills/corsa-integration/skills/corsa-integration ~/.claude/skills/
cp -r corsa-skills/corsa-data-pipeline/skills/corsa-data-pipeline ~/.claude/skills/
cp -r corsa-skills/corsa-webhook-debugging/skills/corsa-webhook-debugging ~/.claude/skills/
cp -r corsa-skills/corsa-rule-authoring/skills/corsa-rule-authoring ~/.claude/skills/
cp -r corsa-skills/corsa-rule-engine-authoring/skills/corsa-rule-engine-authoring ~/.claude/skills/
cp -r corsa-skills/corsa-rule-spec-drafting/skills/corsa-rule-spec-drafting ~/.claude/skills/
cp -r corsa-skills/corsa-workflow-authoring/skills/corsa-workflow-authoring ~/.claude/skills/
cp -r corsa-skills/corsa-workflow-scoping/skills/corsa-workflow-scoping ~/.claude/skills/
```

### VS Code / GitHub Copilot

```bash
git clone https://github.com/corsa-labs/corsa-skills.git
cp -r corsa-skills/corsa-integration/skills/corsa-integration .github/skills/
cp -r corsa-skills/corsa-data-pipeline/skills/corsa-data-pipeline .github/skills/
cp -r corsa-skills/corsa-webhook-debugging/skills/corsa-webhook-debugging .github/skills/
cp -r corsa-skills/corsa-rule-authoring/skills/corsa-rule-authoring .github/skills/
cp -r corsa-skills/corsa-rule-engine-authoring/skills/corsa-rule-engine-authoring .github/skills/
cp -r corsa-skills/corsa-rule-spec-drafting/skills/corsa-rule-spec-drafting .github/skills/
cp -r corsa-skills/corsa-workflow-authoring/skills/corsa-workflow-authoring .github/skills/
cp -r corsa-skills/corsa-workflow-scoping/skills/corsa-workflow-scoping .github/skills/
```

## Resources

- [Corsa Documentation](https://docs.corsa.finance)
- [SDK on npm](https://www.npmjs.com/package/@corsa-labs/sdk)
- [Agent Skills Specification](https://agentskills.io/specification)
