# websec-writeups

Web security lab writeups — OWASP Top 10 hands-on, from the
[PortSwigger Web Security Academy](https://portswigger.net/web-security).

Each writeup follows the same format: **attack → impact → fix**, with the root
cause explained and tied to real-world application/API security.

## Writeups

| Topic | OWASP | Labs |
|---|---|---|
| [Authentication attacks](authentication-attacks.md) | A07 | Username enumeration, brute-force bypass, 2FA bypass, password-reset abuse |
| [Access Control & IDOR](access-control-idor.md) | A01 | Unprotected admin, role tampering, IDOR, GUID leakage, password disclosure |
| [Cross-Site Scripting (XSS)](cross-site-scripting.md) | A03 | Reflected (HTML/JS context), stored, DOM-based |
| [Server-Side Request Forgery (SSRF)](ssrf.md) | A10 | Localhost, internal network scan, blacklist bypass, open-redirect chaining, blind/OAST |
| [Path Traversal & OS Command Injection](path-traversal-command-injection.md) | A01 / A03 | Arbitrary file read, RCE via shell injection |

## Tools

`Burp Suite` · browser DevTools · `Python`

## Note

All testing was performed exclusively in PortSwigger's authorized lab
environment.
