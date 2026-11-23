# ⚙️ ZIP Game API — AdonisJS 6

Backend REST de **ZIP Game** (Práctica I · Desarrollo para Dispositivos Inteligentes · UTD · 10°A BIS).
Valida las soluciones, calcula el puntaje **sólo en el servidor** y guarda todo en MySQL 8.

| Tecnología | Uso |
|---|---|
| AdonisJS 6 + TypeScript | API REST (controladores → servicios → repositorios) |
| Lucid ORM + MySQL 8 | Modelos, migraciones y seeders |
| VineJS | Validación de todas las peticiones |
| @adonisjs/auth (access tokens) | Autenticación con token Bearer |
| Japa | Pruebas unitarias y funcionales |

## Requisitos

- Node.js **20.6 o superior** (`node -v`)
- MySQL **8.x** (`mysql --version`)

## Instalación

### 1. Crear la base de datos (una sola vez, como root)

Edita `database/sql/01_create_database.sql`, cambia `CAMBIA_ESTA_CONTRASENA` por tu contraseña y ejecútalo
(en HeidiSQL / Workbench, o en consola):

```powershell
mysql -u root -p < database/sql/01_create_database.sql
```

Crea las bases `zip_game` y `zip_game_test` y el usuario `zip_user`. **No subas a GitHub el archivo con tu contraseña real.**

### 2. Dependencias y variables de entorno

```powershell
npm install
Copy-Item .env.example .env
node ace generate:key
Copy-Item .env.test.example .env.test
```

Abre `.env` y escribe en `DB_PASSWORD` la contraseña de `zip_user`.

### 3. Tablas y datos de demostración

```powershell
node ace migration:run
node ace db:seed
```

Se crean 6 tablas y 10 niveles (Tutorial + 3 fáciles, 3 medios, 3 difíciles) más el usuario de demostración
**demo@zipgame.test / demo12345**.

## Ejecución

```powershell
node ace serve --hmr
```

API en `http://localhost:3333` (las rutas empiezan con `/api`). Para ver las rutas: `node ace list:routes`.

## Pruebas

```powershell
node ace test unit          # 25 pruebas de lógica (sin BD)
node ace test functional    # 12 pruebas de la API contra zip_game_test
node ace test 2>&1 | Tee-Object -FilePath evidencias_backend.txt
```

Resultado esperado: **37 pruebas aprobadas**.

## Estructura

```text
app/
├── controllers/   auth, users, levels, games, leaderboard   (reciben, validan y delegan)
├── services/      validador, solucionador, generador, pistas, puntaje, dificultad adaptativa
├── repositories/  acceso a datos (Lucid)
├── validators/    VineJS
├── models/        user, level, game_session, game_move, score
├── middleware/    auth (Bearer)
├── exceptions/    errores sin stack traces en producción
└── utils/         tablero, RNG con semilla, presentadores
database/
├── migrations/    6 tablas
├── seeders/       niveles y usuario demo
└── sql/           scripts de apoyo y esquema de referencia
tests/
├── unit/
└── functional/
```

## Endpoints

| Método | Endpoint | Auth | Propósito |
|:---:|---|:---:|---|
| POST | `/api/auth/register` | — | Registrar jugador |
| POST | `/api/auth/login` | — | Iniciar sesión |
| POST | `/api/auth/logout` | ✔ | Cerrar sesión |
| GET | `/api/users/me` | ✔ | Perfil y estadísticas |
| GET | `/api/levels` | ✔ | Niveles (sin solución) |
| GET | `/api/levels/:id` | ✔ | Un nivel |
| POST | `/api/levels/generate` | ✔ | Generar nivel con solución garantizada |
| POST | `/api/games` | ✔ | Iniciar partida |
| GET | `/api/games/:id` | ✔ | Estado de una partida propia |
| POST | `/api/games/:id/hint` | ✔ | Pista inteligente |
| POST | `/api/games/:id/finish` | ✔ | Validar solución y calcular puntaje |
| GET | `/api/games/history` | ✔ | Historial |
| GET | `/api/leaderboard` | — | Ranking |

La documentación detallada (cuerpos, respuestas y códigos de error) está en `docs/`.

## Seguridad

- Contraseñas con hash scrypt; nunca se devuelven en el JSON.
- La solución del nivel nunca se envía al cliente.
- Puntaje calculado en el servidor, dentro de una transacción con `SELECT … FOR UPDATE`.
- Una partida sólo genera un puntaje (`UNIQUE` en `scores.game_session_id`) → 409 si se intenta terminar dos veces.
- `.env` y `.env.test` están en `.gitignore`.

## Problemas comunes

| Síntoma | Solución |
|---|---|
| `ER_ACCESS_DENIED_ERROR` | Revisa `DB_USER` / `DB_PASSWORD` en `.env` |
| `ER_BAD_DB_ERROR` | Ejecuta `database/sql/01_create_database.sql` |
| `401` en todas las rutas | Envía `Authorization: Bearer <token>` |
| Error CORS en Chrome | En desarrollo se permite cualquier `localhost`; revisa que `NODE_ENV=development` |
