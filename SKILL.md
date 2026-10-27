---
name: backend
description: Design, create, refactor, or extend maintainable Python backends using the ASAAS modular FastAPI architecture. Use for new backend projects, API features, database-backed services, backend structure reviews, or refactoring an existing backend into this architecture when the user explicitly requests it; default to SQLite unless the user or existing project requires another database.
---

# Backend Foundation

## Goal

Build simple, maintainable FastAPI backends organized around cohesive entities or business capabilities, with clear boundaries between HTTP handling, business logic, data validation, and persistence.

## Start With Context

Before proposing or changing code:

1. Read the repository instructions and existing documentation.
2. Inspect the current project structure, dependencies, and representative files.
3. Preserve established conventions when working in an existing project unless the user explicitly asks to change them.
4. Confirm the requested scope before introducing new architecture, infrastructure, or deployment workflows.

For a new project or an unstructured backend, use the defaults in this skill.

## New Project Intake

Before designing or scaffolding a new backend, make sure the user has described what the application does. Establish enough product context to make sound backend decisions, including:

- The application's purpose and intended users.
- Its core workflows or features.
- The main data the application needs to store.
- Authentication and user-role requirements.
- Required external services or integrations.
- Known hosting, deployment, or operational constraints.

Ask focused questions for any missing information that would materially change the architecture, data model, API contract, security, or implementation scope. Do not force the user to answer irrelevant questions or delay a small prototype for decisions that can safely remain open. When proceeding with non-material details unspecified, state the assumptions clearly and keep them easy to change.

## Default Stack

- Python with FastAPI
- Pydantic models for request and response validation
- Async endpoints and async I/O where the underlying work is asynchronous
- SQLite with `aiosqlite`
- Parameterized raw SQL through a central database wrapper
- Uvicorn as the application server
- `pydantic-settings` or an equivalent environment-based settings layer
- Standard Python `logging`

SQLite is the default, not a permanent constraint. Use another database, driver, or ORM when the user requests it or the existing project already uses it. Preserve the same module boundaries when changing persistence technology.

## Architecture

Organize application code by cohesive entity or business capability under `modules/`. A module may contain:

- `router.py`: HTTP routing, dependency injection, request validation, response selection, and HTTP-specific errors.
- `schemas.py`: Pydantic request and response contracts for the module's API boundary.
- `service.py`: Business rules and multi-step workflows.
- `repository.py`: Module-specific database queries and persistence mapping.

Not every module needs every file. A simple CRUD module may use a router, schemas, and a repository without a service when the service would only forward calls.

Keep shared project responsibilities outside the modules:

- `migrations/`: Ordered, versioned schema changes.
- `utils/`: Small, genuinely shared technical helpers.
- `data/`: Runtime SQLite data such as `data/app.db`.
- `main.py`: Application creation, lifespan, middleware, and router registration.
- `config.py`: Environment-driven settings.
- `database.py`: Connection management and common query helpers.

Keep dependencies flowing inward: routers call services or repositories, services call repositories, and repositories own module-specific persistence. Do not put HTTP concerns in services or repositories, and do not put SQL in routers or services. Repositories return internal dictionary-like records rather than public API response schemas.

Read [references/project-structure.md](references/project-structure.md) when creating a project, deciding where new files belong, or splitting a growing module.

## Implementation Rules

- Keep routers thin. Validate inputs, call the relevant service or repository, and translate expected failures into HTTP responses.
- Define structured request bodies and public response models in the module's `schemas.py`, not inside route functions or `router.py`. Simple path and query parameters may remain in endpoint signatures.
- Put reusable business behavior in services rather than route functions.
- Keep SQL inside repositories and always bind values as parameters.
- Define explicit response models for public endpoints.
- For new projects, use repository classes with static methods and service classes with instance methods. Do not mix static services, instance services, and module-level service functions within one project. Preserve an established project's convention unless the user explicitly requests refactoring.
- Use a logger created with `logging.getLogger(__name__)`; do not use `print()` for application diagnostics.
- Keep configuration outside source code and never commit secrets or environment-specific credentials.
- Prefer the smallest design that satisfies the current requirements. Do not add abstractions for hypothetical scale.
- Use `router.py`, `schemas.py`, `service.py`, and `repository.py` as the default module structure. A module may omit a file when that responsibility does not exist.
- Do not add other module-local files or folders by default. If another file would provide a meaningful architectural benefit, explain and recommend it to the user before adding it.
- Do not add files for minor organizational improvements. When an additional file is justified, prefer a purpose-specific name over a catch-all name such as `utils.py`.
- Start a module with a small set of files. Split `service.py` or `repository.py` into subfolders only when it contains several distinct responsibilities.
- Treat a cohesive business capability or substantial technical capability as a module; a module may own no tables, one table, or several related tables.
- Keep a capability inside its owning module when only that module uses it. Promote it to a top-level module when multiple modules use it or it has substantial independent behavior and evolution.
- Allow a module to contain submodules when it owns several substantial, independently understandable capabilities with their own routes, schemas, rules, or persistence behavior.
- Keep a helper inside its module unless it is technical, domain-independent, and used by multiple modules.

Read [references/fastapi-patterns.md](references/fastapi-patterns.md) when implementing endpoints, schemas, services, repositories, lifecycle behavior, or error handling.

Read [references/ai-heavy-projects.md](references/ai-heavy-projects.md) only when AI or machine learning is central to the product and the backend contains multiple model-driven components, providers, pipelines, or experimentation workflows. Do not read it for one isolated AI feature or a simple model API call.

## Database and Migrations

For the default SQLite implementation:

- Use a central async database wrapper.
- Store the default SQLite database at `data/app.db` unless the user or deployment environment requires another path.
- Return dictionary-like rows from query helpers.
- Provide common `execute`, `fetch_one`, `fetch_all`, and transaction helpers as needed.
- Enable appropriate SQLite safety and concurrency settings, such as foreign keys, WAL mode, and a busy timeout, based on the application's needs.
- Apply every schema change through an ordered migration. Do not rely on manually changing a deployed database.
- Keep fresh-install schema behavior and upgrade behavior consistent.

Recommend PostgreSQL when actual requirements include multiple application instances writing concurrently, sustained write concurrency, several services sharing one database, managed high availability or replication, PostgreSQL-specific features, or hosting without reliable persistent local storage. Do not recommend PostgreSQL merely because a project may grow or uses AI.

Read [references/database-and-migrations.md](references/database-and-migrations.md) whenever creating or changing persistence, tables, indexes, transactions, or migrations.

## Workflow

1. Establish the requested behavior and affected modules.
2. Inspect existing patterns before choosing filenames or interfaces.
3. Define or update the affected module's schemas and persistence behavior.
4. Implement repository behavior, service orchestration when needed, and then the router boundary.
5. Register routers and lifecycle hooks only when required.
6. Verify the smallest meaningful path through the change, including failure behavior relevant to the request.
7. Update existing implementation documentation after the behavior is verified, clearly distinguishing completed behavior from proposals.

Do not create a test suite by default for a new or small project. When a project has grown substantially and repeated development makes regression risk meaningful, ask the user whether they want to introduce testing before adding a framework, test structure, or test files. Preserve and use existing tests when working in a project that already has them.

## Boundaries

- Do not introduce an ORM, message broker, cache, container setup, CI/CD pipeline, cloud service, or new architectural layer without a requirement.
- Do not copy project-specific business logic into a reusable backend.
- Do not reorganize a working repository merely to match this preferred structure.
- Do not commit, push, deploy, or modify external systems unless explicitly authorized.
