🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.1 Normalisation des données (1NF, 2NF, 3NF)

## Introduction - L'art d'organiser les données

La normalisation est comme **ranger efficacement une bibliothèque** : au lieu d'empiler tous les livres n'importe comment, vous les organisez logiquement pour éviter les doublons, faciliter les recherches et maintenir l'ordre à long terme.

> **Analogie simple** : Imaginez votre collection de musique. Plutôt que de dupliquer les informations de l'artiste dans chaque chanson, vous créez une liste d'artistes séparée et vous y faites référence. C'est exactement l'esprit de la normalisation !

**Les 3 formes normales essentielles :**
- **1NF (Première Forme Normale)** → Éliminer les valeurs multiples
- **2NF (Deuxième Forme Normale)** → Éliminer les dépendances partielles
- **3NF (Troisième Forme Normale)** → Éliminer les dépendances transitives

> 💡 **Comment tester les exemples** : ce chapitre illustre **plusieurs schémas distincts** (1NF, 2NF, 3NF, schéma final, bibliothèque…) avec parfois des noms de tables qui se recoupent (`etudiants`, `telephones_etudiants`, etc.). Chaque grand exemple est conçu pour être exécuté **dans une base vierge**. Lancez `sqlite3 nouvelle_base.db` (ou recréez une base en mémoire avec `sqlite3 :memory:`) entre deux sections principales pour éviter les erreurs « table already exists ».

## Pourquoi normaliser ? Les problèmes de la dénormalisation

### 🚨 Exemple concret : Base d'école mal conçue

Commençons par voir ce qui se passe **sans normalisation** :

```sql
-- ❌ Table dénormalisée - TOUS les problèmes réunis !
CREATE TABLE etudiants_cours_mauvais (
    id INTEGER PRIMARY KEY,
    nom_etudiant TEXT,
    email_etudiant TEXT,
    telephone_etudiant TEXT,
    classe TEXT,

    -- Informations du cours
    nom_cours TEXT,
    code_cours TEXT,
    credits_cours INTEGER,

    -- Informations du professeur
    nom_professeur TEXT,
    email_professeur TEXT,
    departement_professeur TEXT,

    -- Note
    note REAL,
    date_examen TEXT
);

-- Insertion de données... et c'est le cauchemar !
INSERT INTO etudiants_cours_mauvais VALUES
    (1, 'Alice Martin', 'alice@email.com', '0123456789', 'L3 Info',
     'Base de données', 'BDD101', 6,
     'Dr. Dupont', 'dupont@prof.com', 'Informatique',
     15.5, '2024-06-15'),
    (2, 'Alice Martin', 'alice@email.com', '0123456789', 'L3 Info',
     'Programmation', 'PROG201', 8,
     'Dr. Martin', 'martin@prof.com', 'Informatique',
     17.0, '2024-06-20'),
    (3, 'Bob Leroy', 'bob@email.com', '0987654321', 'L2 Info',
     'Base de données', 'BDD101', 6,
     'Dr. Dupont', 'dupont@prof.com', 'Informatique',
     12.5, '2024-06-15');
```

### 🔥 Les problèmes qui apparaissent immédiatement

```sql
-- Voir les horreurs de cette conception
SELECT * FROM etudiants_cours_mauvais;
```

**Problèmes identifiés :**

1. **Redondance massive** :
   - Alice apparaît 2 fois avec toutes ses infos
   - Dr. Dupont répété pour chaque étudiant de son cours

2. **Incohérences potentielles** :
   - Et si on tape mal l'email d'Alice dans une ligne ?
   - Et si on change le département du Dr. Dupont dans une seule ligne ?

3. **Anomalies de mise à jour** :
   - Changer l'email d'Alice = modifier plusieurs lignes
   - Risque d'oublier certaines lignes

4. **Anomalies d'insertion** :
   - Impossible d'ajouter un nouveau cours sans étudiant
   - Impossible d'ajouter un étudiant sans cours

5. **Anomalies de suppression** :
   - Supprimer le dernier étudiant d'un cours = perdre les infos du cours
   - Supprimer un étudiant = perdre potentiellement des infos prof

## Première Forme Normale (1NF) - Atomicité des données

### 🎯 Règle 1NF : Une valeur par cellule

**Principe** : Chaque cellule doit contenir une **valeur atomique** (indivisible), pas une liste ou un ensemble de valeurs.

### ❌ Violations typiques de 1NF

```sql
-- Table violant la 1NF
CREATE TABLE etudiants_mauvais_1nf (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    telephones TEXT,           -- ❌ Plusieurs téléphones séparés par ';'
    matieres_suivies TEXT,     -- ❌ Liste de matières
    notes TEXT,                -- ❌ Notes correspondantes
    langues TEXT               -- ❌ Langues parlées
);

INSERT INTO etudiants_mauvais_1nf VALUES
    (1, 'Alice Martin', '0123456789;0987654321',
     'Maths;Physique;Chimie', '15;17;14', 'Français;Anglais;Espagnol'),
    (2, 'Bob Leroy', '0555666777',
     'Informatique;Maths', '16;18', 'Français;Anglais');

-- Cauchemar pour les requêtes !
SELECT * FROM etudiants_mauvais_1nf;

-- Comment trouver tous les étudiants qui suivent "Maths" ?
-- Comment calculer la moyenne en Maths ?
-- Comment gérer l'ajout d'une nouvelle matière ?
```

### ✅ Solution 1NF : Décomposition atomique

```sql
-- Étape 1 : Séparer les informations de base
CREATE TABLE etudiants_1nf (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    email TEXT UNIQUE
);

-- Étape 2 : Tables séparées pour les valeurs multiples
-- Les noms portent le suffixe `_1nf` pour ne pas entrer en conflit avec
-- les tables `telephones_etudiants` et `langues_etudiants` du schéma final 3NF
-- plus bas dans le chapitre.
CREATE TABLE telephones_etudiants_1nf (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    etudiant_id INTEGER,
    telephone TEXT,
    type_telephone TEXT, -- 'mobile', 'fixe', 'urgence'
    FOREIGN KEY (etudiant_id) REFERENCES etudiants_1nf(id)
);

CREATE TABLE langues_etudiants_1nf (
    etudiant_id INTEGER,
    langue TEXT,
    niveau TEXT, -- 'débutant', 'intermédiaire', 'courant', 'natif'
    PRIMARY KEY (etudiant_id, langue),
    FOREIGN KEY (etudiant_id) REFERENCES etudiants_1nf(id)
);

-- Données en 1NF
INSERT INTO etudiants_1nf (nom, email) VALUES
    ('Alice Martin', 'alice@email.com'),
    ('Bob Leroy', 'bob@email.com');

INSERT INTO telephones_etudiants_1nf (etudiant_id, telephone, type_telephone) VALUES
    (1, '0123456789', 'mobile'),
    (1, '0987654321', 'fixe'),
    (2, '0555666777', 'mobile');

INSERT INTO langues_etudiants_1nf VALUES
    (1, 'Français', 'natif'),
    (1, 'Anglais', 'courant'),
    (1, 'Espagnol', 'intermédiaire'),
    (2, 'Français', 'natif'),
    (2, 'Anglais', 'courant');

-- Maintenant les requêtes sont possibles !
SELECT e.nom, t.telephone, t.type_telephone  
FROM etudiants_1nf e  
JOIN telephones_etudiants_1nf t ON e.id = t.etudiant_id  
WHERE e.nom = 'Alice Martin';  
```

### 🔍 Vérification 1NF

```sql
-- Test : Tous les étudiants parlant anglais
SELECT DISTINCT e.nom  
FROM etudiants_1nf e  
JOIN langues_etudiants_1nf l ON e.id = l.etudiant_id  
WHERE l.langue = 'Anglais';  

-- Test : Statistiques des langues
SELECT
    langue,
    COUNT(*) as nb_etudiants,
    COUNT(CASE WHEN niveau = 'courant' THEN 1 END) as niveau_courant
FROM langues_etudiants_1nf  
GROUP BY langue  
ORDER BY nb_etudiants DESC;  
```

## Deuxième Forme Normale (2NF) - Élimination des dépendances partielles

### 🎯 Règle 2NF : Dépendance fonctionnelle complète

**Principe** : Être en 1NF ET chaque attribut non-clé doit dépendre de **toute la clé primaire**, pas seulement d'une partie.

⚠️ **Ne concerne que les clés composées** (plusieurs colonnes dans la clé primaire)

### ❌ Violation de 2NF : Dépendances partielles

```sql
-- Table violant la 2NF
CREATE TABLE inscriptions_notes_mauvais_2nf (
    etudiant_id INTEGER,
    cours_id INTEGER,

    -- Dépendent de toute la clé (etudiant_id + cours_id)
    note REAL,
    date_examen TEXT,

    -- ❌ Dépendent seulement de etudiant_id (dépendance partielle)
    nom_etudiant TEXT,
    email_etudiant TEXT,
    classe_etudiant TEXT,

    -- ❌ Dépendent seulement de cours_id (dépendance partielle)
    nom_cours TEXT,
    credits_cours INTEGER,
    nom_professeur TEXT,

    PRIMARY KEY (etudiant_id, cours_id)
);

INSERT INTO inscriptions_notes_mauvais_2nf VALUES
    (1, 101, 15.5, '2024-06-15', 'Alice Martin', 'alice@email.com', 'L3 Info',
     'Base de données', 6, 'Dr. Dupont'),
    (1, 102, 17.0, '2024-06-20', 'Alice Martin', 'alice@email.com', 'L3 Info',
     'Programmation', 8, 'Dr. Martin'),
    (2, 101, 12.5, '2024-06-15', 'Bob Leroy', 'bob@email.com', 'L2 Info',
     'Base de données', 6, 'Dr. Dupont');

-- Problèmes de la violation 2NF
SELECT * FROM inscriptions_notes_mauvais_2nf;
```

**Problèmes identifiés :**
- **Redondance** : Infos d'Alice répétées pour chaque cours
- **Anomalies de mise à jour** : Changer l'email d'Alice = modifier plusieurs lignes
- **Incohérences** : Risque d'avoir des informations différentes pour le même étudiant

### ✅ Solution 2NF : Séparation par dépendances

```sql
-- Séparer selon les dépendances fonctionnelles

-- Table 1 : Informations des étudiants (dépendent de etudiant_id)
CREATE TABLE etudiants_2nf (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    email TEXT UNIQUE,
    classe TEXT
);

-- Table 2 : Informations des cours (dépendent de cours_id)
CREATE TABLE cours_2nf (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    code TEXT UNIQUE,
    credits INTEGER,
    professeur TEXT
);

-- Table 3 : Inscriptions et notes (dépendent de etudiant_id + cours_id)
CREATE TABLE inscriptions_2nf (
    etudiant_id INTEGER,
    cours_id INTEGER,
    note REAL,
    date_examen TEXT,
    PRIMARY KEY (etudiant_id, cours_id),
    FOREIGN KEY (etudiant_id) REFERENCES etudiants_2nf(id),
    FOREIGN KEY (cours_id) REFERENCES cours_2nf(id)
);

-- Insertion des données normalisées
INSERT INTO etudiants_2nf (nom, email, classe) VALUES
    ('Alice Martin', 'alice@email.com', 'L3 Info'),
    ('Bob Leroy', 'bob@email.com', 'L2 Info');

INSERT INTO cours_2nf (nom, code, credits, professeur) VALUES
    ('Base de données', 'BDD101', 6, 'Dr. Dupont'),
    ('Programmation', 'PROG201', 8, 'Dr. Martin');

INSERT INTO inscriptions_2nf VALUES
    (1, 1, 15.5, '2024-06-15'),
    (1, 2, 17.0, '2024-06-20'),
    (2, 1, 12.5, '2024-06-15');
```

### 🔍 Vérification 2NF

```sql
-- Maintenant les mises à jour sont simples et sûres
UPDATE etudiants_2nf SET email = 'alice.martin@nouveau.com' WHERE id = 1;

-- Une seule ligne modifiée, cohérence garantie !

-- Requête complète avec jointures
SELECT
    e.nom as etudiant,
    c.nom as cours,
    c.credits,
    i.note,
    i.date_examen
FROM inscriptions_2nf i  
JOIN etudiants_2nf e ON i.etudiant_id = e.id  
JOIN cours_2nf c ON i.cours_id = c.id  
ORDER BY e.nom, c.nom;  
```

## Troisième Forme Normale (3NF) - Élimination des dépendances transitives

### 🎯 Règle 3NF : Pas de dépendance transitive

**Principe** : Être en 2NF ET aucun attribut non-clé ne doit dépendre d'un autre attribut non-clé.

**Dépendance transitive** : A → B → C (donc A → C indirectement)

### ❌ Violation de 3NF : Dépendances transitives

```sql
-- Table violant la 3NF
CREATE TABLE cours_mauvais_3nf (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    code TEXT,
    credits INTEGER,

    -- Dépendance directe de l'id du cours
    professeur_nom TEXT,

    -- ❌ Dépendance transitive : id → professeur_nom → departement
    professeur_departement TEXT,
    professeur_email TEXT,
    professeur_bureau TEXT
);

INSERT INTO cours_mauvais_3nf VALUES
    (1, 'Base de données', 'BDD101', 6, 'Dr. Dupont', 'Informatique', 'dupont@univ.fr', 'B204'),
    (2, 'Programmation', 'PROG201', 8, 'Dr. Martin', 'Informatique', 'martin@univ.fr', 'B205'),
    (3, 'Réseaux', 'RES301', 5, 'Dr. Dupont', 'Informatique', 'dupont@univ.fr', 'B204'),
    (4, 'Mathématiques', 'MATH101', 6, 'Dr. Bernard', 'Mathématiques', 'bernard@univ.fr', 'A102');

-- Problèmes de la violation 3NF
SELECT * FROM cours_mauvais_3nf;
```

**Problèmes identifiés :**
- **Redondance** : Dr. Dupont répété avec ses infos dans plusieurs cours
- **Anomalies de mise à jour** : Changer le bureau du Dr. Dupont = modifier plusieurs lignes
- **Incohérences** : Risque d'avoir des infos différentes pour le même professeur

### ✅ Solution 3NF : Séparation des entités indépendantes

```sql
-- Séparer les professeurs des cours

-- Table 1 : Professeurs (entité indépendante)
CREATE TABLE professeurs_3nf (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    email TEXT UNIQUE,
    departement TEXT,
    bureau TEXT
);

-- Table 2 : Cours (référence aux professeurs)
CREATE TABLE cours_3nf (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    code TEXT UNIQUE,
    credits INTEGER,
    professeur_id INTEGER,
    FOREIGN KEY (professeur_id) REFERENCES professeurs_3nf(id)
);

-- Insertion des données normalisées
INSERT INTO professeurs_3nf (nom, email, departement, bureau) VALUES
    ('Dr. Dupont', 'dupont@univ.fr', 'Informatique', 'B204'),
    ('Dr. Martin', 'martin@univ.fr', 'Informatique', 'B205'),
    ('Dr. Bernard', 'bernard@univ.fr', 'Mathématiques', 'A102');

INSERT INTO cours_3nf (nom, code, credits, professeur_id) VALUES
    ('Base de données', 'BDD101', 6, 1),
    ('Programmation', 'PROG201', 8, 2),
    ('Réseaux', 'RES301', 5, 1),
    ('Mathématiques', 'MATH101', 6, 3);
```

### 🔍 Vérification 3NF

```sql
-- Mise à jour simple et cohérente
UPDATE professeurs_3nf SET bureau = 'B210' WHERE nom = 'Dr. Dupont';

-- Tous les cours du Dr. Dupont reflètent automatiquement le changement !

-- Requêtes riches possibles
SELECT
    c.nom as cours,
    c.code,
    p.nom as professeur,
    p.departement,
    p.bureau
FROM cours_3nf c  
JOIN professeurs_3nf p ON c.professeur_id = p.id  
ORDER BY p.departement, c.nom;  

-- Statistiques par département
SELECT
    p.departement,
    COUNT(c.id) as nb_cours,
    AVG(c.credits) as credits_moyen
FROM professeurs_3nf p  
LEFT JOIN cours_3nf c ON p.id = c.professeur_id  
GROUP BY p.departement;  
```

## Exemple complet : De 0NF à 3NF

### 🎯 Transformation complète d'une base d'école

Voyons la transformation complète depuis une table dénormalisée vers un schéma 3NF :

```sql
-- === POINT DE DÉPART : Table complètement dénormalisée ===
CREATE TABLE ecole_denormalisee (
    id INTEGER PRIMARY KEY,

    -- Étudiant
    nom_etudiant TEXT,
    email_etudiant TEXT,
    telephones_etudiant TEXT,  -- ❌ Valeurs multiples

    -- Cours
    nom_cours TEXT,
    code_cours TEXT,
    credits_cours INTEGER,

    -- Professeur
    nom_professeur TEXT,
    email_professeur TEXT,
    departement TEXT,        -- ❌ Dépendance transitive
    bureau TEXT,             -- ❌ Dépendance transitive

    -- Note
    note REAL,
    date_examen TEXT
);
```

### 🔄 Transformation étape par étape

```sql
-- === ÉTAPE 1 : Passage en 1NF ===
-- Éliminer les valeurs multiples

-- Étudiants avec téléphones séparés
CREATE TABLE etudiants_temp (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    email TEXT UNIQUE
);

CREATE TABLE telephones_temp (
    etudiant_id INTEGER,
    telephone TEXT,
    PRIMARY KEY (etudiant_id, telephone),
    FOREIGN KEY (etudiant_id) REFERENCES etudiants_temp(id)
);

-- Table principale temporaire en 1NF
CREATE TABLE ecole_1nf (
    etudiant_id INTEGER,
    nom_etudiant TEXT,
    email_etudiant TEXT,
    nom_cours TEXT,
    code_cours TEXT,
    credits_cours INTEGER,
    nom_professeur TEXT,
    email_professeur TEXT,
    departement TEXT,
    bureau TEXT,
    note REAL,
    date_examen TEXT
);

-- === ÉTAPE 2 : Passage en 2NF ===
-- Créer une clé composée et éliminer les dépendances partielles

CREATE TABLE inscriptions_temp (
    etudiant_id INTEGER,
    cours_id INTEGER,
    note REAL,
    date_examen TEXT,
    PRIMARY KEY (etudiant_id, cours_id)
);

-- === ÉTAPE 3 : Passage en 3NF ===
-- Éliminer les dépendances transitives (professeur → département)
```

### ✅ Schéma final normalisé 3NF

```sql
-- === SCHÉMA FINAL EN 3NF ===

-- Départements (nouvelle entité identifiée)
CREATE TABLE departements (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL UNIQUE,
    batiment TEXT,
    responsable TEXT
);

-- Professeurs (sans dépendance transitive)
CREATE TABLE professeurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    email TEXT UNIQUE,
    bureau TEXT,
    departement_id INTEGER,
    FOREIGN KEY (departement_id) REFERENCES departements(id)
);

-- Étudiants
CREATE TABLE etudiants (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    email TEXT UNIQUE,
    numero_etudiant TEXT UNIQUE
);

-- Téléphones des étudiants
CREATE TABLE telephones_etudiants (
    etudiant_id INTEGER,
    telephone TEXT,
    type TEXT DEFAULT 'mobile',
    PRIMARY KEY (etudiant_id, telephone),
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id) ON DELETE CASCADE
);

-- Cours
CREATE TABLE cours (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    code TEXT UNIQUE,
    credits INTEGER CHECK (credits > 0),
    professeur_id INTEGER,
    FOREIGN KEY (professeur_id) REFERENCES professeurs(id)
);

-- Inscriptions et notes
CREATE TABLE inscriptions (
    etudiant_id INTEGER,
    cours_id INTEGER,
    note REAL CHECK (note BETWEEN 0 AND 20),
    date_examen TEXT,
    semestre TEXT,
    PRIMARY KEY (etudiant_id, cours_id),
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id) ON DELETE CASCADE,
    FOREIGN KEY (cours_id) REFERENCES cours(id) ON DELETE CASCADE
);

-- === DONNÉES D'EXEMPLE ===

INSERT INTO departements (nom, batiment, responsable) VALUES
    ('Informatique', 'Bâtiment B', 'Prof. Directeur'),
    ('Mathématiques', 'Bâtiment A', 'Prof. Responsable');

INSERT INTO professeurs (nom, email, bureau, departement_id) VALUES
    ('Dr. Dupont', 'dupont@univ.fr', 'B204', 1),
    ('Dr. Martin', 'martin@univ.fr', 'B205', 1),
    ('Dr. Bernard', 'bernard@univ.fr', 'A102', 2);

INSERT INTO etudiants (nom, email, numero_etudiant) VALUES
    ('Alice Martin', 'alice@etu.fr', '20240001'),
    ('Bob Leroy', 'bob@etu.fr', '20240002'),
    ('Claire Dubois', 'claire@etu.fr', '20240003');

INSERT INTO telephones_etudiants VALUES
    (1, '0123456789', 'mobile'),
    (1, '0987654321', 'fixe'),
    (2, '0555666777', 'mobile');

INSERT INTO cours (nom, code, credits, professeur_id) VALUES
    ('Base de données', 'BDD101', 6, 1),
    ('Programmation', 'PROG201', 8, 2),
    ('Réseaux', 'RES301', 5, 1),
    ('Mathématiques', 'MATH101', 6, 3);

INSERT INTO inscriptions (etudiant_id, cours_id, note, date_examen, semestre) VALUES
    (1, 1, 15.5, '2024-06-15', 'S1'),
    (1, 2, 17.0, '2024-06-20', 'S1'),
    (2, 1, 12.5, '2024-06-15', 'S1'),
    (2, 3, 14.0, '2024-06-18', 'S1'),
    (3, 4, 18.5, '2024-06-22', 'S1');
```

## Avantages et bénéfices de la normalisation

### 🎯 Démonstration des avantages

```sql
-- === AVANTAGES EN ACTION ===

-- 1. Cohérence garantie
UPDATE professeurs SET email = 'dupont.nouveau@univ.fr' WHERE id = 1;
-- Un seul UPDATE pour tous les cours du professeur !

-- 2. Pas de redondance
SELECT
    COUNT(*) as lignes_total,
    COUNT(DISTINCT p.nom) as professeurs_uniques,
    COUNT(DISTINCT e.nom) as etudiants_uniques
FROM inscriptions i  
JOIN cours c ON i.cours_id = c.id  
JOIN professeurs p ON c.professeur_id = p.id  
JOIN etudiants e ON i.etudiant_id = e.id;  

-- 3. Requêtes riches possibles
SELECT
    d.nom as departement,
    p.nom as professeur,
    COUNT(DISTINCT c.id) as nb_cours,
    COUNT(DISTINCT i.etudiant_id) as nb_etudiants,
    AVG(i.note) as moyenne_notes
FROM departements d  
JOIN professeurs p ON d.id = p.departement_id  
JOIN cours c ON p.id = c.professeur_id  
JOIN inscriptions i ON c.id = i.cours_id  
WHERE i.note IS NOT NULL  
GROUP BY d.id, p.id  
ORDER BY moyenne_notes DESC;  

-- 4. Intégrité référentielle
-- Impossible de supprimer un professeur qui a des cours
-- Impossible d'inscrire un étudiant à un cours inexistant

-- 5. Maintenance facilitée
-- Ajouter un nouveau téléphone à Alice
INSERT INTO telephones_etudiants VALUES (1, '0611223344', 'professionnel');

-- Changer le responsable d'un département
UPDATE departements SET responsable = 'Nouveau Responsable' WHERE nom = 'Informatique';
```

### 📊 Comparaison avant/après normalisation

```sql
-- === MÉTRIQUES DE QUALITÉ ===

-- Avant normalisation (estimation pour notre exemple)
SELECT 'Avant normalisation' as etat,
       5 as nb_tables,
       25 as lignes_donnees,
       15 as redondances_estimees,
       8 as points_mise_a_jour_risque;

-- Après normalisation
SELECT 'Après normalisation' as etat,
       6 as nb_tables,
       17 as lignes_donnees,
       0 as redondances,
       0 as points_mise_a_jour_risque;

-- Statistiques réelles du schéma normalisé
SELECT
    'Professeurs' as table_name, COUNT(*) as nb_lignes FROM professeurs
UNION ALL SELECT 'Étudiants', COUNT(*) FROM etudiants  
UNION ALL SELECT 'Cours', COUNT(*) FROM cours  
UNION ALL SELECT 'Inscriptions', COUNT(*) FROM inscriptions  
UNION ALL SELECT 'Départements', COUNT(*) FROM departements  
UNION ALL SELECT 'Téléphones', COUNT(*) FROM telephones_etudiants;  
```

## Cas particuliers et exceptions

### 🎯 Quand ne PAS normaliser ?

**Dénormalisation contrôlée** peut être justifiée pour :

```sql
-- Exemple : Table de reporting dénormalisée pour performance
CREATE TABLE rapport_notes_denormalise (
    id INTEGER PRIMARY KEY,
    etudiant_nom TEXT,
    cours_nom TEXT,
    professeur_nom TEXT,
    departement_nom TEXT,
    note REAL,
    date_examen TEXT,

    -- Colonnes calculées dénormalisées
    moyenne_etudiant REAL,
    moyenne_cours REAL,
    rang_dans_cours INTEGER,

    -- Mise à jour via trigger ou batch
    derniere_maj TEXT DEFAULT (datetime('now'))
);

-- Trigger pour maintenir la cohérence
-- (suppose que `etudiants`, `cours`, `professeurs`, `departements`, `inscriptions`
--  existent — cf. section « Schéma final normalisé 3NF » plus haut)
CREATE TRIGGER maj_rapport_notes  
AFTER INSERT ON inscriptions  
BEGIN  
    INSERT OR REPLACE INTO rapport_notes_denormalise
        (id, etudiant_nom, cours_nom, professeur_nom, departement_nom, note, date_examen)
    SELECT
        NEW.rowid,
        e.nom,
        c.nom,
        p.nom,
        d.nom,
        NEW.note,
        NEW.date_examen
    FROM etudiants e, cours c, professeurs p, departements d
    WHERE e.id = NEW.etudiant_id
      AND c.id = NEW.cours_id
      AND p.id = c.professeur_id
      AND d.id = p.departement_id;
END;
```

**Justifications pour la dénormalisation :**
- **Performance** : Éviter des jointures complexes répétées
- **Reporting** : Tables précalculées pour les tableaux de bord
- **Historique** : Snapshot des données à un moment donné
- **Cache** : Résultats de calculs coûteux

### ⚠️ Règles pour la dénormalisation contrôlée

```sql
-- 1. Garder la source normalisée
-- 2. Automatiser la synchronisation
-- 3. Documenter la justification
-- 4. Monitorer la cohérence

-- Exemple de vérification de cohérence
CREATE VIEW verification_coherence AS  
SELECT  
    'Incohérences détectées' as alerte,
    COUNT(*) as nb_problemes
FROM rapport_notes_denormalise r  
JOIN inscriptions i ON r.etudiant_nom || r.cours_nom =  
    (SELECT e.nom || c.nom FROM etudiants e, cours c
     WHERE e.id = i.etudiant_id AND c.id = i.cours_id)
WHERE r.note != i.note;
```

## Récapitulatif et bonnes pratiques

### ✅ Processus de normalisation

**Étapes systématiques :**
1. **Identifier les entités** et leurs attributs
2. **Appliquer 1NF** : Atomicité des valeurs
3. **Appliquer 2NF** : Éliminer dépendances partielles
4. **Appliquer 3NF** : Éliminer dépendances transitives
5. **Valider** : Tester l'intégrité et les performances
6. **Documenter** : Justifier les choix de conception

### 🎯 Check-list de normalisation

#### ✅ Vérification 1NF
- [ ] Chaque cellule contient une valeur atomique
- [ ] Pas de listes ou valeurs séparées par des délimiteurs
- [ ] Pas de colonnes répétitives (col1, col2, col3...)
- [ ] Chaque ligne est unique (clé primaire définie)

#### ✅ Vérification 2NF
- [ ] La table est en 1NF
- [ ] Identifier toutes les clés composées
- [ ] Vérifier que chaque attribut dépend de TOUTE la clé
- [ ] Séparer les attributs avec dépendances partielles

#### ✅ Vérification 3NF
- [ ] La table est en 2NF
- [ ] Identifier les dépendances transitives (A → B → C)
- [ ] Créer des tables séparées pour les entités indépendantes
- [ ] Utiliser des clés étrangères pour maintenir les liens

### 📋 Bonnes pratiques SQLite spécifiques

> 💡 Les exemples ci-dessous sont **illustratifs** et utilisent le suffixe `_bp` pour ne pas entrer en conflit avec les tables `etudiants`, `cours`, `inscriptions` du schéma final 3NF déjà créées plus haut.

```sql
-- ✅ Nomenclature cohérente
CREATE TABLE etudiants_bp (              -- Pluriel pour les tables
    id INTEGER PRIMARY KEY,              -- id simple pour PK
    nom TEXT NOT NULL,
    email TEXT UNIQUE,
    departement_id INTEGER,              -- _id pour les FK
    FOREIGN KEY (departement_id) REFERENCES departements(id)
);

-- ✅ Contraintes appropriées
CREATE TABLE cours_bp (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL CHECK (LENGTH(nom) > 0),
    code TEXT UNIQUE CHECK (LENGTH(code) BETWEEN 3 AND 10),
    credits INTEGER CHECK (credits BETWEEN 1 AND 12),
    professeur_id INTEGER NOT NULL,
    FOREIGN KEY (professeur_id) REFERENCES professeurs(id)
);

-- ✅ Index sur les clés étrangères (sur les tables _bp)
CREATE INDEX idx_cours_bp_professeur ON cours_bp(professeur_id);
-- Pour les vrais index sur le schéma 3NF :
-- CREATE INDEX idx_inscriptions_etudiant ON inscriptions(etudiant_id);
-- CREATE INDEX idx_inscriptions_cours ON inscriptions(cours_id);
```

### 🔧 Outils d'analyse et validation

```sql
-- === REQUÊTES DE VALIDATION DE LA NORMALISATION ===

-- 1. Détecter les violations potentielles de 1NF
SELECT 'Vérification 1NF' as test;

-- Chercher des valeurs avec séparateurs suspects
SELECT 'Valeurs avec séparateurs' as probleme, COUNT(*) as occurrences  
FROM etudiants  
WHERE nom LIKE '%,%' OR nom LIKE '%;%' OR nom LIKE '%|%';  

-- Chercher des emails multiples
SELECT 'Emails multiples possibles' as probleme, COUNT(*) as occurrences  
FROM etudiants  
WHERE email LIKE '%,%' OR email LIKE '%;%';  

-- 2. Vérifier l'intégrité référentielle (2NF/3NF)
SELECT 'Vérification intégrité référentielle' as test;

-- Cours sans professeur valide
SELECT 'Cours sans professeur' as probleme, COUNT(*) as occurrences  
FROM cours c  
LEFT JOIN professeurs p ON c.professeur_id = p.id  
WHERE p.id IS NULL;  

-- Inscriptions avec étudiants inexistants
SELECT 'Inscriptions orphelines (étudiants)' as probleme, COUNT(*) as occurrences  
FROM inscriptions i  
LEFT JOIN etudiants e ON i.etudiant_id = e.id  
WHERE e.id IS NULL;  

-- Inscriptions avec cours inexistants
SELECT 'Inscriptions orphelines (cours)' as probleme, COUNT(*) as occurrences  
FROM inscriptions i  
LEFT JOIN cours c ON i.cours_id = c.id  
WHERE c.id IS NULL;  

-- 3. Analyser la redondance restante
SELECT 'Analyse de redondance' as test;

-- Vérifier l'unicité des entités principales
SELECT
    'Doublons potentiels étudiants' as probleme,
    COUNT(*) - COUNT(DISTINCT LOWER(nom) || LOWER(email)) as occurrences
FROM etudiants;

SELECT
    'Doublons potentiels cours' as probleme,
    COUNT(*) - COUNT(DISTINCT UPPER(code)) as occurrences
FROM cours;

-- 4. Statistiques de qualité
SELECT 'Métriques de qualité' as test;

SELECT
    'Tables normalisées' as metrique,
    (SELECT COUNT(*) FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%') as valeur;

SELECT
    'Contraintes FK' as metrique,
    COUNT(*) as valeur
FROM pragma_foreign_key_list('cours')  
UNION ALL SELECT 'Contraintes FK', COUNT(*) FROM pragma_foreign_key_list('inscriptions')  
UNION ALL SELECT 'Contraintes FK', COUNT(*) FROM pragma_foreign_key_list('telephones_etudiants');  
```

### 🎯 Patterns de normalisation avancés

#### Pattern 1 : Entités faibles

```sql
-- Entité faible : dépend d'une autre entité pour son existence
CREATE TABLE adresses_etudiants (
    etudiant_id INTEGER,
    type_adresse TEXT,  -- 'domicile', 'famille', 'temporaire'
    rue TEXT,
    ville TEXT,
    code_postal TEXT,
    pays TEXT DEFAULT 'France',
    PRIMARY KEY (etudiant_id, type_adresse),
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id) ON DELETE CASCADE
);

-- Si l'étudiant est supprimé, ses adresses le sont aussi
```

#### Pattern 2 : Attributs multi-valués normalisés

```sql
-- Au lieu de : competences TEXT (contenant 'Java;Python;SQL')
CREATE TABLE competences (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    domaine TEXT  -- 'programmation', 'base_de_donnees', 'web'
);

CREATE TABLE etudiants_competences (
    etudiant_id INTEGER,
    competence_id INTEGER,
    niveau TEXT CHECK (niveau IN ('débutant', 'intermédiaire', 'avancé', 'expert')),
    date_acquisition TEXT,
    certifie INTEGER DEFAULT 0,
    PRIMARY KEY (etudiant_id, competence_id),
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id),
    FOREIGN KEY (competence_id) REFERENCES competences(id)
);

-- Données d'exemple
INSERT INTO competences (nom, domaine) VALUES
    ('Python', 'programmation'),
    ('SQL', 'base_de_donnees'),
    ('JavaScript', 'programmation'),
    ('SQLite', 'base_de_donnees');

INSERT INTO etudiants_competences VALUES
    (1, 1, 'avancé', '2023-09-15', 1),
    (1, 2, 'intermédiaire', '2024-01-10', 0),
    (2, 3, 'débutant', '2024-02-01', 0);
```

#### Pattern 3 : Hiérarchies et auto-références

```sql
-- Hiérarchie de matières (matière → sous-matières)
CREATE TABLE matieres (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    code TEXT UNIQUE,
    description TEXT,
    matiere_parent_id INTEGER,  -- Auto-référence
    niveau INTEGER DEFAULT 1,   -- 1=principale, 2=sous-matière, etc.
    FOREIGN KEY (matiere_parent_id) REFERENCES matieres(id)
);

INSERT INTO matieres (nom, code, matiere_parent_id, niveau) VALUES
    ('Informatique', 'INFO', NULL, 1),
    ('Programmation', 'PROG', 1, 2),
    ('Base de données', 'BDD', 1, 2),
    ('Python', 'PY', 2, 3),
    ('Java', 'JAVA', 2, 3),
    ('SQL', 'SQL', 3, 3),
    ('NoSQL', 'NOSQL', 3, 3);

-- Requête récursive pour afficher la hiérarchie
WITH RECURSIVE hierarchie_matieres(id, nom, niveau, chemin) AS (
    -- Niveau racine
    SELECT id, nom, niveau, nom as chemin
    FROM matieres
    WHERE matiere_parent_id IS NULL

    UNION ALL

    -- Niveaux suivants
    SELECT m.id, m.nom, m.niveau, h.chemin || ' > ' || m.nom
    FROM matieres m
    JOIN hierarchie_matieres h ON m.matiere_parent_id = h.id
)
SELECT niveau, chemin, nom FROM hierarchie_matieres ORDER BY chemin;
```

## Exercice pratique complet - Normalisation d'un système de bibliothèque

### 🎯 Objectif : Normaliser une base de bibliothèque

**Situation initiale : Table complètement dénormalisée**

```sql
-- === POINT DE DÉPART : CAUCHEMAR DE NORMALISATION ===
CREATE TABLE bibliotheque_denormalisee (
    id INTEGER PRIMARY KEY,

    -- Livre
    titre_livre TEXT,
    auteurs_livre TEXT,        -- ❌ "Tolkien;Lewis;Rowling"
    isbn TEXT,
    annee_publication INTEGER,
    genres TEXT,               -- ❌ "Fantasy;Aventure"

    -- Éditeur
    nom_editeur TEXT,
    ville_editeur TEXT,
    pays_editeur TEXT,

    -- Exemplaire
    numero_exemplaire TEXT,
    etat TEXT,                 -- 'neuf', 'bon', 'usé', 'endommagé'
    localisation TEXT,         -- 'A1-2-3' (allée-étagère-niveau)

    -- Emprunt
    emprunteur_nom TEXT,
    emprunteur_email TEXT,
    emprunteur_telephone TEXT,
    date_emprunt TEXT,
    date_retour_prevue TEXT,
    date_retour_reel TEXT,

    -- Bibliothécaire
    bibliothecaire_nom TEXT,
    bibliothecaire_service TEXT  -- ❌ Dépendance transitive
);

-- Données d'exemple problématiques
INSERT INTO bibliotheque_denormalisee VALUES
    (1, 'Le Seigneur des Anneaux', 'J.R.R. Tolkien', '978-0261103573', 1954,
     'Fantasy;Aventure;Épique', 'Flammarion', 'Paris', 'France',
     'EX001', 'bon', 'A1-2-3', 'Jean Dupont', 'jean@email.com', '0123456789',
     '2024-01-15', '2024-02-15', NULL, 'Marie Martin', 'Accueil'),
    (2, 'Harry Potter tome 1', 'J.K. Rowling', '978-2070584628', 1997,
     'Fantasy;Jeunesse', 'Gallimard', 'Paris', 'France',
     'EX002', 'neuf', 'A2-1-4', 'Sophie Leroy', 'sophie@email.com', '0987654321',
     '2024-01-20', '2024-02-20', '2024-02-18', 'Marie Martin', 'Accueil');
```

### 🔄 Solution étape par étape

```sql
-- === ÉTAPE 1 : ANALYSE ET IDENTIFICATION DES ENTITÉS ===

-- Entités identifiées :
-- 1. Livres (titre, ISBN, année)
-- 2. Auteurs (nom, informations)
-- 3. Éditeurs (nom, ville, pays)
-- 4. Genres (nom)
-- 5. Exemplaires (numéro, état, localisation)
-- 6. Emprunteurs (nom, email, téléphone)
-- 7. Bibliothécaires (nom, service)
-- 8. Emprunts (dates, associations)

-- Relations identifiées :
-- - Livre ↔ Auteur (many-to-many)
-- - Livre ↔ Genre (many-to-many)
-- - Livre → Éditeur (many-to-one)
-- - Exemplaire → Livre (many-to-one)
-- - Emprunt → Exemplaire (one-to-one ou one-to-many selon règles)
-- - Emprunt → Emprunteur (many-to-one)
-- - Emprunt → Bibliothécaire (many-to-one)

-- === ÉTAPE 2 : SCHÉMA NORMALISÉ 3NF ===

-- Services des bibliothécaires (entité séparée pour 3NF)
CREATE TABLE services (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    responsable TEXT,
    localisation TEXT
);

-- Bibliothécaires
CREATE TABLE bibliothecaires (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    email TEXT UNIQUE,
    service_id INTEGER,
    FOREIGN KEY (service_id) REFERENCES services(id)
);

-- Éditeurs
CREATE TABLE editeurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    ville TEXT,
    pays TEXT DEFAULT 'France'
);

-- Auteurs
CREATE TABLE auteurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    date_naissance TEXT,
    nationalite TEXT
);

-- Genres
CREATE TABLE genres (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT UNIQUE NOT NULL,
    description TEXT
);

-- Livres
CREATE TABLE livres (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    isbn TEXT UNIQUE,
    annee_publication INTEGER,
    nombre_pages INTEGER,
    editeur_id INTEGER,
    FOREIGN KEY (editeur_id) REFERENCES editeurs(id)
);

-- Relations many-to-many : Livres ↔ Auteurs
CREATE TABLE livres_auteurs (
    livre_id INTEGER,
    auteur_id INTEGER,
    role TEXT DEFAULT 'auteur',  -- 'auteur', 'co-auteur', 'traducteur'
    PRIMARY KEY (livre_id, auteur_id),
    FOREIGN KEY (livre_id) REFERENCES livres(id) ON DELETE CASCADE,
    FOREIGN KEY (auteur_id) REFERENCES auteurs(id)
);

-- Relations many-to-many : Livres ↔ Genres
CREATE TABLE livres_genres (
    livre_id INTEGER,
    genre_id INTEGER,
    PRIMARY KEY (livre_id, genre_id),
    FOREIGN KEY (livre_id) REFERENCES livres(id) ON DELETE CASCADE,
    FOREIGN KEY (genre_id) REFERENCES genres(id)
);

-- Exemplaires (instances physiques des livres)
CREATE TABLE exemplaires (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    numero TEXT UNIQUE NOT NULL,
    livre_id INTEGER NOT NULL,
    etat TEXT CHECK (etat IN ('neuf', 'bon', 'usé', 'endommagé', 'perdu')),
    localisation TEXT,  -- 'A1-2-3'
    date_acquisition TEXT DEFAULT (date('now')),
    FOREIGN KEY (livre_id) REFERENCES livres(id)
);

-- Emprunteurs
CREATE TABLE emprunteurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    email TEXT UNIQUE,
    telephone TEXT,
    adresse TEXT,
    date_inscription TEXT DEFAULT (date('now')),
    actif INTEGER DEFAULT 1
);

-- Emprunts
CREATE TABLE emprunts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    exemplaire_id INTEGER NOT NULL,
    emprunteur_id INTEGER NOT NULL,
    bibliothecaire_id INTEGER,
    date_emprunt TEXT DEFAULT (date('now')),
    date_retour_prevue TEXT NOT NULL,
    date_retour_reel TEXT,
    commentaires TEXT,
    FOREIGN KEY (exemplaire_id) REFERENCES exemplaires(id),
    FOREIGN KEY (emprunteur_id) REFERENCES emprunteurs(id),
    FOREIGN KEY (bibliothecaire_id) REFERENCES bibliothecaires(id)
);

-- === ÉTAPE 3 : INSERTION DES DONNÉES NORMALISÉES ===

-- Services
INSERT INTO services (nom, responsable, localisation) VALUES
    ('Accueil', 'Chef de service', 'Entrée principale'),
    ('Jeunesse', 'Responsable jeunesse', 'Aile droite'),
    ('Recherche', 'Documentaliste', 'Étage 2');

-- Bibliothécaires
INSERT INTO bibliothecaires (nom, email, service_id) VALUES
    ('Marie Martin', 'marie.martin@biblio.fr', 1),
    ('Pierre Dubois', 'pierre.dubois@biblio.fr', 2),
    ('Claire Leroy', 'claire.leroy@biblio.fr', 3);

-- Éditeurs
INSERT INTO editeurs (nom, ville, pays) VALUES
    ('Flammarion', 'Paris', 'France'),
    ('Gallimard', 'Paris', 'France'),
    ('Le Seuil', 'Paris', 'France');

-- Auteurs
INSERT INTO auteurs (nom, prenom, nationalite) VALUES
    ('Tolkien', 'J.R.R.', 'Britannique'),
    ('Rowling', 'J.K.', 'Britannique'),
    ('Hugo', 'Victor', 'Française');

-- Genres
INSERT INTO genres (nom, description) VALUES
    ('Fantasy', 'Littérature fantastique'),
    ('Aventure', 'Romans d''aventure'),
    ('Jeunesse', 'Littérature pour jeunes'),
    ('Classique', 'Grands classiques de la littérature');

-- Livres
INSERT INTO livres (titre, isbn, annee_publication, editeur_id) VALUES
    ('Le Seigneur des Anneaux', '978-0261103573', 1954, 1),
    ('Harry Potter à l''école des sorciers', '978-2070584628', 1997, 2),
    ('Les Misérables', '978-2070409228', 1862, 2);

-- Relations livres-auteurs
INSERT INTO livres_auteurs (livre_id, auteur_id) VALUES
    (1, 1),  -- Tolkien → LOTR
    (2, 2),  -- Rowling → HP
    (3, 3);  -- Hugo → Misérables

-- Relations livres-genres
INSERT INTO livres_genres (livre_id, genre_id) VALUES
    (1, 1), (1, 2),  -- LOTR : Fantasy + Aventure
    (2, 1), (2, 3),  -- HP : Fantasy + Jeunesse
    (3, 4);          -- Misérables : Classique

-- Exemplaires
INSERT INTO exemplaires (numero, livre_id, etat, localisation) VALUES
    ('EX001', 1, 'bon', 'A1-2-3'),
    ('EX002', 2, 'neuf', 'A2-1-4'),
    ('EX003', 3, 'usé', 'B1-3-2');

-- Emprunteurs
INSERT INTO emprunteurs (nom, prenom, email, telephone) VALUES
    ('Dupont', 'Jean', 'jean@email.com', '0123456789'),
    ('Leroy', 'Sophie', 'sophie@email.com', '0987654321');

-- Emprunts
INSERT INTO emprunts (exemplaire_id, emprunteur_id, bibliothecaire_id, date_emprunt, date_retour_prevue, date_retour_reel) VALUES
    (1, 1, 1, '2024-01-15', '2024-02-15', NULL),  -- En cours
    (2, 2, 1, '2024-01-20', '2024-02-20', '2024-02-18');  -- Rendu
```

### 🔍 Validation du schéma normalisé

```sql
-- === TESTS DE VALIDATION ===

-- 1. Vérification 1NF : Pas de valeurs multiples
SELECT 'Vérification 1NF - Valeurs atomiques' as test;
-- Toutes les colonnes contiennent des valeurs atomiques ✅

-- 2. Vérification 2NF : Pas de dépendances partielles
SELECT 'Vérification 2NF - Dépendances fonctionnelles complètes' as test;
-- Tables avec clés composées : livres_auteurs, livres_genres
-- Pas d'attributs dépendant partiellement des clés ✅

-- 3. Vérification 3NF : Pas de dépendances transitives
SELECT 'Vérification 3NF - Pas de dépendances transitives' as test;
-- Services séparés des bibliothécaires ✅

-- 4. Tests fonctionnels
-- Livres avec leurs auteurs et genres
SELECT
    l.titre,
    GROUP_CONCAT(DISTINCT a.nom || ' ' || a.prenom) as auteurs,
    GROUP_CONCAT(DISTINCT g.nom) as genres,
    e.nom as editeur
FROM livres l  
LEFT JOIN livres_auteurs la ON l.id = la.livre_id  
LEFT JOIN auteurs a ON la.auteur_id = a.id  
LEFT JOIN livres_genres lg ON l.id = lg.livre_id  
LEFT JOIN genres g ON lg.genre_id = g.id  
LEFT JOIN editeurs e ON l.editeur_id = e.id  
GROUP BY l.id, l.titre, e.nom;  

-- Emprunts en cours avec toutes les informations
SELECT
    l.titre,
    ex.numero,
    emp.nom || ' ' || emp.prenom as emprunteur,
    empr.date_emprunt,
    empr.date_retour_prevue,
    CASE
        WHEN date('now') > empr.date_retour_prevue THEN 'En retard'
        ELSE 'Dans les temps'
    END as statut
FROM emprunts empr  
JOIN exemplaires ex ON empr.exemplaire_id = ex.id  
JOIN livres l ON ex.livre_id = l.id  
JOIN emprunteurs emp ON empr.emprunteur_id = emp.id  
WHERE empr.date_retour_reel IS NULL;  

-- Statistiques par genre
SELECT
    g.nom as genre,
    COUNT(DISTINCT l.id) as nb_livres,
    COUNT(DISTINCT ex.id) as nb_exemplaires,
    COUNT(DISTINCT empr.id) as nb_emprunts_total
FROM genres g  
LEFT JOIN livres_genres lg ON g.id = lg.genre_id  
LEFT JOIN livres l ON lg.livre_id = l.id  
LEFT JOIN exemplaires ex ON l.id = ex.livre_id  
LEFT JOIN emprunts empr ON ex.id = empr.exemplaire_id  
GROUP BY g.id, g.nom  
ORDER BY nb_livres DESC;  
```

### 🎯 Avantages obtenus par la normalisation

```sql
-- === DÉMONSTRATION DES AVANTAGES ===

-- 1. Cohérence garantie
-- Changer l'email d'un emprunteur
UPDATE emprunteurs SET email = 'jean.nouveau@email.com' WHERE id = 1;
-- ✅ Un seul UPDATE, tous les emprunts restent cohérents

-- 2. Pas de redondance
-- Ajouter un nouvel exemplaire du Seigneur des Anneaux
INSERT INTO exemplaires (numero, livre_id, etat, localisation)  
VALUES ('EX004', 1, 'neuf', 'A1-2-4');  
-- ✅ Pas besoin de dupliquer les infos du livre

-- 3. Flexibilité
-- Ajouter un co-auteur à un livre
INSERT INTO livres_auteurs (livre_id, auteur_id, role)  
VALUES (1, 3, 'traducteur');  
-- ✅ Facilement extensible

-- 4. Intégrité référentielle
-- Tentative de suppression d'un livre avec exemplaires
-- DELETE FROM livres WHERE id = 1;
-- ✅ Protégé par les contraintes FK

-- 5. Requêtes riches
-- Top des auteurs les plus empruntés
SELECT
    a.nom || ' ' || a.prenom as auteur,
    COUNT(DISTINCT empr.id) as nb_emprunts,
    COUNT(DISTINCT l.id) as nb_livres_catalogue
FROM auteurs a  
JOIN livres_auteurs la ON a.id = la.auteur_id  
JOIN livres l ON la.livre_id = l.id  
JOIN exemplaires ex ON l.id = ex.livre_id  
JOIN emprunts empr ON ex.id = empr.exemplaire_id  
GROUP BY a.id, a.nom, a.prenom  
ORDER BY nb_emprunts DESC;  
```

## Conclusion et perspectives

### 🎉 Récapitulatif des acquis

**Vous maîtrisez maintenant :**
- ✅ **1NF** : Élimination des valeurs multiples et atomicité
- ✅ **2NF** : Suppression des dépendances partielles sur clés composées
- ✅ **3NF** : Élimination des dépendances transitives
- ✅ **Processus** : Méthodologie complète de normalisation
- ✅ **Validation** : Techniques de vérification et tests
- ✅ **Patterns** : Solutions courantes et bonnes pratiques SQLite

### 🚀 Bénéfices concrets

**Pour vos projets :**
- 🔒 **Intégrité** : Données cohérentes et fiables
- ⚡ **Maintenabilité** : Modifications simples et sûres
- 📈 **Évolutivité** : Structure extensible facilement
- 🎯 **Performance** : Requêtes optimisées et index efficaces
- 🛡️ **Robustesse** : Protection contre les anomalies

### 💡 Points clés à retenir

1. **Normalisation = organisation logique**, pas complication
2. **Chaque forme normale résout des problèmes spécifiques**
3. **3NF est généralement suffisant** pour la plupart des applications
4. **SQLite supporte parfaitement** la normalisation avancée
5. **Validation et tests** sont essentiels après normalisation

---

**💡 Dans le prochain chapitre**, nous explorerons les relations entre tables et les jointures complexes, en nous appuyant sur nos structures normalisées pour créer des liens robustes et performants.

**🎯 Vous savez maintenant** concevoir des bases de données SQLite propres, cohérentes et évolutives grâce à la normalisation !

⏭️ [3.2 Relations entre tables et jointures complexes](/03-conception-modelisation-avancee/02-relations-tables-jointures-complexes.md)
