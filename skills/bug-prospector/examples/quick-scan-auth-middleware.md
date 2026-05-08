# Bug Prospector Report: auth middleware (Quick 3 scan)

> [!NOTE]
> This is a **synthesized example** of bug-prospector's lighter-weight workflow: a single file, the "Quick 3" lens preset, run before opening a PR. The skill's other example file in this directory shows the heavier multi-file, multi-lens workflow on a Swift codebase. This one shows what a focused single-file run looks like on a TypeScript backend file. File paths and findings are illustrative.

**Date:** 2026-04-22
**Scope:** `src/middleware/auth.ts` (single file, ~80 lines, just-finished work)
**Platforms:** Node.js backend, TypeScript
**Lenses Applied:** Assumption Audit · Error Path Exerciser · Boundary Conditions ("Quick 3")
**Files Analyzed:** 1

---

## How this run started

The user finished writing a JWT verification middleware. Instead of opening the PR immediately, they ran bug-prospector against the new file with the "Quick 3" preset (the three lenses with the highest historical bug yield: Assumption + Error Path + Boundary). The full 7-lens run is overkill for a single 80-line file; Quick 3 takes about a minute and surfaces the most common single-file mistakes.

Invocation:

```
/bug-prospector quick src/middleware/auth.ts
```

---

## Summary

| Status | Count |
|--------|-------|
| Bugs Found | 3 |
| Fragile Code | 1 |
| OK (Already Guarded) | 2 |
| Needs Review | 0 |

---

## Issue Rating Table

All BUG and FRAGILE findings rated and sorted by Urgency then ROI:

| # | Finding | Lens | Urgency | Risk: Fix | Risk: No Fix | ROI | Blast Radius | Fix Effort |
|---|---------|------|---------|-----------|-------------|-----|-------------|------------|
| 1 | `auth.ts:34` — `jwt.verify(token, secret)` does not pass an `algorithms` option; accepts any signature algorithm including `none`, allowing forged tokens | Assumption | 🔴 Critical | ⚪ Low | 🔴 Critical | 🟠 Excellent | ⚪ 1 file | Trivial |
| 2 | `auth.ts:48` — `req.headers.authorization` accessed without checking type; if a client sends multiple `Authorization` headers, Node returns an array and `.split(' ')` throws | Boundary | 🟡 High | ⚪ Low | 🟡 High | 🟠 Excellent | ⚪ 1 file | Trivial |
| 3 | `auth.ts:61` — `catch (err) { res.status(401).send() }` swallows the underlying error type; expired tokens, invalid signatures, and malformed JWTs all return identical 401 with no log entry | Error Path | 🟡 High | ⚪ Low | 🟢 Medium | 🟠 Excellent | ⚪ 1 file | Small |
| F1 | `auth.ts:22` — `process.env.JWT_SECRET` read at module load; if env var is missing, the middleware silently uses `undefined` as the secret, which `jsonwebtoken` accepts and produces unverifiable tokens | Assumption | 🟢 Medium | ⚪ Low | 🟡 High | 🟢 Good | ⚪ 1 file | Trivial |

**Urgency scale:** 🔴 CRITICAL (auth bypass / data loss) · 🟡 HIGH (incorrect behavior) · 🟢 MEDIUM (degraded UX or hidden risk) · ⚪ LOW (cosmetic)

---

## Detailed Findings

### 1. `auth.ts:34` — algorithm confusion attack possible

**Lens:** Assumption Audit

**Assumption:** `jwt.verify` will use the same algorithm to verify as was used to sign.

**Violation scenario:** An attacker crafts a JWT signed with the `none` algorithm (no signature) or with HMAC-SHA256 using the public key of an RSA-signed system. By default, `jsonwebtoken` will accept these because the library trusts the token's `alg` header. An attacker who reads the public key from an OIDC discovery endpoint can forge admin tokens.

**Consequence:** Complete authentication bypass. This is a well-documented historical vulnerability class (algorithm confusion) — the fix has been a documented best practice since 2015.

**Current code:**
```typescript
const decoded = jwt.verify(token, JWT_SECRET);
```

**Suggested fix:**
```typescript
const decoded = jwt.verify(token, JWT_SECRET, {
  algorithms: ['HS256'],  // or whichever algorithm you actually use
});
```

The `algorithms` option locks verification to the algorithm(s) you expect. Tokens signed with anything else are rejected before signature verification runs.

---

### 2. `auth.ts:48` — multiple Authorization headers crash the middleware

**Lens:** Boundary Conditions

**Assumption:** `req.headers.authorization` is always a single string.

**Violation scenario:** Most clients send one `Authorization` header. But Node's `http` module concatenates multiple headers with the same name into an array. A malformed client (or an attacker probing the API) can send two `Authorization` headers, causing `req.headers.authorization` to be `string[]` rather than `string`. Calling `.split(' ')` on an array throws a `TypeError`. The middleware crashes; depending on the framework, this can either return a 500 (information leak) or hang the connection (denial of service vector).

**Consequence:** Unhandled crash on hostile input. The fix is to validate the type before using it.

**Current code:**
```typescript
const authHeader = req.headers.authorization;
const [scheme, token] = authHeader.split(' ');
```

**Suggested fix:**
```typescript
const authHeader = req.headers.authorization;
if (typeof authHeader !== 'string') {
  return res.status(401).json({ error: 'invalid_authorization_header' });
}
const [scheme, token] = authHeader.split(' ');
```

The early return handles both "header missing" (`undefined`) and "multiple headers" (`string[]`) with a single check.

---

### 3. `auth.ts:61` — error type lost; debugging is impossible

**Lens:** Error Path Exerciser

**Assumption:** Any 401 from this middleware is good enough information for the client and operator.

**Violation scenario:** A user reports "I keep getting logged out after 30 seconds." The operator looks at the logs. Nothing — the catch block swallows all errors. They can't tell whether tokens are being rejected because of expiration, signature failure, malformed JWT, missing fields, clock skew, or a JWT_SECRET rotation. Every category of failure looks identical from outside.

**Consequence:** Production debugging is impossible. Real auth bugs become irreproducible support tickets. The fix is cheap: classify the error type and log it (without exposing it to the client).

**Current code:**
```typescript
} catch (err) {
  res.status(401).send();
}
```

**Suggested fix:**
```typescript
} catch (err) {
  const reason =
    err instanceof jwt.TokenExpiredError ? 'expired' :
    err instanceof jwt.JsonWebTokenError ? 'invalid' :
    'unknown';
  logger.warn({ reason, requestId: req.id }, 'auth rejected');
  res.status(401).json({ error: reason });
}
```

The client sees a slightly more useful response. Logs distinguish the failure modes. Sentry (or equivalent) captures `unknown` as a real error to investigate.

---

### F1. `auth.ts:22` — JWT_SECRET silently undefined when env var missing

**Lens:** Assumption Audit

**Current behavior:** If `JWT_SECRET` is not in the environment, `process.env.JWT_SECRET` returns `undefined`. The library accepts `undefined` as a secret. Tokens signed with `undefined` are unverifiable, but the failure mode is "verification always fails" rather than a clear startup error.

**Breaking scenario:** A misconfigured deployment (typo in `.env`, missing `kubectl set` step, secret rotation that didn't re-deploy) starts a service that boots, accepts traffic, and rejects every request with a generic 401. Operators see "all auth is broken" with no log line saying *why*, because the actual cause was missing config at startup.

**Recommendation:** Fail at module load if the secret isn't set.

```typescript
const JWT_SECRET = process.env.JWT_SECRET;
if (!JWT_SECRET || JWT_SECRET.length < 32) {
  throw new Error('JWT_SECRET must be set and at least 32 characters');
}
```

The service refuses to start with bad config, instead of starting and silently rejecting all auth.

---

## Already Guarded (Reference)

These were checked by the lenses but found to be correctly handled:

- `auth.ts:38` — Token expiration is honored because `jwt.verify` checks the `exp` claim by default. No additional guard needed.
- `auth.ts:55` — User ID extracted from token is type-checked against the User schema before being used downstream. Mismatch throws and is caught.

---

## Why these three lenses, not all seven

The "Quick 3" preset (Assumption + Error Path + Boundary) catches the highest-yield single-file bugs. For an 80-line file you've just finished writing, the other four lenses are usually overkill:

- **State Machine** — there's no state machine in a stateless middleware
- **Data Lifecycle** — no entities are created, modified, or deleted here
- **Time-Dependent** — token expiration is the only time-dependent piece, and it's handled by the library
- **Platform Divergence** — this is server-side Node.js; no platform branches

Run all 7 when auditing a feature spanning multiple files (the heavier example in this directory shows that). For a single new file before opening a PR, Quick 3 is the right preset.

---

## After You Fix These Findings

bug-prospector finds bug *patterns*. After you fix any BUG finding above, the companion skill [bug-echo](https://github.com/Terryc21/bug-echo) can scan the rest of the codebase for sibling instances:

```
/bug-echo
```

For finding 1 (algorithm confusion) — bug-echo's diff-mode would infer the pattern from the addition of the `algorithms:` option and find every other call to `jwt.verify` in the codebase that doesn't have it. That single sweep could close the bug class everywhere it lives.

For finding 2 (multiple-headers crash) — describe-mode bug-echo with the pattern *"any access to req.headers.<x> as a string without a typeof check"* would find sibling locations.

bug-prospector finds first instances by reasoning about behavior. bug-echo closes bug classes by pattern-matching from a known-good fix. Used together, you find the bug AND every place it lives.

---

*Real bug-prospector quick-scan runs produce reports in this format. The findings here were synthesized for documentation; the report shape, lens taxonomy, and rating table format match what real runs produce.*
