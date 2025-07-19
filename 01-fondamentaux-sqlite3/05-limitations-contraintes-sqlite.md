🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.5 Limitations et contraintes de SQLite

## Introduction - Pourquoi parler des limitations ?

Comprendre les limitations de SQLite n'est pas négatif, c'est **essentiel** ! Connaître les limites d'un outil vous permet de :
- **Faire des choix éclairés** pour vos projets
- **Éviter les mauvaises surprises** en cours de développement
- **Optimiser** votre utilisation de SQLite
- **Anticiper** les moments où migrer vers autre chose

> **Pensée positive** : Chaque limitation de SQLite correspond à un choix de design qui lui donne ses qualités uniques. Ces "limites" sont souvent le prix de sa simplicité !

## Vue d'ensemble des principales limitations

### 📊 Résumé visuel

| Domaine | Limitation | Impact | Solution alternative |
|---------|------------|--------|---------------------|
| **Concurrence** | 1 seule écriture à la fois | Sites web multi-utilisateurs | MySQL/PostgreSQL |
| **Réseau** | Accès local uniquement | Applications distribuées | MongoDB, PostgreSQL |
| **Taille** | Recommandé < 1TB | Big Data | PostgreSQL, Oracle |
| **Types** | 5 types seulement | Données complexes | PostgreSQL |
| **Fonctionnalités** | Pas de procédures stockées | Logique métier complexe | MySQL/PostgreSQL |

## 1. Limitations de concurrence

### 🚦 Le verrou d'écriture unique

**Principe :** SQLite utilise un système de verrous simple mais restrictif.

```
Lecture  : ✅ Plusieurs utilisateurs simultanés
Écriture : ❌ Un seul utilisateur à la fois
```

### Démonstration pratique

```bash
# Terminal 1
sqlite3 test_concurrence.db
```

```sql
-- Terminal 1 : Commencer une transaction
BEGIN;
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
-- Résultat : "database is locked"
```

```sql
-- Terminal 1 : Finir la transaction
COMMIT;
-- Maintenant Terminal 2 peut écrire
```

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

**❌ Problématique pour :**
```python
# Site web (plusieurs utilisateurs simultanés)
@app.route('/commande', methods=['POST'])
def nouvelle_commande():
    # Si 10 personnes commandent en même temps...
    # 9 vont avoir "database is locked" !
    conn = sqlite3.connect('boutique.db')
    # ... problème de concurrence
```

### 🔧 Solutions et contournements

**Pour améliorer la concurrence :**

```sql
-- Utiliser WAL mode (Write-Ahead Logging)
PRAGMA journal_mode = WAL;

-- Réduire les timeouts de transaction
PRAGMA busy_timeout = 30000;  -- 30 secondes

-- Optimiser les transactions
BEGIN IMMEDIATE;  -- Plus agressif
INSERT INTO table VALUES (...);
COMMIT;
```

**Résultat avec WAL :**
- ✅ Lectures simultanées avec écritures
- ✅ Meilleures performances de concurrence
- ❌ Toujours 1 seule écriture à la fois

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

**1. Partage de fichiers réseau (DÉCONSEILLÉ)**
```bash
# Partage Windows/SMB
sqlite3 "//serveur/partage/base.db"

# Partage NFS Linux
sqlite3 "/mnt/nfs/base.db"
```

**Problèmes :**
- ❌ Corruption possible des données
- ❌ Performance dégradée
- ❌ Verrous de fichiers problématiques

**2. API REST avec SQLite en backend**
```python
# Serveur avec SQLite local
from flask import Flask, jsonify
import sqlite3

app = Flask(__name__)

@app.route('/users')
def get_users():
    conn = sqlite3.connect('local.db')
    users = conn.execute("SELECT * FROM users").fetchall()
    conn.close()
    return jsonify(users)

# Clients se connectent via HTTP, pas directement à SQLite
```

**3. Synchronisation bidirectionnelle**
```bash
# Chaque client a sa copie locale
# Synchronisation périodique via rsync, git, ou outil custom
rsync -av base.db serveur:/backup/
```

## 3. Limitations de taille

### 📏 Limites théoriques vs pratiques

**Limites théoriques :**
- Taille de base : 281 TB (2^48 bytes)
- Taille de chaîne : 1 milliard de caractères
- Nombre de colonnes : 2000 par table
- Taille de requête : 1 million de caractères

**Limites pratiques recommandées :**
- Taille de base : < 1 TB pour de bonnes performances
- Nombre d'enregistrements : < 100 millions par table
- Taille d'enregistrement : < 100 MB par row

### 📊 Performance selon la taille

```sql
-- Test de performance simple
CREATE TABLE test_taille (
    id INTEGER PRIMARY KEY,
    donnees TEXT
);

-- 1000 enregistrements : ⚡ Très rapide
-- 100,000 enregistrements : ✅ Rapide
-- 10,000,000 enregistrements : ⚠️ Acceptable
-- 100,000,000 enregistrements : ❌ Lent
```

### 🎯 Cas d'usage selon la taille

**✅ Excellent pour (< 1 GB) :**
- Applications mobiles
- Bases de configuration
- Caches locaux
- Prototypes et développement

**⚠️ Acceptable (1-10 GB) :**
- Applications d'analyse de données
- Sites web à trafic modéré
- Applications métier locales

**❌ Pas recommandé (> 100 GB) :**
- Entrepôts de données
- Applications avec croissance massive
- Big Data et analytics

## 4. Limitations des types de données

### 🏷️ Seulement 5 types de base

SQLite ne connaît que :

```sql
CREATE TABLE exemple (
    id INTEGER,      -- Nombres entiers
    nom TEXT,        -- Chaînes de caractères
    prix REAL,       -- Nombres décimaux
    photo BLOB,      -- Données binaires
    actif NULL       -- Valeur nulle
);
```

### ❌ Types absents (comparé à PostgreSQL)

```sql
-- Ces types n'existent PAS dans SQLite :

-- Types de dates sophistiqués
birthday DATE,                    -- Pas de type DATE natif
created_at TIMESTAMP,             -- Pas de TIMESTAMP natif

-- Types numériques précis
prix DECIMAL(10,2),               -- Pas de DECIMAL exact
pourcentage NUMERIC(5,2),         -- Approximation avec REAL

-- Types avancés
coordonnees POINT,                -- Pas de géolocalisation native
config JSON,                      -- JSON supporté mais limité
tableau ARRAY,                    -- Pas de tableaux
```

### 🔧 Solutions et contournements

**Dates et heures :**
```sql
-- Stocker comme TEXT (ISO 8601)
CREATE TABLE events (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    date_event TEXT DEFAULT (datetime('now'))  -- '2024-01-15 14:30:00'
);

-- Stocker comme INTEGER (timestamp Unix)
CREATE TABLE logs (
    id INTEGER PRIMARY KEY,
    message TEXT,
    timestamp INTEGER DEFAULT (strftime('%s', 'now'))  -- 1705329000
);

-- Fonctions de conversion
SELECT datetime(timestamp, 'unixepoch') FROM logs;
SELECT strftime('%s', '2024-01-15 14:30:00');
```

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

**JSON (support limité) :**
```sql
-- SQLite 3.45+ supporte JSON
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    preferences TEXT  -- Stocker JSON comme TEXT
);

-- Fonctions JSON disponibles
INSERT INTO users (nom, preferences) VALUES
    ('Alice', '{"theme": "dark", "lang": "fr"}');

SELECT nom, json_extract(preferences, '$.theme') as theme
FROM users;
```

## 5. Limitations fonctionnelles

### 🚫 Fonctionnalités manquantes

**Pas de procédures stockées :**
```sql
-- Impossible dans SQLite
CREATE PROCEDURE calculer_total(user_id INT)
BEGIN
    -- logique complexe
END;
```

**Solution :** Logique dans l'application
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

**Pas de RIGHT JOIN :**
```sql
-- Impossible dans SQLite
SELECT * FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- Solution : Inverser en LEFT JOIN
SELECT * FROM orders o
LEFT JOIN users u ON o.user_id = u.id;
```

**Pas de CHECK contraintes complexes :**
```sql
-- Limité dans SQLite
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    prix REAL CHECK (prix > 0),           -- ✅ Simple OK
    dates_promo TEXT CHECK (              -- ❌ Complexe impossible
        date(debut_promo) < date(fin_promo)
    )
);
```

### 🔧 ALTER TABLE limité

```sql
-- SQLite ne permet que :
ALTER TABLE users ADD COLUMN phone TEXT;        -- ✅ Ajouter colonne
ALTER TABLE users RENAME TO customers;          -- ✅ Renommer table
ALTER TABLE users RENAME COLUMN nom TO name;    -- ✅ Renommer colonne

-- Impossible :
ALTER TABLE users DROP COLUMN email;            -- ❌ Supprimer colonne
ALTER TABLE users MODIFY COLUMN age BIGINT;     -- ❌ Modifier type
ALTER TABLE users ADD CONSTRAINT fk_...;        -- ❌ Ajouter contrainte
```

**Solution pour supprimer une colonne :**
```sql
-- Méthode complète mais fastidieuse
BEGIN;

-- 1. Créer nouvelle table sans la colonne
CREATE TABLE users_new (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    -- email supprimé
    age INTEGER
);

-- 2. Copier les données
INSERT INTO users_new SELECT id, nom, age FROM users;

-- 3. Supprimer ancienne table
DROP TABLE users;

-- 4. Renommer la nouvelle
ALTER TABLE users_new RENAME TO users;

COMMIT;
```

## 6. Limitations de déploiement

### 🏢 Pas pour l'entreprise traditionnelle

**Manque pour l'entreprise :**
- ❌ Pas de gestion d'utilisateurs/rôles
- ❌ Pas de réplication master/slave
- ❌ Pas de partitioning automatique
- ❌ Pas de haute disponibilité
- ❌ Monitoring limité

**SQLite ne remplace PAS :**
- Serveurs de base de données d'entreprise
- Solutions haute disponibilité
- Systèmes transactionnels critiques
- Applications bancaires/financières

## 7. Quand migrer vers autre chose ?

### 🚨 Signaux d'alarme

**Trafic :**
```bash
# Si vos logs montrent souvent :
"database is locked"
"database busy"

# Ou si vous avez :
> 50 utilisateurs simultanés
> 100 requêtes d'écriture/minute
```

**Taille :**
```bash
# Vérifiez la taille de votre base
ls -lh *.db

# Si > 10GB et croissance rapide
# Considérez PostgreSQL
```

**Complexité :**
```sql
-- Si vous vous retrouvez à faire souvent :
-- Requêtes avec 10+ JOINs
-- Logique métier complexe dans l'app
-- Calculs lourds répétitifs
```

### 🔄 Stratégies de migration

**Migration graduelle :**
```python
# Phase 1 : Garde SQLite pour les données locales
# Phase 2 : PostgreSQL pour les données partagées
# Phase 3 : Migration complète si nécessaire

class DatabaseRouter:
    def route_query(self, query_type, table):
        if table in ['cache', 'preferences', 'logs']:
            return sqlite_connection
        else:
            return postgresql_connection
```

## 🎯 Exercice pratique - Identifier les limitations

**Scénario :** Analysez ces cas d'usage et identifiez les limitations de SQLite :

### Cas 1 : Blog personnel
```python
# 1 auteur, 100 articles, 1000 visiteurs/jour
def publier_article(titre, contenu):
    conn = sqlite3.connect('blog.db')
    conn.execute("INSERT INTO articles (titre, contenu) VALUES (?, ?)",
                (titre, contenu))
    conn.commit()
```
**Question :** SQLite convient-il ? Pourquoi ?

### Cas 2 : Boutique en ligne
```python
# 50 commandes simultanées possibles
def passer_commande(user_id, produits):
    conn = sqlite3.connect('boutique.db')
    conn.execute("BEGIN")
    for produit in produits:
        conn.execute("UPDATE stock SET quantite = quantite - ? WHERE id = ?",
                    (produit.qte, produit.id))
    conn.execute("INSERT INTO commandes ...")
    conn.commit()
```
**Question :** Quels problèmes peut-il y avoir ?

### Cas 3 : Application mobile
```swift
// iPhone app, 1 utilisateur, données locales
func sauvegarder_photo(image_data, metadata) {
    let db = SQLite.open("photos.db")
    db.execute("INSERT INTO photos (data, metadata) VALUES (?, ?)",
               image_data, metadata)
}
```
**Question :** SQLite est-il adapté ?

**Réponses :**

1. **Blog personnel** : ✅ Parfait ! 1 auteur = pas de concurrence, taille raisonnable
2. **Boutique en ligne** : ❌ Problème de concurrence sur les stocks, risque de "database locked"
3. **Application mobile** : ✅ Cas d'usage idéal pour SQLite

## Récapitulatif des limitations

### 🎯 Les "vraies" limitations

**Concurrence :**
- 1 seule écriture simultanée
- Peut bloquer sur forte charge

**Réseau :**
- Accès local uniquement
- Pas de protocole réseau natif

**Fonctionnalités :**
- Types de données limités
- Moins de fonctions SQL avancées

### 💡 Les "fausses" limitations

**Taille :** 281TB théorique, 1TB pratique = largement suffisant pour 90% des cas

**Performance :** Souvent plus rapide que MySQL/PostgreSQL sur les petites requêtes

**Simplicité :** Ce n'est pas une limitation, c'est un avantage !

### 🎯 Règle d'or

> **SQLite est parfait jusqu'à ce qu'il ne le soit plus.** Quand vous atteignez ses limites, elles sont généralement évidentes. Jusque-là, profitez de sa simplicité !

---

**🎉 Félicitations !** Vous avez maintenant une vision complète des fondamentaux de SQLite. Dans le module suivant, nous commencerons à pratiquer avec les bases du langage SQL dans SQLite.

**💡 Points clés à retenir :**
- Chaque limitation correspond à un choix de design
- SQLite excelle dans 90% des cas d'usage
- Connaître les limites aide à faire les bons choix
- La migration est possible quand nécessaire

⏭️
