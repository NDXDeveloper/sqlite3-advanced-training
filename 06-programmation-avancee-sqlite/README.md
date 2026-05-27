🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6. Programmation avancée avec SQLite

## Introduction

Après avoir maîtrisé les fondamentaux de SQLite, les requêtes complexes et l'optimisation des performances, nous entrons maintenant dans le domaine de la programmation avancée. Cette section vous permettra de transformer SQLite d'un simple système de gestion de base de données en un outil puissant et extensible, parfaitement intégré dans vos applications.

### Pourquoi la programmation avancée avec SQLite ?

SQLite n'est pas seulement une base de données relationnelle classique. Sa nature embarquée et sa flexibilité en font une plateforme idéale pour :

- **Étendre les fonctionnalités** : Ajouter des fonctions métier spécifiques directement dans la base
- **Optimiser les performances** : Traiter les données au plus près de leur stockage
- **Créer des solutions sur mesure** : Développer des modules spécialisés pour des besoins particuliers
- **Intégrer des fonctionnalités avancées** : Recherche plein texte, analyse de données, traitement d'images

### Architecture extensible de SQLite

SQLite propose plusieurs mécanismes d'extension qui font sa richesse :

#### 1. Fonctions définies par l'utilisateur (UDF)
Les UDF permettent d'ajouter vos propres fonctions SQL, écrites dans le langage de votre choix (C, Python, etc.). Ces fonctions s'intègrent naturellement dans vos requêtes SQL comme des fonctions natives.

```sql
-- Exemple d'utilisation d'une UDF personnalisée
SELECT nom, calcul_distance_gps(lat1, lon1, lat2, lon2) as distance  
FROM trajets;  
```

#### 2. Extensions et modules chargeables
SQLite supporte un système d'extensions dynamiques qui permet de charger des modules à l'exécution. Ces extensions peuvent ajouter :
- De nouvelles fonctions
- Des types de données personnalisés
- Des algorithmes de tri spécialisés
- Des mécanismes d'indexation avancés

#### 3. Tables virtuelles
Mécanisme puissant qui permet de créer des "tables" qui ne stockent pas réellement de données mais qui interfacent avec des sources externes :
- Fichiers CSV ou JSON
- APIs REST
- Autres bases de données
- Systèmes de fichiers

### Cas d'usage de la programmation avancée

#### Applications métier
- **Finance** : Fonctions de calcul d'intérêts composés, d'amortissements
- **Géolocalisation** : Calculs de distances, zones géographiques
- **E-commerce** : Algorithmes de recommandation, calculs de prix dynamiques
- **Analyse de données** : Fonctions statistiques personnalisées

#### Intégration système
- **Migration de données** : Fonctions de transformation et nettoyage
- **Interfaçage** : Connexion avec des systèmes legacy
- **Traitement batch** : Automatisation de traitements complexes

### Gestion avancée des transactions

Au niveau programmation avancée, la gestion transactionnelle devient cruciale :

#### Modes de transaction
- **DEFERRED** : Transaction différée (par défaut) — verrous acquis au premier accès
- **IMMEDIATE** : Verrou d'écriture acquis dès `BEGIN`
- **EXCLUSIVE** : Verrou d'écriture + (en rollback journal seulement) blocage des lecteurs

> ℹ️ Ces trois noms désignent des **modes d'acquisition de verrous**, pas des niveaux d'isolation au sens ACID. SQLite a un seul niveau d'isolation : **SERIALIZABLE**. Détails dans la section 6.3.

#### Contrôle fin des transactions
```sql
-- Début de transaction avec niveau spécifique
BEGIN IMMEDIATE TRANSACTION;

-- Points de sauvegarde
SAVEPOINT etape1;
-- ... opérations ...
ROLLBACK TO etape1; -- ou RELEASE etape1;

-- Validation finale
COMMIT;
```

### Sauvegarde et récupération programmatique

SQLite offre des APIs avancées pour la sauvegarde :

#### Backup API
Permet la sauvegarde à chaud de bases de données en cours d'utilisation, avec :
- Copie incrémentale
- Gestion des verrous
- Progression trackable

#### Stratégies de récupération
- **Point-in-time recovery** : Restauration à un moment précis
- **Sauvegarde différentielle** : Optimisation de l'espace de stockage
- **Réplication** : Synchronisation entre instances

### Recherche plein texte (FTS)

SQLite intègre nativement un moteur de recherche plein texte sophistiqué :

#### FTS5 - Fonctionnalités avancées
- **Classement par pertinence** : Scoring automatique avec `bm25()`
- **Highlight** : Mise en évidence des termes via `snippet()` et `highlight()`
- **Requêtes complexes** : Opérateurs booléens (`AND`/`OR`/`NOT`), proximité (`NEAR`), recherche par phrase
- **Préfixes & stemming** : Auto-complétion via `prefix='2 3 4'`, racinisation anglaise via le tokenizer `porter`

> ℹ️ FTS5 ne fournit **pas nativement** de recherche floue/Levenshtein (tolérance aux fautes de frappe). Pour cela, il faut combiner FTS5 avec une extension externe (par exemple `spellfix1`) ou implémenter une logique applicative au-dessus.

```sql
-- Création d'une table FTS5
CREATE VIRTUAL TABLE articles_fts USING fts5(
    titre,
    contenu,
    content='articles',
    content_rowid='id'
);

-- Recherche avec classement
SELECT articles.*, rank  
FROM articles_fts  
JOIN articles ON articles.id = articles_fts.rowid  
WHERE articles_fts MATCH 'sqlite AND programmation'  
ORDER BY rank;  
```

### Gestion d'erreurs et debugging

La programmation avancée nécessite une gestion robuste des erreurs :

#### Codes d'erreur SQLite
- **SQLITE_OK** (0) : Succès
- **SQLITE_ERROR** (1) : Erreur SQL générique
- **SQLITE_BUSY** (5) : Base occupée
- **SQLITE_CONSTRAINT** (19) : Violation de contrainte

#### Techniques de debugging
- **Logs détaillés** : Traçage des opérations
- **Profiling** : Mesure des performances
- **Tests unitaires** : Validation automatisée

### Performance et monitoring

Au niveau avancé, le monitoring devient essentiel :

#### Métriques clés
- Temps de réponse des requêtes
- Utilisation mémoire
- Taille des fichiers de base
- Fréquence des opérations de vacuum

#### Optimisation continue
- **Auto-vacuum** : Configuration automatique
- **WAL mode** : Write-Ahead Logging pour les performances
- **Memory-mapped I/O** : Accès direct en mémoire

### Prérequis pour cette section

Pour aborder efficacement cette section, vous devriez maîtriser :
- Les bases du SQL et de SQLite (sections 1-2)
- Les requêtes avancées (section 4)
- L'optimisation des performances (section 5)
- Un langage de programmation (Python, C, Java recommandés)

### Structure de la section

Cette section se décompose en **six parties complémentaires** :

| # | Section | Sujet principal | Lien |
|--:|---------|-----------------|------|
| 6.1 | **Fonctions définies par l'utilisateur (UDF)** | Création de fonctions SQL personnalisées (scalaires, agrégats, fenêtrage) | [→ 6.1](/06-programmation-avancee-sqlite/01-fonctions-definies-utilisateur-udf.md) |
| 6.2 | **Extensions et modules chargeables** | Développement et chargement d'extensions C/Python | [→ 6.2](/06-programmation-avancee-sqlite/02-extensions-sqlite-modules-chargeables.md) |
| 6.3 | **Gestion des transactions** | Modes DEFERRED/IMMEDIATE/EXCLUSIVE, savepoints, mode WAL | [→ 6.3](/06-programmation-avancee-sqlite/03-gestion-transactions-niveaux-isolation.md) |
| 6.4 | **Sauvegarde et restauration (backup API)** | API native, stratégies de sauvegarde et de récupération | [→ 6.4](/06-programmation-avancee-sqlite/04-sauvegarde-restauration-backup-api.md) |
| 6.5 | **Gestion des erreurs et exceptions** | Hiérarchie d'exceptions, retry, fallback, monitoring | [→ 6.5](/06-programmation-avancee-sqlite/05-gestion-erreurs-exceptions.md) |
| 6.6 | **Recherche plein texte avec FTS5** | Table virtuelle FTS5, bm25, snippet, tokenizers | [→ 6.6](/06-programmation-avancee-sqlite/06-recherche-plein-texte-avec-fts5.md) |

Chaque sous-section combinera théorie, exemples pratiques et projets concrets pour vous permettre de maîtriser ces aspects avancés de SQLite.

---

*Cette section vous donnera les clés pour transformer SQLite en un véritable outil de développement d'applications robustes et performantes.*

⏭️ [6.1 Fonctions définies par l'utilisateur (UDF)](/06-programmation-avancee-sqlite/01-fonctions-definies-utilisateur-udf.md)
