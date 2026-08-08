# Testing

## Automated suite

```bash
php modules/addons/licenseforge/tests/run.php
```

39 tests, no database or WHMCS required, so it runs in CI:

```yaml
- name: LicenseForge tests
  run: php modules/addons/licenseforge/tests/run.php
```

Exit code 0 = all passed. Coverage:

| Group | What is asserted |
| --- | --- |
| **License key generation** | Format matches configuration; custom formats honoured; 2,000 keys with no collision; collision retry and clean failure; ≥80 bits of entropy; input normalisation rejects injection-shaped keys |
| **Domain matching** | Scheme/port/path/case/trailing-dot normalisation; suffix attack (`example.com.evil.tld`) blocked; prefix attack (`notexample.com`) blocked; subdomains blocked unless allowed; `*.example.com` covers subdomains but **not** the apex; dev/staging hosts recognised |
| **IP handling** | v4/v6 normalisation, IPv4-mapped IPv6, port stripping; v4 and v6 CIDR matching; forged `X-Forwarded-For` ignored unless the peer is a trusted proxy; path normalisation incl. Windows |
| **Version constraints** | Numeric comparison (`1.2.9 < 1.10.0`); wildcard, `+`, range, comparison and multi-constraint expressions; min/max/allowed checks with reasons |
| **Request normalisation** | Key/domain/directory cleaning; installation ID stable across IP changes; missing-parameter reporting; control characters stripped; metadata capped |
| **Status model** | Transition table enforced; only `active` is usable; status→error-code mapping; every error code has a message and a 4xx/5xx status |
| **API signing** | Signature is canonical (method/endpoint case-insensitive) and deterministic; changing secret, endpoint, timestamp, nonce or body changes it; **SDK and server produce identical signatures** |
| **Offline signatures** | Genuine Ed25519 payload verifies; payload tampered under the old signature is rejected; malformed and corrupted tokens rejected; SDK rejects a payload signed by a different key; SDK refuses to verify with no public key configured; SDK refuses a plain-HTTP server URL |
| **Input handling** | XSS payloads escaped; list parsing and de-duplication; date validation and ISO-8601 conversion; machine-ID validation |
| **Secret storage** | AES-256-GCM round trip; unique ciphertexts; tampered ciphertext rejected; constant-time comparison; 500 unique URL-safe tokens |

### Adding a test

```php
T::group('My area');

T::run('does the thing', static function (): void {
    T::assertSame('expected', subject(), 'What this proves');
});
```

Assertions available: `assertTrue`, `assertFalse`, `assertSame`, `assertNotSame`,
`assertNull`, `assertNotNull`, `assertMatches`, `assertThrows`.

---

## Manual test cases

Run these against a staging WHMCS after installation or upgrade. Each is written so a
non-author can execute it.

### License creation

| # | Steps | Expected |
| --- | --- | --- |
| C1 | Configure a product, order it as a test client, mark the invoice paid | A license appears under Licenses with status `Active`, linked to the client and service |
| C2 | Issue 20 licenses manually from the Licenses page | 20 distinct keys, all matching the configured format |
| C3 | Create a license for a product with default entitlements set | The license shows those entitlements on its detail page |
| C4 | Order a product with licensing **disabled** | No license is created |

### Activation

| # | Steps | Expected |
| --- | --- | --- |
| A1 | Activate with a valid key, correct product and domain | `success: true`, activation count 1, offline token present |
| A2 | Repeat the identical activation | Still succeeds; activation count stays 1 (idempotent) |
| A3 | Activate from a second installation with limit 1 | `ACTIVATION_LIMIT` with `details.used`/`details.limit`; limit email sent |
| A4 | Activate on a domain other than the bound one | `DOMAIN_MISMATCH` |
| A5 | Activate with `www.` and with a port suffix on the bound domain | Both succeed (normalisation) |
| A6 | Enable IP locking, then activate from a different IP | `IP_MISMATCH` |
| A7 | Enable machine locking, activate, then change `machine_id` | `MACHINE_MISMATCH` |
| A8 | Activate with the wrong `product_id` | `PRODUCT_MISMATCH` |
| A9 | Activate with a version outside the product's rules | `VERSION_NOT_SUPPORTED` |
| A10 | Deactivate the installation, then activate a new one | Succeeds — the slot was freed |

### Validation

| # | Steps | Expected |
| --- | --- | --- |
| V1 | Validate an active, activated license | `success: true`, `last_validated_at` updated |
| V2 | Validate a key that was never activated | `success: true` with `needs_activation: true` |
| V3 | Set the expiry to the past with a grace period configured; validate | Succeeds with `grace.active: true` and an end date |
| V4 | Let the grace window pass, run `cron.php --task=grace`; validate | `LICENSE_EXPIRED` |
| V5 | Suspend the license; validate | `LICENSE_SUSPENDED` **on the very next call** |
| V6 | Revoke the license; validate | `LICENSE_REVOKED`; all activations show `revoked` |
| V7 | Validate a nonexistent key | `INVALID_LICENSE`; `lfg_validations` stores only the key **hash** |

### Reissue

| # | Steps | Expected |
| --- | --- | --- |
| R1 | Customer reissues from the client area | Installations released; reissue count +1; history row; email sent |
| R2 | Reissue again inside the cooldown | `REISSUE_COOLDOWN` with `available_at` |
| R3 | Exhaust the reissue allowance | `REISSUE_LIMIT` |
| R4 | Enable "requires approval"; customer requests a reissue | Status `pending`; nothing released yet; appears on the Reissues page |
| R5 | Approve it | Reissue applies; audit shows `reissue_approved` then `reissued` |
| R6 | Admin reissue with "generate a new key" | New key issued, old key recorded in history and no longer valid |
| R7 | Customer A POSTs a reissue for customer B's license ID | "License not found"; `client.license_access_denied` audited |

### Lifecycle

Walk one license through the whole path and confirm each step is audited:

```
Created → Active → Suspended → Reactivated → Expired → Revoked
```

| # | Trigger | Expected license status |
| --- | --- | --- |
| L1 | Order accepted, invoice unpaid | `Pending` |
| L2 | Invoice paid | `Active` |
| L3 | Service suspended in WHMCS | `Suspended` |
| L4 | Service unsuspended | `Active` |
| L5 | Renewal invoice paid | Expiry extended to the new next-due date; expiry reminder markers cleared |
| L6 | Expiry passes, cron runs | `Expired` (after grace) |
| L7 | Admin revokes | `Revoked`; activations revoked |
| L8 | Service terminated | `Terminated` |
| L9 | Upgrade to another licensed product | Behaviour matches the product's upgrade setting |

### Security

| # | Test | Expected |
| --- | --- | --- |
| S1 | POST to an admin action without `lfg_token` | Rejected with a token error; `csrf.rejected` audited |
| S2 | Replay a valid signed request verbatim | `REPLAY_DETECTED` |
| S3 | Send a request with a timestamp 10 minutes old | `TIMESTAMP_INVALID` |
| S4 | Alter one byte of the body, keep the signature | `SIGNATURE_INVALID` |
| S5 | Use a credential without the `activate` scope | `SCOPE_DENIED` |
| S6 | Use an IP-restricted credential from another IP | `IP_NOT_ALLOWED` |
| S7 | Exceed the per-IP validation limit | HTTP 429 + `Retry-After` |
| S8 | Try 20 random keys from one IP | Rate limited; `key_enumeration` abuse event raised |
| S9 | `license_key=' OR '1'='1` | `INVALID_LICENSE`, no SQL error, nothing in the error log |
| S10 | Set an admin note to `<script>alert(1)</script>`, reload | Rendered as text, not executed |
| S11 | Request `lib/Support/Crypto.php` and `storage/master.key` over HTTP | 403/404 |
| S12 | Reuse a download token | Second attempt refused |
| S13 | Request a download after the license is suspended | Refused at redemption, `download.denied` audited |
| S14 | Point a download's `file_path` at `../../configuration.php` | Rejected by storage-root confinement |
| S15 | Edit a cached offline token's expiry and re-verify | SDK returns invalid |

### Performance smoke test

```bash
# 1,000 sequential validations against staging
time for i in $(seq 1 1000); do ./validate.sh >/dev/null; done
```

Expect a low double-digit millisecond median. Every hot-path lookup is index-backed:
`license_key` (unique), `(license_id, installation_id)` (unique), and the rate-limit `bucket`
(unique). If you see table scans, check that migrations completed
(**Settings → Applied migrations**).

---

## Database-backed integration tests

The unit suite deliberately avoids the database so it runs anywhere. To exercise
`LicenseManager`, `ActivationService` and `ReissueService` end to end, run them inside a
WHMCS bootstrap against a **staging** database:

```php
<?php
require '/path/to/whmcs/init.php';
require '/path/to/whmcs/modules/addons/licenseforge/bootstrap.php';

use LicenseForge\Licensing\{LicenseManager, LicenseRequest, ActivationService, LicenseStatus};

$license = LicenseManager::create(['product_id' => 1, 'client_id' => 1, 'max_activations' => 1]);

$request = LicenseRequest::fromArray([
    'license_key' => $license->license_key,
    'product_id'  => 'my-product',
    'domain'      => 'example.com',
], '203.0.113.10');

assert(ActivationService::activate($request)->isOk());

// A second, different installation must hit the limit.
$second = LicenseRequest::fromArray([
    'license_key'     => $license->license_key,
    'product_id'      => 'my-product',
    'domain'          => 'other.example.com',
    'installation_id' => 'install-2',
], '203.0.113.11');

assert(ActivationService::activate($second)->code() === 'ACTIVATION_LIMIT');

LicenseManager::suspend((int) $license->id, 'test');
assert(ActivationService::validate($request)->code() === 'LICENSE_SUSPENDED');
```

Never run these against production: they create and mutate real licensing records.
