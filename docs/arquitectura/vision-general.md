---
sidebar_position: 1
title: Visión general
---

# Visión general de la arquitectura

**Message Broker Celula** está dividido en tres repositorios que se
comunican entre sí por HTTP. El principio central del diseño es
**database-centric**: casi toda la lógica de negocio (validaciones,
permisos, cálculos) vive en Stored Procedures/Views/Functions de SQL
Server (`MessageBrokerDB`), no en el código de la aplicación.

```
┌────────────────────┐ HTTP (REST) ┌──────────────────┐
│ frontend-landing   │ ─────────────────────────▶ │ backend-core   │
│ Next.js 16 + TS    │ ◀───────────────────────── │ FastAPI (Python)│
└────────────────────┘         └────────┬──────────┘
                                   │ Stored Procedures /
                                   │ Views / Functions
                                   ▼
                              ┌──────────────────┐
                              │ SQL Server       │
                              │ MessageBrokerDB  │
                              └──────────────────┘
```

## frontend-landing

Aplicación web en Next.js 16 (App Router) y TypeScript, monorepo con npm
workspaces (`apps/landing`). Cubre autenticación, dashboard y el flujo de
aprovisionamiento de bases de datos. Se despliega con Docker detrás de
Caddy como reverse proxy, y habla con el backend a través de la variable
`NEXT_PUBLIC_API_URL`.

## backend-core

API REST en FastAPI (Python) — ver [Backend Core](/docs/backend-core) para
el detalle de cada módulo. Su única responsabilidad es autenticar
(OAuth Google/GitHub + JWT), aplicar rate limiting, y **traducir cada
petición a un Stored Procedure/View/Function**. No implementa reglas de
negocio en Python — esa es la regla de oro del proyecto.

## SQL Server (MessageBrokerDB)

Fuente de verdad de toda la lógica: validaciones, cálculos, permisos y
flujos de negocio. `backend-core` solo conoce los nombres de los Stored
Procedures a través de una única clase (`SQLServerRepository`); el resto
del código depende de interfaces (`Protocol`) sin saber que SQL Server
existe.

## Por qué database-centric

Mantener la lógica en la base de datos, en vez de en el backend, permite
que las reglas de negocio (límites, permisos, validaciones) sean
consistentes sin importar qué cliente llame a la API, y reduce la
superficie de código en Python a algo que solo autentica y despacha.
