# Mass Assignment (Auto-Binding / Parameter Tampering)

## 1. Ta'rif
Mass (auto) assignment — server client yuborgan barcha field'larni avtomatik modelga bog'laydi (binding) → foydalanuvchi o'zi o'zgartira olmaydigan field'ni (role, isAdmin, balance, status, price) o'zgartirishi mumkin. REST/ORM frameworklarda ko'p.

## 2. Qayerdan kelib chiqadi
- Frameworks: Rails `params[:user]`, Laravel `$request->all()`, Django `ModelForm` (without exclude field), Spring `@ModelAttribute`, Node `Object.assign`, PHP `extract($_POST)`
- `PUT/PATCH /api/user` — request body'dagi barcha field model'ga o'tadi
- Referenced nested objects (`{"user_id": ...}`) 
- API version old field mapping

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Sencitive pasw changers | `{"role":"admin"}` |
| Balance/credit | `{"balance": 9999}` |
| Auth/consent | `{"twoFAEnabled": false}` |
| Meta fields | `{"isVerified": true, "id": 2}` |
| Nested objects | `{"owner": {"id": 1}}` |
| Extra params in POST | login'da `isAdmin=1` |

## 4. Metodologiya
1. Auth'dan keyin profile update endpoint: `PUT/POST /api/user`, `/api/me`, `PATCH /api/profile` → body JSON.
2. Qaysi field server'da qo'llanadi? (`name, email, phone`).
3. Extra field'larni qo'shib yuborish:
   - `"role":"admin"`, `"isAdmin":true`, `"is_admin":1`
   - `"verified":true`, `"email_verified":1`, `"status":"active"`
   - `"credit":100000`, `"balance": 9999`, `"plan":"enterprise"`
   - `"id": 3`, `"user_id": 1` (boshqa user'ga yozish)
4. Response'ni tekshiring: role o'zgarmadimi / response'da admin true qaytdimi?
5. `x-www-form-urlencoded` da ham sinang (form binding).
6. Nested/renaming: `"role":"admin"` → kebab/camel variantlar.

## 5. Request misollari
```
PATCH /api/users/me HTTP/1.1
Host: target.com
Authorization: Bearer <token>
Content-Type: application/json

{"name": "test", "role": "admin"}

PUT /v2/profile HTTP/1.1
Content-Type: application/json
{"user": {"id": 1, "isSuperAdmin": true}}

POST /register HTTP/1.1
Content-Type: application/x-www-form-urlencoded
username=a&password=b&isAdmin=1
```
## 6. Zaif code misollari
```python
# Django ZAIF (ModelForm default)
# forms.ModelForm  -> form.save() user mengjidan pays -> is_superuser bind bo'ladi (agar allowed bo'lsa)

# Rails ZAIF
user.update(params[:user).merge(current_user))   # params mass

# ZAIF (Node)
user = Object.assign(user, req.body)   # req.body {premium:true} → o'zgaradi

# XAVFSIZ (Django)
class ProfileForm(forms.ModelForm):
    class Meta:
        model = User
        fields = ['name', 'email']   # faqat ro'yxat!
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Har bir update endpoint — body'ga extra field qo'shib sinash
- [ ] Mutations: `role, isAdmin, verified, paid, balance, credits, premium, status, level, plan, org, id`
- [ ] Boshqa user id (user_id, owner_id) — mass + IDOR combo
- [ ] Nested JSON `{"user":{"role":"admin"}}`
- [ ] Form (urlencoded) + JSON both — binding har xil
- [ ] GraphQL mutations — input type'da hidden field
- [ ] Multi-step forms — step 2 bo'lsa step 1 field'i repo keliggi
- [ ] Response'da mindiq field'lar — refle
## 8. Himoya
- Allowlist (fields include) — serfaqat kerakli field'lar
- DTO (Data Transfer Object) compared to entity — map explicitly
- `__proto__`, not expose admin field names
- Validation that rejects unknown keys