# Security review and hardening

## Threat model

| # | Threat | Control |
| --- | --- | --- |
| 1 | Guessing or enumerating license keys | ≥80-bit random keys; per-IP and per-key rate limits; identical error text for unknown vs. wrong-product keys; unknown keys logged only as hashes; `key_enumeration` abuse signal |
| 2 | Sharing one license across many sites | Domain/IP/directory/machine binding; activation slots; domain and IP churn detection; concurrent-installation sweep |
| 3 | Replaying a captured API request | Timestamp window + single-use nonce (`lfg_api_nonces`, unique index) |
| 4 | Forging a licensing response | Offline payloads signed with Ed25519/RSA; private key encrypted at rest and never served |
| 5 | Running on a revoked license offline | Bounded `offline_until`; SDK refuses expired payloads and re-checks the domain locally |
| 6 | SQL injection | All queries built through the query builder (prepared statements); no string-concatenated SQL anywhere |
| 7 | XSS in the admin or client area | Smarty `\|escape` on all output; `Input::e()` for controller-built fragments |
| 8 | CSRF on state changes | Per-session token required on every POST in both areas |
| 9 | Horizontal access (customer reading another's license) | Every client-area query filters on `client_id` via `ownedLicense()`; denials audited |
| 10 | Source-IP spoofing via proxy headers | `REMOTE_ADDR` is authoritative unless the peer matches `trusted_proxies` |
| 11 | Direct download of protected files | Single-use, expiring tokens; entitlement re-checked at redemption; path traversal blocked; storage root confinement |
| 12 | Timing attacks on secrets | `hash_equals` for keys, signatures, domains and machine IDs; signature computed even for unknown API keys |
| 13 | Secret disclosure via logs | Audit metadata redacts anything matching `secret`, `password`, `token`, `signature`, `private_key`, `authorization` |
| 14 | Information leakage in errors | Fixed error catalogue; exceptions logged server-side, never returned |
| 15 | Accidental destruction of licensing data | Deactivation preserves everything; deletion is soft; destructive uninstall needs an explicit confirmation string |

## Control detail

### Input handling

`LicenseRequest::fromArray()` is the only place untrusted licensing input is processed.
It strips control characters, enforces per-field length caps, normalises domains, IPs and
paths, and limits metadata to 25 scalar entries. Admin and client-area input goes through
`Input::str()`/`int()`/`bool()`/`toList()`, which apply the same discipline.

### Cryptography

| Purpose | Primitive |
| --- | --- |
| Offline payload signing | Ed25519 (libsodium), or RSA-2048/SHA-256 |
| API request authentication | HMAC-SHA256 |
| Secret storage | AES-256-GCM with a 12-byte random nonce and AAD |
| Master key derivation | HKDF-SHA256 over an on-disk secret + the WHMCS instance hash |
| Key/secret lookup hashes | HMAC-SHA256 (not bare SHA-256) |
| Random material | `random_bytes()` / `random_int()` only |

The master key lives in `storage/master.key` (0600) and is combined with a WHMCS instance
value, so a database dump alone does not yield signing keys or API secrets. **Back this file
up** — losing it means regenerating every signing key and API credential.

### Rate limiting

Fixed-window counters keyed by `(scope, identifier)`, one indexed row per bucket:

| Bucket | Default |
| --- | --- |
| `api:{endpoint}` per IP | 120/min validate, 20/min activate |
| `activate:key` per license | 10/hour |
| `api:failures` per IP | 30/5 min → `ABUSE_DETECTED` |
| `reissue:client` per client | 5/hour |

Limits are applied **before** the license lookup, so an attacker cannot use timing or load to
probe key existence. The limiter fails open on infrastructure errors — a database hiccup must
not lock out every paying customer — and the failure is logged.

### Auditing

Every state change writes to `lfg_audit_logs` with actor type/id/name, IP, action, result and
redacted metadata. Auditing is wrapped in its own try/catch so a logging failure can never
break the operation it is recording; such failures go to the PHP error log.

Recorded actions include: license created/updated/deleted/restored, status changes and
*rejected* status changes, activations and releases, reissues (requested/approved/rejected/
completed/denied), feature changes, expiry changes, settings changes, signing-key generation
and activation, API credential lifecycle, authentication failures, CSRF rejections, client
access denials, abuse flags, and cron runs.

## Deployment hardening

1. **HTTPS everywhere.** Enable HSTS. The API already refuses non-loopback plain HTTP.
2. **Block direct access** to `lib/`, `templates/`, `tests/`, `sdk/` and `storage/`. Apache
   `.htaccess` files are shipped; nginx users need the `location` block from
   [installation.md](installation.md#2-install-the-files).
3. **Keep release files outside the web root** and set `download_storage_path` to that
   directory.
4. **Scope API credentials narrowly** — one per product line, no `admin` scope, IP-restricted
   where the caller has a stable address.
5. **Set `trusted_proxies`** if and only if you actually run behind a proxy; leaving it blank
   is the safe default.
6. **Restrict admin access** to the roles that need it (Configure → Access Control).
7. **Monitor** `lfg_abuse_events` and the failed-validation count on the dashboard.
8. **Rotate** signing keys yearly and API secrets when a build is retired or leaked.
9. **Back up** the database *and* `storage/master.key`.

## Known limitations

State these plainly to yourself before shipping:

- **Client-side enforcement is advisory.** Any code running on a customer's machine can be
  patched. Signatures and bindings raise the cost of tampering; they do not make it
  impossible. Design pricing and support expectations accordingly.
- **The API secret in a distributed build is not truly secret.** See the note in
  [integration.md](integration.md#2-configure-the-client). It authenticates the application,
  not the customer.
- **Offline validity is a deliberate trade.** A revoked license keeps working offline until
  `offline_until` passes. Shorten the window if that matters more than outage tolerance.
- **IP locking is fragile** on dynamic addresses, CDNs and load balancers. Domain and machine
  binding are usually better choices.
- **Abuse detection is heuristic.** Review alerts before enabling `abuse_auto_suspend`.

## Verifying the controls yourself

```bash
# Unit suite: key entropy, domain/IP bypasses, signing, offline verification
php modules/addons/licenseforge/tests/run.php

# Confirm no direct SQL string interpolation exists
grep -rn "Capsule::select\|->raw(\|DB::statement" modules/addons/licenseforge/lib/

# Confirm every state-changing admin/client action requires CSRF
grep -rn "requireCsrf" modules/addons/licenseforge/lib/

# Confirm protected directories are not web-reachable
curl -sI https://billing.example.com/modules/addons/licenseforge/lib/Support/Crypto.php
curl -sI https://billing.example.com/modules/addons/licenseforge/storage/master.key
# both must return 403 or 404
```

Manual checks worth running once per release are listed in [testing.md](testing.md#security).
