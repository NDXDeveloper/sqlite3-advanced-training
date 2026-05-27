🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.6 Implémenter la recherche plein texte avec FTS5

## Qu'est-ce que la recherche plein texte ?

La **recherche plein texte** (Full-Text Search - FTS) permet de rechercher des mots ou expressions dans de grandes quantités de texte, bien plus efficacement qu'avec un simple `LIKE '%terme%'`.

### Analogie simple
Imaginez la différence entre :
- **Recherche basique** : Chercher un mot dans un livre en lisant page par page
- **Recherche plein texte** : Utiliser l'index d'un livre pour trouver instantanément toutes les occurrences

### Pourquoi utiliser FTS5 ?

**Avantages de FTS5 :**
- ⚡ **Performance** : Recherche instantanée même sur millions de documents
- 🎯 **Pertinence** : Classement des résultats par score de pertinence
- 🔍 **Flexibilité** : Recherche approximative, opérateurs booléens, proximité
- ✨ **Fonctionnalités avancées** : Surlignage, extraits, suggestions

**Cas d'usage typiques :**
- 📚 Moteur de recherche pour blog ou site web
- 📖 Recherche dans documentation technique
- 📧 Recherche d'emails ou messages
- 🗞️ Analyse d'articles de presse
- 🎵 Recherche dans métadonnées musicales

## Premiers pas avec FTS5

### Vérifier la disponibilité de FTS5

```python
import sqlite3

def verifier_fts5():
    """Vérifie si FTS5 est disponible dans votre installation SQLite"""

    conn = sqlite3.connect(':memory:')

    try:
        # Tenter de créer une table FTS5
        conn.execute("CREATE VIRTUAL TABLE test_fts USING fts5(content)")
        print("✅ FTS5 est disponible !")
        return True

    except sqlite3.OperationalError as e:
        if "no such module: fts5" in str(e):
            print("❌ FTS5 n'est pas disponible dans votre installation SQLite")
            print("💡 Solution : Installer une version récente de SQLite (3.20+)")
        else:
            print(f"❌ Erreur inattendue : {e}")
        return False

    finally:
        conn.close()

# Vérifier avant de continuer
if verifier_fts5():
    print("🚀 Prêt à explorer FTS5 !")
else:
    print("⚠️ Installez une version récente de SQLite pour suivre ce tutoriel")
```

### Création d'une table FTS5 basique

```python
def creer_premiere_table_fts():
    """Création d'une table FTS5 simple pour débuter"""

    conn = sqlite3.connect(':memory:')

    # Créer une table FTS5 pour des articles de blog
    conn.execute("""
        CREATE VIRTUAL TABLE articles_fts USING fts5(
            titre,
            contenu,
            auteur
        )
    """)

    # Insérer quelques articles d'exemple
    articles_exemple = [
        ("Introduction à Python", "Python est un langage de programmation facile à apprendre. Il est utilisé dans le développement web, la science des données, et l'intelligence artificielle.", "Alice Martin"),
        ("Guide SQLite", "SQLite est une base de données légère et rapide. Elle est parfaite pour les applications mobiles et les prototypes.", "Bob Durand"),
        ("Développement web moderne", "Le développement web utilise des technologies comme HTML, CSS, JavaScript et Python. Les frameworks modernes facilitent la création d'applications.", "Alice Martin"),
        ("Intelligence artificielle", "L'IA révolutionne de nombreux domaines. Machine learning et deep learning sont des branches importantes de l'intelligence artificielle.", "Charlie Dubois"),
        ("Bases de données NoSQL", "Les bases NoSQL comme MongoDB offrent une alternative aux bases relationnelles. Elles sont adaptées aux données non-structurées.", "Alice Martin")
    ]

    # Insérer les données
    conn.executemany(
        "INSERT INTO articles_fts (titre, contenu, auteur) VALUES (?, ?, ?)",
        articles_exemple
    )

    print("✅ Table FTS5 créée avec 5 articles d'exemple")

    return conn

# Test de base
def premiere_recherche():
    """Première recherche simple avec FTS5"""

    conn = creer_premiere_table_fts()

    # Rechercher tous les articles contenant "Python"
    cursor = conn.execute("""
        SELECT titre, auteur, snippet(articles_fts, 1, '<mark>', '</mark>', '...', 32) as extrait
        FROM articles_fts
        WHERE articles_fts MATCH 'Python'
    """)

    print("\n🔍 Recherche : 'Python'")
    print("-" * 50)

    for titre, auteur, extrait in cursor:
        print(f"📰 {titre}")
        print(f"✍️  {auteur}")
        print(f"📝 {extrait}")
        print()

    conn.close()

premiere_recherche()
```

## Syntaxe de recherche FTS5

### Recherches de base

```python
def demo_syntaxes_recherche():
    """Démonstration des différentes syntaxes de recherche FTS5"""

    conn = creer_premiere_table_fts()

    # Types de recherches possibles
    recherches = [
        ("Terme simple", "Python"),
        ("Plusieurs termes (ET)", "Python développement"),
        ("Opérateur OR", "Python OR JavaScript"),
        ("Phrase exacte", '"intelligence artificielle"'),
        ("Exclusion (NOT)", "développement NOT web"),
        ("Préfixe", "program*"),
        ("Colonne spécifique", "auteur:Alice"),
        ("Proximité", "Python NEAR/3 programmation")
    ]

    for description, requete in recherches:
        print(f"\n🔍 {description}: '{requete}'")
        print("-" * 40)

        try:
            cursor = conn.execute(
                "SELECT titre FROM articles_fts WHERE articles_fts MATCH ?",
                (requete,)
            )

            resultats = cursor.fetchall()

            if resultats:
                for i, (titre,) in enumerate(resultats, 1):
                    print(f"  {i}. {titre}")
            else:
                print("  Aucun résultat trouvé")

        except sqlite3.OperationalError as e:
            print(f"  ❌ Erreur de syntaxe : {e}")

    conn.close()

demo_syntaxes_recherche()
```

### Recherche avec classement par pertinence

```python
def demo_classement_pertinence():
    """Démonstration du classement par pertinence avec FTS5"""

    conn = creer_premiere_table_fts()

    # Ajouter plus d'articles pour un test plus intéressant
    articles_supplementaires = [
        ("Python avancé", "Ce guide Python avancé couvre les concepts complexes de Python. Programmation Python pour experts.", "David Tech"),
        ("Tutoriel Python débutant", "Apprendre Python facilement. Python est parfait pour débuter en programmation.", "Emma Code"),
        ("Python dans l'IA", "Python est le langage de choix pour l'intelligence artificielle et le machine learning.", "Frank AI"),
        ("Histoire de Python", "Python a été créé par Guido van Rossum. L'évolution de Python à travers les années.", "Grace History")
    ]

    conn.executemany(
        "INSERT INTO articles_fts (titre, contenu, auteur) VALUES (?, ?, ?)",
        articles_supplementaires
    )

    # Recherche avec score de pertinence
    print("🎯 CLASSEMENT PAR PERTINENCE - Recherche : 'Python'")
    print("=" * 60)

    cursor = conn.execute("""
        SELECT
            titre,
            auteur,
            bm25(articles_fts) as score,
            snippet(articles_fts, 1, '**', '**', '...', 20) as extrait
        FROM articles_fts
        WHERE articles_fts MATCH 'Python'
        ORDER BY bm25(articles_fts)
    """)

    for i, (titre, auteur, score, extrait) in enumerate(cursor, 1):
        print(f"{i}. 📰 {titre} (Score: {score:.3f})")
        print(f"   ✍️ {auteur}")
        print(f"   📝 {extrait}")
        print()

    # Recherche pondérée par colonne
    print("⚖️ PONDÉRATION PAR COLONNE")
    print("-" * 30)
    print("(Titre plus important que contenu)")

    cursor = conn.execute("""
        SELECT
            titre,
            bm25(articles_fts, 10.0, 1.0, 1.0) as score_pondere
        FROM articles_fts
        WHERE articles_fts MATCH 'Python'
        ORDER BY score_pondere
        LIMIT 3
    """)

    for i, (titre, score) in enumerate(cursor, 1):
        print(f"  {i}. {titre} (Score pondéré: {score:.3f})")

    conn.close()

demo_classement_pertinence()
```

## Indexation et synchronisation

### Table FTS5 externe (recommandée)

```python
def demo_table_fts_externe():
    """Démonstration d'une table FTS5 externe synchronisée avec une table normale"""

    conn = sqlite3.connect(':memory:')

    # 1. Créer la table principale
    conn.execute("""
        CREATE TABLE articles (
            id INTEGER PRIMARY KEY,
            titre TEXT NOT NULL,
            contenu TEXT NOT NULL,
            auteur TEXT NOT NULL,
            date_publication DATE DEFAULT CURRENT_DATE,
            statut TEXT DEFAULT 'publié',
            vues INTEGER DEFAULT 0
        )
    """)

    # 2. Créer la table FTS5 externe
    conn.execute("""
        CREATE VIRTUAL TABLE articles_fts USING fts5(
            titre,
            contenu,
            auteur,
            content='articles',      -- Table source
            content_rowid='id'       -- Clé primaire de la table source
        )
    """)

    # 3. Insérer des données dans la table principale
    articles = [
        ("Guide complet Python", "Python est un langage polyvalent utilisé en développement web, data science et IA.", "Alice Develop", "2024-01-15"),
        ("Introduction à SQLite", "SQLite est une base de données embarquée parfaite pour les applications légères.", "Bob Database", "2024-01-20"),
        ("Tutoriel JavaScript", "JavaScript moderne avec ES6+, async/await et les frameworks React/Vue.", "Charlie Frontend", "2024-01-25"),
        ("Machine Learning Python", "Utiliser scikit-learn et TensorFlow pour créer des modèles d'IA.", "Alice Develop", "2024-02-01"),
        ("Bases de données avancées", "Concepts avancés : indexation, optimisation, transactions ACID.", "Bob Database", "2024-02-05")
    ]

    cursor = conn.executemany(
        "INSERT INTO articles (titre, contenu, auteur, date_publication) VALUES (?, ?, ?, ?)",
        articles
    )

    # 4. Synchroniser avec la table FTS5
    conn.execute("INSERT INTO articles_fts(articles_fts) VALUES('rebuild')")

    print("✅ Tables créées et synchronisées")

    # 5. Démonstration de recherche avec données supplémentaires
    def rechercher_avec_details(terme):
        """Recherche avec accès aux données complètes de la table principale"""

        print(f"\n🔍 Recherche : '{terme}'")
        print("-" * 50)

        cursor = conn.execute("""
            SELECT
                a.id,
                a.titre,
                a.auteur,
                a.date_publication,
                a.vues,
                bm25(articles_fts) as score,
                snippet(articles_fts, 1, '**', '**', '...', 25) as extrait
            FROM articles_fts
            JOIN articles a ON a.id = articles_fts.rowid
            WHERE articles_fts MATCH ?
            ORDER BY bm25(articles_fts)
        """, (terme,))

        for row in cursor:
            article_id, titre, auteur, date_pub, vues, score, extrait = row
            print(f"📰 {titre} (ID: {article_id}, Score: {score:.3f})")
            print(f"   ✍️ {auteur} | 📅 {date_pub} | 👀 {vues} vues")
            print(f"   📝 {extrait}")
            print()

    # Tests de recherche
    rechercher_avec_details("Python")
    rechercher_avec_details("base données")

    return conn

conn = demo_table_fts_externe()
```

### Mise à jour automatique avec triggers

```python
def configurer_synchronisation_automatique():
    """Configure la synchronisation automatique entre table principale et FTS5"""

    # Utiliser la connexion de l'exemple précédent
    global conn

    # ⚠️ IMPORTANT : avec `content='articles'` (table externe), les triggers
    #    DOIVENT utiliser le pattern spécifique FTS5 pour DELETE et UPDATE.
    #    Un `UPDATE articles_fts SET ...` ou `DELETE FROM articles_fts` DIRECT
    #    sur une table FTS5 contentless/external corrompt l'index
    #    (`database disk image is malformed`). La doc SQLite recommande :
    #
    #    INSERT INTO articles_fts(articles_fts, rowid, ...) VALUES('delete', OLD.rowid, ...);
    #    pour supprimer une entrée d'index avant de l'ajouter à nouveau.

    # 1. Trigger INSERT — pattern standard
    conn.execute("""
        CREATE TRIGGER articles_fts_insert AFTER INSERT ON articles
        BEGIN
            INSERT INTO articles_fts(rowid, titre, contenu, auteur)
            VALUES (NEW.id, NEW.titre, NEW.contenu, NEW.auteur);
        END
    """)

    # 2. Trigger UPDATE — DELETE explicite puis INSERT
    conn.execute("""
        CREATE TRIGGER articles_fts_update AFTER UPDATE ON articles
        BEGIN
            INSERT INTO articles_fts(articles_fts, rowid, titre, contenu, auteur)
            VALUES('delete', OLD.id, OLD.titre, OLD.contenu, OLD.auteur);
            INSERT INTO articles_fts(rowid, titre, contenu, auteur)
            VALUES (NEW.id, NEW.titre, NEW.contenu, NEW.auteur);
        END
    """)

    # 3. Trigger DELETE — pattern 'delete' obligatoire en external content
    conn.execute("""
        CREATE TRIGGER articles_fts_delete AFTER DELETE ON articles
        BEGIN
            INSERT INTO articles_fts(articles_fts, rowid, titre, contenu, auteur)
            VALUES('delete', OLD.id, OLD.titre, OLD.contenu, OLD.auteur);
        END
    """)

    print("✅ Triggers de synchronisation créés")

    # Test de la synchronisation automatique
    print("\n🧪 Test de synchronisation automatique")
    print("-" * 40)

    # Insérer un nouvel article
    cursor = conn.execute("""
        INSERT INTO articles (titre, contenu, auteur)
        VALUES (?, ?, ?)
        RETURNING id
    """, ("Test auto-sync", "Cet article teste la synchronisation automatique avec FTS5.", "Test User"))

    new_id = cursor.fetchone()[0]
    print(f"📝 Nouvel article inséré (ID: {new_id})")

    # Vérifier qu'il est recherchable immédiatement
    cursor = conn.execute("""
        SELECT titre FROM articles_fts WHERE articles_fts MATCH 'auto-sync'
    """)

    if cursor.fetchone():
        print("✅ Article immédiatement disponible dans la recherche")
    else:
        print("❌ Problème de synchronisation")

    # Modifier l'article
    conn.execute("""
        UPDATE articles
        SET titre = 'Test synchronisation modifiée'
        WHERE id = ?
    """, (new_id,))

    print("✏️ Article modifié")

    # Vérifier la mise à jour dans FTS5
    cursor = conn.execute("""
        SELECT titre FROM articles_fts WHERE articles_fts MATCH 'synchronisation modifiée'
    """)

    if cursor.fetchone():
        print("✅ Modification synchronisée avec FTS5")
    else:
        print("❌ Problème de synchronisation de mise à jour")

    # Supprimer l'article de test
    conn.execute("DELETE FROM articles WHERE id = ?", (new_id,))
    print("🗑️ Article supprimé")

    # Vérifier la suppression dans FTS5
    cursor = conn.execute("""
        SELECT COUNT(*) FROM articles_fts WHERE rowid = ?
    """, (new_id,))

    count = cursor.fetchone()[0]
    if count == 0:
        print("✅ Suppression synchronisée avec FTS5")
    else:
        print("❌ Problème de synchronisation de suppression")

configurer_synchronisation_automatique()
```

## Fonctionnalités avancées

### Highlight et extraits

> ℹ️ **`snippet()` vs `highlight()`** : FTS5 propose **deux fonctions complémentaires** :  
> - **`snippet(table, col, prefix, suffix, ellipsis, nbTokens)`** : retourne un **extrait court** (snippet) — utile pour les résultats de recherche style Google. Le paramètre `nbTokens` limite la longueur (positif) ou prend toute la colonne (négatif comme `-1`).  
> - **`highlight(table, col, prefix, suffix)`** : retourne le **contenu ENTIER** de la colonne avec les termes match entourés des balises. Pas de troncature, utile pour afficher un document entier annoté.  
>
> ```sql
> -- Comparaison :
> SELECT snippet(articles_fts, 1, '<b>', '</b>', '...', 10)   -- extrait court 10 tokens
> FROM articles_fts WHERE articles_fts MATCH 'sqlite';
>
> SELECT highlight(articles_fts, 1, '<b>', '</b>')             -- contenu entier annoté
> FROM articles_fts WHERE articles_fts MATCH 'sqlite';
> ```

```python
def demo_highlight_extraits():
    """Démonstration des fonctionnalités de surlignage et d'extraits"""

    global conn

    # Ajouter un article plus long pour les tests
    long_article = """
    Python est un langage de programmation interprété, multi-paradigme et multiplateformes.
    Il favorise la programmation impérative structurée, fonctionnelle et orientée objet.
    Il est doté d'un typage dynamique fort, d'une gestion automatique de la mémoire par
    ramasse-miettes et d'un système de gestion d'exception. Python est un langage qui se
    veut simple et lisible. Il est développé depuis 1989 par Guido van Rossum et de nombreux
    bénévoles. Python est utilisé dans de nombreux domaines comme le développement web,
    l'analyse de données, l'intelligence artificielle, l'automatisation de tâches,
    et bien d'autres. Sa syntaxe claire et sa vaste bibliothèque standard en font un
    choix populaire pour les débutants comme pour les développeurs expérimentés.
    """

    conn.execute("""
        INSERT INTO articles (titre, contenu, auteur)
        VALUES (?, ?, ?)
    """, ("Python : Guide complet", long_article, "Expert Python"))

    print("🎨 DÉMONSTRATION HIGHLIGHT ET EXTRAITS")
    print("=" * 50)

    # 1. Fonction snippet() basique
    print("1️⃣ Extrait basique avec snippet()")
    cursor = conn.execute("""
        SELECT
            titre,
            snippet(articles_fts, 1, '[', ']', '...', 15) as extrait_court
        FROM articles_fts
        WHERE articles_fts MATCH 'Python programmation'
        LIMIT 1
    """)

    for titre, extrait in cursor:
        print(f"   📰 {titre}")
        print(f"   📝 {extrait}")

    # 2. Highlight avec balises HTML
    print("\n2️⃣ Highlight avec balises HTML")
    cursor = conn.execute("""
        SELECT
            titre,
            snippet(articles_fts, 1, '<mark>', '</mark>', '...', 25) as extrait_html
        FROM articles_fts
        WHERE articles_fts MATCH 'Python développement'
        LIMIT 1
    """)

    for titre, extrait in cursor:
        print(f"   📰 {titre}")
        print(f"   🌐 {extrait}")

    # 3. Extraits multiples avec fonction personnalisée
    print("\n3️⃣ Extraits multiples")

    def extraire_passages(texte, terme, nb_mots=10):
        """Fonction personnalisée pour extraire plusieurs passages"""
        mots = texte.lower().split()
        terme_lower = terme.lower()
        passages = []

        for i, mot in enumerate(mots):
            if terme_lower in mot:
                debut = max(0, i - nb_mots // 2)
                fin = min(len(mots), i + nb_mots // 2 + 1)
                passage = ' '.join(mots[debut:fin])
                # Surligner le terme
                passage = passage.replace(terme_lower, f"**{terme_lower}**")
                passages.append(passage)

        return passages[:3]  # Limiter à 3 passages

    cursor = conn.execute("""
        SELECT titre, contenu
        FROM articles a
        JOIN articles_fts ON a.id = articles_fts.rowid
        WHERE articles_fts MATCH 'Python'
        ORDER BY bm25(articles_fts)
        LIMIT 1
    """)

    for titre, contenu in cursor:
        passages = extraire_passages(contenu, "Python")
        print(f"   📰 {titre}")
        for i, passage in enumerate(passages, 1):
            print(f"   📝 Passage {i}: ...{passage}...")

    # 4. Highlight de titre
    print("\n4️⃣ Highlight dans le titre")
    cursor = conn.execute("""
        SELECT
            snippet(articles_fts, 0, '🔍', '🔍', '', -1) as titre_highlight
        FROM articles_fts
        WHERE articles_fts MATCH 'Python'
        LIMIT 3
    """)

    for (titre_highlight,) in cursor:
        print(f"   🏷️ {titre_highlight}")

demo_highlight_extraits()
```

### Recherche avec filtres avancés

```python
def demo_recherche_avancee():
    """Démonstration de recherches avancées avec filtres"""

    global conn

    # Ajouter des métadonnées pour les filtres
    conn.execute("""
        ALTER TABLE articles ADD COLUMN categorie TEXT DEFAULT 'général'
    """)

    conn.execute("""
        ALTER TABLE articles ADD COLUMN difficulte TEXT DEFAULT 'débutant'
    """)

    # Mettre à jour quelques articles avec des catégories
    mises_a_jour = [
        ("programmation", "intermédiaire", "Guide complet Python"),
        ("base-de-donnees", "débutant", "Introduction à SQLite"),
        ("web", "avancé", "Tutoriel JavaScript"),
        ("ia", "avancé", "Machine Learning Python"),
        ("base-de-donnees", "expert", "Bases de données avancées")
    ]

    for categorie, difficulte, titre_pattern in mises_a_jour:
        conn.execute("""
            UPDATE articles
            SET categorie = ?, difficulte = ?
            WHERE titre LIKE ?
        """, (categorie, difficulte, f"%{titre_pattern}%"))

    print("🔧 RECHERCHE AVANCÉE AVEC FILTRES")
    print("=" * 40)

    # Fonction de recherche avec filtres
    def recherche_avec_filtres(terme_recherche, categorie=None, difficulte=None, auteur=None):
        """Recherche avec filtres multiples"""

        sql = """
            SELECT
                a.titre,
                a.auteur,
                a.categorie,
                a.difficulte,
                bm25(articles_fts) as score,
                snippet(articles_fts, 1, '**', '**', '...', 20) as extrait
            FROM articles_fts
            JOIN articles a ON a.id = articles_fts.rowid
            WHERE articles_fts MATCH ?
        """

        params = [terme_recherche]

        if categorie:
            sql += " AND a.categorie = ?"
            params.append(categorie)

        if difficulte:
            sql += " AND a.difficulte = ?"
            params.append(difficulte)

        if auteur:
            sql += " AND a.auteur LIKE ?"
            params.append(f"%{auteur}%")

        sql += " ORDER BY bm25(articles_fts)"

        return conn.execute(sql, params).fetchall()

    # Tests de recherche avec différents filtres
    print("1️⃣ Recherche simple : 'Python'")
    resultats = recherche_avec_filtres("Python")
    for titre, auteur, cat, diff, score, extrait in resultats:
        print(f"   📰 {titre} | 🏷️ {cat} | 📊 {diff} | ✍️ {auteur}")

    print("\n2️⃣ Recherche : 'Python' + catégorie 'programmation'")
    resultats = recherche_avec_filtres("Python", categorie="programmation")
    for titre, auteur, cat, diff, score, extrait in resultats:
        print(f"   📰 {titre} | 🏷️ {cat} | 📊 {diff}")

    print("\n3️⃣ Recherche : 'base' + difficulté 'avancé'")
    resultats = recherche_avec_filtres("base", difficulte="avancé")
    for titre, auteur, cat, diff, score, extrait in resultats:
        print(f"   📰 {titre} | 🏷️ {cat} | 📊 {diff}")

    print("\n4️⃣ Recherche : articles de 'Alice'")
    resultats = recherche_avec_filtres("*", auteur="Alice")  # * pour tous les articles
    for titre, auteur, cat, diff, score, extrait in resultats:
        print(f"   📰 {titre} | ✍️ {auteur}")

demo_recherche_avancee()
```

### Recherche de suggestions et auto-complétion

```python
def demo_suggestions_autocompletion():
    """Démonstration de suggestions et auto-complétion"""

    global conn

    print("💡 SUGGESTIONS ET AUTO-COMPLÉTION")
    print("=" * 40)

    # 1. Table pour stocker les termes populaires
    conn.execute("""
        CREATE TABLE IF NOT EXISTS termes_populaires (
            terme TEXT PRIMARY KEY,
            frequence INTEGER DEFAULT 1,
            derniere_utilisation DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)

    # 2. Fonction pour enregistrer les recherches
    def enregistrer_recherche(terme):
        """Enregistre un terme de recherche pour les statistiques"""
        conn.execute("""
            INSERT INTO termes_populaires (terme, frequence, derniere_utilisation)
            VALUES (?, 1, CURRENT_TIMESTAMP)
            ON CONFLICT(terme) DO UPDATE SET
                frequence = frequence + 1,
                derniere_utilisation = CURRENT_TIMESTAMP
        """, (terme.lower(),))

    # 3. Fonction d'auto-complétion basée sur les titres
    def auto_completion(prefixe, limite=5):
        """Génère des suggestions d'auto-complétion"""

        # Recherche dans les titres d'articles
        cursor = conn.execute("""
            SELECT DISTINCT titre
            FROM articles
            WHERE titre LIKE ?
            ORDER BY titre
            LIMIT ?
        """, (f"%{prefixe}%", limite))

        suggestions_titres = [row[0] for row in cursor]

        # Recherche dans les termes populaires
        cursor = conn.execute("""
            SELECT terme
            FROM termes_populaires
            WHERE terme LIKE ?
            ORDER BY frequence DESC, derniere_utilisation DESC
            LIMIT ?
        """, (f"{prefixe}%", limite))

        suggestions_termes = [row[0] for row in cursor]

        # Combiner et dédupliquer
        toutes_suggestions = list(set(suggestions_titres + suggestions_termes))

        return toutes_suggestions[:limite]

    # 4. Recherche avec correction approximative
    def recherche_avec_correction(terme):
        """Recherche avec suggestions de correction"""

        # Essayer la recherche normale d'abord
        cursor = conn.execute("""
            SELECT COUNT(*) FROM articles_fts WHERE articles_fts MATCH ?
        """, (terme,))

        nb_resultats = cursor.fetchone()[0]

        if nb_resultats > 0:
            return terme, nb_resultats, []

        # Si aucun résultat, proposer des corrections
        corrections = []

        # Recherche par préfixe (pour fautes de frappe en fin de mot)
        if len(terme) > 3:
            prefixe = terme[:-1] + "*"
            cursor = conn.execute("""
                SELECT COUNT(*) FROM articles_fts WHERE articles_fts MATCH ?
            """, (prefixe,))

            if cursor.fetchone()[0] > 0:
                corrections.append(terme[:-1])

        # Recherche de termes similaires dans les titres
        cursor = conn.execute("""
            SELECT DISTINCT titre
            FROM articles
            WHERE titre LIKE ?
            LIMIT 3
        """, (f"%{terme[:3]}%",))

        for (titre,) in cursor:
            mots = titre.lower().split()
            for mot in mots:
                if mot.startswith(terme[:2]) and mot != terme:
                    corrections.append(mot)

        return terme, 0, list(set(corrections))[:3]

    # Tests des fonctionnalités
    print("🔍 Test d'auto-complétion")

    # Simuler quelques recherches populaires
    recherches_test = ["python", "python programmation", "base de données", "javascript", "intelligence"]
    for terme in recherches_test:
        enregistrer_recherche(terme)

    # Test auto-complétion
    prefixes_test = ["py", "base", "java", "intel"]

    for prefixe in prefixes_test:
        suggestions = auto_completion(prefixe)
        print(f"   '{prefixe}' → {suggestions}")

    print("\n🔧 Test de correction de recherche")

    # Test correction
    termes_test = ["pytho", "javascrip", "intelligence", "inexistant"]

    for terme in termes_test:
        terme_original, nb_resultats, corrections = recherche_avec_correction(terme)

        if nb_resultats > 0:
            print(f"   '{terme}' → {nb_resultats} résultat(s) trouvé(s)")
        else:
            if corrections:
                print(f"   '{terme}' → Aucun résultat. Suggestions: {corrections}")
            else:
                print(f"   '{terme}' → Aucun résultat, aucune suggestion")

demo_suggestions_autocompletion()
```

## Optimisation des performances FTS5

### Configuration et tuning

```python
def optimiser_performances_fts():
    """Optimisation des performances pour FTS5"""

    print("⚡ OPTIMISATION DES PERFORMANCES FTS5")
    print("=" * 50)

    # Nouvelle connexion pour les tests de performance
    conn = sqlite3.connect(':memory:')

    # 1. Configuration des paramètres FTS5
    print("1️⃣ Configuration des paramètres FTS5")

    # Table avec paramètres optimisés
    conn.execute("""
        CREATE VIRTUAL TABLE documents_optimises USING fts5(
            titre,
            contenu,
            -- Paramètres d'optimisation
            tokenize='porter ascii',  -- ⚠️ Stemming Porter EN ANGLAIS uniquement
            prefix='2 3 4',           -- Index de préfixes pour auto-complétion
            columnsize=0,             -- Désactiver le stockage de taille (plus rapide)
            detail=none               -- Désactiver les détails de position (plus rapide)
        )
    """)

    print("   ✅ Table avec tokenizer Porter (stemming anglais)")
    print("   ✅ Index de préfixes pour auto-complétion rapide")
    print("   ✅ Paramètres optimisés pour la vitesse")

    # 2. Test du stemming (réduction des mots à leur racine)
    print("\n2️⃣ Test du stemming Porter (EN ANGLAIS)")

    # ⚠️ IMPORTANT : Porter est un algorithme de stemming pour la LANGUE ANGLAISE
    #    Il ne fonctionne PAS bien sur le français. Vérification expérimentale :
    #    `mange` ne trouvera PAS `mangent`, ni `boivent` → `boire`.
    #    Pour le français : utiliser `unicode61` (pas de stemming) ou compiler SQLite
    #    avec un tokenizer ICU/Snowball multilingue.
    documents_test = [
        # Documents en ANGLAIS — Porter fait du stemming utile ici
        ("Programming guide", "This document covers programming and programs in depth."),
        ("Web development", "Modern web development requires careful developing."),
        ("Unit testing", "Tests allow you to test code efficiently when tested.")
    ]

    conn.executemany(
        "INSERT INTO documents_optimises (titre, contenu) VALUES (?, ?)",
        documents_test
    )

    # Test des recherches avec stemming
    recherches_stemming = [
        "program",       # Trouve "programming", "programs" (suffixes -ing, -s)
        "develop",       # Trouve "development", "developing", "developer"
        "test"           # Trouve "tests", "testing", "tested"
    ]

    for terme in recherches_stemming:
        cursor = conn.execute(
            "SELECT titre FROM documents_optimises WHERE documents_optimises MATCH ?",
            (terme,)
        )
        resultats = [row[0] for row in cursor]
        print(f"   '{terme}' → {resultats}")

    # 3. Test des préfixes rapides
    print("\n3️⃣ Test des préfixes optimisés")

    prefixes_test = ["prog*", "dev*", "test*"]

    for prefixe in prefixes_test:
        cursor = conn.execute(
            "SELECT titre FROM documents_optimises WHERE documents_optimises MATCH ?",
            (prefixe,)
        )
        resultats = [row[0] for row in cursor]
        print(f"   '{prefixe}' → {resultats}")

    # 4. Mesure de performance
    print("\n4️⃣ Mesure de performance")

    import time

    # Créer une grande quantité de données pour le test
    print("   Génération de données de test...")

    import random

    mots_base = ["python", "javascript", "html", "css", "react", "vue", "angular",
                 "node", "express", "django", "flask", "fastapi", "database",
                 "sql", "nosql", "mongodb", "postgresql", "mysql", "sqlite"]

    # Générer 1000 documents
    documents_perf = []
    for i in range(1000):
        titre = f"Document {i}: " + " ".join(random.choices(mots_base, k=3))
        contenu = " ".join(random.choices(mots_base, k=20))
        documents_perf.append((titre, contenu))

    # Test insertion
    start_time = time.time()
    conn.executemany(
        "INSERT INTO documents_optimises (titre, contenu) VALUES (?, ?)",
        documents_perf
    )
    insert_time = time.time() - start_time

    print(f"   📝 Insertion de 1000 documents: {insert_time:.3f}s")

    # Test recherche
    start_time = time.time()
    for _ in range(100):
        terme = random.choice(mots_base)
        cursor = conn.execute(
            "SELECT COUNT(*) FROM documents_optimises WHERE documents_optimises MATCH ?",
            (terme,)
        )
        cursor.fetchone()
    search_time = time.time() - start_time

    print(f"   🔍 100 recherches: {search_time:.3f}s ({search_time*10:.1f}ms par recherche)")

    conn.close()

    # 5. Recommandations d'optimisation
    print("\n💡 RECOMMANDATIONS D'OPTIMISATION")
    print("-" * 40)

    recommendations = [
        "🔧 Utilisez 'detail=none' si vous n'avez pas besoin de highlight",
        "📊 Activez 'columnsize=0' pour désactiver les statistiques de colonnes",
        "🌿 Utilisez le tokenizer 'porter' pour le stemming en anglais",
        "🔤 Configurez 'prefix' pour l'auto-complétion rapide",
        "📈 Exécutez 'OPTIMIZE' périodiquement sur les tables FTS5",
        "💾 Considérez 'content=' pour les tables externes",
        "🧹 Supprimez régulièrement les anciens documents pour maintenir les performances"
    ]

    for rec in recommendations:
        print(f"   {rec}")

optimiser_performances_fts()
```

### Maintenance et optimisation

```python
def maintenance_fts5():
    """Opérations de maintenance pour FTS5"""

    print("🔧 MAINTENANCE FTS5")
    print("=" * 30)

    conn = sqlite3.connect('exemple_fts.db')

    # Créer une table d'exemple
    conn.execute("""
        CREATE VIRTUAL TABLE IF NOT EXISTS docs USING fts5(
            titre, contenu
        )
    """)

    # Ajouter des données
    for i in range(100):
        conn.execute(
            "INSERT INTO docs (titre, contenu) VALUES (?, ?)",
            (f"Document {i}", f"Contenu du document numéro {i} avec du texte.")
        )

    # 1. Optimisation de l'index
    print("1️⃣ Optimisation de l'index FTS5")

    start_time = time.time()
    conn.execute("INSERT INTO docs(docs) VALUES('optimize')")
    optimize_time = time.time() - start_time

    print(f"   ✅ Optimisation terminée en {optimize_time:.3f}s")

    # 2. Reconstruction complète de l'index
    print("\n2️⃣ Reconstruction de l'index")

    start_time = time.time()
    conn.execute("INSERT INTO docs(docs) VALUES('rebuild')")
    rebuild_time = time.time() - start_time

    print(f"   ✅ Reconstruction terminée en {rebuild_time:.3f}s")

    # 3. Vérification de l'intégrité
    print("\n3️⃣ Vérification de l'intégrité")

    try:
        conn.execute("INSERT INTO docs(docs) VALUES('integrity-check')")
        print("   ✅ Index FTS5 intègre")
    except sqlite3.Error as e:
        print(f"   ❌ Problème d'intégrité détecté: {e}")

    # 4. Statistiques de l'index
    print("\n4️⃣ Statistiques de l'index")

    # Taille de l'index
    cursor = conn.execute("SELECT COUNT(*) FROM docs")
    nb_documents = cursor.fetchone()[0]

    # Statistiques de base
    cursor = conn.execute("PRAGMA table_info(docs)")
    colonnes = cursor.fetchall()

    print(f"   📊 Nombre de documents: {nb_documents}")
    print(f"   📋 Colonnes indexées: {len([c for c in colonnes if c[1] != 'rowid'])}")

    # 5. Nettoyage automatique
    print("\n5️⃣ Fonction de nettoyage automatique")

    def nettoyer_fts5_automatique():
        """Fonction de nettoyage automatique à exécuter périodiquement"""

        operations = [
            ("Suppression des documents vides", "DELETE FROM docs WHERE titre = '' OR contenu = ''"),
            ("Optimisation", "INSERT INTO docs(docs) VALUES('optimize')"),
            ("Mise à jour des statistiques", "ANALYZE docs")
        ]

        for description, sql in operations:
            try:
                start = time.time()
                conn.execute(sql)
                duree = time.time() - start
                print(f"      ✅ {description}: {duree:.3f}s")
            except sqlite3.Error as e:
                print(f"      ❌ {description}: {e}")

    nettoyer_fts5_automatique()

    conn.close()

maintenance_fts5()
```

## Applications pratiques complètes

### Moteur de recherche pour blog

```python
def creer_moteur_recherche_blog():
    """Création d'un moteur de recherche complet pour blog"""

    print("📰 MOTEUR DE RECHERCHE POUR BLOG")
    print("=" * 40)

    conn = sqlite3.connect('blog_search.db')

    # 1. Structure complète de la base

    # Table des articles
    conn.execute("""
        CREATE TABLE IF NOT EXISTS articles (
            id INTEGER PRIMARY KEY,
            titre TEXT NOT NULL,
            contenu TEXT NOT NULL,
            resume TEXT,
            auteur_id INTEGER,
            categorie_id INTEGER,
            tags TEXT,  -- JSON ou CSV des tags
            date_publication DATETIME DEFAULT CURRENT_TIMESTAMP,
            date_modification DATETIME DEFAULT CURRENT_TIMESTAMP,
            statut TEXT DEFAULT 'publié',
            vues INTEGER DEFAULT 0,
            likes INTEGER DEFAULT 0
        )
    """)

    # Table FTS5 optimisée
    # ⚠️ Contenu de blog en FRANÇAIS → utiliser `unicode61` (générique).
    #    `porter` ne ferait pas de stemming utile pour le français.
    conn.execute("""
        CREATE VIRTUAL TABLE IF NOT EXISTS articles_search USING fts5(
            titre,
            contenu,
            resume,
            tags,
            content='articles',
            content_rowid='id',
            tokenize='unicode61 remove_diacritics 2',
            prefix='2 3 4'
        )
    """)

    # Tables de métadonnées
    conn.execute("""
        CREATE TABLE IF NOT EXISTS auteurs (
            id INTEGER PRIMARY KEY,
            nom TEXT NOT NULL,
            email TEXT UNIQUE
        )
    """)

    conn.execute("""
        CREATE TABLE IF NOT EXISTS categories (
            id INTEGER PRIMARY KEY,
            nom TEXT NOT NULL,
            description TEXT
        )
    """)

    # Table des recherches pour analytics
    conn.execute("""
        CREATE TABLE IF NOT EXISTS recherches_log (
            id INTEGER PRIMARY KEY,
            terme TEXT NOT NULL,
            nb_resultats INTEGER,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
            ip_utilisateur TEXT,
            temps_reponse REAL
        )
    """)

    # 2. Données d'exemple

    # Auteurs
    auteurs = [
        (1, "Alice Martin", "alice@blog.com"),
        (2, "Bob Durant", "bob@blog.com"),
        (3, "Claire Dubois", "claire@blog.com")
    ]
    conn.executemany("INSERT OR REPLACE INTO auteurs VALUES (?, ?, ?)", auteurs)

    # Catégories
    categories = [
        (1, "Programmation", "Articles sur le développement logiciel"),
        (2, "Data Science", "Analyse de données et IA"),
        (3, "Web Development", "Développement web moderne")
    ]
    conn.executemany("INSERT OR REPLACE INTO categories VALUES (?, ?, ?)", categories)

    # Articles
    articles_blog = [
        (1, "Introduction à Python pour débutants",
         "Python est un langage de programmation puissant et facile à apprendre. Dans cet article, nous explorons les bases de Python, sa syntaxe simple et ses applications variées. Python est utilisé dans le développement web, la science des données, l'intelligence artificielle et l'automatisation.",
         "Guide complet pour débuter avec Python", 1, 1, "python,débutant,tutoriel,programmation",
         "2024-01-15 10:00:00", "2024-01-15 10:00:00", "publié", 1250, 45),

        (2, "Machine Learning avec scikit-learn",
         "Scikit-learn est la bibliothèque de référence pour le machine learning en Python. Cet article couvre l'installation, les algorithmes principaux comme la régression linéaire, les arbres de décision, et les réseaux de neurones. Nous verrons également l'évaluation des modèles et la validation croisée.",
         "Tutoriel complet sur scikit-learn", 2, 2, "machine-learning,scikit-learn,python,ia",
         "2024-01-20 14:30:00", "2024-01-20 14:30:00", "publié", 890, 32),

        (3, "Développement d'APIs REST avec FastAPI",
         "FastAPI est un framework moderne pour créer des APIs REST en Python. Plus rapide que Flask et Django REST, il offre une validation automatique des données, une documentation automatique avec OpenAPI, et un support natif pour l'asynchrone. Parfait pour les microservices.",
         "Guide FastAPI pour APIs modernes", 1, 3, "fastapi,api,rest,python,web",
         "2024-01-25 09:15:00", "2024-01-25 09:15:00", "publié", 650, 28),

        (4, "Analyse de données avec Pandas",
         "Pandas est la bibliothèque incontournable pour l'analyse de données en Python. Manipulation de DataFrames, nettoyage de données, jointures, agrégations, et visualisations. Nous verrons également l'intégration avec NumPy, Matplotlib et Seaborn pour des analyses complètes.",
         "Maîtriser Pandas pour l'analyse de données", 3, 2, "pandas,data-science,python,analyse",
         "2024-02-01 16:45:00", "2024-02-01 16:45:00", "publié", 1100, 55),

        (5, "Vue.js 3 : Composition API et réactivité",
         "Vue.js 3 introduit la Composition API qui révolutionne l'écriture de composants Vue. Plus flexible que l'Options API, elle permet une meilleure réutilisabilité du code et une logique plus claire. Découvrez aussi le nouveau système de réactivité et les performances améliorées.",
         "Guide complet Vue.js 3 et Composition API", 2, 3, "vuejs,javascript,frontend,web",
         "2024-02-05 11:20:00", "2024-02-05 11:20:00", "publié", 780, 38)
    ]

    conn.executemany("INSERT OR REPLACE INTO articles VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)", articles_blog)

    # Synchroniser avec FTS5
    conn.execute("INSERT INTO articles_search(articles_search) VALUES('rebuild')")

    print("✅ Base de données du blog créée avec 5 articles")

    # 3. Classe de moteur de recherche

    class MoteurRechercheBlog:
        def __init__(self, db_path):
            self.conn = sqlite3.connect(db_path)
            self.conn.row_factory = sqlite3.Row  # Pour accès par nom de colonne

        def rechercher(self, terme, categorie=None, auteur=None, limite=10, offset=0):
            """Recherche avancée avec filtres"""

            start_time = time.time()

            # Construction de la requête
            sql = """
                SELECT
                    a.id,
                    a.titre,
                    a.resume,
                    a.date_publication,
                    a.vues,
                    a.likes,
                    aut.nom as auteur_nom,
                    cat.nom as categorie_nom,
                    a.tags,
                    bm25(articles_search) as score,
                    snippet(articles_search, 1, '<mark>', '</mark>', '...', 32) as extrait
                FROM articles_search
                JOIN articles a ON a.id = articles_search.rowid
                LEFT JOIN auteurs aut ON aut.id = a.auteur_id
                LEFT JOIN categories cat ON cat.id = a.categorie_id
                WHERE articles_search MATCH ?
                AND a.statut = 'publié'
            """

            params = [terme]

            if categorie:
                sql += " AND cat.nom = ?"
                params.append(categorie)

            if auteur:
                sql += " AND aut.nom LIKE ?"
                params.append(f"%{auteur}%")

            sql += " ORDER BY bm25(articles_search) LIMIT ? OFFSET ?"
            params.extend([limite, offset])

            # Exécution
            cursor = self.conn.execute(sql, params)
            resultats = cursor.fetchall()

            # Log de la recherche
            temps_reponse = time.time() - start_time
            self._log_recherche(terme, len(resultats), temps_reponse)

            return [dict(row) for row in resultats]

        def rechercher_suggestions(self, prefixe, limite=5):
            """Auto-complétion basée sur les titres et tags"""

            # Recherche dans les titres
            cursor = self.conn.execute("""
                SELECT DISTINCT titre as suggestion, 'titre' as type
                FROM articles
                WHERE titre LIKE ? AND statut = 'publié'
                ORDER BY vues DESC
                LIMIT ?
            """, (f"%{prefixe}%", limite))

            suggestions = [dict(row) for row in cursor]

            # Recherche dans les tags
            cursor = self.conn.execute("""
                SELECT DISTINCT tags as suggestion, 'tag' as type
                FROM articles
                WHERE tags LIKE ? AND statut = 'publié'
                LIMIT ?
            """, (f"%{prefixe}%", limite))

            # Parser les tags (format CSV)
            for row in cursor:
                tags = row[0].split(',')
                for tag in tags:
                    tag = tag.strip()
                    if tag.lower().startswith(prefixe.lower()):
                        suggestions.append({'suggestion': tag, 'type': 'tag'})

            return suggestions[:limite]

        def articles_similaires(self, article_id, limite=3):
            """Trouve des articles similaires basés sur les tags et le contenu"""

            # Récupérer les tags de l'article
            cursor = self.conn.execute(
                "SELECT tags FROM articles WHERE id = ?", (article_id,)
            )
            row = cursor.fetchone()
            if not row:
                return []

            tags = row[0].split(',')

            # Construire une requête de recherche avec les tags
            termes_recherche = []
            for tag in tags[:3]:  # Limiter à 3 tags principaux
                termes_recherche.append(tag.strip())

            if not termes_recherche:
                return []

            requete = ' OR '.join(termes_recherche)

            cursor = self.conn.execute("""
                SELECT
                    a.id,
                    a.titre,
                    a.resume,
                    aut.nom as auteur_nom,
                    bm25(articles_search) as score
                FROM articles_search
                JOIN articles a ON a.id = articles_search.rowid
                LEFT JOIN auteurs aut ON aut.id = a.auteur_id
                WHERE articles_search MATCH ?
                AND a.id != ?
                AND a.statut = 'publié'
                ORDER BY bm25(articles_search)
                LIMIT ?
            """, (requete, article_id, limite))

            return [dict(row) for row in cursor]

        def statistiques_recherche(self, jours=30):
            """Statistiques des recherches des derniers jours"""

            # ⚠️ NE PAS interpoler `jours` via `.format()` dans le SQL : risque
            #    d'injection SQL si `jours` venait d'une entrée utilisateur.
            #    On construit le modificateur de date côté Python puis on le
            #    passe en paramètre lié.
            modif_date = f'-{int(jours)} days'   # int() pour sécuriser
            cursor = self.conn.execute("""
                SELECT
                    terme,
                    COUNT(*) as nb_recherches,
                    AVG(nb_resultats) as avg_resultats,
                    AVG(temps_reponse * 1000) as avg_temps_ms
                FROM recherches_log
                WHERE timestamp > datetime('now', ?)
                GROUP BY terme
                ORDER BY nb_recherches DESC
                LIMIT 10
            """, (modif_date,))

            return [dict(row) for row in cursor]

        def _log_recherche(self, terme, nb_resultats, temps_reponse):
            """Enregistre la recherche pour analytics"""
            self.conn.execute("""
                INSERT INTO recherches_log (terme, nb_resultats, temps_reponse)
                VALUES (?, ?, ?)
            """, (terme, nb_resultats, temps_reponse))
            self.conn.commit()

    # 4. Tests du moteur de recherche

    moteur = MoteurRechercheBlog('blog_search.db')

    print("\n🔍 Tests du moteur de recherche")
    print("-" * 40)

    # Test recherche simple
    resultats = moteur.rechercher("Python")
    print(f"📰 Recherche 'Python': {len(resultats)} résultat(s)")
    for article in resultats[:2]:
        print(f"   • {article['titre']} (Score: {article['score']:.3f})")
        print(f"     👤 {article['auteur_nom']} | 🏷️ {article['categorie_nom']} | 👀 {article['vues']} vues")
        print(f"     📝 {article['extrait']}")
        print()

    # Test recherche avec filtre
    resultats = moteur.rechercher("Python", categorie="Data Science")
    print(f"🔬 Recherche 'Python' + catégorie 'Data Science': {len(resultats)} résultat(s)")
    for article in resultats:
        print(f"   • {article['titre']}")

    # Test auto-complétion
    suggestions = moteur.rechercher_suggestions("Py")
    print(f"\n💡 Suggestions pour 'Py': {suggestions}")

    # Test articles similaires
    similaires = moteur.articles_similaires(1)  # Article Python
    print(f"\n🔗 Articles similaires à l'article 1:")
    for article in similaires:
        print(f"   • {article['titre']} (Score: {article['score']:.3f})")

    conn.close()

    return moteur

moteur_blog = creer_moteur_recherche_blog()
```

### Interface web simple avec Flask

```python
def creer_interface_web_recherche():
    """Création d'une interface web simple pour la recherche"""

    print("🌐 INTERFACE WEB POUR LA RECHERCHE")
    print("=" * 40)

    # Template HTML simple
    html_template = '''
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Recherche Blog</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        .search-box { width: 100%; padding: 15px; font-size: 16px; border: 2px solid #ddd; border-radius: 8px; margin-bottom: 20px; }
        .filters { margin-bottom: 20px; }
        .filter { margin-right: 15px; }
        .results { }
        .article { border: 1px solid #eee; border-radius: 8px; padding: 20px; margin-bottom: 15px; }
        .article h3 { margin-top: 0; color: #333; }
        .article-meta { color: #666; font-size: 14px; margin-bottom: 10px; }
        .article-excerpt { line-height: 1.6; }
        .highlight { background-color: #ffeb3b; padding: 2px 4px; border-radius: 3px; }
        .stats { background: #f5f5f5; padding: 15px; border-radius: 8px; margin-bottom: 20px; }
        .suggestions { background: #e3f2fd; padding: 10px; border-radius: 5px; margin-bottom: 15px; }
    </style>
</head>
<body>
    <h1>🔍 Recherche dans le Blog</h1>

    <form method="GET">
        <input type="text" name="q" class="search-box" placeholder="Rechercher des articles..."
               value="{{ query or '' }}" autofocus>

        <div class="filters">
            <select name="categorie" class="filter">
                <option value="">Toutes les catégories</option>
                <option value="Programmation" {{ 'selected' if categorie == 'Programmation' }}>Programmation</option>
                <option value="Data Science" {{ 'selected' if categorie == 'Data Science' }}>Data Science</option>
                <option value="Web Development" {{ 'selected' if categorie == 'Web Development' }}>Web Development</option>
            </select>

            <input type="submit" value="Rechercher" class="filter">
        </div>
    </form>

    {% if query %}
        <div class="stats">
            📊 <strong>{{ nb_resultats }}</strong> résultat(s) trouvé(s) pour "<strong>{{ query }}</strong>"
            {% if temps_reponse %} en {{ temps_reponse }}ms{% endif %}
        </div>

        {% if suggestions %}
        <div class="suggestions">
            💡 Suggestions:
            {% for suggestion in suggestions %}
                <a href="?q={{ suggestion.suggestion }}">{{ suggestion.suggestion }}</a>
                {% if not loop.last %} • {% endif %}
            {% endfor %}
        </div>
        {% endif %}

        <div class="results">
            {% for article in resultats %}
            <div class="article">
                <h3>{{ article.titre }}</h3>
                <div class="article-meta">
                    👤 {{ article.auteur_nom }} •
                    🏷️ {{ article.categorie_nom }} •
                    📅 {{ article.date_publication[:10] }} •
                    👀 {{ article.vues }} vues •
                    ❤️ {{ article.likes }} likes •
                    🎯 Score: {{ "%.3f"|format(article.score) }}
                </div>
                <div class="article-excerpt">
                    {{ article.extrait|safe }}
                </div>
                <div style="margin-top: 10px;">
                    🏷️ Tags: {{ article.tags }}
                </div>
            </div>
            {% endfor %}
        </div>

        {% if not resultats %}
        <div class="article">
            <h3>Aucun résultat trouvé</h3>
            <p>Essayez avec d'autres termes de recherche ou vérifiez l'orthographe.</p>
        </div>
        {% endif %}
    {% else %}
        <div class="stats">
            👋 Entrez un terme de recherche pour commencer
        </div>
    {% endif %}

    <script>
        // Auto-complétion simple (à améliorer avec AJAX)
        document.querySelector('.search-box').addEventListener('input', function(e) {
            const query = e.target.value;
            if (query.length >= 2) {
                // Ici on pourrait faire un appel AJAX pour l'auto-complétion
                console.log('Recherche suggestions pour:', query);
            }
        });
    </script>
</body>
</html>
    '''

    # Code Flask simplifié pour démonstration
    flask_code = '''
from flask import Flask, request, render_template_string  
import sqlite3  
import time  

app = Flask(__name__)

class MoteurRechercheBlog:
    # ... (code de la classe précédente) ...
    pass

@app.route('/')
def recherche():
    query = request.args.get('q', '').strip()
    categorie = request.args.get('categorie', '')

    resultats = []
    suggestions = []
    nb_resultats = 0
    temps_reponse = None

    if query:
        start_time = time.time()

        moteur = MoteurRechercheBlog('blog_search.db')

        # Recherche principale
        resultats = moteur.rechercher(
            query,
            categorie=categorie if categorie else None,
            limite=20
        )

        nb_resultats = len(resultats)

        # Suggestions si peu de résultats
        if nb_resultats < 3:
            suggestions = moteur.rechercher_suggestions(query[:3], limite=5)

        temps_reponse = int((time.time() - start_time) * 1000)  # en ms

    return render_template_string(html_template,
                                query=query,
                                categorie=categorie,
                                resultats=resultats,
                                suggestions=suggestions,
                                nb_resultats=nb_resultats,
                                temps_reponse=temps_reponse)

@app.route('/api/suggestions')
def api_suggestions():
    """API pour l'auto-complétion en AJAX"""
    prefixe = request.args.get('q', '').strip()

    if len(prefixe) < 2:
        return {'suggestions': []}

    moteur = MoteurRechercheBlog('blog_search.db')
    suggestions = moteur.rechercher_suggestions(prefixe, limite=8)

    return {'suggestions': suggestions}

@app.route('/stats')
def statistiques():
    """Page de statistiques de recherche"""
    moteur = MoteurRechercheBlog('blog_search.db')
    stats = moteur.statistiques_recherche(30)

    stats_html = '''
    <h1>📊 Statistiques de recherche (30 derniers jours)</h1>
    <table border="1" style="border-collapse: collapse; width: 100%;">
        <tr style="background: #f0f0f0;">
            <th style="padding: 10px;">Terme</th>
            <th style="padding: 10px;">Nb recherches</th>
            <th style="padding: 10px;">Résultats moyens</th>
            <th style="padding: 10px;">Temps moyen (ms)</th>
        </tr>
        {% for stat in stats %}
        <tr>
            <td style="padding: 8px;">{{ stat.terme }}</td>
            <td style="padding: 8px;">{{ stat.nb_recherches }}</td>
            <td style="padding: 8px;">{{ "%.1f"|format(stat.avg_resultats) }}</td>
            <td style="padding: 8px;">{{ "%.1f"|format(stat.avg_temps_ms) }}</td>
        </tr>
        {% endfor %}
    </table>
    <br><a href="/">← Retour à la recherche</a>
    '''

    return render_template_string(stats_html, stats=stats)

if __name__ == '__main__':
    app.run(debug=True, port=5000)
    '''

    print("💾 Code Flask généré pour l'interface web")
    print("🚀 Pour lancer : python app.py puis aller sur http://localhost:5000")

    return flask_code

interface_code = creer_interface_web_recherche()
```

## Cas d'usage avancés et extensions

### Recherche multilingue

```python
def demo_recherche_multilingue():
    """Démonstration de la recherche multilingue avec FTS5"""

    print("🌍 RECHERCHE MULTILINGUE")
    print("=" * 30)

    conn = sqlite3.connect(':memory:')

    # Table multilingue
    conn.execute("""
        CREATE VIRTUAL TABLE documents_multilingue USING fts5(
            titre,
            contenu,
            langue,
            tokenize='unicode61 remove_diacritics 2'  -- Support Unicode + suppression accents
        )
    """)

    # Données multilingues
    documents = [
        ("Guide Python", "Python est un langage de programmation facile à apprendre. Il utilise des bibliothèques comme NumPy et Pandas.", "fr"),
        ("Python Guide", "Python is an easy-to-learn programming language. It uses libraries like NumPy and Pandas.", "en"),
        ("Guía Python", "Python es un lenguaje de programación fácil de aprender. Utiliza bibliotecas como NumPy y Pandas.", "es"),
        ("Tutorial JavaScript", "JavaScript est le langage du web moderne. Il permet de créer des applications interactives.", "fr"),
        ("JavaScript Tutorial", "JavaScript is the language of modern web. It allows creating interactive applications.", "en"),
        ("Base de données", "SQLite est une base de données légère et rapide. Parfaite pour les applications mobiles.", "fr"),
        ("Database basics", "SQLite is a lightweight and fast database. Perfect for mobile applications.", "en"),
        ("Café et programmation", "Boire du café aide à programmer plus efficacement. C'est prouvé scientifiquement!", "fr")
    ]

    conn.executemany(
        "INSERT INTO documents_multilingue (titre, contenu, langue) VALUES (?, ?, ?)",
        documents
    )

    # Tests de recherche multilingue
    recherches_test = [
        ("Python", None),           # Toutes langues
        ("programming", None),      # Anglais
        ("programmation", None),    # Français
        ("aplicaciones", None),     # Espagnol
        ("base données", "fr"),     # Français spécifique
        ("database", "en")          # Anglais spécifique
    ]

    for terme, langue_filtre in recherches_test:
        print(f"\n🔍 Recherche: '{terme}'" + (f" (langue: {langue_filtre})" if langue_filtre else ""))

        sql = "SELECT titre, langue FROM documents_multilingue WHERE documents_multilingue MATCH ?"
        params = [terme]

        if langue_filtre:
            sql += " AND langue = ?"
            params.append(langue_filtre)

        cursor = conn.execute(sql, params)
        resultats = cursor.fetchall()

        for titre, langue in resultats:
            print(f"   📄 {titre} ({langue})")

    # Recherche avec synonymes simples
    print("\n🔗 Recherche avec synonymes")

    synonymes = {
        'programming': ['programmation', 'programación'],
        'database': ['base de données', 'base de datos'],
        'tutorial': ['guide', 'tutoriel', 'guía']
    }

    def recherche_avec_synonymes(terme_original):
        """Recherche étendue avec synonymes"""
        termes = [terme_original]

        # Ajouter les synonymes
        for terme_syn, liste_syn in synonymes.items():
            if terme_original.lower() == terme_syn:
                termes.extend(liste_syn)
            elif terme_original.lower() in liste_syn:
                termes.append(terme_syn)
                termes.extend([s for s in liste_syn if s != terme_original.lower()])

        # Construire requête OR
        requete = ' OR '.join(termes)

        cursor = conn.execute(
            "SELECT titre, langue FROM documents_multilingue WHERE documents_multilingue MATCH ?",
            (requete,)
        )

        return cursor.fetchall()

    # Test avec synonymes
    terme_test = "programming"
    resultats = recherche_avec_synonymes(terme_test)
    print(f"   Recherche '{terme_test}' avec synonymes:")
    for titre, langue in resultats:
        print(f"   📄 {titre} ({langue})")

    conn.close()

demo_recherche_multilingue()
```

### Indexation de fichiers

```python
def demo_indexation_fichiers():
    """Démonstration d'indexation de fichiers avec FTS5"""

    print("📁 INDEXATION DE FICHIERS")
    print("=" * 30)

    import os
    import mimetypes
    from datetime import datetime          # ← requis pour datetime.fromtimestamp() plus bas
    from pathlib import Path

    conn = sqlite3.connect('indexation_fichiers.db')

    # Table pour métadonnées de fichiers
    conn.execute("""
        CREATE TABLE IF NOT EXISTS fichiers (
            id INTEGER PRIMARY KEY,
            chemin TEXT UNIQUE NOT NULL,
            nom TEXT NOT NULL,
            taille INTEGER,
            type_mime TEXT,
            date_modification DATETIME,
            date_indexation DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)

    # Table FTS5 pour le contenu
    conn.execute("""
        CREATE VIRTUAL TABLE IF NOT EXISTS fichiers_contenu USING fts5(
            nom_fichier,
            contenu_texte,
            mots_cles,
            content='fichiers',
            content_rowid='id'
        )
    """)

    class IndexeurFichiers:
        def __init__(self, db_conn):
            self.conn = db_conn
            self.types_supportes = {
                '.txt': self._lire_texte,
                '.md': self._lire_texte,
                '.py': self._lire_code,
                '.js': self._lire_code,
                '.html': self._lire_html,
                '.css': self._lire_texte,
                '.sql': self._lire_code
            }

        def indexer_repertoire(self, chemin_repertoire):
            """Indexe récursivement un répertoire"""
            chemin = Path(chemin_repertoire)

            if not chemin.exists():
                print(f"❌ Répertoire inexistant: {chemin}")
                return

            fichiers_indexes = 0

            for fichier in chemin.rglob('*'):
                if fichier.is_file():
                    if self._indexer_fichier(fichier):
                        fichiers_indexes += 1

            print(f"✅ {fichiers_indexes} fichiers indexés")

        def _indexer_fichier(self, chemin_fichier):
            """Indexe un fichier individuel"""
            try:
                # Vérifier si déjà indexé et pas modifié
                stat = chemin_fichier.stat()
                date_modif = datetime.fromtimestamp(stat.st_mtime)

                cursor = self.conn.execute(
                    "SELECT date_modification FROM fichiers WHERE chemin = ?",
                    (str(chemin_fichier),)
                )

                row = cursor.fetchone()
                if row and row[0] >= date_modif.isoformat():
                    return False  # Déjà à jour

                # Métadonnées du fichier
                nom = chemin_fichier.name
                taille = stat.st_size
                type_mime = mimetypes.guess_type(str(chemin_fichier))[0]

                # Lire le contenu si type supporté
                extension = chemin_fichier.suffix.lower()
                contenu = ""
                mots_cles = []

                if extension in self.types_supportes:
                    contenu, mots_cles = self.types_supportes[extension](chemin_fichier)

                # Sauvegarder en base
                cursor = self.conn.execute("""
                    INSERT OR REPLACE INTO fichiers
                    (chemin, nom, taille, type_mime, date_modification)
                    VALUES (?, ?, ?, ?, ?)
                    RETURNING id
                """, (str(chemin_fichier), nom, taille, type_mime, date_modif.isoformat()))

                fichier_id = cursor.fetchone()[0]

                # Indexer le contenu
                if contenu:
                    self.conn.execute("""
                        INSERT OR REPLACE INTO fichiers_contenu
                        (rowid, nom_fichier, contenu_texte, mots_cles)
                        VALUES (?, ?, ?, ?)
                    """, (fichier_id, nom, contenu, ' '.join(mots_cles)))

                self.conn.commit()
                return True

            except Exception as e:
                print(f"❌ Erreur indexation {chemin_fichier}: {e}")
                return False

        def _lire_texte(self, chemin):
            """Lit un fichier texte simple"""
            try:
                with open(chemin, 'r', encoding='utf-8') as f:
                    contenu = f.read()

                # Extraire mots-clés simples (mots de plus de 3 caractères)
                mots = contenu.lower().split()
                mots_cles = list(set([m for m in mots if len(m) > 3 and m.isalpha()]))

                return contenu, mots_cles[:20]  # Limiter à 20 mots-clés

            except UnicodeDecodeError:
                return "", []

        def _lire_code(self, chemin):
            """Lit un fichier de code"""
            contenu, mots_cles = self._lire_texte(chemin)

            # Ajouter des mots-clés spécifiques au code
            extension = chemin.suffix.lower()
            if extension == '.py':
                mots_cles.extend(['python', 'script'])
            elif extension == '.js':
                mots_cles.extend(['javascript', 'script'])
            elif extension == '.sql':
                mots_cles.extend(['sql', 'database', 'query'])

            return contenu, mots_cles

        def _lire_html(self, chemin):
            """Lit un fichier HTML en extrayant le texte"""
            try:
                with open(chemin, 'r', encoding='utf-8') as f:
                    contenu_html = f.read()

                # Extraction basique du texte (sans BeautifulSoup pour simplicité)
                import re

                # Supprimer les balises
                texte = re.sub(r'<[^>]+>', ' ', contenu_html)
                # Supprimer les entités HTML courantes
                texte = texte.replace('&nbsp;', ' ').replace('&amp;', '&')
                # Nettoyer les espaces
                texte = re.sub(r'\s+', ' ', texte).strip()

                mots = texte.lower().split()
                mots_cles = list(set([m for m in mots if len(m) > 3 and m.isalpha()]))

                return texte, mots_cles[:20]

            except Exception:
                return "", []

        def rechercher_fichiers(self, terme, type_mime=None, limite=10):
            """Recherche dans les fichiers indexés"""

            sql = """
                SELECT
                    f.chemin,
                    f.nom,
                    f.taille,
                    f.type_mime,
                    bm25(fichiers_contenu) as score,
                    snippet(fichiers_contenu, 1, '**', '**', '...', 32) as extrait
                FROM fichiers_contenu fc
                JOIN fichiers f ON f.id = fc.rowid
                WHERE fichiers_contenu MATCH ?
            """

            params = [terme]

            if type_mime:
                sql += " AND f.type_mime LIKE ?"
                params.append(f"%{type_mime}%")

            sql += " ORDER BY bm25(fichiers_contenu) LIMIT ?"
            params.append(limite)

            cursor = self.conn.execute(sql, params)
            return cursor.fetchall()

    # Démonstration avec des fichiers de test

    # Créer quelques fichiers de test en mémoire (simulation)
    print("📝 Simulation d'indexation de fichiers")

    # Simuler l'indexation en insérant directement des données
    fichiers_test = [
        ("readme.txt", "Ceci est un fichier README pour le projet Python. Il contient des instructions d'installation et d'utilisation.", "text/plain"),
        ("script.py", "import sqlite3\ndef main():\n    conn = sqlite3.connect('test.db')\n    print('Hello World')", "text/x-python"),
        ("index.html", "<html><head><title>Mon Site</title></head><body><h1>Bienvenue</h1><p>Site de démonstration</p></body></html>", "text/html"),
        ("styles.css", "body { font-family: Arial; } h1 { color: blue; }", "text/css"),
        ("data.sql", "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT);", "application/sql")
    ]

    for i, (nom, contenu, mime_type) in enumerate(fichiers_test, 1):
        # Insérer métadonnées
        cursor = conn.execute("""
            INSERT INTO fichiers (id, chemin, nom, taille, type_mime, date_modification)
            VALUES (?, ?, ?, ?, ?, datetime('now'))
        """, (i, f"/demo/{nom}", nom, len(contenu), mime_type))

        # Indexer contenu
        conn.execute("""
            INSERT INTO fichiers_contenu (rowid, nom_fichier, contenu_texte, mots_cles)
            VALUES (?, ?, ?, ?)
        """, (i, nom, contenu, nom.replace('.', ' ')))

    conn.commit()

    # Tests de recherche
    indexeur = IndexeurFichiers(conn)

    recherches_test = [
        ("Python", None),
        ("HTML", None),
        ("site", None),
        ("database", None),
        ("sqlite", "python")
    ]

    for terme, type_filtre in recherches_test:
        print(f"\n🔍 Recherche: '{terme}'" + (f" (type: {type_filtre})" if type_filtre else ""))

        resultats = indexeur.rechercher_fichiers(terme, type_filtre)

        for chemin, nom, taille, mime_type, score, extrait in resultats:
            print(f"   📄 {nom} ({mime_type}) - {taille} bytes")
            print(f"      📁 {chemin}")
            print(f"      📝 {extrait}")
            print(f"      🎯 Score: {score:.3f}")

    conn.close()

demo_indexation_fichiers()
```

## Performance et monitoring avancés

### Benchmarking et optimisation

```python
def benchmark_fts5():
    """Benchmark complet des performances FTS5"""

    print("⚡ BENCHMARK FTS5")
    print("=" * 20)

    import time
    import random
    import string

    # Préparer les données de test
    def generer_document(taille_mots=100):
        """Génère un document aléatoire"""
        mots_base = ["python", "javascript", "html", "css", "react", "vue", "django",
                     "flask", "database", "sql", "nosql", "mongodb", "postgresql",
                     "development", "programming", "coding", "software", "application",
                     "framework", "library", "tutorial", "guide", "example"]

        titre = " ".join(random.choices(mots_base, k=4))
        contenu = " ".join(random.choices(mots_base, k=taille_mots))

        return titre, contenu

    # Tests avec différentes configurations
    configurations = [
        ("Basique", "CREATE VIRTUAL TABLE docs_basic USING fts5(titre, contenu)"),
        ("Optimisé", "CREATE VIRTUAL TABLE docs_opt USING fts5(titre, contenu, tokenize='porter ascii', prefix='2 3', detail=none)"),
        ("Externe", "CREATE VIRTUAL TABLE docs_ext USING fts5(titre, contenu, content='docs_source', content_rowid='id')")
    ]

    resultats_bench = {}

    for nom_config, sql_creation in configurations:
        print(f"\n🧪 Test configuration: {nom_config}")

        conn = sqlite3.connect(':memory:')

        # Créer table source pour configuration externe
        if "content=" in sql_creation:
            conn.execute("CREATE TABLE docs_source (id INTEGER PRIMARY KEY, titre TEXT, contenu TEXT)")

        # Créer table FTS5
        conn.execute(sql_creation)

        # Extraire le nom de la table FTS5 du SQL : `CREATE VIRTUAL TABLE <nom> USING fts5(...)`.
        # Le nom est en position [3] dans le split par espaces.
        table_name = sql_creation.split()[3]

        # Test d'insertion
        print("   📝 Test insertion...")
        nb_docs = 1000
        documents = [generer_document() for _ in range(nb_docs)]

        start_time = time.time()

        if "content=" in sql_creation:
            # Insertion pour table externe
            conn.executemany("INSERT INTO docs_source (titre, contenu) VALUES (?, ?)", documents)
            conn.execute(f"INSERT INTO {table_name}({table_name}) VALUES('rebuild')")
        else:
            # Insertion directe
            conn.executemany(f"INSERT INTO {table_name} (titre, contenu) VALUES (?, ?)", documents)

        temps_insertion = time.time() - start_time
        print(f"      ⏱️ {nb_docs} documents en {temps_insertion:.3f}s ({nb_docs/temps_insertion:.0f} docs/s)")

        # Test de recherche
        print("   🔍 Test recherche...")
        termes_recherche = ["python", "javascript", "database", "development", "tutorial"]

        start_time = time.time()
        nb_recherches = 100

        for _ in range(nb_recherches):
            terme = random.choice(termes_recherche)
            cursor = conn.execute(f"SELECT COUNT(*) FROM {table_name} WHERE {table_name} MATCH ?", (terme,))
            cursor.fetchone()

        temps_recherche = time.time() - start_time
        print(f"      ⏱️ {nb_recherches} recherches en {temps_recherche:.3f}s ({temps_recherche*1000/nb_recherches:.1f}ms par recherche)")

        # Test recherche complexe
        print("   🔧 Test recherche complexe...")
        start_time = time.time()

        cursor = conn.execute(f"""
            SELECT titre, bm25({table_name}) as score,
                   snippet({table_name}, 1, '<b>', '</b>', '...', 20) as extrait
            FROM {table_name}
            WHERE {table_name} MATCH 'python OR javascript'
            ORDER BY bm25({table_name})
            LIMIT 10
        """)

        resultats = cursor.fetchall()
        temps_complexe = time.time() - start_time

        print(f"      ⏱️ Recherche complexe: {temps_complexe:.3f}s, {len(resultats)} résultats")

        # Stocker résultats
        resultats_bench[nom_config] = {
            'insertion': temps_insertion,
            'recherche_simple': temps_recherche,
            'recherche_complexe': temps_complexe,
            'docs_par_sec': nb_docs / temps_insertion,
            'ms_par_recherche': temps_recherche * 1000 / nb_recherches
        }

        conn.close()

    # Résumé comparatif
    print(f"\n📊 RÉSUMÉ COMPARATIF")
    print("=" * 50)

    print(f"{'Configuration':<15} {'Docs/s':<10} {'ms/recherche':<15} {'Score global':<12}")
    print("-" * 55)

    for nom, resultats in resultats_bench.items():
        score_global = (resultats['docs_par_sec'] / 100) + (10 / resultats['ms_par_recherche'])
        print(f"{nom:<15} {resultats['docs_par_sec']:<10.0f} {resultats['ms_par_recherche']:<15.1f} {score_global:<12.2f}")

benchmark_fts5()
```

## Récapitulatif et conclusion

### Ce que vous avez appris sur FTS5

```python
def conclusion_fts5():
    """Conclusion complète de la section FTS5"""

    print("🎯 CONCLUSION - RECHERCHE PLEIN TEXTE FTS5")
    print("=" * 60)

    competences_acquises = {
        "🔍 Fondamentaux FTS5": [
            "✅ Comprendre les avantages de la recherche plein texte",
            "✅ Créer et configurer des tables FTS5",
            "✅ Maîtriser la syntaxe de recherche avancée",
            "✅ Utiliser les opérateurs booléens et la proximité"
        ],

        "⚙️ Configuration et optimisation": [
            "✅ Choisir les bons tokenizers (porter, unicode61)",
            "✅ Configurer les préfixes pour l'auto-complétion",
            "✅ Optimiser avec detail=none et columnsize=0",
            "✅ Maintenir les performances avec OPTIMIZE"
        ],

        "🎨 Fonctionnalités avancées": [
            "✅ Générer des extraits avec snippet()",
            "✅ Surligner les termes recherchés",
            "✅ Classer par pertinence avec bm25()",
            "✅ Implémenter des suggestions intelligentes"
        ],

        "🏗️ Architecture et intégration": [
            "✅ Synchroniser avec des tables externes",
            "✅ Mettre en place des triggers automatiques",
            "✅ Créer des APIs de recherche robustes",
            "✅ Développer des interfaces web complètes"
        ],

        "🌍 Cas d'usage avancés": [
            "✅ Recherche multilingue et accents",
            "✅ Indexation de fichiers et documents",
            "✅ Moteurs de recherche pour blogs/sites",
            "✅ Systèmes de recommandation"
        ],

        "📊 Performance et monitoring": [
            "✅ Benchmarker les performances FTS5",
            "✅ Optimiser les requêtes lentes",
            "✅ Surveiller l'utilisation et les statistiques",
            "✅ Maintenir les index en production"
        ]
    }

    for categorie, competences in competences_acquises.items():
        print(f"\n{categorie}")
        for competence in competences:
            print(f"  {competence}")

    print(f"\n🏆 NIVEAU ATTEINT")
    print("Vous maîtrisez maintenant FTS5 de niveau professionnel !")

    # Exemples d'applications réelles
    print(f"\n🚀 APPLICATIONS RÉELLES POSSIBLES")
    applications = [
        "📰 Moteur de recherche pour site web/blog",
        "📚 Système de recherche documentaire",
        "📧 Recherche dans emails/messages",
        "🎵 Recherche dans métadonnées musicales",
        "📖 Base de connaissances d'entreprise",
        "🛒 Recherche produits e-commerce",
        "📋 Recherche dans logs applicatifs",
        "🎯 Système de recommandation de contenu"
    ]

    for app in applications:
        print(f"  {app}")

    # Code template de démarrage
    print(f"\n📝 TEMPLATE DE DÉMARRAGE RAPIDE")
    template = '''
-- 1. Créer la structure
CREATE TABLE articles (
    id INTEGER PRIMARY KEY,
    titre TEXT NOT NULL,
    contenu TEXT NOT NULL,
    date_creation DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE VIRTUAL TABLE articles_search USING fts5(
    titre, contenu,
    content='articles',
    content_rowid='id',
    -- ⚠️ Tokenizer : choisir selon la langue principale.
    --   `unicode61 remove_diacritics 2` : générique, multilingue, gère les accents (RECOMMANDÉ pour le français)
    --   `porter ascii` : stemming spécifique à l'anglais (cherche "running" trouve "run")
    tokenize='unicode61 remove_diacritics 2',
    prefix='2 3'
);

-- 2. Triggers de synchronisation (pattern OBLIGATOIRE pour external content)
CREATE TRIGGER articles_insert AFTER INSERT ON articles  
BEGIN  
    INSERT INTO articles_search(rowid, titre, contenu)
    VALUES (NEW.id, NEW.titre, NEW.contenu);
END;

CREATE TRIGGER articles_update AFTER UPDATE ON articles  
BEGIN  
    -- En external content : 'delete' explicite puis nouvel INSERT
    INSERT INTO articles_search(articles_search, rowid, titre, contenu)
    VALUES('delete', OLD.id, OLD.titre, OLD.contenu);
    INSERT INTO articles_search(rowid, titre, contenu)
    VALUES (NEW.id, NEW.titre, NEW.contenu);
END;

CREATE TRIGGER articles_delete AFTER DELETE ON articles  
BEGIN  
    INSERT INTO articles_search(articles_search, rowid, titre, contenu)
    VALUES('delete', OLD.id, OLD.titre, OLD.contenu);
END;

-- 3. Recherche avec classement
SELECT
    a.titre,
    bm25(articles_search) as score,
    snippet(articles_search, 1, '<mark>', '</mark>', '...', 32) as extrait
FROM articles_search s  
JOIN articles a ON a.id = s.rowid  
WHERE articles_search MATCH ?  
ORDER BY bm25(articles_search)  
LIMIT 10;  
    '''

    print(template)

    print(f"\n💡 CONSEILS POUR LA SUITE")
    conseils = [
        "🧪 Testez FTS5 sur vos propres données",
        "📈 Mesurez les performances vs recherche LIKE",
        "🔧 Expérimentez avec différents tokenizers",
        "🌐 Intégrez dans une application web réelle",
        "📊 Surveillez les métriques en production",
        "👥 Partagez vos expériences avec la communauté"
    ]

    for conseil in conseils:
        print(f"  {conseil}")

conclusion_fts5()
```

## Guide de révision rapide

```python
def guide_revision():
    """Guide de révision rapide de tous les concepts"""

    print("📖 GUIDE DE RÉVISION RAPIDE")
    print("=" * 35)

    revision_map = {
        "🎯 UDF (6.1)": {
            "concept_cle": "Fonctions SQL personnalisées intégrées",
            "code_exemple": """
# Fonction scalaire simple
def ma_fonction(param):
    return param * 2

conn.create_function("double", 1, ma_fonction)
# Usage: SELECT double(5) -> 10
            """,
            "points_cles": [
                "create_function() pour Python",
                "Gestion des NULL et erreurs",
                "Fonctions d'agrégation avec classes"
            ]
        },

        "🔧 Extensions (6.2)": {
            "concept_cle": "Modules C/Python étendant SQLite",
            "code_exemple": """
# Extension Python avec APSW
class MonExtension:
    def ma_methode(self, param):
        return param.upper()

conn.create_scalar_function("upper_custom", ext.ma_methode)
            """,
            "points_cles": [
                "Extensions C pour performance",
                "Tables virtuelles",
                "Chargement dynamique"
            ]
        },

        "⚡ Transactions (6.3)": {
            "concept_cle": "Contrôle avancé des transactions",
            "code_exemple": """
# Transaction avec savepoint
BEGIN IMMEDIATE;
  INSERT INTO table1 VALUES (...);
  SAVEPOINT etape1;
    INSERT INTO table2 VALUES (...);
  ROLLBACK TO etape1;  -- Annule seulement table2
COMMIT;
            """,
            "points_cles": [
                "DEFERRED/IMMEDIATE/EXCLUSIVE",
                "SAVEPOINT pour rollback partiel",
                "Mode WAL pour concurrence"
            ]
        },

        "💾 Backup (6.4)": {
            "concept_cle": "Sauvegarde cohérente avec l'API native SQLite",
            "code_exemple": """
# Sauvegarde avec API native (sûre même si la base est ouverte)
source = sqlite3.connect('source.db')  
backup = sqlite3.connect('backup.db')  

source.backup(backup, progress=callback)  
backup.close()  
source.close()  
            """,
            "points_cles": [
                "API backup native SQLite (jamais `cp`)",
                "Rotation et compression gzip",
                "Litestream pour réplication continue"
            ]
        },

        "🛡️ Gestion erreurs (6.5)": {
            "concept_cle": "Hiérarchie d'exceptions + pattern Repository",
            "code_exemple": """
# Pattern recommandé
try:
    rows = conn.execute("INSERT INTO users (email) VALUES (?) RETURNING id", (email,)).fetchall()
    conn.commit()
    return rows[0][0]
except sqlite3.IntegrityError as e:
    conn.rollback()
    if "UNIQUE constraint failed" in str(e):
        raise ConflictError(f"Email '{email}' déjà utilisé") from e
    raise
except sqlite3.Error as e:
    conn.rollback()
    raise RepositoryError(f"Erreur DB : {e}") from e
            """,
            "points_cles": [
                "Hiérarchie : ordre spécifique → général dans except",
                "Exceptions métier : ValidationError / NotFoundError / ConflictError",
                "Retry avec backoff exponentiel (delay * 2**tentative)"
            ]
        },

        "🔍 FTS5 (6.6)": {
            "concept_cle": "Recherche plein texte avancée",
            "code_exemple": """
# Table FTS5 optimisée
# tokenize : 'unicode61 remove_diacritics 2' (multilingue/FR) ou 'porter ascii' (EN avec stemming)
CREATE VIRTUAL TABLE docs USING fts5(
    titre, contenu,
    tokenize='unicode61 remove_diacritics 2',
    prefix='2 3'
);

# Recherche avec ranking
SELECT *, bm25(docs) as score,
       snippet(docs, 1, '<b>', '</b>', '...', 32)
FROM docs WHERE docs MATCH 'python OR javascript'  
ORDER BY bm25(docs);  
            """,
            "points_cles": [
                "Configuration tokenizers",
                "Ranking BM25 et snippet()",
                "Auto-complétion avec préfixes"
            ]
        }
    }

    for section, details in revision_map.items():
        print(f"\n{section}")
        print(f"💡 {details['concept_cle']}")
        print(f"📝 Exemple code:{details['code_exemple']}")
        print("🎯 Points clés:")
        for point in details['points_cles']:
            print(f"   • {point}")
        print("-" * 50)

guide_revision()
```

## Conclusion finale

🎉 **Félicitations !** Vous maîtrisez maintenant FTS5 et l'ensemble de la programmation avancée SQLite (chapitre 6).

**Compétences acquises** :
- ✅ UDF (scalaires, agrégats, fenêtrage Python 3.11+)
- ✅ Extensions SQLite (chargement dynamique, écriture en C)
- ✅ Transactions et modes de verrouillage (DEFERRED/IMMEDIATE/EXCLUSIVE)
- ✅ Sauvegarde et restauration (Backup API)
- ✅ Gestion d'erreurs robuste (hiérarchie, retry, pattern Repository)
- ✅ Recherche plein texte FTS5 (bm25, snippet, tokenizers, optimisation)

**Pour aller plus loin** :
- 📖 [Documentation officielle SQLite](https://sqlite.org/docs.html)
- 🔍 [FTS5 reference](https://sqlite.org/fts5.html)
- 💾 [Backup API](https://sqlite.org/backup.html)
- 🛠️ [sqlite-utils](https://sqlite-utils.datasette.io/) — outil Python avancé
- ☁️ [Litestream](https://litestream.io/) — réplication continue vers S3/GCS/B2

Vous avez maintenant toutes les compétences pour créer des moteurs de recherche, des applications web robustes, et des systèmes de base de données de niveau professionnel avec SQLite. Le chapitre suivant abordera l'**intégration de SQLite dans des applications** (Python sqlite3, C/Java/JS, ORM, APIs REST, synchronisation).

**Continuez à pratiquer, innover et partager vos connaissances !** 🚀

⏭️ [Module 7 : Intégration et APIs](/07-integration-apis/README.md)
