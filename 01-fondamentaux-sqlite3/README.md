🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 1 : Fondamentaux de SQLite3

> **Niveau** : 🟢 Débutant · **Durée estimée** : 2 à 3 heures de lecture + manipulation · **Prérequis** : aucune connaissance en bases de données

## Objectifs du module

À l'issue de ce module, vous serez capable de :

- ✅ Comprendre ce qu'est SQLite3 et ses spécificités uniques
- ✅ Installer SQLite et configurer un environnement de travail propre sur Windows, macOS ou Linux
- ✅ Identifier les différences clés entre SQLite et les autres systèmes de gestion de bases de données (SGBD)
- ✅ Comprendre l'architecture *serverless* et le concept de fichier de base unique
- ✅ Connaître les **vraies** limitations de SQLite (et les **mythes** à oublier) pour choisir le bon outil selon le contexte
- ✅ Lancer le shell `sqlite3`, créer une base, manipuler quelques tables avec des commandes SQL et avec les méta-commandes du shell (`.tables`, `.schema`, `.headers`, …)

## Prérequis

- Notions de base en informatique (fichiers, dossiers, ligne de commande)
- Aucune connaissance préalable en bases de données n'est requise
- Curiosité et envie d'apprendre !

## Plan du module

Ce module couvre les aspects fondamentaux essentiels pour bien commencer avec SQLite3 :

### [1.1 Introduction à SQLite : caractéristiques et cas d'usage](/01-fondamentaux-sqlite3/01-introduction-sqlite-caracteristiques-cas-usage.md)
Découverte de SQLite, ses points forts et ses domaines d'application privilégiés.

### [1.2 Installation et configuration des outils](/01-fondamentaux-sqlite3/02-installation-configuration-outils.md)
Guide pratique pour installer SQLite sur différents systèmes d'exploitation et configurer votre environnement de développement.

### [1.3 Différences avec les autres SGBD (MySQL, PostgreSQL)](/01-fondamentaux-sqlite3/03-differences-autres-sgbd-mysql-postgresql.md)
Comparaison détaillée pour comprendre quand choisir SQLite plutôt qu'une autre solution.

### [1.4 Architecture serverless et fichier de base unique](/01-fondamentaux-sqlite3/04-architecture-serverless-fichier-base-unique.md)
Exploration de ce qui rend SQLite unique : son architecture sans serveur et sa philosophie du fichier unique.

### [1.5 Limitations et contraintes de SQLite](/01-fondamentaux-sqlite3/05-limitations-contraintes-sqlite.md)
Tour d'horizon des limites techniques et fonctionnelles à connaître avant de se lancer.

## Pourquoi commencer par les fondamentaux ?

SQLite3 n'est pas « juste une autre base de données ». Sa philosophie et son architecture uniques en font un outil à part dans le paysage des SGBD. Comprendre ses spécificités dès le départ vous permettra de :

- **Éviter les pièges** : Ne pas appliquer les réflexes des SGBD traditionnels là où ils ne conviennent pas
- **Maximiser ses avantages** : Tirer parti de ses points forts (simplicité, portabilité, performance)
- **Choisir en connaissance de cause** : Savoir quand SQLite est le bon choix pour votre projet
- **Optimiser votre apprentissage** : Comprendre le « pourquoi » avant le « comment »

## À retenir

> SQLite est **différent** par design. Il ne s'agit pas d'une version simplifiée d'un SGBD traditionnel, mais d'une approche fondamentalement différente de la gestion des données. Cette différence est sa plus grande force… et peut parfois être source de confusion si on l'aborde avec les mauvaises attentes.

## Quelques chiffres pour situer SQLite

- 📅 Créé en **mai 2000** par **D. Richard Hipp** (plus de 25 ans de maturité)
- 🌍 **Le moteur de base de données le plus déployé au monde** : plus de **1 000 milliards** de bases actives selon ses auteurs (chaque smartphone en contient des centaines)
- 🏛️ Format de fichier **recommandé par la Library of Congress** pour la préservation numérique
- 🔓 **Domaine public** : aucune licence, aucune redevance
- 🛡️ Engagement officiel des développeurs : **support du format de fichier jusqu'en 2050 au minimum**
- 🧪 **Une des bases de code les plus testées au monde** : ratio de **590:1** entre code de test (~92 millions de lignes) et code de production (~156 000 lignes), couverture 100 % MC/DC (norme aéronautique DO-178B)

## Carte mentale du module

```
                    ┌──────────────────────────┐
                    │ Module 1 : Fondamentaux  │
                    └──────────────┬───────────┘
                                   │
        ┌──────────────┬───────────┼───────────┬──────────────┐
        ▼              ▼           ▼           ▼              ▼
   1.1 Qu'est-ce  1.2 Comment   1.3 En quoi  1.4 Comment   1.5 Quelles
   que c'est ?    l'installer ? c'est        ça marche     sont ses
                                différent ?  dedans ?      limites ?
```

## Comment lire ce module

- **Lisez dans l'ordre** : chaque section construit sur la précédente.
- **Manipulez en parallèle** : ouvrez un terminal et reproduisez les exemples — c'est ainsi qu'on retient.
- **Survolez d'abord la conclusion** de chaque section (« Récapitulatif » / « À retenir ») : c'est un bon repère pour savoir où vous allez.
- **Aucune connaissance préalable** n'est supposée : tout terme technique nouveau est défini à sa première apparition.

### Conventions de typographie utilisées dans ce module

| Élément | Signification |
|---------|---------------|
| `code en chasse fixe` | Une commande shell, un mot-clé SQL ou un nom de fichier |
| 💡 | Astuce ou bonne pratique |
| ⚠️ | Piège à éviter ou point qui mérite attention |
| ℹ️ | Information complémentaire ou contexte historique |
| ✅ / ❌ | Bonne / mauvaise approche, ou fonctionnalité disponible / absente |

### Glossaire express (vous croiserez ces termes — ne paniquez pas)

| Sigle | Signification |
|-------|---------------|
| **SGBD** | Système de Gestion de Base de Données (Database Management System en anglais) |
| **SQL** | Structured Query Language — le langage standard d'interrogation des bases relationnelles |
| **ACID** | Atomicity, Consistency, Isolation, Durability — les 4 propriétés des transactions fiables |
| **WAL** | Write-Ahead Logging — mode de journalisation amélioré de SQLite |
| **MVCC** | Multi-Version Concurrency Control — modèle de concurrence utilisé par PostgreSQL/MySQL (InnoDB) |
| **CRUD** | Create, Read, Update, Delete — les 4 opérations de base sur les données |
| **PRAGMA** | Instructions spéciales de configuration propres à SQLite |
| **B-tree** | Arbre B équilibré — structure de données utilisée pour le stockage des tables et index |

> Tous ces concepts seront expliqués dans ce module ou dans les suivants. Pas besoin de tout retenir tout de suite.

---

**Prêt à découvrir SQLite3 ?** Passons maintenant au premier chapitre pour comprendre ce qui rend cette base de données si particulière et populaire.

⏭️ [1.1 Introduction à SQLite : caractéristiques et cas d'usage](/01-fondamentaux-sqlite3/01-introduction-sqlite-caracteristiques-cas-usage.md)
