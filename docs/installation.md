# Installation, upgrade and removal

## 1. Requirements check

| | WHMCS 8.x | WHMCS 9.x |
| --- | --- | --- |
| PHP | 8.0+ | **8.2+** (8.3 recommended) |
| MySQL / MariaDB | 5.7+ / 10.3+ | 8.0 recommended |
| ionCube (WHMCS itself) | per WHMCS | 13.0.2+ |

LicenseForge ships as plain PHP source — it is not ionCube-encoded, so the loader version
matters only to WHMCS itself.

```bash
php -r 'printf("PHP %s\nopenssl:%d json:%d curl:%d sodium:%d\n",
  PHP_VERSION,
  extension_loaded("openssl"), extension_loaded("json"),
  extension_loaded("curl"), extension_loaded("sodium"));'
```

`sodium` is optional but recommended — with it, offline payloads are signed with Ed25519
(64-byte signatures, fast verification). Without it the module falls back to RSA-2048/SHA-256.

Your WHMCS installation must be reachable over HTTPS. The licensing API refuses plain HTTP
from anything other than loopback.

## 2. Install the files

```bash
cp -r licenseforge /path/to/whmcs/modules/addons/
cd /path/to/whmcs/modules/addons/licenseforge
chown -R www-data:www-data .
find . -type d -exec chmod 750 {} \;
find . -type f -exec chmod 640 {} \;
```

The module creates `storage/` on first use to hold the master key that wraps signing keys
and API secrets. It must be writable by the web user and unreachable from the web — the
module writes a deny-all `.htaccess`, but on nginx add:

```nginx
location ~ ^/modules/addons/licenseforge/(lib|templates|tests|sdk|storage)/ {
    deny all;
    return 404;
}
```

## 3. Activate

**WHMCS Admin → System Settings → Addon Modules → LicenseForge → Activate.**

Activation is transactional and idempotent. It:

1. runs all migrations (creates the `lfg_*` tables),
2. seeds default settings and the built-in feature catalogue,
3. generates the first offline signing key pair,
4. creates a default API credential.

> The API key **and secret** are shown once, on the activation confirmation. Copy the
> secret immediately — it can be re-displayed only by rotating it, which invalidates the
> old value. Then grant access: **Configure → Access Control**, select the admin roles that
> may manage licensing.

## 4. Configure

1. **Addons → LicenseForge → Settings** — key format, defaults, locking, rate limits,
   retention and email templates. See [configuration.md](configuration.md).
2. **Products** — pick a WHMCS product, enable licensing, set duration and limits.
3. **Downloads** *(optional)* — set a storage root outside the web root, then add files.
4. **API** — create one credential per product or per integrator.

## 5. Cron

The `DailyCronJob` hook already runs maintenance once a day. If you use short grace periods
or want prompt expiry enforcement, add a more frequent job:

```cron
*/15 * * * * php /path/to/whmcs/modules/addons/licenseforge/cron.php --quiet
```

Individual tasks can be scheduled separately:

```bash
php cron.php --task=expire      # move due licenses to expired
php cron.php --task=grace       # open and close grace windows
php cron.php --task=reminders   # expiry reminder emails
php cron.php --task=abuse       # concurrent-installation sweep
php cron.php --task=cleanup     # log retention, token/nonce pruning
php cron.php --task=stale       # release installations that stopped checking in
php cron.php --task=all         # everything (default)
```

Every task is idempotent — running it twice produces the same state as running it once, so
overlapping cron runs are harmless.

## 6. Email templates

Create these as **General** templates under Setup → Email Templates, using the names on the
Settings tab (rename them there if you prefer your own naming):

| Template | Sent when |
| --- | --- |
| LicenseForge License Created | A license is issued |
| LicenseForge License Activated | First activation of an installation |
| LicenseForge License Expiring | 30/14/7/1 days before expiry (configurable) |
| LicenseForge License Expired | Expiry, after any grace window |
| LicenseForge License Suspended | License moves to suspended |
| LicenseForge License Reissued | A reissue completes |
| LicenseForge Activation Limit Reached | An activation is refused for lack of slots |
| LicenseForge Suspicious Activity | An abuse signal fires |

Merge fields: `{$license_key}`, `{$license_status}`, `{$license_product}`,
`{$license_expires}`, `{$license_domain}`, `{$license_activations}`, `{$license_reissues}`,
plus per-event fields `{$days_remaining}`, `{$previous_key}`, `{$activation_domain}`,
`{$activation_ip}`, `{$status_reason}`, `{$abuse_signal}`, `{$abuse_summary}`.

Emails are de-duplicated through `lfg_notifications`, so a customer never receives the same
reminder twice — and the markers are cleared automatically when a renewal moves the
expiry date.

## Upgrading

1. Back up the database (`mysqldump --tables … lfg_*` at minimum) and the module directory,
   **including `storage/master.key`** — without it, signing keys and API secrets cannot be
   decrypted.
2. Replace the module directory, keeping `storage/`.
3. Open the module once in the admin area, or visit Addon Modules — `licenseforge_upgrade()`
   runs the outstanding migrations.
4. Confirm on **Settings → Applied migrations** that the new entries are listed.

Migrations are additive and never edited after release, so downgrading the files is safe as
long as you do not need the newer columns.

## Deactivation and uninstall

**Deactivating keeps all data.** Licenses, activations and audit history remain; reactivate
at any time and everything is where you left it. This is deliberate — deactivating a module
by accident must never destroy customer licensing records.

To remove the data permanently, request the destructive path explicitly by appending the
confirmation parameter to the deactivation request:

```
addonmodules.php?action=deactivate&module=licenseforge&licenseforge_destroy_data=DELETE-ALL-LICENSING-DATA
```

This drops every `lfg_*` table and cannot be undone. Take a backup first. After that, delete
the module directory (including `storage/`).
