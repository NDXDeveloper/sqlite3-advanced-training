🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.3 Opérations CRUD : CREATE, READ, UPDATE, DELETE

## Introduction - Les 4 piliers de la manipulation de données

CRUD est l'acronyme des quatre opérations fondamentales que vous effectuerez quotidiennement avec SQLite :

- **C**reate → **Créer** des données (INSERT)
- **R**ead → **Lire** des données (SELECT)
- **U**pdate → **Modifier** des données (UPDATE)
- **D**elete → **Supprimer** des données (DELETE)

Ces opérations constituent 90% de votre travail avec une base de données. Maîtrisez-les, et vous pourrez construire n'importe quelle application !

> **Pour les débutants** : Pensez à CRUD comme aux actions de base sur un carnet d'adresses : ajouter un contact, le consulter, modifier ses informations, ou le supprimer.

## Préparation - Création de notre base d'exemple

Avant de pratiquer les opérations CRUD, créons une base de données simple mais réaliste :

```sql
-- Créer notre base d'exercices
sqlite3 boutique_crud.db

-- Configuration optimale (rappel du module 2.2)
PRAGMA journal_mode = WAL;       -- persistant  
PRAGMA synchronous = NORMAL;     -- par connexion  
PRAGMA busy_timeout = 5000;      -- par connexion  
PRAGMA foreign_keys = ON;        -- par connexion — indispensable pour activer les FK  

-- Création des tables d'exemple
CREATE TABLE clients (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT NOT NULL,
    email TEXT UNIQUE,
    telephone TEXT,
    date_inscription TEXT DEFAULT (datetime('now')),
    actif INTEGER DEFAULT 1
);

CREATE TABLE produits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    description TEXT,
    prix REAL NOT NULL CHECK (prix > 0),
    stock INTEGER DEFAULT 0 CHECK (stock >= 0),
    categorie TEXT,
    date_creation TEXT DEFAULT (datetime('now'))
);

CREATE TABLE commandes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    client_id INTEGER,
    date_commande TEXT DEFAULT (datetime('now')),
    statut TEXT DEFAULT 'en_attente' CHECK (statut IN ('en_attente', 'confirmee', 'expediee', 'livree', 'annulee')),
    total REAL DEFAULT 0,
    FOREIGN KEY (client_id) REFERENCES clients(id)
);

-- Vérifier la structure
.tables
.schema
```

## CREATE - Créer des données (INSERT)

### 🆕 Insertion simple

L'instruction INSERT permet d'ajouter de nouvelles données :

```sql
-- Syntaxe de base
INSERT INTO table (colonne1, colonne2, ...) VALUES (valeur1, valeur2, ...);

-- Exemple concret
INSERT INTO clients (nom, prenom, email, telephone)  
VALUES ('Dupont', 'Jean', 'jean.dupont@email.com', '0123456789');  

-- Vérifier l'insertion
SELECT * FROM clients;
```

### 📝 Insertions multiples

```sql
-- Insérer plusieurs enregistrements en une fois
INSERT INTO clients (nom, prenom, email, telephone) VALUES
    ('Martin', 'Sophie', 'sophie.martin@email.com', '0234567890'),
    ('Durand', 'Pierre', 'pierre.durand@email.com', '0345678901'),
    ('Leroy', 'Marie', 'marie.leroy@email.com', '0456789012'),
    ('Bernard', 'Paul', 'paul.bernard@email.com', '0567890123');

-- Vérifier toutes les insertions
SELECT COUNT(*) as nombre_clients FROM clients;  
SELECT * FROM clients ORDER BY id;  
```

### 🛍️ Insertion avec valeurs par défaut

```sql
-- Insertion minimale (les valeurs par défaut sont utilisées)
INSERT INTO clients (nom, prenom) VALUES ('Moreau', 'Julie');

-- Insertion en spécifiant certaines valeurs par défaut
INSERT INTO clients (nom, prenom, actif) VALUES ('Petit', 'Luc', 0);

-- Voir le résultat avec les dates automatiques
SELECT id, nom, prenom, date_inscription, actif FROM clients  
WHERE nom IN ('Moreau', 'Petit');  
```

### 🏷️ Insertion de produits avec différents types

```sql
INSERT INTO produits (nom, description, prix, stock, categorie) VALUES
    ('Laptop Gaming', 'Ordinateur portable haute performance', 1299.99, 15, 'Informatique'),
    ('Souris Ergonomique', 'Souris sans fil ergonomique', 45.50, 50, 'Informatique'),
    ('Clavier Mécanique', 'Clavier rétroéclairé switches bleus', 89.90, 25, 'Informatique'),
    ('Casque Audio', 'Casque circum-auriculaire', 149.00, 8, 'Audio'),
    ('Webcam HD', 'Caméra 1080p pour visioconférence', 79.99, 12, 'Informatique');

-- Vérifier avec un aperçu formaté
SELECT
    nom,
    prix || '€' as prix_formatte,
    stock || ' unités' as stock_formatte,
    categorie
FROM produits;
```

### 🔄 Insertion avec récupération d'ID

```sql
-- Insérer et récupérer l'ID généré
INSERT INTO clients (nom, prenom, email)  
VALUES ('Nouveau', 'Client', 'nouveau@email.com');  

-- Voir le dernier ID inséré
SELECT last_insert_rowid() AS dernier_id;
```

> ⚠️ **Attention à `last_insert_rowid()`** : cette fonction renvoie l'ID du **dernier INSERT exécuté sur cette connexion**. Si vous faites un autre INSERT entre-temps, la valeur change. **Toujours capturer la valeur immédiatement** :  
> ```sql
> -- ❌ Risque : last_insert_rowid() a peut-être changé entre les deux appels
> INSERT INTO clients (nom, prenom) VALUES ('Test', 'Alice');
> INSERT INTO commandes (client_id) VALUES (last_insert_rowid());  -- OK ici
> INSERT INTO logs (msg) VALUES ('Nouvelle commande');             -- Modifie le rowid !
> SELECT last_insert_rowid();   -- Donne l'id du log, pas de la commande !
> ```

**✨ Alternative moderne : `RETURNING`** (depuis SQLite **3.35 (2021)**) :
```sql
-- Récupérer directement l'ID (et d'autres colonnes) avec l'INSERT
INSERT INTO clients (nom, prenom, email)  
VALUES ('Nouveau', 'Client', 'nouveau@email.com')  
RETURNING id, datetime('now') AS quand;  

-- Utilisable aussi avec UPDATE et DELETE
DELETE FROM clients WHERE actif = 0  
RETURNING id, nom, prenom;  -- Voir ce qu'on a supprimé  
```
`RETURNING` est plus sûr (atomique) et plus expressif que `last_insert_rowid()` — utilisez-le quand votre version SQLite est ≥ 3.35.

### ⚠️ Gestion des erreurs d'insertion

```sql
-- Tentative d'insertion avec email dupliqué (va échouer)
INSERT INTO clients (nom, prenom, email)  
VALUES ('Test', 'Doublon', 'jean.dupont@email.com');  
-- Error: UNIQUE constraint failed: clients.email

-- Insertion avec gestion du conflit (disponible depuis SQLite 3.0)
INSERT OR IGNORE INTO clients (nom, prenom, email)  
VALUES ('Test', 'Ignore', 'jean.dupont@email.com');  

-- Alternative : mise à jour en cas de conflit (REMPLACE toute la ligne)
INSERT OR REPLACE INTO clients (nom, prenom, email)  
VALUES ('Dupont', 'Jean-Nouveau', 'jean.dupont@email.com');  

-- UPSERT moderne (depuis SQLite 3.24, juin 2018) — plus puissant et précis
INSERT INTO clients (nom, prenom, email)  
VALUES ('Dupont', 'Jean-Nouveau', 'jean.dupont@email.com')  
ON CONFLICT(email) DO UPDATE SET  
    nom = excluded.nom,
    prenom = excluded.prenom;
-- excluded.<col> = la valeur qu'on a tenté d'insérer

-- Vérifier le résultat
SELECT * FROM clients WHERE email = 'jean.dupont@email.com';
```

> 💡 **Choisir entre OR REPLACE et ON CONFLICT** :  
> - `INSERT OR REPLACE` : **supprime la ligne en conflit** puis insère la nouvelle. Attention, cascade les triggers DELETE et peut perdre des données dans d'autres colonnes non spécifiées.  
> - `INSERT … ON CONFLICT … DO UPDATE` : effectue un vrai UPDATE ciblé sur la ligne existante. **Recommandé en pratique**.

## READ - Lire des données (SELECT)

### 👁️ Sélection de base

```sql
-- Sélectionner toutes les colonnes
SELECT * FROM clients;

-- Sélectionner des colonnes spécifiques
SELECT nom, prenom, email FROM clients;

-- Sélection avec alias pour plus de clarté
-- ⚠️ Utilisez des guillemets DOUBLES (ou des crochets) pour les alias contenant
--   des espaces. Les guillemets SIMPLES sont des littéraux de chaîne en SQL standard ;
--   SQLite les tolère par compatibilité, mais c'est à éviter.
SELECT
    nom    AS "Nom de famille",
    prenom AS "Prénom",
    email  AS "Adresse email"
FROM clients;
```

### 🔍 Filtrage avec WHERE

```sql
-- Filtrer par critère simple
SELECT * FROM clients WHERE actif = 1;

-- Filtres multiples avec AND
SELECT nom, prenom, email  
FROM clients  
WHERE actif = 1 AND nom LIKE 'D%';  

-- Filtres avec OR
SELECT * FROM produits  
WHERE categorie = 'Informatique' OR prix < 100;  

-- Filtres avec IN
SELECT * FROM produits  
WHERE categorie IN ('Informatique', 'Audio');  

-- Filtres avec plage de valeurs
SELECT nom, prix FROM produits  
WHERE prix BETWEEN 50 AND 150;  
```

### 📊 Tri et limitation

```sql
-- Tri croissant
SELECT nom, prix FROM produits ORDER BY prix;

-- Tri décroissant
SELECT nom, prix FROM produits ORDER BY prix DESC;

-- Tri multiple
SELECT nom, prix, categorie FROM produits  
ORDER BY categorie, prix DESC;  

-- Limitation du nombre de résultats
SELECT nom, prix FROM produits  
ORDER BY prix DESC  
LIMIT 3;  

-- Pagination avec OFFSET
SELECT nom, prix FROM produits  
ORDER BY nom  
LIMIT 3 OFFSET 3;  -- Affiche les résultats 4, 5, 6  
```

### 🔢 Fonctions d'agrégation

```sql
-- Compter les enregistrements
SELECT COUNT(*) as total_clients FROM clients;  
SELECT COUNT(*) as clients_actifs FROM clients WHERE actif = 1;  

-- Valeurs min, max, moyenne
SELECT
    MIN(prix) as prix_minimum,
    MAX(prix) as prix_maximum,
    AVG(prix) as prix_moyen,
    SUM(stock * prix) as valeur_stock_total
FROM produits;

-- Groupement par catégorie
SELECT
    categorie,
    COUNT(*) as nombre_produits,
    AVG(prix) as prix_moyen,
    SUM(stock) as stock_total
FROM produits  
GROUP BY categorie;  
```

### 🔗 Jointures simples

```sql
-- Jointure pour voir les commandes avec les noms des clients
SELECT
    c.nom,
    c.prenom,
    cmd.id as numero_commande,
    cmd.date_commande,
    cmd.statut,
    cmd.total
FROM clients c  
JOIN commandes cmd ON c.id = cmd.client_id  
ORDER BY cmd.date_commande DESC;  

-- Jointure avec filtre
SELECT
    c.nom || ' ' || c.prenom as client_complet,
    COUNT(cmd.id) as nombre_commandes
FROM clients c  
LEFT JOIN commandes cmd ON c.id = cmd.client_id  
GROUP BY c.id, c.nom, c.prenom  
HAVING COUNT(cmd.id) > 0;  
```

## UPDATE - Modifier des données (UPDATE)

### ✏️ Modification simple

```sql
-- Syntaxe de base
UPDATE table SET colonne = nouvelle_valeur WHERE condition;

-- Exemple : Mettre à jour le téléphone d'un client
UPDATE clients  
SET telephone = '0123456999'  
WHERE email = 'jean.dupont@email.com';  

-- Vérifier la modification
SELECT nom, prenom, telephone  
FROM clients  
WHERE email = 'jean.dupont@email.com';  
```

### 🔄 Modifications multiples

```sql
-- Modifier plusieurs colonnes en même temps
UPDATE clients  
SET  
    telephone = '0987654321',
    email = 'sophie.martin.nouveau@email.com'
WHERE nom = 'Martin' AND prenom = 'Sophie';

-- Modifier plusieurs enregistrements
UPDATE produits  
SET prix = prix * 0.9  -- Remise de 10%  
WHERE categorie = 'Informatique';  

-- Vérifier les modifications
SELECT nom, prix, categorie FROM produits WHERE categorie = 'Informatique';
```

### 📈 Modifications calculées

```sql
-- Augmenter le stock de tous les produits
UPDATE produits SET stock = stock + 10;

-- Mise à jour conditionnelle avec calcul
UPDATE produits  
SET prix = CASE  
    WHEN stock < 10 THEN prix * 1.1  -- +10% si stock faible
    WHEN stock > 50 THEN prix * 0.95 -- -5% si stock élevé
    ELSE prix  -- Prix inchangé
END;

-- Mise à jour avec sous-requête (recalculer le total à partir d'une autre table)
-- Exemple : appliquer une remise correspondant à 1% de la valeur de stock de la catégorie
UPDATE commandes  
SET total = (  
    SELECT SUM(p.prix * p.stock)
    FROM produits p
    WHERE p.categorie = 'Informatique'
) * 0.01
WHERE statut = 'en_attente';
```

> 💡 Contrairement aux `CHECK` qui interdisent les sous-requêtes (voir 2.4), `UPDATE` et `INSERT` les acceptent sans restriction.

### 🕒 Modifications avec horodatage

```sql
-- Ajouter une colonne de dernière modification
ALTER TABLE clients ADD COLUMN derniere_modification TEXT;

-- Mettre à jour avec timestamp automatique
UPDATE clients  
SET  
    telephone = '0111222333',
    derniere_modification = datetime('now')
WHERE id = 1;

-- Voir le résultat
SELECT nom, prenom, telephone, derniere_modification  
FROM clients  
WHERE id = 1;  
```

### ⚠️ Sécurité des UPDATE

```sql
-- ❌ DANGER : UPDATE sans WHERE (modifie TOUS les enregistrements)
-- UPDATE clients SET actif = 0;  -- NE PAS FAIRE !

-- ✅ SÉCURISÉ : Toujours utiliser WHERE
UPDATE clients SET actif = 0 WHERE id = 999;  -- Un seul client

-- ✅ Test avant modification massive
SELECT COUNT(*) FROM clients WHERE date_inscription < '2023-01-01';
-- Si le résultat est cohérent, alors :
UPDATE clients SET actif = 0 WHERE date_inscription < '2023-01-01';

-- ✅ Vérification après modification
SELECT actif, COUNT(*) FROM clients GROUP BY actif;
```

## DELETE - Supprimer des données (DELETE)

### 🗑️ Suppression simple

```sql
-- Syntaxe de base
DELETE FROM table WHERE condition;

-- Supprimer un client spécifique
DELETE FROM clients WHERE email = 'test@email.com';

-- Vérifier la suppression
SELECT COUNT(*) FROM clients;
```

### 🧹 Suppressions ciblées

```sql
-- Supprimer les clients inactifs
DELETE FROM clients WHERE actif = 0;

-- Supprimer les produits sans stock
DELETE FROM produits WHERE stock = 0;

-- Supprimer les anciennes commandes annulées
DELETE FROM commandes  
WHERE statut = 'annulee'  
AND date_commande < date('now', '-1 year');  

-- Vérifier les suppressions
SELECT
    (SELECT COUNT(*) FROM clients) as clients_restants,
    (SELECT COUNT(*) FROM produits) as produits_restants,
    (SELECT COUNT(*) FROM commandes) as commandes_restantes;
```

### 🔗 Suppression avec contraintes

```sql
-- Tentative de suppression d'un client avec commandes (peut échouer)
DELETE FROM clients WHERE id = 1;
-- Possible erreur : FOREIGN KEY constraint failed

-- Solution 1 : Supprimer d'abord les commandes liées
DELETE FROM commandes WHERE client_id = 1;  
DELETE FROM clients WHERE id = 1;  

-- Solution 2 : Désactiver plutôt que supprimer
UPDATE clients SET actif = 0 WHERE id = 1;
```

### 🔄 Suppression avec sauvegarde

```sql
-- Créer une table de sauvegarde avant suppression
CREATE TABLE clients_supprimes AS  
SELECT *, datetime('now') as date_suppression  
FROM clients  
WHERE actif = 0;  

-- Vérifier la sauvegarde
SELECT COUNT(*) FROM clients_supprimes;

-- Maintenant supprimer en sécurité
DELETE FROM clients WHERE actif = 0;

-- Vérifier que les données sont bien sauvegardées
SELECT * FROM clients_supprimes;
```

### ⚠️ Sécurité des DELETE

```sql
-- ❌ CATASTROPHIQUE : DELETE sans WHERE
-- DELETE FROM clients;  -- Supprime TOUT !

-- ✅ SÉCURISÉ : Test avec SELECT d'abord
SELECT COUNT(*) FROM commandes WHERE statut = 'annulee';
-- Si le nombre est correct :
DELETE FROM commandes WHERE statut = 'annulee';

-- ✅ Suppression progressive pour gros volumes
DELETE FROM logs WHERE id IN (
    SELECT id FROM logs
    WHERE date_log < date('now', '-30 days')
    LIMIT 1000
);
```

## Transactions — atomicité des opérations CRUD

### 🔒 Pourquoi grouper les opérations en transaction ?

Une **transaction** regroupe plusieurs instructions SQL en une seule unité **atomique** : soit toutes réussissent (`COMMIT`), soit aucune (`ROLLBACK`). C'est essentiel pour maintenir la **cohérence** des données quand plusieurs opérations sont logiquement liées.

```sql
-- ❌ SANS transaction : risque de demi-virement si la deuxième instruction échoue
UPDATE comptes SET solde = solde - 100 WHERE id = 1;
-- ⚠️ Si une coupure de courant arrive ici, le compte 1 est débité mais
--    le compte 2 n'est jamais crédité → argent perdu !
UPDATE comptes SET solde = solde + 100 WHERE id = 2;

-- ✅ AVEC transaction : tout ou rien
BEGIN;
    UPDATE comptes SET solde = solde - 100 WHERE id = 1;
    UPDATE comptes SET solde = solde + 100 WHERE id = 2;
COMMIT;
-- → si quoi que ce soit échoue avant COMMIT, l'ensemble est annulé automatiquement
```

### 🎯 Modes de transaction : DEFERRED, IMMEDIATE, EXCLUSIVE

```sql
-- DEFERRED (défaut) : aucun verrou tant qu'on ne fait que des lectures
BEGIN DEFERRED;  
SELECT * FROM clients;          -- pas encore de verrou  
UPDATE clients SET actif = 0;   -- prend le verrou ICI seulement  
COMMIT;  

-- IMMEDIATE : prend tout de suite un verrou « réservé » (empêche d'autres writers)
-- Recommandé quand vous SAVEZ que vous allez écrire — évite les deadlocks.
BEGIN IMMEDIATE;  
UPDATE clients SET actif = 0 WHERE id = 1;  
COMMIT;  

-- EXCLUSIVE : verrou exclusif d'emblée (bloque même les lectures)
-- Rarement utile sauf migration de schéma.
BEGIN EXCLUSIVE;  
ALTER TABLE clients ADD COLUMN nouveau_champ TEXT;  
COMMIT;  
```

> 💡 **Bonne pratique** : en mode WAL, utilisez `BEGIN IMMEDIATE` quand vous prévoyez d'écrire. Cela évite l'erreur `SQLITE_BUSY` qui survient quand SQLite essaye de passer du mode lecture au mode écriture alors qu'un autre processus écrit déjà.

### 🔁 ROLLBACK et SAVEPOINT

```sql
-- ROLLBACK simple : tout annuler
BEGIN;
    INSERT INTO clients (nom, prenom) VALUES ('Erreur', 'Test');
    -- Oups, mauvais choix
ROLLBACK;  -- la ligne n'existe pas dans la base

-- SAVEPOINT : points de restauration partielle dans une grosse transaction
-- (rappel : la table `clients` exige nom + prenom NOT NULL, donc on fournit les deux)
BEGIN;
    INSERT INTO clients (nom, prenom) VALUES ('Test', 'Alice');

    SAVEPOINT etape_2;
    INSERT INTO clients (nom, prenom) VALUES ('Test', 'Bob');
    INSERT INTO clients (nom, prenom) VALUES ('Test', 'Charlie');
    -- Si on veut annuler uniquement Bob et Charlie :
    ROLLBACK TO etape_2;

    INSERT INTO clients (nom, prenom) VALUES ('Test', 'David');
COMMIT;
-- Résultat : Alice et David sont insérés, Bob et Charlie non.
```

### 🚀 Transactions et performance des INSERT massifs

Sans transaction explicite, chaque `INSERT` est implicitement entouré d'un `BEGIN`/`COMMIT`, ce qui force une **synchronisation disque** à chaque ligne. Pour insérer beaucoup de données, **toujours grouper** :

```sql
-- ❌ LENT : ~1000 commits = ~1000 fsync (peut prendre plusieurs secondes)
INSERT INTO logs (msg) VALUES ('msg 1');  
INSERT INTO logs (msg) VALUES ('msg 2');  
-- ... 1000 fois

-- ✅ RAPIDE : 1 seul commit = 1 fsync
BEGIN;  
INSERT INTO logs (msg) VALUES ('msg 1');  
INSERT INTO logs (msg) VALUES ('msg 2');  
-- ... 1000 fois
COMMIT;
-- Gain de performance souvent ×50 à ×500 selon le disque !
```

## 📥 INSERT INTO ... SELECT — copier d'une table à l'autre

```sql
-- Copier des données d'une table à une autre (archivage, duplication, transformation)
INSERT INTO clients_archives (nom, prenom, email, date_archivage)  
SELECT nom, prenom, email, datetime('now')  
FROM clients  
WHERE actif = 0;  

-- Avec transformation à la volée
INSERT INTO produits_solde (nom, prix_original, prix_solde)  
SELECT  
    nom,
    prix,
    ROUND(prix * 0.7, 2)   -- -30%
FROM produits  
WHERE stock > 50;  

-- Créer une table directement à partir d'une requête
CREATE TABLE rapport_clients_inactifs AS  
SELECT id, nom, prenom, date_inscription  
FROM clients  
WHERE actif = 0;  
-- ⚠️ Limitation : CREATE TABLE AS ne copie PAS les contraintes (PK, FK, CHECK, UNIQUE)
--    ni les index. C'est utile pour des snapshots, pas pour une vraie copie de schéma.
```

## 🎯 Exercice pratique complet - Gestion d'une bibliothèque

### Objectif : Pratiquer toutes les opérations CRUD

```sql
-- === SETUP ===
sqlite3 bibliotheque_crud.db

PRAGMA journal_mode = WAL;

CREATE TABLE auteurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    pays TEXT,
    date_naissance TEXT
);

CREATE TABLE livres (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    auteur_id INTEGER,
    isbn TEXT UNIQUE,
    annee_publication INTEGER,
    pages INTEGER,
    disponible INTEGER DEFAULT 1,
    FOREIGN KEY (auteur_id) REFERENCES auteurs(id)
);

CREATE TABLE emprunts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    livre_id INTEGER,
    emprunteur TEXT NOT NULL,
    date_emprunt TEXT DEFAULT (datetime('now')),
    date_retour_prevue TEXT,
    date_retour_reel TEXT,
    FOREIGN KEY (livre_id) REFERENCES livres(id)
);

-- === CREATE : Ajouter des données ===

-- 1. Ajouter des auteurs
INSERT INTO auteurs (nom, prenom, pays, date_naissance) VALUES
    ('Hugo', 'Victor', 'France', '1802-02-26'),
    ('Orwell', 'George', 'Royaume-Uni', '1903-06-25'),
    ('García Márquez', 'Gabriel', 'Colombie', '1927-03-06'),
    ('Rowling', 'J.K.', 'Royaume-Uni', '1965-07-31');

-- 2. Ajouter des livres
INSERT INTO livres (titre, auteur_id, isbn, annee_publication, pages) VALUES
    ('Les Misérables', 1, '978-2-07-040570-8', 1862, 1488),
    ('Notre-Dame de Paris', 1, '978-2-07-037689-1', 1831, 512),
    ('1984', 2, '978-0-452-28423-4', 1949, 328),
    ('La Ferme des animaux', 2, '978-0-452-28424-1', 1945, 144),
    ('Cent ans de solitude', 3, '978-0-06-088328-7', 1967, 417),
    ('Harry Potter à l''école des sorciers', 4, '978-2-07-054100-0', 1997, 320);

-- 3. Créer quelques emprunts
INSERT INTO emprunts (livre_id, emprunteur, date_retour_prevue) VALUES
    (1, 'Marie Dubois', date('now', '+14 days')),
    (3, 'Pierre Martin', date('now', '+14 days')),
    (6, 'Sophie Leroy', date('now', '+14 days'));

-- === READ : Consulter les données ===

-- 4. Lister tous les livres avec leurs auteurs
SELECT
    l.titre,
    a.nom || ' ' || a.prenom as auteur,
    l.annee_publication,
    l.pages,
    CASE WHEN l.disponible = 1 THEN 'Disponible' ELSE 'Emprunté' END as statut
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
ORDER BY a.nom, l.titre;  

-- 5. Trouver les livres empruntes actuellement
SELECT
    l.titre,
    e.emprunteur,
    e.date_emprunt,
    e.date_retour_prevue,
    CASE
        WHEN date('now') > e.date_retour_prevue THEN 'En retard'
        ELSE 'Dans les temps'
    END as statut_retour
FROM emprunts e  
JOIN livres l ON e.livre_id = l.id  
WHERE e.date_retour_reel IS NULL;  

-- 6. Statistiques par auteur
SELECT
    a.nom || ' ' || a.prenom as auteur,
    COUNT(l.id) as nombre_livres,
    AVG(l.pages) as pages_moyennes,
    MIN(l.annee_publication) as premiere_publication
FROM auteurs a  
LEFT JOIN livres l ON a.id = l.auteur_id  
GROUP BY a.id, a.nom, a.prenom  
ORDER BY nombre_livres DESC;  

-- === UPDATE : Modifier les données ===

-- 7. Marquer un livre comme indisponible quand il est emprunté
UPDATE livres  
SET disponible = 0  
WHERE id IN (SELECT livre_id FROM emprunts WHERE date_retour_reel IS NULL);  

-- 8. Corriger une erreur dans un titre
UPDATE livres  
SET titre = 'Harry Potter à l''école des sorciers (Édition française)'  
WHERE isbn = '978-2-07-054100-0';  

-- 9. Mettre à jour la date de retour réel
UPDATE emprunts  
SET date_retour_reel = datetime('now')  
WHERE livre_id = 1 AND emprunteur = 'Marie Dubois';  

-- Remettre le livre disponible
UPDATE livres  
SET disponible = 1  
WHERE id = 1;  

-- === DELETE : Supprimer des données ===

-- 10. Supprimer les emprunts terminés depuis plus de 1 an
DELETE FROM emprunts  
WHERE date_retour_reel IS NOT NULL  
AND date_retour_reel < date('now', '-1 year');  

-- 11. Supprimer un livre (attention aux contraintes)
-- D'abord supprimer ses emprunts
DELETE FROM emprunts WHERE livre_id = 4;
-- Puis le livre lui-même
DELETE FROM livres WHERE id = 4;

-- === VÉRIFICATIONS FINALES ===

-- Voir l'état final de la bibliothèque
SELECT 'Auteurs' as table_name, COUNT(*) as nombre FROM auteurs  
UNION ALL  
SELECT 'Livres', COUNT(*) FROM livres  
UNION ALL  
SELECT 'Emprunts actifs', COUNT(*) FROM emprunts WHERE date_retour_reel IS NULL  
UNION ALL  
SELECT 'Livres disponibles', COUNT(*) FROM livres WHERE disponible = 1;  

-- Rapport final détaillé
SELECT
    l.titre,
    a.nom as auteur,
    CASE WHEN l.disponible = 1 THEN '📚 Disponible' ELSE '📖 Emprunté' END as statut,
    COALESCE(e.emprunteur, 'Aucun') as emprunteur_actuel
FROM livres l  
JOIN auteurs a ON l.auteur_id = a.id  
LEFT JOIN emprunts e ON l.id = e.livre_id AND e.date_retour_reel IS NULL  
ORDER BY l.titre;  
```

## Récapitulatif des bonnes pratiques CRUD

### ✅ CREATE (INSERT)

- **Spécifiez les colonnes** : `INSERT INTO table (col1, col2) VALUES (...)`
- **Utilisez les insertions multiples** pour de meilleures performances
- **Gérez les conflits** avec `INSERT OR IGNORE`, `INSERT OR REPLACE`, ou **mieux** : `INSERT … ON CONFLICT DO UPDATE` (UPSERT)
- **Récupérez les IDs** avec `RETURNING id` (≥ 3.35) ou `last_insert_rowid()` (à capturer aussitôt)

### ✅ READ (SELECT)

- **Limitez les colonnes** : Sélectionnez seulement ce dont vous avez besoin
- **Utilisez des alias** pour la clarté
- **Indexez les colonnes** fréquemment filtrées
- **Paginez les gros résultats** avec `LIMIT` et `OFFSET`

### ✅ UPDATE

- **TOUJOURS utiliser WHERE** sauf cas très spécifique
- **Testez avec SELECT** avant un UPDATE massif
- **Sauvegardez** avant les modifications importantes
- **Vérifiez le résultat** après modification

### ✅ DELETE

- **JAMAIS sans WHERE** sauf pour vider une table
- **Sauvegardez** les données avant suppression
- **Respectez les contraintes** de clés étrangères
- **Utilisez UPDATE** pour désactiver plutôt que supprimer

### 🔒 Sécurité générale

```sql
-- ✅ Bonne pratique : transaction pour opérations liées
BEGIN TRANSACTION;  
UPDATE clients SET actif = 0 WHERE id = 123;  
INSERT INTO clients_archives SELECT * FROM clients WHERE id = 123;  
DELETE FROM clients WHERE id = 123;  
COMMIT;  

-- ✅ Test de cohérence
SELECT COUNT(*) FROM clients WHERE actif NOT IN (0, 1);  -- Doit être 0

-- ✅ Vérification d'intégrité
PRAGMA integrity_check;
```

## Points clés à retenir

- **CRUD = 90% de votre travail** avec une base de données
- **INSERT** : Privilégier les insertions multiples pour les performances
- **SELECT** : Optimiser avec des index et limiter les résultats
- **UPDATE** : Toujours tester et sauvegarder avant modification
- **DELETE** : Extrême prudence, préférer la désactivation
- **Transactions** : Grouper les opérations liées pour la cohérence
- **Vérifications** : Toujours contrôler les résultats de vos opérations

---

**💡 Dans le prochain chapitre**, nous explorerons les contraintes (PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK) qui garantissent l'intégrité et la cohérence de vos données.

**🎯 Vous maîtrisez maintenant** :
- Les quatre opérations fondamentales CRUD
- Les techniques avancées d'insertion et de modification (`UPSERT` inclus)
- Les bonnes pratiques de sécurité et de performance
- La gestion des erreurs et des contraintes

⏭️ [2.4 Contraintes : PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK](/02-bases-langage-sql-sqlite/04-contraintes-primary-key-foreign-key-unique-check.md)
