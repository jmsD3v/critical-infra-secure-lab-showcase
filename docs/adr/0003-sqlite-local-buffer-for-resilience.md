# ADR 0003 — Buffer local en SQLite para resiliencia del agente

## Estado
Aceptado

## Contexto
Un enlace de campo (satelital, radio, celular en un yacimiento) se corta.
Un agente que simplemente descarta lecturas cuando falla el push HTTP
pierde datos de producción reales — inaceptable en oil & gas, donde esas
lecturas alimentan reportes regulatorios y de producción.

## Decisión
`field_agent.buffer.SQLiteBuffer` persiste en disco (`/data` en el
contenedor, volumen `field_agent_buffer`) toda lectura que el agente
decidió reportar (post reporte-por-exceptión) y que no pudo entregar al
backend. En cada ciclo, antes de evaluar la lectura actual, el agente
intenta vaciar el buffer en orden FIFO (`_flush_buffer` en `agent.py`):

- Si el backend responde `200/201/202`, la fila se marca `synced=1`.
- Si el backend rechaza o no responde, se corta el flush ahí (no se
  saltea filas) para no romper el orden cronológico de sincronización.

SQLite y no otra cosa: el agente corre en un contenedor de recursos
mínimos en la zona OT, no necesita un motor cliente-servidor para un
buffer local de un solo proceso, y el archivo sobrevive un restart del
contenedor mientras el volumen persista.

## Consecuencias
- Ante un corte de red, el agente sigue funcionando con el PLC (sigue
  leyendo Modbus con normalidad) y solo acumula localmente lo que no
  pudo enviar — sin pérdida de datos, sin bloquear el loop principal.
- Al reconectar, la sincronización es automática, sin intervención
  manual ni reinicio del agente.
- Trade-off aceptado: si el volumen se pierde (ej. se borra el
  contenedor sin el volumen), se pierde el buffer acumulado no
  sincronizado. Igual que un buffer físico en una RTU real con memoria
  no redundada.
