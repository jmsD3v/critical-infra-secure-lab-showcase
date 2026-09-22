# ADR 0005 — PostgreSQL para persistencia, sin Alembic

## Estado
Aceptado

## Contexto
El laboratorio pide explícitamente PostgreSQL. La alternativa típica para
series temporales de telemetría sería TimescaleDB (extensión de Postgres
especializada en hypertables), pero el volumen de datos de este lab
(un pozo, reporte por excepción, no miles de sensores a alta frecuencia)
no lo justifica.

## Decisión
- **PostgreSQL 16** simple, con SQLAlchemy 2.0 (driver `psycopg` v3) y un
  esquema relacional directo: `telemetry_readings`, `alarms`,
  `security_events`, `equipment_commands`, `users`.
- **Sin Alembic**: las tablas se crean con `Base.metadata.create_all()` en
  el arranque del backend (`app/main.py`, lifespan). Para un laboratorio
  de portfolio sin múltiples entornos ni historial de esquema que
  versionar, migraciones formales agregan complejidad sin aportar nada
  demostrable — se documenta como mejora de producción en el roadmap.
- **Dedup por `UNIQUE(device_id, sequence)`** en `telemetry_readings`: la
  misma lectura puede llegar por HTTP y por MQTT (ver docs/adr sobre MQTT
  secundario); el segundo intento de insertar el mismo `(device_id,
  sequence)` dispara `IntegrityError`, que el pipeline de ingesta atrapa
  y trata como no-op, no como error.

## Consecuencias
- Setup mínimo: un solo `docker compose up` deja el esquema listo, sin
  paso de migración separado que alguien pueda olvidarse de correr.
- Si el modelo de datos cambia en desarrollo, `create_all()` no altera
  columnas existentes — hay que borrar el volumen (`docker compose down
  -v`) para aplicar cambios de esquema. Aceptable en este contexto de
  laboratorio, documentado para quien lo clone.
- TimescaleDB, particionado por tiempo y migraciones versionadas quedan
  en el roadmap como evolución hacia un uso a escala real.
