🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.5 : Synchronisation et réplication de données

## Introduction

Imaginez que vous développez une application mobile qui doit fonctionner même sans connexion Internet, ou une application web où plusieurs utilisateurs travaillent simultanément sur les mêmes données. C'est là qu'interviennent la synchronisation et la réplication de données.

Dans cette section, nous allons apprendre à :
- **Synchroniser** des données entre différentes bases SQLite
- **Répliquer** des données pour assurer la disponibilité
- **Gérer les conflits** quand plusieurs personnes modifient les mêmes données
- **Créer des applications** qui fonctionnent hors ligne

## Concepts de base

### Qu'est-ce que la synchronisation ?

La **synchronisation** consiste à maintenir plusieurs copies d'une base de données à jour. Pensez à vos contacts sur votre téléphone qui se synchronisent avec le cloud, ou à un document Google Docs qui se met à jour en temps réel.

### Qu'est-ce que la réplication ?

La **réplication** crée des copies identiques d'une base de données sur plusieurs serveurs pour assurer la disponibilité et améliorer les performances.

### Pourquoi synchroniser avec SQLite ?

SQLite est parfait pour :
- **Applications mobiles** : Base locale + synchronisation cloud
- **Applications hors ligne** : Travailler sans connexion
- **Performance** : Données locales = accès rapide
- **Simplicité** : Pas de serveur de base de données complexe

## Types de synchronisation

### 1. Synchronisation unidirectionnelle (One-way)

Les données vont dans un seul sens : de la source vers la destination.

```
Serveur Central → Application Mobile
     (maître)         (esclave)
```

**Exemple** : Une application de catalogue produits où seul le serveur peut modifier les données.

### 2. Synchronisation bidirectionnelle (Two-way)

Les données peuvent être modifiées des deux côtés et se synchronisent mutuellement.

```
Application A ⟷ Serveur Central ⟷ Application B
```

**Exemple** : Une application de notes où vous pouvez modifier depuis votre téléphone ou votre ordinateur.

### 3. Synchronisation multi-maître

Plusieurs sources peuvent modifier les données simultanément.

```
App Mobile A ⟷ App Mobile B ⟷ App Web
```

**Exemple** : Une application collaborative comme un chat ou un tableau partagé.

## Architecture de base pour la synchronisation

### Composants essentiels

1. **Timestamp/Version** : Pour savoir quelle donnée est la plus récente
2. **Identifiants uniques** : Pour identifier chaque enregistrement
3. **Journal des modifications** : Pour traquer les changements
4. **Résolution de conflits** : Pour gérer les modifications simultanées

### Structure de table avec synchronisation

```sql
CREATE TABLE personnes (
    id TEXT PRIMARY KEY,              -- UUID unique
    nom TEXT NOT NULL,
    email TEXT,
    age INTEGER,

    -- Métadonnées de synchronisation
    created_at INTEGER NOT NULL,      -- Timestamp de création
    updated_at INTEGER NOT NULL,      -- Timestamp de modification
    version INTEGER DEFAULT 1,        -- Version de l'enregistrement
    is_deleted INTEGER DEFAULT 0,     -- Suppression logique
    last_sync INTEGER DEFAULT 0       -- Dernière synchronisation
);

-- Table pour traquer les changements
CREATE TABLE sync_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name TEXT NOT NULL,
    record_id TEXT NOT NULL,
    operation TEXT NOT NULL,          -- INSERT, UPDATE, DELETE
    timestamp INTEGER NOT NULL,
    data TEXT,                        -- Données JSON de l'enregistrement
    synced INTEGER DEFAULT 0          -- 0 = pas encore synchronisé
);
```

## Implémentation simple de synchronisation

### 1. Système de synchronisation de base

**sync-manager.js**
```javascript
const sqlite3 = require('sqlite3').verbose();
const { v4: uuidv4 } = require('uuid');

class SyncManager {
    constructor(dbPath) {
        this.db = new sqlite3.Database(dbPath);
        this.setupTables();
        this.setupTriggers();
    }

    setupTables() {
        this.db.serialize(() => {
            // Table principale avec métadonnées de sync
            this.db.run(`
                CREATE TABLE IF NOT EXISTS personnes (
                    id TEXT PRIMARY KEY,
                    nom TEXT NOT NULL,
                    email TEXT,
                    age INTEGER,
                    created_at INTEGER NOT NULL,
                    updated_at INTEGER NOT NULL,
                    version INTEGER DEFAULT 1,
                    is_deleted INTEGER DEFAULT 0,
                    last_sync INTEGER DEFAULT 0
                )
            `);

            // Journal des modifications
            this.db.run(`
                CREATE TABLE IF NOT EXISTS sync_log (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    table_name TEXT NOT NULL,
                    record_id TEXT NOT NULL,
                    operation TEXT NOT NULL,
                    timestamp INTEGER NOT NULL,
                    data TEXT,
                    synced INTEGER DEFAULT 0
                )
            `);

            // Métadonnées de synchronisation
            this.db.run(`
                CREATE TABLE IF NOT EXISTS sync_metadata (
                    key TEXT PRIMARY KEY,
                    value TEXT,
                    updated_at INTEGER
                )
            `);
        });
    }

    setupTriggers() {
        // Trigger pour logger les insertions
        this.db.run(`
            CREATE TRIGGER IF NOT EXISTS log_insert_personnes
            AFTER INSERT ON personnes
            BEGIN
                INSERT INTO sync_log (table_name, record_id, operation, timestamp, data)
                VALUES (
                    'personnes',
                    NEW.id,
                    'INSERT',
                    strftime('%s', 'now') * 1000,
                    json_object(
                        'id', NEW.id,
                        'nom', NEW.nom,
                        'email', NEW.email,
                        'age', NEW.age,
                        'created_at', NEW.created_at,
                        'updated_at', NEW.updated_at,
                        'version', NEW.version
                    )
                );
            END
        `);

        // Trigger pour logger les mises à jour
        this.db.run(`
            CREATE TRIGGER IF NOT EXISTS log_update_personnes
            AFTER UPDATE ON personnes
            WHEN OLD.updated_at != NEW.updated_at
            BEGIN
                INSERT INTO sync_log (table_name, record_id, operation, timestamp, data)
                VALUES (
                    'personnes',
                    NEW.id,
                    'UPDATE',
                    strftime('%s', 'now') * 1000,
                    json_object(
                        'id', NEW.id,
                        'nom', NEW.nom,
                        'email', NEW.email,
                        'age', NEW.age,
                        'created_at', NEW.created_at,
                        'updated_at', NEW.updated_at,
                        'version', NEW.version,
                        'is_deleted', NEW.is_deleted
                    )
                );
            END
        `);

        // Trigger pour logger les suppressions
        this.db.run(`
            CREATE TRIGGER IF NOT EXISTS log_delete_personnes
            AFTER UPDATE ON personnes
            WHEN NEW.is_deleted = 1 AND OLD.is_deleted = 0
            BEGIN
                INSERT INTO sync_log (table_name, record_id, operation, timestamp, data)
                VALUES (
                    'personnes',
                    NEW.id,
                    'DELETE',
                    strftime('%s', 'now') * 1000,
                    json_object('id', NEW.id)
                );
            END
        `);
    }

    // Créer une nouvelle personne avec métadonnées de sync
    async creerPersonne(nom, email, age) {
        const id = uuidv4();
        const timestamp = Date.now();

        return new Promise((resolve, reject) => {
            this.db.run(`
                INSERT INTO personnes (id, nom, email, age, created_at, updated_at)
                VALUES (?, ?, ?, ?, ?, ?)
            `, [id, nom, email, age, timestamp, timestamp], function(err) {
                if (err) reject(err);
                else resolve({ id, nom, email, age, created_at: timestamp, updated_at: timestamp });
            });
        });
    }

    // Mettre à jour une personne
    async mettreAJourPersonne(id, donnees) {
        const timestamp = Date.now();
        const champs = [];
        const valeurs = [];

        // Construire la requête dynamiquement
        Object.keys(donnees).forEach(champ => {
            if (['nom', 'email', 'age'].includes(champ)) {
                champs.push(`${champ} = ?`);
                valeurs.push(donnees[champ]);
            }
        });

        if (champs.length === 0) {
            throw new Error('Aucun champ valide à mettre à jour');
        }

        champs.push('updated_at = ?', 'version = version + 1');
        valeurs.push(timestamp, id);

        return new Promise((resolve, reject) => {
            this.db.run(`
                UPDATE personnes
                SET ${champs.join(', ')}
                WHERE id = ? AND is_deleted = 0
            `, valeurs, function(err) {
                if (err) reject(err);
                else if (this.changes === 0) reject(new Error('Personne non trouvée'));
                else resolve({ id, ...donnees, updated_at: timestamp });
            });
        });
    }

    // Suppression logique
    async supprimerPersonne(id) {
        const timestamp = Date.now();

        return new Promise((resolve, reject) => {
            this.db.run(`
                UPDATE personnes
                SET is_deleted = 1, updated_at = ?, version = version + 1
                WHERE id = ? AND is_deleted = 0
            `, [timestamp, id], function(err) {
                if (err) reject(err);
                else if (this.changes === 0) reject(new Error('Personne non trouvée'));
                else resolve({ id, deleted: true });
            });
        });
    }

    // Récupérer les changements non synchronisés
    async getChangementsNonSynchro() {
        return new Promise((resolve, reject) => {
            this.db.all(`
                SELECT * FROM sync_log
                WHERE synced = 0
                ORDER BY timestamp ASC
            `, (err, rows) => {
                if (err) reject(err);
                else resolve(rows.map(row => ({
                    ...row,
                    data: row.data ? JSON.parse(row.data) : null
                })));
            });
        });
    }

    // Marquer les changements comme synchronisés
    async marquerCommeSynchro(logIds) {
        const placeholders = logIds.map(() => '?').join(',');

        return new Promise((resolve, reject) => {
            this.db.run(`
                UPDATE sync_log
                SET synced = 1
                WHERE id IN (${placeholders})
            `, logIds, function(err) {
                if (err) reject(err);
                else resolve(this.changes);
            });
        });
    }

    // Appliquer des changements depuis une source externe
    async appliquerChangements(changements) {
        const results = [];

        for (const changement of changements) {
            try {
                await this.appliquerChangement(changement);
                results.push({ success: true, changement });
            } catch (error) {
                results.push({ success: false, changement, error: error.message });
            }
        }

        return results;
    }

    async appliquerChangement(changement) {
        const { operation, data, timestamp } = changement;

        switch (operation) {
            case 'INSERT':
                await this.insertOuUpdate(data, timestamp);
                break;
            case 'UPDATE':
                await this.insertOuUpdate(data, timestamp);
                break;
            case 'DELETE':
                await this.supprimerLogiquement(data.id, timestamp);
                break;
            default:
                throw new Error(`Opération inconnue: ${operation}`);
        }
    }

    async insertOuUpdate(data, timestamp) {
        return new Promise((resolve, reject) => {
            // Vérifier si l'enregistrement existe déjà
            this.db.get('SELECT updated_at FROM personnes WHERE id = ?', [data.id], (err, row) => {
                if (err) {
                    reject(err);
                    return;
                }

                if (!row) {
                    // Insertion
                    this.db.run(`
                        INSERT INTO personnes (id, nom, email, age, created_at, updated_at, version, last_sync)
                        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                    `, [data.id, data.nom, data.email, data.age, data.created_at,
                        data.updated_at, data.version, timestamp], function(err) {
                        if (err) reject(err);
                        else resolve({ operation: 'INSERT', id: data.id });
                    });
                } else if (row.updated_at < data.updated_at) {
                    // Mise à jour seulement si plus récent
                    this.db.run(`
                        UPDATE personnes
                        SET nom = ?, email = ?, age = ?, updated_at = ?, version = ?, last_sync = ?
                        WHERE id = ?
                    `, [data.nom, data.email, data.age, data.updated_at,
                        data.version, timestamp, data.id], function(err) {
                        if (err) reject(err);
                        else resolve({ operation: 'UPDATE', id: data.id });
                    });
                } else {
                    // Conflit : données locales plus récentes
                    resolve({ operation: 'CONFLICT', id: data.id, message: 'Données locales plus récentes' });
                }
            });
        });
    }

    async supprimerLogiquement(id, timestamp) {
        return new Promise((resolve, reject) => {
            this.db.run(`
                UPDATE personnes
                SET is_deleted = 1, updated_at = ?, last_sync = ?
                WHERE id = ?
            `, [timestamp, timestamp, id], function(err) {
                if (err) reject(err);
                else resolve({ operation: 'DELETE', id });
            });
        });
    }

    // Obtenir le timestamp de la dernière synchronisation
    async getDerniereSync() {
        return new Promise((resolve, reject) => {
            this.db.get(`
                SELECT value FROM sync_metadata
                WHERE key = 'last_sync_timestamp'
            `, (err, row) => {
                if (err) reject(err);
                else resolve(row ? parseInt(row.value) : 0);
            });
        });
    }

    // Mettre à jour le timestamp de synchronisation
    async setDerniereSync(timestamp) {
        return new Promise((resolve, reject) => {
            this.db.run(`
                INSERT OR REPLACE INTO sync_metadata (key, value, updated_at)
                VALUES ('last_sync_timestamp', ?, ?)
            `, [timestamp.toString(), timestamp], function(err) {
                if (err) reject(err);
                else resolve();
            });
        });
    }
}

module.exports = SyncManager;
```

### 2. Client de synchronisation

**sync-client.js**
```javascript
const axios = require('axios');

class SyncClient {
    constructor(syncManager, serverUrl) {
        this.syncManager = syncManager;
        this.serverUrl = serverUrl;
        this.isOnline = true;
        this.syncInterval = null;
    }

    // Démarrer la synchronisation automatique
    startAutoSync(intervalMs = 30000) { // 30 secondes par défaut
        console.log('🔄 Démarrage de la synchronisation automatique...');

        this.syncInterval = setInterval(async () => {
            if (this.isOnline) {
                try {
                    await this.synchroniser();
                } catch (error) {
                    console.error('Erreur sync automatique:', error.message);
                }
            }
        }, intervalMs);
    }

    // Arrêter la synchronisation automatique
    stopAutoSync() {
        if (this.syncInterval) {
            clearInterval(this.syncInterval);
            this.syncInterval = null;
            console.log('⏹️ Synchronisation automatique arrêtée');
        }
    }

    // Synchronisation complète
    async synchroniser() {
        console.log('🔄 Début de la synchronisation...');

        try {
            // 1. Envoyer les changements locaux au serveur
            await this.envoyerChangementsLocaux();

            // 2. Récupérer les changements du serveur
            await this.recupererChangementsServeur();

            // 3. Mettre à jour le timestamp de synchronisation
            await this.syncManager.setDerniereSync(Date.now());

            console.log('✅ Synchronisation terminée avec succès');
            return { success: true };

        } catch (error) {
            console.error('❌ Erreur de synchronisation:', error.message);
            throw error;
        }
    }

    async envoyerChangementsLocaux() {
        const changements = await this.syncManager.getChangementsNonSynchro();

        if (changements.length === 0) {
            console.log('📤 Aucun changement local à envoyer');
            return;
        }

        console.log(`📤 Envoi de ${changements.length} changements au serveur...`);

        try {
            const response = await axios.post(`${this.serverUrl}/sync/push`, {
                changements: changements
            });

            if (response.data.success) {
                // Marquer les changements comme synchronisés
                const logIds = changements.map(c => c.id);
                await this.syncManager.marquerCommeSynchro(logIds);
                console.log('✅ Changements envoyés avec succès');
            } else {
                throw new Error('Échec de l\'envoi des changements');
            }

        } catch (error) {
            if (error.code === 'ECONNREFUSED' || error.code === 'ENOTFOUND') {
                this.isOnline = false;
                console.log('🔌 Mode hors ligne activé');
            }
            throw error;
        }
    }

    async recupererChangementsServeur() {
        const derniereSync = await this.syncManager.getDerniereSync();

        try {
            console.log('📥 Récupération des changements du serveur...');

            const response = await axios.get(`${this.serverUrl}/sync/pull`, {
                params: { since: derniereSync }
            });

            const changements = response.data.changements || [];

            if (changements.length === 0) {
                console.log('📥 Aucun changement serveur à appliquer');
                return;
            }

            console.log(`📥 Application de ${changements.length} changements du serveur...`);

            const results = await this.syncManager.appliquerChangements(changements);

            // Analyser les résultats
            const success = results.filter(r => r.success).length;
            const conflicts = results.filter(r => !r.success).length;

            console.log(`✅ ${success} changements appliqués`);
            if (conflicts > 0) {
                console.log(`⚠️ ${conflicts} conflits détectés`);
            }

            this.isOnline = true;

        } catch (error) {
            if (error.code === 'ECONNREFUSED' || error.code === 'ENOTFOUND') {
                this.isOnline = false;
                console.log('🔌 Mode hors ligne activé');
            }
            throw error;
        }
    }

    // Synchronisation manuelle
    async syncMaintenant() {
        console.log('🔄 Synchronisation manuelle demandée...');
        return await this.synchroniser();
    }

    // Vérifier le statut de connexion
    async verifierConnexion() {
        try {
            await axios.get(`${this.serverUrl}/health`, { timeout: 5000 });
            this.isOnline = true;
            return true;
        } catch (error) {
            this.isOnline = false;
            return false;
        }
    }

    // Obtenir le statut de synchronisation
    async getStatutSync() {
        const changements = await this.syncManager.getChangementsNonSynchro();
        const derniereSync = await this.syncManager.getDerniereSync();

        return {
            isOnline: this.isOnline,
            changementsEnAttente: changements.length,
            derniereSync: new Date(derniereSync),
            autoSyncActive: this.syncInterval !== null
        };
    }
}

module.exports = SyncClient;
```

### 3. Serveur de synchronisation simple

**sync-server.js**
```javascript
const express = require('express');
const SyncManager = require('./sync-manager');

const app = express();
app.use(express.json());

// Instance du gestionnaire de synchronisation côté serveur
const serverSyncManager = new SyncManager('./server-database.db');

// Health check
app.get('/health', (req, res) => {
    res.json({ status: 'ok', timestamp: Date.now() });
});

// Recevoir les changements des clients (push)
app.post('/sync/push', async (req, res) => {
    try {
        const { changements } = req.body;

        if (!changements || !Array.isArray(changements)) {
            return res.status(400).json({
                success: false,
                error: 'Changements invalides'
            });
        }

        console.log(`📥 Réception de ${changements.length} changements d'un client`);

        // Appliquer les changements sur le serveur
        const results = await serverSyncManager.appliquerChangements(changements);

        // Analyser les résultats
        const success = results.filter(r => r.success).length;
        const conflicts = results.filter(r => !r.success).length;

        console.log(`✅ ${success} changements appliqués, ${conflicts} conflits`);

        res.json({
            success: true,
            applied: success,
            conflicts: conflicts,
            results: results
        });

    } catch (error) {
        console.error('Erreur push sync:', error);
        res.status(500).json({
            success: false,
            error: error.message
        });
    }
});

// Envoyer les changements aux clients (pull)
app.get('/sync/pull', async (req, res) => {
    try {
        const since = parseInt(req.query.since) || 0;

        console.log(`📤 Envoi des changements depuis ${new Date(since)}`);

        // Récupérer les changements depuis le timestamp donné
        const changements = await getChangementsDepuis(since);

        console.log(`📤 ${changements.length} changements à envoyer`);

        res.json({
            success: true,
            changements: changements,
            timestamp: Date.now()
        });

    } catch (error) {
        console.error('Erreur pull sync:', error);
        res.status(500).json({
            success: false,
            error: error.message
        });
    }
});

// Fonction pour récupérer les changements depuis un timestamp
async function getChangementsDepuis(since) {
    return new Promise((resolve, reject) => {
        serverSyncManager.db.all(`
            SELECT * FROM sync_log
            WHERE timestamp > ?
            ORDER BY timestamp ASC
        `, [since], (err, rows) => {
            if (err) {
                reject(err);
            } else {
                const changements = rows.map(row => ({
                    id: row.id,
                    table_name: row.table_name,
                    record_id: row.record_id,
                    operation: row.operation,
                    timestamp: row.timestamp,
                    data: row.data ? JSON.parse(row.data) : null
                }));
                resolve(changements);
            }
        });
    });
}

// Endpoint pour obtenir toutes les personnes (utile pour debug)
app.get('/personnes', async (req, res) => {
    try {
        serverSyncManager.db.all(`
            SELECT * FROM personnes
            WHERE is_deleted = 0
            ORDER BY nom
        `, (err, rows) => {
            if (err) {
                res.status(500).json({ error: err.message });
            } else {
                res.json({ personnes: rows });
            }
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Démarrer le serveur
const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
    console.log(`🚀 Serveur de synchronisation démarré sur le port ${PORT}`);
    console.log(`📡 Endpoints disponibles :`);
    console.log(`   GET  /health`);
    console.log(`   POST /sync/push`);
    console.log(`   GET  /sync/pull`);
    console.log(`   GET  /personnes`);
});

module.exports = app;
```

## Application exemple complète

### 4. Application client avec interface

**app-client.js**
```javascript
const readline = require('readline');
const SyncManager = require('./sync-manager');
const SyncClient = require('./sync-client');

class AppClient {
    constructor() {
        this.syncManager = new SyncManager('./client-database.db');
        this.syncClient = new SyncClient(this.syncManager, 'http://localhost:3001');
        this.rl = readline.createInterface({
            input: process.stdin,
            output: process.stdout
        });
    }

    async demarrer() {
        console.log('🎉 Application de gestion de personnes avec synchronisation');
        console.log('📱 Base de données locale avec sync cloud\n');

        // Démarrer la synchronisation automatique
        this.syncClient.startAutoSync(10000); // Toutes les 10 secondes

        await this.afficherMenu();
    }

    async afficherMenu() {
        console.log('\n📋 Menu principal :');
        console.log('1. Ajouter une personne');
        console.log('2. Lister les personnes');
        console.log('3. Modifier une personne');
        console.log('4. Supprimer une personne');
        console.log('5. Synchroniser maintenant');
        console.log('6. Statut de synchronisation');
        console.log('7. Mode hors ligne/en ligne');
        console.log('0. Quitter');

        this.rl.question('\nChoisissez une option : ', async (choix) => {
            await this.traiterChoix(choix);
        });
    }

    async traiterChoix(choix) {
        try {
            switch (choix) {
                case '1':
                    await this.ajouterPersonne();
                    break;
                case '2':
                    await this.listerPersonnes();
                    break;
                case '3':
                    await this.modifierPersonne();
                    break;
                case '4':
                    await this.supprimerPersonne();
                    break;
                case '5':
                    await this.synchroniserMaintenant();
                    break;
                case '6':
                    await this.afficherStatutSync();
                    break;
                case '7':
                    await this.toggleModeConnexion();
                    break;
                case '0':
                    await this.quitter();
                    return;
                default:
                    console.log('❌ Option invalide');
            }
        } catch (error) {
            console.error('❌ Erreur:', error.message);
        }

        await this.afficherMenu();
    }

    async ajouterPersonne() {
        console.log('\n➕ Ajouter une nouvelle personne');

        const nom = await this.poserQuestion('Nom : ');
        const email = await this.poserQuestion('Email : ');
        const ageStr = await this.poserQuestion('Âge : ');
        const age = ageStr ? parseInt(ageStr) : null;

        try {
            const personne = await this.syncManager.creerPersonne(nom, email, age);
            console.log('✅ Personne ajoutée :', personne.nom);
            console.log('📤 Sera synchronisée automatiquement');
        } catch (error) {
            console.error('❌ Erreur lors de l\'ajout :', error.message);
        }
    }

    async listerPersonnes() {
        console.log('\n📋 Liste des personnes');

        return new Promise((resolve) => {
            this.syncManager.db.all(`
                SELECT * FROM personnes
                WHERE is_deleted = 0
                ORDER BY nom
            `, (err, rows) => {
                if (err) {
                    console.error('❌ Erreur :', err.message);
                } else if (rows.length === 0) {
                    console.log('📭 Aucune personne trouvée');
                } else {
                    console.log(`\n👥 ${rows.length} personne(s) :`);
                    rows.forEach((p, index) => {
                        const syncStatus = p.last_sync > 0 ? '🔄' : '📤';
                        console.log(`${index + 1}. ${p.nom} (${p.age} ans) - ${p.email} ${syncStatus}`);
                        console.log(`   ID: ${p.id} | Version: ${p.version}`);
                    });
                }
                resolve();
            });
        });
    }

    async modifierPersonne() {
        console.log('\n✏️ Modifier une personne');

        const id = await this.poserQuestion('ID de la personne : ');

        // Vérifier que la personne existe
        const personne = await this.obtenirPersonneParId(id);
        if (!personne) {
            console.log('❌ Personne non trouvée');
            return;
        }

        console.log(`\nPersonne actuelle : ${personne.nom} (${personne.age} ans) - ${personne.email}`);
        console.log('Laissez vide pour conserver la valeur actuelle\n');

        const nom = await this.poserQuestion(`Nouveau nom [${personne.nom}] : `);
        const email = await this.poserQuestion(`Nouvel email [${personne.email}] : `);
        const ageStr = await this.poserQuestion(`Nouvel âge [${personne.age}] : `);

        const donnees = {};
        if (nom) donnees.nom = nom;
        if (email) donnees.email = email;
        if (ageStr) donnees.age = parseInt(ageStr);

        if (Object.keys(donnees).length === 0) {
            console.log('📝 Aucune modification effectuée');
            return;
        }

        try {
            await this.syncManager.mettreAJourPersonne(id, donnees);
            console.log('✅ Personne modifiée avec succès');
            console.log('📤 Sera synchronisée automatiquement');
        } catch (error) {
            console.error('❌ Erreur lors de la modification :', error.message);
        }
    }

    async supprimerPersonne() {
        console.log('\n🗑️ Supprimer une personne');

        const id = await this.poserQuestion('ID de la personne : ');

        const personne = await this.obtenirPersonneParId(id);
        if (!personne) {
            console.log('❌ Personne non trouvée');
            return;
        }

        console.log(`\nPersonne à supprimer : ${personne.nom} (${personne.age} ans)`);
        const confirmation = await this.poserQuestion('Confirmer la suppression ? (oui/non) : ');

        if (confirmation.toLowerCase() === 'oui') {
            try {
                await this.syncManager.supprimerPersonne(id);
                console.log('✅ Personne supprimée (suppression logique)');
                console.log('📤 Sera synchronisée automatiquement');
            } catch (error) {
                console.error('❌ Erreur lors de la suppression :', error.message);
            }
        } else {
            console.log('❌ Suppression annulée');
        }
    }

    async synchroniserMaintenant() {
        console.log('\n🔄 Synchronisation manuelle...');

        try {
            await this.syncClient.syncMaintenant();
            console.log('✅ Synchronisation terminée');
        } catch (error) {
            console.error('❌ Erreur de synchronisation :', error.message);
            if (error.code === 'ECONNREFUSED') {
                console.log('🔌 Serveur non disponible - travail en mode hors ligne');
            }
        }
    }

    async afficherStatutSync() {
        console.log('\n📊 Statut de synchronisation');

        try {
            const statut = await this.syncClient.getStatutSync();

            console.log(`🌐 Connexion : ${statut.isOnline ? '✅ En ligne' : '🔌 Hors ligne'}`);
            console.log(`🔄 Sync automatique : ${statut.autoSyncActive ? '✅ Activée' : '❌ Désactivée'}`);
            console.log(`📤 Changements en attente : ${statut.changementsEnAttente}`);
            console.log(`🕒 Dernière sync : ${statut.derniereSync.toLocaleString()}`);

            if (statut.changementsEnAttente > 0) {
                console.log('\n⚠️ Des modifications locales attendent d\'être synchronisées');
            }

        } catch (error) {
            console.error('❌ Erreur :', error.message);
        }
    }

    async toggleModeConnexion() {
        const isOnline = await this.syncClient.verifierConnexion();

        if (isOnline) {
            console.log('\n🔌 Mode hors ligne simulé');
            this.syncClient.isOnline = false;
            console.log('📱 Vous pouvez continuer à travailler localement');
        } else {
            console.log('\n🌐 Tentative de reconnexion...');
            const reconnected = await this.syncClient.verifierConnexion();
            if (reconnected) {
                console.log('✅ Reconnecté ! Synchronisation en cours...');
                try {
                    await this.syncClient.syncMaintenant();
                } catch (error) {
                    console.error('❌ Erreur sync après reconnexion :', error.message);
                }
            } else {
                console.log('❌ Impossible de se connecter au serveur');
            }
        }
    }

    async obtenirPersonneParId(id) {
        return new Promise((resolve, reject) => {
            this.syncManager.db.get(`
                SELECT * FROM personnes
                WHERE id = ? AND is_deleted = 0
            `, [id], (err, row) => {
                if (err) reject(err);
                else resolve(row);
            });
        });
    }

    async poserQuestion(question) {
        return new Promise((resolve) => {
            this.rl.question(question, (reponse) => {
                resolve(reponse.trim());
            });
        });
    }

    async quitter() {
        console.log('\n👋 Arrêt de l\'application...');

        // Arrêter la synchronisation automatique
        this.syncClient.stopAutoSync();

        // Dernière synchronisation
        try {
            console.log('🔄 Synchronisation finale...');
            await this.syncClient.syncMaintenant();
        } catch (error) {
            console.log('⚠️ Synchronisation finale échouée :', error.message);
        }

        this.rl.close();
        process.exit(0);
    }
}

// Démarrer l'application
const app = new AppClient();
app.demarrer().catch(console.error);

module.exports = AppClient;
```

## Gestion avancée des conflits

### 1. Détection et résolution automatique

**conflict-resolver.js**
```javascript
class ConflictResolver {
    constructor(syncManager) {
        this.syncManager = syncManager;
        this.strategies = {
            'timestamp': this.resolveByTimestamp.bind(this),
            'version': this.resolveByVersion.bind(this),
            'manual': this.resolveManually.bind(this),
            'merge': this.mergeFields.bind(this)
        };
    }

    async detecterConflits(changementServeur, changementLocal) {
        // Vérifier si c'est le même enregistrement
        if (changementServeur.record_id !== changementLocal.record_id) {
            return null; // Pas de conflit
        }

        // Vérifier si les timestamps se chevauchent
        const timeDiff = Math.abs(changementServeur.timestamp - changementLocal.timestamp);
        const isConflict = timeDiff < 60000; // Conflit si moins de 1 minute d'écart

        if (isConflict) {
            return {
                type: 'concurrent_modification',
                serveur: changementServeur,
                local: changementLocal,
                timeDiff: timeDiff
            };
        }

        return null;
    }

    async resoudreConflit(conflit, strategie = 'timestamp') {
        console.log(`⚔️ Résolution de conflit avec stratégie: ${strategie}`);

        const resolver = this.strategies[strategie];
        if (!resolver) {
            throw new Error(`Stratégie de résolution inconnue: ${strategie}`);
        }

        return await resolver(conflit);
    }

    // Stratégie 1: Le plus récent gagne (timestamp)
    async resolveByTimestamp(conflit) {
        const { serveur, local } = conflit;

        if (serveur.timestamp > local.timestamp) {
            console.log('🏆 Changement serveur plus récent - appliqué');
            return {
                resolution: 'server_wins',
                action: 'apply_server_change',
                winner: serveur
            };
        } else {
            console.log('🏆 Changement local plus récent - conservé');
            return {
                resolution: 'local_wins',
                action: 'keep_local_change',
                winner: local
            };
        }
    }

    // Stratégie 2: Version la plus élevée gagne
    async resolveByVersion(conflit) {
        const { serveur, local } = conflit;

        const serverVersion = serveur.data?.version || 0;
        const localVersion = local.data?.version || 0;

        if (serverVersion > localVersion) {
            console.log('🏆 Version serveur plus élevée - appliquée');
            return {
                resolution: 'server_wins',
                action: 'apply_server_change',
                winner: serveur
            };
        } else {
            console.log('🏆 Version locale plus élevée - conservée');
            return {
                resolution: 'local_wins',
                action: 'keep_local_change',
                winner: local
            };
        }
    }

    // Stratégie 3: Fusion intelligente des champs
    async mergeFields(conflit) {
        const { serveur, local } = conflit;

        const serverData = serveur.data;
        const localData = local.data;

        // Fusion field par field
        const merged = { ...localData };

        // Règles de fusion spécifiques
        if (serverData.nom && serverData.nom !== localData.nom) {
            // Si les noms sont différents, garder le plus long (plus d'infos)
            if (serverData.nom.length > localData.nom.length) {
                merged.nom = serverData.nom;
            }
        }

        if (serverData.email && serverData.email !== localData.email) {
            // Pour l'email, toujours prendre le plus récent
            if (serveur.timestamp > local.timestamp) {
                merged.email = serverData.email;
            }
        }

        if (serverData.age && serverData.age !== localData.age) {
            // Pour l'âge, prendre la valeur non-nulle ou la plus récente
            if (!localData.age && serverData.age) {
                merged.age = serverData.age;
            }
        }

        console.log('🔀 Fusion des champs effectuée');

        return {
            resolution: 'merged',
            action: 'apply_merged_data',
            mergedData: merged,
            changes: this.calculateChanges(localData, merged)
        };
    }

    // Stratégie 4: Résolution manuelle (pour les interfaces utilisateur)
    async resolveManually(conflit) {
        console.log('👤 Résolution manuelle requise');

        return {
            resolution: 'manual_required',
            action: 'prompt_user',
            options: {
                keepLocal: conflit.local,
                useServer: conflit.serveur,
                merge: await this.mergeFields(conflit)
            }
        };
    }

    calculateChanges(original, modified) {
        const changes = {};

        Object.keys(modified).forEach(key => {
            if (original[key] !== modified[key]) {
                changes[key] = {
                    from: original[key],
                    to: modified[key]
                };
            }
        });

        return changes;
    }

    // Appliquer la résolution de conflit
    async appliquerResolution(resolution) {
        switch (resolution.action) {
            case 'apply_server_change':
                return await this.syncManager.appliquerChangement(resolution.winner);

            case 'keep_local_change':
                // Ne rien faire, garder la version locale
                return { status: 'local_kept' };

            case 'apply_merged_data':
                return await this.appliquerDonneesFusionnees(resolution.mergedData);

            case 'prompt_user':
                throw new Error('Résolution manuelle requise');

            default:
                throw new Error(`Action de résolution inconnue: ${resolution.action}`);
        }
    }

    async appliquerDonneesFusionnees(mergedData) {
        return new Promise((resolve, reject) => {
            const timestamp = Date.now();

            this.syncManager.db.run(`
                UPDATE personnes
                SET nom = ?, email = ?, age = ?, updated_at = ?, version = version + 1
                WHERE id = ?
            `, [mergedData.nom, mergedData.email, mergedData.age, timestamp, mergedData.id],
            function(err) {
                if (err) reject(err);
                else resolve({ status: 'merged', changes: this.changes });
            });
        });
    }
}

module.exports = ConflictResolver;
```

### 2. Interface de résolution de conflits

**conflict-ui.js**
```javascript
const readline = require('readline');

class ConflictUI {
    constructor(conflictResolver) {
        this.resolver = conflictResolver;
        this.rl = readline.createInterface({
            input: process.stdin,
            output: process.stdout
        });
    }

    async gererConflit(conflit) {
        console.log('\n⚔️ CONFLIT DÉTECTÉ ⚔️');
        console.log('━'.repeat(50));

        await this.afficherDetailsConflit(conflit);

        const choix = await this.demanderResolution();

        switch (choix) {
            case '1':
                return await this.resolver.resoudreConflit(conflit, 'timestamp');
            case '2':
                return await this.garderLocal(conflit);
            case '3':
                return await this.prendreServeur(conflit);
            case '4':
                return await this.fusionnerManuellement(conflit);
            case '5':
                return await this.ignorer(conflit);
            default:
                console.log('❌ Choix invalide, résolution automatique par timestamp');
                return await this.resolver.resoudreConflit(conflit, 'timestamp');
        }
    }

    async afficherDetailsConflit(conflit) {
        const { serveur, local } = conflit;

        console.log(`📋 Enregistrement: ${serveur.record_id}`);
        console.log(`⏰ Écart temporel: ${Math.round(conflit.timeDiff / 1000)}s`);
        console.log('');

        // Afficher les versions en conflit
        console.log('🔴 VERSION LOCALE:');
        this.afficherDonnees(local.data, local.timestamp);

        console.log('\n🔵 VERSION SERVEUR:');
        this.afficherDonnees(serveur.data, serveur.timestamp);

        console.log('\n🔍 DIFFÉRENCES:');
        this.afficherDifferences(local.data, serveur.data);
    }

    afficherDonnees(data, timestamp) {
        if (!data) {
            console.log('  (Supprimé)');
            return;
        }

        console.log(`  Nom: ${data.nom || 'N/A'}`);
        console.log(`  Email: ${data.email || 'N/A'}`);
        console.log(`  Âge: ${data.age || 'N/A'}`);
        console.log(`  Version: ${data.version || 'N/A'}`);
        console.log(`  Modifié: ${new Date(timestamp).toLocaleString()}`);
    }

    afficherDifferences(local, serveur) {
        const champs = ['nom', 'email', 'age'];
        let differences = false;

        champs.forEach(champ => {
            if (local?.[champ] !== serveur?.[champ]) {
                differences = true;
                console.log(`  ${champ}: "${local?.[champ] || 'N/A'}" → "${serveur?.[champ] || 'N/A'}"`);
            }
        });

        if (!differences) {
            console.log('  (Aucune différence de contenu détectée)');
        }
    }

    async demanderResolution() {
        console.log('\n🎯 Comment résoudre ce conflit ?');
        console.log('1. Automatique (le plus récent gagne)');
        console.log('2. Garder la version locale');
        console.log('3. Prendre la version serveur');
        console.log('4. Fusionner manuellement');
        console.log('5. Ignorer ce conflit');

        return new Promise((resolve) => {
            this.rl.question('\nVotre choix (1-5) : ', (reponse) => {
                resolve(reponse.trim());
            });
        });
    }

    async garderLocal(conflit) {
        console.log('🔴 Conservation de la version locale');
        return {
            resolution: 'local_wins',
            action: 'keep_local_change',
            winner: conflit.local
        };
    }

    async prendreServeur(conflit) {
        console.log('🔵 Application de la version serveur');
        return {
            resolution: 'server_wins',
            action: 'apply_server_change',
            winner: conflit.serveur
        };
    }

    async fusionnerManuellement(conflit) {
        console.log('\n🔀 Fusion manuelle des données');

        const local = conflit.local.data;
        const serveur = conflit.serveur.data;
        const fusionne = { ...local };

        // Demander pour chaque champ différent
        const champs = ['nom', 'email', 'age'];

        for (const champ of champs) {
            if (local?.[champ] !== serveur?.[champ]) {
                const valeur = await this.choisirValeur(champ, local?.[champ], serveur?.[champ]);
                if (valeur !== null) {
                    fusionne[champ] = valeur;
                }
            }
        }

        console.log('✅ Fusion terminée');

        return {
            resolution: 'merged',
            action: 'apply_merged_data',
            mergedData: fusionne
        };
    }

    async choisirValeur(champ, valeurLocale, valeurServeur) {
        console.log(`\n${champ.toUpperCase()}:`);
        console.log(`1. Local: "${valeurLocale || 'N/A'}"`);
        console.log(`2. Serveur: "${valeurServeur || 'N/A'}"`);
        console.log(`3. Saisir une nouvelle valeur`);

        return new Promise((resolve) => {
            this.rl.question(`Choisir pour ${champ} (1-3) : `, (choix) => {
                switch (choix.trim()) {
                    case '1':
                        resolve(valeurLocale);
                        break;
                    case '2':
                        resolve(valeurServeur);
                        break;
                    case '3':
                        this.rl.question(`Nouvelle valeur pour ${champ} : `, (nouvelleValeur) => {
                            resolve(nouvelleValeur.trim() || null);
                        });
                        break;
                    default:
                        console.log('Choix invalide, conservation de la valeur locale');
                        resolve(valeurLocale);
                }
            });
        });
    }

    async ignorer(conflit) {
        console.log('⏭️ Conflit ignoré');
        return {
            resolution: 'ignored',
            action: 'no_action',
            reason: 'User chose to ignore conflict'
        };
    }

    close() {
        this.rl.close();
    }
}

module.exports = ConflictUI;
```

## Optimisations pour la performance

### 1. Synchronisation incrémentale

**incremental-sync.js**
```javascript
class IncrementalSync {
    constructor(syncManager) {
        this.syncManager = syncManager;
        this.batchSize = 100; // Traiter par lots de 100
    }

    async synchroniserIncrementalement() {
        console.log('🔄 Synchronisation incrémentale...');

        // 1. Synchroniser par petits lots
        await this.syncParLots();

        // 2. Optimiser la base de données après sync
        await this.optimiserApresSync();

        // 3. Nettoyer les anciens logs
        await this.nettoyerAncienLogs();
    }

    async syncParLots() {
        let offset = 0;
        let hasMore = true;

        while (hasMore) {
            const changements = await this.getChangementsParLot(offset, this.batchSize);

            if (changements.length === 0) {
                hasMore = false;
                break;
            }

            console.log(`📦 Traitement du lot ${offset / this.batchSize + 1} (${changements.length} éléments)`);

            // Traiter le lot
            await this.traiterLot(changements);

            offset += this.batchSize;

            // Petite pause pour éviter de surcharger
            await this.pause(100);
        }
    }

    async getChangementsParLot(offset, limit) {
        return new Promise((resolve, reject) => {
            this.syncManager.db.all(`
                SELECT * FROM sync_log
                WHERE synced = 0
                ORDER BY timestamp ASC
                LIMIT ? OFFSET ?
            `, [limit, offset], (err, rows) => {
                if (err) reject(err);
                else resolve(rows);
            });
        });
    }

    async traiterLot(changements) {
        // Traiter les changements par type pour optimiser
        const parType = this.grouperParType(changements);

        for (const [type, items] of Object.entries(parType)) {
            await this.traiterParType(type, items);
        }
    }

    grouperParType(changements) {
        return changements.reduce((groupes, changement) => {
            const type = changement.operation;
            if (!groupes[type]) groupes[type] = [];
            groupes[type].push(changement);
            return groupes;
        }, {});
    }

    async traiterParType(type, changements) {
        switch (type) {
            case 'INSERT':
                await this.traiterInsertions(changements);
                break;
            case 'UPDATE':
                await this.traiterMisesAJour(changements);
                break;
            case 'DELETE':
                await this.traiterSuppressions(changements);
                break;
        }
    }

    async traiterInsertions(insertions) {
        // Préparer une insertion en lot
        const values = insertions.map(ins => {
            const data = JSON.parse(ins.data);
            return [data.id, data.nom, data.email, data.age, data.created_at, data.updated_at, data.version];
        });

        const placeholders = values.map(() => '(?, ?, ?, ?, ?, ?, ?)').join(', ');
        const flatValues = values.flat();

        return new Promise((resolve, reject) => {
            this.syncManager.db.run(`
                INSERT OR IGNORE INTO personnes (id, nom, email, age, created_at, updated_at, version)
                VALUES ${placeholders}
            `, flatValues, function(err) {
                if (err) reject(err);
                else {
                    console.log(`  ➕ ${this.changes} insertions traitées`);
                    resolve();
                }
            });
        });
    }

    async traiterMisesAJour(mises_a_jour) {
        // Traiter les mises à jour une par une (plus complexe à grouper)
        let count = 0;

        for (const maj of mises_a_jour) {
            try {
                await this.syncManager.appliquerChangement(maj);
                count++;
            } catch (error) {
                console.warn(`⚠️ Erreur mise à jour ${maj.record_id}:`, error.message);
            }
        }

        console.log(`  ✏️ ${count} mises à jour traitées`);
    }

    async traiterSuppressions(suppressions) {
        const ids = suppressions.map(sup => JSON.parse(sup.data).id);
        const placeholders = ids.map(() => '?').join(', ');

        return new Promise((resolve, reject) => {
            this.syncManager.db.run(`
                UPDATE personnes
                SET is_deleted = 1, updated_at = strftime('%s', 'now') * 1000
                WHERE id IN (${placeholders})
            `, ids, function(err) {
                if (err) reject(err);
                else {
                    console.log(`  🗑️ ${this.changes} suppressions traitées`);
                    resolve();
                }
            });
        });
    }

    async optimiserApresSync() {
        return new Promise((resolve, reject) => {
            console.log('⚡ Optimisation de la base de données...');

            this.syncManager.db.serialize(() => {
                this.syncManager.db.run('ANALYZE', (err) => {
                    if (err) console.warn('⚠️ Erreur ANALYZE:', err.message);
                });

                this.syncManager.db.run('PRAGMA optimize', (err) => {
                    if (err) console.warn('⚠️ Erreur PRAGMA optimize:', err.message);
                    else console.log('✅ Optimisation terminée');
                    resolve();
                });
            });
        });
    }

    async nettoyerAncienLogs() {
        const cutoff = Date.now() - (7 * 24 * 60 * 60 * 1000); // 7 jours

        return new Promise((resolve, reject) => {
            this.syncManager.db.run(`
                DELETE FROM sync_log
                WHERE synced = 1 AND timestamp < ?
            `, [cutoff], function(err) {
                if (err) reject(err);
                else {
                    console.log(`🧹 ${this.changes} anciens logs nettoyés`);
                    resolve();
                }
            });
        });
    }

    async pause(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

module.exports = IncrementalSync;
```

### 2. Compression des données

**compression-utils.js**
```javascript
const zlib = require('zlib');

class CompressionUtils {
    static async compresserDonnees(data) {
        const json = JSON.stringify(data);
        return new Promise((resolve, reject) => {
            zlib.gzip(json, (err, compressed) => {
                if (err) reject(err);
                else resolve(compressed.toString('base64'));
            });
        });
    }

    static async decompresserDonnees(compressedData) {
        const buffer = Buffer.from(compressedData, 'base64');
        return new Promise((resolve, reject) => {
            zlib.gunzip(buffer, (err, decompressed) => {
                if (err) reject(err);
                else {
                    try {
                        const data = JSON.parse(decompressed.toString());
                        resolve(data);
                    } catch (parseErr) {
                        reject(parseErr);
                    }
                }
            });
        });
    }

    static calculerTauxCompression(original, compresse) {
        const originalSize = Buffer.from(JSON.stringify(original)).length;
        const compressedSize = Buffer.from(compresse, 'base64').length;
        const ratio = ((originalSize - compressedSize) / originalSize * 100).toFixed(1);

        return {
            originalSize,
            compressedSize,
            ratio: `${ratio}%`,
            savings: originalSize - compressedSize
        };
    }
}

// Exemple d'utilisation dans la synchronisation
class CompressedSyncClient extends SyncClient {
    async envoyerChangementsLocaux() {
        const changements = await this.syncManager.getChangementsNonSynchro();

        if (changements.length === 0) {
            console.log('📤 Aucun changement local à envoyer');
            return;
        }

        console.log(`📤 Compression de ${changements.length} changements...`);

        try {
            // Compresser les données
            const compressedData = await CompressionUtils.compresserDonnees(changements);
            const stats = CompressionUtils.calculerTauxCompression(changements, compressedData);

            console.log(`💾 Compression: ${stats.originalSize}B → ${stats.compressedSize}B (${stats.ratio})`);

            const response = await axios.post(`${this.serverUrl}/sync/push-compressed`, {
                data: compressedData,
                count: changements.length
            });

            if (response.data.success) {
                const logIds = changements.map(c => c.id);
                await this.syncManager.marquerCommeSynchro(logIds);
                console.log('✅ Changements compressés envoyés avec succès');
                console.log(`📊 Économie de bande passante: ${stats.savings} bytes`);
            } else {
                throw new Error('Échec de l\'envoi des changements compressés');
            }

        } catch (error) {
            if (error.code === 'ECONNREFUSED' || error.code === 'ENOTFOUND') {
                this.isOnline = false;
                console.log('🔌 Mode hors ligne activé');
            }
            throw error;
        }
    }

    async recupererChangementsServeur() {
        const derniereSync = await this.syncManager.getDerniereSync();

        try {
            console.log('📥 Récupération des changements compressés du serveur...');

            const response = await axios.get(`${this.serverUrl}/sync/pull-compressed`, {
                params: { since: derniereSync }
            });

            if (!response.data.data) {
                console.log('📥 Aucun changement serveur à appliquer');
                return;
            }

            // Décompresser les données
            const changements = await CompressionUtils.decompresserDonnees(response.data.data);
            const stats = CompressionUtils.calculerTauxCompression(changements, response.data.data);

            console.log(`📥 Décompression: ${stats.compressedSize}B → ${stats.originalSize}B`);
            console.log(`📥 Application de ${changements.length} changements du serveur...`);

            const results = await this.syncManager.appliquerChangements(changements);

            // Analyser les résultats
            const success = results.filter(r => r.success).length;
            const conflicts = results.filter(r => !r.success).length;

            console.log(`✅ ${success} changements appliqués`);
            if (conflicts > 0) {
                console.log(`⚠️ ${conflicts} conflits détectés`);
            }

            this.isOnline = true;

        } catch (error) {
            if (error.code === 'ECONNREFUSED' || error.code === 'ENOTFOUND') {
                this.isOnline = false;
                console.log('🔌 Mode hors ligne activé');
            }
            throw error;
        }
    }
}

module.exports = { CompressionUtils, CompressedSyncClient };
```

## Synchronisation en temps réel avec WebSockets

### 1. Serveur WebSocket pour sync temps réel

**realtime-sync-server.js**
```javascript
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const SyncManager = require('./sync-manager');

class RealtimeSyncServer {
    constructor(port = 3001) {
        this.app = express();
        this.server = http.createServer(this.app);
        this.io = socketIo(this.server, {
            cors: {
                origin: "*",
                methods: ["GET", "POST"]
            }
        });
        this.port = port;

        this.syncManager = new SyncManager('./server-database.db');
        this.connectedClients = new Map();

        this.setupMiddleware();
        this.setupRoutes();
        this.setupWebSocket();
    }

    setupMiddleware() {
        this.app.use(express.json());
    }

    setupRoutes() {
        // Routes HTTP classiques
        this.app.get('/health', (req, res) => {
            res.json({
                status: 'ok',
                timestamp: Date.now(),
                connectedClients: this.connectedClients.size
            });
        });

        // Sync HTTP pour compatibilité
        this.app.post('/sync/push', async (req, res) => {
            try {
                const { changements } = req.body;
                const results = await this.syncManager.appliquerChangements(changements);

                // Diffuser les changements en temps réel aux autres clients
                this.diffuserChangements(changements, req.body.clientId);

                res.json({ success: true, results });
            } catch (error) {
                res.status(500).json({ success: false, error: error.message });
            }
        });
    }

    setupWebSocket() {
        this.io.on('connection', (socket) => {
            console.log(`🔗 Client connecté: ${socket.id}`);

            // Enregistrer le client
            this.connectedClients.set(socket.id, {
                socket: socket,
                lastSeen: Date.now(),
                subscriptions: new Set()
            });

            // Gestion des événements de synchronisation
            socket.on('sync:register', (data) => {
                this.handleClientRegister(socket, data);
            });

            socket.on('sync:push', async (data) => {
                await this.handleSyncPush(socket, data);
            });

            socket.on('sync:subscribe', (data) => {
                this.handleSubscribe(socket, data);
            });

            socket.on('disconnect', () => {
                console.log(`📤 Client déconnecté: ${socket.id}`);
                this.connectedClients.delete(socket.id);
            });

            // Ping pour maintenir la connexion
            socket.on('ping', () => {
                socket.emit('pong');
                if (this.connectedClients.has(socket.id)) {
                    this.connectedClients.get(socket.id).lastSeen = Date.now();
                }
            });
        });

        // Nettoyage périodique des clients inactifs
        setInterval(() => {
            this.nettoyerClientsInactifs();
        }, 60000); // Toutes les minutes
    }

    handleClientRegister(socket, data) {
        const client = this.connectedClients.get(socket.id);
        if (client) {
            client.clientInfo = data;
            console.log(`📋 Client enregistré: ${data.name || socket.id}`);

            // Envoyer l'état actuel
            socket.emit('sync:registered', {
                serverId: 'main-server',
                timestamp: Date.now(),
                status: 'connected'
            });
        }
    }

    async handleSyncPush(socket, data) {
        try {
            const { changements } = data;

            console.log(`📥 Réception de ${changements.length} changements de ${socket.id}`);

            // Appliquer les changements
            const results = await this.syncManager.appliquerChangements(changements);

            // Confirmer au client émetteur
            socket.emit('sync:push:result', {
                success: true,
                results: results,
                timestamp: Date.now()
            });

            // Diffuser aux autres clients en temps réel
            this.diffuserChangements(changements, socket.id);

        } catch (error) {
            console.error('Erreur sync push:', error);
            socket.emit('sync:push:result', {
                success: false,
                error: error.message,
                timestamp: Date.now()
            });
        }
    }

    handleSubscribe(socket, data) {
        const client = this.connectedClients.get(socket.id);
        if (client) {
            const { tables } = data;
            tables.forEach(table => client.subscriptions.add(table));
            console.log(`🔔 ${socket.id} s'abonne à: ${tables.join(', ')}`);
        }
    }

    diffuserChangements(changements, emetteurId) {
        changements.forEach(changement => {
            // Diffuser à tous les clients sauf l'émetteur
            this.connectedClients.forEach((client, clientId) => {
                if (clientId !== emetteurId &&
                    client.subscriptions.has(changement.table_name)) {

                    client.socket.emit('sync:change', {
                        changement: changement,
                        timestamp: Date.now(),
                        source: 'realtime'
                    });
                }
            });
        });
    }

    nettoyerClientsInactifs() {
        const now = Date.now();
        const timeout = 5 * 60 * 1000; // 5 minutes

        this.connectedClients.forEach((client, clientId) => {
            if (now - client.lastSeen > timeout) {
                console.log(`🧹 Nettoyage client inactif: ${clientId}`);
                client.socket.disconnect();
                this.connectedClients.delete(clientId);
            }
        });
    }

    start() {
        this.server.listen(this.port, () => {
            console.log(`🚀 Serveur de synchronisation temps réel démarré sur le port ${this.port}`);
            console.log(`🔗 WebSocket: ws://localhost:${this.port}`);
            console.log(`📡 HTTP: http://localhost:${this.port}`);
        });
    }

    // Méthodes utilitaires pour broadcasting
    broadcastToAll(event, data) {
        this.connectedClients.forEach((client) => {
            client.socket.emit(event, data);
        });
    }

    getConnectedClientsInfo() {
        const clients = [];
        this.connectedClients.forEach((client, clientId) => {
            clients.push({
                id: clientId,
                info: client.clientInfo || {},
                subscriptions: Array.from(client.subscriptions),
                lastSeen: new Date(client.lastSeen)
            });
        });
        return clients;
    }
}

// Démarrer le serveur
if (require.main === module) {
    const server = new RealtimeSyncServer();
    server.start();
}

module.exports = RealtimeSyncServer;
```

### 2. Client WebSocket temps réel

**realtime-sync-client.js**
```javascript
const io = require('socket.io-client');
const SyncManager = require('./sync-manager');

class RealtimeSyncClient {
    constructor(syncManager, serverUrl, clientName) {
        this.syncManager = syncManager;
        this.serverUrl = serverUrl;
        this.clientName = clientName;
        this.socket = null;
        this.isConnected = false;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
        this.subscriptions = new Set(['personnes']); // Tables à surveiller

        this.eventHandlers = new Map();
    }

    async connect() {
        console.log(`🔗 Connexion au serveur de sync temps réel: ${this.serverUrl}`);

        this.socket = io(this.serverUrl, {
            autoConnect: true,
            reconnection: true,
            reconnectionAttempts: this.maxReconnectAttempts,
            reconnectionDelay: 1000
        });

        this.setupEventHandlers();

        return new Promise((resolve, reject) => {
            this.socket.on('connect', () => {
                console.log('✅ Connecté au serveur temps réel');
                this.isConnected = true;
                this.reconnectAttempts = 0;
                this.registerClient();
                resolve();
            });

            this.socket.on('connect_error', (error) => {
                console.error('❌ Erreur de connexion:', error.message);
                this.isConnected = false;
                reject(error);
            });
        });
    }

    setupEventHandlers() {
        // Gestion de la connexion
        this.socket.on('disconnect', (reason) => {
            console.log('📤 Déconnecté du serveur temps réel:', reason);
            this.isConnected = false;
        });

        this.socket.on('reconnect', (attemptNumber) => {
            console.log(`🔄 Reconnecté après ${attemptNumber} tentatives`);
            this.isConnected = true;
            this.registerClient();
        });

        this.socket.on('reconnect_error', (error) => {
            this.reconnectAttempts++;
            console.error(`❌ Échec de reconnexion ${this.reconnectAttempts}/${this.maxReconnectAttempts}`);
        });

        // Gestion de l'enregistrement
        this.socket.on('sync:registered', (data) => {
            console.log('📋 Enregistré auprès du serveur:', data);
            this.subscribeToTables();
        });

        // Gestion des changements en temps réel
        this.socket.on('sync:change', async (data) => {
            await this.handleRealtimeChange(data);
        });

        // Résultats de synchronisation
        this.socket.on('sync:push:result', (data) => {
            if (data.success) {
                console.log('✅ Synchronisation confirmée par le serveur');
            } else {
                console.error('❌ Erreur sync serveur:', data.error);
            }
        });

        // Ping/Pong pour maintenir la connexion
        this.socket.on('pong', () => {
            // Connexion maintenue
        });

        // Démarrer le ping
        setInterval(() => {
            if (this.isConnected) {
                this.socket.emit('ping');
            }
        }, 30000); // Toutes les 30 secondes
    }

    registerClient() {
        this.socket.emit('sync:register', {
            name: this.clientName,
            type: 'sqlite-client',
            version: '1.0.0',
            timestamp: Date.now()
        });
    }

    subscribeToTables() {
        this.socket.emit('sync:subscribe', {
            tables: Array.from(this.subscriptions)
        });
        console.log(`🔔 Abonné aux tables: ${Array.from(this.subscriptions).join(', ')}`);
    }

    async handleRealtimeChange(data) {
        const { changement, timestamp, source } = data;

        console.log(`⚡ Changement temps réel reçu: ${changement.operation} sur ${changement.record_id}`);

        try {
            // Appliquer le changement localement
            await this.syncManager.appliquerChangement(changement);

            // Émettre un événement pour l'interface utilisateur
            this.emitToApp('data:changed', {
                table: changement.table_name,
                recordId: changement.record_id,
                operation: changement.operation,
                data: changement.data
            });

        } catch (error) {
            console.error('❌ Erreur application changement temps réel:', error.message);
        }
    }

    async pushChangements(changements) {
        if (!this.isConnected) {
            throw new Error('Non connecté au serveur temps réel');
        }

        console.log(`📤 Envoi temps réel de ${changements.length} changements`);

        this.socket.emit('sync:push', {
            changements: changements,
            clientId: this.socket.id,
            timestamp: Date.now()
        });
    }

    // Interface pour l'application
    onDataChanged(callback) {
        this.eventHandlers.set('data:changed', callback);
    }

    onConnectionChanged(callback) {
        this.eventHandlers.set('connection:changed', callback);

        // Émettre l'état actuel
        callback(this.isConnected);

        // Écouter les changements de connexion
        this.socket.on('connect', () => callback(true));
        this.socket.on('disconnect', () => callback(false));
    }

    emitToApp(event, data) {
        const handler = this.eventHandlers.get(event);
        if (handler) {
            handler(data);
        }
    }

    disconnect() {
        if (this.socket) {
            console.log('📤 Déconnexion du serveur temps réel');
            this.socket.disconnect();
            this.isConnected = false;
        }
    }

    getStatus() {
        return {
            connected: this.isConnected,
            reconnectAttempts: this.reconnectAttempts,
            subscriptions: Array.from(this.subscriptions),
            socketId: this.socket?.id
        };
    }
}

module.exports = RealtimeSyncClient;
```

## Application avec sync temps réel

### 3. Interface utilisateur en temps réel

**realtime-app.js**
```javascript
const SyncManager = require('./sync-manager');
const RealtimeSyncClient = require('./realtime-sync-client');
const readline = require('readline');

class RealtimeApp {
    constructor() {
        this.syncManager = new SyncManager('./client-realtime.db');
        this.realtimeClient = new RealtimeSyncClient(
            this.syncManager,
            'http://localhost:3001',
            `Client-${process.pid}`
        );

        this.rl = readline.createInterface({
            input: process.stdin,
            output: process.stdout
        });

        this.isRunning = false;
        this.setupRealtimeHandlers();
    }

    setupRealtimeHandlers() {
        // Écouter les changements de données en temps réel
        this.realtimeClient.onDataChanged((changeInfo) => {
            this.handleDataChange(changeInfo);
        });

        // Écouter les changements de connexion
        this.realtimeClient.onConnectionChanged((isConnected) => {
            const status = isConnected ? '🟢 EN LIGNE' : '🔴 HORS LIGNE';
            console.log(`\n📡 Statut connexion: ${status}`);
            if (this.isRunning) {
                this.afficherPrompt();
            }
        });
    }

    handleDataChange(changeInfo) {
        const { table, recordId, operation, data } = changeInfo;

        console.log(`\n⚡ MISE À JOUR TEMPS RÉEL ⚡`);
        console.log(`📋 Table: ${table}`);
        console.log(`🆔 ID: ${recordId}`);
        console.log(`⚙️ Opération: ${operation}`);

        if (data) {
            console.log(`📄 Données:`, data);
        }

        // Émettre un bip sonore pour attirer l'attention
        process.stdout.write('\u0007');

        if (this.isRunning) {
            this.afficherPrompt();
        }
    }

    async start() {
        console.log('🚀 Application de synchronisation temps réel');
        console.log('⚡ Les modifications d\'autres clients apparaîtront automatiquement\n');

        try {
            await this.realtimeClient.connect();
            this.isRunning = true;
            await this.afficherMenu();
        } catch (error) {
            console.error('❌ Impossible de se connecter au serveur temps réel');
            console.log('📱 Fonctionnement en mode local uniquement');
            this.isRunning = true;
            await this.afficherMenu();
        }
    }

    async afficherMenu() {
        console.log('\n📋 Menu principal (temps réel):');
        console.log('1. Ajouter une personne');
        console.log('2. Lister les personnes');
        console.log('3. Modifier une personne');
        console.log('4. Supprimer une personne');
        console.log('5. Statut de connexion');
        console.log('6. Statistiques temps réel');
        console.log('0. Quitter');

        this.afficherPrompt();
    }

    afficherPrompt() {
        const status = this.realtimeClient.isConnected ? '🟢' : '🔴';
        this.rl.question(`\n${status} Choisissez une option : `, async (choix) => {
            await this.traiterChoix(choix);
        });
    }

    async traiterChoix(choix) {
        try {
            switch (choix) {
                case '1':
                    await this.ajouterPersonneTempsReel();
                    break;
                case '2':
                    await this.listerPersonnes();
                    break;
                case '3':
                    await this.modifierPersonneTempsReel();
                    break;
                case '4':
                    await this.supprimerPersonneTempsReel();
                    break;
                case '5':
                    await this.afficherStatutConnexion();
                    break;
                case '6':
                    await this.afficherStatistiquesTempsReel();
                    break;
                case '0':
                    await this.quitter();
                    return;
                default:
                    console.log('❌ Option invalide');
            }
        } catch (error) {
            console.error('❌ Erreur:', error.message);
        }

        await this.afficherMenu();
    }

    async ajouterPersonneTempsReel() {
        console.log('\n➕ Ajouter une nouvelle personne (temps réel)');

        const nom = await this.poserQuestion('Nom : ');
        const email = await this.poserQuestion('Email : ');
        const ageStr = await this.poserQuestion('Âge : ');
        const age = ageStr ? parseInt(ageStr) : null;

        try {
            // Créer localement
            const personne = await this.syncManager.creerPersonne(nom, email, age);
            console.log('✅ Personne ajoutée localement:', personne.nom);

            // Synchroniser en temps réel si connecté
            if (this.realtimeClient.isConnected) {
                const changements = await this.syncManager.getChangementsNonSynchro();
                await this.realtimeClient.pushChangements(changements);
                await this.syncManager.marquerCommeSynchro(changements.map(c => c.id));
                console.log('⚡ Synchronisé en temps réel avec les autres clients');
            } else {
                console.log('📱 Sera synchronisé lors de la prochaine connexion');
            }

        } catch (error) {
            console.error('❌ Erreur lors de l\'ajout :', error.message);
        }
    }

    async modifierPersonneTempsReel() {
        console.log('\n✏️ Modifier une personne (temps réel)');

        const id = await this.poserQuestion('ID de la personne : ');

        const personne = await this.obtenirPersonneParId(id);
        if (!personne) {
            console.log('❌ Personne non trouvée');
            return;
        }

        console.log(`\nPersonne actuelle : ${personne.nom} (${personne.age} ans) - ${personne.email}`);
        console.log('⚡ ATTENTION: Cette modification sera visible par tous les clients connectés!\n');

        const nom = await this.poserQuestion(`Nouveau nom [${personne.nom}] : `);
        const email = await this.poserQuestion(`Nouvel email [${personne.email}] : `);
        const ageStr = await this.poserQuestion(`Nouvel âge [${personne.age}] : `);

        const donnees = {};
        if (nom) donnees.nom = nom;
        if (email) donnees.email = email;
        if (ageStr) donnees.age = parseInt(ageStr);

        if (Object.keys(donnees).length === 0) {
            console.log('📝 Aucune modification effectuée');
            return;
        }

        try {
            await this.syncManager.mettreAJourPersonne(id, donnees);
            console.log('✅ Personne modifiée localement');

            // Synchroniser en temps réel
            if (this.realtimeClient.isConnected) {
                const changements = await this.syncManager.getChangementsNonSynchro();
                await this.realtimeClient.pushChangements(changements);
                await this.syncManager.marquerCommeSynchro(changements.map(c => c.id));
                console.log('⚡ Modification propagée en temps réel');
            }

        } catch (error) {
            console.error('❌ Erreur lors de la modification :', error.message);
        }
    }

    async supprimerPersonneTempsReel() {
        console.log('\n🗑️ Supprimer une personne (temps réel)');

        const id = await this.poserQuestion('ID de la personne : ');

        const personne = await this.obtenirPersonneParId(id);
        if (!personne) {
            console.log('❌ Personne non trouvée');
            return;
        }

        console.log(`\nPersonne à supprimer : ${personne.nom} (${personne.age} ans)`);
        console.log('⚡ ATTENTION: Cette suppression sera visible par tous les clients!');
        const confirmation = await this.poserQuestion('Confirmer la suppression ? (oui/non) : ');

        if (confirmation.toLowerCase() === 'oui') {
            try {
                await this.syncManager.supprimerPersonne(id);
                console.log('✅ Personne supprimée localement');

                // Synchroniser en temps réel
                if (this.realtimeClient.isConnected) {
                    const changements = await this.syncManager.getChangementsNonSynchro();
                    await this.realtimeClient.pushChangements(changements);
                    await this.syncManager.marquerCommeSynchro(changements.map(c => c.id));
                    console.log('⚡ Suppression propagée en temps réel');
                }

            } catch (error) {
                console.error('❌ Erreur lors de la suppression :', error.message);
            }
        } else {
            console.log('❌ Suppression annulée');
        }
    }

    async listerPersonnes() {
        console.log('\n📋 Liste des personnes (données temps réel)');

        return new Promise((resolve) => {
            this.syncManager.db.all(`
                SELECT * FROM personnes
                WHERE is_deleted = 0
                ORDER BY updated_at DESC, nom
            `, (err, rows) => {
                if (err) {
                    console.error('❌ Erreur :', err.message);
                } else if (rows.length === 0) {
                    console.log('📭 Aucune personne trouvée');
                } else {
                    console.log(`\n👥 ${rows.length} personne(s) :`);
                    rows.forEach((p, index) => {
                        const recentlyUpdated = (Date.now() - p.updated_at) < 60000 ? '⚡' : '';
                        const syncStatus = p.last_sync > 0 ? '🔄' : '📤';

                        console.log(`${index + 1}. ${p.nom} (${p.age} ans) - ${p.email} ${syncStatus}${recentlyUpdated}`);
                        console.log(`   ID: ${p.id} | Version: ${p.version} | Modifié: ${new Date(p.updated_at).toLocaleString()}`);
                    });

                    console.log('\n⚡ = Modifié récemment | 🔄 = Synchronisé | 📤 = En attente');
                }
                resolve();
            });
        });
    }

    async afficherStatutConnexion() {
        console.log('\n📡 Statut de connexion temps réel');

        const status = this.realtimeClient.getStatus();

        console.log(`🌐 Connexion : ${status.connected ? '✅ Connecté' : '❌ Déconnecté'}`);
        console.log(`🆔 Socket ID : ${status.socketId || 'N/A'}`);
        console.log(`🔄 Tentatives de reconnexion : ${status.reconnectAttempts}`);
        console.log(`🔔 Abonnements : ${status.subscriptions.join(', ')}`);

        if (status.connected) {
            console.log('⚡ Toutes les modifications sont synchronisées en temps réel');
        } else {
            console.log('📱 Mode hors ligne - les modifications sont stockées localement');
        }
    }

    async afficherStatistiquesTempsReel() {
        console.log('\n📊 Statistiques temps réel');

        // Statistiques de base de données
        const stats = await this.obtenirStatistiquesDB();

        // Statistiques de synchronisation
        const syncStats = await this.obtenirStatistiquesSync();

        console.log('📋 Base de données:');
        console.log(`  👥 Personnes actives: ${stats.totalPersonnes}`);
        console.log(`  📅 Créées aujourd'hui: ${stats.creesAujourdhui}`);
        console.log(`  ✏️ Modifiées aujourd'hui: ${stats.modifieesAujourdhui}`);

        console.log('\n🔄 Synchronisation:');
        console.log(`  📤 Changements en attente: ${syncStats.changementsEnAttente}`);
        console.log(`  ⚡ Dernière sync temps réel: ${syncStats.derniereSync}`);
        console.log(`  📊 Total synchronisé: ${syncStats.totalSynchronise}`);

        if (this.realtimeClient.isConnected) {
            console.log('\n🌐 Temps réel:');
            console.log(`  🔗 Connecté depuis: ${this.calculerDureeConnexion()}`);
            console.log(`  📡 Ping moyen: ${await this.mesurerLatence()}ms`);
        }
    }

    async obtenirStatistiquesDB() {
        return new Promise((resolve, reject) => {
            const aujourd = new Date();
            aujourd.setHours(0, 0, 0, 0);
            const timestampAujourdhui = aujourd.getTime();

            this.syncManager.db.get(`
                SELECT
                    COUNT(*) as total_personnes,
                    COUNT(CASE WHEN created_at >= ? THEN 1 END) as crees_aujourdhui,
                    COUNT(CASE WHEN updated_at >= ? AND created_at < ? THEN 1 END) as modifiees_aujourdhui
                FROM personnes
                WHERE is_deleted = 0
            `, [timestampAujourdhui, timestampAujourdhui, timestampAujourdhui], (err, row) => {
                if (err) reject(err);
                else resolve({
                    totalPersonnes: row.total_personnes,
                    creesAujourdhui: row.crees_aujourdhui,
                    modifieesAujourdhui: row.modifiees_aujourdhui
                });
            });
        });
    }

    async obtenirStatistiquesSync() {
        return new Promise((resolve, reject) => {
            this.syncManager.db.all(`
                SELECT
                    COUNT(CASE WHEN synced = 0 THEN 1 END) as en_attente,
                    COUNT(CASE WHEN synced = 1 THEN 1 END) as synchronise,
                    MAX(timestamp) as derniere_sync
                FROM sync_log
            `, (err, rows) => {
                if (err) reject(err);
                else {
                    const row = rows[0];
                    resolve({
                        changementsEnAttente: row.en_attente,
                        totalSynchronise: row.synchronise,
                        derniereSync: row.derniere_sync ?
                            new Date(row.derniere_sync).toLocaleString() : 'Jamais'
                    });
                }
            });
        });
    }

    calculerDureeConnexion() {
        // Simulation - dans une vraie app, on stockerait l'heure de connexion
        return '2 minutes';
    }

    async mesurerLatence() {
        return new Promise((resolve) => {
            const start = Date.now();
            this.realtimeClient.socket.emit('ping');

            this.realtimeClient.socket.once('pong', () => {
                const latence = Date.now() - start;
                resolve(latence);
            });

            // Timeout après 5 secondes
            setTimeout(() => resolve(999), 5000);
        });
    }

    async obtenirPersonneParId(id) {
        return new Promise((resolve, reject) => {
            this.syncManager.db.get(`
                SELECT * FROM personnes
                WHERE id = ? AND is_deleted = 0
            `, [id], (err, row) => {
                if (err) reject(err);
                else resolve(row);
            });
        });
    }

    async poserQuestion(question) {
        return new Promise((resolve) => {
            this.rl.question(question, (reponse) => {
                resolve(reponse.trim());
            });
        });
    }

    async quitter() {
        console.log('\n👋 Arrêt de l\'application temps réel...');

        this.isRunning = false;

        // Déconnecter du serveur temps réel
        if (this.realtimeClient.isConnected) {
            this.realtimeClient.disconnect();
        }

        this.rl.close();
        console.log('✅ Application fermée');
        process.exit(0);
    }
}

// Démarrer l'application
if (require.main === module) {
    const app = new RealtimeApp();
    app.start().catch(console.error);
}

module.exports = RealtimeApp;
```

## Déploiement en production

### 1. Configuration Docker pour la synchronisation

**docker-compose.yml**
```yaml
version: '3.8'

services:
  # Serveur de synchronisation principal
  sync-server:
    build:
      context: .
      dockerfile: Dockerfile.sync-server
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
      - DATABASE_PATH=/app/data/server.db
      - REDIS_URL=redis://redis:6379
    volumes:
      - sync_data:/app/data
      - ./logs:/app/logs
    depends_on:
      - redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis pour la mise en cache et la coordination
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped
    command: redis-server --appendonly yes

  # Nginx pour le load balancing
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx-sync.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - sync-server
    restart: unless-stopped

  # Monitoring avec Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

volumes:
  sync_data:
  redis_data:
  grafana_data:
```

**Dockerfile.sync-server**
```dockerfile
FROM node:18-alpine

# Installer sqlite3 et autres dépendances
RUN apk add --no-cache sqlite curl

WORKDIR /app

# Copier les fichiers de dépendances
COPY package*.json ./

# Installer les dépendances
RUN npm ci --only=production

# Copier le code source
COPY . .

# Créer les dossiers nécessaires
RUN mkdir -p /app/data /app/logs

# Créer un utilisateur non-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    chown -R nodejs:nodejs /app

USER nodejs

# Exposer le port
EXPOSE 3001

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3001/health || exit 1

# Commande de démarrage
CMD ["node", "realtime-sync-server.js"]
```

### 2. Monitoring et observabilité

**monitoring/sync-metrics.js**
```javascript
const client = require('prom-client');

class SyncMetrics {
    constructor() {
        // Créer un registre pour les métriques
        this.register = new client.Registry();

        // Métriques par défaut (CPU, mémoire, etc.)
        client.collectDefaultMetrics({ register: this.register });

        // Métriques spécifiques à la synchronisation
        this.connectedClients = new client.Gauge({
            name: 'sync_connected_clients_total',
            help: 'Nombre de clients connectés',
            registers: [this.register]
        });

        this.syncOperations = new client.Counter({
            name: 'sync_operations_total',
            help: 'Nombre total d\'opérations de synchronisation',
            labelNames: ['operation', 'status'],
            registers: [this.register]
        });

        this.syncLatency = new client.Histogram({
            name: 'sync_operation_duration_seconds',
            help: 'Durée des opérations de synchronisation',
            labelNames: ['operation'],
            buckets: [0.001, 0.01, 0.1, 1, 5, 10],
            registers: [this.register]
        });

        this.dataChanges = new client.Counter({
            name: 'sync_data_changes_total',
            help: 'Nombre de changements de données',
            labelNames: ['table', 'operation'],
            registers: [this.register]
        });

        this.conflictsResolved = new client.Counter({
            name: 'sync_conflicts_resolved_total',
            help: 'Nombre de conflits résolus',
            labelNames: ['resolution_strategy'],
            registers: [this.register]
        });
    }

    // Mettre à jour le nombre de clients connectés
    setConnectedClients(count) {
        this.connectedClients.set(count);
    }

    // Enregistrer une opération de synchronisation
    recordSyncOperation(operation, status, duration) {
        this.syncOperations.inc({ operation, status });
        if (duration !== undefined) {
            this.syncLatency.observe({ operation }, duration);
        }
    }

    // Enregistrer un changement de données
    recordDataChange(table, operation) {
        this.dataChanges.inc({ table, operation });
    }

    // Enregistrer la résolution d'un conflit
    recordConflictResolution(strategy) {
        this.conflictsResolved.inc({ resolution_strategy: strategy });
    }

    // Obtenir toutes les métriques
    async getMetrics() {
        return this.register.metrics();
    }
}

module.exports = SyncMetrics;
```

**Intégration dans le serveur :**
```javascript
// Dans realtime-sync-server.js
const SyncMetrics = require('./monitoring/sync-metrics');

class RealtimeSyncServer {
    constructor(port = 3001) {
        // ... code existant ...
        this.metrics = new SyncMetrics();
        this.setupMetricsRoutes();
    }

    setupMetricsRoutes() {
        // Endpoint pour Prometheus
        this.app.get('/metrics', async (req, res) => {
            res.set('Content-Type', this.metrics.register.contentType);
            const metrics = await this.metrics.getMetrics();
            res.end(metrics);
        });
    }

    setupWebSocket() {
        this.io.on('connection', (socket) => {
            // ... code existant ...

            // Mettre à jour les métriques
            this.metrics.setConnectedClients(this.connectedClients.size);
        });

        // ... reste du code ...
    }

    async handleSyncPush(socket, data) {
        const startTime = Date.now();

        try {
            const { changements } = data;

            // Enregistrer les métriques
            changements.forEach(changement => {
                this.metrics.recordDataChange(changement.table_name, changement.operation);
            });

            const results = await this.syncManager.appliquerChangements(changements);

            // Mesurer la durée
            const duration = (Date.now() - startTime) / 1000;
            this.metrics.recordSyncOperation('push', 'success', duration);

            // ... reste du code ...

        } catch (error) {
            const duration = (Date.now() - startTime) / 1000;
            this.metrics.recordSyncOperation('push', 'error', duration);
            // ... gestion d'erreur ...
        }
    }
}
```

### 3. Scripts de déploiement et maintenance

**scripts/deploy-sync.sh**
```bash
#!/bin/bash

set -e

echo "🚀 Déploiement du système de synchronisation..."

# Variables
ENVIRONMENT=${1:-staging}
VERSION=$(git rev-parse --short HEAD)
IMAGE_NAME="sync-server:$VERSION"

echo "📋 Environnement: $ENVIRONMENT"
echo "📋 Version: $VERSION"

# Construire l'image Docker
echo "🔨 Construction de l'image Docker..."
docker build -t $IMAGE_NAME -f Dockerfile.sync-server .

# Tagger pour le registry
docker tag $IMAGE_NAME "registry.example.com/$IMAGE_NAME"

# Pousser vers le registry
echo "📤 Push vers le registry..."
docker push "registry.example.com/$IMAGE_NAME"

# Déployer avec docker-compose
echo "🚢 Déploiement..."
export SYNC_SERVER_IMAGE="registry.example.com/$IMAGE_NAME"
docker-compose -f docker-compose.$ENVIRONMENT.yml up -d

# Vérifier le déploiement
echo "✅ Vérification du déploiement..."
sleep 10

# Test de santé
for i in {1..5}; do
    if curl -f http://localhost:3001/health; then
        echo "✅ Serveur de synchronisation opérationnel"
        break
    else
        echo "⏳ Tentative $i/5..."
        sleep 5
    fi
done

# Afficher les logs
echo "📋 Logs récents:"
docker-compose -f docker-compose.$ENVIRONMENT.yml logs --tail=50 sync-server

echo "🎉 Déploiement terminé!"
```

**scripts/backup-sync-data.sh**
```bash
#!/bin/bash

# Script de sauvegarde pour les données de synchronisation

BACKUP_DIR="/backups/sync"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

echo "💾 Sauvegarde des données de synchronisation..."

# Créer le dossier de sauvegarde
mkdir -p $BACKUP_DIR

# Sauvegarder la base de données principale
echo "📋 Sauvegarde de la base de données..."
docker exec sync-server_sync-server_1 sqlite3 /app/data/server.db ".backup /tmp/backup_$DATE.db"
docker cp sync-server_sync-server_1:/tmp/backup_$DATE.db $BACKUP_DIR/

# Sauvegarder les logs de synchronisation
echo "📄 Sauvegarde des logs..."
tar -czf $BACKUP_DIR/logs_$DATE.tar.gz -C . logs/

# Sauvegarder la configuration Redis
echo "🔴 Sauvegarde Redis..."
docker exec sync-server_redis_1 redis-cli BGSAVE
sleep 5
docker cp sync-server_redis_1:/data/dump.rdb $BACKUP_DIR/redis_$DATE.rdb

# Compresser toutes les sauvegardes
echo "🗜️ Compression..."
tar -czf $BACKUP_DIR/sync_complete_$DATE.tar.gz -C $BACKUP_DIR \
    backup_$DATE.db logs_$DATE.tar.gz redis_$DATE.rdb

# Nettoyer les fichiers temporaires
rm $BACKUP_DIR/backup_$DATE.db $BACKUP_DIR/logs_$DATE.tar.gz $BACKUP_DIR/redis_$DATE.rdb

# Nettoyer les anciennes sauvegardes
echo "🧹 Nettoyage des anciennes sauvegardes..."
find $BACKUP_DIR -name "sync_complete_*.tar.gz" -mtime +$RETENTION_DAYS -delete

# Vérifier l'intégrité
echo "🔍 Vérification de l'intégrité..."
if tar -tzf $BACKUP_DIR/sync_complete_$DATE.tar.gz > /dev/null; then
    echo "✅ Sauvegarde créée avec succès: sync_complete_$DATE.tar.gz"

    # Taille de la sauvegarde
    SIZE=$(du -h $BACKUP_DIR/sync_complete_$DATE.tar.gz | cut -f1)
    echo "📊 Taille: $SIZE"
else
    echo "❌ Erreur: Sauvegarde corrompue"
    exit 1
fi

echo "💾 Sauvegarde terminée!"
```

## Bonnes pratiques et conseils

### 1. Checklist de production

**✅ Sécurité**
- [ ] Authentification des clients de synchronisation
- [ ] Chiffrement des données en transit (HTTPS/WSS)
- [ ] Validation stricte des données reçues
- [ ] Rate limiting pour éviter les abus
- [ ] Logs d'audit pour tracer les modifications

**✅ Performance**
- [ ] Compression des données synchronisées
- [ ] Synchronisation par lots (batching)
- [ ] Index optimisés sur les tables de synchronisation
- [ ] Nettoyage périodique des anciens logs
- [ ] Mise en cache des requêtes fréquentes

**✅ Fiabilité**
- [ ] Gestion des reconnexions automatiques
- [ ] Résolution de conflits robuste
- [ ] Sauvegarde automatique des données
- [ ] Health checks complets
- [ ] Monitoring et alertes

**✅ Scalabilité**
- [ ] Load balancing pour plusieurs serveurs
- [ ] Partitionnement des données si nécessaire
- [ ] Optimisation des requêtes SQL
- [ ] Configuration de la base pour la charge
- [ ] Tests de charge réguliers

### 2. Patterns recommandés

**Pattern Event Sourcing pour la sync**
```javascript
// Au lieu de synchroniser l'état, synchroniser les événements
const events = [
    { type: 'PersonneCreated', data: { nom: 'Alice' }, timestamp: 123456 },
    { type: 'PersonneUpdated', data: { id: '1', nom: 'Alice Martin' }, timestamp: 123457 }
];
```

**Pattern CQRS pour optimiser**
```javascript
// Séparer les modèles de lecture et d'écriture
class SyncWriteModel {
    // Optimisé pour les modifications
}

class SyncReadModel {
    // Optimisé pour les requêtes
}
```

**Pattern Saga pour les transactions distribuées**
```javascript
// Gérer les transactions complexes sur plusieurs clients
class SyncSaga {
    async execute(steps) {
        const compensations = [];
        try {
            for (const step of steps) {
                await step.execute();
                compensations.push(step.compensate);
            }
        } catch (error) {
            // Annuler toutes les étapes précédentes
            for (const compensate of compensations.reverse()) {
                await compensate();
            }
            throw error;
        }
    }
}
```

### 3. Résolution de problèmes courants

**Problème : Conflits de synchronisation fréquents**
```javascript
// Solution : Augmenter la granularité des timestamps
const timestamp = Date.now() * 1000 + performance.now() % 1000;

// Ou utiliser des UUIDs ordonnés
const { v1: uuidv1 } = require('uuid');
const orderedId = uuidv1();
```

**Problème : Synchronisation lente**
```javascript
// Solution : Synchronisation incrémentale avec checksums
class IncrementalSync {
    async getChangedRecords(lastChecksum) {
        return db.query(`
            SELECT * FROM personnes
            WHERE checksum != ? OR updated_at > ?
        `, [lastChecksum, lastSyncTime]);
    }
}
```

**Problème : Déconnexions fréquentes**
```javascript
// Solution : Heartbeat et reconnexion intelligente
class RobustClient {
    setupHeartbeat() {
        setInterval(() => {
            if (this.socket.connected) {
                this.socket.emit('heartbeat');
            } else {
                this.reconnect();
            }
        }, 30000);
    }
}
```

## Conclusion

### Ce que vous avez appris

Dans cette section sur la synchronisation et réplication SQLite, vous avez découvert :

1. **Les concepts fondamentaux** de la synchronisation de données
2. **L'implémentation pratique** d'un système de sync bidirectionnelle
3. **La gestion des conflits** avec plusieurs stratégies de résolution
4. **La synchronisation temps réel** avec WebSockets
5. **L'optimisation des performances** avec compression et batching
6. **Le déploiement en production** avec Docker et monitoring

### Cas d'usage réels

**Applications mobiles** :
- Synchronisation hors ligne avec le cloud
- Collaboration en temps réel
- Sauvegarde automatique des données

**Applications web collaboratives** :
- Édition simultanée de documents
- Chat et messaging
- Tableaux de bord temps réel

**Systèmes distribués** :
- Réplication multi-sites
- Synchronisation entre data centers
- Systèmes de cache distribué

### Prochaines étapes

**Pour approfondir** :
1. Étudier les algorithmes de consensus (Raft, PBFT)
2. Implémenter des CRDTs (Conflict-free Replicated Data Types)
3. Explorer les bases de données distribuées (CouchDB, MongoDB)
4. Apprendre les patterns Event Sourcing et CQRS

**Projets à réaliser** :
1. Application de notes collaboratives
2. Chat temps réel avec historique synchronisé
3. Système de gestion de stock multi-magasins
4. Application de suivi de projet en équipe

### Ressources pour aller plus loin

- **CouchDB Replication** : Inspiration pour les algorithmes de sync
- **Operational Transform** : Pour la collaboration temps réel
- **WebRTC** : Pour la synchronisation peer-to-peer
- **Apache Kafka** : Pour les systèmes de streaming de données

La synchronisation de données est un domaine complexe mais passionnant. Avec SQLite comme base, vous avez maintenant les outils pour créer des applications robustes qui fonctionnent aussi bien hors ligne qu'en ligne, avec une synchronisation transparente pour vos utilisateurs.

**Bonne synchronisation ! ⚡🔄**

⏭️
