🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.4 Sauvegardes automatisées et stratégies de récupération

## Introduction aux sauvegardes SQLite

Les sauvegardes sont essentielles pour protéger vos données contre la perte, la corruption, ou les erreurs humaines. Avec SQLite, la sauvegarde présente des défis particuliers car il s'agit d'un fichier unique qui peut être en cours d'utilisation.

### Pourquoi les sauvegardes sont-elles critiques ?

**Scénarios de perte de données :**
- 💻 **Panne matérielle** : Disque dur défaillant, serveur en panne
- 🔥 **Sinistres** : Incendie, inondation, catastrophe naturelle
- 🦠 **Malware/Ransomware** : Chiffrement malveillant des données
- 👤 **Erreur humaine** : Suppression accidentelle, mauvaise manipulation
- 🐛 **Bug logiciel** : Corruption de données, mise à jour ratée
- 🔒 **Vol/Sécurité** : Accès non autorisé, destruction intentionnelle

**Exemples concrets :**
```
🏥 Hôpital : Perte des dossiers patients = problème critique
💰 Banque : Corruption des transactions = désastre financier
🛒 E-commerce : Perte des commandes = perte de revenus
📚 École : Suppression des notes = problème administratif majeur
```

### Spécificités des sauvegardes SQLite

**Avantages de SQLite :**
- ✅ Fichier unique = sauvegarde simple
- ✅ Pas de processus serveur = moins de complexité
- ✅ Portable = fonctionne sur toutes plateformes
- ✅ Auto-contenu = pas de dépendances externes

**Défis particuliers :**
- ⚠️ Base active = risque de corruption pendant la copie
- ⚠️ Transactions longues = problème de cohérence
- ⚠️ WAL mode = fichiers multiples (-wal, -shm)
- ⚠️ Verrouillage = sauvegarde peut bloquer l'application

## Méthodes de sauvegarde SQLite

### 1. Copie simple de fichier (À éviter en production)

```python
import shutil  
import sqlite3  
from datetime import datetime  

def sauvegarde_simple_dangereuse(source_db, destination):
    """
    ❌ MÉTHODE DANGEREUSE - À des fins éducatives uniquement
    Ne pas utiliser sur une base active !
    """
    print("⚠️ ATTENTION: Méthode dangereuse pour base active")

    try:
        # Copie directe du fichier - PEUT CORROMPRE LES DONNÉES
        shutil.copy2(source_db, destination)
        print(f"✅ Fichier copié vers {destination}")

        # Problèmes potentiels de cette approche:
        print("🚨 Risques de cette méthode:")
        print("   - Corruption si la base est en cours d'utilisation")
        print("   - Transactions incomplètes")
        print("   - Fichiers WAL/SHM non synchronisés")
        return True

    except Exception as e:
        print(f"❌ Erreur lors de la copie: {e}")
        return False

# Exemple d'utilisation (NE PAS FAIRE EN PRODUCTION)
# sauvegarde_simple_dangereuse('ma_base.db', 'backup_dangereux.db')
```

**Pourquoi cette méthode est dangereuse :**
- Si quelqu'un écrit dans la base pendant la copie → corruption
- Les fichiers WAL (-wal) et SHM (-shm) peuvent être désynchronisés
- Transactions en cours peuvent être coupées en deux

### 2. API de sauvegarde SQLite (Méthode recommandée)

> ⚠️ **Python 3.12+ : adapter datetime obligatoire (avec normalisation UTC)**  
>  
> Comme expliqué en 8.2 et 8.3, deux pièges combinés : `datetime` direct à `execute()` est déprécié, ET `CURRENT_TIMESTAMP` est en UTC alors que `datetime.now()` est en local. Sans normalisation, les rapports de rétention/nettoyage **manquent des entrées** silencieusement sur les systèmes hors UTC.

```python
import sqlite3  
import os  
from datetime import datetime, timezone  

# Adapter Python 3.12+ + normalisation UTC (cf. notes des sections 8.2/8.3)
def _adapter_dt_utc(d):
    if d.tzinfo is None:
        d = d.astimezone(timezone.utc)
    else:
        d = d.astimezone(timezone.utc)
    return d.strftime("%Y-%m-%d %H:%M:%S")
sqlite3.register_adapter(datetime, _adapter_dt_utc)

class GestionnaireSauvegarde:
    def __init__(self, base_source):
        self.base_source = base_source

    def sauvegarde_sqlite_api(self, fichier_destination):
        """
        ✅ MÉTHODE RECOMMANDÉE - Utilise l'API de sauvegarde SQLite
        Cette méthode est sûre même avec une base active
        """
        print(f"💾 Sauvegarde sécurisée de {self.base_source}")
        print(f"📁 Destination: {fichier_destination}")

        try:
            # Connexion à la base source
            source_conn = sqlite3.connect(self.base_source)

            # Créer la base de destination
            dest_conn = sqlite3.connect(fichier_destination)

            # Effectuer la sauvegarde page par page
            # Cette méthode respecte les verrous et assure la cohérence
            source_conn.backup(dest_conn)

            # Fermer les connexions
            dest_conn.close()
            source_conn.close()

            # Vérifier la taille du fichier créé
            taille = os.path.getsize(fichier_destination)
            print(f"✅ Sauvegarde terminée ({taille:,} bytes)")

            return True

        except Exception as e:
            print(f"❌ Erreur lors de la sauvegarde: {e}")
            return False

    def sauvegarde_avec_verification(self, fichier_destination):
        """Sauvegarde avec vérification d'intégrité"""
        print(f"🔍 Sauvegarde avec vérification...")

        # Effectuer la sauvegarde
        if not self.sauvegarde_sqlite_api(fichier_destination):
            return False

        # Vérifier l'intégrité de la sauvegarde
        try:
            test_conn = sqlite3.connect(fichier_destination)
            cursor = test_conn.cursor()

            # Test d'intégrité — PRAGMA integrity_check peut retourner
            # plusieurs lignes en cas de corruption multiple, d'où fetchall().
            cursor.execute("PRAGMA integrity_check")
            lignes_integrite = [row[0] for row in cursor.fetchall()]

            if lignes_integrite == ['ok']:
                print("✅ Intégrité de la sauvegarde vérifiée")

                # Compter les tables pour validation supplémentaire
                cursor.execute("SELECT COUNT(*) FROM sqlite_master WHERE type='table'")
                nb_tables = cursor.fetchone()[0]
                print(f"📋 {nb_tables} tables dans la sauvegarde")

                test_conn.close()
                return True
            else:
                print(f"❌ {len(lignes_integrite)} problème(s) d'intégrité :")
                for ligne in lignes_integrite[:5]:
                    print(f"   • {ligne}")
                test_conn.close()
                return False

        except Exception as e:
            print(f"❌ Erreur lors de la vérification: {e}")
            return False

    def sauvegarde_incrementielle(self, fichier_destination, derniere_sauvegarde=None):
        """
        Sauvegarde incrémentielle basée sur les timestamps.

        ⚠️ LIMITES IMPORTANTES :
        - SQLite ne supporte pas nativement les sauvegardes incrémentales.
        - En mode WAL, les écritures vont dans le fichier `.db-wal` SANS modifier
          le mtime du fichier `.db` principal tant qu'aucun checkpoint n'a eu lieu.
          → la détection par mtime peut RATER des modifications récentes.
        - Solution robuste : utiliser **Litestream** (réplication continue du WAL)
          plutôt que de réinventer l'incrémental.
        """
        print("📈 Sauvegarde incrémentielle (méthode basique)...")

        if not derniere_sauvegarde:
            print("   Première sauvegarde - sauvegarde complète")
            return self.sauvegarde_avec_verification(fichier_destination)

        try:
            # Forcer un checkpoint pour que les changements WAL soient pris en compte
            conn = sqlite3.connect(self.base_source)
            conn.execute("PRAGMA wal_checkpoint(PASSIVE)")
            conn.close()

            # Comparer les dates de modification (après checkpoint)
            source_modif = os.path.getmtime(self.base_source)
            backup_modif = os.path.getmtime(derniere_sauvegarde)

            if source_modif <= backup_modif:
                print("✅ Aucune modification depuis la dernière sauvegarde")
                return True

            print("🔄 Modifications détectées - sauvegarde nécessaire")
            return self.sauvegarde_avec_verification(fichier_destination)

        except Exception as e:
            print(f"❌ Erreur sauvegarde incrémentielle: {e}")
            return False

# Utilisation du gestionnaire de sauvegarde
gestionnaire = GestionnaireSauvegarde('ma_base.db')

print("=== TEST DES MÉTHODES DE SAUVEGARDE ===")

# Sauvegarde basique sécurisée
timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')  
fichier_backup = f'backup_{timestamp}.db'  

if gestionnaire.sauvegarde_sqlite_api(fichier_backup):
    print(f"✅ Sauvegarde créée: {fichier_backup}")

# Sauvegarde avec vérification
fichier_backup_verifie = f'backup_verifie_{timestamp}.db'  
gestionnaire.sauvegarde_avec_verification(fichier_backup_verifie)  
```

### Litestream : sauvegarde continue vers S3 (recommandé 2024+)

[Litestream](https://litestream.io/) est devenu **le standard de facto** pour la sauvegarde des bases SQLite en production. Plutôt que de planifier des snapshots périodiques, Litestream **réplique en continu** le WAL vers un stockage objet (S3, Azure Blob, GCS, SFTP). Avantages décisifs :

- **RPO de quelques secondes** au lieu de "dernière sauvegarde minuit"
- **Aucune modification du code applicatif** : Litestream lit simplement le journal WAL
- **Restauration point-in-time** : retour à n'importe quel instant
- **Coût stockage faible** : utilise les classes "infrequent access" de S3

**Installation et configuration de base** :

```bash
# Installation Linux (binaire unique, pas de dépendances)
# Version 0.5.11 — dernière stable en avril 2026 ; vérifier sur
# https://github.com/benbjohnson/litestream/releases pour la plus récente.
wget https://github.com/benbjohnson/litestream/releases/download/v0.5.11/litestream-v0.5.11-linux-amd64.tar.gz  
tar -xzf litestream-v0.5.11-linux-amd64.tar.gz  
sudo mv litestream /usr/local/bin/  

# Vérifier
litestream version
```

> 💡 **Migration v0.3.x → v0.5.x** : la branche 0.5 a introduit un nouveau format  
> de stockage incompatible avec 0.3. Si vous avez des sauvegardes existantes en  
> 0.3.x, consultez la [Migration Guide](https://litestream.io/docs/migration/)  
> avant de mettre à jour en production. Pour une nouvelle installation, prenez  
> directement 0.5.x.

**`/etc/litestream.yml`** :
```yaml
dbs:
  - path: /var/lib/app/data.db
    replicas:
      - type: s3
        bucket: mon-bucket-backup
        path: production/data.db
        region: eu-west-1
        # AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY via env
        retention: 168h           # 7 jours d'historique
        snapshot-interval: 24h    # snapshot complet quotidien
        sync-interval: 1s         # envoi continu toutes les secondes
```

**Démarrage en service systemd** :
```bash
sudo tee /etc/systemd/system/litestream.service <<'EOF'
[Unit]
Description=Litestream replicator  
After=network.target  

[Service]
ExecStart=/usr/local/bin/litestream replicate  
Restart=always  
EnvironmentFile=/etc/litestream.env  

[Install]
WantedBy=multi-user.target  
EOF  

sudo systemctl enable --now litestream
```

**Restauration depuis S3** :
```bash
# Dernière version
litestream restore -o /var/lib/app/data.db s3://mon-bucket/production/data.db

# Point-in-time recovery — retour à un instant précis
litestream restore \
    -timestamp 2026-05-27T10:30:00Z \
    -o /var/lib/app/data.db \
    s3://mon-bucket/production/data.db
```

> ⚠️ **Pré-requis Litestream** :  
> - **Mode WAL obligatoire** (`PRAGMA journal_mode = WAL`) — Litestream lit le `.db-wal`  
> - **Un seul processus Litestream par base** (sinon corruption)  
> - **Asymétrique uniquement** : primary → N replicas en lecture seule. Pour multi-maître, voir cr-sqlite, ElectricSQL ou PowerSync (cf. 7.5)

### LiteFS : haute disponibilité avec failover

[LiteFS](https://fly.io/docs/litefs/) (par les mêmes auteurs que Litestream) va plus loin : système de fichiers FUSE qui réplique au niveau **page** vers un cluster distribué. Permet plusieurs nœuds avec **failover automatique**. Compromis : un seul writer à la fois (élection de leader via Consul), les lecteurs voient des données avec un léger lag.

### 3. Sauvegarde en ligne de commande

```bash
#!/bin/bash
# Script de sauvegarde SQLite sécurisé

# Configuration
SOURCE_DB="ma_base.db"  
BACKUP_DIR="/backup/sqlite"  
DATE=$(date +%Y%m%d_%H%M%S)  
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.db"  

echo "🔄 Début de la sauvegarde SQLite"  
echo "Source: $SOURCE_DB"  
echo "Destination: $BACKUP_FILE"  

# Créer le répertoire de sauvegarde si nécessaire
mkdir -p "$BACKUP_DIR"

# Méthode 1: Utiliser l'API de sauvegarde SQLite (recommandée)
sqlite3 "$SOURCE_DB" ".backup '$BACKUP_FILE'"

if [ $? -eq 0 ]; then
    echo "✅ Sauvegarde SQLite réussie"

    # Vérifier l'intégrité
    echo "🔍 Vérification de l'intégrité..."
    INTEGRITY=$(sqlite3 "$BACKUP_FILE" "PRAGMA integrity_check;")

    if [ "$INTEGRITY" = "ok" ]; then
        echo "✅ Intégrité vérifiée"

        # Comprimer la sauvegarde pour économiser l'espace
        echo "📦 Compression en cours..."
        gzip "$BACKUP_FILE"

        if [ $? -eq 0 ]; then
            echo "✅ Sauvegarde compressée: $BACKUP_FILE.gz"

            # Nettoyer les anciennes sauvegardes (garder 30 jours)
            find "$BACKUP_DIR" -name "backup_*.db.gz" -mtime +30 -delete
            echo "🧹 Anciennes sauvegardes nettoyées"

            # Envoyer une notification de succès
            echo "$(date): Sauvegarde réussie - $BACKUP_FILE.gz" >> /var/log/sqlite_backup.log

        else
            echo "❌ Erreur lors de la compression"
        fi
    else
        echo "❌ Problème d'intégrité détecté: $INTEGRITY"
        rm "$BACKUP_FILE"
        exit 1
    fi
else
    echo "❌ Échec de la sauvegarde SQLite"
    exit 1
fi

echo "✅ Processus de sauvegarde terminé"
```

## Automatisation des sauvegardes

### Planificateur de sauvegardes Python

```python
import schedule  
import time  
import threading  
import os  
import smtplib  
from datetime import datetime, timedelta  
from email.mime.text import MIMEText  
from email.mime.multipart import MIMEMultipart  

class PlanificateurSauvegarde:
    def __init__(self, base_source, repertoire_backup="/backup"):
        self.base_source = base_source
        self.repertoire_backup = repertoire_backup
        self.gestionnaire = GestionnaireSauvegarde(base_source)
        self.derniere_sauvegarde = None
        self.historique_sauvegardes = []

        # ⚠️ Configuration des notifications via variables d'environnement
        #    JAMAIS de mot de passe SMTP en dur dans le code (commit Git = fuite).
        #    Préférer un secret manager en production (Vault, AWS Secrets, ...).
        self.email_config = {
            'smtp_server': os.environ.get('SMTP_SERVER', 'smtp.gmail.com'),
            'smtp_port': int(os.environ.get('SMTP_PORT', '587')),
            'email_from': os.environ.get('SMTP_FROM', 'app@example.com'),
            'email_to': os.environ.get('SMTP_TO', 'admin@example.com'),
            'password': os.environ.get('SMTP_PASSWORD'),  # None si non défini
        }
        # Désactiver l'envoi si mot de passe manquant plutôt que crasher au runtime
        self.email_actif = bool(self.email_config['password'])
        if not self.email_actif:
            print("ℹ️ SMTP_PASSWORD non définie — notifications email désactivées.")

    def sauvegarde_quotidienne(self):
        """Exécute une sauvegarde quotidienne"""
        print(f"\n🔄 === SAUVEGARDE QUOTIDIENNE {datetime.now()} ===")

        # Nom du fichier avec timestamp
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        fichier_backup = f"{self.repertoire_backup}/daily_backup_{timestamp}.db"

        # Créer le répertoire si nécessaire
        os.makedirs(self.repertoire_backup, exist_ok=True)

        # Effectuer la sauvegarde
        debut = time.time()
        succes = self.gestionnaire.sauvegarde_avec_verification(fichier_backup)
        duree = time.time() - debut

        # Enregistrer dans l'historique
        resultat = {
            'timestamp': datetime.now(),
            'fichier': fichier_backup,
            'succes': succes,
            'duree_secondes': duree,
            'type': 'quotidienne'
        }
        self.historique_sauvegardes.append(resultat)

        if succes:
            self.derniere_sauvegarde = fichier_backup
            print(f"✅ Sauvegarde quotidienne réussie ({duree:.1f}s)")
            print(f"📁 Fichier: {fichier_backup}")

            # Compresser pour économiser l'espace
            self._comprimer_sauvegarde(fichier_backup)

        else:
            print("❌ Échec de la sauvegarde quotidienne")
            self._envoyer_alerte("Échec sauvegarde quotidienne",
                               f"La sauvegarde du {datetime.now()} a échoué")

        return succes

    def sauvegarde_hebdomadaire(self):
        """Exécute une sauvegarde hebdomadaire avec archivage long terme"""
        print(f"\n📦 === SAUVEGARDE HEBDOMADAIRE {datetime.now()} ===")

        timestamp = datetime.now().strftime('%Y_semaine_%U')
        fichier_backup = f"{self.repertoire_backup}/weekly_backup_{timestamp}.db"

        os.makedirs(self.repertoire_backup, exist_ok=True)

        debut = time.time()
        succes = self.gestionnaire.sauvegarde_avec_verification(fichier_backup)
        duree = time.time() - debut

        resultat = {
            'timestamp': datetime.now(),
            'fichier': fichier_backup,
            'succes': succes,
            'duree_secondes': duree,
            'type': 'hebdomadaire'
        }
        self.historique_sauvegardes.append(resultat)

        if succes:
            print(f"✅ Sauvegarde hebdomadaire réussie ({duree:.1f}s)")
            self._comprimer_sauvegarde(fichier_backup)

            # Nettoyer les sauvegardes quotidiennes anciennes
            self._nettoyer_sauvegardes_anciennes('daily_backup_', 7)

        else:
            print("❌ Échec de la sauvegarde hebdomadaire")
            self._envoyer_alerte("Échec sauvegarde hebdomadaire",
                               f"La sauvegarde hebdomadaire du {datetime.now()} a échoué")

        return succes

    def sauvegarde_urgence(self, raison="Manuel"):
        """Exécute une sauvegarde d'urgence"""
        print(f"\n🚨 === SAUVEGARDE D'URGENCE ===")
        print(f"Raison: {raison}")

        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        fichier_backup = f"{self.repertoire_backup}/emergency_backup_{timestamp}.db"

        os.makedirs(self.repertoire_backup, exist_ok=True)

        debut = time.time()
        succes = self.gestionnaire.sauvegarde_avec_verification(fichier_backup)
        duree = time.time() - debut

        resultat = {
            'timestamp': datetime.now(),
            'fichier': fichier_backup,
            'succes': succes,
            'duree_secondes': duree,
            'type': 'urgence',
            'raison': raison
        }
        self.historique_sauvegardes.append(resultat)

        if succes:
            print(f"✅ Sauvegarde d'urgence réussie ({duree:.1f}s)")
            # Notifier immédiatement le succès
            self._envoyer_notification("Sauvegarde d'urgence réussie",
                                     f"Sauvegarde d'urgence créée: {fichier_backup}")
        else:
            print("❌ Échec de la sauvegarde d'urgence")
            self._envoyer_alerte("CRITIQUE: Échec sauvegarde d'urgence",
                               f"Sauvegarde d'urgence a échoué. Raison: {raison}")

        return succes

    def _comprimer_sauvegarde(self, fichier):
        """Compresse une sauvegarde pour économiser l'espace"""
        try:
            import gzip

            with open(fichier, 'rb') as f_in:
                with gzip.open(f"{fichier}.gz", 'wb') as f_out:
                    f_out.writelines(f_in)

            # Supprimer le fichier non compressé
            os.remove(fichier)

            taille_originale = os.path.getsize(f"{fichier}.gz")
            print(f"📦 Sauvegarde compressée: {fichier}.gz ({taille_originale:,} bytes)")

        except Exception as e:
            print(f"⚠️ Erreur lors de la compression: {e}")

    def _nettoyer_sauvegardes_anciennes(self, prefixe, jours_retention):
        """Supprime les sauvegardes plus anciennes que la période de rétention"""
        try:
            import glob

            date_limite = datetime.now() - timedelta(days=jours_retention)
            pattern = f"{self.repertoire_backup}/{prefixe}*.gz"

            fichiers_supprimes = 0
            for fichier in glob.glob(pattern):
                if os.path.getmtime(fichier) < date_limite.timestamp():
                    os.remove(fichier)
                    fichiers_supprimes += 1

            if fichiers_supprimes > 0:
                print(f"🧹 {fichiers_supprimes} anciennes sauvegardes supprimées")

        except Exception as e:
            print(f"⚠️ Erreur lors du nettoyage: {e}")

    def _envoyer_notification(self, sujet, message):
        """Envoie une notification par email (silencieusement skippé si désactivé)"""
        if not self.email_actif:
            return  # Pas de mot de passe SMTP → on n'essaie pas
        try:
            msg = MIMEMultipart()
            msg['From'] = self.email_config['email_from']
            msg['To'] = self.email_config['email_to']
            msg['Subject'] = f"SQLite Backup: {sujet}"

            body = f"""
Notification de sauvegarde SQLite

{message}

Base de données: {self.base_source}  
Timestamp: {datetime.now()}  

Historique récent:
{self._generer_historique_recent()}
            """

            msg.attach(MIMEText(body, 'plain'))

            server = smtplib.SMTP(self.email_config['smtp_server'],
                                 self.email_config['smtp_port'])
            server.starttls()
            server.login(self.email_config['email_from'],
                        self.email_config['password'])

            text = msg.as_string()
            server.sendmail(self.email_config['email_from'],
                           self.email_config['email_to'], text)
            server.quit()

            print("📧 Notification envoyée par email")

        except Exception as e:
            print(f"⚠️ Erreur envoi email: {e}")

    def _envoyer_alerte(self, sujet, message):
        """Envoie une alerte critique"""
        print(f"🚨 ALERTE: {sujet}")
        print(f"   {message}")
        # En production, intégrer avec votre système d'alertes
        # (Slack, PagerDuty, SMS, etc.)
        self._envoyer_notification(f"ALERTE: {sujet}", message)

    def _generer_historique_recent(self):
        """Génère un résumé de l'historique récent"""
        if not self.historique_sauvegardes:
            return "Aucune sauvegarde dans l'historique"

        recent = self.historique_sauvegardes[-5:]  # 5 dernières
        lignes = []

        for sauvegarde in recent:
            statut = "✅" if sauvegarde['succes'] else "❌"
            lignes.append(
                f"{statut} {sauvegarde['timestamp'].strftime('%Y-%m-%d %H:%M')} "
                f"({sauvegarde['type']}) - {sauvegarde['duree_secondes']:.1f}s"
            )

        return "\n".join(lignes)

    def configurer_planning(self):
        """Configure le planning des sauvegardes automatiques"""
        print("⚙️ Configuration du planning des sauvegardes...")

        # Sauvegarde quotidienne à 2h du matin
        schedule.every().day.at("02:00").do(self.sauvegarde_quotidienne)

        # Sauvegarde hebdomadaire le dimanche à 1h du matin
        schedule.every().sunday.at("01:00").do(self.sauvegarde_hebdomadaire)

        print("📅 Planning configuré:")
        print("   • Quotidienne: tous les jours à 02:00")
        print("   • Hebdomadaire: dimanche à 01:00")
        print("   • Urgence: sur demande")

    def demarrer_planificateur(self):
        """Démarre le planificateur en arrière-plan"""
        def run_scheduler():
            while True:
                schedule.run_pending()
                time.sleep(60)  # Vérifier chaque minute

        # Lancer le planificateur dans un thread séparé
        scheduler_thread = threading.Thread(target=run_scheduler, daemon=True)
        scheduler_thread.start()

        print("🚀 Planificateur de sauvegardes démarré")
        print("   (Thread en arrière-plan)")

    def afficher_statut(self):
        """Affiche le statut des sauvegardes"""
        print("\n📊 === STATUT DES SAUVEGARDES ===")

        if not self.historique_sauvegardes:
            print("ℹ️ Aucune sauvegarde dans l'historique")
            return

        # Dernière sauvegarde
        derniere = self.historique_sauvegardes[-1]
        statut = "✅ Succès" if derniere['succes'] else "❌ Échec"
        print(f"Dernière sauvegarde: {derniere['timestamp']} - {statut}")

        # Statistiques
        total = len(self.historique_sauvegardes)
        succes = sum(1 for s in self.historique_sauvegardes if s['succes'])
        taux_succes = (succes / total * 100) if total > 0 else 0

        print(f"Total sauvegardes: {total}")
        print(f"Taux de succès: {taux_succes:.1f}% ({succes}/{total})")

        # Prochaine sauvegarde planifiée
        prochains_jobs = schedule.jobs
        if prochains_jobs:
            prochain = min(job.next_run for job in prochains_jobs)
            print(f"Prochaine sauvegarde: {prochain}")

# Utilisation du planificateur
planificateur = PlanificateurSauvegarde('ma_base.db', '/backup/sqlite')

print("=== DÉMONSTRATION PLANIFICATEUR ===")

# Configuration et démarrage
planificateur.configurer_planning()

# Test d'une sauvegarde manuelle
planificateur.sauvegarde_quotidienne()

# Test d'une sauvegarde d'urgence
planificateur.sauvegarde_urgence("Test du système")

# Afficher le statut
planificateur.afficher_statut()

# En production, démarrer le planificateur
# planificateur.demarrer_planificateur()
print("\nℹ️ Pour activer en production, décommenter 'demarrer_planificateur()'")
```

### Configuration avec crontab (Linux/Mac)

```bash
# Éditer le crontab
# crontab -e

# Ajouter ces lignes pour automatiser les sauvegardes

# Sauvegarde quotidienne à 2h du matin
0 2 * * * /home/user/scripts/backup_sqlite.sh

# Sauvegarde hebdomadaire le dimanche à 1h du matin
0 1 * * 0 /home/user/scripts/backup_sqlite_weekly.sh

# Vérification d'intégrité tous les jours à 3h
0 3 * * * /home/user/scripts/verify_backup.sh

# Nettoyage mensuel des anciennes sauvegardes
0 4 1 * * /home/user/scripts/cleanup_old_backups.sh
```

## Stratégies de récupération

### Détection et réparation de corruption

```python
import sqlite3  
import shutil  
from datetime import datetime  

class GestionnaireRecuperation:
    def __init__(self, base_problematique):
        self.base_problematique = base_problematique

    def diagnostiquer_probleme(self):
        """Diagnostique les problèmes de la base de données"""
        print(f"🔍 Diagnostic de {self.base_problematique}")
        print("=" * 50)

        problemes = []

        # 1. Vérifier si le fichier existe
        if not os.path.exists(self.base_problematique):
            problemes.append("CRITIQUE: Fichier de base de données introuvable")
            return problemes

        # 2. Vérifier si le fichier est vide
        taille = os.path.getsize(self.base_problematique)
        if taille == 0:
            problemes.append("CRITIQUE: Fichier de base de données vide")
            return problemes

        print(f"📏 Taille du fichier: {taille:,} bytes")

        # 3. Tenter d'ouvrir la base
        try:
            conn = sqlite3.connect(self.base_problematique)
            cursor = conn.cursor()

            # 4. Test basique de lecture
            try:
                cursor.execute("SELECT name FROM sqlite_master WHERE type='table' LIMIT 1")
                print("✅ Connexion à la base réussie")
            except Exception as e:
                problemes.append(f"ERREUR: Impossible de lire les métadonnées - {e}")
                conn.close()
                return problemes

            # 5. Test d'intégrité complet
            print("🔍 Vérification d'intégrité en cours...")
            try:
                cursor.execute("PRAGMA integrity_check")
                resultats_integrite = cursor.fetchall()

                if len(resultats_integrite) == 1 and resultats_integrite[0][0] == 'ok':
                    print("✅ Intégrité: OK")
                else:
                    print("❌ Problèmes d'intégrité détectés:")
                    for resultat in resultats_integrite[:10]:  # Limiter l'affichage
                        print(f"   • {resultat[0]}")
                        problemes.append(f"INTÉGRITÉ: {resultat[0]}")

                    if len(resultats_integrite) > 10:
                        print(f"   ... et {len(resultats_integrite) - 10} autres problèmes")

            except Exception as e:
                problemes.append(f"ERREUR: Impossible de vérifier l'intégrité - {e}")

            # 6. Vérifier les index
            try:
                cursor.execute("PRAGMA foreign_key_check")
                violations_fk = cursor.fetchall()

                if violations_fk:
                    print(f"⚠️ {len(violations_fk)} violations de clés étrangères")
                    for violation in violations_fk[:5]:
                        problemes.append(f"FK_VIOLATION: Table {violation[0]}")
                else:
                    print("✅ Clés étrangères: OK")

            except Exception as e:
                problemes.append(f"ERREUR: Vérification clés étrangères - {e}")

            # 7. Test de performance basique
            try:
                debut = time.time()
                cursor.execute("SELECT COUNT(*) FROM sqlite_master")
                duree = time.time() - debut

                if duree > 5.0:  # Plus de 5 secondes pour une requête simple
                    problemes.append(f"PERFORMANCE: Requête lente ({duree:.1f}s)")
                else:
                    print(f"✅ Performance basique: OK ({duree:.3f}s)")

            except Exception as e:
                problemes.append(f"ERREUR: Test de performance - {e}")

            conn.close()

        except Exception as e:
            problemes.append(f"CRITIQUE: Impossible de se connecter à la base - {e}")

        # Résumé du diagnostic
        print(f"\n📋 RÉSUMÉ DU DIAGNOSTIC")
        print("=" * 30)

        if not problemes:
            print("✅ Aucun problème détecté")
        else:
            print(f"❌ {len(problemes)} problème(s) détecté(s):")
            for probleme in problemes:
                print(f"   • {probleme}")

        return problemes

    def tentative_reparation_automatique(self):
        """Tente de réparer automatiquement la base de données"""
        print(f"\n🔧 TENTATIVE DE RÉPARATION AUTOMATIQUE")
        print("=" * 50)

        # Créer une sauvegarde avant réparation — ⚠️ utiliser l'API .backup()
        # de SQLite : si la base est ouverte ailleurs, shutil.copy peut
        # produire une copie incohérente.
        from contextlib import closing
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        backup_avant = f"{self.base_problematique}.avant_reparation_{timestamp}"

        try:
            with closing(sqlite3.connect(self.base_problematique)) as src, \
                 closing(sqlite3.connect(backup_avant)) as dst:
                src.backup(dst)
            print(f"💾 Sauvegarde créée avant réparation: {backup_avant}")
        except Exception as e:
            print(f"⚠️ Impossible de créer une sauvegarde: {e}")
            print("🚨 ARRÊT - Réparation trop risquée sans sauvegarde")
            return False

        reparations_effectuees = []

        try:
            # ⚠️ isolation_level=None requis pour VACUUM (qui ne peut pas
            #    s'exécuter dans une transaction implicite du module sqlite3).
            conn = sqlite3.connect(self.base_problematique, isolation_level=None)
            cursor = conn.cursor()

            # 1. Tentative de récupération via VACUUM
            print("🔄 Tentative VACUUM...")
            try:
                cursor.execute("VACUUM")
                print("✅ VACUUM réussi")
                reparations_effectuees.append("VACUUM")
            except Exception as e:
                print(f"❌ VACUUM échoué: {e}")

            # 2. Réindexation
            print("🔄 Réindexation...")
            try:
                cursor.execute("REINDEX")
                conn.commit()
                print("✅ REINDEX réussi")
                reparations_effectuees.append("REINDEX")
            except Exception as e:
                print(f"❌ REINDEX échoué: {e}")

            # 3. Optimisation
            print("🔄 Optimisation...")
            try:
                cursor.execute("PRAGMA optimize")
                conn.commit()
                print("✅ Optimisation réussie")
                reparations_effectuees.append("OPTIMIZE")
            except Exception as e:
                print(f"❌ Optimisation échouée: {e}")

            conn.close()

            # 4. Vérification post-réparation
            print("\n🔍 Vérification post-réparation...")
            problemes_restants = self.diagnostiquer_probleme()

            if not problemes_restants:
                print("🎉 Réparation automatique réussie!")
                print(f"✅ Réparations effectuées: {', '.join(reparations_effectuees)}")
                return True
            else:
                print("⚠️ Problèmes persistent après réparation automatique")
                print("➡️ Réparation manuelle ou restauration nécessaire")
                return False

        except Exception as e:
            print(f"❌ Erreur durant la réparation: {e}")
            print("🔄 Restauration de la sauvegarde...")

            # Ici la base cible est inutilisable, on peut utiliser shutil sûrement
            # (pas de connexion ouverte) — mais pour cohérence on garde .backup().
            try:
                with closing(sqlite3.connect(backup_avant)) as src, \
                     closing(sqlite3.connect(self.base_problematique)) as dst:
                    src.backup(dst)
                print("✅ Base restaurée à l'état précédent")
            except Exception as restore_error:
                print(f"🚨 CRITIQUE: Impossible de restaurer - {restore_error}")

            return False

    def exporter_donnees_recuperables(self, fichier_export):
        """Exporte les données récupérables vers un nouveau fichier"""
        print(f"\n💾 EXPORT DES DONNÉES RÉCUPÉRABLES")
        print("=" * 50)

        try:
            # Connexion à la base problématique
            conn_source = sqlite3.connect(self.base_problematique)
            cursor_source = conn_source.cursor()

            # Créer une nouvelle base pour l'export
            conn_dest = sqlite3.connect(fichier_export)
            cursor_dest = conn_dest.cursor()

            donnees_exportees = 0
            tables_exportees = 0
            erreurs_export = []

            # Récupérer la liste des tables
            try:
                cursor_source.execute("SELECT name, sql FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%'")
                tables = cursor_source.fetchall()

                print(f"📋 {len(tables)} tables trouvées")

                for nom_table, sql_creation in tables:
                    print(f"\n🔄 Export de la table '{nom_table}'...")

                    try:
                        # Recréer la structure de la table
                        if sql_creation:
                            cursor_dest.execute(sql_creation)
                            print(f"   ✅ Structure de '{nom_table}' créée")

                        # Tenter d'exporter les données
                        cursor_source.execute(f"SELECT * FROM '{nom_table}'")
                        rows = cursor_source.fetchall()

                        if rows:
                            # Obtenir les noms de colonnes
                            colonnes = [description[0] for description in cursor_source.description]
                            placeholders = ','.join(['?' for _ in colonnes])

                            # Insérer les données
                            cursor_dest.executemany(
                                f"INSERT INTO '{nom_table}' VALUES ({placeholders})",
                                rows
                            )

                            print(f"   ✅ {len(rows)} enregistrements exportés")
                            donnees_exportees += len(rows)
                        else:
                            print(f"   ℹ️ Table '{nom_table}' vide")

                        tables_exportees += 1

                    except Exception as e:
                        erreur = f"Table '{nom_table}': {e}"
                        erreurs_export.append(erreur)
                        print(f"   ❌ Erreur: {e}")

                # Sauvegarder l'export
                conn_dest.commit()

                print(f"\n📊 RÉSUMÉ DE L'EXPORT")
                print(f"✅ Tables exportées: {tables_exportees}/{len(tables)}")
                print(f"✅ Enregistrements exportés: {donnees_exportees:,}")

                if erreurs_export:
                    print(f"⚠️ Erreurs rencontrées: {len(erreurs_export)}")
                    for erreur in erreurs_export[:5]:  # Afficher les 5 premières
                        print(f"   • {erreur}")
                    if len(erreurs_export) > 5:
                        print(f"   ... et {len(erreurs_export) - 5} autres erreurs")

                # Vérifier l'intégrité de l'export
                cursor_dest.execute("PRAGMA integrity_check")
                lignes_integrite = [row[0] for row in cursor_dest.fetchall()]

                if lignes_integrite == ['ok']:
                    print("✅ Intégrité de l'export vérifiée")
                    return True
                else:
                    print(f"❌ {len(lignes_integrite)} problème(s) d'intégrité dans l'export :")
                    for ligne in lignes_integrite[:5]:
                        print(f"   • {ligne}")
                    return False

            except Exception as e:
                print(f"❌ Erreur lors de l'export: {e}")
                return False

        except Exception as e:
            print(f"❌ Impossible d'ouvrir la base source: {e}")
            return False

        finally:
            # Fermeture défensive — les connexions peuvent ne pas avoir été
            # créées si l'ouverture initiale a échoué (NameError sinon).
            for c in (locals().get('conn_source'), locals().get('conn_dest')):
                if c is not None:
                    try:
                        c.close()
                    except sqlite3.Error:
                        pass

    def restaurer_depuis_sauvegarde(self, fichier_sauvegarde, remplacer=False):
        """Restaure la base depuis une sauvegarde"""
        print(f"\n♻️ RESTAURATION DEPUIS SAUVEGARDE")
        print("=" * 50)
        print(f"Source: {fichier_sauvegarde}")
        print(f"Destination: {self.base_problematique}")

        if not os.path.exists(fichier_sauvegarde):
            print(f"❌ Fichier de sauvegarde introuvable: {fichier_sauvegarde}")
            return False

        # Vérifier l'intégrité de la sauvegarde
        print("🔍 Vérification de l'intégrité de la sauvegarde...")
        try:
            conn_test = sqlite3.connect(fichier_sauvegarde)
            cursor_test = conn_test.cursor()
            cursor_test.execute("PRAGMA integrity_check")
            lignes_integrite = [row[0] for row in cursor_test.fetchall()]
            conn_test.close()

            if lignes_integrite != ['ok']:
                print(f"❌ Sauvegarde corrompue ({len(lignes_integrite)} signalements) :")
                for ligne in lignes_integrite[:5]:
                    print(f"   • {ligne}")
                return False
            else:
                print("✅ Sauvegarde intègre")

        except Exception as e:
            print(f"❌ Impossible de vérifier la sauvegarde: {e}")
            return False

        # Créer une sauvegarde de l'état actuel si demandé
        # ⚠️ Toujours utiliser l'API .backup() de SQLite plutôt que shutil.copy
        from contextlib import closing
        if not remplacer and os.path.exists(self.base_problematique):
            timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
            backup_actuel = f"{self.base_problematique}.avant_restauration_{timestamp}"

            try:
                with closing(sqlite3.connect(self.base_problematique)) as src, \
                     closing(sqlite3.connect(backup_actuel)) as dst:
                    src.backup(dst)
                print(f"💾 État actuel sauvegardé: {backup_actuel}")
            except Exception as e:
                print(f"⚠️ Impossible de sauvegarder l'état actuel: {e}")

        # Effectuer la restauration
        # ⚠️ La base cible est supprimée d'abord, donc shutil.copy2 est sûr ici
        #    (aucune connexion ouverte sur la nouvelle base). Pour les bases
        #    en cours d'utilisation, préférer src.backup(dst).
        try:
            if os.path.exists(self.base_problematique):
                os.remove(self.base_problematique)

            shutil.copy2(fichier_sauvegarde, self.base_problematique)

            # Vérifier la restauration
            print("🔍 Vérification de la restauration...")
            conn_verifie = sqlite3.connect(self.base_problematique)
            cursor_verifie = conn_verifie.cursor()

            cursor_verifie.execute("SELECT COUNT(*) FROM sqlite_master WHERE type='table'")
            nb_tables = cursor_verifie.fetchone()[0]

            cursor_verifie.execute("PRAGMA integrity_check")
            lignes_integrite = [row[0] for row in cursor_verifie.fetchall()]

            conn_verifie.close()

            if lignes_integrite == ['ok']:
                print(f"✅ Restauration réussie - {nb_tables} tables restaurées")
                return True
            else:
                print(f"❌ {len(lignes_integrite)} problème(s) après restauration :")
                for ligne in lignes_integrite[:5]:
                    print(f"   • {ligne}")
                return False

        except Exception as e:
            print(f"❌ Erreur lors de la restauration: {e}")
            return False

# Utilisation du gestionnaire de récupération
print("=== DÉMONSTRATION RÉCUPÉRATION ===")

# Simuler une base avec problèmes (pour les tests)
base_test = 'test_problematique.db'

# Créer une base de test avec quelques données
conn_test = sqlite3.connect(base_test)  
cursor_test = conn_test.cursor()  

cursor_test.execute('''
    CREATE TABLE test_donnees (
        id INTEGER PRIMARY KEY,
        nom TEXT,
        valeur INTEGER
    )
''')

cursor_test.execute("INSERT INTO test_donnees (nom, valeur) VALUES (?, ?)", ("Test1", 100))  
cursor_test.execute("INSERT INTO test_donnees (nom, valeur) VALUES (?, ?)", ("Test2", 200))  

conn_test.commit()  
conn_test.close()  

# Tester le gestionnaire de récupération
recuperation = GestionnaireRecuperation(base_test)

# Diagnostic
problemes = recuperation.diagnostiquer_probleme()

# Si pas de problèmes, on peut tester l'export
if not problemes:
    print("\n=== TEST EXPORT DONNÉES ===")
    recuperation.exporter_donnees_recuperables('export_test.db')

# Nettoyage (FileNotFoundError si déjà supprimé, pas besoin de paniquer)
for f in (base_test, 'export_test.db'):
    try:
        os.remove(f)
    except FileNotFoundError:
        pass
```

## Récupération point-in-time

### Système de journalisation pour récupération temporelle

```python
import sqlite3  
import json  
import time  
from datetime import datetime, timedelta  

class JournalisationPIT:
    """Point-in-Time Recovery pour SQLite"""

    def __init__(self, base_principale, repertoire_logs="/logs"):
        self.base_principale = base_principale
        self.repertoire_logs = repertoire_logs
        self.fichier_journal = f"{repertoire_logs}/journal_transactions.log"
        self.dernier_checkpoint = None

        os.makedirs(repertoire_logs, exist_ok=True)
        self._initialiser_journal()

    def _initialiser_journal(self):
        """Initialise le système de journalisation"""
        if not os.path.exists(self.fichier_journal):
            with open(self.fichier_journal, 'w') as f:
                f.write(f"# Journal SQLite PIT - Démarré le {datetime.now()}\n")

        print(f"📄 Journal initialisé: {self.fichier_journal}")

    def enregistrer_transaction(self, operation, table, donnees_avant=None, donnees_apres=None):
        """Enregistre une transaction dans le journal"""
        timestamp = datetime.now().isoformat()

        entree_journal = {
            'timestamp': timestamp,
            'operation': operation,
            'table': table,
            'donnees_avant': donnees_avant,
            'donnees_apres': donnees_apres,
            'transaction_id': int(time.time() * 1000000)  # Microseconds pour unicité
        }

        try:
            with open(self.fichier_journal, 'a') as f:
                f.write(json.dumps(entree_journal) + '\n')
        except Exception as e:
            print(f"⚠️ Erreur écriture journal: {e}")

    def creer_checkpoint(self, description="Checkpoint automatique"):
        """Crée un point de contrôle (snapshot complet)"""
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        fichier_checkpoint = f"{self.repertoire_logs}/checkpoint_{timestamp}.db"

        print(f"📸 Création du checkpoint: {description}")

        try:
            # Sauvegarde complète
            conn_source = sqlite3.connect(self.base_principale)
            conn_dest = sqlite3.connect(fichier_checkpoint)

            conn_source.backup(conn_dest)

            conn_dest.close()
            conn_source.close()

            # Enregistrer les métadonnées du checkpoint
            metadata_checkpoint = {
                'timestamp': datetime.now().isoformat(),
                'fichier': fichier_checkpoint,
                'description': description,
                'type': 'checkpoint'
            }

            self.dernier_checkpoint = metadata_checkpoint

            with open(self.fichier_journal, 'a') as f:
                f.write(json.dumps(metadata_checkpoint) + '\n')

            print(f"✅ Checkpoint créé: {fichier_checkpoint}")
            return fichier_checkpoint

        except Exception as e:
            print(f"❌ Erreur création checkpoint: {e}")
            return None

    def lister_checkpoints(self):
        """Liste tous les checkpoints disponibles"""
        checkpoints = []

        try:
            with open(self.fichier_journal, 'r') as f:
                for ligne in f:
                    if ligne.startswith('#'):  # Ignorer les commentaires
                        continue

                    try:
                        entree = json.loads(ligne.strip())
                        if entree.get('type') == 'checkpoint':
                            checkpoints.append(entree)
                    except json.JSONDecodeError:
                        continue

            return checkpoints

        except Exception as e:
            print(f"❌ Erreur lecture checkpoints: {e}")
            return []

    def recuperer_a_instant(self, timestamp_cible, fichier_restauration):
        """Récupère la base à un instant donné"""
        print(f"⏰ RÉCUPÉRATION POINT-IN-TIME")
        print(f"Timestamp cible: {timestamp_cible}")
        print(f"Fichier de sortie: {fichier_restauration}")
        print("=" * 50)

        # Convertir le timestamp si c'est une chaîne
        if isinstance(timestamp_cible, str):
            timestamp_cible = datetime.fromisoformat(timestamp_cible)

        # Trouver le checkpoint le plus récent avant le timestamp cible
        checkpoints = self.lister_checkpoints()
        checkpoint_base = None

        for cp in reversed(checkpoints):  # Plus récent en premier
            cp_time = datetime.fromisoformat(cp['timestamp'])
            if cp_time <= timestamp_cible:
                checkpoint_base = cp
                break

        if not checkpoint_base:
            print("❌ Aucun checkpoint trouvé avant le timestamp cible")
            return False

        print(f"📸 Checkpoint de base: {checkpoint_base['timestamp']}")

        # Copier le checkpoint comme base
        try:
            shutil.copy2(checkpoint_base['fichier'], fichier_restauration)
            print(f"✅ Checkpoint copié vers {fichier_restauration}")
        except Exception as e:
            print(f"❌ Erreur copie checkpoint: {e}")
            return False

        # Rejouer les transactions entre le checkpoint et le timestamp cible
        transactions_a_rejouer = []

        try:
            with open(self.fichier_journal, 'r') as f:
                for ligne in f:
                    if ligne.startswith('#'):
                        continue

                    try:
                        entree = json.loads(ligne.strip())

                        # Ignorer les checkpoints
                        if entree.get('type') == 'checkpoint':
                            continue

                        entree_time = datetime.fromisoformat(entree['timestamp'])
                        checkpoint_time = datetime.fromisoformat(checkpoint_base['timestamp'])

                        # Transaction entre checkpoint et cible
                        if checkpoint_time < entree_time <= timestamp_cible:
                            transactions_a_rejouer.append(entree)

                    except (json.JSONDecodeError, KeyError):
                        continue

            print(f"🔄 {len(transactions_a_rejouer)} transactions à rejouer")

            # Rejouer les transactions
            if transactions_a_rejouer:
                conn_restore = sqlite3.connect(fichier_restauration)
                cursor_restore = conn_restore.cursor()

                transactions_rejouees = 0
                erreurs_rejeu = 0

                for transaction in transactions_a_rejouer:
                    try:
                        self._rejouer_transaction(cursor_restore, transaction)
                        transactions_rejouees += 1
                    except Exception as e:
                        print(f"⚠️ Erreur rejeu transaction {transaction.get('transaction_id')}: {e}")
                        erreurs_rejeu += 1

                conn_restore.commit()
                conn_restore.close()

                print(f"✅ Transactions rejouées: {transactions_rejouees}")
                if erreurs_rejeu > 0:
                    print(f"⚠️ Erreurs de rejeu: {erreurs_rejeu}")

            # Vérifier l'intégrité du résultat
            conn_verifie = sqlite3.connect(fichier_restauration)
            cursor_verifie = conn_verifie.cursor()
            cursor_verifie.execute("PRAGMA integrity_check")
            lignes_integrite = [row[0] for row in cursor_verifie.fetchall()]
            conn_verifie.close()

            if lignes_integrite == ['ok']:
                print("✅ Récupération point-in-time réussie")
                return True
            else:
                print(f"❌ {len(lignes_integrite)} problème(s) d'intégrité :")
                for ligne in lignes_integrite[:5]:
                    print(f"   • {ligne}")
                return False

        except Exception as e:
            print(f"❌ Erreur lors de la récupération: {e}")
            return False

    def _rejouer_transaction(self, cursor, transaction):
        """Rejoue une transaction spécifique"""
        operation = transaction['operation']
        table = transaction['table']
        donnees_apres = transaction.get('donnees_apres')

        if operation == 'INSERT' and donnees_apres:
            # Reconstituer l'INSERT
            colonnes = list(donnees_apres.keys())
            valeurs = list(donnees_apres.values())
            placeholders = ','.join(['?' for _ in valeurs])

            sql = f"INSERT INTO {table} ({','.join(colonnes)}) VALUES ({placeholders})"
            cursor.execute(sql, valeurs)

        elif operation == 'UPDATE' and donnees_apres:
            # Reconstituer l'UPDATE (simplifié)
            # En production, il faudrait gérer les conditions WHERE
            colonnes = list(donnees_apres.keys())
            if 'id' in donnees_apres:
                set_clause = ','.join([f"{col}=?" for col in colonnes if col != 'id'])
                valeurs = [donnees_apres[col] for col in colonnes if col != 'id']
                valeurs.append(donnees_apres['id'])

                sql = f"UPDATE {table} SET {set_clause} WHERE id=?"
                cursor.execute(sql, valeurs)

        elif operation == 'DELETE':
            # Pour DELETE, on utiliserait normalement les données_avant
            # Ici c'est une implémentation simplifiée
            pass

# Utilisation de la journalisation PIT
print("=== DÉMONSTRATION POINT-IN-TIME RECOVERY ===")

# Créer une base de test
base_test = 'test_pit.db'  
conn = sqlite3.connect(base_test)  
cursor = conn.cursor()  

cursor.execute('''
    CREATE TABLE commandes (
        id INTEGER PRIMARY KEY,
        client TEXT,
        montant REAL,
        date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
''')

# Initialiser le système PIT
pit = JournalisationPIT(base_test, '/tmp/logs_pit')

# Créer un checkpoint initial
pit.creer_checkpoint("État initial vide")

# Simuler quelques transactions avec journalisation
transactions_test = [
    ('INSERT', 'commandes', None, {'id': 1, 'client': 'Alice', 'montant': 100.0}),
    ('INSERT', 'commandes', None, {'id': 2, 'client': 'Bob', 'montant': 200.0}),
    ('UPDATE', 'commandes', {'montant': 200.0}, {'id': 2, 'client': 'Bob', 'montant': 250.0}),
]

for op, table, avant, apres in transactions_test:
    # Enregistrer dans le journal
    pit.enregistrer_transaction(op, table, avant, apres)

    # Simuler un délai
    time.sleep(0.1)

print(f"\n📋 Checkpoints disponibles:")  
checkpoints = pit.lister_checkpoints()  
for cp in checkpoints:  
    print(f"   • {cp['timestamp']}: {cp['description']}")

# Test de récupération
timestamp_milieu = datetime.now() - timedelta(seconds=5)  
fichier_recupere = '/tmp/base_recuperee.db'  

# pit.recuperer_a_instant(timestamp_milieu, fichier_recupere)

conn.close()

# Nettoyage (FileNotFoundError attrapé spécifiquement, pas except: nu)
try:
    os.remove(base_test)
    # os.remove(fichier_recupere)
except FileNotFoundError:
    pass

print("✅ Démonstration PIT terminée")
```

## Tests et validation des sauvegardes

### Suite de tests automatisés

```python
import unittest  
import tempfile  
import os  
import sqlite3  
from datetime import datetime, timedelta  

class TestSauvegardes(unittest.TestCase):
    def setUp(self):
        """Prépare l'environnement de test"""
        self.temp_dir = tempfile.mkdtemp()
        self.base_test = os.path.join(self.temp_dir, 'test.db')
        self.backup_dir = os.path.join(self.temp_dir, 'backups')

        os.makedirs(self.backup_dir, exist_ok=True)

        # Créer une base de test avec des données
        self._creer_base_test()

    def tearDown(self):
        """Nettoie après les tests"""
        import shutil
        shutil.rmtree(self.temp_dir, ignore_errors=True)

    def _creer_base_test(self):
        """Crée une base de test avec des données"""
        conn = sqlite3.connect(self.base_test)
        cursor = conn.cursor()

        cursor.execute('''
            CREATE TABLE clients (
                id INTEGER PRIMARY KEY,
                nom TEXT NOT NULL,
                email TEXT UNIQUE,
                solde REAL DEFAULT 0.0
            )
        ''')

        cursor.execute('''
            CREATE TABLE commandes (
                id INTEGER PRIMARY KEY,
                client_id INTEGER,
                montant REAL,
                date_commande TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (client_id) REFERENCES clients(id)
            )
        ''')

        # Insérer des données de test
        for i in range(100):
            cursor.execute(
                "INSERT INTO clients (nom, email, solde) VALUES (?, ?, ?)",
                (f"Client{i}", f"client{i}@test.com", i * 10.0)
            )

        for i in range(200):
            cursor.execute(
                "INSERT INTO commandes (client_id, montant) VALUES (?, ?)",
                ((i % 100) + 1, (i + 1) * 5.0)
            )

        conn.commit()
        conn.close()

    def test_sauvegarde_api_sqlite(self):
        """Test de la méthode de sauvegarde API SQLite"""
        gestionnaire = GestionnaireSauvegarde(self.base_test)
        fichier_backup = os.path.join(self.backup_dir, 'test_api.db')

        # Effectuer la sauvegarde
        resultat = gestionnaire.sauvegarde_sqlite_api(fichier_backup)

        # Vérifications
        self.assertTrue(resultat, "La sauvegarde devrait réussir")
        self.assertTrue(os.path.exists(fichier_backup), "Le fichier de sauvegarde devrait exister")

        # Vérifier que la sauvegarde contient les bonnes données
        conn_backup = sqlite3.connect(fichier_backup)
        cursor_backup = conn_backup.cursor()

        cursor_backup.execute("SELECT COUNT(*) FROM clients")
        nb_clients = cursor_backup.fetchone()[0]
        self.assertEqual(nb_clients, 100, "La sauvegarde devrait contenir 100 clients")

        cursor_backup.execute("SELECT COUNT(*) FROM commandes")
        nb_commandes = cursor_backup.fetchone()[0]
        self.assertEqual(nb_commandes, 200, "La sauvegarde devrait contenir 200 commandes")

        conn_backup.close()

    def test_sauvegarde_avec_verification(self):
        """Test de la sauvegarde avec vérification d'intégrité"""
        gestionnaire = GestionnaireSauvegarde(self.base_test)
        fichier_backup = os.path.join(self.backup_dir, 'test_verifie.db')

        resultat = gestionnaire.sauvegarde_avec_verification(fichier_backup)

        self.assertTrue(resultat, "La sauvegarde avec vérification devrait réussir")
        self.assertTrue(os.path.exists(fichier_backup), "Le fichier de sauvegarde devrait exister")

    def test_sauvegarde_incrementielle(self):
        """Test de la sauvegarde incrémentielle"""
        gestionnaire = GestionnaireSauvegarde(self.base_test)

        # Première sauvegarde
        fichier_backup1 = os.path.join(self.backup_dir, 'incrementiel1.db')
        resultat1 = gestionnaire.sauvegarde_incrementielle(fichier_backup1)
        self.assertTrue(resultat1, "Première sauvegarde incrémentielle devrait réussir")

        # Deuxième sauvegarde immédiate (aucun changement)
        fichier_backup2 = os.path.join(self.backup_dir, 'incrementiel2.db')
        resultat2 = gestionnaire.sauvegarde_incrementielle(fichier_backup2, fichier_backup1)
        self.assertTrue(resultat2, "Sauvegarde incrémentielle sans changement devrait réussir")

    def test_diagnostic_base_saine(self):
        """Test de diagnostic sur une base saine"""
        recuperation = GestionnaireRecuperation(self.base_test)
        problemes = recuperation.diagnostiquer_probleme()

        self.assertEqual(len(problemes), 0, "Une base saine ne devrait avoir aucun problème")

    def test_export_donnees_recuperables(self):
        """Test d'export des données récupérables"""
        recuperation = GestionnaireRecuperation(self.base_test)
        fichier_export = os.path.join(self.backup_dir, 'export_test.db')

        resultat = recuperation.exporter_donnees_recuperables(fichier_export)

        self.assertTrue(resultat, "L'export devrait réussir")
        self.assertTrue(os.path.exists(fichier_export), "Le fichier d'export devrait exister")

        # Vérifier le contenu exporté
        conn_export = sqlite3.connect(fichier_export)
        cursor_export = conn_export.cursor()

        cursor_export.execute("SELECT COUNT(*) FROM clients")
        nb_clients = cursor_export.fetchone()[0]
        self.assertEqual(nb_clients, 100, "Tous les clients devraient être exportés")

        conn_export.close()

    def test_restauration_depuis_sauvegarde(self):
        """Test de restauration depuis une sauvegarde"""
        # Créer une sauvegarde
        gestionnaire = GestionnaireSauvegarde(self.base_test)
        fichier_backup = os.path.join(self.backup_dir, 'test_restauration.db')
        gestionnaire.sauvegarde_sqlite_api(fichier_backup)

        # Modifier la base originale
        conn = sqlite3.connect(self.base_test)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM clients WHERE id <= 50")
        conn.commit()
        conn.close()

        # Vérifier que la modification a eu lieu
        conn = sqlite3.connect(self.base_test)
        cursor = conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM clients")
        nb_clients_apres = cursor.fetchone()[0]
        conn.close()
        self.assertEqual(nb_clients_apres, 50, "50 clients devraient avoir été supprimés")

        # Restaurer depuis la sauvegarde
        recuperation = GestionnaireRecuperation(self.base_test)
        resultat = recuperation.restaurer_depuis_sauvegarde(fichier_backup, remplacer=True)

        self.assertTrue(resultat, "La restauration devrait réussir")

        # Vérifier que les données sont restaurées
        conn = sqlite3.connect(self.base_test)
        cursor = conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM clients")
        nb_clients_restaure = cursor.fetchone()[0]
        conn.close()
        self.assertEqual(nb_clients_restaure, 100, "Tous les clients devraient être restaurés")

    def test_planificateur_sauvegarde(self):
        """Test du planificateur de sauvegardes"""
        planificateur = PlanificateurSauvegarde(self.base_test, self.backup_dir)

        # Test sauvegarde quotidienne
        resultat = planificateur.sauvegarde_quotidienne()
        self.assertTrue(resultat, "La sauvegarde quotidienne devrait réussir")

        # Vérifier l'historique
        self.assertEqual(len(planificateur.historique_sauvegardes), 1,
                        "L'historique devrait contenir une sauvegarde")

        # Vérifier que le fichier existe
        derniere_sauvegarde = planificateur.historique_sauvegardes[-1]
        self.assertTrue(derniere_sauvegarde['succes'], "La sauvegarde devrait avoir réussi")

class TestValidationSauvegardes:
    """Tests de validation approfondis des sauvegardes"""

    def __init__(self, base_originale, fichier_sauvegarde):
        self.base_originale = base_originale
        self.fichier_sauvegarde = fichier_sauvegarde

    def valider_completude(self):
        """Valide que la sauvegarde contient toutes les données"""
        print("🔍 Validation de la complétude...")

        erreurs = []

        try:
            conn_orig = sqlite3.connect(self.base_originale)
            conn_backup = sqlite3.connect(self.fichier_sauvegarde)

            cursor_orig = conn_orig.cursor()
            cursor_backup = conn_backup.cursor()

            # Comparer le nombre de tables
            cursor_orig.execute("SELECT COUNT(*) FROM sqlite_master WHERE type='table'")
            nb_tables_orig = cursor_orig.fetchone()[0]

            cursor_backup.execute("SELECT COUNT(*) FROM sqlite_master WHERE type='table'")
            nb_tables_backup = cursor_backup.fetchone()[0]

            if nb_tables_orig != nb_tables_backup:
                erreurs.append(f"Nombre de tables différent: {nb_tables_orig} vs {nb_tables_backup}")

            # Comparer table par table
            cursor_orig.execute("SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%'")
            tables = cursor_orig.fetchall()

            for (nom_table,) in tables:
                # Comparer le nombre d'enregistrements
                cursor_orig.execute(f"SELECT COUNT(*) FROM '{nom_table}'")
                count_orig = cursor_orig.fetchone()[0]

                cursor_backup.execute(f"SELECT COUNT(*) FROM '{nom_table}'")
                count_backup = cursor_backup.fetchone()[0]

                if count_orig != count_backup:
                    erreurs.append(f"Table {nom_table}: {count_orig} vs {count_backup} enregistrements")

                # Comparer un checksum colonne par colonne.
                # ⚠️ `CAST(* AS TEXT)` n'est PAS du SQL valide. On construit
                #    la requête à partir des colonnes réelles via PRAGMA table_info.
                #
                # ⚠️ PIÈGE — `SUM(LENGTH(...))` SEUL est trompeur :
                #    'Client_1' (8 chars) et 'MODIFIED' (8 chars) ont la même
                #    longueur → le checksum ne détecte PAS la modification !
                #    Pour un vrai contrôle, on combine plusieurs agrégats
                #    (MIN, MAX, SUM, COUNT) qui dépendent du contenu réel.
                #    L'idéal serait un hash cryptographique (SHA-256), mais
                #    SQLite n'a pas de fonction de hash native — il faudrait
                #    charger une extension ou faire le hash côté Python.
                try:
                    cursor_orig.execute(f"PRAGMA table_info('{nom_table}')")
                    colonnes = [row[1] for row in cursor_orig.fetchall()]
                    if not colonnes:
                        continue

                    # Combiner COUNT, SUM(LENGTH), MIN, MAX par colonne →
                    # détecte les modifications de longueur ET de contenu
                    # (sauf permutations de valeurs, qui restent un cas limite).
                    agregats = []
                    for c in colonnes:
                        col_quote = f'"{c}"'
                        agregats.extend([
                            f"SUM(LENGTH(IFNULL(CAST({col_quote} AS TEXT), '')))",
                            f"MIN(CAST({col_quote} AS TEXT))",
                            f"MAX(CAST({col_quote} AS TEXT))",
                        ])
                    requete_checksum = (
                        f"SELECT COUNT(*), {', '.join(agregats)} "
                        f"FROM \"{nom_table}\""
                    )

                    cursor_orig.execute(requete_checksum)
                    checksum_orig = cursor_orig.fetchone()
                    cursor_backup.execute(requete_checksum)
                    checksum_backup = cursor_backup.fetchone()

                    if checksum_orig != checksum_backup:
                        erreurs.append(f"Checksum différent pour {nom_table}")
                except Exception:
                    # Certaines tables peuvent ne pas supporter cette méthode
                    pass

            conn_orig.close()
            conn_backup.close()

            if erreurs:
                print("❌ Erreurs de complétude détectées:")
                for erreur in erreurs:
                    print(f"   • {erreur}")
                return False
            else:
                print("✅ Complétude validée")
                return True

        except Exception as e:
            print(f"❌ Erreur lors de la validation: {e}")
            return False

    def valider_integrite(self):
        """Valide l'intégrité de la sauvegarde"""
        print("🔍 Validation de l'intégrité...")

        try:
            conn_backup = sqlite3.connect(self.fichier_sauvegarde)
            cursor_backup = conn_backup.cursor()

            # Test d'intégrité SQLite
            cursor_backup.execute("PRAGMA integrity_check")
            resultats = cursor_backup.fetchall()

            if len(resultats) == 1 and resultats[0][0] == 'ok':
                print("✅ Intégrité SQLite: OK")
                integrite_ok = True
            else:
                print("❌ Problèmes d'intégrité SQLite:")
                for resultat in resultats[:5]:
                    print(f"   • {resultat[0]}")
                integrite_ok = False

            # Test des clés étrangères
            cursor_backup.execute("PRAGMA foreign_key_check")
            violations_fk = cursor_backup.fetchall()

            if violations_fk:
                print(f"⚠️ {len(violations_fk)} violations de clés étrangères")
                for violation in violations_fk[:5]:
                    print(f"   • Table {violation[0]}: violation ligne {violation[1]}")
                integrite_ok = False
            else:
                print("✅ Clés étrangères: OK")

            conn_backup.close()
            return integrite_ok

        except Exception as e:
            print(f"❌ Erreur validation intégrité: {e}")
            return False

    def valider_performance(self):
        """Valide que la sauvegarde a des performances acceptables"""
        print("🔍 Validation des performances...")

        try:
            import time

            conn_backup = sqlite3.connect(self.fichier_sauvegarde)
            cursor_backup = conn_backup.cursor()

            # Test de performance simple
            debut = time.time()
            cursor_backup.execute("SELECT COUNT(*) FROM sqlite_master")
            duree_metadata = time.time() - debut

            # Test sur une requête plus complexe si possible
            cursor_backup.execute("SELECT name FROM sqlite_master WHERE type='table' LIMIT 1")
            table_test = cursor_backup.fetchone()

            if table_test:
                debut = time.time()
                cursor_backup.execute(f"SELECT COUNT(*) FROM '{table_test[0]}'")
                duree_count = time.time() - debut

                print(f"⚡ Temps de réponse métadonnées: {duree_metadata:.3f}s")
                print(f"⚡ Temps de réponse COUNT: {duree_count:.3f}s")

                # Seuils de performance (ajustables)
                if duree_metadata > 1.0 or duree_count > 5.0:
                    print("⚠️ Performances dégradées détectées")
                    return False
                else:
                    print("✅ Performances acceptables")
                    return True

            conn_backup.close()

        except Exception as e:
            print(f"❌ Erreur test performance: {e}")
            return False

    def rapport_validation_complet(self):
        """Génère un rapport de validation complet"""
        print("\n📋 === RAPPORT DE VALIDATION SAUVEGARDE ===")
        print(f"Base originale: {self.base_originale}")
        print(f"Sauvegarde: {self.fichier_sauvegarde}")
        print(f"Date validation: {datetime.now()}")
        print("=" * 60)

        resultats = {
            'completude': self.valider_completude(),
            'integrite': self.valider_integrite(),
            'performance': self.valider_performance()
        }

        # Calcul du score global
        score = sum(resultats.values())
        score_pourcentage = (score / len(resultats)) * 100

        print(f"\n📊 SCORE GLOBAL: {score}/{len(resultats)} ({score_pourcentage:.1f}%)")

        if score == len(resultats):
            print("🎉 Sauvegarde VALIDÉE - Prête pour utilisation")
            statut = "VALIDEE"
        elif score >= len(resultats) * 0.67:  # 67% minimum
            print("⚠️ Sauvegarde ACCEPTABLE - Quelques problèmes mineurs")
            statut = "ACCEPTABLE"
        else:
            print("❌ Sauvegarde NON FIABLE - Ne pas utiliser")
            statut = "NON_FIABLE"

        return {
            'statut': statut,
            'score': score_pourcentage,
            'details': resultats,
            'timestamp': datetime.now().isoformat()
        }

# Exécution des tests
if __name__ == '__main__':
    print("🧪 === SUITE DE TESTS SAUVEGARDES SQLITE ===\n")

    # Tests unitaires
    print("📋 Exécution des tests unitaires...")
    suite = unittest.TestLoader().loadTestsFromTestCase(TestSauvegardes)
    runner = unittest.TextTestRunner(verbosity=2)
    resultat_tests = runner.run(suite)

    # Résumé des tests
    print(f"\n📊 === RÉSUMÉ TESTS UNITAIRES ===")
    print(f"Tests exécutés: {resultat_tests.testsRun}")
    print(f"Succès: {resultat_tests.testsRun - len(resultat_tests.failures) - len(resultat_tests.errors)}")
    print(f"Échecs: {len(resultat_tests.failures)}")
    print(f"Erreurs: {len(resultat_tests.errors)}")

    if resultat_tests.failures:
        print("\n❌ ÉCHECS:")
        for test, traceback in resultat_tests.failures:
            print(f"- {test}")

    if resultat_tests.errors:
        print("\n💥 ERREURS:")
        for test, traceback in resultat_tests.errors:
            print(f"- {test}")

    # Test de validation sur un exemple
    print(f"\n🔍 === TEST DE VALIDATION EXEMPLE ===")

    # Créer une base temporaire pour la démonstration
    import tempfile
    temp_dir = tempfile.mkdtemp()
    base_demo = os.path.join(temp_dir, 'demo.db')
    backup_demo = os.path.join(temp_dir, 'demo_backup.db')

    # Créer base de démo
    conn_demo = sqlite3.connect(base_demo)
    cursor_demo = conn_demo.cursor()
    cursor_demo.execute("CREATE TABLE test (id INTEGER PRIMARY KEY, data TEXT)")
    cursor_demo.execute("INSERT INTO test (data) VALUES ('test1'), ('test2'), ('test3')")
    conn_demo.commit()
    conn_demo.close()

    # Créer sauvegarde
    gestionnaire_demo = GestionnaireSauvegarde(base_demo)
    gestionnaire_demo.sauvegarde_sqlite_api(backup_demo)

    # Valider la sauvegarde
    validateur = TestValidationSauvegardes(base_demo, backup_demo)
    rapport = validateur.rapport_validation_complet()

    print(f"\n✅ Statut final: {rapport['statut']} ({rapport['score']:.1f}%)")

    # Nettoyage
    import shutil
    shutil.rmtree(temp_dir, ignore_errors=True)
```

## Documentation et procédures opérationnelles

### Guide de procédures d'urgence

```markdown
# 🚨 PROCÉDURES D'URGENCE SQLITE - GUIDE DE RÉCUPÉRATION

## ⚡ ACTIONS IMMÉDIATES EN CAS DE PROBLÈME

### 1. Évaluation rapide (< 2 minutes)
```bash
# Vérifier si le fichier existe
ls -la ma_base.db

# Taille du fichier
du -h ma_base.db

# Test d'ouverture basique
sqlite3 ma_base.db ".tables"

# Intégrité rapide
sqlite3 ma_base.db "PRAGMA quick_check;"
```

### 2. Sauvegarde d'urgence (< 5 minutes)
```bash
# ⚠️ ORDRE IMPORTANT : essayer d'abord la sauvegarde SÛRE (`.backup`).
#    Le `cp` direct n'est acceptable QUE si la base est inaccessible
#    (ex : `sqlite3 .backup` échoue avec "database is locked") — dans ce cas
#    on prend un snapshot "best-effort" pour analyse forensique.

# 1. Méthode SÛRE — préférée
sqlite3 ma_base.db ".backup 'ma_base.db.BACKUP_URGENCE.$(date +%Y%m%d_%H%M%S)'" \
    || \
# 2. Fallback `cp` si `.backup` échoue (snapshot potentiellement incohérent)
cp ma_base.db ma_base.db.URGENCE.$(date +%Y%m%d_%H%M%S)
```

### 3. Diagnostic approfondi (< 10 minutes)
```python
# Utiliser le gestionnaire de récupération
from gestion_recuperation import GestionnaireRecuperation

recuperation = GestionnaireRecuperation('ma_base.db')  
problemes = recuperation.diagnostiquer_probleme()  

if problemes:
    print("🚨 PROBLÈMES DÉTECTÉS:")
    for p in problemes:
        print(f"   • {p}")
```

## 🔧 PROCÉDURES DE RÉCUPÉRATION PAR SCENARIO

### Scenario A: Base de données corrompue
```bash
# 1. Arrêter immédiatement l'application
sudo systemctl stop mon-application

# 2. Sauvegarde d'urgence
cp ma_base.db ma_base.db.CORRUPT.$(date +%Y%m%d_%H%M%S)

# 3. Tentative de réparation
sqlite3 ma_base.db "VACUUM;"  
sqlite3 ma_base.db "REINDEX;"  

# 4. Vérification
sqlite3 ma_base.db "PRAGMA integrity_check;"

# 5. Si échec: restaurer depuis sauvegarde
cp /backup/derniere_sauvegarde.db ma_base.db
```

### Scenario B: Fichier de base manquant
```bash
# 1. Rechercher des copies
find /var -name "*.db" -mtime -1 2>/dev/null | grep ma_base

# 2. Vérifier les sauvegardes
ls -la /backup/*.db

# 3. Restaurer la plus récente
cp /backup/ma_base_YYYYMMDD_HHMMSS.db ma_base.db

# 4. Vérifier l'intégrité
sqlite3 ma_base.db "PRAGMA integrity_check;"
```

### Scenario C: Performance dégradée
```bash
# 1. Vérifier l'espace disque
df -h

# 2. Analyser la base
sqlite3 ma_base.db "PRAGMA analysis_limit=1000; PRAGMA optimize;"

# 3. Nettoyer si nécessaire
sqlite3 ma_base.db "VACUUM;"

# 4. Redémarrer l'application
sudo systemctl restart mon-application
```

## 📞 ESCALADE ET CONTACTS

### Niveau 1: Problème mineur (< 1h d'impact)
- Auto-résolution avec procédures standard
- Log dans /var/log/sqlite_incidents.log

### Niveau 2: Problème majeur (> 1h d'impact)
- Contacter: admin-db@entreprise.com
- Téléphone: +33 1 XX XX XX XX

### Niveau 3: Désastre (perte de données)
- Contacter immédiatement: urgence-it@entreprise.com
- Téléphone: +33 6 XX XX XX XX (24h/24)
- Activer le plan de continuité d'activité

## 📋 CHECKLIST POST-INCIDENT

□ Problème résolu et service restauré
□ Cause racine identifiée
□ Documentation mise à jour
□ Formation équipe si nécessaire
□ Procédures améliorées
□ Tests de non-régression effectués
```

### Script de maintenance automatisé

```python
#!/usr/bin/env python3
"""
Script de maintenance automatisé SQLite
À exécuter quotidiennement via cron
"""

import os  
import sys  
import sqlite3  
import shutil  
import logging  
from datetime import datetime, timedelta  
import argparse  
import json  

class MaintenanceAutomatisee:
    def __init__(self, config_file='maintenance_config.json'):
        self.config = self._charger_configuration(config_file)
        self._configurer_logging()

    def _charger_configuration(self, config_file):
        """Charge la configuration depuis un fichier JSON"""
        config_defaut = {
            "bases_a_maintenir": [
                {"chemin": "/app/data/production.db", "priorite": "haute"},
                {"chemin": "/app/data/analytics.db", "priorite": "moyenne"}
            ],
            "backup_directory": "/backup/sqlite",
            "retention_jours": 30,
            "seuil_taille_vacuum_mb": 100,
            "notification_email": "admin@entreprise.com",
            "maintenance_window": {
                "debut": "02:00",
                "fin": "04:00"
            }
        }

        def _fusion_profonde(defaut, surcouche):
            """Fusion récursive : préserve les sous-dictionnaires par défaut
            que l'utilisateur ne redéfinit pas explicitement."""
            if not isinstance(surcouche, dict):
                return surcouche if surcouche is not None else defaut
            resultat = dict(defaut)
            for cle, valeur in surcouche.items():
                if cle in resultat and isinstance(resultat[cle], dict):
                    resultat[cle] = _fusion_profonde(resultat[cle], valeur)
                else:
                    resultat[cle] = valeur
            return resultat

        try:
            with open(config_file, 'r') as f:
                config_fichier = json.load(f) or {}
                # ⚠️ Fusion superficielle = sous-dictionnaires (ex.
                #    `maintenance_window`) entièrement remplacés au lieu
                #    d'être complétés. Voir fusion récursive ci-dessus.
                return _fusion_profonde(config_defaut, config_fichier)
        except FileNotFoundError:
            print(f"⚠️ Config non trouvée, utilisation config par défaut")
            return config_defaut

    def _configurer_logging(self):
        """Configure le système de logging"""
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler('/var/log/sqlite_maintenance.log'),
                logging.StreamHandler(sys.stdout)
            ]
        )
        self.logger = logging.getLogger(__name__)

    def verifier_fenetre_maintenance(self):
        """Vérifie si nous sommes dans la fenêtre de maintenance"""
        maintenant = datetime.now().time()
        debut = datetime.strptime(self.config['maintenance_window']['debut'], '%H:%M').time()
        fin = datetime.strptime(self.config['maintenance_window']['fin'], '%H:%M').time()

        if debut <= maintenant <= fin:
            return True
        else:
            self.logger.warning(f"Hors fenêtre de maintenance ({debut}-{fin})")
            return False

    def maintenance_base(self, chemin_base, priorite="moyenne"):
        """Effectue la maintenance d'une base spécifique"""
        self.logger.info(f"🔧 Début maintenance: {chemin_base} (priorité: {priorite})")

        resultats = {
            'base': chemin_base,
            'timestamp': datetime.now().isoformat(),
            'operations': [],
            'erreurs': [],
            'statut': 'success'
        }

        try:
            # 1. Vérification d'intégrité
            self.logger.info("🔍 Vérification d'intégrité...")
            if self._verifier_integrite(chemin_base):
                resultats['operations'].append('integrity_check_ok')
            else:
                resultats['erreurs'].append('integrity_check_failed')
                resultats['statut'] = 'warning'

            # 2. Analyse de la taille pour VACUUM
            taille_mb = os.path.getsize(chemin_base) / (1024 * 1024)
            self.logger.info(f"📏 Taille actuelle: {taille_mb:.1f} MB")

            if taille_mb > self.config['seuil_taille_vacuum_mb']:
                self.logger.info("🗜️ VACUUM en cours...")
                if self._executer_vacuum(chemin_base):
                    resultats['operations'].append('vacuum_executed')

                    # Mesurer la réduction de taille
                    nouvelle_taille = os.path.getsize(chemin_base) / (1024 * 1024)
                    reduction = taille_mb - nouvelle_taille
                    self.logger.info(f"✅ VACUUM terminé - {reduction:.1f} MB libérés")
                else:
                    resultats['erreurs'].append('vacuum_failed')

            # 3. Optimisation
            self.logger.info("⚡ Optimisation en cours...")
            if self._optimiser_base(chemin_base):
                resultats['operations'].append('optimize_executed')

            # 4. Sauvegarde (priorité haute seulement)
            if priorite == "haute":
                self.logger.info("💾 Sauvegarde priorité haute...")
                if self._creer_sauvegarde(chemin_base):
                    resultats['operations'].append('backup_created')
                else:
                    resultats['erreurs'].append('backup_failed')
                    resultats['statut'] = 'error'

            # 5. Nettoyage des anciens WAL/SHM
            self._nettoyer_fichiers_temporaires(chemin_base)
            resultats['operations'].append('temp_files_cleaned')

        except Exception as e:
            self.logger.error(f"❌ Erreur maintenance {chemin_base}: {e}")
            resultats['erreurs'].append(str(e))
            resultats['statut'] = 'error'

        self.logger.info(f"✅ Maintenance terminée: {chemin_base}")
        return resultats

    def _verifier_integrite(self, chemin_base):
        """Vérifie l'intégrité d'une base (toutes lignes d'integrity_check)"""
        try:
            conn = sqlite3.connect(chemin_base)
            cursor = conn.cursor()
            cursor.execute("PRAGMA integrity_check")
            lignes_integrite = [row[0] for row in cursor.fetchall()]
            conn.close()
            if lignes_integrite != ['ok']:
                self.logger.warning(
                    f"Intégrité KO sur {chemin_base} : "
                    f"{len(lignes_integrite)} signalement(s) — "
                    f"premier : {lignes_integrite[0]}"
                )
                return False
            return True
        except Exception as e:
            self.logger.error(f"Erreur vérification intégrité: {e}")
            return False

    def _executer_vacuum(self, chemin_base):
        """Exécute VACUUM sur une base (mode autocommit obligatoire)"""
        try:
            # ⚠️ isolation_level=None : VACUUM ne peut PAS s'exécuter dans
            #    une transaction (ouverte implicitement par le module sqlite3
            #    en mode par défaut). Sans cela : "cannot VACUUM within a transaction".
            conn = sqlite3.connect(chemin_base, isolation_level=None)
            conn.execute("VACUUM")
            conn.close()
            return True
        except Exception as e:
            self.logger.error(f"Erreur VACUUM: {e}")
            return False

    def _optimiser_base(self, chemin_base):
        """Optimise une base de données"""
        try:
            conn = sqlite3.connect(chemin_base)
            conn.execute("PRAGMA optimize")
            conn.close()
            return True
        except Exception as e:
            self.logger.error(f"Erreur optimisation: {e}")
            return False

    def _creer_sauvegarde(self, chemin_base):
        """Crée une sauvegarde de la base"""
        try:
            timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
            nom_base = os.path.basename(chemin_base)
            fichier_backup = os.path.join(
                self.config['backup_directory'],
                f"{nom_base}.backup_{timestamp}.db"
            )

            os.makedirs(self.config['backup_directory'], exist_ok=True)

            conn_source = sqlite3.connect(chemin_base)
            conn_dest = sqlite3.connect(fichier_backup)
            conn_source.backup(conn_dest)
            conn_dest.close()
            conn_source.close()

            # Comprimer la sauvegarde
            import gzip
            with open(fichier_backup, 'rb') as f_in:
                with gzip.open(f"{fichier_backup}.gz", 'wb') as f_out:
                    f_out.writelines(f_in)
            os.remove(fichier_backup)

            self.logger.info(f"💾 Sauvegarde créée: {fichier_backup}.gz")
            return True

        except Exception as e:
            self.logger.error(f"Erreur sauvegarde: {e}")
            return False

    def _nettoyer_fichiers_temporaires(self, chemin_base):
        """Effectue un checkpoint WAL pour réduire la taille du journal.

        ⚠️ DANGER ABSOLU : ne JAMAIS supprimer manuellement les fichiers
           `-wal` et `-shm`. En mode WAL, le fichier `-wal` contient des
           transactions COMMITÉES qui n'ont pas encore été appliquées au
           fichier principal. Le supprimer = perte de données silencieuse.

        La bonne pratique est d'exécuter un checkpoint TRUNCATE qui :
          - force l'écriture des pages WAL dans le fichier principal,
          - tronque le fichier `-wal` à zéro octet (mais ne le supprime pas),
          - laisse SQLite gérer le fichier `-shm` lui-même.
        """
        try:
            conn = sqlite3.connect(chemin_base)
            cursor = conn.cursor()
            # Le résultat est (busy, log_pages, checkpointed_pages)
            busy, log_pages, checkpointed = cursor.execute(
                "PRAGMA wal_checkpoint(TRUNCATE)"
            ).fetchone()
            conn.close()

            if busy == 0:
                self.logger.debug(
                    f"🧹 Checkpoint WAL effectué sur {chemin_base} "
                    f"({checkpointed} pages traitées)"
                )
            else:
                self.logger.warning(
                    f"⚠️ Checkpoint WAL bloqué sur {chemin_base} "
                    f"(transactions en cours)"
                )
        except Exception as e:
            self.logger.warning(f"Impossible de checkpointer {chemin_base}: {e}")

    def nettoyer_anciennes_sauvegardes(self):
        """Supprime les sauvegardes anciennes"""
        self.logger.info("🧹 Nettoyage des anciennes sauvegardes...")

        repertoire_backup = self.config['backup_directory']
        retention_jours = self.config['retention_jours']

        if not os.path.exists(repertoire_backup):
            return

        date_limite = datetime.now() - timedelta(days=retention_jours)
        fichiers_supprimes = 0

        for fichier in os.listdir(repertoire_backup):
            chemin_fichier = os.path.join(repertoire_backup, fichier)

            if os.path.isfile(chemin_fichier) and fichier.endswith('.gz'):
                date_modif = datetime.fromtimestamp(os.path.getmtime(chemin_fichier))

                if date_modif < date_limite:
                    try:
                        os.remove(chemin_fichier)
                        fichiers_supprimes += 1
                        self.logger.debug(f"Sauvegarde ancienne supprimée: {fichier}")
                    except Exception as e:
                        self.logger.warning(f"Impossible de supprimer {fichier}: {e}")

        if fichiers_supprimes > 0:
            self.logger.info(f"🧹 {fichiers_supprimes} anciennes sauvegardes supprimées")
        else:
            self.logger.info("ℹ️ Aucune sauvegarde ancienne à supprimer")

    def generer_rapport(self, resultats_maintenance):
        """Génère un rapport de maintenance"""
        self.logger.info("📊 Génération du rapport de maintenance...")

        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        fichier_rapport = f"/var/log/maintenance_report_{timestamp}.json"

        rapport = {
            'timestamp': datetime.now().isoformat(),
            'version_script': '1.0',
            'config_utilisee': self.config,
            'resultats': resultats_maintenance,
            'resume': self._calculer_resume(resultats_maintenance)
        }

        try:
            with open(fichier_rapport, 'w') as f:
                json.dump(rapport, f, indent=2, default=str)

            self.logger.info(f"📄 Rapport sauvegardé: {fichier_rapport}")
            return fichier_rapport

        except Exception as e:
            self.logger.error(f"Erreur génération rapport: {e}")
            return None

    def _calculer_resume(self, resultats):
        """Calcule un résumé des résultats de maintenance"""
        total_bases = len(resultats)
        succes = sum(1 for r in resultats if r['statut'] == 'success')
        warnings = sum(1 for r in resultats if r['statut'] == 'warning')
        erreurs = sum(1 for r in resultats if r['statut'] == 'error')

        total_operations = sum(len(r['operations']) for r in resultats)
        total_erreurs = sum(len(r['erreurs']) for r in resultats)

        return {
            'bases_total': total_bases,
            'bases_succes': succes,
            'bases_warnings': warnings,
            'bases_erreurs': erreurs,
            'taux_succes': (succes / total_bases * 100) if total_bases > 0 else 0,
            'operations_total': total_operations,
            'erreurs_total': total_erreurs
        }

    def envoyer_notification(self, rapport_fichier):
        """Envoie une notification par email"""
        try:
            import smtplib
            from email.mime.text import MIMEText
            from email.mime.multipart import MIMEMultipart
            from email.mime.base import MIMEBase
            from email import encoders

            # Lire le rapport
            with open(rapport_fichier, 'r') as f:
                rapport = json.load(f)

            resume = rapport['resume']

            # Créer le message
            msg = MIMEMultipart()
            msg['From'] = "maintenance-sqlite@entreprise.com"
            msg['To'] = self.config['notification_email']
            msg['Subject'] = f"Rapport Maintenance SQLite - {datetime.now().strftime('%Y-%m-%d')}"

            # Corps du message
            body = f"""
Rapport de maintenance SQLite automatisée

📊 RÉSUMÉ:
• Bases traitées: {resume['bases_total']}
• Succès: {resume['bases_succes']} ({resume['taux_succes']:.1f}%)
• Avertissements: {resume['bases_warnings']}
• Erreurs: {resume['bases_erreurs']}

• Total opérations: {resume['operations_total']}
• Total erreurs: {resume['erreurs_total']}

📋 DÉTAILS:
"""

            for resultat in rapport['resultats']:
                status_emoji = "✅" if resultat['statut'] == 'success' else "⚠️" if resultat['statut'] == 'warning' else "❌"
                body += f"\n{status_emoji} {resultat['base']}"
                if resultat['operations']:
                    body += f"\n   Opérations: {', '.join(resultat['operations'])}"
                if resultat['erreurs']:
                    body += f"\n   Erreurs: {', '.join(resultat['erreurs'])}"

            body += f"\n\nRapport complet en pièce jointe.\n\nMaintenance automatisée SQLite"

            msg.attach(MIMEText(body, 'plain'))

            # Attacher le rapport JSON
            with open(rapport_fichier, "rb") as attachment:
                part = MIMEBase('application', 'octet-stream')
                part.set_payload(attachment.read())

            encoders.encode_base64(part)
            part.add_header(
                'Content-Disposition',
                f'attachment; filename= {os.path.basename(rapport_fichier)}'
            )
            msg.attach(part)

            # Envoyer (configuration SMTP à adapter)
            # server = smtplib.SMTP('localhost', 587)
            # server.sendmail(msg['From'], msg['To'], msg.as_string())
            # server.quit()

            self.logger.info(f"📧 Notification préparée pour {self.config['notification_email']}")

        except Exception as e:
            self.logger.error(f"Erreur envoi notification: {e}")

    def executer_maintenance_complete(self):
        """Exécute le cycle complet de maintenance"""
        self.logger.info("🚀 === DÉBUT MAINTENANCE AUTOMATISÉE ===")

        # Vérifier la fenêtre de maintenance
        if not self.verifier_fenetre_maintenance():
            self.logger.warning("⏰ Maintenance annulée - hors fenêtre autorisée")
            return False

        resultats_maintenance = []

        try:
            # Traiter chaque base de données
            for base_config in self.config['bases_a_maintenir']:
                chemin = base_config['chemin']
                priorite = base_config.get('priorite', 'moyenne')

                if os.path.exists(chemin):
                    resultat = self.maintenance_base(chemin, priorite)
                    resultats_maintenance.append(resultat)
                else:
                    self.logger.warning(f"⚠️ Base introuvable: {chemin}")
                    resultats_maintenance.append({
                        'base': chemin,
                        'statut': 'error',
                        'erreurs': ['file_not_found'],
                        'operations': []
                    })

            # Nettoyage des anciennes sauvegardes
            self.nettoyer_anciennes_sauvegardes()

            # Génération du rapport
            rapport_fichier = self.generer_rapport(resultats_maintenance)

            # Notification
            if rapport_fichier:
                self.envoyer_notification(rapport_fichier)

            self.logger.info("✅ === MAINTENANCE TERMINÉE ===")
            return True

        except Exception as e:
            self.logger.error(f"❌ Erreur critique maintenance: {e}")
            return False

def main():
    """Fonction principale du script"""
    parser = argparse.ArgumentParser(description='Maintenance automatisée SQLite')
    parser.add_argument('--config', default='maintenance_config.json',
                       help='Fichier de configuration')
    parser.add_argument('--force', action='store_true',
                       help='Forcer la maintenance hors fenêtre')
    parser.add_argument('--dry-run', action='store_true',
                       help='Simulation sans modifications')

    args = parser.parse_args()

    # Initialiser la maintenance
    maintenance = MaintenanceAutomatisee(args.config)

    if args.force:
        maintenance.config['maintenance_window'] = {'debut': '00:00', 'fin': '23:59'}

    if args.dry_run:
        print("🔍 MODE SIMULATION - Aucune modification ne sera effectuée")
        # Implémenter la logique de simulation
        return

    # Exécuter la maintenance
    succes = maintenance.executer_maintenance_complete()

    # Code de sortie pour les scripts système
    sys.exit(0 if succes else 1)

if __name__ == '__main__':
    main()
```

## Bonnes pratiques et recommandations

### Stratégie de sauvegarde 3-2-1

La règle **3-2-1** est un standard de l'industrie pour les sauvegardes :

- **3** copies de vos données (1 originale + 2 sauvegardes)
- **2** supports de stockage différents (local + réseau, ou SSD + HDD)
- **1** copie hors site (cloud, site distant)

```python
class Strategie321:
    """Implémentation de la stratégie 3-2-1 pour SQLite"""

    def __init__(self, base_source):
        self.base_source = base_source
        self.emplacements = {
            'local': '/backup/local',
            'nas': '/mnt/nas/backup',
            'cloud': '/mnt/cloud/backup'
        }

    def implementer_strategie_321(self):
        """Implémente la stratégie complète 3-2-1"""
        print("🔄 Implémentation stratégie 3-2-1...")

        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        gestionnaire = GestionnaireSauvegarde(self.base_source)

        resultats = {}

        # 1. Sauvegarde locale (support 1)
        fichier_local = f"{self.emplacements['local']}/backup_local_{timestamp}.db"
        os.makedirs(self.emplacements['local'], exist_ok=True)

        print("💾 Sauvegarde locale...")
        resultats['local'] = gestionnaire.sauvegarde_avec_verification(fichier_local)

        # 2. Sauvegarde NAS (support 2)
        if os.path.exists(self.emplacements['nas']):
            fichier_nas = f"{self.emplacements['nas']}/backup_nas_{timestamp}.db"
            print("🗄️ Sauvegarde NAS...")
            resultats['nas'] = gestionnaire.sauvegarde_avec_verification(fichier_nas)
        else:
            print("⚠️ NAS non disponible")
            resultats['nas'] = False

        # 3. Sauvegarde cloud (hors site)
        if os.path.exists(self.emplacements['cloud']):
            fichier_cloud = f"{self.emplacements['cloud']}/backup_cloud_{timestamp}.db"
            print("☁️ Sauvegarde cloud...")
            resultats['cloud'] = gestionnaire.sauvegarde_avec_verification(fichier_cloud)
        else:
            print("⚠️ Cloud non disponible")
            resultats['cloud'] = False

        # Évaluation de la conformité 3-2-1
        copies_reussies = sum(resultats.values())
        supports_differents = sum(1 for v in resultats.values() if v)
        hors_site = resultats.get('cloud', False)

        print(f"\n📊 ÉVALUATION STRATÉGIE 3-2-1:")
        print(f"✅ Copies réussies: {copies_reussies}/3")
        print(f"✅ Supports différents: {supports_differents}/2")
        print(f"✅ Hors site: {'Oui' if hors_site else 'Non'}")

        conforme_321 = copies_reussies >= 3 and supports_differents >= 2 and hors_site

        if conforme_321:
            print("🎉 Stratégie 3-2-1 CONFORME")
        else:
            print("⚠️ Stratégie 3-2-1 NON CONFORME")
            print("📋 Actions recommandées:")
            if copies_reussies < 3:
                print("   • Augmenter le nombre de copies")
            if supports_differents < 2:
                print("   • Utiliser des supports de stockage différents")
            if not hors_site:
                print("   • Mettre en place une sauvegarde hors site")

        return conforme_321, resultats

# Exemple d'utilisation
strategie = Strategie321('ma_base.db')  
conforme, resultats = strategie.implementer_strategie_321()  
```

### Checklist finale de déploiement

```markdown
# ✅ CHECKLIST SAUVEGARDES ET RÉCUPÉRATION SQLITE

## 📋 AVANT DÉPLOIEMENT

### Configuration de base
□ Méthode de sauvegarde sélectionnée (API SQLite recommandée)
□ Répertoires de sauvegarde créés avec permissions appropriées
□ Script de sauvegarde testé sur données de développement
□ Vérification d'intégrité automatique configurée
□ Compression des sauvegardes activée si nécessaire

### Planification automatique
□ Crontab ou planificateur configuré
□ Fenêtres de maintenance définies
□ Fréquence de sauvegarde déterminée selon RTO/RPO
□ Rotation et rétention des sauvegardes configurées
□ Monitoring des tâches de sauvegarde

### Tests et validation
□ Procédure de restauration testée et documentée
□ Tests de récupération point-in-time si nécessaire
□ Validation de l'intégrité des sauvegardes
□ Tests de performance sur sauvegardes
□ Procédures d'urgence documentées et testées

## 🔧 APRÈS DÉPLOIEMENT

### Surveillance
□ Logs de sauvegarde surveillés quotidiennement
□ Alertes configurées pour échecs de sauvegarde
□ Métriques de performance suivies
□ Taille des sauvegardes monitored
□ Tests de restauration périodiques planifiés

### Documentation
□ Procédures de sauvegarde documentées
□ Contacts d'urgence mis à jour
□ Procédures de récupération détaillées
□ Formation équipe effectuée
□ Runbooks d'incident créés

### Conformité
□ Stratégie 3-2-1 implémentée si applicable
□ Exigences réglementaires respectées
□ Chiffrement des sauvegardes si requis
□ Audit des accès aux sauvegardes
□ Plan de continuité d'activité testé
```

## Conclusion et points clés

### Récapitulatif des concepts essentiels

Les sauvegardes et la récupération SQLite nécessitent une approche méthodique adaptée aux spécificités de cette base de données. Voici les points essentiels à retenir :

#### **🔑 Principes fondamentaux**

1. **Utilisez l'API de sauvegarde SQLite** - C'est la seule méthode sûre pour sauvegarder une base active
2. **Automatisez tout** - Sauvegardes manuelles = sauvegardes oubliées
3. **Testez vos restaurations** - Une sauvegarde non testée est une sauvegarde inutile
4. **Planifiez la récupération** - Définissez vos RTO (Recovery Time Objective) et RPO (Recovery Point Objective)
5. **Documentez vos procédures** - En situation d'urgence, la documentation est vitale

#### **⚡ Méthodes recommandées par contexte**

**Applications critiques (production) :**
- Sauvegarde API SQLite + vérification d'intégrité
- Fréquence : toutes les heures ou quotidienne selon les besoins
- Stratégie 3-2-1 implémentée
- Tests de restauration hebdomadaires

**Applications de développement :**
- Sauvegarde API SQLite simple
- Fréquence : quotidienne
- Rétention : 7-30 jours
- Tests de restauration mensuels

**Applications embarquées/IoT :**
- Sauvegarde compressée pour économiser l'espace
- Transmission vers serveur central
- Rétention locale limitée
- Récupération automatique en cas de corruption

#### **🚨 Erreurs courantes à éviter**

```python
# ❌ À NE JAMAIS FAIRE
shutil.copy('base_active.db', 'backup.db')  # Risque de corruption

# ❌ Sauvegarde sans vérification
# Les données peuvent être corrompues sans que vous le sachiez

# ❌ Sauvegardes non testées
# Découvrir que les sauvegardes sont inutilisables pendant un incident

# ❌ Pas de stratégie de rétention
# Disque plein à cause des sauvegardes accumulées

# ✅ BONNE PRATIQUE
conn_source = sqlite3.connect('base_active.db')  
conn_dest = sqlite3.connect('backup.db')  
conn_source.backup(conn_dest)  # API sécurisée  
conn_dest.close()  
conn_source.close()  

# Toujours vérifier l'intégrité
conn_test = sqlite3.connect('backup.db')  
cursor_test = conn_test.cursor()  
cursor_test.execute("PRAGMA integrity_check")  
# ...
```

#### **📊 Métriques importantes à surveiller**

```python
metriques_cles = {
    'taux_succes_sauvegarde': '>99%',        # Sauvegardes réussies
    'duree_sauvegarde': '<5min',             # Temps d'exécution
    'taille_sauvegarde': 'croissance_normale', # Évolution de la taille
    'espace_disque_restant': '>20%',         # Espace disponible
    'age_derniere_sauvegarde': '<24h',       # Fraîcheur des sauvegardes
    'taux_succes_restauration': '100%',      # Tests de restauration
    'rto_actuel': '<4h',                     # Temps de récupération
    'rpo_actuel': '<1h'                      # Perte de données max
}
```

### **🔄 Cycle de vie des sauvegardes**

1. **Planification** : Définir RTO/RPO, fréquence, rétention
2. **Implémentation** : Développer scripts et automatisation
3. **Tests** : Valider sauvegardes et procédures de restauration
4. **Déploiement** : Mise en production avec monitoring
5. **Surveillance** : Monitoring continu et alertes
6. **Optimisation** : Amélioration continue des processus
7. **Documentation** : Mise à jour des procédures

### **📚 Pour aller plus loin**

- **Solutions SQLite spécifiques** :
  - **[Litestream](https://litestream.io/)** : réplication continue vers S3/Azure/GCS (recommandé)
  - **[LiteFS](https://fly.io/docs/litefs/)** : FS distribué pour clustering HA
  - **[Turso](https://turso.tech/) / LibSQL** : SQLite managé avec réplication multi-régions
  - **[restic](https://restic.net/)** : sauvegarde générique chiffrée et déduplicée
- **Standards industriels** : ITIL, ISO 27001 pour la gestion des sauvegardes
- **Outils complémentaires d'entreprise** : Bacula, Amanda, Duplicity, BorgBackup
- **Monitoring** : Nagios, Zabbix, Prometheus + Grafana

Les sauvegardes ne sont pas seulement une assurance contre la perte de données - elles sont un élément fondamental de la fiabilité de votre application. Un système de sauvegarde bien conçu et régulièrement testé vous donnera la tranquillité d'esprit nécessaire pour faire évoluer votre application en toute confiance.

**💡 Conseil final :** Commencez simple avec l'API de sauvegarde SQLite et une planification quotidienne, puis enrichissez progressivement votre stratégie selon vos besoins de disponibilité et de conformité.

---

*Cette section complète le chapitre 8.4 sur les sauvegardes automatisées et stratégies de récupération. La section suivante (8.5) abordera le monitoring et la maintenance préventive des bases de données SQLite.*

⏭️
