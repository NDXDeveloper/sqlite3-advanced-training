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

### 1. Copie simple du fichier (`cp` / `copy`)

```bash
# ❌ DANGEREUX si la base est ouverte par un autre processus en écriture
cp ma_base.db ma_base_backup.db
```

**Problèmes :**
- Risque de corruption si des écritures sont en cours pendant la copie
- Pas de cohérence si la base est utilisée en mode WAL (fichier `-wal` séparé)
- Aucun contrôle sur la progression

### 1bis. Commande `.backup` du shell sqlite3 (SÛRE)

```bash
# ✅ SÛRE : utilise l'API de backup SQLite (cohérence garantie)
sqlite3 ma_base.db ".backup ma_base_backup.db"
```

> 💡 **Important** : `.backup` du shell `sqlite3` **utilise en interne l'API de backup officielle** (`sqlite3_backup_init`, `sqlite3_backup_step`...). C'est donc **équivalent** à la méthode Python `source.backup(dest)` et **safe** même si la base est utilisée concurremment. Ce n'est PAS une simple copie de fichier.  
>  
> Différence pratique avec l'API Python : `.backup` du shell est synchrone et ne donne pas de callback de progression — pour une sauvegarde incrémentale avec barre de progression, utiliser l'API Python (section suivante).

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

## Sauvegarde vers le cloud (résumé)

Pour respecter la règle 3-2-1 (3 copies, 2 supports, 1 hors-site), uploader les backups vers un stockage cloud :

**AWS S3** (via `boto3`) :
```python
import boto3  
s3 = boto3.client('s3')  
s3.upload_file('backup.db.gz', 'mon-bucket', 'backups/app_2026-01-15.db.gz',  
               ExtraArgs={'StorageClass': 'STANDARD_IA', 'ServerSideEncryption': 'AES256'})
```

**Alternatives** :
- **Backblaze B2** (moins cher que S3 pour le stockage long terme)
- **rsync** vers un serveur distant
- **rclone** (interface unifiée pour ~30 fournisseurs cloud)
- **Litestream** : réplication continue en streaming WAL vers S3/GCS/B2 (recommandé pour minimiser la perte de données — RPO < 1 seconde possible)

💡 Pour SQLite spécifiquement, **Litestream** est souvent la meilleure solution : il réplique en continu le journal WAL vers un object store et permet une restauration à un point quelconque dans le temps.

## Surveillance et alertes (résumé)

Pour être notifié en cas d'échec de sauvegarde :

**Approche basique** : envoyer un email via `smtplib` à la fin de chaque exécution (échec ou succès selon le contexte).

```python
import smtplib  
from email.message import EmailMessage  

def alerter_echec_backup(erreur: str):
    msg = EmailMessage()
    msg['Subject'] = '❌ Échec sauvegarde SQLite'
    msg['From'], msg['To'] = 'backup@x.com', 'admin@x.com'
    msg.set_content(f'Erreur : {erreur}')
    with smtplib.SMTP_SSL('smtp.gmail.com', 465) as s:
        s.login(user, app_password)
        s.send_message(msg)
```

**Approche moderne** : utiliser un service tiers (Sentry pour les exceptions, PagerDuty/Opsgenie pour les alertes critiques, Slack/Discord webhooks pour les notifications d'équipe). Ces services gèrent déjà la déduplication, l'escalade, les acquittements.

## Monitoring avec métriques (résumé)

Pour suivre l'évolution des backups dans le temps, on peut stocker un **historique** dans une base SQLite dédiée :

```sql
CREATE TABLE historique_backups (
    id INTEGER PRIMARY KEY,
    timestamp DATETIME,
    base_source TEXT,
    taille_source_mb REAL,
    taille_backup_mb REAL,
    duree_seconde REAL,
    compression_ratio REAL,
    succes BOOLEAN,
    erreur TEXT
);
```

À chaque sauvegarde, insérer une ligne. Pour les rapports : `SELECT` avec agrégations (`AVG(duree_seconde)`, `SUM(taille_backup_mb)`, taux d'échec...). On peut aussi alimenter un système de monitoring externe (Prometheus, Datadog, CloudWatch) pour bénéficier de graphes et alertes prêts à l'emploi.

## Dashboard web (résumé)

Pour visualiser l'état des backups, on peut créer un mini-dashboard HTTP avec `http.server.BaseHTTPRequestHandler` (sans dépendance) ou un framework comme Flask/FastAPI.

**Endpoints typiques** :
- `GET /` : page HTML avec graphes JavaScript (Chart.js, D3)
- `GET /api/stats` : JSON des métriques globales
- `GET /api/backups` : JSON de la liste des backups récents

Pour un usage sérieux, préférer **Prometheus + Grafana** : exposer les métriques au format Prometheus dans le script de backup et utiliser un Grafana existant pour la visualisation. Cela évite de réimplémenter un dashboard et bénéficie de l'écosystème.

## Script de production en CLI (résumé)

Pour un usage en production, on peut envelopper `GestionnaireBackup` dans un script CLI avec `argparse`, supportant des actions : `backup`, `rotate`, `list`, `restore`, `health`, `report`.

Pattern recommandé :
```python
import argparse, json  
from pathlib import Path  

parser = argparse.ArgumentParser()  
parser.add_argument('action', choices=['backup', 'rotate', 'list', 'restore'])  
parser.add_argument('--config', help='Fichier JSON de configuration')  
args = parser.parse_args()  

# Charger la config (fusion profonde avec valeurs par défaut)
config = charger_config(args.config)  
manager = GestionnaireBackup(config['base_de_donnees'], config['repertoire_backup'])  
# Dispatch sur args.action ...
```

⚠️ Pour la **fusion de configuration JSON**, utiliser une `deep_merge` récursive — `dict.update()` standard écrase entièrement les sous-dictionnaires (`alertes`, `rotation`...), perdant les valeurs par défaut non remplacées.

## Déploiement en production (résumé)

Pour automatiser la sauvegarde en production sous Linux :

**Via cron** (simple) :
```bash
# crontab -e
0 2 * * * /usr/bin/sqlite3 /opt/data/app.db ".backup /opt/backups/app_$(date +\%Y\%m\%d).db"
```

**Via systemd timer** (plus moderne) : créer une unit `.service` qui exécute le script Python de sauvegarde + une unit `.timer` qui le déclenche périodiquement (`OnCalendar=daily`).

**Considérations production** :
- Stocker les backups sur un volume séparé (idéalement réseau ou cloud)
- Surveiller l'espace disque (les backups s'accumulent vite)
- Tester régulièrement la restauration (un backup non testé n'est pas un backup)
- Chiffrer si les données sont sensibles (`gpg`, `age`, ou SQLCipher)

## Récapitulatif et bonnes pratiques

### Ce que vous avez appris

✅ **API de sauvegarde SQLite native** : Utilisation de `Connection.backup()` (Python) et `.backup` (shell) pour des sauvegardes cohérentes — sûres même si la base est en cours d'utilisation.

✅ **Sauvegarde progressive** : Découpe en blocs de N pages avec callback de progression — essentiel pour les grosses bases (>1 Go).

✅ **`GestionnaireBackup` complet** : Compression `gzip`, vérification d'intégrité (`PRAGMA integrity_check`), calcul de checksum SHA256, métadonnées dans un fichier `.info`.

✅ **Rotation intelligente** : Stratégie de conservation par âge (quotidien / hebdomadaire / mensuel) — équilibre entre granularité récente et espace disque.

✅ **Restauration** : Décompression et vérification d'intégrité du fichier restauré avant écrasement de la cible.

✅ **Planification** : Utilisation de la bibliothèque `schedule` dans un thread daemon pour automatiser sans cron externe (ou cron/systemd timer pour la production).

✅ **Connaissance des écosystèmes** (résumés) : cloud (S3, B2, Litestream), alertes (Sentry, PagerDuty), monitoring (Prometheus, Grafana) — pour savoir quand utiliser un outil dédié plutôt que de réimplémenter.

### Stratégies de sauvegarde recommandées

**Règle 3-2-1** : 3 copies sur 2 supports différents dont 1 hors site.

**Fréquences** :
- Quotidienne pour les données critiques
- Hebdomadaire pour les données importantes
- Mensuelle pour l'archivage long terme

**Tests de restauration** :
- Mensuel : test de restauration complète
- Trimestriel : simulation de récupération après incident
- Annuel : audit complet du système

### Checklist de mise en production

- [ ] Tester la sauvegarde sur des données réelles (volume, contenu)
- [ ] Vérifier l'intégrité des backups (`PRAGMA integrity_check`)
- [ ] Configurer un canal d'alerte (email, Slack, Sentry)
- [ ] **Tester la restauration complète** (un backup non testé n'est pas un backup)
- [ ] Documenter la procédure de récupération
- [ ] Surveiller l'espace disque du répertoire de backups

### Points clés à retenir

🔑 **Cohérence avant tout** : utiliser l'API de backup SQLite (ou `.backup` du shell), **jamais** une simple copie de fichier `cp` si la base est ouverte.

🔑 **Automatisation** : les sauvegardes manuelles sont vouées à l'oubli.

🔑 **Vérification** : un backup non testé n'est pas un backup fiable.

🔑 **3-2-1** : minimum 3 copies, 2 supports, 1 hors site.

🔑 **Litestream** : pour SQLite spécifiquement, c'est souvent la meilleure solution de sauvegarde continue (RPO < 1s).

### Ressources utiles

- 📖 [SQLite Backup API](https://www.sqlite.org/backup.html) (documentation officielle)
- ☁️ [Litestream](https://litestream.io/) — réplication continue vers S3/GCS/B2
- 🛠️ [sqlite-utils](https://sqlite-utils.datasette.io/) — outils Python avancés
- 🔍 [DB Browser for SQLite](https://sqlitebrowser.org/) — interface graphique pour inspection

Cette section vous a donné les outils essentiels pour sauvegarder et restaurer vos bases SQLite. Dans la section suivante (6.5), nous explorerons la **gestion des erreurs et exceptions** — comment rendre vos applications robustes face aux pannes, verrous, contraintes, et autres aléas.

⏭️ [6.5 Gestion des erreurs et exceptions](/06-programmation-avancee-sqlite/05-gestion-erreurs-exceptions.md)
