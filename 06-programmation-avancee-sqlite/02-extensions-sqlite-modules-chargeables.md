🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.2 Extensions SQLite et modules chargeables

## Qu'est-ce qu'une extension SQLite ?

Une **extension SQLite** est un module externe qui étend les capacités de SQLite au-delà de ses fonctionnalités de base. Contrairement aux UDF (fonctions définies par l'utilisateur) qui ajoutent seulement des fonctions, les extensions peuvent ajouter :

- De nouvelles fonctions SQL
- Des types de données personnalisés
- Des algorithmes de tri et de collation
- Des tables virtuelles
- Des mécanismes d'indexation avancés
- Des interfaces vers des systèmes externes

### Analogie simple
Pensez à SQLite comme à un smartphone de base, et aux extensions comme aux applications que vous installez pour ajouter de nouvelles fonctionnalités. Chaque extension apporte ses propres capacités sans modifier le système principal.

## Extensions courantes et populaires

### 1. Extension JSON1
Ajoute un support complet pour les données JSON.

```sql
-- Intégrée AU CŒUR de SQLite depuis la version 3.38 (février 2022).
-- Sur les versions antérieures, il fallait charger l'extension JSON1.
SELECT json_extract('{"nom": "Alice", "age": 30}', '$.nom') as nom;
-- Résultat : Alice

-- Opérateurs raccourcis (SQLite 3.38+) :
SELECT '{"nom": "Alice"}' ->> '$.nom';   -- Alice (texte SQL)  
SELECT '{"nom": "Alice"}' ->  '$.nom';   -- "Alice" (JSON valide)  
```

### 2. Extension FTS (Full-Text Search)
Permet la recherche en texte intégral.

```sql
-- Création d'une table de recherche
CREATE VIRTUAL TABLE documents_fts USING fts5(titre, contenu);

-- Recherche
SELECT * FROM documents_fts WHERE documents_fts MATCH 'python AND sqlite';
```

### 3. Extension R-Tree
Pour les requêtes spatiales et géométriques.

```sql
-- Création d'un index spatial
CREATE VIRTUAL TABLE spatial_index USING rtree(
   id,              -- identifiant unique
   minX, maxX,      -- coordonnées X
   minY, maxY       -- coordonnées Y
);
```

### 4. Fonctions mathématiques
Ajoutent des fonctions trigonométriques, logarithmiques, etc.

```sql
-- ℹ️ Intégrées AU CŒUR de SQLite depuis 3.35 (mars 2021), lorsque le binaire est
--    compilé avec `-DSQLITE_ENABLE_MATH_FUNCTIONS`. C'est le cas des binaires
--    distribués sur les paquets Linux récents et le shell `sqlite3` officiel.
--    Plus besoin d'extension externe pour la plupart des installations.
SELECT sin(3.14159/2), log(10), ln(10), sqrt(16), pi(), pow(2, 10);
-- → 1.0, 1.0, 2.302585..., 4.0, 3.141592..., 1024.0
-- ⚠️ ATTENTION : en SQLite `log(x)` est le logarithme en base 10 (équivalent `log10(x)`).
--    Pour le logarithme NATUREL (base e), utiliser `ln(x)`. C'est l'inverse de Python
--    où `math.log(x)` est le logarithme naturel.

-- Vérifier la disponibilité sur votre binaire :
SELECT count(*) FROM pragma_compile_options  
WHERE compile_options = 'ENABLE_MATH_FUNCTIONS';  
-- 1 = fonctions math disponibles, 0 = à recompiler ou charger via extension
```

## Comment charger une extension

### Méthode 1 : Via l'interface SQLite CLI

```bash
# Démarrer SQLite en ligne de commande
sqlite3 ma_base.db

# Charger une extension
.load ./mon_extension.so

# Ou sous Windows
.load ./mon_extension.dll
```

### Méthode 2 : Via Python

```python
import sqlite3

# Activer le chargement d'extensions
conn = sqlite3.connect('ma_base.db')  
conn.enable_load_extension(True)  

# Charger une extension
conn.load_extension('./mon_extension.so')

# Désactiver pour la sécurité (optionnel)
conn.enable_load_extension(False)
```

### Méthode 3 : Via d'autres langages

```javascript
// Node.js avec better-sqlite3
const Database = require('better-sqlite3');  
const db = new Database('ma_base.db');  

// Charger une extension
db.loadExtension('./mon_extension');
```

## Créer sa première extension simple (en C)

Bien que plus complexe que les UDF Python, créer une extension C reste accessible. Voici un exemple pas à pas :

### Structure de base d'une extension

```c
#include "sqlite3ext.h"
SQLITE_EXTENSION_INIT1

// Fonction qui sera disponible en SQL
static void ma_fonction(sqlite3_context *context, int argc, sqlite3_value **argv)
{
    // Vérifier le nombre d'arguments
    if (argc != 1) {
        sqlite3_result_error(context, "ma_fonction() nécessite exactement un argument", -1);
        return;
    }

    // Récupérer l'argument
    const char *texte = (const char*)sqlite3_value_text(argv[0]);
    if (texte == NULL) {
        sqlite3_result_null(context);
        return;
    }

    // Traitement simple : compter les caractères
    int longueur = strlen(texte);

    // Retourner le résultat
    sqlite3_result_int(context, longueur);
}

// Point d'entrée de l'extension
int sqlite3_extension_init(sqlite3 *db, char **pzErrMsg, const sqlite3_api_routines *pApi)
{
    SQLITE_EXTENSION_INIT2(pApi);

    // Enregistrer la fonction
    sqlite3_create_function(db, "ma_fonction", 1, SQLITE_UTF8, NULL, ma_fonction, NULL, NULL);

    return SQLITE_OK;
}
```

### Compilation de l'extension

```bash
# Linux/Mac
gcc -shared -fPIC -o ma_extension.so ma_extension.c

# Windows avec MinGW
gcc -shared -o ma_extension.dll ma_extension.c
```

## Extension pratique : Fonctions de texte avancées

Créons une extension plus utile avec plusieurs fonctions de manipulation de texte :

```c
#include "sqlite3ext.h"
#include <string.h>
#include <ctype.h>
SQLITE_EXTENSION_INIT1

// Fonction pour inverser une chaîne
static void inverser_texte(sqlite3_context *context, int argc, sqlite3_value **argv)
{
    if (argc != 1) {
        sqlite3_result_error(context, "inverser_texte() nécessite un argument", -1);
        return;
    }

    const char *texte = (const char*)sqlite3_value_text(argv[0]);
    if (!texte) {
        sqlite3_result_null(context);
        return;
    }

    int len = strlen(texte);
    char *resultat = sqlite3_malloc(len + 1);
    if (!resultat) {
        sqlite3_result_error_nomem(context);
        return;
    }

    // Inverser la chaîne
    for (int i = 0; i < len; i++) {
        resultat[i] = texte[len - 1 - i];
    }
    resultat[len] = '\0';

    sqlite3_result_text(context, resultat, len, sqlite3_free);
}

// Fonction pour compter les mots
static void compter_mots(sqlite3_context *context, int argc, sqlite3_value **argv)
{
    if (argc != 1) {
        sqlite3_result_error(context, "compter_mots() nécessite un argument", -1);
        return;
    }

    const char *texte = (const char*)sqlite3_value_text(argv[0]);
    if (!texte) {
        sqlite3_result_int(context, 0);
        return;
    }

    int mots = 0;
    int dans_mot = 0;

    for (int i = 0; texte[i]; i++) {
        if (isspace(texte[i])) {
            dans_mot = 0;
        } else if (!dans_mot) {
            dans_mot = 1;
            mots++;
        }
    }

    sqlite3_result_int(context, mots);
}

// Fonction pour convertir en "title case"
static void title_case(sqlite3_context *context, int argc, sqlite3_value **argv)
{
    if (argc != 1) {
        sqlite3_result_error(context, "title_case() nécessite un argument", -1);
        return;
    }

    const char *texte = (const char*)sqlite3_value_text(argv[0]);
    if (!texte) {
        sqlite3_result_null(context);
        return;
    }

    int len = strlen(texte);
    char *resultat = sqlite3_malloc(len + 1);
    if (!resultat) {
        sqlite3_result_error_nomem(context);
        return;
    }

    int nouveau_mot = 1;
    for (int i = 0; i <= len; i++) {
        if (isspace(texte[i]) || texte[i] == '\0') {
            resultat[i] = texte[i];
            nouveau_mot = 1;
        } else if (nouveau_mot) {
            resultat[i] = toupper(texte[i]);
            nouveau_mot = 0;
        } else {
            resultat[i] = tolower(texte[i]);
        }
    }

    sqlite3_result_text(context, resultat, len, sqlite3_free);
}

// Point d'entrée de l'extension
int sqlite3_extension_init(sqlite3 *db, char **pzErrMsg, const sqlite3_api_routines *pApi)
{
    SQLITE_EXTENSION_INIT2(pApi);

    // Enregistrer toutes nos fonctions
    sqlite3_create_function(db, "inverser_texte", 1, SQLITE_UTF8, NULL, inverser_texte, NULL, NULL);
    sqlite3_create_function(db, "compter_mots", 1, SQLITE_UTF8, NULL, compter_mots, NULL, NULL);
    sqlite3_create_function(db, "title_case", 1, SQLITE_UTF8, NULL, title_case, NULL, NULL);

    return SQLITE_OK;
}
```

### Utilisation de l'extension

```sql
-- Après avoir chargé l'extension
SELECT
    titre,
    inverser_texte(titre) as titre_inverse,
    compter_mots(description) as nb_mots,
    title_case(auteur) as auteur_formate
FROM articles;
```

## Extensions Python avec APSW

Pour les débutants, APSW (Another Python SQLite Wrapper) offre une approche plus accessible :

### Installation

```bash
pip install apsw
```

### Création d'une extension Python

```python
import apsw  
import re  
from datetime import datetime, timedelta  

class MonExtension:
    def __init__(self):
        pass

    def valider_iban(self, iban):
        """Valide un code IBAN"""
        if not iban:
            return False

        # Nettoyer l'IBAN
        iban = re.sub(r'\s+', '', iban.upper())

        # Vérification basique de la longueur
        if len(iban) < 15 or len(iban) > 34:
            return False

        # Algorithme de validation simplifié
        # (Dans un vrai projet, utilisez une bibliothèque spécialisée)
        return re.match(r'^[A-Z]{2}[0-9]{2}[A-Z0-9]+$', iban) is not None

    def calculer_jours_ouvres(self, date_debut, date_fin):
        """Calcule le nombre de jours ouvrés entre deux dates"""
        if not date_debut or not date_fin:
            return None

        try:
            if isinstance(date_debut, str):
                debut = datetime.strptime(date_debut, '%Y-%m-%d')
            else:
                debut = date_debut

            if isinstance(date_fin, str):
                fin = datetime.strptime(date_fin, '%Y-%m-%d')
            else:
                fin = date_fin

            jours_ouvres = 0
            current = debut

            while current <= fin:
                # 0 = Lundi, 6 = Dimanche
                if current.weekday() < 5:  # Lundi à Vendredi
                    jours_ouvres += 1
                current += timedelta(days=1)

            return jours_ouvres

        except (ValueError, TypeError):
            return None

    def formater_numero_compte(self, numero):
        """Formate un numéro de compte bancaire français"""
        if not numero:
            return None

        # Supprimer tous les caractères non numériques
        numero_clean = re.sub(r'\D', '', str(numero))

        if len(numero_clean) != 11:
            return numero  # Retourner tel quel si pas le bon format

        # Format : XXXXX XXXXX XX
        return f"{numero_clean[:5]} {numero_clean[5:10]} {numero_clean[10:]}"

def charger_extension(connection):
    """Charge l'extension dans une connexion APSW"""
    ext = MonExtension()

    # Enregistrer les fonctions
    connection.create_scalar_function("valider_iban", ext.valider_iban)
    connection.create_scalar_function("jours_ouvres", ext.calculer_jours_ouvres)
    connection.create_scalar_function("formater_compte", ext.formater_numero_compte)

# Utilisation
if __name__ == "__main__":
    conn = apsw.Connection("test.db")
    charger_extension(conn)

    # Test des fonctions
    cursor = conn.cursor()

    # Créer une table de test
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS comptes (
            id INTEGER PRIMARY KEY,
            nom TEXT,
            iban TEXT,
            numero_compte TEXT,
            date_ouverture TEXT,
            date_cloture TEXT
        )
    ''')

    # Insérer des données de test
    test_data = [
        ("Alice Martin", "FR1420041010050500013M02606", "12345678901", "2020-01-15", "2024-06-30"),
        ("Bob Durand", "INVALID_IBAN", "98765432109", "2021-03-10", None),
        ("Claire Dubois", "DE89370400440532013000", "11122334455", "2019-05-20", "2024-07-15"),
    ]

    cursor.executemany(
        "INSERT OR REPLACE INTO comptes (nom, iban, numero_compte, date_ouverture, date_cloture) VALUES (?, ?, ?, ?, ?)",
        test_data
    )

    # Utiliser nos fonctions personnalisées
    # ⚠️ En SQL standard, les chaînes utilisent des guillemets SIMPLES.
    #    Les guillemets doubles sont pour les identifiants (noms de colonnes/tables).
    #    SQLite tolère `"..."` comme chaîne pour rétrocompatibilité mais c'est non-standard.
    results = cursor.execute('''
        SELECT
            nom,
            iban,
            CASE
                WHEN valider_iban(iban) THEN '✓ IBAN valide'
                ELSE '✗ IBAN invalide'
            END as statut_iban,
            formater_compte(numero_compte) as compte_formate,
            date_ouverture,
            date_cloture,
            jours_ouvres(date_ouverture, COALESCE(date_cloture, date('now'))) as duree_ouvres
        FROM comptes
    ''').fetchall()

    print("Rapport des comptes :")
    print("-" * 100)
    for row in results:
        print(f"{row[0]:15} | {row[2]:15} | Compte: {row[3]} | Durée: {row[6]} jours ouvrés")

    conn.close()
```

## Tables virtuelles : concept avancé

Les extensions peuvent également créer des **tables virtuelles**, qui permettent d'interfacer SQLite avec des sources de données externes :

### Exemple conceptuel : Table virtuelle CSV

```python
import apsw  
import csv  
import os  

class CSVVirtualTable:
    """Table virtuelle qui lit des fichiers CSV"""

    def __init__(self, fichier_csv):
        self.fichier_csv = fichier_csv
        self.colonnes = []
        self.donnees = []
        self._charger_csv()

    def _charger_csv(self):
        """Charge le fichier CSV en mémoire"""
        if os.path.exists(self.fichier_csv):
            with open(self.fichier_csv, 'r', encoding='utf-8') as f:
                reader = csv.DictReader(f)
                self.colonnes = reader.fieldnames
                self.donnees = list(reader)

    def Create(self, db, modulename, dbname, tablename, *args):
        """Crée la structure de la table virtuelle"""
        # Définir le schéma basé sur les colonnes du CSV
        schema = "CREATE TABLE x("
        for i, col in enumerate(self.colonnes):
            if i > 0:
                schema += ", "
            schema += f"{col} TEXT"
        schema += ")"

        return schema, CSVVirtualCursor(self.donnees)

    Connect = Create  # Même logique pour la connexion

class CSVVirtualCursor:
    """Curseur pour parcourir les données de la table virtuelle"""

    def __init__(self, donnees):
        self.donnees = donnees
        self.position = 0

    def Filter(self, indexnum, indexname, constraintargs):
        """Applique les filtres (WHERE)"""
        self.position = 0

    def Eof(self):
        """Vérifie si on a atteint la fin"""
        return self.position >= len(self.donnees)

    def Rowid(self):
        """Retourne l'ID de la ligne courante"""
        return self.position

    def Column(self, number):
        """Retourne la valeur d'une colonne pour la ligne courante"""
        if self.position < len(self.donnees):
            row = self.donnees[self.position]
            colonnes = list(row.keys())
            if number < len(colonnes):
                return row[colonnes[number]]
        return None

    def Next(self):
        """Passe à la ligne suivante"""
        self.position += 1

# Utilisation (exemple conceptuel - nécessite plus de code pour fonctionner)
# conn.create_module("csv_reader", CSVVirtualTable)
# conn.execute("CREATE VIRTUAL TABLE ma_table USING csv_reader('data.csv')")
```

## Extensions populaires à connaître

### 1. SQLite-Utils (Python)
Outil en ligne de commande avec de nombreuses extensions.

```bash
# Installation
pip install sqlite-utils

# Utilisation avec extensions
sqlite-utils insert ma_base.db table data.csv --csv  
sqlite-utils transform ma_base.db table --type colonne INTEGER  
```

### 2. Extension Spatialite
Pour les données géospatiales.

```sql
-- Après chargement de Spatialite
SELECT ST_Distance(
    ST_GeomFromText('POINT(2.3522 48.8566)'),  -- Paris
    ST_GeomFromText('POINT(2.2945 48.8584)')   -- Arc de Triomphe
);
```

### 3. Extension UUID
Génération d'identifiants uniques.

```sql
-- ⚠️ La fonction uuid() N'EST PAS native dans SQLite — il faut charger une
--    extension externe (par exemple `sqlite-uuid` ou écrire une UDF maison).
--    Solution simple sans extension : générer côté applicatif (Python uuid.uuid4()).

-- Option 1 : extension externe (à compiler/charger)
.load ./sqlite_uuid.so
SELECT uuid() as nouvel_id;
-- → "550e8400-e29b-41d4-a716-446655440000"

-- Option 2 : UDF Python maison (sans extension externe)
-- Côté Python :
--   import uuid
--   conn.create_function("uuid", 0, lambda: str(uuid.uuid4()), deterministic=False)

-- Option 3 : valeur générée par défaut côté SQL pur (sans extension)
--   Utiliser `randomblob(16)` (peu lisible, mais natif) :
CREATE TABLE commandes (
    id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    produit TEXT,
    quantite INTEGER
);
INSERT INTO commandes (produit, quantite) VALUES ('Widget', 5);  
SELECT id FROM commandes;  
-- → ex : "a3f2c89e1d4b7e6f9c2a1b8d5e3f7a92"  (hex aléatoire 32 chars)
```

## Bonnes pratiques pour les extensions

### 1. Sécurité
```python
# ✅ Toujours valider les entrées
def ma_fonction_securisee(valeur):
    if valeur is None:
        return None

    # Validation du type et des limites
    try:
        valeur_float = float(valeur)
        if valeur_float < 0 or valeur_float > 1000000:
            return None
        return valeur_float * 2
    except (TypeError, ValueError):
        return None

# ❌ Éviter les opérations dangereuses
def fonction_dangereuse(chemin_fichier):
    # Ne jamais faire cela !
    with open(chemin_fichier, 'r') as f:  # Risque de sécurité
        return f.read()
```

### 2. Gestion d'erreurs
```c
// En C, toujours vérifier les allocations mémoire
char *resultat = sqlite3_malloc(taille);  
if (!resultat) {  
    sqlite3_result_error_nomem(context);
    return;
}

// Libérer la mémoire correctement
sqlite3_result_text(context, resultat, -1, sqlite3_free);
```

### 3. Performance
```python
# ✅ Cache pour éviter les recalculs
class ExtensionAvecCache:
    def __init__(self):
        self.cache = {}

    def fonction_avec_cache(self, cle):
        if cle in self.cache:
            return self.cache[cle]

        resultat = calcul_couteux(cle)
        self.cache[cle] = resultat
        return resultat

# ❌ Éviter les opérations coûteuses répétées
def fonction_lente(valeur):
    # Recalcule à chaque fois - inefficace !
    return calcul_tres_complexe(valeur)
```

## Débogage des extensions

### Techniques de débogage

```python
import apsw  
import logging  

# Configuration du logging
logging.basicConfig(level=logging.DEBUG)  
logger = logging.getLogger(__name__)  

def ma_fonction_debug(valeur):
    logger.debug(f"Entrée : {valeur}")

    try:
        resultat = traitement(valeur)
        logger.debug(f"Résultat : {resultat}")
        return resultat
    except Exception as e:
        logger.error(f"Erreur dans ma_fonction : {e}")
        return None
```

### Test des extensions

```python
import unittest  
import tempfile  
import os  

class TestMonExtension(unittest.TestCase):
    def setUp(self):
        """Prépare un environnement de test"""
        self.db_file = tempfile.mktemp()
        self.conn = apsw.Connection(self.db_file)
        charger_extension(self.conn)

    def tearDown(self):
        """Nettoie après les tests"""
        self.conn.close()
        if os.path.exists(self.db_file):
            os.unlink(self.db_file)

    def test_valider_iban(self):
        """Test de la fonction de validation IBAN"""
        cursor = self.conn.cursor()

        # Test IBAN valide
        result = cursor.execute("SELECT valider_iban('FR1420041010050500013M02606')").fetchone()
        self.assertTrue(result[0])

        # Test IBAN invalide
        result = cursor.execute("SELECT valider_iban('INVALID')").fetchone()
        self.assertFalse(result[0])

        # Test valeur NULL
        result = cursor.execute("SELECT valider_iban(NULL)").fetchone()
        self.assertFalse(result[0])

if __name__ == '__main__':
    unittest.main()
```

## Déploiement et distribution

### 1. Extensions C/C++
```bash
# Créer un package
mkdir mon_extension_package  
cp mon_extension.so mon_extension_package/  
cp README.md mon_extension_package/  
cp LICENSE mon_extension_package/  

# Archive
tar -czf mon_extension.tar.gz mon_extension_package/
```

### 2. Extensions Python
```python
# setup.py pour distribution
from setuptools import setup

setup(
    name='mon-extension-sqlite',
    version='1.0.0',
    description='Extension SQLite personnalisée',
    py_modules=['mon_extension'],
    install_requires=['apsw'],
    classifiers=[
        'Development Status :: 4 - Beta',
        'Intended Audience :: Developers',
        'License :: OSI Approved :: MIT License',
        'Programming Language :: Python :: 3',
    ],
)
```

## Récapitulatif et points clés

### Ce que vous avez appris
- ✅ **Concept des extensions** : Modules qui étendent SQLite au-delà des UDF
- ✅ **Types d'extensions** : Fonctions, tables virtuelles, types de données
- ✅ **Chargement d'extensions** : Méthodes via CLI, Python, autres langages
- ✅ **Création d'extensions** : En C et en Python avec APSW
- ✅ **Extensions populaires** : JSON, FTS, R-Tree, Spatialite
- ✅ **Bonnes pratiques** : Sécurité, performance, tests

### Avantages des extensions
- **Extensibilité** : Ajout de fonctionnalités complexes
- **Performance** : Traitement optimisé au niveau de la base
- **Réutilisabilité** : Partage entre projets et équipes
- **Modularité** : Chargement à la demande

### Quand utiliser les extensions
- ✅ Fonctionnalités complexes nécessitant plusieurs composants
- ✅ Interfaçage avec des systèmes externes
- ✅ Types de données spécialisés
- ✅ Algorithmes de performance critique

### Quand préférer les UDF simples
- ❌ Fonctions isolées simples
- ❌ Prototypage rapide
- ❌ Logique qui change fréquemment

### Prochaines étapes
Dans la section suivante (6.3), nous explorerons la **gestion avancée des transactions**, incluant les modes de verrouillage, les points de sauvegarde, et les techniques de récupération d'erreur.

⏭️ [6.3 Gestion des transactions et niveaux d'isolation](/06-programmation-avancee-sqlite/03-gestion-transactions-niveaux-isolation.md)
