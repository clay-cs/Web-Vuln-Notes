# Authentication Vulnerabilities

## 1. Ta'rif
Auth — foydalanuvchi kimligini isbotlash. Zaiflik: bypass login, brute-force, password reset abuse, predictble token, 2FA bypass, default creds, session fixation, response tampering. Average impact: account takeover (ATO).

## 2. Qayerdan kelib chiqadi
- Login endpoint: ishlab chiquvchi autobypass (hardcoded admin qushiq)
- SQLi/NoSQLi in login (auth bypass) → sqli/nosql note
- Default/weak password, no rate limit
- Password reset: token predictble (`id=1`), `email` param o'zgartirish, token log'da
- 2FA: response'da code mavjud, 2FA kod'ni parametrda o'zgartirib boshqa user, race 2FA
- Predictable session: username bilan o'zgaruvchi, seq
- Username enumeration: "user topilmadi" vs "password noto'g'ri"
- OAuth misconfig → oauth/note
- JWT algorithm → jwt/note

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| Brute-force | user/pass, no lockout |
| Credential stuffing | leak'd list |
| Password reset flaw | link/token abuse |
| 2FA bypass | batch, response tampering, brute |
| Username enumeration | response/error/time |
| Auth bypass | param, cookie, logic |
| Default creds | admin/admin, admin/password |
| Session fixation | pre-auth session ID |

## 4. Metodologiya
1. Login flow'ni to'liq o'rganing: register → login → reset → 2FA → session.
2. Response farqlari: `Invalid username` vs `Invalid password` — enumeration.
3. Brute-force: Burp Intruder + wordlist (rockyou top-1000), rate limit'ni aylanib (XFF, rotate IP).
4. Password reset:
   - Link token: predictble? `reset.php?id=1` (admin user)
   - `email`/`user` param'tan o'zgartiring
   - Reset code'ni API response/JS'da izlash
   - Host header ichida reset link (host header injection)
5. 2FA:
   - Response'da `otp`/`verification_code` bo'lsa
   - Batch/race — 100 parallel request kodni buzish (Brute force small space: 6 soni)
   - 2FA'ni mustaqil bajarish (step-skip)
   - Predictble (phone'ga yuborilgan sms) — bir xil code qayta
6. Session:
   - Login'dan keyin session o'zgarmasa → session fixation
   - Session predictable (JWT/eger)
7. OAuth/JWT misconfig alohida boblar.

## 5. Request misollari
```
# Username enumeration
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded
username=admin&password=wrong
# vs admin2 →

# Password reset URL manipulation
POST /reset HTTP/1.1
user=admin&email=attacker@evil.com
# yoki
GET /reset?token=<predict>&id=1 HTTP/1.1

# 2FA bypass — response
POST /verify-2fa HTTP/1.1
{"user": "victim", "code": "000000"}   # brute 000000-999999 (time)
```

## 6. Zaif code misollari
```php
// ZAIF — hardcoded bypass
if ($_POST['pass'] == 'admin123' || $_POST['isadmin'] == '1') {...}

// ZAIF — password reset
$id = $_GET['id'];   // token yo'q; faqat id
reset_password($id, $_GET['newpass']);

// ZAIF — 2FA
if ($_POST['code'] == $stored_code) {...}   // rate limit yo'q, kichik space

// ZAIF — reset link host
$link = "http://" . $_SERVER['HTTP_HOST'] . "/reset?t=$token";
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Response va status farqi — enumeration
- [ ] Reset email foydalanuvchi email'ini o'zgartirishga/USDT
- [ ] Token predictble, `id=1`, `md5(username)`
- [ ] 2FA code response'da, log'da, predictable (timestamps)
- [ ] Rate limit yo'q → brute (not big, always test!)
- [ ] Session old — fixation; bir xil session ID
- [ ] Cookie'da `admin=0` → `admin=1` (cookie tampering)
- [ ] Remember-me / persistent token predictble
- [ ] OAuth/JWT/SSO errors — bo'sh state
- [ ] Default creds test: admin/admin, admin/password123, admin/qwerty

## 8. Himoya
- Rate limiting/lockout, MFA, verify email/phone, secure tokens (CSPRNG), expire
- HttpOnly Secure SameSite cookie, unique session registers
- Password policy + breach check, never trust client flags