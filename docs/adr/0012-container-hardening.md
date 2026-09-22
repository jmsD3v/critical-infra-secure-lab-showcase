# ADR 0012 — Endurecimiento de contenedores: cap_drop por defecto, sumar solo lo indispensable

## Estado
Aceptado (verificado con `docker compose up --build` real, no solo revisado en papel)

## Contexto
`docker-compose.yml` ya traía `no-new-privileges` y `cap_drop: ALL` en
`plc-simulator` y `field-agent` desde el scaffold inicial (ver ADR 0004).
Faltaba extenderlo al resto de los servicios. El plan inicial era
`cap_drop: ALL` en los ocho servicios sin excepción — la corrida real
mostró que ese plan era demasiado optimista.

## Decisión final (post-verificación)

- **`cap_drop: ALL` en `plc-simulator`, `field-agent` y `backend`** — los
  tres son código propio, con `USER` no-root definido en el Dockerfile
  desde build time, sin ningún patrón de "arrancar como root y bajar
  privilegios" en runtime. Estos tres además llevan `read_only: true` +
  `tmpfs: [/tmp]`.
  - `plc-simulator` necesita además `cap_add: [NET_BIND_SERVICE]` para
    bindear el puerto Modbus estándar (502, privilegiado) como usuario
    no-root.
- **`postgres`, `redis`, `mosquitto`, `dashboard` (nginx) SIN `cap_drop`**:
  se probó con `cap_drop: ALL` en los cuatro y los cuatro rompieron su
  arranque, cada uno por el mismo motivo de fondo — son imágenes
  oficiales cuyo entrypoint arranca como root, hace `chown`/`chmod` de
  sus directorios de trabajo, y recién ahí baja privilegios al usuario
  de servicio (patrón estándar en imágenes Docker oficiales). Sin
  `CAP_CHOWN`/`CAP_SETUID`/`CAP_SETGID` ninguna de las dos partes es
  posible:
  - `postgres`: `chmod: /var/lib/postgresql/data: Operation not permitted`
    → `error: failed switching to 'postgres': operation not permitted`.
  - `redis`: `setpriv: setresuid failed: Operation not permitted`.
  - `mosquitto`: `chown: /mosquitto/data: Operation not permitted` →
    `Error setting groups whilst dropping privileges`.
  - `dashboard`/nginx: `chown("/var/cache/nginx/client_temp", 101) failed
    (1: Operation not permitted)` — nginx master arranca como root para
    poder crear ese directorio con el owner correcto antes de forkear
    workers no-root.
  Los cuatro quedaron con `security_opt: no-new-privileges:true`
  solamente — el endurecimiento que sí se pudo verificar sin romper nada.
- **`dashboard` escucha en 8080 (no privilegiado) en vez de 80** dentro
  del contenedor: evita necesitar `NET_BIND_SERVICE` de vuelta para un
  problema distinto al de cap_drop (bindear <1024) — no hay razón de
  realismo de dominio para insistir con el puerto 80 en un servidor de
  estáticos.

## Consecuencias
- Verificado de punta a punta: los ocho servicios levantan y pasan sus
  healthchecks (`docker compose up --build`), el agente reporta
  telemetría real, el dashboard la muestra en vivo, el comando de
  actuador Start/Stop funciona a través de todo el camino (dashboard →
  backend → field-agent → Modbus write → PLC simulado), y los dos
  escenarios de incidente corren limpios contra el stack real.
- Límites de recursos (`mem_limit`/`cpus`) para los cuatro servicios de
  terceros quedan en el roadmap.
- Lección concreta (vale para cualquier lector, no solo para este repo):
  "cap_drop: ALL en todo" suena bien en un README de seguridad pero rompe
  cualquier imagen oficial con el patrón root→chown→setuid, que es la
  mayoría de las imágenes de infraestructura (bases de datos, colas,
  proxies). El hardening real es *auditar cada imagen*, no aplicar la
  misma regla a ciegas en los ocho servicios.
