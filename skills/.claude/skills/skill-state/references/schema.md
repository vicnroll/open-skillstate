# Extended fields

These fields validate like any other, but they do not appear in `skillstate get` output until they have content. Use them when they apply; leaving them empty costs nothing.

## `hypotheses`

Only hypotheses still operationally useful — ones that would otherwise get re-tested. Each entry is an object:

```bash
skillstate patch --stdin <<'EOF'
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
skillstate patch --stdin <<'EOF'
{ "hypotheses": { "version-outside-critical-section": null } }
EOF
```

Most useful in `mode: exploration`, where the point of the work is narrowing down causes.

## `blockers`

Only unresolved conditions actually preventing progress — an external dependency, a missing credential, a decision only the user can make. Delete each one the moment it clears; a stale blocker is worse than none, because it makes a fresh session believe work is stuck when it is not.

```bash
skillstate patch --stdin <<'EOF'
{ "blockers": { "staging-db-unreachable": "Staging DB refuses connections since the 14:00 deploy" } }
EOF
```

`status: "blocked"` and a populated `blockers` belong together. Setting one without the other leaves the state contradictory.

## `constraints`

Requirements that shape later decisions and are not visible in the code: compatibility guarantees, explicit non-goals, deadlines, or something the user ruled out.

```bash
skillstate patch --stdin <<'EOF'
{
  "constraints": {
    "public-api-stays-compatible": "The public API must not break for 2.x consumers",
    "no-new-dependencies": "User explicitly ruled out adding dependencies for this task"
  }
}
EOF
```

A constraint differs from a decision in where it came from: a decision is something you chose and could revisit with reason; a constraint is imposed from outside and you do not get to relax it on your own.

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

```jsonc
{ "verification": {
    "checks": { "go-test-all": { "name": "go test ./...", "result": "passed", "details": "all packages" } },
    "overall": "passed" } }
```

`result` is `passed`, `failed` or `not_run`. `overall` is `not_run`, `partial`, `passed` or `failed`. Record the outcome, never the raw log.

Setting `status: "completed"` with `verification.overall: "not_run"` is allowed but should be deliberate: it states that the work is done and unverified, which is a real and sometimes correct thing to say.
