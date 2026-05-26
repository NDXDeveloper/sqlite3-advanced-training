🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 2 : Bases du langage SQL avec SQLite

> **Niveau** : 🟢 Débutant · **Durée estimée** : 4 à 6 heures de lecture + manipulation · **Prérequis** : Module 1 terminé

## Objectifs du module

À l'issue de ce module, vous serez capable de :

- ✅ Maîtriser les **5 classes de stockage** de SQLite (NULL, INTEGER, REAL, TEXT, BLOB) et le concept d'**affinité de type**
- ✅ Créer et gérer des bases de données SQLite : structure, métadonnées, sauvegarde, multi-bases
- ✅ Effectuer toutes les opérations **CRUD** (`INSERT`, `SELECT`, `UPDATE`, `DELETE`) avec confiance
- ✅ Implémenter les contraintes essentielles : `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`
- ✅ Construire des requêtes efficaces avec `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, `HAVING`
- ✅ Connaître les **pièges spécifiques à SQLite** : division entière, NULL, types dynamiques, sous-requêtes interdites dans CHECK, etc.

## Prérequis

- ✅ Module 1 terminé (vous savez ce qu'est SQLite et comment l'ouvrir)
- ✅ SQLite ≥ 3.38 installé et fonctionnel (`sqlite3 --version`)
- ✅ Notions de base : table, ligne, colonne (sinon, voir le glossaire du [README du module 1](/01-fondamentaux-sqlite3/README.md))
- ✅ Un terminal ouvert à côté de la lecture pour exécuter les exemples

## Plan du module

| # | Section | Sujet principal | Lien |
|--:|---------|-----------------|------|
| 2.1 | **Types de données** | NULL, INTEGER, REAL, TEXT, BLOB ; affinité ; mode STRICT | [→ 2.1](/02-bases-langage-sql-sqlite/01-types-donnees-sqlite-text-integer-real-blob-null.md) |
| 2.2 | **Création et gestion** | Création d'une base, `PRAGMA`, sauvegardes, `ATTACH` multi-bases | [→ 2.2](/02-bases-langage-sql-sqlite/02-creation-gestion-bases-donnees.md) |
| 2.3 | **CRUD** | `INSERT` (avec UPSERT), `SELECT`, `UPDATE`, `DELETE` | [→ 2.3](/02-bases-langage-sql-sqlite/03-operations-crud-create-read-update-delete.md) |
| 2.4 | **Contraintes** | `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK` (et triggers) | [→ 2.4](/02-bases-langage-sql-sqlite/04-contraintes-primary-key-foreign-key-unique-check.md) |
| 2.5 | **Requêtes de base** | `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, `HAVING`, agrégations | [→ 2.5](/02-bases-langage-sql-sqlite/05-requetes-base-select-where-order-by-group-by-having.md) |

## Pourquoi ce module est crucial ?

### 🎯 Fondations indispensables

Ce module établit les bases pratiques de tout travail sérieux avec SQLite. **90 % de votre temps avec une base de données** se passera à écrire les requêtes vues ici. Sans ces fondations, les modules suivants (optimisation, programmation avancée, intégrations) seront inaccessibles.

### 🔧 Spécificités SQLite à connaître absolument

SQLite a son propre style qui diffère parfois subtilement du SQL standard :

- **Affinité de type** : la valeur porte son type, pas la colonne (« manifest typing »)
- **`INTEGER PRIMARY KEY`** est un alias du `ROWID` interne — très rapide, et `AUTOINCREMENT` est rarement nécessaire
- **`PRAGMA foreign_keys = ON`** à activer à **chaque connexion** sinon les FK sont ignorées
- **Pas de sous-requêtes dans les `CHECK`** — utiliser des triggers à la place
- **Division `INTEGER / INTEGER = INTEGER`** (troncature, comme en C) — piège fréquent
- **NULL** se compare avec `IS NULL` / `IS NOT NULL`, jamais avec `= NULL`

Tous ces points sont expliqués en détail dans les sections du module.

## Carte mentale du module

```
                  ┌─────────────────────────────────────┐
                  │   Module 2 : Bases du langage SQL   │
                  └─────────────────┬───────────────────┘
                                    │
       ┌────────────┬───────────────┼───────────────┬────────────┐
       ▼            ▼               ▼               ▼            ▼
   2.1 Quels    2.2 Comment      2.3 Comment    2.4 Comment   2.5 Comment
   types ?      créer/gérer      ajouter,       garantir      retrouver
                la base ?        lire, modi-    l'intégrité   les bonnes
                                 fier, suppr. ? des données ? données ?
```

## Comment lire ce module

- **Manipulez en parallèle** : chaque exemple SQL est conçu pour être exécuté tel quel dans `sqlite3`
- **Ne sautez pas la section 2.1** : comprendre les types est essentiel pour la suite
- **Activez WAL et `foreign_keys`** dès le départ pour éviter les surprises :
  ```sql
  PRAGMA journal_mode = WAL;
  PRAGMA foreign_keys = ON;
  ```
- **Consultez [/Notes.md](/Notes.md)** pour un récapitulatif condensé après chaque section

## Conventions et terminologie

| Sigle | Signification |
|-------|---------------|
| **CRUD** | Create, Read, Update, Delete — les 4 opérations sur les données |
| **PK** | Primary Key (clé primaire) |
| **FK** | Foreign Key (clé étrangère) |
| **DDL** | Data Definition Language (`CREATE`, `ALTER`, `DROP`) |
| **DML** | Data Manipulation Language (`INSERT`, `UPDATE`, `DELETE`, `SELECT`) |
| **UPSERT** | INSERT … ON CONFLICT … DO UPDATE (mise à jour si conflit) |
| **CTE** | Common Table Expression (vue temporaire `WITH …`) |

## Bonnes pratiques à adopter dès maintenant

- ✅ **Toujours spécifier les colonnes** dans un `INSERT` : `INSERT INTO t (a, b) VALUES (...)`
- ✅ **Toujours utiliser `WHERE`** dans un `UPDATE` ou un `DELETE` (sinon catastrophe)
- ✅ **Préférer `IS NULL` / `IS NOT NULL`** à `= NULL`
- ✅ **Utiliser des paramètres liés** (`?`) depuis un langage hôte, jamais de concaténation de chaînes
- ✅ **Activer `PRAGMA foreign_keys = ON`** à chaque connexion
- ✅ **Stocker l'argent en centimes** (INTEGER) plutôt qu'en REAL (erreurs d'arrondi)
- ✅ **Stocker les dates** en TEXT ISO-8601 (`'2026-05-26 14:30:00'`) ou en INTEGER (timestamp Unix)

## Récapitulatif

À la fin de ce module, vous aurez les **briques techniques** pour :

- Concevoir un schéma de base relationnel propre
- Manipuler les données en toute sécurité (CRUD)
- Garantir l'intégrité avec les contraintes
- Extraire l'information avec des requêtes SQL efficaces
- Éviter les pièges spécifiques à SQLite

Les modules suivants (3 à 9) approfondiront chacun de ces aspects (normalisation, optimisation, programmation, intégrations, sécurité, projets pratiques).

---

**Prêt à pratiquer ?** Le premier chapitre explore le système de types unique de SQLite.

⏭️ [2.1 Types de données SQLite (TEXT, INTEGER, REAL, BLOB, NULL)](/02-bases-langage-sql-sqlite/01-types-donnees-sqlite-text-integer-real-blob-null.md)
