---
sidebar_position: 5
title: Administración
---

# Administración

Endpoints restringidos al rol **admin** (`require_role("admin")`, validado
también en los propios Stored Procedures como defensa en profundidad).
Dan visibilidad global sobre usuarios, roles y bases de datos/servicios de
toda la plataforma.

| Endpoint | Qué hace |
|---|---|
| `GET /admin/users` | Lista todos los usuarios registrados (paginado) |
| `PATCH /admin/users/{user_id}/role` | Cambia el rol de un usuario |
| `GET /admin/databases` | Vista global de todas las bases de datos aprovisionadas (filtrable por `status`: `ACTIVA`/`PAUSADA`/`ELIMINADA`) |
| `GET /admin/services` | Vista global de todos los servicios/subdominios registrados (filtrable por `status`: `ACTIVO`/`CAIDO`/`PAUSADO`/`ELIMINADO`) |

Todos devuelven `403` si el usuario no tiene rol admin, y `503` si falla la
integración con SQL Server.
