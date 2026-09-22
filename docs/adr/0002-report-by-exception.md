# ADR 0002 — Reporte por excepción en vez de polling completo

## Estado
Aceptado

## Contexto
Un SCADA de oil & gas real (DNP3 en la mayoría de los casos) no reenvía
cada lectura de cada sensor en cada ciclo de scan: eso satura el enlace
(a veces satelital o de radio, de ancho de banda limitado) sin aportar
información nueva. En cambio, reporta solo cuando un valor cambia más
allá de un umbral ("deadband") o vence un heartbeat.

## Decisión
`field_agent.agent.ReportByExceptionGate` decide si una lectura se reporta:

1. **Deadband por magnitud**: cada tag (`pressure_psi`, `temperature_c`,
   `flow_rate_m3d`, `tank_level_pct`) tiene su propio umbral de cambio
   mínimo (`RBE_*_DEADBAND_*` en `.env`).
2. **Cambio de estado de la bomba**: siempre se reporta, sin deadband —
   un evento discreto (running/stopped/fault) no admite "casi cambió".
3. **Heartbeat**: si no hubo nada que reportar en `RBE_HEARTBEAT_SECONDS`,
   se fuerza un reporte igual. Sin esto, el gateway no podría distinguir
   "todo estable" de "el agente murió" — el heartbeat es la señal de vida.

El PLC simulado sigue respondiendo a *cada* poll Modbus del agente (eso sí
es continuo, como un RTU real) — el reporte por excepción se aplica en la
frontera agente → backend, no en la frontera PLC → agente.

## Consecuencias
- Reduce drásticamente el volumen de datos hacia el backend en operación
  normal (pozo estable = casi silencio), como en un SCADA real.
- El histórico en PostgreSQL refleja "eventos relevantes", no una serie
  temporal de paso fijo — al graficarlo, el dashboard interpola entre
  puntos reportados.
- Trade-off aceptado: un cambio lento y sostenido que nunca cruza el
  deadband en un solo paso, pero sí acumulado, no se reporta hasta el
  heartbeat. Es la misma limitación que tienen los sistemas reales con
  deadband — se documenta, no se "arregla" artificialmente.
