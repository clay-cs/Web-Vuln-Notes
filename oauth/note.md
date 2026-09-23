# OAuth 2.0 / SSO Abuse

## 1. Ta'rif
OAuth — uchinchi tomon (Provider) orqali login (Google/Facebook/GitHub) yoki API'ga ruxsat. Zaifliklar: `redirect_uri` validation, state missing, token in redirect/LocalStorage, authorization code leak, account binding (email), open redirect'lar chain.

## 2. Qayerdan kelib chiqadi
- `redirect_uri` o'zgartirilsa — code/access token attacker'ga ketadi
- `state` yo'q / not bound — CSRF (login CSRF)
- `scope` manipulyatsiya
- Client secret leak
- Code exchange validation yo'q
- Account binding by email (boshqa OAuth account — ATO)
- Open redirect endpoint — chain OAuth bilan
- JWT `sub` vs provider user confusion
- ID token ile flow (implicit) — token in URL

## 3. Turlari
| Tur | Misol |
|-----|-------|
| redirect_uri manipulation | `https://target/cb?x=https://evil` |
| Missing state | CSRF / login CSRF |
| Client secret in JS/URL | exfil |
| Account binding flaw | attacker email ↔ victim OAuth |
| Token in URL | referer/оgran leakage |
| Scope escalation | `scope=admin` |
| Wildcard/loopback | `redirect_uri=*`, `localhost` |
| Code reuse | code single-use eslanma |

## 4. Metodologiya
1. OAuth flow'ni kuzat: login button → authorize → redirect_uri → callback?code= → exchange → token.
2. `redirect_uri` test:
   - `redirect_uri=https://target.com/callback`
   - `redirect_uri=https://target.com/callback@evil.com`
   - `redirect_uri=https://target.com.evil.com/callback`
   - `redirect_uri=https://evil.com/target.com`
   - `redirect_uri=https://target.com/callback%2f%2fevil.com`
   - `redirect_uri=https://target.com/callback/?x=...` ga fragment; path suffix
   - Subdomain (`redirect_uri=evil.target.com`)
   - `//evil.com`, `/\evil.com`, `https://evil.com`
   - Open redirect'lar ichida: `https://target.com/r?url=https://evil.com`
   - List check: exact vs prefix vs regex
3. `state` yo'q bo'lsa — attack:
   - Victim'ni https://target.com/login/oauth?provider=google&redirect_uri=... 
   - Javob... Login CSRF.
4. Code/access token:
   - URL fragment'da bo'lsa (`#access_token=...`) — referer leak / history
   - LocalStorage'da bo'lsa — XSS
5. Account binding:
   - Victim akkauntini attacker Google'ga bind qilish — `account_id` pre-login ga
   - `email` matching — attacker email orqali provider; victim email'ni o'xshash
6. Scope — token'da ko'proq scope; admin API check
7. Open redirect — OAuth bilan o'ralash ganı.

## 5. PoC / request
```
# redirect_uri tampering
GET /oauth/authorize?client_id=X&response_type=code&redirect_uri=https://evil.com/cb HTTP/1.1

# state missing
/dnbt — n shart

# PoC CSRF (state yo'q):
<a href="https://target.com/login/oauth?client_id=X&redirect_uri=https://target.com/cb">Login</a>
# — username victor'ning account'iga attacker Google bind bo'ladi (agar bunding binding)

# callback open redirect chain
GET /cb?code=AUTHCODE&redirect_uri=https://target.com/open?u=https://evil.com HTTP/1.1
```
## 6. Zaif code misollari
```javascript
// ZAIF — redirect_uri tekshirilmaydi
return res.redirect(`https://provider/oauth?redirect_uri=${req.query.redirect_uri}`);

// ZAIF — state yo'q
app.get('/cb', async (req,res) => {
  code = req.query.code;
  token = await exchange(code);        // state o'rnatilmagan
});

// ZAIF — binding by email/username only
if (userByEmail(googleEmail)) bind(userByEmail(googleEmail), googleId);
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] `redirect_uri` exact match validationmi
- [ ] `state` ishlatiladimi va session'ga bog'liqmi
- [ ] Open redirect bilan OAuth yo'llash
- [ ] Code single-use, tume, bound to client
- [ ] Tokenlar saqlangani / URL /JS
- [ ] Scope — amalda scope kerak; permission escalation
- [ ] Account linking — email match ishonchsiz
- [ ] Session after OAuth — host cookie vs provider
- [ ] `response_type=token` implicit — fragment leak
## 8. Himoya
- Strict redirect_uri allowlist (exact match)
- `state` (random, session-bound)
- PKCE (code_challenge) — public clients
- Account linking confirm (verification interaction)
- Tokens: HttpOnly, server storage, short-lived refresh