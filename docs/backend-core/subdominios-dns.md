---
sidebar_position: 2
title: Subdominios DNS por usuario
---

# Subdominios DNS por usuario (Entregable 3)

Cualquier usuario autenticado puede solicitar un subdominio propio bajo
`coderhivex.com`, con el formato:

```
[nombre-elegido-por-el-usuario].[nombre-de-su-celula].coderhivex.com
```

Por ejemplo, si el usuario pertenece a la célula `alpha` y pide el nombre
`airflow`, el subdominio resultante es `airflow.alpha.coderhivex.com`.

## Decisión de arquitectura

El backend ya tenía un modelo `Célula` → `Servicio` que calculaba
exactamente este esquema de direccionamiento (`[servicio].[celula].<dominio>`),
pero como una cadena de texto inerte — nunca llamaba a un proveedor DNS real,
y el dominio estaba fijo a `andrescortes.dev` (el dominio interno de la
plataforma, no el de los subdominios de usuario).

En vez de crear un modelo paralelo, se reutilizó `Servicio` para esta
funcionalidad:

- **Crear un servicio ahora crea un registro DNS real** (tipo `A`, "proxied"
  en Cloudflare) antes de guardar la fila en SQL Server.
- El dominio de los servicios se cambió a `coderhivex.com` (el de las
  células en sí — `[celula].andrescortes.dev` — no cambió; es un concepto
  distinto, interno de la plataforma).
- **Alcance deliberadamente limitado**: este flujo crea el registro DNS y
  habilita HTTPS automático (vía el proxy de Cloudflare), pero **no**
  configura nginx para enrutar tráfico real hacia el servicio del usuario —
  eso es un problema de *deployment* del servicio en sí, fuera del alcance
  de "gestión de DNS".

## Por qué HTTPS es automático

El registro se crea con `"proxied": true` en Cloudflare. Eso significa que
Cloudflare actúa como el borde TLS del subdominio (Universal SSL) sin que el
backend tenga que gestionar ningún certificado — el registro empieza a
responder HTTPS válido en cuanto Cloudflare termina de emitir el
certificado (normalmente segundos, puede tardar unos minutos la primera vez
en una zona nueva).

## Flujo

1. El usuario llama `POST /celulas/{celula_id}/services` con el nombre que
   eligió (p. ej. `airflow`).
2. El backend valida que el nombre sea un label DNS válido.
3. El backend crea el registro **A** "proxied" en Cloudflare
   (`airflow.alpha.coderhivex.com`) — el recurso real, primero.
4. El backend llama a `sp_CrearServicio`, que valida cuota, nombre único
   dentro de la célula y que la célula esté activa.
   - Si SQL Server **rechaza** la operación, el backend elimina el registro
     DNS recién creado (compensación) y responde `400` con el motivo exacto.
   - Si SQL Server **acepta**, el trigger `trg_Servicios_Subdominio` calcula
     `subdominio_completo` y el backend responde `201` con el dominio final.

El orden es deliberado: se crea el recurso real (el registro DNS) **antes**
de persistir en SQL Server, con limpieza automática si SQL rechaza la
operación — el mismo patrón que usa el aprovisionamiento de bases de datos
MySQL en este backend.

## Endpoints

Todos requieren `Authorization: Bearer <token>` salvo que se indique lo
contrario.

### `POST /celulas/{celula_id}/services`

Crea el servicio y su registro DNS real.

```json
// Request
{
  "service_name": "airflow",
  "service_type": "other",
  "puerto_interno": 8080
}
```

```json
// 201
{
  "service_id": "…",
  "celula_id": "…",
  "service_name": "airflow",
  "service_type": "other",
  "domain": "https://airflow.alpha.coderhivex.com",
  "database_id": null
}
```

`service_name` se valida como un label DNS real (minúsculas, alfanumérico,
guiones permitidos pero no al inicio/final, máximo 63 caracteres) — un
nombre inválido nunca llega a intentar crear un registro DNS.

Errores relevantes:

| Código | Motivo |
|---|---|
| `422` | `service_name` no es un label DNS válido |
| `400` | Célula suspendida, nombre repetido en esa célula, o **límite de 5 subdominios activos por célula alcanzado** |
| `503` | Cloudflare no respondió o rechazó la operación |

### `GET /celulas/{celula_id}/services`

Lista los servicios (y sus dominios) de una célula.

### `GET /celulas/{celula_id}/services/{service_id}/dns-status`

Verifica si el registro ya propagó, consultando directamente el resolver
DNS-over-HTTPS de Cloudflare (no el caché de un resolver intermedio):

```json
{
  "fqdn": "airflow.alpha.coderhivex.com",
  "propagated": true
}
```

### `DELETE /celulas/{celula_id}/services/{service_id}`

Elimina el servicio **de verdad** (antes de esta funcionalidad, "eliminar"
solo pausaba el servicio — no existía un borrado real). Borra primero el
registro DNS real en Cloudflare y luego marca `ELIMINADO` en SQL Server; si
el borrado en Cloudflare falla, la fila de SQL queda intacta y la operación
es segura de reintentar.

Solo puede eliminarlo el dueño (un usuario cuya célula coincide con la del
servicio) o un administrador.

### `GET /admin/services` (requiere rol `ADMIN`)

Vista global de todos los subdominios creados por todos los usuarios, para
auditoría — paginada, con filtro opcional por estado
(`ACTIVO`/`CAIDO`/`PAUSADO`/`ELIMINADO`).

```bash
curl $BACKEND/admin/services?page=1&page_size=50 \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

La revocación administrativa reutiliza el mismo `DELETE
/celulas/{celula_id}/services/{service_id}` — un administrador puede borrar
el subdominio de cualquier usuario, no solo los propios.

## Límites

- **5 subdominios activos por célula** (validado en `sp_CrearServicio`, no
  en Python — toda la lógica de negocio vive en la base de datos).
- El nombre debe ser único dentro de la célula (y, al ser cada célula
  también única globalmente, esto garantiza subdominios globalmente
  únicos).

## Estado de verificación

El código está desplegado en producción y cubierto por tests unitarios con
un cliente Cloudflare simulado (creación, colisión, borrado idempotente,
verificación de propagación). La verificación en vivo contra la API real de
Cloudflare está pendiente de que se ajuste el *Client IP Filtering* del
token en el dashboard de Cloudflare — actualmente rechaza tanto la IP del
VPS como cualquier IP externa que no esté en la lista permitida. En cuanto
se ajuste, el mismo flujo end-to-end (crear → confirmar en Cloudflare →
propagación → HTTPS → borrar) se verificará contra producción real.
