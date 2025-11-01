# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Repository Overview

This repository provides Docker containerization for [Frappe](https://github.com/frappe/frappe) and [ERPNext](https://github.com/frappe/erpnext). It contains multi-stage Dockerfiles, docker-compose configurations for different deployment scenarios, and development tooling for building and testing Frappe/ERPNext applications in containers.

## Core Architecture

The project uses a multi-service container architecture with these key components:

- **Backend**: Frappe/ERPNext application server (Gunicorn)
- **Frontend**: Nginx reverse proxy serving static assets
- **WebSocket**: Node.js socket.io server for real-time features
- **Queue Workers**: Background task processors (short/long queues)
- **Scheduler**: Cron-like job scheduler
- **Database**: MariaDB or PostgreSQL
- **Cache/Queue**: Redis for caching and task queuing
- **Configurator**: One-time setup container for initial configuration

Container images are built using Docker Buildx Bake with multi-stage builds targeting different environments (base, build, erpnext).

## Essential Commands

### Quick Start / Testing
```bash
# Clone and quick test setup
git clone https://github.com/frappe/frappe_docker
cd frappe_docker
docker compose -f pwd.yml up -d

# For ARM64 architecture
docker buildx bake --no-cache --set "*.platform=linux/arm64"
# Then modify pwd.yml to add platform: linux/arm64 and use :latest tags
```

### Building Images
```bash
# Build all default targets (erpnext, base, build)
FRAPPE_VERSION=develop ERPNEXT_VERSION=develop docker buildx bake

# Build specific targets
FRAPPE_VERSION=version-15 ERPNEXT_VERSION=version-15 docker buildx bake erpnext

# Build bench image
docker buildx bake bench
```

### Development Environment
```bash
# Setup development containers
cp -R devcontainer-example .devcontainer
cp -R development/vscode-example development/.vscode

# Start dev containers manually
docker-compose -f .devcontainer/docker-compose.yml up -d

# Enter development container
docker exec -e "TERM=xterm-256color" -w /workspace/development -it devcontainer-frappe-1 bash

# Create new bench inside container
bench init --skip-redis-config-generation frappe-bench
cd frappe-bench

# Configure for containerized services
bench set-config -g db_host mariadb
bench set-config -g redis_cache redis://redis-cache:6379
bench set-config -g redis_queue redis://redis-queue:6379
bench set-config -g redis_socketio redis://redis-queue:6379

# Create new site
bench new-site --mariadb-user-host-login-scope=% development.localhost
bench --site development.localhost set-config developer_mode 1
bench --site development.localhost install-app erpnext
```

### Testing and Linting
```bash
# Install and run pre-commit hooks
pip install pre-commit  # or brew install pre-commit
pre-commit install
pre-commit run --all-files

# Run Python tests
python3 -m venv venv
source venv/bin/activate
pip install -r requirements-test.txt
pytest
```

### Production Deployments
```bash
# Basic production setup with MariaDB
docker compose -f compose.yaml -f overrides/compose.mariadb.yaml up -d

# With custom domain and SSL
docker compose -f compose.yaml -f overrides/compose.mariadb.yaml -f overrides/compose.https.yaml up -d

# Multi-bench setup
docker compose -f compose.yaml -f overrides/compose.mariadb.yaml -f overrides/compose.multi-bench.yaml up -d
```

## Development Setup Patterns

### VSCode Dev Containers
The repository is configured for VSCode Remote Container development:
- Copy `devcontainer-example/` to `.devcontainer/`
- Use "Reopen in Container" command
- Development happens in `/workspace/development/` inside container
- Multiple Python (pyenv) and Node (nvm) versions available

### Manual Bench Setup vs Scripted
Two approaches for creating development benches:
1. Manual: Step-by-step `bench init`, `bench get-app`, configuration
2. Scripted: `python installer.py` with `apps-example.json` configuration

### Database Options
- MariaDB (default): Use `compose.mariadb.yaml` override
- PostgreSQL: Use `compose.postgres.yaml` override and `--db-type postgres` in bench commands

## Key Environment Variables

Referenced from `example.env`:
- `ERPNEXT_VERSION`: Image tag version (e.g., v15.77.0)
- `DB_PASSWORD`: Database password (default: 123)
- `DB_HOST`/`DB_PORT`: External database connection
- `REDIS_CACHE`/`REDIS_QUEUE`: External Redis connections
- `FRAPPE_SITE_NAME_HEADER`: Site resolution override
- `LETSENCRYPT_EMAIL`: For SSL certificate generation

## Build System Details

Uses Docker Buildx Bake (`docker-bake.hcl`) with:
- Multi-architecture support (amd64/arm64)
- Version-based tagging with major version shortcuts
- Configurable Frappe/ERPNext repository sources
- Multi-stage builds for optimized production images

## Site Operations

Common bench commands for site management:
```bash
# Inside development container or on sites
bench new-site mysite.localhost
bench --site mysite.localhost install-app erpnext
bench --site mysite.localhost set-config developer_mode 1
bench --site mysite.localhost migrate
bench --site mysite.localhost clear-cache
bench --site mysite.localhost backup
```

## Important Notes

- Sites must end with `.localhost` for local development
- Default passwords: MariaDB root=`admin`, site admin=`admin`  
- The `/workspace/development` directory is mounted and git-ignored for development work
- Production containers restrict bash commands and show warning message
- Images are built for both Frappe framework and ERPNext application layers
- Custom apps can be added by modifying the build process or using existing bench commands
