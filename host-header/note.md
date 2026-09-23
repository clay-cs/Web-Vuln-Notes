# Host Header Injection

## 1. Ta'rif
Host header — sayt nomi. Zaiflik: server `Host` header'idan qandaydir link/redirect/URL/rodlight, reset email, asset yuklash uchun foydalansa — attacker bu header'ni o'zgartirib bolsa → poizoning, acc poisoning, cache poisoning, SSRF/char url.

## 2. Qayerdan kelib chiqadi
- `$_SERVER['HTTP_HOST']` / `req.headers.host` ni parol/email/redirect/link'da ishlatish
- Canonical link, absolute redirect, password reset link (email), asset URL
- Virtual host routing: bir server, ko'p site — Host header yo'naltiriladi
- Cache: Host header cache key bo'lmasa → cache poisoning
- Drop/error: `Host` bo'lmasa default route (vhost default)

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Password reset poizoning | `Host: attacker.com` → email'dagi link attacker |
| Cache poisoning | key'siz Host → boshqa user'ga |
| Virtual host / routing | default bo'sh vhost |
| Domain mismatch | `Host: x.target.com` — bymashtirish |
| Route-to / SSRF | Host header XML/feed'ni net locator sifatida |

## 4. Metodologiya
1. Nima qaysi resource Host'dan: response'da absolute URL qidirish:
   - canonical, og:url, redirect Location, asset src/href, reset link.
2. Host'ni o'zgartirib sinash:
   ```
   Host: evil.com
   Host: victim.com@evil.com
   Host: victim.com.evil.com
   Host: victim.com:evil.com
   Host: attacker.com TO want receive bot callback → collaborator
   ```
3. HTTP/1.1: `Host` + 2-name (`X-Forwarded-Host` ko'pchilik):
   - `X-Forwarded-Host: evil.com`
   - `X-Forwarded-Server`, `Forwarded: host=evil.com`, `X-Original-Host`
4. Absent/duplicate Host:
   - 2 name: qaysini ishlatadi
   - `Host:` bo'sh — default route?
5. Password reset:
   - `POST /forgot`  Host: attacker
   - Email — `https://attacker.com/reset?token=...`
   → token'ni attacker oladi, account take-over (POC: o'z email)
6. Cache poisoning: `Host` cache unkeyed bo'lsa
7. SSRF: internal Vhost migrations.
8. Absolute path `Host` header old → `//host/...`.

## 5. Request misollari
```
POST /forgot HTTP/1.1
Host: evil.com
Content-Type: application/x-www-form-urlencoded

email=victim@mail.com
# → email: <a href="https://evil.com/reset?t=TOKEN">link</a>

GET / HTTP/1.1
Host: target.com
X-Forwarded-Host: evil.com
# → reflect: <link rel="canonical" href="https://evil.com/">
```
## 6. Zaif code misollari
```php
// ZAIF
$arr = $_SERVER['HTTP_HOST'];
$reset_link = "https://{$_SERVER['HTTP_HOST']}/reset?tk=" . $tk;  // email

// ZAIF
header("Location: https://" . $_SERVER['HTTP_HOST'] . "/login");  // Host overriding

// ZAIF cache poison:
// cache key — URL path, lekin response Host'ga qarab
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Reflection: canonical, redirect, asset src, og:url
- [ ] Password reset / verification email
- [ ] X-Forwarded-Host (FAAS, proxies)
- [ ] Duplicate Host (proxy allowlist)
- [ ] Cache poisoning — Host unkeyed, Vary
- [ ] IP: `Host: 127.0.0.1`, `Host: 192.168.0.1:8080` — internal routing
- [ ] HTTP/1.1 absolute-form (`GET http://target/...`)
- [ ] Column/word: `Host: target.com:2file` — port parsing error verbose
## 8. Himoya
- Whitelist Host for routing (deny default)
- Use relative URLs (or server config URL) not Host header
- Validate `Host` against allowlist; drop `X-Forwarded-Host` unless trusted proxy
- Reset links: fixed domain, token bound to user+session
- Include Host in cache key / `Vary: Host` where applicable