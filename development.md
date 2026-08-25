# FastAPI Project - Development

## Local Development

For local development, run PostgreSQL and Mailpit with Docker Compose, and run the FastAPI and Vite development servers locally.

Start the supporting services:

```bash
docker compose up -d db mailpit
```

Then, from the `backend` directory, install the dependencies and prepare the database:

```bash
uv sync
uv run bash scripts/prestart.sh
```

Start the FastAPI development server:

```bash
uv run fastapi dev
```

In another terminal, from the project root, install the frontend dependencies and start the Vite development server:

```bash
bun install
bun run dev
```

Now you can open these URLs:

Frontend development server: <http://localhost:5173>

Backend API: <http://localhost:8000>

Automatic interactive API documentation with Swagger UI: <http://localhost:8000/docs>

Mailpit: <http://localhost:8025>

The frontend development server uses the backend at `http://localhost:8000`, as configured in `frontend/.env`.

### Frontend Served by FastAPI

Build the frontend from the `frontend` directory:

```bash
bun run build
```

The build is written to `backend/app/frontend` and served by FastAPI at <http://localhost:8000>. Rebuild the frontend after making frontend changes.

## Full Stack with Docker Compose

To run the backend and built frontend in Docker Compose:

```bash
docker compose run --rm backend bash scripts/prestart.sh
docker compose watch
```

Now you can open these URLs:

Application, with the frontend and API served by FastAPI: <http://localhost:8000>

Automatic interactive API documentation with Swagger UI: <http://localhost:8000/docs>

Adminer, database web administration: <http://localhost:8080>

Traefik UI, to see how the routes are being handled by the proxy: <http://localhost:8090>

Mailpit: <http://localhost:8025>

Stop a locally running FastAPI server before starting the Compose backend because both use port `8000`.

**Note**: The first time you start the stack, it might take a minute for all the services to be ready. To monitor it, use `docker compose logs`, or `docker compose logs backend` for the backend service.

## Mailpit

[Mailpit](https://mailpit.axllent.org) captures emails sent during local development instead of delivering them. The local backend connects to it at `localhost:1025`, and the Compose backend connects to the `mailpit` service. Captured emails are available at <http://localhost:8025>.

## Docker Compose Files and Environment Variables

The main `compose.yml` file contains the configuration shared by the whole stack. Docker Compose loads it automatically.

The `compose.override.yml` file adds local development settings, such as mounting the source code as a volume. Docker Compose also loads it automatically and applies it on top of `compose.yml`.

The `compose.deploy.yml` file contains the deployment-specific settings, including HTTPS and automatic certificate handling. It is explicitly combined with `compose.yml` when deploying the application.

The backend reads local settings from the `.env` file. Docker Compose also uses it for variable interpolation and passes the settings each container needs.

After changing variables, make sure you restart the stack:

```bash
docker compose watch
```

## The `.env` File

The tracked `.env` file contains local development defaults, passwords, and other configuration. Its hostnames use `localhost` for processes running on your machine. Docker Compose overrides hostnames such as the database and SMTP server with their Compose service names.

Do not store deployment secrets in `.env`. Configure them as described in the [FastAPI Cloud deployment guide](./deployment.md) or the [Docker Compose deployment guide](./deployment-docker-compose.md).

## How to Add a New Entity

The backend models live in `backend/app/models.py`. The project pattern is to define one SQLModel for the database table, a create/update schema, and a public response schema.

A typical entity follows this pattern:

```python
class WidgetBase(SQLModel):
    name: str = Field(min_length=1, max_length=255)


class WidgetCreate(WidgetBase):
    pass


class WidgetUpdate(SQLModel):
    name: str | None = Field(default=None, min_length=1, max_length=255)


class Widget(WidgetBase, table=True):
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    owner_id: uuid.UUID = Field(foreign_key="user.id", nullable=False, ondelete="CASCADE")
    owner: User | None = Relationship(back_populates="widgets")


class WidgetPublic(WidgetBase):
    id: uuid.UUID
    owner_id: uuid.UUID
```

When you add a new entity, also update the related code in the same order:

1. Add the SQLModel and relationships in `backend/app/models.py`.
2. Add any database queries or helper functions in `backend/app/crud.py`.
3. Add a route file in `backend/app/api/routes/`, for example `widgets.py`.
4. Register the router in `backend/app/api/main.py`.
5. Create and apply a database migration with Alembic under `backend/app/alembic/versions/`.
6. Add or update backend tests in `backend/tests/`.

If the entity should be visible in the generated frontend client, regenerate the client after the backend schema changes with:

```bash
bash ./scripts/generate-client.sh
```

## How to Add a New Route

The project uses file-based API routes in `backend/app/api/routes/` and file-based frontend routes in `frontend/src/routes/`.

### Backend API Route

Create a new file such as `backend/app/api/routes/widgets.py` and define a router similar to the existing `items` route:

```python
from fastapi import APIRouter, HTTPException

from app.api.deps import CurrentUser, SessionDep
from app.crud import create_widget
from app.models import Widget, WidgetCreate, WidgetPublic

router = APIRouter(prefix="/widgets", tags=["widgets"])


@router.get("/", response_model=list[WidgetPublic])
def read_widgets(session: SessionDep, current_user: CurrentUser) -> list[Widget]:
    return get_widgets(session=session, current_user=current_user)


@router.post("/", response_model=WidgetPublic)
def create_widget_route(
    *, session: SessionDep, current_user: CurrentUser, widget_in: WidgetCreate
) -> Widget:
    return create_widget(session=session, current_user=current_user, widget_in=widget_in)
```

Then include it in `backend/app/api/main.py`:

```python
from app.api.routes import widgets

api_router.include_router(widgets.router)
```

The route should use `SessionDep` for the DB session, `CurrentUser` for auth checks, and a `response_model` so FastAPI serializes the data consistently.

### Frontend Route

The frontend uses `@tanstack/react-router` with file-based routes under `frontend/src/routes/`. To add a page or route, create a file matching the route structure, for example `frontend/src/routes/_layout/widgets.tsx`.

The route file should follow the same pattern as the existing item pages:

```tsx
import { createFileRoute } from "@tanstack/react-router"

export const Route = createFileRoute("/_layout/widgets")({
  component: Widgets,
})

function Widgets() {
  return <div>Widgets</div>
}
```

If the page needs data from the API, use the generated client and `useSuspenseQuery` or `useQuery` the same way as `frontend/src/routes/_layout/items.tsx` does.

For backend changes that affect the OpenAPI spec, regenerate the frontend client before using new route data:

```bash
bash ./scripts/generate-client.sh
```

## Pre-commit Hooks and Code Linting

The project uses [prek](https://prek.j178.dev/), a modern alternative to [pre-commit](https://pre-commit.com/), for code linting and formatting.

You can find a file `.pre-commit-config.yaml` with configurations at the root of the project.

### Install `prek` to Run Automatically

`prek` is already part of the dependencies of the project.

From the project root, install the Git hook so that `prek` runs automatically before each commit:

```bash
uv run prek install -f
```

The `-f` flag forces the installation, in case there was already a `pre-commit` hook previously installed.

Now whenever you try to commit, for example with:

```bash
git commit
```

`prek` will check and format the code you are about to commit. If it modifies any files, add those files to Git again before committing.

### Run `prek` Manually

You can also run `prek` manually on all files from the project root:

```bash
uv run prek run --all-files
```
