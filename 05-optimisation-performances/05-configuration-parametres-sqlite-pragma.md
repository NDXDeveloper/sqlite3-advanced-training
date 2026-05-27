🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.5 Configuration des paramètres SQLite (PRAGMA)

## Introduction aux PRAGMA

Les **PRAGMA** sont comme les réglages de votre voiture : ils permettent d'ajuster SQLite pour qu'il fonctionne au mieux selon vos besoins. Tout comme vous réglez vos rétroviseurs et votre siège avant de conduire, vous pouvez configurer SQLite pour optimiser ses performances.

### Qu'est-ce qu'un PRAGMA ?

Un PRAGMA est une commande spéciale SQLite qui :
- Configure le comportement de la base de données
- Contrôle les performances et la sécurité
- Ajuste l'utilisation de la mémoire et du disque
- Modifie les modes de fonctionnement

**Syntaxe de base :**
```sql
-- Lire une configuration
PRAGMA nom_parametre;

-- Modifier une configuration
PRAGMA nom_parametre = valeur;
```

## PRAGMA essentiels pour les performances

### 1. Cache Size - Taille du cache mémoire

Le cache, c'est comme la mémoire de travail de SQLite. Plus il est grand, moins SQLite a besoin de lire le disque.

```sql
-- Voir la taille actuelle du cache
PRAGMA cache_size;
-- Résultat typique : -2000 (≈ 2 Mio par défaut)

-- Augmenter le cache à ~64 Mio
PRAGMA cache_size = -64000;  -- ⚠️ valeur NÉGATIVE = Kio (kibioctets, 1024 octets)
                             --     -64000 = 64000 Kio ≈ 62.5 Mio (et non 64 Mo exacts)

-- Ou en nombre de pages (1 page = 4096 octets par défaut)
PRAGMA cache_size = 16000;   -- 16000 pages × 4096 = 62.5 Mio
```

> 💡 **Subtilité Kio vs Mo** : SQLite raisonne en **Kibibytes** (1024) et non en kilobytes (1000). Les valeurs « ~64 Mo » mentionnées partout dans ce chapitre sont en réalité **62.5 Mibibytes**. La différence est sans importance pratique, mais à connaître pour ne pas être surpris au calcul.

**Recommandations :**
- **Petite base (< 100MB) :** 8-16MB de cache
- **Base moyenne (100MB-1GB) :** 64-128MB de cache
- **Grande base (> 1GB) :** 256MB+ de cache

**Test pratique :**
```sql
-- Tester l'impact du cache
.timer on

-- Avec petit cache
PRAGMA cache_size = -8000;  -- 8MB  
SELECT COUNT(*) FROM grande_table WHERE condition_complexe;  
-- Temps : ex. 2.5 secondes

-- Avec grand cache
PRAGMA cache_size = -64000;  -- 64MB  
SELECT COUNT(*) FROM grande_table WHERE condition_complexe;  
-- Temps : ex. 1.2 secondes → 2x plus rapide !
```

### 2. Journal Mode - Mode de journalisation

Le journal mode détermine comment SQLite gère la durabilité et la concurrence.

```sql
-- Voir le mode actuel
PRAGMA journal_mode;

-- Modes disponibles
PRAGMA journal_mode = DELETE;    -- Par défaut, compatible  
PRAGMA journal_mode = WAL;       -- Recommandé pour performances  
PRAGMA journal_mode = MEMORY;    -- Rapide mais risqué  
PRAGMA journal_mode = OFF;       -- Très rapide mais dangereux  
```

**Comparaison des modes :**

| Mode | Performance | Concurrence | Sécurité | Usage |
|------|-------------|-------------|----------|-------|
| DELETE | Normale | Faible | Élevée | Par défaut, compatible partout |
| **WAL** | **Élevée** | **Élevée** | **Élevée** | **Recommandé** pour la plupart des cas |
| MEMORY | Très élevée | Faible | ❌ Crash = corruption | Bases jetables uniquement |
| OFF | Maximale | Faible | ❌ Pas de rollback, pas de durabilité | Imports massifs ponctuels, données reconstructibles |

> ⚠️ **Précisions sur la « sécurité »** :  
> - **MEMORY** : le journal est en RAM → un crash de processus (panic, kill -9, coupure de courant) laisse la base dans un état **incohérent et irréparable**. C'est différent d'une simple « perte de la dernière transaction ».  
> - **OFF** : SQLite n'écrit AUCUN journal → même un `ROLLBACK` explicite ne peut plus annuler une transaction. La base reste dans l'état où le crash l'a laissée — peut être complètement corrompue.  
> - **DELETE/WAL** : garantissent la **récupération automatique** au prochain `open()` (replay du journal).

**Mode WAL en détail :**
```sql
-- Activer le mode WAL (Write-Ahead Logging)
PRAGMA journal_mode = WAL;
-- ⚠️ Retourne 'wal' si succès, le mode précédent sinon (échec silencieux !)
--    Toujours vérifier : SELECT * FROM pragma_journal_mode();

-- Avantages du mode WAL :
-- ✅ Lectures et écritures simultanées (1 writer + N readers en parallèle)
-- ✅ Meilleures performances générales sur les écritures
-- ✅ Récupération après crash améliorée
-- ✅ Transactions plus rapides (1 seul fsync au lieu de 2)
```

> 💡 **WAL est PERSISTANT** : contrairement aux autres modes (`DELETE`, `MEMORY`, `OFF` qui sont propres à la connexion courante), une fois la base passée en WAL, **elle y reste** — y compris après fermeture/réouverture, et pour toutes les futures connexions. Pas besoin de réappliquer `PRAGMA journal_mode = WAL` à chaque ouverture.

> ⚠️ **Limites du mode WAL** :  
> - Ne fonctionne **pas** sur les systèmes de fichiers réseau (NFS, SMB) — utilise du shared memory mapping local  
> - 3 fichiers visibles au lieu d'1 : `mabase.db`, `mabase.db-wal`, `mabase.db-shm`  
> - Le fichier `-wal` peut grossir si beaucoup de lecteurs actifs empêchent le checkpoint ; lancer `PRAGMA wal_checkpoint(TRUNCATE);` pour forcer la consolidation

### 3. Synchronous - Niveau de synchronisation

Contrôle combien SQLite attend que les données soient réellement écrites sur le disque.

```sql
-- Voir le niveau actuel
PRAGMA synchronous;

-- Niveaux disponibles
PRAGMA synchronous = OFF;      -- 0: Aucune synchronisation (rapide, risqué)  
PRAGMA synchronous = NORMAL;   -- 1: Synchronisation normale (bon compromis)  
PRAGMA synchronous = FULL;     -- 2: Synchronisation complète (sûr, lent)  
PRAGMA synchronous = EXTRA;    -- 3: Synchronisation maximale (très sûr, très lent)  
```

> 💡 **Valeur par défaut selon le mode journal** :  
> - Mode rollback (DELETE, TRUNCATE, PERSIST) : **`FULL`** par défaut — sécurité maximale, plus lent  
> - Mode WAL : **`NORMAL`** par défaut — sûr car le WAL garantit déjà la cohérence, plus rapide  
>  
> Autrement dit : en passant en WAL avec `synchronous = NORMAL`, on obtient **à la fois** plus de sécurité **et** plus de vitesse qu'en rollback `FULL`. C'est pourquoi WAL + NORMAL est le couple recommandé pour la plupart des cas.

**Recommandations par contexte :**

**Pour le développement :**
```sql
-- Configuration rapide pour le développement
PRAGMA synchronous = OFF;  
PRAGMA journal_mode = MEMORY;  
-- ⚠️ DANGER : avec `journal_mode = MEMORY`, un crash de processus (kill -9, panic,
--    coupure de courant) corrompt SILENCIEUSEMENT la base — pas de récupération
--    possible. À réserver aux bases JETABLES (tests, caches reconstructibles).
--    Pour le dev sur des données importantes : WAL + NORMAL convient (rapide ET sûr).
```

**Pour la production :**
```sql
-- Configuration équilibrée pour la production
PRAGMA synchronous = NORMAL;  
PRAGMA journal_mode = WAL;  
-- Bon compromis performance/sécurité
```

**Pour les données critiques :**
```sql
-- Configuration sécurisée pour données importantes
PRAGMA synchronous = FULL;  
PRAGMA journal_mode = WAL;  
-- Maximum de sécurité
```

### 4. Temp Store - Stockage des données temporaires

Détermine où SQLite stocke les tables temporaires et les index.

```sql
-- Voir la configuration actuelle
PRAGMA temp_store;

-- Options disponibles
PRAGMA temp_store = DEFAULT;  -- 0: Système par défaut  
PRAGMA temp_store = FILE;     -- 1: Fichiers temporaires sur disque  
PRAGMA temp_store = MEMORY;   -- 2: En mémoire (plus rapide)  
```

**Pour les performances :**
```sql
-- Stockage temporaire en mémoire pour plus de rapidité
PRAGMA temp_store = MEMORY;
```

> ⚠️ **`PRAGMA temp_store_directory` est déprécié** par les développeurs SQLite. Il continue de fonctionner pour la compatibilité, mais les nouvelles applications doivent l'éviter. Pour contrôler l'emplacement des fichiers temporaires, utiliser la variable d'environnement **`SQLITE_TMPDIR`** côté système, ou la définir au démarrage de l'application.

## Configuration optimisée par cas d'usage

### Configuration pour application web

```sql
-- Configuration optimisée pour une application web
PRAGMA journal_mode = WAL;           -- Concurrence améliorée  
PRAGMA synchronous = NORMAL;         -- Bon compromis  
PRAGMA cache_size = -64000;          -- 64MB de cache  
PRAGMA temp_store = MEMORY;          -- Temporaires en mémoire  
PRAGMA mmap_size = 268435456;        -- 256MB memory mapping  

-- Vérifier les changements
SELECT
    'journal_mode: ' || (SELECT * FROM pragma_journal_mode()),
    'synchronous: ' || (SELECT * FROM pragma_synchronous()),
    'cache_size: ' || (SELECT * FROM pragma_cache_size());
```

### Configuration pour analytics/reporting

```sql
-- Configuration pour gros volumes de lecture
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  
PRAGMA cache_size = -256000;         -- 256MB cache (plus gros)  
PRAGMA temp_store = MEMORY;  
PRAGMA mmap_size = 1073741824;       -- 1GB memory mapping  
PRAGMA threads = 4;                  -- ⚠️ Threads auxiliaires pour ORDER BY/INDEX —  
                                     --    requiert un binaire compilé avec SQLITE_MAX_WORKER_THREADS > 0  
                                     --    (cas standard sur la plupart des distributions Linux).
```

### Configuration pour applications mobiles

```sql
-- Configuration optimisée pour mobile (économie batterie/mémoire)
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  
PRAGMA cache_size = -16000;          -- 16MB cache (plus petit)  
PRAGMA temp_store = MEMORY;  
PRAGMA auto_vacuum = INCREMENTAL;    -- Nettoyage automatique  
```

### Configuration pour tests unitaires

```sql
-- Configuration pour tests rapides
PRAGMA journal_mode = MEMORY;        -- Très rapide  
PRAGMA synchronous = OFF;            -- Pas de synchronisation  
PRAGMA cache_size = -32000;          -- 32MB cache  
PRAGMA temp_store = MEMORY;  
-- ⚠️ Uniquement pour les tests !
```

## PRAGMA de monitoring et diagnostic

### 1. Informations sur la base de données

```sql
-- Taille et structure
PRAGMA page_count;          -- Nombre total de pages  
PRAGMA page_size;           -- Taille d'une page (généralement 4096)  
PRAGMA freelist_count;      -- Pages libres (fragmentation)  

-- Calculer la taille totale
SELECT
    (SELECT * FROM pragma_page_count()) *
    (SELECT * FROM pragma_page_size()) / 1024.0 / 1024.0
    AS 'Taille DB (MB)';
```

### 2. Statistiques d'utilisation

```sql
-- Statistiques détaillées de la base
PRAGMA database_list;       -- Lister toutes les bases attachées  
PRAGMA table_list;          -- Lister toutes les tables  
PRAGMA compile_options;     -- Options de compilation SQLite  

-- Informations sur une table spécifique
PRAGMA table_info(nom_table);    -- Structure de la table  
PRAGMA index_list(nom_table);    -- Index de la table  
PRAGMA foreign_key_list(nom_table);  -- Clés étrangères  
```

### 3. Intégrité et maintenance

```sql
-- Vérifier l'intégrité de la base
PRAGMA integrity_check;
-- ou plus rapide :
PRAGMA quick_check;

-- Statistiques pour l'optimiseur
PRAGMA optimize;            -- Optimisation automatique  
ANALYZE;                    -- Mise à jour des statistiques  
```

### `PRAGMA optimize` en détail (recommandation officielle SQLite)

`PRAGMA optimize` est **l'outil de maintenance n°1** recommandé par les développeurs SQLite. Il analyse l'état actuel de la base et lance **uniquement les opérations utiles** (ANALYZE ciblés sur les index modifiés, etc.) — beaucoup plus malin et rapide qu'un `ANALYZE` global aveugle.

**Quand l'appeler :**
- À la **fermeture de chaque connexion long-lived** (serveur web, démon).
- À l'**ouverture** d'une connexion **après une longue période d'inactivité** (par sécurité).
- **Pas besoin** après chaque transaction : trop fréquent, gaspillage.

```sql
-- Appel simple (production) — pas de sortie si rien à faire
PRAGMA optimize;
-- Équivalent au masque par défaut 0x10000 + 0x0002 dans la plupart des versions :
-- lance les ANALYZE détectés comme nécessaires sur les tables modifiées.

-- Mode verbeux (debug) : préfixe 0x10000 pour activer le reporting détaillé
PRAGMA optimize=0x10002;
-- Sortie typique : "ANALYZE 'main'.'idx_categorie' ;" ou rien

-- Forcer une analyse plus complète (très rare en pratique)
PRAGMA optimize=0xfffe;
-- Active tous les bits sauf 0x0001 — réservé aux cas où vous suspectez
-- des statistiques très obsolètes.
```

> ℹ️ **Masque de bits** : la valeur passée à `PRAGMA optimize` est un masque où chaque bit active une optimisation spécifique. Les bits exacts ont **évolué entre versions** de SQLite — pour les détails à jour, consulter la documentation officielle : [sqlite.org/pragma.html#pragma_optimize](https://www.sqlite.org/pragma.html#pragma_optimize).  
>  
> En pratique, **trois appels suffisent** pour 99 % des cas :  
> - `PRAGMA optimize;` — appel normal (production)  
> - `PRAGMA optimize=0x10002;` — diagnostic verbeux (debug, ne change rien d'irréversible)  
> - `PRAGMA optimize=0xfffe;` — forcer l'analyse maximale (très rare)

### Intégrer `PRAGMA optimize` dans son application

**Python — pattern recommandé :**
```python
import sqlite3  
import atexit  

class ManagedSQLite:
    def __init__(self, db_path):
        self.conn = sqlite3.connect(db_path)
        # Configuration recommandée
        self.conn.execute("PRAGMA journal_mode = WAL")
        self.conn.execute("PRAGMA synchronous = NORMAL")
        self.conn.execute("PRAGMA cache_size = -64000")
        # Enregistrer la cleanup pour TOUTE fin de processus
        atexit.register(self.close)

    def close(self):
        if self.conn:
            # ⚠️ PRAGMA optimize AVANT de fermer = stats à jour pour la prochaine
            #    connexion. Quasi-instantané si rien à faire, indispensable sur
            #    base modifiée.
            self.conn.execute("PRAGMA optimize")
            self.conn.close()
            self.conn = None

# Usage
db = ManagedSQLite('ma_base.db')
# ... requêtes ...
# À la fin du processus, atexit appelle db.close() qui exécute PRAGMA optimize
```

**Pour un serveur web (long-lived avec pool de connexions)** :
```python
# Stratégie 1 : à chaque retour de connexion au pool
def return_connection(conn):
    conn.execute("PRAGMA optimize")
    pool.put(conn)

# Stratégie 2 (mieux) : périodiquement, dans un thread de maintenance
import threading, time  
def maintenance_loop(pool, interval=3600):  # toutes les heures  
    while True:
        time.sleep(interval)
        with pool.acquire() as conn:
            conn.execute("PRAGMA optimize")
```

> 💡 **`PRAGMA optimize` vs `ANALYZE` brut** :  
> - `ANALYZE` (sans arg) recalcule **TOUTES** les stats — coûteux sur grosse base.  
> - `PRAGMA optimize` ne fait `ANALYZE` que sur les index qui en ont **réellement besoin** (modifications significatives depuis le dernier ANALYZE). En production, **toujours préférer** `PRAGMA optimize`.

> ⚠️ **Piège** : `PRAGMA optimize` se base sur un compteur interne de modifications. Si vous **importez en masse** avec `journal_mode = OFF` ou via `.import`, ce compteur peut être incorrect. Dans ce cas, lancer un `ANALYZE` complet une fois après l'import, puis revenir au cycle `PRAGMA optimize`.

## PRAGMA avancés pour l'optimisation

### 1. Memory Mapping

Le memory mapping permet à SQLite d'accéder aux données comme si elles étaient en mémoire.

```sql
-- Voir la configuration actuelle
PRAGMA mmap_size;

-- Activer memory mapping (recommandé pour gros fichiers)
PRAGMA mmap_size = 268435456;  -- 256MB
-- ou
PRAGMA mmap_size = 1073741824; -- 1GB pour très gros fichiers

-- Désactiver memory mapping
PRAGMA mmap_size = 0;
```

**Avantages du memory mapping :**
- Accès plus rapide aux données fréquemment utilisées
- Réduction de la charge CPU
- Meilleure utilisation de la mémoire système

### 2. Auto Vacuum

Contrôle le nettoyage automatique de l'espace libre.

```sql
-- Voir le mode actuel
PRAGMA auto_vacuum;

-- Modes disponibles
PRAGMA auto_vacuum = NONE;         -- 0: Pas de nettoyage auto  
PRAGMA auto_vacuum = FULL;         -- 1: Nettoyage complet (lent)  
PRAGMA auto_vacuum = INCREMENTAL;  -- 2: Nettoyage incrémental (recommandé)  
```

> ⚠️ **Piège majeur** : `auto_vacuum` ne peut être changé librement après création de tables !  
>  
> | Transition | Comportement |  
> |---|---|  
> | `NONE` → `FULL` ou `INCREMENTAL` sur base avec tables | **Nécessite un `VACUUM`** complet pour prendre effet |  
> | `FULL` ↔ `INCREMENTAL` | Changement immédiat, à tout moment |  
> | `FULL`/`INCREMENTAL` → `NONE` | **Toujours `VACUUM` requis**, même base vide |  
>  
> **Bonne pratique** : définir `auto_vacuum` **avant** toute création de table sur une base neuve. Sinon, prévoir un `VACUUM;` (qui réécrit toute la base — peut être long sur de gros fichiers).

**Usage du mode incrémental :**
```sql
-- Activer le mode incrémental dès la création de la base (idéal)
PRAGMA auto_vacuum = INCREMENTAL;
-- (puis créer les tables ici)

-- Sur une base existante, il faut compléter par :
VACUUM;

-- Plus tard : nettoyer manuellement quand nécessaire
PRAGMA incremental_vacuum(1000);  -- Nettoie 1000 pages
```

### 3. Clés étrangères

```sql
-- Voir si les clés étrangères sont activées (rappel : OFF par défaut !)
PRAGMA foreign_keys;

-- Activer les contraintes de clés étrangères
PRAGMA foreign_keys = ON;   -- Léger surcoût en écriture, mais garde-fou indispensable

-- ⚠️ DANGER : désactiver les FK pour un import "rapide" laisse passer
--    silencieusement des données qui violent les contraintes !
PRAGMA foreign_keys = OFF;
-- [Import potentiellement non-cohérent]
PRAGMA foreign_keys = ON;
-- SQLite NE VÉRIFIE PAS les données existantes à la réactivation.
-- Les lignes orphelines restent en place sans erreur.
-- ✅ TOUJOURS lancer la vérification après un import :
PRAGMA foreign_key_check;     -- liste les lignes en violation (table, rowid, parent, FK#)
-- Vide = OK. Sinon : corriger AVANT de remettre les FK en service applicatif.
```

> 💡 Pour un import massif, alternative plus sûre : garder `foreign_keys = ON` mais utiliser une grande **transaction unique** (`BEGIN ... COMMIT`) avec `PRAGMA defer_foreign_keys = ON` qui reporte la vérification au `COMMIT`. Si une seule violation, toute la transaction est annulée — pas de données orphelines.

## Profils de configuration prêts à l'emploi

### Profil "Performance Web"

```sql
-- Configuration optimisée pour application web
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  
PRAGMA cache_size = -64000;  
PRAGMA temp_store = MEMORY;  
PRAGMA mmap_size = 268435456;  
PRAGMA auto_vacuum = INCREMENTAL;  
PRAGMA foreign_keys = ON;  

-- Vérification
.echo on
PRAGMA journal_mode;  
PRAGMA synchronous;  
PRAGMA cache_size;  
```

### Profil "Analytics"

```sql
-- Configuration pour lecture intensive
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  
PRAGMA cache_size = -256000;    -- Cache plus important  
PRAGMA temp_store = MEMORY;  
PRAGMA mmap_size = 1073741824;  -- Memory mapping agressif  
PRAGMA auto_vacuum = NONE;      -- Pas de nettoyage automatique  
```

### Profil "Développement"

```sql
-- Configuration pour développement (sûre ET rapide)
-- ⚠️ Évitez `journal_mode = MEMORY` + `synchronous = OFF` même en dev :
--    un crash IDE / un reboot inattendu peut corrompre votre base de test
--    et faire perdre des heures de mise en place de données.
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  
PRAGMA cache_size = -32000;  
PRAGMA temp_store = MEMORY;  
PRAGMA foreign_keys = ON;       -- Garder les contraintes en dev  
```

```sql
-- Profil "Tests unitaires jetables" (base recréée à chaque run, OK à perdre)
PRAGMA journal_mode = MEMORY;  
PRAGMA synchronous = OFF;  
PRAGMA cache_size = -32000;  
PRAGMA temp_store = MEMORY;  
PRAGMA foreign_keys = ON;  
```

### Profil "Production Sécurisée"

```sql
-- Configuration pour données critiques
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = FULL;      -- Sécurité maximale  
PRAGMA cache_size = -128000;  
PRAGMA temp_store = MEMORY;  
PRAGMA auto_vacuum = INCREMENTAL;  
PRAGMA foreign_keys = ON;  
PRAGMA integrity_check;         -- Vérification à l'ouverture  
```

## Appliquer les configurations automatiquement

### 1. Dans une application Python

```python
import sqlite3

def configure_sqlite_performance(conn):
    """Configure SQLite pour de meilleures performances.

    ⚠️ Appeler IMMÉDIATEMENT après connect() — avant toute requête,
       car `PRAGMA journal_mode = WAL` est ignoré silencieusement si
       une transaction est en cours.
    """
    # Configuration de base
    conn.execute("PRAGMA journal_mode = WAL")
    conn.execute("PRAGMA synchronous = NORMAL")
    conn.execute("PRAGMA cache_size = -64000")     # ~64 Mio
    conn.execute("PRAGMA temp_store = MEMORY")
    conn.execute("PRAGMA mmap_size = 268435456")   # 256 Mio

    # ℹ️ Pas de conn.commit() : les PRAGMA ne sont pas des écritures
    #    qui nécessitent un commit. Le mode WAL, lui, est PERSISTANT —
    #    pas besoin de le réappliquer aux connexions suivantes.

    # Vérification systématique recommandée
    mode_actuel = conn.execute("PRAGMA journal_mode").fetchone()[0]
    if mode_actuel != 'wal':
        raise RuntimeError(f"Activation WAL échouée — mode actuel : {mode_actuel}")

# Usage
conn = sqlite3.connect('ma_base.db')  
configure_sqlite_performance(conn)  
```

> ⚠️ **Piège silencieux** : `PRAGMA journal_mode = WAL` est **refusé** si une transaction est active. Le pragma retourne alors le mode précédent **sans erreur** — facile à manquer. C'est pourquoi le code ci-dessus vérifie le mode après application.

### 2. Fichier de configuration

```sql
-- config_performance.sql
-- À exécuter à chaque connexion

PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  
PRAGMA cache_size = -64000;  
PRAGMA temp_store = MEMORY;  
PRAGMA mmap_size = 268435456;  

-- Vérification
SELECT 'Configuration appliquée' as status;
```

```bash
# Appliquer la configuration
sqlite3 ma_base.db < config_performance.sql
```

### 3. Configuration par défaut dans l'application

```python
# Configuration automatique à chaque connexion
class ConfiguredSQLite:
    def __init__(self, db_path, config_profile='web'):
        self.db_path = db_path
        self.config_profile = config_profile

    def connect(self):
        conn = sqlite3.connect(self.db_path)
        self._apply_config(conn)
        return conn

    def _apply_config(self, conn):
        configs = {
            'web': [
                "PRAGMA journal_mode = WAL",
                "PRAGMA synchronous = NORMAL",
                "PRAGMA cache_size = -64000",
                "PRAGMA temp_store = MEMORY"
            ],
            'analytics': [
                "PRAGMA journal_mode = WAL",
                "PRAGMA synchronous = NORMAL",
                "PRAGMA cache_size = -256000",
                "PRAGMA mmap_size = 1073741824"
            ]
        }

        for pragma in configs.get(self.config_profile, []):
            conn.execute(pragma)
```

## Mesurer l'impact des configurations

### Benchmark avant/après

```sql
-- Script de test performance
.timer on
.echo on

-- Configuration par défaut
PRAGMA journal_mode = DELETE;  
PRAGMA synchronous = FULL;  
PRAGMA cache_size = -2000;  

-- Test de performance
SELECT 'Test avec config par défaut' as test;  
SELECT COUNT(*) FROM large_table WHERE complex_condition;  

-- Configuration optimisée
PRAGMA journal_mode = WAL;  
PRAGMA synchronous = NORMAL;  
PRAGMA cache_size = -64000;  
PRAGMA temp_store = MEMORY;  

-- Même test
SELECT 'Test avec config optimisée' as test;  
SELECT COUNT(*) FROM large_table WHERE complex_condition;  
```

### Monitoring des performances

```sql
-- Voir l'utilisation actuelle
PRAGMA cache_size;  
PRAGMA page_count;  
PRAGMA freelist_count;  

-- Calculer l'efficacité du cache
-- ⚠️ `cache_size` peut être NÉGATIF (= taille en Kio) ou POSITIF (= nb de pages).
--    On normalise en pages pour pouvoir comparer à `page_count` (toujours en pages).
WITH config AS (
    SELECT
        (SELECT * FROM pragma_cache_size()) AS raw_cache,
        (SELECT * FROM pragma_page_size())  AS page_size_bytes,
        (SELECT * FROM pragma_page_count()) AS total_pages
)
SELECT
    'Cache : ' || CASE
        WHEN raw_cache < 0 THEN ABS(raw_cache) * 1024 / page_size_bytes
        ELSE raw_cache
    END || ' pages'                                            AS cache_info,
    'DB    : ' || total_pages || ' pages'                       AS db_info,
    'Ratio : ' || ROUND(
        CASE
            WHEN raw_cache < 0 THEN ABS(raw_cache) * 1024.0 / page_size_bytes
            ELSE raw_cache * 1.0
        END / NULLIF(total_pages, 0) * 100, 1
    ) || '%'                                                    AS cache_ratio
FROM config;
-- Ratio > 100 % = TOUTE la base tient dans le cache → idéal pour les lectures.
```

## Troubleshooting et problèmes courants

### Problème 1 : Base de données lente après configuration

```sql
-- Diagnostic
PRAGMA integrity_check;  -- Vérifier l'intégrité  
PRAGMA optimize;         -- Optimiser automatiquement  
ANALYZE;                 -- Recalculer les statistiques  

-- Si problème persiste, revenir aux paramètres par défaut
PRAGMA journal_mode = DELETE;  
PRAGMA synchronous = FULL;  
PRAGMA cache_size = -2000;  
```

### Problème 2 : Consommation mémoire excessive

```sql
-- Réduire l'utilisation mémoire
PRAGMA cache_size = -16000;     -- Réduire le cache  
PRAGMA mmap_size = 0;           -- Désactiver memory mapping  
PRAGMA temp_store = FILE;       -- Temporaires sur disque  
```

### Problème 3 : Fichier WAL qui grandit

```sql
-- Le fichier WAL peut devenir gros, le nettoyer
PRAGMA wal_checkpoint(FULL);    -- Checkpoint complet

-- Ou revenir au mode DELETE temporairement
PRAGMA journal_mode = DELETE;  
PRAGMA journal_mode = WAL;      -- Rebasculer en WAL  
```

## Guide de référence rapide

### PRAGMA essentiels par priorité

1. **Performance immédiate :**
```sql
PRAGMA journal_mode = WAL;  
PRAGMA cache_size = -64000;  
```

2. **Optimisation approfondie :**
```sql
PRAGMA synchronous = NORMAL;  
PRAGMA temp_store = MEMORY;  
PRAGMA mmap_size = 268435456;  
```

3. **Maintenance :**
```sql
PRAGMA auto_vacuum = INCREMENTAL;  
PRAGMA optimize;  
```

### Commandes de diagnostic

```sql
-- Santé de la base
PRAGMA integrity_check;  
PRAGMA optimize;  

-- Informations système
PRAGMA page_count;  
PRAGMA freelist_count;  
PRAGMA database_list;  

-- Configuration actuelle
PRAGMA journal_mode;  
PRAGMA synchronous;  
PRAGMA cache_size;  
```

## Exercices pratiques

### Exercice 1 : Optimiser une base existante

1. **Créez une base de test avec beaucoup de données**
2. **Mesurez les performances par défaut**
3. **Appliquez les configurations recommandées**
4. **Mesurez l'amélioration**

```sql
-- Créer des données de test
CREATE TABLE test_performance (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    valeur REAL,
    date_creation DATE
);

-- Insérer 100 000 lignes
-- ⚠️ `RANDOM()` retourne un entier 64-bit SIGNÉ (peut être négatif).
--    Sans `ABS(...)`, `RANDOM() % 365` donne une valeur négative ~50 % du temps,
--    ce qui produit une chaîne `'--N days'` que `date()` interprète comme NULL.
--    Résultat : ~50 % des `date_creation` seraient NULL. D'où le `ABS()`.
WITH RECURSIVE cnt(x) AS (
    SELECT 1
    UNION ALL
    SELECT x+1 FROM cnt
    WHERE x < 100000
)
INSERT INTO test_performance (nom, valeur, date_creation)  
SELECT  
    'nom_' || x,
    ABS(RANDOM()) % 1000,                                          -- 0..999
    date('now', '-' || (ABS(RANDOM()) % 365) || ' days')           -- 365 derniers jours
FROM cnt;

-- Test avant optimisation
.timer on
SELECT COUNT(*) FROM test_performance WHERE valeur > 500;

-- Appliquer optimisations et retester
```

### Exercice 2 : Configuration d'application

Créez un script qui :
1. Détecte le type d'usage (web, analytics, mobile)
2. Applique la configuration appropriée
3. Valide que la configuration est correcte
4. Mesure l'impact sur une requête type

## Résumé et bonnes pratiques

### Configurations recommandées par défaut

**Pour la plupart des applications :**
```sql
PRAGMA journal_mode = WAL;      -- Concurrence améliorée  
PRAGMA synchronous = NORMAL;    -- Bon compromis  
PRAGMA cache_size = -64000;     -- 64MB cache  
PRAGMA temp_store = MEMORY;     -- Temporaires en mémoire  
```

### Points clés à retenir

✅ **WAL mode** améliore la concurrence et les performances  
✅ **Cache plus grand** = moins d'accès disque = plus rapide  
✅ **NORMAL sync** est un bon compromis sécurité/performance  
✅ **Memory mapping** accélère l'accès aux gros fichiers  
✅ **Mesurer l'impact** de chaque changement

### Règles d'or

1. **Commencez simple** : WAL + cache plus grand
2. **Mesurez toujours** l'impact avant/après
3. **Adaptez au contexte** : dev ≠ prod ≠ mobile
4. **Documentez** vos configurations
5. **Monitorer** en production

**Rappel important :** Une bonne configuration PRAGMA peut améliorer les performances de 2x à 10x sans changer une ligne de votre code !

Félicitations ! Vous maîtrisez maintenant toutes les techniques d'optimisation SQLite : du planificateur de requêtes aux index, en passant par l'analyse des plans d'exécution et la configuration système. Votre expertise vous permettra de transformer n'importe quelle base SQLite lente en machine de guerre ultra-performante ! 🚀

⏭️ [Module 6 : Programmation avancée](/06-programmation-avancee-sqlite/README.md)
