# Lab 1: Containerised Web Mapping Environment and Hello Map

## CMPU4058 - Advanced Web Mapping - Week 1

**Duration:** 3 hours  
**Delivery:** Docker Compose development environment  
**Services:** Django, PostgreSQL/PostGIS, pgAdmin

### Overview

This lab establishes the repeatable environment used throughout Weeks 1-11. Instead of installing PostgreSQL, PostGIS, or Python packages directly on the host computer, you will run the application stack in three containers:
```
        Browser[Browser] -->|http://localhost:8000| Web[Django container]
        Web -->|Docker network| Database[PostgreSQL + PostGIS container]
        Browser -->|http://localhost:5050| Admin[pgAdmin container]
        Admin -->|Docker network| Database
```

By the end of the lab, you will have a Django "Hello Map" site centred on Dublin, a PostGIS-enabled database, and pgAdmin for inspecting the database visually. The same `compose.yaml` pattern will be extended in later weeks.

### Learning Objectives

By the end of this lab, you will be able to:

1.  Run a multi-container web mapping environment with Docker Compose.
2.  Explain the role of the Django, PostGIS, and pgAdmin services.
3.  Configure Django to connect to PostGIS using environment variables.
4.  Create and migrate a GeoDjango application.
5.  Inspect PostGIS and application tables through pgAdmin.
6.  Serve an interactive Leaflet map centred on Dublin.
7.  Diagnose basic container, database, and application failures using logs and health checks.

### Scenario

You are building a proof-of-concept map for the Dublin Tourism Board. The first deliverable is a small but real web application that displays a map of Dublin and proves it can store spatial data. Your team needs a setup every developer can start with the same commands, regardless of whether they use macOS, Windows, or Linux.

### Prerequisites

Install the following on your computer:

*   Docker Desktop, running and configured to use Docker Compose v2.
*   VS Code or another editor.
*   Git (recommended).
*   A modern web browser.

Verify Docker before continuing:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

You do not need a local PostgreSQL, PostGIS, pgAdmin, Python, or virtual-environment installation for this lab.

## Part 1: Create the Project and Container Configuration (45 minutes)

Create a project folder and enter it:

```bash
mkdir hello-map-dublin
cd hello-map-dublin
```

Create this initial structure. Files marked `you will create` are completed in the following steps.

    hello-map-dublin/
    ├── compose.yaml                 # you will create
    ├── .env                         # local configuration; do not commit
    ├── .env.example                 # safe template to commit
    ├── .gitignore
    ├── requirements.txt
    ├── Dockerfile                   # you will create
    ├── manage.py                    # created by Django
    ├── hello_map/                   # created by Django
    └── mapping/                     # created by Django
    

### 1.1 Application Dependencies

Create `requirements.txt`:

```text
Django>=5.0,<6.0
psycopg[binary]>=3.1
django-environ>=0.11
```

`psycopg` is the PostgreSQL driver used by Django. GeoDjango support comes from Django and the geospatial libraries installed in the application image below.

### 1.2 Django Dockerfile

Create `Dockerfile` in the project root:

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update \
    && apt-get install --yes --no-install-recommends \
        binutils \
        gdal-bin \
        libgdal-dev \
        libgeos-dev \
        libproj-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

### 1.3 Environment Variables

Create `.env.example`:

```bash
POSTGRES_DB=hello_map_dublin
POSTGRES_USER=awm_developer
POSTGRES_PASSWORD=<your password>
DATABASE_HOST=db
DATABASE_PORT=5432
PGADMIN_DEFAULT_EMAIL=<your_email@tudublin.ie>
PGADMIN_DEFAULT_PASSWORD=<your_password>
DJANGO_SECRET_KEY=replace-with-a-long-random-development-key
DJANGO_DEBUG=True
```

Copy it to `.env` and replace both password placeholders. `.env` is for your computer only.

```bash
cp .env.example .env
```

Create `.gitignore`:

```text
.env
__pycache__/
*.py[cod]
staticfiles/
.DS_Store
```

### 1.4 Docker Compose Services

Create `compose.yaml`:

```yaml
services:
  db:
    image: postgis/postgis:16-3.4
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgis_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

  pgadmin:
    image: dpage/pgadmin4:9
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_DEFAULT_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_DEFAULT_PASSWORD}
    ports:
      - "5050:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    depends_on:
      db:
        condition: service_healthy

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    environment:
      DATABASE_NAME: ${POSTGRES_DB}
      DATABASE_USER: ${POSTGRES_USER}
      DATABASE_PASSWORD: ${POSTGRES_PASSWORD}
      DATABASE_HOST: ${DATABASE_HOST}
      DATABASE_PORT: ${DATABASE_PORT}
      SECRET_KEY: ${DJANGO_SECRET_KEY}
      DEBUG: ${DJANGO_DEBUG}
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgis_data:
  pgadmin_data:
```

Compose creates a private network automatically. Within that network, Django and pgAdmin reach PostgreSQL at `db:5432`; `localhost` would incorrectly refer to the container itself. The database has no host port because use of pgAdmin and `docker compose exec` is sufficient for this lab.

Start and build the environment:

```bash
docker compose up --build -d
docker compose ps
```

Expected result: `db` is `healthy`; `pgadmin` is running; `web` may exit at this point because the Django project has not yet been created. That is expected.

## Part 2: Create and Configure Django (45 minutes)

Run Django commands inside the `web` image. `--rm` starts a one-off container and removes it when the command ends.

```bash
docker compose run --rm web django-admin startproject hello_map .
docker compose run --rm web python manage.py startapp mapping
```

Update `hello_map/settings.py` as follows:

1.  Add `'django.contrib.gis'` and `'mapping'` to `INSTALLED_APPS`.
2.  Replace the default `DATABASES` setting with the PostGIS configuration below.
3.  Read `SECRET_KEY` and `DEBUG` from environment variables.

```python
import os

DATABASES = {
    "default": {
        "ENGINE": "django.contrib.gis.db.backends.postgis",
        "NAME": os.environ["DATABASE_NAME"],
        "USER": os.environ["DATABASE_USER"],
        "PASSWORD": os.environ["DATABASE_PASSWORD"],
        "HOST": os.environ["DATABASE_HOST"],
        "PORT": os.environ["DATABASE_PORT"],
    }
}
```

Run Django migrations and create an administrator account:

```bash
docker compose run --rm web python manage.py migrate
docker compose run --rm web python manage.py createsuperuser
docker compose up -d web
```

Open [http://localhost:8000/admin/](http://localhost:8000/admin/) and sign in with the account you created. The Django administration site confirms that the application can read and write to PostgreSQL.

## Part 3: Verify and Use PostGIS Through pgAdmin (30 minutes)

Open [http://localhost:5050](http://localhost:5050) and sign in using `PGADMIN_DEFAULT_EMAIL` and `PGADMIN_DEFAULT_PASSWORD` from `.env`.

Register the database server:

| pgAdmin field | Value |
|---|---|
| Name | `Hello Map PostGIS` |
| Host name/address | `db` |
| Port | `5432` |
| Maintenance database | value of `POSTGRES_DB` |
| Username | value of `POSTGRES_USER` |
| Password | value of `POSTGRES_PASSWORD` |

In pgAdmin's Query Tool, run:

```sql
SELECT PostGIS_Version();
SELECT current_database(), current_user;
```

Both statements must return a result. The PostGIS image enables the extension in the database created by `POSTGRES_DB`. In case, the query return an error, create postgis extension first by running the following command:

```sql
CREATE EXTENSION postgis;
```

You can run the same verification without a browser:

```bash
source .env
docker compose exec db psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT PostGIS_Version();"
```

Part 4: Build the Hello Map (45 minutes)
----------------------------------------

Create a `mapping` view and template that render a Leaflet map. The completed page must:

*   load at `http://localhost:8000/`;
*   centre on Dublin at latitude `53.3498`, longitude `-6.2603`;
*   support zooming and panning;
*   show markers for at least Trinity College Dublin, Dublin Castle, and Temple Bar;
*   include meaningful popup text for each marker.

Use the Leaflet CDN in the template for this first exercise. Connect the root URL to the mapping view, then restart the service after source changes when required:

### 4.1 Create the Mapping View

Update `mapping/views.py`:

```python
from django.shortcuts import render

def map_view(request):
    context = {
        'dublin_lat': 53.3498,
        'dublin_lon': -6.2603,
    }
    return render(request, 'mapping/map.html', context)
```

### 4.2 Configure URLs

Update `hello_map/urls.py`:

```python
from django.contrib import admin
from django.urls import path
from mapping.views import map_view

urlpatterns = [
    path('', map_view, name='map'),
    path('admin/', admin.site.urls),
]
```

### 4.3 Create the Map Template

Create the directory `mapping/templates/mapping/` and add `map.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hello Map Dublin</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.css" />
    <script src="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.js"></script>
    <style>
        body {
            margin: 0;
            padding: 0;
        }
        #map {
            position: absolute;
            top: 0;
            bottom: 0;
            width: 100%;
            height: 100vh;
        }
    </style>
</head>
<body>
    <div id="map"></div>
    <script>
        const map = L.map('map').setView([{{ dublin_lat }}, {{ dublin_lon }}], 13);

        L.tileLayer('https://{{s}}.tile.openstreetmap.org/{{z}}/{{x}}/{{y}}.png', {
            attribution: '© OpenStreetMap contributors',
            maxZoom: 19
        }).addTo(map);

        const markers = [
            {
                lat: 53.3432,
                lon: -6.2575,
                name: 'Trinity College Dublin',
                popup: 'Trinity College Dublin - Founded 1592'
            },
            {
                lat: 53.3414,
                lon: -6.2725,
                name: 'Dublin Castle',
                popup: 'Dublin Castle - Historic seat of power'
            },
            {
                lat: 53.3416,
                lon: -6.2831,
                name: 'Temple Bar',
                popup: 'Temple Bar - Historic neighborhood'
            }
        ];

        markers.forEach(function(marker) {
            L.marker([marker.lat, marker.lon])
                .addTo(map)
                .bindPopup('<b>' + marker.name + '</b><br>' + marker.popup);
        });
    </script>
</body>
</html>
```

### 4.4 Run the Application again

Restart the Django development server:

```bash
docker compose restart web
```

For this bind-mounted development setup, Django normally reloads Python and template changes automatically; restarting is useful when environment or container settings change.

## Submission Evidence

Submit the project repository, excluding `.env`, and provide screenshots showing:

1.  `docker compose ps` with the PostGIS health check passing.
<img width="953" height="95" alt="Screenshot 2026-09-15 143243" src="https://github.com/user-attachments/assets/f694ce38-cd05-4470-9199-9186d7de801d" />

2.  The Hello Map in the browser.
<img width="2253" height="1348" alt="image" src="https://github.com/user-attachments/assets/8bc94450-34ec-4aaf-903b-6b1774460a68" />

4.  pgAdmin connected to the `Hello Map PostGIS` server.
<img width="1127" height="462" alt="Screenshot 2026-09-15 143431" src="https://github.com/user-attachments/assets/0209fd71-2a0f-4bcb-a415-99ca6f5305b6" />

5.  The result of `SELECT PostGIS_Version();`.
<img width="479" height="271" alt="Screenshot 2026-09-15 142942" src="https://github.com/user-attachments/assets/544932c8-a266-4ea3-afef-f5fb2c0c8f6b" />

6.  The Django admin site with your administrator account.
<img width="1127" height="602" alt="Screenshot 2026-09-15 143716" src="https://github.com/user-attachments/assets/a1287086-d9dd-4025-8757-42321c61e365" />

## Troubleshooting

Symptom

Check

Likely resolution

`port is already allocated`

`docker compose ps` and other local services

Stop the process using port `8000` or `5050`, or change the left side of the relevant port mapping.

Django cannot connect to PostgreSQL

`docker compose logs web`

Confirm `DATABASE_HOST=db`, then check `docker compose ps` shows `db` as healthy.

pgAdmin cannot find the database

Server registration fields

Use host `db`, not `localhost`; pgAdmin is also running in a container.

Changes to `POSTGRES_DB` do not apply

Docker volume state

Database initialisation values only apply to a new volume. For a disposable lab reset, use `docker compose down -v` and then `docker compose up --build -d`.

The web service exits after startup

`docker compose logs web`

Create the Django project first, run migrations, and start the web service again.

## Useful Commands

```bash
# Start all services and follow their logs
docker compose up --build

# View service status
docker compose ps

# Follow one service's logs
docker compose logs -f web

# Run a Django management command
docker compose run --rm web python manage.py check

# Stop containers but retain database data (recommended since we will use it in following weeks)
docker compose down

# Remove containers and all lab database data (irreversible - use with caution)
docker compose down -v
```

Looking Ahead
-------------

Keep the three services and the `.env` contract throughout the module. Future weeks add spatial models, GeoJSON APIs, external data, authentication, and deployment configuration while retaining this local development workflow.
