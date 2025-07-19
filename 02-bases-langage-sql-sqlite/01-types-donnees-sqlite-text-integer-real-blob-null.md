🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.1 Types de données SQLite (TEXT, INTEGER, REAL, BLOB, NULL)

## Introduction - La simplicité révolutionnaire de SQLite

SQLite adopte une approche unique et révolutionnaire des types de données qui le distingue de tous les autres systèmes de bases de données. Là où MySQL ou PostgreSQL ont des dizaines de types stricts, SQLite n'en a que **5 types principaux** avec une **flexibilité extraordinaire**.

> **Analogie simple** : Si les autres SGBD sont comme des tiroirs étiquetés où chaque objet doit aller dans le bon compartiment, SQLite est comme une boîte magique qui s'adapte intelligemment à ce que vous y mettez !

**Les 5 types de SQLite :**
- **TEXT** → Chaînes de caractères
- **INTEGER** → Nombres entiers
- **REAL** → Nombres décimaux
- **BLOB** → Données binaires
- **NULL** → Valeur absente

## Le système de types dynamique - Révolution conceptuelle

### 🔄 Types dynamiques vs types statiques

**Autres SGBD (types statiques) :**
```sql
-- MySQL/PostgreSQL : Types fixes et stricts
CREATE TABLE users (
    id INT PRIMARY KEY,              -- EXACTEMENT un entier
    name VARCHAR(50) NOT NULL,       -- EXACTEMENT 50 caractères max
    age TINYINT UNSIGNED,           -- EXACTEMENT 0-255
    salary DECIMAL(10,2)            -- EXACTEMENT 10 digits, 2 décimales
);

-- ❌ Cette insertion échouerait
INSERT INTO users VALUES (1, 'Un nom beaucoup trop long pour 50 caractères', 25, 1500.123);
```

**SQLite (types dynamiques) :**
```sql
-- SQLite : Types flexibles et adaptatifs
CREATE TABLE users (
    id INTEGER PRIMARY KEY,          -- Suggère entier, mais accepte autre chose
    name TEXT,                      -- Texte de n'importe quelle longueur
    age INTEGER,                    -- Suggère entier
    salary REAL                     -- Suggère décimal
);

-- ✅ Toutes ces insertions fonctionnent !
INSERT INTO users VALUES (1, 'Alice', 25, 1500.50);
INSERT INTO users VALUES ('2', 'Bob avec un nom très très long', '30', '2000');
INSERT INTO users VALUES (3.5, 'Charlie', 25.8, 1500);
```

### 🎯 Affinité de type - Le secret de SQLite

SQLite utilise le concept d'**affinité de type** : chaque colonne a une **préférence** pour un type, mais peut stocker autre chose si nécessaire.

```sql
-- Démonstration de l'affinité
CREATE TABLE demo_affinite (
    id INTEGER,
    nom TEXT,
    prix REAL,
    data BLOB
);

-- Insertions variées
INSERT INTO demo_affinite VALUES
    (1, 'Produit A', 15.50, 'données'),
    ('2', 123, '20', X'48656C6C6F'),      -- Valeurs "mixtes"
    (3.14, NULL, 25, 'texte normal');

-- Voir ce qui est réellement stocké
SELECT
    id,
    typeof(id) as type_id,
    nom,
    typeof(nom) as type_nom,
    prix,
    typeof(prix) as type_prix,
    data,
    typeof(data) as type_data
FROM demo_affinite;
```

## TEXT - Le type universel

### 📝 Caractéristiques du type TEXT

```sql
-- TEXT peut contenir n'importe quel texte
CREATE TABLE exemples_text (
    id INTEGER PRIMARY KEY,
    court TEXT,
    long TEXT,
    special TEXT,
    unicode TEXT
);

INSERT INTO exemples_text (court, long, special, unicode) VALUES
    (
        'Court',
        'Un texte très long qui peut contenir plusieurs paragraphes et même des retours à la ligne
        comme celui-ci, sans aucune limitation de longueur contrairement aux VARCHAR d''autres SGBD',
        'Texte avec "guillemets", apostrophes'', caractères spéciaux: @#$%^&*()',
        'Unicode: 🎉 Émojis, ñoël, 中文, العربية, русский'
    );

-- Vérifier les longueurs
SELECT
    court,
    LENGTH(court) as longueur_court,
    long,
    LENGTH(long) as longueur_long,
    unicode,
    LENGTH(unicode) as longueur_unicode
FROM exemples_text;
```

### 🛠️ Fonctions utiles avec TEXT

```sql
-- Table d'exemple pour les fonctions
CREATE TABLE clients (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    prenom TEXT,
    email TEXT,
    telephone TEXT
);

INSERT INTO clients (nom, prenom, email, telephone) VALUES
    ('Dupont', 'Jean', 'jean.dupont@email.com', '0123456789'),
    ('MARTIN', 'sophie', 'Sophie.Martin@GMAIL.COM', '06.12.34.56.78'),
    ('Le Roy', 'Pierre-Antoine', 'pierre.leroy@entreprise.fr', '+33 6 12 34 56 78');

-- Manipulation de chaînes
SELECT
    nom,
    prenom,
    -- Concaténation
    nom || ' ' || prenom as nom_complet,
    -- Changement de casse
    UPPER(nom) as nom_majuscule,
    LOWER(email) as email_minuscule,
    -- Nettoyage
    TRIM(telephone) as tel_nettoye,
    -- Extraction de parties
    SUBSTR(email, 1, INSTR(email, '@') - 1) as username,
    SUBSTR(email, INSTR(email, '@') + 1) as domaine,
    -- Longueur
    LENGTH(nom) as longueur_nom,
    -- Remplacement
    REPLACE(telephone, '.', '-') as tel_tirets,
    REPLACE(telephone, ' ', '') as tel_sans_espaces
FROM clients;
```

### 🔍 Recherche et filtrage avec TEXT

```sql
-- Recherches textuelles
SELECT * FROM clients WHERE nom LIKE 'Du%';                    -- Commence par "Du"
SELECT * FROM clients WHERE email LIKE '%@gmail.com';          -- Se termine par "@gmail.com"
SELECT * FROM clients WHERE nom LIKE '%ar%';                   -- Contient "ar"
SELECT * FROM clients WHERE LENGTH(prenom) > 6;                -- Prénom long
SELECT * FROM clients WHERE email NOT LIKE '%@gmail.%';        -- Pas Gmail

-- Recherche insensible à la casse
SELECT * FROM clients WHERE LOWER(nom) = 'martin';
SELECT * FROM clients WHERE UPPER(email) LIKE '%GMAIL%';

-- Recherche avec expressions régulières (si activées)
-- SELECT * FROM clients WHERE email REGEXP '^[a-z]+\.[a-z]+@[a-z]+\.(com|fr)$';
```

### ⚠️ Pièges courants avec TEXT

```sql
-- Piège 1 : Comparaisons numériques sur du texte
CREATE TABLE test_pieges (valeur TEXT);
INSERT INTO test_pieges VALUES ('1'), ('2'), ('10'), ('20');

-- ❌ Tri alphabétique (pas numérique !)
SELECT * FROM test_pieges ORDER BY valeur;
-- Résultat : 1, 10, 2, 20

-- ✅ Tri numérique correct
SELECT * FROM test_pieges ORDER BY CAST(valeur AS INTEGER);
-- Résultat : 1, 2, 10, 20

-- Piège 2 : Espaces invisibles
INSERT INTO clients (nom, prenom) VALUES ('Durand ', ' Marie');  -- Espaces !

-- ❌ Cette recherche peut échouer
SELECT * FROM clients WHERE nom = 'Durand';

-- ✅ Recherche avec nettoyage
SELECT * FROM clients WHERE TRIM(nom) = 'Durand';
```

## INTEGER - Les nombres entiers

### 🔢 Caractéristiques du type INTEGER

```sql
-- INTEGER peut stocker des entiers de différentes tailles
CREATE TABLE exemples_integer (
    id INTEGER PRIMARY KEY,
    petit INTEGER,
    moyen INTEGER,
    grand INTEGER,
    tres_grand INTEGER
);

INSERT INTO exemples_integer (petit, moyen, grand, tres_grand) VALUES
    (1, 1000, 1000000, 9223372036854775807),      -- Valeurs normales
    (-1, -1000, -1000000, -9223372036854775808),  -- Valeurs négatives
    (0, NULL, 42, 123456789012345);               -- Zéro et NULL

-- Vérifier les types et valeurs
SELECT
    petit, typeof(petit),
    moyen, typeof(moyen),
    grand, typeof(grand),
    tres_grand, typeof(tres_grand)
FROM exemples_integer;
```

### 🔄 Conversions automatiques avec INTEGER

```sql
-- Table de démonstration des conversions
CREATE TABLE demo_conversions (
    id INTEGER PRIMARY KEY,
    valeur_stockee,  -- Pas de type spécifié
    type_reel TEXT
);

-- Insertions avec différents types
INSERT INTO demo_conversions (valeur_stockee, type_reel) VALUES
    (123, 'entier direct'),
    ('456', 'texte numérique'),
    (78.9, 'décimal qui sera arrondi'),
    ('12.34', 'texte décimal'),
    ('abc123', 'texte non numérique'),
    (NULL, 'valeur nulle');

-- Voir les conversions
SELECT
    valeur_stockee,
    typeof(valeur_stockee) as type_sqlite,
    type_reel,
    CAST(valeur_stockee AS INTEGER) as force_integer,
    CAST(valeur_stockee AS REAL) as force_real
FROM demo_conversions;
```

### 📊 Opérations mathématiques avec INTEGER

```sql
-- Table pour les calculs
CREATE TABLE calculs (
    a INTEGER,
    b INTEGER
);

INSERT INTO calculs VALUES
    (10, 3),
    (15, 4),
    (7, 2),
    (-5, 3);

SELECT
    a, b,
    a + b as addition,
    a - b as soustraction,
    a * b as multiplication,
    a / b as division,              -- Attention : division réelle !
    a / b as division_reelle,
    a % b as modulo,
    ABS(a) as valeur_absolue,
    MIN(a, b) as minimum,
    MAX(a, b) as maximum,
    ROUND(a / CAST(b AS REAL), 2) as division_precise
FROM calculs;
```

### 🎯 INTEGER comme booléens

```sql
-- SQLite n'a pas de type BOOLEAN, on utilise INTEGER
CREATE TABLE produits (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    prix REAL,
    actif INTEGER,      -- 0 = faux, 1 = vrai
    en_promotion INTEGER DEFAULT 0,
    nouveau INTEGER DEFAULT 1
);

INSERT INTO produits (nom, prix, actif, en_promotion, nouveau) VALUES
    ('Laptop', 999.99, 1, 0, 1),
    ('Souris', 25.50, 1, 1, 0),
    ('Clavier ancien', 45.00, 0, 0, 0);

-- Requêtes avec booléens
SELECT nom, prix FROM produits WHERE actif = 1;                    -- Produits actifs
SELECT nom, prix FROM produits WHERE actif AND en_promotion;       -- Actifs ET en promo
SELECT nom, prix FROM produits WHERE NOT actif;                    -- Produits inactifs

-- Conversion en texte lisible
SELECT
    nom,
    CASE actif WHEN 1 THEN 'Actif' ELSE 'Inactif' END as statut,
    CASE en_promotion WHEN 1 THEN 'En promotion' ELSE 'Prix normal' END as promo
FROM produits;
```

## REAL - Les nombres décimaux

### 🔢 Caractéristiques du type REAL

```sql
-- REAL stocke des nombres à virgule flottante
CREATE TABLE exemples_real (
    id INTEGER PRIMARY KEY,
    prix REAL,
    pourcentage REAL,
    scientifique REAL,
    precision_test REAL
);

INSERT INTO exemples_real (prix, pourcentage, scientifique, precision_test) VALUES
    (19.99, 15.5, 1.23e-4, 1.0/3.0),
    (1299.50, 0.05, 6.022e23, 0.1 + 0.2),
    (-45.67, 100.0, -1.5e10, 2.0/3.0);

-- Voir les valeurs stockées
SELECT
    prix,
    pourcentage || '%' as pct_formate,
    scientifique,
    precision_test,
    ROUND(precision_test, 10) as precision_arrondie,
    typeof(prix) as type_prix
FROM exemples_real;
```

### ⚠️ Problèmes de précision avec REAL

```sql
-- Démonstration des problèmes de virgule flottante
CREATE TABLE demo_precision (
    calcul TEXT,
    resultat REAL,
    attendu REAL
);

INSERT INTO demo_precision VALUES
    ('0.1 + 0.2', 0.1 + 0.2, 0.3),
    ('1.0 / 3.0 * 3.0', 1.0/3.0*3.0, 1.0),
    ('0.1 * 10', 0.1 * 10, 1.0);

SELECT
    calcul,
    resultat,
    attendu,
    resultat = attendu as est_egal,
    ABS(resultat - attendu) < 0.000001 as quasi_egal,
    ROUND(resultat, 2) as resultat_arrondi
FROM demo_precision;
```

### 💰 Gestion de l'argent - Techniques alternatives

```sql
-- ❌ Mauvaise pratique : Stocker l'argent en REAL
CREATE TABLE produits_mauvais (
    nom TEXT,
    prix REAL  -- Problème de précision !
);

-- ✅ Bonne pratique : Stocker en centimes (INTEGER)
CREATE TABLE produits_bons (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    prix_centimes INTEGER,  -- Prix en centimes
    devise TEXT DEFAULT 'EUR'
);

INSERT INTO produits_bons (nom, prix_centimes) VALUES
    ('Livre', 1250),       -- 12.50€
    ('DVD', 999),          -- 9.99€
    ('Console', 29999);    -- 299.99€

-- Affichage correct des prix
SELECT
    nom,
    prix_centimes / 100.0 as prix_euros,
    (prix_centimes / 100) || '.' ||
    CASE
        WHEN (prix_centimes % 100) < 10 THEN '0' || (prix_centimes % 100)
        ELSE (prix_centimes % 100)
    END || ' €' as prix_formate,
    devise
FROM produits_bons;

-- Calculs exacts avec les centimes
SELECT
    nom,
    prix_centimes,
    -- TVA 20% exacte
    prix_centimes * 120 / 100 as prix_ttc_centimes,
    (prix_centimes * 120 / 100) / 100.0 as prix_ttc_euros
FROM produits_bons;
```

### 📐 Fonctions mathématiques avec REAL

```sql
-- Table pour les fonctions mathématiques
CREATE TABLE mesures (
    longueur REAL,
    largeur REAL,
    angle_degres REAL
);

INSERT INTO mesures VALUES
    (3.5, 4.2, 45),
    (10.0, 5.0, 90),
    (7.8, 7.8, 30);

SELECT
    longueur,
    largeur,
    -- Géométrie
    longueur * largeur as surface,
    2 * (longueur + largeur) as perimetre,
    SQRT(longueur*longueur + largeur*largeur) as diagonale,
    -- Fonctions mathématiques
    ABS(longueur - largeur) as difference_absolue,
    MIN(longueur, largeur) as plus_petit_cote,
    MAX(longueur, largeur) as plus_grand_cote,
    -- Arrondis
    ROUND(longueur * largeur, 2) as surface_arrondie,
    CAST(longueur * largeur AS INTEGER) as surface_entiere
FROM mesures;
```

## BLOB - Les données binaires

### 📁 Caractéristiques du type BLOB

BLOB (Binary Large Object) stocke des données binaires brutes : images, fichiers, documents...

```sql
-- Table pour stocker des fichiers
CREATE TABLE fichiers (
    id INTEGER PRIMARY KEY,
    nom_fichier TEXT,
    type_mime TEXT,
    taille INTEGER,
    contenu BLOB,
    date_upload TEXT DEFAULT (datetime('now'))
);

-- Insertion de données binaires simples
INSERT INTO fichiers (nom_fichier, type_mime, taille, contenu) VALUES
    ('hello.txt', 'text/plain', 13, 'Hello, World!'),
    ('data.bin', 'application/octet-stream', 4, X'DEADBEEF'),
    ('config.json', 'application/json', 25, '{"active": true, "port": 8080}');

-- Voir les données
SELECT
    nom_fichier,
    type_mime,
    taille,
    LENGTH(contenu) as taille_reelle,
    SUBSTR(contenu, 1, 20) as debut_contenu,
    typeof(contenu) as type_sqlite
FROM fichiers;
```

### 🔧 Manipulation des BLOB

```sql
-- Fonctions utiles avec BLOB
SELECT
    nom_fichier,
    -- Taille en bytes
    LENGTH(contenu) as taille_bytes,
    -- Conversion en hexadécimal
    HEX(SUBSTR(contenu, 1, 10)) as hex_debut,
    -- Recherche dans le contenu
    CASE
        WHEN contenu LIKE '%json%' THEN 'Contient "json"'
        WHEN contenu LIKE '%Hello%' THEN 'Contient "Hello"'
        ELSE 'Autre contenu'
    END as analyse_contenu,
    -- Hash simple (non cryptographique)
    ABS(RANDOM()) % 1000000 as pseudo_hash
FROM fichiers;

-- Comparaison de tailles
SELECT
    AVG(LENGTH(contenu)) as taille_moyenne,
    MIN(LENGTH(contenu)) as plus_petit,
    MAX(LENGTH(contenu)) as plus_grand,
    SUM(LENGTH(contenu)) as total_stockage
FROM fichiers;
```

### 📸 Cas d'usage typiques pour BLOB

```sql
-- Table pour une galerie photo simple
CREATE TABLE photos (
    id INTEGER PRIMARY KEY,
    titre TEXT,
    description TEXT,
    largeur INTEGER,
    hauteur INTEGER,
    data_image BLOB,        -- L'image elle-même
    miniature BLOB,         -- Miniature
    date_prise TEXT,
    appareil TEXT
);

-- Métadonnées sans les vraies images (trop lourdes pour l'exemple)
INSERT INTO photos (titre, description, largeur, hauteur, date_prise, appareil) VALUES
    ('Sunset', 'Coucher de soleil sur la plage', 1920, 1080, '2024-07-15 19:30:00', 'Canon EOS R'),
    ('Portrait', 'Portrait en studio', 800, 1200, '2024-07-16 14:15:00', 'Nikon D850');

-- Calculs sur les métadonnées
SELECT
    titre,
    largeur || 'x' || hauteur as resolution,
    ROUND(largeur * hauteur / 1000000.0, 1) || ' MP' as megapixels,
    CASE
        WHEN largeur > hauteur THEN 'Paysage'
        WHEN hauteur > largeur THEN 'Portrait'
        ELSE 'Carré'
    END as orientation,
    COALESCE(LENGTH(data_image), 0) as taille_image_bytes,
    date_prise,
    appareil
FROM photos;
```

### ⚠️ Bonnes pratiques avec BLOB

```sql
-- ❌ Éviter pour de gros fichiers
-- SQLite peut théoriquement stocker jusqu'à 1GB par BLOB,
-- mais ce n'est pas recommandé

-- ✅ Mieux : Stocker le chemin du fichier
CREATE TABLE documents_optimise (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    chemin_fichier TEXT,    -- Chemin vers le fichier sur disque
    type_mime TEXT,
    taille INTEGER,
    hash_md5 TEXT,          -- Pour vérifier l'intégrité
    date_modification TEXT
);

INSERT INTO documents_optimise VALUES
    (1, 'Manuel.pdf', '/uploads/2024/07/manuel_v1.2.pdf', 'application/pdf', 2048576, 'a1b2c3d4...', '2024-07-15'),
    (2, 'Photo.jpg', '/uploads/2024/07/IMG_001.jpg', 'image/jpeg', 1536000, 'e5f6g7h8...', '2024-07-15');

-- ✅ BLOB approprié pour de petites données binaires
CREATE TABLE configurations (
    id INTEGER PRIMARY KEY,
    nom_config TEXT,
    parametres BLOB,        -- Petites données de config sérialisées
    version INTEGER
);
```

## NULL - L'absence de valeur

### ❓ Comprendre NULL

NULL représente **l'absence de valeur**, pas une valeur vide ou zéro !

```sql
-- Table démontrant les différents cas de NULL
CREATE TABLE demo_null (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    age INTEGER,
    salaire REAL,
    actif INTEGER,
    commentaires TEXT
);

INSERT INTO demo_null (id, nom, age, salaire, actif, commentaires) VALUES
    (1, 'Alice', 30, 3000.0, 1, 'Employée modèle'),
    (2, 'Bob', NULL, 2500.0, 1, NULL),           -- Âge et commentaires inconnus
    (3, 'Charlie', 25, NULL, 1, ''),             -- Salaire inconnu, commentaire vide
    (4, '', 35, 0.0, 0, 'En congé'),            -- Nom vide (différent de NULL)
    (5, NULL, NULL, NULL, NULL, NULL);          -- Tout est NULL

-- Comprendre les différences
SELECT
    id,
    nom,
    nom IS NULL as nom_est_null,
    nom = '' as nom_est_vide,
    LENGTH(nom) as longueur_nom,
    age,
    age IS NULL as age_inconnue,
    salaire,
    salaire IS NULL as salaire_inconnu,
    salaire = 0 as salaire_zero,
    commentaires,
    commentaires IS NULL as commentaire_null,
    commentaires = '' as commentaire_vide
FROM demo_null;
```

### 🔍 Recherches avec NULL

```sql
-- Recherches impliquant NULL
SELECT * FROM demo_null WHERE age IS NULL;              -- Âges inconnus
SELECT * FROM demo_null WHERE age IS NOT NULL;          -- Âges connus
SELECT * FROM demo_null WHERE salaire IS NULL;          -- Salaires inconnus
SELECT * FROM demo_null WHERE salaire = 0;              -- Salaires à zéro
SELECT * FROM demo_null WHERE commentaires IS NULL;     -- Pas de commentaires
SELECT * FROM demo_null WHERE commentaires = '';        -- Commentaires vides

-- ❌ Erreurs courantes avec NULL
SELECT * FROM demo_null WHERE age = NULL;               -- ❌ Ne fonctionne pas !
SELECT * FROM demo_null WHERE age != NULL;              -- ❌ Ne fonctionne pas !

-- ✅ Syntaxe correcte
SELECT * FROM demo_null WHERE age IS NULL;
SELECT * FROM demo_null WHERE age IS NOT NULL;
```

### 🛠️ Fonctions pour gérer NULL

```sql
-- Fonctions utiles avec NULL
SELECT
    id,
    nom,
    age,
    salaire,
    -- COALESCE : Première valeur non-NULL
    COALESCE(nom, 'Nom inconnu') as nom_avec_defaut,
    COALESCE(age, 0) as age_avec_defaut,
    COALESCE(salaire, 0.0) as salaire_avec_defaut,

    -- NULLIF : NULL si les valeurs sont égales
    NULLIF(nom, '') as nom_null_si_vide,
    NULLIF(salaire, 0) as salaire_null_si_zero,

    -- IFNULL : Equivalent à COALESCE pour 2 valeurs
    IFNULL(commentaires, 'Aucun commentaire') as commentaires_avec_defaut,

    -- Calculs avec NULL
    age + 10 as age_plus_10,                    -- NULL + 10 = NULL
    COALESCE(age, 0) + 10 as age_calcul_sur
FROM demo_null;
```

### 📊 Agrégations et NULL

```sql
-- Comportement des fonctions d'agrégation avec NULL
SELECT
    COUNT(*) as total_lignes,
    COUNT(nom) as noms_non_null,
    COUNT(age) as ages_non_null,
    COUNT(salaire) as salaires_non_null,

    AVG(age) as age_moyen,                      -- Ignore les NULL
    AVG(COALESCE(age, 0)) as age_moyen_avec_zero,

    SUM(salaire) as somme_salaires,             -- Ignore les NULL
    SUM(COALESCE(salaire, 0)) as somme_avec_zero,

    MIN(age) as age_minimum,
    MAX(salaire) as salaire_maximum
FROM demo_null;

-- Statistiques sur les NULL
SELECT
    'Age' as colonne,
    COUNT(age) as valeurs_presentes,
    COUNT(*) - COUNT(age) as valeurs_null,
    ROUND(COUNT(age) * 100.0 / COUNT(*), 2) || '%' as pourcentage_present
FROM demo_null
UNION ALL
SELECT
    'Salaire',
    COUNT(salaire),
    COUNT(*) - COUNT(salaire),
    ROUND(COUNT(salaire) * 100.0 / COUNT(*), 2) || '%'
FROM demo_null;
```

## 🎯 Exercice pratique complet - Système de produits

### Objectif : Maîtriser tous les types de données

```sql
-- === CRÉATION D'UN CATALOGUE PRODUITS COMPLET ===
sqlite3 catalogue_types.db

-- Table principale utilisant tous les types
CREATE TABLE produits_complets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- TEXT : Informations textuelles
    nom TEXT NOT NULL,
    description TEXT,
    reference TEXT UNIQUE,

    -- INTEGER : Nombres entiers et booléens
    stock INTEGER DEFAULT 0,
    stock_minimum INTEGER DEFAULT 5,
    actif INTEGER DEFAULT 1,
    nouveaute INTEGER DEFAULT 0,

    -- REAL : Prix et mesures
    prix_ht REAL NOT NULL,
    taux_tva REAL DEFAULT 0.20,
    poids_kg REAL,

    -- INTEGER pour l'argent (centimes)
    prix_centimes INTEGER,  -- Prix en centimes pour éviter les erreurs

    -- BLOB : Données binaires
    image_miniature BLOB,
    fiche_technique BLOB,

    -- NULL géré explicitement
    date_fin_vie TEXT,      -- NULL = produit toujours en vie
    date_creation TEXT DEFAULT (datetime('now'))
);

-- Table des catégories
CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    description TEXT
);

-- Table de liaison (relation many-to-many)
CREATE TABLE produits_categories (
    produit_id INTEGER,
    categorie_id INTEGER,
    PRIMARY KEY (produit_id, categorie_id),
    FOREIGN KEY (produit_id) REFERENCES produits_complets(id),
    FOREIGN KEY (categorie_id) REFERENCES categories(id)
);

-- === INSERTION DES DONNÉES ===

-- Catégories
INSERT INTO categories (nom, description) VALUES
    ('Électronique', 'Appareils électroniques et gadgets'),
    ('Livre', 'Livres et publications'),
    ('Mode', 'Vêtements et accessoires'),
    ('Maison', 'Articles pour la maison');

-- Produits avec tous types de données
INSERT INTO produits_complets (
    nom, description, reference, stock, actif, nouveaute,
    prix_ht, taux_tva, poids_kg, prix_centimes,
    date_fin_vie
) VALUES
    (
        'Smartphone XY-2024',
        'Smartphone dernière génération avec écran OLED',
        'TEL-XY-2024-001',
        15, 1, 1,
        499.99, 0.20, 0.18, 49999,
        NULL  -- Toujours en vie
    ),
    (
        'Livre "SQLite pour débutants"',
        'Guide complet pour apprendre SQLite de zéro',
        'LIV-SQL-001',
        50, 1, 0,
        29.90, 0.055, 0.35, 2990,  -- Taux de TVA livre : 5.5%
        NULL
    ),
    (
        'T-shirt vintage',
        'T-shirt coton bio collection vintage',
        'VET-TSH-VIN-M',
        8, 1, 0,
        19.90, 0.20, 0.15, 1990,
        '2024-12-31'  -- Fin de collection
    ),
    (
        'Ancien modèle téléphone',
        'Modèle dépassé, liquidation de stock',
        'TEL-OLD-001',
        3, 0, 0,  -- Plus actif
        99.99, 0.20, 0.20, 9999,
        '2024-06-30'  -- Arrêt définitif
    ),
    (
        'Mug personnalisé',
        NULL,  -- Pas de description
        'MAI-MUG-001',
        NULL,  -- Stock inconnu
        1, 1,
        12.50, 0.20, 0.35, 1250,
        NULL
    );

-- Relations produits-catégories
INSERT INTO produits_categories (produit_id, categorie_id) VALUES
    (1, 1),  -- Smartphone → Électronique
    (2, 2),  -- Livre → Livre
    (3, 3),  -- T-shirt → Mode
    (4, 1),  -- Ancien téléphone → Électronique
    (5, 4);  -- Mug → Maison

-- === EXPLORATION DES TYPES DE DONNÉES ===

-- 1. Analyse des types stockés
SELECT
    id,
    nom,
    typeof(nom) as type_nom,
    stock,
    typeof(stock) as type_stock,
    prix_ht,
    typeof(prix_ht) as type_prix,
    actif,
    typeof(actif) as type_actif,
    date_fin_vie,
    typeof(date_fin_vie) as type_date
FROM produits_complets;

-- 2. Gestion des NULL et valeurs par défaut
SELECT
    nom,
    -- TEXT avec NULL
    COALESCE(description, 'Aucune description disponible') as description_complete,
    -- INTEGER avec NULL
    COALESCE(stock, 0) as stock_affiche,
    CASE
        WHEN stock IS NULL THEN '❓ Stock inconnu'
        WHEN stock = 0 THEN '❌ Rupture'
        WHEN stock < 5 THEN '⚠️ Stock faible'
        ELSE '✅ En stock'
    END as statut_stock,
    -- NULL vs valeurs spéciales
    CASE
        WHEN date_fin_vie IS NULL THEN '♾️ Produit permanent'
        WHEN date(date_fin_vie) <= date('now') THEN '🔴 Fin de vie atteinte'
        ELSE '🟡 Fin de vie programmée: ' || date_fin_vie
    END as statut_vie
FROM produits_complets;

-- 3. Conversions et calculs de types
SELECT
    nom,
    reference,
    -- Conversions INTEGER
    stock,
    CAST(stock AS REAL) as stock_decimal,
    CAST(stock AS TEXT) as stock_texte,
    -- Conversions REAL
    prix_ht,
    CAST(prix_ht AS INTEGER) as prix_entier,
    ROUND(prix_ht, 0) as prix_arrondi,
    -- Calculs prix (éviter les erreurs de virgule flottante)
    prix_centimes,
    prix_centimes / 100.0 as prix_depuis_centimes,
    -- Calcul TTC exact en centimes
    ROUND(prix_centimes * (1 + taux_tva)) as prix_ttc_centimes,
    ROUND(prix_centimes * (1 + taux_tva)) / 100.0 as prix_ttc_euros,
    -- Comparaison des méthodes
    prix_ht * (1 + taux_tva) as prix_ttc_float,
    ABS(prix_ht * (1 + taux_tva) - (prix_centimes * (1 + taux_tva) / 100.0)) as difference_precision
FROM produits_complets;

-- 4. Manipulation de chaînes (TEXT)
SELECT
    nom,
    reference,
    -- Analyse des références
    LENGTH(reference) as longueur_ref,
    SUBSTR(reference, 1, 3) as categorie_ref,
    SUBSTR(reference, -3) as fin_ref,
    -- Recherche dans le nom
    CASE
        WHEN LOWER(nom) LIKE '%smartphone%' OR LOWER(nom) LIKE '%téléphone%' THEN 'Téléphonie'
        WHEN LOWER(nom) LIKE '%livre%' THEN 'Littérature'
        WHEN LOWER(nom) LIKE '%shirt%' THEN 'Textile'
        ELSE 'Autre'
    END as famille_deduite,
    -- Nettoyage et formatage
    TRIM(UPPER(SUBSTR(nom, 1, 1)) || LOWER(SUBSTR(nom, 2))) as nom_propre,
    REPLACE(reference, '-', '_') as reference_underscore
FROM produits_complets;

-- 5. Booléens avec INTEGER
SELECT
    nom,
    actif,
    nouveaute,
    -- Logique booléenne
    actif AND nouveaute as actif_et_nouveau,
    actif OR nouveaute as actif_ou_nouveau,
    NOT actif as inactif,
    -- Comptage conditionnel
    CASE WHEN actif = 1 THEN 1 ELSE 0 END as compteur_actif,
    -- Conversion en texte lisible
    CASE actif
        WHEN 1 THEN '✅ Actif'
        ELSE '❌ Inactif'
    END as statut_activite,
    CASE nouveaute
        WHEN 1 THEN '🆕 Nouveau'
        ELSE '📦 Standard'
    END as badge_nouveaute
FROM produits_complets;

-- === REQUÊTES AVANCÉES AVEC LES TYPES ===

-- 6. Statistiques par type de données
SELECT
    'Analyse du catalogue' as titre,
    COUNT(*) as total_produits,
    COUNT(description) as avec_description,
    COUNT(*) - COUNT(description) as sans_description,
    COUNT(stock) as stock_connu,
    COUNT(*) - COUNT(stock) as stock_inconnu,
    SUM(CASE WHEN actif = 1 THEN 1 ELSE 0 END) as produits_actifs,
    SUM(CASE WHEN nouveaute = 1 THEN 1 ELSE 0 END) as nouveautes,
    AVG(prix_ht) as prix_moyen,
    SUM(COALESCE(stock, 0) * prix_centimes) / 100.0 as valeur_stock_total
FROM produits_complets;

-- 7. Analyse de qualité des données
SELECT
    'Contrôle qualité' as section,
    'Prix' as donnee,
    COUNT(CASE WHEN prix_ht != prix_centimes/100.0 THEN 1 END) as incoherences
FROM produits_complets
UNION ALL
SELECT
    'Contrôle qualité',
    'Références',
    COUNT(CASE WHEN LENGTH(reference) < 5 THEN 1 END)
FROM produits_complets
UNION ALL
SELECT
    'Contrôle qualité',
    'Stock négatif',
    COUNT(CASE WHEN stock < 0 THEN 1 END)
FROM produits_complets
UNION ALL
SELECT
    'Contrôle qualité',
    'Prix négatif',
    COUNT(CASE WHEN prix_ht < 0 THEN 1 END)
FROM produits_complets;

-- 8. Recherche multi-critères avec tous les types
SELECT
    p.nom,
    c.nom as categorie,
    p.prix_ht,
    p.stock,
    p.actif,
    p.nouveaute
FROM produits_complets p
JOIN produits_categories pc ON p.id = pc.produit_id
JOIN categories c ON pc.categorie_id = c.id
WHERE
    -- Filtre TEXT
    p.nom LIKE '%phone%' OR p.nom LIKE '%livre%'
    -- Filtre REAL
    AND p.prix_ht BETWEEN 20.0 AND 500.0
    -- Filtre INTEGER (booléen)
    AND p.actif = 1
    -- Filtre NULL
    AND p.date_fin_vie IS NULL
    -- Filtre INTEGER (numérique)
    AND COALESCE(p.stock, 0) > 5;

-- === DÉMONSTRATION DES PIÈGES ET SOLUTIONS ===

-- 9. Pièges courants avec les types
CREATE TABLE demo_pieges (
    id INTEGER PRIMARY KEY,
    valeur_texte TEXT,
    valeur_nombre INTEGER
);

INSERT INTO demo_pieges (valeur_texte, valeur_nombre) VALUES
    ('10', 10),
    ('2', 2),
    ('100', 100),
    ('20', 20);

-- Piège : Tri alphabétique vs numérique
SELECT 'Tri alphabétique du texte' as type_tri, valeur_texte
FROM demo_pieges
ORDER BY valeur_texte
UNION ALL
SELECT 'Tri numérique du texte', valeur_texte
FROM demo_pieges
ORDER BY CAST(valeur_texte AS INTEGER)
UNION ALL
SELECT 'Tri numérique normal', CAST(valeur_nombre AS TEXT)
FROM demo_pieges
ORDER BY valeur_nombre;

-- 10. Conversions et vérifications de types
SELECT
    nom,
    reference,
    stock,
    -- Vérifications de type
    CASE
        WHEN typeof(stock) = 'integer' THEN 'Stock est un entier'
        WHEN typeof(stock) = 'null' THEN 'Stock est NULL'
        ELSE 'Stock a un type inattendu: ' || typeof(stock)
    END as verification_stock,
    -- Conversions sécurisées
    CASE
        WHEN stock IS NULL THEN 'Stock inconnu'
        WHEN CAST(stock AS TEXT) = stock THEN 'Conversion OK'
        ELSE 'Problème de conversion'
    END as test_conversion,
    -- Validation des données
    CASE
        WHEN prix_ht <= 0 THEN 'Prix invalide'
        WHEN LENGTH(reference) < 5 THEN 'Référence trop courte'
        WHEN actif NOT IN (0, 1) THEN 'Statut actif invalide'
        ELSE 'Données OK'
    END as validation
FROM produits_complets;

-- === OPTIMISATIONS ET BONNES PRATIQUES ===

-- 11. Index sur différents types (pour information)
-- CREATE INDEX idx_produits_nom ON produits_complets(nom);           -- TEXT
-- CREATE INDEX idx_produits_prix ON produits_complets(prix_ht);      -- REAL
-- CREATE INDEX idx_produits_stock ON produits_complets(stock);       -- INTEGER
-- CREATE INDEX idx_produits_actif ON produits_complets(actif);       -- INTEGER (booléen)
-- CREATE INDEX idx_produits_ref ON produits_complets(reference);     -- TEXT UNIQUE

-- 12. Rapport final avec tous les types
SELECT
    '=== RAPPORT FINAL DU CATALOGUE ===' as titre
UNION ALL
SELECT '📊 STATISTIQUES GÉNÉRALES'
UNION ALL
SELECT 'Total produits: ' || COUNT(*) FROM produits_complets
UNION ALL
SELECT 'Produits actifs: ' || SUM(actif) FROM produits_complets
UNION ALL
SELECT 'Nouveautés: ' || SUM(nouveaute) FROM produits_complets
UNION ALL
SELECT 'Valeur stock: ' || ROUND(SUM(COALESCE(stock,0) * prix_ht), 2) || '€' FROM produits_complets
UNION ALL
SELECT ''
UNION ALL
SELECT '📈 ANALYSE PAR TYPE'
UNION ALL
SELECT 'Prix moyen: ' || ROUND(AVG(prix_ht), 2) || '€' FROM produits_complets WHERE prix_ht IS NOT NULL
UNION ALL
SELECT 'Stock moyen: ' || ROUND(AVG(COALESCE(stock, 0)), 1) FROM produits_complets
UNION ALL
SELECT 'Poids moyen: ' || ROUND(AVG(poids_kg), 3) || 'kg' FROM produits_complets WHERE poids_kg IS NOT NULL
UNION ALL
SELECT ''
UNION ALL
SELECT '⚠️ ALERTES'
UNION ALL
SELECT 'Produits sans description: ' || (SELECT COUNT(*) FROM produits_complets WHERE description IS NULL)
UNION ALL
SELECT 'Produits sans stock défini: ' || (SELECT COUNT(*) FROM produits_complets WHERE stock IS NULL)
UNION ALL
SELECT 'Produits fin de vie: ' || (SELECT COUNT(*) FROM produits_complets WHERE date_fin_vie IS NOT NULL AND date(date_fin_vie) <= date('now'));
```

## Bonnes pratiques et récapitulatif

### ✅ Choisir le bon type de données

#### 📝 Pour TEXT
- **Utilisez pour** : Noms, descriptions, références, emails, URLs
- **Longueur** : Illimitée (pas de VARCHAR(50) comme MySQL)
- **Recherche** : LIKE, GLOB, fonctions de chaînes
- **Piège** : Attention au tri alphabétique vs numérique

#### 🔢 Pour INTEGER
- **Utilisez pour** : IDs, quantités, booléens (0/1), prix en centimes
- **Plage** : -9,223,372,036,854,775,808 à 9,223,372,036,854,775,807
- **Conversion** : Automatique depuis TEXT numérique
- **Piège** : Division entre entiers donne un résultat réel

#### 💰 Pour REAL
- **Utilisez pour** : Pourcentages, mesures, calculs scientifiques
- **Évitez pour** : Argent (utilisez INTEGER en centimes)
- **Précision** : ~15 chiffres significatifs
- **Piège** : Erreurs de virgule flottante (0.1 + 0.2 ≠ 0.3)

#### 📁 Pour BLOB
- **Utilisez pour** : Petites données binaires, miniatures, configs
- **Évitez pour** : Gros fichiers (préférez chemin sur disque)
- **Limite** : 1GB théorique, mais 1MB recommandé
- **Piège** : Pas de recherche textuelle native

#### ❓ Pour NULL
- **Signifie** : Valeur inconnue/absente (≠ vide ≠ zéro)
- **Opérateurs** : IS NULL, IS NOT NULL (jamais = NULL)
- **Fonctions** : COALESCE, NULLIF, IFNULL
- **Piège** : NULL dans les calculs donne NULL

### 🎯 Patterns recommandés

```sql
-- ✅ Argent en centimes
prix_centimes INTEGER  -- 1250 = 12.50€

-- ✅ Booléens explicites
actif INTEGER CHECK (actif IN (0, 1))

-- ✅ Dates en TEXT ISO
date_creation TEXT DEFAULT (datetime('now'))

-- ✅ Gestion NULL avec défaut
COALESCE(stock, 0) AS stock_display

-- ✅ Validation avec CHECK
prix REAL CHECK (prix > 0)
```

### ❌ Pièges à éviter

```sql
-- ❌ Comparaison directe avec NULL
WHERE age = NULL          -- Ne marche pas !

-- ❌ Division entière inattendue
SELECT 5 / 2;            -- Donne 2.5 (réel), pas 2

-- ❌ Calculs d'argent en REAL
prix REAL                -- Erreurs d'arrondi !

-- ❌ Tri alphabétique sur nombres
ORDER BY numero_texte    -- '10' < '2' !

-- ❌ Oublier les conversions
CAST(texte_nombre AS INTEGER)
```

## Conclusion du chapitre 2.1

### 🎉 Ce que vous avez maîtrisé

**Système de types SQLite :**
- ✅ **5 types principaux** et leur utilisation appropriée
- ✅ **Affinité de type** et conversions automatiques
- ✅ **Flexibilité unique** par rapport aux autres SGBD
- ✅ **Gestion des NULL** et bonnes pratiques

**Techniques pratiques :**
- ✅ Manipulation de chaînes avec TEXT
- ✅ Calculs précis avec INTEGER (centimes)
- ✅ Évitement des pièges REAL
- ✅ Stockage binaire avec BLOB
- ✅ Gestion robuste des NULL

**Patterns de conception :**
- ✅ Stockage d'argent sans erreur
- ✅ Booléens avec INTEGER
- ✅ Validation avec CHECK
- ✅ Conversions sécurisées

### 🚀 Vous savez maintenant

- Choisir le type approprié pour chaque usage
- Éviter les pièges courants de conversion
- Manipuler efficacement tous les types
- Gérer les NULL de manière robuste
- Optimiser le stockage selon le contexte

---

**💡 Dans le prochain chapitre** (2.2), nous verrons comment créer et gérer des bases de données SQLite, en appliquant votre nouvelle maîtrise des types de données.

**🎯 Vous maîtrisez maintenant** les fondations des données SQLite ! Ces connaissances sont essentielles pour tous les chapitres suivants.

⏭️
