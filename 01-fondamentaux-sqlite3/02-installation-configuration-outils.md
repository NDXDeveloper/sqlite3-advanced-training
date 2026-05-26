🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.2 Installation et configuration des outils

## Vue d'ensemble

Pour travailler avec SQLite, nous avons besoin de plusieurs outils :
- **SQLite3** : Le moteur de base de données lui-même
- **Un client en ligne de commande** : Pour exécuter des requêtes SQL
- **Un éditeur de texte** : Pour écrire nos scripts SQL
- **Optionnel** : Une interface graphique pour visualiser nos données

Ne vous inquiétez pas, nous allons tout installer ensemble étape par étape !

## Étape 1 : Vérifier si SQLite est déjà installé

Avant d'installer quoi que ce soit, vérifions si SQLite n'est pas déjà présent sur votre système.

### Sur tous les systèmes

Ouvrez un **terminal** (ou **invite de commande** sur Windows) et tapez :

```bash
sqlite3 --version
```

Si vous voyez quelque chose comme `3.53.1 2026-05-05 ...`, félicitations ! SQLite est déjà installé. Toute version `3.x` récente (à partir de 3.38 environ) conviendra parfaitement à cette formation.

Si vous obtenez une erreur comme `'sqlite3' n'est pas reconnu`, pas de panique, nous allons l'installer.

> ℹ️ **Quelle version utiliser ?** Au moment de la rédaction (mai 2026), la dernière version stable est **3.53.1**. La formation a été pensée pour être compatible avec **toute version 3.38 ou supérieure** (publiée début 2022) — la majorité des distributions Linux à jour, macOS récent et Windows 10/11 fournissent au moins cette version.

### Comment ouvrir un terminal ?

**Windows** :
- Appuyez sur `Win + R`, tapez `cmd` et appuyez sur Entrée
- Ou cherchez « Invite de commandes » (ou « Terminal Windows ») dans le menu Démarrer

**macOS** :
- Appuyez sur `Cmd + Espace`, tapez « Terminal » et appuyez sur Entrée
- Ou allez dans *Applications > Utilitaires > Terminal*

**Linux** :
- Appuyez sur `Ctrl + Alt + T`
- Ou cherchez « Terminal » dans vos applications

## Étape 2 : Installation de SQLite

### 🍎 Sur macOS

**Option 1 : SQLite est pré-installé !**
SQLite est inclus avec macOS, donc vous n'avez rien à faire. Testez avec `sqlite3 --version`.

> ⚠️ La version livrée par Apple est souvent **plus ancienne** que la dernière release (Apple met à jour SQLite lors des grandes versions de macOS). Pour cette formation, toute version ≥ 3.38 suffira ; si vous avez besoin de fonctionnalités récentes (JSONB, ANALYZE amélioré…), passez par Homebrew.

**Option 2 : Installation via Homebrew (pour avoir la dernière version)**
```bash
# Installer Homebrew si pas déjà fait
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Installer SQLite
brew install sqlite

# La version Homebrew n'est pas dans le PATH par défaut sur Apple Silicon ;
# vous pouvez l'invoquer explicitement :
$(brew --prefix sqlite)/bin/sqlite3 --version
```

### 🐧 Sur Linux (Ubuntu/Debian)

```bash
# Mettre à jour la liste des paquets
sudo apt update

# Installer SQLite
sudo apt install sqlite3

# Vérifier l'installation
sqlite3 --version
```

**Pour d'autres distributions Linux :**
- **Fedora / RHEL / CentOS Stream** : `sudo dnf install sqlite`
- **openSUSE** : `sudo zypper install sqlite3`
- **Arch Linux / Manjaro** : `sudo pacman -S sqlite`
- **Alpine Linux** : `apk add sqlite`

> 💡 **WSL (Windows Subsystem for Linux)** : si vous développez sur Windows mais préférez l'environnement Linux, WSL2 + Ubuntu est une excellente option. L'installation y est identique à Ubuntu/Debian ci-dessus.

### 🪟 Sur Windows

**Option 1 : Téléchargement direct (Recommandé pour débutants)**

1. Allez sur [https://sqlite.org/download.html](https://sqlite.org/download.html)
2. Dans la section *« Precompiled Binaries for Windows »*, téléchargez :
   - **`sqlite-tools-win-x64-XXXXXXX.zip`** pour Windows 64 bits moderne (recommandé)
   - ou `sqlite-tools-win-arm64-XXXXXXX.zip` pour les machines Windows ARM (Surface ARM, etc.)
3. Décompressez le fichier ZIP dans un dossier, par exemple `C:\sqlite`
4. **Important** : ajoutez ce dossier au PATH de Windows

> 💡 Le pack `sqlite-tools-…` contient `sqlite3.exe` (le shell interactif) ainsi que des utilitaires bonus : `sqldiff.exe` (comparer deux bases), `sqlite3_analyzer.exe` (statistiques détaillées) et `sqlite3_rsync.exe` (synchronisation).

**Comment ajouter SQLite au PATH sur Windows :**

1. Ouvrez « Paramètres système avancés » (recherchez dans le menu Démarrer)
2. Cliquez sur « Variables d'environnement »
3. Dans « Variables système », trouvez et sélectionnez `Path`
4. Cliquez sur « Modifier »
5. Cliquez sur « Nouveau »
6. Ajoutez le chemin vers votre dossier SQLite (ex : `C:\sqlite`)
7. Cliquez sur « OK » partout
8. **Redémarrez votre invite de commandes** (les variables d'environnement ne sont chargées qu'à l'ouverture d'un nouveau terminal)

**Option 2 : Via Windows Package Manager (winget — déjà présent sur Windows 10/11)**
```powershell
winget install SQLite.SQLite
```

**Option 3 : Via Chocolatey**
```powershell
choco install sqlite
```

**Option 4 : Via Scoop**
```powershell
scoop install sqlite
```

## Étape 3 : Test de l'installation

Une fois l'installation terminée, testez que tout fonctionne :

```bash
# Vérifier la version
sqlite3 --version

# Créer une base de données de test (ou ouvrir une existante)
sqlite3 test.db

# Vous devriez voir l'invite SQLite :
# sqlite>
```

Si vous voyez `sqlite>`, c'est parfait ! Vous êtes dans l'interface en ligne de commande de SQLite.

Pour sortir, plusieurs options :
- taper la **méta-commande** `.quit` (ou `.exit`),
- ou appuyer sur **Ctrl+D** (Linux/macOS) / **Ctrl+Z** puis Entrée (Windows) pour signaler la fin d'entrée.

> 💡 **Sans entrer dans le shell** : on peut aussi exécuter une requête en une seule ligne depuis le terminal système :
> ```bash
> sqlite3 test.db "SELECT sqlite_version();"
> ```
> Pratique pour les scripts shell ou les vérifications rapides.

## Étape 4 : Interface en ligne de commande - Premier contact

Reprenons notre session SQLite :

```bash
sqlite3 test.db
```

Voici les commandes de base à connaître.

> ⚠️ **Distinction importante** : les commandes qui **commencent par un point** (`.help`, `.tables`, `.quit`…) sont des **commandes du shell `sqlite3`** (le programme en ligne de commande), **pas** des instructions SQL. Elles ne fonctionnent qu'à l'intérieur du shell, jamais depuis un programme (Python, Java…). En revanche, les **commandes SQL** (qui se terminent par `;`) fonctionnent partout.

### Commandes du shell (commencent par un point)

```text
-- AIDE & EXPLORATION
.help              Affiche la liste complète des commandes ;
                   .help PATTERN cherche celles qui contiennent PATTERN
.databases         Liste les bases actuellement ouvertes (main, temp, attachées)
.tables            Liste les tables (et vues) de la base courante
.schema            Affiche TOUTES les instructions CREATE de la base
.schema users      Idem mais seulement pour la table « users »
.indexes [TABLE]   Liste les index (optionnellement filtrés par table)
.show              Affiche les paramètres actuels du shell (mode, headers, etc.)

-- AFFICHAGE
.headers on        Affiche les en-têtes de colonnes
.mode column       Affichage en colonnes alignées
.mode box          Affichage avec bordures Unicode (joli, depuis 3.33)
.mode qbox         Idem box + chaînes entre guillemets (depuis 3.33)
.mode table        Format Markdown (réutilisable dans la doc)
.mode csv          Sortie CSV
.mode json         Sortie JSON
.nullvalue NULL    Comment afficher les valeurs NULL (par défaut : vide)
.width 20 10 30    Force la largeur des colonnes en mode column

-- FICHIERS
.read mon_script.sql       Exécute les instructions SQL d'un fichier
.output resultat.txt       Redirige la sortie vers un fichier
.output stdout             Revient à l'affichage écran
.dump                      Exporte tout le schéma + données en SQL portable
.backup sauvegarde.db      Copie la base entière (transaction-safe)

-- PERFORMANCE
.timer on          Affiche le temps d'exécution après chaque requête
.stats on          Statistiques détaillées (lectures, écritures)
.eqp on            EXPLAIN QUERY PLAN automatique sur chaque requête

-- QUITTER
.quit              ou .exit
```

> 💡 Astuce : tapez **`.help`** seul pour la liste complète, ou **`.help mode`** pour ne voir que les commandes liées au mode d'affichage.

### Votre première requête SQL

Activons d'abord un affichage lisible, puis créons une petite table :

```sql
-- Affichage : en-têtes et colonnes alignées
.headers on
.mode column

-- Créer une table simple
CREATE TABLE personnes (
    nom TEXT,
    age INTEGER
);

-- Insérer des données (une seule instruction multi-lignes possible)
INSERT INTO personnes (nom, age) VALUES
    ('Alice', 25),
    ('Bob',   30);

-- Voir les données
SELECT * FROM personnes;
```

Vous devriez obtenir un affichage du genre :

```
nom    age
-----  ---
Alice  25  
Bob    30  
```

Essayez aussi `.mode box` pour un rendu plus joli :

```
┌───────┬─────┐
│  nom  │ age │
├───────┼─────┤
│ Alice │ 25  │
│ Bob   │ 30  │
└───────┴─────┘
```

> 💡 **Conseil pédagogique** : prenez l'habitude de terminer chaque instruction SQL par un **point-virgule** (`;`). Le shell `sqlite3` continue d'attendre la suite tant qu'il ne voit pas de `;` — c'est ce qui vous permet d'écrire une requête sur plusieurs lignes. Si vous voyez l'invite changer de `sqlite>` à `   ...>`, c'est qu'il vous attend !  
>  
> 🆘 **Coincé dans `...>` ?** Si vous avez tapé une commande mal formée et que SQLite reste en attente, tapez `;` sur une ligne vide pour terminer, ou utilisez `.cancel` (depuis 3.39) pour annuler la commande en cours.

## Étape 5 : Outils complémentaires recommandés

### Éditeurs de texte avec coloration SQL

**Gratuits et simples :**
- **Visual Studio Code** : excellent support SQL, multi-OS
- **Notepad++** (Windows) : léger avec coloration syntaxique
- **Sublime Text** : rapide et élégant

**Installation de VS Code (recommandé) :**
1. Allez sur [https://code.visualstudio.com](https://code.visualstudio.com)
2. Téléchargez et installez
3. Installez l'extension **« SQLite »** d'**alexcvzz** (identifiant `alexcvzz.vscode-sqlite`) : elle permet d'ouvrir directement des fichiers `.db` dans VS Code et d'exécuter des requêtes depuis un fichier `.sql` (clic-droit → *Run Query*)

### Interfaces graphiques (optionnelles)

Pour visualiser vos données plus facilement :

**Gratuits, open source :**
- **DB Browser for SQLite** : simple, efficace, multi-plateforme (Windows, macOS, Linux)
  - Téléchargez sur [https://sqlitebrowser.org](https://sqlitebrowser.org)
- **DBeaver Community** : plus avancé, gère aussi de nombreux autres SGBD
  - Téléchargez sur [https://dbeaver.io](https://dbeaver.io)
- **SQLiteStudio** : alternative légère et complète
  - Téléchargez sur [https://sqlitestudio.pl](https://sqlitestudio.pl)

**En ligne (aucune installation) :**
- **SQLite Viewer** : [https://sqliteviewer.app](https://sqliteviewer.app) — parfait pour examiner rapidement un fichier `.db` sans rien installer

### Clients en ligne de commande améliorés

Le shell `sqlite3` officiel est très bien, mais si vous voulez l'**auto-complétion**, la **coloration syntaxique** et un **historique multi-ligne**, jetez un œil à :

- **litecli** : `pip install litecli` — client moderne avec auto-complétion intelligente
  - Site : [https://litecli.com](https://litecli.com)

## Étape 6 : Configuration de l'environnement de travail

Créons un dossier pour notre formation :

```bash
# Créer un dossier de travail
mkdir formation-sqlite  
cd formation-sqlite  

# Créer des sous-dossiers
mkdir bases-donnees  
mkdir scripts  

# Structure obtenue :
# formation-sqlite/
# ├── bases-donnees/    (nos fichiers .db)
# └── scripts/          (nos fichiers .sql)
```

### Astuce : un fichier de configuration personnel `~/.sqliterc`

Le shell `sqlite3` lit automatiquement un fichier `~/.sqliterc` au démarrage (sous Linux/macOS) ou `%USERPROFILE%\.sqliterc` (sous Windows). On peut y placer ses préférences par défaut :

```text
-- ~/.sqliterc : appliqué à chaque session sqlite3

-- Affichage soigné
.headers on
.mode column

-- Afficher explicitement les valeurs NULL au lieu d'une cellule vide
.nullvalue NULL

-- Décommentez la ligne ci-dessous si vous voulez chronométrer chaque requête
-- .timer on
```

Ainsi, plus besoin de retaper ces commandes à chaque fois !

> ℹ️ Ce fichier peut aussi contenir des **instructions SQL** qui s'exécutent à l'ouverture de chaque base — par exemple `PRAGMA foreign_keys = ON;` que nous verrons au module 3.

## Vérification finale - Checklist

Vérifiez que tout fonctionne :

- [ ] `sqlite3 --version` affiche une version
- [ ] `sqlite3 test.db` ouvre l'interface SQLite
- [ ] `.help` dans SQLite affiche l'aide
- [ ] Vous pouvez créer une table et insérer des données
- [ ] Vous avez un éditeur de texte configuré
- [ ] Votre dossier de travail est créé

## Résolution de problèmes courants

### « sqlite3 n'est pas reconnu » (Windows)

**Solutions :**
1. Vérifiez que le dossier d'installation est bien dans votre `PATH` (variables d'environnement → Path)
2. **Redémarrez votre invite de commandes** : les variables d'environnement ne sont chargées qu'à l'ouverture d'un nouveau terminal
3. Sinon, utilisez le chemin complet : `C:\sqlite\sqlite3.exe`

### « Permission denied » (Linux/macOS)

**Solutions :**
1. Vérifiez que vous avez les droits d'écriture dans le dossier où vous lancez `sqlite3 ma_base.db` (la base est créée sur place !)
2. Utilisez `sudo` **uniquement pour l'installation**, jamais pour ouvrir vos propres bases
3. Si vous avez téléchargé un binaire SQLite manuellement, rendez-le exécutable : `chmod +x sqlite3`

### « Database is locked »

**Causes courantes :**
1. Une autre session `sqlite3` est restée ouverte sur le même fichier — fermez-la (`.quit`)
2. Un éditeur graphique (DB Browser, DBeaver) tient le fichier — fermez la connexion dans l'éditeur
3. Une transaction `BEGIN` sans `COMMIT` ni `ROLLBACK` dans une autre session — terminez-la

### Fichier de base corrompu (rarissime)

**Diagnostic :**
```sql
PRAGMA integrity_check;     -- doit renvoyer "ok"
```

**Solutions :**
1. Sauvegardez d'abord ! `cp ma_base.db ma_base.bak.db`
2. Tentez une récupération via le shell : `sqlite3 ma_base.db ".recover" | sqlite3 nouvelle_base.db`
3. En dernier recours, supprimez et recréez : `rm test.db && sqlite3 nouvelle_base.db`

## 🎯 Mise en pratique guidée

Maintenant que tout est installé, créons notre première vraie base de données.

**Étape 1 — Ouvrir le shell sur une nouvelle base :**

```bash
sqlite3 bibliotheque.db
```

**Étape 2 — Définir la structure et insérer quelques données :**

```sql
CREATE TABLE livres (
    titre  TEXT,
    auteur TEXT,
    annee  INTEGER
);

INSERT INTO livres VALUES ('1984', 'George Orwell', 1949);  
INSERT INTO livres VALUES ('Le Petit Prince', 'Antoine de Saint-Exupéry', 1943);  
INSERT INTO livres VALUES ('Harry Potter à l''école des sorciers', 'J.K. Rowling', 1997);  
```

> 💡 Remarquez l'apostrophe doublée dans `l''école` : en SQL, on échappe un apostrophe en le **doublant** à l'intérieur d'une chaîne entre apostrophes.

**Étape 3 — Lire les données et quitter :**

```sql
.headers on
.mode column
SELECT * FROM livres;
.quit
```

**Étape 4 — Vérifier que le fichier existe** (depuis le terminal système) :

```bash
ls -lh bibliotheque.db
```

Vous devriez voir un fichier de quelques kilo-octets contenant toute la base !

## Récapitulatif

**Ce que nous avons accompli :**
- ✅ Installation de SQLite sur votre système
- ✅ Configuration de l'environnement de travail
- ✅ Premier contact avec l'interface en ligne de commande
- ✅ Installation d'outils complémentaires
- ✅ Création de votre première base de données

**À retenir :**
- SQLite fonctionne directement en ligne de commande via `sqlite3`
- Les commandes du **shell** commencent par un point (`.help`, `.quit`) — elles n'existent **que dans le shell `sqlite3`**
- Les commandes **SQL** se terminent par un point-virgule (`;`) et fonctionnent partout
- Un fichier `.db` (ou `.sqlite`) contient toute votre base de données
- Le fichier `~/.sqliterc` permet de personnaliser le shell de façon permanente
- Vous avez maintenant tous les outils pour commencer !

---

**➡️ Dans le prochain chapitre**, nous verrons en détail les différences entre SQLite et les autres systèmes de bases de données comme MySQL et PostgreSQL.

⏭️ [1.3 Différences avec les autres SGBD (MySQL, PostgreSQL)](/01-fondamentaux-sqlite3/03-differences-autres-sgbd-mysql-postgresql.md)
