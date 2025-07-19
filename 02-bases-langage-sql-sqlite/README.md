🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Module 2 : Bases du langage SQL avec SQLite

## Objectifs du module

À l'issue de ce module, vous serez capable de :

- Maîtriser les 5 types de données spécifiques à SQLite et leur utilisation pratique
- Créer et gérer des bases de données SQLite de manière autonome
- Effectuer toutes les opérations CRUD (Create, Read, Update, Delete) avec confiance
- Implémenter et comprendre les contraintes essentielles (PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK)
- Construire des requêtes efficaces avec SELECT, WHERE, ORDER BY, GROUP BY et HAVING
- Appliquer les bonnes pratiques SQL dans le contexte spécifique de SQLite

## Prérequis

- Avoir terminé le **Module 1 : Fondamentaux de SQLite3**
- SQLite installé et fonctionnel sur votre système
- Compréhension des concepts de base (tables, lignes, colonnes)
- Familiarité avec l'interface en ligne de commande SQLite

## Plan du module

Ce module vous accompagne dans la découverte progressive du langage SQL adapté à SQLite :

### 2.1 Types de données SQLite (TEXT, INTEGER, REAL, BLOB, NULL)
Exploration approfondie du système de types unique de SQLite, ses avantages et ses pièges à éviter.

### 2.2 Création et gestion des bases de données
Maîtrise complète de la création, modification et organisation des bases de données SQLite.

### 2.3 Opérations CRUD : CREATE, READ, UPDATE, DELETE
Les quatre opérations fondamentales expliquées avec de nombreux exemples pratiques.

### 2.4 Contraintes : PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK
Garantir l'intégrité et la cohérence de vos données avec les contraintes appropriées.

### 2.5 Requêtes de base : SELECT, WHERE, ORDER BY, GROUP BY, HAVING
Construction de requêtes efficaces pour extraire exactement les informations dont vous avez besoin.

## Pourquoi ce module est crucial ?

### 🎯 Fondations solides

Ce module établit les bases indispensables pour tout travail sérieux avec SQLite. Contrairement à d'autres SGBD, SQLite a ses propres spécificités qu'il faut maîtriser :

- **Système de types unique** : SQLite gère les types différemment des autres bases de données
- **Simplicité trompeuse** : Certaines opérations semblent simples mais cachent des subtilités
- **Optimisations spécifiques** : Les bonnes pratiques SQLite diffèrent parfois des standards SQL

### 🔧 Approche pratique avant tout

Plutôt que d'apprendre la théorie SQL générale, ce module se concentre sur :

- **SQLite en pratique** : Chaque concept est illustré avec des exemples SQLite réels
- **Erreurs courantes** : Anticipation des pièges fréquents rencontrés par les débutants
- **Bonnes pratiques** : Dès le départ, développement des bons réflexes
- **Cas concrets** : Utilisation d'exemples tirés de situations réelles

### 🚀 Progression pédagogique

Le module suit une progression logique et naturelle :

1. **Types de données** → Comprendre comment SQLite stocke l'information
2. **Bases de données** → Savoir organiser et structurer ses données
3. **CRUD** → Maîtriser les opérations essentielles du quotidien
4. **Contraintes** → Garantir la qualité et l'intégrité des données
5. **Requêtes** → Extraire efficacement l'information recherchée

## Ce qui rend SQLite différent

### 💡 Types dynamiques vs types statiques

**Autres SGBD (MySQL, PostgreSQL) :**
```sql
-- Types stricts et fixes
CREATE TABLE users (
    id INT PRIMARY KEY,           -- Entier obligatoire
    name VARCHAR(50) NOT NULL,    -- Chaîne de 50 caractères max
    age TINYINT UNSIGNED         -- Entier 0-255
);
```

**SQLite :**
```sql
-- Types flexibles et adaptatifs
CREATE TABLE users (
    id INTEGER PRIMARY KEY,       -- Suggère entier, mais flexible
    name TEXT,                   -- Texte de longueur variable
    age INTEGER                  -- Entier, mais accepte autre chose
);
```

### 🔄 Conversion automatique

SQLite effectue des conversions automatiques intelligentes :

```sql
INSERT INTO users VALUES (1, 'Alice', '25');     -- '25' devient 25
INSERT INTO users VALUES ('2', 'Bob', 30);       -- '2' devient 2
```

Cette flexibilité est un atout... mais peut aussi créer des surprises !

### 📁 Simplicité de gestion

Contrairement aux autres SGBD, avec SQLite :

- **Pas d'utilisateurs** à gérer (un fichier = une base)
- **Pas de permissions** complexes
- **Pas de configuration** réseau
- **Modifications directes** possibles sur le fichier

## Méthodologie d'apprentissage

### 🎯 Apprentissage par la pratique

Chaque section suit le même pattern :

1. **Concept théorique** : Explication claire et concise
2. **Exemples simples** : Démonstrations pas à pas
3. **Cas pratiques** : Applications concrètes
4. **Exercices guidés** : Mise en pratique immédiate
5. **Pièges à éviter** : Erreurs courantes et solutions

### 🔧 Environnement de travail

Pour ce module, vous travaillerez avec :

```bash
# Structure recommandée
module2-sql/
├── exercices/          # Vos exercices pratiques
├── exemples/           # Bases d'exemple fournies
└── projets/            # Mini-projets du module
```

### 📊 Projet fil rouge

Tout au long du module, nous construirons ensemble une **base de données de bibliothèque** qui évoluera avec chaque chapitre :

- **2.1** → Définition des types de données
- **2.2** → Création de la structure de base
- **2.3** → Ajout, modification, suppression des données
- **2.4** → Ajout des contraintes d'intégrité
- **2.5** → Requêtes d'analyse et de reporting

## Outils et ressources

### 🛠️ Outils recommandés

- **Ligne de commande SQLite** : Pour les manipulations de base
- **Éditeur de texte** : Pour écrire vos scripts SQL
- **DB Browser for SQLite** (optionnel) : Pour visualiser vos données

### 📚 Ressources du module

- **Aide-mémoires** : Synthèses des commandes importantes
- **Scripts d'exemple** : Code réutilisable pour vos projets
- **Jeux de données** : Données réalistes pour pratiquer
- **Solutions commentées** : Explications détaillées des exercices

## À quoi vous attendre

### ✅ Ce que vous maîtriserez

À la fin de ce module, vous serez capable de :

- Créer des bases de données SQLite robustes et bien structurées
- Manipuler efficacement les données avec toutes les opérations CRUD
- Utiliser les contraintes pour garantir la qualité des données
- Construire des requêtes SQL précises et performantes
- Éviter les erreurs courantes spécifiques à SQLite
- Adopter les bonnes pratiques dès le départ

### 🎯 Niveau de complexité

Ce module reste **accessible aux débutants** tout en étant **complet** :

- **Exemples simples** pour comprendre les concepts
- **Exercices progressifs** pour monter en compétence
- **Explications détaillées** des spécificités SQLite
- **Références** aux standards SQL quand pertinent

## Message d'encouragement

### 🚀 Vous êtes prêt !

Si vous avez terminé le Module 1, vous avez déjà franchi l'étape la plus importante : comprendre la philosophie unique de SQLite. Maintenant, nous allons passer à la pratique !

### 💪 Pas à pas

N'hésitez pas à :
- **Prendre votre temps** sur chaque section
- **Expérimenter** au-delà des exemples proposés
- **Poser des questions** (même à vous-même !)
- **Revenir en arrière** si nécessaire

### 🎯 Objectif réaliste

À la fin de ce module, vous ne serez pas expert SQL, mais vous aurez des **bases solides** pour :
- Créer vos propres applications avec SQLite
- Comprendre et modifier du code SQL existant
- Progresser vers les modules plus avancés
- Résoudre la plupart des problèmes quotidiens

## Récapitulatif

**Ce module vous donnera :**
- ✅ Maîtrise des types de données SQLite
- ✅ Compétences complètes en CRUD
- ✅ Utilisation appropriée des contraintes
- ✅ Construction de requêtes efficaces
- ✅ Bonnes pratiques spécifiques à SQLite

**Avec une approche :**
- 🎯 Pratique et concrète
- 📚 Progressive et pédagogique
- 🔧 Centrée sur SQLite
- 💡 Riche en exemples réels

---

**🎯 Prêt à plonger dans le SQL SQLite ?** Le prochain chapitre explore le système de types unique de SQLite, fondement de tout le reste !

**💡 Conseil pour commencer** : Préparez un dossier de travail et ouvrez votre terminal. Nous allons beaucoup pratiquer !

⏭️
