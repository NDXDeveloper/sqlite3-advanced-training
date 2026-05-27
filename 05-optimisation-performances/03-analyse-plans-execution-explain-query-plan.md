🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.3 Analyse des plans d'exécution avec EXPLAIN QUERY PLAN

## Introduction à EXPLAIN QUERY PLAN

`EXPLAIN QUERY PLAN` est votre meilleur ami pour optimiser SQLite ! C'est comme avoir une radiographie de vos requêtes : cela vous montre exactement ce que fait SQLite pour récupérer vos données.

### Pourquoi c'est important ?

Imaginez que vous demandez à quelqu'un de trouver un livre dans une bibliothèque :
- **Méthode lente :** "Je vais regarder livre par livre dans toute la bibliothèque"
- **Méthode rapide :** "Je vais directement au bon rayon grâce au catalogue"

`EXPLAIN QUERY PLAN` vous dit quelle méthode SQLite utilise pour votre requête !

## Syntaxe et utilisation de base

### Format simple

```text
EXPLAIN QUERY PLAN  
SELECT colonne FROM nom_table WHERE condition;  
```
(`table`, en SQL, est un mot-clé réservé — on l'a remplacé par `nom_table` pour que le template soit copiable-collable.)

### Premier exemple concret

```sql
-- Créons une table d'exemple
CREATE TABLE employes (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    age INTEGER,
    departement TEXT,
    salaire REAL
);

-- Insérons quelques données
INSERT INTO employes VALUES
(1, 'Alice', 28, 'IT', 50000),
(2, 'Bob', 35, 'RH', 45000),
(3, 'Charlie', 42, 'Finance', 60000),
(4, 'Diana', 29, 'IT', 52000),
(5, 'Eve', 38, 'Marketing', 48000);

-- Analysons cette requête
EXPLAIN QUERY PLAN  
SELECT nom, salaire FROM employes WHERE departement = 'IT';  
```

**Résultat typique :**
```
SCAN employes
```

**Traduction :** SQLite va regarder chaque ligne de la table `employes` une par une.

## Comprendre les résultats d'EXPLAIN QUERY PLAN

### Structure de base du résultat

Les résultats s'affichent sous cette forme :
```
|--SCAN employes
```

Le préfixe `|--` indique le niveau de l'opération dans le plan d'exécution.

### Vocabulaire essentiel

| Terme | Signification | Performance |
|-------|---------------|-------------|
| **SCAN** | Lecture complète de la table | 🐌 Lent sur grandes tables (mais OK sur petite ou avec INDEX, voir ci-dessous) |
| **SEARCH** | Lecture ciblée via un index | 🚀 Rapide |
| **USING INDEX `nom`** | Précise quel index est utilisé | ✅ Très bon signe |
| **USING INTEGER PRIMARY KEY** | Accès direct au `rowid` (cas le plus rapide) | ⭐ Optimal |
| **USING INDEX `sqlite_autoindex_…`** | Index automatique d'une `PRIMARY KEY` non-INTEGER ou `UNIQUE` | ⭐ Optimal |
| **USING COVERING INDEX** | L'index contient toutes les colonnes nécessaires → pas de lecture de la table | 🚀 Très bon |
| **USE TEMP B-TREE FOR** | Structure temporaire pour ORDER BY/GROUP BY/DISTINCT | ⚠️ Coûteux en mémoire |
| **MULTI-INDEX OR** | Utilise plusieurs index pour un `OR` | ✅ Bon |
| **CORRELATED SCALAR SUBQUERY** | Sous-requête ré-évaluée pour chaque ligne | ⚠️ Lent si table externe grande |

> ℹ️ Attention : `SCAN ... USING INDEX ...` (sans `SEARCH`) signifie que SQLite parcourt l'index **dans l'ordre** — c'est très rapide quand il y a un `ORDER BY` ou un `LIMIT` qui peuvent s'arrêter tôt. Ce **n'est pas** un `SCAN` complet de la table.

## Exemples détaillés avec interprétations

### Exemple 1 : Recherche simple sans index

```sql
EXPLAIN QUERY PLAN  
SELECT * FROM employes WHERE age > 30;  
```

**Résultat :**
```
SCAN employes
```

**Interprétation :**
- SQLite lit **chaque ligne** de la table
- Vérifie si `age > 30` pour chaque employé
- ⚠️ **Problème :** Lent sur une grande table !

**Solution :**
```sql
-- Créer un index sur la colonne age
CREATE INDEX idx_age ON employes(age);

-- Même requête maintenant
EXPLAIN QUERY PLAN  
SELECT * FROM employes WHERE age > 30;  
```

**Nouveau résultat :**
```
SEARCH employes USING INDEX idx_age (age>?)
```

**Interprétation améliorée :**
- SQLite utilise l'index pour aller directement aux bonnes valeurs
- 🚀 **Beaucoup plus rapide !**

### Exemple 2 : Recherche par clé primaire

```sql
EXPLAIN QUERY PLAN  
SELECT nom FROM employes WHERE id = 3;  
```

**Résultat :**
```
SEARCH employes USING INTEGER PRIMARY KEY (rowid=?)
```

**Interprétation :**
- Utilise la clé primaire (optimal !)
- Accès direct à la ligne → très rapide
- ⭐ **Parfait !**

### Exemple 3 : Tri avec ORDER BY

```sql
EXPLAIN QUERY PLAN  
SELECT nom, salaire FROM employes ORDER BY salaire DESC;  
```

**Résultat sans index :**
```
SCAN employes  
USE TEMP B-TREE FOR ORDER BY  
```

**Interprétation :**
1. SQLite lit toute la table (`SCAN`)
2. Crée une structure temporaire pour trier (`TEMP B-TREE`)
3. ⚠️ **Double travail = lent !**

**Avec index sur salaire :**
```sql
CREATE INDEX idx_salaire ON employes(salaire DESC);

EXPLAIN QUERY PLAN  
SELECT nom, salaire FROM employes ORDER BY salaire DESC;  
```

**Nouveau résultat :**
```
SCAN employes USING INDEX idx_salaire
```

**Interprétation améliorée :**
- Les données sont déjà triées dans l'index
- Plus besoin de structure temporaire
- 🚀 **Plus rapide et moins de mémoire !**

## Cas complexes avec jointures

### Exemple avec JOIN

```sql
-- Créons une deuxième table
CREATE TABLE departements (
    nom TEXT PRIMARY KEY,
    budget INTEGER,
    manager TEXT
);

INSERT INTO departements VALUES
('IT', 100000, 'Alice'),
('RH', 50000, 'Bob'),
('Finance', 75000, 'Charlie'),
('Marketing', 60000, 'Eve');

-- Requête avec jointure
EXPLAIN QUERY PLAN  
SELECT e.nom, e.salaire, d.budget  
FROM employes e  
JOIN departements d ON e.departement = d.nom;  
```

**Résultat :**
```
|--SCAN e
`--SEARCH d USING INDEX sqlite_autoindex_departements_1 (nom=?)
```

**Interprétation :**
1. SQLite lit chaque employé (`SCAN e`)
2. Pour chaque employé, cherche le département correspondant via **l'index automatique** créé pour la `PRIMARY KEY` non-INTEGER (`sqlite_autoindex_departements_1`)
3. ✅ **Efficace** : avec une `INTEGER PRIMARY KEY`, on verrait à la place `USING INTEGER PRIMARY KEY` (accès direct au ROWID, encore plus rapide)

### Jointure sans index : que fait vraiment SQLite ?

```sql
-- Ajoutons une table sans clé appropriée
CREATE TABLE projets (
    id INTEGER PRIMARY KEY,
    nom_projet TEXT,
    responsable TEXT,  -- Pas d'index !
    budget INTEGER
);

EXPLAIN QUERY PLAN  
SELECT e.nom, p.nom_projet  
FROM employes e  
JOIN projets p ON e.nom = p.responsable;  
```

**Résultat fréquent (SQLite 3.8+) :**
```
|--SCAN e
`--SEARCH p USING AUTOMATIC COVERING INDEX (responsable=?)
```

**Interprétation :**
- Pas d'index permanent sur `projets.responsable` ⇒ SQLite **construit un index temporaire en mémoire** au début de la requête (`AUTOMATIC COVERING INDEX`), puis l'utilise pour la jointure
- ⚠️ **Inefficace** : l'index temporaire est jeté à la fin de la requête et **reconstruit à chaque exécution**. Pour une requête fréquente, c'est du gâchis.

**Pire cas (sur de très petites tables où l'autoindex n'est pas rentable) :**
```
|--SCAN e
`--SCAN p
```
- Boucle imbriquée (« nested loop ») : pour chaque ligne de `e`, parcours complet de `p`
- Coût : *O(n × m)* — comportement type produit cartésien

**Solution permanente :**
```sql
-- Créer un VRAI index (persistant, partagé entre toutes les requêtes)
CREATE INDEX idx_responsable ON projets(responsable);

-- Maintenant le plan utilise l'index permanent :
-- |--SCAN e
-- `--SEARCH p USING INDEX idx_responsable (responsable=?)
```

> 💡 **`AUTOMATIC COVERING INDEX` n'est pas magique** : il signale que SQLite **manque** d'un index permanent et bricole une solution à la volée. Si ce plan apparaît sur une requête fréquente, **ajoutez l'index manquant**.

## Requêtes avec sous-requêtes

### Sous-requête simple

```sql
EXPLAIN QUERY PLAN  
SELECT nom FROM employes  
WHERE departement IN (  
    SELECT nom FROM departements WHERE budget > 60000
);
```

**Résultat :**
```
SCAN employes  
LIST SUBQUERY 1  
  SCAN departements
```

**Interprétation :**
1. SQLite exécute d'abord la sous-requête (`LIST SUBQUERY 1`)
2. Puis utilise les résultats pour filtrer les employés
3. ✅ **Approche efficace** car la sous-requête ne s'exécute qu'une fois

### Sous-requête corrélée (attention !)

```sql
EXPLAIN QUERY PLAN  
SELECT nom FROM employes e1  
WHERE salaire > (  
    SELECT AVG(salaire) FROM employes e2
    WHERE e2.departement = e1.departement
);
```

**Résultat :**
```
SCAN e1  
CORRELATED SCALAR SUBQUERY 1  
  SCAN e2
```

**Interprétation :**
- ⚠️ **Problème :** La sous-requête s'exécute pour chaque ligne de e1 !
- Si 1000 employés → 1000 exécutions de la sous-requête
- **Très lent !**

## Techniques d'optimisation basées sur les plans

### 1. Identifier les SCAN problématiques

**Règle :** Un `SCAN` sur une table > 1000 lignes est souvent un problème.

```sql
-- ❌ Problématique
EXPLAIN QUERY PLAN  
SELECT * FROM employes WHERE departement = 'IT' AND age > 25;  
-- Résultat : SCAN employes

-- ✅ Solution
CREATE INDEX idx_dept_age ON employes(departement, age);
-- Nouveau résultat : SEARCH employes USING INDEX idx_dept_age
```

### 2. Éliminer les TEMP B-TREE

```sql
-- ❌ Problématique
EXPLAIN QUERY PLAN  
SELECT * FROM employes ORDER BY departement, salaire DESC;  
-- Résultat : SCAN employes + USE TEMP B-TREE FOR ORDER BY

-- ✅ Solution
CREATE INDEX idx_dept_sal ON employes(departement, salaire DESC);
-- Nouveau résultat : SCAN employes USING INDEX idx_dept_sal
```

### 3. Optimiser les jointures

> ℹ️ **Idée reçue à oublier** : « la syntaxe `FROM a, b WHERE a.x = b.x` produit un produit cartésien, il faut absolument utiliser `JOIN` ». **Faux** : SQLite (et tous les SGBD modernes) traitent les deux syntaxes **strictement identiquement** quand une condition de jointure est présente dans le `WHERE`. `EXPLAIN QUERY PLAN` produit le même plan.

```sql
-- Ces deux requêtes produisent EXACTEMENT le même plan d'exécution :
EXPLAIN QUERY PLAN  
SELECT e.nom, d.budget  
FROM employes e, departements d                          -- syntaxe historique  
WHERE e.departement = d.nom AND d.budget > 50000;  

EXPLAIN QUERY PLAN  
SELECT e.nom, d.budget  
FROM employes e  
JOIN departements d ON e.departement = d.nom             -- syntaxe ANSI moderne  
WHERE d.budget > 50000;  
-- Plan identique pour les deux : SCAN e + SEARCH d USING INDEX sqlite_autoindex_…
```

> 💡 **Préférer `JOIN ... ON ...`** reste recommandé pour la **lisibilité** (séparation claire entre condition de jointure et filtres `WHERE`) et pour éviter le **vrai piège** : un produit cartésien accidentel quand on oublie la condition `a.x = b.x` dans le `WHERE`. Avec `JOIN ... ON`, SQLite refuse même la requête si on omet le `ON` (erreur de syntaxe), donc impossible d'oublier la jointure.

**Le vrai souci de performance reste l'absence d'index sur la colonne de jointure :**

```sql
-- ❌ Inefficace : pas d'index permanent sur projets.responsable
-- (table déjà créée plus haut dans le chapitre avec `nom_projet TEXT, ..., budget INTEGER`,
--  on la réutilise telle quelle)
EXPLAIN QUERY PLAN  
SELECT e.nom, p.nom_projet FROM employes e JOIN projets p ON e.nom = p.responsable;  
-- Plan : SCAN e + SEARCH p USING AUTOMATIC COVERING INDEX (responsable=?)
-- ⚠️ SQLite construit un index temporaire EN MÉMOIRE à chaque exécution
--    (signal qu'il manque un vrai index permanent — détails dans la section précédente).

-- ✅ Avec index permanent sur la clé de jointure
CREATE INDEX IF NOT EXISTS idx_responsable ON projets(responsable);
-- Plan : SCAN e + SEARCH p USING INDEX idx_responsable (responsable=?)
-- L'index permanent est partagé entre toutes les requêtes, pas reconstruit.
```

## Mode d'affichage détaillé

### Activer plus de détails

```sql
-- Voir plus d'informations sur les coûts
.eqp on

-- Ou pour une requête spécifique
EXPLAIN QUERY PLAN  
SELECT * FROM employes WHERE age BETWEEN 25 AND 35;  
```

### Estimation du nombre de lignes — où la trouver ?

> ⚠️ **Idée reçue** : on lit parfois que `EXPLAIN QUERY PLAN` afficherait `(~10 rows)` à la fin des lignes. **Faux pour SQLite** : c'est une notation PostgreSQL. SQLite n'inclut **aucune** estimation de cardinalité dans la sortie d'`EXPLAIN QUERY PLAN`.  
>  
> Pour avoir l'équivalent d'une « estimation de sélectivité » côté SQLite, il faut consulter directement la table système peuplée par `ANALYZE` :  
>
> ```sql
> ANALYZE;
> SELECT tbl, idx, stat FROM sqlite_stat1;
> -- stat = "N k1 k2 …" où N = nb total de lignes,
> --        k1 = lignes moyennes par valeur de la 1ʳᵉ colonne d'index, etc.
> ```
>  
> Pour des coûts détaillés au niveau bytecode (rare en pratique), utiliser `EXPLAIN` (sans `QUERY PLAN`) ou `.eqp full` dans le shell.

## Cas pratiques d'optimisation

### Cas 1 : Rapport de ventes lent

```sql
-- Table de ventes avec beaucoup de données
CREATE TABLE ventes (
    id INTEGER PRIMARY KEY,
    vendeur TEXT,
    produit TEXT,
    montant REAL,
    date_vente DATE,
    region TEXT
);

-- Requête problématique
EXPLAIN QUERY PLAN  
SELECT vendeur, SUM(montant) as total  
FROM ventes  
WHERE date_vente >= '2024-01-01'  
  AND region = 'Nord'
GROUP BY vendeur  
ORDER BY total DESC;  
```

**Résultat avant optimisation :**
```
|--SCAN ventes
|--USE TEMP B-TREE FOR GROUP BY
`--USE TEMP B-TREE FOR ORDER BY
```

**Problèmes identifiés :**
1. `SCAN` complet de la table
2. Deux structures temporaires (GROUP BY et ORDER BY)

**Solution d'optimisation :**
```sql
-- Index composite pour le WHERE
-- ⚠️ Ordre crucial : la colonne avec ÉGALITÉ (`region = ?`) doit venir AVANT
--    celle avec inégalité (`date_vente >= ?`). Inverser l'ordre rend l'index
--    inutilisable pour la condition d'égalité.
CREATE INDEX idx_region_date ON ventes(region, date_vente);
```

**Résultat après optimisation :**
```
|--SEARCH ventes USING INDEX idx_region_date (region=? AND date_vente>?)
|--USE TEMP B-TREE FOR GROUP BY
`--USE TEMP B-TREE FOR ORDER BY
```

**Amélioration :** Le `SCAN` complet devient un `SEARCH` ciblé → on lit uniquement les ventes de la région Nord depuis le 1ᵉʳ janvier 2024 (au lieu de toute la table).

> 💡 **Pour aller plus loin** : les deux `TEMP B-TREE` restants sont **incompressibles** ici car `GROUP BY vendeur` et `ORDER BY total DESC` portent sur des colonnes différentes (et `total` est un agrégat, donc impossible à pré-trier dans un index). Acceptable tant que le `SEARCH` a déjà filtré la majorité des lignes.

### Cas 2 : Pagination inefficace

```sql
-- Pagination classique problématique
EXPLAIN QUERY PLAN  
SELECT * FROM employes  
ORDER BY nom  
LIMIT 20 OFFSET 1000;  
```

**Résultat :**
```
SCAN employes  
USE TEMP B-TREE FOR ORDER BY  
```

**Problème :** SQLite trie TOUS les employés pour ensuite ignorer les 1000 premiers !

**Solution :**
```sql
-- Index sur la colonne de tri
CREATE INDEX idx_nom ON employes(nom);

-- Nouveau plan
EXPLAIN QUERY PLAN  
SELECT * FROM employes  
ORDER BY nom  
LIMIT 20 OFFSET 1000;  
```

**Résultat optimisé :**
```
`--SCAN employes USING INDEX idx_nom
```

**Amélioration :** Plus de `TEMP B-TREE FOR ORDER BY` ! Le mot « SCAN » est trompeur : SQLite parcourt l'index `idx_nom` **dans l'ordre** et n'a besoin de visiter que les 1020 premières entrées (offset 1000 + limit 20) avant de s'arrêter.

> ⚠️ **Limite de l'approche `OFFSET`** : même avec index, SQLite doit parcourir et jeter les 1000 premières entrées. Pour de très gros OFFSET (>10 000), préférer la **pagination par curseur** : `WHERE nom > 'dernier_nom_vu' ORDER BY nom LIMIT 20` (lecture directe à partir du dernier élément vu, sans rien jeter).

## Outils complémentaires pour l'analyse

### Boîte à outils complète du shell SQLite

Le **shell `sqlite3`** propose un ensemble de commandes « point » (préfixées par `.`) extrêmement utiles pour le profilage. Voici la check-list complète à activer avant toute session d'optimisation.

```text
─────────────────────────────────────────────────────────────────────
COMMANDE              EFFET                                        UTILITÉ
─────────────────────────────────────────────────────────────────────
.timer on             Affiche le temps réel/user/sys après          ⭐⭐⭐
                      chaque requête.

.stats on             Affiche les compteurs internes après          ⭐⭐⭐
                      chaque requête : Memory Used, Pages Read,
                      Pages Written, Lookaside slots, etc.

.eqp on               Préfixe AUTOMATIQUEMENT chaque SELECT par     ⭐⭐⭐
                      EXPLAIN QUERY PLAN — plus besoin de l'écrire.

.eqp full             Comme .eqp on, mais ajoute EXPLAIN bytecode   ⭐⭐
                      complet (très verbeux, pour cas extrêmes).

.eqp trigger          Inclut les plans des triggers déclenchés.     ⭐

.echo on              Affiche chaque commande avant exécution       ⭐⭐
                      (utile pour scripts/journaux).

.explain auto         Active le mode colonne pour la sortie         ⭐⭐
                      d'EXPLAIN (lisible vs format brut).

.headers on           Affiche le nom des colonnes en en-tête.       ⭐⭐⭐
.mode column          Sortie en colonnes alignées (vs pipes).       ⭐⭐⭐
.mode box             Sortie en tableau ASCII (très lisible).       ⭐⭐⭐

.schema [table]       Affiche le DDL (CREATE TABLE/INDEX/VIEW).     ⭐⭐⭐
.indexes [table]      Liste les index d'une table (ou tous).        ⭐⭐⭐
.tables               Liste toutes les tables et vues.              ⭐⭐
.dbinfo               Métadonnées de la base (taille, pages, etc.)  ⭐⭐

.scanstats vm2        Compteurs détaillés par étape du plan         ⭐
                      (requiert SQLITE_ENABLE_STMT_SCANSTATUS).
─────────────────────────────────────────────────────────────────────
```

### Workflow recommandé pour profiler une requête

```sql
-- Étape 1 : préparer l'environnement
.timer on
.stats on
.eqp on
.headers on
.mode box

-- Étape 2 : lancer la requête à profiler — sortie typique :
SELECT departement, COUNT(*) FROM employes GROUP BY departement;
```

**Sortie complète obtenue :**
```text
QUERY PLAN
|--SCAN employes
`--USE TEMP B-TREE FOR GROUP BY

┌──────────────┬──────────┐
│ departement  │ COUNT(*) │
├──────────────┼──────────┤
│ Finance      │  124     │
│ IT           │  428     │
│ Marketing    │  201     │
│ RH           │   89     │
└──────────────┴──────────┘

Memory Used:                         84 (max 152) bytes  
Number of Outstanding Allocations:   0 (max 17)  
Number of Pcache Overflow Bytes:     0 (max 0)  
Largest Allocation:                  120 bytes  
Lookaside Slots Used:                0 (max 8)  
Successful lookaside attempts:       55  
Lookaside failures due to size:      0  
Lookaside failures due to OOM:       0  
Pager Heap Usage:                    8696 bytes  
Page cache hits:                     24  
Page cache misses:                   1  
Page cache writes:                   0  
Schema Heap Usage:                   1992 bytes  
Statement Heap/Lookaside Usage:      1280 bytes  
Fullscan Steps:                      842  
Sort Operations:                     1  
Autoindex Inserts:                   0  
Virtual Machine Steps:               2554  
Reprepare operations:                0  
Number of times run:                 1  
Memory used by prepared stmt:        2288  
Bytes received by read():            11944  
Bytes sent to write():               8835  
Read() system calls:                 22  
Write() system calls:                5  
Bytes read from storage:             0  
Bytes written to storage:            0  
Run Time: real 0.002 user 0.001876 sys 0.000000  
```

> ℹ️ La liste exacte des compteurs varie selon la version de SQLite et les options de compilation. La sortie ci-dessus correspond à **SQLite 3.45.1** avec les options standard de Debian/Ubuntu. Les compteurs les plus stables (présents depuis longtemps) sont ceux marqués comme « à surveiller » ci-dessous.

**Métriques à surveiller particulièrement :**
- **`Fullscan Steps`** : nombre de lignes scannées sans index. Doit être proche de 0 sur grandes tables.
- **`Sort Operations`** : >0 signifie un `TEMP B-TREE` — index manquant pour `ORDER BY`/`GROUP BY` ?
- **`Autoindex Inserts`** : >0 signifie qu'un `AUTOMATIC COVERING INDEX` a été construit en mémoire — il manque un index permanent.
- **`Page cache misses`** : si très élevé, augmenter `PRAGMA cache_size` peut aider.
- **`Virtual Machine Steps`** : ordre de grandeur des opérations VDBE — proportionnel à la complexité réelle.

### Forcer la mise à jour des statistiques

```sql
-- SQLite utilise des statistiques pour optimiser
-- Les mettre à jour après de gros changements :
ANALYZE employes;

-- Ou toute la base :
ANALYZE;

-- Sur très grosse base : limiter le coût d'ANALYZE (SQLite 3.32+)
PRAGMA analysis_limit = 1000;  -- max 1000 échantillons par index  
ANALYZE;                        -- beaucoup plus rapide, précision quasi-identique  
```

### Astuce : profiler en non-interactif

```bash
# Une session de profilage complète en un seul shell
sqlite3 ma_base.db <<'EOF'
.timer on
.stats on
.eqp on
EXPLAIN QUERY PLAN SELECT … FROM … WHERE …;  
SELECT … FROM … WHERE …;  
EOF  
```

> 💡 **Pour Python / autres langages** : les commandes `.timer`, `.stats`, `.eqp` sont **propres au shell `sqlite3`** — elles n'existent pas dans les bindings (sqlite3 Python, libsqlite C, …). Pour profiler depuis du code applicatif :  
> - **Python** : `time.perf_counter()` autour de `cursor.execute()`, et `cursor.execute("EXPLAIN QUERY PLAN " + sql)` pour récupérer le plan.  
> - **C/Java/JS** : utiliser `sqlite3_stmt_scanstatus()` (API C) si compilé avec `SQLITE_ENABLE_STMT_SCANSTATUS`.

## Guide de diagnostic rapide

### Checklist pour analyser un plan

1. **Cherchez les mots-clés problématiques :**
   - `SCAN` sur grande table → créer un index
   - `TEMP B-TREE` répétés → optimiser ORDER BY/GROUP BY
   - `SCAN` dans les deux tables d'un JOIN → ajouter index sur clé de jointure

2. **Vérifiez la sélectivité (via `sqlite_stat1` après `ANALYZE`) :**
   - Index sur colonne très sélective (peu de doublons) = idéal
   - Index sur colonne peu sélective (beaucoup de doublons) = potentiellement inefficace

3. **Analysez l'ordre des opérations :**
   - Les filtres (`WHERE`) doivent être appliqués tôt
   - Les tris (`ORDER BY`) devraient utiliser des index

### Exemples de plans optimaux

```text
-- ✅ Excellent plan (recherche multi-critères avec index composite)
SEARCH employes USING INDEX idx_dept_age (departement=? AND age>?)

-- ✅ Bon plan avec jointure (table externe scannée, jointure indexée)
SCAN e USING INDEX idx_dept  
SEARCH d USING INTEGER PRIMARY KEY (rowid=?)       -- pour INTEGER PRIMARY KEY  
-- ou : SEARCH d USING INDEX sqlite_autoindex_X_1   -- pour PRIMARY KEY non-INTEGER

-- ✅ Optimal pour tri (parcours d'index, pas de TEMP B-TREE)
SCAN employes USING INDEX idx_salaire_desc
```

### Exemples de plans problématiques

```text
-- ❌ Problématique
SCAN employes                                    (sur table > 10 000 lignes)

-- ❌ Très problématique : jointure sans index permanent
--    Soit SQLite construit un AUTOMATIC COVERING INDEX (signal d'index manquant) :
SCAN e  
SEARCH p USING AUTOMATIC COVERING INDEX          (reconstruit à chaque requête)  
--    Soit, sur très petites tables, double SCAN :
SCAN table1  
SCAN table2  

-- ❌ Gourmand en mémoire — multiples TEMP B-TREE
USE TEMP B-TREE FOR ORDER BY  
USE TEMP B-TREE FOR GROUP BY  
```

## Exercices pratiques

### Exercice 1 : Diagnostic d'une application e-commerce

```sql
-- Tables d'un site e-commerce
CREATE TABLE clients (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    email TEXT UNIQUE,
    ville TEXT,
    date_inscription DATE
);

CREATE TABLE commandes (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,
    total REAL,
    date_commande DATE,
    statut TEXT
);

-- Requêtes à analyser et optimiser :

-- 1. Recherche de client
EXPLAIN QUERY PLAN  
SELECT * FROM clients WHERE email = 'alice@email.com';  

-- 2. Commandes d'un client
EXPLAIN QUERY PLAN  
SELECT * FROM commandes WHERE client_id = 123 ORDER BY date_commande DESC;  

-- 3. Rapport mensuel
EXPLAIN QUERY PLAN  
SELECT c.nom, COUNT(*) as nb_commandes, SUM(co.total) as total  
FROM clients c  
JOIN commandes co ON c.id = co.client_id  
WHERE co.date_commande >= '2024-01-01'  
GROUP BY c.id, c.nom  
ORDER BY total DESC;  
```

**Mission :** Analysez chaque plan et proposez les index nécessaires.

### Exercice 2 : Optimisation progressive

1. **Créez une table de test avec beaucoup de données**
2. **Testez cette requête complexe :**
```sql
SELECT vendeur,
       COUNT(*) as nb_ventes,
       AVG(montant) as moyenne,
       SUM(montant) as total
FROM ventes v  
WHERE date_vente BETWEEN '2024-01-01' AND '2024-12-31'  
  AND region IN ('Nord', 'Sud')
  AND montant > 100
GROUP BY vendeur  
HAVING COUNT(*) > 5  
ORDER BY total DESC  
LIMIT 10;  
```

3. **Analysez le plan avec `EXPLAIN QUERY PLAN`**
4. **Créez les index un par un et observez l'évolution du plan**
5. **Mesurez l'amélioration des performances avec `.timer on`**

## Résumé et bonnes pratiques

### Points clés à retenir

✅ **EXPLAIN QUERY PLAN** vous montre exactement ce que fait SQLite  
✅ **SCAN** = lent sur grandes tables → créer des index  
✅ **SEARCH USING INDEX** = rapide → bon signe !  
✅ **TEMP B-TREE** = utilisation mémoire → optimiser ORDER BY/GROUP BY  
✅ **Mesurer avant/après** avec `.timer on` pour valider les améliorations

### Workflow d'optimisation

1. **Identifier** les requêtes lentes de votre application
2. **Analyser** avec `EXPLAIN QUERY PLAN`
3. **Chercher** les `SCAN` et `TEMP B-TREE` problématiques
4. **Créer** les index appropriés
5. **Vérifier** l'amélioration du plan
6. **Mesurer** le gain de performance réel

### Règles d'or

- **Un SCAN sur > 1000 lignes** → probablement besoin d'un index
- **Plusieurs TEMP B-TREE** → optimiser les tris et groupements
- **SCAN dans les deux tables d'un JOIN** → ajouter index sur clé de jointure
- **Toujours mesurer l'impact** → certaines optimisations peuvent être contre-productives

`EXPLAIN QUERY PLAN` transforme l'optimisation SQLite d'un art mystérieux en une science précise ! Dans la section suivante, nous verrons comment appliquer concrètement toutes ces connaissances pour optimiser les requêtes les plus lentes de vos applications.

⏭️ [5.4 Optimisation des requêtes lentes](/05-optimisation-performances/04-optimisation-requetes-lentes.md)
