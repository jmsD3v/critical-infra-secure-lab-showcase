# ADR 0006 — Redis como cola de ingesta MQTT y como limitador de intentos de login

## Estado
Aceptado

## Contexto
El enunciado menciona Redis como pieza "opcional... para colas". Había que
decidir si integrarlo de verdad con un rol concreto, o dejarlo listado sin
uso real (lo segundo hubiera sido cargo-cult: meter un servicio en
`docker-compose.yml` solo porque el enunciado lo nombra).

Dos problemas del backend encajan de forma genuina con Redis:

1. **Puente entre el hilo MQTT y el event loop de FastAPI**: `paho-mqtt`
   no es asyncio-nativo — corre su propio hilo. Llamar directo a
   SQLAlchemy (sync) o al `ConnectionManager` de WebSocket (async) desde
   ese hilo es frágil (cruzar threads con asyncio a mano, sin una cola de
   por medio, es una fuente clásica de race conditions).
2. **Bloqueo por fuerza bruta en `/api/v1/auth/login`**: necesita un
   contador con expiración (`INCR` + `EXPIRE`) compartido entre requests
   — exactamente el caso de uso de un store en memoria con TTL nativo.

## Decisión
- **Cola de ingesta MQTT** (`app/ingestion/mqtt_subscriber.py` +
  `mqtt_consumer.py`): el hilo de `paho-mqtt` hace `LPUSH` de cada
  mensaje crudo a una lista de Redis (`telemetry:mqtt_queue`); un task
  async separado hace `BRPOP` sobre esa misma lista y recién ahí entra al
  pipeline de ingesta compartido (`app/ingestion/pipeline.py`). Redis es
  el desacople real entre el mundo del hilo MQTT y el mundo asyncio.
- **Bloqueo de login** (`app/core/rate_limit.py`): `login_attempts:{user}`
  cuenta intentos fallidos con TTL de `LOGIN_LOCKOUT_WINDOW_SECONDS`; al
  superar `LOGIN_LOCKOUT_MAX_ATTEMPTS` se setea `login_locked:{user}` con
  TTL de `LOGIN_LOCKOUT_DURATION_SECONDS`. Ver el escenario de incidente 1.

## Consecuencias
- Redis no es una dependencia decorativa: si se cae, se pierde tanto la
  ingesta por MQTT (el camino HTTP directo sigue funcionando) como el
  bloqueo por fuerza bruta (degradación aceptada, documentada, no falla
  silenciosa e inadvertida).
- En tests, se reemplaza por `fakeredis` (ver `backend/tests/conftest.py`)
  — nada de infraestructura externa para correr `pytest`.
