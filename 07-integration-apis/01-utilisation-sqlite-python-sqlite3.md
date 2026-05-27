🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.1 : Utilisation de SQLite avec Python (sqlite3)

## Introduction

Python et SQLite forment un duo parfait pour les débutants en programmation et base de données. Le module `sqlite3` est inclus par défaut dans Python, ce qui signifie qu'aucune installation supplémentaire n'est nécessaire. Cette section vous guidera pas à pas dans l'utilisation de SQLite avec Python, en partant des concepts de base jusqu'aux techniques avancées.

## Pourquoi Python et SQLite ?

**Simplicité** : Pas de serveur à configurer, pas de mots de passe complexes à retenir. Votre base de données est un simple fichier sur votre ordinateur.

**Intégration native** : Le module `sqlite3` est déjà installé avec Python. Vous pouvez commencer immédiatement.

**Apprentissage idéal** : Parfait pour apprendre les concepts de base de données sans la complexité des SGBD serveur.

**Portabilité** : Votre code et votre base de données fonctionnent sur Windows, Mac et Linux.

## Premiers pas : Connexion et création

### Importer le module

```python
import sqlite3
```

### Créer ou se connecter à une base de données

```python
# Se connecter à une base de données (la crée si elle n'existe pas)
connexion = sqlite3.connect('ma_base.db')

# Créer un curseur pour exécuter des commandes SQL
curseur = connexion.cursor()

print("Connexion réussie à la base de données !")
```

**💡 Conseil pour débutants** : Le fichier `ma_base.db` est créé dans le **répertoire courant d'exécution** (celui depuis lequel vous lancez le script, pas forcément celui qui contient le `.py` — vérifiez avec `os.getcwd()` en cas de doute). Pour fixer l'emplacement, utilisez un **chemin absolu** : `sqlite3.connect('/chemin/absolu/ma_base.db')` ou `sqlite3.connect(Path(__file__).parent / 'ma_base.db')`.

### Créer votre première table

```python
# Créer une table simple pour stocker des informations sur des personnes
curseur.execute('''
    CREATE TABLE IF NOT EXISTS personnes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nom TEXT NOT NULL,
        age INTEGER,
        email TEXT UNIQUE
    )
''')

print("Table 'personnes' créée avec succès !")
```

**Explication ligne par ligne** :
- `CREATE TABLE IF NOT EXISTS` : Crée la table seulement si elle n'existe pas déjà
- `id INTEGER PRIMARY KEY AUTOINCREMENT` : Colonne d'identifiant unique qui s'incrémente automatiquement
- `nom TEXT NOT NULL` : Colonne texte obligatoire (ne peut pas être vide)
- `age INTEGER` : Colonne pour stocker un nombre entier (peut être vide)
- `email TEXT UNIQUE` : Colonne texte avec contrainte d'unicité

> 💡 **`AUTOINCREMENT` vs `INTEGER PRIMARY KEY` seul** : en SQLite, `INTEGER PRIMARY KEY` est déjà un alias du `ROWID` interne et s'auto-incrémente sans le mot-clé `AUTOINCREMENT`. La différence : sans `AUTOINCREMENT`, SQLite peut réutiliser les IDs des lignes supprimées (rarement gênant) ; avec `AUTOINCREMENT`, il garantit la stricte monotonie au prix d'une table système supplémentaire (`sqlite_sequence`) et d'un léger surcoût en performance. **Recommandation** : n'utilisez `AUTOINCREMENT` que si vous tenez à interdire la réutilisation d'IDs (par exemple pour des audits).

## Opérations CRUD (Create, Read, Update, Delete)

### Create - Insérer des données

#### Insérer une seule ligne

```python
# Méthode sécurisée avec des paramètres (recommandée)
curseur.execute("""
    INSERT INTO personnes (nom, age, email)
    VALUES (?, ?, ?)
""", ("Alice Martin", 25, "alice.martin@email.com"))

print("Personne ajoutée avec succès !")
```

**⚠️ Important** : Utilisez toujours les paramètres `?` plutôt que de concaténer des chaînes. Cela protège contre les injections SQL.

> 💡 **Récupérer l'ID de la ligne insérée** : après un `INSERT`, utilisez `curseur.lastrowid` pour obtenir l'ID auto-incrémenté généré. Pratique pour chaîner des INSERT dans des tables liées.  
>
> ```python
> curseur.execute("INSERT INTO personnes (nom, age) VALUES (?, ?)", ("Eve", 22))
> nouvel_id = curseur.lastrowid
> print(f"Eve a l'ID : {nouvel_id}")
> ```
>  
> **Depuis SQLite 3.35** (mars 2021), vous pouvez aussi utiliser `RETURNING` :  
>
> ```python
> curseur.execute(
>     "INSERT INTO personnes (nom, age) VALUES (?, ?) RETURNING id, nom",
>     ("Hugo", 28)
> )
> id_hugo, nom_hugo = curseur.fetchone()
> connexion.commit()  # ⚠️ fetchone() AVANT commit() — sinon retourne None
> ```
>  
> Vérifiez la version embarquée par votre Python avec `sqlite3.sqlite_version` (Python 3.10+ inclut généralement SQLite ≥ 3.35).

#### Insérer plusieurs lignes en une fois

```python
# Données à insérer
nouvelles_personnes = [
    ("Bob Dupont", 30, "bob.dupont@email.com"),
    ("Claire Rousseau", 28, "claire.rousseau@email.com"),
    ("David Moreau", 35, "david.moreau@email.com")
]

# Insertion en lot
curseur.executemany("""
    INSERT INTO personnes (nom, age, email)
    VALUES (?, ?, ?)
""", nouvelles_personnes)

print(f"{len(nouvelles_personnes)} personnes ajoutées !")
```

### Read - Lire des données

#### Lire toutes les données

```python
# Exécuter une requête de sélection
curseur.execute("SELECT * FROM personnes")

# Récupérer tous les résultats
toutes_personnes = curseur.fetchall()

print("Liste de toutes les personnes :")  
for personne in toutes_personnes:  
    id_personne, nom, age, email = personne
    print(f"ID: {id_personne}, Nom: {nom}, Âge: {age}, Email: {email}")
```

#### Lire avec des conditions

```python
# Rechercher les personnes de plus de 30 ans
curseur.execute("SELECT nom, age FROM personnes WHERE age > ?", (30,))

personnes_agees = curseur.fetchall()  
print("\nPersonnes de plus de 30 ans :")  
for nom, age in personnes_agees:  
    print(f"{nom} - {age} ans")
```

#### Utiliser fetchone() pour récupérer une ligne à la fois

```python
curseur.execute("SELECT * FROM personnes")

print("\nPersonnes une par une :")  
while True:  
    personne = curseur.fetchone()
    if personne is None:
        break
    print(f"Nom: {personne[1]}, Email: {personne[3]}")
```

> 💡 **Plus idiomatique : itérer directement sur le curseur** — le curseur est un itérateur Python, vous pouvez donc l'utiliser dans une `for` sans appeler `fetchone()` :  
>
> ```python
> curseur.execute("SELECT * FROM personnes")
> for personne in curseur:           # itération paresseuse (streaming)
>     print(f"Nom: {personne[1]}")
> ```
>  
> **Avantage mémoire** : contrairement à `fetchall()` qui charge tout en RAM, l'itération est paresseuse et reste efficace même sur des millions de lignes.

### Update - Mettre à jour des données

```python
# Mettre à jour l'âge d'une personne spécifique
nouvel_age = 31  
email_recherche = "bob.dupont@email.com"  

curseur.execute("""
    UPDATE personnes
    SET age = ?
    WHERE email = ?
""", (nouvel_age, email_recherche))

# Vérifier combien de lignes ont été modifiées
lignes_modifiees = curseur.rowcount  
print(f"{lignes_modifiees} ligne(s) mise(s) à jour")  
```

### Delete - Supprimer des données

```python
# Supprimer une personne par son email
curseur.execute("DELETE FROM personnes WHERE email = ?", ("claire.rousseau@email.com",))

lignes_supprimees = curseur.rowcount  
print(f"{lignes_supprimees} personne(s) supprimée(s)")  
```

## Gestion des transactions

Les transactions permettent de grouper plusieurs opérations et de les annuler si quelque chose se passe mal.

```python
try:
    # Plusieurs opérations dans une transaction
    curseur.execute("INSERT INTO personnes (nom, age, email) VALUES (?, ?, ?)",
                   ("Nouvelle Personne", 40, "nouvelle@email.com"))

    curseur.execute("UPDATE personnes SET age = age + 1 WHERE age < 30")

    # Confirmer toutes les modifications
    connexion.commit()
    print("Transaction réussie !")

except sqlite3.Error as e:
    # Annuler toutes les modifications en cas d'erreur
    connexion.rollback()
    print(f"Erreur lors de la transaction : {e}")
```

> ⚠️ **Piège du module `sqlite3` en Python** : par défaut, `sqlite3.connect(...)` utilise `isolation_level=""` (mode "transparent transactions") — le module **ouvre automatiquement une transaction implicite** avant le premier DML (`INSERT`/`UPDATE`/`DELETE`) et la commit (selon Python ≥ 3.6) à la prochaine instruction non-DML. Conséquence :  
>  
> - **Pas besoin** de `BEGIN` explicite — la transaction démarre toute seule au premier DML.  
> - Si vous écrivez `connexion.execute("BEGIN")` alors qu'une transaction est déjà ouverte, vous obtiendrez `OperationalError: cannot start a transaction within a transaction`.  
> - Pour reprendre le contrôle total (BEGIN/COMMIT manuels), passez `sqlite3.connect('ma_base.db', isolation_level=None)` — c'est le mode "autocommit" du module ; vos `BEGIN IMMEDIATE` / `COMMIT` deviennent alors explicites et fiables.  
>  
> **Python 3.12+** introduit un nouveau paramètre `autocommit=False` (LEGACY) ou `autocommit=True` (PEP 249) plus prévisible — préférez-le sur les versions récentes.

## Gestion des erreurs courantes

### Erreur de contrainte d'unicité

```python
try:
    curseur.execute("""
        INSERT INTO personnes (nom, age, email)
        VALUES (?, ?, ?)
    """, ("Test", 25, "alice.martin@email.com"))  # Email déjà existant

except sqlite3.IntegrityError as e:
    print(f"Erreur d'intégrité : {e}")
    print("Cet email existe déjà dans la base de données")
```

### Vérifier si une table existe

```python
def table_existe(nom_table):
    curseur.execute("""
        SELECT name FROM sqlite_master
        WHERE type='table' AND name=?
    """, (nom_table,))
    return curseur.fetchone() is not None

if table_existe("personnes"):
    print("La table 'personnes' existe")
else:
    print("La table 'personnes' n'existe pas")
```

## Techniques avancées pour débutants

### Utiliser Row Factory pour des résultats plus lisibles

```python
# Configurer la connexion pour retourner des dictionnaires
connexion.row_factory = sqlite3.Row

curseur = connexion.cursor()  
curseur.execute("SELECT * FROM personnes LIMIT 2")  

personnes = curseur.fetchall()  
for personne in personnes:  
    # Accès par nom de colonne
    print(f"Nom: {personne['nom']}, Âge: {personne['age']}")

    # Ou conversion en dictionnaire
    dict_personne = dict(personne)
    print(dict_personne)
```

### Créer une fonction utilitaire simple

```python
def ajouter_personne(nom, age, email):
    """Fonction helper pour ajouter une personne"""
    try:
        curseur.execute("""
            INSERT INTO personnes (nom, age, email)
            VALUES (?, ?, ?)
        """, (nom, age, email))
        connexion.commit()
        print(f"✅ {nom} ajouté(e) avec succès")
        return True
    except sqlite3.IntegrityError:
        print(f"❌ Erreur : L'email {email} existe déjà")
        return False

# Utilisation
ajouter_personne("Emma Leroy", 27, "emma.leroy@email.com")
```

### Types Python ↔ SQLite et dates

Le module `sqlite3` mappe automatiquement les types simples : `int` ↔ INTEGER, `float` ↔ REAL, `str` ↔ TEXT, `bytes` ↔ BLOB, `None` ↔ NULL. Mais pour les **dates et datetimes**, l'aller-retour nécessite un peu plus d'attention :

```python
import sqlite3  
from datetime import date, datetime  

# Sans configuration : Python écrit la date au format ISO 8601,
# mais en lecture on récupère une simple chaîne (pas un objet date).
conn = sqlite3.connect('test.db')  
conn.execute("CREATE TABLE evt (jour DATE, instant TIMESTAMP)")  
conn.execute(  
    "INSERT INTO evt VALUES (?, ?)",
    (date(2026, 5, 27), datetime(2026, 5, 27, 14, 30, 0))
)
row = conn.execute("SELECT jour, instant FROM evt").fetchone()  
print(type(row[0]), row[0])   # <class 'str'> '2026-05-27'  ← chaîne, pas date !  
```

Pour récupérer des objets Python natifs, activez la **détection de type** :

```python
# detect_types=PARSE_DECLTYPES : conversion basée sur le type DÉCLARÉ
#                                dans le CREATE TABLE (ex. 'DATE', 'TIMESTAMP')
# detect_types=PARSE_COLNAMES   : basé sur le nom de colonne (`AS "x [date]"`)
conn = sqlite3.connect('test.db', detect_types=sqlite3.PARSE_DECLTYPES)  
row = conn.execute("SELECT jour, instant FROM evt").fetchone()  
print(type(row[0]), row[0])   # <class 'datetime.date'> 2026-05-27 ✅  
```

> ⚠️ **Python 3.12+ — adapters/converters dépréciés par défaut** : depuis Python 3.12, les adaptateurs implicites pour `datetime.date`/`datetime.datetime` émettent un `DeprecationWarning` et seront retirés dans une version future. La pratique recommandée en 2026 :  
> - **Stocker les dates en TEXT au format ISO 8601** (`datetime.isoformat()`) et reconvertir explicitement (`datetime.fromisoformat()`).  
> - **Ou enregistrer vos propres adapters** via `sqlite3.register_adapter()` et `sqlite3.register_converter()` pour garder le mapping automatique.  
>  
> Ne stockez **jamais** des dates au format `str(datetime.now())` (espace entre date et heure) — `fromisoformat()` accepte les deux formats depuis Python 3.11, mais l'ISO 8601 strict avec `T` reste plus portable.

### Connexion et threads — `check_same_thread`

Par défaut, **une connexion SQLite Python ne peut être utilisée que dans le thread qui l'a créée** :

```python
import sqlite3  
import threading  

conn = sqlite3.connect('test.db')

def worker():
    # ⚠️ Lève : ProgrammingError: SQLite objects created in a thread
    # can only be used in that same thread
    conn.execute("SELECT 1")

threading.Thread(target=worker).start()
```

**Solutions** :
- **Option recommandée** : créer une connexion **par thread** (les fichiers SQLite supportent plusieurs connexions concurrentes via le mode WAL).
- **Option avancée** : `sqlite3.connect('test.db', check_same_thread=False)` désactive le contrôle, mais **vous devez alors gérer vous-même la synchronisation** (verrou Python autour des appels) — sinon corruption garantie.

### PRAGMA essentiels à activer dès l'ouverture

SQLite possède des **PRAGMA** (commandes de configuration) qui modifient le comportement du moteur. Trois sont quasi-incontournables côté application :

```python
import sqlite3

conn = sqlite3.connect('ma_base.db')

# 1️⃣ Activer les clés étrangères — DÉSACTIVÉES par défaut pour compatibilité historique
#    À refaire à CHAQUE nouvelle connexion (le réglage n'est pas persistant).
conn.execute("PRAGMA foreign_keys = ON")

# 2️⃣ Mode WAL (Write-Ahead Logging) : autorise les lectures concurrentes pendant
#    qu'une écriture est en cours. Persistant : à faire UNE fois sur la base.
conn.execute("PRAGMA journal_mode = WAL")

# 3️⃣ Timeout pour les verrous (par défaut : 0 ms → erreur immédiate "database is locked").
#    Avec 5000 ms, SQLite ré-essaie pendant 5 secondes avant d'abandonner.
conn.execute("PRAGMA busy_timeout = 5000")

# Bonus : forcer l'intégrité du schéma (SQLite n'applique pas tous les types par défaut)
conn.execute("PRAGMA trusted_schema = OFF")  # défaut en 3.31+, mais explicite = sûr
```

> 💡 **Astuce — un helper réutilisable** : encapsulez ces PRAGMA dans une factory de connexion pour ne pas les oublier :  
>
> ```python
> def open_db(path):
>     conn = sqlite3.connect(path)
>     conn.execute("PRAGMA foreign_keys = ON")
>     conn.execute("PRAGMA journal_mode = WAL")
>     conn.execute("PRAGMA busy_timeout = 5000")
>     conn.row_factory = sqlite3.Row
>     return conn
> ```

### Base de données en mémoire (`:memory:`)

Pour les **tests unitaires** ou les **traitements éphémères**, SQLite peut tourner entièrement en RAM — pas de fichier, performances maximales :

```python
# Base de données en RAM, détruite à la fermeture de la connexion
conn = sqlite3.connect(':memory:')  
conn.execute("CREATE TABLE tmp (id INTEGER PRIMARY KEY, val TEXT)")  
conn.execute("INSERT INTO tmp (val) VALUES (?)", ("hello",))  
print(conn.execute("SELECT * FROM tmp").fetchone())  # (1, 'hello')  
conn.close()  # tout est perdu, c'est voulu  
```

> ⚠️ **`:memory:` n'est PAS partagé entre connexions** : chaque `sqlite3.connect(':memory:')` crée une base distincte. Si vous voulez partager une base en mémoire entre plusieurs connexions (par exemple un pool de threads), utilisez le **mode URI partagé** :  
>
> ```python
> conn1 = sqlite3.connect('file::memory:?cache=shared', uri=True)
> conn2 = sqlite3.connect('file::memory:?cache=shared', uri=True)
> # conn1 et conn2 voient maintenant la même base en mémoire
> ```

### Types SQLite : 5 classes de stockage, pas plus

Contrairement aux SGBD classiques, **SQLite n'a que 5 classes de stockage** : `NULL`, `INTEGER`, `REAL`, `TEXT`, `BLOB`. Les types comme `VARCHAR(100)`, `TIMESTAMP`, `DATETIME`, `BOOLEAN`, `DECIMAL` sont **acceptés** dans `CREATE TABLE` mais SQLite les mappe à l'une des 5 classes via les **règles d'affinité de type** :

| Type déclaré dans CREATE TABLE | Affinité réelle |
|---|---|
| `INT`, `INTEGER`, `BIGINT`, `TINYINT` | INTEGER |
| `TEXT`, `VARCHAR(N)`, `CHAR(N)`, `CLOB` | TEXT |
| `REAL`, `DOUBLE`, `FLOAT` | REAL |
| `BLOB` ou type absent | BLOB |
| `NUMERIC`, `DECIMAL`, `BOOLEAN`, `DATE`, `DATETIME`, `TIMESTAMP` | NUMERIC (essaie INTEGER/REAL, sinon TEXT) |

**Conséquences pratiques** :
- Une colonne `nom VARCHAR(100)` accepte une chaîne de 1 000 000 caractères — la longueur n'est **pas appliquée**.
- Une colonne `BOOLEAN` stocke en fait 0 ou 1 (INTEGER).
- Une colonne `TIMESTAMP DEFAULT CURRENT_TIMESTAMP` stocke en réalité une chaîne ISO 8601 (`'2026-05-27 14:30:00'`).

> 💡 **Mode STRICT (depuis SQLite 3.37, octobre 2021)** : si vous voulez un typage rigide, ajoutez `STRICT` à la fin du `CREATE TABLE` :  
>
> ```sql
> CREATE TABLE personnes (
>     id   INTEGER PRIMARY KEY,
>     nom  TEXT NOT NULL,
>     age  INTEGER
> ) STRICT;
> -- Insérer 'abc' dans age → datatype mismatch
> ```

### Context Manager (with statement)

Le bloc `with` sur une connexion SQLite gère automatiquement le **commit/rollback** selon le résultat :

```python
def exemple_avec_context_manager():
    with sqlite3.connect('ma_base.db') as conn:
        curseur = conn.cursor()

        curseur.execute("INSERT INTO personnes (nom, age, email) VALUES (?, ?, ?)",
                       ("Frank", 33, "frank@email.com"))

        curseur.execute("SELECT COUNT(*) FROM personnes")
        nombre_personnes = curseur.fetchone()[0]

        print(f"Nombre total de personnes : {nombre_personnes}")

    # ⚠️ À la sortie du `with` :
    #   - Si aucune exception : conn.commit() est appelé automatiquement
    #   - Si une exception : conn.rollback() est appelé automatiquement
    # MAIS la connexion N'EST PAS FERMÉE — il faut appeler conn.close() explicitement
    # (ou utiliser contextlib.closing — voir ci-dessous).
    conn.close()
```

> ⚠️ **PIÈGE FRÉQUENT — souvent mal documenté** : contrairement à un fichier (`with open(...)`), `with sqlite3.connect(...)` **ne ferme pas** la connexion à la sortie du bloc. Il ne fait que **commit/rollback automatique**. La connexion reste ouverte (et le fichier `.db` reste verrouillé) jusqu'à `conn.close()` explicite ou destruction par le garbage collector.

#### Pour fermer automatiquement, utilisez `contextlib.closing`

```python
from contextlib import closing  
import sqlite3  

# Fermeture garantie de la connexion à la sortie du bloc
with closing(sqlite3.connect('ma_base.db')) as conn:
    with conn:  # Imbrication : commit/rollback automatique
        curseur = conn.cursor()
        curseur.execute("INSERT INTO personnes (nom, age) VALUES (?, ?)", ("Gina", 29))
# Ici : commit fait par le `with conn:`, close fait par le `with closing(...)`
```

## Exemple complet : Gestionnaire de contacts simple

```python
import sqlite3  
from datetime import datetime  

class GestionnaireContacts:
    def __init__(self, fichier_db="contacts.db"):
        self.connexion = sqlite3.connect(fichier_db)
        self.connexion.row_factory = sqlite3.Row
        self.curseur = self.connexion.cursor()
        self.creer_table()

    def creer_table(self):
        self.curseur.execute('''
            CREATE TABLE IF NOT EXISTS contacts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nom TEXT NOT NULL,
                telephone TEXT,
                email TEXT UNIQUE,
                date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        self.connexion.commit()

    def ajouter_contact(self, nom, telephone=None, email=None):
        try:
            self.curseur.execute("""
                INSERT INTO contacts (nom, telephone, email)
                VALUES (?, ?, ?)
            """, (nom, telephone, email))
            self.connexion.commit()
            print(f"✅ Contact '{nom}' ajouté")
            return True
        except sqlite3.IntegrityError:
            print(f"❌ L'email {email} existe déjà")
            return False

    def lister_contacts(self):
        self.curseur.execute("SELECT * FROM contacts ORDER BY nom")
        contacts = self.curseur.fetchall()

        if not contacts:
            print("Aucun contact trouvé")
            return

        print("\n📋 Liste des contacts :")
        for contact in contacts:
            print(f"  {contact['nom']} - {contact['telephone']} - {contact['email']}")

    def rechercher_contact(self, terme):
        self.curseur.execute("""
            SELECT * FROM contacts
            WHERE nom LIKE ? OR email LIKE ?
        """, (f"%{terme}%", f"%{terme}%"))

        contacts = self.curseur.fetchall()
        if contacts:
            print(f"\n🔍 Contacts trouvés pour '{terme}' :")
            for contact in contacts:
                print(f"  {contact['nom']} - {contact['email']}")
        else:
            print(f"Aucun contact trouvé pour '{terme}'")

    def fermer(self):
        self.connexion.close()

    # Rendre la classe utilisable avec `with` → fermeture automatique garantie
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.fermer()
        # return False : on ne supprime pas les exceptions éventuelles

# Exemple d'utilisation — TOUT doit être dans le bloc `if __name__ == "__main__":`
# sinon le code s'exécute à chaque import du module.
if __name__ == "__main__":
    # Version classique avec try/finally
    gestionnaire = GestionnaireContacts()
    try:
        gestionnaire.ajouter_contact("Jean Dupont", "01.23.45.67.89", "jean@example.com")
        gestionnaire.ajouter_contact("Marie Martin", "09.87.65.43.21", "marie@example.com")
        gestionnaire.lister_contacts()
        gestionnaire.rechercher_contact("jean")
    finally:
        gestionnaire.fermer()

    # Version recommandée — grâce à __enter__/__exit__ ci-dessus :
    with GestionnaireContacts() as g:
        g.ajouter_contact("Sophie Bernard", "06.12.34.56.78", "sophie@example.com")
        g.lister_contacts()
    # Fermeture garantie ici, même en cas d'exception
```

## Outils Python pratiques à connaître

### `executescript()` : exécuter plusieurs statements d'un coup

Idéal pour appliquer un script SQL complet (création de schéma, migration) :

```python
conn = sqlite3.connect('app.db')  
conn.executescript("""  
    CREATE TABLE IF NOT EXISTS auteurs (
        id INTEGER PRIMARY KEY,
        nom TEXT NOT NULL
    );

    CREATE TABLE IF NOT EXISTS livres (
        id INTEGER PRIMARY KEY,
        titre TEXT NOT NULL,
        auteur_id INTEGER REFERENCES auteurs(id)
    );

    CREATE INDEX IF NOT EXISTS idx_livres_auteur ON livres(auteur_id);
""")
```

> ⚠️ **Particularité importante** : `executescript()` **commit la transaction en cours** avant d'exécuter le script, et **ne crée pas de transaction implicite** autour du script. Si vous voulez l'atomicité, encadrez explicitement avec `BEGIN`/`COMMIT` dans le script lui-même.

### `iterdump()` : exporter la base au format SQL

Pratique pour générer un fichier `.sql` portable (équivalent du `.dump` du shell SQLite) :

```python
conn = sqlite3.connect('app.db')  
with open('backup.sql', 'w', encoding='utf-8') as f:  
    for ligne in conn.iterdump():
        f.write(f"{ligne}\n")
# backup.sql contient : BEGIN TRANSACTION; CREATE TABLE...; INSERT...; COMMIT;
```

Pour restaurer ensuite : `conn.executescript(open('backup.sql').read())`.

### `Connection.backup()` : sauvegarde binaire en ligne

Plus efficace que `iterdump()` pour de grosses bases — copie page par page **sans bloquer les écritures** (voir chapitre 6.4) :

```python
from contextlib import closing

with closing(sqlite3.connect('app.db')) as src, \
     closing(sqlite3.connect('backup.db')) as dst:
    src.backup(
        dst,
        pages=100,  # copie 100 pages puis cède la main aux écrivains
        progress=lambda status, remaining, total:
            print(f"Copié {total - remaining}/{total} pages")
    )
# Les deux connexions sont fermées proprement à la sortie du `with`
```

### `create_function()` : ajouter une fonction SQL en Python

Étendez SQL avec n'importe quelle fonction Python (voir chapitre 6.1 pour les UDF) :

```python
import re

def matches_regex(pattern, value):
    return 1 if re.search(pattern, value or "") else 0

conn = sqlite3.connect('app.db')  
conn.create_function("REGEXP_LIKE", 2, matches_regex, deterministic=True)  

# Maintenant utilisable directement en SQL :
rows = conn.execute(
    "SELECT * FROM personnes WHERE REGEXP_LIKE(?, nom)",
    (r"^[A-D]",)
).fetchall()
```

> 💡 **`deterministic=True` (Python 3.8+)** : indispensable si vous voulez utiliser votre fonction dans un index. Sans cela, SQLite la considère comme non-déterministe et refuse l'indexation.

## Bonnes pratiques pour débutants

### 1. Toujours fermer les connexions

```python
# Méthode classique : try/finally
connexion = sqlite3.connect('ma_base.db')  
try:  
    # Votre code ici
    pass
finally:
    connexion.close()  # Fermeture garantie même en cas d'exception

# Avec un context manager — attention au piège (cf. section précédente) :
with sqlite3.connect('ma_base.db') as connexion:
    # Votre code ici
    pass
# ⚠️ Ici : commit/rollback automatique, MAIS la connexion N'EST PAS fermée
connexion.close()  # Toujours nécessaire en complément

# Méthode recommandée : combiner contextlib.closing + with conn
from contextlib import closing  
with closing(sqlite3.connect('ma_base.db')) as connexion:  
    with connexion:  # commit/rollback automatique
        # Votre code ici
        pass
# Ici : commit ET fermeture garantis
```

### 2. Utiliser des requêtes préparées

```python
# ❌ Dangereux (injection SQL possible)
nom_recherche = "Alice"  
curseur.execute(f"SELECT * FROM personnes WHERE nom = '{nom_recherche}'")  

# ✅ Sécurisé
curseur.execute("SELECT * FROM personnes WHERE nom = ?", (nom_recherche,))
```

### 3. Gérer les erreurs

```python
try:
    curseur.execute("SELECT * FROM table_inexistante")
except sqlite3.OperationalError as e:
    print(f"Erreur SQL : {e}")
except sqlite3.Error as e:
    print(f"Erreur SQLite générale : {e}")
```

## Exercices pratiques

### Exercice 1 : Bibliothèque personnelle
Créez une base de données pour gérer votre collection de livres avec les colonnes : titre, auteur, année_publication, lu (booléen).

### Exercice 2 : Journal de dépenses
Créez une application simple pour enregistrer vos dépenses quotidiennes avec : date, catégorie, montant, description.

### Exercice 3 : Système de notes d'étudiants
Créez deux tables liées : une pour les étudiants et une pour leurs notes, avec calcul automatique de moyenne.

## Ressources supplémentaires

- **Documentation officielle** : [sqlite3 - Python Documentation](https://docs.python.org/3/library/sqlite3.html)
- **SQLite Browser** : Outil graphique pour visualiser vos bases SQLite
- **Commandes utiles** : `.schema`, `.tables`, `.help` dans la console SQLite

## Résumé

Vous avez maintenant les bases solides pour utiliser SQLite avec Python :

✅ **Connexion et création** de bases de données  
✅ **Opérations CRUD** complètes  
✅ **Gestion des erreurs** et des transactions  
✅ **Bonnes pratiques** de sécurité  
✅ **Exemples pratiques** réutilisables

SQLite avec Python est un excellent moyen d'apprendre les concepts de base de données. Une fois ces concepts maîtrisés, vous pourrez facilement passer à d'autres SGBD plus complexes.

⏭️
