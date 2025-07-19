🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.4 Expressions régulières avec REGEXP

## Introduction

Imaginez que vous devez trouver tous les emails valides dans votre base de données, ou tous les titres de livres qui commencent par une voyelle et se terminent par un "e". Avec les fonctions SQL classiques comme LIKE, c'est très compliqué ! Les **expressions régulières** (regex) sont comme un "langage de recherche super-puissant" qui permet de décrire des patterns complexes en quelques caractères.

Pensez aux expressions régulières comme à des "formules magiques" pour chercher des motifs dans le texte !

## ⚠️ Note importante sur SQLite

**SQLite ne supporte pas REGEXP par défaut !** Contrairement à MySQL ou PostgreSQL, vous devez soit :
1. **Compiler SQLite avec l'extension REGEXP** (rarement disponible)
2. **Utiliser une extension** comme `sqlite3-pcre`
3. **Simuler REGEXP** avec d'autres fonctions SQLite

Dans ce tutoriel, nous verrons les **deux approches** :
- Comment utiliser REGEXP (si disponible)
- Comment **simuler les regex** avec les fonctions SQLite natives

## Qu'est-ce qu'une expression régulière ?

Une **expression régulière** (regex) est une séquence de caractères qui décrit un motif de recherche. Elle permet de trouver, valider ou extraire des données selon des règles complexes.

### Exemples simples :
- `^[A-Z]` = Commence par une majuscule
- `@.*\.com$` = Contient @ et se termine par .com
- `\d{4}` = Exactement 4 chiffres
- `[aeiou]` = Contient une voyelle

## Syntaxe de base des expressions régulières

### Caractères spéciaux essentiels

| Symbole | Signification | Exemple | Trouve |
|---------|---------------|---------|--------|
| `.` | N'importe quel caractère | `a.c` | abc, axc, a5c |
| `^` | Début de chaîne | `^Hello` | Hello World |
| `$` | Fin de chaîne | `end$` | The end |
| `*` | 0 ou plus | `ab*c` | ac, abc, abbc |
| `+` | 1 ou plus | `ab+c` | abc, abbc |
| `?` | 0 ou 1 | `ab?c` | ac, abc |
| `\d` | Chiffre (0-9) | `\d+` | 123, 42 |
| `\w` | Lettre/chiffre/_ | `\w+` | hello, test_123 |
| `\s` | Espace blanc | `\s+` | espace, tab |
| `[abc]` | Un parmi a,b,c | `[aeiou]` | voyelles |
| `[^abc]` | Tout sauf a,b,c | `[^aeiou]` | consonnes |
| `{n}` | Exactement n fois | `\d{4}` | 1234 |
| `{n,m}` | Entre n et m fois | `\d{2,4}` | 12, 123, 1234 |

## Préparation des données pour nos exemples

```sql
-- Ajoutons des données avec des patterns intéressants
INSERT INTO clients VALUES
(4, 'Jean-Pierre Dupont', 'jp.dupont@gmail.com', 'France'),
(5, 'mary.smith@yahoo.com', 'mary.smith@yahoo.com', 'USA'),
(6, 'invalid-email', 'not-an-email', 'France'),
(7, 'Dr. Watson', 'watson@holmes.co.uk', 'Royaume-Uni'),
(8, 'Anne-Marie O''Connor', 'anne.marie@hotmail.fr', 'France');

INSERT INTO livres VALUES
(11, 'Alice au pays des merveilles', 19.99, 1, 4, 3, '1865-07-04'),
(12, 'Écrire pour les nuls', 25.50, 2, 6, 8, '2020-01-15'),
(13, '1984', 16.75, 3, 4, 5, '1949-06-08'),
(14, 'Le Guide du routard', 22.00, 1, 6, 2, '2023-03-20'),
(15, 'HTML & CSS', 35.99, 2, 6, 7, '2022-11-10');

-- Table pour exemples de validation
CREATE TABLE donnees_test (
    id INTEGER PRIMARY KEY,
    email TEXT,
    telephone TEXT,
    code_postal TEXT,
    isbn TEXT
);

INSERT INTO donnees_test VALUES
(1, 'user@example.com', '01.23.45.67.89', '75001', '978-2-123456-78-9'),
(2, 'invalid@', '0123456789', '75', '123456789'),
(3, 'test@domain.co.uk', '+33 1 23 45 67 89', '69000', '2-123456-78-9'),
(4, 'bad-email', '01-23-45-67-89', 'ABCDE', 'invalid-isbn'),
(5, 'good@test.org', '06.12.34.56.78', '13001', '978-0-123456-78-X');
```

## Méthode 1 : Utilisation de REGEXP (si disponible)

Si votre installation SQLite supporte REGEXP :

### Syntaxe de base
```sql
SELECT * FROM table WHERE colonne REGEXP 'pattern';
```

### Exemples avec REGEXP

#### 1. Validation d'emails
```sql
-- Emails valides (pattern simplifié)
SELECT nom, email
FROM clients
WHERE email REGEXP '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$';
```

#### 2. Recherche dans les titres
```sql
-- Livres commençant par une voyelle
SELECT titre
FROM livres
WHERE titre REGEXP '^[AEIOUaeiouÀÉÈÊËàéèêë]';

-- Livres contenant des chiffres
SELECT titre
FROM livres
WHERE titre REGEXP '\d';

-- Livres avec des mots de 3 lettres exactement
SELECT titre
FROM livres
WHERE titre REGEXP '\b\w{3}\b';
```

#### 3. Validation de données
```sql
-- Codes postaux français (5 chiffres)
SELECT * FROM donnees_test
WHERE code_postal REGEXP '^\d{5}$';

-- Numéros de téléphone français
SELECT * FROM donnees_test
WHERE telephone REGEXP '^(\+33\s?|0)[1-9](\s?\d{2}){4}$';

-- ISBN valides (format simplifié)
SELECT * FROM donnees_test
WHERE isbn REGEXP '^\d{1,5}-\d{1,7}-\d{1,7}-[\dX]$';
```

## Méthode 2 : Simulation avec les fonctions SQLite natives

Comme REGEXP n'est pas toujours disponible, voici comment simuler les patterns les plus courants :

### Fonctions SQLite utiles pour les patterns

#### GLOB - Pattern matching simple
```sql
-- GLOB utilise * et ? (comme les wildcards système)
-- * = 0 ou plus de caractères
-- ? = exactement 1 caractère
-- [abc] = un caractère parmi a, b, c

-- Emails se terminant par .com
SELECT nom, email FROM clients WHERE email GLOB '*.com';

-- Titres commençant par une majuscule
SELECT titre FROM livres WHERE titre GLOB '[A-Z]*';

-- Codes postaux de 5 chiffres
SELECT * FROM donnees_test WHERE code_postal GLOB '[0-9][0-9][0-9][0-9][0-9]';
```

#### Combinaison de fonctions pour patterns complexes

```sql
-- Validation d'email (méthode basique)
WITH emails_valides AS (
    SELECT
        nom,
        email,
        CASE
            WHEN email LIKE '%@%'
            AND email LIKE '%.%'
            AND LENGTH(email) - LENGTH(REPLACE(email, '@', '')) = 1  -- Un seul @
            AND SUBSTR(email, 1, 1) != '@'  -- Ne commence pas par @
            AND SUBSTR(email, -1, 1) != '@'  -- Ne finit pas par @
            AND SUBSTR(email, INSTR(email, '@') + 1) LIKE '%.%'  -- Domaine avec point
            THEN 'Valide'
            ELSE 'Invalide'
        END as statut_email
    FROM clients
)
SELECT * FROM emails_valides WHERE statut_email = 'Valide';
```

### Patterns complexes avec fonctions SQLite

#### 1. Recherche de mots spécifiques
```sql
-- Livres contenant "guide" (insensible à la casse)
SELECT titre
FROM livres
WHERE UPPER(titre) LIKE '%GUIDE%';

-- Titres avec exactement 3 mots
SELECT titre
FROM livres
WHERE (LENGTH(titre) - LENGTH(REPLACE(titre, ' ', ''))) = 2;  -- 2 espaces = 3 mots

-- Titres commençant par une voyelle
SELECT titre
FROM livres
WHERE UPPER(SUBSTR(titre, 1, 1)) IN ('A', 'E', 'I', 'O', 'U', 'À', 'É', 'È', 'Ê', 'Ë');
```

#### 2. Validation de formats
```sql
-- Numéros de téléphone français (format simple)
WITH validation_tel AS (
    SELECT
        telephone,
        CASE
            WHEN telephone GLOB '[0-9][0-9].[0-9][0-9].[0-9][0-9].[0-9][0-9].[0-9][0-9]' THEN 'Format point'
            WHEN telephone GLOB '[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]' THEN 'Format continu'
            WHEN telephone GLOB '[0-9][0-9]-[0-9][0-9]-[0-9][0-9]-[0-9][0-9]-[0-9][0-9]' THEN 'Format tiret'
            WHEN telephone LIKE '+33%' THEN 'Format international'
            ELSE 'Format invalide'
        END as format_tel
    FROM donnees_test
)
SELECT telephone, format_tel FROM validation_tel;

-- Codes postaux par pattern
SELECT
    code_postal,
    CASE
        WHEN code_postal GLOB '[0-9][0-9][0-9][0-9][0-9]' THEN 'Code postal français'
        WHEN LENGTH(code_postal) != 5 THEN 'Longueur incorrecte'
        ELSE 'Format invalide'
    END as type_code
FROM donnees_test;
```

#### 3. Extraction et nettoyage de données
```sql
-- Extraire le domaine des emails
SELECT
    email,
    CASE
        WHEN email LIKE '%@%' THEN SUBSTR(email, INSTR(email, '@') + 1)
        ELSE 'Email invalide'
    END as domaine
FROM clients;

-- Nettoyer les numéros de téléphone (garder seulement les chiffres)
WITH tel_nettoyes AS (
    SELECT
        telephone,
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(telephone, '.', ''),
                    '-', ''),
                ' ', ''),
            '+33', '0'),
        '(', '') as tel_chiffres
    FROM donnees_test
)
SELECT
    telephone,
    tel_chiffres,
    CASE
        WHEN LENGTH(tel_chiffres) = 10 AND tel_chiffres GLOB '[0-9]*' THEN 'Valide'
        ELSE 'Invalide'
    END as statut
FROM tel_nettoyes;
```

## Cas d'usage pratiques

### 1. Nettoyage et validation de données

```sql
-- Audit complet des données clients
WITH audit_clients AS (
    SELECT
        id,
        nom,
        email,
        pays,
        -- Validation nom
        CASE
            WHEN nom IS NULL OR TRIM(nom) = '' THEN 'Nom manquant'
            WHEN LENGTH(nom) < 2 THEN 'Nom trop court'
            WHEN nom LIKE '%@%' THEN 'Nom contient @'
            ELSE 'Nom valide'
        END as statut_nom,

        -- Validation email
        CASE
            WHEN email IS NULL OR TRIM(email) = '' THEN 'Email manquant'
            WHEN email NOT LIKE '%@%' THEN 'Pas de @'
            WHEN email NOT LIKE '%.%' THEN 'Pas de domaine'
            WHEN LENGTH(email) - LENGTH(REPLACE(email, '@', '')) != 1 THEN 'Plusieurs @'
            WHEN SUBSTR(email, 1, 1) = '@' OR SUBSTR(email, -1, 1) = '@' THEN '@ mal placé'
            WHEN LENGTH(email) < 5 THEN 'Email trop court'
            ELSE 'Email valide'
        END as statut_email,

        -- Validation pays
        CASE
            WHEN pays IS NULL OR TRIM(pays) = '' THEN 'Pays manquant'
            WHEN LENGTH(pays) < 2 THEN 'Pays trop court'
            ELSE 'Pays valide'
        END as statut_pays
    FROM clients
)
SELECT
    id,
    nom,
    email,
    pays,
    statut_nom,
    statut_email,
    statut_pays,
    CASE
        WHEN statut_nom = 'Nom valide'
         AND statut_email = 'Email valide'
         AND statut_pays = 'Pays valide' THEN '✅ Données complètes'
        ELSE '⚠️ Données à corriger'
    END as evaluation_globale
FROM audit_clients
ORDER BY evaluation_globale, id;
```

### 2. Recherche intelligente de livres

```sql
-- Recherche avancée dans les titres
WITH recherche_livres AS (
    SELECT
        L.titre,
        L.prix,
        A.nom as auteur,
        -- Catégorisation par type de titre
        CASE
            WHEN UPPER(L.titre) LIKE '%GUIDE%' OR UPPER(L.titre) LIKE '%MANUEL%' THEN '📖 Guide/Manuel'
            WHEN L.titre GLOB '[0-9]*' THEN '🔢 Commence par chiffre'
            WHEN UPPER(SUBSTR(L.titre, 1, 1)) IN ('A', 'E', 'I', 'O', 'U') THEN '🔤 Commence par voyelle'
            WHEN UPPER(L.titre) LIKE '%&%' OR UPPER(L.titre) LIKE '%ET%' THEN '🔗 Titre composé'
            WHEN LENGTH(L.titre) - LENGTH(REPLACE(L.titre, ' ', '')) >= 3 THEN '📚 Titre long'
            ELSE '📝 Titre simple'
        END as type_titre,

        -- Complexité du titre
        LENGTH(L.titre) as longueur,
        LENGTH(L.titre) - LENGTH(REPLACE(L.titre, ' ', '')) + 1 as nb_mots,

        -- Caractères spéciaux
        CASE
            WHEN L.titre LIKE '%-%' OR L.titre LIKE '%''%' OR L.titre LIKE '%"%' THEN 'Contient ponctuation'
            ELSE 'Ponctuation simple'
        END as ponctuation
    FROM livres L
    JOIN auteurs A ON L.auteur_id = A.id
)
SELECT
    titre,
    auteur,
    prix,
    type_titre,
    nb_mots,
    ponctuation,
    CASE
        WHEN nb_mots <= 2 THEN '📖 Titre court'
        WHEN nb_mots <= 4 THEN '📚 Titre moyen'
        ELSE '📜 Titre long'
    END as categorie_longueur
FROM recherche_livres
ORDER BY type_titre, nb_mots;
```

### 3. Analyse de patterns dans les données de vente

```sql
-- Analyser les patterns de commandes
WITH patterns_commandes AS (
    SELECT
        C.date_commande,
        CL.nom,
        CL.email,
        L.titre,
        -- Pattern temporel
        CASE
            WHEN CAST(strftime('%w', C.date_commande) AS INTEGER) IN (0, 6) THEN '🏖️ Weekend'
            ELSE '💼 Semaine'
        END as periode,

        -- Pattern client (basé sur email)
        CASE
            WHEN CL.email LIKE '%.com' THEN '🌐 Domaine .com'
            WHEN CL.email LIKE '%.fr' THEN '🇫🇷 Domaine .fr'
            WHEN CL.email LIKE '%.org' THEN '🏛️ Domaine .org'
            ELSE '🌍 Autre domaine'
        END as type_domaine,

        -- Pattern livre
        CASE
            WHEN UPPER(L.titre) LIKE '%GUIDE%' THEN '📖 Guide'
            WHEN L.titre GLOB '[0-9]*' THEN '🔢 Titre numérique'
            WHEN LENGTH(L.titre) > 20 THEN '📚 Titre long'
            ELSE '📝 Titre standard'
        END as type_livre
    FROM commandes C
    JOIN clients CL ON C.client_id = CL.id
    JOIN livres L ON C.livre_id = L.id
)
SELECT
    periode,
    type_domaine,
    type_livre,
    COUNT(*) as nb_commandes,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM commandes), 1) as pourcentage
FROM patterns_commandes
GROUP BY periode, type_domaine, type_livre
HAVING nb_commandes > 0
ORDER BY nb_commandes DESC;
```

## Fonctions utiles pour remplacer REGEXP

### 1. Fonction de validation d'email complète

```sql
-- Créer une vue pour validation d'emails
CREATE VIEW emails_analyses AS
WITH validation_email AS (
    SELECT
        id,
        nom,
        email,
        -- Tests de base
        email LIKE '%@%' as a_arobase,
        email LIKE '%.%' as a_point,
        LENGTH(email) - LENGTH(REPLACE(email, '@', '')) = 1 as un_seul_arobase,
        LENGTH(email) >= 5 as longueur_min,
        SUBSTR(email, 1, 1) != '@' as debut_valide,
        SUBSTR(email, -1, 1) NOT IN ('@', '.') as fin_valide,
        -- Tests avancés
        INSTR(email, '@') > 1 as partie_locale_existe,
        LENGTH(SUBSTR(email, INSTR(email, '@') + 1)) > 3 as domaine_suffisant,
        INSTR(SUBSTR(email, INSTR(email, '@') + 1), '.') > 0 as domaine_a_point
    FROM clients
)
SELECT
    id,
    nom,
    email,
    a_arobase AND a_point AND un_seul_arobase AND longueur_min
    AND debut_valide AND fin_valide AND partie_locale_existe
    AND domaine_suffisant AND domaine_a_point as email_valide,
    CASE
        WHEN NOT a_arobase THEN 'Manque @'
        WHEN NOT a_point THEN 'Manque point dans domaine'
        WHEN NOT un_seul_arobase THEN 'Plusieurs @'
        WHEN NOT longueur_min THEN 'Trop court'
        WHEN NOT debut_valide THEN 'Commence par @'
        WHEN NOT fin_valide THEN 'Finit mal'
        WHEN NOT partie_locale_existe THEN 'Pas de nom avant @'
        WHEN NOT domaine_suffisant THEN 'Domaine trop court'
        WHEN NOT domaine_a_point THEN 'Domaine sans extension'
        ELSE 'Email valide'
    END as message_erreur
FROM validation_email;

-- Utilisation
SELECT * FROM emails_analyses WHERE NOT email_valide;
```

### 2. Extraction de patterns

```sql
-- Extraire différentes parties d'informations
WITH extractions AS (
    SELECT
        email,
        -- Extraire le nom d'utilisateur (avant @)
        SUBSTR(email, 1, INSTR(email, '@') - 1) as username,

        -- Extraire le domaine (après @)
        SUBSTR(email, INSTR(email, '@') + 1) as domaine,

        -- Extraire l'extension (après le dernier point)
        SUBSTR(
            email,
            LENGTH(email) - INSTR(REVERSE(email), '.') + 2
        ) as extension
    FROM clients
    WHERE email LIKE '%@%'
)
SELECT
    email,
    username,
    domaine,
    extension,
    -- Analyse du username
    CASE
        WHEN username LIKE '%.%' THEN 'Avec point'
        WHEN username LIKE '%_%' THEN 'Avec underscore'
        WHEN LENGTH(username) <= 3 THEN 'Court'
        ELSE 'Standard'
    END as type_username,
    -- Analyse du domaine
    CASE
        WHEN domaine LIKE 'gmail.%' THEN 'Gmail'
        WHEN domaine LIKE 'yahoo.%' THEN 'Yahoo'
        WHEN domaine LIKE 'hotmail.%' THEN 'Hotmail'
        ELSE 'Autre'
    END as fournisseur
FROM extractions;
```

## Exercices pratiques

### Exercice 1 (Débutant)
Trouvez tous les livres dont le titre commence par une voyelle en utilisant les fonctions SQLite natives.

<details>
<summary>Solution</summary>

```sql
SELECT titre
FROM livres
WHERE UPPER(SUBSTR(titre, 1, 1)) IN ('A', 'E', 'I', 'O', 'U', 'À', 'É', 'È', 'Ê', 'Ë')
ORDER BY titre;
```
</details>

### Exercice 2 (Intermédiaire)
Créez une requête qui valide si les emails de la table `donnees_test` respectent un format basique (contient @, se termine par .com/.fr/.org, au moins 5 caractères).

<details>
<summary>Solution</summary>

```sql
SELECT
    email,
    CASE
        WHEN email LIKE '%@%'
         AND (email LIKE '%.com' OR email LIKE '%.fr' OR email LIKE '%.org')
         AND LENGTH(email) >= 5
         AND LENGTH(email) - LENGTH(REPLACE(email, '@', '')) = 1
        THEN '✅ Valide'
        ELSE '❌ Invalide'
    END as statut,
    CASE
        WHEN email NOT LIKE '%@%' THEN 'Manque @'
        WHEN NOT (email LIKE '%.com' OR email LIKE '%.fr' OR email LIKE '%.org') THEN 'Extension incorrecte'
        WHEN LENGTH(email) < 5 THEN 'Trop court'
        WHEN LENGTH(email) - LENGTH(REPLACE(email, '@', '')) != 1 THEN 'Problème avec @'
        ELSE 'OK'
    END as erreur
FROM donnees_test
ORDER BY statut DESC;
```
</details>

### Exercice 3 (Avancé)
Créez une fonction de "nettoyage intelligent" qui standardise les numéros de téléphone français en format 01.23.45.67.89, en gérant les différents formats d'entrée.

<details>
<summary>Solution</summary>

```sql
WITH nettoyage_telephone AS (
    SELECT
        telephone as original,
        -- Étape 1: Supprimer tous les caractères non-numériques sauf +
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(
                            REPLACE(telephone, ' ', ''),
                        '.', ''),
                    '-', ''),
                '(', ''),
            ')', ''),
        '/', '') as etape1,

        -- Étape 2: Gérer le +33
        CASE
            WHEN REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(
                            REPLACE(
                                REPLACE(telephone, ' ', ''),
                            '.', ''),
                        '-', ''),
                    '(', ''),
                ')', ''),
            '/', '') LIKE '+33%'
            THEN '0' || SUBSTR(
                REPLACE(
                    REPLACE(
                        REPLACE(
                            REPLACE(
                                REPLACE(
                                    REPLACE(telephone, ' ', ''),
                                '.', ''),
                            '-', ''),
                        '(', ''),
                    ')', ''),
                '/', ''), 4)
            ELSE REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(
                            REPLACE(
                                REPLACE(telephone, ' ', ''),
                            '.', ''),
                        '-', ''),
                    '(', ''),
                ')', ''),
            '/', '')
        END as chiffres_seulement
    FROM donnees_test
),
formatage_final AS (
    SELECT
        original,
        chiffres_seulement,
        -- Vérifier si on a exactement 10 chiffres commençant par 0
        CASE
            WHEN LENGTH(chiffres_seulement) = 10
             AND SUBSTR(chiffres_seulement, 1, 1) = '0'
             AND chiffres_seulement GLOB '[0-9]*'
            THEN SUBSTR(chiffres_seulement, 1, 2) || '.' ||
                 SUBSTR(chiffres_seulement, 3, 2) || '.' ||
                 SUBSTR(chiffres_seulement, 5, 2) || '.' ||
                 SUBSTR(chiffres_seulement, 7, 2) || '.' ||
                 SUBSTR(chiffres_seulement, 9, 2)
            ELSE NULL
        END as numero_formate
    FROM nettoyage_telephone
)
SELECT
    original,
    numero_formate,
    CASE
        WHEN numero_formate IS NOT NULL THEN '✅ Formaté avec succès'
        WHEN LENGTH(chiffres_seulement) != 10 THEN '❌ Longueur incorrecte'
        WHEN SUBSTR(chiffres_seulement, 1, 1) != '0' THEN '❌ Ne commence pas par 0'
        WHEN chiffres_seulement NOT GLOB '[0-9]*' THEN '❌ Contient des caractères non-numériques'
        ELSE '❌ Format non reconnu'
    END as statut
FROM formatage_final
ORDER BY statut DESC;
```
</details>

## Ressources et outils

### Extensions SQLite pour REGEXP
- **sqlite3-pcre** : Extension PCRE pour SQLite
- **SQLite avec ICU** : Support Unicode et regex
- **DB Browser for SQLite** : Interface graphique avec recherche

### Alternatives en code
```python
# En Python avec sqlite3
import sqlite3
import re

conn = sqlite3.connect('database.db')

def regexp(expr, item):
    reg = re.compile(expr)
    return reg.search(item) is not None

conn.create_function("REGEXP", 2, regexp)
```

## Résumé

Les **expressions régulières** offrent une puissance incomparable pour la recherche et validation de patterns :

### 🎯 **Capacités principales :**
- **Validation de formats** (emails, téléphones, codes postaux)
- **Recherche de patterns complexes** dans le texte
- **Extraction de données** structurées
- **Nettoyage intelligent** des données

### 🔧 **Approches dans SQLite :**
- **REGEXP natif** : Si extension disponible (idéal)
- **GLOB** : Pattern matching simple mais efficace
- **Fonctions combinées** : LIKE, SUBSTR, INSTR, LENGTH
- **Logique conditionnelle** : CASE WHEN pour validation complexe

### ✅ **Bonnes pratiques :**
- **Commencez simple** : LIKE et GLOB avant les regex complexes
- **Validez étape par étape** : Décomposez les validations complexes
- **Gérez les cas limites** : NULL, chaînes vides, caractères spéciaux
- **Testez thoroughly** : Vérifiez avec des données variées

### 📋 **Cas d'usage essentiels :**
- **Nettoyage de données** : Formats, doublons, erreurs
- **Validation d'entrées** : Emails, téléphones, identifiants
- **Analyse de contenu** : Recherche dans textes, catégorisation
- **Transformation** : Extraction, reformatage, standardisation

Même sans REGEXP natif, SQLite offre des outils puissants pour gérer les patterns de données. La clé est de combiner intelligemment les fonctions disponibles !

Dans la prochaine section, nous découvrirons les **requêtes complexes avec CASE, COALESCE et NULLIF** pour une logique conditionnelle encore plus sophistiquée.

⏭️
