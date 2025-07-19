🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.3 Gestion des clés étrangères et contraintes référentielles

## Introduction - Les gardiens de l'intégrité

Les clés étrangères sont comme les **liens de confiance** dans une communauté : elles garantissent que chaque référence pointe vers quelque chose qui existe réellement. Dans SQLite, leur gestion nécessite une attention particulière car elles ne sont pas activées par défaut !

> **Analogie simple** : Imaginez un système de bibliothèque où chaque fiche d'emprunt doit référencer un livre et un lecteur existants. Les clés étrangères empêchent qu'on crée une fiche d'emprunt pour un livre imaginaire ou un lecteur inexistant.

**Ce que nous allons maîtriser :**
- **Activation et configuration** des contraintes FK dans SQLite
- **Actions référentielles** (CASCADE, SET NULL, RESTRICT, SET DEFAULT)
- **Gestion des cycles** et dépendances complexes
- **Strategies de migration** et maintenance des contraintes
- **Debugging et résolution** des problèmes FK

## Spécificités SQLite - Comprendre les différences

### 🔧 Activation obligatoire des clés étrangères

**Point crucial :** SQLite désactive les clés étrangères par défaut pour des raisons de compatibilité !

```sql
-- ❌ État par défaut : FK désactivées
sqlite3 test_fk.db
PRAGMA foreign_keys;  -- Retourne 0 (désactivé)

-- Tentative de violation sans protection
CREATE TABLE parents (id INTEGER PRIMARY KEY, nom TEXT);
CREATE TABLE enfants (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    parent_id INTEGER,
    FOREIGN KEY (parent_id) REFERENCES parents(id)
);

INSERT INTO enfants VALUES (1, 'Alice', 999);  -- ✅ Accepté malgré parent inexistant !
SELECT * FROM enfants;  -- Alice avec parent_id = 999 (inexistant)

-- ✅ Activation correcte
PRAGMA foreign_keys = ON;
PRAGMA foreign_keys;  -- Retourne 1 (activé)

-- Maintenant la protection fonctionne
INSERT INTO enfants VALUES (2, 'Bob', 888);    -- ❌ Erreur : FK constraint failed

-- Nettoyage pour les exemples suivants
DROP TABLE enfants;
DROP TABLE parents;
```

### ⚠️ Activation par session

```sql
-- ⚠️ IMPORTANT : À faire à chaque connexion !
PRAGMA foreign_keys = ON;

-- Vérification systématique dans vos scripts
CREATE VIEW check_fk_status AS
SELECT
    CASE
        WHEN (SELECT foreign_keys FROM pragma_foreign_keys()) = 1
        THEN '✅ Clés étrangères ACTIVÉES'
        ELSE '❌ Clés étrangères DÉSACTIVÉES - DANGER !'
    END as statut;

SELECT * FROM check_fk_status;
```

## Configuration de base - Structure d'exemple

Créons un système de gestion d'école complet pour explorer toutes les facettes des FK :

```sql
-- === SYSTÈME D'ÉCOLE AVEC CONTRAINTES RÉFÉRENTIELLES ===
sqlite3 ecole_fk_complete.db
PRAGMA foreign_keys = ON;

-- 1. Entités de base (sans dépendances)
CREATE TABLE departements (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    code TEXT UNIQUE NOT NULL,
    budget REAL CHECK (budget >= 0),
    date_creation TEXT DEFAULT (date('now'))
);

CREATE TABLE niveaux_etude (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,  -- 'L1', 'L2', 'L3', 'M1', 'M2'
    description TEXT,
    duree_semestres INTEGER DEFAULT 2
);

-- 2. Entités avec dépendances simples
CREATE TABLE professeurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    email TEXT UNIQUE,
    telephone TEXT,
    bureau TEXT,
    date_embauche TEXT DEFAULT (date('now')),

    -- FK vers département
    departement_id INTEGER NOT NULL,
    FOREIGN KEY (departement_id) REFERENCES departements(id)
        ON DELETE RESTRICT  -- Pas de suppression si professeurs attachés
        ON UPDATE CASCADE   -- Mise à jour automatique de l'ID
);

CREATE TABLE etudiants (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    numero_etudiant TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE,
    date_naissance TEXT,
    date_inscription TEXT DEFAULT (date('now')),
    actif INTEGER DEFAULT 1,

    -- FK vers niveau d'étude
    niveau_id INTEGER NOT NULL,
    FOREIGN KEY (niveau_id) REFERENCES niveaux_etude(id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);

-- 3. Entités avec dépendances multiples
CREATE TABLE cours (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    code TEXT UNIQUE NOT NULL,
    description TEXT,
    credits INTEGER CHECK (credits BETWEEN 1 AND 12),
    heures_cours INTEGER CHECK (heures_cours > 0),

    -- FK multiples
    professeur_responsable_id INTEGER NOT NULL,
    departement_id INTEGER NOT NULL,
    niveau_requis_id INTEGER,

    -- Contraintes référentielles
    FOREIGN KEY (professeur_responsable_id) REFERENCES professeurs(id)
        ON DELETE RESTRICT,
    FOREIGN KEY (departement_id) REFERENCES departements(id)
        ON DELETE RESTRICT,
    FOREIGN KEY (niveau_requis_id) REFERENCES niveaux_etude(id)
        ON DELETE SET NULL  -- Si niveau supprimé, cours reste mais sans prérequis
);

-- 4. Tables de liaison avec FK multiples
CREATE TABLE inscriptions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    etudiant_id INTEGER NOT NULL,
    cours_id INTEGER NOT NULL,
    semestre TEXT NOT NULL,
    annee_universitaire TEXT NOT NULL,
    date_inscription TEXT DEFAULT (datetime('now')),
    statut TEXT DEFAULT 'inscrit' CHECK (statut IN ('inscrit', 'valide', 'abandonne')),
    note_finale REAL CHECK (note_finale BETWEEN 0 AND 20),

    -- Contrainte unique composite
    UNIQUE (etudiant_id, cours_id, semestre, annee_universitaire),

    -- FK avec actions différenciées
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id)
        ON DELETE CASCADE,  -- Si étudiant supprimé, supprimer ses inscriptions
    FOREIGN KEY (cours_id) REFERENCES cours(id)
        ON DELETE RESTRICT  -- Empêcher suppression cours avec inscriptions
);

-- === DONNÉES D'EXEMPLE ===

-- Départements
INSERT INTO departements (id, nom, code, budget) VALUES
    (1, 'Informatique', 'INFO', 150000),
    (2, 'Mathématiques', 'MATH', 120000),
    (3, 'Physique', 'PHYS', 140000);

-- Niveaux d'étude
INSERT INTO niveaux_etude (id, nom, description, duree_semestres) VALUES
    (1, 'L1', 'Première année de licence', 2),
    (2, 'L2', 'Deuxième année de licence', 2),
    (3, 'L3', 'Troisième année de licence', 2),
    (4, 'M1', 'Première année de master', 2),
    (5, 'M2', 'Deuxième année de master', 2);

-- Professeurs
INSERT INTO professeurs (nom, prenom, email, departement_id) VALUES
    ('Dupont', 'Alice', 'alice.dupont@univ.fr', 1),
    ('Martin', 'Bob', 'bob.martin@univ.fr', 1),
    ('Bernard', 'Claire', 'claire.bernard@univ.fr', 2),
    ('Leroy', 'David', 'david.leroy@univ.fr', 3);

-- Étudiants
INSERT INTO etudiants (nom, prenom, numero_etudiant, email, niveau_id) VALUES
    ('Moreau', 'Emma', '20240001', 'emma.moreau@etu.fr', 3),
    ('Dubois', 'Lucas', '20240002', 'lucas.dubois@etu.fr', 2),
    ('Petit', 'Camille', '20240003', 'camille.petit@etu.fr', 4),
    ('Roux', 'Thomas', '20240004', 'thomas.roux@etu.fr', 3);

-- Cours
INSERT INTO cours (nom, code, credits, professeur_responsable_id, departement_id, niveau_requis_id) VALUES
    ('Base de données', 'INFO301', 6, 1, 1, 3),
    ('Algèbre linéaire', 'MATH201', 5, 3, 2, 2),
    ('Mécanique quantique', 'PHYS401', 6, 4, 3, 4),
    ('Programmation Python', 'INFO201', 4, 2, 1, 2);

-- Inscriptions
INSERT INTO inscriptions (etudiant_id, cours_id, semestre, annee_universitaire, statut, note_finale) VALUES
    (1, 1, 'S1', '2024-2025', 'valide', 15.5),
    (1, 4, 'S1', '2024-2025', 'inscrit', NULL),
    (2, 2, 'S1', '2024-2025', 'valide', 14.0),
    (3, 3, 'S1', '2024-2025', 'inscrit', NULL),
    (4, 1, 'S1', '2024-2025', 'valide', 12.5);
```

## Actions référentielles - Contrôler les cascades

### 🎯 Comprendre les 4 actions principales

```sql
-- Démonstration des actions référentielles
CREATE TABLE demo_actions (
    id INTEGER PRIMARY KEY,
    action_type TEXT,
    description TEXT
);

INSERT INTO demo_actions VALUES
    (1, 'RESTRICT', 'Empêche la suppression/modification si références existent'),
    (2, 'CASCADE', 'Propage la suppression/modification aux références'),
    (3, 'SET NULL', 'Met NULL dans les FK lors de suppression/modification'),
    (4, 'SET DEFAULT', 'Met valeur par défaut dans les FK'),
    (5, 'NO ACTION', 'Vérifie contrainte à la fin de transaction (défaut)');
```

### 🔄 CASCADE - Propagation en cascade

```sql
-- Table avec CASCADE pour tester
CREATE TABLE tests_cascade_parent (
    id INTEGER PRIMARY KEY,
    nom TEXT
);

CREATE TABLE tests_cascade_enfant (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,
    donnee TEXT,
    FOREIGN KEY (parent_id) REFERENCES tests_cascade_parent(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

-- Données de test
INSERT INTO tests_cascade_parent VALUES (1, 'Parent A'), (2, 'Parent B');
INSERT INTO tests_cascade_enfant VALUES
    (1, 1, 'Enfant 1-1'),
    (2, 1, 'Enfant 1-2'),
    (3, 2, 'Enfant 2-1');

-- Test CASCADE DELETE
SELECT 'Avant suppression' as moment, * FROM tests_cascade_enfant;

DELETE FROM tests_cascade_parent WHERE id = 1;

SELECT 'Après suppression parent 1' as moment, * FROM tests_cascade_enfant;
-- Résultat : Enfants 1-1 et 1-2 automatiquement supprimés

-- Test CASCADE UPDATE
UPDATE tests_cascade_parent SET id = 20 WHERE id = 2;

SELECT 'Après modification parent 2→20' as moment,
       parent_id, donnee FROM tests_cascade_enfant;
-- Résultat : parent_id de l'enfant 2-1 automatiquement mis à jour vers 20

-- Nettoyage
DROP TABLE tests_cascade_enfant;
DROP TABLE tests_cascade_parent;
```

### 🚫 RESTRICT - Protection stricte

```sql
-- Notre système d'école utilise RESTRICT pour la protection
-- Test : Tentative de suppression d'un département avec professeurs
DELETE FROM departements WHERE id = 1;
-- ❌ Erreur : FOREIGN KEY constraint failed

-- Solution : Supprimer d'abord les dépendances
SELECT 'Professeurs dans département Informatique' as info,
       COUNT(*) as nombre
FROM professeurs WHERE departement_id = 1;

-- Pour supprimer, il faut d'abord gérer les professeurs
-- UPDATE professeurs SET departement_id = 2 WHERE departement_id = 1;
-- Puis : DELETE FROM departements WHERE id = 1;
```

### ❓ SET NULL - Mise à NULL conditionnelle

```sql
-- Exemple dans notre système : niveau_requis_id peut être NULL
DELETE FROM niveaux_etude WHERE id = 5;  -- Supprimer M2

-- Vérifier l'effet sur les cours
SELECT nom, code, niveau_requis_id
FROM cours
WHERE niveau_requis_id IS NULL OR niveau_requis_id = 5;

-- Réinsérer pour les tests suivants
INSERT INTO niveaux_etude (id, nom, description)
VALUES (5, 'M2', 'Deuxième année de master');
```

### 🔄 SET DEFAULT - Valeur de fallback

```sql
-- Exemple avec SET DEFAULT
CREATE TABLE categories_cours (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE,
    couleur TEXT DEFAULT '#FFFFFF'
);

CREATE TABLE cours_avec_default (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    categorie_id INTEGER DEFAULT 1,  -- Catégorie "Général" par défaut
    FOREIGN KEY (categorie_id) REFERENCES categories_cours(id)
        ON DELETE SET DEFAULT
);

INSERT INTO categories_cours VALUES (1, 'Général', '#CCCCCC'), (2, 'Spécialisé', '#FF0000');
INSERT INTO cours_avec_default VALUES (1, 'Cours test', 2);

-- Test SET DEFAULT
DELETE FROM categories_cours WHERE id = 2;
SELECT * FROM cours_avec_default;  -- categorie_id revient à 1 (DEFAULT)

-- Nettoyage
DROP TABLE cours_avec_default;
DROP TABLE categories_cours;
```

## Gestion des cycles et dépendances complexes

### 🔄 Problème des références circulaires

```sql
-- Problème classique : Employé ↔ Manager
CREATE TABLE employes_cyclique (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    manager_id INTEGER,
    -- ❌ Problème : Comment insérer le premier employé ?
    FOREIGN KEY (manager_id) REFERENCES employes_cyclique(id)
);

-- ❌ Impossible d'insérer car chaque employé nécessite un manager existant
-- INSERT INTO employes_cyclique VALUES (1, 'Directeur', 1);  -- Auto-référence
```

### ✅ Solutions pour les cycles

```sql
-- Solution 1 : Permettre NULL pour casser le cycle
CREATE TABLE employes_solution1 (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    manager_id INTEGER,  -- NULL = pas de manager (directeur)
    FOREIGN KEY (manager_id) REFERENCES employes_solution1(id)
        ON DELETE SET NULL
);

-- Insertion possible
INSERT INTO employes_solution1 VALUES
    (1, 'Directeur général', NULL),        -- Pas de manager
    (2, 'Directeur IT', 1),                -- Manager = Directeur général
    (3, 'Chef équipe DB', 2),              -- Manager = Directeur IT
    (4, 'Développeur senior', 3);          -- Manager = Chef équipe

-- Solution 2 : DEFERRABLE (pas supporté par SQLite, simulation)
-- On peut utiliser des transactions pour gérer l'ordre d'insertion

-- Solution 3 : Tables séparées pour casser les cycles
CREATE TABLE employes_base (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    poste TEXT
);

CREATE TABLE hierarchie_employes (
    employe_id INTEGER,
    manager_id INTEGER,
    date_debut TEXT DEFAULT (date('now')),
    date_fin TEXT,
    PRIMARY KEY (employe_id, manager_id, date_debut),
    FOREIGN KEY (employe_id) REFERENCES employes_base(id),
    FOREIGN KEY (manager_id) REFERENCES employes_base(id)
);

-- Insertion sans problème de cycle
INSERT INTO employes_base VALUES (1, 'Alice', 'Directeur'), (2, 'Bob', 'Manager');
INSERT INTO hierarchie_employes (employe_id, manager_id) VALUES (2, 1);  -- Bob managé par Alice
```

### 🎯 Pattern pour dépendances complexes

```sql
-- Exemple : Cours avec prérequis multiples
CREATE TABLE prerequis_cours (
    cours_id INTEGER,
    cours_prerequis_id INTEGER,
    obligatoire INTEGER DEFAULT 1,  -- 1=obligatoire, 0=recommandé
    PRIMARY KEY (cours_id, cours_prerequis_id),
    FOREIGN KEY (cours_id) REFERENCES cours(id) ON DELETE CASCADE,
    FOREIGN KEY (cours_prerequis_id) REFERENCES cours(id) ON DELETE CASCADE,

    -- Empêcher l'auto-référence
    CHECK (cours_id != cours_prerequis_id)
);

-- Données d'exemple
INSERT INTO prerequis_cours VALUES
    (1, 4, 1),  -- BDD nécessite Python
    (3, 2, 1);  -- Mécanique quantique nécessite Algèbre

-- Requête pour vérifier les prérequis
WITH RECURSIVE prerequis_complets AS (
    -- Cours sans prérequis
    SELECT c.id, c.nom, 0 as niveau_prerequis
    FROM cours c
    WHERE c.id NOT IN (SELECT cours_id FROM prerequis_cours)

    UNION ALL

    -- Cours avec prérequis déjà traités
    SELECT c.id, c.nom, pc.niveau_prerequis + 1
    FROM cours c
    JOIN prerequis_cours p ON c.id = p.cours_id
    JOIN prerequis_complets pc ON p.cours_prerequis_id = pc.id
    WHERE c.id NOT IN (SELECT id FROM prerequis_complets)
)
SELECT nom, niveau_prerequis
FROM prerequis_complets
ORDER BY niveau_prerequis, nom;
```

## Diagnostic et debugging des contraintes

### 🔍 Outils de diagnostic

```sql
-- 1. Vérifier l'état des clés étrangères
PRAGMA foreign_keys;

-- 2. Lister toutes les contraintes FK d'une table
PRAGMA foreign_key_list(cours);
PRAGMA foreign_key_list(inscriptions);

-- 3. Vérifier l'intégrité référentielle globale
PRAGMA foreign_key_check;

-- 4. Vérifier une table spécifique
PRAGMA foreign_key_check(inscriptions);

-- 5. Détecter les violations actuelles (si FK désactivées)
SELECT 'Professeurs orphelins' as probleme,
       COUNT(*) as nombre
FROM professeurs p
LEFT JOIN departements d ON p.departement_id = d.id
WHERE d.id IS NULL

UNION ALL

SELECT 'Étudiants sans niveau',
       COUNT(*)
FROM etudiants e
LEFT JOIN niveaux_etude n ON e.niveau_id = n.id
WHERE n.id IS NULL

UNION ALL

SELECT 'Cours sans professeur',
       COUNT(*)
FROM cours c
LEFT JOIN professeurs p ON c.professeur_responsable_id = p.id
WHERE p.id IS NULL

UNION ALL

SELECT 'Inscriptions orphelines (étudiants)',
       COUNT(*)
FROM inscriptions i
LEFT JOIN etudiants e ON i.etudiant_id = e.id
WHERE e.id IS NULL;
```

### 🚨 Résolution des violations

```sql
-- Exemple : Réparer des violations détectées
CREATE TABLE violations_fk (
    table_source TEXT,
    rowid INTEGER,
    table_cible TEXT,
    fk_column TEXT,
    message TEXT
);

-- Simuler des violations pour démonstration
PRAGMA foreign_keys = OFF;  -- Désactiver temporairement

-- Créer des données problématiques
INSERT INTO cours (nom, code, credits, professeur_responsable_id, departement_id)
VALUES ('Cours orphelin', 'ORPH999', 3, 999, 999);

INSERT INTO inscriptions (etudiant_id, cours_id, semestre, annee_universitaire)
VALUES (999, 1, 'S1', '2024-2025');

PRAGMA foreign_keys = ON;   -- Réactiver

-- Identifier les violations
INSERT INTO violations_fk
SELECT
    'cours' as table_source,
    rowid,
    'professeurs' as table_cible,
    'professeur_responsable_id' as fk_column,
    'Professeur inexistant: ' || professeur_responsable_id
FROM cours
WHERE professeur_responsable_id NOT IN (SELECT id FROM professeurs);

-- Stratégies de réparation
-- Option 1 : Supprimer les enregistrements invalides
DELETE FROM cours
WHERE professeur_responsable_id NOT IN (SELECT id FROM professeurs);

-- Option 2 : Corriger les références
UPDATE cours
SET professeur_responsable_id = 1  -- Assigner à un professeur valide
WHERE professeur_responsable_id NOT IN (SELECT id FROM professeurs);

-- Option 3 : Créer les enregistrements manquants
INSERT INTO professeurs (id, nom, prenom, departement_id)
SELECT DISTINCT
    professeur_responsable_id,
    'Professeur inconnu',
    'ID ' || professeur_responsable_id,
    1  -- Département par défaut
FROM cours
WHERE professeur_responsable_id NOT IN (SELECT id FROM professeurs);
```

## Migration et maintenance des contraintes

### 🔄 Ajouter des FK à une base existante

```sql
-- Situation : Base existante sans FK
CREATE TABLE migration_test (
    id INTEGER PRIMARY KEY,
    nom TEXT
);

CREATE TABLE migration_enfants (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,  -- Pas de FK actuellement
    donnee TEXT
);

-- Données existantes (potentiellement problématiques)
INSERT INTO migration_test VALUES (1, 'Parent 1'), (2, 'Parent 2');
INSERT INTO migration_enfants VALUES
    (1, 1, 'OK'),
    (2, 2, 'OK'),
    (3, 999, 'PROBLÈME');  -- Référence vers parent inexistant

-- Processus de migration
BEGIN TRANSACTION;

-- 1. Identifier et nettoyer les violations
CREATE TABLE enfants_invalides AS
SELECT * FROM migration_enfants
WHERE parent_id NOT IN (SELECT id FROM migration_test);

DELETE FROM migration_enfants
WHERE parent_id NOT IN (SELECT id FROM migration_test);

-- 2. Créer nouvelle table avec FK
CREATE TABLE migration_enfants_new (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER NOT NULL,
    donnee TEXT,
    FOREIGN KEY (parent_id) REFERENCES migration_test(id)
        ON DELETE CASCADE
);

-- 3. Migrer les données valides
INSERT INTO migration_enfants_new
SELECT * FROM migration_enfants;

-- 4. Remplacer l'ancienne table
DROP TABLE migration_enfants;
ALTER TABLE migration_enfants_new RENAME TO migration_enfants;

COMMIT;

-- Vérification
PRAGMA foreign_key_check(migration_enfants);
```

### 🔧 Modification des actions référentielles

```sql
-- Impossible de modifier directement les actions FK dans SQLite
-- Il faut recréer la table

-- Exemple : Changer CASCADE en RESTRICT
BEGIN TRANSACTION;

-- 1. Créer nouvelle table avec actions modifiées
CREATE TABLE inscriptions_new (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    etudiant_id INTEGER NOT NULL,
    cours_id INTEGER NOT NULL,
    semestre TEXT NOT NULL,
    annee_universitaire TEXT NOT NULL,
    date_inscription TEXT DEFAULT (datetime('now')),
    statut TEXT DEFAULT 'inscrit' CHECK (statut IN ('inscrit', 'valide', 'abandonne')),
    note_finale REAL CHECK (note_finale BETWEEN 0 AND 20),

    UNIQUE (etudiant_id, cours_id, semestre, annee_universitaire),

    -- Actions modifiées
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id)
        ON DELETE RESTRICT,  -- Changé de CASCADE à RESTRICT
    FOREIGN KEY (cours_id) REFERENCES cours(id)
        ON DELETE RESTRICT
);

-- 2. Copier les données
INSERT INTO inscriptions_new
SELECT * FROM inscriptions;

-- 3. Recréer les index si nécessaire
CREATE INDEX idx_inscriptions_new_etudiant ON inscriptions_new(etudiant_id);
CREATE INDEX idx_inscriptions_new_cours ON inscriptions_new(cours_id);

-- 4. Remplacer
DROP TABLE inscriptions;
ALTER TABLE inscriptions_new RENAME TO inscriptions;

COMMIT;
```

## Optimisation et performance des FK

### ⚡ Index automatiques et manuels

```sql
-- SQLite crée automatiquement des index pour les PK et UNIQUE
-- Mais PAS pour les FK !

-- Vérifier les index existants
.indexes

-- Créer des index pour optimiser les FK
CREATE INDEX idx_professeurs_departement ON professeurs(departement_id);
CREATE INDEX idx_etudiants_niveau ON etudiants(niveau_id);
CREATE INDEX idx_cours_professeur ON cours(professeur_responsable_id);
CREATE INDEX idx_cours_departement ON cours(departement_id);
CREATE INDEX idx_inscriptions_etudiant ON inscriptions(etudiant_id);
CREATE INDEX idx_inscriptions_cours ON inscriptions(cours_id);

-- Index composites pour requêtes fréquentes
CREATE INDEX idx_inscriptions_semestre_annee
    ON inscriptions(semestre, annee_universitaire);
```

### 📊 Analyse de performance

```sql
-- Mesurer l'impact des FK sur les performances
.timer ON

-- Test sans FK (simulation)
CREATE TABLE test_sans_fk (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,  -- Pas de FK
    donnee TEXT
);

-- Test avec FK
CREATE TABLE test_avec_fk (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,
    donnee TEXT,
    FOREIGN KEY (parent_id) REFERENCES migration_test(id)
);

-- Mesurer les insertions
INSERT INTO test_sans_fk
SELECT value, value % 1000, 'donnee' || value
FROM generate_series(1, 10000);

INSERT INTO test_avec_fk
SELECT value, (value % 2) + 1, 'donnee' || value
FROM generate_series(1, 10000);

.timer OFF

-- Analyser les plans d'exécution
EXPLAIN QUERY PLAN
SELECT * FROM test_avec_fk t1
JOIN migration_test t2 ON t1.parent_id = t2.id;
```

## Patterns avancés avec FK

### 🎯 Soft Delete avec FK

```sql
-- Pattern : Garder les références même après "suppression"
CREATE TABLE articles (
    id INTEGER PRIMARY KEY,
    titre TEXT,
    contenu TEXT,
    supprime INTEGER DEFAULT 0,
    date_suppression TEXT
);

CREATE TABLE commentaires (
    id INTEGER PRIMARY KEY,
    article_id INTEGER NOT NULL,
    contenu TEXT,
    supprime INTEGER DEFAULT 0,

    -- FK vers article (même supprimé)
    FOREIGN KEY (article_id) REFERENCES articles(id)
        ON DELETE RESTRICT  -- Empêcher suppression physique
);

-- Vue pour articles "actifs"
CREATE VIEW articles_actifs AS
SELECT * FROM articles WHERE supprime = 0;

-- Trigger pour soft delete en cascade
CREATE TRIGGER soft_delete_article
AFTER UPDATE ON articles
WHEN NEW.supprime = 1 AND OLD.supprime = 0
BEGIN
    UPDATE commentaires
    SET supprime = 1, date_suppression = datetime('now')
    WHERE article_id = NEW.id AND supprime = 0;
END;
```

### 🔄 Versioning avec FK

```sql
-- Pattern : Versioning d'entités avec FK stables
CREATE TABLE documents_versions (
    id INTEGER PRIMARY KEY,
    document_id INTEGER NOT NULL,  -- ID stable du document
    version INTEGER NOT NULL,
    titre TEXT,
    contenu TEXT,
    actuel INTEGER DEFAULT 0,
    date_creation TEXT DEFAULT (datetime('now')),

    UNIQUE (document_id, version),
    CHECK (actuel IN (0, 1))
);

CREATE TABLE permissions_documents (
    user_id INTEGER,
    document_id INTEGER,  -- Référence l'ID stable, pas la version
    permission TEXT,

    PRIMARY KEY (user_id, document_id),
    -- FK vers l'ID stable (pas vers documents_versions.id)
    FOREIGN KEY (document_id) REFERENCES documents_versions(document_id)
        ON DELETE CASCADE
);

-- Trigger pour maintenir une seule version actuelle
CREATE TRIGGER version_actuelle_unique
BEFORE UPDATE ON documents_versions
WHEN NEW.actuel = 1 AND OLD.actuel = 0
BEGIN
    UPDATE documents_versions
    SET actuel = 0
    WHERE document_id = NEW.document_id AND actuel = 1;
END;
```

## Récapitulatif et bonnes pratiques

### ✅ Check-list des FK bien configurées

#### 🎯 Configuration de base
- [ ] **PRAGMA foreign_keys = ON** dans chaque session
- [ ] **Actions référentielles** appropriées selon la logique métier
- [ ] **Index** sur toutes les colonnes FK pour les performances
- [ ] **Noms cohérents** pour les colonnes FK (_id suffix)

#### 🔧 Actions référentielles
- [ ] **CASCADE** : Seulement quand la suppression parent implique suppression enfants
- [ ] **RESTRICT** : Pour protéger les données critiques
- [ ] **SET NULL** : Quand la référence peut être optionnelle
- [ ] **SET DEFAULT** : Avec valeurs par défaut sensées

#### 🛡️ Intégrité et diagnostic
- [ ] **Tests d'intégrité** réguliers avec PRAGMA foreign_key_check
- [ ] **Plan de migration** pour changements de structure
- [ ] **Gestion des cycles** avec NULL ou tables séparées
- [ ] **Monitoring** des violations et erreurs FK

### 🎯 Patterns de décision pour les actions FK

```sql
-- Guide de décision pour choisir l'action appropriée
CREATE TABLE guide_actions_fk (
    scenario TEXT,
    relation_type TEXT,
    action_recommandee TEXT,
    justification TEXT
);

INSERT INTO guide_actions_fk VALUES
    ('Suppression utilisateur', 'Utilisateur → Commandes', 'CASCADE', 'Commandes sans utilisateur = non-sens'),
    ('Suppression produit', 'Produit → Lignes commande', 'RESTRICT', 'Conserver historique des ventes'),
    ('Suppression catégorie', 'Catégorie → Produits', 'SET NULL', 'Produit peut exister sans catégorie'),
    ('Suppression département', 'Département → Employés', 'RESTRICT', 'Réassigner avant suppression'),
    ('Modification ID parent', 'Parent → Enfants', 'CASCADE', 'Maintenir cohérence automatiquement'),
    ('Suppression cours facultatif', 'Cours → Prérequis', 'SET NULL', 'Prérequis devient optionnel'),
    ('Suppression devise de base', 'Devise → Prix', 'SET DEFAULT', 'Revenir à devise par défaut');

-- Consultation du guide
SELECT * FROM guide_actions_fk WHERE relation_type LIKE '%→ Commandes%';
```

### 🔥 Anti-patterns à éviter absolument

```sql
-- ❌ ANTI-PATTERN 1 : FK sans index
CREATE TABLE mauvais_exemple_1 (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,  -- ❌ Pas d'index sur cette FK !
    FOREIGN KEY (parent_id) REFERENCES parents(id)
);
-- Impact : Performances dégradées sur les jointures et vérifications FK

-- ✅ CORRECT
CREATE TABLE bon_exemple_1 (
    id INTEGER PRIMARY KEY,
    parent_id INTEGER,
    FOREIGN KEY (parent_id) REFERENCES parents(id)
);
CREATE INDEX idx_bon_exemple_1_parent ON bon_exemple_1(parent_id);

-- ❌ ANTI-PATTERN 2 : CASCADE aveugle
CREATE TABLE mauvais_exemple_2 (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,
    montant REAL,
    FOREIGN KEY (client_id) REFERENCES clients(id)
        ON DELETE CASCADE  -- ❌ Supprime toutes les factures si client supprimé !
);

-- ✅ CORRECT : Soft delete ou RESTRICT
CREATE TABLE bon_exemple_2 (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,
    montant REAL,
    supprime INTEGER DEFAULT 0,
    FOREIGN KEY (client_id) REFERENCES clients(id)
        ON DELETE RESTRICT  -- Force à gérer manuellement
);

-- ❌ ANTI-PATTERN 3 : Oublier PRAGMA foreign_keys
-- Développeur pense que les FK fonctionnent mais elles sont désactivées
PRAGMA foreign_keys;  -- Toujours vérifier !

-- ❌ ANTI-PATTERN 4 : Cycles non gérés
CREATE TABLE employes_mauvais (
    id INTEGER PRIMARY KEY,
    manager_id INTEGER NOT NULL,  -- ❌ Impossible d'insérer le premier !
    FOREIGN KEY (manager_id) REFERENCES employes_mauvais(id)
);

-- ✅ CORRECT : Permettre NULL pour casser le cycle
CREATE TABLE employes_bon (
    id INTEGER PRIMARY KEY,
    manager_id INTEGER,  -- NULL = pas de manager
    FOREIGN KEY (manager_id) REFERENCES employes_bon(id)
);
```

### 🎯 Scripts d'automatisation pour la maintenance

```sql
-- === SCRIPTS DE MAINTENANCE AUTOMATISÉE ===

-- 1. Vérification quotidienne de l'intégrité
CREATE VIEW rapport_integrite_quotidien AS
SELECT
    datetime('now') as date_verification,
    'foreign_key_check' as test,
    CASE
        WHEN EXISTS (SELECT 1 FROM pragma_foreign_key_check())
        THEN '❌ VIOLATIONS DÉTECTÉES'
        ELSE '✅ Intégrité OK'
    END as resultat;

-- 2. Audit des performances FK
CREATE TABLE audit_performance_fk (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name TEXT,
    fk_column TEXT,
    has_index INTEGER,
    row_count INTEGER,
    last_check TEXT DEFAULT (datetime('now'))
);

-- Procédure pour remplir l'audit (simulée)
INSERT INTO audit_performance_fk (table_name, fk_column, has_index, row_count)
SELECT
    'inscriptions' as table_name,
    'etudiant_id' as fk_column,
    CASE WHEN EXISTS (
        SELECT 1 FROM sqlite_master
        WHERE type='index' AND sql LIKE '%etudiant_id%'
    ) THEN 1 ELSE 0 END as has_index,
    (SELECT COUNT(*) FROM inscriptions) as row_count

UNION ALL

SELECT
    'inscriptions',
    'cours_id',
    CASE WHEN EXISTS (
        SELECT 1 FROM sqlite_master
        WHERE type='index' AND sql LIKE '%cours_id%'
    ) THEN 1 ELSE 0 END,
    (SELECT COUNT(*) FROM inscriptions);

-- 3. Détection automatique des FK manquantes
CREATE VIEW fk_manquantes_detectees AS
SELECT
    'professeurs' as table_source,
    'departement_id' as colonne_fk,
    'departements' as table_cible,
    CASE WHEN EXISTS (
        SELECT 1 FROM pragma_foreign_key_list('professeurs')
        WHERE "table" = 'departements'
    ) THEN '✅ FK existe' ELSE '❌ FK manquante' END as statut

UNION ALL

SELECT
    'cours',
    'professeur_responsable_id',
    'professeurs',
    CASE WHEN EXISTS (
        SELECT 1 FROM pragma_foreign_key_list('cours')
        WHERE "table" = 'professeurs'
    ) THEN '✅ FK existe' ELSE '❌ FK manquante' END;
```

### 🔧 Outils de debugging avancés

```sql
-- === BOÎTE À OUTILS DE DEBUGGING FK ===

-- 1. Fonction pour analyser une violation FK
CREATE VIEW debug_fk_violations AS
WITH violations AS (
    SELECT
        "table" as table_name,
        rowid,
        "parent" as parent_table,
        fkid as fk_column_index
    FROM pragma_foreign_key_check()
)
SELECT
    v.table_name,
    v.rowid,
    v.parent_table,
    'Enregistrement #' || v.rowid || ' dans ' || v.table_name ||
    ' référence inexistante dans ' || v.parent_table as description_violation
FROM violations v;

-- 2. Analyseur de cardinalités FK
CREATE VIEW analyse_cardinalites_fk AS
SELECT
    'etudiants → inscriptions' as relation,
    COUNT(DISTINCT e.id) as entites_parent,
    COUNT(i.id) as entites_enfant,
    ROUND(CAST(COUNT(i.id) AS REAL) / COUNT(DISTINCT e.id), 2) as ratio_moyen,
    MAX(nb_inscriptions) as max_par_parent
FROM etudiants e
LEFT JOIN inscriptions i ON e.id = i.etudiant_id
LEFT JOIN (
    SELECT etudiant_id, COUNT(*) as nb_inscriptions
    FROM inscriptions
    GROUP BY etudiant_id
) stats ON e.id = stats.etudiant_id

UNION ALL

SELECT
    'cours → inscriptions',
    COUNT(DISTINCT c.id),
    COUNT(i.id),
    ROUND(CAST(COUNT(i.id) AS REAL) / COUNT(DISTINCT c.id), 2),
    MAX(nb_inscriptions)
FROM cours c
LEFT JOIN inscriptions i ON c.id = i.cours_id
LEFT JOIN (
    SELECT cours_id, COUNT(*) as nb_inscriptions
    FROM inscriptions
    GROUP BY cours_id
) stats ON c.id = stats.cours_id;

-- 3. Générateur de script de réparation
CREATE VIEW scripts_reparation_fk AS
SELECT
    'DELETE FROM ' || "table" || ' WHERE rowid = ' || rowid || ';' as script_suppression,
    'UPDATE ' || "table" || ' SET fk_column = NULL WHERE rowid = ' || rowid || ';' as script_nullify,
    "table" as table_concernee,
    rowid as ligne_problematique
FROM pragma_foreign_key_check();

-- 4. Détecteur de patterns problématiques
CREATE VIEW patterns_problematiques AS
SELECT
    'Tables sans FK mais avec colonnes *_id' as pattern,
    COUNT(*) as occurrences,
    'Probable FK manquante' as recommandation
FROM (
    SELECT name FROM sqlite_master WHERE type='table'
    AND sql LIKE '%_id %'
    AND name NOT IN (
        SELECT DISTINCT "table" FROM pragma_foreign_key_list(name)
    )
)

UNION ALL

SELECT
    'FK sans index correspondant',
    COUNT(*),
    'Créer index pour performance'
FROM (
    SELECT DISTINCT fkl."table", fkl."from"
    FROM sqlite_master sm
    CROSS JOIN pragma_foreign_key_list(sm.name) fkl
    WHERE sm.type = 'table'
    AND NOT EXISTS (
        SELECT 1 FROM sqlite_master idx
        WHERE idx.type = 'index'
        AND idx.sql LIKE '%' || fkl."from" || '%'
    )
);
```

### 🎯 Exercice pratique complet - Système de e-commerce

```sql
-- === EXERCICE FINAL : E-COMMERCE AVEC FK COMPLEXES ===

-- Objectif : Créer un système e-commerce complet avec toutes les FK bien configurées

sqlite3 ecommerce_fk_complet.db
PRAGMA foreign_keys = ON;

-- 1. Entités de base
CREATE TABLE regions (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    code_postal_pattern TEXT  -- Ex: '75XXX' pour Paris
);

CREATE TABLE categories_produits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT UNIQUE NOT NULL,
    parent_id INTEGER,
    niveau INTEGER DEFAULT 1,
    FOREIGN KEY (parent_id) REFERENCES categories_produits(id)
        ON DELETE RESTRICT  -- Empêcher suppression si sous-catégories
);

-- 2. Utilisateurs avec adresses
CREATE TABLE utilisateurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT UNIQUE NOT NULL,
    nom TEXT NOT NULL,
    prenom TEXT,
    date_inscription TEXT DEFAULT (datetime('now')),
    actif INTEGER DEFAULT 1
);

CREATE TABLE adresses (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    utilisateur_id INTEGER NOT NULL,
    type_adresse TEXT CHECK (type_adresse IN ('domicile', 'travail', 'livraison')),
    rue TEXT NOT NULL,
    ville TEXT NOT NULL,
    code_postal TEXT NOT NULL,
    region_id INTEGER,
    defaut INTEGER DEFAULT 0,

    -- FK avec différentes actions
    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs(id)
        ON DELETE CASCADE,  -- Supprimer adresses si utilisateur supprimé
    FOREIGN KEY (region_id) REFERENCES regions(id)
        ON DELETE SET NULL,  -- Région peut être supprimée, adresse reste

    -- Contrainte : une seule adresse par défaut par utilisateur et type
    UNIQUE (utilisateur_id, type_adresse, defaut)
    CHECK (defaut IN (0, 1))
);

-- 3. Produits avec relations complexes
CREATE TABLE fournisseurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT UNIQUE NOT NULL,
    email TEXT,
    actif INTEGER DEFAULT 1
);

CREATE TABLE produits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    description TEXT,
    sku TEXT UNIQUE NOT NULL,
    prix_ht REAL CHECK (prix_ht > 0),
    tva_taux REAL DEFAULT 0.20,
    stock INTEGER DEFAULT 0 CHECK (stock >= 0),
    actif INTEGER DEFAULT 1,
    date_creation TEXT DEFAULT (datetime('now')),

    -- FK vers entités de référence
    categorie_id INTEGER NOT NULL,
    fournisseur_id INTEGER,

    FOREIGN KEY (categorie_id) REFERENCES categories_produits(id)
        ON DELETE RESTRICT,  -- Protéger les catégories utilisées
    FOREIGN KEY (fournisseur_id) REFERENCES fournisseurs(id)
        ON DELETE SET NULL   -- Fournisseur peut disparaître
);

-- 4. Commandes avec logique métier complexe
CREATE TABLE statuts_commande (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    ordre_workflow INTEGER UNIQUE,  -- 1=panier, 2=confirmee, 3=payee, etc.
    final INTEGER DEFAULT 0
);

CREATE TABLE commandes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    numero_commande TEXT UNIQUE NOT NULL,
    utilisateur_id INTEGER NOT NULL,
    date_commande TEXT DEFAULT (datetime('now')),
    statut_id INTEGER NOT NULL DEFAULT 1,

    -- Adresses figées (snapshot au moment de la commande)
    adresse_livraison_id INTEGER,
    adresse_facturation_id INTEGER,

    -- Totaux calculés
    sous_total_ht REAL DEFAULT 0,
    total_tva REAL DEFAULT 0,
    total_ttc REAL DEFAULT 0,

    -- FK avec logique métier
    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs(id)
        ON DELETE RESTRICT,  -- Conserver historique commandes
    FOREIGN KEY (statut_id) REFERENCES statuts_commande(id)
        ON DELETE RESTRICT,
    FOREIGN KEY (adresse_livraison_id) REFERENCES adresses(id)
        ON DELETE RESTRICT,  -- Adresses figées, ne pas supprimer
    FOREIGN KEY (adresse_facturation_id) REFERENCES adresses(id)
        ON DELETE RESTRICT
);

-- 5. Lignes de commande avec contraintes métier
CREATE TABLE lignes_commande (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    commande_id INTEGER NOT NULL,
    produit_id INTEGER NOT NULL,
    quantite INTEGER CHECK (quantite > 0),
    prix_unitaire_ht REAL CHECK (prix_unitaire_ht > 0),
    tva_taux REAL,

    -- Totaux figés
    total_ligne_ht REAL,
    total_ligne_tva REAL,
    total_ligne_ttc REAL,

    -- FK avec actions logiques
    FOREIGN KEY (commande_id) REFERENCES commandes(id)
        ON DELETE CASCADE,   -- Supprimer lignes si commande supprimée
    FOREIGN KEY (produit_id) REFERENCES produits(id)
        ON DELETE RESTRICT,  -- Garder historique même si produit supprimé

    -- Contrainte : pas deux fois le même produit dans une commande
    UNIQUE (commande_id, produit_id)
);

-- === DONNÉES D'EXEMPLE ===

-- Régions
INSERT INTO regions VALUES
    (1, 'Île-de-France', '7[0-9]XXX|9[0-9]XXX'),
    (2, 'Provence-Alpes-Côte d''Azur', '0[0-9]XXX|1[0-9]XXX');

-- Catégories hiérarchiques
INSERT INTO categories_produits (id, nom, parent_id, niveau) VALUES
    (1, 'Électronique', NULL, 1),
    (2, 'Vêtements', NULL, 1),
    (3, 'Smartphones', 1, 2),
    (4, 'Ordinateurs', 1, 2),
    (5, 'T-shirts', 2, 2);

-- Statuts de commande
INSERT INTO statuts_commande VALUES
    (1, 'Panier', 1, 0),
    (2, 'Confirmée', 2, 0),
    (3, 'Payée', 3, 0),
    (4, 'Expédiée', 4, 0),
    (5, 'Livrée', 5, 1),
    (6, 'Annulée', 99, 1);

-- Fournisseurs
INSERT INTO fournisseurs (nom, email) VALUES
    ('TechCorp', 'contact@techcorp.com'),
    ('FashionStyle', 'info@fashionstyle.fr');

-- Utilisateurs et adresses
INSERT INTO utilisateurs (email, nom, prenom) VALUES
    ('alice@example.com', 'Dupont', 'Alice'),
    ('bob@example.com', 'Martin', 'Bob');

INSERT INTO adresses (utilisateur_id, type_adresse, rue, ville, code_postal, region_id, defaut) VALUES
    (1, 'domicile', '123 rue de la Paix', 'Paris', '75001', 1, 1),
    (1, 'travail', '456 avenue des Champs', 'Paris', '75008', 1, 1),
    (2, 'domicile', '789 boulevard Canebière', 'Marseille', '13001', 2, 1);

-- Produits
INSERT INTO produits (nom, sku, prix_ht, categorie_id, fournisseur_id, stock) VALUES
    ('iPhone 15', 'IP15-128', 916.67, 3, 1, 50),
    ('MacBook Pro', 'MBP-M3', 1666.67, 4, 1, 20),
    ('T-shirt Bio', 'TSH-BIO-M', 20.83, 5, 2, 100);

-- Commandes
INSERT INTO commandes (numero_commande, utilisateur_id, statut_id, adresse_livraison_id, adresse_facturation_id) VALUES
    ('CMD-2024-001', 1, 2, 1, 1),
    ('CMD-2024-002', 2, 3, 3, 3);

-- Lignes de commande
INSERT INTO lignes_commande (commande_id, produit_id, quantite, prix_unitaire_ht, tva_taux) VALUES
    (1, 1, 1, 916.67, 0.20),
    (1, 3, 2, 20.83, 0.20),
    (2, 2, 1, 1666.67, 0.20);

-- === TESTS ET VALIDATIONS ===

-- 1. Vérifier l'intégrité globale
PRAGMA foreign_key_check;

-- 2. Tester les cascades
BEGIN TRANSACTION;

-- Test CASCADE : Supprimer utilisateur doit supprimer ses adresses
SELECT COUNT(*) as adresses_avant FROM adresses WHERE utilisateur_id = 2;
DELETE FROM utilisateurs WHERE id = 2;
SELECT COUNT(*) as adresses_apres FROM adresses WHERE utilisateur_id = 2;

ROLLBACK;  -- Annuler pour garder les données

-- 3. Tester RESTRICT : Impossible de supprimer produit avec commandes
DELETE FROM produits WHERE id = 1;  -- ❌ Doit échouer

-- 4. Analyser les relations
SELECT
    u.nom || ' ' || u.prenom as client,
    c.numero_commande,
    s.nom as statut,
    COUNT(lc.id) as nb_articles,
    SUM(lc.quantite * lc.prix_unitaire_ht * (1 + lc.tva_taux)) as total_ttc
FROM utilisateurs u
JOIN commandes c ON u.id = c.utilisateur_id
JOIN statuts_commande s ON c.statut_id = s.id
LEFT JOIN lignes_commande lc ON c.id = lc.commande_id
GROUP BY u.id, c.id
ORDER BY c.date_commande DESC;

-- 5. Vérifier les contraintes métier
-- Tentative de créer deux adresses par défaut du même type (doit échouer)
INSERT INTO adresses (utilisateur_id, type_adresse, rue, ville, code_postal, defaut)
VALUES (1, 'domicile', 'Autre rue', 'Paris', '75002', 1);  -- ❌ Doit échouer

-- === TRIGGERS POUR MAINTENIR L'INTÉGRITÉ ===

-- Trigger pour calculer automatiquement les totaux
CREATE TRIGGER calculer_totaux_ligne
AFTER INSERT ON lignes_commande
BEGIN
    UPDATE lignes_commande SET
        total_ligne_ht = quantite * prix_unitaire_ht,
        total_ligne_tva = quantite * prix_unitaire_ht * tva_taux,
        total_ligne_ttc = quantite * prix_unitaire_ht * (1 + tva_taux)
    WHERE id = NEW.id;

    -- Recalculer totaux commande
    UPDATE commandes SET
        sous_total_ht = (SELECT SUM(total_ligne_ht) FROM lignes_commande WHERE commande_id = NEW.commande_id),
        total_tva = (SELECT SUM(total_ligne_tva) FROM lignes_commande WHERE commande_id = NEW.commande_id),
        total_ttc = (SELECT SUM(total_ligne_ttc) FROM lignes_commande WHERE commande_id = NEW.commande_id)
    WHERE id = NEW.commande_id;
END;

-- Trigger pour valider les transitions de statut
CREATE TRIGGER valider_transition_statut
BEFORE UPDATE ON commandes
WHEN OLD.statut_id != NEW.statut_id
BEGIN
    SELECT CASE
        WHEN NEW.statut_id <= OLD.statut_id AND NEW.statut_id != 6 THEN  -- Sauf annulation
            RAISE(ABORT, 'Transition de statut invalide')
        WHEN OLD.statut_id = 6 THEN  -- Commande annulée
            RAISE(ABORT, 'Impossible de modifier une commande annulée')
    END;
END;

-- Test du trigger
UPDATE commandes SET statut_id = 3 WHERE id = 1;  -- ✅ OK (2→3)
-- UPDATE commandes SET statut_id = 1 WHERE id = 1;  -- ❌ Erreur (3→1 interdit)
```

### 🏆 Évaluation finale - Maîtrise des FK

```sql
-- === QUIZ D'AUTO-ÉVALUATION ===

CREATE TABLE quiz_fk_maitrise (
    question TEXT,
    reponse_correcte TEXT,
    explication TEXT
);

INSERT INTO quiz_fk_maitrise VALUES
    ('Que fait PRAGMA foreign_keys par défaut dans SQLite ?',
     'Retourne 0 (désactivé)',
     'SQLite désactive les FK par défaut pour compatibilité'),

    ('Quelle action FK utiliser pour un système de commentaires ?',
     'CASCADE sur utilisateur, RESTRICT sur article',
     'Supprimer commentaires si utilisateur supprimé, mais garder historique articles'),

    ('Comment résoudre une référence circulaire employé-manager ?',
     'Permettre NULL pour manager_id',
     'NULL casse le cycle, directeur n''a pas de manager'),

    ('Pourquoi créer des index sur les colonnes FK ?',
     'Optimiser performances jointures et vérifications',
     'SQLite ne crée pas automatiquement d''index sur FK'),

    ('Que faire avant de supprimer une table référencée ?',
     'Vérifier aucune FK ne pointe vers elle',
     'PRAGMA foreign_key_check pour détecter violations');

-- Afficher le quiz
SELECT
    '📝 QUESTION: ' || question as quiz,
    '✅ RÉPONSE: ' || reponse_correcte as solution,
    '💡 EXPLICATION: ' || explication as detail
FROM quiz_fk_maitrise;
```

## Conclusion et perspectives

### 🎉 Ce que vous maîtrisez maintenant

**Compétences techniques acquises :**
- ✅ **Configuration** : Activation et gestion des FK dans SQLite
- ✅ **Actions référentielles** : Choix approprié selon le contexte métier
- ✅ **Diagnostic** : Détection et résolution des violations FK
- ✅ **Performance** : Optimisation avec index et monitoring
- ✅ **Patterns avancés** : Gestion des cycles, soft delete, versioning

**Compétences architecturales développées :**
- ✅ **Conception robuste** : Intégrité référentielle garantie
- ✅ **Évolutivité** : Stratégies de migration et maintenance
- ✅ **Debugging** : Outils et méthodes de résolution de problèmes
- ✅ **Best practices** : Anti-patterns évités, patterns recommandés

### 🚀 Impact sur vos projets

**Qualité des données :**
- 🔒 **Intégrité garantie** : Plus de références orphelines
- ⚡ **Performance optimisée** : Index appropriés sur FK
- 🛡️ **Robustesse** : Actions référentielles bien pensées
- 📊 **Maintenabilité** : Structure claire et documentée

**Développement professionnel :**
- 🎯 **Conception experte** : Choix techniques éclairés
- 🔧 **Debugging efficace** : Résolution rapide des problèmes FK
- 📈 **Évolutivité** : Bases qui supportent la croissance
- 💡 **Mentoring** : Capacité à guider d'autres développeurs

### 💡 Points clés à retenir

1. **SQLite ≠ autres SGBD** : FK désactivées par défaut, activation obligatoire
2. **Actions FK = logique métier** : CASCADE, RESTRICT, SET NULL selon le contexte
3. **Index = performance** : Obligatoires sur toutes les colonnes FK
4. **Cycles = piège courant** : NULL ou tables séparées pour les résoudre
5. **Diagnostic = maintenance** : PRAGMA foreign_key_check régulièrement

---

**💡 Dans le prochain chapitre**, nous explorerons les triggers : création, types et cas d'usage, en nous appuyant sur nos contraintes FK bien maîtrisées pour automatiser la logique métier complexe.

**🎯 Vous maîtrisez maintenant** les clés étrangères SQLite comme un expert ! Vos bases de données sont désormais robustes, cohérentes et évolutives.

⏭️
