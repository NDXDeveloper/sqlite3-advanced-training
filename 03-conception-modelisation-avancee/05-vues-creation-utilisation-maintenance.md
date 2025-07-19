🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.5 - Vues : création, utilisation et maintenance

## Qu'est-ce qu'une vue ?

Une **vue** (view en anglais) est une "table virtuelle" qui présente les données d'une ou plusieurs tables réelles sous une forme particulière. Elle ne stocke pas de données elle-même, mais affiche le résultat d'une requête SQL préalablement définie.

### Analogie simple
Imaginez une vue comme une **vitrine de magasin** :
- La vitrine (vue) montre seulement une sélection des produits du magasin
- Les vrais produits sont stockés dans l'entrepôt (tables)
- Vous pouvez changer l'arrangement de la vitrine sans modifier l'entrepôt
- Plusieurs vitrines peuvent montrer les mêmes produits sous différents angles

## Pourquoi utiliser les vues ?

Les vues offrent plusieurs avantages :

### 1. **Simplification des requêtes complexes**
Au lieu de réécrire une longue requête avec jointures, vous l'encapsulez dans une vue.

### 2. **Sécurité et contrôle d'accès**
Vous pouvez montrer seulement certaines colonnes ou certaines lignes aux utilisateurs.

### 3. **Abstraction des données**
Cache la complexité de la structure des tables aux utilisateurs finaux.

### 4. **Réutilisabilité**
Une fois créée, la vue peut être utilisée comme une table normale dans d'autres requêtes.

### 5. **Maintien de la compatibilité**
Si la structure des tables change, les vues peuvent maintenir une interface stable.

## Types de vues

### Vues simples
Basées sur une seule table, généralement modifiables.

### Vues complexes
Basées sur plusieurs tables avec jointures, groupements, ou fonctions d'agrégation. Généralement non modifiables directement.

## Création d'une vue

### Syntaxe de base
```sql
CREATE VIEW nom_de_la_vue AS
SELECT colonnes
FROM table(s)
WHERE conditions;
```

### Syntaxe complète
```sql
CREATE [TEMP|TEMPORARY] VIEW [IF NOT EXISTS] nom_de_la_vue [(colonnes)]
AS SELECT requête;
```

## Exemples pratiques pour débutants

### Préparation : Créons des tables d'exemple

```sql
-- Table des employés
CREATE TABLE employes (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    email TEXT UNIQUE,
    salaire REAL,
    departement_id INTEGER,
    date_embauche DATE,
    actif BOOLEAN DEFAULT 1
);

-- Table des départements
CREATE TABLE departements (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    budget REAL,
    responsable_id INTEGER
);

-- Table des projets
CREATE TABLE projets (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    budget REAL,
    date_debut DATE,
    date_fin DATE,
    statut TEXT DEFAULT 'En cours'
);

-- Table d'association employé-projet
CREATE TABLE employe_projet (
    employe_id INTEGER,
    projet_id INTEGER,
    role TEXT,
    heures_allouees INTEGER,
    PRIMARY KEY (employe_id, projet_id)
);
```

### Insérons quelques données de test

```sql
INSERT INTO departements VALUES
    (1, 'Informatique', 100000, NULL),
    (2, 'Marketing', 75000, NULL),
    (3, 'Ressources Humaines', 50000, NULL);

INSERT INTO employes VALUES
    (1, 'Dupont', 'Jean', 'jean.dupont@email.com', 45000, 1, '2022-01-15', 1),
    (2, 'Martin', 'Marie', 'marie.martin@email.com', 52000, 1, '2021-03-10', 1),
    (3, 'Bernard', 'Paul', 'paul.bernard@email.com', 38000, 2, '2023-05-20', 1),
    (4, 'Dubois', 'Sophie', 'sophie.dubois@email.com', 41000, 3, '2022-09-01', 0);

INSERT INTO projets VALUES
    (1, 'Site Web E-commerce', 50000, '2024-01-01', '2024-06-30', 'En cours'),
    (2, 'Application Mobile', 80000, '2024-02-15', '2024-12-31', 'En cours'),
    (3, 'Refonte Intranet', 30000, '2023-10-01', '2024-03-31', 'Terminé');
```

## Exemples de vues simples

### Exemple 1 : Vue des employés actifs

```sql
CREATE VIEW employes_actifs AS
SELECT id, nom, prenom, email, salaire, departement_id
FROM employes
WHERE actif = 1;
```

**Utilisation :**
```sql
-- Utiliser la vue comme une table normale
SELECT * FROM employes_actifs;

-- Filtrer sur la vue
SELECT nom, prenom, salaire
FROM employes_actifs
WHERE salaire > 40000;
```

### Exemple 2 : Vue avec calculs

```sql
CREATE VIEW salaires_avec_primes AS
SELECT
    id,
    nom,
    prenom,
    salaire,
    salaire * 0.1 AS prime_annuelle,
    salaire + (salaire * 0.1) AS salaire_total,
    CASE
        WHEN salaire > 50000 THEN 'Élevé'
        WHEN salaire > 40000 THEN 'Moyen'
        ELSE 'Bas'
    END AS niveau_salaire
FROM employes
WHERE actif = 1;
```

## Exemples de vues complexes avec jointures

### Exemple 3 : Vue employés avec département

```sql
CREATE VIEW employes_complet AS
SELECT
    e.id,
    e.nom,
    e.prenom,
    e.email,
    e.salaire,
    d.nom AS nom_departement,
    d.budget AS budget_departement,
    e.date_embauche
FROM employes e
LEFT JOIN departements d ON e.departement_id = d.id
WHERE e.actif = 1;
```

**Utilisation :**
```sql
-- Voir tous les employés avec leur département
SELECT nom, prenom, nom_departement FROM employes_complet;

-- Filtrer par département
SELECT * FROM employes_complet WHERE nom_departement = 'Informatique';
```

### Exemple 4 : Vue avec agrégations

```sql
CREATE VIEW statistiques_departements AS
SELECT
    d.nom AS departement,
    COUNT(e.id) AS nombre_employes,
    AVG(e.salaire) AS salaire_moyen,
    MIN(e.salaire) AS salaire_min,
    MAX(e.salaire) AS salaire_max,
    SUM(e.salaire) AS masse_salariale,
    d.budget,
    ROUND((SUM(e.salaire) / d.budget) * 100, 2) AS pourcentage_budget
FROM departements d
LEFT JOIN employes e ON d.id = e.departement_id AND e.actif = 1
GROUP BY d.id, d.nom, d.budget;
```

## Vues avec colonnes nommées

Vous pouvez spécifier les noms des colonnes de la vue :

```sql
CREATE VIEW resume_employes (identifiant, nom_complet, remuneration, service) AS
SELECT
    id,
    nom || ' ' || prenom,
    salaire,
    departement_id
FROM employes
WHERE actif = 1;
```

## Vues temporaires

Les vues temporaires existent seulement pendant la session :

```sql
CREATE TEMP VIEW employes_temp AS
SELECT * FROM employes WHERE date_embauche > '2023-01-01';
```

## Modification des données à travers les vues

### Vues modifiables
Les vues **simples** (basées sur une seule table, sans agrégations) sont modifiables :

```sql
-- Cette vue est modifiable
CREATE VIEW employes_info AS
SELECT id, nom, prenom, email, salaire
FROM employes;

-- Vous pouvez faire des INSERT, UPDATE, DELETE
UPDATE employes_info SET salaire = salaire * 1.05 WHERE id = 1;
```

### Vues non modifiables
Les vues **complexes** ne sont généralement pas modifiables directement :

```sql
-- Cette vue n'est PAS modifiable (jointure et agrégation)
CREATE VIEW stats_complexe AS
SELECT d.nom, COUNT(e.id) as nb_employes
FROM departements d
LEFT JOIN employes e ON d.id = e.departement_id
GROUP BY d.id, d.nom;
```

### Rendre les vues complexes modifiables avec INSTEAD OF

```sql
CREATE VIEW vue_employe_departement AS
SELECT
    e.id as employe_id,
    e.nom,
    e.prenom,
    d.nom as departement_nom
FROM employes e
JOIN departements d ON e.departement_id = d.id;

-- Trigger pour permettre les modifications
CREATE TRIGGER update_vue_employe_departement
    INSTEAD OF UPDATE ON vue_employe_departement
BEGIN
    UPDATE employes
    SET nom = NEW.nom, prenom = NEW.prenom
    WHERE id = NEW.employe_id;
END;
```

## Gestion et maintenance des vues

### Lister toutes les vues

```sql
-- Voir toutes les vues
SELECT name FROM sqlite_master WHERE type = 'view';

-- Voir les vues avec leur définition
SELECT name, sql FROM sqlite_master WHERE type = 'view';
```

### Voir la définition d'une vue

```sql
SELECT sql FROM sqlite_master WHERE type = 'view' AND name = 'employes_actifs';
```

### Modifier une vue

SQLite ne supporte pas `ALTER VIEW`. Il faut supprimer et recréer :

```sql
-- Supprimer l'ancienne vue
DROP VIEW IF EXISTS employes_actifs;

-- Recréer avec la nouvelle définition
CREATE VIEW employes_actifs AS
SELECT id, nom, prenom, email, salaire, departement_id, date_embauche
FROM employes
WHERE actif = 1;
```

### Supprimer une vue

```sql
DROP VIEW IF EXISTS nom_de_la_vue;
```

## Cas d'usage avancés

### Vue pour les rapports

```sql
CREATE VIEW rapport_mensuel AS
SELECT
    strftime('%Y-%m', date_embauche) AS mois_embauche,
    COUNT(*) AS nouveaux_employes,
    AVG(salaire) AS salaire_moyen_nouveaux
FROM employes
WHERE actif = 1
GROUP BY strftime('%Y-%m', date_embauche)
ORDER BY mois_embauche DESC;
```

### Vue pour la sécurité (masquer des colonnes sensibles)

```sql
CREATE VIEW employes_public AS
SELECT
    id,
    nom,
    prenom,
    departement_id,
    CASE
        WHEN salaire > 50000 THEN 'Élevé'
        WHEN salaire > 40000 THEN 'Moyen'
        ELSE 'Standard'
    END AS niveau_salaire
FROM employes
WHERE actif = 1;
-- Le salaire exact n'est pas visible
```

### Vue récursive (pour les hiérarchies)

```sql
-- Table hiérarchique d'exemple
CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    parent_id INTEGER
);

INSERT INTO categories VALUES
    (1, 'Électronique', NULL),
    (2, 'Ordinateurs', 1),
    (3, 'Smartphones', 1),
    (4, 'Laptops', 2),
    (5, 'Desktop', 2);

-- Vue récursive pour afficher la hiérarchie
CREATE VIEW hierarchie_categories AS
WITH RECURSIVE cat_hierarchie(id, nom, parent_id, niveau, chemin) AS (
    -- Cas de base : catégories racines
    SELECT id, nom, parent_id, 0, nom
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Cas récursif : sous-catégories
    SELECT c.id, c.nom, c.parent_id, h.niveau + 1, h.chemin || ' > ' || c.nom
    FROM categories c
    JOIN cat_hierarchie h ON c.parent_id = h.id
)
SELECT * FROM cat_hierarchie ORDER BY chemin;
```

## Bonnes pratiques

### 1. Nommage cohérent
```sql
-- Préfixer les vues pour les identifier facilement
CREATE VIEW v_employes_actifs AS ...
CREATE VIEW vue_rapport_mensuel AS ...
```

### 2. Documenter les vues complexes
```sql
-- Vue pour le tableau de bord RH
-- Affiche les statistiques par département avec calculs de performance
CREATE VIEW v_dashboard_rh AS
SELECT
    -- Colonnes et calculs...
```

### 3. Optimisation des performances
```sql
-- Créer des index sur les colonnes utilisées dans les vues fréquemment consultées
CREATE INDEX idx_employes_departement ON employes(departement_id);
CREATE INDEX idx_employes_actif ON employes(actif);
```

### 4. Vues avec paramètres simulés
```sql
-- Vue pour les employés d'un département (utiliser avec WHERE)
CREATE VIEW v_employes_par_dept AS
SELECT
    e.*,
    d.nom as dept_nom
FROM employes e
JOIN departements d ON e.departement_id = d.id
WHERE e.actif = 1;

-- Utilisation : SELECT * FROM v_employes_par_dept WHERE dept_nom = 'Informatique';
```

## Exercices pratiques

### Exercice 1 : Vue simple
Créez une vue qui affiche seulement les employés gagnant plus de 40 000€ avec leurs noms complets (nom + prénom).

### Exercice 2 : Vue avec jointure
Créez une vue qui montre chaque projet avec le nombre d'employés assignés.

### Exercice 3 : Vue d'agrégation
Créez une vue qui affiche pour chaque département : le nombre d'employés, le salaire total, et le salaire moyen.

### Exercice 4 : Vue sécurisée
Créez une vue publique qui masque les informations sensibles (salaires exacts, emails) mais montre les informations générales des employés.

## Avantages et inconvénients

### Avantages
- ✅ Simplification des requêtes complexes
- ✅ Réutilisabilité du code
- ✅ Sécurité et contrôle d'accès
- ✅ Abstraction de la complexité
- ✅ Maintien de la compatibilité

### Inconvénients
- ❌ Performance : les vues complexes peuvent être lentes
- ❌ Pas de stockage de données (recalcul à chaque requête)
- ❌ Limitations sur les modifications
- ❌ Peut masquer la complexité réelle des requêtes

## Optimisation des performances des vues

### 1. Utiliser des vues matérialisées (simulation)
```sql
-- SQLite n'a pas de vues matérialisées natives, mais on peut simuler :
CREATE TABLE cache_statistiques_dept AS
SELECT * FROM statistiques_departements;

-- Recréer périodiquement avec un trigger ou une tâche planifiée
```

### 2. Index appropriés
```sql
-- Créer des index sur les colonnes utilisées dans les jointures et filtres des vues
CREATE INDEX idx_employes_departement_actif ON employes(departement_id, actif);
```

### 3. Limiter la complexité
```sql
-- Éviter les vues trop complexes, préférer plusieurs vues simples
CREATE VIEW v_employes_base AS
SELECT id, nom, prenom, departement_id FROM employes WHERE actif = 1;

CREATE VIEW v_employes_avec_dept AS
SELECT e.*, d.nom as dept_nom
FROM v_employes_base e
JOIN departements d ON e.departement_id = d.id;
```

## Résumé

Les vues SQLite sont un outil puissant pour simplifier l'accès aux données et améliorer la sécurité. Elles agissent comme des "tables virtuelles" qui présentent les données sous une forme adaptée aux besoins spécifiques.

**Points clés à retenir :**
- Les vues ne stockent pas de données, elles affichent le résultat de requêtes
- Les vues simples sont modifiables, les vues complexes généralement non
- Elles améliorent la réutilisabilité et la sécurité
- Attention aux performances avec les vues complexes
- Utilisez `INSTEAD OF` triggers pour rendre modifiables les vues complexes

**Syntaxe à mémoriser :**
```sql
CREATE VIEW nom_vue AS SELECT ...;
DROP VIEW nom_vue;
SELECT name FROM sqlite_master WHERE type = 'view';
```

Dans le prochain chapitre, nous aborderons les requêtes avancées et les techniques d'optimisation.

⏭️
