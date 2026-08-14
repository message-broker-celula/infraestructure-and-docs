---
sidebar_position: 1
title: Visión general
---

# Frontend Landing

Aplicación web en **Next.js 16** (App Router, Turbopack) y **TypeScript**
estricto, con Tailwind CSS. Monorepo con npm workspaces
(`apps/landing` es la app real). Consume la API de
[backend-core](/docs/backend-core) — nunca contiene lógica de negocio
propia, solo la muestra y la orquesta.

## Rutas principales

| Ruta | Qué hace |
|---|---|
| `/` | Landing pública (marketing + métricas públicas de `GET /metrics`) |
| `/login` | Botones de login con Google/GitHub |
| `/auth/callback` | Recibe el `access_token` que redirige el backend tras el OAuth |
| `/dashboard` | Panel del usuario autenticado |
| `/provision` | Flujo de creación de una base de datos nueva |

## Configuración

Variables de entorno (`.env`):

- `NEXT_PUBLIC_CELL_NAME` — nombre de la célula (ej. `mbro`), arma la URL
  del sitio y de la API: `https://{NEXT_PUBLIC_CELL_NAME}.andrescortes.dev`
  y `https://api.{NEXT_PUBLIC_CELL_NAME}.andrescortes.dev`.

## Despliegue

Docker (build standalone de Next.js) detrás de **Caddy** como reverse
proxy, con GitHub Actions para CI/CD.
