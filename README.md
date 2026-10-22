# php2shell-protection (PHP 5.2+)

php2shell-protection is a small, generic test kit to check whether an Apache `.htaccess` rule rejects `multipart/form-data` POST requests while allowing ordinary URL-encoded POSTs. No site-specific configuration is included.

## Security context: official PHP security advisory (21 October 2026)

**Critical: patch PHP. This project is an Apache multipart POST filtering and functional testing toolkit, NOT a complete mitigation for either CVE.** PHP officially disclosed two critical vulnerabilities that can allow unauthenticated remote code execution. All PHP installations serving web requests may be affected, including PHP-FPM, mod_php, php-cgi, FrankenPHP, and LiteSpeed. The PHP security team describes the highest risk for mod_php, including a demonstrated attack before application code runs.

| Vulnerability | Official description | Scope of this toolkit |
| --- | --- | --- |
| **CVE-2026-109618** (GHSA-w8hw-w9fx-j79w) | **RCE via out-of-memory (OOM) corruption.** An OOM event can interrupt initialization of internal request data; cleanup may subsequently process invalid data. | Blocking multipart POSTs does **not** address the general OOM vulnerability. |
| **CVE-2026-109619** (GHSA-6wmf-4rv5-p3rp) | **RCE via stale global state.** Request-owned state can remain after its memory is freed and be used by a later request on the same worker. | Session upload progress is one confirmed attack path, but PHP warns there are **other variants**. Blocking multipart POSTs is only a partial control. |

### Official fixed versions and legacy installations

Upgrade to a vendor-patched build, including these upstream fixed releases:

- **PHP 8.2.35** (8.2 reaches upstream end of life on 31 December 2026)
- **PHP 8.3.36**
- **PHP 8.4.27**
- **PHP 8.5.12**

PHP 8.1 and earlier are out of upstream support. Some Linux distributions and LTS vendors may offer backported fixes. The PHP security team also supplied minimal source backports for PHP **7.4, 8.0, and 8.1**. **PHP 5.2 has no upstream patch listed**; move to a maintained runtime or seek an appropriate vendor-supported migration plan. Do not assume that an older runtime is safe because a particular newer feature is absent.

### Separate PHP memory-safety reports and verified SimpleXML regression (21 October 2026)

**These reports are distinct from the php2shell CVEs.** They do not demonstrate that the security fixes for CVE-2026-109618 or CVE-2026-109619 are ineffective.

**Earlier research report (relationship unconfirmed):** A researcher publicly reported an AddressSanitizer-detected double-free in PHP 8.5.12 and clarified that their finding was **not** one of the php2shell vulnerabilities patched on 21 October. The publicly shared crash output alone does not establish remote reachability, code execution, or a CVE assignment. We cannot establish from the available information whether this report describes the same underlying defect as the GitHub issue below.

**Maintainer-verified PHP regression — [php/php-src issue #24488](https://github.com/php/php-src/issues/24488):** A separately documented bug report describes a reproducible double-free involving `SimpleXMLElement` property tables and PHP garbage collection in **PHP 8.4.27 and 8.5.12**. The PHP project marked the issue **Status: Verified**. The reporter reproduced `zend_mm_heap corrupted` and, with the system allocator, a double-free; they traced the regression to a garbage-collection change associated with GH-24050. They also reported failures in a Laravel/Pest test suite. This establishes a real stability and memory-safety concern in these versions, **not** remote exploitability or RCE.

A proposed fix is tracked in **[php/php-src pull request #24494](https://github.com/php/php-src/pull/24494)**. Its existence does not mean a fix has been merged or shipped; consult the linked issue, pull request, and PHP release notes for current status.

**Operational guidance:** Continue deploying the official php2shell security updates; do not revert to an unpatched release solely because of this separate regression. If your application uses SimpleXML and experiences crashes, track issue #24488 and the corresponding PHP maintenance releases. The Apache multipart filters and request-body limits in this repository **have not been validated as mitigations** for the SimpleXML issue or the earlier research report.

This repository does not include or link to private exploit material. The public PHP bug tracker and proposed fix are linked for transparent, verifiable context.

### Public research update: configuration-dependent RCE demonstration on PHP 8.5.12 (22 October 2026)

A publicly released proof of concept reports command execution on **PHP 8.5.12** using an **Apache 2.4 / `mod_php` laboratory configuration**. The reported mechanism is a **session-handling logic and application-object interaction**, not the memory-corruption mechanisms addressed by **CVE-2026-109618** and **CVE-2026-109619**. A demonstration showing command output in a controlled environment is noteworthy, but does **not** establish unauthenticated RCE against a default PHP installation.

**Important prerequisites disclosed by the research:** The newer demonstration depends on **different session serialization formats being used when session data is written and read**, a suitable **application-defined object with security-sensitive cleanup behavior**, and **session upload-progress processing**. The published laboratory setup uses non-default upload-progress cleanup behavior. The research also discusses a **separate, older session-poisoning chain requiring an application-level local file inclusion vulnerability**. These should not be conflated with each other or with the two October 21 CVEs.

**What remains unconfirmed:** Independent validation by PHP maintainers, general exploitability on ordinary deployments, applicability to CGI/FastCGI or PHP-FPM, and whether the newer demonstration merits a separate PHP-core CVE. The published report is not proof that the October 21 security fixes were bypassed. It also should not be conflated with the separately tracked SimpleXML garbage-collection regression.

**Defensive implications:** Where uploads are unnecessary, the Apache multipart-blocking rule in this project may interrupt the upload-progress stage. Disabling `session.upload_progress.enabled` at the PHP configuration level is a more targeted defense against chains that require that feature. Review applications for inconsistent session serialization settings, unsafe deserialization behavior, risky destructor side effects, and local file inclusion. **These project mitigations have not been independently validated against the newly published demonstration**, and request-body size limits alone are not a proven defense against it.

**Responsible-publication choice:** Although proof-of-concept material is now public, this README intentionally omits the researcher's identity, exploit repository links, payloads, commands, and attack paths. The aim is to provide actionable defensive context without facilitating reproduction.

### Additional post-patch research: reported session and memory-safety findings (22 October 2026)

Independent public research describes additional behavior on PHP releases containing the 21 October security fixes. **These are research claims and laboratory demonstrations, not new PHP security advisories.** Do not equate successful execution in a controlled configuration with unauthenticated RCE against a default deployment, or assume these findings reopen CVE-2026-109618 / CVE-2026-109619.

- **Session-handling logic and session disruption:** The research reports command execution where applications or virtual hosts read and write shared session data using incompatible serialization formats, combined with application-defined object behavior. It also reports an upload-progress session-key handling issue and potential session-file disruption, including tests described as using default settings. The broader scope and default-installation impact remain **author-reported**, not independently established here. Review shared session storage, serializer consistency, application deserialization and session lifecycle assumptions. Disable session upload progress if not needed.
- **SimpleXML garbage-collection double-free:** The underlying regression is separately tracked and maintainer-verified in [php/php-src #24488](https://github.com/php/php-src/issues/24488), with proposed fix [#24494](https://github.com/php/php-src/pull/24494). Additional research reports a reproducible remotely triggered process crash in a specially constructed PHP web endpoint. That supports a **demonstrated denial-of-service scenario in that test application**, not general remote exploitability or RCE on ordinary installations. The multipart filter and request-body limits here are **not validated controls** for this issue.
- **`unserialize()` re-entry use-after-free:** A separate report describes an AddressSanitizer-detected memory-safety violation on PHP 8.5.12. Its effect on ordinary non-instrumented builds, remote reachability, and exploitability remain unconfirmed. Audit use of `unserialize()` on untrusted data and avoid unsafe object lifecycle callbacks; do not interpret the ASAN report as confirmed RCE.
- **Patch-scope analysis:** The research also analyzes protections added in the October 21 fixes. This analysis is not itself a separate vulnerability report.

**Defensive priorities:** Install official PHP fixes; disable `session.upload_progress.enabled` at the effective server level when unnecessary; keep session serialization consistent across applications and workers sharing storage; avoid deserializing untrusted data; fix application-level file inclusion and unsafe object behavior; and use multipart blocking and body-size limits only where compatible with legitimate traffic. These measures cover **selected prerequisites**, not every newly reported issue. This project has **not independently reproduced or validated** all the claims or tested its mitigations against all of them.

**Publication policy:** This README intentionally does not reproduce payloads, link to exploit packages, or provide attack instructions. Research contributions are acknowledged in the Credits section without promoting exploit distribution.

### Temporary measures recommended by the PHP security team

These are **risk reductions, not complete fixes**. Confirm settings in the active web SAPI and restart affected workers as appropriate.

**For CVE-2026-109618 (OOM corruption):**

- Consider `memory_limit = -1` **temporarily** in an effective PHP configuration that applications cannot override. This removes the predictable PHP-enforced memory-limit failure involved in the reported OOM corruption path, **but it does not prevent real allocation failures or constitute a complete fix**. Removing PHP's limit can increase the risk of memory-exhaustion denial of service; use suitable worker isolation, container/cgroup memory controls, and monitoring. Do not use `RLIMIT_AS` or `RLIMIT_DATA` as substitutes, because allocation failures caused by those limits can still reach the vulnerable path.

**Optional temporary PHP configuration (CVE-2026-109618 only):**

```ini
; TEMPORARY mitigation while arranging a patched PHP upgrade
; Apply in the effective server-level PHP configuration.
memory_limit = -1
```

**Operational caution:** This is **not a recommended default for all sites** and is **not a mitigation for CVE-2026-109619**. It can allow a single request to consume excessive memory, potentially destabilizing a shared host. Verify the effective setting for the web-facing SAPI and restart workers as needed. Prefer upgrading PHP; do not add this directive to the separate session-hardening example by default. See the [official CVE-2026-109618 advisory](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) and [Daubois's analysis](https://daubois.dev/blog/cve-2026-109618-php2shell-rumors/) for context.
- Reduce `post_max_size`: PHP suggests **8M** where uploads are unnecessary or **32M** where uploads are required, and keep `upload_max_filesize` below `post_max_size`. Evaluate application needs before changing limits.
- Enforce request size and complexity limits at the web server or proxy, including decoded size, field count, multipart part count, header count and nesting depth, even when `Content-Length` is absent or unusable.
- Prefer container/cgroup boundaries that terminate only affected workers; PHP explicitly advises against substituting `RLIMIT_AS` or `RLIMIT_DATA` because those limits can trigger vulnerable allocation failures.

**For CVE-2026-109619 (stale global state):**

- Set `session.upload_progress.enabled = 0` in effective server-level PHP configuration and **restart every worker**. PHP confirms this blocks session-related attack paths, **not all variants**. Do not rely on `ini_set()` or FPM `.user.ini`: the configuration must apply before request parsing.
- On PHP 8.5, set `fatal_error_backtraces = 0` and restart workers to block the confirmed fatal-backtrace path (again, not all variants).
- Remove invalid/deprecated per-request settings that generate early diagnostics, and address attacker-reachable fatal errors and timeouts. Merely hiding diagnostics does not mitigate the issue.

### Request-body limits: Apache `.htaccess` and PHP configuration

The PHP security team also recommends **reducing POST/upload sizes** to lower exposure to **CVE-2026-109618** (the out-of-memory corruption vulnerability). Suggested `post_max_size` values are **8M when uploads are unnecessary** and **32M when uploads are required**. Keep `upload_max_filesize` smaller than `post_max_size`. These limits are risk reductions, **not fixes**, and they do not address every exploit path. `post_max_size` is a PHP setting, while Apache's `LimitRequestBody` limits the entire HTTP request body in bytes, for any content type or method. They are related controls, **not interchangeable settings**.

**Example A: sites that do not require multipart forms or uploads (default `http/.htaccess` and `https/.htaccess`):**

```apache
# 8 MiB maximum request body
LimitRequestBody 8388608

RewriteEngine On
RewriteCond %{REQUEST_METHOD} ^POST$ [NC]
RewriteCond %{HTTP:Content-Type} ^[[:space:]]*multipart/form-data([;,[:space:]]|$) [NC]
RewriteRule ^ - [F,L]
```

This preserves ordinary POSTs up to the configured limit, while blocking matching multipart POSTs regardless of size. The accompanying PHP configuration recommendation is:

```ini
post_max_size = 8M
upload_max_filesize = 2M
```

**Example B: sites that require multipart uploads (alternative example only):**

```apache
# 32 MiB maximum request body; multipart uploads are permitted
LimitRequestBody 33554432
```

With this alternative, **do not include the blanket multipart rejection rule** on upload endpoints. Apply additional controls (authentication, strict upload validation, WAF/request complexity limits) and prioritize upgrading PHP. Suggested PHP settings:

```ini
post_max_size = 32M
upload_max_filesize = 30M
```

The PHP settings above are **illustrative configurations** consistent with the suggested caps, not mandatory upload-file sizes. `post_max_size` includes multipart overhead and form fields, so leave sufficient headroom below the overall request limit for legitimate uploads. On **CGI/FastCGI/PHP-FPM**, `.htaccess` generally cannot set `post_max_size`, `upload_max_filesize`, or `memory_limit` using `php_value`; set them in the effective PHP configuration or hosting control panel instead. `php_value` in `.htaccess` applies only to appropriate Apache PHP module setups, and can cause HTTP 500 on CGI/FastCGI. Never paste `php_value` lines into the provided examples without verifying your PHP handler.

### Optional extreme hardening: 64 KiB or 4 KiB request bodies

For **mostly static sites without file uploads**, administrators who can tolerate stricter request-size restrictions may choose a substantially lower Apache body limit than the 8 MiB example above. These are **optional, application-specific limits**, not thresholds endorsed by the PHP security team and **not proven to prevent exploitation** of either CVE.

**Aggressive starting point — 64 KiB:**

```apache
# Maximum HTTP request body: 64 KiB
LimitRequestBody 65536

RewriteEngine On
RewriteCond %{REQUEST_METHOD} ^POST$ [NC]
RewriteCond %{HTTP:Content-Type} ^[[:space:]]*multipart/form-data([;,[:space:]]|$) [NC]
RewriteRule ^ - [F,L]
```

**Very restrictive option — 4 KiB:** Replace the first directive with:

```apache
# Maximum HTTP request body: 4 KiB
LimitRequestBody 4096
```

A 4 KiB limit may suit simple login or contact forms but can break larger forms, JSON/API requests, administrative operations, and other legitimate traffic. A 64 KiB limit is a more practical starting point for testing on many small sites, **not a universal recommendation**. Apply limits to appropriate paths if the rest of the application needs larger bodies. Test representative workflows and monitor **413 Content Too Large** responses before and after deployment. Keep the multipart rejection rule only where multipart requests are genuinely unnecessary.

For defense in depth, also configure PHP's effective `post_max_size` to reflect the application's actual needs, but remember that `post_max_size` and `LimitRequestBody` have different scopes and semantics; PHP's `post_max_size` is specified in PHP size notation and cannot normally be set through `.htaccess` with CGI/FastCGI. Neither limit caps multipart part counts or request-processing complexity by itself. **A small request can still trigger a vulnerability; patching is essential.**

`LimitRequestBody` is a core Apache directive permitted in `.htaccess` **only when the server allows the relevant overrides**. If adding it causes HTTP 500, inspect the Apache error log and remove the directive until the hosting provider confirms it is allowed. Apache may return **413 Content Too Large** when the body exceeds the configured limit. Check for upstream proxies or routes that bypass Apache and enforce equivalent limits there. Other complexity limits (header count, multipart part count, field count, nesting depth) may require virtual-host, reverse-proxy, or WAF configuration and cannot all be implemented with `.htaccess` alone.

### Why this project blocks multipart POST requests

**Origins of this project:** This toolkit was developed before the official PHP security advisories were published, based on preliminary information circulating in the security community and early access to zero-day proof-of-concept material that our team obtained and reviewed. That material helped inform our initial mitigation approach. No private exploit code, payloads, or reproduction instructions are included or linked.

**The official advisory now confirms session upload progress as one CVE-2026-109619 attack path, but it does not confirm that multipart blocking prevents all exploitation of that CVE or CVE-2026-109618.** The toolkit is a temporary defense-in-depth measure, not a substitute for patched PHP releases.

PHP can parse multipart form data **before the application script executes**; an application need not explicitly implement a file-upload page for the parser to be reachable. This toolkit checks whether Apache rejects a standard multipart POST at the web-server layer. The included rule deliberately blocks **all** matching multipart POSTs, including forms with no attached file. Do not enable it globally where legitimate multipart forms or uploads are needed.

The test outcome **403 / PHP not executed** verifies only the tested request path. It is **not** a vulnerability scanner, does not exercise either exploit, and cannot certify that a server is safe from either CVE.

### PHP configuration and PHP 5.2 compatibility

Where supported (PHP 5.4+), the upload-progress setting can be disabled in the effective `php.ini` or an appropriate server/FPM configuration **before request parsing**, followed by a restart of every relevant worker:

```ini
session.upload_progress.enabled = 0
```

Check the web SAPI's effective value rather than only the CLI value. **PHP 5.2.17 does not implement this directive**, but it still processes multipart bodies. Blocking multipart requests at Apache is independent of the PHP version if the rule is evaluated before forwarding the request; it remains a partial, route-specific compensating control.

### Official references

- PHP security announcement (21 October 2026): https://www.php.net/archive/2026.php
- CVE-2026-109618 / OOM corruption: https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w
- CVE-2026-109619 / stale global state: https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp
- PHP supported versions: https://www.php.net/supported-versions.php
- PHP upload progress documentation: https://www.php.net/manual/en/session.upload-progress.php

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

For applications that **do not require multipart requests**, prefer a tested **edge-level multipart POST rejection rule** such as the Apache example in this package (or an equivalent Nginx/WAF rule). Where multipart uploads must remain available, disabling `session.upload_progress.enabled` in the effective PHP configuration is more targeted if the feature is unnecessary. Neither approach substitutes for a fixed PHP release. These recommendations address only selected multipart/session-progress paths, not the full scope of either officially disclosed CVE.

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

### Optional PHP session hardening example

The package includes [`php-session-hardening.ini.example`](php-session-hardening.ini.example) as a **separate, opt-in configuration example**. It is **not installed automatically**, and the HTTP/HTTPS `.htaccess` files retain their existing multipart-blocking and request-size rules. The example is intended for administrators who can change the effective PHP configuration; **Apache `.htaccess` does not generally configure PHP directives under CGI/FastCGI or PHP-FPM**.

The example covers:

- **`session.upload_progress.enabled = Off`** — the most directly relevant setting for the session upload-progress prerequisite in the published demonstrations. Configure it before PHP parses requests, and verify it in the web SAPI after restarting/reloading workers. This directive is unavailable in PHP 5.2.
- **`session.serialize_handler = php`** — an *example* of a single consistent format, **not** a blanket security fix. All applications/workers sharing session storage must agree on the format. Changing it without planning may invalidate or misinterpret existing sessions; coordinate migrations or session resets. Do not assume a serializer switch removes unsafe application-defined object behavior.
- **`session.use_strict_mode = 1`** — general protection against adoption of uninitialized session IDs, provided the active session save handler supports ID validation. Not a demonstrated fix for the published RCE chain.
- **`session.cookie_httponly = 1`** — general cookie hardening; does not prevent server-side session poisoning or file inclusion.
- **`session.cookie_secure = 1`** — supplied **commented out**; enable only when session cookies are used exclusively over HTTPS and deployment/proxy settings are correct.

**Deployment:** Copy only appropriate settings into the active server-level `php.ini` or equivalent PHP-FPM pool configuration, reload/restart affected PHP workers, and check the effective values through the **web SAPI**. Verify login/logout, session persistence, all applications sharing session files, and existing session compatibility. PHP 5.2 deployments cannot use the upload-progress or strict-mode directives as substitutes for patching or migration. These settings do not remediate application LFI, dangerous deserialization callbacks, or the full set of October 2026 vulnerabilities.

### Interpretation and response priorities

A test result of ordinary POST **200 / PHP executed** and multipart POST **403 / PHP not executed** demonstrates the intended request-filtering behavior for the tested route. It does **not** prove every route is covered, that the rejection came from this exact rewrite rule, or that either CVE is mitigated. Other HTTP methods, alternate vhosts, upstream routing, and other vulnerabilities remain outside the test's scope.

**Recommended response:** upgrade immediately to a supported, fixed PHP release; until then, reduce public exposure, restrict unnecessary request types at the web-server or WAF layer, and test business-critical functionality. End-of-life PHP versions such as 5.2.17 do not receive upstream security fixes.

## Contents

- `http/`: `checkblock.php` targets `http://`.
- `https/`: `checkblock.php` targets `https://`.
- Each folder includes `target.php` and an example `.htaccess` rule with an 8 MiB Apache request-body limit.
- `testblock.php`: standalone, browser-based multipart POST test (no cURL required).
- `nginx/php2shell.conf`: equivalent rule for Nginx.

## Requirements

- PHP 5.2 or newer with the cURL extension enabled.
- Apache with `mod_rewrite` enabled and `.htaccess` rewrite overrides permitted.
- Server-side HTTP(S) requests to the selected hostname must work.

## Usage

1. Select **either** `http/` or `https/` depending on how your server can reach itself. Do not deploy both to the same directory.
2. Upload `target.php` and `checkblock.php` into a temporary web-accessible test directory.
3. Add the example rewrite rule and request-body limit to the appropriate existing `.htaccess` (or create one). If your site already has redirects, place the blocking rule **before** them. Preserve your existing rules. Note that nested `.htaccess` files and virtual-host configuration can change the effective behavior.
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

The packaged HTTP and HTTPS examples include an **8 MiB** request-body cap. See the two deployment profiles above if your application requires uploads.

```apache
LimitRequestBody 8388608

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
- **413 Content Too Large:** `LimitRequestBody` rejected an oversized HTTP request body. This is separate from the multipart-specific 403 test.
- **HTTP 500 after adding `LimitRequestBody`:** The server may disallow the directive in `.htaccess`; inspect the Apache error log and ask your hosting provider.
- **Additional ErrorDocument 403:** Apache may be trying to use an inaccessible custom error document; this does not change the original 403 result.

## Security limitations

A 403 indicates that the tested request was rejected, but does not by itself prove which component rejected it. This is a **functional configuration test**, not a vulnerability scanner or proof of complete protection from either PHP CVE. It does not test alternate methods, malformed headers, other virtual hosts, or routes that bypass the rule. For vulnerabilities in the PHP runtime, upgrading to a supported patched PHP release remains essential. PHP 5.2 is end-of-life.

## License

This project is distributed under the [MIT License](LICENSE).

## Credits

**Felipe Daragon, Syhunt** — project author.

**Andrew Dorman** — early security-research contributions and subsequent technical analysis of post-patch session-handling gaps, the SimpleXML garbage-collection regression, and a reported `unserialize()` memory-safety issue. His research helped inform the expanded defensive guidance; independently unverified claims remain identified as such.

**Nicolas Krassas** and **hash_s3c** — information contributions on X.

**Neil Smith** — suggested the additional Content-Type matching hardening for nonstandard comma-separated multipart header variants.
