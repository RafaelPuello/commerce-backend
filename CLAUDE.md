# CLAUDE.md - Saleor Platform (E-Commerce Backend)

This file provides guidance to Claude Code (claude.ai/code) when working with the Saleor Platform local development environment.

## Overview

Saleor Platform is a headless e-commerce backend providing GraphQL API, admin dashboard, and payment/order management. This directory contains the **local development environment** for DigiDex's e-commerce infrastructure. It is not used for production deployment but supports the Storefront frontend at `/shop`.

**Integration with DigiDex:**
- Serves the Storefront (`../storefront`) via GraphQL API
- Accessible at `https://api.digidex.bio/graphql/` (production) and `http://api.localhost:10000/graphql/` (development)
- Routed via Traefik on shared `digidex-net` network

## Service Architecture

### Backend Services (on `digidex-net` network - discoverable by Traefik)

| Service | Image | Purpose | Port | Key Details |
|---------|-------|---------|------|-------------|
| **api** | ghcr.io/saleor/saleor:3.22 | GraphQL API, order management, product catalog | 8000 | Main backend; handles all business logic |
| **worker** | ghcr.io/saleor/saleor:3.22 | Celery async task processor | N/A | Processes orders, emails, webhooks, background jobs |
| **dashboard** | ghcr.io/saleor/saleor-dashboard:latest | Admin UI for managing products, orders, customers | 9000 | Dev-only; not exposed in production |

### Infrastructure Services (on isolated `saleor-backend-tier` network - internal)

| Service | Image | Purpose | Port | Key Details |
|---------|-------|---------|------|-------------|
| **db** | postgres:15 | PostgreSQL database | 5432 | Persistent data storage; initialized with `replica_user.sql` |
| **cache** | valkey/valkey:latest | Redis-compatible cache & Celery broker | 6379 | In-memory caching, session storage, task queue |
| **jaeger** | jaegertracing/all-in-one:latest | Distributed tracing & APM | 16686 (UI), 4317/4318 (OTLP) | Optional; traces API and worker requests |
| **mailpit** | axllent/mailpit:latest | SMTP test server & web UI | 8025 (UI), 1025 (SMTP) | Captures outgoing emails during development |

**Network Isolation:** Backend infrastructure services (db, cache, jaeger, mailpit) run on an isolated `saleor-backend-tier` bridge network. API and Worker services can access them, but external traffic cannot. This provides security during development.

## Commands

### Initial Setup (Development)
```bash
cd saleor-platform

# Start all services
docker compose up

# In another terminal, initialize database
docker compose run --rm api python3 manage.py migrate

# Populate with sample products, orders, customers
docker compose run --rm api python3 manage.py populatedb --createsuperuser
```

The `populatedb --createsuperuser` flag creates:
- Demo superuser: `admin@example.com` / password: `admin`
- Sample products, collections, categories
- Sample orders and customers

### Running Services (Development)

```bash
# All services (API, Worker, Dashboard, DB, Cache, etc.)
docker compose up

# API and Worker only (no Dashboard, test email, tracing)
docker compose up api worker db cache

# Specific service
docker compose up api              # Just API (requires db, cache running)

# Run in background
docker compose up -d

# Stop all services
docker compose stop

# Stop and remove containers (data persists in volumes)
docker compose down

# Stop and remove everything including data
docker compose down --volumes
```

### Testing
```bash
# Run pytest suite
docker compose run --rm api pytest -n logical --allow-hosts cache

# Run specific test file
docker compose run --rm api pytest tests/graphql/e2e/

# With coverage
docker compose run --rm api pytest --cov
```

### Database Management

```bash
# Run database migrations
docker compose run --rm api python3 manage.py migrate

# Check migration status
docker compose run --rm api python3 manage.py showmigrations

# Create new migration
docker compose run --rm api python3 manage.py makemigrations

# Reset database (WARNING: deletes all data)
docker compose down --volumes db
docker compose up db
docker compose run --rm api python3 manage.py migrate
```

### Admin & Debugging

```bash
# Create additional superuser
docker compose run --rm api python3 manage.py createsuperuser

# Django shell for manual queries
docker compose run --rm api python3 manage.py shell

# View logs in real-time
docker compose logs -f api

# View logs for specific service
docker compose logs -f worker
docker compose logs -f db
```

### Cleanup & Rebuild

```bash
# Full restart (clean containers, rebuild images)
docker compose down
docker compose build
docker compose up

# Remove unused images/volumes (frees disk space)
docker image prune
docker volume prune
```

## Configuration

### Environment Files

Configuration is split into layered `.env` files:

**common.env** (Shared across all services)
```bash
# Channel configuration
SALEOR_CHANNEL_DEFAULT_SLUG=default
SALEOR_CHANNEL_DEFAULT_CURRENCY=USD

# IP filtering (for security)
ALLOWED_HOSTS=api.localhost,api.127.0.0.1
```

**backend.env** (API and Worker specific)
```bash
# Database connection
DATABASE_URL=postgresql://saleor:saleor@db/saleor

# Cache backend (Redis-compatible Valkey)
REDIS_URL=redis://cache:6379/

# Email (uses mailpit in dev)
EMAIL_URL=smtp://mailpit:1025

# Celery broker
CELERY_BROKER_URL=redis://cache:6379/1

# Jaeger tracing (optional)
JAEGER_ENABLED=true
OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317

# Django security
SECRET_KEY=dev-secret-key-change-in-production
DEBUG=true
```

**replica_user.sql** (Database initialization)
- Creates read-only PostgreSQL user for replica databases
- Executes on first `db` service startup via `init` volume

### Service Port Reference

**Development Environment:**
- API GraphQL: `http://api.localhost:10000/graphql/` (routed via Traefik)
- Dashboard: `http://admin.localhost:10000` (routed via Traefik; dev-only)
- Mailpit: `http://localhost:8025` (test email inbox)
- Jaeger: `http://localhost:16686` (tracing dashboard)

**Production Environment:**
- API GraphQL: `https://api.digidex.bio/graphql/` (routed via Traefik)
- Dashboard: Not exposed (security)
- Mailpit/Jaeger: Not exposed

## Port Conflict Notes

**Important:** The Saleor API runs on port 8000 inside the container. This may conflict with other Django services in development:
- **CMS backend**: 8000
- **App backend**: 8000 (dev), 8003 (prod)
- **ID backend**: 8001 (non-standard, no conflict)

In development, Docker Compose and Traefik handle routing by service name, so no host-level conflicts occur. If running Saleor outside Docker, adjust port bindings in `docker-compose.yml` to avoid conflicts with other local services.

## Development Workflow

### Adding a GraphQL Mutation or Query
1. Write mutation/query in Saleor backend source
2. Run `docker compose run --rm api python3 manage.py test` to verify
3. Query is immediately available in GraphQL schema
4. Storefront can query the new mutation/query

### Processing Orders Asynchronously
- Worker service processes order events, sends emails, integrates payments
- Tasks are queued in Redis (via Celery)
- Monitor worker: `docker compose logs -f worker`

### Debugging API Issues
```bash
# Check logs for errors
docker compose logs api | grep -i error

# Django shell with full environment
docker compose run --rm api python3 manage.py shell
>>> from saleor.product.models import Product
>>> Product.objects.all()
```

### Testing Email Integration
- Mailpit captures all outgoing email (SMTP)
- View captured emails: http://localhost:8025
- Useful for testing order confirmation, password reset, etc.

## Important Gotchas

### Database Persistence
By default, Docker volumes persist database data between `docker compose up/down` cycles. To reset: `docker compose down --volumes`

### Worker Requires Cache
The Celery worker needs Redis/Valkey running to process tasks. If only the API is running, background tasks queue but don't execute.

### GraphQL Schema Caching
Changes to schema may require clearing cache:
```bash
docker compose exec cache redis-cli flushall
```

### Ports in Docker vs. Host
Internal service ports (8000, 5432) are NOT exposed on the host by default. Access via container names (e.g., `http://api:8000/` from within Docker network). External access goes through Traefik (production) or direct container exposure (dev).

### OTEL/Jaeger Optional
Jaeger tracing is optional for local development. Set `JAEGER_ENABLED=false` in `backend.env` to skip it if disk space is limited.

## Saleor Documentation

- [Saleor API Docs](https://docs.saleor.io/)
- [GraphQL Schema Explorer](http://api.localhost:10000/graphql/) (development)
- [GraphQL API Reference](https://docs.saleor.io/developer/api-reference)
- [Webhook Events](https://docs.saleor.io/developer/api-reference/webhooks)
