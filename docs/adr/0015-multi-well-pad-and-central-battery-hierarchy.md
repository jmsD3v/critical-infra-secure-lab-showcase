# ADR 0015 — Locación multi-pozo con batería central, no un pozo suelto

## Estado
Aceptado.

## Contexto
El laboratorio original simula **un solo pozo** con un único PLC y un
único field-agent. Eso ejercita bien el patrón OT/IT (ver ADR 0001-0004),
pero no representa un yacimiento real: una locación no convencional de
Vaca Muerta agrupa varios pozos que comparten un múltiple de recolección
y entregan a una batería de tanques central con separación y medición de
custodia antes de exportar. Es la misma lección que dejó el parque solar
de la serie OT-Integraciones (`solar-ot-lab`): un solo equipo no
demuestra jerarquía, y la jerarquía es justamente lo que un entrevistador
de OT reconoce.

## Decisión
Se reemplaza el pozo único por una locación **VMN-14** (designación de
laboratorio, no un pad real) con:

- **8 pozos** (`well-01` a `well-08`), cada uno con su unidad de bombeo
  mecánico: presión de cabeza, temperatura de boca de pozo, caudal
  individual, corriente del motor, carga en la varilla pulida, velocidad
  (spm) y estado de la bomba (parada/marcha/falla). Ya no tienen tanque
  propio — el nivel de tanque pasa a la batería central, como en una
  locación real.
- **Múltiple de recolección (manifold)**: presión de línea, caudal
  total y una válvula por pozo (abierta/cerrada), análogo a los
  alimentadores de un STS en el solar.
- **Batería de tanques central**: separador trifásico (gas/petróleo/
  agua) con presión y nivel de interfase, 3 tanques de almacenamiento
  con nivel y temperatura, y una unidad LACT (Lease Automatic Custody
  Transfer) que certifica caudal, densidad API y corte de agua (BSW)
  antes de la bomba de despacho al oleoducto.

Contenedores nuevos, siguiendo el mismo patrón bloque/central del solar:

- `pad-simulator`: Modbus TCP que sirve el mapa completo de los 8 pozos
  + manifold (equivalente a un `block-simulator`).
- `battery-simulator`: simula separador, tanques y LACT; expone además
  el outstation DNP3 hacia el despacho del oleoducto (ver ADR 0016).
- `field-agent-pad`: RTU que lee `pad-simulator`, aplica reporte por
  excepción por pozo y publica.
- `field-agent-central`: lee `battery-simulator` y actúa de master DNP3
  hacia `dispatch` (equivalente al agente central del solar con IEC104).

## Consecuencias
- El Modbus map, el modelo de datos, el simulador, el agente, las
  reglas de alarma, la retención y el dashboard se reescriben para la
  jerarquía nueva — no es un ajuste incremental sobre el pozo único.
- El backend gana un nivel de agregación más (locación → pozo) que no
  existía.
- El nombre "VMN-14" es una designación de laboratorio: no se afirma
  que corresponda a una locación real ni a un operador real.
