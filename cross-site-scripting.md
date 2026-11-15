# Cross-Site Scripting (XSS) — PortSwigger Web Security Academy

> Writeup covering 4 XSS labs across all three types (OWASP Top 10 — A03: Injection).
> Author: Yuri Vinhote · Tool: browser, Burp Suite

XSS lets an attacker run JavaScript in another user's browser, in the trusted context of the vulnerable site — enabling session/cookie theft and actions on the victim's behalf. The root cause is always the same: **user input reaches the page without being encoded for the context it lands in** (HTML, HTML attribute, JavaScript string, or a client-side sink).

---

## 1. Reflected XSS into HTML context (nothing encoded)

The search term was reflected straight into the HTML with no encoding.

**Attack**
- Searched `<script>alert(1)</script>` — it was inserted as a real tag and executed.

**Impact** — Any victim who opens a crafted link runs attacker JavaScript.

**Fix** — HTML-encode output (`<` to `&lt;`, `>` to `&gt;`) so input renders as text.

---

## 2. Reflected XSS into a JavaScript string (angle brackets HTML-encoded)

Angle brackets were encoded (no new tags possible), but the input landed inside a JavaScript string: `var searchTerms = '<input>';`.

**Attack**
- Payload: `'-alert(1)-'`
- Result: `var searchTerms = ''-alert(1)-'';` — breaks out of the string and runs `alert(1)` as part of an expression, keeping valid syntax.

**Impact** — Script execution even though HTML tags were blocked — different context, different break-out.

**Fix** — Encode for the JavaScript context (escape quotes/backslashes), or avoid reflecting user input into script at all.

---

## 3. Stored XSS into HTML context (nothing encoded)

A blog comment was stored and later rendered to every visitor without encoding.

**Attack**
- Posted a comment containing `<script>alert(1)</script>`.
- The payload is saved server-side and executes for anyone who views the post — no link, no social engineering needed.

**Impact** — The most dangerous XSS: one malicious comment hits every visitor of the page.

**Fix** — Encode on output when rendering stored content; validate/sanitize on input as defense in depth.

---

## 4. DOM-based XSS in `document.write` sink using `location.search`

No server involvement: the page's own JavaScript read the search term from the URL (`location.search`) and wrote it into the DOM via `document.write`, inside an `<img>` `src` attribute.

**Attack**
1. Searched `teste` and inspected the DOM: `<img src="/resources/images/tracker.gif?searchTerms=teste">`.
2. Payload: `"><svg onload=alert(1)>`
   - `">` closes the `<img>` tag
   - `<svg onload=alert(1)>` creates a new element whose `onload` fires `alert(1)`.

**Impact** — Client-side script execution; the server never sees the payload, so server-side filters don't help.

**Fix** — Avoid dangerous sinks (`document.write`, `innerHTML`) with untrusted data; use safe APIs (`textContent`) and context-aware encoding in client code.

---

## Key takeaway

XSS is one bug in four contexts — HTML body, HTML attribute, JavaScript string, and client-side DOM sinks — and each needs encoding appropriate to **where the input lands**, not a single blanket filter. Blocking `<script>` is not enough: attributes, JS strings and DOM sinks each offer a different break-out (`onmouseover`, `'-alert()-'`, `<svg onload>`). Modern frameworks (React/Angular/Vue) and CSP mitigate most of this by default, but legacy apps, admin panels and ERP/intranet systems — where output is hand-built — remain a rich source of XSS.

*Labs completed on PortSwigger Web Security Academy — authorized lab environment only.*
