# Corsa Skills

AI skills for integrating with the [Corsa](https://corsa.finance) compliance platform. Built on the [Agent Skills](https://agentskills.io) open standard — works with Cursor, Claude Code, VS Code / Copilot, OpenAI Codex CLI, Gemini CLI, and 10+ more AI tools.

## Available Skills

| Skill | Status | Description |
|-------|--------|-------------|
| **corsa-integration** | Available | Full API integration guide — SDK setup, authentication, data ingestion, webhooks, BYOK encryption |
| **corsa-webhook-debugging** | Coming soon | Debug webhook delivery, signature verification, and event handling |
| **corsa-rule-authoring** | Coming soon | Create and manage compliance rules for transaction monitoring |

## Installation

### Cursor (Marketplace)

Install directly from the [Cursor Marketplace](https://cursor.com/marketplace) — search for **Corsa**.

### Cursor (Manual)

```bash
git clone https://github.com/corsa-labs/corsa-skills.git
cp -r corsa-skills/corsa-integration/skills/corsa-integration ~/.cursor/skills/
```

### Claude Code

```bash
git clone https://github.com/corsa-labs/corsa-skills.git
cp -r corsa-skills/corsa-integration/skills/corsa-integration ~/.claude/skills/
```

### VS Code / GitHub Copilot

```bash
git clone https://github.com/corsa-labs/corsa-skills.git
cp -r corsa-skills/corsa-integration/skills/corsa-integration .github/skills/
```

### Universal CLI

```bash
npx ai-agent-skills install corsa-integration --agent cursor
```

## Resources

- [Corsa Documentation](https://docs.corsa.finance)
- [API Reference](https://api.corsa.finance/api-docs/)
- [SDK on npm](https://www.npmjs.com/package/@corsa-labs/sdk)
- [Agent Skills Specification](https://agentskills.io/specification)
