# Session Management

## 1. Ta'rif
Authentication'ning davomi — session: server user'ni session ID orqali taniydi. Zaiflik: predictable/tame session, session fixation, o'chmagan cookie, httponly yo'q, secure yo'q, samе-пар viral, CSRF ga ochiq, session'i logda/qisqa.

## 2. Qayerdan kelib chiqadi
- Session ID avtomatik username/timestamp/seq asosida
- Server session'ni keyin o'chirmaydi (logout da)
- Cookie flaglar yo'q: Secure, HttpOnly, SameSite
- Session'da sensitive data (is_admin flag) — client'da
- Session'in URL da `/sid=...`
- ID o'zgartirilmaydi login'dan keyin (fixation)

## 3. Turlari
| Zaiflik | Misol |
|---------|-------|
| Predictable session ID | `md5(username)`, `userID-123` |
| Session fixation | pre-login ID saqlanadi |
| Sensitive data in session | `role=admin` cookie |
| Weak expiry | 30 kun o'chmaydi |
| Cookie flags | no HttpOnly/Secure |
| Session DNS rebinding samе-site | no SameSite → CSRF |

## 4. Metodologiya
1. Login'dan keyin session cookie'ni oling — uzunligi, entropiya:
   - 6-8 farq belgili, kalit uchun URL-Safe? predictble?
   - `Set-Cookie: sid=admin` → o'zgartiring
2. Session fixation test:
   - `Set-Cookie: sid=ATTACKER` qo'yib, login qiling — login'dan keyin sid o'zgarsa safe, o'zgarmasa zaif.
3. Predictable test: bir necha login → bir necha sid: pattern? 
   - Berilgan sonida token entropiyani hisoblang (32 hex miqdori).
4. Cookie flags:
   - `Secure` yo'q → HTTP bo'yicha yuboriladi (MitM)
   - `HttpOnly` yo'q → XSS bilan steal
   - `SameSite=None` → CSRF
5. Session leak:
   - Referer'da sid (external redirect), log'da, `?sid=` URL
6. Logout'dan keyin sid serverda aktiv qolishi — remember-me mavjud.
7. Parallel active session: logout boshqalarini o'chirmaydi.

## 5. Request misollari
```
# Fixation noqulay: cookie'ni berib login
GET /login HTTP/1.1
Cookie: sessionid=c9f7b3a1
# qaytadigan Set-Cookie o'zgarsa yaxshi, o'zgarmasa fixation -> exploit

# Admin flag in cookie
Cookie: isadmin=0; session=...
Cookie: isadmin=true; session=...

# sid in URL
GET /account?sid=abc123 HTTP/1.1
```

## 6. Zaif code misollari
```php
// ZAIF — session id predictble
$sid = md5($username);          // bilinadigan
$sid = $_GET['sid'];            // user tanlagan → fixation
setcookie('sid', $sid);

// ZAIF — eski session o'chmaydi
logout():  // session_destroy() yo'q

// ZAIF — flags yo'q
setcookie('sid', $id);          // secure/httponly yo'q
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Session token uzunligi/entropiya — qanchalik predictble
- [ ] Login/login session_n (regenerate) yo'qmi
- [ ] Logout → serverda session destroy
- [ ] HttpOnly, Secure, SameSite (=Lax/Strict)
- [ ] URL/Referer/access log'da sid
- [ ] Client-side role/permission flags
- [ ] Remember-me token predictble, endless
- [ ] Session in multi-server/no sticky balans uzin

## 8. Himoya
- CSPRNG session, login'da regenerate, logout destroy
- Cookie: `Secure; HttpOnly; SameSite=Lax`
- Session server'da, role/permissions server'da
- Periodic expiry + inactivity timeout