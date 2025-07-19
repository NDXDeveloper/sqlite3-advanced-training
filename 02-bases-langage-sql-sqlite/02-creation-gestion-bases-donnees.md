🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.2 Création et gestion des bases de données

## Introduction - Au cœur de SQLite

Contrairement aux autres SGBD où créer une base de données peut être complexe, SQLite rend cette tâche **incroyablement simple**. Dans ce chapitre, nous allons explorer toutes les facettes de la création et gestion des bases de données SQLite, des opérations les plus basiques aux techniques avancées.

> **Rappel important** : Dans SQLite, une base de données = un fichier. Cette simplicité cache une richesse de possibilités que nous allons découvrir ensemble !

## Création d'une base de données

### 🎯 Méthode de base - Création automatique

La façon la plus simple de créer une base SQLite :

```bash
# Créer ou ouvrir une base de données
sqlite3 ma_nouvelle_base.db
```

**Ce qui se passe :**
- Si le fichier n'existe pas → SQLite le crée automatiquement
- Si le fichier existe → SQLite l'ouvre
- Le fichier reste vide jusqu'à la première opération

### 📁 Démonstration pratique

```bash
# Vérifier qu'aucune base n'existe
ls *.db
# (aucun fichier .db)

# Créer une nouvelle base
sqlite3 bibliotheque.db

# Dans SQLite, vérifier qu'on est bien connecté
sqlite> .databases
main: /chemin/vers/bibliotheque.db

# Quitter sans rien faire
sqlite> .quit

# Vérifier ce qui s'est passé
ls -la bibliotheque.db
# Le fichier existe maintenant (0 bytes ou très petit)
```

### 🔍 Anatomie d'une nouvelle base

```sql
-- Se connecter à la base
sqlite3 bibliotheque.db

-- Voir les informations de la base
.databases

-- Voir les tables (aucune pour l'instant)
.tables

-- Voir le schéma complet (vide)
.schema

-- Informations sur le fichier
PRAGMA database_list;
```

## Organisation des bases de données

### 📂 Conventions de nommage

**Bonnes pratiques pour nommer vos bases :**

```bash
# ✅ Noms descriptifs et explicites
sqlite3 gestion_bibliotheque.db
sqlite3 comptabilite_2024.db
sqlite3 inventaire_magasin.db

# ✅ Sans espaces ni caractères spéciaux
sqlite3 analyse_donnees.db
sqlite3 cache_application.db

# ❌ À éviter
sqlite3 "ma base.db"          # Espaces problématiques
sqlite3 données_été.db        # Caractères accentués
sqlite3 base#1.db            # Caractères spéciaux
```

### 🗂️ Organisation par projet

**Structure recommandée :**

```bash
# Organisation par dossiers
projet_web/
├── backend/
│   ├── production.db      # Base de production
│   ├── development.db     # Base de développement
│   └── test.db           # Base de tests
├── backup/
│   ├── production_2024-01-15.db
│   └── production_2024-01-14.db
└── scripts/
    ├── schema.sql        # Définition des tables
    ├── data_sample.sql   # Données d'exemple
    └── migration.sql     # Scripts de migration
```

## Gestion des métadonnées

### 📊 Informations sur la base de données

SQLite stocke des informations sur lui-même que vous pouvez consulter :

```sql
-- Informations générales
PRAGMA database_list;

-- Configuration actuelle
PRAGMA journal_mode;        -- Mode de journal (DELETE, WAL, etc.)
PRAGMA synchronous;         -- Niveau de synchronisation
PRAGMA page_size;           -- Taille des pages en bytes
PRAGMA page_count;          -- Nombre de pages
PRAGMA freelist_count;      -- Pages libres

-- Taille de la base
SELECT page_count * page_size as taille_bytes
FROM pragma_page_count(), pragma_page_size();
```

### 🔧 Configuration de base

**Optimisations recommandées pour une nouvelle base :**

```sql
-- Activer le mode WAL pour de meilleures performances
PRAGMA journal_mode = WAL;

-- Optimiser les performances
PRAGMA synchronous = NORMAL;

-- Définir un timeout pour les verrous
PRAGMA busy_timeout = 30000;  -- 30 secondes

-- Vérifier la configuration
PRAGMA journal_mode;
PRAGMA synchronous;
PRAGMA busy_timeout;
```

## Création de structure - Tables et schémas

### 🏗️ Première table dans une nouvelle base

```sql
-- Créer notre base de bibliothèque
sqlite3 bibliotheque.db

-- Créer la première table
CREATE TABLE auteurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    nationalite TEXT,
    date_naissance TEXT,
    date_creation TEXT DEFAULT (datetime('now')),
    actif INTEGER DEFAULT 1
);

-- Vérifier la création
.tables
.schema auteurs

-- Informations détaillées sur la table
PRAGMA table_info(auteurs);
```

### 📋 Structure complète d'exemple

```sql
-- Créer plusieurs tables liées
CREATE TABLE auteurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    biographie TEXT,
    date_creation TEXT DEFAULT (datetime('now'))
);

CREATE TABLE editeurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL UNIQUE,
    adresse TEXT,
    telephone TEXT
);

CREATE TABLE livres (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    isbn TEXT UNIQUE,
    auteur_id INTEGER,
    editeur_id INTEGER,
    date_publication TEXT,
    pages INTEGER CHECK (pages > 0),
    prix REAL CHECK (prix >= 0),
    stock INTEGER DEFAULT 0,
    FOREIGN KEY (auteur_id) REFERENCES auteurs(id),
    FOREIGN KEY (editeur_id) REFERENCES editeurs(id)
);

-- Vérifier la structure complète
.schema
```

## Modification de la structure

### ✏️ Ajouter des éléments

```sql
-- Ajouter une nouvelle table
CREATE TABLE emprunts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    livre_id INTEGER,
    emprunteur TEXT NOT NULL,
    date_emprunt TEXT DEFAULT (datetime('now')),
    date_retour_prevue TEXT,
    date_retour_reel TEXT,
    FOREIGN KEY (livre_id) REFERENCES livres(id)
);

-- Ajouter une colonne à une table existante
ALTER TABLE auteurs ADD COLUMN email TEXT;
ALTER TABLE auteurs ADD COLUMN site_web TEXT;

-- Renommer une colonne (SQLite 3.25+)
ALTER TABLE auteurs RENAME COLUMN date_creation TO created_at;

-- Renommer une table
ALTER TABLE auteurs RENAME TO authors;
ALTER TABLE authors RENAME TO auteurs;  -- On remet le nom original
```

### 🔧 Modifications complexes

SQLite ne permet pas toutes les modifications. Pour des changements complexes :

```sql
-- Exemple : Supprimer une colonne (méthode complète)
BEGIN TRANSACTION;

-- 1. Créer une nouvelle table sans la colonne unwanted
CREATE TABLE auteurs_new (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    nationalite TEXT,
    date_naissance TEXT
    -- email et site_web supprimés
);

-- 2. Copier les données
INSERT INTO auteurs_new (id, nom, prenom, nationalite, date_naissance)
SELECT id, nom, prenom, nationalite, date_naissance FROM auteurs;

-- 3. Supprimer l'ancienne table
DROP TABLE auteurs;

-- 4. Renommer la nouvelle table
ALTER TABLE auteurs_new RENAME TO auteurs;

COMMIT;
```

## Sauvegarde et restauration

### 💾 Sauvegarde simple

```bash
# Méthode 1 : Copie de fichier (base fermée)
cp bibliotheque.db backup_bibliotheque_$(date +%Y%m%d).db

# Méthode 2 : Export SQL
sqlite3 bibliotheque.db .dump > backup_bibliotheque.sql

# Méthode 3 : Backup API (base ouverte)
sqlite3 bibliotheque.db "VACUUM INTO 'backup_clean.db'"
```

### 🔄 Restauration

```bash
# Depuis une copie de fichier
cp backup_bibliotheque_20240115.db bibliotheque_restored.db

# Depuis un dump SQL
sqlite3 nouvelle_bibliotheque.db < backup_bibliotheque.sql

# Vérification de la restauration
sqlite3 nouvelle_bibliotheque.db .tables
sqlite3 nouvelle_bibliotheque.db "SELECT COUNT(*) FROM auteurs;"
```

## Gestion multi-bases

### 🔗 Attacher plusieurs bases

SQLite permet de travailler avec plusieurs bases simultanément :

```sql
-- Ouvrir la base principale
sqlite3 bibliotheque.db

-- Attacher une base de statistiques
ATTACH DATABASE 'stats.db' AS stats;

-- Attacher une base d'archive
ATTACH DATABASE 'archive.db' AS archive;

-- Voir toutes les bases attachées
.databases

-- Utiliser les bases attachées
CREATE TABLE stats.visiteurs (
    date TEXT,
    nombre_visites INTEGER
);

-- Copier des données entre bases
INSERT INTO archive.anciens_livres
SELECT * FROM main.livres WHERE date_publication < '2000-01-01';

-- Détacher les bases
DETACH DATABASE stats;
DETACH DATABASE archive;
```

### 📊 Exemple pratique multi-bases

```sql
-- Base principale : données courantes
sqlite3 production.db

-- Attacher base de logs
ATTACH DATABASE 'logs.db' AS logs;

-- Créer une vue combinée
CREATE VIEW rapport_complet AS
SELECT
    l.titre,
    l.auteur_id,
    COUNT(logs.acces.livre_id) as consultations
FROM main.livres l
LEFT JOIN logs.acces ON l.id = logs.acces.livre_id
GROUP BY l.id;

-- Utiliser la vue
SELECT * FROM rapport_complet WHERE consultations > 10;
```

## Maintenance et optimisation

### 🧹 Nettoyage de la base

```sql
-- Réorganiser et optimiser la base
VACUUM;

-- Analyser les statistiques pour l'optimiseur
ANALYZE;

-- Vérifier l'intégrité
PRAGMA integrity_check;

-- Voir l'espace utilisé
PRAGMA page_count;
PRAGMA freelist_count;

-- Statistiques détaillées
SELECT
    name,
    tbl_name,
    type,
    sql
FROM sqlite_master;
```

### ⚡ Optimisations courantes

```sql
-- Configuration pour de meilleures performances
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = 10000;
PRAGMA temp_store = memory;

-- Index automatiques (attention aux performances)
PRAGMA automatic_index = ON;

-- Voir la configuration actuelle
PRAGMA compile_options;
```

## Gestion des erreurs et récupération

### 🚨 Problèmes courants et solutions

**Base corrompue :**
```sql
-- Diagnostic
PRAGMA integrity_check;

-- Tentative de récupération
PRAGMA quick_check;

-- Récupération par dump/restore si nécessaire
.output recovery.sql
.dump
.quit

sqlite3 recovered.db < recovery.sql
```

**Base verrouillée :**
```sql
-- Vérifier les connexions actives
.databases

-- Forcer la fermeture des connexions
PRAGMA busy_timeout = 0;

-- Redémarrer avec timeout plus long
PRAGMA busy_timeout = 30000;
```

## 🎯 Exercice pratique complet

### Objectif : Créer une base de données de gestion de projet

```sql
-- 1. Créer la base
sqlite3 gestion_projets.db

-- 2. Configuration optimale
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 30000;

-- 3. Créer la structure
CREATE TABLE projets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL UNIQUE,
    description TEXT,
    date_debut TEXT,
    date_fin_prevue TEXT,
    statut TEXT DEFAULT 'en_cours' CHECK (statut IN ('planifie', 'en_cours', 'termine', 'annule')),
    budget REAL CHECK (budget >= 0),
    created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE employes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    poste TEXT,
    email TEXT UNIQUE,
    salaire REAL CHECK (salaire > 0)
);

CREATE TABLE assignations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    projet_id INTEGER,
    employe_id INTEGER,
    role TEXT,
    date_assignation TEXT DEFAULT (datetime('now')),
    heures_prevues REAL,
    heures_realisees REAL DEFAULT 0,
    FOREIGN KEY (projet_id) REFERENCES projets(id) ON DELETE CASCADE,
    FOREIGN KEY (employe_id) REFERENCES employes(id),
    UNIQUE(projet_id, employe_id, role)
);

-- 4. Ajouter des données d'exemple
INSERT INTO projets (nom, description, date_debut, date_fin_prevue, budget) VALUES
    ('Site Web E-commerce', 'Développement boutique en ligne', '2024-01-15', '2024-04-15', 50000),
    ('App Mobile', 'Application mobile iOS/Android', '2024-02-01', '2024-06-01', 75000),
    ('Migration Base', 'Migration vers nouveau serveur', '2024-03-01', '2024-03-31', 15000);

INSERT INTO employes (nom, prenom, poste, email, salaire) VALUES
    ('Dupont', 'Jean', 'Développeur Senior', 'j.dupont@entreprise.com', 4500),
    ('Martin', 'Sarah', 'Chef de Projet', 's.martin@entreprise.com', 5200),
    ('Dubois', 'Pierre', 'Designer', 'p.dubois@entreprise.com', 3800),
    ('Leroy', 'Marie', 'Développeur Junior', 'm.leroy@entreprise.com', 3200);

INSERT INTO assignations (projet_id, employe_id, role, heures_prevues) VALUES
    (1, 1, 'Développeur Lead', 200),
    (1, 2, 'Chef de Projet', 100),
    (1, 3, 'Designer', 80),
    (2, 1, 'Développeur Senior', 150),
    (2, 4, 'Développeur Junior', 120),
    (3, 1, 'Expert Technique', 60),
    (3, 2, 'Coordinateur', 40);

-- 5. Vérifications
.tables
.schema

SELECT COUNT(*) as nb_projets FROM projets;
SELECT COUNT(*) as nb_employes FROM employes;
SELECT COUNT(*) as nb_assignations FROM assignations;

-- 6. Sauvegarde
.backup backup_gestion_projets.db

-- 7. Test de requête complexe
SELECT
    p.nom as projet,
    e.nom || ' ' || e.prenom as employe,
    a.role,
    a.heures_prevues,
    p.budget / (SELECT COUNT(*) FROM assignations WHERE projet_id = p.id) as budget_par_personne
FROM projets p
JOIN assignations a ON p.id = a.projet_id
JOIN employes e ON a.employe_id = e.id
ORDER BY p.nom, a.role;
```

## Récapitulatif des bonnes pratiques

### ✅ À faire systématiquement

1. **Nommage cohérent** : Utilisez des conventions claires
2. **Configuration optimale** : WAL mode + timeouts appropriés
3. **Sauvegarde régulière** : Automatisez vos backups
4. **Vérification d'intégrité** : PRAGMA integrity_check périodique
5. **Documentation** : Commentez vos schémas et scripts

### ❌ À éviter absolument

1. **Modifications directes** du fichier .db en binaire
2. **Partage réseau** du fichier de base
3. **Noms avec espaces** ou caractères spéciaux
4. **Modifications concurrentes** sans gestion des verrous
5. **Bases trop volumineuses** sans maintenance

### 🎯 Points clés à retenir

- **Simplicité** : Une base SQLite = un fichier
- **Flexibilité** : Modification de structure possible mais limitée
- **Performance** : Configuration appropriée essentielle
- **Fiabilité** : Sauvegardes et vérifications régulières
- **Évolutivité** : Planifiez la croissance dès le départ

---

**💡 Dans le prochain chapitre**, nous explorerons les opérations CRUD (Create, Read, Update, Delete) qui constituent le cœur de toute interaction avec une base de données SQLite.

**🎯 Vous maîtrisez maintenant** :
- La création et configuration de bases SQLite
- La gestion des métadonnées et optimisations
- Les techniques de sauvegarde et restauration
- L'organisation multi-bases et la maintenance

⏭️
