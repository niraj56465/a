# Review Examples

## Good Bug + Contract Finding (report this)

```
- [High] `src/auth/session.ts:47` — expired sessions can pass through auth middleware as authenticated.
  Evidence: `refreshToken()` catches JWT expiry and returns `null`; callers in `src/api/middleware.ts:23` and `src/api/middleware.ts:89` dereference the result without a null check.
  Break path: expired token → `refreshToken()` returns `null` → middleware treats missing refresh result as existing session → protected route executes.
  Suggested fix: throw on expired refresh or make both middleware callers explicitly reject `null`.
  Confidence: High
```

Why it's good:
- Names exact files and lines
- Uses the required schema
- Traces the caller chain
- Describes concrete runtime impact

## Good Security + Production-Risk Finding (report this)

```
- [Critical] `src/files/download.ts:61` — user-controlled paths can escape the export directory.
  Evidence: `req.query.path` is joined with `EXPORT_ROOT` but never normalized or checked to remain under `EXPORT_ROOT`.
  Break path: attacker sends `?path=../../.env` → `path.join()` resolves outside export root → handler streams secret file.
  Suggested fix: normalize the resolved path and reject it unless it starts with the normalized export root plus path separator.
  Confidence: High
```

Why it's good:
- Identifies attacker-controlled input and sink
- Shows concrete exploit path
- Gives a specific fix direction

## Bad Finding: Speculation (do NOT report this)

```
- [Medium] `src/utils/helpers.ts:12` — this function could potentially return undefined if the input is malformed.
```

Why it's bad:
- "Could potentially" is speculation
- No caller or input source was checked
- No upstream validation analysis
- No concrete break path

## Bad Finding: Style Nit (do NOT report this)

```
- [Low] `src/config.ts:5` — consider using `const` instead of `let`.
```

Why it's bad:
- Style preference, not a bug
- No runtime or security impact
- Wastes reviewer attention

## Dismissed Finding Example

```
Dismissed: `src/provision.ts:88` appears to use a different host/container path, but `docker-compose.yml` bind-mounts the host path into that container path. Runtime behavior is correct.
```

Why it's dismissed:
- Looks suspicious in isolation
- Architecture/runtime context explains it
- Reporting it would create noise

## Docs-Only PR

```
## PR Review: Update README (#42)

**Verdict:** APPROVE

Docs-only change. No code, config, or behavioral changes. No concerns.
```
