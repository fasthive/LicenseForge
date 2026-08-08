# Troubleshooting

Start here: **Addons → LicenseForge → Audit Log**. Almost every question ("why did this
license change?", "who suspended it?", "why was that request refused?") is answered by
filtering the audit trail on the license ID.

Server-side technical detail goes to the PHP error log prefixed `[LicenseForge]`. API
responses never contain it.

```bash
grep LicenseForge /var/log/php-fpm/error.log | tail -50
```

---

## Installation

**Activation fails with a database error.**
The MySQL user needs `CREATE`, `ALTER` and `INDEX`. Check which migration stopped:
Settings → Applied migrations. Migrations are idempotent — fix the permission and activate
again; completed steps are skipped.

**"The template engine is unavailable."**
`\WHMCS\Smarty` could not be constructed, which normally means the module was loaded outside
a WHMCS request. Confirm you are opening it via `addonmodules.php?module=licenseforge`.

**Signing key generation fails.**
`openssl` (and ideally `sodium`) must be loaded, and `storage/` must be writable by the web
user. Check with `php -m | grep -E 'openssl|sodium'` and `ls -la modules/addons/licenseforge/storage`.

---

## Licenses are not being created

Check in this order:

1. **Is the product configured?** Products tab — it must be listed with licensing *enabled*.
2. **Does the WHMCS product ID match?** The row must point at the product the customer ordered.
3. **Did the service reach a provisioning hook?** Manually setting a service to Active in the
   database bypasses `AfterModuleCreate`. Use the WHMCS interface, or run the invoice-paid
   path.
4. **Is the module enabled?** Settings → `module_enabled`.
5. **Audit log**: filter action `hook.license_provisioned`. If it is absent, the hook never
   fired; if it is present with a failure, the reason is in the metadata.

To backfill a license for an existing service, issue one manually from the Licenses page with
the correct Service ID — the client area and future hooks will pick it up.

---

## API problems

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `AUTH_REQUIRED` | No `X-LF-Key` header | Set `api_key` and `api_secret` in the SDK |
| `SIGNATURE_INVALID` | Canonical string mismatch | The body must be hashed **exactly as sent** — sign the serialised string, not a re-encoded copy. Endpoint is the bare name (`activate`), lowercase |
| `TIMESTAMP_INVALID` | Client clock drift | Run NTP on the client; as a stopgap raise `request_max_skew_seconds` |
| `REPLAY_DETECTED` | Nonce reused | Generate a fresh nonce per request (`bin2hex(random_bytes(16))`) |
| `UNSUPPORTED_ENDPOINT` | `PATH_INFO` stripped by the web server | Use `?action=validate` instead of `/license/validate` |
| `SERVICE_UNAVAILABLE` | Module disabled | Settings → enable the module |
| HTTP 403 "must be accessed over HTTPS" | Request arrived as plain HTTP | Fix TLS; if terminating TLS at a proxy, set `trusted_proxies` and ensure `X-Forwarded-Proto` is passed |
| Empty response body | Fatal error before output | Check the PHP error log for `[LicenseForge]` |

**Debugging a signature mismatch.** Print the canonical string on both sides — they must be
byte-identical:

```php
echo implode("\n", ['POST', 'validate', $timestamp, $nonce, hash('sha256', $body)]);
```

The usual culprit is `json_encode()` being called twice with different flags, producing a
different body from the one that was hashed.

---

## Validation failures

**`DOMAIN_MISMATCH` when the domain looks right.**
Compare what the client actually sent (License detail → Validation history) with the bound
domain. Common causes: the software sends an internal hostname rather than the public
domain; the customer moved from `example.com` to `app.example.com` with subdomains disabled;
or a staging domain is blocked by `allow_local_domains`. Fix by adding the domain to
*Additional domains* on the license, enabling subdomains for the product, or reissuing.

**`IP_MISMATCH` on a working server.**
IP locking rarely survives real hosting — dynamic addresses, load balancers and CDNs all
break it. Prefer domain or machine binding. If you must keep it, add the extra addresses (or
a CIDR range) to *Additional IPs*.

**`ACTIVATION_LIMIT` but the customer says they have one install.**
Old installations that were never deactivated still hold slots. Check the Installations table
on the license detail page — release the stale ones, or run
`php cron.php --task=stale` to release everything that has stopped checking in.

**`MACHINE_MISMATCH` after a server migration.**
The SDK's machine ID derives from hostname + OS + install path, so a migration changes it.
This is what reissuing is for; alternatively reset activations from the admin page.

**`VERSION_NOT_SUPPORTED`.**
Compare the version the client sent against the product's rules. Remember that `1.x` does not
match `2.0`, and `3.0+` means "3.0 or newer".

---

## Client area

**"My Licenses" is empty.**
The license's `client_id` must match the logged-in user. Check the license detail page; if it
shows a different client (or `0` for a manually created license), correct it.

**The menu item is missing.**
The hook adds it under *Services*. Heavily customised themes may not use the standard
navbar — link to `index.php?m=licenseforge` directly from your template.

**"Security token mismatch."**
The session expired while the page was open. Reload and retry. If it happens constantly,
check that PHP sessions are working and that any caching layer is not serving the client area
from cache.

---

## Downloads

**"You are not currently entitled to that download."**
The reason is precise and audited — filter the audit log on `download.denied` and read
`metadata.reason`. It is one of: license status not in the required list, a missing feature
entitlement, a client-group restriction, or a version beyond what the license covers.

**"The requested file is no longer available."**
The file is missing or unreadable, or it resolves outside `download_storage_path`. Check the
error log for `download file missing`, and confirm the web user can read the file.

**Download link expires immediately.**
Tokens are single use — a browser prefetching the link consumes it. Increase
`download_token_ttl` if your customers are on slow links; the single-use property is
deliberate and not configurable.

---

## Emails

**No licensing emails arrive.**
1. `notify_enabled` must be on.
2. The template names on the Settings tab must match **General** templates that actually
   exist in WHMCS.
3. The license must have a `client_id`.
4. Reminders are de-duplicated: a customer receives each threshold once per expiry date.
   Clearing them is automatic on renewal; to re-send during testing, delete the relevant
   rows from `lfg_notifications`.

Check the WHMCS Email Message Log to see whether WHMCS accepted the send.

---

## Performance

**Validation calls are slow.**
- Confirm migrations completed — the hot path depends on the unique indexes on
  `license_key` and `(license_id, installation_id)`.
- Check the size of `lfg_validations`. If retention is long and volume is high, lower
  `validation_log_retention` or turn off `log_validations` (failures are still recorded).
- Make sure cleanup is running: `php cron.php --task=cleanup`.

**The licenses list is slow.**
Searching by client name or email resolves through `tblclients` first. Searching by key,
domain, IP or service ID is index-backed and much faster — prefer those.

---

## Recovery

**I lost `storage/master.key`.**
Signing keys and API secrets cannot be decrypted. Recover by: generating a new signing key
(Settings → Generate signing key) and shipping its public key in your next SDK release; and
rotating every API credential and updating your builds. Licenses, activations and history are
unaffected — they are not encrypted.

**I deactivated the module by mistake.**
Nothing was deleted. Reactivate it; all data returns. Only the explicit
`licenseforge_destroy_data=DELETE-ALL-LICENSING-DATA` path removes data.

**I deleted a license by mistake.**
Deletion is soft (`deleted_at`). Restore it with:

```sql
UPDATE lfg_licenses SET deleted_at = NULL WHERE id = <id>;
```

---

## Getting more detail

Temporarily raise verbosity by watching the audit log live while reproducing the problem:

```sql
SELECT created_at, action, result, actor_type, ip_address, metadata
FROM lfg_audit_logs
WHERE license_id = <id>
ORDER BY id DESC
LIMIT 50;
```

and the licensing decisions themselves:

```sql
SELECT created_at, endpoint, success, error_code, domain, ip_address, version, duration_ms
FROM lfg_validations
WHERE license_id = <id>
ORDER BY id DESC
LIMIT 50;
```
