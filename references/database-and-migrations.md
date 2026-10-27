# Database and Migrations

Use this reference when creating or changing the database wrapper, repositories, transactions, tables, indexes, or migration behavior.

## Database Selection

Default to SQLite with `aiosqlite` for a new project unless requirements indicate otherwise. SQLite is normally appropriate when the application runs as one instance, write concurrency is moderate, persistent local storage is available, operational simplicity is valuable, and PostgreSQL-specific features are unnecessary.

Recommend PostgreSQL when actual requirements include multiple application instances writing concurrently, sustained write concurrency, several services sharing one database, managed high availability or replication, PostgreSQL-specific querying or locking, or hosting without reliable persistent local storage. Explain the concrete reason for the recommendation. Do not recommend PostgreSQL merely because a project may grow or uses AI.

Preserve the existing database technology when modifying an established project unless the user requests a migration. Changing the database must not collapse module boundaries. Module-specific queries remain in repositories even when the driver, placeholder syntax, connection management, or migration tooling changes.

## SQLite Data Location

Use `data/app.db` as the default SQLite database path unless the user or deployment environment requires another location:

```env
DATABASE_PATH=./data/app.db
```

Keep runtime database files in `data/` and schema-change history in `migrations/`.

## Central SQLite Wrapper

A lightweight wrapper should own the configured database path, connection creation, row configuration, and generic helpers. It must not contain module-specific queries.

```python
from contextlib import asynccontextmanager
from typing import Any, AsyncIterator

import aiosqlite

from config import settings


class Database:
    _path = settings.database_path

    @classmethod
    @asynccontextmanager
    async def connection(cls) -> AsyncIterator[aiosqlite.Connection]:
        async with aiosqlite.connect(cls._path) as connection:
            connection.row_factory = aiosqlite.Row
            await connection.execute("PRAGMA foreign_keys = ON")
            yield connection

    @classmethod
    async def execute(cls, query: str, params: tuple[Any, ...] = ()) -> None:
        async with cls.connection() as connection:
            await connection.execute(query, params)
            await connection.commit()

    @classmethod
    async def fetch_one(
        cls,
        query: str,
        params: tuple[Any, ...] = (),
    ) -> dict[str, Any] | None:
        async with cls.connection() as connection:
            cursor = await connection.execute(query, params)
            row = await cursor.fetchone()
            return dict(row) if row else None

    @classmethod
    async def fetch_all(
        cls,
        query: str,
        params: tuple[Any, ...] = (),
    ) -> list[dict[str, Any]]:
        async with cls.connection() as connection:
            cursor = await connection.execute(query, params)
            rows = await cursor.fetchall()
            return [dict(row) for row in rows]
```

Adapt helpers to actual needs. Add `execute_many`, transaction support, or returned identifiers only when callers require them.

## SQLite Configuration

Consider these settings during initialization:

- `PRAGMA foreign_keys = ON` to enforce declared relationships on every connection.
- `PRAGMA journal_mode = WAL` when the application's read/write pattern benefits from it.
- `PRAGMA busy_timeout` to tolerate short lock contention.
- `PRAGMA synchronous = NORMAL` with WAL when its durability tradeoff is acceptable.

Choose settings deliberately and document meaningful tradeoffs. A pragma applied only to one connection may not affect later connections, so distinguish persistent settings from per-connection settings.

## Query Safety

- Bind all values using the driver's placeholders.
- Never interpolate untrusted input into SQL.
- Allowlist any dynamic identifiers or sort directions.
- Select explicit columns for stable mappings instead of relying on `SELECT *`.
- Use a transaction for related writes that form one business operation.
- Add indexes based on observed query patterns and constraints, not speculation.

## Migration Model

Use monotonically increasing versions. Each migration should declare its version, describe one coherent schema change, and provide an async `up` operation.

```python
class Migration:
    version: int
    description: str

    @staticmethod
    async def up(connection) -> None:
        raise NotImplementedError


class Migration001(Migration):
    version = 1
    description = "Create items table"

    @staticmethod
    async def up(connection) -> None:
        await connection.execute(
            """
            CREATE TABLE items (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
            )
            """
        )
```

Track the applied version in a dedicated migration table or SQLite's `PRAGMA user_version`. Apply pending migrations in order inside controlled transactions where supported.

## Fresh Installs and Upgrades

Both paths must produce compatible schemas:

- A fresh database should receive the current baseline plus every migration not included in that baseline.
- An existing database should receive only migrations newer than its recorded version.
- Never reuse an applied migration version.
- Do not edit the meaning of a migration that may already have run; add a new migration instead.
- Seed data only when the application genuinely requires it, and make the behavior safe for reruns when appropriate.

Avoid maintaining a baseline schema and migrations that silently drift apart. When squashing old migrations, document the baseline version and verify both a clean installation and an upgrade from the oldest supported version.

## Schema-Change Workflow

1. Identify the current schema version and supported upgrade paths.
2. Add the next migration without renumbering existing migrations.
3. Update fresh-install behavior if the project maintains a separate baseline.
4. Update repository queries and schema mappings affected by the change.
5. Verify a clean database and an upgrade database when both paths are supported.
6. Document the completed change in the project's existing implementation documentation.
