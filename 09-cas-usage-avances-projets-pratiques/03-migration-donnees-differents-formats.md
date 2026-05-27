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
# ⚠️ NE PAS inclure 'sqlite3' (stdlib) ni 'csv' (stdlib) dans pip install
pip install pandas openpyxl xlsxwriter lxml requests pymysql psycopg2-binary chardet
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
        # Créer les dossiers parents AVANT setup_logging et init_database :
        # sqlite3.connect() et logging.FileHandler() n'ouvrent pas les
        # dossiers manquants → OperationalError / FileNotFoundError sinon.
        Path(self.db_path).parent.mkdir(parents=True, exist_ok=True)
        Path('logs').mkdir(parents=True, exist_ok=True)

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
        # ⚠️ Valider le nom de table pour éviter une injection via pandas.to_sql.
        #    pandas ne valide pas strictement les noms ; un nom comme
        #    "x; DROP TABLE clients" pourrait être problématique selon la version.
        import re
        if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', nom_table):
            raise ValueError(f"Nom de table invalide : {nom_table!r}")

        with sqlite3.connect(self.db_path) as conn:
            # `to_sql(if_exists='replace')` crée la table en inférant les types
            # depuis le DataFrame — pas besoin de méthode séparée pour le DDL.
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

    # Gérer les colonnes dupliquées — `pd.io.common.dedup_names` est une API
    # INTERNE non publique de pandas (peut casser sans préavis). On déduplique
    # manuellement : 'col', 'col', 'col' → 'col', 'col_1', 'col_2'
    seen = {}
    new_cols = []
    for c in df.columns:
        if c in seen:
            seen[c] += 1
            new_cols.append(f"{c}_{seen[c]}")
        else:
            seen[c] = 0
            new_cols.append(c)
    df.columns = new_cols

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
    """Tente de convertir une colonne vers le type le plus approprié.

    ⚠️ ATTENTION : `errors='coerce'` met silencieusement les valeurs
       non convertibles à NaN/NaT. Pour des colonnes mixtes type/valeur,
       cela peut causer une perte de données invisible.
       Vérifier le ratio de NaN avant d'accepter la conversion.

    ⚠️ Pas de `bare except:` ici : on cible TypeError/ValueError/AttributeError
       pour éviter de masquer KeyboardInterrupt/SystemExit.
    """
    erreurs = []

    if df[nom_colonne].dtype != 'object':
        return df, erreurs  # déjà typé, rien à faire

    try:
        # Essayer entiers d'abord (avec Int64 nullable pour gérer NaN)
        converted = pd.to_numeric(df[nom_colonne], errors='coerce')
        # Vérifier que ce sont bien des entiers (pas de partie décimale)
        if converted.dropna().apply(lambda x: x == int(x)).all():
            df[nom_colonne] = converted.astype('Int64')
        else:
            df[nom_colonne] = converted  # float
    except (TypeError, ValueError, AttributeError):
        # Essayer dates ensuite
        try:
            df[nom_colonne] = pd.to_datetime(df[nom_colonne], errors='coerce')
        except (TypeError, ValueError, AttributeError) as e:
            erreurs.append(f"Conversion impossible pour {nom_colonne} : {e}")
            # Garder comme texte (pas de modification)

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
    """Importe des données JSON vers SQLite.

    ⚠️ `if_exists='replace'` est destructeur : DROP TABLE + recréation.
       Si la table contient des index, triggers ou vues custom, ils sont
       perdus. Utiliser `'append'` ou `'fail'` selon besoin.

    ⚠️ `json.load()` charge tout le fichier en mémoire. Pour des JSON > 1 GB,
       préférer un parser streaming (`ijson` : `pip install ijson`).
    """
    # Validation du nom de table (interpolé en SQL par pandas.to_sql)
    import re
    if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', nom_table):
        raise ValueError(f"Nom de table invalide : {nom_table!r}")

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

        # Import dans SQLite (nom_table validé en regex ci-dessus)
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
    """Exporte une table SQLite vers JSON."""
    # Validation du nom de table (interpolé en SQL)
    import re
    if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', nom_table):
        raise ValueError(f"Nom de table invalide : {nom_table!r}")

    options = options or {
        'format': 'records',    # 'records', 'index', 'values'
        'indent': 2,
        'date_format': 'iso',
        'include_metadata': True
    }

    try:
        self.logger.info(f"📤 Export {nom_table} → {chemin_sortie}")

        # Récupérer les données (nom_table validé en regex)
        with sqlite3.connect(self.db_path) as conn:
            df = pd.read_sql_query(f'SELECT * FROM "{nom_table}"', conn)

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
    """Exporte plusieurs tables vers un fichier Excel multi-feuilles.

    ⚠️ Chaque nom de table dans `tables` est validé : interpolé en SQL,
       il doit être un identifiant strict (anti-injection).
    """
    import re
    pattern = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
    for t in tables:
        if not pattern.match(t):
            raise ValueError(f"Nom de table invalide : {t!r}")

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

                # Récupérer les données (nom_table validé en regex au début)
                with sqlite3.connect(self.db_path) as conn:
                    df = pd.read_sql_query(f'SELECT * FROM "{nom_table}"', conn)

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
                            # Calculer la largeur max. Sur un DataFrame vide,
                            # .map(len).max() vaut NaN → fallback sur la
                            # longueur de l'en-tête uniquement.
                            if len(df) > 0:
                                max_data = int(df[col].astype(str).map(len).max())
                            else:
                                max_data = 0
                            max_length = max(max_data, len(str(col)))
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
    """Migre des tables SQLite vers un autre SGBD.

    ⚠️ Validation OBLIGATOIRE des noms de tables : interpolés en SQL pour
       SELECT, TRUNCATE et CREATE TABLE.
    """
    import re
    pattern = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
    for t in tables:
        if not pattern.match(t):
            raise ValueError(f"Nom de table invalide : {t!r}")

    options = options or {
        'batch_size': 1000,
        'create_tables': True,
        'truncate_target': False,
        'handle_conflicts': 'replace'  # 'replace', 'ignore', 'error'
    }

    connection = None
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
        # Quotage cible : backticks pour MySQL, double-quotes ANSI sinon
        target_quote = '`' if sgbd_config['type'] == 'mysql' else '"'

        resultats = {}

        for nom_table in tables:
            self.logger.info(f"📋 Migration table : {nom_table}")

            try:
                # Récupérer les données SQLite (nom_table validé en regex au début).
                # Pour la source SQLite, les doubles-quotes ANSI suffisent.
                with sqlite3.connect(self.db_path) as sqlite_conn:
                    df = pd.read_sql_query(f'SELECT * FROM "{nom_table}"', sqlite_conn)

                # Créer la table dans le SGBD cible si nécessaire
                if options['create_tables']:
                    self.creer_table_sgbd(connection, nom_table, df, sgbd_config['type'])

                # Vider la table si demandé (nom_table validé + quotage cible adapté)
                if options['truncate_target']:
                    with connection.cursor() as cursor:
                        cursor.execute(
                            f'TRUNCATE TABLE {target_quote}{nom_table}{target_quote}'
                        )
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

        return resultats

    except Exception as e:
        error_msg = f"Erreur migration SGBD : {str(e)}"
        self.logger.error(error_msg)
        return {'success': False, 'error': error_msg}
    finally:
        # Garantir la fermeture de la connexion même en cas d'exception
        if connection is not None:
            try:
                connection.close()
            except Exception:
                pass

def inserer_par_lots(self, connection, nom_table, df, batch_size, sgbd_type):
    """Insère les lignes du DataFrame par lots dans le SGBD cible.

    ⚠️ Validation OBLIGATOIRE de `nom_table` ET des noms de colonnes : tous
       interpolés en SQL. Le quotage diffère selon le SGBD (backticks pour
       MySQL, double-quotes ANSI pour PostgreSQL).
    """
    import re
    ident = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
    if not ident.match(nom_table):
        raise ValueError(f"Nom de table invalide : {nom_table!r}")

    # Placeholder et quotage selon le SGBD
    placeholder = '%s' if sgbd_type in ('mysql', 'postgresql') else '?'
    quote = '`' if sgbd_type == 'mysql' else '"'

    cols = list(df.columns)
    for c in cols:
        if not ident.match(str(c)):
            raise ValueError(f"Nom de colonne invalide : {c!r}")
    cols_sql = ', '.join(f'{quote}{c}{quote}' for c in cols)
    placeholders = ', '.join([placeholder] * len(cols))
    sql = (
        f"INSERT INTO {quote}{nom_table}{quote} "
        f"({cols_sql}) VALUES ({placeholders})"
    )

    total = 0
    with connection.cursor() as cursor:
        for start in range(0, len(df), batch_size):
            chunk = df.iloc[start:start + batch_size]
            # Convertir en liste de tuples pour executemany
            data = [tuple(row) for row in chunk.itertuples(index=False, name=None)]
            cursor.executemany(sql, data)
            total += len(chunk)
            self.logger.info(f"   Lot {start // batch_size + 1} : {len(chunk)} lignes")
    connection.commit()
    return total

def creer_table_sgbd(self, connection, nom_table, df, sgbd_type):
    """Crée une table dans le SGBD cible basée sur le DataFrame.

    ⚠️ nom_table ET les noms de colonnes du DataFrame sont interpolés en SQL :
       les deux DOIVENT respecter ^[a-zA-Z_][a-zA-Z0-9_]*$ (pas d'espace, pas
       de quotes, pas de caractères spéciaux). Sinon ValueError.
    """
    import re
    ident = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
    if not ident.match(nom_table):
        raise ValueError(f"Nom de table invalide : {nom_table!r}")

    # Mapping élargi. Pandas peut produire int32, float32, datetime[us|ns],
    # tz-aware, et depuis pandas 3.x un dtype 'str' (au lieu de 'object').
    # On utilise des préfixes pour les datetime qui varient en résolution.
    type_mapping = {
        'mysql': {
            'object': 'TEXT', 'str': 'TEXT',
            'int64': 'BIGINT', 'int32': 'INT', 'int16': 'SMALLINT', 'int8': 'TINYINT',
            'float64': 'DOUBLE', 'float32': 'FLOAT',
            'bool': 'BOOLEAN',
        },
        'postgresql': {
            'object': 'TEXT', 'str': 'TEXT',
            'int64': 'BIGINT', 'int32': 'INTEGER', 'int16': 'SMALLINT', 'int8': 'SMALLINT',
            'float64': 'DOUBLE PRECISION', 'float32': 'REAL',
            'bool': 'BOOLEAN',
        },
    }

    def _map_dtype(dtype_str):
        """Mappe un dtype pandas → type SQL en gérant les variantes datetime."""
        if dtype_str in type_mapping[sgbd_type]:
            return type_mapping[sgbd_type][dtype_str]
        # Variantes datetime : datetime64[ns], datetime64[us], datetime64[ns, UTC]…
        if dtype_str.startswith('datetime64'):
            if 'UTC' in dtype_str or ',' in dtype_str:
                return 'TIMESTAMP WITH TIME ZONE' if sgbd_type == 'postgresql' else 'DATETIME'
            return 'TIMESTAMP' if sgbd_type == 'postgresql' else 'DATETIME'
        return 'TEXT'  # fallback

    # Quotage des identifiants selon le SGBD
    quote = '`' if sgbd_type == 'mysql' else '"'

    columns = []
    for col_name, dtype in df.dtypes.items():
        col_name_str = str(col_name)
        if not ident.match(col_name_str):
            raise ValueError(
                f"Nom de colonne invalide : {col_name_str!r}. "
                "Renommez-la dans le DataFrame avant migration."
            )
        col_type = _map_dtype(str(dtype))
        columns.append(f"{quote}{col_name_str}{quote} {col_type}")

    create_sql = (
        f"CREATE TABLE IF NOT EXISTS {quote}{nom_table}{quote} "
        f"({', '.join(columns)})"
    )

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
        # Réutiliser le logger du migrateur (sinon AttributeError plus loin)
        self.logger = self.migrator.logger

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

    def executer_export(self, etape):
        """Exécute une étape d'export selon son `format`."""
        self.logger.info(f"📤 Export : {etape['name']}")
        format_export = etape.get('format', 'json')
        tables = etape['tables']
        output = etape['output']

        if format_export == 'excel':
            self.migrator.exporter_vers_excel(tables, output)
        elif format_export == 'json':
            # JSON = un fichier par table (sinon chaque itération écraserait
            # la précédente). On suffixe le chemin de sortie si >1 table.
            import os
            base, ext = os.path.splitext(output)
            ext = ext or '.json'
            for t in tables:
                cible = output if len(tables) == 1 else f"{base}_{t}{ext}"
                self.migrator.exporter_vers_json(t, cible)
        elif format_export == 'csv':
            # CSV = un fichier par table. Si plusieurs tables et `output` est
            # un fichier unique, on suffixe par le nom de table pour éviter
            # que chaque itération n'écrase la précédente.
            import os, re, pandas as pd
            ident = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
            base, ext = os.path.splitext(output)
            ext = ext or '.csv'
            with sqlite3.connect(self.migrator.db_path) as conn:
                for t in tables:
                    if not ident.match(t):
                        raise ValueError(f"Nom de table invalide : {t!r}")
                    cible = output if len(tables) == 1 else f"{base}_{t}{ext}"
                    df = pd.read_sql_query(f'SELECT * FROM "{t}"', conn)
                    df.to_csv(cible, index=False, encoding='utf-8')
        else:
            raise ValueError(f"Format d'export non supporté : {format_export}")

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
import time

class MigrationMonitor:
    def __init__(self, migrator):
        self.migrator = migrator
        # Réutiliser le logger du migrateur, sinon les self.logger.* plantent
        self.logger = migrator.logger
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

        # `resultat` peut être None (migration sans valeur de retour) ou un dict ;
        # ne PAS faire `'key' in None` → TypeError.
        if isinstance(resultat, dict) and 'records_imported' in resultat:
            if resultat['records_imported'] < self.seuils['taille_min_donnees']:
                self.envoyer_alerte_donnees(
                    f"Trop peu de données importées: {resultat['records_imported']}"
                )

    def verifier_qualite_donnees(self, resultat):
        """Vérifie la qualité des données migrées"""

        if (isinstance(resultat, dict)
                and 'errors' in resultat
                and len(resultat['errors']) > 0):
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
    """Migre de gros volumes par petits lots.

    ⚠️ `source_query` doit être en DUR (constante) ou validée — pas de
       chaîne user-controlled : on l'interpole en f-string pour ajouter
       LIMIT/OFFSET, donc tout SQL injecté serait exécuté.

    ⚠️ `target_table` est utilisé par pandas.to_sql() en aval ; on impose
       quand même un identifiant strict pour éviter des comportements
       surprenants selon les versions de pandas.

    ⚠️ Pour les TRÈS grosses tables (>1 M lignes), préférer une fenêtre
       par clé primaire plutôt qu'OFFSET (qui devient O(n²)) :
       `WHERE id > ? ORDER BY id LIMIT ?` avec `last_id` du chunk précédent.
    """
    import re
    # Validation minimale : on refuse les ; pour éviter les commandes multiples
    if ';' in source_query:
        raise ValueError("source_query ne doit pas contenir ';'")
    if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', target_table):
        raise ValueError(f"target_table invalide : {target_table!r}")

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
        # Note : pas de time.sleep — laisser SQLite enchaîner les chunks,
        # un sleep arbitraire ne sert qu'à ralentir la migration.

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

> ⚠️ **SQLite et threads en écriture — piège majeur** : SQLite n'autorise qu'**un seul writer simultané** par base. Avec un `ThreadPoolExecutor` qui écrit en parallèle :  
> - **En mode rollback journal** (défaut) : `database is locked` quasi-garanti  
> - **En mode WAL** : les lecteurs peuvent paralléliser, mais les writers restent sérialisés en pratique  
>  
> **Stratégies viables** :  
> 1. **Parallel reads + serial writes** : threads pour lire/transformer en mémoire, un seul thread qui sérialise les écritures via une `queue.Queue`  
> 2. **Bases SQLite séparées par worker**, puis fusion finale via `ATTACH DATABASE`  
> 3. **Mode WAL + `PRAGMA busy_timeout = 30000`** + retry sur `OperationalError` — viable pour des charges légères  
>  
> Pour des migrations massives parallélisables, **DuckDB** ou **PostgreSQL** sont plus adaptés.

```python
import concurrent.futures  
from multiprocessing import cpu_count  

def migrer_en_parallele(self, liste_fichiers, max_workers=None):
    """Migre plusieurs fichiers en parallèle.

    ⚠️ Voir l'encadré ci-dessus : convient surtout aux **lectures/parsing**
       parallèles. Les écritures dans SQLite restent sérialisées même avec
       plusieurs threads → ce ThreadPoolExecutor parallélise surtout le
       parsing CSV/Excel/JSON, l'écriture SQLite reste un goulot d'étranglement.
    """

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
    """Migre seulement les nouvelles données depuis la dernière migration.

    ⚠️ `source_table` et `date_column` sont interpolés en SQL (les identifiants
       ne supportent pas les paramètres `?`). Validation regex obligatoire.
       La valeur `derniere_migration`, elle, passe par un paramètre lié.
    """
    import re
    ident = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
    for nom, val in [('source_table', source_table),
                     ('target_table', target_table),
                     ('date_column', date_column)]:
        if not ident.match(val):
            raise ValueError(f"{nom} invalide : {val!r}")

    # Récupérer la date de dernière migration
    with sqlite3.connect(self.db_path) as conn:
        cursor = conn.execute(
            "SELECT MAX(migration_date) FROM migrations_log WHERE target_table = ?",
            (target_table,)
        )
        derniere_migration = cursor.fetchone()[0]

    # Construire la requête incrémentale : valeur via paramètre lié
    if derniere_migration:
        query = f'SELECT * FROM "{source_table}" WHERE "{date_column}" > ?'
        params = (derniere_migration,)
    else:
        query = f'SELECT * FROM "{source_table}"'
        params = ()

    # Exécuter la migration incrémentale.
    # NOTE : `migrer_avec_requete(query, target_table, params)` doit être
    # implémentée selon votre logique — typiquement elle ouvre une connexion,
    # exécute la query paramétrée et insère le résultat via `to_sql`.
    return self.migrer_avec_requete(query, target_table, params)
```

### Synchronisation bidirectionnelle

> 📝 **Squelette conceptuel** : les méthodes `detecter_conflits`,  
> `resoudre_conflits`, `pousser_modifications_locales` et  
> `tirer_modifications_distantes` ne sont pas implémentées ci-dessous — elles  
> dépendent de votre stratégie de résolution de conflits. Voir l'encadré  
> « Solutions modernes pour conflits robustes » du module 9.1 (HLC, CRDTs,  
> UUIDv7) pour le détail des approches.

```python
def synchroniser_bidirectionnelle(self, table_locale, table_distante, cle_primaire):
    """Synchronise les modifications dans les deux sens (squelette)."""

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
    """Analyse la qualité des données après migration.

    ⚠️ `nom_table` et les noms de colonnes sont interpolés en SQL (PRAGMA
       et identifiants ne supportent pas les paramètres préparés `?`).
       Validation regex obligatoire si la valeur vient de l'extérieur.
    """
    import re
    pattern = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_]*$')
    if not pattern.match(nom_table):
        raise ValueError(f"Nom de table invalide : {nom_table!r}")

    with sqlite3.connect(self.db_path) as conn:
        # Statistiques générales — nom_table est validé ci-dessus
        stats = {
            'total_rows': conn.execute(f'SELECT COUNT(*) FROM "{nom_table}"').fetchone()[0],
            'columns': []
        }

        # Analyse par colonne : on récupère les noms de colonnes EXISTANTES
        # via PRAGMA — donc forcément valides puisque la table existe.
        cursor = conn.execute(f'PRAGMA table_info("{nom_table}")')
        colonnes = cursor.fetchall()

        for colonne in colonnes:
            nom_col = colonne[1]
            # Double-vérification car PRAGMA peut renvoyer des noms inhabituels
            if not pattern.match(nom_col):
                continue

            # Stats de la colonne (identifiants validés, sûr à interpoler)
            col_stats = {
                'name': nom_col,
                'type': colonne[2],
                'null_count': conn.execute(
                    f'SELECT COUNT(*) FROM "{nom_table}" WHERE "{nom_col}" IS NULL'
                ).fetchone()[0],
                'unique_count': conn.execute(
                    f'SELECT COUNT(DISTINCT "{nom_col}") FROM "{nom_table}"'
                ).fetchone()[0],
                'sample_values': []
            }

            # Échantillon de valeurs
            cursor = conn.execute(
                f'SELECT DISTINCT "{nom_col}" FROM "{nom_table}" LIMIT 5'
            )
            col_stats['sample_values'] = [row[0] for row in cursor.fetchall()]

            stats['columns'].append(col_stats)

    return stats

def generer_rapport_qualite(self, stats):
    """Génère un rapport de qualité des données"""

    print("=" * 60)
    print(f"RAPPORT DE QUALITÉ - {stats['total_rows']} enregistrements")
    print("=" * 60)

    # Garde-fou : table vide → division par zéro sur les pourcentages
    total = stats['total_rows']
    if total == 0:
        print("ℹ️ Table vide : aucune statistique à calculer.")
        return

    for col in stats['columns']:
        null_pct = (col['null_count'] / total) * 100
        unique_pct = (col['unique_count'] / total) * 100

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

⏭️ [9.4 Projet complet : création d'une application web avec SQLite](/09-cas-usage-avances-projets-pratiques/04-projet-complet-creation-application-web-sqlite.md)
