🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 7 : Intégration et APIs

## Introduction

SQLite ne se limite pas à être un simple fichier de base de données consulté via des requêtes SQL. Sa véritable puissance réside dans sa capacité à s'intégrer harmonieusement dans des écosystèmes applicatifs complexes. Ce chapitre explore les différentes approches pour incorporer SQLite dans vos projets de développement, depuis l'intégration native avec des langages de programmation jusqu'à la création d'APIs REST sophistiquées.

## Pourquoi intégrer SQLite dans vos applications ?

SQLite présente des avantages uniques pour l'intégration applicative :

**Simplicité de déploiement** : Contrairement aux SGBD traditionnels qui nécessitent une installation serveur, SQLite s'intègre directement dans votre application. Un seul fichier suffit à contenir l'ensemble de votre base de données, simplifiant considérablement le processus de déploiement.

**Performance optimale** : L'absence de couche réseau entre l'application et la base de données élimine la latence réseau. Les accès aux données se font directement au niveau du système de fichiers, offrant des performances exceptionnelles pour les applications à charge modérée.

**Intégration native** : SQLite dispose de bindings officiels pour la plupart des langages de programmation modernes. Cette intégration native permet un contrôle fin des opérations de base de données depuis votre code applicatif.

**Portabilité maximale** : Le format de fichier SQLite est standardisé et multiplateforme. Votre base de données fonctionne identiquement sur Windows, Linux, macOS, et même sur des systèmes embarqués.

## Architecture d'intégration typique

L'intégration de SQLite suit généralement une architecture en couches :

**Couche Application** : Votre logique métier qui définit les besoins en données et orchestre les opérations.

**Couche d'Accès aux Données (DAL)** : Interface qui abstrait les opérations SQLite et traduit les besoins applicatifs en requêtes SQL optimisées.

**Couche SQLite** : Le moteur de base de données lui-même, gérant la persistance, les transactions, et l'optimisation des requêtes.

**Couche Système de Fichiers** : Gestion physique du fichier de base de données sur le support de stockage.

Cette architecture modulaire permet une séparation claire des responsabilités et facilite la maintenance du code.

## Patterns d'intégration courants

### 1. Intégration Directe
L'application communique directement avec SQLite via les APIs natives du langage. Cette approche offre un contrôle maximal mais requiert une gestion explicite des connexions, transactions, et erreurs.

### 2. Couche d'Abstraction (Repository Pattern)
Création d'une couche d'abstraction qui encapsule les opérations SQLite. Cette approche améliore la testabilité et permet de changer facilement de système de base de données si nécessaire.

### 3. ORM (Object-Relational Mapping)
Utilisation d'un framework ORM qui mappe automatiquement les objets du langage vers les tables SQLite. Cette approche accélère le développement mais peut introduire une complexité supplémentaire.

### 4. API REST/GraphQL
Exposition des données SQLite via des APIs web, permettant l'accès depuis des clients distants tout en conservant les avantages de SQLite côté serveur.

## Considérations architecturales

### Gestion de la concurrence
SQLite gère nativement la concurrence via un système de verrous, mais les stratégies d'accès concurrent varient selon le langage d'intégration. Il est crucial de comprendre le modèle de threading de votre langage pour optimiser l'accès concurrent.

### Gestion des connexions
Contrairement aux SGBD serveur, SQLite n'a pas besoin d'un pool de connexions au sens classique (pas de coût d'établissement TCP, pas d'authentification). Cependant, dans un programme **multithread avec WAL**, créer **une connexion par thread** (ou utiliser une petite pool de 3–10 connexions de lecture + 1 connexion d'écriture) reste une bonne pratique pour permettre les lectures en parallèle. Évitez en revanche le partage d'une même connexion entre plusieurs threads sans verrou : `check_same_thread=False` en Python ou équivalents dans d'autres langages exigent une synchronisation manuelle.

### Sérialisation des données
L'intégration implique souvent la sérialisation/désérialisation entre les types natifs du langage et les types SQLite. Une stratégie claire de mapping des types est essentielle.

### Gestion d'erreurs
Chaque langage expose différemment les erreurs SQLite. Une stratégie cohérente de gestion d'erreurs améliore la robustesse de l'application.

## Bonnes pratiques transversales

**Préparation des requêtes** : Utilisez systématiquement les requêtes préparées pour améliorer les performances et la sécurité.

**Gestion transactionnelle** : Encapsulez les opérations logiques dans des transactions pour garantir la cohérence des données.

**Validation des données** : Implémentez une validation robuste côté application en complément des contraintes SQLite.

**Logging et monitoring** : Mettez en place une stratégie de logging des opérations critiques et de monitoring des performances.

**Tests d'intégration** : Développez des tests spécifiques pour valider l'intégration SQLite dans différents scénarios d'usage.

## Quand choisir SQLite — et quand passer à autre chose

Pour rester honnête, voici quelques règles pour décider si SQLite est la bonne base pour votre projet d'intégration.

### ✅ SQLite est un excellent choix si…
- **Charge de lecture dominante** : analytics, catalogues, configuration, lectures fréquentes et écritures occasionnelles.
- **Application desktop, mobile ou embarquée** : un fichier `.db` qui suit l'application est idéal.
- **Backend web modeste à moyen** : un blog, un wiki, un SaaS B2B jusqu'à ~ 100 000 utilisateurs actifs/jour est tout à fait gérable avec SQLite en mode WAL.
- **Tests automatisés** : `:memory:` rend les tests d'intégration très rapides.
- **Edge computing / IoT** : SQLite tient sur quelques centaines de Ko de RAM.
- **Cache local / offline-first** : combiné à une sync (voir 7.5), c'est la base de quasi toutes les apps mobiles modernes.

### ⚠️ SQLite reste possible mais demande de l'attention si…
- **Écritures concurrentes intenses** : un seul writer à la fois par base. Le mode WAL atténue le problème mais le plafond reste autour de quelques centaines d'écritures par seconde. Si vous insérez en lot, regroupez en transactions.
- **Données très volumineuses** : SQLite gère sans souci jusqu'à plusieurs centaines de Go, mais `VACUUM` devient lent et la sauvegarde simple-fichier n'est plus pratique.
- **Plusieurs machines accédant à la même base** : SQLite n'est pas conçu pour les filesystems réseau (NFS, SMB) — verrouillage non garanti. Solutions : `Litestream` pour la réplication continue, ou passez à `LibSQL`/`Turso` pour la réplication multi-régions.

### ❌ Choisissez plutôt PostgreSQL / MySQL / MariaDB si…
- Vous avez **plusieurs centaines d'utilisateurs en écriture concurrente** (e-commerce sur grosses promos, chat temps réel mondial, plateformes collaboratives intensives).
- Vous avez besoin de **rôles et permissions fins** côté base (`GRANT`/`REVOKE` par utilisateur, row-level security).
- Vous voulez de la **réplication maître-esclave native**, du failover automatique, du sharding.
- Vous utilisez des **fonctionnalités SQL avancées** absentes en SQLite : `MERGE`, vraies procédures stockées (PL/pgSQL), `MATERIALIZED VIEW` rafraîchissables, types riches (`UUID` natif, `JSONB` indexable, types géographiques avec `PostGIS`).
- Vous voulez du **streaming de changements** (logical replication, CDC) facile à brancher sur Kafka/Debezium.

### 🔄 Migration progressive
Si votre projet démarre avec SQLite et finit par dépasser ses limites, la migration vers PostgreSQL est bien outillée :
- **`pgloader`** : convertit une base SQLite en Postgres en une commande.
- **ORM identique** : SQLAlchemy, Sequelize, Hibernate, etc., changent seulement la chaîne de connexion.
- **Solutions hybrides** : `ElectricSQL` ou `PowerSync` permettent même de garder SQLite côté client et Postgres côté serveur, le meilleur des deux mondes.

## Objectifs de ce chapitre

Ce chapitre vous guidera à travers les spécificités d'intégration de SQLite avec différents langages et frameworks. Vous apprendrez à :

- Maîtriser l'API native de votre langage de prédilection
- Choisir et configurer des ORM adaptés à SQLite
- Concevoir des APIs REST performantes avec SQLite en backend
- Implémenter des stratégies de synchronisation pour les applications distribuées
- Optimiser les performances d'accès selon le contexte d'intégration

Chaque section combine théorie et exemples pratiques pour vous permettre d'implémenter des solutions robustes et performantes.

⏭️
