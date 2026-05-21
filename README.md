# My-Server Deployment Guide

This repository contains the Docker deployment scaffold for a backend service, plus supporting infrastructure for MariaDB, phpMyAdmin, Redis, and example Nginx reverse-proxy configuration.

The project is split into two Docker Compose files:

- `server-compose.yml`: shared infrastructure services
- `compose.yml`: the backend application service

## Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Important First Step: Create Docker Network](#important-first-step-create-docker-network)
- [Repository Structure](#repository-structure)
- [Configuration](#configuration)
- [Infrastructure Services](#infrastructure-services)
- [Application Service](#application-service)
- [How to Deploy](#how-to-deploy)
- [Nginx Reverse Proxy](#nginx-reverse-proxy)
- [Persistent Data](#persistent-data)
- [Health Check and Startup Behavior](#health-check-and-startup-behavior)
- [Operations](#operations)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)

## Overview

This repository is intended to run a backend API stack using Docker. It provides:

- A MariaDB database container
- A phpMyAdmin container for database administration
- A Redis container
- A backend API container image reference
- A startup script that waits for the database and manages Aerich migrations
- Example Nginx configuration for HTTP and HTTPS proxying

## Architecture

All services join the same external Docker network named `server_net`.

Core runtime layout:

1. `mariadb` provides the SQL database.
2. `redis` provides in-memory caching or queue support.
3. `phpmyadmin` exposes database administration on port `3300`.
4. `api` runs the backend application on port `8000`.
5. Nginx can proxy public traffic to the API and phpMyAdmin.

## Prerequisites

Before deployment, make sure the server has:

- Docker installed
- Docker Compose plugin installed
- A valid `.env` file for the application service
- A backend image pushed to the configured image registry
- DNS configured for your API domain if using Nginx

## Important First Step: Create Docker Network

Create the shared external Docker network before starting any service:

```bash
docker network create server_net
```

This is required because both `server-compose.yml` and `compose.yml` reference `server_net` as an external network. If it does not exist, Docker Compose will fail to start the services.

## Repository Structure

```text
.
├── README.md
├── REDME.md
├── Dockerfile
├── compose.yml
├── server-compose.yml
├── start.sh
└── nginx/
    ├── api.domain.com
    └── api.domain.com.ssl
```

## Configuration

### `server-compose.yml`

This file defines the shared infrastructure services:

- `mariadb`
- `phpmyadmin`
- `redis`

It also attaches them to the external network `server_net`.

### `compose.yml`

This file defines the application service:

- Compose project name: `PROJECT_NAME`
- Service name: `api`
- Container port exposed: `8000`
- External network: `server_net`

The application image is pulled from:

```text
${IMAGE_NAMESPACE:-moynul2k3}/PROJECT_NAME-backend:${IMAGE_TAG:-latest}
```

You must replace `PROJECT_NAME` with your real project name before production use.

### Environment File

The application service loads environment variables from:

```text
.env
```

At minimum, the backend runtime likely needs values such as:

- `DATABASE_URL` or `DB_HOST` / `DB_PORT`
- `PORT`
- `WEB_CONCURRENCY`
- `RUN_MIGRATIONS`
- `AUTO_CREATE_MIGRATIONS`
- `FAKE_UPGRADE_ON_CONFLICT`
- `MIGRATION_NAME`

Exact application-specific variables depend on the backend codebase and are not included in this repository snapshot.

## Infrastructure Services

### MariaDB

Defined in `server-compose.yml` with:

- Image: `mariadb:10.11`
- Container name: `mariadb`
- Internal network only
- Persistent bind mount: `./data/mariadb:/var/lib/mysql`

Configured environment variables:

- `MYSQL_ROOT_PASSWORD=root2k3`
- `MYSQL_DATABASE=moynul`
- `MYSQL_USER=moynul`
- `MYSQL_PASSWORD=moynul2k3`

### phpMyAdmin

Defined in `server-compose.yml` with:

- Image: `phpmyadmin/phpmyadmin:latest`
- Container name: `phpmyadmin`
- Host port mapping: `3300:80`

Access it locally at:

```text
http://YOUR_SERVER_IP:3300
```

Or through Nginx at:

```text
https://api.your-domain.com/phpmyadmin/
```

if you apply the provided Nginx example.

### Redis

Defined in `server-compose.yml` with:

- Image: `redis:7-alpine`
- Container name: `redis`
- Named volume: `redis_data:/data`

Redis is not exposed directly to the host in the current configuration.

## Application Service

The backend service is defined in `compose.yml`.

### Runtime Behavior

- Pulls a prebuilt image from a registry
- Restarts automatically unless stopped manually
- Loads environment variables from `.env`
- Mounts local `./media` into `/app/media`
- Mounts local `./migrations` into `/app/migrations`
- Exposes port `8000`
- Runs a health check against `http://localhost:8000/`

### Mounted Directories

- `./media`: for uploaded or generated media files
- `./migrations`: for Aerich migration files

### Health Check

The container is considered healthy only if:

```bash
curl -fsS http://localhost:8000/
```

returns success inside the container.

## How to Deploy

### 1. Create the Docker network

```bash
docker network create server_net
```

### 2. Start infrastructure services

```bash
docker compose -f server-compose.yml up -d
```

This starts:

- MariaDB
- phpMyAdmin
- Redis

### 3. Create the application environment file

Create a `.env` file in the repository root and define the variables required by your backend image.

Example skeleton:

```env
DATABASE_URL=mysql://moynul:moynul2k3@mariadb:3306/moynul
DB_HOST=mariadb
DB_PORT=3306
PORT=8000
WEB_CONCURRENCY=3
RUN_MIGRATIONS=true
AUTO_CREATE_MIGRATIONS=false
FAKE_UPGRADE_ON_CONFLICT=true
MIGRATION_NAME=server_auto
```

Adjust these values to match your application.

### 4. Update placeholders in `compose.yml`

Replace:

- `PROJECT_NAME`

with your actual project name.

Also confirm:

- `IMAGE_NAMESPACE`
- `IMAGE_TAG`

resolve to the correct image in your registry.

### 5. Start the application service

```bash
docker compose -f compose.yml up -d
```

### 6. Verify running containers

```bash
docker compose -f server-compose.yml ps
docker compose -f compose.yml ps
```

### 7. Check logs

```bash
docker compose -f server-compose.yml logs -f
docker compose -f compose.yml logs -f
```

## Nginx Reverse Proxy

The `nginx/` directory contains two example server blocks:

- `nginx/api.domain.com`: HTTP proxy example
- `nginx/api.domain.com.ssl`: HTTPS proxy example with Certbot-managed TLS

### What the Nginx config does

- Proxies `/` to the backend on `127.0.0.1:8000`
- Proxies `/phpmyadmin/` to `127.0.0.1:3300`
- Supports WebSocket upgrade headers
- Sets `client_max_body_size 300M`

### Required changes before use

Replace:

- `api.DOMAIN.COM`

with your real domain in both Nginx files.

For the SSL version, confirm these certificate paths exist:

- `/etc/letsencrypt/live/api.DOMAIN.COM/fullchain.pem`
- `/etc/letsencrypt/live/api.DOMAIN.COM/privkey.pem`

## Persistent Data

The deployment stores data in:

- `./data/mariadb`: MariaDB database files
- Docker volume `redis_data`: Redis persistence
- `./media`: backend media files
- `./migrations`: Aerich migration files

Make sure these paths are backed up as needed.

## Health Check and Startup Behavior

The `start.sh` script is the application entrypoint.

### What it does

1. Reads database connection information from `DB_HOST` / `DB_PORT` or `DATABASE_URL`
2. Waits until the database port is reachable
3. Ensures migration directories exist
4. Runs Aerich migration logic
5. Starts Uvicorn for `app.main:app`

### Migration behavior

`start.sh` supports the following behavior flags:

- `RUN_MIGRATIONS=true`: run migration logic on startup
- `AUTO_CREATE_MIGRATIONS=true`: run `aerich migrate`
- `FAKE_UPGRADE_ON_CONFLICT=true`: retry with `aerich upgrade --fake` for known duplicate/existing schema conflicts
- `MIGRATION_NAME=server_auto`: migration name used when auto-generating migrations

### Application server startup

The script starts the app with:

```bash
uvicorn app.main:app --host 0.0.0.0 --port "$PORT" --workers "$WORKERS"
```

This means the backend image is expected to include:

- an importable module at `app.main`
- a FastAPI or ASGI app exposed as `app`
- Aerich installed
- all runtime dependencies already built into the image

## Operations

### Stop infrastructure

```bash
docker compose -f server-compose.yml down
```

### Stop application

```bash
docker compose -f compose.yml down
```

### Restart infrastructure

```bash
docker compose -f server-compose.yml restart
```

### Restart application

```bash
docker compose -f compose.yml restart
```

### Pull the latest backend image and recreate the app

```bash
docker compose -f compose.yml pull
docker compose -f compose.yml up -d
```

### View application logs

```bash
docker compose -f compose.yml logs -f api
```

### View database logs

```bash
docker compose -f server-compose.yml logs -f mariadb
```

## Troubleshooting

### Error: external network `server_net` not found

Run:

```bash
docker network create server_net
```

### Application cannot connect to database

Check:

- MariaDB container is running
- `.env` database variables are correct
- `DB_HOST` points to `mariadb` when using Compose networking
- both stacks are attached to `server_net`

### Health check failing

Check:

- the backend listens on port `8000`
- the `/` route returns HTTP 200
- the container includes `curl`

### Migration failures on startup

Review:

```bash
docker compose -f compose.yml logs -f api
```

If the schema already exists, `FAKE_UPGRADE_ON_CONFLICT=true` may allow recovery for duplicate object errors.

### phpMyAdmin not reachable

Check:

- port `3300` is open on the host
- the container is running
- Nginx proxy rules are correct if accessing through `/phpmyadmin/`

## Security Notes

This repository currently includes hard-coded database credentials in `server-compose.yml`. That is acceptable for local testing only. For any real deployment:

- move secrets to environment variables or Docker secrets
- use strong passwords
- restrict phpMyAdmin access
- avoid exposing database tooling publicly unless protected
- replace placeholder domains and image names
- review firewall rules for ports `3300` and `8000`

## Notes

- `REDME.md` exists in the repository but is empty and appears to be a typo.
- `README.md` should be treated as the primary documentation file.
