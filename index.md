# WEB Penetration Testing Notes — INDEX

Har bir vuln uchun alohida papka va `note.md`. Har bir note tarkibi:

1. **Ta'rif / nima bu**
2. **Qayerdan kelib chiqadi (root cause)**
3. **Turlari**
4. **Metodologiya (step-by-step)**
5. **Payload'lar (sinab ko'rish)**
6. **Request misollari**
7. **Zaif code misollari (PHP/JS/Python/Java)**
8. **Nimalarga ahamiyat berish kerak (checklist)**
9. **Himoya (remediation)**

---

| # | Zaiflik | Papka |
|---|---------|-------|
| 1 | SQL Injection | `sqli/note.md` |
| 2 | XSS (Reflected/Stored/DOM) | `xss/note.md` |
| 3 | LFI / RFI / Path Traversal | `lfi/note.md` |
| 4 | SSRF | `ssrf/note.md` |
| 5 | XXE | `xxe/note.md` |
| 6 | RCE + Command Injection | `rce/note.md` |
| 7 | SSTI (Server-Side Template Injection) | `ssti/note.md` |
| 8 | IDOR / BOLA | `idor/note.md` |
| 9 | Business Logic Bugs | `business-logic/note.md` |
| 10 | Authentication flaws | `auth/note.md` |
| 11 | Session Management | `session/note.md` |
| 12 | CSRF | `csrf/note.md` |
| 13 | File Upload RCE | `file-upload/note.md` |
| 14 | Open Redirect | `open-redirect/note.md` |
| 15 | CORS misconfiguration | `cors/note.md` |
| 16 | Insecure Deserialization | `deserialization/note.md` |
| 17 | Race Condition / TOCTOU | `race-condition/note.md` |
| 18 | Broken Access Control | `access-control/note.md` |
| 19 | NoSQL Injection | `nosql/note.md` |
| 20 | LDAP / XPath / XML Injection | `injection-extra/note.md` |
| 21 | Mass Assignment | `mass-assignment/note.md` |
| 22 | HTTP Request Smuggling | `http-smuggling/note.md` |
| 23 | Web Cache Poisoning | `cache-poisoning/note.md` |
| 24 | Prototype Pollution | `prototype-pollution/note.md` |
| 25 | GraphQL Abuse | `graphql/note.md` |
| 26 | API security (REST/JSON) | `api/note.md` |
| 27 | JWT Attacks | `jwt/note.md` |
| 28 | OAuth 2.0 / SSO Abuse | `oauth/note.md` |
| 29 | Log Poisoning → RCE | `log-poisoning/note.md` |
| 30 | LLM / AI Attacks (Prompt Injection...) | `llm-attack/note.md` |
| 31 | WebSocket Abuse | `websocket/note.md` |
| 32 | DOM Clobbering | `dom-clobbering/note.md` |
| 33 | Clickjacking | `clickjacking/note.md` |
| 34 | Host Header Injection | `host-header/note.md` |
| 35 | HTTP Parameter Pollution | `hpp/note.md` |
| 36 | Subdomain Takeover | `subdomain-takeover/note.md` |
| 37 | CSP Bypass | `csp/note.md` |

---

## Umumiy methodology (barcha vuln uchun)

1. **Enumeration** — foydalanuvchi input qayerda va qanday ishlaydi (GET/POST headers/cookies/JSON body).
2. **Har bir input'ni zaif deb qarash** — ID, name, file path, redirect URL, template, XML, JSON.
3. **Fingerprinting** — texnologiya (Wappalyzer, response headers), framework, version.
4. **Auto + manual** — Burp Suite katalog + qo'lda "aql bilan" payload.
5. **Validation** — vuln ekanini isbotlash (limit yo'q sintaksis xato ham HUJJAT bo'ladi — evidence!).
6. **Impact baholash** — faqat bo'sh zaiflik emas, real impact ko'rsatish (RCE, data leak, account takeover).
7. **WAF obhod** — environment, unicode, encoding, case-swapping, chunking.

## Asosiy vositalar
- Browser + DevTools (Network, Storage, Debugger)
- Burp Suite Professional (Repeater, Intruder, Decoder)
- ffuf / feroxbuster (fuzzing)
- SQLMap, nuclei, nikto (auto)
- ffuf, wfuzz (wordlist fuzzing)
- dirsearch / gobuster (directory bruteforce)
- `curl`, `jq`, `openssl`, `nmap`

## Checklist: istalgan request'ni ko'rganingda
- [ ] Qaysi parametr input sifatida ishlatiladi?
- [ ] Server da qayerga yetib boradi? (DB, FS, HTTP request, template, shell)
- [ ] Encoding/decoding bormi? (URL, base64, JSON->XML)
- [ ] Auth check qayerda? Client'da (JS) yoki server'da?
- [ ] Rate limit / size limit bormi?
- [ ] Response'dagi error/har-har xatti (verbose error = info leak)