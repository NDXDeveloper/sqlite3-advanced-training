🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 5 : Optimisation des performances

## Introduction

L'optimisation des performances est un aspect crucial du développement avec SQLite3, particulièrement lorsque vos applications traitent de gros volumes de données ou nécessitent des temps de réponse rapides. Bien que SQLite soit réputé pour sa simplicité et sa légèreté, des techniques d'optimisation appropriées peuvent considérablement améliorer ses performances.

### Pourquoi optimiser SQLite ?

SQLite, malgré sa réputation de base de données "légère", peut gérer des bases de données de plusieurs téraoctets et traiter des milliers de requêtes par seconde lorsqu'il est correctement optimisé. Cependant, sans optimisation, même des opérations simples peuvent devenir lentes sur des datasets modérément importants.

**Problèmes de performance courants :**
- Requêtes dont le temps croît **linéairement** (`SCAN` complet, *O(n)*) ou **quadratiquement** (jointure sans index, *O(n × m)*) avec la taille des données
- Opérations d'écriture qui bloquent les lectures pendant de longues périodes
- Consommation excessive de mémoire lors du traitement de grandes requêtes
- Temps d'attente élevés lors d'opérations concurrentes

### Architecture et spécificités de SQLite

Pour optimiser efficacement SQLite, il est essentiel de comprendre ses caractéristiques uniques :

**Architecture serverless :**
- Pas de processus serveur séparé
- Accès direct au fichier de base de données
- Moins de latence réseau mais limitations d'accès concurrent

**Moteur de stockage unique :**
- Un seul fichier pour toute la base de données
- Gestion des pages de données de taille fixe (généralement 4096 octets)
- **Cache par connexion** par défaut (chaque connexion a son propre cache mémoire) ; un mode « shared cache » optionnel existe mais est **officiellement déconseillé** par les développeurs SQLite — préférer le mode WAL pour la concurrence

**Modèle de verrouillage :**
- Verrouillage au niveau du fichier (database-level locking) en mode rollback journal classique ; verrous plus fins (1 writer + N readers) en mode WAL
- Transactions ACID avec un **seul niveau d'isolation** : `SERIALIZABLE` par défaut (`READ UNCOMMITTED` n'existe qu'en mode `shared cache`, cas rare)
- Mécanisme de Write-Ahead Logging (WAL) pour améliorer la concurrence

### Métriques de performance à surveiller

**Temps de réponse des requêtes :**
```sql
-- Activation du timing pour mesurer les performances
.timer on
SELECT COUNT(*) FROM large_table WHERE condition;
```

**Utilisation des ressources :**
- Consommation mémoire du cache
- Nombre de pages lues/écrites
- Fragmentation du fichier de base de données

**Concurrence :**
- Nombre de verrous en attente
- Durée des transactions
- Conflits de lecture/écriture

### Outils d'analyse des performances

**Commandes SQLite intégrées :**
```sql
-- Statistiques sur l'utilisation de la base
.dbinfo

-- Informations sur le cache et les pages
.stats on

-- Profiling des requêtes
.eqp on  -- Affiche le plan d'exécution
```

**PRAGMA d'introspection (structure et configuration, pas statistiques) :**
```sql
-- Configuration / environnement
PRAGMA compile_options;     -- flags de compilation utilisés pour le binaire SQLite  
PRAGMA database_list;       -- bases attachées (main, temp, ATTACHées)  
PRAGMA table_info(nom_table); -- colonnes de la table (nom, type, NOT NULL, PK)  

-- Informations sur les index
PRAGMA index_list(nom_table);  -- index existants sur la table  
PRAGMA index_info(nom_index);  -- colonnes couvertes par un index donné  

-- Pour les vraies STATISTIQUES (utilisées par le planificateur), c'est :
-- ANALYZE; puis SELECT * FROM sqlite_stat1;
```

### Méthodologie d'optimisation

**1. Mesurer avant d'optimiser :**
Toujours établir une baseline de performance avant d'apporter des modifications. Utilisez des jeux de données réalistes et des requêtes représentatives de votre usage.

**2. Identifier les goulots d'étranglement :**
- Requêtes les plus lentes
- Tables sans index appropriés
- Opérations qui scannent entièrement les tables

**3. Optimiser de manière itérative :**
- Une modification à la fois
- Mesurer l'impact de chaque changement
- Valider que l'optimisation ne casse pas d'autres fonctionnalités

**4. Tester en conditions réelles :**
- Volume de données représentatif
- Patterns d'accès concurrent
- Contraintes matérielles similaires à la production

### Types d'optimisations SQLite

**Optimisations structurelles :**
- Conception de schéma efficace
- Normalisation appropriée
- Choix des types de données optimaux

**Optimisations d'index :**
- Index sur les colonnes de recherche fréquente
- Index composites pour les requêtes multi-colonnes
- Gestion de la fragmentation des index

**Optimisations de requêtes :**
- Réécriture des requêtes complexes
- Utilisation appropriée des jointures
- Optimisation des sous-requêtes

**Optimisations de configuration :**
- Paramètres PRAGMA adaptés à l'usage
- Configuration du cache et de la mémoire
- Choix du mode journal approprié

### Limites et compromis

**Limitations de SQLite à considérer :**
- Concurrence limitée en écriture (**un seul writer** simultané, même en WAL)
- Pas de parallélisation **d'une même requête** (threads auxiliaires possibles uniquement pour les `ORDER BY` et création d'index via `PRAGMA threads`)
- Pas d'optimisation distribuée (pas de sharding natif, à émuler avec `ATTACH` ou des wrappers comme Litestream/rqlite)

**Compromis performance vs autres critères :**
- **Performance vs Simplicité :** Des optimisations complexes peuvent nuire à la maintenabilité
- **Performance vs Sécurité :** Certaines optimisations peuvent réduire la durabilité des données
- **Performance vs Portabilité :** Des optimisations spécifiques peuvent limiter la compatibilité

### Objectifs de ce chapitre

À la fin de ce chapitre, vous serez capable de :

1. **Diagnostiquer** les problèmes de performance dans vos applications SQLite
2. **Analyser** les plans d'exécution pour identifier les optimisations possibles
3. **Créer et gérer** des index efficaces pour accélérer vos requêtes
4. **Configurer** SQLite pour obtenir les meilleures performances selon votre contexte
5. **Optimiser** les requêtes complexes et résoudre les problèmes de lenteur

### Structure du chapitre

| # | Section | Sujet principal | Lien |
|--:|---------|-----------------|------|
| 5.1 | **Planificateur de requêtes** | Comment SQLite choisit la méthode d'exécution | [→ 5.1](/05-optimisation-performances/01-comprendre-planificateur-requetes-sqlite.md) |
| 5.2 | **Index** | Création, maintenance, types d'index, index partiels | [→ 5.2](/05-optimisation-performances/02-creation-gestion-index.md) |
| 5.3 | **`EXPLAIN QUERY PLAN`** | Lire et interpréter les plans d'exécution | [→ 5.3](/05-optimisation-performances/03-analyse-plans-execution-explain-query-plan.md) |
| 5.4 | **Requêtes lentes** | Techniques de réécriture, sous-requêtes, jointures | [→ 5.4](/05-optimisation-performances/04-optimisation-requetes-lentes.md) |
| 5.5 | **PRAGMA** | `journal_mode`, `synchronous`, `cache_size`, `mmap_size` | [→ 5.5](/05-optimisation-performances/05-configuration-parametres-sqlite-pragma.md) |

Chaque section combinera théorie, exemples pratiques et exercices pour vous permettre d'appliquer immédiatement ces concepts à vos projets.

---

*Prêt à plonger dans l'optimisation SQLite ? La section suivante vous révélera les secrets du planificateur de requêtes, le cœur de l'intelligence de SQLite pour l'exécution efficace de vos requêtes.*

⏭️ [5.1 Comprendre le planificateur de requêtes SQLite](/05-optimisation-performances/01-comprendre-planificateur-requetes-sqlite.md)
