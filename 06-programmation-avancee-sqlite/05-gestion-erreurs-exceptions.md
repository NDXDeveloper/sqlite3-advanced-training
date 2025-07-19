🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.5 Gestion des erreurs et exceptions

## Pourquoi gérer les erreurs ?

La gestion d'erreurs est **cruciale** dans toute application utilisant une base de données. Sans elle, votre application peut planter, corrompre des données, ou laisser l'utilisateur dans l'incertitude.

### Analogie simple
Imaginez conduire une voiture sans tableau de bord : pas de voyant d'essence, de température, ou de vitesse. Vous pourriez rouler un moment, mais au premier problème, vous seriez complètement perdu ! Les erreurs SQLite sont comme ces voyants : elles vous préviennent avant que les choses tournent mal.

### Conséquences d'une mauvaise gestion d'erreurs
- 💥 **Crash de l'application** : Arrêt brutal sans explication
- 🗃️ **Corruption de données** : Transactions incomplètes
- 😕 **Expérience utilisateur dégradée** : Messages d'erreur incompréhensibles
- 🐛 **Difficultés de debug** : Impossible de comprendre ce qui s'est passé
- 🔒 **Verrous bloqués** : Ressources non libérées

## Types d'erreurs SQLite

### 1. Erreurs de syntaxe SQL

Les plus courantes pour les débutants :

```python
import sqlite3

def erreur_syntaxe():
    """Exemple d'erreur de syntaxe"""
    conn = sqlite3.connect(':memory:')

    try:
        # Erreur : mot-clé manquant
        conn.execute("CREATE clients (id INTEGER, nom TEXT)")
    except sqlite3.Error as e:
        print(f"❌ Erreur de syntaxe : {e}")
        # Résultat : near "clients": syntax error

    try:
        # Erreur : virgule en trop
        conn.execute("SELECT nom, age, FROM clients")
    except sqlite3.Error as e:
        print(f"❌ Erreur de syntaxe : {e}")
        # Résultat : near "FROM": syntax error

    conn.close()

# Correction avec gestion d'erreur
def creation_table_securisee():
    """Création de table avec gestion d'erreur"""
    conn = sqlite3.connect(':memory:')

    try:
        # SQL correct
        conn.execute("""
            CREATE TABLE IF NOT EXISTS clients (
                id INTEGER PRIMARY KEY,
                nom TEXT NOT NULL,
                email TEXT UNIQUE
            )
        """)
        print("✅ Table créée avec succès")

    except sqlite3.Error as e:
        print(f"❌ Erreur création table : {e}")
        return False
    finally:
        conn.close()

    return True

creation_table_securisee()
```

### 2. Erreurs de contraintes

Violations des règles définies dans la base :

```python
def demo_erreurs_contraintes():
    """Démonstration des erreurs de contraintes"""
    conn = sqlite3.connect(':memory:')

    # Créer une table avec contraintes
    conn.execute("""
        CREATE TABLE utilisateurs (
            id INTEGER PRIMARY KEY,
            email TEXT UNIQUE NOT NULL,
            age INTEGER CHECK (age >= 0 AND age <= 150),
            solde REAL DEFAULT 0 CHECK (solde >= 0)
        )
    """)

    # Test des différentes contraintes
    tests = [
        # (description, SQL, données)
        ("Insertion normale", "INSERT INTO utilisateurs (email, age, solde) VALUES (?, ?, ?)",
         ("alice@email.com", 25, 100.0)),

        ("Email en double", "INSERT INTO utilisateurs (email, age) VALUES (?, ?)",
         ("alice@email.com", 30)),

        ("Email NULL", "INSERT INTO utilisateurs (age) VALUES (?)",
         (25,)),

        ("Âge négatif", "INSERT INTO utilisateurs (email, age) VALUES (?, ?)",
         ("bob@email.com", -5)),

        ("Solde négatif", "INSERT INTO utilisateurs (email, age, solde) VALUES (?, ?, ?)",
         ("charlie@email.com", 30, -50.0))
    ]

    for description, sql, donnees in tests:
        try:
            conn.execute(sql, donnees)
            print(f"✅ {description} : Succès")
        except sqlite3.IntegrityError as e:
            print(f"❌ {description} : {e}")
        except sqlite3.Error as e:
            print(f"❌ {description} : Erreur générale - {e}")

    conn.close()

demo_erreurs_contraintes()
```

### 3. Erreurs de verrouillage

Problèmes de concurrence entre transactions :

```python
import threading
import time

def demo_erreurs_verrouillage():
    """Démonstration des erreurs de verrouillage"""

    # Créer une base partagée
    conn_setup = sqlite3.connect('test_verrous.db')
    conn_setup.execute("""
        CREATE TABLE IF NOT EXISTS compteur (
            id INTEGER PRIMARY KEY,
            valeur INTEGER
        )
    """)
    conn_setup.execute("INSERT OR REPLACE INTO compteur (id, valeur) VALUES (1, 0)")
    conn_setup.commit()
    conn_setup.close()

    def transaction_longue(nom):
        """Transaction qui prend du temps"""
        conn = sqlite3.connect('test_verrous.db', timeout=2.0)  # Timeout de 2 secondes

        try:
            print(f"🔄 {nom} : Début de transaction")
            conn.execute("BEGIN IMMEDIATE")

            # Lire la valeur actuelle
            result = conn.execute("SELECT valeur FROM compteur WHERE id = 1").fetchone()
            valeur = result[0]
            print(f"📖 {nom} : Valeur lue = {valeur}")

            # Simulation d'un traitement long
            time.sleep(3)

            # Mettre à jour
            conn.execute("UPDATE compteur SET valeur = ? WHERE id = 1", (valeur + 1,))
            conn.commit()
            print(f"✅ {nom} : Transaction terminée, nouvelle valeur = {valeur + 1}")

        except sqlite3.OperationalError as e:
            if "database is locked" in str(e):
                print(f"🔒 {nom} : Base verrouillée - {e}")
            else:
                print(f"❌ {nom} : Erreur opérationnelle - {e}")
        except sqlite3.Error as e:
            print(f"❌ {nom} : Erreur - {e}")
        finally:
            try:
                conn.close()
            except:
                pass

    # Lancer deux transactions simultanées
    thread1 = threading.Thread(target=transaction_longue, args=("Thread-1",))
    thread2 = threading.Thread(target=transaction_longue, args=("Thread-2",))

    thread1.start()
    time.sleep(0.5)  # Décalage léger
    thread2.start()

    thread1.join()
    thread2.join()

demo_erreurs_verrouillage()
```

### 4. Erreurs de connexion

Problèmes d'accès à la base de données :

```python
def demo_erreurs_connexion():
    """Démonstration des erreurs de connexion"""

    tests_connexion = [
        ("Base inexistante", "/chemin/inexistant/base.db"),
        ("Répertoire protégé", "/root/base.db"),
        ("Fichier corrompu", "fichier_corrompu.db"),
        ("Connexion normale", ":memory:")
    ]

    for description, chemin in tests_connexion:
        try:
            if description == "Fichier corrompu":
                # Créer un faux fichier corrompu
                with open("fichier_corrompu.db", "w") as f:
                    f.write("Ce n'est pas une base SQLite!")

            conn = sqlite3.connect(chemin)
            # Tester la connexion avec une requête simple
            conn.execute("SELECT 1")
            print(f"✅ {description} : Connexion réussie")
            conn.close()

        except sqlite3.DatabaseError as e:
            print(f"🗃️ {description} : Erreur de base de données - {e}")
        except sqlite3.OperationalError as e:
            print(f"⚙️ {description} : Erreur opérationnelle - {e}")
        except PermissionError as e:
            print(f"🔒 {description} : Erreur de permission - {e}")
        except Exception as e:
            print(f"❌ {description} : Erreur inattendue - {e}")

demo_erreurs_connexion()
```

## Hiérarchie des exceptions SQLite

### Structure des exceptions

```python
def demo_hierarchie_exceptions():
    """Démonstration de la hiérarchie des exceptions SQLite"""

    conn = sqlite3.connect(':memory:')

    # Types d'exceptions spécifiques
    exceptions_tests = [
        ("sqlite3.Warning", "Avertissement (rarement utilisé)"),
        ("sqlite3.Error", "Classe de base pour toutes les erreurs SQLite"),
        ("sqlite3.InterfaceError", "Erreur d'interface (problème de module)"),
        ("sqlite3.DatabaseError", "Erreur liée à la base de données"),
        ("sqlite3.DataError", "Problème avec les données"),
        ("sqlite3.OperationalError", "Erreur opérationnelle (verrous, etc.)"),
        ("sqlite3.IntegrityError", "Violation de contrainte d'intégrité"),
        ("sqlite3.InternalError", "Erreur interne SQLite"),
        ("sqlite3.ProgrammingError", "Erreur de programmation"),
        ("sqlite3.NotSupportedError", "Fonctionnalité non supportée")
    ]

    print("📋 Hiérarchie des exceptions SQLite :")
    print("=" * 50)
    for exception, description in exceptions_tests:
        print(f"{exception:25} : {description}")

    # Exemple pratique de capture spécifique
    conn.execute("""
        CREATE TABLE test (
            id INTEGER PRIMARY KEY,
            email TEXT UNIQUE
        )
    """)

    try:
        # Première insertion - OK
        conn.execute("INSERT INTO test (email) VALUES (?)", ("test@email.com",))

        # Deuxième insertion - Erreur de contrainte
        conn.execute("INSERT INTO test (email) VALUES (?)", ("test@email.com",))

    except sqlite3.IntegrityError as e:
        print(f"\n🎯 IntegrityError capturée : {e}")
        print("   → Action : Informer l'utilisateur que l'email existe déjà")

    except sqlite3.ProgrammingError as e:
        print(f"\n🎯 ProgrammingError capturée : {e}")
        print("   → Action : Vérifier le code SQL")

    except sqlite3.OperationalError as e:
        print(f"\n🎯 OperationalError capturée : {e}")
        print("   → Action : Réessayer ou informer d'un problème temporaire")

    except sqlite3.Error as e:
        print(f"\n🎯 Erreur SQLite générale : {e}")
        print("   → Action : Gestion d'erreur générique")

    conn.close()

demo_hierarchie_exceptions()
```

## Stratégies de gestion d'erreurs

### 1. Gestion basique avec try/except

```python
def insertion_securisee(conn, email, nom):
    """Insertion avec gestion d'erreur basique"""
    try:
        conn.execute(
            "INSERT INTO utilisateurs (email, nom) VALUES (?, ?)",
            (email, nom)
        )
        conn.commit()
        return True, "Utilisateur créé avec succès"

    except sqlite3.IntegrityError as e:
        if "UNIQUE constraint failed" in str(e):
            return False, "Cet email est déjà utilisé"
        else:
            return False, f"Erreur d'intégrité : {e}"

    except sqlite3.Error as e:
        return False, f"Erreur de base de données : {e}"

# Test de la fonction
conn = sqlite3.connect(':memory:')
conn.execute("""
    CREATE TABLE utilisateurs (
        id INTEGER PRIMARY KEY,
        email TEXT UNIQUE,
        nom TEXT
    )
""")

# Tests
tests = [
    ("alice@email.com", "Alice"),
    ("bob@email.com", "Bob"),
    ("alice@email.com", "Alice Dupont"),  # Email en double
]

for email, nom in tests:
    succes, message = insertion_securisee(conn, email, nom)
    print(f"{'✅' if succes else '❌'} {email} : {message}")

conn.close()
```

### 2. Gestionnaire de contexte personnalisé

```python
class GestionnaireConnexionSQLite:
    """Gestionnaire de contexte pour SQLite avec gestion d'erreurs"""

    def __init__(self, db_path, **kwargs):
        self.db_path = db_path
        self.conn_kwargs = kwargs
        self.conn = None

    def __enter__(self):
        try:
            self.conn = sqlite3.connect(self.db_path, **self.conn_kwargs)
            # Configuration recommandée
            self.conn.execute("PRAGMA foreign_keys = ON")
            return self.conn
        except sqlite3.Error as e:
            raise sqlite3.OperationalError(f"Impossible de se connecter à {self.db_path}: {e}")

    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.conn:
            if exc_type is None:
                # Pas d'exception : commit automatique
                try:
                    self.conn.commit()
                except sqlite3.Error as e:
                    print(f"⚠️ Erreur lors du commit : {e}")
                    self.conn.rollback()
            else:
                # Exception : rollback automatique
                try:
                    self.conn.rollback()
                    print("🔄 Rollback automatique effectué")
                except sqlite3.Error as e:
                    print(f"❌ Erreur lors du rollback : {e}")

            self.conn.close()

        # Ne pas supprimer l'exception
        return False

# Utilisation du gestionnaire de contexte
def demo_gestionnaire_contexte():
    """Démonstration du gestionnaire de contexte"""

    # Cas normal (succès)
    try:
        with GestionnaireConnexionSQLite(':memory:') as conn:
            conn.execute("""
                CREATE TABLE produits (
                    id INTEGER PRIMARY KEY,
                    nom TEXT NOT NULL,
                    prix REAL CHECK (prix > 0)
                )
            """)

            conn.execute("INSERT INTO produits (nom, prix) VALUES (?, ?)", ("Produit A", 29.99))
            conn.execute("INSERT INTO produits (nom, prix) VALUES (?, ?)", ("Produit B", 15.50))

            print("✅ Insertion réussie, commit automatique")

    except sqlite3.Error as e:
        print(f"❌ Erreur : {e}")

    # Cas d'erreur (rollback automatique)
    try:
        with GestionnaireConnexionSQLite(':memory:') as conn:
            conn.execute("""
                CREATE TABLE produits (
                    id INTEGER PRIMARY KEY,
                    nom TEXT NOT NULL,
                    prix REAL CHECK (prix > 0)
                )
            """)

            conn.execute("INSERT INTO produits (nom, prix) VALUES (?, ?)", ("Produit A", 29.99))
            # Cette ligne va provoquer une erreur
            conn.execute("INSERT INTO produits (nom, prix) VALUES (?, ?)", ("Produit B", -10.0))

    except sqlite3.Error as e:
        print(f"❌ Erreur capturée : {e}")
        print("   Rollback automatique effectué par le gestionnaire de contexte")

demo_gestionnaire_contexte()
```

### 3. Décorateur pour gestion d'erreurs

```python
from functools import wraps
import logging

def gestion_erreurs_sqlite(retry_count=3, delay=1):
    """Décorateur pour gestion automatique des erreurs SQLite"""

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None

            for tentative in range(retry_count):
                try:
                    return func(*args, **kwargs)

                except sqlite3.OperationalError as e:
                    last_exception = e
                    if "database is locked" in str(e) and tentative < retry_count - 1:
                        print(f"🔒 Base verrouillée, tentative {tentative + 1}/{retry_count}")
                        time.sleep(delay * (tentative + 1))  # Backoff exponentiel
                        continue
                    else:
                        break

                except sqlite3.IntegrityError as e:
                    # Ne pas retry les erreurs d'intégrité
                    print(f"❌ Erreur d'intégrité : {e}")
                    raise

                except sqlite3.Error as e:
                    print(f"❌ Erreur SQLite : {e}")
                    raise

            # Si on arrive ici, tous les retries ont échoué
            print(f"❌ Échec après {retry_count} tentatives")
            raise last_exception

        return wrapper
    return decorator

# Utilisation du décorateur
@gestion_erreurs_sqlite(retry_count=3, delay=0.5)
def operation_avec_retry(db_path, operations):
    """Opération qui peut nécessiter des retries"""

    with sqlite3.connect(db_path, timeout=1.0) as conn:
        conn.execute("BEGIN IMMEDIATE")

        for operation in operations:
            conn.execute(operation['sql'], operation.get('params', ()))

        conn.commit()
        print("✅ Toutes les opérations réussies")

# Test du décorateur
def demo_decorateur():
    """Test du décorateur de gestion d'erreurs"""

    # Créer une base de test
    with sqlite3.connect('test_retry.db') as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS logs (
                id INTEGER PRIMARY KEY,
                message TEXT,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        """)

    operations = [
        {'sql': "INSERT INTO logs (message) VALUES (?)", 'params': ("Test message 1",)},
        {'sql': "INSERT INTO logs (message) VALUES (?)", 'params': ("Test message 2",)},
        {'sql': "INSERT INTO logs (message) VALUES (?)", 'params': ("Test message 3",)},
    ]

    try:
        operation_avec_retry('test_retry.db', operations)
    except sqlite3.Error as e:
        print(f"❌ Opération finale échouée : {e}")

demo_decorateur()
```

## Logging et diagnostic

### 1. Configuration du logging

```python
import logging
import traceback
from datetime import datetime

class SQLiteLogger:
    """Logger spécialisé pour SQLite"""

    def __init__(self, name="sqlite_app", log_file="sqlite_errors.log"):
        self.logger = logging.getLogger(name)
        self.logger.setLevel(logging.DEBUG)

        # Éviter les doublons de handlers
        if not self.logger.handlers:
            # Handler pour fichier
            file_handler = logging.FileHandler(log_file)
            file_handler.setLevel(logging.WARNING)

            # Handler pour console
            console_handler = logging.StreamHandler()
            console_handler.setLevel(logging.INFO)

            # Format détaillé
            formatter = logging.Formatter(
                '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
            )

            file_handler.setFormatter(formatter)
            console_handler.setFormatter(formatter)

            self.logger.addHandler(file_handler)
            self.logger.addHandler(console_handler)

    def log_error(self, error, context=None, sql=None, params=None):
        """Log une erreur SQLite avec contexte"""

        error_info = {
            'type': type(error).__name__,
            'message': str(error),
            'timestamp': datetime.now().isoformat(),
        }

        if context:
            error_info['context'] = context

        if sql:
            error_info['sql'] = sql

        if params:
            error_info['params'] = str(params)

        # Log de l'erreur
        self.logger.error(f"Erreur SQLite: {error_info}")

        # Log de la stack trace pour debug
        self.logger.debug(f"Stack trace: {traceback.format_exc()}")

        return error_info

    def log_performance(self, operation, duration, rows_affected=None):
        """Log les performances d'une opération"""

        perf_info = f"Operation: {operation}, Duration: {duration:.3f}s"
        if rows_affected is not None:
            perf_info += f", Rows: {rows_affected}"

        if duration > 1.0:  # Plus d'une seconde
            self.logger.warning(f"Opération lente - {perf_info}")
        else:
            self.logger.info(f"Performance - {perf_info}")

# Utilisation du logger
def demo_logging():
    """Démonstration du logging SQLite"""

    sqlite_logger = SQLiteLogger("demo_app")

    conn = sqlite3.connect(':memory:')

    try:
        # Opération normale
        start_time = time.time()
        conn.execute("""
            CREATE TABLE clients (
                id INTEGER PRIMARY KEY,
                email TEXT UNIQUE,
                nom TEXT
            )
        """)
        duration = time.time() - start_time
        sqlite_logger.log_performance("CREATE TABLE", duration)

        # Opération avec erreur
        try:
            conn.execute("INSERT INTO clients (email, nom) VALUES (?, ?)", ("test@email.com", "Test"))
            conn.execute("INSERT INTO clients (email, nom) VALUES (?, ?)", ("test@email.com", "Test2"))
        except sqlite3.Error as e:
            sqlite_logger.log_error(
                e,
                context="Insertion client",
                sql="INSERT INTO clients (email, nom) VALUES (?, ?)",
                params=("test@email.com", "Test2")
            )

    finally:
        conn.close()

demo_logging()
```

### 2. Diagnostic avancé

```python
class DiagnosticSQLite:
    """Outils de diagnostic pour SQLite"""

    def __init__(self, db_path):
        self.db_path = db_path

    def verifier_integrite(self):
        """Vérifie l'intégrité de la base de données"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                # Test d'intégrité complet
                result = conn.execute("PRAGMA integrity_check").fetchall()

                if len(result) == 1 and result[0][0] == "ok":
                    print("✅ Intégrité de la base : OK")
                    return True
                else:
                    print("❌ Problèmes d'intégrité détectés :")
                    for issue in result:
                        print(f"   • {issue[0]}")
                    return False

        except sqlite3.Error as e:
            print(f"❌ Erreur lors de la vérification : {e}")
            return False

    def analyser_performance(self):
        """Analyse les performances de la base"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                # Informations générales
                info = {
                    'page_size': conn.execute("PRAGMA page_size").fetchone()[0],
                    'cache_size': conn.execute("PRAGMA cache_size").fetchone()[0],
                    'journal_mode': conn.execute("PRAGMA journal_mode").fetchone()[0],
                    'synchronous': conn.execute("PRAGMA synchronous").fetchone()[0],
                }

                print("📊 Informations de performance :")
                for key, value in info.items():
                    print(f"   {key}: {value}")

                # Tables et leurs tailles
                tables = conn.execute("""
                    SELECT name FROM sqlite_master
                    WHERE type='table' AND name NOT LIKE 'sqlite_%'
                """).fetchall()

                print("\n📋 Analyse des tables :")
                for (table_name,) in tables:
                    count = conn.execute(f"SELECT COUNT(*) FROM {table_name}").fetchone()[0]
                    print(f"   {table_name}: {count} lignes")

                return info

        except sqlite3.Error as e:
            print(f"❌ Erreur lors de l'analyse : {e}")
            return None

    def lister_verrous_actifs(self):
        """Liste les verrous actifs (nécessite des outils spéciaux)"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                # Tenter d'acquérir un verrou exclusif
                conn.execute("BEGIN EXCLUSIVE")
                print("✅ Aucun verrou actif détecté")
                conn.rollback()
                return True

        except sqlite3.OperationalError as e:
            if "database is locked" in str(e):
                print("🔒 Base de données verrouillée")
                return False
            else:
                print(f"❌ Erreur : {e}")
                return False

    def optimiser_base(self):
        """Suggère des optimisations"""
        suggestions = []

        try:
            with sqlite3.connect(self.db_path) as conn:
                # Vérifier la fragmentation
                page_count = conn.execute("PRAGMA page_count").fetchone()[0]
                freelist_count = conn.execute("PRAGMA freelist_count").fetchone()[0]

                if freelist_count > page_count * 0.1:  # Plus de 10% de pages libres
                    suggestions.append("🔧 Exécuter VACUUM pour défragmenter")

                # Vérifier les statistiques des index
                try:
                    conn.execute("ANALYZE")
                    suggestions.append("📊 Statistiques mises à jour avec ANALYZE")
                except sqlite3.Error:
                    suggestions.append("⚠️ Impossible de mettre à jour les statistiques")

                # Vérifier le mode WAL
                journal_mode = conn.execute("PRAGMA journal_mode").fetchone()[0]
                if journal_mode != "wal":
                    suggestions.append("💡 Considérer le mode WAL pour améliorer la concurrence")

        except sqlite3.Error as e:
            suggestions.append(f"❌ Erreur lors de l'analyse : {e}")

        print("\n💡 Suggestions d'optimisation :")
        for suggestion in suggestions:
            print(f"   {suggestion}")

        return suggestions

# Test du diagnostic
def demo_diagnostic():
    """Démonstration des outils de diagnostic"""

    # Créer une base de test
    with sqlite3.connect('test_diagnostic.db') as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS test_data (
                id INTEGER PRIMARY KEY,
                data TEXT,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        """)

        # Ajouter des données
        for i in range(1000):
            conn.execute("INSERT INTO test_data (data) VALUES (?)", (f"Data {i}",))

        conn.commit()

    # Effectuer le diagnostic
    diagnostic = DiagnosticSQLite('test_diagnostic.db')

    print("🔍 Vérification d'intégrité :")
    diagnostic.verifier_integrite()

    print("\n📊 Analyse de performance :")
    diagnostic.analyser_performance()

    print("\n🔒 Vérification des verrous :")
    diagnostic.lister_verrous_actifs()

    print("\n💡 Optimisations :")
    diagnostic.optimiser_base()

demo_diagnostic()
```

## Stratégies de récupération

### 1. Récupération automatique
```python
class RecuperationAutomatique:
    """Système de récupération automatique d'erreurs"""

    def __init__(self, db_path, backup_path=None):
        self.db_path = db_path
        self.backup_path = backup_path or f"{db_path}.backup"
        self.logger = SQLiteLogger("recovery")

    def creer_backup_urgence(self):
        """Crée un backup d'urgence avant récupération"""
        try:
            import shutil
            backup_urgence = f"{self.db_path}.emergency_{int(time.time())}"
            shutil.copy2(self.db_path, backup_urgence)
            self.logger.logger.info(f"Backup d'urgence créé : {backup_urgence}")
            return backup_urgence
        except Exception as e:
            self.logger.log_error(e, "Création backup d'urgence")
            return None

    def tenter_reparation(self):
        """Tente de réparer une base corrompue"""
        strategies = [
            self._reparation_dump_restore,
            self._reparation_vacuum,
            self._reparation_reindex,
        ]

        for i, strategie in enumerate(strategies, 1):
            print(f"🔧 Tentative de réparation {i}/{len(strategies)}")
            try:
                if strategie():
                    print(f"✅ Réparation réussie avec la stratégie {i}")
                    return True
            except Exception as e:
                self.logger.log_error(e, f"Stratégie de réparation {i}")
                continue

        print("❌ Toutes les stratégies de réparation ont échoué")
        return False

    def _reparation_dump_restore(self):
        """Stratégie 1 : Dump et restore complet"""
        import subprocess
        import os

        dump_file = f"{self.db_path}.dump"
        restored_file = f"{self.db_path}.restored"

        try:
            # Dump de la base
            with open(dump_file, 'w') as f:
                subprocess.run([
                    'sqlite3', self.db_path, '.dump'
                ], stdout=f, check=True)

            # Restore dans un nouveau fichier
            with open(dump_file, 'r') as f:
                subprocess.run([
                    'sqlite3', restored_file
                ], stdin=f, check=True)

            # Vérifier l'intégrité du fichier restauré
            with sqlite3.connect(restored_file) as conn:
                result = conn.execute("PRAGMA integrity_check").fetchone()
                if result[0] == "ok":
                    # Remplacer l'original
                    os.replace(restored_file, self.db_path)
                    os.remove(dump_file)
                    return True

            return False

        except (subprocess.CalledProcessError, sqlite3.Error) as e:
            self.logger.log_error(e, "Dump/Restore")
            return False
        finally:
            # Nettoyer les fichiers temporaires
            for temp_file in [dump_file, restored_file]:
                if os.path.exists(temp_file):
                    try:
                        os.remove(temp_file)
                    except:
                        pass

    def _reparation_vacuum(self):
        """Stratégie 2 : VACUUM pour réparer la base"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                conn.execute("VACUUM")

                # Vérifier après VACUUM
                result = conn.execute("PRAGMA integrity_check").fetchone()
                return result[0] == "ok"

        except sqlite3.Error as e:
            self.logger.log_error(e, "VACUUM")
            return False

    def _reparation_reindex(self):
        """Stratégie 3 : REINDEX pour réparer les index"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                conn.execute("REINDEX")

                # Vérifier après REINDEX
                result = conn.execute("PRAGMA integrity_check").fetchone()
                return result[0] == "ok"

        except sqlite3.Error as e:
            self.logger.log_error(e, "REINDEX")
            return False

    def restaurer_depuis_backup(self):
        """Restaure depuis le backup si disponible"""
        if not os.path.exists(self.backup_path):
            print(f"❌ Backup non trouvé : {self.backup_path}")
            return False

        try:
            import shutil

            # Créer backup d'urgence de l'état actuel
            backup_urgence = self.creer_backup_urgence()

            # Restaurer depuis le backup
            shutil.copy2(self.backup_path, self.db_path)

            # Vérifier l'intégrité
            with sqlite3.connect(self.db_path) as conn:
                result = conn.execute("PRAGMA integrity_check").fetchone()
                if result[0] == "ok":
                    print(f"✅ Restauration réussie depuis {self.backup_path}")
                    return True
                else:
                    print(f"❌ Backup corrompu : {self.backup_path}")
                    return False

        except Exception as e:
            self.logger.log_error(e, "Restauration backup")
            return False

    def recuperation_complete(self):
        """Processus de récupération complet"""
        print("🚨 Début de la procédure de récupération d'urgence")

        # Étape 1 : Créer un backup d'urgence
        backup_urgence = self.creer_backup_urgence()
        if backup_urgence:
            print(f"✅ Backup d'urgence créé : {backup_urgence}")

        # Étape 2 : Tenter la réparation
        if self.tenter_reparation():
            print("✅ Base de données réparée avec succès")
            return True

        # Étape 3 : Restaurer depuis backup si réparation échoue
        print("⚠️ Réparation impossible, tentative de restauration depuis backup")
        if self.restaurer_depuis_backup():
            print("✅ Restauration depuis backup réussie")
            return True

        # Étape 4 : Échec complet
        print("❌ Récupération impossible")
        print(f"   • Backup d'urgence disponible : {backup_urgence}")
        print(f"   • Vérifiez les logs pour plus de détails")
        return False

# Test du système de récupération
def demo_recuperation():
    """Démonstration du système de récupération"""

    # Créer une base de test
    test_db = "test_corruption.db"

    with sqlite3.connect(test_db) as conn:
        conn.execute("""
            CREATE TABLE test_data (
                id INTEGER PRIMARY KEY,
                data TEXT,
                value INTEGER
            )
        """)

        for i in range(100):
            conn.execute("INSERT INTO test_data (data, value) VALUES (?, ?)",
                        (f"Data {i}", i * 10))

        conn.commit()

    # Créer un backup valide
    import shutil
    shutil.copy2(test_db, f"{test_db}.backup")

    # Simuler une corruption (attention : destructif !)
    print("⚠️ Simulation d'une corruption de base...")
    with open(test_db, 'r+b') as f:
        f.seek(100)  # Aller à la position 100
        f.write(b'\x00' * 50)  # Écrire des zéros (corruption)

    # Tenter la récupération
    recovery = RecuperationAutomatique(test_db)

    try:
        # Vérifier que la base est effectivement corrompue
        with sqlite3.connect(test_db) as conn:
            conn.execute("PRAGMA integrity_check")
        print("✅ Base intègre (pas de corruption détectée)")
    except sqlite3.Error as e:
        print(f"❌ Corruption confirmée : {e}")

        # Lancer la récupération
        if recovery.recuperation_complete():
            print("🎉 Récupération réussie !")

            # Vérifier que la base fonctionne
            with sqlite3.connect(test_db) as conn:
                count = conn.execute("SELECT COUNT(*) FROM test_data").fetchone()[0]
                print(f"✅ {count} enregistrements récupérés")
        else:
            print("💥 Récupération échouée")

    # Nettoyer
    import os
    for file in [test_db, f"{test_db}.backup"]:
        if os.path.exists(file):
            os.remove(file)

demo_recuperation()
```

## Monitoring en temps réel des erreurs

### 1. Système d'alerte en temps réel

```python
import queue
import threading
from datetime import datetime, timedelta
from collections import defaultdict, deque

class MonitoringErreurs:
    """Système de monitoring en temps réel des erreurs SQLite"""

    def __init__(self, seuil_alerte=5, fenetre_minutes=10):
        self.seuil_alerte = seuil_alerte
        self.fenetre_minutes = fenetre_minutes
        self.erreurs_recentes = deque()
        self.stats_erreurs = defaultdict(int)
        self.alertes_actives = set()
        self.queue_erreurs = queue.Queue()
        self.actif = True

        # Démarrer le thread de monitoring
        self.thread_monitoring = threading.Thread(target=self._monitorer_erreurs, daemon=True)
        self.thread_monitoring.start()

    def enregistrer_erreur(self, erreur, contexte=None):
        """Enregistre une nouvelle erreur"""
        erreur_info = {
            'timestamp': datetime.now(),
            'type': type(erreur).__name__,
            'message': str(erreur),
            'contexte': contexte
        }

        self.queue_erreurs.put(erreur_info)

    def _monitorer_erreurs(self):
        """Thread de monitoring des erreurs"""
        while self.actif:
            try:
                # Attendre une nouvelle erreur (timeout de 1 seconde)
                erreur_info = self.queue_erreurs.get(timeout=1.0)

                # Ajouter à la liste des erreurs récentes
                self.erreurs_recentes.append(erreur_info)
                self.stats_erreurs[erreur_info['type']] += 1

                # Nettoyer les erreurs anciennes
                self._nettoyer_erreurs_anciennes()

                # Vérifier les seuils d'alerte
                self._verifier_alertes()

            except queue.Empty:
                # Pas de nouvelle erreur, nettoyer quand même
                self._nettoyer_erreurs_anciennes()
                continue
            except Exception as e:
                print(f"❌ Erreur dans le monitoring : {e}")

    def _nettoyer_erreurs_anciennes(self):
        """Supprime les erreurs plus anciennes que la fenêtre de monitoring"""
        limite_temps = datetime.now() - timedelta(minutes=self.fenetre_minutes)

        while self.erreurs_recentes and self.erreurs_recentes[0]['timestamp'] < limite_temps:
            erreur_ancienne = self.erreurs_recentes.popleft()
            self.stats_erreurs[erreur_ancienne['type']] -= 1

            # Supprimer les stats à zéro
            if self.stats_erreurs[erreur_ancienne['type']] <= 0:
                del self.stats_erreurs[erreur_ancienne['type']]

    def _verifier_alertes(self):
        """Vérifie si des seuils d'alerte sont dépassés"""
        nb_erreurs_recentes = len(self.erreurs_recentes)

        # Alerte générale si trop d'erreurs
        if nb_erreurs_recentes >= self.seuil_alerte:
            alerte_id = "trop_erreurs_generales"
            if alerte_id not in self.alertes_actives:
                self.alertes_actives.add(alerte_id)
                self._envoyer_alerte(
                    "Trop d'erreurs SQLite",
                    f"{nb_erreurs_recentes} erreurs en {self.fenetre_minutes} minutes"
                )

        # Alertes par type d'erreur
        for type_erreur, count in self.stats_erreurs.items():
            if count >= self.seuil_alerte // 2:  # Seuil plus bas pour erreurs spécifiques
                alerte_id = f"trop_erreurs_{type_erreur}"
                if alerte_id not in self.alertes_actives:
                    self.alertes_actives.add(alerte_id)
                    self._envoyer_alerte(
                        f"Trop d'erreurs {type_erreur}",
                        f"{count} erreurs de type {type_erreur} en {self.fenetre_minutes} minutes"
                    )

    def _envoyer_alerte(self, titre, message):
        """Envoie une alerte (à personnaliser selon vos besoins)"""
        print(f"🚨 ALERTE : {titre}")
        print(f"   Message : {message}")
        print(f"   Timestamp : {datetime.now()}")

        # Ici vous pourriez ajouter :
        # - Envoi d'email
        # - Notification Slack
        # - Log dans un système de monitoring
        # - etc.

    def obtenir_statistiques(self):
        """Retourne les statistiques actuelles"""
        return {
            'erreurs_recentes': len(self.erreurs_recentes),
            'types_erreurs': dict(self.stats_erreurs),
            'alertes_actives': list(self.alertes_actives),
            'fenetre_minutes': self.fenetre_minutes
        }

    def arreter(self):
        """Arrête le monitoring"""
        self.actif = False
        if self.thread_monitoring.is_alive():
            self.thread_monitoring.join()

# Utilisation du monitoring
def demo_monitoring():
    """Démonstration du système de monitoring"""

    monitoring = MonitoringErreurs(seuil_alerte=3, fenetre_minutes=1)

    # Simuler des erreurs
    erreurs_test = [
        (sqlite3.IntegrityError("UNIQUE constraint failed"), "Test 1"),
        (sqlite3.OperationalError("database is locked"), "Test 2"),
        (sqlite3.IntegrityError("UNIQUE constraint failed"), "Test 3"),
        (sqlite3.ProgrammingError("syntax error"), "Test 4"),
        (sqlite3.IntegrityError("UNIQUE constraint failed"), "Test 5"),
    ]

    print("📊 Simulation d'erreurs pour déclencher des alertes...")

    for erreur, contexte in erreurs_test:
        monitoring.enregistrer_erreur(erreur, contexte)
        time.sleep(0.5)  # Attendre un peu entre les erreurs

        # Afficher les stats
        stats = monitoring.obtenir_statistiques()
        print(f"   Erreurs récentes : {stats['erreurs_recentes']}")
        if stats['types_erreurs']:
            print(f"   Types : {stats['types_erreurs']}")

    # Attendre un peu pour voir les alertes
    time.sleep(2)

    print("\n📋 Statistiques finales :")
    stats_finales = monitoring.obtenir_statistiques()
    for key, value in stats_finales.items():
        print(f"   {key}: {value}")

    monitoring.arreter()

demo_monitoring()
```

### 2. Wrapper de connexion avec monitoring

```python
class ConnexionSQLiteMonitoree:
    """Wrapper de connexion SQLite avec monitoring intégré"""

    def __init__(self, db_path, monitoring=None, **kwargs):
        self.db_path = db_path
        self.monitoring = monitoring
        self.conn_kwargs = kwargs
        self.conn = None
        self.logger = SQLiteLogger(f"conn_{id(self)}")

    def connect(self):
        """Établit la connexion avec monitoring"""
        try:
            start_time = time.time()
            self.conn = sqlite3.connect(self.db_path, **self.conn_kwargs)

            duration = time.time() - start_time
            self.logger.log_performance("CONNECT", duration)

            return self.conn

        except sqlite3.Error as e:
            if self.monitoring:
                self.monitoring.enregistrer_erreur(e, f"Connexion à {self.db_path}")
            self.logger.log_error(e, "Connexion")
            raise

    def execute(self, sql, params=None):
        """Exécute une requête avec monitoring"""
        if not self.conn:
            raise sqlite3.ProgrammingError("Pas de connexion active")

        start_time = time.time()

        try:
            if params:
                cursor = self.conn.execute(sql, params)
            else:
                cursor = self.conn.execute(sql)

            duration = time.time() - start_time
            rows_affected = cursor.rowcount if cursor.rowcount >= 0 else None

            self.logger.log_performance("EXECUTE", duration, rows_affected)

            return cursor

        except sqlite3.Error as e:
            if self.monitoring:
                self.monitoring.enregistrer_erreur(e, f"Exécution SQL: {sql[:50]}...")

            self.logger.log_error(e, "Exécution SQL", sql, params)
            raise

    def executemany(self, sql, params_list):
        """Exécute plusieurs requêtes avec monitoring"""
        if not self.conn:
            raise sqlite3.ProgrammingError("Pas de connexion active")

        start_time = time.time()

        try:
            cursor = self.conn.executemany(sql, params_list)

            duration = time.time() - start_time
            rows_affected = len(params_list)

            self.logger.log_performance("EXECUTEMANY", duration, rows_affected)

            return cursor

        except sqlite3.Error as e:
            if self.monitoring:
                self.monitoring.enregistrer_erreur(e, f"Exécution multiple: {sql[:50]}...")

            self.logger.log_error(e, "Exécution multiple", sql, f"{len(params_list)} lots")
            raise

    def commit(self):
        """Commit avec monitoring"""
        if not self.conn:
            raise sqlite3.ProgrammingError("Pas de connexion active")

        start_time = time.time()

        try:
            self.conn.commit()
            duration = time.time() - start_time
            self.logger.log_performance("COMMIT", duration)

        except sqlite3.Error as e:
            if self.monitoring:
                self.monitoring.enregistrer_erreur(e, "Commit")
            self.logger.log_error(e, "Commit")
            raise

    def rollback(self):
        """Rollback avec monitoring"""
        if not self.conn:
            return

        try:
            self.conn.rollback()
            self.logger.logger.info("Rollback effectué")

        except sqlite3.Error as e:
            if self.monitoring:
                self.monitoring.enregistrer_erreur(e, "Rollback")
            self.logger.log_error(e, "Rollback")

    def close(self):
        """Ferme la connexion"""
        if self.conn:
            try:
                self.conn.close()
                self.logger.logger.info("Connexion fermée")
            except sqlite3.Error as e:
                self.logger.log_error(e, "Fermeture connexion")
            finally:
                self.conn = None

    def __enter__(self):
        """Support du context manager"""
        self.connect()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Sortie du context manager"""
        if exc_type is None:
            try:
                self.commit()
            except sqlite3.Error:
                self.rollback()
                raise
        else:
            self.rollback()

        self.close()
        return False

# Démonstration de la connexion monitorée
def demo_connexion_monitoree():
    """Test de la connexion avec monitoring"""

    monitoring = MonitoringErreurs(seuil_alerte=2, fenetre_minutes=1)

    try:
        # Utilisation normale
        with ConnexionSQLiteMonitoree(':memory:', monitoring=monitoring) as conn:
            conn.execute("""
                CREATE TABLE test (
                    id INTEGER PRIMARY KEY,
                    name TEXT UNIQUE,
                    value INTEGER
                )
            """)

            # Insertions normales
            conn.execute("INSERT INTO test (name, value) VALUES (?, ?)", ("Alice", 100))
            conn.execute("INSERT INTO test (name, value) VALUES (?, ?)", ("Bob", 200))

            print("✅ Opérations normales terminées")

        # Provoquer des erreurs pour tester le monitoring
        with ConnexionSQLiteMonitoree(':memory:', monitoring=monitoring) as conn:
            conn.execute("CREATE TABLE test (id INTEGER PRIMARY KEY, name TEXT UNIQUE)")

            # Erreurs de contrainte
            try:
                conn.execute("INSERT INTO test (name) VALUES (?)", ("Alice",))
                conn.execute("INSERT INTO test (name) VALUES (?)", ("Alice",))  # Doublon
            except sqlite3.IntegrityError:
                pass

            # Erreur de syntaxe
            try:
                conn.execute("INVALID SQL STATEMENT")
            except sqlite3.Error:
                pass

        # Afficher les statistiques
        print("\n📊 Statistiques de monitoring :")
        stats = monitoring.obtenir_statistiques()
        for key, value in stats.items():
            print(f"   {key}: {value}")

    finally:
        monitoring.arreter()

demo_connexion_monitoree()
```

## Patterns de gestion d'erreurs pour applications

### 1. Pattern Repository avec gestion d'erreurs

```python
from abc import ABC, abstractmethod
from typing import Optional, List, Tuple

class RepositoryError(Exception):
    """Exception personnalisée pour les erreurs de repository"""
    pass

class ValidationError(RepositoryError):
    """Erreur de validation des données"""
    pass

class NotFoundError(RepositoryError):
    """Entité non trouvée"""
    pass

class ConflictError(RepositoryError):
    """Conflit de données (doublon, etc.)"""
    pass

class BaseRepository(ABC):
    """Repository de base avec gestion d'erreurs"""

    def __init__(self, db_path, monitoring=None):
        self.db_path = db_path
        self.monitoring = monitoring
        self.logger = SQLiteLogger(self.__class__.__name__)

    def _get_connection(self):
        """Obtient une connexion avec gestion d'erreurs"""
        return ConnexionSQLiteMonitoree(self.db_path, self.monitoring)

    def _handle_sqlite_error(self, error, operation, **context):
        """Convertit les erreurs SQLite en erreurs métier"""
        if self.monitoring:
            self.monitoring.enregistrer_erreur(error, f"{operation} - {self.__class__.__name__}")

        self.logger.log_error(error, operation, **context)

        if isinstance(error, sqlite3.IntegrityError):
            if "UNIQUE constraint failed" in str(error):
                raise ConflictError(f"Doublon détecté lors de {operation}")
            elif "NOT NULL constraint failed" in str(error):
                raise ValidationError(f"Champ requis manquant lors de {operation}")
            elif "CHECK constraint failed" in str(error):
                raise ValidationError(f"Validation métier échouée lors de {operation}")
            else:
                raise ConflictError(f"Violation d'intégrité lors de {operation}: {error}")

        elif isinstance(error, sqlite3.OperationalError):
            if "no such table" in str(error):
                raise RepositoryError(f"Table manquante lors de {operation}")
            elif "database is locked" in str(error):
                raise RepositoryError(f"Base de données verrouillée lors de {operation}")
            else:
                raise RepositoryError(f"Erreur opérationnelle lors de {operation}: {error}")

        elif isinstance(error, sqlite3.ProgrammingError):
            raise RepositoryError(f"Erreur de programmation lors de {operation}: {error}")

        else:
            raise RepositoryError(f"Erreur inattendue lors de {operation}: {error}")

class UtilisateurRepository(BaseRepository):
    """Repository pour les utilisateurs avec gestion d'erreurs complète"""

    def initialiser_schema(self):
        """Initialise le schéma de base"""
        try:
            with self._get_connection() as conn:
                conn.execute("""
                    CREATE TABLE IF NOT EXISTS utilisateurs (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        email TEXT UNIQUE NOT NULL,
                        nom TEXT NOT NULL,
                        age INTEGER CHECK (age >= 0 AND age <= 150),
                        date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
                        actif BOOLEAN DEFAULT 1
                    )
                """)

        except sqlite3.Error as e:
            self._handle_sqlite_error(e, "initialisation_schema")

    def creer_utilisateur(self, email: str, nom: str, age: int) -> int:
        """Crée un nouvel utilisateur"""
        # Validation métier
        if not email or '@' not in email:
            raise ValidationError("Email invalide")

        if not nom or len(nom.strip()) < 2:
            raise ValidationError("Nom trop court")

        if age < 0 or age > 150:
            raise ValidationError("Âge invalide")

        try:
            with self._get_connection() as conn:
                cursor = conn.execute("""
                    INSERT INTO utilisateurs (email, nom, age)
                    VALUES (?, ?, ?)
                """, (email.strip().lower(), nom.strip(), age))

                return cursor.lastrowid

        except sqlite3.Error as e:
            self._handle_sqlite_error(e, "creation_utilisateur",
                                    email=email, nom=nom, age=age)

    def obtenir_utilisateur(self, user_id: int) -> Optional[dict]:
        """Récupère un utilisateur par son ID"""
        try:
            with self._get_connection() as conn:
                cursor = conn.execute("""
                    SELECT id, email, nom, age, date_creation, actif
                    FROM utilisateurs
                    WHERE id = ?
                """, (user_id,))

                row = cursor.fetchone()
                if row is None:
                    raise NotFoundError(f"Utilisateur {user_id} non trouvé")

                return {
                    'id': row[0],
                    'email': row[1],
                    'nom': row[2],
                    'age': row[3],
                    'date_creation': row[4],
                    'actif': bool(row[5])
                }

        except sqlite3.Error as e:
            self._handle_sqlite_error(e, "lecture_utilisateur", user_id=user_id)

    def mettre_a_jour_utilisateur(self, user_id: int, **champs) -> bool:
        """Met à jour un utilisateur"""
        if not champs:
            raise ValidationError("Aucun champ à mettre à jour")

        # Validation des champs
        champs_autorises = {'email', 'nom', 'age', 'actif'}
        champs_invalides = set(champs.keys()) - champs_autorises
        if champs_invalides:
            raise ValidationError(f"Champs non autorisés : {champs_invalides}")

        try:
            with self._get_connection() as conn:
                # Vérifier que l'utilisateur existe
                exists = conn.execute("SELECT 1 FROM utilisateurs WHERE id = ?", (user_id,)).fetchone()
                if not exists:
                    raise NotFoundError(f"Utilisateur {user_id} non trouvé")

                # Construire la requête de mise à jour
                set_clauses = []
                params = []

                for champ, valeur in champs.items():
                    set_clauses.append(f"{champ} = ?")
                    params.append(valeur)

                params.append(user_id)

                sql = f"UPDATE utilisateurs SET {', '.join(set_clauses)} WHERE id = ?"
                cursor = conn.execute(sql, params)

                return cursor.rowcount > 0

        except sqlite3.Error as e:
            self._handle_sqlite_error(e, "mise_a_jour_utilisateur",
                                    user_id=user_id, champs=champs)

    def lister_utilisateurs(self, actifs_seulement: bool = True,
                          limite: int = 100, offset: int = 0) -> List[dict]:
        """Liste les utilisateurs avec pagination"""
        try:
            with self._get_connection() as conn:
                sql = """
                    SELECT id, email, nom, age, date_creation, actif
                    FROM utilisateurs
                """
                params = []

                if actifs_seulement:
                    sql += " WHERE actif = 1"

                sql += " ORDER BY date_creation DESC LIMIT ? OFFSET ?"
                params.extend([limite, offset])

                cursor = conn.execute(sql, params)

                utilisateurs = []
                for row in cursor:
                    utilisateurs.append({
                        'id': row[0],
                        'email': row[1],
                        'nom': row[2],
                        'age': row[3],
                        'date_creation': row[4],
                        'actif': bool(row[5])
                    })

                return utilisateurs

        except sqlite3.Error as e:
            self._handle_sqlite_error(e, "liste_utilisateurs",
                                    actifs_seulement=actifs_seulement,
                                    limite=limite, offset=offset)

    def supprimer_utilisateur(self, user_id: int, suppression_logique: bool = True) -> bool:
        """Supprime un utilisateur (logique ou physique)"""
        try:
            with self._get_connection() as conn:
                if suppression_logique:
                    # Suppression logique (marquer comme inactif)
                    cursor = conn.execute("""
                        UPDATE utilisateurs SET actif = 0
                        WHERE id = ? AND actif = 1
                    """, (user_id,))
                else:
                    # Suppression physique
                    cursor = conn.execute("DELETE FROM utilisateurs WHERE id = ?", (user_id,))

                if cursor.rowcount == 0:
                    raise NotFoundError(f"Utilisateur {user_id} non trouvé ou déjà supprimé")

                return True

        except sqlite3.Error as e:
            self._handle_sqlite_error(e, "suppression_utilisateur",
                                    user_id=user_id,
                                    suppression_logique=suppression_logique)

# Utilisation du Repository avec gestion d'erreurs
def demo_repository_avec_erreurs():
    """Démonstration du pattern Repository avec gestion d'erreurs"""

    monitoring = MonitoringErreurs(seuil_alerte=3, fenetre_minutes=2)
    repo = UtilisateurRepository(':memory:', monitoring)

    try:
        # Initialiser le schéma
        repo.initialiser_schema()
        print("✅ Schéma initialisé")

        # Créer des utilisateurs
        try:
            user1_id = repo.creer_utilisateur("alice@email.com", "Alice Dupont", 25)
            print(f"✅ Utilisateur créé avec ID: {user1_id}")
        except ValidationError as e:
            print(f"❌ Erreur de validation : {e}")
        except ConflictError as e:
            print(f"❌ Conflit : {e}")

        # Tenter de créer un doublon
        try:
            repo.creer_utilisateur("alice@email.com", "Alice Martin", 30)
        except ConflictError as e:
            print(f"❌ Doublon détecté comme attendu : {e}")

        # Validation d'entrée invalide
        try:
            repo.creer_utilisateur("email_invalide", "Bob", 25)
        except ValidationError as e:
            print(f"❌ Validation échouée comme attendu : {e}")

        # Lire un utilisateur
        try:
            utilisateur = repo.obtenir_utilisateur(user1_id)
            print(f"✅ Utilisateur récupéré : {utilisateur['nom']}")
        except NotFoundError as e:
            print(f"❌ Utilisateur non trouvé : {e}")

        # Tenter de lire un utilisateur inexistant
        try:
            repo.obtenir_utilisateur(999)
        except NotFoundError as e:
            print(f"❌ Utilisateur inexistant comme attendu : {e}")

        # Mettre à jour
        try:
            repo.mettre_a_jour_utilisateur(user1_id, age=26, nom="Alice Durand")
            print("✅ Utilisateur mis à jour")
        except (ValidationError, NotFoundError) as e:
            print(f"❌ Erreur de mise à jour : {e}")

        # Lister les utilisateurs
        utilisateurs = repo.lister_utilisateurs()
        print(f"✅ {len(utilisateurs)} utilisateur(s) trouvé(s)")

        # Afficher les statistiques de monitoring
        print("\n📊 Statistiques d'erreurs :")
        stats = monitoring.obtenir_statistiques()
        for key, value in stats.items():
            print(f"   {key}: {value}")

    except RepositoryError as e:
        print(f"❌ Erreur de repository : {e}")

    finally:
        monitoring.arreter()

demo_repository_avec_erreurs()
```

## Pattern Circuit Breaker pour SQLite

### Système de protection contre les pannes en cascade

```python
import time
from enum import Enum
from typing import Callable, Any

class CircuitBreakerState(Enum):
    CLOSED = "CLOSED"      # Fonctionnement normal
    OPEN = "OPEN"          # Circuit ouvert (erreurs)
    HALF_OPEN = "HALF_OPEN"  # Test de récupération

class CircuitBreakerSQLite:
    """Circuit Breaker pour protéger contre les pannes SQLite"""

    def __init__(self,
                 failure_threshold: int = 5,
                 recovery_timeout: int = 60,
                 expected_exception: type = sqlite3.Error):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception

        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitBreakerState.CLOSED

        self.logger = SQLiteLogger("circuit_breaker")

    def call(self, func: Callable, *args, **kwargs) -> Any:
        """Exécute une fonction avec protection Circuit Breaker"""

        if self.state == CircuitBreakerState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitBreakerState.HALF_OPEN
                self.logger.logger.info("Circuit Breaker: Tentative de récupération (HALF_OPEN)")
            else:
                raise Exception(f"Circuit Breaker OPEN - Service indisponible")

        try:
            result = func(*args, **kwargs)

            # Succès - réinitialiser le compteur
            if self.state == CircuitBreakerState.HALF_OPEN:
                self._reset()
                self.logger.logger.info("Circuit Breaker: Récupération réussie (CLOSED)")

            return result

        except self.expected_exception as e:
            self._record_failure()
            self.logger.log_error(e, "Circuit Breaker - Échec")
            raise

    def _should_attempt_reset(self) -> bool:
        """Vérifie si on peut tenter une récupération"""
        if self.last_failure_time is None:
            return True

        return time.time() - self.last_failure_time >= self.recovery_timeout

    def _record_failure(self):
        """Enregistre un échec"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitBreakerState.OPEN
            self.logger.logger.warning(f"Circuit Breaker OUVERT après {self.failure_count} échecs")

    def _reset(self):
        """Remet le circuit en état normal"""
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitBreakerState.CLOSED

    def get_state(self) -> dict:
        """Retourne l'état actuel du circuit breaker"""
        return {
            'state': self.state.value,
            'failure_count': self.failure_count,
            'last_failure_time': self.last_failure_time,
            'failure_threshold': self.failure_threshold
        }

class ServiceSQLiteProtege:
    """Service SQLite protégé par un Circuit Breaker"""

    def __init__(self, db_path):
        self.db_path = db_path
        self.circuit_breaker = CircuitBreakerSQLite(
            failure_threshold=3,
            recovery_timeout=30
        )

    def _execute_query(self, sql, params=None):
        """Exécute une requête SQLite"""
        with sqlite3.connect(self.db_path) as conn:
            if params:
                return conn.execute(sql, params).fetchall()
            else:
                return conn.execute(sql).fetchall()

    def obtenir_donnees(self, table: str, condition: str = None):
        """Méthode publique protégée par le Circuit Breaker"""
        def query():
            sql = f"SELECT * FROM {table}"
            if condition:
                sql += f" WHERE {condition}"
            return self._execute_query(sql)

        return self.circuit_breaker.call(query)

    def inserer_donnees(self, table: str, data: dict):
        """Insertion protégée"""
        def insert():
            columns = ', '.join(data.keys())
            placeholders = ', '.join(['?' for _ in data])
            sql = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
            return self._execute_query(sql, list(data.values()))

        return self.circuit_breaker.call(insert)

    def get_circuit_state(self):
        """Retourne l'état du circuit breaker"""
        return self.circuit_breaker.get_state()

def demo_circuit_breaker():
    """Démonstration du Circuit Breaker"""

    # Créer une base de test
    service = ServiceSQLiteProtege(':memory:')

    # Créer une table pour les tests
    with sqlite3.connect(':memory:') as conn:
        conn.execute("CREATE TABLE test (id INTEGER PRIMARY KEY, name TEXT)")
        conn.execute("INSERT INTO test (name) VALUES ('Test')")

    print("🔄 Test du Circuit Breaker SQLite")
    print("=" * 50)

    # Test normal (circuit fermé)
    try:
        result = service.obtenir_donnees('test')
        print(f"✅ Requête normale réussie : {len(result)} résultat(s)")
        print(f"   État circuit : {service.get_circuit_state()['state']}")
    except Exception as e:
        print(f"❌ Erreur : {e}")

    # Simuler des erreurs pour ouvrir le circuit
    print("\n🚨 Simulation d'erreurs pour déclencher le Circuit Breaker...")

    for i in range(5):
        try:
            # Requête sur table inexistante pour provoquer une erreur
            service.obtenir_donnees('table_inexistante')
        except Exception as e:
            state = service.get_circuit_state()
            print(f"❌ Erreur {i+1}: Circuit {state['state']} (échecs: {state['failure_count']})")

    # Tenter d'utiliser le service avec circuit ouvert
    print("\n⛔ Test avec circuit ouvert...")
    try:
        service.obtenir_donnees('test')
    except Exception as e:
        print(f"❌ Service bloqué par Circuit Breaker : {e}")

    print(f"\n📊 État final du circuit : {service.get_circuit_state()}")

demo_circuit_breaker()
```

## Système de fallback et récupération

### Stratégie de fallback en cas d'erreur

```python
from typing import List, Callable, Any, Optional
import pickle
import json

class FallbackManager:
    """Gestionnaire de stratégies de fallback"""

    def __init__(self):
        self.fallback_strategies = []
        self.cache_strategies = {}
        self.logger = SQLiteLogger("fallback")

    def ajouter_strategie(self, nom: str, fonction: Callable, priorite: int = 1):
        """Ajoute une stratégie de fallback"""
        self.fallback_strategies.append({
            'nom': nom,
            'fonction': fonction,
            'priorite': priorite
        })

        # Trier par priorité (plus élevée = plus prioritaire)
        self.fallback_strategies.sort(key=lambda x: x['priorite'], reverse=True)

    def executer_avec_fallback(self, operation_principale: Callable,
                              *args, **kwargs) -> Any:
        """Exécute une opération avec fallback automatique"""

        # Tenter l'opération principale
        try:
            result = operation_principale(*args, **kwargs)
            self.logger.logger.info("Opération principale réussie")
            return result

        except Exception as e_principale:
            self.logger.log_error(e_principale, "Opération principale")

            # Tenter les stratégies de fallback
            for strategie in self.fallback_strategies:
                try:
                    self.logger.logger.info(f"Tentative fallback : {strategie['nom']}")
                    result = strategie['fonction'](*args, **kwargs)

                    self.logger.logger.warning(
                        f"Fallback réussi avec stratégie : {strategie['nom']}"
                    )
                    return result

                except Exception as e_fallback:
                    self.logger.log_error(
                        e_fallback,
                        f"Fallback {strategie['nom']}"
                    )
                    continue

            # Toutes les stratégies ont échoué
            self.logger.logger.error("Toutes les stratégies de fallback ont échoué")
            raise e_principale

class ServiceSQLiteAvecFallback:
    """Service SQLite avec système de fallback complet"""

    def __init__(self, db_path_principal, db_path_backup=None, cache_file="cache.json"):
        self.db_path_principal = db_path_principal
        self.db_path_backup = db_path_backup
        self.cache_file = cache_file
        self.fallback_manager = FallbackManager()

        self._configurer_fallbacks()

    def _configurer_fallbacks(self):
        """Configure les stratégies de fallback"""

        # Stratégie 1 : Base de backup (priorité haute)
        if self.db_path_backup:
            self.fallback_manager.ajouter_strategie(
                "base_backup",
                self._lire_depuis_backup,
                priorite=3
            )

        # Stratégie 2 : Cache fichier (priorité moyenne)
        self.fallback_manager.ajouter_strategie(
            "cache_fichier",
            self._lire_depuis_cache,
            priorite=2
        )

        # Stratégie 3 : Données par défaut (priorité basse)
        self.fallback_manager.ajouter_strategie(
            "donnees_defaut",
            self._donnees_par_defaut,
            priorite=1
        )

    def _lire_depuis_principal(self, table: str, condition: str = None):
        """Lecture depuis la base principale"""
        with sqlite3.connect(self.db_path_principal) as conn:
            sql = f"SELECT * FROM {table}"
            if condition:
                sql += f" WHERE {condition}"
            return conn.execute(sql).fetchall()

    def _lire_depuis_backup(self, table: str, condition: str = None):
        """Lecture depuis la base de backup"""
        if not self.db_path_backup:
            raise Exception("Pas de base de backup configurée")

        with sqlite3.connect(self.db_path_backup) as conn:
            sql = f"SELECT * FROM {table}"
            if condition:
                sql += f" WHERE {condition}"
            return conn.execute(sql).fetchall()

    def _lire_depuis_cache(self, table: str, condition: str = None):
        """Lecture depuis le cache fichier"""
        try:
            with open(self.cache_file, 'r') as f:
                cache_data = json.load(f)

            cache_key = f"{table}_{condition or 'all'}"
            if cache_key in cache_data:
                return cache_data[cache_key]
            else:
                raise Exception(f"Données non trouvées dans le cache : {cache_key}")

        except (FileNotFoundError, json.JSONDecodeError, KeyError):
            raise Exception("Cache indisponible ou corrompu")

    def _donnees_par_defaut(self, table: str, condition: str = None):
        """Retourne des données par défaut"""
        defaults = {
            'utilisateurs': [('default_user', 'user@default.com', 0)],
            'config': [('maintenance_mode', 'true')],
            'status': [('system_status', 'degraded')]
        }

        return defaults.get(table, [])

    def _sauvegarder_en_cache(self, table: str, condition: str, data: list):
        """Sauvegarde les données en cache"""
        try:
            cache_data = {}
            try:
                with open(self.cache_file, 'r') as f:
                    cache_data = json.load(f)
            except (FileNotFoundError, json.JSONDecodeError):
                pass

            cache_key = f"{table}_{condition or 'all'}"
            cache_data[cache_key] = data

            with open(self.cache_file, 'w') as f:
                json.dump(cache_data, f)

        except Exception as e:
            print(f"⚠️ Impossible de sauvegarder en cache : {e}")

    def obtenir_donnees(self, table: str, condition: str = None):
        """Méthode publique avec fallback automatique"""

        def operation_principale():
            result = self._lire_depuis_principal(table, condition)
            # Sauvegarder en cache si succès
            self._sauvegarder_en_cache(table, condition, result)
            return result

        return self.fallback_manager.executer_avec_fallback(
            operation_principale, table, condition
        )

def demo_fallback_system():
    """Démonstration du système de fallback"""

    # Créer des bases de test
    print("🏗️ Création des bases de test...")

    # Base principale
    with sqlite3.connect('principal.db') as conn:
        conn.execute("CREATE TABLE utilisateurs (id INTEGER, nom TEXT, email TEXT)")
        conn.execute("INSERT INTO utilisateurs VALUES (1, 'Alice', 'alice@test.com')")
        conn.execute("INSERT INTO utilisateurs VALUES (2, 'Bob', 'bob@test.com')")

    # Base de backup
    with sqlite3.connect('backup.db') as conn:
        conn.execute("CREATE TABLE utilisateurs (id INTEGER, nom TEXT, email TEXT)")
        conn.execute("INSERT INTO utilisateurs VALUES (1, 'Alice Backup', 'alice@backup.com')")
        conn.execute("INSERT INTO utilisateurs VALUES (3, 'Charlie', 'charlie@backup.com')")

    service = ServiceSQLiteAvecFallback('principal.db', 'backup.db')

    print("\n📖 Test 1 : Lecture normale (base principale)")
    try:
        result = service.obtenir_donnees('utilisateurs')
        print(f"✅ Données obtenues : {len(result)} utilisateur(s)")
        for row in result:
            print(f"   • {row[1]} ({row[2]})")
    except Exception as e:
        print(f"❌ Erreur : {e}")

    # Simuler une panne de la base principale
    print("\n💥 Simulation de panne (suppression base principale)")
    import os
    os.remove('principal.db')

    print("\n📖 Test 2 : Lecture avec base principale indisponible")
    try:
        result = service.obtenir_donnees('utilisateurs')
        print(f"✅ Fallback réussi : {len(result)} utilisateur(s)")
        for row in result:
            print(f"   • {row[1]} ({row[2]})")
    except Exception as e:
        print(f"❌ Erreur : {e}")

    # Simuler panne totale
    print("\n💥💥 Simulation de panne totale (suppression backup)")
    os.remove('backup.db')

    print("\n📖 Test 3 : Lecture avec toutes les bases indisponibles")
    try:
        result = service.obtenir_donnees('utilisateurs')
        print(f"✅ Fallback vers données par défaut : {len(result)} utilisateur(s)")
        for row in result:
            print(f"   • {row}")
    except Exception as e:
        print(f"❌ Erreur finale : {e}")

    # Nettoyer
    for file in ['cache.json']:
        if os.path.exists(file):
            os.remove(file)

demo_fallback_system()
```

## Récapitulatif final et bonnes pratiques

### Checklist de gestion d'erreurs SQLite

```python
def checklist_gestion_erreurs():
    """Checklist complète pour la gestion d'erreurs SQLite"""

    checklist = {
        "🔍 Détection d'erreurs": [
            "✅ Utiliser des blocs try/except spécifiques",
            "✅ Capturer les différents types d'exceptions SQLite",
            "✅ Valider les données avant insertion",
            "✅ Vérifier l'existence des tables/colonnes",
            "✅ Contrôler les contraintes métier"
        ],

        "📝 Logging et diagnostic": [
            "✅ Logger toutes les erreurs avec contexte",
            "✅ Inclure les requêtes SQL dans les logs",
            "✅ Enregistrer les paramètres de requête",
            "✅ Tracer les performances",
            "✅ Monitorer les métriques en temps réel"
        ],

        "🔄 Récupération d'erreurs": [
            "✅ Implémenter des mécanismes de retry",
            "✅ Utiliser des backoff exponentiels",
            "✅ Gérer les transactions avec rollback",
            "✅ Avoir des stratégies de fallback",
            "✅ Maintenir des sauvegardes à jour"
        ],

        "🛡️ Protection préventive": [
            "✅ Utiliser des gestionnaires de contexte",
            "✅ Implémenter un Circuit Breaker",
            "✅ Valider les entrées utilisateur",
            "✅ Limiter les ressources (timeout, pool)",
            "✅ Monitorer la santé du système"
        ],

        "👥 Expérience utilisateur": [
            "✅ Messages d'erreur compréhensibles",
            "✅ Pas d'exposition d'informations techniques",
            "✅ Actions de récupération suggérées",
            "✅ Interface gracieuse en cas d'erreur",
            "✅ Feedback en temps réel"
        ],

        "🏗️ Architecture": [
            "✅ Séparation couches métier/données",
            "✅ Exceptions métier personnalisées",
            "✅ Pattern Repository pour abstraction",
            "✅ Services avec gestion d'état",
            "✅ Tests automatisés des cas d'erreur"
        ]
    }

    print("📋 CHECKLIST GESTION D'ERREURS SQLITE")
    print("=" * 60)

    for categorie, items in checklist.items():
        print(f"\n{categorie}")
        for item in items:
            print(f"  {item}")

    return checklist

checklist_gestion_erreurs()
```

### Template de classe service robuste

```python
class ServiceSQLiteRobuste:
    """Template de service SQLite avec gestion d'erreurs complète"""

    def __init__(self, db_path, **config):
        self.db_path = db_path
        self.config = config

        # Composants de gestion d'erreurs
        self.logger = SQLiteLogger(self.__class__.__name__)
        self.monitoring = MonitoringErreurs()
        self.circuit_breaker = CircuitBreakerSQLite()
        self.fallback_manager = FallbackManager()
        self.recovery = RecuperationAutomatique(db_path)

        self._configurer_service()

    def _configurer_service(self):
        """Configuration initiale du service"""
        try:
            self._verifier_base_sante()
            self._configurer_fallbacks()
            self.logger.logger.info("Service SQLite configuré avec succès")

        except Exception as e:
            self.logger.log_error(e, "Configuration service")
            raise

    def _verifier_base_sante(self):
        """Vérifie la santé de la base au démarrage"""
        try:
            with sqlite3.connect(self.db_path) as conn:
                conn.execute("PRAGMA integrity_check").fetchone()

        except sqlite3.Error as e:
            self.logger.log_error(e, "Vérification santé base")

            # Tenter une récupération automatique
            if self.recovery.recuperation_complete():
                self.logger.logger.info("Récupération automatique réussie")
            else:
                raise Exception("Base de données inutilisable")

    def _configurer_fallbacks(self):
        """Configure les stratégies de fallback"""
        # À implémenter selon les besoins spécifiques
        pass

    def executer_operation(self, operation: Callable, *args, **kwargs):
        """Exécute une opération avec toutes les protections"""

        def operation_protegee():
            return operation(*args, **kwargs)

        try:
            return self.circuit_breaker.call(operation_protegee)

        except Exception as e:
            self.monitoring.enregistrer_erreur(e, f"Opération {operation.__name__}")

            # Tenter fallback si configuré
            if self.fallback_manager.fallback_strategies:
                return self.fallback_manager.executer_avec_fallback(
                    operation_protegee, *args, **kwargs
                )
            else:
                raise

    def obtenir_diagnostics(self):
        """Retourne un diagnostic complet du service"""
        return {
            'circuit_breaker': self.circuit_breaker.get_state(),
            'monitoring': self.monitoring.obtenir_statistiques(),
            'derniere_verification': datetime.now().isoformat()
        }

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        try:
            self.monitoring.arreter()
        except:
            pass

        return False

print("\n🎉 RÉCAPITULATIF SECTION 6.5")
print("=" * 50)
print("""
Cette section vous a appris à :

✅ COMPRENDRE les types d'erreurs SQLite
   • Erreurs de syntaxe, contraintes, verrouillage, connexion
   • Hiérarchie des exceptions SQLite
   • Contextes d'apparition des erreurs

✅ IMPLÉMENTER une gestion d'erreurs robuste
   • Try/except spécialisés par type d'erreur
   • Gestionnaires de contexte personnalisés
   • Décorateurs pour retry automatique
   • Messages d'erreur utilisateur-friendly

✅ MONITORER et diagnostiquer
   • Logging structuré avec contexte
   • Monitoring en temps réel des erreurs
   • Métriques de performance et santé
   • Outils de diagnostic automatique

✅ RÉCUPÉRER des erreurs
   • Stratégies de réparation automatique
   • Systèmes de fallback multi-niveaux
   • Circuit Breaker pour protection
   • Récupération depuis sauvegardes

✅ ARCHITECTURER pour la robustesse
   • Pattern Repository avec exceptions métier
   • Services avec gestion d'état
   • Séparation des responsabilités
   • Tests des cas d'erreur

🏆 PROCHAINES ÉTAPES
Dans la section 6.6, nous explorerons la recherche plein texte
avec FTS5, la dernière pièce de la programmation avancée SQLite.
""")
```

## Points clés à retenir

### Philosophie de gestion d'erreurs

1. **Échec rapide** : Détecter et signaler les erreurs tôt
2. **Récupération gracieuse** : Toujours avoir un plan B
3. **Transparence** : Logger tout pour le debugging
4. **Expérience utilisateur** : Messages clairs et actions possibles
5. **Prévention** : Valider avant d'agir
6. **Résilience** : Le système doit continuer malgré les erreurs

### Erreurs courantes à éviter

❌ **Capturer toutes les exceptions** avec `except Exception:`
❌ **Ignorer les erreurs** silencieusement
❌ **Messages techniques** exposés aux utilisateurs
❌ **Pas de logging** des erreurs
❌ **Pas de stratégie de récupération**
❌ **Tests uniquement des cas de succès**

### Meilleures pratiques par niveau

#### 🥉 **Niveau Débutant**
```python
# ✅ Gestion basique mais correcte
def insertion_basique(conn, email, nom):
    try:
        conn.execute("INSERT INTO users (email, nom) VALUES (?, ?)", (email, nom))
        conn.commit()
        print("✅ Utilisateur créé")
        return True
    except sqlite3.IntegrityError:
        print("❌ Email déjà utilisé")
        return False
    except sqlite3.Error as e:
        print(f"❌ Erreur de base de données : {e}")
        conn.rollback()
        return False
```

#### 🥈 **Niveau Intermédiaire**
```python
# ✅ Gestion avec logging et validation
import logging

def insertion_intermediaire(conn, email, nom):
    logger = logging.getLogger(__name__)

    # Validation préalable
    if not email or '@' not in email:
        raise ValueError("Email invalide")

    if not nom or len(nom.strip()) < 2:
        raise ValueError("Nom trop court")

    try:
        conn.execute("INSERT INTO users (email, nom) VALUES (?, ?)",
                    (email.lower().strip(), nom.strip()))
        conn.commit()
        logger.info(f"Utilisateur créé : {email}")
        return True

    except sqlite3.IntegrityError as e:
        if "UNIQUE constraint failed" in str(e):
            logger.warning(f"Tentative de doublon : {email}")
            raise ValueError("Email déjà utilisé")
        else:
            logger.error(f"Violation d'intégrité : {e}")
            raise

    except sqlite3.Error as e:
        logger.error(f"Erreur SQLite lors de l'insertion : {e}")
        conn.rollback()
        raise RuntimeError("Erreur lors de la création de l'utilisateur")
```

#### 🥇 **Niveau Avancé**
```python
# ✅ Gestion complète avec monitoring et récupération
class GestionnaireUtilisateur:
    def __init__(self, db_manager, monitoring_service=None):
        self.db_manager = db_manager
        self.monitoring = monitoring_service
        self.logger = logging.getLogger(self.__class__.__name__)
        self.metrics = {'creations': 0, 'echecs': 0}

    @retry(max_attempts=3, backoff_strategy='exponential')
    @circuit_breaker(failure_threshold=5)
    def creer_utilisateur(self, email: str, nom: str) -> UserResult:
        # Validation avec exceptions métier
        self._valider_donnees_utilisateur(email, nom)

        try:
            with self.db_manager.transaction() as tx:
                user_id = tx.execute(
                    "INSERT INTO users (email, nom) VALUES (?, ?) RETURNING id",
                    (email.lower().strip(), nom.strip())
                ).fetchone()[0]

                # Log structuré
                self.logger.info("Utilisateur créé", extra={
                    'user_id': user_id,
                    'email': email,
                    'operation': 'create_user'
                })

                self.metrics['creations'] += 1
                return UserResult.success(user_id)

        except sqlite3.IntegrityError as e:
            self._handle_integrity_error(e, email)
        except sqlite3.OperationalError as e:
            self._handle_operational_error(e, email)
        except Exception as e:
            self._handle_unexpected_error(e, email)

    def _valider_donnees_utilisateur(self, email: str, nom: str):
        errors = []

        if not email or not re.match(r'^[^@]+@[^@]+\.[^@]+$', email):
            errors.append("Email invalide")

        if not nom or len(nom.strip()) < 2:
            errors.append("Nom trop court (minimum 2 caractères)")

        if len(nom.strip()) > 100:
            errors.append("Nom trop long (maximum 100 caractères)")

        if errors:
            raise ValidationError(errors)

    def _handle_integrity_error(self, error: sqlite3.IntegrityError, email: str):
        self.metrics['echecs'] += 1

        if "UNIQUE constraint failed: users.email" in str(error):
            self.logger.warning(f"Tentative de création avec email existant : {email}")
            raise ConflictError("Cette adresse email est déjà utilisée")
        else:
            self.logger.error(f"Violation d'intégrité inattendue : {error}")
            if self.monitoring:
                self.monitoring.record_error('integrity_violation', error)
            raise BusinessError("Impossible de créer l'utilisateur (données invalides)")
```

## Guide de migration progressive

### Étape 1 : Audit de l'existant

```python
def auditer_gestion_erreurs_existante(fichiers_code):
    """Outil d'audit pour évaluer la gestion d'erreurs actuelle"""

    problemes_detectes = {
        'except_trap_all': [],  # except: ou except Exception:
        'silent_failures': [],  # pass sans logging
        'no_rollback': [],      # pas de rollback en cas d'erreur
        'technical_messages': [], # messages techniques exposés
        'no_validation': []     # pas de validation d'entrée
    }

    recommendations = []

    print("🔍 AUDIT DE LA GESTION D'ERREURS")
    print("=" * 50)

    # Patterns à rechercher dans le code
    patterns_problematiques = {
        'except_trap_all': [
            r'except\s*:',
            r'except\s+Exception\s*:'
        ],
        'silent_failures': [
            r'except.*:\s*pass',
            r'except.*:\s*continue'
        ],
        'no_logging': [
            r'except.*sqlite3\..*:(?!.*log)'
        ]
    }

    # Simulation d'analyse (en réalité, analyserait les fichiers)
    print("📊 Résultats de l'audit :")
    print("   • 15 blocs try/except trouvés")
    print("   • 8 utilisent 'except Exception:' (à corriger)")
    print("   • 3 ignorent les erreurs silencieusement")
    print("   • 5 n'ont pas de logging")
    print("   • 2 exposent des messages techniques")

    recommendations = [
        "🔧 Remplacer 'except Exception:' par des exceptions spécifiques",
        "📝 Ajouter du logging dans tous les blocs except",
        "✅ Implémenter des validations d'entrée",
        "👥 Créer des messages d'erreur orientés utilisateur",
        "🔄 Ajouter des mécanismes de rollback",
        "📊 Intégrer un système de monitoring"
    ]

    print("\n💡 Recommandations :")
    for i, rec in enumerate(recommendations, 1):
        print(f"   {i}. {rec}")

    return problemes_detectes, recommendations

# Lancer l'audit
auditer_gestion_erreurs_existante([])
```

### Étape 2 : Plan de migration par priorité

```python
class PlanMigrationGestionErreurs:
    """Plan de migration progressive de la gestion d'erreurs"""

    def __init__(self):
        self.phases = [
            {
                'nom': 'Phase 1 - Sécurisation critique',
                'duree': '1-2 semaines',
                'priorite': 'HAUTE',
                'objectifs': [
                    'Éliminer les "except:" sans gestion',
                    'Ajouter rollback manquants',
                    'Corriger les fuites de ressources',
                    'Valider les entrées critiques'
                ],
                'livrable': 'Code sans crashs critiques'
            },
            {
                'nom': 'Phase 2 - Logging et monitoring',
                'duree': '2-3 semaines',
                'priorite': 'HAUTE',
                'objectifs': [
                    'Implémenter logging structuré',
                    'Ajouter métriques de base',
                    'Créer alertes sur erreurs critiques',
                    'Dashboard de santé basique'
                ],
                'livrable': 'Visibilité complète sur les erreurs'
            },
            {
                'nom': 'Phase 3 - Exceptions métier',
                'duree': '2-4 semaines',
                'priorite': 'MOYENNE',
                'objectifs': [
                    'Créer hiérarchie d\'exceptions métier',
                    'Implémenter pattern Repository',
                    'Messages d\'erreur utilisateur',
                    'Tests des cas d\'erreur'
                ],
                'livrable': 'API robuste avec gestion d\'erreurs propre'
            },
            {
                'nom': 'Phase 4 - Résilience avancée',
                'duree': '3-4 semaines',
                'priorite': 'MOYENNE',
                'objectifs': [
                    'Circuit Breaker',
                    'Stratégies de fallback',
                    'Récupération automatique',
                    'Monitoring avancé'
                ],
                'livrable': 'Système auto-réparant'
            },
            {
                'nom': 'Phase 5 - Optimisation',
                'duree': '2-3 semaines',
                'priorite': 'BASSE',
                'objectifs': [
                    'Optimisation des performances',
                    'Métriques avancées',
                    'Prédiction d\'erreurs',
                    'Documentation complète'
                ],
                'livrable': 'Système optimisé et documenté'
            }
        ]

    def afficher_plan(self):
        print("📋 PLAN DE MIGRATION - GESTION D'ERREURS")
        print("=" * 60)

        for i, phase in enumerate(self.phases, 1):
            print(f"\n🎯 {phase['nom']}")
            print(f"   ⏱️  Durée estimée : {phase['duree']}")
            print(f"   🔥 Priorité : {phase['priorite']}")
            print(f"   📦 Livrable : {phase['livrable']}")
            print(f"   📋 Objectifs :")
            for obj in phase['objectifs']:
                print(f"      • {obj}")

    def generer_checklist_phase(self, numero_phase):
        """Génère une checklist détaillée pour une phase"""
        if numero_phase < 1 or numero_phase > len(self.phases):
            return None

        phase = self.phases[numero_phase - 1]

        checklist = {
            'phase': phase['nom'],
            'taches': []
        }

        if numero_phase == 1:  # Phase critique
            checklist['taches'] = [
                "☐ Identifier tous les blocs try/except problématiques",
                "☐ Remplacer 'except:' par exceptions spécifiques",
                "☐ Ajouter rollback dans toutes les transactions",
                "☐ Fermer toutes les connexions dans finally",
                "☐ Valider tous les paramètres d'entrée",
                "☐ Tester les scénarios de panne",
                "☐ Code review avec focus sécurité"
            ]
        elif numero_phase == 2:  # Logging
            checklist['taches'] = [
                "☐ Configurer le système de logging",
                "☐ Ajouter logs dans tous les blocs except",
                "☐ Implémenter métriques de base",
                "☐ Créer dashboard de monitoring",
                "☐ Configurer alertes critiques",
                "☐ Tester les alertes",
                "☐ Former l'équipe au monitoring"
            ]

        return checklist

plan = PlanMigrationGestionErreurs()
plan.afficher_plan()

print("\n📝 Checklist Phase 1 :")
checklist = plan.generer_checklist_phase(1)
for tache in checklist['taches']:
    print(f"   {tache}")
```

### Étape 3 : Kit de démarrage rapide

```python
def generer_kit_demarrage():
    """Génère un kit de démarrage pour gestion d'erreurs SQLite"""

    kit_files = {
        'sqlite_errors.py': '''
"""
Module de gestion d'erreurs SQLite - Kit de démarrage
Copiez ce fichier dans votre projet et adaptez selon vos besoins
"""

import sqlite3
import logging
from functools import wraps
from typing import Optional, Any

# === EXCEPTIONS MÉTIER ===

class BusinessError(Exception):
    """Exception de base pour les erreurs métier"""
    pass

class ValidationError(BusinessError):
    """Erreur de validation des données"""
    pass

class NotFoundError(BusinessError):
    """Ressource non trouvée"""
    pass

class ConflictError(BusinessError):
    """Conflit de données"""
    pass

# === GESTIONNAIRE DE BASE ===

class SQLiteManager:
    """Gestionnaire SQLite avec gestion d'erreurs intégrée"""

    def __init__(self, db_path: str):
        self.db_path = db_path
        self.logger = logging.getLogger(self.__class__.__name__)

    def connect(self):
        """Connexion avec gestion d'erreurs"""
        try:
            conn = sqlite3.connect(self.db_path)
            conn.execute("PRAGMA foreign_keys = ON")
            return conn
        except sqlite3.Error as e:
            self.logger.error(f"Erreur connexion à {self.db_path}: {e}")
            raise BusinessError("Impossible de se connecter à la base de données")

    def execute_safe(self, sql: str, params: Optional[tuple] = None) -> Any:
        """Exécution sécurisée avec gestion d'erreurs"""
        try:
            with self.connect() as conn:
                if params:
                    result = conn.execute(sql, params)
                else:
                    result = conn.execute(sql)
                conn.commit()
                return result

        except sqlite3.IntegrityError as e:
            self._handle_integrity_error(e, sql, params)
        except sqlite3.OperationalError as e:
            self._handle_operational_error(e, sql, params)
        except sqlite3.Error as e:
            self.logger.error(f"Erreur SQLite: {e}")
            raise BusinessError("Erreur lors de l'opération sur la base de données")

    def _handle_integrity_error(self, error, sql, params):
        """Gestion des erreurs d'intégrité"""
        self.logger.warning(f"Violation d'intégrité: {error}")

        if "UNIQUE constraint failed" in str(error):
            raise ConflictError("Cette donnée existe déjà")
        elif "NOT NULL constraint failed" in str(error):
            raise ValidationError("Champ obligatoire manquant")
        elif "CHECK constraint failed" in str(error):
            raise ValidationError("Données invalides")
        else:
            raise BusinessError("Violation des règles de données")

    def _handle_operational_error(self, error, sql, params):
        """Gestion des erreurs opérationnelles"""
        self.logger.error(f"Erreur opérationnelle: {error}")

        if "database is locked" in str(error):
            raise BusinessError("Base de données temporairement indisponible")
        elif "no such table" in str(error):
            raise BusinessError("Structure de base de données invalide")
        else:
            raise BusinessError("Erreur technique temporaire")

# === DÉCORATEURS UTILES ===

def handle_sqlite_errors(func):
    """Décorateur pour gestion automatique des erreurs SQLite"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except BusinessError:
            # Re-lancer les erreurs métier
            raise
        except sqlite3.Error as e:
            logger = logging.getLogger(func.__module__)
            logger.error(f"Erreur SQLite dans {func.__name__}: {e}")
            raise BusinessError("Erreur lors de l'opération")
        except Exception as e:
            logger = logging.getLogger(func.__module__)
            logger.error(f"Erreur inattendue dans {func.__name__}: {e}")
            raise BusinessError("Erreur système inattendue")

    return wrapper

# === EXEMPLE D'UTILISATION ===

class UserService:
    """Service utilisateur avec gestion d'erreurs"""

    def __init__(self, db_path: str):
        self.db = SQLiteManager(db_path)

    @handle_sqlite_errors
    def create_user(self, email: str, name: str) -> int:
        """Crée un utilisateur avec validation"""
        # Validation
        if not email or '@' not in email:
            raise ValidationError("Email invalide")

        if not name or len(name.strip()) < 2:
            raise ValidationError("Nom trop court")

        # Insertion
        result = self.db.execute_safe(
            "INSERT INTO users (email, name) VALUES (?, ?) RETURNING id",
            (email.lower().strip(), name.strip())
        )

        return result.fetchone()[0]

    @handle_sqlite_errors
    def get_user(self, user_id: int) -> dict:
        """Récupère un utilisateur"""
        result = self.db.execute_safe(
            "SELECT id, email, name FROM users WHERE id = ?",
            (user_id,)
        )

        row = result.fetchone()
        if not row:
            raise NotFoundError(f"Utilisateur {user_id} non trouvé")

        return {
            'id': row[0],
            'email': row[1],
            'name': row[2]
        }

# Configuration du logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
        ''',

        'example_usage.py': '''
"""
Exemple d'utilisation du kit de démarrage
"""

from sqlite_errors import UserService, ValidationError, NotFoundError, ConflictError, BusinessError
import sqlite3

def demo_usage():
    """Démonstration de l'utilisation du kit"""

    # Créer la base de test
    with sqlite3.connect('test_kit.db') as conn:
        conn.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY,
                email TEXT UNIQUE NOT NULL,
                name TEXT NOT NULL
            )
        ''')

    # Utiliser le service
    service = UserService('test_kit.db')

    try:
        # Créer un utilisateur
        user_id = service.create_user("alice@test.com", "Alice")
        print(f"✅ Utilisateur créé avec ID: {user_id}")

        # Récupérer l'utilisateur
        user = service.get_user(user_id)
        print(f"✅ Utilisateur récupéré: {user['name']}")

        # Tenter de créer un doublon
        service.create_user("alice@test.com", "Alice Duplicate")

    except ValidationError as e:
        print(f"❌ Validation: {e}")
    except ConflictError as e:
        print(f"❌ Conflit: {e}")
    except NotFoundError as e:
        print(f"❌ Non trouvé: {e}")
    except BusinessError as e:
        print(f"❌ Erreur métier: {e}")

if __name__ == "__main__":
    demo_usage()
        ''',

        'tests_errors.py': '''
"""
Tests pour la gestion d'erreurs
"""

import pytest
import sqlite3
import tempfile
import os
from sqlite_errors import UserService, ValidationError, NotFoundError, ConflictError

class TestErrorHandling:
    """Tests de gestion d'erreurs"""

    def setup_method(self):
        """Setup pour chaque test"""
        self.temp_db = tempfile.mktemp(suffix='.db')

        # Créer la structure
        with sqlite3.connect(self.temp_db) as conn:
            conn.execute('''
                CREATE TABLE users (
                    id INTEGER PRIMARY KEY,
                    email TEXT UNIQUE NOT NULL,
                    name TEXT NOT NULL
                )
            ''')

        self.service = UserService(self.temp_db)

    def teardown_method(self):
        """Cleanup après chaque test"""
        if os.path.exists(self.temp_db):
            os.unlink(self.temp_db)

    def test_validation_email_invalide(self):
        """Test validation email invalide"""
        with pytest.raises(ValidationError, match="Email invalide"):
            self.service.create_user("email_invalide", "Test User")

    def test_validation_nom_trop_court(self):
        """Test validation nom trop court"""
        with pytest.raises(ValidationError, match="Nom trop court"):
            self.service.create_user("test@email.com", "A")

    def test_conflit_email_unique(self):
        """Test conflit sur email unique"""
        self.service.create_user("test@email.com", "User 1")

        with pytest.raises(ConflictError, match="existe déjà"):
            self.service.create_user("test@email.com", "User 2")

    def test_utilisateur_non_trouve(self):
        """Test utilisateur non trouvé"""
        with pytest.raises(NotFoundError, match="non trouvé"):
            self.service.get_user(999)

    def test_creation_et_lecture_normale(self):
        """Test du cas normal"""
        user_id = self.service.create_user("test@email.com", "Test User")
        user = self.service.get_user(user_id)

        assert user['email'] == "test@email.com"
        assert user['name'] == "Test User"

# Lancer les tests avec: pytest tests_errors.py -v
        '''
    }

    print("📦 KIT DE DÉMARRAGE GÉNÉRÉ")
    print("=" * 40)

    for filename, content in kit_files.items():
        print(f"\n📄 {filename}")
        print("   Fichier prêt à copier dans votre projet")
        # Dans un vrai projet, vous écririez ces fichiers
        # with open(filename, 'w') as f:
        #     f.write(content)

    print("\n🚀 INSTRUCTIONS D'UTILISATION :")
    print("1. Copiez les 3 fichiers dans votre projet")
    print("2. Adaptez selon vos besoins spécifiques")
    print("3. Lancez les tests pour vérifier")
    print("4. Intégrez progressivement dans votre code existant")

    return kit_files

generer_kit_demarrage()
```

## Tests de la gestion d'erreurs

### Stratégies de test complètes

```python
import pytest
import sqlite3
import tempfile
import os
from unittest.mock import patch, MagicMock

class TestStrategiesGestionErreurs:
    """Stratégies complètes de test pour la gestion d'erreurs"""

    def setup_method(self):
        """Configuration pour chaque test"""
        self.temp_db = tempfile.mktemp(suffix='.db')

    def teardown_method(self):
        """Nettoyage après chaque test"""
        if os.path.exists(self.temp_db):
            os.unlink(self.temp_db)

    # === TESTS DE VALIDATION ===

    def test_validation_donnees_entree(self):
        """Test de validation des données d'entrée"""
        service = UserService(self.temp_db)

        # Cas de validation échouée
        test_cases = [
            ("", "Test User", "Email vide"),
            ("email_sans_arobase", "Test User", "Email sans @"),
            ("test@email.com", "", "Nom vide"),
            ("test@email.com", "A", "Nom trop court"),
            (None, "Test User", "Email None"),
            ("test@email.com", None, "Nom None")
        ]

        for email, nom, description in test_cases:
            with pytest.raises(ValidationError, match=".*"):
                service.create_user(email, nom)
                print(f"❌ {description} : Validation échouée comme attendu")

    # === TESTS DE CONTRAINTES ===

    def test_contraintes_integrite(self):
        """Test des contraintes d'intégrité"""
        # Créer base avec contraintes
        with sqlite3.connect(self.temp_db) as conn:
            conn.execute('''
                CREATE TABLE test_constraints (
                    id INTEGER PRIMARY KEY,
                    email TEXT UNIQUE NOT NULL,
                    age INTEGER CHECK (age >= 0 AND age <= 150),
                    balance REAL CHECK (balance >= 0)
                )
            ''')

        db_manager = SQLiteManager(self.temp_db)

        # Test contrainte UNIQUE
        db_manager.execute_safe(
            "INSERT INTO test_constraints (email, age) VALUES (?, ?)",
            ("test@email.com", 25)
        )

        with pytest.raises(ConflictError):
            db_manager.execute_safe(
                "INSERT INTO test_constraints (email, age) VALUES (?, ?)",
                ("test@email.com", 30)
            )

        # Test contrainte CHECK
        with pytest.raises(ValidationError):
            db_manager.execute_safe(
                "INSERT INTO test_constraints (email, age) VALUES (?, ?)",
                ("test2@email.com", -5)
            )

    # === TESTS DE CONCURRENCE ===

    def test_gestion_verrous(self):
        """Test de gestion des verrous de base de données"""
        import threading
        import time

        # Créer base de test
        with sqlite3.connect(self.temp_db) as conn:
            conn.execute("CREATE TABLE counter (value INTEGER)")
            conn.execute("INSERT INTO counter VALUES (0)")

        errors_caught = []

        def transaction_longue():
            try:
                with sqlite3.connect(self.temp_db, timeout=1.0) as conn:
                    conn.execute("BEGIN EXCLUSIVE")
                    time.sleep(2)  # Simulation opération longue
                    conn.execute("UPDATE counter SET value = value + 1")
                    conn.commit()
            except sqlite3.OperationalError as e:
                errors_caught.append(str(e))

        # Lancer deux transactions simultanées
        thread1 = threading.Thread(target=transaction_longue)
        thread2 = threading.Thread(target=transaction_longue)

        thread1.start()
        time.sleep(0.1)  # Petit délai
        thread2.start()

        thread1.join()
        thread2.join()

        # Vérifier qu'au moins une erreur de verrou a été capturée
        assert any("database is locked" in error for error in errors_caught)
        print("✅ Gestion des verrous testée avec succès")

    # === TESTS DE RÉCUPÉRATION ===

    def test_recuperation_apres_erreur(self):
        """Test de récupération après erreur"""

        # Créer une base avec données initiales
        with sqlite3.connect(self.temp_db) as conn:
            conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
            conn.execute("INSERT INTO users (name) VALUES ('Alice')")

        service = UserService(self.temp_db)

        # Simuler une corruption temporaire
        with patch('sqlite3.connect') as mock_connect:
            # Premier appel : erreur
            mock_connect.side_effect = sqlite3.DatabaseError("Database corrupted")

            with pytest.raises(BusinessError):
                service.get_user(1)

            # Deuxième appel : récupération
            mock_connect.side_effect = None
            mock_connect.return_value.__enter__.return_value.execute.return_value.fetchone.return_value = (1, "Alice")

            # La récupération devrait fonctionner
            user = service.get_user(1)
            assert user['name'] == "Alice"
            print("✅ Récupération après erreur testée")

    # === TESTS DE MONITORING ===

    def test_monitoring_erreurs(self):
        """Test du système de monitoring des erreurs"""

        monitoring = MonitoringErreurs(seuil_alerte=2, fenetre_minutes=1)

        # Simuler plusieurs erreurs
        erreurs_test = [
            sqlite3.IntegrityError("UNIQUE constraint failed"),
            sqlite3.OperationalError("database is locked"),
            sqlite3.IntegrityError("UNIQUE constraint failed")
        ]

        for erreur in erreurs_test:
            monitoring.enregistrer_erreur(erreur, "Test monitoring")
            time.sleep(0.1)  # Petit délai entre les erreurs

        # Attendre que le monitoring traite les erreurs
        time.sleep(1)

        stats = monitoring.obtenir_statistiques()

        # Vérifications
        assert stats['erreurs_recentes'] == len(erreurs_test)
        assert 'IntegrityError' in stats['types_erreurs']
        assert stats['types_erreurs']['IntegrityError'] == 2

        monitoring.arreter()
        print("✅ Monitoring des erreurs testé avec succès")

    # === TESTS DE CIRCUIT BREAKER ===

    def test_circuit_breaker_functionality(self):
        """Test complet du Circuit Breaker"""

        circuit_breaker = CircuitBreakerSQLite(failure_threshold=3, recovery_timeout=1)

        def operation_qui_echoue():
            raise sqlite3.OperationalError("Simulation d'échec")

        def operation_qui_reussit():
            return "Succès"

        # Phase 1 : Circuit fermé, échecs comptabilisés
        for i in range(3):
            with pytest.raises(sqlite3.OperationalError):
                circuit_breaker.call(operation_qui_echoue)

        state = circuit_breaker.get_state()
        assert state['state'] == 'OPEN'
        print("✅ Circuit ouvert après échecs répétés")

        # Phase 2 : Circuit ouvert, appels bloqués
        with pytest.raises(Exception, match="Circuit Breaker OPEN"):
            circuit_breaker.call(operation_qui_reussit)
        print("✅ Appels bloqués avec circuit ouvert")

        # Phase 3 : Attendre la période de récupération
        time.sleep(1.1)  # Attendre recovery_timeout

        # Phase 4 : Test de récupération
        result = circuit_breaker.call(operation_qui_reussit)
        assert result == "Succès"

        state = circuit_breaker.get_state()
        assert state['state'] == 'CLOSED'
        print("✅ Circuit fermé après récupération réussie")

    # === TESTS DE FALLBACK ===

    def test_strategies_fallback(self):
        """Test des stratégies de fallback"""

        fallback_manager = FallbackManager()

        # Configurer les stratégies
        def fallback_cache():
            return "Données depuis cache"

        def fallback_defaut():
            return "Données par défaut"

        fallback_manager.ajouter_strategie("cache", fallback_cache, priorite=2)
        fallback_manager.ajouter_strategie("defaut", fallback_defaut, priorite=1)

        # Test fallback quand opération principale échoue
        def operation_principale():
            raise sqlite3.OperationalError("Base indisponible")

        result = fallback_manager.executer_avec_fallback(operation_principale)
        assert result == "Données depuis cache"
        print("✅ Fallback vers cache testé")

        # Test avec cache qui échoue aussi
        def fallback_cache_echec():
            raise Exception("Cache indisponible")

        fallback_manager.fallback_strategies[0]['fonction'] = fallback_cache_echec

        result = fallback_manager.executer_avec_fallback(operation_principale)
        assert result == "Données par défaut"
        print("✅ Fallback vers défaut testé")

# Lancer tous les tests
if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

## Scénarios de test avancés

### Tests d'intégration pour cas réels

```python
class TestScenariosReels:
    """Tests de scénarios réels complexes"""

    def test_scenario_ecommerce_complet(self):
        """Test d'un scénario e-commerce avec gestion d'erreurs complète"""

        # Setup base e-commerce
        with sqlite3.connect(':memory:') as conn:
            conn.execute('''
                CREATE TABLE produits (
                    id INTEGER PRIMARY KEY,
                    nom TEXT NOT NULL,
                    prix REAL CHECK (prix > 0),
                    stock INTEGER CHECK (stock >= 0)
                )
            ''')

            conn.execute('''
                CREATE TABLE commandes (
                    id INTEGER PRIMARY KEY,
                    client_email TEXT NOT NULL,
                    total REAL,
                    statut TEXT DEFAULT 'en_cours'
                )
            ''')

            conn.execute('''
                CREATE TABLE lignes_commande (
                    id INTEGER PRIMARY KEY,
                    commande_id INTEGER REFERENCES commandes(id),
                    produit_id INTEGER REFERENCES produits(id),
                    quantite INTEGER CHECK (quantite > 0),
                    prix_unitaire REAL
                )
            ''')

            # Données de test
            conn.execute("INSERT INTO produits (nom, prix, stock) VALUES ('Produit A', 29.99, 10)")
            conn.execute("INSERT INTO produits (nom, prix, stock) VALUES ('Produit B', 15.50, 5)")

        class ServiceCommande:
            def __init__(self, db_path):
                self.db = SQLiteManager(db_path)

            @handle_sqlite_errors
            def passer_commande(self, client_email, items):
                """Passe une commande avec vérification de stock"""

                # Validation
                if not client_email or '@' not in client_email:
                    raise ValidationError("Email client invalide")

                if not items:
                    raise ValidationError("Commande vide")

                with self.db.connect() as conn:
                    try:
                        conn.execute("BEGIN")

                        # Créer la commande
                        cursor = conn.execute(
                            "INSERT INTO commandes (client_email, total) VALUES (?, 0) RETURNING id",
                            (client_email,)
                        )
                        commande_id = cursor.fetchone()[0]

                        total = 0

                        for item in items:
                            produit_id = item['produit_id']
                            quantite = item['quantite']

                            # Vérifier le stock
                            cursor = conn.execute(
                                "SELECT prix, stock FROM produits WHERE id = ?",
                                (produit_id,)
                            )

                            row = cursor.fetchone()
                            if not row:
                                raise NotFoundError(f"Produit {produit_id} non trouvé")

                            prix, stock = row

                            if stock < quantite:
                                raise ConflictError(f"Stock insuffisant pour produit {produit_id}")

                            # Mettre à jour le stock
                            conn.execute(
                                "UPDATE produits SET stock = stock - ? WHERE id = ?",
                                (quantite, produit_id)
                            )

                            # Ajouter ligne de commande
                            conn.execute(
                                '''INSERT INTO lignes_commande
                                   (commande_id, produit_id, quantite, prix_unitaire)
                                   VALUES (?, ?, ?, ?)''',
                                (commande_id, produit_id, quantite, prix)
                            )

                            total += prix * quantite

                        # Mettre à jour le total
                        conn.execute(
                            "UPDATE commandes SET total = ?, statut = 'confirmee' WHERE id = ?",
                            (total, commande_id)
                        )

                        conn.commit()
                        return commande_id

                    except Exception:
                        conn.rollback()
                        raise

        # Tests du service
        service = ServiceCommande(':memory:')

        # Test commande normale
        commande_id = service.passer_commande("client@test.com", [
            {'produit_id': 1, 'quantite': 2},
            {'produit_id': 2, 'quantite': 1}
        ])

        assert commande_id is not None
        print("✅ Commande normale passée avec succès")

        # Test stock insuffisant
        with pytest.raises(ConflictError, match="Stock insuffisant"):
            service.passer_commande("client2@test.com", [
                {'produit_id': 1, 'quantite': 20}  # Plus que le stock disponible
            ])
        print("✅ Stock insuffisant détecté correctement")

        # Test produit inexistant
        with pytest.raises(NotFoundError, match="non trouvé"):
            service.passer_commande("client3@test.com", [
                {'produit_id': 999, 'quantite': 1}  # Produit inexistant
            ])
        print("✅ Produit inexistant détecté correctement")

    def test_scenario_migration_donnees(self):
        """Test de migration de données avec gestion d'erreurs"""

        class MigrateurDonnees:
            def __init__(self, source_db, target_db):
                self.source_db = source_db
                self.target_db = target_db
                self.logger = logging.getLogger(self.__class__.__name__)
                self.errors_count = 0
                self.success_count = 0

            def migrer_avec_gestion_erreurs(self):
                """Migration avec gestion complète des erreurs"""

                try:
                    with sqlite3.connect(self.source_db) as source:
                        with sqlite3.connect(self.target_db) as target:

                            # Préparer la structure cible
                            target.execute('''
                                CREATE TABLE IF NOT EXISTS users_migrated (
                                    id INTEGER PRIMARY KEY,
                                    email TEXT UNIQUE,
                                    name TEXT,
                                    migration_status TEXT DEFAULT 'success',
                                    migration_error TEXT
                                )
                            ''')

                            # Récupérer les données source
                            cursor = source.execute("SELECT id, email, name FROM users")

                            for row in cursor:
                                try:
                                    user_id, email, name = row

                                    # Validation des données
                                    if not email or '@' not in email:
                                        raise ValidationError(f"Email invalide: {email}")

                                    if not name or len(name.strip()) < 2:
                                        raise ValidationError(f"Nom invalide: {name}")

                                    # Insertion avec gestion des doublons
                                    try:
                                        target.execute(
                                            '''INSERT INTO users_migrated
                                               (id, email, name) VALUES (?, ?, ?)''',
                                            (user_id, email.lower().strip(), name.strip())
                                        )
                                        self.success_count += 1

                                    except sqlite3.IntegrityError as e:
                                        if "UNIQUE constraint failed" in str(e):
                                            # Log du doublon mais continuer
                                            self.logger.warning(f"Doublon ignoré: {email}")
                                            target.execute(
                                                '''INSERT INTO users_migrated
                                                   (id, email, name, migration_status, migration_error)
                                                   VALUES (?, ?, ?, 'skipped', 'duplicate')''',
                                                (user_id, email, name)
                                            )
                                        else:
                                            raise

                                except ValidationError as e:
                                    # Enregistrer l'erreur mais continuer
                                    self.errors_count += 1
                                    self.logger.error(f"Validation échouée pour user {user_id}: {e}")

                                    target.execute(
                                        '''INSERT INTO users_migrated
                                           (id, email, name, migration_status, migration_error)
                                           VALUES (?, ?, ?, 'error', ?)''',
                                        (user_id, email or '', name or '', str(e))
                                    )

                            target.commit()

                            return {
                                'success': self.success_count,
                                'errors': self.errors_count,
                                'total': self.success_count + self.errors_count
                            }

                except Exception as e:
                    self.logger.error(f"Erreur fatale de migration: {e}")
                    raise

        # Créer base source avec données problématiques
        source_db = ':memory:'
        target_db = ':memory:'

        with sqlite3.connect(source_db) as conn:
            conn.execute("CREATE TABLE users (id INTEGER, email TEXT, name TEXT)")

            # Données de test avec problèmes variés
            test_data = [
                (1, "alice@test.com", "Alice"),           # Normal
                (2, "bob@test.com", "Bob Martin"),        # Normal
                (3, "email_invalide", "Charlie"),         # Email invalide
                (4, "david@test.com", "D"),               # Nom trop court
                (5, "alice@test.com", "Alice Duplicate"), # Doublon email
                (6, "", "Empty Email"),                   # Email vide
                (7, "valid@test.com", ""),                # Nom vide
            ]

            conn.executemany("INSERT INTO users VALUES (?, ?, ?)", test_data)

        # Effectuer la migration
        migrateur = MigrateurDonnees(source_db, target_db)

        with patch('sqlite3.connect') as mock_connect:
            # Mock pour simuler les connexions
            mock_source = MagicMock()
            mock_target = MagicMock()

            # Configurer les mocks pour simuler le comportement réel
            # (simplification pour l'exemple)

            resultat = migrateur.migrer_avec_gestion_erreurs()

            assert resultat['total'] > 0
            assert resultat['errors'] > 0  # On s'attend à des erreurs

            print(f"✅ Migration terminée: {resultat['success']} succès, {resultat['errors']} erreurs")

# Tests de performance sous charge
class TestPerformanceSousCharge:
    """Tests de performance et stabilité sous charge"""

    def test_gestion_erreurs_sous_charge(self):
        """Test de la gestion d'erreurs sous charge élevée"""

        import threading
        import time
        from concurrent.futures import ThreadPoolExecutor, as_completed

        # Préparer la base
        test_db = tempfile.mktemp(suffix='.db')

        with sqlite3.connect(test_db) as conn:
            conn.execute('''
                CREATE TABLE stress_test (
                    id INTEGER PRIMARY KEY,
                    data TEXT,
                    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
                )
            ''')

        monitoring = MonitoringErreurs(seuil_alerte=50, fenetre_minutes=2)

        def operation_stress(thread_id, iterations):
            """Opération qui génère du stress et des erreurs"""
            service = SQLiteManager(test_db)
            errors_caught = 0
            success_count = 0

            for i in range(iterations):
                try:
                    if i % 10 == 0:
                        # Provoquer des erreurs périodiquement
                        service.execute_safe("INSERT INTO nonexistent_table VALUES (?)", (i,))
                    else:
                        # Opération normale
                        service.execute_safe("INSERT INTO stress_test (data) VALUES (?)", (f"Thread {thread_id} - {i}",))
                        success_count += 1

                except BusinessError:
                    errors_caught += 1
                    monitoring.enregistrer_erreur(
                        sqlite3.OperationalError("Table inexistante"),
                        f"Thread {thread_id}"
                    )

                # Petite pause pour simuler le travail
                time.sleep(0.001)

            return {
                'thread_id': thread_id,
                'success': success_count,
                'errors': errors_caught
            }

        # Lancer test de charge
        num_threads = 10
        iterations_per_thread = 100

        start_time = time.time()

        with ThreadPoolExecutor(max_workers=num_threads) as executor:
            futures = [
                executor.submit(operation_stress, i, iterations_per_thread)
                for i in range(num_threads)
            ]

            results = []
            for future in as_completed(futures):
                results.append(future.result())

        duration = time.time() - start_time

        # Analyser les résultats
        total_success = sum(r['success'] for r in results)
        total_errors = sum(r['errors'] for r in results)
        total_operations = total_success + total_errors

        print(f"\n📊 RÉSULTATS TEST DE CHARGE")
        print(f"   Durée: {duration:.2f} secondes")
        print(f"   Opérations totales: {total_operations}")
        print(f"   Succès: {total_success}")
        print(f"   Erreurs: {total_errors}")
        print(f"   Taux de succès: {total_success/total_operations*100:.1f}%")
        print(f"   Opérations/seconde: {total_operations/duration:.1f}")

        # Vérifier les métriques de monitoring
        stats = monitoring.obtenir_statistiques()
        print(f"   Erreurs monitorées: {stats['erreurs_recentes']}")

        monitoring.arreter()

        # Nettoyage
        if os.path.exists(test_db):
            os.unlink(test_db)

        # Assertions
        assert total_success > 0, "Aucune opération réussie"
        assert total_errors > 0, "Aucune erreur générée (test invalide)"
        assert duration < 30, "Test trop lent"

        print("✅ Test de charge terminé avec succès")

if __name__ == "__main__":
    # Exécuter tous les tests
    test_scenarios = TestScenariosReels()
    test_scenarios.test_scenario_ecommerce_complet()
    test_scenarios.test_scenario_migration_donnees()

    test_perf = TestPerformanceSousCharge()
    test_perf.test_gestion_erreurs_sous_charge()
```

## Conclusion de la section 6.5

### Récapitulatif complet de la gestion d'erreurs

```python
def conclusion_gestion_erreurs():
    """Conclusion complète de la section gestion d'erreurs"""

    print("🎯 CONCLUSION - GESTION D'ERREURS SQLITE")
    print("=" * 60)

    competences_acquises = {
        "🔍 Diagnostic et détection": [
            "✅ Identifier les différents types d'erreurs SQLite",
            "✅ Comprendre la hiérarchie des exceptions",
            "✅ Diagnostiquer les problèmes de performance",
            "✅ Détecter les corruptions et problèmes d'intégrité"
        ],

        "🛡️ Prévention et validation": [
            "✅ Valider les données avant insertion",
            "✅ Gérer les contraintes d'intégrité",
            "✅ Prévenir les injections SQL",
            "✅ Contrôler les accès concurrents"
        ],

        "🔧 Gestion et récupération": [
            "✅ Implémenter des try/except spécialisés",
            "✅ Créer des exceptions métier personnalisées",
            "✅ Développer des stratégies de retry",
            "✅ Mettre en place des mécanismes de fallback"
        ],

        "📊 Monitoring et observabilité": [
            "✅ Logger les erreurs avec contexte",
            "✅ Monitorer les métriques en temps réel",
            "✅ Créer des alertes automatiques",
            "✅ Générer des rapports de santé"
        ],

        "🏗️ Architecture robuste": [
            "✅ Pattern Repository avec gestion d'erreurs",
            "✅ Services avec Circuit Breaker",
            "✅ Systèmes de récupération automatique",
            "✅ Tests complets des cas d'erreur"
        ],

        "👥 Expérience utilisateur": [
            "✅ Messages d'erreur clairs et utiles",
            "✅ Actions de récupération suggérées",
            "✅ Interface gracieuse en cas de problème",
            "✅ Feedback en temps réel"
        ]
    }

    for categorie, competences in competences_acquises.items():
        print(f"\n{categorie}")
        for competence in competences:
            print(f"  {competence}")

    print(f"\n🏆 NIVEAU ATTEINT")
    print("Vous maîtrisez maintenant la gestion d'erreurs SQLite de niveau professionnel !")

    print(f"\n📚 RESSOURCES POUR ALLER PLUS LOIN")
    ressources = [
        "📖 Documentation SQLite officielle sur les codes d'erreur",
        "🔗 Patterns de résilience (Circuit Breaker, Bulkhead, etc.)",
        "📊 Outils de monitoring (Prometheus, Grafana)",
        "🧪 Frameworks de test (pytest, unittest, property-based testing)",
        "🏗️ Architecture Event-Driven pour gestion d'erreurs",
        "☁️ Observabilité distribuée (OpenTelemetry, Jaeger)"
    ]

    for ressource in ressources:
        print(f"  {ressource}")

    print(f"\n🚀 PROCHAINES ÉTAPES RECOMMANDÉES")
    etapes = [
        "1. Appliquer ces techniques à vos projets existants",
        "2. Créer une bibliothèque de gestion d'erreurs réutilisable",
        "3. Former votre équipe aux bonnes pratiques",
        "4. Intégrer dans vos processus CI/CD",
        "5. Mesurer l'impact sur la qualité produit",
        "6. Contribuer à la communauté (articles, talks, code)"
    ]

    for etape in etapes:
        print(f"  {etape}")

    print(f"\n💡 CONSEIL FINAL")
    print("""
    La gestion d'erreurs n'est pas un ajout optionnel à votre code,
    c'est une partie intégrante de l'architecture logicielle.

    Un système robuste n'est pas un système qui ne tombe jamais en panne,
    mais un système qui sait comment récupérer gracieusement des pannes.

    Continuez à pratiquer, tester, et améliorer vos techniques.
    Vos utilisateurs vous remercieront ! 🙏
    """)

conclusion_gestion_erreurs()
```

### Mémo de référence rapide

```python
def memo_reference_rapide():
    """Mémo de référence rapide pour la gestion d'erreurs SQLite"""

    memo = """
    📋 MÉMO - GESTION D'ERREURS SQLITE
    ================================

    🔥 URGENCES (À CORRIGER IMMÉDIATEMENT)
    ❌ except: ou except Exception:          → Utiliser exceptions spécifiques
    ❌ pass sans logging                     → Ajouter logging d'erreur
    ❌ Pas de rollback en cas d'erreur       → Ajouter try/finally avec rollback
    ❌ Connexions non fermées               → Utiliser context managers

    ✅ PATTERNS DE BASE
    try:
        # Opération SQLite
    except sqlite3.IntegrityError as e:
        # Gestion spécifique des contraintes
    except sqlite3.OperationalError as e:
        # Gestion des verrous, tables manquantes
    except sqlite3.Error as e:
        # Gestion générale SQLite
    finally:
        # Nettoyage (fermeture connexions)

    🛡️ VALIDATION PRÉVENTIVE
    • Valider AVANT insertion
    • Vérifier existence des tables/colonnes
    • Contrôler les types de données
    • Limiter la taille des requêtes

    📝 LOGGING ESSENTIEL
    logger.error("Erreur SQLite", extra={
        'sql': sql_query,
        'params': parameters,
        'error': str(error),
        'context': operation_context
    })

    🔄 RETRY INTELLIGENT
    @retry(max_attempts=3, backoff='exponential')
    def operation_avec_retry():
        # Opération qui peut échouer temporairement

    🚨 CIRCUIT BREAKER
    if circuit_breaker.is_open():
        return fallback_response()

    📊 MONITORING
    • Compter les erreurs par type
    • Mesurer les temps de réponse
    • Alerter sur seuils dépassés
    • Générer rapports de santé

    👥 MESSAGES UTILISATEUR
    ❌ "sqlite3.IntegrityError: UNIQUE constraint failed"
    ✅ "Cette adresse email est déjà utilisée"

    🧪 TESTS OBLIGATOIRES
    • Cas d'erreur (pas seulement succès)
    • Conditions de charge
    • Récupération après panne
    • Validation des limites

    🏗️ ARCHITECTURE ROBUSTE
    Service → Repository → Database
       ↓         ↓           ↓
    Exceptions → Logging → Monitoring
    métier     structuré   alertes
    """

    print(memo)

memo_reference_rapide()
```

## Transition vers la section suivante

```python
def transition_vers_fts():
    """Transition vers la section 6.6 - Recherche plein texte"""

    print("🔗 TRANSITION VERS LA SECTION 6.6")
    print("=" * 50)

    print("""
    Félicitations ! Vous maîtrisez maintenant la gestion d'erreurs SQLite
    de niveau professionnel. Votre code est robuste, monitore, et récupère
    gracieusement des pannes.

    🎯 PROCHAINE ÉTAPE : RECHERCHE PLEIN TEXTE (FTS5)

    Dans la section 6.6, nous explorerons :

    🔍 FONCTIONNALITÉS FTS5
    • Recherche en texte intégral
    • Classement par pertinence
    • Recherche floue et approximative
    • Highlight des résultats
    • Requêtes complexes avec opérateurs

    💡 CAS D'USAGE PRATIQUES
    • Moteur de recherche pour blog/site web
    • Recherche dans documentation
    • Analyse de logs et données textuelles
    • Recherche dans emails/messages
    • Base de connaissances

    🏗️ INTÉGRATION AVANCÉE
    • FTS5 avec applications web
    • Performance et optimisation
    • Indexation en temps réel
    • Multi-langues et synonymes
    • APIs de recherche

    Cette dernière section complètera votre maîtrise de la programmation
    avancée SQLite en vous donnant les outils pour créer des fonctionnalités
    de recherche de niveau professionnel.

    Prêt pour cette dernière aventure ? 🚀
    """)

transition_vers_fts()
```

## Points clés de cette section 6.5

### Ce que vous avez maîtrisé

✅ **Diagnostic d'erreurs** : Identification et classification des erreurs SQLite
✅ **Gestion préventive** : Validation, contraintes, et bonnes pratiques
✅ **Récupération robuste** : Retry, Circuit Breaker, et stratégies de fallback
✅ **Monitoring avancé** : Logging, métriques, et alertes automatiques
✅ **Architecture résiliente** : Patterns professionnels et tests complets
✅ **Expérience utilisateur** : Messages clairs et récupération gracieuse

### Impact sur vos projets

🏆 **Code plus fiable** : Moins de crashs et de bugs en production
📊 **Visibilité totale** : Monitoring et diagnostics complets
🔧 **Maintenance simplifiée** : Problèmes détectés et résolus rapidement
👥 **Utilisateurs satisfaits** : Interface robuste et messages clairs
🚀 **Déploiements sereins** : Confiance dans la stabilité du système

La gestion d'erreurs n'est plus un mystère pour vous - c'est maintenant un atout professionnel qui distingue vos applications !

⏭️
