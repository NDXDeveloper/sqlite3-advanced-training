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

## Mise en pratique : Application Android avec SQLite

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

### Exemple 2 : Code Python pour une app mobile (avec Kivy)

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
        """Recherche dans les notes"""
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                '''SELECT id, titre, contenu, date_creation
                   FROM notes
                   WHERE titre LIKE ? OR contenu LIKE ?
                   ORDER BY date_modification DESC''',
                (f'%{terme_recherche}%', f'%{terme_recherche}%')
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
    def __init__(self, db_path):
        self.db_path = db_path
        # Configuration SQLite pour mobile
        self.init_pragmas()

    def init_pragmas(self):
        """Configure SQLite pour les performances mobiles"""
        with sqlite3.connect(self.db_path) as conn:
            # Réduire l'usage mémoire
            conn.execute("PRAGMA cache_size = -2000")  # 2MB de cache

            # Mode WAL pour de meilleures performances
            conn.execute("PRAGMA journal_mode = WAL")

            # Optimisation pour SSD (smartphones)
            conn.execute("PRAGMA synchronous = NORMAL")

            # Délai avant fermeture des connexions
            conn.execute("PRAGMA wal_autocheckpoint = 1000")
```

### 2. Pagination efficace

```python
def lister_notes_paginees(self, page=1, taille_page=20):
    """Pagination optimisée pour les listes longues"""
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
def sauvegarder_base(self, chemin_sauvegarde):
    """Sauvegarde la base vers un fichier"""
    import shutil

    # Forcer l'écriture du WAL
    with sqlite3.connect(self.db_path) as conn:
        conn.execute("PRAGMA wal_checkpoint(FULL)")

    # Copier le fichier
    shutil.copy2(self.db_path, chemin_sauvegarde)
    print(f"Base sauvegardée vers {chemin_sauvegarde}")

def restaurer_base(self, chemin_sauvegarde):
    """Restaure la base depuis un fichier"""
    import shutil

    # Vérifier que le fichier existe
    if not os.path.exists(chemin_sauvegarde):
        raise FileNotFoundError("Fichier de sauvegarde introuvable")

    # Remplacer la base actuelle
    shutil.copy2(chemin_sauvegarde, self.db_path)
    print("Base restaurée avec succès")
```

## Synchronisation avec un serveur

### Stratégie de synchronisation simple

```python
import requests
import json

class SyncManager:
    def __init__(self, database, server_url):
        self.db = database
        self.server_url = server_url

    def synchroniser_vers_serveur(self):
        """Envoie les modifications locales vers le serveur"""
        # Récupérer les notes non synchronisées
        with sqlite3.connect(self.db.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                "SELECT * FROM notes WHERE synchronise = 0"
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

                if response.status_code == 200:
                    # Marquer comme synchronisée
                    with sqlite3.connect(self.db.db_path) as conn:
                        conn.execute(
                            "UPDATE notes SET synchronise = 1 WHERE id = ?",
                            (note['id'],)
                        )
                    print(f"Note {note['id']} synchronisée")

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
        """Fusionne une note venant du serveur"""
        # Vérifier si la note existe localement
        with sqlite3.connect(self.db.db_path) as conn:
            cursor = conn.execute(
                "SELECT date_modification FROM notes WHERE id = ?",
                (note_serveur['id'],)
            )
            note_locale = cursor.fetchone()

            if note_locale is None:
                # Nouvelle note, l'ajouter
                conn.execute(
                    '''INSERT INTO notes (id, titre, contenu, date_creation,
                                        date_modification, synchronise)
                       VALUES (?, ?, ?, ?, ?, 1)''',
                    (note_serveur['id'], note_serveur['titre'],
                     note_serveur['contenu'], note_serveur['date_creation'],
                     note_serveur['date_modification'])
                )
            else:
                # Comparer les dates de modification
                # (logique de résolution de conflits)
                pass
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

**Astuce :** Ne stockez jamais les images directement dans SQLite, seulement leurs chemins !

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
- **Tester sur vraie device** (pas seulement émulateur)

### ❌ À éviter

- **Bloquer l'UI** avec des requêtes longues
- **Stocker des fichiers** volumineux dans la base
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

⏭️
