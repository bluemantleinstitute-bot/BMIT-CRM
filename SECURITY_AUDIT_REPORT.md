# SECURITY AUDIT REPORT — WACRM (WhatsApp CRM)

**Audit Date:** July 25, 2026
**Auditor:** Cybersecurity Analysis (Automated)
**Scope:** Full codebase static analysis — `src/`, `mcp-server/`, `supabase/`, config files
**Methodology:** OWASP Top 10, CWE/SANS 25, secrets scanning, auth/authz review, dependency audit

---

## EXECUTIVE SUMMARY

| Severity | Count |
|----------|-------|
| CRITICAL | 2 |
| HIGH | 4 |
| MEDIUM | 8 |
| LOW | 11 |
| INFORMATIONAL | 7 |

**Overall Assessment:** The WACRM codebase demonstrates **strong security engineering** for a self-hosted CRM. Encryption at rest (AES-256-GCM), Supabase RLS for multi-tenant isolation, webhook HMAC verification, SSRF protection, and comprehensive RBAC are well-implemented. However, **two critical and four high-severity issues** require immediate remediation — most notably the complete absence of CSRF protection and exposed production secrets.

---

## CRITICAL SEVERITY

### C-1: No CSRF Protection on Any State-Changing Endpoint

**CWE-352: Cross-Site Request Forgery**
**CVSS: 8.8 (High)**

**Affected:** All 48 POST/PUT/PATCH/DELETE API routes — zero CSRF tokens, zero `SameSite` enforcement.

The application authenticates via Supabase SSR cookies. Browsers automatically attach cookies to same-origin requests. A malicious page can trigger state-changing requests from any authenticated user.

**High-risk attack scenarios:**
- `POST /api/account/transfer-ownership` — transfer account ownership
- `DELETE /api/account/api-keys/[id]` — revoke all API keys
- `POST /api/whatsapp/broadcast` — send unauthorized broadcasts
- `POST /api/account/invitations` — invite attacker-controlled emails
- `PATCH /api/account/members/[userId]/role` — escalate privileges
- `POST /api/whatsapp/send` — send messages as the victim

**Location:** Every route under `src/app/api/` except `api/whatsapp/webhook` (which uses HMAC).

**Recommendation:**
1. Implement CSRF tokens (e.g., `csrf-csrf` library or double-submit cookie pattern)
2. Add `SameSite=Strict` or `SameSite=Lax` to all auth cookies
3. Validate `Origin`/`Referer` headers as defense-in-depth

---

### C-2: Production Secrets Exposed in `.env.local`

**CWE-798: Use of Hard-coded Credentials**
**CVSS: 9.1 (Critical)**

The `.env.local` file contains live production secrets:
- Supabase anon key (JWT)
- Supabase service-role key (bypasses ALL RLS — full DB access)
- AES-256-GCM encryption key (`ENCRYPTION_KEY`)
- Meta App Secret (`META_APP_SECRET`)

The `.gitignore` correctly excludes `.env*`, but if this file was ever committed (e.g., `git add -f .env.local`), **all secrets are permanently compromised in git history**.

**Impact:** Service-role key alone grants unrestricted read/write to all database tables, bypassing all RLS policies.

**Recommendation:**
1. Verify this file has NEVER been committed: `git log --all --full-history -- .env.local`
2. If committed, rotate ALL secrets immediately and purge git history
3. Add `.env.local` to a pre-commit hook blocklist
4. Use a secrets manager (Vault, Doppler, or Supabase Vault) for production deployments

---

## HIGH SEVERITY

### H-1: Middleware Only Protects `/api/whatsapp/*` — All Other API Routes Unprotected at Gateway Level

**CWE-862: Missing Authorization**
**CVSS: 7.5**

The Next.js middleware (`src/middleware.ts:80-86`) only enforces auth on `/api/whatsapp/*` (excluding `/webhook`). All other API routes (`/api/account/*`, `/api/ai/*`, `/api/automations/*`, `/api/flows/*`, `/api/v1/*`, `/api/invitations/*`, `/api/quick-replies/*`) rely **entirely on internal auth checks** within each route handler.

**Risk:** Any new route that forgets to call `getCurrentAccount()`, `requireRole()`, or `requireApiKey()` is **completely unprotected** — no defense-in-depth.

**Affected routes (45+):**
- `src/app/api/account/route.ts`
- `src/app/api/account/members/route.ts`
- `src/app/api/ai/config/route.ts`
- `src/app/api/automations/route.ts`
- `src/app/api/flows/route.ts`
- `src/app/api/v1/contacts/route.ts`
- (and ~39 more)

**Recommendation:**
Expand middleware to protect all `/api/*` routes by default, with explicit exemptions for public endpoints (webhook, invitation peek).

---

### H-2: Database Error Messages Leaked to Clients

**CWE-209: Generation of Error Message Containing Sensitive Information**
**CVSS: 6.5**

Multiple API routes return raw `error.message` from database operations in 500 responses. This leaks internal schema details, SQL error text, and database structure.

**Affected files:**
| File | Line(s) | Leaked Content |
|------|---------|----------------|
| `src/app/api/flows/route.ts` | 43 | `error.message` |
| `src/app/api/flows/[id]/route.ts` | 143, 154, 168, 210 | `error.message` |
| `src/app/api/flows/[id]/activate/route.ts` | 116 | `error.message` |
| `src/app/api/flows/[id]/runs/route.ts` | 56 | `error.message` |
| `src/app/api/flows/cron/route.ts` | 63 | `error.message` |
| `src/app/api/quick-replies/route.ts` | 19, 77 | `error.message` |
| `src/app/api/quick-replies/[id]/route.ts` | 80, 101 | `error.message` |
| `src/app/api/automations/route.ts` | 23, 124, 131 | Raw error object |
| `src/app/api/automations/[id]/route.ts` | 39, 123, 128, 156 | `error.message` |
| `src/app/api/automations/[id]/duplicate/route.ts` | 34, 53, 82 | `error.message` |
| `src/app/api/automations/cron/route.ts` | 36 | `error.message` |

**Contrast with good practice:** `src/lib/auth/account.ts:69-74` returns `"Internal server error"`.

**Recommendation:** Replace all `error.message` in 500 responses with generic messages. Log the actual error server-side.

---

### H-3: CSP is Report-Only (No Enforcement)

**CWE-693: Protection Mechanism Failure**
**CVSS: 6.1**

The Content-Security-Policy is set as `Content-Security-Policy-Report-Only` (`next.config.ts:39`). Violations are logged but **not blocked**. Even if an XSS payload executes, CSP will not prevent it.

Additionally, the CSP includes `'unsafe-inline'` and `'unsafe-eval'` in `script-src`, which significantly weakens any future enforcement.

**Recommendation:**
1. After validation period, flip to `Content-Security-Policy`
2. Migrate to nonce-based CSP to eliminate `unsafe-inline`
3. Remove `unsafe-eval` (no application code uses `eval()`)

---

### H-4: Next.js High-Severity Vulnerabilities (16 dependencies)

**CWE-1395: Dependency on Vulnerable Third-Party Component**
**CVSS: 7.5 (combined)**

`npm audit` reports **16 vulnerabilities (3 moderate, 13 high)**:

| Package | Severity | CVE/Advisory | Description |
|---------|----------|--------------|-------------|
| `next` | HIGH | GHSA-6gpp-xcg3-4w24 | Middleware/proxy bypass via Turbopack |
| `next` | HIGH | GHSA-m99w-x7hq-7vfj | DoS via Server Actions |
| `next` | HIGH | GHSA-89xv-2m56-2m9x | SSRF in Server Actions |
| `next` | HIGH | GHSA-68g3-v927-f742 | Cache confusion with request bodies |
| `next` | HIGH | GHSA-4633-3j49-mh5q | Cache confusion with invalid UTF-8 |
| `next` | HIGH | GHSA-4c39-4ccg-62r3 | Unbounded Server Action payload |
| `next` | HIGH | GHSA-p9j2-gv94-2wf4 | SSRF in rewrites |
| `next` | HIGH | GHSA-q8wf-6r8g-63ch | DoS via SVG image optimization |
| `next` | HIGH | GHSA-955p-x3mx-jcvp | Unauthenticated endpoint disclosure |
| `sharp` | HIGH | GHSA-f88m-g3jw-g9cj | libvips CVEs (CVE-2026-33327+) |
| `postcss` | HIGH | GHSA-r28c-9q8g-f849 | Path traversal via source maps |
| `brace-expansion` | HIGH | GHSA-mh99-v99m-4gvg | DoS via unbounded expansion |
| `@hono/node-server` | MODERATE | GHSA-frvp-7c67-39w9 | Path traversal on Windows |

**Recommendation:** Run `npm audit fix --force` to update to patched versions (requires breaking changes testing).

---

## MEDIUM SEVERITY

### M-1: `javascript:` Protocol Injection via Unvalidated URLs

**CWE-79: Cross-site Scripting (Reflected/Stored)**

User-influenced URLs rendered in `href`/`src` attributes without scheme validation:

| Location | Line | Attribute | Source |
|----------|------|-----------|--------|
| `src/components/inbox/message-bubble.tsx` | 184 | `<a href={message.media_url}>` | WhatsApp API / DB |
| `src/components/inbox/contact-sidebar.tsx` | 142 | `<img src={contact.avatar_url}>` | User-settable |
| `src/components/inbox/conversation-list.tsx` | 465 | `<img src={contact.avatar_url}>` | User-settable |
| `src/components/layout/header.tsx` | 87 | `<img src={profile.avatar_url}>` | User-settable |
| `src/components/layout/sidebar.tsx` | 338 | `<img src={profile.avatar_url}>` | User-settable |
| `src/components/settings/members-tab.tsx` | 359 | `<img src={member.avatar_url}>` | User-settable |
| `src/components/flows/forms/node-config-form.tsx` | 982 | `<a href={cfg.media_url}>` | Admin-settable |

**Impact:** If `media_url` or `avatar_url` contains `javascript:alert(document.cookie)`, clicking the link executes arbitrary JS in the user's session.

**Recommendation:** Add URL scheme validation:
```ts
function safeUrl(url: string): string | undefined {
  try {
    const parsed = new URL(url, window.location.origin);
    if (['http:', 'https:', 'blob:'].includes(parsed.protocol)) return url;
  } catch {}
  return undefined;
}
```

---

### M-2: Cookie Security Flags Not Explicitly Set

**CWE-614: Sensitive Cookie in HTTPS Session Without 'Secure' Attribute**

The application delegates cookie configuration to Supabase SSR defaults. No explicit `SameSite`, `HttpOnly`, or `Secure` flags are set in `src/middleware.ts` or `src/lib/supabase/server.ts`.

Supabase defaults are typically `httpOnly: true`, `secure: true`, `sameSite: 'lax'` — but this is library behavior, not application-enforced.

**Recommendation:** Explicitly set cookie flags:
```ts
cookies: {
  setAll(cookiesToSet) {
    cookiesToSet.forEach(({ name, value, options }) => {
      request.cookies.set(name, value);
      supabaseResponse.cookies.set(name, value, {
        ...options,
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'lax',
        path: '/',
      });
    });
  },
}
```

---

### M-3: Invitation URL Host Derived from Attacker-Controlled Headers

**CWE-601: Open Redirect**

When `NEXT_PUBLIC_SITE_URL` is not set, `getBaseUrl()` in `src/app/api/account/invitations/route.ts:94-135` trusts `X-Forwarded-Host` and `Host` headers. An attacker POSTing to `/api/account/invitations` with `Host: phishing.example.com` generates invite URLs pointing at their domain.

**Mitigation:** `NEXT_PUBLIC_SITE_URL` is set in `.env.local` (line 43), and `ALLOWED_INVITE_HOSTS` provides additional defense. But a deployment that forgets both variables is vulnerable.

**Recommendation:** Make `NEXT_PUBLIC_SITE_URL` required (fail startup if missing) or always validate against `ALLOWED_INVITE_HOSTS`.

---

### M-4: PostgREST `.or()` Filter Interpolation with User-Controlled Search

**CWE-89: Improper Neutralization of Special Elements used in an SQL Command ('SQL Injection') (Low risk)**

`src/app/api/v1/contacts/route.ts:59` interpolates a sanitized search string into a PostgREST `.or()` filter:
```ts
query = query.or(`name.ilike.*${search}*,phone.ilike.*${search}*`);
```

The `sanitizeSearch()` function (line 31-33) strips most dangerous characters but allows `.` and `@`, which could theoretically interact with PostgREST grammar.

**Current exploitability:** Very low — the sanitizer strips `,`, `(`, `)`, and `*` which are the critical PostgREST grammar characters.

**Recommendation:** Switch to a whitelist approach — only allow digits, `+`, `-`, `.`, spaces, and Unicode letters.

---

### M-5: Automations Engine Uses Dynamic `.select()` Without Allowlist

**CWE-89: Improper Neutralization of Special Elements used in an SQL Command**

`src/lib/automations/engine.ts:669` passes `cfg.operand` (from automation config) directly into `.select()`:
```ts
.select(cfg.operand)
```

The flows engine (`src/lib/flows/engine.ts:492-497`) has an explicit allowlist, but the automations engine does not.

**Current exploitability:** Low — the value comes from the database (admin-configured), and PostgREST rejects invalid column names. But the service-role client bypasses RLS.

**Recommendation:** Add allowlist: `const ALLOWED = ['name', 'email', 'company']; if (!ALLOWED.includes(cfg.operand)) throw new Error('Invalid operand');`

---

### M-6: PostgREST Cursor Interpolation

**CWE-89: Improper Neutralization of Special Elements used in an SQL Command**

`src/lib/api/v1/pagination.ts:106` interpolates cursor values into `.or()` filters:
```ts
return `created_at.lt.${cursor.createdAt},and(created_at.eq.${cursor.createdAt},id.lt.${cursor.id})`;
```

**Current exploitability:** Very low — cursor values are base64url-encoded and validated against strict UUID/ISO-8601 regex before interpolation.

**Recommendation:** Already well-defended. Document the security consideration explicitly in code.

---

### M-7: Meta API Error Messages Leaked to Clients

**CWE-209: Generation of Error Message Contensing Sensitive Information**

WhatsApp config and template routes forward Meta API error messages directly to the client, potentially revealing API internals, token validity, and WABA configuration details.

**Affected files:**
| File | Line(s) |
|------|---------|
| `src/app/api/whatsapp/config/route.ts` | 146, 248, 322 |
| `src/app/api/whatsapp/config/verify-registration/route.ts` | 111, 134 |
| `src/app/api/whatsapp/templates/submit/route.ts` | 256 |
| `src/app/api/whatsapp/templates/[id]/route.ts` | 226, 325 |
| `src/app/api/whatsapp/templates/sync/route.ts` | 317 |

**Recommendation:** Log Meta errors server-side; return generic "Meta API error" to client.

---

### M-8: Inconsistent Cron Endpoint Auth (Timing-Safe vs Plain Comparison)

**CWE-208: Observable Timing Discrepancy**

`src/app/api/automations/cron/route.ts:23` uses plain string comparison (`supplied !== expected`), while `src/app/api/flows/cron/route.ts:38-46` uses `crypto.timingSafeEqual`.

**Recommendation:** Use `timingSafeEqual` in both endpoints.

---

## LOW SEVERITY

### L-1: Legacy CBC Decryption Without MAC

**CWE-353: Missing Support for Integrity Check**

`src/lib/whatsapp/encryption.ts:79-96` — Legacy CBC decryption path has no MAC verification. New encryptions use AES-256-GCM (authenticated), but existing legacy-format ciphertext is vulnerable to manipulation.

### L-2: ILIKE Wildcard Injection in Broadcast Audience Filter

**CWE-89: Improper Neutralization of Wildcards**

`src/hooks/use-broadcast-sending.ts:306` — `%` and `_` in filter values are not escaped before ILIKE interpolation. Client-side only; affects the user's own broadcast audience selection.

### L-3: Client-Side-Only File Upload Size/Type Validation

**CWE-434: Unrestricted Upload of File with Dangerous Type**

`src/lib/storage/upload-media.ts:17-34` — MIME type and size limits validated client-side only. Supabase Storage bucket provides server-side 16 MB backstop.

### L-4: DNS Rebinding Residual Risk in Webhook SSRF Protection

**CWE-918: Server-Side Request Forgery**

`src/lib/webhooks/ssrf.ts:16-18` — DNS rebinding (public→private flip between check and connect) is documented as not defended against.

### L-5: DNS Re-Resolution Race Between SSRF Check and Fetch

**CWE-367: Time-of-check Time-of-use (TOCTOU)**

DNS is resolved twice — once in SSRF check (`ssrf.ts:77`) and again in `fetch` (`deliver.ts:114`). Race condition possible but mitigated by `redirect: 'manual'`.

### L-6: Verify Token Comparison Uses `===` (Not Constant-Time)

**CWE-208: Observable Timing Discrepancy**

`src/app/api/whatsapp/webhook/route.ts:125` — GET verification uses `===` instead of `timingSafeEqual`. POST signature verification correctly uses constant-time comparison.

### L-7: Database Schema Fully Exposed in Source Control

**CWE-200: Exposure of Sensitive Information**

35 migration files in `supabase/migrations/` contain complete DDL. Anyone with repo access can reconstruct the entire schema and RLS policies.

### L-8: DRY_RUN Env Var Accessible in Production

**CWE-489: Active Debug Code**

`src/app/api/whatsapp/templates/submit/route.ts:141-143` — `WHATSAPP_TEMPLATES_DRY_RUN` can be set in production to bypass Meta API calls.

### L-9: Console.log Leaking PII (Phone Numbers)

**CWE-532: Insertion of Sensitive Information into Log File**

`src/lib/whatsapp/send-message.ts:434-436` — Logs phone numbers (PII) to server logs.

### L-10: Media Route mediaId Not Sanitized

**CWE-88: Improper Neutralization of Argument delimiters in a Command**

`src/app/api/whatsapp/media/[mediaId]/route.ts:11,68` — mediaId passed directly to Meta API URL without path traversal sanitization. Meta API rejects invalid IDs.

### L-11: ENCRYPTION_KEY Uses Non-Null Assertion

**CWE-252: Unchecked Return Value**

`src/lib/whatsapp/encryption.ts:29` — `process.env.ENCRYPTION_KEY!` will cause cryptic failure at encrypt/decrypt time rather than at startup if missing.

---

## INFORMATIONAL

### I-1: No `eval()`, `exec()`, or `child_process` Usage

Zero instances found in application code. Good.

### I-2: No Server Actions

All routes use App Router Route Handlers, not Server Actions. Good.

### I-3: No `innerHTML`/`outerHTML` Assignments

All user content rendered through React JSX (auto-escaped). Good.

### I-4: No `postMessage` Handlers Without Origin Validation

No postMessage handlers found. Good.

### I-5: No CORS Misconfiguration

No explicit CORS; same-origin policy enforced by default. Correct for cookie-based auth.

### I-6: `dangerouslySetInnerHTML` Usage (10 instances)

All render developer-controlled translation strings, not user data. Low risk.

### I-7: MCP Server Security

Read-only by default, API key auth, stdio transport (not network-accessible). Well-designed.

---

## POSITIVE SECURITY PATTERNS (COMMENDABLE)

1. **AES-256-GCM encryption** for all secrets at rest (access tokens, verify tokens, webhook secrets)
2. **API keys stored as SHA-256 hashes** — never plaintext
3. **Comprehensive Supabase RLS** on all tables with account-scoped multi-tenant isolation
4. **Webhook HMAC-SHA256 verification** with constant-time comparison
5. **SSRF protection** with full DNS resolution, private IP blocking, and `redirect: 'manual'`
6. **Security headers** (HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy)
7. **UUID validation** on all route parameters
8. **Rate limiting** on all sensitive endpoints
9. **Centralized RBAC** with single source of truth (`src/lib/auth/roles.ts`)
10. **Webhook auto-disable** on consecutive failures with atomic counting
11. **Constant-time comparison** for webhook signatures and cron secrets (mostly)
12. **Error handling** that hides internal details (mostly — see H-2)
13. **Proper Supabase token rotation** handling in middleware

---

## REMEDIATION PRIORITY

| Priority | Finding | Effort |
|----------|---------|--------|
| P0 (Immediate) | C-1: CSRF Protection | 2-3 days |
| P0 (Immediate) | C-2: Rotate/Secure Secrets | 1 day |
| P1 (This Sprint) | H-4: Update Dependencies | 1 day |
| P1 (This Sprint) | H-2: Fix Error Message Leakage | 0.5 day |
| P1 (This Sprint) | H-3: Enforce CSP | 1 day |
| P2 (Next Sprint) | H-1: Middleware Gateway Auth | 1 day |
| P2 (Next Sprint) | M-1: URL Scheme Validation | 0.5 day |
| P2 (Next Sprint) | M-2: Cookie Security Flags | 0.5 day |
| P2 (Next Sprint) | M-3: Required SITE_URL | 0.5 day |
| P3 (Backlog) | M-4 through M-8 | 1 day total |
| P3 (Backlog) | L-1 through L-11 | 2 days total |

---

## DEPENDENCIES AUDIT SUMMARY

```
16 vulnerabilities (3 moderate, 13 high)

Key packages requiring update:
- next: 16.2.6 → 16.2.11+ (9 high-severity CVEs)
- sharp: <0.35.0 (4 CVEs in libvips)
- postcss: <=8.5.17 (path traversal)
- brace-expansion: <=5.0.7 (DoS)
- @hono/node-server: <2.0.5 (path traversal on Windows)

Run: npm audit fix --force
```

---

*End of Security Audit Report*
