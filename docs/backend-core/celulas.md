---
sidebar_position: 6
title: Células
---

# Células (espacios de equipo)

Una **célula** es el espacio de trabajo de un equipo. Cada célula puede
registrar **servicios** (frontend, api, etc.), cada uno respaldado por un
subdominio real en Cloudflare DNS, con el esquema:

```
https://[celula].<root_domain>

https://[servicio].[celula].<root_domain>

```
## Endpoints

| Endpoint | Qué hace |
|---|---|
| `POST /celulas` | Crea una célula nueva, del usuario autenticado |
| `GET /celulas` | Lista las células visibles para el usuario |
| `GET /celulas/{celula_id}` | Obtiene una célula puntual |
| `POST /celulas/{celula_id}/services` | Registra un servicio bajo la célula (crea el subdominio real en Cloudflare) |
| `GET /celulas/{celula_id}/services` | Lista los servicios de una célula |
| `DELETE /celulas/{celula_id}/services/{service_id}` | Elimina un servicio **y su registro DNS real** |
| `GET /celulas/{celula_id}/services/{service_id}/dns-status` | Verifica si el registro DNS ya propagó |
| `PATCH /celulas/services/{service_id}/estado` | Cambia el estado de un servicio (`sp_CambiarEstadoServicio`) |

## Sobre el DNS

El registro de un servicio no solo crea una fila en SQL Server — dispara
una llamada real a la API de Cloudflare para crear el registro DNS. Por
eso existe `GET .../dns-status`: para que el frontend pueda hacer polling
y saber cuándo el subdominio ya es alcanzable, en vez de asumir que está
listo apenas responde el `POST`.
