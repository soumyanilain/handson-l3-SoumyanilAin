# Hands-On L3 — Docker & a Multi-Container Microservice

ITCS 6190/8190 — Cloud Computing for Data Analysis  
A Flask web app backed by a Redis cache, orchestrated with Docker Compose, plus a standalone PostgreSQL container exercise.

## Stack

| Service | Image | Ports |
|---|---|---|
| `web` | built from local `Dockerfile` (`python:3.12-alpine`) | `8000:5000` |
| `redis` | `redis:alpine` | internal only |
| `postgres1` (standalone) | `postgres:16` | `5432:5432` |

## Repository Contents

| File | Purpose |
|---|---|
| `app.py` | Flask application with the Redis hit counter |
| `requirements.txt` | Python dependencies |
| `Dockerfile` | Build instructions for the `web` image |
| `compose.yaml` | Service definitions for `web` and `redis` |
| `.gitignore` | Keeps `__pycache__/` and other artefacts out of the repo |
| `screenshots.docx` | Report with Docker Desktop and output screenshots |

## Execution Steps

### 1. Verify Docker

```bash
docker --version
```

On Windows, Docker Desktop needs WSL 2. If it isn't present, install it from an administrative PowerShell and reboot:

```powershell
wsl --install
```

### 2. PostgreSQL container

Pull the image:

```bash
docker pull postgres:16
```

Start an instance:

```bash
docker run -d -p 5432:5432 --name postgres1 -e POSTGRES_PASSWORD=pass12345 postgres:16
```

Open a shell inside it and connect with `psql`:

```bash
docker exec -it postgres1 bash
psql -d postgres -U postgres
```

Verify the database is live:

```sql
\l
SELECT version();
CREATE TABLE test (id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO test (name) VALUES ('hello docker');
SELECT * FROM test;
```

Leave `psql` with `\q`, leave the container with `exit`, then clean up so the port is released:

```bash
docker stop postgres1
docker rm postgres1
```

> If port 5432 is already taken by a local PostgreSQL install, publish a different host port: `-p 5433:5432`.

### 3. Build and run the Flask + Redis stack

```bash
docker compose up --build
```

Open <http://localhost:8000> and refresh a few times — the counter increments because Redis is holding the `hits` key.

Confirm both containers are running:

```bash
docker compose ps
docker compose logs -f web
```

Read the counter straight out of the cache:

```bash
docker compose exec redis redis-cli get hits
```

Tear it down:

```bash
docker compose down
```

## What I Learned

- **Container networking by service name.** `redis.Redis(host='redis', ...)` works because Compose puts both services on a shared user-defined bridge network and registers each service name in its internal DNS. The same code fails when run on the host, where `redis` resolves to nothing.
- **`depends_on` controls start order, not readiness.** It waits for the Redis container to start, not for Redis to accept connections. That's exactly why `get_hit_count()` retries five times with a half-second backoff instead of assuming the cache is up.
- **Port publishing is a host-to-container mapping.** `"8000:5000"` means the browser talks to 8000 on my machine while Flask still listens on 5000 inside the container. `EXPOSE 5000` is documentation for humans and tooling; it doesn't publish anything by itself.
- **`FLASK_RUN_HOST=0.0.0.0` is mandatory in a container.** Flask's default bind of `127.0.0.1` only accepts traffic from inside the container's own network namespace, so the published port would connect to nothing.
- **Layer caching rewards ordering.** Copying `requirements.txt` and running `pip install` *before* `COPY . .` means editing application code doesn't invalidate the dependency layer, so rebuilds take seconds instead of minutes.
- **Alpine needs a toolchain for wheels.** `apk add gcc musl-dev linux-headers` exists because Alpine uses musl instead of glibc, so packages without musl wheels get compiled from source.
- **Dependencies live in the image, not on the host.** Running `python app.py` on Windows fails with `ModuleNotFoundError: No module named 'redis'` — the environment the app needs only exists inside the container.
- **Stateless containers, stateful services.** `docker compose down` destroys the Redis container and the counter resets — persistence would need a named volume, which this exercise deliberately doesn't have.

## Errors Encountered

All errors hit during this hands-on are logged under the [Issues](../../issues) tab, one per distinct error, each with the `bug` label and a closing comment describing the fix.
