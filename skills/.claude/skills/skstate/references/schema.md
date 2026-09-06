# Extended fields

These fields validate like any other, but they do not appear in `skstate get` output until they have content. Use them when they apply; leaving them empty costs nothing.

## `hypotheses`

Only hypotheses still operationally useful — ones that would otherwise get re-tested. Each entry is an object:

```bash
skstate patch --stdin <<'EOF'
{
  "hypotheses": {
    "version-outside-critical-section": {
      "statement": "The version increment happens outside the critical section",
      "status": "open"
    }
  }
}
EOF
```

`status` is `open`, `confirmed` or `rejected`. Delete a hypothesis once it no longer prevents repeated work:

```bash
skstate patch --stdin <<'EOF'
{ "hypotheses": { "version-outside-critical-section": null } }
EOF
```

Most useful in `mode: exploration`, where the point of the work is narrowing down causes.

## `blockers`

Only unresolved conditions actually preventing progress — an external dependency, a missing credential, a decision only the user can make. Delete each one the moment it clears; a stale blocker is worse than none, because it makes a fresh session believe work is stuck when it is not.

```bash
skstate patch --stdin <<'EOF'
{ "blockers": { "staging-db-unreachable": "Staging DB refuses connections since the 14:00 deploy" } }
EOF
```

`status: "blocked"` and a populated `blockers` usually belong together. Setting one without the other can leave a confusing state even though the schema permits it.

## `constraints`

Requirements that shape later decisions and are not visible in the code: compatibility guarantees, explicit non-goals, deadlines, or something the user ruled out.

```bash
skstate patch --stdin <<'EOF'
{
  "constraints": {
    "public-api-stays-compatible": "The public API must not break for 2.x consumers",
    "no-new-dependencies": "User explicitly ruled out adding dependencies for this task"
  }
}
EOF
```

A constraint differs from a decision in where it came from: a decision is something you chose and could revisit with reason; a constraint is imposed from outside and you do not get to relax it on your own.

Constraints are promotable under orchestration because an externally imposed requirement normally survives the worker that discovered it.

---

# Core field detail

## `files`

```jsonc
{ "files": {
    "relevant": { "middleware-auth-go": "middleware/auth.go" },
    "modified": { "auth-test-go": "auth_test.go" } } }
```

Keep `modified` aligned with the actual working tree. If a change gets reverted, remove the entry.

## `verification`

Write checks only:

```bash
skstate patch --stdin <<'EOF'
{
  "verification": {
    "checks": {
      "go-test-all": {
        "name": "go test ./...",
        "result": "passed",
        "details": "all packages"
      }
    }
  }
}
EOF
```

A check `result` is `passed`, `failed` or `not_run`.

`verification.overall` is **runtime-owned and derived**. Do not include it in a patch. The CLI computes it from all checks currently in the state:

- no checks, or all `not_run` → `not_run`
- any `failed` → `failed`
- all `passed` → `passed`
- a remaining mix of `passed` and `not_run` → `partial`

Under merge, `verification.checks` is promotable; `overall` is recalculated in the destination after promotion.
