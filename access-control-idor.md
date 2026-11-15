# Access Control & IDOR — PortSwigger Web Security Academy

> Writeup covering 5 access control labs (OWASP Top 10 — A01: Broken Access Control).
> Author: Yuri Vinhote · Tool: Burp Suite, browser DevTools

The common root cause across every lab below: **the server trusted something the client controls or can see** (a path, a cookie, a parameter, an object ID) to decide access, instead of enforcing authorization server-side against the logged-in session.

---

## 1. Unprotected admin functionality

An admin panel existed at a "hidden" path, but the path was not protected and was disclosed in `/robots.txt`.

**Attack**
1. Requested `/robots.txt`, which listed `Disallow: /administrator-panel`.
2. Browsed directly to `/administrator-panel` — it loaded with no authentication.
3. Deleted the target user from the exposed admin interface.

**Impact** — Any anonymous user reaches admin functions (delete users, etc.).

**Fix** — Enforce an admin-role check server-side on every sensitive route. Hiding a path is not access control (security through obscurity).

---

## 2. User role controlled by request parameter

The application decided admin status from a value the client controls (an `Admin` cookie / request parameter).

**Attack**
1. Logged in as a normal user; `/admin` returned 401 (not authorized).
2. In Burp, the request carried `Cookie: Admin=false`.
3. Changed it to `Admin=true` and replayed — the admin interface loaded.
4. Deleted the target user (`/admin/delete?username=carlos` returned 302).

**Impact** — Privilege escalation to admin by editing a client-side value.

**Fix** — Derive the role from the server-side session, never from a cookie or parameter the user can edit.

---

## 3. Insecure Direct Object Reference (IDOR)

Chat transcripts were served from predictable files (`/download-transcript/N.txt`) with no ownership check.

**Attack**
1. Started a live chat as my user and clicked "View transcript".
2. The download pointed to `/download-transcript/2.txt`.
3. Changed the ID to `/download-transcript/1.txt` — got another user's transcript, which contained their password.
4. Logged in as that user.

**Impact** — Any user reads other users' private data (and credentials) by changing an ID.

**Fix** — On every object access, verify server-side that the resource belongs to the logged-in user (ownership check). Direct references alone are not authorization.

---

## 4. User ID controlled by request parameter, with unpredictable IDs

Account pages were keyed by a GUID (`/my-account?id=<guid>`) instead of a guessable number — but the GUID leaked elsewhere.

**Attack**
1. Noticed my account used a random GUID in `id=`.
2. Found the target user (`carlos`) as the author of a blog post; his GUID was exposed in the author link.
3. Used that GUID in `/my-account?id=<carlos-guid>` and read his API key.
4. Submitted the API key to confirm the takeover.

**Impact** — Account takeover despite "unguessable" IDs, because the ID was disclosed publicly.

**Fix** — Unpredictable IDs are not access control. Still enforce an ownership check server-side; never expose a user's internal ID in public content.

---

## 5. User ID controlled by request parameter with password disclosure

The "My account" page rendered the user's password into the HTML (masked with `type="password"`), and the account was selectable by `id`.

**Attack**
1. Changed `id=` to the `administrator` account.
2. Opened the page source — the password field's `value` contained the admin password in clear text (only visually masked).
3. Logged in as `administrator` and deleted the target user.

**Impact** — Full admin account takeover by reading a password sent to the client.

**Fix** — Never send a password to the client, masked or not. It must never leave the server.

---

## Key takeaway

Broken Access Control is one failure wearing five masks: a hidden path, an editable cookie, a swappable object ID, a leaked "secret" ID, and a password shipped to the browser. The defense is always the same — **authorization is decided on the server, per session, per object; the client is never trusted to enforce it, and sensitive data never leaves the server.** This is the #1 class of flaw in real ERP/API back-ends, where endpoints return records by ID without checking ownership.

*Labs completed on PortSwigger Web Security Academy — authorized lab environment only.*
