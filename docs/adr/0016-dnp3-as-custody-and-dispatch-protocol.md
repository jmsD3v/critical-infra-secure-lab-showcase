# ADR 0016 — DNP3 como protocolo hacia el despacho del oleoducto

## Estado
Aceptado.

## Contexto
El roadmap original ya señalaba a DNP3 como "protocolo de facto en SCADA
de oil & gas de transmisión/distribución". Con la batería central de la
ADR 0015 existe ahora un punto real donde tiene sentido: la medición de
custodia (LACT) y el estado de despacho hacia el operador del oleoducto,
igual que el parque solar usa IEC 60870-5-104 hacia el despacho de la
red eléctrica. Son protocolos distintos a propósito — cada vertical de
la serie OT-Integraciones usa el que le corresponde en la práctica real,
no el mismo por conveniencia.

## Decisión
- Se implementa un subconjunto de DNP3 propio (`dnp3.py`), sin
  librerías de terceros, con el mismo criterio que `iec104.py` en el
  solar: capa de enlace con CRC-16/DNP, capa de transporte con
  segmentación, y un subconjunto de capa de aplicación suficiente para
  representar el caso de uso real:
  - **Binary Input (g1/g2)**: estado de bomba de despacho, alarma de
    BSW alto, alarma de nivel de tanque.
  - **Analog Input (g30/g32)**: caudal certificado LACT, densidad API,
    corte de agua (BSW %), nivel de los 3 tanques.
  - **Class 0/1/2/3 poll** y **eventos espontáneos con deadband**,
    igual que el reporte por excepción del resto del laboratorio.
- `battery-simulator` corre el outstation DNP3 (puerto 20000, el
  estándar de la industria). `field-agent-central` es el master: sondea
  por integridad periódica y recibe eventos espontáneos.
- El enlace DNP3 se ata al estado del vínculo con el operador del
  oleoducto (`pipeline_link`), igual que `operator_link` en el solar:
  si se cae, el outstation deja de responder y el dashboard lo refleja
  como un corte de comunicación, no como una lectura en cero.

## Consecuencias
- El laboratorio pasa a demostrar tres protocolos de campo reales
  (Modbus TCP, MQTT, DNP3) en vez de dos.
- DNP3 real usa autenticación a nivel de aplicación (DNP3 Secure
  Authentication, IEEE 1815) en despliegues productivos; se documenta
  como próximo paso y no se implementa acá, igual que TLS interno para
  Modbus quedó fuera del alcance de ADR 0012.
