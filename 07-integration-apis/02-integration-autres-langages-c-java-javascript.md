🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.2 : Intégration avec d'autres langages (C, Java, JavaScript)

## Introduction

Après avoir découvert SQLite avec Python, vous vous demandez peut-être comment l'utiliser avec d'autres langages de programmation. La beauté de SQLite réside dans sa compatibilité universelle : que vous programmiez en C, Java, JavaScript ou d'autres langages, SQLite s'adapte parfaitement à votre environnement de développement.

Cette section vous guidera dans l'intégration de SQLite avec trois langages populaires, en expliquant les spécificités de chaque approche de manière accessible aux débutants.

## Vue d'ensemble des approches par langage

| Langage | Type d'intégration | Complexité | Cas d'usage typique |
|---------|-------------------|------------|---------------------|
| **C** | API native directe | Moyenne | Applications système, performance maximale |
| **Java** | Driver JDBC | Facile | Applications d'entreprise, Android |
| **JavaScript** | Bibliothèques externes | Facile | Applications web, Node.js |

## SQLite avec le langage C

### Pourquoi utiliser SQLite en C ?

Le langage C offre l'accès le plus direct à SQLite car SQLite est lui-même écrit en C. Cette approche vous donne :
- **Performance maximale** : Aucune couche d'abstraction
- **Contrôle total** : Accès à toutes les fonctionnalités SQLite
- **Intégration système** : Parfait pour les applications système

### Installation et configuration

#### Sur Linux/Ubuntu :
```bash
sudo apt-get install libsqlite3-dev
```

#### Sur macOS :
```bash
# SQLite est généralement déjà installé
# Sinon via Homebrew :
brew install sqlite3
```

#### Sur Windows :
Téléchargez les fichiers de développement depuis [sqlite.org](https://sqlite.org/download.html)

### Premier programme en C

Créez un fichier `exemple_sqlite.c` :

```c
#include <stdio.h>
#include <sqlite3.h>
#include <stdlib.h>

int main() {
    sqlite3 *db;
    int resultat;

    // Ouvrir ou créer une base de données
    resultat = sqlite3_open("exemple.db", &db);

    if (resultat != SQLITE_OK) {
        printf("Erreur lors de l'ouverture : %s\n", sqlite3_errmsg(db));
        return 1;
    }

    printf("Base de données ouverte avec succès !\n");

    // Fermer la base de données
    sqlite3_close(db);
    return 0;
}
```

**Compilation** :
```bash
gcc -o exemple_sqlite exemple_sqlite.c -lsqlite3
```

### Créer une table et insérer des données

```c
#include <stdio.h>
#include <sqlite3.h>
#include <stdlib.h>
#include <string.h>

// Fonction de callback pour traiter les résultats
static int callback(void *data, int argc, char **argv, char **azColName) {
    printf("Enregistrement trouvé :\n");
    for (int i = 0; i < argc; i++) {
        printf("  %s = %s\n", azColName[i], argv[i] ? argv[i] : "NULL");
    }
    printf("\n");
    return 0;
}

int main() {
    sqlite3 *db;
    char *errMsg = 0;
    int resultat;

    // Ouvrir la base de données
    resultat = sqlite3_open("personnes.db", &db);
    if (resultat != SQLITE_OK) {
        printf("Erreur : %s\n", sqlite3_errmsg(db));
        return 1;
    }

    // Créer une table
    const char *sql_create =
        "CREATE TABLE IF NOT EXISTS personnes ("
        "id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "nom TEXT NOT NULL,"
        "age INTEGER);";

    resultat = sqlite3_exec(db, sql_create, 0, 0, &errMsg);
    if (resultat != SQLITE_OK) {
        printf("Erreur SQL : %s\n", errMsg);
        sqlite3_free(errMsg);
        return 1;
    }
    printf("Table créée avec succès !\n");

    // Insérer des données
    const char *sql_insert =
        "INSERT INTO personnes (nom, age) VALUES "
        "('Alice Martin', 25),"
        "('Bob Dupont', 30),"
        "('Claire Rousseau', 28);";

    resultat = sqlite3_exec(db, sql_insert, 0, 0, &errMsg);
    if (resultat != SQLITE_OK) {
        printf("Erreur insertion : %s\n", errMsg);
        sqlite3_free(errMsg);
    } else {
        printf("Données insérées avec succès !\n");
    }

    // Lire les données
    const char *sql_select = "SELECT * FROM personnes;";
    printf("\nContenu de la table :\n");
    resultat = sqlite3_exec(db, sql_select, callback, 0, &errMsg);

    if (resultat != SQLITE_OK) {
        printf("Erreur lecture : %s\n", errMsg);
        sqlite3_free(errMsg);
    }

    // Fermer la base de données
    sqlite3_close(db);
    return 0;
}
```

### Requêtes préparées en C (plus sécurisé)

```c
#include <stdio.h>
#include <sqlite3.h>

int ajouter_personne(sqlite3 *db, const char *nom, int age) {
    sqlite3_stmt *stmt;
    int resultat;

    // Préparer la requête
    const char *sql = "INSERT INTO personnes (nom, age) VALUES (?, ?);";
    resultat = sqlite3_prepare_v2(db, sql, -1, &stmt, NULL);

    if (resultat != SQLITE_OK) {
        printf("Erreur préparation : %s\n", sqlite3_errmsg(db));
        return resultat;
    }

    // Lier les paramètres
    sqlite3_bind_text(stmt, 1, nom, -1, SQLITE_STATIC);
    sqlite3_bind_int(stmt, 2, age);

    // Exécuter la requête
    resultat = sqlite3_step(stmt);

    if (resultat == SQLITE_DONE) {
        printf("Personne ajoutée : %s (%d ans)\n", nom, age);
    } else {
        printf("Erreur exécution : %s\n", sqlite3_errmsg(db));
    }

    // Libérer les ressources
    sqlite3_finalize(stmt);
    return resultat;
}

int main() {
    sqlite3 *db;

    if (sqlite3_open("personnes.db", &db) == SQLITE_OK) {
        ajouter_personne(db, "David Moreau", 35);
        ajouter_personne(db, "Emma Leroy", 27);
        sqlite3_close(db);
    }

    return 0;
}
```

## SQLite avec Java

### Pourquoi Java et SQLite ?

Java offre une approche orientée objet plus familière pour beaucoup de développeurs :
- **JDBC standard** : Interface familière pour les développeurs Java
- **Portabilité** : Fonctionne sur toutes les plateformes Java
- **Écosystème riche** : Nombreuses bibliothèques complémentaires
- **Android** : SQLite est la base de données par défaut sur Android

### Configuration du projet

#### Avec Maven (pom.xml)

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.exemple</groupId>
    <artifactId>sqlite-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.xerial</groupId>
            <artifactId>sqlite-jdbc</artifactId>
            <version>3.43.0.0</version>
        </dependency>
    </dependencies>
</project>
```

#### Avec Gradle (build.gradle)

```gradle
dependencies {
    implementation 'org.xerial:sqlite-jdbc:3.43.0.0'
}
```

### Premier programme Java

```java
import java.sql.*;

public class SQLiteExample {
    public static void main(String[] args) {
        // URL de connexion SQLite
        String url = "jdbc:sqlite:exemple.db";

        try (Connection conn = DriverManager.getConnection(url)) {
            if (conn != null) {
                System.out.println("Connexion réussie à SQLite !");

                // Afficher la version de SQLite
                DatabaseMetaData meta = conn.getMetaData();
                System.out.println("Driver: " + meta.getDriverName());
                System.out.println("Version: " + meta.getDriverVersion());
            }
        } catch (SQLException e) {
            System.err.println("Erreur : " + e.getMessage());
        }
    }
}
```

### Classe complète pour gérer les personnes

```java
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class GestionnairePersonnes {
    private static final String DB_URL = "jdbc:sqlite:personnes.db";

    // Initialiser la base de données
    public static void initialiserDB() {
        String sqlCreate = """
            CREATE TABLE IF NOT EXISTS personnes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nom TEXT NOT NULL,
                age INTEGER,
                email TEXT UNIQUE
            );
        """;

        try (Connection conn = DriverManager.getConnection(DB_URL);
             Statement stmt = conn.createStatement()) {

            stmt.execute(sqlCreate);
            System.out.println("Table 'personnes' créée ou existe déjà.");

        } catch (SQLException e) {
            System.err.println("Erreur création table : " + e.getMessage());
        }
    }

    // Ajouter une personne
    public static boolean ajouterPersonne(String nom, int age, String email) {
        String sql = "INSERT INTO personnes(nom, age, email) VALUES(?, ?, ?)";

        try (Connection conn = DriverManager.getConnection(DB_URL);
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setString(1, nom);
            pstmt.setInt(2, age);
            pstmt.setString(3, email);

            int lignesAffectees = pstmt.executeUpdate();
            System.out.println("✅ " + nom + " ajouté(e) avec succès !");
            return lignesAffectees > 0;

        } catch (SQLException e) {
            System.err.println("❌ Erreur ajout : " + e.getMessage());
            return false;
        }
    }

    // Lire toutes les personnes
    public static void listerPersonnes() {
        String sql = "SELECT id, nom, age, email FROM personnes ORDER BY nom";

        try (Connection conn = DriverManager.getConnection(DB_URL);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {

            System.out.println("\n📋 Liste des personnes :");
            System.out.println("ID | Nom                | Âge | Email");
            System.out.println("-".repeat(50));

            while (rs.next()) {
                System.out.printf("%-2d | %-18s | %-3d | %s%n",
                    rs.getInt("id"),
                    rs.getString("nom"),
                    rs.getInt("age"),
                    rs.getString("email")
                );
            }

        } catch (SQLException e) {
            System.err.println("Erreur lecture : " + e.getMessage());
        }
    }

    // Rechercher par nom
    public static void rechercherPersonne(String nomRecherche) {
        String sql = "SELECT * FROM personnes WHERE nom LIKE ? ORDER BY nom";

        try (Connection conn = DriverManager.getConnection(DB_URL);
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setString(1, "%" + nomRecherche + "%");
            ResultSet rs = pstmt.executeQuery();

            System.out.println("\n🔍 Résultats pour '" + nomRecherche + "' :");
            boolean trouve = false;

            while (rs.next()) {
                trouve = true;
                System.out.printf("  %s (%d ans) - %s%n",
                    rs.getString("nom"),
                    rs.getInt("age"),
                    rs.getString("email")
                );
            }

            if (!trouve) {
                System.out.println("  Aucun résultat trouvé.");
            }

        } catch (SQLException e) {
            System.err.println("Erreur recherche : " + e.getMessage());
        }
    }

    // Mettre à jour l'âge
    public static boolean mettreAJourAge(int id, int nouvelAge) {
        String sql = "UPDATE personnes SET age = ? WHERE id = ?";

        try (Connection conn = DriverManager.getConnection(DB_URL);
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setInt(1, nouvelAge);
            pstmt.setInt(2, id);

            int lignesAffectees = pstmt.executeUpdate();
            if (lignesAffectees > 0) {
                System.out.println("✅ Âge mis à jour pour l'ID " + id);
                return true;
            } else {
                System.out.println("❌ Aucune personne trouvée avec l'ID " + id);
                return false;
            }

        } catch (SQLException e) {
            System.err.println("Erreur mise à jour : " + e.getMessage());
            return false;
        }
    }

    // Programme principal de démonstration
    public static void main(String[] args) {
        // Initialiser la base de données
        initialiserDB();

        // Ajouter quelques personnes
        ajouterPersonne("Alice Martin", 25, "alice.martin@email.com");
        ajouterPersonne("Bob Dupont", 30, "bob.dupont@email.com");
        ajouterPersonne("Claire Rousseau", 28, "claire.rousseau@email.com");

        // Lister toutes les personnes
        listerPersonnes();

        // Rechercher
        rechercherPersonne("Alice");

        // Mettre à jour
        mettreAJourAge(1, 26);

        // Afficher le résultat final
        listerPersonnes();
    }
}
```

### Gestion des transactions en Java

```java
public static void exempleTransaction() {
    try (Connection conn = DriverManager.getConnection(DB_URL)) {
        // Désactiver l'auto-commit
        conn.setAutoCommit(false);

        try {
            // Première opération
            PreparedStatement stmt1 = conn.prepareStatement(
                "INSERT INTO personnes (nom, age, email) VALUES (?, ?, ?)"
            );
            stmt1.setString(1, "Transaction Test 1");
            stmt1.setInt(2, 35);
            stmt1.setString(3, "test1@email.com");
            stmt1.executeUpdate();

            // Deuxième opération
            PreparedStatement stmt2 = conn.prepareStatement(
                "INSERT INTO personnes (nom, age, email) VALUES (?, ?, ?)"
            );
            stmt2.setString(1, "Transaction Test 2");
            stmt2.setInt(2, 40);
            stmt2.setString(3, "test2@email.com");
            stmt2.executeUpdate();

            // Confirmer toutes les opérations
            conn.commit();
            System.out.println("✅ Transaction réussie !");

        } catch (SQLException e) {
            // Annuler en cas d'erreur
            conn.rollback();
            System.err.println("❌ Transaction annulée : " + e.getMessage());
        } finally {
            // Réactiver l'auto-commit
            conn.setAutoCommit(true);
        }

    } catch (SQLException e) {
        System.err.println("Erreur connexion : " + e.getMessage());
    }
}
```

## SQLite avec JavaScript

### JavaScript côté serveur (Node.js)

#### Installation des dépendances

```bash
npm init -y
npm install sqlite3
# ou pour une version plus moderne :
npm install better-sqlite3
```

### Exemple avec sqlite3 (approche callback)

```javascript
const sqlite3 = require('sqlite3').verbose();
const path = require('path');

// Créer ou ouvrir une base de données
const dbPath = path.join(__dirname, 'personnes.db');
const db = new sqlite3.Database(dbPath, (err) => {
    if (err) {
        console.error('Erreur ouverture :', err.message);
    } else {
        console.log('✅ Connecté à SQLite');
    }
});

// Créer une table
db.serialize(() => {
    db.run(`
        CREATE TABLE IF NOT EXISTS personnes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            nom TEXT NOT NULL,
            age INTEGER,
            email TEXT UNIQUE
        )
    `, (err) => {
        if (err) {
            console.error('Erreur création table :', err.message);
        } else {
            console.log('📋 Table créée ou existe déjà');
        }
    });
});

// Fonctions utilitaires
function ajouterPersonne(nom, age, email) {
    const stmt = db.prepare('INSERT INTO personnes (nom, age, email) VALUES (?, ?, ?)');

    stmt.run([nom, age, email], function(err) {
        if (err) {
            console.error('❌ Erreur ajout :', err.message);
        } else {
            console.log(`✅ ${nom} ajouté(e) avec l'ID ${this.lastID}`);
        }
    });

    stmt.finalize();
}

function listerPersonnes() {
    db.all('SELECT * FROM personnes ORDER BY nom', [], (err, rows) => {
        if (err) {
            console.error('Erreur lecture :', err.message);
            return;
        }

        console.log('\n📋 Liste des personnes :');
        console.log('ID | Nom                | Âge | Email');
        console.log('-'.repeat(50));

        rows.forEach(row => {
            console.log(`${row.id.toString().padStart(2)} | ${row.nom.padEnd(18)} | ${row.age.toString().padStart(3)} | ${row.email}`);
        });
    });
}

function rechercherPersonne(terme) {
    db.all(
        'SELECT * FROM personnes WHERE nom LIKE ? OR email LIKE ?',
        [`%${terme}%`, `%${terme}%`],
        (err, rows) => {
            if (err) {
                console.error('Erreur recherche :', err.message);
                return;
            }

            console.log(`\n🔍 Résultats pour '${terme}' :`);
            if (rows.length === 0) {
                console.log('  Aucun résultat trouvé');
            } else {
                rows.forEach(row => {
                    console.log(`  ${row.nom} (${row.age} ans) - ${row.email}`);
                });
            }
        }
    );
}

// Exemple d'utilisation
setTimeout(() => {
    ajouterPersonne('Alice Martin', 25, 'alice.martin@email.com');
    ajouterPersonne('Bob Dupont', 30, 'bob.dupont@email.com');
    ajouterPersonne('Claire Rousseau', 28, 'claire.rousseau@email.com');

    setTimeout(() => {
        listerPersonnes();
        rechercherPersonne('Alice');

        // Fermer la base de données
        setTimeout(() => {
            db.close((err) => {
                if (err) {
                    console.error('Erreur fermeture :', err.message);
                } else {
                    console.log('🔒 Connexion fermée');
                }
            });
        }, 1000);
    }, 500);
}, 100);
```

### Exemple avec better-sqlite3 (approche synchrone)

```javascript
const Database = require('better-sqlite3');

class GestionnairePersonnes {
    constructor(fichierDB = 'personnes.db') {
        this.db = new Database(fichierDB);
        this.initialiser();
    }

    initialiser() {
        // Créer la table
        const createTable = this.db.prepare(`
            CREATE TABLE IF NOT EXISTS personnes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nom TEXT NOT NULL,
                age INTEGER,
                email TEXT UNIQUE
            )
        `);

        createTable.run();
        console.log('📋 Table initialisée');

        // Préparer les requêtes
        this.ajouterStmt = this.db.prepare(
            'INSERT INTO personnes (nom, age, email) VALUES (?, ?, ?)'
        );
        this.listerStmt = this.db.prepare(
            'SELECT * FROM personnes ORDER BY nom'
        );
        this.rechercherStmt = this.db.prepare(
            'SELECT * FROM personnes WHERE nom LIKE ? OR email LIKE ?'
        );
        this.mettreAJourStmt = this.db.prepare(
            'UPDATE personnes SET age = ? WHERE id = ?'
        );
        this.supprimerStmt = this.db.prepare(
            'DELETE FROM personnes WHERE id = ?'
        );
    }

    ajouterPersonne(nom, age, email) {
        try {
            const result = this.ajouterStmt.run(nom, age, email);
            console.log(`✅ ${nom} ajouté(e) avec l'ID ${result.lastInsertRowid}`);
            return result.lastInsertRowid;
        } catch (error) {
            console.error(`❌ Erreur ajout ${nom} :`, error.message);
            return null;
        }
    }

    listerPersonnes() {
        try {
            const personnes = this.listerStmt.all();

            console.log('\n📋 Liste des personnes :');
            console.log('ID | Nom                | Âge | Email');
            console.log('-'.repeat(60));

            personnes.forEach(p => {
                console.log(`${p.id.toString().padStart(2)} | ${p.nom.padEnd(18)} | ${p.age.toString().padStart(3)} | ${p.email}`);
            });

            return personnes;
        } catch (error) {
            console.error('❌ Erreur liste :', error.message);
            return [];
        }
    }

    rechercherPersonnes(terme) {
        try {
            const personnes = this.rechercherStmt.all(`%${terme}%`, `%${terme}%`);

            console.log(`\n🔍 Résultats pour '${terme}' :`);
            if (personnes.length === 0) {
                console.log('  Aucun résultat trouvé');
            } else {
                personnes.forEach(p => {
                    console.log(`  ${p.nom} (${p.age} ans) - ${p.email}`);
                });
            }

            return personnes;
        } catch (error) {
            console.error('❌ Erreur recherche :', error.message);
            return [];
        }
    }

    mettreAJourAge(id, nouvelAge) {
        try {
            const result = this.mettreAJourStmt.run(nouvelAge, id);
            if (result.changes > 0) {
                console.log(`✅ Âge mis à jour pour l'ID ${id}`);
                return true;
            } else {
                console.log(`❌ Aucune personne trouvée avec l'ID ${id}`);
                return false;
            }
        } catch (error) {
            console.error('❌ Erreur mise à jour :', error.message);
            return false;
        }
    }

    supprimerPersonne(id) {
        try {
            const result = this.supprimerStmt.run(id);
            if (result.changes > 0) {
                console.log(`✅ Personne supprimée (ID ${id})`);
                return true;
            } else {
                console.log(`❌ Aucune personne trouvée avec l'ID ${id}`);
                return false;
            }
        } catch (error) {
            console.error('❌ Erreur suppression :', error.message);
            return false;
        }
    }

    // Transaction exemple
    ajouterPlusieurPersonnes(personnes) {
        const transaction = this.db.transaction((personnes) => {
            for (const personne of personnes) {
                this.ajouterStmt.run(personne.nom, personne.age, personne.email);
            }
        });

        try {
            transaction(personnes);
            console.log(`✅ ${personnes.length} personnes ajoutées en transaction`);
            return true;
        } catch (error) {
            console.error('❌ Erreur transaction :', error.message);
            return false;
        }
    }

    fermer() {
        this.db.close();
        console.log('🔒 Base de données fermée');
    }
}

// Exemple d'utilisation
const gestionnaire = new GestionnairePersonnes();

// Ajouter des personnes individuellement
gestionnaire.ajouterPersonne('Alice Martin', 25, 'alice.martin@email.com');
gestionnaire.ajouterPersonne('Bob Dupont', 30, 'bob.dupont@email.com');

// Ajouter plusieurs personnes en transaction
const nouvellesPersonnes = [
    { nom: 'Claire Rousseau', age: 28, email: 'claire.rousseau@email.com' },
    { nom: 'David Moreau', age: 35, email: 'david.moreau@email.com' },
    { nom: 'Emma Leroy', age: 27, email: 'emma.leroy@email.com' }
];

gestionnaire.ajouterPlusieurPersonnes(nouvellesPersonnes);

// Opérations diverses
gestionnaire.listerPersonnes();
gestionnaire.rechercherPersonnes('Martin');
gestionnaire.mettreAJourAge(1, 26);

// Fermer proprement
gestionnaire.fermer();
```

### JavaScript dans le navigateur (WebSQL - Obsolète)

**⚠️ Note importante** : WebSQL est obsolète et ne doit plus être utilisé. Pour SQLite dans le navigateur, utilisez plutôt :

#### IndexedDB (recommandé pour le navigateur)
```javascript
// Exemple simple avec IndexedDB (alternative moderne)
function ouvrirDB() {
    return new Promise((resolve, reject) => {
        const request = indexedDB.open('PersonnesDB', 1);

        request.onerror = () => reject(request.error);
        request.onsuccess = () => resolve(request.result);

        request.onupgradeneeded = (event) => {
            const db = event.target.result;
            const objectStore = db.createObjectStore('personnes',
                { keyPath: 'id', autoIncrement: true });

            objectStore.createIndex('nom', 'nom', { unique: false });
            objectStore.createIndex('email', 'email', { unique: true });
        };
    });
}

async function ajouterPersonne(nom, age, email) {
    const db = await ouvrirDB();
    const transaction = db.transaction(['personnes'], 'readwrite');
    const store = transaction.objectStore('personnes');

    return new Promise((resolve, reject) => {
        const request = store.add({ nom, age, email });
        request.onsuccess = () => resolve(request.result);
        request.onerror = () => reject(request.error);
    });
}
```

## Comparaison des approches

### Tableau récapitulatif

| Critère | C | Java | JavaScript (Node.js) |
|---------|---|------|---------------------|
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Facilité** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Sécurité** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Écosystème** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Courbe d'apprentissage** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### Conseils pour choisir

**Choisissez C si** :
- Vous développez des applications système
- La performance est critique
- Vous avez besoin d'un contrôle total
- Vous travaillez déjà en C/C++

**Choisissez Java si** :
- Vous développez des applications d'entreprise
- Vous travaillez sur Android
- Vous voulez une approche orientée objet solide
- L'écosystème Java est important pour vous

**Choisissez JavaScript/Node.js si** :
- Vous développez des applications web
- La rapidité de développement est importante
- Vous voulez unifier frontend et backend
- Vous débutez en programmation de base de données

## Bonnes pratiques communes à tous les langages

### 1. Toujours utiliser des requêtes préparées

```c
// ❌ Dangereux en C
sprintf(query, "SELECT * FROM users WHERE name = '%s'", user_input);

// ✅ Sécurisé en C
sqlite3_prepare_v2(db, "SELECT * FROM users WHERE name = ?", -1, &stmt, NULL);
sqlite3_bind_text(stmt, 1, user_input, -1, SQLITE_STATIC);
```

```java
// ❌ Dangereux en Java
String sql = "SELECT * FROM users WHERE name = '" + userInput + "'";

// ✅ Sécurisé en Java
PreparedStatement pstmt = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
pstmt.setString(1, userInput);
```

```javascript
// ❌ Dangereux en JavaScript
db.all(`SELECT * FROM users WHERE name = '${userInput}'`);

// ✅ Sécurisé en JavaScript
db.all('SELECT * FROM users WHERE name = ?', [userInput]);
```

### 2. Gestion des erreurs appropriée

**En C** :
```c
if (sqlite3_step(stmt) != SQLITE_DONE) {
    printf("Erreur : %s\n", sqlite3_errmsg(db));
    sqlite3_finalize(stmt);
    return -1;
}
```

**En Java** :
```java
try {
    // Opérations de base de données
} catch (SQLException e) {
    logger.error("Erreur base de données : " + e.getMessage());
    // Gestion appropriée de l'erreur
}
```

**En JavaScript** :
```javascript
try {
    const result = stmt.run(params);
} catch (error) {
    console.error('Erreur SQLite :', error.message);
    // Gestion appropriée de l'erreur
}
```

### 3. Fermeture appropriée des ressources

**En C** :
```c
sqlite3_finalize(stmt);  // Libérer la requête préparée
sqlite3_close(db);       // Fermer la base de données
```

**En Java** :
```java
// Utiliser try-with-resources (Java 7+)
try (Connection conn = DriverManager.getConnection(url);
     PreparedStatement pstmt = conn.prepareStatement(sql)) {
    // Utilisation automatique des ressources
} // Fermeture automatique
```

**En JavaScript** :
```javascript
// sqlite3
db.close((err) => {
    if (err) console.error(err.message);
});

// better-sqlite3
db.close();
```

## Exemples d'intégration avancés

### Système de logging multi-langages

#### Version C
```c
#include <stdio.h>
#include <sqlite3.h>
#include <time.h>
#include <string.h>

typedef struct {
    sqlite3 *db;
    sqlite3_stmt *insert_stmt;
} Logger;

Logger* logger_init(const char* db_path) {
    Logger *logger = malloc(sizeof(Logger));

    if (sqlite3_open(db_path, &logger->db) != SQLITE_OK) {
        free(logger);
        return NULL;
    }

    // Créer la table de logs
    const char *create_sql =
        "CREATE TABLE IF NOT EXISTS logs ("
        "id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "timestamp TEXT NOT NULL,"
        "level TEXT NOT NULL,"
        "message TEXT NOT NULL"
        ");";

    sqlite3_exec(logger->db, create_sql, NULL, NULL, NULL);

    // Préparer la requête d'insertion
    sqlite3_prepare_v2(logger->db,
        "INSERT INTO logs (timestamp, level, message) VALUES (?, ?, ?)",
        -1, &logger->insert_stmt, NULL);

    return logger;
}

void logger_log(Logger *logger, const char *level, const char *message) {
    char timestamp[64];
    time_t now = time(NULL);
    strftime(timestamp, sizeof(timestamp), "%Y-%m-%d %H:%M:%S", localtime(&now));

    sqlite3_bind_text(logger->insert_stmt, 1, timestamp, -1, SQLITE_STATIC);
    sqlite3_bind_text(logger->insert_stmt, 2, level, -1, SQLITE_STATIC);
    sqlite3_bind_text(logger->insert_stmt, 3, message, -1, SQLITE_STATIC);

    sqlite3_step(logger->insert_stmt);
    sqlite3_reset(logger->insert_stmt);
}

void logger_cleanup(Logger *logger) {
    sqlite3_finalize(logger->insert_stmt);
    sqlite3_close(logger->db);
    free(logger);
}
```

#### Version Java
```java
import java.sql.*;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class DatabaseLogger {
    private Connection conn;
    private PreparedStatement insertStmt;
    private static final DateTimeFormatter FORMATTER =
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    public DatabaseLogger(String dbPath) throws SQLException {
        String url = "jdbc:sqlite:" + dbPath;
        this.conn = DriverManager.getConnection(url);

        // Créer la table
        String createSql = """
            CREATE TABLE IF NOT EXISTS logs (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp TEXT NOT NULL,
                level TEXT NOT NULL,
                message TEXT NOT NULL
            );
        """;

        try (Statement stmt = conn.createStatement()) {
            stmt.execute(createSql);
        }

        // Préparer la requête d'insertion
        this.insertStmt = conn.prepareStatement(
            "INSERT INTO logs (timestamp, level, message) VALUES (?, ?, ?)"
        );
    }

    public void log(String level, String message) {
        try {
            String timestamp = LocalDateTime.now().format(FORMATTER);

            insertStmt.setString(1, timestamp);
            insertStmt.setString(2, level);
            insertStmt.setString(3, message);

            insertStmt.executeUpdate();
        } catch (SQLException e) {
            System.err.println("Erreur logging : " + e.getMessage());
        }
    }

    public void info(String message) { log("INFO", message); }
    public void warn(String message) { log("WARN", message); }
    public void error(String message) { log("ERROR", message); }

    public void close() throws SQLException {
        if (insertStmt != null) insertStmt.close();
        if (conn != null) conn.close();
    }

    // Méthode pour lire les logs récents
    public void afficherLogsRecents(int limite) {
        String sql = "SELECT * FROM logs ORDER BY id DESC LIMIT ?";

        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setInt(1, limite);
            ResultSet rs = pstmt.executeQuery();

            System.out.println("📋 Logs récents :");
            while (rs.next()) {
                System.out.printf("[%s] %s: %s%n",
                    rs.getString("timestamp"),
                    rs.getString("level"),
                    rs.getString("message")
                );
            }
        } catch (SQLException e) {
            System.err.println("Erreur lecture logs : " + e.getMessage());
        }
    }
}

// Exemple d'utilisation
class ExempleLogger {
    public static void main(String[] args) {
        try (DatabaseLogger logger = new DatabaseLogger("app.db")) {
            logger.info("Application démarrée");
            logger.warn("Attention : mémoire faible");
            logger.error("Erreur de connexion réseau");

            logger.afficherLogsRecents(10);
        } catch (SQLException e) {
            System.err.println("Erreur : " + e.getMessage());
        }
    }
}
```

#### Version JavaScript (Node.js)
```javascript
const Database = require('better-sqlite3');

class DatabaseLogger {
    constructor(dbPath = 'app.db') {
        this.db = new Database(dbPath);
        this.init();
    }

    init() {
        // Créer la table de logs
        this.db.exec(`
            CREATE TABLE IF NOT EXISTS logs (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp TEXT NOT NULL,
                level TEXT NOT NULL,
                message TEXT NOT NULL
            );
        `);

        // Préparer les requêtes
        this.insertStmt = this.db.prepare(
            'INSERT INTO logs (timestamp, level, message) VALUES (?, ?, ?)'
        );

        this.selectRecentStmt = this.db.prepare(
            'SELECT * FROM logs ORDER BY id DESC LIMIT ?'
        );
    }

    log(level, message) {
        const timestamp = new Date().toISOString().replace('T', ' ').slice(0, 19);

        try {
            this.insertStmt.run(timestamp, level, message);
        } catch (error) {
            console.error('Erreur logging :', error.message);
        }
    }

    info(message) { this.log('INFO', message); }
    warn(message) { this.log('WARN', message); }
    error(message) { this.log('ERROR', message); }
    debug(message) { this.log('DEBUG', message); }

    getRecentLogs(limit = 10) {
        try {
            return this.selectRecentStmt.all(limit);
        } catch (error) {
            console.error('Erreur lecture logs :', error.message);
            return [];
        }
    }

    displayRecentLogs(limit = 10) {
        const logs = this.getRecentLogs(limit);

        console.log('📋 Logs récents :');
        logs.forEach(log => {
            const emoji = {
                'INFO': 'ℹ️',
                'WARN': '⚠️',
                'ERROR': '❌',
                'DEBUG': '🐛'
            }[log.level] || '📝';

            console.log(`${emoji} [${log.timestamp}] ${log.level}: ${log.message}`);
        });
    }

    // Méthode pour nettoyer les anciens logs
    cleanOldLogs(daysToKeep = 30) {
        const cutoffDate = new Date();
        cutoffDate.setDate(cutoffDate.getDate() - daysToKeep);
        const cutoffString = cutoffDate.toISOString().replace('T', ' ').slice(0, 19);

        const deleteStmt = this.db.prepare('DELETE FROM logs WHERE timestamp < ?');
        const result = deleteStmt.run(cutoffString);

        this.info(`Nettoyage : ${result.changes} anciens logs supprimés`);
        return result.changes;
    }

    // Statistiques des logs
    getLogStats() {
        const statsStmt = this.db.prepare(`
            SELECT
                level,
                COUNT(*) as count,
                DATE(timestamp) as date
            FROM logs
            WHERE DATE(timestamp) = DATE('now')
            GROUP BY level, DATE(timestamp)
            ORDER BY count DESC
        `);

        return statsStmt.all();
    }

    close() {
        this.db.close();
    }
}

// Exemple d'utilisation avancée
const logger = new DatabaseLogger();

// Simuler des événements d'application
logger.info('🚀 Application démarrée');
logger.info('📊 Chargement de la configuration');
logger.warn('⚠️ Mémoire utilisée à 80%');
logger.error('❌ Échec de connexion à l\'API externe');
logger.debug('🔍 Variable X = 42');

// Afficher les logs récents
logger.displayRecentLogs(5);

// Afficher les statistiques du jour
console.log('\n📊 Statistiques du jour :');
const stats = logger.getLogStats();
stats.forEach(stat => {
    console.log(`  ${stat.level}: ${stat.count} occurrences`);
});

// Nettoyer les anciens logs
logger.cleanOldLogs(7); // Garder seulement 7 jours

// Fermer proprement
logger.close();
```

## Migration entre langages

Parfois, vous devez migrer un projet d'un langage à un autre. Voici comment procéder :

### 1. Préserver le schéma de base de données
```sql
-- Exporter le schéma (compatible tous langages)
.schema > schema.sql

-- Exporter les données
.dump > backup.sql
```

### 2. Adapter les types de données

| SQLite | C | Java | JavaScript |
|--------|---|------|------------|
| INTEGER | int, long | int, long | number |
| REAL | double | double | number |
| TEXT | char* | String | string |
| BLOB | void* | byte[] | Buffer |

### 3. Script de migration automatisé

```bash
#!/bin/bash
# migrate_db.sh - Script de migration de données

SOURCE_DB="$1"
TARGET_LANG="$2"

echo "Migration de $SOURCE_DB vers $TARGET_LANG"

# Exporter le schéma
sqlite3 "$SOURCE_DB" ".schema" > schema.sql

# Exporter les données en CSV pour import facile
sqlite3 "$SOURCE_DB" -csv -header "SELECT * FROM table_name;" > data.csv

case "$TARGET_LANG" in
    "java")
        echo "Génération du code Java..."
        # Générer automatiquement les classes Java
        ;;
    "javascript")
        echo "Génération du code JavaScript..."
        # Générer automatiquement les modules Node.js
        ;;
    "c")
        echo "Génération du code C..."
        # Générer automatiquement les structures C
        ;;
esac

echo "Migration terminée !"
```

## Exercices pratiques

### Exercice 1 : Gestionnaire de tâches multi-langages
Implémentez un gestionnaire de tâches simple dans les trois langages avec :
- Ajout de tâches (titre, description, priorité, statut)
- Marquage comme terminé
- Filtrage par statut et priorité
- Statistiques (nombre de tâches par statut)

### Exercice 2 : Système de cache
Créez un système de cache simple qui :
- Stocke des paires clé-valeur avec expiration
- Nettoie automatiquement les entrées expirées
- Fournit des statistiques d'utilisation

### Exercice 3 : Convertisseur de données
Développez un outil qui :
- Lit des données d'un fichier CSV
- Les insère dans SQLite
- Permet d'exporter vers différents formats (JSON, XML)

## Ressources et outils

### Documentation officielle
- **SQLite C API** : [https://sqlite.org/capi3ref.html](https://sqlite.org/capi3ref.html)
- **JDBC SQLite** : [https://github.com/xerial/sqlite-jdbc](https://github.com/xerial/sqlite-jdbc)
- **Node.js sqlite3** : [https://github.com/mapbox/node-sqlite3](https://github.com/mapbox/node-sqlite3)
- **better-sqlite3** : [https://github.com/JoshuaWise/better-sqlite3](https://github.com/JoshuaWise/better-sqlite3)

### Outils de développement
- **DB Browser for SQLite** : Interface graphique multiplateforme
- **SQLite Expert** : Outil professionnel pour Windows
- **Postbird** : Client léger et moderne

### Extensions utiles
- **SQLite JSON1** : Support natif du JSON
- **SQLite FTS5** : Recherche full-text avancée
- **SQLite R*Tree** : Index spatial pour données géographiques

## Résumé

Cette section vous a présenté l'intégration de SQLite avec trois langages majeurs :

✅ **C** : Performance maximale et contrôle total
✅ **Java** : Robustesse et écosystème entreprise
✅ **JavaScript** : Simplicité et développement web

**Points clés à retenir** :
- Chaque langage a ses spécificités mais les concepts restent similaires
- Toujours utiliser des requêtes préparées pour la sécurité
- Gérer proprement les erreurs et fermer les ressources
- Choisir le langage selon le contexte de votre projet

Avec ces bases solides, vous pouvez maintenant intégrer SQLite dans vos projets, quel que soit le langage choisi. La prochaine section explorera l'utilisation des ORM pour simplifier encore davantage le développement.

⏭️
