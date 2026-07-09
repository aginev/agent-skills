# skills

A collection of agent skills. Each skill lives in its own directory under
`skills/` and is defined by a `SKILL.md` file with YAML frontmatter (`name`,
`description`) followed by the skill's instructions.

## Installation

Install these skills with:

```bash
npx skills add aginev/agent-skills
```

## Available skills

| Skill | Description |
| --- | --- |
| [`critically-evaluate`](skills/critically-evaluate/SKILL.md) | Get an adversarial second opinion on a design doc, ADR, RFC, spec, or plan by running OpenAI's Codex CLI against it, then having Claude critically review Codex's feedback point by point. |

## Structure

```
skills/
  <skill-name>/
    SKILL.md        # frontmatter + instructions
```

## Adding a skill

1. Create a new directory under `skills/` named after the skill.
2. Add a `SKILL.md` with frontmatter:

```markdown
---
name: my-skill
description: >-
  When and why an agent should trigger this skill.
---

# My skill

Instructions the agent follows when the skill is invoked.
```

3. Write the description so it clearly states the triggers and the deliverable —
   the agent uses it to decide when to reach for the skill.
