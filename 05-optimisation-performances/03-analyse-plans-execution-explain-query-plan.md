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

```sql
EXPLAIN QUERY PLAN
SELECT colonne FROM table WHERE condition;
```

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
| **SCAN** | Lecture complète de la table | 🐌 Lent sur grandes tables |
| **SEARCH** | Utilisation d'un index | 🚀 Rapide |
| **USING INDEX** | Précise quel index est utilisé | ✅ Très bon signe |
| **USING PRIMARY KEY** | Utilise la clé primaire | ⭐ Optimal |
| **TEMP B-TREE** | Structure temporaire créée | ⚠️ Coûteux en mémoire |

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
SCAN e
SEARCH d USING PRIMARY KEY (nom=?)
```

**Interprétation :**
1. SQLite lit chaque employé (`SCAN e`)
2. Pour chaque employé, cherche le département correspondant en utilisant la clé primaire (`SEARCH d`)
3. ✅ **Efficace** grâce à la clé primaire sur `departements.nom`

### Jointure problématique

```sql
-- Ajoutons une table sans clé appropriée
CREATE TABLE projets (
    id INTEGER PRIMARY KEY,
    nom_projet TEXT,
    responsable TEXT,  -- Pas de clé étrangère !
    budget INTEGER
);

EXPLAIN QUERY PLAN
SELECT e.nom, p.nom_projet
FROM employes e
JOIN projets p ON e.nom = p.responsable;
```

**Résultat problématique :**
```
SCAN e
SCAN p
```

**Interprétation :**
- SQLite doit scanner les deux tables complètement
- ⚠️ **Très lent !** (produit cartésien puis filtrage)

**Solution :**
```sql
-- Créer un index sur la colonne de jointure
CREATE INDEX idx_responsable ON projets(responsable);

-- Maintenant le plan est meilleur :
-- SCAN e
-- SEARCH p USING INDEX idx_responsable (responsable=?)
```

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

```sql
-- ❌ Jointure lente
EXPLAIN QUERY PLAN
SELECT e.nom, d.budget
FROM employes e, departements d
WHERE e.departement = d.nom AND d.budget > 50000;
-- Résultat : SCAN e + SCAN d (produit cartésien !)

-- ✅ Meilleure syntaxe
EXPLAIN QUERY PLAN
SELECT e.nom, d.budget
FROM employes e
JOIN departements d ON e.departement = d.nom
WHERE d.budget > 50000;
-- Résultat : SCAN e + SEARCH d USING PRIMARY KEY
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

### Informations de coût

Parfois, SQLite affiche des estimations de coût :
```
SEARCH employes USING INDEX idx_age (age>? AND age<?) (~10 rows)
```

**Interprétation :**
- `~10 rows` : SQLite estime récupérer environ 10 lignes
- Plus le nombre est petit, plus l'opération est sélective (bien !)

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
SCAN ventes
USE TEMP B-TREE FOR GROUP BY
USE TEMP B-TREE FOR ORDER BY
```

**Problèmes identifiés :**
1. `SCAN` complet de la table
2. Deux structures temporaires (GROUP BY et ORDER BY)

**Solution d'optimisation :**
```sql
-- Index composite pour le WHERE
CREATE INDEX idx_date_region ON ventes(date_vente, region);

-- Index pour optimiser le GROUP BY
CREATE INDEX idx_vendeur ON ventes(vendeur);
```

**Résultat après optimisation :**
```
SEARCH ventes USING INDEX idx_date_region (date_vente>? AND region=?)
USE TEMP B-TREE FOR GROUP BY
USE TEMP B-TREE FOR ORDER BY
```

**Amélioration :** Le SCAN devient SEARCH → beaucoup plus rapide !

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
SEARCH employes USING INDEX idx_nom
```

**Amélioration :** Plus de tri temporaire, accès direct aux bonnes lignes !

## Outils complementaires pour l'analyse

### 1. Mesurer le temps d'exécution

```sql
-- Activer le timer
.timer on

-- Comparer avant/après optimisation
SELECT COUNT(*) FROM employes WHERE departement = 'IT';
-- Temps affiché automatiquement
```

### 2. Analyser l'utilisation mémoire

```sql
-- Statistiques détaillées
.stats on

-- Votre requête
SELECT * FROM employes ORDER BY salaire DESC;

-- Affiche l'usage mémoire et les pages lues
```

### 3. Forcer la mise à jour des statistiques

```sql
-- SQLite utilise des statistiques pour optimiser
-- Les mettre à jour après de gros changements :
ANALYZE employes;

-- Ou toute la base :
ANALYZE;
```

## Guide de diagnostic rapide

### Checklist pour analyser un plan

1. **Cherchez les mots-clés problématiques :**
   - `SCAN` sur grande table → créer un index
   - `TEMP B-TREE` répétés → optimiser ORDER BY/GROUP BY
   - `SCAN` dans les deux tables d'un JOIN → ajouter index sur clé de jointure

2. **Vérifiez la sélectivité :**
   - `(~10 rows)` = très sélectif = bon
   - `(~10000 rows)` = peu sélectif = problématique

3. **Analysez l'ordre des opérations :**
   - Les filtres (`WHERE`) doivent être appliqués tôt
   - Les tris (`ORDER BY`) devraient utiliser des index

### Exemples de plans optimaux

```sql
-- ✅ Excellent plan
SEARCH employes USING INDEX idx_dept_age (departement=? AND age>?)

-- ✅ Bon plan avec jointure
SCAN e USING INDEX idx_dept
SEARCH d USING PRIMARY KEY (nom=?)

-- ✅ Optimal pour tri
SCAN employes USING INDEX idx_salaire_desc
```

### Exemples de plans problématiques

```sql
-- ❌ Problématique
SCAN employes (sur table > 10000 lignes)

-- ❌ Très problématique
SCAN table1
SCAN table2 (jointure sans index)

-- ❌ Gourmand en mémoire
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

⏭️
