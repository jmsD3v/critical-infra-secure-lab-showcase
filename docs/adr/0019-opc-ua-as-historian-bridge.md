# ADR 0019 — OPC UA como puente hacia el historian corporativo

## Estado
Aceptado.

## Contexto
Investigando qué protocolos se usan realmente en SCADA de oil & gas
onshore (ver hallazgos citados en el README), aparece un patrón que
Modbus, MQTT y DNP3 no cubren: el puente estandarizado entre el SCADA de
planta y los sistemas corporativos de IT (historian tipo OSIsoft PI o
AVEVA, MES). Ese rol lo cumple **OPC UA**, con sus modos DA (Data Access,
valores en vivo) y HDA (Historical Data Access). Es además transversal a
cualquier vertical OT, no específico de oil & gas — a diferencia de DNP3
(ADR 0016), que sí es propio de esta industria.

## Decisión
- El **backend** (zona IT) expone un servidor OPC UA de solo lectura en
  el puerto estándar 4840, usando `asyncua` (librería madura, MIT) en vez
  de una implementación propia: a diferencia de Modbus/DNP3/IEC104, que
  son protocolos acotados y con valor pedagógico al implementarlos a
  mano, OPC UA es un estándar enorme (discovery, subscripciones,
  seguridad a nivel de sesión, modelado de información) donde reinventar
  la rueda no demuestra nada que una librería no demuestre mejor. Es la
  misma decisión de ingeniería que tomaría un integrador real.
- Expone un namespace con la jerarquía completa (locación → pozo /
  múltiple / batería), como nodos de solo lectura que se actualizan con
  cada ingesta — un cliente OPC UA (historian, o `UAExpert` para
  verificar a mano) puede suscribirse a cualquier variable.
- Seguridad: `Basic256Sha256` con certificado propio del servidor +
  autenticación por usuario/contraseña: nunca anónimo, aun siendo de
  solo lectura, porque la telemetría de producción es información
  comercialmente sensible en una operación real.

## Consecuencias
- El laboratorio pasa a demostrar cuatro protocolos reales (Modbus TCP,
  MQTT, DNP3, OPC UA) cubriendo tres capas distintas: campo, IIoT y
  puente hacia IT/historian — más variedad que el resto de la serie
  OT-Integraciones.
- HART (nivel de instrumento individual, no de RTU) y EtherNet/IP
  (redundante con Modbus/DNP3 en este alcance) quedan documentados como
  investigados y descartados a propósito, no como huecos sin evaluar.
