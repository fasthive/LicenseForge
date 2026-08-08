# Licensing API reference

Base URL (default):

```
https://billing.example.com/modules/addons/licenseforge/api/index.php
```

Endpoints are addressed as `{base}/license/{endpoint}`, or as `{base}?action={endpoint}` if
your environment strips `PATH_INFO`. Both forms are equivalent.

All requests and responses are JSON (`application/json`). Form-encoded bodies are also
accepted. **HTTPS is mandatory** — plain HTTP from a non-loopback address is refused.

---

## Authentication

Requests are signed with HMAC-SHA256. The secret never travels over the wire.

### Canonical string

```
METHOD \n endpoint \n timestamp \n nonce \n sha256_hex(body)
```

- `METHOD` — uppercase HTTP method (`POST`)
- `endpoint` — lowercase endpoint name (`activate`, not `/license/activate`)
- `timestamp` — Unix seconds
- `nonce` — 8–128 URL-safe characters, unique per credential within the skew window
- `sha256_hex(body)` — hex SHA-256 of the exact raw request body (empty string if no body)

### Headers

| Header | Value |
| --- | --- |
| `X-LF-Key` | The public API key (`lfk_…`) |
| `X-LF-Timestamp` | The timestamp used in the canonical string |
| `X-LF-Nonce` | The nonce used in the canonical string |
| `X-LF-Signature` | Lowercase hex `hmac_sha256(secret, canonical)` |

### Reference implementation

```php
$body      = json_encode(['license_key' => 'LIC-…', 'product_id' => 'my-product']);
$timestamp = (string) time();
$nonce     = bin2hex(random_bytes(16));

$canonical = implode("\n", ['POST', 'activate', $timestamp, $nonce, hash('sha256', $body)]);
$signature = hash_hmac('sha256', $canonical, $apiSecret);

$headers = [
    'Content-Type: application/json',
    "X-LF-Key: {$apiKey}",
    "X-LF-Timestamp: {$timestamp}",
    "X-LF-Nonce: {$nonce}",
    "X-LF-Signature: {$signature}",
];
```

```bash
# curl equivalent
TS=$(date +%s); NONCE=$(openssl rand -hex 16)
BODY='{"license_key":"LIC-7F92-A82D-91BC-72F4","product_id":"my-product"}'
HASH=$(printf '%s' "$BODY" | openssl dgst -sha256 -hex | awk '{print $2}')
CANON=$(printf 'POST\nvalidate\n%s\n%s\n%s' "$TS" "$NONCE" "$HASH")
SIG=$(printf '%s' "$CANON" | openssl dgst -sha256 -hmac "$API_SECRET" -hex | awk '{print $2}')

curl -sS -X POST "$BASE/license/validate" \
  -H "Content-Type: application/json" \
  -H "X-LF-Key: $API_KEY" -H "X-LF-Timestamp: $TS" \
  -H "X-LF-Nonce: $NONCE" -H "X-LF-Signature: $SIG" \
  -d "$BODY"
```

### Rules enforced

1. The credential must exist, be active, and not be past its expiry.
2. The source IP must match the credential's allow list, when one is set.
3. `|now − timestamp|` must be within `request_max_skew_seconds` (default 300).
4. The nonce must not have been used before by this credential — replays return
   `REPLAY_DETECTED`. The nonce is only consumed *after* the signature verifies, so a
   third party cannot burn a legitimate client's nonce.
5. The signature must match, compared in constant time.
6. The credential must hold the scope the endpoint requires.

The signature is computed even for an unknown API key, so response timing does not reveal
which keys exist. Authentication failures are audited with only a 12-character key prefix —
never the signature or secret.

Scopes: `activate`, `validate`, `deactivate`, `info`, and `admin` (grants all).

---

## Endpoints

| Endpoint | Methods | Scope | Purpose |
| --- | --- | --- | --- |
| `activate` | POST | `activate` | Bind a license to an installation |
| `validate` | POST, GET | `validate` | Periodic check of an existing installation |
| `heartbeat` | POST, GET | `validate` | Alias of `validate` |
| `deactivate` | POST | `deactivate` | Release an installation slot |
| `info` | POST, GET | `info` | Read license state without a licensing decision |
| `publickeys` | GET | `info` | Public keys for offline verification |
| `ping` | GET | `info` | Service health |

### Request parameters

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `license_key` | string | always | Whitespace and case are normalised |
| `product_id` | string | `activate` | Product slug or WHMCS product ID |
| `domain` | string | when domain-locked | Any URL form; normalised to a bare host |
| `ip` | string | no | A hint only — the observed source IP governs IP locks |
| `directory` | string | when directory-locked | Installation path |
| `machine_id` | string | when machine-locked | 4–128 chars of `[A-Za-z0-9._:-]` |
| `installation_id` | string | no | Derived from machine/domain when omitted |
| `version` | string | no | Software version, e.g. `2.4.0` |
| `metadata` | object | no | ≤25 scalar key/value pairs, ≤190 chars each |
| `reason` | string | `deactivate` only | Recorded in the audit trail |

Aliases are accepted for convenience: `key`/`licensekey`, `product`/`product_slug`,
`host`/`hostname`/`url`, `path`/`install_path`, `machine`/`hardware_id`, `software_version`.

---

### POST /license/activate

```json
{
  "license_key": "LIC-7F92-A82D-91BC-72F4",
  "product_id":  "my-product",
  "domain":      "example.com",
  "directory":   "/var/www/app",
  "machine_id":  "a3f19c…",
  "version":     "2.4.0",
  "metadata":    { "php": "8.2.7", "os": "Linux" }
}
```

Server-side sequence: authenticate → rate-limit → locate license → check status and expiry →
check product → check version → check domain/IP/directory/machine → check activation slots →
create or refresh the activation → sign and return.

**200 OK**

```json
{
  "success": true,
  "status": "active",
  "license": {
    "key": "LIC-7F92-A82D-91BC-72F4",
    "status": "active",
    "product": "My Software",
    "product_id": "my-product",
    "is_trial": false,
    "is_lifetime": false,
    "issued_at": "2026-01-15T09:22:11Z",
    "activated_at": "2026-01-15T09:30:02Z",
    "expires_at": "2027-01-15T00:00:00Z",
    "domain": "example.com",
    "features": ["api_access", "premium_reports"],
    "activations": { "used": 1, "limit": 3 },
    "reissues":    { "used": 0, "limit": 3 },
    "version": {
      "current": "2.4.0", "latest": "2.5.1",
      "minimum": "2.0", "maximum": null, "allowed": "2.x"
    },
    "validation": {
      "interval_hours": 24,
      "last_validated_at": "2026-01-15T09:30:02Z",
      "next_check_after": "2026-01-16T09:30:02Z"
    },
    "activation": {
      "installation_id": "inst-9c2f…",
      "domain": "example.com",
      "ip": "203.0.113.10",
      "machine_id": "a3f19c…",
      "status": "active",
      "activated_at": "2026-01-15T09:30:02Z"
    }
  },
  "offline": {
    "token": "eyJsaWNlbnNlX2lkIjo…​.MEUCIQ…",
    "expires_at": "2026-01-22T09:30:02Z",
    "algorithm": "ed25519",
    "key_id": 1
  }
}
```

Re-activating the same installation is **idempotent**: it refreshes the record and does not
consume another slot.

**403 Forbidden**

```json
{
  "success": false,
  "error": {
    "code": "ACTIVATION_LIMIT",
    "message": "The activation limit for this license has been reached.",
    "details": { "used": 3, "limit": 3 }
  }
}
```

### POST /license/validate

Same parameters (`product_id` optional). Responses match `activate`, plus:

- `"needs_activation": true` — the key is valid but this installation is not yet activated.
- `"grace": {"active": true, "ends_at": "…"}` — expired but inside the grace window.

### POST /license/deactivate

```json
{ "license_key": "LIC-…", "machine_id": "a3f19c…", "reason": "uninstall" }
```

```json
{ "success": true, "status": "active", "message": "The installation has been deactivated." }
```

The license itself stays active; only the installation slot is freed.

### GET|POST /license/info

Returns the same `license` object without making an activation decision — useful for
"licence details" screens inside your product. Never returns customer personal data.

### GET /license/publickeys

```json
{
  "success": true,
  "keys": [
    { "id": 2, "algorithm": "ed25519", "public_key": "3Xk…", "fingerprint": "9a1f…", "active": true },
    { "id": 1, "algorithm": "ed25519", "public_key": "pQ2…", "fingerprint": "44bc…", "active": false }
  ]
}
```

Retired keys are listed so clients can still verify payloads issued before a rotation.
Private keys are never exposed by any endpoint.

### GET /license/ping

```json
{ "success": true, "service": "licenseforge", "version": "1.0.0", "time": "2026-01-15T09:30:02Z" }
```

---

## Error codes

Every failure has the same shape. Branch on `error.code`; treat `error.message` as display
text that may be reworded between releases.

```json
{ "success": false, "error": { "code": "LICENSE_EXPIRED", "message": "The license has expired." } }
```

### Licensing

| Code | HTTP | Meaning |
| --- | --- | --- |
| `INVALID_LICENSE` | 403 | Unknown key, or a key that no longer resolves |
| `LICENSE_EXPIRED` | 403 | Past expiry and any grace window |
| `LICENSE_SUSPENDED` | 403 | Suspended, usually with the WHMCS service |
| `LICENSE_REVOKED` | 403 | Revoked by an administrator |
| `LICENSE_TERMINATED` | 403 | Service terminated |
| `LICENSE_PENDING` | 403 | Issued but not yet active (awaiting payment) |
| `PRODUCT_MISMATCH` | 403 | Key belongs to a different product |
| `DOMAIN_MISMATCH` | 403 | Domain not authorised |
| `IP_MISMATCH` | 403 | Source IP not authorised |
| `DIRECTORY_MISMATCH` | 403 | Installation path differs from the bound one |
| `MACHINE_MISMATCH` | 403 | Machine identifier differs |
| `ACTIVATION_LIMIT` | 403 | No free activation slots (`details.used`, `details.limit`) |
| `ACTIVATION_NOT_FOUND` | 404 | Installation is not activated (or was released) |
| `VERSION_NOT_SUPPORTED` | 403 | Version outside the licensed range |
| `REISSUE_LIMIT` | 403 | Reissue allowance exhausted |
| `REISSUE_COOLDOWN` | 429 | Too soon (`details.available_at`) |
| `REISSUE_NOT_ALLOWED` | 403 | Self-service reissuing disabled |
| `REISSUE_PENDING` | 409 | A request is awaiting approval |

### Protocol and authentication

| Code | HTTP | Meaning |
| --- | --- | --- |
| `INVALID_REQUEST` | 400 | Malformed request |
| `MISSING_PARAMETER` | 400 | Required field absent (`details.missing`) |
| `UNSUPPORTED_ENDPOINT` | 404 | Unknown endpoint (`details.available`) |
| `METHOD_NOT_ALLOWED` | 405 | Wrong HTTP method (see the `Allow` header) |
| `AUTH_REQUIRED` | 401 | No credentials supplied |
| `AUTH_INVALID` | 401 | Unknown, disabled, or malformed credential |
| `AUTH_EXPIRED` | 401 | Credential past its expiry |
| `SIGNATURE_INVALID` | 401 | Signature mismatch |
| `TIMESTAMP_INVALID` | 401 | Clock skew too large |
| `REPLAY_DETECTED` | 409 | Nonce already used |
| `IP_NOT_ALLOWED` | 403 | Credential used from a disallowed IP |
| `SCOPE_DENIED` | 403 | Credential lacks the required scope |

### Throttling and service

| Code | HTTP | Meaning |
| --- | --- | --- |
| `RATE_LIMITED` | 429 | Limit exceeded; honour `Retry-After` |
| `ABUSE_DETECTED` | 429 | Blocked by abuse protection |
| `SERVICE_UNAVAILABLE` | 503 | Licensing disabled or unavailable |
| `INTERNAL_ERROR` | 500 | Unexpected error — details are logged server-side only |

---

## Offline payloads

A successful `activate` or `validate` includes a signed envelope:

```
base64url(json_payload) "." base64url(signature)
```

Payload fields: `license_id`, `license_key`, `product_id`, `customer_id`, `status`,
`expires_at`, `domain`, `ip`, `machine_id`, `installation_id`, `features`, `min_version`,
`max_version`, `allowed_versions`, `issued_at`, `offline_until`, `nonce`, plus `_key_id`
and `_algorithm` so a client can pick the right public key during rotation.

Verify it locally against the public key (see
[integration.md](integration.md#offline-validation)). The private key never leaves the
server and is stored encrypted at rest.

---

## Client behaviour guidance

- **Cache.** Honour `license.validation.interval_hours`; do not call on every page load.
- **Retry.** Retry transport failures and 5xx with backoff. Never retry a 403 — that is a
  decision, not an outage.
- **Rate limits.** On 429, wait for `Retry-After`.
- **Outages.** Fall back to the signed offline payload, and stop when `offline_until` passes.
- **Do not fail open.** If you cannot verify a license, degrade or stop — do not silently
  continue as fully licensed.

## Response guarantees

- `success` is always present and boolean.
- Failures always carry `error.code` and `error.message`; `error.details` is optional.
- Timestamps are ISO-8601 UTC (`YYYY-MM-DDTHH:MM:SSZ`) or `null`.
- No response contains customer personal data, internal IDs beyond the license/customer ID,
  database errors or stack traces.
- Error text is identical for "unknown key" and "key belonging to another product" at the
  lookup stage, so responses cannot be used to enumerate valid keys.
