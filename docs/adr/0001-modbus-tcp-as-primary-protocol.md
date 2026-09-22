# ADR 0001 — Modbus TCP como protocolo de campo primario

## Estado
Aceptado

## Contexto
El laboratorio simula un pozo petrolero en Vaca Muerta. En oil & gas real, el
protocolo de campo dominante en RTUs y PLCs de generación reciente es Modbus
TCP (simple, sin autenticación nativa, texto plano) o DNP3 (más robusto,
pensado para SCADA de transmisión/distribución eléctrica y también usado en
oil & gas, con soporte de reporte por excepción nativo).

## Decisión
Usamos **Modbus TCP simulado** (vía `pymodbus`) como protocolo principal
entre el PLC simulado (`plc-simulator`) y el agente de campo (`field-agent`).
No implementamos DNP3 real: es significativamente más complejo (capas de
aplicación/enlace propias, autenticación segura DNP3-SA) y no aporta más
valor pedagógico que Modbus para demostrar la integración OT/IT de este
laboratorio.

Replicamos el patrón de **reporte por excepción** de DNP3 (ADR 0002) por
software en el agente, en vez de depender de una feature nativa del
protocolo — Modbus TCP no lo soporta a nivel protocolo, así que la lógica
vive en `field_agent/agent.py`.

## Consecuencias
- Modbus TCP no tiene autenticación ni cifrado nativos → la superficie
  insegura del protocolo queda contenida dentro de `ot-net` (red Docker
  `internal: true`, ver ADR 0004), nunca expuesta a la zona IT ni a internet.
- DNP3 queda documentado como mejora futura en el roadmap del README, para
  quien quiera ver una integración más fiel a un SCADA real de oil & gas.
- MQTT se agrega como protocolo secundario (ver
  `0010-mqtt-as-secondary-protocol.md`) para mostrar un gateway
  multi-protocolo, no como reemplazo de Modbus.

## Nota de implementación
`pymodbus` queda fijado en `>=3.6,<3.8` (`field-agent/pyproject.toml`). A
partir de 3.8, la librería deprecó `ModbusSlaveContext` /
`ModbusSequentialDataBlock` (con su API `getValues`/`setValues`) en favor de
un modelo nuevo `SimData`/`SimDevice` pensado para pymodbus 4. Se detectó
probando contra la versión más nueva disponible (3.15.0) durante el
desarrollo: el datastore clásico ya no expone esos métodos. Se optó por
fijar la versión estable anterior en vez de migrar a la API nueva — no
aporta valor pedagógico extra para este laboratorio y agrega inestabilidad
de una librería en pleno rediseño.
