# /constraints — agent skill

Declare file-level constraints agents MUST respect.

```
npx skills add neerajdad123-byte/-constraints
```

## Usage

```
/constraints only src/auth.ts        # agent can only edit auth.ts
/constraints except .env* secrets/   # agent must never touch these
/constraints show                    # see active constraints
/constraints clear                   # remove all constraints
```

## How it works

Drops a `.constraints.yaml` at workspace root. Agents check it before every file edit.
Works across sub-agents and multi-turn sessions.

## Example

```
User: /constraints only src/auth.ts src/login.ts
User: refactor the auth module
Agent: [edits src/auth.ts, src/login.ts — nothing else]
```
