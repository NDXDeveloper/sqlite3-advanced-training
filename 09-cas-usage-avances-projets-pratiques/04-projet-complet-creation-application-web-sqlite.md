🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.4 : Projet complet : création d'une application web avec SQLite

## Introduction : Construire une vraie application web

Dans cette section, nous allons créer ensemble une **application web complète** : un système de gestion de bibliothèque personnelle. Ce projet vous permettra de mettre en pratique tous les concepts SQLite appris jusqu'ici dans un contexte réel.

### Notre projet : "Ma Bibliothèque"

Nous allons développer une application qui permet de :

- **Gérer une collection** de livres personnels
- **Rechercher et filtrer** les livres
- **Suivre les emprunts** et retours
- **Générer des statistiques** de lecture
- **Exporter des données** pour sauvegarde

### Technologies utilisées

- **Backend** : Python avec Flask (framework web simple)
- **Base de données** : SQLite (bien sûr !)
- **Frontend** : HTML/CSS/JavaScript (simple et efficace)
- **API** : REST pour les échanges de données

### Pourquoi ce projet ?

Ce projet est idéal pour apprendre car il couvre :

- Conception de base de données relationnelle
- Développement d'API REST
- Interface utilisateur interactive
- Gestion des fichiers et uploads
- Authentification simple
- Déploiement d'application

## Architecture de l'application

### Structure du projet

```
ma_bibliotheque/
├── app/
│   ├── __init__.py              # Configuration Flask
│   ├── models.py                # Modèles de données
│   ├── routes.py                # Routes de l'API
│   ├── database.py              # Gestionnaire de base
│   └── auth.py                  # Authentification
├── static/
│   ├── css/
│   │   └── style.css           # Styles CSS
│   ├── js/
│   │   └── app.js              # JavaScript frontend
│   └── uploads/                # Images de couvertures
├── templates/
│   ├── base.html               # Template de base
│   ├── index.html              # Page d'accueil
│   ├── books.html              # Liste des livres
│   └── stats.html              # Statistiques
├── data/
│   └── bibliotheque.db         # Base SQLite
├── requirements.txt            # Dépendances Python
├── config.py                   # Configuration
└── run.py                      # Point d'entrée
```

## Étape 1 : Conception de la base de données

### Analyse des besoins

Avant de créer les tables, analysons ce que nous devons stocker :

**Entités principales :**
- **Livres** : titre, auteur, ISBN, genre, etc.
- **Utilisateurs** : nom, email, mot de passe
- **Emprunts** : qui a emprunté quoi et quand
- **Genres** : catégories de livres

**Relations :**
- Un livre appartient à un genre
- Un utilisateur peut emprunter plusieurs livres
- Un livre peut être emprunté par plusieurs utilisateurs (historique)

### Schéma de base de données

```python
# app/database.py
import sqlite3
import os
from datetime import datetime
from werkzeug.security import generate_password_hash

class DatabaseManager:
    def __init__(self, db_path="data/bibliotheque.db"):
        self.db_path = db_path
        self.ensure_directory()
        self.init_database()

    def ensure_directory(self):
        """S'assure que le dossier data existe"""
        os.makedirs(os.path.dirname(self.db_path), exist_ok=True)

    def init_database(self):
        """Initialise la base de données avec toutes les tables"""
        with sqlite3.connect(self.db_path) as conn:
            # Activer les clés étrangères
            conn.execute("PRAGMA foreign_keys = ON")

            # Table des genres
            conn.execute('''
                CREATE TABLE IF NOT EXISTS genres (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    nom TEXT UNIQUE NOT NULL,
                    description TEXT,
                    couleur TEXT DEFAULT '#3498db',
                    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
                )
            ''')

            # Table des utilisateurs
            conn.execute('''
                CREATE TABLE IF NOT EXISTS utilisateurs (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    nom TEXT NOT NULL,
                    email TEXT UNIQUE NOT NULL,
                    mot_de_passe_hash TEXT NOT NULL,
                    est_admin BOOLEAN DEFAULT 0,
                    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                    last_login DATETIME
                )
            ''')

            # Table des livres
            conn.execute('''
                CREATE TABLE IF NOT EXISTS livres (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    titre TEXT NOT NULL,
                    auteur TEXT NOT NULL,
                    isbn TEXT UNIQUE,
                    genre_id INTEGER,
                    annee_publication INTEGER,
                    nombre_pages INTEGER,
                    langue TEXT DEFAULT 'fr',
                    resume TEXT,
                    couverture_url TEXT,
                    date_ajout DATETIME DEFAULT CURRENT_TIMESTAMP,
                    disponible BOOLEAN DEFAULT 1,
                    note_moyenne REAL DEFAULT 0,
                    nombre_evaluations INTEGER DEFAULT 0,
                    FOREIGN KEY (genre_id) REFERENCES genres (id)
                )
            ''')

            # Table des emprunts
            conn.execute('''
                CREATE TABLE IF NOT EXISTS emprunts (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    livre_id INTEGER NOT NULL,
                    utilisateur_id INTEGER NOT NULL,
                    date_emprunt DATETIME DEFAULT CURRENT_TIMESTAMP,
                    date_retour_prevue DATETIME NOT NULL,
                    date_retour_effective DATETIME,
                    statut TEXT DEFAULT 'actif',  -- 'actif', 'retourne', 'retard'
                    commentaire TEXT,
                    FOREIGN KEY (livre_id) REFERENCES livres (id),
                    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs (id)
                )
            ''')

            # Table des évaluations
            conn.execute('''
                CREATE TABLE IF NOT EXISTS evaluations (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    livre_id INTEGER NOT NULL,
                    utilisateur_id INTEGER NOT NULL,
                    note INTEGER CHECK (note >= 1 AND note <= 5),
                    commentaire TEXT,
                    date_evaluation DATETIME DEFAULT CURRENT_TIMESTAMP,
                    FOREIGN KEY (livre_id) REFERENCES livres (id),
                    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs (id),
                    UNIQUE(livre_id, utilisateur_id)  -- Un seul avis par utilisateur par livre
                )
            ''')

            # Index pour optimiser les requêtes
            self.create_indexes(conn)

            # Données de base
            self.insert_sample_data(conn)

    def create_indexes(self, conn):
        """Crée les index pour optimiser les performances"""
        indexes = [
            "CREATE INDEX IF NOT EXISTS idx_livres_titre ON livres(titre)",
            "CREATE INDEX IF NOT EXISTS idx_livres_auteur ON livres(auteur)",
            "CREATE INDEX IF NOT EXISTS idx_livres_genre ON livres(genre_id)",
            "CREATE INDEX IF NOT EXISTS idx_livres_disponible ON livres(disponible)",
            "CREATE INDEX IF NOT EXISTS idx_emprunts_statut ON emprunts(statut)",
            "CREATE INDEX IF NOT EXISTS idx_emprunts_dates ON emprunts(date_emprunt, date_retour_prevue)",
            "CREATE INDEX IF NOT EXISTS idx_utilisateurs_email ON utilisateurs(email)",
            "CREATE INDEX IF NOT EXISTS idx_evaluations_livre ON evaluations(livre_id)"
        ]

        for index_sql in indexes:
            conn.execute(index_sql)

    def insert_sample_data(self, conn):
        """Insert des données d'exemple pour commencer"""

        # Vérifier si on a déjà des données
        cursor = conn.execute("SELECT COUNT(*) FROM genres")
        if cursor.fetchone()[0] > 0:
            return  # Données déjà présentes

        # Genres par défaut
        genres_data = [
            ('Roman', 'Fiction littéraire', '#e74c3c'),
            ('Science-Fiction', 'Littérature de science-fiction', '#9b59b6'),
            ('Fantasy', 'Littérature fantastique', '#8e44ad'),
            ('Policier', 'Romans policiers et thrillers', '#34495e'),
            ('Biographie', 'Biographies et autobiographies', '#16a085'),
            ('Histoire', 'Livres d\'histoire', '#f39c12'),
            ('Sciences', 'Ouvrages scientifiques', '#27ae60'),
            ('Informatique', 'Livres techniques informatique', '#2980b9')
        ]

        conn.executemany(
            "INSERT INTO genres (nom, description, couleur) VALUES (?, ?, ?)",
            genres_data
        )

        # Utilisateur admin par défaut
        admin_hash = generate_password_hash('admin123')
        conn.execute('''
            INSERT INTO utilisateurs (nom, email, mot_de_passe_hash, est_admin)
            VALUES (?, ?, ?, ?)
        ''', ('Administrateur', 'admin@bibliotheque.local', admin_hash, 1))

        # Quelques livres d'exemple
        livres_data = [
            ('Le Petit Prince', 'Antoine de Saint-Exupéry', '9782070408504', 1, 1943, 96, 'fr',
             'L\'histoire d\'un petit prince qui voyage de planète en planète.'),
            ('Dune', 'Frank Herbert', '9782221221051', 2, 1965, 688, 'fr',
             'Sur la planète Arrakis, seule source de l\'épice la plus précieuse de l\'univers.'),
            ('Le Seigneur des Anneaux', 'J.R.R. Tolkien', '9782070612888', 3, 1954, 1216, 'fr',
             'L\'épopée de Frodon et de l\'Anneau unique en Terre du Milieu.'),
            ('Clean Code', 'Robert C. Martin', '9780132350884', 8, 2008, 464, 'en',
             'Guide pour écrire du code propre et maintenable.')
        ]

        conn.executemany('''
            INSERT INTO livres (titre, auteur, isbn, genre_id, annee_publication,
                              nombre_pages, langue, resume)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ''', livres_data)

        conn.commit()
        print("✅ Base de données initialisée avec des données d'exemple")
```

## Étape 2 : Modèles de données (Models)

### Création des classes modèles

```python
# app/models.py
import sqlite3
from datetime import datetime, timedelta
from werkzeug.security import check_password_hash, generate_password_hash

class BaseModel:
    """Classe de base pour tous les modèles"""

    def __init__(self, db_path="data/bibliotheque.db"):
        self.db_path = db_path

    def get_connection(self):
        """Retourne une connexion à la base"""
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row  # Pour accès par nom de colonne
        conn.execute("PRAGMA foreign_keys = ON")
        return conn

class Livre(BaseModel):
    """Modèle pour gérer les livres"""

    def create(self, livre_data):
        """Crée un nouveau livre"""
        with self.get_connection() as conn:
            cursor = conn.execute('''
                INSERT INTO livres (titre, auteur, isbn, genre_id, annee_publication,
                                  nombre_pages, langue, resume, couverture_url)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                livre_data['titre'], livre_data['auteur'], livre_data.get('isbn'),
                livre_data.get('genre_id'), livre_data.get('annee_publication'),
                livre_data.get('nombre_pages'), livre_data.get('langue', 'fr'),
                livre_data.get('resume'), livre_data.get('couverture_url')
            ))
            return cursor.lastrowid

    def get_all(self, filters=None):
        """Récupère tous les livres avec filtres optionnels"""
        query = '''
            SELECT l.*, g.nom as genre_nom, g.couleur as genre_couleur
            FROM livres l
            LEFT JOIN genres g ON l.genre_id = g.id
        '''
        params = []

        # Ajouter des filtres
        conditions = []
        if filters:
            if filters.get('genre_id'):
                conditions.append("l.genre_id = ?")
                params.append(filters['genre_id'])

            if filters.get('disponible') is not None:
                conditions.append("l.disponible = ?")
                params.append(filters['disponible'])

            if filters.get('search'):
                conditions.append("(l.titre LIKE ? OR l.auteur LIKE ?)")
                search_term = f"%{filters['search']}%"
                params.extend([search_term, search_term])

        if conditions:
            query += " WHERE " + " AND ".join(conditions)

        query += " ORDER BY l.titre"

        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            return [dict(row) for row in cursor.fetchall()]

    def get_by_id(self, livre_id):
        """Récupère un livre par son ID"""
        with self.get_connection() as conn:
            cursor = conn.execute('''
                SELECT l.*, g.nom as genre_nom, g.couleur as genre_couleur
                FROM livres l
                LEFT JOIN genres g ON l.genre_id = g.id
                WHERE l.id = ?
            ''', (livre_id,))
            row = cursor.fetchone()
            return dict(row) if row else None

    def update(self, livre_id, livre_data):
        """Met à jour un livre"""
        set_clauses = []
        params = []

        for field in ['titre', 'auteur', 'isbn', 'genre_id', 'annee_publication',
                     'nombre_pages', 'langue', 'resume', 'couverture_url']:
            if field in livre_data:
                set_clauses.append(f"{field} = ?")
                params.append(livre_data[field])

        if not set_clauses:
            return False

        params.append(livre_id)
        query = f"UPDATE livres SET {', '.join(set_clauses)} WHERE id = ?"

        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            return cursor.rowcount > 0

    def delete(self, livre_id):
        """Supprime un livre (si pas d'emprunts actifs)"""
        with self.get_connection() as conn:
            # Vérifier qu'il n'y a pas d'emprunts actifs
            cursor = conn.execute(
                "SELECT COUNT(*) FROM emprunts WHERE livre_id = ? AND statut = 'actif'",
                (livre_id,)
            )
            if cursor.fetchone()[0] > 0:
                return False, "Impossible de supprimer : livre actuellement emprunté"

            # Supprimer le livre
            cursor = conn.execute("DELETE FROM livres WHERE id = ?", (livre_id,))
            return cursor.rowcount > 0, "Livre supprimé avec succès"

    def update_availability(self, livre_id, disponible):
        """Met à jour la disponibilité d'un livre"""
        with self.get_connection() as conn:
            cursor = conn.execute(
                "UPDATE livres SET disponible = ? WHERE id = ?",
                (disponible, livre_id)
            )
            return cursor.rowcount > 0

class Utilisateur(BaseModel):
    """Modèle pour gérer les utilisateurs"""

    def create(self, user_data):
        """Crée un nouvel utilisateur"""
        password_hash = generate_password_hash(user_data['mot_de_passe'])

        with self.get_connection() as conn:
            try:
                cursor = conn.execute('''
                    INSERT INTO utilisateurs (nom, email, mot_de_passe_hash, est_admin)
                    VALUES (?, ?, ?, ?)
                ''', (
                    user_data['nom'], user_data['email'],
                    password_hash, user_data.get('est_admin', False)
                ))
                return cursor.lastrowid
            except sqlite3.IntegrityError:
                return None  # Email déjà utilisé

    def authenticate(self, email, mot_de_passe):
        """Authentifie un utilisateur"""
        with self.get_connection() as conn:
            cursor = conn.execute(
                "SELECT * FROM utilisateurs WHERE email = ?",
                (email,)
            )
            user = cursor.fetchone()

            if user and check_password_hash(user['mot_de_passe_hash'], mot_de_passe):
                # Mettre à jour last_login
                conn.execute(
                    "UPDATE utilisateurs SET last_login = CURRENT_TIMESTAMP WHERE id = ?",
                    (user['id'],)
                )
                return dict(user)
            return None

    def get_by_id(self, user_id):
        """Récupère un utilisateur par ID"""
        with self.get_connection() as conn:
            cursor = conn.execute(
                "SELECT * FROM utilisateurs WHERE id = ?",
                (user_id,)
            )
            row = cursor.fetchone()
            return dict(row) if row else None

    def get_all(self):
        """Récupère tous les utilisateurs"""
        with self.get_connection() as conn:
            cursor = conn.execute(
                "SELECT id, nom, email, est_admin, created_at, last_login FROM utilisateurs"
            )
            return [dict(row) for row in cursor.fetchall()]

class Emprunt(BaseModel):
    """Modèle pour gérer les emprunts"""

    def create(self, emprunt_data):
        """Crée un nouvel emprunt"""
        # Date de retour prévue : 2 semaines par défaut
        date_retour_prevue = datetime.now() + timedelta(days=14)

        with self.get_connection() as conn:
            # Vérifier que le livre est disponible
            cursor = conn.execute(
                "SELECT disponible FROM livres WHERE id = ?",
                (emprunt_data['livre_id'],)
            )
            livre = cursor.fetchone()

            if not livre or not livre['disponible']:
                return None, "Livre non disponible"

            # Créer l'emprunt
            cursor = conn.execute('''
                INSERT INTO emprunts (livre_id, utilisateur_id, date_retour_prevue, commentaire)
                VALUES (?, ?, ?, ?)
            ''', (
                emprunt_data['livre_id'], emprunt_data['utilisateur_id'],
                date_retour_prevue, emprunt_data.get('commentaire')
            ))

            emprunt_id = cursor.lastrowid

            # Marquer le livre comme non disponible
            conn.execute(
                "UPDATE livres SET disponible = 0 WHERE id = ?",
                (emprunt_data['livre_id'],)
            )

            return emprunt_id, "Emprunt créé avec succès"

    def return_book(self, emprunt_id, commentaire=None):
        """Marque un livre comme retourné"""
        with self.get_connection() as conn:
            # Récupérer l'emprunt
            cursor = conn.execute(
                "SELECT livre_id FROM emprunts WHERE id = ? AND statut = 'actif'",
                (emprunt_id,)
            )
            emprunt = cursor.fetchone()

            if not emprunt:
                return False, "Emprunt non trouvé ou déjà retourné"

            # Marquer comme retourné
            conn.execute('''
                UPDATE emprunts
                SET date_retour_effective = CURRENT_TIMESTAMP,
                    statut = 'retourne',
                    commentaire = COALESCE(?, commentaire)
                WHERE id = ?
            ''', (commentaire, emprunt_id))

            # Rendre le livre disponible
            conn.execute(
                "UPDATE livres SET disponible = 1 WHERE id = ?",
                (emprunt['livre_id'],)
            )

            return True, "Livre retourné avec succès"

    def get_active_loans(self, user_id=None):
        """Récupère les emprunts actifs"""
        query = '''
            SELECT e.*, l.titre, l.auteur, l.couverture_url, u.nom as nom_utilisateur
            FROM emprunts e
            JOIN livres l ON e.livre_id = l.id
            JOIN utilisateurs u ON e.utilisateur_id = u.id
            WHERE e.statut = 'actif'
        '''
        params = []

        if user_id:
            query += " AND e.utilisateur_id = ?"
            params.append(user_id)

        query += " ORDER BY e.date_retour_prevue"

        with self.get_connection() as conn:
            cursor = conn.execute(query, params)
            return [dict(row) for row in cursor.fetchall()]

    def get_overdue_loans(self):
        """Récupère les emprunts en retard"""
        with self.get_connection() as conn:
            cursor = conn.execute('''
                SELECT e.*, l.titre, l.auteur, u.nom as nom_utilisateur, u.email
                FROM emprunts e
                JOIN livres l ON e.livre_id = l.id
                JOIN utilisateurs u ON e.utilisateur_id = u.id
                WHERE e.statut = 'actif' AND e.date_retour_prevue < DATE('now')
                ORDER BY e.date_retour_prevue
            ''')
            return [dict(row) for row in cursor.fetchall()]

    def get_user_history(self, user_id):
        """Récupère l'historique d'emprunts d'un utilisateur"""
        with self.get_connection() as conn:
            cursor = conn.execute('''
                SELECT e.*, l.titre, l.auteur, l.couverture_url
                FROM emprunts e
                JOIN livres l ON e.livre_id = l.id
                WHERE e.utilisateur_id = ?
                ORDER BY e.date_emprunt DESC
            ''', (user_id,))
            return [dict(row) for row in cursor.fetchall()]

class Genre(BaseModel):
    """Modèle pour gérer les genres"""

    def get_all(self):
        """Récupère tous les genres"""
        with self.get_connection() as conn:
            cursor = conn.execute("SELECT * FROM genres ORDER BY nom")
            return [dict(row) for row in cursor.fetchall()]

    def get_with_counts(self):
        """Récupère les genres avec le nombre de livres"""
        with self.get_connection() as conn:
            cursor = conn.execute('''
                SELECT g.*, COUNT(l.id) as nombre_livres
                FROM genres g
                LEFT JOIN livres l ON g.id = l.genre_id
                GROUP BY g.id
                ORDER BY g.nom
            ''')
            return [dict(row) for row in cursor.fetchall()]

    def create(self, genre_data):
        """Crée un nouveau genre"""
        with self.get_connection() as conn:
            try:
                cursor = conn.execute('''
                    INSERT INTO genres (nom, description, couleur)
                    VALUES (?, ?, ?)
                ''', (
                    genre_data['nom'],
                    genre_data.get('description', ''),
                    genre_data.get('couleur', '#3498db')
                ))
                return cursor.lastrowid
            except sqlite3.IntegrityError:
                return None  # Genre déjà existant
```

## Étape 3 : API REST avec Flask

### Configuration Flask

```python
# app/__init__.py
from flask import Flask
from werkzeug.utils import secure_filename
import os

def create_app():
    app = Flask(__name__)

    # Configuration
    app.config['SECRET_KEY'] = 'votre-clé-secrète-ici'
    app.config['UPLOAD_FOLDER'] = 'static/uploads'
    app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024  # 16MB max

    # S'assurer que le dossier d'upload existe
    os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)

    # Enregistrer les blueprints
    from app.routes import bp as main_bp
    app.register_blueprint(main_bp)

    return app
```

### Routes principales

```python
# app/routes.py
from flask import Blueprint, request, jsonify, render_template, session, redirect, url_for, flash
from werkzeug.utils import secure_filename
import os
from app.models import Livre, Utilisateur, Emprunt, Genre
from app.database import DatabaseManager

bp = Blueprint('main', __name__)

# Initialiser la base au démarrage
db_manager = DatabaseManager()

# Middleware pour vérifier l'authentification
def login_required(f):
    from functools import wraps
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            return redirect(url_for('main.login'))
        return f(*args, **kwargs)
    return decorated_function

def admin_required(f):
    from functools import wraps
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session or not session.get('is_admin', False):
            flash('Accès administrateur requis', 'error')
            return redirect(url_for('main.index'))
        return f(*args, **kwargs)
    return decorated_function

# Routes de pages
@bp.route('/')
def index():
    """Page d'accueil"""
    livre_model = Livre()

    # Statistiques rapides
    stats = {
        'total_livres': len(livre_model.get_all()),
        'livres_disponibles': len(livre_model.get_all({'disponible': True})),
        'emprunts_actifs': len(Emprunt().get_active_loans())
    }

    # Derniers livres ajoutés
    derniers_livres = livre_model.get_all()[:6]  # 6 derniers

    return render_template('index.html', stats=stats, derniers_livres=derniers_livres)

@bp.route('/books')
@login_required
def books():
    """Page de gestion des livres"""
    return render_template('books.html')

@bp.route('/stats')
@login_required
def stats():
    """Page des statistiques"""
    return render_template('stats.html')

# Routes d'authentification
@bp.route('/login', methods=['GET', 'POST'])
def login():
    """Connexion utilisateur"""
    if request.method == 'POST':
        email = request.form['email']
        password = request.form['password']

        user_model = Utilisateur()
        user = user_model.authenticate(email, password)

        if user:
            session['user_id'] = user['id']
            session['user_name'] = user['nom']
            session['is_admin'] = user['est_admin']
            flash('Connexion réussie', 'success')
            return redirect(url_for('main.index'))
        else:
            flash('Email ou mot de passe incorrect', 'error')

    return render_template('login.html')

@bp.route('/logout')
def logout():
    """Déconnexion"""
    session.clear()
    flash('Vous avez été déconnecté', 'info')
    return redirect(url_for('main.login'))

@bp.route('/register', methods=['GET', 'POST'])
def register():
    """Inscription d'un nouvel utilisateur"""
    if request.method == 'POST':
        nom = request.form['nom']
        email = request.form['email']
        password = request.form['password']

        user_model = Utilisateur()
        user_id = user_model.create({
            'nom': nom,
            'email': email,
            'mot_de_passe': password
        })

        if user_id:
            flash('Inscription réussie, vous pouvez vous connecter', 'success')
            return redirect(url_for('main.login'))
        else:
            flash('Email déjà utilisé', 'error')

    return render_template('register.html')

# API Routes
@bp.route('/api/books')
@login_required
def api_books():
    """API : Liste des livres avec filtres"""
    livre_model = Livre()

    # Récupérer les filtres de la query string
    filters = {}
    if request.args.get('genre_id'):
        filters['genre_id'] = request.args.get('genre_id')
    if request.args.get('search'):
        filters['search'] = request.args.get('search')
    if request.args.get('disponible'):
        filters['disponible'] = request.args.get('disponible') == 'true'

    livres = livre_model.get_all(filters)
    return jsonify(livres)

@bp.route('/api/books', methods=['POST'])
@admin_required
def api_create_book():
    """API : Créer un nouveau livre"""
    data = request.get_json()

    # Validation de base
    required_fields = ['titre', 'auteur']
    for field in required_fields:
        if not data.get(field):
            return jsonify({'error': f'Champ requis: {field}'}), 400

    livre_model = Livre()
    livre_id = livre_model.create(data)

    if livre_id:
        return jsonify({'id': livre_id, 'message': 'Livre créé avec succès'}), 201
    else:
        return jsonify({'error': 'Erreur lors de la création'}), 500

@bp.route('/api/books/<int:book_id>')
@login_required
def api_get_book(book_id):
    """API : Récupérer un livre par ID"""
    livre_model = Livre()
    livre = livre_model.get_by_id(book_id)

    if livre:
        return jsonify(livre)
    else:
        return jsonify({'error': 'Livre non trouvé'}), 404

@bp.route('/api/books/<int:book_id>', methods=['PUT'])
@admin_required
def api_update_book(book_id):
    """API : Mettre à jour un livre"""
    data = request.get_json()

    livre_model = Livre()
    success = livre_model.update(book_id, data)

    if success:
        return jsonify({'message': 'Livre mis à jour avec succès'})
    else:
        return jsonify({'error': 'Livre non trouvé ou erreur de mise à jour'}), 404

@bp.route('/api/books/<int:book_id>', methods=['DELETE'])
@admin_required
def api_delete_book(book_id):
    """API : Supprimer un livre"""
    livre_model = Livre()
    success, message = livre_model.delete(book_id)

    if success:
        return jsonify({'message': message})
    else:
        return jsonify({'error': message}), 400

@bp.route('/api/loans')
@login_required
def api_loans():
    """API : Liste des emprunts actifs"""
    emprunt_model = Emprunt()

    # Si utilisateur normal, seulement ses emprunts
    if not session.get('is_admin', False):
        emprunts = emprunt_model.get_active_loans(session['user_id'])
    else:
        emprunts = emprunt_model.get_active_loans()

    return jsonify(emprunts)

@bp.route('/api/loans', methods=['POST'])
@login_required
def api_create_loan():
    """API : Créer un nouvel emprunt"""
    data = request.get_json()

    # Validation
    if not data.get('livre_id'):
        return jsonify({'error': 'ID du livre requis'}), 400

    # Ajouter l'ID utilisateur de la session
    data['utilisateur_id'] = session['user_id']

    emprunt_model = Emprunt()
    emprunt_id, message = emprunt_model.create(data)

    if emprunt_id:
        return jsonify({'id': emprunt_id, 'message': message}), 201
    else:
        return jsonify({'error': message}), 400

@bp.route('/api/loans/<int:loan_id>/return', methods=['POST'])
@login_required
def api_return_book(loan_id):
    """API : Retourner un livre"""
    data = request.get_json() or {}
    commentaire = data.get('commentaire')

    emprunt_model = Emprunt()
    success, message = emprunt_model.return_book(loan_id, commentaire)

    if success:
        return jsonify({'message': message})
    else:
        return jsonify({'error': message}), 400

@bp.route('/api/loans/overdue')
@admin_required
def api_overdue_loans():
    """API : Emprunts en retard (admin seulement)"""
    emprunt_model = Emprunt()
    emprunts = emprunt_model.get_overdue_loans()
    return jsonify(emprunts)

@bp.route('/api/genres')
def api_genres():
    """API : Liste des genres"""
    genre_model = Genre()
    genres = genre_model.get_with_counts()
    return jsonify(genres)

@bp.route('/api/stats')
@login_required
def api_stats():
    """API : Statistiques de la bibliothèque"""
    livre_model = Livre()
    emprunt_model = Emprunt()
    genre_model = Genre()

    # Statistiques générales
    tous_livres = livre_model.get_all()
    emprunts_actifs = emprunt_model.get_active_loans()
    emprunts_retard = emprunt_model.get_overdue_loans()

    stats = {
        'livres': {
            'total': len(tous_livres),
            'disponibles': len([l for l in tous_livres if l['disponible']]),
            'empruntes': len([l for l in tous_livres if not l['disponible']])
        },
        'emprunts': {
            'actifs': len(emprunts_actifs),
            'en_retard': len(emprunts_retard)
        },
        'genres': genre_model.get_with_counts()
    }

    # Si admin, ajouter plus de détails
    if session.get('is_admin', False):
        stats['emprunts']['details_retard'] = emprunts_retard

    return jsonify(stats)

@bp.route('/api/upload', methods=['POST'])
@admin_required
def api_upload_cover():
    """API : Upload d'image de couverture"""
    if 'file' not in request.files:
        return jsonify({'error': 'Aucun fichier'}), 400

    file = request.files['file']
    if file.filename == '':
        return jsonify({'error': 'Aucun fichier sélectionné'}), 400

    # Vérifier l'extension
    allowed_extensions = {'png', 'jpg', 'jpeg', 'gif', 'webp'}
    if '.' not in file.filename or \
       file.filename.rsplit('.', 1)[1].lower() not in allowed_extensions:
        return jsonify({'error': 'Format de fichier non autorisé'}), 400

    # Sauvegarder le fichier
    filename = secure_filename(file.filename)
    # Ajouter timestamp pour éviter les conflits
    import time
    filename = f"{int(time.time())}_{filename}"

    file_path = os.path.join(current_app.config['UPLOAD_FOLDER'], filename)
    file.save(file_path)

    # Retourner l'URL relative
    return jsonify({'url': f'/static/uploads/{filename}'})

@bp.route('/api/search')
@login_required
def api_search():
    """API : Recherche avancée"""
    query = request.args.get('q', '').strip()
    if not query:
        return jsonify([])

    livre_model = Livre()
    livres = livre_model.get_all({'search': query})

    # Limiter les résultats pour l'autocomplétion
    limite = int(request.args.get('limit', 10))
    return jsonify(livres[:limite])
```

## Frontend - Templates HTML

### Template de base

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Ma Bibliothèque{% endblock %}</title>

    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <!-- Font Awesome pour les icônes -->
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <!-- CSS personnalisé -->
    <link href="{{ url_for('static', filename='css/style.css') }}" rel="stylesheet">
</head>
<body>
    <!-- Navigation -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{{ url_for('main.index') }}">
                <i class="fas fa-book"></i> Ma Bibliothèque
            </a>

            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>

            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.index') }}">
                            <i class="fas fa-home"></i> Accueil
                        </a>
                    </li>
                    {% if session.user_id %}
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.books') }}">
                            <i class="fas fa-books"></i> Livres
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.stats') }}">
                            <i class="fas fa-chart-bar"></i> Statistiques
                        </a>
                    </li>
                    {% endif %}
                </ul>

                <ul class="navbar-nav">
                    {% if session.user_id %}
                    <li class="nav-item dropdown">
                        <a class="nav-link dropdown-toggle" href="#" id="navbarDropdown" role="button" data-bs-toggle="dropdown">
                            <i class="fas fa-user"></i> {{ session.user_name }}
                        </a>
                        <ul class="dropdown-menu">
                            <li><a class="dropdown-item" href="#" onclick="showUserLoans()">Mes emprunts</a></li>
                            {% if session.is_admin %}
                            <li><hr class="dropdown-divider"></li>
                            <li><a class="dropdown-item" href="#" onclick="showAdminPanel()">Administration</a></li>
                            {% endif %}
                            <li><hr class="dropdown-divider"></li>
                            <li><a class="dropdown-item" href="{{ url_for('main.logout') }}">Déconnexion</a></li>
                        </ul>
                    </li>
                    {% else %}
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.login') }}">Connexion</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.register') }}">Inscription</a>
                    </li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>

    <!-- Messages flash -->
    {% with messages = get_flashed_messages(with_categories=true) %}
        {% if messages %}
        <div class="container mt-3">
            {% for category, message in messages %}
            <div class="alert alert-{{ 'danger' if category == 'error' else category }} alert-dismissible fade show" role="alert">
                {{ message }}
                <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
            </div>
            {% endfor %}
        </div>
        {% endif %}
    {% endwith %}

    <!-- Contenu principal -->
    <main class="container-fluid py-4">
        {% block content %}{% endblock %}
    </main>

    <!-- Footer -->
    <footer class="bg-light text-center py-3 mt-5">
        <div class="container">
            <small class="text-muted">
                © 2024 Ma Bibliothèque - Propulsé par SQLite et Flask
            </small>
        </div>
    </footer>

    <!-- Bootstrap JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
    <!-- JavaScript personnalisé -->
    <script src="{{ url_for('static', filename='js/app.js') }}"></script>

    {% block scripts %}{% endblock %}
</body>
</html>
```

### Page d'accueil

```html
<!-- templates/index.html -->
{% extends "base.html" %}

{% block title %}Accueil - Ma Bibliothèque{% endblock %}

{% block content %}
<div class="container">
    <!-- Hero Section -->
    <div class="row mb-5">
        <div class="col-12">
            <div class="jumbotron bg-gradient text-white rounded-3 p-5" style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);">
                <h1 class="display-4">Bienvenue dans Ma Bibliothèque</h1>
                <p class="lead">Gérez votre collection de livres, suivez vos emprunts et découvrez vos statistiques de lecture.</p>
                {% if not session.user_id %}
                <a class="btn btn-light btn-lg" href="{{ url_for('main.register') }}" role="button">
                    <i class="fas fa-user-plus"></i> Commencer maintenant
                </a>
                {% endif %}
            </div>
        </div>
    </div>

    <!-- Statistiques rapides -->
    <div class="row mb-5">
        <div class="col-md-4">
            <div class="card text-center h-100">
                <div class="card-body">
                    <i class="fas fa-book fa-3x text-primary mb-3"></i>
                    <h3 class="card-title">{{ stats.total_livres }}</h3>
                    <p class="card-text">Livres dans la collection</p>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card text-center h-100">
                <div class="card-body">
                    <i class="fas fa-check-circle fa-3x text-success mb-3"></i>
                    <h3 class="card-title">{{ stats.livres_disponibles }}</h3>
                    <p class="card-text">Livres disponibles</p>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card text-center h-100">
                <div class="card-body">
                    <i class="fas fa-hand-holding fa-3x text-warning mb-3"></i>
                    <h3 class="card-title">{{ stats.emprunts_actifs }}</h3>
                    <p class="card-text">Emprunts en cours</p>
                </div>
            </div>
        </div>
    </div>

    <!-- Derniers livres ajoutés -->
    {% if derniers_livres %}
    <div class="row">
        <div class="col-12">
            <h2 class="mb-4">
                <i class="fas fa-plus-circle text-primary"></i>
                Derniers livres ajoutés
            </h2>
        </div>
    </div>

    <div class="row">
        {% for livre in derniers_livres %}
        <div class="col-lg-2 col-md-3 col-sm-4 col-6 mb-4">
            <div class="card h-100">
                {% if livre.couverture_url %}
                <img src="{{ livre.couverture_url }}" class="card-img-top" alt="{{ livre.titre }}" style="height: 200px; object-fit: cover;">
                {% else %}
                <div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 200px;">
                    <i class="fas fa-book fa-3x text-muted"></i>
                </div>
                {% endif %}

                <div class="card-body p-2">
                    <h6 class="card-title mb-1" title="{{ livre.titre }}">
                        {{ livre.titre[:30] }}{% if livre.titre|length > 30 %}...{% endif %}
                    </h6>
                    <small class="text-muted">{{ livre.auteur }}</small>

                    {% if livre.genre_nom %}
                    <div class="mt-1">
                        <span class="badge" style="background-color: {{ livre.genre_couleur }};">
                            {{ livre.genre_nom }}
                        </span>
                    </div>
                    {% endif %}

                    <div class="mt-2">
                        {% if livre.disponible %}
                        <span class="badge bg-success">Disponible</span>
                        {% else %}
                        <span class="badge bg-warning">Emprunté</span>
                        {% endif %}
                    </div>
                </div>
            </div>
        </div>
        {% endfor %}
    </div>
    {% endif %}

    <!-- Appel à l'action -->
    {% if session.user_id %}
    <div class="row mt-5">
        <div class="col-12 text-center">
            <div class="card">
                <div class="card-body">
                    <h3>Que souhaitez-vous faire ?</h3>
                    <div class="mt-3">
                        <a href="{{ url_for('main.books') }}" class="btn btn-primary btn-lg me-3">
                            <i class="fas fa-search"></i> Parcourir les livres
                        </a>
                        {% if session.is_admin %}
                        <button class="btn btn-success btn-lg" onclick="showAddBookModal()">
                            <i class="fas fa-plus"></i> Ajouter un livre
                        </button>
                        {% endif %}
                    </div>
                </div>
            </div>
        </div>
    </div>
    {% endif %}
</div>
{% endblock %}
```

### Page de gestion des livres

```html
<!-- templates/books.html -->
{% extends "base.html" %}

{% block title %}Livres - Ma Bibliothèque{% endblock %}

{% block content %}
<div class="container-fluid">
    <div class="row">
        <!-- Sidebar des filtres -->
        <div class="col-md-3">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-filter"></i> Filtres</h5>
                </div>
                <div class="card-body">
                    <!-- Recherche -->
                    <div class="mb-3">
                        <label class="form-label">Recherche</label>
                        <input type="text" class="form-control" id="searchInput"
                               placeholder="Titre ou auteur...">
                    </div>

                    <!-- Filtrer par genre -->
                    <div class="mb-3">
                        <label class="form-label">Genre</label>
                        <select class="form-select" id="genreFilter">
                            <option value="">Tous les genres</option>
                        </select>
                    </div>

                    <!-- Filtrer par disponibilité -->
                    <div class="mb-3">
                        <label class="form-label">Disponibilité</label>
                        <select class="form-select" id="availabilityFilter">
                            <option value="">Tous</option>
                            <option value="true">Disponibles</option>
                            <option value="false">Empruntés</option>
                        </select>
                    </div>

                    <!-- Bouton reset -->
                    <button class="btn btn-outline-secondary w-100" onclick="resetFilters()">
                        <i class="fas fa-undo"></i> Réinitialiser
                    </button>
                </div>
            </div>
        </div>

        <!-- Zone principale -->
        <div class="col-md-9">
            <!-- Header avec actions -->
            <div class="d-flex justify-content-between align-items-center mb-4">
                <h1><i class="fas fa-books"></i> Bibliothèque</h1>

                <div>
                    <!-- Bouton vue -->
                    <div class="btn-group me-2" role="group">
                        <button type="button" class="btn btn-outline-primary active" onclick="setView('grid')" id="gridViewBtn">
                            <i class="fas fa-th"></i>
                        </button>
                        <button type="button" class="btn btn-outline-primary" onclick="setView('list')" id="listViewBtn">
                            <i class="fas fa-list"></i>
                        </button>
                    </div>

                    {% if session.is_admin %}
                    <button class="btn btn-success" onclick="showAddBookModal()">
                        <i class="fas fa-plus"></i> Ajouter un livre
                    </button>
                    {% endif %}
                </div>
            </div>

            <!-- Zone de contenu des livres -->
            <div id="booksContainer">
                <!-- Les livres seront chargés ici via JavaScript -->
                <div class="text-center py-5">
                    <div class="spinner-border text-primary" role="status">
                        <span class="visually-hidden">Chargement...</span>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

<!-- Modal d'ajout/édition de livre -->
<div class="modal fade" id="bookModal" tabindex="-1">
    <div class="modal-dialog modal-lg">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title" id="bookModalTitle">Ajouter un livre</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <form id="bookForm">
                    <input type="hidden" id="bookId">

                    <div class="row">
                        <div class="col-md-8">
                            <div class="mb-3">
                                <label class="form-label">Titre *</label>
                                <input type="text" class="form-control" id="bookTitle" required>
                            </div>

                            <div class="mb-3">
                                <label class="form-label">Auteur *</label>
                                <input type="text" class="form-control" id="bookAuthor" required>
                            </div>

                            <div class="row">
                                <div class="col-md-6">
                                    <div class="mb-3">
                                        <label class="form-label">ISBN</label>
                                        <input type="text" class="form-control" id="bookIsbn">
                                    </div>
                                </div>
                                <div class="col-md-6">
                                    <div class="mb-3">
                                        <label class="form-label">Genre</label>
                                        <select class="form-select" id="bookGenre">
                                            <option value="">Sélectionner...</option>
                                        </select>
                                    </div>
                                </div>
                            </div>

                            <div class="row">
                                <div class="col-md-6">
                                    <div class="mb-3">
                                        <label class="form-label">Année de publication</label>
                                        <input type="number" class="form-control" id="bookYear" min="1000" max="2030">
                                    </div>
                                </div>
                                <div class="col-md-6">
                                    <div class="mb-3">
                                        <label class="form-label">Nombre de pages</label>
                                        <input type="number" class="form-control" id="bookPages" min="1">
                                    </div>
                                </div>
                            </div>
                        </div>

                        <div class="col-md-4">
                            <div class="mb-3">
                                <label class="form-label">Image de couverture</label>
                                <div class="text-center">
                                    <div id="coverPreview" class="mb-2" style="height: 200px; border: 2px dashed #ddd; display: flex; align-items: center; justify-content: center;">
                                        <i class="fas fa-image fa-3x text-muted"></i>
                                    </div>
                                    <input type="file" class="form-control" id="bookCover" accept="image/*" onchange="previewCover(this)">
                                </div>
                            </div>
                        </div>
                    </div>

                    <div class="mb-3">
                        <label class="form-label">Résumé</label>
                        <textarea class="form-control" id="bookSummary" rows="3"></textarea>
                    </div>
                </form>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Annuler</button>
                <button type="button" class="btn btn-primary" onclick="saveBook()">Enregistrer</button>
            </div>
        </div>
    </div>
</div>

<!-- Modal de détails du livre -->
<div class="modal fade" id="bookDetailsModal" tabindex="-1">
    <div class="modal-dialog modal-lg">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title" id="bookDetailsTitle"></h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body" id="bookDetailsBody">
                <!-- Détails du livre chargés via JavaScript -->
            </div>
            <div class="modal-footer" id="bookDetailsFooter">
                <!-- Actions chargées via JavaScript -->
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% block scripts %}
<script>
// Variables globales
let currentBooks = [];
let currentView = 'grid';
let currentFilters = {};

// Initialisation de la page
document.addEventListener('DOMContentLoaded', function() {
    loadGenres();
    loadBooks();
    setupEventListeners();
});

// Configuration des event listeners
function setupEventListeners() {
    // Recherche en temps réel
    document.getElementById('searchInput').addEventListener('input', function() {
        debounce(filterBooks, 300)();
    });

    // Filtres
    document.getElementById('genreFilter').addEventListener('change', filterBooks);
    document.getElementById('availabilityFilter').addEventListener('change', filterBooks);
}

// Fonction debounce pour la recherche
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

// Charger les genres
function loadGenres() {
    fetch('/api/genres')
        .then(response => response.json())
        .then(genres => {
            const selects = ['genreFilter', 'bookGenre'];
            selects.forEach(selectId => {
                const select = document.getElementById(selectId);
                // Garder la première option
                const firstOption = select.querySelector('option');
                select.innerHTML = '';
                select.appendChild(firstOption);

                genres.forEach(genre => {
                    const option = document.createElement('option');
                    option.value = genre.id;
                    option.textContent = `${genre.nom} (${genre.nombre_livres})`;
                    select.appendChild(option);
                });
            });
        })
        .catch(error => console.error('Erreur chargement genres:', error));
}

// Charger les livres
function loadBooks() {
    const params = new URLSearchParams(currentFilters);

    fetch(`/api/books?${params}`)
        .then(response => response.json())
        .then(books => {
            currentBooks = books;
            displayBooks(books);
        })
        .catch(error => {
            console.error('Erreur chargement livres:', error);
            document.getElementById('booksContainer').innerHTML =
                '<div class="alert alert-danger">Erreur lors du chargement des livres</div>';
        });
}

// Afficher les livres
function displayBooks(books) {
    const container = document.getElementById('booksContainer');

    if (books.length === 0) {
        container.innerHTML = `
            <div class="text-center py-5">
                <i class="fas fa-search fa-3x text-muted mb-3"></i>
                <h3>Aucun livre trouvé</h3>
                <p class="text-muted">Essayez de modifier vos critères de recherche</p>
            </div>
        `;
        return;
    }

    if (currentView === 'grid') {
        displayBooksGrid(books, container);
    } else {
        displayBooksList(books, container);
    }
}

// Affichage en grille
function displayBooksGrid(books, container) {
    container.innerHTML = `
        <div class="row">
            ${books.map(book => `
                <div class="col-xl-2 col-lg-3 col-md-4 col-sm-6 mb-4">
                    <div class="card h-100">
                        <div class="position-relative">
                            ${book.couverture_url ?
                                `<img src="${book.couverture_url}" class="card-img-top" style="height: 200px; object-fit: cover;" alt="${book.titre}">` :
                                `<div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 200px;">
                                    <i class="fas fa-book fa-3x text-muted"></i>
                                </div>`
                            }

                            <!-- Badge de disponibilité -->
                            <div class="position-absolute top-0 end-0 m-2">
                                ${book.disponible ?
                                    '<span class="badge bg-success">Disponible</span>' :
                                    '<span class="badge bg-warning">Emprunté</span>'
                                }
                            </div>

                            <!-- Menu d'actions (admin) -->
                            ${isAdmin ? `
                                <div class="position-absolute top-0 start-0 m-2">
                                    <div class="dropdown">
                                        <button class="btn btn-sm btn-outline-light" type="button" data-bs-toggle="dropdown">
                                            <i class="fas fa-ellipsis-v"></i>
                                        </button>
                                        <ul class="dropdown-menu">
                                            <li><a class="dropdown-item" href="#" onclick="editBook(${book.id})">
                                                <i class="fas fa-edit"></i> Modifier
                                            </a></li>
                                            <li><a class="dropdown-item text-danger" href="#" onclick="deleteBook(${book.id})">
                                                <i class="fas fa-trash"></i> Supprimer
                                            </a></li>
                                        </ul>
                                    </div>
                                </div>
                            ` : ''}
                        </div>

                        <div class="card-body d-flex flex-column">
                            <h6 class="card-title mb-1" title="${book.titre}">
                                ${book.titre.length > 25 ? book.titre.substring(0, 25) + '...' : book.titre}
                            </h6>
                            <small class="text-muted mb-2">${book.auteur}</small>

                            ${book.genre_nom ? `
                                <div class="mb-2">
                                    <span class="badge" style="background-color: ${book.genre_couleur};">
                                        ${book.genre_nom}
                                    </span>
                                </div>
                            ` : ''}

                            ${book.annee_publication ? `
                                <small class="text-muted mb-2">${book.annee_publication}</small>
                            ` : ''}

                            <div class="mt-auto">
                                <button class="btn btn-primary btn-sm w-100" onclick="showBookDetails(${book.id})">
                                    <i class="fas fa-eye"></i> Détails
                                </button>

                                ${book.disponible && !isAdmin ? `
                                    <button class="btn btn-success btn-sm w-100 mt-1" onclick="borrowBook(${book.id})">
                                        <i class="fas fa-hand-holding"></i> Emprunter
                                    </button>
                                ` : ''}
                            </div>
                        </div>
                    </div>
                </div>
            `).join('')}
        </div>
    `;
}

// Affichage en liste
function displayBooksList(books, container) {
    container.innerHTML = `
        <div class="table-responsive">
            <table class="table table-hover">
                <thead class="table-light">
                    <tr>
                        <th>Couverture</th>
                        <th>Titre</th>
                        <th>Auteur</th>
                        <th>Genre</th>
                        <th>Année</th>
                        <th>Pages</th>
                        <th>Statut</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    ${books.map(book => `
                        <tr>
                            <td>
                                ${book.couverture_url ?
                                    `<img src="${book.couverture_url}" style="width: 40px; height: 60px; object-fit: cover;" alt="${book.titre}">` :
                                    `<div class="bg-light d-flex align-items-center justify-content-center" style="width: 40px; height: 60px;">
                                        <i class="fas fa-book text-muted"></i>
                                    </div>`
                                }
                            </td>
                            <td>
                                <strong>${book.titre}</strong>
                                ${book.isbn ? `<br><small class="text-muted">ISBN: ${book.isbn}</small>` : ''}
                            </td>
                            <td>${book.auteur}</td>
                            <td>
                                ${book.genre_nom ? `
                                    <span class="badge" style="background-color: ${book.genre_couleur};">
                                        ${book.genre_nom}
                                    </span>
                                ` : '-'}
                            </td>
                            <td>${book.annee_publication || '-'}</td>
                            <td>${book.nombre_pages || '-'}</td>
                            <td>
                                ${book.disponible ?
                                    '<span class="badge bg-success">Disponible</span>' :
                                    '<span class="badge bg-warning">Emprunté</span>'
                                }
                            </td>
                            <td>
                                <div class="btn-group btn-group-sm">
                                    <button class="btn btn-outline-primary" onclick="showBookDetails(${book.id})" title="Détails">
                                        <i class="fas fa-eye"></i>
                                    </button>

                                    ${book.disponible && !isAdmin ? `
                                        <button class="btn btn-outline-success" onclick="borrowBook(${book.id})" title="Emprunter">
                                            <i class="fas fa-hand-holding"></i>
                                        </button>
                                    ` : ''}

                                    ${isAdmin ? `
                                        <button class="btn btn-outline-secondary" onclick="editBook(${book.id})" title="Modifier">
                                            <i class="fas fa-edit"></i>
                                        </button>
                                        <button class="btn btn-outline-danger" onclick="deleteBook(${book.id})" title="Supprimer">
                                            <i class="fas fa-trash"></i>
                                        </button>
                                    ` : ''}
                                </div>
                            </td>
                        </tr>
                    `).join('')}
                </tbody>
            </table>
        </div>
    `;
}

// Changer de vue
function setView(view) {
    currentView = view;

    // Mettre à jour les boutons
    document.getElementById('gridViewBtn').classList.toggle('active', view === 'grid');
    document.getElementById('listViewBtn').classList.toggle('active', view === 'list');

    // Réafficher les livres
    displayBooks(currentBooks);
}

// Filtrer les livres
function filterBooks() {
    currentFilters = {};

    const search = document.getElementById('searchInput').value;
    const genre = document.getElementById('genreFilter').value;
    const availability = document.getElementById('availabilityFilter').value;

    if (search) currentFilters.search = search;
    if (genre) currentFilters.genre_id = genre;
    if (availability) currentFilters.disponible = availability;

    loadBooks();
}

// Réinitialiser les filtres
function resetFilters() {
    document.getElementById('searchInput').value = '';
    document.getElementById('genreFilter').value = '';
    document.getElementById('availabilityFilter').value = '';

    currentFilters = {};
    loadBooks();
}

// Afficher les détails d'un livre
function showBookDetails(bookId) {
    fetch(`/api/books/${bookId}`)
        .then(response => response.json())
        .then(book => {
            document.getElementById('bookDetailsTitle').textContent = book.titre;

            document.getElementById('bookDetailsBody').innerHTML = `
                <div class="row">
                    <div class="col-md-4">
                        ${book.couverture_url ?
                            `<img src="${book.couverture_url}" class="img-fluid rounded" alt="${book.titre}">` :
                            `<div class="bg-light d-flex align-items-center justify-content-center rounded" style="height: 300px;">
                                <i class="fas fa-book fa-4x text-muted"></i>
                            </div>`
                        }
                    </div>
                    <div class="col-md-8">
                        <dl class="row">
                            <dt class="col-sm-3">Titre:</dt>
                            <dd class="col-sm-9">${book.titre}</dd>

                            <dt class="col-sm-3">Auteur:</dt>
                            <dd class="col-sm-9">${book.auteur}</dd>

                            ${book.isbn ? `
                                <dt class="col-sm-3">ISBN:</dt>
                                <dd class="col-sm-9">${book.isbn}</dd>
                            ` : ''}

                            ${book.genre_nom ? `
                                <dt class="col-sm-3">Genre:</dt>
                                <dd class="col-sm-9">
                                    <span class="badge" style="background-color: ${book.genre_couleur};">
                                        ${book.genre_nom}
                                    </span>
                                </dd>
                            ` : ''}

                            ${book.annee_publication ? `
                                <dt class="col-sm-3">Année:</dt>
                                <dd class="col-sm-9">${book.annee_publication}</dd>
                            ` : ''}

                            ${book.nombre_pages ? `
                                <dt class="col-sm-3">Pages:</dt>
                                <dd class="col-sm-9">${book.nombre_pages}</dd>
                            ` : ''}

                            <dt class="col-sm-3">Langue:</dt>
                            <dd class="col-sm-9">${book.langue || 'Non spécifiée'}</dd>

                            <dt class="col-sm-3">Statut:</dt>
                            <dd class="col-sm-9">
                                ${book.disponible ?
                                    '<span class="badge bg-success">Disponible</span>' :
                                    '<span class="badge bg-warning">Emprunté</span>'
                                }
                            </dd>

                            <dt class="col-sm-3">Ajouté le:</dt>
                            <dd class="col-sm-9">${new Date(book.date_ajout).toLocaleDateString('fr-FR')}</dd>
                        </dl>

                        ${book.resume ? `
                            <h6>Résumé:</h6>
                            <p class="text-muted">${book.resume}</p>
                        ` : ''}
                    </div>
                </div>
            `;

            // Actions dans le footer
            let footerActions = `
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Fermer</button>
            `;

            if (book.disponible && !isAdmin) {
                footerActions = `
                    <button type="button" class="btn btn-success" onclick="borrowBook(${book.id}); closeModal('bookDetailsModal');">
                        <i class="fas fa-hand-holding"></i> Emprunter
                    </button>
                ` + footerActions;
            }

            if (isAdmin) {
                footerActions = `
                    <button type="button" class="btn btn-primary" onclick="editBook(${book.id}); closeModal('bookDetailsModal');">
                        <i class="fas fa-edit"></i> Modifier
                    </button>
                    <button type="button" class="btn btn-danger" onclick="deleteBook(${book.id}); closeModal('bookDetailsModal');">
                        <i class="fas fa-trash"></i> Supprimer
                    </button>
                ` + footerActions;
            }

            document.getElementById('bookDetailsFooter').innerHTML = footerActions;

            new bootstrap.Modal(document.getElementById('bookDetailsModal')).show();
        })
        .catch(error => {
            console.error('Erreur:', error);
            showAlert('Erreur lors du chargement des détails', 'danger');
        });
}

// Emprunter un livre
function borrowBook(bookId) {
    if (!confirm('Confirmer l\'emprunt de ce livre ?')) return;

    fetch('/api/loans', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            livre_id: bookId
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.error) {
            showAlert(data.error, 'danger');
        } else {
            showAlert(data.message, 'success');
            loadBooks(); // Recharger pour mettre à jour le statut
        }
    })
    .catch(error => {
        console.error('Erreur:', error);
        showAlert('Erreur lors de l\'emprunt', 'danger');
    });
}

// Afficher modal d'ajout de livre
function showAddBookModal() {
    document.getElementById('bookModalTitle').textContent = 'Ajouter un livre';
    document.getElementById('bookForm').reset();
    document.getElementById('bookId').value = '';
    document.getElementById('coverPreview').innerHTML = '<i class="fas fa-image fa-3x text-muted"></i>';

    new bootstrap.Modal(document.getElementById('bookModal')).show();
}

// Modifier un livre
function editBook(bookId) {
    fetch(`/api/books/${bookId}`)
        .then(response => response.json())
        .then(book => {
            document.getElementById('bookModalTitle').textContent = 'Modifier le livre';
            document.getElementById('bookId').value = book.id;
            document.getElementById('bookTitle').value = book.titre;
            document.getElementById('bookAuthor').value = book.auteur;
            document.getElementById('bookIsbn').value = book.isbn || '';
            document.getElementById('bookGenre').value = book.genre_id || '';
            document.getElementById('bookYear').value = book.annee_publication || '';
            document.getElementById('bookPages').value = book.nombre_pages || '';
            document.getElementById('bookSummary').value = book.resume || '';

            // Afficher la couverture existante
            if (book.couverture_url) {
                document.getElementById('coverPreview').innerHTML =
                    `<img src="${book.couverture_url}" style="max-width: 100%; max-height: 200px;">`;
            } else {
                document.getElementById('coverPreview').innerHTML = '<i class="fas fa-image fa-3x text-muted"></i>';
            }

            new bootstrap.Modal(document.getElementById('bookModal')).show();
        })
        .catch(error => {
            console.error('Erreur:', error);
            showAlert('Erreur lors du chargement du livre', 'danger');
        });
}

// Sauvegarder un livre
function saveBook() {
    const bookId = document.getElementById('bookId').value;
    const isEdit = bookId !== '';

    const bookData = {
        titre: document.getElementById('bookTitle').value,
        auteur: document.getElementById('bookAuthor').value,
        isbn: document.getElementById('bookIsbn').value,
        genre_id: document.getElementById('bookGenre').value || null,
        annee_publication: document.getElementById('bookYear').value || null,
        nombre_pages: document.getElementById('bookPages').value || null,
        resume: document.getElementById('bookSummary').value
    };

    // Validation côté client
    if (!bookData.titre || !bookData.auteur) {
        showAlert('Titre et auteur sont obligatoires', 'danger');
        return;
    }

    const url = isEdit ? `/api/books/${bookId}` : '/api/books';
    const method = isEdit ? 'PUT' : 'POST';

    fetch(url, {
        method: method,
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify(bookData)
    })
    .then(response => response.json())
    .then(data => {
        if (data.error) {
            showAlert(data.error, 'danger');
        } else {
            showAlert(data.message, 'success');
            closeModal('bookModal');
            loadBooks();
        }
    })
    .catch(error => {
        console.error('Erreur:', error);
        showAlert('Erreur lors de la sauvegarde', 'danger');
    });
}

// Supprimer un livre
function deleteBook(bookId) {
    if (!confirm('Êtes-vous sûr de vouloir supprimer ce livre ?')) return;

    fetch(`/api/books/${bookId}`, {
        method: 'DELETE'
    })
    .then(response => response.json())
    .then(data => {
        if (data.error) {
            showAlert(data.error, 'danger');
        } else {
            showAlert(data.message, 'success');
            loadBooks();
        }
    })
    .catch(error => {
        console.error('Erreur:', error);
        showAlert('Erreur lors de la suppression', 'danger');
    });
}

// Preview de la couverture
function previewCover(input) {
    if (input.files && input.files[0]) {
        const reader = new FileReader();
        reader.onload = function(e) {
            document.getElementById('coverPreview').innerHTML =
                `<img src="${e.target.result}" style="max-width: 100%; max-height: 200px;">`;
        };
        reader.readAsDataURL(input.files[0]);
    }
}

// Utilitaires
function closeModal(modalId) {
    const modal = bootstrap.Modal.getInstance(document.getElementById(modalId));
    if (modal) modal.hide();
}

function showAlert(message, type) {
    const alertContainer = document.querySelector('.container');
    const alert = document.createElement('div');
    alert.className = `alert alert-${type} alert-dismissible fade show`;
    alert.innerHTML = `
        ${message}
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    `;

    // Insérer au début du container
    alertContainer.insertBefore(alert, alertContainer.firstChild);

    // Auto-remove après 5 secondes
    setTimeout(() => {
        if (alert.parentNode) {
            alert.remove();
        }
    }, 5000);
}

// Variables globales nécessaires
const isAdmin = document.body.dataset.isAdmin === 'true';

// Fonctions pour les emprunts (à appeler depuis les menus)
function showUserLoans() {
    fetch('/api/loans')
        .then(response => response.json())
        .then(loans => {
            let content = '<h5>Mes emprunts en cours</h5>';

            if (loans.length === 0) {
                content += '<p class="text-muted">Aucun emprunt en cours</p>';
            } else {
                content += '<div class="list-group">';
                loans.forEach(loan => {
                    const dueDate = new Date(loan.date_retour_prevue);
                    const isOverdue = dueDate < new Date();

                    content += `
                        <div class="list-group-item">
                            <div class="d-flex justify-content-between align-items-start">
                                <div>
                                    <h6 class="mb-1">${loan.titre}</h6>
                                    <p class="mb-1">par ${loan.auteur}</p>
                                    <small class="text-muted">
                                        À rendre le:
                                        <span class="${isOverdue ? 'text-danger fw-bold' : ''}">${dueDate.toLocaleDateString('fr-FR')}</span>
                                        ${isOverdue ? ' (EN RETARD)' : ''}
                                    </small>
                                </div>
                                <button class="btn btn-sm btn-outline-primary" onclick="returnBook(${loan.id})">
                                    Retourner
                                </button>
                            </div>
                        </div>
                    `;
                });
                content += '</div>';
            }

            // Créer et afficher un modal simple
            showSimpleModal('Mes emprunts', content);
        })
        .catch(error => {
            console.error('Erreur:', error);
            showAlert('Erreur lors du chargement des emprunts', 'danger');
        });
}

function returnBook(loanId) {
    if (!confirm('Confirmer le retour de ce livre ?')) return;

    fetch(`/api/loans/${loanId}/return`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        }
    })
    .then(response => response.json())
    .then(data => {
        if (data.error) {
            showAlert(data.error, 'danger');
        } else {
            showAlert(data.message, 'success');
            showUserLoans(); // Recharger la liste
            loadBooks(); // Mettre à jour l'affichage des livres
        }
    })
    .catch(error => {
        console.error('Erreur:', error);
        showAlert('Erreur lors du retour', 'danger');
    });
}

function showSimpleModal(title, content) {
    // Supprimer le modal existant s'il y en a un
    const existingModal = document.getElementById('simpleModal');
    if (existingModal) {
        existingModal.remove();
    }

    // Créer un nouveau modal
    const modalHtml = `
        <div class="modal fade" id="simpleModal" tabindex="-1">
            <div class="modal-dialog">
                <div class="modal-content">
                    <div class="modal-header">
                        <h5 class="modal-title">${title}</h5>
                        <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                    </div>
                    <div class="modal-body">
                        ${content}
                    </div>
                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Fermer</button>
                    </div>
                </div>
            </div>
        </div>
    `;

    document.body.insertAdjacentHTML('beforeend', modalHtml);
    new bootstrap.Modal(document.getElementById('simpleModal')).show();
}

function showAdminPanel() {
    window.location.href = '/admin'; // À implémenter si besoin
}
```

## CSS Personnalisé

```css
/* static/css/style.css */

/* Variables CSS personnalisées */
:root {
    --primary-color: #3498db;
    --secondary-color: #2c3e50;
    --success-color: #27ae60;
    --warning-color: #f39c12;
    --danger-color: #e74c3c;
    --light-bg: #f8f9fa;
    --shadow: 0 2px 10px rgba(0,0,0,0.1);
    --border-radius: 8px;
}

/* Styles généraux */
body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background-color: var(--light-bg);
}

/* Navigation */
.navbar-brand {
    font-weight: bold;
    font-size: 1.5rem;
}

.navbar-nav .nav-link {
    font-weight: 500;
    transition: color 0.3s ease;
}

.navbar-nav .nav-link:hover {
    color: rgba(255,255,255,0.8) !important;
}

/* Cards */
.card {
    border: none;
    box-shadow: var(--shadow);
    transition: transform 0.2s ease, box-shadow 0.2s ease;
    border-radius: var(--border-radius);
}

.card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 20px rgba(0,0,0,0.15);
}

.card-img-top {
    border-radius: var(--border-radius) var(--border-radius) 0 0;
}

/* Hero Section */
.jumbotron {
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
}

/* Boutons */
.btn {
    border-radius: var(--border-radius);
    font-weight: 500;
    transition: all 0.3s ease;
}

.btn:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.btn-primary {
    background-color: var(--primary-color);
    border-color: var(--primary-color);
}

.btn-success {
    background-color: var(--success-color);
    border-color: var(--success-color);
}

/* Badges */
.badge {
    border-radius: 20px;
    padding: 0.4em 0.8em;
    font-weight: 500;
}

/* Tables */
.table {
    background-color: white;
    border-radius: var(--border-radius);
    overflow: hidden;
    box-shadow: var(--shadow);
}

.table th {
    background-color: var(--light-bg);
    border-bottom: 2px solid #dee2e6;
    font-weight: 600;
}

.table tbody tr:hover {
    background-color: rgba(52, 152, 219, 0.05);
}

/* Formulaires */
.form-control, .form-select {
    border-radius: var(--border-radius);
    border: 1px solid #ddd;
    transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.form-control:focus, .form-select:focus {
    border-color: var(--primary-color);
    box-shadow: 0 0 0 0.2rem rgba(52, 152, 219, 0.25);
}

/* Modals */
.modal-content {
    border: none;
    border-radius: var(--border-radius);
    box-shadow: 0 10px 30px rgba(0,0,0,0.2);
}

.modal-header {
    background-color: var(--light-bg);
    border-bottom: 1px solid #dee2e6;
    border-radius: var(--border-radius) var(--border-radius) 0 0;
}

.modal-footer {
    background-color: var(--light-bg);
    border-top: 1px solid #dee2e6;
    border-radius: 0 0 var(--border-radius) var(--border-radius);
}

/* Alerts */
.alert {
    border: none;
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
}

/* Sidebar filtres */
.card-header {
    background-color: var(--primary-color);
    color: white;
    border-bottom: none;
    font-weight: 600;
}

/* Animations */
@keyframes fadeIn {
    from { opacity: 0; transform: translateY(20px); }
    to { opacity: 1; transform: translateY(0); }
}

.card {
    animation: fadeIn 0.5s ease;
}

/* Responsive */
@media (max-width: 768px) {
    .container-fluid {
        padding: 0 10px;
    }

    .card-body {
        padding: 1rem 0.75rem;
    }

    .btn {
        padding: 0.375rem 0.5rem;
        font-size: 0.875rem;
    }

    .table-responsive {
        font-size: 0.875rem;
    }
}

/* Styles spécifiques aux livres */
.book-cover {
    transition: transform 0.3s ease;
}

.book-cover:hover {
    transform: scale(1.05);
}

.book-card {
    position: relative;
    overflow: hidden;
}

.book-card::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: linear-gradient(135deg, rgba(52, 152, 219, 0.1), rgba(155, 89, 182, 0.1));
    opacity: 0;
    transition: opacity 0.3s ease;
}

.book-card:hover::before {
    opacity: 1;
}

/* Indicateurs de statut */
.status-available {
    border-left: 4px solid var(--success-color);
}

.status-borrowed {
    border-left: 4px solid var(--warning-color);
}

.status-overdue {
    border-left: 4px solid var(--danger-color);
}

# 9.4 - Finalisation : CSS, Config et Déploiement

## CSS (suite et finalisation)

/* Spinner de chargement */
.loading-spinner {
    display: inline-block;
    width: 20px;
    height: 20px;
    border: 3px solid #f3f3f3;
    border-top: 3px solid var(--primary-color);
    border-radius: 50%;
    animation: spin 1s linear infinite;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

.loading-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(255, 255, 255, 0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 9999;
}

/* Styles pour les statistiques */
.stat-card {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border-radius: var(--border-radius);
    padding: 2rem;
    text-align: center;
    box-shadow: var(--shadow);
}

.stat-number {
    font-size: 3rem;
    font-weight: bold;
    margin-bottom: 0.5rem;
}

.stat-label {
    font-size: 1.1rem;
    opacity: 0.9;
}

/* Charts et graphiques */
.chart-container {
    background: white;
    border-radius: var(--border-radius);
    padding: 1.5rem;
    box-shadow: var(--shadow);
    margin-bottom: 2rem;
}

/* Styles pour la recherche avancée */
.search-container {
    position: relative;
}

.search-suggestions {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    background: white;
    border: 1px solid #ddd;
    border-top: none;
    border-radius: 0 0 var(--border-radius) var(--border-radius);
    box-shadow: var(--shadow);
    z-index: 1000;
    max-height: 300px;
    overflow-y: auto;
}

.search-suggestion {
    padding: 0.75rem 1rem;
    cursor: pointer;
    border-bottom: 1px solid #f0f0f0;
    transition: background-color 0.2s ease;
}

.search-suggestion:hover {
    background-color: var(--light-bg);
}

.search-suggestion:last-child {
    border-bottom: none;
}

/* Styles pour les emprunts */
.loan-item {
    border-left: 4px solid var(--primary-color);
    background: white;
    border-radius: var(--border-radius);
    padding: 1rem;
    margin-bottom: 1rem;
    box-shadow: var(--shadow);
}

.loan-item.overdue {
    border-left-color: var(--danger-color);
    background-color: #fff5f5;
}

.loan-item.due-soon {
    border-left-color: var(--warning-color);
    background-color: #fffbf0;
}

/* Styles pour les genres */
.genre-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 1rem;
    margin-bottom: 2rem;
}

.genre-card {
    padding: 1.5rem;
    text-align: center;
    border-radius: var(--border-radius);
    color: white;
    text-decoration: none;
    transition: transform 0.3s ease, box-shadow 0.3s ease;
    box-shadow: var(--shadow);
}

.genre-card:hover {
    transform: translateY(-3px);
    box-shadow: 0 6px 20px rgba(0,0,0,0.15);
    color: white;
    text-decoration: none;
}

.genre-count {
    font-size: 2rem;
    font-weight: bold;
    margin-bottom: 0.5rem;
}

/* Pagination */
.pagination {
    justify-content: center;
    margin-top: 2rem;
}

.page-link {
    border-radius: var(--border-radius);
    margin: 0 2px;
    border: 1px solid #ddd;
    color: var(--primary-color);
}

.page-link:hover {
    background-color: var(--primary-color);
    border-color: var(--primary-color);
    color: white;
}

.page-item.active .page-link {
    background-color: var(--primary-color);
    border-color: var(--primary-color);
}

/* Styles pour les messages toast */
.toast-container {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 1100;
}

.toast {
    border: none;
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
}

/* Styles d'impression */
@media print {
    .navbar, .btn, .modal, .sidebar {
        display: none !important;
    }

    .card {
        border: 1px solid #ddd !important;
        box-shadow: none !important;
    }

    .container-fluid {
        margin: 0 !important;
        padding: 0 !important;
    }
}

/* Accessibilité */
.sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}

/* Focus visible pour l'accessibilité */
.btn:focus-visible,
.form-control:focus-visible,
.form-select:focus-visible {
    outline: 2px solid var(--primary-color);
    outline-offset: 2px;
}

/* Dark mode (optionnel) */
@media (prefers-color-scheme: dark) {
    :root {
        --light-bg: #2c3e50;
        --shadow: 0 2px 10px rgba(0,0,0,0.3);
    }

    body {
        background-color: #34495e;
        color: #ecf0f1;
    }

    .card {
        background-color: var(--light-bg);
        color: #ecf0f1;
    }

    .table {
        background-color: var(--light-bg);
        color: #ecf0f1;
    }

    .modal-content {
        background-color: var(--light-bg);
        color: #ecf0f1;
    }
}
```

## Configuration et fichiers de démarrage

### Configuration principale

```python
# config.py
import os
from datetime import timedelta

class Config:
    """Configuration de base"""
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key-change-in-production'

    # Base de données
    DATABASE_PATH = os.environ.get('DATABASE_PATH') or 'data/bibliotheque.db'

    # Upload de fichiers
    UPLOAD_FOLDER = 'static/uploads'
    MAX_CONTENT_LENGTH = 16 * 1024 * 1024  # 16MB
    ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif', 'webp'}

    # Session
    PERMANENT_SESSION_LIFETIME = timedelta(days=7)

    # Pagination
    BOOKS_PER_PAGE = 12
    LOANS_PER_PAGE = 20

    # Email (pour les notifications de retard)
    MAIL_SERVER = os.environ.get('MAIL_SERVER')
    MAIL_PORT = int(os.environ.get('MAIL_PORT') or 587)
    MAIL_USE_TLS = os.environ.get('MAIL_USE_TLS', 'true').lower() in ['true', 'on', '1']
    MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')

    # Sécurité
    WTF_CSRF_ENABLED = True
    WTF_CSRF_TIME_LIMIT = 3600

class DevelopmentConfig(Config):
    """Configuration pour le développement"""
    DEBUG = True
    TESTING = False

class ProductionConfig(Config):
    """Configuration pour la production"""
    DEBUG = False
    TESTING = False

    # Renforcer la sécurité en production
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = 'Lax'

class TestingConfig(Config):
    """Configuration pour les tests"""
    TESTING = True
    DATABASE_PATH = ':memory:'  # Base en mémoire pour les tests
    WTF_CSRF_ENABLED = False

# Dictionnaire des configurations
config = {
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'testing': TestingConfig,
    'default': DevelopmentConfig
}
```

### Point d'entrée principal

```python
# run.py
import os
from app import create_app
from app.database import DatabaseManager

# Déterminer l'environnement
config_name = os.environ.get('FLASK_CONFIG') or 'default'

# Créer l'application
app = create_app(config_name)

# Initialiser la base de données au démarrage
with app.app_context():
    db_manager = DatabaseManager()
    print("✅ Base de données initialisée")

if __name__ == '__main__':
    # Configuration pour le développement
    port = int(os.environ.get('PORT', 5000))
    debug = os.environ.get('FLASK_ENV') == 'development'

    print(f"🚀 Démarrage de Ma Bibliothèque sur le port {port}")
    print(f"🔧 Mode debug: {debug}")
    print(f"📁 Base de données: {app.config['DATABASE_PATH']}")

    app.run(
        host='0.0.0.0',
        port=port,
        debug=debug,
        threaded=True
    )
```

### Dépendances du projet

```txt
# requirements.txt
Flask==2.3.3
Werkzeug==2.3.7
Jinja2==3.1.2
click==8.1.7
itsdangerous==2.1.2
MarkupSafe==2.1.3

# Pour les uploads d'images
Pillow==10.0.1

# Pour l'envoi d'emails
Flask-Mail==0.9.1

# Pour la validation des formulaires
Flask-WTF==1.1.1
WTForms==3.0.1

# Pour les tests
pytest==7.4.2
pytest-flask==1.2.0

# Outils de développement
python-dotenv==1.0.0
flask-shell-ipython==1.4.0
```

### Variables d'environnement

```bash
# .env (fichier d'environnement pour le développement)
FLASK_ENV=development
FLASK_CONFIG=development
SECRET_KEY=your-secret-key-here
DATABASE_PATH=data/bibliotheque.db

# Configuration email (optionnel)
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_USE_TLS=true
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password

# Configuration pour la production
# FLASK_ENV=production
# FLASK_CONFIG=production
```

## Scripts d'administration

### Script de gestion des données

```python
# scripts/manage_data.py
import sys
import os
sys.path.append(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

import sqlite3
import csv
from datetime import datetime
from app.models import Livre, Utilisateur, Emprunt, Genre
from app.database import DatabaseManager

class DataManager:
    def __init__(self):
        self.db_manager = DatabaseManager()

    def export_books_csv(self, filename='exports/livres_export.csv'):
        """Exporte tous les livres vers un fichier CSV"""
        os.makedirs(os.path.dirname(filename), exist_ok=True)

        livre_model = Livre()
        livres = livre_model.get_all()

        with open(filename, 'w', newline='', encoding='utf-8') as csvfile:
            fieldnames = ['id', 'titre', 'auteur', 'isbn', 'genre', 'annee_publication',
                         'nombre_pages', 'langue', 'resume', 'disponible', 'date_ajout']
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)

            writer.writeheader()
            for livre in livres:
                writer.writerow({
                    'id': livre['id'],
                    'titre': livre['titre'],
                    'auteur': livre['auteur'],
                    'isbn': livre['isbn'],
                    'genre': livre['genre_nom'],
                    'annee_publication': livre['annee_publication'],
                    'nombre_pages': livre['nombre_pages'],
                    'langue': livre['langue'],
                    'resume': livre['resume'],
                    'disponible': livre['disponible'],
                    'date_ajout': livre['date_ajout']
                })

        print(f"✅ {len(livres)} livres exportés vers {filename}")

    def import_books_csv(self, filename):
        """Importe des livres depuis un fichier CSV"""
        if not os.path.exists(filename):
            print(f"❌ Fichier {filename} non trouvé")
            return

        livre_model = Livre()
        genre_model = Genre()
        genres = {g['nom']: g['id'] for g in genre_model.get_all()}

        imported = 0
        errors = 0

        with open(filename, 'r', encoding='utf-8') as csvfile:
            reader = csv.DictReader(csvfile)

            for row in reader:
                try:
                    genre_id = genres.get(row.get('genre'))

                    livre_data = {
                        'titre': row['titre'],
                        'auteur': row['auteur'],
                        'isbn': row.get('isbn') or None,
                        'genre_id': genre_id,
                        'annee_publication': int(row['annee_publication']) if row.get('annee_publication') else None,
                        'nombre_pages': int(row['nombre_pages']) if row.get('nombre_pages') else None,
                        'langue': row.get('langue', 'fr'),
                        'resume': row.get('resume')
                    }

                    if livre_model.create(livre_data):
                        imported += 1
                    else:
                        errors += 1
                        print(f"❌ Erreur import: {row['titre']}")

                except Exception as e:
                    errors += 1
                    print(f"❌ Erreur ligne {reader.line_num}: {e}")

        print(f"✅ Import terminé: {imported} livres importés, {errors} erreurs")

    def backup_database(self, backup_path=None):
        """Crée une sauvegarde de la base de données"""
        if backup_path is None:
            timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
            backup_path = f"backups/bibliotheque_backup_{timestamp}.db"

        os.makedirs(os.path.dirname(backup_path), exist_ok=True)

        import shutil
        shutil.copy2(self.db_manager.db_path, backup_path)
        print(f"✅ Sauvegarde créée: {backup_path}")

    def restore_database(self, backup_path):
        """Restaure la base depuis une sauvegarde"""
        if not os.path.exists(backup_path):
            print(f"❌ Fichier de sauvegarde {backup_path} non trouvé")
            return

        if input("⚠️ Confirmer la restauration ? (toutes les données actuelles seront perdues) [y/N]: ").lower() != 'y':
            print("Restauration annulée")
            return

        import shutil
        shutil.copy2(backup_path, self.db_manager.db_path)
        print(f"✅ Base restaurée depuis {backup_path}")

    def cleanup_old_loans(self, days=365):
        """Nettoie les anciens emprunts terminés"""
        with sqlite3.connect(self.db_manager.db_path) as conn:
            cursor = conn.execute('''
                DELETE FROM emprunts
                WHERE statut = 'retourne'
                AND date_retour_effective < DATE('now', '-{} days')
            '''.format(days))

            deleted = cursor.rowcount
            print(f"✅ {deleted} anciens emprunts supprimés")

    def generate_stats_report(self):
        """Génère un rapport de statistiques"""
        livre_model = Livre()
        emprunt_model = Emprunt()

        tous_livres = livre_model.get_all()
        emprunts_actifs = emprunt_model.get_active_loans()
        emprunts_retard = emprunt_model.get_overdue_loans()

        print("=" * 50)
        print("📊 RAPPORT DE STATISTIQUES")
        print("=" * 50)
        print(f"📚 Total livres: {len(tous_livres)}")
        print(f"✅ Livres disponibles: {len([l for l in tous_livres if l['disponible']])}")
        print(f"📖 Livres empruntés: {len([l for l in tous_livres if not l['disponible']])}")
        print(f"🤝 Emprunts actifs: {len(emprunts_actifs)}")
        print(f"⚠️ Emprunts en retard: {len(emprunts_retard)}")
        print("=" * 50)

if __name__ == '__main__':
    import argparse

    parser = argparse.ArgumentParser(description='Gestion des données de la bibliothèque')
    parser.add_argument('action', choices=['export', 'import', 'backup', 'restore', 'cleanup', 'stats'],
                       help='Action à effectuer')
    parser.add_argument('--file', help='Fichier pour import/export/backup/restore')
    parser.add_argument('--days', type=int, default=365, help='Nombre de jours pour le nettoyage')

    args = parser.parse_args()
    manager = DataManager()

    if args.action == 'export':
        filename = args.file or f'exports/livres_{datetime.now().strftime("%Y%m%d")}.csv'
        manager.export_books_csv(filename)

    elif args.action == 'import':
        if not args.file:
            print("❌ Fichier requis pour l'import")
            sys.exit(1)
        manager.import_books_csv(args.file)

    elif args.action == 'backup':
        manager.backup_database(args.file)

    elif args.action == 'restore':
        if not args.file:
            print("❌ Fichier de sauvegarde requis")
            sys.exit(1)
        manager.restore_database(args.file)

    elif args.action == 'cleanup':
        manager.cleanup_old_loans(args.days)

    elif args.action == 'stats':
        manager.generate_stats_report()
```

## Déploiement et mise en production

### Configuration pour Docker

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Variables d'environnement
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV FLASK_ENV=production

# Répertoire de travail
WORKDIR /app

# Installer les dépendances système
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copier les requirements et installer les dépendances Python
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copier le code de l'application
COPY . .

# Créer les dossiers nécessaires
RUN mkdir -p data static/uploads backups exports logs

# Exposer le port
EXPOSE 5000

# Commande de démarrage
CMD ["python", "run.py"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  bibliotheque:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - ./data:/app/data
      - ./static/uploads:/app/static/uploads
      - ./backups:/app/backups
    environment:
      - FLASK_ENV=production
      - FLASK_CONFIG=production
      - SECRET_KEY=your-super-secret-key-here
    restart: unless-stopped

  # Service de sauvegarde automatique
  backup:
    image: python:3.11-slim
    volumes:
      - ./data:/data
      - ./backups:/backups
    command: |
      sh -c "
      while true; do
        sleep 86400;  # 24 heures
        cp /data/bibliotheque.db /backups/auto_backup_$$(date +%Y%m%d_%H%M%S).db;
        find /backups -name 'auto_backup_*.db' -type f -mtime +7 -delete;
        echo 'Sauvegarde automatique effectuée';
      done"
    restart: unless-stopped
```

### Script de déploiement

```bash
#!/bin/bash
# deploy.sh

echo "🚀 Déploiement de Ma Bibliothèque"

# Arrêter l'application existante
docker-compose down

# Sauvegarder la base actuelle
if [ -f "data/bibliotheque.db" ]; then
    cp data/bibliotheque.db backups/pre_deploy_$(date +%Y%m%d_%H%M%S).db
    echo "✅ Sauvegarde pré-déploiement créée"
fi

# Récupérer les dernières modifications
git pull origin main

# Reconstruire et démarrer
docker-compose up --build -d

# Attendre que l'application démarre
sleep 10

# Vérifier que l'application fonctionne
if curl -f http://localhost:5000 > /dev/null 2>&1; then
    echo "✅ Application déployée avec succès"
    echo "🌐 Accessible sur http://localhost:5000"
else
    echo "❌ Erreur de déploiement"
    docker-compose logs
    exit 1
fi

echo "📊 Statistiques post-déploiement:"
python scripts/manage_data.py stats
```

## Tests automatisés

### Configuration des tests

```python
# tests/conftest.py
import pytest
import tempfile
import os
from app import create_app
from app.database import DatabaseManager

@pytest.fixture
def app():
    """Fixture pour l'application Flask de test"""
    # Créer un fichier temporaire pour la base de test
    db_fd, db_path = tempfile.mkstemp()

    app = create_app('testing')
    app.config['DATABASE_PATH'] = db_path
    app.config['TESTING'] = True

    with app.app_context():
        # Initialiser la base de test
        db_manager = DatabaseManager(db_path)

    yield app

    # Nettoyage
    os.close(db_fd)
    os.unlink(db_path)

@pytest.fixture
def client(app):
    """Client de test Flask"""
    return app.test_client()

@pytest.fixture
def runner(app):
    """Runner CLI de test"""
    return app.test_cli_runner()
```

### Tests des modèles

```python
# tests/test_models.py
import pytest
from app.models import Livre, Utilisateur, Emprunt, Genre

def test_livre_creation(app):
    """Test de création d'un livre"""
    with app.app_context():
        livre_model = Livre()

        livre_data = {
            'titre': 'Test Livre',
            'auteur': 'Auteur Test',
            'isbn': '1234567890',
            'annee_publication': 2024
        }

        livre_id = livre_model.create(livre_data)
        assert livre_id is not None

        # Vérifier que le livre a été créé
        livre = livre_model.get_by_id(livre_id)
        assert livre['titre'] == 'Test Livre'
        assert livre['auteur'] == 'Auteur Test'

def test_utilisateur_authentification(app):
    """Test d'authentification utilisateur"""
    with app.app_context():
        user_model = Utilisateur()

        # Créer un utilisateur
        user_data = {
            'nom': 'Test User',
            'email': 'test@example.com',
            'mot_de_passe': 'password123'
        }

        user_id = user_model.create(user_data)
        assert user_id is not None

        # Tester l'authentification
        user = user_model.authenticate('test@example.com', 'password123')
        assert user is not None
        assert user['nom'] == 'Test User'

        # Tester avec un mauvais mot de passe
        user = user_model.authenticate('test@example.com', 'wrongpassword')
        assert user is None

def test_emprunt_workflow(app):
    """Test du workflow d'emprunt"""
    with app.app_context():
        livre_model = Livre()
        user_model = Utilisateur()
        emprunt_model = Emprunt()

        # Créer un livre et un utilisateur
        livre_id = livre_model.create({
            'titre': 'Livre Test',
            'auteur': 'Auteur Test'
        })

        user_id = user_model.create({
            'nom': 'User Test',
            'email': 'user@test.com',
            'mot_de_passe': 'password'
        })

        # Créer un emprunt
        emprunt_id, message = emprunt_model.create({
            'livre_id': livre_id,
            'utilisateur_id': user_id
        })

        assert emprunt_id is not None

        # Vérifier que le livre n'est plus disponible
        livre = livre_model.get_by_id(livre_id)
        assert not livre['disponible']

        # Retourner le livre
        success, message = emprunt_model.return_book(emprunt_id)
        assert success

        # Vérifier que le livre est à nouveau disponible
        livre = livre_model.get_by_id(livre_id)
        assert livre['disponible']
```

## Conclusion du projet

### Récapitulatif des fonctionnalités implémentées

✅ **Gestion complète des livres** : CRUD avec images, genres, recherche
✅ **Système d'utilisateurs** : Authentification, rôles admin/utilisateur
✅ **Gestion des emprunts** : Emprunts, retours, suivi des retards
✅ **Interface moderne** : Responsive design, vues grille/liste
✅ **API REST** : Endpoints complets pour toutes les fonctionnalités
✅ **Base SQLite optimisée** : Index, contraintes, performances
✅ **Outils d'administration** : Scripts de gestion, sauvegarde, import/export
✅ **Déploiement production** : Docker, configuration sécurisée
✅ **Tests automatisés** : Couverture des fonctionnalités principales

### Fonctionnalités avancées possibles

Pour aller plus loin, vous pourriez ajouter :

- 📧 **Notifications email** pour les retards
- 📊 **Tableaux de bord** avec graphiques interactifs
- 🔍 **Recherche full-text** avec SQLite FTS5
- 📱 **API mobile** pour une app smartphone
- 🌐 **Internationalisation** multi-langues
- 📖 **Système de recommandations** basé sur l'historique
- 💾 **Synchronisation cloud** avec services externes
- 🔐 **Authentification OAuth** (Google, GitHub)

### Points clés de l'architecture

Ce projet démontre les **bonnes pratiques** avec SQLite :

1. **Architecture MVC** claire et maintenable
2. **Séparation des responsabilités** entre couches
3. **Optimisation des requêtes** avec index appropriés
4. **Gestion des erreurs** robuste et complète
5. **Sécurité** des données et authentification
6. **Tests** automatisés pour la fiabilité
7. **Déploiement** professionnel avec Docker

### Retour d'expérience

Ce projet montre que **SQLite est parfaitement adapté** pour des applications web de taille moyenne. Sa simplicité de déploiement, ses performances et sa fiabilité en font un choix excellent pour de nombreux cas d'usage.

L'application "Ma Bibliothèque" est un exemple concret et complet qui peut servir de base pour vos propres projets utilisant SQLite.

---

**🎉 Félicitations !**

Vous avez maintenant une application web complète et fonctionnelle utilisant SQLite. Ce projet illustre parfaitement comment tirer parti de toute la puissance de SQLite dans un contexte réel et professionnel.

⏭️
