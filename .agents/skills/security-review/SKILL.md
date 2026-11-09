---
name: security-review
description: Review code changes for security defects following OWASP Top 10:2025. Use when adding authentication, handling user input or uploads, touching secrets, creating API endpoints, writing SQL, adding dependencies, or implementing payments and other sensitive features.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated following best security practices described in OWASP TOP 10:2025.
---

# Security Review

A review procedure, not a pen test. Use it on a diff, a feature, or a whole module. It pairs with SAST/DAST/dependency scanners — it does not replace them.

**Review outcome is one of:**

| Severity | Meaning | Action |
|---|---|---|
| **Blocker** | Exploitable now, or no control exists at a trust boundary | Fix before merge |
| **Warning** | Control exists but is partial, bypassable, or unverified | Fix or record with owner |
| **Note** | Hardening / defense in depth | Optional, record if declined |

"Planned later" is not a pass. An accepted deviation needs a `shortcut: <limit>, <when to upgrade>` comment and an entry in the repo's risk list.

## When to Activate

- Adding or changing authentication, authorization, sessions, or MFA
- Accepting user input, file uploads, or callback/webhook payloads
- Adding or changing API endpoints
- Writing SQL, migrations, or dynamic query fragments
- Touching secrets, `.env*`, CI/CD, or deployment config
- Adding or updating dependencies or lockfiles
- Implementing payments, refunds, admin actions, or business limits
- Storing, transmitting, or logging personal data

## How to Review

1. Map the change to the categories below (table) and read only those sections.
2. For each: walk the "Review for" list, then run the probes adapted to the stack.
3. Classify each finding; do not average them.
4. Report as: `A0x [CWE-nnn] file:line — what is wrong — the fix`, plus severity.
5. Re-run the probes that apply before closing the review.

| The change touches | Start at |
|---|---|
| Roles, routes, object ownership, CORS, redirects, paths | A01 |
| Env, headers, error output, static/file serving, defaults | A02 |
| Manifests, lockfiles, base images, CI/CD | A03 |
| Passwords, tokens, keys, encryption, randomness, TLS | A04 |
| Query/command/template construction, output encoding | A05 |
| Payments, business logic, admin flows, uploads, throttling | A06 |
| Login, recovery, session lifecycle, credential policy | A07 |
| Callbacks, deserialization, client-carried state, build artifacts | A08 |
| Logging, audit trails, alerting | A09 |
| Error branches, panics, transactions, resource limits | A10 |

Probes are written for `rg`; use `grep -rn` or PowerShell `Select-String -Path <glob> -Pattern <regex>` if `rg` is unavailable. Treat every probe as a heuristic that produces candidates, not verdicts — multi-line constructs, builders, and ORMs will be missed.

---

## A01 — Broken Access Control

The #1 risk. Authorization must live in server-side code the client cannot influence.

**Review for**

- Deny by default: every route has an explicit access decision, enforced in one reusable place. There is no "authenticated = authorized", and privileged surfaces are unreachable by URL guessing alone.
- Object-level ownership on records: the handler filters by the authenticated subject, not by an id from the request (CWE-639).
- No privilege decision from request body/query/headers, hidden fields, or client state (CWE-602).
- CORS is an allowlist of exact origins; never `*` together with credentials; never "reflect any origin".
- CSRF protection on every state-changing request (token or strict origin/`Sec-Fetch-Site` check) (CWE-352).
- Path handling resolves and confines to the intended root (CWE-22); no client path reaches the filesystem or a template include.
- Outbound requests (webhooks, fetchers, URL previews, image proxies) go to allowlisted hosts over HTTPS — no user-supplied URL (SSRF, CWE-918).

**Probes**

```bash
rg -n 'role|is_admin|can_|permission|authorize' src | rg -v 'test'
rg -n '(req|request|cookies|params|query|body|payload)\.(role|is_admin|user_id|owner_id)' src
rg -n 'Access-Control-Allow-Origin|allow_origin|CorsLayer|CORSMiddleware' src
rg -n '@csrf|csrf|SameSite|X-CSRF|Sec-Fetch-Site' src
rg -n 'FileResponse|send_file|sendFile|readFile|open\(|Path\(' src
rg -n 'http://|https?://' src | rg -v 'localhost|127\.0\.0\.1'
```

## A02 — Security Misconfiguration

**Review for**

- Secrets absent from `.env.example` (placeholders only), images, build args, and static dirs.
- Anything served statically is public by definition: no source, backups, `.env`, source maps, or admin assets in the web root.
- Baseline headers present: `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options`/`frame-ancestors`, `Referrer-Policy`, `Strict-Transport-Security` on HTTPS. A header is never removed to make a feature work.
- Framework defaults that protect (origin checks, host checks, auto-escaping, debug off) stay on.
- Errors to clients carry no stack traces, driver messages, versions, or file paths.
- Cookie attributes are correct by construction and not re-derived per request; `Secure` is a deployment fact.
- No default accounts, sample apps, directory listing, or verbose debug endpoints in production.

**Probes**

```bash
rg -n 'debug\s*[=:]\s*(true|True|1)|DEBUG\s*=' .
rg -n 'Content-Security-Policy|X-Frame-Options|X-Content-Type-Options|Referrer-Policy|Strict-Transport-Security' .
rg -n 'Set-Cookie|cookie\(|set_cookie|HttpOnly|SameSite|Secure' src
rg -n 'permissive|allow_all|origin.?=.?"\*"|\*.*credentials' src
rg -n 'static|public_dir|STATIC_ROOT|serve\(' src
```

## A03 — Software Supply Chain Failures

**Review for**

- Lockfile updated in the same change as the manifest; lockfiles are never hand-edited.
- New dependency justified: maintained, official registry, pinned version, needed (not a wrapper around a few lines).
- Dependency audit and secret scan are CI merge gates and are not disabled, downgraded, or `continue-on-error`.
- Base/runtime images pinned by digest or explicit version, from a trusted registry, non-root, immutable.
- No `curl | sh`, no installs from a URL, fork, or personal namespace when upstream exists.
- Build inputs (action/plugin versions, SBOM tooling) tracked and updated deliberately.

**Probes**

```bash
git diff --name-only | rg '(package\.json|Cargo\.toml|pyproject\.toml|go\.mod|Gemfile|composer\.json|lock)'
rg -n 'no-audit|--ignore|audit-level|continue-on-error|allow-failure' .github .gitlab-ci.yml ci 2>/dev/null
rg -n 'FROM .*:latest|image: .*:latest|curl .*\| *(ba)?sh' .
```

## A04 — Cryptographic Failures

**Review for**

- Passwords with a memory-hard KDF (argon2id, scrypt, PBKDF2-HMAC-SHA-256+) and per-user salt; never reversible, never MD5/SHA1, never encrypted.
- Tokens signed server-side with a key from secret storage; algorithm pinned by the verifier, never taken from the token header (CWE-347). `exp` enforced; `iss`/`aud` validated when multi-tenant.
- Access tokens short-lived; refresh tokens stored hashed, rotated, revocable, and replay-detected.
- Encryption is AEAD (AES-GCM / ChaCha20-Poly1305); unique nonce per key; no ECB, no custom construction, no static IV.
- Randomness for tokens/ids/keys from a CSPRNG — never `Math.random`, `rand()` without crypto seeding, timestamps, or counters.
- Keys 128-bit+ (256 for symmetric), from env/HSM/secret manager, never in source, images, or logs; rotation path exists.
- TLS 1.2+ enforced everywhere, including DB and outbound calls; certificate verification never disabled; no silent HTTP fallback.

**Probes**

```bash
rg -n 'md5|sha1|DES|ECB|CBC|base64\(.*password' -i src
rg -n 'Math\.random|random\.random|rand\(\)|new Random\(|time\(\)' src
rg -n 'verify\s*=\s*false|danger_accept_invalid|InsecureSkipVerify|NoTls|sslmode=disable' .
rg -n 'decode\(|verify\(|Validation|jwt\.' src | rg -i 'alg|secret|key'
```

**Reject / accept**

```python
# FAIL: alg from the token, no required claims
jwt.decode(token, key, algorithms=[jwt.get_unverified_header(token)["alg"]])

# PASS: fixed algorithm, required claims, short expiry
jwt.decode(token, key, algorithms=["RS256"], options={"require": ["exp", "iss", "aud"]})
```

## A05 — Injection

Applies to every interpreter: SQL, NoSQL, OS shell, templates, LDAP, XPath, logs, and HTML.

**Review for**

- Data and command kept separate. SQL uses bind parameters; identifiers (table/column/`ORDER BY`) come from an allowlist `match`/`switch`, never from input (CWE-89).
- Query builders/ORMs used correctly: raw-string escape hatches (`raw`, `literal`, `unsafe`, `executescript`) are reviewed individually.
- No shell/eval/`new Function`/`pickle`/`yaml.load` reachable from input (CWE-95).
- Server-side validation at the trust boundary is positive (allowlist) and applied to every field, including enums and ids.
- Output encoding matches the context (HTML, attribute, URL, JS, CSS); rich text sanitized with a maintained sanitizer before storage/rendering. Raw-HTML escape hatches (`{@html}`, `dangerouslySetInnerHTML`, `v-html`, `<%-`, `|safe`) are justified per use and never fed user content (CWE-79).
- User-supplied filenames/headers cannot inject path traversal or CRLF (CWE-93).
- Template engines do not render user-provided template source; no expression language over input.

**Probes**

```bash
rg -n 'format!\(|f"|% *\(|%s.*%|\+ *["\x27] *(SELECT|INSERT|UPDATE|DELETE|WHERE)' src
rg -n 'query(_one|_opt|_as)?\(|execute\(|executemany\(' src | rg -v '\$[0-9]|%s|\?'
rg -n 'ORDER BY.*\{|LIMIT.*\{|ASC|DESC' src | rg '\{|format|f"'
rg -n 'Command::new|child_process|subprocess|os\.system|exec\(|eval\(|Function\(' src
rg -n 'dangerouslySetInnerHTML|\{@html|<%-|v-html|innerHTML\s*=' web
```

**Reject / accept**

```python
# FAIL: value inside the statement
cur.execute(f"SELECT * FROM users WHERE email = '{email}'")

# PASS: value bound, driver handles quoting (`?`, `$1` elsewhere)
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```

## A06 — Insecure Design

Missing controls cannot be fixed by implementation. Ask what the flow must prevent.

**Review for**

- Every externally triggered state change (payment, refund, order, coupon, invite) verifies authenticity, reconciles server-side truth, and is idempotent — client-supplied status/amount/price is never trusted (CWE-602).
- Business limits (stock, coupon reuse, quantity, value caps, rate of interaction) enforced in the data layer or transaction, not in the UI (CWE-799).
- Anti-automation on abusable flows: login, registration, recovery, checkout, upload, search.
- No client-only enforcement of a security property (CWE-693).
- Uploads: content sniffed by magic bytes, extension/content agree, size capped, stored outside the executable/served namespace, served through a controlled route, no SVG/HTML (CWE-434).
- Privileged flows designed with MFA and least privilege; failure states defined and fail closed.
- Data classified: what is sensitive, where it lives, how long, who may see it — retention and minimization by design.

**Probes**

```bash
rg -n 'amount|price|total|discount|coupon|status' src | rg -i 'body|json|payload|query|params'
rg -n 'rate.?limit|throttl|attempts|lockout' src
rg -n 'upload|multipart|file\.' src
rg -n 'mfa|totp|webauthn|2fa' src
```

**Reject:** `if body.status == "paid": mark_paid(order)` · `price = body.price` · a discount recomputed in the browser.
**Accept:** fetch status from the provider, compare to stored order total, transition in one transaction with an idempotency key.

## A07 — Authentication Failures

**Review for**

- No default, seeded, or sample credentials; no credentials in fixtures that can reach production.
- Login, register, and recovery return identical responses for unknown-user vs wrong-secret (no enumeration, CWE-204/CWE-203).
- Session/token issued fresh on login (no fixation, CWE-384); all sessions invalidated on logout, password change, password reset, email change, and privilege change.
- Account recovery uses high-entropy single-use tokens with short TTL, not JWTs, not knowledge-based questions (CWE-640).
- Throttling and lockout on every credential endpoint, per account and per source, without creating a lockout DoS.
- Password policy follows NIST 800-63b: length over complexity, block breached/common passwords, no forced rotation.
- Token claims validated: signature, expiry, issuer, audience, subject, role. Unknown role → deny.
- Auth library is vetted and current; no hand-rolled comparison (constant-time where relevant), no custom session format.

**Probes**

```bash
rg -n 'password|secret|admin' src | rg -i 'default|seed|fixture|test|hardcod'
rg -n 'Invalid credentials|not found|does not exist|unknown user' src
rg -n 'revoke|invalidate|logout|rotate' src
rg -n 'algorithms|require|audience|issuer|verify_exp|validate_exp' src
```

## A08 — Software or Data Integrity Failures

**Review for**

- Inbound callbacks/webhooks verify a signature or re-query the source before mutating anything.
- Untrusted input is parsed as data only: no native deserialization of client payloads (`pickle`, Java serialization, `yaml.load`, `unserialize`) (CWE-502).
- Client-carried state (steps, carts, selections, feature flags) is re-validated against server state; security decisions never come from it (CWE-565).
- Updates/plugins/actions come from trusted sources and are verified (signature, checksum, pinned digest) (CWE-494).
- No secrets baked into images or build args; artifacts immutable and promoted, not rebuilt per environment.
- No tokens, personal data, or security state in `localStorage`, `sessionStorage`, or URLs (CWE-922).

**Probes**

```bash
rg -n 'pickle|yaml\.load|unserialize|Marshal\.load|ObjectInputStream|readObject' src
rg -n 'localStorage|sessionStorage|history\.(push|replace)State|document\.cookie' web
rg -n 'webhook|callback|signature|hmac|verify' src
rg -n 'ARG |ENV ' Dockerfile* **/Dockerfile 2>/dev/null
```

## A09 — Security Logging & Alerting Failures

Logging exists so an attack is detectable and provable, not so requests can be replayed.

**Review for**

- Security events logged with actor, action, target, source, outcome, correlation id: auth success/failure, authorization denial, privilege change, admin mutation, payment transition, validation/CSRF rejections, rate-limit hits (CWE-778).
- Never logged: passwords, tokens, `Authorization`/`Cookie` values, card data, secrets, full PII, full request bodies of credential/payment endpoints (CWE-532).
- Log forging prevented: untrusted values encoded or emitted as structured fields, never concatenated into the message (CWE-117).
- Audit records are append-only/tamper-evident and retained per the repo's policy, with a purge rule for personal data.
- Thresholds alert a human: burst failed logins, 401/403 spikes, 5xx spikes, gateway failures, unusual admin actions.
- Log sinks are access-controlled; logs are backed up and not only on the compromised host.

**Probes**

```bash
rg -n 'log|logger|tracing|console\.' src | rg -i 'password|token|authorization|cookie|secret|card|rut|cpf'
rg -n 'log.*(\{|\+|format!|f")' src
rg -n 'audit|security event|login.*fail|denied|forbidden' src
```

## A10 — Mishandling Exceptional Conditions

A control that fails open is not a control.

**Review for**

- Errors at a security decision fail closed: deny/log, never fall through to allow, empty-but-valid, or default role (CWE-636).
- No panics/uncaught exceptions from input-derived values: missing fields, bad casts, `unwrap`/`[0]`/`!!`, unchecked null, division, index (CWE-248/CWE-476).
- No blanket catch that swallows an auth/DB/parse failure and returns success (CWE-391).
- Client errors are generic; the cause is logged, not returned (CWE-209).
- Multi-step writes are atomic and roll back on any failure; no half-applied order/payment/stock, no retry that double-applies (CWE-460).
- Missing/extra/invalid parameters produce 4xx, not 5xx.
- Every resource is bounded: body size, upload size, array length, pagination, timeouts, concurrency, retries with backoff (CWE-400).
- Cleanup runs on every path: temp files, file handles, connections, locks.

**Probes**

```bash
rg -n 'unwrap\(\)|expect\(|\.get\(.*\)\.|!!|\[0\]|\.first\(\)' src | rg -v 'test|spec'
rg -n 'except\s*:?\s*(pass|$)|catch\s*\([^)]*\)\s*\{\s*\}' src
rg -n 'except Exception|catch \(.*Exception|catch \(Throwable' src
rg -n 'return (true|ok|success)' src | rg -i 'except|catch|err'
rg -n 'limit|LIMIT|timeout|max_|MAX_' src
```

**Reject / accept**

```python
# FAIL: hides the failure and grants access
try:
    user = repo.get(user_id)
except Exception:
    return {"ok": True}

# PASS: fails closed, logs the cause, generic response
try:
    user = repo.get(user_id)
except RepoError:
    logger.exception("user lookup failed", extra={"user_id": user_id})
    raise HTTPException(status_code=500, detail="Internal error")
```

---

## Pre-Merge Gate

Run the project's own verification commands, plus:

- [ ] Secret scan clean (`gitleaks`/`trufflehog`/platform equivalent)
- [ ] Dependency audit clean for changed manifests (`cargo audit`, `pnpm audit`, `pip-audit`, `govulncheck`, …)
- [ ] Lint/typecheck/tests green, including the security-specific tests touched by the change
- [ ] Probes for the touched categories re-run and explained
- [ ] Every accepted deviation has a `shortcut:` comment or a risk-list entry

Before a deploy, re-walk the A0x sections covering anything that release changed.

## Adapting This Skill to a Repo

Keep this file generic. Project facts belong in the repo's own security doc or area rules, in a short profile like:

```md
Security profile
- Secrets: <store> · rotation: <policy>
- AuthN: <issuer/library/algorithm> · token TTL: <value> · refresh: <storage/rotation>
- AuthZ: <roles> · enforcement point: <middleware/module> · ownership rule: <how>
- Sessions: <cookie names/attributes> · CSRF: <mechanism> · CORS: <allowlist source>
- Crypto: <KDF params> · <AEAD for sensitive columns> · key custody: <where>
- DB: <driver> · bind syntax: <$1 / ? / %s> · validation library: <name>
- Uploads: <limits> · storage/serving: <where>
- CI gates: <secret scan> · <dependency audit> · <SAST>
- Logging: <facility> · retention: <window> · alert channels: <where>
```

Then keep the A01–A10 structure and swap only the probes. Where the project already has a security document (e.g. `docs/security.md`, a compliance or retention matrix, PCI scope), link it: the profile is a cache, that document is the authority.

## Limitations

- Probes find candidates, not proof. Follow the data flow before calling something a finding.
- Not covered: infrastructure hardening, cloud IAM, incident response, privacy-law duties, and language-specific traps (prototype pollution, type juggling, unsafe reflection). Link the project's documents and the language's own checklist instead.

## Resources

- `references/OWASP_TOP10_2025.MD` — full OWASP Top 10:2025 text with prevention guidance and mapped CWEs
- [OWASP Top 10:2025](https://top10.owasp.org/2025/)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [OWASP WSTG](https://owasp.org/www-project-web-security-testing-guide/)
