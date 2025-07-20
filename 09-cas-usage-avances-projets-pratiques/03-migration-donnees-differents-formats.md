🔝 Retour au [Sommaire](/SOMMAIRE.md)
# 9.3 : Migration de données entre différents formats

## Introduction : L'art de la migration de données

Dans le monde réel, les données n'existent jamais dans un seul format. Vous devrez souvent **importer des données** depuis Excel, CSV, JSON, ou d'autres bases de données, et parfois **exporter vos données SQLite** vers ces mêmes formats.

Cette section vous apprendra à maîtriser ces migrations de données de manière robuste et efficace.

### Pourquoi migrer des données ?

- **Consolidation** : Rassembler des données éparpillées
- **Changement d'outil** : Passer d'Excel à une vraie base de données
- **Intégration** : Connecter différents systèmes
- **Sauvegarde** : Exporter pour archivage ou partage
- **Analyse** : Importer pour traitement avec SQLite

### Types de migrations courantes

```
Sources communes → SQLite → Destinations fréquentes

CSV files     ↘         ↗ Excel (.xlsx)
Excel files   → SQLite ← JSON files
JSON/XML      ↗         ↘ Autres SGBD
Autres SGBD   ↙         ↖ API REST
```

## Préparation de l'environnement

### Installation des bibliothèques nécessaires

```bash
# Bibliothèques pour différents formats
pip install pandas openpyxl xlsxwriter lxml requests pymysql psycopg2-binary
```

### Structure de projet pour migrations

```
migration_projet/
├── data/
│   ├── source/           # Données d'origine
│   ├── target/           # Données converties
│   └── migration.db      # Base SQLite de travail
├── scripts/
│   ├── import_csv.py     # Import depuis CSV
│   ├── import_excel.py   # Import depuis Excel
│   ├── import_json.py    # Import depuis JSON
│   ├── export_data.py    # Export vers différents formats
│   └── migrate_db.py     # Migration entre SGBD
├── config/
│   └── connections.ini   # Configuration des connexions
└── logs/
    └── migration.log     # Journalisation des opérations
```

## Classe de base pour les migrations

### Gestionnaire de migration universel

```python
import sqlite3
import pandas as pd
import json
import logging
from datetime import datetime
from pathlib import Path

class MigrateurDonnees:
    def __init__(self, db_path="data/migration.db"):
        self.db_path = db_path
        self.setup_logging()
        self.init_database()

    def setup_logging(self):
        """Configure le système de logs"""
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler('logs/migration.log'),
                logging.StreamHandler()
            ]
        )
        self.logger = logging.getLogger(__name__)

    def init_database(self):
        """Initialise la base SQLite"""
        with sqlite3.connect(self.db_path) as conn:
            # Table de suivi des migrations
            conn.execute('''
                CREATE TABLE IF NOT EXISTS migrations_log (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    source_file TEXT NOT NULL,
                    target_table TEXT NOT NULL,
                    records_processed INTEGER,
                    records_success INTEGER,
                    records_errors INTEGER,
                    migration_date DATETIME DEFAULT CURRENT_TIMESTAMP,
                    status TEXT DEFAULT 'COMPLETED',
                    error_details TEXT
                )
            ''')

    def log_migration(self, source_file, target_table, processed, success, errors, status='COMPLETED', error_details=None):
        """Enregistre les détails d'une migration"""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute('''
                INSERT INTO migrations_log
                (source_file, target_table, records_processed, records_success, records_errors, status, error_details)
                VALUES (?, ?, ?, ?, ?, ?, ?)
            ''', (source_file, target_table, processed, success, errors, status, error_details))
```

## Migration depuis CSV

### Import CSV robuste avec validation

```python
def importer_csv(self, chemin_csv, nom_table, options=None):
    """Importe un fichier CSV vers SQLite avec validation complète"""

    # Options par défaut
    options = options or {
        'encoding': 'utf-8',
        'delimiter': ',',
        'quotechar': '"',
        'skip_rows': 0,
        'clean_headers': True,
        'validate_data': True
    }

    try:
        self.logger.info(f"🚀 Début import CSV : {chemin_csv} → {nom_table}")

        # 1. Lecture et inspection du fichier
        if not Path(chemin_csv).exists():
            raise FileNotFoundError(f"Fichier non trouvé : {chemin_csv}")

        # Détecter l'encodage automatiquement
        encoding = self.detecter_encodage(chemin_csv)

        # Lire le CSV avec pandas
        df = pd.read_csv(
            chemin_csv,
            encoding=encoding,
            delimiter=options['delimiter'],
            quotechar=options['quotechar'],
            skiprows=options['skip_rows'],
            low_memory=False
        )

        self.logger.info(f"📊 {len(df)} lignes lues depuis {Path(chemin_csv).name}")

        # 2. Nettoyage des en-têtes
        if options['clean_headers']:
            df = self.nettoyer_entetes(df)

        # 3. Validation et nettoyage des données
        if options['validate_data']:
            df_clean, erreurs = self.nettoyer_donnees_csv(df)
            self.logger.info(f"✅ {len(df_clean)} lignes valides, {len(erreurs)} erreurs")
        else:
            df_clean = df
            erreurs = []

        # 4. Import dans SQLite
        with sqlite3.connect(self.db_path) as conn:
            # Créer la table avec les bons types
            self.creer_table_depuis_dataframe(conn, nom_table, df_clean)

            # Insérer les données
            df_clean.to_sql(nom_table, conn, if_exists='replace', index=False)

        # 5. Logging de la migration
        self.log_migration(
            chemin_csv, nom_table,
            len(df), len(df_clean), len(erreurs)
        )

        self.logger.info(f"✅ Import terminé : {len(df_clean)} lignes dans {nom_table}")

        return {
            'success': True,
            'records_imported': len(df_clean),
            'errors': erreurs,
            'table_name': nom_table
        }

    except Exception as e:
        error_msg = f"Erreur lors de l'import CSV : {str(e)}"
        self.logger.error(error_msg)

        self.log_migration(
            chemin_csv, nom_table,
            0, 0, 1, 'FAILED', error_msg
        )

        return {
            'success': False,
            'error': error_msg
        }

def detecter_encodage(self, chemin_fichier):
    """Détecte automatiquement l'encodage d'un fichier"""
    import chardet

    with open(chemin_fichier, 'rb') as f:
        # Lire les premiers 10000 octets pour détecter l'encodage
        raw_data = f.read(10000)
        result = chardet.detect(raw_data)
        encoding = result['encoding']
        confidence = result['confidence']

        self.logger.info(f"🔍 Encodage détecté : {encoding} (confiance: {confidence:.2f})")

        # Fallback vers utf-8 si la confiance est faible
        if confidence < 0.7:
            encoding = 'utf-8'
            self.logger.warning("⚠️ Confiance faible, utilisation d'UTF-8 par défaut")

        return encoding

def nettoyer_entetes(self, df):
    """Nettoie les en-têtes de colonnes"""
    # Supprimer les espaces et caractères spéciaux
    df.columns = df.columns.str.strip()
    df.columns = df.columns.str.replace(' ', '_')
    df.columns = df.columns.str.replace('[^a-zA-Z0-9_]', '', regex=True)
    df.columns = df.columns.str.lower()

    # Gérer les colonnes dupliquées
    df.columns = pd.io.common.dedup_names(df.columns, is_potential_multiindex=False)

    return df

def nettoyer_donnees_csv(self, df):
    """Nettoie et valide les données CSV"""
    df_clean = df.copy()
    erreurs = []

    # Supprimer les lignes complètement vides
    avant = len(df_clean)
    df_clean = df_clean.dropna(how='all')
    lignes_vides = avant - len(df_clean)

    if lignes_vides > 0:
        self.logger.info(f"🧹 {lignes_vides} lignes vides supprimées")

    # Nettoyer les chaînes de caractères
    for col in df_clean.select_dtypes(include=['object']).columns:
        df_clean[col] = df_clean[col].astype(str).str.strip()
        df_clean[col] = df_clean[col].replace('nan', None)

    # Convertir les types de données quand c'est possible
    for col in df_clean.columns:
        df_clean, col_erreurs = self.convertir_type_colonne(df_clean, col)
        erreurs.extend(col_erreurs)

    return df_clean, erreurs

def convertir_type_colonne(self, df, nom_colonne):
    """Tente de convertir une colonne vers le type le plus approprié"""
    erreurs = []

    try:
        # Tenter conversion numérique
        if df[nom_colonne].dtype == 'object':
            # Essayer d'abord les entiers
            try:
                df[nom_colonne] = pd.to_numeric(df[nom_colonne], errors='coerce').astype('Int64')
            except:
                # Sinon essayer les décimaux
                try:
                    df[nom_colonne] = pd.to_numeric(df[nom_colonne], errors='coerce')
                except:
                    # Essayer les dates
                    try:
                        df[nom_colonne] = pd.to_datetime(df[nom_colonne], errors='coerce')
                    except:
                        pass  # Garder comme texte
    except Exception as e:
        erreurs.append(f"Erreur conversion colonne {nom_colonne}: {str(e)}")

    return df, erreurs
```

## Migration depuis Excel

### Import Excel multi-feuilles

```python
def importer_excel(self, chemin_excel, options=None):
    """Importe un fichier Excel (toutes les feuilles ou spécifiques)"""

    options = options or {
        'sheets': None,  # None = toutes les feuilles
        'header_row': 0,
        'skip_rows': None,
        'clean_data': True
    }

    try:
        self.logger.info(f"📗 Import Excel : {chemin_excel}")

        # Lire le fichier Excel
        excel_file = pd.ExcelFile(chemin_excel)
        self.logger.info(f"📋 Feuilles disponibles : {excel_file.sheet_names}")

        # Déterminer quelles feuilles importer
        if options['sheets'] is None:
            sheets_to_import = excel_file.sheet_names
        else:
            sheets_to_import = options['sheets']

        resultats = {}

        for sheet_name in sheets_to_import:
            self.logger.info(f"📄 Traitement de la feuille : {sheet_name}")

            try:
                # Lire la feuille
                df = pd.read_excel(
                    chemin_excel,
                    sheet_name=sheet_name,
                    header=options['header_row'],
                    skiprows=options['skip_rows']
                )

                # Nettoyer le nom de table (nom de feuille)
                nom_table = self.nettoyer_nom_table(sheet_name)

                # Nettoyage des données si demandé
                if options['clean_data']:
                    df = self.nettoyer_entetes(df)
                    df, erreurs = self.nettoyer_donnees_csv(df)

                # Import dans SQLite
                with sqlite3.connect(self.db_path) as conn:
                    df.to_sql(nom_table, conn, if_exists='replace', index=False)

                self.log_migration(
                    f"{chemin_excel}:{sheet_name}", nom_table,
                    len(df), len(df), 0
                )

                resultats[sheet_name] = {
                    'table_name': nom_table,
                    'records_imported': len(df),
                    'success': True
                }

                self.logger.info(f"✅ Feuille {sheet_name} → table {nom_table} ({len(df)} lignes)")

            except Exception as e:
                error_msg = f"Erreur feuille {sheet_name}: {str(e)}"
                self.logger.error(error_msg)
                resultats[sheet_name] = {'success': False, 'error': error_msg}

        return resultats

    except Exception as e:
        error_msg = f"Erreur lors de l'import Excel : {str(e)}"
        self.logger.error(error_msg)
        return {'success': False, 'error': error_msg}

def nettoyer_nom_table(self, nom):
    """Nettoie un nom pour en faire un nom de table SQLite valide"""
    import re

    # Supprimer caractères spéciaux et espaces
    nom_clean = re.sub(r'[^a-zA-Z0-9_]', '_', nom)
    nom_clean = nom_clean.strip('_').lower()

    # S'assurer que ça commence par une lettre
    if nom_clean and nom_clean[0].isdigit():
        nom_clean = 'table_' + nom_clean

    # Limiter la longueur
    if len(nom_clean) > 30:
        nom_clean = nom_clean[:30]

    return nom_clean or 'table_import'
```

## Migration depuis/vers JSON

### Import JSON avec structures complexes

```python
def importer_json(self, chemin_json, nom_table, options=None):
    """Importe des données JSON vers SQLite"""

    options = options or {
        'normalize': True,     # Aplatir les objets imbriqués
        'max_level': 2,        # Niveau max d'imbrication
        'array_handling': 'expand'  # 'expand' ou 'serialize'
    }

    try:
        self.logger.info(f"📄 Import JSON : {chemin_json}")

        # Lire le fichier JSON
        with open(chemin_json, 'r', encoding='utf-8') as f:
            data = json.load(f)

        # Déterminer la structure
        if isinstance(data, dict):
            # Si c'est un objet, chercher les tableaux
            df = self.extraire_dataframe_depuis_objet(data, options)
        elif isinstance(data, list):
            # Si c'est un tableau
            df = self.extraire_dataframe_depuis_tableau(data, options)
        else:
            raise ValueError("Format JSON non supporté")

        self.logger.info(f"📊 {len(df)} enregistrements extraits")

        # Import dans SQLite
        with sqlite3.connect(self.db_path) as conn:
            df.to_sql(nom_table, conn, if_exists='replace', index=False)

        self.log_migration(
            chemin_json, nom_table,
            len(df), len(df), 0
        )

        self.logger.info(f"✅ Import JSON terminé : {len(df)} lignes dans {nom_table}")

        return {
            'success': True,
            'records_imported': len(df),
            'table_name': nom_table
        }

    except Exception as e:
        error_msg = f"Erreur lors de l'import JSON : {str(e)}"
        self.logger.error(error_msg)
        return {'success': False, 'error': error_msg}

def extraire_dataframe_depuis_tableau(self, data, options):
    """Extrait un DataFrame depuis un tableau JSON"""

    if options['normalize']:
        # Utiliser json_normalize pour aplatir
        df = pd.json_normalize(
            data,
            max_level=options['max_level'],
            errors='ignore'
        )
    else:
        # Création directe du DataFrame
        df = pd.DataFrame(data)

    return df

def extraire_dataframe_depuis_objet(self, data, options):
    """Extrait un DataFrame depuis un objet JSON complexe"""

    # Chercher le premier tableau dans l'objet
    for key, value in data.items():
        if isinstance(value, list) and len(value) > 0:
            self.logger.info(f"📋 Utilisation du tableau '{key}' ({len(value)} éléments)")
            return self.extraire_dataframe_depuis_tableau(value, options)

    # Si pas de tableau trouvé, créer un DataFrame avec l'objet
    return pd.DataFrame([data])
```

### Export vers JSON

```python
def exporter_vers_json(self, nom_table, chemin_sortie, options=None):
    """Exporte une table SQLite vers JSON"""

    options = options or {
        'format': 'records',    # 'records', 'index', 'values'
        'indent': 2,
        'date_format': 'iso',
        'include_metadata': True
    }

    try:
        self.logger.info(f"📤 Export {nom_table} → {chemin_sortie}")

        # Récupérer les données
        with sqlite3.connect(self.db_path) as conn:
            df = pd.read_sql_query(f"SELECT * FROM {nom_table}", conn)

        # Préparer les données pour JSON
        if options['date_format'] == 'iso':
            # Convertir les dates au format ISO
            for col in df.select_dtypes(include=['datetime64']).columns:
                df[col] = df[col].dt.strftime('%Y-%m-%d %H:%M:%S')

        # Structure de sortie
        output_data = {
            'data': df.to_dict(orient=options['format']),
            'metadata': {
                'table_name': nom_table,
                'record_count': len(df),
                'columns': list(df.columns),
                'export_date': datetime.now().isoformat(),
                'source_database': self.db_path
            } if options['include_metadata'] else None
        }

        # Si pas de métadonnées, exporter directement les données
        if not options['include_metadata']:
            output_data = output_data['data']

        # Écrire le fichier JSON
        with open(chemin_sortie, 'w', encoding='utf-8') as f:
            json.dump(output_data, f, indent=options['indent'], ensure_ascii=False, default=str)

        self.logger.info(f"✅ Export JSON terminé : {len(df)} enregistrements")

        return {
            'success': True,
            'records_exported': len(df),
            'output_file': chemin_sortie
        }

    except Exception as e:
        error_msg = f"Erreur lors de l'export JSON : {str(e)}"
        self.logger.error(error_msg)
        return {'success': False, 'error': error_msg}
```

## Migration vers Excel

### Export Excel multi-feuilles avec formatage

```python
def exporter_vers_excel(self, tables, chemin_sortie, options=None):
    """Exporte plusieurs tables vers un fichier Excel multi-feuilles"""

    options = options or {
        'format_headers': True,
        'auto_adjust_columns': True,
        'add_summary': True,
        'freeze_panes': True
    }

    try:
        self.logger.info(f"📊 Export Excel : {len(tables)} tables → {chemin_sortie}")

        with pd.ExcelWriter(chemin_sortie, engine='xlsxwriter') as writer:

            # Données de résumé
            summary_data = []

            for nom_table in tables:
                self.logger.info(f"📋 Export table : {nom_table}")

                # Récupérer les données
                with sqlite3.connect(self.db_path) as conn:
                    df = pd.read_sql_query(f"SELECT * FROM {nom_table}", conn)

                # Nom de feuille (limité à 31 caractères pour Excel)
                sheet_name = nom_table[:31]

                # Écrire dans Excel
                df.to_excel(writer, sheet_name=sheet_name, index=False)

                # Formatage de la feuille
                if options['format_headers'] or options['auto_adjust_columns']:
                    worksheet = writer.sheets[sheet_name]
                    workbook = writer.book

                    # Format des en-têtes
                    if options['format_headers']:
                        header_format = workbook.add_format({
                            'bold': True,
                            'text_wrap': True,
                            'valign': 'top',
                            'fg_color': '#D7E4BC',
                            'border': 1
                        })

                        for col_num, value in enumerate(df.columns.values):
                            worksheet.write(0, col_num, value, header_format)

                    # Ajustement automatique des colonnes
                    if options['auto_adjust_columns']:
                        for i, col in enumerate(df.columns):
                            # Calculer la largeur max
                            max_length = max(
                                df[col].astype(str).map(len).max(),
                                len(str(col))
                            )
                            worksheet.set_column(i, i, min(max_length + 2, 50))

                    # Figer les volets
                    if options['freeze_panes']:
                        worksheet.freeze_panes(1, 0)

                # Ajouter aux stats de résumé
                summary_data.append({
                    'Table': nom_table,
                    'Feuille': sheet_name,
                    'Lignes': len(df),
                    'Colonnes': len(df.columns),
                    'Taille_MB': round(df.memory_usage(deep=True).sum() / 1024 / 1024, 2)
                })

            # Créer une feuille de résumé
            if options['add_summary']:
                summary_df = pd.DataFrame(summary_data)
                summary_df.to_excel(writer, sheet_name='_Résumé', index=False)

                # Formatage du résumé
                summary_sheet = writer.sheets['_Résumé']
                summary_format = writer.book.add_format({
                    'bold': True,
                    'fg_color': '#4F81BD',
                    'font_color': 'white'
                })

                for col_num, value in enumerate(summary_df.columns.values):
                    summary_sheet.write(0, col_num, value, summary_format)

        total_records = sum(item['Lignes'] for item in summary_data)
        self.logger.info(f"✅ Export Excel terminé : {total_records} enregistrements total")

        return {
            'success': True,
            'tables_exported': len(tables),
            'total_records': total_records,
            'output_file': chemin_sortie
        }

    except Exception as e:
        error_msg = f"Erreur lors de l'export Excel : {str(e)}"
        self.logger.error(error_msg)
        return {'success': False, 'error': error_msg}
```

## Migration entre SGBD

### Migration vers MySQL/PostgreSQL

```python
def migrer_vers_sgbd(self, tables, sgbd_config, options=None):
    """Migre des tables SQLite vers un autre SGBD"""

    options = options or {
        'batch_size': 1000,
        'create_tables': True,
        'truncate_target': False,
        'handle_conflicts': 'replace'  # 'replace', 'ignore', 'error'
    }

    try:
        # Connexion au SGBD cible
        if sgbd_config['type'] == 'mysql':
            import pymysql
            connection = pymysql.connect(**sgbd_config['params'])
        elif sgbd_config['type'] == 'postgresql':
            import psycopg2
            connection = psycopg2.connect(**sgbd_config['params'])
        else:
            raise ValueError(f"SGBD non supporté : {sgbd_config['type']}")

        self.logger.info(f"🔄 Migration vers {sgbd_config['type']}")

        resultats = {}

        for nom_table in tables:
            self.logger.info(f"📋 Migration table : {nom_table}")

            try:
                # Récupérer les données SQLite
                with sqlite3.connect(self.db_path) as sqlite_conn:
                    df = pd.read_sql_query(f"SELECT * FROM {nom_table}", sqlite_conn)

                # Créer la table dans le SGBD cible si nécessaire
                if options['create_tables']:
                    self.creer_table_sgbd(connection, nom_table, df, sgbd_config['type'])

                # Vider la table si demandé
                if options['truncate_target']:
                    with connection.cursor() as cursor:
                        cursor.execute(f"TRUNCATE TABLE {nom_table}")
                    connection.commit()

                # Insérer par lots
                records_inserted = self.inserer_par_lots(
                    connection, nom_table, df,
                    options['batch_size'], sgbd_config['type']
                )

                resultats[nom_table] = {
                    'success': True,
                    'records_migrated': records_inserted
                }

                self.logger.info(f"✅ Table {nom_table} migrée : {records_inserted} lignes")

            except Exception as e:
                error_msg = f"Erreur table {nom_table}: {str(e)}"
                self.logger.error(error_msg)
                resultats[nom_table] = {'success': False, 'error': error_msg}

        connection.close()
        return resultats

    except Exception as e:
        error_msg = f"Erreur migration SGBD : {str(e)}"
        self.logger.error(error_msg)
        return {'success': False, 'error': error_msg}

def creer_table_sgbd(self, connection, nom_table, df, sgbd_type):
    """Crée une table dans le SGBD cible basée sur le DataFrame"""

    # Mapping des types SQLite vers autres SGBD
    type_mapping = {
        'mysql': {
            'object': 'TEXT',
            'int64': 'BIGINT',
            'float64': 'DOUBLE',
            'datetime64[ns]': 'DATETIME',
            'bool': 'BOOLEAN'
        },
        'postgresql': {
            'object': 'TEXT',
            'int64': 'BIGINT',
            'float64': 'DOUBLE PRECISION',
            'datetime64[ns]': 'TIMESTAMP',
            'bool': 'BOOLEAN'
        }
    }

    # Construire la requête CREATE TABLE
    columns = []
    for col_name, dtype in df.dtypes.items():
        col_type = type_mapping[sgbd_type].get(str(dtype), 'TEXT')
        columns.append(f"{col_name} {col_type}")

    create_sql = f"CREATE TABLE IF NOT EXISTS {nom_table} ({', '.join(columns)})"

    with connection.cursor() as cursor:
        cursor.execute(create_sql)
    connection.commit()
```

## Exercices pratiques

### Exercice 1 : Migration complète d'un système

Vous avez reçu des données clients dans différents formats :

```
data/
├── clients_2023.csv      # Données clients année dernière
├── commandes.xlsx        # Feuilles : commandes, produits, livraisons
└── feedback.json         # Avis clients au format JSON
```

**Objectif :** Créer une base SQLite unifiée avec toutes ces données.

**Étapes :**
1. Analyser chaque fichier pour comprendre sa structure
2. Nettoyer et valider les données
3. Créer un schéma relationnel cohérent
4. Importer toutes les données
5. Créer des vues pour faciliter les requêtes

### Exercice 2 : Synchronisation bidirectionnelle

Créez un système qui :

1. **Importe** des données depuis une API REST
2. **Les traite** dans SQLite
3. **Exporte** les résultats vers Excel
4. **Synchronise** les modifications

```python
# Exemple de structure
def synchroniser_donnees():
    # 1. Récupérer depuis API
    donnees_api = requests.get('https://api.exemple.com/data').json()

    # 2. Importer dans SQLite
    migrator.importer_json_data(donnees_api, 'donnees_externes')

    # 3. Traitement avec SQL
    # ... requêtes de traitement ...

    # 4. Export vers Excel
    migrator.exporter_vers_excel(['donnees_traitees'], 'rapport_final.xlsx')

    # 5. Renvoyer les modifications à l'API
    # ... code de synchronisation ...
```

### Exercice 3 : Migration de legacy vers moderne

Migrez un ancien système Access (.mdb) vers SQLite moderne :

**Défis à résoudre :**
- Encodage des caractères spéciaux
- Types de données obsolètes
- Relations complexes
- Données corrompues ou incohérentes

## Automatisation des migrations

### Script de migration automatisé

```python
class AutoMigrator:
    def __init__(self, config_file="config/migration_config.json"):
        with open(config_file, 'r') as f:
            self.config = json.load(f)

        self.migrator = MigrateurDonnees()

    def executer_plan_migration(self):
        """Exécute un plan de migration complet"""

        plan = self.config['migration_plan']

        for etape in plan:
            self.logger.info(f"🎯 Étape : {etape['name']}")

            if etape['type'] == 'import_csv':
                self.migrator.importer_csv(
                    etape['source'],
                    etape['target_table'],
                    etape.get('options', {})
                )

            elif etape['type'] == 'import_excel':
                self.migrator.importer_excel(
                    etape['source'],
                    etape.get('options', {})
                )

            elif etape['type'] == 'import_json':
                self.migrator.importer_json(
                    etape['source'],
                    etape['target_table'],
                    etape.get('options', {})
                )

            elif etape['type'] == 'transform':
                self.executer_transformations(etape['transformations'])

            elif etape['type'] == 'export':
                self.executer_export(etape)

            elif etape['type'] == 'validate':
                self.valider_donnees(etape['validations'])

    def executer_transformations(self, transformations):
        """Exécute des transformations SQL"""
        with sqlite3.connect(self.migrator.db_path) as conn:
            for transform in transformations:
                self.logger.info(f"🔄 Transformation : {transform['name']}")
                conn.execute(transform['sql'])
                conn.commit()

    def valider_donnees(self, validations):
        """Valide la qualité des données migrées"""
        with sqlite3.connect(self.migrator.db_path) as conn:
            for validation in validations:
                cursor = conn.execute(validation['sql'])
                result = cursor.fetchone()[0]

                if validation['operator'] == 'equals' and result != validation['expected']:
                    raise ValueError(f"Validation échouée : {validation['name']}")
                elif validation['operator'] == 'greater_than' and result <= validation['expected']:
                    raise ValueError(f"Validation échouée : {validation['name']}")

                self.logger.info(f"✅ Validation réussie : {validation['name']}")

# Exemple de fichier de configuration migration_config.json
migration_config_example = {
    "migration_plan": [
        {
            "name": "Import clients CSV",
            "type": "import_csv",
            "source": "data/source/clients.csv",
            "target_table": "clients",
            "options": {
                "encoding": "utf-8",
                "clean_headers": True,
                "validate_data": True
            }
        },
        {
            "name": "Import commandes Excel",
            "type": "import_excel",
            "source": "data/source/commandes.xlsx",
            "options": {
                "sheets": ["commandes", "produits"],
                "clean_data": True
            }
        },
        {
            "name": "Normalisation des données",
            "type": "transform",
            "transformations": [
                {
                    "name": "Nettoyage emails",
                    "sql": "UPDATE clients SET email = LOWER(TRIM(email)) WHERE email IS NOT NULL"
                },
                {
                    "name": "Création table commandes_enrichies",
                    "sql": """
                        CREATE TABLE commandes_enrichies AS
                        SELECT
                            c.*,
                            cl.nom as nom_client,
                            cl.email as email_client
                        FROM commandes c
                        JOIN clients cl ON c.client_id = cl.id
                    """
                }
            ]
        },
        {
            "name": "Validation des données",
            "type": "validate",
            "validations": [
                {
                    "name": "Nombre de clients importés",
                    "sql": "SELECT COUNT(*) FROM clients",
                    "operator": "greater_than",
                    "expected": 0
                },
                {
                    "name": "Emails uniques",
                    "sql": "SELECT COUNT(*) FROM clients WHERE email IS NOT NULL GROUP BY email HAVING COUNT(*) > 1",
                    "operator": "equals",
                    "expected": 0
                }
            ]
        },
        {
            "name": "Export final",
            "type": "export",
            "format": "excel",
            "tables": ["clients", "commandes_enrichies"],
            "output": "data/target/donnees_migrees.xlsx"
        }
    ]
}
```

### Monitoring et alertes

```python
class MigrationMonitor:
    def __init__(self, migrator):
        self.migrator = migrator
        self.seuils = {
            'taux_erreur_max': 0.05,  # 5% d'erreurs max
            'temps_execution_max': 3600,  # 1 heure max
            'taille_min_donnees': 100  # Minimum 100 enregistrements
        }

    def surveiller_migration(self, fonction_migration, *args, **kwargs):
        """Surveille une migration et génère des alertes"""

        debut = time.time()

        try:
            # Exécuter la migration
            resultat = fonction_migration(*args, **kwargs)

            fin = time.time()
            duree = fin - debut

            # Vérifications
            self.verifier_performances(duree, resultat)
            self.verifier_qualite_donnees(resultat)

            # Log de succès
            self.logger.info(f"✅ Migration surveillée terminée en {duree:.2f}s")

            return resultat

        except Exception as e:
            # Alerte en cas d'échec
            self.envoyer_alerte_echec(str(e))
            raise

    def verifier_performances(self, duree, resultat):
        """Vérifie les performances de la migration"""

        if duree > self.seuils['temps_execution_max']:
            self.envoyer_alerte_performance(f"Migration trop lente: {duree:.2f}s")

        if 'records_imported' in resultat:
            if resultat['records_imported'] < self.seuils['taille_min_donnees']:
                self.envoyer_alerte_donnees(
                    f"Trop peu de données importées: {resultat['records_imported']}"
                )

    def verifier_qualite_donnees(self, resultat):
        """Vérifie la qualité des données migrées"""

        if 'errors' in resultat and len(resultat['errors']) > 0:
            taux_erreur = len(resultat['errors']) / resultat.get('records_processed', 1)

            if taux_erreur > self.seuils['taux_erreur_max']:
                self.envoyer_alerte_qualite(
                    f"Taux d'erreur élevé: {taux_erreur:.2%}"
                )

    def envoyer_alerte_echec(self, message):
        """Envoie une alerte en cas d'échec de migration"""
        # Ici vous pouvez implémenter l'envoi d'email, Slack, etc.
        self.logger.error(f"🚨 ALERTE ÉCHEC MIGRATION: {message}")

    def envoyer_alerte_performance(self, message):
        """Alerte de performance"""
        self.logger.warning(f"⚠️ ALERTE PERFORMANCE: {message}")

    def envoyer_alerte_donnees(self, message):
        """Alerte sur la quantité de données"""
        self.logger.warning(f"⚠️ ALERTE DONNÉES: {message}")

    def envoyer_alerte_qualite(self, message):
        """Alerte sur la qualité des données"""
        self.logger.warning(f"⚠️ ALERTE QUALITÉ: {message}")
```

## Optimisations pour grandes migrations

### Migration par chunks pour gros volumes

```python
def migrer_gros_volume(self, source_query, target_table, chunk_size=10000):
    """Migre de gros volumes par petits lots"""

    offset = 0
    total_migre = 0

    while True:
        # Récupérer un chunk
        chunk_query = f"{source_query} LIMIT {chunk_size} OFFSET {offset}"

        with sqlite3.connect(self.db_path) as conn:
            df_chunk = pd.read_sql_query(chunk_query, conn)

        # Arrêter si plus de données
        if len(df_chunk) == 0:
            break

        # Traiter le chunk
        self.traiter_chunk(df_chunk, target_table)

        total_migre += len(df_chunk)
        offset += chunk_size

        self.logger.info(f"📦 Chunk traité: {total_migre} enregistrements")

        # Pause pour éviter la surcharge
        time.sleep(0.1)

    self.logger.info(f"✅ Migration gros volume terminée: {total_migre} enregistrements")

def traiter_chunk(self, df_chunk, target_table):
    """Traite un chunk de données"""

    # Nettoyage du chunk
    df_clean = self.nettoyer_donnees_csv(df_chunk)[0]

    # Insert dans la table cible
    with sqlite3.connect(self.db_path) as conn:
        df_clean.to_sql(target_table, conn, if_exists='append', index=False)
```

### Parallélisation des migrations

```python
import concurrent.futures
from multiprocessing import cpu_count

def migrer_en_parallele(self, liste_fichiers, max_workers=None):
    """Migre plusieurs fichiers en parallèle"""

    if max_workers is None:
        max_workers = min(cpu_count(), len(liste_fichiers))

    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:

        # Soumettre toutes les tâches
        futures = {}
        for fichier_info in liste_fichiers:
            future = executor.submit(
                self.migrer_fichier_unique,
                fichier_info['path'],
                fichier_info['table'],
                fichier_info.get('options', {})
            )
            futures[future] = fichier_info['path']

        # Collecter les résultats
        resultats = {}
        for future in concurrent.futures.as_completed(futures):
            fichier = futures[future]
            try:
                resultat = future.result()
                resultats[fichier] = resultat
                self.logger.info(f"✅ Migration parallèle terminée: {fichier}")
            except Exception as e:
                self.logger.error(f"❌ Erreur migration parallèle {fichier}: {e}")
                resultats[fichier] = {'success': False, 'error': str(e)}

    return resultats

def migrer_fichier_unique(self, chemin_fichier, nom_table, options):
    """Migre un seul fichier (utilisé pour la parallélisation)"""

    extension = Path(chemin_fichier).suffix.lower()

    if extension == '.csv':
        return self.importer_csv(chemin_fichier, nom_table, options)
    elif extension in ['.xlsx', '.xls']:
        return self.importer_excel(chemin_fichier, options)
    elif extension == '.json':
        return self.importer_json(chemin_fichier, nom_table, options)
    else:
        raise ValueError(f"Format de fichier non supporté: {extension}")
```

## Bonnes pratiques pour la migration

### ✅ À faire

- **Sauvegarder** les données originales avant migration
- **Valider** les données après chaque étape
- **Logger** toutes les opérations pour audit
- **Tester** sur un échantillon avant migration complète
- **Documenter** le processus de migration
- **Prévoir** un plan de rollback en cas d'échec
- **Monitorer** les performances et la qualité

### ❌ À éviter

- **Migrer sans validation** préalable des données
- **Ignorer les erreurs** de conversion de types
- **Négliger l'encodage** des caractères
- **Oublier les contraintes** de clés étrangères
- **Surcharger** la mémoire avec de gros datasets
- **Migrer en production** sans tests préalables

## Cas d'usage avancés

### Migration incrémentale

```python
def migration_incrementale(self, source_table, target_table, date_column):
    """Migre seulement les nouvelles données depuis la dernière migration"""

    # Récupérer la date de dernière migration
    with sqlite3.connect(self.db_path) as conn:
        cursor = conn.execute(
            "SELECT MAX(migration_date) FROM migrations_log WHERE target_table = ?",
            (target_table,)
        )
        derniere_migration = cursor.fetchone()[0]

    # Construire la requête incrémentale
    if derniere_migration:
        where_clause = f"WHERE {date_column} > '{derniere_migration}'"
    else:
        where_clause = ""

    query = f"SELECT * FROM {source_table} {where_clause}"

    # Exécuter la migration incrémentale
    return self.migrer_avec_requete(query, target_table)
```

### Synchronisation bidirectionnelle

```python
def synchroniser_bidirectionnelle(self, table_locale, table_distante, cle_primaire):
    """Synchronise les modifications dans les deux sens"""

    # 1. Identifier les conflits
    conflits = self.detecter_conflits(table_locale, table_distante, cle_primaire)

    # 2. Résoudre les conflits (stratégie à définir)
    self.resoudre_conflits(conflits)

    # 3. Appliquer les modifications locales vers distant
    self.pousser_modifications_locales(table_locale, table_distante)

    # 4. Récupérer les modifications distantes
    self.tirer_modifications_distantes(table_distante, table_locale)

    # 5. Marquer la synchronisation
    self.marquer_synchronisation(table_locale)
```

## Outils de debugging et diagnostic

### Analyseur de qualité de données

```python
def analyser_qualite_migration(self, nom_table):
    """Analyse la qualité des données après migration"""

    with sqlite3.connect(self.db_path) as conn:
        # Statistiques générales
        stats = {
            'total_rows': conn.execute(f"SELECT COUNT(*) FROM {nom_table}").fetchone()[0],
            'columns': []
        }

        # Analyse par colonne
        cursor = conn.execute(f"PRAGMA table_info({nom_table})")
        colonnes = cursor.fetchall()

        for colonne in colonnes:
            nom_col = colonne[1]

            # Stats de la colonne
            col_stats = {
                'name': nom_col,
                'type': colonne[2],
                'null_count': conn.execute(f"SELECT COUNT(*) FROM {nom_table} WHERE {nom_col} IS NULL").fetchone()[0],
                'unique_count': conn.execute(f"SELECT COUNT(DISTINCT {nom_col}) FROM {nom_table}").fetchone()[0],
                'sample_values': []
            }

            # Échantillon de valeurs
            cursor = conn.execute(f"SELECT DISTINCT {nom_col} FROM {nom_table} LIMIT 5")
            col_stats['sample_values'] = [row[0] for row in cursor.fetchall()]

            stats['columns'].append(col_stats)

    return stats

def generer_rapport_qualite(self, stats):
    """Génère un rapport de qualité des données"""

    print("=" * 60)
    print(f"RAPPORT DE QUALITÉ - {stats['total_rows']} enregistrements")
    print("=" * 60)

    for col in stats['columns']:
        null_pct = (col['null_count'] / stats['total_rows']) * 100
        unique_pct = (col['unique_count'] / stats['total_rows']) * 100

        print(f"\n📊 {col['name']} ({col['type']})")
        print(f"   Valeurs nulles: {col['null_count']} ({null_pct:.1f}%)")
        print(f"   Valeurs uniques: {col['unique_count']} ({unique_pct:.1f}%)")
        print(f"   Échantillon: {col['sample_values']}")

        # Alertes qualité
        if null_pct > 50:
            print(f"   ⚠️ ATTENTION: Trop de valeurs nulles")
        if unique_pct < 5 and col['name'] != 'id':
            print(f"   ⚠️ ATTENTION: Peu de diversité dans les données")
```

## Conclusion

La migration de données est un aspect crucial de tout projet impliquant SQLite. Une approche méthodique, avec validation, logging et monitoring, garantit des migrations fiables et traçables.

Les scripts et techniques présentés dans cette section constituent une boîte à outils complète pour gérer tous vos besoins de migration de données, du simple import CSV à la synchronisation complexe entre systèmes.

---

**Points clés à retenir :**

- **Planifiez** toujours vos migrations avec des tests préalables
- **Validez** les données à chaque étape du processus
- **Automatisez** les tâches répétitives avec des scripts robustes
- **Surveillez** la qualité et les performances des migrations
- **Documentez** vos processus pour faciliter la maintenance
- **Préparez** des stratégies de rollback en cas de problème
- **Testez** sur des échantillons avant les migrations complètes

⏭️
