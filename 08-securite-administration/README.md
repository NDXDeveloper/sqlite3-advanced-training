🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8. Sécurité et administration

## Introduction à la sécurité et à l'administration SQLite

La sécurité et l'administration des bases de données SQLite présentent des défis uniques comparés aux systèmes de gestion de bases de données traditionnels. Contrairement aux SGBD client-serveur comme MySQL ou PostgreSQL, SQLite fonctionne comme une bibliothèque intégrée directement dans l'application, ce qui modifie fondamentalement l'approche de la sécurité.

### Spécificités de la sécurité SQLite

**Architecture serverless et implications sécuritaires**

L'architecture serverless de SQLite signifie qu'il n'y a pas de processus serveur dédié, pas d'authentification réseau, et pas de gestion centralisée des utilisateurs. La sécurité repose principalement sur :

- **Sécurité au niveau du système de fichiers** : Les permissions d'accès au fichier de base de données sont gérées par le système d'exploitation
- **Sécurité applicative** : L'application elle-même devient responsable du contrôle d'accès et de l'authentification
- **Sécurité physique** : Le fichier de base étant stocké localement, sa protection physique devient critique

**Modèle de menaces spécifique**

Les menaces de sécurité avec SQLite diffèrent des SGBD traditionnels :

1. **Accès direct au fichier** : Un attaquant ayant accès au système de fichiers peut potentiellement lire directement le fichier de base
2. **Injection SQL** : Reste une menace majeure, particulièrement dans les applications web
3. **Corruption de données** : Accès concurrent non contrôlé ou arrêt brutal de l'application
4. **Fuite de données** : Données sensibles non chiffrées dans le fichier de base
5. **Élévation de privilèges** : Via l'exploitation de vulnérabilités dans l'application hôte

### Principes fondamentaux de l'administration SQLite

**Gestion du cycle de vie des données**

L'administration SQLite implique une approche différente de la gestion traditionnelle :

- **Déploiement** : Distribution et installation des fichiers de base avec l'application
- **Migration** : Gestion des versions de schéma lors des mises à jour d'application
- **Maintenance** : Optimisation périodique (VACUUM, REINDEX) intégrée à l'application
- **Surveillance** : Monitoring des performances et de l'intégrité via l'application

**Stratégies de sauvegarde et récupération**

La nature de fichier unique de SQLite simplifie certains aspects de la sauvegarde mais en complique d'autres :

- **Sauvegarde à chaud** : Utilisation de l'API de sauvegarde SQLite pour éviter les corruptions
- **Réplication** : Mise en place de mécanismes de synchronisation pour les environnements distribués
- **Récupération** : Stratégies de restauration et de réparation de fichiers corrompus

### Défis administratifs courants

**Performance et optimisation**

- **Croissance des fichiers** : Gestion de l'espace disque et fragmentation
- **Verrouillage de base** : Gestion des accès concurrents et des timeouts
- **Configuration PRAGMA** : Optimisation des paramètres selon le contexte d'usage

**Intégrité et cohérence**

- **Vérification d'intégrité** : Contrôles réguliers avec PRAGMA integrity_check
- **Gestion des transactions** : Assurer la cohérence lors d'opérations complexes
- **Récupération après incident** : Procédures de restauration en cas de corruption

### Bonnes pratiques générales

**Architecture sécurisée**

1. **Principe du moindre privilège** : L'application ne doit accéder qu'aux données nécessaires
2. **Défense en profondeur** : Combinaison de plusieurs couches de sécurité
3. **Validation stricte** : Contrôle rigoureux des entrées utilisateur
4. **Audit et traçabilité** : Logging des opérations sensibles

**Opérations administratives**

1. **Maintenance préventive** : Planification des opérations de maintenance
2. **Monitoring continu** : Surveillance des métriques de performance
3. **Gestion des incidents** : Procédures de réponse aux problèmes
4. **Documentation** : Maintien d'une documentation à jour des procédures

### Outils et ressources

**Outils de ligne de commande**

- **SQLite CLI** (`sqlite3`) : interface principale pour l'administration manuelle
- **sqlite3_analyzer** : analyse détaillée de la structure et utilisation de l'espace (téléchargeable depuis [sqlite.org](https://sqlite.org/download.html))
- **Scripts personnalisés** : automatisation des tâches récurrentes (cron, systemd timers)

**Outils de développement et clients graphiques**

- **DB Browser for SQLite** : multi-plateforme, gratuit ([sqlitebrowser.org](https://sqlitebrowser.org/))
- **SQLiteStudio** : alternative légère et open-source
- **DBeaver Community** : client universel multi-BDD
- **TablePlus** / **JetBrains DataGrip** : clients professionnels (payants)

**Outils spécifiques sécurité et administration**

- **SQLCipher** : fork de SQLite avec chiffrement AES-256 transparent du fichier (voir 8.1)
- **Litestream** : réplication continue vers S3/Azure/GCS pour la sauvegarde et disponibilité (voir 8.4)
- **LiteFS** : système de fichiers distribué pour clustering SQLite avec failover
- **`pragma_*` virtual tables** : interroger les PRAGMA depuis SQL pour le monitoring

> ⚠️ **À ne JAMAIS faire** : utiliser SQLite sur un **partage réseau** (NFS, SMB, SSHFS). Le verrouillage de fichier n'est pas garanti correctement sur ces filesystems → corruption probable. Pour la haute disponibilité ou l'accès multi-machine, utilisez Litestream, LiteFS, ou migrez vers un SGBD client-serveur.

### Prérequis Python pour ce chapitre

Plusieurs bibliothèques Python sont utilisées dans les exemples des sections 8.1 à 8.5. Pour exécuter l'ensemble des exemples, installez d'un coup :

```bash
# Crypto + bindings SQLite chiffré (8.1)
pip install sqlcipher3-binary cryptography argon2-cffi bcrypt

# Audit / monitoring / dashboard (8.3, 8.5)
pip install psutil schedule flask requests pyyaml structlog python-json-logger

# Optionnel — analyse forensique avancée (8.3)
pip install networkx matplotlib
```

> 💡 **Sélection minimale** : si vous voulez juste suivre les sections 8.2 (permissions), 8.3 (audit) et 8.4 (sauvegardes) sans chiffrement ni dashboard, seul `pip install cryptography` suffit (le module `sqlite3` est dans la stdlib Python).

**Niveau de difficulté** : ce chapitre suppose connue la **section 7 (transactions et concurrence)** car beaucoup d'exemples utilisent WAL, `Connection.backup()` et `isolation_level=None`. Si ces concepts sont flous, relire 7.1 et 7.4 avant d'attaquer 8.4.

### 🔧 Boilerplate à mettre en haut de tout module du chapitre

Les sections 8.2 → 8.5 supposent **trois lignes de configuration** en début d'application. Sans elles, des bugs silencieux apparaissent (rapports temporels faux, `DeprecationWarning` Python 3.12+, performances WAL non activées). À copier-coller **une seule fois** dans votre point d'entrée :

```python
import sqlite3  
from datetime import datetime, timezone  

# ── 1. Adapter datetime → "YYYY-MM-DD HH:MM:SS" en UTC ─────────────────────
#    À appeler UNE SEULE FOIS au démarrage du processus (registre global).
#    Règle 2 problèmes d'un coup :
#      (a) DeprecationWarning Python 3.12+ sur datetime → cursor.execute()
#      (b) Bug silencieux UTC/local : CURRENT_TIMESTAMP est en UTC alors que
#          datetime.now() est en local → filtres temporels faux selon le
#          fuseau (rapports « dernière heure » qui ratent des entrées).
def _adapter_dt_utc(d):
    if d.tzinfo is None:
        d = d.astimezone(timezone.utc)
    else:
        d = d.astimezone(timezone.utc)
    return d.strftime("%Y-%m-%d %H:%M:%S")
sqlite3.register_adapter(datetime, _adapter_dt_utc)

# ── 2. Init UNE FOIS par base : PRAGMA persistants (journal_mode) ──────────
#    `journal_mode = WAL` est écrit dans le header de la base → persiste.
#    Bénéfices : lecture concurrente, prérequis Litestream (8.4), `.backup()`
#    sur base active fonctionne mieux.
def init_base(chemin):
    """À appeler une fois après création d'une nouvelle base SQLite."""
    conn = sqlite3.connect(chemin)
    conn.execute("PRAGMA journal_mode = WAL")
    conn.close()

# ── 3. Wrapper de connexion : PRAGMA per-connection (foreign_keys) ─────────
#    ⚠️ PIÈGE : `PRAGMA foreign_keys = ON` est PER-CONNECTION (pas persistant).
#    Si vous ne le rejouez pas à chaque ouverture, les FK ne sont PAS vérifiées.
#    → utilisez ce wrapper partout au lieu de sqlite3.connect() direct.
def ouvrir_connexion(chemin, **kwargs):
    """Ouverture de connexion avec les PRAGMA per-connection rejoués."""
    conn = sqlite3.connect(chemin, **kwargs)
    conn.execute("PRAGMA foreign_keys = ON")
    return conn
```

> 💡 **Pourquoi pas dans chaque section ?** Pour éviter la duplication. Chaque fichier 8.x rappelle ces lignes dans un encadré `⚠️` mais le code complet et **validé par test** est ici. Si vous copiez-collez un exemple isolé d'une section, pensez à y ajouter ce boilerplate.  
>  
> **Distinction critique entre `init_base()` et `ouvrir_connexion()`** :  
> - **Persistant (dans `init_base`)** : `journal_mode`, `page_size`, `auto_vacuum`, `application_id`, `user_version`  
> - **Per-connection (dans `ouvrir_connexion`)** : `foreign_keys`, `recursive_triggers`, `cache_size`, `temp_store`, `synchronous` (sauf en WAL où il devient persistant), `mmap_size`  
>  
> Confondre les deux est un piège classique : on active `foreign_keys` à la création de la base, puis on s'étonne que les contraintes ne soient pas vérifiées en production.

### Vue d'ensemble des sections suivantes

Les sections suivantes de ce chapitre aborderont en détail :

- **Chiffrement des bases de données** : Protection des données au repos et techniques de chiffrement
- **Gestion des permissions et contrôle d'accès** : Stratégies d'authentification et d'autorisation
- **Audit et logging** : Traçabilité des opérations et détection d'anomalies
- **Sauvegardes automatisées** : Mise en place de stratégies de sauvegarde robustes
- **Monitoring et maintenance préventive** : Surveillance continue et optimisation proactive

Cette approche progressive permettra de maîtriser tous les aspects de la sécurité et de l'administration SQLite, depuis les concepts fondamentaux jusqu'aux implémentations les plus avancées.

---

*Dans la section suivante, nous explorerons les techniques de chiffrement des bases de données SQLite, un aspect crucial pour la protection des données sensibles.*

⏭️
