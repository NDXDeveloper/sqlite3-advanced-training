🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.5 Requêtes complexes avec CASE, COALESCE, NULLIF

## Introduction

Imaginez que vous devez créer un rapport où les prix sont catégorisés en "Cher", "Moyen", "Abordable", où les valeurs NULL sont remplacées par des valeurs par défaut, ou encore où vous voulez traiter différemment les données selon certaines conditions. Les fonctions **CASE**, **COALESCE** et **NULLIF** sont vos "outils de logique conditionnelle" qui transforment vos requêtes simples en véritables programmes intelligents !

Pensez à ces fonctions comme aux **"Si... alors... sinon"** de vos requêtes SQL !

## CASE : La logique conditionnelle

### Qu'est-ce que CASE ?

**CASE** est l'équivalent du "if-else" en SQL. Il permet d'appliquer différentes logiques selon les conditions rencontrées dans vos données.

### Deux syntaxes de CASE

#### 1. CASE simple (comparaison directe)

```text
CASE colonne
    WHEN valeur1 THEN resultat1
    WHEN valeur2 THEN resultat2
    ELSE resultat_par_defaut
END
```

#### 2. CASE avec conditions (plus flexible)

```text
CASE
    WHEN condition1 THEN resultat1
    WHEN condition2 THEN resultat2
    ELSE resultat_par_defaut
END
```

## Préparation des données

Ajoutons des données avec des valeurs NULL et variées pour nos exemples :

```sql
-- Ajoutons plus de variété dans nos données
-- Note : la table `livres` du 4.2 a `prix REAL NOT NULL`. Pour illustrer
-- la gestion des valeurs « manquantes », on utilise 0.00 comme sentinelle
-- (qui sera convertie en NULL par NULLIF(prix, 0) dans les requêtes).
INSERT INTO livres VALUES
(16, 'Livre gratuit', 0.00, 1, 4, 100, '2023-01-01'),
(17, 'Livre premium', 99.99, 2, 4, 1, '2023-06-15'),
(18, 'Livre sans prix', 0.00, 3, 4, 5, '2023-03-20');

-- Note : la table `clients` du chapitre 4.2 a `nom TEXT NOT NULL`.
-- Pour illustrer la gestion des valeurs « vides », on utilise une chaîne vide
-- (qui sera convertie en NULL par NULLIF plus loin) plutôt qu'un NULL direct.
INSERT INTO clients VALUES
(9, '', 'anonyme@test.com', 'Inconnu'),
(10, 'Client VIP', 'vip@luxury.com', NULL),
(11, ' ', 'vide@test.com', 'France');

-- Table avec plus de variations
CREATE TABLE employes (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    prenom TEXT,
    salaire REAL,
    departement TEXT,
    date_embauche DATE,
    manager_id INTEGER
);

INSERT INTO employes VALUES
(1, 'Dupont', 'Jean', 35000, 'IT', '2020-01-15', NULL),
(2, 'Martin', 'Marie', 42000, 'Marketing', '2019-03-20', 1),
(3, 'Bernard', NULL, 38000, 'IT', '2021-06-10', 1),
(4, 'Petit', 'Pierre', NULL, 'Finance', '2022-02-01', 2),
(5, 'Moreau', 'Sophie', 55000, NULL, '2018-09-12', NULL),
(6, 'Leroy', 'Paul', 28000, 'Marketing', '2023-01-08', 2);

-- Table de ventes avec données manquantes
CREATE TABLE ventes_detaillees (
    id INTEGER PRIMARY KEY,
    vendeur TEXT,
    montant REAL,
    commission REAL,
    bonus REAL,
    region TEXT,
    trimestre INTEGER
);

INSERT INTO ventes_detaillees VALUES
(1, 'Alice', 1500, 150, NULL, 'Nord', 1),
(2, 'Bob', 2200, NULL, 200, 'Sud', 1),
(3, 'Charlie', NULL, 180, 100, 'Est', 2),
(4, 'Diana', 1800, 200, NULL, NULL, 2),
(5, 'Eve', 0, 0, 0, 'Ouest', 3);
```

## CASE : Exemples de base

### Exemple 1 : Catégorisation des prix
```sql
-- Catégoriser les livres par gamme de prix
SELECT
    titre,
    prix,
    CASE
        WHEN prix IS NULL THEN '❓ Prix non défini'
        WHEN prix = 0 THEN '🆓 Gratuit'
        WHEN prix < 15 THEN '💰 Abordable'
        WHEN prix < 30 THEN '💸 Moyen'
        WHEN prix < 50 THEN '💎 Cher'
        ELSE '👑 Premium'
    END as gamme_prix
FROM livres  
ORDER BY prix NULLS LAST;  
```

### Exemple 2 : CASE simple vs CASE avec conditions
```sql
-- CASE simple : comparaison directe
SELECT
    nom,
    pays,
    CASE pays
        WHEN 'France' THEN '🇫🇷 Français'
        WHEN 'USA' THEN '🇺🇸 Américain'
        WHEN 'Japon' THEN '🇯🇵 Japonais'
        WHEN 'Royaume-Uni' THEN '🇬🇧 Britannique'
        ELSE '🌍 Autre nationalité'
    END as nationalite
FROM auteurs;

-- CASE avec conditions : plus flexible
SELECT
    nom,
    pays,
    CASE
        WHEN pays IS NULL THEN '❓ Nationalité inconnue'
        WHEN pays IN ('France', 'Belgique', 'Suisse') THEN '🇫🇷 Francophone'
        WHEN pays IN ('USA', 'Canada', 'Royaume-Uni') THEN '🇬🇧 Anglophone'
        WHEN LENGTH(pays) < 3 THEN '⚠️ Code pays invalide'
        ELSE '🌍 Autre (' || pays || ')'
    END as zone_linguistique
FROM auteurs;
```

## CASE : Exemples avancés

### 1. Calculs conditionnels
```sql
-- Calcul de remises selon différents critères
SELECT
    titre,
    prix,
    stock,
    CASE
        WHEN prix IS NULL THEN 0
        WHEN stock > 10 THEN prix * 0.9  -- 10% de remise si beaucoup de stock
        WHEN stock > 5 THEN prix * 0.95   -- 5% de remise si stock moyen
        WHEN stock <= 2 THEN prix * 1.1   -- 10% de majoration si stock faible
        ELSE prix
    END as prix_ajuste,
    CASE
        WHEN stock > 10 THEN '🔥 Promo -10%'
        WHEN stock > 5 THEN '✨ Promo -5%'
        WHEN stock <= 2 THEN '⚠️ Majoration +10%'
        ELSE '➡️ Prix normal'
    END as statut_prix
FROM livres  
ORDER BY stock;  
```

### 2. Agrégations conditionnelles
```sql
-- Statistiques avec conditions complexes
SELECT
    departement,
    COUNT(*) as total_employes,
    -- Compter selon conditions
    COUNT(CASE WHEN salaire >= 40000 THEN 1 END) as salaires_eleves,
    COUNT(CASE WHEN salaire < 30000 THEN 1 END) as salaires_faibles,
    COUNT(CASE WHEN prenom IS NOT NULL THEN 1 END) as avec_prenom,

    -- Moyennes conditionnelles
    ROUND(AVG(CASE WHEN salaire IS NOT NULL THEN salaire END), 0) as salaire_moyen,
    ROUND(AVG(CASE WHEN salaire >= 40000 THEN salaire END), 0) as moyenne_salaires_eleves,

    -- Sommes conditionnelles
    SUM(CASE
        WHEN salaire IS NULL THEN 0
        WHEN salaire >= 50000 THEN salaire * 0.1  -- Bonus 10% pour hauts salaires
        WHEN salaire >= 35000 THEN salaire * 0.05  -- Bonus 5% pour salaires moyens
        ELSE salaire * 0.02  -- Bonus 2% pour autres
    END) as total_bonus_estime
FROM employes  
WHERE departement IS NOT NULL  
GROUP BY departement  
ORDER BY total_employes DESC;  
```

### 3. CASE imbriqués (conditions complexes)
```sql
-- Évaluation complexe des performances employés
SELECT
    nom,
    prenom,
    salaire,
    departement,
    date_embauche,
    CASE
        WHEN departement IS NULL THEN '❓ Département non défini'
        WHEN salaire IS NULL THEN '💰 Salaire à négocier'
        ELSE
            CASE departement
                WHEN 'IT' THEN
                    CASE
                        WHEN salaire >= 45000 THEN '🌟 Senior IT'
                        WHEN salaire >= 35000 THEN '⚡ IT Confirmé'
                        ELSE '🚀 Junior IT'
                    END
                WHEN 'Marketing' THEN
                    CASE
                        WHEN salaire >= 40000 THEN '📊 Marketing Senior'
                        WHEN salaire >= 30000 THEN '📈 Marketing Confirmé'
                        ELSE '📢 Marketing Junior'
                    END
                WHEN 'Finance' THEN
                    CASE
                        WHEN salaire >= 50000 THEN '💎 Finance Senior'
                        WHEN salaire >= 35000 THEN '💼 Finance Confirmé'
                        ELSE '📋 Finance Junior'
                    END
                ELSE '🏢 ' || departement || ' - Non classé'
            END
    END as niveau_poste,

    -- Ancienneté et évolution
    CASE
        WHEN julianday('now') - julianday(date_embauche) >= 365 * 3 THEN '🎯 Vétéran (3+ ans)'
        WHEN julianday('now') - julianday(date_embauche) >= 365 * 1 THEN '⚡ Expérimenté (1-3 ans)'
        ELSE '🌱 Nouveau (<1 an)'
    END as anciennete
FROM employes  
ORDER BY departement, salaire DESC NULLS LAST;  
```

## COALESCE : Gestion intelligente des NULL

### Qu'est-ce que COALESCE ?

**COALESCE** retourne la première valeur non-NULL parmi une liste d'expressions. C'est l'outil parfait pour gérer les valeurs manquantes !

```text
COALESCE(valeur1, valeur2, valeur3, ..., valeur_par_defaut)
```

### Exemples de base avec COALESCE

```sql
-- Remplacer les valeurs NULL par des valeurs par défaut
SELECT
    nom,
    prenom,
    COALESCE(nom, 'Nom manquant') as nom_propre,
    COALESCE(prenom, 'Prénom manquant') as prenom_propre,
    COALESCE(nom || ' ' || prenom, nom, prenom, 'Employé anonyme') as nom_complet
FROM employes;

-- Utiliser plusieurs sources pour combler les manques
SELECT
    nom,
    salaire,
    departement,
    COALESCE(salaire, 30000) as salaire_avec_defaut,
    COALESCE(departement, 'Non assigné') as dept_avec_defaut,
    -- Priorité : salaire réel, puis estimation par département, puis minimum
    COALESCE(
        salaire,
        CASE departement
            WHEN 'IT' THEN 38000
            WHEN 'Marketing' THEN 35000
            WHEN 'Finance' THEN 40000
            ELSE 32000
        END,
        25000
    ) as salaire_estime
FROM employes;
```

### Exemples avancés avec COALESCE

#### 1. Calculs avec valeurs manquantes
```sql
-- Calculer la rémunération totale en gérant les NULL
SELECT
    vendeur,
    montant,
    commission,
    bonus,
    region,
    -- Version basique (incorrecte avec NULL)
    montant + commission + bonus as total_incorrect,

    -- Version correcte avec COALESCE
    COALESCE(montant, 0) + COALESCE(commission, 0) + COALESCE(bonus, 0) as total_remuneration,

    -- Calcul de commission avec fallback
    COALESCE(
        commission,  -- Commission réelle si disponible
        montant * 0.1,  -- Sinon 10% du montant
        100  -- Sinon commission minimale
    ) as commission_finale,

    -- Région avec gestion intelligente
    COALESCE(region, 'Non assignée') as region_finale
FROM ventes_detaillees;
```

#### 2. Agrégations robustes
```sql
-- Statistiques avec gestion des NULL
SELECT
    region,
    COUNT(*) as nb_ventes,
    -- Sommes avec COALESCE
    SUM(COALESCE(montant, 0)) as total_montant,
    SUM(COALESCE(commission, 0)) as total_commission,
    SUM(COALESCE(bonus, 0)) as total_bonus,

    -- Moyennes plus précises
    ROUND(AVG(COALESCE(montant, 0)), 2) as moyenne_montant,
    ROUND(AVG(montant), 2) as moyenne_montant_non_null_seulement,

    -- Statistiques conditionnelles
    COUNT(CASE WHEN montant IS NOT NULL THEN 1 END) as nb_montants_renseignes,
    COUNT(CASE WHEN commission IS NOT NULL THEN 1 END) as nb_commissions_renseignees,

    -- Taux de complétude
    ROUND(
        COUNT(CASE WHEN montant IS NOT NULL THEN 1 END) * 100.0 / COUNT(*), 1
    ) as taux_montant_complete
FROM ventes_detaillees  
WHERE COALESCE(region, 'Non assignée') != 'Non assignée'  -- Exclure les régions non assignées  
GROUP BY region  
ORDER BY total_montant DESC;  
```

## NULLIF : Convertir des valeurs en NULL

### Qu'est-ce que NULLIF ?

**NULLIF** retourne NULL si deux expressions sont égales, sinon retourne la première expression. C'est utile pour "nettoyer" des données en convertissant certaines valeurs en NULL.

```text
NULLIF(expression1, expression2)
-- Si expression1 = expression2, retourne NULL
-- Sinon retourne expression1
```

### Exemples de base avec NULLIF

```sql
-- Convertir les valeurs "vides" en NULL
SELECT
    nom,
    email,
    pays,
    NULLIF(nom, '') as nom_nettoye,  -- Chaîne vide devient NULL
    NULLIF(pays, 'Inconnu') as pays_nettoye,  -- "Inconnu" devient NULL
    NULLIF(TRIM(nom), '') as nom_trim_nettoye  -- Espaces + vide devient NULL
FROM clients;

-- Nettoyer les données numériques
SELECT
    vendeur,
    montant,
    commission,
    bonus,
    NULLIF(montant, 0) as montant_nettoye,  -- 0 devient NULL
    NULLIF(commission, 0) as commission_nettoyee,
    NULLIF(bonus, 0) as bonus_nettoye
FROM ventes_detaillees;
```

### Exemples avancés avec NULLIF

#### 1. Nettoyage de données complexe
```sql
-- Nettoyage intelligent des données employés
WITH donnees_nettoyees AS (
    SELECT
        id,
        NULLIF(TRIM(nom), '') as nom,
        NULLIF(TRIM(prenom), '') as prenom,
        NULLIF(salaire, 0) as salaire,  -- Salaire 0 = non renseigné
        NULLIF(TRIM(departement), '') as departement,
        date_embauche,
        NULLIF(manager_id, 0) as manager_id  -- Manager 0 = pas de manager
    FROM employes
)
SELECT
    id,
    COALESCE(nom, 'Nom_' || id) as nom_final,
    COALESCE(prenom, 'Prénom_' || id) as prenom_final,
    salaire,
    departement,
    date_embauche,
    manager_id,
    -- Indicateurs de qualité des données
    CASE
        WHEN nom IS NULL THEN '⚠️ Nom manquant'
        WHEN prenom IS NULL THEN '⚠️ Prénom manquant'
        WHEN salaire IS NULL THEN '⚠️ Salaire manquant'
        WHEN departement IS NULL THEN '⚠️ Département manquant'
        ELSE '✅ Données complètes'
    END as qualite_donnees
FROM donnees_nettoyees  
ORDER BY qualite_donnees, id;  
```

#### 2. Calculs statistiques précis
```sql
-- Éviter les divisions par zéro et calculs incorrects
SELECT
    region,
    vendeur,
    montant,
    commission,
    -- Division sécurisée
    ROUND(
        COALESCE(commission, 0) * 100.0 / NULLIF(montant, 0), 2
    ) as taux_commission_pourcent,

    -- Ratio avec gestion des cas spéciaux
    CASE
        WHEN NULLIF(montant, 0) IS NULL THEN 'Montant invalide'
        WHEN commission IS NULL THEN 'Commission non renseignée'
        WHEN commission = 0 THEN 'Pas de commission'
        ELSE ROUND(commission * 100.0 / montant, 2) || '%'
    END as analyse_commission,

    -- Performance relative
    CASE
        WHEN NULLIF(montant, 0) IS NULL THEN '❓ Non évaluable'
        WHEN montant >= 2000 THEN '🌟 Excellent'
        WHEN montant >= 1500 THEN '✅ Bon'
        WHEN montant >= 1000 THEN '⚠️ Moyen'
        ELSE '❌ Faible'
    END as performance
FROM ventes_detaillees  
WHERE COALESCE(region, '') != ''  -- Exclure les régions vides  
ORDER BY montant DESC NULLS LAST;  
```

## Combinaison des trois fonctions

### Exemple complexe : Rapport de performance complet

```sql
-- Rapport de performance vendeur ultra-complet
WITH statistiques_vendeur AS (
    SELECT
        vendeur,
        -- Nettoyage avec NULLIF
        NULLIF(montant, 0) as montant_valide,
        NULLIF(commission, 0) as commission_valide,
        NULLIF(bonus, 0) as bonus_valide,
        COALESCE(region, 'Non assignée') as region_finale
    FROM ventes_detaillees
),
calculs_vendeur AS (
    SELECT
        vendeur,
        COUNT(*) as nb_ventes,
        -- Totaux avec gestion NULL
        SUM(COALESCE(montant_valide, 0)) as total_montant,
        SUM(COALESCE(commission_valide, 0)) as total_commission,
        SUM(COALESCE(bonus_valide, 0)) as total_bonus,

        -- Moyennes
        ROUND(AVG(montant_valide), 2) as moyenne_montant,

        -- Complétion des données
        COUNT(montant_valide) as nb_montants_valides,
        COUNT(commission_valide) as nb_commissions_valides,
        COUNT(bonus_valide) as nb_bonus_valides
    FROM statistiques_vendeur
    GROUP BY vendeur
)
SELECT
    vendeur,
    nb_ventes,
    total_montant,
    total_commission,
    total_bonus,

    -- Calcul de la rémunération totale
    total_montant + total_commission + total_bonus as remuneration_totale,

    -- Classification performance avec CASE
    CASE
        WHEN total_montant >= 2000 THEN '🏆 Top Performer'
        WHEN total_montant >= 1500 THEN '🌟 Bon vendeur'
        WHEN total_montant >= 1000 THEN '⚡ Vendeur moyen'
        WHEN total_montant > 0 THEN '📈 Débutant'
        ELSE '❌ Pas de ventes'
    END as niveau_performance,

    -- Analyse de la complétude des données
    CASE
        WHEN nb_montants_valides = nb_ventes
         AND nb_commissions_valides = nb_ventes
         AND nb_bonus_valides = nb_ventes THEN '✅ Données complètes'
        WHEN nb_montants_valides = nb_ventes THEN '⚠️ Montants OK, autres incomplets'
        WHEN nb_montants_valides = 0 THEN '❌ Aucun montant valide'
        ELSE '⚠️ Données partielles'
    END as qualite_donnees,

    -- Taux de commission moyen
    CASE
        WHEN NULLIF(total_montant, 0) IS NULL THEN 'N/A'
        ELSE ROUND(total_commission * 100.0 / total_montant, 1) || '%'
    END as taux_commission_moyen,

    -- Recommandations
    CASE
        WHEN total_montant = 0 THEN '🎯 Formation commerciale urgente'
        WHEN total_commission = 0 AND total_montant > 0 THEN '💰 Revoir système commission'
        WHEN nb_commissions_valides < nb_ventes THEN '📋 Compléter données commission'
        WHEN total_montant >= 2000 THEN '🎉 Félicitations ! Maintenir niveau'
        ELSE '📈 Potentiel d''amélioration'
    END as recommandation
FROM calculs_vendeur  
ORDER BY total_montant DESC;  
```

## Exemples de cas d'usage métier

### 1. Système de fidélité client

```sql
-- Calcul des points de fidélité avec règles complexes
WITH historique_achats AS (
    SELECT
        C.client_id,
        CL.nom,
        COUNT(C.id) as nb_commandes,
        SUM(L.prix * C.quantite) as total_achats,
        MIN(C.date_commande) as premiere_commande,
        MAX(C.date_commande) as derniere_commande
    FROM commandes C
    JOIN livres L ON C.livre_id = L.id
    JOIN clients CL ON C.client_id = CL.id
    GROUP BY C.client_id, CL.nom
)
SELECT
    nom,
    nb_commandes,
    ROUND(total_achats, 2) as total_achats,

    -- Calcul des points avec règles complexes
    CASE
        WHEN total_achats >= 200 THEN ROUND(total_achats * 2, 0)  -- Double points VIP
        WHEN total_achats >= 100 THEN ROUND(total_achats * 1.5, 0)  -- 50% bonus
        ELSE ROUND(total_achats, 0)  -- Points standard
    END as points_fidelite,

    -- Statut client
    CASE
        WHEN nb_commandes >= 5 AND total_achats >= 200 THEN '👑 VIP Platinum'
        WHEN nb_commandes >= 3 AND total_achats >= 100 THEN '🌟 VIP Gold'
        WHEN nb_commandes >= 2 OR total_achats >= 50 THEN '✨ Fidèle'
        ELSE '🆕 Nouveau'
    END as statut_fidelite,

    -- Ancienneté
    CASE
        WHEN julianday('now') - julianday(premiere_commande) >= 365 THEN '🎯 Client établi'
        WHEN julianday('now') - julianday(premiere_commande) >= 90 THEN '⚡ Client régulier'
        ELSE '🌱 Client récent'
    END as anciennete,

    -- Recommandation marketing
    CASE
        WHEN julianday('now') - julianday(derniere_commande) >= 90 THEN
            '📧 Campagne de réactivation'
        WHEN total_achats >= 150 AND nb_commandes < 3 THEN
            '🎁 Offre fidélité pour gros achats'
        WHEN nb_commandes >= 3 AND total_achats < 100 THEN
            '💰 Offre montée en gamme'
        ELSE '✅ Suivi standard'
    END as action_marketing
FROM historique_achats  
ORDER BY total_achats DESC;  
```

### 2. Tableau de bord des stocks

```sql
-- Analyse intelligente des stocks avec alertes
SELECT
    L.titre,
    L.prix,
    L.stock,
    COALESCE(SUM(C.quantite), 0) as total_vendus,

    -- Calcul du stock optimal (estimation)
    CASE
        WHEN COALESCE(SUM(C.quantite), 0) = 0 THEN 5  -- Stock minimal pour nouveaux produits
        ELSE COALESCE(SUM(C.quantite), 0) * 2  -- 2x les ventes comme stock optimal
    END as stock_optimal,

    -- Statut du stock
    CASE
        WHEN L.stock = 0 THEN '❌ Rupture'
        WHEN L.stock <= COALESCE(SUM(C.quantite), 0) * 0.5 THEN '⚠️ Stock critique'
        WHEN L.stock <= COALESCE(SUM(C.quantite), 0) THEN '🔶 Stock faible'
        WHEN L.stock >= COALESCE(SUM(C.quantite), 0) * 3 THEN '📦 Surstock'
        ELSE '✅ Stock correct'
    END as statut_stock,

    -- Priorité de réapprovisionnement
    CASE
        WHEN L.stock = 0 AND COALESCE(SUM(C.quantite), 0) > 0 THEN '🚨 URGENT'
        WHEN L.stock <= COALESCE(SUM(C.quantite), 0) * 0.5 THEN '🔥 Priorité haute'
        WHEN L.stock <= COALESCE(SUM(C.quantite), 0) THEN '⚡ Priorité normale'
        ELSE '➡️ Pas urgent'
    END as priorite_reappro,

    -- Quantité suggérée à commander
    -- ⚠️ SQLite n'a PAS `GREATEST`/`LEAST` (≠ MySQL/PostgreSQL).
    --    On utilise `MAX(a, b, …)` / `MIN(a, b, …)` qui, **avec plusieurs arguments**,
    --    sont des fonctions scalaires (et non agrégats).
    CASE
        WHEN L.stock = 0 THEN MAX(COALESCE(SUM(C.quantite), 0) * 2, 5)
        WHEN L.stock <= COALESCE(SUM(C.quantite), 0) * 0.5 THEN
            MAX(COALESCE(SUM(C.quantite), 0) * 2 - L.stock, 1)
        ELSE 0
    END as quantite_a_commander,

    -- Valeur du stock
    COALESCE(L.stock * NULLIF(L.prix, 0), 0) as valeur_stock,

    -- Performance du produit
    CASE
        WHEN COALESCE(SUM(C.quantite), 0) = 0 THEN '😴 Aucune vente'
        WHEN COALESCE(SUM(C.quantite), 0) >= 5 THEN '🔥 Best-seller'
        WHEN COALESCE(SUM(C.quantite), 0) >= 3 THEN '✅ Bon vendeur'
        ELSE '📈 Vendeur moyen'
    END as performance_vente
FROM livres L  
LEFT JOIN commandes C ON L.id = C.livre_id  
GROUP BY L.id, L.titre, L.prix, L.stock  
ORDER BY  
    CASE
        WHEN L.stock = 0 THEN 1
        WHEN L.stock <= COALESCE(SUM(C.quantite), 0) * 0.5 THEN 2
        ELSE 3
    END,
    COALESCE(SUM(C.quantite), 0) DESC;
```

## Exercices pratiques

### Exercice 1 (Débutant)
Créez une requête qui affiche tous les livres avec une colonne "Prix_Categorie" qui indique "Gratuit" pour prix = 0, "Abordable" pour prix < 20, "Cher" pour prix >= 20, et "Prix non défini" pour les NULL.

<details>
<summary>Solution</summary>

```sql
SELECT
    titre,
    prix,
    CASE
        WHEN prix IS NULL THEN 'Prix non défini'
        WHEN prix = 0 THEN 'Gratuit'
        WHEN prix < 20 THEN 'Abordable'
        ELSE 'Cher'
    END as Prix_Categorie
FROM livres  
ORDER BY prix NULLS LAST;  
```
</details>

### Exercice 2 (Intermédiaire)
Utilisez COALESCE pour créer une colonne "nom_affichage" qui affiche le nom complet (nom + prénom) des employés, ou juste le nom si le prénom manque, ou "Employé_" + id si les deux manquent.

<details>
<summary>Solution</summary>

```sql
SELECT
    id,
    nom,
    prenom,
    COALESCE(
        CASE
            WHEN nom IS NOT NULL AND prenom IS NOT NULL
            THEN nom || ' ' || prenom
            ELSE NULL
        END,
        nom,
        'Employé_' || id
    ) as nom_affichage
FROM employes  
ORDER BY id;  
```
</details>

### Exercice 3 (Avancé)
Créez un rapport de ventes qui utilise les trois fonctions (CASE, COALESCE, NULLIF) pour :
- Nettoyer les montants (0 devient NULL avec NULLIF)
- Remplacer les valeurs manquantes par des estimations (COALESCE)
- Catégoriser les performances (CASE)
- Calculer une prime avec des règles complexes

<details>
<summary>Solution</summary>

```sql
WITH ventes_nettoyees AS (
    SELECT
        vendeur,
        NULLIF(montant, 0) as montant_propre,
        NULLIF(commission, 0) as commission_propre,
        NULLIF(bonus, 0) as bonus_propre,
        COALESCE(region, 'Non assignée') as region_finale,
        trimestre
    FROM ventes_detaillees
),
ventes_completes AS (
    SELECT
        vendeur,
        region_finale,
        trimestre,
        -- Estimation intelligente des valeurs manquantes
        COALESCE(
            montant_propre,
            -- Si pas de montant, estimer selon région et trimestre
            CASE region_finale
                WHEN 'Nord' THEN 1600
                WHEN 'Sud' THEN 1800
                WHEN 'Est' THEN 1400
                WHEN 'Ouest' THEN 1500
                ELSE 1500
            END
        ) as montant_final,

        COALESCE(
            commission_propre,
            -- Si pas de commission, calculer 10% du montant estimé
            COALESCE(montant_propre, 1500) * 0.1
        ) as commission_finale,

        COALESCE(bonus_propre, 0) as bonus_final
    FROM ventes_nettoyees
)
SELECT
    vendeur,
    region_finale,
    trimestre,
    montant_final,
    commission_finale,
    bonus_final,

    -- Total rémunération
    montant_final + commission_finale + bonus_final as remuneration_totale,

    -- Catégorisation des performances avec CASE
    CASE
        WHEN montant_final >= 2000 THEN '🏆 Excellence'
        WHEN montant_final >= 1600 THEN '🌟 Très bon'
        WHEN montant_final >= 1200 THEN '✅ Satisfaisant'
        WHEN montant_final >= 800 THEN '⚠️ À améliorer'
        ELSE '❌ Insuffisant'
    END as categorie_performance,

    -- Calcul de prime avec règles complexes
    CASE
        -- Prime excellence : 15% du montant + bonus fixe
        WHEN montant_final >= 2000 THEN
            ROUND(montant_final * 0.15 + 300, 2)
        -- Prime très bon : 10% du montant + bonus variable selon région
        WHEN montant_final >= 1600 THEN
            ROUND(montant_final * 0.10 +
                CASE region_finale
                    WHEN 'Sud' THEN 200  -- Région difficile
                    WHEN 'Nord' THEN 150
                    ELSE 100
                END, 2)
        -- Prime satisfaisant : 5% du montant
        WHEN montant_final >= 1200 THEN
            ROUND(montant_final * 0.05, 2)
        -- Pas de prime en dessous
        ELSE 0
    END as prime_calculee,

    -- Analyse par rapport à la moyenne régionale
    CASE
        WHEN montant_final > (
            SELECT AVG(montant_final)
            FROM ventes_completes vc2
            WHERE vc2.region_finale = ventes_completes.region_finale
        ) THEN '📈 Au-dessus moyenne régionale'
        ELSE '📉 En-dessous moyenne régionale'
    END as position_regionale,

    -- Recommandations personnalisées
    CASE
        WHEN montant_final >= 2000 AND commission_finale / montant_final < 0.08 THEN
            '💰 Excellent vendeur - Revoir structure commission'
        WHEN montant_final < 1000 AND region_finale = 'Sud' THEN
            '🎯 Formation spécialisée région Sud requise'
        WHEN bonus_final = 0 AND montant_final >= 1500 THEN
            '🎁 Éligible programme bonus'
        WHEN trimestre = 1 AND montant_final < 1200 THEN
            '📈 Plan d''amélioration T2 nécessaire'
        WHEN montant_final >= 1600 THEN
            '✨ Candidat programme mentorat'
        ELSE '📋 Suivi standard'
    END as recommandation
FROM ventes_completes  
ORDER BY remuneration_totale DESC;  
```
</details>

## Bonnes pratiques et astuces

### ✅ **Bonnes pratiques CASE :**

1. **Ordre des conditions** : Mettez les conditions les plus spécifiques en premier

```text
-- ✅ BON : du plus spécifique au plus général
CASE
    WHEN prix IS NULL THEN 'Prix manquant'
    WHEN prix = 0 THEN 'Gratuit'
    WHEN prix < 10 THEN 'Très abordable'
    WHEN prix < 30 THEN 'Abordable'
    ELSE 'Cher'
END

-- ❌ MAUVAIS : condition générale en premier
CASE
    WHEN prix < 30 THEN 'Abordable'  -- Capturera aussi les prix < 10
    WHEN prix < 10 THEN 'Très abordable'  -- Ne sera jamais atteint !
    ELSE 'Cher'
END
```

2. **Toujours prévoir un ELSE** pour éviter les NULL inattendus

```text
-- ✅ BON
CASE
    WHEN condition1 THEN 'Résultat1'
    WHEN condition2 THEN 'Résultat2'
    ELSE 'Cas non prévu'  -- Sécurité
END

-- ⚠️ ATTENTION : peut retourner NULL
CASE
    WHEN condition1 THEN 'Résultat1'
    WHEN condition2 THEN 'Résultat2'
    -- Pas de ELSE = NULL si aucune condition
END
```

### ✅ **Bonnes pratiques COALESCE :**

1. **Ordre logique** : du plus spécifique au plus général

```text
-- ✅ BON : ordre logique
COALESCE(
    salaire_reel,           -- Valeur précise
    salaire_estime,         -- Estimation
    salaire_par_defaut,     -- Valeur par défaut
    25000                   -- Minimum absolu
)

-- ❌ MOINS BON : valeur par défaut en premier
COALESCE(25000, salaire_reel)  -- Retournera toujours 25000 !
```

2. **Types de données cohérents**

```text
-- ✅ BON : même type
COALESCE(prix_actuel, prix_precedent, 0.0)

-- ❌ ATTENTION : mélange de types
COALESCE(prix_actuel, 'Prix manquant', 0)  -- Problème de types !
```

### ✅ **Bonnes pratiques NULLIF :**

1. **Nettoyage systématique**

```sql
-- ✅ BON : nettoyer avant traitement
WITH donnees_propres AS (
    SELECT
        NULLIF(TRIM(nom), '') as nom_propre,
        NULLIF(salaire, 0) as salaire_valide,
        NULLIF(TRIM(departement), '') as dept_propre
    FROM employes
)
SELECT * FROM donnees_propres WHERE nom_propre IS NOT NULL;
```

2. **Éviter les divisions par zéro**

```text
-- ✅ BON : sécurisé (suppose une table `statistiques(total_ventes, nb_clients)`)
SELECT
    total_ventes,
    nb_clients,
    ROUND(total_ventes / NULLIF(nb_clients, 0), 2) as moyenne_par_client
FROM statistiques;

-- ⚠️ NULL SILENCIEUX : SQLite ne lève PAS d'erreur sur `x / 0` (≠ PostgreSQL/MySQL en mode strict),
--    il retourne `NULL`. Sans NULLIF, ce NULL « anormal » se mélange aux NULL « légitimes » et
--    pollue les agrégations. Utiliser NULLIF rend l'intention explicite et permet ensuite de
--    traiter le cas via COALESCE.
SELECT total_ventes / nb_clients FROM statistiques;  -- NULL si nb_clients = 0, sans avertissement
```

### 🚫 **Erreurs courantes à éviter :**

1. **Confusion entre NULL et chaîne vide**

```sql
-- ⚠️ ATTENTION : '' n'est pas NULL !
SELECT COALESCE('', 'valeur_defaut');  -- Retourne '' pas 'valeur_defaut'
```

```text
-- ✅ SOLUTION : nettoyer d'abord (sur n'importe quelle colonne TEXT `champ`)
SELECT COALESCE(NULLIF(TRIM(champ), ''), 'valeur_defaut') FROM ma_table;
```

2. **CASE sans ELSE dans des calculs**

```text
-- ❌ DANGEREUX : peut créer des NULL inattendus
SELECT
    prix * CASE WHEN categorie = 'Premium' THEN 1.2 END as prix_ajuste
FROM produits;  -- NULL pour les non-Premium !

-- ✅ CORRECT
SELECT
    prix * CASE WHEN categorie = 'Premium' THEN 1.2 ELSE 1.0 END as prix_ajuste
FROM produits;
```

## Techniques avancées de combinaison

### 1. Pipeline de nettoyage complet

```sql
-- Exemple de nettoyage de données en pipeline
-- (la table `employes` du 4.5 n'a pas de colonne `email` ; on travaille sur les colonnes existantes)
WITH etape1_nullif AS (
    -- Étape 1 : Convertir les "fausses" valeurs en NULL
    SELECT
        id,
        NULLIF(TRIM(nom), '') as nom,
        NULLIF(TRIM(prenom), '') as prenom,
        NULLIF(salaire, 0) as salaire,
        NULLIF(TRIM(departement), '') as departement
    FROM employes
),
etape2_coalesce AS (
    -- Étape 2 : Remplacer les NULL par des valeurs par défaut intelligentes
    -- On garde aussi les colonnes originales (nom, prenom, salaire, departement)
    -- pour le calcul du score qualité dans l'étape 3.
    SELECT
        id,
        nom, prenom, salaire, departement,
        COALESCE(nom, 'Nom_' || id) as nom_final,
        COALESCE(prenom, 'Prénom_' || id) as prenom_final,
        COALESCE(
            salaire,
            CASE COALESCE(departement, 'Inconnu')
                WHEN 'IT' THEN 40000
                WHEN 'Marketing' THEN 35000
                WHEN 'Finance' THEN 42000
                ELSE 30000
            END
        ) as salaire_final,
        COALESCE(departement, 'Non assigné') as departement_final
    FROM etape1_nullif
)
-- Étape 3 : Catégorisation avec CASE
SELECT
    id,
    nom_final,
    prenom_final,
    nom_final || ' ' || prenom_final as nom_complet,
    salaire_final,
    departement_final,

    -- Classification salariale
    CASE
        WHEN salaire_final >= 50000 THEN '💎 Senior'
        WHEN salaire_final >= 40000 THEN '⚡ Confirmé'
        WHEN salaire_final >= 30000 THEN '📈 Junior'
        ELSE '🌱 Stagiaire'
    END as niveau_salaire,

    -- Score de qualité des données originales
    CASE
        WHEN nom IS NOT NULL AND prenom IS NOT NULL
         AND salaire IS NOT NULL AND departement IS NOT NULL THEN 100
        WHEN nom IS NOT NULL AND salaire IS NOT NULL THEN 75
        WHEN nom IS NOT NULL THEN 50
        ELSE 25
    END as score_qualite_donnees
FROM etape2_coalesce  
ORDER BY score_qualite_donnees DESC, salaire_final DESC;  
```

### 2. Système de scoring complexe (RFM)

> 💡 **Modèle RFM** (Recency / Frequency / Monetary) : approche classique de segmentation client basée sur trois dimensions — quand a-t-il commandé pour la dernière fois (récence), à quelle fréquence, et pour combien. Nous allons le construire **en 3 étapes** pour rester lisible.

#### Étape 1 — Agréger les indicateurs bruts par client

Première CTE : pour chaque client, on calcule les agrégats nécessaires au scoring. Le `LEFT JOIN` garantit qu'un client sans commande apparaît quand même (avec `nb_commandes = 0`).

```sql
WITH scoring_base AS (
    SELECT
        CL.id,
        CL.nom,
        COUNT(C.id) as nb_commandes,
        COALESCE(SUM(L.prix * C.quantite), 0) as total_achats,
        COALESCE(AVG(L.prix * C.quantite), 0) as panier_moyen,
        MAX(C.date_commande) as derniere_commande,
        MIN(C.date_commande) as premiere_commande
    FROM clients CL
    LEFT JOIN commandes C ON CL.id = C.client_id
    LEFT JOIN livres L ON C.livre_id = L.id
    GROUP BY CL.id, CL.nom
)
SELECT * FROM scoring_base;   -- 👀 Sortie typique : 1 ligne par client avec ses agrégats
```

#### Étape 2 — Convertir chaque agrégat en score partiel (CASE)

Deuxième CTE : on transforme chaque indicateur brut en score sur une échelle fixe (0 → max). Chaque CASE encode une **règle métier**. La somme maximale = 30 + 40 + 20 + 10 = **100 points**.

```sql
WITH scoring_base AS (
    -- (… identique à l'étape 1 …)
    SELECT CL.id, CL.nom, COUNT(C.id) as nb_commandes,
           COALESCE(SUM(L.prix * C.quantite), 0) as total_achats,
           COALESCE(AVG(L.prix * C.quantite), 0) as panier_moyen,
           MAX(C.date_commande) as derniere_commande,
           MIN(C.date_commande) as premiere_commande
    FROM clients CL
    LEFT JOIN commandes C ON CL.id = C.client_id
    LEFT JOIN livres L ON C.livre_id = L.id
    GROUP BY CL.id, CL.nom
),
scoring_detaille AS (
    SELECT
        *,
        -- Score fréquence (0-30 points) : combien de commandes ?
        CASE
            WHEN nb_commandes >= 10 THEN 30
            WHEN nb_commandes >= 5  THEN 25
            WHEN nb_commandes >= 3  THEN 20
            WHEN nb_commandes >= 2  THEN 15
            WHEN nb_commandes  = 1  THEN 10
            ELSE 0
        END as score_frequence,

        -- Score montant (0-40 points) : combien dépensé au total ?
        CASE
            WHEN total_achats >= 500 THEN 40
            WHEN total_achats >= 300 THEN 35
            WHEN total_achats >= 200 THEN 30
            WHEN total_achats >= 100 THEN 25
            WHEN total_achats >= 50  THEN 20
            WHEN total_achats  > 0   THEN 15
            ELSE 0
        END as score_montant,

        -- Score récence (0-20 points) : combien de jours depuis la dernière commande ?
        CASE
            WHEN derniere_commande IS NULL                                  THEN 0
            WHEN julianday('now') - julianday(derniere_commande) <= 30      THEN 20
            WHEN julianday('now') - julianday(derniere_commande) <= 90      THEN 15
            WHEN julianday('now') - julianday(derniere_commande) <= 180     THEN 10
            WHEN julianday('now') - julianday(derniere_commande) <= 365     THEN 5
            ELSE 0
        END as score_recence,

        -- Score fidélité (0-10 points) : combien de temps depuis la 1ʳᵉ commande ?
        CASE
            WHEN premiere_commande IS NULL                                  THEN 0
            WHEN julianday('now') - julianday(premiere_commande) >= 730     THEN 10
            WHEN julianday('now') - julianday(premiere_commande) >= 365     THEN 8
            WHEN julianday('now') - julianday(premiere_commande) >= 180     THEN 6
            WHEN julianday('now') - julianday(premiere_commande) >= 90      THEN 4
            ELSE 2
        END as score_fidelite
    FROM scoring_base
)
SELECT nom, score_frequence, score_montant, score_recence, score_fidelite  
FROM scoring_detaille;   -- 👀 Sortie : 4 scores partiels par client  
```

#### Étape 3 — Sommer, segmenter, et recommander une action

Troisième et dernière CTE : on additionne les 4 scores pour obtenir le **score_total** (sur 100), puis on l'utilise pour segmenter le client et choisir une action marketing.

> ⚠️ Pourquoi une 3ᵉ CTE pour le `score_total` ? **En SQLite, on ne peut pas réutiliser un alias dans le même `SELECT`** (sauf en `ORDER BY`/`GROUP BY`). Sans cette CTE, on serait obligé de réécrire la somme entière à chaque utilisation.

```sql
WITH scoring_base AS (
    -- (… identique aux étapes précédentes …)
    SELECT CL.id, CL.nom, COUNT(C.id) as nb_commandes,
           COALESCE(SUM(L.prix * C.quantite), 0) as total_achats,
           COALESCE(AVG(L.prix * C.quantite), 0) as panier_moyen,
           MAX(C.date_commande) as derniere_commande,
           MIN(C.date_commande) as premiere_commande
    FROM clients CL
    LEFT JOIN commandes C ON CL.id = C.client_id
    LEFT JOIN livres L ON C.livre_id = L.id
    GROUP BY CL.id, CL.nom
),
scoring_detaille AS (
    -- (… identique à l'étape 2, 4 scores partiels …)
    SELECT *,
        CASE WHEN nb_commandes >= 10 THEN 30 WHEN nb_commandes >= 5 THEN 25
             WHEN nb_commandes >= 3  THEN 20 WHEN nb_commandes >= 2 THEN 15
             WHEN nb_commandes  = 1  THEN 10 ELSE 0 END as score_frequence,
        CASE WHEN total_achats >= 500 THEN 40 WHEN total_achats >= 300 THEN 35
             WHEN total_achats >= 200 THEN 30 WHEN total_achats >= 100 THEN 25
             WHEN total_achats >= 50  THEN 20 WHEN total_achats > 0   THEN 15 ELSE 0 END as score_montant,
        CASE WHEN derniere_commande IS NULL THEN 0
             WHEN julianday('now') - julianday(derniere_commande) <= 30  THEN 20
             WHEN julianday('now') - julianday(derniere_commande) <= 90  THEN 15
             WHEN julianday('now') - julianday(derniere_commande) <= 180 THEN 10
             WHEN julianday('now') - julianday(derniere_commande) <= 365 THEN 5 ELSE 0 END as score_recence,
        CASE WHEN premiere_commande IS NULL THEN 0
             WHEN julianday('now') - julianday(premiere_commande) >= 730 THEN 10
             WHEN julianday('now') - julianday(premiere_commande) >= 365 THEN 8
             WHEN julianday('now') - julianday(premiere_commande) >= 180 THEN 6
             WHEN julianday('now') - julianday(premiere_commande) >= 90  THEN 4 ELSE 2 END as score_fidelite
    FROM scoring_base
),
scoring_avec_total AS (
    -- CTE dédiée juste à la somme, pour pouvoir l'utiliser dans CASE / ORDER BY
    SELECT *,
           score_frequence + score_montant + score_recence + score_fidelite AS score_total
    FROM scoring_detaille
)
SELECT
    nom, nb_commandes, ROUND(total_achats, 2) as total_achats,
    score_frequence, score_montant, score_recence, score_fidelite,
    score_total,
    -- Segmentation
    CASE
        WHEN score_total >= 80 THEN '👑 Champion'
        WHEN score_total >= 60 THEN '🌟 Fidèle'
        WHEN score_total >= 40 THEN '✅ Régulier'
        WHEN score_total >= 20 THEN '📈 Potentiel'
        WHEN score_total >  0  THEN '🌱 Nouveau'
        ELSE '😴 Inactif'
    END as segment_client,
    -- Action marketing
    CASE
        WHEN score_recence = 0 AND score_frequence > 15 THEN '📧 Campagne de réactivation'
        WHEN score_montant >= 30 AND score_frequence < 20 THEN '🎁 Offre fidélité'
        WHEN score_total >= 70 THEN '🏆 Programme VIP'
        WHEN score_total >= 40 THEN '📈 Upselling'
        WHEN score_total >= 20 THEN '💌 Newsletter engagement'
        ELSE '🎯 Acquisition'
    END as action_recommandee
FROM scoring_avec_total  
ORDER BY score_total DESC;  
```

> 🎓 **Récapitulatif pédagogique** :  
> 1. **CTE 1 (`scoring_base`)** → agrégats bruts (`nb_commandes`, `total_achats`, dates extrêmes)  
> 2. **CTE 2 (`scoring_detaille`)** → 4 scores partiels via `CASE` (échelles métier)  
> 3. **CTE 3 (`scoring_avec_total`)** → somme + segmentation + recommandation  
>  
> Cette décomposition rend le code lisible, **testable étape par étape** (chaque CTE peut être exécutée seule pour vérifier sa sortie), et facile à maintenir : on peut ajuster les seuils d'une étape sans toucher aux autres.

## Résumé

Les fonctions **CASE**, **COALESCE** et **NULLIF** sont les piliers de la logique conditionnelle en SQL :

### 🎯 **CASE - La logique conditionnelle :**
- **If-else de SQL** pour appliquer différentes logiques
- **Deux syntaxes** : simple (comparaison directe) et complexe (conditions)
- **Essentiel pour** : catégorisation, calculs conditionnels, transformations
- **Toujours prévoir un ELSE** pour éviter les NULL inattendus

### 🔧 **COALESCE - Gestion des NULL :**
- **Première valeur non-NULL** dans une liste d'expressions
- **Parfait pour** : valeurs par défaut, fusion de sources, robustesse
- **Ordre important** : du plus spécifique au plus général
- **Alternative à** : CASE complexes pour gérer les NULL

### 🧹 **NULLIF - Nettoyage des données :**
- **Convertit des valeurs en NULL** si elles sont égales
- **Utile pour** : nettoyer les 0, chaînes vides, valeurs "placeholder"
- **Évite les divisions par zéro** et erreurs de calcul
- **Complément de COALESCE** dans les pipelines de nettoyage

### 💡 **Techniques avancées :**
- **Combinaison des trois** pour des logiques sophistiquées
- **Pipelines de nettoyage** : NULLIF → COALESCE → CASE
- **Scoring et classification** automatiques
- **Rapports intelligents** avec recommandations

### 📋 **Cas d'usage essentiels :**
- **Transformation de données** : catégorisation, normalisation
- **Gestion de la qualité** : nettoyage, validation, complétude
- **Business intelligence** : scoring, segmentation, KPI
- **Rapports dynamiques** : conditions métier, alertes, recommandations

Ces trois fonctions transforment SQL d'un simple langage de requête en un véritable outil de programmation logique. Maîtrisées ensemble, elles permettent de créer des requêtes intelligentes qui s'adaptent aux données et aux besoins métier !

Dans la prochaine section, nous découvrirons comment **interroger et manipuler des données JSON** dans SQLite, une fonctionnalité moderne pour gérer des données semi-structurées.

⏭️ [4.6 Interrogation et manipulation des données JSON](/04-requetes-avancees-optimisation/06-interrogation-manipulation-donnees-json.md)
