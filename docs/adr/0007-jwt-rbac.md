# ADR 0007 — JWT + RBAC de dos roles para la API y el dashboard

## Estado
Aceptado

## Contexto
El enunciado pide autenticación JWT con RBAC: rol operador de solo
lectura, rol administrador con control de actuadores.

## Decisión
- **JWT firmado con HS256** (`python-jose`), `sub`=username, `role` como
  claim propio, expiración corta (`JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30`
  por defecto). Sin refresh token: para el alcance de este laboratorio,
  re-loguearse cada 30 minutos es una simplificación aceptable — ver
  roadmap.
- **Dos roles nada más**: `operator` (lectura: telemetría, histórico,
  alarmas) y `admin` (todo lo anterior + emitir comandos de actuador +
  ver el log de eventos de seguridad). `app/api/deps.require_role(*roles)`
  es una dependencia de FastAPI parametrizable, no un sistema de permisos
  genérico — no hacía falta más para dos roles reales.
- **Contraseñas con `bcrypt` directo**, no `passlib`: se detectó durante
  el desarrollo que `passlib` 1.7.4 (sin releases desde 2020) rompe
  contra versiones de `bcrypt` >=4.1 — un self-test interno de passlib
  (`detect_wrap_bug`) asume el truncado silencioso de secrets >72 bytes
  que las versiones viejas de `bcrypt` hacían, y la versión moderna tira
  `ValueError` en su lugar. Se resolvió usando `bcrypt.hashpw`/`checkpw`
  directo (ver `app/core/security.py`), sin la capa de compatibilidad de
  passlib.
- **Todo acceso denegado (401 sin token / token inválido, 403 rol
  insuficiente) se registra como `SecurityEvent`** vía
  `app/security_log/logger.py` — no solo se rechaza, queda auditado.

## Consecuencias
- Dos roles cubren exactamente lo pedido; agregar un tercero (ej.
  "auditor", solo lectura del log de seguridad) es extender la tupla de
  `require_role(...)` en cada ruta, sin tocar el modelo.
- Usuarios seed (`admin`/`operador`) con contraseñas en `.env.example` en
  texto plano: aceptable para un laboratorio de demo, documentado como
  "no usar en producción" en el propio archivo — ver roadmap sobre SSO/
  proveedor de identidad externo.
