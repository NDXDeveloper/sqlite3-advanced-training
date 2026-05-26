🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.5 Requêtes de base : SELECT, WHERE, ORDER BY, GROUP BY, HAVING

## Introduction - L'art d'interroger vos données

Maintenant que vous savez créer des bases de données robustes avec des contraintes appropriées, il est temps d'apprendre à **extraire efficacement les informations** dont vous avez besoin. La requête SELECT est votre outil principal pour "poser des questions" à votre base de données.

> **Analogie simple** : Si votre base de données est une bibliothèque bien organisée, les requêtes SELECT sont comme un bibliothécaire expert qui peut instantanément vous trouver exactement les livres que vous cherchez selon vos critères.

**Les 5 clauses principales que nous allons maîtriser :**
- **SELECT** → Quoi récupérer
- **WHERE** → Quelles conditions
- **ORDER BY** → Dans quel ordre
- **GROUP BY** → Comment regrouper
- **HAVING** → Conditions sur les groupes

## Préparation - Base de données complète

Créons une base de données réaliste de librairie pour nos exemples :

```sql
-- Créer notre base d'exemple
sqlite3 librairie_requetes.db

PRAGMA foreign_keys = ON;

-- Structure complète
CREATE TABLE auteurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    date_naissance TEXT,
    nationalite TEXT,
    date_deces TEXT
);

CREATE TABLE editeurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    ville TEXT,
    pays TEXT DEFAULT 'France',
    annee_creation INTEGER
);

CREATE TABLE categories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL UNIQUE,
    description TEXT
);

CREATE TABLE livres (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    auteur_id INTEGER,
    editeur_id INTEGER,
    categorie_id INTEGER,
    isbn TEXT UNIQUE,
    annee_publication INTEGER,
    pages INTEGER,
    prix REAL,
    stock INTEGER DEFAULT 0,
    note_moyenne REAL CHECK (note_moyenne BETWEEN 1 AND 5),
    nombre_avis INTEGER DEFAULT 0,
    FOREIGN KEY (auteur_id) REFERENCES auteurs(id),
    FOREIGN KEY (editeur_id) REFERENCES editeurs(id),
    FOREIGN KEY (categorie_id) REFERENCES categories(id)
);

CREATE TABLE ventes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    livre_id INTEGER,
    date_vente TEXT DEFAULT (date('now')),
    quantite INTEGER DEFAULT 1,
    prix_unitaire REAL,
    client_nom TEXT,
    FOREIGN KEY (livre_id) REFERENCES livres(id)
);

-- === INSERTION DES DONNÉES D'EXEMPLE ===

-- Auteurs
INSERT INTO auteurs (nom, prenom, date_naissance, nationalite, date_deces) VALUES
    ('Hugo', 'Victor', '1802-02-26', 'Française', '1885-05-22'),
    ('Camus', 'Albert', '1913-11-07', 'Française', '1960-01-04'),
    ('Orwell', 'George', '1903-06-25', 'Britannique', '1950-01-21'),
    ('Tolkien', 'J.R.R.', '1892-01-03', 'Britannique', '1973-09-02'),
    ('Rowling', 'J.K.', '1965-07-31', 'Britannique', NULL),
    ('García Márquez', 'Gabriel', '1927-03-06', 'Colombienne', '2014-04-17'),
    ('Murakami', 'Haruki', '1949-01-12', 'Japonaise', NULL),
    ('Eco', 'Umberto', '1932-01-05', 'Italienne', '2016-02-19');

-- Éditeurs
INSERT INTO editeurs (nom, ville, pays, annee_creation) VALUES
    ('Gallimard', 'Paris', 'France', 1911),
    ('Le Seuil', 'Paris', 'France', 1935),
    ('Flammarion', 'Paris', 'France', 1875),
    ('Penguin Books', 'Londres', 'Royaume-Uni', 1935),
    ('Bloomsbury', 'Londres', 'Royaume-Uni', 1986),
    ('Random House', 'New York', 'États-Unis', 1927);

-- Catégories
INSERT INTO categories (nom, description) VALUES
    ('Roman', 'Littérature romanesque'),
    ('Fantasy', 'Littérature fantastique et imaginaire'),
    ('Science-Fiction', 'Anticipation et fiction scientifique'),
    ('Dystopie', 'Romans d''anticipation pessimiste'),
    ('Réalisme magique', 'Romans mêlant réalité et éléments magiques'),
    ('Philosophie', 'Ouvrages philosophiques'),
    ('Biographie', 'Récits de vie'),
    ('Essai', 'Textes d''analyse et de réflexion');

-- Livres avec données variées
INSERT INTO livres (titre, auteur_id, editeur_id, categorie_id, annee_publication, pages, prix, stock, note_moyenne, nombre_avis) VALUES
    ('Les Misérables', 1, 1, 1, 1862, 1488, 15.90, 12, 4.5, 156),
    ('Notre-Dame de Paris', 1, 1, 1, 1831, 512, 12.50, 8, 4.2, 89),
    ('L''Étranger', 2, 1, 1, 1942, 186, 8.90, 15, 4.3, 234),
    ('La Peste', 2, 2, 1, 1947, 279, 9.50, 10, 4.4, 167),
    ('1984', 3, 4, 4, 1949, 415, 10.20, 20, 4.7, 512),
    ('La Ferme des animaux', 3, 4, 4, 1945, 144, 7.80, 25, 4.1, 298),
    ('Le Seigneur des anneaux', 4, 4, 2, 1954, 1216, 25.00, 6, 4.8, 1024),
    ('Bilbo le Hobbit', 4, 4, 2, 1937, 360, 12.90, 14, 4.6, 456),
    ('Harry Potter à l''école des sorciers', 5, 5, 2, 1997, 320, 8.99, 30, 4.7, 2156),
    ('Harry Potter et la chambre des secrets', 5, 5, 2, 1998, 364, 9.99, 22, 4.6, 1834),
    ('Cent ans de solitude', 6, 6, 5, 1967, 417, 13.50, 9, 4.4, 342),
    ('Chronique d''une mort annoncée', 6, 2, 5, 1981, 122, 8.50, 12, 4.2, 156),
    ('Norwegian Wood', 7, 2, 1, 1987, 296, 11.50, 18, 4.1, 267),
    ('Kafka sur le rivage', 7, 2, 5, 2002, 615, 14.90, 11, 4.3, 189),
    ('Le Nom de la rose', 8, 1, 1, 1980, 624, 16.50, 7, 4.5, 423);

-- Ventes avec dates variées
INSERT INTO ventes (livre_id, date_vente, quantite, prix_unitaire, client_nom) VALUES
    (1, '2024-01-15', 2, 15.90, 'Marie Dupont'),
    (5, '2024-01-16', 1, 10.20, 'Jean Martin'),
    (9, '2024-01-17', 3, 8.99, 'Sophie Leroy'),
    (1, '2024-01-18', 1, 15.90, 'Pierre Durand'),
    (7, '2024-01-20', 1, 25.00, 'Alice Bernard'),
    (5, '2024-01-22', 2, 10.20, 'Paul Moreau'),
    (9, '2024-01-25', 1, 8.99, 'Julie Petit'),
    (10, '2024-01-28', 2, 9.99, 'Marc Dubois'),
    (3, '2024-02-01', 1, 8.90, 'Claire Martin'),
    (11, '2024-02-03', 1, 13.50, 'Thomas Leroy'),
    (15, '2024-02-05', 1, 16.50, 'Emma Durand'),
    (7, '2024-02-08', 1, 25.00, 'Nicolas Blanc'),
    (2, '2024-02-10', 2, 12.50, 'Léa Moreau'),
    (13, '2024-02-12', 1, 11.50, 'Antoine Roux'),
    (6, '2024-02-15', 3, 7.80, 'Camille Dubois');

-- Vérifier notre structure
.tables
SELECT COUNT(*) as nb_auteurs FROM auteurs;  
SELECT COUNT(*) as nb_livres FROM livres;  
SELECT COUNT(*) as nb_ventes FROM ventes;  
```

## SELECT - Choisir les données à récupérer

### 🎯 SELECT de base

La clause SELECT détermine **quelles colonnes** vous voulez récupérer :

```sql
-- Sélectionner toutes les colonnes
SELECT * FROM livres;

-- Sélectionner des colonnes spécifiques
SELECT titre, prix FROM livres;

-- Sélectionner avec alias pour plus de clarté
SELECT
    titre AS "Titre du livre",
    prix AS "Prix en euros",
    pages AS "Nombre de pages"
FROM livres;

-- Sélectionner avec calculs
SELECT
    titre,
    prix,
    pages,
    ROUND(prix / pages * 100, 2) AS "Centimes par page"
FROM livres;
-- 💡 Ici prix est REAL et pages est INTEGER : REAL/INTEGER = REAL.
-- Si les deux étaient INTEGER, il faudrait écrire `prix * 1.0 / pages` pour éviter
-- la troncature (cf. piège INTEGER/INTEGER vu en 2.1).
```

### 🔢 SELECT avec fonctions et expressions

```sql
-- Fonctions de chaînes
SELECT
    UPPER(titre) AS titre_majuscules,
    LENGTH(titre) AS longueur_titre,
    SUBSTR(titre, 1, 20) || '...' AS titre_court
FROM livres  
WHERE LENGTH(titre) > 20;  

-- Fonctions numériques
SELECT
    titre,
    prix,
    ROUND(prix, 0) AS prix_arrondi,
    ABS(prix - 12.00) AS ecart_prix_moyen,
    MIN(prix, 15.00) AS prix_plafonne   -- MIN scalaire (deux args) ≠ MIN agrégée
FROM livres;
-- 💡 SQLite a deux MIN/MAX :
--    - **scalaires** : MIN(a, b, c, …) → la plus petite des valeurs passées sur la ligne
--    - **agrégées** : MIN(col) → la plus petite valeur de col sur toutes les lignes du groupe
-- La distinction se fait par le nombre d'arguments.

-- Fonctions de dates
SELECT
    titre,
    annee_publication,
    -- Âge dynamique calculé par rapport à l'année courante
    CAST(strftime('%Y', 'now') AS INTEGER) - annee_publication AS age_livre,
    CASE
        WHEN annee_publication >= 2000 THEN 'Récent'
        WHEN annee_publication >= 1950 THEN 'Moderne'
        ELSE 'Classique'
    END AS periode
FROM livres;
```

### 💡 SELECT avec CASE (conditions)

```sql
-- Classification des livres par prix
SELECT
    titre,
    prix,
    CASE
        WHEN prix < 10 THEN 'Économique'
        WHEN prix BETWEEN 10 AND 15 THEN 'Standard'
        WHEN prix BETWEEN 15 AND 20 THEN 'Premium'
        ELSE 'Luxe'
    END AS gamme_prix,
    CASE
        WHEN stock = 0 THEN '❌ Rupture'
        WHEN stock < 5 THEN '⚠️ Stock faible'
        WHEN stock < 15 THEN '✅ Disponible'
        ELSE '📦 Stock important'
    END AS statut_stock
FROM livres;

-- Recommandations basées sur les notes
-- 💡 Important : l'ordre des WHEN compte ! Le premier qui matche est retenu.
-- Ici on filtre les cas "peu d'avis" en premier pour qu'ils ne soient pas
-- masqués par les conditions suivantes.
SELECT
    titre,
    note_moyenne,
    nombre_avis,
    CASE
        WHEN nombre_avis < 10 THEN '❓ Peu d''avis'                              -- en premier
        WHEN note_moyenne >= 4.5 AND nombre_avis >= 100 THEN '⭐ Bestseller'
        WHEN note_moyenne >= 4.0 AND nombre_avis >= 50  THEN '👍 Recommandé'
        WHEN note_moyenne >= 3.5                        THEN '✅ Correct'
        ELSE '👎 À éviter'
    END AS recommendation
FROM livres  
WHERE note_moyenne IS NOT NULL;  
```

## WHERE - Filtrer les données

### 🔍 Filtres simples

```sql
-- Filtres de base
SELECT titre, prix FROM livres WHERE prix < 10;

SELECT titre, auteur_id FROM livres WHERE auteur_id = 5;  -- J.K. Rowling

SELECT * FROM livres WHERE stock = 0;  -- Livres en rupture

-- Filtres avec opérateurs de comparaison
SELECT titre, annee_publication  
FROM livres  
WHERE annee_publication >= 2000;  

SELECT titre, pages  
FROM livres  
WHERE pages BETWEEN 200 AND 500;  

-- Filtres sur texte
SELECT * FROM auteurs WHERE nationalite = 'Française';

SELECT * FROM livres WHERE titre LIKE '%Harry Potter%';

SELECT * FROM auteurs WHERE nom LIKE 'M%';  -- Noms commençant par M
```

### 🔗 Filtres combinés avec AND, OR, NOT

```sql
-- Combinaisons avec AND
SELECT titre, prix, stock  
FROM livres  
WHERE prix < 15 AND stock > 10;  

SELECT titre, note_moyenne, nombre_avis  
FROM livres  
WHERE note_moyenne >= 4.5 AND nombre_avis >= 100;  

-- Combinaisons avec OR
SELECT titre, categorie_id  
FROM livres  
WHERE categorie_id = 2 OR categorie_id = 4;  -- Fantasy ou Dystopie  

SELECT * FROM auteurs  
WHERE nationalite = 'Française' OR nationalite = 'Britannique';  

-- Utilisation de NOT
SELECT titre, stock  
FROM livres  
WHERE NOT (stock = 0);  -- Équivalent à stock != 0  

SELECT * FROM auteurs  
WHERE date_deces IS NOT NULL;  -- Auteurs décédés  

-- Parenthèses pour la priorité
SELECT titre, prix, stock, categorie_id  
FROM livres  
WHERE (prix < 10 OR stock > 20) AND categorie_id IN (1, 2);  
```

### 📋 Filtres avec IN, EXISTS, NULL

```sql
-- Filtre avec IN (liste de valeurs)
SELECT titre, auteur_id  
FROM livres  
WHERE auteur_id IN (1, 3, 5);  -- Hugo, Orwell, Rowling  

SELECT nom FROM editeurs  
WHERE pays IN ('France', 'Royaume-Uni');  

-- Filtre avec EXISTS (sous-requête)
SELECT a.nom, a.prenom  
FROM auteurs a  
WHERE EXISTS (  
    SELECT 1 FROM livres l
    WHERE l.auteur_id = a.id AND l.prix > 20
);

-- Gestion des valeurs NULL
SELECT titre, note_moyenne  
FROM livres  
WHERE note_moyenne IS NULL;  

SELECT nom, date_deces  
FROM auteurs  
WHERE date_deces IS NOT NULL;  

-- Remplacer NULL par une valeur par défaut
SELECT
    nom,
    COALESCE(date_deces, 'Vivant') AS statut
FROM auteurs;
```

### 🎯 Filtres avancés avec fonctions

```sql
-- Filtres sur des calculs
SELECT titre, prix, pages, ROUND(prix/pages*100, 2) AS cout_par_page  
FROM livres  
WHERE prix/pages*100 < 5;  -- Moins de 5 centimes par page  

-- Filtres sur des fonctions de texte
SELECT * FROM livres  
WHERE LENGTH(titre) > 25;  

SELECT * FROM auteurs  
WHERE UPPER(nom) LIKE '%GAR%';  

-- Filtres sur des fonctions de date (utiliser l'année courante dynamiquement)
SELECT titre, annee_publication,
       CAST(strftime('%Y', 'now') AS INTEGER) - annee_publication AS age
FROM livres  
WHERE CAST(strftime('%Y', 'now') AS INTEGER) - annee_publication < 30;  -- Moins de 30 ans  

-- Filtres complexes
SELECT
    l.titre,
    a.nom,
    l.prix,
    l.note_moyenne
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
WHERE l.prix BETWEEN 8 AND 15  
    AND l.note_moyenne >= 4.0
    AND a.nationalite != 'Française'
    AND l.stock > 5;
```

## DISTINCT - Éliminer les doublons

### 🎯 SELECT DISTINCT

`DISTINCT` retire les lignes en double du résultat. Attention, le doublon est mesuré sur **toutes les colonnes du SELECT**, pas seulement la première.

```sql
-- Toutes les nationalités d'auteurs (avec doublons)
SELECT nationalite FROM auteurs;
-- Résultat : Française, Française, Britannique, Britannique, Britannique, Colombienne, ...

-- Nationalités UNIQUES
SELECT DISTINCT nationalite FROM auteurs;
-- Résultat : Française, Britannique, Colombienne, Japonaise, Italienne

-- ⚠️ DISTINCT s'applique à la combinaison de TOUTES les colonnes
SELECT DISTINCT nationalite, prenom FROM auteurs;
-- → un (nationalite, prenom) unique = une ligne

-- DISTINCT dans un agrégat : compter les valeurs DISTINCTES
SELECT
    COUNT(*)               AS total_livres,
    COUNT(DISTINCT auteur_id) AS nb_auteurs_differents
FROM livres;
```

> 💡 **DISTINCT vs GROUP BY** : pour un simple « valeurs uniques », les deux marchent. `SELECT DISTINCT col FROM t` ≡ `SELECT col FROM t GROUP BY col`. Mais `GROUP BY` permet aussi d'appliquer des agrégats — préférez-le quand vous combinez plusieurs calculs.

## Opérateurs ensemblistes - UNION, INTERSECT, EXCEPT

SQLite supporte les **3 opérateurs ensemblistes** standards (`UNION`, `INTERSECT`, `EXCEPT`) pour combiner les résultats de plusieurs `SELECT`. `UNION` a deux variantes selon le traitement des doublons :

| Opérateur | Effet | Doublons |
|-----------|-------|----------|
| `UNION` | Union de A et B | éliminés |
| `UNION ALL` | Union de A et B | conservés (plus rapide) |
| `INTERSECT` | A ∩ B (lignes communes) | éliminés |
| `EXCEPT` | A \ B (lignes de A absentes de B) | éliminés |

```sql
-- UNION : auteurs français OU britanniques (sans doublon)
SELECT nom, nationalite FROM auteurs WHERE nationalite = 'Française'  
UNION  
SELECT nom, nationalite FROM auteurs WHERE nationalite = 'Britannique';  

-- UNION ALL : conserve les doublons (plus rapide, à utiliser si on SAIT qu'il n'y a pas de doublon)
SELECT 'Livres' AS categorie, COUNT(*) AS total FROM livres  
UNION ALL  
SELECT 'Auteurs', COUNT(*) FROM auteurs  
UNION ALL  
SELECT 'Ventes', COUNT(*) FROM ventes;  

-- INTERSECT : nationalités d'auteurs présentes ET dans la table editeurs.pays
-- (intersection des ensembles de valeurs)
SELECT nationalite FROM auteurs  
INTERSECT  
SELECT pays FROM editeurs;  
-- Résultat : 'Française' uniquement (les autres nationalités ne correspondent à aucun pays d'éditeur)

-- EXCEPT : auteurs dont la nationalité n'a PAS d'éditeur associé
SELECT DISTINCT nationalite FROM auteurs  
EXCEPT  
SELECT pays FROM editeurs;  
```

> ⚠️ **Règles importantes** :  
> - Les `SELECT` combinés doivent avoir **le même nombre de colonnes** et des **types compatibles**.  
> - Les noms de colonnes du **premier** `SELECT` sont utilisés pour le résultat.  
> - **Un seul `ORDER BY`** autorisé, à la **toute fin**, qui s'applique à l'ensemble.  
> - Pour trier avant la combinaison, encapsuler chaque `SELECT` dans une sous-requête.

```sql
-- ❌ INTERDIT : ORDER BY au milieu d'un UNION
SELECT nom FROM auteurs ORDER BY nom  -- erreur !  
UNION  
SELECT titre FROM livres;  

-- ✅ ORDER BY uniquement à la fin
SELECT nom AS valeur FROM auteurs  
UNION  
SELECT titre FROM livres  
ORDER BY valeur;  
```

## ORDER BY - Trier les résultats

### 📊 Tri simple

```sql
-- Tri croissant (par défaut)
SELECT titre, prix FROM livres ORDER BY prix;

-- Tri décroissant
SELECT titre, prix FROM livres ORDER BY prix DESC;

-- Tri alphabétique
SELECT nom, prenom FROM auteurs ORDER BY nom, prenom;

-- Tri par plusieurs colonnes
SELECT titre, annee_publication, prix  
FROM livres  
ORDER BY annee_publication DESC, prix ASC;  
```

### 🎯 Tri avancé

```sql
-- Tri avec calculs
SELECT
    titre,
    prix,
    pages,
    ROUND(prix/pages*100, 2) AS cout_par_page
FROM livres  
ORDER BY cout_par_page DESC;  

-- Tri conditionnel avec CASE
SELECT
    titre,
    stock,
    CASE
        WHEN stock = 0 THEN 1      -- Ruptures en premier
        WHEN stock < 5 THEN 2      -- Stock faible ensuite
        ELSE 3                     -- Autres en dernier
    END AS priorite
FROM livres  
ORDER BY priorite, stock;  

-- Tri avec NULL en dernier
SELECT nom, date_deces  
FROM auteurs  
ORDER BY date_deces DESC NULLS LAST;  

-- Tri par popularité (note et nombre d'avis)
SELECT
    titre,
    note_moyenne,
    nombre_avis,
    (note_moyenne * LOG(nombre_avis + 1)) AS score_popularite
FROM livres  
WHERE note_moyenne IS NOT NULL  
ORDER BY score_popularite DESC  
LIMIT 10;  
```

### 📈 Tri avec jointures

```sql
-- Tri par nom d'auteur
SELECT
    l.titre,
    a.nom || ' ' || a.prenom AS auteur,
    l.prix
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
ORDER BY a.nom, a.prenom, l.titre;  

-- Tri par catégorie puis par prix
SELECT
    l.titre,
    c.nom AS categorie,
    l.prix
FROM livres l  
JOIN categories c ON l.categorie_id = c.id  
ORDER BY c.nom, l.prix DESC;  

-- Top des ventes par chiffre d'affaires
SELECT
    l.titre,
    SUM(v.quantite * v.prix_unitaire) AS ca_total,
    COUNT(v.id) AS nb_ventes
FROM livres l  
JOIN ventes v ON l.id = v.livre_id  
GROUP BY l.id, l.titre  
ORDER BY ca_total DESC;  
```

## GROUP BY - Regrouper les données

### 📊 Regroupements simples

```sql
-- Compter par catégorie
SELECT
    categorie_id,
    COUNT(*) AS nombre_livres
FROM livres  
GROUP BY categorie_id;  

-- Avec les noms de catégories
SELECT
    c.nom AS categorie,
    COUNT(l.id) AS nombre_livres,
    AVG(l.prix) AS prix_moyen
FROM categories c  
LEFT JOIN livres l ON c.id = l.categorie_id  
GROUP BY c.id, c.nom  
ORDER BY nombre_livres DESC;  

-- Statistiques par auteur
SELECT
    a.nom || ' ' || a.prenom AS auteur,
    COUNT(l.id) AS nombre_livres,
    MIN(l.prix) AS prix_min,
    MAX(l.prix) AS prix_max,
    AVG(l.prix) AS prix_moyen,
    SUM(l.pages) AS total_pages
FROM auteurs a  
JOIN livres l ON a.id = l.auteur_id  
GROUP BY a.id, a.nom, a.prenom  
ORDER BY nombre_livres DESC;  
```

### 🔢 Fonctions d'agrégation

```sql
-- Toutes les fonctions d'agrégation principales
SELECT
    COUNT(*) AS total_livres,
    COUNT(DISTINCT auteur_id) AS nb_auteurs_distincts,
    MIN(prix) AS prix_minimum,
    MAX(prix) AS prix_maximum,
    AVG(prix) AS prix_moyen,
    SUM(prix * stock) AS valeur_stock_total,
    ROUND(AVG(note_moyenne), 2) AS note_moyenne_generale
FROM livres  
WHERE prix IS NOT NULL;  

-- Statistiques par décennie
SELECT
    (annee_publication / 10) * 10 AS decennie,
    COUNT(*) AS nombre_livres,
    AVG(pages) AS pages_moyennes,
    AVG(prix) AS prix_moyen
FROM livres  
WHERE annee_publication IS NOT NULL  
GROUP BY (annee_publication / 10) * 10  
ORDER BY decennie;  

-- Répartition par tranches de prix
-- 💡 SQLite (comme PostgreSQL et MySQL) permet de réutiliser l'alias dans GROUP BY,
-- HAVING et ORDER BY (mais PAS dans WHERE, qui s'évalue avant SELECT).
SELECT
    CASE
        WHEN prix < 10 THEN 'Moins de 10€'
        WHEN prix < 15 THEN '10-15€'
        WHEN prix < 20 THEN '15-20€'
        ELSE 'Plus de 20€'
    END AS tranche_prix,
    COUNT(*) AS nombre_livres,
    ROUND(AVG(pages), 0) AS pages_moyennes
FROM livres  
WHERE prix IS NOT NULL  
GROUP BY tranche_prix          -- on réutilise l'alias plutôt que de répéter le CASE  
ORDER BY MIN(prix);  
```

### 📈 Analyses temporelles avec GROUP BY

```sql
-- Ventes par mois
SELECT
    strftime('%Y-%m', date_vente) AS mois,
    COUNT(*) AS nb_ventes,
    SUM(quantite) AS livres_vendus,
    SUM(quantite * prix_unitaire) AS chiffre_affaires,
    AVG(prix_unitaire) AS prix_moyen
FROM ventes  
GROUP BY strftime('%Y-%m', date_vente)  
ORDER BY mois;  

-- Évolution des publications par décennie (alias réutilisé dans GROUP BY)
SELECT
    CASE
        WHEN annee_publication < 1900 THEN 'XIXe siècle'
        WHEN annee_publication < 1950 THEN '1900-1949'
        WHEN annee_publication < 2000 THEN '1950-1999'
        ELSE '2000 et plus'
    END AS periode,
    COUNT(*) AS nombre_livres,
    AVG(pages) AS pages_moyennes,
    GROUP_CONCAT(DISTINCT a.nationalite) AS nationalites
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
WHERE annee_publication IS NOT NULL  
GROUP BY periode  
ORDER BY MIN(annee_publication);  
```

## HAVING - Filtrer les groupes

### 🎯 Différence entre WHERE et HAVING

```sql
-- WHERE filtre AVANT le regroupement
-- HAVING filtre APRÈS le regroupement

-- Exemple avec WHERE (filtre les livres avant regroupement)
SELECT
    categorie_id,
    COUNT(*) AS nb_livres,
    AVG(prix) AS prix_moyen
FROM livres  
WHERE prix > 10  -- Filtre: seulement les livres > 10€  
GROUP BY categorie_id;  

-- Exemple avec HAVING (filtre les groupes après regroupement)
SELECT
    categorie_id,
    COUNT(*) AS nb_livres,
    AVG(prix) AS prix_moyen
FROM livres  
GROUP BY categorie_id  
HAVING COUNT(*) > 2;  -- Filtre: seulement les catégories avec plus de 2 livres  

-- Combinaison WHERE + HAVING
SELECT
    categorie_id,
    COUNT(*) AS nb_livres,
    AVG(prix) AS prix_moyen
FROM livres  
WHERE prix > 5           -- Filtre d'abord les livres > 5€  
GROUP BY categorie_id  
HAVING COUNT(*) > 1      -- Puis garde seulement les catégories avec plus d'1 livre  
ORDER BY prix_moyen DESC;  
```

### 📊 HAVING avec fonctions d'agrégation

```sql
-- Auteurs prolifiques (plus de 2 livres)
SELECT
    a.nom || ' ' || a.prenom AS auteur,
    COUNT(l.id) AS nombre_livres,
    AVG(l.prix) AS prix_moyen,
    SUM(l.pages) AS total_pages
FROM auteurs a  
JOIN livres l ON a.id = l.auteur_id  
GROUP BY a.id, a.nom, a.prenom  
HAVING COUNT(l.id) > 1  -- Plus d'un livre  
ORDER BY nombre_livres DESC;  

-- Catégories rentables (prix moyen élevé)
SELECT
    c.nom AS categorie,
    COUNT(l.id) AS nb_livres,
    AVG(l.prix) AS prix_moyen,
    MAX(l.prix) AS prix_max
FROM categories c  
JOIN livres l ON c.id = l.categorie_id  
GROUP BY c.id, c.nom  
HAVING AVG(l.prix) > 12  -- Prix moyen > 12€  
ORDER BY prix_moyen DESC;  

-- Éditeurs avec gros catalogue
SELECT
    e.nom AS editeur,
    e.ville,
    COUNT(l.id) AS nb_livres,
    AVG(l.note_moyenne) AS note_moyenne,
    SUM(l.stock * l.prix) AS valeur_stock
FROM editeurs e  
JOIN livres l ON e.id = l.editeur_id  
GROUP BY e.id, e.nom, e.ville  
HAVING COUNT(l.id) >= 2 AND AVG(l.note_moyenne) >= 4.0  
ORDER BY nb_livres DESC;  
```

### 🔍 HAVING avec conditions complexes

```sql
-- Auteurs avec des livres bien notés ET bien vendus
SELECT
    a.nom || ' ' || a.prenom AS auteur,
    COUNT(DISTINCT l.id) AS nb_livres,
    AVG(l.note_moyenne) AS note_moyenne,
    COUNT(DISTINCT v.id) AS nb_ventes,
    SUM(v.quantite * v.prix_unitaire) AS ca_total
FROM auteurs a  
JOIN livres l ON a.id = l.auteur_id  
LEFT JOIN ventes v ON l.id = v.livre_id  
WHERE l.note_moyenne IS NOT NULL  
GROUP BY a.id, a.nom, a.prenom  
HAVING AVG(l.note_moyenne) > 4.0 AND COUNT(DISTINCT v.id) > 0  
ORDER BY ca_total DESC;  

-- Périodes productives (beaucoup de livres ET diversité des auteurs)
SELECT
    (annee_publication / 10) * 10 AS decennie,
    COUNT(*) AS nb_livres,
    COUNT(DISTINCT auteur_id) AS nb_auteurs,
    AVG(pages) AS pages_moyennes,
    AVG(prix) AS prix_moyen
FROM livres  
WHERE annee_publication IS NOT NULL  
GROUP BY (annee_publication / 10) * 10  
HAVING COUNT(*) >= 2 AND COUNT(DISTINCT auteur_id) >= 2  
ORDER BY decennie DESC;  

-- Mois avec bonnes ventes (volume ET chiffre d'affaires)
SELECT
    strftime('%Y-%m', date_vente) AS mois,
    COUNT(*) AS nb_transactions,
    SUM(quantite) AS livres_vendus,
    SUM(quantite * prix_unitaire) AS ca,
    AVG(quantite * prix_unitaire) AS panier_moyen
FROM ventes  
GROUP BY strftime('%Y-%m', date_vente)  
HAVING SUM(quantite) >= 3 AND SUM(quantite * prix_unitaire) >= 50  
ORDER BY ca DESC;  
```

## Combinaison complète - Requêtes avancées

### 🎯 Exemple complet avec toutes les clauses

```sql
-- Analyse complète des performances par auteur
SELECT
    a.nom || ' ' || COALESCE(a.prenom, '') AS auteur,
    a.nationalite,
    COUNT(DISTINCT l.id) AS nb_livres_catalogue,
    COUNT(DISTINCT v.id) AS nb_ventes,
    COALESCE(SUM(v.quantite), 0) AS total_exemplaires_vendus,
    COALESCE(SUM(v.quantite * v.prix_unitaire), 0) AS chiffre_affaires,
    AVG(l.prix) AS prix_moyen_catalogue,
    AVG(l.note_moyenne) AS note_moyenne,
    AVG(l.pages) AS pages_moyennes,
    CASE
        WHEN COUNT(DISTINCT v.id) = 0 THEN 'Aucune vente'
        WHEN SUM(v.quantite * v.prix_unitaire) < 50 THEN 'Faibles ventes'
        WHEN SUM(v.quantite * v.prix_unitaire) < 100 THEN 'Ventes correctes'
        ELSE 'Bonnes ventes'
    END AS performance_commerciale
FROM auteurs a  
JOIN livres l ON a.id = l.auteur_id  
LEFT JOIN ventes v ON l.id = v.livre_id  
WHERE l.prix IS NOT NULL                    -- WHERE: Filtrer avant regroupement  
GROUP BY a.id, a.nom, a.prenom, a.nationalite  
HAVING COUNT(DISTINCT l.id) >= 1            -- HAVING: Filtrer après regroupement  
ORDER BY chiffre_affaires DESC, note_moyenne DESC  -- ORDER BY: Trier le résultat final  
LIMIT 10;                                   -- LIMIT: Limiter aux 10 premiers  

-- Décomposition de cette requête complexe :
-- 1. FROM + JOIN : Tables et relations
-- 2. WHERE : Éliminer les livres sans prix
-- 3. GROUP BY : Regrouper par auteur
-- 4. SELECT : Calculer les agrégations et transformations
-- 5. HAVING : Garder seulement les auteurs avec au moins 1 livre
-- 6. ORDER BY : Trier par CA puis note
-- 7. LIMIT : Top 10
```

### 📊 Tableaux de bord avec requêtes complexes

```sql
-- Dashboard commercial mensuel
-- ⚠️ NOTE — SQLite : group_concat(DISTINCT X, sep) est INTERDIT
--    (« DISTINCT aggregates must have exactly one argument »).
--    Avec DISTINCT, le séparateur est forcément ',' par défaut.
--    Pour conserver un séparateur personnalisé, on pré-calcule le TOP 3 dans une CTE
--    afin d'éliminer les doublons EN AMONT, puis on applique group_concat sans DISTINCT.
WITH top_ventes AS (
    -- Calculer le rang des livres par mois (un livre = une ligne par mois)
    SELECT
        v2.livre_id,
        strftime('%Y-%m', v2.date_vente) AS mois,
        SUM(v2.quantite) AS ventes_livre,
        ROW_NUMBER() OVER (
            PARTITION BY strftime('%Y-%m', v2.date_vente)
            ORDER BY SUM(v2.quantite) DESC
        ) AS rang_mois
    FROM ventes v2
    GROUP BY v2.livre_id, strftime('%Y-%m', v2.date_vente)
),
top_3_par_mois AS (
    -- Ne garder que le TOP 3 par mois → plus de doublons → plus besoin de DISTINCT
    SELECT
        tv.mois,
        GROUP_CONCAT(l.titre || ' (' || tv.ventes_livre || ')', ', ') AS top_3_livres
    FROM top_ventes tv
    JOIN livres l ON l.id = tv.livre_id
    WHERE tv.rang_mois <= 3
    GROUP BY tv.mois
)
SELECT
    strftime('%Y-%m', v.date_vente) AS mois,
    COUNT(DISTINCT v.client_nom) AS nb_clients_uniques,
    COUNT(v.id) AS nb_transactions,
    SUM(v.quantite) AS livres_vendus,
    SUM(v.quantite * v.prix_unitaire) AS chiffre_affaires,
    AVG(v.quantite * v.prix_unitaire) AS panier_moyen,
    COUNT(DISTINCT l.categorie_id) AS nb_categories_vendues,
    t3.top_3_livres
FROM ventes v  
JOIN livres l ON v.livre_id = l.id  
LEFT JOIN top_3_par_mois t3 ON t3.mois = strftime('%Y-%m', v.date_vente)  
GROUP BY strftime('%Y-%m', v.date_vente), t3.top_3_livres  
HAVING COUNT(v.id) > 0  
ORDER BY mois DESC;  

-- Analyse de la concurrence par catégorie
SELECT
    c.nom AS categorie,
    COUNT(l.id) AS nb_livres_disponibles,
    COUNT(DISTINCT l.auteur_id) AS nb_auteurs_distincts,
    COUNT(DISTINCT l.editeur_id) AS nb_editeurs_distincts,
    MIN(l.prix) AS prix_min,
    MAX(l.prix) AS prix_max,
    AVG(l.prix) AS prix_moyen,
    ROUND(AVG(l.note_moyenne), 2) AS note_moyenne_categorie,
    SUM(l.stock) AS stock_total,
    -- Répartition par tranches de prix
    COUNT(CASE WHEN l.prix < 10 THEN 1 END) AS nb_economique,
    COUNT(CASE WHEN l.prix BETWEEN 10 AND 20 THEN 1 END) AS nb_standard,
    COUNT(CASE WHEN l.prix > 20 THEN 1 END) AS nb_premium,
    -- Performance commerciale
    COALESCE(SUM(v.quantite), 0) AS total_vendus,
    COALESCE(SUM(v.quantite * v.prix_unitaire), 0) AS ca_categorie
FROM categories c  
LEFT JOIN livres l ON c.id = l.categorie_id  
LEFT JOIN ventes v ON l.id = v.livre_id  
WHERE l.id IS NOT NULL  -- Seulement les catégories avec des livres  
GROUP BY c.id, c.nom  
HAVING COUNT(l.id) > 0  
ORDER BY ca_categorie DESC, note_moyenne_categorie DESC;  
```

### 🔍 Requêtes d'analyse avec fenêtres (Window Functions)

> ℹ️ Les **window functions** sont un sujet plus avancé, normalement traité au **module 4.3**. On en donne un aperçu ici car elles sont devenues incontournables — depuis SQLite **3.25 (2018)**. Ne vous inquiétez pas si tout n'est pas clair immédiatement : le module 4 reprendra chaque fonction en détail.

```sql
-- Classement des livres avec fonctions de fenêtre
SELECT
    l.titre,
    a.nom AS auteur,
    c.nom AS categorie,
    l.prix,
    l.note_moyenne,
    l.nombre_avis,
    -- Rang par prix dans la catégorie
    RANK() OVER (PARTITION BY l.categorie_id ORDER BY l.prix DESC) AS rang_prix_categorie,
    -- Rang par note globalement
    DENSE_RANK() OVER (ORDER BY l.note_moyenne DESC) AS rang_note_global,
    -- Prix moyen de la catégorie
    AVG(l.prix) OVER (PARTITION BY l.categorie_id) AS prix_moyen_categorie,
    -- Écart par rapport à la moyenne de la catégorie
    ROUND(l.prix - AVG(l.prix) OVER (PARTITION BY l.categorie_id), 2) AS ecart_prix_moyen,
    -- Pourcentage du prix dans la catégorie
    ROUND(
        l.prix * 100.0 / SUM(l.prix) OVER (PARTITION BY l.categorie_id),
        2
    ) AS pct_prix_categorie,
    -- Numéro de ligne pour pagination
    ROW_NUMBER() OVER (ORDER BY l.note_moyenne DESC, l.nombre_avis DESC) AS numero_ligne
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
JOIN categories c ON l.categorie_id = c.id  
WHERE l.prix IS NOT NULL AND l.note_moyenne IS NOT NULL  
ORDER BY rang_note_global, rang_prix_categorie;  

-- Évolution temporelle des ventes
SELECT
    v.date_vente,
    l.titre,
    v.quantite,
    v.prix_unitaire,
    v.quantite * v.prix_unitaire AS ca_transaction,
    -- Cumul des ventes par livre
    SUM(v.quantite) OVER (
        PARTITION BY v.livre_id
        ORDER BY v.date_vente
        ROWS UNBOUNDED PRECEDING
    ) AS ventes_cumulees_livre,
    -- Cumul du CA global
    SUM(v.quantite * v.prix_unitaire) OVER (
        ORDER BY v.date_vente
        ROWS UNBOUNDED PRECEDING
    ) AS ca_cumule_global,
    -- Moyenne mobile sur 3 transactions
    AVG(v.quantite * v.prix_unitaire) OVER (
        ORDER BY v.date_vente
        ROWS 2 PRECEDING
    ) AS moyenne_mobile_3_transactions,
    -- Jour précédent et suivant
    LAG(v.quantite * v.prix_unitaire, 1) OVER (ORDER BY v.date_vente) AS ca_precedent,
    LEAD(v.quantite * v.prix_unitaire, 1) OVER (ORDER BY v.date_vente) AS ca_suivant
FROM ventes v  
JOIN livres l ON v.livre_id = l.id  
ORDER BY v.date_vente, v.id;  
```

### 🎯 Sous-requêtes et requêtes corrélées

```sql
-- Livres au-dessus de la moyenne de leur catégorie
SELECT
    l.titre,
    a.nom AS auteur,
    c.nom AS categorie,
    l.prix,
    -- Sous-requête pour la moyenne de la catégorie
    (SELECT AVG(l2.prix)
     FROM livres l2
     WHERE l2.categorie_id = l.categorie_id) AS prix_moyen_categorie,
    -- Écart par rapport à la moyenne
    ROUND(l.prix - (SELECT AVG(l2.prix)
                    FROM livres l2
                    WHERE l2.categorie_id = l.categorie_id), 2) AS ecart_moyenne
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
JOIN categories c ON l.categorie_id = c.id  
WHERE l.prix > (  
    SELECT AVG(l3.prix)
    FROM livres l3
    WHERE l3.categorie_id = l.categorie_id
)
ORDER BY ecart_moyenne DESC;

-- Auteurs avec tous leurs livres bien notés (> 4.0)
SELECT
    a.nom || ' ' || COALESCE(a.prenom, '') AS auteur,
    COUNT(l.id) AS nb_livres,
    MIN(l.note_moyenne) AS note_minimum,
    AVG(l.note_moyenne) AS note_moyenne_auteur
FROM auteurs a  
JOIN livres l ON a.id = l.auteur_id  
WHERE NOT EXISTS (  
    -- Sous-requête corrélée : aucun livre mal noté
    SELECT 1
    FROM livres l2
    WHERE l2.auteur_id = a.id
    AND (l2.note_moyenne < 4.0 OR l2.note_moyenne IS NULL)
)
GROUP BY a.id, a.nom, a.prenom  
ORDER BY note_moyenne_auteur DESC;  

-- Livres bestsellers de leur mois de vente
SELECT DISTINCT
    l.titre,
    a.nom AS auteur,
    v.date_vente,
    mois_ventes.total_vendus_mois,
    mois_ventes.max_vendus_livre_mois
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
JOIN ventes v ON l.id = v.livre_id  
JOIN (  
    -- Sous-requête : statistiques par mois
    SELECT
        strftime('%Y-%m', v2.date_vente) AS mois,
        SUM(v2.quantite) AS total_vendus_mois,
        MAX(livre_ventes.total_livre) AS max_vendus_livre_mois
    FROM ventes v2
    JOIN (
        -- Sous-sous-requête : total par livre par mois
        SELECT
            livre_id,
            strftime('%Y-%m', date_vente) AS mois,
            SUM(quantite) AS total_livre
        FROM ventes
        GROUP BY livre_id, strftime('%Y-%m', date_vente)
    ) livre_ventes ON v2.livre_id = livre_ventes.livre_id
        AND strftime('%Y-%m', v2.date_vente) = livre_ventes.mois
    GROUP BY strftime('%Y-%m', v2.date_vente)
) mois_ventes ON strftime('%Y-%m', v.date_vente) = mois_ventes.mois
WHERE v.livre_id IN (
    -- Le livre doit être le plus vendu de son mois
    SELECT v3.livre_id
    FROM ventes v3
    WHERE strftime('%Y-%m', v3.date_vente) = strftime('%Y-%m', v.date_vente)
    GROUP BY v3.livre_id
    HAVING SUM(v3.quantite) = mois_ventes.max_vendus_livre_mois
)
ORDER BY v.date_vente DESC;
```

## Optimisation et bonnes pratiques

### ⚡ Requêtes performantes

```sql
-- ❌ Requête non optimisée
SELECT
    l.titre,
    a.nom,
    (SELECT COUNT(*) FROM ventes v WHERE v.livre_id = l.id) AS nb_ventes
FROM livres l, auteurs a  
WHERE l.auteur_id = a.id  
AND LENGTH(l.titre) > 10  
ORDER BY (SELECT SUM(v.quantite) FROM ventes v WHERE v.livre_id = l.id) DESC;  

-- ✅ Requête optimisée
SELECT
    l.titre,
    a.nom,
    COALESCE(v_stats.nb_ventes, 0) AS nb_ventes,
    COALESCE(v_stats.total_quantite, 0) AS total_vendus
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  -- JOIN explicite  
LEFT JOIN (  
    -- Pré-calculer les statistiques de vente
    SELECT
        livre_id,
        COUNT(*) AS nb_ventes,
        SUM(quantite) AS total_quantite
    FROM ventes
    GROUP BY livre_id
) v_stats ON l.id = v_stats.livre_id
WHERE LENGTH(l.titre) > 10  -- Filtre précoce  
ORDER BY total_vendus DESC;  

-- Index recommandés pour cette requête :
-- CREATE INDEX idx_livres_auteur ON livres(auteur_id);
-- CREATE INDEX idx_ventes_livre ON ventes(livre_id);
-- CREATE INDEX idx_livres_titre_length ON livres(LENGTH(titre));
```

### 📊 Pagination efficace

```sql
-- Pagination avec LIMIT et OFFSET
SELECT
    l.titre,
    a.nom AS auteur,
    l.prix,
    l.note_moyenne
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
ORDER BY l.note_moyenne DESC, l.id  
LIMIT 10 OFFSET 20;  -- Page 3 (20 = 2 * 10)  

-- Pagination par curseur (plus efficace pour grandes tables)
SELECT
    l.titre,
    a.nom AS auteur,
    l.prix,
    l.note_moyenne,
    l.id
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
WHERE (l.note_moyenne, l.id) < (4.3, 145)  -- Derniers valeurs de la page précédente  
ORDER BY l.note_moyenne DESC, l.id DESC  
LIMIT 10;  

-- Comptage total pour la pagination (optimisé)
SELECT
    COUNT(*) OVER() AS total_resultats,
    l.titre,
    a.nom AS auteur,
    l.prix
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
WHERE l.prix BETWEEN 10 AND 20  
ORDER BY l.prix  
LIMIT 10 OFFSET 0;  
```

### 🔍 Requêtes de diagnostic et maintenance

```sql
-- Analyse de la qualité des données
SELECT
    'Livres sans prix' AS probleme,
    COUNT(*) AS nombre
FROM livres  
WHERE prix IS NULL  
UNION ALL  
SELECT  
    'Livres sans note',
    COUNT(*)
FROM livres  
WHERE note_moyenne IS NULL  
UNION ALL  
SELECT  
    'Livres sans stock',
    COUNT(*)
FROM livres  
WHERE stock = 0  
UNION ALL  
SELECT  
    'Auteurs sans livres',
    COUNT(*)
FROM auteurs a  
WHERE NOT EXISTS (SELECT 1 FROM livres l WHERE l.auteur_id = a.id);  

-- Détection des outliers (prix anormaux)
SELECT
    titre,
    prix,
    (SELECT AVG(prix) FROM livres WHERE prix IS NOT NULL) AS prix_moyen_global,
    (SELECT AVG(prix) FROM livres l2 WHERE l2.categorie_id = l.categorie_id) AS prix_moyen_categorie,
    ABS(prix - (SELECT AVG(prix) FROM livres l3 WHERE l3.categorie_id = l.categorie_id)) AS ecart_absolu
FROM livres l  
WHERE prix IS NOT NULL  
    AND ABS(prix - (SELECT AVG(prix) FROM livres l4 WHERE l4.categorie_id = l.categorie_id)) >
        (SELECT 2 * AVG(ABS(prix - prix_moy))
         FROM (SELECT prix, AVG(prix) OVER() AS prix_moy FROM livres WHERE prix IS NOT NULL))
ORDER BY ecart_absolu DESC;

-- Rapport de cohérence référentielle
SELECT
    'Livres avec auteur inexistant' AS probleme,
    COUNT(*) AS nombre,
    GROUP_CONCAT(DISTINCT l.auteur_id) AS ids_problematiques
FROM livres l  
LEFT JOIN auteurs a ON l.auteur_id = a.id  
WHERE a.id IS NULL  
UNION ALL  
SELECT  
    'Ventes avec livre inexistant',
    COUNT(*),
    GROUP_CONCAT(DISTINCT v.livre_id)
FROM ventes v  
LEFT JOIN livres l ON v.livre_id = l.id  
WHERE l.id IS NULL;  
```

## 🎯 Exercice pratique final - Tableau de bord complet

### Objectif : Créer un dashboard d'analyse commerciale complet

```sql
-- === TABLEAU DE BORD COMMERCIAL ===

-- 1. Vue d'ensemble générale
-- ⚠️ La syntaxe `valeur::TEXT` est propre à PostgreSQL ; en SQLite, utilisez CAST(valeur AS TEXT)
-- (ou laissez la concaténation `||` faire la conversion implicite vers TEXT)
SELECT 'RÉSUMÉ GÉNÉRAL' AS section, NULL AS detail, NULL AS valeur  
UNION ALL  
SELECT '', 'Nombre total de livres au catalogue', CAST(COUNT(*) AS TEXT) FROM livres  
UNION ALL  
SELECT '', 'Nombre d''auteurs', CAST(COUNT(*) AS TEXT) FROM auteurs  
UNION ALL  
SELECT '', 'Nombre de catégories', CAST(COUNT(*) AS TEXT) FROM categories  
UNION ALL  
SELECT '', 'Valeur totale du stock', CAST(ROUND(SUM(prix * stock), 2) AS TEXT) || '€' FROM livres WHERE prix IS NOT NULL  
UNION ALL  
SELECT '', 'Chiffre d''affaires total', CAST(ROUND(SUM(quantite * prix_unitaire), 2) AS TEXT) || '€' FROM ventes;  

-- 2. Top 5 des bestsellers
SELECT 'TOP BESTSELLERS' AS section,
       CAST(ROW_NUMBER() OVER (ORDER BY SUM(v.quantite) DESC) AS TEXT) || '. ' || l.titre AS detail,
       CAST(SUM(v.quantite) AS TEXT) || ' exemplaires vendus' AS valeur
FROM livres l  
JOIN ventes v ON l.id = v.livre_id  
GROUP BY l.id, l.titre  
ORDER BY SUM(v.quantite) DESC  
LIMIT 5;  

-- 3. Analyse par catégorie
WITH stats_categories AS (
    SELECT
        c.nom AS categorie,
        COUNT(l.id) AS nb_livres,
        COALESCE(SUM(v.quantite), 0) AS total_vendus,
        COALESCE(SUM(v.quantite * v.prix_unitaire), 0) AS ca,
        AVG(l.prix) AS prix_moyen,
        AVG(l.note_moyenne) AS note_moyenne
    FROM categories c
    LEFT JOIN livres l ON c.id = l.categorie_id
    LEFT JOIN ventes v ON l.id = v.livre_id
    GROUP BY c.id, c.nom
    HAVING COUNT(l.id) > 0
)
SELECT
    'PERFORMANCE PAR CATÉGORIE' AS section,
    categorie AS detail,
    'CA: ' || ROUND(ca, 2) || '€ | ' ||
    'Vendus: ' || total_vendus || ' | ' ||
    'Note: ' || ROUND(note_moyenne, 1) || '/5' AS valeur
FROM stats_categories  
ORDER BY ca DESC;  

-- 4. Évolution temporelle
WITH ventes_mensuelles AS (
    SELECT
        strftime('%Y-%m', date_vente) AS mois,
        SUM(quantite) AS livres_vendus,
        SUM(quantite * prix_unitaire) AS ca_mois,
        COUNT(DISTINCT client_nom) AS clients_uniques
    FROM ventes
    GROUP BY strftime('%Y-%m', date_vente)
)
SELECT
    'ÉVOLUTION MENSUELLE' AS section,
    mois AS detail,
    CAST(ca_mois AS TEXT) || '€ (' || livres_vendus || ' livres, ' || clients_uniques || ' clients)' AS valeur
FROM ventes_mensuelles  
ORDER BY mois DESC;  

-- 5. Alertes stock
SELECT 'ALERTES STOCK' AS section,
       CASE
           WHEN stock = 0 THEN '🔴 RUPTURE: ' || titre
           WHEN stock < 5 THEN '🟡 FAIBLE: ' || titre || ' (' || stock || ' restants)'
           ELSE '🟢 OK: ' || titre
       END AS detail,
       CAST(stock AS TEXT) AS valeur
FROM livres  
WHERE stock < 10  
ORDER BY stock, titre;  

-- 6. Analyse de rentabilité par auteur
WITH rentabilite_auteurs AS (
    SELECT
        a.nom || ' ' || COALESCE(a.prenom, '') AS auteur,
        COUNT(DISTINCT l.id) AS nb_livres,
        COALESCE(SUM(v.quantite * v.prix_unitaire), 0) AS ca_auteur,
        AVG(l.note_moyenne) AS note_moyenne
    FROM auteurs a
    JOIN livres l ON a.id = l.auteur_id
    LEFT JOIN ventes v ON l.id = v.livre_id
    GROUP BY a.id, a.nom, a.prenom
    HAVING COUNT(DISTINCT l.id) > 0
)
SELECT
    'RENTABILITÉ AUTEURS' AS section,
    auteur AS detail,
    CAST(ROUND(ca_auteur, 2) AS TEXT) || '€ (' || nb_livres || ' livres, note: ' || ROUND(note_moyenne, 1) || ')' AS valeur
FROM rentabilite_auteurs  
WHERE ca_auteur > 0  
ORDER BY ca_auteur DESC  
LIMIT 5;  

-- 7. Recommandations automatiques
SELECT 'RECOMMANDATIONS' AS section,
       CASE
           WHEN action_type = 'restock' THEN '📦 RÉAPPROVISIONNER: ' || titre
           WHEN action_type = 'promotion' THEN '💰 PROMOTION: ' || titre
           WHEN action_type = 'mise_avant' THEN '⭐ METTRE EN AVANT: ' || titre
       END AS detail,
       justification AS valeur
FROM (
    -- Livres à réapprovisionner (stock faible + bonnes ventes)
    SELECT
        l.titre,
        'restock' AS action_type,
        'Stock: ' || l.stock || ', Vendus: ' || COALESCE(SUM(v.quantite), 0) AS justification
    FROM livres l
    LEFT JOIN ventes v ON l.id = v.livre_id
    WHERE l.stock < 5
    GROUP BY l.id, l.titre, l.stock
    HAVING COALESCE(SUM(v.quantite), 0) > 0

    UNION ALL

    -- Livres à promouvoir (prix élevé + stock important)
    SELECT
        l.titre,
        'promotion',
        'Prix: ' || l.prix || '€, Stock: ' || l.stock
    FROM livres l
    WHERE l.prix > 15 AND l.stock > 15

    UNION ALL

    -- Livres à mettre en avant (bien notés + peu vendus)
    SELECT
        l.titre,
        'mise_avant',
        'Note: ' || l.note_moyenne || '/5, Vendus: ' || COALESCE(SUM(v.quantite), 0)
    FROM livres l
    LEFT JOIN ventes v ON l.id = v.livre_id
    WHERE l.note_moyenne >= 4.5 AND l.nombre_avis >= 10
    GROUP BY l.id, l.titre, l.note_moyenne
    HAVING COALESCE(SUM(v.quantite), 0) < 3
) recommandations
ORDER BY action_type, titre;
```

## Récapitulatif et bonnes pratiques

### ✅ Maîtrise des clauses SELECT

**Ordre syntaxique** (l'ordre dans lequel vous écrivez la requête) :
```sql
SELECT … FROM … [JOIN …] [WHERE …] [GROUP BY …] [HAVING …] [ORDER BY …] [LIMIT …]
```

**Ordre logique d'évaluation** (l'ordre dans lequel le moteur exécute) :
1. **FROM** + **JOIN** — déterminer les tables et les jointures
2. **WHERE** — filtrer les lignes individuelles
3. **GROUP BY** — regrouper
4. **HAVING** — filtrer les groupes (après agrégation)
5. **SELECT** — calculer les colonnes finales (y compris les agrégats)
6. **DISTINCT** — éliminer les doublons
7. **ORDER BY** — trier
8. **LIMIT** / **OFFSET** — découper

> 💡 Cette différence explique pourquoi un alias défini dans `SELECT` n'est pas utilisable dans `WHERE` (qui s'exécute avant), mais l'est dans `ORDER BY` (qui s'exécute après).

### 🎯 Checklist des bonnes pratiques

#### ✅ Performance

- [ ] **Utiliser des JOIN explicites** plutôt que des virgules
- [ ] **Filtrer tôt** avec WHERE avant GROUP BY
- [ ] **Éviter SELECT *** sauf pour l'exploration
- [ ] **Limiter les résultats** avec LIMIT quand approprié
- [ ] **Utiliser des index** sur les colonnes de filtrage et tri

#### ✅ Lisibilité

- [ ] **Indenter** et **structurer** vos requêtes
- [ ] **Utiliser des alias** explicites pour les colonnes calculées
- [ ] **Commenter** les requêtes complexes
- [ ] **Découper** les requêtes très complexes en vues ou sous-requêtes

#### ✅ Robustesse

- [ ] **Gérer les NULL** avec COALESCE ou IS NULL/IS NOT NULL
- [ ] **Valider les données** avec WHERE appropriés
- [ ] **Tester** avec différents jeux de données
- [ ] **Prévoir les cas limites** (division par zéro, etc.)

### 💡 Points clés à retenir

1. **WHERE vs HAVING** : WHERE filtre avant regroupement, HAVING après
2. **NULL handling** : Toujours prévoir la gestion des valeurs NULL
3. **Fonctions d'agrégation** : COUNT, SUM, AVG, MIN, MAX ignorent les NULL
4. **CASE expressions** : Puissantes pour la logique conditionnelle
5. **Sous-requêtes** : Utilisées avec modération pour la lisibilité
6. **Window functions** : Excellentes pour les classements et analyses

## Conclusion du Module 2.5

### 🎉 Ce que vous avez appris

**Techniques de requêtage maîtrisées :**
- ✅ **SELECT avancé** : Expressions, fonctions, CASE
- ✅ **WHERE sophistiqué** : Filtres complexes, sous-requêtes
- ✅ **ORDER BY** : Tri multi-critères et conditionnel
- ✅ **GROUP BY + agrégations** : Analyses statistiques
- ✅ **HAVING** : Filtrage de groupes
- ✅ **Combinaisons complexes** : Requêtes multi-clauses

**Compétences analytiques développées :**
- ✅ Création de tableaux de bord
- ✅ Analyses temporelles et tendances
- ✅ Détection d'anomalies et outliers
- ✅ Recommandations automatisées
- ✅ Reporting commercial avancé

### 🚀 Vous êtes maintenant capable de

- Extraire n'importe quelle information de vos bases SQLite
- Créer des rapports d'analyse sophistiqués
- Optimiser vos requêtes pour de bonnes performances
- Construire des tableaux de bord interactifs
- Détecter et corriger les problèmes de données

---

**💡 Dans le prochain module**, nous passerons à la **conception et modélisation avancée** où vous apprendrez à structurer des bases de données complexes avec la normalisation, les relations avancées et les patterns de conception.

**🎯 Vous maîtrisez maintenant** les requêtes SQLite ! Vous pouvez interroger efficacement vos données et en extraire des insights précieux pour vos applications et analyses.

⏭️ [Module 3 : Conception et modélisation avancée](/03-conception-modelisation-avancee/README.md)
