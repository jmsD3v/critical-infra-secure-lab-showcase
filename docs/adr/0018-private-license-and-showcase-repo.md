# ADR 0018 — Código privado, con repo showcase público

## Estado
Aceptado.

## Contexto
El repo se construyó originalmente en MIT y nunca se publicó en GitHub.
Con el resto de la serie OT-Integraciones (`solar-ot-lab`) el autor
decidió mantener el código de sus laboratorios privado, mostrando el
trabajo mediante un repo showcase público con capturas, video y
documentación de diseño, sin exponer el código fuente. Corresponde
aplicar el mismo criterio acá para que toda la serie sea consistente.

## Decisión
- La licencia pasa de MIT a "todos los derechos reservados" (ver
  `LICENSE`).
- El repo se publica en GitHub como **privado**.
- Se genera `critical-infra-secure-lab-showcase`, público, con el mismo
  script de la serie (`scripts/make_showcase.py`, portado desde el
  solar): README, capturas, video, GIF y los documentos de diseño
  (ADRs, auditoría de seguridad), sin código fuente.

## Consecuencias
- Quien evalúa el proyecto ve el diseño y el resultado sin poder
  clonar ni reutilizar el código.
- El README y el showcase no pueden incluir nada operativamente
  sensible (credenciales, tokens reales, IPs internas) porque ambos
  quedan visibles públicamente por diseño.
