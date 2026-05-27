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

1. **SQLCipher** (solution la plus populaire, open-source)
2. **SQLite Multiple Ciphers** (fork actif, supporte plusieurs algorithmes : ChaCha20, AES, RC4-MultiCiphers, etc. — voir [utelle/SQLite3MultipleCiphers](https://github.com/utelle/SQLite3MultipleCiphers))
3. **SQLite Encryption Extension (SEE)** (solution commerciale officielle de SQLite Inc.)
4. **Chiffrement au niveau système de fichiers** (LUKS, fscrypt, BitLocker, FileVault)
5. **Chiffrement applicatif des champs sensibles** (Fernet, AES-GCM par colonne)

> 💡 **Quand choisir quoi ?**  
> - **SQLCipher** : applications mobiles, desktop ; large écosystème, bindings dans tous les langages.  
> - **SQLite Multiple Ciphers** : si besoin de ChaCha20-Poly1305 (mobile/embarqué, moins coûteux que AES sans accélération matérielle) ou de compatibilité avec System.Data.SQLite.  
> - **SEE** : si besoin de support commercial officiel et de la garantie « code identique au SQLite officiel ».  
> - **FS-level** : si pas de contrôle sur le code et qu'un chiffrement global du disque suffit.  
> - **Par champ** : si seuls quelques champs sont sensibles ET qu'il faut conserver l'usage de SQLite standard (ex. partage de la base avec des outils qui ne supportent pas SQLCipher).

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
# Doit afficher quelque chose comme : 4.15.0 community (versions 4.x recommandées,
# dernière stable au moment de la rédaction — avril 2026)
```

> ℹ️ **SQLCipher 3 vs 4 — différences majeures (2018+)** : SQLCipher 4 a renforcé considérablement la sécurité par rapport à la v3 :  
>  
> | Critère | SQLCipher 3 (par défaut) | SQLCipher 4 (par défaut) |  
> |---|---|---|  
> | Algorithme HMAC | HMAC-SHA1 | **HMAC-SHA512** |  
> | Itérations PBKDF2 | 64 000 | **256 000** |  
> | Taille de page | 1 024 octets | **4 096 octets** |  
>  
> **Ouvrir une base SQLCipher 3 avec un binaire SQLCipher 4** nécessite un mode de compatibilité explicite :  
> ```sql  
> PRAGMA key = 'mot_de_passe';  
> PRAGMA cipher_compatibility = 3;   -- ⚠️ avant tout autre PRAGMA  
> -- ou migrer définitivement vers v4 avec sqlcipher_export  
> ```

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

> ⚠️ **`pysqlcipher3` n'est plus activement maintenu** depuis 2021 (problèmes connus sur Python 3.12+). Les alternatives recommandées en 2026 :  
>  
> ```bash  
> # Option 1 (recommandée) : sqlcipher3-binary (wheels pré-compilés)  
> pip install sqlcipher3-binary  
>  
> # Option 2 : sqlcipher3 (compilation locale, nécessite libsqlcipher-dev)  
> pip install sqlcipher3  
>  
> # Option 3 (historique, à éviter sur Python ≥ 3.12) :  
> pip install pysqlcipher3  
> ```  
>  
> L'API reste la même : `from sqlcipher3 import dbapi2 as sqlite` (ou `pysqlcipher3` pour l'ancienne).

#### Code Python basique

```python
# Recommandé : sqlcipher3-binary (maintenu, wheels pré-compilés)
from sqlcipher3 import dbapi2 as sqlite

# Connexion à une base chiffrée
conn = sqlite.connect('ma_base_chiffree.db')

# Définir la clé de chiffrement
# ⚠️ Pour un usage en production, échapper les apostrophes (cf. section
#    "Gestion sécurisée des clés" plus bas) ou passer la clé en hexadécimal.
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
from sqlcipher3 import dbapi2 as sqlite  
import sys  

def ouvrir_base_chiffree(chemin_base, mot_de_passe):
    try:
        conn = sqlite.connect(chemin_base)
        # ⚠️ Échapper les apostrophes pour éviter une injection SQL si le
        #    mot de passe vient d'une source externe (utilisateur, env, etc.)
        mdp_quote = mot_de_passe.replace("'", "''")
        conn.execute(f"PRAGMA key='{mdp_quote}'")

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

#### Sur Linux

> ⚠️ **`eCryptfs` est obsolète** depuis Ubuntu 18.04 (plus officiellement supporté). Préférez les alternatives modernes ci-dessous.

```bash
# ✅ Option moderne 1 : fscrypt (chiffrement natif au niveau dossier, ext4/F2FS)
sudo apt install fscrypt  
sudo fscrypt setup  
fscrypt encrypt /chemin/mon-dossier-chiffré  
# Puis y placer la base SQLite

# ✅ Option moderne 2 : LUKS (chiffrement de volume entier, le plus courant)
sudo cryptsetup luksFormat /dev/sdb1  
sudo cryptsetup open /dev/sdb1 mavolume  
sudo mkfs.ext4 /dev/mapper/mavolume  
sudo mount /dev/mapper/mavolume /mnt/chiffré  

# ✅ Option 3 : gocryptfs (FUSE, ne nécessite pas root, multi-plateforme)
gocryptfs -init /chemin/cipher  
gocryptfs /chemin/cipher /chemin/plain  
```

#### Sur Windows avec BitLocker
- Activer BitLocker sur le disque contenant la base (volume entier)
- Les fichiers sont automatiquement chiffrés/déchiffrés
- Pour un dossier spécifique : **EFS (Encrypting File System)** est aussi disponible

#### Sur macOS avec FileVault
- FileVault chiffre tout le disque système (recommandé par Apple)
- Pour un dossier spécifique : créer une **image disque chiffrée** (`.dmg`) via Utilitaire de disque

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
import os  

class ChiffreurChamps:
    """
    Chiffrement sélectif de champs avec Fernet (AES-128-CBC + HMAC-SHA256).

    ⚠️ BUG fréquent à éviter : NE PAS générer la clé dans __init__() —
    sinon chaque instance a sa propre clé et ne peut PAS déchiffrer les
    données écrites par une autre instance. La clé DOIT être persistée
    (variable d'env, fichier sécurisé, keystore, KMS).

    Pour AES-256-GCM (plus moderne), voir `cryptography.hazmat.primitives.ciphers.aead.AESGCM`.
    """
    def __init__(self, cle: bytes = None):
        # Charger la clé depuis l'environnement OU en argument explicite
        if cle is None:
            cle_env = os.environ.get('FERNET_KEY')
            if cle_env:
                cle = cle_env.encode()
            else:
                raise ValueError(
                    "Aucune clé fournie. Générez-en une UNE fois avec :\n"
                    "  python -c \"from cryptography.fernet import Fernet; "
                    "print(Fernet.generate_key().decode())\"\n"
                    "puis stockez-la dans la variable d'environnement FERNET_KEY."
                )
        self.cle = cle
        self.chiffreur = Fernet(self.cle)

    def chiffrer_texte(self, texte: str) -> str:
        """Chiffre un texte et retourne une chaîne base64"""
        chiffre = self.chiffreur.encrypt(texte.encode('utf-8'))
        # Fernet retourne déjà du base64 url-safe → pas besoin de re-encoder
        return chiffre.decode('utf-8')

    def dechiffrer_texte(self, texte_chiffre: str) -> str:
        """Déchiffre un texte depuis base64"""
        texte_bytes = self.chiffreur.decrypt(texte_chiffre.encode('utf-8'))
        return texte_bytes.decode('utf-8')

# Exemple d'utilisation
# Au premier lancement, générer la clé : os.environ['FERNET_KEY'] = Fernet.generate_key().decode()
# puis sauvegarder cette clé dans un coffre-fort (Vault, AWS KMS, etc.)
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

> ⚠️ **Limitation du chiffrement par champ : pas de recherche indexée**  
>  
> Fernet utilise un IV aléatoire à chaque chiffrement : `chiffrer("toto")` produit un texte différent à chaque appel. Conséquences pratiques :  
>  
> ```sql  
> -- ❌ NE FONCTIONNE PAS : la recherche par email chiffré échouera toujours  
> SELECT * FROM clients WHERE email_chiffre = ?;  -- (chiffrement de l'email recherché)  
> ```  
>  
> **Solutions pour rechercher sur un champ chiffré** :  
>  
> 1. **Hash déterministe en colonne séparée** (recherche par égalité uniquement) :  
>    ```python  
>    import hmac, hashlib  
>    cle_hash = os.environ['HASH_KEY'].encode()  # clé distincte de la clé de chiffrement  
>    email_hash = hmac.new(cle_hash, email.lower().encode(), hashlib.sha256).hexdigest()  
>    # Stocker email_hash en index pour la recherche par égalité  
>    ```  
>    HMAC plutôt que SHA256 simple pour empêcher les attaques par dictionnaire si la base fuite.  
>  
> 2. **Blind index** ou **Encrypted Search** : techniques avancées (CryptDB, Acra). Hors scope pour SQLite simple.  
>  
> 3. **Tout déchiffrer puis filtrer en mémoire** : viable seulement pour de petits jeux de données.  
>  
> 4. **AES-GCM-SIV** (déterministe pour égalité, voir `cryptography.hazmat.primitives.ciphers.aead.AESGCMSIV`) : permet la recherche par égalité tout en gardant un chiffrement authentifié, mais perd la propriété d'IND-CPA en cas de réutilisation de la même clé+nonce sur des plaintexts identiques.

## Considérations de performance

### Impact du chiffrement sur les performances

Le chiffrement a un coût en performance :

```python
import time  
import sqlite3  
# ⚠️ pysqlcipher3 est obsolète depuis 2021 — utiliser sqlcipher3 (binary) :
from sqlcipher3 import dbapi2 as sqlite_chiffre

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
-- ⚠️ Ordre des PRAGMA : `key` doit être défini AVANT tout autre PRAGMA
--    sinon le fichier n'est pas déchiffré et les autres PRAGMA échouent.
PRAGMA key = 'mon_mot_de_passe';

-- Optimiser la taille de page pour le chiffrement (4096 = défaut SQLCipher 4)
PRAGMA cipher_page_size = 4096;

-- Optimiser le cache (10000 pages × page_size = ~40 Mo de cache)
PRAGMA cache_size = 10000;

-- Utiliser WAL mode pour de meilleures performances en lecture concurrente
PRAGMA journal_mode = WAL;
```

> 💡 **PRAGMA spécifiques SQLCipher** (à connaître pour le tuning et le débogage) :  
> - `cipher_version` : affiche la version de SQLCipher (ex. `4.15.0 community`)  
> - `cipher_kdf_iter` : nombre d'itérations PBKDF2 (défaut 256 000 en v4). Réduire = plus rapide à ouvrir mais moins résistant au brute-force.  
> - `cipher_default_kdf_iter` : valeur par défaut au niveau processus  
> - `cipher_compatibility` : 1, 2, 3 ou 4 pour ouvrir des bases anciennes  
> - `cipher_integrity_check` : vérifie l'intégrité de chaque page (lent mais complet)  
> - `cipher_profile = '/tmp/sqlcipher.log'` : trace toutes les opérations crypto (debug uniquement)

### Bindings pour d'autres langages

SQLCipher est disponible dans la plupart des écosystèmes :

| Langage | Package / bibliothèque |
|---|---|
| **C / C++** | Lier avec `libsqlcipher` (remplace `libsqlite3`) |
| **Python** | `sqlcipher3-binary` (recommandé), `sqlcipher3`, `pysqlcipher3` (obsolète) |
| **Node.js** | `@journeyapps/sqlcipher` (drop-in replacement de `sqlite3`) |
| **Java / Android** | [SQLCipher for Android](https://www.zetetic.net/sqlcipher/sqlcipher-for-android/) (intégration officielle) |
| **iOS / Swift** | [SQLCipher for iOS](https://www.zetetic.net/sqlcipher/ios-tutorial/) — souvent via CocoaPods/SPM |
| **.NET / C#** | `Microsoft.Data.Sqlite` + binaire SQLCipher, ou `System.Data.SQLite` avec `HasOfficialMixedMode` |
| **Go** | `github.com/CovenantSQL/go-sqlite3-encrypt` ou compilation manuelle de `mattn/go-sqlite3` avec tags |
| **Rust** | `rusqlite` avec la feature `sqlcipher` activée |

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
import secrets  

# Lire depuis une variable d'environnement
password = os.environ.get('DB_PASSWORD')

# Ou demander à l'utilisateur (sans écho terminal)
if not password:
    password = getpass.getpass("Mot de passe de la base de données: ")

# ⚠️ PIÈGE MAJEUR : PRAGMA n'accepte PAS les paramètres préparés `?` !
#    `conn.execute("PRAGMA key=?", (password,))` → ne fonctionne PAS,
#    la clé est interprétée littéralement comme `?`.
#
# ✅ Solution sûre 1 : échapper les apostrophes via doublement (style SQL)
quoted = password.replace("'", "''")  
conn.execute(f"PRAGMA key = '{quoted}'")  

# ✅ Solution 2 (encore mieux) : passer la clé en hexadécimal — pas de
# problème d'échappement, et la clé brute de 256 bits évite la dérivation
# PBKDF2 (utile pour les performances si on a déjà une clé dérivée d'un KDF).
#
# Formats acceptés par SQLCipher pour PRAGMA key :
#   • x'<64 hex>'  → 32 octets (clé seule) ; le sel vient du début du fichier
#   • x'<96 hex>'  → 32 octets clé + 16 octets sel ; reproductible/portable
#
# La seconde forme évite que SQLCipher dérive le sel du fichier ; utile pour
# rejouer une clé sur une base recréée depuis zéro.
key_hex = secrets.token_hex(32)  # 64 hex chars = 256 bits (à stocker hors-base)  
conn.execute(f"PRAGMA key = \"x'{key_hex}'\"")  
# Le format x'...' demande à SQLCipher d'utiliser directement la clé
# binaire sans passer par PBKDF2 (suppose une clé déjà à haute entropie).

# Effacer la variable Python après utilisation
password = None
```

> ⚠️ **Limite Python sur l'effacement mémoire** : `password = None` ne garantit PAS que la chaîne disparaisse de la RAM (Python ne fournit pas d'effacement sécurisé). Pour des secrets très sensibles, utilisez **`bytearray`** (mutable, effaçable via `for i in range(len(b)): b[i] = 0`) ou des bibliothèques spécialisées comme `pynacl.utils.SecureBytes`.

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
from sqlcipher3 import dbapi2 as sqlite

def sauvegarder_base_chiffree(source, destination, mot_de_passe, mot_de_passe_dest=None):
    """
    Sauvegarde une base chiffrée via `sqlcipher_export()`.

    Args:
        source : chemin de la base SQLCipher source
        destination : chemin du fichier cible (sera créé)
        mot_de_passe : clé de la base source
        mot_de_passe_dest : clé de la base destination
          • None ou ''  → destination NON chiffrée (⚠️ données en clair sur disque)
          • valeur      → destination chiffrée avec cette clé (rotation possible)

    ⚠️ Échapper le mot de passe : si quelqu'un peut fournir un mot de passe
    avec des apostrophes (via interface utilisateur, etc.), il faut les doubler
    pour éviter une injection dans le PRAGMA. Cf. section "Gestion sécurisée des clés".

    💡 sqlcipher_export() VS .backup() : sqlcipher_export() permet de changer
    la clé entre source et destination (cas typique : rotation, ou migration
    SQLCipher 3 → 4). `.backup()` standard conserve la même clé/algo.
    """
    conn_source = sqlite.connect(source)

    # ✅ Échapper les apostrophes (PRAGMA n'accepte pas les paramètres préparés)
    mdp_quote = mot_de_passe.replace("'", "''")
    conn_source.execute(f"PRAGMA key = '{mdp_quote}'")

    # Vérifier que la clé source est correcte AVANT de toucher à la destination
    conn_source.execute("SELECT count(*) FROM sqlite_master").fetchone()

    # Échapper aussi le chemin et la clé de destination
    dest_quote = destination.replace("'", "''")
    if mot_de_passe_dest:
        cle_dest_quote = mot_de_passe_dest.replace("'", "''")
        conn_source.execute(
            f"ATTACH DATABASE '{dest_quote}' AS backup KEY '{cle_dest_quote}'"
        )
    else:
        # KEY '' = destination en clair (utile pour audit, à éviter en prod)
        conn_source.execute(f"ATTACH DATABASE '{dest_quote}' AS backup KEY ''")

    conn_source.execute("SELECT sqlcipher_export('backup')")
    conn_source.execute("DETACH DATABASE backup")
    conn_source.close()

    type_dest = "chiffrée" if mot_de_passe_dest else "EN CLAIR"
    print(f"✅ Sauvegarde {type_dest} créée : {destination}")
```

### Attaques par canaux auxiliaires (side-channels)

Le chiffrement protège le **contenu** des données, mais pas nécessairement leurs **métadonnées**. Un attaquant peut déduire des informations sans déchiffrer :

| Canal auxiliaire | Information leakée | Mitigation |
|---|---|---|
| **Taille du fichier** | Volume approximatif de données | Padding artificiel, ou base de taille fixe (allouer N pages d'avance) |
| **Date/heure de modification (mtime)** | Quand des écritures ont lieu | `touch` périodique, ou WAL constamment actif |
| **Patterns d'accès disque** | Quelles pages sont lues souvent | Cache mémoire applicatif, requêtes batch |
| **Temps de réponse** | Présence d'un enregistrement (timing attack) | `hmac.compare_digest` pour les comparaisons sensibles |
| **Logs / coredumps** | Le mot de passe lui-même | Désactiver les coredumps : `ulimit -c 0`, `prctl(PR_SET_DUMPABLE, 0)` |
| **Mémoire swap** | Clé en RAM écrite sur disque | Verrouiller les pages : `mlock()` (Linux), désactiver le swap |
| **Brute-force hors ligne** | Test de millions de clés en parallèle sur GPU | Itérations PBKDF2 élevées (256 000 défaut v4), Argon2 si disponible |

> 💡 **Exemple concret** : un attaquant qui voit grossir un fichier `audit.db` après chaque connexion bancaire peut en déduire l'activité de l'utilisateur, **sans déchiffrer**. Pour les bases très sensibles, envisagez d'écrire dans un journal de taille fixe rotatif.

### Stockage matériel des clés (HSM / TPM / Secure Enclave)

Stocker la clé maître **dans le code, un fichier ou même une variable d'environnement** reste vulnérable à un dump mémoire ou à une exfiltration du système de fichiers. Pour les environnements critiques :

- **TPM 2.0** (Linux/Windows) : la clé est scellée par le TPM et ne quitte jamais la puce. Outils : `tpm2-tools`, `clevis` (auto-unlock LUKS au boot).
  ```bash
  # Exemple : sceller un secret dans le TPM
  echo -n "ma_cle_sqlcipher" | tpm2_create -C primary.ctx -u sealed.pub -r sealed.priv -i -
  ```
- **Secure Enclave** (macOS/iOS) : via `SecKeychain` / `Keychain Services`. La clé est manipulée par le Secure Enclave Processor (SEP), inaccessible au système.
- **Android Keystore** : `AndroidKeyStore` avec `setUserAuthenticationRequired(true)` — la clé n'est utilisable qu'après authentification biométrique.
- **HSM réseau** (production serveur) : AWS CloudHSM, Azure Key Vault HSM, YubiHSM 2. La clé reste dans le HSM ; on lui envoie les données à (dé)chiffrer.
- **Coffres-forts logiciels** : HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager — compromis : la clé sort du coffre en RAM, mais l'accès est audité et révocable.

```python
# Exemple : récupération d'un secret depuis AWS Secrets Manager
import boto3, json  
client = boto3.client('secretsmanager', region_name='eu-west-1')  
secret = json.loads(client.get_secret_value(SecretId='prod/sqlcipher/master')['SecretString'])  
conn.execute(f"PRAGMA key = \"x'{secret['hex_key']}'\"")  
```

### Rotation périodique des clés

Même chiffrée, une base devrait voir sa clé tournée régulièrement (typiquement tous les 6 à 12 mois, ou immédiatement après le départ d'un administrateur).

```python
# Procédure de rotation sécurisée
def rotation_cle(chemin_base, ancienne_cle, nouvelle_cle):
    """Rotation atomique de la clé SQLCipher."""
    from sqlcipher3 import dbapi2 as sqlite
    conn = sqlite.connect(chemin_base)
    # 1. Ouvrir avec l'ancienne clé
    ancienne_quote = ancienne_cle.replace("'", "''")
    conn.execute(f"PRAGMA key = '{ancienne_quote}'")
    # 2. Vérifier que la base est bien lisible (sinon rekey échouera silencieusement)
    conn.execute("SELECT count(*) FROM sqlite_master").fetchone()
    # 3. Effectuer la rotation
    nouvelle_quote = nouvelle_cle.replace("'", "''")
    conn.execute(f"PRAGMA rekey = '{nouvelle_quote}'")
    conn.close()
    print("✅ Clé tournée. Toutes les pages ont été re-chiffrées.")
```

> ⚠️ **`PRAGMA rekey` n'est PAS atomique sur très grosse base** : il réécrit toutes les pages, ce qui peut prendre plusieurs minutes et nécessiter ~2× l'espace disque. Pour les bases > 1 Go, mieux vaut faire un `sqlcipher_export()` vers une nouvelle base, puis remplacer l'ancienne après vérification.

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
