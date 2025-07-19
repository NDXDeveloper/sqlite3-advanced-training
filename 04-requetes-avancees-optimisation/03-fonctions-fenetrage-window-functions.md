🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.3 Fonctions de fenêtrage (WINDOW functions)

## Introduction

Imaginez que vous voulez classer vos livres par prix dans chaque catégorie, ou calculer la différence entre le prix d'un livre et celui du livre précédent dans la liste. Avec les fonctions traditionnelles, vous devriez faire des jointures complexes ou plusieurs requêtes. Les **fonctions de fenêtrage** (WINDOW functions) permettent de faire ces calculs **en une seule requête** !

Pensez aux fonctions de fenêtrage comme à une "loupe magique" qui peut regarder autour de chaque ligne pour faire des comparaisons et des calculs sophistiqués.

## Qu'est-ce qu'une fonction de fenêtrage ?

Une **fonction de fenêtrage** effectue un calcul sur un ensemble de lignes liées à la ligne actuelle, sans avoir besoin de regrouper les données (contrairement à GROUP BY). Elle "regarde" à travers une "fenêtre" de données autour de chaque ligne.

### Différence clé avec GROUP BY

```sql
-- GROUP BY : Résume les données (moins de lignes en sortie)
SELECT categorie_id, AVG(prix) as prix_moyen
FROM livres
GROUP BY categorie_id;
-- Résultat : 1 ligne par catégorie

-- WINDOW : Garde toutes les lignes + ajoute des calculs
SELECT
    titre,
    prix,
    categorie_id,
    AVG(prix) OVER (PARTITION BY categorie_id) as prix_moyen_categorie
FROM livres;
-- Résultat : Toutes les lignes + la moyenne par catégorie sur chaque ligne
```

## Syntaxe de base

```sql
SELECT
    colonnes,
    fonction_window() OVER (
        [PARTITION BY colonne(s)]
        [ORDER BY colonne(s)]
        [ROWS/RANGE BETWEEN ... AND ...]
    ) as nom_calcul
FROM table;
```

### Composants de la clause OVER :

- **PARTITION BY** : Divise les données en groupes (comme GROUP BY, mais sans résumer)
- **ORDER BY** : Définit l'ordre dans chaque partition
- **ROWS/RANGE** : Définit la "taille de la fenêtre" (optionnel)

## Préparation des données

Enrichissons notre base de données pour les exemples :

```sql
-- Ajoutons plus de données réalistes
INSERT INTO livres VALUES
(7, 'Carrie', 14.50, 2, 4, 7, '1974-04-05'),
(8, 'Misery', 17.25, 2, 4, 3, '1987-06-08'),
(9, 'La Peste', 13.75, 1, 4, 4, '1947-06-10'),
(10, 'Létranger', 11.99, 1, 4, 6, '1942-05-19');

INSERT INTO commandes VALUES
(7, 1, 7, 1, '2024-01-21'),
(8, 2, 8, 2, '2024-01-22'),
(9, 3, 9, 1, '2024-01-23'),
(10, 1, 10, 1, '2024-01-24'),
(11, 2, 1, 1, '2024-02-01'),
(12, 3, 3, 2, '2024-02-02');

-- Table pour les ventes quotidiennes
CREATE TABLE ventes_quotidiennes (
    date_vente DATE PRIMARY KEY,
    montant REAL
);

INSERT INTO ventes_quotidiennes VALUES
('2024-01-15', 120.50),
('2024-01-16', 85.75),
('2024-01-17', 156.25),
('2024-01-18', 92.30),
('2024-01-19', 134.80),
('2024-01-20', 78.90),
('2024-01-21', 145.60),
('2024-01-22', 167.40),
('2024-01-23', 103.20),
('2024-01-24', 189.75);
```

## Types de fonctions de fenêtrage

### 1. 🏆 Fonctions de classement

#### RANK() - Classement avec égalités
```sql
-- Classer les livres par prix (du plus cher au moins cher)
SELECT
    titre,
    prix,
    RANK() OVER (ORDER BY prix DESC) as rang,
    DENSE_RANK() OVER (ORDER BY prix DESC) as rang_dense,
    ROW_NUMBER() OVER (ORDER BY prix DESC) as numero_ligne
FROM livres
ORDER BY prix DESC;
```

**Différences entre les fonctions de classement :**
- **RANK()** : 1, 2, 2, 4, 5 (saute les rangs après égalité)
- **DENSE_RANK()** : 1, 2, 2, 3, 4 (ne saute pas les rangs)
- **ROW_NUMBER()** : 1, 2, 3, 4, 5 (numéro unique par ligne)

#### Classement par catégorie
```sql
-- Top 3 des livres les plus chers par catégorie
WITH livres_classes AS (
    SELECT
        L.titre,
        L.prix,
        C.nom as categorie,
        RANK() OVER (PARTITION BY L.categorie_id ORDER BY L.prix DESC) as rang_categorie
    FROM livres L
    JOIN categories C ON L.categorie_id = C.id
)
SELECT
    categorie,
    titre,
    prix,
    rang_categorie,
    CASE rang_categorie
        WHEN 1 THEN '🥇 Le plus cher'
        WHEN 2 THEN '🥈 Deuxième'
        WHEN 3 THEN '🥉 Troisième'
        ELSE '📚 Autre'
    END as medaille
FROM livres_classes
WHERE rang_categorie <= 3
ORDER BY categorie, rang_categorie;
```

### 2. 📊 Fonctions de navigation

#### LAG() et LEAD() - Valeurs précédentes et suivantes
```sql
-- Comparer le prix de chaque livre avec le précédent et le suivant
SELECT
    titre,
    prix,
    LAG(prix) OVER (ORDER BY prix) as prix_precedent,
    LEAD(prix) OVER (ORDER BY prix) as prix_suivant,
    prix - LAG(prix) OVER (ORDER BY prix) as difference_precedent,
    ROUND(
        (prix - LAG(prix) OVER (ORDER BY prix)) * 100.0 / LAG(prix) OVER (ORDER BY prix),
        1
    ) as pourcent_augmentation
FROM livres
ORDER BY prix;
```

#### FIRST_VALUE() et LAST_VALUE() - Première et dernière valeur
```sql
-- Comparer chaque livre avec le moins cher et le plus cher de sa catégorie
SELECT
    L.titre,
    L.prix,
    C.nom as categorie,
    FIRST_VALUE(L.prix) OVER (
        PARTITION BY L.categorie_id
        ORDER BY L.prix
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as prix_min_categorie,
    LAST_VALUE(L.prix) OVER (
        PARTITION BY L.categorie_id
        ORDER BY L.prix
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as prix_max_categorie,
    L.prix - FIRST_VALUE(L.prix) OVER (
        PARTITION BY L.categorie_id
        ORDER BY L.prix
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as ecart_avec_min
FROM livres L
JOIN categories C ON L.categorie_id = C.id
ORDER BY C.nom, L.prix;
```

### 3. 📈 Fonctions d'agrégation avec fenêtrage

#### SUM(), AVG(), COUNT() avec OVER
```sql
-- Analyse cumulative des ventes
SELECT
    date_vente,
    montant,
    -- Somme cumulative
    SUM(montant) OVER (ORDER BY date_vente) as cumul_ventes,
    -- Moyenne mobile sur 3 jours
    ROUND(
        AVG(montant) OVER (
            ORDER BY date_vente
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ), 2
    ) as moyenne_3jours,
    -- Pourcentage du total
    ROUND(
        montant * 100.0 / SUM(montant) OVER (), 1
    ) as pourcent_total,
    -- Comparaison avec la moyenne générale
    CASE
        WHEN montant > AVG(montant) OVER () THEN '📈 Au-dessus'
        ELSE '📉 En-dessous'
    END as vs_moyenne
FROM ventes_quotidiennes
ORDER BY date_vente;
```

## Clause ROWS et RANGE - Définir la fenêtre

### Types de fenêtres

```sql
-- Différents types de fenêtres pour moyenne mobile
SELECT
    date_vente,
    montant,
    -- Fenêtre de 3 lignes (ligne actuelle + 2 précédentes)
    ROUND(AVG(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) as moyenne_3_lignes,

    -- Fenêtre centrée (1 avant + actuelle + 1 après)
    ROUND(AVG(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ), 2) as moyenne_centree,

    -- Fenêtre depuis le début
    ROUND(AVG(montant) OVER (
        ORDER BY date_vente
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ), 2) as moyenne_cumulative,

    -- Fenêtre de toutes les lignes
    ROUND(AVG(montant) OVER (
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ), 2) as moyenne_totale
FROM ventes_quotidiennes
ORDER BY date_vente;
```

### Options de fenêtrage :

- **CURRENT ROW** : Ligne actuelle
- **n PRECEDING** : n lignes avant
- **n FOLLOWING** : n lignes après
- **UNBOUNDED PRECEDING** : Depuis le début
- **UNBOUNDED FOLLOWING** : Jusqu'à la fin

## Exemples avancés et cas d'usage

### 1. Analyse des tendances de ventes

```sql
-- Analyser les tendances avec indicateurs multiples
WITH analyse_ventes AS (
    SELECT
        date_vente,
        montant,
        -- Moyennes mobiles
        ROUND(AVG(montant) OVER (
            ORDER BY date_vente
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ), 2) as moy_3j,
        ROUND(AVG(montant) OVER (
            ORDER BY date_vente
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ), 2) as moy_7j,
        -- Valeurs précédentes
        LAG(montant, 1) OVER (ORDER BY date_vente) as vente_hier,
        LAG(montant, 7) OVER (ORDER BY date_vente) as vente_semaine_derniere,
        -- Percentiles
        NTILE(4) OVER (ORDER BY montant) as quartile
    FROM ventes_quotidiennes
)
SELECT
    date_vente,
    montant,
    moy_3j,
    moy_7j,
    -- Tendance court terme (vs hier)
    CASE
        WHEN vente_hier IS NULL THEN '➡️ Premier jour'
        WHEN montant > vente_hier THEN '📈 Hausse'
        WHEN montant < vente_hier THEN '📉 Baisse'
        ELSE '➡️ Stable'
    END as tendance_1j,
    -- Tendance moyen terme (vs semaine dernière)
    CASE
        WHEN vente_semaine_derniere IS NULL THEN '➡️ Première semaine'
        WHEN montant > vente_semaine_derniere * 1.1 THEN '🔥 Forte hausse'
        WHEN montant > vente_semaine_derniere THEN '📈 Hausse'
        WHEN montant < vente_semaine_derniere * 0.9 THEN '❄️ Forte baisse'
        WHEN montant < vente_semaine_derniere THEN '📉 Baisse'
        ELSE '➡️ Stable'
    END as tendance_7j,
    -- Positionnement
    CASE quartile
        WHEN 4 THEN '⭐ Top 25%'
        WHEN 3 THEN '✅ Bon (75%)'
        WHEN 2 THEN '⚠️ Moyen (50%)'
        ELSE '❌ Faible (25%)'
    END as performance
FROM analyse_ventes
ORDER BY date_vente;
```

### 2. Analyse des comportements clients

```sql
-- Analyser les patterns d'achat des clients
WITH historique_clients AS (
    SELECT
        C.client_id,
        CL.nom,
        C.date_commande,
        L.prix * C.quantite as montant_commande,
        -- Numéro de commande pour ce client
        ROW_NUMBER() OVER (PARTITION BY C.client_id ORDER BY C.date_commande) as numero_commande,
        -- Jours depuis la dernière commande
        COALESCE(
            julianday(C.date_commande) - julianday(LAG(C.date_commande) OVER (
                PARTITION BY C.client_id ORDER BY C.date_commande
            )), 0
        ) as jours_depuis_derniere,
        -- Montant de la commande précédente
        LAG(L.prix * C.quantite) OVER (
            PARTITION BY C.client_id ORDER BY C.date_commande
        ) as montant_precedent,
        -- Total cumulé pour ce client
        SUM(L.prix * C.quantite) OVER (
            PARTITION BY C.client_id
            ORDER BY C.date_commande
            ROWS UNBOUNDED PRECEDING
        ) as total_cumule
    FROM commandes C
    JOIN livres L ON C.livre_id = L.id
    JOIN clients CL ON C.client_id = CL.id
)
SELECT
    nom,
    date_commande,
    montant_commande,
    numero_commande,
    ROUND(jours_depuis_derniere, 0) as jours_entre_commandes,
    -- Évolution du montant
    CASE
        WHEN numero_commande = 1 THEN '🎉 Première commande'
        WHEN montant_commande > montant_precedent THEN '📈 Augmentation'
        WHEN montant_commande < montant_precedent THEN '📉 Diminution'
        ELSE '➡️ Même montant'
    END as evolution_montant,
    -- Fidélité
    CASE
        WHEN numero_commande = 1 THEN '🆕 Nouveau'
        WHEN numero_commande <= 3 THEN '👍 Régulier'
        ELSE '⭐ Fidèle'
    END as statut_fidelite,
    total_cumule
FROM historique_clients
ORDER BY nom, date_commande;
```

### 3. Analyse de performance produits

```sql
-- Analyser la performance des livres avec ranking multiple
WITH stats_livres AS (
    SELECT
        L.id,
        L.titre,
        L.prix,
        A.nom as auteur,
        C.nom as categorie,
        -- Ventes
        COALESCE(SUM(CO.quantite), 0) as total_vendus,
        COALESCE(SUM(L.prix * CO.quantite), 0) as revenus_total,
        COUNT(CO.id) as nb_commandes,
        -- Stocks
        L.stock,
        -- Date
        L.date_publication
    FROM livres L
    JOIN auteurs A ON L.auteur_id = A.id
    JOIN categories C ON L.categorie_id = C.id
    LEFT JOIN commandes CO ON L.id = CO.livre_id
    GROUP BY L.id, L.titre, L.prix, A.nom, C.nom, L.stock, L.date_publication
)
SELECT
    titre,
    auteur,
    categorie,
    prix,
    total_vendus,
    revenus_total,
    stock,
    -- Rankings globaux
    RANK() OVER (ORDER BY total_vendus DESC) as rang_ventes,
    RANK() OVER (ORDER BY revenus_total DESC) as rang_revenus,
    RANK() OVER (ORDER BY prix DESC) as rang_prix,
    -- Rankings par catégorie
    RANK() OVER (PARTITION BY categorie ORDER BY total_vendus DESC) as rang_ventes_categorie,
    -- Percentiles
    NTILE(5) OVER (ORDER BY total_vendus) as quintile_ventes,
    -- Comparaisons
    CASE
        WHEN total_vendus > AVG(total_vendus) OVER () THEN '⭐ Au-dessus moyenne'
        WHEN total_vendus = 0 THEN '❌ Aucune vente'
        ELSE '⚠️ En-dessous moyenne'
    END as performance_ventes,
    -- Ratio stock/ventes
    CASE
        WHEN total_vendus = 0 THEN '📦 Surstock'
        WHEN stock < total_vendus THEN '🔥 Stock faible'
        WHEN stock < total_vendus * 2 THEN '⚠️ Stock juste'
        ELSE '✅ Stock OK'
    END as statut_stock
FROM stats_livres
ORDER BY total_vendus DESC, revenus_total DESC;
```

## Clause WINDOW - Réutiliser les définitions

Pour éviter la répétition, vous pouvez nommer vos fenêtres :

```sql
-- Au lieu de répéter la définition OVER
SELECT
    titre,
    prix,
    RANK() OVER (PARTITION BY categorie_id ORDER BY prix DESC) as rang,
    AVG(prix) OVER (PARTITION BY categorie_id ORDER BY prix DESC) as moy,
    COUNT(*) OVER (PARTITION BY categorie_id ORDER BY prix DESC) as nb
FROM livres;

-- Utilisez WINDOW pour définir une fois
SELECT
    titre,
    prix,
    RANK() OVER win as rang,
    AVG(prix) OVER win as moy,
    COUNT(*) OVER win as nb
FROM livres
WINDOW win AS (PARTITION BY categorie_id ORDER BY prix DESC);
```

## Exercices pratiques

### Exercice 1 (Débutant)
Affichez tous les livres avec leur rang par prix (du plus cher au moins cher) et indiquez si chaque livre est dans le top 3.

<details>
<summary>Solution</summary>

```sql
SELECT
    titre,
    prix,
    RANK() OVER (ORDER BY prix DESC) as rang,
    CASE
        WHEN RANK() OVER (ORDER BY prix DESC) <= 3 THEN '🏆 Top 3'
        ELSE '📚 Autre'
    END as statut
FROM livres
ORDER BY prix DESC;
```
</details>

### Exercice 2 (Intermédiaire)
Pour chaque livre, calculez la différence de prix avec le livre suivant et précédent dans l'ordre alphabétique des titres.

<details>
<summary>Solution</summary>

```sql
SELECT
    titre,
    prix,
    LAG(prix) OVER (ORDER BY titre) as prix_precedent,
    LEAD(prix) OVER (ORDER BY titre) as prix_suivant,
    prix - LAG(prix) OVER (ORDER BY titre) as diff_precedent,
    LEAD(prix) OVER (ORDER BY titre) - prix as diff_suivant
FROM livres
ORDER BY titre;
```
</details>

### Exercice 3 (Avancé)
Créez un tableau de bord des ventes quotidiennes avec :
- Le montant du jour
- La moyenne mobile sur 3 jours
- Le pourcentage de croissance par rapport à la veille
- Le rang du jour (du meilleur au pire)
- Un indicateur de tendance

<details>
<summary>Solution</summary>

```sql
WITH analyse_complete AS (
    SELECT
        date_vente,
        montant,
        -- Moyenne mobile 3 jours
        ROUND(AVG(montant) OVER (
            ORDER BY date_vente
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ), 2) as moyenne_3j,
        -- Valeur précédente
        LAG(montant) OVER (ORDER BY date_vente) as montant_veille,
        -- Rang par performance
        RANK() OVER (ORDER BY montant DESC) as rang_performance
    FROM ventes_quotidiennes
)
SELECT
    date_vente,
    montant,
    moyenne_3j,
    -- Croissance vs veille
    CASE
        WHEN montant_veille IS NULL THEN NULL
        ELSE ROUND(
            ((montant - montant_veille) * 100.0 / montant_veille), 1
        )
    END as croissance_pourcent,
    rang_performance,
    -- Indicateur de tendance
    CASE
        WHEN montant_veille IS NULL THEN '➡️ Premier jour'
        WHEN montant > montant_veille * 1.2 THEN '🚀 Forte hausse'
        WHEN montant > montant_veille THEN '📈 Hausse'
        WHEN montant < montant_veille * 0.8 THEN '📉 Forte baisse'
        WHEN montant < montant_veille THEN '📉 Baisse'
        ELSE '➡️ Stable'
    END as tendance,
    -- Performance relative
    CASE
        WHEN rang_performance <= 3 THEN '🏆 Excellent'
        WHEN rang_performance <= 6 THEN '✅ Bon'
        ELSE '⚠️ À améliorer'
    END as evaluation
FROM analyse_complete
ORDER BY date_vente;
```
</details>

## Bonnes pratiques et optimisation

### ✅ **Bonnes pratiques :**

1. **Utilisez ORDER BY quand nécessaire :**
```sql
-- ✅ BON : ORDER BY pour les fonctions qui en ont besoin
SELECT prix, LAG(prix) OVER (ORDER BY date_publication) FROM livres;

-- ❌ ATTENTION : RANK() sans ORDER BY n'a pas de sens
SELECT prix, RANK() OVER () FROM livres; -- Tous auront rang 1 !
```

2. **Optimisez avec WINDOW :**
```sql
-- ✅ BON : Réutilise la définition
SELECT
    titre,
    RANK() OVER win,
    AVG(prix) OVER win
FROM livres
WINDOW win AS (PARTITION BY categorie_id ORDER BY prix);
```

3. **Attention aux fenêtres complètes :**
```sql
-- ⚠️ PEUT ÊTRE LENT : Fenêtre très large
SELECT montant, AVG(montant) OVER (
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) FROM ventes_quotidiennes;

-- ✅ MIEUX : Si vous voulez juste la moyenne générale
SELECT montant, (SELECT AVG(montant) FROM ventes_quotidiennes)
FROM ventes_quotidiennes;
```

### 🚫 **Erreurs courantes :**

1. **Confondre avec GROUP BY :**
```sql
-- ❌ ERREUR : Mélanger GROUP BY et WINDOW
SELECT categorie_id, COUNT(*), RANK() OVER (ORDER BY prix)
FROM livres GROUP BY categorie_id; -- Ne marchera pas !

-- ✅ CORRECT : Soit l'un, soit l'autre
SELECT categorie_id, COUNT(*) FROM livres GROUP BY categorie_id;
-- OU
SELECT titre, prix, RANK() OVER (ORDER BY prix) FROM livres;
```

## Résumé

Les **fonctions de fenêtrage** sont un outil puissant pour l'analyse de données :

### 🎯 **Types principaux :**
- **Classement** : RANK(), DENSE_RANK(), ROW_NUMBER(), NTILE()
- **Navigation** : LAG(), LEAD(), FIRST_VALUE(), LAST_VALUE()
- **Agrégation** : SUM(), AVG(), COUNT(), MIN(), MAX() avec OVER

### 📊 **Cas d'usage typiques :**
- **Classements et tops** (meilleurs vendeurs, produits les plus chers)
- **Analyses temporelles** (tendances, moyennes mobiles, comparaisons)
- **Comparaisons ligne par ligne** (évolution, différences)
- **Calculs cumulatifs** (totaux cumulés, pourcentages)

### ⚡ **Avantages :**
- **Gardent toutes les lignes** (contrairement à GROUP BY)
- **Calculs sophistiqués** en une seule requête
- **Performance** souvent meilleure que les sous-requêtes corrélées
- **Lisibilité** du code améliorée

### 📋 **Points clés :**
- **PARTITION BY** = divise en groupes sans résumer
- **ORDER BY** = définit l'ordre pour les calculs
- **ROWS/RANGE** = taille de la fenêtre de calcul
- **WINDOW** = réutilise les définitions

Les fonctions de fenêtrage transforment votre capacité d'analyse ! Elles sont particulièrement puissantes combinées avec les CTE pour créer des rapports sophistiqués et des tableaux de bord détaillés.

Dans la prochaine section, nous découvrirons comment utiliser les **expressions régulières** pour des recherches et validations de données avancées.

⏭️
