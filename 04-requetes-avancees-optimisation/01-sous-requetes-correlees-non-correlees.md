🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.1 Sous-requêtes corrélées et non-corrélées

## Introduction

Les sous-requêtes sont l'un des concepts les plus puissants de SQL. Elles permettent de résoudre des problèmes complexes en décomposant une question difficile en plusieurs étapes plus simples. Imaginez que vous voulez trouver tous les clients qui ont dépensé plus que la moyenne : vous devez d'abord calculer la moyenne, puis comparer chaque client à cette moyenne. Les sous-requêtes permettent de faire cela en une seule requête !

## Qu'est-ce qu'une sous-requête ?

Une **sous-requête** (ou requête imbriquée) est une requête SQL placée à l'intérieur d'une autre requête SQL. Elle peut apparaître dans les clauses SELECT, WHERE, HAVING, ou FROM.

### Exemple simple pour commencer

```sql
-- Trouver tous les livres plus chers que le prix moyen
SELECT titre, prix
FROM livres
WHERE prix > (SELECT AVG(prix) FROM livres);
```

Dans cet exemple, `(SELECT AVG(prix) FROM livres)` est la sous-requête qui calcule le prix moyen.

## Les deux types de sous-requêtes

### 1. Sous-requêtes non-corrélées (indépendantes)

Une sous-requête **non-corrélée** peut être exécutée indépendamment de la requête principale. Elle ne fait pas référence aux colonnes de la requête externe.

#### Caractéristiques :
- ✅ Exécutée **une seule fois**
- ✅ Résultat utilisé par la requête principale
- ✅ Plus rapide en général
- ✅ Plus facile à comprendre et déboguer

#### Exemple détaillé :

```sql
-- Créons d'abord notre base de données d'exemple
CREATE TABLE livres (
    id INTEGER PRIMARY KEY,
    titre TEXT NOT NULL,
    prix REAL NOT NULL,
    auteur_id INTEGER,
    stock INTEGER
);

CREATE TABLE auteurs (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    pays TEXT
);

-- Insérons quelques données
INSERT INTO auteurs VALUES
(1, 'Victor Hugo', 'France'),
(2, 'Stephen King', 'USA'),
(3, 'Haruki Murakami', 'Japon');

INSERT INTO livres VALUES
(1, 'Les Misérables', 15.99, 1, 5),
(2, 'Notre-Dame de Paris', 12.50, 1, 3),
(3, 'Ça', 18.99, 2, 8),
(4, 'Shining', 16.75, 2, 2),
(5, 'Kafka sur le rivage', 14.99, 3, 6);
```

**Exemple 1 : Livres plus chers que la moyenne**
```sql
SELECT titre, prix
FROM livres
WHERE prix > (
    SELECT AVG(prix)
    FROM livres
);
```
*Résultat : Ça (18.99€) et Shining (16.75€)*

**Exemple 2 : Livres des auteurs français**
```sql
SELECT titre, prix
FROM livres
WHERE auteur_id IN (
    SELECT id
    FROM auteurs
    WHERE pays = 'France'
);
```
*Résultat : Les Misérables et Notre-Dame de Paris*

### 2. Sous-requêtes corrélées (dépendantes)

Une sous-requête **corrélée** fait référence aux colonnes de la requête externe. Elle doit être exécutée pour chaque ligne de la requête principale.

#### Caractéristiques :
- ⚠️ Exécutée **pour chaque ligne** de la requête externe
- ⚠️ Plus lente en général
- ⚠️ Plus complexe à comprendre
- ✅ Permet des comparaisons ligne par ligne très puissantes

#### Exemple détaillé :

```sql
-- Ajoutons une table de commandes pour nos exemples
CREATE TABLE commandes (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,
    livre_id INTEGER,
    quantite INTEGER,
    date_commande DATE
);

INSERT INTO commandes VALUES
(1, 101, 1, 2, '2024-01-15'),
(2, 102, 3, 1, '2024-01-16'),
(3, 101, 5, 3, '2024-01-17'),
(4, 103, 2, 1, '2024-01-18'),
(5, 102, 4, 2, '2024-01-19');
```

**Exemple 1 : Livres qui ont été commandés**
```sql
SELECT titre, prix
FROM livres L
WHERE EXISTS (
    SELECT 1
    FROM commandes C
    WHERE C.livre_id = L.id  -- Référence à la table externe !
);
```
*Notice : `L.id` dans la sous-requête fait référence à la requête externe*

**Exemple 2 : Auteurs avec le livre le plus cher par auteur**
```sql
SELECT nom
FROM auteurs A
WHERE EXISTS (
    SELECT 1
    FROM livres L
    WHERE L.auteur_id = A.id
    AND L.prix = (
        SELECT MAX(prix)
        FROM livres L2
        WHERE L2.auteur_id = A.id
    )
);
```

## Différences visuelles pour mieux comprendre

### Sous-requête non-corrélée :
```
1. SQLite exécute la sous-requête → Résultat : 15.84 (prix moyen)
2. SQLite utilise ce résultat dans la requête principale
3. Une seule exécution de la sous-requête !

SELECT titre FROM livres WHERE prix > (SELECT AVG(prix) FROM livres);
                                      ↑
                                Exécutée 1 fois
```

### Sous-requête corrélée :
```
Pour chaque livre dans la requête principale :
1. Livre "Les Misérables" → Exécute la sous-requête pour ce livre
2. Livre "Notre-Dame" → Exécute la sous-requête pour ce livre
3. Livre "Ça" → Exécute la sous-requête pour ce livre
...

SELECT titre FROM livres L WHERE EXISTS (SELECT 1 FROM commandes WHERE livre_id = L.id);
                                         ↑
                                    Exécutée pour chaque ligne !
```

## Opérateurs couramment utilisés avec les sous-requêtes

### 1. **IN / NOT IN**
```sql
-- Livres des auteurs américains
SELECT titre
FROM livres
WHERE auteur_id IN (
    SELECT id FROM auteurs WHERE pays = 'USA'
);

-- Livres qui ne sont PAS des auteurs français
SELECT titre
FROM livres
WHERE auteur_id NOT IN (
    SELECT id FROM auteurs WHERE pays = 'France'
);
```

### 2. **EXISTS / NOT EXISTS**
```sql
-- Auteurs qui ont publié des livres
SELECT nom
FROM auteurs A
WHERE EXISTS (
    SELECT 1 FROM livres L WHERE L.auteur_id = A.id
);

-- Livres jamais commandés
SELECT titre
FROM livres L
WHERE NOT EXISTS (
    SELECT 1 FROM commandes C WHERE C.livre_id = L.id
);
```

### 3. **Comparateurs avec sous-requêtes**
```sql
-- Livre le plus cher
SELECT titre, prix
FROM livres
WHERE prix = (SELECT MAX(prix) FROM livres);

-- Livres plus chers que tous les livres de Victor Hugo
SELECT titre, prix
FROM livres
WHERE prix > ALL (
    SELECT prix FROM livres L
    JOIN auteurs A ON L.auteur_id = A.id
    WHERE A.nom = 'Victor Hugo'
);

-- Livres plus chers qu'au moins un livre de Stephen King
SELECT titre, prix
FROM livres
WHERE prix > ANY (
    SELECT prix FROM livres L
    JOIN auteurs A ON L.auteur_id = A.id
    WHERE A.nom = 'Stephen King'
);
```

## Sous-requêtes dans différentes clauses

### Dans la clause SELECT
```sql
-- Afficher le prix et la différence avec le prix moyen
SELECT
    titre,
    prix,
    prix - (SELECT AVG(prix) FROM livres) AS difference_moyenne
FROM livres;
```

### Dans la clause FROM
```sql
-- Sous-requête utilisée comme table temporaire
SELECT
    pays,
    AVG(prix_moyen_auteur) as prix_moyen_pays
FROM (
    SELECT
        A.pays,
        A.nom,
        AVG(L.prix) as prix_moyen_auteur
    FROM auteurs A
    JOIN livres L ON A.id = L.auteur_id
    GROUP BY A.id, A.nom, A.pays
) AS stats_auteurs
GROUP BY pays;
```

### Dans la clause HAVING
```sql
-- Auteurs dont le prix moyen est supérieur au prix moyen global
SELECT
    A.nom,
    AVG(L.prix) as prix_moyen
FROM auteurs A
JOIN livres L ON A.id = L.auteur_id
GROUP BY A.id, A.nom
HAVING AVG(L.prix) > (
    SELECT AVG(prix) FROM livres
);
```

## Exemples pratiques du monde réel

### Exemple 1 : Analyse des ventes
```sql
-- Clients qui ont dépensé plus que la moyenne
SELECT
    client_id,
    SUM(L.prix * C.quantite) as total_depense
FROM commandes C
JOIN livres L ON C.livre_id = L.id
GROUP BY client_id
HAVING SUM(L.prix * C.quantite) > (
    SELECT AVG(total_par_client)
    FROM (
        SELECT SUM(L2.prix * C2.quantite) as total_par_client
        FROM commandes C2
        JOIN livres L2 ON C2.livre_id = L2.id
        GROUP BY client_id
    )
);
```

### Exemple 2 : Gestion des stocks
```sql
-- Livres avec un stock inférieur à la moyenne ET qui ont des commandes
SELECT titre, stock
FROM livres L
WHERE stock < (SELECT AVG(stock) FROM livres)
AND EXISTS (
    SELECT 1 FROM commandes C WHERE C.livre_id = L.id
);
```

## Pièges à éviter

### 1. **Attention avec NULL et NOT IN**
```sql
-- ❌ PROBLÈME : Si un auteur a un pays NULL, cette requête ne retournera RIEN
SELECT titre
FROM livres
WHERE auteur_id NOT IN (
    SELECT id FROM auteurs WHERE pays = 'France'  -- Si un id est NULL...
);

-- ✅ SOLUTION : Gérer les NULL explicitement
SELECT titre
FROM livres
WHERE auteur_id NOT IN (
    SELECT id FROM auteurs WHERE pays = 'France' AND id IS NOT NULL
);
```

### 2. **Performance des sous-requêtes corrélées**
```sql
-- ❌ LENT : Sous-requête corrélée qui peut être évitée
SELECT titre
FROM livres L
WHERE EXISTS (
    SELECT 1 FROM auteurs A WHERE A.id = L.auteur_id AND A.pays = 'France'
);

-- ✅ PLUS RAPIDE : Utiliser une jointure
SELECT L.titre
FROM livres L
JOIN auteurs A ON L.auteur_id = A.id
WHERE A.pays = 'France';
```

## Quand utiliser chaque type ?

### Utilisez les sous-requêtes non-corrélées quand :
- Vous avez besoin d'une valeur calculée (moyenne, maximum, etc.)
- La sous-requête peut être exécutée indépendamment
- Vous voulez filtrer sur une liste de valeurs

### Utilisez les sous-requêtes corrélées quand :
- Vous avez besoin de comparaisons ligne par ligne
- Vous voulez tester l'existence d'enregistrements liés
- Les jointures ne permettent pas d'exprimer facilement votre logique

## Exercices pratiques

### Exercice 1 (Débutant)
Trouvez tous les livres dont le prix est supérieur au prix du livre le plus cher de Victor Hugo.

<details>
<summary>Solution</summary>

```sql
SELECT titre, prix
FROM livres
WHERE prix > (
    SELECT MAX(L.prix)
    FROM livres L
    JOIN auteurs A ON L.auteur_id = A.id
    WHERE A.nom = 'Victor Hugo'
);
```
</details>

### Exercice 2 (Intermédiaire)
Trouvez les auteurs qui ont écrit au moins un livre plus cher que 15€.

<details>
<summary>Solution</summary>

```sql
SELECT DISTINCT nom
FROM auteurs A
WHERE EXISTS (
    SELECT 1
    FROM livres L
    WHERE L.auteur_id = A.id
    AND L.prix > 15
);
```
</details>

### Exercice 3 (Avancé)
Pour chaque auteur, affichez son nom et le nombre de ses livres qui sont plus chers que le prix moyen de TOUS les livres.

<details>
<summary>Solution</summary>

```sql
SELECT
    A.nom,
    (SELECT COUNT(*)
     FROM livres L
     WHERE L.auteur_id = A.id
     AND L.prix > (SELECT AVG(prix) FROM livres)
    ) as livres_chers
FROM auteurs A;
```
</details>

## Résumé

Les sous-requêtes sont un outil puissant qui vous permet de :
- 🎯 **Décomposer** des problèmes complexes en étapes simples
- 🔍 **Filtrer** des données avec des critères sophistiqués
- 📊 **Calculer** des statistiques et les utiliser dans vos requêtes
- 🔗 **Comparer** des données ligne par ligne

**Points clés à retenir :**
- Les sous-requêtes non-corrélées sont plus rapides mais moins flexibles
- Les sous-requêtes corrélées sont plus puissantes mais plus coûteuses
- Attention aux valeurs NULL avec NOT IN
- Parfois une jointure est plus efficace qu'une sous-requête corrélée

Dans la prochaine section, nous découvrirons les **Expressions de Table Communes (CTE)**, qui offrent une alternative plus lisible et parfois plus performante aux sous-requêtes complexes !


⏭️
