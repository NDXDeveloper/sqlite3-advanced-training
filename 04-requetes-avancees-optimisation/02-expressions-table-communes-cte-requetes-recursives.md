🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.2 Expressions de table communes (CTE) et requêtes récursives

## Introduction

Imaginez que vous devez construire une requête très complexe avec plusieurs sous-requêtes imbriquées. Votre code devient rapidement illisible et difficile à maintenir. Les **Expressions de Table Communes** (ou **CTE** pour "Common Table Expressions") sont comme des "variables temporaires" pour vos requêtes SQL. Elles vous permettent de diviser une requête complexe en plusieurs étapes plus simples et réutilisables.

Pensez aux CTE comme à des "brouillons" que vous pouvez nommer et réutiliser dans votre requête principale !

## Qu'est-ce qu'une CTE ?

Une **CTE** est une table temporaire nommée qui existe uniquement pendant l'exécution de votre requête. Elle se définit avec la clause `WITH` et peut être utilisée comme n'importe quelle table dans votre requête principale.

### Syntaxe de base

```sql
WITH nom_de_la_cte AS (
    -- Votre requête SELECT ici
)
SELECT * FROM nom_de_la_cte;
```

## Pourquoi utiliser les CTE ?

### ✅ **Avantages des CTE :**
- **Lisibilité** : Code plus clair et organisé
- **Réutilisation** : Une CTE peut être référencée plusieurs fois
- **Débogage** : Plus facile de tester chaque partie séparément
- **Maintenance** : Plus simple à modifier et comprendre
- **Requêtes récursives** : Seul moyen d'écrire des requêtes récursives en SQL

### Comparaison : Sous-requête vs CTE

**Avec sous-requête (difficile à lire) :**
```sql
SELECT
    titre,
    prix,
    prix - (SELECT AVG(prix) FROM livres) as diff_moyenne
FROM livres
WHERE auteur_id IN (
    SELECT A.id
    FROM auteurs A
    JOIN (
        SELECT auteur_id, AVG(prix) as prix_moyen
        FROM livres
        GROUP BY auteur_id
        HAVING AVG(prix) > 15
    ) stats ON A.id = stats.auteur_id
);
```

**Avec CTE (lisible et clair) :**
```sql
WITH auteurs_chers AS (
    SELECT auteur_id, AVG(prix) as prix_moyen
    FROM livres
    GROUP BY auteur_id
    HAVING AVG(prix) > 15
),
prix_moyen_global AS (
    SELECT AVG(prix) as moyenne FROM livres
)
SELECT
    L.titre,
    L.prix,
    L.prix - PMG.moyenne as diff_moyenne
FROM livres L
JOIN auteurs_chers AC ON L.auteur_id = AC.auteur_id
CROSS JOIN prix_moyen_global PMG;
```

## Préparation de la base de données

Créons une base de données plus riche pour nos exemples :

```sql
-- Tables de base
CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    parent_id INTEGER REFERENCES categories(id)
);

CREATE TABLE auteurs (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    pays TEXT,
    date_naissance DATE
);

CREATE TABLE livres (
    id INTEGER PRIMARY KEY,
    titre TEXT NOT NULL,
    prix REAL NOT NULL,
    auteur_id INTEGER REFERENCES auteurs(id),
    categorie_id INTEGER REFERENCES categories(id),
    stock INTEGER,
    date_publication DATE
);

CREATE TABLE clients (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    email TEXT UNIQUE,
    pays TEXT
);

CREATE TABLE commandes (
    id INTEGER PRIMARY KEY,
    client_id INTEGER REFERENCES clients(id),
    livre_id INTEGER REFERENCES livres(id),
    quantite INTEGER,
    date_commande DATE
);

-- Données d'exemple
INSERT INTO categories VALUES
(1, 'Littérature', NULL),
(2, 'Fiction', 1),
(3, 'Non-fiction', 1),
(4, 'Roman', 2),
(5, 'Poésie', 2),
(6, 'Histoire', 3),
(7, 'Biographie', 3);

INSERT INTO auteurs VALUES
(1, 'Victor Hugo', 'France', '1802-02-26'),
(2, 'Stephen King', 'USA', '1947-09-21'),
(3, 'Haruki Murakami', 'Japon', '1949-01-12'),
(4, 'Agatha Christie', 'Royaume-Uni', '1890-09-15');

INSERT INTO livres VALUES
(1, 'Les Misérables', 15.99, 1, 4, 5, '1862-03-30'),
(2, 'Notre-Dame de Paris', 12.50, 1, 4, 3, '1831-03-16'),
(3, 'Ça', 18.99, 2, 4, 8, '1986-09-15'),
(4, 'Shining', 16.75, 2, 4, 2, '1977-01-28'),
(5, 'Kafka sur le rivage', 14.99, 3, 4, 6, '2002-09-12'),
(6, 'Le Crime de lOrient-Express', 13.99, 4, 4, 4, '1934-01-01');

INSERT INTO clients VALUES
(1, 'Marie Dubois', 'marie@email.com', 'France'),
(2, 'John Smith', 'john@email.com', 'USA'),
(3, 'Yuki Tanaka', 'yuki@email.com', 'Japon');

INSERT INTO commandes VALUES
(1, 1, 1, 2, '2024-01-15'),
(2, 2, 3, 1, '2024-01-16'),
(3, 1, 5, 1, '2024-01-17'),
(4, 3, 2, 1, '2024-01-18'),
(5, 2, 4, 2, '2024-01-19'),
(6, 1, 6, 1, '2024-01-20');
```

## Types de CTE

### 1. CTE Simple (non-récursive)

#### Exemple 1 : Statistiques de base
```sql
-- Calculer les statistiques par auteur
WITH stats_auteurs AS (
    SELECT
        A.nom,
        COUNT(L.id) as nb_livres,
        AVG(L.prix) as prix_moyen,
        MIN(L.prix) as prix_min,
        MAX(L.prix) as prix_max
    FROM auteurs A
    LEFT JOIN livres L ON A.id = L.auteur_id
    GROUP BY A.id, A.nom
)
SELECT
    nom,
    nb_livres,
    ROUND(prix_moyen, 2) as prix_moyen,
    prix_min,
    prix_max,
    ROUND(prix_max - prix_min, 2) as ecart_prix
FROM stats_auteurs
WHERE nb_livres > 0
ORDER BY prix_moyen DESC;
```

#### Exemple 2 : Analyses de ventes
```sql
-- Analyser les ventes par client et par mois
WITH ventes_mensuelles AS (
    SELECT
        C.client_id,
        CL.nom as nom_client,
        strftime('%Y-%m', C.date_commande) as mois,
        SUM(L.prix * C.quantite) as total_mois,
        COUNT(C.id) as nb_commandes
    FROM commandes C
    JOIN livres L ON C.livre_id = L.id
    JOIN clients CL ON C.client_id = CL.id
    GROUP BY C.client_id, CL.nom, strftime('%Y-%m', C.date_commande)
),
moyennes_clients AS (
    SELECT
        client_id,
        nom_client,
        AVG(total_mois) as moyenne_mensuelle,
        SUM(total_mois) as total_client
    FROM ventes_mensuelles
    GROUP BY client_id, nom_client
)
SELECT
    VM.nom_client,
    VM.mois,
    VM.total_mois,
    MC.moyenne_mensuelle,
    CASE
        WHEN VM.total_mois > MC.moyenne_mensuelle THEN '📈 Au-dessus'
        ELSE '📉 En-dessous'
    END as tendance
FROM ventes_mensuelles VM
JOIN moyennes_clients MC ON VM.client_id = MC.client_id
ORDER BY VM.nom_client, VM.mois;
```

### 2. CTE Multiples

Vous pouvez définir plusieurs CTE dans une seule requête :

```sql
-- Analyse complète : auteurs, livres et ventes
WITH auteurs_productifs AS (
    SELECT
        A.id,
        A.nom,
        COUNT(L.id) as nb_livres
    FROM auteurs A
    LEFT JOIN livres L ON A.id = L.auteur_id
    GROUP BY A.id, A.nom
    HAVING COUNT(L.id) >= 2
),
livres_populaires AS (
    SELECT
        L.id,
        L.titre,
        L.auteur_id,
        COUNT(C.id) as nb_commandes,
        SUM(C.quantite) as total_vendus
    FROM livres L
    LEFT JOIN commandes C ON L.id = C.livre_id
    GROUP BY L.id, L.titre, L.auteur_id
),
classement_final AS (
    SELECT
        AP.nom as auteur,
        LP.titre,
        LP.nb_commandes,
        LP.total_vendus,
        RANK() OVER (PARTITION BY AP.id ORDER BY LP.total_vendus DESC) as rang_auteur
    FROM auteurs_productifs AP
    JOIN livres_populaires LP ON AP.id = LP.auteur_id
)
SELECT
    auteur,
    titre,
    total_vendus,
    CASE rang_auteur
        WHEN 1 THEN '🥇 Best-seller'
        WHEN 2 THEN '🥈 Second'
        ELSE '🥉 Autre'
    END as statut
FROM classement_final
WHERE total_vendus > 0
ORDER BY auteur, rang_auteur;
```

## Requêtes récursives avec CTE

Les **requêtes récursives** permettent de traiter des données **hiérarchiques** (comme des arbres ou des structures parent-enfant). SQLite supporte les CTE récursives depuis la version 3.8.3.

### Syntaxe des CTE récursives

```sql
WITH RECURSIVE nom_cte AS (
    -- PARTIE 1: Cas de base (ancrage)
    SELECT ...

    UNION ALL

    -- PARTIE 2: Partie récursive
    SELECT ... FROM nom_cte ...
)
SELECT * FROM nom_cte;
```

### Exemple 1 : Hiérarchie de catégories

```sql
-- Afficher toute la hiérarchie des catégories
WITH RECURSIVE hierarchie_categories AS (
    -- Cas de base : catégories racine (sans parent)
    SELECT
        id,
        nom,
        parent_id,
        0 as niveau,
        nom as chemin
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Partie récursive : enfants des catégories déjà trouvées
    SELECT
        c.id,
        c.nom,
        c.parent_id,
        hc.niveau + 1,
        hc.chemin || ' > ' || c.nom as chemin
    FROM categories c
    JOIN hierarchie_categories hc ON c.parent_id = hc.id
)
SELECT
    CASE niveau
        WHEN 0 THEN nom
        WHEN 1 THEN '  └─ ' || nom
        WHEN 2 THEN '    └─ ' || nom
        ELSE REPEAT('  ', niveau) || '└─ ' || nom
    END as arbre,
    niveau,
    chemin
FROM hierarchie_categories
ORDER BY chemin;
```

**Résultat attendu :**
```
arbre                niveau  chemin
------------------   ------  -------------------------
Littérature         0       Littérature
  └─ Fiction        1       Littérature > Fiction
    └─ Roman        2       Littérature > Fiction > Roman
    └─ Poésie       2       Littérature > Fiction > Poésie
  └─ Non-fiction    1       Littérature > Non-fiction
    └─ Histoire     2       Littérature > Non-fiction > Histoire
    └─ Biographie   2       Littérature > Non-fiction > Biographie
```

### Exemple 2 : Séquence de nombres

```sql
-- Générer une séquence de 1 à 10
WITH RECURSIVE sequence AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM sequence WHERE n < 10
)
SELECT n FROM sequence;
```

### Exemple 3 : Analyse temporelle récursive

```sql
-- Générer tous les mois entre deux dates
WITH RECURSIVE mois_periode AS (
    SELECT
        DATE('2024-01-01') as date_mois,
        DATE('2024-12-31') as date_fin

    UNION ALL

    SELECT
        DATE(date_mois, '+1 month'),
        date_fin
    FROM mois_periode
    WHERE DATE(date_mois, '+1 month') <= date_fin
),
ventes_par_mois AS (
    SELECT
        strftime('%Y-%m', C.date_commande) as mois,
        SUM(L.prix * C.quantite) as total_ventes,
        COUNT(C.id) as nb_commandes
    FROM commandes C
    JOIN livres L ON C.livre_id = L.id
    GROUP BY strftime('%Y-%m', C.date_commande)
)
SELECT
    strftime('%Y-%m', MP.date_mois) as mois,
    COALESCE(VPM.total_ventes, 0) as ventes,
    COALESCE(VPM.nb_commandes, 0) as commandes,
    CASE
        WHEN VPM.total_ventes IS NULL THEN '❌ Aucune vente'
        WHEN VPM.total_ventes > 50 THEN '🔥 Excellent'
        WHEN VPM.total_ventes > 20 THEN '✅ Bon'
        ELSE '⚠️ Faible'
    END as performance
FROM mois_periode MP
LEFT JOIN ventes_par_mois VPM ON strftime('%Y-%m', MP.date_mois) = VPM.mois
ORDER BY MP.date_mois;
```

## Cas d'usage avancés

### 1. Calculs de moyennes mobiles

```sql
-- Moyenne mobile des ventes sur 3 mois
WITH ventes_chronologiques AS (
    SELECT
        strftime('%Y-%m', date_commande) as mois,
        SUM(L.prix * C.quantite) as ventes_mois
    FROM commandes C
    JOIN livres L ON C.livre_id = L.id
    GROUP BY strftime('%Y-%m', date_commande)
    ORDER BY mois
),
avec_numero_ligne AS (
    SELECT
        *,
        ROW_NUMBER() OVER (ORDER BY mois) as numero_ligne
    FROM ventes_chronologiques
)
SELECT
    V1.mois,
    V1.ventes_mois,
    ROUND(AVG(V2.ventes_mois), 2) as moyenne_mobile_3mois
FROM avec_numero_ligne V1
JOIN avec_numero_ligne V2 ON V2.numero_ligne BETWEEN V1.numero_ligne - 2 AND V1.numero_ligne
GROUP BY V1.mois, V1.ventes_mois, V1.numero_ligne
ORDER BY V1.mois;
```

### 2. Analyse de cohortes simples

```sql
-- Analyser les comportements d'achat par cohorte de clients
WITH premiere_commande AS (
    SELECT
        client_id,
        MIN(date_commande) as premiere_date,
        strftime('%Y-%m', MIN(date_commande)) as cohorte
    FROM commandes
    GROUP BY client_id
),
activite_cohorte AS (
    SELECT
        PC.cohorte,
        COUNT(DISTINCT PC.client_id) as taille_cohorte,
        COUNT(DISTINCT C.client_id) as clients_actifs_mois,
        strftime('%Y-%m', C.date_commande) as mois_activite
    FROM premiere_commande PC
    LEFT JOIN commandes C ON PC.client_id = C.client_id
    GROUP BY PC.cohorte, strftime('%Y-%m', C.date_commande)
)
SELECT
    cohorte,
    mois_activite,
    taille_cohorte,
    clients_actifs_mois,
    ROUND(
        (clients_actifs_mois * 100.0 / taille_cohorte), 1
    ) as retention_pourcent
FROM activite_cohorte
WHERE mois_activite IS NOT NULL
ORDER BY cohorte, mois_activite;
```

## Bonnes pratiques et optimisation

### ✅ **Bonnes pratiques :**

1. **Noms explicites :**
```sql
-- ✅ BON
WITH clients_fideles AS (...)

-- ❌ MAUVAIS
WITH cte1 AS (...)
```

2. **Une responsabilité par CTE :**
```sql
-- ✅ BON : Chaque CTE a un rôle clair
WITH ventes_par_mois AS (...),
     moyennes_mobiles AS (...),
     comparaisons AS (...)

-- ❌ MAUVAIS : CTE qui fait tout
WITH calculs_complexes AS (
    -- 50 lignes de logique différente
)
```

3. **Commentaires pour les CTE complexes :**
```sql
WITH RECURSIVE hierarchie_categories AS (
    -- Étape 1 : Trouver les catégories racine
    SELECT ...

    UNION ALL

    -- Étape 2 : Descendre d'un niveau à chaque itération
    SELECT ...
)
```

### ⚠️ **Limitations et attention :**

1. **Performance des CTE récursives :**
```sql
-- Limitez toujours la récursion pour éviter les boucles infinies
WITH RECURSIVE suite AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM suite WHERE n < 1000  -- ⚠️ Toujours une condition d'arrêt !
)
```

2. **Réutilisation vs Performance :**
```sql
-- Si une CTE n'est utilisée qu'une fois, parfois une sous-requête est plus efficace
WITH simple_calcul AS (
    SELECT AVG(prix) as moyenne FROM livres
)
SELECT titre FROM livres, simple_calcul WHERE prix > moyenne;

-- Peut être plus simple comme :
SELECT titre FROM livres WHERE prix > (SELECT AVG(prix) FROM livres);
```

## Exercices pratiques

### Exercice 1 (Débutant)
Utilisez une CTE pour trouver tous les livres d'auteurs qui ont publié plus de 1 livre, en affichant le titre, l'auteur et le nombre total de livres de cet auteur.

<details>
<summary>Solution</summary>

```sql
WITH auteurs_prolifiques AS (
    SELECT
        A.id,
        A.nom,
        COUNT(L.id) as nb_livres
    FROM auteurs A
    JOIN livres L ON A.id = L.auteur_id
    GROUP BY A.id, A.nom
    HAVING COUNT(L.id) > 1
)
SELECT
    L.titre,
    AP.nom as auteur,
    AP.nb_livres
FROM livres L
JOIN auteurs_prolifiques AP ON L.auteur_id = AP.id
ORDER BY AP.nom, L.titre;
```
</details>

### Exercice 2 (Intermédiaire)
Créez une CTE récursive pour générer la suite de Fibonacci jusqu'au 10ème terme.

<details>
<summary>Solution</summary>

```sql
WITH RECURSIVE fibonacci AS (
    -- Cas de base : F(0) = 0, F(1) = 1
    SELECT 0 as position, 0 as valeur
    UNION ALL
    SELECT 1, 1

    UNION ALL

    -- Cas récursif : F(n) = F(n-1) + F(n-2)
    SELECT
        f1.position + 1,
        f1.valeur + f2.valeur
    FROM fibonacci f1
    JOIN fibonacci f2 ON f2.position = f1.position - 1
    WHERE f1.position < 10
    AND f1.position = (
        SELECT MAX(position) FROM fibonacci
    )
)
SELECT position, valeur FROM fibonacci ORDER BY position;
```
</details>

### Exercice 3 (Avancé)
Créez une analyse complète des ventes avec plusieurs CTE qui calcule :
- Les ventes par mois
- La croissance par rapport au mois précédent
- Le classement des mois par performance
- La moyenne mobile sur 3 mois

<details>
<summary>Solution</summary>

```sql
WITH ventes_mensuelles AS (
    SELECT
        strftime('%Y-%m', C.date_commande) as mois,
        SUM(L.prix * C.quantite) as total_ventes,
        COUNT(C.id) as nb_commandes
    FROM commandes C
    JOIN livres L ON C.livre_id = L.id
    GROUP BY strftime('%Y-%m', C.date_commande)
),
avec_precedent AS (
    SELECT
        *,
        LAG(total_ventes) OVER (ORDER BY mois) as ventes_mois_precedent
    FROM ventes_mensuelles
),
avec_croissance AS (
    SELECT
        *,
        CASE
            WHEN ventes_mois_precedent IS NULL THEN NULL
            ELSE ROUND(
                ((total_ventes - ventes_mois_precedent) * 100.0 / ventes_mois_precedent), 1
            )
        END as croissance_pourcent
    FROM avec_precedent
),
avec_classement AS (
    SELECT
        *,
        RANK() OVER (ORDER BY total_ventes DESC) as rang_performance
    FROM avec_croissance
),
avec_moyenne_mobile AS (
    SELECT
        AC1.*,
        ROUND(AVG(AC2.total_ventes), 2) as moyenne_mobile_3mois
    FROM avec_classement AC1
    JOIN avec_classement AC2 ON AC2.mois BETWEEN
        date(AC1.mois || '-01', '-2 months') AND AC1.mois || '-01'
    GROUP BY AC1.mois, AC1.total_ventes, AC1.nb_commandes,
             AC1.croissance_pourcent, AC1.rang_performance
)
SELECT
    mois,
    total_ventes,
    nb_commandes,
    croissance_pourcent || '%' as croissance,
    rang_performance,
    moyenne_mobile_3mois,
    CASE
        WHEN rang_performance = 1 THEN '🏆 Meilleur mois'
        WHEN croissance_pourcent > 0 THEN '📈 En croissance'
        WHEN croissance_pourcent < 0 THEN '📉 En baisse'
        ELSE '➡️ Premier mois'
    END as tendance
FROM avec_moyenne_mobile
ORDER BY mois;
```
</details>

## Résumé

Les **CTE (Common Table Expressions)** sont un outil puissant qui transforme votre façon d'écrire du SQL :

### 🎯 **Avantages principaux :**
- **Lisibilité** : Code structuré et facile à comprendre
- **Réutilisation** : Une CTE peut être référencée plusieurs fois
- **Débogage** : Test facile de chaque partie séparément
- **Récursion** : Seul moyen de traiter des données hiérarchiques

### 📋 **Types de CTE :**
- **Simples** : Pour décomposer des requêtes complexes
- **Multiples** : Pour créer des pipelines de transformation
- **Récursives** : Pour les données hiérarchiques et séquences

### ⚡ **Quand les utiliser :**
- Requêtes avec plusieurs niveaux de sous-requêtes
- Calculs intermédiaires réutilisés
- Données hiérarchiques (arbres, catégories)
- Analyses temporelles complexes
- Amélioration de la lisibilité du code

### 🚨 **Points d'attention :**
- Toujours limiter les CTE récursives
- Préférer les jointures pour les cas simples
- Nommer clairement vos CTE
- Une responsabilité par CTE

Les CTE sont particulièrement utiles pour l'**analyse de données** et les **rapports complexes**. Elles vous permettent de penser votre requête comme une série d'étapes logiques plutôt que comme une requête monolithique !

Dans la prochaine section, nous découvrirons les **fonctions de fenêtrage**, qui complètent parfaitement les CTE pour des analyses encore plus sophistiquées.


⏭️
