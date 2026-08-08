# Architecture

## Components

```
                     ┌───────────────────────────────┐
   Your software ───▶│  api/index.php                │
   (LicenseClient)   │  Api\Request → Api\Server     │
                     └──────────────┬────────────────┘
                                    │
                     ┌──────────────▼────────────────┐
                     │ Api\Auth   (HMAC + nonce)     │
                     │ RateLimiter (fixed window)    │
                     └──────────────┬────────────────┘
                                    │
                     ┌──────────────▼────────────────┐
                     │ Licensing\ActivationService   │
                     │ Licensing\ValidationService   │◀── ProductConfig (policy resolution)
                     │ Licensing\ReissueService      │
                     └──────────────┬────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
      Licensing\LicenseManager  Support\Audit   Licensing\AbuseDetector
              │                     │                     │
              └─────────────────────┴─────────────────────┘
                                    │
                            Support\Db (Capsule)
                                    │
                              MySQL / MariaDB
                                    ▲
      WHMCS hooks (hooks.php) ──────┤        Admin\Controller ──▶ templates/admin/*.tpl
      WHMCS cron (cron.php)  ───────┘        Client\Controller ─▶ templates/client*.tpl
```

Each layer has one job:

- **`Support\*`** — infrastructure with no licensing knowledge: database access, settings,
  cryptography, rate limiting, auditing, input handling, view rendering.
- **`Database\Schema`** — numbered, idempotent migrations.
- **`Licensing\*`** — the domain. `ValidationService::evaluate()` is the single decision
  point; every endpoint and both UIs go through it.
- **`Api\*`** — HTTP concerns only: parsing, authentication, response shaping, error codes.
- **`Admin\*` / `Client\*`** — controllers that render Smarty templates.

## Request flow: validation

1. `api/index.php` boots WHMCS, refuses non-HTTPS, and captures the request.
2. `Api\Server` routes the endpoint and checks the method.
3. `RateLimiter` applies per-IP and per-endpoint limits *before* any database lookup of the key.
4. `Api\Auth` verifies the HMAC signature, timestamp window, nonce (replay protection),
   IP allow list and scope. The signature is computed even for unknown keys so timing does
   not reveal which API keys exist.
5. `LicenseRequest::fromArray()` normalises every field — this is the only place untrusted
   input is cleaned.
6. `ActivationService` resolves the license and the matching activation.
7. `ValidationService::evaluate()` runs the rule pipeline:
   status → expiry/grace → product → version → domain → IP → directory → machine → slots.
8. The outcome is recorded (`lfg_validations`, counters on the license and activation),
   fed to `AbuseDetector`, and returned as a fixed-shape JSON document.
9. On success the response carries a signed offline payload valid for the configured window.

## Lifecycle

```
  order placed
      │  AcceptOrder / AfterModuleCreate
      ▼
   PENDING ──payment (InvoicePaid)──▶ ACTIVE ──service suspended──▶ SUSPENDED
                                       │  ▲                            │
                          expiry ──────┘  └──── payment / unsuspend ───┘
                              │
                              ▼
                    (grace window, still usable)
                              │
                              ▼
                          EXPIRED ──admin──▶ REVOKED / TERMINATED
```

Transitions are enforced by `LicenseStatus::canTransition()`; a rejected transition is
audited rather than silently ignored. Each transition writes an audit record and may send
a WHMCS email.

## Database schema

All tables are prefixed `lfg_`.

| Table | Purpose | Key indexes |
| --- | --- | --- |
| `lfg_migrations` | Applied schema versions | unique `migration` |
| `lfg_settings` | Global key/value configuration | unique `setting_key` |
| `lfg_signing_keys` | Offline signing key pairs (private key encrypted) | `is_active`, `fingerprint` |
| `lfg_products` | Per-WHMCS-product licensing policy | unique `whmcs_product_id`, `product_slug` |
| `lfg_features` | Feature catalogue | unique `slug` |
| `lfg_licenses` | The licenses | unique `license_key`; `status`, `client_id`, `service_id`, `expires_at`, composite `(client_id,status)` and `(product_id,status)` |
| `lfg_activations` | Installation bindings | unique `(license_id, installation_id)`; `machine_id`, `domain`, `last_validated_at` |
| `lfg_license_features` | Per-license entitlements | unique `(license_id, feature_slug)` |
| `lfg_reissues` | Reissue requests and history | `license_id`, `status` |
| `lfg_validations` | Every licensing call | `license_id`, `ip_address`, `success`, `created_at` |
| `lfg_downloads` | Protected file definitions | `product_id`, `version` |
| `lfg_download_tokens` | Single-use download grants | unique `token_hash`, `expires_at` |
| `lfg_audit_logs` | Audit trail | `action`, `license_id`, `actor_type`, `created_at` |
| `lfg_abuse_events` | Abuse signals | `signal`, `severity`, `resolved` |
| `lfg_notifications` | Email de-duplication | unique `(license_id, notification_key)` |
| `lfg_api_credentials` | API keys and hashed secrets | unique `api_key` |
| `lfg_api_nonces` | Replay protection | unique `nonce_hash`, `expires_at` |
| `lfg_rate_limits` | Fixed-window counters | unique `bucket`, `expires_at` |

### Relationships

```
lfg_products 1───* lfg_licenses 1───* lfg_activations
                      │  │  │
                      │  │  └──* lfg_license_features ──▶ lfg_features (by slug)
                      │  └─────* lfg_reissues
                      └────────* lfg_validations, lfg_audit_logs, lfg_abuse_events

lfg_products 1───* lfg_downloads 1───* lfg_download_tokens ──▶ lfg_licenses
```

Referential integrity is enforced in application code rather than with MySQL foreign keys,
so the module can be installed alongside any WHMCS storage engine configuration and so
license history survives the deletion of a related record.

**Soft deletes.** `lfg_licenses.deleted_at` marks removed licenses. Every query path uses
`whereNull('deleted_at')`; history tables keep their rows. Nothing is ever hard-deleted
except expired rate-limit buckets, nonces, download tokens and log rows past their
retention window.

### Design choices worth knowing

- **Unknown license keys are never stored in clear.** `lfg_validations.license_key_hash`
  holds a SHA-256 hash, so a leaked log cannot be mined for valid or near-miss keys.
- **`installation_id` is derived, not trusted.** If the client does not supply one, it is
  hashed from the machine ID (or domain+directory) so a raw filesystem path never becomes
  a primary key, and so the identifier does not change when the source IP changes.
- **Policy is resolved in one place.** `ProductConfig::policy()` merges product columns
  (nullable = inherit) over global settings; `policyForLicense()` then layers per-license
  overrides on top.
- **Counters are denormalised deliberately.** `activation_count`, `validation_count` and
  `failed_validation_count` live on the license so the hot validation path does not need
  aggregate queries; `recalculateActivationCount()` is the single writer of the first.
