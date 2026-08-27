# Plan de déploiement — ASV

*Application de Suivi Vétérinaire · Mélissa Bedhomme · CDA RNCP37873 · Septembre 2026*

---

## 1. Vue d'ensemble

Le projet ASV est composé de **trois dépôts Git** déployés ensemble :

| Repo | Rôle | Technologie |
|---|---|---|
| **ASV** (repo principal) | Infra Docker, Nginx, Monitoring | Docker Compose, Nginx 1.27, Prometheus, Grafana |
| **ASV-backend** | API REST | Symfony 7.4 / PHP 8.2-FPM |
| **ASV-frontend** | Application web/mobile | Expo 54 / React Native Web |

Le backend et le frontend sont des sous-répertoires (`backend/` et `mobile-web/`) du repo principal, chacun avec son propre `.git`.

### Services Docker

| Service | Port hôte | Rôle |
|---|---|---|
| nginx | 8080 | Reverse proxy → PHP-FPM |
| php (PHP-FPM) | — | Exécution Symfony |
| mysql | — | Base de données |
| phpmyadmin | 8081 | Interface BDD (dev) |
| grafana | 3000 | Dashboard monitoring |
| prometheus | 9090 | Collecte métriques |
| node-exporter | 9100 | Métriques système hôte |
| mysqld-exporter | 9104 | Métriques MySQL |
| cadvisor | 8082 | Métriques containers Docker |
| mailer (Mailpit) | 8025 (UI) / 1025 (SMTP) | Capture mails (dev) |

---

## 2. Prérequis

### Machine cible

- Docker Engine ≥ 26 et Docker Compose v2 (`docker compose` sans tiret)
- Git
- 2 Go de RAM disponibles minimum

### Pour le build mobile natif (optionnel)

- Node.js 20+ et npm
- Expo CLI : `npm install -g expo-cli`
- Android : Android Studio + émulateur ou appareil physique
- iOS : Xcode sur macOS uniquement

---

## 3. Clonage des dépôts

Les trois dépôts sont **indépendants** (pas de submodule Git — `backend/` et `mobile-web/` sont même listés dans le `.gitignore` du repo principal). Le clone du repo principal seul ne suffit donc pas : les trois clones ci-dessous sont obligatoires.

```bash
# Cloner le repo principal (infra Docker)
git clone https://github.com/M-BEDH/ASV.git
cd ASV

# Cloner le backend et le frontend dans leurs sous-dossiers respectifs (obligatoire)
git clone https://github.com/M-BEDH/ASV-backend.git backend
git clone https://github.com/M-BEDH/ASV-frontend.git mobile-web
```

---

## 4. Configuration des variables d'environnement

### 4.1 Repo principal — infra Docker

```bash
cp .env.example .env
```

Renseigner toutes les valeurs dans `.env` :

| Variable | Description | Exemple |
|---|---|---|
| `MYSQL_ROOT_PASSWORD` | Mot de passe root MySQL | `MonRootSecret!` |
| `MYSQL_DATABASE` | Nom de la base | `asv_db` |
| `MYSQL_USER` | Utilisateur applicatif | `asv_user` |
| `MYSQL_PASSWORD` | Mot de passe utilisateur | `MonUserSecret!` |
| `GRAFANA_ADMIN_USER` | Login Grafana | `admin` |
| `GRAFANA_ADMIN_PASSWORD` | Mot de passe Grafana | `MonGrafanaSecret!` |

### 4.2 Backend Symfony

Créer `backend/.env.local` (non commité, surcharge `.env`) :

```bash
cp backend/.env backend/.env.local
```

Renseigner dans `backend/.env.local` :

| Variable | Valeur pour Docker | Notes |
|---|---|---|
| `APP_ENV` | `prod` | `dev` en développement |
| `APP_SECRET` | Chaîne aléatoire 32 caractères | `openssl rand -hex 16` |
| `DATABASE_URL` | `mysql://asv_user:MonUserSecret!@mysql:3306/asv_db?serverVersion=8.0&charset=utf8mb4` | L'hôte est `mysql` (nom du service Docker) |
| `JWT_SECRET_KEY` | `%kernel.project_dir%/config/jwt/private.pem` | Chemin vers la clé privée |
| `JWT_PUBLIC_KEY` | `%kernel.project_dir%/config/jwt/public.pem` | Chemin vers la clé publique |
| `JWT_PASSPHRASE` | Passphrase choisie | Générée à l'étape 5 |
| `MAILER_DSN` | `smtp://mailer:1025` | `null://null` si mail non utilisé |
| `CORS_ALLOW_ORIGIN` | `'^https?://(localhost\|127\.0\.0\.1)(:[0-9]+)?$'` | Adapter au domaine en production |

### 4.3 Frontend Expo

Créer `mobile-web/.env` (⚠️ pas `.env.local`) :

```bash
# mobile-web/.env
EXPO_PUBLIC_API_URL=http://localhost:8080
```

En production, remplacer `localhost:8080` par l'URL publique de l'API.

> **Important :** utiliser `.env` et non `.env.local`. C'est le fichier `.env` qui doit être présent au moment du build EAS pour que la variable soit embarquée dans l'APK.

---

## 5. Fichiers secrets non commités

### 5.1 Clés JWT (backend)

Les clés RSA doivent être générées une seule fois et placées dans `backend/config/jwt/` :

```bash
# Depuis le container PHP (après docker compose up)
docker compose exec php php bin/console lexik:jwt:generate-keypair
```

Cela crée :
- `backend/config/jwt/private.pem`
- `backend/config/jwt/public.pem`

Ces fichiers sont ignorés par `.gitignore` et doivent être conservés en lieu sûr. Les régénérer invalide tous les tokens JWT existants.

### 5.2 Credentials mysqld-exporter

Créer le fichier `monitoring/mysqld-exporter/.my.cnf` (ignoré par `.gitignore`) :

```ini
[client]
user=asv_user
password=MonUserSecret!
host=mysql
port=3306
```

L'utilisateur MySQL doit avoir les droits suivants (à accorder après initialisation de la BDD) :

```sql
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'asv_user'@'%';
FLUSH PRIVILEGES;
```

---

## 6. Démarrage de l'infrastructure

### 6.1 Premier démarrage

```bash
# Depuis la racine du projet ASV/
docker compose up -d
```

Docker va :
1. Construire l'image PHP depuis `backend/Dockerfile` (PHP 8.2-FPM + APCu + extensions)
2. Démarrer MySQL, créer la base et l'utilisateur
3. Démarrer Nginx, PHP-FPM, Prometheus, Grafana, Mailpit, cAdvisor, node-exporter, mysqld-exporter

Vérifier que tous les services sont up :

```bash
docker compose ps
```

### 6.2 Ordre de démarrage interne

Docker Compose gère les dépendances via `depends_on`. L'ordre effectif est :

```
mysql → php → nginx
mysql → mysqld-exporter
mysql → phpmyadmin
prometheus → grafana
```

---

## 7. Initialisation de la base de données

### Option A — Depuis le dump SQL (environnement vierge)

Le fichier `asv_db.sql` à la racine contient le schéma complet exporté depuis phpMyAdmin :

```bash
docker compose exec -T mysql mysql -u asv_user -pMonUserSecret! asv_db < asv_db.sql
```

Puis appliquer les migrations Doctrine (12 migrations de mars-avril 2026) :

```bash
docker compose exec php php bin/console doctrine:migrations:migrate --no-interaction
```

### Option B — Depuis les entités Doctrine (base vide)

Si la base est vide et qu'on préfère recréer le schéma depuis les entités :

```bash
docker compose exec php php bin/console doctrine:schema:create --no-interaction
```

> **Note :** La première migration (`Version20260310141259`) suppose que la table `users` existe déjà. En environnement vierge, utiliser l'Option A ou l'Option B (schema:create), pas `migrations:migrate` seul.

### Données de développement (optionnel)

```bash
docker compose exec php php bin/console doctrine:fixtures:load --no-interaction
```

---

## 8. Build du frontend web

Pour générer le build web statique :

```bash
cd mobile-web
npm install
npx expo export --platform web
# Résultat dans mobile-web/dist/
```

Le dossier `dist/` peut être servi par n'importe quel serveur web statique (Nginx, Apache, CDN).

---

## 9. Vérifications post-déploiement

### 9.1 Santé des services

| URL | Service | Vérification attendue |
|---|---|---|
| `http://localhost:8080/api/auth/login` | API Symfony | Réponse JSON (401 sans credentials) |
| `http://localhost:8080/metrics` | Métriques Prometheus Symfony | Page texte avec métriques `asv_*` |
| `http://localhost:8081` | phpMyAdmin | Interface de connexion |
| `http://localhost:3000` | Grafana | Dashboard ASV App visible |
| `http://localhost:9090` | Prometheus | Interface Prometheus, targets UP |
| `http://localhost:8025` | Mailpit | Interface de capture mails |
| `http://localhost:8082` | cAdvisor | Interface métriques containers |

### 9.2 Prometheus — vérifier que tous les targets sont UP

Dans l'interface Prometheus (`http://localhost:9090/targets`), les jobs suivants doivent être en état `UP` :
- `prometheus`
- `node-exporter`
- `cadvisor`
- `mysqld-exporter`
- `symfony`

### 9.3 Tests PHPUnit (backend)

```bash
docker compose exec php php bin/phpunit
```

### 9.4 Droits MySQL pour mysqld-exporter

```bash
docker compose exec mysql mysql -u root -p${MYSQL_ROOT_PASSWORD} -e \
  "GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'asv_user'@'%'; FLUSH PRIVILEGES;"
```

---

## 10. Monitoring — Grafana

Le dashboard **ASV App** est provisionné automatiquement au démarrage depuis :
- `monitoring/grafana/provisioning/datasources/datasource.yml` — datasource Prometheus
- `monitoring/grafana/provisioning/dashboards/dashboard.yml` — config chargement
- `monitoring/grafana/provisioning/dashboards/asv-dashboard.json` — dashboard JSON

Le dashboard expose 5 sections : santé containers, trafic HTTP, métriques métier (connexions/inscriptions par rôle), métriques MySQL, CPU/RAM par container.

Connexion Grafana : identifiants définis dans `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD`.

---

## 11. CI/CD

| Repo | Pipeline | Déclencheur |
|---|---|---|
| ASV (infra) | Validation `docker compose config` | push/PR sur `master` |
| ASV-backend | Tests PHPUnit avec MySQL réel | push/PR sur `master` |
| ASV-frontend | TypeScript check + build Expo web | push/PR sur `master` |

Les secrets CI sont configurés dans **GitHub → Settings → Secrets** de chaque repo.

---

## 12. Rollback

### Précaution — sauvegarder la BDD avant tout rollback

```bash
docker compose exec mysql mysqldump -u asv_user -pMonUserSecret! asv_db > backup_avant_rollback.sql
```

### Rollback BDD (annuler la dernière migration)

```bash
docker compose exec php php bin/console doctrine:migrations:migrate prev --no-interaction
```

### Rollback infra Docker

```bash
# Arrêter les services sans supprimer les volumes (données MySQL conservées)
docker compose down

# Revenir à la version précédente du code
git checkout <commit-précédent>

# Relancer en rebuildant l'image si le Dockerfile ou les dépendances ont changé
docker compose up -d --build
```

### Suppression complète (reset total)

```bash
# Arrête les services ET supprime les volumes (perte des données MySQL)
docker compose down -v
```

---

## 13. Récapitulatif des fichiers à créer manuellement

Ces fichiers sont ignorés par `.gitignore` et doivent être créés sur chaque environnement :

| Fichier | Étape | Contenu |
|---|---|---|
| `.env` | §4.1 | Variables infra Docker |
| `backend/.env.local` | §4.2 | Variables Symfony |
| `mobile-web/.env` | §4.3 | URL API frontend |
| `monitoring/mysqld-exporter/.my.cnf` | §5.2 | Credentials MySQL exporter |
| `backend/config/jwt/private.pem` | §5.1 | Clé privée JWT (générée) |
| `backend/config/jwt/public.pem` | §5.1 | Clé publique JWT (générée) |

---

*Mélissa Bedhomme — Dossier professionnel CDA RNCP37873 — Septembre 2026*
