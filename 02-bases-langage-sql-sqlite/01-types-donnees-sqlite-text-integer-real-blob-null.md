🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.1 Types de données SQLite (TEXT, INTEGER, REAL, BLOB, NULL)

## Introduction - La simplicité révolutionnaire de SQLite

SQLite adopte une approche unique et révolutionnaire des types de données qui le distingue de tous les autres systèmes de bases de données. Là où MySQL ou PostgreSQL ont des dizaines de types stricts, SQLite n'en a que **5 classes de stockage** (« storage classes ») avec une **flexibilité extraordinaire**.

> **Analogie simple** : Si les autres SGBD sont comme des tiroirs étiquetés où chaque objet doit aller dans le bon compartiment, SQLite est comme une boîte magique qui s'adapte intelligemment à ce que vous y mettez !

**Les 5 classes de stockage de SQLite :**
- **TEXT** → Chaînes de caractères (UTF-8, UTF-16BE, UTF-16LE)
- **INTEGER** → Nombres entiers signés (1 à 8 octets selon la grandeur)
- **REAL** → Nombres à virgule flottante IEEE 754 sur 8 octets
- **BLOB** → Données binaires brutes
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
    id INTEGER PRIMARY KEY,          -- ⚠️ EXCEPTION : alias du ROWID, doit rester entier
    name TEXT,                       -- Texte de n'importe quelle longueur
    age INTEGER,                     -- Suggère entier, mais accepte du texte numérique
    salary REAL                      -- Suggère décimal, mais accepte du texte numérique
);

-- ✅ Ces insertions fonctionnent toutes — conversion automatique selon l'affinité
INSERT INTO users VALUES (1, 'Alice', 25, 1500.50);  
INSERT INTO users VALUES ('2', 'Bob avec un nom très très long', '30', '2000');  
INSERT INTO users VALUES (3, 'Charlie', 25.8, 1500);  

-- ❌ MAIS attention : INTEGER PRIMARY KEY est UN CAS PARTICULIER
-- C'est un alias du ROWID interne et il N'ACCEPTE QUE des entiers.
INSERT INTO users VALUES (3.5, 'Erreur', 25, 1500);
-- Error: datatype mismatch
-- → Le « manifest typing » général de SQLite ne s'applique PAS au ROWID/INTEGER PRIMARY KEY.
```

### 🎯 Affinité de type - Le secret de SQLite

SQLite utilise le concept d'**affinité de type** : chaque colonne a une **préférence** pour un type, mais peut stocker autre chose si nécessaire. Les auteurs appellent ce système **« manifest typing »** : c'est la **valeur** qui porte son type, pas la **colonne**.

> 💡 **Depuis SQLite 3.37 (novembre 2021)**, on peut activer le mode **`STRICT`** qui rétablit un typage rigide à la manière de MySQL/PostgreSQL :  
> ```sql
> CREATE TABLE produits (
>     id   INTEGER,
>     nom  TEXT,
>     prix REAL
> ) STRICT;   -- Toute valeur de type incompatible sera rejetée
> ```

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

### 📐 Règles d'affinité — comment SQLite « devine » l'affinité d'une colonne

SQLite est très permissif : il accepte n'importe quelle déclaration de type (`VARCHAR(255)`, `BIGINT`, `DECIMAL(10,2)`, `TIMESTAMP`, ou même un mot inventé) et en déduit une **affinité** parmi 5 valeurs (`TEXT`, `NUMERIC`, `INTEGER`, `REAL`, `BLOB`) selon des **règles textuelles** appliquées au type déclaré :

| Si la déclaration contient… | Affinité résultante | Exemples acceptés équivalents |
|------------------------------|---------------------|-------------------------------|
| `INT` (n'importe où) | `INTEGER` | `INT`, `INTEGER`, `BIGINT`, `TINYINT`, `MEDIUMINT` |
| `CHAR`, `CLOB`, `TEXT` | `TEXT` | `TEXT`, `VARCHAR(255)`, `CHAR(10)`, `NCHAR`, `CLOB` |
| `BLOB` ou **aucun type** | `BLOB` (NONE) | `BLOB`, `` (rien) |
| `REAL`, `FLOA`, `DOUB` | `REAL` | `REAL`, `DOUBLE`, `FLOAT`, `DOUBLE PRECISION` |
| (rien des précédents) | `NUMERIC` | `NUMERIC`, `DECIMAL(10,2)`, `BOOLEAN`, `DATE`, `DATETIME` |

> 💡 **Conséquence importante** : `VARCHAR(50)` en SQLite ≡ `TEXT`. La longueur entre parenthèses est **ignorée** ! Une chaîne de 1000 caractères y rentrera sans erreur. Si vous voulez imposer une longueur maximale, utilisez un `CHECK (length(col) <= 50)` ou activez le mode `STRICT`.

```sql
-- Démonstration : toutes ces colonnes ont l'affinité TEXT
CREATE TABLE demo_affinite_text (
    a VARCHAR(255),
    b CHAR(10),
    c TEXT,
    d CLOB,
    e NCHAR(20)
);

INSERT INTO demo_affinite_text VALUES
    ('chaine_de_plus_de_dix_caracteres', 'chaine_de_plus_de_dix_caracteres',
     'chaine_de_plus_de_dix_caracteres', 'chaine_de_plus_de_dix_caracteres',
     'chaine_de_plus_de_dix_caracteres');
-- Aucune limite appliquée — toutes acceptent la chaîne longue !

-- Démonstration : NUMERIC est l'affinité par défaut pour les types « inhabituels »
CREATE TABLE demo_affinite_numeric (
    a NUMERIC,
    b DECIMAL(10,2),
    c BOOLEAN,
    d DATE,
    e TIMESTAMP
);
-- Toutes ces colonnes ont l'affinité NUMERIC : elles tentent de stocker
-- en INTEGER/REAL si possible, sinon en TEXT.
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
-- Recherches textuelles avec LIKE
SELECT * FROM clients WHERE nom LIKE 'Du%';                    -- Commence par "Du"  
SELECT * FROM clients WHERE email LIKE '%@gmail.com';          -- Se termine par "@gmail.com"  
SELECT * FROM clients WHERE nom LIKE '%ar%';                   -- Contient "ar"  
SELECT * FROM clients WHERE LENGTH(prenom) > 6;                -- Prénom long  
SELECT * FROM clients WHERE email NOT LIKE '%@gmail.%';        -- Pas Gmail  

-- Recherche explicitement insensible à la casse (ASCII uniquement — voir l'avertissement Unicode plus bas)
SELECT * FROM clients WHERE LOWER(nom) = 'martin';  
SELECT * FROM clients WHERE LOWER(email) LIKE '%gmail%';  
```

> ⚠️ **Important — `LIKE` et la casse en SQLite** :  
> - **Par défaut, `LIKE` est insensible à la casse pour les caractères ASCII** : `'a' LIKE 'A'` → vrai, et `'Sophie@GMAIL.COM' LIKE '%@gmail.com'` → **vrai** aussi.  
> - **Mais sensible à la casse pour les caractères Unicode non-ASCII** : `'é' LIKE 'É'` → **faux** par défaut.  
> - Pour rendre `LIKE` sensible à la casse partout : `PRAGMA case_sensitive_like = ON;`  
> - ⚠️ **Piège** : `LOWER()` et `UPPER()` natifs de SQLite **ne traitent QUE l'ASCII**. `LOWER('É')` renvoie `É` et non `é` ! Pour une comparaison Unicode-aware, deux options :  
>   1. Charger l'**extension ICU** (`SELECT load_extension('icu')`) qui fournit des `LOWER`/`UPPER` complets et le collation Unicode  
>   2. Normaliser les données côté application avant insertion (stocker une colonne « casefolded »)

> ℹ️ **GLOB et REGEXP** :  
> - **`GLOB`** : matche selon les règles des wildcards Unix (`*`, `?`, `[abc]`), toujours **sensible à la casse**.  
>   ```sql
>   SELECT * FROM clients WHERE nom GLOB 'Du*';                -- Commence par "Du" (sensible)
>   SELECT * FROM clients WHERE telephone GLOB '0[1-9]*';      -- Téléphone FR valide
>   ```
> - **`REGEXP`** : nécessite une extension regex chargée (pas built-in). Voir module 4.4.  
>   ```sql
>   -- ❌ Erreur en SQLite standard : "no such function: REGEXP"
>   -- SELECT * FROM clients WHERE email REGEXP '^[a-z]+@[a-z]+\.(com|fr)$';
>   ```

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
    a / b as division_entiere,      -- ⚠️ INTEGER / INTEGER = INTEGER (troncature) !
    a % b as modulo,
    ABS(a) as valeur_absolue,
    MIN(a, b) as minimum,
    MAX(a, b) as maximum,
    -- Pour obtenir une vraie division réelle, caster au moins un opérande :
    a * 1.0 / b as division_reelle,
    ROUND(a * 1.0 / b, 2) as division_precise
FROM calculs;
-- Exemple : 10/3 = 3 (entier tronqué), 10*1.0/3 = 3.333... (réel)
```

### 🎯 INTEGER comme booléens

```sql
-- SQLite n'a pas de type BOOLEAN distinct : on stocke 0 (faux) ou 1 (vrai) en INTEGER.
-- Bonne pratique : verrouiller les valeurs avec un CHECK
CREATE TABLE produits (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    prix REAL,
    actif         INTEGER NOT NULL DEFAULT 1 CHECK (actif IN (0, 1)),
    en_promotion  INTEGER NOT NULL DEFAULT 0 CHECK (en_promotion IN (0, 1)),
    nouveau       INTEGER NOT NULL DEFAULT 1 CHECK (nouveau IN (0, 1))
);

-- Depuis SQLite 3.23 (2018), les mots-clés TRUE / FALSE sont reconnus
-- comme alias de 1 / 0 — pratique pour la lisibilité.
INSERT INTO produits (nom, prix, actif, en_promotion, nouveau) VALUES
    ('Laptop',         999.99, TRUE,  FALSE, TRUE),
    ('Souris',          25.50, TRUE,  TRUE,  FALSE),
    ('Clavier ancien',  45.00, FALSE, FALSE, FALSE);

-- Requêtes avec booléens
SELECT nom, prix FROM produits WHERE actif = 1;                    -- Produits actifs  
SELECT nom, prix FROM produits WHERE actif AND en_promotion;       -- Actifs ET en promo  
SELECT nom, prix FROM produits WHERE NOT actif;                    -- Produits inactifs  

-- Conversion en texte lisible
SELECT
    nom,
    CASE actif        WHEN 1 THEN 'Actif'        ELSE 'Inactif'     END AS statut,
    CASE en_promotion WHEN 1 THEN 'En promotion' ELSE 'Prix normal' END AS promo
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
    -- Méthode simple : division REAL pour l'affichage
    prix_centimes / 100.0 AS prix_euros,
    -- Méthode formatée : printf gère l'arrondi et le zéro initial
    printf('%.2f €', prix_centimes / 100.0) AS prix_formate,
    devise
FROM produits_bons;

-- Calculs exacts avec les centimes (tout reste INTEGER, pas d'erreur d'arrondi)
SELECT
    nom,
    prix_centimes AS prix_ht_centimes,
    -- TVA 20% : multiplier d'abord, puis diviser pour garder la précision
    prix_centimes * 120 / 100 AS prix_ttc_centimes,
    printf('%.2f €', (prix_centimes * 120 / 100) / 100.0) AS prix_ttc_formate
FROM produits_bons;
-- 💡 Le tour de passe-passe « *120 / 100 » : on multiplie avant de diviser
-- pour éviter de perdre les centimes par troncature INTEGER.
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
-- ⚠️ Subtilité : la colonne BLOB n'a AUCUNE affinité (pas de conversion automatique).
-- Une chaîne 'Hello' est donc stockée comme TEXT, pas comme BLOB.
-- Pour forcer le stockage en BLOB, utiliser la syntaxe X'...' ou CAST(... AS BLOB).
INSERT INTO fichiers (nom_fichier, type_mime, taille, contenu) VALUES
    ('hello.txt',   'text/plain',               13, CAST('Hello, World!' AS BLOB)),
    ('data.bin',    'application/octet-stream',  4, X'DEADBEEF'),
    ('config.json', 'application/json',         27, CAST('{"active":true,"port":8080}' AS BLOB));

-- Voir les données et leur type réel
SELECT
    nom_fichier,
    type_mime,
    taille,
    LENGTH(contenu) AS taille_reelle,    -- en octets pour un BLOB
    typeof(contenu) AS type_sqlite,      -- 'blob' grâce au CAST
    HEX(SUBSTR(contenu, 1, 8)) AS debut_hex
FROM fichiers;
```

### 🔧 Manipulation des BLOB

```sql
-- Fonctions utiles avec BLOB
SELECT
    nom_fichier,
    -- Taille en octets
    LENGTH(contenu) AS taille_octets,
    -- Conversion en hexadécimal (utile pour inspecter du binaire)
    HEX(SUBSTR(contenu, 1, 10)) AS hex_debut,
    -- Recherche textuelle dans le BLOB (fonctionne sur le contenu interprété comme bytes)
    CASE
        WHEN CAST(contenu AS TEXT) LIKE '%json%'  THEN 'Contient "json"'
        WHEN CAST(contenu AS TEXT) LIKE '%Hello%' THEN 'Contient "Hello"'
        ELSE 'Contenu binaire ou autre'
    END AS analyse_contenu,
    -- Pseudo-identifiant aléatoire (non cryptographique)
    ABS(RANDOM()) % 1000000 AS pseudo_hash
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
-- SQLite peut théoriquement stocker jusqu'à 1 milliard d'octets par BLOB
-- (limite dure ≈ 2,1 Go), mais ce n'est pas recommandé en pratique

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

## 📅 Les dates et heures en SQLite — un cas particulier

### Pas de type natif DATE / DATETIME / TIMESTAMP

Contrairement à MySQL ou PostgreSQL, **SQLite n'a aucun type natif pour les dates ou les heures**. Une « date » est toujours représentée par l'une des **5 classes de stockage** existantes. La doc officielle propose **3 conventions** — les exemples ci-dessous représentent tous le **même instant** `2026-05-26 14:30:00 UTC` :

| Convention | Stockage | Exemple (= `2026-05-26 14:30:00 UTC`) | Avantages | Inconvénients |
|------------|----------|---------------------------------------|-----------|---------------|
| **TEXT ISO 8601** | `TEXT` | `'2026-05-26 14:30:00'` | Lisible, triable lexicalement, compatible avec toutes les fonctions `date/time` | Plus volumineux (~19 octets) |
| **REAL Julian Day** | `REAL` | `2461187.10416667` | Compact, calcul d'intervalles direct (1 unité = 1 jour) | Illisible à l'œil nu, précision flottante |
| **INTEGER Unix epoch** | `INTEGER` | `1779805800` | Très compact (8 octets), portable avec autres systèmes | Pas de précision sub-seconde, illisible |

> 💡 **Recommandation pédagogique** : pour cette formation, on utilise **TEXT ISO 8601** par défaut — c'est le format le plus lisible et le plus interopérable. Tous les exemples du module suivent cette convention.

### 🛠️ Les fonctions date/time intégrées

SQLite fournit **5 fonctions** qui acceptent indifféremment les 3 conventions (avec un suffixe `'unixepoch'` ou `'julianday'` pour les non-TEXT) :

```sql
-- Les 5 fonctions principales (résultat = TEXT au format ISO 8601 sauf julianday)
SELECT date('now')             AS aujourd_hui;       -- '2026-05-26'  
SELECT time('now')             AS maintenant_heure;  -- '14:30:00'  
SELECT datetime('now')         AS maintenant;        -- '2026-05-26 14:30:00'  
SELECT julianday('now')        AS julian;            -- 2461187.10417 (REAL)  
SELECT strftime('%Y-%m', 'now') AS annee_mois;       -- '2026-05'  

-- Modificateurs : ajouter/retrancher des intervalles
SELECT date('now', '+7 days')          AS dans_une_semaine;  
SELECT date('now', 'start of month')   AS debut_du_mois;  
SELECT date('2026-05-26', '+1 year', '-3 months') AS combine;  

-- Conversion depuis Unix epoch (1779805800 = 2026-05-26 14:30:00 UTC, cohérent avec le tableau)
SELECT datetime(1779805800, 'unixepoch') AS depuis_epoch;

-- Différence entre 2 dates (en jours)
SELECT julianday('2026-12-31') - julianday('2026-05-26') AS jours_restants;
```

### ⏰ Fuseaux horaires — UTC par défaut

`'now'` renvoie l'heure **en UTC**. Pour obtenir l'heure locale, ajouter le modificateur `'localtime'` :

```sql
SELECT datetime('now')              AS utc;         -- ex: '2026-05-26 12:30:00'  
SELECT datetime('now', 'localtime') AS heure_locale; -- ex: '2026-05-26 14:30:00' (Paris +2)  

-- 💡 Bonne pratique : STOCKER en UTC, AFFICHER en local
-- Cela évite les ambiguïtés autour des changements d'heure et facilite la portabilité.
INSERT INTO logs (date_creation) VALUES (datetime('now'));  -- toujours UTC  
SELECT datetime(date_creation, 'localtime') FROM logs;       -- conversion à l'affichage  
```

### ⚠️ Pièges classiques avec les dates

```sql
-- ❌ Piège 1 : Format non-ISO non trié correctement (lexicographiquement)
CREATE TABLE evenements_bug (date_evt TEXT);  
INSERT INTO evenements_bug VALUES ('26/05/2026'), ('05/05/2026'), ('15/01/2026');  
SELECT date_evt FROM evenements_bug ORDER BY date_evt;  
-- Résultat : '05/05/2026', '15/01/2026', '26/05/2026'  ← tri NON chronologique !

-- ✅ Solution : toujours ISO 8601 (AAAA-MM-JJ)
CREATE TABLE evenements_ok (date_evt TEXT);  
INSERT INTO evenements_ok VALUES ('2026-05-26'), ('2026-05-05'), ('2026-01-15');  
SELECT date_evt FROM evenements_ok ORDER BY date_evt;  
-- Résultat : '2026-01-15', '2026-05-05', '2026-05-26' ← tri chronologique correct

-- ❌ Piège 2 : Les fonctions retournent NULL sur format invalide
SELECT date('26-05-2026');  -- NULL (silencieusement !)

-- ✅ Solution : valider avec un CHECK dès la création
CREATE TABLE rendez_vous (
    id INTEGER PRIMARY KEY,
    date_rdv TEXT CHECK (date(date_rdv) IS NOT NULL)  -- refuse les formats invalides
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
-- ⚠️ Toujours encadrer le bloc OR avec des parenthèses :
--    sans elles, AND lie plus fort que OR, ce qui change le sens !
SELECT
    p.nom,
    c.nom AS categorie,
    p.prix_ht,
    p.stock,
    p.actif,
    p.nouveaute
FROM produits_complets p  
JOIN produits_categories pc ON p.id = pc.produit_id  
JOIN categories c ON pc.categorie_id = c.id  
WHERE  
    -- Filtre TEXT (bloc OR encadré par des parenthèses !)
    (p.nom LIKE '%phone%' OR p.nom LIKE '%livre%')
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
-- ⚠️ Dans une compound SELECT (UNION/UNION ALL), un seul ORDER BY est autorisé
-- à la TOUTE FIN, et il s'applique à l'ensemble du résultat. Pour trier
-- chaque sous-requête séparément, encapsuler chacune dans une sous-requête.

-- Tri alphabétique (le défaut sur du TEXT)
SELECT 'Tri alphabétique' AS type_tri, valeur_texte  
FROM demo_pieges  
ORDER BY valeur_texte;          -- ⚠️ Résultat : '10', '100', '2', '20'  

-- Tri numérique sur du texte : caster en INTEGER
SELECT 'Tri numérique cast' AS type_tri, valeur_texte  
FROM demo_pieges  
ORDER BY CAST(valeur_texte AS INTEGER);   -- ✅ Résultat : '2', '10', '20', '100'  

-- Tri numérique natif (colonne déjà INTEGER)
SELECT 'Tri INTEGER natif' AS type_tri, valeur_nombre  
FROM demo_pieges  
ORDER BY valeur_nombre;          -- ✅ Résultat : 2, 10, 20, 100  

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
- **Longueur** : pratiquement illimitée pour l'usage courant — limite par défaut de **1 milliard d'octets** par chaîne (pas de `VARCHAR(50)` strict comme MySQL)
- **Recherche** : `LIKE`, `GLOB`, fonctions de chaînes
- **Piège** : attention au tri alphabétique vs numérique (`'10' < '2'`)

#### 🔢 Pour INTEGER
- **Utilisez pour** : IDs, quantités, booléens (0/1), prix en centimes
- **Plage** : −9 223 372 036 854 775 808 à 9 223 372 036 854 775 807 (entier signé 64 bits)
- **Conversion** : Automatique depuis TEXT numérique
- **Piège** : `INTEGER / INTEGER = INTEGER` (troncature) ! Pour une division réelle, caster au moins un opérande en REAL (`a * 1.0 / b`)

#### 💰 Pour REAL
- **Utilisez pour** : Pourcentages, mesures, calculs scientifiques
- **Évitez pour** : Argent (utilisez INTEGER en centimes)
- **Précision** : ~15 chiffres significatifs
- **Piège** : Erreurs de virgule flottante (0.1 + 0.2 ≠ 0.3)

#### 📁 Pour BLOB
- **Utilisez pour** : Petites données binaires, miniatures, configs
- **Évitez pour** : Gros fichiers (préférez chemin sur disque)
- **Limite** : 1 milliard d'octets par défaut (limite dure ≈ 2,1 Go) — mais en pratique, < 1 Mo par BLOB recommandé
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
SELECT 5 / 2;            -- Donne 2 (INTEGER/INTEGER tronque vers 0), pas 2.5 !
-- Pour obtenir 2.5 :
SELECT 5 * 1.0 / 2;      -- ou SELECT 5 / 2.0;  ou SELECT CAST(5 AS REAL) / 2;

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
- ✅ **5 classes de stockage** et leur utilisation appropriée
- ✅ **Affinité de type** et conversions automatiques
- ✅ **Manifest typing** et flexibilité unique par rapport aux autres SGBD
- ✅ **Gestion des NULL** et bonnes pratiques
- ✅ **Mode `STRICT`** pour ceux qui préfèrent un typage rigide

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

⏭️ [2.2 Création et gestion des bases de données](/02-bases-langage-sql-sqlite/02-creation-gestion-bases-donnees.md)
