# Rapport de tests — Application ASV

**Projet :** Application de Suivi Vétérinaire (ASV)
**Titre professionnel visé :** CDA — RNCP37873
**Date :** 27/08/2026 (mise à jour — version initiale du 15/04/2026)
**Auteure :** Mélissa Bedhomme

---

## 1. Introduction

Ce rapport présente les résultats de l'exécution des tests automatisés de l'application ASV.
Les tests ont été écrits en suivant la démarche TDD (Test Driven Development) :
- **Phase RED** : le test est écrit en premier, avec le comportement attendu correct ; il échoue car le code (ou une partie du code) n'existe pas encore ou est incomplet
- **Phase GREEN** : le code minimal nécessaire est écrit pour faire passer le test
- **Refactor** : le code est ensuite nettoyé si besoin, sans changer le comportement, en s'appuyant sur le test comme filet de sécurité

Deux outils ont été utilisés :
- **PHPUnit** pour le backend Symfony (tests unitaires sur les entités + tests d'intégration sur l'API REST)
- **Jest** pour le frontend React Native (tests unitaires sur les fonctions utilitaires + tests de composants)

> **Note sur cette mise à jour :** depuis la version initiale du 15/04/2026, un nouveau test d'intégration (`tests/Controller/AdoptionFlowTest.php`) a été ajouté pour couvrir le parcours adoption refuge → inscription en clinique (cf. `CLAUDE.md`), et `MedicalConsultationControllerTest.php` s'est enrichi de 6 cas supplémentaires.

---

## 2. Résultats des tests frontend — Jest

### 2.1 Démarche TDD — `utils/dateUtils.ts` (fonction `pad`)

Le test attend `pad(5) === '05'`.

Commande exécutée :
```
npx jest utils/dateUtils.test.ts --no-coverage
```

**Phase RED** — sortie réelle obtenue :
```
FAIL utils/dateUtils.test.ts
  pad
    ✕ ajoute un zéro devant un nombre à 1 chiffre (6 ms)
    ✕ ne modifie pas un nombre à 2 chiffres
    ✕ gère le zéro
  dateToDisplay
    ✕ formate une date au format JJ-MM-AAAA HH:MM
    ✕ formate correctement une date en fin de mois
    ✕ formate correctement minuit (00:00)

  ● pad › ajoute un zéro devant un nombre à 1 chiffre

    TypeError: (0 , _dateUtils.pad) is not a function

      5 |     expect(pad(5)).toBe('05');

  ● dateToDisplay › formate une date au format JJ-MM-AAAA HH:MM

    ReferenceError: pad is not defined

      2 | export const dateToDisplay = (d: Date): string =>
      3 |   `${pad(d.getDate())}-${pad(d.getMonth() + 1)}-${d.getFullYear()} ${pad(d.getHours())}:${pad(d.getMinutes())}`;

Tests: 6 failed, 6 total
```

**Phase GREEN** — sortie réelle obtenue :
```
PASS utils/dateUtils.test.ts
  pad
    ✓ ajoute un zéro devant un nombre à 1 chiffre
    ✓ ne modifie pas un nombre à 2 chiffres
    ✓ gère le zéro
  dateToDisplay
    ✓ formate une date au format JJ-MM-AAAA HH:MM
    ✓ formate correctement une date en fin de mois
    ✓ formate correctement minuit (00:00)

Tests: 6 passed, 6 total
```

### 2.2 Démarche TDD — `components/FieldLabel.tsx`

Le test attend l'affichage du texte `'requis'` quand la prop `required` vaut `true`.

Commande exécutée :
```
npx jest components/FieldLabel.test.tsx --no-coverage
```

**Phase RED** — sortie réelle obtenue :
```
FAIL components/FieldLabel.test.tsx
  FieldLabel
    ✓ affiche le texte du label
    ✓ n'affiche pas 'requis' par défaut
    ✕ affiche 'requis' quand required={true}

  ● FieldLabel › affiche 'requis' quand required={true}

    Unable to find an element with text: requis

Tests: 1 failed, 2 passed, 3 total
```

**Phase GREEN** — sortie réelle obtenue :
```
PASS components/FieldLabel.test.tsx
  FieldLabel
    ✓ affiche le texte du label
    ✓ n'affiche pas 'requis' par défaut
    ✓ affiche 'requis' quand required={true}

Tests: 3 passed, 3 total
```

### 2.3 Résultat final Jest

Commande exécutée :
```
npx jest --no-coverage
```

Sortie réelle obtenue :
```
PASS utils/dateUtils.test.ts (7.928 s)
PASS components/FieldLabel.test.tsx (18.807 s)

Test Suites: 2 passed, 2 total
Tests:       9 passed, 9 total
Snapshots:   0 total
Time:        31.533 s
Ran all test suites.
```

**Bilan :** 9 tests, 9 passés, 0 échoué

| Suite | Tests | Résultat |
|---|---|---|
| `utils/dateUtils.test.ts` | 6 | ✅ |
| `components/FieldLabel.test.tsx` | 3 | ✅ |
| **Total** | **9** | **✅** |

---

## 3. Résultats des tests backend — PHPUnit

### 3.1 Démarche TDD — `src/Entity/Animal.php` (`setNom`)

Le test `testNomEtEspece` attend que `getNom()` retourne `'Rex'` après `setNom('Rex')`.

Commande exécutée :
```
docker compose exec php bin/phpunit tests/Entity/AnimalTest.php
```

**Phase RED** — sortie réelle obtenue :
```
E....                                                               5 / 5 (100%)

There was 1 error:

1) App\Tests\Entity\AnimalTest::testNomEtEspece
Error: Call to undefined method App\Entity\Animal::setNom()

/var/www/html/tests/Entity/AnimalTest.php:13

Tests: 5, Assertions: 11, Errors: 1.
```

**Phase GREEN** — sortie réelle obtenue :
```
.....                                                               5 / 5 (100%)

OK (5 tests, 13 assertions)
```

### 3.2 Démarche TDD — `src/Entity/User.php` (contrainte `#[Assert\Email]`)

Le test `testEmailFormatValide` attend qu'un email invalide (`'pas-un-email'`) produise une erreur de validation.

Commande exécutée :
```
docker compose exec php bin/phpunit tests/Entity/UserTest.php
```

**Phase RED** — sortie réelle obtenue :
```
.......F                                                            8 / 8 (100%)

There was 1 failure:

1) App\Tests\Entity\UserTest::testEmailFormatValide
Failed asserting that 0 is greater than 0.

Tests: 8, Assertions: 13, Failures: 1.
```

**Phase GREEN** — sortie réelle obtenue :
```
........                                                            8 / 8 (100%)

OK (8 tests, 14 assertions)
```

### 3.3 Résultat final PHPUnit

Commande exécutée :
```
docker compose exec php bin/phpunit tests/
```

Sortie réelle obtenue :
```
PHPUnit 11.5.55 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.2.33
Configuration: /var/www/html/phpunit.dist.xml

..............................................                    46 / 46 (100%)

Time: 00:33.691, Memory: 42.50 MB

OK (46 tests, 91 assertions)
```

**Bilan :** 46 tests, 91 assertions, 0 échoué

| Suite | Tests | Assertions | Résultat |
|---|---|---|---|
| `tests/Entity/AnimalTest.php` | 5 | 13 | ✅ |
| `tests/Entity/UserTest.php` | 8 | 14 | ✅ |
| `tests/Controller/UserControllerTest.php` | 6 | 8 | ✅ |
| `tests/Controller/AnimalControllerTest.php` | 7 | 10 | ✅ |
| `tests/Controller/OwnerControllerTest.php` | 6 | 9 | ✅ |
| `tests/Controller/MedicalConsultationControllerTest.php` *(+6 depuis la v. du 15/04)* | 13 | 18 | ✅ |
| `tests/Controller/AdoptionFlowTest.php` **(nouveau)** | 1 | 19 | ✅ |
| **Total** | **46** | **91** | **✅** |

### 3.4 Nouveau scénario — `tests/Controller/AdoptionFlowTest.php`

Ce test d'intégration couvre le parcours complet **refuge → adoption → inscription en vraie clinique** :

1. Un animal est recueilli par un refuge, sans propriétaire.
2. Le staff du refuge voit l'animal et son historique de consultations.
3. L'adoption est enregistrée : un `Owner` est créé comme simple trace de l'adoptant, **jamais rattaché** au refuge (`clinicIds` vide vérifié).
4. Aucun précompte `User` client n'est créé tant qu'aucune vraie clinique n'a enregistré cet email (`register` renvoie 403 entre-temps).
5. Après adoption, le staff du refuge garde l'accès à l'animal et à son historique.
6. Une vraie clinique enregistre ensuite un `Owner` avec le même email : le serveur retrouve la trace laissée par le refuge via l'email mais, celle-ci n'ayant aucune clinique rattachée, il crée un **nouvel** `Owner` pour la clinique plutôt que de réutiliser la trace refuge.
7. Le précompte client est alors créé, rattaché à la vraie clinique et jamais au refuge.

---

## 4. Bilan global

| Catégorie | Outil | Tests | Résultat |
|---|---|---|---|
| Tests unitaires frontend | Jest | 6 | ✅ |
| Tests de composants frontend | Jest + @testing-library/react-native | 3 | ✅ |
| Tests unitaires backend | PHPUnit (Entity) | 13 | ✅ |
| Tests d'intégration backend | PHPUnit (Controller, dont adoption) | 33 | ✅ |
| **Total** | | **55** | **✅ 55 passés** |
