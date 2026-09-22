# ADR 0014 — CI básica (lint/test/build), no el pipeline completo

## Estado
Aceptado

## Contexto
El roadmap pedido explícitamente incluye "integración CI/CD" como mejora
futura — pero dejar cero automatización hasta esa mejora futura tampoco
tenía sentido: correr los tests en cada push es la parte barata y de
valor inmediato.

## Decisión
`.github/workflows/ci.yml` con cuatro jobs independientes en paralelo:
`field-agent` y `backend` (instalan con extras `[dev]` y corren
`pytest`), `dashboard` (`pnpm install --frozen-lockfile` + `pnpm run
build`, que ya incluye el type-check de TypeScript), y `compose-config`
(valida que `docker-compose.yml` resuelve sin errores de sintaxis con
`docker compose config`). Nada de build de imágenes, publicación a
registry, ni despliegue — eso es justamente lo que queda en el roadmap
como "CI/CD" completo.

## Consecuencias
- Cualquier PR que rompa un test, el build del dashboard, o la sintaxis
  de `docker-compose.yml` falla en CI antes de mergear.
- No valida que el stack completo levante de verdad (`docker compose up`
  con los servicios corriendo) — eso quedó verificado manualmente en esta
  etapa del desarrollo (ver README, sección de verificación) y es
  candidato natural para sumar como job de CI cuando se implemente el
  pipeline completo del roadmap.
