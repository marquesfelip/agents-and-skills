---
name: your-skill-name
description: '<Action verb> <output noun>. Use when: <trigger phrase 1>, <trigger phrase 2>, <trigger phrase 3>, <trigger phrase 4>. <Optional: what tools it invokes or extra context.>'
argument-hint: 'Optional: hint shown in chat when user types /<skill-name>'
---

# Skill Title

## When to Use
- <Trigger condition 1>
- <Trigger condition 2>
- <Trigger condition 3>

---

## Step 1 — <Phase Name>

<Instructions for this phase. Be specific about what the agent should do.>

Decision table (if branching logic applies):

| Condition | Action |
|---|---|
| <Case A> | <Do this> |
| <Case B> | <Do that> |

---

## Step 2 — <Phase Name>

<Instructions. Reference assets with relative paths.>

Load [template](./assets/template.md) and fill in:
- `<PLACEHOLDER_1>` → <what goes here>
- `<PLACEHOLDER_2>` → <what goes here>

---

## Step 3 — <Phase Name>

<Instructions. If invoking a script:>

Run [setup script](./scripts/setup.sh) with:
```
<example invocation>
```

---

## Quality Checks
- [ ] <Requirement 1>
- [ ] <Requirement 2>
- [ ] <Requirement 3>

---

## After Completion

Tell the user:
1. What was produced
2. How to use it
3. Suggested next steps
