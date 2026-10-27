# FastAPI Implementation Patterns

Use this reference when implementing HTTP endpoints, Pydantic schemas, services, repositories, lifecycle behavior, or errors.

## Application Lifecycle

Use FastAPI's lifespan mechanism for shared startup and shutdown work such as initializing the database and closing long-lived resources.

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from database import Database


@asynccontextmanager
async def lifespan(app: FastAPI):
    await Database.initialize()
    yield
    await Database.close()


app = FastAPI(lifespan=lifespan)
```

Do not perform expensive imports, network calls, or model loading at module import time unless the existing application deliberately requires it.

## Routers

Define routes with `APIRouter`, explicit response models, and HTTP-specific error handling. Keep the route focused on the HTTP boundary.

```python
from fastapi import APIRouter, HTTPException, status

from modules.items.repository import ItemRepository
from modules.items.schemas import ItemResponse

router = APIRouter(prefix="/items")


@router.get("/{item_id}", response_model=ItemResponse)
async def get_item(item_id: int) -> ItemResponse:
    item = await ItemRepository.get_by_id(item_id)
    if item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found",
        )
    return ItemResponse(**item)
```

Use dependency injection for request-scoped concerns such as authentication, authorization, or database sessions when the selected database design requires them.

## Schemas

Use separate request and response models when their contracts differ. Apply meaningful validation at the boundary and avoid exposing internal database rows by accident.

Define structured request bodies and public response models in the module's `schemas.py`, not inside endpoint functions or `router.py`. Simple path and query parameters may remain in endpoint signatures.

```python
from pydantic import BaseModel, Field


class CreateItemRequest(BaseModel):
    name: str = Field(min_length=1, max_length=120)


class ItemResponse(BaseModel):
    id: int
    name: str
```

Prefer precise field types and constraints over validation hidden inside route functions. Do not duplicate business rules across Pydantic validators and services; validation of shape belongs in schemas, while domain decisions belong in services.

## Services

Create a service when work spans repositories, calls an external system, implements business policy, or needs reuse outside one route. A trivial CRUD route may call its repository directly when a service would only forward the call.

For new projects, define services as classes with instance methods and pass dependencies through the constructor when needed. Do not mix instance services, static service methods, and module-level service functions within one project. Do not put SQL or HTTP-specific exceptions in services, and avoid introducing a dependency-injection framework without a demonstrated need.

Preserve an established project's service convention unless the user explicitly requests refactoring. Do not create a service that merely forwards every call to a repository.

## Repositories

Repositories own module-specific persistence. They return internal dictionary-like records rather than public API response schemas. Reuse one record type across methods that return the same meaningful data shape; do not create request and response classes for every repository method.

For new projects, define repositories as stateless classes with static methods. Keep this convention consistent across modules.

```python
from typing import TypedDict

from database import Database


class ItemRecord(TypedDict):
    id: int
    name: str


class ItemRepository:
    @staticmethod
    async def get_by_id(item_id: int) -> ItemRecord | None:
        row = await Database.fetch_one(
            "SELECT id, name FROM items WHERE id = ?",
            (item_id,),
        )
        if row is None:
            return None
        return ItemRecord(id=row["id"], name=row["name"])
```

Keep a small record type in `repository.py`. If a module later has many internal types shared across several files, move them only when that separation becomes useful.

Keep dynamic SQL tightly controlled. Bind user-provided values as parameters. If column names or sort directions must be dynamic, select them from an explicit allowlist rather than interpolating arbitrary input.

## Async Boundaries

- Use `async def` when awaiting database, filesystem, HTTP, or other asynchronous work.
- Do not call blocking libraries directly inside async routes. Use an async alternative, a thread offload, or an appropriate worker when needed.
- Do not make a function async when it performs only synchronous in-memory work.
- Keep transactions explicit when several writes must succeed or fail together.

## Errors and Logging

Represent expected client-facing failures with appropriate FastAPI/Starlette responses. Log unexpected operational failures with useful context, then preserve the application's established exception policy.

Do not leak credentials, tokens, personal data, raw database internals, or stack traces through public error responses.

## Verification

Verify behavior in proportion to the change by exercising the affected success path and meaningful failure paths such as validation failure, missing records, conflicts, or transaction rollback.
