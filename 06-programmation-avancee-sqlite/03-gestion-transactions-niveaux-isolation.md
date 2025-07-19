🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.3 Gestion des transactions et niveaux d'isolation

## Qu'est-ce qu'une transaction ?

Une **transaction** est un groupe d'opérations de base de données qui doivent être exécutées ensemble comme une seule unité. Soit toutes les opérations réussissent, soit aucune n'est appliquée.

### Analogie bancaire
Imaginez un virement bancaire entre deux comptes :
1. Débiter 100€ du compte A
2. Créditer 100€ au compte B

Ces deux opérations forment une transaction. Si l'une échoue (par exemple, panne de courant entre les deux), la transaction entière doit être annulée pour éviter de perdre 100€ ou d'en créer de nulle part.

### Les propriétés ACID

Les transactions respectent les propriétés **ACID** :

- **A**tomicité : Tout ou rien - toutes les opérations réussissent ou aucune
- **C**ohérence : La base reste dans un état valide
- **I**solation : Les transactions concurrent n'interfèrent pas
- **D**urabilité : Une fois validée, la transaction est permanente

## Transactions de base dans SQLite

### Syntaxe simple

```sql
-- Démarrer une transaction
BEGIN;

-- Opérations de la transaction
INSERT INTO comptes (nom, solde) VALUES ('Alice', 1000);
UPDATE comptes SET solde = solde - 100 WHERE nom = 'Alice';
UPDATE comptes SET solde = solde + 100 WHERE nom = 'Bob';

-- Valider la transaction
COMMIT;
```

### Annulation d'une transaction

```sql
BEGIN;

INSERT INTO produits (nom, prix) VALUES ('Gadget', 29.99);
UPDATE stock SET quantite = quantite - 1 WHERE produit_id = 1;

-- Oups, erreur détectée - annuler tout
ROLLBACK;
-- Aucune des opérations ci-dessus n'est appliquée
```

### Exemple pratique : Commande e-commerce

```sql
-- Traitement d'une commande complète
BEGIN;

-- 1. Créer la commande
INSERT INTO commandes (client_id, date_commande, total)
VALUES (123, datetime('now'), 89.97);

-- Récupérer l'ID de la commande
-- (En pratique, vous utiliseriez last_insert_rowid())

-- 2. Ajouter les articles
INSERT INTO lignes_commande (commande_id, produit_id, quantite, prix_unitaire)
VALUES (1001, 15, 2, 29.99);

INSERT INTO lignes_commande (commande_id, produit_id, quantite, prix_unitaire)
VALUES (1001, 22, 1, 29.99);

-- 3. Mettre à jour le stock
UPDATE produits SET stock = stock - 2 WHERE id = 15;
UPDATE produits SET stock = stock - 1 WHERE id = 22;

-- 4. Enregistrer le paiement
INSERT INTO paiements (commande_id, montant, statut)
VALUES (1001, 89.97, 'validé');

-- Tout s'est bien passé, valider
COMMIT;
```

## Niveaux d'isolation des transactions

SQLite propose trois niveaux d'isolation, qui contrôlent comment les transactions interagissent entre elles :

### 1. DEFERRED (Par défaut)

La transaction ne verrouille rien au début. Les verrous sont acquis au moment du premier accès.

```sql
-- Transaction différée (comportement par défaut)
BEGIN;
-- ou explicitement :
BEGIN DEFERRED;

SELECT * FROM comptes;  -- Pas de verrou
UPDATE comptes SET solde = 500 WHERE id = 1;  -- Verrou acquis ici
COMMIT;
```

**Avantages :**
- ✅ Performance optimale pour les lectures
- ✅ Verrous minimaux
- ✅ Bon pour les transactions majoritairement en lecture

**Inconvénients :**
- ❌ Risque de conflit si plusieurs écritures simultanées
- ❌ Peut échouer tard dans la transaction

### 2. IMMEDIATE

La transaction acquiert immédiatement un verrou d'écriture (mais permet encore la lecture).

```sql
-- Transaction immédiate
BEGIN IMMEDIATE;

-- Verrou d'écriture déjà acquis
SELECT * FROM comptes;        -- OK, lecture autorisée
UPDATE comptes SET solde = 500 WHERE id = 1;  -- OK, écriture protégée

COMMIT;
```

**Avantages :**
- ✅ Protection contre les conflits d'écriture
- ✅ Détection précoce des problèmes de concurrence
- ✅ Bon pour les transactions mixtes lecture/écriture

**Inconvénients :**
- ❌ Peut bloquer d'autres transactions en écriture
- ❌ Performance réduite par rapport à DEFERRED

### 3. EXCLUSIVE

La transaction acquiert un verrou exclusif total. Aucune autre transaction ne peut accéder à la base.

```sql
-- Transaction exclusive
BEGIN EXCLUSIVE;

-- Accès exclusif total à la base de données
SELECT * FROM comptes;
UPDATE comptes SET solde = 500 WHERE id = 1;
DELETE FROM commandes WHERE date_commande < '2020-01-01';

COMMIT;
```

**Avantages :**
- ✅ Aucun conflit possible
- ✅ Performance maximale pour les opérations lourdes
- ✅ Bon pour les migrations ou maintenance

**Inconvénients :**
- ❌ Bloque complètement les autres accès
- ❌ Peut créer des goulots d'étranglement

## Démonstration pratique des niveaux d'isolation

Créons un exemple concret pour comprendre les différences :

```python
import sqlite3
import threading
import time

# Création d'une base de test
def creer_base_test():
    conn = sqlite3.connect('test_transactions.db')
    conn.execute('''
        CREATE TABLE IF NOT EXISTS compteur (
            id INTEGER PRIMARY KEY,
            valeur INTEGER
        )
    ''')
    conn.execute("INSERT OR REPLACE INTO compteur (id, valeur) VALUES (1, 0)")
    conn.commit()
    conn.close()

def transaction_deferred(nom):
    """Simulation d'une transaction DEFERRED"""
    conn = sqlite3.connect('test_transactions.db')
    try:
        conn.execute("BEGIN DEFERRED")
        print(f"{nom}: Transaction DEFERRED démarrée")

        # Lecture (pas de verrou)
        result = conn.execute("SELECT valeur FROM compteur WHERE id = 1").fetchone()
        valeur_actuelle = result[0]
        print(f"{nom}: Valeur lue = {valeur_actuelle}")

        # Simulation d'un traitement
        time.sleep(2)

        # Écriture (verrou acquis maintenant)
        nouvelle_valeur = valeur_actuelle + 1
        conn.execute("UPDATE compteur SET valeur = ? WHERE id = 1", (nouvelle_valeur,))
        print(f"{nom}: Valeur mise à jour = {nouvelle_valeur}")

        conn.execute("COMMIT")
        print(f"{nom}: Transaction DEFERRED validée")

    except sqlite3.Error as e:
        print(f"{nom}: Erreur - {e}")
        conn.execute("ROLLBACK")
    finally:
        conn.close()

def transaction_immediate(nom):
    """Simulation d'une transaction IMMEDIATE"""
    conn = sqlite3.connect('test_transactions.db')
    try:
        conn.execute("BEGIN IMMEDIATE")
        print(f"{nom}: Transaction IMMEDIATE démarrée (verrou d'écriture acquis)")

        result = conn.execute("SELECT valeur FROM compteur WHERE id = 1").fetchone()
        valeur_actuelle = result[0]
        print(f"{nom}: Valeur lue = {valeur_actuelle}")

        time.sleep(2)

        nouvelle_valeur = valeur_actuelle + 10
        conn.execute("UPDATE compteur SET valeur = ? WHERE id = 1", (nouvelle_valeur,))
        print(f"{nom}: Valeur mise à jour = {nouvelle_valeur}")

        conn.execute("COMMIT")
        print(f"{nom}: Transaction IMMEDIATE validée")

    except sqlite3.Error as e:
        print(f"{nom}: Erreur - {e}")
        conn.execute("ROLLBACK")
    finally:
        conn.close()

# Test de concurrence
if __name__ == "__main__":
    creer_base_test()

    print("=== Test de concurrence DEFERRED ===")
    t1 = threading.Thread(target=transaction_deferred, args=("Thread-1",))
    t2 = threading.Thread(target=transaction_deferred, args=("Thread-2",))

    t1.start()
    time.sleep(0.5)  # Décalage léger
    t2.start()

    t1.join()
    t2.join()

    print("\n=== Test de concurrence IMMEDIATE ===")
    t3 = threading.Thread(target=transaction_immediate, args=("Thread-3",))
    t4 = threading.Thread(target=transaction_immediate, args=("Thread-4",))

    t3.start()
    time.sleep(0.5)
    t4.start()

    t3.join()
    t4.join()
```

## Points de sauvegarde (SAVEPOINT)

Les points de sauvegarde permettent de créer des "sous-transactions" qu'on peut annuler partiellement :

### Syntaxe de base

```sql
BEGIN;

-- Opération 1
INSERT INTO clients (nom) VALUES ('Alice');

-- Créer un point de sauvegarde
SAVEPOINT etape1;

-- Opération 2
INSERT INTO clients (nom) VALUES ('Bob');
UPDATE clients SET email = 'bob@email.com' WHERE nom = 'Bob';

-- Problème détecté, revenir au point de sauvegarde
ROLLBACK TO etape1;
-- Alice est conservée, Bob est supprimé

-- Continuer avec d'autres opérations
INSERT INTO clients (nom) VALUES ('Charlie');

COMMIT;
-- Seuls Alice et Charlie sont dans la base
```

### Exemple pratique : Import de données

```sql
-- Import d'un fichier de données avec gestion d'erreurs
BEGIN;

SAVEPOINT debut_import;

-- Import du premier lot
INSERT INTO produits (nom, prix) VALUES ('Produit A', 10.99);
INSERT INTO produits (nom, prix) VALUES ('Produit B', 15.50);

SAVEPOINT lot1_ok;

-- Import du deuxième lot (peut échouer)
INSERT INTO produits (nom, prix) VALUES ('Produit C', 7.99);
-- INSERT INTO produits (nom, prix) VALUES ('Produit D', 'PRIX_INVALIDE'); -- Erreur !

-- En cas d'erreur, revenir au lot 1
-- ROLLBACK TO lot1_ok;

-- Continuer l'import
INSERT INTO produits (nom, prix) VALUES ('Produit E', 12.75);

-- Libérer le point de sauvegarde (optionnel)
RELEASE lot1_ok;

COMMIT;
```

### Points de sauvegarde imbriqués

```sql
BEGIN;

INSERT INTO commandes (client_id) VALUES (1);
SAVEPOINT commande_creee;

    INSERT INTO lignes_commande (produit_id, quantite) VALUES (101, 2);
    SAVEPOINT ligne1_ajoutee;

        UPDATE stock SET quantite = quantite - 2 WHERE produit_id = 101;
        SAVEPOINT stock1_mis_a_jour;

        -- Si problème ici, on peut revenir à différents niveaux
        -- ROLLBACK TO stock1_mis_a_jour;
        -- ROLLBACK TO ligne1_ajoutee;
        -- ROLLBACK TO commande_creee;

    INSERT INTO lignes_commande (produit_id, quantite) VALUES (102, 1);

COMMIT;
```

## Gestion des erreurs et timeout

### Configuration des timeout

```python
import sqlite3

# Configuration du timeout pour éviter les blocages
conn = sqlite3.connect('ma_base.db', timeout=30.0)  # 30 secondes

# Configuration via PRAGMA
conn.execute("PRAGMA busy_timeout = 30000")  # 30 secondes en millisecondes
```

### Gestion robuste des erreurs

```python
import sqlite3
import time
import random

def transaction_robuste():
    """Transaction avec gestion d'erreurs et retry"""
    max_retries = 3
    retry_count = 0

    while retry_count < max_retries:
        conn = None
        try:
            conn = sqlite3.connect('ma_base.db', timeout=10.0)

            # Démarrer la transaction
            conn.execute("BEGIN IMMEDIATE")

            # Opérations de la transaction
            conn.execute("INSERT INTO logs (message, timestamp) VALUES (?, datetime('now'))",
                        ("Transaction test",))

            # Simulation d'une opération qui peut échouer
            if random.random() < 0.3:  # 30% de chance d'échouer
                raise Exception("Erreur simulée")

            conn.execute("UPDATE compteurs SET valeur = valeur + 1 WHERE nom = 'test'")

            # Valider la transaction
            conn.execute("COMMIT")
            print("Transaction réussie")
            return True

        except sqlite3.DatabaseError as e:
            print(f"Erreur de base de données (tentative {retry_count + 1}): {e}")
            if conn:
                try:
                    conn.execute("ROLLBACK")
                except:
                    pass

        except Exception as e:
            print(f"Erreur générale (tentative {retry_count + 1}): {e}")
            if conn:
                try:
                    conn.execute("ROLLBACK")
                except:
                    pass

        finally:
            if conn:
                conn.close()

        retry_count += 1
        if retry_count < max_retries:
            # Attente exponentielle avant retry
            wait_time = 2 ** retry_count + random.uniform(0, 1)
            print(f"Retry dans {wait_time:.2f} secondes...")
            time.sleep(wait_time)

    print("Transaction échouée après tous les retries")
    return False
```

## Mode WAL (Write-Ahead Logging)

Le mode WAL améliore la gestion des transactions en permettant des lectures simultanées pendant les écritures :

### Activation du mode WAL

```sql
-- Activer le mode WAL
PRAGMA journal_mode = WAL;

-- Vérifier le mode actuel
PRAGMA journal_mode;
```

### Avantages du mode WAL

```python
import sqlite3
import threading
import time

def lecteur_wal(nom):
    """Processus de lecture en mode WAL"""
    conn = sqlite3.connect('base_wal.db')

    for i in range(5):
        try:
            result = conn.execute("SELECT COUNT(*) FROM transactions").fetchone()
            print(f"{nom}: Lecture {i+1} - {result[0]} transactions")
            time.sleep(1)
        except sqlite3.Error as e:
            print(f"{nom}: Erreur lecture - {e}")

    conn.close()

def ecrivain_wal(nom):
    """Processus d'écriture en mode WAL"""
    conn = sqlite3.connect('base_wal.db')

    for i in range(3):
        try:
            conn.execute("BEGIN IMMEDIATE")
            conn.execute("INSERT INTO transactions (description) VALUES (?)",
                        (f"Transaction {nom}-{i+1}",))
            time.sleep(2)  # Simulation d'opération longue
            conn.execute("COMMIT")
            print(f"{nom}: Écriture {i+1} terminée")
        except sqlite3.Error as e:
            print(f"{nom}: Erreur écriture - {e}")
            conn.execute("ROLLBACK")

    conn.close()

# Configuration de la base en mode WAL
def configurer_wal():
    conn = sqlite3.connect('base_wal.db')
    conn.execute("PRAGMA journal_mode = WAL")
    conn.execute('''
        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY,
            description TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.close()

# Test de concurrence en mode WAL
if __name__ == "__main__":
    configurer_wal()

    print("=== Test mode WAL - Lectures simultanées pendant écritures ===")

    # Démarrer un écrivain
    t_ecrivain = threading.Thread(target=ecrivain_wal, args=("Ecrivain",))

    # Démarrer plusieurs lecteurs
    t_lecteur1 = threading.Thread(target=lecteur_wal, args=("Lecteur-1",))
    t_lecteur2 = threading.Thread(target=lecteur_wal, args=("Lecteur-2",))

    t_ecrivain.start()
    time.sleep(0.5)
    t_lecteur1.start()
    t_lecteur2.start()

    t_ecrivain.join()
    t_lecteur1.join()
    t_lecteur2.join()
```

## Optimisation des performances transactionnelles

### Regroupement d'opérations

```python
# ❌ Inefficace : Une transaction par opération
def insert_lent(donnees):
    conn = sqlite3.connect('ma_base.db')
    for item in donnees:
        conn.execute("BEGIN")
        conn.execute("INSERT INTO items (nom, valeur) VALUES (?, ?)", item)
        conn.execute("COMMIT")
    conn.close()

# ✅ Efficace : Toutes les opérations dans une transaction
def insert_rapide(donnees):
    conn = sqlite3.connect('ma_base.db')
    conn.execute("BEGIN")
    for item in donnees:
        conn.execute("INSERT INTO items (nom, valeur) VALUES (?, ?)", item)
    conn.execute("COMMIT")
    conn.close()

# ✅ Encore plus efficace : Avec executemany
def insert_optimise(donnees):
    conn = sqlite3.connect('ma_base.db')
    conn.execute("BEGIN")
    conn.executemany("INSERT INTO items (nom, valeur) VALUES (?, ?)", donnees)
    conn.execute("COMMIT")
    conn.close()
```

### Configuration des performances

```sql
-- Optimisations pour les transactions
PRAGMA synchronous = NORMAL;  -- Balance sécurité/performance
PRAGMA cache_size = 10000;    -- Cache plus important
PRAGMA temp_store = MEMORY;   -- Stocker les temp en RAM
PRAGMA journal_mode = WAL;    -- Mode WAL pour concurrence
```

## Patterns courants de transactions

### 1. Pattern de mise à jour conditionnelle

```sql
BEGIN;

-- Vérifier une condition avant mise à jour
SELECT stock FROM produits WHERE id = 123;

-- Si stock suffisant (vérifié par l'application)
UPDATE produits SET stock = stock - 5 WHERE id = 123 AND stock >= 5;

-- Vérifier que la mise à jour a eu lieu
-- Si aucune ligne affectée, le stock était insuffisant
-- ROLLBACK ou COMMIT selon le résultat

COMMIT;
```

### 2. Pattern d'audit automatique

```sql
BEGIN;

-- Opération principale
UPDATE comptes SET solde = solde + 100 WHERE numero = '12345';

-- Enregistrement automatique de l'audit
INSERT INTO audit_comptes (
    numero_compte,
    ancienne_valeur,
    nouvelle_valeur,
    operation,
    timestamp,
    utilisateur
) VALUES (
    '12345',
    (SELECT solde - 100 FROM comptes WHERE numero = '12345'),
    (SELECT solde FROM comptes WHERE numero = '12345'),
    'CREDIT',
    datetime('now'),
    'user123'
);

COMMIT;
```

### 3. Pattern de queue de traitement

```sql
-- Traitement atomique d'une queue
BEGIN;

-- Prendre le premier élément de la queue
SELECT id, donnees FROM queue_traitement
WHERE statut = 'en_attente'
ORDER BY date_creation
LIMIT 1;

-- Marquer comme en cours de traitement
UPDATE queue_traitement
SET statut = 'en_cours', date_debut = datetime('now')
WHERE id = ?;

COMMIT;

-- Traiter les données (hors transaction)
-- ...

-- Marquer comme terminé
BEGIN;
UPDATE queue_traitement
SET statut = 'termine', date_fin = datetime('now')
WHERE id = ?;
COMMIT;
```

## Debugging et monitoring des transactions

### Surveillance des verrous

```sql
-- Voir les informations sur les verrous (nécessite compilation spéciale)
PRAGMA lock_status;

-- Voir le mode de journal actuel
PRAGMA journal_mode;

-- Statistiques sur la base
PRAGMA database_list;
```

### Logging des transactions longues

```python
import sqlite3
import time
import logging

class TransactionLogger:
    def __init__(self, db_path, seuil_seconde=5):
        self.db_path = db_path
        self.seuil_seconde = seuil_seconde
        self.logger = logging.getLogger('transactions')

    def __enter__(self):
        self.conn = sqlite3.connect(self.db_path)
        self.debut = time.time()
        self.conn.execute("BEGIN")
        return self.conn

    def __exit__(self, exc_type, exc_val, exc_tb):
        try:
            if exc_type is None:
                self.conn.execute("COMMIT")
                duree = time.time() - self.debut
                if duree > self.seuil_seconde:
                    self.logger.warning(f"Transaction longue: {duree:.2f}s")
            else:
                self.conn.execute("ROLLBACK")
                self.logger.error(f"Transaction annulée: {exc_val}")
        finally:
            self.conn.close()

# Utilisation
with TransactionLogger('ma_base.db') as conn:
    conn.execute("INSERT INTO logs (message) VALUES (?)", ("Test",))
    time.sleep(6)  # Simulation d'opération longue
    # Warning automatique si > 5 secondes
```

## Cas d'usage avancés

### Migration de données avec rollback

```python
def migrer_donnees_avec_rollback():
    """Migration avec possibilité de rollback complet"""
    conn = sqlite3.connect('ma_base.db')

    try:
        conn.execute("BEGIN EXCLUSIVE")  # Verrou total pour migration

        # Étape 1: Sauvegarder les données existantes
        conn.execute("""
            CREATE TABLE backup_clients AS
            SELECT * FROM clients
        """)

        conn.execute("SAVEPOINT backup_cree")

        # Étape 2: Modifier la structure
        conn.execute("ALTER TABLE clients ADD COLUMN email_verifie BOOLEAN DEFAULT 0")

        conn.execute("SAVEPOINT structure_modifiee")

        # Étape 3: Migrer les données
        conn.execute("""
            UPDATE clients SET email_verifie = 1
            WHERE email IS NOT NULL AND email LIKE '%@%.%'
        """)

        # Étape 4: Vérifier l'intégrité
        result = conn.execute("SELECT COUNT(*) FROM clients WHERE email_verifie IS NULL").fetchone()
        if result[0] > 0:
            raise Exception("Données corrompues détectées")

        # Étape 5: Nettoyer la sauvegarde
        conn.execute("DROP TABLE backup_clients")

        conn.execute("COMMIT")
        print("Migration réussie")

    except Exception as e:
        print(f"Erreur de migration: {e}")
        print("Rollback en cours...")

        try:
            # Restaurer depuis la sauvegarde si elle existe
            conn.execute("ROLLBACK TO backup_cree")
            conn.execute("DROP TABLE IF EXISTS backup_clients")
            conn.execute("ROLLBACK")
        except:
            conn.execute("ROLLBACK")

        print("Rollback terminé")

    finally:
        conn.close()
```

## Récapitulatif et bonnes pratiques

### Règles d'or des transactions

1. **Minimiser la durée** : Transactions courtes = moins de conflits
2. **Gérer les erreurs** : Toujours avoir un plan de rollback
3. **Choisir le bon niveau** : DEFERRED pour lecture, IMMEDIATE pour écriture
4. **Utiliser les savepoints** : Pour les opérations complexes
5. **Optimiser les performances** : Regrouper les opérations

### Checklist de débogage

Quand une transaction pose problème :

- ✅ Vérifier les verrous (timeout, conflits)
- ✅ Mesurer la durée d'exécution
- ✅ Analyser les points de contention
- ✅ Vérifier la gestion d'erreurs
- ✅ Tester la concurrence

### Patterns à éviter

```sql
-- ❌ Transactions imbriquées (SQLite ne les supporte pas vraiment)
BEGIN;
  BEGIN;  -- Ignoré !
  COMMIT; -- Commit de la transaction principale !
COMMIT;   -- Erreur : pas de transaction active

-- ❌ Transactions trop longues
BEGIN;
-- 1000 opérations complexes
COMMIT;

-- ❌ Oublier la gestion d'erreurs
BEGIN;
UPDATE comptes SET solde = solde - 1000 WHERE id = 1;
-- Si erreur ici, la transaction reste ouverte !
COMMIT;
```

### Ce que vous avez appris

- ✅ **Concept des transactions** : Atomicité, cohérence, isolation, durabilité
- ✅ **Niveaux d'isolation** : DEFERRED, IMMEDIATE, EXCLUSIVE
- ✅ **Points de sauvegarde** : Rollback partiel avec SAVEPOINT
- ✅ **Gestion d'erreurs** : Timeout, retry, logging
- ✅ **Mode WAL** : Amélioration de la concurrence
- ✅ **Optimisation** : Regroupement, configuration
- ✅ **Patterns avancés** : Migration, audit, queue

### Prochaines étapes

Dans la section suivante (6.4), nous explorerons la **sauvegarde et restauration** avec l'API de backup de SQLite, incluant les stratégies de sauvegarde à chaud et la réplication de données.

⏭️
