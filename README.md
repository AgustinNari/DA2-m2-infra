# Citizen Services M2 — Infrastructure

Infrastructure repository for the M2 citizen-services platform.

The repository currently provides local orchestration for the frontend, backend, and PostgreSQL database through Docker Compose.

> This repository is currently under active development.

## Current Stack

- Docker
- Docker Compose
- PostgreSQL 17
- Spring Boot backend
- React frontend

## Current Services

The Compose environment includes:

### PostgreSQL

- PostgreSQL 17
- Persistent Docker volume
- Health check
- Environment-based credentials

### Backend

- Spring Boot service
- PostgreSQL connection through the internal Docker network
- Port `8080`

### Frontend

- React frontend
- Containerized build
- API base path configured as `/api`
- Port `8081`

## Local Configuration

Copy:

```text
compose/.env.example
```

to:

```text
compose/.env
```

and provide local database credentials.

The current Compose configuration expects the frontend and backend repositories to exist at the relative paths used by the development workspace.

## Running Locally

From `compose/`:

```bash
docker compose up --build
```

Stop the environment with:

```bash
docker compose down
```

## Status

This repository is evolving alongside the wider platform infrastructure.

Additional deployment, cloud, reverse-proxy, and automation components should be documented here as they become part of the repository's actual `main` branch.
