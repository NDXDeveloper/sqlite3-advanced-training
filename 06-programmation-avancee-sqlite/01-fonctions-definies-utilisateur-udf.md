🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.1 Fonctions définies par l'utilisateur (UDF)

## Qu'est-ce qu'une fonction définie par l'utilisateur ?

Une **fonction définie par l'utilisateur** (UDF - User Defined Function) est une fonction personnalisée que vous créez et qui s'intègre directement dans SQLite. Une fois créée, elle peut être utilisée dans vos requêtes SQL comme n'importe quelle fonction native (LENGTH, UPPER, SUM, etc.).

### Pourquoi créer ses propres fonctions ?

Imaginez que vous gérez une base de données de clients et que vous devez régulièrement calculer la distance entre deux adresses. Au lieu de faire ce calcul dans votre code Python ou JavaScript à chaque fois, vous pourriez créer une fonction `calculer_distance()` directement dans SQLite :

```sql
-- Au lieu de récupérer les données et calculer dans votre code
SELECT nom, latitude, longitude FROM clients;

-- Vous pourriez faire directement :
SELECT nom, calculer_distance(latitude, longitude, 48.8566, 2.3522) as distance_paris
FROM clients
WHERE distance_paris < 50;
```

## Types de fonctions UDF

SQLite supporte trois types de fonctions personnalisées :

### 1. Fonctions scalaires
Prennent un ou plusieurs arguments et retournent une seule valeur.

**Exemples d'usage :**
- Calculer une distance géographique
- Formater un numéro de téléphone
- Appliquer une formule métier complexe

### 2. Fonctions d'agrégation
Traitent un ensemble de lignes et retournent un résultat unique (comme SUM ou AVG).

**Exemples d'usage :**
- Calculer une médiane
- Concaténer des valeurs avec un séparateur personnalisé
- Effectuer des calculs statistiques avancés

### 3. Fonctions de fenêtrage
Opèrent sur un ensemble de lignes liées à la ligne courante (utilisées avec OVER).

## Créer des UDF avec Python

Python est l'un des langages les plus accessibles pour débuter avec les UDF SQLite. Voici comment procéder :

### Installation et configuration

```python
import sqlite3
import math

# Connexion à la base de données
conn = sqlite3.connect('ma_base.db')
```

### Première fonction simple

Créons une fonction qui convertit une température de Celsius vers Fahrenheit :

```python
def celsius_vers_fahrenheit(celsius):
    """Convertit des degrés Celsius en Fahrenheit"""
    if celsius is None:
        return None
    return (celsius * 9/5) + 32

# Enregistrement de la fonction dans SQLite
conn.create_function("c_vers_f", 1, celsius_vers_fahrenheit)
```

**Utilisation :**
```sql
-- Dans une requête SQL
SELECT ville, temperature_c, c_vers_f(temperature_c) as temperature_f
FROM meteo;
```

### Fonction avec plusieurs paramètres

Calculons la distance entre deux points géographiques :

```python
def distance_gps(lat1, lon1, lat2, lon2):
    """
    Calcule la distance en km entre deux points GPS
    Utilise la formule de Haversine
    """
    # Vérification des valeurs nulles
    if any(v is None for v in [lat1, lon1, lat2, lon2]):
        return None

    # Conversion en radians
    lat1, lon1, lat2, lon2 = map(math.radians, [lat1, lon1, lat2, lon2])

    # Formule de Haversine
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    a = math.sin(dlat/2)**2 + math.cos(lat1) * math.cos(lat2) * math.sin(dlon/2)**2
    c = 2 * math.asin(math.sqrt(a))

    # Rayon de la Terre en kilomètres
    r = 6371
    return c * r

# Enregistrement : nom, nombre de paramètres, fonction
conn.create_function("distance_gps", 4, distance_gps)
```

**Utilisation pratique :**
```sql
-- Trouver tous les restaurants dans un rayon de 5km
SELECT nom, adresse,
       distance_gps(restaurant_lat, restaurant_lon, 48.8566, 2.3522) as distance
FROM restaurants
WHERE distance < 5
ORDER BY distance;
```

### Fonction de validation

Créons une fonction qui valide un numéro de téléphone français :

```python
import re

def valider_telephone_fr(numero):
    """Valide un numéro de téléphone français"""
    if numero is None:
        return False

    # Nettoie le numéro (supprime espaces, tirets, points)
    numero_clean = re.sub(r'[\s\-\.]', '', str(numero))

    # Vérifie le format français (10 chiffres commençant par 0)
    pattern = r'^0[1-9][0-9]{8}$'
    return bool(re.match(pattern, numero_clean))

conn.create_function("tel_valide", 1, valider_telephone_fr)
```

**Utilisation :**
```sql
-- Identifier les numéros de téléphone invalides
SELECT nom, telephone
FROM clients
WHERE NOT tel_valide(telephone);

-- Compter les clients avec des numéros valides
SELECT COUNT(*) as clients_tel_valide
FROM clients
WHERE tel_valide(telephone);
```

## Fonctions d'agrégation personnalisées

Les fonctions d'agrégation sont plus complexes car elles doivent traiter un ensemble de valeurs. Voici comment créer une fonction qui calcule la médiane :

```python
class MedianeAgregate:
    def __init__(self):
        self.values = []

    def step(self, value):
        """Appelée pour chaque ligne"""
        if value is not None:
            self.values.append(value)

    def finalize(self):
        """Appelée à la fin pour calculer le résultat"""
        if not self.values:
            return None

        self.values.sort()
        n = len(self.values)

        if n % 2 == 0:
            # Nombre pair : moyenne des deux valeurs centrales
            return (self.values[n//2 - 1] + self.values[n//2]) / 2
        else:
            # Nombre impair : valeur centrale
            return self.values[n//2]

# Enregistrement de la fonction d'agrégation
conn.create_aggregate("mediane", 1, MedianeAgregate)
```

**Utilisation :**
```sql
-- Calculer la médiane des salaires par département
SELECT departement,
       AVG(salaire) as salaire_moyen,
       mediane(salaire) as salaire_median
FROM employes
GROUP BY departement;
```

## Gestion des erreurs dans les UDF

Il est important de gérer les erreurs proprement dans vos fonctions :

```python
def diviser_securise(a, b):
    """Division sécurisée qui gère la division par zéro"""
    try:
        if a is None or b is None:
            return None
        if b == 0:
            return None  # ou lever une exception selon le besoin
        return float(a) / float(b)
    except (TypeError, ValueError):
        return None  # Gestion des types incompatibles

conn.create_function("div_secure", 2, diviser_securise)
```

## Fonctions avec logique métier complexe

Voici un exemple plus avancé qui calcule un score de fidélité client :

```python
from datetime import datetime, timedelta

def score_fidelite(nb_commandes, montant_total, date_premiere_commande, date_derniere_commande):
    """
    Calcule un score de fidélité basé sur :
    - Nombre de commandes
    - Montant total dépensé
    - Ancienneté du client
    - Récence de la dernière commande
    """
    if any(v is None for v in [nb_commandes, montant_total, date_premiere_commande]):
        return 0

    try:
        # Conversion des dates
        if isinstance(date_premiere_commande, str):
            premiere = datetime.strptime(date_premiere_commande, '%Y-%m-%d')
        else:
            premiere = date_premiere_commande

        if date_derniere_commande and isinstance(date_derniere_commande, str):
            derniere = datetime.strptime(date_derniere_commande, '%Y-%m-%d')
        else:
            derniere = datetime.now()

        # Calculs
        anciennete_jours = (datetime.now() - premiere).days
        recence_jours = (datetime.now() - derniere).days

        # Score basé sur différents critères
        score = 0

        # Points pour le nombre de commandes (max 40 points)
        score += min(nb_commandes * 2, 40)

        # Points pour le montant (1 point par tranche de 50€, max 30 points)
        score += min(montant_total // 50, 30)

        # Points pour l'ancienneté (1 point par mois, max 20 points)
        score += min(anciennete_jours // 30, 20)

        # Malus pour l'inactivité
        if recence_jours > 365:
            score *= 0.5  # Réduction de 50% si pas d'achat depuis 1 an
        elif recence_jours > 180:
            score *= 0.8  # Réduction de 20% si pas d'achat depuis 6 mois

        return round(score, 2)

    except (ValueError, TypeError):
        return 0

conn.create_function("score_fidelite", 4, score_fidelite)
```

**Utilisation :**
```sql
-- Classer les clients par score de fidélité
SELECT
    nom,
    email,
    score_fidelite(
        nb_commandes,
        montant_total,
        date_premiere_commande,
        date_derniere_commande
    ) as score
FROM clients
ORDER BY score DESC
LIMIT 10;

-- Identifier les clients à risque (score faible)
SELECT COUNT(*) as clients_a_risque
FROM clients
WHERE score_fidelite(nb_commandes, montant_total, date_premiere_commande, date_derniere_commande) < 10;
```

## Optimisation et bonnes pratiques

### Performance
```python
# ✅ Bon : vérification rapide des cas simples
def ma_fonction(value):
    if value is None:
        return None
    if value == 0:
        return 0
    # Traitement complexe seulement si nécessaire
    return traitement_complexe(value)

# ❌ Éviter : calculs inutiles
def ma_fonction_lente(value):
    result = traitement_complexe(value)  # Même si value est None !
    if value is None:
        return None
    return result
```

### Gestion de la mémoire
```python
# ✅ Pour les fonctions d'agrégation, limiter la mémoire
class MedianeOptimisee:
    def __init__(self):
        self.values = []
        self.max_values = 10000  # Limite pour éviter les problèmes mémoire

    def step(self, value):
        if value is not None and len(self.values) < self.max_values:
            self.values.append(value)
```

### Documentation et tests
```python
def ma_fonction(param1, param2):
    """
    Description claire de ce que fait la fonction

    Args:
        param1: Type et description
        param2: Type et description

    Returns:
        Type et description du résultat

    Examples:
        ma_fonction(10, 5) -> 15
        ma_fonction(None, 5) -> None
    """
    # Implémentation...
```

## Exemple complet : système de notation

Voici un exemple complet qui combine plusieurs concepts :

```python
import sqlite3
import math
import re
from datetime import datetime

# Connexion à la base
conn = sqlite3.connect('exemple.db')

# 1. Fonction de validation d'email
def valider_email(email):
    if email is None:
        return False
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

# 2. Fonction de calcul d'âge
def calculer_age(date_naissance):
    if date_naissance is None:
        return None
    try:
        if isinstance(date_naissance, str):
            naissance = datetime.strptime(date_naissance, '%Y-%m-%d')
        else:
            naissance = date_naissance

        aujourd_hui = datetime.now()
        age = aujourd_hui.year - naissance.year

        # Ajustement si l'anniversaire n'a pas encore eu lieu cette année
        if (aujourd_hui.month, aujourd_hui.day) < (naissance.month, naissance.day):
            age -= 1

        return age
    except:
        return None

# 3. Fonction de catégorisation par âge
def categorie_age(age):
    if age is None:
        return "Inconnu"
    if age < 18:
        return "Mineur"
    elif age < 25:
        return "Jeune adulte"
    elif age < 65:
        return "Adulte"
    else:
        return "Senior"

# Enregistrement des fonctions
conn.create_function("valider_email", 1, valider_email)
conn.create_function("calculer_age", 1, calculer_age)
conn.create_function("categorie_age", 1, categorie_age)

# Création d'une table d'exemple
conn.execute('''
CREATE TABLE IF NOT EXISTS utilisateurs (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    email TEXT,
    date_naissance TEXT
)
''')

# Insertion de données de test
utilisateurs_test = [
    ("Alice Dupont", "alice@email.com", "1985-03-15"),
    ("Bob Martin", "bob.email.invalide", "1990-07-22"),
    ("Claire Durand", "claire@test.fr", "2005-12-10"),
    ("David Rousseau", "david@work.org", "1955-08-30"),
]

conn.executemany(
    "INSERT OR REPLACE INTO utilisateurs (nom, email, date_naissance) VALUES (?, ?, ?)",
    utilisateurs_test
)

# Utilisation des fonctions dans une requête complète
resultat = conn.execute('''
SELECT
    nom,
    email,
    CASE
        WHEN valider_email(email) THEN "✓ Email valide"
        ELSE "✗ Email invalide"
    END as statut_email,
    date_naissance,
    calculer_age(date_naissance) as age,
    categorie_age(calculer_age(date_naissance)) as categorie
FROM utilisateurs
ORDER BY calculer_age(date_naissance)
''')

print("Rapport des utilisateurs :")
print("-" * 80)
for row in resultat:
    print(f"{row[0]:15} | {row[1]:20} | {row[2]:15} | Age: {row[4]:2} | {row[5]}")

conn.close()
```

## Récapitulatif et points clés

### Ce que vous avez appris
- ✅ **Concept des UDF** : Fonctions personnalisées intégrées à SQLite
- ✅ **Types de fonctions** : Scalaires, d'agrégation, de fenêtrage
- ✅ **Implémentation Python** : Création et enregistrement de fonctions
- ✅ **Gestion d'erreurs** : Validation des entrées et cas limites
- ✅ **Bonnes pratiques** : Performance, mémoire, documentation

### Avantages des UDF
- **Réutilisabilité** : Une fois créée, utilisable dans toutes vos requêtes
- **Performance** : Traitement au niveau de la base de données
- **Maintenabilité** : Logique métier centralisée
- **Simplicité** : Requêtes SQL plus lisibles et expressives

### Quand utiliser les UDF
- ✅ Calculs complexes répétitifs
- ✅ Validation de données
- ✅ Transformations métier spécifiques
- ✅ Fonctions d'agrégation personnalisées

### Quand éviter les UDF
- ❌ Opérations très lourdes (préférer le traitement applicatif)
- ❌ Fonctions trop spécialisées utilisées rarement
- ❌ Logique qui change fréquemment

### Prochaines étapes
Dans la section suivante (6.2), nous verrons comment créer des **extensions SQLite** plus avancées, qui permettent d'ajouter non seulement des fonctions, mais aussi de nouveaux types de données et des fonctionnalités système complètes.

⏭️
