# Development Guide

## COMMANDS

```bash
Run formatter: docker compose run --rm app bin/cli fix-style
Run test: docker compose run --rm app pytest -q --show-capture=log --disable-warnings tests/test_file.py
Run all tests and format and linter validations: docker compose run --rm app pytest -q --show-capture=log --disable-warnings
Run typing validations: docker compose run --rm app bin/cli test-typing
See logs: docker compose logs -f app
Update OpenAPI specs: docker compose run --rm app pandora export-openapi-spec
```

## RECOMMENDATIONS

### Async vs Sync

```
Database operations + AsyncSession → async def + await
External API calls + async client → async def + await
File I/O + aiofiles → async def + await
CPU-bound work + ThreadPoolExecutor → def (sync)
Simple data processing → def (sync)
SQLAlchemy Session (not AsyncSession) → def (sync, no await)
```

### File Structure

```
File > 500 lines → Split into smaller files
Function > 50 lines → Split into smaller functions
New feature domain → New module/package
Bug fix/enhancement → Edit existing file
```

## CODE TEMPLATES

### Python 3.12

```python
# Type aliases
type UserId = int
type UserDict = dict[str, Any]

# Simplified generics
class Repository[T]:
    def __init__(self, model: type[T]) -> None:
        self.model = model

# f-string debugging
log.debug(f"{variable=}")

# Exception groups
try:
    operations()
except* ValueError as eg:
    for error in eg.exceptions:
        handle_value_error(error)
```

### Pydantic v2

```python
from pydantic import BaseModel, Field, field_validator, computed_field, model_serializer

class ModelTemplate(BaseModel):
    name: str = Field(..., min_length=1, description="Description")
    email: str = Field(..., pattern=r'^[^@]+@[^@]+\.[^@]+$')
    age: int | None = Field(default=None, ge=0, le=150)
    type: str = Field(discriminator='type')  # For polymorphic models
  
    @field_validator('email', mode='after')
    @classmethod
    def validate_email(cls, v: str) -> str:
        return v.lower()
  
    @computed_field
    @property
    def display_name(self) -> str:
        return f"{self.name} <{self.email}>"
  
    @model_serializer
    def serialize_model(self) -> dict[str, Any]:
        return {"custom": "serialization"}
  
    model_config = {
        "json_schema_extra": {"examples": [{"name": "Example", "email": "user@example.com"}]},
        "json_schema_serialization_defaults_required": True
    }
```

### SQLAlchemy 2.0

Do not use explicit commit and rollback if the application uses a transaction manager.

```python
from sqlalchemy import select, func
from sqlalchemy.orm import selectinload, joinedload
from sqlalchemy.ext.hybrid import hybrid_property

# Query with explicit relationship loading
async def get_entity_with_relations(session: AsyncSession, entity_id: int):
    stmt = select(Entity).options(
        selectinload(Entity.one_to_many_rel),
        joinedload(Entity.one_to_one_rel)
    ).where(Entity.id == entity_id)
  
    result = await session.execute(stmt)
    return result.scalar_one_or_none()

# Bulk operations
async def bulk_create(session: AsyncSession, entities: list[dict]):
    await session.bulk_insert_mappings(Entity, entities)
    await session.commit()

# Hybrid properties
class Entity(Base):
    first_name: Mapped[str]
    last_name: Mapped[str]
  
    @hybrid_property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"
  
    @full_name.expression
    @classmethod
    def full_name(cls):
        return func.concat(cls.first_name, ' ', cls.last_name)

# Connection pooling configuration
engine = create_async_engine(
    url,
    pool_size=20,
    max_overflow=30,
    pool_recycle=3600
)
```

### FastAPI

```python
from fastapi import APIRouter, Depends, BackgroundTasks, HTTPException, status
from contextlib import asynccontextmanager

# Dependency injection with proper scoping
class SingletonService:
    def __init__(self): pass

singleton_service = SingletonService()

def get_singleton() -> SingletonService:
    return singleton_service

@asynccontextmanager
async def get_async_session():
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()

# API versioning
v1_router = APIRouter(prefix="/api/v1", tags=["v1"])
v2_router = APIRouter(prefix="/api/v2", tags=["v2"])

# Background tasks
@app.post("/process/")
async def process_data(
    data: RequestModel,
    background_tasks: BackgroundTasks,
    service: Service = Depends(get_service)
):
    background_tasks.add_task(heavy_processing, data)
    return ResponseModel(status="processing", id=data.id)

# Middleware order (CRITICAL)
app.add_middleware(CORSMiddleware)
app.add_middleware(AuthenticationMiddleware)
app.add_middleware(LoggingMiddleware)
app.add_middleware(ErrorHandlingMiddleware)

# Exception handling
class BusinessError(Exception):
    def __init__(self, message: str, details: dict | None = None):
        self.message = message
        self.details = details or {}
        super().__init__(message)

@app.exception_handler(BusinessError)
async def business_error_handler(request, exc: BusinessError):
    raise HTTPException(
        status_code=status.HTTP_400_BAD_REQUEST,
        detail={"message": exc.message, "details": exc.details}
    )
```

## PATTERNS

### Manager Pattern

```python
from dataclasses import dataclass
from sqlalchemy.ext.asyncio import AsyncSession

@dataclass(slots=True, frozen=True)
class EntityManager:
    session: AsyncSession
  
    async def create_entity(self, data: dict) -> Entity:
        if not self._is_valid_data(data):
            raise ValueError("Invalid data")
  
        entity = Entity(**data)
        self.session.add(entity)
        await self.session.commit()
        return entity
  
    def _is_valid_data(self, data: dict) -> bool:  # Private method
        return bool(data.get("required_field"))

# In routes (keep thin)
@router.post("/entities/")
async def create_entity(
    request: EntityCreateRequest,
    manager: EntityManager = Depends(get_entity_manager)
) -> EntityResponse:
    return await manager.create_entity(request.model_dump())
```

### Error Handling Pattern

```python
async def process_entity(entity_id: int, session: AsyncSession) -> Entity:
    # Guard clauses first
    if entity_id <= 0:
        raise ValueError("Entity ID must be positive")
  
    entity = await session.get(Entity, entity_id)
    if not entity:
        raise NotFoundError(f"Entity {entity_id} not found")
  
    if not entity.is_active:
        raise ValidationError("Entity is inactive")
  
    # Happy path last
    return await process_active_entity(entity)
```

### Factory Pattern

```python
import factory
from factory.alchemy import SQLAlchemyModelFactory

class EntityFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Entity
        sqlalchemy_session = Session
        sqlalchemy_session_persistence = "flush"
        sqlalchemy_get_or_create = ("unique_field",)
  
    name = factory.Faker("name")
    email = factory.Sequence(lambda n: f"user{n}@example.com")
    related_entity = factory.SubFactory(RelatedEntityFactory)
  
    @factory.post_generation
    def many_to_many_rel(self, create, extracted, **kwargs):
        if not create or not extracted:
            return
        for item in extracted:
            self.many_to_many_rel.append(item)
```

## RULES

### Method Visibility

- Public methods: Minimal, stable interface, well-documented
- Private methods: `_method_name`, implementation details only
- When making private: Remove direct tests, test through public interface

### Type Hints

- Use proper type hints for all function parameters and returns
- Use `| None` instead of Optional
- Use `list[T]` instead of `List[T]`
- Use `type` statement for aliases: `type UserId = int`

### Model Separation

- Always separate request/response models from entity models
- Never reuse entity models for API contracts
- Use specific models for each API version
