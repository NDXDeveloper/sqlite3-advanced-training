# 📚 Formation SQLite3 — du débutant au développeur avancé

> **9 modules · 47 sections · ~65 000 lignes** de cours (en français). Tous les exemples sont prêts à copier-coller et exécuter sur SQLite ≥ 3.38.

## Parcours d'apprentissage suggéré

| Niveau | Modules conseillés | Objectif |
|---|---|---|
| 🟢 **Débutant** | 1 · 2 | Démarrer : installation, SQL de base, CRUD |
| 🟡 **Intermédiaire** | 3 · 4 | Concevoir : modélisation, requêtes avancées |
| 🟠 **Avancé** | 5 · 6 · 7 | Optimiser et intégrer : performance, transactions, ORM/APIs |
| 🔴 **Expert / production** | 8 · 9 | Sécuriser et déployer : chiffrement, audit, tests, cas réels |

> 💡 La fiche [Notes.md](/Notes.md) regroupe tous les concepts, pièges et syntaxes critiques sur une seule page — utile en révision ou en référence rapide.

---

## Sommaire détaillé

### 🟢 [1. Fondamentaux de SQLite3](/01-fondamentaux-sqlite3/README.md)
*Architecture serverless, installation, différences avec MySQL/PostgreSQL, limites.*

- [1.1 : Introduction à SQLite : caractéristiques et cas d'usage](/01-fondamentaux-sqlite3/01-introduction-sqlite-caracteristiques-cas-usage.md)
- [1.2 : Installation et configuration des outils](/01-fondamentaux-sqlite3/02-installation-configuration-outils.md)
- [1.3 : Différences avec les autres SGBD (MySQL, PostgreSQL)](/01-fondamentaux-sqlite3/03-differences-autres-sgbd-mysql-postgresql.md)
- [1.4 : Architecture serverless et fichier de base unique](/01-fondamentaux-sqlite3/04-architecture-serverless-fichier-base-unique.md)
- [1.5 : Limitations et contraintes de SQLite](/01-fondamentaux-sqlite3/05-limitations-contraintes-sqlite.md)

### 🟢 [2. Bases du langage SQL avec SQLite](/02-bases-langage-sql-sqlite/README.md)
*Types dynamiques, CRUD, contraintes, requêtes SELECT/WHERE/GROUP BY.*

- [2.1 : Types de données SQLite (TEXT, INTEGER, REAL, BLOB, NULL)](/02-bases-langage-sql-sqlite/01-types-donnees-sqlite-text-integer-real-blob-null.md)
- [2.2 : Création et gestion des bases de données](/02-bases-langage-sql-sqlite/02-creation-gestion-bases-donnees.md)
- [2.3 : Opérations CRUD : CREATE, READ, UPDATE, DELETE](/02-bases-langage-sql-sqlite/03-operations-crud-create-read-update-delete.md)
- [2.4 : Contraintes : PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK](/02-bases-langage-sql-sqlite/04-contraintes-primary-key-foreign-key-unique-check.md)
- [2.5 : Requêtes de base : SELECT, WHERE, ORDER BY, GROUP BY, HAVING](/02-bases-langage-sql-sqlite/05-requetes-base-select-where-order-by-group-by-having.md)

### 🟡 [3. Conception et modélisation avancée](/03-conception-modelisation-avancee/README.md)
*Normalisation, jointures, clés étrangères différées, triggers, vues.*

- [3.1 : Normalisation des données (1NF, 2NF, 3NF)](/03-conception-modelisation-avancee/01-normalisation-donnees-1nf-2nf-3nf.md)
- [3.2 : Relations entre tables et jointures complexes](/03-conception-modelisation-avancee/02-relations-tables-jointures-complexes.md)
- [3.3 : Gestion des clés étrangères et contraintes référentielles](/03-conception-modelisation-avancee/03-gestion-cles-etrangeres-contraintes-referentielles.md)
- [3.4 : Triggers : création, types et cas d'usage](/03-conception-modelisation-avancee/04-triggers-creation-types-cas-usage.md)
- [3.5 : Vues : création, utilisation et maintenance](/03-conception-modelisation-avancee/05-vues-creation-utilisation-maintenance.md)

### 🟡 [4. Requêtes avancées et optimisation](/04-requetes-avancees-optimisation/README.md)
*Sous-requêtes, CTE récursives, window functions, REGEXP, CASE, JSON/JSONB.*

- [4.1 : Sous-requêtes corrélées et non-corrélées](/04-requetes-avancees-optimisation/01-sous-requetes-correlees-non-correlees.md)
- [4.2 : Expressions de table communes (CTE) et requêtes récursives](/04-requetes-avancees-optimisation/02-expressions-table-communes-cte-requetes-recursives.md)
- [4.3 : Fonctions de fenêtrage (WINDOW functions)](/04-requetes-avancees-optimisation/03-fonctions-fenetrage-window-functions.md)
- [4.4 : Expressions régulières avec REGEXP](/04-requetes-avancees-optimisation/04-expressions-regulieres-regexp.md)
- [4.5 : Requêtes complexes avec CASE, COALESCE, NULLIF](/04-requetes-avancees-optimisation/05-requetes-complexes-case-coalesce-nullif.md)
- [4.6 : Interrogation et manipulation de données JSON](/04-requetes-avancees-optimisation/06-interrogation-manipulation-donnees-json.md)

### 🟠 [5. Optimisation des performances](/05-optimisation-performances/README.md)
*Planificateur, index (simple, composite, partiel, COLLATE), EXPLAIN QUERY PLAN, PRAGMA.*

- [5.1 : Comprendre le planificateur de requêtes SQLite](/05-optimisation-performances/01-comprendre-planificateur-requetes-sqlite.md)
- [5.2 : Création et gestion des index](/05-optimisation-performances/02-creation-gestion-index.md)
- [5.3 : Analyse des plans d'exécution avec EXPLAIN QUERY PLAN](/05-optimisation-performances/03-analyse-plans-execution-explain-query-plan.md)
- [5.4 : Optimisation des requêtes lentes](/05-optimisation-performances/04-optimisation-requetes-lentes.md)
- [5.5 : Configuration des paramètres SQLite (PRAGMA)](/05-optimisation-performances/05-configuration-parametres-sqlite-pragma.md)

### 🟠 [6. Programmation avancée avec SQLite](/06-programmation-avancee-sqlite/README.md)
*UDF Python, extensions, transactions (DEFERRED/IMMEDIATE), backup API, FTS5.*

- [6.1 : Fonctions définies par l'utilisateur (UDF)](/06-programmation-avancee-sqlite/01-fonctions-definies-utilisateur-udf.md)
- [6.2 : Extensions SQLite et modules chargeables](/06-programmation-avancee-sqlite/02-extensions-sqlite-modules-chargeables.md)
- [6.3 : Gestion des transactions et niveaux d'isolation](/06-programmation-avancee-sqlite/03-gestion-transactions-niveaux-isolation.md)
- [6.4 : Sauvegarde et restauration (backup API)](/06-programmation-avancee-sqlite/04-sauvegarde-restauration-backup-api.md)
- [6.5 : Gestion des erreurs et exceptions](/06-programmation-avancee-sqlite/05-gestion-erreurs-exceptions.md)
- [6.6 : Implémenter la recherche plein texte avec FTS5](/06-programmation-avancee-sqlite/06-recherche-plein-texte-avec-fts5.md)

### 🟠 [7. Intégration et APIs](/07-integration-apis/README.md)
*Bindings Python/C/Java/JavaScript, ORMs modernes (SQLModel/Drizzle), API REST, synchronisation.*

- [7.1 : Utilisation de SQLite avec Python (sqlite3)](/07-integration-apis/01-utilisation-sqlite-python-sqlite3.md)
- [7.2 : Intégration avec d'autres langages (C, Java, JavaScript)](/07-integration-apis/02-integration-autres-langages-c-java-javascript.md)
- [7.3 : ORM et SQLite : bonnes pratiques](/07-integration-apis/03-orm-sqlite-bonnes-pratiques.md)
- [7.4 : APIs REST avec SQLite en backend](/07-integration-apis/04-apis-rest-sqlite-backend.md)
- [7.5 : Synchronisation et réplication de données](/07-integration-apis/05-synchronisation-replication-donnees.md)

### 🔴 [8. Sécurité et administration](/08-securite-administration/README.md)
*SQLCipher, RBAC/ABAC, audit par triggers, sauvegardes Litestream, monitoring.*

- [8.1 : Chiffrement des bases de données SQLite](/08-securite-administration/01-chiffrement-bases-donnees-sqlite.md)
- [8.2 : Gestion des permissions et contrôle d'accès](/08-securite-administration/02-gestion-permissions-controle-acces.md)
- [8.3 : Audit et logging des opérations](/08-securite-administration/03-audit-logging-operations.md)
- [8.4 : Sauvegardes automatisées et stratégies de récupération](/08-securite-administration/04-sauvegardes-automatisees-strategies-recuperation.md)
- [8.5 : Monitoring et maintenance préventive](/08-securite-administration/05-monitoring-maintenance-preventive.md)

### 🔴 [9. Cas d'usage avancés et projets pratiques](/09-cas-usage-avances-projets-pratiques/README.md)
*Mobile (Room/GRDB), analyse (DuckDB/Polars), migration multi-format, projet web Flask, tests pytest.*

- [9.1 : SQLite pour les applications mobiles](/09-cas-usage-avances-projets-pratiques/01-sqlite-applications-mobiles.md)
- [9.2 : Analyse de données avec SQLite](/09-cas-usage-avances-projets-pratiques/02-analyse-donnees-sqlite.md)
- [9.3 : Migration de données entre différents formats](/09-cas-usage-avances-projets-pratiques/03-migration-donnees-differents-formats.md)
- [9.4 : Projet complet : création d'une application web avec SQLite](/09-cas-usage-avances-projets-pratiques/04-projet-complet-creation-application-web-sqlite.md)
- [9.5 : Tests unitaires et d'intégration avec SQLite](/09-cas-usage-avances-projets-pratiques/05-tests-unitaires-integration-sqlite.md)

---

## Ressources complémentaires

- 📋 [**Notes.md** — Fiche de révision condensée](/Notes.md) : tous les concepts, pièges et syntaxes critiques sur une seule page (cheat-sheet)
- 🏠 [**README.md** — Page d'accueil de la formation](/README.md)
- 🔗 [Documentation officielle SQLite](https://www.sqlite.org/docs.html) · [Litestream](https://litestream.io/) · [SQLCipher](https://www.zetetic.net/sqlcipher/)
