# Documentación de la API REST — ZIP Game

Backend **AdonisJS 6** (`zip_game_api`). Todas las respuestas son JSON.

- **URL base:** `http://localhost:3333/api` (emulador Android: `http://10.0.2.2:3333/api`)
- **Autenticación:** token Bearer. Se obtiene en `register` o `login` y se envía en cada petición protegida:
  ```http
  Authorization: Bearer oat_MQ.xxxxxxxxxxxxxxxx
  ```
  El token expira en 7 días. En la base de datos sólo se guarda su hash.
- **Celdas:** el tablero es 6×6 y cada celda es un número de 0 a 35 (`fila × 6 + columna`).
- **Dificultades:** `easy`, `medium`, `hard`.
- Los JSON de este documento son **ejemplos** del formato; los ids, celdas y métricas cambian en cada base de datos.

## Resumen

| Método | Endpoint | Auth | Propósito | Éxito |
|---|---|---|---|---|
| GET | `/` (sin `/api`) | No | Estado del servidor | 200 |
| POST | `/api/auth/register` | No | Registrar jugador | 201 |
| POST | `/api/auth/login` | No | Iniciar sesión | 200 |
| POST | `/api/auth/logout` | Sí | Cerrar sesión (revoca el token) | 200 |
| GET | `/api/users/me` | Sí | Perfil y estadísticas | 200 |
| GET | `/api/levels` | Sí | Niveles disponibles (sin solución) | 200 |
| GET | `/api/levels/:id` | Sí | Un nivel (sin solución) | 200 |
| POST | `/api/levels/generate` | Sí | Generar y guardar un nivel nuevo | 201 |
| POST | `/api/games` | Sí | Iniciar partida | 201 |
| GET | `/api/games/:id` | Sí | Estado de una partida propia | 200 |
| POST | `/api/games/:id/hint` | Sí | Pedir una pista | 200 |
| POST | `/api/games/:id/finish` | Sí | Validar la solución y terminar | 200 |
| GET | `/api/games/history` | Sí | Historial del jugador | 200 |
| GET | `/api/leaderboard` | No | Ranking | 200 |

Los 9 endpoints mínimos de la práctica están incluidos, más los de funciones avanzadas (`hint` y `levels/generate`) y tres de apoyo (`logout`, `games/:id` y `/`).

## Códigos de respuesta

| Código | Cuándo |
|---|---|
| 200 | Éxito |
| 201 | Recurso creado (registro, partida, nivel generado) |
| 400 | Credenciales inválidas en el login |
| 401 | Sin token o token inválido/expirado |
| 404 | El recurso no existe **o pertenece a otro jugador** (no se revela cuál de los dos) |
| 409 | La partida ya fue terminada |
| 422 | Error de validación de datos (VineJS) o solución inválida |
| 500 | Error interno. En producción sólo responde `{ "message": "Internal server error", "code": "E_INTERNAL" }`, sin stack trace ni SQL |

Formatos de error:

```json
// 422 de validación (VineJS)
{ "errors": [ { "field": "email", "rule": "database.unique", "message": "The email has already been taken" } ] }

// 401 sin token
{ "errors": [ { "message": "Unauthorized access" } ] }

// Errores del juego (404, 409, 422 de solución)
{ "message": "This game session is already finished", "code": "E_ALREADY_FINISHED" }
```

---

## Autenticación

### POST `/api/auth/register`

Crea un jugador y devuelve su token. La contraseña se guarda con hash scrypt.

| Campo | Tipo | Reglas |
|---|---|---|
| `username` | string | 3–30 caracteres, sólo letras, números y `_`, único |
| `email` | string | correo válido, máx. 254, único |
| `password` | string | 8–64 caracteres |

```json
// Petición
{ "username": "test", "email": "test@mail.com", "password": "secreto123" }

// 201
{
  "user": {
    "id": 2, "username": "test", "email": "test@mail.com",
    "recommended_difficulty": "easy", "created_at": "2026-11-23T05:35:27.000+00:00"
  },
  "token": { "type": "bearer", "value": "oat_Mg.xxxxxxxx", "expires_at": "2026-11-30T05:35:27.000Z" }
}
```

Errores: **422** si falta un campo, el formato es incorrecto o el usuario/correo ya existe.

### POST `/api/auth/login`

| Campo | Tipo | Reglas |
|---|---|---|
| `email` | string | correo válido |
| `password` | string | requerido |

Respuesta **200** con el mismo formato que `register` (`user` + `token`).

Errores: **400** credenciales inválidas · **422** datos con formato incorrecto.

### POST `/api/auth/logout` 🔒

Revoca el token con el que se hizo la petición.

```json
// 200
{ "message": "Logged out" }
```

---

## Perfil

### GET `/api/users/me` 🔒

Datos del jugador autenticado, estadísticas y niveles completados.

```json
// 200
{
  "user": { "id": 2, "username": "test", "email": "test@mail.com", "recommended_difficulty": "medium", "created_at": "…" },
  "stats": {
    "games_played": 4, "games_completed": 4, "levels_completed": 4,
    "total_points": 3670, "best_score": 1290
  },
  "completed_levels": [
    { "id": 1, "name": "Tutorial", "difficulty": "easy", "best_score": 1290 }
  ]
}
```

---

## Niveles

Las respuestas de niveles **nunca incluyen la solución**; sólo la posición de los números.

### GET `/api/levels` 🔒

| Query | Tipo | Opcional |
|---|---|---|
| `difficulty` | `easy` \| `medium` \| `hard` | Sí (filtra) |

```json
// 200
{
  "recommended_difficulty": "medium",
  "data": [
    {
      "id": 1, "name": "Tutorial", "difficulty": "easy", "size": 6,
      "checkpoints": [6, 0, 14, 23, 35, 30, 18, 25, 3, 11, 20, 33],
      "checkpoint_count": 12, "difficulty_score": 18, "is_generated": false,
      "completed": true, "best_score": 1290
    }
  ]
}
```

`checkpoints[0]` es la celda del número 1, `checkpoints[1]` la del 2, etc. `completed` y `best_score` son del jugador que hace la petición.

### GET `/api/levels/:id` 🔒

```json
// 200
{ "data": { "id": 1, "name": "Tutorial", "difficulty": "easy", "size": 6, "checkpoints": […], "checkpoint_count": 12, "difficulty_score": 18, "is_generated": false } }
```

Errores: **404** `{ "message": "Resource not found", "code": "E_NOT_FOUND" }`.

### POST `/api/levels/generate` 🔒 *(función avanzada)*

Genera un tablero nuevo a partir de un camino hamiltoniano (movimientos *backbite*), comprueba con el solucionador que tiene solución, mide su dificultad y lo guarda en MySQL.

| Campo | Tipo | Reglas |
|---|---|---|
| `difficulty` | `easy` \| `medium` \| `hard` | requerido |
| `name` | string | opcional, 3–80 caracteres (si no se envía se genera, p. ej. "Breeze #809") |

```json
// Petición
{ "difficulty": "easy" }

// 201
{
  "data": { "id": 10, "name": "Breeze #809", "difficulty": "easy", "size": 6, "checkpoints": […], "checkpoint_count": 11, "difficulty_score": 22, "is_generated": true },
  "metrics": {
    "checkpointCount": 11, "averageGap": 3.5, "solutionCount": 1,
    "searchEffort": 412, "turns": 21, "score": 22
  }
}
```

`metrics` explica la dificultad: cantidad de números (fácil 10–13, media 7–9, difícil 4–6), separación promedio entre ellos, número de soluciones posibles, esfuerzo de búsqueda del solucionador y vueltas del camino.

---

## Partidas

### POST `/api/games` 🔒

Inicia una partida. La hora de inicio la pone el servidor.

| Campo | Tipo | Reglas |
|---|---|---|
| `level_id` | entero | opcional. Si no se envía, el servidor elige un nivel pendiente de la dificultad recomendada (dificultad adaptativa) |

```json
// Petición
{ "level_id": 1 }

// 201
{
  "session": {
    "id": 7, "level_id": 1, "status": "in_progress", "difficulty": "easy",
    "started_at": "2026-11-23T15:46:37.000+00:00", "finished_at": null,
    "duration_seconds": null, "mistakes": 0, "hints_used": 0, "failed_attempts": 0
  },
  "level": { "id": 1, "name": "Tutorial", "difficulty": "easy", "size": 6, "checkpoints": […], "…": "…" }
}
```

Errores: **404** `E_NOT_FOUND` (el nivel no existe) · **404** `E_NO_LEVELS` (sin `level_id` y no quedan niveles pendientes; hay que generar uno).

### GET `/api/games/:id` 🔒

```json
// 200
{ "session": { "id": 7, "level_id": 1, "status": "completed", "…": "…" } }
```

Errores: **404** `E_SESSION_NOT_FOUND` si no existe o es de otro jugador.

### POST `/api/games/:id/hint` 🔒 *(función avanzada)*

El servidor revisa el camino actual y sugiere **sólo la siguiente celda**, o cuántos pasos deshacer si el camino ya no tiene salida. Cada pista suma 1 a `hints_used` y resta 150 puntos al final.

| Campo | Tipo | Reglas |
|---|---|---|
| `path` | entero[] | celdas 0–35, máx. 36 (puede ir vacío) |

```json
// Petición
{ "path": [6, 0, 1, 7] }

// 200 — seguir adelante
{ "hint": { "type": "next", "next_cell": 8 }, "hints_used": 1 }

// 200 — el camino no tiene salida
{ "hint": { "type": "backtrack", "undo_steps": 22, "next_cell": 13 }, "hints_used": 1 }
```

Errores: **404** `E_SESSION_NOT_FOUND` · **409** `E_ALREADY_FINISHED` · **422** si el camino enviado ya rompe una regla (mismos códigos que en *finish*) o ya está completo (`E_COMPLETE`).

### POST `/api/games/:id/finish` 🔒

Valida la solución completa **en el servidor** y, si es correcta, calcula el puntaje, guarda los movimientos y el puntaje en una transacción y actualiza la dificultad recomendada. El cliente nunca envía puntos.

| Campo | Tipo | Reglas |
|---|---|---|
| `path` | entero[] | celdas 0–35, 1 a 36 elementos |
| `moves` | objeto[] | opcional, máx. 5000. Cada uno: `type` (`add` \| `undo` \| `invalid` \| `restart`), `cell` (0–35 o null), `elapsed_ms` (≥ 0) |

```json
// Petición
{
  "path": [6, 0, 1, 7, 8, 2, "…36 celdas…"],
  "moves": [
    { "type": "add", "cell": 6, "elapsed_ms": 3401 },
    { "type": "invalid", "cell": 12, "elapsed_ms": 8766 },
    { "type": "undo", "cell": 7, "elapsed_ms": 5723 }
  ]
}

// 200 — solución válida
{
  "message": "Puzzle solved!",
  "session": { "id": 7, "status": "completed", "duration_seconds": 15, "mistakes": 1, "hints_used": 0, "…": "…" },
  "score": {
    "points": 1290, "duration_seconds": 15, "mistakes": 1, "hints_used": 0,
    "breakdown": { "base": 1000, "time_bonus": 315, "time_penalty": 0, "mistake_penalty": 25, "hint_penalty": 0 }
  },
  "performance": "excellent",
  "difficulty": { "previous": "easy", "recommended": "medium", "change": "increase", "reason": "2 excellent games in the last 3" }
}

// 422 — solución inválida (la partida sigue abierta y suma un intento fallido)
{
  "message": "The path covers 30 of 36 cells",
  "code": "E_INCOMPLETE",
  "details": { "index": 30 },
  "session": { "id": 7, "status": "in_progress", "failed_attempts": 1, "…": "…" }
}
```

Motivos de solución inválida (`code`):

| Código | Regla que se rompió |
|---|---|
| `E_EMPTY_PATH` | El camino está vacío |
| `E_OUT_OF_BOUNDS` | Una celda está fuera del tablero |
| `E_INVALID_START` | No empieza en el número 1 |
| `E_REPEATED_CELL` | Repite una celda |
| `E_NOT_ADJACENT` | Un paso no es horizontal/vertical (diagonal o salto) |
| `E_CHECKPOINT_ORDER` | Llega a un número fuera de orden |
| `E_PREMATURE_END` | Llega al número más alto antes de llenar el tablero |
| `E_INCOMPLETE` | No pasa por las 36 celdas |
| `E_INVALID_END` | No termina en el número más alto |

`details.index` es la posición dentro de `path` donde se detectó el error.

Otros errores: **404** `E_SESSION_NOT_FOUND` · **409** `E_ALREADY_FINISHED` (evita un segundo puntaje; además `scores.game_session_id` es UNIQUE).

**Cálculo del puntaje (sólo en el servidor):**

| | Fácil | Media | Difícil |
|---|---|---|---|
| Base | 1000 | 2000 | 3000 |
| Tiempo meta | 120 s | 180 s | 300 s |

`puntos = base + 3 × (segundos por debajo de la meta) − 2 × (segundos por encima de la meta) − 25 × errores − 150 × pistas`, con un **mínimo del 10 % de la base** (terminar siempre da puntos). Los errores se cuentan con los movimientos `invalid` enviados en `moves`, y la duración con las horas del servidor.

**Dificultad adaptativa:** cada partida se califica como `excellent` (dentro del tiempo meta, sin pistas y con máx. 2 errores), `poor` (más del doble del tiempo meta, 3 o más pistas o 10 o más errores) o `normal`. Con 2 partidas excelentes entre las últimas 3 de la dificultad actual, la recomendación sube; con 2 deficientes, baja; si no, se mantiene. Sólo afecta las partidas siguientes.

### GET `/api/games/history` 🔒

Partidas del jugador autenticado (nunca las de otros), terminadas y en curso, de la más reciente a la más antigua. `points` es `null` si la partida no tiene puntaje.

| Query | Tipo | Por defecto |
|---|---|---|
| `page` | entero ≥ 1 | 1 |
| `per_page` | entero 1–50 | 10 |

```json
// 200
{
  "meta": { "total": 4, "per_page": 10, "current_page": 1, "last_page": 1 },
  "data": [
    {
      "id": 8, "level_id": 5, "status": "completed", "difficulty": "easy",
      "started_at": "…", "finished_at": "…", "duration_seconds": 523,
      "mistakes": 5, "hints_used": 1, "failed_attempts": 0,
      "level": { "id": 5, "name": "Meadow" }, "points": 100
    }
  ]
}
```

---

## Ranking

### GET `/api/leaderboard`

Público (no requiere token).

| Query | Tipo | Por defecto |
|---|---|---|
| `limit` | entero 1–100 | 20 |

```json
// 200
{
  "rules": "Sum of best score per level; ties: more levels, less total time",
  "data": [
    { "rank": 1, "user_id": 1, "username": "demo", "total_score": 5026, "levels_completed": 3, "total_time_seconds": 78, "best_single_score": 2459 },
    { "rank": 2, "user_id": 2, "username": "test", "total_score": 3670, "levels_completed": 4, "total_time_seconds": 643, "best_single_score": 1290 }
  ]
}
```

**Regla de orden:**
1. Por cada nivel se toma el **mejor puntaje** del jugador (repetir un nivel no suma dos veces).
2. `total_score` = suma de esos mejores puntajes. Ordena de mayor a menor.
3. Desempates: más niveles completados → menor tiempo total → registro más antiguo.

---

## Seguridad aplicada

- Todas las entradas se validan con **VineJS** (cuerpo y query).
- Rutas protegidas con el middleware `auth` (token Bearer). Un jugador no puede ver, pedir pistas ni terminar partidas ajenas: responde 404.
- Contraseñas con hash scrypt; los presentadores (`app/utils/presenters.ts`) definen exactamente qué campos salen, así que ni la contraseña ni la solución del nivel aparecen en ninguna respuesta.
- El puntaje y la duración se calculan en el servidor; el cliente no puede enviarlos.
- `finish` usa una transacción con `SELECT … FOR UPDATE`, responde 409 si la partida ya terminó y `scores.game_session_id` es UNIQUE.
- CORS permite `localhost` con cualquier puerto en desarrollo (Flutter Web) y orígenes extra con `CORS_ORIGINS`.
- Variables sensibles en `.env`, fuera de GitHub.

## Probar rápido con curl

```bash
# login
curl -s -X POST http://localhost:3333/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"demo@zipgame.test","password":"<tu-contraseña>"}'

# niveles (pega el token del login)
curl -s http://localhost:3333/api/levels -H "Authorization: Bearer <token>"

# ranking (público)
curl -s http://localhost:3333/api/leaderboard
```

Las rutas registradas se pueden listar con `node ace list:routes`.
