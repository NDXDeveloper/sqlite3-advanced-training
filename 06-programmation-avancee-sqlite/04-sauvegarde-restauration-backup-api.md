🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.4 Sauvegarde et restauration (backup API)

## Pourquoi sauvegarder vos données ?

La sauvegarde de bases de données est **critique** pour toute application. Imaginez perdre toutes vos données clients, commandes, ou configurations suite à une panne matérielle, une erreur humaine, ou une corruption de fichier.

### Analogie simple
Pensez à la sauvegarde comme à une **assurance pour vos données** : vous espérez ne jamais en avoir besoin, mais quand c'est le cas, elle vous sauve la mise !

### Scenarios de perte de données
- 💾 **Panne matérielle** : Disque dur qui crash
- 🔥 **Sinistre** : Incendie, inondation
- 👤 **Erreur humaine** : Suppression accidentelle
- 🐛 **Bug logiciel** : Corruption de données
- 🦠 **Malware** : Ransomware, virus
- ⚡ **Coupure électrique** : Corruption pendant écriture

## Types de sauvegarde SQLite

### 1. Copie simple du fichier

La méthode la plus basique (mais dangereuse si la base est en cours d'utilisation) :

```bash
# ❌ Dangereux si la base est utilisée
cp ma_base.db ma_base_backup.db

# ❌ Risque de corruption
sqlite3 ma_base.db ".backup ma_base_backup.db"
```

**Problèmes :**
- Risque de corruption si des écritures sont en cours
- Pas de contrôle sur la progression
- Aucune garantie de cohérence

### 2. Backup API (Recommandé)

SQLite fournit une API de sauvegarde spécialisée qui garantit la cohérence :

```python
import sqlite3

def sauvegarde_simple(source_db, backup_db):
    """Sauvegarde basique avec l'API SQLite"""

    # Connexion à la base source
    source = sqlite3.connect(source_db)

    # Connexion à la base de destination (créée si inexistante)
    backup = sqlite3.connect(backup_db)

    try:
        # Effectuer la sauvegarde
        source.backup(backup)
        print(f"✅ Sauvegarde réussie : {source_db} → {backup_db}")

    except sqlite3.Error as e:
        print(f"❌ Erreur de sauvegarde : {e}")

    finally:
        source.close()
        backup.close()

# Utilisation
sauvegarde_simple('ma_base.db', 'sauvegarde_2024_01_15.db')
```

### 3. Sauvegarde incrémentale

L'API permet également des sauvegardes par petits blocs :

```python
import sqlite3
import time

def sauvegarde_progressive(source_db, backup_db, pages_par_fois=100):
    """Sauvegarde progressive pour grandes bases de données"""

    source = sqlite3.connect(source_db)
    backup = sqlite3.connect(backup_db)

    try:
        # Démarrer la sauvegarde
        backup_obj = source.backup(backup, pages=pages_par_fois, progress=callback_progression)

        print(f"✅ Sauvegarde progressive terminée : {source_db} → {backup_db}")

    except sqlite3.Error as e:
        print(f"❌ Erreur de sauvegarde progressive : {e}")

    finally:
        source.close()
        backup.close()

def callback_progression(status, remaining, total):
    """Callback appelé pendant la sauvegarde"""
    progression = ((total - remaining) / total) * 100
    print(f"📊 Progression : {progression:.1f}% ({total - remaining}/{total} pages)")

    # Petite pause pour ne pas monopoliser les ressources
    time.sleep(0.01)

# Utilisation pour une grosse base
sauvegarde_progressive('grosse_base.db', 'sauvegarde_grosse_base.db')
```

## Système de sauvegarde complet

Créons un système de sauvegarde professionnel avec rotation, compression et vérification :

```python
import sqlite3
import os
import shutil
import gzip
import time
from datetime import datetime, timedelta
import hashlib
import logging

class GestionnaireBackup:
    def __init__(self, db_path, repertoire_backup="./backups"):
        self.db_path = db_path
        self.repertoire_backup = repertoire_backup
        self.logger = self._configurer_logging()

        # Créer le répertoire de sauvegarde s'il n'existe pas
        os.makedirs(repertoire_backup, exist_ok=True)

    def _configurer_logging(self):
        """Configure le système de logs"""
        logger = logging.getLogger('backup')
        logger.setLevel(logging.INFO)

        if not logger.handlers:
            handler = logging.StreamHandler()
            formatter = logging.Formatter(
                '%(asctime)s - %(levelname)s - %(message)s'
            )
            handler.setFormatter(formatter)
            logger.addHandler(handler)

        return logger

    def sauvegarder(self, comprimer=True, verifier=True):
        """Effectue une sauvegarde complète avec options"""

        # Générer un nom de fichier horodaté
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        nom_base = os.path.basename(self.db_path).replace('.db', '')
        nom_backup = f"{nom_base}_backup_{timestamp}.db"
        chemin_backup = os.path.join(self.repertoire_backup, nom_backup)

        debut = time.time()
        self.logger.info(f"🚀 Début de sauvegarde : {self.db_path}")

        try:
            # Étape 1 : Sauvegarde avec l'API SQLite
            self._effectuer_backup(chemin_backup)

            # Étape 2 : Vérification de l'intégrité
            if verifier:
                self._verifier_integrite(chemin_backup)

            # Étape 3 : Compression
            if comprimer:
                chemin_backup = self._comprimer_backup(chemin_backup)

            # Étape 4 : Calculer et enregistrer le checksum
            checksum = self._calculer_checksum(chemin_backup)
            self._enregistrer_metadata(chemin_backup, checksum)

            duree = time.time() - debut
            taille = os.path.getsize(chemin_backup)

            self.logger.info(f"✅ Sauvegarde terminée en {duree:.2f}s")
            self.logger.info(f"📁 Fichier : {chemin_backup}")
            self.logger.info(f"📊 Taille : {taille / (1024*1024):.2f} MB")
            self.logger.info(f"🔐 Checksum : {checksum}")

            return chemin_backup

        except Exception as e:
            self.logger.error(f"❌ Erreur de sauvegarde : {e}")
            # Nettoyer le fichier partiel
            if os.path.exists(chemin_backup):
                os.remove(chemin_backup)
            raise

    def _effectuer_backup(self, chemin_destination):
        """Effectue la sauvegarde avec l'API SQLite"""
        source = sqlite3.connect(self.db_path)
        destination = sqlite3.connect(chemin_destination)

        try:
            # Sauvegarde avec callback de progression
            def progression(status, remaining, total):
                if total > 1000:  # Afficher seulement pour les grosses bases
                    pct = ((total - remaining) / total) * 100
                    if int(pct) % 10 == 0:  # Afficher tous les 10%
                        self.logger.info(f"📊 Progression : {pct:.0f}%")

            source.backup(destination, progress=progression)

        finally:
            source.close()
            destination.close()

    def _verifier_integrite(self, chemin_backup):
        """Vérifie l'intégrité de la sauvegarde"""
        self.logger.info("🔍 Vérification de l'intégrité...")

        conn = sqlite3.connect(chemin_backup)
        try:
            # Test d'intégrité SQLite
            result = conn.execute("PRAGMA integrity_check").fetchone()
            if result[0] != "ok":
                raise Exception(f"Intégrité compromise : {result[0]}")

            # Test de quelques requêtes de base
            conn.execute("SELECT name FROM sqlite_master LIMIT 1").fetchall()

            self.logger.info("✅ Intégrité vérifiée")

        finally:
            conn.close()

    def _comprimer_backup(self, chemin_backup):
        """Comprime la sauvegarde avec gzip"""
        self.logger.info("🗜️ Compression en cours...")

        chemin_compresse = chemin_backup + '.gz'

        with open(chemin_backup, 'rb') as f_in:
            with gzip.open(chemin_compresse, 'wb') as f_out:
                shutil.copyfileobj(f_in, f_out)

        # Supprimer le fichier non compressé
        os.remove(chemin_backup)

        self.logger.info("✅ Compression terminée")
        return chemin_compresse

    def _calculer_checksum(self, chemin_fichier):
        """Calcule le checksum SHA256 du fichier"""
        sha256 = hashlib.sha256()

        with open(chemin_fichier, 'rb') as f:
            for chunk in iter(lambda: f.read(4096), b""):
                sha256.update(chunk)

        return sha256.hexdigest()

    def _enregistrer_metadata(self, chemin_backup, checksum):
        """Enregistre les métadonnées de la sauvegarde"""
        metadata_file = chemin_backup + '.info'

        metadata = {
            'fichier_source': self.db_path,
            'fichier_backup': chemin_backup,
            'date_creation': datetime.now().isoformat(),
            'checksum': checksum,
            'taille': os.path.getsize(chemin_backup)
        }

        with open(metadata_file, 'w') as f:
            for key, value in metadata.items():
                f.write(f"{key}: {value}\n")

    def rotation_backups(self, garder_jours=30, garder_hebdo=12, garder_mensuel=12):
        """Rotation intelligente des sauvegardes"""
        self.logger.info("🔄 Rotation des sauvegardes...")

        fichiers_backup = []

        # Lister tous les fichiers de sauvegarde
        for fichier in os.listdir(self.repertoire_backup):
            if fichier.endswith('.db.gz') and 'backup_' in fichier:
                chemin_complet = os.path.join(self.repertoire_backup, fichier)
                timestamp = os.path.getctime(chemin_complet)
                fichiers_backup.append((chemin_complet, timestamp))

        # Trier par date (plus récent en premier)
        fichiers_backup.sort(key=lambda x: x[1], reverse=True)

        maintenant = time.time()
        a_conserver = set()

        # Règles de conservation
        for chemin, timestamp in fichiers_backup:
            age_jours = (maintenant - timestamp) / (24 * 3600)

            # Garder tous les backups récents
            if age_jours <= garder_jours:
                a_conserver.add(chemin)

            # Garder un backup par semaine pour les backups anciens
            elif age_jours <= garder_hebdo * 7:
                semaine = int(age_jours // 7)
                if not any(int((maintenant - t) / (24 * 3600)) // 7 == semaine for p, t in fichiers_backup if p in a_conserver):
                    a_conserver.add(chemin)

            # Garder un backup par mois pour les très anciens
            elif age_jours <= garder_mensuel * 30:
                mois = int(age_jours // 30)
                if not any(int((maintenant - t) / (24 * 3600)) // 30 == mois for p, t in fichiers_backup if p in a_conserver):
                    a_conserver.add(chemin)

        # Supprimer les fichiers non conservés
        supprimes = 0
        for chemin, _ in fichiers_backup:
            if chemin not in a_conserver:
                os.remove(chemin)
                # Supprimer aussi le fichier de métadonnées
                metadata_file = chemin + '.info'
                if os.path.exists(metadata_file):
                    os.remove(metadata_file)
                supprimes += 1

        self.logger.info(f"✅ Rotation terminée : {len(a_conserver)} conservés, {supprimes} supprimés")

    def lister_backups(self):
        """Liste tous les backups disponibles"""
        backups = []

        for fichier in os.listdir(self.repertoire_backup):
            if fichier.endswith('.db.gz') and 'backup_' in fichier:
                chemin_complet = os.path.join(self.repertoire_backup, fichier)
                timestamp = os.path.getctime(chemin_complet)
                taille = os.path.getsize(chemin_complet)

                backups.append({
                    'fichier': fichier,
                    'chemin': chemin_complet,
                    'date': datetime.fromtimestamp(timestamp),
                    'taille_mb': taille / (1024 * 1024)
                })

        # Trier par date (plus récent en premier)
        backups.sort(key=lambda x: x['date'], reverse=True)

        return backups

# Utilisation du système de sauvegarde
if __name__ == "__main__":
    # Créer une base de test
    conn = sqlite3.connect('ma_base_test.db')
    conn.execute('''
        CREATE TABLE IF NOT EXISTS clients (
            id INTEGER PRIMARY KEY,
            nom TEXT,
            email TEXT,
            date_creation DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')

    # Ajouter quelques données
    for i in range(1000):
        conn.execute("INSERT INTO clients (nom, email) VALUES (?, ?)",
                    (f"Client {i}", f"client{i}@email.com"))

    conn.commit()
    conn.close()

    # Effectuer une sauvegarde
    backup_manager = GestionnaireBackup('ma_base_test.db')

    try:
        fichier_backup = backup_manager.sauvegarder(comprimer=True, verifier=True)
        print(f"\n📁 Sauvegarde créée : {fichier_backup}")

        # Lister les backups
        print("\n📋 Backups disponibles :")
        for backup in backup_manager.lister_backups():
            print(f"  • {backup['fichier']} - {backup['date'].strftime('%Y-%m-%d %H:%M')} - {backup['taille_mb']:.2f} MB")

        # Effectuer la rotation
        backup_manager.rotation_backups()

    except Exception as e:
        print(f"Erreur : {e}")
```

## Restauration de sauvegardes

### Restauration simple

```python
import sqlite3
import gzip
import shutil
import os

def restaurer_backup(chemin_backup, base_destination):
    """Restaure une sauvegarde vers une nouvelle base"""

    print(f"🔄 Restauration en cours : {chemin_backup} → {base_destination}")

    try:
        # Vérifier que le fichier de sauvegarde existe
        if not os.path.exists(chemin_backup):
            raise FileNotFoundError(f"Fichier de sauvegarde introuvable : {chemin_backup}")

        # Supprimer la base de destination si elle existe
        if os.path.exists(base_destination):
            print(f"⚠️ Suppression de la base existante : {base_destination}")
            os.remove(base_destination)

        # Décompresser si nécessaire
        if chemin_backup.endswith('.gz'):
            print("📦 Décompression...")
            chemin_temp = chemin_backup[:-3]  # Enlever .gz

            with gzip.open(chemin_backup, 'rb') as f_in:
                with open(chemin_temp, 'wb') as f_out:
                    shutil.copyfileobj(f_in, f_out)

            # Copier vers la destination finale
            shutil.move(chemin_temp, base_destination)
        else:
            # Copie directe
            shutil.copy2(chemin_backup, base_destination)

        # Vérifier l'intégrité de la base restaurée
        conn = sqlite3.connect(base_destination)
        try:
            result = conn.execute("PRAGMA integrity_check").fetchone()
            if result[0] != "ok":
                raise Exception(f"Base restaurée corrompue : {result[0]}")

            print("✅ Restauration réussie et intégrité vérifiée")

        finally:
            conn.close()

    except Exception as e:
        print(f"❌ Erreur de restauration : {e}")
        # Nettoyer en cas d'erreur
        if os.path.exists(base_destination):
            os.remove(base_destination)
        raise

# Utilisation
restaurer_backup('backups/ma_base_backup_20240115_143022.db.gz', 'ma_base_restauree.db')
```

### Système de restauration avec choix interactif

```python
def restauration_interactive(backup_manager):
    """Interface interactive pour choisir une sauvegarde à restaurer"""

    backups = backup_manager.lister_backups()

    if not backups:
        print("❌ Aucune sauvegarde disponible")
        return

    print("\n📋 Sauvegardes disponibles :")
    print("=" * 60)

    for i, backup in enumerate(backups, 1):
        print(f"{i:2d}. {backup['fichier']}")
        print(f"    📅 Date : {backup['date'].strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"    📊 Taille : {backup['taille_mb']:.2f} MB")
        print()

    while True:
        try:
            choix = input("Choisissez une sauvegarde (numéro) ou 'q' pour quitter : ")

            if choix.lower() == 'q':
                return

            index = int(choix) - 1
            if 0 <= index < len(backups):
                backup_choisi = backups[index]
                break
            else:
                print("❌ Numéro invalide")

        except ValueError:
            print("❌ Veuillez entrer un numéro valide")

    # Demander confirmation
    print(f"\n⚠️ Vous allez restaurer : {backup_choisi['fichier']}")
    confirmation = input("Êtes-vous sûr ? (oui/non) : ")

    if confirmation.lower() in ['oui', 'o', 'yes', 'y']:
        nom_base = input("Nom de la base restaurée (sans .db) : ")
        if not nom_base.endswith('.db'):
            nom_base += '.db'

        try:
            restaurer_backup(backup_choisi['chemin'], nom_base)
            print(f"✅ Base restaurée avec succès : {nom_base}")
        except Exception as e:
            print(f"❌ Erreur lors de la restauration : {e}")
    else:
        print("❌ Restauration annulée")

# Utilisation
backup_manager = GestionnaireBackup('ma_base.db')
restauration_interactive(backup_manager)
```

## Sauvegarde automatisée avec planification

### Script de sauvegarde quotidienne

```python
import schedule
import time
import threading
from datetime import datetime

class SauvegardeAutomatique:
    def __init__(self, db_path, repertoire_backup="./backups"):
        self.backup_manager = GestionnaireBackup(db_path, repertoire_backup)
        self.en_cours = False

    def sauvegarde_quotidienne(self):
        """Effectue une sauvegarde quotidienne"""
        if self.en_cours:
            print("⏳ Sauvegarde déjà en cours, ignorée")
            return

        self.en_cours = True
        try:
            print(f"🌅 Début de sauvegarde quotidienne - {datetime.now()}")

            # Sauvegarde avec compression
            self.backup_manager.sauvegarder(comprimer=True, verifier=True)

            # Rotation des anciens backups
            self.backup_manager.rotation_backups(
                garder_jours=7,     # Garder 7 jours de backups quotidiens
                garder_hebdo=4,     # Garder 4 semaines de backups hebdomadaires
                garder_mensuel=12   # Garder 12 mois de backups mensuels
            )

            print("✅ Sauvegarde quotidienne terminée")

        except Exception as e:
            print(f"❌ Erreur sauvegarde quotidienne : {e}")
        finally:
            self.en_cours = False

    def demarrer_planification(self):
        """Démarre la planification automatique"""
        # Programmer la sauvegarde tous les jours à 2h du matin
        schedule.every().day.at("02:00").do(self.sauvegarde_quotidienne)

        # Programmer une rotation hebdomadaire le dimanche
        schedule.every().sunday.at("01:00").do(self.backup_manager.rotation_backups)

        print("⏰ Planification activée :")
        print("  • Sauvegarde quotidienne : 02:00")
        print("  • Rotation hebdomadaire : Dimanche 01:00")

        # Boucle d'exécution en arrière-plan
        def executer_planification():
            while True:
                schedule.run_pending()
                time.sleep(60)  # Vérifier toutes les minutes

        # Démarrer dans un thread séparé
        thread = threading.Thread(target=executer_planification, daemon=True)
        thread.start()

        return thread

# Utilisation
if __name__ == "__main__":
    sauvegarde_auto = SauvegardeAutomatique('ma_base.db')

    # Démarrer la planification
    thread_planification = sauvegarde_auto.demarrer_planification()

    # Effectuer une sauvegarde immédiate pour test
    sauvegarde_auto.sauvegarde_quotidienne()

    print("Appuyez sur Ctrl+C pour arrêter...")
    try:
        # Garder le programme en vie
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\n👋 Arrêt de la planification")
```

## Sauvegarde vers le cloud

### Intégration avec des services cloud

```python
import boto3
import os
from datetime import datetime

class SauvegardeCloud:
    def __init__(self, backup_manager, config_aws=None):
        self.backup_manager = backup_manager

        # Configuration AWS S3 (exemple)
        if config_aws:
            self.s3_client = boto3.client(
                's3',
                aws_access_key_id=config_aws['access_key'],
                aws_secret_access_key=config_aws['secret_key'],
                region_name=config_aws.get('region', 'eu-west-1')
            )
            self.bucket_name = config_aws['bucket']
        else:
            self.s3_client = None

    def sauvegarder_vers_cloud(self, vers_s3=True):
        """Sauvegarde locale puis upload vers le cloud"""

        # 1. Effectuer la sauvegarde locale
        print("🔄 Sauvegarde locale...")
        fichier_local = self.backup_manager.sauvegarder(comprimer=True)

        if not vers_s3 or not self.s3_client:
            return fichier_local

        # 2. Upload vers S3
        print("☁️ Upload vers S3...")
        try:
            nom_fichier = os.path.basename(fichier_local)
            cle_s3 = f"backups/sqlite/{datetime.now().year}/{nom_fichier}"

            self.s3_client.upload_file(
                fichier_local,
                self.bucket_name,
                cle_s3,
                ExtraArgs={
                    'StorageClass': 'STANDARD_IA',  # Stockage peu fréquent
                    'ServerSideEncryption': 'AES256'  # Chiffrement
                }
            )

            print(f"✅ Upload S3 réussi : s3://{self.bucket_name}/{cle_s3}")

            # Optionnel : supprimer le fichier local après upload
            # os.remove(fichier_local)

            return cle_s3

        except Exception as e:
            print(f"❌ Erreur upload S3 : {e}")
            raise

    def restaurer_depuis_cloud(self, cle_s3, fichier_local):
        """Télécharge et restaure une sauvegarde depuis S3"""

        if not self.s3_client:
            raise Exception("Client S3 non configuré")

        try:
            # Télécharger depuis S3
            print(f"📥 Téléchargement depuis S3 : {cle_s3}")
            self.s3_client.download_file(self.bucket_name, cle_s3, fichier_local)

            # Restaurer la base
            base_restauree = fichier_local.replace('.gz', '_restaure.db')
            restaurer_backup(fichier_local, base_restauree)

            print(f"✅ Restauration depuis cloud réussie : {base_restauree}")
            return base_restauree

        except Exception as e:
            print(f"❌ Erreur restauration cloud : {e}")
            raise

# Configuration exemple
config_aws = {
    'access_key': 'VOTRE_ACCESS_KEY',
    'secret_key': 'VOTRE_SECRET_KEY',
    'region': 'eu-west-1',
    'bucket': 'mon-bucket-backups'
}

# Utilisation
backup_manager = GestionnaireBackup('ma_base.db')
backup_cloud = SauvegardeCloud(backup_manager, config_aws)

# Sauvegarde avec upload cloud
backup_cloud.sauvegarder_vers_cloud(vers_s3=True)
```

## Surveillance et alertes

### Système d'alertes par email

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class AlertesBackup:
    def __init__(self, config_email):
        self.config = config_email

    def envoyer_alerte(self, sujet, message, type_alerte="info"):
        """Envoie une alerte par email"""

        emoji_map = {
            "success": "✅",
            "warning": "⚠️",
            "error": "❌",
            "info": "ℹ️"
        }

        try:
            msg = MIMEMultipart()
            msg['From'] = self.config['from']
            msg['To'] = self.config['to']
            msg['Subject'] = f"{emoji_map.get(type_alerte, '')} {sujet}"

            corps = f"""
            <html>
            <body>
                <h2>{sujet}</h2>
                <p>{message}</p>
                <hr>
                <p><small>Alerte automatique du système de sauvegarde SQLite</small></p>
            </body>
            </html>
            """

            msg.attach(MIMEText(corps, 'html'))

            # Connexion SMTP
            with smtplib.SMTP(self.config['smtp_server'], self.config['smtp_port']) as server:
                server.starttls()
                server.login(self.config['username'], self.config['password'])
                server.send_message(msg)

            print(f"📧 Alerte envoyée : {sujet}")

        except Exception as e:
            print(f"❌ Erreur envoi email : {e}")

class GestionnaireBackupAvecAlertes(GestionnaireBackup):
    def __init__(self, db_path, repertoire_backup="./backups", config_alertes=None):
        super().__init__(db_path, repertoire_backup)
        self.alertes = AlertesBackup(config_alertes) if config_alertes else None

    def sauvegarder(self, **kwargs):
        """Sauvegarde avec alertes automatiques"""
        try:
            debut = time.time()
            fichier_backup = super().sauvegarder(**kwargs)
            duree = time.time() - debut

            # Alerte de succès
            if self.alertes:
                taille = os.path.getsize(fichier_backup) / (1024 * 1024)  # MB
                message = f"""
                Sauvegarde réussie de la base de données !

                📁 Base source : {self.db_path}
                💾 Fichier backup : {os.path.basename(fichier_backup)}
                ⏱️ Durée : {duree:.2f} secondes
                📊 Taille : {taille:.2f} MB
                📅 Date : {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
                """

                self.alertes.envoyer_alerte(
                    "Sauvegarde SQLite réussie",
                    message,
                    "success"
                )

            return fichier_backup

        except Exception as e:
            # Alerte d'erreur
            if self.alertes:
                message = f"""
                ERREUR lors de la sauvegarde !

                📁 Base source : {self.db_path}
                ❌ Erreur : {str(e)}
                📅 Date : {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

                Action requise : Vérifier la base de données et l'espace disque.
                """

                self.alertes.envoyer_alerte(
                    "ERREUR Sauvegarde SQLite",
                    message,
                    "error"
                )

            raise  # Re-lancer l'exception

    def verifier_sante_backups(self):
        """Vérifie la santé du système de sauvegarde"""
        rapport = {
            'total_backups': 0,
            'dernier_backup': None,
            'espace_utilise_mb': 0,
            'backups_corrompus': [],
            'alertes': []
        }

        try:
            backups = self.lister_backups()
            rapport['total_backups'] = len(backups)

            if backups:
                rapport['dernier_backup'] = backups[0]['date']
                rapport['espace_utilise_mb'] = sum(b['taille_mb'] for b in backups)

                # Vérifier l'âge du dernier backup
                age_heures = (datetime.now() - backups[0]['date']).total_seconds() / 3600

                if age_heures > 48:  # Plus de 48h
                    rapport['alertes'].append({
                        'type': 'warning',
                        'message': f"Dernier backup vieux de {age_heures:.1f} heures"
                    })

                # Vérifier l'intégrité des 3 derniers backups
                for backup in backups[:3]:
                    try:
                        if backup['chemin'].endswith('.gz'):
                            # Tester la décompression
                            with gzip.open(backup['chemin'], 'rb') as f:
                                f.read(1024)  # Lire un petit bloc
                        else:
                            # Tester l'ouverture SQLite
                            conn = sqlite3.connect(backup['chemin'])
                            conn.execute("PRAGMA integrity_check").fetchone()
                            conn.close()
                    except Exception as e:
                        rapport['backups_corrompus'].append({
                            'fichier': backup['fichier'],
                            'erreur': str(e)
                        })
            else:
                rapport['alertes'].append({
                    'type': 'error',
                    'message': "Aucun backup trouvé !"
                })

            # Envoyer un rapport de santé si nécessaire
            if self.alertes and (rapport['alertes'] or rapport['backups_corrompus']):
                self._envoyer_rapport_sante(rapport)

            return rapport

        except Exception as e:
            self.logger.error(f"Erreur lors de la vérification de santé : {e}")
            return rapport

    def _envoyer_rapport_sante(self, rapport):
        """Envoie un rapport de santé par email"""

        # Déterminer le niveau d'alerte
        type_alerte = "info"
        if rapport['backups_corrompus']:
            type_alerte = "error"
        elif any(a['type'] == 'warning' for a in rapport['alertes']):
            type_alerte = "warning"

        message = f"""
        Rapport de santé du système de sauvegarde

        📊 Statistiques générales :
        • Total de backups : {rapport['total_backups']}
        • Espace utilisé : {rapport['espace_utilise_mb']:.2f} MB
        """

        if rapport['dernier_backup']:
            message += f"• Dernier backup : {rapport['dernier_backup'].strftime('%Y-%m-%d %H:%M:%S')}\n"

        if rapport['alertes']:
            message += "\n⚠️ Alertes :\n"
            for alerte in rapport['alertes']:
                message += f"• {alerte['message']}\n"

        if rapport['backups_corrompus']:
            message += "\n❌ Backups corrompus détectés :\n"
            for backup in rapport['backups_corrompus']:
                message += f"• {backup['fichier']} : {backup['erreur']}\n"

        self.alertes.envoyer_alerte(
            "Rapport de santé - Système de sauvegarde",
            message,
            type_alerte
        )

# Configuration des alertes email
config_email = {
    'smtp_server': 'smtp.gmail.com',
    'smtp_port': 587,
    'username': 'votre.email@gmail.com',
    'password': 'votre_mot_de_passe_app',  # Mot de passe d'application
    'from': 'votre.email@gmail.com',
    'to': 'admin@entreprise.com'
}

# Utilisation avec alertes
backup_manager = GestionnaireBackupAvecAlertes(
    'ma_base.db',
    './backups',
    config_email
)

# Test du système
try:
    backup_manager.sauvegarder()
    rapport = backup_manager.verifier_sante_backups()
    print("Rapport de santé :", rapport)
except Exception as e:
    print(f"Erreur : {e}")
```

## Monitoring avancé avec métriques

### Système de métriques détaillées

```python
import json
import sqlite3
from collections import defaultdict
from datetime import datetime, timedelta

class MetriquesBackup:
    def __init__(self, fichier_metriques="backup_metrics.db"):
        self.fichier_metriques = fichier_metriques
        self._initialiser_base_metriques()

    def _initialiser_base_metriques(self):
        """Initialise la base de données des métriques"""
        conn = sqlite3.connect(self.fichier_metriques)

        conn.execute('''
            CREATE TABLE IF NOT EXISTS historique_backups (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp DATETIME,
                base_source TEXT,
                taille_source_mb REAL,
                taille_backup_mb REAL,
                duree_seconde REAL,
                compression_ratio REAL,
                succes BOOLEAN,
                erreur TEXT,
                type_backup TEXT
            )
        ''')

        conn.execute('''
            CREATE TABLE IF NOT EXISTS metriques_quotidiennes (
                date DATE PRIMARY KEY,
                nb_backups INTEGER,
                nb_echecs INTEGER,
                duree_moyenne REAL,
                taille_moyenne_mb REAL,
                espace_total_mb REAL
            )
        ''')

        conn.commit()
        conn.close()

    def enregistrer_backup(self, infos_backup):
        """Enregistre les informations d'un backup"""
        conn = sqlite3.connect(self.fichier_metriques)

        try:
            conn.execute('''
                INSERT INTO historique_backups (
                    timestamp, base_source, taille_source_mb, taille_backup_mb,
                    duree_seconde, compression_ratio, succes, erreur, type_backup
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                datetime.now(),
                infos_backup['base_source'],
                infos_backup.get('taille_source_mb', 0),
                infos_backup.get('taille_backup_mb', 0),
                infos_backup.get('duree_seconde', 0),
                infos_backup.get('compression_ratio', 0),
                infos_backup.get('succes', True),
                infos_backup.get('erreur', None),
                infos_backup.get('type_backup', 'manuel')
            ))

            conn.commit()

        finally:
            conn.close()

    def calculer_metriques_quotidiennes(self, date=None):
        """Calcule les métriques pour une date donnée"""
        if date is None:
            date = datetime.now().date()

        conn = sqlite3.connect(self.fichier_metriques)

        try:
            # Récupérer les données du jour
            cursor = conn.execute('''
                SELECT COUNT(*) as nb_backups,
                       SUM(CASE WHEN NOT succes THEN 1 ELSE 0 END) as nb_echecs,
                       AVG(duree_seconde) as duree_moyenne,
                       AVG(taille_backup_mb) as taille_moyenne_mb,
                       SUM(taille_backup_mb) as espace_total_mb
                FROM historique_backups
                WHERE DATE(timestamp) = ?
            ''', (date,))

            result = cursor.fetchone()

            # Insérer ou mettre à jour les métriques quotidiennes
            conn.execute('''
                INSERT OR REPLACE INTO metriques_quotidiennes
                (date, nb_backups, nb_echecs, duree_moyenne, taille_moyenne_mb, espace_total_mb)
                VALUES (?, ?, ?, ?, ?, ?)
            ''', result)

            conn.commit()

            return {
                'date': date,
                'nb_backups': result[0] or 0,
                'nb_echecs': result[1] or 0,
                'duree_moyenne': result[2] or 0,
                'taille_moyenne_mb': result[3] or 0,
                'espace_total_mb': result[4] or 0,
                'taux_reussite': ((result[0] - result[1]) / result[0] * 100) if result[0] > 0 else 0
            }

        finally:
            conn.close()

    def generer_rapport_hebdomadaire(self):
        """Génère un rapport hebdomadaire détaillé"""
        conn = sqlite3.connect(self.fichier_metriques)

        try:
            # Dernière semaine
            date_fin = datetime.now().date()
            date_debut = date_fin - timedelta(days=7)

            # Métriques globales
            cursor = conn.execute('''
                SELECT COUNT(*) as total_backups,
                       SUM(CASE WHEN NOT succes THEN 1 ELSE 0 END) as total_echecs,
                       AVG(duree_seconde) as duree_moyenne,
                       MAX(duree_seconde) as duree_max,
                       MIN(duree_seconde) as duree_min,
                       AVG(compression_ratio) as compression_moyenne,
                       SUM(taille_backup_mb) as espace_total_mb
                FROM historique_backups
                WHERE DATE(timestamp) BETWEEN ? AND ?
            ''', (date_debut, date_fin))

            globales = cursor.fetchone()

            # Tendances par jour
            cursor = conn.execute('''
                SELECT DATE(timestamp) as jour,
                       COUNT(*) as nb_backups,
                       AVG(duree_seconde) as duree_moyenne,
                       SUM(taille_backup_mb) as taille_totale
                FROM historique_backups
                WHERE DATE(timestamp) BETWEEN ? AND ?
                GROUP BY DATE(timestamp)
                ORDER BY jour
            ''', (date_debut, date_fin))

            tendances = cursor.fetchall()

            # Erreurs fréquentes
            cursor = conn.execute('''
                SELECT erreur, COUNT(*) as nb_occurrences
                FROM historique_backups
                WHERE DATE(timestamp) BETWEEN ? AND ? AND erreur IS NOT NULL
                GROUP BY erreur
                ORDER BY nb_occurrences DESC
                LIMIT 5
            ''', (date_debut, date_fin))

            erreurs = cursor.fetchall()

            rapport = {
                'periode': f"{date_debut} à {date_fin}",
                'globales': {
                    'total_backups': globales[0] or 0,
                    'total_echecs': globales[1] or 0,
                    'taux_reussite': ((globales[0] - globales[1]) / globales[0] * 100) if globales[0] > 0 else 0,
                    'duree_moyenne': globales[2] or 0,
                    'duree_max': globales[3] or 0,
                    'duree_min': globales[4] or 0,
                    'compression_moyenne': globales[5] or 0,
                    'espace_total_mb': globales[6] or 0
                },
                'tendances_quotidiennes': [
                    {
                        'jour': t[0],
                        'nb_backups': t[1],
                        'duree_moyenne': t[2],
                        'taille_totale': t[3]
                    } for t in tendances
                ],
                'erreurs_frequentes': [
                    {
                        'erreur': e[0],
                        'occurrences': e[1]
                    } for e in erreurs
                ]
            }

            return rapport

        finally:
            conn.close()

class GestionnaireBackupAvecMetriques(GestionnaireBackupAvecAlertes):
    def __init__(self, db_path, repertoire_backup="./backups", config_alertes=None):
        super().__init__(db_path, repertoire_backup, config_alertes)
        self.metriques = MetriquesBackup()

    def sauvegarder(self, **kwargs):
        """Sauvegarde avec collecte de métriques"""
        debut = time.time()
        infos_backup = {
            'base_source': self.db_path,
            'type_backup': kwargs.get('type_backup', 'manuel')
        }

        try:
            # Taille de la base source
            if os.path.exists(self.db_path):
                infos_backup['taille_source_mb'] = os.path.getsize(self.db_path) / (1024 * 1024)

            # Effectuer la sauvegarde
            fichier_backup = super().sauvegarder(**kwargs)

            # Calculer les métriques
            duree = time.time() - debut
            taille_backup = os.path.getsize(fichier_backup) / (1024 * 1024)

            infos_backup.update({
                'duree_seconde': duree,
                'taille_backup_mb': taille_backup,
                'compression_ratio': infos_backup.get('taille_source_mb', 0) / taille_backup if taille_backup > 0 else 0,
                'succes': True
            })

            # Enregistrer les métriques
            self.metriques.enregistrer_backup(infos_backup)

            return fichier_backup

        except Exception as e:
            # Enregistrer l'échec
            infos_backup.update({
                'duree_seconde': time.time() - debut,
                'succes': False,
                'erreur': str(e)
            })

            self.metriques.enregistrer_backup(infos_backup)
            raise

    def rapport_performance(self, jours=30):
        """Génère un rapport de performance détaillé"""
        date_fin = datetime.now().date()
        date_debut = date_fin - timedelta(days=jours)

        conn = sqlite3.connect(self.metriques.fichier_metriques)

        try:
            # Performance globale
            cursor = conn.execute('''
                SELECT
                    COUNT(*) as total_backups,
                    AVG(duree_seconde) as duree_moyenne,
                    AVG(taille_backup_mb) as taille_moyenne,
                    AVG(compression_ratio) as compression_moyenne,
                    SUM(CASE WHEN NOT succes THEN 1 ELSE 0 END) as nb_echecs
                FROM historique_backups
                WHERE DATE(timestamp) BETWEEN ? AND ?
            ''', (date_debut, date_fin))

            perf_globale = cursor.fetchone()

            # Évolution dans le temps
            cursor = conn.execute('''
                SELECT
                    DATE(timestamp) as jour,
                    AVG(duree_seconde) as duree_moy,
                    AVG(taille_backup_mb) as taille_moy,
                    COUNT(*) as nb_backups
                FROM historique_backups
                WHERE DATE(timestamp) BETWEEN ? AND ?
                GROUP BY DATE(timestamp)
                ORDER BY jour DESC
                LIMIT 10
            ''', (date_debut, date_fin))

            evolution = cursor.fetchall()

            # Recommandations basées sur les données
            recommandations = []

            if perf_globale[1] and perf_globale[1] > 300:  # Plus de 5 minutes
                recommandations.append("⚠️ Durée de sauvegarde élevée - considérer l'optimisation")

            if perf_globale[3] and perf_globale[3] < 2:  # Compression faible
                recommandations.append("💡 Compression faible - vérifier les paramètres de compression")

            if perf_globale[4] and perf_globale[4] > perf_globale[0] * 0.1:  # Plus de 10% d'échecs
                recommandations.append("🚨 Taux d'échec élevé - investigation requise")

            rapport = {
                'periode': f"{date_debut} à {date_fin}",
                'performance_globale': {
                    'total_backups': perf_globale[0] or 0,
                    'duree_moyenne_sec': perf_globale[1] or 0,
                    'taille_moyenne_mb': perf_globale[2] or 0,
                    'compression_moyenne': perf_globale[3] or 0,
                    'taux_echec_pct': (perf_globale[4] / perf_globale[0] * 100) if perf_globale[0] > 0 else 0
                },
                'evolution_recente': [
                    {
                        'jour': e[0],
                        'duree_moy': e[1],
                        'taille_moy': e[2],
                        'nb_backups': e[3]
                    } for e in evolution
                ],
                'recommandations': recommandations
            }

            return rapport

        finally:
            conn.close()

# Utilisation du système avec métriques
if __name__ == "__main__":
    # Créer une base de test
    conn = sqlite3.connect('test_metriques.db')
    conn.execute('''
        CREATE TABLE IF NOT EXISTS test_data (
            id INTEGER PRIMARY KEY,
            data TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')

    # Ajouter des données
    for i in range(5000):
        conn.execute("INSERT INTO test_data (data) VALUES (?)", (f"Test data {i}" * 10,))

    conn.commit()
    conn.close()

    # Tester le système avec métriques
    backup_manager = GestionnaireBackupAvecMetriques('test_metriques.db')

    # Effectuer plusieurs sauvegardes pour générer des métriques
    for i in range(3):
        try:
            print(f"\n🔄 Sauvegarde #{i+1}")
            backup_manager.sauvegarder(type_backup='test_automatique')
            time.sleep(1)  # Pause entre les sauvegardes
        except Exception as e:
            print(f"Erreur sauvegarde #{i+1}: {e}")

    # Générer un rapport de performance
    print("\n📊 Rapport de performance :")
    rapport = backup_manager.rapport_performance(jours=1)

    print(f"Période : {rapport['periode']}")
    print(f"Total backups : {rapport['performance_globale']['total_backups']}")
    print(f"Durée moyenne : {rapport['performance_globale']['duree_moyenne_sec']:.2f}s")
    print(f"Taille moyenne : {rapport['performance_globale']['taille_moyenne_mb']:.2f} MB")
    print(f"Compression moyenne : {rapport['performance_globale']['compression_moyenne']:.2f}x")
    print(f"Taux d'échec : {rapport['performance_globale']['taux_echec_pct']:.1f}%")

    if rapport['recommandations']:
        print("\n📋 Recommandations :")
        for rec in rapport['recommandations']:
            print(f"  {rec}")

    # Générer métriques quotidiennes
    metriques_jour = backup_manager.metriques.calculer_metriques_quotidiennes()
    print(f"\n📅 Métriques du jour :")
    print(f"  Backups : {metriques_jour['nb_backups']}")
    print(f"  Échecs : {metriques_jour['nb_echecs']}")
    print(f"  Taux de réussite : {metriques_jour['taux_reussite']:.1f}%")
```

## Dashboard web simple

### Interface web basique pour monitoring

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import urllib.parse

class DashboardBackup(BaseHTTPRequestHandler):
    def __init__(self, backup_manager, *args, **kwargs):
        self.backup_manager = backup_manager
        super().__init__(*args, **kwargs)

    def do_GET(self):
        """Gère les requêtes GET"""

        if self.path == '/':
            self._serve_dashboard()
        elif self.path == '/api/stats':
            self._serve_stats()
        elif self.path == '/api/backups':
            self._serve_backups_list()
        else:
            self._serve_404()

    def _serve_dashboard(self):
        """Page principale du dashboard"""
        html = '''
        <!DOCTYPE html>
        <html>
        <head>
            <title>Dashboard Sauvegarde SQLite</title>
            <style>
                body { font-family: Arial, sans-serif; margin: 20px; }
                .card { background: #f5f5f5; padding: 20px; margin: 10px 0; border-radius: 5px; }
                .metric { display: inline-block; margin: 10px 20px 10px 0; }
                .metric-value { font-size: 2em; font-weight: bold; color: #2196F3; }
                .metric-label { color: #666; }
                .status-ok { color: #4CAF50; }
                .status-warning { color: #FF9800; }
                .status-error { color: #F44336; }
                table { width: 100%; border-collapse: collapse; }
                th, td { padding: 10px; text-align: left; border-bottom: 1px solid #ddd; }
                th { background-color: #f2f2f2; }
            </style>
        </head>
        <body>
            <h1>🔧 Dashboard Sauvegarde SQLite</h1>

            <div class="card">
                <h2>📊 Statistiques en temps réel</h2>
                <div id="stats">Chargement...</div>
            </div>

            <div class="card">
                <h2>📁 Dernières sauvegardes</h2>
                <div id="backups">Chargement...</div>
            </div>

            <script>
                function loadStats() {
                    fetch('/api/stats')
                        .then(response => response.json())
                        .then(data => {
                            document.getElementById('stats').innerHTML = `
                                <div class="metric">
                                    <div class="metric-value">${data.total_backups}</div>
                                    <div class="metric-label">Total Backups</div>
                                </div>
                                <div class="metric">
                                    <div class="metric-value ${data.taux_reussite >= 95 ? 'status-ok' : data.taux_reussite >= 80 ? 'status-warning' : 'status-error'}">${data.taux_reussite.toFixed(1)}%</div>
                                    <div class="metric-label">Taux de réussite</div>
                                </div>
                                <div class="metric">
                                    <div class="metric-value">${data.espace_total_mb.toFixed(1)} MB</div>
                                    <div class="metric-label">Espace utilisé</div>
                                </div>
                                <div class="metric">
                                    <div class="metric-value">${data.duree_moyenne.toFixed(1)}s</div>
                                    <div class="metric-label">Durée moyenne</div>
                                </div>
                            `;
                        });
                }

                function loadBackups() {
                    fetch('/api/backups')
                        .then(response => response.json())
                        .then(data => {
                            let html = '<table><tr><th>Fichier</th><th>Date</th><th>Taille</th><th>Status</th></tr>';
                            data.forEach(backup => {
                                html += `<tr>
                                    <td>${backup.fichier}</td>
                                    <td>${backup.date}</td>
                                    <td>${backup.taille_mb.toFixed(2)} MB</td>
                                    <td><span class="status-ok">✅ OK</span></td>
                                </tr>`;
                            });
                            html += '</table>';
                            document.getElementById('backups').innerHTML = html;
                        });
                }

                // Charger les données au démarrage
                loadStats();
                loadBackups();

                // Actualiser toutes les 30 secondes
                setInterval(() => {
                    loadStats();
                    loadBackups();
                }, 30000);
            </script>
        </body>
        </html>
        '''

        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(html.encode())

    def _serve_stats(self):
        """API des statistiques"""
        try:
            rapport = self.backup_manager.rapport_performance(jours=7)
            stats = rapport['performance_globale']

            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps(stats).encode())

        except Exception as e:
            self._serve_error(str(e))

    def _serve_backups_list(self):
        """API de la liste des backups"""
        try:
            backups = self.backup_manager.lister_backups()

            # Formater pour l'API
            backups_formatted = []
            for backup in backups[:10]:  # Derniers 10
                backups_formatted.append({
                    'fichier': backup['fichier'],
                    'date': backup['date'].strftime('%Y-%m-%d %H:%M:%S'),
                    'taille_mb': backup['taille_mb']
                })

            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps(backups_formatted).encode())

        except Exception as e:
            self._serve_error(str(e))

    def _serve_404(self):
        """Page 404"""
        self.send_response(404)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(b'<h1>404 - Page non trouvee</h1>')

    def _serve_error(self, error_msg):
        """Erreur API"""
        self.send_response(500)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        self.wfile.write(json.dumps({'error': error_msg}).encode())

def demarrer_dashboard(backup_manager, port=8080):
    """Démarre le dashboard web"""

    def handler(*args, **kwargs):
        DashboardBackup(backup_manager, *args, **kwargs)

    server = HTTPServer(('localhost', port), handler)
    print(f"🌐 Dashboard démarré sur http://localhost:{port}")
    print("Appuyez sur Ctrl+C pour arrêter")

    try:
        server.serve_forever()
    except KeyboardInterrupt:
        print("\n👋 Dashboard arrêté")
        server.shutdown()

# Utilisation du dashboard
if __name__ == "__main__":
    backup_manager = GestionnaireBackupAvecMetriques('ma_base.db')
    demarrer_dashboard(backup_manager, port=8080)
```

## Script de sauvegarde production complet

### Script tout-en-un pour environnement de production

```python
#!/usr/bin/env python3
"""
Script de sauvegarde SQLite complet pour production
Inclut toutes les fonctionnalités vues dans ce tutoriel
"""

import argparse
import sys
import signal
import logging
from pathlib import Path

class BackupProduction:
    def __init__(self, config_file=None):
        self.config = self._charger_configuration(config_file)
        self.backup_manager = None
        self._configurer_logging()

    def _charger_configuration(self, config_file):
        """Charge la configuration depuis un fichier JSON"""
        config_defaut = {
            'base_de_donnees': 'ma_base.db',
            'repertoire_backup': './backups',
            'compression': True,
            'verification': True,
            'rotation': {
                'garder_jours': 7,
                'garder_hebdo': 4,
                'garder_mensuel': 12
            },
            'alertes': {
                'email_active': False,
                'smtp_server': 'smtp.gmail.com',
                'smtp_port': 587,
                'from': '',
                'to': '',
                'username': '',
                'password': ''
            },
            'cloud': {
                's3_active': False,
                'bucket': '',
                'access_key': '',
                'secret_key': '',
                'region': 'eu-west-1'
            },
            'planification': {
                'auto_backup': False,
                'heure_quotidienne': '02:00',
                'jour_hebdomadaire': 'dimanche'
            }
        }

        if config_file and Path(config_file).exists():
            try:
                with open(config_file, 'r') as f:
                    config_fichier = json.load(f)
                # Fusionner avec la config par défaut
                config_defaut.update(config_fichier)
            except Exception as e:
                print(f"⚠️ Erreur lecture config {config_file}: {e}")
                print("Utilisation de la configuration par défaut")

        return config_defaut

    def _configurer_logging(self):
        """Configure le système de logging"""
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler('backup.log'),
                logging.StreamHandler(sys.stdout)
            ]
        )
        self.logger = logging.getLogger(__name__)

    def initialiser(self):
        """Initialise le gestionnaire de backup"""
        config_alertes = None
        if self.config['alertes']['email_active']:
            config_alertes = self.config['alertes']

        self.backup_manager = GestionnaireBackupAvecMetriques(
            self.config['base_de_donnees'],
            self.config['repertoire_backup'],
            config_alertes
        )

        self.logger.info("✅ Système de sauvegarde initialisé")

    def sauvegarde_manuelle(self):
        """Effectue une sauvegarde manuelle"""
        try:
            self.logger.info("🚀 Début de sauvegarde manuelle")

            fichier_backup = self.backup_manager.sauvegarder(
                comprimer=self.config['compression'],
                verifier=self.config['verification'],
                type_backup='manuel'
            )

            self.logger.info(f"✅ Sauvegarde manuelle terminée : {fichier_backup}")
            return fichier_backup

        except Exception as e:
            self.logger.error(f"❌ Erreur sauvegarde manuelle : {e}")
            sys.exit(1)

    def rotation_backups(self):
        """Effectue la rotation des backups"""
        try:
            self.logger.info("🔄 Début de rotation des backups")

            rotation_config = self.config['rotation']
            self.backup_manager.rotation_backups(
                garder_jours=rotation_config['garder_jours'],
                garder_hebdo=rotation_config['garder_hebdo'],
                garder_mensuel=rotation_config['garder_mensuel']
            )

            self.logger.info("✅ Rotation des backups terminée")

        except Exception as e:
            self.logger.error(f"❌ Erreur rotation : {e}")

    def verifier_sante(self):
        """Vérifie la santé du système de sauvegarde"""
        try:
            self.logger.info("🔍 Vérification de la santé du système")

            rapport = self.backup_manager.verifier_sante_backups()

            print("\n📊 Rapport de santé :")
            print(f"  Total backups : {rapport['total_backups']}")

            if rapport['dernier_backup']:
                print(f"  Dernier backup : {rapport['dernier_backup']}")

            print(f"  Espace utilisé : {rapport['espace_utilise_mb']:.2f} MB")

            if rapport['alertes']:
                print("  ⚠️ Alertes :")
                for alerte in rapport['alertes']:
                    print(f"    • {alerte['message']}")

            if rapport['backups_corrompus']:
                print("  ❌ Backups corrompus :")
                for backup in rapport['backups_corrompus']:
                    print(f"    • {backup['fichier']}")

            if not rapport['alertes'] and not rapport['backups_corrompus']:
                print("  ✅ Système en bonne santé")

            return rapport

        except Exception as e:
            self.logger.error(f"❌ Erreur vérification santé : {e}")
            return None

    def lister_backups(self):
        """Liste tous les backups disponibles"""
        try:
            backups = self.backup_manager.lister_backups()

            if not backups:
                print("❌ Aucun backup trouvé")
                return

            print(f"\n📋 {len(backups)} backup(s) disponible(s) :")
            print("=" * 80)

            for i, backup in enumerate(backups, 1):
                print(f"{i:2d}. {backup['fichier']}")
                print(f"    📅 {backup['date'].strftime('%Y-%m-%d %H:%M:%S')}")
                print(f"    📊 {backup['taille_mb']:.2f} MB")
                print()

            return backups

        except Exception as e:
            self.logger.error(f"❌ Erreur listing backups : {e}")
            return []

    def restaurer_interactif(self):
        """Restauration interactive"""
        backups = self.lister_backups()
        if not backups:
            return

        while True:
            try:
                choix = input("Numéro du backup à restaurer (ou 'q' pour quitter) : ")

                if choix.lower() == 'q':
                    return

                index = int(choix) - 1
                if 0 <= index < len(backups):
                    backup_choisi = backups[index]
                    break
                else:
                    print("❌ Numéro invalide")

            except ValueError:
                print("❌ Veuillez entrer un numéro valide")

        # Confirmation
        print(f"\n⚠️ Restauration de : {backup_choisi['fichier']}")
        confirmation = input("Continuer ? (oui/non) : ")

        if confirmation.lower() not in ['oui', 'o', 'yes', 'y']:
            print("❌ Restauration annulée")
            return

        # Nom de la base restaurée
        nom_base = input("Nom de fichier pour la base restaurée : ")
        if not nom_base.endswith('.db'):
            nom_base += '.db'

        try:
            restaurer_backup(backup_choisi['chemin'], nom_base)
            print(f"✅ Restauration réussie : {nom_base}")
        except Exception as e:
            print(f"❌ Erreur restauration : {e}")

    def rapport_performance(self, jours=30):
        """Génère un rapport de performance"""
        try:
            rapport = self.backup_manager.rapport_performance(jours)

            print(f"\n📊 Rapport de performance ({rapport['periode']}) :")
            print("=" * 60)

            perf = rapport['performance_globale']
            print(f"Total backups      : {perf['total_backups']}")
            print(f"Durée moyenne      : {perf['duree_moyenne_sec']:.2f}s")
            print(f"Taille moyenne     : {perf['taille_moyenne_mb']:.2f} MB")
            print(f"Compression moy.   : {perf['compression_moyenne']:.2f}x")
            print(f"Taux d'échec       : {perf['taux_echec_pct']:.1f}%")

            if rapport['recommandations']:
                print("\n📋 Recommandations :")
                for rec in rapport['recommandations']:
                    print(f"  {rec}")

            return rapport

        except Exception as e:
            self.logger.error(f"❌ Erreur rapport performance : {e}")
            return None

    def demarrer_service(self):
        """Démarre le service de sauvegarde automatique"""
        if not self.config['planification']['auto_backup']:
            print("❌ Sauvegarde automatique désactivée dans la configuration")
            return

        sauvegarde_auto = SauvegardeAutomatique(
            self.config['base_de_donnees'],
            self.config['repertoire_backup']
        )

        # Démarrer la planification
        thread_planification = sauvegarde_auto.demarrer_planification()

        # Gestion propre de l'arrêt
        def signal_handler(signum, frame):
            print("\n🛑 Arrêt du service de sauvegarde...")
            sys.exit(0)

        signal.signal(signal.SIGINT, signal_handler)
        signal.signal(signal.SIGTERM, signal_handler)

        print("🚀 Service de sauvegarde automatique démarré")
        print("Appuyez sur Ctrl+C pour arrêter")

        try:
            # Garder le service en vie
            while True:
                time.sleep(1)
        except KeyboardInterrupt:
            print("\n👋 Service arrêté")

    def generer_config_exemple(self, fichier='backup_config.json'):
        """Génère un fichier de configuration exemple"""
        config_exemple = {
            "base_de_donnees": "ma_base.db",
            "repertoire_backup": "./backups",
            "compression": True,
            "verification": True,
            "rotation": {
                "garder_jours": 7,
                "garder_hebdo": 4,
                "garder_mensuel": 12
            },
            "alertes": {
                "email_active": False,
                "smtp_server": "smtp.gmail.com",
                "smtp_port": 587,
                "from": "backup@monentreprise.com",
                "to": "admin@monentreprise.com",
                "username": "backup@monentreprise.com",
                "password": "mot_de_passe_application"
            },
            "cloud": {
                "s3_active": False,
                "bucket": "mon-bucket-backups",
                "access_key": "AKIAIOSFODNN7EXAMPLE",
                "secret_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
                "region": "eu-west-1"
            },
            "planification": {
                "auto_backup": True,
                "heure_quotidienne": "02:00",
                "jour_hebdomadaire": "dimanche"
            }
        }

        try:
            with open(fichier, 'w') as f:
                json.dump(config_exemple, f, indent=4)

            print(f"✅ Configuration exemple créée : {fichier}")
            print("Modifiez ce fichier selon vos besoins")

        except Exception as e:
            print(f"❌ Erreur création config : {e}")

def main():
    """Point d'entrée principal du script"""
    parser = argparse.ArgumentParser(
        description="Système de sauvegarde SQLite avancé",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Exemples d'utilisation :
  %(prog)s backup                    # Sauvegarde manuelle
  %(prog)s rotate                    # Rotation des backups
  %(prog)s health                    # Vérification de santé
  %(prog)s list                      # Lister les backups
  %(prog)s restore                   # Restauration interactive
  %(prog)s report --days 30          # Rapport de performance
  %(prog)s service                   # Démarrer le service auto
  %(prog)s config                    # Générer config exemple
  %(prog)s dashboard --port 8080     # Démarrer le dashboard web
        """
    )

    parser.add_argument(
        'action',
        choices=['backup', 'rotate', 'health', 'list', 'restore', 'report', 'service', 'config', 'dashboard'],
        help='Action à effectuer'
    )

    parser.add_argument(
        '--config', '-c',
        help='Fichier de configuration JSON'
    )

    parser.add_argument(
        '--days', '-d',
        type=int,
        default=30,
        help='Nombre de jours pour le rapport (défaut: 30)'
    )

    parser.add_argument(
        '--port', '-p',
        type=int,
        default=8080,
        help='Port pour le dashboard web (défaut: 8080)'
    )

    args = parser.parse_args()

    # Initialiser le système
    backup_prod = BackupProduction(args.config)
    backup_prod.initialiser()

    # Exécuter l'action demandée
    try:
        if args.action == 'backup':
            backup_prod.sauvegarde_manuelle()

        elif args.action == 'rotate':
            backup_prod.rotation_backups()

        elif args.action == 'health':
            backup_prod.verifier_sante()

        elif args.action == 'list':
            backup_prod.lister_backups()

        elif args.action == 'restore':
            backup_prod.restaurer_interactif()

        elif args.action == 'report':
            backup_prod.rapport_performance(args.days)

        elif args.action == 'service':
            backup_prod.demarrer_service()

        elif args.action == 'config':
            backup_prod.generer_config_exemple()

        elif args.action == 'dashboard':
            demarrer_dashboard(backup_prod.backup_manager, args.port)

    except KeyboardInterrupt:
        print("\n👋 Opération interrompue")
        sys.exit(0)
    except Exception as e:
        print(f"❌ Erreur fatale : {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

## Guide d'installation et déploiement

### 1. Installation des dépendances

```bash
# Créer un environnement virtuel
python3 -m venv backup_env
source backup_env/bin/activate  # Linux/Mac
# ou
backup_env\Scripts\activate     # Windows

# Installer les dépendances
pip install schedule boto3
```

### 2. Configuration système

```bash
# Créer les répertoires
mkdir -p /opt/sqlite-backup/backups
mkdir -p /var/log/sqlite-backup

# Copier le script
cp backup_production.py /opt/sqlite-backup/
chmod +x /opt/sqlite-backup/backup_production.py

# Générer la configuration
cd /opt/sqlite-backup
python backup_production.py config
```

### 3. Configuration systemd (Linux)

```ini
# /etc/systemd/system/sqlite-backup.service
[Unit]
Description=Service de sauvegarde SQLite automatique
After=network.target

[Service]
Type=simple
User=backup
Group=backup
WorkingDirectory=/opt/sqlite-backup
ExecStart=/opt/sqlite-backup/backup_env/bin/python /opt/sqlite-backup/backup_production.py service
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Activer et démarrer le service
sudo systemctl enable sqlite-backup
sudo systemctl start sqlite-backup
sudo systemctl status sqlite-backup
```

### 4. Configuration cron pour sauvegarde

```bash
# Éditer le crontab
crontab -e

# Ajouter une ligne pour sauvegarde quotidienne à 2h
0 2 * * * /opt/sqlite-backup/backup_env/bin/python /opt/sqlite-backup/backup_production.py backup >> /var/log/sqlite-backup/cron.log 2>&1

# Rotation hebdomadaire le dimanche à 1h
0 1 * * 0 /opt/sqlite-backup/backup_env/bin/python /opt/sqlite-backup/backup_production.py rotate >> /var/log/sqlite-backup/cron.log 2>&1
```

## Récapitulatif et bonnes pratiques

### Ce que vous avez appris dans cette section

✅ **API de sauvegarde SQLite** : Utilisation de l'API native pour des sauvegardes cohérentes

✅ **Système complet de sauvegarde** :
- Sauvegarde progressive avec callback
- Compression et vérification d'intégrité
- Calcul de checksum pour validation
- Métadonnées de sauvegarde

✅ **Rotation intelligente** : Stratégie de conservation par âge (quotidien/hebdomadaire/mensuel)

✅ **Alertes et monitoring** :
- Notifications email automatiques
- Métriques de performance
- Rapports de santé
- Dashboard web en temps réel

✅ **Automation** :
- Planification automatique
- Service système
- Configuration par fichier JSON
- Interface en ligne de commande

✅ **Intégration cloud** : Upload vers AWS S3 avec chiffrement

✅ **Production-ready** :
- Gestion d'erreurs robuste
- Logging complet
- Configuration flexible
- Interface utilisateur

### Stratégies de sauvegarde recommandées

#### Règle 3-2-1
- **3** copies de vos données
- Sur **2** supports différents
- **1** copie hors site (cloud)

#### Fréquences recommandées
- **Quotidienne** : Pour les données critiques
- **Hebdomadaire** : Pour les données importantes
- **Mensuelle** : Pour l'archivage long terme

#### Tests de restauration
- **Mensuel** : Test de restauration complète
- **Trimestriel** : Test de récupération disaster
- **Annuel** : Audit complet du système

### Checklist de déploiement

#### Avant la mise en production
- [ ] Tests de sauvegarde sur données réelles
- [ ] Validation de l'intégrité des backups
- [ ] Configuration des alertes
- [ ] Test de restauration complète
- [ ] Documentation des procédures
- [ ] Formation de l'équipe

#### Monitoring continu
- [ ] Surveillance des échecs de sauvegarde
- [ ] Vérification de l'espace disque
- [ ] Contrôle des performances
- [ ] Tests périodiques de restauration
- [ ] Mise à jour de la documentation

### Évolutions possibles

#### Fonctionnalités avancées
- **Sauvegarde différentielle** : Sauvegarder seulement les changements
- **Chiffrement local** : Chiffrer les backups avant stockage
- **Multi-cloud** : Sauvegarde sur plusieurs providers
- **Interface web avancée** : Dashboard avec graphiques et alertes
- **API REST** : Interface programmatique pour intégration

#### Intégrations
- **Kubernetes** : Déploiement en conteneur
- **Terraform** : Infrastructure as Code
- **Ansible** : Automatisation du déploiement
- **Prometheus** : Métriques pour monitoring
- **Grafana** : Visualisation avancée

### Ressources utiles

#### Documentation officielle
- [SQLite Backup API](https://www.sqlite.org/backup.html)
- [SQLite PRAGMA statements](https://www.sqlite.org/pragma.html)
- [SQLite WAL mode](https://www.sqlite.org/wal.html)

#### Outils complémentaires
- **sqlite3-tools** : Utilitaires en ligne de commande
- **DB Browser for SQLite** : Interface graphique
- **sqlitebrowser** : Alternative open source
- **sqlite-utils** : Outils Python avancés

### Points clés à retenir

🔑 **Cohérence avant tout** : Utilisez toujours l'API de backup SQLite plutôt qu'une copie de fichier

🔑 **Automatisation** : Les sauvegardes manuelles sont vouées à l'échec

🔑 **Vérification** : Testez régulièrement vos sauvegardes en les restaurant

🔑 **Monitoring** : Surveillez les métriques et alertes

🔑 **Documentation** : Documentez vos procédures de sauvegarde et de restauration

🔑 **Test régulier** : Un backup non testé n'est pas un backup fiable

Cette section vous a donné tous les outils pour créer un système de sauvegarde professionnel pour vos bases de données SQLite. Dans la section suivante (6.5), nous explorerons la **recherche plein texte avec FTS5**, la dernière pièce du puzzle de la programmation avancée avec SQLite.

⏭️
