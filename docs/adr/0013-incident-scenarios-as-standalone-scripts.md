# ADR 0013 — Escenarios de incidente como scripts standalone, no como tests de pytest

## Estado
Aceptado

## Contexto
El enunciado pide simular al menos dos escenarios de incidente
(credenciales inválidas, escritura no autorizada desde OT) y "mostrar"
cómo el sistema los detecta, bloquea y registra. Ya existen tests de
`pytest` que ejercitan RBAC y el bloqueo de login contra una DB en
memoria (`backend/tests/test_auth_rbac.py`) — la pregunta era si alcanzaba
con eso o si valía la pena algo más.

## Decisión
Dos scripts en `incident-scenarios/`, separados de la suite de tests:

- Corren contra el **stack real** levantado con `docker compose up`
  (PostgreSQL, Redis, Mosquitto de verdad), no contra una DB en memoria
  con mocks — la demostración vale más si el "incidente" ocurre contra
  la misma infraestructura que un evaluador puede levantar y mirar.
- **Sin dependencias externas** (solo `urllib` de la stdlib): se pueden
  correr con `python incident-scenarios/scenario_x.py` en cualquier
  máquina con Python 3, sin instalar nada ni activar un venv — baja la
  fricción para que alguien los corra mientras mira el repo.
- Imprimen cada paso en texto legible (qué se intentó, qué respondió el
  backend, qué quedó en el log de seguridad) en vez de solo pasar/fallar
  en silencio como un test — el objetivo es que se puedan *leer* como
  una demostración, no solo ejecutarlos.
- El escenario 2 suma una verificación extra vía `docker compose exec`
  que confirma la segmentación de red real (no alcanzable, no solo
  "no autorizado a nivel aplicación") — algo que un test de `pytest`
  contra `TestClient` no podría probar, porque ahí no existe una red
  Docker real de por medio.

Los tests de `pytest` (`backend/tests/test_auth_rbac.py`) se mantienen
igual — cubren la misma lógica de forma rápida y repetible en CI, sin
depender de que el stack completo esté levantado. Son complementarios,
no redundantes: pytest para verificación continua, estos scripts para
demostración end-to-end.

## Consecuencias
- Los scripts asumen las credenciales seed de `.env.example` — si
  alguien cambia `.env`, tiene que actualizar las constantes al
  principio de cada script (documentado en `incident-scenarios/README.md`).
- La verificación de segmentación de red del escenario 2 es *best-effort*:
  si Docker no está disponible desde donde se corre el script, se salta
  con un aviso en vez de fallar — el resto del escenario (auth + RBAC)
  no depende de eso.
