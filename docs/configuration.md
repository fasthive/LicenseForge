# Configuration reference

Settings live in two places:

- **Addon Modules → LicenseForge → Configure** — a handful of bootstrap fields.
- **Addons → LicenseForge → Settings** — everything below, validated and audited.

Per-product configuration (**Products → Configure a product**) overrides the global values.
Any product field left blank or set to *Inherit global* follows the global default, so you
can change a policy once and have it apply everywhere it has not been deliberately overridden.

Resolution order for any policy value:

```
per-license override  →  product setting  →  global setting  →  built-in default
```

---

## General

| Setting | Default | Meaning |
| --- | --- | --- |
| `license_server_url` | *(blank)* | Public API URL advertised to SDKs. Blank uses `[System URL]/modules/addons/licenseforge/api/index.php`. |
| `module_enabled` | on | Master switch. When off, the API returns `SERVICE_UNAVAILABLE` and hooks stop firing — use during maintenance. |

## License key format

| Setting | Default | Meaning |
| --- | --- | --- |
| `key_prefix` | `LIC` | Fixed prefix. Products can override it, so keys identify the product at a glance. |
| `key_segments` | `4` | Number of random groups. |
| `key_segment_length` | `4` | Characters per group. |
| `key_separator` | `-` | Group separator. |
| `key_alphabet` | `crockford` | `crockford` (base32 without I/L/O/U — resists transcription errors), `hex`, or `alnum`. |
| `key_uppercase` | on | Emit uppercase keys. |

The default (`LIC-XXXX-XXXX-XXXX-XXXX`, Crockford) gives 80 bits of entropy — roughly
1.2×10²⁴ possibilities. Keep at least 3 segments of 4; the Settings page will accept less,
but short keys become guessable and the rate limiter is then your only defence.

Changing the format affects **new keys only**. Existing keys keep working.

## Defaults for new licenses

| Setting | Default | Meaning |
| --- | --- | --- |
| `default_duration_days` | `365` | Validity from issue. `0` issues lifetime licenses. Ignored when the service has a next-due date, which takes precedence so billing and licensing stay aligned. |
| `default_trial_days` | `14` | Duration for licenses flagged as trials. |
| `default_max_activations` | `1` | Concurrent installations allowed. `0` means unlimited. |
| `default_max_reissues` | `3` | Self-service reissues allowed. `0` blocks self-service entirely. |
| `default_grace_days` | `7` | Days a license keeps working after expiry. `0` disables the grace period. |
| `validation_interval_hours` | `24` | How often software should re-check. Returned to the SDK, which uses it as its cache TTL. |
| `offline_validity_days` | `7` | Lifetime of a signed offline payload. Longer = more tolerant of outages, slower to reflect a revocation. |

> **The trade-off to think about:** offline validity is the maximum time a revoked license
> can keep running without network access. Seven days suits desktop software; for
> server-side products consider 1–2 days.

## Binding rules

| Setting | Default | Meaning |
| --- | --- | --- |
| `lock_domain` | on | Bind the license to a domain. |
| `lock_ip` | off | Bind to a source IP. Fine for dedicated servers, painful behind dynamic IPs or CDNs. |
| `lock_directory` | off | Bind to the installation path. Breaks when customers move the install. |
| `lock_machine` | off | Bind to the SDK's machine identifier. |
| `allow_subdomains` | on | `app.example.com` satisfies a license bound to `example.com`. |
| `allow_www_normalisation` | on | Treat `www.example.com` and `example.com` as the same host. |
| `allow_local_domains` | on | Permit `localhost`, `*.test`, `dev.*`, `staging.*` — turn off to force production-only use. |

Domains are normalised (scheme, port, path, case, trailing dot, IDN → punycode) before every
comparison, so case or URL-shape differences cannot bypass a lock.

## Reissuing

| Setting | Default | Meaning |
| --- | --- | --- |
| `reissue_self_service` | on | Customers may reissue from the client area. |
| `reissue_cooldown_hours` | `24` | Minimum interval between customer-initiated reissues. |
| `reissue_requires_approval` | off | Customer requests queue for admin approval instead of applying immediately. |
| `rate_limit_reissue_client` | `5` | Reissue attempts per client per hour. |

Admin-initiated reissues bypass limits, cooldown and approval — deliberately, so support can
always unblock a customer.

## Service status mapping

Each maps a WHMCS service state to a license status. Valid targets: `active`, `pending`,
`suspended`, `expired`, `revoked`, `terminated`.

| Setting | Default |
| --- | --- |
| `map_active` | `active` |
| `map_pending` | `pending` |
| `map_suspended` | `suspended` |
| `map_terminated` | `terminated` |
| `map_cancelled` | `revoked` |
| `map_fraud` | `revoked` |

Mappings are still subject to the transition table: a revoked license will not be silently
reactivated by a service edit, and the attempt is audited.

## Security

| Setting | Default | Meaning |
| --- | --- | --- |
| `require_api_auth` | on | Require signed requests. **Leave this on.** Disabling it means anyone who learns a license key can query it. |
| `signature_algorithm` | `auto` | `auto` prefers Ed25519 when libsodium is present, else RSA. |
| `request_max_skew_seconds` | `300` | Permitted clock difference. Lower is safer; too low breaks clients with drifting clocks. |
| `trusted_proxies` | *(blank)* | IPs/CIDRs whose forwarding headers are honoured. **Blank means no proxy header is ever trusted** — correct unless you actually run behind a proxy. |
| `trusted_proxy_header` | `X-Forwarded-For` | Header consulted for trusted peers. |

## Rate limiting

| Setting | Default | Scope |
| --- | --- | --- |
| `rate_window_seconds` | `60` | Window length for the per-IP buckets. |
| `rate_limit_validate_ip` | `120` | Validation calls per IP per window. |
| `rate_limit_activate_ip` | `20` | Activations per IP per window. |
| `rate_limit_activate_key` | `10` | Activations per license key per hour. |
| `rate_limit_failed_ip` | `30` | Failures per IP per 5 minutes before `ABUSE_DETECTED`. |
| `rate_limit_reissue_client` | `5` | Reissues per client per hour. |

Set any limit to `0` to disable it. Exceeded limits return HTTP 429 with `Retry-After`.

Sizing hint: with a 24-hour check interval, one installation makes ~1 validation call a day.
A limit of 120/minute per IP is generous for shared hosting (many sites, one IP) while still
stopping a scripted attack.

## Abuse detection

| Setting | Default | Meaning |
| --- | --- | --- |
| `abuse_window_hours` | `24` | Analysis window for all signals. |
| `abuse_failed_threshold` | `15` | Failures from one IP before flagging. |
| `abuse_domain_changes` | `5` | Distinct domains for one license before flagging. |
| `abuse_ip_changes` | `10` | Distinct IPs for one license before flagging. |
| `abuse_auto_suspend` | off | Automatically suspend on a high-severity signal. |

> Leave `abuse_auto_suspend` off until you have watched the alerts for a while. Legitimate
> patterns — a customer testing on several staging domains, a hosting migration — can look
> like abuse, and an automatic suspension turns a false positive into a support incident.

## Downloads

| Setting | Default | Meaning |
| --- | --- | --- |
| `download_protection` | on | Enforce license checks. Off makes downloads public — only for free tools. |
| `download_storage_path` | *(blank)* | Root for release files. When set, every configured path is confined beneath it; traversal is rejected. |
| `download_token_ttl` | `900` | Seconds a download link stays valid. Tokens are single-use. |

Keep release files **outside** the web root and set the storage root to that directory.

## Logging and retention

| Setting | Default | Meaning |
| --- | --- | --- |
| `log_validations` | on | Record successful checks too. Turn off on very high-volume installs — failures are always recorded. |
| `validation_log_retention` | `90` | Days of validation history. |
| `audit_log_retention` | `730` | Days of audit history. `0` keeps it forever. |

Volume estimate: 10,000 licenses × 1 check/day ≈ 300,000 rows/month, roughly 60 MB with
indexes. Adjust retention accordingly.

## Notifications

| Setting | Default | Meaning |
| --- | --- | --- |
| `notify_enabled` | on | Master switch for licensing emails. |
| `notify_expiry_days` | `30,14,7,1` | Days before expiry to send reminders. |
| `email_*` | *(names)* | WHMCS General template names — see [installation.md](installation.md#6-email-templates). |

---

## Per-product settings

Everything above that is per-license in nature, plus:

| Field | Meaning |
| --- | --- |
| **Slug** | The `product_id` your software sends. Keep it stable — changing it breaks existing installs. |
| **Key prefix** | Product-specific key prefix. |
| **Latest version** | Reported to clients so they can prompt for updates. |
| **Version rules** | `min_version`, `max_version`, and an allowed-set expression (`1.x, 2.x, 3.0+`). |
| **Default entitlements** | Features granted to every new license of this product. |
| **Activate on provisioning** | Issue as `active` rather than `pending`. |
| **Suspend with the service** | Follow WHMCS suspension. |
| **Expire automatically** | Let cron move due licenses to `expired`. |
| **On upgrade/downgrade** | `carry_over` (retarget the same license), `extend` (also add duration), or `new_license`. |
| **On renewal** | `extend` (default), `reset` (from today), or `none`. |
