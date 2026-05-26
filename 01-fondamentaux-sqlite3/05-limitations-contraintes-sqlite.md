🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.5 Limitations et contraintes de SQLite

## Introduction - Pourquoi parler des limitations ?

Comprendre les limitations de SQLite n'est pas négatif, c'est **essentiel** ! Connaître les limites d'un outil vous permet de :
- **Faire des choix éclairés** pour vos projets
- **Éviter les mauvaises surprises** en cours de développement
- **Optimiser** votre utilisation de SQLite
- **Anticiper** les moments où migrer vers autre chose

> **Pensée positive** : chaque limitation de SQLite correspond à un choix de design qui lui donne ses qualités uniques. Ces « limites » sont souvent le prix — assumé — de sa simplicité et de sa fiabilité.

## Vue d'ensemble des principales limitations

### 📊 Résumé visuel

| Domaine | Limitation | Impact | Solution alternative |
|---------|------------|--------|---------------------|
| **Concurrence en écriture** | Un seul écrivain à la fois | Charges fortement écriture-intensives | MySQL/PostgreSQL, ou wrappers (rqlite, dqlite, Litestream) |
| **Réseau** | Accès local uniquement (pas de protocole client/serveur intégré) | Applications distribuées multi-serveurs | MySQL, PostgreSQL, MongoDB |
| **Taille** | 281 To max, < 1 To recommandé en pratique | Big Data, entrepôts de données | PostgreSQL, ClickHouse, BigQuery |
| **Types** | 5 classes de stockage (NULL, INTEGER, REAL, TEXT, BLOB) | Besoin de types riches natifs | PostgreSQL (JSON, géo, tableaux…) |
| **Fonctionnalités** | Pas de procédures stockées côté serveur | Logique métier côté base | MySQL/PostgreSQL |

## 1. Limitations de concurrence

### 🚦 Le verrou d'écriture unique

**Principe :** SQLite utilise un système de verrous simple mais restrictif.

```
Lecture  : ✅ Plusieurs utilisateurs simultanés
Écriture : ❌ Un seul utilisateur à la fois
```

### Démonstration pratique

**Prérequis** : créer d'abord la table dans la base de test.

```bash
sqlite3 test_concurrence.db "CREATE TABLE users (id INTEGER PRIMARY KEY, nom TEXT);"
```

Puis ouvrez **deux terminaux** sur le même fichier :

```bash
# Terminal 1
sqlite3 test_concurrence.db
```

```sql
-- Terminal 1 : Commencer une transaction d'écriture
BEGIN IMMEDIATE;  
INSERT INTO users (nom) VALUES ('Alice');  
-- Ne pas faire COMMIT encore !
```

```bash
# Terminal 2 (dans un autre terminal)
sqlite3 test_concurrence.db
```

```sql
-- Terminal 2 : Essayer d'écrire
INSERT INTO users (nom) VALUES ('Bob');
-- Résultat : "Error: database is locked"
-- (par défaut, échec immédiat ; voir busy_timeout plus bas pour attendre)
```

```sql
-- Terminal 1 : Finir la transaction
COMMIT;
-- Maintenant Terminal 2 peut écrire (rejouez l'INSERT)
```

> 🔬 **Variante WAL** : refaites la même expérience après avoir activé `PRAGMA journal_mode = WAL;`. Vous verrez que le Terminal 2 peut maintenant **lire** pendant que le Terminal 1 écrit. L'écriture concurrente, elle, reste limitée à un écrivain à la fois.

### 📈 Impact selon le type d'application

**✅ Pas de problème pour :**
```python
# Application mobile (1 utilisateur)
def ajouter_note(titre, contenu):
    conn = sqlite3.connect('notes.db')
    conn.execute("INSERT INTO notes (titre, contenu) VALUES (?, ?)",
                (titre, contenu))
    conn.commit()
    conn.close()
```

**⚠️ Plus délicat pour :**
```python
# Site web (plusieurs utilisateurs écrivant simultanément)
@app.route('/commande', methods=['POST'])
def nouvelle_commande():
    # Si 10 personnes commandent en même temps sans configuration adaptée,
    # certaines requêtes peuvent recevoir "database is locked" !
    # → Solution : activer WAL + busy_timeout (voir plus bas).
    conn = sqlite3.connect('boutique.db')
    # ... gérer les écritures concurrentes correctement
```

### 🔧 Solutions et contournements

**Pour améliorer la concurrence — la « combo magique » recommandée :**

```sql
-- 1) Activer le mode WAL (Write-Ahead Logging) — recommandé en production
PRAGMA journal_mode = WAL;

-- 2) Faire attendre l'écrivain au lieu d'échouer immédiatement
PRAGMA busy_timeout = 5000;   -- attend jusqu'à 5 secondes

-- 3) Mode synchrone "normal" : suffisant avec WAL, plus rapide que FULL
PRAGMA synchronous = NORMAL;

-- 4) Démarrer une transaction d'écriture "agressive" qui pose le verrou tout de suite
BEGIN IMMEDIATE;  
INSERT INTO ma_table VALUES (...);  
COMMIT;  
```

**Résultat avec WAL + busy_timeout + synchronous=NORMAL :**
- ✅ Les lectures **ne sont plus bloquées par les écritures** (et inversement)
- ✅ Bien plus d'écritures/seconde possibles
- ✅ Le mode WAL est **persistant** : une fois activé, il reste activé même après fermeture du fichier
- ✅ `BEGIN IMMEDIATE` évite les conflits *« lecteur qui veut devenir écrivain »*
- ❌ Toujours **un seul écrivain à la fois** (c'est un trait fondamental de SQLite)
- ⚠️ WAL nécessite que le fichier soit sur un **système de fichiers local** (pas NFS, pas de partage réseau — voir section 2)

> 💡 **Combien d'écritures/seconde concrètement ?** Sur un SSD moderne, avec WAL et synchronous=NORMAL, SQLite peut soutenir plusieurs milliers d'écritures par seconde sur un seul thread, et tenir des dizaines de milliers de transactions/seconde si on regroupe les insertions. C'est largement suffisant pour 95 % des applications.

## 2. Limitations réseau

### 🌐 Accès local uniquement

SQLite **ne fonctionne pas** en réseau comme MySQL ou PostgreSQL.

**Impossible :**
```bash
# Ceci ne marchera JAMAIS
sqlite3 "réseau://serveur-distant/base.db"
```

**Pourquoi cette limitation ?**
- Fichier unique = accès direct au système de fichiers
- Pas de protocole réseau intégré
- Architecture serverless = pas de serveur d'écoute

### 🔧 Solutions de contournement

**1. Partage de fichiers réseau (DÉCONSEILLÉ — risque de corruption)**
```bash
# Partage Windows/SMB
sqlite3 "//serveur/partage/base.db"

# Partage NFS Linux
sqlite3 "/mnt/nfs/base.db"
```

**Problèmes documentés par les auteurs de SQLite** :
- ❌ Les verrous POSIX `fcntl()` sont **mal implémentés** par de nombreux systèmes de fichiers réseau (SMB, NFS, AFP) — deux clients peuvent croire qu'ils ont chacun le verrou exclusif
- ❌ Risque réel de **corruption de la base** si plusieurs clients écrivent en même temps
- ❌ Le mode WAL **ne fonctionne pas** sur ces systèmes de fichiers (nécessite un partage de mémoire local)
- ❌ Performance dégradée (latence du réseau pour chaque syscall)

Référence officielle : [sqlite.org/howtocorrupt.html](https://www.sqlite.org/howtocorrupt.html) — section *« Filesystems with broken or missing lock implementations »*.

**2. API REST avec SQLite en backend**
```python
# Le serveur expose SQLite via HTTP — les clients ne se connectent jamais
# directement au fichier .db, ce qui contourne le problème du verrou réseau.
from flask import Flask, jsonify  
import sqlite3  

app = Flask(__name__)

def get_conn():
    conn = sqlite3.connect('local.db')
    conn.row_factory = sqlite3.Row     # accès par nom de colonne
    return conn

@app.route('/users')
def get_users():
    with get_conn() as conn:
        rows = conn.execute("SELECT id, nom, email FROM users").fetchall()
    return jsonify([dict(r) for r in rows])
```

**3. Synchronisation bidirectionnelle**
```bash
# Chaque client a sa copie locale
# Synchronisation périodique via rsync, git, ou outil dédié
rsync -av base.db serveur:/backup/
```

**4. Wrappers réseau dédiés (l'écosystème moderne)**

Si vous voulez vraiment exposer une base SQLite sur le réseau de manière sûre, plusieurs projets ouvrent cette voie :

- **[Litestream](https://litestream.io/)** : réplique en continu une base SQLite vers un stockage objet (S3, GCS, Backblaze). Conçu pour la **sauvegarde** et le *disaster recovery*.
- **[rqlite](https://rqlite.io/)** : transforme SQLite en base **distribuée et tolérante aux pannes** via le protocole Raft (consensus).
- **[dqlite](https://dqlite.io/)** : version distribuée de SQLite utilisée par Canonical pour MicroK8s/LXD.
- **[Cloudflare D1](https://developers.cloudflare.com/d1/)** : SQLite as a Service exposé via HTTP, basé sur le moteur SQLite original.
- **[Turso](https://turso.tech/)** : fork (libSQL) avec API HTTP et réplication multi-régions.

Ces solutions montrent que la limitation « pas de réseau » n'est pas un mur — c'est juste qu'il faut ajouter une couche logicielle au-dessus.

## 3. Limitations de taille

### 📏 Limites théoriques vs pratiques

**Limites théoriques (source : [sqlite.org/limits.html](https://www.sqlite.org/limits.html))** :
- **Taille de base** : environ **281 To** (avec page size max de 65 536 octets ; ≈ 17 To avec la page size par défaut de 4096 octets)
- **Taille d'une chaîne ou d'un BLOB** : **1 milliard d'octets** par défaut (limite dure : ≈ 2,1 Go)
- **Nombre de colonnes par table** : 2 000 (configurable jusqu'à 32 767)
- **Taille d'une requête SQL** : 1 milliard d'octets
- **Nombre de bases attachées** (`ATTACH DATABASE`) : 10 par défaut (max 125)
- **Nombre de lignes** : 2⁶⁴ en théorie, en pratique limité par la taille du fichier

**Limites pratiques recommandées :**
- **Taille de base** : < 1 To pour rester confortable (le site officiel parle plutôt de centaines de Go en routine)
- **Nombre d'enregistrements** : SQLite gère facilement des dizaines de millions de lignes ; au-delà, les performances dépendent surtout des **index** et de la **forme des requêtes**
- **Taille d'un enregistrement** : éviter les BLOB > 100 Mo (préférer un stockage de fichiers externes avec un chemin en base)

### 📊 Performance selon la taille (ordres de grandeur indicatifs)

Les chiffres ci-dessous sont des **ordres de grandeur** sur un disque SSD moderne, pour une requête simple bien indexée. Les performances réelles dépendent surtout des **index**, de la **forme des requêtes** et du **schéma**.

| Nombre de lignes | Comportement typique avec index appropriés |
|------------------|--------------------------------------------|
| 1 000 | ⚡ Instantané (microsecondes) |
| 100 000 | ✅ Rapide (millisecondes) |
| 10 000 000 | ✅ Toujours rapide pour les recherches par index ; les `GROUP BY` complets demandent quelques secondes |
| 100 000 000 | ⚠️ Acceptable si bien indexé, mais lent pour les balayages complets |
| 1 000 000 000+ | ❌ Hors zone de confort — envisager un SGBD analytique (PostgreSQL, ClickHouse, DuckDB) |

### 🎯 Cas d'usage selon la taille

**✅ Excellent pour (< 1 Go) :**
- Applications mobiles
- Bases de configuration
- Caches locaux
- Prototypes et développement

**✅ Confortable (1 à 10 Go) :**
- Applications d'analyse de données
- Sites web à trafic modéré
- Applications métier locales
- Notebooks data science

**⚠️ Demande de l'attention (10 à 100 Go) :**
- Bien penser les index (chaque GROUP BY/JOIN sans index devient coûteux)
- Mode WAL recommandé
- Surveiller `PRAGMA cache_size`

**❌ Pas recommandé (> 500 Go en routine) :**
- Entrepôts de données
- Applications avec croissance massive
- Big Data et analytics — passer à PostgreSQL, ClickHouse, DuckDB, ou un cloud datawarehouse

## 4. Limitations des types de données

### 🏷️ Seulement 5 classes de stockage

SQLite ne distingue que **5 classes de stockage** (« storage classes ») :

| Classe | Description |
|--------|-------------|
| `NULL`    | Valeur nulle (absence de valeur) |
| `INTEGER` | Entier signé sur 1, 2, 3, 4, 6 ou 8 octets selon la grandeur |
| `REAL`    | Flottant IEEE 754 sur 8 octets |
| `TEXT`    | Chaîne (UTF-8, UTF-16BE ou UTF-16LE) |
| `BLOB`    | Données binaires brutes |

```sql
CREATE TABLE exemple (
    id    INTEGER,   -- Nombres entiers
    nom   TEXT,      -- Chaînes de caractères
    prix  REAL,      -- Nombres décimaux flottants
    photo BLOB       -- Données binaires
);
-- NULL n'est pas un type qu'on déclare : c'est la « valeur d'absence »,
-- qui peut apparaître dans n'importe quelle colonne non NOT NULL.
```

> 🎯 **Type affinity** : quand on déclare `prix DECIMAL(10,2)`, SQLite **accepte la déclaration** mais lui attribue une **affinité NUMERIC** (la valeur sera stockée en INTEGER, REAL ou TEXT selon ce qu'on insère). Le type déclaré sert d'indication, pas de contrainte (sauf en mode `STRICT`).  
>  
> 💡 Cette flexibilité est appelée **« manifest typing »** par les auteurs de SQLite : c'est la **valeur** qui porte son type, pas la **colonne**. C'est l'inverse du modèle SQL traditionnel, et c'est un choix de design délibéré (depuis SQLite 3.0 en 2004).

### ❌ Types absents (comparé à PostgreSQL)

```sql
-- Ces types n'existent PAS comme types natifs distincts dans SQLite :

-- Dates et heures
birthday   DATE,        -- Pas de type DATE distinct (stocké en TEXT/INTEGER/REAL)  
created_at TIMESTAMP,   -- Idem  
duree      INTERVAL,    -- Pas de type INTERVAL  

-- Booléens
actif      BOOLEAN,     -- Pas de BOOLEAN natif : SQLite reconnaît les littéraux
                        -- TRUE / FALSE depuis 3.23 (2018) mais les stocke en 1 / 0

-- Numériques précis
prix         DECIMAL(10,2),  -- Pas de décimal exact natif (existe via l'extension officielle "decimal")  
pourcentage  NUMERIC(5,2),   -- Stocké en INTEGER/REAL via affinité NUMERIC  

-- Identifiants
uuid         UUID,      -- Pas de type UUID (le stocker en TEXT ou BLOB de 16 octets)

-- Types avancés
coordonnees  POINT,     -- Pas de géolocalisation native (extension SpatiaLite disponible)  
tableau      ARRAY,     -- Pas de tableaux (mais JSON permet de stocker des listes)  
ip           INET,      -- Pas de type IP  
plage_dates  TSRANGE,   -- Pas de types "range"  
enum_statut  ENUM(...), -- Pas d'énumérations natives (utiliser CHECK + valeurs autorisées)  
```

> ✅ **JSON est en revanche très bien supporté** : module JSON1 disponible depuis SQLite **3.9 (2015)**, intégré par défaut depuis **3.38 (février 2022)**, format binaire **JSONB** disponible depuis **3.45 (janvier 2024)** pour de meilleures performances.

### 🔧 Solutions et contournements

**Dates et heures :**
```sql
-- Approche 1 : stocker comme TEXT au format ISO 8601 (recommandé pour la lisibilité)
CREATE TABLE events (
    id         INTEGER PRIMARY KEY,
    nom        TEXT,
    date_event TEXT DEFAULT CURRENT_TIMESTAMP   -- '2026-05-26 14:30:00'
);

-- Approche 2 : stocker comme INTEGER (timestamp Unix, plus compact et rapide pour le tri)
CREATE TABLE logs (
    id        INTEGER PRIMARY KEY,
    message   TEXT,
    timestamp INTEGER DEFAULT (unixepoch())     -- ex: 1748277000
);

-- Conversions entre formats
SELECT datetime(timestamp, 'unixepoch')          FROM logs;     -- INTEGER → TEXT  
SELECT unixepoch('2026-05-26 14:30:00');                        -- TEXT → INTEGER  

-- Calculs sur les dates
SELECT date('now', '+7 days');                                  -- la date dans une semaine  
SELECT date('now', 'start of month');                           -- 1er du mois courant  
SELECT julianday('now') - julianday('2000-01-01') AS jours_ecoulés;  
```

> 💡 La fonction `unixepoch()` est disponible depuis SQLite **3.38**. Avant, on utilisait `strftime('%s', 'now')` (toujours valide).

**Nombres décimaux précis :**
```sql
-- Stocker en centimes pour éviter les erreurs de virgule flottante
CREATE TABLE produits (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    prix_centimes INTEGER  -- 1250 = 12.50€
);

-- Conversion lors de l'affichage
SELECT nom, prix_centimes / 100.0 as prix_euros FROM produits;
```

**JSON (en réalité très bien supporté) :**
```sql
-- Les fonctions JSON sont built-in depuis SQLite 3.38 (février 2022)
CREATE TABLE users (
    id          INTEGER PRIMARY KEY,
    nom         TEXT,
    preferences TEXT    -- JSON stocké en TEXT (ou BLOB pour JSONB depuis 3.45)
);

-- Insertion d'un document JSON
INSERT INTO users (nom, preferences) VALUES
    ('Alice', '{"theme": "dark", "lang": "fr"}');

-- Extraction de valeurs
SELECT nom,
       preferences ->> '$.theme' AS theme,      -- opérateur ->> (texte)
       json_extract(preferences, '$.lang') AS lang
FROM users;

-- Mise à jour d'une clé
UPDATE users  
SET preferences = json_set(preferences, '$.theme', 'light')  
WHERE id = 1;  
```

## 5. Limitations fonctionnelles

### 🚫 Fonctionnalités manquantes (les vraies)

**Pas de procédures stockées :**
```sql
-- Impossible dans SQLite
CREATE PROCEDURE calculer_total(user_id INT)  
BEGIN  
    -- logique complexe
END;
```

**Solution :** logique dans l'application, ou via des **fonctions définies par l'utilisateur (UDF)** enregistrées depuis le langage hôte (Python, C, etc. — module 6).

```python
def calculer_total(user_id):
    conn = sqlite3.connect('base.db')
    # Logique implémentée en Python
    total = conn.execute("""
        SELECT SUM(prix) FROM commandes
        WHERE user_id = ?
    """, (user_id,)).fetchone()[0]
    conn.close()
    return total
```

**Pas de gestion native d'utilisateurs et de droits SQL** (pas de `GRANT`/`REVOKE`) : la sécurité passe par les **permissions du système de fichiers** sur le fichier `.db` et, optionnellement, par la callback **`sqlite3_set_authorizer`** côté application.

### ✅ Idées reçues à corriger

> 💡 De nombreuses ressources sur le web mentionnent des limitations qui **ne sont plus valables** dans les versions modernes de SQLite. Faisons le point.

**✅ RIGHT JOIN et FULL OUTER JOIN existent** depuis SQLite **3.39 (juin 2022)** :
```sql
-- Désormais valide en SQLite ≥ 3.39
SELECT *  
FROM users u  
RIGHT JOIN orders o ON u.id = o.user_id;  

SELECT *  
FROM users u  
FULL OUTER JOIN orders o ON u.id = o.user_id;  
```

**✅ ALTER TABLE DROP COLUMN existe** depuis SQLite **3.35 (mars 2021)** :
```sql
-- Désormais valide en SQLite ≥ 3.35
ALTER TABLE users DROP COLUMN email;
```

**✅ Les contraintes CHECK peuvent être complexes** : elles acceptent n'importe quelle expression SQL déterministe, y compris des appels de fonctions :
```sql
CREATE TABLE promotions (
    id            INTEGER PRIMARY KEY,
    debut_promo   TEXT NOT NULL,
    fin_promo     TEXT NOT NULL,
    CHECK (date(debut_promo) < date(fin_promo))   -- ✅ OK !
);
```

**✅ Les colonnes générées (GENERATED COLUMNS) existent** depuis SQLite **3.31 (janvier 2020)** :
```sql
CREATE TABLE produits (
    id           INTEGER PRIMARY KEY,
    prix_ht      REAL NOT NULL,
    tva          REAL NOT NULL DEFAULT 0.20,
    prix_ttc     REAL GENERATED ALWAYS AS (prix_ht * (1 + tva)) STORED
    -- VIRTUAL (par défaut) : calculé à la volée
    -- STORED            : matérialisé sur disque
);
```

**✅ Les UPSERT (`INSERT ... ON CONFLICT`) existent** depuis SQLite **3.24 (juin 2018)** :
```sql
INSERT INTO scores (joueur, points) VALUES ('Alice', 10)  
ON CONFLICT(joueur) DO UPDATE SET points = points + excluded.points;  
```

**✅ Les CTE récursives existent** depuis SQLite **3.8.3 (février 2014)** — on les verra au module 4.

**✅ Les fonctions de fenêtrage (WINDOW functions) existent** depuis SQLite **3.25 (septembre 2018)** — `ROW_NUMBER()`, `RANK()`, `LAG()`, `LEAD()` etc. fonctionnent comme dans PostgreSQL (module 4).

### 🔧 ALTER TABLE — ce qui reste limité

Voici les opérations `ALTER TABLE` réellement supportées en SQLite moderne (≥ 3.35) :

```sql
-- ✅ Pris en charge
ALTER TABLE users ADD COLUMN phone TEXT;  
ALTER TABLE users DROP COLUMN email;  
ALTER TABLE users RENAME TO customers;  
ALTER TABLE users RENAME COLUMN nom TO name;  

-- ❌ Toujours non pris en charge directement
ALTER TABLE users MODIFY COLUMN age BIGINT;            -- changer le type  
ALTER TABLE users ADD CONSTRAINT fk_dep FOREIGN KEY... -- ajouter une contrainte  
ALTER TABLE users DROP CONSTRAINT ...                  -- supprimer une contrainte  
```

**Solution pour modifier le type d'une colonne ou ses contraintes :** la procédure officielle de SQLite ([documentation](https://www.sqlite.org/lang_altertable.html#otheralter)) consiste à **recréer la table** :

```sql
PRAGMA foreign_keys = OFF;  
BEGIN;  

-- 1. Créer la nouvelle table avec la structure désirée
CREATE TABLE users_new (
    id   INTEGER PRIMARY KEY,
    nom  TEXT NOT NULL,
    age  BIGINT CHECK (age >= 0)
);

-- 2. Copier les données
INSERT INTO users_new (id, nom, age)  
SELECT id, nom, age FROM users;  

-- 3. Supprimer l'ancienne table
DROP TABLE users;

-- 4. Renommer la nouvelle
ALTER TABLE users_new RENAME TO users;

-- 5. Recréer index, triggers, vues qui dépendaient de l'ancienne table
-- (à adapter selon votre schéma)

COMMIT;  
PRAGMA foreign_keys = ON;  
```

## 6. Limitations de déploiement

### 🏢 Pas pour l'entreprise traditionnelle (sans aide)

**Manque pour l'entreprise (en standalone) :**
- ❌ Pas de gestion d'utilisateurs/rôles SQL (`GRANT`/`REVOKE`)
- ❌ Pas de réplication maître/esclave native
- ❌ Pas de partitionnement automatique (sharding)
- ❌ Pas de haute disponibilité avec failover automatique
- ❌ Monitoring intégré limité (mais `PRAGMA` et `sqlite3_analyzer` aident)

**SQLite ne remplace PAS (en standalone) :**
- Serveurs de base de données d'entreprise multi-utilisateurs
- Solutions haute disponibilité avec failover
- Systèmes transactionnels critiques distribués

> 💡 **Nuance importante** : avec des outils comme **rqlite** (consensus Raft), **Litestream** (réplication continue vers S3) ou **dqlite**, SQLite peut être déployé en haute disponibilité ou en lecture-distribuée. Airbus fait voler son A350 avec du SQLite — la fiabilité n'est pas le problème, ce sont les **fonctionnalités d'orchestration** qui ne sont pas dans le moteur de base.

## 7. Quand migrer vers autre chose ?

### 🚨 Signaux d'alarme

**Trafic :**
```bash
# Si vos logs montrent souvent (malgré busy_timeout et WAL activés) :
"database is locked"
"database busy"

# Ou si vous avez :
> de nombreuses écritures concurrentes provenant de plusieurs processus/serveurs
> des transactions d'écriture qui se chevauchent constamment
```

> ⚠️ Avant de migrer, vérifiez d'abord que vous avez activé `PRAGMA journal_mode = WAL;` et `PRAGMA busy_timeout = 5000;` — cela résout 90 % des problèmes de "database is locked" en pratique.

**Taille :**
```bash
# Vérifiez la taille de votre base
ls -lh *.db

# Si > 100 Go avec croissance rapide ET requêtes analytiques lourdes,
# envisagez PostgreSQL / ClickHouse / DuckDB selon le profil de charge.
```

**Complexité :**
```sql
-- Si vous vous retrouvez à faire souvent :
-- Requêtes avec 10+ JOINs
-- Logique métier complexe dans l'app
-- Calculs lourds répétitifs
```

### 🔄 Stratégies de migration

**Migration graduelle :** plutôt qu'une bascule big-bang, on peut conserver SQLite pour certains rôles (cache local, préférences, journaux) et déplacer le cœur transactionnel vers un SGBD client/serveur :

```text
Phase 1 — Tout en SQLite (état initial)  
Phase 2 — Le SGBD principal devient PostgreSQL ;  
          SQLite reste pour les caches locaux, configs, données embarquées
Phase 3 — Évaluation : SQLite est-il toujours utile localement ? (souvent oui)
```

**Outils pratiques de migration :**
- **`sqlite3 base.db .dump`** → produit un script SQL portable (à retoucher pour les différences de dialecte : `AUTOINCREMENT` vs `SERIAL`, etc.)
- **`pgloader`** : outil mature pour migrer SQLite/MySQL/MS SQL → PostgreSQL, gère les conversions de types
- **ORM (SQLAlchemy, Django, Prisma, Drizzle…)** : si vous utilisez déjà un ORM, changer la connection string suffit souvent

## 🎯 Mise en situation - Identifier les limitations

**Scénario :** analysez ces cas d'usage et identifiez les limitations de SQLite.

### Cas 1 : Blog personnel
```python
# 1 auteur, 100 articles, 1 000 visiteurs/jour
def publier_article(titre, contenu):
    conn = sqlite3.connect('blog.db')
    conn.execute("INSERT INTO articles (titre, contenu) VALUES (?, ?)",
                 (titre, contenu))
    conn.commit()
```
**Question :** SQLite convient-il ? Pourquoi ?

### Cas 2 : Boutique en ligne
```python
# Pic de 50 commandes simultanées possible
def passer_commande(user_id, produits):
    conn = sqlite3.connect('boutique.db')
    conn.execute("BEGIN")
    for produit in produits:
        conn.execute("UPDATE stock SET quantite = quantite - ? WHERE id = ?",
                     (produit.qte, produit.id))
    conn.execute("INSERT INTO commandes ...")
    conn.commit()
```
**Question :** quels problèmes peut-il y avoir ?

### Cas 3 : Application mobile
```text
// Pseudo-code — application iPhone, 1 utilisateur, données locales
function sauvegarder_photo(image_data, metadata) {
    db = sqlite_open("photos.db")
    db.execute("INSERT INTO photos (data, metadata) VALUES (?, ?)",
               image_data, metadata)
}
```
**Question :** SQLite est-il adapté ?

### Solutions et raisonnement

1. **Blog personnel → ✅ Parfait pour SQLite.** Un seul auteur écrit, 1 000 visiteurs/jour ne représentent que ~12 lectures/seconde au pic — SQLite peut tenir cent fois plus. Sauvegarde = simple copie du fichier.
2. **Boutique en ligne → ⚠️ Possible avec précautions.** Le scénario annonce un **pic de 50 commandes simultanées**, ce qui correspond à environ 50 transactions d'écriture concurrentes. Avec **WAL activé**, un **busy_timeout** de quelques secondes et des transactions courtes (`BEGIN IMMEDIATE` + COMMIT rapide), une PME modeste passera. Mais dès que les pics deviennent réguliers ou montent à plusieurs centaines de commandes concurrentes (avec décrémentation de stock), un SGBD client/serveur avec MVCC est plus adapté.
3. **Application mobile → ✅ Cas d'usage idéal.** Un seul utilisateur, données locales, pas de réseau, faible empreinte mémoire. C'est même pour ça qu'Android et iOS intègrent SQLite nativement.

## Récapitulatif des limitations

### 🎯 Les « vraies » limitations

**Concurrence en écriture :**
- Un seul écrivain simultané sur un même fichier
- Peut bloquer sous forte charge écriture multi-processus (atténué par WAL)

**Réseau :**
- Accès local uniquement (pas de protocole client/serveur intégré)
- Solutions tierces : Litestream, rqlite, dqlite, Turso, Cloudflare D1

**Fonctionnalités côté serveur :**
- Pas de procédures stockées SQL (mais UDF depuis l'application)
- Pas de `GRANT`/`REVOKE` (la sécurité passe par le système de fichiers)
- Types de données natifs limités (à compenser via affinité ou STRICT)

### 💡 Les « fausses » limitations (mythes courants à corriger)

| Idée reçue | Réalité |
|------------|---------|
| « SQLite n'est pas un vrai SGBD » | ACID complet, conformité SQL étendue, fiabilité aéronautique |
| « SQLite ne tient pas la charge web » | sqlite.org sert lui-même 500 000 requêtes/jour ; règle officielle : OK jusqu'à ~100 000 hits/jour |
| « Pas de RIGHT JOIN » | **Disponible depuis SQLite 3.39 (2022)** |
| « Pas de FULL OUTER JOIN » | **Disponible depuis SQLite 3.39 (2022)** |
| « Pas de DROP COLUMN » | **Disponible depuis SQLite 3.35 (2021)** |
| « Pas vraiment de JSON » | **Built-in depuis 3.38 (2022), JSONB depuis 3.45 (2024)** |
| « Pas de window functions » | **Disponibles depuis SQLite 3.25 (2018)** |
| « Pas d'UPSERT » | **`INSERT … ON CONFLICT` depuis 3.24 (2018)** |
| « Pas de colonnes calculées » | **`GENERATED ALWAYS AS` depuis 3.31 (2020)** |
| « Pas de CTE récursives » | **Disponibles depuis 3.8.3 (2014)** |
| « Limité à quelques Mo » | 281 To en théorie, des bases de centaines de Go en production |
| « Types pas vérifiés » | Mode `STRICT` disponible depuis 3.37 (2021) pour ceux qui veulent du typage rigide |

### 🎯 Règle d'or

> **SQLite est parfait jusqu'à ce qu'il ne le soit plus.** Quand vous atteignez ses limites, elles sont généralement évidentes (vous verrez « database is locked » se multiplier dans vos logs malgré WAL + busy_timeout). Jusque-là, profitez de sa simplicité !

---

**🎉 Félicitations !** Vous avez maintenant une vision complète des fondamentaux de SQLite. Dans le module suivant, nous commencerons à pratiquer avec les bases du langage SQL dans SQLite.

**💡 Points clés à retenir :**
- Chaque limitation correspond à un choix de design (simplicité, fiabilité, portabilité)
- SQLite excelle dans la majorité des cas d'usage courants
- Connaître les limites — et les mythes — aide à faire les bons choix
- La migration vers un SGBD client/serveur reste toujours possible si nécessaire
- L'écosystème moderne (WAL, Litestream, rqlite, JSON…) repousse de nombreuses limites historiques

⏭️ [Module 2 : Bases du langage SQL avec SQLite](/02-bases-langage-sql-sqlite/01-types-donnees-sqlite-text-integer-real-blob-null.md)
