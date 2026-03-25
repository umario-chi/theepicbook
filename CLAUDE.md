# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TheEpicBook is an online bookstore e-commerce application built with Node.js/Express, Handlebars templating, Sequelize ORM, and MySQL 8.0. It runs as a three-container Docker stack (MySQL, Node.js app, Nginx reverse proxy) deployed to GCP via a GitOps pipeline.

## Commands

```bash
# Install dependencies
npm install

# Start the server (port 8080 by default, configurable via PORT env var)
npm start

# Lint (ESLint) — this is also what `npm test` runs
npm run lint

# Run full local stack (MySQL + app + Nginx on port 80)
docker-compose up -d

# Build individual Docker images
docker build -f docker/app/Dockerfile .          # backend
docker build -f docker/nginx/Dockerfile docker/nginx  # frontend/proxy
docker build docker/mysql/                        # database
```

## Architecture

**Three-tier MVC application:**
- **Nginx** (port 80) → reverse proxy to **Express app** (port 8080) → **MySQL** (port 3306)
- Entry point: `server.js` — sets up Express, Handlebars engine, routes, then syncs Sequelize models and starts listening.

**Key directories:**
- `routes/` — `html-routes.js` (page rendering), `cart-api-routes.js` (REST cart API)
- `models/` — Sequelize models: Author (1:many → Book), Book (many:many → Cart via CartBook junction), Cart (1:1 → Checkout)
- `views/` — Handlebars templates with `layouts/main.handlebars` as master layout
- `public/` — Static assets (Materialize CSS, client JS, images)
- `config/config.json` — Sequelize DB connection config per environment (development uses localhost, production uses Docker service name `mysql`)

**Database:** `bookstore` database. Schema in `db/BuyTheBook_Schema.sql`, seed data in `db/author_seed.sql` and `db/books_seed.sql`. The MySQL Docker image auto-initializes from `docker/mysql/init.sql`.

## CI/CD Pipeline

GitHub Actions workflows in `.github/workflows/`:
- **ci.yml** — Triggered on push/PR to main. Runs Snyk, SonarQube, Gitleaks, ESLint, npm audit, Hadolint (Dockerfiles), then builds + scans (Trivy) + pushes Docker images to GCP Artifact Registry.
- **cd.yml** — Triggered on CI success on main. Updates image tags in `gitops/dev/docker-compose.yml`, then SSHs into GCE VM via IAP to deploy.

Images are tagged with both the commit SHA and `latest`, pushed to `us-central1-docker.pkg.dev/expandox-project-2/cloudopshub/`.

## Infrastructure

- `terraform/` — GCP infrastructure modules (VPC, compute, storage, load balancer, Cloud Armor, etc.) with per-environment configs in `terraform/environments/{dev,staging,prod}/`.
- `gitops/{dev,staging,prod}/` — Per-environment Docker Compose manifests for ArgoCD-style deployment.
- `monitoring/` — Prometheus, Grafana, and Alertmanager configs bundled with production compose stacks.

## Code Style

ESLint enforces (`.eslintrc.json`): double quotes, semicolons, 2-space indentation, camelCase, strict equality (`===`), no unused variables, curly braces required. The `npm test` command runs only linting — there are no unit or integration tests.
