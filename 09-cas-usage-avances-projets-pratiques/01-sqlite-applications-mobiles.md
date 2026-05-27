🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.1 : SQLite pour les applications mobiles

## Introduction : Pourquoi SQLite sur mobile ?

SQLite est **la** solution de base de données locale pour les applications mobiles. Que vous développiez pour Android ou iOS, SQLite est déjà intégré dans le système d'exploitation et prêt à être utilisé.

### Avantages de SQLite sur mobile

- **Aucune installation requise** : SQLite est déjà présent sur tous les appareils
- **Fonctionnement hors ligne** : Vos données sont toujours accessibles
- **Performance optimale** : Accès direct aux données sans réseau
- **Stockage efficace** : Un seul fichier compact contient toute votre base
- **Synchronisation possible** : Facilite la sauvegarde vers le cloud

## Concepts fondamentaux pour le mobile

### Stockage local vs distant

```
┌─────────────────┐    ┌─────────────────┐
│   Application   │    │   Serveur Web   │
│     Mobile      │◄──►│                 │
│                 │    │   PostgreSQL    │
│   SQLite local  │    │     MySQL       │
└─────────────────┘    └─────────────────┘
```

**Stratégie hybride recommandée :**
- SQLite pour les données locales et le cache
- Synchronisation périodique avec le serveur
- Fonctionnement offline garanti

### Structure typique d'une app mobile avec SQLite

```
MonApp/
├── database/
│   ├── database_helper.py      # Gestionnaire principal
│   ├── models.py               # Définition des tables
│   └── migrations.py           # Évolutions de schéma
├── sync/
│   └── sync_manager.py         # Synchronisation serveur
└── app.py                      # Application principale
```

## Wrappers natifs recommandés (2026)

> 💡 **En pratique, on accède rarement à SQLite « brut » sur mobile.** Les wrappers natifs apportent : DSL typé, migrations automatiques, génération de code, intégration ORM/Reactive. Les exemples Python ci-dessous restent valides pour comprendre le SQL, mais en production mobile, préférez :

| Plateforme | Wrapper recommandé | Particularités |
|---|---|---|
| **Android (Kotlin/Java)** | [**Room**](https://developer.android.com/training/data-storage/room) (officiel Google) | Annotations `@Entity`, `@Dao`, `@Query` ; intégration Flow/LiveData ; migrations automatiques |
| **iOS (Swift)** | [**GRDB.swift**](https://github.com/groue/GRDB.swift) | Swift-first, type-safe ; alternative à Core Data plus proche du SQL |
| **iOS (Swift)** | **Core Data** (Apple) | Framework historique Apple ; abstraction très haute mais courbe d'apprentissage |
| **Flutter** | [**Drift**](https://drift.simonbinder.eu/) (ex-moor) | DSL type-safe, Streams reactive, multi-plateforme |
| **React Native** | [**WatermelonDB**](https://watermelondb.dev/) ou [**op-sqlite**](https://github.com/OP-Engineering/op-sqlite) | WatermelonDB pour de gros datasets ; op-sqlite plus rapide brut |
| **Multi-plateforme** | [**Realm**](https://www.mongodb.com/docs/realm/) | Pas SQLite, base mobile dédiée — alternative si vous voulez quitter SQLite |

> ⚠️ **Kivy/Python sur mobile** : déployable mais **rare en production**. Si vous suivez les exemples Python ci-dessous, le code est correct côté logique mais l'écosystème (taille d'APK, performance, intégration UI native) limite l'usage réel. Pour de vraies apps : Kotlin/Swift ou cross-platform Flutter/React Native.

## Mise en pratique : Concepts illustrés avec Python

### Exemple 1 : Créer une base de données simple

Imaginons une application de prise de notes. Voici comment structurer la base :

```sql
-- Création de la table des notes
CREATE TABLE notes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    contenu TEXT,
    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
    date_modification DATETIME DEFAULT CURRENT_TIMESTAMP,
    synchronise BOOLEAN DEFAULT 0  -- Pour la sync serveur
);

-- Index pour optimiser les recherches
CREATE INDEX idx_notes_date ON notes(date_modification);  
CREATE INDEX idx_notes_sync ON notes(synchronise);  
```

### Exemple 2 : Implémentation Python (concepts portables aux wrappers natifs)

```python
import sqlite3  
from datetime import datetime  

class NotesDatabase:
    def __init__(self, db_path="notes.db"):
        self.db_path = db_path
        self.init_database()

    def init_database(self):
        """Initialise la base de données"""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute('''
                CREATE TABLE IF NOT EXISTS notes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    titre TEXT NOT NULL,
                    contenu TEXT,
                    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
                    date_modification DATETIME DEFAULT CURRENT_TIMESTAMP,
                    synchronise BOOLEAN DEFAULT 0
                )
            ''')

            # Création des index
            conn.execute('CREATE INDEX IF NOT EXISTS idx_notes_date ON notes(date_modification)')
            conn.execute('CREATE INDEX IF NOT EXISTS idx_notes_sync ON notes(synchronise)')

    def ajouter_note(self, titre, contenu=""):
        """Ajoute une nouvelle note"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.execute(
                "INSERT INTO notes (titre, contenu) VALUES (?, ?)",
                (titre, contenu)
            )
            return cursor.lastrowid

    def modifier_note(self, note_id, titre=None, contenu=None):
        """Modifie une note existante"""
        updates = []
        params = []

        if titre is not None:
            updates.append("titre = ?")
            params.append(titre)

        if contenu is not None:
            updates.append("contenu = ?")
            params.append(contenu)

        if updates:
            updates.append("date_modification = CURRENT_TIMESTAMP")
            updates.append("synchronise = 0")  # Marquer comme non synchronisée
            params.append(note_id)

            with sqlite3.connect(self.db_path) as conn:
                conn.execute(
                    f"UPDATE notes SET {', '.join(updates)} WHERE id = ?",
                    params
                )

    def supprimer_note(self, note_id):
        """Supprime une note"""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("DELETE FROM notes WHERE id = ?", (note_id,))

    def lister_notes(self, limite=50):
        """Récupère les notes récentes"""
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row  # Pour accès par nom de colonne
            cursor = conn.execute(
                '''SELECT id, titre, contenu, date_creation, date_modification
                   FROM notes
                   ORDER BY date_modification DESC
                   LIMIT ?''',
                (limite,)
            )
            return [dict(row) for row in cursor.fetchall()]

    def rechercher_notes(self, terme_recherche):
        """Recherche LIKE dans les notes.

        ⚠️ Échapper les wildcards LIKE (`%` et `_`) saisis par l'utilisateur :
           sinon `chercher %` retourne TOUTES les notes au lieu de chercher
           littéralement le caractère `%`. On utilise `ESCAPE '\\'` pour
           introduire un caractère d'échappement.

        💡 Pour des recherches plein-texte performantes (recherche par mots,
           pertinence BM25, snippets), préférer FTS5 (cf. ch. 6.6) — surtout
           sur mobile où les bases peuvent dépasser plusieurs MB.
        """
        terme_echappe = (
            terme_recherche.replace('\\', '\\\\')
                           .replace('%', '\\%')
                           .replace('_', '\\_')
        )
        pattern = f'%{terme_echappe}%'
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                '''SELECT id, titre, contenu, date_creation
                   FROM notes
                   WHERE titre LIKE ? ESCAPE '\\'
                      OR contenu LIKE ? ESCAPE '\\'
                   ORDER BY date_modification DESC''',
                (pattern, pattern)
            )
            return [dict(row) for row in cursor.fetchall()]

# Exemple d'utilisation
if __name__ == "__main__":
    db = NotesDatabase()

    # Ajouter des notes
    note1_id = db.ajouter_note("Ma première note", "Contenu de la première note")
    note2_id = db.ajouter_note("Liste courses", "Lait, Pain, Œufs")

    # Lister les notes
    notes = db.lister_notes()
    print("Mes notes :")
    for note in notes:
        print(f"- {note['titre']} (ID: {note['id']})")

    # Rechercher
    resultats = db.rechercher_notes("première")
    print(f"\nRecherche 'première' : {len(resultats)} résultat(s)")
```

## Optimisations spécifiques au mobile

### 1. Gestion de la mémoire

```python
class OptimizedDatabase:
    """Configure SQLite pour les performances mobiles.

    ⚠️ PIÈGE CRITIQUE — Persistance des PRAGMA : seuls `journal_mode` (WAL)
       est **persistant au niveau base** (stocké dans le fichier .db). TOUS
       les autres PRAGMA ci-dessous (`cache_size`, `synchronous`,
       `wal_autocheckpoint`, `secure_delete`, `busy_timeout`) sont
       **per-connection** et **perdus à la fermeture**.

       Conséquence : ouvrir une connexion juste pour les définir puis la
       fermer ne sert à RIEN sauf pour WAL. On doit donc :
       1. Activer WAL une fois dans `init_persistent_pragmas()` (à l'init)
       2. Rejouer les pragmas per-connection dans chaque `get_connection()`

    Considérations particulières mobile :
    - **Batterie** : chaque écriture flash consomme. Grouper les écritures
      en transactions, éviter les `fsync()` excessifs (synchronous=NORMAL).
    - **Mémoire** : la RAM mobile est limitée. Cache modeste (2-4 MB).
    - **Stockage flash** : la durée de vie est limitée par le nombre d'écritures.
    """

    def __init__(self, db_path):
        self.db_path = db_path
        self.init_persistent_pragmas()

    def init_persistent_pragmas(self):
        """Active les PRAGMA persistants (= stockés dans le fichier .db).
        Pour cette base, seul `journal_mode = WAL` rentre dans cette catégorie.
        """
        with sqlite3.connect(self.db_path) as conn:
            # Mode WAL : persistant — survit aux fermetures/réouvertures
            conn.execute("PRAGMA journal_mode = WAL")

    def get_connection(self):
        """Retourne une connexion configurée. Rejoue à CHAQUE ouverture les
        pragmas per-connection — sans cela, ils retombent aux défauts SQLite.
        """
        conn = sqlite3.connect(self.db_path)
        # Réduire l'usage mémoire (négatif = Kibibytes, donc -2000 = 2 MiB)
        conn.execute("PRAGMA cache_size = -2000")
        # synchronous=NORMAL (vs FULL) : moins de fsync, économie batterie,
        # safe en mode WAL (les transactions COMMITTED restent atomiques).
        conn.execute("PRAGMA synchronous = NORMAL")
        # Auto-checkpoint quand le WAL atteint 1000 pages (~4 MB)
        conn.execute("PRAGMA wal_autocheckpoint = 1000")
        # Économie de cycles flash : ne pas réécrire les pages supprimées.
        # OFF en clair = données restantes lisibles avec un dump brut →
        # à coupler avec SQLCipher (cf. 8.1) si données sensibles.
        conn.execute("PRAGMA secure_delete = OFF")
        # Busy timeout : attendre 5s avant 'database is locked' (utile si
        # une autre app/thread mobile écrit en même temps).
        conn.execute("PRAGMA busy_timeout = 5000")
        return conn
```

### 2. Pagination efficace

```python
def lister_notes_paginees(self, page=1, taille_page=20):
    """Pagination par OFFSET — simple mais coûteuse sur grosses listes.

    ⚠️ `OFFSET N` parcourt et JETTE les N premières lignes avant LIMIT —
       c'est O(N) par requête, soit O(N²) pour parcourir toutes les pages.
       Pour une app mobile avec 10 000+ notes et scroll infini, préférer
       la pagination par CURSEUR (keyset pagination) :

           WHERE date_modification < ? ORDER BY date_modification DESC LIMIT ?
           -- où `?` est la date_modification de la DERNIÈRE note de la page
           -- précédente (passée par le client).

       Avec un index sur `date_modification`, chaque page est O(log N) et
       le coût ne dépend PAS de la profondeur (page 1 ou page 1000 = même prix).
    """
    offset = (page - 1) * taille_page

    with sqlite3.connect(self.db_path) as conn:
        conn.row_factory = sqlite3.Row

        # Compter le total
        total = conn.execute("SELECT COUNT(*) FROM notes").fetchone()[0]

        # Récupérer la page
        cursor = conn.execute(
            '''SELECT id, titre, substr(contenu, 1, 100) as apercu,
                      date_modification
               FROM notes
               ORDER BY date_modification DESC
               LIMIT ? OFFSET ?''',
            (taille_page, offset)
        )

        notes = [dict(row) for row in cursor.fetchall()]

        return {
            'notes': notes,
            'page': page,
            'total_pages': (total + taille_page - 1) // taille_page,
            'total_notes': total
        }
```

### 3. Sauvegarde et restauration

```python
import os  
from contextlib import closing  
import sqlite3  

def sauvegarder_base(self, chemin_sauvegarde):
    """Sauvegarde sûre via l'API .backup() de SQLite.

    ⚠️ NE JAMAIS utiliser shutil.copy/cp sur une base SQLite ouverte :
       si une transaction est en cours, la copie peut être incohérente
       (corruption). L'API .backup() copie page par page de façon
       transactionnellement sûre, même sur une base active.
    """
    with closing(sqlite3.connect(self.db_path)) as src, \
         closing(sqlite3.connect(chemin_sauvegarde)) as dst:
        src.backup(dst)
    print(f"Base sauvegardée vers {chemin_sauvegarde}")

def restaurer_base(self, chemin_sauvegarde):
    """Restaure la base depuis une sauvegarde.

    ⚠️ Toutes les connexions sur self.db_path DOIVENT être fermées avant
       l'appel, sinon SQLite peut détecter une base ouverte et le remplacement
       provoquera des erreurs sur les connexions existantes.
    """
    if not os.path.exists(chemin_sauvegarde):
        raise FileNotFoundError("Fichier de sauvegarde introuvable")

    # Sur une base cible fermée, .backup() depuis la source est sûr
    with closing(sqlite3.connect(chemin_sauvegarde)) as src, \
         closing(sqlite3.connect(self.db_path)) as dst:
        src.backup(dst)
    print("Base restaurée avec succès")
```

> ⚠️ **Sur Android/iOS** : pour le chiffrement de la base locale (RGPD, données médicales, mots de passe utilisateur), utilisez **SQLCipher** (voir 8.1). Sur Android, la clé peut être stockée dans `AndroidKeyStore` avec `setUserAuthenticationRequired(true)` ; sur iOS, dans le `Keychain` (Secure Enclave).

### Thread-safety sur mobile

> ⚠️ **Piège classique mobile : accès concurrents depuis l'UI thread et les workers**. Sur Android comme iOS, les apps utilisent des threads de fond (coroutines Kotlin, GCD/async-await Swift) pour ne pas bloquer l'UI. SQLite est compilé en mode `threadsafe=1` par défaut, mais :  
>  
> - **Python `sqlite3`** : `check_same_thread=True` par défaut → une connexion ouverte dans le thread A ne peut pas être utilisée dans le thread B. Passer `check_same_thread=False` au `connect()` ET sérialiser les accès soi-même (lock applicatif), OU utiliser une connexion par thread (pattern « connection pool »).  
> - **Room (Android)** : gère le pool de connexions et la sérialisation automatiquement, mais **ne JAMAIS appeler les Dao depuis le main thread** — utilisez `suspend fun` ou `Flow`.  
> - **GRDB (iOS)** : propose `DatabaseQueue` (sériel) et `DatabasePool` (lectures concurrentes + une écriture). Choisir `DatabasePool` quand WAL est activé pour profiter des lectures parallèles.  
> - **WAL activé partout** : indispensable pour permettre des lectures pendant qu'une écriture est en cours. Sans WAL (mode rollback journal), un seul accès à la fois — l'UI freeze quand un worker écrit.

## Synchronisation avec un serveur

> ⚠️ **Limites du « last-write-wins » naïf** : la stratégie ci-dessous utilise `date_modification` Python (heure locale du device) pour résoudre les conflits. Deux devices mal synchronisés sur l'horloge (clock skew) peuvent silencieusement écraser les modifications l'un de l'autre.  
>  
> **Solutions modernes 2024+ pour conflits robustes** :  
> - **Hybrid Logical Clocks (HLC)** : timestamp physique + compteur logique, résiste au clock skew  
> - **CRDTs** (Conflict-free Replicated Data Types) — bibliothèques `Yjs`, `Automerge`, `Loro`  
> - **cr-sqlite** : extension SQLite qui ajoute des CRDT au niveau colonne  
> - **PowerSync / ElectricSQL** : sync managée SQLite ↔ Postgres avec CRDT  
> - **UUIDv7** (RFC 9562, 2024) au lieu d'auto-increment : ID timestamp-based, lexicographiquement ordonnés, pas de collision entre devices

### Stratégie de synchronisation simple (pédagogique, à robustifier en prod)

```python
import requests  
import json  

class SyncManager:
    def __init__(self, database, server_url):
        self.db = database
        self.server_url = server_url

    def synchroniser_vers_serveur(self):
        """Envoie les modifications locales vers le serveur"""
        # Récupérer les notes non synchronisées (projection explicite : on
        # n'envoie PAS la colonne `synchronise` qui est un flag local).
        with sqlite3.connect(self.db.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                """SELECT id, titre, contenu, date_creation, date_modification
                   FROM notes WHERE synchronise = 0"""
            )
            notes_a_sync = [dict(row) for row in cursor.fetchall()]

        for note in notes_a_sync:
            try:
                # Envoyer au serveur
                response = requests.post(
                    f"{self.server_url}/api/notes",
                    json=note,
                    timeout=10
                )

                # 200 (OK) ou 201 (Created) selon convention de l'API serveur
                if response.status_code in (200, 201):
                    # Marquer comme synchronisée
                    with sqlite3.connect(self.db.db_path) as conn:
                        conn.execute(
                            "UPDATE notes SET synchronise = 1 WHERE id = ?",
                            (note['id'],)
                        )
                    print(f"Note {note['id']} synchronisée")
                else:
                    print(f"Note {note['id']} : HTTP {response.status_code} — non marquée")

            except requests.RequestException as e:
                print(f"Erreur sync note {note['id']}: {e}")
                # On continue avec les autres notes

    def synchroniser_depuis_serveur(self):
        """Récupère les nouvelles données du serveur"""
        try:
            response = requests.get(f"{self.server_url}/api/notes/updates")

            if response.status_code == 200:
                notes_serveur = response.json()

                for note_data in notes_serveur:
                    # Logique de fusion des données
                    # (ici simplifié, en réalité plus complexe)
                    self.fusionner_note(note_data)

        except requests.RequestException as e:
            print(f"Erreur lors de la sync serveur: {e}")

    def fusionner_note(self, note_serveur):
        """Fusionne une note venant du serveur (stratégie last-write-wins).

        ⚠️ Stratégie naïve : on garde la version avec `date_modification`
           la plus récente. Vulnérable au clock skew entre devices.
           Pour une résolution robuste, voir l'encadré CRDTs ci-dessus.
        """
        with sqlite3.connect(self.db.db_path) as conn:
            cursor = conn.execute(
                "SELECT date_modification FROM notes WHERE id = ?",
                (note_serveur['id'],)
            )
            row = cursor.fetchone()

            if row is None:
                # Nouvelle note : on l'insère telle quelle, déjà marquée sync.
                # INSERT OR REPLACE pour éviter une race condition si une autre
                # connexion a inséré entre temps.
                conn.execute(
                    '''INSERT OR REPLACE INTO notes
                          (id, titre, contenu, date_creation,
                           date_modification, synchronise)
                       VALUES (?, ?, ?, ?, ?, 1)''',
                    (note_serveur['id'], note_serveur['titre'],
                     note_serveur['contenu'], note_serveur['date_creation'],
                     note_serveur['date_modification'])
                )
            else:
                # Conflit : last-write-wins comparé sur date_modification ISO.
                # Les chaînes ISO 8601 sont comparables lexicographiquement,
                # MAIS il faut que les deux côtés utilisent le MÊME format :
                #   '2026-05-27 12:00:00'        (sans microsecondes)
                #   '2026-05-27 12:00:00.123456' (avec microsecondes)
                # Le second est lexicographiquement > au premier MÊME pour le
                # même instant. Solution : tronquer aux secondes des deux côtés
                # avant comparaison, ou imposer un format unique côté serveur.
                date_locale = row[0]
                date_serveur = note_serveur['date_modification']
                if date_serveur > date_locale:
                    conn.execute(
                        '''UPDATE notes SET titre = ?, contenu = ?,
                               date_modification = ?, synchronise = 1
                           WHERE id = ?''',
                        (note_serveur['titre'], note_serveur['contenu'],
                         date_serveur, note_serveur['id'])
                    )
                # Sinon : la version locale est plus récente, on garde.
```

## Exercices pratiques

### Exercice 1 : Application de contact simple

Créez une application de carnet d'adresses avec :

1. **Table contacts** avec nom, téléphone, email
2. **Fonction d'ajout** de nouveau contact
3. **Recherche** par nom ou téléphone
4. **Tri** par nom alphabétique

**Structure de base :**
```sql
CREATE TABLE contacts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT NOT NULL,
    prenom TEXT,
    telephone TEXT,
    email TEXT,
    date_ajout DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Exercice 2 : Gestion des photos

Étendez l'application de notes pour supporter les photos :

1. **Table photos** liée aux notes
2. **Stockage du chemin** vers le fichier image
3. **Miniatures** pour l'affichage rapide

**Astuce nuancée :** L'idée « ne stockez jamais d'images dans SQLite » est trop catégorique. Le [benchmark officiel SQLite « 35% Faster Than The Filesystem »](https://www.sqlite.org/fasterthanfs.html) montre que **pour des fichiers < 100 KB**, le stockage BLOB en SQLite est **plus rapide** que le système de fichiers et limite la fragmentation. La bonne règle :
- **< 100 KB (miniatures, icônes)** : BLOB en SQLite est OK voire préférable
- **> 1 MB (photos pleine résolution, vidéos)** : fichier séparé + chemin en base
- **Entre les deux** : à benchmarker selon votre profil d'accès

### Exercice 3 : Synchronisation offline

Implémentez un système de synchronisation qui :

1. **Détecte** les changements locaux
2. **Met en file d'attente** les synchronisations
3. **Reprend automatiquement** quand la connexion revient

## Bonnes pratiques pour mobile

### ✅ À faire

- **Utiliser les index** pour les recherches fréquentes
- **Paginer** les grandes listes
- **Gérer les erreurs** de base de données
- **Sauvegarder régulièrement** les données importantes
- **Tester sur un vrai appareil** (pas seulement émulateur)

### ❌ À éviter

- **Bloquer l'UI** avec des requêtes longues
- **Stocker de gros fichiers (>1 MB)** directement dans la base — pour les petits fichiers (<100 KB), le BLOB SQLite est souvent plus rapide que le système de fichiers, cf. note sur les photos plus haut
- **Oublier de fermer** les connexions
- **Ignorer les erreurs** de synchronisation
- **Négliger la sauvegarde** avant mise à jour app

## Outils de développement utiles

### Pour Android
- **Android Studio** : Avec l'inspecteur de base de données
- **ADB** : Pour examiner les fichiers SQLite sur device

### Pour iOS
- **Xcode** : Avec Core Data ou SQLite direct
- **Simulator** : Pour accéder aux fichiers de l'app

### Multi-plateforme
- **DB Browser for SQLite** : Pour examiner vos fichiers .db
- **SQLite Viewer** : Extensions pour navigateurs web

## Conclusion

SQLite est l'épine dorsale du stockage local mobile. Sa simplicité d'usage, combinée à sa robustesse, en fait le choix naturel pour toute application nécessitant une persistance de données.

Dans la prochaine section, nous explorerons comment utiliser SQLite pour l'analyse de données, une autre de ses forces méconnues.

---

**Points clés à retenir :**
- SQLite est intégré nativement dans Android et iOS
- Privilégiez le stockage local avec synchronisation périodique
- Optimisez pour la performance mobile (mémoire, batterie)
- Implémentez une stratégie de sauvegarde robuste
- Testez toujours sur de vrais appareils

⏭️ [9.2 Analyse de données avec SQLite](/09-cas-usage-avances-projets-pratiques/02-analyse-donnees-sqlite.md)
