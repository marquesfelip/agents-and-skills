---
name: skill-creator
description: 'Create a high-quality SKILL.md from scratch. Use when: writing a new skill, creating a skill file, packaging a workflow into a skill, making a reusable skill, building skill template, skill from conversation, create skill, new skill. Interviews user, selects scope, designs keyword-rich description, structures workflow with decision tables, adds supporting assets, and validates against best practices.'
argument-hint: 'Describe the workflow or domain you want to package as a skill'
---

# Skill Creator

## Purpose
Produce a discoverable, self-contained `SKILL.md` that follows VS Code agent skill best practices — ready to trigger automatically or via slash command.

## Required Reading
Load these before proceeding:
- [Skill reference](copilot-skill:/agent-customization/references/skills.md)
- [Agent-customization overview](copilot-skill:/agent-customization/SKILL.md)

---

## Step 1 — Gather Requirements

If the `ask-questions` tool is available, use it. Otherwise ask these sequentially in chat:

1. **What should this skill produce?** (One-sentence output description)
2. **When should it activate?** (List 4–6 trigger phrases — the more specific, the better)
3. **Who uses it?**
   - Just me → personal scope (`~/.copilot/skills/<name>/`)
   - My team → workspace scope (`.github/skills/<name>/`)
4. **Does it need bundled assets?** (scripts, templates, reference docs)
5. **What tools should it invoke?** (terminal, file system, subagents, ask-questions, web fetch?)
6. **Are there decision branches?** (different paths depending on user input or context)

---

## Step 2 — Verify It Should Be a Skill

Use the [primitive decision guide](./references/primitive-decision-guide.md). If criteria fail, suggest the right primitive instead.

| Question | Skill ✓ | Alternative |
|---|---|---|
| Multi-step, repeatable workflow? | Yes | Single prompt |
| Needs bundled assets (scripts/templates)? | Yes | Prompt or Instructions |
| On-demand (not always-on)? | Yes | Instructions file |
| Same tool set for all steps? | Yes | Custom Agent (if needs isolation) |
| Single focused task with only input params? | No | Prompt file |

---

## Step 3 — Design the Name

Rules:
- 1–64 characters, lowercase alphanumeric + hyphens only
- **Must exactly match the folder name** (silent failure if mismatched)
- Action-oriented: `review-pr`, `generate-tests`, `create-migration`, `skill-creator`

---

## Step 4 — Craft the Description

The `description` is the **discovery surface**. If trigger keywords are missing, the skill won't be found.

**Pattern:**
```
'<Action verb> <output noun>. Use when: <trigger1>, <trigger2>, <trigger3>, <trigger4>. <What it invokes or produces as extra context.>'
```

**Checklist:**
- [ ] Opens with the action verb + output noun
- [ ] Contains `"Use when:"` with 4–6 specific trigger phrases
- [ ] Under 1024 characters
- [ ] Mentions key tools invoked (if notable)
- [ ] Wrapped in single quotes (avoids YAML colon issues)
- [ ] No unescaped colons inside the value

---

## Step 5 — Structure the SKILL.md Body

Use [skill-template.md](./assets/skill-template.md) as the starting scaffold. Fill it in with:

### Sections to Include

| Section | Required | Notes |
|---|---|---|
| `## When to Use` | Yes | Bullet list of trigger conditions |
| `## Required Reading` | If loading refs | Links to skill assets or other skills |
| `## Step N — <Phase>` | Yes (numbered) | One step per logical phase |
| Decision tables | If branching | Use markdown tables for if/else logic |
| `## Quality Checks` | Yes | Checkboxes to validate before finishing |
| `## After Completion` | Recommended | What to tell the user when done |

### Guidelines
- Keep SKILL.md **under 500 lines** — offload deep content to `./references/`
- Use decision tables for branching logic (more scannable than prose)
- Reference all bundled assets with relative paths: `[template](./assets/template.md)`
- End each step with a clear output artifact or completion signal

---

## Step 6 — Plan Supporting Assets

Evaluate each asset tier:

| Tier | Folder | Create When |
|---|---|---|
| Reference docs | `./references/` | Domain knowledge that exceeds ~50 lines in SKILL.md |
| Scripts | `./scripts/` | Executable automation the skill invokes |
| Templates | `./assets/` | Boilerplate the user fills in or copies |

For a skill-creator skill, a `./assets/skill-template.md` starter file is almost always worth including.

---

## Step 7 — Save and Validate

1. Create the folder at the correct scope location
2. Save `SKILL.md` inside it
3. Save supporting assets in their subfolders
4. Run this validation checklist:

- [ ] `name` in frontmatter **exactly** matches the folder name
- [ ] `description` is single-quoted and under 1024 chars
- [ ] No tabs in YAML frontmatter (spaces only)
- [ ] Body has `## When to Use` section
- [ ] Body has numbered, actionable `## Step N` sections
- [ ] Every asset reference uses `./` relative paths
- [ ] Quality Checks section is present

---

## Step 8 — Iterate

1. Identify the weakest part (vague step, missing branch, thin description)
2. Ask the user **one targeted question** about that specific weakness
3. Refine, re-validate, re-save

---

## After Completion

Tell the user:
1. **What the skill produces** — one-sentence summary
2. **How to invoke it** — example slash command or trigger phrase
3. **2–3 example prompts** they can try immediately
4. **Suggested next customizations** — companion prompt file, instructions file, or related agent
