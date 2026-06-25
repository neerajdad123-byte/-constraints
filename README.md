# /constraints — agent skill

The last word before any file changes on disk. Hard boundaries agents CANNOT cross.

```
npx skills add neerajdad123-byte/-constraints
```

## Two ways to lock it down

### Explicit commands
```
/constraints only src/auth.ts        # agent can ONLY edit auth.ts
/constraints except .env* secrets/   # agent must NEVER touch these
/constraints show                    # see what's locked
/constraints clear                   # unlock everything
```

### Natural language — just say it
```
"make changes here only"            → allowlist on current file
"don't touch the config"            → denylist on config
"stay out of tests/"                → denylist on tests/**
"just src/, nothing else"           → allowlist on src/**
"remove all constraints"            → clear
```

## What makes it work

| Technique | How it enforces |
|---|---|
| **RFC 2119** language | MUST / NEVER / REQUIRED — non-negotiable |
| **Persistence sweep** | ACTIVE EVERY RESPONSE — no drift |
| **Start-of-turn checkpoint** | reads `.constraints.yaml` first thing every turn |
| **Pre-operation hook** | checks BEFORE every `edit`/`write`/`delete` |
| **Default-deny** | when uncertain → BLOCK, don't risk it |
| **Sub-agent propagation** | constraints flow to every spawned agent |
| **Self-protection** | agent NEVER edits `.constraints.yaml` |
| **Violation protocol** | block → tell user → show context → suggest → continue |

## How it works

Drops a `.constraints.yaml` at workspace root. Agent reads it at the start of
every turn, checks every file operation against it, and blocks violations
with a clear message.

```yaml
# .constraints.yaml
mode: allowlist
patterns:
  - "src/auth.ts"
  - "src/login.ts"
```

## Enforcement is absolute

```
User: make changes in src/auth.ts only
Agent: Constraint set: allowlist → [src/auth.ts]
User: now also update the config
Agent: Constraint blocked: "src/config.ts" is not in the allowed set.
       Active: allowlist → [src/auth.ts]
       Use /constraints show or /constraints clear to adjust.
```

The agent doesn't argue. It doesn't "maybe this once." It blocks, tells you
why, and suggests how to fix it.
