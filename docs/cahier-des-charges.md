# Cahier des charges — ASV (Application de Suivi Vétérinaire)

## 1. Contexte et présentation du besoin

ASV est né d'abord d'un constat côté propriétaires d'animaux : ceux-ci disposent rarement d'un suivi centralisé et visuel de l'historique médical de leurs animaux, cet historique restant dispersé entre les cliniques ou peu formalisé. Le projet s'est ensuite élargi pour répondre aussi aux besoins des structures — cliniques vétérinaires,  refuges, associations. **ASV** doit donc fournir un outil numérique centralisé qui couvre à la fois le suivi côté propriétaire et la gestion côté structure, avec des règles d'accès qui s'adaptent au type d'établissement et au rôle de chaque utilisateur.

Le projet est développé dans le cadre du titre professionnel **Concepteur Développeur d'Applications (CDA, RNCP37873)**.

### 1.1 Définitions

| Terme | Définition |
|---|---|
| Clinique / Établissement | Structure vétérinaire de type `clinique`, `refuge` ou `association` |
| Propriétaire | Personne physique rattachée à un ou plusieurs animaux, optionnellement liée à un compte utilisateur `client` |
| Animal | Entité centrale du suivi médical : espèce, race, propriétaire (facultatif), historique de consultations |
| Consultation médicale | Acte vétérinaire daté, rattaché à un animal, un vétérinaire et une clinique |
| Pré-compte | Compte créé par un responsable/administrateur, activé ensuite par son titulaire — aucune inscription libre |
| Multi-tenant | Isolation des données par établissement : chaque utilisateur n'accède qu'aux données de son propre établissement, sauf exceptions métier définies |

### 1.2 Objectifs

**Court terme**
- Sécuriser l'accès aux données selon l'établissement et le rôle de chaque utilisateur
- Fournir une application mobile/web utilisable sur le terrain (smartphone) comme au comptoir (tablette/desktop)
- Fournir une interface d'administration pour la création des établissements et de leurs responsables
- Assurer une supervision opérationnelle minimale (monitoring)

**Long terme (évolutions envisagées, hors périmètre actuel)**
- Prescriptions, suivi vaccinal, gestion de stock
- Notifications (rappels de vaccination, de rendez-vous)
- Prise de rendez-vous en ligne par le client
- Pipeline de déploiement automatisé (build + tests + mise en production)

### 1.3 Hors périmètre -- pour une V3

- Tests de charge/performance
- Paiement en ligne, facturation
- Notifications push/email envoyées à l'utilisateur
- Prise de rendez-vous par le client (consultation en lecture seule uniquement)
- Diffusion sur stores publics (usage interne/démonstration à ce stade)

---

## 2. Public cible et personas

Six profils utilisateurs doivent être servis par l'application, avec des besoins et des niveaux d'accès distincts.

### Persona 1 — Nadia, Cliente (propriétaire d'animal)

- **Contexte** : 42 ans, propriétaire de deux chats suivis dans une clinique de quartier. Utilise principalement son smartphone, à l'aise avec les usages numériques courants mais pas technicienne.
- **Objectifs** : consulter l'historique médical de ses animaux sans avoir à appeler la clinique ; retrouver ses animaux même si elle change de clinique.
- **Besoins d'accès** : lecture seule sur ses propres animaux et leurs consultations, quelle que soit la clinique où ils sont suivis. Peut être liée à plusieurs cliniques si ses animaux y sont suivis séparément.
- **Limites** : ne peut ni modifier les fiches de ses animaux ni prendre rendez-vous (peut en revanche modifier ou supprimer sa propre fiche propriétaire) ; l'activation de son compte dépend d'un pré-compte créé par la clinique (elle doit utiliser le même email).
- **Niveau numérique** : intermédiaire.

### Persona 2 — Karim, Bénévole en refuge associatif

- **Contexte** : 29 ans, bénévole 2 jours/semaine dans un refuge associatif, souvent seule présence sur place. Peu formé aux outils informatiques métier.
- **Objectifs** : enregistrer rapidement un animal recueilli, mettre à jour son état sans dépendre en permanence d'un vétérinaire ou d'un responsable.
- **Besoins d'accès** : écriture sur les fiches animaux du refuge uniquement ; aucune donnée médicale sensible, aucun accès aux fiches propriétaires (un animal en refuge n'a normalement pas de propriétaire).
- **Limites** : pas d'accès à la donnée médicale, même en refuge — restriction volontaire du projet, réservée au personnel qualifié.
- **Niveau numérique** : basique, utilise surtout un smartphone.

### Persona 3 — Léa, Assistante vétérinaire

- **Contexte** : 26 ans, assistante dans une clinique classique, gère l'accueil et la mise à jour des dossiers.
- **Objectifs** : créer et tenir à jour les fiches animaux et propriétaires de sa clinique, préparer les dossiers avant consultation.
- **Besoins d'accès** : écriture sur les fiches animaux/propriétaires de sa clinique ; peut créer et modifier des consultations médicales (mêmes droits que le vétérinaire sur ce point, mais ne peut pas être désigné comme le vétérinaire traitant) ; aucun accès à la gestion d'équipe.
- **Niveau numérique** : intermédiaire à bon.

### Persona 4 — Dr. Moreau, Vétérinaire

- **Contexte** : 38 ans, vétérinaire salarié, réalise les consultations et pose les diagnostics.
- **Objectifs** : consulter l'historique médical complet d'un animal avant de statuer, créer/modifier des consultations rapidement entre deux rendez-vous.
- **Besoins d'accès** : lecture/écriture complète sur les animaux, propriétaires et consultations de sa clinique.
- **Limites** : ne peut pas intervenir sur une consultation d'une autre clinique, même en cas de remplacement ponctuel.
- **Niveau numérique** : bon, utilise l'app entre deux consultations sur smartphone ou tablette.

### Persona 5 — Sophie, Responsable de clinique

- **Contexte** : 45 ans, gérante et responsable d'une clinique de 6 personnes.
- **Objectifs** : constituer son équipe, modifier les informations de sa clinique, superviser l'activité globale.
- **Besoins d'accès** : tous les droits du vétérinaire, plus la création/modification de comptes staff et l'édition des informations de l'établissement.
- **Limites** : ne peut créer des comptes que dans sa propre clinique ; dépend d'un administrateur pour la création initiale de sa clinique.
- **Niveau numérique** : bon.

### Persona 6 — Admin technique (super-administrateur)

- **Contexte** : administrateur technique de la plateforme, seul profil à accéder à l'interface d'administration globale, en dehors de l'application métier mobile/web.
- **Objectifs** : créer les établissements et leurs responsables, superviser l'ensemble des données tous établissements confondus, dépanner en cas d'anomalie.
- **Besoins d'accès** : accès total, via une interface d'administration distincte de l'application métier.
- **Niveau numérique** : expert (profil technique).

---

## 3. Exigences d'accès par rôle (BF06, BF07)

| Rôle | Lecture | Écriture |
|---|---|---|
| Client | Ses propres animaux et consultations, toutes cliniques confondues | Sa propre fiche propriétaire uniquement (modification, suppression/anonymisation) ; aucun droit sur les animaux ni les consultations |
| Bénévole | Établissement affecté | Animaux, uniquement en refuge/association ; jamais propriétaires ni consultations |
| Assistant | Établissement affecté | Animaux, propriétaires et consultations de son établissement (mêmes droits que le vétérinaire, sauf être désigné praticien traitant) |
| Vétérinaire | Établissement affecté | Animaux, propriétaires et consultations de son établissement |
| Responsable | Établissement affecté | Idem vétérinaire + gestion d'équipe + informations de l'établissement |
| Super-administrateur | Tous établissements | Tous établissements, via l'interface d'administration |

**Règle métier — propriétaire et type d'établissement** : un propriétaire ne doit jamais être rattaché à un refuge ou une association, seulement à une clinique vétérinaire à proprement parler. Un animal en refuge/association n'a donc normalement pas de propriétaire tant qu'il n'a pas été adopté ; l'adoption crée une fiche propriétaire comme simple trace de l'adoptant, sans la rattacher au refuge — celui-ci choisit ensuite sa clinique vétérinaire séparément.

---

## 4. Exigences fonctionnelles

### 4.1 Authentification et gestion de compte (BF01)

- Connexion par email/mot de passe ; sélection de l'établissement si le compte est rattaché à plusieurs
- Activation de compte par pré-compte uniquement (pas d'inscription libre)
- Règle de mot de passe : minimum 6 caractères, au moins une majuscule, une minuscule, un chiffre, un caractère spécial

### 4.2 Gestion des établissements (BF02)

Consultation de la liste des établissements (pour l'inscription), consultation et modification des informations de son propre établissement (responsable). Création d'un établissement et de son responsable réservée à l'administration.

### 4.3 Gestion des propriétaires (BF03)

Création, consultation, modification, suppression d'une fiche propriétaire, limitées à l'établissement de l'utilisateur. Détection de doublon par email au sein d'un même établissement. La suppression anonymise les données personnelles plutôt que de les effacer. Un client (rôle `client`) peut également modifier ou supprimer sa propre fiche propriétaire, quelle que soit sa clinique.

### 4.4 Gestion des animaux (BF04)

Création, consultation, modification, suppression d'une fiche animal. Un animal peut changer de propriétaire (flux d'adoption). Un client voit ses animaux quelle que soit la clinique où ils sont suivis.

### 4.5 Consultations médicales (BF05)

Création et modification réservées au personnel vétérinaire qualifié (vétérinaire, responsable, assistant) — seul un utilisateur avec `isVet` à vrai (typiquement un vétérinaire, ou un responsable qui l'est aussi) peut être désigné comme praticien traitant sur la consultation. Une consultation comporte un motif (obligatoire), un compte-rendu et des traitements (facultatifs). Visibilité en lecture adaptée selon le profil (client : ses animaux ; staff : son établissement).

### 4.6 Gestion d'équipe

Création de comptes pour le personnel (assistant, vétérinaire, bénévole), réservée au responsable, limitée à son propre établissement. Modification de rôle réservée au responsable. Suppression de compte réservée au responsable ou au vétérinaire.

### 4.7 Administration (BF08)

Interface d'administration dédiée, réservée au super-administrateur, permettant la gestion de l'ensemble des données tous établissements confondus.

---

## 5. Exigences non fonctionnelles

### 5.1 Sécurité et protection des données

- Authentification requise sur l'ensemble des fonctionnalités, hors connexion/inscription/liste publique des établissements
- Mots de passe soumis à une règle de complexité et jamais stockés en clair
- Protection contre l'exécution de scripts et le détournement de trafic (en-têtes de sécurité HTTP)
- Anonymisation des données personnelles à la suppression d'un compte propriétaire ou utilisateur, plutôt qu'un effacement physique (traçabilité de l'historique médical préservée)

### 5.2 Disponibilité et supervision (BF10)

Le système doit exposer des indicateurs opérationnels minimaux (disponibilité, activité par rôle) permettant une supervision de son fonctionnement.

### 5.3 Accessibilité et ergonomie

Interface utilisable en environnement clinique (tablette/desktop) comme sur le terrain (smartphone), avec support d'un thème clair/sombre et des attributs d'accessibilité de base.

### 5.4 Multi-plateforme (BF09)

L'application doit fonctionner sur Android et Web à partir d'une base de code commune.

---

## 6. Critères de recette

Le projet est considéré conforme au besoin si :

1. Chaque rôle applicatif n'accède qu'aux fonctionnalités et données autorisées par la matrice d'accès (§3), y compris les exceptions refuge/association — vérifiable par les tests d'intégration dédiés aux interdictions par rôle.
2. Un propriétaire n'est jamais rattaché à un refuge ou une association, quelle que soit la voie de création utilisée — vérifiable par un scénario de bout en bout (création en refuge → adoption → inscription en clinique).
3. Toute route de l'API, hors liste blanche publique documentée, exige une authentification valide.
4. La suppression d'un compte propriétaire ou utilisateur anonymise ses données personnelles sans les effacer physiquement, ni casser l'historique médical associé.
5. L'application est utilisable sans divergence fonctionnelle majeure sur Android et Web.
6. Les indicateurs de connexion et d'inscription par rôle sont visibles dans le tableau de bord de supervision.

