🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.5 Monitoring et maintenance préventive

## Introduction au monitoring SQLite

Le monitoring et la maintenance préventive sont essentiels pour maintenir votre base de données SQLite en bonne santé et éviter les problèmes avant qu'ils ne deviennent critiques. Contrairement aux bases de données serveur qui ont des outils de monitoring intégrés, SQLite nécessite une approche personnalisée.

### Pourquoi monitorer SQLite ?

**Prévenir les problèmes avant qu'ils surviennent :**
- 📈 **Croissance de la base** : Anticiper les besoins d'espace disque
- ⚡ **Performance dégradée** : Détecter les ralentissements progressifs
- 🔒 **Verrous prolongés** : Identifier les transactions bloquantes
- 💾 **Fragmentation** : Surveiller l'efficacité du stockage
- 🐛 **Corruption naissante** : Détecter les problèmes d'intégrité

**Exemples concrets de problèmes évitables :**
```
🏪 E-commerce : Base qui ralentit pendant les pics de vente
🏥 Hôpital : Transactions longues qui bloquent l'accès aux dossiers
📱 App mobile : Base qui grossit trop et ralentit l'application
💰 Banque : Corruption de données non détectée à temps
```

### Spécificités du monitoring SQLite

**Avantages :**
- ✅ Pas de processus serveur à surveiller
- ✅ Métriques simples à collecter
- ✅ Faible overhead sur les performances
- ✅ Intégration facile dans l'application

**Défis particuliers :**
- ⚠️ Pas d'outils de monitoring natifs
- ⚠️ Métriques limitées comparé aux SGBD serveur
- ⚠️ Surveillance dépendante de l'application
- ⚠️ Pas de monitoring réseau (pas de connexions)

## Métriques essentielles à surveiller

### 1. Métriques de performance

```python
import sqlite3
import time
import os
from datetime import datetime
from collections import defaultdict

class MonitoringPerformance:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base
        self.historique_metriques = []
        self.seuils_alerte = {
            'temps_reponse_ms': 1000,    # 1 seconde
            'taille_mb': 1000,           # 1 GB
            'fragmentation_pct': 25,     # 25%
            'nb_tables': 100,            # 100 tables
            'duree_vacuum_ms': 30000     # 30 secondes
        }

    def mesurer_temps_reponse(self, requete_test="SELECT COUNT(*) FROM sqlite_master"):
        """Mesure le temps de réponse d'une requête simple"""
        try:
            debut = time.time()

            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()
            cursor.execute(requete_test)
            resultat = cursor.fetchone()
            conn.close()

            duree_ms = (time.time() - debut) * 1000

            return {
                'temps_reponse_ms': duree_ms,
                'requete': requete_test,
                'timestamp': datetime.now(),
                'statut': 'OK' if duree_ms < self.seuils_alerte['temps_reponse_ms'] else 'LENT'
            }

        except Exception as e:
            return {
                'temps_reponse_ms': -1,
                'requete': requete_test,
                'timestamp': datetime.now(),
                'statut': 'ERREUR',
                'erreur': str(e)
            }

    def analyser_taille_base(self):
        """Analyse la taille et la fragmentation de la base"""
        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Taille du fichier principal
            taille_fichier = os.path.getsize(self.chemin_base)
            taille_mb = taille_fichier / (1024 * 1024)

            # Informations sur les pages
            cursor.execute("PRAGMA page_count")
            nb_pages = cursor.fetchone()[0]

            cursor.execute("PRAGMA page_size")
            taille_page = cursor.fetchone()[0]

            cursor.execute("PRAGMA freelist_count")
            pages_libres = cursor.fetchone()[0]

            # Calcul de la fragmentation
            if nb_pages > 0:
                fragmentation_pct = (pages_libres / nb_pages) * 100
            else:
                fragmentation_pct = 0

            # Nombre de tables
            cursor.execute("SELECT COUNT(*) FROM sqlite_master WHERE type='table'")
            nb_tables = cursor.fetchone()[0]

            conn.close()

            return {
                'taille_fichier_bytes': taille_fichier,
                'taille_mb': taille_mb,
                'nb_pages': nb_pages,
                'taille_page': taille_page,
                'pages_libres': pages_libres,
                'fragmentation_pct': fragmentation_pct,
                'nb_tables': nb_tables,
                'timestamp': datetime.now(),
                'alertes': self._evaluer_alertes_taille(taille_mb, fragmentation_pct, nb_tables)
            }

        except Exception as e:
            return {
                'erreur': str(e),
                'timestamp': datetime.now()
            }

    def _evaluer_alertes_taille(self, taille_mb, fragmentation_pct, nb_tables):
        """Évalue les alertes basées sur la taille"""
        alertes = []

        if taille_mb > self.seuils_alerte['taille_mb']:
            alertes.append(f"Taille importante: {taille_mb:.1f} MB")

        if fragmentation_pct > self.seuils_alerte['fragmentation_pct']:
            alertes.append(f"Fragmentation élevée: {fragmentation_pct:.1f}%")

        if nb_tables > self.seuils_alerte['nb_tables']:
            alertes.append(f"Nombreuses tables: {nb_tables}")

        return alertes

    def mesurer_performance_requetes(self, requetes_test=None):
        """Mesure la performance d'un ensemble de requêtes"""
        if requetes_test is None:
            requetes_test = [
                "SELECT COUNT(*) FROM sqlite_master",
                "PRAGMA integrity_check",
                "PRAGMA quick_check"
            ]

        resultats = []

        for requete in requetes_test:
            debut = time.time()

            try:
                conn = sqlite3.connect(self.chemin_base)
                cursor = conn.cursor()
                cursor.execute(requete)
                cursor.fetchall()  # Récupérer tous les résultats
                conn.close()

                duree_ms = (time.time() - debut) * 1000

                resultats.append({
                    'requete': requete,
                    'duree_ms': duree_ms,
                    'statut': 'OK'
                })

            except Exception as e:
                resultats.append({
                    'requete': requete,
                    'duree_ms': -1,
                    'statut': 'ERREUR',
                    'erreur': str(e)
                })

        return {
            'timestamp': datetime.now(),
            'resultats': resultats,
            'duree_totale_ms': sum(r['duree_ms'] for r in resultats if r['duree_ms'] > 0)
        }

    def snapshot_complet(self):
        """Prend un snapshot complet des métriques"""
        print(f"📊 Snapshot des métriques - {datetime.now()}")
        print("=" * 50)

        # 1. Temps de réponse
        temps_reponse = self.mesurer_temps_reponse()
        print(f"⚡ Temps de réponse: {temps_reponse['temps_reponse_ms']:.1f}ms ({temps_reponse['statut']})")

        # 2. Analyse de taille
        analyse_taille = self.analyser_taille_base()
        if 'erreur' not in analyse_taille:
            print(f"📏 Taille: {analyse_taille['taille_mb']:.1f} MB")
            print(f"📋 Tables: {analyse_taille['nb_tables']}")
            print(f"🔧 Fragmentation: {analyse_taille['fragmentation_pct']:.1f}%")

            if analyse_taille['alertes']:
                print("🚨 Alertes:")
                for alerte in analyse_taille['alertes']:
                    print(f"   • {alerte}")
        else:
            print(f"❌ Erreur analyse taille: {analyse_taille['erreur']}")

        # 3. Performance des requêtes
        perf_requetes = self.mesurer_performance_requetes()
        print(f"🔍 Tests requêtes: {perf_requetes['duree_totale_ms']:.1f}ms total")

        for resultat in perf_requetes['resultats']:
            status_emoji = "✅" if resultat['statut'] == 'OK' else "❌"
            if resultat['statut'] == 'OK':
                print(f"   {status_emoji} {resultat['requete'][:30]}... ({resultat['duree_ms']:.1f}ms)")
            else:
                print(f"   {status_emoji} {resultat['requete'][:30]}... ERREUR")

        # Enregistrer dans l'historique
        snapshot = {
            'timestamp': datetime.now(),
            'temps_reponse': temps_reponse,
            'analyse_taille': analyse_taille,
            'performance_requetes': perf_requetes
        }

        self.historique_metriques.append(snapshot)

        return snapshot

# Utilisation du monitoring de performance
monitor = MonitoringPerformance('ma_base.db')

print("=== DÉMONSTRATION MONITORING PERFORMANCE ===")

# Créer une base de test si elle n'existe pas
if not os.path.exists('ma_base.db'):
    conn = sqlite3.connect('ma_base.db')
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE test (id INTEGER PRIMARY KEY, data TEXT)")
    cursor.execute("INSERT INTO test (data) VALUES ('test')")
    conn.commit()
    conn.close()

# Prendre un snapshot
snapshot = monitor.snapshot_complet()
```

### 2. Surveillance de l'intégrité

```python
import sqlite3
import hashlib
import json
from datetime import datetime, timedelta

class MonitoringIntegrite:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base
        self.historique_checks = []
        self.signatures_tables = {}

    def verifier_integrite_complete(self):
        """Effectue une vérification complète d'intégrité"""
        print("🔍 Vérification d'intégrité complète...")

        resultats = {
            'timestamp': datetime.now(),
            'tests': [],
            'statut_global': 'OK',
            'erreurs': []
        }

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # 1. Test d'intégrité SQLite
            print("   🧪 Test intégrité SQLite...")
            cursor.execute("PRAGMA integrity_check")
            check_results = cursor.fetchall()

            if len(check_results) == 1 and check_results[0][0] == 'ok':
                resultats['tests'].append({
                    'test': 'sqlite_integrity_check',
                    'statut': 'OK',
                    'details': 'Intégrité SQLite validée'
                })
                print("      ✅ Intégrité SQLite: OK")
            else:
                resultats['statut_global'] = 'ERREUR'
                erreurs_integrite = [r[0] for r in check_results[:5]]  # Limiter à 5
                resultats['tests'].append({
                    'test': 'sqlite_integrity_check',
                    'statut': 'ERREUR',
                    'details': erreurs_integrite
                })
                resultats['erreurs'].extend(erreurs_integrite)
                print("      ❌ Problèmes d'intégrité SQLite détectés")

            # 2. Vérification des clés étrangères
            print("   🔗 Test clés étrangères...")
            cursor.execute("PRAGMA foreign_key_check")
            fk_violations = cursor.fetchall()

            if not fk_violations:
                resultats['tests'].append({
                    'test': 'foreign_key_check',
                    'statut': 'OK',
                    'details': 'Aucune violation de clé étrangère'
                })
                print("      ✅ Clés étrangères: OK")
            else:
                resultats['statut_global'] = 'AVERTISSEMENT'
                violations = [f"Table {v[0]}, ligne {v[1]}" for v in fk_violations[:5]]
                resultats['tests'].append({
                    'test': 'foreign_key_check',
                    'statut': 'AVERTISSEMENT',
                    'details': violations
                })
                print(f"      ⚠️ {len(fk_violations)} violations de clés étrangères")

            # 3. Vérification des signatures de tables
            print("   📋 Vérification signatures tables...")
            signatures_actuelles = self._calculer_signatures_tables(cursor)

            if self.signatures_tables:
                tables_modifiees = []
                for table, signature in signatures_actuelles.items():
                    if table in self.signatures_tables:
                        if self.signatures_tables[table] != signature:
                            tables_modifiees.append(table)

                if tables_modifiees:
                    resultats['tests'].append({
                        'test': 'table_signatures',
                        'statut': 'INFO',
                        'details': f"Tables modifiées: {', '.join(tables_modifiees)}"
                    })
                    print(f"      ℹ️ {len(tables_modifiees)} tables modifiées")
                else:
                    resultats['tests'].append({
                        'test': 'table_signatures',
                        'statut': 'OK',
                        'details': 'Aucune modification de structure détectée'
                    })
                    print("      ✅ Structures de tables inchangées")
            else:
                resultats['tests'].append({
                    'test': 'table_signatures',
                    'statut': 'INFO',
                    'details': 'Première signature des tables'
                })
                print("      ℹ️ Première signature des tables")

            # Mettre à jour les signatures
            self.signatures_tables = signatures_actuelles

            # 4. Test de lecture sur toutes les tables
            print("   📊 Test lecture toutes tables...")
            cursor.execute("SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%'")
            tables = cursor.fetchall()

            tables_lisibles = 0
            tables_erreur = []

            for (nom_table,) in tables:
                try:
                    cursor.execute(f"SELECT COUNT(*) FROM '{nom_table}'")
                    cursor.fetchone()
                    tables_lisibles += 1
                except Exception as e:
                    tables_erreur.append(f"{nom_table}: {str(e)}")

            if not tables_erreur:
                resultats['tests'].append({
                    'test': 'table_readability',
                    'statut': 'OK',
                    'details': f'{tables_lisibles} tables lisibles'
                })
                print(f"      ✅ {tables_lisibles} tables lisibles")
            else:
                resultats['statut_global'] = 'ERREUR'
                resultats['tests'].append({
                    'test': 'table_readability',
                    'statut': 'ERREUR',
                    'details': tables_erreur
                })
                resultats['erreurs'].extend(tables_erreur)
                print(f"      ❌ {len(tables_erreur)} tables illisibles")

            conn.close()

        except Exception as e:
            resultats['statut_global'] = 'ERREUR'
            resultats['erreurs'].append(f"Erreur connexion: {str(e)}")
            print(f"   ❌ Erreur lors des tests: {e}")

        # Enregistrer dans l'historique
        self.historique_checks.append(resultats)

        print(f"\n📊 Statut global: {resultats['statut_global']}")
        if resultats['erreurs']:
            print("🚨 Erreurs détectées:")
            for erreur in resultats['erreurs'][:3]:  # Limiter l'affichage
                print(f"   • {erreur}")

        return resultats

    def _calculer_signatures_tables(self, cursor):
        """Calcule les signatures (hash) des structures de tables"""
        signatures = {}

        try:
            cursor.execute("SELECT name, sql FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%'")
            tables = cursor.fetchall()

            for nom_table, sql_creation in tables:
                if sql_creation:
                    # Créer une signature basée sur la structure
                    signature = hashlib.md5(sql_creation.encode()).hexdigest()
                    signatures[nom_table] = signature

        except Exception as e:
            print(f"⚠️ Erreur calcul signatures: {e}")

        return signatures

    def detecter_corruption_progressive(self):
        """Détecte une corruption progressive en comparant les checksums"""
        print("🔍 Détection corruption progressive...")

        if len(self.historique_checks) < 2:
            print("   ℹ️ Historique insuffisant pour comparaison")
            return None

        dernier_check = self.historique_checks[-1]
        precedent_check = self.historique_checks[-2]

        # Comparer les résultats d'intégrité
        dernier_ok = dernier_check['statut_global'] == 'OK'
        precedent_ok = precedent_check['statut_global'] == 'OK'

        if precedent_ok and not dernier_ok:
            print("   🚨 CORRUPTION DÉTECTÉE - La base était OK, maintenant en erreur")
            return 'CORRUPTION_DETECTEE'
        elif not precedent_ok and not dernier_ok:
            print("   ⚠️ Problèmes persistants détectés")
            return 'PROBLEMES_PERSISTANTS'
        elif not precedent_ok and dernier_ok:
            print("   ✅ Problèmes résolus")
            return 'PROBLEMES_RESOLUS'
        else:
            print("   ✅ État stable - Aucun problème")
            return 'STABLE'

    def generer_rapport_integrite(self):
        """Génère un rapport d'intégrité détaillé"""
        if not self.historique_checks:
            print("Aucun check d'intégrité dans l'historique")
            return None

        print("\n📋 === RAPPORT D'INTÉGRITÉ ===")

        # Statistiques générales
        total_checks = len(self.historique_checks)
        checks_ok = sum(1 for c in self.historique_checks if c['statut_global'] == 'OK')
        taux_succes = (checks_ok / total_checks * 100) if total_checks > 0 else 0

        print(f"📊 Statistiques générales:")
        print(f"   • Total vérifications: {total_checks}")
        print(f"   • Taux de succès: {taux_succes:.1f}% ({checks_ok}/{total_checks})")

        # Dernier check
        dernier = self.historique_checks[-1]
        print(f"\n🔍 Dernière vérification ({dernier['timestamp']}):")
        print(f"   • Statut: {dernier['statut_global']}")
        print(f"   • Tests effectués: {len(dernier['tests'])}")

        for test in dernier['tests']:
            status_emoji = {"OK": "✅", "ERREUR": "❌", "AVERTISSEMENT": "⚠️", "INFO": "ℹ️"}
            emoji = status_emoji.get(test['statut'], "❓")
            print(f"   {emoji} {test['test']}: {test['statut']}")

        # Tendances
        if total_checks >= 3:
            derniers_3 = self.historique_checks[-3:]
            statuts = [c['statut_global'] for c in derniers_3]

            print(f"\n📈 Tendance (3 derniers checks): {' → '.join(statuts)}")

            if all(s == 'OK' for s in statuts):
                print("   ✅ Tendance stable et positive")
            elif statuts[-1] != 'OK':
                print("   ⚠️ Dégradation récente détectée")

        return {
            'total_checks': total_checks,
            'taux_succes': taux_succes,
            'dernier_check': dernier,
            'timestamp': datetime.now()
        }

# Utilisation du monitoring d'intégrité
monitor_integrite = MonitoringIntegrite('ma_base.db')

print("\n=== DÉMONSTRATION MONITORING INTÉGRITÉ ===")

# Effectuer une vérification complète
resultats_integrite = monitor_integrite.verifier_integrite_complete()

# Générer un rapport
rapport = monitor_integrite.generer_rapport_integrite()
```

### 3. Surveillance des performances système

```python
import psutil
import sqlite3
import threading
import time
from collections import deque
from datetime import datetime

class MonitoringSysteme:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base
        self.metriques_systeme = deque(maxlen=100)  # Garder 100 mesures max
        self.monitoring_actif = False
        self.thread_monitoring = None

    def mesurer_utilisation_ressources(self):
        """Mesure l'utilisation des ressources système"""
        try:
            # CPU
            cpu_percent = psutil.cpu_percent(interval=1)

            # Mémoire
            memoire = psutil.virtual_memory()

            # Disque (répertoire de la base)
            repertoire_base = os.path.dirname(os.path.abspath(self.chemin_base))
            disque = psutil.disk_usage(repertoire_base)

            # Processus Python actuel
            process = psutil.Process()
            memoire_process = process.memory_info()

            # Taille du fichier de base
            taille_base = 0
            if os.path.exists(self.chemin_base):
                taille_base = os.path.getsize(self.chemin_base)

            metriques = {
                'timestamp': datetime.now(),
                'cpu_percent': cpu_percent,
                'memoire_total_gb': memoire.total / (1024**3),
                'memoire_disponible_gb': memoire.available / (1024**3),
                'memoire_utilisee_percent': memoire.percent,
                'disque_total_gb': disque.total / (1024**3),
                'disque_libre_gb': disque.free / (1024**3),
                'disque_utilise_percent': (disque.used / disque.total) * 100,
                'process_memoire_mb': memoire_process.rss / (1024**2),
                'taille_base_mb': taille_base / (1024**2),
                'nombre_threads': threading.active_count()
            }

            return metriques

        except Exception as e:
            return {
                'timestamp': datetime.now(),
                'erreur': str(e)
            }

    def verifier_verrous_base(self):
        """Vérifie s'il y a des verrous sur la base de données"""
        try:
            # Essayer une connexion rapide
            debut = time.time()

            conn = sqlite3.connect(self.chemin_base, timeout=5.0)
            cursor = conn.cursor()

            # Test de lecture rapide
            cursor.execute("SELECT COUNT(*) FROM sqlite_master")
            cursor.fetchone()

            # Test d'écriture rapide (si possible)
            try:
                cursor.execute("PRAGMA quick_check")
                verification_ok = True
            except:
                verification_ok = False

            conn.close()

            duree_ms = (time.time() - debut) * 1000

            return {
                'timestamp': datetime.now(),
                'verrous_detectes': False,
                'duree_connexion_ms': duree_ms,
                'verification_ok': verification_ok,
                'statut': 'OK' if duree_ms < 1000 else 'LENT'
            }

        except sqlite3.OperationalError as e:
            if "database is locked" in str(e).lower():
                return {
                    'timestamp': datetime.now(),
                    'verrous_detectes': True,
                    'erreur': str(e),
                    'statut': 'VERROUILLE'
                }
            else:
                return {
                    'timestamp': datetime.now(),
                    'verrous_detectes': False,
                    'erreur': str(e),
                    'statut': 'ERREUR'
                }
        except Exception as e:
            return {
                'timestamp': datetime.now(),
                'erreur': str(e),
                'statut': 'ERREUR'
            }

    def analyser_croissance_base(self, periode_jours=7):
        """Analyse la croissance de la base sur une période"""
        if len(self.metriques_systeme) < 2:
            return None

        # Filtrer les métriques sur la période
        maintenant = datetime.now()
        limite = maintenant - timedelta(days=periode_jours)

        metriques_periode = [
            m for m in self.metriques_systeme
            if m.get('timestamp', maintenant) >= limite and 'taille_base_mb' in m
        ]

        if len(metriques_periode) < 2:
            return None

        # Calculer la croissance
        premiere_mesure = metriques_periode[0]
        derniere_mesure = metriques_periode[-1]

        taille_initiale = premiere_mesure['taille_base_mb']
        taille_finale = derniere_mesure['taille_base_mb']

        croissance_mb = taille_finale - taille_initiale
        croissance_percent = (croissance_mb / taille_initiale * 100) if taille_initiale > 0 else 0

        duree_jours = (derniere_mesure['timestamp'] - premiere_mesure['timestamp']).days
        croissance_par_jour = croissance_mb / duree_jours if duree_jours > 0 else 0

        return {
            'periode_jours': duree_jours,
            'taille_initiale_mb': taille_initiale,
            'taille_finale_mb': taille_finale,
            'croissance_mb': croissance_mb,
            'croissance_percent': croissance_percent,
            'croissance_par_jour_mb': croissance_par_jour,
            'timestamp': datetime.now()
        }

    def demarrer_monitoring_continu(self, intervalle_secondes=60):
        """Démarre le monitoring continu en arrière-plan"""
        if self.monitoring_actif:
            print("⚠️ Monitoring déjà actif")
            return

        self.monitoring_actif = True

        def boucle_monitoring():
            print(f"🚀 Monitoring continu démarré (intervalle: {intervalle_secondes}s)")

            while self.monitoring_actif:
                try:
                    # Mesurer les ressources
                    metriques = self.mesurer_utilisation_ressources()

                    if 'erreur' not in metriques:
                        self.metriques_systeme.append(metriques)

                        # Vérifier les seuils critiques
                        alertes = self._evaluer_seuils_critiques(metriques)
                        if alertes:
                            print(f"🚨 {datetime.now()}: {', '.join(alertes)}")

                    time.sleep(intervalle_secondes)

                except Exception as e:
                    print(f"⚠️ Erreur monitoring: {e}")
                    time.sleep(intervalle_secondes)

            print("🛑 Monitoring continu arrêté")

        self.thread_monitoring = threading.Thread(target=boucle_monitoring, daemon=True)
        self.thread_monitoring.start()

    def arreter_monitoring(self):
        """Arrête le monitoring continu"""
        self.monitoring_actif = False
        if self.thread_monitoring and self.thread_monitoring.is_alive():
            self.thread_monitoring.join(timeout=5)

    def _evaluer_seuils_critiques(self, metriques):
        """Évalue si des seuils critiques sont dépassés"""
        alertes = []

        # Seuils configurables
        seuils = {
            'cpu_percent': 80,
            'memoire_utilisee_percent': 85,
            'disque_utilise_percent': 90,
            'process_memoire_mb': 500,
            'taille_base_mb': 1000
        }

        # Vérifier chaque seuil
        if metriques.get('cpu_percent', 0) > seuils['cpu_percent']:
            alertes.append(f"CPU élevé: {metriques['cpu_percent']:.1f}%")

        if metriques.get('memoire_utilisee_percent', 0) > seuils['memoire_utilisee_percent']:
            alertes.append(f"Mémoire élevée: {metriques['memoire_utilisee_percent']:.1f}%")

        if metriques.get('disque_utilise_percent', 0) > seuils['disque_utilise_percent']:
            alertes.append(f"Disque plein: {metriques['disque_utilise_percent']:.1f}%")

        if metriques.get('process_memoire_mb', 0) > seuils['process_memoire_mb']:
            alertes.append(f"Processus gourmand: {metriques['process_memoire_mb']:.1f}MB")

        if metriques.get('taille_base_mb', 0) > seuils['taille_base_mb']:
            alertes.append(f"Base volumineuse: {metriques['taille_base_mb']:.1f}MB")

        return alertes

    def generer_rapport_systeme(self):
        """Génère un rapport des performances système"""
        if not self.metriques_systeme:
            print("Aucune métrique système collectée")
            return None

        print("\n🖥️ === RAPPORT PERFORMANCES SYSTÈME ===")

        # Dernières métriques
        dernieres = self.metriques_systeme[-1]

        print(f"📊 État actuel ({dernieres['timestamp']}):")
        print(f"   🖥️ CPU: {dernieres.get('cpu_percent', 0):.1f}%")
        print(f"   💾 Mémoire: {dernieres.get('memoire_utilisee_percent', 0):.1f}% utilisée")
        print(f"   💽 Disque: {dernieres.get('disque_utilise_percent', 0):.1f}% utilisé")
        print(f"   📄 Base SQLite: {dernieres.get('taille_base_mb', 0):.1f} MB")
        print(f"   ⚙️ Processus: {dernieres.get('process_memoire_mb', 0):.1f} MB")

        # Moyennes sur les dernières mesures
        if len(self.metriques_systeme) >= 5:
            dernieres_5 = list(self.metriques_systeme)[-5:]

            moyenne_cpu = sum(m.get('cpu_percent', 0) for m in dernieres_5) / len(dernieres_5)
            moyenne_mem = sum(m.get('memoire_utilisee_percent', 0) for m in dernieres_5) / len(dernieres_5)

            print(f"\n📈 Moyennes (5 dernières mesures):")
            print(f"   🖥️ CPU moyen: {moyenne_cpu:.1f}%")
            print(f"   💾 Mémoire moyenne: {moyenne_mem:.1f}%")

        # Analyse de croissance
        croissance = self.analyser_croissance_base(7)
        if croissance:
            print(f"\n📈 Croissance base (7 jours):")
            print(f"   📏 Taille: {croissance['taille_initiale_mb']:.1f} → {croissance['taille_finale_mb']:.1f} MB")
            print(f"   📊 Croissance: +{croissance['croissance_mb']:.1f} MB ({croissance['croissance_percent']:.1f}%)")
            print(f"   🗓️ Par jour: +{croissance['croissance_par_jour_mb']:.1f} MB/jour")

        return {
            'dernieres_metriques': dernieres,
            'croissance': croissance,
            'timestamp': datetime.now()
        }

# Utilisation du monitoring système
monitor_systeme = MonitoringSysteme('ma_base.db')

print("\n=== DÉMONSTRATION MONITORING SYSTÈME ===")

# Mesurer une fois
metriques = monitor_systeme.mesurer_utilisation_ressources()
if 'erreur' not in metriques:
    print("📊 Métriques système actuelles:")
    print(f"   🖥️ CPU: {metriques['cpu_percent']:.1f}%")
    print(f"   💾 Mémoire: {metriques['memoire_utilisee_percent']:.1f}%")
    print(f"   💽 Disque: {metriques['disque_utilise_percent']:.1f}%")

# Vérifier les verrous
verrous = monitor_systeme.verifier_verrous_base()
print(f"🔒 Statut verrous: {verrous['statut']} ({verrous.get('duree_connexion_ms', 0):.1f}ms)")
```

## Maintenance préventive automatisée

### Système de maintenance intelligente

```python
import sqlite3
import os
import time
import schedule
from datetime import datetime, timedelta
from enum import Enum

class NiveauMaintenance(Enum):
    LEGER = "leger"
    STANDARD = "standard"
    COMPLET = "complet"
    URGENCE = "urgence"

class MaintenancePreventive:
    def __init__(self, chemin_base):
        self.chemin_base = chemin_base
        self.seuils_maintenance = {
            'fragmentation_pct': 15,     # VACUUM si > 15%
            'taille_mb': 100,            # Optimisation si > 100MB
            'jours_sans_maintenance': 7, # Maintenance forcée après 7 jours
            'nb_tables': 50,             # Analyse si > 50 tables
            'duree_requete_ms': 500      # Optimisation si requêtes > 500ms
        }
        self.derniere_maintenance = {}
        self.historique_maintenance = []

    def evaluer_besoin_maintenance(self):
        """Évalue le niveau de maintenance nécessaire"""
        print("🔍 Évaluation du besoin de maintenance...")

        evaluation = {
            'timestamp': datetime.now(),
            'niveau_recommande': NiveauMaintenance.LEGER,
            'raisons': [],
            'metriques': {},
            'urgence': False
        }

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # 1. Analyser la fragmentation
            cursor.execute("PRAGMA freelist_count")
            pages_libres = cursor.fetchone()[0]

            cursor.execute("PRAGMA page_count")
            pages_totales = cursor.fetchone()[0]

            fragmentation_pct = (pages_libres / pages_totales * 100) if pages_totales > 0 else 0
            evaluation['metriques']['fragmentation_pct'] = fragmentation_pct

            if fragmentation_pct > self.seuils_maintenance['fragmentation_pct']:
                evaluation['niveau_recommande'] = NiveauMaintenance.STANDARD
                evaluation['raisons'].append(f"Fragmentation élevée: {fragmentation_pct:.1f}%")

            # 2. Analyser la taille
            taille_mb = os.path.getsize(self.chemin_base) / (1024 * 1024)
            evaluation['metriques']['taille_mb'] = taille_mb

            if taille_mb > self.seuils_maintenance['taille_mb']:
                if evaluation['niveau_recommande'].value == "leger":
                    evaluation['niveau_recommande'] = NiveauMaintenance.STANDARD
                evaluation['raisons'].append(f"Taille importante: {taille_mb:.1f}MB")

            # 3. Vérifier l'intégrité
            cursor.execute("PRAGMA quick_check")
            check_result = cursor.fetchone()[0]

            if check_result != 'ok':
                evaluation['niveau_recommande'] = NiveauMaintenance.URGENCE
                evaluation['urgence'] = True
                evaluation['raisons'].append(f"Problème d'intégrité: {check_result}")

            # 4. Analyser les performances
            debut = time.time()
            cursor.execute("SELECT COUNT(*) FROM sqlite_master")
            duree_ms = (time.time() - debut) * 1000
            evaluation['metriques']['duree_requete_ms'] = duree_ms

            if duree_ms > self.seuils_maintenance['duree_requete_ms']:
                if evaluation['niveau_recommande'].value in ["leger", "standard"]:
                    evaluation['niveau_recommande'] = NiveauMaintenance.COMPLET
                evaluation['raisons'].append(f"Requêtes lentes: {duree_ms:.1f}ms")

            # 5. Vérifier le temps depuis dernière maintenance
            derniere = self.derniere_maintenance.get('complet')
            if derniere:
                jours_sans_maintenance = (datetime.now() - derniere).days
                evaluation['metriques']['jours_sans_maintenance'] = jours_sans_maintenance

                if jours_sans_maintenance > self.seuils_maintenance['jours_sans_maintenance']:
                    if evaluation['niveau_recommande'].value == "leger":
                        evaluation['niveau_recommande'] = NiveauMaintenance.STANDARD
                    evaluation['raisons'].append(f"Pas de maintenance depuis {jours_sans_maintenance} jours")

            conn.close()

        except Exception as e:
            evaluation['niveau_recommande'] = NiveauMaintenance.URGENCE
            evaluation['urgence'] = True
            evaluation['raisons'].append(f"Erreur évaluation: {str(e)}")

        print(f"📋 Niveau recommandé: {evaluation['niveau_recommande'].value.upper()}")
        if evaluation['raisons']:
            print("📝 Raisons:")
            for raison in evaluation['raisons']:
                print(f"   • {raison}")

        return evaluation

    def executer_maintenance_legere(self):
        """Exécute une maintenance légère"""
        print("🔧 === MAINTENANCE LÉGÈRE ===")

        resultats = {
            'timestamp': datetime.now(),
            'niveau': NiveauMaintenance.LEGER,
            'operations': [],
            'duree_totale_s': 0,
            'succes': True,
            'erreurs': []
        }

        debut_total = time.time()

        try:
            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # 1. Optimisation rapide
            print("⚡ Optimisation rapide...")
            debut = time.time()
            cursor.execute("PRAGMA optimize")
            duree = time.time() - debut

            resultats['operations'].append({
                'operation': 'optimize',
                'duree_s': duree,
                'statut': 'OK'
            })
            print(f"   ✅ Optimisation terminée ({duree:.1f}s)")

            # 2. Nettoyage du cache
            print("🧹 Nettoyage cache...")
            cursor.execute("PRAGMA shrink_memory")

            resultats['operations'].append({
                'operation': 'shrink_memory',
                'duree_s': 0.1,
                'statut': 'OK'
            })
            print("   ✅ Cache nettoyé")

            conn.close()

        except Exception as e:
            resultats['succes'] = False
            resultats['erreurs'].append(str(e))
            print(f"   ❌ Erreur: {e}")

        resultats['duree_totale_s'] = time.time() - debut_total
        self.derniere_maintenance['leger'] = datetime.now()
        self.historique_maintenance.append(resultats)

        print(f"✅ Maintenance légère terminée ({resultats['duree_totale_s']:.1f}s)")
        return resultats

    def executer_maintenance_standard(self):
        """Exécute une maintenance standard"""
        print("🔧 === MAINTENANCE STANDARD ===")

        resultats = {
            'timestamp': datetime.now(),
            'niveau': NiveauMaintenance.STANDARD,
            'operations': [],
            'duree_totale_s': 0,
            'succes': True,
            'erreurs': []
        }

        debut_total = time.time()

        try:
            # 1. Maintenance légère d'abord
            print("📋 Étape 1: Maintenance légère...")
            maintenance_legere = self.executer_maintenance_legere()
            resultats['operations'].extend(maintenance_legere['operations'])

            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # 2. Réindexation
            print("📊 Étape 2: Réindexation...")
            debut = time.time()
            cursor.execute("REINDEX")
            duree = time.time() - debut

            resultats['operations'].append({
                'operation': 'reindex',
                'duree_s': duree,
                'statut': 'OK'
            })
            print(f"   ✅ Réindexation terminée ({duree:.1f}s)")

            # 3. Analyse des statistiques
            print("📈 Étape 3: Analyse statistiques...")
            debut = time.time()
            cursor.execute("ANALYZE")
            duree = time.time() - debut

            resultats['operations'].append({
                'operation': 'analyze',
                'duree_s': duree,
                'statut': 'OK'
            })
            print(f"   ✅ Analyse terminée ({duree:.1f}s)")

            conn.close()

        except Exception as e:
            resultats['succes'] = False
            resultats['erreurs'].append(str(e))
            print(f"   ❌ Erreur: {e}")

        resultats['duree_totale_s'] = time.time() - debut_total
        self.derniere_maintenance['standard'] = datetime.now()
        self.historique_maintenance.append(resultats)

        print(f"✅ Maintenance standard terminée ({resultats['duree_totale_s']:.1f}s)")
        return resultats

    def executer_maintenance_complete(self):
        """Exécute une maintenance complète"""
        print("🔧 === MAINTENANCE COMPLÈTE ===")

        resultats = {
            'timestamp': datetime.now(),
            'niveau': NiveauMaintenance.COMPLET,
            'operations': [],
            'duree_totale_s': 0,
            'succes': True,
            'erreurs': []
        }

        debut_total = time.time()

        try:
            # 1. Maintenance standard d'abord
            print("📋 Étape 1: Maintenance standard...")
            maintenance_standard = self.executer_maintenance_standard()
            resultats['operations'].extend(maintenance_standard['operations'])

            # 2. VACUUM (la plus longue opération)
            print("🗜️ Étape 2: VACUUM (peut prendre du temps)...")
            debut = time.time()

            conn = sqlite3.connect(self.chemin_base)
            cursor = conn.cursor()

            # Mesurer l'espace avant VACUUM
            taille_avant = os.path.getsize(self.chemin_base)

            cursor.execute("VACUUM")

            # Mesurer l'espace après VACUUM
            taille_apres = os.path.getsize(self.chemin_base)
            espace_libere = taille_avant - taille_apres

            duree = time.time() - debut

            resultats['operations'].append({
                'operation': 'vacuum',
                'duree_s': duree,
                'statut': 'OK',
                'espace_libere_mb': espace_libere / (1024 * 1024)
            })
            print(f"   ✅ VACUUM terminé ({duree:.1f}s, {espace_libere/(1024*1024):.1f}MB libérés)")

            # 3. Vérification finale d'intégrité
            print("🔍 Étape 3: Vérification intégrité finale...")
            debut = time.time()
            cursor.execute("PRAGMA integrity_check")
            check_result = cursor.fetchone()[0]
            duree = time.time() - debut

            resultats['operations'].append({
                'operation': 'integrity_check',
                'duree_s': duree,
                'statut': 'OK' if check_result == 'ok' else 'ERREUR',
                'resultat': check_result
            })

            if check_result == 'ok':
                print(f"   ✅ Intégrité vérifiée ({duree:.1f}s)")
            else:
                print(f"   ❌ Problème d'intégrité: {check_result}")
                resultats['erreurs'].append(f"Intégrité: {check_result}")

            conn.close()

        except Exception as e:
            resultats['succes'] = False
            resultats['erreurs'].append(str(e))
            print(f"   ❌ Erreur: {e}")

        resultats['duree_totale_s'] = time.time() - debut_total
        self.derniere_maintenance['complet'] = datetime.now()
        self.historique_maintenance.append(resultats)

        print(f"✅ Maintenance complète terminée ({resultats['duree_totale_s']:.1f}s)")
        return resultats

    def maintenance_automatique(self):
        """Effectue une maintenance automatique basée sur l'évaluation"""
        print("🤖 === MAINTENANCE AUTOMATIQUE ===")

        # Évaluer le besoin
        evaluation = self.evaluer_besoin_maintenance()

        # Exécuter la maintenance appropriée
        if evaluation['niveau_recommande'] == NiveauMaintenance.URGENCE:
            print("🚨 Maintenance d'urgence requise!")
            return self.executer_maintenance_complete()
        elif evaluation['niveau_recommande'] == NiveauMaintenance.COMPLET:
            return self.executer_maintenance_complete()
        elif evaluation['niveau_recommande'] == NiveauMaintenance.STANDARD:
            return self.executer_maintenance_standard()
        else:
            return self.executer_maintenance_legere()

    def planifier_maintenance_automatique(self):
        """Planifie la maintenance automatique"""
        print("📅 Planification de la maintenance automatique...")

        # Maintenance légère quotidienne à 3h
        schedule.every().day.at("03:00").do(self.executer_maintenance_legere)

        # Maintenance standard hebdomadaire le dimanche à 2h
        schedule.every().sunday.at("02:00").do(self.executer_maintenance_standard)

        # Maintenance complète mensuelle le 1er du mois à 1h
        schedule.every().month.do(self.executer_maintenance_complete)

        print("✅ Planning configuré:")
        print("   • Légère: tous les jours à 03:00")
        print("   • Standard: dimanche à 02:00")
        print("   • Complète: 1er du mois à 01:00")

    def generer_rapport_maintenance(self):
        """Génère un rapport de maintenance"""
        if not self.historique_maintenance:
            print("Aucune maintenance dans l'historique")
            return None

        print("\n🔧 === RAPPORT DE MAINTENANCE ===")

        # Statistiques générales
        total_maintenances = len(self.historique_maintenance)
        maintenances_reussies = sum(1 for m in self.historique_maintenance if m['succes'])
        taux_succes = (maintenances_reussies / total_maintenances * 100) if total_maintenances > 0 else 0

        print(f"📊 Statistiques générales:")
        print(f"   • Total maintenances: {total_maintenances}")
        print(f"   • Taux de succès: {taux_succes:.1f}% ({maintenances_reussies}/{total_maintenances})")

        # Dernière maintenance
        derniere = self.historique_maintenance[-1]
        print(f"\n🔧 Dernière maintenance ({derniere['timestamp']}):")
        print(f"   • Niveau: {derniere['niveau'].value}")
        print(f"   • Durée: {derniere['duree_totale_s']:.1f}s")
        print(f"   • Statut: {'✅ Succès' if derniere['succes'] else '❌ Échec'}")
        print(f"   • Opérations: {len(derniere['operations'])}")

        if derniere['erreurs']:
            print("   🚨 Erreurs:")
            for erreur in derniere['erreurs']:
                print(f"      • {erreur}")

        # Répartition par niveau
        niveaux = {}
        for maintenance in self.historique_maintenance:
            niveau = maintenance['niveau'].value
            niveaux[niveau] = niveaux.get(niveau, 0) + 1

        print(f"\n📋 Répartition par niveau:")
        for niveau, count in niveaux.items():
            print(f"   • {niveau.capitalize()}: {count}")

        # Temps de maintenance moyen par niveau
        print(f"\n⏱️ Durées moyennes:")
        for niveau in set(m['niveau'].value for m in self.historique_maintenance):
            maintenances_niveau = [m for m in self.historique_maintenance if m['niveau'].value == niveau]
            duree_moyenne = sum(m['duree_totale_s'] for m in maintenances_niveau) / len(maintenances_niveau)
            print(f"   • {niveau.capitalize()}: {duree_moyenne:.1f}s")

        return {
            'total_maintenances': total_maintenances,
            'taux_succes': taux_succes,
            'derniere_maintenance': derniere,
            'repartition_niveaux': niveaux,
            'timestamp': datetime.now()
        }

# Utilisation de la maintenance préventive
maintenance = MaintenancePreventive('ma_base.db')

print("\n=== DÉMONSTRATION MAINTENANCE PRÉVENTIVE ===")

# Évaluer le besoin de maintenance
evaluation = maintenance.evaluer_besoin_maintenance()

# Effectuer une maintenance automatique
if evaluation['niveau_recommande'] != NiveauMaintenance.URGENCE:
    resultats = maintenance.maintenance_automatique()

    if resultats['succes']:
        print(f"\n🎉 Maintenance réussie en {resultats['duree_totale_s']:.1f}s")
    else:
        print(f"\n⚠️ Maintenance avec erreurs: {resultats['erreurs']}")

# Générer un rapport
rapport = maintenance.generer_rapport_maintenance()
```

## Dashboard de monitoring centralisé

### Interface de monitoring en temps réel

```python
import json
import time
from datetime import datetime, timedelta
from flask import Flask, render_template, jsonify
import threading

class DashboardMonitoring:
    def __init__(self, bases_a_monitorer):
        self.bases_a_monitorer = bases_a_monitorer  # Liste des chemins des bases
        self.moniteurs = {}
        self.app = Flask(__name__)
        self.donnees_dashboard = {
            'derniere_mise_a_jour': None,
            'bases': {},
            'alertes': [],
            'statistiques_globales': {}
        }

        # Initialiser les moniteurs pour chaque base
        for chemin_base in bases_a_monitorer:
            nom_base = os.path.basename(chemin_base)
            self.moniteurs[nom_base] = {
                'performance': MonitoringPerformance(chemin_base),
                'integrite': MonitoringIntegrite(chemin_base),
                'systeme': MonitoringSysteme(chemin_base),
                'maintenance': MaintenancePreventive(chemin_base)
            }

        self._configurer_routes()

    def _configurer_routes(self):
        """Configure les routes Flask pour le dashboard"""

        @self.app.route('/')
        def dashboard():
            return render_template('dashboard.html')

        @self.app.route('/api/donnees')
        def api_donnees():
            return jsonify(self.donnees_dashboard)

        @self.app.route('/api/base/<nom_base>')
        def api_base_details(nom_base):
            if nom_base in self.donnees_dashboard['bases']:
                return jsonify(self.donnees_dashboard['bases'][nom_base])
            else:
                return jsonify({'erreur': 'Base non trouvée'}), 404

        @self.app.route('/api/maintenance/<nom_base>')
        def api_declencher_maintenance(nom_base):
            if nom_base in self.moniteurs:
                try:
                    maintenance = self.moniteurs[nom_base]['maintenance']
                    resultats = maintenance.maintenance_automatique()
                    return jsonify({
                        'statut': 'succès',
                        'resultats': resultats
                    })
                except Exception as e:
                    return jsonify({
                        'statut': 'erreur',
                        'erreur': str(e)
                    }), 500
            else:
                return jsonify({'erreur': 'Base non trouvée'}), 404

    def collecter_donnees(self):
        """Collecte les données de toutes les bases"""
        self.donnees_dashboard['derniere_mise_a_jour'] = datetime.now().isoformat()
        alertes_globales = []

        for nom_base, moniteurs in self.moniteurs.items():
            print(f"📊 Collecte données pour {nom_base}...")

            donnees_base = {
                'nom': nom_base,
                'derniere_collecte': datetime.now().isoformat(),
                'statut_global': 'OK',
                'alertes': []
            }

            try:
                # 1. Métriques de performance
                perf = moniteurs['performance'].snapshot_complet()
                donnees_base['performance'] = {
                    'temps_reponse_ms': perf['temps_reponse']['temps_reponse_ms'],
                    'taille_mb': perf['analyse_taille'].get('taille_mb', 0),
                    'fragmentation_pct': perf['analyse_taille'].get('fragmentation_pct', 0),
                    'nb_tables': perf['analyse_taille'].get('nb_tables', 0)
                }

                # Alertes de performance
                if perf['analyse_taille'].get('alertes', []):
                    donnees_base['alertes'].extend(perf['analyse_taille']['alertes'])
                    alertes_globales.extend([f"{nom_base}: {a}" for a in perf['analyse_taille']['alertes']])

                # 2. Intégrité
                integrite = moniteurs['integrite'].verifier_integrite_complete()
                donnees_base['integrite'] = {
                    'statut': integrite['statut_global'],
                    'nb_tests': len(integrite['tests']),
                    'erreurs': integrite['erreurs']
                }

                if integrite['statut_global'] != 'OK':
                    donnees_base['statut_global'] = integrite['statut_global']
                    alertes_globales.append(f"{nom_base}: Problème d'intégrité")

                # 3. Système
                systeme = moniteurs['systeme'].mesurer_utilisation_ressources()
                if 'erreur' not in systeme:
                    donnees_base['systeme'] = {
                        'cpu_percent': systeme['cpu_percent'],
                        'memoire_percent': systeme['memoire_utilisee_percent'],
                        'disque_percent': systeme['disque_utilise_percent']
                    }

                    # Alertes système
                    alertes_systeme = moniteurs['systeme']._evaluer_seuils_critiques(systeme)
                    if alertes_systeme:
                        donnees_base['alertes'].extend(alertes_systeme)
                        alertes_globales.extend([f"{nom_base}: {a}" for a in alertes_systeme])

                # 4. Maintenance
                evaluation_maintenance = moniteurs['maintenance'].evaluer_besoin_maintenance()
                donnees_base['maintenance'] = {
                    'niveau_recommande': evaluation_maintenance['niveau_recommande'].value,
                    'raisons': evaluation_maintenance['raisons'],
                    'urgence': evaluation_maintenance['urgence']
                }

                if evaluation_maintenance['urgence']:
                    donnees_base['statut_global'] = 'CRITIQUE'
                    alertes_globales.append(f"{nom_base}: Maintenance urgente requise")

    except Exception as e:
                donnees_base['statut_global'] = 'ERREUR'
                donnees_base['erreur'] = str(e)
                alertes_globales.append(f"{nom_base}: Erreur monitoring ({str(e)})")
                print(f"❌ Erreur collecte {nom_base}: {e}")

            self.donnees_dashboard['bases'][nom_base] = donnees_base

        # Statistiques globales
        self.donnees_dashboard['alertes'] = alertes_globales
        self.donnees_dashboard['statistiques_globales'] = self._calculer_statistiques_globales()

    def _calculer_statistiques_globales(self):
        """Calcule les statistiques globales"""
        total_bases = len(self.donnees_dashboard['bases'])
        bases_ok = sum(1 for b in self.donnees_dashboard['bases'].values()
                      if b['statut_global'] == 'OK')
        bases_erreur = sum(1 for b in self.donnees_dashboard['bases'].values()
                          if b['statut_global'] == 'ERREUR')
        bases_critique = sum(1 for b in self.donnees_dashboard['bases'].values()
                            if b['statut_global'] == 'CRITIQUE')

        taille_totale_mb = sum(b.get('performance', {}).get('taille_mb', 0)
                              for b in self.donnees_dashboard['bases'].values())

        return {
            'total_bases': total_bases,
            'bases_ok': bases_ok,
            'bases_erreur': bases_erreur,
            'bases_critique': bases_critique,
            'taux_sante': (bases_ok / total_bases * 100) if total_bases > 0 else 0,
            'taille_totale_mb': taille_totale_mb,
            'total_alertes': len(self.donnees_dashboard['alertes'])
        }

    def demarrer_collecte_continue(self, intervalle_minutes=5):
        """Démarre la collecte continue de données"""
        def boucle_collecte():
            while True:
                try:
                    self.collecter_donnees()
                    print(f"✅ Données collectées à {datetime.now()}")
                except Exception as e:
                    print(f"❌ Erreur collecte: {e}")

                time.sleep(intervalle_minutes * 60)

        thread_collecte = threading.Thread(target=boucle_collecte, daemon=True)
        thread_collecte.start()
        print(f"🚀 Collecte continue démarrée (intervalle: {intervalle_minutes}min)")

    def demarrer_dashboard(self, host='localhost', port=5000, debug=False):
        """Démarre le serveur web du dashboard"""
        print(f"🌐 Dashboard disponible sur http://{host}:{port}")
        self.app.run(host=host, port=port, debug=debug)

# Template HTML pour le dashboard (à sauvegarder dans templates/dashboard.html)
TEMPLATE_DASHBOARD = """
<!DOCTYPE html>
<html>
<head>
    <title>Dashboard SQLite Monitoring</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: Arial, sans-serif; margin: 0; padding: 20px; background: #f5f5f5; }
        .header { background: #2c3e50; color: white; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
        .stats-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 20px; margin-bottom: 20px; }
        .stat-card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .stat-value { font-size: 2em; font-weight: bold; color: #3498db; }
        .bases-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(400px, 1fr)); gap: 20px; }
        .base-card { background: white; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); overflow: hidden; }
        .base-header { padding: 15px; background: #34495e; color: white; }
        .base-content { padding: 15px; }
        .status-ok { background: #27ae60; }
        .status-erreur { background: #e74c3c; }
        .status-critique { background: #e74c3c; animation: blink 1s infinite; }
        .alertes { background: #fff3cd; border: 1px solid #ffeaa7; padding: 10px; border-radius: 4px; margin: 10px 0; }
        .metric { margin: 5px 0; padding: 5px; background: #ecf0f1; border-radius: 4px; }
        @keyframes blink { 50% { opacity: 0.5; } }
        .refresh-btn { background: #3498db; color: white; border: none; padding: 10px 20px; border-radius: 4px; cursor: pointer; }
        .maintenance-btn { background: #f39c12; color: white; border: none; padding: 5px 10px; border-radius: 4px; cursor: pointer; margin: 5px; }
    </style>
</head>
<body>
    <div class="header">
        <h1>📊 Dashboard SQLite Monitoring</h1>
        <p>Surveillance en temps réel des bases de données SQLite</p>
        <button class="refresh-btn" onclick="rafraichirDonnees()">🔄 Actualiser</button>
        <span id="derniere-maj"></span>
    </div>

    <div class="stats-grid" id="stats-globales">
        <!-- Statistiques globales -->
    </div>

    <div id="alertes-globales">
        <!-- Alertes globales -->
    </div>

    <div class="bases-grid" id="bases-container">
        <!-- Cartes des bases -->
    </div>

    <script>
        let donnees = {};

        function rafraichirDonnees() {
            fetch('/api/donnees')
                .then(response => response.json())
                .then(data => {
                    donnees = data;
                    afficherStatistiquesGlobales();
                    afficherAlertesGlobales();
                    afficherBases();
                    document.getElementById('derniere-maj').textContent =
                        'Dernière mise à jour: ' + new Date(data.derniere_mise_a_jour).toLocaleString();
                })
                .catch(error => console.error('Erreur:', error));
        }

        function afficherStatistiquesGlobales() {
            const stats = donnees.statistiques_globales;
            const container = document.getElementById('stats-globales');

            container.innerHTML = `
                <div class="stat-card">
                    <div class="stat-value">${stats.total_bases}</div>
                    <div>Bases surveillées</div>
                </div>
                <div class="stat-card">
                    <div class="stat-value" style="color: #27ae60">${stats.bases_ok}</div>
                    <div>Bases en santé</div>
                </div>
                <div class="stat-card">
                    <div class="stat-value" style="color: #e74c3c">${stats.bases_erreur + stats.bases_critique}</div>
                    <div>Bases avec problèmes</div>
                </div>
                <div class="stat-card">
                    <div class="stat-value">${stats.taille_totale_mb.toFixed(1)} MB</div>
                    <div>Taille totale</div>
                </div>
                <div class="stat-card">
                    <div class="stat-value">${stats.taux_sante.toFixed(1)}%</div>
                    <div>Taux de santé</div>
                </div>
            `;
        }

        function afficherAlertesGlobales() {
            const container = document.getElementById('alertes-globales');

            if (donnees.alertes && donnees.alertes.length > 0) {
                container.innerHTML = `
                    <div class="alertes">
                        <h3>🚨 Alertes actives (${donnees.alertes.length})</h3>
                        ${donnees.alertes.map(alerte => `<div>• ${alerte}</div>`).join('')}
                    </div>
                `;
            } else {
                container.innerHTML = '';
            }
        }

        function afficherBases() {
            const container = document.getElementById('bases-container');

            container.innerHTML = Object.entries(donnees.bases).map(([nom, base]) => `
                <div class="base-card">
                    <div class="base-header status-${base.statut_global.toLowerCase()}">
                        <h3>${base.nom}</h3>
                        <span>Statut: ${base.statut_global}</span>
                    </div>
                    <div class="base-content">
                        ${base.performance ? `
                            <h4>📊 Performance</h4>
                            <div class="metric">⚡ Temps de réponse: ${base.performance.temps_reponse_ms.toFixed(1)}ms</div>
                            <div class="metric">📏 Taille: ${base.performance.taille_mb.toFixed(1)} MB</div>
                            <div class="metric">🔧 Fragmentation: ${base.performance.fragmentation_pct.toFixed(1)}%</div>
                            <div class="metric">📋 Tables: ${base.performance.nb_tables}</div>
                        ` : ''}

                        ${base.integrite ? `
                            <h4>🔍 Intégrité</h4>
                            <div class="metric">Statut: ${base.integrite.statut}</div>
                            <div class="metric">Tests: ${base.integrite.nb_tests}</div>
                        ` : ''}

                        ${base.systeme ? `
                            <h4>🖥️ Système</h4>
                            <div class="metric">CPU: ${base.systeme.cpu_percent.toFixed(1)}%</div>
                            <div class="metric">Mémoire: ${base.systeme.memoire_percent.toFixed(1)}%</div>
                            <div class="metric">Disque: ${base.systeme.disque_percent.toFixed(1)}%</div>
                        ` : ''}

                        ${base.maintenance ? `
                            <h4>🔧 Maintenance</h4>
                            <div class="metric">Niveau recommandé: ${base.maintenance.niveau_recommande}</div>
                            ${base.maintenance.urgence ? '<div style="color: red;">⚠️ URGENCE</div>' : ''}
                            <button class="maintenance-btn" onclick="declencherMaintenance('${nom}')">
                                🔧 Lancer maintenance
                            </button>
                        ` : ''}

                        ${base.alertes && base.alertes.length > 0 ? `
                            <div class="alertes">
                                <strong>⚠️ Alertes:</strong>
                                ${base.alertes.map(alerte => `<div>• ${alerte}</div>`).join('')}
                            </div>
                        ` : ''}
                    </div>
                </div>
            `).join('');
        }

        function declencherMaintenance(nomBase) {
            if (confirm(`Lancer la maintenance automatique pour ${nomBase} ?`)) {
                fetch(`/api/maintenance/${nomBase}`)
                    .then(response => response.json())
                    .then(data => {
                        if (data.statut === 'succès') {
                            alert('Maintenance terminée avec succès');
                            rafraichirDonnees();
                        } else {
                            alert('Erreur lors de la maintenance: ' + data.erreur);
                        }
                    })
                    .catch(error => {
                        alert('Erreur: ' + error);
                    });
            }
        }

        // Actualisation automatique toutes les 30 secondes
        setInterval(rafraichirDonnees, 30000);

        // Chargement initial
        rafraichirDonnees();
    </script>
</body>
</html>
"""

# Utilisation du dashboard
def demo_dashboard():
    """Démonstration du dashboard de monitoring"""

    # Créer le répertoire templates si nécessaire
    os.makedirs('templates', exist_ok=True)

    # Sauvegarder le template HTML
    with open('templates/dashboard.html', 'w', encoding='utf-8') as f:
        f.write(TEMPLATE_DASHBOARD)

    # Liste des bases à monitorer
    bases_a_monitorer = ['ma_base.db']  # Ajouter d'autres bases ici

    # Créer le dashboard
    dashboard = DashboardMonitoring(bases_a_monitorer)

    # Effectuer une collecte initiale
    dashboard.collecter_donnees()

    # Démarrer la collecte continue
    dashboard.demarrer_collecte_continue(intervalle_minutes=1)  # Test avec 1 min

    print("\n=== DASHBOARD DE MONITORING ===")
    print("🌐 Le dashboard sera disponible sur http://localhost:5000")
    print("📊 Fonctionnalités:")
    print("   • Vue d'ensemble en temps réel")
    print("   • Alertes automatiques")
    print("   • Déclenchement de maintenance")
    print("   • Actualisation automatique")

    # Démarrer le serveur (décommenter pour lancer réellement)
    # dashboard.demarrer_dashboard(debug=True)

    return dashboard

# Pour les tests, créer une version simplifiée
if __name__ == "__main__":
    print("=== DÉMONSTRATION MONITORING COMPLET ===")

    # Tests des différents moniteurs
    if os.path.exists('ma_base.db'):
        print("\n1. 📊 Test monitoring performance...")
        monitor_perf = MonitoringPerformance('ma_base.db')
        snapshot = monitor_perf.snapshot_complet()

        print("\n2. 🔍 Test monitoring intégrité...")
        monitor_integrite = MonitoringIntegrite('ma_base.db')
        integrite = monitor_integrite.verifier_integrite_complete()

        print("\n3. 🔧 Test maintenance préventive...")
        maintenance = MaintenancePreventive('ma_base.db')
        evaluation = maintenance.evaluer_besoin_maintenance()

        print(f"\n📋 RÉSUMÉ:")
        print(f"   Performance: {snapshot['temps_reponse']['statut']}")
        print(f"   Intégrité: {integrite['statut_global']}")
        print(f"   Maintenance: {evaluation['niveau_recommande'].value}")

        # Créer le dashboard (sans le lancer)
        dashboard = demo_dashboard()
        print("\n✅ Dashboard configuré et prêt")
    else:
        print("❌ Base de test non trouvée. Créez 'ma_base.db' pour tester.")
```

## Scripts d'automatisation et alertes

### Système d'alertes avancé

```python
import smtplib
import requests
import json
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime

class SystemeAlertes:
    def __init__(self, config_alertes):
        self.config = config_alertes
        self.alertes_actives = {}
        self.historique_alertes = []

    def evaluer_alertes(self, donnees_monitoring):
        """Évalue les données et déclenche les alertes appropriées"""
        nouvelles_alertes = []

        for nom_base, donnees in donnees_monitoring.items():
            alertes_base = self._evaluer_alertes_base(nom_base, donnees)
            nouvelles_alertes.extend(alertes_base)

        # Traiter les nouvelles alertes
        for alerte in nouvelles_alertes:
            self._traiter_alerte(alerte)

        return nouvelles_alertes

    def _evaluer_alertes_base(self, nom_base, donnees):
        """Évalue les alertes pour une base spécifique"""
        alertes = []

        # Alertes de performance
        if 'performance' in donnees:
            perf = donnees['performance']

            if perf.get('temps_reponse_ms', 0) > self.config['seuils']['temps_reponse_ms']:
                alertes.append({
                    'type': 'PERFORMANCE',
                    'niveau': 'WARNING',
                    'base': nom_base,
                    'message': f"Temps de réponse élevé: {perf['temps_reponse_ms']:.1f}ms",
                    'timestamp': datetime.now(),
                    'valeur': perf['temps_reponse_ms'],
                    'seuil': self.config['seuils']['temps_reponse_ms']
                })

            if perf.get('fragmentation_pct', 0) > self.config['seuils']['fragmentation_pct']:
                alertes.append({
                    'type': 'FRAGMENTATION',
                    'niveau': 'INFO',
                    'base': nom_base,
                    'message': f"Fragmentation élevée: {perf['fragmentation_pct']:.1f}%",
                    'timestamp': datetime.now(),
                    'valeur': perf['fragmentation_pct'],
                    'seuil': self.config['seuils']['fragmentation_pct']
                })

        # Alertes d'intégrité
        if 'integrite' in donnees:
            if donnees['integrite']['statut'] == 'ERREUR':
                alertes.append({
                    'type': 'INTEGRITE',
                    'niveau': 'CRITICAL',
                    'base': nom_base,
                    'message': "Problème d'intégrité détecté",
                    'timestamp': datetime.now(),
                    'details': donnees['integrite'].get('erreurs', [])
                })

        # Alertes système
        if 'systeme' in donnees:
            sys = donnees['systeme']

            if sys.get('disque_percent', 0) > self.config['seuils']['disque_percent']:
                alertes.append({
                    'type': 'DISQUE',
                    'niveau': 'WARNING',
                    'base': nom_base,
                    'message': f"Espace disque faible: {sys['disque_percent']:.1f}%",
                    'timestamp': datetime.now(),
                    'valeur': sys['disque_percent'],
                    'seuil': self.config['seuils']['disque_percent']
                })

        return alertes

    def _traiter_alerte(self, alerte):
        """Traite une alerte (envoi notifications, etc.)"""
        cle_alerte = f"{alerte['base']}_{alerte['type']}"

        # Vérifier si c'est une nouvelle alerte ou si elle persiste
        if cle_alerte not in self.alertes_actives:
            # Nouvelle alerte
            self.alertes_actives[cle_alerte] = alerte
            self._envoyer_notifications(alerte, 'NOUVELLE')
        else:
            # Alerte persistante - mettre à jour
            self.alertes_actives[cle_alerte]['timestamp'] = alerte['timestamp']

        # Ajouter à l'historique
        self.historique_alertes.append(alerte)

    def _envoyer_notifications(self, alerte, type_notification):
        """Envoie les notifications pour une alerte"""

        # Email
        if self.config.get('email', {}).get('actif', False):
            self._envoyer_email(alerte, type_notification)

        # Slack/Teams
        if self.config.get('webhook', {}).get('actif', False):
            self._envoyer_webhook(alerte, type_notification)

        # SMS (simulation)
        if self.config.get('sms', {}).get('actif', False):
            self._envoyer_sms(alerte, type_notification)

    def _envoyer_email(self, alerte, type_notification):
        """Envoie une alerte par email"""
        try:
            config_email = self.config['email']

            msg = MIMEMultipart()
            msg['From'] = config_email['from']
            msg['To'] = config_email['to']
            msg['Subject'] = f"[SQLite Alert] {alerte['niveau']} - {alerte['base']}"

            # Corps du message
            emoji_niveau = {
                'INFO': 'ℹ️',
                'WARNING': '⚠️',
                'CRITICAL': '🚨'
            }

            body = f"""
{emoji_niveau.get(alerte['niveau'], '❓')} ALERTE SQLite {alerte['niveau']}

Base de données: {alerte['base']}
Type: {alerte['type']}
Message: {alerte['message']}
Timestamp: {alerte['timestamp']}

Détails techniques:
{json.dumps(alerte, indent=2, default=str)}

Dashboard: {self.config.get('dashboard_url', 'N/A')}
            """

            msg.attach(MIMEText(body, 'plain'))

            # Envoyer (simulation - adapter selon votre config SMTP)
            print(f"📧 Email simulé envoyé pour {alerte['base']} - {alerte['type']}")

        except Exception as e:
            print(f"❌ Erreur envoi email: {e}")

    def _envoyer_webhook(self, alerte, type_notification):
        """Envoie une alerte via webhook (Slack, Teams, etc.)"""
        try:
            webhook_url = self.config['webhook']['url']

            # Format Slack
            couleur = {
                'INFO': '#36a64f',      # Vert
                'WARNING': '#ff9900',   # Orange
                'CRITICAL': '#ff0000'   # Rouge
            }

            payload = {
                "text": f"🚨 Alerte SQLite: {alerte['base']}",
                "attachments": [{
                    "color": couleur.get(alerte['niveau'], '#808080'),
                    "fields": [
                        {
                            "title": "Base de données",
                            "value": alerte['base'],
                            "short": True
                        },
                        {
                            "title": "Type",
                            "value": alerte['type'],
                            "short": True
                        },
                        {
                            "title": "Niveau",
                            "value": alerte['niveau'],
                            "short": True
                        },
                        {
                            "title": "Message",
                            "value": alerte['message'],
                            "short": False
                        }
                    ],
                    "footer": "SQLite Monitoring",
                    "ts": int(alerte['timestamp'].timestamp())
                }]
            }

            # Simulation d'envoi
            print(f"🔗 Webhook simulé envoyé pour {alerte['base']} - {alerte['type']}")
            # requests.post(webhook_url, json=payload)

        except Exception as e:
            print(f"❌ Erreur envoi webhook: {e}")

    def _envoyer_sms(self, alerte, type_notification):
        """Envoie une alerte par SMS (simulation)"""
        if alerte['niveau'] == 'CRITICAL':
            message = f"URGENT SQLite: {alerte['base']} - {alerte['message']}"
            print(f"📱 SMS simulé envoyé: {message}")

    def resoudre_alerte(self, base, type_alerte):
        """Marque une alerte comme résolue"""
        cle_alerte = f"{base}_{type_alerte}"

        if cle_alerte in self.alertes_actives:
            alerte_resolue = self.alertes_actives.pop(cle_alerte)

            # Notification de résolution
            alerte_resolue['resolu'] = True
            alerte_resolue['timestamp_resolution'] = datetime.now()

            self._envoyer_notifications(alerte_resolue, 'RESOLUE')

            return True

        return False

    def generer_rapport_alertes(self, periode_jours=7):
        """Génère un rapport des alertes"""
        limite = datetime.now() - timedelta(days=periode_jours)
        alertes_periode = [a for a in self.historique_alertes if a['timestamp'] >= limite]

        print(f"\n🚨 === RAPPORT ALERTES ({periode_jours} jours) ===")
        print(f"Total alertes: {len(alertes_periode)}")
        print(f"Alertes actives: {len(self.alertes_actives)}")

        # Répartition par niveau
        niveaux = {}
        for alerte in alertes_periode:
            niveau = alerte['niveau']
            niveaux[niveau] = niveaux.get(niveau, 0) + 1

        print("\n📊 Répartition par niveau:")
        for niveau, count in niveaux.items():
            print(f"   • {niveau}: {count}")

        # Répartition par type
        types = {}
        for alerte in alertes_periode:
            type_alerte = alerte['type']
            types[type_alerte] = types.get(type_alerte, 0) + 1

        print("\n📋 Répartition par type:")
        for type_alerte, count in types.items():
            print(f"   • {type_alerte}: {count}")

        # Alertes actives
        if self.alertes_actives:
            print("\n🔴 Alertes actuellement actives:")
            for cle, alerte in self.alertes_actives.items():
                duree = datetime.now() - alerte['timestamp']
                print(f"   • {alerte['base']} - {alerte['type']} (depuis {duree})")

        return {
            'total_alertes': len(alertes_periode),
            'alertes_actives': len(self.alertes_actives),
            'repartition_niveaux': niveaux,
            'repartition_types': types
        }

# Configuration d'exemple
config_alertes = {
    'seuils': {
        'temps_reponse_ms': 1000,
        'fragmentation_pct': 20,
        'disque_percent': 85,
        'memoire_percent': 90
    },
    'email': {
        'actif': True,
        'from': 'monitoring@entreprise.com',
        'to': 'admin@entreprise.com',
        'smtp_server': 'smtp.entreprise.com'
    },
    'webhook': {
        'actif': True,
        'url': 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
    },
    'sms': {
        'actif': True,
        'numero': '+33123456789'
    },
    'dashboard_url': 'http://monitoring.entreprise.com:5000'
}

# Utilisation du système d'alertes
systeme_alertes = SystemeAlertes(config_alertes)

print("=== DÉMONSTRATION SYSTÈME D'ALERTES ===")

# Simuler des données de monitoring avec problèmes
donnees_test = {
    'ma_base.db': {
        'performance': {
            'temps_reponse_ms': 1500,  # Au-dessus du seuil
            'fragmentation_pct': 25,   # Au-dessus du seuil
            'taille_mb': 150
        },
        'integrite': {
            'statut': 'OK'
        },
        'systeme': {
            'disque_percent': 90,      # Au-dessus du seuil
            'memoire_percent': 75
        }
    }
}

# Évaluer les alertes
alertes = systeme_alertes.evaluer_alertes(donnees_test)
print(f"\n📊 {len(alertes)} alertes générées")

# Générer un rapport
rapport = systeme_alertes.generer_rapport_alertes(7)
```

## Conclusion et recommandations

### Bonnes pratiques de monitoring SQLite

Le monitoring et la maintenance préventive sont essentiels pour maintenir des performances optimales et éviter les problèmes critiques. Voici les recommandations clés :

#### **🔑 Principes de base**

1. **Monitoring proactif** - Surveillez avant que les problèmes ne surviennent
2. **Automatisation** - Automatisez la collecte de métriques et la maintenance
3. **Alertes intelligentes** - Configurez des seuils pertinents pour éviter le bruit
4. **Documentation** - Documentez vos procédures et seuils
5. **Tests réguliers** - Testez vos alertes et procédures de maintenance

#### **📊 Métriques essentielles à surveiller**

```python
metriques_critiques = {
    'performance': {
        'temps_reponse_ms': '<1000',
        'fragmentation_pct': '<20',
        'taille_croissance_quotidienne': '<5MB'
    },
    'integrite': {
        'integrity_check': 'ok',
        'foreign_key_violations': '0',
        'corruption_detectee': 'false'
    },
    'systeme': {
        'cpu_utilisation': '<80%',
        'memoire_disponible': '>20%',
        'espace_disque': '>15%'
    },
    'maintenance': {
        'derniere_maintenance': '<7_jours',
        'vacuum_recommande': 'false',
        'erreurs_maintenance': '0'
    }
}
```

#### **🔧 Calendrier de maintenance recommandé**

```python
planning_maintenance = {
    'quotidien': {
        'heure': '03:00',
        'operations': [
            'verification_integrite_rapide',
            'collecte_metriques',
            'nettoyage_cache',
            'optimisation_legere'
        ],
        'duree_estimee': '5-10 minutes'
    },
    'hebdomadaire': {
        'jour': 'dimanche',
        'heure': '02:00',
        'operations': [
            'reindexation_complete',
            'analyse_statistiques',
            'verification_fragmentation',
            'nettoyage_logs'
        ],
        'duree_estimee': '15-30 minutes'
    },
    'mensuel': {
        'jour': '1er du mois',
        'heure': '01:00',
        'operations': [
            'vacuum_complet',
            'verification_integrite_complete',
            'analyse_tendances',
            'archivage_logs',
            'test_restauration'
        ],
        'duree_estimee': '30-60 minutes'
    }
}
```

#### **🚨 Configuration d'alertes par criticité**

```python
niveaux_alertes = {
    'CRITIQUE': {
        'exemples': [
            'corruption_donnees',
            'base_inaccessible',
            'espace_disque_<5%',
            'erreur_integrite'
        ],
        'notifications': ['email', 'sms', 'webhook'],
        'delai_notification': 'immediat',
        'escalade': '15_minutes_si_non_acknowledge'
    },
    'AVERTISSEMENT': {
        'exemples': [
            'temps_reponse_>2s',
            'fragmentation_>25%',
            'maintenance_en_retard',
            'espace_disque_<15%'
        ],
        'notifications': ['email', 'webhook'],
        'delai_notification': '5_minutes',
        'escalade': '1_heure_si_non_resolue'
    },
    'INFO': {
        'exemples': [
            'maintenance_terminee',
            'croissance_normale_detectee',
            'optimisation_reussie'
        ],
        'notifications': ['webhook'],
        'delai_notification': 'batch_quotidien',
        'escalade': 'aucune'
    }
}
```

#### **📈 Tableaux de bord recommandés**

```python
dashboards_recommandes = {
    'vue_operationnelle': {
        'audience': 'equipe_dev_ops',
        'metriques': [
            'statut_temps_reel',
            'alertes_actives',
            'performance_requetes',
            'utilisation_ressources'
        ],
        'frequence_maj': '30_secondes'
    },
    'vue_strategique': {
        'audience': 'management',
        'metriques': [
            'disponibilite_mensuelle',
            'tendances_croissance',
            'couts_maintenance',
            'incidents_majeurs'
        ],
        'frequence_maj': 'quotidienne'
    },
    'vue_technique': {
        'audience': 'dba_developpeurs',
        'metriques': [
            'plans_execution',
            'index_utilisation',
            'locks_contentions',
            'io_patterns'
        ],
        'frequence_maj': 'temps_reel'
    }
}
```

### **📋 Checklist de déploiement monitoring**

```markdown
# ✅ CHECKLIST MONITORING ET MAINTENANCE SQLITE

## 📊 Configuration de base
□ Métriques de performance configurées
□ Vérification d'intégrité automatisée
□ Surveillance ressources système activée
□ Seuils d'alerte définis et testés

## 🔧 Maintenance préventive
□ Planning de maintenance configuré
□ Scripts de maintenance testés
□ Fenêtres de maintenance définies
□ Procédures d'urgence documentées

## 🚨 Système d'alertes
□ Notifications email configurées
□ Webhooks Slack/Teams fonctionnels
□ Escalade automatique en place
□ Tests d'alertes effectués

## 📈 Dashboard et reporting
□ Dashboard web déployé
□ Collecte de données automatisée
□ Rapports périodiques configurés
□ Accès équipe configuré

## 🔄 Automatisation
□ Scripts cron/scheduler configurés
□ Logs de maintenance centralisés
□ Archivage automatique en place
□ Monitoring du monitoring activé

## 📚 Documentation
□ Runbooks maintenance créés
□ Procédures d'escalade documentées
□ Contacts d'urgence mis à jour
□ Formation équipe effectuée
```

### **🎯 Recommandations par environnement**

#### **Production**
- Monitoring 24/7 avec alertes immédiates
- Maintenance pendant fenêtres planifiées uniquement
- Sauvegarde avant chaque maintenance
- Dashboard accessible à l'équipe d'astreinte

#### **Pré-production/Staging**
- Tests de nouvelles procédures de maintenance
- Validation des seuils d'alerte
- Simulation de pannes et récupération
- Formation équipe sur nouveaux outils

#### **Développement**
- Monitoring basique pour détecter les patterns
- Maintenance moins fréquente
- Tests de performance sur datasets réels
- Expérimentation de nouvelles métriques

### **🔮 Évolutions futures**

```python
roadmap_monitoring = {
    'court_terme': [
        'integration_ia_detection_anomalies',
        'predictions_maintenance_preventive',
        'auto_tuning_parametres',
        'correlation_cross_bases'
    ],
    'moyen_terme': [
        'machine_learning_optimisation',
        'monitoring_distribue',
        'analytics_avances',
        'automation_complete_maintenance'
    ],
    'long_terme': [
        'self_healing_databases',
        'optimisation_continue_autonome',
        'predictive_scaling',
        'zero_downtime_maintenance'
    ]
}
```

### **📚 Ressources et outils**

#### **Outils de monitoring**
- **Grafana + InfluxDB** : Pour dashboards avancés
- **Prometheus** : Collecte de métriques time-series
- **Nagios/Zabbix** : Monitoring d'infrastructure
- **Custom Python scripts** : Monitoring spécialisé SQLite

#### **Solutions cloud**
- **AWS CloudWatch** : Pour instances EC2
- **Google Cloud Monitoring** : Pour GCP
- **Azure Monitor** : Pour Microsoft Azure
- **Datadog/New Relic** : Solutions SaaS complètes

#### **Bibliothèques utiles**
```python
outils_recommandes = {
    'python': [
        'psutil',      # Métriques système
        'schedule',    # Planification tâches
        'flask',       # Dashboard web
        'requests',    # Notifications HTTP
        'sqlite3'      # Interface SQLite native
    ],
    'monitoring': [
        'prometheus_client',  # Exposition métriques
        'grafana_api',       # Intégration Grafana
        'influxdb_client',   # Base time-series
        'elasticsearch',     # Recherche logs
    ],
    'alertes': [
        'smtplib',          # Email
        'twilio',           # SMS
        'slack_sdk',        # Slack
        'pagerduty',        # Escalade
    ]
}
```

Le monitoring et la maintenance préventive transforment votre approche de SQLite d'une gestion réactive à une gestion proactive. Investir dans ces outils et processus vous fait économiser du temps, évite les pannes critiques et améliore significativement la fiabilité de vos applications.

**💡 Conseil final :** Commencez simple avec les métriques de base (performance, intégrité, taille) et une maintenance hebdomadaire, puis enrichissez progressivement votre système selon vos besoins opérationnels.

---

*Cette section conclut le chapitre 8.5 sur le monitoring et la maintenance préventive SQLite, et complète l'ensemble du chapitre 8 sur la sécurité et l'administration des bases de données SQLite.*

⏭️
