# ADR 0010 — MQTT como protocolo secundario, no como reemplazo de Modbus

## Estado
Aceptado (implementación funcional, decidida junto con el usuario al
arrancar el proyecto — ver también ADR 0001).

## Contexto
El enunciado pide MQTT como soporte "opcional" del agente de campo. Había
que decidir si implementarlo de verdad o solo documentarlo. Se optó por
implementarlo funcional: un broker Mosquitto real en `dmz-net` y el
backend suscripto de verdad, para mostrar un gateway que integra dos
protocolos de campo distintos — un escenario común en modernización de
oil & gas, donde conviven RTUs Modbus viejas con sensores IIoT que hablan
MQTT.

## Decisión
- `field-agent` publica **la misma lectura** que decide reportar (ya
  filtrada por reporte por excepción, ver ADR 0002) tanto por HTTP al
  backend como por MQTT al topic `oilfield/vacamuerta/well-07/telemetry`
  en Mosquitto. No es un stream separado de "todo el tráfico" — es el
  mismo evento por dos caminos.
- El backend se suscribe a ese topic (`app/ingestion/mqtt_subscriber.py`)
  y la lectura entra al mismo pipeline de ingesta que el POST HTTP (ver
  ADR 0006 sobre la cola Redis intermedia). La deduplicación por
  `UNIQUE(device_id, sequence)` (ADR 0005) es lo que permite que ambos
  caminos entreguen la misma lectura sin duplicarla en la base.
- **Mosquitto sin TLS ni autenticación de cliente** (`allow_anonymous
  true`): simplificación deliberada de laboratorio, contenida dentro de
  `dmz-net` (nunca expuesta a `it-net` ni al host). Documentado en
  `infra/mosquitto/mosquitto.conf`.

## Consecuencias
- Demuestra un gateway multi-protocolo real, no solo mencionado en el
  README.
- Si Mosquitto se cae, la ingesta HTTP directa sigue funcionando sin
  degradación — MQTT es estrictamente adicional, nunca el único camino.
- mTLS + ACLs por topic en Mosquitto quedan en el roadmap para un uso
  productivo real.
