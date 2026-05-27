🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 4 : Requêtes avancées et optimisation

## Introduction

Après avoir maîtrisé les fondamentaux de SQLite et les techniques de conception de base de données, nous entrons maintenant dans le domaine des requêtes avancées et de l'optimisation. Ce chapitre représente un tournant crucial dans votre apprentissage de SQLite, vous permettant de passer d'un utilisateur intermédiaire à un développeur capable de résoudre des problèmes complexes avec élégance et efficacité.

## Pourquoi les requêtes avancées sont-elles essentielles ?

Dans le monde réel, les besoins en matière de données dépassent rapidement les simples opérations CRUD. Vous serez confronté à des défis comme :

- **Analyser des tendances temporelles** dans vos données de vente
- **Calculer des agrégations complexes** avec des conditions spécifiques
- **Traiter des données hiérarchiques** comme des structures organisationnelles
- **Manipuler des données JSON** stockées dans vos tables
- **Optimiser des requêtes** qui traitent des millions d'enregistrements

## Vue d'ensemble du chapitre

Ce chapitre couvre six domaines clés qui transformeront votre approche de SQLite :

| # | Section | Sujet principal | Lien |
|--:|---------|-----------------|------|
| 4.1 | **Sous-requêtes** | corrélées vs non-corrélées, `EXISTS`, `IN`, équivalents SQLite de `ANY/ALL` | [→ 4.1](/04-requetes-avancees-optimisation/01-sous-requetes-correlees-non-correlees.md) |
| 4.2 | **CTE & récursion** | `WITH`, `WITH RECURSIVE`, hiérarchies | [→ 4.2](/04-requetes-avancees-optimisation/02-expressions-table-communes-cte-requetes-recursives.md) |
| 4.3 | **Window functions** | `OVER`, `PARTITION BY`, `ROW_NUMBER`, `LAG`, `LEAD` | [→ 4.3](/04-requetes-avancees-optimisation/03-fonctions-fenetrage-window-functions.md) |
| 4.4 | **REGEXP** | pattern matching, validation de données | [→ 4.4](/04-requetes-avancees-optimisation/04-expressions-regulieres-regexp.md) |
| 4.5 | **CASE / COALESCE / NULLIF** | logique conditionnelle, gestion des `NULL` | [→ 4.5](/04-requetes-avancees-optimisation/05-requetes-complexes-case-coalesce-nullif.md) |
| 4.6 | **JSON** | `json_extract`, `->`, `->>`, JSONB | [→ 4.6](/04-requetes-avancees-optimisation/06-interrogation-manipulation-donnees-json.md) |

### 🎯 **Sous-requêtes et corrélations**
Apprenez à décomposer des problèmes complexes en requêtes imbriquées intelligentes. Nous explorerons la différence entre sous-requêtes corrélées et non-corrélées, leurs cas d'usage optimaux, et comment éviter les pièges de performance.

### 🔄 **Expressions de table communes (CTE)**
Découvrez la puissance des CTE pour structurer vos requêtes complexes et gérer des données hiérarchiques avec les requêtes récursives. Cette technique révolutionnera votre façon d'aborder les problèmes de données complexes.

### 📊 **Fonctions de fenêtrage**
Maîtrisez les fonctions WINDOW pour effectuer des calculs sophistiqués comme les moyennes mobiles, les classements, et les comparaisons par rapport aux valeurs précédentes ou suivantes.

### 🔍 **Expressions régulières**
Exploitez la puissance du pattern matching avec REGEXP pour des recherches et validations de données avancées.

### 🧮 **Logique conditionnelle avancée**
Perfectionnez votre utilisation de CASE, COALESCE, et NULLIF pour créer des requêtes intelligentes qui s'adaptent aux données.

### 📋 **Manipulation de données JSON**
Explorez les capacités JSON native de SQLite pour stocker et interroger des données semi-structurées efficacement.

## Prérequis et préparation

Avant de commencer ce chapitre, assurez-vous de maîtriser :

- Les jointures de base : `INNER`, `LEFT` (`RIGHT` et `FULL OUTER` ne sont disponibles que **depuis SQLite 3.39**)
- Les fonctions d'agrégation (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`)
- Les clauses `GROUP BY` et `HAVING`
- La création et modification de tables

## Approche pédagogique

Chaque section de ce chapitre suivra une approche structurée :

1. **Concept théorique** : Explication claire du principe
2. **Syntaxe détaillée** : Description complète de la syntaxe avec tous les paramètres
3. **Exemples pratiques** : Cas d'usage concrets avec des données réalistes
4. **Bonnes pratiques** : Conseils d'optimisation et erreurs à éviter
5. **Exercices** : Applications pratiques pour renforcer l'apprentissage

## Base de données d'exemple

Tout au long de ce chapitre, nous utiliserons une base de données d'exemple représentant un système de gestion d'une **librairie en ligne**. Les tables sont nommées **en français** dans tous les chapitres pour rester cohérent avec le reste de la formation :

```text
Tables principales du fil rouge :
- clients      (clients de la librairie)
- livres       (catalogue de livres)
- auteurs      (auteurs des livres)
- categories   (catégories de livres, hiérarchiques)
- commandes    (commandes passées par les clients)
```

Ces tables sont créées et peuplées dans le chapitre **4.2 (CTE)**, qui sert de base commune aux chapitres **4.3 → 4.5**. Le chapitre **4.6 (JSON)** utilise des tables dédiées avec suffixe `_json` (autonomes).

Cette base de données nous permettra d'explorer des scénarios réalistes comme l'analyse des ventes, le calcul de recommandations, ou l'évaluation de la performance des auteurs.

## Objectifs d'apprentissage

À la fin de ce chapitre, vous serez capable de :

- ✅ Concevoir des requêtes complexes utilisant des sous-requêtes optimisées
- ✅ Utiliser les CTE pour résoudre des problèmes de données hiérarchiques
- ✅ Implémenter des analyses sophistiquées avec les fonctions de fenêtrage
- ✅ Manipuler des données textuelles avec les expressions régulières
- ✅ Gérer des données JSON natives dans SQLite
- ✅ Optimiser les performances de vos requêtes complexes
- ✅ Appliquer la logique conditionnelle avancée dans vos requêtes

## Conseil pour réussir

Les requêtes avancées peuvent sembler intimidantes au premier abord. La clé du succès réside dans :

- **La pratique régulière** : Testez chaque exemple sur votre propre installation SQLite
- **La décomposition** : Divisez les problèmes complexes en étapes simples
- **L'expérimentation** : N'hésitez pas à modifier les exemples pour explorer différentes approches
- **La patience** : Ces concepts demandent du temps pour être parfaitement assimilés

Préparez-vous à découvrir la vraie puissance de SQLite et à développer des compétences qui feront la différence dans vos projets professionnels !

---

**Prochaine section** : [4.1 Sous-requêtes corrélées et non-corrélées](/04-requetes-avancees-optimisation/01-sous-requetes-correlees-non-correlees.md)

⏭️ [4.1 Sous-requêtes corrélées et non-corrélées](/04-requetes-avancees-optimisation/01-sous-requetes-correlees-non-correlees.md)
