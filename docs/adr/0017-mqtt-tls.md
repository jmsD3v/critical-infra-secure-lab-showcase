# ADR 0017 — MQTT con TLS, cierra la brecha que dejó ADR 0010

## Estado
Aceptado.

## Contexto
ADR 0010 dejó Mosquitto sin TLS ni autenticación de cliente como
"simplificación deliberada de laboratorio". El parque solar de la serie
(`solar-ot-lab`) cerró esa misma brecha con MQTT sobre TLS 1.3 por
defecto en su entorno público; corresponde aplicar el mismo estándar acá
en vez de dejarlo en el roadmap indefinidamente.

## Decisión
- Mosquitto pasa a exigir TLS (puerto 8883) con un certificado propio
  generado para el laboratorio (`infra/mosquitto/certs/`, no versionado,
  se genera con un script en el primer `docker compose up`).
- Autenticación por usuario/contraseña por agente (`agent-pad`,
  `agent-central`, `backend`) en vez de `allow_anonymous true`, con ACL
  por tópico: cada agente solo publica en `oilfield/vmn14/<su-locación
  o batería>/#`; el backend solo lee.
- `MQTT_TLS_ONLY` como variable de entorno (por defecto `true`);
  desarrollo local puede desactivarlo explícitamente si hace falta
  depurar tráfico en claro, pero el valor por defecto siempre es seguro.

## Consecuencias
- Mismo patrón que el solar: nadie en la red puede leer telemetría ni
  suplantar a un agente sin el certificado y las credenciales.
- mTLS por dispositivo (autenticación por certificado de cliente, no
  solo usuario/contraseña) queda para un uso productivo real, igual que
  ya lo señalaba el roadmap original.
