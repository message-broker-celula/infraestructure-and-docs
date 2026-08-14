---
sidebar_position: 1
title: Bases de datos como servicio
---

# Bases de datos como servicio (Entregable 2)

`backend-core` es el equipo responsable de dar bases de datos reales a
cualquier otro equipo de la plataforma — hoy **MySQL** y **PostgreSQL**,
aprovisionadas como contenedores Docker dedicados (uno por base de datos,
un solo tenant cada uno), con el mismo flujo de autoservicio para ambos
motores: se elige motor/versión, se elige un nombre, y en segundos se
recibe una conexión real y funcional.

No es una simulación ni una promesa de "próximamente" — es la misma
infraestructura que usa el propio dashboard de la plataforma para darle
una base de datos a cada usuario.

## Arquitectura, en corto

Un sidecar separado (`provisioner`), el único proceso con acceso al socket
de Docker, crea/detiene/borra los contenedores reales. `backend-core` nunca
toca Docker directamente — le pide al sidecar por HTTP interno, con un
secreto compartido. Cada motor tiene su propio perfil de recursos (memoria,
CPU, flags de arranque) ajustado para no comprometer el VPS compartido:

| Motor | Imagen | Límite de memoria | Notas |
|---|---|---|---|
| MySQL | `mysql:8.0` / `8.4` | 320 MB | `innodb_buffer_pool_size` e `performance_schema` recortados |
| PostgreSQL | `postgres:16` | 192 MB | `shared_buffers`/`max_connections`/`work_mem` recortados |

Hay además un tope global de contenedores aprovisionados simultáneos — si
se alcanza, `POST /databases` responde `503` en vez de arriesgar tumbar el
VPS por falta de memoria.

## Flujo de autoservicio

1. `GET /databases/engines` — listar qué motor/versión se puede pedir *ahora
   mismo* (ver abajo por qué esto importa).
2. `POST /databases` — crear, indicando motor, versión y nombre elegidos.
3. `GET /databases/{id}/credentials` — obtener host, puerto, usuario y
   contraseña reales para conectarse con cualquier cliente estándar
   (`psql`, `mysql`, DBeaver, el driver del lenguaje que sea).
4. Lifecycle normal: `GET /databases` (listar las propias), pausar,
   reanudar, borrar.

Todos los endpoints requieren `Authorization: Bearer <token>` de la
plataforma (el mismo login OAuth de siempre, no una clave aparte).

### Por qué `GET /databases/engines` existe

El catálogo interno (`Motores`) puede tener registrados motores que esta
instancia todavía no sabe aprovisionar de verdad (hoy, por ejemplo,
`SQLSERVER` está en el catálogo pero no implementado como contenedor
real). Este endpoint ya filtra eso — devuelve **solo** lo que
`POST /databases` va a aceptar de verdad, para que un picker en el
frontend nunca ofrezca una opción que después falle.

```json
// GET /databases/engines
{
  "engines": [
    { "nombre_motor": "MYSQL", "version_motor": "8.0" },
    { "nombre_motor": "MYSQL", "version_motor": "8.4" },
    { "nombre_motor": "POSTGRES", "version_motor": "16" }
  ]
}
```

### Crear una base de datos

```json
// POST /databases
{
  "nombre_motor": "POSTGRES",
  "version_motor": "16",
  "nombre_bd": "inventario"
}
```

```json
// 201
{
  "database_id": "…",
  "status": "active",
  "detail": "Database created"
}
```

`nombre_bd` se sanitiza automáticamente a un identificador seguro
(minúsculas, solo `a-z0-9_`, máximo 40 caracteres) — si queda vacío después
de sanitizar (o no se manda), se genera uno por defecto (`app_db`). Todos
los campos son opcionales por compatibilidad, pero lo normal es mandarlos
explícitos.

**Límite:** 5 bases de datos activas por usuario (entre todos los
motores combinados). Al llegar al límite, `POST /databases` responde `400`
con un mensaje claro — no hay un endpoint de cuota aparte, basta con contar
el resultado de `GET /databases`.

### Obtener credenciales de conexión

```json
// GET /databases/{id}/credentials
{
  "host": "…",
  "port": 30412,
  "database_name": "inventario",
  "username": "app_user",
  "password": "…",
  "algoritmo": "AES256"
}
```

## Ejemplo de conexión real (PostgreSQL)

Una vez creada la base y obtenidas las credenciales, se conecta con
cualquier cliente Postgres estándar, sin nada especial de por medio:

```python
import psycopg2

conn = psycopg2.connect(
    host="…",       # de GET /databases/{id}/credentials
    port=30412,
    dbname="inventario",
    user="app_user",
    password="…",
)
```

```bash
psql "host=… port=30412 dbname=inventario user=app_user password=…"
```

## Seguridad

- Cada base de datos vive en su propio contenedor de un solo tenant — no
  hay bases de distintos usuarios compartiendo el mismo motor.
- La contraseña se cifra en reposo (AES-256, `ENCRYPTBYKEY`/`CertCredenciales`)
  — el backend nunca la guarda ni la loguea en texto plano.
- El contenedor real se aprovisiona **antes** que la fila en SQL Server; si
  el registro en SQL falla después, el contenedor recién creado se destruye
  automáticamente en vez de quedar huérfano.

## PostgreSQL para otros equipos (sin login humano)

Todo lo de arriba asume un usuario humano con sesión OAuth en la
plataforma. Para que **otro equipo** consuma el servicio de PostgreSQL
directo desde su propio backend — sin que un humano tenga que iniciar
sesión en el navegador cada vez — existe un canal aparte, machine-to-
machine, bajo `/public/postgres`. Es el mismo patrón que este backend usa
para consumir el Ollama Gateway de IA: registro público → API key de larga
duración → `Authorization: Bearer` en cada llamada, sin OAuth de por
medio.

### Registro (una sola vez)

```json
// POST /public/postgres/register -- sin autenticación previa
{
  "team_name": "Idempotencia",
  "contact_email": "equipo@idempotencia.dev"
}

// 201 -- api_key solo se muestra ACA
{
  "client_id": "…",
  "api_key": "pgk_live_…",
  "key_prefix": "pgk_live_9tK2h"
}
```

### Ciclo de vida (misma forma que `/databases`, pero con API key)

| Endpoint | Igual que |
|---|---|
| `POST /public/postgres/databases` (solo `nombre_bd`, motor fijo a `POSTGRES`) | `POST /databases` |
| `GET /public/postgres/databases` | `GET /databases` |
| `GET /public/postgres/databases/{id}/credentials` | `GET /databases/{id}/credentials` |
| `DELETE /public/postgres/databases/{id}` | `DELETE /databases/{id}` |
| `POST /public/postgres/api-key/rotate` | -- |
| `DELETE /public/postgres/api-key` | -- |

### Por qué esto no necesita tocar nada del aprovisionamiento

Al registrarse, el equipo externo se guarda como una fila "sombra" en
`Usuarios` (`oauth_provider = 'EXTERNAL_API'`, sin login real detrás). Con
eso, **todo** el sistema de bases de datos ya descrito arriba —
`sp_CrearBD`, el límite de 5 bases activas, el cifrado AES-256, el
aprovisionamiento real vía el sidecar — sigue funcionando exactamente
igual, sin ningún cambio. Lo único nuevo es cómo se autentica la llamada
(clave API en vez de JWT) y el registro público.

## Estado de verificación

Ambos motores están desplegados en producción y verificados en vivo (no
solo con tests): se creó, se conectó de verdad (`SELECT version()`), se
revisó el consumo real de memoria contra el límite configurado, y se borró
una base de datos real de cada motor contra la API real, no contra un
entorno de prueba aparte.

El canal `/public/postgres` también está verificado de punta a punta:
registro real → base PostgreSQL real creada y conectada
(`SELECT version()` respondió) → rotación de clave (la anterior murió al
instante, `401` confirmado) → borrado de la base → revocación de la
clave — ciclo completo contra producción, no un entorno aparte.
