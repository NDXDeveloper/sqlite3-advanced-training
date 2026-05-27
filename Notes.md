# SQLite3 - Notes Essentielles : Concepts et Syntaxes

## 1. Fondamentaux de SQLite3

### Concepts incontournables
- **Architecture serverless** : pas de processus serveur séparé, bibliothèque liée à l'application
- **Base de données = 1 fichier** (`.db`, `.sqlite`, `.sqlite3`) ; reconnu par son en-tête `SQLite format 3\0` (16 octets)
- **Typage dynamique (« manifest typing »)** : la valeur porte son type, pas la colonne — mode `STRICT` disponible depuis 3.37 si typage rigide voulu
- **ACID compliant**, isolation **SERIALIZABLE** par défaut
- **Concurrence** : nombre illimité de lecteurs simultanés, **un seul écrivain à la fois** par base
- **Limites** : pas de procédures stockées SQL, pas de `GRANT`/`REVOKE`, pas de réplication native
- **Domaine public** ; engagement de support du format jusqu'en 2050 minimum

### Syntaxes critiques
```sql
-- Ouvrir/créer une base
sqlite3 database.db        -- depuis le terminal système
.open database.db          -- depuis le shell sqlite3

-- Informations système
.databases                 -- liste les bases ouvertes (main, temp, ATTACHées)
.tables                    -- liste les tables et vues
.schema [nom_table]        -- CREATE statements (toute la base ou une table)
.indexes [nom_table]       -- liste les index
.quit                      -- ou .exit, ou Ctrl+D

-- Vérifications utiles
SELECT sqlite_version();   -- version courante  
PRAGMA integrity_check;    -- doit renvoyer 'ok'  
PRAGMA foreign_keys = ON;  -- à activer à CHAQUE connexion (non persistant)  
```

## 2. Bases du langage SQL

### Concepts incontournables
- **5 classes de stockage** : NULL, INTEGER, REAL, TEXT, BLOB
- **Affinité de type** (TEXT, NUMERIC, INTEGER, REAL, BLOB) : conversion automatique selon le contexte
- **`INTEGER PRIMARY KEY`** = alias du ROWID interne (très rapide ; ajouter `AUTOINCREMENT` seulement si on veut interdire la réutilisation d'IDs — petit coût en perf)
- ⚠️ **Exception au manifest typing** : `INTEGER PRIMARY KEY` n'accepte QUE des entiers — insérer `3.5` lève `datatype mismatch`. Les autres colonnes INTEGER acceptent tout type (le « manifest typing » général).
- **Contraintes** : `NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT`, `PRIMARY KEY`, `FOREIGN KEY`
- **`RETURNING`** (depuis 3.35) : `INSERT`/`UPDATE`/`DELETE … RETURNING col1, col2, …` renvoie en une étape les lignes modifiées — particulièrement utile pour récupérer un `id` auto-incrémenté sans faire un `SELECT last_insert_rowid()` séparé.

### Syntaxes critiques
```sql
-- Création de table
CREATE TABLE users (
    id         INTEGER PRIMARY KEY,                        -- alias du ROWID
    name       TEXT NOT NULL,
    email      TEXT UNIQUE,
    age        INTEGER CHECK(age >= 0),
    created_at TEXT DEFAULT CURRENT_TIMESTAMP              -- ISO-8601
);

-- Table STRICT (typage rigide, depuis 3.37)
CREATE TABLE produits (
    id   INTEGER PRIMARY KEY,
    nom  TEXT NOT NULL,
    prix REAL
) STRICT;

-- CRUD de base
INSERT INTO users (name, email, age) VALUES ('John', 'john@email.com', 25);  
SELECT * FROM users WHERE age > 18 ORDER BY name;  
UPDATE users SET age = 26 WHERE id = 1;  
DELETE FROM users WHERE id = 1;  

-- UPSERT (INSERT ... ON CONFLICT, depuis 3.24)
INSERT INTO scores (joueur, points) VALUES ('Alice', 10)  
ON CONFLICT(joueur) DO UPDATE SET points = points + excluded.points;  

-- RETURNING (depuis 3.35) : récupérer en une étape les lignes affectées
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')  
    RETURNING id, created_at;                  -- pas besoin de SELECT séparé  
UPDATE users SET status = 'archived'  
    WHERE last_login < date('now', '-1 year')  
    RETURNING id, name;                        -- liste des comptes archivés  
DELETE FROM sessions WHERE expires_at < datetime('now')  
    RETURNING id;                              -- liste des sessions purgées  

-- Requêtes groupées
SELECT department, COUNT(*), AVG(age)  
FROM users  
GROUP BY department  
HAVING COUNT(*) > 5;  
```

## 3. Conception et modélisation avancée

### Concepts incontournables
- **Normalisation** : éliminer la redondance (1NF → 2NF → 3NF)
- **Clés étrangères** : `PRAGMA foreign_keys = ON;` à activer à **chaque connexion** (non persistant)
- **FK différée** : `DEFERRABLE INITIALLY DEFERRED` — vérification au `COMMIT` (utile pour insérer un cycle dans une transaction)
- **FK cible obligatoire** : une `FOREIGN KEY` doit pointer vers la `PRIMARY KEY` ou une colonne `UNIQUE` (sinon `foreign key mismatch`)
- **Triggers** : code exécuté automatiquement sur événements (`BEFORE`/`AFTER` `INSERT`/`UPDATE`/`DELETE`)
- ⚠️ **`RAISE(ABORT, '…')` exige une chaîne littérale constante** — pas de `||` ni de `NEW.col` (sinon `Parse error`)
- ⚠️ **`AFTER UPDATE OF col`** se déclenche dès que `col` figure dans le `SET`, même si `OLD.col = NEW.col` — ajouter `WHEN OLD.col <> NEW.col` pour ne réagir qu'aux vrais changements
- **Vues** : requêtes stockées comme tables virtuelles
- ⚠️ **En SQLite, TOUTES les vues sont en lecture seule** (≠ PostgreSQL/MySQL) — `UPDATE/INSERT/DELETE` direct échoue avec `cannot modify because it is a view`. Pour rendre une vue modifiable, utiliser un trigger **`INSTEAD OF`** (réservé aux vues).
- **Colonnes générées** : `GENERATED ALWAYS AS (...) [VIRTUAL|STORED]` (depuis 3.31)
- **Index UNIQUE partiel** : `CREATE UNIQUE INDEX ... ON t(col) WHERE condition` — contraintes conditionnelles (ex. : « une seule ligne par défaut »)

### Syntaxes critiques
```sql
-- Activation des clés étrangères
PRAGMA foreign_keys = ON;

-- Clé étrangère
CREATE TABLE orders (
    id      INTEGER PRIMARY KEY,
    user_id INTEGER,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Trigger pour mettre à jour updated_at automatiquement à chaque UPDATE.
-- ⚠️ SQLite ne permet PAS de modifier NEW.colonne dans un trigger (contrairement
--    à PostgreSQL). On utilise donc un trigger AFTER UPDATE qui fait un second
--    UPDATE, avec une garde anti-récursion via la clause WHEN.
CREATE TRIGGER users_set_updated_at  
AFTER UPDATE ON users  
WHEN NEW.updated_at = OLD.updated_at   -- évite la récursion infinie  
BEGIN  
    UPDATE users SET updated_at = datetime('now') WHERE id = NEW.id;
END;
-- Au INSERT, la valeur par défaut suffit :
-- created_at TEXT DEFAULT (datetime('now'))
-- updated_at TEXT DEFAULT (datetime('now'))

-- Vue (toujours en lecture seule en SQLite)
CREATE VIEW active_users AS  
SELECT * FROM users WHERE status = 'active';  

-- Rendre une vue modifiable : trigger INSTEAD OF (réservé aux vues)
CREATE TRIGGER soft_delete_active_user
    INSTEAD OF DELETE ON active_users
BEGIN
    UPDATE users SET status = 'deleted' WHERE id = OLD.id;
END;

-- FK différée : cycle inséré dans une transaction
CREATE TABLE employes (
    id         INTEGER PRIMARY KEY,
    manager_id INTEGER NOT NULL,
    FOREIGN KEY (manager_id) REFERENCES employes(id)
        DEFERRABLE INITIALLY DEFERRED
);
BEGIN;
    INSERT INTO employes VALUES (1, 1);  -- self-ref vérifiée au COMMIT
COMMIT;

-- Index UNIQUE partiel : « une seule adresse par défaut par utilisateur »
CREATE UNIQUE INDEX idx_adresse_defaut
    ON adresses(utilisateur_id, type_adresse)
    WHERE defaut = 1;

-- Colonne générée (calculée à la volée)
CREATE TABLE produits (
    id       INTEGER PRIMARY KEY,
    prix_ht  REAL NOT NULL,
    tva      REAL NOT NULL DEFAULT 0.20,
    prix_ttc REAL GENERATED ALWAYS AS (prix_ht * (1 + tva)) STORED
);
```

## 4. Requêtes avancées et optimisation

### Concepts incontournables
- **Sous-requêtes** : requêtes imbriquées (corrélées vs non-corrélées)
- ⚠️ **`ALL` / `ANY` / `SOME` NON supportés** par SQLite (`WHERE x > ALL (…)` → `Parse error`). Équivalents : `> ALL → > (SELECT MAX(…))`, `> ANY → > (SELECT MIN(…))`, `= ANY → IN`, `<> ALL → NOT IN`
- **CTE** : `WITH … AS (...)` — plus lisible que sous-requêtes, supporte la récursion (depuis 3.8.3)
- ⚠️ **CTE récursive : UNE SEULE référence** à la CTE dans la partie récursive (pas de double `JOIN cte`) — pour Fibonacci ou similaires, on propage 2 colonnes (a, b)
- ⚠️ **Alias SELECT non réutilisable dans le même SELECT** (sauf dans `ORDER BY` / `GROUP BY`) — pour réutiliser un calcul, l'encapsuler dans une CTE intermédiaire
- **Window functions** : `OVER (PARTITION BY … ORDER BY …)` (depuis 3.25)
- **`NULLS FIRST` / `NULLS LAST`** dans `ORDER BY` (depuis 3.30)
- **JOIN** : `INNER` et `LEFT` (toujours disponibles) ; **`RIGHT`** et **`FULL OUTER`** depuis 3.39
- **JSON** : built-in depuis 3.38, format binaire **JSONB** depuis 3.45
- ⚠️ **Booléens JSON → INTEGER** : `json_extract` et `->>` retournent `1` / `0` pour `true` / `false` (pas les chaînes `'true'` / `'false'`). Donc `WHERE data ->> '$.actif' = 'true'` renvoie **toujours 0 ligne** ; comparer à `1` à la place. `json_type(...)` retourne en revanche `'true'` / `'false'` comme noms de type.
- ⚠️ **`JSON_SET`/`JSON_OBJECT` + booléen SQL = INTEGER** : `JSON_SET(j, '$.x', true)` ou `JSON_OBJECT('k', true)` stocke `1` (integer JSON), PAS un vrai booléen JSON. Pour stocker un vrai `true`/`false`, passer `JSON('true')` / `JSON('false')`. Visible via `JSON_TYPE` (`'integer'` vs `'true'`). En lecture via `->>`, le résultat est `1`/`0` dans les deux cas.
- ⚠️ **LIKE sur JSON brut = faux positifs** : `JSON_EXTRACT(prefs, '$.cats') LIKE '%fiction%'` matche aussi `"non-fiction"`. Pour une recherche exacte, garder `JSON_EACH` (au prix de la perf) ou inclure les guillemets : `LIKE '%"fiction"%'`.
- ⚠️ **REGEXP natif limité** : si l'extension est compilée, l'implémentation n'accepte **PAS `\d` à l'intérieur d'une classe `[…]`** (`[\dX]` → `unknown \ escape`). Écrire `[0-9X]` à la place.
- ⚠️ **GLOB « tout chiffre » = piège** : `s GLOB '[0-9]*'` matche « commence par un chiffre » (le `*` accepte n'importe quoi ensuite, donc `'0abc'` passe). Pour vérifier que **tous** les caractères sont des chiffres, utiliser `s NOT GLOB '*[^0-9]*'` (négation de « contient un non-chiffre »).
- ⚠️ **Division par zéro → NULL silencieux** : `10/0` ne lève PAS d'erreur en SQLite (≠ PostgreSQL/MySQL), retourne `NULL`. Le NULL anormal se confond avec des NULL légitimes et pollue les agrégations. Toujours utiliser `NULLIF(diviseur, 0)` pour rendre l'intention explicite et pouvoir le traiter via `COALESCE`.

### Syntaxes critiques
```sql
-- Sous-requête
SELECT * FROM users WHERE age > (SELECT AVG(age) FROM users);

-- Équivalents SQLite des quantificateurs ALL / ANY (non supportés)
SELECT * FROM produits WHERE prix > (SELECT MAX(prix) FROM concurrents);  -- > ALL  
SELECT * FROM produits WHERE prix > (SELECT MIN(prix) FROM concurrents);  -- > ANY  

-- CTE simple
WITH user_stats AS (
    SELECT department, COUNT(*) AS count, AVG(salary) AS avg_salary
    FROM employees GROUP BY department
)
SELECT * FROM user_stats WHERE count > 10;

-- CTE récursive (générer une suite, parcourir un arbre…)
WITH RECURSIVE n(i) AS (
    SELECT 1
    UNION ALL
    SELECT i + 1 FROM n WHERE i < 10
)
SELECT * FROM n;

-- Window functions
SELECT name, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rang,
       LAG(salary)  OVER (PARTITION BY department ORDER BY hire_date)   AS prev_salary
FROM employees;

-- JOINs modernes (depuis 3.39)
SELECT * FROM users u RIGHT JOIN orders o ON u.id = o.user_id;  
SELECT * FROM users u FULL OUTER JOIN orders o ON u.id = o.user_id;  

-- JSON : extraction par chemin
-- `->`  : retourne du JSON (chaînes avec guillemets : "fr")
-- `->>` : retourne une valeur SQL (chaînes sans guillemets : fr)
-- `json_extract(...)` : équivalent à `->>`
SELECT data ->> '$.name'                AS nom,              -- forme SQL (TEXT)
       data -> '$.address'              AS adresse_json,     -- forme JSON (sous-objet)
       json_extract(data, '$.address.city') AS ville
FROM products  
WHERE json_valid(data);  

-- JSON : modification
UPDATE users SET preferences = json_set(preferences, '$.theme', 'dark')  
WHERE id = 1;  
```

## 5. Optimisation des performances

### Concepts incontournables
- **Index** : structure B-tree pour accélérer les recherches
- ⚠️ **`INTEGER PRIMARY KEY` n'est PAS indexé** au sens classique : c'est un alias du `rowid` (la table est physiquement triée par cette clé). `EXPLAIN QUERY PLAN` affiche `SEARCH USING INTEGER PRIMARY KEY (rowid=?)` — encore plus rapide qu'un index séparé. Toute autre `PRIMARY KEY` ou `UNIQUE` crée un index automatique `sqlite_autoindex_<table>_<n>`.
- **EXPLAIN QUERY PLAN** : analyse du plan d'exécution. Affiche `SCAN`, `SEARCH`, `USING INDEX`, `TEMP B-TREE`, `MULTI-INDEX OR`, `AUTOMATIC COVERING INDEX`, `CORRELATED SCALAR SUBQUERY`.
- ⚠️ **`SCAN ... USING INDEX`** ≠ `SCAN` simple : parcours d'index **dans l'ordre**, très rapide quand `ORDER BY` ou `LIMIT` peuvent court-circuiter. Pas de TEMP B-TREE pour tri.
- ⚠️ **`AUTOMATIC COVERING INDEX`** dans EXPLAIN = **signal d'index manquant** : SQLite construit un index temporaire EN MÉMOIRE à chaque exécution. Ajouter un vrai `CREATE INDEX` permanent.
- **PRAGMA** : paramètres de configuration. La plupart sont **par connexion** ; seul `journal_mode = WAL` est **persistant** (reste actif entre toutes les connexions futures).
- **`ANALYZE`** : collecte les stats dans `sqlite_stat1`. Non automatique — à relancer après gros INSERT/DELETE.
- ⚠️ **EXPLAIN QUERY PLAN n'affiche PAS la cardinalité estimée** (pas de `(~N rows)` — c'est PostgreSQL). Pour estimer la sélectivité : `SELECT * FROM sqlite_stat1` après `ANALYZE`.

### Pièges du planificateur de requêtes
- ⚠️ **`MULTI-INDEX OR` automatique** : `WHERE a = ? OR b > ?` utilise les deux index (si présents) sans réécriture en `UNION`. `UNION` ajoute même un TEMP B-TREE pour dédupliquer — souvent **plus lent** que le `OR` original.
- ⚠️ **`FROM a, b WHERE a.x = b.x` ≠ produit cartésien** : SQLite l'optimise **identiquement** à `JOIN ... ON`. Le vrai produit cartésien arrive seulement sans condition de jointure (ou sans index — coût `O(n × m)`).
- ⚠️ **Ordre des `JOIN INNER` réordonné automatiquement** par SQLite selon `sqlite_stat1`. L'ordre d'écriture n'influence pas le plan (sauf `CROSS JOIN` qui le force).
- ⚠️ **Index partiel** : la requête doit contenir **la même condition WHERE** que l'index (ou plus restrictive). Sinon SCAN.
- ⚠️ **`COLLATE NOCASE` exige un index DÉDIÉ** avec la même collation : un index sur `nom` ne suffit PAS pour `WHERE nom = 'X' COLLATE NOCASE`. Créer `CREATE INDEX … ON t(nom COLLATE NOCASE)`.
- ⚠️ **Index composite : égalité avant inégalité**. `CREATE INDEX idx ON t(region, date)` est utile pour `WHERE region = ? AND date > ?` ; l'inverser rend la 1ʳᵉ colonne inutilisable.
- **`EXISTS` vs `IN`** : pas universel. `IN` matérialise une liste (mieux si sous-requête petite + table externe grande). `EXISTS` court-circuite (mieux avec `LIMIT` ou table externe petite). Toujours mesurer.

### Pièges des PRAGMA (configuration)
- ⚠️ **`PRAGMA journal_mode = WAL` ignoré silencieusement en transaction** : retourne le mode précédent sans erreur. Appeler **avant toute requête**, puis vérifier : `SELECT * FROM pragma_journal_mode()`.
- ⚠️ **Cache par connexion** par défaut (pas partagé). Le mode `shared cache` est officiellement **déconseillé** par les développeurs SQLite — utiliser WAL pour la concurrence.
- ⚠️ **`PRAGMA auto_vacuum` ne peut être changé après création de tables** : transition `NONE → FULL/INCREMENTAL` exige un `VACUUM;` complet pour prendre effet. À définir AVANT toute table sur base neuve.
- ⚠️ **`PRAGMA foreign_keys = OFF` puis `ON` ne re-vérifie pas l'existant** : les lignes orphelines créées pendant `OFF` restent silencieusement. Toujours faire `PRAGMA foreign_key_check;` après un import. Alternative sûre : transaction unique + `PRAGMA defer_foreign_keys = ON`.
- ⚠️ **`PRAGMA temp_store_directory` est déprécié** — utiliser la variable d'environnement `SQLITE_TMPDIR` à la place.
- ℹ️ **`synchronous` par défaut diffère selon journal_mode** : `FULL` en rollback, `NORMAL` en WAL. D'où la recommandation WAL + NORMAL (plus rapide ET plus sûr que rollback + FULL).
- ⚠️ **`cache_size = N` (négatif)** = N Kibibytes (1024). `-64000` ≈ 62.5 Mio (pas 64 Mo exacts).

### Syntaxes critiques
```sql
-- Index simple, composite, partiel
CREATE INDEX idx_user_email ON users(email);  
CREATE UNIQUE INDEX idx_user_name_dept ON users(name, department);  
CREATE INDEX idx_orders_recent ON orders(created_at) WHERE status = 'open'; -- partiel  
CREATE INDEX idx_nom_nocase ON users(nom COLLATE NOCASE);  -- pour `WHERE … COLLATE NOCASE`  

-- Analyse de performance
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = 'test@email.com';  
ANALYZE;                          -- stats pour le planificateur (non automatique)  
SELECT tbl, idx, stat FROM sqlite_stat1;  -- consulter les stats brutes  

-- Introspection
PRAGMA table_info(users);  
PRAGMA index_list(users);  
PRAGMA foreign_key_list(orders);  
PRAGMA foreign_key_check;         -- liste les violations FK (à lancer après import)  

-- Tuning courant
PRAGMA journal_mode = WAL;        -- PERSISTANT ; vérifier le retour !  
PRAGMA synchronous = NORMAL;      -- safe avec WAL, plus rapide que FULL  
PRAGMA busy_timeout = 5000;       -- attend 5 s avant "database is locked"  
PRAGMA cache_size = -20000;       -- valeur NÉGATIVE = Kio (ici ~19,5 Mio)  
                                  -- valeur positive = nombre de pages
PRAGMA temp_store = MEMORY;       -- tables temporaires en RAM  
PRAGMA mmap_size = 268435456;     -- mmap 256 Mio (lectures plus rapides)  
PRAGMA optimize;                  -- recommandé à la fermeture de connexion  
```

## 6. Programmation avancée

### Concepts incontournables
- **Transactions** : ensemble d'opérations atomiques (`BEGIN` / `COMMIT` / `ROLLBACK`)
- **Isolation** : **SERIALIZABLE** uniquement (toujours). `READ UNCOMMITTED` n'existe qu'en mode `shared cache`, cas rare et déconseillé.
- **3 modes de BEGIN (≠ niveaux d'isolation !)** : DEFERRED (par défaut, verrou tardif), IMMEDIATE (verrou d'écriture immédiat — recommandé en écriture), EXCLUSIVE (verrou total ; en mode WAL n'empêche PAS les lecteurs)
- **UDF** : fonctions personnalisées enregistrées depuis le langage hôte (Python `create_function`/`create_aggregate`/`create_window_function`)
- **FTS5** : recherche plein texte intégrée (table virtuelle, `bm25()`, `snippet()`, `highlight()`)
- **Backup API** : copie cohérente même si la base est ouverte (`.backup` shell ou `source.backup(dest)` Python)

### Pièges Python/SQLite à connaître

#### Module `sqlite3` Python

- ⚠️ **`with sqlite3.connect()` NE FERME PAS la connexion** — seulement commit/rollback. Utiliser `try/finally` avec `conn.close()` explicite, ou `contextlib.closing(sqlite3.connect(...))` pour vraiment libérer la ressource.
- ⚠️ **`isolation_level = None` requis pour `BEGIN` manuel** : par défaut, le module sqlite3 Python gère les transactions implicitement. Pour utiliser `conn.execute("BEGIN IMMEDIATE")` manuellement, faire `conn.isolation_level = None` (Python ≤3.11) ou `sqlite3.connect(..., autocommit=True)` (Python 3.12+).
- ⚠️ **`INSERT ... RETURNING` + `commit()`** : le curseur doit être **consommé avant commit**, sinon `cannot commit transaction - SQL statements in progress`. Pattern : `rows = cur.fetchall()` PUIS `conn.commit()`.
- ⚠️ **`:memory:` non partagé** : chaque `sqlite3.connect(':memory:')` crée une base **différente**. Pour qu'un service voie les tables créées au setup, utiliser un **fichier temporaire** (`tempfile.mktemp()`) ou `'file::memory:?cache=shared'` avec `uri=True`.
- ⚠️ **`traceback.format_exc()` HORS bloc except retourne `"NoneType: None"`** — utiliser `traceback.format_exception(type(e), e, e.__traceback__)` pour récupérer la trace d'une exception sauvegardée.
- ⚠️ **`source.backup(dest)` peut bloquer** si une transaction non commitée est ouverte sur source. Faire `source.commit()` avant `source.backup(...)`.

#### Hiérarchie des exceptions sqlite3

```
Exception
└── Error                                   (base de toutes les erreurs DB)
    ├── InterfaceError                      (problème module Python lui-même)
    └── DatabaseError                       (erreur côté base)
        ├── DataError                       (donnée invalide : overflow, format...)
        ├── OperationalError                (verrou, table absente, disk full...)
        ├── IntegrityError                  (UNIQUE/FK/CHECK violation)
        ├── InternalError                   (erreur interne SQLite)
        ├── ProgrammingError                (curseur fermé, API mal utilisée...)
        └── NotSupportedError               (fonctionnalité indisponible)
```

⚠️ **Ordre des `except` : spécifique → général**. `OperationalError` est sous-classe de `DatabaseError`, donc capturer `DatabaseError` AVANT `OperationalError` rend cette dernière inatteignable.

#### Triggers FTS5 en external content

⚠️ **Avec `content='table_source'`, les triggers UPDATE/DELETE DOIVENT utiliser le pattern `'delete'`** — un `UPDATE table_fts SET ...` ou `DELETE FROM table_fts WHERE rowid = ?` direct **corrompt l'index** (`database disk image is malformed`) :

```sql
-- ❌ INCORRECT (corrompt l'index FTS5 external content)
CREATE TRIGGER t_upd AFTER UPDATE ON articles BEGIN
    UPDATE articles_fts SET titre = NEW.titre WHERE rowid = NEW.id;
END;

-- ✅ CORRECT
CREATE TRIGGER t_upd AFTER UPDATE ON articles BEGIN
    INSERT INTO articles_fts(articles_fts, rowid, titre, contenu)
        VALUES('delete', OLD.id, OLD.titre, OLD.contenu);
    INSERT INTO articles_fts(rowid, titre, contenu)
        VALUES (NEW.id, NEW.titre, NEW.contenu);
END;
```

#### Tokenizers FTS5

- **`unicode61`** : multilingue, gère Unicode. Variante `unicode61 remove_diacritics 2` enlève les accents (recommandé pour le français).
- **`porter`** : stemming **EN ANGLAIS UNIQUEMENT** (`running` → `run`). Ne fait pas de stemming utile en français (`mange` ≠ `mangent`).

#### Fonctions mathématiques SQLite

- ⚠️ **`log(x)` est en base 10**, pas naturel (contrairement à Python `math.log`). Pour le logarithme naturel : `ln(x)`.

### Syntaxes critiques

```sql
-- Transaction d'écriture recommandée
BEGIN IMMEDIATE;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;             -- ou ROLLBACK en cas d'erreur

-- SAVEPOINT (sous-transactions ; vraie imbrication)
SAVEPOINT etape1;
    UPDATE ...;
RELEASE etape1;     -- valide ; ou ROLLBACK TO etape1 pour annuler

-- FTS5 basique
CREATE VIRTUAL TABLE documents_fts USING fts5(title, content);  
INSERT INTO documents_fts VALUES ('SQLite', 'recherche plein texte intégrée');  
SELECT title, bm25(documents_fts) AS score,  
       snippet(documents_fts, 1, '<b>', '</b>', '...', 32) AS extrait
FROM documents_fts  
WHERE documents_fts MATCH 'sqlite AND performance'  
ORDER BY bm25(documents_fts);    -- score négatif : plus bas = meilleur  

-- FTS5 external content + triggers de synchronisation (pattern OBLIGATOIRE)
CREATE VIRTUAL TABLE articles_fts USING fts5(
    titre, contenu,
    content='articles', content_rowid='id',
    tokenize='unicode61 remove_diacritics 2'  -- français
);
CREATE TRIGGER a_ins AFTER INSERT ON articles BEGIN
    INSERT INTO articles_fts(rowid, titre, contenu) VALUES (NEW.id, NEW.titre, NEW.contenu);
END;  
CREATE TRIGGER a_upd AFTER UPDATE ON articles BEGIN  
    INSERT INTO articles_fts(articles_fts, rowid, titre, contenu)
        VALUES('delete', OLD.id, OLD.titre, OLD.contenu);
    INSERT INTO articles_fts(rowid, titre, contenu) VALUES (NEW.id, NEW.titre, NEW.contenu);
END;  
CREATE TRIGGER a_del AFTER DELETE ON articles BEGIN  
    INSERT INTO articles_fts(articles_fts, rowid, titre, contenu)
        VALUES('delete', OLD.id, OLD.titre, OLD.contenu);
END;

-- Sauvegarde / restauration
-- Shell : .backup utilise l'API native, sûr même base ouverte
.backup sauvegarde.db
-- Pour restaurer : ouvrir simplement le fichier (.restore n'existe PAS)
sqlite3 sauvegarde.db

-- Outil dédié à la synchro (publié octobre 2024, expérimental) :
-- sqlite3_rsync ma_base.db serveur:/chemin/ma_base.db
-- Algorithme type rsync : ne renvoie que les pages modifiées ; nécessite WAL côté origine.
```

```python
# UDF Python avec deterministic=True (essentiel pour index sur expression)
conn.create_function("c_vers_f", 1, lambda c: c*9/5+32 if c else None,
                     deterministic=True)

# Backup API Python
src = sqlite3.connect('source.db')  
dst = sqlite3.connect('backup.db')  
src.backup(dst, pages=100, progress=lambda status, rem, tot: print(f"{(tot-rem)/tot*100:.0f}%"))  
src.close(); dst.close()  

# Pattern Repository minimal (exceptions métier > exceptions SQLite)
class RepositoryError(Exception): pass  
class ValidationError(RepositoryError): pass  
class ConflictError(RepositoryError): pass  

def creer_user(conn, email):
    conn_local = sqlite3.connect('app.db')      # PAS :memory: (non partagé)
    try:
        cur = conn_local.execute(
            "INSERT INTO users (email) VALUES (?) RETURNING id", (email,))
        rows = cur.fetchall()                   # ← AVANT commit !
        conn_local.commit()
        return rows[0][0]
    except sqlite3.IntegrityError as e:         # spécifique avant général
        conn_local.rollback()
        if "UNIQUE" in str(e):
            raise ConflictError(f"Email '{email}' déjà utilisé") from e
        raise
    finally:
        conn_local.close()                      # ← `with` ne ferme PAS
```

## 7. Intégration et APIs

### Concepts incontournables
- **Connexions** : peu coûteuses, mais à réutiliser plutôt qu'en ouvrir/fermer en boucle
- **Prepared statements** : sécurité (anti-injection) **et** performance (réutilisation du plan d'exécution)
- **Une connexion par thread** en multithread + mode WAL : autorise les lectures concurrentes
- **ORM vs SQL natif** : ORM pour les schémas évolutifs, SQL natif pour les requêtes critiques
- **`with conn:`** en Python = `commit/rollback` automatique — **ne ferme PAS** la connexion (cf. pièges)
- **Base en mémoire** : `sqlite3.connect(':memory:')` — idéale pour les tests, **non partagée** entre connexions
- **Bulk insert** dans une seule transaction : ~25-100× plus rapide que des INSERT autonomes (chacun fait un fsync)

### Pièges Python à connaître

- **`with sqlite3.connect()` NE FERME PAS la connexion** — contrairement à `with open(...)`. Seul `commit/rollback` est automatique. Combinez avec `contextlib.closing` pour la fermeture :
  ```python
  with closing(sqlite3.connect('db.db')) as conn:
      with conn:                          # commit/rollback auto
          conn.execute("INSERT ...")
  # Ici : commit + close garantis
  ```
- **`isolation_level=""` par défaut** : le module ouvre une transaction implicite avant le 1er DML. `connexion.execute("BEGIN")` lève alors `OperationalError: cannot start a transaction within a transaction`. Pour reprendre le contrôle : `sqlite3.connect(..., isolation_level=None)` (mode autocommit) ou Python 3.12+ `autocommit=True`.
- **`detect_types=PARSE_DECLTYPES`** : sans ce flag, lire une colonne `DATE/TIMESTAMP` retourne une **chaîne**, pas un objet `date/datetime`.
- **Python 3.12+ : `datetime` → `execute()` dépréciée + piège UTC silencieux** — passer un `datetime` directement à `cursor.execute()` lève un `DeprecationWarning` (retiré dans une future version). **PIRE** : `CURRENT_TIMESTAMP` SQLite est **toujours en UTC** alors que `datetime.now()` Python est en **local** → un filtre `WHERE timestamp > date_local` rate des entrées **sans erreur** sur les systèmes hors UTC. Solution combinée : un `register_adapter` qui normalise en UTC.
  ```python
  from datetime import datetime, timezone
  def _adapter_dt_utc(d):
      if d.tzinfo is None:
          d = d.astimezone(timezone.utc)
      else:
          d = d.astimezone(timezone.utc)
      return d.strftime("%Y-%m-%d %H:%M:%S")
  sqlite3.register_adapter(datetime, _adapter_dt_utc)
  ```
- **`check_same_thread=True` par défaut** : `ProgrammingError` si on utilise une connexion depuis un autre thread. Solutions : une connexion par thread (recommandé), ou `check_same_thread=False` **avec un verrou explicite**.
- **`:memory:` non partagé** : chaque `connect(':memory:')` crée une base distincte. Pour partager : `sqlite3.connect('file::memory:?cache=shared', uri=True)`.
- **RETURNING + `fetchone()` AVANT `commit()`** : sinon `fetchone()` retourne `None`.
- **`curseur.lastrowid`** après INSERT pour l'ID auto-généré (alternative simple à `RETURNING`).

### Pièges Flask / SQLAlchemy / FastAPI

- **`Model.query` déprécié en Flask-SQLAlchemy 3** → `db.session.execute(db.select(Personne)).scalars().all()`.
- **`Personne.query.get(id)` déprécié** → `db.session.get(Personne, id)`.
- **`session.saveOrUpdate()` (Hibernate 6) déprécié** → `session.merge()` ; `session.delete()` → `session.remove()`.
- **SQLAlchemy 1.x → 2.0** : `session.query(...)` reste supporté mais le style 2.0 est `session.execute(select(...))`. `declarative_base()` → `class Base(DeclarativeBase): pass`.
- **`datetime.utcnow` déprécié Python 3.12+** → `datetime.now(timezone.utc)`.
- **`Mapped[List["Article"]] = relationship(...)`** : SQLAlchemy 2.0 infère la cible/collection automatiquement.
- **🐛 FastAPI + SQLModel avec `table=True`** : la validation Pydantic des champs requis **ne fonctionne pas** (POST `{}` passe → IntegrityError 500). **Solution** : séparer `Base / Table / Create / Read` (voir 7.4) :
  ```python
  class PersonneBase(SQLModel):
      nom: str; email: str = Field(unique=True)
  class Personne(PersonneBase, table=True):
      id: int | None = Field(default=None, primary_key=True)
  class PersonneCreate(PersonneBase): pass          # ← validation Pydantic OK
  class PersonneRead(PersonneBase):
      id: int
  ```

### Pièges C (`<sqlite3.h>`)

- **`SQLITE_STATIC` vs `SQLITE_TRANSIENT`** dans `sqlite3_bind_text` :
  - `SQLITE_STATIC` : SQLite ne copie pas — `ptr` doit rester valide jusqu'à `sqlite3_step()`. À réserver aux **littéraux** ou buffers statiques.
  - `SQLITE_TRANSIENT` : SQLite **copie** immédiatement. Sûr partout. **Choix par défaut** quand on n'est pas certain.
- **`sqlite3_close()` MÊME en cas d'échec de `sqlite3_open()`** : le handle est alloué quand même → fuite mémoire sinon.
- **`sqlite3_reset(stmt)` entre exécutions** d'un statement préparé en boucle (10× plus rapide que repréparer).
- **`localtime_r()` au lieu de `localtime()`** : `localtime` n'est PAS thread-safe (variable statique).
- **`BEGIN IMMEDIATE`** plutôt que `BEGIN` (= DEFERRED) : acquiert le verrou tout de suite, évite `SQLITE_BUSY` différé.

### Pièges JavaScript / Node.js

- **`mapbox/node-sqlite3` → `TryGhost/node-sqlite3`** (repo transféré en 2023).
- **`JoshuaWise/better-sqlite3` → `WiseLibs/better-sqlite3`** depuis 2023.
- **better-sqlite3 est synchrone** : 2-5× plus rapide que `sqlite3` callback-based, **mais peut bloquer l'event loop** sur de grosses requêtes.
- **PRAGMA recommandés better-sqlite3** : `journal_mode = WAL`, `synchronous = NORMAL`, `foreign_keys = ON`.
- **`@sqlite.org/sqlite-wasm`** : build WASM **officielle** SQLite pour le navigateur, supporte OPFS pour la persistance fichier.
- **OPFS nécessite des headers serveur** : `Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp` (vérifier `crossOriginIsolated === true`).
- **WebSQL** : définitivement supprimé de Chrome (2023), jamais supporté par Firefox/Safari. Ne plus l'utiliser.
- **`npm ci --only=production` déprécié** (npm 9+) → `npm ci --omit=dev`.

### Pièges Java / JDBC / Hibernate

- **JDBC en mode auto-commit `true` par défaut** : chaque `executeUpdate()` est une mini-transaction. Pour grouper : `setAutoCommit(false)` + `commit()` final (oubli = perte silencieuse).
- **`Statement.RETURN_GENERATED_KEYS`** + `pstmt.getGeneratedKeys()` pour récupérer l'ID auto-incrémenté.
- **Hibernate 6** : groupId `org.hibernate` → **`org.hibernate.orm`**. `SQLiteDialect` déplacé dans **`org.hibernate.community.dialect`** (module `hibernate-community-dialects` séparé).
- **try-with-resources** + `implements AutoCloseable` pour fermeture automatique des `Connection`/`PreparedStatement`/`ResultSet`.

### 🚨 Pièges sécurité critiques

- **JAMAIS `hashlib.sha256/md5/sha1` pour stocker un mot de passe** — ces hash sont *trop rapides* (milliards/seconde sur GPU).
  - ✅ **Argon2** (`pip install argon2-cffi`) — recommandé OWASP 2023+
  - ✅ **bcrypt** cost ≥ 12 (`pip install bcrypt` / `npm install bcrypt`)
  - ✅ **PBKDF2-HMAC-SHA256** ≥ 600 000 itérations (OWASP 2023), ou SHA-512 ≥ 210 000 iter
- **`crypto.createCipher` / `createDecipher` SUPPRIMÉS dans Node 22** (et l'IV n'était pas utilisé → cryptographiquement faux). Utiliser **`createCipheriv`/`createDecipheriv`** :
  ```javascript
  const iv = crypto.randomBytes(12);   // 12 octets pour AES-GCM
  const cipher = crypto.createCipheriv('aes-256-gcm', SECRET_KEY, iv);
  // ...puis cipher.getAuthTag() à la fin
  ```
- **JWT — algorithm confusion (CVE-2015-9235)** : toujours passer `algorithms: ['HS256']` explicite à `jwt.verify()`, sinon un attaquant peut forger des tokens avec `alg: "none"` ou bascule HS256↔RS256.
- **JWT secret ≥ 32 chars** (`crypto.randomBytes(64).toString('hex')`), **jamais de fallback en clair** dans le code.
- **JWT** : pas de données sensibles dans le payload (signé ≠ chiffré, lisible via jwt.io).
- **`str(e)` au client = leak d'info** (schéma BDD, requêtes SQL). En prod : logger côté serveur + message générique `{"error": "Erreur serveur"}` au client.
- **CORS `origin: '*'`** : OK en dev, **JAMAIS en prod** — restreindre explicitement aux origines autorisées.
- **`X-XSS-Protection` déconseillé** par OWASP 2024+ (peut introduire des vulnérabilités) → utiliser **CSP stricte** à la place.
- **« Sanitize input »** est un **antipattern OWASP** — préférer :
  1. Requêtes paramétrées (anti-injection SQL)
  2. **Échappement à la SORTIE** (anti-XSS via `escapeHtml()` ou framework qui échappe auto)
  3. Tokens anti-CSRF
- **`with app.run(debug=True)` JAMAIS en prod** : expose un debugger interactif (RCE possible). Utiliser `gunicorn`/`uWSGI`/`Waitress`.
- **`express.json({ limit: '1mb' })`** : sans limite, body de 2 Go → OOM trivial.

### 🚨 Pièges architecturaux SQLite

- **PM2 cluster + SQLite = CORRUPTION** : `instances: 'max', exec_mode: 'cluster'` est un **anti-pattern majeur**. SQLite a **un seul writer** par base. Utiliser `instances: 1, exec_mode: 'fork'`.
- **`cp` sur base SQLite ouverte = backup potentiellement corrompu** (transaction en cours). Utiliser :
  ```bash
  sqlite3 ma_base.db ".backup '/chemin/backup.db'"
  # Puis vérifier : sqlite3 backup.db "PRAGMA integrity_check;"
  ```
- **SQLite sur NFS / SMB** : verrouillage non garanti → corruption possible. Pour la HA, voir Litestream / LiteFS / Turso.
- **`PRAGMA foreign_keys = ON`** est **non persistant** : à refaire à CHAQUE connexion.
- **`PRAGMA journal_mode = WAL`** est persistant : à faire UNE fois (sans autre connexion active à ce moment).
- **`PRAGMA busy_timeout = 5000`** (5 s) plutôt que 0 par défaut (erreur immédiate "database is locked").
- **VACUUM** : nécessite ~2× l'espace disque libre, verrou exclusif. Alternative : `auto_vacuum = INCREMENTAL` + `PRAGMA incremental_vacuum(N)`.

### Synchronisation et clock skew

- **`Date.now()` entre clients = fragile** : 2 horloges divergentes → écritures écrasées silencieusement (« last write wins » naïf).
- **Solutions modernes 2024+** :
  - **Hybrid Logical Clocks (HLC)** : timestamp physique + compteur logique, résiste au clock skew.
  - **CRDTs** (Conflict-free Replicated Data Types) : convergence mathématiquement garantie. Bibliothèques : `Yjs`, `Automerge`, `Loro`.
  - **UUIDv7** (RFC 9562, 2024) : ID basé sur timestamp, **ordonné lexicographiquement** — parfait pour les index BTree SQLite, évite les page-splits aléatoires.
- **❌ Code BUGGY courant** : `Date.now() * 1000 + performance.now() % 1000` — mélange microsecondes et millisecondes, décalages de plusieurs secondes.
- **✅ Précision sub-milliseconde** : `performance.timeOrigin + performance.now()`.
- **UPSERT atomique** (SQLite 3.24+) — alternative au pattern « SELECT puis INSERT/UPDATE » avec race condition :
  ```sql
  INSERT INTO personnes (id, nom, updated_at) VALUES (?, ?, ?)
  ON CONFLICT(id) DO UPDATE SET
      nom = excluded.nom, updated_at = excluded.updated_at
  WHERE excluded.updated_at > personnes.updated_at;
  -- Rejette automatiquement les versions plus anciennes
  ```

### Solutions modernes 2024-2026 (référence rapide)

**Frameworks REST**
- **Python** : **FastAPI** + SQLModel (validation Pydantic, OpenAPI auto)
- **Node** : **Hono** (Edge-ready, type-safe), Fastify, Elysia (Bun)
- **Rust** : **Axum 0.8** + rusqlite (note : syntaxe `{id}` au lieu de `:id`)
- **Go** : Chi, Gin

**ORMs modernes**
- **Python** : SQLModel (FastAPI), Peewee (léger), Tortoise (async)
- **TypeScript** : **Prisma**, **Drizzle**, Kysely (query builder type-safe)
- **Mobile** : Room (Android), GRDB.swift (iOS)

**SQLite dans le navigateur**
- **`@sqlite.org/sqlite-wasm`** (officiel, OPFS) — recommandé
- `sql.js` (export/import manuel)
- IndexedDB (alternative non-SQL standard)

**Synchronisation / réplication SQLite**
- **Litestream** : réplication continue vers S3 (asymétrique, primary → replicas R/O)
- **LiteFS** : FS distribué FUSE pour clustering avec failover
- **cr-sqlite** : extension CRDT au niveau colonne
- **ElectricSQL / PowerSync** : sync SQLite ↔ Postgres avec CRDT
- **Turso / LibSQL** : SQLite avec réplication multi-régions

### Syntaxes critiques (Python — checklist d'ouverture sûre)
```python
import sqlite3  
from contextlib import closing  

def open_db(path):
    """Factory pour ouvrir une connexion avec tous les PRAGMA essentiels."""
    conn = sqlite3.connect(path)
    conn.row_factory = sqlite3.Row          # accès par nom de colonne
    conn.execute("PRAGMA foreign_keys = ON")    # à refaire à chaque connexion
    conn.execute("PRAGMA journal_mode = WAL")   # persistant, lectures concurrentes
    conn.execute("PRAGMA busy_timeout = 5000")  # retry pendant 5 s si verrouillé
    return conn

# Usage avec fermeture garantie
with closing(open_db('db.db')) as conn:
    with conn:                              # commit/rollback auto
        # Anti-injection SQL : paramètre `?`
        conn.execute("INSERT INTO users (name, email) VALUES (?, ?)", (name, email))

        # RETURNING (SQLite 3.35+) : équivalent moderne de lastrowid
        cur = conn.execute(
            "INSERT INTO users (name) VALUES (?) RETURNING id, created_at",
            ("Alice",)
        )
        new_id, ts = cur.fetchone()         # ⚠️ AVANT commit()
    # commit ici par `with conn:`

    # Bulk insert dans une transaction = 25-100× plus rapide
    with conn:
        conn.executemany("INSERT INTO logs (msg) VALUES (?)",
                         [(m,) for m in messages])

    # Mode STRICT (SQLite 3.37+) — typage rigide
    conn.execute("""CREATE TABLE t (
        id INTEGER PRIMARY KEY, nom TEXT NOT NULL, age INTEGER
    ) STRICT""")
# close ici par `closing()`
```

## 8. Sécurité et administration

### Concepts incontournables
- **Injection SQL** : **TOUJOURS** utiliser des paramètres liés (`?` ou `:nom`), jamais de concaténation
- **SQLite n'a PAS de GRANT/REVOKE** : permissions gérées par le système de fichiers ET au niveau applicatif
- **Chiffrement** : [SQLCipher](https://www.zetetic.net/sqlcipher/) (fork SQLite, AES-256 transparent du fichier), v4.15.0 stable en 2026
- **Alternative SQLCipher** : [SQLite3 Multiple Ciphers](https://github.com/utelle/SQLite3MultipleCiphers) (ChaCha20-Poly1305 + AES + autres)
- **Audit trail** : via triggers + table dédiée OU API `sqlite3_set_authorizer()`
- **Réplication / sauvegarde continue** : Litestream v0.5.11+ (S3/Azure/GCS), LiteFS (FUSE cluster + failover)

### Boilerplate à mettre en haut de tout module
```python
import sqlite3  
from datetime import datetime, timezone  

# ── 1. Adapter datetime → UTC (règle DeprecationWarning Python 3.12+ ET piège UTC/local) ──
def _adapter_dt_utc(d):
    if d.tzinfo is None:
        d = d.astimezone(timezone.utc)
    else:
        d = d.astimezone(timezone.utc)
    return d.strftime("%Y-%m-%d %H:%M:%S")
sqlite3.register_adapter(datetime, _adapter_dt_utc)

# ── 2. PRAGMA PERSISTANTS : init UNE FOIS par base (écrits dans le header) ──
def init_base(chemin):
    conn = sqlite3.connect(chemin)
    conn.execute("PRAGMA journal_mode = WAL")   # persiste sur la base
    conn.close()

# ── 3. PRAGMA PER-CONNECTION : wrapper rejoue à chaque ouverture ──
def ouvrir_connexion(chemin, **kwargs):
    conn = sqlite3.connect(chemin, **kwargs)
    conn.execute("PRAGMA foreign_keys = ON")    # ⚠️ NON persistant
    return conn
```

### Pièges sécurité critiques

#### Mots de passe et hashing
- ⚠️ **`hashlib.sha256/md5/sha1` interdit pour les mots de passe** : trop rapide (milliards/s sur GPU).
  - ✅ **Argon2** (OWASP 2023+) : `pip install argon2-cffi`, `PasswordHasher().hash(...)` / `.verify(...)`
  - ✅ **bcrypt** cost ≥ 12 (2026)
  - ✅ **PBKDF2-HMAC-SHA256** ≥ 600 000 itérations (OWASP 2023) ou SHA-512 ≥ 210 000
- ⚠️ **Comparaison `hash1 == hash2` vulnérable au timing attack** — utiliser `hmac.compare_digest(...)` (constant-time).
- ⚠️ **NIST SP 800-63B (2017+) abandonne** les règles "majuscule + minuscule + chiffre + caractère spécial" (poussent vers patterns prévisibles type "Password1!"). Préférer : longueur ≥ 8 (12 idéal), max 64+, vérification contre HaveIBeenPwned (k-anonymity API).

#### SQLCipher (chiffrement de fichier)
- ⚠️ **`PRAGMA n'accepte PAS les paramètres préparés `?`** : `conn.execute("PRAGMA key=?", (pwd,))` ne marche **PAS** (interprété littéralement). Solution : échapper `'` ou format hex.
  ```python
  mdp_quote = mot_de_passe.replace("'", "''")
  conn.execute(f"PRAGMA key='{mdp_quote}'")
  # OU clé hex 64 chars (32 octets = 256 bits, sel dérivé du fichier) :
  conn.execute(f"PRAGMA key = \"x'{secrets.token_hex(32)}'\"")
  ```
- ⚠️ **`PRAGMA key` doit venir AVANT TOUT autre PRAGMA** sinon erreur sur fichier non déchiffré.
- ⚠️ **SQLCipher 3 ≠ 4** : v3 défaut HMAC-SHA1, PBKDF2 64k, pages 1024 ; v4 défaut HMAC-SHA512, PBKDF2 256k, pages 4096. **Ouvrir une base v3 avec un binaire v4** exige `PRAGMA cipher_compatibility = 3;` immédiatement après `PRAGMA key`. Migration définitive : `sqlcipher_export()`.
- ⚠️ **`PRAGMA rekey` non atomique sur grosse base** : réécrit toutes les pages, ~2× espace disque libre requis. Pour bases > 1 Go, préférer `sqlcipher_export()` vers nouvelle base puis remplacement.
- ⚠️ **`pysqlcipher3` obsolète depuis 2021** (problèmes Python 3.12+). Utiliser **`sqlcipher3-binary`** (wheels pré-compilés officiels).

#### Chiffrement par champ (Fernet, AES-GCM)
- ⚠️ **Fernet/AES-GCM = IV aléatoire** : `chiffrer("toto")` produit un texte différent à chaque appel → **pas de recherche indexée par égalité** sur le champ chiffré. Solutions :
  1. Colonne séparée avec HMAC déterministe (`hmac.new(cle_hash, val.lower().encode(), sha256).hexdigest()`)
  2. AES-GCM-SIV (déterministe pour égalité, voir `cryptography.hazmat.primitives.ciphers.aead.AESGCMSIV`)
- ⚠️ **BUG : clé Fernet générée dans `__init__()`** → chaque instance a sa propre clé, impossible de déchiffrer entre instances. La clé DOIT être persistée (env, KMS, Vault).

#### Permissions et `current_user()`
- ⚠️ **`current_user()` N'EXISTE PAS en SQLite** (c'est PostgreSQL/MySQL). Pas de notion d'utilisateur courant côté base. Solutions :
  1. Filtrer côté application (passer l'ID en paramètre lié)
  2. UDF Python `conn.create_function("current_user_id", 0, lambda: uid)`
  3. Table TEMP `session_actuelle` (cf. audit ci-dessous)
- ⚠️ **CREATE TRIGGER n'accepte pas les paramètres `?`** pour les identifiants (noms de tables/colonnes). Si dynamique, **validation regex stricte obligatoire** : `re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', nom)`. Sinon injection SQL via DDL.

### Audit avec triggers — pattern par session

#### ⚠️ Piège majeur : trigger persistant NE peut PAS référencer une TEMP table
- `CREATE TRIGGER ... FROM temp.session_actuelle` → erreur SQLite : `trigger X cannot reference objects in database temp`
- Solution **validée par test** : utiliser **`CREATE TEMP TRIGGER`** (recréé à chaque ouverture de connexion authentifiée), qui lui peut référencer une `CREATE TEMP TABLE`. Isolation par connexion garantie.

```sql
-- Tables PERSISTANTES (DDL une fois)
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,  -- UTC !
    nom_utilisateur TEXT, action TEXT, table_cible TEXT,
    enregistrement_id INTEGER,
    anciennes_valeurs TEXT, nouvelles_valeurs TEXT  -- JSON
);

-- TEMP table + TEMP triggers : à recréer à CHAQUE connexion authentifiée
CREATE TEMP TABLE session_actuelle (nom_utilisateur TEXT);  
INSERT INTO session_actuelle VALUES ('alice');  

CREATE TEMP TRIGGER audit_clients_insert AFTER INSERT ON clients  
FOR EACH ROW BEGIN  
    INSERT INTO audit_log (nom_utilisateur, action, table_cible, enregistrement_id, nouvelles_valeurs)
    VALUES (
        COALESCE((SELECT nom_utilisateur FROM session_actuelle LIMIT 1), 'SYSTEM'),
        'INSERT', 'clients', NEW.id,
        json_object('nom', NEW.nom, 'email', NEW.email)
    );
END;
```

### Sauvegardes — pièges critiques

- ⚠️ **`shutil.copy` / `cp` sur base SQLite OUVERTE = corruption probable** si une transaction est en vol. Toujours `Connection.backup()` (Python) ou `.backup` (CLI sqlite3) — copie page par page transactionnellement sûre.
  ```python
  with closing(sqlite3.connect(src)) as s, closing(sqlite3.connect(dst)) as d:
      s.backup(d)
  ```
- ⚠️ **JAMAIS supprimer les fichiers `-wal` et `-shm` manuellement** : le `.db-wal` contient des transactions **commitées** non encore appliquées au fichier principal → **perte de données silencieuse**. Pour réduire la taille du WAL : `PRAGMA wal_checkpoint(TRUNCATE)`.
- ⚠️ **Détection incrémentale par `os.path.getmtime()` ratée en WAL** : les écritures vont dans `.db-wal` sans modifier le mtime du `.db` principal tant qu'aucun checkpoint n'a eu lieu. → Forcer `PRAGMA wal_checkpoint(PASSIVE)` avant comparaison de mtime, OU utiliser Litestream.
- ⚠️ **`PRAGMA integrity_check` peut retourner PLUSIEURS LIGNES** en cas de corruption (jusqu'à 100). Utiliser `fetchall()` et tester `== ['ok']`, sinon `fetchone()` rate les corruptions secondaires.
- ⚠️ **VACUUM en Python = `isolation_level=None` OU `conn.commit()` avant** : par défaut le module sqlite3 ouvre une transaction implicite → `cannot VACUUM from within a transaction`. VACUUM nécessite aussi ~2× l'espace disque libre et verrou exclusif.
- ⚠️ **Checksum « contenu identique »** : `SUM(LENGTH(col))` seul est trompeur car `'Client_1'` et `'MODIFIED'` ont la même longueur (8). Combiner `COUNT + SUM(LENGTH) + MIN + MAX` par colonne, ou hasher côté Python.

### Attaques par canaux auxiliaires (side-channels)
Le chiffrement protège le **contenu** mais pas les **métadonnées** :

| Canal | Info leakée | Mitigation |
|---|---|---|
| Taille du fichier | Volume de données | Padding, base de taille fixe |
| mtime / WAL activity | Quand des écritures ont lieu | `touch` périodique, WAL constant |
| Patterns d'accès disque | Pages chaudes | Cache mémoire applicatif |
| Temps de réponse | Présence d'enregistrement | `hmac.compare_digest` |
| Coredumps / swap | Clé en RAM sur disque | `ulimit -c 0`, `mlock()`, swap off |
| Brute-force GPU | Test de millions de clés | PBKDF2 256k+ ou Argon2 |

### Stockage matériel des clés (HSM / TPM / Secure Enclave)
- **TPM 2.0** (Linux/Windows) : clé scellée dans la puce. Outils : `tpm2-tools`, `clevis`.
- **Secure Enclave** (macOS/iOS) : via `Keychain Services`, manipulée par le SEP.
- **Android Keystore** : `setUserAuthenticationRequired(true)` pour exiger la biométrie.
- **HSM réseau** : AWS CloudHSM, Azure Key Vault HSM, YubiHSM 2.
- **Coffres-forts logiciels** : HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager (audit + révocation).

### Monitoring et maintenance
- ⚠️ **`schedule.every().month` N'EXISTE PAS** dans la bibliothèque `schedule` Python (mois variables). Pour planifier mensuel : `schedule.every().day.at("01:00").do(lambda: maintenance() if datetime.now().day == 1 else None)`.
- ⚠️ **Pop-up déprécations Python 3.12+** : `syslog` est UNIX-only — sur Windows utiliser `logging.handlers.SysLogHandler` (TCP/UDP vers serveur distant).
- ⚠️ **`bare except:`** attrape aussi `KeyboardInterrupt` et `SystemExit` — toujours `except Exception:` ou un type spécifique (`sqlite3.Error`, `json.JSONDecodeError`).

### Syntaxes critiques
```python
# Hash mot de passe robuste (PBKDF2-HMAC-SHA256, 600k iter)
import hashlib, secrets, hmac  
sel = secrets.token_hex(32)  
h = hashlib.pbkdf2_hmac('sha256', mdp.encode(), sel.encode(), 600_000).hex()  

# Vérification timing-safe
if hmac.compare_digest(h_calcule, h_stocke):
    print("OK")

# Sauvegarde sûre d'une base SQLite ouverte
from contextlib import closing  
with closing(sqlite3.connect(src)) as s, closing(sqlite3.connect(dst)) as d:  
    s.backup(d)

# VACUUM avec isolation_level=None (sinon échec en transaction implicite)
conn = sqlite3.connect(base, isolation_level=None)  
conn.execute("VACUUM")  
conn.close()  

# Checkpoint WAL SÛRE pour réduire le .db-wal (NE PAS supprimer le fichier !)
busy, log, ck = conn.execute("PRAGMA wal_checkpoint(TRUNCATE)").fetchone()

# Rotation de clé SQLCipher
conn.execute(f"PRAGMA key = '{ancienne_quote}'")  
conn.execute("SELECT count(*) FROM sqlite_master").fetchone()  # vérifier clé OK  
conn.execute(f"PRAGMA rekey = '{nouvelle_quote}'")  
```

```sql
-- Vérification d'intégrité — fetchall() obligatoire (peut retourner plusieurs lignes)
PRAGMA integrity_check;       -- 'ok' si saine, sinon liste de problèmes  
PRAGMA quick_check;           -- plus rapide, vérifications partielles  
PRAGMA foreign_key_check;     -- liste les violations FK  

-- Maintenance combinée recommandée
ANALYZE;                      -- stats pour planificateur  
PRAGMA optimize;              -- analyse incrémentale (à la fermeture de connexion)  
PRAGMA wal_checkpoint(TRUNCATE);  -- compacte le -wal  
VACUUM;                       -- compacte le fichier principal (verrou exclusif)  
```

### Litestream — sauvegarde continue (réplication WAL → S3)
```yaml
# /etc/litestream.yml
dbs:
  - path: /var/lib/app/data.db
    replicas:
      - type: s3
        bucket: backup-prod
        path: data.db
        region: eu-west-1
        retention: 168h          # 7 jours
        snapshot-interval: 24h
        sync-interval: 1s        # RPO ~1 seconde
```
```bash
# Restauration point-in-time
litestream restore -o /var/lib/app/data.db \
    -timestamp 2026-05-27T10:30:00Z \
    s3://backup-prod/data.db
```
> ⚠️ **Pré-requis** : `journal_mode = WAL` obligatoire ; **un seul** processus Litestream par base ; primary → N replicas R/O (pas multi-master).

## 9. Cas d'usage avancés

### Concepts incontournables
- **WAL mode** : améliore la concurrence (lectures pendant écritures), persistant, **système de fichiers local uniquement** (pas NFS/SMB)
- **ATTACH** : jusqu'à 10 bases ouvertes simultanément (configurable)
- **VACUUM** : réorganisation/compactage ; `auto_vacuum` pour récupération automatique
- **Migration** : stratégies pour évolution de schéma (ADD/DROP/RENAME COLUMN)
- **`WITHOUT ROWID`** (depuis 3.8.2) : optimisation pour clés primaires non-INTEGER
- **Wrappers réseau** : Litestream (réplication S3), rqlite (Raft), dqlite, Turso, Cloudflare D1

### Syntaxes critiques
```sql
-- WAL mode (recommandé en production)
PRAGMA journal_mode = WAL;

-- Attacher plusieurs bases
ATTACH DATABASE 'other.db' AS other;  
SELECT * FROM main.users  
UNION ALL  
SELECT * FROM other.users;  
DETACH DATABASE other;  

-- Maintenance
VACUUM;                    -- réorganise et compacte (verrouille la base)  
ANALYZE;                   -- met à jour les statistiques pour le planificateur  
PRAGMA auto_vacuum = FULL; -- à définir AVANT de créer les tables  

-- Migration de schéma (toutes versions ≥ 3.35)
ALTER TABLE users ADD COLUMN phone TEXT;  
ALTER TABLE users DROP COLUMN obsolete;       -- depuis 3.35  
ALTER TABLE users RENAME TO customers;  
ALTER TABLE customers RENAME COLUMN nom TO name;  

-- Table WITHOUT ROWID (clé primaire texte composite p. ex.)
CREATE TABLE cles_valeurs (
    cle    TEXT PRIMARY KEY,
    valeur TEXT
) WITHOUT ROWID;
```

## Commandes CLI essentielles

```text
-- AFFICHAGE
.headers on              en-têtes de colonnes
.mode column             colonnes alignées
.mode box                bordures Unicode (depuis 3.33) -- joli
.mode qbox               idem + quotes sur les chaînes
.mode table              tableau ASCII avec bordures +---+ (depuis 3.33)
.mode markdown           tableau Markdown (réutilisable dans la doc, depuis 3.33)
.mode csv                sortie CSV
.mode json               sortie JSON
.nullvalue NULL          comment afficher NULL (par défaut : vide)
.width 20 10 30          force la largeur en mode column

-- IMPORT / EXPORT
.import --csv data.csv table_name        depuis 3.32 (1ʳᵉ ligne = en-têtes)
.import --skip 1 data.csv table_name     ancienne méthode (.mode csv préalable)
.output export.csv                       redirige vers un fichier
.output stdout                           revient à l'écran
.dump                                    exporte schéma + données en SQL

-- INFORMATIONS
.databases               bases ouvertes (main, temp, ATTACHées)
.tables                  tables et vues
.schema [TABLE]          CREATE statements
.indexes [TABLE]         index (.indices fonctionne aussi)
.show                    paramètres actuels du shell

-- SCRIPTS & SAUVEGARDE
.read script.sql         exécute un fichier SQL
.backup sauvegarde.db    sauvegarde cohérente, base ouverte OK

-- PERFORMANCE
.timer on                temps d'exécution
.stats on                stats détaillées
.eqp on                  EXPLAIN QUERY PLAN auto sur chaque requête

-- QUITTER
.quit                    ou .exit, ou Ctrl+D
```

## Mémo des types et affinités

SQLite distingue **5 classes de stockage** (le type effectif d'une valeur) et **5 affinités** (le type *recommandé* pour une colonne) :

### Classes de stockage (réellement stockées dans la base)
| Classe | Description |
|--------|-------------|
| `NULL` | Valeur absente (n'importe quelle colonne sans `NOT NULL` peut être NULL) |
| `INTEGER` | Entier signé sur 1, 2, 3, 4, 6 ou 8 octets selon la valeur |
| `REAL` | Flottant IEEE 754 sur 8 octets |
| `TEXT` | Chaîne (UTF-8, UTF-16BE ou UTF-16LE) |
| `BLOB` | Données binaires brutes |

### Affinités (déduites du type déclaré)
| Type déclaré | Affinité attribuée |
|--------------|---------------------|
| `INT`, `INTEGER`, `BIGINT`, `TINYINT`… | `INTEGER` |
| `TEXT`, `VARCHAR(n)`, `CHAR(n)`, `CLOB` | `TEXT` |
| `BLOB`, *ou type non reconnu* | `BLOB` |
| `REAL`, `DOUBLE`, `FLOAT` | `REAL` |
| `NUMERIC`, `DECIMAL(p,s)`, `BOOLEAN`, `DATE`, `DATETIME` | `NUMERIC` |

> 💡 En mode `STRICT` (depuis 3.37), le typage est rigide : on doit déclarer un parmi `INT`, `INTEGER`, `REAL`, `TEXT`, `BLOB`, `ANY`, et toute insertion d'un type incompatible est rejetée.

## Fonctions absentes en SQLite et équivalents

SQLite est volontairement minimaliste. Plusieurs fonctions courantes dans MySQL/PostgreSQL n'existent pas en natif — voici les contournements :

| Fonction absente | Équivalent SQLite | Remarque |
|---|---|---|
| `GREATEST(a, b, …)` | `MAX(a, b, …)` | `MAX` à plusieurs arguments est **scalaire** (pas agrégat) |
| `LEAST(a, b, …)` | `MIN(a, b, …)` | `MIN` à plusieurs arguments est **scalaire** |
| `REPEAT(s, n)` | `substr('xxxxxxxxxxxxxxxx', 1, n)` (chaîne pré-allouée) | Pas d'équivalent générique |
| `REVERSE(s)` | aucun natif | À simuler via CTE récursive ou côté application |
| `LPAD(s, n, p)` / `RPAD(s, n, p)` | `printf('%-*s', n, s)` (right-pad) ou concat manuelle | Le `*` accepte une largeur dynamique |
| `IF(cond, a, b)` | `iif(cond, a, b)` (depuis 3.32) ou `CASE WHEN cond THEN a ELSE b END` | |
| `CONCAT(a, b)` | `a \|\| b` | L'opérateur `\|\|` est la seule manière |
| `NOW()` / `CURDATE()` | `datetime('now')` / `date('now')` | UTC par défaut, ajouter `'localtime'` si besoin |
| `STRING_AGG(col, sep)` | `group_concat(col, sep)` | |

> ⚠️ **`ALTER TABLE … ADD COLUMN … STORED` interdit en SQLite** : on ne peut ajouter qu'une colonne `GENERATED ALWAYS AS (…) VIRTUAL` via `ALTER TABLE`. Pour `STORED`, il faut créer la table avec dès le départ (`CREATE TABLE … STORED`).

## Versions clés à connaître

| Version | Date | Apport notable |
|---------|------|----------------|
| 3.0 | 2004 | Format actuel, manifest typing |
| 3.7 | 2010 | Mode WAL |
| 3.8.2 | 2013 | `WITHOUT ROWID` |
| 3.8.3 | 2014 | CTE récursives |
| 3.9 | 2015 | JSON1 (extension) |
| 3.24 | 2018 | `INSERT ... ON CONFLICT` (UPSERT) |
| 3.25 | 2018 | Window functions |
| 3.30 | 2019 | `NULLS FIRST` / `NULLS LAST` dans `ORDER BY` |
| 3.31 | 2020 | Colonnes générées |
| 3.32 | 2020 | `iif()`, opérateur `->`, `.import --csv` |
| 3.33 | 2020 | `.mode box`, `qbox`, `markdown`, `table` ; `sqlite_schema` (anciennement `sqlite_master`) |
| 3.35 | 2021 | `ALTER TABLE DROP COLUMN`, **`RETURNING`**, fonctions mathématiques |
| 3.37 | 2021 | Tables `STRICT` |
| 3.38 | 2022 | JSON built-in, opérateur `->>`, `unixepoch()` |
| 3.39 | 2022 | `RIGHT JOIN`, `FULL OUTER JOIN` |
| 3.45 | 2024 | Format binaire JSONB |
