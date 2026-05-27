🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.2 Création et gestion des index

## Qu'est-ce qu'un index ?

Un **index** en base de données, c'est comme l'index d'un livre ou l'annuaire téléphonique : une structure qui permet de trouver rapidement une information sans avoir à tout parcourir.

### Analogie avec un dictionnaire

**Sans index (scan complet) :**
- Pour trouver le mot "zèbre", vous lisez page par page depuis le début
- Très lent sur un gros dictionnaire !

**Avec index :**
- Vous ouvrez directement aux pages commençant par "Z"
- Puis vous cherchez "zè", puis "zèb"
- Beaucoup plus rapide !

### Dans SQLite

```sql
-- Sans index : SQLite lit TOUS les employés
SELECT * FROM employes WHERE nom = 'Alice';

-- Avec index sur nom : SQLite va directement aux noms commençant par 'A'
CREATE INDEX idx_nom ON employes(nom);  
SELECT * FROM employes WHERE nom = 'Alice';  -- Beaucoup plus rapide !  
```

## Types d'index dans SQLite

### 1. Index automatiques (Primary Key et UNIQUE)

SQLite crée automatiquement des index pour les contraintes `PRIMARY KEY` (sauf `INTEGER PRIMARY KEY`) et `UNIQUE` :

```sql
-- Table de référence utilisée dans tous les exemples de ce chapitre
CREATE TABLE employes (
    id          INTEGER PRIMARY KEY,  -- ⚠️ PAS d'index créé : cette colonne EST le ROWID
                                      --    (la table elle-même est triée par cette clé)
    nom         TEXT,
    email       TEXT UNIQUE,          -- ✅ Index automatique : sqlite_autoindex_employes_1
    age         INTEGER,               -- utilisé pour les index simples plus loin
    departement TEXT,                  -- utilisé pour les index composites
    salaire     REAL,                  -- utilisé pour les index DESC
    statut      TEXT                   -- utilisé pour les index partiels
);
```

> 💡 **Pourquoi pas d'index pour `INTEGER PRIMARY KEY` ?** En SQLite, `INTEGER PRIMARY KEY` est un **alias direct du `rowid`** (l'identifiant interne de chaque ligne). La table est physiquement stockée triée par `rowid`, donc un lookup `WHERE id = 5` est déjà ultra-rapide sans index séparé — `EXPLAIN QUERY PLAN` affiche `SEARCH USING INTEGER PRIMARY KEY (rowid=?)`. C'est **plus efficace** qu'un index classique. Toute autre `PRIMARY KEY` (non-INTEGER) crée un index automatique nommé `sqlite_autoindex_<table>_<n>`.

**Vérification :**
```sql
-- Voir tous les index d'une table
PRAGMA index_list(employes);
-- Sortie : seulement sqlite_autoindex_employes_1 (pour email UNIQUE),
--          AUCUNE entrée pour la PRIMARY KEY `id`.
```

### 2. Index simples (une seule colonne)

```sql
-- Créer un index sur la colonne 'age'
CREATE INDEX idx_age ON employes(age);

-- Maintenant cette requête sera rapide :
SELECT * FROM employes WHERE age = 30;
```

### 3. Index composites (plusieurs colonnes)

```sql
-- Index sur plusieurs colonnes
CREATE INDEX idx_dept_age ON employes(departement, age);

-- Optimise ces requêtes :
SELECT * FROM employes WHERE departement = 'IT' AND age > 25;  
SELECT * FROM employes WHERE departement = 'IT';  -- Premier critère seulement  
```

**Important :** L'ordre des colonnes compte ! L'index ci-dessus optimise :
- ✅ `departement = 'IT' AND age > 25`
- ✅ `departement = 'IT'` (utilise juste la première colonne)
- ❌ `age > 25` (ne peut pas utiliser l'index efficacement)

### Visualisation : pourquoi l'ordre des colonnes compte

Un index composite est trié **lexicographiquement** : d'abord sur la 1ʳᵉ colonne, puis sur la 2ᵉ **à l'intérieur de chaque valeur** de la 1ʳᵉ.

```
INDEX idx_dept_age ON employes(departement, age)

     departement │  age  │ rowid
    ─────────────┼───────┼────────
       Finance   │  31   │   42      ┐
       Finance   │  45   │   17      │  Bloc 'Finance'
       Finance   │  52   │    9      │  (trié par age)
       Finance   │  61   │   23      ┘
       IT        │  25   │   55      ┐
       IT        │  28   │    7      │  Bloc 'IT'
       IT        │  30   │   14      │  (trié par age)
       IT        │  38   │    2      │
       IT        │  42   │   33      ┘
       Marketing │  29   │   48      ┐  Bloc 'Marketing'
       Marketing │  35   │   19      ┘  (trié par age)


  Requête : WHERE departement = 'IT' AND age > 30
            └────────┬────────┘    └─────┬─────┘
            saut direct au bloc 'IT'    parcours dans le bloc
            (égalité = clé 1)            (inégalité = clé 2)

  → SEARCH employes USING INDEX idx_dept_age (departement=? AND age>?)


  Requête : WHERE age > 30 SEULE
                  └──────┘
                  Sans la 1ʳᵉ clé, impossible de sauter directement.
                  Il faudrait visiter TOUS les blocs ('Finance', 'IT',
                  'Marketing'...) puis filtrer chacun par age.
                  → SCAN employes (l'index est inutilisable ici)
```

**Règle mnémotechnique** : « **égalité avant inégalité** ». La (ou les) colonne(s) avec `=` ou `IN (...)` doivent venir en tête de l'index. La colonne avec `>`, `<`, `BETWEEN` peut suivre, mais aucune colonne après elle ne servira au filtrage par l'index (juste au tri éventuel).

## Créer des index efficaces

### Syntaxe de base

```text
-- Syntaxe générale (template, pas exécutable tel quel)
CREATE INDEX nom_de_l_index ON nom_table(colonne1, colonne2, ...);
```

```sql
-- Exemples pratiques (utilisent la table `employes` créée plus haut)
CREATE INDEX idx_nom ON employes(nom);  
CREATE INDEX idx_salaire ON employes(salaire);  
CREATE INDEX idx_dept_salaire ON employes(departement, salaire);  
```

### Index avec conditions (Index partiels)

```sql
-- Index seulement sur les employés actifs
CREATE INDEX idx_employes_actifs  
ON employes(nom)  
WHERE statut = 'actif';  

-- ✅ Cette requête utilise l'index partiel (la condition WHERE = condition de l'index)
SELECT * FROM employes WHERE nom = 'Alice' AND statut = 'actif';

-- ❌ Celle-ci NE l'utilise PAS — l'index ne contient que les lignes 'actif'
SELECT * FROM employes WHERE nom = 'Alice';                          -- SCAN  
SELECT * FROM employes WHERE nom = 'Alice' AND statut = 'inactif';   -- SCAN  
```

**Avantage :** Index plus petit et plus rapide !

> 💡 **Règle d'or des index partiels** : la requête doit contenir **la même condition** (ou une condition plus restrictive) que celle de l'index pour que SQLite décide de l'utiliser. Sans cela, il préfère un `SCAN` complet — vérifier avec `EXPLAIN QUERY PLAN`.

### Index sur expressions

```sql
-- Index sur une expression calculée
CREATE INDEX idx_nom_upper ON employes(UPPER(nom));

-- Optimise cette recherche insensible à la casse :
SELECT * FROM employes WHERE UPPER(nom) = 'ALICE';
```

### Index sur expressions JSON (lien avec module 4)

Lorsque vous stockez du JSON dans une colonne TEXT (très courant pour les configurations, métadonnées, événements…), une recherche `WHERE json_extract(payload, '$.cle') = ?` provoque un `SCAN` complet par défaut — la fonction empêche l'utilisation d'un index classique.

**Solution : index sur l'expression `json_extract` elle-même.**

```sql
-- Table d'événements avec payload JSON
CREATE TABLE evenements (
    id      INTEGER PRIMARY KEY,
    payload TEXT      -- contient du JSON : {"type":"clic","user":"alice","ts":1700000000}
);

INSERT INTO evenements (payload) VALUES
    ('{"type":"clic","user":"alice","ts":1000}'),
    ('{"type":"vue","user":"bob","ts":1100}'),
    ('{"type":"clic","user":"charlie","ts":1200}');

-- ❌ Sans index sur l'expression : SCAN complet
EXPLAIN QUERY PLAN  
SELECT * FROM evenements WHERE json_extract(payload, '$.type') = 'clic';  
-- Plan : SCAN evenements

-- ✅ Index sur l'expression JSON
CREATE INDEX idx_type_json ON evenements(json_extract(payload, '$.type'));

EXPLAIN QUERY PLAN  
SELECT * FROM evenements WHERE json_extract(payload, '$.type') = 'clic';  
-- Plan : SEARCH evenements USING INDEX idx_type_json (<expr>=?)
```

**Conditions pour que l'index JSON soit utilisé :**
- La requête doit utiliser **exactement la même expression** que l'index : `json_extract(payload, '$.type')` est différent de `payload->>'$.type'` côté planificateur (même si le résultat est identique). Choisir une syntaxe et s'y tenir.
- L'expression doit être **déterministe** : `json_extract(...)` l'est ; `random_blob(json_extract(...))` ne l'est pas.

**Variante : opérateur `->>` (SQLite 3.38+)** — strictement équivalent et plus lisible :
```sql
-- Définition cohérente (même expression dans l'index et la requête)
CREATE INDEX idx_user_json ON evenements(payload ->> '$.user');

SELECT * FROM evenements WHERE payload ->> '$.user' = 'alice';
-- ✅ Utilise idx_user_json
```

**Alternative : colonne générée + index** (plus verbeux mais expression visible dans `PRAGMA table_info`) :
```sql
ALTER TABLE evenements ADD COLUMN type_evt TEXT
    GENERATED ALWAYS AS (json_extract(payload, '$.type')) VIRTUAL;
CREATE INDEX idx_type_evt ON evenements(type_evt);

SELECT * FROM evenements WHERE type_evt = 'clic';
-- ✅ Utilise idx_type_evt
```

> 💡 **Quand préférer l'index direct sur expression vs la colonne générée ?**  
> - **Index direct sur `json_extract`** : moins de code, pas de modification de schéma. Idéal si l'expression est utilisée à un seul endroit.  
> - **Colonne générée + index** : si la même clé JSON est lue dans de nombreuses requêtes (la colonne devient un raccourci lisible), ou si vous voulez l'inclure dans un index composite avec d'autres colonnes.

> ⚠️ **Piège fréquent** : si la clé JSON peut être **absente** ou de **type variable** (parfois STRING, parfois INTEGER), `json_extract` retourne `NULL` ou un type différent — l'index reste utilisable, mais les comparaisons doivent être cohérentes (`= 'clic'` vs `= 1`). Voir le module 4 pour les détails sur le typage JSON dans SQLite.

## Exemple pratique complet

### Créons une base de données e-commerce

```sql
-- Table des produits
CREATE TABLE produits (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    prix REAL NOT NULL,
    categorie TEXT,
    marque TEXT,
    stock INTEGER DEFAULT 0,
    date_ajout DATE DEFAULT CURRENT_DATE,
    actif BOOLEAN DEFAULT 1
);

-- Insérons quelques données de test
INSERT INTO produits (nom, prix, categorie, marque, stock) VALUES
('iPhone 14', 899.99, 'electronique', 'Apple', 50),
('Samsung Galaxy', 799.99, 'electronique', 'Samsung', 30),
('MacBook Pro', 1299.99, 'ordinateur', 'Apple', 10),
('Nike Air Max', 129.99, 'chaussure', 'Nike', 100),
('Adidas Stan Smith', 89.99, 'chaussure', 'Adidas', 75);
```

### Analysons les requêtes courantes

```sql
-- Requête 1 : Recherche par nom
EXPLAIN QUERY PLAN  
SELECT * FROM produits WHERE nom LIKE '%iPhone%';  
-- Résultat : SCAN produits (lent !)

-- Requête 2 : Filtre par catégorie et prix
EXPLAIN QUERY PLAN  
SELECT * FROM produits WHERE categorie = 'electronique' AND prix < 1000;  
-- Résultat : SCAN produits (lent !)

-- Requête 3 : Top 10 des produits les plus chers
EXPLAIN QUERY PLAN  
SELECT * FROM produits ORDER BY prix DESC LIMIT 10;  
-- Résultat : SCAN + USE TEMP B-TREE (très lent !)
```

### Créons les index appropriés

```sql
-- Index pour les recherches par catégorie et prix
CREATE INDEX idx_categorie_prix ON produits(categorie, prix);

-- Index pour les tris par prix
CREATE INDEX idx_prix ON produits(prix DESC);

-- Index pour les recherches par marque
CREATE INDEX idx_marque ON produits(marque);

-- Index partiel pour les produits actifs
CREATE INDEX idx_produits_actifs ON produits(nom) WHERE actif = 1;
```

### Vérifions l'amélioration

```sql
-- Requête 2 maintenant optimisée :
EXPLAIN QUERY PLAN  
SELECT * FROM produits WHERE categorie = 'electronique' AND prix < 1000;  
-- Résultat : SEARCH produits USING INDEX idx_categorie_prix

-- Requête 3 maintenant optimisée :
EXPLAIN QUERY PLAN  
SELECT * FROM produits ORDER BY prix DESC LIMIT 10;  
-- Résultat : SCAN produits USING INDEX idx_prix
-- ℹ️ Le mot "SCAN" est trompeur ici : SQLite parcourt l'index DANS L'ORDRE
--    (sans trier à part) puis prend les 10 premiers via LIMIT — très rapide.
--    C'est différent d'un "SCAN" sur la table : pas de TEMP B-TREE pour ORDER BY.
```

## Gestion et maintenance des index

### Lister tous les index

```sql
-- Index d'une table spécifique
PRAGMA index_list(produits);

-- Détails d'un index spécifique
PRAGMA index_info(idx_categorie_prix);

-- Tous les index de la base
SELECT name, tbl_name FROM sqlite_master WHERE type = 'index';
```

### Supprimer des index

```sql
-- Supprimer un index devenu inutile
DROP INDEX idx_ancien_index;

-- ℹ️ Bonne nouvelle : SQLite REFUSE de supprimer les index automatiques (UNIQUE / PK).
--    Tenter `DROP INDEX sqlite_autoindex_employes_1;` lève une erreur :
--    "index associated with UNIQUE or PRIMARY KEY constraint cannot be dropped"
--    Pour retirer l'index, il faut retirer la contrainte (ALTER TABLE ou recréer la table).
```

### Reconstruire les index

```sql
-- Reconstruire tous les index (après beaucoup de modifications)
REINDEX;

-- Reconstruire les index d'une table
REINDEX produits;

-- Reconstruire un index spécifique
REINDEX idx_categorie_prix;
```

## Stratégies d'indexation avancées

### 1. Index de couverture (Covering Index)

```sql
-- Au lieu de juste indexer les colonnes du WHERE
CREATE INDEX idx_basic ON produits(categorie);

-- Créer un index qui "couvre" toute la requête
CREATE INDEX idx_covering ON produits(categorie, nom, prix);

-- Cette requête n'aura même pas besoin de lire la table !
SELECT nom, prix FROM produits WHERE categorie = 'electronique';
```

### 2. Index pour les jointures

```sql
-- Table des commandes
CREATE TABLE commandes (
    id INTEGER PRIMARY KEY,
    produit_id INTEGER,
    quantite INTEGER,
    date_commande DATE,
    FOREIGN KEY(produit_id) REFERENCES produits(id)
);

-- Index pour optimiser les jointures
CREATE INDEX idx_produit_id ON commandes(produit_id);

-- Maintenant cette jointure sera rapide :
SELECT p.nom, c.quantite  
FROM produits p  
JOIN commandes c ON p.id = c.produit_id  
WHERE p.categorie = 'electronique';  
```

### 3. Index pour les requêtes avec ORDER BY

```sql
-- Pour cette requête courante :
SELECT * FROM produits  
WHERE categorie = 'electronique'  
ORDER BY prix DESC;  

-- Cet index optimise à la fois le WHERE et l'ORDER BY :
CREATE INDEX idx_cat_prix_desc ON produits(categorie, prix DESC);
```

## Bonnes pratiques

### ✅ Quand créer un index

**Créez un index si :**
- La colonne est souvent utilisée dans `WHERE`
- La colonne est utilisée pour `ORDER BY`
- La colonne est utilisée dans les `JOIN`
- Vous avez des requêtes lentes sur de grandes tables

**Exemple typique :**
```sql
-- Cette requête est exécutée 1000 fois par jour
SELECT * FROM employes WHERE departement = ? AND statut = 'actif';

-- Index recommandé :
CREATE INDEX idx_dept_statut ON employes(departement) WHERE statut = 'actif';
```

### ❌ Quand éviter les index

**Évitez les index si :**
- Table très petite (< 1000 lignes)
- Colonne modifiée très fréquemment
- Colonne avec peu de valeurs différentes (ex: sexe: M/F)

### 📊 Règles de dimensionnement

```sql
-- Pour une table de 1 million de lignes :
-- ✅ Index sur ID (PRIMARY KEY) - obligatoire
-- ✅ Index sur colonnes de recherche fréquente - 2-3 index max
-- ✅ Index composite pour requêtes complexes - 1-2 index
-- ❌ Éviter plus de 5-6 index par table
```

## Mesurer l'impact des index

### Avant/Après avec timing

```sql
-- Activer le timing
.timer on

-- Test sans index
SELECT COUNT(*) FROM produits WHERE prix BETWEEN 100 AND 500;
-- Temps : ex. 0.15 secondes

-- Créer l'index
CREATE INDEX idx_prix_range ON produits(prix);

-- Même requête avec index
SELECT COUNT(*) FROM produits WHERE prix BETWEEN 100 AND 500;
-- Temps : ex. 0.02 secondes → 7x plus rapide !
```

### Benchmark concret sur 1 million de lignes

Pour rendre l'impact des index tangible, voici un script reproductible (testé sous **SQLite 3.45.1**) qui mesure les temps réels avant et après création des index.

**Script complet :**
```sql
-- Préparation : table de test avec 1 million de lignes
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  

CREATE TABLE produits (
    id        INTEGER PRIMARY KEY,
    nom       TEXT NOT NULL,
    prix      REAL,
    categorie TEXT,
    stock     INTEGER
);

-- Génération des 1 000 000 lignes (≈1 seconde)
BEGIN;  
WITH RECURSIVE cnt(x) AS (  
    SELECT 1 UNION ALL SELECT x + 1 FROM cnt WHERE x < 1000000
)
INSERT INTO produits (nom, prix, categorie, stock)  
SELECT  
    'produit_' || x,
    (ABS(RANDOM()) % 10000) / 10.0,           -- prix entre 0 et 999.9
    CASE ABS(RANDOM()) % 5
        WHEN 0 THEN 'electronique'
        WHEN 1 THEN 'vetement'
        WHEN 2 THEN 'alimentaire'
        WHEN 3 THEN 'meuble'
        ELSE 'divers'
    END,
    ABS(RANDOM()) % 1000
FROM cnt;  
COMMIT;  

.timer on
```

**Mesures AVANT index (SCAN complet) :**
```sql
SELECT COUNT(*) FROM produits WHERE categorie = 'electronique';
-- → 199 856 lignes  |  Temps : ≈ 82 ms  |  Plan : SCAN produits

SELECT id, nom, prix FROM produits ORDER BY prix DESC LIMIT 10;
-- → 10 lignes  |  Temps : ≈ 93 ms  |  Plan : SCAN + USE TEMP B-TREE FOR ORDER BY

SELECT COUNT(*) FROM produits WHERE categorie = 'electronique' AND prix < 500;
-- → 99 907 lignes  |  Temps : ≈ 90 ms  |  Plan : SCAN produits
```

**Création des index (≈2.6 s au total) :**
```sql
CREATE INDEX idx_categorie ON produits(categorie);         -- ≈ 580 ms  
CREATE INDEX idx_prix      ON produits(prix DESC);          -- ≈ 830 ms  
CREATE INDEX idx_cat_prix  ON produits(categorie, prix);    -- ≈ 1180 ms  
ANALYZE;                                                    -- ≈ 260 ms  
```

**Mesures APRÈS index :**
```sql
SELECT COUNT(*) FROM produits WHERE categorie = 'electronique';
-- → 199 856 lignes  |  Temps : ≈ 11 ms  ⚡ 7× plus rapide
-- Plan : SEARCH produits USING COVERING INDEX idx_cat_prix (categorie=?)

SELECT id, nom, prix FROM produits ORDER BY prix DESC LIMIT 10;
-- → 10 lignes  |  Temps : ≈ 0.1 ms  ⚡ ~900× plus rapide
-- Plan : SCAN produits USING INDEX idx_prix (parcours dans l'ordre, arrêt après 10)

SELECT COUNT(*) FROM produits WHERE categorie = 'electronique' AND prix < 500;
-- → 99 907 lignes  |  Temps : ≈ 7 ms  ⚡ 13× plus rapide
-- Plan : SEARCH produits USING COVERING INDEX idx_cat_prix (categorie=? AND prix<?)
```

**Tableau récapitulatif :**

| Requête | Avant index | Après index | Accélération |
|---|---:|---:|---:|
| `WHERE categorie = ?` | 82 ms | 11 ms | **7×** |
| `ORDER BY prix DESC LIMIT 10` | 93 ms | 0.1 ms | **~900×** |
| `WHERE categorie = ? AND prix < ?` | 90 ms | 7 ms | **13×** |

**Enseignements concrets :**

1. **L'accélération dépend du type de requête** : un `ORDER BY ... LIMIT` profite massivement de l'index (l'index est trié, on s'arrête après N lignes), bien plus qu'un `WHERE` qui retourne 20 % des lignes.
2. **`COVERING INDEX`** : l'index `idx_cat_prix(categorie, prix)` couvre toutes les colonnes du `WHERE` ET retourne `COUNT(*)` sans toucher à la table → encore plus rapide.
3. **Coût de création** : ~2.6 secondes pour 3 index sur 1 M lignes. Acceptable en production avec `ANALYZE` final.
4. **Plus la table est grande, plus l'écart se creuse** : à 10 M lignes, l'écart passe de ~10× à ~50-100×.

> ⚠️ **Mise en garde sur les temps mesurés** : les chiffres ci-dessus dépendent du matériel (SSD/HDD, RAM, CPU), du cache OS, et de l'état chaud/froid de la base. **Reproduisez le test sur votre matériel** — les **ratios** restent stables, pas les valeurs absolues.

### Analyser l'utilisation des index

```sql
-- Voir si un index est utilisé
EXPLAIN QUERY PLAN  
SELECT * FROM produits WHERE prix > 100;  

-- Si vous voyez "USING INDEX", c'est bon !
-- Si vous voyez "SCAN", l'index n'est pas utilisé
```

## Problèmes courants et solutions

### Problème 1 : Index non utilisé

```sql
-- ❌ Cette requête n'utilisera pas l'index « normal » sur `nom`
SELECT * FROM employes WHERE UPPER(nom) = 'ALICE';

-- ✅ Solution A : INDEX sur l'expression
--    (`IF NOT EXISTS` car nous avons déjà créé cet index plus haut dans le chapitre)
CREATE INDEX IF NOT EXISTS idx_nom_upper ON employes(UPPER(nom));  
SELECT * FROM employes WHERE UPPER(nom) = 'ALICE';  
-- → utilise idx_nom_upper

-- ✅ Solution B : INDEX dédié COLLATE NOCASE (la collation doit MATCHER la requête)
CREATE INDEX idx_nom_nocase ON employes(nom COLLATE NOCASE);  
SELECT * FROM employes WHERE nom = 'Alice' COLLATE NOCASE;  
-- → utilise idx_nom_nocase
-- ⚠️ Un index « normal » sur `nom` (sans COLLATE NOCASE) ne suffit PAS pour cette
--    requête avec `COLLATE NOCASE` : les collations doivent correspondre exactement.
```

### Problème 2 : Ordre des colonnes dans index composite

**Règle pratique :** mettre **en premier** la colonne avec **égalité** (`=`, `IN`), **ensuite** celle avec **inégalité** (`>`, `<`, `BETWEEN`). La sélectivité compte aussi, mais l'opérateur prime.

```sql
-- ❌ Mauvais ordre : la 1ʳᵉ colonne de l'index n'apparaît pas dans le WHERE
CREATE INDEX idx_mauvais ON employes(age, departement);  
SELECT * FROM employes WHERE departement = 'IT';  -- Index inutilisable !  

-- ✅ Bon ordre : on commence par la colonne effectivement filtrée
CREATE INDEX idx_bon ON employes(departement, age);  
SELECT * FROM employes WHERE departement = 'IT';                -- ✅ utilise idx_bon  
SELECT * FROM employes WHERE departement = 'IT' AND age > 30;   -- ✅ utilise idx_bon  
SELECT * FROM employes WHERE age > 30;                          -- ❌ ne peut PAS utiliser idx_bon  
```

> 💡 **Pourquoi ?** Un index B-tree composite est trié d'abord sur la 1ʳᵉ colonne, puis sur la 2ᵉ à l'intérieur de chaque valeur de la 1ʳᵉ. C'est comme un annuaire trié par ville puis par nom : on peut chercher « tous les Dupont à Paris » très vite, mais « tous les Dupont, peu importe la ville » oblige à parcourir tout l'annuaire.

### Problème 3 : Trop d'index

```sql
-- ❌ Trop d'index ralentit les écritures
CREATE INDEX idx1 ON produits(nom);  
CREATE INDEX idx2 ON produits(prix);  
CREATE INDEX idx3 ON produits(categorie);  
CREATE INDEX idx4 ON produits(marque);  
CREATE INDEX idx5 ON produits(stock);  

-- ✅ Mieux : index composites intelligents
CREATE INDEX idx_recherche ON produits(categorie, prix);  
CREATE INDEX idx_gestion ON produits(marque, stock) WHERE actif = 1;  
```

## Exercices pratiques

### Exercice 1 : Diagnostic d'une table lente

1. Créez cette table avec beaucoup de données :
```sql
CREATE TABLE ventes (
    id INTEGER PRIMARY KEY,
    vendeur TEXT,
    produit TEXT,
    montant REAL,
    region TEXT,
    date_vente DATE
);

-- Insérez 10 000 lignes de données de test (utilisez un script)
```

2. Testez ces requêtes et mesurez leur temps :
```sql
.timer on
SELECT * FROM ventes WHERE vendeur = 'Alice';  
SELECT * FROM ventes WHERE region = 'Nord' AND montant > 1000;  
SELECT vendeur, SUM(montant) FROM ventes GROUP BY vendeur ORDER BY SUM(montant) DESC;  
```

3. Créez les index appropriés et mesurez l'amélioration

### Exercice 2 : Optimisation d'une requête complexe

```sql
-- Requête à optimiser :
SELECT v.vendeur, p.nom, SUM(v.montant)  
FROM ventes v  
JOIN produits p ON v.produit = p.nom  
WHERE v.date_vente >= '2024-01-01'  
  AND p.categorie = 'electronique'
GROUP BY v.vendeur, p.nom  
ORDER BY SUM(v.montant) DESC;  
```

Créez les index nécessaires pour optimiser cette requête.

## Guide de dépannage rapide

### Ma requête est lente - Checklist

1. **Vérifier le plan d'exécution**
```text
EXPLAIN QUERY PLAN votre_requete;
```

2. **Chercher les SCAN sur grandes tables**
- Si vous voyez `SCAN table_name` sur une table > 1000 lignes → créez un index !

3. **Vérifier les colonnes du WHERE**
```text
-- Si votre WHERE utilise ces colonnes :
WHERE colonne1 = ? AND colonne2 > ?
-- Créez cet index :
CREATE INDEX idx_optim ON ma_table(colonne1, colonne2);
```

4. **Tester l'amélioration**
```sql
.timer on
-- Votre requête avant et après
```

### Index créé mais pas utilisé ?

**Causes possibles :**
- Statistiques obsolètes → `ANALYZE nom_table;`
- Fonction dans le WHERE → créer index sur l'expression
- Mauvais ordre des colonnes → recréer l'index dans le bon ordre
- Requête récupère trop de données → index partiel ou requête plus sélective

## Résumé

Les index SQLite :

✅ **Accélèrent drastiquement** les requêtes sur grandes tables  
✅ **Se créent facilement** avec `CREATE INDEX`  
✅ **Peuvent être composites** (plusieurs colonnes)  
✅ **Peuvent être partiels** (avec condition WHERE)  
✅ **Optimisent les tris** (ORDER BY) aussi

**Points clés à retenir :**
- Index = raccourci pour trouver les données rapidement
- Créez des index sur vos colonnes de recherche fréquente
- L'ordre des colonnes dans un index composite est crucial
- Trop d'index ralentit les écritures
- Mesurez toujours l'impact avec `.timer on`

**Règle d'or :** Un index bien placé peut transformer une requête de 10 secondes en 0.01 seconde !

Dans la section suivante, nous apprendrons à analyser en détail les plans d'exécution avec `EXPLAIN QUERY PLAN` pour devenir des experts en optimisation SQLite.

⏭️ [5.3 Analyse des plans d'exécution avec EXPLAIN QUERY PLAN](/05-optimisation-performances/03-analyse-plans-execution-explain-query-plan.md)
