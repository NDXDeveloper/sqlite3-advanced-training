🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.2 Gestion des permissions et contrôle d'accès

## Introduction au contrôle d'accès SQLite

Contrairement aux bases de données serveur comme MySQL ou PostgreSQL qui ont des systèmes d'utilisateurs intégrés, SQLite n'a **pas de gestion native des utilisateurs**. Cela signifie que vous devez implémenter votre propre système de contrôle d'accès au niveau de votre application.

### Pourquoi gérer les permissions ?

**Scénarios nécessitant un contrôle d'accès :**
- Application multi-utilisateurs (web, desktop)
- Différents niveaux d'autorisation (admin, utilisateur, invité)
- Séparation des données sensibles
- Conformité aux réglementations (RGPD, sécurité des données)
- Protection contre les accès non autorisés

**Ce que nous allons apprendre :**
- Comprendre les limitations de SQLite
- Implémenter un système d'authentification
- Créer des niveaux de permissions
- Sécuriser l'accès aux données
- Gérer les sessions utilisateur

## Comprendre les limitations SQLite

### Ce que SQLite NE peut PAS faire nativement

```sql
-- Ces commandes n'existent PAS dans SQLite :
CREATE USER 'utilisateur'@'localhost' IDENTIFIED BY 'password';  -- ❌
GRANT SELECT ON table TO utilisateur;                            -- ❌
REVOKE INSERT ON table FROM utilisateur;                         -- ❌
```

### Comment SQLite gère la sécurité

SQLite s'appuie sur :
1. **Sécurité du système de fichiers** (permissions OS)
2. **Sécurité de l'application** (votre code)
3. **Chiffrement** (vu dans la section précédente)

```bash
# Exemple de permissions au niveau OS
chmod 600 ma_base.db          # Seul le propriétaire peut lire/écrire
chmod 644 ma_base.db          # Propriétaire: lecture/écriture, autres: lecture seule
```

## Conception d'un système d'authentification

### Structure de base pour gérer les utilisateurs

Commençons par créer les tables nécessaires :

```sql
-- Table des utilisateurs
CREATE TABLE utilisateurs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom_utilisateur TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE NOT NULL,
    mot_de_passe_hash TEXT NOT NULL,
    sel TEXT NOT NULL,                    -- Salt pour sécuriser le hash
    role TEXT NOT NULL DEFAULT 'utilisateur',
    actif BOOLEAN DEFAULT 1,
    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
    derniere_connexion DATETIME
);

-- Table des rôles et permissions
CREATE TABLE roles (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    description TEXT
);

-- Table des permissions
CREATE TABLE permissions (
    id INTEGER PRIMARY KEY,
    nom TEXT UNIQUE NOT NULL,
    description TEXT,
    table_cible TEXT,                     -- Quelle table est concernée
    operation TEXT                        -- SELECT, INSERT, UPDATE, DELETE
);

-- Table de liaison rôles-permissions
CREATE TABLE role_permissions (
    role_id INTEGER,
    permission_id INTEGER,
    PRIMARY KEY (role_id, permission_id),
    FOREIGN KEY (role_id) REFERENCES roles(id),
    FOREIGN KEY (permission_id) REFERENCES permissions(id)
);

-- Table des sessions (optionnel, pour les apps web)
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    utilisateur_id INTEGER NOT NULL,
    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP,
    date_expiration DATETIME NOT NULL,
    adresse_ip TEXT,
    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs(id)
);
```

### Initialisation des rôles de base

```sql
-- Créer les rôles principaux
INSERT INTO roles (nom, description) VALUES
    ('admin', 'Administrateur - accès complet'),
    ('utilisateur', 'Utilisateur standard - accès limité'),
    ('invite', 'Invité - lecture seule');

-- Créer des permissions de base
INSERT INTO permissions (nom, description, table_cible, operation) VALUES
    ('lire_tous_utilisateurs', 'Peut voir tous les utilisateurs', 'utilisateurs', 'SELECT'),
    ('modifier_utilisateurs', 'Peut modifier les utilisateurs', 'utilisateurs', 'UPDATE'),
    ('supprimer_utilisateurs', 'Peut supprimer des utilisateurs', 'utilisateurs', 'DELETE'),
    ('lire_donnees', 'Peut lire les données métier', '*', 'SELECT'),
    ('modifier_donnees', 'Peut modifier les données métier', '*', 'UPDATE');

-- Assigner les permissions aux rôles
-- Admin : toutes les permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id
FROM roles r, permissions p
WHERE r.nom = 'admin';

-- Utilisateur : lecture et modification des données
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id
FROM roles r, permissions p
WHERE r.nom = 'utilisateur'
AND p.nom IN ('lire_donnees', 'modifier_donnees');

-- Invité : lecture seule
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id
FROM roles r, permissions p
WHERE r.nom = 'invite'
AND p.nom = 'lire_donnees';
```

## Implémentation en Python

### Classe de gestion des utilisateurs

```python
import sqlite3
import hashlib
import secrets
import datetime
from typing import Optional, List

class GestionnaireUtilisateurs:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base
        self.utilisateur_actuel = None

    def _generer_sel(self) -> str:
        """Génère un sel aléatoire pour sécuriser le mot de passe"""
        return secrets.token_hex(32)

    def _hasher_mot_de_passe(self, mot_de_passe: str, sel: str) -> str:
        """Hash un mot de passe avec un sel"""
        return hashlib.pbkdf2_hmac('sha256',
                                   mot_de_passe.encode('utf-8'),
                                   sel.encode('utf-8'),
                                   100000).hex()

    def creer_utilisateur(self, nom_utilisateur: str, email: str,
                         mot_de_passe: str, role: str = 'utilisateur') -> bool:
        """Crée un nouvel utilisateur"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Vérifier si l'utilisateur existe déjà
            cursor.execute("SELECT id FROM utilisateurs WHERE nom_utilisateur = ? OR email = ?",
                          (nom_utilisateur, email))
            if cursor.fetchone():
                print("❌ Utilisateur ou email déjà existant")
                return False

            # Générer le sel et hasher le mot de passe
            sel = self._generer_sel()
            mot_de_passe_hash = self._hasher_mot_de_passe(mot_de_passe, sel)

            # Insérer le nouvel utilisateur
            cursor.execute("""
                INSERT INTO utilisateurs (nom_utilisateur, email, mot_de_passe_hash, sel, role)
                VALUES (?, ?, ?, ?, ?)
            """, (nom_utilisateur, email, mot_de_passe_hash, sel, role))

            conn.commit()
            conn.close()

            print(f"✅ Utilisateur '{nom_utilisateur}' créé avec succès")
            return True

        except sqlite3.Error as e:
            print(f"❌ Erreur lors de la création : {e}")
            return False

    def authentifier(self, nom_utilisateur: str, mot_de_passe: str) -> bool:
        """Authentifie un utilisateur"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Récupérer les infos de l'utilisateur
            cursor.execute("""
                SELECT id, mot_de_passe_hash, sel, role, actif
                FROM utilisateurs
                WHERE nom_utilisateur = ?
            """, (nom_utilisateur,))

            resultat = cursor.fetchone()
            if not resultat:
                print("❌ Utilisateur non trouvé")
                return False

            user_id, mot_de_passe_stocke, sel, role, actif = resultat

            # Vérifier si le compte est actif
            if not actif:
                print("❌ Compte désactivé")
                return False

            # Vérifier le mot de passe
            mot_de_passe_hash = self._hasher_mot_de_passe(mot_de_passe, sel)
            if mot_de_passe_hash != mot_de_passe_stocke:
                print("❌ Mot de passe incorrect")
                return False

            # Authentification réussie
            self.utilisateur_actuel = {
                'id': user_id,
                'nom_utilisateur': nom_utilisateur,
                'role': role
            }

            # Mettre à jour la dernière connexion
            cursor.execute("""
                UPDATE utilisateurs
                SET derniere_connexion = CURRENT_TIMESTAMP
                WHERE id = ?
            """, (user_id,))

            conn.commit()
            conn.close()

            print(f"✅ Connexion réussie pour {nom_utilisateur} ({role})")
            return True

        except sqlite3.Error as e:
            print(f"❌ Erreur d'authentification : {e}")
            return False

    def a_permission(self, permission_requise: str) -> bool:
        """Vérifie si l'utilisateur actuel a une permission"""
        if not self.utilisateur_actuel:
            return False

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Vérifier la permission via le rôle
            cursor.execute("""
                SELECT COUNT(*) FROM role_permissions rp
                JOIN roles r ON rp.role_id = r.id
                JOIN permissions p ON rp.permission_id = p.id
                WHERE r.nom = ? AND p.nom = ?
            """, (self.utilisateur_actuel['role'], permission_requise))

            resultat = cursor.fetchone()[0]
            conn.close()

            return resultat > 0

        except sqlite3.Error:
            return False

    def obtenir_permissions(self) -> List[str]:
        """Retourne la liste des permissions de l'utilisateur actuel"""
        if not self.utilisateur_actuel:
            return []

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("""
                SELECT p.nom FROM role_permissions rp
                JOIN roles r ON rp.role_id = r.id
                JOIN permissions p ON rp.permission_id = p.id
                WHERE r.nom = ?
            """, (self.utilisateur_actuel['role'],))

            permissions = [row[0] for row in cursor.fetchall()]
            conn.close()

            return permissions

        except sqlite3.Error:
            return []

    def deconnexion(self):
        """Déconnecte l'utilisateur actuel"""
        if self.utilisateur_actuel:
            print(f"👋 Déconnexion de {self.utilisateur_actuel['nom_utilisateur']}")
            self.utilisateur_actuel = None
        else:
            print("Aucun utilisateur connecté")
```

### Utilisation de base

```python
# Initialiser le gestionnaire
gestionnaire = GestionnaireUtilisateurs('ma_base.db')

# Créer des utilisateurs de test
gestionnaire.creer_utilisateur('admin', 'admin@email.com', 'password123', 'admin')
gestionnaire.creer_utilisateur('alice', 'alice@email.com', 'motdepasse456', 'utilisateur')
gestionnaire.creer_utilisateur('bob', 'bob@email.com', 'secret789', 'invite')

# Test d'authentification
if gestionnaire.authentifier('alice', 'motdepasse456'):
    print("Connexion réussie !")

    # Vérifier les permissions
    print("Permissions disponibles :", gestionnaire.obtenir_permissions())

    if gestionnaire.a_permission('lire_donnees'):
        print("✅ Peut lire les données")

    if gestionnaire.a_permission('modifier_utilisateurs'):
        print("✅ Peut modifier les utilisateurs")
    else:
        print("❌ Ne peut pas modifier les utilisateurs")

# Déconnexion
gestionnaire.deconnexion()
```

## Décorateurs pour protéger les fonctions

### Système de décorateurs Python

```python
from functools import wraps

def necessite_permission(permission_requise):
    """Décorateur pour protéger une fonction avec une permission"""
    def decorateur(func):
        @wraps(func)
        def wrapper(self, *args, **kwargs):
            if not hasattr(self, 'gestionnaire_utilisateurs'):
                raise Exception("Gestionnaire d'utilisateurs non initialisé")

            if not self.gestionnaire_utilisateurs.a_permission(permission_requise):
                raise PermissionError(f"Permission '{permission_requise}' requise")

            return func(self, *args, **kwargs)
        return wrapper
    return decorateur

def necessite_connexion(func):
    """Décorateur pour s'assurer qu'un utilisateur est connecté"""
    @wraps(func)
    def wrapper(self, *args, **kwargs):
        if not hasattr(self, 'gestionnaire_utilisateurs'):
            raise Exception("Gestionnaire d'utilisateurs non initialisé")

        if not self.gestionnaire_utilisateurs.utilisateur_actuel:
            raise Exception("Connexion requise")

        return func(self, *args, **kwargs)
    return wrapper

# Exemple d'application protégée
class ApplicationMetier:
    def __init__(self, chemin_base):
        self.gestionnaire_utilisateurs = GestionnaireUtilisateurs(chemin_base)
        self.conn = sqlite3.connect(chemin_base)

    @necessite_connexion
    @necessite_permission('lire_donnees')
    def lire_donnees_clients(self):
        """Lecture des données clients (nécessite permission)"""
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM clients")
        return cursor.fetchall()

    @necessite_connexion
    @necessite_permission('modifier_donnees')
    def modifier_client(self, client_id, nouveaux_data):
        """Modification d'un client (nécessite permission)"""
        cursor = self.conn.cursor()
        cursor.execute("UPDATE clients SET nom = ? WHERE id = ?",
                      (nouveaux_data['nom'], client_id))
        self.conn.commit()
        print(f"Client {client_id} modifié")

    @necessite_connexion
    @necessite_permission('lire_tous_utilisateurs')
    def lister_utilisateurs(self):
        """Liste tous les utilisateurs (admin seulement)"""
        cursor = self.conn.cursor()
        cursor.execute("SELECT nom_utilisateur, email, role FROM utilisateurs")
        return cursor.fetchall()

# Utilisation
app = ApplicationMetier('ma_base.db')

# Connexion comme utilisateur normal
if app.gestionnaire_utilisateurs.authentifier('alice', 'motdepasse456'):
    try:
        # Ceci fonctionne (alice a la permission lire_donnees)
        clients = app.lire_donnees_clients()
        print("Données clients récupérées")

        # Ceci fonctionne aussi (alice a permission modifier_donnees)
        app.modifier_client(1, {'nom': 'Nouveau nom'})

        # Ceci va échouer (alice n'est pas admin)
        utilisateurs = app.lister_utilisateurs()

    except PermissionError as e:
        print(f"❌ Accès refusé : {e}")
```

## Gestion des sessions (pour applications web)

### Système de sessions

```python
import uuid
from datetime import datetime, timedelta

class GestionnaireSession:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base

    def creer_session(self, utilisateur_id: int, duree_heures: int = 24) -> str:
        """Crée une nouvelle session pour un utilisateur"""
        session_id = str(uuid.uuid4())
        expiration = datetime.now() + timedelta(hours=duree_heures)

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("""
                INSERT INTO sessions (id, utilisateur_id, date_expiration)
                VALUES (?, ?, ?)
            """, (session_id, utilisateur_id, expiration))

            conn.commit()
            conn.close()

            return session_id

        except sqlite3.Error as e:
            print(f"❌ Erreur création session : {e}")
            return None

    def valider_session(self, session_id: str) -> Optional[dict]:
        """Valide une session et retourne les infos utilisateur"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("""
                SELECT u.id, u.nom_utilisateur, u.role, s.date_expiration
                FROM sessions s
                JOIN utilisateurs u ON s.utilisateur_id = u.id
                WHERE s.id = ? AND s.date_expiration > CURRENT_TIMESTAMP
            """, (session_id,))

            resultat = cursor.fetchone()
            conn.close()

            if resultat:
                return {
                    'id': resultat[0],
                    'nom_utilisateur': resultat[1],
                    'role': resultat[2],
                    'expiration': resultat[3]
                }
            return None

        except sqlite3.Error:
            return None

    def supprimer_session(self, session_id: str):
        """Supprime une session (déconnexion)"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("DELETE FROM sessions WHERE id = ?", (session_id,))

            conn.commit()
            conn.close()

        except sqlite3.Error as e:
            print(f"❌ Erreur suppression session : {e}")

    def nettoyer_sessions_expirees(self):
        """Supprime les sessions expirées"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("DELETE FROM sessions WHERE date_expiration <= CURRENT_TIMESTAMP")
            sessions_supprimees = cursor.rowcount

            conn.commit()
            conn.close()

            print(f"🧹 {sessions_supprimees} sessions expirées supprimées")

        except sqlite3.Error as e:
            print(f"❌ Erreur nettoyage : {e}")

# Exemple d'utilisation avec Flask (web)
"""
from flask import Flask, request, session, jsonify

app = Flask(__name__)
gestionnaire_session = GestionnaireSession('ma_base.db')

@app.route('/login', methods=['POST'])
def login():
    nom_utilisateur = request.json['nom_utilisateur']
    mot_de_passe = request.json['mot_de_passe']

    gestionnaire = GestionnaireUtilisateurs('ma_base.db')
    if gestionnaire.authentifier(nom_utilisateur, mot_de_passe):
        session_id = gestionnaire_session.creer_session(gestionnaire.utilisateur_actuel['id'])
        return jsonify({'session_id': session_id, 'statut': 'connecte'})
    else:
        return jsonify({'erreur': 'Authentification échouée'}), 401

@app.route('/protected')
def protected():
    session_id = request.headers.get('Authorization')
    if not session_id:
        return jsonify({'erreur': 'Session requise'}), 401

    utilisateur = gestionnaire_session.valider_session(session_id)
    if not utilisateur:
        return jsonify({'erreur': 'Session invalide'}), 401

    return jsonify({'message': f'Bonjour {utilisateur["nom_utilisateur"]}'})
"""
```

## Contrôle d'accès au niveau des données

### Filtrage par propriétaire

```python
class AccesSecurise:
    def __init__(self, chemin_base, gestionnaire_utilisateurs):
        self.chemin_base = chemin_base
        self.gestionnaire = gestionnaire_utilisateurs

    def lire_mes_documents(self):
        """Lit seulement les documents de l'utilisateur connecté"""
        if not self.gestionnaire.utilisateur_actuel:
            raise Exception("Connexion requise")

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Filtrer par propriétaire
        cursor.execute("""
            SELECT id, titre, contenu, date_creation
            FROM documents
            WHERE proprietaire_id = ?
        """, (self.gestionnaire.utilisateur_actuel['id'],))

        documents = cursor.fetchall()
        conn.close()

        return documents

    def modifier_document(self, document_id, nouveau_contenu):
        """Modifie un document seulement si l'utilisateur en est propriétaire"""
        if not self.gestionnaire.utilisateur_actuel:
            raise Exception("Connexion requise")

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Vérifier la propriété
        cursor.execute("""
            SELECT proprietaire_id FROM documents WHERE id = ?
        """, (document_id,))

        resultat = cursor.fetchone()
        if not resultat:
            raise Exception("Document non trouvé")

        proprietaire_id = resultat[0]
        utilisateur_id = self.gestionnaire.utilisateur_actuel['id']

        # Vérifier les droits (propriétaire ou admin)
        if (proprietaire_id != utilisateur_id and
            not self.gestionnaire.a_permission('modifier_tous_documents')):
            raise PermissionError("Vous ne pouvez modifier que vos propres documents")

        # Effectuer la modification
        cursor.execute("""
            UPDATE documents
            SET contenu = ?, date_modification = CURRENT_TIMESTAMP
            WHERE id = ?
        """, (nouveau_contenu, document_id))

        conn.commit()
        conn.close()

        print(f"✅ Document {document_id} modifié")
```

### Vues avec sécurité intégrée

```sql
-- Vue pour utilisateurs normaux (seulement leurs données)
CREATE VIEW mes_commandes AS
SELECT c.id, c.date_commande, c.total, c.statut
FROM commandes c
WHERE c.client_id = (
    -- Cette partie devrait être remplacée par l'ID de l'utilisateur connecté
    SELECT id FROM utilisateurs WHERE nom_utilisateur = current_user()
);

-- Vue pour admins (toutes les données)
CREATE VIEW toutes_commandes AS
SELECT c.id, c.date_commande, c.total, c.statut,
       u.nom_utilisateur as client
FROM commandes c
JOIN utilisateurs u ON c.client_id = u.id;
```

## Audit et logging des accès

### Système d'audit

```python
import json
from datetime import datetime

class AuditLogger:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base
        self._creer_table_audit()

    def _creer_table_audit(self):
        """Crée la table d'audit si elle n'existe pas"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS audit_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                utilisateur_id INTEGER,
                action TEXT NOT NULL,
                table_cible TEXT,
                enregistrement_id INTEGER,
                anciennes_valeurs TEXT,  -- JSON
                nouvelles_valeurs TEXT,  -- JSON
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                adresse_ip TEXT,
                user_agent TEXT
            )
        """)

        conn.commit()
        conn.close()

    def enregistrer_action(self, utilisateur_id, action, table_cible=None,
                          enregistrement_id=None, anciennes_valeurs=None,
                          nouvelles_valeurs=None, adresse_ip=None):
        """Enregistre une action dans le log d'audit"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO audit_log
            (utilisateur_id, action, table_cible, enregistrement_id,
             anciennes_valeurs, nouvelles_valeurs, adresse_ip)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, (
            utilisateur_id, action, table_cible, enregistrement_id,
            json.dumps(anciennes_valeurs) if anciennes_valeurs else None,
            json.dumps(nouvelles_valeurs) if nouvelles_valeurs else None,
            adresse_ip
        ))

        conn.commit()
        conn.close()

    def lire_audit(self, utilisateur_id=None, limite=100):
        """Lit les entrées d'audit"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        if utilisateur_id:
            cursor.execute("""
                SELECT timestamp, action, table_cible, anciennes_valeurs, nouvelles_valeurs
                FROM audit_log
                WHERE utilisateur_id = ?
                ORDER BY timestamp DESC
                LIMIT ?
            """, (utilisateur_id, limite))
        else:
            cursor.execute("""
                SELECT u.nom_utilisateur, al.timestamp, al.action, al.table_cible
                FROM audit_log al
                LEFT JOIN utilisateurs u ON al.utilisateur_id = u.id
                ORDER BY al.timestamp DESC
                LIMIT ?
            """, (limite,))

        resultats = cursor.fetchall()
        conn.close()

        return resultats

# Utilisation avec les opérations
class GestionnaireSecurise:
    def __init__(self, chemin_base, gestionnaire_utilisateurs):
        self.chemin_base = chemin_base
        self.gestionnaire = gestionnaire_utilisateurs
        self.audit = AuditLogger(chemin_base)

    def modifier_utilisateur_avec_audit(self, user_id, nouveaux_donnees):
        """Modifie un utilisateur en enregistrant l'audit"""
        if not self.gestionnaire.a_permission('modifier_utilisateurs'):
            raise PermissionError("Permission insuffisante")

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Récupérer les anciennes valeurs
        cursor.execute("SELECT nom_utilisateur, email, role FROM utilisateurs WHERE id = ?", (user_id,))
        ancien = cursor.fetchone()
        if not ancien:
            raise Exception("Utilisateur non trouvé")

        anciennes_valeurs = {
            'nom_utilisateur': ancien[0],
            'email': ancien[1],
            'role': ancien[2]
        }

        # Effectuer la modification
        cursor.execute("""
            UPDATE utilisateurs
            SET nom_utilisateur = ?, email = ?, role = ?
            WHERE id = ?
        """, (nouveaux_donnees['nom_utilisateur'], nouveaux_donnees['email'],
              nouveaux_donnees['role'], user_id))

        conn.commit()
        conn.close()

        # Enregistrer dans l'audit
        self.audit.enregistrer_action(
            utilisateur_id=self.gestionnaire.utilisateur_actuel['id'],
            action='UPDATE_USER',
            table_cible='utilisateurs',
            enregistrement_id=user_id,
            anciennes_valeurs=anciennes_valeurs,
            nouvelles_valeurs=nouveaux_donnees
        )

        print(f"✅ Utilisateur {user_id} modifié (action auditée)")
```

## Sécurité avancée

### Protection contre l'injection SQL

```python
class RequeteSecurisee:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base

    def rechercher_utilisateurs_securise(self, terme_recherche):
        """Recherche sécurisée avec paramètres liés"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # ✅ CORRECT : Utilisation de paramètres liés
        cursor.execute("""
            SELECT nom_utilisateur, email
            FROM utilisateurs
            WHERE nom_utilisateur LIKE ? OR email LIKE ?
        """, (f'%{terme_recherche}%', f'%{terme_recherche}%'))

        resultats = cursor.fetchall()
        conn.close()
        return resultats

    def rechercher_utilisateurs_dangereux(self, terme_recherche):
        """❌ DANGEREUX : Vulnérable à l'injection SQL"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # ❌ JAMAIS faire cela :
        requete = f"""
            SELECT nom_utilisateur, email
            FROM utilisateurs
            WHERE nom_utilisateur LIKE '%{terme_recherche}%'
        """
        cursor.execute(requete)  # Vulnérable !

        resultats = cursor.fetchall()
        conn.close()
        return resultats

# Test de sécurité
recherche = RequeteSecurisee('ma_base.db')

# Test normal
resultats = recherche.rechercher_utilisateurs_securise('alice')
print("Résultats normaux :", resultats)

# Test d'injection (sera bloqué avec la méthode sécurisée)
terme_malveillant = "'; DROP TABLE utilisateurs; --"
try:
    # Avec la méthode sécurisée, ceci sera traité comme un terme de recherche normal
    resultats = recherche.rechercher_utilisateurs_securise(terme_malveillant)
    print("✅ Injection bloquée, recherche traitée normalement")
except Exception as e:
    print(f"Erreur : {e}")
```

### Validation et assainissement des entrées

```python
import re
from typing import Union

class ValidateurEntrees:
    @staticmethod
    def valider_nom_utilisateur(nom_utilisateur: str) -> bool:
        """Valide un nom d'utilisateur (alphanumériques + underscore, 3-20 caractères)"""
        pattern = r'^[a-zA-Z0-9_]{3,20}$'
        return bool(re.match(pattern, nom_utilisateur))

    @staticmethod
    def valider_email(email: str) -> bool:
        """Valide un email avec une regex basique"""
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))

    @staticmethod
    def valider_mot_de_passe(mot_de_passe: str) -> tuple[bool, str]:
        """Valide un mot de passe et retourne le résultat + message"""
        if len(mot_de_passe) < 8:
            return False, "Le mot de passe doit contenir au moins 8 caractères"

        if not re.search(r'[A-Z]', mot_de_passe):
            return False, "Le mot de passe doit contenir au moins une majuscule"

        if not re.search(r'[a-z]', mot_de_passe):
            return False, "Le mot de passe doit contenir au moins une minuscule"

        if not re.search(r'\d', mot_de_passe):
            return False, "Le mot de passe doit contenir au moins un chiffre"

        if not re.search(r'[!@#$%^&*(),.?":{}|<>]', mot_de_passe):
            return False, "Le mot de passe doit contenir au moins un caractère spécial"

        return True, "Mot de passe valide"

    @staticmethod
    def assainir_entree(texte: str, max_longueur: int = 255) -> str:
        """Assainit une entrée texte"""
        if not isinstance(texte, str):
            texte = str(texte)

        # Supprimer les caractères dangereux
        texte = re.sub(r'[<>"\';\\]', '', texte)

        # Limiter la longueur
        texte = texte[:max_longueur]

        # Supprimer les espaces en début/fin
        texte = texte.strip()

        return texte

# Gestionnaire d'utilisateurs avec validation renforcée
class GestionnaireUtilisateursSecurise(GestionnaireUtilisateurs):
    def creer_utilisateur(self, nom_utilisateur: str, email: str,
                         mot_de_passe: str, role: str = 'utilisateur') -> bool:
        """Version sécurisée de création d'utilisateur avec validation"""

        # Validation des entrées
        if not ValidateurEntrees.valider_nom_utilisateur(nom_utilisateur):
            print("❌ Nom d'utilisateur invalide (3-20 caractères alphanumériques)")
            return False

        if not ValidateurEntrees.valider_email(email):
            print("❌ Adresse email invalide")
            return False

        valide, message = ValidateurEntrees.valider_mot_de_passe(mot_de_passe)
        if not valide:
            print(f"❌ {message}")
            return False

        # Vérifier que le rôle est valide
        roles_autorises = ['admin', 'utilisateur', 'invite']
        if role not in roles_autorises:
            print(f"❌ Rôle invalide. Rôles autorisés : {roles_autorises}")
            return False

        # Assainir les entrées
        nom_utilisateur = ValidateurEntrees.assainir_entree(nom_utilisateur)
        email = ValidateurEntrees.assainir_entree(email)

        # Appeler la méthode parent avec les données validées
        return super().creer_utilisateur(nom_utilisateur, email, mot_de_passe, role)

# Exemple d'utilisation avec validation
gestionnaire_securise = GestionnaireUtilisateursSecurise('ma_base.db')

# Tests de validation
test_cases = [
    ('alice123', 'alice@email.com', 'MotDePasse123!', 'utilisateur'),  # ✅ Valide
    ('ab', 'alice@email.com', 'MotDePasse123!', 'utilisateur'),        # ❌ Nom trop court
    ('alice123', 'email_invalide', 'MotDePasse123!', 'utilisateur'),   # ❌ Email invalide
    ('alice123', 'alice@email.com', '123', 'utilisateur'),             # ❌ Mot de passe faible
    ('alice123', 'alice@email.com', 'MotDePasse123!', 'hacker'),       # ❌ Rôle invalide
]

for nom, email, mdp, role in test_cases:
    print(f"\nTest : {nom}, {email}, {'*' * len(mdp)}, {role}")
    gestionnaire_securise.creer_utilisateur(nom, email, mdp, role)
```

## Protection contre les attaques par force brute

### Système de limitation des tentatives

```python
import time
from collections import defaultdict
from datetime import datetime, timedelta

class ProtectionForceBrute:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base
        self.tentatives = defaultdict(list)  # IP -> liste des tentatives
        self.comptes_bloques = {}  # nom_utilisateur -> timestamp de déblocage
        self._creer_table_tentatives()

    def _creer_table_tentatives(self):
        """Crée la table pour tracker les tentatives de connexion"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS tentatives_connexion (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nom_utilisateur TEXT,
                adresse_ip TEXT,
                succes BOOLEAN,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        """)

        conn.commit()
        conn.close()

    def enregistrer_tentative(self, nom_utilisateur: str, adresse_ip: str, succes: bool):
        """Enregistre une tentative de connexion"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO tentatives_connexion (nom_utilisateur, adresse_ip, succes)
            VALUES (?, ?, ?)
        """, (nom_utilisateur, adresse_ip, succes))

        conn.commit()
        conn.close()

        # Mettre à jour le cache en mémoire
        maintenant = datetime.now()
        self.tentatives[adresse_ip].append(maintenant)

        # Nettoyer les anciennes tentatives (garder seulement les 15 dernières minutes)
        seuil = maintenant - timedelta(minutes=15)
        self.tentatives[adresse_ip] = [
            t for t in self.tentatives[adresse_ip] if t > seuil
        ]

    def est_bloque_par_ip(self, adresse_ip: str, limite: int = 5) -> bool:
        """Vérifie si une IP est bloquée pour trop de tentatives"""
        maintenant = datetime.now()
        seuil = maintenant - timedelta(minutes=15)

        # Compter les tentatives récentes
        tentatives_recentes = [t for t in self.tentatives[adresse_ip] if t > seuil]

        return len(tentatives_recentes) >= limite

    def est_compte_bloque(self, nom_utilisateur: str) -> bool:
        """Vérifie si un compte spécifique est temporairement bloqué"""
        if nom_utilisateur in self.comptes_bloques:
            deblocage = self.comptes_bloques[nom_utilisateur]
            if datetime.now() < deblocage:
                return True
            else:
                # Le blocage a expiré
                del self.comptes_bloques[nom_utilisateur]
                return False
        return False

    def bloquer_compte(self, nom_utilisateur: str, duree_minutes: int = 30):
        """Bloque temporairement un compte"""
        deblocage = datetime.now() + timedelta(minutes=duree_minutes)
        self.comptes_bloques[nom_utilisateur] = deblocage
        print(f"🔒 Compte '{nom_utilisateur}' bloqué pendant {duree_minutes} minutes")

    def obtenir_tentatives_echouees_recentes(self, nom_utilisateur: str) -> int:
        """Compte les tentatives échouées récentes pour un utilisateur"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Dernières 15 minutes
        seuil = datetime.now() - timedelta(minutes=15)

        cursor.execute("""
            SELECT COUNT(*) FROM tentatives_connexion
            WHERE nom_utilisateur = ? AND succes = 0 AND timestamp > ?
        """, (nom_utilisateur, seuil))

        count = cursor.fetchone()[0]
        conn.close()

        return count

# Gestionnaire d'utilisateurs avec protection force brute
class GestionnaireUtilisateursProtege(GestionnaireUtilisateursSecurise):
    def __init__(self, chemin_base: str):
        super().__init__(chemin_base)
        self.protection = ProtectionForceBrute(chemin_base)

    def authentifier(self, nom_utilisateur: str, mot_de_passe: str,
                    adresse_ip: str = 'localhost') -> bool:
        """Authentification avec protection contre la force brute"""

        # Vérifier si l'IP est bloquée
        if self.protection.est_bloque_par_ip(adresse_ip):
            print("❌ IP temporairement bloquée pour trop de tentatives")
            return False

        # Vérifier si le compte est bloqué
        if self.protection.est_compte_bloque(nom_utilisateur):
            print("❌ Compte temporairement bloqué")
            return False

        # Tentative d'authentification
        succes = super().authentifier(nom_utilisateur, mot_de_passe)

        # Enregistrer la tentative
        self.protection.enregistrer_tentative(nom_utilisateur, adresse_ip, succes)

        if not succes:
            # Vérifier s'il faut bloquer le compte
            tentatives_echouees = self.protection.obtenir_tentatives_echouees_recentes(nom_utilisateur)
            if tentatives_echouees >= 3:  # 3 échecs = blocage temporaire
                self.protection.bloquer_compte(nom_utilisateur, 30)

        return succes

# Test de protection
gestionnaire_protege = GestionnaireUtilisateursProtege('ma_base.db')

# Simuler des attaques
print("=== Test de protection contre force brute ===")

# Créer un utilisateur de test
gestionnaire_protege.creer_utilisateur('test_user', 'test@email.com', 'MotDePasse123!', 'utilisateur')

# Simuler plusieurs tentatives échouées
for i in range(5):
    print(f"\nTentative {i+1} avec mauvais mot de passe...")
    resultat = gestionnaire_protege.authentifier('test_user', 'mauvais_password', '192.168.1.100')
    if not resultat:
        print(f"Échec de connexion {i+1}")

    time.sleep(0.1)  # Petite pause pour simuler des tentatives espacées

# Tenter avec le bon mot de passe après les échecs
print("\nTentative avec le bon mot de passe...")
resultat = gestionnaire_protege.authentifier('test_user', 'MotDePasse123!', '192.168.1.100')
if resultat:
    print("✅ Connexion réussie")
else:
    print("❌ Connexion bloquée malgré le bon mot de passe")
```

## Contrôle d'accès basé sur les attributs (ABAC)

### Système ABAC avancé

```python
from datetime import datetime, time
import json

class ControleurAccesABAC:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base
        self._creer_tables_abac()

    def _creer_tables_abac(self):
        """Crée les tables pour le contrôle d'accès basé sur les attributs"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Table des politiques d'accès
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS politiques_acces (
                id INTEGER PRIMARY KEY,
                nom TEXT UNIQUE NOT NULL,
                description TEXT,
                regles_json TEXT NOT NULL,  -- Règles en JSON
                actif BOOLEAN DEFAULT 1
            )
        """)

        # Table des attributs utilisateur
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS attributs_utilisateur (
                utilisateur_id INTEGER,
                attribut TEXT,
                valeur TEXT,
                PRIMARY KEY (utilisateur_id, attribut),
                FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs(id)
            )
        """)

        # Table des attributs de ressource
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS attributs_ressource (
                ressource_type TEXT,
                ressource_id INTEGER,
                attribut TEXT,
                valeur TEXT,
                PRIMARY KEY (ressource_type, ressource_id, attribut)
            )
        """)

        conn.commit()
        conn.close()

    def definir_attribut_utilisateur(self, utilisateur_id: int, attribut: str, valeur: str):
        """Définit un attribut pour un utilisateur"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT OR REPLACE INTO attributs_utilisateur (utilisateur_id, attribut, valeur)
            VALUES (?, ?, ?)
        """, (utilisateur_id, attribut, valeur))

        conn.commit()
        conn.close()

    def definir_attribut_ressource(self, ressource_type: str, ressource_id: int,
                                  attribut: str, valeur: str):
        """Définit un attribut pour une ressource"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT OR REPLACE INTO attributs_ressource
            (ressource_type, ressource_id, attribut, valeur)
            VALUES (?, ?, ?, ?)
        """, (ressource_type, ressource_id, attribut, valeur))

        conn.commit()
        conn.close()

    def creer_politique(self, nom: str, description: str, regles: dict):
        """Crée une nouvelle politique d'accès"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            INSERT INTO politiques_acces (nom, description, regles_json)
            VALUES (?, ?, ?)
        """, (nom, description, json.dumps(regles)))

        conn.commit()
        conn.close()

    def obtenir_attributs_utilisateur(self, utilisateur_id: int) -> dict:
        """Récupère tous les attributs d'un utilisateur"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT attribut, valeur FROM attributs_utilisateur
            WHERE utilisateur_id = ?
        """, (utilisateur_id,))

        attributs = dict(cursor.fetchall())
        conn.close()

        return attributs

    def obtenir_attributs_ressource(self, ressource_type: str, ressource_id: int) -> dict:
        """Récupère tous les attributs d'une ressource"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT attribut, valeur FROM attributs_ressource
            WHERE ressource_type = ? AND ressource_id = ?
        """, (ressource_type, ressource_id))

        attributs = dict(cursor.fetchall())
        conn.close()

        return attributs

    def evaluer_acces(self, utilisateur_id: int, action: str,
                     ressource_type: str, ressource_id: int) -> bool:
        """Évalue si l'accès est autorisé selon les politiques ABAC"""

        # Obtenir les attributs
        attributs_user = self.obtenir_attributs_utilisateur(utilisateur_id)
        attributs_ressource = self.obtenir_attributs_ressource(ressource_type, ressource_id)

        # Attributs contextuels (heure, jour, etc.)
        maintenant = datetime.now()
        attributs_contexte = {
            'heure': maintenant.hour,
            'jour_semaine': maintenant.weekday(),  # 0 = lundi
            'timestamp': maintenant.isoformat()
        }

        # Obtenir les politiques actives
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT regles_json FROM politiques_acces WHERE actif = 1
        """)

        politiques = cursor.fetchall()
        conn.close()

        # Évaluer chaque politique
        for politique_row in politiques:
            regles = json.loads(politique_row[0])

            if self._evaluer_regles(regles, action, attributs_user,
                                  attributs_ressource, attributs_contexte):
                return True

        return False

    def _evaluer_regles(self, regles: dict, action: str,
                       attributs_user: dict, attributs_ressource: dict,
                       attributs_contexte: dict) -> bool:
        """Évalue les règles d'une politique"""

        # Vérifier l'action
        if 'actions' in regles and action not in regles['actions']:
            return False

        # Vérifier les conditions utilisateur
        if 'conditions_utilisateur' in regles:
            for condition in regles['conditions_utilisateur']:
                attribut = condition['attribut']
                operateur = condition['operateur']
                valeur_attendue = condition['valeur']

                valeur_utilisateur = attributs_user.get(attribut)

                if not self._evaluer_condition(valeur_utilisateur, operateur, valeur_attendue):
                    return False

        # Vérifier les conditions de ressource
        if 'conditions_ressource' in regles:
            for condition in regles['conditions_ressource']:
                attribut = condition['attribut']
                operateur = condition['operateur']
                valeur_attendue = condition['valeur']

                valeur_ressource = attributs_ressource.get(attribut)

                if not self._evaluer_condition(valeur_ressource, operateur, valeur_attendue):
                    return False

        # Vérifier les conditions contextuelles
        if 'conditions_contexte' in regles:
            for condition in regles['conditions_contexte']:
                attribut = condition['attribut']
                operateur = condition['operateur']
                valeur_attendue = condition['valeur']

                valeur_contexte = attributs_contexte.get(attribut)

                if not self._evaluer_condition(valeur_contexte, operateur, valeur_attendue):
                    return False

        return True

    def _evaluer_condition(self, valeur_actuelle, operateur: str, valeur_attendue) -> bool:
        """Évalue une condition individuelle"""
        if valeur_actuelle is None:
            return False

        try:
            if operateur == 'equals':
                return str(valeur_actuelle) == str(valeur_attendue)
            elif operateur == 'not_equals':
                return str(valeur_actuelle) != str(valeur_attendue)
            elif operateur == 'greater_than':
                return float(valeur_actuelle) > float(valeur_attendue)
            elif operateur == 'less_than':
                return float(valeur_actuelle) < float(valeur_attendue)
            elif operateur == 'in':
                return str(valeur_actuelle) in valeur_attendue
            elif operateur == 'not_in':
                return str(valeur_actuelle) not in valeur_attendue
            elif operateur == 'between':
                min_val, max_val = valeur_attendue
                return min_val <= float(valeur_actuelle) <= max_val
        except (ValueError, TypeError):
            return False

        return False

# Exemple d'utilisation du système ABAC
controleur_abac = ControleurAccesABAC('ma_base.db')

# Définir des attributs utilisateur
controleur_abac.definir_attribut_utilisateur(1, 'departement', 'IT')
controleur_abac.definir_attribut_utilisateur(1, 'niveau_clearance', '3')
controleur_abac.definir_attribut_utilisateur(1, 'anciennete_annees', '5')

# Définir des attributs de ressource
controleur_abac.definir_attribut_ressource('document', 1, 'classification', 'confidentiel')
controleur_abac.definir_attribut_ressource('document', 1, 'departement_proprietaire', 'IT')

# Créer une politique d'accès
politique_it_confidentiel = {
    'actions': ['read', 'update'],
    'conditions_utilisateur': [
        {'attribut': 'departement', 'operateur': 'equals', 'valeur': 'IT'},
        {'attribut': 'niveau_clearance', 'operateur': 'greater_than', 'valeur': '2'}
    ],
    'conditions_ressource': [
        {'attribut': 'classification', 'operateur': 'equals', 'valeur': 'confidentiel'},
        {'attribut': 'departement_proprietaire', 'operateur': 'equals', 'valeur': 'IT'}
    ],
    'conditions_contexte': [
        {'attribut': 'heure', 'operateur': 'between', 'valeur': [8, 18]}  # Heures de bureau
    ]
}

controleur_abac.creer_politique(
    'acces_documents_it_confidentiels',
    'Accès aux documents confidentiels IT pendant les heures de bureau',
    politique_it_confidentiel
)

# Tester l'accès
utilisateur_id = 1
autorise = controleur_abac.evaluer_acces(utilisateur_id, 'read', 'document', 1)

if autorise:
    print("✅ Accès autorisé au document")
else:
    print("❌ Accès refusé au document")
```

## Intégration avec des frameworks web

### Exemple avec Flask

```python
from flask import Flask, request, jsonify, session
from functools import wraps

app = Flask(__name__)
app.secret_key = 'votre-cle-secrete-tres-longue-et-complexe'

# Initialisation des gestionnaires
gestionnaire = GestionnaireUtilisateursProtege('ma_base.db')
controleur_abac = ControleurAccesABAC('ma_base.db')

def connexion_requise(f):
    """Décorateur pour vérifier la connexion"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'utilisateur_id' not in session:
            return jsonify({'erreur': 'Connexion requise'}), 401
        return f(*args, **kwargs)
    return decorated_function

def permission_requise(action, ressource_type=None):
    """Décorateur pour vérifier les permissions ABAC"""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if 'utilisateur_id' not in session:
                return jsonify({'erreur': 'Connexion requise'}), 401

            utilisateur_id = session['utilisateur_id']

            # Pour les ressources spécifiques, récupérer l'ID depuis les arguments
            ressource_id = kwargs.get('ressource_id', request.json.get('ressource_id') if request.json else None)

            if ressource_type and ressource_id:
                autorise = controleur_abac.evaluer_acces(utilisateur_id, action, ressource_type, ressource_id)
            else:
                # Pour les actions générales, utiliser le système de permissions classique
                autorise = gestionnaire.a_permission(action)

            if not autorise:
                return jsonify({'erreur': 'Permission insuffisante'}), 403

            return f(*args, **kwargs)
        return decorated_function
    return decorator

@app.route('/login', methods=['POST'])
def login():
    """Endpoint de connexion"""
    data = request.json
    nom_utilisateur = data.get('nom_utilisateur')
    mot_de_passe = data.get('mot_de_passe')
    adresse_ip = request.remote_addr

    if gestionnaire.authentifier(nom_utilisateur, mot_de_passe, adresse_ip):
        session['utilisateur_id'] = gestionnaire.utilisateur_actuel['id']
        session['nom_utilisateur'] = gestionnaire.utilisateur_actuel['nom_utilisateur']
        session['role'] = gestionnaire.utilisateur_actuel['role']

        return jsonify({
            'message': 'Connexion réussie',
            'utilisateur': session['nom_utilisateur'],
            'role': session['role']
        })
    else:
        return jsonify({'erreur': 'Authentification échouée'}), 401

@app.route('/logout', methods=['POST'])
@connexion_requise
def logout():
    """Endpoint de déconnexion"""
    nom_utilisateur = session.get('nom_utilisateur')
    session.clear()
    return jsonify({'message': f'Déconnexion réussie pour {nom_utilisateur}'})

@app.route('/documents/<int:ressource_id>', methods=['GET'])
@connexion_requise
@permission_requise('read', 'document')
def lire_document(ressource_id):
    """Lecture d'un document avec contrôle ABAC"""
    # Ici, implémenter la logique de lecture du document
    return jsonify({
        'message': f'Document {ressource_id} lu avec succès',
        'contenu': 'Contenu du document...'
    })

@app.route('/documents/<int:ressource_id>', methods=['PUT'])
@connexion_requise
@permission_requise('update', 'document')
def modifier_document(ressource_id):
    """Modification d'un document avec contrôle ABAC"""
    data = request.json
    nouveau_contenu = data.get('contenu')

    # Ici, implémenter la logique de modification
    return jsonify({
        'message': f'Document {ressource_id} modifié avec succès'
    })

@app.route('/admin/utilisateurs', methods=['GET'])
@connexion_requise
@permission_requise('lire_tous_utilisateurs')
def lister_utilisateurs_admin():
    """Liste des utilisateurs (admin seulement)"""
    # Implémenter la liste des utilisateurs
    return jsonify({
        'utilisateurs': [
            {'id': 1, 'nom': 'admin', 'role': 'admin'},
            {'id': 2, 'nom': 'alice', 'role': 'utilisateur'}
        ]
    })

if __name__ == '__main__':
    app.run(debug=True)
```

## Récapitulatif et bonnes pratiques

### Checklist de sécurité complète

```python
class ChecklistSecurite:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base

    def audit_complet(self):
        """Effectue un audit de sécurité complet"""
        print("🔍 === AUDIT DE SÉCURITÉ SQLITE ===\n")

        resultats = {
            'chiffrement': self._verifier_chiffrement(),
            'permissions_fichier': self._verifier_permissions_fichier(),
            'utilisateurs_faibles': self._detecter_mots_de_passe_faibles(),
            'comptes_inactifs': self._detecter_comptes_inactifs(),
            'tentatives_suspectes': self._analyser_tentatives_connexion(),
            'integrite': self._verifier_integrite()
        }

        self._generer_rapport(resultats)

    def _verifier_chiffrement(self):
        """Vérifie si la base est chiffrée"""
        try:
            # Tenter d'ouvrir sans mot de passe
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()
            cursor.execute("SELECT name FROM sqlite_master LIMIT 1")
            conn.close()

            return {
                'statut': '❌ CRITIQUE',
                'message': 'Base de données NON chiffrée',
                'recommandation': 'Activer le chiffrement avec SQLCipher'
            }
        except sqlite3.DatabaseError:
            return {
                'statut': '✅ OK',
                'message': 'Base de données chiffrée ou protégée',
                'recommandation': 'Continuer à utiliser le chiffrement'
            }

    def _verifier_permissions_fichier(self):
        """Vérifie les permissions au niveau du système de fichiers"""
        import os
        import stat

        try:
            file_stat = os.stat(self.chemin_base)
            permissions = oct(file_stat.st_mode)[-3:]

            # Vérifier si le fichier est lisible par d'autres
            if int(permissions[2]) > 0:  # Permissions pour "autres"
                return {
                    'statut': '⚠️ ATTENTION',
                    'message': f'Permissions trop ouvertes: {permissions}',
                    'recommandation': 'chmod 600 pour restreindre l\'accès au propriétaire uniquement'
                }
            else:
                return {
                    'statut': '✅ OK',
                    'message': f'Permissions appropriées: {permissions}',
                    'recommandation': 'Maintenir les permissions restrictives'
                }
        except FileNotFoundError:
            return {
                'statut': '❌ ERREUR',
                'message': 'Fichier de base de données non trouvé',
                'recommandation': 'Vérifier le chemin de la base de données'
            }

    def _detecter_mots_de_passe_faibles(self):
        """Détecte les utilisateurs avec des mots de passe potentiellement faibles"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Rechercher des patterns de mots de passe faibles
            # Note: on ne peut pas voir les mots de passe en clair, mais on peut détecter
            # des comptes créés récemment ou avec des patterns suspects
            cursor.execute("""
                SELECT nom_utilisateur, date_creation
                FROM utilisateurs
                WHERE date_creation > datetime('now', '-7 days')
            """)

            comptes_recents = cursor.fetchall()
            conn.close()

            if comptes_recents:
                return {
                    'statut': '⚠️ ATTENTION',
                    'message': f'{len(comptes_recents)} comptes créés récemment',
                    'recommandation': 'Vérifier que ces comptes utilisent des mots de passe forts',
                    'details': comptes_recents
                }
            else:
                return {
                    'statut': '✅ OK',
                    'message': 'Aucun compte récent détecté',
                    'recommandation': 'Continuer à appliquer une politique de mots de passe forts'
                }
        except sqlite3.Error as e:
            return {
                'statut': '❌ ERREUR',
                'message': f'Erreur lors de la vérification: {e}',
                'recommandation': 'Vérifier la structure de la base de données'
            }

    def _detecter_comptes_inactifs(self):
        """Détecte les comptes qui n'ont pas été utilisés récemment"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("""
                SELECT nom_utilisateur, derniere_connexion, actif
                FROM utilisateurs
                WHERE derniere_connexion < datetime('now', '-90 days')
                   OR derniere_connexion IS NULL
            """)

            comptes_inactifs = cursor.fetchall()
            conn.close()

            if comptes_inactifs:
                return {
                    'statut': '⚠️ ATTENTION',
                    'message': f'{len(comptes_inactifs)} comptes inactifs depuis +90 jours',
                    'recommandation': 'Désactiver ou supprimer les comptes inutilisés',
                    'details': comptes_inactifs
                }
            else:
                return {
                    'statut': '✅ OK',
                    'message': 'Tous les comptes sont actifs',
                    'recommandation': 'Continuer à surveiller l\'activité des comptes'
                }
        except sqlite3.Error as e:
            return {
                'statut': '❌ ERREUR',
                'message': f'Erreur lors de la vérification: {e}',
                'recommandation': 'Vérifier la structure de la base de données'
            }

    def _analyser_tentatives_connexion(self):
        """Analyse les tentatives de connexion suspectes"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Chercher des patterns suspects dans les dernières 24h
            cursor.execute("""
                SELECT adresse_ip, COUNT(*) as tentatives,
                       SUM(CASE WHEN succes = 0 THEN 1 ELSE 0 END) as echecs
                FROM tentatives_connexion
                WHERE timestamp > datetime('now', '-1 day')
                GROUP BY adresse_ip
                HAVING echecs > 5
                ORDER BY echecs DESC
            """)

            ips_suspectes = cursor.fetchall()
            conn.close()

            if ips_suspectes:
                return {
                    'statut': '❌ CRITIQUE',
                    'message': f'{len(ips_suspectes)} adresses IP avec activité suspecte',
                    'recommandation': 'Bloquer les IP suspectes et renforcer la surveillance',
                    'details': ips_suspectes
                }
            else:
                return {
                    'statut': '✅ OK',
                    'message': 'Aucune activité suspecte détectée',
                    'recommandation': 'Continuer la surveillance des tentatives de connexion'
                }
        except sqlite3.Error as e:
            return {
                'statut': '⚠️ ATTENTION',
                'message': f'Table tentatives_connexion non trouvée: {e}',
                'recommandation': 'Activer le logging des tentatives de connexion'
            }

    def _verifier_integrite(self):
        """Vérifie l'intégrité de la base de données"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("PRAGMA integrity_check")
            resultat = cursor.fetchone()[0]
            conn.close()

            if resultat == 'ok':
                return {
                    'statut': '✅ OK',
                    'message': 'Intégrité de la base de données vérifiée',
                    'recommandation': 'Effectuer des vérifications régulières'
                }
            else:
                return {
                    'statut': '❌ CRITIQUE',
                    'message': f'Problème d\'intégrité détecté: {resultat}',
                    'recommandation': 'Restaurer depuis une sauvegarde ou réparer la base'
                }
        except sqlite3.Error as e:
            return {
                'statut': '❌ ERREUR',
                'message': f'Erreur lors de la vérification: {e}',
                'recommandation': 'Vérifier l\'accès à la base de données'
            }

    def _generer_rapport(self, resultats):
        """Génère un rapport de sécurité lisible"""
        print("📊 === RAPPORT DE SÉCURITÉ ===\n")

        score_total = 0
        score_max = len(resultats) * 100

        for categorie, resultat in resultats.items():
            print(f"🔍 {categorie.upper().replace('_', ' ')}")
            print(f"   Statut: {resultat['statut']}")
            print(f"   Message: {resultat['message']}")
            print(f"   Recommandation: {resultat['recommandation']}")

            if 'details' in resultat:
                print(f"   Détails: {resultat['details'][:3]}...")  # Limiter l'affichage

            # Calcul du score
            if '✅' in resultat['statut']:
                score_total += 100
            elif '⚠️' in resultat['statut']:
                score_total += 50
            # ❌ = 0 points

            print()

        pourcentage = (score_total / score_max) * 100
        print(f"📈 SCORE DE SÉCURITÉ: {pourcentage:.1f}% ({score_total}/{score_max} points)")

        if pourcentage >= 80:
            print("🟢 Niveau de sécurité: EXCELLENT")
        elif pourcentage >= 60:
            print("🟡 Niveau de sécurité: ACCEPTABLE (améliorations recommandées)")
        else:
            print("🔴 Niveau de sécurité: INSUFFISANT (actions urgentes requises)")

# Utilisation de l'audit de sécurité
auditeur = ChecklistSecurite('ma_base.db')
auditeur.audit_complet()
```

## Guide de déploiement sécurisé

### Configuration de production

```python
class ConfigurationProduction:
    def __init__(self):
        self.recommandations = []

    def configurer_environnement_securise(self, chemin_base, chemin_config):
        """Configure un environnement de production sécurisé"""
        print("🚀 === CONFIGURATION PRODUCTION SÉCURISÉE ===\n")

        # 1. Vérifier les permissions système
        self._configurer_permissions_systeme(chemin_base)

        # 2. Configurer les paramètres SQLite optimaux
        self._configurer_sqlite_production(chemin_base)

        # 3. Mettre en place la surveillance
        self._configurer_surveillance(chemin_base)

        # 4. Configurer les sauvegardes automatiques
        self._configurer_sauvegardes(chemin_base)

        # 5. Générer le fichier de configuration
        self._generer_config_production(chemin_config)

        self._afficher_recommandations()

    def _configurer_permissions_systeme(self, chemin_base):
        """Configure les permissions au niveau système"""
        import os
        import pwd
        import grp

        print("🔒 Configuration des permissions système...")

        try:
            # Changer les permissions du fichier
            os.chmod(chemin_base, 0o600)  # Lecture/écriture pour le propriétaire uniquement
            print("   ✅ Permissions fichier configurées (600)")

            # Vérifier le propriétaire
            file_stat = os.stat(chemin_base)
            owner = pwd.getpwuid(file_stat.st_uid).pw_name
            group = grp.getgrgid(file_stat.st_gid).gr_name

            print(f"   📋 Propriétaire: {owner}:{group}")

            if owner == 'root':
                self.recommandations.append(
                    "⚠️ La base appartient à root. Considérer un utilisateur dédié pour l'application."
                )

        except Exception as e:
            print(f"   ❌ Erreur permissions: {e}")
            self.recommandations.append(
                "❌ Configurer manuellement les permissions du fichier de base."
            )

    def _configurer_sqlite_production(self, chemin_base):
        """Configure SQLite avec des paramètres optimaux pour la production"""
        print("⚙️ Configuration SQLite pour production...")

        try:
            conn = sqlite3.connect(chemin_base)
            cursor = conn.cursor()

            # Configuration recommandée pour la production
            configurations = [
                ("PRAGMA journal_mode = WAL", "Mode WAL pour de meilleures performances"),
                ("PRAGMA synchronous = NORMAL", "Synchronisation équilibrée"),
                ("PRAGMA cache_size = 10000", "Cache augmenté"),
                ("PRAGMA temp_store = memory", "Stockage temporaire en mémoire"),
                ("PRAGMA mmap_size = 268435456", "Memory mapping (256MB)"),
                ("PRAGMA optimize", "Optimisation automatique"),
            ]

            for pragma, description in configurations:
                cursor.execute(pragma)
                print(f"   ✅ {description}")

            conn.commit()
            conn.close()

        except Exception as e:
            print(f"   ❌ Erreur configuration SQLite: {e}")
            self.recommandations.append(
                "❌ Vérifier et appliquer manuellement les paramètres PRAGMA recommandés."
            )

    def _configurer_surveillance(self, chemin_base):
        """Met en place un système de surveillance basique"""
        print("👁️ Configuration de la surveillance...")

        script_surveillance = f"""#!/bin/bash
# Script de surveillance SQLite - {chemin_base}

# Vérifier la taille du fichier
taille=$(stat -f%z "{chemin_base}" 2>/dev/null || stat -c%s "{chemin_base}" 2>/dev/null)
echo "$(date): Taille base de données: $taille bytes" >> /var/log/sqlite_monitoring.log

# Vérifier l'intégrité (une fois par jour)
if [ $(date +%H) -eq 2 ]; then
    sqlite3 "{chemin_base}" "PRAGMA integrity_check;" >> /var/log/sqlite_integrity.log 2>&1
fi

# Alerter si la base devient trop volumineuse (>1GB par défaut)
if [ $taille -gt 1073741824 ]; then
    echo "$(date): ALERTE - Base de données dépasse 1GB" >> /var/log/sqlite_alerts.log
fi
"""

        try:
            with open('/tmp/sqlite_monitor.sh', 'w') as f:
                f.write(script_surveillance)
            os.chmod('/tmp/sqlite_monitor.sh', 0o755)
            print("   ✅ Script de surveillance créé: /tmp/sqlite_monitor.sh")
            self.recommandations.append(
                "📋 Ajouter le script de surveillance au crontab: */15 * * * * /tmp/sqlite_monitor.sh"
            )
        except Exception as e:
            print(f"   ⚠️ Impossible de créer le script de surveillance: {e}")

    def _configurer_sauvegardes(self, chemin_base):
        """Configure un système de sauvegarde automatique"""
        print("💾 Configuration des sauvegardes...")

        script_backup = f"""#!/bin/bash
# Script de sauvegarde SQLite automatique

BACKUP_DIR="/backup/sqlite"
DB_PATH="{chemin_base}"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.db"

# Créer le répertoire de sauvegarde si nécessaire
mkdir -p "$BACKUP_DIR"

# Effectuer la sauvegarde avec sqlite3
sqlite3 "$DB_PATH" ".backup '$BACKUP_FILE'"

if [ $? -eq 0 ]; then
    echo "$(date): Sauvegarde réussie: $BACKUP_FILE" >> /var/log/sqlite_backup.log

    # Comprimer la sauvegarde
    gzip "$BACKUP_FILE"

    # Supprimer les sauvegardes de plus de 30 jours
    find "$BACKUP_DIR" -name "backup_*.db.gz" -mtime +30 -delete

else
    echo "$(date): ERREUR - Échec de la sauvegarde" >> /var/log/sqlite_backup.log
fi
"""

        try:
            with open('/tmp/sqlite_backup.sh', 'w') as f:
                f.write(script_backup)
            os.chmod('/tmp/sqlite_backup.sh', 0o755)
            print("   ✅ Script de sauvegarde créé: /tmp/sqlite_backup.sh")
            self.recommandations.append(
                "📋 Programmer les sauvegardes: 0 2 * * * /tmp/sqlite_backup.sh (tous les jours à 2h)"
            )
        except Exception as e:
            print(f"   ⚠️ Impossible de créer le script de sauvegarde: {e}")

    def _generer_config_production(self, chemin_config):
        """Génère un fichier de configuration pour la production"""
        print("📝 Génération du fichier de configuration...")

        config = {
            "database": {
                "path": "{{ DATABASE_PATH }}",
                "encryption": True,
                "backup_enabled": True,
                "monitoring_enabled": True
            },
            "security": {
                "max_login_attempts": 3,
                "session_timeout_minutes": 30,
                "password_policy": {
                    "min_length": 8,
                    "require_uppercase": True,
                    "require_lowercase": True,
                    "require_digits": True,
                    "require_special_chars": True
                }
            },
            "performance": {
                "journal_mode": "WAL",
                "cache_size": 10000,
                "synchronous": "NORMAL",
                "temp_store": "memory"
            },
            "logging": {
                "audit_enabled": True,
                "log_level": "INFO",
                "max_log_size_mb": 100
            }
        }

        try:
            import json
            with open(chemin_config, 'w') as f:
                json.dump(config, f, indent=2)
            print(f"   ✅ Configuration sauvegardée: {chemin_config}")
        except Exception as e:
            print(f"   ⚠️ Erreur sauvegarde configuration: {e}")

    def _afficher_recommandations(self):
        """Affiche toutes les recommandations collectées"""
        print("\n🎯 === RECOMMANDATIONS DE DÉPLOIEMENT ===")

        if not self.recommandations:
            print("✅ Aucune action supplémentaire requise.")
            return

        for i, recommandation in enumerate(self.recommandations, 1):
            print(f"{i}. {recommandation}")

        print("\n📚 DOCUMENTATION SUPPLÉMENTAIRE:")
        print("- Consulter la documentation SQLite pour les optimisations avancées")
        print("- Mettre en place une surveillance des performances applicatives")
        print("- Tester régulièrement les procédures de récupération")
        print("- Former l'équipe aux bonnes pratiques de sécurité SQLite")

# Utilisation du configurateur de production
configurateur = ConfigurationProduction()
configurateur.configurer_environnement_securise('ma_base.db', 'config_production.json')
```

## Tests de sécurité et validation

### Suite de tests de sécurité

```python
import unittest
import tempfile
import os

class TestsSecuriteSQLite(unittest.TestCase):
    def setUp(self):
        """Prépare un environnement de test"""
        self.temp_db = tempfile.NamedTemporaryFile(delete=False, suffix='.db')
        self.temp_db.close()
        self.chemin_base = self.temp_db.name

        # Initialiser la base de test
        self.gestionnaire = GestionnaireUtilisateursSecurise(self.chemin_base)
        self._creer_donnees_test()

    def tearDown(self):
        """Nettoie après les tests"""
        try:
            os.unlink(self.chemin_base)
        except:
            pass

    def _creer_donnees_test(self):
        """Crée des données de test"""
        self.gestionnaire.creer_utilisateur('admin_test', 'admin@test.com', 'AdminPass123!', 'admin')
        self.gestionnaire.creer_utilisateur('user_test', 'user@test.com', 'UserPass456!', 'utilisateur')

    def test_authentification_valide(self):
        """Test d'authentification avec des identifiants valides"""
        resultat = self.gestionnaire.authentifier('admin_test', 'AdminPass123!')
        self.assertTrue(resultat, "L'authentification devrait réussir avec des identifiants valides")
        self.assertIsNotNone(self.gestionnaire.utilisateur_actuel)

    def test_authentification_invalide(self):
        """Test d'authentification avec des identifiants invalides"""
        resultat = self.gestionnaire.authentifier('admin_test', 'mauvais_password')
        self.assertFalse(resultat, "L'authentification devrait échouer avec un mauvais mot de passe")
        self.assertIsNone(self.gestionnaire.utilisateur_actuel)

    def test_creation_utilisateur_donnees_invalides(self):
        """Test de création d'utilisateur avec des données invalides"""
        # Nom d'utilisateur trop court
        resultat = self.gestionnaire.creer_utilisateur('ab', 'test@email.com', 'ValidPass123!', 'utilisateur')
        self.assertFalse(resultat, "La création devrait échouer avec un nom trop court")

        # Email invalide
        resultat = self.gestionnaire.creer_utilisateur('validuser', 'email_invalide', 'ValidPass123!', 'utilisateur')
        self.assertFalse(resultat, "La création devrait échouer avec un email invalide")

        # Mot de passe faible
        resultat = self.gestionnaire.creer_utilisateur('validuser', 'valid@email.com', 'weak', 'utilisateur')
        self.assertFalse(resultat, "La création devrait échouer avec un mot de passe faible")

    def test_permissions_roles(self):
        """Test du système de permissions par rôles"""
        # Connexion en tant qu'admin
        self.gestionnaire.authentifier('admin_test', 'AdminPass123!')
        self.assertTrue(self.gestionnaire.a_permission('lire_tous_utilisateurs'))

        # Déconnexion et reconnexion en tant qu'utilisateur normal
        self.gestionnaire.deconnexion()
        self.gestionnaire.authentifier('user_test', 'UserPass456!')
        self.assertFalse(self.gestionnaire.a_permission('lire_tous_utilisateurs'))
        self.assertTrue(self.gestionnaire.a_permission('lire_donnees'))

    def test_protection_injection_sql(self):
        """Test de protection contre l'injection SQL"""
        recherche = RequeteSecurisee(self.chemin_base)

        # Tentative d'injection SQL
        terme_malveillant = "'; DROP TABLE utilisateurs; --"
        try:
            resultats = recherche.rechercher_utilisateurs_securise(terme_malveillant)
            # Si nous arrivons ici, l'injection a été bloquée (ce qui est bien)
            self.assertIsInstance(resultats, list)
        except Exception:
            self.fail("La recherche sécurisée ne devrait pas lever d'exception")

    def test_force_brute_protection(self):
        """Test de protection contre les attaques par force brute"""
        gestionnaire_protege = GestionnaireUtilisateursProtege(self.chemin_base)

        # Créer un utilisateur de test
        gestionnaire_protege.creer_utilisateur('test_brute', 'brute@test.com', 'TestPass123!', 'utilisateur')

        # Simuler plusieurs tentatives échouées
        for i in range(4):
            resultat = gestionnaire_protege.authentifier('test_brute', 'mauvais_password', '127.0.0.1')
            self.assertFalse(resultat)

        # Vérifier que le compte est maintenant bloqué même avec le bon mot de passe
        resultat = gestionnaire_protege.authentifier('test_brute', 'TestPass123!', '127.0.0.1')
        self.assertFalse(resultat, "Le compte devrait être bloqué après plusieurs tentatives échouées")

    def test_validation_entrees(self):
        """Test de validation des entrées utilisateur"""
        # Test de validation d'email
        self.assertTrue(ValidateurEntrees.valider_email('test@example.com'))
        self.assertFalse(ValidateurEntrees.valider_email('email_invalide'))

        # Test de validation de nom d'utilisateur
        self.assertTrue(ValidateurEntrees.valider_nom_utilisateur('user123'))
        self.assertFalse(ValidateurEntrees.valider_nom_utilisateur('ab'))  # Trop court

        # Test de validation de mot de passe
        valide, message = ValidateurEntrees.valider_mot_de_passe('MotDePasse123!')
        self.assertTrue(valide)

        valide, message = ValidateurEntrees.valider_mot_de_passe('weak')
        self.assertFalse(valide)

    def test_audit_logging(self):
        """Test du système d'audit"""
        audit = AuditLogger(self.chemin_base)

        # Enregistrer une action
        audit.enregistrer_action(
            utilisateur_id=1,
            action='CREATE_USER',
            table_cible='utilisateurs',
            nouvelles_valeurs={'nom': 'nouveau_user'}
        )

        # Vérifier que l'action est enregistrée
        logs = audit.lire_audit(utilisateur_id=1, limite=10)
        self.assertGreater(len(logs), 0, "L'action devrait être enregistrée dans l'audit")

# Exécution des tests
if __name__ == '__main__':
    print("🧪 === TESTS DE SÉCURITÉ SQLITE ===\n")

    # Créer une suite de tests
    suite = unittest.TestLoader().loadTestsFromTestCase(TestsSecuriteSQLite)

    # Exécuter les tests avec des détails
    runner = unittest.TextTestRunner(verbosity=2)
    resultat = runner.run(suite)

    # Afficher un résumé
    print(f"\n📊 === RÉSUMÉ DES TESTS ===")
    print(f"Tests exécutés: {resultat.testsRun}")
    print(f"Succès: {resultat.testsRun - len(resultat.failures) - len(resultat.errors)}")
    print(f"Échecs: {len(resultat.failures)}")
    print(f"Erreurs: {len(resultat.errors)}")

    if resultat.failures:
        print("\n❌ ÉCHECS:")
        for test, trace in resultat.failures:
            print(f"- {test}: {trace.split('AssertionError: ')[-1].split('\\n')[0]}")

    if resultat.errors:
        print("\n💥 ERREURS:")
        for test, trace in resultat.errors:
            print(f"- {test}: {trace.split('\\n')[-2]}")

    if len(resultat.failures) == 0 and len(resultat.errors) == 0:
        print("\n🎉 Tous les tests de sécurité sont passés avec succès!")
    else:
        print("\n⚠️ Des problèmes de sécurité ont été détectés. Vérifiez les échecs ci-dessus.")
```

## Conclusion et points clés

### Résumé des concepts essentiels

**🔑 Points clés à retenir:**

1. **SQLite n'a pas de gestion native des utilisateurs** - vous devez l'implémenter au niveau applicatif
2. **La sécurité repose sur plusieurs couches** : chiffrement + permissions système + contrôle applicatif
3. **Toujours utiliser des requêtes paramétrées** pour éviter l'injection SQL
4. **Implémenter une protection contre la force brute** avec limitation des tentatives
5. **Auditer et surveiller** les accès et actions sensibles
6. **Valider rigoureusement** toutes les entrées utilisateur
7. **Tester régulièrement** la sécurité avec des tests automatisés

### Checklist finale de déploiement

```
☐ Base de données chiffrée (SQLCipher ou équivalent)
☐ Permissions système restrictives (600 ou équivalent)
☐ Système d'authentification robuste
☐ Gestion des rôles et permissions implémentée
☐ Protection contre l'injection SQL (requêtes paramétrées)
☐ Protection contre la force brute (limitation des tentatives)
☐ Validation stricte des entrées utilisateur
☐ Système d'audit et de logging opérationnel
☐ Sauvegardes automatisées configurées
☐ Surveillance et monitoring en place
☐ Tests de sécurité automatisés
☐ Documentation des procédures de sécurité
☐ Formation de l'équipe aux bonnes pratiques
☐ Plan de réponse aux incidents défini
☐ Revues de sécurité régulières planifiées
```

### Matrice de responsabilités sécuritaires

| Couche de sécurité | Responsable | Technologies | Niveau critique |
|-------------------|-------------|--------------|-----------------|
| **Chiffrement des données** | Développeur/Admin | SQLCipher, GPG | 🔴 Critique |
| **Permissions système** | Administrateur système | chmod, chown, ACL | 🔴 Critique |
| **Authentification** | Développeur | Application custom | 🔴 Critique |
| **Autorisation/Permissions** | Développeur | RBAC/ABAC custom | 🟡 Important |
| **Validation des entrées** | Développeur | Regex, filtres | 🟡 Important |
| **Protection force brute** | Développeur | Rate limiting | 🟡 Important |
| **Audit et logging** | Développeur/Admin | Logs applicatifs | 🟢 Recommandé |
| **Surveillance** | Administrateur | Scripts, monitoring | 🟢 Recommandé |
| **Sauvegardes** | Administrateur | Scripts automatisés | 🔴 Critique |

### Guide de dépannage sécuritaire

#### Problèmes d'authentification courants

```python
class DiagnosticSecurite:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base

    def diagnostiquer_probleme_connexion(self, nom_utilisateur, symptomes):
        """Diagnostique les problèmes de connexion"""
        print(f"🔍 Diagnostic pour l'utilisateur: {nom_utilisateur}")

        if "mot de passe incorrect" in symptomes:
            self._verifier_mot_de_passe(nom_utilisateur)
        elif "compte bloqué" in symptomes:
            self._verifier_blocage_compte(nom_utilisateur)
        elif "base cryptée" in symptomes:
            self._verifier_chiffrement()
        elif "permissions refusées" in symptomes:
            self._verifier_permissions_fichier()

    def _verifier_mot_de_passe(self, nom_utilisateur):
        """Vérifications liées au mot de passe"""
        print("🔐 Vérification du mot de passe...")

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("SELECT id FROM utilisateurs WHERE nom_utilisateur = ?", (nom_utilisateur,))
            if not cursor.fetchone():
                print("   ❌ Utilisateur inexistant")
                return

            print("   ✅ Utilisateur trouvé dans la base")
            print("   💡 Solutions possibles:")
            print("      - Vérifier la casse du mot de passe")
            print("      - Réinitialiser le mot de passe si nécessaire")
            print("      - Vérifier l'algorithme de hashage utilisé")

            conn.close()

        except Exception as e:
            print(f"   ❌ Erreur lors de la vérification: {e}")

    def _verifier_blocage_compte(self, nom_utilisateur):
        """Vérifications liées au blocage de compte"""
        print("🔒 Vérification du blocage de compte...")

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Vérifier les tentatives récentes
            cursor.execute("""
                SELECT COUNT(*) FROM tentatives_connexion
                WHERE nom_utilisateur = ? AND succes = 0
                AND timestamp > datetime('now', '-15 minutes')
            """, (nom_utilisateur,))

            tentatives_recentes = cursor.fetchone()[0]

            if tentatives_recentes >= 3:
                print(f"   ⚠️ {tentatives_recentes} tentatives échouées récentes")
                print("   💡 Solutions:")
                print("      - Attendre 15-30 minutes")
                print("      - Débloquer manuellement le compte")
                print("      - Vérifier les logs pour détecter une attaque")
            else:
                print("   ✅ Pas de blocage détecté par tentatives")

            conn.close()

        except Exception as e:
            print(f"   ❌ Table tentatives_connexion non trouvée: {e}")
            print("   💡 Le système de protection force brute n'est peut-être pas activé")

    def _verifier_chiffrement(self):
        """Vérifications liées au chiffrement"""
        print("🔐 Vérification du chiffrement...")

        try:
            # Tenter d'ouvrir sans mot de passe
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()
            cursor.execute("SELECT name FROM sqlite_master LIMIT 1")
            conn.close()

            print("   ⚠️ Base non chiffrée ou mot de passe non requis")

        except sqlite3.DatabaseError as e:
            if "encrypted" in str(e) or "file is not a database" in str(e):
                print("   ✅ Base chiffrée détectée")
                print("   💡 Solutions:")
                print("      - Vérifier que PRAGMA key est défini")
                print("      - Vérifier la validité du mot de passe de chiffrement")
                print("      - Utiliser SQLCipher au lieu de SQLite standard")
            else:
                print(f"   ❌ Erreur inattendue: {e}")

    def _verifier_permissions_fichier(self):
        """Vérifications des permissions système"""
        print("📁 Vérification des permissions fichier...")

        import os
        import stat

        try:
            file_stat = os.stat(self.chemin_base)
            permissions = oct(file_stat.st_mode)[-3:]

            print(f"   📋 Permissions actuelles: {permissions}")

            if not os.access(self.chemin_base, os.R_OK):
                print("   ❌ Pas de permission de lecture")
            elif not os.access(self.chemin_base, os.W_OK):
                print("   ❌ Pas de permission d'écriture")
            else:
                print("   ✅ Permissions de lecture/écriture OK")

            print("   💡 Solutions si problème:")
            print("      - chmod 600 pour le propriétaire uniquement")
            print("      - chown pour changer le propriétaire")
            print("      - Vérifier que l'application s'exécute avec le bon utilisateur")

        except FileNotFoundError:
            print("   ❌ Fichier de base de données non trouvé")
            print("   💡 Vérifier le chemin de la base de données")

# Outil de diagnostic
diagnostic = DiagnosticSecurite('ma_base.db')

# Exemples d'utilisation
print("=== EXEMPLES DE DIAGNOSTIC ===")
diagnostic.diagnostiquer_probleme_connexion('alice', ["mot de passe incorrect"])
print()
diagnostic.diagnostiquer_probleme_connexion('bob', ["compte bloqué"])
```

### Scripts d'urgence

#### Script de déverrouillage d'urgence

```python
class OutilsUrgence:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base

    def deverrouiller_compte_urgence(self, nom_utilisateur, mot_de_passe_admin):
        """Déverrouille un compte en urgence (admin uniquement)"""
        print(f"🚨 DÉVERROUILLAGE D'URGENCE pour {nom_utilisateur}")

        # Vérifier les privilèges admin
        gestionnaire = GestionnaireUtilisateurs(self.chemin_base)
        if not gestionnaire.authentifier('admin', mot_de_passe_admin):
            print("❌ Authentification admin échouée")
            return False

        if not gestionnaire.a_permission('modifier_utilisateurs'):
            print("❌ Privilèges insuffisants")
            return False

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Supprimer les tentatives de connexion récentes
            cursor.execute("""
                DELETE FROM tentatives_connexion
                WHERE nom_utilisateur = ?
            """, (nom_utilisateur,))

            # Réactiver le compte s'il était désactivé
            cursor.execute("""
                UPDATE utilisateurs
                SET actif = 1
                WHERE nom_utilisateur = ?
            """, (nom_utilisateur,))

            conn.commit()
            conn.close()

            print(f"✅ Compte {nom_utilisateur} déverrouillé")
            return True

        except Exception as e:
            print(f"❌ Erreur lors du déverrouillage: {e}")
            return False

    def reinitialiser_mot_de_passe_urgence(self, nom_utilisateur, nouveau_mot_de_passe, mot_de_passe_admin):
        """Réinitialise un mot de passe en urgence"""
        print(f"🔑 RÉINITIALISATION MOT DE PASSE pour {nom_utilisateur}")

        # Vérification admin (similaire à ci-dessus)
        gestionnaire = GestionnaireUtilisateurs(self.chemin_base)
        if not gestionnaire.authentifier('admin', mot_de_passe_admin):
            print("❌ Authentification admin échouée")
            return False

        try:
            # Utiliser le gestionnaire pour changer le mot de passe
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Générer nouveau hash
            import secrets
            import hashlib
            sel = secrets.token_hex(32)
            nouveau_hash = hashlib.pbkdf2_hmac('sha256',
                                              nouveau_mot_de_passe.encode('utf-8'),
                                              sel.encode('utf-8'),
                                              100000).hex()

            cursor.execute("""
                UPDATE utilisateurs
                SET mot_de_passe_hash = ?, sel = ?, actif = 1
                WHERE nom_utilisateur = ?
            """, (nouveau_hash, sel, nom_utilisateur))

            conn.commit()
            conn.close()

            print(f"✅ Mot de passe réinitialisé pour {nom_utilisateur}")
            print("⚠️ Demander à l'utilisateur de changer son mot de passe dès sa prochaine connexion")
            return True

        except Exception as e:
            print(f"❌ Erreur lors de la réinitialisation: {e}")
            return False

    def sauvegarder_urgence(self):
        """Effectue une sauvegarde d'urgence"""
        print("💾 SAUVEGARDE D'URGENCE")

        import shutil
        from datetime import datetime

        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        backup_path = f"{self.chemin_base}.emergency_backup_{timestamp}"

        try:
            shutil.copy2(self.chemin_base, backup_path)
            print(f"✅ Sauvegarde d'urgence créée: {backup_path}")
            return backup_path
        except Exception as e:
            print(f"❌ Échec de la sauvegarde: {e}")
            return None

    def analyser_securite_rapide(self):
        """Analyse rapide de sécurité pour détecter les problèmes urgents"""
        print("🔍 ANALYSE DE SÉCURITÉ RAPIDE")

        problemes_critiques = []

        # Vérifier l'intégrité
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()
            cursor.execute("PRAGMA integrity_check")
            integrite = cursor.fetchone()[0]
            if integrite != 'ok':
                problemes_critiques.append(f"Intégrité compromise: {integrite}")
            conn.close()
        except Exception as e:
            problemes_critiques.append(f"Erreur accès base: {e}")

        # Vérifier les permissions fichier
        import os
        if os.path.exists(self.chemin_base):
            file_stat = os.stat(self.chemin_base)
            permissions = oct(file_stat.st_mode)[-3:]
            if int(permissions[2]) > 0:  # Autres ont des permissions
                problemes_critiques.append(f"Permissions trop ouvertes: {permissions}")

        # Vérifier les tentatives d'attaque récentes
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()
            cursor.execute("""
                SELECT COUNT(*) FROM tentatives_connexion
                WHERE succes = 0 AND timestamp > datetime('now', '-1 hour')
            """)
            tentatives_recentes = cursor.fetchone()[0]
            if tentatives_recentes > 10:
                problemes_critiques.append(f"Nombreuses tentatives d'intrusion: {tentatives_recentes}")
            conn.close()
        except:
            pass  # Table peut ne pas exister

        # Rapport
        if problemes_critiques:
            print("🚨 PROBLÈMES CRITIQUES DÉTECTÉS:")
            for probleme in problemes_critiques:
                print(f"   ❌ {probleme}")
            print("\n💡 ACTIONS RECOMMANDÉES:")
            print("   1. Effectuer une sauvegarde immédiate")
            print("   2. Corriger les problèmes identifiés")
            print("   3. Analyser les logs pour détecter une intrusion")
            print("   4. Changer les mots de passe si nécessaire")
        else:
            print("✅ Aucun problème critique détecté")

# Outils d'urgence
outils_urgence = OutilsUrgence('ma_base.db')

# Exemple d'utilisation en urgence
print("🚨 === PROCÉDURES D'URGENCE ===")
outils_urgence.analyser_securite_rapide()
```

### Documentation de référence rapide

#### Commandes essentielles de dépannage

```bash
# === DIAGNOSTIC RAPIDE ===

# 1. Vérifier l'intégrité de la base
sqlite3 ma_base.db "PRAGMA integrity_check;"

# 2. Vérifier les permissions fichier
ls -la ma_base.db
stat ma_base.db

# 3. Tester la connexion basique
sqlite3 ma_base.db ".tables"

# 4. Vérifier si la base est chiffrée
file ma_base.db
hexdump -C ma_base.db | head -1

# === CORRECTIONS RAPIDES ===

# 1. Corriger les permissions
chmod 600 ma_base.db
chown monapp:monapp ma_base.db

# 2. Sauvegarde d'urgence
cp ma_base.db ma_base.db.backup.$(date +%Y%m%d_%H%M%S)

# 3. Nettoyer les logs de tentatives (si trop volumineux)
sqlite3 ma_base.db "DELETE FROM tentatives_connexion WHERE timestamp < datetime('now', '-7 days');"

# 4. Optimiser la base après incident
sqlite3 ma_base.db "VACUUM; REINDEX; PRAGMA optimize;"
```

#### Requêtes SQL de maintenance sécuritaire

```sql
-- === REQUÊTES DE SURVEILLANCE ===

-- 1. Comptes inactifs (plus de 90 jours)
SELECT nom_utilisateur, derniere_connexion,
       julianday('now') - julianday(derniere_connexion) as jours_inactivite
FROM utilisateurs
WHERE derniere_connexion < datetime('now', '-90 days')
   OR derniere_connexion IS NULL;

-- 2. Tentatives de connexion suspectes (dernières 24h)
SELECT adresse_ip, nom_utilisateur, COUNT(*) as tentatives,
       SUM(CASE WHEN succes = 0 THEN 1 ELSE 0 END) as echecs
FROM tentatives_connexion
WHERE timestamp > datetime('now', '-1 day')
GROUP BY adresse_ip, nom_utilisateur
HAVING echecs > 3
ORDER BY echecs DESC;

-- 3. Actions d'audit sensibles récentes
SELECT u.nom_utilisateur, al.action, al.table_cible, al.timestamp
FROM audit_log al
JOIN utilisateurs u ON al.utilisateur_id = u.id
WHERE al.action IN ('DELETE', 'UPDATE_USER', 'CREATE_USER')
  AND al.timestamp > datetime('now', '-7 days')
ORDER BY al.timestamp DESC;

-- 4. Utilisation de l'espace par table
SELECT name,
       (SELECT COUNT(*) FROM pragma_table_info(name)) as colonnes,
       (SELECT COUNT(*) FROM sqlite_master WHERE tbl_name = name AND type = 'index') as index_count
FROM sqlite_master
WHERE type = 'table'
  AND name NOT LIKE 'sqlite_%';

-- === REQUÊTES DE NETTOYAGE ===

-- 1. Supprimer les anciennes tentatives de connexion
DELETE FROM tentatives_connexion
WHERE timestamp < datetime('now', '-30 days');

-- 2. Supprimer les sessions expirées
DELETE FROM sessions
WHERE date_expiration < datetime('now');

-- 3. Nettoyer les logs d'audit anciens (garder 1 an)
DELETE FROM audit_log
WHERE timestamp < datetime('now', '-1 year');

-- === REQUÊTES D'URGENCE ===

-- 1. Désactiver tous les comptes sauf admin (en cas d'intrusion)
UPDATE utilisateurs SET actif = 0 WHERE role != 'admin';

-- 2. Forcer la déconnexion de toutes les sessions
DELETE FROM sessions;

-- 3. Bloquer temporairement toutes les connexions depuis une IP
INSERT INTO tentatives_connexion (nom_utilisateur, adresse_ip, succes, timestamp)
SELECT 'BLOCKED', '192.168.1.100', 0, datetime('now')
FROM (SELECT 1 UNION SELECT 1 UNION SELECT 1 UNION SELECT 1 UNION SELECT 1);
```

### Formation et sensibilisation

#### Programme de formation recommandé

```
🎓 FORMATION SÉCURITÉ SQLITE - PROGRAMME

MODULE 1: Fondamentaux (2h)
├── Architecture SQLite et implications sécuritaires
├── Différences avec les SGBD serveur
├── Modèle de menaces spécifique
└── Bonnes pratiques générales

MODULE 2: Chiffrement et protection des données (3h)
├── SQLCipher: installation et utilisation
├── Gestion sécurisée des clés
├── Chiffrement sélectif des champs
└── Sauvegarde des données chiffrées

MODULE 3: Authentification et autorisations (4h)
├── Conception d'un système d'authentification
├── Gestion des rôles et permissions
├── Protection contre la force brute
└── Systèmes RBAC et ABAC

MODULE 4: Sécurisation du code (3h)
├── Protection contre l'injection SQL
├── Validation et assainissement des entrées
├── Gestion sécurisée des erreurs
└── Tests de sécurité automatisés

MODULE 5: Surveillance et audit (2h)
├── Mise en place des logs d'audit
├── Détection d'intrusions
├── Surveillance des performances
└── Procédures d'incident

MODULE 6: Déploiement et maintenance (2h)
├── Configuration de production
├── Scripts de surveillance
├── Sauvegardes automatisées
└── Procédures d'urgence

ÉVALUATION PRATIQUE (2h)
├── Audit d'une base existante
├── Correction de vulnérabilités
├── Mise en place d'un système complet
└── Simulation de réponse à incident
```

## Conclusion générale

La gestion des permissions et le contrôle d'accès dans SQLite représentent un défi unique qui nécessite une approche structurée et multicouche. Contrairement aux systèmes de bases de données traditionnels, SQLite délègue entièrement la responsabilité de la sécurité à l'application et à l'environnement d'exécution.

### Principes fondamentaux à retenir

1. **Sécurité en profondeur** : Combiner chiffrement, permissions système, contrôles applicatifs et surveillance
2. **Validation rigoureuse** : Ne jamais faire confiance aux données entrantes
3. **Principe du moindre privilège** : Accorder uniquement les permissions nécessaires
4. **Audit et traçabilité** : Enregistrer toutes les actions sensibles
5. **Tests continus** : Valider régulièrement la sécurité avec des tests automatisés

### Évolutions futures

Le domaine de la sécurité SQLite continue d'évoluer avec :
- **Nouvelles extensions de chiffrement** plus performantes
- **Intégration avec des systèmes d'identité** modernes (OAuth, SAML)
- **Outils de surveillance** spécialisés pour SQLite
- **Frameworks de sécurité** dédiés aux applications SQLite

### Ressources pour aller plus loin

- **Documentation officielle SQLite** : https://sqlite.org/security.html
- **Projet SQLCipher** : https://www.zetetic.net/sqlcipher/
- **OWASP Application Security** : Principes généraux applicables
- **Communautés spécialisées** : Forums et groupes dédiés à la sécurité SQLite

La sécurité n'est jamais un état final mais un processus continu d'amélioration et d'adaptation aux nouvelles menaces. La formation de l'équipe, la mise en place de procédures rigoureuses et la surveillance constante sont les clés d'un système SQLite sécurisé et fiable.

---

*Cette section complète le chapitre 8.2 sur la gestion des permissions et le contrôle d'accès. La section suivante (8.3) abordera l'audit et le logging des opérations SQLite.*

⏭️
