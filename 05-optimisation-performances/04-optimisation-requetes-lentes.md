🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4 Optimisation des requêtes lentes

## Introduction : Identifier et résoudre les requêtes lentes

Une requête lente, c'est comme un embouteillage sur l'autoroute : cela ralentit toute votre application et frustre vos utilisateurs. Dans cette section, nous allons apprendre à identifier ces "embouteillages" et à les résoudre efficacement.

### Qu'est-ce qu'une requête lente ?

**Critères généraux :**
- Plus de 100ms pour une requête simple
- Plus de 1 seconde pour une requête complexe
- Temps qui augmente de façon exponentielle avec les données

**Impact sur l'application :**
- Interface qui "freeze"
- Utilisateurs qui abandonnent
- Serveur surchargé

## Méthodologie d'optimisation

### Étape 1 : Mesurer et identifier

**Activer le timing :**
```sql
.timer on
```

**Identifier les requêtes problématiques :**
```sql
-- Exemple de requête lente
SELECT c.nom, COUNT(*) as nb_commandes, SUM(co.total) as chiffre_affaire
FROM clients c
LEFT JOIN commandes co ON c.id = co.client_id
WHERE c.ville = 'Paris'
GROUP BY c.id, c.nom
ORDER BY chiffre_affaire DESC;

-- Temps : 2.5 secondes → problématique !
```

### Étape 2 : Analyser le plan d'exécution

```sql
EXPLAIN QUERY PLAN
SELECT c.nom, COUNT(*) as nb_commandes, SUM(co.total) as chiffre_affaire
FROM clients c
LEFT JOIN commandes co ON c.id = co.client_id
WHERE c.ville = 'Paris'
GROUP BY c.id, c.nom
ORDER BY chiffre_affaire DESC;
```

**Résultat problématique :**
```
SCAN c
SCAN co
USE TEMP B-TREE FOR GROUP BY
USE TEMP B-TREE FOR ORDER BY
```

**Problèmes identifiés :**
1. Double SCAN (très lent)
2. Deux structures temporaires (gourmand en mémoire)

### Étape 3 : Appliquer les optimisations

Nous allons voir différentes techniques dans les sections suivantes.

## Optimisation des clauses WHERE

### Problème : Conditions non indexées

```sql
-- ❌ Lent : pas d'index sur ville
SELECT * FROM clients WHERE ville = 'Paris';
-- Plan : SCAN clients
-- Temps : 500ms sur 100 000 clients
```

**Solution :**
```sql
-- ✅ Créer un index
CREATE INDEX idx_ville ON clients(ville);

-- Maintenant :
-- Plan : SEARCH clients USING INDEX idx_ville (ville=?)
-- Temps : 5ms → 100x plus rapide !
```

### Problème : Conditions avec fonctions

```sql
-- ❌ Très lent : fonction dans WHERE
SELECT * FROM clients WHERE UPPER(nom) = 'DUPONT';
-- Plan : SCAN clients (index inutilisable)
```

**Solutions :**

**Option 1 : Index sur expression**
```sql
CREATE INDEX idx_nom_upper ON clients(UPPER(nom));
SELECT * FROM clients WHERE UPPER(nom) = 'DUPONT';
-- Plan : SEARCH clients USING INDEX idx_nom_upper
```

**Option 2 : Recherche insensible à la casse**
```sql
-- Plus simple et souvent suffisant
SELECT * FROM clients WHERE nom = 'Dupont' COLLATE NOCASE;
```

**Option 3 : Normaliser les données**
```sql
-- Ajouter une colonne normalisée
ALTER TABLE clients ADD COLUMN nom_normalise TEXT;
UPDATE clients SET nom_normalise = UPPER(nom);
CREATE INDEX idx_nom_norm ON clients(nom_normalise);
```

### Problème : Conditions avec LIKE inefficaces

```sql
-- ❌ Très lent : LIKE avec % au début
SELECT * FROM clients WHERE nom LIKE '%dupont%';
-- Plan : SCAN clients (index totalement inutilisable)
```

**Solutions :**

**Pour recherche par préfixe :**
```sql
-- ✅ Efficace : LIKE avec % à la fin seulement
SELECT * FROM clients WHERE nom LIKE 'Dup%';
-- Plan : SEARCH clients USING INDEX idx_nom (nom>? AND nom<?)
```

**Pour recherche full-text :**
```sql
-- ✅ Utiliser FTS (Full-Text Search)
CREATE VIRTUAL TABLE clients_fts USING fts5(nom, email, content='clients');
INSERT INTO clients_fts SELECT nom, email FROM clients;

-- Recherche full-text très rapide
SELECT * FROM clients_fts WHERE clients_fts MATCH 'dupont';
```

## Optimisation des JOIN

### Problème : JOIN sans index

```sql
-- Tables d'exemple
CREATE TABLE commandes (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,  -- Pas d'index !
    total REAL,
    date_commande DATE
);

-- ❌ JOIN très lent
EXPLAIN QUERY PLAN
SELECT c.nom, co.total
FROM clients c
JOIN commandes co ON c.id = co.client_id
WHERE c.ville = 'Paris';
```

**Résultat problématique :**
```
SEARCH c USING INDEX idx_ville (ville=?)
SCAN co
```

**Solution :**
```sql
-- ✅ Index sur la clé de jointure
CREATE INDEX idx_client_id ON commandes(client_id);

-- Nouveau plan :
-- SEARCH c USING INDEX idx_ville (ville=?)
-- SEARCH co USING INDEX idx_client_id (client_id=?)
```

### Optimisation de l'ordre des JOIN

```sql
-- Table avec beaucoup de lignes
CREATE TABLE logs (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    action TEXT,
    date_log DATETIME
);

-- ❌ Ordre suboptimal
SELECT u.nom, l.action
FROM logs l  -- Table énorme en premier !
JOIN users u ON l.user_id = u.id
WHERE u.ville = 'Paris' AND l.date_log >= '2024-01-01';
```

**Optimisation :**
```sql
-- ✅ Commencer par la table la plus sélective
SELECT u.nom, l.action
FROM users u  -- Table plus petite et filtrée en premier
JOIN logs l ON u.id = l.user_id
WHERE u.ville = 'Paris' AND l.date_log >= '2024-01-01';

-- Avec les bons index :
CREATE INDEX idx_users_ville ON users(ville);
CREATE INDEX idx_logs_date_user ON logs(date_log, user_id);
```

### Transformer les sous-requêtes en JOIN

```sql
-- ❌ Sous-requête lente
SELECT nom FROM clients
WHERE id IN (
    SELECT client_id FROM commandes
    WHERE total > 1000
);
-- Plan : SCAN clients + LIST SUBQUERY (SCAN commandes)
```

**Optimisation :**
```sql
-- ✅ JOIN équivalent mais plus rapide
SELECT DISTINCT c.nom
FROM clients c
JOIN commandes co ON c.id = co.client_id
WHERE co.total > 1000;

-- Avec index :
CREATE INDEX idx_commandes_total ON commandes(total, client_id);
-- Plan : SEARCH commandes + SEARCH clients
```

## Optimisation des ORDER BY et GROUP BY

### Problème : Tri sans index

```sql
-- ❌ Tri coûteux
SELECT nom, salaire FROM employes ORDER BY salaire DESC;
-- Plan : SCAN employes + USE TEMP B-TREE FOR ORDER BY
-- Temps : 800ms sur 50 000 employés
```

**Solution :**
```sql
-- ✅ Index pour le tri
CREATE INDEX idx_salaire_desc ON employes(salaire DESC);

-- Nouveau plan :
-- SCAN employes USING INDEX idx_salaire_desc
-- Temps : 50ms → 16x plus rapide !
```

### Optimisation des GROUP BY

```sql
-- ❌ GROUP BY lent
SELECT departement, AVG(salaire)
FROM employes
GROUP BY departement;
-- Plan : SCAN employes + USE TEMP B-TREE FOR GROUP BY
```

**Solution :**
```sql
-- ✅ Index sur la colonne de groupement
CREATE INDEX idx_departement ON employes(departement);

-- Encore mieux : index composite
CREATE INDEX idx_dept_salaire ON employes(departement, salaire);
-- Peut calculer AVG(salaire) directement depuis l'index !
```

### Optimisation des requêtes avec LIMIT

```sql
-- ❌ LIMIT inefficace sans index
SELECT * FROM employes ORDER BY date_embauche DESC LIMIT 10;
-- Plan : SCAN + TEMP B-TREE + tri de TOUS les employés !
```

**Solution :**
```sql
-- ✅ Index sur la colonne de tri
CREATE INDEX idx_date_embauche ON employes(date_embauche DESC);

-- Plan optimisé :
-- SEARCH employes USING INDEX idx_date_embauche LIMIT 10
-- Prend directement les 10 premiers !
```

## Optimisation des requêtes complexes

### Cas réel : Tableau de bord e-commerce

```sql
-- ❌ Requête très lente (5+ secondes)
SELECT
    c.nom,
    c.email,
    COUNT(co.id) as nb_commandes,
    SUM(co.total) as chiffre_affaire,
    MAX(co.date_commande) as derniere_commande,
    AVG(co.total) as panier_moyen
FROM clients c
LEFT JOIN commandes co ON c.id = co.client_id
WHERE c.date_inscription >= '2024-01-01'
  AND c.ville IN ('Paris', 'Lyon', 'Marseille')
  AND (co.date_commande IS NULL OR co.date_commande >= '2024-01-01')
GROUP BY c.id, c.nom, c.email
HAVING nb_commandes > 0 OR c.date_inscription >= '2024-06-01'
ORDER BY chiffre_affaire DESC NULLS LAST
LIMIT 100;
```

**Analyse du plan (avant optimisation) :**
```sql
EXPLAIN QUERY PLAN [requête ci-dessus];
```

**Résultat problématique :**
```
SCAN c
SCAN co
USE TEMP B-TREE FOR GROUP BY
USE TEMP B-TREE FOR ORDER BY
```

### Optimisation étape par étape

**Étape 1 : Index pour les filtres WHERE**
```sql
-- Index composite pour les filtres sur clients
CREATE INDEX idx_clients_optim ON clients(date_inscription, ville);

-- Index pour le filtre sur commandes
CREATE INDEX idx_commandes_date ON commandes(date_commande, client_id);
```

**Étape 2 : Optimiser le GROUP BY**
```sql
-- Index pour améliorer le groupement
CREATE INDEX idx_clients_group ON clients(id, nom, email);
```

**Étape 3 : Optimiser l'ORDER BY**
```sql
-- Pour cela, on peut créer une vue matérialisée ou repenser la requête
-- Alternative : index sur une colonne calculée
```

**Étape 4 : Réécriture de la requête**
```sql
-- ✅ Version optimisée avec CTE
WITH stats_clients AS (
    SELECT
        client_id,
        COUNT(*) as nb_commandes,
        SUM(total) as chiffre_affaire,
        MAX(date_commande) as derniere_commande,
        AVG(total) as panier_moyen
    FROM commandes
    WHERE date_commande >= '2024-01-01'
    GROUP BY client_id
)
SELECT
    c.nom,
    c.email,
    COALESCE(s.nb_commandes, 0) as nb_commandes,
    COALESCE(s.chiffre_affaire, 0) as chiffre_affaire,
    s.derniere_commande,
    s.panier_moyen
FROM clients c
LEFT JOIN stats_clients s ON c.id = s.client_id
WHERE c.date_inscription >= '2024-01-01'
  AND c.ville IN ('Paris', 'Lyon', 'Marseille')
  AND (s.nb_commandes > 0 OR c.date_inscription >= '2024-06-01')
ORDER BY COALESCE(s.chiffre_affaire, 0) DESC
LIMIT 100;
```

**Résultat :** Temps passé de 5 secondes à 200ms !

## Techniques avancées d'optimisation

### 1. Requêtes avec EXISTS vs IN

```sql
-- ❌ IN peut être lent sur grandes listes
SELECT nom FROM clients
WHERE id IN (SELECT client_id FROM commandes WHERE total > 1000);
```

```sql
-- ✅ EXISTS souvent plus efficace
SELECT nom FROM clients c
WHERE EXISTS (
    SELECT 1 FROM commandes co
    WHERE co.client_id = c.id AND co.total > 1000
);
```

### 2. UNION vs OR

```sql
-- ❌ OR peut empêcher l'utilisation d'index
SELECT * FROM employes
WHERE departement = 'IT' OR salaire > 80000;
-- Peut forcer un SCAN même avec des index
```

```sql
-- ✅ UNION peut utiliser les index séparément
SELECT * FROM employes WHERE departement = 'IT'
UNION
SELECT * FROM employes WHERE salaire > 80000;
```

### 3. Dénormalisation sélective

```sql
-- Si cette requête est très fréquente :
SELECT c.nom, COUNT(co.id) as nb_commandes
FROM clients c
LEFT JOIN commandes co ON c.id = co.client_id
GROUP BY c.id, c.nom;

-- ✅ Ajouter une colonne dénormalisée
ALTER TABLE clients ADD COLUMN nb_commandes INTEGER DEFAULT 0;

-- Maintenir avec des triggers
CREATE TRIGGER update_nb_commandes
AFTER INSERT ON commandes
BEGIN
    UPDATE clients
    SET nb_commandes = nb_commandes + 1
    WHERE id = NEW.client_id;
END;
```

## Cas pratiques d'optimisation

### Cas 1 : Recherche avec pagination

**Problème initial :**
```sql
-- ❌ Pagination inefficace
SELECT * FROM produits
WHERE categorie = 'electronique'
ORDER BY prix DESC
LIMIT 20 OFFSET 1000;
-- OFFSET élévé = très lent !
```

**Solution optimisée :**
```sql
-- ✅ Pagination par curseur
-- Première page
SELECT * FROM produits
WHERE categorie = 'electronique'
ORDER BY prix DESC, id DESC
LIMIT 20;

-- Pages suivantes (avec le dernier prix/id de la page précédente)
SELECT * FROM produits
WHERE categorie = 'electronique'
  AND (prix < 299.99 OR (prix = 299.99 AND id < 12345))
ORDER BY prix DESC, id DESC
LIMIT 20;

-- Index nécessaire :
CREATE INDEX idx_pagination ON produits(categorie, prix DESC, id DESC);
```

### Cas 2 : Calcul de statistiques

**Problème initial :**
```sql
-- ❌ Recalcul à chaque fois
SELECT
    AVG(total) as panier_moyen,
    COUNT(*) as nb_commandes,
    SUM(total) as chiffre_affaire
FROM commandes
WHERE date_commande >= date('now', '-30 days');
-- Recalcule 30 jours de données à chaque requête !
```

**Solution optimisée :**
```sql
-- ✅ Table de statistiques pré-calculées
CREATE TABLE stats_quotidiennes (
    date_stat DATE PRIMARY KEY,
    nb_commandes INTEGER,
    chiffre_affaire REAL,
    panier_moyen REAL
);

-- Calcul incrémental quotidien
INSERT OR REPLACE INTO stats_quotidiennes
SELECT
    date_commande,
    COUNT(*),
    SUM(total),
    AVG(total)
FROM commandes
WHERE date_commande = date('now', '-1 day')
GROUP BY date_commande;

-- Requête ultra-rapide
SELECT
    AVG(panier_moyen) as panier_moyen,
    SUM(nb_commandes) as nb_commandes,
    SUM(chiffre_affaire) as chiffre_affaire
FROM stats_quotidiennes
WHERE date_stat >= date('now', '-30 days');
```

## Outils de diagnostic et monitoring

### 1. Activer le profiling

```sql
-- Timing détaillé
.timer on

-- Statistiques d'exécution
.stats on

-- Plan d'exécution automatique
.eqp on
```

### 2. Analyser l'utilisation des ressources

```sql
-- Voir l'utilisation du cache
PRAGMA cache_size;
PRAGMA page_count;
PRAGMA page_size;

-- Statistiques de la base
.dbinfo
```

### 3. Logging des requêtes lentes

```python
# Exemple en Python pour logger les requêtes lentes
import sqlite3
import time

class SlowQueryConnection:
    def __init__(self, db_path, slow_threshold=0.1):
        self.conn = sqlite3.connect(db_path)
        self.slow_threshold = slow_threshold

    def execute_with_timing(self, query, params=None):
        start_time = time.time()

        if params:
            result = self.conn.execute(query, params)
        else:
            result = self.conn.execute(query)

        execution_time = time.time() - start_time

        if execution_time > self.slow_threshold:
            print(f"SLOW QUERY ({execution_time:.3f}s): {query}")

        return result
```

## Checklist d'optimisation

### Avant d'optimiser

✅ **Identifier** les requêtes réellement lentes (mesurer !)
✅ **Comprendre** le contexte métier (fréquence d'exécution)
✅ **Analyser** le plan d'exécution actuel
✅ **Estimer** le gain potentiel

### Techniques par ordre de priorité

1. **Index sur colonnes WHERE** (gain : 10-100x)
2. **Index sur colonnes ORDER BY** (gain : 5-50x)
3. **Optimisation des JOIN** (gain : 2-20x)
4. **Réécriture de requêtes** (gain : 1.5-10x)
5. **Dénormalisation sélective** (gain : 2-5x)

### Après optimisation

✅ **Mesurer** l'amélioration réelle
✅ **Vérifier** que les autres requêtes ne sont pas impactées
✅ **Documenter** les changements
✅ **Monitorer** les performances en production

## Exercices pratiques

### Exercice 1 : Optimisation d'un forum

```sql
-- Schema d'un forum
CREATE TABLE utilisateurs (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    email TEXT UNIQUE,
    ville TEXT,
    date_inscription DATE
);

CREATE TABLE topics (
    id INTEGER PRIMARY KEY,
    titre TEXT,
    auteur_id INTEGER,
    categorie TEXT,
    date_creation DATETIME,
    nb_vues INTEGER DEFAULT 0
);

CREATE TABLE messages (
    id INTEGER PRIMARY KEY,
    topic_id INTEGER,
    auteur_id INTEGER,
    contenu TEXT,
    date_message DATETIME
);

-- Requêtes à optimiser :

-- 1. Page d'accueil : derniers topics actifs
SELECT t.titre, u.nom, COUNT(m.id) as nb_messages, MAX(m.date_message) as derniere_activite
FROM topics t
JOIN utilisateurs u ON t.auteur_id = u.id
LEFT JOIN messages m ON t.id = m.topic_id
GROUP BY t.id, t.titre, u.nom
ORDER BY derniere_activite DESC
LIMIT 20;

-- 2. Recherche d'utilisateurs par ville
SELECT nom, email FROM utilisateurs WHERE ville = 'Paris';

-- 3. Messages d'un topic avec pagination
SELECT m.contenu, u.nom, m.date_message
FROM messages m
JOIN utilisateurs u ON m.auteur_id = u.id
WHERE m.topic_id = 123
ORDER BY m.date_message
LIMIT 10 OFFSET 50;
```

**Mission :** Analysez et optimisez chaque requête.

### Exercice 2 : E-commerce analytics

```sql
-- Requête analytics complexe à optimiser
SELECT
    p.nom as produit,
    p.categorie,
    COUNT(DISTINCT c.id) as nb_clients_uniques,
    COUNT(ci.id) as nb_ventes,
    SUM(ci.quantite * ci.prix_unitaire) as chiffre_affaire,
    AVG(ci.quantite * ci.prix_unitaire) as panier_moyen
FROM produits p
JOIN commande_items ci ON p.id = ci.produit_id
JOIN commandes c ON ci.commande_id = c.id
WHERE c.date_commande >= '2024-01-01'
  AND c.statut = 'validee'
  AND p.categorie IN ('electronique', 'informatique')
GROUP BY p.id, p.nom, p.categorie
HAVING chiffre_affaire > 10000
ORDER BY chiffre_affaire DESC
LIMIT 50;
```

## Résumé et bonnes pratiques

### Règles d'or pour l'optimisation

✅ **Mesurez avant d'optimiser** - `.timer on` est votre ami
✅ **Un problème à la fois** - changement incrémental
✅ **Index sur colonnes WHERE** - priorité n°1
✅ **EXPLAIN QUERY PLAN** - votre boussole d'optimisation
✅ **Testez en conditions réelles** - données et charge représentatives

### Signaux d'alarme

🚨 **SCAN sur table > 10 000 lignes** → Index urgent
🚨 **Plusieurs TEMP B-TREE** → Repenser la requête
🚨 **Temps > 1 seconde** → Optimisation critique
🚨 **Performance dégradée avec croissance** → Architecture à revoir

### Ordre des optimisations

1. **Index simples** sur colonnes WHERE fréquentes
2. **Index composites** pour requêtes multi-critères
3. **Optimisation JOIN** et sous-requêtes
4. **Réécriture** de requêtes complexes
5. **Dénormalisation** pour cas spécifiques

**Objectif :** Transformer vos requêtes de tortues en fusées ! 🐢 → 🚀

La section suivante vous apprendra à configurer SQLite lui-même pour obtenir les meilleures performances possibles avec les paramètres PRAGMA.

⏭️
