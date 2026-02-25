---
applyTo: "backend/**"
---

# Backend Instructions

## Project Structure

- `app/models.py` — **All** SQLModel models in one file. Never split models across files.
- `app/crud.py` — Standalone CRUD functions (not class-based). All take `session: Session` as keyword-only arg via `*`.
- `app/api/routes/` — One file per resource, each defines `router = APIRouter(prefix="/...", tags=["..."])`.
- `app/api/main.py` — Registers all routers; `private` router only mounted when `ENVIRONMENT == "local"`.
- `app/api/deps.py` — Dependency injection types: `SessionDep`, `CurrentUser`, `get_current_active_superuser`.
- `app/core/config.py` — `pydantic-settings` `Settings` class reading from top-level `../.env`. Singleton: `settings`.
- `app/core/security.py` — Password hashing via `pwdlib` (Argon2 + Bcrypt), JWT via `pyjwt` (HS256). **Not passlib.**
- `app/core/db.py` — Engine creation + `init_db()` for seeding the first superuser.
- `app/alembic/` — Migration scripts. Run from `backend/` dir: `alembic revision --autogenerate -m "msg"`.

## Model Pattern

Every entity requires this hierarchy in `app/models.py`:

```python
class EntityBase(SQLModel):
    field: str = Field(min_length=1, max_length=255)

class EntityCreate(EntityBase):
    pass

class EntityUpdate(EntityBase):
    field: str | None = Field(default=None, ...)  # type: ignore

class Entity(EntityBase, table=True):
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    created_at: datetime | None = Field(
        default_factory=get_datetime_utc,
        sa_type=DateTime(timezone=True),  # type: ignore
    )

class EntityPublic(EntityBase):
    id: uuid.UUID
    created_at: datetime | None = None

class EntitiesPublic(SQLModel):
    data: list[EntityPublic]
    count: int
```

## CRUD Pattern

```python
def create_entity(*, session: Session, entity_in: EntityCreate, owner_id: uuid.UUID) -> Entity:
    db_obj = Entity.model_validate(entity_in, update={"owner_id": owner_id})
    session.add(db_obj)
    session.commit()
    session.refresh(db_obj)
    return db_obj
```

- Use `model_validate(..., update={})` for adding computed fields (hashed passwords, owner IDs).
- Use `sqlmodel_update()` for partial updates: `db_obj.sqlmodel_update(data_dict)`.
- Use `model_dump(exclude_unset=True)` when handling optional update fields.

## Route Pattern

```python
router = APIRouter(prefix="/entities", tags=["entities"])

@router.get("/", response_model=EntitiesPublic)
def read_entities(session: SessionDep, current_user: CurrentUser, skip: int = 0, limit: int = 100) -> Any:
    ...
```

- Admin-only routes: use `dependencies=[Depends(get_current_active_superuser)]`.
- Ownership checks: superusers see all, regular users filtered by `owner_id == current_user.id`.
- Register new routers in `app/api/main.py` via `api_router.include_router(module.router)`.

## Testing

- Tests run inside Docker: `docker compose exec backend bash scripts/test.sh`
- Fixtures in `tests/conftest.py`: `db` (session-scoped), `client` (module-scoped), `superuser_token_headers`, `normal_user_token_headers`.
- Test utilities in `tests/utils/`: `random_lower_string()`, `random_email()`, `create_random_user(db)`, `create_random_item(db)`.
- Tests use `TestClient` + real DB (not mocks). Auth via token headers dict: `{"Authorization": "Bearer ..."}`.
- Test files mirror route files: `tests/api/routes/test_items.py` ↔ `app/api/routes/items.py`.

## Linting & Types

- `ruff check` for linting (rules: E, W, F, I, B, C4, UP, ARG001, T201 — print statements forbidden).
- `mypy --strict` for type checking. Alembic dir is excluded.
- Use `# type: ignore` where SQLModel/Pydantic patterns conflict with mypy strict mode (see existing examples).

## Adding a New Entity

1. Add model hierarchy to `app/models.py`
2. Add CRUD functions to `app/crud.py`
3. Create `app/api/routes/entity.py` with router
4. Register router in `app/api/main.py`
5. Generate migration: `alembic revision --autogenerate -m "add entity"`
6. Add tests in `tests/api/routes/test_entity.py` + factory in `tests/utils/entity.py`
