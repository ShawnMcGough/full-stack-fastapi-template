# Copilot Instructions

## Architecture

Full-stack app: **FastAPI** backend + **React** frontend, orchestrated via Docker Compose with Traefik reverse proxy.

- **Backend** (`backend/`): FastAPI with SQLModel ORM, PostgreSQL, Alembic migrations, JWT auth. All API routes are under `/api/v1` prefix.
- **Frontend** (`frontend/`): React 19 + TypeScript, TanStack Router (file-based routing), TanStack Query, shadcn/ui + Tailwind CSS v4, Vite.
- **API contract**: The frontend client (`frontend/src/client/`) is **auto-generated** from the backend's OpenAPI schema — never edit files in `src/client/` manually!

## Backend Patterns

### Models (single-file pattern in `backend/app/models.py`)
All SQLModel models live in one file. Each entity follows a hierarchy:
- `EntityBase(SQLModel)` — shared fields (used for validation)
- `EntityCreate(EntityBase)` — creation payload  
- `EntityUpdate(EntityBase)` — update payload (fields optional)
- `Entity(EntityBase, table=True)` — DB table model with `id: uuid.UUID` primary key
- `EntityPublic(EntityBase)` — API response schema
- `EntitiesPublic(SQLModel)` — paginated list response with `data` + `count`

### CRUD (`backend/app/crud.py`)
Standalone functions (not class-based), taking `session: Session` as keyword arg. Pattern: `create_entity(*, session, entity_create) -> Entity`.

### API Routes (`backend/app/api/routes/`)
Each route file defines a `router = APIRouter(prefix="/...", tags=["..."])` and uses dependency injection via `Annotated` types from `backend/app/api/deps.py`:
- `SessionDep` — DB session
- `CurrentUser` — authenticated user
- `get_current_active_superuser` — admin-only dependency

### Configuration
Settings use `pydantic-settings` in `backend/app/core/config.py`, reading from a top-level `.env` file. The singleton `settings` object is imported throughout.

### Security
Password hashing uses `pwdlib` with Argon2+Bcrypt hashers (not passlib). JWT via `pyjwt` with HS256.

### Database Migrations
Run from `backend/` directory: `alembic revision --autogenerate -m "description"`, `alembic upgrade head`. Alembic config points to `app/alembic/`. The `prestart.sh` script runs migrations + seeds initial superuser.

## Frontend Patterns

### Routing (TanStack Router — file-based)
Routes defined in `frontend/src/routes/`. The `_layout.tsx` route wraps authenticated pages with sidebar layout and redirects to `/login` if not authenticated. Route tree is auto-generated in `routeTree.gen.ts`.

### API Client (auto-generated)
Generated via `@hey-api/openapi-ts` from OpenAPI spec. Services are class-based (e.g., `UsersService`, `ItemsService`, `LoginService`). Used with TanStack Query for data fetching.

### State Management
- Auth: JWT token stored in `localStorage`, checked via `isLoggedIn()` in `hooks/useAuth.ts`
- Server state: TanStack Query with `queryKey` patterns like `["currentUser"]`, `["users"]`, `["items"]`
- Forms: `react-hook-form` + `zod` validation

### UI Components
shadcn/ui components in `frontend/src/components/ui/` — don't edit these directly. Use `@` path alias (maps to `src/`). Biome is used for linting/formatting (not ESLint/Prettier).

## Key Commands

| Task | Command |
|------|---------|
| Start all services | `docker compose up -d` (from project root) |
| Backend dev server | Runs via Docker: `fastapi run --reload app/main.py` |
| Backend tests | `docker compose exec backend bash scripts/test.sh` (uses pytest + coverage) |
| Frontend E2E tests | `docker compose run --rm playwright bunx playwright test` |
| Regenerate API client | `bash scripts/generate-client.sh` (extracts OpenAPI JSON from backend, runs openapi-ts) |
| Lint frontend | `bun run lint` (Biome) |
| Lint backend | `ruff check` / `mypy` (strict mode enabled) |
| New Alembic migration | `cd backend && alembic revision --autogenerate -m "msg"` |

## Environment

All config flows through the top-level `.env` file. Required variables: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `SECRET_KEY`, `FIRST_SUPERUSER`, `FIRST_SUPERUSER_PASSWORD`, `FRONTEND_HOST`. The `ENVIRONMENT` variable (`local`/`staging`/`production`) controls behavior — the `/api/v1/private` routes are only mounted in `local` mode.

## When Adding a New Entity

1. Add models to `backend/app/models.py` (Base → Create → Update → DB → Public → ListPublic)
2. Add CRUD functions to `backend/app/crud.py`
3. Create route file in `backend/app/api/routes/`, register in `backend/app/api/main.py`
4. Create Alembic migration
5. Regenerate frontend client via `scripts/generate-client.sh`
6. Add frontend route files in `frontend/src/routes/_layout/`
