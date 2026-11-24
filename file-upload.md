# File Upload Vulnerabilities — PortSwigger Web Security Academy

> Writeup covering remote code execution via web shell upload
> (OWASP Top 10 — A03/A05). Author: Yuri Vinhote · Tools: browser, Burp Suite

File upload flaws arise when an application lets users upload files and then
stores or serves them without properly validating **what** the file is and
**how** it will be handled. The worst case: the server saves an executable file
(a PHP/JSP script) in a web-reachable folder and executes it on request — turning
"upload a profile picture" into **remote code execution**.

---

## Remote code execution via web shell upload

The avatar upload performed **no validation** on the uploaded file before storing
it on the server's filesystem, under a web-accessible path (`/files/avatars/`).

**Attack**
1. Created a one-line PHP **web shell** that reads a server file:
   ```php
   <?php echo file_get_contents('/home/carlos/secret'); ?>
   ```
2. Uploaded it as the account avatar (`shell.php`) — no check blocked it.
3. Located the stored file's URL (the avatar `src` pointed to
   `/files/avatars/shell.php`).
4. Requested that URL directly. The server **executed the PHP** and returned the
   contents of `/home/carlos/secret`, which solved the lab.

**Impact** — Remote Code Execution. `file_get_contents` was just a demonstration;
the same shell can run arbitrary commands (`system($_GET['c'])`), read configs
and credentials, or open a reverse shell for full server control.

**Fix**
- Validate the file **server-side**: check the extension against an allowlist,
  the `Content-Type`, and the real content (magic bytes) — not just one of them.
- Store uploads **outside the web root**, or in a location where the server will
  not execute them, and serve them through a handler that sets a safe
  `Content-Type` and `Content-Disposition`.
- Rename uploaded files to a generated name; never trust the client filename.

---

## Common bypasses (when validation exists)

Real apps do validate, so exploitation becomes a filter-bypass game — the same
muscle as the SSRF and path-traversal labs:

| Server defense | Bypass idea |
|---|---|
| Blocks `.php` extension | alternative executable extensions (`.phtml`, `.php5`, `.phar`) |
| Checks `Content-Type` only | upload PHP but set `Content-Type: image/jpeg` |
| Checks magic bytes only | prepend valid image header, then PHP after it |
| Strips `.php` once | `shell.pHp`, `shell.php.`, `shell.php%00.jpg` (null byte) |
| Blocks execution in upload dir | path traversal in filename to write elsewhere (`../`) |
| Config-based (`.htaccess`) | upload a crafted `.htaccess` to re-enable execution |

---

## Key takeaway

An upload field is a direct channel from the attacker to the server's filesystem.
The vulnerability is never "the user uploaded a file" — it's the combination of
**weak validation + executable storage location**. Defense requires *both* a real
content check (allowlist + magic bytes) *and* a storage/serving strategy that
guarantees uploaded content is never executed.

This maps directly onto enterprise/ERP systems, which are full of upload
endpoints — NF-e / document attachments, product images, spreadsheet imports,
resume uploads. A backend that saves a received file to the server path without
validating content and execution context carries exactly this risk.

*Labs completed on PortSwigger Web Security Academy — authorized lab environment only.*
