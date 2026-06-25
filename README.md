# /constraints — agent skill

Declare file-level constraints agents MUST respect. Works with explicit commands
AND natural language.

```
npx skills add neerajdad123-byte/-constraints
```

## Two ways to set constraints

### Explicit commands
```
/constraints only src/auth.ts        # agent can only edit auth.ts
/constraints except .env* secrets/   # agent must never touch these
/constraints show                    # see active constraints
/constraints clear                   # remove all constraints
```

### Natural language (just say it)
```
"make changes in this file only"     → allowlist: current file
"don't touch the config"            → denylist: config files
"stay out of the tests folder"      → denylist: tests/**
"just src/, nothing else"           → allowlist: src/**
"remove all constraints"            → clear
```

## How it works

Drops a `.constraints.yaml` at workspace root. Agent checks it before every file
edit — including sub-agents and multi-turn sessions. Natural language is parsed
inline from any message.

## Example

```
User: make changes in src/auth.ts only, don't touch anything else
Agent: Constraint set: allowlist → [src/auth.ts]
User: refactor the auth module
Agent: [edits src/auth.ts — nothing else]
```
