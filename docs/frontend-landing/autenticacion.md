---
sidebar_position: 2
title: Autenticación
---

# Autenticación en el frontend

El frontend nunca maneja credenciales — solo redirige al backend y recibe
un `access_token` de vuelta.

## Flujo

1. En `/login`, el usuario elige Google o GitHub → el frontend redirige
   directo a `GET /auth/{provider}` del backend (no hay llamada `fetch`,
   es una navegación completa del navegador).
2. El backend hace todo el intercambio OAuth y redirige de vuelta a
   `/auth/callback?access_token=...`.
3. `/auth/callback` lee el token de la URL, **lo borra de la URL visible
   inmediatamente** (`history.replaceState`) por seguridad, lo guarda, y
   redirige a `/dashboard` o `/provision` según si el usuario ya
   aprovisionó una base de datos o no.

## Dónde vive el token

- El **access token** se guarda en `localStorage`
  (`lib/auth/storage.ts`, clave `dbhosting.access_token`).
- El **refresh token** nunca lo toca el frontend directamente — vive en
  una cookie httpOnly que pone el backend, y el cliente HTTP siempre manda
  `credentials: "include"` para que el navegador la envíe sola.

## Renovación automática (401 → refresh → retry)

`lib/api/client.ts` intercepta cualquier respuesta `401`: si la petición
llevaba un token, llama a `POST /auth/refresh` (la cookie httpOnly viaja
sola), guarda el nuevo access token, y **reintenta la petición original
una sola vez** (con un flag `_isRetry` para no entrar en loop). Si el
refresh también falla, limpia el token guardado y redirige a `/login`.
