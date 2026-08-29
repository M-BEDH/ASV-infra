# ASV — Application de Suivi Vétérinaire

Application de gestion multi-établissements pour cliniques vétérinaires, refuges et associations.
Développée dans le cadre du titre professionnel **Concepteur Développeur d'Applications (RNCP37873)**.

## Architecture

| Composant | Technologie | Repo |
|---|---|---|
| API REST | Symfony 7 / PHP 8.2 | [ASV-backend](https://github.com/M-BEDH/ASV-backend) 
| Application web/mobile | Expo / React Native | [ASV-frontend](https://github.com/M-BEDH/ASV-frontend) 
| Infra | Docker Compose + Nginx | ce repo |

## Services Docker

| Service | URL locale |
|---|---|
| API Symfony | http://localhost:8080 |
| phpMyAdmin | http://localhost:8081 |
| Grafana (monitoring) | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Mailpit (mails dev) | http://localhost:8025 |

## Prérequis

Créer le fichier `monitoring/mysqld-exporter/.my.cnf` (non commité) :

```ini
[client]
user=MYSQL_USER
password=MYSQL_PASSWORD
host=mysql
port=3306
```

## Lancer le projet

```bash
cp .env.example .env  # renseigner les variables puis à la racine du projet :
docker compose up -d
```

## Documents

Le dossier `docs/` contient : 
- Diagrammes UML
- Plan de tests
