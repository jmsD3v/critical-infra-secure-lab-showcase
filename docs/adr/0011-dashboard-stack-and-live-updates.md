# ADR 0011 — Dashboard: Vite + React + TypeScript, WebSocket con semilla REST

## Estado
Aceptado

## Contexto
El enunciado pide un dashboard en React + TypeScript con tendencias en
tiempo real, estado de equipos, alarmas activas y log de eventos.

## Decisión
- **Vite**, no Create React App ni Next.js: es un SPA puro que habla con
  una API/WebSocket externa — no hay SSR ni rutas de servidor que
  justifiquen un framework full-stack. Vite da un build rápido y un
  `Dockerfile` de dos etapas simple (build → nginx sirviendo estático).
- **Sin librería de estado/fetching** (Redux, TanStack Query, etc.): dos
  hooks a medida alcanzan — `useAuth` (login/logout/localStorage) y
  `useTelemetryWebSocket` (conexión WS con reconexión automática). Menos
  superficie para un dashboard de un solo dispositivo con un puñado de
  vistas.
- **WebSocket como fuente de verdad en vivo + REST como semilla inicial**:
  al montar, `Dashboard.tsx` pide histórico (`/telemetry/history`) y
  alarmas por REST para no arrancar en blanco, y a partir de ahí el
  WebSocket (`/ws/telemetry`) va agregando lecturas/alarmas nuevas. Ver
  también `app/ws/routes.py` en el backend sobre el JWT por query param.
- **Tema oscuro industrial** (gris oscuro, nunca negro puro — coherente
  con paneles HMI/SCADA reales) con CSS plano (sin Tailwind ni CSS-in-JS):
  una sola hoja de estilos (`src/styles.css`) con variables CSS, alcanza
  para el tamaño de esta UI.
- **pnpm**, no npm: consistente con el resto de los proyectos del
  portfolio en este disco.

## Consecuencias
- Bundle único sin code-splitting (~530 kB sin gzip, ~154 kB gzip, según
  el build) — aceptable para este alcance; code-splitting por ruta queda
  como mejora si el dashboard crece.
- RBAC también en el frontend (controles de actuador y tab de seguridad
  ocultos para `operator`), pero es solo UX — la autorización real vive
  en el backend (`require_role`, ver ADR 0007); el frontend nunca es el
  único punto de control.
