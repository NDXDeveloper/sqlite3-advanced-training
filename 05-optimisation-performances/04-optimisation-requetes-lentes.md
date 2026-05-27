🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4 Optimisation des requêtes lentes

## Introduction : Identifier et résoudre les requêtes lentes

Une requête lente, c'est comme un embouteillage sur l'autoroute : cela ralentit toute votre application et frustre vos utilisateurs. Dans cette section, nous allons apprendre à identifier ces "embouteillages" et à les résoudre efficacement.

### Qu'est-ce qu'une requête lente ?

**Critères généraux :**
- Plus de 100ms pour une requête simple
- Plus de 1 seconde pour une requête complexe
- Temps qui croît de façon **linéaire** (`SCAN`, *O(n)*) ou **quadratique** (jointure sans index, *O(n × m)*) avec la taille des données — au lieu d'être quasi-constant grâce aux index (*O(log n)*)

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

**Option 2 : Recherche insensible à la casse (index `COLLATE NOCASE` requis)**
```sql
-- ⚠️ Le COLLATE de la requête doit MATCHER celui de l'index.
--    Un index sur `nom` simple n'est PAS utilisé pour une comparaison COLLATE NOCASE.
CREATE INDEX idx_nom_nocase ON clients(nom COLLATE NOCASE);  
SELECT * FROM clients WHERE nom = 'Dupont' COLLATE NOCASE;  
-- Plan : SEARCH clients USING INDEX idx_nom_nocase (nom=?)
```

**Option 3 : Normaliser les données dans une colonne dédiée**
```sql
-- Ajouter une colonne normalisée maintenue manuellement
ALTER TABLE clients ADD COLUMN nom_normalise TEXT;  
UPDATE clients SET nom_normalise = UPPER(nom);  
CREATE INDEX idx_nom_norm ON clients(nom_normalise);  
-- ⚠️ Penser à actualiser `nom_normalise` à chaque INSERT/UPDATE de `nom`
--    (via trigger, code applicatif, ou colonne GENERATED — voir option 4).
```

**Option 4 : Colonne générée VIRTUAL (calculée automatiquement)**
```sql
-- SQLite calcule UPPER(nom) à la volée, et l'index la matérialise
-- ⚠️ Avec ALTER TABLE, seul VIRTUAL est autorisé (pas STORED).
ALTER TABLE clients ADD COLUMN nom_upper TEXT
    GENERATED ALWAYS AS (UPPER(nom)) VIRTUAL;
CREATE INDEX idx_nom_upper_calc ON clients(nom_upper);  
SELECT * FROM clients WHERE nom_upper = 'DUPONT';  
```

### Problème : Conditions avec LIKE inefficaces

```sql
-- ❌ Très lent : LIKE avec % au début
SELECT * FROM clients WHERE nom LIKE '%dupont%';
-- Plan : SCAN clients (index totalement inutilisable)
```

**Solutions :**

**Pour recherche par préfixe :** ⚠️ piège vicieux, voir aussi l'anti-pattern n°3 plus bas dans cette section.

```sql
-- ❌ Naïvement : LIKE 'Dup%' avec un index « normal » → SCAN par défaut !
--    Raison : LIKE est case-insensitive par défaut, l'index BINARY case-sensitive.
CREATE INDEX idx_nom ON clients(nom);  
EXPLAIN QUERY PLAN SELECT * FROM clients WHERE nom LIKE 'Dup%';  
-- Plan : SCAN clients   ← inattendu !

-- ✅ Solution préférée : GLOB (toujours case-sensitive, utilise l'index BINARY)
EXPLAIN QUERY PLAN SELECT * FROM clients WHERE nom GLOB 'Dup*';
-- Plan : SEARCH clients USING COVERING INDEX idx_nom (nom>? AND nom<?)

-- ✅ Alternative : index COLLATE NOCASE qui matche la sémantique LIKE
CREATE INDEX idx_nom_nocase ON clients(nom COLLATE NOCASE);  
EXPLAIN QUERY PLAN SELECT * FROM clients WHERE nom LIKE 'Dup%';  
-- Plan : SEARCH clients USING COVERING INDEX idx_nom_nocase (nom>? AND nom<?)
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

> ℹ️ **Bonne nouvelle** : SQLite **réordonne automatiquement** les `JOIN INNER` selon les statistiques (`ANALYZE`) pour commencer par la table la plus sélective. L'ordre dans lequel vous écrivez les tables n'influence généralement **pas** le plan exécuté — `EXPLAIN QUERY PLAN` produit le même résultat dans les deux sens.

```sql
-- Ces deux écritures produisent le même plan (avec ANALYZE et bons index) :
SELECT u.nom, l.action  
FROM logs l                    -- "table énorme" en 1ʳᵉ position en écriture  
JOIN users u ON l.user_id = u.id  
WHERE u.ville = 'Paris' AND l.date_log >= '2024-01-01';  

SELECT u.nom, l.action  
FROM users u                   -- "table sélective" en 1ʳᵉ position  
JOIN logs l ON u.id = l.user_id  
WHERE u.ville = 'Paris' AND l.date_log >= '2024-01-01';  
```

**Ce qui compte vraiment** : les **index** disponibles, pas l'ordre d'écriture.

```sql
CREATE INDEX idx_users_ville ON users(ville);                -- filtre sélectif  
CREATE INDEX idx_logs_date_user ON logs(date_log, user_id);  -- filtre + jointure  
```

> ⚠️ **Exceptions** où l'ordre compte :  
> - `CROSS JOIN` : forçage explicite de l'ordre, SQLite respecte la séquence écrite  
> - Tables sans statistiques (pas de `ANALYZE`) : l'optimiseur utilise des heuristiques moins précises  
> - Requêtes avec sous-requêtes ou CTE matérialisées : l'ordre peut alors influencer

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
-- ✅ Index sur la colonne de tri (sens DESC pour matcher l'ORDER BY)
CREATE INDEX idx_date_embauche ON employes(date_embauche DESC);

-- Plan optimisé :
-- SCAN employes USING INDEX idx_date_embauche
-- ℹ️ "SCAN" est trompeur ici : SQLite parcourt l'index dans l'ordre déjà trié
--    et s'arrête après 10 lignes grâce au LIMIT — pas de TEMP B-TREE.
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

**Analyse du plan (avant optimisation) :** préfixer la requête ci-dessus avec `EXPLAIN QUERY PLAN`.

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

## Synthèse : les anti-patterns SQL à connaître

Cette section regroupe les **erreurs les plus fréquentes** qui font qu'un index existant n'est pas utilisé. Toutes ont été validées par exécution sous **SQLite 3.45.1**.

### 1. Fonction sur colonne indexée → SCAN forcé

```sql
-- ❌ ANTI-PATTERN : la fonction empêche l'usage de idx_categorie
SELECT * FROM produits WHERE UPPER(categorie) = 'ELECTRONIQUE';
-- Plan : SCAN produits

-- ✅ CORRECTION 1 : index sur l'expression elle-même
CREATE INDEX idx_categorie_upper ON produits(UPPER(categorie));

-- ✅ CORRECTION 2 : index COLLATE NOCASE + requête avec COLLATE NOCASE
CREATE INDEX idx_categorie_nocase ON produits(categorie COLLATE NOCASE);  
SELECT * FROM produits WHERE categorie = 'electronique' COLLATE NOCASE;  
```

**Variantes du piège** : `LOWER()`, `TRIM()`, `SUBSTR()`, `CAST()`, `date()`, `julianday()`, `strftime()`, opérateurs arithmétiques (`prix * 2`, `id + 1`)… **Toute** transformation de la colonne indexée à gauche du `=` bloque l'index.

### 2. LIKE avec joker en tête → SCAN forcé

```sql
-- ❌ ANTI-PATTERN : '%xxx' ou '%xxx%' = index inutilisable, même un index sur 'nom'
SELECT * FROM produits WHERE nom LIKE '%phone%';
-- Plan : SCAN produits
```

**Pour recherche infixe, deux vraies solutions :**

```sql
-- ✅ Solution 1 : FTS5 (recherche plein texte, module 6)
CREATE VIRTUAL TABLE produits_fts USING fts5(nom, content='produits', content_rowid='id');  
INSERT INTO produits_fts(produits_fts) VALUES('rebuild');  
SELECT * FROM produits_fts WHERE produits_fts MATCH 'phone';  
-- Plan : SCAN produits_fts VIRTUAL TABLE INDEX 0:M1 — ultra-rapide

-- ✅ Solution 2 : trigramme (génération applicative ou extension)
-- Stocker un index de trigrammes parallèle à la table
```

### 3. LIKE de préfixe : piège vicieux de la collation

Contrairement à ce que la documentation laisse croire, **`LIKE 'prefixe%'` n'utilise PAS automatiquement un index simple**. Raison : `LIKE` est **case-insensitive par défaut**, alors qu'un index sur TEXT utilise la collation `BINARY` (case-sensitive).

```sql
-- ❌ ANTI-PATTERN apparent : LIKE de préfixe avec index BINARY
CREATE INDEX idx_nom ON produits(nom);  
EXPLAIN QUERY PLAN SELECT * FROM produits WHERE nom LIKE 'produit_1%';  
-- Plan : SCAN produits   ← ⚠️ inattendu !
-- Temps mesuré : ≈ 95 ms sur 1 M lignes
```

**Trois corrections possibles** (du meilleur au moins bon) :

```sql
-- ✅ CORRECTION 1 : utiliser GLOB (toujours case-sensitive, utilise l'index BINARY)
SELECT * FROM produits WHERE nom GLOB 'produit_1*';
-- Plan : SEARCH produits USING INDEX idx_nom (nom>? AND nom<?)
-- Temps : ≈ 6 ms (15× plus rapide)
-- Sémantique : GLOB utilise * et ?, pas % et _

-- ✅ CORRECTION 2 : index COLLATE NOCASE (match la sémantique LIKE)
CREATE INDEX idx_nom_nocase ON produits(nom COLLATE NOCASE);  
SELECT * FROM produits WHERE nom LIKE 'produit_1%';  
-- Plan : SEARCH produits USING INDEX idx_nom_nocase
-- Temps : ≈ 40 ms

-- ⚠️ CORRECTION 3 : PRAGMA case_sensitive_like = ON (global à la connexion)
-- Casse la compatibilité ailleurs dans l'application, à éviter sauf cas isolé.
PRAGMA case_sensitive_like = ON;  
SELECT * FROM produits WHERE nom LIKE 'produit_1%';  
-- Plan : SEARCH produits USING INDEX idx_nom (nom>? AND nom<?)
-- L'index BINARY est maintenant utilisé car LIKE devient case-sensitive comme lui.
```

> 💡 **Règle simple** : pour les recherches de préfixe sur des colonnes indexées, préférer **`GLOB`** à `LIKE`. Plus rapide, plus prévisible, pas de surprise de collation.

### 4. `OR` sur colonnes avec un seul index → SCAN forcé

```sql
-- ❌ ANTI-PATTERN : `commentaire` non indexé, donc tout l'OR bascule en SCAN
CREATE INDEX idx_age ON employes(age);
-- pas d'index sur 'commentaire'
SELECT * FROM employes WHERE age < 25 OR commentaire LIKE '%urgent%';
-- Plan : SCAN employes
```

**Correction** : indexer **chaque** branche du `OR`, ou se passer du `OR` (FTS pour la recherche texte) :

```sql
-- ✅ Indexer les deux branches → MULTI-INDEX OR automatique (si égalité ou inégalité simple)
CREATE INDEX idx_age          ON employes(age);  
CREATE INDEX idx_commentaire  ON employes(commentaire);  -- utile pour égalité, PAS pour LIKE infixe  
-- Pour `commentaire LIKE '%urgent%'`, FTS5 reste la SEULE solution rapide
-- (voir module 6.6) — créer une table virtuelle FTS5 en miroir de la colonne.
```

### 5. Sous-requête corrélée non indexée → boucle quadratique

```sql
-- ❌ ANTI-PATTERN : N requêtes internes pour N lignes externes
SELECT nom FROM employes e1  
WHERE salaire > (  
    SELECT AVG(salaire) FROM employes e2 WHERE e2.departement = e1.departement
);
-- Plan : SCAN e1 + CORRELATED SCALAR SUBQUERY (SCAN e2) → O(n²)
```

**Correction** : pré-calculer avec une CTE ou window function :

```sql
-- ✅ Window function : la moyenne par département calculée UNE FOIS
SELECT nom FROM (
    SELECT nom, salaire, AVG(salaire) OVER (PARTITION BY departement) AS avg_dept
    FROM employes
) WHERE salaire > avg_dept;
-- Plan : un seul SCAN, complexité O(n)
```

### 6. `SELECT *` quand quelques colonnes suffisent

```sql
-- ❌ ANTI-PATTERN : force la lecture de TOUTES les colonnes (BLOB inclus)
SELECT * FROM produits WHERE categorie = 'electronique';
-- → SQLite doit lire les pages de la TABLE, même si idx_categorie existe.

-- ✅ Demander seulement les colonnes nécessaires + index couvrant
SELECT id, nom, prix FROM produits WHERE categorie = 'electronique';
-- Si idx_cat_prix(categorie, prix) existe et que id=rowid :
-- Plan : SEARCH produits USING COVERING INDEX idx_cat_prix
-- → aucune lecture de la table principale ! Très rapide.
```

### 7. `COUNT(colonne)` vs `COUNT(*)`

```sql
-- ❌ Pas équivalent : COUNT(col) IGNORE les NULL → SQLite doit lire chaque valeur
SELECT COUNT(commentaire) FROM employes;

-- ✅ COUNT(*) compte les lignes sans toucher aux colonnes → utilise l'index le plus petit
SELECT COUNT(*) FROM employes;
-- Plan : SEARCH employes USING COVERING INDEX (le plus petit dispo)
```

> ⚠️ **Sémantique différente** : `COUNT(col)` retourne le nombre de lignes où `col IS NOT NULL`. Ne pas remplacer aveuglément si la nuance NULL compte.

### 8. `DISTINCT` abusif (souvent inutile sur des PK)

```sql
-- ❌ ANTI-PATTERN : DISTINCT sur des colonnes qui contiennent déjà la PK
SELECT DISTINCT id, nom FROM employes;
-- → SQLite ajoute un USE TEMP B-TREE pour DISTINCT inutilement
--   (id étant la PK, les lignes sont déjà uniques)

-- ✅ Sans DISTINCT
SELECT id, nom FROM employes;
```

**Autre cas fréquent** : `SELECT DISTINCT … FROM a JOIN b ON …` quand on aurait dû écrire `SELECT … FROM a WHERE EXISTS (SELECT 1 FROM b WHERE …)`. Le `EXISTS` évite le `DISTINCT` ET les doublons en amont.

### 9. `NOT IN` avec sous-requête pouvant contenir NULL

```sql
-- ❌ PIÈGE SÉMANTIQUE : si la sous-requête contient UN seul NULL → 0 ligne en retour
SELECT * FROM clients WHERE id NOT IN (SELECT client_id FROM commandes);
-- Si une commande a client_id = NULL → résultat VIDE (logique tri-valuée SQL)
```

**Correction** :
```sql
-- ✅ NOT EXISTS gère correctement les NULL
SELECT * FROM clients c WHERE NOT EXISTS (
    SELECT 1 FROM commandes co WHERE co.client_id = c.id
);

-- ✅ Ou filtrer les NULL explicitement
SELECT * FROM clients WHERE id NOT IN (
    SELECT client_id FROM commandes WHERE client_id IS NOT NULL
);
```

### 10. Index sur colonnes à faible cardinalité

```sql
-- ❌ ANTI-PATTERN : index sur une colonne booléenne ou à 2-3 valeurs distinctes
CREATE INDEX idx_actif ON clients(actif);  -- actif = 0 ou 1 uniquement
-- Si ~50 % des lignes sont actives, l'index est presque inutile :
-- SQLite préférera souvent un SCAN.
```

**Cas où ça redevient utile** :
- En **index partiel** : `CREATE INDEX idx_clients_actifs ON clients(nom) WHERE actif = 1;` — l'index ne contient que les actifs (souvent <50 %) et accélère les requêtes filtrant sur ce critère.
- En **2ᵉ colonne d'un index composite** : `(date_inscription, actif)` permet de combiner inégalité date + égalité booléenne.

### Récapitulatif des anti-patterns

| # | Anti-pattern | Conséquence | Correction principale |
|--:|---|---|---|
| 1 | Fonction sur colonne (`UPPER`, `CAST`, arithmétique...) | SCAN | Index sur expression |
| 2 | `LIKE '%xxx%'` | SCAN | FTS5 |
| 3 | `LIKE 'xxx%'` sur index BINARY | SCAN | `GLOB` ou index `COLLATE NOCASE` |
| 4 | `OR` avec une branche non indexée | SCAN | Indexer toutes les branches |
| 5 | Sous-requête corrélée non indexée | O(n²) | CTE ou window function |
| 6 | `SELECT *` | Lecture inutile de colonnes | `SELECT col1, col2` + index couvrant |
| 7 | `COUNT(colonne_pleine)` | Lecture de la colonne | `COUNT(*)` (sauf besoin de NULL) |
| 8 | `DISTINCT` quand inutile | `TEMP B-TREE` inutile | Vérifier l'unicité réelle |
| 9 | `NOT IN` avec NULL possible | Résultat vide silencieux | `NOT EXISTS` |
| 10 | Index sur colonne à 2 valeurs | Index ignoré | Index partiel ou composite |

## Techniques avancées d'optimisation

### 1. Requêtes avec `EXISTS` vs `IN`

**Idée reçue répandue** : `EXISTS` serait toujours plus rapide que `IN`. **C'est plus nuancé en SQLite.**

```sql
-- Forme IN : la sous-requête est exécutée UNE FOIS et matérialisée en liste
SELECT nom FROM clients  
WHERE id IN (SELECT client_id FROM commandes WHERE total > 1000);  
-- Plan : SEARCH clients USING INTEGER PRIMARY KEY (rowid=?)
--        + LIST SUBQUERY (SCAN commandes)
```

```sql
-- Forme EXISTS : la sous-requête est CORRÉLÉE, ré-évaluée par ligne
SELECT nom FROM clients c  
WHERE EXISTS (  
    SELECT 1 FROM commandes co
    WHERE co.client_id = c.id AND co.total > 1000
);
-- Plan : SCAN c + CORRELATED SCALAR SUBQUERY (SEARCH co USING INDEX idx_client_id)
```

> 💡 **En pratique** :  
> - **`IN` est souvent meilleur** quand la sous-requête retourne peu de lignes (la liste tient en mémoire) et que la table externe est grande  
> - **`EXISTS` est souvent meilleur** quand on peut arrêter dès qu'une correspondance est trouvée (clause `LIMIT`, ou table externe petite)  
> - SQLite peut transformer l'un en l'autre selon ses statistiques — **toujours mesurer avec `EXPLAIN QUERY PLAN`** plutôt que de présumer

> ⚠️ **Vrai piège à retenir** : `NOT IN` avec une sous-requête pouvant contenir `NULL` retourne **0 ligne** (sémantique tri-valuée). Préférer `NOT EXISTS` qui gère correctement les NULL.

### 2. `OR` et l'optimisation `MULTI-INDEX OR`

Quand les deux branches du `OR` portent sur des **colonnes indexées**, SQLite utilise automatiquement l'optimisation `MULTI-INDEX OR` — pas besoin de réécrire en `UNION` :

```sql
-- ✅ Si idx_departement ET idx_salaire existent, SQLite utilise les deux
SELECT * FROM employes  
WHERE departement = 'IT' OR salaire > 80000;  
-- Plan : MULTI-INDEX OR
--        ├── SEARCH employes USING INDEX idx_departement (departement=?)
--        └── SEARCH employes USING INDEX idx_salaire (salaire>?)
```

```sql
-- ⚠️ Si UNE des branches n'est pas indexée, SQLite retombe sur un SCAN.
--    Indexer chaque branche AVANT de penser à réécrire en UNION ALL.
CREATE INDEX idx_departement ON employes(departement);  
CREATE INDEX idx_salaire ON employes(salaire);  
```

> 💡 Si après avoir indexé les deux branches le `OR` reste lent (cas rare), `UNION ALL` peut aider — **pas** `UNION` qui ajoute un `TEMP B-TREE` pour dédupliquer. Toujours mesurer avec `EXPLAIN QUERY PLAN`.

### 3. Dénormalisation sélective

```sql
-- Si cette requête est très fréquente :
SELECT c.nom, COUNT(co.id) as nb_commandes  
FROM clients c  
LEFT JOIN commandes co ON c.id = co.client_id  
GROUP BY c.id, c.nom;  

-- ✅ Ajouter une colonne dénormalisée
ALTER TABLE clients ADD COLUMN nb_commandes INTEGER DEFAULT 0;

-- ⚠️ Maintenir avec TROIS triggers (INSERT, DELETE, UPDATE du client_id),
--    sinon le compteur se désynchronise dès la première suppression/réaffectation.

CREATE TRIGGER nb_cmd_apres_insert  
AFTER INSERT ON commandes  
BEGIN  
    UPDATE clients SET nb_commandes = nb_commandes + 1
    WHERE id = NEW.client_id;
END;

CREATE TRIGGER nb_cmd_apres_delete  
AFTER DELETE ON commandes  
BEGIN  
    UPDATE clients SET nb_commandes = nb_commandes - 1
    WHERE id = OLD.client_id;
END;

CREATE TRIGGER nb_cmd_apres_update  
AFTER UPDATE OF client_id ON commandes  
WHEN OLD.client_id <> NEW.client_id  
BEGIN  
    UPDATE clients SET nb_commandes = nb_commandes - 1 WHERE id = OLD.client_id;
    UPDATE clients SET nb_commandes = nb_commandes + 1 WHERE id = NEW.client_id;
END;
```

> 💡 **Compromis dénormalisation** : on échange des lectures rapides contre des écritures plus coûteuses (chaque `INSERT`/`UPDATE`/`DELETE` sur `commandes` déclenche un `UPDATE` sur `clients`). À réserver aux compteurs lus très fréquemment et modifiés peu souvent.

## Cas pratiques d'optimisation

### Cas 1 : Recherche avec pagination

**Problème initial :**
```sql
-- ❌ Pagination inefficace
SELECT * FROM produits  
WHERE categorie = 'electronique'  
ORDER BY prix DESC  
LIMIT 20 OFFSET 1000;  
-- OFFSET élevé = très lent !
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
-- ⚠️ On stocke `nb_commandes` et `chiffre_affaire` (additifs),
--    pas `panier_moyen` qui ne l'est PAS — on le recalculera à la volée
--    sur la période via SUM/SUM (moyenne pondérée correcte).
CREATE TABLE stats_quotidiennes (
    date_stat       DATE PRIMARY KEY,
    nb_commandes    INTEGER,
    chiffre_affaire REAL
);

-- Calcul incrémental quotidien (à exécuter par un cron / scheduler)
INSERT OR REPLACE INTO stats_quotidiennes  
SELECT  
    date_commande,
    COUNT(*),
    SUM(total)
FROM commandes  
WHERE date_commande = date('now', '-1 day');  

-- Requête ultra-rapide sur la période (30 lignes au lieu de potentiellement millions)
SELECT
    SUM(nb_commandes)                              AS nb_commandes,
    SUM(chiffre_affaire)                           AS chiffre_affaire,
    SUM(chiffre_affaire) * 1.0 / SUM(nb_commandes) AS panier_moyen  -- moyenne pondérée
FROM stats_quotidiennes  
WHERE date_stat >= date('now', '-30 days');  
```

> ⚠️ **Piège mathématique courant** : `AVG(panier_moyen_quotidien)` n'est **pas** le panier moyen global. Exemple : un jour à 10 commandes (panier moyen 10 €) et un jour à 100 commandes (panier moyen 20 €) → `AVG = 15 €`, mais le **vrai** panier moyen pondéré est `(100 + 2000) / (10 + 100) = 19.09 €`. Pour les agrégats dérivés (moyennes, taux, ratios), toujours stocker les **briques additives** (sommes, comptages) et recalculer le dérivé sur la période.

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

⏭️ [5.5 Configuration et paramètres SQLite (PRAGMA)](/05-optimisation-performances/05-configuration-parametres-sqlite-pragma.md)
