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

Si vous voyez quelque chose comme `3.42.0 2023-05-16 12:36:15`, félicitations ! SQLite est déjà installé.

Si vous obtenez une erreur comme `'sqlite3' n'est pas reconnu`, pas de panique, nous allons l'installer.

### Comment ouvrir un terminal ?

**Windows** :
- Appuyez sur `Win + R`, tapez `cmd` et appuyez sur Entrée
- Ou cherchez "Invite de commandes" dans le menu Démarrer

**macOS** :
- Appuyez sur `Cmd + Espace`, tapez "Terminal" et appuyez sur Entrée
- Ou allez dans Applications > Utilitaires > Terminal

**Linux** :
- Appuyez sur `Ctrl + Alt + T`
- Ou cherchez "Terminal" dans vos applications

## Étape 2 : Installation de SQLite

### 🍎 Sur macOS

**Option 1 : SQLite est pré-installé !**
SQLite est inclus avec macOS, donc vous n'avez rien à faire. Testez avec `sqlite3 --version`.

**Option 2 : Installation via Homebrew (pour avoir la dernière version)**
```bash
# Installer Homebrew si pas déjà fait
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Installer SQLite
brew install sqlite3
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
- **CentOS/RHEL/Fedora** : `sudo yum install sqlite3` ou `sudo dnf install sqlite3`
- **Arch Linux** : `sudo pacman -S sqlite`

### 🪟 Sur Windows

**Option 1 : Téléchargement direct (Recommandé pour débutants)**

1. Allez sur [https://sqlite.org/download.html](https://sqlite.org/download.html)
2. Dans la section "Precompiled Binaries for Windows", téléchargez :
   - `sqlite-tools-win32-x86-XXXXXXX.zip` (remplacez XXXXXXX par le numéro de version)
3. Décompressez le fichier ZIP dans un dossier, par exemple `C:\sqlite`
4. **Important** : Ajoutez ce dossier au PATH de Windows

**Comment ajouter SQLite au PATH sur Windows :**

1. Ouvrez "Paramètres système avancés" (cherchez dans le menu Démarrer)
2. Cliquez sur "Variables d'environnement"
3. Dans "Variables système", trouvez et sélectionnez "Path"
4. Cliquez sur "Modifier"
5. Cliquez sur "Nouveau"
6. Ajoutez le chemin vers votre dossier SQLite (ex: `C:\sqlite`)
7. Cliquez sur "OK" partout
8. **Redémarrez votre invite de commandes**

**Option 2 : Via Windows Package Manager**
```bash
# Si vous avez winget installé
winget install sqlite.sqlite
```

**Option 3 : Via Chocolatey**
```bash
# Si vous avez Chocolatey installé
choco install sqlite
```

## Étape 3 : Test de l'installation

Une fois l'installation terminée, testez que tout fonctionne :

```bash
# Vérifier la version
sqlite3 --version

# Créer une base de données de test
sqlite3 test.db

# Vous devriez voir l'invite SQLite :
# sqlite>
```

Si vous voyez `sqlite>`, c'est parfait ! Vous êtes dans l'interface en ligne de commande de SQLite.

Pour sortir, tapez :
```sql
.quit
```

## Étape 4 : Interface en ligne de commande - Premier contact

Reprenons notre session SQLite :

```bash
sqlite3 test.db
```

Voici les commandes de base à connaître :

### Commandes système (commencent par un point)

```sql
-- Voir l'aide
.help

-- Voir les tables existantes
.tables

-- Voir la structure d'une table
.schema nom_table

-- Quitter SQLite
.quit
```

### Votre première requête SQL

```sql
-- Créer une table simple
CREATE TABLE personnes (
    nom TEXT,
    age INTEGER
);

-- Insérer des données
INSERT INTO personnes (nom, age) VALUES ('Alice', 25);
INSERT INTO personnes (nom, age) VALUES ('Bob', 30);

-- Voir les données
SELECT * FROM personnes;
```

## Étape 5 : Outils complémentaires recommandés

### Éditeurs de texte avec coloration SQL

**Gratuits et simples :**
- **Visual Studio Code** : Excellent support SQL, extensions disponibles
- **Notepad++** (Windows) : Léger avec coloration syntaxique
- **Sublime Text** : Rapide et élégant

**Installation de VS Code (recommandé) :**
1. Allez sur [https://code.visualstudio.com](https://code.visualstudio.com)
2. Téléchargez et installez
3. Installez l'extension "SQLite" pour une meilleure expérience

### Interfaces graphiques (optionnelles)

Pour visualiser vos données plus facilement :

**Gratuits :**
- **SQLite Browser** : Simple et efficace
  - Téléchargez sur [https://sqlitebrowser.org](https://sqlitebrowser.org)
- **DBeaver Community** : Plus avancé
  - Téléchargez sur [https://dbeaver.io](https://dbeaver.io)

**En ligne :**
- **SQLite Viewer** : [https://sqliteviewer.app](https://sqliteviewer.app)
- Parfait pour examiner rapidement un fichier .db

## Étape 6 : Configuration de l'environnement de travail

Créons un dossier pour notre formation :

```bash
# Créer un dossier de travail
mkdir formation-sqlite
cd formation-sqlite

# Créer des sous-dossiers
mkdir bases-donnees
mkdir scripts
mkdir exercices

# Structure obtenue :
# formation-sqlite/
# ├── bases-donnees/    (nos fichiers .db)
# ├── scripts/          (nos fichiers .sql)
# └── exercices/        (exercices pratiques)
```

## Vérification finale - Checklist

Vérifiez que tout fonctionne :

- [ ] `sqlite3 --version` affiche une version
- [ ] `sqlite3 test.db` ouvre l'interface SQLite
- [ ] `.help` dans SQLite affiche l'aide
- [ ] Vous pouvez créer une table et insérer des données
- [ ] Vous avez un éditeur de texte configuré
- [ ] Votre dossier de travail est créé

## Résolution de problèmes courants

### "sqlite3 n'est pas reconnu" (Windows)

**Solutions :**
1. Vérifiez que SQLite est dans votre PATH
2. Redémarrez votre invite de commandes
3. Utilisez le chemin complet : `C:\sqlite\sqlite3.exe`

### Permission refusée (Linux/macOS)

**Solutions :**
1. Vérifiez que vous avez les droits d'écriture dans le dossier
2. Utilisez `sudo` si nécessaire pour l'installation
3. Changez les permissions : `chmod +x sqlite3`

### Fichier de base corrompu

**Solutions :**
1. Supprimez le fichier de test : `rm test.db`
2. Recréez-le : `sqlite3 nouvelle_base.db`

## 🎯 Exercice pratique

Maintenant que tout est installé, créons notre première vraie base de données :

1. Créez une base nommée `bibliotheque.db`
2. Créez une table `livres` avec les colonnes : titre, auteur, annee
3. Insérez 3 livres de votre choix
4. Affichez tous les livres avec `SELECT * FROM livres;`
5. Quittez SQLite et vérifiez que le fichier `bibliotheque.db` existe

**Solution :**
```bash
sqlite3 bibliotheque.db
```

```sql
CREATE TABLE livres (
    titre TEXT,
    auteur TEXT,
    annee INTEGER
);

INSERT INTO livres VALUES ('1984', 'George Orwell', 1949);
INSERT INTO livres VALUES ('Le Petit Prince', 'Antoine de Saint-Exupéry', 1943);
INSERT INTO livres VALUES ('Harry Potter', 'J.K. Rowling', 1997);

SELECT * FROM livres;
.quit
```

## Récapitulatif

**Ce que nous avons accompli :**
- ✅ Installation de SQLite sur votre système
- ✅ Configuration de l'environnement de travail
- ✅ Premier contact avec l'interface en ligne de commande
- ✅ Installation d'outils complémentaires
- ✅ Création de votre première base de données

**À retenir :**
- SQLite fonctionne directement en ligne de commande
- Les commandes système commencent par un point (`.help`, `.quit`)
- Un fichier `.db` contient toute votre base de données
- Vous avez maintenant tous les outils pour commencer !

---

**💡 Dans le prochain chapitre**, nous verrons en détail les différences entre SQLite et les autres systèmes de bases de données comme MySQL et PostgreSQL.

⏭️
