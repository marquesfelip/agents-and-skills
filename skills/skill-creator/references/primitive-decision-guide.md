# Primitive Decision Guide

Use this table to confirm a **Skill** is the right primitive before proceeding.

## Primary Decision Tree

```
User wants a customization
│
├─ Always-on? Applies to most work?
│   └─ YES → Workspace Instructions (copilot-instructions.md / AGENTS.md)
│
├─ File-type specific guidance?
│   └─ YES → File Instructions (*.instructions.md with applyTo glob)
│
├─ Integrates an external system or API?
│   └─ YES → MCP Server
│
├─ Enforce deterministic behavior at lifecycle events?
│   └─ YES → Hooks (.github/hooks/*.json)
│
├─ Single focused task with input parameters only?
│   └─ YES → Prompt (*.prompt.md)
│
├─ Needs context isolation OR different tool restrictions per stage?
│   └─ YES → Custom Agent (*.agent.md)
│
└─ Multi-step, on-demand workflow with bundled resources?
    └─ YES → Skill (SKILL.md) ✓
```

## Skill vs. Prompt (Most Common Confusion)

| Factor | Skill | Prompt |
|---|---|---|
| Steps | Multiple phases | Single task |
| Assets | Scripts, templates, refs | None needed |
| Invocation | Slash command OR auto-loaded | Slash command only |
| Complexity | Decision branches OK | Linear input → output |
| Example | `review-security-audit` (multi-step) | `generate-commit-message` (one-shot) |

## Skill vs. Custom Agent

| Factor | Skill | Custom Agent |
|---|---|---|
| Tool set | Same tools throughout | Can restrict per stage |
| Context | Shared conversation | Spawns isolated subagent |
| Output | Inline in conversation | Single return message |
| Use case | Guided workflow | Research + report delegation |

## File Locations Quick Reference

| Scope | Skill | Prompt | Instructions | Agent |
|---|---|---|---|---|
| Workspace | `.github/skills/<name>/` | `.github/prompts/` | `.github/instructions/` | `.github/agents/` |
| Personal | `~/.copilot/skills/<name>/` | `~/AppData/Roaming/Code/User/prompts/` | same | same |
