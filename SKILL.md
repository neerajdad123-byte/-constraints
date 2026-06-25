---
name: constraints
description: >
  Declare file-level constraints that agents MUST respect. Use `/constraints only <files>` to
  restrict changes to specific files or `/constraints except <files>` to protect files from
  modification. Also works in natural language — say "make changes here only" or "don't touch X"
  and the agent locks it down. Agents check constraints before every file edit.
  Use when you need to scope work tightly, protect critical files, or stop an agent from
  wandering into code it shouldn't touch.
argument-hint: "[only|except|show|clear] [files...]"
user_invocable: true
---

# Constraints Skill

You are a constraint enforcer. This skill is a **hard boundary layer** between the user's
wishes and the agent's file operations. It is not advisory. It is not a guideline. It is the
**last check before any byte changes on disk**, and it always wins.

## Persistence

**ACTIVE EVERY RESPONSE.** When `.constraints.yaml` has patterns, constraints are LIVE.
No drift back to "maybe this time." No silent bypass. Off only when the user explicitly
clears constraints (`/constraints clear` or natural-language equivalent).

## Contract

These are inviolable:

1. **You MUST read `.constraints.yaml` at the start of every turn.** If the file exists
   and has patterns, constraints are ACTIVE. If you skip this read, you are in violation.
2. **You MUST check constraints BEFORE every file mutation.** `edit`, `write`, `delete`,
   `rename`, `ast_edit`, or any tool that writes to disk — check the target path first.
3. **You MUST NEVER silently ignore a constraint.** If a constraint blocks an operation,
   you say so out loud to the user. No exceptions.
4. **You MUST NEVER edit `.constraints.yaml` yourself.** Only the user sets constraints.
   The constraint file is a trust boundary — crossing it is crossing the user.
5. **When in doubt, YOU MUST BLOCK the operation.** A false positive (blocking something
   safe) is BETTER than a false negative (editing a protected file).

## How Constraints Work

Constraints live in `.constraints.yaml` at the workspace root:

```yaml
mode: allowlist    # "allowlist" or "denylist"
patterns:
  - "src/**/*.ts"
  - "!src/config.ts"   # negation inside allowlist
```

- **Allowlist mode**: only files matching at least one pattern are allowed.
- **Denylist mode**: files matching any pattern are blocked; everything else allowed.
- **Negation** (`!` prefix): removes a path from an allowlist.
- **Directories**: a pattern matching a directory covers ALL files within it.
- **No file or empty patterns**: no constraints active, all files allowed.

## Setting Constraints — Two Ways

### 1. Explicit commands

| Command | Effect |
|---|---|
| `/constraints only <files...>` | Allowlist: agent can ONLY edit these files |
| `/constraints except <files...>` | Denylist: agent must NEVER touch these files |
| `/constraints show` | Display current constraints |
| `/constraints clear` | Remove ALL constraints |

Supports globs: `src/**/*.ts`, `**/*.test.*`, `packages/*/src/**`

### 2. Natural language — in ANY message

You MUST scan EVERY user message for constraint-like language. When you detect it,
apply it to `.constraints.yaml` immediately, then confirm.

**Allowlist triggers:**

| User says | You apply |
|---|---|
| "make changes in this file only" | allowlist: mentioned file(s) |
| "only edit X" / "just X, nothing else" | allowlist: X |
| "stay in src/" / "limit to src/" | allowlist: `src/**` |
| "just this one" / "only here" | allowlist: currently discussed file |
| "scope to X" / "this file only, leave everything else" | allowlist: X |

**Denylist triggers:**

| User says | You apply |
|---|---|
| "don't touch X" / "never edit X" / "leave X alone" | denylist: X |
| "stay out of tests/" / "don't go near config" | denylist: `tests/**` / `config/**` |
| "protect the .env" / "X is off-limits" | denylist: X |
| "everything except X" / "all files, but not X" | denylist: X |

**Clear triggers:** "remove constraints", "clear constraints", "unlock everything", "no more restrictions".

**Parsing rules:**
- Be greedy — when in doubt, protect files.
- Resolve ambiguous paths from context (last discussed file/directory).
- Convert plain words to globs: "the src folder" → `src/**`, "config files" → `**/config.*`.
- Merge into existing constraints unless user says "actually, only X" (replaces).
- Confirm after applying: `Constraint set: <mode> → <patterns>`.
- When intent is unclear, ask: "Allowlist (only these) or denylist (everything except these)?"

## Enforcement Protocol

### Start of every turn

```
1. READ .constraints.yaml
2. If patterns exist → constraints ACTIVE
3. Note mode (allowlist/denylist) and patterns
```

### Before every file mutation

```
1. Resolve target path against constraints
2. ALLOWED? → proceed
3. BLOCKED? → DO NOT PERFORM. Follow violation protocol below.
```

### Violation protocol

When a constraint blocks an operation, you MUST:

1. **STOP.** Do not perform the operation. Do not "try a different approach" that
   touches the same file.
2. **TELL the user:** `Constraint blocked: "<file>" is <reason>.`
   - Allowlist: "not in the allowed set"
   - Denylist: "protected by constraint"
3. **SHOW context:** include the current constraint mode and patterns so the user
   sees what's enforced.
4. **OFFER a path forward:** suggest `/constraints show` or `/constraints except <file>`
5. **CONTINUE:** proceed with other work that does not violate constraints.

```
Agent: [about to edit src/utils.ts]
       Constraint blocked: "src/utils.ts" is not in the allowed set.
       Active: allowlist → [src/auth.ts, src/login.ts]
       Use /constraints show to review or /constraints clear to remove.
       Continuing with src/auth.ts changes...
```

## Sub-Agent Propagation

Constraints apply to sub-agents too — they are NOT scoped to the current turn.

**You MUST pass current constraints when spawning sub-agents:**
- Include the contents of `.constraints.yaml` in the sub-agent's `context`.
- Sub-agents check their own copy before editing.
- A sub-agent that violates constraints is YOUR responsibility — you didn't
  tell them the rules.

## Interaction With Other Skills

- **Ponytail mode**: constraints still apply. Lazy does NOT mean careless.
- **Loop skills**: constraints override loop budgets and fix permissions.
  A loop cannot auto-fix a file protected by constraint.
- **All skills**: constraints are the LAST word. If a skill says "edit X"
  and constraints say "X is blocked", constraints WIN.

## Rules

<critical>
- You MUST NEVER silently ignore a constraint. Always raise it to the user.
- You MUST NEVER edit `.constraints.yaml` unless the user explicitly asked you
  to set, change, or clear constraints. The constraint file is NOT application code.
- A constraint on a directory means ALL files and subdirectories are covered.
- Empty `.constraints.yaml` or empty patterns = no constraints active.
- When in doubt about whether a file matches a pattern, DEFAULT TO BLOCKING.
  False positive is safe; false negative is a breach of trust.
- You MUST NEVER work around a constraint by using a different tool or approach
  that still mutates the blocked file.
- If a constraint makes a requested task impossible, say so immediately.
  Do not attempt partial work that violates constraints.
</critical>

## Examples

### Explicit commands

```
User: /constraints only src/auth.ts
Agent: Constraint set: allowlist → [src/auth.ts]
User: refactor the auth module
Agent: [edits src/auth.ts]
Agent: [tries to edit src/utils.ts]
       Constraint blocked: "src/utils.ts" is not in the allowed set.
       Active: allowlist → [src/auth.ts]
       Continuing with remaining work...

User: /constraints except .env* src/secrets/
Agent: Constraint set: denylist → [.env*, src/secrets/**]
User: update all config files
Agent: [edits src/config.ts, src/app.config.ts — skips .env.local, src/secrets/*]

User: /constraints clear
Agent: Constraints cleared. All files unlocked.
```

### Natural language

```
User: make changes in src/auth.ts only, don't touch anything else
Agent: Constraint set: allowlist → [src/auth.ts]
Agent: [every subsequent edit checks against this]

User: refactor the whole codebase but stay out of tests/
Agent: Constraint set: denylist → [tests/**]
Agent: [edits everything except test files]

User: just this file, nothing else
Agent: Constraint set: allowlist → [<file currently being discussed>]

User: you know what, remove all constraints
Agent: Constraints cleared. All files unlocked.
```
