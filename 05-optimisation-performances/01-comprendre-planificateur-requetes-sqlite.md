🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.1 Comprendre le planificateur de requêtes SQLite

## Introduction au planificateur de requêtes

Le **planificateur de requêtes** (Query Planner) est le cerveau de SQLite. Quand vous écrivez une requête SQL, le planificateur décide de la meilleure façon de l'exécuter. C'est comme un GPS qui calcule le meilleur itinéraire pour aller d'un point A à un point B.

### Qu'est-ce que le planificateur de requêtes ?

Imaginez que vous cherchez un livre dans une bibliothèque :
- **Sans planificateur :** Vous regardez livre par livre jusqu'à trouver le bon
- **Avec planificateur :** Vous utilisez le catalogue, allez directement au bon rayon, puis à la bonne étagère

Le planificateur SQLite fait exactement cela avec vos données : il trouve le chemin le plus rapide pour récupérer les informations demandées.

## Comment fonctionne le planificateur ?

### Étapes du processus

1. **Analyse de la requête** : SQLite comprend ce que vous demandez
2. **Génération d'options** : Il imagine plusieurs façons de répondre à votre question
3. **Estimation des coûts** : Il calcule combien de temps prendrait chaque option
4. **Sélection du meilleur plan** : Il choisit la méthode la plus rapide

### Visualisation : SCAN vs SEARCH

Pour comprendre **pourquoi** un index accélère les recherches, voici la structure interne de SQLite côté table (séquentielle, triée par `rowid`) et côté index (B-tree trié par valeur).

```
TABLE produits (1 000 000 lignes, lecture séquentielle par rowid)

   rowid │ nom          │ prix   │ categorie
  ───────┼──────────────┼────────┼────────────
      1  │ produit_1    │ 423.7  │ vetement
      2  │ produit_2    │ 12.0   │ alimentaire
      3  │ produit_3    │ 891.2  │ electronique     ← cherche !
      4  │ produit_4    │ 156.8  │ meuble
       ...                          ↑
      N  │ produit_N    │  ...   │  ...        SCAN : lit TOUT, ligne par ligne


INDEX idx_categorie ON produits(categorie)
                                    │
              Structure B-tree triée par 'categorie'
                                    │
                          ┌─────────┴─────────┐
                          │   meuble (root)   │
                          └─────────┬─────────┘
                                    │
                ┌───────────────────┼───────────────────┐
                ▼                   ▼                   ▼
        ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
        │ alimentaire   │   │ electronique  │   │ vetement      │
        │  ↓ rowid=2    │   │  ↓ rowid=3    │   │  ↓ rowid=1    │
        │  ↓ rowid=7    │   │  ↓ rowid=12   │   │  ↓ rowid=5    │
        │  ...          │   │  ...          │   │  ...          │
        └───────────────┘   └───────────────┘   └───────────────┘

  SEARCH 'electronique' : descend l'arbre en log₂(N) sauts (≈ 20 sauts
  pour 1 M lignes), puis lit directement les rowids correspondants.
  → 100× à 1000× plus rapide qu'un SCAN sur grande table.
```

**Pourquoi log₂(N) ?** Un B-tree est un arbre **équilibré** : à chaque niveau on divise par 2 (ou par la « fan-out » du nœud, typiquement quelques dizaines à quelques centaines en SQLite). Donc même pour 1 milliard de lignes, on atteint la bonne entrée en ≈ 30 sauts, contre 1 milliard d'opérations pour un SCAN.

### Exemple simple

Prenons cette requête basique :
```sql
SELECT nom, age FROM employes WHERE age > 30;
```

Le planificateur peut choisir entre :
- **Option 1** : Regarder tous les employés un par un (scan complet)
- **Option 2** : Utiliser un index sur la colonne `age` (s'il existe)

## Découvrir le plan d'exécution

### La commande EXPLAIN QUERY PLAN

Pour voir ce que fait le planificateur, utilisez `EXPLAIN QUERY PLAN` :

```sql
EXPLAIN QUERY PLAN  
SELECT nom, age FROM employes WHERE age > 30;  
```

**Résultat possible :**
```
SCAN employes
```

Cela signifie que SQLite va examiner chaque ligne de la table `employes`.

### Créons un exemple pratique

**1. Créer une table de test :**
```sql
CREATE TABLE employes (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    age INTEGER,
    departement TEXT
);

-- Insérer quelques données
INSERT INTO employes VALUES
(1, 'Alice', 28, 'IT'),
(2, 'Bob', 35, 'RH'),
(3, 'Charlie', 42, 'Finance'),
(4, 'Diana', 29, 'IT'),
(5, 'Eve', 38, 'Marketing');
```

**2. Tester différentes requêtes :**

```sql
-- Requête sans index
EXPLAIN QUERY PLAN  
SELECT * FROM employes WHERE age > 30;  
```

**Résultat :**
```
SCAN employes
```

## Types de plans d'exécution

### 1. SCAN (Balayage complet)

**Ce que c'est :** SQLite lit chaque ligne de la table
```sql
EXPLAIN QUERY PLAN  
SELECT * FROM employes WHERE departement = 'IT';  
```
**Résultat :** `SCAN employes`

**Quand c'est utilisé :**
- Pas d'index disponible
- La requête concerne la majorité des données
- Table très petite

### 2. SEARCH avec index

**Ce que c'est :** SQLite utilise un index pour aller directement aux bonnes données

```sql
-- Créer un index
CREATE INDEX idx_age ON employes(age);

-- Même requête qu'avant
EXPLAIN QUERY PLAN  
SELECT * FROM employes WHERE age > 30;  
```

**Résultat :** `SEARCH employes USING INDEX idx_age (age>?)`

**Avantage :** Beaucoup plus rapide sur les grandes tables !

### 3. Opérations avec jointures

```sql
-- Créer une deuxième table
CREATE TABLE departements (
    nom TEXT PRIMARY KEY,
    budget INTEGER
);

INSERT INTO departements VALUES
('IT', 100000),
('RH', 50000),
('Finance', 75000);

-- Requête avec jointure
EXPLAIN QUERY PLAN  
SELECT e.nom, d.budget  
FROM employes e  
JOIN departements d ON e.departement = d.nom;  
```

**Résultat possible :**
```
|--SCAN e
`--SEARCH d USING INDEX sqlite_autoindex_departements_1 (nom=?)
```

> 💡 Le nom `sqlite_autoindex_departements_1` apparaît parce que `departements(nom)` est une `PRIMARY KEY` **non-INTEGER** : SQLite crée alors un **index automatique** pour la PK. Avec une `INTEGER PRIMARY KEY`, on verrait à la place `USING INTEGER PRIMARY KEY` (encore plus rapide — accès direct au ROWID, sans index séparé).

## Comment interpréter les plans d'exécution

### Vocabulaire important

- **SCAN** : Lecture complète = lent sur grandes tables
- **SEARCH** : Utilisation d'index = rapide
- **USING INDEX** : Un index est utilisé = bon signe !
- **PRIMARY KEY** : Utilise la clé primaire = très efficace

### Exemple d'interprétation

```sql
EXPLAIN QUERY PLAN  
SELECT nom FROM employes WHERE age BETWEEN 25 AND 35 ORDER BY nom;  
```

**Sans index sur age :**
```
SCAN employes  
USE TEMP B-TREE FOR ORDER BY  
```

**Interprétation :**
1. SQLite lit toute la table (`SCAN`)
2. Il crée une structure temporaire pour trier (`TEMP B-TREE`)
3. **Problème :** Double travail = lent !

**Avec index sur age :**
```sql
-- (`IF NOT EXISTS` car nous avons déjà créé `idx_age` plus haut dans le chapitre)
CREATE INDEX IF NOT EXISTS idx_age ON employes(age);
```

**Nouveau plan :**
```
SEARCH employes USING INDEX idx_age (age>? AND age<?)  
USE TEMP B-TREE FOR ORDER BY  
```

**Amélioration :** Moins de lignes à trier grâce au filtre par index !

## Facteurs influençant le planificateur

### 1. Statistiques des tables

SQLite maintient des statistiques sur vos tables pour aider le planificateur à choisir entre `SCAN` et `SEARCH`. Elles ne sont **pas calculées automatiquement** : il faut lancer `ANALYZE`.

```sql
-- ⚠️ `PRAGMA table_info(employes)` retourne la STRUCTURE (colonnes, types),
--    PAS les statistiques. Pour les vraies stats, c'est :
ANALYZE employes;                       -- (re)calcule les stats  
SELECT * FROM sqlite_stat1              -- stats par table/index :  
WHERE tbl = 'employes';                 -- (nb_lignes, sélectivité…)  

-- Optionnel : ANALYZE sans argument met à jour TOUTES les tables
ANALYZE;
```

**Important :** Des statistiques à jour = meilleurs plans ! À relancer après de gros `INSERT`/`DELETE` (ne pas s'inquiéter : `ANALYZE` est rapide même sur de grandes bases).

### Lire et comprendre `sqlite_stat1`

Après `ANALYZE`, SQLite peuple une table système nommée **`sqlite_stat1`** que le planificateur consulte à chaque requête. Comprendre son contenu permet de **diagnostiquer pourquoi un plan choisi semble suboptimal**.

**Format des stats :**
```sql
ANALYZE;  
SELECT * FROM sqlite_stat1;  
-- Exemple de sortie (table produits 1 M lignes, 5 catégories, ~10 000 prix distincts) :
-- produits | idx_categorie  | 1000000 200000
-- produits | idx_prix       | 1000000 100
-- produits | idx_cat_prix   | 1000000 200000 20
```

**Décodage de la colonne `stat`** (format : `"N k1 k2 k3 ..."`) :
- **`N`** : nombre total de lignes dans l'index.
- **`k1`** : nombre **moyen** de lignes ayant la même valeur sur la **1ʳᵉ colonne** d'index (= 1 ⁄ sélectivité).
- **`k2`** : nombre moyen de lignes ayant la même paire (col1, col2).
- **`k3`, `k4`...** : idem pour les colonnes suivantes.

```
sqlite_stat1 décodé pour idx_cat_prix(categorie, prix) :

   "1000000  200000     20"
       │       │         │
       │       │         └─► 20 lignes en moyenne par (categorie, prix)
       │       │             → si on filtre les deux, on lit 20 lignes
       │       │
       │       └─► 200 000 lignes par catégorie (1 M ÷ 5 catégories)
       │           → si on filtre par categorie seule, 200k lignes lues
       │
       └─► 1 000 000 lignes au total
```

**Pourquoi c'est utile :**
- Si `k1` est **proche de N**, l'index a une **faible sélectivité** (peu de valeurs distinctes) → le planificateur pourrait préférer un SCAN.
- Si `k1` est **petit** (1 à quelques dizaines), l'index est **très sélectif** → SEARCH est presque toujours gagnant.
- Décide si un index composite mérite d'exister : si `k1 ≈ k2`, ajouter la 2ᵉ colonne n'apporte rien.

### `sqlite_stat4` : stats par valeur (statistiques fines)

`sqlite_stat1` ne stocke que des **moyennes**. Si une catégorie contient 800 000 lignes et les 4 autres 50 000 chacune, `sqlite_stat1` rapporte « 200 000 lignes par catégorie » — moyenne trompeuse qui peut conduire à un mauvais plan pour la grosse catégorie.

**`sqlite_stat4`** (présent uniquement si SQLite est compilé avec **`SQLITE_ENABLE_STAT4`**) stocke des échantillons concrets de valeurs avec leur fréquence réelle, permettant des plans **différents selon la valeur** passée en paramètre.

```sql
-- Vérifier si sqlite_stat4 est disponible
PRAGMA compile_options;
-- Chercher ENABLE_STAT4 dans la liste ; si absent, sqlite_stat4 n'est pas créé.

-- Si présent, après ANALYZE :
SELECT tbl, idx, neq, nlt, ndlt, sample  
FROM sqlite_stat4  
WHERE tbl = 'produits'  
LIMIT 5;  
-- neq    = nombre de lignes égales à 'sample'
-- nlt    = nombre de lignes strictement inférieures
-- ndlt   = nombre de valeurs distinctes strictement inférieures
-- sample = échantillon de valeur concrète (blob ou texte)
```

**Quand `sqlite_stat4` change la donne :**
```sql
-- Avec sqlite_stat1 seul :
--   "categorie a 200 000 lignes en moyenne par valeur"
--   → SQLite choisit le même plan pour categorie='electronique' et categorie='rare'

-- Avec sqlite_stat4 :
--   "categorie='electronique' = 800 000 lignes (SCAN gagnant)"
--   "categorie='rare'         =  10 000 lignes (SEARCH gagnant)"
--   → Plans différents selon la valeur ! 🎯
```

> ℹ️ **Activation de `SQLITE_ENABLE_STAT4`** : les **binaires officiels** distribués par sqlite.org (et la plupart des paquets Linux) **n'ont pas STAT4 activé** par défaut. Pour l'activer, il faut recompiler SQLite ou utiliser un binaire spécifique. Le coût en taille de base est modéré (quelques % de la taille des index). En général, `sqlite_stat1` suffit largement pour 95 % des cas.

> ⚠️ **`analysis_limit` (SQLite 3.32+)** : sur de très grosses bases, `ANALYZE` peut être long. `PRAGMA analysis_limit = 1000;` limite le nombre d'échantillons consultés par index — plus rapide, légèrement moins précis. Recommandé en production pour les bases >1 Go.

### 2. Taille des données

```sql
-- Petite table : SCAN peut être plus rapide
CREATE TABLE petite_table (id INTEGER, nom TEXT);
-- 10 lignes → SCAN sera probablement choisi

-- Grande table : INDEX devient essentiel
CREATE TABLE grande_table (id INTEGER, nom TEXT);
-- 100 000 lignes → INDEX indispensable
```

### 3. Sélectivité des conditions

**Condition très sélective :** (peu de résultats — souvent 1)
```sql
SELECT * FROM employes WHERE id = 123;
-- → Plan : SEARCH employes USING INTEGER PRIMARY KEY (rowid=?)
-- (accès direct au rowid, encore plus rapide qu'un index classique)
```

**Condition peu sélective :** (beaucoup de résultats)
```sql
SELECT * FROM employes WHERE age IS NOT NULL;
-- → SCAN peut être plus rapide même avec un index, car parcourir l'index
--   PUIS la table pour chaque ligne coûte plus cher qu'un seul SCAN.
```

## Exemples pratiques d'optimisation

### Problème : Requête lente

```text
-- Table illustrative avec 100 000 employés (à générer selon vos besoins)
CREATE TABLE employes_large AS SELECT … FROM … ;   -- placeholder

-- Requête lente sur cette grosse table
SELECT nom, salaire  
FROM employes_large  
WHERE departement = 'IT' AND age > 30;  
```

**Plan actuel :**
```sql
EXPLAIN QUERY PLAN  
SELECT nom, salaire FROM employes_large  
WHERE departement = 'IT' AND age > 30;  
```
**Résultat :** `SCAN employes_large`

### Solution : Ajouter des index

**Option 1 : Index simple**
```sql
CREATE INDEX idx_dept ON employes_large(departement);
```

**Option 2 : Index composite (recommandé)**
```sql
CREATE INDEX idx_dept_age ON employes_large(departement, age);
```

**Nouveau plan :**
```
SEARCH employes_large USING INDEX idx_dept_age (departement=? AND age>?)
```

**Résultat :** Requête 10x à 100x plus rapide !

## Cas particuliers et pièges à éviter

### 1. Les fonctions dans WHERE

**Mauvais :**
```sql
SELECT * FROM employes WHERE UPPER(nom) = 'ALICE';
-- Plan : SCAN employes
-- ⚠️ Même si un index `idx_nom ON employes(nom)` existait, il serait inutilisable :
--    la fonction UPPER() transforme la colonne et bloque l'usage de l'index.
```

**Bon — deux solutions :**
```sql
-- Solution A : INDEX sur l'expression UPPER(nom)
CREATE INDEX idx_nom_upper ON employes(UPPER(nom));  
SELECT * FROM employes WHERE UPPER(nom) = 'ALICE';  
-- Plan : SEARCH employes USING INDEX idx_nom_upper

-- Solution B : COLLATE NOCASE (insensible à la casse) — index DÉDIÉ requis
CREATE INDEX idx_nom_nocase ON employes(nom COLLATE NOCASE);  
SELECT * FROM employes WHERE nom = 'Alice' COLLATE NOCASE;  
-- Plan : SEARCH employes USING INDEX idx_nom_nocase (nom=?)
```

> ⚠️ **Piège du COLLATE** : un index « normal » sur `nom` **n'est PAS utilisé** par une requête avec `COLLATE NOCASE` (et inversement). Les collations doivent correspondre exactement entre l'index et la requête, sinon SQLite fait un `SCAN`.

### 2. `OR` et l'optimisation `MULTI-INDEX OR`

Quand un `OR` porte sur des **colonnes indexées** (ou la même colonne avec un index), SQLite sait depuis longtemps faire une **optimisation `MULTI-INDEX OR`** : il utilise chaque index séparément, puis fusionne les résultats. Pas besoin de réécrire en `UNION` :

```sql
EXPLAIN QUERY PLAN  
SELECT * FROM employes WHERE age < 25 OR age > 65;  
-- Plan attendu (si idx_age existe) :
--   MULTI-INDEX OR
--   ├── SEARCH employes USING INDEX idx_age (age<?)
--   └── SEARCH employes USING INDEX idx_age (age>?)
```

> ⚠️ **Quand `OR` reste lent** : si **au moins une** des branches du `OR` n'a pas d'index utilisable, SQLite retombe sur un `SCAN` complet :  
>
> ```sql
> -- Pas d'index sur 'commentaire' → SCAN forcé
> SELECT * FROM employes WHERE age < 25 OR commentaire LIKE '%urgent%';
> ```
>  
> Dans ce cas, réécrire en `UNION ALL` (et indexer chaque branche) **peut** aider — mais attention : `UNION` (sans `ALL`) introduit un `TEMP B-TREE` pour dédupliquer, ce qui peut coûter plus cher que le `SCAN` qu'on essayait d'éviter. Toujours mesurer avec `EXPLAIN QUERY PLAN` avant de réécrire.

### 3. ORDER BY et LIMIT

**Problème :**
```sql
SELECT * FROM employes ORDER BY nom LIMIT 10;
-- Sans index sur nom : trie TOUT puis prend 10
```

**Solution :**
```sql
CREATE INDEX idx_nom ON employes(nom);
-- Maintenant : prend directement les 10 premiers
```

## Exercices pratiques

### Exercice 1 : Analyser vos requêtes

1. Créez cette table :
```sql
CREATE TABLE produits (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    prix REAL,
    categorie TEXT,
    stock INTEGER
);
```

2. Insérez 1000 produits (utilisez une boucle ou un script)

3. Analysez ces requêtes :
```sql
EXPLAIN QUERY PLAN SELECT * FROM produits WHERE prix > 100;  
EXPLAIN QUERY PLAN SELECT * FROM produits WHERE categorie = 'electronique';  
EXPLAIN QUERY PLAN SELECT * FROM produits ORDER BY prix DESC LIMIT 5;  
```

### Exercice 2 : Optimiser avec des index

1. Créez les index appropriés pour les requêtes ci-dessus
2. Comparez les nouveaux plans d'exécution
3. Mesurez la différence de performance avec `.timer on`

## Conseils pour débuter

### 1. Commencez simple
- Utilisez `EXPLAIN QUERY PLAN` sur vos requêtes lentes
- Cherchez les mots `SCAN` sur les grandes tables
- Ajoutez des index sur les colonnes de vos `WHERE`

### 2. Mesurez l'impact
```text
.timer on
-- Votre requête avant optimisation
SELECT … ;

-- Créer l'index approprié
CREATE INDEX … ;

-- Même requête après optimisation
SELECT … ;
```

### 3. N'optimisez pas prématurément
- Concentrez-vous d'abord sur les requêtes les plus utilisées
- Un index inutile ralentit les écritures
- Gardez vos requêtes simples quand c'est possible

## Résumé

Le planificateur de requêtes SQLite :

✅ **Choisit automatiquement** la meilleure façon d'exécuter vos requêtes  
✅ **Utilise les index** disponibles pour accélérer les recherches  
✅ **Peut être analysé** avec `EXPLAIN QUERY PLAN`  
✅ **S'améliore** avec des statistiques à jour (`ANALYZE`)

**Points clés à retenir :**
- `SCAN` = lent sur grandes tables
- `SEARCH USING INDEX` = rapide
- Créez des index sur vos colonnes de recherche fréquente
- Testez toujours l'impact de vos optimisations

Dans la section suivante, nous verrons comment créer et gérer efficacement ces fameux index qui rendent vos requêtes si rapides !

⏭️ [5.2 Création et gestion des index](/05-optimisation-performances/02-creation-gestion-index.md)
