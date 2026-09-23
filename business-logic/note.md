# Business Logic Vulnerabilities

## 1. Ta'rif
Business logic bug — dastur "logic" xatosi tufayli ishlamaydigan narsani ishlaydi. Bu sintaksis yoki sanitizatsiyadan ko'proq "qanday ishlaydi" o'ynash: narxni o'zgartirish, chegirma ishlatish, step'lar bo'yicha unutish, order stavkam, rate limit, race.

## 2. Qayerdan kelib chiqadi
- Narx / miqdor / valyuta field'lar client'tan keladi (`price=0`, `qty=-1`)
- Chjetning check: step 1 (login) o'tib step 2 (pay) ga to'g'ridan beriladigan referens
- Ma'lumot o'zgartirilganda state validatsiyasi yo'q
- Rate limit / bruteforce qarshilik yo'q
- Business invariants buziladi: saldo salbiy, bonus exploit, qaytarish davla
- Time-based: foydalanilgan coupon qayta ishlatish, purchase race
- Auth/logic: admin'likni API bilan, step'lar order'ni buzlash

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Price/quantity manipulation | `price=0`, `qty=-10` |
| Step/coupon abuse | promo qayta ishlatish, unlimited apply |
| Currency manipulation | EUR vs USD konvertatsiya round |
| Rate/staking abuse | cheklangan aksiya uchun limit aylanib o'tish |
| Race/TOCTOU | savdo va saldo o'rtasida |
| Invariant breaking | salbiy balance, double-withdraw |
| Auth bypass logic | `role=admin` field, step skip |
| Referral/fraud | ko'p akkaunt, referal bonus spam |

## 4. Metodologiya
1. Business flow'ni tushuning: buyurish → to'lov → tasdiqlash; step'lar, chegara.
2. Har bir request body qaysi field server'da qayta ishlanadi (client field'larini o'zgartiring):
   - `price`, `quantity`, `fee`, `discount`, `total`, `currency`, `is_paid`, `status`
   - `<input type="hidden" name="price">` — hidden input'lar!
3. Promo/coupon: bir necha marta ishlatish, boshqa user'ga ishlatish, negative qty.
4. States: qaytarish (refund) dan keyin saldo; order jarayonini hadbka (payment before validation).
5. Race: tez yuborish — standart race-condition (parallel), 'single-use' cheklovdan aylanib o'tish.
6. Acces control logic: har bir API tarmoqni client'ta tekshirish, serverda emas.
7. Limits: rate limit'ni raund aylanib o'tish — header'lar (`X-Forwarded-For`), proxy, `refresh`.
8. Timezone/date math va bonus (round-trip, `2026-02-30`).

## 5. Request misollari
```
# Narx o'zgartirish
POST /api/cart HTTP/1.1
Content-Type: application/json

{"product_id": 42, "price": 0.01, "qty": 1}

# Bonus exploit: mavjud bo'lmagan coupon
POST /api/apply-coupon HTTP/1.1
Content-Type: application/x-www-form-urlencoded

coupon=DOUBLE10&qty=-100

# Race: coupon ikki marta apply
# (parallel request ODDIDE - Burp Intruder / Turbo Intruder)

# Promo referal: ikki hisob / bank Ma'lumot
POST /api/referral HTTP/1.1
{"code": "REF_CODE"}          # variant: bitta code 2 marta ishlatish
```

## 6. Zaif code misollari
```python
# ZAIF — client qiymatiga ishonish
price = request.json["price"]
total = price * qty
# XAVFSIZ: narxni server-side catalog'dan olish

# ZAIF — tekshirish: to'lov keyin qilingan
if not user.paid:
    disable_order(order)
activate(order)                 # race: refunddan keyin cancel order
```

## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Har bir POST body field — qaysi biri client'tan olinib ishlatiladi
- [ ] Hidden inputs, disable/element field'lar (paid, admin, role, vip)
- [ ] Negative numbers, giant numbers, float precision
- [ ] Har bir promo/chegirma — qayta ishlatish, max apply, user'ga bog'liq
- [ ] State transition: qaysi holatdan qaysi holatga — skip step
- [ ] Time-of-check vs time-of-use (TOCTOU)
- [ ] Simultaneous transactions — race
- [ ] `int` vs `float` round — pennies (0.1+0.2 != 0.3)
- [ ] Idempotency key — qayta yuborish bo'yicha

## 8. Himoya
- Server-side narx/limit validatsiya; client'ga ishonma
- Idempotency, uper almashtirilmaydigan table lock/transaction izolyatsiya
- State machine + kuchli hokim analytic
- Barcha log'da audit, rate limit, anomaly detection