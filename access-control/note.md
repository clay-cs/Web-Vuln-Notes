# Broken Access Control (BOLA / BAC / Vertical)

## 1. Ta'rif
Broken access control — foydalanuvchi o'z huquqidan ortiq ressurusga / funksiyaga erishsa: admin panel, boshqa rolning API, boshqa org ma'lumoti. 2021 OWASP #1.

## 2. Qayerdan kelib chiqadi
- Endpoint'da authorization tekshirilmaydi (tekshirilmaslik "hope")
- Client-only role/perms (JS, cookie flag)
- Server endpoints role'dan qat'iy mavjud (admin REST)
- Method-based: `GET /api/users` vs `POST`
- Header trust: `X-Admin: true`, `X-Role`, `X-Forwarded-Host`
- Hidden menu/endpoint (dir bruteforce → admin.php)
- Reference-based: IDOR (qarang idor/note) — horizontal; BAC — vertical
- Mass assignment role o'zgartirish

## 3. Turlari
| Tur | Tavsif |
|------|--------|
| BAC vertical | user → admin funktsiyasi |
| BOLA (IDOR) | user ob'ekti |
| Broken Function Level Auth | admin api user'dan ochiq |
| Horizontal | bir xil role/org |
| Mass assignment | `role=admin` imkoni |

## 4. Metodologiya
1. Kenngasi kengaytirish: auth oldindan foydalaniladigan endpointlar ro'yxati qiling.
2. Har bir request MD:
   - Cookie/bearer bilan oddiy user — admin api'ni sinang
   - Boshqa rol/org'dan
3. Aniqlash: naive differents between role response:
   - 200 vs 403; 200 va data — zaif
4. Bypass yo'llari:
   - HTTP method override: `X-HTTP-Method-Override: DELETE`, `_method=delete`
   - Path filter: `/api/admin/users` → `/api/v1/users`, `./api/../admin`, case
   - Param foydalanish: `?userId=1` boshqa user
   - Header: `X-Admin: true`, `X-Original-URL: /admin`
   - Cookie/flag: `isAdmin=1`
5. Admin panel: `/admin`, `/administrator`, `/panel`, `/manage`, `/internal` fuzz (ffuf).
6. GraphQL — admin field amalga oshirmagan fetch (.graphql/note)
7. IDOR bilan birlashtirish: boshqa org user'ning data horozontal.

## 5. Request misollari
```
# Admin endpoint user token bilan
GET /api/admin/users HTTP/1.1
Authorization: Bearer <user_token>

# Method override
DELETE /api/users/1 HTTP/1.1
X-HTTP-Method-Override: DELETE

# Header trust
GET /api/me HTTP/1.1
X-Role: admin

# Original URL
GET /user/profile?id=1 HTTP/1.1
X-Original-URL: /admin/delete?id=1
```
## 6. Zaif code misollari
```python
# ZAIF — role tekshirilmaydi
@app.get("/api/admin/users")
def admin_users():
    return [u for u in users]   # hech qanday is_admin check yo'q!

# XAVFSIZ
@app.get("/api/admin/users")
@login_required
@admin_required   # middlware
def admin_users(): ...
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Har bir API rol-huquq tekshiruvini — `Authorization: Bearer` yetarli emas
- [ ] Method/header override — `X-HTTP-Method-Override`, `_method`
- [ ] Path fuzz: `/api/v2/admin`, `/admin`, `console`, `/api/users.all`
- [ ] Client flags (`isAdmin`, `role` local storage)
- [ ] Mass assignment (`PUT` json `{"role":"admin"}`)
- [ ] GraphQL kolleksiyon: field-based denials
- [ ] Org/tenant: boshqa org id sinash (horizontal org)
- [ ] Hidden internal endpoints (`/internal`, `/debug`, `/swagger`)
- [ ] OPTIONS / swagger / API documentation leak
## 8. Himoya
- Deny-by-default, role-based access kontrol (RBAC/ABAC)
- Server-side authorization on each endpoint
- Never trust client headers/flags; verify JWT claims
- Consistent enforcement, test matrix rol x resource x method