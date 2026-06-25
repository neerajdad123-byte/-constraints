---
name: constraints
description: >
  Declare file-level constraints that agents MUST respect. Use `/constraints only <files>` to
  restrict changes to specific files or `/constraints except <files>` to protect files from
  modification. Agents check constraints before every file edit. Supports glob patterns.
  Use when you need to scope an agent's work tightly or protect critical files.
argument-hint: "[only|except|show|clear] [files...]"
user_invocable: true
---

# Constraints Skill

You are a constraint enforcer. When this skill is active, you MUST respect all declared
file constraints before making any edit, write, delete, or rename operation.

## How It Works

Constraints are stored in `.constraints.yaml` at the workspace root. This file lists
allowed and denied file patterns. You read it at the start of every turn and check
every file operation against it.

## Constraint Commands

When the user says `/constraints <cmd> [args]`, you parse and apply:

### `/constraints only <files...>`
Restrict ALL edits to ONLY the listed files/globs. Any file not matching these patterns
is off-limits. Example:

```
/constraints only src/auth.ts src/login.ts
/constraints only src/**/*.ts
```

This sets `mode: allowlist` and adds the patterns.

### `/constraints except <files...>`
Protect listed files/globs from ANY modification. Everything else is allowed. Example:

```
/constraints except .env* src/config.ts
/constraints except **/secrets/**
```

This sets `mode: denylist` and adds the patterns.

### `/constraints clear`
Remove all constraints. Agent operates freely again.

### `/constraints show`
Display current constraints to the user.

## File Format (`.constraints.yaml`)

```yaml
mode: allowlist | denylist
patterns:
  - "src/**/*.ts"
  - "!src/config.ts"   # negation inside allowlist means "not this"
```

## Enforcement Rules

1. **Read constraints at the start of every turn.** If `.constraints.yaml` exists and
   has patterns, constraints are ACTIVE.
2. **Check BEFORE every file mutation.** Before `edit`, `write`, `delete`, `rename`, or
   any AST operation that modifies a file, check the target path against constraints.
3. **Allowlist mode**: Only files matching at least one pattern are allowed.
4. **Denylist mode**: Files matching any pattern are blocked; everything else allowed.
5. **Negation patterns** (prefixed with `!`) exclude from a allowlist.
6. **Directories count**: a pattern matching a directory covers all files within it.
7. **Default**: No `.constraints.yaml` or empty patterns → no constraints, all files allowed.

## Violation Response

When a file operation would violate a constraint:

1. **Do NOT perform the operation.**
2. **Tell the user**: "Constraint blocked: `<file>` is not in the allowed set."
   or "Constraint blocked: `<file>` is protected."
3. **Suggest**: show the current constraints (`/constraints show`) or offer to adjust.
4. **Continue** with other work that does not violate constraints.

## Examples

```
User: /constraints only src/auth.ts
User: refactor the auth module
Agent: [edits src/auth.ts]
Agent: [tries to edit src/utils.ts]
Agent: Constraint blocked: "src/utils.ts" is not in the allowed set (only: src/auth.ts).
       Continue with other changes or use /constraints to adjust.

User: /constraints except .env* src/secrets/
User: update all config files
Agent: [edits src/config.ts, src/app.config.ts]
Agent: [skips .env.local — protected by constraint]
```

## Interaction with Other Skills

- **Ponytail mode**: Constraints still apply. Lazy doesn't mean careless.
- **Sub-agents**: Constraints apply to sub-agents too. Pass current constraints in the
  context when spawning sub-agents.
- **Multi-turn**: Constraints persist across turns until cleared.

## Rules

- NEVER silently ignore a constraint. Always raise it to the user.
- NEVER edit `.constraints.yaml` yourself unless the user said `/constraints` — it's
  the constraint store, not application code.
- A constraint on a directory means all files and subdirectories are covered.
- Empty `.constraints.yaml` = no constraints active.
- When in doubt, DEFAULT TO BLOCKING the operation. False positive (blocking something
  safe) is better than false negative (editing a protected file).
