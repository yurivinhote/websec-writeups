# Path Traversal & OS Command Injection — PortSwigger Web Security Academy

> Writeup covering file path traversal and OS command injection
> (OWASP Top 10 — A01/A03). Author: Yuri Vinhote · Tools: browser, Burp Suite

Two classic server-side injection flaws. Both share one root cause: **user input
crosses from data into a privileged context** — the file system in one case, the
OS shell in the other — because the application builds a path or a command by
concatenating untrusted input.

---

## Path Traversal — simple case

The product image was served by a filename parameter the server joined to a base
directory:

```
GET /image?filename=25.jpg   →   /var/www/images/25.jpg
```

**Attack**
- Set `filename=../../../etc/passwd`. Each `../` climbs one directory, escaping
  the images folder and resolving to `/etc/passwd`.
- The response returned the full file — a list of system users, including the
  presence of `mysql`, `postgres` and `mongodb` accounts (useful recon: the
  installed services are disclosed).

**Impact** — Arbitrary file read. `/etc/passwd` is only the proof; the real
prize is config files with credentials (`.env`, `application.properties`),
source code, and SSH keys (`/root/.ssh/id_rsa`).

**Fix** — Avoid passing user input to filesystem APIs. If unavoidable, resolve
the canonical path and verify it stays within the intended base directory; use
an allowlist of filenames.

### Common filter bypasses (defense-in-depth is weak against these)
| Server defense | Bypass |
|---|---|
| blocks `../` | absolute path `/etc/passwd` |
| strips `../` once (non-recursive) | `....//....//etc/passwd` |
| strips `../` then URL-decodes | double encoding `..%252f..%252fetc/passwd` |
| requires base prefix | `/var/www/images/../../../etc/passwd` |
| requires `.png` extension | null byte `../../../etc/passwd%00.png` |

---

## OS Command Injection — simple case

The stock-check feature ran a shell command built from the request parameters:

```
POST /product/stock      body: productId=4&storeId=1
server runs:  stockreport.pl 4 1
```

**Attack**
- Injected a shell separator into `storeId`: `storeId=1|whoami`.
- The server executed `stockreport.pl 4 1|whoami`; the response returned
  `peter-E8yKga` — the output of `whoami` **running on the server**.

**Impact** — Remote Code Execution (RCE), the highest-severity class. `whoami`
is proof; from here an attacker can read files, pull cloud credentials via SSRF
to `169.254.169.254`, or open a reverse shell for full control.

**Fix** — Never build shell commands by concatenating user input. Use APIs that
pass arguments as a separate array (no shell interpretation); if a shell is
unavoidable, strictly allowlist the input.

### Shell separators (command-injection arsenal)
```
|    pipe (chains output)
;    sequential execution
&    background execution
&&   run if previous succeeded
||   run if previous failed
`cmd`  or  $(cmd)   command substitution
```

---

## Key takeaway — the lesson that ties it together

Business validation is **not** security sanitization. Input can pass every
business rule (valid number, existing client, correct date) and still inject,
because the bug lives in **how the input is assembled** into a path or command,
not in whether it's a valid business value. Two consequences the attacker
exploits:

- **The weak field is the door.** The guarded numeric field is safe; the
  loosely-validated free-text field (a description, a reference, a store ID
  treated as string) that reaches a dangerous sink is the injection point.
- **Injection point ≠ impact target.** A payload entered in an innocuous text
  field can alter a protected value elsewhere (e.g. SQL in a "notes" field
  running `UPDATE ... SET balance = ...`), including **second-order** injection
  that fires later, in a different routine, far from where it was stored.

The defense is always the same: treat untrusted input as pure data at the point
of assembly — parameterized queries for SQL, argument arrays for the shell,
context-aware encoding — never string concatenation into an interpreter. This is
exactly where enterprise/ERP back-ends are rich targets: many free-text fields,
hand-built dynamic queries, and data entered in one screen consumed in another.

*Labs completed on PortSwigger Web Security Academy — authorized lab environment only.*
