# Web Cache Poisoning / (Web Cache Deception)

## 1. Ta'rif
Web cache poisoning — attacker saytning cache'iga zararli javobni "yuklaydi", boshqa foydalanuvchilar o'sha cache qiymatini oladi (unkeyed input dagger + sink birga). Web Cache Deception (WCD) — private data ochiq cache'ga tushib qoladi.

## 2. Qayerdan kelib chiqadi (Poisoning)
- Cache key: metode+path (query qismi ko'pincha unkeyed!)
- Unkeyed headers: `X-Forwarded-Host`, `X-Original-URL`, `X-Rewrite-URL`, `Origin`, `X-Host`
- Param-based: `?utm_source=...` cache key'ga kirmasa
- PoC chain: unkeyed input (header/param) → reflection/redirect (sink) → cache key'dan tashqari → boshqa user'ga
- `Vary` header yetishmasa: response user-specific (geo, auth) cache'lanib qolsa

## 3. Web Cache Deception (WCD)
- `GET /account` (auth) lекин cache qoidalar o'xshash static: `/account/nonexist.css` → 200 + account page cache
- Path/X-Original-URL mixup — reverse proxy static ext detection
- Kazakh: user profiling: logout redirect cache; json abc endpoint
- Logout redirect (window: utm non-key) → static page cached

## 4. Metodologiya (poisoning)
1. Cache aniqlang: `X-Cache: hit/miss`, `Age`, `CF-Cache-Status`, `Via`, `X-Varnish`.
2. `?foo=123` — key'ga kiradimi? qayta javob: cache'da ko'rinish.
3. Unkeyed headers (o'zi refleksiya):
   - `X-Forwarded-Host: evil` → redirect/absolute URL'da refleksiya
   - `X-Original-URL: /evil` → 302 location reflection
   - `X-Rewrite-URL`
   - `X-Forwarded-Scheme`, `Forwarded`, `X-Host`
4. Sink toping: response header (Location, Domain cookie), body (redirect, endpoint, canonical), import/script src.
5. Payload (unkeyed) + sink kombinatsiya — key'dan tashqari input:
   - `GET /?cachebuster=1&X-Forwarded-Host=evil.com` → reflect header.
6. Key'in Heisens karakter: `//`, `?`, fragment o'zgartirib request'ni barava route, cache key boshqa.
7. Cookie header unkeyed bo'lsa — `Cookie` himoyalanmagan auth page cache'lanadi (WCD).

## 5. PoC
```
# Poison via unkeyed query param reflected in canonical
GET /about?utm_source=<script>alert(1)</script> HTTP/1.1
→ 301 Location: /about?utm_source=<script>...  (cached — key: /about)
→ boshqa user cache'dan zararli redirect oladi

# X-Forwarded-Host reflection
GET /english HTTP/1.1
Host: target.com
X-Forwarded-Host: evil.com
→ response'da: <link rel="canonical" href="https://evil.com/english">
```
WCD:
```
GET /myprofile/foo.css HTTP/1.1   # proxy static deb cache'laydi
→ 200 account page html
→ keylar: /myprofile/foo.css
```
## 6. Nimalarga ahamiyat berish (checklist)
- [ ] `X-Cache`/`Age`/`Vary`/`CDN` — cache muhit borligi
- [ ] Unkeyed query/header input'larni ayri — refleks qayerda
- [ ] Canonical/open-redirect sink — cache'da import
- [ ] `Cache-Control: private` bo'lgan auth page'larni ham sinash (WCD)
- [ ] Virtual host (X-Forwarded-Host) CDN config
- [ ] WCD: `?.css`, `/nonexist.css`, `X-Original-URL: /nonexist.css`
- [ ] 30 days SameSite effect — redirect window 30 xu
- [ ] Cookie unkeyed bo'lsa — per-user cache difference bo'lmaydi
## 7. Himoya
- Cache key: include all security-relevant headers; `Vary`
- Do not cache auth pages (`Cache-Control: no-store`), check cookies
- Validate X-Forwarded-Host allowlist, avoid reflection
- Cache deception: match full path (not extension) on backend
- Use `Cache-Control: private` etc