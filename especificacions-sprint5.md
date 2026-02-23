# Objectiu Principal

Desenvolupar totes les funcionalitats requerides pel client per assolir una versió "alfa" completa.

## Entorn Servidor (Back-end)

- Continuar implementant endpoints de l'API de Laravel.
- Codi documentat amb comentaris i realització de tests automatitzats.
- Ús d'eines de depuració: Xdebug, Sentry, Laravel Telescope. Investigar i justificar el ús d’aquestes eines.
- Chatbot RAG: Implementar una secció d'ajuda amb un Chatbot basat en documentació pròpia, usant l'API de la IA del servidor de l'institut. Orientat a diferents rols (Superadmin, Tenant Admin, Worker i Usuari final)

## Entorn Client (Front-end)

- Implementar State Management usant una llibreria de gestió d'estat amb Vue (investigar Pinia, Vuex, etc.).
- Millores d'usabilitat: Basades en la filosofia Don't make me think.
- Tests d'usabilitat: Enregistrar proves amb usuaris externs i passar el test de rendiment Lighthouse.

## Desplegament

- PaaS: Desplegar la versió alfa en serveis amb free tier (usant el GitHub Student Developer Pack).
- Desplegar el front-end, el back-end i el microservei IoT per separat.
- Automatització: Cal automatitzar el desplegament multitenant.

## Documentació i Altres Tasques

- Contingut del repositori: Ha d'incloure el Contributor Covenant, decisions preses, llicència EUPL, convencions de codificació i manual de desplegament.
