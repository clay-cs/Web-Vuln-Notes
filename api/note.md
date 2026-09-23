# API Security (REST / JSON)

## 1. Ta'rif
API'lar — frontend/backend kutubxon. 2023 OWASP API Top 10: BOLA, broken auth, excessive data exposure, rate limit, mass assignment, SSRF, injection, security misconfig, improper inventory, unsafe consumption.

## 2. Qayerdan kelib chiqadi
- API docs/swagger ochiq — hujum yuzasi bilinadi
- Versioning/orqaga mos kelmagan (v1/v2), debug endpoints
- Auth yetishmaydi: API key, Bearer, API key log/vs gib
- Excessive data: barcha field (internal fields)
- Error verbose — ishlovchi / inner endpoint /
- Rate limit yo'q — enumeration, brute
- Nested resources IDOR/BOLA
- CORS → cors note
- Mass assignment

## 3. Turlari
| Tur | Misol |
|-----|-------|
| BOLA/IDOR | `GET /api/users/{id}` |
| Broken Auth | tokens predictble, API key in doku |
| Excessive Data | `/api/me` barcha inner fields |
| Rate Limit | ningún brute |
| Security Misconfig | default CORS, debug logs, swagger live |
| Mass Assignment | `PUT` extra fields |
| SSRF | `webhook_url`, import URL |
| Injection | search param/JSON field SQLi |
| Improper Inventory | legacy/internal endpoints |

## 4. Metodologiya
1. **Discover**: swagger.yaml, api-docs, v3/api-docs, /swagger-ui, /redoc, /openapi.json — fuzz (ffuf).
2. Har bir endpoint uchun:
   - Auth test: no token → 200? -> broken auth
   - Har xil user token — horizontal IDOR
   - Role — vertical
3. HTTP methods: GET vs POST vs PUT vs DELETE — method-based access (GET /users vs POST)
4. Request body excess:
   - Extra fields (mass assign)
   - Over-fetching: `?fields=` yoki `?include=` qo'shish orqali sensitive field olingan
   - `?verbose=true`, `?debug=true`
5. Headers:
   - `X-API-Version`, `Accept: application/vnd+json`
   - `X-Original-URL`, `X-Rewrite-URL` (admin endpoints)
6. Rate limit:
   - X-Forwarded-For, X-Real-IP, `X-Client-IP`, `X-Originating-IP` aylantirish
   - `Client-IP: ` — bypass rate limit
   - Header `X-API-KEY` falsh? gal api
7. Injection — hamma parametr operatorlarini fuzz.
8. Auth flow: token expiration, refresh tokens, IDOR in token.

## 5. Request misollari
```
# Excessive data
GET /api/me?include=isAdmin,balance,api_keys HTTP/1.1

# Debug
GET /api/orders?debug=1 HTTP/1.1

# Swagger attempt
GET /swagger-ui.html HTTP/1.1
GET /v2/api-docs HTTP/1.1

# Rate limit bypass
GET /api/register HTTP/1.1
X-Forwarded-For: 1.2.3.4

# Method override
GET /api/admin/users HTTP/1.1
X-HTTP-Method-Override: GET
```
## 6. Nimalarga ahamiyat berish (checklist)
- [ ] Auth har bir endpointda (not only/docs)
- [ ] BOLA: har bir object ID
- [ ] Mass assignment, excessive data exposure
- [ ] CORS, rate limit, JWT misconfig
- [ ] Swagger/openapi leak
- [ ] Version endpoints `v2`, `internal`, `admin`
- [ ] Error verbose, stack traces
- [ ] Injection in all typed params (int/float/string)
- [ ] Token/API key in URL, logs, JS (client secret)
## 7. Himoya
- Consistent auth (JWT/OAuth per endpoint), least privilege
- Allowlist responses (DTO), never dump entities
- Rate limit per-user, IP, and API key (cache)
- Validate input strictly, disable debug/verbose in prod
- Introspection off, version semi (deprecate correctly), test matrix