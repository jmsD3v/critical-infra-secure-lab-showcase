# ADR 0004 — Modelo Purdue implementado con redes Docker segmentadas

## Estado
Aceptado

## Contexto
El enunciado pide separar claramente zona OT y zona IT "en el diseño
lógico". Podíamos documentar el modelo Purdue solo en un diagrama, o
hacerlo cumplir de verdad a nivel de red en el `docker-compose.yml`, de
forma que romper la segmentación sea un error real y no solo una
convención en un dibujo.

## Decisión
Tres redes Docker distintas, mapeadas a niveles Purdue:

| Red Docker | Nivel Purdue | Contenedores | Alcance |
|---|---|---|---|
| `ot-net` | 0-2 (campo / control) | `plc-simulator`, `field-agent` | `internal: true` — sin ruta de salida. |
| `dmz-net` | 3.5 (DMZ / conducto) | `field-agent`, `mosquitto`, `backend` | único puente autorizado OT↔IT. |
| `it-net` | 4-5 (empresa) | `backend`, `postgres`, `redis`, `dashboard` | zona de negocio/visualización. |

`field-agent` es el único contenedor con una pata en `ot-net` y otra en
`dmz-net` — es el conducto, no un atajo. `backend` es el único que cruza
`dmz-net` ↔ `it-net`. `plc-simulator` nunca es alcanzable desde `backend`,
`postgres`, `redis` ni `dashboard`: no hay ruta de red posible, incluso si
alguien tuviera credenciales válidas.

## Consecuencias
- La segmentación es verificable con `docker network inspect` y con
  intentos de conexión fallidos reales (usados en el escenario de
  incidente 2, ver `incident-scenarios/`), no solo con un diagrama.
- Compromete el modelo de amenazas: un atacante que gane ejecución en
  `field-agent` (el nodo con más superficie, por estar en ambas redes) es
  el único punto donde un movimiento lateral OT→IT es siquiera
  topológicamente posible. Por eso es también el contenedor con más
  hardening (`cap_drop: ALL`, `no-new-privileges`).
- Limitación reconocida: en producción esto se resolvería con firewalls
  L3 dedicados, diodos de datos unidireccionales y, en `dmz-net`, mTLS
  real en vez de Mosquitto con `allow_anonymous` — ver roadmap.
