# Components Laravel - Project SIMS Backend

## Arquitectura de Base de Dades

L'arquitectura segueix un **patró modular multi-tenant** on cada entitat de negoci (vehicles, reserves, tickets) està aïllada per tenant.

Utilitza:

- **PostgreSQL** per dades relacionals
- **MongoDB** per ubicacions GPS en temps real

Les relacions són gestionades per **Eloquent ORM** amb **soft deletes** per mantenir historial.

---

# Components Globals

| Component | Ubicació | Funció |
|-----------|----------|--------|
| Controllers | `app/Http/Controllers/` | Gestionen lògica de negoci i responen peticions HTTP |
| Models | `app/Models/` | Representen taules BD amb Eloquent ORM |
| Migrations | `database/migrations/` | Defineixen estructura de base de dades versionada |
| Seeders | `database/seeders/` | Omplen BD amb dades inicials |
| Routes API | `routes/api.php` | Defineixen endpoints REST amb middleware auth |
| Middleware | `app/Http/Middleware/` | Filtren peticions (autenticació, CORS) |
| Policies | `app/Policies/` | Autoritzen operacions segons rol i tenant |
| Form Requests | `app/Http/Requests/` | Validen dades entrada |
| Service Providers | `app/Providers/` | Registren serveis al contenidor |
| Services | `app/Services/` | Lògica negoci reutilitzable |
| Config | `config/` | Configuracions (database, auth, sanctum) |
| Tests | `tests/Feature/`, `tests/Unit/` | Tests automatitzats |

---

# Mòdul: Autenticació i Usuaris

## Components

| Component | Fitxer | Funció |
|-----------|--------|--------|
| Controller | `AuthController.php` | Login, logout amb Sanctum tokens |
| Controller | `UserController.php` | CRUD usuaris amb validació rols |
| Model | `User.php` | Usuari amb Sanctum i Spatie roles |
| Policy | `UserPolicy.php` | Autoritza gestió usuaris per rol |
| Migration | `create_users_table.php` | Taula users amb camps auth |
| Migration | `create_personal_access_tokens_table.php` | Tokens Sanctum |
| Seeder | `DatabaseSeeder.php` | Usuaris prova (admin, client, maintenance) |

## Relacions

- `User` hasMany `Reservations`
- `User` hasMany `Tickets`
- `User` belongsTo `Tenant`

---

# Mòdul: Vehicles

## Components

| Component | Fitxer | Funció |
|-----------|--------|--------|
| Controller | `VehicleController.php` | CRUD vehicles amb filtratge tenant |
| Model | `Vehicle.php` | Vehicle amb tenant_id i soft deletes |
| Policy | `VehiclePolicy.php` | Autoritza operacions per tenant |
| Request | `StoreVehicleRequest.php` | Valida creació vehicles |
| Request | `UpdateVehicleRequest.php` | Valida actualització vehicles |
| Service | `VehicleLocationService.php` | Obté ubicacions GPS de MongoDB |
| Migration | `create_vehicles_table.php` | Taula vehicles amb FK tenant |
| Seeder | `TestDataSeeder.php` | Vehicles de prova amb ubicacions |

## Relacions

- `Vehicle` hasMany `Reservations`
- `Vehicle` belongsTo `Tenant`

---

# Mòdul: Reserves (Reservations)

## Components

| Component | Fitxer | Funció |
|-----------|--------|--------|
| Controller | `ReservationController.php` | Gestió reserves usuaris |
| Controller | `AdminReservationController.php` | Gestió reserves admins |
| Model | `Reservation.php` | Reserva amb estats i FK |
| Policy | `ReservationPolicy.php` | Autoritza reserves per rol |
| Migration | `create_reservations_table.php` | Taula reserves amb estats |

## Relacions

- `Reservation` belongsTo `User`
- `Reservation` belongsTo `Vehicle`
- `Reservation` hasOne `Trip`
- `Reservation` belongsTo `Tenant`

---

# Mòdul: Viatges (Trips)

## Components

| Component | Fitxer | Funció |
|-----------|--------|--------|
| Model | `Trip.php` | Trajecte amb ubicacions i cost |
| Migration | `create_trips_table.php` | Taula trips amb timestamps |
| Migration | `add_status_to_trips_table.php` | Afegeix estats viatge |

## Relacions

- `Trip` belongsTo `Reservation`
- `Trip` belongsTo `User`

---

# Mòdul: Tickets (Suport)

## Components

| Component | Fitxer | Funció |
|-----------|--------|--------|
| Controller | `TicketController.php` | CRUD tickets suport |
| Controller | `TicketMessageController.php` | Missatges dins tickets |
| Model | `Ticket.php` | Ticket amb estats i prioritat |
| Model | `TicketMessage.php` | Missatge de ticket |
| Policy | `TicketPolicy.php` | Autoritza tickets per rol |
| Migration | `create_tickets_table.php` | Taula tickets |
| Migration | `create_ticket_messages_table.php` | Missatges threaded |

## Relacions

- `Ticket` belongsTo `User`
- `Ticket` hasMany `TicketMessages`
- `Ticket` belongsTo `Vehicle` (opcional)

---

# Mòdul: Tenants (Multi-tenant)

## Components

| Component | Fitxer | Funció |
|-----------|--------|--------|
| Controller | `TenantController.php` | CRUD tenants (super admin) |
| Model | `Tenant.php` | Organització / empresa SaaS |
| Migration | `create_tenants_table.php` | Taula tenants |
| Migration | `add_tenant_id_to_business_tables.php` | FK tenant a totes taules |
| Seeder | `TenantsSeeder.php` | Tenants de prova |

## Relacions

- `Tenant` hasMany `Users`
- `Tenant` hasMany `Vehicles`
- `Tenant` hasMany `Reservations`

---

# Mòdul: Rols i Permisos

## Components

| Component | Fitxer | Funció |
|-----------|--------|--------|
| Controller | `RoleController.php` | Gestió rols (Admin, Client, Maintenance) |
| Controller | `PermissionController.php` | Llistat permisos disponibles |
| Policy | `RolePolicy.php` | Autoritza gestió rols |
| Migration | `create_permission_tables.php` | Taules Spatie Permission |
| Seeder | `PermissionsSeeder.php` | Permisos (vehicles.view, etc) |
| Seeder | `RolesSeeder.php` | Rols amb permisos assignats |
| Package | Spatie Laravel Permission | Sistema rols granular |

---

# Packages Utilitzats

| Package | Funció |
|--------|--------|
| Laravel Sanctum | Autenticació API amb tokens |
| Spatie Laravel Permission | Sistema rols i permisos |
| MongoDB Laravel | Driver MongoDB per ubicacions GPS |

---

# Comandes Artisan Principals

```bash
# Migracions
php artisan migrate
php artisan migrate:fresh --seed

# Seeders
php artisan db:seed

# Testing
php artisan test

# Cache
php artisan config:cache
php artisan route:cache

# Desenvolupament
php artisan tinker
php artisan serve
