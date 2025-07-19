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
- Permet de modifier les données avant leur insertion/modification
- Peut empêcher l'opération de se réaliser

### 2. AFTER (Après)
S'exécute **après** que l'opération ait été réalisée avec succès.
- Utilisé pour des actions de suivi (logs, notifications)
- Ne peut pas modifier les données de l'opération en cours

### 3. INSTEAD OF (À la place de)
S'exécute **à la place** de l'opération originale.
- Principalement utilisé avec les vues (views)
- Permet de rendre les vues modifiables

## Événements déclencheurs

Les triggers peuvent réagir à trois types d'opérations :
- **INSERT** : ajout de nouvelles données
- **UPDATE** : modification de données existantes
- **DELETE** : suppression de données

## Syntaxe de base

```sql
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

Maintenant, créons un trigger qui met à jour automatiquement `date_modification` :

```sql
CREATE TRIGGER update_date_modification
    BEFORE UPDATE ON produits
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
BEGIN
    INSERT INTO historique_produits (produit_id, action, ancien_prix, nouveau_prix)
    VALUES (NEW.id, 'MODIFICATION_PRIX', OLD.prix, NEW.prix);
END;
```

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
CREATE TRIGGER audit_suppressions
    BEFORE DELETE ON produits
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, timestamp)
    VALUES ('produits', 'DELETE', 'ID:' || OLD.id || ' Nom:' || OLD.nom, CURRENT_TIMESTAMP);
END;
```

### 2. Calculs automatiques
```sql
-- Table commandes avec total automatique
CREATE TRIGGER calculer_total
    BEFORE INSERT ON lignes_commande
BEGIN
    UPDATE commandes
    SET total = total + (NEW.quantite * NEW.prix_unitaire)
    WHERE id = NEW.commande_id;
END;
```

### 3. Validation métier
```sql
CREATE TRIGGER verifier_stock
    BEFORE UPDATE OF quantite ON produits
    WHEN NEW.quantite < 0
BEGIN
    SELECT RAISE(ABORT, 'Stock insuffisant pour le produit: ' || NEW.nom);
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

### Désactiver temporairement les triggers
```sql
PRAGMA recursive_triggers = OFF;  -- Désactive les triggers récursifs
```

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

## Limitations à connaître

- Les triggers SQLite ne peuvent pas :
  - Retourner de valeurs
  - Utiliser des transactions explicites (BEGIN/COMMIT)
  - Appeler des fonctions définies par l'utilisateur depuis certains contextes

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

⏭️
