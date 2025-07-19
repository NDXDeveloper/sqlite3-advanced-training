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
-- Voir la taille actuelle du cache (en pages)
PRAGMA cache_size;
-- Résultat typique : -2000 (2MB par défaut)

-- Augmenter le cache à 64MB
PRAGMA cache_size = -64000;  -- Le - indique des kilobytes

-- Ou en nombre de pages (1 page = 4KB par défaut)
PRAGMA cache_size = 16000;   -- 16000 pages = 64MB
```

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
| DELETE | Normale | Faible | Élevée | Par défaut |
| **WAL** | **Élevée** | **Élevée** | **Élevée** | **Recommandé** |
| MEMORY | Très élevée | Faible | Moyenne | Tests/cache |
| OFF | Maximale | Faible | Faible | Données temporaires |

**Mode WAL en détail :**
```sql
-- Activer le mode WAL (Write-Ahead Logging)
PRAGMA journal_mode = WAL;

-- Avantages du mode WAL :
-- ✅ Lectures et écritures simultanées
-- ✅ Meilleures performances générales
-- ✅ Récupération après crash améliorée
-- ✅ Transactions plus rapides
```

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

**Recommandations par contexte :**

**Pour le développement :**
```sql
-- Configuration rapide pour le développement
PRAGMA synchronous = OFF;
PRAGMA journal_mode = MEMORY;
-- Attention : risque de perte de données !
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

-- Augmenter la limite mémoire pour les temporaires
PRAGMA temp_store_directory = '';  -- Utilise le répertoire par défaut
```

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
PRAGMA threads = 4;                  -- Multi-threading si supporté
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

**Usage du mode incrémental :**
```sql
-- Activer le mode incrémental
PRAGMA auto_vacuum = INCREMENTAL;

-- Nettoyer manuellement quand nécessaire
PRAGMA incremental_vacuum(1000);  -- Nettoie 1000 pages
```

### 3. Clés étrangères

```sql
-- Voir si les clés étrangères sont activées
PRAGMA foreign_keys;

-- Activer les contraintes de clés étrangères
PRAGMA foreign_keys = ON;   -- ⚠️ Impact sur les performances d'écriture

-- Les désactiver temporairement pour des imports massifs
PRAGMA foreign_keys = OFF;
-- [Import de données]
PRAGMA foreign_keys = ON;
```

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
-- Configuration pour développement rapide
PRAGMA journal_mode = MEMORY;
PRAGMA synchronous = OFF;
PRAGMA cache_size = -32000;
PRAGMA temp_store = MEMORY;
PRAGMA foreign_keys = ON;       -- Garder les contraintes en dev
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
    """Configure SQLite pour de meilleures performances"""
    cursor = conn.cursor()

    # Configuration de base
    cursor.execute("PRAGMA journal_mode = WAL")
    cursor.execute("PRAGMA synchronous = NORMAL")
    cursor.execute("PRAGMA cache_size = -64000")  # 64MB
    cursor.execute("PRAGMA temp_store = MEMORY")
    cursor.execute("PRAGMA mmap_size = 268435456")  # 256MB

    conn.commit()

# Usage
conn = sqlite3.connect('ma_base.db')
configure_sqlite_performance(conn)
```

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
SELECT
    'Cache: ' || (SELECT * FROM pragma_cache_size()) || ' pages' as cache_info,
    'DB: ' || (SELECT * FROM pragma_page_count()) || ' pages' as db_info,
    'Ratio: ' || ROUND(
        CAST((SELECT * FROM pragma_cache_size()) AS FLOAT) /
        (SELECT * FROM pragma_page_count()) * 100, 2
    ) || '%' as cache_ratio;
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
WITH RECURSIVE cnt(x) AS (
    SELECT 1
    UNION ALL
    SELECT x+1 FROM cnt
    WHERE x < 100000
)
INSERT INTO test_performance (nom, valeur, date_creation)
SELECT
    'nom_' || x,
    RANDOM() % 1000,
    date('now', '-' || (RANDOM() % 365) || ' days')
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

⏭️
