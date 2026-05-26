🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.3 Différences avec les autres SGBD (MySQL, PostgreSQL)

## Introduction - Pourquoi comparer ?

Avant de plonger dans SQLite, il est important de comprendre comment il se positionne par rapport aux autres systèmes de gestion de bases de données. Cette comparaison vous aidera à :
- **Choisir le bon outil** pour vos projets
- **Comprendre les spécificités** de SQLite
- **Éviter les erreurs** de conception dues aux mauvaises attentes

> **Note pour débutants** : Si vous n'avez jamais entendu parler de MySQL ou PostgreSQL, ne vous inquiétez pas ! Nous allons tout expliquer simplement.

## Qu'est-ce qu'un SGBD ?

Un **SGBD** (Système de Gestion de Base de Données) est un logiciel qui permet de stocker, organiser et récupérer des données de manière efficace. C'est comme un bibliothécaire numérique ultra-organisé !

**Les principaux SGBD populaires :**
- **SQLite** : notre sujet d'étude — *embarqué, sans serveur*
- **MySQL** : très populaire pour les sites web (LAMP) — appartient à Oracle depuis 2010
- **MariaDB** : fork communautaire de MySQL, totalement open source — souvent un *drop-in replacement*
- **PostgreSQL** : réputé robuste, riche en fonctionnalités, conformité SQL exemplaire
- **Microsoft SQL Server** : solution Microsoft, gratuit en édition Express
- **Oracle Database** : solution d'entreprise historique, payante
- **MongoDB, Redis…** : NoSQL, hors périmètre SQL stricto sensu

## Architecture : La différence fondamentale

### 🏗️ SGBD traditionnels (MySQL, PostgreSQL)

**Architecture client-serveur :**

```
┌─────────────┐                          ┌─────────────────┐
│ Application │                          │   Serveur SGBD  │
│   cliente   │ ────► réseau TCP/IP ───► │ (processus      │
│             │ ◄──── (ou socket) ────── │  permanent)     │
└─────────────┘                          └────────┬────────┘
                                                  │
                                                  ▼
                                          ┌───────────────┐
                                          │   Fichiers    │
                                          │   de données  │
                                          └───────────────┘
```

**Caractéristiques :**
- Le **serveur de base de données** est un **processus séparé** qui tourne en permanence
- Votre **application** se connecte au serveur via le réseau (ou un socket Unix local)
- **Plusieurs applications, depuis plusieurs machines**, peuvent se connecter simultanément
- Nécessite une **installation, configuration et administration** du serveur

### 📁 SQLite

**Architecture intégrée (in-process) :**

```
┌────────────────────────────────┐
│         Application            │
│  ┌──────────────────────────┐  │
│  │   Bibliothèque SQLite    │  │ ◄── appels de fonctions directs
│  │  (libsqlite3 liée)       │  │     (pas de réseau, pas d'IPC)
│  └────────────┬─────────────┘  │
└───────────────┼────────────────┘
                │
                ▼ accès fichier (read/write)
        ┌───────────────┐
        │  ma_base.db   │
        └───────────────┘
```

**Caractéristiques :**
- **Pas de serveur séparé** : SQLite est une bibliothèque **liée à votre application**
- **Accès direct** au fichier de base de données via des appels système (`read`/`write`)
- **Un seul écrivain à la fois** (mais un **nombre illimité de lecteurs** simultanés, et avec le mode WAL la lecture est possible **pendant** une écriture)
- **Aucune installation** de serveur nécessaire

### Analogie simple

**MySQL/PostgreSQL** = Restaurant avec cuisine :
- Personnel spécialisé (serveur de BDD)
- Plusieurs clients simultanés
- Commandes via serveurs (réseau)
- Infrastructure complexe

**SQLite** = Votre cuisine personnelle :
- Vous cuisinez directement
- Accès immédiat aux ingrédients
- Pas besoin de personnel
- Simple et efficace pour la famille

## Comparaison détaillée

### 📊 Tableau comparatif

| Critère | SQLite | MySQL / MariaDB | PostgreSQL |
|---------|--------|-----------------|------------|
| **Architecture** | Intégrée (in-process) | Client-serveur | Client-serveur |
| **Installation** | Aucune (souvent déjà là) | Serveur à installer | Serveur à installer |
| **Configuration initiale** | Zéro | Importante (utilisateurs, droits, réglages) | Importante |
| **Stockage** | Un seul fichier `.db` | Plusieurs fichiers par base | Arborescence par cluster |
| **Écrivains simultanés** | 1 par base | Plusieurs (MVCC) | Plusieurs (MVCC) |
| **Lecteurs simultanés** | Illimité | Illimité | Illimité |
| **Modèle de concurrence** | Verrous + WAL | MVCC (InnoDB) | MVCC |
| **Taille pratique** | Jusqu'à quelques centaines de Go | Plusieurs To | Plusieurs To, voire Po |
| **Parallélisme intra-requête** | ❌ Non | ⚠️ Limité | ✅ Oui (depuis PG 9.6) |
| **Conformité SQL standard** | Bonne (avec quelques différences documentées) | Correcte avec des extensions propres | Très bonne, parmi les meilleures |
| **Procédures stockées** | ❌ (UDF côté hôte) | ✅ | ✅ (PL/pgSQL et autres langages) |
| **Réplication native** | ❌ (outils tiers : Litestream, rqlite, dqlite) | ✅ (master/replica, group replication) | ✅ (streaming, logical replication) |
| **Types avancés (JSON, géo, tableaux)** | JSON ✅, autres via extensions (SpatiaLite…) | JSON ✅ | JSON/JSONB, géo (PostGIS), tableaux, ranges, etc. |
| **Licence** | Domaine public | GPL v2 (MariaDB) / GPL v2 + commercial (MySQL) | PostgreSQL License (BSD-like) |
| **Courbe d'apprentissage** | Très facile | Moyenne | Moyenne à élevée |
| **Maintenance** | Quasi nulle | Régulière (sauvegardes, MAJ) | Régulière |

### 💾 Stockage des données

**SQLite :**
```
ma_base.db  ← Tout est ici !
```

**MySQL 8.x (moteur InnoDB, par défaut depuis MySQL 5.5) :**
```
/var/lib/mysql/
├── ma_base/
│   ├── table1.ibd       (données + index InnoDB)
│   └── table2.ibd
├── ibdata1              (tablespace système)
├── ib_logfile0          (journal de transaction)
├── ib_logfile1
└── mysql.ibd            (dictionnaire système)
                         (la métadonnée SDI est intégrée dans les .ibd)

/var/log/mysql/
├── error.log
└── slow.log
```

> ℹ️ **Note historique** : avant MySQL 8.0, chaque table générait aussi un fichier `.frm` (structure) et, avec l'ancien moteur MyISAM, des fichiers `.MYD` (données) et `.MYI` (index). Tout cela a été remplacé par InnoDB + le format **SDI** (Serialized Dictionary Information).

**PostgreSQL :**
```
/var/lib/postgresql/<version>/main/
├── base/
│   └── 16384/           (chaque base = un dossier dont le nom est l'OID)
├── pg_wal/              (Write-Ahead Log — journaux de transaction)
├── pg_xact/             (état des transactions)
├── global/              (catalogues partagés)
└── postgresql.conf      (configuration)
```

### 🔧 Installation et configuration

**SQLite :**
```bash
# C'est tout !
sqlite3 ma_base.db
```

**MySQL :**
```bash
# 1) Installation du démon
sudo apt install mysql-server

# 2) Configuration initiale (mot de passe root, sécurisation)
sudo mysql_secure_installation

# 3) Démarrage du service
sudo systemctl start mysql

# 4) Se connecter au client mysql en tant qu'admin
mysql -u root -p
```
```sql
-- (À l'invite mysql>) Créer un utilisateur et lui donner les droits :
CREATE USER 'monuser'@'localhost' IDENTIFIED BY 'motdepasse';  
GRANT ALL PRIVILEGES ON ma_base.* TO 'monuser'@'localhost';  
FLUSH PRIVILEGES;  
```

**PostgreSQL :**
```bash
# Installation
sudo apt install postgresql

# Configuration (création utilisateur et base)
sudo -u postgres createuser --interactive  
sudo -u postgres createdb ma_base  

# Fichiers de configuration (remplacer XX par la version installée, ex: 16, 17, 18)
sudo nano /etc/postgresql/XX/main/postgresql.conf  
sudo nano /etc/postgresql/XX/main/pg_hba.conf  

# Démarrage / arrêt du service
sudo systemctl start postgresql  
sudo systemctl status postgresql  
```

## Types de données - Comparaison

### SQLite (système de types dynamique)

```sql
CREATE TABLE exemple (
    id   INTEGER,
    nom  TEXT,
    prix REAL,
    data BLOB
);

-- SQLite accepte ça sans broncher :
INSERT INTO exemple VALUES (1, 'test', '12.50', 123);
-- Le texte '12.50' sera converti automatiquement (affinité REAL)
-- L'entier 123 sera stocké tel quel dans la colonne data
```

> ℹ️ **Bon à savoir** : depuis SQLite 3.37 (novembre 2021), on peut activer des tables **strictement typées** avec le mot-clé `STRICT` :
> ```sql
> CREATE TABLE exemple (
>     id   INTEGER,
>     nom  TEXT,
>     prix REAL
> ) STRICT;  -- Refuse désormais les types incompatibles
> ```

### MySQL/PostgreSQL (types stricts)

```sql
-- MySQL
CREATE TABLE exemple (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    nom        VARCHAR(100) NOT NULL,
    prix       DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- PostgreSQL
CREATE TABLE exemple (
    id         SERIAL PRIMARY KEY,        -- (ou GENERATED ALWAYS AS IDENTITY depuis PG 10)
    nom        VARCHAR(100) NOT NULL,
    prix       NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Différences clés :**
- **SQLite** : types flexibles par défaut, conversion automatique (« type affinity »), mode `STRICT` optionnel
- **MySQL/PostgreSQL** : types stricts, taille définie, vérification systématique à l'insertion

### 🔑 Clés primaires auto-incrémentées : un cas instructif

Le concept de clé primaire auto-incrémentée illustre bien les différences de philosophie.

| SGBD | Syntaxe usuelle |
|------|-----------------|
| **SQLite** | `id INTEGER PRIMARY KEY` (alias automatique du ROWID interne) |
| **MySQL/MariaDB** | `id INT AUTO_INCREMENT PRIMARY KEY` |
| **PostgreSQL** (moderne) | `id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY` |
| **PostgreSQL** (historique) | `id SERIAL PRIMARY KEY` (toujours supporté) |

> 💡 En SQLite, `INTEGER PRIMARY KEY` **réutilise** automatiquement les IDs des lignes supprimées par défaut. Si vous voulez une suite strictement croissante sans réutilisation, ajoutez `AUTOINCREMENT` :
> ```sql
> id INTEGER PRIMARY KEY AUTOINCREMENT
> ```
> Mais attention : cela ajoute un coût (table `sqlite_sequence` consultée à chaque insert). Ne l'utilisez que si votre logique métier l'exige.

## Performance - Quand utiliser quoi ?

### 🚀 SQLite est généralement plus rapide pour :

**Lectures simples (clés, plages, jointures légères) :**
```sql
-- Cette requête sera très rapide avec SQLite
SELECT * FROM users WHERE id = 123;
```

> ⚡ **Pourquoi ?** SQLite est *in-process* : la requête est un simple appel de fonction, sans round-trip réseau ni sérialisation. Les benchmarks officiels (« [35% Faster Than The Filesystem](https://www.sqlite.org/fasterthanfs.html) ») montrent que SQLite lit et écrit de petits BLOBs (~10 Ko) **35 % plus vite** que la lecture/écriture du même contenu en fichiers individuels avec `fread()`/`fwrite()`, et utilise **20 % moins d'espace disque**. Le secret : les `open()`/`close()` système ne sont appelés qu'une fois pour toute la base.

**Applications mono-processus :**
- Applications de bureau
- Applications mobiles
- Outils d'analyse personnels et notebooks (data science)
- Caches locaux et stockage de configuration

### 🏆 MySQL/PostgreSQL sont généralement plus performants pour :

**Charge concurrente élevée avec beaucoup d'écritures :**
```sql
-- Imaginez 500 utilisateurs passant commande en même temps :
-- les SGBD client/serveur gèrent cela nativement grâce au MVCC
-- (Multi-Version Concurrency Control).
```

**Requêtes analytiques complexes sur de gros volumes :**
```sql
-- Cette requête sera typiquement plus efficace sur PostgreSQL
-- grâce à son planificateur plus avancé et au parallélisme de requête :
SELECT u.nom,
       COUNT(c.id)  AS nb_commandes,
       SUM(c.total) AS ca
FROM users u  
LEFT JOIN commandes c ON u.id = c.user_id  
WHERE u.created_at > '2023-01-01'  
GROUP BY u.id  
HAVING COUNT(c.id) > 10  
ORDER BY ca DESC;  
```

**Applications multi-utilisateurs et multi-serveurs :**
- Sites web à fort trafic d'écriture
- Applications d'entreprise distribuées
- Systèmes transactionnels avec réplication

## Cas d'usage - Choisir le bon outil

### ✅ Choisissez SQLite quand :

**Prototypage et apprentissage :**
```bash
# Vous voulez tester une idée rapidement
sqlite3 prototype.db
# Et c'est parti !
```

**Applications locales :**
- Application mobile (notes, contacts)
- Logiciel de comptabilité personnel
- Outil d'analyse de données
- Cache local d'une application

**Projets simples :**
- Blog personnel
- Site vitrine
- Application interne avec < 20 utilisateurs

### ✅ Choisissez MySQL quand :

**Applications web classiques :**
- Sites e-commerce
- CMS (WordPress, Drupal)
- Applications LAMP/LEMP
- APIs REST avec trafic modéré

**Avantages spécifiques :**
- Large communauté et documentation
- Nombreux hébergeurs le supportent
- Bonne intégration avec PHP

### ✅ Choisissez PostgreSQL quand :

**Applications complexes :**
- Systèmes d'information d'entreprise
- Applications avec besoins analytiques
- Projets nécessitant des types de données avancés
- Applications nécessitant des extensions spécialisées

**Avantages spécifiques :**
- Conformité SQL stricte
- Types de données avancés (JSON, géospatial)
- Extensibilité et fonctionnalités avancées

## Migration entre systèmes

### De SQLite vers MySQL/PostgreSQL

**Raisons typiques :**
- Croissance du nombre d'utilisateurs **écrivant en concurrence**
- Besoin de fonctionnalités avancées (réplication, parallélisme, géo, etc.)
- Architecture distribuée multi-serveurs

**Exemple de traduction de schéma :**
```sql
-- SQLite
CREATE TABLE users (
    id  INTEGER PRIMARY KEY,
    nom TEXT
);

-- Devient en MySQL/MariaDB :
CREATE TABLE users (
    id  INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(255) NOT NULL
);

-- Ou en PostgreSQL :
CREATE TABLE users (
    id  INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nom VARCHAR(255) NOT NULL
);
-- (la syntaxe historique `id SERIAL PRIMARY KEY` fonctionne toujours)
```

**Outils utiles pour la migration :**
- **`sqlite3 base.db .dump`** : produit un script SQL portable (à adapter manuellement pour les différences de dialecte)
- **`sqlite3-to-mysql`** (Python) : automatise la conversion SQLite → MySQL
- **`pgloader`** : outil très puissant pour migrer SQLite/MySQL/MS SQL → PostgreSQL

### De MySQL/PostgreSQL vers SQLite

**Raisons typiques :**
- Simplification de l'architecture (moins de maintenance)
- Application mobile, embarquée ou desktop
- Réduction des coûts d'infrastructure
- Edge computing avec base distribuée près de l'utilisateur (Turso, Cloudflare D1)

## Avantages et inconvénients résumés

### 🟢 Avantages de SQLite
- **Simplicité** : zéro configuration, ça marche tout de suite
- **Portabilité** : un fichier = toute la base
- **Performance** : très rapide en local pour les lectures et les petites écritures
- **Fiabilité** : pas de démon qui peut tomber, intégrité ACID garantie
- **Coût** : gratuit, pas d'infrastructure

### 🔴 Inconvénients de SQLite
- **Concurrence en écriture** : un seul écrivain à la fois par base
- **Réseau** : pas d'accès distant natif (mais Litestream, rqlite, Turso comblent ce trou)
- **Scalabilité verticale limitée** : pas adapté aux entrepôts de données multi-To
- **Fonctionnalités côté serveur absentes** : pas de procédures stockées en SQL, pas de gestion d'utilisateurs SQL, pas de parallélisme intra-requête

### 🟢 Avantages MySQL/PostgreSQL
- **Concurrence en écriture** : nombreux écrivains simultanés via MVCC
- **Scalabilité** : gère plusieurs To et la réplication multi-nœuds
- **Fonctionnalités** : riches et étendues (procédures stockées, types avancés, parallélisme)
- **Écosystème** : nombreux outils, ORMs, extensions

### 🔴 Inconvénients MySQL/PostgreSQL
- **Complexité** : installation et configuration initiales
- **Maintenance** : sauvegardes, mises à jour, monitoring permanent
- **Infrastructure** : serveur dédié (ou conteneur) nécessaire
- **Courbe d'apprentissage** : plus longue à maîtriser sérieusement

## 🎯 Mise en situation - Choisir le bon outil

**Scénario :** Votre ami vous demande conseil pour choisir une base de données pour ses projets. Aidez-le à choisir entre SQLite, MySQL et PostgreSQL.

### Cas 1 : Application mobile de notes personnelles
- 1 utilisateur
- Stockage local
- Synchronisation cloud occasionnelle

### Cas 2 : Site e-commerce pour PME
- 50-200 visiteurs simultanés (dont la plupart lisent, quelques-uns commandent)
- Catalogue de 10 000 produits
- Commandes en ligne avec stock à mettre à jour

### Cas 3 : Prototype d'application d'analyse
- Phase de test
- 1 développeur
- Données à analyser (CSV)

### Cas 4 : Système de gestion d'entreprise
- 100 employés se connectant simultanément depuis différentes machines
- Données critiques avec auditabilité requise
- Rapports analytiques complexes

### Solutions et raisonnement

1. **Cas 1 → SQLite.** Parfait pour une app mobile mono-utilisateur : pas de serveur à déployer, sauvegarde = simple copie du fichier, intégré nativement à Android et iOS.
2. **Cas 2 → MySQL** (ou PostgreSQL). 50-200 visiteurs simultanés majoritairement en lecture passeraient en SQLite avec WAL, mais les écritures concurrentes (commandes, stock) et l'accès depuis plusieurs serveurs web justifient un SGBD client/serveur.
3. **Cas 3 → SQLite.** Idéal pour prototyper, et `.import` d'un CSV se fait en une commande. On peut migrer plus tard si nécessaire.
4. **Cas 4 → PostgreSQL.** Accès réseau multi-machines, rapports complexes (CTE récursives, window functions avancées, types JSON/géo), gestion fine des droits, réplication intégrée.

## Récapitulatif

**SQLite se distingue par :**
- Son **architecture intégrée** (pas de serveur)
- Sa **simplicité d'utilisation**
- Sa **portabilité** (un seul fichier)
- Ses **performances** sur les applications locales

**MySQL/PostgreSQL excellent pour :**
- Les **applications multi-utilisateurs**
- Les **gros volumes** de données
- Les **fonctionnalités avancées**
- Les **besoins d'entreprise**

**Le choix dépend de :**
- Nombre d'écrivains simultanés (et non du nombre total d'utilisateurs)
- Volume de données et sa croissance prévisionnelle
- Complexité des besoins fonctionnels et analytiques
- Infrastructure disponible et budget d'exploitation
- Besoin d'accès réseau natif et de réplication

> 💬 **Une règle simple** : *commencez avec SQLite*. La migration vers un SGBD plus lourd reste possible plus tard si vous heurtez ses limites — alors qu'à l'inverse, on regrette souvent d'avoir surdimensionné dès le départ.

---

**➡️ Dans le prochain chapitre**, nous explorerons en détail l'architecture *serverless* de SQLite et ce que signifie concrètement le concept de « fichier de base unique ».

⏭️ [1.4 Architecture serverless et fichier de base unique](/01-fondamentaux-sqlite3/04-architecture-serverless-fichier-base-unique.md)
