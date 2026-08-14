---
sidebar_position: 7
title: Auditoría
---

# Auditoría

Un único endpoint para registrar eventos genéricos de la aplicación.

| Endpoint | Qué hace |
|---|---|
| `POST /audit/events` | Registra un evento de auditoría (`sp_RegistrarEvento`) |

```json
// Request
{
  "evento": "string",
  "id_bd": "string opcional",
  "descripcion": "string opcional",
  "resultado": "string opcional",
  "datos_adicionales": {}
}
```

Requiere usuario autenticado. El evento queda asociado al usuario, su IP,
y opcionalmente a una base de datos puntual (`id_bd`) — útil para trazar
qué usuario hizo qué acción sobre qué recurso.
