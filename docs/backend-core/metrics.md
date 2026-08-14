---
sidebar_position: 8
title: Métricas
---

# Métricas públicas

| Endpoint | Qué hace |
|---|---|
| `GET /metrics` | Métricas agregadas de la plataforma (`fn_MetricasPublicas`) |

**No requiere autenticación** — a propósito: alimenta la sección de
estadísticas públicas de la landing page, pensada para mostrarse a
visitantes antes de que inicien sesión. El frontend hace polling cada
30 segundos. Límite de `60/minute` por rate limiting.
