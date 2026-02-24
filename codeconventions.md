# Guia de Desenvolupament: SIMS (Sprint 5)

Aquest document defineix les regles acordades per l'equip per al desenvolupament d'un codi llegible, mantenible i segur durant el cinquè sprint del projecte.

---

## 1. Informació General del Projecte
* **Nom del projecte:** Fleetly
* **Tecnologies principals:** * **Frontend:** Vue.js 3
    * **Backend:** Laravel (PHP)
    * **DBMS:** PostgreSQL (PgAdmin)
* **Responsables:** Jordi Arnau, Xavier Chao, Sergi Obiol.
* **Última revisió:** 23/02/2026

---

## 2. Principis Generals
* **Prioritat:** Llegibilitat i optimització del rendiment.
* **Idioma:** * Català per a missatges públics i documentació principal.
    * Anglès per a la nomenclatura del codi (variables, funcions, classes).
* **Comentaris:** Només quan siguin estrictament necessaris per aclarir lògica complexa. Han de ser breus i directes.
* **Metodologia:** Treball basat en el marc de treball SCRUM.

---

## 3. Estructura del Projecte

S'utilitzarà una arquitectura clara i estàndard, separant el client del servidor en repositoris o carpetes independents.

### Estructura Frontend (Vue 3 + Vite)
```text
src/
├─ assets/        # Fitxers estàtics (imatges, estils globals)
├─ components/    # Components de la interfície reutilitzables
├─ composables/   # Lògica compartida i Hooks (useAuth.ts)
├─ services/      # Integracions amb l'API i serveis externs
├─ views/         # Pàgines de l'aplicació (LoginPage.vue, etc.)
├─ router/        # Configuració de les rutes de navegació
├─ store/         # Gestió de l'estat global (Pinia)
└─ main.ts
```

### Estructura Backend (Laravel)
```
app/
├─ Http/
│   ├─ Controllers/  # Controladors de l'API
│   ├─ Middleware/   # Filtres de peticions
│   └─ Requests/     # Validacions de dades d'entrada
├─ Models/           # Representació de les taules de la BD
├─ Services/         # Lògica de negoci específica
└─ Providers/        # Registre de serveis del sistema

database/
└─ migrations/       # Definicions de l'esquema de la BD
```

### 4. Convencions de Nomenclatura
## 4.1 Fitxers i Carpetes
Components Vue: PascalCase (Ex: VehicleList.vue).

Controladors/Models: PascalCase (Ex: RentalController.php).

Variables i Funcions: camelCase (Ex: calculateTotalCost).

Constants: SCREAMING_SNAKE_CASE (Ex: TAX_RATE).

Rutes i URLs: kebab-case (Ex: /api/v1/fleet-management).

Directoris i Taules BD: snake_case (Ex: user_profiles).

## 4.2 Variables i Booleans
Idioma: Anglès per a tots els identificadors de codi.

Semàntica: Noms descriptius; evitar lletres soles excepte en iteradors puntuals.

Booleans: Sempre amb prefix d'estat (is, has, can, should). Exemple: isActive, hasSubscription.

### 5. Control de Versions (Git)
Branca principal: main.

Branques de funcionalitat: feature/F-X (X= numero de la tasca) (Ex: feature/F-1).

Commits:

Idioma: Català.

Format: Tipus: Descripció curta (Ex: fix/F-1-ArreglarLogin).

Pull Requests: Requereixen una descripció clara i la revisió de conformitat amb aquestes convencions abans de ser fusionades.

Actualització de branques: Abans de fer el merge, la branca feature/F-X ha d'estar actualitzada amb l'última versió de Develop per evitar conflictes d'últim moment.

Revisió de Codi (Code Review): Cap Pull Request serà fusionada sense l'aprovació d'almenys un altre responsable.

