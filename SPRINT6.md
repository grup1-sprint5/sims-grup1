# Sprint 6 — Chaos & Delivery

Documentació final del projecte **SIMS** (Sistema Intel·ligent de Mobilitat Sostenible).  
Cobreix la configuració completa de l'entorn de desenvolupament, l'entorn de testeig, els canvis del Sprint 6 i el pipeline de CI/CD.

---

## Índex

1. [Visió general del sistema](#1-visió-general-del-sistema)
2. [Arquitectura](#2-arquitectura)
3. [Setup entorn de desenvolupament](#3-setup-entorn-de-desenvolupament)
   - [Backend (Laravel)](#31-backend-laravel)
   - [Frontend (Vue 3)](#32-frontend-vue-3)
   - [Subdominis locals](#33-subdominis-locals)
4. [Setup entorn de testeig](#4-setup-entorn-de-testeig)
   - [Tests de backend (PHPUnit / Pest)](#41-tests-de-backend-phpunit--pest)
   - [Tests de frontend (Vitest)](#42-tests-de-frontend-vitest)
5. [Credencials de prova](#5-credencials-de-prova)
6. [Canvis del Sprint 6](#6-canvis-del-sprint-6)
   - [Bugs resolts (testing inter-equips)](#61-bugs-resolts-testing-inter-equips)
   - [Noves funcionalitats](#62-noves-funcionalitats)
7. [CI/CD — GitHub Actions](#7-cicd--github-actions)
8. [Desplegament amb subdominis](#8-desplegament-amb-subdominis)
9. [Seguretat](#9-seguretat)

---

## 1. Visió general del sistema

SIMS és una plataforma multi-tenant de gestió de flotes de vehicles elèctrics de lloguer.  
Cada empresa (tenant) disposa d'un subdomini propi, base de dades aïllada i interfície diferenciada per a clients i administradors.

| Rol | Accés | Descripció |
|---|---|---|
| **Client** | `<tenant>.grup1-sims.com` | Reserva vehicles, paga amb Stripe, gestiona tiquets |
| **Admin** | `<tenant>.grup1-sims.com/admin` | Gestiona usuaris, vehicles, reserves i tiquets del tenant |
| **SuperAdmin** | `grup1-sims.com/admin/tenants` | Aprova/rebutja peticions de nous tenants |

---

## 2. Arquitectura

```
┌─────────────────────────────────────────────────────┐
│  Frontend  ·  Vue 3 + TypeScript + Vite              │
│  Tailwind CSS · Pinia · Vue Router · Leaflet         │
│  Subdomini per tenant: <slug>.grup1-sims.com         │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS (Nginx)
┌──────────────────────▼──────────────────────────────┐
│  Backend  ·  Laravel 12 + PHP 8.2                   │
│  Multi-tenancy per subdomini (stancl/tenancy)        │
│  API REST  ·  Sanctum Auth  ·  Stripe Webhooks       │
└────────┬──────────────────┬───────────────────────  ┘
         │                  │
┌────────▼────────┐  ┌──────▼──────────┐
│  PostgreSQL 15  │  │  MongoDB 6       │
│  Dades per      │  │  Ubicacions de  │
│  schema/tenant  │  │  vehicles (IoT) │
└─────────────────┘  └─────────────────┘
```

**Repositoris:**
- Backend: `project-sims-backend/`
- Frontend: `project-sims-frontend/`

---

## 3. Setup entorn de desenvolupament

### 3.1 Backend (Laravel)

**Requisits previs:** Docker Desktop, Docker Compose v2

```bash
cd project-sims-backend

# 1. Copia el fitxer d'entorn
cp .env.example .env

# 2. Arrenca els contenidors (app, db PostgreSQL, pgAdmin, MongoDB)
docker compose up -d --build

# 3. Instal·la dependències PHP
docker compose exec app composer install --no-interaction

# 4. Genera la clau de l'aplicació
docker compose exec app php artisan key:generate --force

# 5. Executa les migracions centrals (tenants, domains, cache...)
docker compose exec app php artisan migrate --force

# 6. Executa les migracions de cada tenant (vehicles, reserves, tiquets...)
docker compose exec app php artisan tenants:migrate --no-interaction

# 7. Pobla la base de dades amb dades de demo
docker compose exec app php artisan tenants:seed \
  --class="Database\\Seeders\\DatabaseSeeder" --force --no-interaction
```

**Serveis disponibles un cop arrencats:**

| Servei | URL local |
|---|---|
| API Backend | http://localhost:8001 |
| pgAdmin | http://localhost:8080 |
| MongoDB | localhost:27017 |

**Arrenca el scheduler** (cancel·lació automàtica de reserves):

```bash
docker compose up -d scheduler
```

**Variables d'entorn destacades (`.env`):**

```env
APP_URL=http://localhost:8001

DB_HOST=db
DB_PORT=5432
DB_DATABASE=project_sims
DB_USERNAME=project_user
DB_PASSWORD=supersecret

MONGODB_URI=mongodb://mongo:27017
MONGODB_DATABASE=cluster-iot

# Stripe (pagaments)
STRIPE_KEY=pk_test_xxx
STRIPE_SECRET=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_SUCCESS_URL=http://localhost:5173/client/bookings
STRIPE_CANCEL_URL=http://localhost:5173/client/bookings

# IA (assistent Gemini)
IA_API_URL=https://generativelanguage.googleapis.com/v1beta/openai
IA_API_KEY=xxx
IA_MODEL=gemini-2.0-flash-lite
```

---

### 3.2 Frontend (Vue 3)

**Requisits previs:** Node.js >= 20.19, npm

```bash
cd project-sims-frontend

# Instal·la dependències
npm install

# Mode desenvolupament (hot-reload)
npm run dev
```

El frontend arrenca a **http://localhost:5173**.

**Build de producció:**

```bash
npm run build
# Els estàtics es generen a dist/
```

**Variables d'entorn del frontend (`.env` o `vite.config.ts`):**

El frontend resol el tenant per subdomini; en local, configura els hosts (vegeu [secció 3.3](#33-subdominis-locals)).

---

### 3.3 Subdominis locals

Per provar el routing multi-tenant en local cal afegir entrades al fitxer `hosts`.

**Linux / macOS** (`/etc/hosts`) o **Windows** (`C:\Windows\System32\drivers\etc\hosts`, com a Administrador):

```
127.0.0.1  sims-corp.localhost
127.0.0.1  ecomove.localhost
```

Accedeix als tenants de demo:

| Tenant | URL local frontend |
|---|---|
| SIMS Corp | http://sims-corp.localhost:5173 |
| EcoMove | http://ecomove.localhost:5173 |

> Elimina les entrades quan ja no les necessitis.

---

## 4. Setup entorn de testeig

### 4.1 Tests de backend (PHPUnit / Pest)

Els tests utilitzen una base de dades PostgreSQL **separada** (`sims_test`) per no afectar mai les dades de desenvolupament.

**Pas 1 — Crea el fitxer d'entorn de testeig:**

```bash
cp .env.testing.example .env.testing
```

Contingut mínim de `.env.testing`:

```env
APP_ENV=testing

# Des de la màquina host:
DB_HOST=localhost
DB_PORT=5433
DB_DATABASE=sims_test
DB_USERNAME=project_user
DB_PASSWORD=change_me

# Des de dins del contenidor Docker:
# DB_HOST=db
# DB_PORT=5432

BROADCAST_CONNECTION=null
CACHE_STORE=array
QUEUE_CONNECTION=sync
SESSION_DRIVER=array
MAIL_MAILER=array
```

**Pas 2 — Crea la base de dades de test** (només una vegada):

```bash
# Des de la màquina host (requereix que el contenidor db estigui arrencat)
psql -h localhost -p 5433 -U project_user -d postgres \
  -c "CREATE DATABASE sims_test;"
```

**Pas 3 — Executa les migracions en la BD de test:**

```bash
# Des de la màquina host
php artisan migrate --env=testing

# O des de dins del contenidor
docker compose exec app php artisan migrate --env=testing
```

**Pas 4 — Executa els tests:**

```bash
# Tots els tests
php artisan test --env=testing

# Només tests unitaris
php artisan test --testsuite=Unit --env=testing

# Només tests de funcionalitat
php artisan test --testsuite=Feature --env=testing

# Amb detall de cobertura (si tens Xdebug o PCOV)
php artisan test --env=testing --coverage
```

**Suites de tests disponibles:**

| Suite | Localització | Contingut |
|---|---|---|
| Unit | `tests/Unit/` | Geofencing evaluator, polygon normalizer |
| Feature – MultiTenancy | `tests/Feature/MultiTenancy/` | Aïllament de reserves, rols, vehicles, tiquets i usuaris entre tenants |
| Feature – Tenancy | `tests/Feature/Tenancy/` | Identificació de tenant, seed automàtic, logging, tenant actiu |
| Feature – Geofencing | `tests/Feature/Geofencing/` | API de geofences, guards |
| Feature – CRUD | `tests/Feature/` | Usuaris (`UserCrudTest`), rols (`RoleCrudTest`) |
| Feature – Console | `tests/Feature/Console/` | Comandes Artisan de tenants |

> Els tests reinicialitzen la BD en cada execució via `RefreshDatabase` — no cal netejar manualment.

---

### 4.2 Tests de frontend (Vitest)

**Requisits:** dependències instal·lades (`npm install`)

```bash
cd project-sims-frontend

# Executa els tests una vegada
npm test

# Mode watch (re-executa en canviar fitxers)
npm run test:watch
```

**Configuració de Vitest** (`vitest.config.ts`):

- Entorn: `jsdom` (simula DOM del navegador)
- Inclou tots els fitxers `src/**/*.test.ts`

**Tests disponibles:**

| Fitxer | Què prova |
|---|---|
| `GeofenceEventBadge.test.ts` | Renderitzat del component badge d'esdeveniments de geofence |
| `useGeofenceValidation.test.ts` | Lògica de validació de geofences (composable) |

---

## 5. Credencials de prova

Utilitza l'`organization` (slug del tenant) al login (`/central/login`).

### SIMS Corp (`organization: sims-corp`)

| Email | Contrasenya | Rol |
|---|---|---|
| `client@test.com` | `password` | Client |
| `maria@simscorp.com` | `password` | Client |
| `carlos@simscorp.com` | `password` | Client |
| `admin@test.com` | `password` | Admin |

### EcoMove (`organization: ecomove`)

| Email | Contrasenya | Rol | Nota |
|---|---|---|---|
| `laura@ecomove.es` | `password` | Client | |
| `jorge@ecomove.es` | `password` | Client | |
| `ana@ecomove.es` | `password` | Client | Inactiva — ha de fallar |

---

## 6. Canvis del Sprint 6

### 6.1 Bugs resolts (testing inter-equips)

El testing d'aquest sprint va ser realitzat per un equip extern. A continuació el resum dels problemes detectats i l'estat de resolució:

#### Panell d'administració

| Bug detectat | Estat |
|---|---|
| Perfil d'usuari mostrava textos en anglès | Resolt — traduccions completes a l'apartat admin |
| Faltava botó per mostrar/ocultar contrasenya | Resolt — afegit a tots els camps de password |
| El tauler principal estava en anglès | Resolt — traduït al català |
| Dos botons solapats per accedir al perfil | Resolt — eliminat el desplegable; foto de perfil → perfil; icona de logout al costat |
| Pantalla "Create booking" amb títol erroni (era informació d'una reserva existent) | Resolt — renomenant a "Informació de la reserva" |
| Chat dels tiquets en anglès | Resolt — traduït al català |
| Error 500 en crear un nou Tenant | Pendent d'investigació |
| Ubicació de vehicles no vinculable als sensors IoT | Pendent — es resoldrà quan el subsistema IoT estigui configurat |

#### Vista de client

| Bug detectat | Estat |
|---|---|
| Els vehicles no es carregaven al mapa | Resolt — els clients veuen vehicles i poden fer reserves i pagaments |
| Les reserves no s'anul·laven automàticament en caducar | Resolt — el scheduler ara cancel·la reserves expirades |
| Ruta `/settings` inexistent generava 404 | Resolt — redirecció a `/profile` |
| Pàgina de sensors visible per als clients sense context | Mantingut temporalment per a proves de IoT |

---

### 6.2 Noves funcionalitats

#### Pagament amb Stripe

Integració completa del flux de pagament per a reserves:

1. El client crea una reserva → queda en estat `pending` + `payment_status=unpaid`
2. Es genera una sessió de checkout Stripe: `POST /api/reservations/{id}/checkout-session`
3. L'usuari és redirigit a Stripe i completa el pagament
4. Stripe notifica el backend via webhook → reserva marcada com `payment_status=paid`
5. L'activació (`POST /api/reservations/{id}/activate`) només es permet si el pagament s'ha completat

**Test local de webhook Stripe:**

```bash
stripe listen --forward-to http://localhost:8001/api/payments/stripe/webhook
```

#### Routing per subdomini (multi-tenant)

Cada tenant disposa del seu propi subdomini:

- **Producció:** `sims-corp.grup1-sims.com`, `ecomove.grup1-sims.com`
- **Local (dev):** `sims-corp.localhost:5173`, `ecomove.localhost:5173`

El backend resol el tenant per subdomini via `InitializeTenancyByDomainOrHeader`. La capçalera `X-Tenant` segueix suportada com a fallback.

#### Registre públic de nous tenants

Nou flux per a empreses que volen unir-se a la plataforma:

- Formulari públic: `/register-company`
- L'empresa omple nom, slug, CIF i dades de contacte
- La petició queda en estat `pending` i es notifica al SuperAdmin per email
- El SuperAdmin aprova/rebutja des de `/admin/tenant-requests`
- En aprovar, es crea el tenant i es migren els esquemes automàticament

#### Assistent IA (Gemini)

Integrat a la plataforma el model `gemini-2.0-flash-lite` com a assistent contextual.  
La clau d'API es configura via secret de GitHub Actions (`IA_API_KEY`) i s'injecta al servidor en cada desplegament.

#### Cancel·lació automàtica de reserves (scheduler)

El scheduler de Laravel executa cada minut i cancel·la les reserves `pending` que hagin superat el `activation_deadline`.

---

## 7. CI/CD — GitHub Actions

Tots dos repositoris disposen de workflows que s'executen en fer push a la branca `develop`.

### Backend (`.github/workflows/deploy.yml`)

| Pas | Acció |
|---|---|
| Checkout | Clona el codi |
| Connectivitat SSH | Verifica accés al servidor (port 1303) |
| Fix permisos | `chown` previ al SCP |
| Upload SCP | Puja `app/`, `config/`, `database/`, `routes/`, `public/`... |
| Injecta variables IA | Escriu `IA_API_URL`, `IA_MODEL`, `IA_API_KEY` al `.env` del servidor |
| Deploy remot | `composer install`, recupera contenidors DB/App si estan aturats, `migrate`, `config:cache`, `route:cache`, `view:cache` |
| Verifica IA | Comprova que `IA_API_KEY` ha quedat al config cache (falla el deploy si no) |

**Secrets necessaris a GitHub:**

| Secret | Descripció |
|---|---|
| `DEPLOY_HOST_IP` | IP del VPS |
| `DEPLOY_USER` | Usuari SSH |
| `DEPLOY_SSH_KEY` | Clau privada SSH |
| `IA_API_KEY` | Clau API de Google Gemini |

### Frontend (`.github/workflows/deploy.yml`)

| Pas | Acció |
|---|---|
| Checkout | Clona el codi |
| Build | `npm ci && npm run build` |
| Prepara carpeta remota | Crea `/var/www/sims-front/dist` i neteja contingut anterior |
| Upload SCP | Puja el contingut de `dist/` |
| Nginx reload | `sudo systemctl reload nginx` |

---

## 8. Desplegament amb subdominis

### Requisits del servidor

- DNS wildcard configurat: `*.grup1-sims.com` → IP del VPS
- Nginx amb `server_name .grup1-sims.com;` (accepta tots els subdominis)
- Certificat SSL wildcard (Let's Encrypt amb DNS challenge recomanat)
- PostgreSQL accessible
- Docker i Docker Compose v2

### Nginx — configuració de referència

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

    # Proxy API al backend
    location /api/ {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Aprovació d'un nou tenant (producció)

1. L'empresa omple el formulari a `/register-company`
2. El SuperAdmin accedeix a `/admin/tenant-requests` i aprova la petició
3. El backend crea automàticament el schema PostgreSQL, executa les migracions i registra el subdomini a la taula `domains`
4. El tenant és accessible a `<slug>.grup1-sims.com` immediatament

### Reset complet (destructor — ús restringit)

```bash
docker compose exec app php artisan migrate:fresh --force
docker compose exec app php artisan db:seed \
  --class="Database\\Seeders\\TenantsSeeder" --no-interaction
docker compose exec app php artisan tenants:migrate --no-interaction
docker compose exec app php artisan tenants:seed \
  --class="Database\\Seeders\\DatabaseSeeder" --force --no-interaction
```

---

## 9. Seguretat

### Mesures aplicades

| Àmbit | Mesura |
|---|---|
| **Autenticació** | Laravel Sanctum (tokens HTTP-only) |
| **Autorització** | Policies per recurs + rols/permisos per tenant (spatie/laravel-permission) |
| **Aïllament de dades** | Schema PostgreSQL separat per tenant — consultes sempre dins del context del tenant actiu |
| **CORS** | Configurat a `config/cors.php` per permetre només orígens autoritzats |
| **Pagaments** | Stripe gestiona les dades de targeta — el backend mai emmagatzema dades PAN |
| **Webhooks** | Validació de signatura `STRIPE_WEBHOOK_SECRET` a cada evento rebut |
| **Secrets CI/CD** | Claus sensibles (`IA_API_KEY`, SSH) gestionades via GitHub Secrets, mai al codi |
| **Rate limiting** | Middleware de Laravel per defecte a les rutes API |
| **Permisos de fitxers** | `storage/` i `bootstrap/cache/` amb 777 en contenidor; `dist/` amb 755 servit per Nginx |

### Mesures pendents / millores futures

| Àmbit | Millora recomanada |
|---|---|
| **SSL** | Certificat wildcard automatitzat amb Certbot + DNS challenge |
| **Rate limiting avançat** | Límits per IP i per token personalitzats per endpoint sensible |
| **2FA** | Autenticació de dos factors per a rols Admin i SuperAdmin |
| **Auditoria** | Log d'accions crítiques (creació/eliminació d'usuaris, aprovació de tenants) |
| **Penetration testing** | Revisió d'endpoints d'API amb OWASP ZAP o Burp Suite |
| **Dependency scanning** | Dependabot o `npm audit` / `composer audit` en CI |
| **Headers de seguretat** | CSP, X-Frame-Options, HSTS configurats a Nginx |
| **Backups** | Còpies de seguretat automatitzades de PostgreSQL i MongoDB |
