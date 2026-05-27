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

> ⚠️ **PIÈGE FRÉQUENT à connaître pour ce chapitre** : `with sqlite3.connect(...) as conn:` **NE FERME PAS** la connexion à la sortie du bloc — il ne fait que **commit/rollback automatique** selon le résultat. La connexion reste ouverte jusqu'à `conn.close()` explicite (ou jusqu'à ce que le garbage collector libère l'objet).  
>
> ```python
> # ❌ Fuite potentielle : la connexion reste ouverte après le `with`
> with sqlite3.connect('db.db') as conn:
>     conn.execute("SELECT 1")
> # conn.execute(...) marche ENCORE ici !
>
> # ✅ Pattern correct : combiner `with conn:` (transaction) et `conn.close()`
> conn = sqlite3.connect('db.db')
> try:
>     with conn:               # gère commit/rollback automatique
>         conn.execute("SELECT 1")
> finally:
>     conn.close()             # libère vraiment la ressource
>
> # ✅ Alternative idiomatique : `contextlib.closing()`
> from contextlib import closing
> with closing(sqlite3.connect('db.db')) as conn:
>     with conn:               # transaction
>         conn.execute("SELECT 1")
> # close + commit/rollback automatique
> ```
>  
> Dans un script court qui termine, le piège est sans conséquence (le GC ferme). Dans un serveur long-lived ou une boucle, il provoque des **fuites de fichiers ouverts**.

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
        conn.isolation_level = None   # ← pour `BEGIN ...` manuel (voir 6.3)

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
            except sqlite3.Error:
                pass  # connexion peut déjà être fermée

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

        # ⚠️ ORDRE CRUCIAL : du plus SPÉCIFIQUE au plus GÉNÉRAL.
        # `OperationalError` hérite de `DatabaseError` — si on capture
        # `DatabaseError` en premier, `OperationalError` n'est JAMAIS atteint.
        except PermissionError as e:
            print(f"🔒 {description} : Erreur de permission OS - {e}")
        except sqlite3.OperationalError as e:
            print(f"⚙️ {description} : Erreur opérationnelle - {e}")
        except sqlite3.DatabaseError as e:
            print(f"🗃️ {description} : Erreur de base de données - {e}")
        except Exception as e:
            print(f"❌ {description} : Erreur inattendue - {e}")

demo_erreurs_connexion()
```

## Hiérarchie des exceptions SQLite

### Structure des exceptions

**Arbre d'héritage (à mémoriser pour ordonner correctement les `except`) :**

```text
Warning                                     (avertissement, rarement levé)  
Exception  
└── Error                                   (classe de base de toutes les erreurs DB)
    ├── InterfaceError                      (problème dans le module sqlite3 lui-même)
    └── DatabaseError                       (erreur côté base de données)
        ├── DataError                       (donnée invalide : overflow, format...)
        ├── OperationalError                (verrou, table absente, disk full...)
        ├── IntegrityError                  (violation contrainte UNIQUE/FK/CHECK)
        ├── InternalError                   (erreur interne SQLite)
        ├── ProgrammingError                (mauvais usage de l'API : curseur fermé...)
        └── NotSupportedError               (fonctionnalité indisponible)
```

> ⚠️ **Règle d'or pour `try/except`** : capturer du **plus spécifique au plus général**. Sinon, la classe parente intercepte avant les enfants — et les branches spécialisées ne sont jamais exécutées.

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
import time              # ← requis pour time.sleep() dans le wrapper  
import sqlite3  
import logging  
from functools import wraps  

def gestion_erreurs_sqlite(retry_count=3, delay=1):
    """Décorateur pour gestion automatique des erreurs SQLite.

    Retry uniquement sur `database is locked` (verrou temporaire).
    Backoff exponentiel : `delay * 2**tentative`.
    """

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
                        # Vrai backoff exponentiel : delai * 2^tentative
                        # (1, 2, 4, 8 secondes...). Le code précédent
                        # `delay * (tentative + 1)` était LINÉAIRE (1, 2, 3, 4...).
                        wait = delay * (2 ** tentative)
                        print(f"🔒 Base verrouillée, tentative {tentative + 1}/{retry_count} — attente {wait}s")
                        time.sleep(wait)
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

    conn = sqlite3.connect(db_path, timeout=1.0)
    conn.isolation_level = None   # ← pour `BEGIN ...` manuel
    try:
        conn.execute("BEGIN IMMEDIATE")

        for operation in operations:
            conn.execute(operation['sql'], operation.get('params', ()))

        conn.execute("COMMIT")
        print("✅ Toutes les opérations réussies")
    finally:
        conn.close()

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

        # Stack trace via l'attribut __traceback__ attaché à l'exception.
        # ⚠️ Ne PAS utiliser `traceback.format_exc()` ici : cette fonction ne
        #    retourne le bon traceback que si on est ENCORE dans le bloc except
        #    actif au moment de l'appel. Si log_error() est appelé après
        #    sortie du except (ex : exception capturée et passée plus tard),
        #    `format_exc()` retourne "NoneType: None". `format_exception` est
        #    robuste car il reçoit explicitement le traceback de l'exception.
        if error.__traceback__ is not None:
            tb = ''.join(traceback.format_exception(type(error), error, error.__traceback__))
            self.logger.debug(f"Stack trace:\n{tb}")

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
        """Test simple : tenter un verrou exclusif pour détecter d'autres writers."""
        try:
            conn = sqlite3.connect(self.db_path)
            conn.isolation_level = None   # ← pour `BEGIN ...` manuel
            try:
                conn.execute("BEGIN EXCLUSIVE")
                print("✅ Aucun verrou actif détecté")
                conn.execute("ROLLBACK")
                return True
            finally:
                conn.close()

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
# Imports nécessaires pour ce bloc (au niveau module pour être visibles
# dans TOUTES les méthodes de la classe — `import os` placé dans une seule
# méthode n'est PAS accessible aux autres méthodes).
import os  
import shutil  
import subprocess  
import time  
import sqlite3  

class RecuperationAutomatique:
    """Système de récupération automatique d'erreurs"""

    def __init__(self, db_path, backup_path=None):
        self.db_path = db_path
        self.backup_path = backup_path or f"{db_path}.backup"
        self.logger = SQLiteLogger("recovery")

    def creer_backup_urgence(self):
        """Crée un backup d'urgence avant récupération"""
        try:
            backup_urgence = f"{self.db_path}.emergency_{int(time.time())}"
            shutil.copy2(self.db_path, backup_urgence)
            self.logger.logger.info(f"Backup d'urgence créé : {backup_urgence}")
            return backup_urgence
        except (OSError, shutil.Error) as e:
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
        dump_file = f"{self.db_path}.dump"
        restored_file = f"{self.db_path}.restored"

        try:
            # Dump de la base — encoding='utf-8' obligatoire pour préserver
            # les caractères non-ASCII des données (accents, emojis...).
            with open(dump_file, 'w', encoding='utf-8') as f:
                subprocess.run([
                    'sqlite3', self.db_path, '.dump'
                ], stdout=f, check=True)

            # Restore dans un nouveau fichier
            with open(dump_file, 'r', encoding='utf-8') as f:
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
                    except OSError:
                        pass  # fichier déjà supprimé ou verrouillé

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

## Monitoring en temps réel des erreurs (résumé)

Pour des applications de production, il est utile de **monitorer** les erreurs en temps réel pour détecter rapidement les patterns problématiques (vague d'erreurs `database is locked`, augmentation des `IntegrityError`, etc.).

**Approche typique** :
- Un thread séparé qui consomme une `queue.Queue` d'erreurs enregistrées par l'application.
- Une fenêtre glissante (`collections.deque`) pour compter les erreurs dans les N dernières minutes.
- Un compteur par type d'exception (`defaultdict(int)`).
- Des seuils d'alerte (>= N erreurs en X minutes) qui déclenchent email/Slack/PagerDuty.

Plutôt que de réimplémenter ce monitoring, utiliser un **outil dédié** en production :
- **Sentry** (intégration Python facile)
- **Prometheus + Grafana** (métriques détaillées)
- **logging** Python avec handler SMTP/Slack (cas simples)

Ces outils gèrent déjà : agrégation, déduplication, alertes, dashboards. Réinventer la roue n'est généralement justifié que pour des contraintes de déploiement spécifiques (offline, air-gapped).


## Patterns de gestion d'erreurs pour applications

### Pattern Repository avec exceptions métier

Le pattern Repository **isole l'accès aux données** dans une couche dédiée et convertit les  
erreurs SQLite techniques en **exceptions métier** lisibles. C'est l'approche recommandée  
pour toute application non-triviale.  

```python
import sqlite3  
from typing import Optional  

# 1. Hiérarchie d'exceptions métier
class RepositoryError(Exception):
    """Exception de base pour les erreurs de repository."""

class ValidationError(RepositoryError):
    """Erreur de validation des données (côté métier)."""

class NotFoundError(RepositoryError):
    """Entité non trouvée."""

class ConflictError(RepositoryError):
    """Conflit de données (doublon UNIQUE, etc.)."""


# 2. Repository concret
class UtilisateurRepository:
    """Accès aux données utilisateurs avec gestion d'erreurs métier."""

    def __init__(self, db_path: str):
        self.db_path = db_path

    def _connect(self) -> sqlite3.Connection:
        conn = sqlite3.connect(self.db_path)
        conn.execute("PRAGMA foreign_keys = ON")
        return conn

    def initialiser_schema(self):
        with self._connect() as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS utilisateurs (
                    id INTEGER PRIMARY KEY,
                    email TEXT UNIQUE NOT NULL,
                    nom TEXT NOT NULL,
                    age INTEGER CHECK (age >= 0 AND age <= 150)
                )
            """)

    def creer_utilisateur(self, email: str, nom: str, age: int) -> int:
        # Validation métier AVANT d'aller en base
        if not email or '@' not in email:
            raise ValidationError("Email invalide")
        if not nom or len(nom.strip()) < 2:
            raise ValidationError("Nom trop court (min 2 caractères)")
        if age < 0 or age > 150:
            raise ValidationError("Âge invalide (doit être 0-150)")

        # Insertion avec conversion d'erreurs SQLite → métier
        conn = self._connect()
        try:
            cur = conn.execute(
                "INSERT INTO utilisateurs (email, nom, age) VALUES (?, ?, ?) RETURNING id",
                (email.strip().lower(), nom.strip(), age)
            )
            rows = cur.fetchall()
            conn.commit()
            return rows[0][0]

        except sqlite3.IntegrityError as e:
            conn.rollback()
            if "UNIQUE constraint failed" in str(e):
                raise ConflictError(f"L'email '{email}' est déjà utilisé") from e
            raise ValidationError(f"Violation d'intégrité : {e}") from e

        except sqlite3.Error as e:
            conn.rollback()
            raise RepositoryError(f"Erreur DB lors de la création : {e}") from e

        finally:
            conn.close()

    def obtenir_utilisateur(self, user_id: int) -> dict:
        with self._connect() as conn:
            row = conn.execute(
                "SELECT id, email, nom, age FROM utilisateurs WHERE id = ?",
                (user_id,)
            ).fetchone()

        if row is None:
            raise NotFoundError(f"Utilisateur {user_id} introuvable")

        return {'id': row[0], 'email': row[1], 'nom': row[2], 'age': row[3]}


# 3. Utilisation : on capture les exceptions MÉTIER, pas SQLite
# ⚠️ NE PAS utiliser `:memory:` ici : chaque _connect() ouvrirait une base DIFFÉRENTE.
#    Pour un repository, utiliser un fichier sur disque.
import tempfile, os  
db_file = tempfile.mktemp(suffix='.db')  

repo = UtilisateurRepository(db_file)  
repo.initialiser_schema()  

try:
    uid = repo.creer_utilisateur("alice@example.com", "Alice", 25)
    print(f"✅ Créé : id={uid}")

    # Tentative de doublon
    repo.creer_utilisateur("alice@example.com", "Alice 2", 30)

except ValidationError as e:
    print(f"❌ Données invalides : {e}")
except ConflictError as e:
    print(f"❌ Conflit : {e}")
except NotFoundError as e:
    print(f"❌ Introuvable : {e}")
except RepositoryError as e:
    print(f"❌ Erreur technique : {e}")

if os.path.exists(db_file):
    os.remove(db_file)
```

**Avantages du pattern** :
- Le **code applicatif** ne voit que des exceptions métier claires.
- Les **détails SQLite** (codes d'erreur, messages techniques) restent dans le repository.
- Les **tests unitaires** mockent facilement le repository.
- L'**évolution future** (changer de SGBD, ajouter un cache, etc.) est contenue.

## Pattern Circuit Breaker (résumé)

Pour les services SQLite à forte concurrence, on peut implémenter un **Circuit Breaker** qui ouvre le circuit après N échecs consécutifs et bloque les appels pendant un délai de récupération. Voir [Circuit Breaker pattern](https://martinfowler.com/bliki/CircuitBreaker.html) pour les détails — le pattern est externe au scope strict de SQLite.

## Stratégies de fallback (résumé)

En cas d'indisponibilité de la base principale, on peut chaîner des stratégies de secours :
1. Lecture depuis une base **backup** locale (si disponible)
2. Lecture depuis un **cache fichier** (JSON, pickle)
3. Retour de **données par défaut** dégradées

Implémentation typique : un `FallbackManager` qui essaie chaque stratégie dans l'ordre et retourne la première qui réussit. Logique applicative qui dépasse le scope SQLite strict.

## Récapitulatif et bonnes pratiques

### Checklist en 6 axes

**🔍 Détection d'erreurs**
- ✅ Bloc `try/except` avec exceptions spécifiques (`IntegrityError`, `OperationalError`...)
- ✅ Valider les données AVANT insertion (longueur, format, range)
- ✅ Vérifier l'existence des tables/colonnes au démarrage

**📝 Logging et diagnostic**
- ✅ Logger toutes les erreurs avec contexte (SQL, params, fonction)
- ✅ Inclure la stack trace via `error.__traceback__` (pas `format_exc()` hors except)
- ✅ Tracer les performances (durée, lignes affectées)

**🔄 Récupération**
- ✅ Retry avec backoff EXPONENTIEL (`delay * 2**tentative`, pas linéaire)
- ✅ Rollback systématique dans le `except` puis `raise`
- ✅ Sauvegardes régulières (voir 6.4)

**🛡️ Protection préventive**
- ✅ Gestionnaire de contexte (`with`) pour libérer les ressources
- ✅ Timeout sur les connexions (`sqlite3.connect(db, timeout=30.0)`)
- ✅ Validation côté serveur (ne jamais faire confiance au client)

**👥 Expérience utilisateur**
- ✅ Messages compréhensibles (`'Email déjà utilisé'` vs `'UNIQUE constraint failed: ...'`)
- ✅ Pas d'exposition d'informations techniques sensibles
- ✅ Suggestions d'action quand pertinent

**🏗️ Architecture**
- ✅ Séparation couches métier / accès données (pattern Repository)
- ✅ Exceptions métier personnalisées (`ValidationError`, `NotFoundError`, `ConflictError`)
- ✅ Tests automatisés des cas d'erreur (pas seulement les cas heureux)

### Niveau d'implémentation par maturité

**🥉 Débutant** : `try/except` basique avec exceptions SQLite spécifiques + `rollback`.

**🥈 Intermédiaire** : validation préalable + logging + exceptions métier personnalisées.

**🥇 Avancé** : pattern Repository + retry décorateur + monitoring + tests pytest des cas d'erreur.

## Points clés à retenir

**Philosophie** : Échec rapide, récupération gracieuse, transparence (logs), expérience utilisateur claire, prévention (validation), résilience.

**Erreurs courantes à éviter** :
- ❌ `except Exception:` ou `except:` génériques
- ❌ Ignorer les erreurs silencieusement (`pass`)
- ❌ Exposer des messages techniques aux utilisateurs
- ❌ Pas de logging des erreurs
- ❌ Pas de stratégie de récupération
- ❌ Tester uniquement les cas de succès

**Pattern recommandé** (intermédiaire) :
```python
try:
    conn.execute('INSERT INTO users (email, nom) VALUES (?, ?)',
                 (email.lower().strip(), nom.strip()))
    conn.commit()
except sqlite3.IntegrityError as e:
    if 'UNIQUE constraint failed' in str(e):
        raise ConflictError('Email déjà utilisé')
    raise  # autre violation d'intégrité
except sqlite3.Error as e:
    conn.rollback()
    raise RuntimeError(f'Erreur DB : {e}')
```

## Guide de migration progressive (résumé)

Pour migrer une application existante vers une gestion d'erreurs robuste :

**Phase 1 — Sécurisation critique** (~1-2 semaines) : remplacer les `except:` génériques par des exceptions spécifiques, ajouter les rollbacks manquants, fermer les connexions dans `finally`.

**Phase 2 — Logging et monitoring** (~2-3 semaines) : configurer le logging structuré, ajouter les métriques de base, mettre en place des alertes sur erreurs critiques.

**Phase 3 — Exceptions métier** (~2-4 semaines) : créer une hiérarchie d'exceptions personnalisées, implémenter le pattern Repository, messages utilisateur clairs.

Pour démarrer rapidement, utilisez le kit de démarrage ci-dessous.

## Kit de démarrage rapide

> ℹ️ Ce kit propose une **variante du pattern Repository** présenté plus haut : ici la racine d'exceptions s'appelle `BusinessError` (au lieu de `RepositoryError`), et le `SQLiteManager` factorise la gestion d'erreurs SQLite → métier dans une couche dédiée. Les deux approches sont valides — choisissez selon vos préférences.

Les trois blocs Markdown ci-dessous sont des **fichiers indépendants à copier** dans votre projet.

#### Fichier 1 : `sqlite_errors.py`

```python
"""
Module de gestion d'erreurs SQLite — Kit de démarrage.  
Copiez ce fichier dans votre projet et adaptez selon vos besoins.  
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
    """Conflit de données (doublon, etc.)"""
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

    def execute_safe(self, sql: str, params: Optional[tuple] = None) -> list:
        """Exécution sécurisée avec gestion d'erreurs.

        Retourne la liste des lignes (et non un curseur). Les résultats sont
        consommés AVANT le commit, ce qui évite l'erreur SQLite
        `cannot commit transaction - SQL statements in progress` avec les
        clauses `RETURNING`.
        """
        conn = self.connect()
        try:
            cur = conn.execute(sql, params) if params else conn.execute(sql)
            rows = cur.fetchall()      # consommer AVANT commit
            conn.commit()
            return rows
        except sqlite3.IntegrityError as e:
            conn.rollback()
            self._handle_integrity_error(e, sql, params)
        except sqlite3.OperationalError as e:
            conn.rollback()
            self._handle_operational_error(e, sql, params)
        except sqlite3.Error as e:
            conn.rollback()
            self.logger.error(f"Erreur SQLite: {e}")
            raise BusinessError("Erreur lors de l'opération sur la base de données")
        finally:
            conn.close()

    def _handle_integrity_error(self, error, sql, params):
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
        self.logger.error(f"Erreur opérationnelle: {error}")
        if "database is locked" in str(error):
            raise BusinessError("Base de données temporairement indisponible")
        elif "no such table" in str(error):
            raise BusinessError("Structure de base de données invalide")
        else:
            raise BusinessError("Erreur technique temporaire")

# === DÉCORATEUR UTILE ===

def handle_sqlite_errors(func):
    """Décorateur pour gestion automatique des erreurs SQLite"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except BusinessError:
            raise
        except sqlite3.Error as e:
            logging.getLogger(func.__module__).error(
                f"Erreur SQLite dans {func.__name__}: {e}"
            )
            raise BusinessError("Erreur lors de l'opération")
        except Exception as e:
            logging.getLogger(func.__module__).error(
                f"Erreur inattendue dans {func.__name__}: {e}"
            )
            raise BusinessError("Erreur système inattendue")
    return wrapper

# === EXEMPLE D'UTILISATION ===

class UserService:
    """Service utilisateur avec gestion d'erreurs"""

    def __init__(self, db_path: str):
        self.db = SQLiteManager(db_path)

    @handle_sqlite_errors
    def create_user(self, email: str, name: str) -> int:
        if not email or '@' not in email:
            raise ValidationError("Email invalide")
        if not name or len(name.strip()) < 2:
            raise ValidationError("Nom trop court")
        # execute_safe retourne une liste de rows (pas un curseur).
        rows = self.db.execute_safe(
            "INSERT INTO users (email, name) VALUES (?, ?) RETURNING id",
            (email.lower().strip(), name.strip())
        )
        return rows[0][0]   # première ligne, première colonne

    @handle_sqlite_errors
    def get_user(self, user_id: int) -> dict:
        rows = self.db.execute_safe(
            "SELECT id, email, name FROM users WHERE id = ?", (user_id,)
        )
        if not rows:
            raise NotFoundError(f"Utilisateur {user_id} non trouvé")
        row = rows[0]
        return {'id': row[0], 'email': row[1], 'name': row[2]}

# Configuration du logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

#### Fichier 2 : `example_usage.py`

```python
"""Exemple d'utilisation du kit de démarrage."""

import sqlite3  
from sqlite_errors import (  
    UserService, ValidationError, NotFoundError, ConflictError, BusinessError
)

def demo_usage():
    """Démonstration de l'utilisation du kit."""
    # Créer la base de test (schéma en une ligne pour éviter `'''` imbriqués)
    with sqlite3.connect('test_kit.db') as conn:
        conn.execute(
            "CREATE TABLE IF NOT EXISTS users ("
            " id INTEGER PRIMARY KEY,"
            " email TEXT UNIQUE NOT NULL,"
            " name TEXT NOT NULL)"
        )

    service = UserService('test_kit.db')

    try:
        user_id = service.create_user("alice@test.com", "Alice")
        print(f"✅ Utilisateur créé avec ID: {user_id}")

        user = service.get_user(user_id)
        print(f"✅ Utilisateur récupéré: {user['name']}")

        # Tenter de créer un doublon (lève ConflictError)
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
```

#### Fichier 3 : `tests_errors.py`

```python
"""Tests pytest pour la gestion d'erreurs."""

import os  
import sqlite3  
import tempfile  
import pytest  
from sqlite_errors import (  
    UserService, ValidationError, NotFoundError, ConflictError
)

class TestErrorHandling:
    """Tests de gestion d'erreurs."""

    def setup_method(self):
        self.temp_db = tempfile.mktemp(suffix='.db')
        with sqlite3.connect(self.temp_db) as conn:
            conn.execute(
                "CREATE TABLE users ("
                " id INTEGER PRIMARY KEY,"
                " email TEXT UNIQUE NOT NULL,"
                " name TEXT NOT NULL)"
            )
        self.service = UserService(self.temp_db)

    def teardown_method(self):
        if os.path.exists(self.temp_db):
            os.unlink(self.temp_db)

    def test_validation_email_invalide(self):
        with pytest.raises(ValidationError, match="Email invalide"):
            self.service.create_user("email_invalide", "Test User")

    def test_validation_nom_trop_court(self):
        with pytest.raises(ValidationError, match="Nom trop court"):
            self.service.create_user("test@email.com", "A")

    def test_conflit_email_unique(self):
        self.service.create_user("test@email.com", "User 1")
        with pytest.raises(ConflictError, match="existe déjà"):
            self.service.create_user("test@email.com", "User 2")

    def test_utilisateur_non_trouve(self):
        with pytest.raises(NotFoundError, match="non trouvé"):
            self.service.get_user(999)

    def test_creation_et_lecture_normale(self):
        user_id = self.service.create_user("test@email.com", "Test User")
        user = self.service.get_user(user_id)
        assert user['email'] == "test@email.com"
        assert user['name'] == "Test User"

# Lancer : `pytest tests_errors.py -v`
```

> 🚀 **Mode d'emploi** :  
> 1. Copiez les 3 fichiers dans votre projet (au même niveau).  
> 2. Adaptez selon vos besoins spécifiques (schéma, validations, messages).  
> 3. Lancez `pytest tests_errors.py -v` pour vérifier que tout fonctionne.  
> 4. Intégrez progressivement le pattern dans votre code existant.

## Tests de la gestion d'erreurs (résumé)

Pytest est idéal pour tester votre gestion d'erreurs. Le kit de démarrage présenté plus haut inclut déjà un fichier `tests_errors.py` avec :
- Tests des validations (`ValidationError` pour email invalide, nom trop court)
- Tests des conflits (`ConflictError` pour doublon UNIQUE)
- Tests des cas non trouvés (`NotFoundError` pour ID inexistant)
- Tests du cas normal (création + lecture)

## Tests d'intégration (résumé)

Pour valider votre gestion d'erreurs en conditions réelles :
- Tester un **scénario e-commerce complet** : commande avec stock insuffisant, produit inexistant, paiement échoué.
- Tester une **migration de données** avec validation, doublons, données invalides.
- Tester sous **charge concurrente** : `ThreadPoolExecutor` + base sur disque (`:memory:` n'est PAS partagée).

Le pattern : créer une base **sur fichier temporaire** au setup, et la nettoyer en teardown.

## Conclusion (résumé)

Cette section vous a appris à transformer la gestion d'erreurs SQLite en un véritable atout pour vos applications :

🔍 **Diagnostic et détection** : identifier les types d'erreurs, comprendre la hiérarchie.

🛡️ **Prévention** : valider les données avant insertion, prévenir l'injection SQL.

🔧 **Gestion robuste** : try/except spécialisés, exceptions métier, retry avec backoff.

📊 **Observabilité** : logger avec contexte, monitorer en production (Sentry, Prometheus).

🏗️ **Architecture** : pattern Repository, exceptions métier personnalisées, tests des cas d'erreur.

👥 **UX** : messages clairs et actions de récupération suggérées.


## Transition vers la section 6.6

Vous maîtrisez maintenant la gestion d'erreurs SQLite : hiérarchie d'exceptions, patterns Repository, retry avec backoff exponentiel, monitoring. La section suivante explore la **recherche plein texte** avec FTS5 — un sujet plus algorithmique mais tout aussi essentiel pour les applications réelles.

⏭️ [6.6 Implémenter la recherche plein texte avec FTS5](/06-programmation-avancee-sqlite/06-recherche-plein-texte-avec-fts5.md)
