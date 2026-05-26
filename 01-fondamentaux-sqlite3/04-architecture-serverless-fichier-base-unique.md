🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.4 Architecture serverless et fichier de base unique

## Introduction - Une révolution dans la simplicité

L'architecture serverless de SQLite et son concept de fichier unique représentent une approche révolutionnaire dans le monde des bases de données. Au lieu de suivre le modèle traditionnel complexe, SQLite a choisi la simplicité radicale.

> **Pour les débutants** : « serverless » ne signifie pas « sans serveur informatique », mais « sans logiciel serveur de base de données séparé ». Votre application devient elle-même son propre moteur de base de données !

## Qu'est-ce que l'architecture serverless ?

### 🏗️ Architecture traditionnelle (avec serveur)

Imaginez un **bureau de poste** :

```
[Vous] → [Guichet] → [Employé postal] → [Salle de tri] → [Boîtes postales]
```

**Dans le monde des bases de données :**
```
┌──────────┐    réseau     ┌────────────────┐    appels système    ┌─────────────┐
│   Votre  │  ◄──────────► │    Serveur     │  ◄─────────────────► │  Fichiers   │
│   appli  │   TCP/IP ou   │  MySQL / PG    │                      │ de données  │
└──────────┘  socket Unix  │  (processus    │                      └─────────────┘
                           │   permanent)   │
                           └────────────────┘
```

**Caractéristiques :**
- **Intermédiaire obligatoire** : le serveur de base de données
- **Communication réseau** : même si tout est sur le même ordinateur
- **Processus séparé** : le serveur fonctionne indépendamment de votre application
- **Gestion complexe** : démarrage, arrêt, configuration, sauvegardes, mises à jour

### 📁 Architecture serverless SQLite

Imaginez votre **classeur personnel** : pas d'intermédiaire, vous ouvrez et écrivez directement.

```
┌──────────────────────────────────┐
│         Votre application        │
│   ┌──────────────────────────┐   │
│   │   libsqlite3 (liée)      │   │
│   └────────────┬─────────────┘   │
└────────────────┼─────────────────┘
                 │ syscalls (read/write/fsync)
                 ▼
         ┌──────────────┐
         │  ma_base.db  │
         └──────────────┘
```

**Caractéristiques :**
- **Accès direct** : pas d'intermédiaire, juste des appels de fonctions
- **Aucun réseau** : tout se passe à l'intérieur du processus de votre application
- **Processus unique** : pas de démon à démarrer ou à surveiller
- **Gestion automatique** : rien à configurer

> 💡 « Serverless » ne veut pas dire « sans serveur informatique » : votre programme peut tout à fait être un service web sur un serveur. Cela veut dire **« sans démon de base de données séparé »** — la base est une bibliothèque liée à votre code.

## Le fichier de base unique - Concept révolutionnaire

### 📦 Qu'est-ce qu'un fichier de base unique ?

**Traditionnellement, une base de données = plusieurs fichiers :**

MySQL 8.x (InnoDB) crée typiquement :
```
/var/lib/mysql/
├── ma_base/
│   ├── users.ibd        (données + index InnoDB de la table users)
│   └── products.ibd     (données + index InnoDB de la table products)
├── ibdata1              (tablespace système)
├── ib_logfile0          (redo log — journal de transactions)
├── ib_logfile1
├── ibtmp1               (tablespace temporaire)
├── mysql.ibd            (catalogue système)
└── binlog.000001        (binary log pour réplication)

/var/log/mysql/
├── error.log
└── slow.log

/etc/mysql/
└── my.cnf               (configuration)
```

**SQLite = UN SEUL fichier :**
```
ma_base.db    ← Tout est dedans !
```

> ⚠️ **Petite nuance honnête** : pendant une transaction, SQLite peut créer **temporairement** un fichier de journal à côté de la base :  
> - `ma_base.db-journal` (mode rollback journal, par défaut)  
> - ou `ma_base.db-wal` + `ma_base.db-shm` (mode WAL — Write-Ahead Logging)  
>  
> Ces fichiers sont **éphémères** : ils disparaissent à la fin de la transaction (rollback journal) ou peuvent être absorbés dans le fichier principal (WAL via *checkpoint*). Le fichier de base reste l'unique source de vérité.

### 🎯 Que contient ce fichier unique ?

Le fichier `.db` de SQLite contient **absolument tout** :

```
ma_base.db
├── 📋 Métadonnées (structure des tables)
├── 📊 Données de toutes les tables
├── 🔍 Tous les index
├── 🔐 Contraintes et triggers
├── 👁️ Vues définies
├── ⚙️ Configuration spécifique
└── 🗃️ Espace libre géré automatiquement
```

### Démonstration pratique

Créons une base de données et explorons son contenu :

```bash
# Créer une nouvelle base
sqlite3 exemple.db
```

```sql
-- Activer la vérification des clés étrangères (à faire à chaque connexion en SQLite !)
PRAGMA foreign_keys = ON;

-- Créer plusieurs tables
CREATE TABLE users (
    id    INTEGER PRIMARY KEY,
    nom   TEXT NOT NULL,
    email TEXT UNIQUE
);

CREATE TABLE posts (
    id       INTEGER PRIMARY KEY,
    titre    TEXT,
    contenu  TEXT,
    user_id  INTEGER,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Ajouter des données
INSERT INTO users (nom, email) VALUES
    ('Alice', 'alice@email.com'),
    ('Bob',   'bob@email.com');

INSERT INTO posts (titre, contenu, user_id) VALUES
    ('Mon premier post',     'Contenu intéressant',     1),
    ('SQLite c''est génial', 'Architecture serverless', 1),
    ('Base de données',      'Fichier unique',          2);

-- Créer un index
CREATE INDEX idx_posts_user ON posts(user_id);

-- Créer une vue
CREATE VIEW posts_avec_auteur AS  
SELECT p.titre, p.contenu, u.nom AS auteur  
FROM posts p  
JOIN users u ON p.user_id = u.id;  

.quit
```

**Résultat :** tout cela — tables, index, vue, données — est maintenant stocké dans un **seul fichier** `exemple.db`.

> ⚠️ **À retenir** : par défaut, SQLite **n'applique pas** les contraintes `FOREIGN KEY` (pour des raisons de rétrocompatibilité avec les versions antérieures à 3.6.19, publiée en 2009). Il faut activer `PRAGMA foreign_keys = ON;` **à chaque connexion** — ce réglage n'est **pas persistant**. Nous y reviendrons en détail au **module 3** (section 3.3, *Gestion des clés étrangères et contraintes référentielles*).

## Avantages de l'architecture serverless

### 🚀 Simplicité de déploiement

**Base de données traditionnelle :**
```bash
# Installation du serveur
sudo apt install mysql-server

# Configuration
sudo mysql_secure_installation

# Création de la base
mysql -u root -p  
CREATE DATABASE mon_app;  
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'password';  
GRANT ALL ON mon_app.* TO 'app_user'@'localhost';  

# Déploiement de l'application
scp mon_app.jar serveur:/opt/  
scp database_config.xml serveur:/opt/  

# Configuration réseau
sudo ufw allow 3306
```

**SQLite :**
```bash
# Déploiement (c'est tout !)
scp mon_app.jar serveur:/opt/  
scp ma_base.db serveur:/opt/  
```

### 📱 Portabilité parfaite

```bash
# Sauvegarder votre base
cp ma_base.db ~/backup/

# La déplacer sur un autre ordinateur
scp ma_base.db autre-serveur:/home/user/

# Changer d'OS ? Aucun problème !
# Le même fichier fonctionne sur Windows, Mac, Linux, Android...
```

### ⚡ Performance sans latence réseau

**Base traditionnelle :**
```
Application → TCP/IP → Serveur → Disque
                ↑
         Latence réseau + sérialisation
     (même en local via socket : 100 µs – 1 ms par aller-retour)
```

**SQLite :**
```
Application → libsqlite3 → Fichier
              ↑
   Appel de fonction direct + syscall read/write
   (quelques µs)
```

> 📊 **En pratique** : sur une petite requête `SELECT` par clé primaire, SQLite répond typiquement en **microsecondes**, là où un SGBD client/serveur (même sur localhost) ajoute plusieurs centaines de microsecondes de surcoût protocolaire. Sur 1 000 petites requêtes en boucle, l'écart se voit nettement.

### 🔧 Zéro administration

**Pas de :**
- ❌ Serveur à démarrer/arrêter
- ❌ Utilisateurs à gérer
- ❌ Ports réseau à configurer
- ❌ Logs à surveiller
- ❌ Mises à jour serveur
- ❌ Monitoring de processus

**SQLite s'occupe de tout automatiquement !**

## Le fichier unique en détail

### 📐 Structure interne du fichier

Le fichier SQLite est organisé en **pages** de taille fixe. Depuis SQLite 3.12 (2016), la **taille de page par défaut est 4096 octets**, mais elle peut être configurée à la création de la base de 512 à 65 536 octets (puissances de 2). Chaque type d'objet (table, index, etc.) est stocké dans une structure d'arbre B (**B-tree**).

```
Fichier SQLite (.db)
├── Page 1    : Header de la base (100 octets de métadonnées
│               + racine de la table sqlite_schema)
├── Page 2..  : sqlite_schema (catalogue : tables, index, vues, triggers)
├── Pages X-Y : Table « users »   (B-tree de données)
├── Pages …   : Index sur users.email   (B-tree)
├── Pages …   : Table « posts »   (B-tree de données)
├── Pages …   : Index sur posts.user_id (B-tree)
└── Pages …   : Pages libres (freelist) — réutilisées avant croissance
```

> 🔍 La toute première page commence par la signature ASCII `SQLite format 3` suivie d'un octet nul (16 octets au total : `53 51 4c 69 74 65 20 66 6f 72 6d 61 74 20 33 00` en hexadécimal). Les 100 premiers octets contiennent les métadonnées de la base : taille de page (offset 16-17), version du format (1 = rollback journal, 2 = WAL — offsets 18-19), compteur de modifications, etc. C'est cet en-tête, et non l'extension du fichier, qui permet aux outils (`file`, DB Browser, etc.) de reconnaître une base SQLite.

### 🔍 Examiner le contenu

```sql
-- Voir les tables dans votre base
.tables

-- Voir la structure complète (CREATE TABLE, CREATE INDEX, etc.)
.schema

-- Liste des bases attachées (main, temp, et celles via ATTACH)
PRAGMA database_list;

-- Statistiques des pages
PRAGMA page_count;          -- Nombre total de pages  
PRAGMA page_size;           -- Taille d'une page en octets  
PRAGMA freelist_count;      -- Pages libres réutilisables  

-- Détails d'une table
PRAGMA table_info(users);  
PRAGMA index_list(users);  

-- Vérifier l'intégrité du fichier
PRAGMA integrity_check;     -- Renvoie 'ok' si tout va bien
```

On peut aussi accéder directement à la **table système** `sqlite_schema` (anciennement `sqlite_master`) :

```sql
-- Catalogue de tous les objets de la base
SELECT type, name, tbl_name FROM sqlite_schema;
```

### 🔗 Travailler avec plusieurs fichiers : `ATTACH DATABASE`

Le fait que chaque base soit un fichier unique a un corollaire intéressant : **on peut connecter plusieurs bases dans la même session** et faire des requêtes inter-bases avec `ATTACH` :

```sql
-- Démarrez sur ventes_2025.db
sqlite3 ventes_2025.db
```
```sql
-- Attachez une seconde base (préfixe au choix)
ATTACH DATABASE 'ventes_2026.db' AS v2026;

-- Requête couvrant les deux bases (le préfixe "main" est implicite)
SELECT 'main' AS source, COUNT(*) AS lignes FROM ventes  
UNION ALL  
SELECT 'v2026',          COUNT(*)          FROM v2026.ventes;  

DETACH DATABASE v2026;
```

> ℹ️ Jusqu'à **10 bases** peuvent être attachées simultanément par défaut (configurable au compile-time jusqu'à 125). C'est pratique pour fusionner deux exports, comparer deux versions, ou faire de l'archivage.

### 💾 Gestion automatique de l'espace

SQLite gère automatiquement :

```sql
-- Quand vous supprimez des données
DELETE FROM posts WHERE id < 10;

-- SQLite marque les pages concernées comme "libres" (freelist)
-- et les réutilise pour de futures écritures.
-- Le fichier ne rétrécit donc PAS automatiquement.

-- Pour récupérer réellement l'espace disque, il faut « compacter » :
VACUUM;
```

> 💡 **Auto-vacuum** : si vous voulez que SQLite récupère l'espace au fur et à mesure (sans avoir à lancer `VACUUM`), activez l'option **avant de créer les tables** :
> ```sql
> PRAGMA auto_vacuum = FULL;     -- ou INCREMENTAL
> ```
> Attention : modifier `auto_vacuum` sur une base existante nécessite un `VACUUM` complet.

### 🧱 Tables avec ou sans ROWID

Chaque table SQLite « ordinaire » possède une colonne cachée appelée **`ROWID`** (alias : `oid`, `_rowid_`) — un entier signé 64 bits qui sert de clé interne pour le stockage en B-tree.

```sql
-- Création d'une table standard
CREATE TABLE users (
    id   INTEGER PRIMARY KEY,    -- ALIAS du ROWID interne (optimisation native)
    nom  TEXT
);

SELECT rowid, id, nom FROM users;
-- → rowid et id ont les MÊMES valeurs (id EST le rowid)
```

> 💡 **À retenir** : déclarer `id INTEGER PRIMARY KEY` (sans `AUTOINCREMENT`) n'ajoute **pas** une colonne supplémentaire — `id` devient simplement un nom plus parlant pour le `ROWID`. C'est très efficace.

Depuis **SQLite 3.8.2 (décembre 2013)**, on peut aussi créer des tables **sans ROWID** :

```sql
CREATE TABLE cles_valeurs (
    cle    TEXT PRIMARY KEY,
    valeur TEXT
) WITHOUT ROWID;
```

Les tables `WITHOUT ROWID` :
- ✅ Économisent un peu d'espace disque quand la clé primaire n'est pas un INTEGER
- ✅ Peuvent être plus rapides pour certains motifs d'accès
- ⚠️ Imposent une clé primaire **NOT NULL** (le standard SQL)
- ⚠️ Ne sont pas accessibles via les versions de SQLite < 3.8.2

Ce sont des **optimisations avancées** — nous y reviendrons au module 5.

## Comparaison avec d'autres architectures

### 📊 Tableau comparatif détaillé

| Aspect | SQLite (serverless) | MySQL/PostgreSQL (serveur) | Fichiers CSV/JSON |
|--------|---------------------|----------------------------|-------------------|
| **Fichiers** | 1 seul (+ éventuels `-wal`/`-shm`) | 10 à 100+ par base | 1 par fichier |
| **Processus** | Dans l'application (in-process) | Démon serveur séparé | Aucun |
| **Réseau** | Aucun | TCP/IP (ou socket Unix) | Aucun |
| **Configuration** | Aucune | Importante | Aucune |
| **Concurrence en lecture** | Illimitée | Illimitée | Aucune (chaque app relit le fichier entier) |
| **Concurrence en écriture** | Un écrivain à la fois | Plusieurs (MVCC) | Risque de corruption sans verrou applicatif |
| **Transactions ACID** | ✅ Oui | ✅ Oui | ❌ Aucune |
| **Requêtes SQL** | ✅ Complètes | ✅ Complètes | ❌ Limitées (ou via outils tiers comme DuckDB) |
| **Index** | ✅ Oui | ✅ Oui | ❌ Aucun |
| **Intégrité référentielle** | ✅ (avec `PRAGMA foreign_keys`) | ✅ | ❌ Aucune |
| **Portabilité du « fichier » entre OS** | ✅ Parfaite | ❌ (dépend des fichiers serveur) | ✅ |
| **Outillage / Lecture humaine** | Outils dédiés (CLI, GUI) | Idem | Lisible directement |

### 🎮 Analogies pour mieux comprendre

**SQLite = Console de jeu portable**
- Tout intégré dans un seul appareil
- Fonctionne partout
- Pas de configuration
- Performance optimale pour un joueur

**MySQL = Console de salon + réseau**
- Équipement séparé et complexe
- Configuration réseau nécessaire
- Excellente pour multijoueur
- Performance optimale pour plusieurs joueurs

**Fichiers CSV = Carnet de notes**
- Simple mais très limité
- Aucune vérification d'erreur
- Difficile à maintenir à grande échelle

## Cas d'usage de l'architecture serverless

### ✅ Parfait pour :

**Applications mobiles (Android — API bas niveau) :**
```java
// Android - SQLite intégré au système
SQLiteDatabase db = openOrCreateDatabase("app.db", MODE_PRIVATE, null);
// Aucune configuration réseau !
```

> 💡 En pratique, sur Android moderne on utilise plutôt la bibliothèque **Room** (Jetpack), qui s'appuie sur SQLite mais offre un ORM type-safe et une intégration avec les coroutines Kotlin.

**Applications de bureau (Python) :**
```python
import sqlite3

# Connexion (crée le fichier si nécessaire)
with sqlite3.connect('mon_app.db') as conn:
    conn.execute("PRAGMA foreign_keys = ON;")
    conn.execute("CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, texte TEXT);")
    conn.execute("INSERT INTO notes (texte) VALUES (?)", ("Premier essai",))
# Le with conn: commit automatiquement à la sortie, ou rollback en cas d'exception
```

**Prototypage rapide :**
```bash
# Test d'une idée en 30 secondes
sqlite3 prototype.db
```
```sql
CREATE TABLE test (id INTEGER PRIMARY KEY, data TEXT);  
INSERT INTO test (data) VALUES ('ça marche !');  
SELECT * FROM test;  
```

**Outils d'analyse (importer un CSV en une commande) :**

Supposons un fichier `ventes.csv` :
```csv
date,produit,prix,quantite
2026-01-15,Stylo,2.50,3
2026-01-15,Cahier,4.90,1
2026-01-16,Stylo,2.50,2
```

Dans le shell SQLite :
```sql
-- Variante moderne (SQLite ≥ 3.32) : la 1ère ligne devient automatiquement
-- les noms de colonnes ET la table est créée si elle n'existe pas
.import --csv ventes.csv ventes

-- Variante classique (toutes versions) : la table doit déjà exister,
-- ou bien on saute manuellement la ligne d'en-tête
.mode csv
.import --skip 1 ventes.csv ventes

-- Analysons les ventes :
SELECT produit, SUM(prix * quantite) AS ca  
FROM ventes  
GROUP BY produit  
ORDER BY ca DESC;  
```

> 💡 SQLite est devenu un **couteau suisse de la data science** : combiné à Pandas (`pd.read_sql_query`) ou à des outils comme **DuckDB**, il permet d'analyser efficacement des millions de lignes directement depuis un fichier `.db`.

### ❌ Moins adapté pour :

**Sites web haute concurrence :**
- Nombreuses écritures simultanées
- Utilisateurs de différents serveurs
- Besoins de réplication

**Applications distribuées :**
- Données partagées entre plusieurs serveurs
- Synchronisation temps réel
- Architecture microservices

## 🛠️ Mise en pratique - Comprendre le fichier unique

**Objectif :** Manipuler le fichier unique et observer concrètement sa portabilité.

### Étape 1 : Créer une base complète

```bash
sqlite3 ma_bibliotheque.db
```

```sql
-- Structure complète
CREATE TABLE auteurs (
    id          INTEGER PRIMARY KEY,
    nom         TEXT NOT NULL,
    nationalite TEXT
);

CREATE TABLE livres (
    id        INTEGER PRIMARY KEY,
    titre     TEXT NOT NULL,
    auteur_id INTEGER,
    annee     INTEGER,
    pages     INTEGER,
    FOREIGN KEY (auteur_id) REFERENCES auteurs(id)
);

CREATE INDEX idx_livres_auteur ON livres(auteur_id);  
CREATE INDEX idx_livres_annee  ON livres(annee);  

CREATE VIEW livres_complets AS  
SELECT l.titre, l.annee, l.pages, a.nom AS auteur, a.nationalite  
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id;  

-- Données de test
INSERT INTO auteurs (nom, nationalite) VALUES
    ('Victor Hugo',            'Française'),
    ('William Shakespeare',    'Anglaise'),
    ('Gabriel García Márquez', 'Colombienne');

INSERT INTO livres (titre, auteur_id, annee, pages) VALUES
    ('Les Misérables',       1, 1862, 1488),
    ('Notre-Dame de Paris',  1, 1831,  512),
    ('Hamlet',               2, 1600,  342),
    ('Roméo et Juliette',    2, 1595,  278),
    ('Cent ans de solitude', 3, 1967,  417);

.quit
```

### Étape 2 : Vérifier le fichier unique

```bash
# Voir la taille du fichier
ls -lh ma_bibliotheque.db

# Le fichier contient TOUT !
file ma_bibliotheque.db
# → ma_bibliotheque.db: SQLite 3.x database, ...
```

### Étape 3 : Test de portabilité

```bash
# Copier la base
cp ma_bibliotheque.db copie_bibliotheque.db

# Ouvrir la copie
sqlite3 copie_bibliotheque.db
```

```sql
-- Vérifier que tout est là (données + vues + index)
SELECT * FROM livres_complets;
.schema
.quit
```

### Étape 4 : Partage ultra-simple

```bash
# Partager votre base (le fichier seul suffit !)
# Envoi possible par email, clé USB, scp, rsync, …
cp ma_bibliotheque.db ~/Desktop/a_partager.db

echo "Base partageable créée ! Un seul fichier contient tout."
```

> 💡 **Bonne pratique avant un envoi** : pour s'assurer qu'aucune transaction n'est en cours et qu'aucun fichier auxiliaire (`-wal`, `-shm`, `-journal`) n'est nécessaire, exécutez d'abord `VACUUM;` puis fermez bien le shell. Cela garantit un fichier propre, autonome et compact.

> ⚠️ **Attention à `cp` sur une base ouverte** : copier le fichier `.db` *pendant qu'une application l'utilise* peut produire une copie incohérente (avec des transactions en cours dans `-wal`). Pour une sauvegarde fiable d'une base active, préférez :  
> - la commande `.backup` du shell : `sqlite3 ma_base.db ".backup sauvegarde.db"`,  
> - ou l'utilitaire **`sqlite3_rsync`** livré avec les *tools* officiels.

## Points clés à retenir

### 🎯 Architecture serverless = Simplicité

- **Votre application EST la base de données**
- **Aucun processus externe** à gérer
- **Communication directe** avec les données
- **Déploiement trivial** : copiez le fichier !

### 📁 Fichier unique = Révolution

- **Tout dans un fichier** : structure + données + index
- **Portabilité parfaite** : fonctionne partout
- **Sauvegarde simple** : copiez le fichier
- **Partage facile** : envoyez le fichier

### ⚖️ Compromis à comprendre

**Avantages :**
- Simplicité extrême
- Performance locale excellente (pas de réseau, pas de sérialisation)
- Zéro administration
- Portabilité parfaite entre OS et architectures

**Limites :**
- Un seul écrivain à la fois sur une base donnée
- Pas d'accès réseau natif (mais des wrappers existent : rqlite, dqlite, Litestream)
- Quelques fonctionnalités SQL avancées en moins par rapport à PostgreSQL

---

**➡️ Dans le prochain chapitre**, nous explorerons en détail les limitations et contraintes de SQLite pour savoir exactement dans quelles situations il excelle et quand il faut considérer d'autres solutions.

⏭️ [1.5 Limitations et contraintes de SQLite](/01-fondamentaux-sqlite3/05-limitations-contraintes-sqlite.md)
