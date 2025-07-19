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

### 1. Index automatiques (Primary Key)

SQLite crée automatiquement des index pour :

```sql
CREATE TABLE employes (
    id INTEGER PRIMARY KEY,  -- Index automatique créé !
    nom TEXT,
    email TEXT UNIQUE       -- Index automatique sur email aussi !
);
```

**Vérification :**
```sql
-- Voir tous les index d'une table
PRAGMA index_list(employes);
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

## Créer des index efficaces

### Syntaxe de base

```sql
-- Syntaxe générale
CREATE INDEX nom_de_l_index ON nom_table(colonne1, colonne2, ...);

-- Exemples pratiques
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

-- Optimise cette requête :
SELECT * FROM employes WHERE nom = 'Alice' AND statut = 'actif';
```

**Avantage :** Index plus petit et plus rapide !

### Index sur expressions

```sql
-- Index sur une expression calculée
CREATE INDEX idx_nom_upper ON employes(UPPER(nom));

-- Optimise cette recherche insensible à la casse :
SELECT * FROM employes WHERE UPPER(nom) = 'ALICE';
```

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
-- Résultat : SEARCH produits USING INDEX idx_prix
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

-- Attention : ne supprimez jamais les index automatiques !
-- DROP INDEX sqlite_autoindex_employes_1;  -- ❌ Ne faites pas ça !
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
-- ❌ Cette requête n'utilisera pas l'index sur 'nom'
SELECT * FROM employes WHERE UPPER(nom) = 'ALICE';

-- ✅ Solutions :
-- Option 1 : Index sur l'expression
CREATE INDEX idx_nom_upper ON employes(UPPER(nom));

-- Option 2 : Requête insensible à la casse
SELECT * FROM employes WHERE nom = 'Alice' COLLATE NOCASE;
```

### Problème 2 : Ordre des colonnes dans index composite

```sql
-- ❌ Mauvais ordre
CREATE INDEX idx_mauvais ON employes(age, departement);
SELECT * FROM employes WHERE departement = 'IT';  -- Index peu efficace

-- ✅ Bon ordre (colonne la plus sélective en premier)
CREATE INDEX idx_bon ON employes(departement, age);
SELECT * FROM employes WHERE departement = 'IT';  -- Index très efficace
```

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
```sql
EXPLAIN QUERY PLAN votre_requete;
```

2. **Chercher les SCAN sur grandes tables**
- Si vous voyez `SCAN table_name` sur une table > 1000 lignes → créez un index !

3. **Vérifier les colonnes du WHERE**
```sql
-- Si votre WHERE utilise ces colonnes :
WHERE colonne1 = ? AND colonne2 > ?
-- Créez cet index :
CREATE INDEX idx_optim ON table(colonne1, colonne2);
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

⏭️
