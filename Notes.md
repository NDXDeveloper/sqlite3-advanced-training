# SQLite3 - Notes Essentielles : Concepts et Syntaxes

## 1. Fondamentaux de SQLite3

### Concepts incontournables
- **Architecture serverless** : pas de processus serveur séparé
- **Base de données = 1 fichier** (.db, .sqlite, .sqlite3)
- **Typage dynamique** : les colonnes peuvent stocker différents types
- **ACID compliant** malgré sa simplicité
- **Limites** : pas de gestion multi-utilisateurs intensifs, pas de procédures stockées

### Syntaxes critiques
```sql
-- Ouvrir/créer une base
.open database.db

-- Informations système
.databases
.tables
.schema table_name
.quit
```

## 2. Bases du langage SQL

### Concepts incontournables
- **5 types de stockage** : NULL, INTEGER, REAL, TEXT, BLOB
- **Affinité de type** : conversion automatique selon le contexte
- **Contraintes** : assurent l'intégrité des données

### Syntaxes critiques
```sql
-- Création de table
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    age INTEGER CHECK(age >= 0),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- CRUD de base
INSERT INTO users (name, email, age) VALUES ('John', 'john@email.com', 25);
SELECT * FROM users WHERE age > 18 ORDER BY name;
UPDATE users SET age = 26 WHERE id = 1;
DELETE FROM users WHERE id = 1;

-- Requêtes groupées
SELECT COUNT(*), AVG(age) FROM users GROUP BY department HAVING COUNT(*) > 5;
```

## 3. Conception et modélisation avancée

### Concepts incontournables
- **Normalisation** : éliminer la redondance (1NF→2NF→3NF)
- **Clés étrangères** : doivent être activées manuellement dans SQLite
- **Triggers** : code exécuté automatiquement sur événements
- **Vues** : requêtes stockées comme tables virtuelles

### Syntaxes critiques
```sql
-- Activation des clés étrangères
PRAGMA foreign_keys = ON;

-- Clé étrangère
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Trigger
CREATE TRIGGER update_timestamp
AFTER UPDATE ON users
BEGIN
    UPDATE users SET updated_at = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;

-- Vue
CREATE VIEW active_users AS
SELECT * FROM users WHERE status = 'active';
```

## 4. Requêtes avancées et optimisation

### Concepts incontournables
- **Sous-requêtes** : requêtes imbriquées (corrélées vs non-corrélées)
- **CTE** : expressions de table communes, plus lisibles que sous-requêtes
- **Window functions** : calculs sur ensembles de lignes liées
- **JSON** : SQLite supporte nativement les opérations JSON

### Syntaxes critiques
```sql
-- Sous-requête
SELECT * FROM users WHERE age > (SELECT AVG(age) FROM users);

-- CTE
WITH user_stats AS (
    SELECT department, COUNT(*) as count, AVG(salary) as avg_salary
    FROM employees GROUP BY department
)
SELECT * FROM user_stats WHERE count > 10;

-- Window function
SELECT name, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees;

-- JSON
SELECT json_extract(data, '$.name') FROM products WHERE json_valid(data);
```

## 5. Optimisation des performances

### Concepts incontournables
- **Index** : structure pour accélérer les recherches
- **EXPLAIN QUERY PLAN** : analyse du plan d'exécution
- **PRAGMA** : paramètres de configuration SQLite
- **Statistiques** : SQLite collecte des stats pour optimiser

### Syntaxes critiques
```sql
-- Index
CREATE INDEX idx_user_email ON users(email);
CREATE UNIQUE INDEX idx_user_name_dept ON users(name, department);

-- Analyse de performance
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = 'test@email.com';

-- PRAGMA essentiels
PRAGMA table_info(users);
PRAGMA index_list(users);
PRAGMA cache_size = 10000;
PRAGMA journal_mode = WAL;
```

## 6. Programmation avancée

### Concepts incontournables
- **Transactions** : ensemble d'opérations atomiques
- **Niveaux d'isolation** : READ UNCOMMITTED, SERIALIZABLE
- **UDF** : fonctions personnalisées (via langages de programmation)
- **FTS5** : recherche plein texte intégrée

### Syntaxes critiques
```sql
-- Transaction
BEGIN TRANSACTION;
    INSERT INTO users VALUES (...);
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- ou ROLLBACK en cas d'erreur

-- FTS5
CREATE VIRTUAL TABLE documents_fts USING fts5(title, content);
SELECT * FROM documents_fts WHERE documents_fts MATCH 'sqlite AND performance';

-- Sauvegarde
.backup backup.db
.restore backup.db
```

## 7. Intégration et APIs

### Concepts incontournables
- **Connexions** : gestion du pool de connexions
- **Prepared statements** : sécurité et performance
- **ORM vs SQL natif** : avantages/inconvénients
- **Sérialisation** : gestion des accès concurrents

### Syntaxes critiques (Python)
```python
import sqlite3

# Connexion et cursor
conn = sqlite3.connect('database.db')
cursor = conn.cursor()

# Prepared statement (sécurisé)
cursor.execute("SELECT * FROM users WHERE email = ?", (email,))

# Transaction context manager
with conn:
    cursor.execute("INSERT INTO users VALUES (?, ?)", (name, email))
```

## 8. Sécurité et administration

### Concepts incontournables
- **Injection SQL** : utiliser des paramètres liés
- **Chiffrement** : SQLCipher pour bases chiffrées
- **Permissions fichier** : sécurité au niveau OS
- **Audit trail** : traçabilité des modifications

### Syntaxes critiques
```sql
-- Sécurisation des requêtes (paramètres liés)
-- ✅ Correct
SELECT * FROM users WHERE id = ?;

-- ❌ Vulnérable
SELECT * FROM users WHERE id = ' + user_input + ';

-- Audit avec triggers
CREATE TABLE audit_log (
    table_name TEXT,
    operation TEXT,
    old_values TEXT,
    new_values TEXT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## 9. Cas d'usage avancés

### Concepts incontournables
- **WAL mode** : améliore la concurrence en lecture/écriture
- **ATTACH** : travailler avec plusieurs bases simultanément
- **VACUUM** : réorganisation et nettoyage de la base
- **Migration** : stratégies pour évolution de schéma

### Syntaxes critiques
```sql
-- WAL mode (recommandé en production)
PRAGMA journal_mode = WAL;

-- Attacher plusieurs bases
ATTACH DATABASE 'other.db' AS other;
SELECT * FROM main.users UNION SELECT * FROM other.users;

-- Maintenance
VACUUM; -- Réorganise et compacte
ANALYZE; -- Met à jour les statistiques

-- Migration de schéma
ALTER TABLE users ADD COLUMN phone TEXT;
```

## Commandes CLI essentielles

```bash
# Import/Export
.mode csv
.import data.csv table_name
.output export.csv
.headers on

# Informations
.tables
.schema
.indices table_name

# Performance
.timer on
.stats on
```

## Mémo des types de données

| Type déclaré | Affinité | Stockage |
|--------------|----------|----------|
| INTEGER | INTEGER | Entier |
| TEXT | TEXT | Chaîne UTF-8 |
| REAL | REAL | Flottant 64-bit |
| BLOB | BLOB | Données binaires |
| NULL | NULL | Valeur nulle |
