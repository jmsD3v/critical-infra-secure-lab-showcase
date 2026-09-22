# ADR 0008 — Motor de reglas y detector de anomalías como capas separadas

## Estado
Aceptado

## Contexto
El enunciado describe detección de eventos anómalos en dos lugares con
redacciones distintas: la sección de visualización pide "alarmas
configurables" (umbral, salto brusco, cambio de estado inesperado); la
sección de arquitectura de seguridad pide un "detector de anomalías" que
identifique valores fuera de rango, saltos bruscos **y patrones
inusuales**. Se solapan a propósito — un detector de anomalías serio en
OT normalmente combina reglas determinísticas con algo estadístico.

## Decisión
Dos motores independientes, ambos alimentando la misma tabla `alarms`
(campo `source`: `"rule"` vs `"anomaly"`), visibles juntos en el panel de
alarmas del dashboard:

1. **`app/alarms/engine.py`** — reglas 100% configurables por YAML
   (`rules.yaml`, sin tocar código): umbral alto/bajo, salto máximo entre
   lecturas consecutivas, transición de estado de equipo hacia `fault`.
2. **`app/anomaly/detector.py`** — z-score sobre una ventana móvil de las
   últimas 30 lecturas por `(device_id, tag)`: detecta valores que se
   apartan del comportamiento *reciente*, aunque no crucen ningún umbral
   fijo ni den un salto de un paso al otro (ej. una deriva lenta
   acumulada). Esto es lo que cubre "patrones inusuales" — algo que
   ningún umbral estático puede capturar por diseño.

## Consecuencias
- Cambiar un umbral es editar YAML y reiniciar; cambiar la sensibilidad
  estadística (`z_threshold`, tamaño de ventana) es un parámetro de
  `AnomalyDetector`, no expuesto por config todavía (roadmap).
- El detector de anomalías tiene estado en memoria (las ventanas por
  device+tag) que se pierde al reiniciar el backend — documentado como
  trade-off aceptado para el alcance de este laboratorio, con
  persistencia del estado listada en el roadmap.
- Separar las dos capas hace más legible el "por qué" de cada alarma en
  el dashboard: una regla de umbral dice exactamente qué se violó: un
  z-score alto dice "esto no se parece a lo normal reciente", sin
  necesariamente cruzar ningún límite absoluto.
