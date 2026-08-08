# WHMCS version compatibility

One copy of LicenseForge runs on **WHMCS 8.x and 9.x**. There is no separate build.

| | WHMCS 8.x | WHMCS 9.x |
| --- | --- | --- |
| PHP | 8.0+ | 8.2+ (8.3 recommended) |
| Smarty | 3 | 4 |
| MySQL | 5.7+ | 8.0 recommended |
| ionCube (for WHMCS) | per WHMCS | 13.0.2+ |

LicenseForge is plain PHP source, so ionCube encoding rules do not apply to it.

---

## What WHMCS 9.0 changed, and how the module handles it

### PHP 8.2 minimum

WHMCS 9.0 dropped PHP 7.2–8.1. The module was audited for every removal and deprecation
between 8.0 and 8.4:

| Check | Result |
| --- | --- |
| `${var}` string interpolation (deprecated 8.2) | Not used |
| Dynamic properties on classes (deprecated 8.2) | Not used — every class declares its properties; only `stdClass` rows from the query builder are accessed dynamically, which remains valid |
| `utf8_encode` / `utf8_decode` (deprecated 8.2) | Not used |
| `strftime`, `money_format`, `each`, `create_function` | Not used |
| Implicit nullable parameters (`Type $x = null`, deprecated 8.4) | Audited; the one occurrence was changed to an explicit `?Type` |
| `curl_close()` (deprecated 8.5) | Removed from the SDK — the handle is released when it goes out of scope |
| Partially supported callable strings (deprecated 8.2) | Not used |

The codebase uses `declare(strict_types=1)`, typed properties and typed signatures
throughout, so it is forward-compatible rather than merely not-yet-broken.

### Smarty 4

WHMCS 9.0 moved to Smarty 4 and removed `{php}`, `{include_php}`, `{fetch}`,
`$smarty.template_object`, and the settings that previously re-enabled Smarty PHP tags.

| Check | Result |
| --- | --- |
| `{php}` / `{include_php}` / embedded PHP | Never used |
| `{fetch}` | Never used |
| `$smarty.template_object` | Never used |
| ASP-style `<% %>` tags | Never used |
| `$smarty.foreach.NAME.last` | **Changed** — two occurrences replaced with the `{$item@last}` form, which is valid in Smarty 3, 4 and 5 |
| `{if $var\|in_array:$list}` | Not used — entitlement checkboxes are pre-computed in the controller |

Two further template constructs were replaced not because they are known to break, but
because they depend on parser behaviour I could not verify against a running Smarty 4:

- **Chained array-then-object access** (`{$item.row->license_key}`) — the licenses list and
  dashboard now receive flat arrays from `Controller::decorate()`.
- **Arrays indexed by an object property** (`{$statusLabels[$license->status]}`,
  `{$counts[$product->id]}`) — status labels and per-product counts are now resolved in PHP
  and passed as scalars.

This also removed presentation logic from the templates, so it is a net improvement
regardless of Smarty version.

### Template rendering

`Support\View::smarty()` uses `\WHMCS\Smarty` when present — it exists in both 8.x and 9.x,
wrapping Smarty 3 and Smarty 4 respectively — and falls back to a bare `\Smarty` instance
with its own compile directory otherwise. Only `setTemplateDir()`, `assign()` and `fetch()`
are used; all three are unchanged in Smarty 3, 4 and 5. Render failures are caught, logged
server-side and surfaced as an inline notice rather than a white screen.

### Module and hook APIs

The addon module contract (`_config`, `_activate`, `_deactivate`, `_upgrade`, `_output`,
`_clientarea`, `_sidebar`) and `add_hook()` are unchanged in 9.0. The hooks used —
`AfterModuleCreate`, `AcceptOrder`, `AfterModuleSuspend`, `AfterModuleUnsuspend`,
`AfterModuleTerminate`, `AfterModuleChangePackage`, `ServiceEdit`, `ServiceDelete`,
`CancellationRequest`, `InvoicePaid`, `DailyCronJob`, `ClientAreaPrimaryNavbar`,
`ClientAreaPageProductDetails` — are all current, non-deprecated hook points.

Every WHMCS integration point is defensive: `localAPI()` is called only after
`function_exists()`, `\WHMCS\Config\Setting::getValue()` and all `tblclients` / `tblhosting`
/ `tblproducts` / `tblinvoiceitems` queries are wrapped in try/catch with sensible
fallbacks, and the navbar hook swallows its own errors so a theme change can never break
the client area.

### Nexus cart and the Buy Flow API

WHMCS 9.0 introduces a new cart and order flow. LicenseForge does not integrate with the
cart, order form or checkout at all — it reacts to *provisioning and billing* events
(`AfterModuleCreate`, `AcceptOrder`, `InvoicePaid`), which fire identically regardless of
which cart produced the order. This is why the "custom cart integrations may break"
warning in the 9.0 notes does not apply here.

### Database layer

All queries go through `Capsule` (`Support\Db`) using the query builder — `table()`,
`where()`, `insertGetId()`, `update()`, `increment()`, `pluck()`, `forPage()`, `when()`,
`whereBetween()`, `distinct()`, `transaction()`. These are stable Illuminate APIs present in
every version WHMCS has bundled. Schema changes use the Blueprint builder through the same
Capsule connection.

One raw expression is used, in `AbuseDetector::sweep()` and
`Controller::licenseCountsByProduct()`: `Db::raw('license_id, COUNT(*) as live')`. It
contains no user input.

---

## Verification performed

```bash
# All files parse under the target PHP version
for f in $(find . -name '*.php'); do php -l "$f"; done

# Unit suite (no WHMCS, no database)
php tests/run.php     # 39 passed, 0 failed
```

Both were run on PHP 8.2 semantics. The audits above were carried out by grepping the
codebase for each removed or deprecated construct; the commands are reproducible:

```bash
grep -rn '\${' --include='*.php' lib api sdk                    # 8.2 interpolation
grep -rnE '\b(each|create_function|utf8_encode|strftime)\(' .   # removed functions
grep -rnE '\{php\}|\{include_php|\{fetch |smarty\.template_object' templates/
grep -rn 'smarty.foreach' templates/
grep -rnE '\[\$[a-zA-Z_]+->' templates/                         # array indexed by object property
grep -rnE '\$[a-zA-Z_]+\.[a-zA-Z_]+->' templates/               # chained array→object access
```

All return no results on the current tree.

## What has *not* been verified

Be clear about the boundary: the module has not been executed inside a running WHMCS 9.x
installation, because none was available here. Static analysis, the unit suite and the
documented API contracts are what back the claims above.

Before going live on 9.x, run these on a staging install — they exercise every integration
point the audit could not:

1. Activate the module; confirm all five migrations apply (**Settings → Applied migrations**).
2. Open each admin tab — dashboard, licenses, a license detail page, products, product edit,
   downloads, reissues, abuse, API, audit log, settings. Any Smarty 4 parse error surfaces
   immediately as an inline notice, with detail in the PHP error log.
3. Order a licensed product end to end through the **new Nexus cart**; confirm the license is
   created and moves to `Active` on payment.
4. Suspend, unsuspend and terminate the service; confirm the license status follows.
5. Run the client area: list, detail, reissue, deactivate an installation, download a file.
6. Call the API with the SDK: activate, validate, deactivate.
7. Run `php cron.php --task=all`.

If step 2 shows a template problem, the error log entry names the template and the Smarty
message — the fix is local to that `.tpl` file and does not touch application logic.
