---
sidebar_position: 3
title: Inteligencia Artificial como servicio
---

# Inteligencia Artificial como servicio (Entregable 3)

La plataforma no implementa su propio motor de IA — consume un **Ollama
Gateway** ya construido por otro equipo, compatible con la API de OpenAI
(mismo contrato: `chat.completions`, `embeddings`, `models`), con su propia
autenticación por API key, límites de tasa, cuotas diarias/mensuales y
tracking de consumo.

El trabajo de `backend-core` es orquestar la **emisión** de esa clave desde
el panel de la plataforma en vez de que cada usuario tenga que registrarse
manualmente en el gateway, y darle visibilidad a su estado/consumo desde el
propio dashboard.

## Decisión de arquitectura

**El backend nunca reenvía las llamadas de inferencia** (`/v1/chat/completions`,
`/v1/embeddings`, etc.). Esas las hace directamente la aplicación del propio
usuario contra el gateway, usando el SDK de OpenAI con la `base_url`
devuelta al emitir la clave — el gateway ya es público, compatible con el
SDK y está pensado para consumirse así. Meter un proxy intermedio solo
agregaría latencia sin ningún beneficio.

Lo que sí hace `backend-core`:

- **Emitir** una clave nueva, registrando al usuario en el gateway con su
  **propio nombre y correo de la plataforma** (nunca datos que el cliente
  pueda inventar en el request — se leen de `Usuarios.nombre`/`correo`, así
  que nadie puede registrar una clave a nombre de otra persona).
- **Guardar** la clave cifrada en SQL Server (mismo mecanismo —
  `ENCRYPTBYKEY`/`CertCredenciales` — que ya cifra las contraseñas de las
  bases de datos MySQL provisionadas).
- **Consultar estado y consumo** en nombre del usuario, descifrando la
  clave del lado del servidor para llamar al gateway, sin que la clave cruda
  vuelva a salir en esas respuestas.
- **Rotar y revocar**.

## Por qué no hay un endpoint de "borrar" clave

El gateway (según su propia guía) no expone un `DELETE` — solo
`register`/`me`/`me/usage`/`rotate-key`, y rotar invalida la clave anterior
**de inmediato, sin periodo de gracia**. `backend-core` aprovecha eso para
implementar "revocar": rota la clave en el gateway (lo que mata la clave
viva al instante) y **descarta** el resultado de esa rotación en vez de
guardarlo — así no queda ninguna clave utilizable ni del lado del gateway ni
del lado de la plataforma.

## Flujo de emisión

1. El usuario llama `POST /ai/api-key`.
2. El backend lee su propio nombre/correo reales desde SQL Server
   (`fn_ObtenerPerfilUsuario`) — nunca del cuerpo del request.
3. El backend registra ese nombre/correo en el gateway
   (`POST /public/clients/register`), que responde con `api_key` (una sola
   vez), `key_prefix` y `client_id`.
4. El backend cifra y guarda la clave (`sp_RegistrarClaveIA`, AES-256).
5. El backend responde `201` con `api_key`, `key_prefix` y `base_url`.

La `api_key` cruda solo aparece en esta respuesta (y en la de `/rotate`) —
igual que el propio gateway garantiza para su respuesta de registro.

## Endpoints

Todos requieren `Authorization: Bearer <token>` de la plataforma (no la
clave del gateway).

### `POST /ai/api-key`

Emite una clave nueva.

```json
// Request (ambos campos opcionales)
{
  "organization": "Equipo Alpha",
  "intended_use": "Clasificación de tickets de soporte"
}
```

```json
// 201
{
  "client_id": 7,
  "api_key": "sk_live_9tK2h...",
  "key_prefix": "sk_live_9tK2",
  "base_url": "https://api.idempotencia.andrescortes.dev/v1"
}
```

`base_url` ya viene lista para pasarse directo como `base_url` del SDK de
OpenAI. Devuelve `400` si el usuario ya tiene una clave activa (hay que
rotarla o revocarla primero, no se permite tener dos a la vez).

### `GET /ai/api-key`

Estado y límites de la clave activa del usuario — **nunca** la clave cruda.

```json
{
  "client_id": 7,
  "key_prefix": "sk_live_9tK2",
  "status": "approved",
  "can_call_api": true,
  "limits": {
    "requests_per_minute": 20,
    "daily_token_limit": 100000,
    "monthly_token_limit": 2000000
  },
  "last_used_at": "2026-08-13T16:20:11"
}
```

Si el gateway ya no reconoce la clave guardada (revocada del lado de ellos,
fuera del control de esta plataforma), responde `400` con un mensaje claro
en vez de un `503` genérico.

### `POST /ai/api-key/rotate`

Emite una clave nueva e invalida la anterior de inmediato. Misma forma de
respuesta que la emisión inicial.

### `DELETE /ai/api-key`

Revoca la clave activa (ver explicación arriba).

### `GET /ai/usage?start=YYYY-MM-DD&end=YYYY-MM-DD`

Reenvía el desglose de consumo del gateway (por día, por modelo, por
endpoint) tal cual — sin duplicar esa lógica en este backend, el gateway ya
es la fuente de verdad del conteo de tokens.

## Ejemplo de uso desde la aplicación del usuario

Una vez emitida la clave, el usuario consume el gateway **directamente**,
sin pasar por `backend-core`:

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk_live_9tK2h...",       # devuelta por POST /ai/api-key
    base_url="https://api.idempotencia.andrescortes.dev/v1",
)

respuesta = client.chat.completions.create(
    model="qwen2.5:3b",
    messages=[{"role": "user", "content": "Hola"}],
)
```

## Seguridad

- La clave nunca se registra en logs — el cliente HTTP interno
  (`app/ai/clients/ollama_gateway_client.py`) solo la usa como header
  `Authorization`, y los errores de autenticación (`401` del gateway) se
  reportan sin incluir el cuerpo de la respuesta, que podría filtrar
  metadata sensible.
- La clave se cifra en reposo (AES-256) con la misma infraestructura de
  llave simétrica que ya protege las contraseñas de MySQL.
- El nombre/correo con el que se registra en el gateway siempre viene de la
  identidad ya autenticada en la plataforma, nunca de un campo editable por
  el cliente.

## Estado de verificación

✅ Verificado de punta a punta contra el gateway real de producción
(`qa.api.idempotencia.andrescortes.dev`), no solo con tests unitarios:

1. `POST /ai/api-key` → `201`, clave real emitida.
2. `GET /ai/api-key` → `status: "approved"`, `can_call_api: true`.
3. **Inferencia real** con la clave emitida, directo con el SDK de OpenAI
   contra `v1/models` y `v1/chat/completions` — el modelo
   (`qwen2.5:3b`) respondió de verdad.
4. `POST /ai/api-key/rotate` → nueva clave; la anterior murió al instante
   (`401` confirmado contra el gateway).
5. `DELETE /ai/api-key` → revocada; `GET /ai/api-key` vuelve a responder
   `400` "No tienes una clave de IA activa", el estado limpio esperado.

Los dos bloqueadores anteriores ya se resolvieron del lado del equipo de
IA: el hostname correcto es `qa.api.idempotencia.andrescortes.dev` (no
`docs.api...`), y la cuenta de servicio interna
(`EXTERNAL_OWNER_USER_ID`) que hacía falta para aceptar altas externas ya
está configurada.
