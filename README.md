# SamaFile - backend-ci

API métier de SamaFile, construite avec FastAPI et PostgreSQL. Le dépôt contient l’API, ses migrations Alembic, les tests et le pipeline CI.

| Élément | Valeur |
| --- | --- |
| Runtime | Python 3.12, FastAPI, SQLAlchemy |
| Base de données | PostgreSQL 16, migrations Alembic |
| Port API | `8000` |
| Santé | `GET /health` |
| OpenAPI | `http://localhost:8000/docs` |
| Dépôt | [MNG43/backend-ci](https://github.com/MNG43/backend-ci) |
| Image Docker Hub | [mng2021/samafile-backend](https://hub.docker.com/r/mng2021/samafile-backend) |

## Démarrage local

Le backend a besoin d’une base PostgreSQL accessible. Pour éviter de configurer séparément la base et le proxy, le dépôt `deployment-cd` fournit la stack locale avec les deux images construites depuis les sources voisines. Consultez son README pour lancer la stack complète.

Pour lancer uniquement l’API contre une base PostgreSQL déjà disponible :

```powershell
cd backend-ci
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Configurez `DATABASE_URL`, `SECRET_KEY` et les autres valeurs nécessaires dans `.env`, puis chargez-les dans votre session PowerShell et démarrez l’API :

```powershell
$env:DATABASE_URL = "postgresql+psycopg://samafile:mot_de_passe@localhost:5432/samafile"
$env:SECRET_KEY = "remplacer-par-une-cle-aleatoire-de-32-caracteres-minimum"
$env:APP_ENV = "development"
uvicorn src.main:app --reload
```

L’API écoute sur `http://localhost:8000`. Vérifiez `/health` et ouvrez `/docs`. Au démarrage, le conteneur applique `alembic upgrade head` avant de lancer Uvicorn. Ne commitez jamais de fichier `.env` ni de vrai secret.

## Tests

Les tests utilisent une base PostgreSQL de test dédiée dont le nom se termine par `_test`. N’utilisez jamais une base de production :

```powershell
python -m pip install -r tests/requirements.txt
$env:APP_ENV = "test"
$env:TEST_DATABASE_URL = "postgresql+psycopg://samafile:mot_de_passe@localhost:5432/samafile_test"
python -m pytest tests
```

## CI et publication Docker

Le `Jenkinsfile` exécute les tests Pytest, Bandit, pip-audit, Hadolint et Trivy, construit l’image et archive les rapports. La publication est contrôlée par le paramètre Jenkins `PUBLISH` (désactivé par défaut). Lorsqu’elle est activée, Jenkins pousse le tag du build et le SHA du commit ; `latest` est aussi poussé pour les branches principales configurées.

L’image publiée est `mng2021/samafile-backend`. Consultez Docker Hub pour les tags disponibles avant de choisir une version dans le pipeline CD.

### Exécuter le pipeline CI dans Jenkins

Configurez le job `samafile-backend-ci` en **Pipeline script from SCM**, avec ce dépôt, la
branche `mng` et le chemin de script `Jenkinsfile`. Pour vérifier une modification sans publier
d'image, lancez **Build with Parameters** avec `PUBLISH` décoché. Cochez-le seulement après un
build réussi pour publier sur Docker Hub.

Le pipeline utilise une base PostgreSQL temporaire pour Pytest, puis exécute Bandit, `pip-audit`,
Hadolint, la construction Docker et Trivy. Les rapports archivés dans Jenkins comprennent
`reports/pytest.xml`, `reports/bandit.json`, `reports/pip-audit.json`,
`reports/trivy-image.txt` et `reports/trivy-fs.txt`.

La publication crée un tag avec le numéro du build et un tag avec le SHA court du commit.
`latest` n'est publié que depuis `main`, `master` ou `develop`; ne supposez donc pas qu'il existe
pour la branche `mng`. Avant un déploiement CD, vérifiez les tags backend réellement disponibles
sur Docker Hub.

Pour déclencher automatiquement le job lors d'un changement, configurez **Configurer > Triggers >
Scrutation de l'outil de gestion de version** avec `H/2 * * * *`. Le polling surveille la branche
SCM configurée; poussez un commit sur `mng` et vérifiez qu'un nouveau build démarre. Il ne publie
pas d'image tant que `PUBLISH` reste décoché.

## Structure

```
backend-ci/
|-- Dockerfile             image de production
|-- Dockerfile.ci          image avec outils de tests et d’analyse
|-- Jenkinsfile            pipeline CI
|-- requirements.txt       dépendances runtime
|-- requirements-ci.txt    outils CI et scans
|-- alembic.ini            configuration des migrations
|-- migrations/            versions Alembic
|-- src/                   API FastAPI et logique métier
|-- tests/                 tests Pytest
```

## Variables d’environnement

Les valeurs d’exemple et les noms des variables sont dans [`.env.example`](.env.example). Les principales variables sont `DATABASE_URL`, `SECRET_KEY`, `CORS_ORIGINS`, `ADMIN_EMAIL`, `ADMIN_PASSWORD` et `APP_ENV`. En production, utilisez une clé JWT forte et des mots de passe uniques. `SEED_DEMO_ACCOUNTS` et `DEMO_PASSWORD` servent uniquement à initialiser des comptes de démonstration.

---

# API pour le frontend (contrat avec M2)

## RÃ¨gles gÃ©nÃ©rales

| Sujet            | Valeur                                                                                                                                    |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| URL              | Une seule adresse pour toute l'API : `http://localhost:8000` (tickets, services et guichets). Tous les chemins commencent par `/api`. |
| Format           | JSON, noms de champs en `snake_case`, tous les chemins commencent par `/api`                                                          |
| Dates            | ISO 8601 en UTC avec `Z`, ex. `2026-11-09T10:00:00Z`. Le SÃ©nÃ©gal est Ã  UTC+0 : l'heure locale est identique.                       |
| CORS             | Origines autorisÃ©es configurÃ©es par `CORS_ORIGINS`                                                                                    |
| Identifiants     | Entiers (`id: 25`)                                                                                                                      |
| Authentification | JWT : en-tÃªte `Authorization: Bearer <jeton>` (voir Â« Comptes, rÃ´les et sÃ©curitÃ© Â»)                                               |
| Actualisation    | Le frontend rafraÃ®chit par polling (toutes les 5 secondes) : pas de WebSocket                                                            |
| Messages         | En franÃ§ais, prÃªts Ã  afficher (champ `message`)                                                                                      |

## Statuts d'un ticket

| Statut          | Signification                              | Passe Ã                                                  |
| --------------- | ------------------------------------------ | -------------------------------------------------------- |
| `WAITING`     | En attente dans la file                    | `CALLED`, `CANCELLED`, `EXPIRED`                   |
| `CALLED`      | AppelÃ©, doit se prÃ©senter au guichet     | `IN_PROGRESS`, `NO_SHOW`, `CANCELLED`, `EXPIRED` |
| `IN_PROGRESS` | En cours de traitement                     | `COMPLETED`                                            |
| `COMPLETED`   | TerminÃ© (final)                           | -                                                        |
| `NO_SHOW`     | Absent                                     | `WAITING` (rÃ©activation)                              |
| `CANCELLED`   | AnnulÃ© (final)                            | -                                                        |
| `EXPIRED`     | ExpirÃ© Ã  la fermeture du service (final) | -                                                        |

Toute autre transition renvoie une erreur `409 INVALID_TRANSITION`.

## Parcours Ã  implÃ©menter

**Usager :** `GET /api/services` (choisir un service ouvert) puis `POST /api/tickets`
puis `GET /api/tickets/{id}` toutes les 5 s (statut, position, message, et banniÃ¨re `notification` si elle n'est pas `null`).

**Agent :** `GET /api/counters` (choisir son guichet) puis `GET /api/counters/{id}/tickets` (la file)
puis `POST /api/counters/{id}/next` (appeler) puis `PATCH /api/tickets/{id}/start` puis `PATCH /api/tickets/{id}/complete`.
Si l'usager ne vient pas : `PATCH /api/tickets/{id}/no-show` (autorisÃ© seulement quand le dÃ©lai est Ã©coulÃ©).
Le dÃ©lai restant se calcule avec `called_at` + `no_show_timeout_seconds` du service.

## DonnÃ©es de dÃ©monstration (crÃ©Ã©es au premier dÃ©marrage)

| Service (`id`)        | Lettre des tickets | Guichets (`id`) | Horaires      |
| ----------------------- | ------------------ | ----------------- | ------------- |
| DÃ©pÃ´t (1)             | A                  | 1 et 2            | 08:00 - 16:00 |
| Retrait (2)             | B                  | 3 et 4            | 08:00 - 16:00 |
| Ouverture de compte (3) | C                  | 5 et 6            | 08:00 - 16:00 |
| Renseignements (4)      | D                  | 7 et 8            | 08:00 - 16:00 |

Le numÃ©ro d'un ticket est `lettre du service` + numÃ©ro du jour sur 3 chiffres (`A001`, `A002`...), et repart Ã  `001` chaque jour.

## Routes : services et tickets

| MÃ©thode | Route                            | But                                             | SuccÃ¨s |
| -------- | -------------------------------- | ----------------------------------------------- | ------- |
| GET      | `/health`                      | SantÃ© du backend et de la base                 | 200     |
| GET      | `/api/services`                | Liste des services (avec ouverture)             | 200     |
| GET      | `/api/services/{id}`           | Un service                                      | 200     |
| POST     | `/api/tickets`                 | CrÃ©er un ticket                                | 201     |
| GET      | `/api/tickets`                 | Lister les tickets (filtres)                    | 200     |
| GET      | `/api/tickets/{id}`            | DÃ©tail d'un ticket                             | 200     |
| GET      | `/api/tickets/{id}/position`   | Position dans la file                           | 200     |
| PATCH    | `/api/tickets/{id}/start`      | `CALLED` vers `IN_PROGRESS`                 | 200     |
| PATCH    | `/api/tickets/{id}/complete`   | `IN_PROGRESS` vers `COMPLETED`              | 200     |
| PATCH    | `/api/tickets/{id}/no-show`    | `CALLED` vers `NO_SHOW`, appelle le suivant | 200     |
| PATCH    | `/api/tickets/{id}/cancel`     | `WAITING` ou `CALLED` vers `CANCELLED`    | 200     |
| POST     | `/api/tickets/{id}/reactivate` | `NO_SHOW` vers `WAITING`, en fin de file    | 200     |

### GET /api/services/

```json
{
  "id": 1,
  "name": "DÃ©pÃ´t",
  "opening_time": "08:00",
  "closing_time": "16:00",
  "ticket_cutoff_time": "15:30",
  "ticket_cutoff_minutes_before_closing": 30,
  "is_open": true,
  "ticket_distribution_open": true,
  "no_show_timeout_seconds": 120
}
```

`GET /api/services` renvoie la liste de ces objets. DÃ©sactiver le bouton Â« Prendre un ticket Â» quand `ticket_distribution_open` vaut `false`.

### POST /api/tickets

Corps : `service_id` obligatoire, `user_name` facultatif (100 caractÃ¨res maximum, ticket anonyme si absent).

```json
{ "service_id": 1, "user_name": "Awa Ndiaye" }
```

RÃ©ponse `201` :

```json
{
  "id": 1,
  "ticket_number": "A001",
  "status": "WAITING",
  "position": 1,
  "message": "Votre ticket a Ã©tÃ© crÃ©Ã©."
}
```

Erreurs : `404 SERVICE_NOT_FOUND`, `409 SERVICE_CLOSED` (hors horaires, le `message` indique l'heure d'ouverture), `422 VALIDATION_ERROR`.

### GET /api/tickets/

```json
{
  "id": 1,
  "ticket_number": "A001",
  "service_id": 1,
  "counter_id": null,
  "status": "WAITING",
  "created_at": "2026-11-09T10:00:00Z",
  "called_at": null,
  "user_name": "Awa Ndiaye",
  "position": 1,
  "people_ahead": 0,
  "notification": "Vous Ãªtes le prochain : prÃ©sentez-vous prÃ¨s du guichet.",
  "message": "Votre ticket est en attente."
}
```

`position`, `people_ahead` et `notification` ne sont renseignÃ©s que si le statut est `WAITING` (sinon `null`).
`counter_id` et `called_at` sont renseignÃ©s Ã  partir de l'appel. Voir la section Â« Notification Â» ci-dessous.
Erreur : `404 TICKET_NOT_FOUND`.

### GET /api/tickets/{id}/position

```json
{
  "id": 2,
  "ticket_number": "A002",
  "status": "WAITING",
  "position": 2,
  "people_ahead": 1,
  "notification": "Plus qu'une personne devant vous. Rapprochez-vous du service."
}
```

### GET /api/tickets

Filtres facultatifs : `service_id`, `counter_id`, `status` (un des 7 statuts), `limit` (1 Ã  200, dÃ©faut 50), `offset` (dÃ©faut 0).
Tri : ordre de la file (FIFO).

```json
{
  "items": [
    {
      "id": 1,
      "ticket_number": "A001",
      "service_id": 1,
      "counter_id": null,
      "status": "WAITING",
      "created_at": "2026-11-09T10:00:00Z",
      "called_at": null
    }
  ],
  "total": 1
}
```

Un `status` inconnu renvoie `422 VALIDATION_ERROR`.

### Notification Â« rapprochez-vous Â»

Quand il reste **10 personnes ou moins** devant un ticket `WAITING`, le backend ajoute un message dans le champ
`notification` de `GET /api/tickets/{id}` et `GET /api/tickets/{id}/position`. Sinon, `notification` vaut `null`.

| `people_ahead` | `notification`                                                     |
| ---------------- | -------------------------------------------------------------------- |
| plus de 10       | `null`                                                             |
| 10               | `"Plus que 10 personnes devant vous. Rapprochez-vous du service."` |
| 1                | `"Plus qu'une personne devant vous. Rapprochez-vous du service."`  |
| 0                | `"Vous Ãªtes le prochain : prÃ©sentez-vous prÃ¨s du guichet."`     |

- `people_ahead` = `position` - 1 : nombre de tickets `WAITING` du mÃªme service passÃ©s avant le sien.
- **CÃ´tÃ© frontend :** au polling (toutes les 5 s), afficher une banniÃ¨re bien visible dÃ¨s que `notification` n'est pas `null`,
  avec le texte tel quel. Le message se met Ã  jour tout seul quand la file avance.
- La notification s'affiche seulement quand la page de suivi est ouverte : pas de SMS ni de notification hors application dans le MVP.
- Le seuil (10) est la constante `APPROACH_THRESHOLD` dans `src/logic.py`.

### start, complete, cancel, reactivate

Aucun corps. RÃ©ponse `200` : le ticket mis Ã  jour, au mÃªme format que `GET /api/tickets/{id}`.
Erreurs : `404 TICKET_NOT_FOUND`, `409 INVALID_TRANSITION`, et pour `reactivate` aussi `409 SERVICE_CLOSED`.
AprÃ¨s `reactivate`, le ticket est `WAITING` et placÃ© **en fin de file**.

### PATCH /api/tickets/{id}/no-show

Aucun corps. AutorisÃ© seulement si le dÃ©lai du service (120 s par dÃ©faut) est Ã©coulÃ© depuis `called_at`.
Le backend appelle ensuite automatiquement le ticket suivant : il est renvoyÃ© dans `next_ticket`
(`null` si la file est vide). RÃ©ponse `200` :

```json
{
  "id": 1,
  "ticket_number": "A001",
  "service_id": 1,
  "counter_id": 1,
  "status": "NO_SHOW",
  "created_at": "2026-11-09T10:00:00Z",
  "called_at": "2026-11-09T10:00:00Z",
  "user_name": "Awa Ndiaye",
  "position": null,
  "people_ahead": null,
  "notification": null,
  "message": "Vous avez Ã©tÃ© dÃ©clarÃ© absent.",
  "next_ticket": {
    "id": 2,
    "ticket_number": "A002",
    "service_id": 1,
    "counter_id": 1,
    "status": "CALLED",
    "created_at": "2026-11-09T10:00:00Z",
    "called_at": "2026-11-09T10:02:01Z",
    "user_name": null,
    "position": null,
    "people_ahead": null,
    "notification": null,
    "message": "Votre ticket est appelÃ©. PrÃ©sentez-vous au guichet."
  }
}
```

Erreurs : `409 NO_SHOW_TOO_EARLY` (avec `details.remaining_seconds`), `409 INVALID_TRANSITION`, `404 TICKET_NOT_FOUND`.

## Routes : guichets

| MÃ©thode | Route                          | But                                        | SuccÃ¨s |
| -------- | ------------------------------ | ------------------------------------------ | ------- |
| GET      | `/api/counters`              | Liste des guichets (filtre `service_id`) | 200     |
| GET      | `/api/counters/{id}/tickets` | File du guichet                            | 200     |
| POST     | `/api/counters/{id}/next`    | Appeler le prochain ticket                 | 200     |

### GET /api/counters

```json
[
  { "id": 1, "name": "DÃ©pÃ´t - guichet 1", "service_id": 1, "service_name": "DÃ©pÃ´t", "is_active": true },
  { "id": 2, "name": "DÃ©pÃ´t - guichet 2", "service_id": 1, "service_name": "DÃ©pÃ´t", "is_active": true }
]
```

### GET /api/counters/{id}/tickets

Tickets du service du guichet, dans l'ordre FIFO. ParamÃ¨tres facultatifs : `status` (dÃ©faut `WAITING`), `limit`, `offset`.
RÃ©ponse au mÃªme format que `GET /api/tickets` (`items` et `total`). Erreur : `404 COUNTER_NOT_FOUND`.

### POST /api/counters/{id}/next

Aucun corps. Le plus ancien ticket `WAITING` du service passe Ã  `CALLED` et est rattachÃ© au guichet.
RÃ©ponse `200` : le ticket appelÃ©, au mÃªme format que `GET /api/tickets/{id}` (`status: "CALLED"`, `counter_id`, `called_at` renseignÃ©s).

Erreurs : `404 COUNTER_NOT_FOUND`, `409 QUEUE_EMPTY` (aucun ticket en attente),
`409 COUNTER_BUSY` (le guichet traite dÃ©jÃ  un ticket : le terminer d'abord), `409 COUNTER_INACTIVE`.

## Format des erreurs

Toutes les erreurs ont la mÃªme forme :

```json
{ "error": { "code": "SERVICE_CLOSED", "message": "Le service est fermÃ©. Aucun ticket ne peut Ãªtre attribuÃ©.", "details": null } }
```

Le frontend teste `error.code` et affiche `error.message`. `details` vaut `null` sauf pour
`VALIDATION_ERROR` (liste `{field, message}`) et `NO_SHOW_TOO_EARLY` (`remaining_seconds`).

| Code                     | HTTP | Cas                                      |
| ------------------------ | ---- | ---------------------------------------- |
| `VALIDATION_ERROR`     | 422  | Corps ou paramÃ¨tre invalide             |
| `SERVICE_NOT_FOUND`    | 404  | Service inexistant ou inactif            |
| `TICKET_NOT_FOUND`     | 404  | Ticket inexistant                        |
| `COUNTER_NOT_FOUND`    | 404  | Guichet inexistant                       |
| `NOT_FOUND`            | 404  | Route inexistante                        |
| `SERVICE_CLOSED`       | 409  | CrÃ©ation ou rÃ©activation hors horaires |
| `INVALID_TRANSITION`   | 409  | Changement de statut interdit            |
| `NO_SHOW_TOO_EARLY`    | 409  | DÃ©lai d'absence pas encore Ã©coulÃ©     |
| `QUEUE_EMPTY`          | 409  | Aucun ticket Ã  appeler                  |
| `COUNTER_BUSY`         | 409  | Guichet dÃ©jÃ  occupÃ©                   |
| `COUNTER_INACTIVE`     | 409  | Guichet inactif                          |
| `DATABASE_UNAVAILABLE` | 503  | Base injoignable (`/health`)           |
| `INTERNAL_ERROR`       | 500  | Erreur inattendue                        |

## RÃ¨gles appliquÃ©es par le backend

- FIFO : le plus ancien ticket `WAITING` du service est appelÃ© en premier.
- Position = 1 + nombre de tickets `WAITING` du mÃªme service passÃ©s avant lui.
- Notification Â« rapprochez-vous Â» dÃ¨s qu'il reste 10 personnes ou moins devant un ticket en attente.
- Un ticket n'est crÃ©Ã© qu'entre l'ouverture (incluse) et la limite de prise (exclue), heure `Africa/Dakar`. Par dÃ©faut, la limite est 30 minutes avant la fermeture ; l'avance est configurable par service (`ticket_cutoff_minutes_before_closing`). Si elle couvre toute la durÃ©e d'ouverture, aucun ticket n'est dÃ©livrÃ©.
- Ã€ la fermeture (ou le lendemain), les tickets `WAITING`, `CALLED` et `IN_PROGRESS` passent Ã  `EXPIRED`.
- Absence : refusÃ©e tant que le dÃ©lai du service n'est pas Ã©coulÃ© ; le ticket suivant est appelÃ© automatiquement.
- RÃ©activation : seulement depuis `NO_SHOW`, service ouvert, ticket replacÃ© en fin de file.

---

# Comptes, rÃ´les et sÃ©curitÃ© (version 2)

## Les trois rÃ´les

| RÃ´le           | Qui                                       | Ce qu'il fait                                                                                                                                                                                                                                                                          |
| --------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **USER**  | Usager                                    | S'inscrit, se connecte, prend un ticket, suit sa position, annule son ticket, consulte son historique, modifie son profil et son mot de passe.                                                                                                                                         |
| **AGENT** | Agent de guichet (assignÃ© Ã  un guichet) | CrÃ©e des comptes usagers et des tickets au comptoir, appelle le suivant **depuis son guichet**, dÃ©marre / termine / dÃ©clare absent, rÃ©active ou annule les tickets de **son service**, rÃ©initialise un mot de passe usager, consulte les statistiques de son service. |
| **ADMIN** | Administrateur                            | Tout ce que fait un agent, sur tous les services et guichets, plus : crÃ©er agents et administrateurs, affecter un agent Ã  un guichet, activer / dÃ©sactiver des comptes, crÃ©er et modifier services et guichets (horaires, dÃ©lai d'absence), statistiques globales.                |

## Tableau des accÃ¨s

| Route                                                                                                           | Public |      USER      |         AGENT         | ADMIN |
| --------------------------------------------------------------------------------------------------------------- | :----: | :------------: | :--------------------: | :---: |
| `GET /health`, `GET /api/services`, `GET /api/services/{id}`, `GET /api/counters`                       |  oui  |      oui      |          oui          |  oui  |
| `POST /api/auth/register`, `POST /api/auth/login`                                                           |  oui  |                |                        |      |
| `GET/PATCH /api/auth/me`, `POST /api/auth/change-password`                                                  |        |      oui      |          oui          |  oui  |
| `POST /api/tickets`                                                                                           |        | pour lui-mÃªme |      son service      | tous |
| `GET /api/me/tickets`                                                                                         |        |      oui      |          oui          |  oui  |
| `GET /api/tickets/{id}`, `GET /api/tickets/{id}/position`, `PATCH /api/tickets/{id}/cancel`               |        |  ses tickets  |      son service      | tous |
| `GET /api/tickets`, `GET /api/counters/{id}/tickets`                                                        |        |                |      son service      | tous |
| `POST /api/counters/{id}/next`                                                                                |        |                |      son guichet      | tous |
| `PATCH /api/tickets/{id}/start`, `/complete`, `/no-show`                                                  |        |                | tickets de son guichet | tous |
| `POST /api/tickets/{id}/reactivate`                                                                           |        |                |      son service      | tous |
| `POST /api/users`, `GET /api/users`, `GET/PATCH /api/users/{id}`, `POST /api/users/{id}/reset-password` |        |                |      comptes USER      | tous |
| `POST/PATCH /api/services`, `POST/PATCH /api/counters`                                                      |        |                |                        |  oui  |
| `GET /api/stats`                                                                                              |        |                |      son service      | tous |

## Authentification

### POST /api/auth/register (201)

Inscription libre d'un usager (rÃ´le USER, jamais autre chose).

```json
{"email": "awa@exemple.sn", "password": "8 caractÃ¨res minimum", "full_name": "Awa Ndiaye", "phone": "771234567"}
```

### POST /api/auth/login (200)

```json
{"email": "awa@exemple.sn", "password": "..."}
```

RÃ©ponse de `register` et `login` :

```json
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 28800,
  "user": {"id": 3, "email": "awa@exemple.sn", "full_name": "Awa Ndiaye", "phone": "771234567",
           "role": "USER", "counter_id": null, "is_active": true}
}
```

Le frontend enregistre `access_token` et l'envoie dans `Authorization: Bearer <access_token>`. Le jeton dure 8 heures. Le champ `user.role` indique quelle interface afficher (usager, guichet agent, administration).

### GET /api/auth/me, PATCH /api/auth/me

`PATCH` accepte `{"full_name": "...", "phone": "..."}`.

### POST /api/auth/change-password

`{"current_password": "...", "new_password": "..."}` â†’ 200, ou 400 `INVALID_PASSWORD`.

## Usager

### POST /api/tickets

`{"service_id": 1}` : `user_name` est facultatif et vaut par dÃ©faut le nom du compte. Le ticket est rattachÃ© au compte.

### GET /api/me/tickets

Historique des tickets du compte, du plus rÃ©cent au plus ancien. Filtres : `status`, `limit`, `offset`. RÃ©ponse : `{"items": [...], "total": n}`.

Le suivi de position ne change pas : interroger `GET /api/tickets/{id}/position` toutes les 5 Ã  10 secondes (le serveur ne pousse pas de notification).

## Agent et administrateur

### POST /api/users (201) : crÃ©er un compte

```json
{"email": "client@exemple.sn", "password": "Motdepasse1", "full_name": "Client", "phone": "77...", "role": "USER"}
```

- Un **agent** ne crÃ©e que des comptes `USER`.
- Un **administrateur** crÃ©e aussi `AGENT` (avec `counter_id` pour l'affecter Ã  un guichet) et `ADMIN`.

### GET /api/users

Filtres : `role`, `q` (recherche nom ou email), `is_active`, `limit`, `offset`. Un agent ne voit que les usagers.

### PATCH /api/users/

Administrateur : `full_name`, `phone`, `role`, `counter_id`, `is_active`. Agent : `full_name`, `phone`, `is_active` d'un usager. Un compte dÃ©sactivÃ© ne peut plus se connecter et son jeton est refusÃ©. On ne peut pas se dÃ©sactiver ni changer son propre rÃ´le (409 `CANNOT_MODIFY_SELF`).

### POST /api/users/{id}/reset-password

`{"new_password": "..."}` : dÃ©finit un nouveau mot de passe (usager qui a oubliÃ© le sien).

### Tickets au comptoir

`POST /api/tickets` par un agent ou un administrateur accepte en plus :

- `user_id` : crÃ©e le ticket pour ce compte usager (l'usager le retrouve dans `GET /api/me/tickets`) ;
- ou seulement `user_name` : personne sans compte.

### Travail au guichet

Un agent appelle le suivant avec `POST /api/counters/{son_counter_id}/next`, puis `start`, `complete` ou `no-show` sur le ticket appelÃ© Ã  son guichet. Son `counter_id` est dans `GET /api/auth/me`. Sans guichet assignÃ© : 403 `NO_COUNTER_ASSIGNED`.

## Administrateur : services, guichets, statistiques

### POST /api/services (201)

```json
{"name": "Conseil", "opening_time": "08:00", "closing_time": "16:00", "ticket_cutoff_minutes_before_closing": 30, "no_show_timeout_seconds": 120}
```

`opening_time` doit prÃ©cÃ©der `closing_time` (sinon 422). RÃ©ponse : le mÃªme format que `GET /api/services/{id}`.

### PATCH /api/services/

Champs facultatifs : `name`, `opening_time`, `closing_time`, `ticket_cutoff_minutes_before_closing` (0 Ã  1440, 30 par dÃ©faut ; 0 autorise la prise jusqu'Ã  la fermeture), `ticket_cutoff_time` (`HH:MM` pour un override prÃ©cis, `null` pour revenir Ã  l'avance configurÃ©e), `no_show_timeout_seconds`, `is_active` (`false` retire le service de la liste publique). La rÃ©ponse expose l'heure effective dans `ticket_cutoff_time`.

### POST /api/counters (201), PATCH /api/counters/

`{"name": "Guichet 3", "service_id": 1}` ; `PATCH` : `name`, `is_active`.

### GET /api/stats

Tableau de bord du jour (filtre facultatif `service_id`) :

```json
{
  "date": "2026-11-10",
  "total_tickets": 12,
  "services": [
    {"service_id": 1, "service_name": "DÃ©pÃ´t", "total_today": 12, "waiting": 3, "called": 1, "in_progress": 1,
     "completed": 5, "no_show": 1, "cancelled": 1, "expired": 0, "average_wait_seconds": 240}
  ]
}
```

## Erreurs liÃ©es aux comptes

| Statut | Code                    | Cas                                                                                                      |
| ------ | ----------------------- | -------------------------------------------------------------------------------------------------------- |
| 401    | `UNAUTHORIZED`        | Jeton absent                                                                                             |
| 401    | `INVALID_TOKEN`       | Jeton invalide, expirÃ© ou compte dÃ©sactivÃ©                                                            |
| 401    | `INVALID_CREDENTIALS` | Email ou mot de passe incorrect                                                                          |
| 400    | `INVALID_PASSWORD`    | Mot de passe actuel incorrect                                                                            |
| 403    | `FORBIDDEN`           | RÃ´le insuffisant ou ressource hors pÃ©rimÃ¨tre (ticket d'un autre usager, autre service, autre guichet) |
| 403    | `NO_COUNTER_ASSIGNED` | Agent sans guichet                                                                                       |
| 404    | `USER_NOT_FOUND`      | Compte introuvable                                                                                       |
| 409    | `EMAIL_ALREADY_USED`  | Email dÃ©jÃ  inscrit                                                                                     |
| 409    | `CANNOT_MODIFY_SELF`  | Auto-dÃ©sactivation ou changement de son propre rÃ´le                                                    |

## Variables d'environnement

| Variable                                            | RÃ´le                                                                                                   |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `DATABASE_URL`                                    | Connexion PostgreSQL (obligatoire)                                                                      |
| `APP_ENV`                                         | `production` en dÃ©ploiement.                                                                         |
| `SECRET_KEY`                                      | ClÃ© JWT d'au moins 32 caractÃ¨res et non placeholder en production.                                    |
| `CORS_ORIGINS`                                    | Origines frontend autorisÃ©es, sÃ©parÃ©es par des virgules.                                             |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `ADMIN_NAME` | CrÃ©ent le premier compte administrateur au dÃ©marrage s'il n'existe pas.                               |
| `SEED_DEMO_ACCOUNTS=true`                         | CrÃ©e `agent1@samafile.sn` (guichet 1), `agent2@samafile.sn` (guichet 2) et `usager@samafile.sn`. |
| `DEMO_PASSWORD`                                   | Mot de passe des comptes de dÃ©monstration (dÃ©faut `Demo12345`).                                     |
| `ACCESS_TOKEN_MINUTES`                            | DurÃ©e de vie du jeton (480 par dÃ©faut).                                                               |

Exemple de lancement pour le dÃ©veloppement et la dÃ©monstration :

```bash
docker run -d --name backend_ci --network samafile -p 8000:8000 \
  -e SECRET_KEY="$(openssl rand -hex 32)" \
  -e ADMIN_EMAIL=admin@samafile.sn -e ADMIN_PASSWORD=UnMotDePasseSolide1 \
  -e SEED_DEMO_ACCOUNTS=true \
  samafile/backend
```

Le schÃ©ma est versionnÃ© par Alembic et migrÃ© avant le dÃ©marrage de l'API.

## Affichage du guichet appelant (Ã©cran de l'usager)

DÃ¨s qu'un guichet appelle un ticket, `GET /api/tickets/{id}` et `GET /api/tickets/{id}/position` renvoient `counter_id` et `counter_name` (par exemple `"DÃ©pÃ´t - guichet 1"`). Le champ `message` devient Â« Votre ticket est appelÃ©. PrÃ©sentez-vous au guichet : DÃ©pÃ´t - guichet 1. Â». Tant que le ticket est en attente, ces champs valent `null`. Deux guichets n'appellent jamais le mÃªme ticket (verrou `FOR UPDATE SKIP LOCKED` en base).
