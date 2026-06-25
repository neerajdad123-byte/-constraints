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

## Two Ways to Set Constraints

### 1. Explicit command: `/constraints <cmd> [args]`

#### `/constraints only <files...>`
Restrict ALL edits to ONLY the listed files/globs. Example:

```
/constraints only src/auth.ts src/login.ts
/constraints only src/**/*.ts
```

This sets `mode: allowlist` and adds the patterns.

#### `/constraints except <files...>`
Protect listed files/globs from ANY modification. Everything else allowed. Example:

```
/constraints except .env* src/config.ts
/constraints except **/secrets/**
```

This sets `mode: denylist` and adds the patterns.

#### `/constraints clear`
Remove all constraints. Agent operates freely again.

#### `/constraints show`
Display current constraints to the user.

### 2. Natural language (inline in any message)

The user can declare constraints in plain English anywhere in their message.
You MUST scan EVERY user message for constraint-like language. When detected,
apply it to `.constraints.yaml` immediately, then confirm with the user.

**Allowlist triggers** (→ `mode: allowlist`):

| User says | You apply |
|---|---|
| "make changes in this file only" / "only edit this file" | allowlist: the mentioned file(s) |
| "just work in src/" / "stay in the src folder" | allowlist: `src/**` |
| "only touch X and Y" / "don't go outside X" | allowlist: X, Y |
| "scope your changes to X" / "limit edits to X" | allowlist: X |
| "this file, nothing else" / "X only, leave everything else" | allowlist: X |
| "just this one" / "only here" | allowlist: the file currently being discussed |

**Denylist triggers** (→ `mode: denylist`):

| User says | You apply |
|---|---|
| "don't touch X" / "never edit X" / "leave X alone" | denylist: X |
| "stay out of tests/" / "don't go near config" | denylist: `tests/**` or `config/**` |
| "protect the .env" / "keep your hands off X" | denylist: X |
| "everything except X" / "all files, but not X" | denylist: X |
| "X is off-limits" / "X is frozen" | denylist: X |

**Clear triggers:**

| User says | You apply |
|---|---|
| "remove constraints" / "clear constraints" / "unlock everything" | clear `.constraints.yaml` |
| "you can edit anything now" / "no more restrictions" | clear `.constraints.yaml` |

**Parsing rules for natural language:**

1. **Be greedy** — if the user sounds like they're setting a boundary, apply it.
   Err on the side of protecting files.
2. **Resolve ambiguous paths** — if they say "this file" or "that folder", look at
   what file/directory is currently being discussed or was last mentioned.
3. **Merge, don't replace** — if constraints already exist, merge the new ones in
   (unless the user clearly means to replace, e.g. "actually, only X").
4. **Confirm** — after applying, reply with: "Constraint set: `mode` → `patterns`."
5. **When unclear** — ask: "Did you mean allowlist (only these files) or denylist
   (everything except these)?"
6. **Globs from plain words** — "the src folder" → `src/**`, "all test files" →
   `**/*.test.*`, "config files" → `**/config.*` or `**/config/**`.

**Examples of natural language → constraints:**

```
User: "make changes in src/auth.ts only, don't touch anything else"
→ mode: allowlist, patterns: [src/auth.ts]

User: "refactor the utils but stay out of the tests folder"
→ mode: denylist, patterns: [tests/**]

User: "just this one file, nothing else matters"
→ mode: allowlist, patterns: [<currently discussed file>]

User: "you can edit anything, just don't touch .env or the secrets dir"
→ mode: denylist, patterns: [.env, secrets/**]

User: "forget the constraints, go wild"
→ clear constraints
```

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

### Explicit commands

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

### Natural language

```
User: "make changes in src/auth.ts only, leave everything else alone"
Agent: Constraint set: allowlist → [src/auth.ts]
Agent: [edits src/auth.ts]
Agent: [tries to edit src/utils.ts]
Agent: Constraint blocked: "src/utils.ts" is not in allowed set.

User: "refactor the whole codebase but don't touch anything in the tests folder"
Agent: Constraint set: denylist → [tests/**]
Agent: [edits src/*.ts, config/*.ts — skips tests/**]

User: "just this file, nothing else"
Agent: Constraint set: allowlist → [<file currently being discussed>]
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
