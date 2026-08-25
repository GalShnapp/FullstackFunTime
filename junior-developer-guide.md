# Junior Developer Guide

This guide explains the usual workflow for extending this application and how
the pieces fit together. For local setup, see [Development](./development.md).

## How to add an entity

In this guide, an **entity** is a piece of data that is stored in the
database and exposed through the API and the user interface. Use `Item` as a
working example when following the existing code.

### 1. Add the backend models

Add the entity's SQLModel classes to
[`backend/app/models.py`](./backend/app/models.py). Keep database and API
concerns separate:

* `ThingBase` contains fields shared by requests and responses.
* `ThingCreate` describes fields accepted when creating a record.
* `ThingUpdate` makes editable fields optional.
* `Thing` has `table=True` and contains the database fields, primary key, and
  relationships.
* `ThingPublic` and `ThingsPublic` describe API responses.

Add validation with SQLModel/Pydantic fields (for example, `min_length` and
`max_length`). If the entity belongs to a user, add an `owner_id` foreign key
and a relationship, and enforce ownership in every endpoint.

### 2. Add database migration

From `backend/`, create and apply a migration after changing a database model:

```bash
uv run alembic revision --autogenerate -m "Add things"
uv run alembic upgrade head
```

Review the generated file under
[`backend/app/alembic/versions/`](./backend/app/alembic/versions/) before
committing it. Migrations are the source of truth for existing databases;
changing the model alone does not change the database schema.

### 3. Add CRUD and API routes

Put reusable database operations in
[`backend/app/crud.py`](./backend/app/crud.py), then add a router under
[`backend/app/api/routes/`](./backend/app/api/routes/). Follow `items.py` for
the normal list, detail, create, update, and delete patterns:

* inject `SessionDep` and `CurrentUser`;
* validate input with the `ThingCreate`/`ThingUpdate` models;
* return `ThingPublic` response models;
* check that a user may read or modify the requested record.

Include the router in
[`backend/app/api/main.py`](./backend/app/api/main.py). FastAPI uses these
routes to build the OpenAPI schema.

### 4. Generate the frontend client

With the backend available, run this command from the repository root:

```bash
bash ./scripts/generate-client.sh
```

The script exports the backend's OpenAPI schema and regenerates
[`frontend/src/client/`](./frontend/src/client/). Do not edit generated files
by hand. Regenerate them whenever an API model or endpoint changes.

### 5. Add the frontend page

Use the generated operations and types in a component under
[`frontend/src/components/`](./frontend/src/components/). Add a route or page
under [`frontend/src/routes/`](./frontend/src/routes/) and add navigation in
the sidebar when the page should be visible there. Follow the existing `Items`
components for forms, tables, loading states, and error handling.

Add or update backend tests in `backend/tests/` and end-to-end tests in
`frontend/tests/`. Test validation, permissions, and the main
create/read/update/delete flows.

## How to build

### Development build

Start PostgreSQL and Mailpit, prepare the database, and run the two development
servers:

```bash
docker compose up -d db mailpit
cd backend
uv sync
uv run bash scripts/prestart.sh
uv run fastapi dev
```

In another terminal, from the repository root:

```bash
bun install
bun run dev
```

Vite serves the frontend at <http://localhost:5173> and proxies API requests
to FastAPI at <http://localhost:8000>. Swagger UI is available at
<http://localhost:8000/docs>.

### Production-style frontend build

From `frontend/`, run:

```bash
bun run build
```

This runs TypeScript checks and Vite, writing the built frontend to
`backend/app/frontend`. FastAPI serves those files from the same origin as the
API at <http://localhost:8000>. Rebuild after frontend changes.

### Build and run the complete stack

Docker Compose builds the backend image, which includes the frontend build:

```bash
docker compose build
docker compose run --rm backend bash scripts/prestart.sh
docker compose up -d --wait
```

The application is then available at <http://localhost:8000>. Use
`docker compose logs backend` if a service is still starting.

## How the tech stack relates to the system

```text
Browser
  │ React + TypeScript + TanStack Router/Query + Tailwind/shadcn/ui
  │ generated OpenAPI client
  ▼
FastAPI application
  │ routes, authentication, validation, and OpenAPI documentation
  ▼
SQLModel / SQLAlchemy  ── Alembic migrations
  │
  ▼
PostgreSQL
```

* **React** renders pages and components in the browser. **TanStack Router**
  selects the page, while **TanStack Query** loads and caches API data.
  **Tailwind CSS** and **shadcn/ui** provide styling and reusable controls.
* **FastAPI** receives HTTP requests, applies dependencies such as the current
  user and database session, validates request data, and returns responses.
  It also publishes the OpenAPI schema.
* **The generated OpenAPI client** turns that schema into typed frontend
  functions, so backend API changes can be used without manually duplicating
  request and response types in TypeScript.
* **SQLModel** defines Python types and database tables together, using
  SQLAlchemy to talk to PostgreSQL. **Alembic** records schema changes as
  repeatable migrations.
* **JWT authentication** identifies users on API requests. The backend's
  authorization checks decide which records each user may access.
* **Docker Compose** runs PostgreSQL, Mailpit, the backend, and supporting
  services consistently. **Traefik** is the reverse proxy in the Compose
  deployment configuration.
* **Mailpit** catches development emails, while **React Email** supplies the
  email templates rendered and sent by the backend.
* **Pytest** tests backend behavior, **Playwright** tests the complete browser
  flow, and **GitHub Actions** runs these checks and deployment workflows.
