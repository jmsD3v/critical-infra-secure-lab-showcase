# ADR 0009 — Token de dispositivo para ingesta, y comandos de actuador en modelo *pull*

## Estado
Aceptado

## Contexto
El enunciado no pide explícitamente autenticar al agente de campo frente
al backend (solo pide JWT para "API y dashboard", es decir, usuarios
humanos). Dejar `POST /api/v1/telemetry` completamente abierto en la DMZ
igual se sintió como un hueco evidente para un laboratorio que se
presenta como "arquitectura de seguridad" — cualquier proceso en
`dmz-net` podría inyectar telemetría falsa sin que quede registrado como
sospechoso.

También había que decidir cómo llegan los comandos de actuador
(start/stop bomba) desde el backend (zona IT) hasta el PLC simulado
(zona OT), dado que `ot-net` es `internal: true` y nada de IT puede
alcanzar `plc-simulator` directamente (ver docs/adr/0004).

## Decisión
- **`X-Device-Token`**: un secreto compartido simple (`DEVICE_INGEST_TOKEN`
  en `.env`), distinto del JWT de usuarios, que `field-agent` manda en
  cada request al backend (`POST /telemetry`, `GET .../commands/pending`,
  `POST .../ack`). El backend lo valida con `verify_device_token`
  (`app/api/deps.py`) y, si falta o no coincide, responde 401 y registra
  un `SecurityEvent` tipo `unauthorized_device_ingest`.
- **Modelo *pull* para comandos**: `field-agent` consulta
  `GET /api/v1/equipment/{device_id}/commands/pending` en cada ciclo y,
  si hay un comando, lo aplica por escritura Modbus y confirma con
  `POST .../ack`. El backend **nunca** abre una conexión saliente hacia
  `ot-net` — ni siquiera existe una ruta de red posible para hacerlo (ver
  ADR 0004). Es el mismo patrón que usan gateways OT reales para evitar
  conexiones entrantes hacia la red de control.

## Consecuencias
- Es un secreto compartido estático, no autenticación fuerte por
  dispositivo (no hay mTLS ni rotación) — reconocido como limitación,
  con mTLS por dispositivo en el roadmap. Igual sube el piso respecto a
  no tener nada: un llamado sin el header queda bloqueado y auditado.
- El modelo pull agrega latencia a los comandos (hasta un ciclo de poll
  del agente, `MODBUS_POLL_INTERVAL_SECONDS`) a cambio de no requerir
  ningún puerto de escucha del lado OT — trade-off intencional a favor de
  seguridad sobre latencia, razonable para control de bombeo (no es un
  interlock de seguridad de milisegundos).
