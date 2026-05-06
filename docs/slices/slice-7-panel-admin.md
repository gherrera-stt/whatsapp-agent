# Slice 7 — Panel admin (catálogo + conversaciones)

## Objetivo
Dar a un administrador una UI web para gestionar el catálogo (CRUD de productos) y supervisar conversaciones recientes con sus mensajes. Primer release del frontend `apps/admin`.

## Dependencias
- Slice 6 completo (necesario para reactivar conversaciones desde admin)
- Variables nuevas: `ADMIN_JWT_SECRET`, `ADMIN_SEED_EMAIL`, `ADMIN_SEED_PASSWORD`

## Alcance
**Incluye:**
- App React + Vite en `apps/admin`
- Login con email/password (un solo admin seed) + **rate limit en login (3 intentos / minuto / IP)**
- JWT de sesión en cookie httpOnly. **`Secure` condicional a `NODE_ENV==='production'`** (en dev sin HTTPS la cookie igual se setea).
- Endpoints `/api/admin/*` autenticados; protección CSRF: cookie `SameSite=Lax` + check `Origin` para mutaciones (POST/PUT)
- CRUD de productos (lista, crear, editar, activar/desactivar)
- Listado paginado de conversaciones (orden por `last_message_at desc`)
- Vista de detalle de conversación: lista de mensajes con `direccion`, contenido, timestamp
- Acción "Reactivar" en conversación con `estado='humano'`
- **Seed de admin via script TS** (`scripts/seedAdmin.ts`), no SQL (bcrypt necesita JS)

**No incluye:** SSO, multi-admin, roles granulares, edición de pedidos desde admin (es lectura), métricas (esto es Slice 8/8.5), logs de traces LLM (Slice 8.5).

## Diseño técnico

### Modelo de datos (delta)
```sql
CREATE TABLE admins (
  id            BIGSERIAL PRIMARY KEY,
  email         TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  nombre        TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE admin_login_attempts (
  ip         TEXT NOT NULL,
  ts         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  email      TEXT,
  exitoso    BOOLEAN NOT NULL DEFAULT FALSE
);
CREATE INDEX idx_admin_login_attempts_ip_ts ON admin_login_attempts (ip, ts);
```

### Seed de admin (script TS)
`scripts/seedAdmin.ts`:
1. Lee `ADMIN_SEED_EMAIL` y `ADMIN_SEED_PASSWORD` de env (aborta si faltan).
2. Hash con `bcrypt` (12 rounds).
3. `INSERT ... ON CONFLICT (email) DO NOTHING`.
4. Idempotente: re-correrlo no re-hashea ni cambia password existente. Para resetear contraseña hay un comando aparte `scripts/resetAdminPassword.ts` documentado en runbook (Slice 8).

> SQL no puede hashear bcrypt; por eso el seed va en script TS, no en `infra/seeds/admin.sql`.

### API admin
Todos bajo prefijo `/api/admin`. Middleware `requireAdmin` valida cookie JWT.

| Método | Path | Acción |
|---|---|---|
| `POST` | `/auth/login` | `{email,password}` → set cookie + `{ok:true}` |
| `POST` | `/auth/logout` | clear cookie |
| `GET`  | `/me` | datos del admin actual |
| `GET`  | `/products` | `?q=&activo=&limit=&offset=` |
| `POST` | `/products` | crear |
| `PUT`  | `/products/:id` | editar |
| `POST` | `/products/:id/toggle` | flip `activo` |
| `GET`  | `/conversations` | `?estado=&limit=&offset=` |
| `GET`  | `/conversations/:id` | detalle con últimos N mensajes |
| `GET`  | `/conversations/:id/messages` | paginado |
| `POST` | `/conversations/:id/reactivar` | volver a `activa` |
| `GET`  | `/orders` | `?estado=&phone=&limit=&offset=` (lectura) |
| `GET`  | `/orders/:referencia` | detalle |

### Validación
- `zod` por endpoint (input + output).
- 400 con detalle de validación; 401 sin cookie; 403 si admin no existe; 404 si recurso no existe.

### Frontend (React)
Páginas:
- `/login` — formulario, redirige si ya autenticado
- `/products` — tabla, búsqueda, switch activo, modal de edición
- `/conversations` — tabla con `phone`, último mensaje, estado, `last_message_at`
- `/conversations/:id` — historial completo + botón "Reactivar" si `humano`
- `/orders` — tabla de pedidos (lectura)

Estado: `react-query` para cache + invalidaciones. UI con `tailwind` o `mantine`.

### Auth
- POST `/auth/login`:
  1. Antes de validar credenciales, contar intentos en `admin_login_attempts` para esa IP en el último minuto. Si > 3 → 429 con `Retry-After: 60`.
  2. INSERT registro en `admin_login_attempts` (con `exitoso=false` por default).
  3. bcrypt compare → si OK, UPDATE `exitoso=true`.
  4. Emitir JWT `{adminId,iat,exp}` firmado HS256, expira 12 h, cookie:
     - `HttpOnly: true` siempre.
     - `Secure: true` SÓLO si `NODE_ENV==='production'` (en dev sin HTTPS, `Secure:true` rompe la sesión silenciosamente).
     - `SameSite: 'Lax'`.
- POST/PUT mutaciones bajo `/api/admin/*`: middleware verifica que `Origin` o `Referer` coincida con el host del backend; rechaza si no, defensa adicional contra CSRF aún cuando `SameSite=Lax` ya cubre la mayoría.
- Vite proxy en dev: `/api → http://localhost:3000/api`.

### Performance
- `GET /products` con `LIMIT 50` por defecto, `offset` paginado.
- `GET /conversations` con `LIMIT 30`, ordenado por `last_message_at`.
- Detalle de conversación: últimos 50 mensajes; lazy-load anteriores.

## Archivos a crear / modificar
- `infra/migrations/<timestamp>_admins.sql` (incluye `admin_login_attempts`)
- `apps/backend/scripts/seedAdmin.ts` (seed con bcrypt, idempotente)
- `apps/backend/scripts/resetAdminPassword.ts` (utilidad para runbook)
- `apps/backend/src/routes/admin/{auth,products,conversations,orders}.ts`
- `apps/backend/src/middleware/requireAdmin.ts`
- `apps/backend/src/middleware/csrfOrigin.ts` (check Origin/Referer en mutaciones)
- `apps/backend/src/middleware/loginRateLimit.ts` (3/min/IP)
- `apps/backend/src/services/auth/jwt.ts`
- `apps/backend/src/services/auth/passwords.ts`
- `apps/admin/{package.json,tsconfig.json,vite.config.ts,index.html}`
- `apps/admin/src/{main.tsx,App.tsx,routes.tsx}`
- `apps/admin/src/pages/{Login,Products,Conversations,ConversationDetail,Orders}.tsx`
- `apps/admin/src/lib/{api.ts,auth.ts}`
- `apps/admin/src/components/*` (tabla genérica, modal, etc.)
- `packages/shared/src/types.ts` (tipos Product, Conversation, Order)

## Criterios de aceptación
- [ ] `POST /api/admin/auth/login` con credenciales válidas → 200 + cookie.
- [ ] Endpoint admin sin cookie válida → 401.
- [ ] **4° intento de login en < 60 s desde la misma IP → 429 con `Retry-After`** (test con loop).
- [ ] Admin lista productos paginados con búsqueda por nombre.
- [ ] Admin crea, edita y desactiva productos; cambios visibles en la siguiente cotización.
- [ ] Admin lista conversaciones; cada fila muestra phone, estado, timestamp.
- [ ] Admin abre una conversación y ve historial completo en orden cronológico.
- [ ] Conversación en `humano` muestra botón "Reactivar"; al usarlo, vuelve a `activa`, inserta `[HUMAN HANDOFF fin]` y el agente responde el siguiente mensaje.
- [ ] Pedidos visibles en lectura con su estado y total.
- [ ] Frontend bloquea rutas si no autenticado y redirige a `/login`.
- [ ] Logout limpia cookie y bloquea acceso siguiente.
- [ ] Validación zod retorna 400 con mensajes legibles.
- [ ] Cookie tiene `HttpOnly` siempre; `Secure` sólo en producción; `SameSite=Lax` siempre.
- [ ] **`scripts/seedAdmin.ts` idempotente:** correrlo dos veces no falla y no re-hashea.
- [ ] Mutación POST/PUT con `Origin` ajeno → 403 (test).

## Tests requeridos
- **Unit:** hash de password (bcrypt round-trip). Generador/verificador JWT.
- **Integración:** endpoints admin con Postgres efímero. Verifica 401 sin cookie, 200 con cookie, 403/404 según corresponda.
- **Frontend:** smoke test de cada página con `react-testing-library` (renderiza, llama API mock).
- **E2E manual:** login → editar producto → crear pedido vía WhatsApp → ver pedido en admin.

## Demo
```bash
# Backend
npm run db:migrate && npm run db:seed
npm run dev --workspace=apps/backend

# Admin
npm run dev --workspace=apps/admin
# Abrir http://localhost:5173, login con $ADMIN_SEED_EMAIL
# - Editar precio de un producto
# - Desde teléfono, pedir cotización del producto → precio nuevo
# - "Quiero hablar con humano" desde teléfono → conversación pasa a 'humano'
# - En admin, reactivar conversación → siguiente mensaje recibe respuesta
```

## Riesgos del slice
- **JWT secret comprometido:** documentar rotación; en v1 un solo secret aceptable.
- **CORS mal configurado en producción:** dev usa proxy Vite; prod necesita CORS explícito o servir admin desde el mismo origen.
- **Cookie `Secure: true` en dev sin HTTPS rompe sesión:** mitigado con condicional `NODE_ENV==='production'`.
- **Brute force en login:** mitigado con rate limit 3/min/IP; en producción considerar también captcha y bloqueo por email tras N fallos consecutivos.
- **Lista de mensajes muy larga rompe rendering:** paginación lazy desde el inicio.
- **Admin único en seed:** si se pierde la contraseña, `scripts/resetAdminPassword.ts` + runbook (Slice 8).
- **Tabla `admin_login_attempts` crece sin límite:** GC oportunista en cada inserción (`DELETE WHERE ts < NOW() - INTERVAL '1 day'`).
