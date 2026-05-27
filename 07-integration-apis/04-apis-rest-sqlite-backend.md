🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.4 : APIs REST avec SQLite en backend

## Introduction

Une API REST (Representational State Transfer) est comme un serveur de restaurant : vous passez une commande (requête HTTP), et on vous apporte ce que vous avez demandé (réponse JSON). C'est un moyen standardisé pour que différentes applications communiquent entre elles via Internet.

Dans cette section, nous allons apprendre à créer des APIs REST qui utilisent SQLite comme base de données. C'est parfait pour :
- **Applications web** : Frontend (React, Vue) + Backend API
- **Applications mobiles** : iOS/Android qui consomment une API
- **Microservices** : Petits services spécialisés
- **Prototypage rapide** : Tester des idées rapidement

## Qu'est-ce qu'une API REST ?

### Concepts de base

**REST** utilise les méthodes HTTP standard :
- `GET` : Récupérer des données (comme "Montre-moi...")
- `POST` : Créer de nouvelles données (comme "Ajoute...")
- `PUT` : Mettre à jour des données complètes (comme "Remplace...")
- `PATCH` : Mettre à jour partiellement (comme "Modifie juste...")
- `DELETE` : Supprimer des données (comme "Efface...")

### Structure typique d'une API REST

```
GET    /api/personnes          → Lister toutes les personnes  
GET    /api/personnes/123      → Récupérer la personne avec l'ID 123  
POST   /api/personnes          → Créer une nouvelle personne  
PUT    /api/personnes/123      → Mettre à jour complètement la personne 123  
PATCH  /api/personnes/123      → Mettre à jour partiellement la personne 123  
DELETE /api/personnes/123      → Supprimer la personne 123  
```

### Format des données

Les APIs REST échangent généralement des données au format JSON :

```json
{
  "id": 123,
  "nom": "Alice Martin",
  "age": 25,
  "email": "alice.martin@email.com",
  "date_creation": "2024-01-15T10:30:00Z"
}
```

## API REST avec Python (Flask)

Flask est un framework web Python simple et parfait pour les débutants.

### Installation

```bash
pip install flask flask-sqlalchemy flask-cors
```

### Structure du projet

```
mon-api/
├── app.py              # Application principale
├── models.py           # Modèles de données
├── config.py           # Configuration
├── requirements.txt    # Dépendances
└── instance/
    └── database.db     # Base de données SQLite
```

### Configuration de base

**config.py**
```python
import os

class Config:
    # Configuration de la base de données
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
        'sqlite:///instance/database.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False

    # Configuration générale
    # ⚠️ En production, la SECRET_KEY DOIT venir de l'environnement, sans fallback.
    #    Le fallback ci-dessous est uniquement pour le développement local.
    #    Un secret en clair dans le code = compromission de toutes les sessions
    #    et tokens signés si le code fuite.
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key-change-in-production'
    DEBUG = True

# Pour la production, utilisez plutôt :
class ProductionConfig(Config):
    DEBUG = False
    # SECRET_KEY ne doit JAMAIS avoir de fallback en prod
    SECRET_KEY = os.environ['SECRET_KEY']  # KeyError si absent → c'est voulu
```

> 💡 **Note Flask-SQLAlchemy 3** : `Model.query` (par exemple `Personne.query.all()`) reste supporté mais est **deprecated**. La syntaxe recommandée est `db.session.execute(db.select(Personne)).scalars().all()`. Les exemples ci-dessous utilisent l'API legacy pour rester concis et lisibles, mais privilégiez la nouvelle syntaxe dans le code neuf.

**models.py**
```python
from flask_sqlalchemy import SQLAlchemy  
from datetime import datetime, timezone  
import re  

db = SQLAlchemy()

class Personne(db.Model):
    __tablename__ = 'personnes'

    id = db.Column(db.Integer, primary_key=True)
    nom = db.Column(db.String(100), nullable=False)
    age = db.Column(db.Integer, nullable=True)
    email = db.Column(db.String(120), unique=True, nullable=False)
    # ⚠️ datetime.utcnow déprécié depuis Python 3.12 → datetime.now(timezone.utc)
    date_creation = db.Column(db.DateTime, default=lambda: datetime.now(timezone.utc))

    def __init__(self, nom, age, email):
        self.nom = nom
        self.age = age
        self.email = email
        self.valider()

    def valider(self):
        """Validation des données"""
        if not self.nom or len(self.nom.strip()) < 2:
            raise ValueError("Le nom doit contenir au moins 2 caractères")

        if self.age is not None and (self.age < 0 or self.age > 150):
            raise ValueError("L'âge doit être entre 0 et 150 ans")

        if not re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', self.email):
            raise ValueError("Format d'email invalide")

    def to_dict(self):
        """Convertir en dictionnaire pour JSON"""
        return {
            'id': self.id,
            'nom': self.nom,
            'age': self.age,
            'email': self.email,
            'date_creation': self.date_creation.isoformat() if self.date_creation else None
        }

    def __repr__(self):
        return f'<Personne {self.nom}>'
```

### Application Flask complète

**app.py**
```python
from flask import Flask, request, jsonify  
from flask_cors import CORS  
from models import db, Personne  
from config import Config  
import os  

# Créer l'application Flask
app = Flask(__name__)  
app.config.from_object(Config)  

# Initialiser les extensions
db.init_app(app)  
CORS(app)  # Permettre les requêtes cross-origin  

# Créer le dossier instance s'il n'existe pas
os.makedirs('instance', exist_ok=True)

# Créer les tables
with app.app_context():
    db.create_all()

# ==================== ROUTES API ====================
#
# ⚠️ NOTE TRANSVERSALE SUR LES MESSAGES D'ERREUR :
# Les routes ci-dessous retournent `str(e)` au client en cas d'exception non
# prévue. C'est PÉDAGOGIQUE mais à proscrire en PRODUCTION : un message comme
# "no such column: u.password_hash" révèle votre schéma à un attaquant.
#
# Pratique recommandée :
#   try:
#       ...
#   except SQLAlchemyError as e:
#       app.logger.exception("Erreur BDD")           # log complet côté serveur
#       return jsonify({'error': 'Erreur serveur'}), 500   # message générique
#   except ValueError as e:
#       return jsonify({'error': str(e)}), 400       # OK : erreur métier explicite
#
# Distinguer aussi 500 (bug serveur) de 503 (BDD indisponible).

@app.route('/api/health', methods=['GET'])
def health_check():
    """Point de santé de l'API"""
    return jsonify({
        'status': 'OK',
        'message': 'API fonctionnelle',
        'version': '1.0.0'
    })

@app.route('/api/personnes', methods=['GET'])
def lister_personnes():
    """GET /api/personnes - Lister toutes les personnes"""
    try:
        # Paramètres de pagination optionnels
        page = request.args.get('page', 1, type=int)
        per_page = request.args.get('per_page', 10, type=int)
        per_page = min(per_page, 100)  # Limiter à 100 par page maximum
        # ⚠️ Toujours imposer une limite max sur per_page : sans ça, un client
        #    peut demander per_page=1000000 et faire tomber le serveur (OOM).

        # Paramètre de recherche optionnel
        search = request.args.get('search', '').strip()

        # Construire la requête
        query = Personne.query

        if search:
            query = query.filter(Personne.nom.ilike(f'%{search}%'))

        # Pagination
        personnes_paginated = query.paginate(
            page=page,
            per_page=per_page,
            error_out=False
        )

        # Formater la réponse
        return jsonify({
            'personnes': [personne.to_dict() for personne in personnes_paginated.items],
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': personnes_paginated.total,
                'pages': personnes_paginated.pages,
                'has_next': personnes_paginated.has_next,
                'has_prev': personnes_paginated.has_prev
            }
        })

    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/api/personnes/<int:person_id>', methods=['GET'])
def obtenir_personne(person_id):
    """GET /api/personnes/123 - Récupérer une personne spécifique"""
    try:
        # ⚠️ Personne.query.get(id) est déprécié en Flask-SQLAlchemy 3 / SQLAlchemy 2.0
        # → utilisez db.session.get(Personne, id) à la place
        personne = db.session.get(Personne, person_id)

        if not personne:
            return jsonify({'error': 'Personne non trouvée'}), 404

        return jsonify(personne.to_dict())

    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/api/personnes', methods=['POST'])
def creer_personne():
    """POST /api/personnes - Créer une nouvelle personne"""
    try:
        # Vérifier que c'est du JSON
        if not request.is_json:
            return jsonify({'error': 'Content-Type doit être application/json'}), 400

        data = request.get_json()

        # Vérifier les champs requis
        required_fields = ['nom', 'email']
        for field in required_fields:
            if field not in data or not data[field]:
                return jsonify({'error': f'Le champ {field} est requis'}), 400

        # Vérifier si l'email existe déjà
        if Personne.query.filter_by(email=data['email']).first():
            return jsonify({'error': 'Cet email est déjà utilisé'}), 409

        # Créer la nouvelle personne
        nouvelle_personne = Personne(
            nom=data['nom'].strip(),
            age=data.get('age'),
            email=data['email'].strip().lower()
        )

        # Sauvegarder en base
        db.session.add(nouvelle_personne)
        db.session.commit()

        return jsonify(nouvelle_personne.to_dict()), 201

    except ValueError as e:
        return jsonify({'error': str(e)}), 400
    except Exception as e:
        db.session.rollback()
        return jsonify({'error': str(e)}), 500

@app.route('/api/personnes/<int:person_id>', methods=['PUT'])
def mettre_a_jour_personne(person_id):
    """PUT /api/personnes/123 - Mettre à jour complètement une personne"""
    try:
        personne = db.session.get(Personne, person_id)

        if not personne:
            return jsonify({'error': 'Personne non trouvée'}), 404

        if not request.is_json:
            return jsonify({'error': 'Content-Type doit être application/json'}), 400

        data = request.get_json()

        # Vérifier les champs requis
        required_fields = ['nom', 'email']
        for field in required_fields:
            if field not in data or not data[field]:
                return jsonify({'error': f'Le champ {field} est requis'}), 400

        # Vérifier si l'email existe déjà (sauf pour cette personne)
        existing = Personne.query.filter_by(email=data['email']).first()
        if existing and existing.id != person_id:
            return jsonify({'error': 'Cet email est déjà utilisé'}), 409

        # Mettre à jour les champs
        personne.nom = data['nom'].strip()
        personne.age = data.get('age')
        personne.email = data['email'].strip().lower()

        # Valider et sauvegarder
        personne.valider()
        db.session.commit()

        return jsonify(personne.to_dict())

    except ValueError as e:
        return jsonify({'error': str(e)}), 400
    except Exception as e:
        db.session.rollback()
        return jsonify({'error': str(e)}), 500

@app.route('/api/personnes/<int:person_id>', methods=['PATCH'])
def modifier_personne(person_id):
    """PATCH /api/personnes/123 - Modifier partiellement une personne"""
    try:
        personne = db.session.get(Personne, person_id)

        if not personne:
            return jsonify({'error': 'Personne non trouvée'}), 404

        if not request.is_json:
            return jsonify({'error': 'Content-Type doit être application/json'}), 400

        data = request.get_json()

        if not data:
            return jsonify({'error': 'Aucune donnée à modifier'}), 400

        # Mettre à jour seulement les champs fournis
        if 'nom' in data:
            personne.nom = data['nom'].strip()

        if 'age' in data:
            personne.age = data['age']

        if 'email' in data:
            new_email = data['email'].strip().lower()
            # Vérifier l'unicité de l'email
            existing = Personne.query.filter_by(email=new_email).first()
            if existing and existing.id != person_id:
                return jsonify({'error': 'Cet email est déjà utilisé'}), 409
            personne.email = new_email

        # Valider et sauvegarder
        personne.valider()
        db.session.commit()

        return jsonify(personne.to_dict())

    except ValueError as e:
        return jsonify({'error': str(e)}), 400
    except Exception as e:
        db.session.rollback()
        return jsonify({'error': str(e)}), 500

@app.route('/api/personnes/<int:person_id>', methods=['DELETE'])
def supprimer_personne(person_id):
    """DELETE /api/personnes/123 - Supprimer une personne"""
    try:
        personne = db.session.get(Personne, person_id)

        if not personne:
            return jsonify({'error': 'Personne non trouvée'}), 404

        db.session.delete(personne)
        db.session.commit()

        return jsonify({'message': 'Personne supprimée avec succès'}), 200

    except Exception as e:
        db.session.rollback()
        return jsonify({'error': str(e)}), 500

@app.route('/api/stats', methods=['GET'])
def obtenir_statistiques():
    """GET /api/stats - Statistiques générales"""
    try:
        total_personnes = Personne.query.count()

        # Statistiques d'âge
        ages = db.session.query(Personne.age).filter(Personne.age.isnot(None)).all()
        ages_values = [age[0] for age in ages]

        stats = {
            'total_personnes': total_personnes,
            'age_stats': {
                'count': len(ages_values),
                'moyenne': round(sum(ages_values) / len(ages_values), 1) if ages_values else 0,
                'minimum': min(ages_values) if ages_values else 0,
                'maximum': max(ages_values) if ages_values else 0
            }
        }

        return jsonify(stats)

    except Exception as e:
        return jsonify({'error': str(e)}), 500

# ==================== GESTION D'ERREURS ====================

@app.errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Endpoint non trouvé'}), 404

@app.errorhandler(405)
def method_not_allowed(error):
    return jsonify({'error': 'Méthode HTTP non autorisée'}), 405

@app.errorhandler(500)
def internal_error(error):
    db.session.rollback()
    return jsonify({'error': 'Erreur interne du serveur'}), 500

# ==================== LANCEMENT DE L'APPLICATION ====================

if __name__ == '__main__':
    print("🚀 Démarrage de l'API...")
    print("📋 Endpoints disponibles :")
    print("   GET    /api/health")
    print("   GET    /api/personnes")
    print("   GET    /api/personnes/<id>")
    print("   POST   /api/personnes")
    print("   PUT    /api/personnes/<id>")
    print("   PATCH  /api/personnes/<id>")
    print("   DELETE /api/personnes/<id>")
    print("   GET    /api/stats")
    print("\n📡 API accessible sur : http://localhost:5000")

    # ⚠️ 3 dangers de cette ligne en production :
    #   1. debug=True : expose un debugger interactif (RCE possible si l'attaquant
    #      a la PIN — voir CVE Werkzeug Debugger). À DÉSACTIVER en prod.
    #   2. host='0.0.0.0' : écoute sur toutes les interfaces réseau ; en local,
    #      préférez '127.0.0.1'.
    #   3. app.run() lance le serveur de développement de Flask, jamais conçu
    #      pour la production. Utilisez gunicorn, uWSGI ou Waitress en prod :
    #         gunicorn -w 4 -b 0.0.0.0:5000 app:app
    app.run(debug=True, host='127.0.0.1', port=5000)
```

> ⚠️ **CORS et configuration sécurité** : `CORS(app)` sans argument autorise **toutes les origines** par défaut, ce qui est acceptable en développement mais **dangereux en production** — n'importe quel site malveillant pourrait appeler votre API depuis le navigateur d'un utilisateur authentifié. En production, restreignez explicitement :
> ```python
> CORS(app, origins=["https://monsite.com", "https://admin.monsite.com"])
> ```

### Test de l'API avec curl

```bash
# 1. Vérifier que l'API fonctionne
curl http://localhost:5000/api/health

# 2. Créer une nouvelle personne
curl -X POST http://localhost:5000/api/personnes \
  -H "Content-Type: application/json" \
  -d '{
    "nom": "Alice Martin",
    "age": 25,
    "email": "alice.martin@email.com"
  }'

# 3. Lister toutes les personnes
curl http://localhost:5000/api/personnes

# 4. Récupérer une personne spécifique
curl http://localhost:5000/api/personnes/1

# 5. Modifier une personne (PATCH)
curl -X PATCH http://localhost:5000/api/personnes/1 \
  -H "Content-Type: application/json" \
  -d '{"age": 26}'

# 6. Supprimer une personne
curl -X DELETE http://localhost:5000/api/personnes/1

# 7. Obtenir des statistiques
curl http://localhost:5000/api/stats
```

## API REST avec Node.js (Express)

Express.js est le framework web le plus populaire pour Node.js.

### Installation

```bash
npm init -y  
npm install express sqlite3 cors helmet morgan dotenv  
npm install --save-dev nodemon  
```

### Structure du projet

```
mon-api-node/
├── server.js           # Serveur principal
├── models/
│   └── personne.js     # Modèle Personne
├── routes/
│   └── personnes.js    # Routes des personnes
├── middleware/
│   └── validation.js   # Middleware de validation
├── config/
│   └── database.js     # Configuration base de données
├── package.json
└── .env                # Variables d'environnement
```

### Configuration

**.env**
```
PORT=3000  
DATABASE_PATH=./database.db  
NODE_ENV=development  
```

**config/database.js**
```javascript
const sqlite3 = require('sqlite3').verbose();  
const path = require('path');  

class Database {
    constructor() {
        this.db = null;
    }

    connect() {
        return new Promise((resolve, reject) => {
            const dbPath = process.env.DATABASE_PATH || './database.db';

            this.db = new sqlite3.Database(dbPath, (err) => {
                if (err) {
                    console.error('❌ Erreur connexion SQLite:', err.message);
                    reject(err);
                } else {
                    console.log('✅ Connecté à SQLite');
                    this.createTables().then(resolve).catch(reject);
                }
            });
        });
    }

    createTables() {
        return new Promise((resolve, reject) => {
            const createTableSQL = `
                CREATE TABLE IF NOT EXISTS personnes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    nom TEXT NOT NULL,
                    age INTEGER,
                    email TEXT UNIQUE NOT NULL,
                    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP
                )
            `;

            this.db.run(createTableSQL, (err) => {
                if (err) {
                    reject(err);
                } else {
                    console.log('📋 Table personnes créée ou existe déjà');
                    resolve();
                }
            });
        });
    }

    getDb() {
        return this.db;
    }

    close() {
        return new Promise((resolve) => {
            if (this.db) {
                this.db.close((err) => {
                    if (err) {
                        console.error('Erreur fermeture DB:', err.message);
                    } else {
                        console.log('🔒 Connexion SQLite fermée');
                    }
                    resolve();
                });
            } else {
                resolve();
            }
        });
    }
}

module.exports = new Database();
```

**models/personne.js**
```javascript
class Personne {
    constructor(db) {
        this.db = db;
    }

    // Validation des données
    valider(data) {
        const errors = [];

        if (!data.nom || data.nom.trim().length < 2) {
            errors.push('Le nom doit contenir au moins 2 caractères');
        }

        if (data.age !== undefined && (data.age < 0 || data.age > 150)) {
            errors.push('L\'âge doit être entre 0 et 150 ans');
        }

        const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
        if (!data.email || !emailRegex.test(data.email)) {
            errors.push('Format d\'email invalide');
        }

        return errors;
    }

    // Créer une nouvelle personne
    creer(data) {
        return new Promise((resolve, reject) => {
            const errors = this.valider(data);
            if (errors.length > 0) {
                reject(new Error(errors.join(', ')));
                return;
            }

            const sql = `
                INSERT INTO personnes (nom, age, email)
                VALUES (?, ?, ?)
            `;

            this.db.run(sql, [data.nom.trim(), data.age, data.email.toLowerCase()], function(err) {
                if (err) {
                    if (err.message.includes('UNIQUE constraint failed')) {
                        reject(new Error('Cet email est déjà utilisé'));
                    } else {
                        reject(err);
                    }
                } else {
                    resolve({
                        id: this.lastID,
                        nom: data.nom.trim(),
                        age: data.age,
                        email: data.email.toLowerCase()
                    });
                }
            });
        });
    }

    // Lister toutes les personnes avec pagination
    lister(options = {}) {
        return new Promise((resolve, reject) => {
            const page = parseInt(options.page) || 1;
            const limit = Math.min(parseInt(options.limit) || 10, 100);
            const offset = (page - 1) * limit;
            const search = options.search || '';

            let sql = `
                SELECT id, nom, age, email, date_creation
                FROM personnes
            `;
            let countSql = 'SELECT COUNT(*) as total FROM personnes';
            let params = [];

            if (search) {
                sql += ' WHERE nom LIKE ?';
                countSql += ' WHERE nom LIKE ?';
                params.push(`%${search}%`);
            }

            sql += ' ORDER BY nom LIMIT ? OFFSET ?';
            params.push(limit, offset);

            // Compter le total d'abord
            this.db.get(countSql, search ? [`%${search}%`] : [], (err, countResult) => {
                if (err) {
                    reject(err);
                    return;
                }

                const total = countResult.total;
                const totalPages = Math.ceil(total / limit);

                // Puis récupérer les données
                this.db.all(sql, params, (err, rows) => {
                    if (err) {
                        reject(err);
                    } else {
                        resolve({
                            personnes: rows,
                            pagination: {
                                page,
                                limit,
                                total,
                                totalPages,
                                hasNext: page < totalPages,
                                hasPrev: page > 1
                            }
                        });
                    }
                });
            });
        });
    }

    // Récupérer une personne par ID
    obtenirParId(id) {
        return new Promise((resolve, reject) => {
            const sql = `
                SELECT id, nom, age, email, date_creation
                FROM personnes
                WHERE id = ?
            `;

            this.db.get(sql, [id], (err, row) => {
                if (err) {
                    reject(err);
                } else {
                    resolve(row);
                }
            });
        });
    }

    // Mettre à jour une personne
    mettreAJour(id, data) {
        return new Promise((resolve, reject) => {
            const errors = this.valider(data);
            if (errors.length > 0) {
                reject(new Error(errors.join(', ')));
                return;
            }

            const sql = `
                UPDATE personnes
                SET nom = ?, age = ?, email = ?
                WHERE id = ?
            `;

            this.db.run(sql, [data.nom.trim(), data.age, data.email.toLowerCase(), id], function(err) {
                if (err) {
                    if (err.message.includes('UNIQUE constraint failed')) {
                        reject(new Error('Cet email est déjà utilisé'));
                    } else {
                        reject(err);
                    }
                } else if (this.changes === 0) {
                    reject(new Error('Personne non trouvée'));
                } else {
                    resolve({
                        id: parseInt(id),
                        nom: data.nom.trim(),
                        age: data.age,
                        email: data.email.toLowerCase()
                    });
                }
            });
        });
    }

    // Modifier partiellement une personne
    modifierPartiellement(id, data) {
        return new Promise((resolve, reject) => {
            // D'abord récupérer la personne existante
            this.obtenirParId(id).then(personne => {
                if (!personne) {
                    reject(new Error('Personne non trouvée'));
                    return;
                }

                // Fusionner les données
                const dataComplete = {
                    nom: data.nom !== undefined ? data.nom : personne.nom,
                    age: data.age !== undefined ? data.age : personne.age,
                    email: data.email !== undefined ? data.email : personne.email
                };

                // Utiliser la méthode de mise à jour complète
                this.mettreAJour(id, dataComplete).then(resolve).catch(reject);
            }).catch(reject);
        });
    }

    // Supprimer une personne
    supprimer(id) {
        return new Promise((resolve, reject) => {
            const sql = 'DELETE FROM personnes WHERE id = ?';

            this.db.run(sql, [id], function(err) {
                if (err) {
                    reject(err);
                } else if (this.changes === 0) {
                    reject(new Error('Personne non trouvée'));
                } else {
                    resolve({ message: 'Personne supprimée avec succès' });
                }
            });
        });
    }

    // Obtenir des statistiques
    obtenirStatistiques() {
        return new Promise((resolve, reject) => {
            const sql = `
                SELECT
                    COUNT(*) as total,
                    COUNT(age) as avec_age,
                    AVG(age) as age_moyen,
                    MIN(age) as age_min,
                    MAX(age) as age_max
                FROM personnes
            `;

            this.db.get(sql, [], (err, row) => {
                if (err) {
                    reject(err);
                } else {
                    resolve({
                        total_personnes: row.total,
                        age_stats: {
                            count: row.avec_age,
                            moyenne: row.age_moyen ? Math.round(row.age_moyen * 10) / 10 : 0,
                            minimum: row.age_min || 0,
                            maximum: row.age_max || 0
                        }
                    });
                }
            });
        });
    }
}

module.exports = Personne;
```

**middleware/validation.js**
```javascript
// Middleware pour valider les données JSON
const validateJson = (req, res, next) => {
    if (['POST', 'PUT', 'PATCH'].includes(req.method)) {
        if (!req.is('application/json')) {
            return res.status(400).json({
                error: 'Content-Type doit être application/json'
            });
        }

        if (!req.body || Object.keys(req.body).length === 0) {
            return res.status(400).json({
                error: 'Corps de la requête vide'
            });
        }
    }

    next();
};

// Middleware pour valider les paramètres d'ID
const validateId = (req, res, next) => {
    const id = parseInt(req.params.id);

    if (isNaN(id) || id <= 0) {
        return res.status(400).json({
            error: 'ID invalide'
        });
    }

    req.params.id = id;
    next();
};

// Middleware pour valider les paramètres de pagination
const validatePagination = (req, res, next) => {
    if (req.query.page) {
        const page = parseInt(req.query.page);
        if (isNaN(page) || page <= 0) {
            return res.status(400).json({
                error: 'Le paramètre page doit être un entier positif'
            });
        }
        req.query.page = page;
    }

    if (req.query.limit) {
        const limit = parseInt(req.query.limit);
        if (isNaN(limit) || limit <= 0 || limit > 100) {
            return res.status(400).json({
                error: 'Le paramètre limit doit être entre 1 et 100'
            });
        }
        req.query.limit = limit;
    }

    next();
};

module.exports = {
    validateJson,
    validateId,
    validatePagination
};
```

## routes/personnes.js

```javascript
const express = require('express');  
const Personne = require('../models/personne');  
const { validateJson, validateId, validatePagination } = require('../middleware/validation');  

const router = express.Router();

// Initialiser le modèle Personne avec la DB
let personneModel;

const initRouter = (db) => {
    personneModel = new Personne(db);
    return router;
};

// GET /api/personnes - Lister les personnes
router.get('/', validatePagination, async (req, res) => {
    try {
        const options = {
            page: req.query.page || 1,
            limit: req.query.limit || 10,
            search: req.query.search || ''
        };

        const result = await personneModel.lister(options);
        res.json(result);

    } catch (error) {
        console.error('Erreur liste personnes:', error);
        res.status(500).json({ error: error.message });
    }
});

// GET /api/personnes/:id - Récupérer une personne
router.get('/:id', validateId, async (req, res) => {
    try {
        const personne = await personneModel.obtenirParId(req.params.id);

        if (!personne) {
            return res.status(404).json({ error: 'Personne non trouvée' });
        }

        res.json(personne);

    } catch (error) {
        console.error('Erreur récupération personne:', error);
        res.status(500).json({ error: error.message });
    }
});

// POST /api/personnes - Créer une personne
router.post('/', validateJson, async (req, res) => {
    try {
        const nouvellePersonne = await personneModel.creer(req.body);
        res.status(201).json(nouvellePersonne);

    } catch (error) {
        console.error('Erreur création personne:', error);

        if (error.message.includes('email est déjà utilisé')) {
            res.status(409).json({ error: error.message });
        } else {
            res.status(400).json({ error: error.message });
        }
    }
});

// PUT /api/personnes/:id - Mettre à jour complètement
router.put('/:id', validateId, validateJson, async (req, res) => {
    try {
        // Vérifier les champs requis
        const requiredFields = ['nom', 'email'];
        for (const field of requiredFields) {
            if (!req.body[field]) {
                return res.status(400).json({
                    error: `Le champ ${field} est requis`
                });
            }
        }

        const personneModifiee = await personneModel.mettreAJour(req.params.id, req.body);
        res.json(personneModifiee);

    } catch (error) {
        console.error('Erreur mise à jour personne:', error);

        if (error.message === 'Personne non trouvée') {
            res.status(404).json({ error: error.message });
        } else if (error.message.includes('email est déjà utilisé')) {
            res.status(409).json({ error: error.message });
        } else {
            res.status(400).json({ error: error.message });
        }
    }
});

// PATCH /api/personnes/:id - Modifier partiellement
router.patch('/:id', validateId, validateJson, async (req, res) => {
    try {
        const personneModifiee = await personneModel.modifierPartiellement(req.params.id, req.body);
        res.json(personneModifiee);

    } catch (error) {
        console.error('Erreur modification personne:', error);

        if (error.message === 'Personne non trouvée') {
            res.status(404).json({ error: error.message });
        } else if (error.message.includes('email est déjà utilisé')) {
            res.status(409).json({ error: error.message });
        } else {
            res.status(400).json({ error: error.message });
        }
    }
});

// DELETE /api/personnes/:id - Supprimer une personne
router.delete('/:id', validateId, async (req, res) => {
    try {
        const result = await personneModel.supprimer(req.params.id);
        res.json(result);

    } catch (error) {
        console.error('Erreur suppression personne:', error);

        if (error.message === 'Personne non trouvée') {
            res.status(404).json({ error: error.message });
        } else {
            res.status(500).json({ error: error.message });
        }
    }
});

module.exports = initRouter;
```

## server.js

```javascript
require('dotenv').config();  
const express = require('express');  
const cors = require('cors');  
const helmet = require('helmet');  
const morgan = require('morgan');  
const database = require('./config/database');  
const personnesRoutes = require('./routes/personnes');  
const Personne = require('./models/personne');  

const app = express();  
const PORT = process.env.PORT || 3000;  

// ==================== MIDDLEWARE ====================

// Sécurité
app.use(helmet());

// CORS
app.use(cors({
    origin: process.env.NODE_ENV === 'production'
        ? ['https://monsite.com']
        : ['http://localhost:3000', 'http://localhost:8080'],
    credentials: true
}));

// Logging des requêtes
app.use(morgan(process.env.NODE_ENV === 'production' ? 'combined' : 'dev'));

// Parsing JSON
app.use(express.json({ limit: '10mb' }));  
app.use(express.urlencoded({ extended: true }));  

// ==================== ROUTES ====================

// Health check
app.get('/api/health', (req, res) => {
    res.json({
        status: 'OK',
        message: 'API fonctionnelle',
        version: '1.0.0',
        environment: process.env.NODE_ENV || 'development',
        timestamp: new Date().toISOString()
    });
});

// Routes des personnes (initialisées après connexion DB)
let personnesRouter;

// Statistiques
app.get('/api/stats', async (req, res) => {
    try {
        const personneModel = new Personne(database.getDb());
        const stats = await personneModel.obtenirStatistiques();
        res.json(stats);
    } catch (error) {
        console.error('Erreur statistiques:', error);
        res.status(500).json({ error: error.message });
    }
});

// ==================== GESTION D'ERREURS ====================

// Route non trouvée
app.use('*', (req, res) => {
    res.status(404).json({
        error: 'Endpoint non trouvé',
        requested: req.originalUrl,
        available_endpoints: [
            'GET /api/health',
            'GET /api/personnes',
            'GET /api/personnes/:id',
            'POST /api/personnes',
            'PUT /api/personnes/:id',
            'PATCH /api/personnes/:id',
            'DELETE /api/personnes/:id',
            'GET /api/stats'
        ]
    });
});

// Gestionnaire d'erreurs global
app.use((err, req, res, next) => {
    console.error('Erreur non gérée:', err);

    res.status(err.status || 500).json({
        error: process.env.NODE_ENV === 'production'
            ? 'Erreur interne du serveur'
            : err.message
    });
});

// ==================== DÉMARRAGE DU SERVEUR ====================

async function startServer() {
    try {
        // Connexion à la base de données
        await database.connect();

        // Initialiser les routes des personnes après connexion DB
        personnesRouter = personnesRoutes(database.getDb());
        app.use('/api/personnes', personnesRouter);

        // Démarrer le serveur
        const server = app.listen(PORT, () => {
            console.log('\n🚀 Serveur API démarré !');
            console.log(`📡 URL: http://localhost:${PORT}`);
            console.log(`🌍 Environnement: ${process.env.NODE_ENV || 'development'}`);
            console.log('\n📋 Endpoints disponibles :');
            console.log('   GET    /api/health');
            console.log('   GET    /api/personnes');
            console.log('   GET    /api/personnes/:id');
            console.log('   POST   /api/personnes');
            console.log('   PUT    /api/personnes/:id');
            console.log('   PATCH  /api/personnes/:id');
            console.log('   DELETE /api/personnes/:id');
            console.log('   GET    /api/stats');
            console.log('\n💡 Utilisez Ctrl+C pour arrêter le serveur');
        });

        // Gestion de l'arrêt propre
        process.on('SIGINT', async () => {
            console.log('\n🛑 Arrêt du serveur...');
            server.close(async () => {
                await database.close();
                console.log('✅ Serveur arrêté proprement');
                process.exit(0);
            });
        });

    } catch (error) {
        console.error('❌ Erreur démarrage serveur:', error);
        process.exit(1);
    }
}

// Démarrer l'application
startServer();

module.exports = app;
```

## Client web HTML/JavaScript

> ⚠️ **Note de sécurité — XSS via `innerHTML`** : l'exemple de client ci-dessous insère le nom et l'email directement dans `innerHTML` avec interpolation (`${personne.nom}`). Si un nom contient du HTML (`<script>alert(1)</script>`), il sera exécuté. **Pour un client de production**, utilisez `textContent` (sécurisé par défaut) ou échappez avec une fonction comme :  
>
> ```javascript
> const escapeHtml = (s) => String(s).replace(/[&<>"']/g, c => ({
>     '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;'
> }[c]));
> // puis : `<h3>${escapeHtml(personne.nom)}</h3>`
> ```
>  
> Ou utilisez un framework (React/Vue/Svelte) qui échappe automatiquement.

**public/index.html**
```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Client API Personnes</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f5f5f5;
        }

        .container {
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
            margin-bottom: 20px;
        }

        .form-group {
            margin-bottom: 15px;
        }

        label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }

        input, button {
            width: 100%;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            font-size: 16px;
        }

        button {
            background-color: #007bff;
            color: white;
            cursor: pointer;
            margin-top: 10px;
        }

        button:hover {
            background-color: #0056b3;
        }

        .personne-item {
            border: 1px solid #ddd;
            padding: 15px;
            margin-bottom: 10px;
            border-radius: 4px;
            background-color: #f9f9f9;
        }

        .actions {
            margin-top: 10px;
        }

        .btn-small {
            width: auto;
            margin-right: 10px;
            padding: 5px 15px;
        }

        .btn-danger {
            background-color: #dc3545;
        }

        .btn-danger:hover {
            background-color: #c82333;
        }

        .status {
            padding: 10px;
            margin: 10px 0;
            border-radius: 4px;
        }

        .success {
            background-color: #d4edda;
            color: #155724;
            border: 1px solid #c3e6cb;
        }

        .error {
            background-color: #f8d7da;
            color: #721c24;
            border: 1px solid #f5c6cb;
        }
    </style>
</head>
<body>
    <h1>🧑‍💼 Gestionnaire de Personnes</h1>

    <!-- Formulaire d'ajout -->
    <div class="container">
        <h2>➕ Ajouter une personne</h2>
        <form id="personneForm">
            <div class="form-group">
                <label for="nom">Nom :</label>
                <input type="text" id="nom" required>
            </div>

            <div class="form-group">
                <label for="age">Âge :</label>
                <input type="number" id="age" min="0" max="150">
            </div>

            <div class="form-group">
                <label for="email">Email :</label>
                <input type="email" id="email" required>
            </div>

            <button type="submit">Ajouter</button>
        </form>

        <div id="status"></div>
    </div>

    <!-- Statistiques -->
    <div class="container">
        <h2>📊 Statistiques</h2>
        <div id="stats">Chargement...</div>
        <button onclick="chargerStatistiques()">Actualiser</button>
    </div>

    <!-- Liste des personnes -->
    <div class="container">
        <h2>📋 Liste des personnes</h2>
        <div style="margin-bottom: 15px;">
            <input type="text" id="searchInput" placeholder="Rechercher par nom...">
            <button onclick="chargerPersonnes()">🔍 Rechercher</button>
        </div>
        <div id="personnesList">Chargement...</div>
        <button onclick="chargerPersonnes()">🔄 Actualiser la liste</button>
    </div>

    <script>
        const API_BASE = 'http://localhost:3000/api';

        // Utilitaire pour afficher les messages
        function afficherMessage(message, type = 'success') {
            const statusDiv = document.getElementById('status');
            statusDiv.innerHTML = `<div class="${type}">${message}</div>`;
            setTimeout(() => statusDiv.innerHTML = '', 5000);
        }

        // Charger les statistiques
        async function chargerStatistiques() {
            try {
                const response = await fetch(`${API_BASE}/stats`);
                const stats = await response.json();

                document.getElementById('stats').innerHTML = `
                    <p><strong>Total de personnes :</strong> ${stats.total_personnes}</p>
                    <p><strong>Âge moyen :</strong> ${stats.age_stats.moyenne} ans</p>
                    <p><strong>Âge minimum :</strong> ${stats.age_stats.minimum} ans</p>
                    <p><strong>Âge maximum :</strong> ${stats.age_stats.maximum} ans</p>
                `;
            } catch (error) {
                console.error('Erreur chargement stats:', error);
                document.getElementById('stats').innerHTML = '❌ Erreur de chargement';
            }
        }

        // Charger la liste des personnes
        async function chargerPersonnes() {
            try {
                const search = document.getElementById('searchInput').value;
                const url = search
                    ? `${API_BASE}/personnes?search=${encodeURIComponent(search)}`
                    : `${API_BASE}/personnes`;

                const response = await fetch(url);
                const data = await response.json();

                const listDiv = document.getElementById('personnesList');

                if (data.personnes.length === 0) {
                    listDiv.innerHTML = '<p>Aucune personne trouvée.</p>';
                    return;
                }

                listDiv.innerHTML = data.personnes.map(personne => `
                    <div class="personne-item">
                        <h3>${personne.nom}</h3>
                        <p><strong>Âge :</strong> ${personne.age || 'Non renseigné'}</p>
                        <p><strong>Email :</strong> ${personne.email}</p>
                        <p><strong>Créé le :</strong> ${new Date(personne.date_creation).toLocaleDateString('fr-FR')}</p>
                        <div class="actions">
                            <button class="btn-small" onclick="modifierPersonne(${personne.id})">✏️ Modifier</button>
                            <button class="btn-small btn-danger" onclick="supprimerPersonne(${personne.id}, '${personne.nom}')">🗑️ Supprimer</button>
                        </div>
                    </div>
                `).join('');

                // Afficher les infos de pagination
                if (data.pagination) {
                    listDiv.innerHTML += `
                        <p><em>Page ${data.pagination.page} sur ${data.pagination.totalPages}
                           (${data.pagination.total} personnes au total)</em></p>
                    `;
                }

            } catch (error) {
                console.error('Erreur chargement personnes:', error);
                document.getElementById('personnesList').innerHTML = '❌ Erreur de chargement';
            }
        }

        // Ajouter une personne
        document.getElementById('personneForm').addEventListener('submit', async (e) => {
            e.preventDefault();

            const data = {
                nom: document.getElementById('nom').value,
                age: parseInt(document.getElementById('age').value) || undefined,
                email: document.getElementById('email').value
            };

            try {
                const response = await fetch(`${API_BASE}/personnes`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(data)
                });

                const result = await response.json();

                if (response.ok) {
                    afficherMessage(`✅ ${result.nom} ajouté(e) avec succès !`);
                    document.getElementById('personneForm').reset();
                    chargerPersonnes();
                    chargerStatistiques();
                } else {
                    afficherMessage(`❌ Erreur : ${result.error}`, 'error');
                }

            } catch (error) {
                afficherMessage(`❌ Erreur réseau : ${error.message}`, 'error');
            }
        });

        // Modifier une personne (exemple simple)
        async function modifierPersonne(id) {
            const nouvelAge = prompt('Nouvel âge :');
            if (nouvelAge === null) return;

            try {
                const response = await fetch(`${API_BASE}/personnes/${id}`, {
                    method: 'PATCH',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ age: parseInt(nouvelAge) || undefined })
                });

                const result = await response.json();

                if (response.ok) {
                    afficherMessage(`✅ ${result.nom} modifié(e) avec succès !`);
                    chargerPersonnes();
                    chargerStatistiques();
                } else {
                    afficherMessage(`❌ Erreur : ${result.error}`, 'error');
                }

            } catch (error) {
                afficherMessage(`❌ Erreur réseau : ${error.message}`, 'error');
            }
        }

        // Supprimer une personne
        async function supprimerPersonne(id, nom) {
            if (!confirm(`Êtes-vous sûr de vouloir supprimer ${nom} ?`)) {
                return;
            }

            try {
                const response = await fetch(`${API_BASE}/personnes/${id}`, {
                    method: 'DELETE'
                });

                const result = await response.json();

                if (response.ok) {
                    afficherMessage(`✅ ${nom} supprimé(e) avec succès !`);
                    chargerPersonnes();
                    chargerStatistiques();
                } else {
                    afficherMessage(`❌ Erreur : ${result.error}`, 'error');
                }

            } catch (error) {
                afficherMessage(`❌ Erreur réseau : ${error.message}`, 'error');
            }
        }

        // Charger les données au démarrage
        window.addEventListener('DOMContentLoaded', () => {
            chargerPersonnes();
            chargerStatistiques();
        });
    </script>
</body>
</html>
```

## Tests automatisés

**tests/api.test.js**
```javascript
const request = require('supertest');  
const app = require('../server');  

describe('API Personnes', () => {
    let server;
    let personneId;

    beforeAll((done) => {
        server = app.listen(done);
    });

    afterAll((done) => {
        server.close(done);
    });

    describe('GET /api/health', () => {
        test('Devrait retourner le statut OK', async () => {
            const res = await request(app).get('/api/health');

            expect(res.statusCode).toBe(200);
            expect(res.body.status).toBe('OK');
        });
    });

    describe('POST /api/personnes', () => {
        test('Devrait créer une nouvelle personne', async () => {
            const nouvellePersonne = {
                nom: 'Test User',
                age: 30,
                email: 'test@example.com'
            };

            const res = await request(app)
                .post('/api/personnes')
                .send(nouvellePersonne)
                .expect(201);

            expect(res.body.nom).toBe(nouvellePersonne.nom);
            expect(res.body.id).toBeDefined();

            personneId = res.body.id; // Sauvegarder pour les autres tests
        });

        test('Devrait rejeter une personne avec email invalide', async () => {
            const personneInvalide = {
                nom: 'Test',
                email: 'email-invalide'
            };

            await request(app)
                .post('/api/personnes')
                .send(personneInvalide)
                .expect(400);
        });

        test('Devrait rejeter un email déjà utilisé', async () => {
            const personneDupliquee = {
                nom: 'Autre Test',
                email: 'test@example.com' // Même email que le premier test
            };

            await request(app)
                .post('/api/personnes')
                .send(personneDupliquee)
                .expect(409);
        });
    });

    describe('GET /api/personnes', () => {
        test('Devrait lister les personnes', async () => {
            const res = await request(app)
                .get('/api/personnes')
                .expect(200);

            expect(Array.isArray(res.body.personnes)).toBe(true);
            expect(res.body.pagination).toBeDefined();
        });

        test('Devrait supporter la recherche', async () => {
            const res = await request(app)
                .get('/api/personnes?search=Test')
                .expect(200);

            expect(res.body.personnes.length).toBeGreaterThan(0);
            expect(res.body.personnes[0].nom).toContain('Test');
        });
    });

    describe('GET /api/personnes/:id', () => {
        test('Devrait récupérer une personne spécifique', async () => {
            const res = await request(app)
                .get(`/api/personnes/${personneId}`)
                .expect(200);

            expect(res.body.id).toBe(personneId);
            expect(res.body.nom).toBe('Test User');
        });

        test('Devrait retourner 404 pour un ID inexistant', async () => {
            await request(app)
                .get('/api/personnes/99999')
                .expect(404);
        });
    });

    describe('PATCH /api/personnes/:id', () => {
        test('Devrait modifier partiellement une personne', async () => {
            const modifications = { age: 31 };

            const res = await request(app)
                .patch(`/api/personnes/${personneId}`)
                .send(modifications)
                .expect(200);

            expect(res.body.age).toBe(31);
            expect(res.body.nom).toBe('Test User'); // Inchangé
        });
    });

    describe('DELETE /api/personnes/:id', () => {
        test('Devrait supprimer une personne', async () => {
            await request(app)
                .delete(`/api/personnes/${personneId}`)
                .expect(200);

            // Vérifier que la personne n'existe plus
            await request(app)
                .get(`/api/personnes/${personneId}`)
                .expect(404);
        });
    });

    describe('GET /api/stats', () => {
        test('Devrait retourner les statistiques', async () => {
            const res = await request(app)
                .get('/api/stats')
                .expect(200);

            expect(res.body.total_personnes).toBeDefined();
            expect(res.body.age_stats).toBeDefined();
        });
    });
});
```

## Authentification JWT

**middleware/auth.js**
```javascript
const jwt = require('jsonwebtoken');

// ⚠️ JAMAIS de fallback en clair pour un secret JWT : si le code fuite (Git public,
//    bundle JS exposé, etc.) un attaquant peut forger des tokens admin valides.
//    En production : refusez de démarrer si JWT_SECRET n'est pas défini.
const JWT_SECRET = process.env.JWT_SECRET;  
if (!JWT_SECRET || JWT_SECRET.length < 32) {  
    throw new Error(
        "JWT_SECRET manquant ou trop court (< 32 caractères).\n" +
        "Générez : node -e \"console.log(require('crypto').randomBytes(64).toString('hex'))\""
    );
}

// Middleware d'authentification
const authenticateToken = (req, res, next) => {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN

    if (!token) {
        return res.status(401).json({ error: 'Token d\'accès requis' });
    }

    // ⚠️ Toujours vérifier explicitement l'algorithme attendu. Sans cela,
    //    un attaquant peut forger un token avec `alg: "none"` ou exploiter
    //    la faille "algorithm confusion" (HS256 ↔ RS256). Voir CVE-2015-9235.
    jwt.verify(token, JWT_SECRET, { algorithms: ['HS256'] }, (err, user) => {
        if (err) {
            return res.status(403).json({ error: 'Token invalide' });
        }

        req.user = user;
        next();
    });
};

// Générer un token JWT
const generateToken = (user) => {
    return jwt.sign(
        {
            sub: String(user.id),       // "subject" (recommandé par RFC 7519)
            email: user.email,
            // ⚠️ N'embarquez JAMAIS de données sensibles (mot de passe, secrets).
            //    Le JWT est SIGNÉ mais PAS chiffré — son payload est lisible par
            //    n'importe qui via jwt.io ou base64-decode.
        },
        JWT_SECRET,
        {
            algorithm: 'HS256',
            expiresIn: '15m',           // ⚠️ 15 min > 24h : un token compromis
                                        //    expire vite. Utilisez un refresh
                                        //    token séparé (longue durée, stocké
                                        //    en BDD, révocable) pour renouveler.
            issuer: 'mon-api',
            audience: 'mon-frontend'
        }
    );
};

module.exports = {
    authenticateToken,
    generateToken
};
```

> 🔒 **Bonnes pratiques JWT 2026** :  
> - **Algorithme explicite** : `algorithms: ['HS256']` à la vérification, sinon faille « algorithm confusion ».  
> - **Secret ≥ 256 bits** (`crypto.randomBytes(32)` minimum, idéalement 64) — un secret faible se brute-force hors-ligne.  
> - **Durée courte** (15 min – 1 h) + **refresh token séparé** stocké en BDD (révocable).  
> - **Pas de données sensibles** dans le payload : il est seulement *signé*, pas *chiffré* (utilisez JWE si vous voulez du chiffrement).  
> - **Stockage côté client** : `HttpOnly` + `Secure` cookies > `localStorage` (qui est vulnérable au XSS).

**routes/auth.js**

> ⚠️ **AVERTISSEMENT** : l'exemple « simplifié » ci-dessous (comparaison en clair `password === 'password123'`) est **uniquement pédagogique** pour montrer le flux JWT. **En production**, vous DEVEZ : (1) stocker les utilisateurs dans une table dédiée avec mot de passe **hashé via bcrypt/argon2**, (2) utiliser une fonction de comparaison à **temps constant** comme `bcrypt.compare()` (qui résiste aux timing attacks), (3) limiter les tentatives via rate-limiting. La version sécurisée suit l'exemple simplifié.

**Version pédagogique (ne pas utiliser telle quelle en production)** :

```javascript
const express = require('express');  
const { generateToken } = require('../middleware/auth');  

const router = express.Router();

// POST /api/auth/login — VERSION SIMPLIFIÉE PÉDAGOGIQUE
router.post('/login', async (req, res) => {
    try {
        const { email, password } = req.body;

        // ⚠️ DANGER : comparaison en clair, identifiants codés en dur.
        //    EN PRODUCTION → voir la version sécurisée ci-dessous.
        if (email === 'admin@example.com' && password === 'password123') {
            const user = { id: 1, email: 'admin@example.com' };
            const token = generateToken(user);

            res.json({
                message: 'Connexion réussie',
                token,
                user: { id: user.id, email: user.email }
            });
        } else {
            res.status(401).json({ error: 'Identifiants invalides' });
        }

    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

module.exports = router;
```

**Version sécurisée avec table utilisateurs et bcrypt** :

```javascript
const express = require('express');  
const bcrypt = require('bcrypt');  
const { generateToken } = require('../middleware/auth');  

const router = express.Router();

// Récupération de l'utilisateur depuis la BDD (à adapter selon votre couche d'accès)
async function findUserByEmail(db, email) {
    return new Promise((resolve, reject) => {
        db.get(
            'SELECT id, email, password_hash FROM utilisateurs WHERE email = ?',
            [email.toLowerCase()],
            (err, row) => err ? reject(err) : resolve(row)
        );
    });
}

router.post('/login', async (req, res) => {
    try {
        const { email, password } = req.body;

        if (!email || !password) {
            return res.status(400).json({ error: 'Email et mot de passe requis' });
        }

        const user = await findUserByEmail(req.db, email);

        // ⚠️ TIMING ATTACK : si l'utilisateur n'existe pas, on doit quand même
        //    exécuter bcrypt.compare() avec un hash factice pour que la durée
        //    de réponse soit identique dans les deux cas. Sinon un attaquant
        //    peut deviner les emails valides en chronométrant les réponses.
        const hashToCompare = user
            ? user.password_hash
            : '$2b$12$invalidhashinvalidhashinvalidhashinvalidhashinvalid';

        const valid = await bcrypt.compare(password, hashToCompare);

        if (!valid || !user) {
            return res.status(401).json({ error: 'Identifiants invalides' });
        }

        const token = generateToken({ id: user.id, email: user.email });
        res.json({
            message: 'Connexion réussie',
            token,
            user: { id: user.id, email: user.email }
        });
    } catch (error) {
        console.error('Erreur login:', error);
        res.status(500).json({ error: 'Erreur serveur' });
    }
});

// POST /api/auth/register — création d'un compte
router.post('/register', async (req, res) => {
    try {
        const { email, password } = req.body;

        // Validation basique
        if (!email || !password || password.length < 8) {
            return res.status(400).json({
                error: 'Email requis, mot de passe ≥ 8 caractères'
            });
        }

        // bcrypt avec cost factor 12 (recommandé 2024+ ; 10 est trop faible)
        const password_hash = await bcrypt.hash(password, 12);

        // Insertion (gérer l'erreur de doublon UNIQUE)
        await new Promise((resolve, reject) => {
            req.db.run(
                'INSERT INTO utilisateurs (email, password_hash) VALUES (?, ?)',
                [email.toLowerCase(), password_hash],
                function (err) {
                    if (err) return reject(err);
                    resolve(this.lastID);
                }
            );
        });

        res.status(201).json({ message: 'Compte créé' });
    } catch (error) {
        if (error.message.includes('UNIQUE')) {
            return res.status(409).json({ error: 'Cet email est déjà utilisé' });
        }
        res.status(500).json({ error: 'Erreur serveur' });
    }
});

module.exports = router;
```

> 🔒 **Trois points clés** :  
> - **bcrypt cost ≥ 12** en 2026 (réévaluer périodiquement — voir [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html))  
> - **Réponses à durée constante** : même temps de réponse que l'utilisateur existe ou non  
> - **Limitation du débit** sur `/login` (cf. `rateLimit` plus loin) pour ralentir le brute-force

## Monitoring et sécurité

### Rate limiting

```javascript
const rateLimit = require('express-rate-limit');

// Limite générale
const generalLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // 100 requêtes par fenêtre
    message: {
        error: 'Trop de requêtes, réessayez plus tard'
    },
    standardHeaders: true,
    legacyHeaders: false
});

// Limite pour la création
const createLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 10, // 10 créations par fenêtre
    message: {
        error: 'Trop de créations, réessayez plus tard'
    }
});

// Appliquer les limites
app.use('/api/', generalLimiter);

// ⚠️ `app.use(path, methods, mw)` n'existe pas en Express — `methods` n'est pas
// un argument valide. Pour limiter uniquement les POST, soit on l'attache
// directement à la route, soit on filtre dans un middleware :

// Option 1 : attaché à la route (le plus propre)
app.post('/api/personnes', createLimiter, /* handler */);

// Option 2 : filtre conditionnel
app.use('/api/personnes', (req, res, next) => {
    if (req.method === 'POST') return createLimiter(req, res, next);
    next();
});
```

### Monitoring des métriques

```javascript
// Middleware de monitoring simple
let requestCount = 0;  
let errorCount = 0;  
const startTime = Date.now();  

const metrics = {
    requests: () => requestCount,
    errors: () => errorCount,
    uptime: () => Math.floor((Date.now() - startTime) / 1000),
    memory: () => process.memoryUsage()
};

app.use((req, res, next) => {
    requestCount++;
    next();
});

app.use((err, req, res, next) => {
    errorCount++;
    next(err);
});

// Endpoint de métriques
app.get('/api/metrics', (req, res) => {
    res.json({
        requests_total: metrics.requests(),
        errors_total: metrics.errors(),
        uptime_seconds: metrics.uptime(),
        memory_usage: metrics.memory(),
        timestamp: new Date().toISOString()
    });
});
```

## Health checks détaillés

```javascript
const healthChecks = {
    database: async () => {
        try {
            const db = database.getDb();
            return new Promise((resolve, reject) => {
                db.get('SELECT 1', (err) => {
                    if (err) reject(err);
                    else resolve(true);
                });
            });
        } catch (error) {
            throw new Error('Database connection failed');
        }
    },

    memory: () => {
        const usage = process.memoryUsage();
        const limit = 512 * 1024 * 1024; // 512MB
        if (usage.rss > limit) {
            throw new Error('Memory usage too high');
        }
        return { usage: Math.round(usage.rss / 1024 / 1024) + 'MB' };
    },

    disk: () => {
        const fs = require('fs');
        try {
            const stats = fs.statSync('./');
            return { available: true };
        } catch (error) {
            throw new Error('Disk access failed');
        }
    },

    api: async () => {
        // Tester une requête simple
        try {
            const personneModel = new Personne(database.getDb());
            await personneModel.obtenirStatistiques();
            return { api_responsive: true };
        } catch (error) {
            throw new Error('API not responding');
        }
    }
};

app.get('/api/health/detailed', async (req, res) => {
    const results = {};
    let overallStatus = 'healthy';

    for (const [name, check] of Object.entries(healthChecks)) {
        try {
            results[name] = {
                status: 'healthy',
                details: await check()
            };
        } catch (error) {
            results[name] = {
                status: 'unhealthy',
                error: error.message
            };
            overallStatus = 'unhealthy';
        }
    }

    const statusCode = overallStatus === 'healthy' ? 200 : 503;
    res.status(statusCode).json({
        status: overallStatus,
        timestamp: new Date().toISOString(),
        checks: results,
        uptime: process.uptime(),
        version: process.env.npm_package_version || '1.0.0'
    });
});
```

## Documentation Swagger automatique

### Installation et configuration

```bash
npm install swagger-jsdoc swagger-ui-express
```

**swagger.js**
```javascript
const swaggerJsdoc = require('swagger-jsdoc');  
const swaggerUi = require('swagger-ui-express');  

const options = {
    definition: {
        openapi: '3.0.0',
        info: {
            title: 'API Personnes',
            version: '1.0.0',
            description: 'API REST pour gérer des personnes avec SQLite',
            contact: {
                name: 'Support API',
                email: 'support@example.com'
            }
        },
        servers: [
            {
                url: process.env.NODE_ENV === 'production'
                    ? 'https://api.monsite.com'
                    : 'http://localhost:3000',
                description: process.env.NODE_ENV === 'production'
                    ? 'Serveur de production'
                    : 'Serveur de développement',
            },
        ],
        components: {
            schemas: {
                Personne: {
                    type: 'object',
                    required: ['nom', 'email'],
                    properties: {
                        id: {
                            type: 'integer',
                            description: 'ID unique de la personne',
                            example: 1
                        },
                        nom: {
                            type: 'string',
                            minLength: 2,
                            maxLength: 100,
                            description: 'Nom de la personne',
                            example: 'Alice Martin'
                        },
                        age: {
                            type: 'integer',
                            minimum: 0,
                            maximum: 150,
                            description: 'Âge de la personne',
                            example: 25
                        },
                        email: {
                            type: 'string',
                            format: 'email',
                            description: 'Adresse email unique',
                            example: 'alice.martin@email.com'
                        },
                        date_creation: {
                            type: 'string',
                            format: 'date-time',
                            description: 'Date de création',
                            example: '2024-01-15T10:30:00Z'
                        },
                    },
                },
                PersonneInput: {
                    type: 'object',
                    required: ['nom', 'email'],
                    properties: {
                        nom: {
                            type: 'string',
                            minLength: 2,
                            maxLength: 100,
                            example: 'Alice Martin'
                        },
                        age: {
                            type: 'integer',
                            minimum: 0,
                            maximum: 150,
                            example: 25
                        },
                        email: {
                            type: 'string',
                            format: 'email',
                            example: 'alice.martin@email.com'
                        }
                    },
                },
                Error: {
                    type: 'object',
                    properties: {
                        error: {
                            type: 'string',
                            description: 'Message d\'erreur',
                            example: 'Personne non trouvée'
                        },
                    },
                },
                PaginatedPersonnes: {
                    type: 'object',
                    properties: {
                        personnes: {
                            type: 'array',
                            items: {
                                $ref: '#/components/schemas/Personne'
                            }
                        },
                        pagination: {
                            type: 'object',
                            properties: {
                                page: { type: 'integer', example: 1 },
                                limit: { type: 'integer', example: 10 },
                                total: { type: 'integer', example: 50 },
                                totalPages: { type: 'integer', example: 5 },
                                hasNext: { type: 'boolean', example: true },
                                hasPrev: { type: 'boolean', example: false }
                            }
                        }
                    }
                },
                Stats: {
                    type: 'object',
                    properties: {
                        total_personnes: { type: 'integer', example: 50 },
                        age_stats: {
                            type: 'object',
                            properties: {
                                count: { type: 'integer', example: 45 },
                                moyenne: { type: 'number', example: 32.5 },
                                minimum: { type: 'integer', example: 18 },
                                maximum: { type: 'integer', example: 65 }
                            }
                        }
                    }
                }
            },
            securitySchemes: {
                bearerAuth: {
                    type: 'http',
                    scheme: 'bearer',
                    bearerFormat: 'JWT'
                }
            }
        },
    },
    apis: ['./routes/*.js'], // Chemin vers les fichiers contenant les annotations
};

const specs = swaggerJsdoc(options);

module.exports = { specs, swaggerUi };
```

### Annotations dans les routes

**Ajouter dans routes/personnes.js :**

```javascript
/**
 * @swagger
 * tags:
 *   name: Personnes
 *   description: Gestion des personnes
 */

/**
 * @swagger
 * /api/personnes:
 *   get:
 *     summary: Lister les personnes
 *     tags: [Personnes]
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           minimum: 1
 *           default: 1
 *         description: Numéro de page
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           minimum: 1
 *           maximum: 100
 *           default: 10
 *         description: Nombre d'éléments par page
 *       - in: query
 *         name: search
 *         schema:
 *           type: string
 *         description: Terme de recherche dans le nom
 *     responses:
 *       200:
 *         description: Liste des personnes avec pagination
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/PaginatedPersonnes'
 *       500:
 *         description: Erreur serveur
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */

/**
 * @swagger
 * /api/personnes:
 *   post:
 *     summary: Créer une nouvelle personne
 *     tags: [Personnes]
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/PersonneInput'
 *     responses:
 *       201:
 *         description: Personne créée avec succès
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Personne'
 *       400:
 *         description: Données invalides
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       409:
 *         description: Email déjà utilisé
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */

/**
 * @swagger
 * /api/personnes/{id}:
 *   get:
 *     summary: Récupérer une personne par ID
 *     tags: [Personnes]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID de la personne
 *     responses:
 *       200:
 *         description: Personne trouvée
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Personne'
 *       404:
 *         description: Personne non trouvée
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */

/**
 * @swagger
 * /api/personnes/{id}:
 *   patch:
 *     summary: Modifier partiellement une personne
 *     tags: [Personnes]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID de la personne
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             properties:
 *               nom:
 *                 type: string
 *                 minLength: 2
 *                 maxLength: 100
 *               age:
 *                 type: integer
 *                 minimum: 0
 *                 maximum: 150
 *               email:
 *                 type: string
 *                 format: email
 *     responses:
 *       200:
 *         description: Personne modifiée avec succès
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Personne'
 *       404:
 *         description: Personne non trouvée
 *       400:
 *         description: Données invalides
 */

/**
 * @swagger
 * /api/personnes/{id}:
 *   delete:
 *     summary: Supprimer une personne
 *     tags: [Personnes]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *         description: ID de la personne
 *     responses:
 *       200:
 *         description: Personne supprimée avec succès
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 message:
 *                   type: string
 *                   example: "Personne supprimée avec succès"
 *       404:
 *         description: Personne non trouvée
 */

/**
 * @swagger
 * /api/stats:
 *   get:
 *     summary: Obtenir les statistiques
 *     tags: [Stats]
 *     responses:
 *       200:
 *         description: Statistiques des personnes
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Stats'
 */
```

### Intégrer Swagger dans server.js

```javascript
// Dans server.js - ajouter après les autres imports
const { specs, swaggerUi } = require('./swagger');

// Documentation Swagger
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(specs, {
    explorer: true,
    customCss: `
        .swagger-ui .topbar { display: none }
        .swagger-ui .info { margin: 20px 0 }
        .swagger-ui .scheme-container { background: #f7f7f7; padding: 10px }
    `,
    customSiteTitle: 'API Personnes - Documentation',
    swaggerOptions: {
        persistAuthorization: true,
        docExpansion: 'none',
        filter: true,
        showRequestHeaders: true
    }
}));

// Ajouter dans les logs de démarrage
console.log('📚 Documentation disponible sur : http://localhost:3000/api-docs');
```

## Validation avancée avec Joi

**middleware/validation-joi.js**
```javascript
const Joi = require('joi');

// Schémas de validation
const schemas = {
    personne: Joi.object({
        nom: Joi.string().min(2).max(100).trim().required()
            .messages({
                'string.min': 'Le nom doit contenir au moins 2 caractères',
                'string.max': 'Le nom ne peut pas dépasser 100 caractères',
                'any.required': 'Le nom est requis'
            }),
        age: Joi.number().integer().min(0).max(150).optional()
            .messages({
                'number.min': 'L\'âge doit être positif',
                'number.max': 'L\'âge ne peut pas dépasser 150 ans',
                'number.integer': 'L\'âge doit être un nombre entier'
            }),
        email: Joi.string().email().lowercase().required()
            .messages({
                'string.email': 'Format d\'email invalide',
                'any.required': 'L\'email est requis'
            })
    }),

    personnePartielle: Joi.object({
        nom: Joi.string().min(2).max(100).trim().optional(),
        age: Joi.number().integer().min(0).max(150).optional(),
        email: Joi.string().email().lowercase().optional()
    }).min(1).messages({
        'object.min': 'Au moins un champ doit être fourni pour la modification'
    }),

    pagination: Joi.object({
        page: Joi.number().integer().min(1).optional(),
        limit: Joi.number().integer().min(1).max(100).optional(),
        search: Joi.string().max(100).optional()
    })
};

// Middleware de validation
const validate = (schema) => {
    return (req, res, next) => {
        const { error, value } = schema.validate(req.body, {
            abortEarly: false, // Retourner toutes les erreurs
            stripUnknown: true // Supprimer les champs non définis
        });

        if (error) {
            const errors = error.details.map(detail => ({
                field: detail.path.join('.'),
                message: detail.message
            }));

            return res.status(400).json({
                error: 'Données invalides',
                details: errors
            });
        }

        req.body = value; // Utiliser les données validées et nettoyées
        next();
    };
};

// Validation des paramètres de requête
const validateQuery = (schema) => {
    return (req, res, next) => {
        const { error, value } = schema.validate(req.query, {
            abortEarly: false,
            stripUnknown: true
        });

        if (error) {
            const errors = error.details.map(detail => ({
                field: detail.path.join('.'),
                message: detail.message
            }));

            return res.status(400).json({
                error: 'Paramètres invalides',
                details: errors
            });
        }

        req.query = value;
        next();
    };
};

module.exports = {
    schemas,
    validate,
    validateQuery
};
```

## Déploiement et production

### Configuration pour la production

**.env.production**
```bash
NODE_ENV=production  
PORT=8000  
DATABASE_PATH=/var/lib/app/database.db  
JWT_SECRET=your-super-secure-jwt-secret-change-me  
CORS_ORIGIN=https://yourapp.com,https://admin.yourapp.com  
LOG_LEVEL=info  
RATE_LIMIT_WINDOW_MS=900000  
RATE_LIMIT_MAX=1000  
```

### Dockerfile

```dockerfile
# Node 18 a atteint sa fin de vie en avril 2025 — utilisez 20 LTS ou 22 LTS
FROM node:22-alpine

# Créer un utilisateur non-root
RUN addgroup -g 1001 -S nodejs  
RUN adduser -S nodejs -u 1001  

# Définir le répertoire de travail
WORKDIR /app

# Copier les fichiers de dépendances
COPY package*.json ./

# Installer les dépendances (--omit=dev remplace --only=production déprécié en npm 9+)
RUN npm ci --omit=dev && npm cache clean --force

# Copier le code source
COPY --chown=nodejs:nodejs . .

# Créer les dossiers nécessaires
RUN mkdir -p /app/data /app/logs && chown -R nodejs:nodejs /app

# Changer vers l'utilisateur non-root
USER nodejs

# Exposer le port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node healthcheck.js || exit 1

# Commande de démarrage
CMD ["node", "server.js"]
```

### Script de health check

**healthcheck.js**
```javascript
const http = require('http');

const options = {
    hostname: 'localhost',
    port: process.env.PORT || 3000,
    path: '/api/health',
    method: 'GET',
    timeout: 2000
};

const request = http.request(options, (res) => {
    if (res.statusCode === 200) {
        process.exit(0);
    } else {
        console.error(`Health check failed with status ${res.statusCode}`);
        process.exit(1);
    }
});

request.on('error', (err) => {
    console.error('Health check failed:', err.message);
    process.exit(1);
});

request.on('timeout', () => {
    console.error('Health check timeout');
    request.destroy();
    process.exit(1);
});

request.end();
```

### Docker Compose pour production

**docker-compose.prod.yml**
```yaml
# ℹ️ Depuis Docker Compose v2 (2023), la clé `version:` est dépréciée et ignorée.
#    Docker Compose v1 (Python) atteint sa fin de vie en juin 2023.
#    Vous pouvez retirer cette ligne sur Docker Engine récent.
version: '3.8'  # gardé pour compatibilité avec d'anciens runners CI

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_PATH=/app/data/database.db
      - JWT_SECRET=${JWT_SECRET}
      - CORS_ORIGIN=${CORS_ORIGIN}
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - ./static:/var/www/html:ro
    depends_on:
      - api
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  # Optionnel : Monitoring avec Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    restart: unless-stopped

volumes:
  prometheus_data:
```

### Configuration Nginx

**nginx.conf**
```nginx
events {
    worker_connections 1024;
}

http {
    upstream api {
        server api:3000;
    }

    # Limite de débit
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    # Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # Headers de sécurité
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    # ⚠️ X-XSS-Protection est DÉCONSEILLÉ par OWASP en 2024+ : le filtre XSS
    #    des navigateurs (basé sur ce header) a été retiré de Chrome (≥ 78) et
    #    Edge (≥ 79), et il peut introduire des vulnérabilités dans les
    #    navigateurs qui l'implémentent encore. Préférez une Content-Security-Policy
    #    stricte (frame-ancestors, default-src 'self', etc.).
    # add_header X-XSS-Protection "1; mode=block" always;  # OBSOLÈTE
    add_header Content-Security-Policy "default-src 'self'; frame-ancestors 'none'; base-uri 'none'" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    server {
        listen 80;
        server_name localhost;

        # Redirection vers HTTPS en production
        # return 301 https://$server_name$request_uri;

        # Health check pour le load balancer
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # API
        location /api/ {
            limit_req zone=api burst=20 nodelay;

            proxy_pass http://api;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;

            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # Documentation Swagger
        location /api-docs {
            proxy_pass http://api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Servir les fichiers statiques
        location / {
            root /var/www/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }

        # Logs
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
    }
}
```

### Scripts de déploiement

**scripts/deploy.sh**
```bash
#!/bin/bash

set -e

echo "🚀 Déploiement de l'API..."

# Variables
IMAGE_NAME="api-personnes"  
CONTAINER_NAME="api-personnes-prod"  
BACKUP_DIR="/backups"  
DATE=$(date +%Y%m%d_%H%M%S)  

# Créer le dossier de sauvegarde
mkdir -p $BACKUP_DIR

# Sauvegarder la base de données actuelle
if [ -f "./data/database.db" ]; then
    echo "💾 Sauvegarde de la base de données..."
    # ⚠️ NE PAS utiliser `cp` sur une base SQLite en cours d'utilisation :
    #    si une transaction est en cours, vous obtenez un fichier corrompu
    #    (cf. chapitre 6.4 sur la Backup API).
    #    Utilisez la commande `.backup` du shell SQLite qui copie de façon
    #    transactionnellement sûre, même base ouverte en écriture :
    sqlite3 ./data/database.db ".backup '$BACKUP_DIR/database_backup_$DATE.db'"
    # Alternative : si vous tournez avec Litestream, vos backups sont déjà
    # continus dans S3 — pas besoin de snapshot manuel avant déploiement.
fi

# Construire la nouvelle image
echo "🔨 Construction de l'image Docker..."  
docker build -t $IMAGE_NAME:latest .  

# Arrêter l'ancien conteneur
echo "🛑 Arrêt de l'ancien conteneur..."  
docker stop $CONTAINER_NAME || true  
docker rm $CONTAINER_NAME || true  

# Démarrer le nouveau conteneur
echo "🏃 Démarrage du nouveau conteneur..."  
docker run -d \  
    --name $CONTAINER_NAME \
    --restart unless-stopped \
    -p 3000:3000 \
    -v $(pwd)/data:/app/data \
    -v $(pwd)/logs:/app/logs \
    --env-file .env.production \
    $IMAGE_NAME:latest

# Vérifier que le conteneur démarre correctement
echo "✅ Vérification du démarrage..."  
sleep 10  

if docker ps | grep -q $CONTAINER_NAME; then
    echo "✅ Déploiement réussi !"

    # Test de santé
    if curl -f http://localhost:3000/api/health; then
        echo "✅ API opérationnelle"
    else
        echo "❌ API ne répond pas"
        exit 1
    fi
else
    echo "❌ Échec du déploiement"
    exit 1
fi

# Nettoyer les anciennes images
echo "🧹 Nettoyage..."  
docker image prune -f  

echo "🎉 Déploiement terminé avec succès !"
```

### Monitoring avec PM2 (alternative à Docker)

> ⚠️ **PIÈGE MAJEUR — Cluster mode + SQLite ne font pas bon ménage** : la configuration historique de PM2 propose `instances: 'max'` + `exec_mode: 'cluster'` pour scaler sur tous les CPUs. Mais avec **SQLite, plusieurs processus écrivant sur le même fichier `.db` provoquent** :  
> - des erreurs **"SQLITE_BUSY: database is locked"** dès qu'il y a de la concurrence,  
> - et dans le pire des cas, **corruption de la base** si un processus crashe avec un fichier WAL/SHM dans un état inconsistant.  
>  
> **Règle d'or** : avec SQLite, **un seul processus écrit** à la fois sur la base. Pour scaler, vous avez deux options :  
> 1. **Mode `fork` (un seul process)** : `instances: 1` + worker threads internes au process si besoin de parallélisme CPU sur des calculs.  
> 2. **Architecture multi-process + base distribuée** : passez à Postgres/MySQL, ou utilisez **Turso/LibSQL** qui résout ce problème avec un serveur réseau frontal.

**ecosystem.config.js (corrigé pour SQLite)**
```javascript
module.exports = {
    apps: [{
        name: 'api-personnes',
        script: 'server.js',

        // ⚠️ Un seul process en mode fork (PAS de cluster avec SQLite !)
        instances: 1,
        exec_mode: 'fork',

        env: {
            NODE_ENV: 'development',
            PORT: 3000
        },
        env_production: {
            NODE_ENV: 'production',
            PORT: 8000,
            DATABASE_PATH: '/var/lib/app/database.db'
        },
        // Redémarrer si l'utilisation mémoire dépasse 500MB
        max_memory_restart: '500M',

        // Redémarrer en cas de crash
        autorestart: true,

        // Délai avant redémarrage en cas de crash
        restart_delay: 5000,

        // Logging
        log_file: '/var/log/app/combined.log',
        out_file: '/var/log/app/out.log',
        error_file: '/var/log/app/error.log',
        log_date_format: 'YYYY-MM-DD HH:mm:ss Z',

        // Monitoring
        monitoring: false
    }]
};
```

> 💡 **« Mais comment je scale alors ? »** Si un seul process Node.js n'est pas assez :  
> - **Mettez un reverse proxy devant** (Nginx, Caddy, HAProxy) pour gérer TLS et load-balancer **uniquement les lectures** vers des replicas créés via Litestream/LiteFS.  
> - **Confiez les calculs CPU-intensifs aux Worker threads** Node.js (qui partagent le process).  
> - **Découpez l'app en microservices** chacun avec sa propre base SQLite (sharding par domaine).  
> - **Ou migrez la base** vers Postgres si la limite « un writer » devient bloquante.

**Commandes PM2 utiles :**
```bash
# Démarrer l'application
pm2 start ecosystem.config.js --env production

# Voir le statut
pm2 status

# Logs en temps réel
pm2 logs

# Redémarrer
pm2 restart api-personnes

# Rechargement sans interruption (zero-downtime)
pm2 reload api-personnes

# Monitoring
pm2 monit

# Sauvegarder la configuration
pm2 save

# Auto-start au boot
pm2 startup
```

## Sécurité avancée

### Chiffrement des données sensibles

**utils/encryption.js**

> ⚠️ **Important — bugs corrigés par rapport à la version originelle de ce cours** :  
> 1. `crypto.createCipher`/`createDecipher` ont été **supprimées de Node.js 22** (et étaient dépréciées depuis Node 10). De plus, elles **n'utilisaient pas l'IV** — c'était cryptographiquement faux. Il faut utiliser `createCipheriv`/`createDecipheriv`.  
> 2. `pbkdf2Sync` avec `10 000` itérations est **bien trop faible** en 2026. OWASP recommande au minimum **600 000 itérations** pour PBKDF2-HMAC-SHA256, ou **210 000** pour SHA-512. Mieux encore : utilisez `argon2` ou `bcrypt`.  
> 3. `crypto.randomBytes(32)` comme fallback de clé : si la variable d'environnement manque, une **nouvelle clé est générée à chaque démarrage**, rendant tous les chiffrés précédents illisibles. **Imposez** la présence de la variable.

```javascript
const crypto = require('crypto');

const ALGORITHM = 'aes-256-gcm';

// ⚠️ La clé de chiffrement DOIT venir de l'environnement (jamais de fallback).
// Format attendu : 32 octets bruts en base64 ou hex
function loadKey() {
    const hex = process.env.ENCRYPTION_KEY_HEX;
    if (!hex || hex.length !== 64) {
        throw new Error(
            "ENCRYPTION_KEY_HEX manquant ou invalide (32 octets = 64 caractères hex)\n" +
            "  Générez une clé : node -e \"console.log(require('crypto').randomBytes(32).toString('hex'))\""
        );
    }
    return Buffer.from(hex, 'hex');
}
const SECRET_KEY = loadKey();

class Encryption {
    static encrypt(text) {
        // GCM exige un IV de 12 octets (96 bits) ; 16 fonctionne aussi mais 12 est l'optimum NIST
        const iv = crypto.randomBytes(12);

        // ✅ createCipheriv passe explicitement clé + IV (correct cryptographiquement)
        const cipher = crypto.createCipheriv(ALGORITHM, SECRET_KEY, iv);

        const encrypted = Buffer.concat([
            cipher.update(text, 'utf8'),
            cipher.final()
        ]);

        const authTag = cipher.getAuthTag();

        return {
            encrypted: encrypted.toString('base64'),
            iv: iv.toString('base64'),
            authTag: authTag.toString('base64')
        };
    }

    static decrypt({ encrypted, iv, authTag }) {
        const decipher = crypto.createDecipheriv(
            ALGORITHM,
            SECRET_KEY,
            Buffer.from(iv, 'base64')
        );
        decipher.setAuthTag(Buffer.from(authTag, 'base64'));

        const decrypted = Buffer.concat([
            decipher.update(Buffer.from(encrypted, 'base64')),
            decipher.final()  // ⚠️ lève une exception si authTag invalide → données altérées
        ]);

        return decrypted.toString('utf8');
    }

    // ✅ PBKDF2 avec 600 000 itérations (OWASP 2023 minimum pour SHA-256)
    //    Pour SHA-512, OWASP recommande 210 000 itérations.
    //    Encore mieux : utilisez `bcrypt` ou `argon2` (npm install argon2).
    static hashPassword(password) {
        const salt = crypto.randomBytes(16).toString('hex');
        const hash = crypto.pbkdf2Sync(password, salt, 210000, 64, 'sha512').toString('hex');
        return { hash, salt, iterations: 210000, algorithm: 'sha512' };
    }

    static verifyPassword(password, hash, salt, iterations = 210000, algorithm = 'sha512') {
        const candidate = crypto.pbkdf2Sync(password, salt, iterations, 64, algorithm).toString('hex');

        // ✅ Comparaison à temps constant pour éviter les timing attacks
        try {
            return crypto.timingSafeEqual(
                Buffer.from(hash, 'hex'),
                Buffer.from(candidate, 'hex')
            );
        } catch {
            return false;  // longueurs différentes → buffers incompatibles
        }
    }
}

module.exports = Encryption;
```

**Génération d'une clé de chiffrement (à faire une fois, sauvegarder dans `.env`)** :

```bash
# Sortie : une chaîne hex de 64 caractères à mettre dans ENCRYPTION_KEY_HEX
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### Audit et logging des actions sensibles

**middleware/audit.js**
```javascript
const fs = require('fs').promises;  
const path = require('path');  

class AuditLogger {
    constructor() {
        this.logPath = path.join(__dirname, '../logs/audit.log');
        this.ensureLogDirectory();
    }

    async ensureLogDirectory() {
        const logDir = path.dirname(this.logPath);
        try {
            await fs.mkdir(logDir, { recursive: true });
        } catch (error) {
            console.error('Erreur création dossier logs:', error);
        }
    }

    async log(action, details, req) {
        const logEntry = {
            timestamp: new Date().toISOString(),
            action,
            user: req.user?.id || 'anonymous',
            ip: req.ip || req.connection.remoteAddress,
            userAgent: req.get('User-Agent'),
            details,
            sessionId: req.sessionID
        };

        try {
            const logLine = JSON.stringify(logEntry) + '\n';
            await fs.appendFile(this.logPath, logLine);
        } catch (error) {
            console.error('Erreur écriture audit log:', error);
        }
    }
}

const auditLogger = new AuditLogger();

// Middleware d'audit pour les actions sensibles
const auditAction = (action) => {
    return (req, res, next) => {
        // Capturer la réponse originale
        const originalSend = res.json;

        res.json = function(data) {
            // Logger seulement si l'action a réussi
            if (res.statusCode < 400) {
                auditLogger.log(action, {
                    method: req.method,
                    url: req.originalUrl,
                    body: req.body,
                    params: req.params,
                    statusCode: res.statusCode
                }, req);
            }

            return originalSend.call(this, data);
        };

        next();
    };
};

module.exports = { auditLogger, auditAction };
```

### Protection contre les attaques courantes

**middleware/security.js**
```javascript
const rateLimit = require('express-rate-limit');  
const slowDown = require('express-slow-down');  
const MongoStore = require('rate-limit-mongo'); // Si vous utilisez MongoDB pour stocker les limites  

// Protection contre les attaques par force brute
const strictLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // 5 tentatives max
    message: {
        error: 'Trop de tentatives, compte temporairement bloqué',
        retryAfter: '15 minutes'
    },
    standardHeaders: true,
    legacyHeaders: false,
    // Utiliser une base de données pour persister entre les redémarrages
    // store: new MongoStore({ ... })
});

// Ralentissement progressif
const speedLimiter = slowDown({
    windowMs: 15 * 60 * 1000, // 15 minutes
    delayAfter: 2, // Commencer à ralentir après 2 requêtes
    delayMs: 500, // Ajouter 500ms de délai par requête excédentaire
    maxDelayMs: 20000, // Délai maximum de 20 secondes
});

// Validation des headers
const validateHeaders = (req, res, next) => {
    // Vérifier Content-Type pour les requêtes POST/PUT/PATCH
    if (['POST', 'PUT', 'PATCH'].includes(req.method)) {
        const contentType = req.get('Content-Type');
        if (!contentType || !contentType.includes('application/json')) {
            return res.status(400).json({
                error: 'Content-Type application/json requis'
            });
        }
    }

    // Vérifier la taille des headers
    const headerSize = JSON.stringify(req.headers).length;
    if (headerSize > 8192) { // 8KB max
        return res.status(431).json({
            error: 'Headers trop volumineux'
        });
    }

    next();
};

// ⚠️ ANTIPATTERN : « sanitiser » l'entrée avec une liste noire de regex
// donne un FAUX sentiment de sécurité. Les bonnes pratiques OWASP sont :
//
//   1. Pour les injections SQL : **requêtes paramétrées** (déjà fait via `?`).
//   2. Pour les XSS : **échapper à la SORTIE** (au rendu HTML), pas à l'entrée.
//      Stockez les données en l'état ; échappez seulement quand vous les
//      insérez dans du HTML/JS via `escapeHtml()` ou un framework
//      (React/Vue/Svelte échappent automatiquement).
//   3. Pour les CSRF : **tokens anti-CSRF** (csurf, double-submit cookie).
//
// La fonction ci-dessous bloque des chaînes légitimes (ex. "function(x)" dans
// une description de doc) et est facilement contournable par encoding. Elle
// reste présentée à des fins pédagogiques mais N'EST PAS la bonne réponse.
const sanitizeInput = (req, res, next) => {
    const dangerousPatterns = [
        /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi,
        /javascript:/gi,
        /on\w+\s*=/gi,
        /expression\s*\(/gi,
        /eval\s*\(/gi,
        /function\s*\(/gi
    ];

    const sanitizeObject = (obj) => {
        if (typeof obj === 'string') {
            for (const pattern of dangerousPatterns) {
                if (pattern.test(obj)) {
                    throw new Error('Contenu potentiellement dangereux détecté');
                }
            }
            return obj.trim();
        }

        if (Array.isArray(obj)) {
            return obj.map(sanitizeObject);
        }

        if (obj && typeof obj === 'object') {
            const sanitized = {};
            for (const [key, value] of Object.entries(obj)) {
                sanitized[key] = sanitizeObject(value);
            }
            return sanitized;
        }

        return obj;
    };

    try {
        if (req.body) {
            req.body = sanitizeObject(req.body);
        }
        if (req.query) {
            req.query = sanitizeObject(req.query);
        }
        next();
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
};

module.exports = {
    strictLimiter,
    speedLimiter,
    validateHeaders,
    sanitizeInput
};
```

### Configuration HTTPS et certificats

**scripts/generate-ssl.sh**
```bash
#!/bin/bash

# Générer des certificats auto-signés pour le développement
# NE PAS UTILISER EN PRODUCTION

echo "🔐 Génération des certificats SSL pour le développement..."

# Créer le dossier SSL
mkdir -p ssl

# Générer la clé privée
openssl genpkey -algorithm RSA -out ssl/private-key.pem -pkcs8 -aes256

# Générer le certificat auto-signé
openssl req -new -x509 -key ssl/private-key.pem -out ssl/certificate.pem -days 365 \
    -subj "/C=FR/ST=Paris/L=Paris/O=Dev/OU=API/CN=localhost"

echo "✅ Certificats générés dans le dossier ssl/"  
echo "⚠️  ATTENTION : Ces certificats sont pour le développement uniquement !"  
```

**Configuration HTTPS dans server.js**
```javascript
const https = require('https');  
const fs = require('fs');  

// Configuration HTTPS pour la production
if (process.env.NODE_ENV === 'production' && process.env.ENABLE_HTTPS === 'true') {
    try {
        const httpsOptions = {
            key: fs.readFileSync(process.env.SSL_KEY_PATH || './ssl/private-key.pem'),
            cert: fs.readFileSync(process.env.SSL_CERT_PATH || './ssl/certificate.pem'),
            // Configurations de sécurité
            secureOptions: crypto.constants.SSL_OP_NO_TLSv1 | crypto.constants.SSL_OP_NO_TLSv1_1,
            ciphers: 'ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20:!aNULL:!MD5:!DSS',
            honorCipherOrder: true
        };

        const httpsServer = https.createServer(httpsOptions, app);
        httpsServer.listen(PORT + 1, () => {
            console.log(`🔒 Serveur HTTPS démarré sur le port ${PORT + 1}`);
        });

        // Rediriger HTTP vers HTTPS
        const httpApp = express();
        httpApp.use((req, res) => {
            res.redirect(301, `https://${req.headers.host}${req.url}`);
        });
        httpApp.listen(PORT, () => {
            console.log(`↗️ Redirection HTTP vers HTTPS sur le port ${PORT}`);
        });

    } catch (error) {
        console.error('❌ Erreur configuration HTTPS:', error);
        process.exit(1);
    }
}
```

## Tests de charge et performance

### Test de charge avec Artillery

**load-test.yml**
```yaml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
    - duration: 120
      arrivalRate: 50
    - duration: 60
      arrivalRate: 100
  defaults:
    headers:
      Content-Type: 'application/json'

scenarios:
  - name: "Test API complète"
    weight: 70
    flow:
      - get:
          url: "/api/health"
      - get:
          url: "/api/personnes"
          qs:
            page: 1
            limit: 10
      - post:
          url: "/api/personnes"
          json:
            nom: "Test User {{ $randomString() }}"
            age: "{{ $randomInt(18, 65) }}"
            email: "test-{{ $randomString() }}@example.com"
          capture:
            - json: "$.id"
              as: "personneId"
      - get:
          url: "/api/personnes/{{ personneId }}"
      - patch:
          url: "/api/personnes/{{ personneId }}"
          json:
            age: "{{ $randomInt(18, 65) }}"
      - delete:
          url: "/api/personnes/{{ personneId }}"

  - name: "Test lecture seule"
    weight: 30
    flow:
      - get:
          url: "/api/personnes"
          qs:
            search: "Test"
      - get:
          url: "/api/stats"
```

**Commandes de test :**
```bash
# Installer Artillery
npm install -g artillery

# Lancer le test de charge
artillery run load-test.yml

# Test rapide
artillery quick --duration 60 --rate 10 http://localhost:3000/api/health

# Générer un rapport
artillery run load-test.yml --output report.json  
artillery report report.json  
```

### Optimisation des performances

**middleware/compression.js**
```javascript
const compression = require('compression');

// Configuration de compression optimisée
const compressionConfig = compression({
    level: 6, // Niveau de compression (1-9)
    threshold: 1024, // Compresser seulement si > 1KB
    filter: (req, res) => {
        // Ne pas compresser si le client ne le supporte pas
        if (req.headers['x-no-compression']) {
            return false;
        }

        // Utiliser le filtre par défaut de compression
        return compression.filter(req, res);
    }
});

module.exports = compressionConfig;
```

**Cache intelligent pour les requêtes**
```javascript
const NodeCache = require('node-cache');

class SmartCache {
    constructor() {
        this.cache = new NodeCache({
            stdTTL: 300, // 5 minutes par défaut
            checkperiod: 60, // Vérifier les expirations chaque minute
            useClones: false // Améliore les performances
        });

        this.stats = {
            hits: 0,
            misses: 0,
            sets: 0
        };
    }

    get(key) {
        const value = this.cache.get(key);
        if (value !== undefined) {
            this.stats.hits++;
            return value;
        }
        this.stats.misses++;
        return null;
    }

    set(key, value, ttl = null) {
        this.stats.sets++;
        if (ttl) {
            return this.cache.set(key, value, ttl);
        }
        return this.cache.set(key, value);
    }

    del(key) {
        return this.cache.del(key);
    }

    flush() {
        return this.cache.flushAll();
    }

    getStats() {
        return {
            ...this.stats,
            keys: this.cache.keys().length,
            hitRate: this.stats.hits / (this.stats.hits + this.stats.misses) * 100
        };
    }
}

const cache = new SmartCache();

// Middleware de cache pour les requêtes GET
const cacheMiddleware = (ttl = 300) => {
    return (req, res, next) => {
        if (req.method !== 'GET') {
            return next();
        }

        const key = `${req.originalUrl}`;
        const cached = cache.get(key);

        if (cached) {
            return res.json(cached);
        }

        // Intercepter la réponse pour la mettre en cache
        const originalJson = res.json;
        res.json = function(data) {
            if (res.statusCode === 200) {
                cache.set(key, data, ttl);
            }
            return originalJson.call(this, data);
        };

        next();
    };
};

// Invalidation du cache lors des modifications
const invalidateCache = (patterns = []) => {
    return (req, res, next) => {
        const originalJson = res.json;
        res.json = function(data) {
            if (res.statusCode < 300) {
                // Invalider les clés correspondant aux patterns
                patterns.forEach(pattern => {
                    const regex = new RegExp(pattern);
                    cache.cache.keys().forEach(key => {
                        if (regex.test(key)) {
                            cache.del(key);
                        }
                    });
                });
            }
            return originalJson.call(this, data);
        };
        next();
    };
};

module.exports = { cache, cacheMiddleware, invalidateCache };
```

## Maintenance et sauvegarde

### Script de sauvegarde automatique

**scripts/backup.sh**
```bash
#!/bin/bash
set -euo pipefail   # stop sur erreur, variable non définie, ou échec dans un pipe

# Configuration
DB_PATH="${DATABASE_PATH:-./database.db}"  
BACKUP_DIR="${BACKUP_DIR:-./backups}"  
RETENTION_DAYS="${RETENTION_DAYS:-30}"  
DATE=$(date +%Y%m%d_%H%M%S)  
BACKUP_FILE="$BACKUP_DIR/database_backup_$DATE.db"  

# Créer le dossier de sauvegarde
mkdir -p "$BACKUP_DIR"

echo "🔄 Début de la sauvegarde de la base de données..."

# Vérifier que la base de données existe
if [ ! -f "$DB_PATH" ]; then
    echo "❌ Base de données non trouvée : $DB_PATH"
    exit 1
fi

# ⚠️ NE PAS utiliser `cp` sur une base SQLite en cours d'utilisation :
#    si une transaction est en cours, la copie peut être incohérente ou corrompue.
#    On utilise la commande `.backup` du shell SQLite qui copie de façon
#    transactionnellement sûre, même base ouverte en écriture (cf. chapitre 6.4).
if sqlite3 "$DB_PATH" ".backup '$BACKUP_FILE'"; then
    echo "✅ Sauvegarde créée : $BACKUP_FILE"

    # Vérifier l'intégrité de la copie AVANT de la compresser
    if [ "$(sqlite3 "$BACKUP_FILE" 'PRAGMA integrity_check;')" != "ok" ]; then
        echo "❌ Sauvegarde corrompue — abandon"
        rm -f "$BACKUP_FILE"
        exit 1
    fi
    echo "✅ Intégrité SQLite vérifiée (PRAGMA integrity_check = ok)"

    # Compresser la sauvegarde
    if gzip "$BACKUP_FILE"; then
        echo "✅ Sauvegarde compressée : $BACKUP_FILE.gz"
        BACKUP_FILE="$BACKUP_FILE.gz"
    fi

    # Vérifier l'intégrité
    if [ "${BACKUP_FILE%.gz}" != "$BACKUP_FILE" ]; then
        # Fichier compressé
        if gzip -t "$BACKUP_FILE"; then
            echo "✅ Intégrité vérifiée"
        else
            echo "❌ Erreur d'intégrité de la sauvegarde"
            exit 1
        fi
    fi

    # Nettoyer les anciennes sauvegardes
    echo "🧹 Nettoyage des anciennes sauvegardes (>${RETENTION_DAYS} jours)..."
    find "$BACKUP_DIR" -name "database_backup_*.db.gz" -type f -mtime +$RETENTION_DAYS -delete

    # Statistiques
    BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    BACKUP_COUNT=$(ls -1 "$BACKUP_DIR"/database_backup_*.db.gz 2>/dev/null | wc -l)

    echo "📊 Taille de la sauvegarde : $BACKUP_SIZE"
    echo "📊 Nombre total de sauvegardes : $BACKUP_COUNT"
    echo "🎉 Sauvegarde terminée avec succès !"

else
    echo "❌ Échec de la sauvegarde"
    exit 1
fi
```

### Surveillance du système

**monitoring/system-monitor.js**
```javascript
const os = require('os');  
const fs = require('fs').promises;  
const { execSync } = require('child_process');  

class SystemMonitor {
    constructor() {
        this.alerts = [];
        this.thresholds = {
            cpu: 80,      // %
            memory: 85,   // %
            disk: 90,     // %
            load: 2.0     // load average
        };
    }

    async getSystemInfo() {
        const cpuUsage = this.getCPUUsage();
        const memoryUsage = this.getMemoryUsage();
        const diskUsage = await this.getDiskUsage();
        const loadAverage = os.loadavg()[0];

        return {
            timestamp: new Date().toISOString(),
            cpu: Math.round(cpuUsage * 100) / 100,
            memory: Math.round(memoryUsage * 100) / 100,
            disk: Math.round(diskUsage * 100) / 100,
            load: Math.round(loadAverage * 100) / 100,
            uptime: Math.round(process.uptime()),
            nodeMemory: process.memoryUsage(),
            version: process.version
        };
    }

    getCPUUsage() {
        const cpus = os.cpus();
        let totalIdle = 0;
        let totalTick = 0;

        cpus.forEach(cpu => {
            for (const type in cpu.times) {
                totalTick += cpu.times[type];
            }
            totalIdle += cpu.times.idle;
        });

        return 100 - (totalIdle / totalTick * 100);
    }

    getMemoryUsage() {
        const totalMem = os.totalmem();
        const freeMem = os.freemem();
        return ((totalMem - freeMem) / totalMem) * 100;
    }

    async getDiskUsage() {
        try {
            const stats = await fs.stat('./');
            const diskInfo = execSync('df -h .', { encoding: 'utf8' });
            const lines = diskInfo.split('\n');
            const dataLine = lines[1];
            const usage = dataLine.split(/\s+/)[4];
            return parseFloat(usage.replace('%', ''));
        } catch (error) {
            console.error('Erreur lecture disque:', error);
            return 0;
        }
    }

    checkAlerts(systemInfo) {
        const newAlerts = [];

        if (systemInfo.cpu > this.thresholds.cpu) {
            newAlerts.push({
                type: 'cpu',
                message: `Utilisation CPU élevée: ${systemInfo.cpu}%`,
                severity: 'warning'
            });
        }

        if (systemInfo.memory > this.thresholds.memory) {
            newAlerts.push({
                type: 'memory',
                message: `Utilisation mémoire élevée: ${systemInfo.memory}%`,
                severity: 'warning'
            });
        }

        if (systemInfo.disk > this.thresholds.disk) {
            newAlerts.push({
                type: 'disk',
                message: `Utilisation disque élevée: ${systemInfo.disk}%`,
                severity: 'critical'
            });
        }

        if (systemInfo.load > this.thresholds.load) {
            newAlerts.push({
                type: 'load',
                message: `Charge système élevée: ${systemInfo.load}`,
                severity: 'warning'
            });
        }

        this.alerts = newAlerts;
        return newAlerts;
    }

    async startMonitoring(interval = 60000) {
        console.log('🔍 Démarrage du monitoring système...');

        setInterval(async () => {
            try {
                const systemInfo = await this.getSystemInfo();
                const alerts = this.checkAlerts(systemInfo);

                if (alerts.length > 0) {
                    console.warn('⚠️ Alertes système:', alerts);
                    // Ici vous pourriez envoyer des notifications
                }

                // Log des métriques
                console.log(`📊 CPU: ${systemInfo.cpu}% | RAM: ${systemInfo.memory}% | Disk: ${systemInfo.disk}% | Load: ${systemInfo.load}`);

            } catch (error) {
                console.error('Erreur monitoring:', error);
            }
        }, interval);
    }
}

module.exports = SystemMonitor;
```

## Script de maintenance complet

**scripts/maintenance.js**
```javascript
#!/usr/bin/env node

const { execSync } = require('child_process');  
const fs = require('fs').promises;  
const path = require('path');  
const sqlite3 = require('sqlite3').verbose();  

class MaintenanceScript {
    constructor() {
        this.dbPath = process.env.DATABASE_PATH || './database.db';
        this.logPath = './logs';
        this.backupPath = './backups';
    }

    async run() {
        console.log('🔧 Début de la maintenance...\n');

        try {
            await this.cleanLogs();
            await this.optimizeDatabase();
            await this.checkDatabaseIntegrity();
            await this.updateStatistics();
            await this.cleanOldBackups();

            console.log('\n✅ Maintenance terminée avec succès !');
        } catch (error) {
            console.error('\n❌ Erreur durant la maintenance:', error);
            process.exit(1);
        }
    }

    async cleanLogs() {
        console.log('🧹 Nettoyage des logs...');

        try {
            const files = await fs.readdir(this.logPath);
            const cutoffDate = new Date();
            cutoffDate.setDate(cutoffDate.getDate() - 30); // Garder 30 jours

            let cleanedCount = 0;

            for (const file of files) {
                const filePath = path.join(this.logPath, file);
                const stats = await fs.stat(filePath);

                if (stats.mtime < cutoffDate && file.endsWith('.log')) {
                    await fs.unlink(filePath);
                    cleanedCount++;
                }
            }

            console.log(`   ✓ ${cleanedCount} anciens fichiers de log supprimés`);
        } catch (error) {
            console.log(`   ⚠️ Erreur nettoyage logs: ${error.message}`);
        }
    }

    async optimizeDatabase() {
        console.log('⚡ Optimisation de la base de données...');

        // ⚠️ Précautions VACUUM :
        //   • Réécrit la base entière → besoin de ~2× l'espace disque libre.
        //   • Acquiert un verrou exclusif → bloque les autres connexions.
        //   • Sur très grosses bases (> 10 Go), VACUUM peut prendre des heures.
        //   • Préférez `PRAGMA incremental_vacuum` (avec `auto_vacuum = INCREMENTAL`
        //     défini à la création de la base) pour des nettoyages partiels.

        return new Promise((resolve, reject) => {
            const db = new sqlite3.Database(this.dbPath);

            // VACUUM pour réorganiser et compresser
            db.run('VACUUM', (err) => {
                if (err) {
                    console.log(`   ⚠️ Erreur VACUUM: ${err.message}`);
                } else {
                    console.log('   ✓ VACUUM exécuté');
                }

                // ANALYZE pour mettre à jour les statistiques
                db.run('ANALYZE', (err) => {
                    if (err) {
                        console.log(`   ⚠️ Erreur ANALYZE: ${err.message}`);
                    } else {
                        console.log('   ✓ ANALYZE exécuté');
                    }

                    db.close();
                    resolve();
                });
            });
        });
    }

    async checkDatabaseIntegrity() {
        console.log('🔍 Vérification de l\'intégrité de la base...');

        return new Promise((resolve, reject) => {
            const db = new sqlite3.Database(this.dbPath);

            db.get('PRAGMA integrity_check', (err, row) => {
                if (err) {
                    console.log(`   ❌ Erreur vérification: ${err.message}`);
                    reject(err);
                } else if (row && row.integrity_check === 'ok') {
                    console.log('   ✓ Intégrité vérifiée');
                } else {
                    console.log(`   ⚠️ Problème d'intégrité: ${row.integrity_check}`);
                }

                db.close();
                resolve();
            });
        });
    }

    async updateStatistics() {
        console.log('📊 Mise à jour des statistiques...');

        return new Promise((resolve, reject) => {
            const db = new sqlite3.Database(this.dbPath);

            const queries = [
                'UPDATE sqlite_stat1 SET stat = NULL',
                'ANALYZE'
            ];

            let completed = 0;

            queries.forEach(query => {
                db.run(query, (err) => {
                    if (err) {
                        console.log(`   ⚠️ Erreur: ${err.message}`);
                    }

                    completed++;
                    if (completed === queries.length) {
                        console.log('   ✓ Statistiques mises à jour');
                        db.close();
                        resolve();
                    }
                });
            });
        });
    }

    async cleanOldBackups() {
        console.log('🗂️ Nettoyage des anciennes sauvegardes...');

        try {
            const files = await fs.readdir(this.backupPath);
            const cutoffDate = new Date();
            cutoffDate.setDate(cutoffDate.getDate() - 90); // Garder 90 jours

            let cleanedCount = 0;

            for (const file of files) {
                if (file.startsWith('database_backup_')) {
                    const filePath = path.join(this.backupPath, file);
                    const stats = await fs.stat(filePath);

                    if (stats.mtime < cutoffDate) {
                        await fs.unlink(filePath);
                        cleanedCount++;
                    }
                }
            }

            console.log(`   ✓ ${cleanedCount} anciennes sauvegardes supprimées`);
        } catch (error) {
            console.log(`   ⚠️ Erreur nettoyage sauvegardes: ${error.message}`);
        }
    }
}

// Exécuter si appelé directement
if (require.main === module) {
    const maintenance = new MaintenanceScript();
    maintenance.run();
}

module.exports = MaintenanceScript;
```

## Résumé et checklist finale

### ✅ Checklist complète de production

**🏗️ Architecture et code**
- [ ] Structure de projet claire et modulaire
- [ ] Séparation des responsabilités (modèles, routes, middleware)
- [ ] Configuration par variables d'environnement
- [ ] Gestion d'erreurs robuste et centralisée
- [ ] Validation des données à tous les niveaux

**🔒 Sécurité**
- [ ] Authentification JWT implémentée (`algorithms: ['HS256']` explicite, secret ≥ 32 chars)
- [ ] Rate limiting configuré (login + endpoints sensibles)
- [ ] Headers de sécurité (Helmet, CSP, HSTS)
- [ ] **Validation** des entrées (Joi/Zod) + **échappement à la SORTIE** (pas de "sanitize" sur l'entrée)
- [ ] Requêtes paramétrées partout (jamais de concaténation SQL)
- [ ] HTTPS en production + redirection HTTP → HTTPS
- [ ] CORS restreint aux origines autorisées (pas `*`)
- [ ] Mots de passe hashés avec bcrypt (cost ≥ 12) ou Argon2
- [ ] Messages d'erreur génériques au client (pas de `str(e)` brut)
- [ ] Audit des actions sensibles (login, modifications de rôles)
- [ ] Chiffrement des données au repos (AES-256-GCM avec `createCipheriv`)

**📊 Performance et monitoring**
- [ ] Cache intelligent implémenté
- [ ] Compression des réponses
- [ ] Optimisation des requêtes SQL
- [ ] Health checks détaillés
- [ ] Monitoring système
- [ ] Métriques de performance
- [ ] Tests de charge effectués

**🧪 Qualité et tests**
- [ ] Tests unitaires complets
- [ ] Tests d'intégration
- [ ] Tests de charge
- [ ] Documentation API (Swagger)
- [ ] Code review et linting
- [ ] Couverture de code > 80%

**🚀 Déploiement et infrastructure**
- [ ] Containerisation Docker
- [ ] Configuration Nginx/reverse proxy
- [ ] Scripts de déploiement automatisés
- [ ] Sauvegarde automatique
- [ ] Logs centralisés
- [ ] Monitoring et alertes

**🔧 Maintenance**
- [ ] Scripts de maintenance automatisés
- [ ] Nettoyage automatique des logs
- [ ] Optimisation périodique de la DB
- [ ] Vérification d'intégrité
- [ ] Rotation des sauvegardes

### 🎯 Points clés pour débutants

1. **Commencez simple** : Implémentez d'abord les fonctionnalités de base (CRUD)
2. **Sécurisez progressivement** : Ajoutez les mesures de sécurité une par une
3. **Testez continuellement** : Chaque fonctionnalité doit être testée
4. **Documentez tout** : Code, API, procédures de déploiement
5. **Surveillez en production** : Logs, métriques, alertes sont essentiels

### 📚 Ressources pour aller plus loin

**Documentation officielle :**
- **Express.js** : [https://expressjs.com/](https://expressjs.com/)
- **SQLite** : [https://sqlite.org/docs.html](https://sqlite.org/docs.html)
- **Flask** : [https://flask.palletsprojects.com/](https://flask.palletsprojects.com/)
- **Swagger/OpenAPI** : [https://swagger.io/specification/](https://swagger.io/specification/)
- **JWT** : [https://jwt.io/introduction](https://jwt.io/introduction)

**Outils et frameworks :**
- **Postman** : [https://www.postman.com/](https://www.postman.com/) - Test d'APIs
- **Insomnia** : [https://insomnia.rest/](https://insomnia.rest/) - Alternative à Postman
- **Thunder Client** : Extension VS Code pour tester les APIs
- **DB Browser for SQLite** : [https://sqlitebrowser.org/](https://sqlitebrowser.org/)
- **Artillery** : [https://artillery.io/](https://artillery.io/) - Tests de charge
- **Jest** : [https://jestjs.io/](https://jestjs.io/) - Framework de tests JavaScript

**Sécurité :**
- **OWASP API Security** : [https://owasp.org/www-project-api-security/](https://owasp.org/www-project-api-security/)
- **Helmet.js** : [https://helmetjs.github.io/](https://helmetjs.github.io/)
- **Rate limiting strategies** : [https://blog.logrocket.com/rate-limiting-node-js/](https://blog.logrocket.com/rate-limiting-node-js/)

## Exemples de projets complets

### 1. API de blog simple

**Fonctionnalités :**
- Gestion des articles (CRUD)
- Système de commentaires
- Catégories et tags
- Recherche full-text
- Authentication des auteurs

**Technologies :**
- Node.js + Express
- SQLite avec FTS5
- JWT pour l'auth
- Swagger pour la doc

### 2. API de gestion d'inventaire

**Fonctionnalités :**
- Produits et catégories
- Gestion des stocks
- Historique des mouvements
- Alertes de stock bas
- Rapports de vente

**Technologies :**
- Python + Flask
- SQLite avec triggers
- Redis pour le cache
- Grafana pour les métriques

### 3. API de système de réservation

**Fonctionnalités :**
- Gestion des créneaux
- Réservations et annulations
- Notifications par email
- Gestion des conflits
- Calendrier intégré

**Technologies :**
- Node.js + Express
- SQLite avec contraintes complexes
- WebSockets pour temps réel
- Cron jobs pour les rappels

## Patterns architecturaux avancés

### Architecture hexagonale (Ports & Adapters)

```javascript
// Port (interface)
class PersonneRepository {
    async findById(id) { throw new Error('Not implemented'); }
    async save(personne) { throw new Error('Not implemented'); }
    async delete(id) { throw new Error('Not implemented'); }
}

// Adapter SQLite
class SQLitePersonneRepository extends PersonneRepository {
    constructor(db) {
        super();
        this.db = db;
    }

    async findById(id) {
        return new Promise((resolve, reject) => {
            this.db.get('SELECT * FROM personnes WHERE id = ?', [id], (err, row) => {
                if (err) reject(err);
                else resolve(row);
            });
        });
    }

    // ... autres méthodes
}

// Service métier (indépendant de la DB)
class PersonneService {
    constructor(personneRepo) {
        this.personneRepo = personneRepo;
    }

    async obtenirPersonne(id) {
        const personne = await this.personneRepo.findById(id);
        if (!personne) {
            throw new Error('Personne non trouvée');
        }
        return personne;
    }

    async creerPersonne(data) {
        // Logique métier
        this.validerDonnees(data);
        return await this.personneRepo.save(data);
    }

    validerDonnees(data) {
        if (!data.email || !data.email.includes('@')) {
            throw new Error('Email invalide');
        }
        // Autres validations métier
    }
}

// Injection de dépendances
const db = new sqlite3.Database('app.db');  
const personneRepo = new SQLitePersonneRepository(db);  
const personneService = new PersonneService(personneRepo);  
```

### Event Sourcing simple

```javascript
class EventStore {
    constructor(db) {
        this.db = db;
        this.createTable();
    }

    createTable() {
        this.db.run(`
            CREATE TABLE IF NOT EXISTS events (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                aggregate_id TEXT NOT NULL,
                event_type TEXT NOT NULL,
                event_data TEXT NOT NULL,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                version INTEGER NOT NULL
            )
        `);
    }

    async saveEvent(aggregateId, eventType, eventData, expectedVersion) {
        return new Promise((resolve, reject) => {
            this.db.run(`
                INSERT INTO events (aggregate_id, event_type, event_data, version)
                VALUES (?, ?, ?, ?)
            `, [aggregateId, eventType, JSON.stringify(eventData), expectedVersion + 1],
            function(err) {
                if (err) reject(err);
                else resolve(this.lastID);
            });
        });
    }

    async getEvents(aggregateId, fromVersion = 0) {
        return new Promise((resolve, reject) => {
            this.db.all(`
                SELECT * FROM events
                WHERE aggregate_id = ? AND version > ?
                ORDER BY version
            `, [aggregateId, fromVersion], (err, rows) => {
                if (err) reject(err);
                else resolve(rows.map(row => ({
                    ...row,
                    event_data: JSON.parse(row.event_data)
                })));
            });
        });
    }
}

// Exemple d'utilisation
class PersonneAggregate {
    constructor(id) {
        this.id = id;
        this.version = 0;
        this.nom = null;
        this.email = null;
        this.events = [];
    }

    static fromEvents(events) {
        const aggregate = new PersonneAggregate(events[0].aggregate_id);
        events.forEach(event => aggregate.apply(event));
        return aggregate;
    }

    creer(nom, email) {
        this.addEvent('PersonneCreee', { nom, email });
    }

    changerEmail(nouvelEmail) {
        if (nouvelEmail === this.email) return;
        this.addEvent('EmailChange', { ancienEmail: this.email, nouvelEmail });
    }

    addEvent(eventType, eventData) {
        const event = {
            aggregate_id: this.id,
            event_type: eventType,
            event_data: eventData,
            version: this.version + 1
        };
        this.events.push(event);
        this.apply(event);
    }

    apply(event) {
        switch (event.event_type) {
            case 'PersonneCreee':
                this.nom = event.event_data.nom;
                this.email = event.event_data.email;
                break;
            case 'EmailChange':
                this.email = event.event_data.nouvelEmail;
                break;
        }
        this.version = event.version;
    }

    getUncommittedEvents() {
        return this.events;
    }

    markEventsAsCommitted() {
        this.events = [];
    }
}
```

### CQRS (Command Query Responsibility Segregation)

```javascript
// Commands (écriture)
class CreerPersonneCommand {
    constructor(nom, email) {
        this.nom = nom;
        this.email = email;
    }
}

class CommandHandler {
    constructor(eventStore) {
        this.eventStore = eventStore;
    }

    async handle(command) {
        if (command instanceof CreerPersonneCommand) {
            const aggregate = new PersonneAggregate(generateId());
            aggregate.creer(command.nom, command.email);

            const events = aggregate.getUncommittedEvents();
            for (const event of events) {
                await this.eventStore.saveEvent(
                    event.aggregate_id,
                    event.event_type,
                    event.event_data,
                    aggregate.version - 1
                );
            }

            return aggregate.id;
        }
    }
}

// Queries (lecture) - vue matérialisée
class PersonneReadModel {
    constructor(db) {
        this.db = db;
        this.createTable();
    }

    createTable() {
        this.db.run(`
            CREATE TABLE IF NOT EXISTS personnes_view (
                id TEXT PRIMARY KEY,
                nom TEXT,
                email TEXT,
                date_creation DATETIME,
                derniere_modification DATETIME
            )
        `);
    }

    async findById(id) {
        return new Promise((resolve, reject) => {
            this.db.get('SELECT * FROM personnes_view WHERE id = ?', [id], (err, row) => {
                if (err) reject(err);
                else resolve(row);
            });
        });
    }

    async findAll() {
        return new Promise((resolve, reject) => {
            this.db.all('SELECT * FROM personnes_view ORDER BY nom', (err, rows) => {
                if (err) reject(err);
                else resolve(rows);
            });
        });
    }
}

// Projection des événements vers la vue de lecture
class PersonneProjection {
    constructor(readModel) {
        this.readModel = readModel;
    }

    async project(event) {
        switch (event.event_type) {
            case 'PersonneCreee':
                await this.readModel.db.run(`
                    INSERT INTO personnes_view (id, nom, email, date_creation, derniere_modification)
                    VALUES (?, ?, ?, ?, ?)
                `, [event.aggregate_id, event.event_data.nom, event.event_data.email,
                    event.timestamp, event.timestamp]);
                break;

            case 'EmailChange':
                await this.readModel.db.run(`
                    UPDATE personnes_view
                    SET email = ?, derniere_modification = ?
                    WHERE id = ?
                `, [event.event_data.nouvelEmail, event.timestamp, event.aggregate_id]);
                break;
        }
    }
}
```

## Migration vers d'autres bases de données

### Stratégie de migration progressive

**1. Dual Write Pattern**
```javascript
class DualWriteRepository {
    constructor(sqliteRepo, postgresRepo) {
        this.primary = sqliteRepo;    // Ancien système
        this.secondary = postgresRepo; // Nouveau système
        this.migrationEnabled = process.env.DUAL_WRITE_ENABLED === 'true';
    }

    async save(data) {
        // Écrire dans l'ancien système (source de vérité)
        const result = await this.primary.save(data);

        if (this.migrationEnabled) {
            try {
                // Écrire aussi dans le nouveau système
                await this.secondary.save(data);
            } catch (error) {
                // Logger l'erreur mais ne pas faire échouer l'opération
                console.error('Erreur dual write:', error);
            }
        }

        return result;
    }

    async findById(id) {
        // Lire depuis l'ancien système pour l'instant
        return await this.primary.findById(id);
    }
}
```

**2. Script de migration des données**
```javascript
class DataMigrator {
    constructor(sourceDb, targetDb) {
        this.source = sourceDb;
        this.target = targetDb;
        this.batchSize = 1000;
    }

    async migrate() {
        console.log('🔄 Début de la migration...');

        const totalRecords = await this.getTotalRecords();
        let processedRecords = 0;

        while (processedRecords < totalRecords) {
            const batch = await this.getBatch(processedRecords, this.batchSize);

            await this.target.transaction(async (trx) => {
                for (const record of batch) {
                    await this.migrateRecord(record, trx);
                }
            });

            processedRecords += batch.length;
            const progress = (processedRecords / totalRecords * 100).toFixed(1);
            console.log(`📊 Progression: ${progress}% (${processedRecords}/${totalRecords})`);
        }

        console.log('✅ Migration terminée !');
    }

    async getTotalRecords() {
        return new Promise((resolve, reject) => {
            this.source.get('SELECT COUNT(*) as count FROM personnes', (err, row) => {
                if (err) reject(err);
                else resolve(row.count);
            });
        });
    }

    async getBatch(offset, limit) {
        return new Promise((resolve, reject) => {
            this.source.all(
                'SELECT * FROM personnes LIMIT ? OFFSET ?',
                [limit, offset],
                (err, rows) => {
                    if (err) reject(err);
                    else resolve(rows);
                }
            );
        });
    }

    async migrateRecord(record, trx) {
        // Transformer les données si nécessaire
        const transformedRecord = this.transformRecord(record);

        // Insérer dans la nouvelle base
        await trx('personnes').insert(transformedRecord);
    }

    transformRecord(record) {
        return {
            id: record.id,
            nom: record.nom,
            email: record.email,
            age: record.age,
            created_at: record.date_creation,
            updated_at: record.date_creation
        };
    }
}
```

## Optimisations spécifiques à SQLite

### Configuration avancée

```javascript
class OptimizedSQLiteDB {
    constructor(dbPath) {
        this.db = new sqlite3.Database(dbPath);
        this.optimize();
    }

    optimize() {
        // Optimisations de performance
        this.db.serialize(() => {
            // Mode WAL pour de meilleures performances concurrentes
            this.db.run('PRAGMA journal_mode = WAL');

            // Synchronisation moins stricte (plus rapide, moins sûr)
            this.db.run('PRAGMA synchronous = NORMAL');

            // Cache plus important
            this.db.run('PRAGMA cache_size = 10000'); // 10000 pages

            // Délai avant commit automatique
            this.db.run('PRAGMA busy_timeout = 30000'); // 30 secondes

            // Optimisations mémoire
            this.db.run('PRAGMA temp_store = MEMORY');
            this.db.run('PRAGMA mmap_size = 268435456'); // 256MB

            // Optimisations requêtes
            this.db.run('PRAGMA optimize');
        });
    }

    async createIndexes() {
        const indexes = [
            'CREATE INDEX IF NOT EXISTS idx_personnes_email ON personnes(email)',
            'CREATE INDEX IF NOT EXISTS idx_personnes_nom ON personnes(nom)',
            'CREATE INDEX IF NOT EXISTS idx_personnes_date ON personnes(date_creation)',
            // Index composite pour les recherches complexes
            'CREATE INDEX IF NOT EXISTS idx_personnes_nom_email ON personnes(nom, email)',
        ];

        for (const indexSQL of indexes) {
            await this.run(indexSQL);
        }
    }

    run(sql, params = []) {
        return new Promise((resolve, reject) => {
            this.db.run(sql, params, function(err) {
                if (err) reject(err);
                else resolve(this);
            });
        });
    }

    async enableExtensions() {
        // Activer les extensions si compilées
        try {
            await this.run('SELECT load_extension("fts5")');
            console.log('✅ Extension FTS5 activée');
        } catch (error) {
            console.log('⚠️ Extension FTS5 non disponible');
        }
    }
}
```

### Recherche full-text avancée

```javascript
class FullTextSearch {
    constructor(db) {
        this.db = db;
        this.createFTSTable();
    }

    createFTSTable() {
        // Mode STANDALONE : la table FTS5 stocke les données indexées en interne.
        // ⚠️ Cela DOUBLE l'espace disque (les données sont à la fois dans
        //    `personnes` ET dans `personnes_fts`). Pour gros volume, préférez
        //    le mode "external content" — voir chapitre 6.6 sur FTS5.
        this.db.run(`
            CREATE VIRTUAL TABLE IF NOT EXISTS personnes_fts USING fts5(
                nom,
                email,
                content_id UNINDEXED  -- ID de référence vers la table principale
            )
        `);

        // Triggers de synchronisation
        // En mode standalone, UPDATE/DELETE directs sur la table FTS5 sont OK
        // (contrairement au mode "external content" qui requiert le pattern
        //  INSERT INTO personnes_fts(personnes_fts, ...) VALUES('delete', ...)).
        this.db.run(`
            CREATE TRIGGER IF NOT EXISTS personnes_fts_insert AFTER INSERT ON personnes
            BEGIN
                INSERT INTO personnes_fts(nom, email, content_id)
                VALUES (NEW.nom, NEW.email, NEW.id);
            END
        `);

        this.db.run(`
            CREATE TRIGGER IF NOT EXISTS personnes_fts_update AFTER UPDATE ON personnes
            BEGIN
                UPDATE personnes_fts
                SET nom = NEW.nom, email = NEW.email
                WHERE content_id = NEW.id;
            END
        `);

        this.db.run(`
            CREATE TRIGGER IF NOT EXISTS personnes_fts_delete AFTER DELETE ON personnes
            BEGIN
                DELETE FROM personnes_fts WHERE content_id = OLD.id;
            END
        `);
    }

    async search(query, options = {}) {
        const {
            limit = 10,
            offset = 0,
            highlight = true
        } = options;

        const searchQuery = highlight ? `
            SELECT
                p.*,
                highlight(pf.personnes_fts, 0, '<mark>', '</mark>') as nom_highlighted,
                highlight(pf.personnes_fts, 1, '<mark>', '</mark>') as email_highlighted,
                rank
            FROM personnes_fts pf
            JOIN personnes p ON p.id = pf.content_id
            WHERE personnes_fts MATCH ?
            ORDER BY rank
            LIMIT ? OFFSET ?
        ` : `
            SELECT p.*, rank
            FROM personnes_fts pf
            JOIN personnes p ON p.id = pf.content_id
            WHERE personnes_fts MATCH ?
            ORDER BY rank
            LIMIT ? OFFSET ?
        `;

        return new Promise((resolve, reject) => {
            this.db.all(searchQuery, [query, limit, offset], (err, rows) => {
                if (err) reject(err);
                else resolve(rows);
            });
        });
    }

    async searchWithSuggestions(query) {
        // Recherche exacte d'abord
        let results = await this.search(query);

        if (results.length === 0) {
            // Recherche approximative avec wildcards
            const fuzzyQuery = query.split(' ').map(term => `${term}*`).join(' OR ');
            results = await this.search(fuzzyQuery);
        }

        return results;
    }
}
```

## Alternatives modernes aux frameworks présentés

Flask et Express restent solides et largement utilisés, mais l'écosystème 2024-2026 a vu émerger des frameworks **plus rapides, mieux typés, et avec génération automatique de documentation OpenAPI**. À considérer pour un nouveau projet :

### Python — **FastAPI** (alternative recommandée à Flask)

[FastAPI](https://fastapi.tiangolo.com/) combine validation Pydantic, typage Python natif, async/await, et **OpenAPI auto-généré** :

```python
# pip install fastapi uvicorn[standard] sqlmodel
from fastapi import FastAPI, HTTPException  
from sqlmodel import SQLModel, Field, Session, create_engine, select  

# ⚠️ PIÈGE SQLModel important : un modèle avec `table=True` **ne valide
#    PAS les champs comme requis** côté Pydantic — un POST avec un body
#    incomplet passe la validation et explose plus tard avec une
#    IntegrityError SQLite. La pratique recommandée est de séparer :
#
#    1. PersonneBase   : champs partagés (validation Pydantic stricte)
#    2. Personne       : modèle BDD (hérite de Base + table=True)
#    3. PersonneCreate : modèle d'entrée API (hérite de Base, sans id)
#    4. PersonneRead   : modèle de sortie API (hérite de Base + id)

class PersonneBase(SQLModel):
    nom: str
    age: int | None = None
    email: str = Field(unique=True)

class Personne(PersonneBase, table=True):
    id: int | None = Field(default=None, primary_key=True)

class PersonneCreate(PersonneBase):
    pass  # mêmes champs, mais SANS table=True → Pydantic valide bien les champs requis

class PersonneRead(PersonneBase):
    id: int

engine = create_engine("sqlite:///app.db")  
SQLModel.metadata.create_all(engine)  

app = FastAPI()

@app.post("/api/personnes", response_model=PersonneRead, status_code=201)
def creer(personne_in: PersonneCreate):
    # Conversion modèle d'entrée → modèle BDD
    personne = Personne.model_validate(personne_in)
    with Session(engine) as session:
        session.add(personne)
        session.commit()
        session.refresh(personne)
        return personne

@app.get("/api/personnes/{person_id}", response_model=PersonneRead)
def obtenir(person_id: int):
    with Session(engine) as session:
        personne = session.get(Personne, person_id)
        if not personne:
            raise HTTPException(status_code=404, detail="Personne non trouvée")
        return personne

# Lancement : uvicorn main:app --reload
# Documentation auto : http://localhost:8000/docs (Swagger UI) ou /redoc
```

**Avantages vs Flask** : 5–10× plus rapide (basé sur Starlette/asyncio), validation Pydantic intégrée, doc OpenAPI gratuite, support natif async.

> ⚠️ **Bug fréquent à connaître** : avec `class Personne(SQLModel, table=True)` utilisé directement comme body de requête, **un POST `{}` passe la validation Pydantic** et plante avec `IntegrityError: NOT NULL constraint failed` au niveau SQLite (statut 500). La séparation **Base / Table / Create / Read** ci-dessus garantit une validation 422 propre quand un champ requis manque. Voir la [doc officielle SQLModel](https://sqlmodel.tiangolo.com/tutorial/fastapi/multiple-models/).

### Node.js — **Hono** ou **Fastify** (alternatives à Express)

- **[Hono](https://hono.dev/)** : ultra-léger (~15 ko), compatible Node/Bun/Deno/Cloudflare Workers, type-safe TypeScript natif.
  ```typescript
  import { Hono } from 'hono';
  const app = new Hono();
  app.get('/api/personnes/:id', async (c) => {
      const id = c.req.param('id');
      // ... requête SQLite
      return c.json({ id, nom: 'Alice' });
  });
  export default app;
  ```
- **[Fastify](https://fastify.dev/)** : ~2× plus rapide qu'Express, schemas JSON Schema natifs avec validation, plugins matures.
- **[Elysia](https://elysiajs.com/)** : optimisé pour Bun, très haute performance, type-safety extrême via templates littéraux TypeScript.

### Rust — **Axum** + **rusqlite**

Pour des APIs ultra-performantes (10 000+ req/s sur un seul cœur) :

```rust
// Cargo.toml : axum = "0.8", tokio = { version = "1", features = ["full"] },
//              rusqlite = "0.31", serde = { version = "1", features = ["derive"] }
use axum::{Router, routing::get, Json, extract::Path, http::StatusCode};  
use serde::Serialize;  

#[derive(Serialize)]
struct Personne { id: i64, nom: String, age: Option<i32> }

// ⚠️ Axum 0.8 utilise la syntaxe `{id}` pour les paramètres de route
//    (au lieu de `:id` dans les versions ≤ 0.7). Vérifiez votre version.
async fn obtenir_personne(Path(id): Path<i64>) -> Result<Json<Personne>, StatusCode> {
    // ... requête rusqlite (omise pour la concision)
    Ok(Json(Personne { id, nom: "Alice".into(), age: Some(25) }))
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/api/personnes/{id}", get(obtenir_personne));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### Go — **Chi** ou **Gin**

```go
// go get github.com/go-chi/chi/v5
r := chi.NewRouter()  
r.Get("/api/personnes/{id}", func(w http.ResponseWriter, r *http.Request) {  
    id := chi.URLParam(r, "id")
    // ... requête database/sql
})
http.ListenAndServe(":3000", r)
```

### Récapitulatif

| Framework | Langage | Forces | Quand l'utiliser |
|---|---|---|---|
| **Flask** | Python | Mature, énorme écosystème, simple | Projets existants, équipes Python |
| **FastAPI** | Python | Async natif, OpenAPI auto, validation Pydantic | **Nouveau projet Python** |
| **Express** | Node | Standard de facto, simple | Projets existants, prototypes rapides |
| **Hono** | TS / Bun / CF | Edge-ready, type-safe, ultra-léger | **Edge functions, Workers, projet récent** |
| **Fastify** | Node | Performance + schémas JSON | Backend production critique |
| **Axum** | Rust | Performance maximale, sûreté mémoire | Latence < 1 ms, charge élevée |
| **Gin / Chi** | Go | Simplicité, déploiement facile | Microservices, container-friendly |

## Conclusion finale

### Ce que vous avez appris

Dans ce tutoriel complet, vous avez découvert :

1. **Les fondamentaux des APIs REST** avec une approche pratique
2. **L'intégration de SQLite** dans des applications web modernes
3. **La sécurisation d'APIs** avec authentification, validation et protection
4. **L'optimisation des performances** avec cache, compression et monitoring
5. **Le déploiement en production** avec Docker, Nginx et scripts automatisés
6. **La maintenance et surveillance** d'applications en production

### Prochaines étapes recommandées

**Pour les débutants :**
1. Pratiquez en créant une API simple (gestion de tâches, carnet d'adresses)
2. Ajoutez progressivement la sécurité et la validation
3. Apprenez à utiliser Postman ou un autre client REST
4. Explorez la documentation Swagger générée

**Pour le niveau intermédiaire :**
1. Implémentez des patterns architecturaux (Repository, CQRS)
2. Ajoutez des tests automatisés complets
3. Configurez un pipeline CI/CD
4. Explorez les WebSockets pour le temps réel

**Pour le niveau avancé :**
1. Migrez vers PostgreSQL ou MongoDB
2. Implémentez le Event Sourcing
3. Créez une architecture microservices
4. Ajoutez l'observabilité (tracing, métriques)

### Ressources communautaires

**Forums et communautés :**
- **Stack Overflow** : [sqlite] et [express] tags
- **Reddit** : r/webdev, r/node, r/flask
- **Discord** : Serveurs Express.js et SQLite
- **GitHub** : Projets open source à étudier

**Blogs et tutoriels :**
- **MDN Web Docs** : APIs REST et sécurité web
- **Dev.to** : Articles communautaires sur SQLite et APIs
- **Medium** : Tutoriels avancés sur l'architecture

**Outils recommandés :**
- **Swagger Editor** : Conception d'APIs
- **Postman Collections** : Tests et documentation
- **GitHub Actions** : CI/CD automatisé
- **Grafana** : Monitoring et dashboards

### Philosophie du développement

Gardez à l'esprit ces principes essentiels :

1. **Simplicité d'abord** : Commencez simple, ajoutez la complexité progressivement
2. **Sécurité par design** : Intégrez la sécurité dès le début, pas après
3. **Tests continuels** : Testez chaque fonctionnalité dès qu'elle est développée
4. **Documentation vivante** : Maintenez la documentation à jour avec le code
5. **Monitoring permanent** : Surveillez vos applications en production

SQLite et les APIs REST forment une combinaison puissante pour de nombreux projets. Avec les connaissances acquises dans ce tutoriel, vous êtes maintenant équipé pour créer des applications robustes, sécurisées et performantes.

**Bonne programmation ! 🚀**

---

*Ce tutoriel représente un guide complet mais n'est qu'un point de départ. La technologie évoluant constamment, continuez à apprendre, expérimenter et partager vos connaissances avec la communauté.*

⏭️
