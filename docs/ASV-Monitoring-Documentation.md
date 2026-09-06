# Documentation Monitoring
## ASV — Stack Prometheus / Grafana / Symfony

*Mélissa Bedhomme · CDA RNCP36490 · Mai 2026*

---

## 1. Contexte et objectif

Le projet ASV est une application mobile React Native / Expo connectée à une API Symfony (PHP 8.2) et une base de données MySQL 8.0, le tout orchestré via Docker Compose.

L'objectif de ce chantier était de mettre en place un système de monitoring complet permettant de :

- Visualiser l'état de santé des services en temps réel
- Mesurer les performances de l'API (trafic, latence, erreurs)
- Surveiller la base de données MySQL
- Suivre des métriques métier propres à l'application (connexions, inscriptions)
- Centraliser tout cela dans un dashboard Grafana persistant

---

## 2. Architecture globale

Le monitoring s'appuie sur le pattern standard Prometheus / Grafana. Chaque service expose ses métriques, Prometheus les collecte toutes les 15 secondes, et Grafana les visualise.

### Schéma de flux

| Services | → Prometheus | → Grafana |
|---|---|---|
| nginx, php, mysql<br>node-exporter, cadvisor<br>mysqld-exporter | Scrape toutes les 15s<br>Stocke les time-series<br>port 9090 | Dashboard ASV App<br>port 3000 |

---

## 3. Structure des fichiers

Voici où chaque fichier doit être placé dans le projet :

| Fichier | Emplacement | Rôle |
|---|---|---|
| docker-compose.yml | racine/ | Orchestration de tous les services |
| .env | racine/ | Variables d'environnement (mots de passe, users) |
| prometheus.yml | monitoring/prometheus/ | Config des jobs de scraping Prometheus |
| .my.cnf | monitoring/mysqld-exporter/ | Credentials MySQL pour mysqld-exporter |
| datasource.yml | monitoring/grafana/provisioning/datasources/ | Datasource Prometheus dans Grafana |
| dashboard.yml | monitoring/grafana/provisioning/dashboards/ | Config du chargement des dashboards |
| asv-dashboard.json | monitoring/grafana/provisioning/dashboards/ | Dashboard Grafana ASV App |
| Dockerfile | backend/ | Image PHP avec APCu |
| prometheus_metrics.yaml | backend/config/packages/ | Config du bundle Symfony |
| metrics.yaml | backend/config/routes/ | Route /metrics exposée par Symfony |

### Arborescence complète

```
ASV/
├── docker-compose.yml
├── .env
├── backend/
│   ├── Dockerfile
│   ├── config/
│   │   ├── packages/
│   │   │   └── prometheus_metrics.yaml
│   │   └── routes/
│   │       └── metrics.yaml
│   └── src/Controller/Api/AuthController.php
├── nginx/
│   └── default.conf
└── monitoring/
    ├── prometheus/
    │   └── prometheus.yml
    ├── mysqld-exporter/
    │   └── .my.cnf
    └── grafana/
        └── provisioning/
            ├── datasources/
            │   └── datasource.yml
            └── dashboards/
                ├── dashboard.yml
                └── asv-dashboard.json
```

---

## 4. Services mis en place

### 4.1 Node Exporter (port 9100)

Collecte les métriques système de la machine hôte : CPU, RAM, disque, réseau. Utile pour surveiller les ressources globales du serveur, indépendamment de Docker.

- Image : prom/node-exporter
- Emplacement : racine docker-compose.yml
- Aucune configuration supplémentaire requise

### 4.2 cAdvisor (port 8082)

Collecte les métriques de chaque container Docker individuellement : consommation CPU, RAM, réseau par container. Permet de voir quel service consomme le plus.

- Image : gcr.io/cadvisor/cadvisor:latest
- Monte les volumes système Docker en lecture seule
- Sur Windows/WSL, des warnings fsHandler peuvent apparaître — ils sont ignorables

### 4.3 mysqld-exporter (port 9104)

Expose les métriques internes de MySQL : nombre de queries, connexions actives, slow queries, état des tables. Nécessite un fichier de credentials `.my.cnf` car la version 0.19.0 n'accepte plus la variable `DATA_SOURCE_NAME`.

- Image : prom/mysqld-exporter
- Credentials dans monitoring/mysqld-exporter/.my.cnf
- L'utilisateur MySQL doit avoir les droits PROCESS, REPLICATION CLIENT, SELECT

Droits MySQL à accorder :

```sql
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'asv_user'@'%';
FLUSH PRIVILEGES;
```

### 4.4 Prometheus (port 9090)

Moteur central du monitoring. Il scrape (collecte) toutes les 15 secondes les métriques exposées par les autres services, et les stocke sous forme de time-series. L'interface web permet de tester des requêtes PromQL.

- Image : prom/prometheus
- Config : monitoring/prometheus/prometheus.yml
- Jobs configurés : prometheus, node-exporter, cadvisor, mysqld-exporter, symfony

### 4.5 Grafana (port 3000)

Interface de visualisation. Se connecte à Prometheus comme datasource et affiche les dashboards. La configuration est entièrement provisionnée via des fichiers, ce qui garantit la persistance.

- Image : grafana/grafana
- Datasource auto-configurée via datasource.yml (uid: prometheus)
- Dashboard auto-chargé via dashboard.yml + asv-dashboard.json
- `allowUiUpdates: true` permet l'édition depuis l'interface

### 4.6 Bundle Symfony artprima/prometheus-metrics-bundle

Installe une route `/metrics` dans l'API Symfony qui expose automatiquement les métriques HTTP (nombre de requêtes, durée, statuts). Installé dans le container PHP via Composer.

- Route : GET /metrics (accessible via nginx sur le port 8080)
- Namespace des métriques : asv
- Storage : APCu (persistance entre les requêtes)
- APCu installé dans le Dockerfile via `pecl install apcu`

---

## 5. Métriques métier custom

En plus des métriques automatiques, des compteurs custom ont été ajoutés dans `AuthController.php` pour suivre les actions utilisateurs importantes.

### 5.1 Compteur de connexions

Ajouté dans la méthode `login()` après validation des identifiants :

```php
$this->registry
    ->getOrRegisterCounter(
        'asv',
        'user_login_total',
        'Nombre de connexions reussies',
        ['role']
    )
    ->inc([$user->getRole()]);
```

Ce compteur permet de savoir combien de fois chaque type d'utilisateur (client, vétérinaire, responsable...) s'est connecté avec succès.

### 5.2 Compteur d'inscriptions

Ajouté dans la méthode `register()` après la création du compte :

```php
$this->registry
    ->getOrRegisterCounter(
        'asv',
        'user_register_total',
        'Nombre d\'inscriptions',
        ['role']
    )
    ->inc([$user->getRole()]);
```

Ce compteur suit les inscriptions par rôle, utile pour comprendre qui utilise l'application.

### 5.3 Injection du CollectorRegistry

Pour utiliser ces compteurs, le `CollectorRegistry` de Prometheus est injecté dans le constructeur du controller :

```php
use Prometheus\CollectorRegistry;

public function __construct(
    private CollectorRegistry $registry,
) {}
```

---

## 6. Dashboard Grafana — ASV App

Le dashboard `asv-dashboard.json` contient 5 sections avec les panneaux suivants :

### 6.1 Santé des containers

| Panneau | Type | Métrique PromQL |
|---|---|---|
| API (Nginx + PHP-FPM) UP/DOWN | Stat | `up{job="symfony"}` |
| MySQL Exporter UP/DOWN | Stat | `up{job="mysqld-exporter"}` |
| Node Exporter UP/DOWN | Stat | `up{job="node-exporter"}` |
| cAdvisor UP/DOWN | Stat | `up{job="cadvisor"}` |

> Note : Le job « symfony » scrape `/metrics` via la chaîne nginx → php-fpm → Symfony. Si l'un des trois services tombe, Prometheus marque l'ensemble comme DOWN.

### 6.2 Trafic HTTP Symfony

| Panneau | Type | Métrique PromQL |
|---|---|---|
| Requêtes / seconde | Time series | `rate(asv_http_requests_total[5m])` |
| Latence P95 | Time series | `histogram_quantile(0.95, ...)` |
| Total requêtes | Stat | `asv_http_requests_total{action="all"}` |

### 6.3 Métriques métier

| Panneau | Type | Métrique PromQL |
|---|---|---|
| Connexions par rôle | Time series | `asv_user_login_total` |
| Inscriptions par rôle | Time series | `asv_user_register_total` |
| Total connexions | Stat | `sum(asv_user_login_total)` |
| Total inscriptions | Stat | `sum(asv_user_register_total)` |

### 6.4 MySQL

| Panneau | Type | Métrique PromQL |
|---|---|---|
| Queries / seconde | Time series | `rate(mysql_global_status_queries[5m])` |
| Connexions actives | Gauge | `mysql_global_status_threads_connected` |
| Slow Queries | Time series | `rate(mysql_global_status_slow_queries[5m])` |

### 6.5 CPU & RAM par container

| Panneau | Type | Métrique PromQL |
|---|---|---|
| CPU par container | Time series | `rate(container_cpu_usage_seconds_total{name=~"php\|nginx\|mysql"}[5m])` |
| RAM par container | Time series | `container_memory_usage_bytes{name=~"php\|nginx\|mysql"}` |

---

## 7. Notes importantes

### 7.1 Persistance des métriques Symfony

Le storage APCu est utilisé pour persister les métriques entre les requêtes PHP. Sans APCu (avec `in_memory`), les compteurs se remettent à zéro à chaque requête.

- APCu est installé dans le Dockerfile via : `pecl install apcu && docker-php-ext-enable apcu`
- Config dans prometheus_metrics.yaml : `storage: apcu`

### 7.2 Connexions MySQL

Le panneau « Connexions actives MySQL » affiche toujours au moins 1 connexion car mysqld-exporter maintient une connexion permanente pour scraper les métriques. Le nombre réel de connexions applicatives = valeur affichée - 1.

### 7.3 Persistance du dashboard Grafana

Le dashboard est provisionné via des fichiers dans `monitoring/grafana/provisioning/`. Il se recharge automatiquement au démarrage de Grafana. Pour ne pas perdre les modifications faites dans l'interface, `allowUiUpdates: true` est configuré dans `dashboard.yml`.

### 7.4 Réseau Docker

Tous les services doivent être sur le même réseau `app-network` pour que Prometheus puisse les contacter par leur nom de service (ex: mysqld-exporter, cadvisor). Les noms de services dans `prometheus.yml` correspondent exactement aux noms de services dans `docker-compose.yml`.

### 7.5 Ports exposés

| Service | Port | Accès |
|---|---|---|
| Grafana | 3000 | http://localhost:3000 |
| Prometheus | 9090 | http://localhost:9090 |
| API Symfony | 8080 | http://localhost:8080 |
| phpMyAdmin | 8081 | http://localhost:8081 |
| cAdvisor | 8082 | http://localhost:8082 |
| Node Exporter | 9100 | http://localhost:9100/metrics |
| mysqld-exporter | 9104 | http://localhost:9104/metrics |
| Metrics Symfony | 8080 | http://localhost:8080/metrics |

---

*Mélissa Bedhomme — Dossier professionnel CDA RNCP36490 — Mai 2026*
