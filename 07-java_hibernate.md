# Creating an application with Hibernate

In the relational world, several ORM frameworks facilitate the integration between the application and the relational object, among them, the most famous in the Java world is Hibernate. It is an open-source Java ORM created by Gavin King and is currently developed by RedHat. In the Java world, there is a standardization process which is the JSR, Java Specification Request, governed by the JCP, Java Community Process (At the moment As I write this chapter, there is a migration process from the JCP process to the Eclipse Foundation with a new name, which will be discussed in the chapter on Jakarta). The specifications guarantee several benefits, among them several companies, academic institutions, and community members contributing to the projects, totally oriented to the community. And the main thing: blocking the vendor-lock in, so the developer does not have the risk of the project being discontinued by one company, since it may be replaced by others. The specification between the Java world and the relational bank is JPA, which is part of the Java EE platform, now Jakarta EE.

Like Spring, Hibernate has several subprojects to facilitate the development world more focused on the world of data persistence, for example, Hibernate Search, Hibernate Validator, among other things. In this chapter, let's talk a little more about Hibernate and its relationship with the non-relational world with Hibernate OGM.


## What is Hibernate OGM?


To facilitate integration and reduce the learning curve to learn non-relational databases, Hibernate OGM was born. Its approach is quite simple, which is to use the JPA API, which Java developers already know, with NoSQL databases. Currently, Hibernate OGM has support for several projects:


* Cassandra
* CouchDB
* EhCache
* Apache Ignite
* Redis


## Practical example with Hibernate OGM

With a brief introduction to the history of Hibernate and the importance of this project in the Java world, we will move on to the practical part. Its configuration is defined in two parts:

1. The insertion of dependencies, from the maven.
2. Adding the database settings from the `persistence.xml` file inside the` META-INF` folder.


```xml
<dependency>
    <groupId> org.hibernate.ogm </groupId>
    <artifactId> hibernate-ogm-cassandra </artifactId>
    <version> 5.1.0.Final </version>
</dependency>
<dependency>
    <groupId> org.jboss.logging </groupId>
    <artifactId> jboss-logging </artifactId>
    <version> 3.3.0.Final </version>
</dependency>
<dependency>
    <groupId> org.hibernate </groupId>
    <artifactId> hibernate-search-orm </artifactId>
    <version> 5.6.1.Final </version>
</dependency>
```

The configuration between the database and the Java framework is performed from the `persistence.xml` file, following the JPA configuration line, that is, there is no news for a developer who already knows this specification.

* `hibernate.ogm.datastore.provider`: defines which implementation Hibernate will use, in this case, the implementation that uses Cassandra.
* `hibernate.ogm.datastore.database`:  for Cassandra, it is the `keyspace`, that is, for this example, it will be the `library`.
* `hibernate.search.default.directory_provider` and` hibernate.search.default.indexBase`: One of the dependencies in Hibernate OGM Cassandra is Hibernate Search, which is the part of Hibernate that offers full-search for objects managed by Hibernate. Searches can be performed using Lucene or Elasticsearch.


```xml
<? xml version = "1.0" encoding = "utf-8"?>
<persistence xmlns = "http://java.sun.com/xml/ns/persistence"
             xmlns: xsi = "http://www.w3.org/2001/XMLSchema-instance"
             xsi: schemaLocation = "http://java.sun.com/xml/ns/persistence http://java.sun.com/xml/ns/persistence/persistence_2_0.xsd"
             version = "2.0">

    <persistence-unit name = "hibernate">
        <provider> org.hibernate.ogm.jpa.HibernateOgmPersistence </provider>
        <class> com.nosqlxp.cassandra.Book </class>
        <properties>
            <property name = "hibernate.ogm.datastore.provider" value = "org.hibernate.ogm.datastore.cassandra.impl.CassandraDatastoreProvider" />
            <property name = "hibernate.ogm.datastore.host" value = "localhost: 9042" />
            <property name = "hibernate.ogm.datastore.create_database" value = "true" />
            <property name = "hibernate.ogm.datastore.database" value = "library" />
            <property name = "hibernate.search.default.directory_provider" value = "filesystem" />
            <property name = "hibernate.search.default.indexBase" value = "/ tmp / lucene / data" />
        </properties>
    </persistence-unit>
</persistence>
```

With the infrastructure part ready within the project, the next step is modeling. Every annotation of the model is using the annotations of JPA, which reduces the learning curve for a developer who already knows the API of relational persistence within the Java world.


```java
@Entity (name = "book")
@Indexed
@Analyzer (impl = StandardAnalyzer.class)
public class Book {

    @Id
    @DocumentId
    private Long isbn;
    
    @Column
    @Field (analyze = Analyze.NO)
    private String name;
    
    @Column
    @Field
    private String author;
    
    @Column
    @Field
    private String category;

   // getter and setter
}

```

To meet the requirement to search for both the key and the category, we will use the search engine feature with Apache Lucene through Hibernate Search. This feature is really interesting, mainly, to allow the search of fields that are not the key. However, this increases the difficulty of the project, since there will be a challenge to synchronize the information within the database and the search engine.

```java
public class App {
    public static void main (String [] args) {
        EntityManagerFactory managerFactory = Persistence.createEntityManagerFactory ("hibernate");
        EntityManager manager = managerFactory.createEntityManager ();
        manager.getTransaction (). begin ();
    
        Book cleanCode = getBook (1L, "Clean Code", "Robert Cecil Martin");
        Book cleanArchitecture = getBook (2L, "Clean Architecture", "Robert Cecil Martin");
        Book agile = getBook (3L, "Agile Principles, Patterns, and Practices in C #", "Robert Cecil Martin");
        Book effectiveJava = getBook (4L, "Effective Java", "Joshua Bloch");
        Book javaConcurrency = getBook (5L, "Java Concurrency", "Robert Cecil Martin");
    
        manager.merge (cleanCode);
        manager.merge (cleanArchitecture);
        manager.merge (agile);
        manager.merge (effectiveJava);
        manager.merge (javaConcurrency);
        manager.getTransaction (). commit ();
    
        Book book = manager.find (Book.class, 1L);
        System.out.println ("book:" + book);
        managerFactory.close ();
    }
    private static Book getBook (long isbn, String name, String author) {
        Book book = new Book ();
        book.setIsbn (isbn);
        book.setName (name);
        book.setAuthor (author);
        return book;
    }

}
```




As expected, every operation takes place through the `EntityManager`. The information only goes to the database when the transaction is committed. Like? The point is that, even though Cassandra does not have a native transaction in the database, Hibernate ends up simulating this behavior that can be dangerous, especially in distributed environments.

The JPQL feature, Java Persistence Query Language, is a query created for JPA and is also available within Hibernate OGM, all thanks to Hibernate Search, which allows you to search for fields in addition to the partition key. There is a counterpart: this field cannot be analyzed within the search, that is, within the annotation `Field`, the attribute `analysis` will need to be defined as `Analyze.NO` (check how the fieldname was noted within the class ).


```java
public class App2 {


    public static void main (String [] args) {
        EntityManagerFactory managerFactory = Persistence.createEntityManagerFactory ("hibernate");
        EntityManager manager = managerFactory.createEntityManager ();
        manager.getTransaction (). begin ();
        String name = "Clean Code";
        Book cleanCode = getBook (1L, "Clean Code", "Robert Cecil Martin");
        Book cleanArchitecture = getBook (2L, "Clean Architecture", "Robert Cecil Martin");
        Book agile = getBook (3L, "Agile Principles, Patterns, and Practices in C #", "Robert Cecil Martin");
        Book effectiveJava = getBook (4L, "Effective Java", "Joshua Bloch");
        Book javaConcurrency = getBook (5L, "Java Concurrency", "Robert Cecil Martin");
    
        manager.merge (cleanCode);
        manager.merge (cleanArchitecture);
        manager.merge (agile);
        manager.merge (effectiveJava);
        manager.merge (javaConcurrency);
        manager.getTransaction (). commit ();
    
        Query query = manager.createQuery ("select b from book b where name =: name");
        query.setParameter ("name", name);
        List <Book> books = query.getResultList ();
        System.out.println ("books:" + books);
        managerFactory.close ();
    }


    private static Book getBook (long isbn, String name, String author) {
        Book book = new Book ();
        book.setIsbn (isbn);
        book.setName (name);
        book.setAuthor (author);
        return book;
    }

}
```

With the search engine activated in the project, it is possible to carry out searches using all Lucene resources. For example, perform a `term` search, which searches for a word within the text, in addition to defining analyzers, tokens, etc. As the code below shows, `FullTextEntityManager` is an `EntityManager` specialization with search engine capabilities. This makes the search possible for non-ID fields that are searchable, in addition to offering very powerful resources.

```java

public class App3 {


    public static void main (String [] args) {
        EntityManagerFactory managerFactory = Persistence.createEntityManagerFactory ("hibernate");
        EntityManager manager = managerFactory.createEntityManager ();
        manager.getTransaction (). begin ();
        manager.merge (getBook (1L, "Clean Code", "Robert Cecil Martin"));
        manager.merge (getBook (2L, "Clean Architecture", "Robert Cecil Martin"));
        manager.merge (getBook (3L, "Agile Principles, Patterns, and Practices in C #", "Robert Cecil Martin"));
        manager.merge (getBook (4L, "Effective Java", "Joshua Bloch"));
        manager.merge (getBook (5L, "Java Concurrency", "Robert Cecil Martin"));
        manager.getTransaction (). commit ();
        FullTextEntityManager fullTextEntityManager = Search.getFullTextEntityManager (manager);
    
        QueryBuilder qb = fullTextEntityManager.getSearchFactory ()
                .buildQueryBuilder (). forEntity (Book.class) .get ();
        org.apache.lucene.search.Query query = qb
                .keyword ()
                .onFields ("name", "author")
                .matching ("Robert")
                .createQuery ();
    
        Query persistenceQuery = fullTextEntityManager.createFullTextQuery (query, Book.class);
        List <Book> result = persistenceQuery.getResultList ();
        System.out.println (result);
    
        manager.close ();
    }


    private static Book getBook (long isbn, String name, String author) {
        Book book = new Book ();
        book.setIsbn (isbn);
        book.setName (name);
        book.setAuthor (author);
        return book;
    }

}
```

There is still a challenge: since we were unable to represent a UDT Set within JPA, how do we search for the category? The answer comes using the features of Hibernate Search. What we will do is add a new field, the `category`. This field will be a `String` and will contain the categories separated by commas. Then, all the work will be done by the search engine. With that, it will be necessary to make a change within the column family to add the new field.

```sql
ALTER COLUMNFAMILY library.book ADD category text;
```

With the field created, just feed the field, and Hibernate Search integrated with the GMO Cassandra will take care of the heavy work of indexing the field and treating it together with Apache Lucene.


```java
public class App4 {


    public static void main (String [] args) {
        EntityManagerFactory managerFactory = Persistence.createEntityManagerFactory ("hibernate");
        EntityManager manager = managerFactory.createEntityManager ();
        manager.getTransaction (). begin ();
        manager.merge (getBook (1L, "Clean Code", "Robert Cecil Martin", "Java, OO"));
        manager.merge (getBook (2L, "Clean Architecture", "Robert Cecil Martin", "Good practice"));
        manager.merge (getBook (3L, "Agile Principles, Patterns, and Practices in C #", "Robert Cecil Martin", "Good practice"));
        manager.merge (getBook (4L, "Effective Java", "Joshua Bloch", "Java, Good practice"));
        manager.merge (getBook (5L, "Java Concurrency", "Robert Cecil Martin", "Java, OO"));
        manager.merge (getBook (6L, "Nosql Distilled", "Martin Fowler", "Java, OO"));
        manager.getTransaction (). commit ();
    
        FullTextEntityManager fullTextEntityManager = Search.getFullTextEntityManager (manager);
    
        QueryBuilder builder = fullTextEntityManager.getSearchFactory (). BuildQueryBuilder (). ForEntity (Book.class) .get ();
        org.apache.lucene.search.Query luceneQuery = builder.keyword (). onFields ("category"). matching ("Java"). createQuery ();
    
        Query query = fullTextEntityManager.createFullTextQuery (luceneQuery, Book.class);
        List <Book> result = query.getResultList ();
        System.out.println (result);
        managerFactory.close ();
    
    }


    private static Book getBook (long isbn, String name, String author, String category) {
        Book book = new Book ();
        book.setIsbn (isbn);
        book.setName (name);
        book.setAuthor (author);
        book.setCategory (category);
        return book;
    }

}
```

### Conclusion

Using an API that the developer already knows to navigate a new paradigm in the world of persistence is a great strategy that Hibernate takes advantage of. In this chapter, we saw how to use the JPA API to work with Apache Cassandra, which greatly reduces the learning curve for an experienced Java developer. However, this facility takes its toll, since the JPA was made for the relational database. There are several gaps that API tends to not cover in the NoSQL world, for example, asynchronous operations as well as definitions of the existing consistency level in Cassandra. In the next chapter, we will also see a specification of the Java world, however, an API created specifically for the non-relational world.