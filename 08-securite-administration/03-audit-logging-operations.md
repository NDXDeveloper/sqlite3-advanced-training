🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.3 Audit et logging des opérations

## Introduction à l'audit et au logging

L'audit et le logging sont essentiels pour maintenir la sécurité et la traçabilité de votre base de données SQLite. Contrairement aux systèmes de bases de données d'entreprise qui ont des systèmes d'audit intégrés, SQLite nécessite que vous implémentiez votre propre système de suivi des opérations.

### Qu'est-ce que l'audit et le logging ?

**Audit** : Enregistrement systématique des actions effectuées sur la base de données pour des raisons de sécurité, de conformité et de traçabilité.

**Logging** : Enregistrement détaillé des événements et opérations pour le débogage, le monitoring et l'analyse des performances.

### Pourquoi implémenter l'audit dans SQLite ?

**Raisons de sécurité :**
- Détecter les accès non autorisés
- Identifier les tentatives d'intrusion
- Tracer les modifications de données sensibles
- Prouver la conformité réglementaire

**Raisons opérationnelles :**
- Déboguer les problèmes de données
- Analyser les patterns d'utilisation
- Optimiser les performances
- Restaurer des données modifiées par erreur

**Scénarios concrets :**
```
🏥 Application médicale : Tracer qui a consulté quel dossier patient
💰 Application financière : Enregistrer toutes les transactions
📊 Application RH : Auditer les modifications de salaires
🔐 Système d'administration : Logger tous les changements de permissions
```

## Conception d'un système d'audit

### Structure de base pour l'audit

Commençons par créer les tables nécessaires pour un système d'audit complet :

```sql
-- Table principale d'audit
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    utilisateur_id INTEGER,
    nom_utilisateur TEXT,
    action TEXT NOT NULL,                    -- SELECT, INSERT, UPDATE, DELETE
    table_cible TEXT,                        -- Table concernée
    enregistrement_id INTEGER,               -- ID de l'enregistrement modifié
    anciennes_valeurs TEXT,                  -- JSON des anciennes valeurs
    nouvelles_valeurs TEXT,                  -- JSON des nouvelles valeurs
    adresse_ip TEXT,
    user_agent TEXT,
    session_id TEXT,
    duree_ms INTEGER,                        -- Durée de l'opération en millisecondes
    statut TEXT DEFAULT 'SUCCESS',           -- SUCCESS, ERROR, WARNING
    message_erreur TEXT,
    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs(id)
);

-- Table pour les événements de sécurité spécifiques
CREATE TABLE audit_securite (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    type_evenement TEXT NOT NULL,            -- LOGIN, LOGOUT, FAILED_LOGIN, PERMISSION_DENIED
    utilisateur_id INTEGER,
    nom_utilisateur TEXT,
    adresse_ip TEXT,
    details TEXT,                            -- JSON avec détails spécifiques
    niveau_gravite TEXT DEFAULT 'INFO',      -- INFO, WARNING, CRITICAL
    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs(id)
);

-- Table pour l'audit des changements de schéma
CREATE TABLE audit_schema (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    utilisateur_id INTEGER,
    type_operation TEXT,                     -- CREATE_TABLE, ALTER_TABLE, DROP_TABLE, etc.
    nom_objet TEXT,                         -- Nom de la table/index/trigger
    definition_sql TEXT,                    -- SQL complet de la modification
    FOREIGN KEY (utilisateur_id) REFERENCES utilisateurs(id)
);

-- Index pour améliorer les performances des requêtes d'audit
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp);  
CREATE INDEX idx_audit_utilisateur ON audit_log(utilisateur_id);  
CREATE INDEX idx_audit_table ON audit_log(table_cible);  
CREATE INDEX idx_audit_securite_timestamp ON audit_securite(timestamp);  
CREATE INDEX idx_audit_securite_type ON audit_securite(type_evenement);  
```

### Classe Python pour la gestion de l'audit

> ⚠️ **Deux pièges à régler avant tout filtre temporel — voir aussi 8.2**  
>  
> 1. **DeprecationWarning Python 3.12+** : passer un `datetime` à `execute()` est déprécié.  
> 2. **`CURRENT_TIMESTAMP` SQLite est en UTC** : `datetime.now()` Python est en local → un filtre `WHERE timestamp > ?` avec un datetime local **rate des entrées** silencieusement sur les systèmes hors UTC.  
>  
> L'adapter ci-dessous résout les deux problèmes. À enregistrer **une seule fois** au démarrage. Tous les rapports d'audit ci-dessous en dépendent.

```python
import json  
import time  
import sqlite3  
from datetime import datetime, timezone  
from typing import Optional, Dict, Any  

# Adapter datetime → "YYYY-MM-DD HH:MM:SS" en UTC (compatible CURRENT_TIMESTAMP)
# Python 3.12+ requis et normalisation UTC obligatoire pour éviter les bugs silencieux
def _adapter_dt_utc(d):
    if d.tzinfo is None:
        d = d.astimezone(timezone.utc)
    else:
        d = d.astimezone(timezone.utc)
    return d.strftime("%Y-%m-%d %H:%M:%S")
sqlite3.register_adapter(datetime, _adapter_dt_utc)

class AuditManager:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base
        self.utilisateur_actuel = None
        self.session_id = None
        self.adresse_ip = None
        self._creer_tables_audit()

    def _creer_tables_audit(self):
        """Crée les tables d'audit si elles n'existent pas.

        Reprend le DDL présenté plus haut dans le chapitre. Idempotent
        grâce à IF NOT EXISTS.
        """
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()
        cursor.executescript("""
            CREATE TABLE IF NOT EXISTS audit_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                utilisateur_id INTEGER,
                nom_utilisateur TEXT,
                action TEXT NOT NULL,
                table_cible TEXT,
                enregistrement_id INTEGER,
                anciennes_valeurs TEXT,
                nouvelles_valeurs TEXT,
                adresse_ip TEXT,
                user_agent TEXT,
                session_id TEXT,
                duree_ms INTEGER,
                statut TEXT DEFAULT 'SUCCESS',
                message_erreur TEXT
            );
            CREATE TABLE IF NOT EXISTS audit_securite (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                type_evenement TEXT NOT NULL,
                utilisateur_id INTEGER,
                nom_utilisateur TEXT,
                adresse_ip TEXT,
                details TEXT,
                niveau_gravite TEXT DEFAULT 'INFO'
            );
            CREATE TABLE IF NOT EXISTS audit_schema (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                utilisateur_id INTEGER,
                type_operation TEXT,
                nom_objet TEXT,
                definition_sql TEXT
            );
            CREATE INDEX IF NOT EXISTS idx_audit_timestamp ON audit_log(timestamp);
            CREATE INDEX IF NOT EXISTS idx_audit_utilisateur ON audit_log(utilisateur_id);
            CREATE INDEX IF NOT EXISTS idx_audit_table ON audit_log(table_cible);
            CREATE INDEX IF NOT EXISTS idx_audit_securite_timestamp ON audit_securite(timestamp);
            CREATE INDEX IF NOT EXISTS idx_audit_securite_type ON audit_securite(type_evenement);
        """)
        conn.commit()
        conn.close()

    def definir_contexte(self, utilisateur_id: int, nom_utilisateur: str,
                        session_id: str = None, adresse_ip: str = None):
        """Définit le contexte pour les opérations d'audit"""
        self.utilisateur_actuel = {
            'id': utilisateur_id,
            'nom': nom_utilisateur
        }
        self.session_id = session_id
        self.adresse_ip = adresse_ip

    def enregistrer_operation(self, action: str, table_cible: str = None,
                            enregistrement_id: int = None,
                            anciennes_valeurs: Dict = None,
                            nouvelles_valeurs: Dict = None,
                            duree_ms: int = None,
                            statut: str = 'SUCCESS',
                            message_erreur: str = None):
        """Enregistre une opération dans le log d'audit"""

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("""
                INSERT INTO audit_log (
                    utilisateur_id, nom_utilisateur, action, table_cible,
                    enregistrement_id, anciennes_valeurs, nouvelles_valeurs,
                    adresse_ip, session_id, duree_ms, statut, message_erreur
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                self.utilisateur_actuel['id'] if self.utilisateur_actuel else None,
                self.utilisateur_actuel['nom'] if self.utilisateur_actuel else 'SYSTEM',
                action,
                table_cible,
                enregistrement_id,
                json.dumps(anciennes_valeurs) if anciennes_valeurs else None,
                json.dumps(nouvelles_valeurs) if nouvelles_valeurs else None,
                self.adresse_ip,
                self.session_id,
                duree_ms,
                statut,
                message_erreur
            ))

            conn.commit()
            conn.close()

        except Exception as e:
            # En cas d'erreur d'audit, ne pas faire planter l'application
            print(f"Erreur audit: {e}")

    def enregistrer_evenement_securite(self, type_evenement: str,
                                     details: Dict = None,
                                     niveau_gravite: str = 'INFO'):
        """Enregistre un événement de sécurité"""

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            cursor.execute("""
                INSERT INTO audit_securite (
                    type_evenement, utilisateur_id, nom_utilisateur,
                    adresse_ip, details, niveau_gravite
                ) VALUES (?, ?, ?, ?, ?, ?)
            """, (
                type_evenement,
                self.utilisateur_actuel['id'] if self.utilisateur_actuel else None,
                self.utilisateur_actuel['nom'] if self.utilisateur_actuel else 'ANONYMOUS',
                self.adresse_ip,
                json.dumps(details) if details else None,
                niveau_gravite
            ))

            conn.commit()
            conn.close()

        except Exception as e:
            print(f"Erreur audit sécurité: {e}")

    def lire_audit(self, limite: int = 100, filtre_utilisateur: str = None,
                   filtre_action: str = None, date_debut: str = None,
                   date_fin: str = None):
        """Lit les entrées d'audit avec filtres"""

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Construire la requête avec les filtres
        where_clauses = []
        params = []

        if filtre_utilisateur:
            where_clauses.append("nom_utilisateur = ?")
            params.append(filtre_utilisateur)

        if filtre_action:
            where_clauses.append("action = ?")
            params.append(filtre_action)

        if date_debut:
            where_clauses.append("timestamp >= ?")
            params.append(date_debut)

        if date_fin:
            where_clauses.append("timestamp <= ?")
            params.append(date_fin)

        where_clause = ""
        if where_clauses:
            where_clause = "WHERE " + " AND ".join(where_clauses)

        params.append(limite)

        cursor.execute(f"""
            SELECT id, timestamp, nom_utilisateur, action, table_cible,
                   enregistrement_id, anciennes_valeurs, nouvelles_valeurs,
                   adresse_ip, duree_ms, statut, message_erreur
            FROM audit_log
            {where_clause}
            ORDER BY timestamp DESC
            LIMIT ?
        """, params)

        resultats = cursor.fetchall()
        conn.close()

        return resultats

# Exemple d'utilisation basique
audit = AuditManager('ma_base.db')

# Définir le contexte utilisateur
audit.definir_contexte(
    utilisateur_id=1,
    nom_utilisateur='alice',
    session_id='sess_123',
    adresse_ip='192.168.1.100'
)

# Enregistrer une opération
audit.enregistrer_operation(
    action='INSERT',
    table_cible='clients',
    enregistrement_id=42,
    nouvelles_valeurs={'nom': 'Nouveau Client', 'email': 'client@email.com'},
    duree_ms=15
)

# Enregistrer un événement de sécurité
audit.enregistrer_evenement_securite(
    type_evenement='LOGIN',
    details={'methode': 'password', 'succes': True},
    niveau_gravite='INFO'
)
```

## Audit automatique avec des triggers

### Utilisation des triggers SQLite pour l'audit automatique

Les triggers permettent d'automatiser l'audit sans modifier le code applicatif :

> ⚠️ **PIÈGE SQLite critique (vérifié par test)** : un trigger défini dans le schéma `main` **NE PEUT PAS** référencer une table du schéma `temp`. SQLite lève l'erreur :  
> ```  
> trigger audit_clients_insert cannot reference objects in database temp  
> ```  
> Pour avoir une attribution d'audit par session sans race condition, il faut donc utiliser :  
> - une **TEMP TABLE** `session_actuelle` (isolation par connexion ✓)  
> - **ET** un **TEMP TRIGGER** (un trigger temporaire peut référencer une TEMP table ✓)  
>  
> Conséquence : le TEMP TRIGGER doit être recréé à **chaque ouverture de connexion authentifiée** (la TEMP table aussi). Ce n'est pas idéal en termes d'overhead, mais c'est le seul moyen propre d'avoir une attribution par session avec les triggers SQLite.

```sql
-- ── Sur chaque connexion authentifiée, exécuter ce setup une fois ──

-- 1. TEMP TABLE pour l'utilisateur de la connexion en cours (isolation OK)
CREATE TEMP TABLE IF NOT EXISTS session_actuelle (
    nom_utilisateur TEXT
);

-- 2. TEMP TRIGGER d'audit INSERT (peut référencer la TEMP table)
CREATE TEMP TRIGGER IF NOT EXISTS audit_clients_insert  
AFTER INSERT ON clients  
FOR EACH ROW  
BEGIN  
    INSERT INTO audit_log (
        nom_utilisateur, action, table_cible, enregistrement_id,
        nouvelles_valeurs, timestamp
    ) VALUES (
        COALESCE((SELECT nom_utilisateur FROM session_actuelle LIMIT 1), 'SYSTEM'),
        'INSERT',
        'clients',
        NEW.id,
        json_object(
            'nom', NEW.nom,
            'email', NEW.email,
            'telephone', NEW.telephone,
            'date_creation', NEW.date_creation
        ),
        CURRENT_TIMESTAMP
    );
END;

-- 3. TEMP TRIGGER d'audit UPDATE
CREATE TEMP TRIGGER IF NOT EXISTS audit_clients_update  
AFTER UPDATE ON clients  
FOR EACH ROW  
BEGIN  
    INSERT INTO audit_log (
        nom_utilisateur, action, table_cible, enregistrement_id,
        anciennes_valeurs, nouvelles_valeurs, timestamp
    ) VALUES (
        COALESCE((SELECT nom_utilisateur FROM session_actuelle LIMIT 1), 'SYSTEM'),
        'UPDATE',
        'clients',
        NEW.id,
        json_object('nom', OLD.nom, 'email', OLD.email, 'telephone', OLD.telephone),
        json_object('nom', NEW.nom, 'email', NEW.email, 'telephone', NEW.telephone),
        CURRENT_TIMESTAMP
    );
END;

-- 4. TEMP TRIGGER d'audit DELETE
CREATE TEMP TRIGGER IF NOT EXISTS audit_clients_delete  
AFTER DELETE ON clients  
FOR EACH ROW  
BEGIN  
    INSERT INTO audit_log (
        nom_utilisateur, action, table_cible, enregistrement_id,
        anciennes_valeurs, timestamp
    ) VALUES (
        COALESCE((SELECT nom_utilisateur FROM session_actuelle LIMIT 1), 'SYSTEM'),
        'DELETE',
        'clients',
        OLD.id,
        json_object('nom', OLD.nom, 'email', OLD.email, 'telephone', OLD.telephone),
        CURRENT_TIMESTAMP
    );
END;
```

> 💡 **Alternative architecturale si le TEMP TRIGGER par connexion vous semble lourd** :  
> - **Audit applicatif explicite** (via `AuditManager.enregistrer_operation()` plus haut) : pas de magie SQL, attribution garantie, mais il faut penser à appeler la fonction à chaque opération.  
> - **Table `session_actuelle` permanente keyée par PID/UUID** : un trigger persistant lit la session selon un identifiant injecté par l'application via une UDF Python `current_session_id()`. Moins de magie côté SQL, pas de TEMP triggers.

### Classe Python pour gérer l'audit automatique

```python
class AuditAutomatique:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base
        self.audit_manager = AuditManager(chemin_base)

    def definir_utilisateur_session(self, conn, nom_utilisateur: str):
        """Définit l'utilisateur pour les triggers d'audit.

        ⚠️ ARCHITECTURE — pourquoi cette méthode prend une connexion en
           paramètre plutôt que d'en ouvrir une :
           la table `session_actuelle` est une `TEMP TABLE`, donc elle
           n'existe QUE sur la connexion qui l'a créée. Si on ouvrait une
           nouvelle connexion ici, la table créée serait détruite à la
           fermeture, et les requêtes ultérieures (depuis une autre
           connexion) ne la verraient pas → les triggers attribueraient
           toutes les actions à 'SYSTEM'.

           Conséquence pratique : l'application doit garder UNE connexion
           par session utilisateur, créer la table TEMP au moment de
           l'authentification, puis utiliser CETTE connexion pour toutes
           les requêtes de cet utilisateur jusqu'à sa déconnexion.
        """
        cursor = conn.cursor()
        # 1. Créer la table TEMP si elle n'existe pas sur cette connexion
        cursor.execute("""
            CREATE TEMP TABLE IF NOT EXISTS session_actuelle (
                nom_utilisateur TEXT
            )
        """)
        # 2. Réinitialiser et définir l'utilisateur
        cursor.execute("DELETE FROM session_actuelle")
        cursor.execute(
            "INSERT INTO session_actuelle (nom_utilisateur) VALUES (?)",
            (nom_utilisateur,)
        )
        conn.commit()

    def installer_triggers_session(self, conn, nom_table: str, colonnes_a_auditer: list):
        """Installe les TEMP triggers d'audit sur la CONNEXION fournie.

        ⚠️ ARCHITECTURE — pourquoi TEMP TRIGGER plutôt que CREATE TRIGGER :
           SQLite refuse qu'un trigger persistant (schéma `main`) référence
           une table du schéma `temp`. Erreur : « trigger X cannot reference
           objects in database temp ».
           → Pour avoir une attribution par session sans race condition
           multi-utilisateur, on doit utiliser un TEMP TRIGGER, qui peut
           lui référencer la TEMP TABLE `session_actuelle`.

        ⚠️ INJECTION SQL — `CREATE TRIGGER` n'accepte pas `?` pour les
           identifiants. On valide donc chaque nom de table/colonne via une
           regex stricte avant de l'interpoler.

        Cette méthode doit être appelée APRÈS `definir_utilisateur_session`
        sur la même `conn` (sinon `session_actuelle` n'existe pas encore).
        """
        import re
        identifiant_pattern = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
        if not identifiant_pattern.match(nom_table):
            raise ValueError(f"Nom de table invalide : {nom_table!r}")
        for col in colonnes_a_auditer:
            if not identifiant_pattern.match(col):
                raise ValueError(f"Nom de colonne invalide : {col!r}")

        cursor = conn.cursor()
        colonnes_json = ", ".join([f"'{col}', NEW.{col}" for col in colonnes_a_auditer])
        colonnes_json_old = ", ".join([f"'{col}', OLD.{col}" for col in colonnes_a_auditer])

        # CREATE TEMP TRIGGER → trigger attaché à la connexion, supprimé à sa fermeture
        cursor.executescript(f"""
            CREATE TEMP TRIGGER IF NOT EXISTS audit_{nom_table}_insert
            AFTER INSERT ON {nom_table}
            FOR EACH ROW
            BEGIN
                INSERT INTO audit_log (
                    nom_utilisateur, action, table_cible, enregistrement_id,
                    nouvelles_valeurs, timestamp
                ) VALUES (
                    COALESCE((SELECT nom_utilisateur FROM session_actuelle LIMIT 1), 'SYSTEM'),
                    'INSERT', '{nom_table}', NEW.id,
                    json_object({colonnes_json}),
                    CURRENT_TIMESTAMP
                );
            END;

            CREATE TEMP TRIGGER IF NOT EXISTS audit_{nom_table}_update
            AFTER UPDATE ON {nom_table}
            FOR EACH ROW
            BEGIN
                INSERT INTO audit_log (
                    nom_utilisateur, action, table_cible, enregistrement_id,
                    anciennes_valeurs, nouvelles_valeurs, timestamp
                ) VALUES (
                    COALESCE((SELECT nom_utilisateur FROM session_actuelle LIMIT 1), 'SYSTEM'),
                    'UPDATE', '{nom_table}', NEW.id,
                    json_object({colonnes_json_old}),
                    json_object({colonnes_json}),
                    CURRENT_TIMESTAMP
                );
            END;

            CREATE TEMP TRIGGER IF NOT EXISTS audit_{nom_table}_delete
            AFTER DELETE ON {nom_table}
            FOR EACH ROW
            BEGIN
                INSERT INTO audit_log (
                    nom_utilisateur, action, table_cible, enregistrement_id,
                    anciennes_valeurs, timestamp
                ) VALUES (
                    COALESCE((SELECT nom_utilisateur FROM session_actuelle LIMIT 1), 'SYSTEM'),
                    'DELETE', '{nom_table}', OLD.id,
                    json_object({colonnes_json_old}),
                    CURRENT_TIMESTAMP
                );
            END;
        """)
        conn.commit()
        print(f"✅ TEMP triggers d'audit installés pour '{nom_table}' sur cette connexion")

    def supprimer_triggers_audit_table(self, conn, nom_table: str):
        """Supprime les TEMP triggers d'audit pour une table sur la connexion fournie.

        Note : les TEMP triggers sont automatiquement détruits à la fermeture
        de la connexion. Cette méthode n'est utile que pour les désactiver
        en cours de session (ex. import de masse où on veut court-circuiter
        l'audit).
        """
        cursor = conn.cursor()
        for suffixe in ('insert', 'update', 'delete'):
            cursor.execute(f"DROP TRIGGER IF EXISTS audit_{nom_table}_{suffixe}")
        conn.commit()
        print(f"✅ TEMP triggers d'audit supprimés pour '{nom_table}'")

# ─── Exemple d'utilisation complet — UNE connexion par session utilisateur ───
audit_auto = AuditAutomatique('ma_base.db')

# 1. Ouvrir la connexion qui servira toute la session d'Alice
conn = sqlite3.connect('ma_base.db')

# 2. Définir l'utilisateur (crée la TEMP TABLE session_actuelle sur conn)
audit_auto.definir_utilisateur_session(conn, 'alice')

# 3. Installer les TEMP triggers sur conn (DOIT venir APRÈS définir_utilisateur_session
#    car les triggers référencent session_actuelle qui doit déjà exister)
audit_auto.installer_triggers_session(
    conn,
    'clients',
    ['nom', 'email', 'telephone', 'date_creation']
)

# 4. Toutes les modifications faites via CETTE connexion sont auditées comme 'alice'
cursor = conn.cursor()  
cursor.execute("""  
    INSERT INTO clients (nom, email, telephone)
    VALUES ('Test Client', 'test@email.com', '0123456789')
""")
conn.commit()

# 5. Fin de session : fermer la connexion détruit TEMP TABLE + TEMP TRIGGERS
conn.close()

print("✅ Insertion effectuée et auditée comme 'alice'")
```

## Logging des performances et erreurs

### Système de logging des performances

```python
import time  
import logging  
from contextlib import contextmanager  
from functools import wraps  

class SQLitePerformanceLogger:
    def __init__(self, chemin_base: str, seuil_lent_ms: int = 1000):
        self.chemin_base = chemin_base
        self.seuil_lent_ms = seuil_lent_ms
        self.audit_manager = AuditManager(chemin_base)

        # Configuration du logging
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler('sqlite_performance.log'),
                logging.StreamHandler()
            ]
        )
        self.logger = logging.getLogger('SQLitePerformance')

    @contextmanager
    def mesurer_requete(self, requete_sql: str, description: str = ""):
        """Context manager pour mesurer le temps d'exécution d'une requête"""
        debut = time.time()
        debut_ms = int(time.time() * 1000)

        try:
            yield

        finally:
            fin = time.time()
            duree_ms = int((fin - debut) * 1000)

            # Logger les requêtes lentes
            if duree_ms > self.seuil_lent_ms:
                self.logger.warning(
                    f"REQUÊTE LENTE ({duree_ms}ms): {description}\nSQL: {requete_sql[:200]}..."
                )

                # Enregistrer dans l'audit
                self.audit_manager.enregistrer_operation(
                    action='SLOW_QUERY',
                    table_cible='performance',
                    nouvelles_valeurs={
                        'sql': requete_sql[:500],
                        'description': description,
                        'duree_ms': duree_ms
                    },
                    duree_ms=duree_ms,
                    statut='WARNING'
                )
            else:
                self.logger.debug(f"Requête OK ({duree_ms}ms): {description}")

    def decorator_mesure_performance(self, description: str = ""):
        """Décorateur pour mesurer automatiquement les performances"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                avec_description = description or f"{func.__name__}"

                debut = time.time()

                try:
                    resultat = func(*args, **kwargs)
                    statut = 'SUCCESS'
                    erreur = None
                except Exception as e:
                    resultat = None
                    statut = 'ERROR'
                    erreur = str(e)
                    raise
                finally:
                    fin = time.time()
                    duree_ms = int((fin - debut) * 1000)

                    # Logger la performance
                    if statut == 'ERROR':
                        self.logger.error(f"ERREUR ({duree_ms}ms): {avec_description} - {erreur}")
                    elif duree_ms > self.seuil_lent_ms:
                        self.logger.warning(f"LENT ({duree_ms}ms): {avec_description}")
                    else:
                        self.logger.debug(f"OK ({duree_ms}ms): {avec_description}")

                    # Enregistrer dans l'audit
                    self.audit_manager.enregistrer_operation(
                        action='FUNCTION_CALL',
                        table_cible='performance',
                        nouvelles_valeurs={
                            'fonction': func.__name__,
                            'description': avec_description,
                            'args': str(args)[:200],
                            'kwargs': str(kwargs)[:200]
                        },
                        duree_ms=duree_ms,
                        statut=statut,
                        message_erreur=erreur
                    )

                return resultat
            return wrapper
        return decorator

# Exemple d'utilisation du logging de performance
perf_logger = SQLitePerformanceLogger('ma_base.db', seuil_lent_ms=500)

# Utilisation avec context manager
def rechercher_clients_complexe(terme):
    conn = sqlite3.connect('ma_base.db')
    cursor = conn.cursor()

    requete = """
        SELECT c.*, COUNT(co.id) as nb_commandes
        FROM clients c
        LEFT JOIN commandes co ON c.id = co.client_id
        WHERE c.nom LIKE ? OR c.email LIKE ?
        GROUP BY c.id
        ORDER BY nb_commandes DESC
    """

    with perf_logger.mesurer_requete(requete, "Recherche clients avec commandes"):
        cursor.execute(requete, (f'%{terme}%', f'%{terme}%'))
        resultats = cursor.fetchall()

    conn.close()
    return resultats

# Utilisation avec décorateur
@perf_logger.decorator_mesure_performance("Création client avec validation")
def creer_client_complet(nom, email, telephone):
    """Crée un client avec validation complète"""
    conn = sqlite3.connect('ma_base.db')
    cursor = conn.cursor()

    # Validation email unique
    cursor.execute("SELECT id FROM clients WHERE email = ?", (email,))
    if cursor.fetchone():
        raise ValueError("Email déjà utilisé")

    # Insertion
    cursor.execute("""
        INSERT INTO clients (nom, email, telephone, date_creation)
        VALUES (?, ?, ?, CURRENT_TIMESTAMP)
    """, (nom, email, telephone))

    client_id = cursor.lastrowid
    conn.commit()
    conn.close()

    return client_id

# Test des fonctions avec logging automatique
print("=== TEST LOGGING PERFORMANCE ===")

# Test recherche (devrait être rapide)
resultats = rechercher_clients_complexe("test")

# Test création (devrait être rapide)
try:
    nouveau_client = creer_client_complet("Test User", "test@example.com", "0123456789")
    print(f"Client créé avec ID: {nouveau_client}")
except Exception as e:
    print(f"Erreur création client: {e}")
```

## Analyse et reporting des logs d'audit

### Générateur de rapports d'audit

```python
import sqlite3  
import json  
from datetime import datetime, timedelta  
from collections import defaultdict  

class RapportAudit:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base

    def generer_rapport_activite(self, jours: int = 7):
        """Génère un rapport d'activité pour les X derniers jours"""
        print(f"📊 RAPPORT D'ACTIVITÉ - {jours} derniers jours")
        print("=" * 60)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(days=jours)

        # 1. Statistiques générales
        cursor.execute("""
            SELECT COUNT(*) as total_operations,
                   COUNT(DISTINCT utilisateur_id) as utilisateurs_actifs,
                   COUNT(DISTINCT table_cible) as tables_modifiees
            FROM audit_log
            WHERE timestamp > ?
        """, (date_debut,))

        stats = cursor.fetchone()
        print(f"📈 Statistiques générales:")
        print(f"   • Total opérations: {stats[0]:,}")
        print(f"   • Utilisateurs actifs: {stats[1]}")
        print(f"   • Tables modifiées: {stats[2]}")
        print()

        # 2. Répartition par action
        cursor.execute("""
            SELECT action, COUNT(*) as count
            FROM audit_log
            WHERE timestamp > ?
            GROUP BY action
            ORDER BY count DESC
        """, (date_debut,))

        print("🔄 Répartition par type d'action:")
        for action, count in cursor.fetchall():
            print(f"   • {action}: {count:,}")
        print()

        # 3. Utilisateurs les plus actifs
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(*) as operations
            FROM audit_log
            WHERE timestamp > ? AND nom_utilisateur IS NOT NULL
            GROUP BY nom_utilisateur
            ORDER BY operations DESC
            LIMIT 10
        """, (date_debut,))

        print("👤 Utilisateurs les plus actifs:")
        for utilisateur, operations in cursor.fetchall():
            print(f"   • {utilisateur}: {operations:,} opérations")
        print()

        # 4. Tables les plus modifiées
        cursor.execute("""
            SELECT table_cible, COUNT(*) as modifications
            FROM audit_log
            WHERE timestamp > ? AND table_cible IS NOT NULL
            GROUP BY table_cible
            ORDER BY modifications DESC
            LIMIT 10
        """, (date_debut,))

        print("📋 Tables les plus modifiées:")
        for table, modifications in cursor.fetchall():
            print(f"   • {table}: {modifications:,} modifications")
        print()

        # 5. Erreurs et problèmes
        cursor.execute("""
            SELECT statut, COUNT(*) as count
            FROM audit_log
            WHERE timestamp > ? AND statut != 'SUCCESS'
            GROUP BY statut
        """, (date_debut,))

        erreurs = cursor.fetchall()
        if erreurs:
            print("⚠️ Erreurs détectées:")
            for statut, count in erreurs:
                print(f"   • {statut}: {count}")
        else:
            print("✅ Aucune erreur détectée")

        conn.close()

    def generer_rapport_securite(self, jours: int = 30):
        """Génère un rapport de sécurité"""
        print(f"\n🔐 RAPPORT DE SÉCURITÉ - {jours} derniers jours")
        print("=" * 60)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(days=jours)

        # 1. Événements de sécurité
        cursor.execute("""
            SELECT type_evenement, niveau_gravite, COUNT(*) as count
            FROM audit_securite
            WHERE timestamp > ?
            GROUP BY type_evenement, niveau_gravite
            ORDER BY
                CASE niveau_gravite
                    WHEN 'CRITICAL' THEN 1
                    WHEN 'WARNING' THEN 2
                    ELSE 3
                END,
                count DESC
        """, (date_debut,))

        print("🚨 Événements de sécurité:")
        for event_type, niveau, count in cursor.fetchall():
            emoji = "🔴" if niveau == "CRITICAL" else "🟡" if niveau == "WARNING" else "🟢"
            print(f"   {emoji} {event_type} ({niveau}): {count}")
        print()

        # 2. Tentatives de connexion échouées
        cursor.execute("""
            SELECT adresse_ip, COUNT(*) as tentatives
            FROM audit_securite
            WHERE timestamp > ?
              AND type_evenement = 'FAILED_LOGIN'
            GROUP BY adresse_ip
            HAVING tentatives > 3
            ORDER BY tentatives DESC
        """, (date_debut,))

        tentatives_suspectes = cursor.fetchall()
        if tentatives_suspectes:
            print("🚨 Adresses IP suspectes (multiples échecs de connexion):")
            for ip, tentatives in tentatives_suspectes:
                print(f"   • {ip}: {tentatives} tentatives échouées")
        else:
            print("✅ Aucune activité suspecte de connexion détectée")
        print()

        # 3. Accès aux données sensibles
        cursor.execute("""
            SELECT nom_utilisateur, table_cible, COUNT(*) as acces
            FROM audit_log
            WHERE timestamp > ?
              AND table_cible IN ('utilisateurs', 'salaires', 'donnees_personnelles')
              AND action = 'SELECT'
            GROUP BY nom_utilisateur, table_cible
            ORDER BY acces DESC
        """, (date_debut,))

        acces_sensibles = cursor.fetchall()
        if acces_sensibles:
            print("🔍 Accès aux données sensibles:")
            for utilisateur, table, acces in acces_sensibles:
                print(f"   • {utilisateur} → {table}: {acces} consultations")
        else:
            print("ℹ️ Aucun accès aux données sensibles enregistré")
        print()

        # 4. Modifications en dehors des heures de bureau
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(*) as modifications_nocturnes
            FROM audit_log
            WHERE timestamp > ?
              AND action IN ('INSERT', 'UPDATE', 'DELETE')
              AND (CAST(strftime('%H', timestamp) AS INTEGER) < 8
                   OR CAST(strftime('%H', timestamp) AS INTEGER) > 18)
            GROUP BY nom_utilisateur
            ORDER BY modifications_nocturnes DESC
        """, (date_debut,))

        modifications_nocturnes = cursor.fetchall()
        if modifications_nocturnes:
            print("🌙 Modifications en dehors des heures de bureau:")
            for utilisateur, count in modifications_nocturnes:
                print(f"   • {utilisateur}: {count} modifications nocturnes")
        else:
            print("✅ Toutes les modifications ont eu lieu en heures de bureau")

        conn.close()

    def generer_rapport_performance(self, jours: int = 7):
        """Génère un rapport de performance"""
        print(f"\n⚡ RAPPORT DE PERFORMANCE - {jours} derniers jours")
        print("=" * 60)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(days=jours)

        # 1. Requêtes les plus lentes
        cursor.execute("""
            SELECT table_cible, action, AVG(duree_ms) as duree_moyenne,
                   MAX(duree_ms) as duree_max, COUNT(*) as occurrences
            FROM audit_log
            WHERE timestamp > ?
              AND duree_ms IS NOT NULL
              AND duree_ms > 100
            GROUP BY table_cible, action
            ORDER BY duree_moyenne DESC
            LIMIT 10
        """, (date_debut,))

        print("🐌 Opérations les plus lentes (>100ms):")
        for table, action, moy, max_duree, count in cursor.fetchall():
            print(f"   • {table}.{action}: {moy:.1f}ms moyenne, {max_duree}ms max ({count} fois)")
        print()

        # 2. Volume d'opérations par heure
        cursor.execute("""
            SELECT strftime('%H', timestamp) as heure, COUNT(*) as operations
            FROM audit_log
            WHERE timestamp > ?
            GROUP BY strftime('%H', timestamp)
            ORDER BY heure
        """, (date_debut,))

        print("📊 Répartition par heure de la journée:")
        for heure, operations in cursor.fetchall():
            barre = "█" * min(int(operations / 10), 20)  # Graphique simple
            print(f"   {heure}h: {operations:4} ops {barre}")
        print()

        # 3. Erreurs de performance
        cursor.execute("""
            SELECT COUNT(*) as requetes_lentes
            FROM audit_log
            WHERE timestamp > ? AND duree_ms > 1000
        """, (date_debut,))

        requetes_lentes = cursor.fetchone()[0]
        if requetes_lentes > 0:
            print(f"⚠️ {requetes_lentes} requêtes très lentes (>1s) détectées")
        else:
            print("✅ Aucune requête très lente détectée")

        conn.close()

    def detecter_anomalies(self):
        """Détecte des comportements anormaux dans les logs"""
        print("\n🔍 DÉTECTION D'ANOMALIES")
        print("=" * 60)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        anomalies = []

        # 1. Utilisateurs avec activité inhabituelle
        cursor.execute("""
            WITH stats_utilisateur AS (
                SELECT nom_utilisateur,
                       COUNT(*) as operations_aujourd_hui,
                       AVG(COUNT(*)) OVER() as moyenne_generale
                FROM audit_log
                WHERE date(timestamp) = date('now')
                GROUP BY nom_utilisateur
            )
            SELECT nom_utilisateur, operations_aujourd_hui, moyenne_generale
            FROM stats_utilisateur
            WHERE operations_aujourd_hui > moyenne_generale * 3
        """)

        utilisateurs_hyperactifs = cursor.fetchall()
        if utilisateurs_hyperactifs:
            for user, ops, moyenne in utilisateurs_hyperactifs:
                anomalies.append(f"Utilisateur {user}: {ops} opérations (moyenne: {moyenne:.1f})")

        # 2. Suppressions massives
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(*) as suppressions
            FROM audit_log
            WHERE action = 'DELETE'
              AND date(timestamp) = date('now')
            GROUP BY nom_utilisateur
            HAVING suppressions > 10
        """)

        suppressions_massives = cursor.fetchall()
        if suppressions_massives:
            for user, count in suppressions_massives:
                anomalies.append(f"Suppressions massives par {user}: {count} enregistrements")

        # 3. Connexions multiples depuis différentes IP pour le même utilisateur
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(DISTINCT adresse_ip) as ips_differentes
            FROM audit_securite
            WHERE type_evenement = 'LOGIN'
              AND date(timestamp) = date('now')
            GROUP BY nom_utilisateur
            HAVING ips_differentes > 2
        """)

        connexions_multiples = cursor.fetchall()
        if connexions_multiples:
            for user, ips in connexions_multiples:
                anomalies.append(f"Connexions multiples pour {user}: {ips} adresses IP différentes")

        # 4. Modifications de données en cascade (possibles scripts automatiques)
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(*) as modifications_rapides
            FROM audit_log
            WHERE action IN ('INSERT', 'UPDATE', 'DELETE')
              AND timestamp > datetime('now', '-1 hour')
            GROUP BY nom_utilisateur
            HAVING modifications_rapides > 50
        """)

        modifications_rapides = cursor.fetchall()
        if modifications_rapides:
            for user, count in modifications_rapides:
                anomalies.append(f"Modifications très fréquentes par {user}: {count} en 1 heure")

        # Afficher les résultats
        if anomalies:
            print("🚨 Anomalies détectées:")
            for anomalie in anomalies:
                print(f"   • {anomalie}")
            print("\n💡 Recommandations:")
            print("   - Vérifier l'activité de ces utilisateurs")
            print("   - Analyser les logs détaillés")
            print("   - Contacter les utilisateurs si nécessaire")
        else:
            print("✅ Aucune anomalie détectée")

        conn.close()
        return anomalies

# Utilisation des rapports
rapport = RapportAudit('ma_base.db')

print("=== GÉNÉRATION DES RAPPORTS D'AUDIT ===")  
rapport.generer_rapport_activite(7)  
rapport.generer_rapport_securite(30)  
rapport.generer_rapport_performance(7)  
anomalies = rapport.detecter_anomalies()  
```

## Intégration avec des outils de monitoring externes

### Export vers des formats standards

```python
import csv  
import json  
import sqlite3  
from datetime import datetime, timedelta  

class ExporteurAudit:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base

    def exporter_csv(self, nom_fichier: str, jours: int = 30):
        """Exporte les logs d'audit en CSV"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(days=jours)

        cursor.execute("""
            SELECT timestamp, nom_utilisateur, action, table_cible,
                   enregistrement_id, adresse_ip, duree_ms, statut
            FROM audit_log
            WHERE timestamp > ?
            ORDER BY timestamp DESC
        """, (date_debut,))

        with open(nom_fichier, 'w', newline='', encoding='utf-8') as csvfile:
            writer = csv.writer(csvfile)

            # En-têtes
            writer.writerow([
                'Timestamp', 'Utilisateur', 'Action', 'Table',
                'ID_Enregistrement', 'Adresse_IP', 'Duree_ms', 'Statut'
            ])

            # Données
            for row in cursor.fetchall():
                writer.writerow(row)

        conn.close()
        print(f"✅ Export CSV créé: {nom_fichier}")

    def exporter_json(self, nom_fichier: str, jours: int = 30):
        """Exporte les logs d'audit en JSON"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(days=jours)

        cursor.execute("""
            SELECT id, timestamp, nom_utilisateur, action, table_cible,
                   enregistrement_id, anciennes_valeurs, nouvelles_valeurs,
                   adresse_ip, session_id, duree_ms, statut, message_erreur
            FROM audit_log
            WHERE timestamp > ?
            ORDER BY timestamp DESC
        """, (date_debut,))

        logs = []
        for row in cursor.fetchall():
            log_entry = {
                'id': row[0],
                'timestamp': row[1],
                'utilisateur': row[2],
                'action': row[3],
                'table': row[4],
                'enregistrement_id': row[5],
                'anciennes_valeurs': json.loads(row[6]) if row[6] else None,
                'nouvelles_valeurs': json.loads(row[7]) if row[7] else None,
                'adresse_ip': row[8],
                'session_id': row[9],
                'duree_ms': row[10],
                'statut': row[11],
                'message_erreur': row[12]
            }
            logs.append(log_entry)

        with open(nom_fichier, 'w', encoding='utf-8') as jsonfile:
            json.dump({
                'export_date': datetime.now().isoformat(),
                'periode_jours': jours,
                'total_entries': len(logs),
                'logs': logs
            }, jsonfile, indent=2, ensure_ascii=False)

        conn.close()
        print(f"✅ Export JSON créé: {nom_fichier}")

    def generer_metriques_prometheus(self, nom_fichier: str):
        """Génère des métriques au format Prometheus"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Métriques des dernières 24h
        cursor.execute("""
            SELECT action, COUNT(*) as count
            FROM audit_log
            WHERE timestamp > datetime('now', '-1 day')
            GROUP BY action
        """)

        metriques = []

        # Métrique du nombre d'opérations par type
        for action, count in cursor.fetchall():
            metriques.append(f'sqlite_operations_total{{action="{action.lower()}"}} {count}')

        # Métrique des erreurs
        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE timestamp > datetime('now', '-1 day') AND statut != 'SUCCESS'
        """)
        erreurs = cursor.fetchone()[0]
        metriques.append(f'sqlite_errors_total {erreurs}')

        # Métrique du temps de réponse moyen
        cursor.execute("""
            SELECT AVG(duree_ms) FROM audit_log
            WHERE timestamp > datetime('now', '-1 day') AND duree_ms IS NOT NULL
        """)
        duree_moyenne = cursor.fetchone()[0] or 0
        metriques.append(f'sqlite_response_time_avg_ms {duree_moyenne:.2f}')

        # Métrique des utilisateurs actifs
        cursor.execute("""
            SELECT COUNT(DISTINCT nom_utilisateur) FROM audit_log
            WHERE timestamp > datetime('now', '-1 day')
        """)
        utilisateurs_actifs = cursor.fetchone()[0]
        metriques.append(f'sqlite_active_users {utilisateurs_actifs}')

        with open(nom_fichier, 'w') as f:
            f.write("# Métriques SQLite\n")
            f.write("# HELP sqlite_operations_total Nombre total d'opérations par type\n")
            f.write("# TYPE sqlite_operations_total counter\n")
            for metrique in metriques:
                f.write(metrique + '\n')

        conn.close()
        print(f"✅ Métriques Prometheus créées: {nom_fichier}")

# Utilisation des exportateurs
exporteur = ExporteurAudit('ma_base.db')

print("=== EXPORT DES DONNÉES D'AUDIT ===")  
exporteur.exporter_csv('audit_export.csv', 30)  
exporteur.exporter_json('audit_export.json', 30)  
exporteur.generer_metriques_prometheus('sqlite_metrics.prom')  
```

### Intégration avec des systèmes de log centralisés

```python
import requests  
import sqlite3  
import json  
from datetime import datetime  

# ⚠️ `syslog` est UNIX-only (Linux/macOS). Sur Windows, importer conditionnellement
#     ou utiliser `logging.handlers.SysLogHandler` (TCP/UDP vers serveur syslog distant).
try:
    import syslog
    SYSLOG_DISPONIBLE = True
except ImportError:
    SYSLOG_DISPONIBLE = False

class IntegrationMonitoring:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base
        self.audit_manager = AuditManager(chemin_base)

    def envoyer_vers_elastic(self, url_elastic: str, index_name: str = "sqlite-audit"):
        """Envoie les logs vers Elasticsearch"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Récupérer les logs non envoyés (supposons qu'on ait un flag)
        cursor.execute("""
            SELECT id, timestamp, nom_utilisateur, action, table_cible,
                   enregistrement_id, anciennes_valeurs, nouvelles_valeurs,
                   adresse_ip, duree_ms, statut
            FROM audit_log
            WHERE timestamp > datetime('now', '-1 hour')
            ORDER BY timestamp ASC
        """)

        logs_envoyes = 0

        for log_entry in cursor.fetchall():
            doc = {
                "@timestamp": log_entry[1],
                "user": log_entry[2],
                "action": log_entry[3],
                "table": log_entry[4],
                "record_id": log_entry[5],
                "old_values": json.loads(log_entry[6]) if log_entry[6] else None,
                "new_values": json.loads(log_entry[7]) if log_entry[7] else None,
                "ip_address": log_entry[8],
                "duration_ms": log_entry[9],
                "status": log_entry[10],
                "source": "sqlite_audit"
            }

            try:
                response = requests.post(
                    f"{url_elastic}/{index_name}/_doc",
                    json=doc,
                    headers={"Content-Type": "application/json"}
                )

                if response.status_code in [200, 201]:
                    logs_envoyes += 1
                else:
                    print(f"Erreur Elasticsearch: {response.status_code}")

            except Exception as e:
                print(f"Erreur envoi Elasticsearch: {e}")

        conn.close()
        print(f"✅ {logs_envoyes} logs envoyés vers Elasticsearch")

    def envoyer_vers_syslog(self, serveur_syslog: str = 'localhost', port: int = 514):
        """Envoie les événements critiques vers syslog (Unix uniquement).

        ⚠️ Note : le module `syslog` est UNIX-only. Sur Windows, utiliser
        `logging.handlers.SysLogHandler` pour envoyer en TCP/UDP vers un
        serveur syslog distant (plus portable de toute façon).
        """
        if not SYSLOG_DISPONIBLE:
            print("⚠️ Module syslog non disponible (Windows ?). "
                  "Utilisez logging.handlers.SysLogHandler à la place.")
            return
        syslog.openlog("sqlite_audit", syslog.LOG_PID, syslog.LOG_LOCAL0)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Événements critiques des dernières minutes
        cursor.execute("""
            SELECT type_evenement, nom_utilisateur, adresse_ip, details
            FROM audit_securite
            WHERE timestamp > datetime('now', '-5 minutes')
              AND niveau_gravite = 'CRITICAL'
        """)

        for event_type, user, ip, details in cursor.fetchall():
            message = f"SQLITE_CRITICAL: {event_type} by {user} from {ip}"
            if details:
                message += f" - {details}"

            syslog.syslog(syslog.LOG_CRIT, message)

        syslog.closelog()
        conn.close()

    def webhook_alertes(self, webhook_url: str, seuil_erreurs: int = 5):
        """Envoie des alertes via webhook (Slack, Teams, etc.)"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Compter les erreurs récentes
        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE timestamp > datetime('now', '-15 minutes')
              AND statut != 'SUCCESS'
        """)

        erreurs_recentes = cursor.fetchone()[0]

        if erreurs_recentes >= seuil_erreurs:
            # Récupérer les détails des erreurs
            cursor.execute("""
                SELECT action, table_cible, message_erreur, COUNT(*) as count
                FROM audit_log
                WHERE timestamp > datetime('now', '-15 minutes')
                  AND statut != 'SUCCESS'
                GROUP BY action, table_cible, message_erreur
                ORDER BY count DESC
            """)

            erreurs_detail = cursor.fetchall()

            # Construire le message d'alerte
            message = {
                "text": f"🚨 ALERTE SQLite: {erreurs_recentes} erreurs détectées",
                "attachments": [{
                    "color": "danger",
                    "fields": [
                        {
                            "title": "Période",
                            "value": "15 dernières minutes",
                            "short": True
                        },
                        {
                            "title": "Total erreurs",
                            "value": str(erreurs_recentes),
                            "short": True
                        }
                    ]
                }]
            }

            # Ajouter les détails des erreurs
            details_text = ""
            for action, table, erreur, count in erreurs_detail[:5]:  # Limiter à 5
                details_text += f"• {action} sur {table}: {count} fois\\n"
                if erreur:
                    details_text += f"  Erreur: {erreur[:100]}...\\n"

            message["attachments"][0]["fields"].append({
                "title": "Détails des erreurs",
                "value": details_text,
                "short": False
            })

            try:
                response = requests.post(webhook_url, json=message)
                if response.status_code == 200:
                    print("✅ Alerte envoyée via webhook")
                else:
                    print(f"❌ Erreur envoi webhook: {response.status_code}")
            except Exception as e:
                print(f"❌ Erreur webhook: {e}")

        conn.close()

# Configuration et utilisation du monitoring
monitoring = IntegrationMonitoring('ma_base.db')

# Exemple d'utilisation
print("=== INTÉGRATION MONITORING EXTERNE ===")

# Note: Ces URL sont des exemples, adaptez selon votre infrastructure
# monitoring.envoyer_vers_elastic('http://localhost:9200', 'sqlite-audit')
# monitoring.envoyer_vers_syslog('log-server.example.com', 514)
# monitoring.webhook_alertes('https://hooks.slack.com/services/YOUR/WEBHOOK/URL')

print("Configuration monitoring disponible - adaptez les URL selon votre infrastructure")
```

## Maintenance et optimisation des logs

### Nettoyage automatique des logs

```python
from datetime import datetime, timedelta  
import os  

class MaintenanceLogs:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base

    def nettoyer_logs_anciens(self, jours_retention: int = 90):
        """Supprime les logs plus anciens que la période de rétention"""
        print(f"🧹 Nettoyage des logs de plus de {jours_retention} jours...")

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_limite = datetime.now() - timedelta(days=jours_retention)

        # Compter les logs à supprimer
        cursor.execute("SELECT COUNT(*) FROM audit_log WHERE timestamp < ?", (date_limite,))
        logs_a_supprimer = cursor.fetchone()[0]

        cursor.execute("SELECT COUNT(*) FROM audit_securite WHERE timestamp < ?", (date_limite,))
        securite_a_supprimer = cursor.fetchone()[0]

        if logs_a_supprimer > 0 or securite_a_supprimer > 0:
            print(f"   📊 {logs_a_supprimer} entrées audit_log à supprimer")
            print(f"   🔐 {securite_a_supprimer} entrées audit_securite à supprimer")

            # Effectuer le nettoyage
            cursor.execute("DELETE FROM audit_log WHERE timestamp < ?", (date_limite,))
            cursor.execute("DELETE FROM audit_securite WHERE timestamp < ?", (date_limite,))

            conn.commit()
            print("   ✅ Nettoyage terminé")
        else:
            print("   ℹ️ Aucun log ancien à supprimer")

        conn.close()

    def archiver_logs_anciens(self, jours_archive: int = 365, chemin_archive: str = "archive"):
        """Archive les logs anciens dans des fichiers séparés"""
        print(f"📦 Archivage des logs de plus de {jours_archive} jours...")

        os.makedirs(chemin_archive, exist_ok=True)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_limite = datetime.now() - timedelta(days=jours_archive)
        timestamp_archive = datetime.now().strftime('%Y%m%d_%H%M%S')

        # Archive des logs d'audit
        fichier_archive = os.path.join(chemin_archive, f"audit_log_{timestamp_archive}.json")

        cursor.execute("""
            SELECT * FROM audit_log WHERE timestamp < ?
            ORDER BY timestamp
        """, (date_limite,))

        logs_archives = []
        for row in cursor.fetchall():
            logs_archives.append({
                'id': row[0],
                'timestamp': row[1],
                'utilisateur_id': row[2],
                'nom_utilisateur': row[3],
                'action': row[4],
                'table_cible': row[5],
                'enregistrement_id': row[6],
                'anciennes_valeurs': json.loads(row[7]) if row[7] else None,
                'nouvelles_valeurs': json.loads(row[8]) if row[8] else None,
                'adresse_ip': row[9],
                'session_id': row[10],
                'duree_ms': row[11],
                'statut': row[12],
                'message_erreur': row[13]
            })

        if logs_archives:
            with open(fichier_archive, 'w', encoding='utf-8') as f:
                json.dump({
                    'archive_date': datetime.now().isoformat(),
                    'total_entries': len(logs_archives),
                    'logs': logs_archives
                }, f, indent=2, ensure_ascii=False)

            # Supprimer les logs archivés
            cursor.execute("DELETE FROM audit_log WHERE timestamp < ?", (date_limite,))

            print(f"   ✅ {len(logs_archives)} logs archivés dans {fichier_archive}")
        else:
            print("   ℹ️ Aucun log à archiver")

        conn.commit()
        conn.close()

    def optimiser_base_audit(self):
        """Optimise la base de données d'audit"""
        print("⚡ Optimisation de la base d'audit...")

        # ⚠️ VACUUM ne peut PAS s'exécuter dans une transaction. Le module
        #    sqlite3 de Python ouvre une transaction implicite par défaut
        #    (isolation_level=""). Solution : passer isolation_level=None
        #    OU faire conn.commit() juste avant chaque VACUUM.
        conn = sqlite3.connect(self.chemin_base, isolation_level=None)
        cursor = conn.cursor()

        # Analyser la taille avant optimisation
        page_count = cursor.execute("PRAGMA page_count").fetchone()[0]
        page_size = cursor.execute("PRAGMA page_size").fetchone()[0]
        taille_avant = page_count * page_size

        # Exécuter l'optimisation
        # ⚠️ VACUUM nécessite ~2× l'espace disque libre (réécrit toute la base)
        #    et acquiert un verrou exclusif. À éviter sur très grosses bases.
        print("   🔄 VACUUM en cours...")
        cursor.execute("VACUUM")

        print("   📊 ANALYZE en cours...")
        cursor.execute("ANALYZE")

        print("   ⚙️ Optimisation automatique...")
        cursor.execute("PRAGMA optimize")

        # Analyser la taille après optimisation
        page_count = cursor.execute("PRAGMA page_count").fetchone()[0]
        taille_apres = page_count * page_size

        espace_libere = taille_avant - taille_apres
        pourcentage = (espace_libere / taille_avant) * 100 if taille_avant > 0 else 0

        print(f"   ✅ Optimisation terminée")
        print(f"   📉 Espace libéré: {espace_libere / 1024 / 1024:.2f} MB ({pourcentage:.1f}%)")

        conn.close()

    def verifier_integrite_logs(self):
        """Vérifie l'intégrité des tables de logs"""
        print("🔍 Vérification de l'intégrité des logs...")

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Vérification générale
        # ⚠️ PRAGMA integrity_check peut retourner PLUSIEURS lignes en cas
        #    de corruption (une par problème détecté, jusqu'à 100 par défaut).
        #    Il faut donc utiliser fetchall(), sinon les corruptions suivantes
        #    sont ignorées. Une base saine renvoie une seule ligne 'ok'.
        cursor.execute("PRAGMA integrity_check")
        lignes_integrite = [row[0] for row in cursor.fetchall()]

        if lignes_integrite == ['ok']:
            print("   ✅ Intégrité générale: OK")
        else:
            print(f"   ❌ Problèmes d'intégrité ({len(lignes_integrite)} signalements) :")
            for ligne in lignes_integrite:
                print(f"      → {ligne}")

        # Vérifications spécifiques aux logs
        verifications = []

        # 1. Logs orphelins (utilisateur supprimé)
        cursor.execute("""
            SELECT COUNT(*) FROM audit_log al
            LEFT JOIN utilisateurs u ON al.utilisateur_id = u.id
            WHERE al.utilisateur_id IS NOT NULL AND u.id IS NULL
        """)
        logs_orphelins = cursor.fetchone()[0]
        if logs_orphelins > 0:
            verifications.append(f"⚠️ {logs_orphelins} logs avec utilisateurs inexistants")

        # 2. Logs avec timestamps futurs
        cursor.execute("SELECT COUNT(*) FROM audit_log WHERE timestamp > datetime('now')")
        logs_futurs = cursor.fetchone()[0]
        if logs_futurs > 0:
            verifications.append(f"⚠️ {logs_futurs} logs avec timestamp futur")

        # 3. Logs avec JSON invalide
        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE (anciennes_valeurs IS NOT NULL AND json_valid(anciennes_valeurs) = 0)
               OR (nouvelles_valeurs IS NOT NULL AND json_valid(nouvelles_valeurs) = 0)
        """)
        json_invalides = cursor.fetchone()[0]
        if json_invalides > 0:
            verifications.append(f"⚠️ {json_invalides} logs avec JSON invalide")

        if verifications:
            print("   🔍 Problèmes détectés:")
            for probleme in verifications:
                print(f"      {probleme}")
        else:
            print("   ✅ Tous les contrôles spécifiques sont OK")

        conn.close()

# Classe pour automatiser la maintenance
class SchedulerMaintenance:
    def __init__(self, chemin_base: str):
        self.maintenance = MaintenanceLogs(chemin_base)

    def maintenance_quotidienne(self):
        """Routine de maintenance quotidienne"""
        print("🔄 === MAINTENANCE QUOTIDIENNE ===")
        print(f"Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

        # 1. Vérification d'intégrité
        self.maintenance.verifier_integrite_logs()

        # 2. Nettoyage des logs anciens (90 jours par défaut)
        self.maintenance.nettoyer_logs_anciens(90)

        # 3. Optimisation légère
        self.maintenance.optimiser_base_audit()

        print("✅ Maintenance quotidienne terminée\n")

    def maintenance_hebdomadaire(self):
        """Routine de maintenance hebdomadaire"""
        print("🔄 === MAINTENANCE HEBDOMADAIRE ===")
        print(f"Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

        # 1. Archivage des logs très anciens (1 an)
        self.maintenance.archiver_logs_anciens(365, "archive_logs")

        # 2. Optimisation complète
        self.maintenance.optimiser_base_audit()

        # 3. Génération des rapports de performance
        rapport = RapportAudit(self.maintenance.chemin_base)
        rapport.generer_rapport_performance(7)

        print("✅ Maintenance hebdomadaire terminée\n")

    def maintenance_urgence(self):
        """Maintenance d'urgence en cas de problème"""
        print("🚨 === MAINTENANCE D'URGENCE ===")
        print(f"Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

        # 1. Vérification d'intégrité immédiate
        self.maintenance.verifier_integrite_logs()

        # 2. Sauvegarde d'urgence — ⚠️ utiliser l'API .backup() de SQLite
        #    et NON shutil.copy/cp (qui peut corrompre une base en cours
        #    d'utilisation si une transaction est en vol).
        from contextlib import closing
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        backup_path = f"{self.maintenance.chemin_base}.emergency_backup_{timestamp}"
        try:
            with closing(sqlite3.connect(self.maintenance.chemin_base)) as src, \
                 closing(sqlite3.connect(backup_path)) as dst:
                src.backup(dst)
            print(f"   💾 Sauvegarde d'urgence: {backup_path}")
        except Exception as e:
            print(f"   ❌ Échec sauvegarde : {e}")

        # 3. Nettoyage agressif si nécessaire
        self.maintenance.nettoyer_logs_anciens(30)  # Plus agressif

        print("✅ Maintenance d'urgence terminée\n")

# Utilisation de la maintenance automatisée
scheduler = SchedulerMaintenance('ma_base.db')

print("=== DÉMONSTRATION MAINTENANCE ===")  
scheduler.maintenance_quotidienne()  
```

## Configuration avancée de l'audit

### Système d'audit configurable par règles

```python
import yaml  
from typing import Dict, List  

class ConfigurationAudit:
    def __init__(self, chemin_config: str = 'audit_config.yaml'):
        self.chemin_config = chemin_config
        self.config = self._charger_configuration()
        self.audit_manager = None

    def _charger_configuration(self) -> Dict:
        """Charge la configuration depuis un fichier YAML"""
        config_defaut = {
            'audit': {
                'actif': True,
                'niveau_detail': 'COMPLET',  # MINIMAL, STANDARD, COMPLET
                'retention_jours': 90,
                'archivage_jours': 365
            },
            'tables_auditees': {
                'utilisateurs': {
                    'actif': True,
                    'actions': ['INSERT', 'UPDATE', 'DELETE'],
                    'colonnes_sensibles': ['mot_de_passe_hash', 'email'],
                    'niveau_detail': 'COMPLET'
                },
                'commandes': {
                    'actif': True,
                    'actions': ['INSERT', 'UPDATE', 'DELETE'],
                    'colonnes_sensibles': ['montant', 'statut'],
                    'niveau_detail': 'STANDARD'
                },
                'logs_systeme': {
                    'actif': False,
                    'actions': ['DELETE'],
                    'niveau_detail': 'MINIMAL'
                }
            },
            'alertes': {
                'erreurs_consecutives': 5,
                'operations_massives': 100,
                'connexions_echouees': 10,
                'webhook_url': None,
                'email_admin': None
            },
            'performance': {
                'seuil_requete_lente_ms': 1000,
                'log_requetes_lentes': True,
                'optimisation_auto': True
            },
            'securite': {
                'masquer_donnees_sensibles': True,
                'chiffrer_logs': False,
                'ip_autorisees': [],
                'heures_maintenance': ['02:00', '03:00']
            }
        }

        def _fusion_profonde(defaut, surcouche):
            """Fusion récursive : la surcouche complète le défaut sans
            l'écraser pour les sous-dictionnaires."""
            if not isinstance(surcouche, dict):
                return surcouche if surcouche is not None else defaut
            resultat = dict(defaut)
            for cle, valeur in surcouche.items():
                if cle in resultat and isinstance(resultat[cle], dict):
                    resultat[cle] = _fusion_profonde(resultat[cle], valeur)
                else:
                    resultat[cle] = valeur
            return resultat

        try:
            with open(self.chemin_config, 'r', encoding='utf-8') as f:
                config_fichier = yaml.safe_load(f) or {}
                # ⚠️ Une fusion superficielle ({**a, **b}) écraserait
                #    entièrement des sous-dictionnaires comme `tables_auditees`.
                #    Si l'utilisateur ne définit qu'une seule table dans son
                #    YAML, toutes les autres tables par défaut disparaîtraient.
                return _fusion_profonde(config_defaut, config_fichier)
        except FileNotFoundError:
            print(f"⚠️ Fichier de config non trouvé, création de {self.chemin_config}")
            self._sauvegarder_configuration(config_defaut)
            return config_defaut

    def _sauvegarder_configuration(self, config: Dict):
        """Sauvegarde la configuration dans le fichier YAML"""
        with open(self.chemin_config, 'w', encoding='utf-8') as f:
            yaml.dump(config, f, default_flow_style=False, allow_unicode=True)

    def est_table_auditee(self, nom_table: str, action: str) -> bool:
        """Vérifie si une table/action doit être auditée"""
        if not self.config['audit']['actif']:
            return False

        table_config = self.config['tables_auditees'].get(nom_table, {})

        if not table_config.get('actif', False):
            return False

        actions_auditees = table_config.get('actions', [])
        return action.upper() in [a.upper() for a in actions_auditees]

    def obtenir_niveau_detail(self, nom_table: str) -> str:
        """Obtient le niveau de détail pour une table"""
        table_config = self.config['tables_auditees'].get(nom_table, {})
        return table_config.get('niveau_detail', self.config['audit']['niveau_detail'])

    def obtenir_colonnes_sensibles(self, nom_table: str) -> List[str]:
        """Obtient la liste des colonnes sensibles d'une table"""
        table_config = self.config['tables_auditees'].get(nom_table, {})
        return table_config.get('colonnes_sensibles', [])

    def masquer_donnees_sensibles(self, nom_table: str, donnees: Dict) -> Dict:
        """Masque les données sensibles selon la configuration"""
        if not self.config['securite']['masquer_donnees_sensibles']:
            return donnees

        donnees_masquees = donnees.copy()
        colonnes_sensibles = self.obtenir_colonnes_sensibles(nom_table)

        for colonne in colonnes_sensibles:
            if colonne in donnees_masquees:
                valeur = donnees_masquees[colonne]
                if isinstance(valeur, str) and len(valeur) > 4:
                    # Masquer en gardant les 2 premiers et 2 derniers caractères
                    donnees_masquees[colonne] = valeur[:2] + '*' * (len(valeur) - 4) + valeur[-2:]
                else:
                    donnees_masquees[colonne] = '***'

        return donnees_masquees

class AuditManagerAvance(AuditManager):
    """Version avancée du gestionnaire d'audit avec configuration"""

    def __init__(self, chemin_base: str, chemin_config: str = 'audit_config.yaml'):
        super().__init__(chemin_base)
        self.config_audit = ConfigurationAudit(chemin_config)

    def enregistrer_operation_conditionnelle(self, action: str, table_cible: str,
                                           enregistrement_id: int = None,
                                           anciennes_valeurs: Dict = None,
                                           nouvelles_valeurs: Dict = None,
                                           duree_ms: int = None):
        """Enregistre une opération seulement si configurée pour être auditée"""

        # Vérifier si l'audit est nécessaire
        if not self.config_audit.est_table_auditee(table_cible, action):
            return

        # Obtenir le niveau de détail
        niveau_detail = self.config_audit.obtenir_niveau_detail(table_cible)

        # Adapter les données selon le niveau de détail
        if niveau_detail == 'MINIMAL':
            anciennes_valeurs = None
            nouvelles_valeurs = None
        elif niveau_detail == 'STANDARD':
            # Masquer les données sensibles
            if anciennes_valeurs:
                anciennes_valeurs = self.config_audit.masquer_donnees_sensibles(
                    table_cible, anciennes_valeurs
                )
            if nouvelles_valeurs:
                nouvelles_valeurs = self.config_audit.masquer_donnees_sensibles(
                    table_cible, nouvelles_valeurs
                )
        # COMPLET : garder toutes les données

        # Enregistrer l'opération
        super().enregistrer_operation(
            action=action,
            table_cible=table_cible,
            enregistrement_id=enregistrement_id,
            anciennes_valeurs=anciennes_valeurs,
            nouvelles_valeurs=nouvelles_valeurs,
            duree_ms=duree_ms
        )

    def verifier_alertes(self):
        """Vérifie si des alertes doivent être déclenchées"""
        config_alertes = self.config_audit.config['alertes']

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        alertes_declenchees = []

        # 1. Erreurs consécutives
        seuil_erreurs = config_alertes['erreurs_consecutives']
        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE timestamp > datetime('now', '-15 minutes')
              AND statut != 'SUCCESS'
        """)
        erreurs_recentes = cursor.fetchone()[0]

        if erreurs_recentes >= seuil_erreurs:
            alertes_declenchees.append({
                'type': 'ERREURS_CONSECUTIVES',
                'message': f'{erreurs_recentes} erreurs en 15 minutes',
                'gravite': 'CRITICAL'
            })

        # 2. Opérations massives
        seuil_operations = config_alertes['operations_massives']
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(*) as operations
            FROM audit_log
            WHERE timestamp > datetime('now', '-1 hour')
              AND action IN ('INSERT', 'UPDATE', 'DELETE')
            GROUP BY nom_utilisateur
            HAVING operations > ?
        """, (seuil_operations,))

        operations_massives = cursor.fetchall()
        for utilisateur, count in operations_massives:
            alertes_declenchees.append({
                'type': 'OPERATIONS_MASSIVES',
                'message': f'{utilisateur}: {count} opérations en 1 heure',
                'gravite': 'WARNING'
            })

        # 3. Connexions échouées
        seuil_connexions = config_alertes['connexions_echouees']
        cursor.execute("""
            SELECT adresse_ip, COUNT(*) as tentatives
            FROM audit_securite
            WHERE timestamp > datetime('now', '-1 hour')
              AND type_evenement = 'FAILED_LOGIN'
            GROUP BY adresse_ip
            HAVING tentatives > ?
        """, (seuil_connexions,))

        connexions_echouees = cursor.fetchall()
        for ip, tentatives in connexions_echouees:
            alertes_declenchees.append({
                'type': 'CONNEXIONS_ECHOUEES',
                'message': f'IP {ip}: {tentatives} tentatives échouées en 1 heure',
                'gravite': 'CRITICAL'
            })

        conn.close()

        # Envoyer les alertes
        for alerte in alertes_declenchees:
            self._envoyer_alerte(alerte)

        return alertes_declenchees

    def _envoyer_alerte(self, alerte: Dict):
        """Envoie une alerte selon la configuration"""
        config_alertes = self.config_audit.config['alertes']

        message_alerte = f"🚨 ALERTE SQLite: {alerte['type']}\n{alerte['message']}"

        # Webhook
        webhook_url = config_alertes.get('webhook_url')
        if webhook_url:
            try:
                import requests
                payload = {
                    "text": message_alerte,
                    "username": "SQLite Audit",
                    "channel": "#alerts"
                }
                requests.post(webhook_url, json=payload, timeout=10)
            except Exception as e:
                print(f"Erreur envoi webhook: {e}")

        # Email (simulation)
        email_admin = config_alertes.get('email_admin')
        if email_admin:
            print(f"📧 Email envoyé à {email_admin}: {message_alerte}")

        # Log local
        print(f"🔔 {message_alerte}")

# Utilisation de l'audit avancé
audit_avance = AuditManagerAvance('ma_base.db')

# Définir le contexte
audit_avance.definir_contexte(1, 'alice', 'sess_456', '192.168.1.100')

# Test d'enregistrement conditionnel
print("=== TEST AUDIT CONDITIONNEL ===")

# Cette opération sera auditée selon la config
audit_avance.enregistrer_operation_conditionnelle(
    action='UPDATE',
    table_cible='utilisateurs',
    enregistrement_id=1,
    anciennes_valeurs={'email': 'ancien@email.com', 'mot_de_passe_hash': 'hash123'},
    nouvelles_valeurs={'email': 'nouveau@email.com', 'mot_de_passe_hash': 'newhash456'}
)

# Vérifier les alertes
alertes = audit_avance.verifier_alertes()  
if alertes:  
    print(f"⚠️ {len(alertes)} alertes déclenchées")
else:
    print("✅ Aucune alerte")
```

## Outils d'analyse forensique

### Reconstruction d'événements

```python
from datetime import datetime, timedelta  
import networkx as nx  
import matplotlib.pyplot as plt  

class AnalyseForensique:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base

    def reconstituer_chronologie(self, utilisateur: str = None,
                                table: str = None,
                                enregistrement_id: int = None,
                                heures: int = 24):
        """Reconstitue la chronologie des événements"""
        print(f"🔍 ANALYSE FORENSIQUE - {heures}h")
        if utilisateur:
            print(f"👤 Utilisateur: {utilisateur}")
        if table:
            print(f"📋 Table: {table}")
        if enregistrement_id:
            print(f"🆔 Enregistrement: {enregistrement_id}")
        print("=" * 60)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Construire la requête avec filtres
        where_clauses = ["timestamp > datetime('now', '-{} hours')".format(heures)]
        params = []

        if utilisateur:
            where_clauses.append("nom_utilisateur = ?")
            params.append(utilisateur)

        if table:
            where_clauses.append("table_cible = ?")
            params.append(table)

        if enregistrement_id:
            where_clauses.append("enregistrement_id = ?")
            params.append(enregistrement_id)

        where_clause = " AND ".join(where_clauses)

        cursor.execute(f"""
            SELECT timestamp, nom_utilisateur, action, table_cible,
                   enregistrement_id, anciennes_valeurs, nouvelles_valeurs,
                   adresse_ip, statut
            FROM audit_log
            WHERE {where_clause}
            ORDER BY timestamp ASC
        """, params)

        evenements = cursor.fetchall()

        print(f"📊 {len(evenements)} événements trouvés\n")

        for i, event in enumerate(evenements, 1):
            timestamp, user, action, table_cible, record_id, old_vals, new_vals, ip, statut = event

            status_emoji = "✅" if statut == "SUCCESS" else "❌"

            print(f"{i:3d}. {timestamp} {status_emoji}")
            print(f"     👤 {user} @ {ip}")
            print(f"     🔄 {action} sur {table_cible}")
            if record_id:
                print(f"     🆔 Enregistrement: {record_id}")

            # Afficher les changements (les JSON peuvent être tronqués/malformés
            # si la base a été altérée manuellement → on cible json.JSONDecodeError
            # plutôt qu'un except: nu qui masquerait aussi KeyboardInterrupt).
            if action in ['UPDATE', 'DELETE'] and old_vals:
                try:
                    old_data = json.loads(old_vals)
                    print(f"     📤 Avant: {old_data}")
                except json.JSONDecodeError:
                    print(f"     📤 Avant (JSON invalide): {old_vals[:80]}")

            if action in ['INSERT', 'UPDATE'] and new_vals:
                try:
                    new_data = json.loads(new_vals)
                    print(f"     📥 Après: {new_data}")
                except json.JSONDecodeError:
                    print(f"     📥 Après (JSON invalide): {new_vals[:80]}")

            print()

        conn.close()
        return evenements

    def analyser_pattern_utilisateur(self, utilisateur: str, jours: int = 7):
        """Analyse les patterns d'activité d'un utilisateur"""
        print(f"📈 ANALYSE PATTERN UTILISATEUR: {utilisateur}")
        print("=" * 60)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(days=jours)

        # 1. Activité par heure
        cursor.execute("""
            SELECT strftime('%H', timestamp) as heure, COUNT(*) as operations
            FROM audit_log
            WHERE nom_utilisateur = ? AND timestamp > ?
            GROUP BY strftime('%H', timestamp)
            ORDER BY heure
        """, (utilisateur, date_debut))

        print("⏰ Activité par heure:")
        activite_heures = cursor.fetchall()
        for heure, ops in activite_heures:
            barre = "█" * min(ops // 2, 20)
            print(f"   {heure}h: {ops:3d} ops {barre}")
        print()

        # 2. Actions les plus fréquentes
        cursor.execute("""
            SELECT action, table_cible, COUNT(*) as count
            FROM audit_log
            WHERE nom_utilisateur = ? AND timestamp > ?
            GROUP BY action, table_cible
            ORDER BY count DESC
            LIMIT 10
        """, (utilisateur, date_debut))

        print("🔄 Actions les plus fréquentes:")
        for action, table, count in cursor.fetchall():
            print(f"   • {action} sur {table}: {count} fois")
        print()

        # 3. Adresses IP utilisées
        cursor.execute("""
            SELECT adresse_ip, COUNT(*) as connexions,
                   MIN(timestamp) as premiere, MAX(timestamp) as derniere
            FROM audit_log
            WHERE nom_utilisateur = ? AND timestamp > ? AND adresse_ip IS NOT NULL
            GROUP BY adresse_ip
            ORDER BY connexions DESC
        """, (utilisateur, date_debut))

        print("🌐 Adresses IP utilisées:")
        ips = cursor.fetchall()
        for ip, connexions, premiere, derniere in ips:
            print(f"   • {ip}: {connexions} connexions ({premiere} → {derniere})")

        if len(ips) > 2:
            print("   ⚠️ Utilisateur connecté depuis plusieurs IP")

        # 4. Détection d'anomalies
        print("\n🔍 Détection d'anomalies:")
        anomalies = []

        # Activité nocturne
        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE nom_utilisateur = ? AND timestamp > ?
              AND (CAST(strftime('%H', timestamp) AS INTEGER) < 6
                   OR CAST(strftime('%H', timestamp) AS INTEGER) > 22)
        """, (utilisateur, date_debut))

        activite_nocturne = cursor.fetchone()[0]
        if activite_nocturne > 0:
            anomalies.append(f"Activité nocturne: {activite_nocturne} opérations")

        # Volume anormalement élevé
        cursor.execute("""
            SELECT COUNT(*) as operations_total FROM audit_log
            WHERE nom_utilisateur = ? AND timestamp > ?
        """, (utilisateur, date_debut))

        total_ops = cursor.fetchone()[0]
        moyenne_quotidienne = total_ops / jours

        if moyenne_quotidienne > 500:  # Seuil arbitraire
            anomalies.append(f"Volume élevé: {moyenne_quotidienne:.0f} ops/jour")

        if anomalies:
            for anomalie in anomalies:
                print(f"   ⚠️ {anomalie}")
        else:
            print("   ✅ Aucune anomalie détectée")

        conn.close()

    def generer_graphe_activite(self, heures: int = 24, fichier_sortie: str = "audit_graph.png"):
        """Génère un graphe des interactions entre utilisateurs et tables"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(hours=heures)

        cursor.execute("""
            SELECT nom_utilisateur, table_cible, COUNT(*) as poids
            FROM audit_log
            WHERE timestamp > ? AND nom_utilisateur IS NOT NULL AND table_cible IS NOT NULL
            GROUP BY nom_utilisateur, table_cible
            ORDER BY poids DESC
        """, (date_debut,))

        interactions = cursor.fetchall()

        # Créer le graphe
        G = nx.Graph()

        for utilisateur, table, poids in interactions:
            G.add_edge(f"👤{utilisateur}", f"📋{table}", weight=poids)

        # Dessiner le graphe
        plt.figure(figsize=(12, 8))
        pos = nx.spring_layout(G, k=1, iterations=50)

        # Dessiner les nœuds
        user_nodes = [n for n in G.nodes() if n.startswith('👤')]
        table_nodes = [n for n in G.nodes() if n.startswith('📋')]

        nx.draw_networkx_nodes(G, pos, nodelist=user_nodes,
                              node_color='lightblue', node_size=1000, alpha=0.7)
        nx.draw_networkx_nodes(G, pos, nodelist=table_nodes,
                              node_color='lightgreen', node_size=800, alpha=0.7)

        # Dessiner les arêtes avec épaisseur proportionnelle au poids
        edges = G.edges()
        weights = [G[u][v]['weight'] for u, v in edges]
        max_weight = max(weights) if weights else 1

        nx.draw_networkx_edges(G, pos, width=[w/max_weight * 5 for w in weights], alpha=0.6)

        # Ajouter les labels
        nx.draw_networkx_labels(G, pos, font_size=8)

        # Ajouter les poids sur les arêtes
        edge_labels = {(u, v): str(d['weight']) for u, v, d in G.edges(data=True)}
        nx.draw_networkx_edge_labels(G, pos, edge_labels, font_size=6)

        plt.title(f"Graphe d'activité SQLite - {heures}h", fontsize=14)
        plt.axis('off')
        plt.tight_layout()
        plt.savefig(fichier_sortie, dpi=300, bbox_inches='tight')
        plt.close()

        print(f"📊 Graphe d'activité sauvegardé: {fichier_sortie}")

        conn.close()

# Utilisation de l'analyse forensique
forensique = AnalyseForensique('ma_base.db')

print("=== ANALYSE FORENSIQUE ===")

# Reconstituer la chronologie pour un utilisateur spécifique
evenements = forensique.reconstituer_chronologie(
    utilisateur='alice',
    heures=24
)

# Analyser les patterns d'un utilisateur
forensique.analyser_pattern_utilisateur('alice', 7)

# Générer le graphe d'activité
# forensique.generer_graphe_activite(24, "activite_audit.png")
```

## Conformité et reporting réglementaire

### Générateur de rapports de conformité

```python
class ConformiteReglementaire:
    def __init__(self, chemin_base: str):
        self.chemin_base = chemin_base

    def rapport_rgpd(self, periode_jours: int = 30):
        """Génère un rapport de conformité RGPD"""
        print("🇪🇺 RAPPORT DE CONFORMITÉ RGPD")
        print("=" * 60)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(days=periode_jours)

        # 1. Accès aux données personnelles
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(*) as acces_donnees_perso
            FROM audit_log
            WHERE timestamp > ?
              AND table_cible IN ('utilisateurs', 'clients', 'donnees_personnelles')
              AND action = 'SELECT'
            GROUP BY nom_utilisateur
            ORDER BY acces_donnees_perso DESC
        """, (date_debut,))

        print("📋 Accès aux données personnelles:")
        acces_donnees = cursor.fetchall()
        if acces_donnees:
            for utilisateur, acces in acces_donnees:
                print(f"   • {utilisateur}: {acces} consultations")
        else:
            print("   ℹ️ Aucun accès aux données personnelles enregistré")
        print()

        # 2. Modifications de données personnelles
        cursor.execute("""
            SELECT nom_utilisateur, action, COUNT(*) as modifications
            FROM audit_log
            WHERE timestamp > ?
              AND table_cible IN ('utilisateurs', 'clients', 'donnees_personnelles')
              AND action IN ('INSERT', 'UPDATE', 'DELETE')
            GROUP BY nom_utilisateur, action
            ORDER BY modifications DESC
        """, (date_debut,))

        print("✏️ Modifications de données personnelles:")
        for utilisateur, action, count in cursor.fetchall():
            print(f"   • {utilisateur} - {action}: {count} fois")
        print()

        # 3. Suppressions (droit à l'effacement)
        cursor.execute("""
            SELECT COUNT(*) as suppressions_totales,
                   COUNT(DISTINCT enregistrement_id) as enregistrements_supprimes
            FROM audit_log
            WHERE timestamp > ?
              AND action = 'DELETE'
              AND table_cible IN ('utilisateurs', 'clients', 'donnees_personnelles')
        """, (date_debut,))

        suppressions = cursor.fetchone()
        print(f"🗑️ Suppressions (droit à l'effacement):")
        print(f"   • Total suppressions: {suppressions[0]}")
        print(f"   • Enregistrements uniques supprimés: {suppressions[1]}")
        print()

        # 4. Violations potentielles
        cursor.execute("""
            SELECT COUNT(*) as violations_potentielles
            FROM audit_log
            WHERE timestamp > ?
              AND statut != 'SUCCESS'
              AND table_cible IN ('utilisateurs', 'clients', 'donnees_personnelles')
        """, (date_debut,))

        violations = cursor.fetchone()[0]
        if violations > 0:
            print(f"⚠️ Violations potentielles détectées: {violations}")
            print("   → Enquête recommandée")
        else:
            print("✅ Aucune violation détectée")

        conn.close()

    def rapport_sox(self, periode_jours: int = 90):
        """Génère un rapport de conformité SOX (Sarbanes-Oxley)"""
        print("\n🏢 RAPPORT DE CONFORMITÉ SOX")
        print("=" * 60)

        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_debut = datetime.now() - timedelta(days=periode_jours)

        # 1. Contrôles d'accès aux données financières
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(*) as acces_financiers
            FROM audit_log
            WHERE timestamp > ?
              AND table_cible IN ('transactions', 'factures', 'comptabilite', 'salaires')
              AND action IN ('SELECT', 'INSERT', 'UPDATE', 'DELETE')
            GROUP BY nom_utilisateur
            ORDER BY acces_financiers DESC
        """, (date_debut,))

        print("💰 Accès aux données financières:")
        for utilisateur, acces in cursor.fetchall():
            print(f"   • {utilisateur}: {acces} opérations")
        print()

        # 2. Séparation des tâches (Segregation of Duties)
        cursor.execute("""
            SELECT nom_utilisateur,
                   SUM(CASE WHEN action = 'INSERT' THEN 1 ELSE 0 END) as creations,
                   SUM(CASE WHEN action = 'UPDATE' THEN 1 ELSE 0 END) as modifications,
                   SUM(CASE WHEN action = 'DELETE' THEN 1 ELSE 0 END) as suppressions
            FROM audit_log
            WHERE timestamp > ?
              AND table_cible IN ('transactions', 'factures', 'comptabilite')
            GROUP BY nom_utilisateur
        """, (date_debut,))

        print("🔐 Analyse de séparation des tâches:")
        violations_separation = []
        for utilisateur, creations, modifications, suppressions in cursor.fetchall():
            total_actions = creations + modifications + suppressions
            if total_actions > 0:
                # Vérifier si un utilisateur a trop de privilèges
                privileges = sum([1 for x in [creations, modifications, suppressions] if x > 0])
                if privileges >= 3 and total_actions > 10:  # Seuils configurables
                    violations_separation.append(utilisateur)

                print(f"   • {utilisateur}: C:{creations} M:{modifications} S:{suppressions}")

        if violations_separation:
            print("   ⚠️ Violations potentielles de séparation des tâches:")
            for utilisateur in violations_separation:
                print(f"      → {utilisateur}")
        else:
            print("   ✅ Séparation des tâches respectée")
        print()

        # 3. Traçabilité des modifications critiques
        cursor.execute("""
            SELECT COUNT(*) as modifications_critiques
            FROM audit_log
            WHERE timestamp > ?
              AND table_cible IN ('transactions', 'factures', 'comptabilite')
              AND action IN ('UPDATE', 'DELETE')
              AND anciennes_valeurs IS NOT NULL
        """, (date_debut,))

        modifications_critiques = cursor.fetchone()[0]
        print(f"📊 Traçabilité des modifications critiques:")
        print(f"   • {modifications_critiques} modifications tracées avec détails")

        if modifications_critiques > 0:
            print("   ✅ Traçabilité conforme")
        else:
            print("   ⚠️ Aucune modification critique tracée")

        # 4. Contrôles temporels (modifications en dehors des heures)
        cursor.execute("""
            SELECT nom_utilisateur, COUNT(*) as modifications_hors_heures
            FROM audit_log
            WHERE timestamp > ?
              AND table_cible IN ('transactions', 'factures', 'comptabilite')
              AND action IN ('INSERT', 'UPDATE', 'DELETE')
              AND (CAST(strftime('%H', timestamp) AS INTEGER) < 8
                   OR CAST(strftime('%H', timestamp) AS INTEGER) > 18
                   OR strftime('%w', timestamp) IN ('0', '6'))
            GROUP BY nom_utilisateur
            ORDER BY modifications_hors_heures DESC
        """, (date_debut,))

        print("\n⏰ Contrôles temporels (modifications hors heures ouvrables):")
        modifications_hors_heures = cursor.fetchall()
        if modifications_hors_heures:
            for utilisateur, count in modifications_hors_heures:
                print(f"   ⚠️ {utilisateur}: {count} modifications hors heures")
        else:
            print("   ✅ Toutes les modifications en heures ouvrables")

        conn.close()

    def generer_rapport_complet(self, format_export: str = 'json'):
        """Génère un rapport de conformité complet exportable"""
        print(f"\n📄 GÉNÉRATION RAPPORT COMPLET (format: {format_export})")
        print("=" * 60)

        rapport = {
            'metadata': {
                'date_generation': datetime.now().isoformat(),
                'periode_analyse_jours': 30,
                'base_de_donnees': self.chemin_base
            },
            'conformite': {
                'rgpd': self._analyser_conformite_rgpd(),
                'sox': self._analyser_conformite_sox(),
                'iso27001': self._analyser_conformite_iso27001()
            },
            'recommandations': self._generer_recommandations()
        }

        if format_export == 'json':
            fichier = f"rapport_conformite_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
            with open(fichier, 'w', encoding='utf-8') as f:
                json.dump(rapport, f, indent=2, ensure_ascii=False, default=str)
            print(f"✅ Rapport JSON généré: {fichier}")

        elif format_export == 'html':
            fichier = f"rapport_conformite_{datetime.now().strftime('%Y%m%d_%H%M%S')}.html"
            self._generer_rapport_html(rapport, fichier)
            print(f"✅ Rapport HTML généré: {fichier}")

        return rapport

    def _analyser_conformite_rgpd(self) -> Dict:
        """Analyse la conformité RGPD"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        # Vérifications RGPD
        date_limite = datetime.now() - timedelta(days=30)

        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE timestamp > ? AND table_cible IN ('utilisateurs', 'clients')
        """, (date_limite,))
        acces_donnees_perso = cursor.fetchone()[0]

        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE timestamp > ? AND action = 'DELETE'
              AND table_cible IN ('utilisateurs', 'clients')
        """, (date_limite,))
        suppressions_rgpd = cursor.fetchone()[0]

        conn.close()

        score_conformite = 0
        issues = []

        # Scoring RGPD (chaque critère vaut 25 points, total visé : 100)
        # ⚠️ Ce barème pédagogique simpliste doit être adapté au contexte
        #    réel : un score parfait ici ne prouve PAS la conformité légale.
        if acces_donnees_perso > 0:
            score_conformite += 25
        else:
            issues.append("Aucun accès aux données personnelles tracé")

        # Vérifier que la table audit_log existe et contient des données
        #   (signe minimal que le mécanisme d'audit est actif)
        if suppressions_rgpd > 0:
            score_conformite += 25
        else:
            issues.append("Aucune suppression RGPD tracée sur la période")

        return {
            'score': score_conformite,
            'max_score': 100,
            'conformite_percentage': (score_conformite / 100) * 100,
            'acces_donnees_personnelles': acces_donnees_perso,
            'suppressions_rgpd': suppressions_rgpd,
            'issues': issues,
            'statut': 'CONFORME' if score_conformite >= 75 else 'NON_CONFORME'
        }

    def _analyser_conformite_sox(self) -> Dict:
        """Analyse la conformité SOX"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_limite = datetime.now() - timedelta(days=90)

        # Vérifications SOX
        cursor.execute("""
            SELECT COUNT(DISTINCT nom_utilisateur) FROM audit_log
            WHERE timestamp > ? AND table_cible IN ('transactions', 'factures')
        """, (date_limite,))
        utilisateurs_financiers = cursor.fetchone()[0]

        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE timestamp > ? AND table_cible IN ('transactions', 'factures')
              AND action IN ('UPDATE', 'DELETE') AND anciennes_valeurs IS NOT NULL
        """, (date_limite,))
        modifications_tracees = cursor.fetchone()[0]

        conn.close()

        score_sox = 0
        issues = []

        if utilisateurs_financiers > 0:
            score_sox += 40

        if modifications_tracees > 0:
            score_sox += 40
        else:
            issues.append("Modifications financières non tracées en détail")

        return {
            'score': score_sox,
            'max_score': 100,
            'conformite_percentage': (score_sox / 100) * 100,
            'utilisateurs_acces_financier': utilisateurs_financiers,
            'modifications_tracees': modifications_tracees,
            'issues': issues,
            'statut': 'CONFORME' if score_sox >= 75 else 'NON_CONFORME'
        }

    def _analyser_conformite_iso27001(self) -> Dict:
        """Analyse la conformité ISO 27001"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        date_limite = datetime.now() - timedelta(days=30)

        # Vérifications ISO 27001
        cursor.execute("""
            SELECT COUNT(*) FROM audit_securite
            WHERE timestamp > ?
        """, (date_limite,))
        evenements_securite = cursor.fetchone()[0]

        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE timestamp > ? AND statut != 'SUCCESS'
        """, (date_limite,))
        incidents_securite = cursor.fetchone()[0]

        conn.close()

        score_iso = 0
        issues = []

        if evenements_securite > 0:
            score_iso += 50
        else:
            issues.append("Aucun événement de sécurité tracé")

        if incidents_securite == 0:
            score_iso += 30
        elif incidents_securite < 5:
            score_iso += 15
        else:
            issues.append(f"Nombreux incidents de sécurité: {incidents_securite}")

        return {
            'score': score_iso,
            'max_score': 100,
            'conformite_percentage': (score_iso / 100) * 100,
            'evenements_securite': evenements_securite,
            'incidents_securite': incidents_securite,
            'issues': issues,
            'statut': 'CONFORME' if score_iso >= 75 else 'NON_CONFORME'
        }

    def _generer_recommandations(self) -> List[str]:
        """Génère des recommandations d'amélioration"""
        conn = sqlite3.connect(self.chemin_base)
        cursor = conn.cursor()

        recommandations = []

        # Vérifier si l'audit est complet
        cursor.execute("SELECT COUNT(*) FROM audit_log WHERE timestamp > datetime('now', '-7 days')")
        logs_recents = cursor.fetchone()[0]

        if logs_recents < 100:
            recommandations.append("Augmenter la granularité de l'audit - peu d'activité enregistrée")

        # Vérifier la rétention
        cursor.execute("SELECT MIN(timestamp) FROM audit_log")
        plus_ancien = cursor.fetchone()[0]
        if plus_ancien:
            plus_ancien_date = datetime.fromisoformat(plus_ancien)
            if (datetime.now() - plus_ancien_date).days < 90:
                recommandations.append("Augmenter la période de rétention des logs (minimum 90 jours)")

        # Vérifier les erreurs
        cursor.execute("""
            SELECT COUNT(*) FROM audit_log
            WHERE timestamp > datetime('now', '-7 days') AND statut != 'SUCCESS'
        """)
        erreurs_recentes = cursor.fetchone()[0]

        if erreurs_recentes > 10:
            recommandations.append("Investiguer et corriger les erreurs fréquentes dans les logs")

        conn.close()

        if not recommandations:
            recommandations.append("Système d'audit fonctionnel - continuer la surveillance")

        return recommandations

    def _generer_rapport_html(self, rapport: Dict, fichier: str):
        """Génère un rapport HTML lisible"""
        html_content = f"""
<!DOCTYPE html>
<html>
<head>
    <title>Rapport de Conformité SQLite</title>
    <meta charset="utf-8">
    <style>
        body {{ font-family: Arial, sans-serif; margin: 20px; }}
        .header {{ background: #f0f0f0; padding: 20px; border-radius: 5px; }}
        .section {{ margin: 20px 0; padding: 15px; border-left: 4px solid #007acc; }}
        .conforme {{ color: green; font-weight: bold; }}
        .non-conforme {{ color: red; font-weight: bold; }}
        .score {{ font-size: 24px; font-weight: bold; }}
        .issue {{ background: #ffe6e6; padding: 5px; margin: 5px 0; border-radius: 3px; }}
        .recommendation {{ background: #e6f3ff; padding: 5px; margin: 5px 0; border-radius: 3px; }}
    </style>
</head>
<body>
    <div class="header">
        <h1>📊 Rapport de Conformité SQLite</h1>
        <p><strong>Généré le:</strong> {rapport['metadata']['date_generation']}</p>
        <p><strong>Base de données:</strong> {rapport['metadata']['base_de_donnees']}</p>
    </div>

    <div class="section">
        <h2>🇪🇺 Conformité RGPD</h2>
        <div class="score {'conforme' if rapport['conformite']['rgpd']['statut'] == 'CONFORME' else 'non-conforme'}">
            Score: {rapport['conformite']['rgpd']['score']}/100 ({rapport['conformite']['rgpd']['conformite_percentage']:.1f}%)
        </div>
        <p><strong>Statut:</strong> <span class="{'conforme' if rapport['conformite']['rgpd']['statut'] == 'CONFORME' else 'non-conforme'}">{rapport['conformite']['rgpd']['statut']}</span></p>
        <p>Accès aux données personnelles: {rapport['conformite']['rgpd']['acces_donnees_personnelles']}</p>
        <p>Suppressions RGPD: {rapport['conformite']['rgpd']['suppressions_rgpd']}</p>
        {"".join([f'<div class="issue">⚠️ {issue}</div>' for issue in rapport['conformite']['rgpd']['issues']])}
    </div>

    <div class="section">
        <h2>🏢 Conformité SOX</h2>
        <div class="score {'conforme' if rapport['conformite']['sox']['statut'] == 'CONFORME' else 'non-conforme'}">
            Score: {rapport['conformite']['sox']['score']}/100 ({rapport['conformite']['sox']['conformite_percentage']:.1f}%)
        </div>
        <p><strong>Statut:</strong> <span class="{'conforme' if rapport['conformite']['sox']['statut'] == 'CONFORME' else 'non-conforme'}">{rapport['conformite']['sox']['statut']}</span></p>
        <p>Utilisateurs avec accès financier: {rapport['conformite']['sox']['utilisateurs_acces_financier']}</p>
        <p>Modifications tracées: {rapport['conformite']['sox']['modifications_tracees']}</p>
        {"".join([f'<div class="issue">⚠️ {issue}</div>' for issue in rapport['conformite']['sox']['issues']])}
    </div>

    <div class="section">
        <h2>🔒 Conformité ISO 27001</h2>
        <div class="score {'conforme' if rapport['conformite']['iso27001']['statut'] == 'CONFORME' else 'non-conforme'}">
            Score: {rapport['conformite']['iso27001']['score']}/100 ({rapport['conformite']['iso27001']['conformite_percentage']:.1f}%)
        </div>
        <p><strong>Statut:</strong> <span class="{'conforme' if rapport['conformite']['iso27001']['statut'] == 'CONFORME' else 'non-conforme'}">{rapport['conformite']['iso27001']['statut']}</span></p>
        <p>Événements de sécurité: {rapport['conformite']['iso27001']['evenements_securite']}</p>
        <p>Incidents de sécurité: {rapport['conformite']['iso27001']['incidents_securite']}</p>
        {"".join([f'<div class="issue">⚠️ {issue}</div>' for issue in rapport['conformite']['iso27001']['issues']])}
    </div>

    <div class="section">
        <h2>💡 Recommandations</h2>
        {"".join([f'<div class="recommendation">• {rec}</div>' for rec in rapport['recommandations']])}
    </div>
</body>
</html>
        """

        with open(fichier, 'w', encoding='utf-8') as f:
            f.write(html_content)

# Utilisation des rapports de conformité
conformite = ConformiteReglementaire('ma_base.db')

print("=== RAPPORTS DE CONFORMITÉ RÉGLEMENTAIRE ===")  
conformite.rapport_rgpd(30)  
conformite.rapport_sox(90)  

# Générer le rapport complet
rapport_complet = conformite.generer_rapport_complet('json')  
print(f"📊 Score global moyen: {sum([r['score'] for r in rapport_complet['conformite'].values()]) / 3:.1f}/100")  
```

## Conclusion et bonnes pratiques

### Récapitulatif des concepts essentiels

L'audit et le logging dans SQLite nécessitent une approche méthodique et bien planifiée. Voici les points clés à retenir :

#### **🔑 Principes fondamentaux**

1. **Auditez ce qui compte** : Ne pas tout enregistrer, mais cibler les opérations critiques
2. **Pensez performance** : L'audit ne doit pas ralentir l'application
3. **Sécurisez les logs** : Les logs eux-mêmes contiennent des données sensibles
4. **Planifiez la rétention** : Équilibrer besoins réglementaires et espace disque
5. **Automatisez l'analyse** : Les logs ne servent que s'ils sont analysés

#### **📋 Checklist de déploiement**

```
☐ Tables d'audit créées avec index appropriés
☐ Triggers automatiques configurés pour les tables critiques
☐ Système de rotation et archivage des logs configuré
☐ Alertes automatiques en place
☐ Rapports de conformité automatisés
☐ Procédures de maintenance définies
☐ Formation équipe sur l'interprétation des logs
☐ Tests de récupération des données d'audit
☐ Documentation des procédures d'audit
☐ Validation avec l'équipe légale/conformité
```

#### **⚡ Optimisations recommandées**

```sql
-- Optimisations de performance pour les tables d'audit
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  
PRAGMA cache_size = 10000;  

-- Index spécialisés pour l'audit
CREATE INDEX idx_audit_user_date ON audit_log(nom_utilisateur, timestamp);  
CREATE INDEX idx_audit_table_action ON audit_log(table_cible, action);  
CREATE INDEX idx_audit_record ON audit_log(table_cible, enregistrement_id);  
```

#### **🚨 Alertes critiques à configurer**

- **Échecs d'authentification répétés** (>5 en 15 min)
- **Modifications massives** (>100 enregistrements en 1h)
- **Accès hors heures** aux données sensibles
- **Erreurs de base de données** fréquentes
- **Tentatives d'injection SQL** détectées
- **Modifications de schéma** non autorisées

#### **📊 Métriques KPI à surveiller**

```python
# Métriques essentielles à monitorer
kpi_audit = {
    'volume_logs_quotidien': 'Nombre de logs générés par jour',
    'taux_erreur': 'Pourcentage d\'opérations en erreur',
    'temps_reponse_moyen': 'Latence moyenne des opérations',
    'utilisateurs_actifs': 'Nombre d\'utilisateurs uniques par période',
    'pic_activite': 'Heures de plus forte activité',
    'conformite_score': 'Score de conformité réglementaire',
    'taille_logs': 'Espace disque utilisé par les logs',
    'retention_effective': 'Ancienneté des logs les plus anciens'
}
```

### **🎯 Recommandations par type d'application**

#### **Applications web**
- Logger les sessions utilisateur
- Tracer les modifications via API REST
- Implémenter l'audit par adresse IP
- Configurer les alertes de sécurité web

#### **Applications mobiles**
- Audit des synchronisations
- Traçabilité des modifications offline
- Gestion des conflits de données
- Logging des erreurs de connectivité

#### **Applications d'entreprise**
- Conformité réglementaire stricte
- Séparation des environnements (dev/test/prod)
- Intégration avec SIEM d'entreprise
- Rapports automatisés pour audits

#### **Applications IoT/embarquées**
- Audit minimal pour préserver les ressources
- Compression et rotation fréquente des logs
- Transmission sécurisée vers serveurs centraux
- Gestion de la perte de connectivité

### **🔄 Cycle de vie de l'audit**

1. **Planification** : Définir les exigences d'audit
2. **Implémentation** : Développer les mécanismes d'audit
3. **Déploiement** : Activer l'audit en production
4. **Surveillance** : Monitor continu des logs
5. **Analyse** : Génération de rapports réguliers
6. **Optimisation** : Amélioration continue du système
7. **Archivage** : Gestion du cycle de vie des données

### **📚 Ressources pour aller plus loin**

- **Standards de conformité** : GDPR, SOX, ISO 27001, HIPAA
- **Outils de SIEM** : Splunk, ELK Stack, Graylog
- **Frameworks d'audit** : COBIT, NIST Cybersecurity Framework
- **Bibliothèques Python** : `logging` (stdlib), `structlog` (logging structuré), `python-json-logger` (logs JSON)

L'audit et le logging constituent la fondation de la sécurité et de la conformité de votre application SQLite. Un système bien conçu vous permettra non seulement de répondre aux exigences réglementaires, mais aussi d'améliorer continuellement la sécurité et les performances de votre application.

---

*Cette section complète le chapitre 8.3 sur l'audit et le logging des opérations SQLite. La section suivante (8.4) abordera les sauvegardes automatisées et les stratégies de récupération.*

⏭️
