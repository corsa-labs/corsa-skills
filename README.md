# Corsa Skills

AI skills for integrating with the [Corsa](https://corsa.finance) compliance platform. Built on the [Agent Skills](https://agentskills.io) open standard — works with Cursor, Claude Code, VS Code / Copilot, OpenAI Codex CLI, Gemini CLI, and 35+ more AI tools.

## Available Skills

| Skill | Status | Description |
|-------|--------|-------------|
| **corsa-integration** | Available | Full API integration guide — SDK setup, authentication, data ingestion, webhooks |
| **corsa-webhook-debugging** | Available | Debug webhook delivery, signature verification, and event handling |
| **corsa-rule-authoring** | Available | Create and manage compliance rules for transaction monitoring |
| **corsa-data-pipeline** | Available | Build production data pipelines — backfill, real-time sync, entity mapping |
| **corsa-risk-model-authoring** | Available | Author, simulate, and run risk-rating models (risk formulas) for KYC/KYB |

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
cp -r corsa-skills/corsa-risk-model-authoring/skills/corsa-risk-model-authoring ~/.cursor/skills/
```

### Claude Code

```bash
git clone https://github.com/corsa-labs/corsa-skills.git
cp -r corsa-skills/corsa-integration/skills/corsa-integration ~/.claude/skills/
cp -r corsa-skills/corsa-data-pipeline/skills/corsa-data-pipeline ~/.claude/skills/
cp -r corsa-skills/corsa-webhook-debugging/skills/corsa-webhook-debugging ~/.claude/skills/
cp -r corsa-skills/corsa-rule-authoring/skills/corsa-rule-authoring ~/.claude/skills/
cp -r corsa-skills/corsa-risk-model-authoring/skills/corsa-risk-model-authoring ~/.claude/skills/
```

### VS Code / GitHub Copilot

```bash
git clone https://github.com/corsa-labs/corsa-skills.git
cp -r corsa-skills/corsa-integration/skills/corsa-integration .github/skills/
cp -r corsa-skills/corsa-data-pipeline/skills/corsa-data-pipeline .github/skills/
cp -r corsa-skills/corsa-webhook-debugging/skills/corsa-webhook-debugging .github/skills/
cp -r corsa-skills/corsa-rule-authoring/skills/corsa-rule-authoring .github/skills/
cp -r corsa-skills/corsa-risk-model-authoring/skills/corsa-risk-model-authoring .github/skills/
```

## Resources

- [Corsa Documentation](https://docs.corsa.finance)
- [SDK on npm](https://www.npmjs.com/package/@corsa-labs/sdk)
- [Agent Skills Specification](https://agentskills.io/specification)
