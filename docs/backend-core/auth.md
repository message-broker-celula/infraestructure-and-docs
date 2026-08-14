---
sidebar_position: 4
title: Autenticación
---

# Autenticación (OAuth + JWT)

El backend autentica usuarios vía **OAuth con Google o GitHub** — no hay
login con usuario/contraseña propio. Tras el login emite un **access
token JWT** (para el frontend) y un **refresh token con rotación**
(guardado como cookie httpOnly).

## Flujo de login

1. El frontend redirige al usuario a `GET /auth/google` (o `/auth/github`).
2. El backend genera un `state` (protección CSRF), lo guarda en una cookie
   httpOnly y redirige al proveedor OAuth.
3. El proveedor redirige de vuelta a `GET /auth/{provider}/callback` con un
   `code`.
4. El backend valida el `state`, intercambia el `code` por la identidad del
   usuario, y autentica/crea el usuario en SQL Server.
5. Siempre responde con un **302 hacia el frontend**
   (`{FRONTEND_URL}/auth/callback`), nunca con JSON directo — porque este
   endpoint se alcanza por una navegación de nivel superior del navegador,
   no por un `fetch` del frontend.
   - Éxito: `?access_token=...&token_type=bearer`, y setea la cookie
     `refresh_token` (httpOnly, `SameSite=None; Secure`).
   - Error: `?error=<code>&error_description=<mensaje>`.

## Endpoints

| Endpoint | Qué hace |
|---|---|
| `GET /auth/google` | Inicia el flujo OAuth con Google |
| `GET /auth/google/callback` | Callback de Google (ver flujo arriba) |
| `GET /auth/github` | Inicia el flujo OAuth con GitHub |
| `GET /auth/github/callback` | Callback de GitHub |
| `GET /auth/me` | Devuelve el usuario autenticado actual (requiere `Authorization: Bearer <token>`) |
| `POST /auth/refresh` | Rota el refresh token y emite un nuevo access token |
| `POST /auth/logout` | Revoca el/los refresh token(s) y limpia las cookies |

## Refresh tokens y detección de robo

`POST /auth/refresh` acepta el refresh token por body o por cookie. Si SQL
Server detecta que un refresh token **ya rotado** se vuelve a usar (señal
de que fue robado), **revoca todas las sesiones del usuario** — el cliente
debe forzar un login nuevo, no reintentar en silencio.

## Logout

`POST /auth/logout` revoca según lo que tenga disponible, en este orden:

1. Si viene un `refresh_token` en el body → revoca ese token.
2. Si no, pero hay cookie de refresh token → revoca ese.
3. Si no hay ninguno pero el usuario está autenticado (Bearer token) →
   revoca **todos** los refresh tokens de ese usuario.
4. Si no hay nada de lo anterior → `401`.

## Seguridad

- El `state` de OAuth se valida contra una cookie httpOnly propia por
  proveedor (`oauth_state_google` / `oauth_state_github`).
- El refresh token se guarda con `SameSite=None; Secure` (necesario porque
  el frontend y el backend viven en subdominios distintos y el navegador
  debe enviarlo en un `fetch` cross-subdomain).
- El access token nunca se expone en JSON desde el callback — viaja como
  query param en la redirección, y el frontend debe limpiarlo de la URL
  visible inmediatamente (`history.replaceState`).
