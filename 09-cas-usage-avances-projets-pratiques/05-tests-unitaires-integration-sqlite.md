🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.5 : Tests unitaires et d'intégration avec SQLite

## Introduction : Pourquoi tester avec SQLite ?

Les tests sont **essentiels** pour garantir que votre application fonctionne correctement et continue de fonctionner après chaque modification. SQLite, par sa nature simple et portable, est particulièrement adapté aux tests automatisés.

### Avantages de SQLite pour les tests

- **Tests rapides** : Base en mémoire pour des tests ultra-rapides
- **Isolation parfaite** : Chaque test utilise une base fraîche
- **Pas de configuration** : Aucun serveur à démarrer
- **Reproductibilité** : Même comportement sur tous les environnements
- **Facilité de debug** : Fichiers SQLite faciles à inspecter

### Types de tests que nous allons couvrir

1. **Tests unitaires** : Testent une fonction/méthode isolée
2. **Tests d'intégration** : Testent l'interaction entre composants
3. **Tests de données** : Valident l'intégrité et la cohérence
4. **Tests de performance** : Mesurent les performances des requêtes
5. **Tests de migration** : Vérifient les évolutions de schéma

## Préparation de l'environnement de test

### Installation des outils de test

```bash
# Installation des bibliothèques de test
pip install pytest pytest-flask pytest-cov faker

# Pour les tests de performance
pip install pytest-benchmark

# Pour les mocks et fixtures avancées
pip install pytest-mock factory-boy

# Pour les rapports JSON (utilisés par TestDashboard plus bas)
pip install pytest-json-report
```

### Structure du projet de test

```
tests/
├── __init__.py
├── conftest.py                 # Configuration pytest globale
├── fixtures/
│   ├── __init__.py
│   ├── sample_data.py          # Données de test
│   └── factories.py            # Factory pour créer des objets test
├── unit/
│   ├── __init__.py
│   ├── test_models.py          # Tests des modèles
│   ├── test_database.py        # Tests de la couche base
│   └── test_utils.py           # Tests des utilitaires
├── integration/
│   ├── __init__.py
│   ├── test_api.py             # Tests des APIs
│   ├── test_workflows.py       # Tests de workflows complets
│   └── test_auth.py            # Tests d'authentification
├── performance/
│   ├── __init__.py
│   └── test_performance.py     # Tests de performance
└── data/
    ├── __init__.py
    ├── test_migrations.py       # Tests de migration
    └── test_data_integrity.py   # Tests d'intégrité
```

## Configuration de base pour les tests

### Configuration pytest globale

```python
# tests/conftest.py
import pytest  
import tempfile  
import os  
import sqlite3  
from pathlib import Path  

# Ajouter le dossier parent au path pour les imports
import sys  
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..'))  

from app import create_app  
from app.database import DatabaseManager  
from app.models import Livre, Utilisateur, Emprunt, Genre  

@pytest.fixture(scope='session')
def test_app():
    """Application Flask configurée pour les tests.

    ⚠️ PIÈGE évité : `TestingConfig.DATABASE_PATH = ':memory:'` (cf. config.py)
       crée une base DIFFÉRENTE pour chaque `sqlite3.connect(':memory:')` —
       inutilisable car l'app et la fixture admin_user ne verraient PAS les
       mêmes données. On force ici un fichier temporaire partagé pour la
       session de tests.
    """
    import tempfile
    fd, db_path = tempfile.mkstemp(suffix='_test_app.db')
    os.close(fd)

    app = create_app('testing')
    app.config.update({
        'TESTING': True,
        'WTF_CSRF_ENABLED': False,
        'SECRET_KEY': 'test-secret-key',
        # Override du ':memory:' de TestingConfig par un fichier partageable
        'DATABASE_PATH': db_path,
    })

    # Initialiser le schéma dans le fichier de test
    DatabaseManager(db_path)

    yield app

    # Nettoyage en fin de session
    if os.path.exists(db_path):
        os.unlink(db_path)

@pytest.fixture
def app(test_app):
    """Instance d'application pour chaque test.

    ⚠️ Note : la base est PARTAGÉE entre tous les tests d'une même session.
       Pour une isolation totale par test, utiliser un `fixture function-scoped`
       avec recréation du schéma ; mais c'est plus lent (~50ms/test).
    """
    with test_app.app_context():
        yield test_app

@pytest.fixture
def client(app):
    """Client de test Flask"""
    return app.test_client()

@pytest.fixture
def temp_db():
    """Base de données temporaire pour chaque test.

    ⚠️ Windows : `os.unlink()` peut échouer avec PermissionError si une
       connexion sqlite est encore ouverte sur le fichier. On capture
       l'erreur pour ne pas masquer le test réel (cf. PEP 446).
    """
    fd, path = tempfile.mkstemp(suffix='.db')
    os.close(fd)

    # Initialiser la base
    db_manager = DatabaseManager(path)

    yield path

    # Nettoyage (tolérant aux locks Windows résiduels)
    if os.path.exists(path):
        try:
            os.unlink(path)
        except PermissionError:
            pass  # Le fichier sera nettoyé par le système au reboot

@pytest.fixture
def memory_db():
    """Base de données en mémoire partagée (plus rapide qu'un fichier temp).

    ⚠️ PIÈGE 1 : `sqlite3.connect(':memory:')` crée une base PAR connexion.
       Pour partager entre plusieurs connexions, on utilise une URI
       `'file::memory:?cache=shared'` + `uri=True`.

    ⚠️ PIÈGE 2 : pour que les **modèles** (qui font `sqlite3.connect(self.db_path)`)
       voient cette base, leur `get_connection()` doit aussi utiliser `uri=True`
       quand le path commence par `file:`. À adapter dans BaseModel :
         conn = sqlite3.connect(self.db_path, uri=self.db_path.startswith('file:'))

       Si vous ne pouvez pas modifier BaseModel, utilisez plutôt la fixture
       `temp_db` (fichier temporaire) — plus simple.
    """
    db_path = 'file::memory:?cache=shared'
    # On garde une connexion ouverte pour maintenir la base en vie pendant la fixture
    conn = sqlite3.connect(db_path, uri=True)
    db_manager = DatabaseManager(db_path)
    yield db_path
    conn.close()

@pytest.fixture
def db_with_data(temp_db):
    """Base de données avec des données de test"""
    # Ajouter des données de test
    from tests.fixtures.sample_data import create_sample_data
    create_sample_data(temp_db)
    yield temp_db

@pytest.fixture
def authenticated_user(client, app):
    """Utilisateur connecté pour les tests nécessitant une auth.

    ⚠️ Note : la route /register attend 'password' (form data), mais
    Utilisateur.create() attend 'mot_de_passe'. Vérifier le mapping dans
    routes.register avant d'utiliser cette fixture.
    """
    user_data = {
        'nom': 'Test User',
        'email': 'test@example.com',
        'password': 'testpass123'
    }
    client.post('/register', data=user_data)
    client.post('/login', data={
        'email': user_data['email'],
        'password': user_data['password']
    })
    return user_data

@pytest.fixture
def admin_user(client, app):
    """Utilisateur admin connecté.

    ⚠️ BUG ÉVITÉ : l'ancienne version ouvrait une connexion `:memory:` SÉPARÉE
       et y faisait l'INSERT, qui était donc perdu (chaque `connect(':memory:')`
       crée une base différente). Il faut utiliser la VRAIE base de l'app
       (récupérée via app.config['DATABASE_PATH']).
    """
    from werkzeug.security import generate_password_hash
    db_path = app.config['DATABASE_PATH']

    # Créer l'admin dans la BASE DE L'APP (pas une base ':memory:' séparée)
    with sqlite3.connect(db_path) as conn:
        conn.execute(
            "INSERT INTO utilisateurs (nom, email, mot_de_passe_hash, est_admin) "
            "VALUES (?, ?, ?, ?)",
            ('Admin Test', 'admin@test.com', generate_password_hash('admin123'), 1)
        )
        conn.commit()

    # Se connecter via l'API
    client.post('/login', data={
        'email': 'admin@test.com',
        'password': 'admin123'
    })

    return {'email': 'admin@test.com', 'is_admin': True}
```

### Données de test

```python
# tests/fixtures/sample_data.py
import sqlite3  
from datetime import datetime, timedelta  
from werkzeug.security import generate_password_hash  

def create_sample_data(db_path):
    """Crée un jeu de données de test complet"""

    with sqlite3.connect(db_path) as conn:
        # Genres de test
        genres_data = [
            ('Roman', 'Fiction littéraire', '#e74c3c'),
            ('Science-Fiction', 'Littérature SF', '#9b59b6'),
            ('Informatique', 'Livres techniques', '#2980b9'),
            ('Histoire', 'Livres historiques', '#f39c12')
        ]

        cursor = conn.executemany(
            "INSERT INTO genres (nom, description, couleur) VALUES (?, ?, ?)",
            genres_data
        )

        # Utilisateurs de test
        users_data = [
            ('Alice Martin', 'alice@test.com', generate_password_hash('alice123'), 0),
            ('Bob Dupont', 'bob@test.com', generate_password_hash('bob123'), 0),
            ('Admin Test', 'admin@test.com', generate_password_hash('admin123'), 1)
        ]

        conn.executemany('''
            INSERT INTO utilisateurs (nom, email, mot_de_passe_hash, est_admin)
            VALUES (?, ?, ?, ?)
        ''', users_data)

        # Livres de test
        livres_data = [
            ('Le Guide du Routard Galactique', 'Douglas Adams', '9782207249567', 2, 1979, 224, 'fr', 'Guide humoristique pour voyager dans la galaxie'),
            ('Clean Code', 'Robert C. Martin', '9780132350884', 3, 2008, 464, 'en', 'Guide pour écrire du code propre'),
            ('1984', 'George Orwell', '9782070368228', 1, 1949, 372, 'fr', 'Dystopie totalitaire classique'),
            ('Fondation', 'Isaac Asimov', '9782070313518', 2, 1951, 256, 'fr', 'Premier tome de la saga Fondation'),
            ('Histoire de France', 'Ernest Lavisse', '9782012345678', 4, 1900, 500, 'fr', 'Histoire de France classique')
        ]

        conn.executemany('''
            INSERT INTO livres (titre, auteur, isbn, genre_id, annee_publication,
                              nombre_pages, langue, resume)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ''', livres_data)

        # Emprunts de test (quelques uns actifs, d'autres terminés).
        # ⚠️ Utiliser UTC pour cohérence avec CURRENT_TIMESTAMP SQLite.
        #    `datetime.now()` (local) introduirait un décalage selon le fuseau.
        # ⚠️ On formate explicitement en string ISO sans microsecondes pour
        #    matcher le format de `CURRENT_TIMESTAMP` ('YYYY-MM-DD HH:MM:SS') ;
        #    sinon l'adapter datetime par défaut (Python 3.11) ajoute les
        #    microsecondes et Python 3.12+ émet un DeprecationWarning.
        from datetime import timezone
        now = datetime.now(timezone.utc).replace(tzinfo=None)
        def fmt(d):
            return d.strftime('%Y-%m-%d %H:%M:%S')

        emprunts_data = [
            # Emprunts actifs
            (1, 1, fmt(now - timedelta(days=5)),  fmt(now + timedelta(days=9)), None, 'actif', None),
            (2, 2, fmt(now - timedelta(days=10)), fmt(now + timedelta(days=4)), None, 'actif', None),
            # Emprunt en retard
            (3, 1, fmt(now - timedelta(days=20)), fmt(now - timedelta(days=6)), None, 'actif', None),
            # Emprunts terminés
            (4, 2, fmt(now - timedelta(days=30)), fmt(now - timedelta(days=16)),
                   fmt(now - timedelta(days=15)), 'retourne', 'Très bon livre'),
            (5, 1, fmt(now - timedelta(days=45)), fmt(now - timedelta(days=31)),
                   fmt(now - timedelta(days=28)), 'retourne', None),
        ]

        conn.executemany('''
            INSERT INTO emprunts (livre_id, utilisateur_id, date_emprunt,
                                date_retour_prevue, date_retour_effective, statut, commentaire)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        ''', emprunts_data)

        # Mettre à jour la disponibilité des livres
        conn.execute('''
            UPDATE livres SET disponible = 0
            WHERE id IN (SELECT livre_id FROM emprunts WHERE statut = 'actif')
        ''')

        conn.commit()
        print(f"✅ Données de test créées dans {db_path}")

def get_sample_book_data():
    """Retourne des données de livre pour les tests"""
    return {
        'titre': 'Test Book Title',
        'auteur': 'Test Author',
        'isbn': '9781234567890',
        'genre_id': 1,
        'annee_publication': 2024,
        'nombre_pages': 300,
        'langue': 'fr',
        'resume': 'Un livre de test pour valider nos fonctionnalités'
    }

def get_sample_user_data():
    """Retourne des données d'utilisateur pour les tests"""
    return {
        'nom': 'Utilisateur Test',
        'email': 'utilisateur@test.com',
        'mot_de_passe': 'motdepasse123',
        'est_admin': False
    }
```

## Tests unitaires des modèles

### Tests du modèle Livre

```python
# tests/unit/test_models.py
import pytest  
import sqlite3  
from app.models import Livre, Utilisateur, Emprunt, Genre  
from tests.fixtures.sample_data import get_sample_book_data, get_sample_user_data  

class TestLivreModel:
    """Tests unitaires pour le modèle Livre"""

    def test_create_livre_success(self, memory_db):
        """Test de création réussie d'un livre"""
        livre_model = Livre(memory_db)
        livre_data = get_sample_book_data()

        # Créer d'abord un genre
        genre_model = Genre(memory_db)
        genre_id = genre_model.create({
            'nom': 'Test Genre',
            'description': 'Genre de test'
        })
        livre_data['genre_id'] = genre_id

        # Créer le livre
        livre_id = livre_model.create(livre_data)

        assert livre_id is not None
        assert isinstance(livre_id, int)
        assert livre_id > 0

    def test_create_livre_missing_required_fields(self, memory_db):
        """Test d'échec de création avec champs obligatoires manquants.

        ⚠️ `pytest.raises(Exception)` est trop permissif (capturerait aussi
           KeyboardInterrupt). On cible les exceptions attendues :
           - KeyError : si `livre_data['titre']` est absent ET que `create()`
             y accède sans `.get()` (cas actuel : c'est `livre_data['titre']`)
           - sqlite3.IntegrityError : si la contrainte NOT NULL échoue
        """
        livre_model = Livre(memory_db)

        # Test sans titre
        livre_data = get_sample_book_data()
        del livre_data['titre']
        with pytest.raises((KeyError, sqlite3.IntegrityError)):
            livre_model.create(livre_data)

        # Test sans auteur
        livre_data = get_sample_book_data()
        del livre_data['auteur']
        with pytest.raises((KeyError, sqlite3.IntegrityError)):
            livre_model.create(livre_data)

    def test_get_livre_by_id(self, db_with_data):
        """Test de récupération d'un livre par ID"""
        livre_model = Livre(db_with_data)

        # Récupérer un livre existant
        livre = livre_model.get_by_id(1)

        assert livre is not None
        assert livre['id'] == 1
        assert livre['titre'] == 'Le Guide du Routard Galactique'
        assert livre['auteur'] == 'Douglas Adams'
        assert 'genre_nom' in livre  # Vérifier la jointure

    def test_get_livre_nonexistent(self, memory_db):
        """Test de récupération d'un livre inexistant"""
        livre_model = Livre(memory_db)

        livre = livre_model.get_by_id(999)
        assert livre is None

    def test_get_all_livres(self, db_with_data):
        """Test de récupération de tous les livres"""
        livre_model = Livre(db_with_data)

        livres = livre_model.get_all()

        assert len(livres) == 5  # Basé sur nos données de test
        assert all('titre' in livre for livre in livres)
        assert all('auteur' in livre for livre in livres)

    def test_get_all_livres_with_filters(self, db_with_data):
        """Test de filtrage des livres"""
        livre_model = Livre(db_with_data)

        # Filtrer par genre
        livres_sf = livre_model.get_all({'genre_id': 2})
        assert len(livres_sf) == 2  # SF books in our test data

        # Filtrer par disponibilité
        livres_disponibles = livre_model.get_all({'disponible': True})
        livres_empruntes = livre_model.get_all({'disponible': False})

        assert len(livres_disponibles) + len(livres_empruntes) == 5

        # Filtrer par recherche
        livres_recherche = livre_model.get_all({'search': 'Adams'})
        assert len(livres_recherche) == 1
        assert livres_recherche[0]['auteur'] == 'Douglas Adams'

    def test_update_livre(self, db_with_data):
        """Test de mise à jour d'un livre"""
        livre_model = Livre(db_with_data)

        # Mettre à jour un livre
        update_data = {
            'titre': 'Titre Modifié',
            'nombre_pages': 500
        }

        success = livre_model.update(1, update_data)
        assert success is True

        # Vérifier la mise à jour
        livre = livre_model.get_by_id(1)
        assert livre['titre'] == 'Titre Modifié'
        assert livre['nombre_pages'] == 500
        assert livre['auteur'] == 'Douglas Adams'  # Non modifié

    def test_update_livre_nonexistent(self, memory_db):
        """Test de mise à jour d'un livre inexistant"""
        livre_model = Livre(memory_db)

        success = livre_model.update(999, {'titre': 'Test'})
        assert success is False

    def test_delete_livre_available(self, db_with_data):
        """Test de suppression d'un livre disponible"""
        livre_model = Livre(db_with_data)

        # Identifier un livre disponible
        livres_disponibles = livre_model.get_all({'disponible': True})
        livre_id = livres_disponibles[0]['id']

        success, message = livre_model.delete(livre_id)
        assert success is True
        assert 'supprimé' in message.lower()

        # Vérifier que le livre n'existe plus
        livre = livre_model.get_by_id(livre_id)
        assert livre is None

    def test_delete_livre_borrowed(self, db_with_data):
        """Test d'échec de suppression d'un livre emprunté"""
        livre_model = Livre(db_with_data)

        # Identifier un livre emprunté
        livres_empruntes = livre_model.get_all({'disponible': False})
        livre_id = livres_empruntes[0]['id']

        success, message = livre_model.delete(livre_id)
        assert success is False
        assert 'emprunté' in message.lower()

class TestUtilisateurModel:
    """Tests unitaires pour le modèle Utilisateur"""

    def test_create_utilisateur_success(self, memory_db):
        """Test de création réussie d'un utilisateur"""
        user_model = Utilisateur(memory_db)
        user_data = get_sample_user_data()

        user_id = user_model.create(user_data)

        assert user_id is not None
        assert isinstance(user_id, int)
        assert user_id > 0

    def test_create_utilisateur_duplicate_email(self, memory_db):
        """Test d'échec de création avec email dupliqué"""
        user_model = Utilisateur(memory_db)
        user_data = get_sample_user_data()

        # Créer le premier utilisateur
        user_id1 = user_model.create(user_data)
        assert user_id1 is not None

        # Tenter de créer un second avec le même email
        user_id2 = user_model.create(user_data)
        assert user_id2 is None  # Doit échouer

    def test_authenticate_success(self, db_with_data):
        """Test d'authentification réussie"""
        user_model = Utilisateur(db_with_data)

        user = user_model.authenticate('alice@test.com', 'alice123')

        assert user is not None
        assert user['email'] == 'alice@test.com'
        assert user['nom'] == 'Alice Martin'
        assert 'mot_de_passe_hash' in user  # Hash présent mais pas exposé

    def test_authenticate_wrong_password(self, db_with_data):
        """Test d'authentification avec mauvais mot de passe"""
        user_model = Utilisateur(db_with_data)

        user = user_model.authenticate('alice@test.com', 'wrongpassword')
        assert user is None

    def test_authenticate_nonexistent_user(self, memory_db):
        """Test d'authentification d'un utilisateur inexistant"""
        user_model = Utilisateur(memory_db)

        user = user_model.authenticate('inexistant@test.com', 'password')
        assert user is None

class TestEmpruntModel:
    """Tests unitaires pour le modèle Emprunt"""

    def test_create_emprunt_success(self, db_with_data):
        """Test de création réussie d'un emprunt"""
        emprunt_model = Emprunt(db_with_data)

        # Identifier un livre disponible
        livre_model = Livre(db_with_data)
        livres_disponibles = livre_model.get_all({'disponible': True})
        livre_id = livres_disponibles[0]['id']

        emprunt_data = {
            'livre_id': livre_id,
            'utilisateur_id': 1
        }

        emprunt_id, message = emprunt_model.create(emprunt_data)

        assert emprunt_id is not None
        assert 'succès' in message.lower()

        # Vérifier que le livre n'est plus disponible
        livre = livre_model.get_by_id(livre_id)
        assert livre['disponible'] is False

    def test_create_emprunt_unavailable_book(self, db_with_data):
        """Test d'échec d'emprunt d'un livre non disponible"""
        emprunt_model = Emprunt(db_with_data)

        # Identifier un livre déjà emprunté
        livre_model = Livre(db_with_data)
        livres_empruntes = livre_model.get_all({'disponible': False})
        livre_id = livres_empruntes[0]['id']

        emprunt_data = {
            'livre_id': livre_id,
            'utilisateur_id': 1
        }

        emprunt_id, message = emprunt_model.create(emprunt_data)

        assert emprunt_id is None
        assert 'disponible' in message.lower()

    def test_return_book_success(self, db_with_data):
        """Test de retour réussi d'un livre"""
        emprunt_model = Emprunt(db_with_data)
        livre_model = Livre(db_with_data)

        # Identifier un emprunt actif
        emprunts_actifs = emprunt_model.get_active_loans()
        emprunt_id = emprunts_actifs[0]['id']
        livre_id = emprunts_actifs[0]['livre_id']

        success, message = emprunt_model.return_book(emprunt_id, 'Très bon livre')

        assert success is True
        assert 'succès' in message.lower()

        # Vérifier que le livre est à nouveau disponible
        livre = livre_model.get_by_id(livre_id)
        assert livre['disponible'] is True

    def test_return_book_nonexistent(self, memory_db):
        """Test de retour d'un emprunt inexistant"""
        emprunt_model = Emprunt(memory_db)

        success, message = emprunt_model.return_book(999)

        assert success is False
        assert 'trouvé' in message.lower()

    def test_get_active_loans(self, db_with_data):
        """Test de récupération des emprunts actifs"""
        emprunt_model = Emprunt(db_with_data)

        emprunts = emprunt_model.get_active_loans()

        assert len(emprunts) == 3  # Basé sur nos données de test
        assert all(emprunt['statut'] == 'actif' for emprunt in emprunts)
        assert all('titre' in emprunt for emprunt in emprunts)  # Jointure avec livres

    def test_get_overdue_loans(self, db_with_data):
        """Test de récupération des emprunts en retard"""
        emprunt_model = Emprunt(db_with_data)

        emprunts_retard = emprunt_model.get_overdue_loans()

        assert len(emprunts_retard) == 1  # Un emprunt en retard dans nos données
        assert all(emprunt['statut'] == 'actif' for emprunt in emprunts_retard)
```

## Tests d'intégration

### Tests de l'API REST

```python
# tests/integration/test_api.py
import pytest  
import json  
from flask import url_for  

class TestBooksAPI:
    """Tests d'intégration pour l'API des livres"""

    def test_get_books_unauthenticated(self, client):
        """Test d'accès non authentifié à l'API livres"""
        response = client.get('/api/books')
        assert response.status_code == 302  # Redirection vers login

    def test_get_books_authenticated(self, client, authenticated_user, db_with_data):
        """Test de récupération des livres authentifié"""
        response = client.get('/api/books')
        assert response.status_code == 200

        data = json.loads(response.data)
        assert isinstance(data, list)
        assert len(data) > 0
        assert 'titre' in data[0]
        assert 'auteur' in data[0]

    def test_get_books_with_filters(self, client, authenticated_user, db_with_data):
        """Test de filtrage des livres via API"""
        # Test filtrage par genre
        response = client.get('/api/books?genre_id=1')
        assert response.status_code == 200

        data = json.loads(response.data)
        # Tous les livres retournés doivent être du genre 1
        assert all(book.get('genre_id') == 1 for book in data if book.get('genre_id'))

        # Test recherche
        response = client.get('/api/books?search=Adams')
        assert response.status_code == 200

        data = json.loads(response.data)
        assert len(data) == 1
        assert 'Adams' in data[0]['auteur']

    def test_create_book_admin(self, client, admin_user):
        """Test de création de livre par un admin"""
        book_data = {
            'titre': 'Nouveau Livre API',
            'auteur': 'Auteur API',
            'isbn': '9780123456789',
            'annee_publication': 2024
        }

        response = client.post('/api/books',
                              data=json.dumps(book_data),
                              content_type='application/json')

        assert response.status_code == 201

        data = json.loads(response.data)
        assert 'id' in data
        assert 'message' in data
        assert 'succès' in data['message'].lower()

    def test_create_book_non_admin(self, client, authenticated_user):
        """Test d'échec de création de livre par un non-admin"""
        book_data = {
            'titre': 'Livre Non Autorisé',
            'auteur': 'Auteur Non Autorisé'
        }

        response = client.post('/api/books',
                              data=json.dumps(book_data),
                              content_type='application/json')

        assert response.status_code == 302  # Redirection (pas autorisé)

    def test_get_book_by_id(self, client, authenticated_user, db_with_data):
        """Test de récupération d'un livre par ID"""
        response = client.get('/api/books/1')
        assert response.status_code == 200

        data = json.loads(response.data)
        assert data['id'] == 1
        assert 'titre' in data
        assert 'auteur' in data
        assert 'genre_nom' in data  # Vérifier la jointure

    def test_update_book_admin(self, client, admin_user, db_with_data):
        """Test de modification de livre par un admin"""
        update_data = {
            'titre': 'Titre Modifié via API',
            'nombre_pages': 999
        }

        response = client.put('/api/books/1',
                             data=json.dumps(update_data),
                             content_type='application/json')

        assert response.status_code == 200

        # Vérifier la modification
        response = client.get('/api/books/1')
        data = json.loads(response.data)
        assert data['titre'] == 'Titre Modifié via API'
        assert data['nombre_pages'] == 999

    def test_delete_book_admin(self, client, admin_user, db_with_data):
        """Test de suppression de livre par un admin"""
        # Identifier un livre disponible
        response = client.get('/api/books?disponible=true')
        books = json.loads(response.data)
        book_id = books[0]['id']

        response = client.delete(f'/api/books/{book_id}')
        assert response.status_code == 200

        # Vérifier que le livre n'existe plus
        response = client.get(f'/api/books/{book_id}')
        assert response.status_code == 404

class TestLoansAPI:
    """Tests d'intégration pour l'API des emprunts"""

    def test_get_loans_user(self, client, authenticated_user, db_with_data):
        """Test de récupération des emprunts d'un utilisateur"""
        response = client.get('/api/loans')
        assert response.status_code == 200

        data = json.loads(response.data)
        assert isinstance(data, list)
        # Tous les emprunts doivent appartenir à l'utilisateur connecté
        assert all(loan.get('utilisateur_id') == 1 for loan in data)  # User ID from fixture

    def test_create_loan_success(self, client, authenticated_user, db_with_data):
        """Test de création d'emprunt réussi"""
        # Identifier un livre disponible
        response = client.get('/api/books?disponible=true')
        books = json.loads(response.data)
        book_id = books[0]['id']

        loan_data = {'livre_id': book_id}

        response = client.post('/api/loans',
                              data=json.dumps(loan_data),
                              content_type='application/json')

        assert response.status_code == 201

        data = json.loads(response.data)
        assert 'id' in data
        assert 'succès' in data['message'].lower()

    def test_create_loan_unavailable_book(self, client, authenticated_user, db_with_data):
        """Test d'échec d'emprunt d'un livre non disponible"""
        # Identifier un livre emprunté
        response = client.get('/api/books?disponible=false')
        books = json.loads(response.data)
        book_id = books[0]['id']

        loan_data = {'livre_id': book_id}

        response = client.post('/api/loans',
                              data=json.dumps(loan_data),
                              content_type='application/json')

        assert response.status_code == 400

        data = json.loads(response.data)
        assert 'error' in data
        assert 'disponible' in data['error'].lower()

    def test_return_book_success(self, client, authenticated_user, db_with_data):
        """Test de retour de livre réussi"""
        # Récupérer les emprunts actifs de l'utilisateur
        response = client.get('/api/loans')
        loans = json.loads(response.data)

        if loans:  # S'il y a des emprunts actifs
            loan_id = loans[0]['id']

            response = client.post(f'/api/loans/{loan_id}/return',
                                  data=json.dumps({'commentaire': 'Test retour'}),
                                  content_type='application/json')

            assert response.status_code == 200

            data = json.loads(response.data)
            assert 'succès' in data['message'].lower()

    def test_get_overdue_loans_admin(self, client, admin_user, db_with_data):
        """Test de récupération des emprunts en retard (admin)"""
        response = client.get('/api/loans/overdue')
        assert response.status_code == 200

        data = json.loads(response.data)
        assert isinstance(data, list)
        # Vérifier que tous les emprunts sont en retard
        # ⚠️ Comparer aware avec aware (sinon TypeError : naïf vs aware)
        from datetime import datetime, timezone
        now = datetime.now(timezone.utc)
        for loan in data:
            # date_retour_prevue est sérialisée en ISO depuis SQLite (UTC sans tz),
            # on l'attache à UTC pour la comparaison aware.
            ts = loan['date_retour_prevue'].replace('Z', '')
            due_date = datetime.fromisoformat(ts).replace(tzinfo=timezone.utc)
            assert due_date < now

class TestStatsAPI:
    """Tests d'intégration pour l'API des statistiques"""

    def test_get_stats_authenticated(self, client, authenticated_user, db_with_data):
        """Test de récupération des statistiques"""
        response = client.get('/api/stats')
        assert response.status_code == 200

        data = json.loads(response.data)
        assert 'livres' in data
        assert 'emprunts' in data
        assert 'genres' in data

        # Vérifier la structure des stats livres
        assert 'total' in data['livres']
        assert 'disponibles' in data['livres']
        assert 'empruntes' in data['livres']

        # Vérifier la cohérence
        total = data['livres']['total']
        disponibles = data['livres']['disponibles']
        empruntes = data['livres']['empruntes']
        assert total == disponibles + empruntes

    def test_get_stats_admin_details(self, client, admin_user, db_with_data):
        """Test des statistiques détaillées pour admin"""
        response = client.get('/api/stats')
        assert response.status_code == 200

        data = json.loads(response.data)
        # Les admins ont accès aux détails des retards
        if data['emprunts']['en_retard'] > 0:
            assert 'details_retard' in data['emprunts']
            assert isinstance(data['emprunts']['details_retard'], list)

class TestAuthAPI:
    """Tests d'intégration pour l'authentification"""

    def test_login_success(self, client, db_with_data):
        """Test de connexion réussie"""
        response = client.post('/login', data={
            'email': 'alice@test.com',
            'password': 'alice123'
        })

        assert response.status_code == 302  # Redirection après connexion

        # Vérifier que la session est créée
        with client.session_transaction() as sess:
            assert 'user_id' in sess
            assert sess['user_name'] == 'Alice Martin'

    def test_login_wrong_credentials(self, client, db_with_data):
        """Test de connexion avec mauvais identifiants"""
        response = client.post('/login', data={
            'email': 'alice@test.com',
            'password': 'wrongpassword'
        })

        # Vérifier qu'on reste sur la page de login
        assert b'incorrect' in response.data or response.status_code == 200

    def test_logout(self, client, authenticated_user):
        """Test de déconnexion"""
        response = client.get('/logout')
        assert response.status_code == 302  # Redirection

        # Vérifier que la session est vidée
        with client.session_transaction() as sess:
            assert 'user_id' not in sess

    def test_register_success(self, client, memory_db):
        """Test d'inscription réussie"""
        user_data = {
            'nom': 'Nouvel Utilisateur',
            'email': 'nouveau@test.com',
            'password': 'nouveaupass123'
        }

        response = client.post('/register', data=user_data)
        assert response.status_code == 302  # Redirection vers login

        # Vérifier que l'utilisateur peut se connecter
        response = client.post('/login', data={
            'email': user_data['email'],
            'password': user_data['password']
        })
        assert response.status_code == 302

    def test_register_duplicate_email(self, client, db_with_data):
        """Test d'inscription avec email existant"""
        user_data = {
            'nom': 'Autre Utilisateur',
            'email': 'alice@test.com',  # Email déjà utilisé
            'password': 'password123'
        }

        response = client.post('/register', data=user_data)
        # Doit rester sur la page d'inscription avec erreur
        assert b'déjà' in response.data or b'existant' in response.data
```

## Tests de workflows complets

```python
# tests/integration/test_workflows.py
import pytest  
import json  
from datetime import datetime, timedelta  

class TestBorrowReturnWorkflow:
    """Tests du workflow complet d'emprunt et retour"""

    def test_complete_borrow_return_cycle(self, client, authenticated_user, db_with_data):
        """Test d'un cycle complet emprunt-retour"""

        # 1. Récupérer un livre disponible
        response = client.get('/api/books?disponible=true')
        assert response.status_code == 200
        books = json.loads(response.data)
        assert len(books) > 0

        book = books[0]
        book_id = book['id']

        # 2. Emprunter le livre
        loan_data = {'livre_id': book_id}
        response = client.post('/api/loans',
                              data=json.dumps(loan_data),
                              content_type='application/json')
        assert response.status_code == 201

        loan_response = json.loads(response.data)
        loan_id = loan_response['id']

        # 3. Vérifier que le livre n'est plus disponible
        response = client.get(f'/api/books/{book_id}')
        updated_book = json.loads(response.data)
        assert updated_book['disponible'] is False

        # 4. Vérifier que l'emprunt apparaît dans la liste
        response = client.get('/api/loans')
        loans = json.loads(response.data)
        loan_ids = [loan['id'] for loan in loans]
        assert loan_id in loan_ids

        # 5. Retourner le livre
        return_data = {'commentaire': 'Excellent livre!'}
        response = client.post(f'/api/loans/{loan_id}/return',
                              data=json.dumps(return_data),
                              content_type='application/json')
        assert response.status_code == 200

        # 6. Vérifier que le livre est à nouveau disponible
        response = client.get(f'/api/books/{book_id}')
        returned_book = json.loads(response.data)
        assert returned_book['disponible'] is True

        # 7. Vérifier que l'emprunt n'apparaît plus dans les actifs
        response = client.get('/api/loans')
        active_loans = json.loads(response.data)
        active_loan_ids = [loan['id'] for loan in active_loans]
        assert loan_id not in active_loan_ids

    def test_multiple_users_same_book(self, client, db_with_data):
        """Test que plusieurs utilisateurs ne peuvent pas emprunter le même livre"""

        # Connexion utilisateur 1
        client.post('/login', data={
            'email': 'alice@test.com',
            'password': 'alice123'
        })

        # Récupérer un livre disponible
        response = client.get('/api/books?disponible=true')
        books = json.loads(response.data)
        book_id = books[0]['id']

        # Utilisateur 1 emprunte le livre
        loan_data = {'livre_id': book_id}
        response = client.post('/api/loans',
                              data=json.dumps(loan_data),
                              content_type='application/json')
        assert response.status_code == 201

        # Déconnexion utilisateur 1
        client.get('/logout')

        # Connexion utilisateur 2
        client.post('/login', data={
            'email': 'bob@test.com',
            'password': 'bob123'
        })

        # Utilisateur 2 tente d'emprunter le même livre
        response = client.post('/api/loans',
                              data=json.dumps(loan_data),
                              content_type='application/json')
        assert response.status_code == 400  # Doit échouer

        error_data = json.loads(response.data)
        assert 'disponible' in error_data['error'].lower()

class TestAdminWorkflow:
    """Tests des workflows administrateur"""

    def test_admin_book_management_cycle(self, client, admin_user):
        """Test complet de gestion de livre par un admin"""

        # 1. Créer un nouveau livre
        book_data = {
            'titre': 'Livre Admin Test',
            'auteur': 'Auteur Admin',
            'isbn': '9781111111111',
            'annee_publication': 2024,
            'nombre_pages': 250,
            'resume': 'Un livre créé pour tester les fonctions admin'
        }

        response = client.post('/api/books',
                              data=json.dumps(book_data),
                              content_type='application/json')
        assert response.status_code == 201

        creation_response = json.loads(response.data)
        book_id = creation_response['id']

        # 2. Vérifier que le livre existe
        response = client.get(f'/api/books/{book_id}')
        assert response.status_code == 200
        created_book = json.loads(response.data)
        assert created_book['titre'] == book_data['titre']

        # 3. Modifier le livre
        update_data = {
            'titre': 'Livre Admin Test - Modifié',
            'nombre_pages': 300
        }

        response = client.put(f'/api/books/{book_id}',
                             data=json.dumps(update_data),
                             content_type='application/json')
        assert response.status_code == 200

        # 4. Vérifier les modifications
        response = client.get(f'/api/books/{book_id}')
        updated_book = json.loads(response.data)
        assert updated_book['titre'] == update_data['titre']
        assert updated_book['nombre_pages'] == update_data['nombre_pages']
        assert updated_book['auteur'] == book_data['auteur']  # Non modifié

        # 5. Supprimer le livre
        response = client.delete(f'/api/books/{book_id}')
        assert response.status_code == 200

        # 6. Vérifier que le livre n'existe plus
        response = client.get(f'/api/books/{book_id}')
        assert response.status_code == 404

    def test_admin_overdue_management(self, client, admin_user, db_with_data):
        """Test de gestion des emprunts en retard par admin."""

        # 1. Récupérer les emprunts en retard
        response = client.get('/api/loans/overdue')
        assert response.status_code == 200

        overdue_loans = json.loads(response.data)

        if overdue_loans:
            # 2. Vérifier les informations des emprunts en retard
            from datetime import timezone
            now = datetime.now(timezone.utc)
            for loan in overdue_loans:
                assert 'nom_utilisateur' in loan
                assert 'email' in loan
                assert 'titre' in loan
                assert 'date_retour_prevue' in loan

                # date_retour_prevue est en UTC depuis CURRENT_TIMESTAMP SQLite.
                # On l'attache à UTC pour comparer avec datetime.now(utc) (aware).
                due_date = datetime.fromisoformat(loan['date_retour_prevue']).replace(tzinfo=timezone.utc)
                assert due_date < now
```

## Tests de performance

```python
# tests/performance/test_performance.py
import pytest  
import time  
import sqlite3  
from faker import Faker  
from app.models import Livre, Utilisateur, Emprunt  

fake = Faker('fr_FR')

class TestPerformance:
    """Tests de performance pour SQLite"""

    @pytest.fixture
    def large_dataset_db(self, temp_db):
        """Crée une base avec un grand dataset pour les tests de performance"""
        self.create_large_dataset(temp_db, 10000)  # 10k livres
        return temp_db

    def create_large_dataset(self, db_path, num_books):
        """Crée un grand jeu de données pour les tests"""
        with sqlite3.connect(db_path) as conn:
            # Créer des genres
            genres = [
                ('Roman', 'Fiction littéraire', '#e74c3c'),
                ('Science-Fiction', 'SF', '#9b59b6'),
                ('Policier', 'Polar', '#34495e'),
                ('Histoire', 'Livres historiques', '#f39c12'),
                ('Science', 'Ouvrages scientifiques', '#27ae60')
            ]

            conn.executemany(
                "INSERT INTO genres (nom, description, couleur) VALUES (?, ?, ?)",
                genres
            )

            # Créer des livres en lot
            livres_data = []
            for i in range(num_books):
                livres_data.append((
                    fake.sentence(nb_words=3)[:-1],  # Titre
                    fake.name(),  # Auteur
                    fake.isbn13() if i % 3 == 0 else None,  # ISBN (pas tous)
                    fake.random_int(1, 5),  # Genre ID
                    fake.random_int(1950, 2024),  # Année
                    fake.random_int(50, 800),  # Pages
                    'fr',
                    fake.text(200) if i % 5 == 0 else None  # Résumé (pas tous)
                ))

            conn.executemany('''
                INSERT INTO livres (titre, auteur, isbn, genre_id, annee_publication,
                                  nombre_pages, langue, resume)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ''', livres_data)

            print(f"✅ Créé {num_books} livres pour les tests de performance")

    def test_search_performance(self, large_dataset_db):
        """Test des performances de recherche"""
        livre_model = Livre(large_dataset_db)

        # Test de recherche simple
        start_time = time.time()
        results = livre_model.get_all({'search': 'test'})
        search_time = time.time() - start_time

        print(f"🔍 Recherche sur {10000} livres: {search_time:.3f}s")
        assert search_time < 0.5  # Doit être rapide avec les index

    def test_filtering_performance(self, large_dataset_db):
        """Test des performances de filtrage"""
        livre_model = Livre(large_dataset_db)

        # Test filtrage par genre
        start_time = time.time()
        results = livre_model.get_all({'genre_id': 1})
        filter_time = time.time() - start_time

        print(f"📊 Filtrage par genre: {filter_time:.3f}s")
        assert filter_time < 0.1  # Très rapide avec index sur genre_id

    def test_pagination_performance(self, large_dataset_db):
        """Test des performances de pagination"""
        with sqlite3.connect(large_dataset_db) as conn:
            # Test pagination avec LIMIT/OFFSET
            start_time = time.time()

            cursor = conn.execute('''
                SELECT * FROM livres
                ORDER BY titre
                LIMIT 50 OFFSET 5000
            ''')
            results = cursor.fetchall()

            pagination_time = time.time() - start_time

            print(f"📄 Pagination (OFFSET 5000): {pagination_time:.3f}s")
            assert len(results) == 50
            assert pagination_time < 0.2

    def test_aggregate_performance(self, large_dataset_db):
        """Test des performances des requêtes d'agrégation"""
        with sqlite3.connect(large_dataset_db) as conn:
            # Test agrégation par genre
            start_time = time.time()

            cursor = conn.execute('''
                SELECT g.nom, COUNT(l.id) as nb_livres, AVG(l.nombre_pages) as pages_moy
                FROM genres g
                LEFT JOIN livres l ON g.id = l.genre_id
                GROUP BY g.id, g.nom
                ORDER BY nb_livres DESC
            ''')
            results = cursor.fetchall()

            aggregate_time = time.time() - start_time

            print(f"📈 Agrégation par genre: {aggregate_time:.3f}s")
            assert aggregate_time < 0.3
            assert len(results) == 5  # Nos 5 genres

    def test_insert_batch_performance(self, temp_db):
        """Test des performances d'insertion en lot"""
        # Préparer 1000 livres
        livres_data = []
        for i in range(1000):
            livres_data.append((
                f'Livre Test {i}',
                f'Auteur {i}',
                None, None, 2024, 300, 'fr', None
            ))

        with sqlite3.connect(temp_db) as conn:
            # Test insertion une par une (lent)
            start_time = time.time()
            for livre in livres_data[:100]:  # Seulement 100 pour ne pas être trop long
                conn.execute('''
                    INSERT INTO livres (titre, auteur, isbn, genre_id,
                                      annee_publication, nombre_pages, langue, resume)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                ''', livre)
            single_time = time.time() - start_time

            # Test insertion en lot (rapide)
            start_time = time.time()
            conn.executemany('''
                INSERT INTO livres (titre, auteur, isbn, genre_id,
                                  annee_publication, nombre_pages, langue, resume)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ''', livres_data[100:])
            batch_time = time.time() - start_time

            print(f"💾 Insertion 100 livres une par une: {single_time:.3f}s")
            print(f"💾 Insertion 900 livres en lot: {batch_time:.3f}s")
            print(f"📊 Ratio performance: {single_time/batch_time:.1f}x plus rapide")

            assert batch_time < single_time  # Le lot doit être plus rapide

    @pytest.mark.benchmark
    def test_complex_query_benchmark(self, benchmark, large_dataset_db):
        """Benchmark d'une requête complexe avec pytest-benchmark"""

        def complex_query():
            with sqlite3.connect(large_dataset_db) as conn:
                cursor = conn.execute('''
                    SELECT
                        l.titre,
                        l.auteur,
                        g.nom as genre,
                        l.annee_publication,
                        CASE
                            WHEN l.nombre_pages < 200 THEN 'Court'
                            WHEN l.nombre_pages < 400 THEN 'Moyen'
                            ELSE 'Long'
                        END as taille,
                        l.disponible
                    FROM livres l
                    JOIN genres g ON l.genre_id = g.id
                    WHERE l.annee_publication BETWEEN 2000 AND 2024
                    AND l.nombre_pages > 100
                    ORDER BY l.annee_publication DESC, l.titre
                    LIMIT 100
                ''')
                return cursor.fetchall()

        # Exécuter le benchmark
        result = benchmark(complex_query)
        assert len(result) <= 100
        print(f"📊 Requête complexe benchmarkée")

class TestConcurrency:
    """Tests de concurrence avec SQLite"""

    def test_multiple_readers(self, db_with_data):
        """Test de lectures concurrentes"""
        import threading
        import queue

        results_queue = queue.Queue()

        def read_books(db_path, results_queue):
            try:
                livre_model = Livre(db_path)
                books = livre_model.get_all()
                results_queue.put(('success', len(books)))
            except Exception as e:
                results_queue.put(('error', str(e)))

        # Lancer plusieurs lecteurs en parallèle
        threads = []
        for i in range(5):
            thread = threading.Thread(
                target=read_books,
                args=(db_with_data, results_queue)
            )
            threads.append(thread)
            thread.start()

        # Attendre que tous finissent
        for thread in threads:
            thread.join()

        # Vérifier les résultats
        success_count = 0
        while not results_queue.empty():
            status, result = results_queue.get()
            if status == 'success':
                success_count += 1
                assert result == 5  # 5 livres dans nos données de test
            else:
                pytest.fail(f"Erreur de lecture concurrente: {result}")

        assert success_count == 5  # Tous les threads ont réussi

    def test_reader_writer_scenario(self, temp_db):
        """Test de scénario lecteur-écrivain"""
        import threading
        import time

        results = {'reads': 0, 'writes': 0, 'errors': []}

        def reader_task():
            try:
                for i in range(10):
                    livre_model = Livre(temp_db)
                    books = livre_model.get_all()
                    results['reads'] += 1
                    time.sleep(0.01)  # Petite pause
            except Exception as e:
                results['errors'].append(f"Reader error: {e}")

        def writer_task():
            try:
                for i in range(5):
                    livre_model = Livre(temp_db)
                    book_data = {
                        'titre': f'Livre Concurrent {i}',
                        'auteur': f'Auteur {i}'
                    }
                    livre_model.create(book_data)
                    results['writes'] += 1
                    time.sleep(0.02)  # Petite pause
            except Exception as e:
                results['errors'].append(f"Writer error: {e}")

        # Lancer lecteurs et écrivains
        threads = []

        # 2 lecteurs
        for i in range(2):
            thread = threading.Thread(target=reader_task)
            threads.append(thread)
            thread.start()

        # 1 écrivain
        writer_thread = threading.Thread(target=writer_task)
        threads.append(writer_thread)
        writer_thread.start()

        # Attendre la fin
        for thread in threads:
            thread.join()

        # Vérifier les résultats
        print(f"📚 Lectures réussies: {results['reads']}")
        print(f"✏️ Écritures réussies: {results['writes']}")
        print(f"❌ Erreurs: {len(results['errors'])}")

        if results['errors']:
            for error in results['errors']:
                print(f"   {error}")

        assert results['reads'] == 20  # 2 lecteurs × 10 lectures
        assert results['writes'] == 5   # 1 écrivain × 5 écritures
        assert len(results['errors']) == 0  # Aucune erreur
```

## Tests de migration et intégrité des données

```python
# tests/data/test_migrations.py
import pytest  
import sqlite3  
import os  
import tempfile  
from app.database import DatabaseManager  

class TestDatabaseMigrations:
    """Tests des migrations de schéma de base de données"""

    def test_fresh_database_creation(self):
        """Test de création d'une nouvelle base"""
        fd, path = tempfile.mkstemp(suffix='.db')
        os.close(fd)

        try:
            # Créer la base
            db_manager = DatabaseManager(path)

            # Vérifier que toutes les tables existent
            with sqlite3.connect(path) as conn:
                cursor = conn.execute('''
                    SELECT name FROM sqlite_master
                    WHERE type='table' AND name NOT LIKE 'sqlite_%'
                ''')
                tables = [row[0] for row in cursor.fetchall()]

            expected_tables = ['genres', 'utilisateurs', 'livres', 'emprunts', 'evaluations']
            for table in expected_tables:
                assert table in tables, f"Table {table} manquante"

            # Vérifier les index
            cursor = conn.execute('''
                SELECT name FROM sqlite_master
                WHERE type='index' AND name NOT LIKE 'sqlite_%'
            ''')
            indexes = [row[0] for row in cursor.fetchall()]

            # Vérifier qu'il y a des index (au moins quelques-uns)
            assert len(indexes) > 5, "Pas assez d'index créés"

        finally:
            if os.path.exists(path):
                os.unlink(path)

    def test_schema_constraints(self, temp_db):
        """Test des contraintes de schéma"""
        with sqlite3.connect(temp_db) as conn:

            # Test contrainte UNIQUE sur email utilisateur
            conn.execute('''
                INSERT INTO utilisateurs (nom, email, mot_de_passe_hash)
                VALUES ('User1', 'test@example.com', 'hash1')
            ''')

            with pytest.raises(sqlite3.IntegrityError):
                conn.execute('''
                    INSERT INTO utilisateurs (nom, email, mot_de_passe_hash)
                    VALUES ('User2', 'test@example.com', 'hash2')
                ''')

            # Test contrainte FOREIGN KEY
            with pytest.raises(sqlite3.IntegrityError):
                conn.execute('''
                    INSERT INTO livres (titre, auteur, genre_id)
                    VALUES ('Test Book', 'Test Author', 999)  -- Genre inexistant
                ''')

    def test_database_upgrade_simulation(self, temp_db):
        """Simulation d'une mise à jour de schéma"""
        # Simuler l'ajout d'une nouvelle colonne
        with sqlite3.connect(temp_db) as conn:
            # Vérifier que la colonne n'existe pas encore
            cursor = conn.execute("PRAGMA table_info(livres)")
            columns = [col[1] for col in cursor.fetchall()]

            if 'note_moyenne' not in columns:
                # Ajouter la nouvelle colonne
                conn.execute('ALTER TABLE livres ADD COLUMN note_moyenne REAL DEFAULT 0')

                # Vérifier que la colonne a été ajoutée
                cursor = conn.execute("PRAGMA table_info(livres)")
                new_columns = [col[1] for col in cursor.fetchall()]
                assert 'note_moyenne' in new_columns

class TestDataIntegrity:
    """Tests d'intégrité des données"""

    def test_book_availability_consistency(self, db_with_data):
        """Test de cohérence de la disponibilité des livres"""
        with sqlite3.connect(db_with_data) as conn:
            # Récupérer les livres marqués comme non disponibles
            cursor = conn.execute('''
                SELECT id FROM livres WHERE disponible = 0
            ''')
            unavailable_books = [row[0] for row in cursor.fetchall()]

            # Vérifier qu'ils ont tous un emprunt actif
            for book_id in unavailable_books:
                cursor = conn.execute('''
                    SELECT COUNT(*) FROM emprunts
                    WHERE livre_id = ? AND statut = 'actif'
                ''', (book_id,))
                active_loans = cursor.fetchone()[0]
                assert active_loans > 0, f"Livre {book_id} indisponible mais sans emprunt actif"

            # Récupérer les livres marqués comme disponibles
            cursor = conn.execute('''
                SELECT id FROM livres WHERE disponible = 1
            ''')
            available_books = [row[0] for row in cursor.fetchall()]

            # Vérifier qu'ils n'ont pas d'emprunt actif
            for book_id in available_books:
                cursor = conn.execute('''
                    SELECT COUNT(*) FROM emprunts
                    WHERE livre_id = ? AND statut = 'actif'
                ''', (book_id,))
                active_loans = cursor.fetchone()[0]
                assert active_loans == 0, f"Livre {book_id} disponible mais avec emprunt actif"

    def test_loan_data_consistency(self, db_with_data):
        """Test de cohérence des données d'emprunt"""
        with sqlite3.connect(db_with_data) as conn:
            # Vérifier que tous les emprunts ont des dates cohérentes
            cursor = conn.execute('''
                SELECT id, date_emprunt, date_retour_prevue, date_retour_effective
                FROM emprunts
            ''')

            for loan in cursor.fetchall():
                loan_id, date_emprunt, date_retour_prevue, date_retour_effective = loan

                # Date d'emprunt doit être antérieure à la date de retour prévue
                assert date_emprunt < date_retour_prevue, \
                    f"Emprunt {loan_id}: date d'emprunt postérieure à la date de retour prévue"

                # Si retourné, date effective doit être postérieure à l'emprunt
                if date_retour_effective:
                    assert date_emprunt < date_retour_effective, \
                        f"Emprunt {loan_id}: date de retour antérieure à l'emprunt"

    def test_foreign_key_integrity(self, db_with_data):
        """Test d'intégrité des clés étrangères"""
        with sqlite3.connect(db_with_data) as conn:
            # Vérifier que tous les livres référencent des genres existants
            cursor = conn.execute('''
                SELECT l.id, l.titre, l.genre_id
                FROM livres l
                LEFT JOIN genres g ON l.genre_id = g.id
                WHERE l.genre_id IS NOT NULL AND g.id IS NULL
            ''')
            orphaned_books = cursor.fetchall()
            assert len(orphaned_books) == 0, f"Livres avec genres inexistants: {orphaned_books}"

            # Vérifier que tous les emprunts référencent des livres et utilisateurs existants
            cursor = conn.execute('''
                SELECT e.id
                FROM emprunts e
                LEFT JOIN livres l ON e.livre_id = l.id
                LEFT JOIN utilisateurs u ON e.utilisateur_id = u.id
                WHERE l.id IS NULL OR u.id IS NULL
            ''')
            orphaned_loans = cursor.fetchall()
            assert len(orphaned_loans) == 0, f"Emprunts orphelins: {orphaned_loans}"

    def test_data_types_validation(self, db_with_data):
        """Test de validation des types de données"""
        with sqlite3.connect(db_with_data) as conn:
            # Vérifier que les années de publication sont raisonnables
            cursor = conn.execute('''
                SELECT id, titre, annee_publication
                FROM livres
                WHERE annee_publication IS NOT NULL
                AND (annee_publication < 1000 OR annee_publication > 2030)
            ''')
            invalid_years = cursor.fetchall()
            assert len(invalid_years) == 0, f"Années de publication invalides: {invalid_years}"

            # Vérifier que les nombres de pages sont positifs
            cursor = conn.execute('''
                SELECT id, titre, nombre_pages
                FROM livres
                WHERE nombre_pages IS NOT NULL AND nombre_pages <= 0
            ''')
            invalid_pages = cursor.fetchall()
            assert len(invalid_pages) == 0, f"Nombres de pages invalides: {invalid_pages}"

    def test_unique_constraints(self, db_with_data):
        """Test des contraintes d'unicité"""
        with sqlite3.connect(db_with_data) as conn:
            # Vérifier l'unicité des emails
            cursor = conn.execute('''
                SELECT email, COUNT(*)
                FROM utilisateurs
                GROUP BY email
                HAVING COUNT(*) > 1
            ''')
            duplicate_emails = cursor.fetchall()
            assert len(duplicate_emails) == 0, f"Emails dupliqués: {duplicate_emails}"

            # Vérifier l'unicité des ISBN (si définis)
            cursor = conn.execute('''
                SELECT isbn, COUNT(*)
                FROM livres
                WHERE isbn IS NOT NULL
                GROUP BY isbn
                HAVING COUNT(*) > 1
            ''')
            duplicate_isbn = cursor.fetchall()
            assert len(duplicate_isbn) == 0, f"ISBN dupliqués: {duplicate_isbn}"

class TestDataValidation:
    """Tests de validation des données métier"""

    def test_loan_business_rules(self, db_with_data):
        """Test des règles métier pour les emprunts"""
        with sqlite3.connect(db_with_data) as conn:
            # Un utilisateur ne peut pas emprunter le même livre plusieurs fois en même temps
            cursor = conn.execute('''
                SELECT utilisateur_id, livre_id, COUNT(*)
                FROM emprunts
                WHERE statut = 'actif'
                GROUP BY utilisateur_id, livre_id
                HAVING COUNT(*) > 1
            ''')
            multiple_active_loans = cursor.fetchall()
            assert len(multiple_active_loans) == 0, \
                f"Emprunts multiples du même livre: {multiple_active_loans}"

            # Un livre ne peut être emprunté que par un utilisateur à la fois
            cursor = conn.execute('''
                SELECT livre_id, COUNT(*)
                FROM emprunts
                WHERE statut = 'actif'
                GROUP BY livre_id
                HAVING COUNT(*) > 1
            ''')
            multiple_borrowers = cursor.fetchall()
            assert len(multiple_borrowers) == 0, \
                f"Livres empruntés par plusieurs utilisateurs: {multiple_borrowers}"

    def test_data_completeness(self, db_with_data):
        """Test de complétude des données obligatoires"""
        with sqlite3.connect(db_with_data) as conn:
            # Vérifier que tous les livres ont un titre et un auteur
            cursor = conn.execute('''
                SELECT id FROM livres
                WHERE titre IS NULL OR titre = '' OR auteur IS NULL OR auteur = ''
            ''')
            incomplete_books = cursor.fetchall()
            assert len(incomplete_books) == 0, f"Livres incomplets: {incomplete_books}"

            # Vérifier que tous les utilisateurs ont un nom et un email
            cursor = conn.execute('''
                SELECT id FROM utilisateurs
                WHERE nom IS NULL OR nom = '' OR email IS NULL OR email = ''
            ''')
            incomplete_users = cursor.fetchall()
            assert len(incomplete_users) == 0, f"Utilisateurs incomplets: {incomplete_users}"
```

## Tests de régression

```python
# tests/regression/test_regression.py
import pytest  
import json  
from datetime import datetime, timedelta  

class TestRegression:
    """Tests de régression pour éviter la réintroduction de bugs"""

    def test_bug_fix_double_borrow_prevention(self, client, authenticated_user, db_with_data):
        """
        Test de régression: Bug #001 - Un utilisateur pouvait emprunter le même livre deux fois
        Fix: Vérification de l'existence d'un emprunt actif avant création
        """
        # Identifier un livre disponible
        response = client.get('/api/books?disponible=true')
        books = json.loads(response.data)
        book_id = books[0]['id']

        # Premier emprunt (doit réussir)
        loan_data = {'livre_id': book_id}
        response1 = client.post('/api/loans',
                               data=json.dumps(loan_data),
                               content_type='application/json')
        assert response1.status_code == 201

        # Deuxième emprunt du même livre (doit échouer)
        response2 = client.post('/api/loans',
                               data=json.dumps(loan_data),
                               content_type='application/json')
        assert response2.status_code == 400

        error_data = json.loads(response2.data)
        assert 'disponible' in error_data['error'].lower()

    def test_bug_fix_return_book_twice(self, client, authenticated_user, db_with_data):
        """
        Test de régression: Bug #002 - Retour d'un livre déjà retourné causait une erreur
        Fix: Vérification du statut avant retour
        """
        # Créer un emprunt et le retourner
        response = client.get('/api/books?disponible=true')
        books = json.loads(response.data)
        book_id = books[0]['id']

        # Emprunter
        loan_data = {'livre_id': book_id}
        response = client.post('/api/loans',
                              data=json.dumps(loan_data),
                              content_type='application/json')
        loan_id = json.loads(response.data)['id']

        # Premier retour (doit réussir)
        response1 = client.post(f'/api/loans/{loan_id}/return',
                               content_type='application/json')
        assert response1.status_code == 200

        # Deuxième retour (doit échouer proprement)
        response2 = client.post(f'/api/loans/{loan_id}/return',
                               content_type='application/json')
        assert response2.status_code == 400

        error_data = json.loads(response2.data)
        assert 'retourné' in error_data['error'].lower()

    def test_bug_fix_sql_injection_search(self, client, authenticated_user, db_with_data):
        """
        Test de régression: Bug #003 - Injection SQL possible dans la recherche
        Fix: Utilisation de requêtes paramétrées
        """
        # Tenter une injection SQL dans la recherche
        malicious_search = "'; DROP TABLE livres; --"

        response = client.get(f'/api/books?search={malicious_search}')
        assert response.status_code == 200

        # Vérifier que la table existe toujours
        response = client.get('/api/books')
        assert response.status_code == 200
        books = json.loads(response.data)
        assert isinstance(books, list)  # La table n'a pas été supprimée

    def test_bug_fix_empty_search_crash(self, client, authenticated_user, db_with_data):
        """
        Test de régression: Bug #004 - Recherche vide causait un crash
        Fix: Gestion des paramètres vides
        """
        # Recherche avec chaîne vide
        response = client.get('/api/books?search=')
        assert response.status_code == 200

        # Recherche avec espaces seulement
        response = client.get('/api/books?search=   ')
        assert response.status_code == 200

        # Recherche avec caractères spéciaux
        response = client.get('/api/books?search=%')
        assert response.status_code == 200
```

## Tests de charge et stress

```python
# tests/stress/test_stress.py
import pytest  
import threading  
import time  
import sqlite3  
from concurrent.futures import ThreadPoolExecutor, as_completed  
from app.models import Livre, Utilisateur, Emprunt  

class TestStress:
    """Tests de charge et de stress.

    ⚠️ Préalable critique : ces tests SUPPOSENT que la base est en mode WAL
       (`PRAGMA journal_mode = WAL`). Sans WAL, les écritures concurrentes
       sérialisent en mode rollback et provoquent des `database is locked`
       en cascade — le taux de succès attendu de 95% ne sera PAS atteint.

       S'assurer que `DatabaseManager.init_database()` active WAL :
         conn.execute("PRAGMA journal_mode = WAL")
       Et augmenter le busy_timeout sur chaque connexion :
         conn.execute("PRAGMA busy_timeout = 5000")
    """

    def test_concurrent_book_creation(self, temp_db):
        """Test de création concurrente de livres (nécessite WAL — cf. note de classe)"""

        def create_books(start_index, count):
            """Crée des livres dans un thread"""
            livre_model = Livre(temp_db)
            created = []
            errors = []

            for i in range(count):
                try:
                    book_data = {
                        'titre': f'Livre Concurrent {start_index + i}',
                        'auteur': f'Auteur {start_index + i}',
                        'isbn': f'978{start_index:04d}{i:05d}'
                    }
                    book_id = livre_model.create(book_data)
                    if book_id:
                        created.append(book_id)
                    else:
                        errors.append(f"Failed to create book {start_index + i}")
                except Exception as e:
                    errors.append(f"Exception creating book {start_index + i}: {e}")

            return created, errors

        # Lancer plusieurs threads
        num_threads = 5
        books_per_thread = 20

        with ThreadPoolExecutor(max_workers=num_threads) as executor:
            futures = []
            for i in range(num_threads):
                start_index = i * books_per_thread
                future = executor.submit(create_books, start_index, books_per_thread)
                futures.append(future)

            # Collecter les résultats
            total_created = 0
            total_errors = []

            for future in as_completed(futures):
                created, errors = future.result()
                total_created += len(created)
                total_errors.extend(errors)

        print(f"📚 Livres créés en concurrence: {total_created}")
        print(f"❌ Erreurs: {len(total_errors)}")

        # Vérifier que la plupart des créations ont réussi
        expected_total = num_threads * books_per_thread
        success_rate = total_created / expected_total
        assert success_rate > 0.95, f"Taux de succès trop faible: {success_rate:.2%}"

        if total_errors:
            print("Erreurs détaillées:")
            for error in total_errors[:5]:  # Afficher seulement les 5 premières
                print(f"  {error}")

    def test_database_under_load(self, temp_db):
        """Test de la base sous charge mixte (lecture/écriture)"""

        # Créer des données initiales
        livre_model = Livre(temp_db)
        for i in range(50):
            livre_model.create({
                'titre': f'Livre Initial {i}',
                'auteur': f'Auteur {i}'
            })

        results = {
            'reads': 0,
            'writes': 0,
            'updates': 0,
            'errors': []
        }

        def reader_task(task_id):
            """Tâche de lecture intensive"""
            try:
                livre_model = Livre(temp_db)
                for i in range(100):
                    books = livre_model.get_all()
                    results['reads'] += 1
                    if i % 20 == 0:  # Petite pause occasionnelle
                        time.sleep(0.001)
            except Exception as e:
                results['errors'].append(f"Reader {task_id}: {e}")

        def writer_task(task_id):
            """Tâche d'écriture"""
            try:
                livre_model = Livre(temp_db)
                for i in range(20):
                    book_data = {
                        'titre': f'Livre Writer {task_id}-{i}',
                        'auteur': f'Auteur Writer {task_id}'
                    }
                    book_id = livre_model.create(book_data)
                    if book_id:
                        results['writes'] += 1
                    time.sleep(0.005)  # Écriture plus lente
            except Exception as e:
                results['errors'].append(f"Writer {task_id}: {e}")

        def updater_task(task_id):
            """Tâche de mise à jour"""
            try:
                livre_model = Livre(temp_db)
                for i in range(10):
                    # Mettre à jour un livre existant
                    book_id = (task_id % 50) + 1  # ID entre 1 et 50
                    update_data = {
                        'nombre_pages': 200 + i * 10
                    }
                    if livre_model.update(book_id, update_data):
                        results['updates'] += 1
                    time.sleep(0.01)
            except Exception as e:
                results['errors'].append(f"Updater {task_id}: {e}")

        # Lancer le test de charge
        threads = []

        # 3 lecteurs
        for i in range(3):
            thread = threading.Thread(target=reader_task, args=(i,))
            threads.append(thread)

        # 2 écrivains
        for i in range(2):
            thread = threading.Thread(target=writer_task, args=(i,))
            threads.append(thread)

        # 1 updater
        thread = threading.Thread(target=updater_task, args=(0,))
        threads.append(thread)

        # Démarrer tous les threads
        start_time = time.time()
        for thread in threads:
            thread.start()

        # Attendre la fin
        for thread in threads:
            thread.join()

        end_time = time.time()
        duration = end_time - start_time

        print(f"⏱️ Durée du test de charge: {duration:.2f}s")
        print(f"📖 Lectures: {results['reads']}")
        print(f"✏️ Écritures: {results['writes']}")
        print(f"🔄 Mises à jour: {results['updates']}")
        print(f"❌ Erreurs: {len(results['errors'])}")

        # Vérifications
        assert results['reads'] > 290  # 3 lecteurs × 100 - quelques erreurs OK
        assert results['writes'] > 35  # 2 écrivains × 20 - quelques erreurs OK
        assert results['updates'] > 8  # 1 updater × 10 - quelques erreurs OK
        assert len(results['errors']) < 10  # Peu d'erreurs

        # Vitesse minimale (opérations par seconde)
        total_ops = results['reads'] + results['writes'] + results['updates']
        ops_per_second = total_ops / duration
        print(f"🚀 Opérations/seconde: {ops_per_second:.1f}")
        assert ops_per_second > 500  # Au moins 500 ops/sec

    def test_memory_usage_under_load(self, temp_db):
        """Test de l'usage mémoire sous charge"""
        import psutil
        import os

        process = psutil.Process(os.getpid())
        initial_memory = process.memory_info().rss / 1024 / 1024  # MB

        # Créer beaucoup de données
        livre_model = Livre(temp_db)

        for batch in range(10):  # 10 lots de 100 livres
            for i in range(100):
                book_data = {
                    'titre': f'Livre Mémoire {batch}-{i}',
                    'auteur': f'Auteur {batch}-{i}',
                    'resume': 'Lorem ipsum dolor sit amet, consectetur adipiscing elit. ' * 10
                }
                livre_model.create(book_data)

            # Faire des lectures intensives
            for _ in range(50):
                books = livre_model.get_all()

        final_memory = process.memory_info().rss / 1024 / 1024  # MB
        memory_increase = final_memory - initial_memory

        print(f"💾 Mémoire initiale: {initial_memory:.1f} MB")
        print(f"💾 Mémoire finale: {final_memory:.1f} MB")
        print(f"📈 Augmentation: {memory_increase:.1f} MB")

        # L'augmentation de mémoire doit rester raisonnable
        assert memory_increase < 100, f"Augmentation mémoire excessive: {memory_increase:.1f} MB"
```

## Automatisation des tests

### Configuration CI/CD

```yaml
# .github/workflows/tests.yml
name: Tests SQLite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        # Python 3.8/3.9 sont end-of-life en 2026 ; 3.10 fin de support oct. 2026.
        # Garder 3.11 (LTS de facto), 3.12 et 3.13.
        python-version: ['3.11', '3.12', '3.13']

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v5
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest-cov pytest-xdist pytest-benchmark

    - name: Run unit tests
      run: |
        pytest tests/unit/ -v --cov=app --cov-report=xml

    - name: Run integration tests
      run: |
        pytest tests/integration/ -v

    - name: Run performance tests
      run: |
        pytest tests/performance/ -v --benchmark-only

    - name: Run data integrity tests
      run: |
        pytest tests/data/ -v

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v4
      with:
        files: ./coverage.xml
        fail_ci_if_error: true

  stress-test:
    runs-on: ubuntu-latest
    needs: test

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.12'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest psutil

    - name: Run stress tests
      run: |
        pytest tests/stress/ -v --tb=short

    - name: Archive test results
      uses: actions/upload-artifact@v4
      if: failure()
      with:
        name: test-results
        path: |
          pytest.log
          *.db
```

### Script de test local

```bash
#!/bin/bash
# scripts/run_tests.sh

echo "🧪 Exécution de la suite de tests SQLite"

# Couleurs pour l'affichage
RED='\033[0;31m'  
GREEN='\033[0;32m'  
YELLOW='\033[1;33m'  
BLUE='\033[0;34m'  
NC='\033[0m' # No Color  

# Fonction pour afficher les résultats
print_result() {
    if [ $1 -eq 0 ]; then
        echo -e "${GREEN}✅ $2 - SUCCÈS${NC}"
    else
        echo -e "${RED}❌ $2 - ÉCHEC${NC}"
        return 1
    fi
}

# Créer le dossier de rapports
mkdir -p reports

echo -e "${BLUE}📋 Préparation de l'environnement...${NC}"

# Installer les dépendances de test si nécessaire
pip install -q pytest pytest-cov pytest-benchmark faker

echo -e "${BLUE}🔧 Tests unitaires...${NC}"  
pytest tests/unit/ -v --cov=app --cov-report=html:reports/coverage_html --cov-report=term  
print_result $? "Tests unitaires"  

echo -e "${BLUE}🔗 Tests d'intégration...${NC}"  
pytest tests/integration/ -v  
print_result $? "Tests d'intégration"  

echo -e "${BLUE}📊 Tests de performance...${NC}"  
pytest tests/performance/ -v --benchmark-only --benchmark-html=reports/benchmark.html  
print_result $? "Tests de performance"  

echo -e "${BLUE}🛡️ Tests d'intégrité des données...${NC}"  
pytest tests/data/ -v  
print_result $? "Tests d'intégrité"  

echo -e "${BLUE}🚨 Tests de régression...${NC}"  
pytest tests/regression/ -v  
print_result $? "Tests de régression"  

# Tests de stress (optionnels, plus longs)
read -p "Exécuter les tests de stress? (y/N): " -n 1 -r  
echo  
if [[ $REPLY =~ ^[Yy]$ ]]; then  
    echo -e "${YELLOW}⚡ Tests de stress...${NC}"
    pytest tests/stress/ -v
    print_result $? "Tests de stress"
fi

echo -e "${GREEN}📈 Rapports générés dans le dossier 'reports/'${NC}"  
echo -e "${BLUE}🌐 Ouvrir reports/coverage_html/index.html pour voir la couverture${NC}"  
echo -e "${BLUE}📊 Ouvrir reports/benchmark.html pour voir les benchmarks${NC}"  

echo -e "${GREEN}✨ Suite de tests terminée!${NC}"
```

### Configuration pytest avancée

```ini
# pytest.ini
[tool:pytest]
minversion = 6.0  
addopts =  
    -ra
    --strict-markers
    --strict-config
    --disable-warnings
    --tb=short
testpaths = tests  
markers =  
    unit: Tests unitaires rapides
    integration: Tests d'intégration
    performance: Tests de performance
    stress: Tests de charge
    slow: Tests lents (> 5 secondes)
    regression: Tests de régression
    benchmark: Tests de benchmark
filterwarnings =
    ignore::UserWarning
    ignore::DeprecationWarning

# Configuration de couverture
[coverage:run]
source = app  
omit =  
    */tests/*
    */venv/*
    */migrations/*
    run.py

[coverage:report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
```

### Makefile pour l'automatisation

```makefile
# Makefile
.PHONY: test test-unit test-integration test-performance test-all clean install

# Installation
install:
	pip install -r requirements.txt
	pip install pytest pytest-cov pytest-benchmark faker psutil

# Tests rapides (développement)
test:
	pytest tests/unit/ tests/integration/ -v

# Tests unitaires seulement
test-unit:
	pytest tests/unit/ -v --cov=app

# Tests d'intégration seulement
test-integration:
	pytest tests/integration/ -v

# Tests de performance
test-performance:
	pytest tests/performance/ -v --benchmark-only

# Suite complète de tests
test-all:
	pytest tests/ -v --cov=app --cov-report=html --cov-report=term

# Tests de stress
test-stress:
	pytest tests/stress/ -v

# Tests en parallèle (plus rapide)
test-parallel:
	pytest tests/ -n auto --cov=app

# Nettoyage
clean:
	rm -rf .pytest_cache
	rm -rf .coverage
	rm -rf htmlcov
	rm -rf reports
	find . -type d -name __pycache__ -delete
	find . -name "*.pyc" -delete
	find . -name "*.db" -delete

# Lint et formatage
lint:
	flake8 app tests
	black --check app tests

format:
	black app tests
	isort app tests

# Vérification complète (CI)
ci: clean install lint test-all
```

## Meilleures pratiques pour les tests SQLite

### ✅ Bonnes pratiques

1. **Isolation des tests**
   - Utiliser des bases en mémoire ou temporaires
   - Nettoyer après chaque test
   - Éviter les dépendances entre tests

2. **Performance des tests**
   - Privilégier `:memory:` pour les tests unitaires
   - Utiliser des transactions pour les rollbacks rapides
   - Créer des fixtures avec des données minimales

3. **Couverture complète**
   - Tester les cas normaux ET les cas d'erreur
   - Vérifier les contraintes de base de données
   - Tester les performances sous charge

4. **Automatisation**
   - Intégrer dans CI/CD
   - Tests automatiques à chaque commit
   - Rapports de couverture et performance

### ❌ Écueils à éviter

- **Tests dépendants** de l'ordre d'exécution
- **Données persistantes** entre tests
- **Mocks excessifs** qui masquent les vrais problèmes
- **Tests trop longs** qui ralentissent le développement
- **Couverture incomplète** des cas d'erreur

## Monitoring et métriques de tests

### Dashboard de tests en temps réel

```python
# scripts/test_dashboard.py
import json  
import time  
from datetime import datetime, timedelta  
import sqlite3  
import subprocess  
import os  

class TestDashboard:
    """Dashboard pour surveiller les tests et métriques"""

    def __init__(self, db_path="reports/test_metrics.db"):
        self.db_path = db_path
        self.init_metrics_db()

    def init_metrics_db(self):
        """Initialise la base de métriques de tests."""
        # `os.path.dirname('file.db')` retourne '' → makedirs('') = FileNotFoundError.
        parent = os.path.dirname(self.db_path)
        if parent:
            os.makedirs(parent, exist_ok=True)

        with sqlite3.connect(self.db_path) as conn:
            conn.execute('''
                CREATE TABLE IF NOT EXISTS test_runs (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                    test_type TEXT NOT NULL,
                    duration REAL NOT NULL,
                    tests_total INTEGER NOT NULL,
                    tests_passed INTEGER NOT NULL,
                    tests_failed INTEGER NOT NULL,
                    coverage_percent REAL,
                    performance_score REAL,
                    git_commit TEXT,
                    branch TEXT
                )
            ''')

            conn.execute('''
                CREATE TABLE IF NOT EXISTS test_details (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    test_run_id INTEGER,
                    test_name TEXT NOT NULL,
                    test_file TEXT NOT NULL,
                    status TEXT NOT NULL,
                    duration REAL,
                    error_message TEXT,
                    FOREIGN KEY (test_run_id) REFERENCES test_runs (id)
                )
            ''')

    def run_and_record_tests(self, test_type="all"):
        """Exécute les tests et enregistre les métriques"""
        start_time = time.time()

        # Commandes de test selon le type
        test_commands = {
            "unit": ["pytest", "tests/unit/", "-v", "--json-report", "--json-report-file=reports/unit_report.json"],
            "integration": ["pytest", "tests/integration/", "-v", "--json-report", "--json-report-file=reports/integration_report.json"],
            "performance": ["pytest", "tests/performance/", "-v", "--benchmark-json=reports/benchmark.json"],
            "all": ["pytest", "tests/", "-v", "--cov=app", "--cov-report=json:reports/coverage.json", "--json-report", "--json-report-file=reports/full_report.json"]
        }

        if test_type not in test_commands:
            raise ValueError(f"Type de test inconnu: {test_type}")

        print(f"🚀 Exécution des tests: {test_type}")

        # Exécuter les tests
        try:
            result = subprocess.run(
                test_commands[test_type],
                capture_output=True,
                text=True,
                timeout=600  # 10 minutes max
            )

            duration = time.time() - start_time

            # Analyser les résultats
            metrics = self.parse_test_results(test_type, result, duration)

            # Enregistrer en base
            test_run_id = self.save_test_run(test_type, metrics)

            # Afficher le résumé
            self.display_summary(metrics)

            return test_run_id, metrics

        except subprocess.TimeoutExpired:
            print("❌ Tests interrompus (timeout)")
            return None, None
        except Exception as e:
            print(f"❌ Erreur lors de l'exécution: {e}")
            return None, None

    def parse_test_results(self, test_type, result, duration):
        """Parse les résultats de tests"""
        metrics = {
            'duration': duration,
            'tests_total': 0,
            'tests_passed': 0,
            'tests_failed': 0,
            'coverage_percent': None,
            'performance_score': None,
            'git_commit': self.get_git_commit(),
            'branch': self.get_git_branch()
        }

        # Parser le rapport JSON si disponible
        report_files = {
            'unit': 'reports/unit_report.json',
            'integration': 'reports/integration_report.json',
            'all': 'reports/full_report.json'
        }

        if test_type in report_files and os.path.exists(report_files[test_type]):
            try:
                with open(report_files[test_type], 'r') as f:
                    report = json.load(f)

                summary = report.get('summary', {})
                metrics['tests_total'] = summary.get('total', 0)
                metrics['tests_passed'] = summary.get('passed', 0)
                metrics['tests_failed'] = summary.get('failed', 0)

            except Exception as e:
                print(f"⚠️ Erreur parsing rapport JSON: {e}")

        # Parser la couverture
        if test_type == 'all' and os.path.exists('reports/coverage.json'):
            try:
                with open('reports/coverage.json', 'r') as f:
                    coverage = json.load(f)
                metrics['coverage_percent'] = coverage.get('totals', {}).get('percent_covered', 0)
            except Exception as e:
                print(f"⚠️ Erreur parsing couverture: {e}")

        # Parser les benchmarks
        if test_type == 'performance' and os.path.exists('reports/benchmark.json'):
            try:
                with open('reports/benchmark.json', 'r') as f:
                    benchmarks = json.load(f)

                # Calculer un score de performance simple
                total_time = sum(b.get('stats', {}).get('mean', 0) for b in benchmarks.get('benchmarks', []))
                metrics['performance_score'] = 1000 / total_time if total_time > 0 else 0

            except Exception as e:
                print(f"⚠️ Erreur parsing benchmarks: {e}")

        return metrics

    def save_test_run(self, test_type, metrics):
        """Sauvegarde les métriques en base"""
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.execute('''
                INSERT INTO test_runs (test_type, duration, tests_total, tests_passed,
                                     tests_failed, coverage_percent, performance_score,
                                     git_commit, branch)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                test_type, metrics['duration'], metrics['tests_total'],
                metrics['tests_passed'], metrics['tests_failed'],
                metrics['coverage_percent'], metrics['performance_score'],
                metrics['git_commit'], metrics['branch']
            ))

            return cursor.lastrowid

    def get_git_commit(self):
        """Récupère le hash du commit actuel."""
        # Cibler FileNotFoundError (git absent) et CalledProcessError ;
        # `bare except:` masquerait aussi KeyboardInterrupt/SystemExit.
        try:
            result = subprocess.run(['git', 'rev-parse', 'HEAD'],
                                  capture_output=True, text=True)
            return result.stdout.strip()[:8] if result.returncode == 0 else None
        except (FileNotFoundError, subprocess.SubprocessError):
            return None

    def get_git_branch(self):
        """Récupère la branche actuelle."""
        try:
            result = subprocess.run(['git', 'branch', '--show-current'],
                                  capture_output=True, text=True)
            return result.stdout.strip() if result.returncode == 0 else None
        except (FileNotFoundError, subprocess.SubprocessError):
            return None

    def display_summary(self, metrics):
        """Affiche un résumé des résultats"""
        print("\n" + "="*60)
        print("📊 RÉSUMÉ DES TESTS")
        print("="*60)
        print(f"⏱️  Durée: {metrics['duration']:.2f}s")
        print(f"📈 Total: {metrics['tests_total']} tests")
        print(f"✅ Réussis: {metrics['tests_passed']}")
        print(f"❌ Échecs: {metrics['tests_failed']}")

        if metrics['coverage_percent'] is not None:
            print(f"🛡️  Couverture: {metrics['coverage_percent']:.1f}%")

        if metrics['performance_score'] is not None:
            print(f"🚀 Score perf: {metrics['performance_score']:.1f}")

        if metrics['git_commit']:
            print(f"📝 Commit: {metrics['git_commit']}")

        # Statut global
        success_rate = metrics['tests_passed'] / metrics['tests_total'] if metrics['tests_total'] > 0 else 0
        if success_rate == 1.0:
            print("🎉 TOUS LES TESTS RÉUSSIS!")
        elif success_rate > 0.9:
            print("✅ Tests largement réussis")
        elif success_rate > 0.7:
            print("⚠️  Quelques échecs à corriger")
        else:
            print("🚨 Nombreux échecs - action requise")

        print("="*60)

    def generate_trend_report(self, days=30):
        """Génère un rapport de tendance.

        ⚠️ `days` est concaténé dans la chaîne SQL via `?` (et non format-string
           ni f-string) pour éviter toute injection si la valeur vient un jour
           d'un CLI argument. Pour `DATE('now', modifier)`, le modifier doit
           être une chaîne SQLite — on construit '-N days' avec `||`.
        """
        if not isinstance(days, int) or days < 0:
            raise ValueError(f"days doit être un entier positif, reçu : {days!r}")

        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.execute('''
                SELECT
                    DATE(timestamp) as date,
                    test_type,
                    AVG(duration) as avg_duration,
                    AVG(CAST(tests_passed AS REAL) / tests_total * 100) as success_rate,
                    AVG(coverage_percent) as avg_coverage,
                    COUNT(*) as runs_count
                FROM test_runs
                WHERE timestamp > DATE('now', '-' || ? || ' days')
                GROUP BY DATE(timestamp), test_type
                ORDER BY date DESC, test_type
            ''', (days,))

            trends = cursor.fetchall()

            print(f"\n📈 TENDANCES DES TESTS ({days} derniers jours)")
            print("-" * 80)
            print(f"{'Date':<12} {'Type':<12} {'Durée':<8} {'Succès':<8} {'Couv.':<8} {'Runs':<6}")
            print("-" * 80)

            for trend in trends:
                date, test_type, duration, success, coverage, runs = trend
                print(f"{date:<12} {test_type:<12} {duration:.1f}s    {success:.1f}%    {coverage or 0:.1f}%    {runs}")

def generate_html_report():
    """Génère un rapport HTML des tests"""
    dashboard = TestDashboard()

    html_template = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>Dashboard Tests SQLite</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; }
            .metric { display: inline-block; margin: 10px; padding: 15px; border-radius: 5px; min-width: 150px; text-align: center; }
            .success { background-color: #d4edda; color: #155724; }
            .warning { background-color: #fff3cd; color: #856404; }
            .danger { background-color: #f8d7da; color: #721c24; }
            .chart { margin: 20px 0; }
            table { border-collapse: collapse; width: 100%; }
            th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
            th { background-color: #f2f2f2; }
        </style>
    </head>
    <body>
        <h1>🧪 Dashboard Tests SQLite</h1>
        <p>Généré le: {timestamp}</p>

        <h2>📊 Métriques Récentes</h2>
        <div id="metrics">
            {metrics_html}
        </div>

        <h2>📈 Historique des Tests</h2>
        <table>
            <thead>
                <tr>
                    <th>Date</th>
                    <th>Type</th>
                    <th>Durée</th>
                    <th>Tests</th>
                    <th>Succès</th>
                    <th>Couverture</th>
                    <th>Commit</th>
                </tr>
            </thead>
            <tbody>
                {history_html}
            </tbody>
        </table>
    </body>
    </html>
    """

    # Récupérer les données récentes
    with sqlite3.connect(dashboard.db_path) as conn:
        # Métriques récentes
        cursor = conn.execute('''
            SELECT * FROM test_runs
            ORDER BY timestamp DESC
            LIMIT 10
        ''')
        recent_runs = cursor.fetchall()

        # Générer le HTML des métriques
        metrics_html = ""
        if recent_runs:
            latest = recent_runs[0]
            success_rate = latest[5] / latest[4] * 100 if latest[4] > 0 else 0

            status_class = "success" if success_rate == 100 else "warning" if success_rate > 90 else "danger"

            metrics_html = f"""
            <div class="metric {status_class}">
                <h3>Dernier Test</h3>
                <p>{latest[2]} ({latest[1]})</p>
                <p>{success_rate:.1f}% réussis</p>
            </div>
            <div class="metric">
                <h3>Durée</h3>
                <p>{latest[3]:.1f}s</p>
            </div>
            <div class="metric">
                <h3>Couverture</h3>
                <p>{latest[7] or 0:.1f}%</p>
            </div>
            """

        # Générer le HTML de l'historique
        history_html = ""
        for run in recent_runs:
            success_rate = run[5] / run[4] * 100 if run[4] > 0 else 0
            status_class = "success" if success_rate == 100 else "warning" if success_rate > 90 else "danger"

            history_html += f"""
            <tr class="{status_class}">
                <td>{run[1][:16]}</td>
                <td>{run[2]}</td>
                <td>{run[3]:.1f}s</td>
                <td>{run[4]}</td>
                <td>{success_rate:.1f}%</td>
                <td>{run[7] or 0:.1f}%</td>
                <td>{run[9] or 'N/A'}</td>
            </tr>
            """

    # Générer le fichier HTML
    html_content = html_template.format(
        timestamp=datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        metrics_html=metrics_html,
        history_html=history_html
    )

    os.makedirs('reports', exist_ok=True)
    with open('reports/test_dashboard.html', 'w', encoding='utf-8') as f:
        f.write(html_content)

    print("📊 Dashboard HTML généré: reports/test_dashboard.html")

if __name__ == '__main__':
    import sys

    dashboard = TestDashboard()

    if len(sys.argv) > 1:
        command = sys.argv[1]

        if command == "run":
            test_type = sys.argv[2] if len(sys.argv) > 2 else "all"
            dashboard.run_and_record_tests(test_type)

        elif command == "trends":
            days = int(sys.argv[2]) if len(sys.argv) > 2 else 30
            dashboard.generate_trend_report(days)

        elif command == "html":
            generate_html_report()

        else:
            print("Commandes disponibles: run [type], trends [days], html")
    else:
        # Exécution par défaut
        dashboard.run_and_record_tests("all")
        dashboard.generate_trend_report(7)
        generate_html_report()
```

### Tests de sécurité spécifiques à SQLite

```python
# tests/security/test_security.py
import pytest  
import sqlite3  
import json  
import os  # nécessaire pour pytest.mark.skipif(os.name == 'nt', ...) plus bas  
from app.models import Livre, Utilisateur  

class TestSQLiteSecurity:
    """Tests de sécurité spécifiques à SQLite"""

    def test_sql_injection_prevention(self, memory_db):
        """Test de prévention d'injection SQL"""
        livre_model = Livre(memory_db)

        # Créer un livre test
        livre_model.create({
            'titre': 'Test Book',
            'auteur': 'Test Author'
        })

        # Tentatives d'injection SQL
        malicious_inputs = [
            "'; DROP TABLE livres; --",
            "' OR '1'='1",
            "'; UPDATE livres SET titre='HACKED'; --",
            "' UNION SELECT * FROM utilisateurs --",
            "1; DELETE FROM livres; --"
        ]

        # ⚠️ Pour rouvrir une 2e connexion sur la même base en mémoire partagée,
        #    il FAUT passer uri=True (sinon 'file::memory:?cache=shared' est traité
        #    comme un nom de fichier local et une base distincte est créée).
        uri_mode = memory_db.startswith('file:')

        for malicious_input in malicious_inputs:
            try:
                # Ces requêtes ne doivent pas causer de dégâts
                books = livre_model.get_all({'search': malicious_input})

                # Vérifier que la table existe toujours (même base que livre_model)
                with sqlite3.connect(memory_db, uri=uri_mode) as conn:
                    cursor = conn.execute("SELECT COUNT(*) FROM livres")
                    count = cursor.fetchone()[0]
                    assert count > 0, f"Table livres supprimée par injection: {malicious_input}"

                    # Vérifier qu'aucune donnée n'a été modifiée
                    cursor = conn.execute("SELECT titre FROM livres WHERE titre = 'HACKED'")
                    hacked = cursor.fetchall()
                    assert len(hacked) == 0, f"Données modifiées par injection: {malicious_input}"

            except sqlite3.Error as e:
                # Une erreur SQL est acceptable (rejet de l'injection), mais pas
                # une corruption silencieuse. On capture spécifiquement sqlite3.Error
                # pour éviter de masquer KeyboardInterrupt / AssertionError.
                print(f"Injection bloquée (normal): {malicious_input} -> {e}")

    def test_file_path_validation(self):
        """Test de validation des chemins de fichier côté application.

        ⚠️ ATTENTION : `sqlite3.connect(path)` accepte TOUT chemin sans lever
           d'exception — il crée le fichier s'il n'existe pas. Donc tester
           que `sqlite3.connect(malicious_path)` lève une exception est FAUX.

        La protection contre les injections de chemin doit donc se faire
        AU NIVEAU DE L'APPLICATION (validation regex, whitelist de dossiers,
        os.path.realpath() + vérification que le chemin reste dans une
        racine autorisée), pas en comptant sur SQLite.
        """
        import re
        from pathlib import Path

        ROOT_AUTORISE = Path('/app/data').resolve()

        def chemin_securise(chemin: str) -> bool:
            """Valide qu'un chemin reste dans le dossier autorisé.

            ⚠️ `pathlib` interprète les séparateurs selon l'OS — sur Linux,
               `..\\..\\foo` est un nom de FICHIER unique, donc on doit
               rejeter explicitement les patterns de traversal Windows pour
               une protection portable.
            """
            try:
                # 1. Refuser les patterns de traversal Windows (sur Linux, `\` n'est
                #    pas séparateur → `..\..\foo` passerait `relative_to()`)
                if '..\\' in chemin or chemin.startswith('..'):
                    return False
                # 2. Résoudre et vérifier qu'on reste sous la racine autorisée
                resolved = (ROOT_AUTORISE / chemin).resolve()
                resolved.relative_to(ROOT_AUTORISE)
                # 3. Refuser les caractères shell potentiellement dangereux
                if re.search(r'[;&|`$()<>\n\\]', chemin):
                    return False
                return True
            except (ValueError, OSError):
                return False

        # Cas qui DOIVENT être rejetés
        malicious = [
            "../../../etc/passwd",
            "/etc/shadow",
            "..\\..\\windows\\system32\\config\\sam",
            "||whoami",
            "; rm -rf /",
        ]
        for path in malicious:
            assert not chemin_securise(path), f"Chemin malicieux non détecté : {path}"

        # Cas qui doivent être acceptés
        for path in ['bibliotheque.db', 'sub/db.sqlite']:
            assert chemin_securise(path), f"Chemin valide rejeté : {path}"

    def test_privilege_escalation_prevention(self, client, db_with_data):
        """Test de prévention d'escalade de privilèges"""
        # Se connecter comme utilisateur normal
        client.post('/login', data={
            'email': 'alice@test.com',
            'password': 'alice123'
        })

        # Tenter des actions admin (doivent échouer)
        admin_actions = [
            ('POST', '/api/books', {'titre': 'Livre Non Autorisé', 'auteur': 'Auteur'}),
            ('PUT', '/api/books/1', {'titre': 'Modification Non Autorisée'}),
            ('DELETE', '/api/books/1', {}),
            ('GET', '/api/loans/overdue', {})
        ]

        for method, endpoint, data in admin_actions:
            if method == 'GET':
                response = client.get(endpoint)
            elif method == 'POST':
                response = client.post(endpoint,
                                     data=json.dumps(data),
                                     content_type='application/json')
            elif method == 'PUT':
                response = client.put(endpoint,
                                    data=json.dumps(data),
                                    content_type='application/json')
            elif method == 'DELETE':
                response = client.delete(endpoint)

            # Doit être refusé (redirect, 401, 403, etc.)
            assert response.status_code in [302, 401, 403], \
                f"Action admin autorisée pour utilisateur normal: {method} {endpoint}"

    def test_data_leakage_prevention(self, client, db_with_data):
        """Test de prévention de fuite de données sensibles"""
        # Créer un utilisateur et se connecter
        response = client.post('/register', data={
            'nom': 'Test User',
            'email': 'testuser@example.com',
            'password': 'testpass123'
        })

        client.post('/login', data={
            'email': 'testuser@example.com',
            'password': 'testpass123'
        })

        # Vérifier qu'on ne peut pas accéder aux données d'autres utilisateurs
        response = client.get('/api/loans')
        assert response.status_code == 200

        loans = json.loads(response.data)
        # Tous les emprunts retournés doivent appartenir à l'utilisateur connecté
        # (Dans notre cas, l'utilisateur test n'a pas d'emprunts)
        assert len(loans) == 0 or all(loan.get('utilisateur_id') == 3 for loan in loans)

    def test_session_security(self, client):
        """Test de sécurité des sessions"""
        # Se connecter
        response = client.post('/login', data={
            'email': 'alice@test.com',
            'password': 'alice123'
        })

        # Vérifier que la session est sécurisée
        with client.session_transaction() as sess:
            assert 'user_id' in sess
            # Le mot de passe ne doit pas être stocké en session
            assert 'password' not in sess
            assert 'mot_de_passe' not in sess

        # Se déconnecter
        client.get('/logout')

        # Vérifier que la session est nettoyée
        with client.session_transaction() as sess:
            assert 'user_id' not in sess

    @pytest.mark.skipif(os.name == 'nt', reason="Permissions Unix non applicables sur Windows")
    def test_database_file_permissions(self, temp_db):
        """Test des permissions du fichier de base de données.

        ⚠️ Test Unix-only : les bits POSIX (S_IWOTH, S_IXUSR) ne s'appliquent
           pas sur Windows. Sur Windows, la sécurité passe par des ACLs NTFS
           interrogeables via `win32security` (pywin32), pas par stat.S_*.
        """
        import os
        import stat

        # Vérifier que le fichier de base existe
        assert os.path.exists(temp_db)

        # Vérifier les permissions (pas trop permissives)
        file_stat = os.stat(temp_db)
        permissions = stat.filemode(file_stat.st_mode)

        # Le fichier ne doit pas être accessible en écriture par tous
        assert not (file_stat.st_mode & stat.S_IWOTH), \
            f"Fichier DB accessible en écriture par tous: {permissions}"

        # Le fichier ne doit pas être exécutable (par le propriétaire)
        assert not (file_stat.st_mode & stat.S_IXUSR), \
            f"Fichier DB exécutable: {permissions}"
```

## Conclusion

Les tests sont un élément crucial de tout projet utilisant SQLite. Ils garantissent la fiabilité, la performance et la maintenabilité de votre application.

Cette approche complète des tests - des tests unitaires simples aux tests de charge avancés - vous donne tous les outils nécessaires pour :

### ✅ Avantages d'une suite de tests complète

1. **Confiance dans le code** : Détection précoce des bugs
2. **Refactoring sécurisé** : Modifications sans crainte de régression
3. **Documentation vivante** : Les tests servent d'exemples d'usage
4. **Performance garantie** : Surveillance continue des performances
5. **Qualité maintenue** : Standards élevés tout au long du développement

### 🎯 Points clés à retenir

- **SQLite est parfait pour les tests** : Rapidité, isolation, simplicité
- **Automatisation complète** : CI/CD, métriques, rapports
- **Tests multiples niveaux** : Unitaires, intégration, performance, sécurité
- **Monitoring continu** : Dashboard, tendances, alertes
- **Sécurité incluse** : Tests d'injection, privilèges, fuites de données

### 🚀 Mise en pratique

Pour implémenter cette stratégie de tests dans votre projet :

1. **Commencez simple** : Tests unitaires des modèles
2. **Ajoutez progressivement** : Intégration, puis performance
3. **Automatisez tôt** : Scripts, CI/CD, métriques
4. **Surveillez en continu** : Dashboard, alertes, tendances
5. **Améliorez constamment** : Nouveaux tests, optimisations

### 📈 Impact sur la qualité

Avec cette approche, vous obtiendrez :

- **Couverture de code** > 90%
- **Temps de détection** des bugs en minutes
- **Confiance** pour déployer rapidement
- **Performance** surveillée et optimisée
- **Sécurité** validée automatiquement

Les tests ne sont pas un coût mais un **investissement rentable** qui vous fait gagner du temps et évite les problèmes en production.

---

**🎉 Félicitations !**

Vous maîtrisez maintenant tous les aspects des tests avec SQLite, du plus simple au plus avancé. Votre application est maintenant robuste, performante et prête pour la production !

Cette connaissance approfondie des tests SQLite vous permettra de développer des applications fiables et maintenables, avec la confiance nécessaire pour évoluer et s'adapter aux besoins changeants.

---

🎉 **Vous avez terminé la formation SQLite3 !**

- 📋 **Fiche de révision condensée** : [Notes.md](/Notes.md) — l'ensemble des concepts, pièges et syntaxes critiques sur une seule page (cheat-sheet)
- 📖 **Retour au sommaire** : [SOMMAIRE.md](/SOMMAIRE.md)
- 🏠 **Page d'accueil** : [README.md](/README.md)
