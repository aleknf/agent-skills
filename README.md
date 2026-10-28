# Agent Skills

A collection of skills for AI Agents (OpenBot, Claude Code, Gemini Antigravity, etc.) to automate, guide, and streamline software engineering, platform configurations, and presales workflows. It contains original skills and a small set of curated community skills, kept clearly separate.

---

## Installation

Install with the [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
# Interactive: choose skills and target agents
npx skills add aleknf/agent-skills

# Install one specific skill
npx skills add aleknf/agent-skills --skill salesforce-metadata-deploy

# Install globally (available in every project) instead of the current project
npx skills add -g aleknf/agent-skills --skill lark-okr-planner
```

Skills are discovered from `skills/<name>/SKILL.md` and `skills/curated/<name>/SKILL.md`.

### Manual copy

```bash
# OpenBot / local agent
cp -r skills/<skill-name> ~/.agents/skills/

# Gemini / Antigravity
cp -r skills/<skill-name> ~/.gemini/config/skills/
```

For a curated skill, copy from `skills/curated/<skill-name>`.

---

## Original Skills

Written and maintained in this repository.

| Skill | Description |
| :--- | :--- |
| [`salesforce-metadata-deploy`](skills/salesforce-metadata-deploy/SKILL.md) | Guides and automates SFDX source metadata deployments to target Salesforce orgs via `sf project deploy start`, with dry-run validation, multi-pass dependency ordering, and intelligent error diagnostics. |
| [`lark-okr-planner`](skills/lark-okr-planner/SKILL.md) | Drafts and creates OKRs (objectives, key results, weights, alignment) in Lark for any team or role with `lark-cli`, including the rich-text content format that avoids the "saved as `*`" pitfall. |

---

## Curated Community Skills

**Not original work.** These are copies of community skills, included because they pair well with the original ones. Each keeps its upstream license (MIT), credits its author in the `SKILL.md` frontmatter and notice, and is listed in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). Copies are snapshots and may lag behind upstream.

| Skill | Description | Source |
| :--- | :--- | :--- |
| [`lark-event`](skills/curated/lark-event/SKILL.md) | Stream Lark/Feishu real-time events as NDJSON via `lark-cli event consume`. | [larksuite/cli](https://github.com/larksuite/cli) |
| [`lark-base`](skills/curated/lark-base/SKILL.md) | Operate Lark Base (Bitable): tables, fields, records, views, forms, dashboards, workflows. | [larksuite/cli](https://github.com/larksuite/cli) |
| [`obsidian-markdown`](skills/curated/obsidian-markdown/SKILL.md) | Create and edit Obsidian Flavored Markdown: wikilinks, embeds, callouts, properties. | [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills) |
| [`n8n-workflow-patterns`](skills/curated/n8n-workflow-patterns/SKILL.md) | Proven architectural patterns for n8n workflows (webhook, API, database, AI agent, scheduled). | [czlonkowski/n8n-skills](https://github.com/czlonkowski/n8n-skills) |

---

## Support & Sponsorship

If you find these skills helpful in streamlining your workflow or boosting productivity, you can support development via:

- ☕ **Ko-fi**: [ko-fi.com/alllike](https://ko-fi.com/alllike)
- 🪙 **Saweria**: [saweria.co/alllike](https://saweria.co/alllike)

Thank you for your support! 🙌
