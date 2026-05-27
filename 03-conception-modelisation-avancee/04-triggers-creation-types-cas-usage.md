🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.4 - Triggers : création, types et cas d'usage

## Qu'est-ce qu'un trigger ?

Un **trigger** (déclencheur en français) est un bloc de code SQL qui s'exécute automatiquement en réponse à certains événements dans une base de données. Imaginez-le comme un "gardien" qui surveille votre base de données et réagit quand quelque chose se passe.

### Analogie simple
Pensez à un trigger comme à une alarme de maison :
- L'alarme se déclenche automatiquement quand quelqu'un ouvre la porte (événement)
- Elle exécute une action prédéfinie : sonner et envoyer une alerte (code du trigger)
- Vous n'avez pas besoin d'appuyer sur un bouton, tout se fait automatiquement

## Pourquoi utiliser les triggers ?

Les triggers sont utiles pour :
- **Automatiser des tâches répétitives** (mise à jour automatique de totaux, dates de modification)
- **Maintenir la cohérence des données** (vérifications automatiques)
- **Créer un historique des modifications** (audit trail)
- **Appliquer des règles métier complexes**

## Types de triggers dans SQLite

SQLite propose trois types de triggers selon le moment où ils se déclenchent :

### 1. BEFORE (Avant)
S'exécute **avant** que l'opération ait lieu.
- Idéal pour valider/rejeter l'opération via `RAISE(ABORT, ...)`
- En SQLite, on **ne peut pas modifier `NEW.colonne` directement** (≠ PostgreSQL) ; pour ajuster une ligne, on utilise plutôt un trigger `AFTER` avec un `UPDATE` secondaire et une garde `WHEN`

### 2. AFTER (Après)
S'exécute **après** que l'opération ait été réalisée avec succès.
- Utilisé pour des actions de suivi (logs, notifications, calculs dérivés)
- Peut effectuer un `UPDATE` secondaire sur la ligne modifiée (pattern « date_modification »)

### 3. INSTEAD OF (À la place de)
S'exécute **à la place** de l'opération originale.
- Réservé aux **vues** (pas aux tables) — voir aussi le chapitre 3.5
- En SQLite, c'est le **seul moyen** de rendre une vue acceptant `INSERT`/`UPDATE`/`DELETE`

## Événements déclencheurs

Les triggers peuvent réagir à trois types d'opérations :
- **INSERT** : ajout de nouvelles données
- **UPDATE** : modification de données existantes
- **DELETE** : suppression de données

## Syntaxe de base

Notation : `[ … ]` désigne une partie **optionnelle** et `a|b` un choix entre les variantes — c'est de la pseudo-syntaxe descriptive, pas du SQL exécutable.

```text
CREATE TRIGGER nom_du_trigger
    [BEFORE|AFTER|INSTEAD OF] [INSERT|UPDATE|DELETE]
    ON nom_de_la_table
    [WHEN condition]
BEGIN
    -- Code SQL à exécuter
END;
```

## Exemples pratiques pour débutants

### Exemple 1 : Mise à jour automatique de la date de modification

Créons d'abord une table simple :

```sql
CREATE TABLE produits (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    prix REAL NOT NULL,
    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
    date_modification DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Maintenant, créons un trigger qui met à jour automatiquement `date_modification`.

> ⚠️ **Pattern correct pour `date_modification`** : contrairement à PostgreSQL/MySQL, SQLite **ne permet pas d'écrire `NEW.colonne := ...`** dans un trigger. On peut faire un `UPDATE` secondaire dans un `BEFORE UPDATE`, mais le résultat dépend alors des colonnes listées dans l'`UPDATE` original (toute colonne explicitement mise à jour par l'utilisateur écrase ce que le trigger aurait mis). Pour éviter ce piège et garantir un comportement homogène, on utilise un trigger **`AFTER UPDATE`** avec une **garde anti-récursion** dans la clause `WHEN`.

```sql
CREATE TRIGGER update_date_modification
    AFTER UPDATE ON produits
    WHEN NEW.date_modification = OLD.date_modification   -- évite la récursion infinie
BEGIN
    UPDATE produits
    SET date_modification = CURRENT_TIMESTAMP
    WHERE id = NEW.id;
END;
```

**Test du trigger :**
```sql
-- Insérer un produit
INSERT INTO produits (nom, prix) VALUES ('Ordinateur', 999.99);

-- Modifier le prix (le trigger se déclenchera automatiquement)
UPDATE produits SET prix = 899.99 WHERE nom = 'Ordinateur';

-- Vérifier que la date_modification a été mise à jour
SELECT * FROM produits;
```

### Exemple 2 : Validation des données avec BEFORE

Créons un trigger qui empêche l'insertion de prix négatifs :

```sql
CREATE TRIGGER verifier_prix_positif
    BEFORE INSERT ON produits
    WHEN NEW.prix < 0
BEGIN
    SELECT RAISE(ABORT, 'Le prix ne peut pas être négatif');
END;
```

**Test du trigger :**
```sql
-- Ceci va échouer grâce au trigger
INSERT INTO produits (nom, prix) VALUES ('Produit invalide', -50);
-- Erreur : Le prix ne peut pas être négatif

-- Ceci va fonctionner
INSERT INTO produits (nom, prix) VALUES ('Produit valide', 50);
```

### Exemple 3 : Création d'un historique avec AFTER

Créons d'abord une table d'historique :

```sql
CREATE TABLE historique_produits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    produit_id INTEGER,
    action TEXT,
    ancien_prix REAL,
    nouveau_prix REAL,
    date_action DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Trigger pour enregistrer les modifications de prix :

```sql
CREATE TRIGGER historique_prix
    AFTER UPDATE OF prix ON produits
    WHEN OLD.prix != NEW.prix          -- ignore les UPDATE qui ne changent rien
BEGIN
    INSERT INTO historique_produits (produit_id, action, ancien_prix, nouveau_prix)
    VALUES (NEW.id, 'MODIFICATION_PRIX', OLD.prix, NEW.prix);
END;
```

> 💡 **Subtilité `UPDATE OF`** : `AFTER UPDATE OF prix` se déclenche **chaque fois que la colonne `prix` figure dans la clause `SET` de l'`UPDATE`**, même si la nouvelle valeur est identique à l'ancienne (`UPDATE produits SET prix = prix` déclenche le trigger). La clause `WHEN OLD.prix != NEW.prix` évite de logger des « modifications » qui n'en sont pas. À l'inverse, l'historique sans cette garde reflète l'**intention** de l'utilisateur (les tentatives d'écriture), ce qui peut aussi avoir du sens dans certains audits.

**Test du trigger :**
```sql
-- Modifier un prix
UPDATE produits SET prix = 799.99 WHERE id = 1;

-- Vérifier l'historique
SELECT * FROM historique_produits;
```

## Mots-clés spéciaux : OLD et NEW

Dans les triggers, vous pouvez utiliser :
- **NEW** : fait référence aux nouvelles valeurs (INSERT et UPDATE)
- **OLD** : fait référence aux anciennes valeurs (UPDATE et DELETE)

### Exemple avec OLD et NEW :

```sql
-- Table minimale pour rendre l'exemple exécutable
CREATE TABLE logs (id INTEGER PRIMARY KEY AUTOINCREMENT, message TEXT);

CREATE TRIGGER log_changements
    AFTER UPDATE ON produits
BEGIN
    INSERT INTO logs (message) VALUES
    ('Produit ' || OLD.nom || ' : prix changé de ' || OLD.prix || ' à ' || NEW.prix);
END;
```

## La clause WHEN

La clause `WHEN` permet d'ajouter des conditions pour déclencher le trigger :

```sql
-- Table minimale pour rendre l'exemple exécutable
CREATE TABLE alertes (id INTEGER PRIMARY KEY AUTOINCREMENT, message TEXT);

CREATE TRIGGER alerte_prix_eleve
    AFTER INSERT ON produits
    WHEN NEW.prix > 1000
BEGIN
    INSERT INTO alertes (message) VALUES
    ('ATTENTION: Nouveau produit à prix élevé: ' || NEW.nom || ' (' || NEW.prix || '€)');
END;
```

## Cas d'usage courants

### 1. Audit et traçabilité
```sql
-- Table d'audit minimale pour rendre l'exemple exécutable
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name TEXT,
    operation TEXT,
    old_data TEXT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TRIGGER audit_suppressions
    BEFORE DELETE ON produits
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, timestamp)
    VALUES ('produits', 'DELETE', 'ID:' || OLD.id || ' Nom:' || OLD.nom, CURRENT_TIMESTAMP);
END;
```

### 2. Calculs automatiques

```sql
-- Tables minimales pour rendre l'exemple exécutable
CREATE TABLE commandes (
    id INTEGER PRIMARY KEY,
    total REAL DEFAULT 0
);
CREATE TABLE lignes_commande (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    commande_id INTEGER NOT NULL,
    quantite INTEGER NOT NULL,
    prix_unitaire REAL NOT NULL,
    FOREIGN KEY (commande_id) REFERENCES commandes(id)
);

-- Le trigger maintient le total de la commande à chaque nouvelle ligne
CREATE TRIGGER calculer_total
    BEFORE INSERT ON lignes_commande
BEGIN
    UPDATE commandes
    SET total = total + (NEW.quantite * NEW.prix_unitaire)
    WHERE id = NEW.commande_id;
END;
```

### 3. Validation métier

> ⚠️ **Limitation `RAISE`** : SQLite exige que le second argument de `RAISE(ABORT, ...)` soit une **chaîne littérale constante**. La concaténation `||` avec `NEW.col` produit `Parse error: near "||"`. Pour un message dynamique, on insère d'abord dans une table d'audit dédiée puis on lève une erreur générique — mais voir aussi la section « Piège fréquent : INSERT d'audit + RAISE(ABORT) » ci-dessous, qui explique pourquoi cette approche échoue dans la même transaction.

```sql
-- On suppose ici une table `produits_stock` avec une colonne `stock`
CREATE TABLE produits_stock (
    id INTEGER PRIMARY KEY,
    nom TEXT NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0
);

CREATE TRIGGER verifier_stock
    BEFORE UPDATE OF stock ON produits_stock
    WHEN NEW.stock < 0
BEGIN
    SELECT RAISE(ABORT, 'Stock négatif interdit');
END;
```

## Gestion des triggers

### Lister tous les triggers
```sql
SELECT name FROM sqlite_master WHERE type = 'trigger';
```

### Voir la définition d'un trigger
```sql
SELECT sql FROM sqlite_master WHERE type = 'trigger' AND name = 'nom_du_trigger';
```

### Supprimer un trigger
```sql
DROP TRIGGER IF EXISTS nom_du_trigger;
```

### Contrôler le déclenchement en cascade

Par défaut, SQLite **désactive** les triggers récursifs (`recursive_triggers = 0`) depuis la version 3.6.18 : si un trigger modifie une table, cela ne déclenchera pas d'autres triggers sur cette même table en chaîne.

```sql
-- Vérifier l'état actuel (0 = désactivé, 1 = activé)
PRAGMA recursive_triggers;

-- L'activer si besoin (rarement recommandé : risque de boucles infinies)
PRAGMA recursive_triggers = ON;
```

> ⚠️ Pour désactiver **complètement** un trigger sans le supprimer, SQLite ne propose pas d'`ALTER TRIGGER DISABLE`. La seule option est `DROP TRIGGER` puis recréation, ou utiliser une table de configuration et une clause `WHEN` qui consulte un drapeau.

## Bonnes pratiques

1. **Nommage cohérent** : Utilisez des noms explicites (ex: `trg_before_insert_produits`)

2. **Évitez la complexité excessive** : Les triggers doivent rester simples et rapides

3. **Attention aux boucles infinies** : Un trigger qui modifie la même table peut se déclencher lui-même

4. **Testez soigneusement** : Les triggers s'exécutent automatiquement, les erreurs peuvent être difficiles à déboguer

5. **Documentez vos triggers** : Ajoutez des commentaires pour expliquer leur utilité

## Exercices pratiques

### Exercice 1 : Trigger de validation
Créez un trigger qui empêche l'insertion d'un produit avec un nom vide ou null.

### Exercice 2 : Compteur automatique
Créez une table `statistiques` avec un compteur du nombre total de produits, et des triggers pour maintenir ce compteur à jour.

### Exercice 3 : Historique complet
Créez un système d'historique qui enregistre toutes les opérations (INSERT, UPDATE, DELETE) sur la table produits.

## ⚠️ Piège fréquent : `INSERT` d'audit + `RAISE(ABORT)`

Une erreur très courante : combiner un `INSERT` dans une table d'audit avec un `RAISE(ABORT)` dans le **même trigger** pour journaliser puis rejeter l'opération. **Cela ne fonctionne pas** : `RAISE(ABORT)` annule **toute la transaction** en cours — y compris l'`INSERT` dans la table d'audit. Résultat : on rejette l'opération **et** on perd la trace du rejet.

```sql
-- Tables minimales pour rendre l'exemple exécutable
CREATE TABLE users (id INTEGER PRIMARY KEY, email TEXT);  
CREATE TABLE contrainte_violations (id INTEGER PRIMARY KEY AUTOINCREMENT, msg TEXT);  

-- ❌ Trigger BUGGÉ — l'INSERT d'audit est annulé par le RAISE(ABORT)
CREATE TRIGGER log_violations_users  
AFTER INSERT ON users  
WHEN NEW.email NOT LIKE '%@%.%'  
BEGIN  
    INSERT INTO contrainte_violations (msg) VALUES ('Email invalide: ' || NEW.email);
    SELECT RAISE(ABORT, 'Email format invalide');   -- annule tout, y compris l'INSERT précédent
END;

-- Démonstration : on tente d'insérer un email invalide
-- INSERT INTO users (email) VALUES ('pas-un-email');
-- → Échec attendu, ET la ligne dans contrainte_violations est rollback aussi.
-- SELECT * FROM contrainte_violations;  -- (vide)
```

✅ **Approche correcte** : utiliser une contrainte `CHECK` (qui lève l'erreur sans trigger), et faire le logging côté **application** avec une connexion SQLite séparée — ou ne logger que les opérations **réussies** dans un trigger `AFTER` sans `RAISE`.

```sql
-- ✅ Contrainte CHECK pour rejeter, audit en AFTER pour les succès
CREATE TABLE users_v2 (
    id    INTEGER PRIMARY KEY,
    email TEXT NOT NULL CHECK (email LIKE '%@%.%')
);

CREATE TRIGGER audit_users_insert_ok  
AFTER INSERT ON users_v2  
BEGIN  
    INSERT INTO contrainte_violations (msg) VALUES ('Insertion réussie: ' || NEW.email);
END;
```

## INSTEAD OF — Rendre une vue modifiable

Les triggers `INSTEAD OF` sont **réservés aux vues** (pas aux tables) et permettent de définir le comportement d'un `INSERT`, `UPDATE` ou `DELETE` sur une vue, qui n'est normalement pas modifiable.

### Exemple : soft delete via une vue

```sql
CREATE TABLE articles (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    supprime INTEGER DEFAULT 0 CHECK (supprime IN (0, 1)),
    date_suppression TEXT
);

-- Vue qui ne montre que les articles actifs
CREATE VIEW articles_actifs AS  
SELECT id, titre FROM articles WHERE supprime = 0;  

-- Trigger INSTEAD OF : un DELETE sur la vue → soft delete dans la table sous-jacente
CREATE TRIGGER soft_delete_article
    INSTEAD OF DELETE ON articles_actifs
BEGIN
    UPDATE articles
    SET supprime = 1, date_suppression = datetime('now')
    WHERE id = OLD.id;
END;

-- Utilisation : l'utilisateur écrit un DELETE classique mais la ligne reste en base
INSERT INTO articles (titre) VALUES ('Mon article'), ('Autre article');  
DELETE FROM articles_actifs WHERE id = 1;   -- soft delete, pas de suppression physique  
SELECT * FROM articles;                      -- la ligne 1 a supprime=1, date_suppression renseignée  
SELECT * FROM articles_actifs;               -- la ligne 1 n'apparaît plus dans la vue  
```

## Limitations à connaître

- Les triggers SQLite ne peuvent pas :
  - Retourner de valeurs
  - Utiliser des transactions explicites (BEGIN/COMMIT)
  - Appeler des fonctions définies par l'utilisateur depuis certains contextes
  - **Modifier `NEW.colonne` directement** (contrairement à PostgreSQL/MySQL) — utiliser un trigger `AFTER` avec un `UPDATE` secondaire et une garde `WHEN`
  - **Combiner un `INSERT` d'audit avec `RAISE(ABORT)`** dans le même trigger — voir piège ci-dessus

- Ils peuvent impacter les performances si mal utilisés

## Résumé

Les triggers SQLite sont un outil puissant pour automatiser des tâches et maintenir l'intégrité des données. Ils s'exécutent automatiquement en réponse aux opérations INSERT, UPDATE et DELETE, et peuvent s'exécuter BEFORE, AFTER, ou INSTEAD OF ces opérations.

**Points clés à retenir :**
- Les triggers automatisent des tâches répétitives
- Ils peuvent valider des données et maintenir la cohérence
- Utilisez OLD et NEW pour accéder aux valeurs
- La clause WHEN permet d'ajouter des conditions
- Testez toujours soigneusement vos triggers

Dans le prochain chapitre, nous explorerons les vues (views) et comment les utiliser efficacement avec SQLite.

⏭️ [3.5 Vues : création, utilisation et maintenance](/03-conception-modelisation-avancee/05-vues-creation-utilisation-maintenance.md)
