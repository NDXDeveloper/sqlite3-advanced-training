🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.2 Relations entre tables et jointures complexes

## Introduction - L'art de connecter les données

Les relations entre tables sont le **cœur battant** d'une base de données relationnelle bien conçue. Si la normalisation (chapitre 3.1) nous a appris à organiser nos données, les relations nous permettent maintenant de **les reconnecter intelligemment** pour répondre aux besoins métier.

> **Analogie simple** : Imaginez un réseau social où chaque personne est dans une table séparée, mais les amitiés, les messages et les groupes sont stockés dans d'autres tables. Les relations sont comme les liens qui permettent de reconstituer le réseau complet !

**Les types de relations que nous allons maîtriser :**
- **One-to-One (1:1)** → Une personne a un passeport
- **One-to-Many (1:N)** → Un auteur a plusieurs livres
- **Many-to-Many (N:M)** → Des étudiants suivent plusieurs cours
- **Self-Referencing** → Un employé a un manager (qui est aussi un employé)

## Rappel : Base de données école normalisée

Reprenons notre système d'école du chapitre précédent pour explorer les relations :

Dans un terminal, on ouvre une nouvelle base avec `sqlite3 ecole_relations.db`, puis on active les contraintes FK et on crée les tables.

```sql
-- Base d'exemple pour les relations
PRAGMA foreign_keys = ON;  

-- Tables de base (version simplifiée)
CREATE TABLE departements (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    responsable TEXT,
    budget REAL
);

CREATE TABLE professeurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    email TEXT UNIQUE,
    departement_id INTEGER,
    FOREIGN KEY (departement_id) REFERENCES departements(id)
);

CREATE TABLE etudiants (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    numero_etudiant TEXT UNIQUE,
    niveau TEXT  -- 'L1', 'L2', 'L3', 'M1', 'M2'
);

CREATE TABLE cours (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    code TEXT UNIQUE,
    credits INTEGER,
    professeur_id INTEGER,
    FOREIGN KEY (professeur_id) REFERENCES professeurs(id)
);

-- Données de base
INSERT INTO departements VALUES
    (1, 'Informatique', 'Prof. Martin', 150000),
    (2, 'Mathématiques', 'Prof. Bernard', 120000),
    (3, 'Physique', 'Prof. Leroy', 140000);

INSERT INTO professeurs (nom, prenom, email, departement_id) VALUES
    ('Dupont', 'Alice', 'alice.dupont@univ.fr', 1),
    ('Martin', 'Bob', 'bob.martin@univ.fr', 1),
    ('Bernard', 'Claire', 'claire.bernard@univ.fr', 2),
    ('Leroy', 'David', 'david.leroy@univ.fr', 3);

INSERT INTO etudiants (nom, prenom, numero_etudiant, niveau) VALUES
    ('Moreau', 'Emma', '20240001', 'L3'),
    ('Dubois', 'Lucas', '20240002', 'L2'),
    ('Petit', 'Camille', '20240003', 'M1'),
    ('Roux', 'Thomas', '20240004', 'L3');

INSERT INTO cours (nom, code, credits, professeur_id) VALUES
    ('Base de données', 'BDD101', 6, 1),
    ('Programmation Python', 'PY201', 4, 2),
    ('Algèbre linéaire', 'MATH201', 5, 3),
    ('Mécanique quantique', 'PHY301', 6, 4);
```

## Relation One-to-One (1:1) - Un pour un

### 🎯 Concept et cas d'usage

**Définition :** Chaque enregistrement de la table A correspond à **exactement un** enregistrement de la table B, et vice versa.

**Cas d'usage typiques :**
- Personne ↔ Passeport
- Étudiant ↔ Dossier médical confidentiel
- Produit ↔ Fiche technique détaillée
- Employé ↔ Badge d'accès

### 📝 Implémentation One-to-One

```sql
-- Exemple : Étudiant ↔ Dossier personnel confidentiel
CREATE TABLE dossiers_personnels (
    etudiant_id INTEGER PRIMARY KEY,  -- PK = FK (pattern 1:1)
    situation_familiale TEXT,
    revenus_parents REAL,
    bourse_montant REAL,
    contact_urgence_nom TEXT,
    contact_urgence_tel TEXT,
    commentaires_confidentiels TEXT,

    -- Relation 1:1 avec étudiants
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id) ON DELETE CASCADE
);

-- Insertion de données 1:1
INSERT INTO dossiers_personnels VALUES
    (1, 'Célibataire', 45000, 300, 'Marie Moreau', '0123456789', 'Excellent étudiant'),
    (3, 'Marié', 30000, 450, 'Jean Petit', '0987654321', 'Bonne intégration'),
    (4, 'Célibataire', 60000, 150, 'Anne Roux', '0555666777', 'Sportif de haut niveau');

-- Requête 1:1 : Informations complètes des étudiants
SELECT
    e.nom,
    e.prenom,
    e.numero_etudiant,
    dp.situation_familiale,
    dp.bourse_montant,
    dp.contact_urgence_nom
FROM etudiants e  
LEFT JOIN dossiers_personnels dp ON e.id = dp.etudiant_id;  
```

### 🔍 Particularités du 1:1

```sql
-- Vérification de l'intégrité 1:1
SELECT 'Vérification relation 1:1' as test;

-- Chaque étudiant a au maximum 1 dossier
SELECT
    COUNT(*) as nb_etudiants,
    COUNT(DISTINCT dp.etudiant_id) as nb_dossiers,
    COUNT(*) - COUNT(DISTINCT dp.etudiant_id) as etudiants_sans_dossier
FROM etudiants e  
LEFT JOIN dossiers_personnels dp ON e.id = dp.etudiant_id;  

-- Aucun dossier orphelin (sans étudiant)
SELECT COUNT(*) as dossiers_orphelins  
FROM dossiers_personnels dp  
LEFT JOIN etudiants e ON dp.etudiant_id = e.id  
WHERE e.id IS NULL;  
```

### ⚠️ Alternatives au 1:1

```sql
-- Alternative 1 : Colonnes supplémentaires (si toujours rempli)
CREATE TABLE etudiants_avec_infos (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    -- ... autres colonnes
    situation_familiale TEXT,  -- Directement dans la table principale
    bourse_montant REAL
);

-- Alternative 2 : Table unique avec NULLs (si optionnel)
CREATE TABLE etudiants_complets (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    -- ... colonnes de base
    contact_urgence_nom TEXT,      -- NULL si pas renseigné
    contact_urgence_tel TEXT,      -- NULL si pas renseigné
    dossier_confidentiel TEXT      -- NULL si pas de dossier
);

-- ✅ Utilisez 1:1 séparé seulement si :
-- - Données confidentielles à isoler
-- - Performances (colonnes rarement consultées)
-- - Évolutivité (structure différente)
```

## Relation One-to-Many (1:N) - Un vers plusieurs

### 🎯 Concept - La relation la plus courante

**Définition :** Un enregistrement de la table A peut correspondre à **plusieurs** enregistrements de la table B, mais chaque enregistrement de B correspond à **exactement un** enregistrement de A.

**Exemples omniprésents :**
- Département → Professeurs
- Professeur → Cours
- Commande → Lignes de commande
- Article de blog → Commentaires

### 📝 Implémentation One-to-Many

```sql
-- Exemple déjà présent : Département (1) → Professeurs (N)
-- La clé étrangère est du côté "Many"

-- Visualisation de la relation 1:N
SELECT
    d.nom as departement,
    COUNT(p.id) as nb_professeurs,
    GROUP_CONCAT(p.nom || ' ' || p.prenom, ', ') as liste_professeurs
FROM departements d  
LEFT JOIN professeurs p ON d.id = p.departement_id  
GROUP BY d.id, d.nom  
ORDER BY nb_professeurs DESC;  

-- Professeurs sans département (orphelins)
SELECT COUNT(*) as professeurs_sans_departement  
FROM professeurs  
WHERE departement_id IS NULL;  
```

### 🔧 Patterns avancés One-to-Many

```sql
-- Pattern 1 : One-to-Many avec informations sur la relation
CREATE TABLE notes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    etudiant_id INTEGER NOT NULL,
    cours_id INTEGER NOT NULL,
    note REAL CHECK (note BETWEEN 0 AND 20),
    date_evaluation TEXT,
    type_evaluation TEXT, -- 'partiel', 'final', 'tp', 'projet'
    coefficient REAL DEFAULT 1,

    -- Relation N:1 vers étudiant et cours
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id),
    FOREIGN KEY (cours_id) REFERENCES cours(id),

    -- Contrainte métier
    UNIQUE (etudiant_id, cours_id, type_evaluation, date_evaluation)
);

-- Données d'exemple
INSERT INTO notes (etudiant_id, cours_id, note, date_evaluation, type_evaluation, coefficient) VALUES
    (1, 1, 15.5, '2024-03-15', 'partiel', 0.4),
    (1, 1, 17.0, '2024-05-20', 'final', 0.6),
    (1, 2, 14.0, '2024-03-10', 'tp', 0.3),
    (2, 1, 12.5, '2024-03-15', 'partiel', 0.4),
    (3, 3, 18.0, '2024-04-01', 'partiel', 0.5);

-- Requête 1:N avec agrégation
SELECT
    e.nom || ' ' || e.prenom as etudiant,
    c.nom as cours,
    COUNT(n.id) as nb_notes,
    AVG(n.note) as moyenne_simple,
    SUM(n.note * n.coefficient) / SUM(n.coefficient) as moyenne_ponderee
FROM etudiants e  
JOIN notes n ON e.id = n.etudiant_id  
JOIN cours c ON n.cours_id = c.id  
GROUP BY e.id, c.id  
ORDER BY moyenne_ponderee DESC;  
```

### 📊 Pattern hiérarchique (Self-Referencing)

```sql
-- Auto-référence : Employé → Manager (qui est aussi un employé)
CREATE TABLE employes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    poste TEXT,
    salaire REAL,
    manager_id INTEGER,  -- Auto-référence !

    FOREIGN KEY (manager_id) REFERENCES employes(id)
);

INSERT INTO employes (nom, prenom, poste, salaire, manager_id) VALUES
    ('Directeur', 'Jean', 'Directeur général', 8000, NULL),           -- Pas de manager
    ('Martin', 'Sophie', 'Chef département', 5000, 1),               -- Manager = Directeur
    ('Dubois', 'Pierre', 'Professeur senior', 4000, 2),              -- Manager = Sophie
    ('Leroy', 'Marie', 'Professeur junior', 3000, 2),                -- Manager = Sophie
    ('Moreau', 'Paul', 'Assistant', 2500, 3);                        -- Manager = Pierre

-- Requête hiérarchique avec manager
SELECT
    e.nom || ' ' || e.prenom as employe,
    e.poste,
    COALESCE(m.nom || ' ' || m.prenom, 'Aucun manager') as manager,
    e.salaire
FROM employes e  
LEFT JOIN employes m ON e.manager_id = m.id  
ORDER BY e.salaire DESC;  

-- Requête récursive : Toute la hiérarchie
-- ⚠️ Le `WHERE h.niveau < 10` est un **garde-fou anti-cycle** : si les données
--    contenaient une boucle (A manage B et B manage A), la récursion boucleraient
--    indéfiniment avec `UNION ALL`. La limite arrête la propagation après 10
--    niveaux de profondeur, ce qui suffit largement pour un organigramme réaliste.
WITH RECURSIVE hierarchie_employes AS (
    -- Niveau racine (directeur)
    SELECT id, nom, prenom, poste, manager_id, 0 as niveau, nom as chemin
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    -- Niveaux suivants
    SELECT e.id, e.nom, e.prenom, e.poste, e.manager_id, h.niveau + 1,
           h.chemin || ' > ' || e.nom
    FROM employes e
    JOIN hierarchie_employes h ON e.manager_id = h.id
    WHERE h.niveau < 10                      -- garde-fou anti-cycle
)
SELECT
    SUBSTR('        ', 1, niveau * 2) || nom || ' ' || prenom as organisation,
    poste,
    niveau
FROM hierarchie_employes  
ORDER BY chemin;  
```

## Relation Many-to-Many (N:M) - Plusieurs vers plusieurs

### 🎯 Concept - Relations complexes

**Définition :** Plusieurs enregistrements de la table A peuvent correspondre à plusieurs enregistrements de la table B.

**Exemples classiques :**
- Étudiants ↔ Cours (un étudiant suit plusieurs cours, un cours a plusieurs étudiants)
- Acteurs ↔ Films
- Produits ↔ Catégories
- Utilisateurs ↔ Rôles

### 📝 Implémentation Many-to-Many

```sql
-- Table de liaison (junction table) pour Étudiants ↔ Cours
CREATE TABLE inscriptions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    etudiant_id INTEGER NOT NULL,
    cours_id INTEGER NOT NULL,

    -- Informations sur la relation
    date_inscription TEXT DEFAULT (date('now')),
    statut TEXT DEFAULT 'actif' CHECK (statut IN ('actif', 'abandonne', 'termine')),
    note_finale REAL,

    -- Clé composite pour éviter les doublons
    UNIQUE (etudiant_id, cours_id),

    -- Clés étrangères
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id) ON DELETE CASCADE,
    FOREIGN KEY (cours_id) REFERENCES cours(id) ON DELETE CASCADE
);

-- Données d'exemple Many-to-Many
INSERT INTO inscriptions (etudiant_id, cours_id, date_inscription, statut, note_finale) VALUES
    (1, 1, '2024-02-01', 'actif', NULL),        -- Emma → BDD
    (1, 2, '2024-02-01', 'actif', NULL),        -- Emma → Python
    (2, 1, '2024-02-05', 'actif', NULL),        -- Lucas → BDD
    (3, 1, '2024-02-01', 'termine', 16.5),      -- Camille → BDD (terminé)
    (3, 3, '2024-02-01', 'actif', NULL),        -- Camille → Maths
    (4, 2, '2024-02-10', 'abandonne', NULL);    -- Thomas → Python (abandonné)

-- Index pour optimiser les requêtes Many-to-Many
CREATE INDEX idx_inscriptions_etudiant ON inscriptions(etudiant_id);  
CREATE INDEX idx_inscriptions_cours ON inscriptions(cours_id);  
```

### 🔍 Requêtes Many-to-Many

```sql
-- Vue "étudiant" : Quels cours suit chaque étudiant ?
SELECT
    e.nom || ' ' || e.prenom as etudiant,
    e.niveau,
    COUNT(i.cours_id) as nb_cours_inscrits,
    COUNT(CASE WHEN i.statut = 'actif' THEN 1 END) as nb_cours_actifs,
    GROUP_CONCAT(
        c.nom || ' (' || i.statut || ')',
        ', '
    ) as cours_details
FROM etudiants e  
LEFT JOIN inscriptions i ON e.id = i.etudiant_id  
LEFT JOIN cours c ON i.cours_id = c.id  
GROUP BY e.id  
ORDER BY nb_cours_actifs DESC;  

-- Vue "cours" : Quels étudiants suivent chaque cours ?
SELECT
    c.nom as cours,
    c.code,
    p.nom || ' ' || p.prenom as professeur,
    COUNT(i.etudiant_id) as nb_etudiants_inscrits,
    COUNT(CASE WHEN i.statut = 'actif' THEN 1 END) as nb_etudiants_actifs,
    AVG(CASE WHEN i.note_finale IS NOT NULL THEN i.note_finale END) as moyenne_cours
FROM cours c  
JOIN professeurs p ON c.professeur_id = p.id  
LEFT JOIN inscriptions i ON c.id = i.cours_id  
GROUP BY c.id  
ORDER BY nb_etudiants_actifs DESC;  

-- Requête croisée : Matrice étudiants-cours
SELECT
    e.nom as etudiant,
    MAX(CASE WHEN c.nom = 'Base de données' THEN i.statut END) as BDD,
    MAX(CASE WHEN c.nom = 'Programmation Python' THEN i.statut END) as Python,
    MAX(CASE WHEN c.nom = 'Algèbre linéaire' THEN i.statut END) as Maths,
    MAX(CASE WHEN c.nom = 'Mécanique quantique' THEN i.statut END) as Physique
FROM etudiants e  
LEFT JOIN inscriptions i ON e.id = i.etudiant_id  
LEFT JOIN cours c ON i.cours_id = c.id  
GROUP BY e.id, e.nom;  
```

### 🔧 Patterns avancés Many-to-Many

```sql
-- Pattern 1 : Many-to-Many avec attributs complexes
CREATE TABLE collaborations_projets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    etudiant_id INTEGER NOT NULL,
    projet_id INTEGER NOT NULL,

    -- Attributs de la relation
    role TEXT, -- 'chef_projet', 'developpeur', 'testeur', 'designer'
    date_debut TEXT,
    date_fin TEXT,
    heures_travaillees INTEGER DEFAULT 0,
    evaluation_contribution REAL, -- Note sur la contribution

    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id),
    FOREIGN KEY (projet_id) REFERENCES projets(id),
    UNIQUE (etudiant_id, projet_id)
);

-- Pattern 2 : Many-to-Many ternaire (3 entités)
CREATE TABLE planning_cours (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    cours_id INTEGER NOT NULL,
    salle_id INTEGER NOT NULL,
    creneau_id INTEGER NOT NULL,

    -- Un cours, dans une salle, à un créneau donné
    date_cours TEXT,
    duree_minutes INTEGER DEFAULT 120,
    type_seance TEXT, -- 'cours', 'td', 'tp'

    FOREIGN KEY (cours_id) REFERENCES cours(id),
    FOREIGN KEY (salle_id) REFERENCES salles(id),
    FOREIGN KEY (creneau_id) REFERENCES creneaux(id),

    -- Contrainte : pas deux cours en même temps dans la même salle
    UNIQUE (salle_id, creneau_id, date_cours)
);
```

## Jointures complexes et techniques avancées

### 🎯 Types de jointures - Rappel et approfondissement

```sql
-- Préparation : Table des salles pour les exemples
CREATE TABLE salles (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    capacite INTEGER,
    type_salle TEXT
);

INSERT INTO salles VALUES
    (1, 'Amphi A', 200, 'amphitheatre'),
    (2, 'Salle TP-1', 30, 'informatique'),
    (3, 'Salle TD-2', 40, 'classique');

-- Association cours-salles pour les exemples
CREATE TABLE cours_salles (
    cours_id INTEGER,
    salle_id INTEGER,
    jour_semaine TEXT,
    heure_debut TEXT,
    PRIMARY KEY (cours_id, salle_id, jour_semaine, heure_debut),
    FOREIGN KEY (cours_id) REFERENCES cours(id),
    FOREIGN KEY (salle_id) REFERENCES salles(id)
);

INSERT INTO cours_salles VALUES
    (1, 2, 'Lundi', '09:00'),
    (2, 2, 'Mardi', '14:00'),
    (3, 1, 'Mercredi', '10:00');
-- Noter : cours_id=4 (Physique) n'a pas de salle assignée
```

### 🔗 INNER JOIN - Intersection

```sql
-- INNER JOIN : Seulement les cours qui ont une salle assignée
SELECT
    c.nom as cours,
    c.code,
    s.nom as salle,
    s.capacite,
    cs.jour_semaine,
    cs.heure_debut
FROM cours c  
INNER JOIN cours_salles cs ON c.id = cs.cours_id  
INNER JOIN salles s ON cs.salle_id = s.id  
ORDER BY cs.jour_semaine, cs.heure_debut;  
-- Résultat : 3 lignes (cours de Physique exclu car pas de salle)
```

### 🔗 LEFT JOIN - Inclusion avec NULLs

```sql
-- LEFT JOIN : Tous les cours, avec ou sans salle
SELECT
    c.nom as cours,
    c.code,
    COALESCE(s.nom, 'Aucune salle') as salle,
    COALESCE(cs.jour_semaine, 'Non programmé') as planning
FROM cours c  
LEFT JOIN cours_salles cs ON c.id = cs.cours_id  
LEFT JOIN salles s ON cs.salle_id = s.id  
ORDER BY c.nom;  
-- Résultat : 4 lignes (cours de Physique inclus avec NULL)

-- Cours sans salle assignée
SELECT c.nom, c.code  
FROM cours c  
LEFT JOIN cours_salles cs ON c.id = cs.cours_id  
WHERE cs.cours_id IS NULL;  
```

### 🔗 Jointures multiples et complexes

```sql
-- Jointure complète : Étudiants → Inscriptions → Cours → Professeurs → Départements
SELECT
    e.nom || ' ' || e.prenom as etudiant,
    c.nom as cours,
    p.nom || ' ' || p.prenom as professeur,
    d.nom as departement,
    i.statut as statut_inscription,
    COALESCE(i.note_finale, 'En cours') as note
FROM etudiants e  
JOIN inscriptions i ON e.id = i.etudiant_id  
JOIN cours c ON i.cours_id = c.id  
JOIN professeurs p ON c.professeur_id = p.id  
JOIN departements d ON p.departement_id = d.id  
WHERE i.statut IN ('actif', 'termine')  
ORDER BY d.nom, c.nom, e.nom;  

-- Statistiques multi-niveaux
SELECT
    d.nom as departement,
    COUNT(DISTINCT c.id) as nb_cours,
    COUNT(DISTINCT p.id) as nb_professeurs,
    COUNT(DISTINCT i.etudiant_id) as nb_etudiants_inscrits,
    AVG(i.note_finale) as moyenne_generale_dept
FROM departements d  
LEFT JOIN professeurs p ON d.id = p.departement_id  
LEFT JOIN cours c ON p.id = c.professeur_id  
LEFT JOIN inscriptions i ON c.id = i.cours_id AND i.note_finale IS NOT NULL  
GROUP BY d.id, d.nom  
ORDER BY nb_etudiants_inscrits DESC;  
```

### 🎯 Sous-requêtes dans les jointures

```sql
-- Jointure avec sous-requête : Étudiants ayant la meilleure moyenne
WITH moyennes_etudiants AS (
    SELECT
        etudiant_id,
        AVG(note_finale) as moyenne,
        COUNT(*) as nb_cours_termines
    FROM inscriptions
    WHERE note_finale IS NOT NULL
    GROUP BY etudiant_id
    HAVING nb_cours_termines >= 1
)
SELECT
    e.nom || ' ' || e.prenom as etudiant,
    e.niveau,
    me.moyenne,
    me.nb_cours_termines,
    RANK() OVER (ORDER BY me.moyenne DESC) as rang
FROM etudiants e  
JOIN moyennes_etudiants me ON e.id = me.etudiant_id  
ORDER BY me.moyenne DESC;  

-- Jointure avec EXISTS : Professeurs ayant des étudiants actifs
SELECT DISTINCT
    p.nom || ' ' || p.prenom as professeur,
    p.email,
    d.nom as departement
FROM professeurs p  
JOIN departements d ON p.departement_id = d.id  
WHERE EXISTS (  
    SELECT 1
    FROM cours c
    JOIN inscriptions i ON c.id = i.cours_id
    WHERE c.professeur_id = p.id
    AND i.statut = 'actif'
);
```

## Optimisation des jointures et performance

### ⚡ Index pour optimiser les jointures

```sql
-- Index essentiels pour les clés étrangères
CREATE INDEX idx_professeurs_departement ON professeurs(departement_id);  
CREATE INDEX idx_cours_professeur ON cours(professeur_id);  
CREATE INDEX idx_notes_etudiant ON notes(etudiant_id);  
CREATE INDEX idx_notes_cours ON notes(cours_id);  

-- Index composites pour requêtes fréquentes
CREATE INDEX idx_inscriptions_statut_date ON inscriptions(statut, date_inscription);  
CREATE INDEX idx_notes_cours_date ON notes(cours_id, date_evaluation);  

-- Vérifier l'utilisation des index
EXPLAIN QUERY PLAN  
SELECT c.nom, p.nom, COUNT(i.etudiant_id)  
FROM cours c  
JOIN professeurs p ON c.professeur_id = p.id  
LEFT JOIN inscriptions i ON c.id = i.cours_id  
GROUP BY c.id;  
```

### 📊 Analyse de performance des jointures

```sql
-- Créer des données de test plus volumineuses
-- ⚠️ `generate_series` n'est PAS toujours activé sur tous les builds SQLite
--    (extension optionnelle). Pour rester portable, on utilise une CTE récursive.
WITH RECURSIVE serie(value) AS (
    SELECT 5
    UNION ALL
    SELECT value + 1 FROM serie WHERE value < 1000
)
INSERT INTO etudiants (nom, prenom, numero_etudiant, niveau)  
SELECT  
    'Nom' || value,
    'Prenom' || value,
    '2024' || printf('%04d', value + 100),
    CASE (value % 5)
        WHEN 0 THEN 'L1'
        WHEN 1 THEN 'L2'
        WHEN 2 THEN 'L3'
        WHEN 3 THEN 'M1'
        ELSE 'M2'
    END
FROM serie;

-- Test de performance avec EXPLAIN
.timer ON

EXPLAIN QUERY PLAN  
SELECT  
    COUNT(*) as total_inscriptions,
    AVG(CASE WHEN i.note_finale IS NOT NULL THEN i.note_finale END) as moyenne
FROM inscriptions i  
JOIN etudiants e ON i.etudiant_id = e.id  
JOIN cours c ON i.cours_id = c.id  
WHERE e.niveau IN ('L3', 'M1');  

.timer OFF
```

### 🔧 Techniques d'optimisation avancées

```sql
-- Technique 1 : Matérialisation des jointures fréquentes
CREATE VIEW vue_etudiants_cours AS  
SELECT  
    e.id as etudiant_id,
    e.nom || ' ' || e.prenom as etudiant_nom,
    e.niveau,
    c.id as cours_id,
    c.nom as cours_nom,
    c.code,
    i.statut,
    i.note_finale,
    p.nom || ' ' || p.prenom as professeur_nom
FROM etudiants e  
JOIN inscriptions i ON e.id = i.etudiant_id  
JOIN cours c ON i.cours_id = c.id  
JOIN professeurs p ON c.professeur_id = p.id;  

-- Utilisation simplifiée
SELECT * FROM vue_etudiants_cours WHERE niveau = 'M1';

-- Technique 2 : Dénormalisation contrôlée pour reporting
CREATE TABLE rapport_inscriptions_cache (
    id INTEGER PRIMARY KEY,
    etudiant_nom TEXT,
    niveau TEXT,
    cours_nom TEXT,
    departement TEXT,
    note_finale REAL,
    derniere_maj TEXT DEFAULT (datetime('now'))
);

-- Trigger pour maintenir le cache
CREATE TRIGGER maj_cache_inscriptions  
AFTER INSERT ON inscriptions  
BEGIN  
    INSERT INTO rapport_inscriptions_cache (etudiant_nom, niveau, cours_nom, departement, note_finale)
    SELECT
        e.nom || ' ' || e.prenom,
        e.niveau,
        c.nom,
        d.nom,
        NEW.note_finale
    FROM etudiants e, cours c, professeurs p, departements d
    WHERE e.id = NEW.etudiant_id
    AND c.id = NEW.cours_id
    AND p.id = c.professeur_id
    AND d.id = p.departement_id;
END;
```

## Patterns de conception avancés avec relations

### 🎯 Pattern Observer - Audit des relations

```sql
-- Table d'audit pour tracker les changements de relations
CREATE TABLE audit_inscriptions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    action TEXT, -- 'INSERT', 'UPDATE', 'DELETE'
    table_name TEXT DEFAULT 'inscriptions',
    record_id INTEGER,
    etudiant_id INTEGER,
    cours_id INTEGER,
    ancien_statut TEXT,
    nouveau_statut TEXT,
    utilisateur TEXT,
    timestamp TEXT DEFAULT (datetime('now'))
);

-- Triggers d'audit
CREATE TRIGGER audit_inscription_insert  
AFTER INSERT ON inscriptions  
BEGIN  
    INSERT INTO audit_inscriptions (action, record_id, etudiant_id, cours_id, nouveau_statut)
    VALUES ('INSERT', NEW.id, NEW.etudiant_id, NEW.cours_id, NEW.statut);
END;

CREATE TRIGGER audit_inscription_update  
AFTER UPDATE ON inscriptions  
WHEN OLD.statut != NEW.statut  
BEGIN  
    INSERT INTO audit_inscriptions (action, record_id, etudiant_id, cours_id, ancien_statut, nouveau_statut)
    VALUES ('UPDATE', NEW.id, NEW.etudiant_id, NEW.cours_id, OLD.statut, NEW.statut);
END;

CREATE TRIGGER audit_inscription_delete  
AFTER DELETE ON inscriptions  
BEGIN  
    INSERT INTO audit_inscriptions (action, record_id, etudiant_id, cours_id, ancien_statut)
    VALUES ('DELETE', OLD.id, OLD.etudiant_id, OLD.cours_id, OLD.statut);
END;

-- Test du système d'audit
INSERT INTO inscriptions (etudiant_id, cours_id, statut) VALUES (1, 3, 'actif');  
UPDATE inscriptions SET statut = 'termine' WHERE etudiant_id = 1 AND cours_id = 3;  

-- Consulter l'historique
SELECT
    a.timestamp,
    a.action,
    e.nom || ' ' || e.prenom as etudiant,
    c.nom as cours,
    a.ancien_statut,
    a.nouveau_statut
FROM audit_inscriptions a  
LEFT JOIN etudiants e ON a.etudiant_id = e.id  
LEFT JOIN cours c ON a.cours_id = c.id  
ORDER BY a.timestamp DESC;  
```

### 🔄 Pattern State Machine - Gestion d'états

> ⚠️ **Incompatibilité à connaître** : la table `inscriptions` définie plus haut dans ce chapitre limite `statut` à trois valeurs (`'actif'`, `'abandonne'`, `'termine'`) via une contrainte `CHECK`. Pour exécuter le pattern State Machine ci-dessous qui utilise 8 statuts (`candidat`, `accepte`, `inscrit`, …), il faut **soit retirer/élargir cette `CHECK`** sur la table source, **soit appliquer ce pattern sur une autre table dédiée**. Exemple sans `CHECK` restrictive :  
>  
> ```sql
> DROP TABLE IF EXISTS inscriptions;
> CREATE TABLE inscriptions (
>     id INTEGER PRIMARY KEY AUTOINCREMENT,
>     etudiant_id INTEGER NOT NULL,
>     cours_id INTEGER NOT NULL,
>     statut TEXT,            -- pas de CHECK : la validation passe par le trigger
>     note_finale REAL,
>     UNIQUE (etudiant_id, cours_id)
> );
> ```

```sql
-- Table des états possibles et transitions autorisées
CREATE TABLE statuts_inscription (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    description TEXT,
    final INTEGER DEFAULT 0  -- 1 si c'est un état final
);

CREATE TABLE transitions_autorisees (
    id INTEGER PRIMARY KEY,
    statut_source_id INTEGER,
    statut_cible_id INTEGER,
    condition_sql TEXT,  -- Condition à vérifier
    action_sql TEXT,     -- Action à exécuter lors de la transition
    FOREIGN KEY (statut_source_id) REFERENCES statuts_inscription(id),
    FOREIGN KEY (statut_cible_id) REFERENCES statuts_inscription(id)
);

-- Définition des états
INSERT INTO statuts_inscription (id, nom, description, final) VALUES
    (1, 'candidat', 'Candidature déposée', 0),
    (2, 'accepte', 'Candidature acceptée', 0),
    (3, 'inscrit', 'Inscription confirmée', 0),
    (4, 'actif', 'Suit activement le cours', 0),
    (5, 'suspendu', 'Inscription suspendue temporairement', 0),
    (6, 'termine', 'Cours terminé avec succès', 1),
    (7, 'echec', 'Cours terminé en échec', 1),
    (8, 'abandonne', 'Cours abandonné', 1);

-- Transitions autorisées
INSERT INTO transitions_autorisees (statut_source_id, statut_cible_id, condition_sql) VALUES
    (1, 2, 'TRUE'),  -- candidat → accepté (toujours possible)
    (1, 8, 'TRUE'),  -- candidat → abandonné
    (2, 3, 'TRUE'),  -- accepté → inscrit
    (2, 8, 'TRUE'),  -- accepté → abandonné
    (3, 4, 'TRUE'),  -- inscrit → actif
    (4, 5, 'TRUE'),  -- actif → suspendu
    (4, 6, 'note_finale >= 10'),  -- actif → terminé (si note >= 10)
    (4, 7, 'note_finale < 10'),   -- actif → échec (si note < 10)
    (4, 8, 'TRUE'),  -- actif → abandonné
    (5, 4, 'TRUE'),  -- suspendu → actif
    (5, 8, 'TRUE');  -- suspendu → abandonné

-- Fonction pour vérifier et effectuer une transition
CREATE TABLE log_transitions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    inscription_id INTEGER,
    statut_avant TEXT,
    statut_apres TEXT,
    succes INTEGER,
    message TEXT,
    timestamp TEXT DEFAULT (datetime('now'))
);

-- Trigger pour valider les transitions
-- ⚠️ SQLite contrainte : le message de RAISE(ABORT, ...) DOIT être un littéral
--    constant (pas de concaténation `||` ni de référence à OLD/NEW).
--    Voir l'alternative en commentaire après la définition du trigger.
CREATE TRIGGER valider_transition_statut  
BEFORE UPDATE OF statut ON inscriptions  
WHEN OLD.statut != NEW.statut  
BEGIN  
    -- Vérifier si la transition est autorisée
    SELECT CASE
        WHEN NOT EXISTS (
            SELECT 1 FROM transitions_autorisees ta
            JOIN statuts_inscription s1 ON ta.statut_source_id = s1.id
            JOIN statuts_inscription s2 ON ta.statut_cible_id = s2.id
            WHERE s1.nom = OLD.statut AND s2.nom = NEW.statut
        ) THEN
            RAISE(ABORT, 'Transition de statut non autorisée')
    END;

    -- Logger la transition (le détail OLD→NEW est consultable dans log_transitions)
    INSERT INTO log_transitions (inscription_id, statut_avant, statut_apres, succes, message)
    VALUES (NEW.id, OLD.statut, NEW.statut, 1, 'Transition autorisée');
END;

-- 💡 Alternative pour message d'erreur dynamique : pré-loguer puis lever
--    (RAISE annule la transaction y compris ces logs s'ils sont dans la même tx,
--    donc il faut utiliser une table d'audit avec triggers AFTER ou un commit séparé).
```

### 🏗️ Pattern Factory - Relations dynamiques

```sql
-- Pattern pour créer des relations conditionnelles selon le type
CREATE TABLE types_entites (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE,
    table_cible TEXT,
    colonnes_requises TEXT  -- JSON des colonnes spécifiques
);

CREATE TABLE relations_dynamiques (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    entite_source_type INTEGER,
    entite_source_id INTEGER,
    entite_cible_type INTEGER,
    entite_cible_id INTEGER,
    type_relation TEXT,
    metadonnees TEXT,  -- JSON avec infos spécifiques
    actif INTEGER DEFAULT 1,
    FOREIGN KEY (entite_source_type) REFERENCES types_entites(id),
    FOREIGN KEY (entite_cible_type) REFERENCES types_entites(id)
);

-- Exemple : Relations entre différents types d'entités
INSERT INTO types_entites VALUES
    (1, 'etudiant', 'etudiants', '{"nom":"TEXT","prenom":"TEXT"}'),
    (2, 'professeur', 'professeurs', '{"nom":"TEXT","email":"TEXT"}'),
    (3, 'cours', 'cours', '{"nom":"TEXT","code":"TEXT"}'),
    (4, 'projet', 'projets', '{"titre":"TEXT","description":"TEXT"}');

-- Relations dynamiques : Étudiant encadré par Professeur pour un Projet
INSERT INTO relations_dynamiques (entite_source_type, entite_source_id, entite_cible_type, entite_cible_id, type_relation, metadonnees) VALUES
    (1, 1, 2, 1, 'encadrement', '{"role":"directeur","projet_id":1}'),
    (1, 2, 2, 1, 'encadrement', '{"role":"co_directeur","projet_id":1}');
```

## Techniques avancées de jointures SQLite

### 🚀 Window Functions avec relations

```sql
-- Classements et analyses avec window functions
SELECT
    e.nom || ' ' || e.prenom as etudiant,
    c.nom as cours,
    i.note_finale,

    -- Rang dans le cours
    RANK() OVER (PARTITION BY c.id ORDER BY i.note_finale DESC) as rang_cours,

    -- Rang général de l'étudiant
    RANK() OVER (ORDER BY i.note_finale DESC) as rang_general,

    -- Moyenne mobile des notes de l'étudiant
    AVG(i.note_finale) OVER (
        PARTITION BY e.id
        ORDER BY i.date_inscription
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moyenne_mobile,

    -- Écart par rapport à la moyenne du cours
    i.note_finale - AVG(i.note_finale) OVER (PARTITION BY c.id) as ecart_moyenne_cours,

    -- Note précédente et suivante de l'étudiant
    LAG(i.note_finale, 1) OVER (PARTITION BY e.id ORDER BY i.date_inscription) as note_precedente,
    LEAD(i.note_finale, 1) OVER (PARTITION BY e.id ORDER BY i.date_inscription) as note_suivante

FROM etudiants e  
JOIN inscriptions i ON e.id = i.etudiant_id  
JOIN cours c ON i.cours_id = c.id  
WHERE i.note_finale IS NOT NULL  
ORDER BY rang_general;  
```

### 🔗 Jointures récursives avancées

```sql
-- Parcours en profondeur de hiérarchies complexes
-- ⚠️ SQLite n'autorise qu'UNE SEULE référence au CTE dans la branche récursive.
--    Pour gérer des prérequis multiples (« Data Science nécessite Prog ET BDD »),
--    on utilise une table dédiée pour les prérequis (1 ligne par paire matière-prérequis)
--    plutôt qu'une colonne TEXT avec liste séparée par virgules.

CREATE TABLE matieres_hier (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    parent_id INTEGER,
    niveau INTEGER,
    FOREIGN KEY (parent_id) REFERENCES matieres_hier(id)
);

CREATE TABLE matieres_prerequis (
    matiere_id INTEGER NOT NULL,
    prerequis_id INTEGER NOT NULL,
    PRIMARY KEY (matiere_id, prerequis_id),
    FOREIGN KEY (matiere_id) REFERENCES matieres_hier(id),
    FOREIGN KEY (prerequis_id) REFERENCES matieres_hier(id)
);

INSERT INTO matieres_hier VALUES
    (1, 'Informatique', NULL, 1),
    (2, 'Programmation', 1, 2),
    (3, 'Base de données', 1, 2),
    (4, 'Python avancé', 2, 3),
    (5, 'SQL avancé', 3, 3),
    (6, 'Data Science', 1, 2);

INSERT INTO matieres_prerequis VALUES
    (3, 2),  -- BDD nécessite Programmation
    (4, 2),  -- Python avancé nécessite Programmation
    (5, 3),  -- SQL avancé nécessite BDD
    (6, 2),  -- Data Science nécessite Programmation
    (6, 3);  -- Data Science nécessite BDD

-- CTE récursive : parcours descendant de la hiérarchie parent → enfant
-- (UNE seule référence récursive au CTE, conforme à SQLite)
WITH RECURSIVE descente(id, nom, niveau_hier, profondeur, chemin) AS (
    SELECT id, nom, niveau, 0, nom
    FROM matieres_hier
    WHERE parent_id IS NULL
    UNION ALL
    SELECT m.id, m.nom, m.niveau, d.profondeur + 1, d.chemin || ' > ' || m.nom
    FROM matieres_hier m
    JOIN descente d ON m.parent_id = d.id
)
SELECT chemin, profondeur FROM descente ORDER BY chemin;

-- Pour interroger les prérequis d'une matière : simple JOIN, pas besoin de récursion
SELECT m.nom AS matiere, p.nom AS prerequis_direct  
FROM matieres_prerequis mp  
JOIN matieres_hier m ON m.id = mp.matiere_id  
JOIN matieres_hier p ON p.id = mp.prerequis_id  
ORDER BY m.nom, p.nom;  
```

### 🎯 Jointures avec agrégations complexes

```sql
-- Analyses multidimensionnelles avec jointures
CREATE VIEW analyse_performance_complete AS  
SELECT  
    -- Dimension Étudiant
    e.id as etudiant_id,
    e.nom || ' ' || e.prenom as etudiant,
    e.niveau,

    -- Dimension Département
    d.nom as departement,

    -- Métriques de performance
    COUNT(DISTINCT i.cours_id) as nb_cours_suivis,
    COUNT(CASE WHEN i.statut = 'actif' THEN 1 END) as nb_cours_actifs,
    COUNT(CASE WHEN i.statut = 'termine' THEN 1 END) as nb_cours_termines,

    -- Moyennes et classements
    AVG(CASE WHEN i.note_finale IS NOT NULL THEN i.note_finale END) as moyenne_generale,

    -- Comparaisons avec les pairs
    AVG(CASE WHEN i.note_finale IS NOT NULL THEN i.note_finale END) -
        (SELECT AVG(i2.note_finale)
         FROM inscriptions i2
         JOIN etudiants e2 ON i2.etudiant_id = e2.id
         WHERE e2.niveau = e.niveau AND i2.note_finale IS NOT NULL) as ecart_niveau,

    -- Progression temporelle
    CASE
        WHEN COUNT(i.id) >= 2 THEN
            (SELECT i1.note_finale FROM inscriptions i1
             WHERE i1.etudiant_id = e.id AND i1.note_finale IS NOT NULL
             ORDER BY i1.date_inscription DESC LIMIT 1) -
            (SELECT i1.note_finale FROM inscriptions i1
             WHERE i1.etudiant_id = e.id AND i1.note_finale IS NOT NULL
             ORDER BY i1.date_inscription ASC LIMIT 1)
        ELSE NULL
    END as progression,

    -- Répartition par département
    COUNT(DISTINCT d.id) as nb_departements_suivis

FROM etudiants e  
LEFT JOIN inscriptions i ON e.id = i.etudiant_id  
LEFT JOIN cours c ON i.cours_id = c.id  
LEFT JOIN professeurs p ON c.professeur_id = p.id  
LEFT JOIN departements d ON p.departement_id = d.id  
GROUP BY e.id, e.nom, e.prenom, e.niveau;  

-- Utilisation de la vue pour analyses avancées
SELECT
    niveau,
    COUNT(*) as nb_etudiants,
    AVG(moyenne_generale) as moyenne_niveau,
    AVG(progression) as progression_moyenne,
    COUNT(CASE WHEN moyenne_generale >= 14 THEN 1 END) as excellents,
    COUNT(CASE WHEN moyenne_generale < 10 THEN 1 END) as en_difficulte
FROM analyse_performance_complete  
WHERE moyenne_generale IS NOT NULL  
GROUP BY niveau  
ORDER BY moyenne_niveau DESC;  
```

## Gestion des relations à grande échelle

### ⚡ Stratégies de partitioning logique

> ⚠️ **SQLite n'a PAS de partitioning natif** (contrairement à PostgreSQL avec `PARTITION BY` ou `INHERITS`). On simule le partitioning manuellement avec des tables séparées + une vue d'unification.

```sql
-- ✅ Approche SQLite : tables distinctes avec CHECK pour valider la plage
CREATE TABLE inscriptions_2024_s1 (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    etudiant_id INTEGER NOT NULL,
    cours_id INTEGER NOT NULL,
    date_inscription TEXT NOT NULL,
    statut TEXT,
    note_finale REAL,
    CHECK (date_inscription >= '2024-01-01' AND date_inscription < '2024-06-01')
);

CREATE TABLE inscriptions_2024_s2 (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    etudiant_id INTEGER NOT NULL,
    cours_id INTEGER NOT NULL,
    date_inscription TEXT NOT NULL,
    statut TEXT,
    note_finale REAL,
    CHECK (date_inscription >= '2024-06-01' AND date_inscription < '2025-01-01')
);

-- Vue unifiée (lecture seule en pratique pour une vue UNION ALL)
CREATE VIEW inscriptions_toutes AS  
SELECT 's1' AS partition, id, etudiant_id, cours_id, date_inscription, statut, note_finale  
FROM inscriptions_2024_s1  
UNION ALL  
SELECT 's2' AS partition, id, etudiant_id, cours_id, date_inscription, statut, note_finale  
FROM inscriptions_2024_s2;  

-- Index spécialisés par partition
CREATE INDEX idx_insc_2024_s1_etudiant ON inscriptions_2024_s1(etudiant_id);  
CREATE INDEX idx_insc_2024_s2_etudiant ON inscriptions_2024_s2(etudiant_id);  

-- Insertion : il faut choisir la bonne table soi-même
-- (ou utiliser un trigger INSTEAD OF INSERT sur la vue — voir 3.4 et 3.5)
INSERT INTO inscriptions_2024_s1 (etudiant_id, cours_id, date_inscription, statut)  
VALUES (1, 1, '2024-02-01', 'actif');  
```

> 💡 **Alternative** : `ATTACH DATABASE` pour mettre chaque partition dans un fichier `.db` séparé — utile pour l'archivage long terme (un fichier par année par exemple).

### 📊 Matérialisation de jointures fréquentes

```sql
-- Table matérialisée pour les jointures coûteuses
CREATE TABLE materialized_etudiant_cours (
    id INTEGER PRIMARY KEY,
    etudiant_id INTEGER,
    etudiant_nom TEXT,
    niveau TEXT,
    cours_id INTEGER,
    cours_nom TEXT,
    professeur_nom TEXT,
    departement_nom TEXT,
    note_finale REAL,
    statut TEXT,
    derniere_maj TEXT DEFAULT (datetime('now')),

    -- Index pour performances
    UNIQUE (etudiant_id, cours_id)
);

-- Procédure de rafraîchissement
CREATE TRIGGER refresh_materialized_view  
AFTER INSERT ON inscriptions  
BEGIN  
    INSERT OR REPLACE INTO materialized_etudiant_cours
    (etudiant_id, etudiant_nom, niveau, cours_id, cours_nom, professeur_nom, departement_nom, note_finale, statut)
    SELECT
        e.id, e.nom || ' ' || e.prenom, e.niveau,
        c.id, c.nom, p.nom || ' ' || p.prenom, d.nom,
        NEW.note_finale, NEW.statut
    FROM etudiants e, cours c, professeurs p, departements d
    WHERE e.id = NEW.etudiant_id
    AND c.id = NEW.cours_id
    AND p.id = c.professeur_id
    AND d.id = p.departement_id;
END;

-- Requêtes ultra-rapides sur la table matérialisée
SELECT departement_nom, COUNT(*), AVG(note_finale)  
FROM materialized_etudiant_cours  
WHERE statut = 'termine'  
GROUP BY departement_nom;  
```

## Debugging et analyse des relations

### 🔍 Outils de diagnostic

```sql
-- Fonction pour analyser l'intégrité référentielle
CREATE VIEW diagnostic_relations AS  
SELECT  
    'Étudiants sans inscription' as probleme,
    COUNT(*) as occurrences
FROM etudiants e  
LEFT JOIN inscriptions i ON e.id = i.etudiant_id  
WHERE i.etudiant_id IS NULL  

UNION ALL

SELECT
    'Cours sans étudiant inscrit',
    COUNT(*)
FROM cours c  
LEFT JOIN inscriptions i ON c.id = i.cours_id  
WHERE i.cours_id IS NULL  

UNION ALL

SELECT
    'Professeurs sans cours',
    COUNT(*)
FROM professeurs p  
LEFT JOIN cours c ON p.id = c.professeur_id  
WHERE c.professeur_id IS NULL  

UNION ALL

SELECT
    'Départements sans professeur',
    COUNT(*)
FROM departements d  
LEFT JOIN professeurs p ON d.id = p.departement_id  
WHERE p.departement_id IS NULL;  

-- Analyse des cardinalités
CREATE VIEW analyse_cardinalites AS  
SELECT  
    'Étudiants' as entite,
    COUNT(*) as total,
    AVG(nb_cours) as moyenne_relations,
    MAX(nb_cours) as max_relations
FROM (
    SELECT e.id, COUNT(i.cours_id) as nb_cours
    FROM etudiants e
    LEFT JOIN inscriptions i ON e.id = i.etudiant_id
    GROUP BY e.id
)

UNION ALL

SELECT
    'Cours',
    COUNT(*),
    AVG(nb_etudiants),
    MAX(nb_etudiants)
FROM (
    SELECT c.id, COUNT(i.etudiant_id) as nb_etudiants
    FROM cours c
    LEFT JOIN inscriptions i ON c.id = i.cours_id
    GROUP BY c.id
);

-- Détection d'anomalies
SELECT
    'Inscriptions multiples même cours' as anomalie,
    COUNT(*) as occurrences
FROM (
    SELECT etudiant_id, cours_id, COUNT(*) as nb
    FROM inscriptions
    GROUP BY etudiant_id, cours_id
    HAVING nb > 1
);
```

### 📈 Métriques de performance des relations

```sql
-- Table pour stocker les métriques de performance
CREATE TABLE metriques_performance (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom_requete TEXT,
    duree_ms REAL,
    nb_lignes_retournees INTEGER,
    plan_execution TEXT,
    timestamp TEXT DEFAULT (datetime('now'))
);

-- Procédure de benchmark automatique
CREATE VIEW benchmark_jointures AS  
SELECT  
    'Jointure simple (étudiant-inscription)' as type_requete,
    COUNT(*) as nb_resultats,
    'Performance baseline' as commentaire
FROM etudiants e  
JOIN inscriptions i ON e.id = i.etudiant_id  

UNION ALL

SELECT
    'Jointure complexe (5 tables)',
    COUNT(*),
    'Jointure complète avec tous les détails'
FROM etudiants e  
JOIN inscriptions i ON e.id = i.etudiant_id  
JOIN cours c ON i.cours_id = c.id  
JOIN professeurs p ON c.professeur_id = p.id  
JOIN departements d ON p.departement_id = d.id;  
```

## Cas d'usage réels et best practices

### 🎯 Système de recommandations basé sur les relations

```sql
-- Recommandation de cours basée sur les relations
-- Algorithme : pour chaque étudiant i1, on cherche les autres étudiants i2
-- qui ont suivi au moins un cours en commun, puis on récupère les AUTRES cours
-- (i3) suivis par ces étudiants i2 que i1 n'a pas encore suivis.
CREATE VIEW recommandations_cours AS  
WITH cours_similaires AS (  
    SELECT
        i1.etudiant_id,
        i3.cours_id AS cours_recommande,
        COUNT(*) AS score_similarite
    FROM inscriptions i1
    JOIN inscriptions i2
        ON i1.cours_id = i2.cours_id
        AND i1.etudiant_id != i2.etudiant_id
    JOIN inscriptions i3
        ON i2.etudiant_id = i3.etudiant_id
    WHERE i1.statut IN ('termine', 'actif')
    AND i2.statut IN ('termine', 'actif')
    AND i3.statut IN ('termine', 'actif')
    AND i3.cours_id NOT IN (
        SELECT cours_id FROM inscriptions WHERE etudiant_id = i1.etudiant_id
    )
    GROUP BY i1.etudiant_id, i3.cours_id
    HAVING score_similarite >= 2
)
SELECT
    e.nom || ' ' || e.prenom as etudiant,
    c.nom as cours_recommande,
    c.code,
    cs.score_similarite,
    p.nom || ' ' || p.prenom as professeur
FROM cours_similaires cs  
JOIN etudiants e ON cs.etudiant_id = e.id  
JOIN cours c ON cs.cours_recommande = c.id  
JOIN professeurs p ON c.professeur_id = p.id  
ORDER BY cs.score_similarite DESC;  
```

### 🏆 Système de badges et récompenses

```sql
-- Table des achievements basés sur les relations
CREATE TABLE badges (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE,
    description TEXT,
    condition_sql TEXT,
    points INTEGER DEFAULT 10
);

CREATE TABLE etudiants_badges (
    etudiant_id INTEGER,
    badge_id INTEGER,
    date_obtention TEXT DEFAULT (datetime('now')),
    PRIMARY KEY (etudiant_id, badge_id),
    FOREIGN KEY (etudiant_id) REFERENCES etudiants(id),
    FOREIGN KEY (badge_id) REFERENCES badges(id)
);

-- Définition des badges
INSERT INTO badges VALUES
    (1, 'Premier cours', 'Premier cours suivi avec succès', 'note_finale >= 10', 10),
    (2, 'Excellente moyenne', 'Moyenne générale >= 16', 'moyenne >= 16', 50),
    (3, 'Touche-à-tout', 'Cours suivis dans 3 départements différents', 'nb_departements >= 3', 30),
    (4, 'Assidu', '5 cours terminés', 'nb_cours_termines >= 5', 25);

-- Trigger pour attribution automatique des badges
CREATE TRIGGER attribuer_badges  
AFTER UPDATE ON inscriptions  
WHEN NEW.statut = 'termine' AND NEW.note_finale IS NOT NULL  
BEGIN  
    -- Badge "Premier cours"
    INSERT OR IGNORE INTO etudiants_badges (etudiant_id, badge_id)
    SELECT NEW.etudiant_id, 1
    WHERE NEW.note_finale >= 10
    AND NOT EXISTS (
        SELECT 1 FROM etudiants_badges
        WHERE etudiant_id = NEW.etudiant_id AND badge_id = 1
    );

    -- Badge "Excellente moyenne"
    INSERT OR IGNORE INTO etudiants_badges (etudiant_id, badge_id)
    SELECT NEW.etudiant_id, 2
    WHERE (
        SELECT AVG(note_finale)
        FROM inscriptions
        WHERE etudiant_id = NEW.etudiant_id AND note_finale IS NOT NULL
    ) >= 16;
END;

-- Vue des accomplissements
CREATE VIEW tableau_recompenses AS  
SELECT  
    e.nom || ' ' || e.prenom as etudiant,
    COUNT(eb.badge_id) as nb_badges,
    SUM(b.points) as total_points,
    GROUP_CONCAT(b.nom, ', ') as badges_obtenus
FROM etudiants e  
LEFT JOIN etudiants_badges eb ON e.id = eb.etudiant_id  
LEFT JOIN badges b ON eb.badge_id = b.id  
GROUP BY e.id  
ORDER BY total_points DESC;  
```

## Récapitulatif et bonnes pratiques

### ✅ Check-list des relations bien conçues

#### 🎯 Conception
- [ ] **Type de relation approprié** (1:1, 1:N, N:M)
- [ ] **Clés étrangères** dans les bonnes tables
- [ ] **Contraintes référentielles** activées (`PRAGMA foreign_keys = ON`)
- [ ] **Actions ON DELETE/UPDATE** définies selon les besoins métier

#### 🔧 Implémentation
- [ ] **Index** sur toutes les clés étrangères
- [ ] **Noms cohérents** pour les colonnes de liaison
- [ ] **Tables de jonction** bien conçues pour les N:M
- [ ] **Contraintes UNIQUE** appropriées

#### ⚡ Performance
- [ ] **Index composites** pour les requêtes fréquentes
- [ ] **Vues matérialisées** pour les jointures coûteuses
- [ ] **EXPLAIN QUERY PLAN** pour optimiser
- [ ] **Pagination** pour les gros résultats

#### 🛡️ Intégrité
- [ ] **Validation** des transitions d'état
- [ ] **Audit trail** pour les changements critiques
- [ ] **Tests d'intégrité** automatisés
- [ ] **Diagnostic** des anomalies

### 🎯 Patterns à retenir

**1:1 → Utilisation parcimonieuse**
- Données confidentielles à isoler
- Optimisation performance (colonnes rares)
- Extension optionnelle d'entités

**1:N → Le plus courant**
- Clé étrangère du côté "Many"
- Index obligatoire sur la FK
- Actions CASCADE bien réfléchies

**N:M → Table de jonction**
- Toujours une table intermédiaire
- Clé composite sur les deux FK
- Attributs de relation possibles

**Auto-référence → Hiérarchies**
- CTE récursives pour les parcours
- Attention aux cycles infinis
- Niveaux pour limiter la profondeur

---

**💡 Dans le prochain chapitre**, nous explorerons la gestion avancée des clés étrangères et contraintes référentielles, en nous appuyant sur ces relations bien conçues pour garantir l'intégrité des données.

**🎯 Vous maîtrisez maintenant** l'art de connecter vos données avec des relations robustes, performantes et évolutives !

⏭️ [3.3 Gestion des clés étrangères et contraintes référentielles](/03-conception-modelisation-avancee/03-gestion-cles-etrangeres-contraintes-referentielles.md)
