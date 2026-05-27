🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.3 : ORM et SQLite : bonnes pratiques

## Introduction

Un ORM (Object-Relational Mapping) est comme un traducteur entre votre langage de programmation et votre base de données. Au lieu d'écrire du SQL directement, vous manipulez des objets dans votre code, et l'ORM se charge de traduire ces opérations en requêtes SQL. C'est particulièrement utile pour les débutants car cela simplifie grandement le travail avec les bases de données.

Imaginez que vous voulez ajouter une personne à votre base de données :
- **Sans ORM** : `INSERT INTO personnes (nom, age) VALUES ('Alice', 25)`
- **Avec ORM** : `personne = new Personne('Alice', 25); personne.save()`

Cette section vous guidera dans l'utilisation des ORM les plus populaires avec SQLite, en vous donnant les bonnes pratiques pour éviter les pièges courants.

## Qu'est-ce qu'un ORM ?

### Concept de base

Un ORM crée une correspondance entre :
- **Les tables** de votre base de données ↔ **Les classes** de votre code
- **Les lignes** de vos tables ↔ **Les instances d'objets**
- **Les colonnes** ↔ **Les propriétés d'objets**

### Avantages des ORM

**Pour les débutants** :
- ✅ **Simplicité** : Plus besoin d'apprendre le SQL immédiatement
- ✅ **Sécurité** : Protection automatique contre les injections SQL
- ✅ **Productivité** : Développement plus rapide
- ✅ **Validation** : Vérification automatique des données

**Pour tous** :
- ✅ **Portabilité** : Changement de base de données plus facile
- ✅ **Maintenance** : Code plus lisible et organisé
- ✅ **Relations** : Gestion simplifiée des liens entre tables

### Inconvénients à connaître

- ❌ **Performance** : Peut être plus lent que le SQL direct
- ❌ **Complexité cachée** : Peut générer des requêtes inefficaces
- ❌ **Courbe d'apprentissage** : Chaque ORM a ses spécificités
- ❌ **Contrôle limité** : Moins de flexibilité pour les requêtes complexes

## ORM populaires par langage

### Python : SQLAlchemy

SQLAlchemy est l'ORM le plus populaire en Python, très puissant mais avec une courbe d'apprentissage modérée.

#### Installation
```bash
pip install sqlalchemy
```

#### Exemple de base

```python
# SQLAlchemy 2.0+ : style moderne avec DeclarativeBase et select()
from sqlalchemy import create_engine, String, Integer, DateTime, select  
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, sessionmaker  
from datetime import datetime, timezone  

# Configuration de base
class Base(DeclarativeBase):
    pass

engine = create_engine('sqlite:///exemple.db', echo=True)  # echo=True pour voir les requêtes SQL  
Session = sessionmaker(bind=engine)  

# Définition d'un modèle (typage Python moderne)
class Personne(Base):
    __tablename__ = 'personnes'

    id: Mapped[int] = mapped_column(primary_key=True)
    nom: Mapped[str] = mapped_column(String(100))
    age: Mapped[int | None] = mapped_column(default=None)
    email: Mapped[str] = mapped_column(String(120), unique=True)
    date_creation: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(timezone.utc)
    )

    def __repr__(self):
        return f"<Personne(nom='{self.nom}', age={self.age})>"

# Créer les tables
Base.metadata.create_all(engine)

# Utilisation de base (style 2.0 avec context manager)
with Session() as session:
    # Créer une nouvelle personne
    nouvelle_personne = Personne(nom='Alice Martin', age=25, email='alice@email.com')
    session.add(nouvelle_personne)
    session.commit()

    # Rechercher des personnes (style 2.0 avec select())
    personnes = session.execute(select(Personne)).scalars().all()
    for personne in personnes:
        print(personne)

    # Recherche avec filtre
    alice = session.execute(
        select(Personne).filter_by(nom='Alice Martin')
    ).scalar_one_or_none()
    print(f"Alice trouvée : {alice}")

    # Mise à jour
    if alice:
        alice.age = 26
        session.commit()
# La connexion est fermée automatiquement à la sortie du `with`
```

> 💡 **SQLAlchemy 1.x → 2.0** : depuis SQLAlchemy 2.0 (sortie en 2023), le style « legacy » `session.query(Personne).filter_by(...).all()` reste supporté mais le **style 2.0 recommandé** utilise `session.execute(select(...)).scalars().all()`. Les deux fonctionnent ; le 2.0 est plus explicite, mieux typé, et compatible avec `async`.  
>  
> ⚠️ **`declarative_base()` et `datetime.utcnow()`** : ces deux APIs sont **dépréciées** :  
> - `from sqlalchemy.ext.declarative import declarative_base` → utilisez la classe `DeclarativeBase` (importée depuis `sqlalchemy.orm`).  
> - `datetime.utcnow()` est déprécié depuis Python 3.12 → utilisez `datetime.now(timezone.utc)` qui est *timezone-aware*.

#### Gestion des relations avec SQLAlchemy (style 2.0)

```python
from typing import List, Optional  
from sqlalchemy import ForeignKey, String  
from sqlalchemy.orm import Mapped, mapped_column, relationship, DeclarativeBase  

class Base(DeclarativeBase):
    pass

class Utilisateur(Base):
    __tablename__ = 'utilisateurs'

    id: Mapped[int] = mapped_column(primary_key=True)
    nom: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(120), unique=True)

    # Relation vers les articles (one-to-many) — typage explicite
    articles: Mapped[List["Article"]] = relationship(back_populates="auteur")

class Article(Base):
    __tablename__ = 'articles'

    id: Mapped[int] = mapped_column(primary_key=True)
    titre: Mapped[str] = mapped_column(String(200))
    contenu: Mapped[Optional[str]] = mapped_column(String(5000))
    auteur_id: Mapped[int] = mapped_column(ForeignKey('utilisateurs.id'))

    # Relation inverse (many-to-one)
    auteur: Mapped["Utilisateur"] = relationship(back_populates="articles")

# Utilisation des relations
with Session() as session:
    # Créer un utilisateur et des articles en une seule opération
    utilisateur = Utilisateur(nom='Bob Dupont', email='bob@email.com')
    utilisateur.articles = [
        Article(titre='Mon premier article', contenu='Contenu...'),
        Article(titre='Mon second article', contenu='Autre contenu...'),
    ]

    session.add(utilisateur)  # cascade automatique sur articles
    session.commit()

    # Accéder aux relations
    print(f"Articles de {utilisateur.nom}:")
    for article in utilisateur.articles:
        print(f"  - {article.titre}")

    print(f"Auteur de '{utilisateur.articles[0].titre}': {utilisateur.articles[0].auteur.nom}")
```

> 💡 **`Mapped[List[...]]` vs `relationship("...")`** : depuis SQLAlchemy 2.0, le typage `Mapped[List["Article"]]` permet à `relationship` d'inférer automatiquement la cible (`Article`), le type de collection (`list`), et l'option `uselist`. Plus besoin de répéter `relationship("Article", uselist=True, ...)`.

### JavaScript/Node.js : Sequelize

Sequelize est un ORM mature et feature-complete pour Node.js.

#### Installation
```bash
npm install sequelize sqlite3
```

#### Configuration et modèles

```javascript
const { Sequelize, DataTypes } = require('sequelize');

// Configuration de la connexion
const sequelize = new Sequelize({
    dialect: 'sqlite',
    storage: 'exemple.db',
    logging: console.log // Afficher les requêtes SQL
});

// Définition du modèle Personne
const Personne = sequelize.define('Personne', {
    id: {
        type: DataTypes.INTEGER,
        primaryKey: true,
        autoIncrement: true
    },
    nom: {
        type: DataTypes.STRING(100),
        allowNull: false
    },
    age: {
        type: DataTypes.INTEGER
    },
    email: {
        type: DataTypes.STRING(120),
        unique: true,
        validate: {
            isEmail: true
        }
    }
}, {
    tableName: 'personnes',
    timestamps: true // Ajoute automatiquement createdAt et updatedAt
});

// Fonction d'initialisation
async function initialiser() {
    try {
        // Tester la connexion
        await sequelize.authenticate();
        console.log('✅ Connexion réussie à SQLite');

        // Synchroniser les modèles (créer les tables)
        await sequelize.sync({ force: false }); // force: true supprime et recrée les tables
        console.log('📋 Tables synchronisées');

    } catch (error) {
        console.error('❌ Erreur connexion :', error);
    }
}

// Fonctions CRUD
async function exemplesCRUD() {
    // CREATE - Créer une nouvelle personne
    const nouvellePersonne = await Personne.create({
        nom: 'Alice Martin',
        age: 25,
        email: 'alice.martin@email.com'
    });
    console.log('✅ Personne créée :', nouvellePersonne.toJSON());

    // CREATE - Créer plusieurs personnes
    const personnes = await Personne.bulkCreate([
        { nom: 'Bob Dupont', age: 30, email: 'bob.dupont@email.com' },
        { nom: 'Claire Rousseau', age: 28, email: 'claire.rousseau@email.com' }
    ]);
    console.log(`✅ ${personnes.length} personnes créées`);

    // READ - Lire toutes les personnes
    const toutesPersonnes = await Personne.findAll();
    console.log('\n📋 Toutes les personnes :');
    toutesPersonnes.forEach(p => {
        console.log(`  ${p.nom} (${p.age} ans) - ${p.email}`);
    });

    // READ - Recherche avec conditions
    const personnesJeunes = await Personne.findAll({
        where: {
            age: {
                [Sequelize.Op.lt]: 30 // moins de 30 ans
            }
        },
        order: [['nom', 'ASC']]
    });

    console.log('\n👶 Personnes de moins de 30 ans :');
    personnesJeunes.forEach(p => console.log(`  ${p.nom}`));

    // READ - Trouver une personne spécifique
    const alice = await Personne.findOne({
        where: { email: 'alice.martin@email.com' }
    });

    if (alice) {
        console.log(`\n🔍 Personne trouvée : ${alice.nom}`);

        // UPDATE - Mettre à jour
        alice.age = 26;
        await alice.save();
        console.log(`✅ Âge mis à jour : ${alice.age}`);
    }

    // DELETE - Supprimer (soft delete si paranoid: true)
    const personneASupprimer = await Personne.findOne({
        where: { nom: 'Bob Dupont' }
    });

    if (personneASupprimer) {
        await personneASupprimer.destroy();
        console.log('🗑️ Personne supprimée');
    }

    // Compter les personnes restantes
    const nombre = await Personne.count();
    console.log(`\n📊 Nombre total de personnes : ${nombre}`);
}

// Exemple avec relations
const Utilisateur = sequelize.define('Utilisateur', {
    nom: DataTypes.STRING(100),
    email: { type: DataTypes.STRING(120), unique: true }
});

const Article = sequelize.define('Article', {
    titre: DataTypes.STRING(200),
    contenu: DataTypes.TEXT
});

// Définir les relations
Utilisateur.hasMany(Article, { as: 'articles', foreignKey: 'auteurId' });  
Article.belongsTo(Utilisateur, { as: 'auteur', foreignKey: 'auteurId' });  

async function exempleRelations() {
    // Créer un utilisateur avec des articles
    const utilisateur = await Utilisateur.create({
        nom: 'David Moreau',
        email: 'david@email.com',
        articles: [
            { titre: 'Premier article', contenu: 'Contenu du premier article...' },
            { titre: 'Second article', contenu: 'Contenu du second article...' }
        ]
    }, {
        include: [{ association: 'articles' }]
    });

    console.log('✅ Utilisateur créé avec articles');

    // Charger un utilisateur avec ses articles
    const utilisateurAvecArticles = await Utilisateur.findOne({
        where: { email: 'david@email.com' },
        include: ['articles']
    });

    console.log(`\n📚 Articles de ${utilisateurAvecArticles.nom} :`);
    utilisateurAvecArticles.articles.forEach(article => {
        console.log(`  - ${article.titre}`);
    });
}

// Exécution
(async () => {
    await initialiser();
    await exemplesCRUD();
    await exempleRelations();

    // Fermer la connexion
    await sequelize.close();
    console.log('🔒 Connexion fermée');
})();
```

### Java : Hibernate

Hibernate est l'ORM de référence en Java, très puissant mais plus complexe.

#### Configuration (pom.xml)

```xml
<dependencies>
    <dependency>
        <groupId>org.hibernate.orm</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>6.6.4.Final</version>
    </dependency>
    <dependency>
        <groupId>org.xerial</groupId>
        <artifactId>sqlite-jdbc</artifactId>
        <version>3.49.1.0</version>
    </dependency>
    <!-- Dialecte SQLite officiel pour Hibernate 6 -->
    <dependency>
        <groupId>org.hibernate.orm</groupId>
        <artifactId>hibernate-community-dialects</artifactId>
        <version>6.6.4.Final</version>
    </dependency>
</dependencies>
```

> ℹ️ **Note Hibernate 6** : depuis Hibernate 6, le `groupId` est passé de `org.hibernate` à `org.hibernate.orm`. Le dialecte SQLite (`SQLiteDialect`) est disponible dans le module séparé `hibernate-community-dialects`. Vérifiez les dernières versions sur [Maven Central](https://central.sonatype.com/).

#### Configuration Hibernate (hibernate.cfg.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <!-- Configuration de la base de données -->
        <property name="hibernate.connection.driver_class">org.sqlite.JDBC</property>
        <property name="hibernate.connection.url">jdbc:sqlite:exemple.db</property>
        <!-- ⚠️ Hibernate 6 : SQLiteDialect a été déplacé dans le module
             hibernate-community-dialects, package org.hibernate.community.dialect -->
        <property name="hibernate.dialect">org.hibernate.community.dialect.SQLiteDialect</property>

        <!-- Configuration Hibernate -->
        <property name="hibernate.hbm2ddl.auto">update</property>
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>

        <!-- Mapping des entités -->
        <mapping class="com.exemple.model.Personne"/>
        <mapping class="com.exemple.model.Utilisateur"/>
        <mapping class="com.exemple.model.Article"/>
    </session-factory>
</hibernate-configuration>
```

#### Entité Personne

```java
package com.exemple.model;

import jakarta.persistence.*;  
import java.time.LocalDateTime;  

@Entity
@Table(name = "personnes")
public class Personne {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "nom", nullable = false, length = 100)
    private String nom;

    @Column(name = "age")
    private Integer age;

    @Column(name = "email", unique = true, length = 120)
    private String email;

    @Column(name = "date_creation")
    private LocalDateTime dateCreation;

    // Constructeurs
    public Personne() {
        this.dateCreation = LocalDateTime.now();
    }

    public Personne(String nom, Integer age, String email) {
        this();
        this.nom = nom;
        this.age = age;
        this.email = email;
    }

    // Getters et Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getNom() { return nom; }
    public void setNom(String nom) { this.nom = nom; }

    public Integer getAge() { return age; }
    public void setAge(Integer age) { this.age = age; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public LocalDateTime getDateCreation() { return dateCreation; }
    public void setDateCreation(LocalDateTime dateCreation) { this.dateCreation = dateCreation; }

    @Override
    public String toString() {
        return String.format("Personne{id=%d, nom='%s', age=%d, email='%s'}",
                           id, nom, age, email);
    }
}
```

#### Classe utilitaire Hibernate

```java
package com.exemple.util;

import org.hibernate.SessionFactory;  
import org.hibernate.cfg.Configuration;  

public class HibernateUtil {
    private static SessionFactory sessionFactory;

    static {
        try {
            // Créer la SessionFactory à partir de hibernate.cfg.xml
            sessionFactory = new Configuration().configure().buildSessionFactory();
        } catch (Throwable ex) {
            System.err.println("Erreur création SessionFactory : " + ex);
            throw new ExceptionInInitializerError(ex);
        }
    }

    public static SessionFactory getSessionFactory() {
        return sessionFactory;
    }

    public static void shutdown() {
        getSessionFactory().close();
    }
}
```

#### DAO (Data Access Object) pour Personne

```java
package com.exemple.dao;

import com.exemple.model.Personne;  
import com.exemple.util.HibernateUtil;  
import org.hibernate.Session;  
import org.hibernate.Transaction;  
import org.hibernate.query.Query;  

import java.util.List;

public class PersonneDAO {

    // Créer ou mettre à jour une personne
    public void sauvegarder(Personne personne) {
        Transaction transaction = null;
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            transaction = session.beginTransaction();
            // ⚠️ Hibernate 6 : saveOrUpdate() est déprécié.
            //    Pour un objet "détaché" (qui existe déjà en BDD) → session.merge(personne)
            //    Pour un objet "transient" (nouveau, jamais persisté) → session.persist(personne)
            //    merge() couvre les deux cas et reste sûr ; on l'utilise ici par simplicité.
            session.merge(personne);
            transaction.commit();
            System.out.println("✅ Personne sauvegardée : " + personne.getNom());
        } catch (Exception e) {
            if (transaction != null) transaction.rollback();
            System.err.println("❌ Erreur sauvegarde : " + e.getMessage());
        }
    }

    // Trouver une personne par ID
    public Personne trouverParId(Long id) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            return session.get(Personne.class, id);
        } catch (Exception e) {
            System.err.println("❌ Erreur recherche : " + e.getMessage());
            return null;
        }
    }

    // Lister toutes les personnes
    public List<Personne> listerToutes() {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Query<Personne> query = session.createQuery("FROM Personne ORDER BY nom", Personne.class);
            return query.getResultList();
        } catch (Exception e) {
            System.err.println("❌ Erreur liste : " + e.getMessage());
            return List.of();
        }
    }

    // Rechercher par nom
    public List<Personne> rechercherParNom(String nom) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Query<Personne> query = session.createQuery(
                "FROM Personne WHERE nom LIKE :nom ORDER BY nom", Personne.class);
            query.setParameter("nom", "%" + nom + "%");
            return query.getResultList();
        } catch (Exception e) {
            System.err.println("❌ Erreur recherche : " + e.getMessage());
            return List.of();
        }
    }

    // Supprimer une personne
    public void supprimer(Long id) {
        Transaction transaction = null;
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            transaction = session.beginTransaction();
            Personne personne = session.get(Personne.class, id);
            if (personne != null) {
                // ⚠️ Hibernate 6 : session.delete() est déprécié → utilisez session.remove() (style JPA).
                session.remove(personne);
                transaction.commit();
                System.out.println("🗑️ Personne supprimée : " + personne.getNom());
            } else {
                System.out.println("❌ Personne non trouvée avec l'ID : " + id);
            }
        } catch (Exception e) {
            if (transaction != null) transaction.rollback();
            System.err.println("❌ Erreur suppression : " + e.getMessage());
        }
    }

    // Compter le nombre de personnes
    public long compter() {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Query<Long> query = session.createQuery("SELECT COUNT(*) FROM Personne", Long.class);
            return query.getSingleResult();
        } catch (Exception e) {
            System.err.println("❌ Erreur comptage : " + e.getMessage());
            return 0;
        }
    }
}
```

#### Exemple d'utilisation

```java
package com.exemple;

import com.exemple.dao.PersonneDAO;  
import com.exemple.model.Personne;  
import com.exemple.util.HibernateUtil;  

import java.util.List;

public class ExempleHibernate {

    public static void main(String[] args) {
        PersonneDAO dao = new PersonneDAO();

        try {
            // Créer quelques personnes
            System.out.println("🚀 Création de personnes...");
            dao.sauvegarder(new Personne("Alice Martin", 25, "alice@email.com"));
            dao.sauvegarder(new Personne("Bob Dupont", 30, "bob@email.com"));
            dao.sauvegarder(new Personne("Claire Rousseau", 28, "claire@email.com"));

            // Lister toutes les personnes
            System.out.println("\n📋 Liste de toutes les personnes :");
            List<Personne> toutes = dao.listerToutes();
            toutes.forEach(System.out::println);

            // Rechercher une personne
            System.out.println("\n🔍 Recherche de 'Alice' :");
            List<Personne> resultats = dao.rechercherParNom("Alice");
            resultats.forEach(System.out::println);

            // Mettre à jour une personne
            if (!toutes.isEmpty()) {
                Personne premiere = toutes.get(0);
                premiere.setAge(premiere.getAge() + 1);
                dao.sauvegarder(premiere);
                System.out.println("✅ Âge mis à jour pour : " + premiere.getNom());
            }

            // Statistiques
            System.out.println("\n📊 Nombre total de personnes : " + dao.compter());

        } finally {
            // Fermer Hibernate
            HibernateUtil.shutdown();
            System.out.println("🔒 Session Hibernate fermée");
        }
    }
}
```

## Bonnes pratiques avec les ORM

### 1. Configuration et initialisation

#### ✅ Bonnes pratiques

```python
# Python/SQLAlchemy - Configuration centralisée
class DatabaseConfig:
    def __init__(self, database_url, echo=False):
        self.engine = create_engine(database_url, echo=echo)
        self.Session = sessionmaker(bind=self.engine)

    def create_all_tables(self):
        Base.metadata.create_all(self.engine)

    def get_session(self):
        return self.Session()

# Utilisation
config = DatabaseConfig('sqlite:///app.db', echo=True)  
config.create_all_tables()  
```

```javascript
// JavaScript/Sequelize - Configuration environnement
const config = {
    development: {
        dialect: 'sqlite',
        storage: 'dev.db',
        logging: console.log
    },
    production: {
        dialect: 'sqlite',
        storage: 'prod.db',
        logging: false
    }
};

const env = process.env.NODE_ENV || 'development';  
const sequelize = new Sequelize(config[env]);  
```

#### ❌ À éviter

```python
# Configuration dispersée et répétée
engine1 = create_engine('sqlite:///db1.db')  
engine2 = create_engine('sqlite:///db1.db')  # Duplication  
Session1 = sessionmaker(bind=engine1)  
# ... code répétitif
```

### 2. Définition des modèles

> ℹ️ **Note sur le style** : les exemples ci-dessous utilisent indifféremment le **style 1.x** (`Column(Integer, ...)`) et le **style 2.0** (`Mapped[int] = mapped_column(...)`). Les deux fonctionnent en SQLAlchemy 2.x ; le style 1.x reste très répandu dans le code existant. Les principes (clés primaires, contraintes, validation) sont identiques.

#### ✅ Modèles bien structurés

```python
from datetime import datetime, timezone

class Personne(Base):
    __tablename__ = 'personnes'

    # Toujours définir une clé primaire
    id = Column(Integer, primary_key=True)

    # Contraintes explicites
    nom = Column(String(100), nullable=False, index=True)
    email = Column(String(120), unique=True, nullable=False)
    age = Column(Integer, CheckConstraint('age >= 0 AND age <= 150'))

    # Métadonnées utiles — datetime.utcnow déprécié → datetime.now(timezone.utc)
    date_creation = Column(DateTime, default=lambda: datetime.now(timezone.utc), nullable=False)
    date_modification = Column(
        DateTime,
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc)
    )

    # Validation métier
    @validates('email')
    def validate_email(self, key, address):
        if '@' not in address:
            raise ValueError("Email invalide")
        return address

    # Représentation claire
    def __repr__(self):
        return f"<Personne(id={self.id}, nom='{self.nom}')>"
```

#### ❌ Modèles mal définis

```python
class Personne(Base):
    __tablename__ = 'personnes'

    # Pas de contraintes, types vagues
    nom = Column(String)  # Taille non définie
    age = Column(Integer)  # Pas de validation
    # Pas de clé primaire explicite
    # Pas de métadonnées
```

### 3. Gestion des sessions et transactions

#### ✅ Gestion propre des sessions

```python
# Context manager pour SQLAlchemy
from contextlib import contextmanager

@contextmanager
def database_session():
    session = Session()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Utilisation
def ajouter_personne(nom, age, email):
    with database_session() as session:
        personne = Personne(nom=nom, age=age, email=email)
        session.add(personne)
        # Commit automatique si pas d'exception
        return personne.id
```

```javascript
// JavaScript/Sequelize - Transactions automatiques
async function creerUtilisateurAvecArticles(userData, articlesData) {
    const transaction = await sequelize.transaction();

    try {
        const utilisateur = await Utilisateur.create(userData, { transaction });

        const articles = await Promise.all(
            articlesData.map(articleData =>
                Article.create({...articleData, auteurId: utilisateur.id}, { transaction })
            )
        );

        await transaction.commit();
        return { utilisateur, articles };

    } catch (error) {
        await transaction.rollback();
        throw error;
    }
}
```

#### ❌ Gestion risquée

```python
# Session jamais fermée
session = Session()  
personne = Personne(nom="Test")  
session.add(personne)  
# Oubli de commit et close
```

### 4. Requêtes efficaces

#### ✅ Requêtes optimisées

```python
# Chargement eager pour éviter N+1
def lister_utilisateurs_avec_articles():
    with database_session() as session:
        return session.query(Utilisateur)\
                     .options(joinedload(Utilisateur.articles))\
                     .all()

# Requêtes spécifiques avec sélection de colonnes
def lister_noms_emails():
    with database_session() as session:
        return session.query(Personne.nom, Personne.email)\
                     .filter(Personne.age >= 18)\
                     .all()

# Pagination pour de gros datasets
def lister_personnes_pagine(page=1, par_page=10):
    with database_session() as session:
        return session.query(Personne)\
                     .offset((page - 1) * par_page)\
                     .limit(par_page)\
                     .all()
```

```javascript
// Sequelize - Requêtes optimisées
async function listerUtilisateursAvecNombreArticles() {
    return await Utilisateur.findAll({
        attributes: [
            'id', 'nom', 'email',
            [Sequelize.fn('COUNT', Sequelize.col('articles.id')), 'nombreArticles']
        ],
        include: [{
            model: Article,
            as: 'articles',
            attributes: [] // Ne pas charger les articles, juste compter
        }],
        group: ['Utilisateur.id']
    });
}

// Pagination avec comptage
async function listerPersonnesPagine(page = 1, limit = 10) {
    const offset = (page - 1) * limit;

    return await Personne.findAndCountAll({
        offset,
        limit,
        order: [['nom', 'ASC']]
    });
}
```

#### ❌ Requêtes inefficaces

```python
# Problème N+1
utilisateurs = session.query(Utilisateur).all()  
for utilisateur in utilisateurs:  
    # Requête supplémentaire pour chaque utilisateur !
    print(f"{utilisateur.nom} a {len(utilisateur.articles)} articles")
```

### 5. Validation et contraintes

#### ✅ Validation multicouche

```python
from sqlalchemy.orm import validates  
import re  

class Utilisateur(Base):
    __tablename__ = 'utilisateurs'

    id = Column(Integer, primary_key=True)
    nom = Column(String(100), nullable=False)
    email = Column(String(120), unique=True, nullable=False)
    mot_de_passe = Column(String(255), nullable=False)

    @validates('email')
    def validate_email(self, key, email):
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        if not re.match(pattern, email):
            raise ValueError("Format d'email invalide")
        return email.lower()

    @validates('mot_de_passe')
    def validate_password(self, key, password):
        if len(password) < 8:
            raise ValueError("Le mot de passe doit faire au moins 8 caractères")
        return password

    def set_password(self, password):
        # ⚠️ NE JAMAIS utiliser hashlib.sha256 / md5 / sha1 pour des mots de passe !
        # Ces hash sont CONÇUS pour être rapides → un attaquant peut tester
        # des milliards de combinaisons par seconde sur un GPU.
        #
        # ✅ Utilisez un algorithme de "key stretching" conçu pour les mots
        #    de passe (lent + saler automatique) :
        from argon2 import PasswordHasher  # pip install argon2-cffi
        ph = PasswordHasher()
        self.mot_de_passe = ph.hash(password)
        # Alternatives acceptables : bcrypt (pip install bcrypt)
        # ou scrypt/pbkdf2 du module hashlib (avec un sel ≥ 16 octets et ≥ 600 000 itérations)

    def verify_password(self, password):
        from argon2 import PasswordHasher
        from argon2.exceptions import VerifyMismatchError
        try:
            PasswordHasher().verify(self.mot_de_passe, password)
            return True
        except VerifyMismatchError:
            return False
```

> 🔒 **Sécurité des mots de passe — règle absolue** : un hash cryptographique « classique » (SHA-256, MD5, SHA-1) est **inadapté** au stockage de mots de passe. Ces fonctions sont conçues pour être rapides (bonnes pour vérifier l'intégrité d'un fichier, mauvaises pour résister à une attaque par force brute). Utilisez toujours un **password hashing algorithm** spécialement conçu pour cet usage :  
> - **Argon2** (winner du Password Hashing Competition 2015, recommandé par OWASP) → `pip install argon2-cffi`  
> - **bcrypt** (éprouvé depuis 1999, toujours sûr) → `pip install bcrypt`  
> - **scrypt** ou **PBKDF2-HMAC-SHA256** (intégrés dans `hashlib`, avec ≥ 600 000 itérations selon OWASP 2023)  
>  
> Ces algorithmes intègrent le **salage automatique** et un coût configurable qui peut être augmenté avec le temps.

```javascript
// Sequelize - Validation intégrée
const Utilisateur = sequelize.define('Utilisateur', {
    nom: {
        type: DataTypes.STRING(100),
        allowNull: false,
        validate: {
            len: [2, 100],
            notEmpty: true
        }
    },
    email: {
        type: DataTypes.STRING(120),
        allowNull: false,
        unique: true,
        validate: {
            isEmail: true,
            isLowercase: true
        },
        set(value) {
            // Normalisation automatique
            this.setDataValue('email', value.toLowerCase());
        }
    },
    motDePasse: {
        type: DataTypes.STRING(255),
        allowNull: false,
        validate: {
            len: [8, 255],
            is: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/ // Au moins une maj, min, chiffre
        }
    }
}, {
    hooks: {
        beforeCreate: async (utilisateur) => {
            // Hashage automatique du mot de passe
            // ⚠️ Cost factor : 10 était l'historique 2010+, mais OWASP recommande
            //    ≥ 12 en 2026 (réévaluer périodiquement quand le matériel évolue).
            const bcrypt = require('bcrypt');
            utilisateur.motDePasse = await bcrypt.hash(utilisateur.motDePasse, 12);
        }
    }
});
```

#### ❌ Validation insuffisante

```python
# Pas de validation côté ORM
class Utilisateur(Base):
    nom = Column(String)  # Accepte n'importe quoi
    email = Column(String)  # Pas de vérification de format
```

### 6. Gestion des migrations

#### ✅ Migrations versionnées

**Avec Alembic (SQLAlchemy)** :

```bash
# Initialiser Alembic
alembic init alembic

# Générer une migration
alembic revision --autogenerate -m "Ajouter table utilisateurs"

# Appliquer les migrations
alembic upgrade head
```

```python
# Migration exemple (généré par Alembic)
"""Ajouter table utilisateurs

Revision ID: 001  
Create Date: 2024-01-15 10:30:00  

"""
from alembic import op  
import sqlalchemy as sa  

def upgrade():
    op.create_table('utilisateurs',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('nom', sa.String(100), nullable=False),
        sa.Column('email', sa.String(120), nullable=False),
        sa.UniqueConstraint('email')
    )

def downgrade():
    op.drop_table('utilisateurs')
```

**Avec Sequelize CLI** :

```bash
# Installer Sequelize CLI
npm install -g sequelize-cli

# Initialiser
sequelize init

# Générer une migration
sequelize migration:generate --name create-utilisateurs

# Exécuter les migrations
sequelize db:migrate
```

```javascript
// migrations/001-create-utilisateurs.js
'use strict';

module.exports = {
    up: async (queryInterface, Sequelize) => {
        await queryInterface.createTable('Utilisateurs', {
            id: {
                allowNull: false,
                autoIncrement: true,
                primaryKey: true,
                type: Sequelize.INTEGER
            },
            nom: {
                type: Sequelize.STRING(100),
                allowNull: false
            },
            email: {
                type: Sequelize.STRING(120),
                allowNull: false,
                unique: true
            },
            createdAt: {
                allowNull: false,
                type: Sequelize.DATE
            },
            updatedAt: {
                allowNull: false,
                type: Sequelize.DATE
            }
        });
    },

    down: async (queryInterface, Sequelize) => {
        await queryInterface.dropTable('Utilisateurs');
    }
};
```

### 7. Tests avec ORM

#### ✅ Tests bien structurés

```python
# tests/test_models.py
import pytest  
from sqlalchemy import create_engine  
from sqlalchemy.orm import sessionmaker  
from models import Base, Personne  

@pytest.fixture
def db_session():
    # Base de données en mémoire pour les tests
    engine = create_engine('sqlite:///:memory:', echo=False)
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()

    yield session

    session.close()

def test_creation_personne(db_session):
    # Test de création
    personne = Personne(nom="Test User", age=25, email="test@email.com")
    db_session.add(personne)
    db_session.commit()

    # Vérifications
    assert personne.id is not None
    assert personne.nom == "Test User"
    assert personne.age == 25

def test_validation_email(db_session):
    # Test de validation
    with pytest.raises(ValueError):
        personne = Personne(nom="Test", age=25, email="email-invalide")
        db_session.add(personne)
        db_session.commit()

def test_recherche_personne(db_session):
    # Préparer des données de test
    personne1 = Personne(nom="Alice", age=25, email="alice@test.com")
    personne2 = Personne(nom="Bob", age=30, email="bob@test.com")
    db_session.add_all([personne1, personne2])
    db_session.commit()

    # Test de recherche
    resultat = db_session.query(Personne).filter_by(nom="Alice").first()
    assert resultat is not None
    assert resultat.nom == "Alice"
```

```javascript
// tests/models.test.js
const { Sequelize } = require('sequelize');  
const { Personne } = require('../models');  

describe('Modèle Personne', () => {
    let sequelize;

    beforeAll(async () => {
        // Base de données en mémoire pour les tests
        sequelize = new Sequelize('sqlite::memory:', { logging: false });
        await sequelize.sync({ force: true });
    });

    afterAll(async () => {
        await sequelize.close();
    });

    beforeEach(async () => {
        // Nettoyer les données entre les tests
        await Personne.destroy({ where: {}, truncate: true });
    });

    test('Création d\'une personne valide', async () => {
        const personne = await Personne.create({
            nom: 'Test User',
            age: 25,
            email: 'test@email.com'
        });

        expect(personne.id).toBeDefined();
        expect(personne.nom).toBe('Test User');
        expect(personne.age).toBe(25);
    });

    test('Validation email invalide', async () => {
        await expect(Personne.create({
            nom: 'Test',
            age: 25,
            email: 'email-invalide'
        })).rejects.toThrow();
    });

    test('Recherche par critères', async () => {
        await Personne.bulkCreate([
            { nom: 'Alice', age: 25, email: 'alice@test.com' },
            { nom: 'Bob', age: 30, email: 'bob@test.com' }
        ]);

        const personnes = await Personne.findAll({
            where: { age: { [Sequelize.Op.gte]: 30 } }
        });

        expect(personnes).toHaveLength(1);
        expect(personnes[0].nom).toBe('Bob');
    });
});
```

## Cas d'usage avancés

### 1. Système de cache avec ORM

```python
from functools import wraps  
import time  

class CacheManager:
    def __init__(self):
        self.cache = {}
        self.ttl = {}  # Time to live

    def get(self, key):
        if key in self.cache:
            if time.time() < self.ttl[key]:
                return self.cache[key]
            else:
                # Expirée
                del self.cache[key]
                del self.ttl[key]
        return None

    def set(self, key, value, duration=300):  # 5 minutes par défaut
        self.cache[key] = value
        self.ttl[key] = time.time() + duration

cache_manager = CacheManager()

def cache_result(duration=300):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Créer une clé de cache unique
            cache_key = f"{func.__name__}:{str(args)}:{str(kwargs)}"

            # Vérifier le cache
            cached = cache_manager.get(cache_key)
            if cached is not None:
                print(f"📦 Cache hit pour {func.__name__}")
                return cached

            # Exécuter la fonction et mettre en cache
            result = func(*args, **kwargs)
            cache_manager.set(cache_key, result, duration)
            print(f"💾 Résultat mis en cache pour {func.__name__}")
            return result
        return wrapper
    return decorator

# Utilisation avec des requêtes ORM
@cache_result(duration=600)  # Cache 10 minutes
def obtenir_statistiques_utilisateurs():
    with database_session() as session:
        return {
            'total': session.query(Utilisateur).count(),
            'actifs': session.query(Utilisateur).filter_by(actif=True).count(),
            'nouveaux_cette_semaine': session.query(Utilisateur)
                .filter(Utilisateur.date_creation >= datetime.now() - timedelta(days=7))
                .count()
        }
```

### 2. Audit et historique automatique

```python
from sqlalchemy.event import listens_for  
from sqlalchemy import Column, Integer, String, DateTime, Text  
import json  

class AuditLog(Base):
    __tablename__ = 'audit_logs'

    id = Column(Integer, primary_key=True)
    table_name = Column(String(100), nullable=False)
    record_id = Column(Integer, nullable=False)
    action = Column(String(10), nullable=False)  # INSERT, UPDATE, DELETE
    old_values = Column(Text)  # JSON
    new_values = Column(Text)  # JSON
    # datetime.utcnow déprécié Python 3.12+ → utiliser datetime.now(timezone.utc)
    timestamp = Column(DateTime, default=lambda: datetime.now(timezone.utc))
    user_id = Column(Integer)  # Qui a fait la modification

def log_changes(mapper, connection, target):
    """Fonction pour logger les changements"""
    table_name = target.__tablename__
    record_id = target.id

    # Déterminer le type d'action
    if hasattr(target, '_sa_instance_state'):
        if target._sa_instance_state.pending:
            action = 'INSERT'
            old_values = None
            new_values = {c.name: getattr(target, c.name) for c in target.__table__.columns}
        elif target._sa_instance_state.modified:
            action = 'UPDATE'
            # Récupérer les anciennes valeurs
            old_values = {}
            new_values = {}
            for attr in target._sa_instance_state.modified:
                old_values[attr] = target._sa_instance_state.committed_state.get(attr)
                new_values[attr] = getattr(target, attr)

    # Créer l'entrée d'audit
    audit_entry = AuditLog(
        table_name=table_name,
        record_id=record_id,
        action=action,
        old_values=json.dumps(old_values) if old_values else None,
        new_values=json.dumps(new_values) if new_values else None
    )

    # L'ajouter à la session
    session = Session.object_session(target)
    if session:
        session.add(audit_entry)

# Attacher l'audit à tous les modèles qui en ont besoin
@listens_for(Personne, 'after_insert')
@listens_for(Personne, 'after_update')
def log_personne_changes(mapper, connection, target):
    log_changes(mapper, connection, target)
```

### 3. Pattern Repository avec ORM

```python
from abc import ABC, abstractmethod  
from typing import List, Optional, Dict, Any  

class BaseRepository(ABC):
    """Interface de base pour les repositories"""

    @abstractmethod
    def create(self, entity: Any) -> Any:
        pass

    @abstractmethod
    def get_by_id(self, id: int) -> Optional[Any]:
        pass

    @abstractmethod
    def get_all(self) -> List[Any]:
        pass

    @abstractmethod
    def update(self, entity: Any) -> Any:
        pass

    @abstractmethod
    def delete(self, id: int) -> bool:
        pass

class PersonneRepository(BaseRepository):
    """Repository pour les personnes"""

    def __init__(self, session_factory):
        self.session_factory = session_factory

    def create(self, personne_data: Dict[str, Any]) -> Personne:
        with self.session_factory() as session:
            personne = Personne(**personne_data)
            session.add(personne)
            session.commit()
            session.refresh(personne)  # Pour récupérer l'ID généré
            return personne

    def get_by_id(self, id: int) -> Optional[Personne]:
        with self.session_factory() as session:
            return session.query(Personne).filter_by(id=id).first()

    def get_all(self) -> List[Personne]:
        with self.session_factory() as session:
            return session.query(Personne).order_by(Personne.nom).all()

    def get_by_email(self, email: str) -> Optional[Personne]:
        with self.session_factory() as session:
            return session.query(Personne).filter_by(email=email).first()

    def search(self, query: str) -> List[Personne]:
        with self.session_factory() as session:
            return session.query(Personne)\
                         .filter(Personne.nom.ilike(f'%{query}%'))\
                         .order_by(Personne.nom)\
                         .all()

    def update(self, personne: Personne) -> Personne:
        with self.session_factory() as session:
            session.merge(personne)
            session.commit()
            return personne

    def delete(self, id: int) -> bool:
        with self.session_factory() as session:
            personne = session.query(Personne).filter_by(id=id).first()
            if personne:
                session.delete(personne)
                session.commit()
                return True
            return False

    def get_statistics(self) -> Dict[str, Any]:
        with self.session_factory() as session:
            return {
                'total': session.query(Personne).count(),
                'age_moyen': session.query(func.avg(Personne.age)).scalar() or 0,
                'plus_jeune': session.query(func.min(Personne.age)).scalar() or 0,
                'plus_age': session.query(func.max(Personne.age)).scalar() or 0
            }

# Service utilisant le repository
class PersonneService:
    def __init__(self, repository: PersonneRepository):
        self.repository = repository

    def creer_personne(self, nom: str, age: int, email: str) -> Personne:
        # Validation métier
        if self.repository.get_by_email(email):
            raise ValueError(f"L'email {email} est déjà utilisé")

        if age < 0 or age > 150:
            raise ValueError("L'âge doit être entre 0 et 150 ans")

        return self.repository.create({
            'nom': nom,
            'age': age,
            'email': email
        })

    def obtenir_personne(self, id: int) -> Optional[Personne]:
        return self.repository.get_by_id(id)

    def rechercher_personnes(self, query: str) -> List[Personne]:
        if len(query) < 2:
            raise ValueError("La recherche doit contenir au moins 2 caractères")

        return self.repository.search(query)

    def obtenir_statistiques(self) -> Dict[str, Any]:
        return self.repository.get_statistics()

# Utilisation
session_factory = sessionmaker(bind=engine)  
repository = PersonneRepository(session_factory)  
service = PersonneService(repository)  

# Exemples d'utilisation
try:
    # Créer une personne
    personne = service.creer_personne("Alice Martin", 25, "alice@email.com")
    print(f"✅ Personne créée : {personne}")

    # Rechercher
    resultats = service.rechercher_personnes("Alice")
    print(f"🔍 Résultats : {resultats}")

    # Statistiques
    stats = service.obtenir_statistiques()
    print(f"📊 Statistiques : {stats}")

except ValueError as e:
    print(f"❌ Erreur validation : {e}")
```

## Optimisation des performances avec ORM

### 1. Monitoring des requêtes

```python
import time  
from sqlalchemy import event  
from sqlalchemy.engine import Engine  

# Logger les requêtes lentes
@event.listens_for(Engine, "before_cursor_execute")
def receive_before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    context._query_start_time = time.time()

@event.listens_for(Engine, "after_cursor_execute")
def receive_after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    total = time.time() - context._query_start_time

    # Logger les requêtes qui prennent plus de 100ms
    if total > 0.1:
        print(f"⚠️ Requête lente ({total:.3f}s): {statement[:100]}...")

# Compter les requêtes
class QueryCounter:
    def __init__(self):
        self.count = 0

    def reset(self):
        self.count = 0

    def increment(self):
        self.count += 1

query_counter = QueryCounter()

@event.listens_for(Engine, "after_cursor_execute")
def count_queries(conn, cursor, statement, parameters, context, executemany):
    query_counter.increment()

# Utilisation
def fonction_avec_monitoring():
    query_counter.reset()

    # Votre code ici
    with database_session() as session:
        personnes = session.query(Personne).all()
        for personne in personnes:
            print(personne.nom)

    print(f"📊 Nombre de requêtes exécutées : {query_counter.count}")
```

### 2. Optimisation des requêtes N+1

```python
# ❌ Problème N+1
def lister_articles_avec_auteurs_mauvais():
    with database_session() as session:
        articles = session.query(Article).all()  # 1 requête
        for article in articles:
            print(f"{article.titre} par {article.auteur.nom}")  # N requêtes supplémentaires

# ✅ Solution avec jointure
def lister_articles_avec_auteurs_bon():
    with database_session() as session:
        articles = session.query(Article)\
                         .options(joinedload(Article.auteur))\
                         .all()  # 1 seule requête avec JOIN
        for article in articles:
            print(f"{article.titre} par {article.auteur.nom}")

# ✅ Alternative avec contains_eager (quand vous faites le JOIN à la main)
#    Note : `select_related` est un terme de Django ORM, pas de SQLAlchemy.
#    Ici, contains_eager remplit la relation à partir des colonnes du JOIN explicite.
def lister_articles_jointure_explicite():
    with database_session() as session:
        articles = session.query(Article)\
                         .join(Article.auteur)\
                         .options(contains_eager(Article.auteur))\
                         .filter(Utilisateur.actif == True)\
                         .all()
```

> 💡 **`joinedload` vs `selectinload` vs `contains_eager`** :  
> - **`joinedload`** : un seul `SELECT ... LEFT JOIN ...` ; risque de produire un produit cartésien si vous chargez plusieurs *-to-many simultanément.  
> - **`selectinload`** : deux requêtes (une pour le parent, une `WHERE parent_id IN (...)` pour les enfants) — plus rapide pour les *-to-many.  
> - **`contains_eager`** : à utiliser quand vous écrivez le `JOIN` vous-même (avec filtres sur la table jointe).

```javascript
// JavaScript/Sequelize - Éviter N+1
// ❌ Problème N+1
async function listerArticlesAvecAuteursMauvais() {
    const articles = await Article.findAll(); // 1 requête
    for (const article of articles) {
        const auteur = await article.getAuteur(); // N requêtes
        console.log(`${article.titre} par ${auteur.nom}`);
    }
}

// ✅ Solution avec include
async function listerArticlesAvecAuteursBon() {
    const articles = await Article.findAll({
        include: [{
            model: Utilisateur,
            as: 'auteur'
        }]
    }); // 1 seule requête avec JOIN

    articles.forEach(article => {
        console.log(`${article.titre} par ${article.auteur.nom}`);
    });
}
```

### 3. Pagination efficace

**Approche classique avec `OFFSET / LIMIT`** — simple mais s'effondre sur de gros datasets :

```python
def pagination_offset(page=1, per_page=20):
    with database_session() as session:
        offset = (page - 1) * per_page
        query = session.query(Personne)

        # Compter le total (peut être mis en cache si les données changent peu)
        total = query.count()

        # ⚠️ OFFSET = 1 000 000 force SQLite à scanner 1 000 000 lignes
        #    avant d'en retourner 20 → coût linéaire avec la profondeur.
        items = query.offset(offset).limit(per_page).all()

        total_pages = (total + per_page - 1) // per_page
        return {
            'items': items,
            'pagination': {
                'page': page, 'per_page': per_page,
                'total': total, 'total_pages': total_pages,
                'has_next': page < total_pages, 'has_prev': page > 1
            }
        }
```

**Approche moderne — Keyset pagination (« seek pagination »)** : performance constante quelle que soit la profondeur, idéal pour les flux infinis (style Twitter, mobile) :

```python
def pagination_keyset(last_id=None, per_page=20):
    """
    On ne demande plus 'la page 50000' mais 'les 20 après l'id X'.
    Nécessite un index sur la colonne de tri (souvent la clé primaire).
    """
    with database_session() as session:
        query = session.query(Personne).order_by(Personne.id)
        if last_id is not None:
            query = query.filter(Personne.id > last_id)  # ← O(log n) avec index
        items = query.limit(per_page + 1).all()        # +1 pour détecter has_next

        has_next = len(items) > per_page
        items = items[:per_page]
        next_cursor = items[-1].id if has_next and items else None

        return {
            'items': items,
            'next_cursor': next_cursor,  # à passer au prochain appel
            'has_next': has_next,
        }

# Utilisation côté client :
#   page1 = pagination_keyset()
#   page2 = pagination_keyset(last_id=page1['next_cursor'])
```

> 💡 **Quand utiliser quoi ?**  
> - **OFFSET / LIMIT** : pages numérotées (admin BO « page 1, 2, 3… »), datasets < 10 000 lignes.  
> - **Keyset** : timeline / feed infini, datasets > 100 000 lignes, exports paginés API.  
>  
> **Tri composite** : si vous triez sur un champ non unique (ex. `created_at`), utilisez un tuple `(created_at, id)` pour départager les doublons : `WHERE (created_at, id) > (?, ?)`.

## Débogage et diagnostic

### 1. Logging des requêtes SQL

```python
import logging

# Configuration du logging SQLAlchemy
logging.basicConfig()  
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)  

# Ou plus spécifique
engine = create_engine('sqlite:///app.db', echo=True)  # echo=True pour voir toutes les requêtes
```

```javascript
// Sequelize - Logging personnalisé
const sequelize = new Sequelize('sqlite:app.db', {
    logging: (msg) => {
        // Logger personnalisé
        if (msg.includes('SELECT')) {
            console.log('🔍 SELECT:', msg);
        } else if (msg.includes('INSERT')) {
            console.log('➕ INSERT:', msg);
        } else if (msg.includes('UPDATE')) {
            console.log('✏️ UPDATE:', msg);
        } else if (msg.includes('DELETE')) {
            console.log('🗑️ DELETE:', msg);
        }
    }
});
```

### 2. Profiling des performances

```python
import cProfile  
import pstats  
from io import StringIO  

def profiler_fonction(func):
    """Décorateur pour profiler une fonction"""
    def wrapper(*args, **kwargs):
        pr = cProfile.Profile()
        pr.enable()

        result = func(*args, **kwargs)

        pr.disable()
        s = StringIO()
        ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
        ps.print_stats()

        print(f"📊 Profiling de {func.__name__}:")
        print(s.getvalue())

        return result
    return wrapper

@profiler_fonction
def fonction_a_profiler():
    with database_session() as session:
        # Opérations de base de données
        pass
```

## Résumé et recommandations

### ✅ À faire absolument

1. **Toujours utiliser des transactions** pour les opérations critiques
2. **Fermer les sessions** proprement (context managers)
3. **Valider les données** à plusieurs niveaux
4. **Utiliser des requêtes préparées** (automatique avec les ORM)
5. **Optimiser les requêtes** (éviter N+1, pagination)
6. **Tester les modèles** avec des bases de données en mémoire
7. **Versionner les migrations** pour les changements de schéma

### ❌ À éviter absolument

1. **Laisser des sessions ouvertes** indéfiniment
2. **Ignorer les erreurs** de validation
3. **Faire des requêtes dans des boucles** (problème N+1)
4. **Charger de gros datasets** sans pagination
5. **Modifier le schéma** directement sans migrations
6. **Négliger la sécurité** (validation, échappement)

### 🎯 Conseils pour débutants

1. **Commencez simple** : Maîtrisez les opérations CRUD de base
2. **Lisez la documentation** : Chaque ORM a ses spécificités
3. **Utilisez les outils de développement** : Loggez les requêtes SQL
4. **Testez vos modèles** : Écrivez des tests unitaires
5. **Apprenez le SQL** : Comprendre ce qui se passe sous le capot

### 📚 Ressources supplémentaires

**Les ORMs présentés dans ce chapitre** :
- **SQLAlchemy** (Python, le plus mature) : [docs.sqlalchemy.org](https://docs.sqlalchemy.org/)
- **Sequelize** (Node.js) : [sequelize.org](https://sequelize.org/docs/)
- **Hibernate** (Java) : [hibernate.org](https://hibernate.org/orm/documentation/)
- **Alembic** (migrations SQLAlchemy) : [alembic.sqlalchemy.org](https://alembic.sqlalchemy.org/)
- **Sequelize CLI** : [migrations Sequelize](https://sequelize.org/docs/v6/other-topics/migrations/)

**ORMs modernes alternatifs à connaître en 2026** :
- **Python** :
  - **[SQLModel](https://sqlmodel.tiangolo.com/)** : combine SQLAlchemy 2.0 et Pydantic, par l'auteur de FastAPI — typage strict de bout en bout
  - **[Peewee](http://docs.peewee-orm.com/)** : petit ORM léger, idéal pour les scripts et petits projets
  - **[Tortoise ORM](https://tortoise.github.io/)** : ORM async (asyncio), inspiré de Django ORM
  - **[Pony ORM](https://ponyorm.org/)** : syntaxe basée sur les générateurs Python (`SELECT(p for p in Personne if p.age > 18)`)
- **Node.js / TypeScript** :
  - **[Prisma](https://www.prisma.io/)** : ORM moderne avec schéma déclaratif `.prisma`, génération de client typé — très populaire en 2024-2026
  - **[Drizzle ORM](https://orm.drizzle.team/)** : approche "query builder" légère, type-safe, performance proche du SQL brut
  - **[TypeORM](https://typeorm.io/)** : ORM classique avec décorateurs (style Hibernate)
  - **[MikroORM](https://mikro-orm.io/)** : data-mapper pour TypeScript avec unit-of-work
  - **[Kysely](https://kysely.dev/)** : query builder TypeScript ultra-type-safe (pas vraiment un ORM)
- **Java / Kotlin** :
  - **[jOOQ](https://www.jooq.org/)** : type-safe SQL builder, alternative idiomatique à Hibernate
  - **[Exposed](https://github.com/JetBrains/Exposed)** : ORM Kotlin de JetBrains
- **Rust** :
  - **[Diesel](https://diesel.rs/)** : ORM type-safe avec macros
  - **[SeaORM](https://www.sea-ql.org/SeaORM/)** : ORM async pour Tokio
- **Mobile** :
  - **[Room](https://developer.android.com/training/data-storage/room)** (Android) : annotations + génération de DAO à la compilation
  - **[GRDB.swift](https://github.com/groue/GRDB.swift)** (iOS) : type-safe avec Combine/async-await

Cette section vous a donné les bases solides pour utiliser les ORM avec SQLite. La clé du succès est de comprendre ce qui se passe sous le capot tout en profitant de la simplicité des ORM. Commencez par des projets simples et augmentez progressivement la complexité en appliquant les bonnes pratiques présentées ici.


⏭️
