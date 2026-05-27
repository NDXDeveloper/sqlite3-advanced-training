# SQLite3 Advanced Training 🗄️

> **Formation SQLite3 — du débutant au développeur avancé.** 9 modules · 47 sections · ~65 000 lignes de cours, des centaines d'exemples SQL et Python prêts à exécuter, une fiche de révision condensée. Tout en français, sous licence MIT.

---

## 🎯 Objectifs de la formation

À l'issue de ces 9 modules, vous serez capable de :

- Maîtriser les **fondamentaux** de SQLite3 et comprendre en quoi il diffère de MySQL/PostgreSQL
- Concevoir des **schémas relationnels propres** (normalisation, clés étrangères, triggers, vues)
- Écrire des **requêtes avancées** (CTE, fenêtrage, JSON, regex, sous-requêtes)
- **Optimiser les performances** (index, plans d'exécution, PRAGMA, mode WAL)
- Utiliser la **programmation avancée** : UDF, extensions, transactions, FTS5, backup API
- **Intégrer SQLite** dans vos projets Python, C, Java, JavaScript… et derrière une API REST
- Gérer la **sécurité et l'administration** : chiffrement (SQLCipher), audit, sauvegardes, monitoring
- Mener des **cas d'usage avancés** : mobile, analyse de données, migration, application web complète, tests

## ⚡ Pourquoi SQLite ?

- 📅 Conçu au **printemps 2000** par **D. Richard Hipp**, version 1.0 publiée en **août 2000** (plus de 25 ans de maturité)
- 🌍 **Le moteur de base de données le plus déployé au monde** : plus de **1 000 milliards** de bases SQLite actives (chaque smartphone en contient des centaines)
- 🏛️ Format **officiellement recommandé par la Library of Congress** pour la préservation numérique à long terme
- 🔓 **Domaine public** : aucune licence, aucune redevance, utilisable y compris commercialement
- 🛡️ **Engagement officiel** : format de fichier maintenu compatible **au moins jusqu'en 2050**
- 🧪 **Une des bases de code les plus testées au monde** : ratio de **590:1** entre code de test (~92 millions de lignes) et code de production (~156 000 lignes), couverture 100 % MC/DC (norme aéronautique DO-178B)
- 🪶 Bibliothèque légère (~1 Mo), zéro configuration, zéro administration

## 📚 Contenu de la formation

> 📖 **Vue d'ensemble détaillée** : voir [/SOMMAIRE.md](/SOMMAIRE.md) pour la table des matières complète, et [/Notes.md](/Notes.md) pour la **fiche de révision condensée** (cheat-sheet).

### Plan des 9 modules

| # | Module | Niveau | Lien |
|--:|--------|:------:|------|
| 1 | **Fondamentaux de SQLite3** — Architecture, installation, comparaison MySQL/PostgreSQL, limitations | 🟢 Débutant | [→ Module 1](/01-fondamentaux-sqlite3/README.md) |
| 2 | **Bases du langage SQL** — Types, CRUD, contraintes, requêtes essentielles | 🟢 Débutant | [→ Module 2](/02-bases-langage-sql-sqlite/README.md) |
| 3 | **Conception et modélisation avancée** — Normalisation, relations, FK, triggers, vues | 🟡 Intermédiaire | [→ Module 3](/03-conception-modelisation-avancee/README.md) |
| 4 | **Requêtes avancées** — Sous-requêtes, CTE récursives, window functions, JSON, REGEXP | 🟡 Intermédiaire | [→ Module 4](/04-requetes-avancees-optimisation/README.md) |
| 5 | **Optimisation des performances** — Planificateur, index, EXPLAIN QUERY PLAN, PRAGMA | 🟡 Intermédiaire | [→ Module 5](/05-optimisation-performances/README.md) |
| 6 | **Programmation avancée** — UDF, extensions, transactions, backup API, FTS5 | 🔴 Avancé | [→ Module 6](/06-programmation-avancee-sqlite/README.md) |
| 7 | **Intégration et APIs** — Python, C/Java/JS, ORM, API REST, synchronisation | 🔴 Avancé | [→ Module 7](/07-integration-apis/README.md) |
| 8 | **Sécurité et administration** — Chiffrement, permissions, audit, sauvegardes, monitoring | 🔴 Avancé | [→ Module 8](/08-securite-administration/README.md) |
| 9 | **Cas d'usage avancés** — Mobile, analytics, migration, projet web complet, tests | 🔴 Avancé | [→ Module 9](/09-cas-usage-avances-projets-pratiques/README.md) |

### 📋 Fiche de révision

Le fichier **[/Notes.md](/Notes.md)** condense toute la formation en une fiche transversale : concepts clés, syntaxes critiques par module, commandes du shell, tableau des classes de stockage et affinités, et tableau des versions clés de SQLite. Idéal pour réviser ou consulter pendant le code.

## 🚀 Comment utiliser cette formation

1. **Prérequis** : notions de base en informatique (terminal, fichiers, dossiers). Aucune connaissance préalable en bases de données n'est requise.
2. **Progression** : suivez les modules dans l'ordre numérique — chaque module s'appuie sur les précédents.
3. **Pratique** : ouvrez un terminal en parallèle et **reproduisez tous les exemples SQL**. La majorité des concepts ne s'ancrent qu'à la pratique.
4. **Fichier de révision** : gardez [Notes.md](/Notes.md) ouvert à côté — c'est l'aide-mémoire indispensable.

> 💡 La formation contient **uniquement des exemples intégrés aux fichiers `.md`** (pas de dossiers `exercices/` ou `projets/` séparés). Tout est lisible et exécutable directement depuis votre éditeur ou navigateur Markdown.

## 🛠️ Installation rapide

> ℹ️ La formation a été pensée pour fonctionner avec **toute version SQLite ≥ 3.38** (publiée début 2022). Au moment de la rédaction (mai 2026), la version stable est **3.53.x**. Pour les détails et le dépannage, voir [module 1.2 — Installation](/01-fondamentaux-sqlite3/02-installation-configuration-outils.md).

### 🍎 macOS
SQLite est **déjà installé** par défaut. Pour la dernière version :
```bash
brew install sqlite
```

### 🐧 Linux (Ubuntu/Debian)
```bash
sudo apt update  
sudo apt install sqlite3  
```
Autres distributions : `sudo dnf install sqlite` (Fedora) · `sudo pacman -S sqlite` (Arch) · `apk add sqlite` (Alpine).

### 🪟 Windows
Trois options au choix :
```powershell
winget install SQLite.SQLite      # Windows 10/11 — recommandé  
choco install sqlite              # via Chocolatey  
scoop install sqlite              # via Scoop  
```
Ou téléchargez `sqlite-tools-win-x64-*.zip` depuis [sqlite.org/download.html](https://sqlite.org/download.html) et ajoutez le dossier au `PATH`.

### Vérification
```bash
sqlite3 --version
# → 3.x.x  20XX-XX-XX ...
```

## 🎓 Niveau et progression

| Niveau | Modules | Durée estimée |
|--------|---------|---------------|
| 🟢 **Débutant** | 1-2 | 4-6 heures |
| 🟡 **Intermédiaire** | 3-5 | 8-12 heures |
| 🔴 **Avancé** | 6-9 | 15-25 heures |

**Durée totale indicative** : 30 à 45 heures pour parcourir l'intégralité avec manipulation des exemples.

## 💡 Conseils d'apprentissage

- **Manipulez en parallèle** : ouvrez un terminal SQLite à côté de chaque chapitre.
- **Ne sautez pas les exemples** : ils contiennent des subtilités que le texte seul n'explicite pas toujours.
- **Créez vos propres bases d'expérimentation** : la meilleure façon d'apprendre est de casser et reconstruire.
- **Consultez la documentation officielle** : c'est elle qui fait autorité — [sqlite.org/docs.html](https://sqlite.org/docs.html).
- **Revenez régulièrement à [Notes.md](/Notes.md)** : la fiche de révision est conçue pour se relire rapidement entre deux modules.

## 🔗 Ressources externes utiles

- 📖 [Documentation officielle SQLite](https://sqlite.org/docs.html) — la source de vérité
- 📐 [Limites officielles de SQLite](https://www.sqlite.org/limits.html) — limites théoriques et pratiques
- 🧪 [Tests et fiabilité de SQLite](https://www.sqlite.org/testing.html) — la méthodologie de test
- 🗺️ [Quand utiliser SQLite](https://www.sqlite.org/whentouse.html) — cas d'usage officiels
- 🛡️ [Comment corrompre une base SQLite](https://www.sqlite.org/howtocorrupt.html) — à lire pour savoir quoi éviter
- 🖥️ [DB Browser for SQLite](https://sqlitebrowser.org/) — interface graphique libre multi-plateforme
- 🛢️ [DBeaver Community](https://dbeaver.io/) — alternative GUI plus complète
- 🔍 [SQLite Viewer en ligne](https://sqliteviewer.app/) — ouvrir un `.db` dans le navigateur

### Outils complémentaires
- [Litestream](https://litestream.io/) — réplication continue vers S3/GCS/Backblaze
- [rqlite](https://rqlite.io/) — SQLite distribué via Raft
- [Turso](https://turso.tech/) — fork libSQL avec API HTTP et réplication
- [Cloudflare D1](https://developers.cloudflare.com/d1/) — SQLite as a Service

## 📄 Licence

Ce projet est sous licence **MIT**. Voir le fichier [LICENSE](/LICENSE) pour les détails.

> ℹ️ La formation est sous MIT, mais **SQLite lui-même est dans le domaine public** — vous pouvez l'utiliser librement dans tout contexte, y compris commercial.

## 👤 Auteur

**Nicolas DEOUX**
📧 [NDXdev@gmail.com](mailto:NDXdev@gmail.com)

---

*Formation SQLite3 — du débutant au développeur avancé · Édition 2026*

⏭️ **Commencer la formation :** [Module 1 — Fondamentaux de SQLite3](/01-fondamentaux-sqlite3/README.md)
