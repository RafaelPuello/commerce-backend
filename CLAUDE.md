# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Saleor Platform local development environment for e-commerce. It orchestrates multiple Saleor services via Docker Compose and is intended for local development only - not production deployment.

## Architecture

The platform runs these services via Docker Compose:
- **api** (port 8000): Saleor Core GraphQL API (ghcr.io/saleor/saleor:3.22)
- **dashboard** (port 9000): Saleor Dashboard web UI
- **worker**: Celery worker for async tasks (same image as api)
- **db** (port 5432): PostgreSQL 15 database
- **cache** (port 6379): Valkey (Redis-compatible) for caching/Celery broker
- **jaeger** (port 16686): APM/tracing with OTLP endpoints on 4317/4318
- **mailpit** (port 8025): Test email interface (SMTP on 1025)

All backend services communicate via the `saleor-backend-tier` bridge network.

## Commands

### Initial Setup
```bash
docker compose run --rm api python3 manage.py migrate
docker compose run --rm api python3 manage.py populatedb --createsuperuser
```
The `--createsuperuser` flag creates admin@example.com with password "admin".

### Running Services
```bash
docker compose up                    # All services
docker compose up api worker         # Backend only
docker compose stop                  # Stop all
```

### Testing
```bash
docker compose run api pytest -n logical --allow-hosts cache
```

### Database Reset
```bash
docker compose down --volumes db     # Warning: removes all data
```

### Cleanup and Rebuild
```bash
docker compose stop
docker compose rm
docker compose build
```

## Configuration

- `common.env`: Shared settings (channel config, IP filtering)
- `backend.env`: Backend-specific settings (database URLs, secrets, OTEL config)
- `replica_user.sql`: Creates read-only DB user for replicas on init

Ports are bound to 10.0.0.218 rather than localhost for network accessibility.
