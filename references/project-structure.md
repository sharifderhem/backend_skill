# Project Structure

Use this reference when starting a backend, placing a new module, or deciding whether a growing module needs to be split.

## Baseline Layout

```text
project/
|-- main.py
|-- config.py
|-- database.py
|-- requirements.txt
|-- README.md
|-- modules/
|   |-- __init__.py
|   |-- users/
|   |   |-- __init__.py
|   |   |-- router.py
|   |   |-- schemas.py
|   |   |-- service.py
|   |   `-- repository.py
|   `-- orders/
|       |-- __init__.py
|       |-- router.py
|       |-- schemas.py
|       |-- service.py
|       `-- repository.py
|-- migrations/
|   |-- __init__.py
|   `-- migrations.py
|-- utils/
|   `-- __init__.py
`-- data/
    `-- app.db
```

Only create files and directories the project actually needs. A small CRUD module may require a router, schemas, and a repository but no service. Add a service when business rules, orchestration, or reuse justify it.

## Responsibilities

### Application root

- `main.py` creates the FastAPI application, owns the lifespan, configures middleware, and registers routers.
- `config.py` defines typed settings loaded from environment variables or approved configuration sources.
- `database.py` owns connections and generic query helpers. It does not contain feature-specific queries.
- Dependency declarations should distinguish required runtime packages from optional or development-only packages when the project tooling supports that separation.

### Modules

Each module represents a cohesive entity or business capability. Keep its router, schemas, service, and repository together so the module can be understood and changed without navigating several global layer folders.

- `router.py` owns HTTP routes and HTTP-specific behavior. It must not contain SQL or long business workflows.
- `schemas.py` owns the module's public request and response contracts.
- `service.py` owns business rules, multi-step workflows, and coordination. Do not create it when it would only forward repository calls.
- `repository.py` owns module-specific queries and returns internal dictionary-like records, not public API response schemas.

Not every module needs every file.

Use `router.py`, `schemas.py`, `service.py`, and `repository.py` as the default module structure. A module may omit a file when that responsibility does not exist. Do not add other files or folders by default.

If an additional module-local file would provide a meaningful architectural benefit, explain and recommend it to the user before adding it. Do not add files for minor organizational improvements. When an additional file is justified, prefer a purpose-specific name such as `pricing.py` or `file_storage.py` over a catch-all name such as `utils.py`.

### Utilities

Keep a helper inside its module when only that module uses it. Move it to `utils/` only when it is technical, domain-independent, and used by multiple modules. Prefer purpose-specific files such as `datetime.py`, `identifiers.py`, or `pagination.py` over catch-all files such as `helpers.py`, `common.py`, or `misc.py`.

### Runtime data

For the default SQLite setup, store the runtime database at `data/app.db`. Migrations remain in `migrations/`; runtime data and schema-change history have different responsibilities.

## Choosing a Module Boundary

A module represents a cohesive business capability or a substantial technical capability. It does not need to correspond to a database table; one module may own no tables, one table, or several related tables.

Something is a good module when it has a clear responsibility, meaningful rules or workflow, identifiable inputs and outputs, and can usually be changed within its own folder. Do not create a module only because a table, endpoint, helper, or library wrapper exists.

Keep a supporting capability inside its owning module when only that module uses it. Promote it to a top-level module when multiple modules use it or when it has substantial independent behavior and evolution. Prefer names based on purpose, such as `image_processing`, over names based only on implementation technology.

## Growing a Module

Begin with a small set of files:

```text
billing/
|-- router.py
|-- schemas.py
|-- service.py
`-- repository.py
```

When one responsibility grows into several distinct areas, split only that responsibility:

```text
billing/
|-- router.py
|-- schemas.py
|-- services/
|   |-- invoices.py
|   `-- refunds.py
`-- repositories/
    |-- invoices.py
    `-- payments.py
```

A module may contain submodules when it owns several substantial capabilities that have their own routes, schemas, rules, or persistence behavior:

```text
authentication/
|-- login/
|   |-- router.py
|   |-- schemas.py
|   |-- service.py
|   `-- repository.py
`-- password_reset/
    |-- router.py
    |-- schemas.py
    |-- service.py
    `-- repository.py
```

Prefer one module level. Avoid more than two levels unless the domain genuinely requires it. Do not create a submodule for one endpoint, helper, table, or slightly long file, and do not create service and repository folders before they are useful.

## Optional Project Files

Add a `Dockerfile`, Compose configuration, static-file directories, background workers, or deployment files only when the product or delivery environment needs them. These are project choices, not baseline requirements of the architecture.
