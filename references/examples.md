# Review Examples

## Good Finding (report this)

> **[High]** `src/auth/session.ts:47` — `refreshToken()` catches the JWT expiry error but returns `null` instead of throwing. Callers at `src/api/middleware.ts:23` and `src/api/middleware.ts:89` don't check for `null`, so expired sessions silently pass through auth middleware.

Why it's good:
- Names exact files and lines
- Traced the caller chain
- Describes the concrete break path
- A senior engineer would agree

## Bad Finding (do NOT report this)

> **[Medium]** `src/utils/helpers.ts:12` — This function could potentially return undefined if the input is malformed.

Why it's bad:
- "Could potentially" = speculation
- Didn't check if any caller passes malformed input
- Didn't check if there's validation upstream
- No concrete break path

## Bad Finding #2 (do NOT report this)

> **[Low]** `src/config.ts:5` — Consider using `const` instead of `let` here.

Why it's bad:
- Style preference, not a bug
- Wastes reviewer attention
- Adds noise, no signal

## Docs-Only PR

> ## PR Review: Update README (#42)
> **Verdict:** APPROVE
> Docs-only change. No code, config, or behavioral changes. No concerns.
