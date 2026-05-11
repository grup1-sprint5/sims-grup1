# Sprint 6 — Chaos & Delivery

Final documentation for the **SIMS** project (Smart Integrated Mobility System).  
Covers the full setup of the development environment, the testing environment, Sprint 6 changes, and the CI/CD pipeline.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Development Environment Setup](#3-development-environment-setup)
   - [Backend (Laravel)](#31-backend-laravel)
   - [Frontend (Vue 3)](#32-frontend-vue-3)
   - [Local Subdomains](#33-local-subdomains)
4. [Testing Environment Setup](#4-testing-environment-setup)
   - [Backend Tests (PHPUnit / Pest)](#41-backend-tests-phpunit--pest)
   - [Frontend Tests (Vitest)](#42-frontend-tests-vitest)
5. [Test Credentials](#5-test-credentials)
6. [Sprint 6 Changes](#6-sprint-6-changes)
   - [Resolved Bugs (cross-team testing)](#61-resolved-bugs-cross-team-testing)
   - [New Features](#62-new-features)
7. [CI/CD — GitHub Actions](#7-cicd--github-actions)
8. [Subdomain Deployment](#8-subdomain-deployment)
9. [Security](#9-security)

---

## 1. System Overview

SIMS is a multi-tenant platform for managing electric vehicle rental fleets.  
Each company (tenant) has its own subdomain, isolated database, and separate interfaces for clients and administrators.

| Role | Access | Description |
|---|---|---|
| **Client** | `<tenant>.grup1-sims.com` | Books vehicles, pays with Stripe, manages support tickets |
| **Admin** | `<tenant>.grup1-sims.com/admin` | Manages users, vehicles, bookings and tickets within the tenant |
| **SuperAdmin** | `grup1-sims.com/admin/tenants` | Approves or rejects new tenant requests |

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│  Frontend  ·  Vue 3 + TypeScript + Vite              │
│  Tailwind CSS · Pinia · Vue Router · Leaflet         │
│  Subdomain per tenant: <slug>.grup1-sims.com         │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS (Nginx)
┌──────────────────────▼──────────────────────────────┐
│  Backend  ·  Laravel 12 + PHP 8.2                   │
│  Multi-tenancy by subdomain (stancl/tenancy)         │
│  REST API  ·  Sanctum Auth  ·  Stripe Webhooks       │
└────────┬──────────────────┬───────────────────────  ┘
         │                  │
┌────────▼────────┐  ┌──────▼──────────┐
│  PostgreSQL 15  │  │  MongoDB 6       │
│  Data per       │  │  Vehicle         │
│  schema/tenant  │  │  locations (IoT) │
└─────────────────┘  └─────────────────┘
```

**Repositories:**
- Backend: `project-sims-backend/`
- Frontend: `project-sims-frontend/`

---

## 3. Development Environment Setup

### 3.1 Backend (Laravel)

**Prerequisites:** Docker Desktop, Docker Compose v2

```bash
cd project-sims-backend

# 1. Copy the environment file
cp .env.example .env

# 2. Start the containers (app, PostgreSQL db, pgAdmin, MongoDB)
docker compose up -d --build

# 3. Install PHP dependencies
docker compose exec app composer install --no-interaction

# 4. Generate the application key
docker compose exec app php artisan key:generate --force

# 5. Run central migrations (tenants, domains, cache...)
docker compose exec app php artisan migrate --force

# 6. Run per-tenant migrations (vehicles, bookings, tickets...)
docker compose exec app php artisan tenants:migrate --no-interaction

# 7. Seed the database with demo data
docker compose exec app php artisan tenants:seed \
  --class="Database\\Seeders\\DatabaseSeeder" --force --no-interaction
```

**Services available once running:**

| Service | Local URL |
|---|---|
| Backend API | http://localhost:8001 |
| pgAdmin | http://localhost:8080 |
| MongoDB | localhost:27017 |

**Start the scheduler** (automatic booking cancellation):

```bash
docker compose up -d scheduler
```

**Key environment variables (`.env`):**

```env
APP_URL=http://localhost:8001

DB_HOST=db
DB_PORT=5432
DB_DATABASE=project_sims
DB_USERNAME=project_user
DB_PASSWORD=supersecret

MONGODB_URI=mongodb://mongo:27017
MONGODB_DATABASE=cluster-iot

# Stripe (payments)
STRIPE_KEY=pk_test_xxx
STRIPE_SECRET=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_SUCCESS_URL=http://localhost:5173/client/bookings
STRIPE_CANCEL_URL=http://localhost:5173/client/bookings

# AI (Gemini assistant)
IA_API_URL=https://generativelanguage.googleapis.com/v1beta/openai
IA_API_KEY=xxx
IA_MODEL=gemini-2.0-flash-lite
```

---

### 3.2 Frontend (Vue 3)

**Prerequisites:** Node.js >= 20.19, npm

```bash
cd project-sims-frontend

# Install dependencies
npm install

# Development mode (hot-reload)
npm run dev
```

The frontend starts at **http://localhost:5173**.

**Production build:**

```bash
npm run build
# Static files are generated in dist/
```

**Frontend environment variables (`.env` or `vite.config.ts`):**

The frontend resolves the tenant by subdomain. For local testing, configure the hosts file (see [section 3.3](#33-local-subdomains)).

---

### 3.3 Local Subdomains

To test multi-tenant routing locally, add entries to the `hosts` file.

**Linux / macOS** (`/etc/hosts`) or **Windows** (`C:\Windows\System32\drivers\etc\hosts`, as Administrator):

```
127.0.0.1  sims-corp.localhost
127.0.0.1  ecomove.localhost
```

Access the demo tenants at:

| Tenant | Local frontend URL |
|---|---|
| SIMS Corp | http://sims-corp.localhost:5173 |
| EcoMove | http://ecomove.localhost:5173 |

> Remove these entries when no longer needed.

---

## 4. Testing Environment Setup

### 4.1 Backend Tests (PHPUnit / Pest)

Tests use a **separate** PostgreSQL database (`sims_test`) so development data is never affected.

**Step 1 — Create the testing environment file:**

```bash
cp .env.testing.example .env.testing
```

Minimum contents of `.env.testing`:

```env
APP_ENV=testing

# Running tests from the host machine:
DB_HOST=localhost
DB_PORT=5433
DB_DATABASE=sims_test
DB_USERNAME=project_user
DB_PASSWORD=change_me

# Running tests from inside the Docker container:
# DB_HOST=db
# DB_PORT=5432

BROADCAST_CONNECTION=null
CACHE_STORE=array
QUEUE_CONNECTION=sync
SESSION_DRIVER=array
MAIL_MAILER=array
```

**Step 2 — Create the test database** (only once):

```bash
# From the host machine (requires the db container to be running)
psql -h localhost -p 5433 -U project_user -d postgres \
  -c "CREATE DATABASE sims_test;"
```

**Step 3 — Run migrations on the test database:**

```bash
# From the host machine
php artisan migrate --env=testing

# Or from inside the container
docker compose exec app php artisan migrate --env=testing
```

**Step 4 — Run the tests:**

```bash
# All tests
php artisan test --env=testing

# Unit tests only
php artisan test --testsuite=Unit --env=testing

# Feature tests only
php artisan test --testsuite=Feature --env=testing

# With coverage report (requires Xdebug or PCOV)
php artisan test --env=testing --coverage
```

**Available test suites:**

| Suite | Location | Coverage |
|---|---|---|
| Unit | `tests/Unit/` | Geofencing evaluator, polygon normalizer |
| Feature – MultiTenancy | `tests/Feature/MultiTenancy/` | Data isolation of bookings, roles, vehicles, tickets and users across tenants |
| Feature – Tenancy | `tests/Feature/Tenancy/` | Tenant identification, auto-seeding, logging, active tenant check |
| Feature – Geofencing | `tests/Feature/Geofencing/` | Geofence API, guards |
| Feature – CRUD | `tests/Feature/` | Users (`UserCrudTest`), roles (`RoleCrudTest`) |
| Feature – Console | `tests/Feature/Console/` | Tenant Artisan commands |

> Tests reset the database on every run via `RefreshDatabase` — no manual cleanup needed.

---

### 4.2 Frontend Tests (Vitest)

**Prerequisites:** dependencies installed (`npm install`)

```bash
cd project-sims-frontend

# Run tests once
npm test

# Watch mode (re-runs on file changes)
npm run test:watch
```

**Vitest configuration** (`vitest.config.ts`):

- Environment: `jsdom` (simulates browser DOM)
- Includes all `src/**/*.test.ts` files

**Available tests:**

| File | What it tests |
|---|---|
| `GeofenceEventBadge.test.ts` | Rendering of the geofence event badge component |
| `useGeofenceValidation.test.ts` | Geofence validation logic (composable) |

---

## 5. Test Credentials

Use the `organization` field (tenant slug) when logging in (`/central/login`).

### SIMS Corp (`organization: sims-corp`)

| Email | Password | Role |
|---|---|---|
| `client@test.com` | `password` | Client |
| `maria@simscorp.com` | `password` | Client |
| `carlos@simscorp.com` | `password` | Client |
| `admin@test.com` | `password` | Admin |

### EcoMove (`organization: ecomove`)

| Email | Password | Role | Note |
|---|---|---|---|
| `laura@ecomove.es` | `password` | Client | |
| `jorge@ecomove.es` | `password` | Client | |
| `ana@ecomove.es` | `password` | Client | Inactive — expected to fail |

---

## 6. Sprint 6 Changes

### 6.1 Resolved Bugs (cross-team testing)

Testing for this sprint was carried out by an external team. Below is a summary of the issues found and their resolution status:

#### Admin panel

| Bug found | Status |
|---|---|
| User profile displayed text in English | Fixed — full translations added to the admin section |
| No show/hide password button | Fixed — added to all password fields |
| Main dashboard was in English | Fixed — translated to Catalan |
| Two overlapping buttons to access the profile | Fixed — removed the dropdown; profile photo → profile page; logout icon placed next to it |
| "Create booking" screen had a wrong title (it was actually booking detail info) | Fixed — renamed to "Booking information" |
| Ticket chat displayed in English | Fixed — translated to Catalan |
| 500 error when creating a new Tenant | Pending investigation |
| Vehicle location not linkable to IoT sensors | Pending — will be resolved when the IoT subsystem is configured |

#### Client view

| Bug found | Status |
|---|---|
| Vehicles not loading on the map | Fixed — clients can now see vehicles and complete bookings with payment |
| Bookings not automatically cancelled when expired | Fixed — the scheduler now cancels expired pending bookings |
| Route `/settings` did not exist, returning 404 | Fixed — redirects directly to `/profile` |
| Sensors page visible to clients with no context | Kept temporarily for IoT testing purposes |

---

### 6.2 New Features

#### Stripe Payments

Full payment flow integrated for bookings:

1. Client creates a booking → status set to `pending` + `payment_status=unpaid`
2. A Stripe checkout session is created: `POST /api/reservations/{id}/checkout-session`
3. The user is redirected to Stripe and completes the payment
4. Stripe notifies the backend via webhook → booking marked as `payment_status=paid`
5. Activation (`POST /api/reservations/{id}/activate`) is only allowed once payment is confirmed

**Local Stripe webhook testing:**

```bash
stripe listen --forward-to http://localhost:8001/api/payments/stripe/webhook
```

#### Subdomain Routing (multi-tenant)

Each tenant has its own subdomain:

- **Production:** `sims-corp.grup1-sims.com`, `ecomove.grup1-sims.com`
- **Local (dev):** `sims-corp.localhost:5173`, `ecomove.localhost:5173`

The backend resolves the tenant by subdomain via `InitializeTenancyByDomainOrHeader`. The `X-Tenant` header is still supported as a fallback.

#### Public Company Registration

New flow for companies that want to join the platform:

- Public form at: `/register-company`
- The company fills in name, slug, tax ID and contact details
- The request is set to `pending` and the SuperAdmin is notified by email
- The SuperAdmin approves or rejects from `/admin/tenant-requests`
- On approval, the tenant is created and schemas are migrated automatically

#### Geofencing

Administrators can define geographic zones directly on the map. When a vehicle enters or exits one of these zones, the event is automatically recorded. The admin can create, edit and delete zones using a visual map editor, and review the event history for each zone.

#### AI Assistant (Gemini)

The `gemini-2.0-flash-lite` model is integrated into the platform as a contextual assistant, available to clients from any screen via a chat widget. It can answer questions about bookings, vehicles and how the app works without opening a support ticket.  
The API key is configured via a GitHub Actions secret (`IA_API_KEY`) and injected into the server on every deployment.

#### Automatic Booking Cancellation (scheduler)

The Laravel scheduler runs every minute and cancels `pending` bookings that have exceeded their `activation_deadline`.

---

## 7. CI/CD — GitHub Actions

Both repositories have workflows that trigger on every push to the `develop` branch.

### Backend (`.github/workflows/deploy.yml`)

| Step | Action |
|---|---|
| Checkout | Clones the code |
| SSH connectivity | Verifies server access (port 1303) |
| Fix permissions | `chown` before SCP upload |
| SCP upload | Uploads `app/`, `config/`, `database/`, `routes/`, `public/`... |
| Inject AI variables | Writes `IA_API_URL`, `IA_MODEL`, `IA_API_KEY` to the server's `.env` |
| Remote deploy | `composer install`, recovers stopped DB/App containers, `migrate`, `config:cache`, `route:cache`, `view:cache` |
| Verify AI config | Checks that `IA_API_KEY` is present in the config cache — deploy fails if not |

**Required GitHub Secrets:**

| Secret | Description |
|---|---|
| `DEPLOY_HOST_IP` | VPS IP address |
| `DEPLOY_USER` | SSH username |
| `DEPLOY_SSH_KEY` | SSH private key |
| `IA_API_KEY` | Google Gemini API key |

### Frontend (`.github/workflows/deploy.yml`)

| Step | Action |
|---|---|
| Checkout | Clones the code |
| Build | `npm ci && npm run build` |
| Prepare remote folder | Creates `/var/www/sims-front/dist` and clears previous contents |
| SCP upload | Uploads the contents of `dist/` |
| Nginx reload | `sudo systemctl reload nginx` |

---

## 8. Subdomain Deployment

### Server Requirements

- Wildcard DNS configured: `*.grup1-sims.com` → VPS IP
- Nginx with `server_name .grup1-sims.com;` (accepts all subdomains)
- Wildcard SSL certificate (Let's Encrypt with DNS challenge recommended)
- PostgreSQL accessible
- Docker and Docker Compose v2

### Nginx — Reference Configuration

```nginx
server {
    listen 443 ssl;
    server_name .grup1-sims.com;

    ssl_certificate     /etc/letsencrypt/live/grup1-sims.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grup1-sims.com/privkey.pem;

    root /var/www/sims-front/dist;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API to backend
    location /api/ {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Approving a New Tenant (production)

1. The company fills in the form at `/register-company`
2. The SuperAdmin goes to `/admin/tenant-requests` and approves the request
3. The backend automatically creates the PostgreSQL schema, runs migrations and registers the subdomain in the `domains` table
4. The tenant is immediately accessible at `<slug>.grup1-sims.com`

### Full Reset (destructive — restricted use)

```bash
docker compose exec app php artisan migrate:fresh --force
docker compose exec app php artisan db:seed \
  --class="Database\\Seeders\\TenantsSeeder" --no-interaction
docker compose exec app php artisan tenants:migrate --no-interaction
docker compose exec app php artisan tenants:seed \
  --class="Database\\Seeders\\DatabaseSeeder" --force --no-interaction
```

---

## 9. Security

### Applied Measures

| Area | Measure |
|---|---|
| **Authentication** | Laravel Sanctum (HTTP-only tokens) |
| **Authorisation** | Per-resource policies + per-tenant roles and permissions (spatie/laravel-permission) |
| **Data isolation** | Separate PostgreSQL schema per tenant — queries always run within the active tenant context |
| **CORS** | Configured in `config/cors.php` to allow only authorised origins |
| **Payments** | Stripe handles all card data — the backend never stores PAN data |
| **Webhooks** | `STRIPE_WEBHOOK_SECRET` signature validated on every incoming event |
| **CI/CD secrets** | Sensitive keys (`IA_API_KEY`, SSH) managed via GitHub Secrets, never committed to code |
| **Rate limiting** | Laravel default middleware applied to all API routes |
| **File permissions** | `storage/` and `bootstrap/cache/` set to 777 inside the container; `dist/` served by Nginx with 755 |

### Pending / Future Improvements

| Area | Recommended improvement |
|---|---|
| **SSL** | Automated wildcard certificate with Certbot + DNS challenge |
| **Advanced rate limiting** | Custom per-IP and per-token limits on sensitive endpoints |
| **2FA** | Two-factor authentication for Admin and SuperAdmin roles |
| **Audit logging** | Log of critical actions (user creation/deletion, tenant approval) |
| **Penetration testing** | API endpoint review with OWASP ZAP or Burp Suite |
| **Dependency scanning** | Dependabot or `npm audit` / `composer audit` in CI |
| **Security headers** | CSP, X-Frame-Options, HSTS configured in Nginx |
| **Backups** | Automated backups for PostgreSQL and MongoDB |
