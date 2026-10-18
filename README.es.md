<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licencia MIT (ver archivo LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="384">
</div>

# 🔒 OMEGA-WAF

**Firewall de aplicación mod_magnet (script Lua autónomo)**

> Elaborado por **kraynux** para **Omega-server**
> [kraynux.snake-mackarel.ts.net](https://kraynux.snake-mackarel.ts.net)

Página oficial: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#compatibilidad)

**Idiomas:**
[Français](README.md) · [English](README.en.md) · [Español](README.es.md) · [Русский](README.ru.md) · [中文](README.zh-CN.md)

---

**Omega-Waf** es un único script Lua (`omega-waf.lua`), sin dependencias externas, diseñado para ejecutarse en cada petición HTTP a través de un servidor web compatible con `mod_magnet` (lighttpd de forma nativa, u omega-serv mediante su puente de compatibilidad `magnet`). Inspecciona la ruta, la petición y, en POST, el cuerpo de la petición en busca de firmas de ataque conocidas (inyección SQL, XSS, inyección de comandos, recorrido de directorios), aplica limitación de tasa, gestiona una lista negra persistente con escalada progresiva, y registra cada decisión.

Sin instalación en el sentido clásico: un único archivo que copiar, referenciar desde la configuración del servidor, y listo.

## Índice

- [Presentación](#presentación)
- [Instalación](#instalación)
- [Configuración](#configuración)
- [Flujo de decisión](#flujo-de-decisión)
- [Firmas detectadas](#firmas-detectadas)
- [Rutas sensibles vigiladas](#rutas-sensibles-vigiladas)
- [Limitación de tasa y escalada](#limitación-de-tasa-y-escalada)
- [Persistencia](#persistencia)
- [Códigos de respuesta HTTP emitidos](#códigos-de-respuesta-http-emitidos)
- [Compatibilidad](#compatibilidad)
- [Personalización](#personalización)
- [Limitaciones conocidas y fuera de alcance](#limitaciones-conocidas-y-fuera-de-alcance)
- [Solución de problemas](#solución-de-problemas)
- [Licencia](#licencia)

---

## Presentación

### Objetivos

- Filtrar las peticiones maliciosas lo más cerca posible del servidor, antes de que lleguen a la aplicación servida.
- Seguir siendo un único archivo legible, sin dependencias externas ni paso de compilación.
- No tumbar nunca el servidor web: cualquier error interno del script se captura y se registra en lugar de propagarse.
- Limitar el coste de CPU/memoria por petición (cachés temporales, lectura del cuerpo por fragmentos, profundidad de decodificación acotada).
- Ofrecer una escalada progresiva (aviso y luego baneo) en lugar de un baneo seco ante el primer incidente menor.

### Qué hace Omega-Waf

- Bloquea direcciones IP ya baneadas (lista negra persistente en disco), con una excepción para las rutas explícitamente autorizadas (CSS de la página de error, etc.).
- Vigila un conjunto de rutas sensibles (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store`, etc.) y responde con una página de aviso HTML o con un baneo directo en caso de reincidencia.
- Detecta User-Agents conocidos como herramientas de escaneo o bots no deseados (sqlmap, nmap, nikto, dirbuster, crawlers genéricos...).
- Aplica una limitación de tasa global por IP, en todas las rutas.
- Aplica una política dedicada, más permisiva, en las rutas de un motor de búsqueda SearXNG (evita falsos positivos en búsquedas legítimas que contengan palabras que parezcan un ataque).
- Inspecciona la ruta y la query-string (SQLi, XSS, recorrido de directorios, byte NUL).
- Inspecciona el cuerpo de las peticiones POST (SQLi, XSS, inyección de comandos, recorrido de directorios) con un sistema de puntuación acumulativa.
- Gestiona una escalada progresiva: un comportamiento sospechoso aislado genera una alerta o un rechazo puntual; la reincidencia dentro de una ventana de tiempo provoca un baneo definitivo.
- Registra cada decisión (alertas, adiciones a la lista negra) en formato texto o JSON.
- Funciona tanto en lighttpd nativo (rutas codificadas de forma fija) como inyectado en omega-serv (rutas recibidas dinámicamente desde la configuración del servidor).

### Qué no hace el script

- No sustituye a un WAF dedicado de nivel empresarial (ModSecurity, Coraza, un WAF en la nube): las firmas son deliberadamente simples (coincidencia de patrones Lua, no un motor de reglas PCRE real).
- No analiza las cabeceras HTTP una por una más allá del User-Agent.
- No decodifica cuerpos multipart/form-data ni cuerpos comprimidos (gzip) — la inspección opera sobre el texto plano recibido.
- Nunca inspecciona el cuerpo de una petición que no sea POST.
- No realiza ninguna detección de inyección de comandos en la ruta de la URL (elección deliberada, ver [Limitaciones conocidas](#limitaciones-conocidas-y-fuera-de-alcance)).
- No ofrece ninguna interfaz de administración: la configuración se hace editando las variables al inicio del archivo, la lista negra y la whitelist mediante simples archivos de texto.
- No gestiona por sí mismo la rotación de sus registros.

### Advertencia de uso

Las firmas proporcionadas apuntan a familias de ataques comunes pero no constituyen una protección exhaustiva. Una puntuación de detección demasiado baja en un sitio con tráfico inusual (búsqueda técnica, documentación que contenga comandos shell, etc.) puede generar falsos positivos — ajusta `waf_score_threshold` y la whitelist de rutas en consecuencia. Prueba siempre un cambio de firma o de umbral en un entorno no crítico antes de desplegarlo en producción.

---

## Instalación

### Requisitos previos

- Un servidor web que exponga una interfaz compatible con `mod_magnet`:
  - **lighttpd** con el módulo `mod_magnet` activado (`server.modules += ("mod_magnet")`), o
  - **omega-serv** con el extra `magnet` instalado (dependencia `lupa` — ver más abajo y la guía de omega-serv, sección WAF Magnet).
- Un intérprete Lua embebido proporcionado por el anfitrión (lighttpd incluye su propio Lua; omega-serv incluye `lupa`, sin necesidad de intérprete `lua` del sistema en ningún caso — `lupa` se distribuye como rueda precompilada para todas las arquitecturas/versiones de Python habituales, sin compilación local ni cabeceras Lua del sistema necesarias).

#### Instalar el extra `magnet` de omega-serv

**Siempre desde el `.venv` ya creado por `omega-serv/install.sh`, nunca con un `pip` del sistema** — en algunas distribuciones (especialmente Arch Linux), el comando `pip` desnudo no existe (solo `python-pip` lo instala, y no siempre por defecto) o se niega a instalar nada fuera de un entorno virtual (`error: externally-managed-environment`, PEP 668):

```bash
cd /ruta/a/omega-serv

# Opcion A - venv ya activado en la shell actual
source .venv/bin/activate
pip install -e ".[magnet]"

# Opcion B - sin activar el venv (la forma mas portable: nunca depende
# de que 'pip' este presente en el PATH del sistema, ya que el propio
# python del venv siempre incluye el suyo)
.venv/bin/python -m pip install -e ".[magnet]"
```

Si `install.sh` todavía no se ha ejecutado (no hay `.venv/`), ejecútelo primero — crea el entorno virtual e instala allí omega-serv mismo; solo queda añadir el extra `magnet` (opcional) con uno de los dos comandos anteriores.

### Puesta en marcha

1. Copia `omega-waf.lua` a una ubicación legible por el servidor (p. ej. `secure/waf/scripts/omega-waf.lua` para una estructura estilo omega-serv, o cualquier ruta para lighttpd nativo).
2. Crea los directorios/archivos esperados:
   - **omega-serv**: nada que hacer, el servidor crea él mismo los directorios al arrancar (incluido `var/lib/waf/`, compartido con la cuenta del servicio).
   - **lighttpd nativo**: sustituye primero las 5 rutas por rutas absolutas (ver [Rutas en lighttpd nativo](#rutas-en-lighttpd-nativo)), luego crea los directorios y hazlos escribibles por la cuenta de lighttpd (`www-data` en Debian/Ubuntu, `http` en Arch, `lighttpd` en Fedora/Alpine):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. Referencia el script desde la configuración del servidor:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/ruta/a/omega-waf.lua" )
     ```
   - **omega-serv**: activa el WAF Magnet desde la pantalla TUI dedicada (Seguridad activa → WAF → Magnet), que apunta al script e inyecta automáticamente la tabla `omega_magnet_paths` (ver [Compatibilidad](#compatibilidad)).
4. Reinicia (no recargues — una recarga nunca vuelve a cargar un script Lua ya persistente en memoria) el servidor correspondiente.

---

## Configuración

Todas las opciones se ajustan modificando directamente las variables locales al inicio del archivo (sección `CONFIGURATION`, líneas ~24-70). Sin archivo de configuración separado.

| Variable | Función | Valor por defecto |
|---|---|---|
| `blocklist_file` | Archivo de texto de IPs baneadas definitivamente (una IP por línea, `#` para comentarios) — **escrito** por el script | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Directorio de reglas externas (contiene `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Registro de alertas (detecciones, rechazos, excesos) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Registro dedicado a las adiciones a la lista negra | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Archivo de persistencia de los contadores (rate limit / reincidencia), formato tabla Lua | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Activa la inspección del cuerpo de las peticiones POST | `true` |
| `enable_rate_limit` | Activa la limitación de tasa (global + SearXNG) | `true` |
| `enable_auto_block` | Permite la adición automática a la lista negra | `true` |
| `never_auto_block` | Redes **nunca** baneadas automáticamente (la petición sigue rechazada y registrada, solo se evita el baneo); bajo omega-serv, la lista inyectada por el servidor (`omega_never_auto_block`) siempre prevalece | host local, Tailscale, redes locales |
| `use_json_logs` | Cambia los registros de alerta a JSON (si no, texto legible) | `false` |
| `waf_score_threshold` | Puntuación mínima acumulada para rechazar una petición POST | `5` |
| `body_max_inspect` | Tamaño máximo inspeccionado del cuerpo (bytes) | `65536` (64 KB) |
| `chunk_size` | Tamaño de los fragmentos de lectura del cuerpo | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Cuota y ventana (s) de la limitación de tasa global por IP | `60` peticiones / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Cuota y ventana (s) de la limitación de tasa dedicada a las rutas SearXNG | `40` peticiones / `60` s |
| `searx_max_query_len` | Tamaño máximo aceptado de la query-string en las rutas SearXNG | `4096` caracteres |
| `suspicious_threshold` | Número de hits sospechosos antes del baneo definitivo | `3` |
| `suspicious_window` | Ventana deslizante (s) sobre la que se cuentan los hits sospechosos | `300` (5 min) |
| `blocklist_refresh_interval` | Intervalo (s) de relectura del archivo de lista negra | `5` |
| `whitelist_refresh_interval` | Intervalo (s) de relectura de la whitelist de rutas | `15` |

Bajo omega-serv, las 5 primeras rutas (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) se sustituyen automáticamente por las configuradas en la pantalla WAF Magnet (tabla global `omega_magnet_paths` inyectada previamente) — los valores fijos de arriba solo sirven de respaldo si esa tabla está ausente.

> ℹ️ **Cambio del 2026-10-18**: `blocklist_file` pasa de `secure/waf/magnet-blocklist.txt` a `var/lib/waf/magnet-blocklist.txt`. El script **escribe** en este archivo en cada baneo, y el servicio systemd de omega-serv solo puede escribir bajo `var/`: con la ubicación antigua, nunca se registraba ningún baneo automático, sin ningún mensaje. omega-serv recupera el archivo antiguo una sola vez al arrancar (el antiguo se conserva como copia de seguridad); una ruta personalizada en la pantalla WAF Magnet nunca se modifica.

### Rutas en lighttpd nativo

> ⚠️ **Estos 5 valores por defecto están calcados de la propia convención de directorios de omega-serv** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, rutas relativas resueltas respecto a un directorio de trabajo de tipo proyecto) — no son neutrales. En **lighttpd nativo**, donde no existe una convención de directorios equivalente, estas rutas relativas se resuelven respecto al directorio de trabajo del proceso lighttpd en tiempo de ejecución (a menudo `/`), raramente predecible. **Sustituye estas 5 líneas por rutas absolutas** adaptadas a tu instalación, por ejemplo:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Antes de reiniciar, crea estos directorios (comandos en el paso 2 de [Puesta en marcha](#puesta-en-marcha)) respetando estos permisos:
>
> | Archivo | Acceso necesario para la cuenta de lighttpd |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **escritura** (el archivo y su directorio) |
> | `waf_rules_dir` (y `whitelist_paths.txt`) | solo lectura |
>
> Un fallo de escritura es **silencioso** — el script nunca rompe la petición —: un baneo que no aparece en `blocklist_file` es el único síntoma. Compruébalo con `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` (adapta la cuenta). Si la unidad systemd de lighttpd restringe las escrituras (`ProtectSystem=strict`, `ReadWritePaths=`), añade estos directorios.

---

## Flujo de decisión

El script se ejecuta íntegramente dentro de un `pcall` global: cualquier error Lua imprevisto se registra y responde `500`, sin hacer caer nunca el proceso del servidor. Las etapas se ejecutan en este orden, cada una pudiendo detener el procesamiento con una respuesta final:

1. **Lista negra** — si la IP está baneada, la petición se rechaza (`451`), salvo que la ruta solicitada esté explícitamente en la whitelist (sirve por ejemplo la propia página de error).
2. **Rutas sensibles** — si la ruta coincide con un patrón vigilado (`.git`, `.env`, `wp-admin`...), se devuelve una página de aviso HTML (`403`) la primera vez; la reincidencia dentro de la ventana de sospecha provoca un baneo (`451`).
3. **User-Agent escáner** — si el User-Agent coincide con una herramienta de escaneo/bot conocida, `403` la primera vez, `451` en caso de reincidencia.
4. **Límite de tasa global** — si se supera la cuota global por IP, `429`.
5. **Política SearXNG** — para las rutas de búsqueda reconocidas: límite de tasa dedicado (`429`), control del tamaño de la query-string (`414`), rechazo de bytes NUL (`451`), y luego salida inmediata (`0`, continúa) para que una búsqueda legítima nunca pase por las verificaciones agresivas siguientes.
6. **Protección de la ruta** — búsqueda de SQLi y XSS en la ruta decodificada (`451` si se detecta); búsqueda de recorrido de directorios (`451`). Deliberadamente **sin** detección de inyección de comandos en esta etapa (ver [Limitaciones conocidas](#limitaciones-conocidas-y-fuera-de-alcance)).
7. **Protección ligera de la query-string** (fuera de SearXNG) — recorrido de directorios y byte NUL (`451`).
8. **Inspección del cuerpo POST** — solo si `enable_body_inspection` está activo y el método es `POST`: un cuerpo demasiado voluminoso se rechaza directamente (`413`); si no, una puntuación acumulada sobre SQLi/XSS/CMDi/recorrido, rechazo (`451`) si la puntuación alcanza `waf_score_threshold`, con baneo inmediato si la puntuación alcanza 8, si no un simple hit sospechoso.
9. **Sin bloqueo** — la petición continúa con normalidad (`0`).

---

## Firmas detectadas

Cuatro familias de patrones (`patterns`), cada entrada siendo una cadena (sensible a mayúsculas) o una tabla `{ pat = "patrón", ci = true }` (insensible a mayúsculas):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, inyecciones de comandos SQL vía punto y coma (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: etiquetas `<script>`, `<iframe>`, atributos `on*=` (manejadores de eventos), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: encadenamiento de comandos shell (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, variantes con `|`), sustitución de comandos (`` `...` ``, `$(...)`), etiqueta PHP (`<?php`) y funciones de ejecución (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

Más una lista dedicada de **User-Agents de escáneres/bots** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, crawlers de referenciación (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`), y patrones genéricos (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

La decodificación de URL se aplica de forma recursiva (hasta 5 niveles en la ruta, 2 niveles en la query-string) antes de la inspección, para atrapar payloads codificados doblemente.

---

## Rutas sensibles vigiladas

Patrones verificados sobre la ruta decodificada: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Para cada coincidencia, el script intenta determinar si realmente existe un recurso físico en esa ruta (vía `lighty.stat`, con respaldo en `lighty.c.stat` por compatibilidad) y lo indica en la página de aviso y en el registro — útil para distinguir un escaneo a ciegas de una ruta que realmente existe en el servidor.

---

## Limitación de tasa y escalada

Dos limitadores de tasa independientes, basados en contadores en memoria persistidos en disco (ventana deslizante simple, no una ventana deslizante precisa al segundo):

- **Global**: clave `g:<ip>`, `global_rate_max_req` peticiones por `global_rate_window` segundos, en todas las rutas.
- **SearXNG**: clave `sx:<ip>`, `searx_rate_max_req` peticiones por `searx_rate_window` segundos, solo en las rutas de búsqueda reconocidas.

La escalada progresiva (`record_suspicious_hit`) es un mecanismo separado de la limitación de tasa: cada detección no crítica (ruta sensible no reincidente, User-Agent escáner, hit menor en el cuerpo POST) incrementa un contador por IP (clave `susp:<ip>`) sobre una ventana de `suspicious_window` segundos. Una vez alcanzados `suspicious_threshold` hits en esa ventana, la IP se añade a la lista negra definitiva — evita un baneo seco por un único falso positivo, a la vez que banea la reincidencia.

**Redes nunca baneadas**: una dirección de `never_auto_block` (host local, Tailscale `100.64.0.0/10`, redes locales IPv4/IPv6, incluida la forma IPv4 mapeada en IPv6) nunca provoca escalada ni adición a la lista negra; un baneo evitado queda registrado con una línea `BAN EVITE` en `blocklist_log` (nunca `AJOUT BLOCKLIST`, que lee omega-jail). Comparación CIDR en Lua portable, de 5.1 a 5.5 y LuaJIT.

---

## Persistencia

| Archivo | Contenido | Formato |
|---|---|---|
| `blocklist_file` | Una IP baneada por línea, precedida de un comentario con fecha (`# fecha - razón`) | Texto plano |
| `waf_rules_dir/whitelist_paths.txt` | Un patrón Lua (`string.match`) por línea, rutas siempre permitidas incluso para una IP baneada | Texto plano |
| `rate_store_file` | Contadores de limitación de tasa y reincidencia (`{ count = N, first = timestamp }` por clave) | Tabla Lua serializada (`return { ... }`, releída vía `load()`) |
| `alerts_log` | Una línea por detección/rechazo | Texto legible, o JSON si `use_json_logs = true` |
| `blocklist_log` | Una línea por adición a la lista negra | Texto legible |

El archivo de limitación de tasa nunca se reescribe en cada petición (solo como máximo cada 300 segundos, durante la limpieza periódica) para evitar un coste de E/S desproporcionado. La lista negra y la whitelist se cachean en memoria y se releen como máximo cada `blocklist_refresh_interval`/`whitelist_refresh_interval` segundos — una adición provocada por el propio script actualiza la caché de inmediato, sin esperar ese plazo.

---

## Códigos de respuesta HTTP emitidos

| Código | Significado | Disparador |
|---|---|---|
| `0` | Continúa el procesamiento normal (convención `mod_magnet`) | Sin detección |
| `403` | Acceso denegado, primer aviso | Ruta sensible detectada (primera vez), User-Agent escáner (primera vez) |
| `413` | Entidad demasiado grande | Cuerpo POST que supera `body_max_inspect` |
| `414` | URI demasiado larga | Query-string SearXNG que supera `searx_max_query_len` |
| `429` | Demasiadas peticiones | Límite de tasa global o SearXNG superado |
| `451` | No disponible por razones legales (usado aquí como código de baneo) | IP ya baneada, reincidencia tras escalada, SQLi/XSS/recorrido detectado en la ruta o la query-string, byte NUL, puntuación POST ≥ umbral |
| `500` | Error interno | Excepción Lua imprevista (capturada por el `pcall` global) |

---

## Compatibilidad

El script funciona en **dos contextos**, sin ninguna modificación:

- **lighttpd nativo**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` son las API nativas del módulo `mod_magnet`. Las 5 rutas de configuración usan directamente sus valores por defecto fijos.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): un puente Python-Lua (`lupa`) reproduce fielmente el mismo contrato `lighty.*`, con un runtime Lua **persistente** entre peticiones (las cachés temporales y la limitación de tasa sobreviven así correctamente de una petición a otra, igual que en lighttpd). La tabla global `omega_magnet_paths` se inyecta antes de la ejecución desde la configuración de la pantalla WAF Magnet, para que las 5 rutas apunten a las ubicaciones elegidas en la interfaz en lugar de a los valores fijos.

Sin ninguna divergencia de lógica de negocio entre los dos contextos — solo cambia el origen de las 5 rutas de configuración.

### Lo que NO es compatible tal cual

El script depende enteramente del contrato `lighty.*` (API nativa de `mod_magnet`) — no funciona tal cual en otro servidor, aunque este ofrezca alguna forma de scripting Lua:

- **Apache** (`mod_lua`): API diferente, basada en un objeto de petición `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — ninguna llamada `lighty.*` funcionaría.
- **nginx** (`lua-nginx-module` / OpenResty): API diferente (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), con un modelo de fases de ejecución (`access_by_lua`, `content_by_lua`...) distinto del simple retorno de código de `mod_magnet`.
- **Traefik**: ningún scripting Lua nativo (escrito en Go) — solo un plugin Go (Yaegi) permitiría ejecutar lógica personalizada, nunca Lua directamente.
- **Caddy**: ningún módulo Lua oficial — requeriría un plugin de terceros compilado vía `xcaddy`, nada equivalente disponible de forma nativa.

Hacer este script compatible con uno de estos servidores requeriría escribir un puente de compatibilidad dedicado que traduzca su API nativa a `lighty.*` (exactamente lo que hace `lua_waf_adapter.py` para omega-serv) — un trabajo de ingeniería en sí mismo, no un simple ajuste de configuración.

---

## Personalización

- **Adaptar a otra aplicación distinta de SearXNG**: modificar `is_searxng_path()` (lista de rutas) y, si es necesario, eliminar por completo la sección 3 del flujo (política dedicada) si la aplicación protegida no necesita un tratamiento diferenciado de las búsquedas.
- **Añadir rutas siempre permitidas**: editar `waf_rules_dir/whitelist_paths.txt` (un patrón Lua por línea), o añadir una excepción fija en `is_whitelisted_path()`.
- **Ajustar la agresividad**: `waf_score_threshold` (umbral de rechazo POST), `suspicious_threshold`/`suspicious_window` (velocidad de escalada hacia un baneo definitivo), `global_rate_max_req`/`global_rate_window` (límite de tasa general).
- **Cambiar los registros a JSON** (ingesta SIEM/log shipper): `use_json_logs = true`.
- **Desactivar un subsistema sin tocar el resto**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` son independientes.

---

## Limitaciones conocidas y fuera de alcance

- **Sin detección de CMDi en la ruta de la URL**: elección deliberada para evitar falsos positivos en rutas que contengan `.php` (que se confundiría con `<?php` si la detección de CMDi incluyera la etiqueta PHP en esta superficie). La detección de CMDi sigue activa en el cuerpo POST.
- **Firmas mediante coincidencia de patrones Lua, no un motor PCRE real**: las variantes de ataque no cubiertas por los patrones listados no se detectarán — esto no sustituye a un WAF basado en conjuntos de reglas exhaustivos (tipo CRS ModSecurity).
- **Sin análisis de multipart/form-data**: un payload malicioso encapsulado en un campo de formulario multipart puede escapar a la inspección si los patrones no coinciden con el flujo bruto no decodificado.
- **Sin descompresión gzip del cuerpo**: un cuerpo comprimido no se inspecciona en claro.
- **Limitación de tasa aproximada**: ventana deslizante simplificada (reinicio completo al superar la ventana), no un algoritmo de ventana deslizante precisa (sliding log/token bucket).
- **Sin interfaz de supervisión**: toda observación pasa por la lectura directa de los archivos de registro y de la lista negra.

---

## Solución de problemas

- **El script nunca parece ejecutarse**: verifica que `mod_magnet` esté realmente cargado (lighttpd) o que el extra `magnet` esté instalado y el WAF Magnet activado (omega-serv); una recarga de configuración nunca basta para cargar un script Lua modificado, se necesita un reinicio completo en ambos casos.
- **Los contadores de rate limit parecen reiniciarse en cada petición**: síntoma de un runtime Lua no persistente entre peticiones (el script parte de un `rate_store` vacío en cada ejecución) — bajo omega-serv, verifica que la pantalla WAF Magnet indique un runtime persistente activo.
- **Una IP legítima acaba baneada**: consulta `blocklist_log` y `alerts_log` para identificar la regla disparada, ajusta el patrón o el umbral correspondiente, y luego elimina la línea de la IP en `blocklist_file` (la caché en memoria se actualiza en la siguiente recarga, como máximo `blocklist_refresh_interval` segundos después de modificar el archivo).
- **Las búsquedas legítimas de SearXNG son bloqueadas**: verifica `is_searxng_path()` (ruta no reconocida) antes de ajustar los umbrales de limitación de tasa dedicados.
- **`pip install -e ".[magnet]"` falla con `pip: orden no encontrada` o `externally-managed-environment`**: el comando se escribió fuera del `.venv` de omega-serv (algunas distribuciones, especialmente Arch Linux, no exponen ningún `pip` del sistema directamente utilizable). Use `.venv/bin/python -m pip install -e ".[magnet]"` desde el directorio de omega-serv — funciona sin depender de que `pip` esté en el `PATH` del sistema (ver [Instalar el extra `magnet` de omega-serv](#instalar-el-extra-magnet-de-omega-serv)).

---

## Licencia

Omega-Waf se distribuye bajo licencia **MIT**. Consulta el archivo [`LICENSE`](LICENSE) para el texto completo.

---

> **Omega-Waf — Un script, una línea de defensa.**
>
> Un firewall de aplicación mod_magnet en un único archivo Lua, sin dependencias, compatible con lighttpd y omega-serv.
