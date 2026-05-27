🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.2 : Analyse de données avec SQLite

## Introduction : SQLite comme outil d'analyse

Contrairement aux idées reçues, SQLite n'est pas seulement une base de données d'application. C'est un **excellent outil d'analyse de données**, viable jusqu'à plusieurs centaines de GB en lecture mono-process — voir le tableau « Comparaison avec d'autres outils » ci-dessous pour les critères de bascule.

### Pourquoi choisir SQLite pour l'analyse ?

- **Simplicité** : Pas de serveur à installer ou configurer
- **Performance** : Très rapide pour les requêtes analytiques
- **Intégration** : Fonctionne parfaitement avec Python, R, Excel
- **Portabilité** : Un seul fichier contient toutes vos données
- **SQL standard** : Utilisez vos compétences SQL existantes
- **Gratuit** : Aucun coût de licence, même pour usage commercial

### Comparaison avec d'autres outils

Le bon outil dépend moins de la **taille brute** que du **profil d'usage** (lecture seule vs concurrence d'écriture, agrégations vs OLTP). Voici un guide pragmatique :

| Profil | Recommandation |
|---|---|
| **Analyse exploratoire single-user** (< 100 GB, lecture-écriture mono-process) | **SQLite** — parfaitement adapté, jusqu'à plusieurs centaines de GB |
| **Agrégations massives** (window functions, GROUP BY sur millions de lignes) | **DuckDB** — moteur colonne, souvent 10-100× plus rapide |
| **Multi-utilisateurs concurrents avec écritures fréquentes** | **PostgreSQL** — MVCC complet, plusieurs writers |
| **Datasets > 1 TB, requêtes distribuées** | **ClickHouse**, BigQuery, Snowflake, Spark |
| **Workflow data science Python** | **Polars** sur SQLite source, ou DuckDB en mémoire |

> 💡 La taille seule n'est pas un critère bloquant pour SQLite (limite théorique : 281 TB). Le vrai critère est : **avez-vous plusieurs writers concurrents** ? Si oui → PostgreSQL ou Litestream + replicas R/O. Sinon → SQLite reste excellent même à grande échelle.

## Préparation de l'environnement d'analyse

### Installation des outils Python nécessaires

```bash
# Installation des bibliothèques essentielles
# ⚠️ NE PAS inclure 'sqlite3' dans pip install :
#    sqlite3 est dans la bibliothèque standard Python (déjà installé).
#    Le faire installerait un package PyPI tiers homonyme, NON compatible.
pip install pandas matplotlib seaborn plotly chardet
```

> 💡 **Alternative moderne 2024+ pour l'analyse** : [**DuckDB**](https://duckdb.org/) est un moteur OLAP embarqué (comme SQLite mais optimisé colonne par colonne pour l'analytique). Compatible avec les fichiers SQLite via `INSTALL sqlite; LOAD sqlite; ATTACH 'fichier.db' AS sql_db (TYPE sqlite);` — souvent **10-100× plus rapide** que SQLite pur sur de grosses agrégations. SQLite reste préférable pour les charges OLTP (lecture-écriture transactionnelle de petites lignes).  
>  
> [**Polars**](https://pola.rs/) (DataFrame Rust + Python) lit aussi du SQLite directement et surclasse pandas en performance sur les agrégations. Combinaison gagnante 2026 : SQLite pour le stockage transactionnel + DuckDB ou Polars pour l'analytique lourde.

### Structure de projet recommandée

```
analyse_projet/
├── data/
│   ├── raw/              # Données brutes (CSV, Excel, etc.)
│   ├── processed/        # Données nettoyées
│   └── analysis.db       # Base SQLite d'analyse
├── scripts/
│   ├── import_data.py    # Scripts d'import
│   ├── clean_data.py     # Nettoyage des données
│   └── analysis.py       # Analyses principales
├── notebooks/
│   └── exploration.ipynb # Jupyter notebooks
└── exports/
    └── reports/          # Rapports générés
```

## Étape 1 : Import et préparation des données

### Exemple pratique : Analyse des ventes d'un magasin

Imaginons que nous avons des données de ventes en CSV :

```csv
date,produit,categorie,quantite,prix_unitaire,vendeur
2024-01-15,Laptop Dell,Informatique,2,899.99,Marie
2024-01-15,Souris Logitech,Informatique,5,29.99,Pierre
2024-01-16,Chaise Bureau,Mobilier,1,299.99,Marie
```

### Script d'import automatisé

```python
import sqlite3  
import pandas as pd  
from datetime import datetime  

class AnalyseurVentes:
    def __init__(self, db_path="data/analysis.db"):
        self.db_path = db_path
        self.init_database()

    def init_database(self):
        """Initialise la base d'analyse"""
        with sqlite3.connect(self.db_path) as conn:
            # Table principale des ventes
            conn.execute('''
                CREATE TABLE IF NOT EXISTS ventes (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    date TEXT NOT NULL,
                    produit TEXT NOT NULL,
                    categorie TEXT NOT NULL,
                    quantite INTEGER NOT NULL,
                    prix_unitaire REAL NOT NULL,
                    vendeur TEXT NOT NULL,
                    chiffre_affaires REAL GENERATED ALWAYS AS (quantite * prix_unitaire)
                )
            ''')

            # Index pour optimiser les analyses
            conn.execute('CREATE INDEX IF NOT EXISTS idx_ventes_date ON ventes(date)')
            conn.execute('CREATE INDEX IF NOT EXISTS idx_ventes_categorie ON ventes(categorie)')
            conn.execute('CREATE INDEX IF NOT EXISTS idx_ventes_vendeur ON ventes(vendeur)')

    def importer_csv(self, chemin_csv):
        """Importe des données depuis un fichier CSV"""
        try:
            # Lire le CSV avec pandas
            df = pd.read_csv(chemin_csv)

            # Validation basique des données
            colonnes_requises = ['date', 'produit', 'categorie', 'quantite', 'prix_unitaire', 'vendeur']
            for col in colonnes_requises:
                if col not in df.columns:
                    raise ValueError(f"Colonne manquante : {col}")

            # Nettoyage des données
            df = self.nettoyer_donnees(df)

            # ⚠️ La colonne `chiffre_affaires` est GENERATED dans la table.
            #    On ne peut PAS l'insérer (sqlite3.OperationalError: cannot
            #    INSERT into generated column). Si le CSV contient cette
            #    colonne, on doit la retirer avant `to_sql`.
            df = df.drop(columns=['chiffre_affaires'], errors='ignore')

            # Import dans SQLite
            with sqlite3.connect(self.db_path) as conn:
                df.to_sql('ventes', conn, if_exists='append', index=False)

            print(f"✅ {len(df)} lignes importées avec succès")
            return len(df)

        except Exception as e:
            print(f"❌ Erreur lors de l'import : {e}")
            return 0

    def nettoyer_donnees(self, df):
        """Nettoie et valide les données.

        Stratégie : on retourne un DataFrame "propre" SANS modifier l'original
        (df.copy()), et on filtre les lignes invalides au lieu de planter
        l'import complet sur la première ligne fautive.
        """
        df = df.copy()

        # Supprimer les lignes avec des valeurs manquantes
        df = df.dropna()

        # Convertir les dates — `errors='coerce'` met les dates invalides à NaT
        # (au lieu de planter), puis on filtre. Plus robuste qu'un crash brutal.
        df['date'] = pd.to_datetime(df['date'], errors='coerce')
        nb_dates_invalides = df['date'].isna().sum()
        if nb_dates_invalides > 0:
            print(f"⚠️ {nb_dates_invalides} lignes avec date invalide ignorées")
            df = df.dropna(subset=['date'])
        df['date'] = df['date'].dt.strftime('%Y-%m-%d')

        # Valider que quantité et prix sont positifs (avec coercion)
        df['quantite'] = pd.to_numeric(df['quantite'], errors='coerce')
        df['prix_unitaire'] = pd.to_numeric(df['prix_unitaire'], errors='coerce')
        df = df.dropna(subset=['quantite', 'prix_unitaire'])
        df = df[(df['quantite'] > 0) & (df['prix_unitaire'] > 0)]

        # Nettoyer les chaînes de caractères
        for col in ('produit', 'categorie', 'vendeur'):
            df[col] = df[col].astype(str).str.strip()

        return df

# Utilisation
analyseur = AnalyseurVentes()  
analyseur.importer_csv("data/raw/ventes_2024.csv")  
```

## Étape 2 : Analyses descriptives simples

### Statistiques de base

> 📝 **Convention dans ce chapitre** : les méthodes ci-dessous appartiennent à la classe `AnalyseurVentes` définie plus haut. Ajoutez-les à la classe (pas au niveau module) en respectant l'indentation `    def ...`. Les blocs sont séparés pour la lisibilité pédagogique.

```python
# Méthode à ajouter dans la classe AnalyseurVentes
def analyses_de_base(self):
    """Génère les statistiques descriptives de base"""
    with sqlite3.connect(self.db_path) as conn:

        # 1. Vue d'ensemble
        print("=== VUE D'ENSEMBLE ===")
        cursor = conn.execute('''
            SELECT
                COUNT(*) as nombre_ventes,
                COUNT(DISTINCT produit) as nombre_produits,
                COUNT(DISTINCT vendeur) as nombre_vendeurs,
                MIN(date) as premiere_vente,
                MAX(date) as derniere_vente,
                ROUND(SUM(chiffre_affaires), 2) as ca_total
            FROM ventes
        ''')

        stats = cursor.fetchone()
        if stats[0] == 0:
            print("⚠️ Aucune vente dans la base — analyses vides.")
            return  # éviter les ValueError sur les formats sur None

        print(f"Nombre de ventes : {stats[0]:,}")
        print(f"Produits différents : {stats[1]:,}")
        print(f"Vendeurs actifs : {stats[2]}")
        print(f"Période : {stats[3]} à {stats[4]}")
        print(f"CA total : {stats[5] or 0:,.2f} €")

        # 2. Top des catégories
        print("\n=== TOP CATÉGORIES ===")
        cursor = conn.execute('''
            SELECT
                categorie,
                COUNT(*) as nb_ventes,
                SUM(quantite) as quantite_totale,
                ROUND(SUM(chiffre_affaires), 2) as ca_categorie,
                ROUND(AVG(prix_unitaire), 2) as prix_moyen
            FROM ventes
            GROUP BY categorie
            ORDER BY ca_categorie DESC
        ''')

        for row in cursor.fetchall():
            print(f"{row[0]:<15} | {row[1]:>4} ventes | {row[2]:>6} unités | {row[3]:>10,.2f} € | {row[4]:>8.2f} € moy")

        # 3. Performance des vendeurs
        print("\n=== PERFORMANCE VENDEURS ===")
        cursor = conn.execute('''
            SELECT
                vendeur,
                COUNT(*) as nb_ventes,
                ROUND(SUM(chiffre_affaires), 2) as ca_vendeur,
                ROUND(AVG(chiffre_affaires), 2) as vente_moyenne
            FROM ventes
            GROUP BY vendeur
            ORDER BY ca_vendeur DESC
        ''')

        for row in cursor.fetchall():
            print(f"{row[0]:<12} | {row[1]:>4} ventes | {row[2]:>10,.2f} € | {row[3]:>8.2f} € par vente")

# Ajouter cette méthode à la classe AnalyseurVentes
analyseur.analyses_de_base()
```

### Analyses temporelles

```python
def analyser_tendances_temporelles(self):
    """Analyse l'évolution dans le temps"""
    with sqlite3.connect(self.db_path) as conn:

        # Evolution mensuelle du CA
        print("=== ÉVOLUTION MENSUELLE ===")
        cursor = conn.execute('''
            SELECT
                strftime('%Y-%m', date) as mois,
                COUNT(*) as nb_ventes,
                ROUND(SUM(chiffre_affaires), 2) as ca_mensuel,
                ROUND(AVG(chiffre_affaires), 2) as panier_moyen
            FROM ventes
            GROUP BY strftime('%Y-%m', date)
            ORDER BY mois
        ''')

        for row in cursor.fetchall():
            print(f"{row[0]} | {row[1]:>4} ventes | {row[2]:>10,.2f} € | Panier: {row[3]:>7.2f} €")

        # Jour de la semaine le plus profitable
        print("\n=== ANALYSE PAR JOUR DE LA SEMAINE ===")
        cursor = conn.execute('''
            SELECT
                CASE strftime('%w', date)
                    WHEN '0' THEN 'Dimanche'
                    WHEN '1' THEN 'Lundi'
                    WHEN '2' THEN 'Mardi'
                    WHEN '3' THEN 'Mercredi'
                    WHEN '4' THEN 'Jeudi'
                    WHEN '5' THEN 'Vendredi'
                    WHEN '6' THEN 'Samedi'
                END as jour_semaine,
                COUNT(*) as nb_ventes,
                ROUND(AVG(chiffre_affaires), 2) as vente_moyenne
            FROM ventes
            GROUP BY strftime('%w', date)
            ORDER BY vente_moyenne DESC
        ''')

        for row in cursor.fetchall():
            print(f"{row[0]:<10} | {row[1]:>4} ventes | {row[2]:>8.2f} € en moyenne")
```

## Étape 3 : Analyses avancées avec SQL

### Analyses de cohort (clients récurrents)

```python
def analyser_fidelite_vendeurs(self):
    """Analyse la régularité des vendeurs"""
    with sqlite3.connect(self.db_path) as conn:

        print("=== ANALYSE DE RÉGULARITÉ ===")
        cursor = conn.execute('''
            WITH vendeur_stats AS (
                SELECT
                    vendeur,
                    COUNT(DISTINCT date) as jours_actifs,
                    COUNT(*) as total_ventes,
                    julianday(MAX(date)) - julianday(MIN(date)) + 1 as periode_jours,
                    ROUND(SUM(chiffre_affaires), 2) as ca_total
                FROM ventes
                GROUP BY vendeur
            )
            SELECT
                vendeur,
                jours_actifs,
                total_ventes,
                periode_jours,
                ROUND(jours_actifs * 100.0 / periode_jours, 1) as taux_activite,
                ROUND(total_ventes * 1.0 / jours_actifs, 1) as ventes_par_jour_actif,
                ca_total
            FROM vendeur_stats
            ORDER BY taux_activite DESC
        ''')

        print("Vendeur      | Jours actifs | Total ventes | Période | Taux activité | Ventes/jour | CA total")
        print("-" * 95)
        for row in cursor.fetchall():
            print(f"{row[0]:<12} | {row[1]:>11} | {row[2]:>11} | {row[3]:>6.0f} j | {row[4]:>10.1f} % | {row[5]:>9.1f} | {row[6]:>8,.0f} €")
```

### Analyse ABC des produits

```python
def analyser_abc_produits(self):
    """Classification ABC des produits par chiffre d'affaires"""
    with sqlite3.connect(self.db_path) as conn:

        # Calculer le CA par produit et le pourcentage cumulé
        cursor = conn.execute('''
            WITH produit_ca AS (
                SELECT
                    produit,
                    ROUND(SUM(chiffre_affaires), 2) as ca_produit,
                    SUM(quantite) as quantite_totale
                FROM ventes
                GROUP BY produit
            ),
            produit_avec_cumul AS (
                SELECT
                    produit,
                    ca_produit,
                    quantite_totale,
                    ROUND(100.0 * ca_produit / SUM(ca_produit) OVER (), 2) as pourcentage_ca,
                    ROUND(100.0 * SUM(ca_produit) OVER (ORDER BY ca_produit DESC) / SUM(ca_produit) OVER (), 2) as cumul_pourcentage
                FROM produit_ca
                ORDER BY ca_produit DESC
            )
            SELECT
                produit,
                ca_produit,
                quantite_totale,
                pourcentage_ca,
                cumul_pourcentage,
                CASE
                    WHEN cumul_pourcentage <= 80 THEN 'A'
                    WHEN cumul_pourcentage <= 95 THEN 'B'
                    ELSE 'C'
                END as classe_abc
            FROM produit_avec_cumul
            ORDER BY ca_produit DESC
        ''')

        print("=== ANALYSE ABC DES PRODUITS ===")
        print("Produit                | CA        | Qté  | % CA  | % Cumulé | Classe")
        print("-" * 75)

        for row in cursor.fetchall():
            print(f"{row[0]:<22} | {row[1]:>8,.0f} € | {row[2]:>4} | {row[3]:>4.1f}% | {row[4]:>6.1f}% | {row[5]:>6}")
```

## Étape 4 : Visualisation des données

### Intégration avec Matplotlib

```python
import matplotlib.pyplot as plt  
import pandas as pd  

def generer_graphiques(self):
    """Génère des graphiques d'analyse"""

    # 1. Évolution du CA mensuel
    with sqlite3.connect(self.db_path) as conn:
        df_mensuel = pd.read_sql_query('''
            SELECT
                strftime('%Y-%m', date) as mois,
                SUM(chiffre_affaires) as ca_mensuel
            FROM ventes
            GROUP BY strftime('%Y-%m', date)
            ORDER BY mois
        ''', conn)

    plt.figure(figsize=(12, 8))

    # Graphique 1 : Évolution mensuelle
    plt.subplot(2, 2, 1)
    plt.plot(df_mensuel['mois'], df_mensuel['ca_mensuel'], marker='o')
    plt.title('Évolution du CA mensuel')
    plt.xlabel('Mois')
    plt.ylabel('CA (€)')
    plt.xticks(rotation=45)

    # 2. CA par catégorie
    with sqlite3.connect(self.db_path) as conn:
        df_categories = pd.read_sql_query('''
            SELECT
                categorie,
                SUM(chiffre_affaires) as ca_categorie
            FROM ventes
            GROUP BY categorie
            ORDER BY ca_categorie DESC
        ''', conn)

    plt.subplot(2, 2, 2)
    plt.pie(df_categories['ca_categorie'], labels=df_categories['categorie'], autopct='%1.1f%%')
    plt.title('Répartition du CA par catégorie')

    # 3. Performance vendeurs
    with sqlite3.connect(self.db_path) as conn:
        df_vendeurs = pd.read_sql_query('''
            SELECT
                vendeur,
                SUM(chiffre_affaires) as ca_vendeur
            FROM ventes
            GROUP BY vendeur
            ORDER BY ca_vendeur DESC
        ''', conn)

    plt.subplot(2, 2, 3)
    plt.bar(df_vendeurs['vendeur'], df_vendeurs['ca_vendeur'])
    plt.title('CA par vendeur')
    plt.xlabel('Vendeur')
    plt.ylabel('CA (€)')

    # 4. Distribution des prix
    with sqlite3.connect(self.db_path) as conn:
        df_prix = pd.read_sql_query('SELECT prix_unitaire FROM ventes', conn)

    plt.subplot(2, 2, 4)
    plt.hist(df_prix['prix_unitaire'], bins=20, alpha=0.7)
    plt.title('Distribution des prix unitaires')
    plt.xlabel('Prix (€)')
    plt.ylabel('Fréquence')

    plt.tight_layout()
    # ⚠️ Créer le dossier de sortie avant savefig sinon FileNotFoundError
    import os
    os.makedirs('exports', exist_ok=True)
    plt.savefig('exports/analyse_ventes.png', dpi=300, bbox_inches='tight')
    plt.show()

    print("📊 Graphiques sauvegardés dans exports/analyse_ventes.png")
```

## Étape 5 : Génération de rapports automatisés

### Rapport HTML automatique

```python
from datetime import datetime  
from html import escape  # ⚠️ Échappement obligatoire avant injection HTML  

def generer_rapport_html(self):
    """Génère un rapport HTML complet.

    ⚠️ Sécurité : les noms de produits/vendeurs viennent de la BDD et peuvent
       contenir du HTML/JavaScript injecté (XSS stocké). On utilise
       `html.escape()` sur toute valeur de la BDD avant interpolation.
    """

    html_content = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <title>Rapport d'Analyse des Ventes</title>
        <style>
            body {{ font-family: Arial, sans-serif; margin: 40px; }}
            table {{ border-collapse: collapse; width: 100%; margin: 20px 0; }}
            th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
            th {{ background-color: #f2f2f2; }}
            .metric {{ background-color: #e7f3ff; padding: 15px; margin: 10px 0; border-radius: 5px; }}
            .warning {{ background-color: #fff3cd; }}
            .success {{ background-color: #d4edda; }}
        </style>
    </head>
    <body>
        <h1>📊 Rapport d'Analyse des Ventes</h1>
        <p><strong>Généré le :</strong> {datetime.now().strftime('%d/%m/%Y à %H:%M')}</p>

        <h2>Métriques Principales</h2>
    """

    # Ajouter les métriques principales
    with sqlite3.connect(self.db_path) as conn:
        cursor = conn.execute('''
            SELECT
                COUNT(*) as nb_ventes,
                ROUND(SUM(chiffre_affaires), 2) as ca_total,
                ROUND(AVG(chiffre_affaires), 2) as panier_moyen,
                COUNT(DISTINCT vendeur) as nb_vendeurs
            FROM ventes
        ''')
        stats = cursor.fetchone()

        html_content += f'''
        <div class="metric success">
            <strong>Chiffre d'affaires total :</strong> {stats[1]:,.2f} €<br>
            <strong>Nombre de ventes :</strong> {stats[0]:,}<br>
            <strong>Panier moyen :</strong> {stats[2]:.2f} €<br>
            <strong>Vendeurs actifs :</strong> {stats[3]}
        </div>
        '''

        # Table des top produits
        html_content += "<h2>Top 10 des Produits</h2><table><tr><th>Produit</th><th>CA</th><th>Quantité</th></tr>"

        cursor = conn.execute('''
            SELECT produit, ROUND(SUM(chiffre_affaires), 2), SUM(quantite)
            FROM ventes
            GROUP BY produit
            ORDER BY SUM(chiffre_affaires) DESC
            LIMIT 10
        ''')

        for row in cursor.fetchall():
            # ⚠️ escape() obligatoire sur row[0] (nom de produit depuis la BDD)
            #    sinon un produit contenant '<script>...</script>' injecte du JS.
            html_content += (
                f"<tr><td>{escape(str(row[0]))}</td>"
                f"<td>{row[1]:,.2f} €</td>"
                f"<td>{row[2]}</td></tr>"
            )

        html_content += "</table>"

    html_content += """
        <h2>Recommandations</h2>
        <ul>
            <li>Analyser les produits classe A pour optimiser le stock</li>
            <li>Former les vendeurs moins performants</li>
            <li>Identifier les opportunités de cross-selling</li>
        </ul>
    </body>
    </html>
    """

    # Sauvegarder le rapport
    import os
    os.makedirs('exports', exist_ok=True)
    with open('exports/rapport_ventes.html', 'w', encoding='utf-8') as f:
        f.write(html_content)

    print("📄 Rapport HTML généré : exports/rapport_ventes.html")
```

## Exercices pratiques

### Exercice 1 : Analyse de données RH

Créez une base d'analyse pour des données d'employés :

```sql
CREATE TABLE employes (
    id INTEGER PRIMARY KEY,
    nom TEXT,
    departement TEXT,
    salaire REAL,
    date_embauche DATE,
    age INTEGER,
    performance_score REAL
);
```

**Objectifs :**
1. Analyser la distribution des salaires par département
2. Calculer l'ancienneté moyenne
3. Identifier les corrélations âge/salaire/performance

### Exercice 2 : Analyse de logs web

Importez des logs de serveur web et analysez :

```sql
CREATE TABLE logs_web (
    timestamp DATETIME,
    ip_address TEXT,
    method TEXT,
    url TEXT,
    status_code INTEGER,
    response_size INTEGER,
    user_agent TEXT
);
```

**Analyses à réaliser :**
1. Pages les plus visitées
2. Codes d'erreur fréquents
3. Pics de trafic horaires

### Exercice 3 : Dashboard interactif

Utilisez Plotly Dash pour créer un dashboard interactif :

```python
import dash  
from dash import dcc, html  
import plotly.express as px  

# Créer une interface web pour explorer vos données
```

## Optimisations pour l'analyse

### Configuration SQLite pour l'analytique

```python
def optimiser_pour_analyse(self):
    """Configure SQLite pour les performances analytiques"""
    with sqlite3.connect(self.db_path) as conn:
        # Augmenter le cache
        conn.execute("PRAGMA cache_size = -64000")  # 64MB

        # Mode WAL pour les lectures concurrentes
        conn.execute("PRAGMA journal_mode = WAL")

        # Optimiser pour les requêtes en lecture
        conn.execute("PRAGMA temp_store = MEMORY")
        conn.execute("PRAGMA mmap_size = 268435456")  # 256MB

        # Analyser les tables pour optimiser les requêtes
        conn.execute("ANALYZE")
```

### Index stratégiques pour l'analyse

```sql
-- Index composites pour les analyses temporelles
CREATE INDEX idx_ventes_date_vendeur ON ventes(date, vendeur);  
CREATE INDEX idx_ventes_categorie_date ON ventes(categorie, date);  

-- Index pour les agrégations
CREATE INDEX idx_ventes_ca ON ventes(chiffre_affaires);
```

### Bascule pratique vers DuckDB pour les agrégations lourdes

Quand un `SELECT … GROUP BY` sur plusieurs millions de lignes devient lent en SQLite (même avec les bons index), basculer vers **DuckDB** ne demande pas de migrer les données : il lit le fichier SQLite directement.

```python
# pip install duckdb
import duckdb

con = duckdb.connect()  # connexion DuckDB en mémoire  
con.execute("INSTALL sqlite; LOAD sqlite;")  
con.execute("ATTACH 'data/analysis.db' AS s (TYPE sqlite);")  

# La même requête analytique, exécutée par le moteur colonne de DuckDB.
# ⚠️ La colonne `date` est TEXT dans SQLite (cf. CREATE TABLE plus haut).
#    DuckDB exige un cast explicite `::DATE` car `strftime` n'auto-convertit
#    pas les VARCHAR. Sans le cast :
#      BinderException: Could not choose a best candidate function for
#      strftime(VARCHAR, STRING_LITERAL).
df = con.execute("""
    SELECT s.ventes.categorie,
           strftime(s.ventes.date::DATE, '%Y-%m') AS mois,
           SUM(s.ventes.quantite * s.ventes.prix_unitaire) AS ca
    FROM s.ventes
    GROUP BY 1, 2
    ORDER BY 1, 2
""").df()  # retourne un pandas DataFrame
```

> 💡 **Quand basculer ?** Si une agrégation prend plus de quelques secondes en SQLite, le moteur colonne DuckDB est souvent 10-100× plus rapide. Les écritures restent sur SQLite (OLTP), DuckDB sert uniquement de moteur de lecture analytique sur la même base.

## Bonnes pratiques pour l'analyse

### ✅ À faire

- **Nettoyer les données** avant l'analyse
- **Créer des vues** pour les requêtes répétitives
- **Documenter vos analyses** avec des commentaires SQL
- **Versionner vos scripts** d'analyse
- **Automatiser** la génération de rapports

### ❌ À éviter

- **Négliger la validation** des données
- **Faire des JOIN** sans index appropriés
- **Oublier d'analyser** les performances des requêtes
- **Stocker des résultats** sans date de génération
- **Ignorer les valeurs** NULL dans les calculs

## Outils complémentaires

### Pour aller plus loin

- **Jupyter Notebooks** : Pour l'exploration interactive
- **Apache Superset** : Dashboard BI avec SQLite (open source)
- **Metabase** : Outil de visualisation simple (open source)
- **DBeaver Community** : Client SQL universel avec graphiques basiques
- **JetBrains DataGrip** : Client SQL professionnel (payant)
- **DuckDB** : moteur analytique OLAP, lit nativement les fichiers SQLite via `ATTACH 'file.db' AS s (TYPE sqlite)` — souvent **10-100× plus rapide** sur les agrégations lourdes
- **Polars** : DataFrame Rust + Python, surclasse pandas, lit du SQLite

## Conclusion

SQLite est un excellent choix pour l'analyse de données de petite à moyenne taille. Sa simplicité, combinée à la puissance du SQL, permet de réaliser des analyses sophistiquées sans infrastructure complexe.

Les scripts développés dans cette section peuvent être adaptés à vos propres données et constituent une base solide pour vos projets d'analyse.

---

**Points clés à retenir :**
- SQLite est viable pour l'analyse jusqu'à plusieurs centaines de GB (cf. tableau « Comparaison avec d'autres outils »)
- Le vrai critère de bascule est la **concurrence d'écriture**, pas la taille
- L'optimisation des requêtes (index, `ANALYZE`, plans d'exécution) est cruciale
- La visualisation enrichit considérablement l'analyse
- L'automatisation des rapports fait gagner un temps précieux
- Pour les agrégations massives : DuckDB (lit les fichiers SQLite directement)

⏭️ [9.3 Migration de données entre différents formats](/09-cas-usage-avances-projets-pratiques/03-migration-donnees-differents-formats.md)
