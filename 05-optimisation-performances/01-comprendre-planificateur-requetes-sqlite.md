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

### Exemple simple

Prenons cette requête basique :
```sql
SELECT nom, age FROM employes WHERE age > 30;
```

Le planificateur peut choisir entre :
- **Option 1** : Regarder tous les employés un par un (scan complet)
- **Option 2** : Utiliser un index sur la colonne 'age' (si il existe)

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
SCAN e
SEARCH d USING PRIMARY KEY (nom=?)
```

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
CREATE INDEX idx_age ON employes(age);
```

**Nouveau plan :**
```
SEARCH employes USING INDEX idx_age (age>? AND age<?)
USE TEMP B-TREE FOR ORDER BY
```

**Amélioration :** Moins de lignes à trier grâce au filtre par index !

## Facteurs influençant le planificateur

### 1. Statistiques des tables

SQLite maintient des statistiques sur vos tables :
```sql
-- Voir les statistiques
PRAGMA table_info(employes);

-- Mettre à jour les statistiques
ANALYZE employes;
```

**Important :** Des statistiques à jour = meilleurs plans !

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

**Condition très sélective :** (peu de résultats)
```sql
SELECT * FROM employes WHERE id = 123;
-- → INDEX sera utilisé si disponible
```

**Condition peu sélective :** (beaucoup de résultats)
```sql
SELECT * FROM employes WHERE age IS NOT NULL;
-- → SCAN peut être plus rapide même avec un index
```

## Exemples pratiques d'optimisation

### Problème : Requête lente

```sql
-- Table avec 100 000 employés
CREATE TABLE employes_large AS
SELECT ... FROM ... ; -- Imaginez beaucoup de données

-- Requête lente
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
-- Plan : SCAN employes (index inutilisable)
```

**Bon :**
```sql
-- Stocker les noms en majuscules ou utiliser COLLATE NOCASE
SELECT * FROM employes WHERE nom = 'Alice' COLLATE NOCASE;
```

### 2. OR vs UNION

**Moins efficace :**
```sql
SELECT * FROM employes WHERE age < 25 OR age > 65;
-- Peut forcer un SCAN même avec index
```

**Plus efficace :**
```sql
SELECT * FROM employes WHERE age < 25
UNION
SELECT * FROM employes WHERE age > 65;
-- Peut utiliser l'index deux fois
```

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
```sql
.timer on
-- Votre requête avant optimisation
SELECT ... ;

-- Créer l'index
CREATE INDEX ... ;

-- Même requête après optimisation
SELECT ... ;
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

⏭️
