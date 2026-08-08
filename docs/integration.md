# Integrating your software

## 1. Ship the SDK

Copy `sdk/LicenseClient.php` into your product. It has no dependencies beyond `ext-curl`
and `ext-json`, defines everything under `LicenseForge\SDK`, and is MIT licensed so you may
redistribute it.

```php
require __DIR__ . '/vendor/licenseforge/LicenseClient.php';
```

## 2. Configure the client

```php
use LicenseForge\SDK\LicenseClient;

$license = new LicenseClient([
    // Required
    'license_key'    => $settings->get('license_key'),
    'product_id'     => 'my-product',                    // the product slug from WHMCS
    'license_server' => 'https://billing.example.com/modules/addons/licenseforge/api/index.php',

    // API credentials — ship these with your application
    'api_key'        => 'lfk_…',
    'api_secret'     => 'lfs_…',

    // Public key for offline verification (GET /license/publickeys)
    'public_key'           => 'A1b2C3…',
    'public_key_algorithm' => 'ed25519',

    // Behaviour
    'version'      => '2.4.0',
    'cache_file'   => __DIR__ . '/storage/license.cache',
    'cache_ttl'    => 86400,    // re-check the server daily
    'grace_period' => 259200,   // tolerate 3 days past expiry when offline
    'timeout'      => 10,
    'retries'      => 2,
    'verify_tls'   => true,     // never set this to false
]);
```

> **On shipping the API secret.** A secret embedded in distributed software is not a secret
> from a determined attacker — it authenticates *your application*, not the customer, and
> its purpose is to keep the API off the open internet and let you revoke a leaked build.
> Give each release line its own credential with only the scopes it needs (`activate`,
> `validate`, `deactivate`, `info` — never `admin`), and rotate it when you rotate builds.
> The security of an individual license rests on the key, the binding rules and the
> signature — not on the API secret.

## 3. Activate

Call `activate()` once, when the customer enters their key:

```php
$result = $license->activate();

if ($result->isValid()) {
    $settings->set('license_key', $key);
    $settings->set('license_status', 'active');
    return redirect('/dashboard');
}

switch ($result->getErrorCode()) {
    case 'INVALID_LICENSE':
        $error = 'That license key was not recognised. Please check it and try again.';
        break;
    case 'ACTIVATION_LIMIT':
        $error = 'This license is already in use on the maximum number of installations. '
               . 'Deactivate an existing one, or reissue the license from your account.';
        break;
    case 'DOMAIN_MISMATCH':
        $error = 'This license is registered to a different domain. '
               . 'Request a reissue from your account area to move it here.';
        break;
    case 'LICENSE_EXPIRED':
        $error = 'This license has expired. Renew it to continue.';
        break;
    default:
        $error = $result->getErrorMessage();
}
```

## 4. Validate periodically

```php
$result = $license->validate();   // uses the cache until cache_ttl elapses

if (!$result->isValid()) {
    if ($result->needsActivation()) {
        return redirect('/license/activate');
    }
    return show_license_problem($result->getErrorCode(), $result->getErrorMessage());
}

if ($result->inGracePeriod()) {
    show_banner('Your license has expired and is running in a grace period. Please renew.');
}
```

Call it from your own scheduled task, or lazily on an admin page load — the SDK's cache
means only one call per `cache_ttl` actually reaches the server.

`validate(true)` forces a fresh call, e.g. from a "Check license now" button.

## 5. Feature entitlements

```php
if ($license->hasFeature('api_access')) {
    $router->registerApiRoutes();
}

if (!$license->hasFeature('white_label')) {
    $footer->showBranding();
}

$features = $license->getFeatures();  // ['api_access', 'premium_reports', …]
```

Entitlements come from the signed payload, so they are trustworthy offline. Check them where
the feature is used rather than caching a boolean at boot — that way a downgrade takes effect
at the next validation.

## 6. Expiry

```php
if ($license->isExpired()) {
    show_renewal_notice();
}

$expiry = $license->getExpirationDate();   // ?DateTimeImmutable, null for lifetime
if ($expiry !== null && $expiry->getTimestamp() - time() < 14 * 86400) {
    show_banner('Your license expires on ' . $expiry->format('j F Y') . '.');
}
```

## 7. Deactivate on uninstall

```php
register_uninstall_hook(function () use ($license) {
    $license->deactivate('uninstall');   // frees the activation slot
});
```

This is the single best thing you can do to reduce licensing support tickets.

## Offline validation

When the server cannot be reached, the SDK verifies the cached signed payload locally:

1. Verify the Ed25519/RSA signature against the embedded public key. No key configured, or
   a signature mismatch → **not valid**.
2. Confirm the payload's `license_key` matches the configured key.
3. Confirm `offline_until` has not passed.
4. Confirm `status` is `active` and the expiry (plus your `grace_period`) has not passed.
5. Re-check the domain binding locally, so a cached payload copied to another site fails.

```php
$result = $license->validate();

if ($result->getSource() === 'offline') {
    log_notice('Running on a cached license; the licensing server was unreachable.');
}
```

`getSource()` returns `remote`, `cache` or `offline`.

Verify a payload yourself if you need to:

```php
$payload = $license->verifyOfflineToken($token);  // array, or null when not genuine
```

### Key rotation

Payloads carry `_key_id`. When you rotate the signing key in the admin area, the old public
key stays listed on `/license/publickeys`, so previously issued payloads keep verifying.
Ship the new public key in your next release; existing installs continue to work against
their cached payload until it expires and they fetch a freshly signed one.

## Environment detection

The SDK derives these automatically; override any of them in the constructor:

| Value | Derived from | Override |
| --- | --- | --- |
| Domain | `SERVER_NAME`, `HTTP_HOST`, then `gethostname()` | `domain` |
| Directory | `ABSPATH` if defined, else the SDK's parent directory | `directory` |
| Machine ID | SHA-256 of hostname + OS + install path | `machine_id` |
| Installation ID | SHA-256 of machine ID + domain | `installation_id` |

The machine identifier is deliberately non-invasive: it identifies an *installation*, not a
person or a device serial, and collects nothing beyond what activation already needs.

For a CLI or desktop application, set `domain` explicitly (or leave the product not
domain-locked) since there is no HTTP host to detect.

## Full example

```php
final class LicenseGate
{
    private LicenseClient $client;

    public function __construct(array $config)
    {
        $this->client = new LicenseClient($config + [
            'cache_file' => __DIR__ . '/../storage/license.cache',
            'version'    => APP_VERSION,
        ]);
    }

    public function boot(): void
    {
        $result = $this->client->validate();

        if ($result->isValid()) {
            if ($result->inGracePeriod()) {
                Notices::warn('Your license has expired. Renew to avoid interruption.');
            }
            Features::load($result->getFeatures());

            return;
        }

        // A licensing failure disables paid functionality — it never silently
        // continues, and it never destroys the customer's data.
        Features::load([]);
        Notices::error($this->explain($result));

        if (in_array($result->getErrorCode(), ['INVALID_LICENSE', 'ACTIVATION_NOT_FOUND'], true)) {
            Router::forceRedirect('/license/activate');
        }
    }

    private function explain(LicenseResult $result): string
    {
        return match ($result->getErrorCode()) {
            'LICENSE_EXPIRED'      => 'Your license has expired. Renew it in your account.',
            'LICENSE_SUSPENDED'    => 'Your license is suspended. Please check your billing account.',
            'DOMAIN_MISMATCH'      => 'This license belongs to another domain. Reissue it from your account.',
            'ACTIVATION_LIMIT'     => 'All activation slots are in use. Deactivate an old installation first.',
            'VERSION_NOT_SUPPORTED' => 'Your license does not cover this version. Upgrade your plan or downgrade the software.',
            'SERVICE_UNAVAILABLE'  => 'The licensing server is unreachable and the offline period has ended.',
            default                => $result->getErrorMessage() ?? 'Your license could not be verified.',
        };
    }
}
```

## What the SDK will not do

It never silently bypasses a licensing decision. There is no "fail open" mode, no way to
disable TLS verification through configuration alone (`verify_tls` exists for pinned-CA
setups via `ca_bundle`, not for skipping verification in production), and offline operation
is strictly bounded by the signed `offline_until` timestamp. If you need different behaviour
for a specific customer, change it server-side — that is what the admin area is for.
