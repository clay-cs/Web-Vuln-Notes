# IDOR / BOLA (Broken Object Level Authorization)

## 1. Ta'rif
IDOR (Insecure Direct Object Reference) — foydalanuvchi boshqa foydalanuvchining ob'ektiga identifikatorni (ID, email, phone...) o'zgartirib erishganda, ammo server ob'ektning egasini tekshirmaydi. API'da BOLA deyiladi.

## 2. Qayerdan kelib chiqadi
- REST API: `GET /api/user/123`, `PUT /api/order/456`
- Parametr: `id`, `uid`, `order_id`, `invoice`, `file_id`, `account_num`, `email`
- Cookie + ID ishlatilmaydi, server faqat ID ga qarab qaytaradi
- UUID bo'lsa ham enum dan qochib bo'ladi (ba'zan `uuid2` predictable)
- Referral/report/reset/flintash orqali kimningdir ob'ektini yangilash

## 3. Turlari
| Tur | Xulq |
|-----|--------|
| Read IDOR | o'qish chaqiruvchi bo'lmagan ma'lumo (PII, card, shaxsiy) |
| Write IDOR | o'zgartirish/o'chirish (delete), reference ob'ektni o'zgartirish |
| Horizontal | bir xil role, boshqa user data |
| Vertical | past role, yuqori role data (boshqa/ardi) |

## 4. Metodologiya
1. Auth'dan keyin user zaif ob'ekt bilan ishlaydigan endpointlarni map qiling (swagger, devtools).
2. Har bir request'da `id` o'zgartiring: `me` o'rniga `123`, `1`, `0`, `-1`, boshqa user ID.
3. Ikki user yarating (A va B) — A token bilan B'ning ID'ga kirish sinab ko'ring.
4. Response'da `200` + data bo'lsa — IDOR confirmed.
5. Write? `PUT /api/users/me/...` boshqa ID. DELETE ham.
6. UUID predictable? (enum, timestamp-based, sequential).
7. Update foydalanuvchining `email`/`phone`/`balance` kabi field'ni o'zgartirish (mass assignment bilan aralash).
8. References: `{"order": {"user_id": 999}}` mass assign.

## 5. Request misollari
```
# A token bilan B id
GET /api/invoices/10002 HTTP/1.1
Host: target.com
Authorization: Bearer <A_TOKEN>

GET /api/user?id=10001 HTTP/1.1
Cookie: session=<A>

PUT /api/profile/999 HTTP/1.1
Authorization: Bearer <A>
Content-Type: application/json

{"email": "attacker@evil.com"}        # boshqa user profilini egallash

DELETE /api/documents/777 HTTP/1.1
Authorization: Bearer <A>
```

## 6. Zaif code misollari
```python
# ZAIF (Flask)
@app.route("/api/user/<int:uid>")
def get_user(uid):
    # token user tekshirilmaydi!
    return User.query.get(uid).to_json()

# XAVFSIZ
u = User.query.get_or_404(uid)
if u.id != current_user.id:
    abort(403)
```
```java
// ZAIF
@GetMapping("/api/orders/{id}")
public Order get(@PathVariable Long id) {
    return repos.findById(id).get();
}
```

## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Endpoint'da ID + role tekshiruv bormi — token'dagi user vs ob'ekt owner
- [ ] Horizontal va vertical tekshiring
- [ ] `me` magictaniga kirish, sonli ID, UUID v1/v4, base64 ID'lar (decoding)
- [ ] Nested resource: `/api/group/1/members` — group'a membership tekshiriladimi
- [ ] PUT/PATCH mass assignment — `role`, `is_admin`, `balance`
- [ ] UUID bo'lsa ham predictble pattern (timestamp)
- [ ] Blind (200 vs 403) — hattoki faqat foydalanuvchi bor/yo'qligi response farqi
- [ ] Referral/points/balance'i oshirish write'id

## 8. Himoya
- Har bir ob'ektda owner check serverda (never client)
- Row-level security, authorization middleware
- Object ref'lariga server-generated random/UUID (non-enumerable)
- `me` konvensiyasi + backend owner resolve