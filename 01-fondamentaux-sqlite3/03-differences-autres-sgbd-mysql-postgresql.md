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
- **SQLite** : Notre sujet d'étude
- **MySQL** : Très populaire pour les sites web
- **PostgreSQL** : Réputé robuste et riche en fonctionnalités
- **Oracle, SQL Server** : Solutions d'entreprise

## Architecture : La différence fondamentale

### 🏗️ SGBD traditionnels (MySQL, PostgreSQL)

**Architecture client-serveur :**

```
[Application] ←→ [Réseau] ←→ [Serveur de BDD] ←→ [Fichiers de données]
```

**Caractéristiques :**
- Le **serveur de base de données** fonctionne en permanence
- Votre **application** se connecte au serveur via le réseau
- **Plusieurs applications** peuvent se connecter simultanément
- Nécessite une **installation et configuration** du serveur

### 📁 SQLite

**Architecture intégrée :**

```
[Application + SQLite] ←→ [Fichier unique]
```

**Caractéristiques :**
- **Pas de serveur séparé** : SQLite fait partie de votre application
- **Accès direct** au fichier de base de données
- **Une seule application** écrit à la fois (mais plusieurs peuvent lire)
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

| Critère | SQLite | MySQL | PostgreSQL |
|---------|--------|-------|------------|
| **Architecture** | Intégrée | Client-serveur | Client-serveur |
| **Installation** | Aucune | Serveur requis | Serveur requis |
| **Configuration** | Zéro | Importante | Importante |
| **Fichiers** | Un seul | Multiples | Multiples |
| **Utilisateurs simultanés** | Limité | Excellent | Excellent |
| **Taille max recommandée** | < 1 TB | Plusieurs TB | Plusieurs TB |
| **Courbe d'apprentissage** | Facile | Moyenne | Difficile |
| **Maintenance** | Aucune | Régulière | Régulière |

### 💾 Stockage des données

**SQLite :**
```
ma_base.db  ← Tout est ici !
```

**MySQL :**
```
/var/lib/mysql/
├── ma_base/
│   ├── table1.frm
│   ├── table1.MYD
│   ├── table1.MYI
│   └── table2.frm
└── logs/
    ├── error.log
    └── slow.log
```

**PostgreSQL :**
```
/var/lib/postgresql/data/
├── base/
│   └── 16384/  (numéros cryptiques)
├── pg_wal/
└── postgresql.conf
```

### 🔧 Installation et configuration

**SQLite :**
```bash
# C'est tout !
sqlite3 ma_base.db
```

**MySQL :**
```bash
# Installation
sudo apt install mysql-server

# Configuration
sudo mysql_secure_installation

# Création utilisateur
mysql -u root -p
CREATE USER 'monuser'@'localhost' IDENTIFIED BY 'motdepasse';
GRANT ALL PRIVILEGES ON ma_base.* TO 'monuser'@'localhost';

# Démarrage du service
sudo systemctl start mysql
```

**PostgreSQL :**
```bash
# Installation
sudo apt install postgresql

# Configuration
sudo -u postgres createuser --interactive
sudo -u postgres createdb ma_base

# Fichiers de configuration
sudo nano /etc/postgresql/13/main/postgresql.conf
sudo nano /etc/postgresql/13/main/pg_hba.conf
```

## Types de données - Comparaison

### SQLite (System de types dynamique)

```sql
CREATE TABLE exemple (
    id INTEGER,
    nom TEXT,
    prix REAL,
    data BLOB
);

-- SQLite accepte ça sans problème :
INSERT INTO exemple VALUES (1, 'test', '12.50', 123);
-- Le texte '12.50' sera converti automatiquement
```

### MySQL/PostgreSQL (Types stricts)

```sql
-- MySQL
CREATE TABLE exemple (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prix DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- PostgreSQL
CREATE TABLE exemple (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prix NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Différences clés :**
- **SQLite** : Types flexibles, conversion automatique
- **MySQL/PostgreSQL** : Types stricts, taille définie

## Performance - Quand utiliser quoi ?

### 🚀 SQLite est plus rapide pour :

**Lectures simples :**
```sql
-- Cette requête sera très rapide avec SQLite
SELECT * FROM users WHERE id = 123;
```

**Applications mono-utilisateur :**
- Applications de bureau
- Applications mobiles
- Outils d'analyse personnels

### 🏆 MySQL/PostgreSQL sont plus rapides pour :

**Requêtes complexes avec gros volumes :**
```sql
-- Cette requête sera plus efficace sur MySQL/PostgreSQL
SELECT u.nom, COUNT(c.id) as nb_commandes, SUM(c.total) as ca
FROM users u
LEFT JOIN commandes c ON u.id = c.user_id
WHERE u.created_at > '2023-01-01'
GROUP BY u.id
HAVING nb_commandes > 10
ORDER BY ca DESC;
```

**Applications multi-utilisateurs :**
- Sites web avec trafic
- Applications d'entreprise
- Systèmes transactionnels

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
- Croissance du nombre d'utilisateurs
- Besoin de fonctionnalités avancées
- Exigences de performance

**Exemple de migration :**
```sql
-- SQLite
CREATE TABLE users (id INTEGER PRIMARY KEY, nom TEXT);

-- Devient en MySQL :
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(255) NOT NULL
);

-- Ou en PostgreSQL :
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(255) NOT NULL
);
```

### De MySQL/PostgreSQL vers SQLite

**Raisons typiques :**
- Simplification de l'architecture
- Application mobile ou embarquée
- Réduction des coûts d'infrastructure

## Avantages et inconvénients résumés

### 🟢 Avantages de SQLite
- **Simplicité** : Zéro configuration, ça marche tout de suite
- **Portabilité** : Un fichier = toute la base
- **Performance** : Très rapide pour les lectures
- **Fiabilité** : Pas de serveur qui peut tomber
- **Coût** : Gratuit, pas d'infrastructure

### 🔴 Inconvénients de SQLite
- **Concurrence** : Une seule écriture à la fois
- **Réseau** : Pas d'accès distant natif
- **Scalabilité** : Limites sur très gros volumes
- **Fonctionnalités** : Moins riche que PostgreSQL

### 🟢 Avantages MySQL/PostgreSQL
- **Concurrence** : Nombreux utilisateurs simultanés
- **Scalabilité** : Gère de très gros volumes
- **Fonctionnalités** : Riches et étendues
- **Écosystème** : Nombreux outils et extensions

### 🔴 Inconvénients MySQL/PostgreSQL
- **Complexité** : Installation et configuration
- **Maintenance** : Sauvegardes, mises à jour, monitoring
- **Infrastructure** : Serveur dédié nécessaire
- **Courbe d'apprentissage** : Plus difficile à maîtriser

## 🎯 Exercice pratique

**Scénario :** Votre ami vous demande conseil pour choisir une base de données pour ses projets. Aidez-le à choisir entre SQLite, MySQL et PostgreSQL :

1. **Application mobile de notes personnelles**
   - 1 utilisateur
   - Stockage local
   - Synchronisation cloud occasionnelle

2. **Site e-commerce pour PME**
   - 50-200 visiteurs simultanés
   - Catalogue de 10 000 produits
   - Commandes en ligne

3. **Prototype d'application d'analyse**
   - Phase de test
   - 1 développeur
   - Données à analyser (CSV)

4. **Système de gestion d'entreprise**
   - 100 employés
   - Données critiques
   - Rapports complexes

**Réponses :**
1. **SQLite** - Parfait pour une app mobile mono-utilisateur
2. **MySQL** - Idéal pour un site e-commerce classique
3. **SQLite** - Excellent pour prototyper rapidement
4. **PostgreSQL** - Fonctionnalités avancées pour l'entreprise

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
- Nombre d'utilisateurs simultanés
- Volume de données
- Complexité des besoins
- Infrastructure disponible

---

**💡 Dans le prochain chapitre**, nous explorerons en détail l'architecture serverless de SQLite et ce que signifie concrètement le concept de "fichier de base unique".

⏭️
