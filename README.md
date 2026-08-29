# 🧨 Security Write-ups

![Writeups](https://img.shields.io/badge/writeups-5-blue)
![Focus](https://img.shields.io/badge/focus-web%20security-orange)
![License](https://img.shields.io/badge/license-MIT-green)

A collection of hands-on exploitation research, vulnerability analysis, and lab write-ups — written by **Sepehr Abolhasan**.

Everything here comes from **public training labs and CTF-style platforms**. No content in this repo discloses details of a real, unpatched, or unauthorized target. Write-ups exist to document methodology, build a public portfolio, and help others learn to spot the same bug classes.

## Index

| Write-up | Category | Platform | Description |
|---|---|---|---|
| [GYM XSS — p01 to p20](./gym-xss/) | Cross-Site Scripting | Brute Logic XSS GYM | Full context-breakout walkthrough — HTML, attribute, JS-string, and template-literal contexts (20/20 levels). |
| [CORS Origin Reflection — Admin API Key Exfiltration](./cors-api-key-exfil/) | CORS Misconfiguration | PortSwigger Academy | Exploiting a reflected-origin CORS policy to exfiltrate an authenticated admin's API key. |
| [WordPress Core Admin Creation — CSRF](./wordpress-csrf/) | CSRF | Local Docker Lab | Source-level nonce analysis showing how same-origin execution defeats WordPress's CSRF protections. |
| [OAuth Account Hijacking — `redirect_uri`](./oauth-account-hijacking-redirect-uri/) | OAuth Misconfiguration | PortSwigger Academy | Exploiting weak redirect URI validation to steal an admin authorization code, hijack the account, and delete `carlos`. |
| [OAuth Implicit Flow — Access Token Theft](./oauth-implicit-token-theft-open-redirect/) | OAuth / Open Redirect | PortSwigger Academy | Chaining weak callback validation, path traversal, and an open redirect to leak an OAuth access token. |

*More write-ups added as they're completed.*

## About

Security-focused write-ups covering exploitation techniques, payload construction, and remediation guidance for common web vulnerability classes (XSS, SQLi, access control, OAuth, and more as the collection grows). Each write-up aims to explain not just the *payload*, but the *reasoning*: why the context demands that breakout, how the attack chain works, and how to fix it.

## Contact

- Reach out via GitHub Issues or [https://t.me/Sepehr_FK]

---

<sub>Content in this repository is for educational purposes and targets only public training environments. See individual write-ups for platform/source attribution.</sub>
