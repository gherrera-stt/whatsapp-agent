# Slice 0 — Andamiaje

## Objetivo
Repositorio listo para iterar: monorepo TypeScript con CI mínima, base de datos local con catálogo cargado y endpoint de salud. Cimiento sobre el que se apoyan todos los slices siguientes.

## Dependencias
- Node.js ≥ 20 LTS, Docker, npm o pnpm
- `dbmate` instalado localmente (binario o `npx dbmate`)
- Cuenta GitHub (repo creado)
- Variables nuevas: `DATABASE_URL`, `PORT`, `NODE_ENV`, `LOG_LEVEL`, `BUSINESS_TZ` (default `America/Santiago`), `CURRENCY` (default `CLP`)

## Alcance
**Incluye:**
- Estructura de monorepo con workspaces (`apps/backend`, `apps/admin`, `packages/shared`)
- TypeScript en modo `strict` en todos los workspaces
- ESLint + Prettier compartidos, pre-commit con `husky` + `lint-staged`
- `docker-compose.yml` con PostgreSQL 16
- Migration runner **`dbmate`** (SQL puro, naming por timestamp `YYYYMMDDHHMMSS_descripcion.sql`); migración inicial con tabla `products`
- Seed de catálogo (≥ 10 productos de muestra)
- Endpoint `GET /health` → `{status, db, version}`
- Logger `pino` estructurado con `requestId`
- GitHub Actions: lint → typecheck → test → build

**No incluye:** lógica de WhatsApp, Claude, admin, ni autenticación.

## Diseño técnico

### Estructura inicial
```
whatsapp-agent/
├── apps/backend/
│   ├── src/{routes,services,db,domain,index.ts}
│   └── tests/
├── apps/admin/                  # placeholder vacío en este slice
├── packages/shared/             # tipos zod compartidos (vacío inicial)
├── infra/
│   ├── docker-compose.yml
│   ├── migrations/20260506000000_products.sql       # naming dbmate (timestamp)
│   └── seeds/products.sql
├── .github/workflows/ci.yml
├── .env.example
├── package.json                 # workspaces
└── README.md
```

### Modelo de datos (delta)
```sql
-- migrate:up
CREATE TABLE products (
  id           BIGSERIAL PRIMARY KEY,
  sku          TEXT UNIQUE NOT NULL,
  nombre       TEXT NOT NULL,
  descripcion  TEXT,
  precio       NUMERIC(12,2) NOT NULL CHECK (precio >= 0),
  stock        INT NOT NULL DEFAULT 0 CHECK (stock >= 0),
  activo       BOOLEAN NOT NULL DEFAULT TRUE,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_products_nombre_lower ON products (LOWER(nombre));
CREATE INDEX idx_products_activo ON products (activo) WHERE activo = TRUE;

-- migrate:down
DROP TABLE products;
```

> **Idempotencia.** `dbmate` lleva su propia tabla `schema_migrations` y nunca corre dos veces el mismo archivo. No es necesario `CREATE TABLE IF NOT EXISTS` en el SQL — la idempotencia está garantizada por el runner. Si un dev necesita re-bootear desde cero, `dbmate drop && dbmate up`.

### Endpoint `/health`
- Hace `SELECT 1` contra Postgres con timeout de 1 s.
- 200 con `{status:"ok", db:"ok", version}` si DB responde.
- 503 con `{status:"degraded", db:"down"}` si no.

### CI workflow
1. Checkout + setup-node 20
2. `npm ci`
3. `npm run lint`
4. `npm run typecheck`
5. Servicio `postgres:16` levantado, `npm run db:migrate`
6. `npm run test --workspaces`
7. `npm run build --workspaces`

### Configuración con zod
`apps/backend/src/config.ts` valida env al boot y aborta si falta algo. Sin `process.env` directo en el resto del código. Incluye `BUSINESS_TZ` (zona horaria de presentación, validada con `Intl.DateTimeFormat`) y `CURRENCY` (código ISO 4217).

## Archivos a crear / modificar
- `package.json` (root, con `workspaces` y scripts)
- `tsconfig.base.json`
- `.eslintrc.cjs`, `.prettierrc`, `.editorconfig`
- `apps/backend/{package.json,tsconfig.json}`
- `apps/backend/src/index.ts`
- `apps/backend/src/config.ts`
- `apps/backend/src/routes/health.ts`
- `apps/backend/src/db/client.ts`
- `apps/backend/src/logger.ts`
- `infra/docker-compose.yml`
- `infra/migrations/<timestamp>_products.sql`
- `infra/seeds/products.sql`
- `apps/backend/src/lib/format.ts` (formateadores con `BUSINESS_TZ` y `CURRENCY`)
- `.github/workflows/ci.yml`
- `.env.example`
- `.nvmrc` (`20`)
- `README.md`

## Criterios de aceptación
- [ ] `docker compose -f infra/docker-compose.yml up -d` levanta Postgres en `localhost:5432`.
- [ ] `npm install` sin errores.
- [ ] `npm run db:migrate` (alias de `dbmate up`) aplica la migración inicial; segunda corrida es no-op (`dbmate` registra en `schema_migrations`).
- [ ] `npm run db:seed` carga ≥ 10 productos.
- [ ] `npm run dev --workspace=apps/backend` levanta servidor en `:3000`.
- [ ] `curl localhost:3000/health` → 200 con `{status:"ok", db:"ok"}`.
- [ ] Detener Postgres y reintentar `/health` → 503 con `{status:"degraded"}`.
- [ ] Logs son JSON válido (pino) con `level`, `time`, `msg`, `requestId`.
- [ ] CI verde en PR de prueba.
- [ ] `npm run lint` y `npm run typecheck` corren sin errores.
- [ ] `tsconfig` con `strict: true`, `noImplicitAny: true`.
- [ ] `husky` ejecuta `lint-staged` en pre-commit.

## Tests requeridos
- **Unit:** parser de configuración (env válido / inválido).
- **Integración:** `/health` happy path con Testcontainers para Postgres efímero.
- **E2E:** no aplica.

## Demo
```bash
docker compose -f infra/docker-compose.yml up -d
npm install
cp .env.example .env
npm run db:migrate
npm run db:seed
npm run dev --workspace=apps/backend &
curl -s http://localhost:3000/health | jq
psql "$DATABASE_URL" -c "SELECT count(*) FROM products WHERE activo = TRUE;"
```
Esperado: `/health` 200 con db ok, conteo ≥ 10.

## Riesgos del slice
- **Versiones de Node divergentes entre devs:** fijar `engines` en `package.json` y `.nvmrc`.
- **Workspaces mal configurados rompen builds más adelante:** correr `npm run build --workspaces` desde el inicio.
- **Conflicto de naming de migraciones cuando dos devs trabajan en paralelo:** el timestamp de `dbmate` minimiza el riesgo; documentar en `infra/migrations/README.md` que el archivo se nombra al momento de crear el PR (no al iniciar la rama) si dos personas crean migraciones en la misma jornada.
