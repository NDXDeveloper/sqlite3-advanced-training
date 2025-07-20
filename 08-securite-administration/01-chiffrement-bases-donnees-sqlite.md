🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.1 Chiffrement des bases de données SQLite

## Introduction au chiffrement SQLite

Le chiffrement des bases de données SQLite permet de protéger vos données sensibles en les rendant illisibles sans la clé de déchiffrement appropriée. C'est particulièrement important quand votre base de données contient des informations personnelles, financières ou confidentielles.

### Pourquoi chiffrer une base SQLite ?

**Scénarios courants nécessitant le chiffrement :**
- Applications mobiles stockant des données personnelles
- Logiciels desktop avec informations sensibles
- Bases de données contenant des mots de passe ou tokens
- Applications devant respecter des réglementations (RGPD, HIPAA)
- Fichiers de base partagés ou stockés dans le cloud

**Qu'est-ce que le chiffrement protège ?**
- Accès non autorisé au fichier de base
- Vol ou perte de périphérique
- Inspection directe du fichier par des tiers
- Récupération de données sur disques supprimés

## SQLite standard vs SQLite chiffré

### SQLite standard (non chiffré)

SQLite standard ne propose **pas** de chiffrement intégré. Si quelqu'un accède au fichier `.db`, il peut :
- L'ouvrir avec n'importe quel outil SQLite
- Lire toutes les données en clair
- Modifier le contenu facilement

```bash
# Exemple : n'importe qui peut ouvrir votre base
sqlite3 ma_base.db
sqlite> SELECT * FROM utilisateurs;
# Toutes les données sont visibles !
```

### Solutions de chiffrement disponibles

Il existe plusieurs approches pour chiffrer SQLite :

1. **SQLCipher** (solution la plus populaire)
2. **SQLite Encryption Extension (SEE)** (solution commerciale officielle)
3. **Chiffrement au niveau système de fichiers**
4. **Chiffrement applicatif des champs sensibles**

## SQLCipher : La solution open-source

### Qu'est-ce que SQLCipher ?

SQLCipher est une extension open-source de SQLite qui ajoute un chiffrement transparent au niveau du fichier. Il utilise l'algorithme AES-256 pour sécuriser vos données.

**Avantages de SQLCipher :**
- Compatible avec l'API SQLite standard
- Chiffrement AES-256 robuste
- Gratuit et open-source
- Support multi-plateformes
- Large communauté

### Installation de SQLCipher

#### Sur Ubuntu/Debian
```bash
sudo apt-get update
sudo apt-get install sqlcipher libsqlcipher-dev
```

#### Sur Windows
Téléchargez les binaires depuis le site officiel de SQLCipher ou utilisez un gestionnaire de paquets comme vcpkg.

#### Vérification de l'installation
```bash
sqlcipher --version
# Doit afficher quelque chose comme : 3.4.2 community
```

### Utilisation basique de SQLCipher

#### Créer une base chiffrée

```bash
# Lancer SQLCipher
sqlcipher ma_base_chiffree.db

# Définir le mot de passe (OBLIGATOIRE avant toute opération)
sqlite> PRAGMA key = 'mon_mot_de_passe_secret';

# Créer des tables normalement
sqlite> CREATE TABLE utilisateurs (
   id INTEGER PRIMARY KEY,
   nom TEXT NOT NULL,
   email TEXT UNIQUE,
   mot_de_passe TEXT
);

# Insérer des données
sqlite> INSERT INTO utilisateurs (nom, email, mot_de_passe)
        VALUES ('Alice', 'alice@email.com', 'hash_mdp_123');

# Sauvegarder et quitter
sqlite> .quit
```

#### Ouvrir une base chiffrée existante

```bash
sqlcipher ma_base_chiffree.db

# TOUJOURS commencer par définir la clé
sqlite> PRAGMA key = 'mon_mot_de_passe_secret';

# Maintenant vous pouvez faire des requêtes
sqlite> SELECT * FROM utilisateurs;
```

**⚠️ Important :** Si vous oubliez le `PRAGMA key` ou utilisez un mauvais mot de passe, vous obtiendrez des erreurs comme "file is encrypted" ou "database disk image is malformed".

### Gestion des mots de passe

#### Bonnes pratiques pour les mots de passe

```sql
-- Utiliser des mots de passe forts
PRAGMA key = 'unMotDePasseTresComplexeAvecChiffresEtSymboles123!@#';

-- Éviter les mots de passe simples
PRAGMA key = 'password';  -- ❌ Trop simple
PRAGMA key = '123456';    -- ❌ Très faible
```

#### Changer le mot de passe d'une base existante

```sql
-- Ouvrir avec l'ancien mot de passe
PRAGMA key = 'ancien_mot_de_passe';

-- Changer pour un nouveau mot de passe
PRAGMA rekey = 'nouveau_mot_de_passe_plus_fort';

-- La base est maintenant chiffrée avec le nouveau mot de passe
```

#### Supprimer le chiffrement (déchiffrer)

```sql
-- Ouvrir la base chiffrée
PRAGMA key = 'mot_de_passe_actuel';

-- Supprimer le chiffrement
PRAGMA rekey = '';

-- La base n'est plus chiffrée
```

### Utilisation avec Python

#### Installation du module Python

```bash
pip install pysqlcipher3
```

#### Code Python basique

```python
from pysqlcipher3 import dbapi2 as sqlite

# Connexion à une base chiffrée
conn = sqlite.connect('ma_base_chiffree.db')

# Définir la clé de chiffrement
conn.execute("PRAGMA key='mon_mot_de_passe_secret'")

# Utiliser normalement
cursor = conn.cursor()

# Créer une table
cursor.execute('''
    CREATE TABLE IF NOT EXISTS produits (
        id INTEGER PRIMARY KEY,
        nom TEXT NOT NULL,
        prix REAL,
        description TEXT
    )
''')

# Insérer des données
cursor.execute("INSERT INTO produits (nom, prix, description) VALUES (?, ?, ?)",
               ("Laptop", 999.99, "Ordinateur portable haute performance"))

# Lire des données
cursor.execute("SELECT * FROM produits")
resultats = cursor.fetchall()

for produit in resultats:
    print(f"ID: {produit[0]}, Nom: {produit[1]}, Prix: {produit[2]}")

conn.commit()
conn.close()
```

#### Gestion des erreurs avec Python

```python
from pysqlcipher3 import dbapi2 as sqlite
import sys

def ouvrir_base_chiffree(chemin_base, mot_de_passe):
    try:
        conn = sqlite.connect(chemin_base)
        conn.execute(f"PRAGMA key='{mot_de_passe}'")

        # Test simple pour vérifier que le mot de passe est correct
        conn.execute("SELECT name FROM sqlite_master WHERE type='table'")

        print("✅ Base de données ouverte avec succès")
        return conn

    except sqlite.DatabaseError as e:
        print(f"❌ Erreur : Impossible d'ouvrir la base de données")
        print(f"Vérifiez que le mot de passe est correct")
        print(f"Détails : {e}")
        return None

# Utilisation
conn = ouvrir_base_chiffree("ma_base.db", "mon_password")
if conn:
    # Travailler avec la base
    pass
else:
    print("Impossible de continuer sans accès à la base")
```

## Chiffrement au niveau système de fichiers

### Alternative : Chiffrement du système de fichiers

Si vous ne pouvez pas utiliser SQLCipher, vous pouvez chiffrer au niveau du système :

#### Sur Linux avec eCryptfs
```bash
# Créer un répertoire chiffré
sudo mount -t ecryptfs /chemin/source /chemin/destination

# Placer votre base SQLite dans ce répertoire
cp ma_base.db /chemin/destination/
```

#### Sur Windows avec BitLocker
- Activer BitLocker sur le disque contenant la base
- Les fichiers sont automatiquement chiffrés/déchiffrés

#### Avantages et inconvénients

**Avantages :**
- Transparent pour l'application
- Chiffre tous les fichiers du répertoire/disque
- Pas de modification de code nécessaire

**Inconvénients :**
- Protection moins granulaire
- Dépendant du système d'exploitation
- Performances potentiellement impactées

## Chiffrement sélectif des champs

### Chiffrer seulement les données sensibles

Parfois, vous ne voulez chiffrer que certains champs plutôt que toute la base :

```python
from cryptography.fernet import Fernet
import sqlite3
import base64

class ChiffreurChamps:
    def __init__(self):
        # Générer une clé (à faire une seule fois)
        self.cle = Fernet.generate_key()
        self.chiffreur = Fernet(self.cle)

    def chiffrer_texte(self, texte):
        """Chiffre un texte et retourne une chaîne base64"""
        texte_bytes = texte.encode('utf-8')
        chiffre = self.chiffreur.encrypt(texte_bytes)
        return base64.b64encode(chiffre).decode('utf-8')

    def dechiffrer_texte(self, texte_chiffre):
        """Déchiffre un texte depuis base64"""
        chiffre = base64.b64decode(texte_chiffre.encode('utf-8'))
        texte_bytes = self.chiffreur.decrypt(chiffre)
        return texte_bytes.decode('utf-8')

# Exemple d'utilisation
chiffreur = ChiffreurChamps()

# Base SQLite normale (non chiffrée)
conn = sqlite3.connect('base_mixte.db')
cursor = conn.cursor()

cursor.execute('''
    CREATE TABLE IF NOT EXISTS clients (
        id INTEGER PRIMARY KEY,
        nom TEXT,
        email TEXT,
        numero_carte_credit TEXT  -- Ce champ sera chiffré
    )
''')

# Insérer avec chiffrement sélectif
nom = "Jean Dupont"
email = "jean@email.com"
carte = "1234-5678-9012-3456"

# Chiffrer seulement le numéro de carte
carte_chiffree = chiffreur.chiffrer_texte(carte)

cursor.execute("INSERT INTO clients (nom, email, numero_carte_credit) VALUES (?, ?, ?)",
               (nom, email, carte_chiffree))

# Lire et déchiffrer
cursor.execute("SELECT * FROM clients WHERE nom = ?", (nom,))
resultat = cursor.fetchone()

if resultat:
    id_client, nom_lu, email_lu, carte_chiffree_lu = resultat

    # Déchiffrer la carte
    carte_dechiffree = chiffreur.dechiffrer_texte(carte_chiffree_lu)

    print(f"Client: {nom_lu}")
    print(f"Email: {email_lu}")
    print(f"Carte: {carte_dechiffree}")

conn.commit()
conn.close()
```

## Considérations de performance

### Impact du chiffrement sur les performances

Le chiffrement a un coût en performance :

```python
import time
import sqlite3
from pysqlcipher3 import dbapi2 as sqlite_chiffre

def test_performance(nb_insertions=10000):
    # Test base normale
    start = time.time()

    conn_normal = sqlite3.connect(':memory:')
    cursor_normal = conn_normal.cursor()
    cursor_normal.execute('CREATE TABLE test (id INTEGER, data TEXT)')

    for i in range(nb_insertions):
        cursor_normal.execute('INSERT INTO test VALUES (?, ?)', (i, f'donnee_{i}'))

    conn_normal.commit()
    temps_normal = time.time() - start
    conn_normal.close()

    # Test base chiffrée
    start = time.time()

    conn_chiffre = sqlite_chiffre.connect(':memory:')
    conn_chiffre.execute("PRAGMA key='test_password'")
    cursor_chiffre = conn_chiffre.cursor()
    cursor_chiffre.execute('CREATE TABLE test (id INTEGER, data TEXT)')

    for i in range(nb_insertions):
        cursor_chiffre.execute('INSERT INTO test VALUES (?, ?)', (i, f'donnee_{i}'))

    conn_chiffre.commit()
    temps_chiffre = time.time() - start
    conn_chiffre.close()

    print(f"Base normale : {temps_normal:.2f}s")
    print(f"Base chiffrée : {temps_chiffre:.2f}s")
    print(f"Surcoût : {((temps_chiffre/temps_normal - 1) * 100):.1f}%")

# Exécuter le test
test_performance()
```

### Optimisations possibles

```sql
-- Optimiser la taille de page pour le chiffrement
PRAGMA page_size = 4096;

-- Optimiser le cache
PRAGMA cache_size = 10000;

-- Utiliser WAL mode pour de meilleures performances
PRAGMA journal_mode = WAL;
```

## Sécurité et bonnes pratiques

### Gestion sécurisée des clés

**❌ Mauvaises pratiques :**
```python
# Ne JAMAIS faire cela :
password = "motdepasse123"  # En dur dans le code
conn.execute(f"PRAGMA key='{password}'")  # Visible dans les logs
```

**✅ Bonnes pratiques :**
```python
import getpass
import os

# Lire depuis une variable d'environnement
password = os.environ.get('DB_PASSWORD')

# Ou demander à l'utilisateur
if not password:
    password = getpass.getpass("Mot de passe de la base de données: ")

# Utiliser un paramètre sécurisé
conn.execute("PRAGMA key=?", (password,))

# Effacer la variable après utilisation
password = None
```

### Validation de l'intégrité

```sql
-- Vérifier l'intégrité après ouverture
PRAGMA integrity_check;

-- Tester si la clé est correcte
SELECT count(*) FROM sqlite_master;
-- Si erreur, la clé est incorrecte
```

### Sauvegarde des bases chiffrées

```python
def sauvegarder_base_chiffree(source, destination, mot_de_passe):
    """Sauvegarde une base chiffrée vers un fichier non chiffré"""
    conn_source = sqlite.connect(source)
    conn_source.execute(f"PRAGMA key='{mot_de_passe}'")

    # Créer une sauvegarde non chiffrée pour export
    conn_source.execute(f"ATTACH DATABASE '{destination}' AS backup KEY ''")
    conn_source.execute("SELECT sqlcipher_export('backup')")
    conn_source.execute("DETACH DATABASE backup")

    conn_source.close()
    print(f"Sauvegarde créée : {destination}")
```

## Récapitulatif

### Points clés à retenir

1. **SQLite standard n'a pas de chiffrement intégré**
2. **SQLCipher est la solution open-source recommandée**
3. **Toujours définir la clé avant toute opération** : `PRAGMA key`
4. **Gérer les mots de passe de façon sécurisée**
5. **Le chiffrement a un impact sur les performances**
6. **Tester régulièrement l'accès à vos bases chiffrées**

### Checklist de sécurité

- [ ] Mot de passe fort et unique
- [ ] Clé stockée de façon sécurisée (pas en dur)
- [ ] Sauvegarde régulière des bases chiffrées
- [ ] Test de récupération fonctionnel
- [ ] Gestion des erreurs d'accès
- [ ] Documentation des procédures

### Prochaines étapes

Dans la section suivante (8.2), nous verrons comment mettre en place la gestion des permissions et le contrôle d'accès au niveau applicatif pour compléter la sécurité du chiffrement.

---

**💡 Conseil pratique :** Commencez toujours par tester le chiffrement sur des données de test avant de l'appliquer à vos données de production. Assurez-vous d'avoir une stratégie de récupération en cas de perte de mot de passe !

⏭️
