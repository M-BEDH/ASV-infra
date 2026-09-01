# Plan de tests — Application ASV

**Projet :** Application de Suivi Vétérinaire (ASV)
**Titre professionnel visé :** CDA — RNCP37873
**Date :** 27/03/2026
**Auteure :** Mélissa Bedhomme

---

## 1. Objectif

Vérifier le bon fonctionnement des fonctions utilitaires, des entités métier et des endpoints de l'API REST, ainsi que l'affichage correct des composants de l'interface mobile.

---

## 2. Portée (scope)

**Fonctionnalités couvertes par les tests :**
- Entités métier backend : `Animal`, `User` (tests unitaires PHPUnit)
- Endpoints API REST : authentification, animaux, propriétaires, consultations (tests d'intégration PHPUnit)
- Fonctions utilitaires frontend : `pad()`, `dateToDisplay()` (tests unitaires Jest)
- Composant frontend : `FieldLabel` (test de composant Jest)

**Fonctionnalités non couvertes :**
- Interface graphique complète (pas de tests end-to-end)
- Performances et montée en charge
- Tests sur les autres composants React Native (FormModal, ConfirmModal, etc.)
- Tests sur le monitoring (Prometheus, Grafana)

---

## 3. Analyse des risques

| Risque | Impact | Mesure de couverture |
|---|---|---|
| Pollution des données de production par les tests | Élevé | Base de test dédiée `asv_db_test`, isolée de la production |
| Dépendance à AsyncStorage dans les composants | Moyen | `useTheme` mocké dans les tests de composants |
| Faux positifs (tests toujours verts) | Élevé | Démarche TDD appliquée : phase RED vérifiée avant phase GREEN |
| Régression lors d'une modification de l'API | Moyen | Tests d'intégration couvrant tous les endpoints CRUD |

**Risques résiduels :**
- L'interface graphique n'est pas testée automatiquement, des bugs visuels peuvent subsister.
- Les tests d'intégration dépendent de la disponibilité de Docker en local.

---

## 4. Calendrier

| Étape | Date |
|---|---|
| Début des tests unitaires backend (PHPUnit entités) | 14/03/2026 |
| Fin des tests d'intégration backend (PHPUnit contrôleurs) | 18/03/2026 |
| Mise en place de Jest + tests frontend | 25/03/2026 |
| Fin de tous les tests | 27/03/2026 |

---

## 5. Critères d'acceptation

- Tous les tests doivent passer à **100%** : 0 échec toléré
- La démarche TDD doit être démontrée : au moins une capture RED et une capture GREEN par suite de tests
- Les tests doivent s'exécuter sans modifier les données de production

---

## 6. Livrables

| Document | Description |
|---|---|
| `plan-de-tests.md` / PDF | Ce document |
| `rapport-de-tests.md` / PDF | Résultats d'exécution avec captures RED/GREEN |
| `tests/Entity/AnimalTest.php` | Tests unitaires PHPUnit sur l'entité Animal |
| `tests/Entity/UserTest.php` | Tests unitaires PHPUnit sur l'entité User |
| `tests/Controller/*.php` | Tests d'intégration PHPUnit sur l'API REST |
| `utils/dateUtils.test.ts` | Tests unitaires Jest sur les fonctions utilitaires |
| `components/FieldLabel.test.tsx` | Test de composant Jest |

---

## 7. Environnements de test

| Environnement | Détail |
|---|---|
| Backend | Docker, PHP 8.2, Symfony 7, base MySQL `asv_db_test` |
| Frontend | Node.js, Jest 29, jest-expo, @testing-library/react-native 13, React Native 0.81 |
| Outil backend | PHPUnit via `php bin/phpunit tests/` |
| Outil frontend | Jest via `npx jest --no-coverage` |

---

## 8. Tests unitaires frontend — Jest

### 3.1 Fonction `pad()` — `utils/dateUtils.ts`

| ID | Fonctionnalité | Entrée | Résultat attendu | Statut |
|---|---|---|---|---|
| UT-01 | Ajoute un zéro devant un nombre à 1 chiffre | `pad(5)` | `'05'` | ✅ |
| UT-02 | Ne modifie pas un nombre à 2 chiffres | `pad(12)` | `'12'` | ✅ |
| UT-03 | Gère le zéro | `pad(0)` | `'00'` | ✅ |

### 3.2 Fonction `dateToDisplay()` — `utils/dateUtils.ts`

| ID | Fonctionnalité | Entrée | Résultat attendu | Statut |
|---|---|---|---|---|
| UT-04 | Formate une date standard | `new Date(2024, 0, 5, 9, 7)` | `'05-01-2024 09:07'` | ✅ |
| UT-05 | Formate une date en fin de mois | `new Date(2023, 11, 31, 23, 59)` | `'31-12-2023 23:59'` | ✅ |
| UT-06 | Formate minuit (00:00) | `new Date(2024, 5, 1, 0, 0)` | `'01-06-2024 00:00'` | ✅ |

### 3.3 Composant `FieldLabel` — `components/FieldLabel.tsx`

| ID | Fonctionnalité | Entrée | Résultat attendu | Statut |
|---|---|---|---|---|
| CT-01 | Affiche le texte du label | `<FieldLabel>Nom de l'animal</FieldLabel>` | Le texte "Nom de l'animal" est visible | ✅ |
| CT-02 | N'affiche pas "requis" par défaut | `<FieldLabel>Espèce</FieldLabel>` | Le texte "requis" est absent | ✅ |
| CT-03 | Affiche "requis" quand `required={true}` | `<FieldLabel required>Date de naissance</FieldLabel>` | Le texte "requis" est visible | ✅ |

> *Les tests de composants utilisent `@testing-library/react-native` pour rendre le composant et interroger l'arbre de rendu. `useTheme` est mocké pour isoler le composant de son contexte de thème.*

---

## 9. Tests unitaires backend — PHPUnit

### 4.1 Entité `Animal` — `tests/Entity/AnimalTest.php`

| ID | Fonctionnalité | Étapes | Résultat attendu | Statut |
|---|---|---|---|---|
| UT-07 | Setters et getters nom/espèce | Créer un Animal, définir nom et espèce | Les valeurs sont restituées correctement | ✅ |
| UT-08 | UUID auto-généré au format v4 | Instancier un Animal | L'ID est un UUID v4 valide | ✅ |
| UT-09 | Deux animaux ont des IDs distincts | Instancier deux animaux | Les deux IDs sont différents | ✅ |
| UT-10 | `createdAt` défini à la création | Instancier un Animal | `createdAt` est un `DateTimeImmutable` non nul | ✅ |
| UT-11 | Valeurs optionnelles nulles par défaut | Instancier un Animal sans données | Tous les champs optionnels valent `null` | ✅ |

### 4.2 Entité `User` — `tests/Entity/UserTest.php`

| ID | Fonctionnalité | Étapes | Résultat attendu | Statut |
|---|---|---|---|---|
| UT-12 | Setters et getters email/nom | Créer un User, définir email et nom | Les valeurs sont restituées correctement | ✅ |
| UT-13 | UUID auto-généré au format v4 | Instancier un User | L'ID est un UUID v4 valide | ✅ |
| UT-14 | Deux utilisateurs ont des IDs distincts | Instancier deux Users | Les deux IDs sont différents | ✅ |
| UT-15 | Rôle vétérinaire | Définir rôle `veterinaire` | `getRoles()` retourne `['ROLE_VETERINAIRE']` | ✅ |
| UT-16 | Rôle client | Définir rôle `client` | `getRoles()` retourne `['ROLE_CLIENT']` | ✅ |
| UT-17 | `createdAt` défini à la création | Instancier un User | `createdAt` est un `DateTimeImmutable` non nul | ✅ |
| UT-18 | `getUserIdentifier()` retourne l'email | Appeler `getUserIdentifier()` | Retourne l'adresse email | ✅ |
| UT-19 | Validation format email | Passer un email invalide au validateur Symfony | Une violation de contrainte est détectée | ✅ |

---

## 10. Tests d'intégration backend — PHPUnit

### 5.1 Authentification — `tests/Controller/UserControllerTest.php`

| ID | Fonctionnalité | Étapes | Résultat attendu | Statut |
|---|---|---|---|---|
| IT-01 | Inscription d'un nouvel utilisateur | POST `/api/auth/register` avec email/password | HTTP 201, utilisateur créé | ✅ |
| IT-02 | Connexion valide | POST `/api/auth/login` avec identifiants corrects | HTTP 200, token JWT retourné | ✅ |
| IT-03 | Connexion avec mauvais mot de passe | POST `/api/auth/login` avec mauvais password | HTTP 401 | ✅ |
| IT-04 | Accès à une route protégée avec token | GET route protégée + header Authorization | HTTP 200 | ✅ |
| IT-05 | Accès à une route protégée sans token | GET route protégée sans header | HTTP 401 | ✅ |
| IT-06 | Inscription avec email déjà utilisé | POST `/api/auth/register` avec email existant | HTTP 409 ou erreur de validation | ✅ |

### 5.2 Animaux — `tests/Controller/AnimalControllerTest.php`

| ID | Fonctionnalité | Étapes | Résultat attendu | Statut |
|---|---|---|---|---|
| IT-07 | Lister les animaux | GET `/api/animals` avec token | HTTP 200, tableau JSON | ✅ |
| IT-08 | Créer un animal | POST `/api/animals` avec données valides | HTTP 201, animal créé | ✅ |
| IT-09 | Lire un animal par ID | GET `/api/animals/{id}` | HTTP 200, données de l'animal | ✅ |
| IT-10 | Modifier un animal | PUT `/api/animals/{id}` | HTTP 200, données mises à jour | ✅ |
| IT-11 | Supprimer un animal | DELETE `/api/animals/{id}` | HTTP 204 | ✅ |
| IT-12 | Accès refusé sans token | GET `/api/animals` sans header | HTTP 401 | ✅ |
| IT-13 | Animal inexistant | GET `/api/animals/{id_inconnu}` | HTTP 404 | ✅ |

### 5.3 Propriétaires — `tests/Controller/OwnerControllerTest.php`

| ID | Fonctionnalité | Étapes | Résultat attendu | Statut |
|---|---|---|---|---|
| IT-14 | Lister les propriétaires | GET `/api/owners` avec token | HTTP 200, tableau JSON | ✅ |
| IT-15 | Créer un propriétaire | POST `/api/owners` avec données valides | HTTP 201, propriétaire créé | ✅ |
| IT-16 | Lire un propriétaire par ID | GET `/api/owners/{id}` | HTTP 200, données du propriétaire | ✅ |
| IT-17 | Modifier un propriétaire | PUT `/api/owners/{id}` | HTTP 200, données mises à jour | ✅ |
| IT-18 | Supprimer un propriétaire | DELETE `/api/owners/{id}` | HTTP 204 | ✅ |
| IT-19 | Accès refusé sans token | GET `/api/owners` sans header | HTTP 401 | ✅ |

### 5.4 Consultations médicales — `tests/Controller/MedicalConsultationControllerTest.php`

| ID | Fonctionnalité | Étapes | Résultat attendu | Statut |
|---|---|---|---|---|
| IT-20 | Lister les consultations | GET `/api/consultations` avec token | HTTP 200, tableau JSON | ✅ |
| IT-21 | Lire une consultation par ID | GET `/api/consultations/{id}` | HTTP 200, données de la consultation | ✅ |
| IT-22 | Consultation inexistante | GET `/api/consultations/{id_inconnu}` | HTTP 404 | ✅ |
| IT-23 | Lecture refusée entre cliniques différentes | GET `/api/consultations/{id}` avec le token d'un vétérinaire d'une autre clinique | HTTP 403 (`MedicalConsultationVoter::canViewAnimal()`) | ✅ |
| IT-24 | Créer une consultation | POST `/api/consultations` avec données valides | HTTP 201, consultation créée | ✅ |
| IT-25 | Créer une consultation avec champs manquants | POST `/api/consultations` sans `animalId`/`dateConsultation` | HTTP 400 | ✅ |
| IT-26 | Création refusée pour un client | POST `/api/consultations` avec token client | HTTP 403 | ✅ |
| IT-27 | Création refusée pour un bénévole | POST `/api/consultations` avec token bénévole | HTTP 403 | ✅ |
| IT-28 | Modifier une consultation | PUT `/api/consultations/{id}` | HTTP 200, données mises à jour | ✅ |
| IT-29 | Supprimer une consultation | DELETE `/api/consultations/{id}` | HTTP 204 | ✅ |
| IT-30 | Suppression d'une consultation inexistante | DELETE `/api/consultations/{id_inconnu}` | HTTP 404 | ✅ |

### 5.5 Scénario adoption — `tests/Controller/AdoptionFlowTest.php`

| ID | Fonctionnalité | Étapes | Résultat attendu | Statut |
|---|---|---|---|---|
| IT-31 | Adoption en refuge puis inscription client en vraie clinique | Recueil d'un animal en refuge, création d'un Owner (trace de l'adoptant, non rattaché au refuge), tentative de `register` refusée avant qu'une vraie clinique n'intervienne, adoption, puis inscription du même email par une vraie clinique | L'Owner créé par le refuge n'a aucune clinique rattachée ; la clinique crée un nouvel Owner (pas de réutilisation) ; le précompte client final n'est lié qu'à la vraie clinique | ✅ |

---

## 11. Récapitulatif

| Catégorie | Outil | Nb tests | Résultat |
|---|---|---|---|
| Tests unitaires frontend | Jest | 6 | ✅ 6 passés |
| Tests de composants frontend | Jest + @testing-library/react-native | 3 | ✅ 3 passés |
| Tests unitaires backend | PHPUnit | 13 | ✅ 13 passés |
| Tests d'intégration backend | PHPUnit | 31 | ✅ 31 passés |
| **Total** | | **53** | **✅ 53 passés** |
