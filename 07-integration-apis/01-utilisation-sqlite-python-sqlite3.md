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

**💡 Conseil pour débutants** : Le fichier `ma_base.db` sera créé dans le même dossier que votre script Python. Si vous ne voyez pas le fichier, vérifiez que votre script s'est exécuté sans erreur.

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

### Update - Mettre à jour des données

```python
# Mettre à jour l'âge d'une personne spécifique
nouveau_age = 31
email_recherche = "bob.dupont@email.com"

curseur.execute("""
    UPDATE personnes
    SET age = ?
    WHERE email = ?
""", (nouveau_age, email_recherche))

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
    # Commencer une transaction
    connexion.execute("BEGIN")

    # Plusieurs opérations
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

### Context Manager (with statement)

La bonne pratique pour gérer automatiquement la fermeture de connexion :

```python
def exemple_avec_context_manager():
    with sqlite3.connect('ma_base.db') as conn:
        curseur = conn.cursor()

        curseur.execute("SELECT COUNT(*) FROM personnes")
        nombre_personnes = curseur.fetchone()[0]

        print(f"Nombre total de personnes : {nombre_personnes}")

    # La connexion se ferme automatiquement ici
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

# Exemple d'utilisation
if __name__ == "__main__":
    # Créer le gestionnaire
    gestionnaire = GestionnaireContacts()

    # Ajouter quelques contacts
    gestionnaire.ajouter_contact("Jean Dupont", "01.23.45.67.89", "jean@example.com")
    gestionnaire.ajouter_contact("Marie Martin", "09.87.65.43.21", "marie@example.com")

    # Lister tous les contacts
    gestionnaire.lister_contacts()

    # Rechercher un contact
    gestionnaire.rechercher_contact("jean")

    # Fermer la connexion
    gestionnaire.fermer()
```

## Bonnes pratiques pour débutants

### 1. Toujours fermer les connexions

```python
# Méthode classique
connexion = sqlite3.connect('ma_base.db')
try:
    # Votre code ici
    pass
finally:
    connexion.close()

# Méthode recommandée (context manager)
with sqlite3.connect('ma_base.db') as connexion:
    # Votre code ici
    pass
# Fermeture automatique
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
