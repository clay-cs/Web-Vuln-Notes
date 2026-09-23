# Subdomain Takeover

## 1. Ta'rif
Subdomain takeover — saytning CNAME/NS boshqa providerga (GitHub Pages, Heroku, S3, Azure, DigitalOcean, fastly...) ishora qiladi, lekin provider'da resurs egallanmagan/o'chirilgan. Attacker uni egallab, o'sha subdomain'da o'z kontentini ko'rsatadi → cookie poytaxti, phishing, SSL Trust orqali hijack.

## 2. Qayerdan kelib chiqadi
- DNS'da `CNAME` bor, ama target noexist / unclaimed
- Development'da oldi o'chirilgan, CNAME qoldiq
- Pre-listing orqali egallash mumkin (barely)
- Example: `oldsite.target.com CNAME oldsite.github.io` — GitHub repo yo'q

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| CNAME + unclaimed PaaS | GitHub/Heroku/S3/Azure/fastly |
| Deleted domain dangling CNAME | expired domain |
| NS delegation | NS-boshqa (rare) |
| A record IP reuse | IP'ga boshqa service (`can-i-take-over-xyz`) |
| Failing `Instead of OpenStack | `Elastic IP` reassign |

## 4. Metodologiya
1. Subdomain enumeration:
   - Passive: CRT.sh, securitytrails, sublist3r, amass passive
   - Active: `dnsrecon`, `amass enum -active`, `massdns`
2. DNS tekshirish — CNAME ni kuzat: `host -t cname sub.target.com`.
   - CNAME bor va u-hech qayerga ishora qilmaydi → takeover
3. Failing consistent (404 from PaaS) kirish:
   - GitHub: `curl sub.target.com` → "There isn't a GitHub Pages site here."
   - Heroku: "No such app"
   - AWS S3: `NoSuchBucket`
4. Takeover proof: PaaS'da o'z resurs yaratib kontakt (email, POC).
5. Checks: `crt.sh` JSON; tool `subjack`, `nuclei`, `can-i-take-over-xyz` wordlist.

## 5. Commandlar
```bash
# CNAME topish
for s in $(cat subs.txt); do dig +short CNAME $s.target.com; done
# yoki
subjack -w subs.txt -t 100 -ssl -o res.txt

# nuclei (template "takeover-detection")
nuclei -l subs.txt -t http/takeovers/

# crt.sh
curl "https://crt.sh/?q=%25.target.com&output=json"
```
PoC (GitHub Pages misol):
```bash
# 1) sub CNAME -> username.github.io (repo yo'q)
# 2) GitHub'da yangi repo yaratib kontent qo'yish
# 3) curl sub.target.com -> "POC - Takenover"
```
## 6. Nimalarga ahamiyat berish (checklist)
- [ ] CNAME'dan provider detect: `github.io`, `azurewebsites.net`, `s3.amazonaws.com`, `herokuapp`, `fastly`, `shopify`, `surge`, `readme.io`, `gitlab.io`, `cloudfront` (u niche)
- [ ] "Dangling CNAME" — root DNS'da qolgan
- [ ] Pending/expired domain (used old)
- [ ] Email/SSL — `*.target.com` cert masalalari
- [ ] `mail.` `dev.` `test.` `staging.` `beta.` `dev-demo.` — tez-ket-ayi
- [ ] Service unclaimed — hem `Advert` (S3/cloud)
- [ ] Panel mos: Amazon S3 "NoSuchBucket", Azure "Website not found", GitHub "There isn't a GitHub Pages site here"
- [ ] Impact: cookie/SSL trust, phishing (full parent-typo name)
## 7. Himoya
- DNS records/fomaning audit (expired CNAME'lar o'chiriladi) — `can-i-take-over-xyz`
- PaaS resurslari o'chirilmasdan DNS o'zgartirilmasin
- Monitor/automated DNS scanning (nuclei daily)
- Own all subdomains registered; verify cert/hostname ownership