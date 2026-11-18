# Security Review — PrivateLedger

PrivateLedger is a single-page, fully client-side app (`index.html`). Bank
statements (CSV/PDF) and receipt photos are parsed and categorised entirely in
the browser; the on-device LLM (`WEBLLM_URL`) and OCR (`TESSERACT_URL`) also
run client-side via WebGPU/WASM. There is no backend, no account system, and no
network transmission of financial data — confirmed no `fetch`/`XMLHttpRequest`
call anywhere sends transaction data out; the only outbound requests are to
pinned, versioned CDN scripts (Chart.js, pdf.js, web-llm, Tesseract.js) and
Google Fonts.

## 20-point pen-test checklist

| # | Check | Applicable? | Notes |
|---|-------|-------------|-------|
| 1 | HTTP/2 Rapid Reset (CVE-2023-44487) | N/A | No server component in this repo. |
| 2 | bcrypt 72-byte truncation | N/A | No password hashing. |
| 3 | SAML signature bypass (CVE-2024-45409) | N/A | No SAML/SSO. |
| 4 | nOAuth | N/A | No OAuth/auth. |
| 5 | GraphQL alias batching | N/A | No GraphQL. |
| 6 | TOTP replay (RFC 6238 §5.2) | N/A | No TOTP/2FA. |
| 7 | ReDoS | No | Regexes used (CSV/statement parsing, code normalization) are simple character classes/fixed patterns on user-supplied-but-local data; none exhibit catastrophic backtracking. |
| 8 | JWT algorithm confusion | N/A | No JWTs. |
| 9 | HTTP request smuggling | N/A | No server. |
| 10 | Web cache poisoning | N/A | No server/cache layer. |
| 11 | Spoofed X-Forwarded-For | N/A | No server-side request handling. |
| 12 | Postgres views bypassing RLS | N/A | No database. |
| 13 | `Math.random()` for security tokens | No | `Math.random()` is not used anywhere in this repo. |
| 14 | Timing attacks on secret comparison | No | The Pro "unlock code" check (`applyCode` handler) is a plain `===` string comparison against a hardcoded, non-secret code shown in the repo's own error message — there is nothing secret to time-attack. |
| 15 | Padding oracle | N/A | No encryption/decryption. |
| 16 | Server-side prototype pollution | N/A | No server; no merge of untrusted nested objects. |
| 17 | Server-side template injection | N/A | No server, no templating engine. |
| 18 | XXE | N/A | No XML parsing (CSV/PDF only). |
| 19 | Cross-site WebSocket hijacking | N/A | No WebSockets. |
| 20 | Unicode username normalization | N/A | No usernames/accounts. |

## Findings outside the 20-point list

**1. `esc()` is skipped for the transaction `category` field (`index.html:457`).**
`$("txbody").innerHTML` interpolates `t.category` raw, while `t.desc` goes
through `esc()`. In practice this is not exploitable today: `category` is
always the return value of `categorise()` (`index.html:424-426`), which only
ever returns one of the fixed internal labels in `RULES` (e.g. `"Food"`,
`"Rent"`) or the literals `"Income"`/`"Other"` — never attacker- or
CSV-controlled text. Still, it's an inconsistency worth tightening (wrap it in
`esc()` too) so a future change to `categorise()` (e.g. letting a user rename
or add custom categories) doesn't silently reopen an XSS path. Not applied —
pending your approval.

**2. The Free/Pro gate has no server backing it, because there is no server.**
`CONFIG.PRO_CODE` ("LEDGER-PRO-2026") is a hardcoded cleartext string in
`index.html`, and the wrong-code error message literally echoes it back
(`"That code isn't right. It's LEDGER-PRO-2026"`). Anyone can read the source,
or just fail the check once, to unlock Pro for free locally. Given the
product's whole premise — no account, no server, everything on-device — this
isn't a bypass of a security control so much as the gate having no real
backing at all; flagging it only so it's a conscious choice (e.g. acceptable
for a free/open tool) rather than an oversight, since it's not something a
server-side fix can address without adding the backend this app deliberately
avoids.

Neither item is applied — both are deferred to your review per your standing
instructions.
