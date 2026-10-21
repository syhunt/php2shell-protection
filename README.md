# php2shell-protection (PHP 5.2+)

php2shell-protection is a small, generic test kit to check whether an Apache `.htaccess` rule rejects `multipart/form-data` POST requests while allowing ordinary URL-encoded POSTs. No site-specific configuration is included.

## Security context: October 2026 critical PHP vulnerabilities

On 20 October 2026, the PHP security team announced an emergency release scheduled for **21 October 2026 at 14:00 UTC** to address **two critical vulnerabilities, each rated CVSS v4.0 9.2**. Reports identify them as **CVE-2026-109618** and **CVE-2026-109619**. At the time this document was prepared (21 October 2026, before the scheduled release), PHP had **not officially published affected version ranges, root causes, exploitation prerequisites, or fixed releases**. Researchers reported possible remote code execution and broad historical version exposure, but these details were not confirmed by PHP's advisory.

- Official announcement: https://www.php.net/archive/2026.php#2026-10-20-1
- PHP supported versions: https://www.php.net/supported-versions.php
- PHP session configuration: https://www.php.net/manual/en/session.configuration.php
- PHP upload progress: https://www.php.net/manual/en/session.upload-progress.php

### Technical context and scope of the mitigation

Privately supplied research material describing the October 2026 issues indicates a possible **session upload-progress memory-safety attack path** involving crafted `multipart/form-data` POST requests. The material describes issues involving upload-progress bookkeeping while PHP handles a request, **before the application script executes**. This provides a concrete rationale for examining multipart POST filtering; it does **not** independently establish the official root cause, affected PHP releases, CVE-to-mechanism mapping, or exploitability on a given host. The research material and proof-of-concept code are intentionally **not linked, reproduced, or distributed** with this project.

PHP normally parses a `multipart/form-data` POST body during request initialization. An application does not need to read `$_FILES`, implement an upload form, or explicitly enable upload progress for the native multipart parser to be reached. On PHP versions where `session.upload_progress.enabled` is active, the upload-progress mechanism can be engaged by multipart form fields even if the application never uses the resulting session data.

**Defensive options depend on functionality:**

- **No multipart forms needed:** reject multipart POST requests at Apache or an upstream WAF **before they reach PHP**. The example `.htaccess` rule is deliberately strict: it rejects all matching multipart POSTs, including requests without files.
- **Multipart forms required, but no session upload-progress feature needed:** disable `session.upload_progress.enabled` through an effective PHP configuration source, where supported. This is more targeted than blocking all multipart POSTs, but should not be assumed to address any independent parser vulnerability.
- **Uploads and progress required:** prioritize a vendor-fixed PHP runtime and restrict unnecessary endpoints and public exposure. Body-size or rate limits may reduce operational risk but do not guarantee protection against a memory-safety bug triggered by a small request.

The example filter is a **compensating control for the specific multipart POST entry path**. It is not a universal mitigation for all HTTP methods, alternate PHP handlers, or every vulnerability attributed to these CVEs. A successful functional test is not proof of complete CVE remediation.

### Why blocking `PHPSESSID` at Nginx is not recommended

A suggested workaround is to reject requests whenever the Cookie header or query string contains `PHPSESSID`, for example:

```nginx
# NOT RECOMMENDED as a CVE mitigation
if ($http_cookie ~* PHPSESSID) { return 403; }
if ($args ~* PHPSESSID) { return 403; }
```

**Do not deploy this as a general-purpose defense.** It has several important limitations:

- **Breaks normal sessions:** `PHPSESSID` is PHP's default session cookie name. Legitimate visitors may send it on most requests, so this rule can block ordinary pages, logins, and authenticated sessions.
- **Easy to miss custom session names:** applications can change the session name with `session_name()` or PHP configuration; the literal `PHPSESSID` match would not cover those names.
- **Does not block the multipart parser:** a request without that cookie or query-string token can still carry a `multipart/form-data` body. A header/query match does not establish that upload-progress processing cannot be reached.
- **Overbroad matching:** the regular expression matches the text anywhere in the header or query string, including unrelated values, creating unnecessary false positives.

For applications that **do not require multipart requests**, prefer a tested **edge-level multipart POST rejection rule** such as the Apache example in this package (or an equivalent Nginx/WAF rule). Where multipart uploads must remain available, disabling `session.upload_progress.enabled` in the effective PHP configuration is more targeted if the feature is unnecessary. Neither approach substitutes for a fixed PHP release. These recommendations are scoped to the suspected multipart/session-progress attack path, not a guarantee against either CVE.

### PHP configuration: session upload progress

On PHP versions supporting session upload progress (introduced in **PHP 5.4**), administrators whose applications do not need that feature can consider disabling it in an applicable `php.ini` or per-directory configuration:

```ini
session.upload_progress.enabled = Off
```

Verify the effective value from the **web SAPI**, not just command-line PHP:

```php
<?php
$v = ini_get('session.upload_progress.enabled');
echo $v === false ? 'Directive unavailable' : var_export($v, true);
?>
```

An empty string or `0` from an available boolean directive indicates disabled; `false` from `ini_get()` means the directive may be unavailable, so distinguish these cases with strict comparison. Changes may require a PHP-FPM or web-server reload, depending on how PHP is deployed.

**PHP 5.2.17:** `session.upload_progress.enabled` **does not exist**, so adding it to `php.ini` will not provide a mitigation. PHP 5.2 still has native multipart request parsing. `file_uploads = Off` disables upload handling but **does not guarantee that all multipart form-body parsing is skipped**. The `.htaccess` rule in this package operates independently of PHP's version, provided Apache evaluates it before PHP execution.

Disabling session upload progress on newer PHP versions directly removes that optional feature, but **should not be presented as a verified fix for both October 2026 CVEs** without vendor confirmation. Do not infer that the feature's absence on PHP 5.2 makes that runtime safe.

### Interpretation and response priorities

A test result of ordinary POST **200 / PHP executed** and multipart POST **403 / PHP not executed** demonstrates the intended request-filtering behavior for the tested route. It does **not** prove every route is covered, that the rejection came from this exact rewrite rule, or that either CVE is mitigated. Other HTTP methods, alternate vhosts, upstream routing, and other vulnerabilities remain outside the test's scope.

**Recommended response:** upgrade to a supported, fixed PHP release when officially available; until then, reduce public exposure, restrict unnecessary request types at the web-server or WAF layer, and test business-critical functionality. End-of-life PHP versions such as 5.2.17 do not receive upstream security fixes.

## Contents

- `http/`: `checkblock.php` targets `http://`.
- `https/`: `checkblock.php` targets `https://`.
- Each folder includes `target.php` and an example `.htaccess` rule.
- `testblock.php`: standalone, browser-based multipart POST test (no cURL required).
- `nginx/php2shell.conf`: equivalent rule for Nginx.

## Requirements

- PHP 5.2 or newer with the cURL extension enabled.
- Apache with `mod_rewrite` enabled and `.htaccess` rewrite overrides permitted.
- Server-side HTTP(S) requests to the selected hostname must work.

## Usage

1. Select **either** `http/` or `https/` depending on how your server can reach itself. Do not deploy both to the same directory.
2. Upload `target.php` and `checkblock.php` into a temporary web-accessible test directory.
3. Add the example rewrite rule to the appropriate existing `.htaccess` (or create one). If your site already has redirects, place the blocking rule **before** them. Preserve your existing rules. Note that nested `.htaccess` files and virtual-host configuration can change the effective behavior.
4. Open `checkblock.php` in your browser. It makes two small POST requests to `target.php` using the selected scheme. `CURLOPT_FOLLOWLOCATION` is disabled so redirects remain visible.
5. Expected: ordinary POST => HTTP 200 with `PHP Executed: YES`; multipart POST => HTTP 403 with `PHP Executed: NO`.
6. Remove `checkblock.php` and `target.php` after testing. Keep the rewrite rule only if your site does not require multipart forms or uploads.

## Browser-based test (`testblock.php`)

`testblock.php` is an alternative to the cURL-based `checkblock.php` tests. It works with **PHP 5.2+**, requires **no cURL extension**, and sends a real `multipart/form-data` POST from your browser to itself. The form uses a hidden text field rather than a file upload, demonstrating that the block applies to multipart forms even when no file is attached.

1. Upload `testblock.php` into a temporary web-accessible directory **covered by the intended Apache `.htaccess` rule**. You can place it alongside the HTTP or HTTPS test files, but it does not depend on either folder or on a particular URL scheme.
2. Open `testblock.php` in your browser and click **Test Multipart Blocking**.
3. Inspect the response in the embedded frame:
   - **403 Forbidden**: the multipart request was rejected (consistent with the rule working). A custom error page may appear instead of the standard Apache 403 page.
   - **`MULTIPART_REACHED_PHP`**: the multipart POST reached the PHP script; the intended blocking is not effective on this route.
   - **Other status, redirect, or network error**: investigate server routing and access controls before interpreting the result.
4. **Remove `testblock.php` after testing.**

This browser test sends a standards-compliant multipart request; it does **not** test the non-standard comma-separated Content-Type variant. That variant requires a separate controlled HTTP client test.

This test only checks one browser-originated POST to the deployed script. A 403 alone cannot prove *which* server rule rejected it or establish protection against every exploit path. Do not test it merely by visiting `?probe=1` directly: that sends a GET request, not the multipart POST used by the form.

## Example .htaccess

```apache
RewriteEngine On
RewriteCond %{REQUEST_METHOD} ^POST$ [NC]
RewriteCond %{HTTP:Content-Type} ^[[:space:]]*multipart/form-data([;,[:space:]]|$) [NC]
RewriteRule ^ - [F,L]
```

The revised header check also rejects non-standard comma-separated variants such as `Content-Type: multipart/form-data, boundary=example`, as well as standard semicolon-separated parameters. Commas are **not valid multipart parameter separators** under MIME syntax; rejecting this variant is defensive hardening, not evidence that PHP accepts it as multipart. Leading spaces and tabs are tolerated in the match. This is a header-based filter, not a complete HTTP request normalization or parser-discrepancy defense. Test any reverse proxy, alternate PHP routes, and edge cases in your own environment.

The rule blocks **all** matching multipart POST requests, including forms with no attached file. It does not block ordinary URL-encoded POSTs. If the site needs uploads, do not deploy this rule globally without scoping exceptions and testing them.

## Example Nginx

Inside the `location` that passes requests to PHP:

```nginx
if ($http_content_type ~* "^\s*multipart/form-data([;,\s]|$)") {
    return 403;
}
```

Same match as the `.htaccess` rule (case, leading spaces, comma variant), for any method.

## Troubleshooting

- **Both tests return 301/302:** A redirect is occurring; choose the matching scheme or adjust server routing.
- **Both tests return 403:** Other access restrictions may be in effect.
- **Multipart returns 200:** The block is not being applied to the tested path, or the request is routed around it.
- **HTTP 0 / cURL error:** Check DNS, outbound access, local loopback, TLS certificate validity (for HTTPS), and cURL installation. Certificate verification is intentionally not disabled.
- **Additional ErrorDocument 403:** Apache may be trying to use an inaccessible custom error document; this does not change the original 403 result.

## Security limitations

A 403 indicates that the tested request was rejected, but does not by itself prove which component rejected it. This is a **functional configuration test**, not a vulnerability scanner or proof of complete protection from either PHP CVE. It does not test alternate methods, malformed headers, other virtual hosts, or routes that bypass the rule. For vulnerabilities in the PHP runtime, upgrading to a supported patched PHP release remains essential. PHP 5.2 is end-of-life.

## License

This project is distributed under the [MIT License](LICENSE).

## Credits

**Felipe Daragon, Syhunt** — project author.

**Andrew Dorman**, **Nicolas Krassas** and **hash_s3c** — info contributions on X.

**Neil Smith** — suggested the additional Content-Type matching hardening for nonstandard comma-separated multipart header variants.
