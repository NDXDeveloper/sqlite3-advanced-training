🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.6 Interrogation et manipulation de données JSON

## Introduction

Imaginez que vous devez stocker des informations complexes comme les préférences d'un client, les métadonnées d'un produit, ou des configurations variables. Les colonnes traditionnelles deviennent rapidement limitantes ! Le **JSON** (JavaScript Object Notation) permet de stocker des **données semi-structurées** directement dans vos tables SQLite, offrant la flexibilité du NoSQL avec la puissance du SQL.

Pensez au JSON comme à des **"boîtes flexibles"** qui peuvent contenir des structures de données complexes tout en restant interrogeables avec SQL !

## Qu'est-ce que JSON ?

**JSON** est un format de données textuelles qui permet de représenter des objets, tableaux et valeurs de manière structurée mais flexible.

### Structure JSON de base

```json
{
  "nom": "Jean Dupont",
  "age": 35,
  "actif": true,
  "adresse": {
    "rue": "123 rue de la Paix",
    "ville": "Paris",
    "code_postal": "75001"
  },
  "hobbies": ["lecture", "cinema", "sport"],
  "metadata": null
}
```

### Types de données JSON
- **Objets** : `{"clé": "valeur"}`
- **Tableaux** : `["valeur1", "valeur2"]`
- **Chaînes** : `"texte"`
- **Nombres** : `42`, `3.14`
- **Booléens** : `true`, `false`
- **Null** : `null`

## Support JSON dans SQLite

Depuis SQLite 3.38 (2022), les **fonctions JSON** sont intégrées par défaut ! Voici les principales capacités :

### Validation et vérification
- `JSON(text)` : Valide et normalise du JSON
- `JSON_VALID(text)` : Vérifie si une chaîne est du JSON valide
- `JSON_TYPE(json, path)` : Retourne le type d'un élément

### Extraction de données
- `JSON_EXTRACT(json, path)` : Extrait une valeur (résultat **SQL** — texte sans guillemets)
- `json -> path` : Extrait une valeur **au format JSON** (les chaînes gardent leurs `"…"`, utile pour récupérer un sous-objet ou un tableau)
- `json ->> path` : Extrait une valeur **au format SQL** (équivalent à `JSON_EXTRACT` pour les types primitifs)
- `JSON_EXTRACT(json, path1, path2, ...)` : Extrait plusieurs valeurs

### Modification de données
- `JSON_SET(json, path, value)` : Définit/modifie une valeur
- `JSON_INSERT(json, path, value)` : Insère seulement si absent
- `JSON_REPLACE(json, path, value)` : Remplace seulement si présent
- `JSON_REMOVE(json, path)` : Supprime un élément

### Utilitaires avancés
- `JSON_ARRAY(value1, value2, ...)` : Crée un tableau JSON
- `JSON_OBJECT(key1, value1, key2, value2, ...)` : Crée un objet JSON
- `JSON_ARRAY_LENGTH(json, path)` : Longueur d'un tableau
- `JSON_EACH(json, path)` : Décompose en lignes

## Préparation des données

Créons des tables avec des colonnes JSON pour nos exemples :

```sql
-- Table clients avec préférences JSON
CREATE TABLE clients_json (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    email TEXT,
    preferences JSON,
    metadata JSON
);

-- Table produits avec spécifications JSON
CREATE TABLE produits_json (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    prix REAL,
    specifications JSON,
    tags JSON
);

-- Table commandes avec détails JSON
CREATE TABLE commandes_json (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,
    details JSON,
    statut TEXT,
    date_creation DATE DEFAULT CURRENT_DATE
);

-- Insertion de données avec JSON
INSERT INTO clients_json VALUES
(1, 'Alice Martin', 'alice@email.com',
 JSON('{"langue": "fr", "notifications": {"email": true, "sms": false}, "categories_preferees": ["fiction", "science"], "budget_max": 50}'),
 JSON('{"source": "publicite", "date_inscription": "2024-01-15", "score_fidelite": 85}')),

(2, 'Bob Smith', 'bob@email.com',
 JSON('{"langue": "en", "notifications": {"email": false, "sms": true}, "categories_preferees": ["thriller", "biographie"], "budget_max": 75}'),
 JSON('{"source": "bouche_a_oreille", "date_inscription": "2023-06-20", "score_fidelite": 92}')),

(3, 'Charlie Dubois', 'charlie@email.com',
 JSON('{"langue": "fr", "notifications": {"email": true, "sms": true}, "categories_preferees": ["histoire", "voyage", "cuisine"], "budget_max": 30}'),
 JSON('{"source": "recherche_web", "date_inscription": "2024-03-01", "score_fidelite": 67}')),

-- Clients supplémentaires pour des analyses (cohortes, segmentations…) plus parlantes
(4, 'Diana Lopez', 'diana@email.com',
 JSON('{"langue": "es", "notifications": {"email": true, "sms": false}, "categories_preferees": ["fiction", "voyage"], "budget_max": 60}'),
 JSON('{"source": "publicite", "date_inscription": "2024-02-10", "score_fidelite": 78}')),

(5, 'Eve Tanaka', 'eve@email.com',
 JSON('{"langue": "en", "notifications": {"email": true, "sms": true}, "categories_preferees": ["fiction", "cuisine", "art"], "budget_max": 90}'),
 JSON('{"source": "bouche_a_oreille", "date_inscription": "2023-11-05", "score_fidelite": 95}')),

(6, 'Frank Müller', 'frank@email.com',
 JSON('{"langue": "fr", "notifications": {"email": false, "sms": false}, "categories_preferees": ["histoire", "biographie"], "budget_max": 40}'),
 JSON('{"source": "recherche_web", "date_inscription": "2024-04-01", "score_fidelite": 55}'));

INSERT INTO produits_json VALUES
(1, 'Smartphone Pro', 699.99,
 JSON('{"ecran": {"taille": 6.1, "resolution": "2556x1179", "type": "OLED"}, "processeur": "A17 Pro", "stockage": [128, 256, 512], "couleurs": ["noir", "blanc", "bleu"]}'),
 JSON('["electronique", "mobile", "premium", "nouveau"]')),

(2, 'Livre Cuisine', 29.99,
 JSON('{"pages": 320, "format": "broché", "dimensions": {"longueur": 24, "largeur": 16, "epaisseur": 2.5}, "isbn": "978-2-123456-78-9"}'),
 JSON('["livre", "cuisine", "debutant", "français"]')),

(3, 'Casque Audio', 199.99,
 JSON('{"type": "supra-auriculaire", "sans_fil": true, "autonomie": 30, "reduction_bruit": true, "couleurs": ["noir", "argent"]}'),
 JSON('["audio", "sans-fil", "qualite-premium"]'));

INSERT INTO commandes_json VALUES
(1, 1,
 JSON('{"articles": [{"produit_id": 1, "quantite": 1, "prix_unitaire": 699.99}, {"produit_id": 2, "quantite": 2, "prix_unitaire": 29.99}], "livraison": {"adresse": "123 rue de la Paix, Paris", "methode": "standard", "cout": 5.99}, "paiement": {"methode": "carte", "derniers_chiffres": "1234"}}'),
 'en_cours', '2024-01-20'),

(2, 2,
 JSON('{"articles": [{"produit_id": 3, "quantite": 1, "prix_unitaire": 199.99}], "livraison": {"adresse": "456 Oak Street, London", "methode": "express", "cout": 12.99}, "paiement": {"methode": "paypal", "transaction_id": "TXN123456"}}'),
 'expediee', '2024-01-18'),

(3, 3,
 JSON('{"articles": [{"produit_id": 2, "quantite": 3, "prix_unitaire": 29.99}], "livraison": {"adresse": "789 Avenue des Champs, Lyon", "methode": "standard", "cout": 0}, "paiement": {"methode": "virement", "reference": "VIR789"}}'),
 'livree', '2024-01-15'),

-- Commandes supplémentaires pour avoir une vraie diversité de profils
(4, 4,
 JSON('{"articles": [{"produit_id": 1, "quantite": 1, "prix_unitaire": 699.99}], "livraison": {"adresse": "Calle Mayor 15, Madrid", "methode": "express", "cout": 14.99}, "paiement": {"methode": "carte", "derniers_chiffres": "5678"}}'),
 'livree', '2024-02-12'),

(5, 5,
 JSON('{"articles": [{"produit_id": 2, "quantite": 5, "prix_unitaire": 29.99}, {"produit_id": 3, "quantite": 1, "prix_unitaire": 199.99}], "livraison": {"adresse": "Shibuya 3-1, Tokyo", "methode": "standard", "cout": 8.99}, "paiement": {"methode": "paypal", "transaction_id": "TXN789012"}}'),
 'expediee', '2024-03-08'),

(6, 1,
 JSON('{"articles": [{"produit_id": 3, "quantite": 1, "prix_unitaire": 199.99}], "livraison": {"adresse": "123 rue de la Paix, Paris", "methode": "express", "cout": 12.99}, "paiement": {"methode": "carte", "derniers_chiffres": "1234"}}'),
 'livree', '2024-03-15'),

(7, 2,
 JSON('{"articles": [{"produit_id": 1, "quantite": 1, "prix_unitaire": 699.99}, {"produit_id": 3, "quantite": 2, "prix_unitaire": 199.99}], "livraison": {"adresse": "456 Oak Street, London", "methode": "express", "cout": 19.99}, "paiement": {"methode": "carte", "derniers_chiffres": "9876"}}'),
 'en_cours', '2024-04-22'),

(8, 6,
 JSON('{"articles": [{"produit_id": 2, "quantite": 1, "prix_unitaire": 29.99}], "livraison": {"adresse": "Hauptstraße 5, Berlin", "methode": "standard", "cout": 6.99}, "paiement": {"methode": "virement", "reference": "VIR456"}}'),
 'livree', '2024-04-25');
```

> 📊 **Distribution finale** : 6 clients (3 fr, 2 en, 1 es), 8 commandes étalées de janvier à avril 2024 — assez pour des analyses de **cohortes**, de **profils linguistiques** et de **paniers moyens** par client.

## Extraction de données JSON

### 1. Syntaxes d'extraction

```sql
-- Trois variantes pour extraire des données — leurs résultats DIFFÈRENT pour les chaînes
SELECT
    nom,
    -- Forme fonction : valeur SQL (texte sans guillemets)
    JSON_EXTRACT(preferences, '$.langue') as langue_fonction,    -- → fr
    -- Opérateur `->` : valeur JSON (chaîne avec guillemets `"fr"`)
    preferences -> '$.langue' as langue_operateur,               -- → "fr"
    -- Opérateur `->>` : valeur SQL (équivalent à JSON_EXTRACT pour les types primitifs)
    preferences ->> '$.langue' as langue_texte                   -- → fr
FROM clients_json;
```

> 💡 **Règle simple** : utiliser `->>` ou `JSON_EXTRACT` quand on veut **comparer** ou **utiliser** la valeur en SQL (string, nombre, etc.), et `->` quand on veut **garder le JSON intact** (par exemple pour ré-extraire ensuite ou stocker un sous-objet).

### 2. Chemins JSON (JSON Path)

```sql
-- Exemples de chemins JSON
SELECT
    nom,
    -- Accès direct à une propriété
    preferences ->> '$.langue' as langue,

    -- Accès aux objets imbriqués
    preferences ->> '$.notifications.email' as notif_email,
    preferences ->> '$.notifications.sms' as notif_sms,

    -- Accès aux éléments de tableau par index (commence à 0)
    preferences ->> '$.categories_preferees[0]' as premiere_categorie,
    preferences ->> '$.categories_preferees[1]' as deuxieme_categorie,

    -- Extraction de tout un objet
    preferences -> '$.notifications' as notifications_completes,

    -- Budget maximum
    CAST(preferences ->> '$.budget_max' AS INTEGER) as budget_max
FROM clients_json;
```

### 3. Vérification et validation

```sql
-- Vérifier la validité du JSON et les types
SELECT
    nom,
    -- Validation JSON
    JSON_VALID(preferences) as preferences_valides,
    JSON_VALID(metadata) as metadata_valides,

    -- Types des propriétés
    JSON_TYPE(preferences, '$.langue') as type_langue,
    JSON_TYPE(preferences, '$.notifications') as type_notifications,
    JSON_TYPE(preferences, '$.categories_preferees') as type_categories,
    JSON_TYPE(preferences, '$.budget_max') as type_budget,

    -- Longueur des tableaux
    JSON_ARRAY_LENGTH(preferences, '$.categories_preferees') as nb_categories
FROM clients_json;
```

## Filtrage avec JSON

### 1. Conditions sur les valeurs JSON

```sql
-- Filtrer selon les valeurs JSON
SELECT
    nom,
    preferences ->> '$.langue' as langue,
    CAST(preferences ->> '$.budget_max' AS INTEGER) as budget_max
FROM clients_json  
WHERE  
    -- Langue française
    preferences ->> '$.langue' = 'fr'
    -- ET budget supérieur à 40
    AND CAST(preferences ->> '$.budget_max' AS INTEGER) > 40;

-- Clients avec notifications email activées
-- ⚠️ Les booléens JSON sont convertis en INTEGER par SQLite :
--    `true` → 1, `false` → 0 (l'opérateur `->>` retourne ces entiers,
--    PAS les chaînes "true" / "false"). On compare donc à 1, pas à 'true'.
SELECT
    nom,
    email,
    preferences ->> '$.notifications.email' as email_notifications
FROM clients_json  
WHERE preferences ->> '$.notifications.email' = 1;  -- 1 = true en SQLite  

-- Clients fidèles (score > 80)
SELECT
    nom,
    CAST(metadata ->> '$.score_fidelite' AS INTEGER) as score
FROM clients_json  
WHERE CAST(metadata ->> '$.score_fidelite' AS INTEGER) > 80  
ORDER BY score DESC;  
```

### 2. Recherche dans les tableaux JSON

```sql
-- Recherche dans les tableaux - méthode avec JSON_EACH
SELECT DISTINCT
    C.nom,
    C.preferences ->> '$.categories_preferees' as categories,
    cat.value as categorie_individuelle
FROM clients_json C,
     JSON_EACH(C.preferences, '$.categories_preferees') cat
WHERE cat.value = 'fiction';

-- Clients intéressés par la science ou l'histoire
SELECT DISTINCT
    C.nom,
    cat.value as categorie_preferee
FROM clients_json C,
     JSON_EACH(C.preferences, '$.categories_preferees') cat
WHERE cat.value IN ('science', 'histoire')  
ORDER BY C.nom;  

-- Compter les catégories par client
SELECT
    nom,
    JSON_ARRAY_LENGTH(preferences, '$.categories_preferees') as nb_categories,
    preferences ->> '$.categories_preferees' as categories
FROM clients_json  
ORDER BY nb_categories DESC;  
```

## Modification de données JSON

### 1. JSON_SET - Définir/modifier des valeurs

```sql
-- Modifier les préférences d'un client
-- ⚠️ Subtilité : `JSON_SET(j, '$.x', true)` stocke en réalité l'INTEGER `1`
--    (et `false` → `0`), pas un vrai booléen JSON. Pour stocker un vrai booléen,
--    utiliser `JSON('true')` / `JSON('false')` :
--        JSON_SET(j, '$.sms', JSON('true'))   →  {"sms": true}    -- JSON_TYPE = 'true'
--        JSON_SET(j, '$.sms', true)           →  {"sms": 1}       -- JSON_TYPE = 'integer'
--    Côté lecture, `->>` retourne `1`/`0` dans les deux cas — la différence
--    est visible uniquement via `JSON_TYPE` ou si on relit le JSON brut.
UPDATE clients_json  
SET preferences = JSON_SET(  
    preferences,
    '$.budget_max', 100,                    -- Augmenter le budget
    '$.langue', 'fr',                       -- Changer la langue
    '$.notifications.sms', JSON('true'),    -- Activer SMS (vrai booléen JSON)
    '$.nouvelle_propriete', 'test'          -- Ajouter nouvelle propriété
)
WHERE id = 1;

-- Ajouter une catégorie à la liste (complexe)
UPDATE clients_json  
SET preferences = JSON_SET(  
    preferences,
    '$.categories_preferees',
    JSON_ARRAY(
        JSON_EXTRACT(preferences, '$.categories_preferees[0]'),
        JSON_EXTRACT(preferences, '$.categories_preferees[1]'),
        'nouvelle_categorie'
    )
)
WHERE id = 1 AND JSON_ARRAY_LENGTH(preferences, '$.categories_preferees') = 2;
```

### 2. JSON_INSERT et JSON_REPLACE

```sql
-- JSON_INSERT : ajoute seulement si la clé n'existe pas
UPDATE clients_json  
SET metadata = JSON_INSERT(  
    metadata,
    '$.derniere_connexion', '2024-01-25',    -- Nouvelle clé
    '$.score_fidelite', 999                   -- Ignoré car existe déjà
)
WHERE id = 1;

-- JSON_REPLACE : remplace seulement si la clé existe
UPDATE clients_json  
SET metadata = JSON_REPLACE(  
    metadata,
    '$.score_fidelite', 90,                   -- Remplacé car existe
    '$.cle_inexistante', 'valeur'            -- Ignoré car n'existe pas
)
WHERE id = 1;

-- Vérifier les modifications
SELECT
    nom,
    preferences,
    metadata
FROM clients_json  
WHERE id = 1;  
```

### 3. JSON_REMOVE - Supprimer des éléments

```sql
-- Supprimer des propriétés
UPDATE clients_json  
SET preferences = JSON_REMOVE(  
    preferences,
    '$.nouvelle_propriete',                  -- Supprimer propriété ajoutée
    '$.notifications.sms'                    -- Supprimer notification SMS
)
WHERE id = 1;

-- Supprimer un élément de tableau (par index)
UPDATE produits_json  
SET tags = JSON_REMOVE(tags, '$[0]')        -- Supprimer premier tag  
WHERE id = 1;  
```

## Création de JSON dynamique

### 1. JSON_OBJECT et JSON_ARRAY

```sql
-- Créer des objets JSON dynamiquement
SELECT
    nom,
    -- Créer un objet de résumé
    JSON_OBJECT(
        'nom_client', nom,
        'langue', preferences ->> '$.langue',
        'budget', CAST(preferences ->> '$.budget_max' AS INTEGER),
        'score', CAST(metadata ->> '$.score_fidelite' AS INTEGER),
        'nb_categories', JSON_ARRAY_LENGTH(preferences, '$.categories_preferees')
    ) as resume_client
FROM clients_json;

-- Créer des tableaux JSON
SELECT
    nom,
    JSON_ARRAY(
        nom,
        email,
        preferences ->> '$.langue'
    ) as info_base
FROM clients_json;

-- Combiner avec des requêtes complexes
SELECT
    JSON_OBJECT(
        'total_clients', COUNT(*),
        'langues', JSON_GROUP_ARRAY(DISTINCT preferences ->> '$.langue'),
        'budget_moyen', ROUND(AVG(CAST(preferences ->> '$.budget_max' AS REAL)), 2),
        'score_moyen', ROUND(AVG(CAST(metadata ->> '$.score_fidelite' AS REAL)), 2)
    ) as statistiques_globales
FROM clients_json;
```

### 2. Agrégations JSON

```sql
-- JSON_GROUP_ARRAY : rassemble les valeurs en tableau JSON
SELECT
    preferences ->> '$.langue' as langue,
    COUNT(*) as nb_clients,
    JSON_GROUP_ARRAY(nom) as noms_clients,
    JSON_GROUP_ARRAY(CAST(preferences ->> '$.budget_max' AS INTEGER)) as budgets
FROM clients_json  
GROUP BY preferences ->> '$.langue';  

-- JSON_GROUP_OBJECT : rassemble en objet JSON (clé-valeur)
SELECT
    JSON_GROUP_OBJECT(
        nom,
        CAST(metadata ->> '$.score_fidelite' AS INTEGER)
    ) as scores_par_client
FROM clients_json;
```

## Analyses complexes avec JSON

### 1. Analyse des préférences clients

```sql
-- Statistiques détaillées des préférences
WITH categories_detaillees AS (
    SELECT
        C.nom,
        C.preferences ->> '$.langue' as langue,
        CAST(C.preferences ->> '$.budget_max' AS INTEGER) as budget,
        CAST(C.metadata ->> '$.score_fidelite' AS INTEGER) as score,
        cat.value as categorie
    FROM clients_json C,
         JSON_EACH(C.preferences, '$.categories_preferees') cat
),
stats_categories AS (
    SELECT
        categorie,
        COUNT(*) as nb_clients,
        ROUND(AVG(budget), 2) as budget_moyen,
        ROUND(AVG(score), 2) as score_moyen,
        JSON_GROUP_ARRAY(nom) as clients_interesses
    FROM categories_detaillees
    GROUP BY categorie
)
SELECT
    categorie,
    nb_clients,
    budget_moyen,
    score_moyen,
    clients_interesses,
    -- Analyse de popularité
    CASE
        WHEN nb_clients >= 2 THEN '🔥 Populaire'
        ELSE '📚 Niche'
    END as popularite,
    -- Profil économique
    CASE
        WHEN budget_moyen >= 50 THEN '💎 Premium'
        WHEN budget_moyen >= 35 THEN '💰 Standard'
        ELSE '💵 Budget'
    END as profil_budget
FROM stats_categories  
ORDER BY nb_clients DESC, budget_moyen DESC;  
```

### 2. Analyse des commandes JSON

```sql
-- Décomposer et analyser les commandes
WITH commandes_detaillees AS (
    SELECT
        C.id as commande_id,
        C.client_id,
        CL.nom as client_nom,
        C.statut,
        C.date_creation,
        -- Extraire infos de livraison
        C.details ->> '$.livraison.methode' as methode_livraison,
        CAST(C.details ->> '$.livraison.cout' AS REAL) as cout_livraison,
        C.details ->> '$.paiement.methode' as methode_paiement,
        -- Calculer total articles
        (SELECT SUM(
            CAST(article.value ->> '$.quantite' AS INTEGER) *
            CAST(article.value ->> '$.prix_unitaire' AS REAL)
        ) FROM JSON_EACH(C.details, '$.articles') article) as total_articles
    FROM commandes_json C
    JOIN clients_json CL ON C.client_id = CL.id
)
SELECT
    commande_id,
    client_nom,
    statut,
    methode_livraison,
    cout_livraison,
    methode_paiement,
    ROUND(total_articles, 2) as total_articles,
    ROUND(total_articles + cout_livraison, 2) as total_commande,

    -- Analyse de la commande
    CASE
        WHEN total_articles >= 500 THEN '🛒 Grosse commande'
        WHEN total_articles >= 100 THEN '🛍️ Commande normale'
        ELSE '🛒 Petite commande'
    END as taille_commande,

    -- Analyse livraison
    CASE
        WHEN methode_livraison = 'express' THEN '⚡ Livraison rapide'
        WHEN cout_livraison = 0 THEN '🎁 Livraison gratuite'
        ELSE '📦 Livraison standard'
    END as type_livraison,

    -- Profil paiement
    CASE methode_paiement
        WHEN 'carte' THEN '💳 Paiement carte'
        WHEN 'paypal' THEN '🅿️ PayPal'
        WHEN 'virement' THEN '🏦 Virement'
        ELSE '💰 Autre'
    END as profil_paiement
FROM commandes_detaillees  
ORDER BY total_commande DESC;  
```

### 3. Recommandations personnalisées

```sql
-- Système de recommandations basé sur JSON
WITH profils_clients AS (
    SELECT
        id,
        nom,
        preferences ->> '$.langue' as langue,
        CAST(preferences ->> '$.budget_max' AS INTEGER) as budget_max,
        CAST(metadata ->> '$.score_fidelite' AS INTEGER) as score_fidelite,
        -- Extraire les catégories préférées comme chaîne
        (SELECT GROUP_CONCAT(cat.value, ', ')
         FROM JSON_EACH(preferences, '$.categories_preferees') cat) as categories_str
    FROM clients_json
),
recommandations AS (
    SELECT
        PC.id,
        PC.nom,
        PC.budget_max,
        PC.score_fidelite,
        PC.categories_str,

        -- Recommandations produits
        CASE
            WHEN PC.categories_str LIKE '%fiction%' AND PC.budget_max >= 50 THEN
                '📚 Romans premium recommandés'
            WHEN PC.categories_str LIKE '%cuisine%' AND PC.budget_max >= 25 THEN
                '👨‍🍳 Livres de cuisine spécialisés'
            WHEN PC.budget_max >= 100 THEN
                '📱 Électronique haut de gamme'
            ELSE '📖 Sélection générale'
        END as reco_produits,

        -- Stratégie marketing
        CASE
            WHEN PC.score_fidelite >= 90 THEN '👑 Programme VIP exclusif'
            WHEN PC.score_fidelite >= 80 THEN '🌟 Offres fidélité premium'
            WHEN PC.score_fidelite >= 70 THEN '✨ Récompenses standard'
            ELSE '🎯 Programme de fidélisation'
        END as strategie_marketing,

        -- Canal de communication
        CASE
            WHEN PC.langue = 'fr' THEN '🇫🇷 Communication française'
            WHEN PC.langue = 'en' THEN '🇬🇧 Communication anglaise'
            ELSE '🌍 Communication multilingue'
        END as canal_communication
    FROM profils_clients PC
)
SELECT
    nom,
    budget_max,
    score_fidelite,
    categories_str as categories_preferees,
    reco_produits,
    strategie_marketing,
    canal_communication,

    -- Score priorité pour le marketing
    CASE
        WHEN score_fidelite >= 85 AND budget_max >= 75 THEN 'A+ (Priorité maximale)'
        WHEN score_fidelite >= 75 AND budget_max >= 50 THEN 'A (Priorité haute)'
        WHEN score_fidelite >= 65 OR budget_max >= 40 THEN 'B (Priorité normale)'
        ELSE 'C (Priorité faible)'
    END as priorite_marketing
FROM recommandations  
ORDER BY score_fidelite DESC, budget_max DESC;  
```

## Index et performance avec JSON

### 1. Index sur expressions JSON

```sql
-- Créer des index sur les propriétés JSON fréquemment utilisées
CREATE INDEX idx_clients_langue ON clients_json(JSON_EXTRACT(preferences, '$.langue'));  
CREATE INDEX idx_clients_budget ON clients_json(CAST(JSON_EXTRACT(preferences, '$.budget_max') AS INTEGER));  
CREATE INDEX idx_clients_score ON clients_json(CAST(JSON_EXTRACT(metadata, '$.score_fidelite') AS INTEGER));  

-- Index sur plusieurs propriétés JSON
CREATE INDEX idx_clients_profil ON clients_json(
    JSON_EXTRACT(preferences, '$.langue'),
    CAST(JSON_EXTRACT(metadata, '$.score_fidelite') AS INTEGER)
);
```

### 2. Colonnes calculées pour les performances

```sql
-- Ajouter des colonnes calculées pour les requêtes fréquentes
-- ⚠️ SQLite ne permet d'ajouter qu'une colonne **VIRTUAL** via ALTER TABLE
--    (pas STORED). Si on veut STORED, il faut créer la table avec dès le départ.
ALTER TABLE clients_json ADD COLUMN langue_calc TEXT
    GENERATED ALWAYS AS (JSON_EXTRACT(preferences, '$.langue')) VIRTUAL;

ALTER TABLE clients_json ADD COLUMN budget_calc INTEGER
    GENERATED ALWAYS AS (CAST(JSON_EXTRACT(preferences, '$.budget_max') AS INTEGER)) VIRTUAL;

ALTER TABLE clients_json ADD COLUMN score_calc INTEGER
    GENERATED ALWAYS AS (CAST(JSON_EXTRACT(metadata, '$.score_fidelite') AS INTEGER)) VIRTUAL;

-- Index sur les colonnes calculées (plus efficace)
CREATE INDEX idx_clients_langue_calc ON clients_json(langue_calc);  
CREATE INDEX idx_clients_budget_calc ON clients_json(budget_calc);  
CREATE INDEX idx_clients_score_calc ON clients_json(score_calc);  

-- Requête optimisée avec colonnes calculées
SELECT nom, langue_calc, budget_calc, score_calc  
FROM clients_json  
WHERE langue_calc = 'fr' AND budget_calc > 40  
ORDER BY score_calc DESC;  
```

## Exercices pratiques

### Exercice 1 (Débutant)
Trouvez tous les clients qui ont activé les notifications email et affichez leur nom, email et score de fidélité.

<details>
<summary>Solution</summary>

```sql
SELECT
    nom,
    email,
    CAST(metadata ->> '$.score_fidelite' AS INTEGER) as score_fidelite
FROM clients_json  
WHERE preferences ->> '$.notifications.email' = 1  -- booléen JSON `true` → 1 en SQLite  
ORDER BY score_fidelite DESC;  
```
</details>

### Exercice 2 (Intermédiaire)
Créez un rapport qui montre pour chaque catégorie préférée :
- Le nombre de clients intéressés
- Le budget moyen de ces clients
- La liste des noms des clients

<details>
<summary>Solution</summary>

```sql
WITH categories_clients AS (
    SELECT
        cat.value as categorie,
        C.nom,
        CAST(C.preferences ->> '$.budget_max' AS INTEGER) as budget
    FROM clients_json C,
         JSON_EACH(C.preferences, '$.categories_preferees') cat
)
SELECT
    categorie,
    COUNT(*) as nb_clients,
    ROUND(AVG(budget), 2) as budget_moyen,
    JSON_GROUP_ARRAY(nom) as liste_clients
FROM categories_clients  
GROUP BY categorie  
ORDER BY nb_clients DESC, budget_moyen DESC;  
```
</details>

### Exercice 3 (Avancé)
Créez une analyse complète des commandes qui calcule :
- Le total par commande (articles + livraison)
- Le nombre d'articles différents par commande
- La méthode de paiement et livraison
- Une catégorisation de la commande (petite/moyenne/grosse)
- Des recommandations basées sur le profil

<details>
<summary>Solution</summary>

```sql
WITH analyse_commandes AS (
    SELECT
        CO.id as commande_id,
        CO.client_id,
        CL.nom as client_nom,
        CO.statut,
        CO.date_creation,

        -- Calculs sur les articles
        (SELECT COUNT(*)
         FROM JSON_EACH(CO.details, '$.articles')) as nb_articles_differents,

        (SELECT SUM(CAST(article.value ->> '$.quantite' AS INTEGER))
         FROM JSON_EACH(CO.details, '$.articles') article) as quantite_totale,

        (SELECT SUM(
            CAST(article.value ->> '$.quantite' AS INTEGER) *
            CAST(article.value ->> '$.prix_unitaire' AS REAL)
         ) FROM JSON_EACH(CO.details, '$.articles') article) as total_articles,

        -- Infos livraison et paiement
        CO.details ->> '$.livraison.methode' as methode_livraison,
        CAST(CO.details ->> '$.livraison.cout' AS REAL) as cout_livraison,
        CO.details ->> '$.paiement.methode' as methode_paiement,

        -- Profil client
        CL.preferences ->> '$.langue' as langue_client,
        CAST(CL.preferences ->> '$.budget_max' AS INTEGER) as budget_client,
        CAST(CL.metadata ->> '$.score_fidelite' AS INTEGER) as score_client
    FROM commandes_json CO
    JOIN clients_json CL ON CO.client_id = CL.id
)
SELECT
    commande_id,
    client_nom,
    statut,
    date_creation,
    nb_articles_differents,
    quantite_totale,
    ROUND(total_articles, 2) as total_articles,
    ROUND(cout_livraison, 2) as cout_livraison,
    ROUND(total_articles + cout_livraison, 2) as total_commande,
    methode_livraison,
    methode_paiement,

    -- Catégorisation de la commande
    CASE
        WHEN total_articles + cout_livraison >= 500 THEN '🛒 Grosse commande'
        WHEN total_articles + cout_livraison >= 150 THEN '🛍️ Commande moyenne'
        WHEN total_articles + cout_livraison >= 50 THEN '📦 Commande standard'
        ELSE '🛒 Petite commande'
    END as categorie_commande,

    -- Analyse de la livraison
    CASE
        WHEN methode_livraison = 'express' THEN '⚡ Client pressé'
        WHEN cout_livraison = 0 THEN '🎁 Profite livraison gratuite'
        ELSE '📦 Livraison classique'
    END as profil_livraison,

    -- Recommandations basées sur le profil
    CASE
        WHEN score_client >= 90 AND total_articles + cout_livraison >= 300 THEN
            '👑 Client VIP - Offrir service premium'
        WHEN score_client >= 80 AND nb_articles_differents >= 2 THEN
            '🌟 Client fidèle - Proposer bundle/pack'
        WHEN total_articles + cout_livraison >= 200 AND methode_paiement = 'carte' THEN
            '💳 Gros acheteur - Proposer carte fidélité'
        WHEN quantite_totale >= 3 THEN
            '📚 Acheteur volume - Offrir remise quantité'
        WHEN methode_livraison = 'express' THEN
            '⚡ Client pressé - Marketing instantané'
        ELSE '📋 Suivi marketing standard'
    END as recommandations,

    -- Score de priorité commerciale
    CASE
        WHEN score_client >= 85 AND total_articles + cout_livraison >= 250 THEN 'A+ (Priorité max)'
        WHEN score_client >= 75 AND total_articles + cout_livraison >= 150 THEN 'A (Priorité haute)'
        WHEN score_client >= 65 OR total_articles + cout_livraison >= 100 THEN 'B (Priorité normale)'
        ELSE 'C (Priorité faible)'
    END as priorite_commerciale
FROM analyse_commandes  
ORDER BY total_commande DESC, score_client DESC;  
```
</details>

## Bonnes pratiques avec JSON

### ✅ **Structure et validation :**

1. **Validez toujours vos données JSON**
```sql
-- ✅ BON : Valider avant insertion
INSERT INTO clients_json (nom, preferences)  
SELECT 'Nouveau Client', JSON('{"langue": "fr", "budget_max": 50}')  
WHERE JSON_VALID('{"langue": "fr", "budget_max": 50}');  

-- ⚠️ Vérification en requête
SELECT nom, preferences  
FROM clients_json  
WHERE JSON_VALID(preferences) = 1;  
```

2. **Structure cohérente**

```json
// ✅ BON : Structure standardisée
{
  "langue": "fr",
  "notifications": {
    "email": true,
    "sms": false
  },
  "categories_preferees": ["fiction", "science"],
  "budget_max": 50
}
```

> ❌ **À éviter** : structure incohérente — un client avec `"budget_max"`, un autre avec `"budget_maximum"`, ou des types différents pour la même propriété.

3. **Gestion des types**

```sql
-- ✅ BON : Cast explicite pour les calculs
SELECT
    nom,
    CAST(preferences ->> '$.budget_max' AS INTEGER) as budget
FROM clients_json  
WHERE CAST(preferences ->> '$.budget_max' AS INTEGER) > 50;  
```

```text
-- ❌ PROBLÈME : Comparaison de chaînes (alphabétique, pas numérique)
SELECT … WHERE preferences ->> '$.budget_max' > '50'  -- '5' < '50' < '6' !
```

### ✅ **Performance :**

1. **Index sur les propriétés fréquemment utilisées**

> ⚠️ `ALTER TABLE … ADD COLUMN … GENERATED ALWAYS AS … STORED` est interdit en SQLite — on doit utiliser **VIRTUAL** quand on ajoute une colonne via `ALTER TABLE` (`STORED` n'est possible qu'à la création de la table avec `CREATE TABLE`).

```text
-- Pour les requêtes fréquentes (la colonne `langue_calc` a été ajoutée plus haut en VIRTUAL)
CREATE INDEX idx_langue_alt ON clients_json(JSON_EXTRACT(preferences, '$.langue'));
```

2. **Éviter JSON_EACH dans les gros volumes**
```sql
-- ⚠️ PEUT ÊTRE LENT : JSON_EACH sur gros volumes
SELECT COUNT(*) FROM clients_json C, JSON_EACH(C.preferences, '$.categories_preferees');

-- ✅ PLUS RAPIDE mais APPROXIMATIF : recherche LIKE sur le JSON brut
-- ⚠️ Faux positifs possibles : `%fiction%` matche aussi `"non-fiction"`,
--    `"fictionnaire"`, etc. À réserver aux cas où l'approximation est acceptable,
--    ou en pré-filtrage avant un contrôle exact via JSON_EACH.
SELECT COUNT(*) FROM clients_json  
WHERE JSON_EXTRACT(preferences, '$.categories_preferees') LIKE '%"fiction"%';  
```

> 💡 Pour une **recherche exacte** sans faux positifs, garder `JSON_EACH` et  
>    ajouter un index sur une colonne calculée si nécessaire (voir section  
>    « Index et performance avec JSON » plus haut).

### ✅ **Maintenance :**

1. **Documentation des structures JSON**
```sql
-- Documenter la structure attendue
/*
Structure preferences :
{
  "langue": "fr|en|es",           -- Code langue ISO
  "notifications": {
    "email": boolean,             -- Notifications par email
    "sms": boolean                -- Notifications par SMS
  },
  "categories_preferees": [string], -- Liste des catégories
  "budget_max": number            -- Budget maximum en euros
}
*/
```

2. **Vérifications de cohérence**
```sql
-- Contrôle qualité des données JSON
SELECT
    id,
    nom,
    CASE
        WHEN NOT JSON_VALID(preferences) THEN '❌ JSON invalide'
        WHEN JSON_TYPE(preferences, '$.langue') != 'text' THEN '⚠️ Langue invalide'
        WHEN JSON_TYPE(preferences, '$.budget_max') != 'integer' THEN '⚠️ Budget invalide'
        WHEN JSON_TYPE(preferences, '$.categories_preferees') != 'array' THEN '⚠️ Catégories invalides'
        ELSE '✅ Structure valide'
    END as qualite_json
FROM clients_json;
```

## Migrations et évolution du schéma JSON

### Ajouter de nouvelles propriétés

```sql
-- Ajouter une propriété à tous les clients existants
UPDATE clients_json  
SET preferences = JSON_SET(  
    preferences,
    '$.date_derniere_maj', DATE('now'),
    '$.version_schema', 2
);

-- Ajouter avec valeur par défaut conditionnelle
UPDATE clients_json  
SET preferences = JSON_SET(  
    preferences,
    '$.preferences_marketing',
    JSON_OBJECT(
        'newsletter', true,
        'offres_speciales',
        CASE
            WHEN CAST(JSON_EXTRACT(metadata, '$.score_fidelite') AS INTEGER) >= 80 THEN true
            ELSE false
        END
    )
)
WHERE JSON_EXTRACT(preferences, '$.preferences_marketing') IS NULL;
```

### Restructurer les données JSON

```sql
-- Migration : déplacer une propriété
WITH migration AS (
    SELECT
        id,
        -- Extraire l'ancienne valeur
        JSON_EXTRACT(preferences, '$.old_property') as old_value,
        -- Supprimer l'ancienne propriété
        JSON_REMOVE(preferences, '$.old_property') as prefs_cleaned
    FROM clients_json
    WHERE JSON_EXTRACT(preferences, '$.old_property') IS NOT NULL
)
UPDATE clients_json  
SET preferences = JSON_SET(  
    (SELECT prefs_cleaned FROM migration WHERE migration.id = clients_json.id),
    '$.new_location.new_property',
    (SELECT old_value FROM migration WHERE migration.id = clients_json.id)
)
WHERE id IN (SELECT id FROM migration);
```

## Cas d'usage avancés

### 1. Configuration dynamique d'application

```sql
-- Table de configuration avec JSON
CREATE TABLE app_config (
    id INTEGER PRIMARY KEY,
    module TEXT NOT NULL,
    config JSON NOT NULL,
    version TEXT,
    actif BOOLEAN DEFAULT 1,
    date_maj DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO app_config VALUES
(1, 'paiement', JSON('{
  "methodes_acceptees": ["carte", "paypal", "virement"],
  "limites": {
    "min_commande": 5.0,
    "max_commande": 5000.0,
    "max_panier": 10000.0
  },
  "frais": {
    "carte": 0,
    "paypal": 0.25,
    "virement": 0
  },
  "devises_acceptees": ["EUR", "USD", "GBP"]
}'), '1.2', 1, CURRENT_TIMESTAMP),

(2, 'livraison', JSON('{
  "zones": [
    {"nom": "France", "delai": "2-3 jours", "cout": 5.99, "gratuit_au_dessus": 50},
    {"nom": "Europe", "delai": "5-7 jours", "cout": 12.99, "gratuit_au_dessus": 100},
    {"nom": "Monde", "delai": "7-14 jours", "cout": 25.99, "gratuit_au_dessus": 200}
  ],
  "methodes": ["standard", "express", "chronopost"],
  "surcout_express": 10.0
}'), '1.0', 1, CURRENT_TIMESTAMP);

-- Requête pour obtenir la configuration de paiement
SELECT
    module,
    config ->> '$.methodes_acceptees' as methodes,
    CAST(config ->> '$.limites.min_commande' AS REAL) as min_commande,
    CAST(config ->> '$.limites.max_commande' AS REAL) as max_commande,
    config -> '$.frais' as frais_par_methode
FROM app_config  
WHERE module = 'paiement' AND actif = 1;  
```

### 2. Système de logs JSON

```sql
-- Table de logs avec événements JSON
CREATE TABLE logs_evenements (
    id INTEGER PRIMARY KEY,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    niveau TEXT CHECK(niveau IN ('INFO', 'WARN', 'ERROR', 'DEBUG')),
    module TEXT,
    message TEXT,
    contexte JSON,
    utilisateur_id INTEGER
);

-- Insérer des logs avec contexte riche
INSERT INTO logs_evenements (niveau, module, message, contexte, utilisateur_id) VALUES
('INFO', 'commande', 'Nouvelle commande créée', JSON('{
  "commande_id": 123,
  "montant_total": 89.99,
  "nb_articles": 3,
  "methode_paiement": "carte",
  "adresse_livraison": {"ville": "Paris", "code_postal": "75001"},
  "duree_session": 1250,
  "source_traffic": "recherche_google"
}'), 1),

('WARN', 'paiement', 'Tentative de paiement échouée', JSON('{
  "raison": "carte_expiree",
  "montant": 150.00,
  "tentative_numero": 2,
  "carte_type": "visa",
  "derniers_chiffres": "1234",
  "code_retour_banque": "E051"
}'), 2),

('ERROR', 'stock', 'Stock insuffisant', JSON('{
  "produit_id": 456,
  "stock_actuel": 1,
  "quantite_demandee": 3,
  "commande_id": 789,
  "action_prise": "annulation_partielle"
}'), 1);

-- Analyse des erreurs par type
SELECT
    niveau,
    module,
    COUNT(*) as nb_occurrences,
    -- Extraire les raisons d'erreur les plus fréquentes
    JSON_GROUP_ARRAY(contexte ->> '$.raison') as raisons,
    MIN(timestamp) as premiere_occurrence,
    MAX(timestamp) as derniere_occurrence
FROM logs_evenements  
WHERE niveau IN ('WARN', 'ERROR')  
AND timestamp >= DATE('now', '-7 days')  
GROUP BY niveau, module  
ORDER BY nb_occurrences DESC;  

-- Analyse détaillée des paiements échoués
SELECT
    DATE(timestamp) as jour,
    COUNT(*) as nb_echecs,
    JSON_GROUP_ARRAY(DISTINCT contexte ->> '$.raison') as raisons_echec,
    AVG(CAST(contexte ->> '$.montant' AS REAL)) as montant_moyen_echec,
    SUM(CAST(contexte ->> '$.montant' AS REAL)) as perte_ca_estimee
FROM logs_evenements  
WHERE module = 'paiement'  
AND niveau = 'WARN'  
AND message LIKE '%échouée%'  
GROUP BY DATE(timestamp)  
ORDER BY jour DESC;  
```

### 3. Système de paramétrage utilisateur avancé

```sql
-- Paramètres utilisateur ultra-flexibles
CREATE TABLE parametres_utilisateur (
    utilisateur_id INTEGER PRIMARY KEY,
    parametres JSON NOT NULL DEFAULT '{}',
    derniere_maj DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO parametres_utilisateur VALUES
(1, JSON('{
  "interface": {
    "theme": "sombre",
    "langue": "fr",
    "timezone": "Europe/Paris",
    "notifications_desktop": true,
    "sons": false
  },
  "shopping": {
    "tri_defaut": "prix_croissant",
    "affichage": "grille",
    "filtres_sauvegardes": [
      {"nom": "Mes livres", "criteres": {"type": "livre", "prix_max": 50}},
      {"nom": "Électronique", "criteres": {"type": "electronique", "marque": ["Apple", "Samsung"]}}
    ],
    "historique_recherches": ["smartphone", "livre cuisine", "casque audio"]
  },
  "privacy": {
    "partage_donnees": false,
    "cookies_marketing": true,
    "analyse_comportement": true,
    "newsletter": true
  },
  "paiement": {
    "methode_preferee": "carte",
    "sauvegarde_carte": true,
    "factures_email": true,
    "devise_preferee": "EUR"
  }
}'), CURRENT_TIMESTAMP);

-- Requête pour personnaliser l'expérience utilisateur
-- ⚠️ `CAST(… AS BOOLEAN)` est un NO-OP en SQLite : aucun type BOOLEAN strict n'existe,
--    le CAST applique l'affinité NUMERIC et retourne directement l'INTEGER (1/0).
--    Comme `->>` retourne déjà 1/0 pour les booléens JSON, ce CAST est **purement
--    documentaire** — il signale l'intention « cette colonne est un booléen » au lecteur.
--    À utiliser comme convention de lecture, pas comme conversion réelle.
SELECT
    utilisateur_id,
    -- Paramètres d'interface
    parametres ->> '$.interface.theme' as theme,
    parametres ->> '$.interface.langue' as langue,
    CAST(parametres ->> '$.interface.notifications_desktop' AS BOOLEAN) as notif_desktop,  -- intention : booléen

    -- Paramètres shopping
    parametres ->> '$.shopping.tri_defaut' as tri_prefere,
    parametres ->> '$.shopping.affichage' as mode_affichage,
    JSON_ARRAY_LENGTH(parametres, '$.shopping.filtres_sauvegardes') as nb_filtres_sauvegardes,

    -- Préférences privacy
    CAST(parametres ->> '$.privacy.partage_donnees' AS BOOLEAN) as accepte_partage,    -- intention : booléen
    CAST(parametres ->> '$.privacy.newsletter' AS BOOLEAN) as accepte_newsletter,      -- intention : booléen

    -- Configuration paiement
    parametres ->> '$.paiement.methode_preferee' as methode_paiement_pref,
    parametres ->> '$.paiement.devise_preferee' as devise,

    -- Score de personnalisation (plus il y a de paramètres, plus c'est personnalisé)
    CASE
        WHEN JSON_ARRAY_LENGTH(parametres, '$.shopping.filtres_sauvegardes') >= 3 THEN 'Très personnalisé'
        WHEN JSON_ARRAY_LENGTH(parametres, '$.shopping.filtres_sauvegardes') >= 1 THEN 'Personnalisé'
        ELSE 'Paramètres par défaut'
    END as niveau_personnalisation
FROM parametres_utilisateur;

-- Mise à jour intelligente des paramètres
-- Ici, on conserve uniquement les 3 dernières recherches (1 nouvelle + 2 anciennes)
-- pour rester lisible. Pour en garder davantage, ajouter d'autres lignes
-- `JSON_EXTRACT(... '[3]')`, `'[4]'`, etc.
UPDATE parametres_utilisateur  
SET parametres = JSON_SET(  
    parametres,
    '$.shopping.historique_recherches',
    JSON_ARRAY(
        'nouvelle_recherche',  -- La nouvelle recherche en tête de liste
        COALESCE(JSON_EXTRACT(parametres, '$.shopping.historique_recherches[0]'), ''),
        COALESCE(JSON_EXTRACT(parametres, '$.shopping.historique_recherches[1]'), '')
    )
),
derniere_maj = CURRENT_TIMESTAMP  
WHERE utilisateur_id = 1;  
```

## Résumé

Le **JSON dans SQLite** offre une flexibilité exceptionnelle pour les données modernes :

### 🎯 **Capacités principales :**
- **Stockage flexible** : Structures complexes dans une seule colonne
- **Interrogation native** : Syntaxe SQL standard pour extraire/modifier
- **Validation intégrée** : Fonctions de vérification et typage
- **Performance optimisée** : Index et colonnes calculées possibles

### 🔧 **Fonctions essentielles :**
- **Extraction** : `JSON_EXTRACT()`, `->`, `->>`
- **Modification** : `JSON_SET()`, `JSON_INSERT()`, `JSON_REPLACE()`, `JSON_REMOVE()`
- **Creation** : `JSON_OBJECT()`, `JSON_ARRAY()`, `JSON_GROUP_ARRAY()`
- **Validation** : `JSON_VALID()`, `JSON_TYPE()`, `JSON_ARRAY_LENGTH()`

### 📊 **Cas d'usage idéaux :**
- **Configuration dynamique** : Paramètres variables par entité
- **Métadonnées flexibles** : Propriétés évolutives
- **Logs structurés** : Contexte riche pour les événements
- **Préférences utilisateur** : Personnalisation avancée
- **API modernes** : Données semi-structurées

### ✅ **Bonnes pratiques :**
- **Validation systématique** du JSON avant stockage
- **Structure cohérente** au sein d'une même colonne
- **Index sur propriétés fréquentes** ou colonnes calculées
- **Documentation des structures** attendues
- **Cast explicite** pour les calculs numériques

### ⚡ **Performance :**
- **Colonnes calculées** pour les requêtes fréquentes
- **Index sur expressions JSON** pour l'optimisation
- **Éviter JSON_EACH** sur de gros volumes quand possible
- **Requêtes directes** préférables aux décompositions

Le JSON dans SQLite combine le meilleur des deux mondes : la **flexibilité du NoSQL** et la **puissance du SQL relationnel**. C'est l'outil parfait pour les applications modernes qui doivent gérer des données évolutives tout en conservant la robustesse d'une base relationnelle !

Ce chapitre 4 sur les **requêtes avancées et l'optimisation** vous a maintenant donné tous les outils pour :
- Maîtriser les sous-requêtes complexes
- Structurer vos requêtes avec les CTE
- Analyser vos données avec les fonctions de fenêtrage
- Valider et rechercher avec les expressions régulières
- Appliquer une logique métier sophistiquée
- Gérer des données modernes avec JSON

Vous êtes maintenant équipé pour créer des solutions SQL avancées et performantes !

⏭️ [Module 5 : Optimisation des performances](/05-optimisation-performances/README.md)

