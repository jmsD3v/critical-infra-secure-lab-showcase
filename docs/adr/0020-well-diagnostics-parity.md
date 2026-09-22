# ADR 0020 — Diagnóstico de pozo a la par de un sistema real de bombeo

## Estado
Aceptado.

## Contexto
El modelo original de pozo (ver ADR 0015) medía presión de cabeza,
temperatura, caudal, corriente del motor, carga en la varilla y
velocidad — suficiente para el recorrido OT/IT del laboratorio, pero
menos de lo que mide un pozo de bombeo mecánico real. Investigando cómo
son los sistemas de monitoreo de bombeo mecánico reales (pump-off
controllers tipo Weatherford, Lufkin), aparecen tres cosas que un pozo
real sí reporta y este laboratorio no:

- **Presión de casing (anular)**, distinta de la de cabeza (tubing) —
  dos mediciones separadas, no una.
- **Nivel dinámico de fluido**, del que se deriva la sumergencia real de
  la bomba (cuánta columna de fluido la cubre).
- **Diagnóstico de llenado de la bomba**, que un sistema real calcula a
  partir de la carta dinagráfica (carga de la varilla vs. posición) —
  detecta pesca de gas y golpe de fluido antes de que rompan el equipo.

## Decisión
- Se agregan `casing_pressure_psi` y `fluid_level_m` como magnitudes
  propias del pozo, con su propia física en el simulador: la presión de
  casing es un random walk independiente de la de cabeza; el nivel de
  fluido se profundiza mientras la bomba bombea (proporcional al
  caudal) y recupera hacia un nivel estático cuando no.
- En vez de simular la carta dinagráfica punto por punto (carga vs.
  posición del émbolo en cada instante del ciclo), se deriva
  directamente el **resultado** que un sistema real calcularía de esa
  carta: `pump_fillage_pct` (sumergencia real / sumergencia de
  referencia) y `pump_card_diagnosis` (`normal` / `gas_interference` /
  `fluid_pound` / `no_data`) por umbrales sobre ese llenado. Es la misma
  simplificación de alcance que ya se aplicó en el resto del
  laboratorio (no modelar el reservorio real, sí modelar sus síntomas
  de forma creíble) — ver el docstring de `field_agent/simulator.py`.
- `pump_card_diagnosis` se trata como un campo de estado más (igual que
  `pump_status`): un cambio hacia `gas_interference` o `fluid_pound` es
  una transición inesperada que dispara alarma, configurable en
  `rules.yaml` como cualquier otro campo de estado (ver ADR sobre el
  motor de alarmas genérico).
- Se suma un evento aleatorio de "bache de gas" (gas libre entrando a
  la bomba) que profundiza el nivel de fluido de golpe y sube la
  presión de casing — mismo patrón que los demás eventos anómalos del
  simulador, para que el diagnóstico cambie de forma observable en una
  demo corta y no solo tras horas de simulación continua.

## Consecuencias
- El pozo pasa de 6 magnitudes propias a 10, más cerca de lo que un
  ingeniero de producción esperaría ver en un pozo real.
- El % de llenado, no la profundidad cruda del nivel de fluido, es el
  dato sobre el que se pone el umbral de alarma: cada pozo tiene su
  propia profundidad de intake (con dispersión aleatoria), así que un
  umbral fijo sobre `fluid_level_m` no sería comparable entre pozos —
  sobre el llenado normalizado sí.
- Sigue sin simularse la carta dinagráfica real (carga vs. posición
  punto a punto) ni un modelo de reservorio con presión de yacimiento
  declinante — quedan fuera de alcance a propósito, documentado en el
  roadmap del README.
