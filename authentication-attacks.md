# Authentication Attacks — PortSwigger Web Security Academy

> Writeup covering 6 authentication labs (OWASP Top 10 — A07: Identification and Authentication Failures).
> Author: Yuri Vinhote · Tool: Burp Suite (Proxy, Intruder, Repeater)

---

## 1. Username enumeration via different responses

The login returned a different error message for valid vs invalid usernames (e.g. "Invalid username" vs "Incorrect password"), leaking which accounts exist.

**Attack**
1. Sent a login request to Burp Intruder (Sniper), marking the `username` field.
2. Loaded the candidate-usernames wordlist and ran the attack.
3. Sorted by response **Length**; the outlier was the valid username.
4. Repeated the brute force on the `password` field for that user.

**Impact** — Account enumeration + password brute force leading to authentication bypass.

**Fix** — Return a generic, identical error for every failure. Add rate limiting / lockout.

---

## 2. Username enumeration via subtly different responses

Same leak, but the difference was a single character (a missing period), invisible to the eye.

**Attack**
1. Response **Length** was noisy (a random analytics ID changed the size each request).
2. Used Intruder **Grep - Match** on the exact error string instead.
3. The valid username was the one whose response did **not** match the string.

**Impact** — Same as above; shows that Length isn't always reliable — grep the content.

**Fix** — Byte-for-byte identical responses across all auth-failure paths.

---

## 3. Broken brute-force protection (IP block)

The app blocked an account after 3 failed attempts, but the counter reset on any successful login.

**Attack**
1. Intruder **Pitchfork** with two lists: username alternating `carlos`/`wiener`, password alternating candidate/`peter`.
2. Each attempt on `carlos` was followed by a valid `wiener`/`peter` login, resetting the lockout.
3. Set **1 concurrent request** (Resource pool) to preserve order.
4. The valid password for `carlos` returned **302** (and a different Length).

**Impact** — Lockout fully bypassed, unlimited password guessing.

**Fix** — Count failures per account/IP without resetting on valid login; real time-based lockout + rate limit.

---

## 4. 2FA simple bypass

After login the app asked for the 2FA code but never verified on the server that the step was completed — browsing straight to `/my-account` skipped it.

**Attack**
1. Logged in as `carlos`, reaching the 2FA code page.
2. Instead of entering the code, navigated directly to `/my-account`.
3. Got full access with no second factor.

**Impact** — 2FA completely bypassed; account access with only username + password.

**Fix** — Keep a server-side "2FA pending" state; block every authenticated route until the second factor is validated. Never trust the client to follow the flow.

---

## 5. Password reset broken logic

The password-reset request carried the target `username` in its body and the server trusted it — changing it reset another user's password.

**Attack**
1. Requested a reset for my own account (`wiener`) and intercepted the final submit in Burp.
2. The body had `temp-forgot-password-token=...&username=wiener&new-password-1=...`.
3. Changed `username=wiener` to `username=carlos` and forwarded.
4. `carlos`'s password became the one I set, account takeover.

**Impact** — Full account takeover of any user.

**Fix** — Bind the reset token to its user on the server; ignore any client-supplied username.

---

## 6. Username enumeration via response timing

Non-existent users got a fast response; existing users triggered a (slow) password check — the response **time** leaked valid accounts. IP-based blocking was bypassed with `X-Forwarded-For`.

**Attack**
1. Added header `X-Forwarded-For` (spoof a new IP per request) and a very long password (amplifies the time delta).
2. Intruder **Pitchfork**: position 1 = numbers 1-100 (X-Forwarded-For), position 2 = username wordlist.
3. Sorted by **Response received** (time); the consistently slowest username (`app01`) was valid.
4. Brute-forced its password to confirm, logged in.

**Impact** — Account enumeration through a side channel (timing), even with IP blocking in place.

**Fix** — Constant-time auth responses (check a dummy hash even when the user doesn't exist); block per account, not just per IP.

---

## Key takeaway

Each app leaked through a different channel — message text, response length, timing, or trusting client-supplied data. The root pattern: the server must never reveal which code path ran, and never trust the client to enforce authorization or flow.

*Labs completed on PortSwigger Web Security Academy — authorized lab environment only.*
