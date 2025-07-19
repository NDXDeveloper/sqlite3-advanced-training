🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.4 Contraintes : PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK

## Introduction - Les gardiens de vos données

Les contraintes sont comme des **règles de sécurité** pour votre base de données. Elles empêchent l'insertion de données incorrectes ou incohérentes, garantissant ainsi la qualité et l'intégrité de vos informations.

> **Analogie simple** : Imaginez les contraintes comme les règles d'un jeu de société. Elles définissent ce qui est autorisé et ce qui ne l'est pas, empêchant les "tricheries" qui casseraient la logique du jeu.

**Les 4 types de contraintes principales :**
- **PRIMARY KEY** → Identifiant unique obligatoire
- **FOREIGN KEY** → Liaison entre tables
- **UNIQUE** → Valeur unique dans la colonne
- **CHECK** → Règles métier personnalisées

## Préparation - Base d'exemple

Créons une base de données d'école pour illustrer toutes les contraintes :

```sql
-- Créer notre base d'exemple
sqlite3 ecole_contraintes.db

-- Configuration
PRAGMA foreign_keys = ON;  -- ⚠️ Important pour les clés étrangères !

-- Structure de base (sans contraintes d'abord)
CREATE TABLE classes (
    id INTEGER,
    nom TEXT,
    niveau TEXT,
    nombre_max_eleves INTEGER
);

CREATE TABLE eleves (
    id INTEGER,
    nom TEXT,
    prenom TEXT,
    email TEXT,
    age INTEGER,
    classe_id INTEGER
);

CREATE TABLE notes (
    id INTEGER,
    eleve_id INTEGER,
    matiere TEXT,
    note REAL,
    date_evaluation TEXT
);
```

## PRIMARY KEY - L'identifiant unique

### 🔑 Qu'est-ce qu'une clé primaire ?

Une **PRIMARY KEY** est un identifiant unique qui permet de distinguer chaque enregistrement dans une table. C'est comme un **numéro de sécurité sociale** : unique et obligatoire.

**Caractéristiques :**
- ✅ **Unique** : Pas de doublons possibles
- ✅ **Non NULL** : Toujours obligatoire
- ✅ **Immutable** : Ne doit pas changer une fois définie
- ✅ **Une seule par table** : Maximum une clé primaire par table

### 📝 Syntaxes de PRIMARY KEY

```sql
-- Méthode 1 : Colonne auto-incrémentée (recommandée)
CREATE TABLE eleves_v2 (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    email TEXT
);

-- Méthode 2 : PRIMARY KEY simple
CREATE TABLE matieres (
    code TEXT PRIMARY KEY,  -- Ex: 'MATH', 'HIST', 'ANGL'
    nom TEXT NOT NULL,
    coefficient INTEGER DEFAULT 1
);

-- Méthode 3 : PRIMARY KEY composite (plusieurs colonnes)
CREATE TABLE planning (
    classe_id INTEGER,
    jour TEXT,
    heure TEXT,
    matiere TEXT,
    PRIMARY KEY (classe_id, jour, heure)
);
```

### 🔍 Démonstration pratique

```sql
-- Recréer la table classes avec PRIMARY KEY
DROP TABLE classes;
CREATE TABLE classes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL UNIQUE,
    niveau TEXT NOT NULL,
    nombre_max_eleves INTEGER CHECK (nombre_max_eleves > 0)
);

-- Insérer des données valides
INSERT INTO classes (nom, niveau, nombre_max_eleves) VALUES
    ('6ème A', 'Collège', 28),
    ('5ème B', 'Collège', 30),
    ('4ème A', 'Collège', 25);

-- Voir les IDs auto-générés
SELECT * FROM classes;

-- ❌ Tentative d'insertion avec ID en conflit
INSERT INTO classes (id, nom, niveau, nombre_max_eleves)
VALUES (1, '6ème C', 'Collège', 26);
-- Error: UNIQUE constraint failed: classes.id

-- ✅ Insertion sans spécifier l'ID (auto-incrémenté)
INSERT INTO classes (nom, niveau, nombre_max_eleves)
VALUES ('3ème A', 'Collège', 24);

SELECT * FROM classes ORDER BY id;
```

### 🎯 Bonnes pratiques PRIMARY KEY

```sql
-- ✅ RECOMMANDÉ : INTEGER AUTOINCREMENT
CREATE TABLE produits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prix REAL
);

-- ✅ ACCEPTABLE : Clé naturelle courte et stable
CREATE TABLE pays (
    code_iso TEXT PRIMARY KEY,  -- 'FR', 'DE', 'US'
    nom TEXT NOT NULL
);

-- ❌ À ÉVITER : Clé trop longue ou variable
CREATE TABLE utilisateurs_mauvais (
    email TEXT PRIMARY KEY,  -- L'email peut changer !
    nom TEXT,
    prenom TEXT
);

-- ✅ MIEUX : ID auto + contrainte UNIQUE sur email
CREATE TABLE utilisateurs_bon (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL UNIQUE,
    nom TEXT,
    prenom TEXT
);
```

## FOREIGN KEY - Les liaisons entre tables

### 🔗 Qu'est-ce qu'une clé étrangère ?

Une **FOREIGN KEY** crée un lien entre deux tables, garantissant que les références sont cohérentes. C'est comme une **carte d'identité** qui doit correspondre à un citoyen existant.

### ⚙️ Activation des clés étrangères

**Important** : Les clés étrangères sont **désactivées par défaut** dans SQLite !

```sql
-- Vérifier si les FK sont activées
PRAGMA foreign_keys;  -- 0 = désactivé, 1 = activé

-- Activer les clés étrangères (à faire à chaque connexion)
PRAGMA foreign_keys = ON;

-- Vérifier l'activation
PRAGMA foreign_keys;
```

### 📝 Syntaxes de FOREIGN KEY

```sql
-- Recréer la table eleves avec clé étrangère
DROP TABLE eleves;
CREATE TABLE eleves (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    email TEXT UNIQUE,
    age INTEGER CHECK (age BETWEEN 10 AND 18),
    classe_id INTEGER,
    FOREIGN KEY (classe_id) REFERENCES classes(id)
);

-- Alternative : définition inline
CREATE TABLE professeurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    matiere TEXT,
    classe_principale_id INTEGER REFERENCES classes(id)
);
```

### 🔍 Démonstration des contraintes FK

```sql
-- ✅ Insertion valide (classe existe)
INSERT INTO eleves (nom, prenom, email, age, classe_id) VALUES
    ('Dupont', 'Marie', 'marie.dupont@email.com', 12, 1),
    ('Martin', 'Pierre', 'pierre.martin@email.com', 13, 1),
    ('Durand', 'Sophie', 'sophie.durand@email.com', 11, 2);

-- Vérifier les insertions
SELECT e.nom, e.prenom, c.nom as classe
FROM eleves e
JOIN classes c ON e.classe_id = c.id;

-- ❌ Tentative d'insertion avec classe inexistante
INSERT INTO eleves (nom, prenom, age, classe_id)
VALUES ('Erreur', 'Test', 12, 999);
-- Error: FOREIGN KEY constraint failed

-- ❌ Tentative de suppression d'une classe avec élèves
DELETE FROM classes WHERE id = 1;
-- Error: FOREIGN KEY constraint failed
```

### 🔧 Actions sur les clés étrangères

```sql
-- Définir des actions automatiques
CREATE TABLE commandes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    client_id INTEGER,
    total REAL,
    FOREIGN KEY (client_id) REFERENCES clients(id)
        ON DELETE CASCADE     -- Supprime les commandes si client supprimé
        ON UPDATE CASCADE     -- Met à jour les commandes si ID client change
);

-- Autres actions possibles
CREATE TABLE historique (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    eleve_id INTEGER,
    action TEXT,
    FOREIGN KEY (eleve_id) REFERENCES eleves(id)
        ON DELETE SET NULL    -- Met NULL si élève supprimé
        ON UPDATE RESTRICT    -- Interdit la modification de l'ID élève
);
```

### 🔍 Diagnostic des contraintes FK

```sql
-- Voir toutes les contraintes de clés étrangères
PRAGMA foreign_key_list(eleves);

-- Vérifier l'intégrité des clés étrangères
PRAGMA foreign_key_check;

-- Vérifier une table spécifique
PRAGMA foreign_key_check(eleves);
```

## UNIQUE - L'unicité des données

### 🎯 Qu'est-ce que UNIQUE ?

La contrainte **UNIQUE** garantit qu'aucune valeur ne peut être dupliquée dans une colonne ou un ensemble de colonnes.

### 📝 Syntaxes de UNIQUE

```sql
-- UNIQUE sur une colonne
CREATE TABLE utilisateurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT UNIQUE NOT NULL,
    nom_utilisateur TEXT UNIQUE,
    mot_de_passe TEXT NOT NULL
);

-- UNIQUE sur plusieurs colonnes combinées
CREATE TABLE horaires (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    classe_id INTEGER,
    jour TEXT,
    heure TEXT,
    matiere TEXT,
    UNIQUE (classe_id, jour, heure),  -- Pas deux cours en même temps
    FOREIGN KEY (classe_id) REFERENCES classes(id)
);

-- UNIQUE avec nom de contrainte
CREATE TABLE examens (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    matiere TEXT,
    classe_id INTEGER,
    date_examen TEXT,
    CONSTRAINT unique_examen_classe_date UNIQUE (matiere, classe_id, date_examen)
);
```

### 🔍 Démonstration UNIQUE

```sql
-- ✅ Insertions valides
INSERT INTO utilisateurs (email, nom_utilisateur, mot_de_passe) VALUES
    ('admin@ecole.fr', 'admin', 'motdepasse123'),
    ('prof@ecole.fr', 'professeur1', 'autremotdepasse');

-- ❌ Tentative de doublon sur email
INSERT INTO utilisateurs (email, nom_utilisateur, mot_de_passe)
VALUES ('admin@ecole.fr', 'admin2', 'nouveaumotdepasse');
-- Error: UNIQUE constraint failed: utilisateurs.email

-- ❌ Tentative de doublon sur nom_utilisateur
INSERT INTO utilisateurs (email, nom_utilisateur, mot_de_passe)
VALUES ('autre@ecole.fr', 'admin', 'motdepasse');
-- Error: UNIQUE constraint failed: utilisateurs.nom_utilisateur

-- ✅ Mais les valeurs NULL sont autorisées (et multiples)
INSERT INTO utilisateurs (email, mot_de_passe)
VALUES ('sans_pseudo@ecole.fr', 'motdepasse');
INSERT INTO utilisateurs (email, mot_de_passe)
VALUES ('autre_sans_pseudo@ecole.fr', 'motdepasse');

SELECT * FROM utilisateurs;
```

### 🔧 Gestion des conflits UNIQUE

```sql
-- Ignorer les doublons
INSERT OR IGNORE INTO utilisateurs (email, nom_utilisateur, mot_de_passe)
VALUES ('admin@ecole.fr', 'nouveau_admin', 'motdepasse');

-- Remplacer en cas de doublon
INSERT OR REPLACE INTO utilisateurs (email, nom_utilisateur, mot_de_passe)
VALUES ('admin@ecole.fr', 'admin_mis_a_jour', 'nouveau_motdepasse');

-- Vérifier le résultat
SELECT * FROM utilisateurs WHERE email = 'admin@ecole.fr';
```

## CHECK - Les règles métier

### ✅ Qu'est-ce que CHECK ?

La contrainte **CHECK** permet de définir des règles métier personnalisées. C'est comme un **contrôleur qualité** qui vérifie que les données respectent vos règles.

### 📝 Syntaxes de CHECK

```sql
-- CHECK simple sur une colonne
CREATE TABLE produits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prix REAL CHECK (prix > 0),
    stock INTEGER CHECK (stock >= 0),
    taux_tva REAL CHECK (taux_tva BETWEEN 0 AND 1)
);

-- CHECK complexe avec conditions multiples
CREATE TABLE eleves_complet (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    age INTEGER CHECK (age BETWEEN 6 AND 25),
    email TEXT UNIQUE CHECK (email LIKE '%@%.%'),
    niveau TEXT CHECK (niveau IN ('CP', 'CE1', 'CE2', 'CM1', 'CM2', '6ème', '5ème', '4ème', '3ème')),
    moyenne REAL CHECK (moyenne IS NULL OR (moyenne >= 0 AND moyenne <= 20))
);

-- CHECK avec nom de contrainte
CREATE TABLE commandes_check (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    date_commande TEXT NOT NULL,
    date_livraison TEXT,
    montant REAL,
    CONSTRAINT check_dates CHECK (date_livraison IS NULL OR date_livraison >= date_commande),
    CONSTRAINT check_montant_positif CHECK (montant > 0)
);
```

### 🔍 Démonstration CHECK

```sql
-- ✅ Insertions valides
INSERT INTO eleves_complet (nom, prenom, age, email, niveau) VALUES
    ('Durand', 'Alice', 12, 'alice.durand@email.com', '6ème'),
    ('Martin', 'Bob', 14, 'bob.martin@email.com', '4ème');

-- ❌ Âge invalide
INSERT INTO eleves_complet (nom, prenom, age, email, niveau)
VALUES ('Trop', 'Jeune', 5, 'jeune@email.com', 'CP');
-- Error: CHECK constraint failed: age

-- ❌ Email invalide
INSERT INTO eleves_complet (nom, prenom, age, email, niveau)
VALUES ('Email', 'Invalide', 12, 'email_sans_arobase', '6ème');
-- Error: CHECK constraint failed: email

-- ❌ Niveau invalide
INSERT INTO eleves_complet (nom, prenom, age, email, niveau)
VALUES ('Niveau', 'Invalide', 12, 'niveau@email.com', 'Terminale');
-- Error: CHECK constraint failed: niveau

-- ✅ Moyenne NULL autorisée
INSERT INTO eleves_complet (nom, prenom, age, email, niveau, moyenne)
VALUES ('Moyenne', 'Nulle', 13, 'moyenne@email.com', '5ème', NULL);

-- ❌ Moyenne hors limites
INSERT INTO eleves_complet (nom, prenom, age, email, niveau, moyenne)
VALUES ('Moyenne', 'Trop_haute', 13, 'haute@email.com', '5ème', 25);
-- Error: CHECK constraint failed: moyenne
```

### 🎯 CHECK avancés avec fonctions

```sql
-- CHECK avec fonctions SQLite
CREATE TABLE evenements (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    date_debut TEXT CHECK (date(date_debut) = date_debut),  -- Vérifier format date
    date_fin TEXT CHECK (date(date_fin) = date_fin),
    duree_jours INTEGER,
    CHECK (date_fin >= date_debut),  -- Fin après début
    CHECK (duree_jours = julianday(date_fin) - julianday(date_debut) + 1)  -- Cohérence durée
);

-- Test des contraintes avancées
INSERT INTO evenements (nom, date_debut, date_fin, duree_jours)
VALUES ('Vacances été', '2024-07-01', '2024-08-31', 62);

-- ❌ Dates incohérentes
INSERT INTO evenements (nom, date_debut, date_fin, duree_jours)
VALUES ('Erreur', '2024-08-01', '2024-07-01', 1);
-- Error: CHECK constraint failed
```

## Combinaison de toutes les contraintes

### 🏗️ Exemple complet : Système de notes

```sql
-- Table complète avec toutes les contraintes
CREATE TABLE evaluations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- Clés étrangères
    eleve_id INTEGER NOT NULL,
    matiere_code TEXT NOT NULL,
    professeur_id INTEGER,

    -- Données métier avec contraintes
    note REAL CHECK (note BETWEEN 0 AND 20),
    coefficient INTEGER DEFAULT 1 CHECK (coefficient > 0),
    date_evaluation TEXT NOT NULL CHECK (date(date_evaluation) = date_evaluation),
    type_evaluation TEXT CHECK (type_evaluation IN ('controle', 'devoir', 'examen', 'oral')),

    -- Contraintes uniques
    UNIQUE (eleve_id, matiere_code, date_evaluation, type_evaluation),

    -- Clés étrangères avec actions
    FOREIGN KEY (eleve_id) REFERENCES eleves(id) ON DELETE CASCADE,
    FOREIGN KEY (matiere_code) REFERENCES matieres(code) ON UPDATE CASCADE,
    FOREIGN KEY (professeur_id) REFERENCES professeurs(id) ON DELETE SET NULL,

    -- Contraintes métier complexes
    CHECK (date_evaluation <= date('now')),  -- Pas de notes dans le futur
    CHECK (note IS NOT NULL OR type_evaluation = 'absent')  -- Note obligatoire sauf absence
);
```

### 🔍 Tests du système complet

```sql
-- Préparer les données de référence
INSERT INTO matieres (code, nom, coefficient) VALUES
    ('MATH', 'Mathématiques', 3),
    ('FR', 'Français', 3),
    ('HIST', 'Histoire', 2);

-- ✅ Insertion valide
INSERT INTO evaluations (eleve_id, matiere_code, note, date_evaluation, type_evaluation)
VALUES (1, 'MATH', 15.5, '2024-01-15', 'controle');

-- ❌ Note hors limites
INSERT INTO evaluations (eleve_id, matiere_code, note, date_evaluation, type_evaluation)
VALUES (1, 'FR', 25, '2024-01-16', 'devoir');
-- Error: CHECK constraint failed: note

-- ❌ Doublon sur contrainte unique
INSERT INTO evaluations (eleve_id, matiere_code, note, date_evaluation, type_evaluation)
VALUES (1, 'MATH', 12, '2024-01-15', 'controle');
-- Error: UNIQUE constraint failed

-- ✅ Même élève, même matière, même date, mais type différent
INSERT INTO evaluations (eleve_id, matiere_code, note, date_evaluation, type_evaluation)
VALUES (1, 'MATH', 16, '2024-01-15', 'oral');
```

## 🎯 Exercice pratique - Système de bibliothèque avec contraintes

### Objectif : Créer un système complet avec toutes les contraintes

```sql
-- === CRÉATION DU SYSTÈME BIBLIOTHÈQUE ===
sqlite3 bibliotheque_contraintes.db

PRAGMA foreign_keys = ON;

-- Table des auteurs
CREATE TABLE auteurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    date_naissance TEXT CHECK (date(date_naissance) = date_naissance),
    pays TEXT NOT NULL,
    email TEXT UNIQUE CHECK (email IS NULL OR email LIKE '%@%.%'),
    UNIQUE (nom, prenom, date_naissance)  -- Éviter les doublons d'auteurs
);

-- Table des éditeurs
CREATE TABLE editeurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL UNIQUE,
    adresse TEXT,
    telephone TEXT CHECK (telephone IS NULL OR length(telephone) >= 10),
    email TEXT UNIQUE CHECK (email LIKE '%@%.%')
);

-- Table des catégories
CREATE TABLE categories (
    code TEXT PRIMARY KEY CHECK (length(code) = 3 AND code = upper(code)),  -- Ex: 'ROM', 'SCI'
    nom TEXT NOT NULL UNIQUE,
    description TEXT
);

-- Table des livres avec toutes les contraintes
CREATE TABLE livres (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    isbn TEXT UNIQUE CHECK (length(isbn) IN (10, 13)),  -- ISBN-10 ou ISBN-13
    auteur_id INTEGER NOT NULL,
    editeur_id INTEGER,
    categorie_code TEXT NOT NULL,
    annee_publication INTEGER CHECK (annee_publication BETWEEN 1000 AND strftime('%Y', 'now')),
    pages INTEGER CHECK (pages > 0),
    prix REAL CHECK (prix > 0),
    stock INTEGER DEFAULT 0 CHECK (stock >= 0),
    date_acquisition TEXT DEFAULT (date('now')) CHECK (date(date_acquisition) = date_acquisition),

    -- Clés étrangères
    FOREIGN KEY (auteur_id) REFERENCES auteurs(id) ON DELETE RESTRICT,
    FOREIGN KEY (editeur_id) REFERENCES editeurs(id) ON DELETE SET NULL,
    FOREIGN KEY (categorie_code) REFERENCES categories(code) ON UPDATE CASCADE,

    -- Contraintes métier
    CHECK (date_acquisition >= date(strftime('%Y', annee_publication) || '-01-01'))  -- Acquisition après publication
);

-- Table des adhérents
CREATE TABLE adherents (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    numero_carte TEXT NOT NULL UNIQUE CHECK (length(numero_carte) = 8),
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    email TEXT UNIQUE CHECK (email LIKE '%@%.%'),
    telephone TEXT CHECK (telephone IS NULL OR length(telephone) >= 10),
    date_naissance TEXT CHECK (date(date_naissance) = date_naissance),
    date_adhesion TEXT DEFAULT (date('now')) CHECK (date(date_adhesion) = date_adhesion),
    type_adherent TEXT DEFAULT 'standard' CHECK (type_adherent IN ('standard', 'etudiant', 'senior', 'premium')),
    actif INTEGER DEFAULT 1 CHECK (actif IN (0, 1)),

    -- Contraintes métier
    CHECK (date_adhesion >= date_naissance),  -- Adhésion après naissance
    CHECK (date('now') >= date(date_naissance, '+16 years') OR type_adherent = 'etudiant')  -- Majeur sauf étudiant
);

-- Table des emprunts avec contraintes complexes
CREATE TABLE emprunts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    adherent_id INTEGER NOT NULL,
    livre_id INTEGER NOT NULL,
    date_emprunt TEXT DEFAULT (datetime('now')) CHECK (datetime(date_emprunt) = date_emprunt),
    date_retour_prevue TEXT NOT NULL CHECK (datetime(date_retour_prevue) = date_retour_prevue),
    date_retour_reel TEXT CHECK (date_retour_reel IS NULL OR datetime(date_retour_reel) = date_retour_reel),

    -- Clés étrangères
    FOREIGN KEY (adherent_id) REFERENCES adherents(id) ON DELETE CASCADE,
    FOREIGN KEY (livre_id) REFERENCES livres(id) ON DELETE RESTRICT,

    -- Contraintes métier
    CHECK (date_retour_prevue > date_emprunt),  -- Retour après emprunt
    CHECK (date_retour_reel IS NULL OR date_retour_reel >= date_emprunt),  -- Retour cohérent
    UNIQUE (livre_id, date_emprunt) -- Un livre ne peut être emprunté qu'une fois à la fois
);

-- === INSERTION DES DONNÉES DE TEST ===

-- Catégories
INSERT INTO categories (code, nom, description) VALUES
    ('ROM', 'Romans', 'Littérature romanesque'),
    ('SCI', 'Sciences', 'Ouvrages scientifiques'),
    ('HIS', 'Histoire', 'Livres d''histoire'),
    ('INF', 'Informatique', 'Manuels et guides informatiques');

-- Éditeurs
INSERT INTO editeurs (nom, adresse, telephone, email) VALUES
    ('Gallimard', '5 rue Gaston Gallimard, 75007 Paris', '0142224000', 'contact@gallimard.fr'),
    ('Le Seuil', '27 rue Jacob, 75006 Paris', '0142343930', 'info@seuil.com'),
    ('O''Reilly Media', '1005 Gravenstein Highway North, CA, USA', NULL, 'contact@oreilly.com');

-- Auteurs
INSERT INTO auteurs (nom, prenom, date_naissance, pays, email) VALUES
    ('Hugo', 'Victor', '1802-02-26', 'France', NULL),
    ('Camus', 'Albert', '1913-11-07', 'France', NULL),
    ('Knuth', 'Donald', '1938-01-10', 'États-Unis', 'knuth@stanford.edu'),
    ('Tanenbaum', 'Andrew', '1944-03-16', 'Pays-Bas', 'ast@cs.vu.nl');

-- Livres
INSERT INTO livres (titre, isbn, auteur_id, editeur_id, categorie_code, annee_publication, pages, prix) VALUES
    ('Les Misérables', '9782070409570', 1, 1, 'ROM', 1862, 1488, 12.90),
    ('L''Étranger', '9782070360024', 2, 1, 'ROM', 1942, 186, 8.50),
    ('The Art of Computer Programming', '9780201896831', 3, 3, 'INF', 1968, 672, 89.99),
    ('Computer Networks', '9780132126953', 4, 3, 'INF', 2010, 960, 149.95);

-- Adhérents
INSERT INTO adherents (numero_carte, nom, prenom, email, date_naissance, type_adherent) VALUES
    ('00000001', 'Martin', 'Jean', 'jean.martin@email.com', '1985-03-15', 'standard'),
    ('00000002', 'Dubois', 'Marie', 'marie.dubois@email.com', '2000-07-22', 'etudiant'),
    ('00000003', 'Leroy', 'Pierre', 'pierre.leroy@email.com', '1955-12-08', 'senior');

-- Emprunts
INSERT INTO emprunts (adherent_id, livre_id, date_retour_prevue) VALUES
    (1, 1, datetime('now', '+14 days')),
    (2, 3, datetime('now', '+21 days')),  -- Étudiant : prêt plus long
    (3, 2, datetime('now', '+14 days'));

-- === TESTS DES CONTRAINTES ===

-- ✅ Vérifier que tout fonctionne
SELECT
    l.titre,
    a.nom || ' ' || a.prenom as auteur,
    c.nom as categorie,
    e.nom as editeur
FROM livres l
JOIN auteurs a ON l.auteur_id = a.id
JOIN categories c ON l.categorie_code = c.code
LEFT JOIN editeurs e ON l.editeur_id = e.id;

-- ❌ Tests d'erreurs de contraintes

-- ISBN invalide
INSERT INTO livres (titre, isbn, auteur_id, categorie_code, annee_publication, pages, prix)
VALUES ('Test', '123', 1, 'ROM', 2020, 100, 10);
-- Error: CHECK constraint failed: isbn

-- Catégorie inexistante
INSERT INTO livres (titre, isbn, auteur_id, categorie_code, annee_publication, pages, prix)
VALUES ('Test', '1234567890', 1, 'XXX', 2020, 100, 10);
-- Error: FOREIGN KEY constraint failed

-- Adhérent mineur sans type étudiant
INSERT INTO adherents (numero_carte, nom, prenom, date_naissance, type_adherent)
VALUES ('00000099', 'Trop', 'Jeune', '2010-01-01', 'standard');
-- Error: CHECK constraint failed

-- Double emprunt du même livre
INSERT INTO emprunts (adherent_id, livre_id, date_retour_prevue)
VALUES (2, 1, datetime('now', '+7 days'));
-- Error: UNIQUE constraint failed

-- === VÉRIFICATIONS FINALES ===

-- Statistiques générales
SELECT 'Tables' as type, 'Auteurs' as nom, COUNT(*) as nombre FROM auteurs
UNION ALL SELECT 'Tables', 'Livres', COUNT(*) FROM livres
UNION ALL SELECT 'Tables', 'Adhérents', COUNT(*) FROM adherents
UNION ALL SELECT 'Tables', 'Emprunts actifs', COUNT(*) FROM emprunts WHERE date_retour_reel IS NULL;

-- Vérifier l'intégrité
PRAGMA integrity_check;
PRAGMA foreign_key_check;

-- Vue des emprunts en cours
SELECT
    ad.nom || ' ' || ad.prenom as adherent,
    l.titre,
    e.date_emprunt,
    e.date_retour_prevue,
    CASE
        WHEN date('now') > date(e.date_retour_prevue) THEN '⚠️ En retard'
        ELSE '✅ Dans les temps'
    END as statut
FROM emprunts e
JOIN adherents ad ON e.adherent_id = ad.id
JOIN livres l ON e.livre_id = l.id
WHERE e.date_retour_reel IS NULL
ORDER BY e.date_retour_prevue;
```

## Récapitulatif des bonnes pratiques

### ✅ PRIMARY KEY

- **Toujours une PK** par table
- **INTEGER AUTOINCREMENT** pour la simplicité
- **Clés naturelles** seulement si stables et courtes
- **Jamais de données métier** dans la PK

### ✅ FOREIGN KEY

- **Activer explicitement** : `PRAGMA foreign_keys = ON`
- **Définir les actions** appropriées (CASCADE, SET NULL, RESTRICT)
- **Vérifier régulièrement** : `PRAGMA foreign_key_check`
- **Planifier l'ordre** de suppression

### ✅ UNIQUE

- **Préférer UNIQUE à PRIMARY KEY** pour les contraintes métier
- **Combiner plusieurs colonnes** si nécessaire
- **Gérer les NULL** correctement (multiples NULL autorisées)
- **Nommer les contraintes** pour un meilleur debugging

### ✅ CHECK

- **Valider les données** à l'insertion, pas après
- **Utiliser des plages de valeurs** raisonnables
- **Combiner avec NOT NULL** quand approprié
- **Éviter les contraintes trop complexes** (performance)

## Diagnostic et dépannage des contraintes

### 🔍 Commandes de diagnostic

```sql
-- Voir toutes les contraintes d'une table
PRAGMA table_info(ma_table);

-- Voir les clés étrangères spécifiquement
PRAGMA foreign_key_list(ma_table);

-- Vérifier l'intégrité générale
PRAGMA integrity_check;

-- Vérifier les clés étrangères
PRAGMA foreign_key_check;
PRAGMA foreign_key_check(ma_table);

-- Voir le schéma complet
.schema ma_table
```

### 🚨 Erreurs courantes et solutions

**Erreur : FOREIGN KEY constraint failed**

```sql
-- Problème : Tentative d'insertion avec référence inexistante
-- INSERT INTO commandes (client_id) VALUES (999);

-- Solution 1 : Vérifier les données de référence
SELECT id FROM clients WHERE id = 999;

-- Solution 2 : Insérer d'abord la référence
INSERT INTO clients (nom) VALUES ('Nouveau Client');
INSERT INTO commandes (client_id) VALUES (last_insert_rowid());

-- Solution 3 : Utiliser une clé existante
SELECT id FROM clients LIMIT 1;  -- Prendre un ID existant
```

**Erreur : UNIQUE constraint failed**

```sql
-- Problème : Tentative d'insertion d'un doublon
-- INSERT INTO users (email) VALUES ('existing@email.com');

-- Solution 1 : Ignorer le doublon
INSERT OR IGNORE INTO users (email) VALUES ('existing@email.com');

-- Solution 2 : Mettre à jour en cas de conflit
INSERT OR REPLACE INTO users (email, nom) VALUES ('existing@email.com', 'Nouveau Nom');

-- Solution 3 : Vérifier avant insertion
SELECT COUNT(*) FROM users WHERE email = 'existing@email.com';
-- Si 0, alors insérer
```

**Erreur : CHECK constraint failed**

```sql
-- Problème : Donnée ne respectant pas les règles métier
-- INSERT INTO produits (prix) VALUES (-10);

-- Solution : Corriger la donnée
INSERT INTO produits (prix) VALUES (10);

-- Diagnostic : Voir la contrainte
.schema produits
-- Identifier quelle contrainte CHECK a échoué
```

### 🔧 Modification des contraintes existantes

SQLite ne permet pas de modifier directement les contraintes. Voici la méthode complète :

```sql
-- Exemple : Ajouter une contrainte CHECK à une table existante
BEGIN TRANSACTION;

-- 1. Créer une nouvelle table avec les contraintes
CREATE TABLE produits_new (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prix REAL CHECK (prix > 0),           -- Nouvelle contrainte
    stock INTEGER CHECK (stock >= 0),     -- Nouvelle contrainte
    categorie TEXT NOT NULL              -- Nouvelle contrainte
);

-- 2. Copier les données valides (filtrer les invalides)
INSERT INTO produits_new (id, nom, prix, stock, categorie)
SELECT id, nom, prix, stock, COALESCE(categorie, 'Non défini')
FROM produits
WHERE prix > 0 AND stock >= 0;         -- Filtrer les données invalides

-- 3. Sauvegarder les données non conformes si nécessaire
CREATE TABLE produits_invalides AS
SELECT *, 'prix négatif ou stock négatif' as raison
FROM produits
WHERE prix <= 0 OR stock < 0;

-- 4. Supprimer l'ancienne table
DROP TABLE produits;

-- 5. Renommer la nouvelle table
ALTER TABLE produits_new RENAME TO produits;

-- 6. Recréer les index si nécessaire
CREATE INDEX idx_produits_nom ON produits(nom);

COMMIT;

-- Vérifier le résultat
SELECT COUNT(*) FROM produits;
SELECT COUNT(*) FROM produits_invalides;
```

## Contraintes avancées et cas d'usage spéciaux

### 📊 Contraintes conditionnelles

```sql
-- CHECK conditionnel selon une autre colonne
CREATE TABLE commandes_avancees (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type_commande TEXT CHECK (type_commande IN ('standard', 'express', 'premium')),
    prix_base REAL CHECK (prix_base > 0),
    frais_express REAL DEFAULT 0,

    -- Contrainte conditionnelle : frais express obligatoires si type express
    CHECK (
        (type_commande = 'express' AND frais_express > 0) OR
        (type_commande != 'express' AND frais_express = 0)
    ),

    -- Prix total cohérent
    prix_total REAL,
    CHECK (prix_total = prix_base + frais_express)
);

-- Test de la contrainte conditionnelle
INSERT INTO commandes_avancees (type_commande, prix_base, frais_express, prix_total) VALUES
    ('standard', 100, 0, 100),           -- ✅ OK
    ('express', 100, 15, 115),           -- ✅ OK
    ('express', 100, 0, 100);            -- ❌ Erreur : frais express manquants
```

### 🔗 Contraintes multi-tables avec triggers

Parfois, les contraintes simples ne suffisent pas. On peut utiliser des triggers :

```sql
-- Contrainte : Un élève ne peut avoir plus de 3 emprunts simultanés
CREATE TRIGGER limite_emprunts_eleve
BEFORE INSERT ON emprunts
WHEN (
    SELECT COUNT(*)
    FROM emprunts
    WHERE adherent_id = NEW.adherent_id
    AND date_retour_reel IS NULL
) >= 3
BEGIN
    SELECT RAISE(ABORT, 'Un adhérent ne peut avoir plus de 3 emprunts simultanés');
END;

-- Test du trigger
-- Si un adhérent a déjà 3 emprunts, le 4ème sera rejeté

-- Contrainte : Stock automatique lors des emprunts
CREATE TRIGGER update_stock_emprunt
AFTER INSERT ON emprunts
BEGIN
    UPDATE livres
    SET stock = stock - 1
    WHERE id = NEW.livre_id AND stock > 0;

    -- Vérifier que le stock n'est pas négatif
    SELECT CASE
        WHEN (SELECT stock FROM livres WHERE id = NEW.livre_id) < 0
        THEN RAISE(ABORT, 'Stock insuffisant pour cet emprunt')
    END;
END;
```

### 🎯 Contraintes de validation de formats

```sql
-- Validation de formats complexes
CREATE TABLE contacts_avances (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    -- Email avec validation stricte
    email TEXT CHECK (
        email GLOB '*@*.*' AND
        email NOT GLOB '*@*@*' AND
        length(email) > 5
    ),

    -- Téléphone français
    telephone TEXT CHECK (
        telephone IS NULL OR
        telephone GLOB '0[1-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'
    ),

    -- Code postal français
    code_postal TEXT CHECK (
        code_postal IS NULL OR
        (length(code_postal) = 5 AND code_postal GLOB '[0-9][0-9][0-9][0-9][0-9]')
    ),

    -- SIRET français (14 chiffres)
    siret TEXT CHECK (
        siret IS NULL OR
        (length(siret) = 14 AND siret GLOB '[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]')
    ),

    -- URL basique
    site_web TEXT CHECK (
        site_web IS NULL OR
        site_web GLOB 'http*://*.*'
    )
);

-- Tests de validation
INSERT INTO contacts_avances (email, telephone, code_postal, siret, site_web) VALUES
    ('test@example.com', '0123456789', '75001', '12345678901234', 'https://example.com'),  -- ✅ OK
    ('invalid-email', '0123456789', '75001', '12345678901234', 'https://example.com'),     -- ❌ Email invalide
    ('test@example.com', '123456789', '75001', '12345678901234', 'https://example.com'),   -- ❌ Téléphone invalide
    ('test@example.com', '0123456789', '7500', '12345678901234', 'https://example.com');   -- ❌ Code postal invalide
```

## Performance et optimisation des contraintes

### ⚡ Impact sur les performances

```sql
-- Les contraintes ont un coût en performance
-- Mesurer l'impact avec .timer

.timer ON

-- Sans contraintes (rapide)
CREATE TABLE test_sans_contraintes (
    id INTEGER,
    email TEXT,
    age INTEGER
);

-- Insérer 10000 enregistrements
INSERT INTO test_sans_contraintes
SELECT
    value as id,
    'user' || value || '@test.com' as email,
    (value % 80) + 18 as age
FROM generate_series(1, 10000);

-- Avec contraintes (plus lent)
CREATE TABLE test_avec_contraintes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT UNIQUE CHECK (email LIKE '%@%.%'),
    age INTEGER CHECK (age BETWEEN 18 AND 99)
);

-- Même insertion (sera plus lente)
INSERT INTO test_avec_contraintes (email, age)
SELECT
    'user' || value || '@test.com' as email,
    (value % 80) + 18 as age
FROM generate_series(1, 10000);

.timer OFF
```

### 🔧 Optimisation des contraintes

```sql
-- 1. Index automatiques sur les contraintes UNIQUE
-- SQLite crée automatiquement un index pour chaque contrainte UNIQUE

-- 2. Optimiser l'ordre des contraintes CHECK
-- Placer les checks les plus sélectifs en premier
CREATE TABLE optimized_checks (
    id INTEGER PRIMARY KEY,
    status TEXT CHECK (status IN ('active', 'inactive')),  -- Plus sélectif
    score REAL CHECK (score BETWEEN 0 AND 100),           -- Moins sélectif
    name TEXT CHECK (length(name) > 2)                    -- Moins sélectif
);

-- 3. Éviter les contraintes CHECK complexes en production
-- Préférer des triggers ou la validation applicative pour les règles complexes

-- 4. Désactiver temporairement les contraintes pour les imports massifs
PRAGMA foreign_keys = OFF;
-- ... import massif ...
PRAGMA foreign_key_check;  -- Vérifier après import
PRAGMA foreign_keys = ON;
```

### 📊 Monitoring des contraintes

```sql
-- Script de monitoring des violations de contraintes
CREATE TABLE contrainte_violations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name TEXT,
    contrainte_type TEXT,
    error_message TEXT,
    data_attempted TEXT,
    timestamp TEXT DEFAULT (datetime('now'))
);

-- Trigger pour logger les violations (exemple pour une table)
CREATE TRIGGER log_violations_users
AFTER INSERT ON users
WHEN NEW.email NOT LIKE '%@%.%'
BEGIN
    INSERT INTO contrainte_violations (table_name, contrainte_type, error_message, data_attempted)
    VALUES ('users', 'CHECK email', 'Email format invalide', NEW.email);
    SELECT RAISE(ABORT, 'Email format invalide');
END;
```

## Cas d'usage avancés et patterns

### 🏗️ Pattern : Soft Delete avec contraintes

```sql
-- Table avec suppression logique
CREATE TABLE documents (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    contenu TEXT,
    supprime INTEGER DEFAULT 0 CHECK (supprime IN (0, 1)),
    date_suppression TEXT,

    -- Contrainte : date_suppression cohérente avec supprime
    CHECK (
        (supprime = 0 AND date_suppression IS NULL) OR
        (supprime = 1 AND date_suppression IS NOT NULL)
    )
);

-- Vue pour les documents actifs
CREATE VIEW documents_actifs AS
SELECT id, titre, contenu
FROM documents
WHERE supprime = 0;

-- Trigger pour soft delete
CREATE TRIGGER soft_delete_document
INSTEAD OF DELETE ON documents_actifs
BEGIN
    UPDATE documents
    SET supprime = 1, date_suppression = datetime('now')
    WHERE id = OLD.id;
END;
```

### 🔄 Pattern : Versioning avec contraintes

```sql
-- Table avec versioning
CREATE TABLE articles_versions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    article_id INTEGER NOT NULL,
    version INTEGER NOT NULL,
    titre TEXT NOT NULL,
    contenu TEXT,
    auteur_id INTEGER,
    date_creation TEXT DEFAULT (datetime('now')),
    actuel INTEGER DEFAULT 0 CHECK (actuel IN (0, 1)),

    -- Contraintes de versioning
    UNIQUE (article_id, version),
    FOREIGN KEY (auteur_id) REFERENCES users(id)
);

-- Trigger : Une seule version actuelle par article
CREATE TRIGGER version_actuelle_unique
BEFORE UPDATE ON articles_versions
WHEN NEW.actuel = 1 AND OLD.actuel = 0
BEGIN
    UPDATE articles_versions
    SET actuel = 0
    WHERE article_id = NEW.article_id AND actuel = 1;
END;
```

### 🎯 Pattern : État machine avec contraintes

```sql
-- Machine à états pour les commandes
CREATE TABLE commandes_etats (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    numero_commande TEXT UNIQUE NOT NULL,
    etat TEXT NOT NULL CHECK (etat IN ('brouillon', 'confirmee', 'payee', 'expediee', 'livree', 'annulee')),
    date_creation TEXT DEFAULT (datetime('now')),
    date_modification TEXT DEFAULT (datetime('now')),

    -- Contraintes métier selon l'état
    montant REAL CHECK (
        (etat = 'brouillon' AND montant IS NULL) OR
        (etat != 'brouillon' AND montant > 0)
    ),

    date_paiement TEXT CHECK (
        (etat NOT IN ('payee', 'expediee', 'livree') AND date_paiement IS NULL) OR
        (etat IN ('payee', 'expediee', 'livree') AND date_paiement IS NOT NULL)
    )
);

-- Trigger pour valider les transitions d'état
CREATE TRIGGER valider_transition_etat
BEFORE UPDATE ON commandes_etats
WHEN NEW.etat != OLD.etat
BEGIN
    SELECT CASE
        -- Transitions autorisées depuis 'brouillon'
        WHEN OLD.etat = 'brouillon' AND NEW.etat NOT IN ('confirmee', 'annulee') THEN
            RAISE(ABORT, 'Transition invalide depuis brouillon')

        -- Transitions autorisées depuis 'confirmee'
        WHEN OLD.etat = 'confirmee' AND NEW.etat NOT IN ('payee', 'annulee') THEN
            RAISE(ABORT, 'Transition invalide depuis confirmee')

        -- Transitions autorisées depuis 'payee'
        WHEN OLD.etat = 'payee' AND NEW.etat NOT IN ('expediee', 'annulee') THEN
            RAISE(ABORT, 'Transition invalide depuis payee')

        -- Transitions autorisées depuis 'expediee'
        WHEN OLD.etat = 'expediee' AND NEW.etat != 'livree' THEN
            RAISE(ABORT, 'Transition invalide depuis expediee')

        -- États finaux
        WHEN OLD.etat IN ('livree', 'annulee') THEN
            RAISE(ABORT, 'Impossible de modifier une commande livree ou annulee')
    END;

    -- Mettre à jour la date de modification
    UPDATE commandes_etats
    SET date_modification = datetime('now')
    WHERE id = NEW.id;
END;
```

## 🎯 Exercice final - Système de réservation complet

### Objectif : Créer un système de réservation d'hôtel avec toutes les contraintes

```sql
-- === SYSTÈME DE RÉSERVATION HÔTEL ===
sqlite3 hotel_reservations.db

PRAGMA foreign_keys = ON;

-- Types de chambres
CREATE TABLE types_chambres (
    code TEXT PRIMARY KEY CHECK (length(code) <= 10),
    nom TEXT NOT NULL UNIQUE,
    capacite_max INTEGER CHECK (capacite_max BETWEEN 1 AND 8),
    prix_base REAL CHECK (prix_base > 0),
    description TEXT
);

-- Chambres
CREATE TABLE chambres (
    numero TEXT PRIMARY KEY CHECK (numero GLOB '[0-9]*'),
    type_code TEXT NOT NULL,
    etage INTEGER CHECK (etage BETWEEN 0 AND 50),
    vue TEXT CHECK (vue IN ('mer', 'montagne', 'ville', 'cour')),
    balcon INTEGER DEFAULT 0 CHECK (balcon IN (0, 1)),
    accessible_pmr INTEGER DEFAULT 0 CHECK (accessible_pmr IN (0, 1)),
    statut TEXT DEFAULT 'disponible' CHECK (statut IN ('disponible', 'occupee', 'maintenance', 'hors_service')),

    FOREIGN KEY (type_code) REFERENCES types_chambres(code) ON UPDATE CASCADE
);

-- Clients
CREATE TABLE clients (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    civilite TEXT CHECK (civilite IN ('M', 'Mme', 'Mlle')),
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    email TEXT UNIQUE CHECK (email LIKE '%@%.%'),
    telephone TEXT CHECK (length(telephone) >= 10),
    date_naissance TEXT CHECK (date(date_naissance) = date_naissance),
    adresse TEXT,
    ville TEXT,
    code_postal TEXT CHECK (length(code_postal) = 5 AND code_postal GLOB '[0-9]*'),
    pays TEXT DEFAULT 'France',
    date_inscription TEXT DEFAULT (date('now')),

    -- Client majeur
    CHECK (date('now') >= date(date_naissance, '+18 years')),
    UNIQUE (nom, prenom, date_naissance)  -- Éviter les doublons
);

-- Réservations avec contraintes métier complexes
CREATE TABLE reservations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    numero_reservation TEXT UNIQUE NOT NULL,
    client_id INTEGER NOT NULL,
    chambre_numero TEXT NOT NULL,
    date_arrivee TEXT NOT NULL CHECK (date(date_arrivee) = date_arrivee),
    date_depart TEXT NOT NULL CHECK (date(date_depart) = date_depart),
    nombre_adultes INTEGER NOT NULL CHECK (nombre_adultes >= 1),
    nombre_enfants INTEGER DEFAULT 0 CHECK (nombre_enfants >= 0),
    prix_total REAL CHECK (prix_total > 0),
    statut TEXT DEFAULT 'confirmee' CHECK (statut IN ('confirmee', 'arrivee', 'terminee', 'annulee', 'no_show')),
    date_creation TEXT DEFAULT (datetime('now')),
    commentaires TEXT,

    -- Clés étrangères
    FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE RESTRICT,
    FOREIGN KEY (chambre_numero) REFERENCES chambres(numero) ON UPDATE CASCADE,

    -- Contraintes métier essentielles
    CHECK (date_depart > date_arrivee),  -- Départ après arrivée
    CHECK (date_arrivee >= date('now')), -- Pas de réservation dans le passé
    CHECK (nombre_adultes + nombre_enfants <=
        (SELECT capacite_max FROM types_chambres tc
         JOIN chambres c ON tc.code = c.type_code
         WHERE c.numero = chambre_numero)
    ), -- Capacité respectée

    -- Pas de chevauchement de réservations
    UNIQUE (chambre_numero, date_arrivee),

    -- Prix cohérent (au moins le prix de base * nombre de nuits)
    CHECK (prix_total >=
        (julianday(date_depart) - julianday(date_arrivee)) *
        (SELECT prix_base FROM types_chambres tc
         JOIN chambres c ON tc.code = c.type_code
         WHERE c.numero = chambre_numero)
    )
);

-- Services additionnels
CREATE TABLE services (
    code TEXT PRIMARY KEY CHECK (length(code) <= 10),
    nom TEXT NOT NULL UNIQUE,
    prix REAL CHECK (prix >= 0),
    unite TEXT CHECK (unite IN ('nuit', 'sejour', 'personne', 'fois')),
    actif INTEGER DEFAULT 1 CHECK (actif IN (0, 1))
);

-- Services réservés
CREATE TABLE reservations_services (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    reservation_id INTEGER NOT NULL,
    service_code TEXT NOT NULL,
    quantite INTEGER DEFAULT 1 CHECK (quantite > 0),
    prix_unitaire REAL CHECK (prix_unitaire >= 0),

    FOREIGN KEY (reservation_id) REFERENCES reservations(id) ON DELETE CASCADE,
    FOREIGN KEY (service_code) REFERENCES services(code) ON UPDATE CASCADE,

    UNIQUE (reservation_id, service_code)
);

-- Trigger : Empêcher les réservations qui se chevauchent
CREATE TRIGGER empecher_chevauchement
BEFORE INSERT ON reservations
WHEN EXISTS (
    SELECT 1 FROM reservations r
    WHERE r.chambre_numero = NEW.chambre_numero
    AND r.statut NOT IN ('annulee', 'no_show')
    AND NOT (NEW.date_depart <= r.date_arrivee OR NEW.date_arrivee >= r.date_depart)
)
BEGIN
    SELECT RAISE(ABORT, 'Cette chambre est déjà réservée pour ces dates');
END;

-- Trigger : Générer automatiquement le numéro de réservation
CREATE TRIGGER generer_numero_reservation
AFTER INSERT ON reservations
WHEN NEW.numero_reservation = ''
BEGIN
    UPDATE reservations
    SET numero_reservation = 'RES' || strftime('%Y%m%d', 'now') || '-' ||
        printf('%04d', NEW.id)
    WHERE id = NEW.id;
END;

-- Trigger : Mettre à jour le statut de la chambre
CREATE TRIGGER maj_statut_chambre_arrivee
AFTER UPDATE ON reservations
WHEN NEW.statut = 'arrivee' AND OLD.statut = 'confirmee'
BEGIN
    UPDATE chambres
    SET statut = 'occupee'
    WHERE numero = NEW.chambre_numero;
END;

CREATE TRIGGER maj_statut_chambre_depart
AFTER UPDATE ON reservations
WHEN NEW.statut = 'terminee' AND OLD.statut = 'arrivee'
BEGIN
    UPDATE chambres
    SET statut = 'disponible'
    WHERE numero = NEW.chambre_numero;
END;

-- === DONNÉES D'EXEMPLE ===

-- Types de chambres
INSERT INTO types_chambres (code, nom, capacite_max, prix_base, description) VALUES
    ('SINGLE', 'Chambre Simple', 1, 89.00, 'Chambre pour 1 personne'),
    ('DOUBLE', 'Chambre Double', 2, 129.00, 'Chambre pour 2 personnes'),
    ('FAMILY', 'Chambre Familiale', 4, 199.00, 'Chambre pour famille'),
    ('SUITE', 'Suite', 6, 299.00, 'Suite de luxe');

-- Chambres
INSERT INTO chambres (numero, type_code, etage, vue, balcon, accessible_pmr) VALUES
    ('101', 'SINGLE', 1, 'cour', 0, 1),
    ('102', 'DOUBLE', 1, 'ville', 0, 1),
    ('201', 'DOUBLE', 2, 'mer', 1, 0),
    ('202', 'FAMILY', 2, 'mer', 1, 0),
    ('301', 'SUITE', 3, 'mer', 1, 0);

-- Services
INSERT INTO services (code, nom, prix, unite) VALUES
    ('WIFI', 'WiFi Premium', 5.00, 'nuit'),
    ('PARKING', 'Parking', 15.00, 'nuit'),
    ('BREAKFAST', 'Petit-déjeuner', 18.00, 'personne'),
    ('SPA', 'Accès Spa', 25.00, 'personne');

-- Clients
INSERT INTO clients (civilite, nom, prenom, email, telephone, date_naissance, ville, code_postal) VALUES
    ('M', 'Dupont', 'Jean', 'jean.dupont@email.com', '0123456789', '1980-05-15', 'Paris', '75001'),
    ('Mme', 'Martin', 'Sophie', 'sophie.martin@email.com', '0234567890', '1985-08-22', 'Lyon', '69001'),
    ('M', 'Durand', 'Pierre', 'pierre.durand@email.com', '0345678901', '1975-12-10', 'Marseille', '13001');

-- Réservations avec numéro auto-généré
INSERT INTO reservations (numero_reservation, client_id, chambre_numero, date_arrivee, date_depart, nombre_adultes, prix_total) VALUES
    ('', 1, '201', '2024-07-15', '2024-07-18', 2, 387.00),  -- 3 nuits * 129€
    ('', 2, '202', '2024-07-20', '2024-07-25', 2, 995.00),  -- 5 nuits * 199€
    ('', 3, '301', '2024-08-01', '2024-08-05', 4, 1196.00); -- 4 nuits * 299€

-- Services additionnels
INSERT INTO reservations_services (reservation_id, service_code, quantite, prix_unitaire) VALUES
    (1, 'BREAKFAST', 6, 18.00),  -- 2 personnes * 3 nuits
    (1, 'PARKING', 3, 15.00),    -- 3 nuits
    (2, 'BREAKFAST', 10, 18.00), -- 2 personnes * 5 nuits
    (3, 'SPA', 4, 25.00);        -- 4 personnes

-- === REQUÊTES DE VÉRIFICATION ===

-- Vue des réservations complètes
SELECT
    r.numero_reservation,
    cl.nom || ' ' || cl.prenom as client,
    ch.numero as chambre,
    tc.nom as type_chambre,
    r.date_arrivee,
    r.date_depart,
    (julianday(r.date_depart) - julianday(r.date_arrivee)) as nuits,
    r.nombre_adultes + r.nombre_enfants as total_personnes,
    r.prix_total,
    r.statut
FROM reservations r
JOIN clients cl ON r.client_id = cl.id
JOIN chambres ch ON r.chambre_numero = ch.numero
JOIN types_chambres tc ON ch.type_code = tc.code
ORDER BY r.date_arrivee;

-- Occupation par mois
SELECT
    strftime('%Y-%m', date_arrivee) as mois,
    COUNT(*) as nb_reservations,
    SUM(julianday(date_depart) - julianday(date_arrivee)) as total_nuits,
    AVG(prix_total) as prix_moyen
FROM reservations
WHERE statut != 'annulee'
GROUP BY strftime('%Y-%m', date_arrivee)
ORDER BY mois;

-- Vérifier l'intégrité
PRAGMA integrity_check;
PRAGMA foreign_key_check;

-- Tests d'erreurs

-- ❌ Réservation qui se chevauche
INSERT INTO reservations (numero_reservation, client_id, chambre_numero, date_arrivee, date_depart, nombre_adultes, prix_total)
VALUES ('TEST', 1, '201', '2024-07-16', '2024-07-19', 1, 258.00);
-- Error: Cette chambre est déjà réservée pour ces dates

-- ❌ Capacité dépassée
INSERT INTO reservations (numero_reservation, client_id, chambre_numero, date_arrivee, date_depart, nombre_adultes, prix_total)
VALUES ('TEST2', 1, '101', '2024-09-01', '2024-09-03', 3, 178.00);
-- Error: CHECK constraint failed (capacité)

-- ❌ Prix trop bas
INSERT INTO reservations (numero_reservation, client_id, chambre_numero, date_arrivee, date_depart, nombre_adultes, prix_total)
VALUES ('TEST3', 1, '102', '2024-09-01', '2024-09-03', 1, 50.00);
-- Error: CHECK constraint failed (prix minimum)

-- ✅ Réservation valide
INSERT INTO reservations (numero_reservation, client_id, chambre_numero, date_arrivee, date_depart, nombre_adultes, prix_total)
VALUES ('', 1, '102', '2024-09-01', '2024-09-03', 1, 258.00);

-- Vérifier la génération automatique du numéro
SELECT numero_reservation, client_id, chambre_numero, date_arrivee, prix_total
FROM reservations
WHERE client_id = 1 AND chambre_numero = '102';
```

## Maintenance et évolution des contraintes

### 🔄 Stratégies de migration

Quand votre application évolue, vous devez parfois modifier les contraintes existantes :

```sql
-- === MIGRATION : Ajouter une contrainte d'âge minimum ===

-- 1. Analyser les données existantes
SELECT
    COUNT(*) as total_clients,
    COUNT(CASE WHEN date('now') < date(date_naissance, '+16 years') THEN 1 END) as mineurs
FROM clients;

-- 2. Créer une table de sauvegarde pour les données non conformes
CREATE TABLE clients_mineurs_archive AS
SELECT *, 'Migration age minimum' as raison_archive
FROM clients
WHERE date('now') < date(date_naissance, '+16 years');

-- 3. Migration avec transaction
BEGIN TRANSACTION;

-- Créer la nouvelle structure
CREATE TABLE clients_new (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    civilite TEXT CHECK (civilite IN ('M', 'Mme', 'Mlle')),
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    email TEXT UNIQUE CHECK (email LIKE '%@%.%'),
    telephone TEXT CHECK (length(telephone) >= 10),
    date_naissance TEXT CHECK (date(date_naissance) = date_naissance),
    adresse TEXT,
    ville TEXT,
    code_postal TEXT CHECK (length(code_postal) = 5 AND code_postal GLOB '[0-9]*'),
    pays TEXT DEFAULT 'France',
    date_inscription TEXT DEFAULT (date('now')),

    -- Nouvelle contrainte : âge minimum 16 ans
    CHECK (date('now') >= date(date_naissance, '+16 years')),
    UNIQUE (nom, prenom, date_naissance)
);

-- Copier seulement les données conformes
INSERT INTO clients_new
SELECT * FROM clients
WHERE date('now') >= date(date_naissance, '+16 years');

-- Gérer les références dans les autres tables
UPDATE reservations
SET commentaires = COALESCE(commentaires || '; ', '') || 'Client mineur archivé'
WHERE client_id IN (SELECT id FROM clients_mineurs_archive);

-- Remplacer l'ancienne table
DROP TABLE clients;
ALTER TABLE clients_new RENAME TO clients;

COMMIT;

-- 4. Vérification post-migration
SELECT COUNT(*) as clients_restants FROM clients;
SELECT COUNT(*) as clients_archives FROM clients_mineurs_archive;
PRAGMA foreign_key_check(reservations);
```

### 📋 Audit et monitoring des contraintes

```sql
-- === SYSTÈME D'AUDIT DES CONTRAINTES ===

-- Table d'audit des violations
CREATE TABLE contraintes_audit (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name TEXT NOT NULL,
    contrainte_type TEXT NOT NULL,  -- 'PRIMARY KEY', 'FOREIGN KEY', 'UNIQUE', 'CHECK'
    contrainte_detail TEXT,
    action_tentee TEXT,  -- 'INSERT', 'UPDATE', 'DELETE'
    valeurs_rejetees TEXT,
    message_erreur TEXT,
    utilisateur TEXT,
    timestamp TEXT DEFAULT (datetime('now')),
    corrige INTEGER DEFAULT 0 CHECK (corrige IN (0, 1))
);

-- Trigger d'audit pour les violations UNIQUE sur email
CREATE TRIGGER audit_email_unique
BEFORE INSERT ON clients
WHEN NEW.email IN (SELECT email FROM clients)
BEGIN
    INSERT INTO contraintes_audit (
        table_name, contrainte_type, contrainte_detail,
        action_tentee, valeurs_rejetees, message_erreur
    ) VALUES (
        'clients', 'UNIQUE', 'email', 'INSERT',
        'email=' || NEW.email || ', nom=' || NEW.nom,
        'Tentative d''insertion avec email existant'
    );
    SELECT RAISE(ABORT, 'Email déjà utilisé');
END;

-- Vue pour le monitoring
CREATE VIEW violations_recentes AS
SELECT
    date(timestamp) as date_violation,
    table_name,
    contrainte_type,
    COUNT(*) as nombre_violations,
    COUNT(CASE WHEN corrige = 1 THEN 1 END) as corrigees
FROM contraintes_audit
WHERE timestamp >= date('now', '-30 days')
GROUP BY date(timestamp), table_name, contrainte_type
ORDER BY date_violation DESC, nombre_violations DESC;

-- Rapport quotidien des violations
SELECT * FROM violations_recentes;
```

### 🎯 Tests automatisés des contraintes

```sql
-- === SUITE DE TESTS POUR LES CONTRAINTES ===

-- Table pour stocker les résultats de tests
CREATE TABLE tests_contraintes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    test_name TEXT NOT NULL,
    table_tested TEXT NOT NULL,
    contrainte_tested TEXT NOT NULL,
    expected_result TEXT CHECK (expected_result IN ('SUCCESS', 'FAILURE')),
    actual_result TEXT,
    passed INTEGER CHECK (passed IN (0, 1)),
    error_message TEXT,
    timestamp TEXT DEFAULT (datetime('now'))
);

-- Fonction de test générique (simulée avec des requêtes)
-- Test 1 : PRIMARY KEY doit être unique
INSERT INTO tests_contraintes (test_name, table_tested, contrainte_tested, expected_result)
VALUES ('PK_Unique_Test', 'clients', 'PRIMARY KEY', 'FAILURE');

-- Simuler l'insertion qui doit échouer
-- Cette insertion va échouer et on capture le résultat
-- INSERT INTO clients (id, nom, prenom) VALUES (1, 'Test', 'Doublon');

-- Test 2 : Email UNIQUE
INSERT INTO tests_contraintes (test_name, table_tested, contrainte_tested, expected_result)
VALUES ('Email_Unique_Test', 'clients', 'UNIQUE email', 'FAILURE');

-- Test 3 : CHECK âge minimum
INSERT INTO tests_contraintes (test_name, table_tested, contrainte_tested, expected_result)
VALUES ('Age_Minimum_Test', 'clients', 'CHECK age', 'FAILURE');

-- Test 4 : FOREIGN KEY valid
INSERT INTO tests_contraintes (test_name, table_tested, contrainte_tested, expected_result)
VALUES ('FK_Valid_Test', 'reservations', 'FOREIGN KEY client_id', 'SUCCESS');

-- Test 5 : FOREIGN KEY invalid
INSERT INTO tests_contraintes (test_name, table_tested, contrainte_tested, expected_result)
VALUES ('FK_Invalid_Test', 'reservations', 'FOREIGN KEY client_id', 'FAILURE');

-- Rapport des tests
CREATE VIEW rapport_tests AS
SELECT
    test_name,
    table_tested,
    contrainte_tested,
    expected_result,
    CASE WHEN passed = 1 THEN '✅ PASS' ELSE '❌ FAIL' END as status,
    error_message,
    timestamp
FROM tests_contraintes
ORDER BY timestamp DESC;
```

## Documentation et bonnes pratiques finales

### 📚 Documentation des contraintes

```sql
-- === DOCUMENTATION INTÉGRÉE ===

-- Table de documentation des contraintes
CREATE TABLE contraintes_documentation (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name TEXT NOT NULL,
    contrainte_name TEXT,
    contrainte_type TEXT NOT NULL,
    description TEXT NOT NULL,
    justification_metier TEXT,
    exemples_valides TEXT,
    exemples_invalides TEXT,
    date_creation TEXT DEFAULT (date('now')),
    version TEXT DEFAULT '1.0'
);

-- Documentation de nos contraintes
INSERT INTO contraintes_documentation (
    table_name, contrainte_name, contrainte_type, description,
    justification_metier, exemples_valides, exemples_invalides
) VALUES
(
    'clients', 'email_unique', 'UNIQUE',
    'L''email doit être unique pour chaque client',
    'Éviter les doublons de clients et permettre l''identification unique par email',
    'jean@example.com, marie@test.fr',
    'Deux clients avec jean@example.com'
),
(
    'clients', 'age_minimum', 'CHECK',
    'Le client doit avoir au moins 16 ans',
    'Contrainte légale : les mineurs ne peuvent pas réserver seuls',
    'date_naissance: 2000-01-01 (pour 2024)',
    'date_naissance: 2010-01-01 (pour 2024)'
),
(
    'reservations', 'dates_coherentes', 'CHECK',
    'La date de départ doit être postérieure à la date d''arrivée',
    'Logique métier : impossible de partir avant d''arriver',
    'arrivée: 2024-07-01, départ: 2024-07-05',
    'arrivée: 2024-07-05, départ: 2024-07-01'
),
(
    'reservations', 'capacite_chambre', 'CHECK',
    'Le nombre total de personnes ne peut excéder la capacité de la chambre',
    'Sécurité et confort : respecter la capacité maximale',
    '2 adultes dans une chambre double',
    '5 adultes dans une chambre double'
);

-- Vue de documentation utilisateur
CREATE VIEW guide_contraintes AS
SELECT
    table_name as "Table",
    contrainte_type as "Type",
    description as "Description",
    justification_metier as "Pourquoi ?",
    exemples_valides as "Exemples valides",
    exemples_invalides as "À éviter"
FROM contraintes_documentation
ORDER BY table_name, contrainte_type;

-- Afficher le guide
SELECT * FROM guide_contraintes;
```

### 🎯 Check-list finale des bonnes pratiques

#### ✅ Design des contraintes

1. **Planification** :
   - [ ] Identifier toutes les règles métier dès la conception
   - [ ] Distinguer contraintes techniques vs métier
   - [ ] Prévoir l'évolution et la migration

2. **Implémentation** :
   - [ ] Activer `PRAGMA foreign_keys = ON`
   - [ ] Nommer les contraintes complexes
   - [ ] Documenter la justification métier

3. **Performance** :
   - [ ] Placer les contraintes CHECK les plus sélectives en premier
   - [ ] Éviter les contraintes trop complexes
   - [ ] Monitorer l'impact sur les performances

#### ✅ Maintenance

1. **Monitoring** :
   - [ ] Mettre en place un système d'audit des violations
   - [ ] Surveiller les performances d'insertion/modification
   - [ ] Vérifier régulièrement l'intégrité : `PRAGMA integrity_check`

2. **Evolution** :
   - [ ] Planifier les migrations de contraintes
   - [ ] Sauvegarder avant toute modification majeure
   - [ ] Tester les contraintes en environnement de développement

3. **Documentation** :
   - [ ] Maintenir une documentation à jour
   - [ ] Expliquer la justification métier
   - [ ] Fournir des exemples valides/invalides

#### ✅ Gestion des erreurs

1. **Messages d'erreur** :
   - [ ] Utiliser `RAISE(ABORT, 'message explicite')` dans les triggers
   - [ ] Prévoir des messages d'erreur compréhensibles
   - [ ] Logger les violations pour analyse

2. **Récupération** :
   - [ ] Implémenter des stratégies de récupération
   - [ ] Prévoir des données de fallback
   - [ ] Tester les scénarios d'erreur

## Résumé et conclusion

### 🎯 Ce que vous avez appris

**Les 4 types de contraintes maîtrisées** :
- ✅ **PRIMARY KEY** : Identifiants uniques et auto-incrémentés
- ✅ **FOREIGN KEY** : Relations cohérentes entre tables
- ✅ **UNIQUE** : Unicité de données métier importantes
- ✅ **CHECK** : Règles métier personnalisées et validation

**Techniques avancées** :
- ✅ Contraintes conditionnelles et complexes
- ✅ Triggers pour contraintes multi-tables
- ✅ Patterns avancés (soft delete, versioning, état machine)
- ✅ Migration et évolution des contraintes

**Bonnes pratiques de production** :
- ✅ Monitoring et audit des violations
- ✅ Tests automatisés des contraintes
- ✅ Documentation intégrée
- ✅ Gestion d'erreurs robuste

### 💡 Points clés à retenir

1. **Les contraintes sont vos amies** : Elles protègent l'intégrité de vos données
2. **Planifiez dès le début** : Plus facile d'ajouter que de modifier
3. **Documentez tout** : Justifiez chaque contrainte métier
4. **Testez régulièrement** : Vérifiez l'intégrité et les performances
5. **Évoluez avec précaution** : Migrations planifiées et sauvegardées

### 🚀 Pour aller plus loin

Les contraintes SQLite que vous maîtrisez maintenant constituent la fondation de bases de données robustes et fiables. Vous êtes prêt pour :

- **Module 2.5** : Requêtes avancées avec vos données bien contraintes
- **Module 3** : Conception avancée avec normalisation
- **Module 4** : Optimisation des performances
- **Projets réels** : Applications avec intégrité garantie

### 🎯 Auto-évaluation

Vous devriez maintenant être capable de :
- [ ] Créer des tables avec toutes les contraintes appropriées
- [ ] Diagnostiquer et résoudre les violations de contraintes
- [ ] Migrer et faire évoluer des contraintes existantes
- [ ] Implémenter des règles métier complexes
- [ ] Documenter et maintenir un système de contraintes

---

**💡 Dans le prochain chapitre**, nous explorerons les requêtes de base (SELECT, WHERE, ORDER BY, GROUP BY, HAVING) pour extraire efficacement les informations de vos données maintenant parfaitement contraintes et cohérentes.

**🎉 Félicitations !** Vous maîtrisez maintenant l'art de protéger et structurer vos données avec les contraintes SQLite. Vos bases de données sont désormais robustes et fiables !
