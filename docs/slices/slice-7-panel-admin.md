# Slice 7 — Panel admin (catálogo + conversaciones)

## Objetivo
Dar a un administrador una UI web para gestionar el catálogo (CRUD de productos) y supervisar conversaciones recientes con sus mensajes. Primer release del frontend `apps/admin`.

## Dependencias
- Slice 6 completo (necesario para reactivar conversaciones desde admin)
- Variables nuevas: `ADMIN_JWT_SECRET`, `ADMIN_SEED_EMAIL`, `ADMIN_SEED_PASSWORD`

## Alcance
**Incluye:**
- App React + Vite en `apps/admin`
- Login con email/password (un solo admin seed)
- JWT de sesión en cookie httpOnly + endpoints `/api/admin/*` autenticados
- CRUD de productos (lista, crear, editar, activar/desactivar)
- Listado paginado de conversaciones (orden por `last_message_at desc`)
- Vista de detalle de conversación: lista de mensajes con `direccion`, contenido, timestamp
- Acción "Reactivar" en conversación con `estado='humano'`

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
```
Seed: insertar admin con `email=$ADMIN_SEED_EMAIL` y hash bcrypt del `$ADMIN_SEED_PASSWORD`.

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
- POST `/auth/login` → bcrypt compare → JWT `{adminId,iat,exp}` firmado HS256, expira 12 h, cookie `Secure; HttpOnly; SameSite=Lax`.
- Vite proxy en dev: `/api → http://localhost:3000/api`.

### Performance
- `GET /products` con `LIMIT 50` por defecto, `offset` paginado.
- `GET /conversations` con `LIMIT 30`, ordenado por `last_message_at`.
- Detalle de conversación: últimos 50 mensajes; lazy-load anteriores.

## Archivos a crear / modificar
- `infra/migrations/0005_admins.sql`
- `infra/seeds/admin.sql` (seed condicional desde env)
- `apps/backend/src/routes/admin/{auth,products,conversations,orders}.ts`
- `apps/backend/src/middleware/requireAdmin.ts`
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
- [ ] Admin lista productos paginados con búsqueda por nombre.
- [ ] Admin crea, edita y desactiva productos; cambios visibles en la siguiente cotización.
- [ ] Admin lista conversaciones; cada fila muestra phone, estado, timestamp.
- [ ] Admin abre una conversación y ve historial completo en orden cronológico.
- [ ] Conversación en `humano` muestra botón "Reactivar"; al usarlo, vuelve a `activa` y el agente responde el siguiente mensaje.
- [ ] Pedidos visibles en lectura con su estado y total.
- [ ] Frontend bloquea rutas si no autenticado y redirige a `/login`.
- [ ] Logout limpia cookie y bloquea acceso siguiente.
- [ ] Validación zod retorna 400 con mensajes legibles.
- [ ] Cookie `HttpOnly; Secure; SameSite=Lax` en producción.

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
- **Sin rate limit en login:** vulnerable a brute force; añadir limitador básico (3 intentos/min por IP).
- **Lista de mensajes muy larga rompe rendering:** paginación lazy desde el inicio.
- **Admin único en seed:** si se pierde la contraseña, requiere re-seed; documentar.
