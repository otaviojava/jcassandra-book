# Creating an application with Java EE, ops, Jakarta EE

One of the big differences within the Java world is the specifications. These specifications are governed by the JCP body, whose focus is to ensure transparent communication, and strong participation among groups of Java users around the world. In addition to the community, there are several technical benefits, for example, the possibility of enjoying *multi-vendors,* avoiding being tied to a single supplier (the concept of *vendor lock-in*), commitment to backward compatibility, rich documentation produced by several companies, among other benefits. The purpose of this chapter is to talk a little about one of the fruits of this organ: Java EE and its solution for the non-relational world.

## What is Jakarta EE?

One of the big changes in the Java EE world is that in 2017 it was donated to the Eclipse Foundation by Oracle, so the transition process started there. Thus, Oracle is no longer responsible for the Java EE specification, all of this work will be maintained by the Foundation. An important point to note is what was transferred:


* Code that belongs to the APIs
* Documentations
* Reference implementations that belonged to Oracle (remember that there are specifications that are not managed by Oracle, such as Bean Validation and CDI)

However, the right to the name Java was not delivered, so it was not possible to continue calling it “Java EE”. With that, a poll was held inside the community and they elected the new name: `Jakara EE` in addition to the new logo.
So, in general, Jakarta EE is just the new home of Java EE.

<img src="imagens/jakartaee.png" alt="The Jakarta EE Logo" title="The Jakarta EE Logo" style="zoom:25%;" />

As a premise, under a new direction with the Eclipse Foundation, Jakarta EE will maintain compatibility with the latest version of Java EE, version 8, in addition to bringing news to the platform. One of the important points is that, to bring more news to the platform, a new cycle is being created to specify the famous JSRs, with the main focus on facilitating development, making fast deliveries, and receiving feedbacks in faster ways from the community. As a first specification, Eclipse JNoSQL was born, whose focus is on integrating NoSQL and Java databases.


## Using Jakarta NoSQL, Jakarta EE's first specification

Jakarta NoSQL is a framework that integrates Java applications with NoSQL databases. It defines a group of APIs whose purpose is to standardize the communication between most databases and their common operations. This helps to decrease coupling with this type of technology used in current applications.
The project has two layers:

1. **Communication layer**: it is a group of APIs that define communication with non-relational databases. Compared to traditional non-relational banks, they are similar to the JDBC APIs. It contains four modules, one for each type of NoSQL bank: key-value, column family, document, and graphs.

2. **Mapping layer**: API that helps the developer to integrate with the non-relational database, being oriented to annotations and using technologies such as dependency injection and Bean Validation, which makes it simple for developers to use. Comparing with the classic RDBMS, this layer can be compared with JPA or other mapping frameworks like Hibernate.


! [Eclipse JNoSQL architecture] (images / jnosql.png "Jakarta NoSQL architecture")


As with Spring Data, Jakarta NoSQL works in conjunction with a dependency injection engine, however, the CDI specification is used in the project. Using the same principle, we will start with the codes and configuration files that are necessary to make the CDI container lift, in addition to communicating with Cassandra.

> CDI is a very interesting dependency injection framework with several features, such as scope definition, event triggering synchronously and asynchronously, in addition to being the standard of the Java world.

The configuration file is `bean.xml`, which is located inside `META-INF` (the same location as JPA `persistence.xml`). There will also be `CassandraProducer`, which will be responsible for creating a connection with Cassandra.

```xml
<beans xmlns = "http://xmlns.jcp.org/xml/ns/javaee"
       xmlns: xsi = "http://www.w3.org/2001/XMLSchema-instance"
       xsi: schemaLocation = "http://xmlns.jcp.org/xml/ns/javaee
http://xmlns.jcp.org/xml/ns/javaee/beans_1_1.xsd "
       bean-discovery-mode = "all">
</beans>
```

The configuration class “teaches” the CDI how to generate the dependency on `CassandraColumnFamilyManager`, which is the class responsible for performing the communication between Java and the database. The `Settings` class represents the information for connecting to the database. In it, it is possible, for example, to define the password, user, clusters, among other information.

The next step is to carry out the modeling of the book entity. If you come from the JPA world you will see that the concepts are the same, that is, we use the `Entity` annotation to map an entity, the Id annotation to identify the attribute that will be the unique identifier, in addition to the `Column` annotation for identifying the other fields that will be persisted.

```java
@Entity ("book")
public class Book {
    @Id ("isbn")
    private Long isbn;
    @Column
    private String name;
    @Column
    private String author;
    @Column
    private Set <String> categories;
    // getter and setter
}
```

Jakarta NoSQL's main feature is the integration with a dependency injection framework, just like Spring Data, however, the difference is that the specification uses the CDI, which is the specification of the Java world. Another similarity between the integration tools is that the first step is to lift the container. The project has a template class for operations within the mapper: `ColumnTemplate`, however, it works as a skeleton of operations for a mapper for all banks of the column family type, that is, it would be possible to switch to Hbase with little or no impact on the code.


```java
public class App
{
    public static void main (String [] args)
    {
        try (SeContainer container = SeContainerInitializer.newInstance (). initialize ()) {
            ColumnTemplate template = container.select (ColumnTemplate.class) .get ();

            Book cleanCode = getBook (1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet ("Java", "OO"));
            Book cleanArchitecture = getBook (2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet ("Good practice"));
            Book effectiveJava = getBook (3L, "Effective Java", "Joshua Bloch", Sets.newHashSet ("Java", "Good practice"));
            Book nosql = getBook (4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet ("NoSQL", "Good practice"));
    
            template.insert (cleanCode);
            template.insert (cleanArchitecture);
            template.insert (effectiveJava);
            template.insert (nosql);
    
            ColumnQuery query = select (). From ("book"). Build ();
            Stream <Book> books = template.select (query);
            books.forEach (System.out :: println);
        }
    }
    
    private static Book getBook (long isbn, String name, String author, Set <String> categories) {
        Book book = new Book ();
        book.setIsbn (isbn);
        book.setName (name);
        book.setAuthor (author);
        book.setCategories (categories);
        return book;
    }
}
```

The Jakarta NoSQL API has interesting features like the query and the fluent API, it is worth mentioning, that these features are portable to other non-relational databases of the column family type like Apache Hbase. In the following code, we will show queries using the Jakarta NoSQL API using both the fluent API resource and using a query as text, an important point is the use of `Optional` when searching with only a single element, that is, the API was born above Java 8.


```java
public class App2 {


    public static void main (String [] args) {
    
        try (SeContainer container = SeContainerInitializer.newInstance (). initialize ()) {
            ColumnTemplate template = container.select (ColumnTemplate.class) .get ();
    
            Book cleanCode = getBook (1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet ("Java", "OO"));
            Book cleanArchitecture = getBook (2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet ("Good practice"));
            Book effectiveJava = getBook (3L, "Effective Java", "Joshua Bloch", Sets.newHashSet ("Java", "Good practice"));
            Book nosql = getBook (4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet ("NoSQL", "Good practice"));
    
            template.insert (cleanCode);
            template.insert (cleanArchitecture);
            template.insert (effectiveJava);
            template.insert (nosql);


            Optional <Book> book = template.find (Book.class, 1L);
            System.out.println ("Book found:" + book);
    
            template.delete (Book.class, 1L);
    
            System.out.println ("Book found:" + template.find (Book.class, 1L));


            PreparedStatement prepare = template.prepare ("select * from Book where isbn = @isbn");
            prepare.bind ("isbn", 2L);
            Optional <Book> result = prepare.getSingleResult ();
            System.out.println ("prepare:" + result);
        }
    
    }
    
    private static Book getBook (long isbn, String name, String author, Set <String> categories) {
        Book book = new Book ();
        book.setIsbn (isbn);
        book.setName (name);
        book.setAuthor (author);
        book.setCategories (categories);
        return book;
    }

}
```

In the category mapping, the sequence remains similar, with the difference of the `UDT` mapping. Within Jakarta NoSQL, there is an extension feature that allows APIs for specific uses for each database. For example, we will use an extension of Cassandra that allows us to use the annotation `UDT` to map our code to the UDT type, as shown in the following mapping.

```java
@Entity ("category")
public class Category {

    @Id ("name")
    private String name;
    
    @Column
    @UDT ("book")
    private Set <BookType> books;
    // getter and setter
}


public class BookType {

    @Column
    private Long isbn;
    
    @Column
    private String name;
    
    @Column
    private String author;
    
    @Column
    private Set <String> categories;
    // getter and setter
}
```

Once using Cassandra-specific annotations, `CassandraTemplate` will be used, which is a specialization of` ColumnTemplate` with specific features for Cassandra, such as the possibility of defining the level of consistency during the request and making native queries, that is, CQL.


```java
public class App4 {


    public static void main (String [] args) {
        try (SeContainer container = SeContainerInitializer.newInstance (). initialize ()) {
            CassandraTemplate template = container.select (CassandraTemplate.class) .get ();
    
            BookType cleanCode = getBook (1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet ("Java", "OO"));
            BookType cleanArchitecture = getBook (2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet ("Good practice"));
            BookType effectiveJava = getBook (3L, "Effective Java", "Joshua Bloch", Sets.newHashSet ("Java", "Good practice"));
            BookType nosqlDistilled = getBook (4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet ("NoSQL", "Good practice"));


            Category java = getCategory ("Java", Sets.newHashSet (cleanCode, effectiveJava));
            Category oo = getCategory ("OO", Sets.newHashSet (cleanCode, effectiveJava, cleanArchitecture));
            Category goodPractice = getCategory ("Good practice", Sets.newHashSet (cleanCode, effectiveJava, cleanArchitecture, nosqlDistilled));
            Category nosql = getCategory ("NoSQL", Sets.newHashSet (nosqlDistilled));
    
            template.insert (java);
            template.insert (oo);
            template.insert (goodPractice);
            template.insert (nosql);
    
            Optional <Category> category = template.find (Category.class, "Java");
            System.out.println (category);
            template.delete (Category.class, "Java");
    
            PreparedStatement prepare = template.prepare ("select * from Category where name = @name");
            prepare.bind ("name", "NoSQL");
            Optional <Book> result = prepare.getSingleResult ();
            System.out.println ("prepare:" + result);
        }
    
    }
    
    private static Category getCategory (String name, Set <BookType> books) {
        Category category = new Category ();
        category.setName (name);
        category.setBooks (books);
        return category;
    }
    
    private static BookType getBook (long isbn, String name, String author, Set <String> categories) {
        BookType book = new BookType ();
        book.setIsbn (isbn);
        book.setName (name);
        book.setAuthor (author);
        book.setCategories (categories);
        return book;
    }

}
```

In addition to the template classes, Jakarta NoSQL supports the concept of repository interfaces that follows the same principle as Spring Data: interfaces that aim to have a somewhat degree of abstraction to perform queries within the database. It also brings an interface that already has several methods and `method by query` that will be implemented automatically by the framework. In this case, `CassandraRepository` will be used, which is a specialization of` Repository` that allows, for example, the use of the `CQL` annotation, which basically executes Cassandra Query language and converts it to the entity automatically.


```java
public interface BookRepository extends CassandraRepository <Book, Long> {

    Stream <Book> findAll ();
    
    @CQL ("select * from book")
    Stream <Book> findAll1 ();
    
    @Query ("select * from Book")
    Stream <Book> findAll2 ();
}

```

> The difference between the `CQL` and` Query` annotations is that the first executes CQL, which is Cassandra's native query, thus exclusive to the framework, and the second is the Jakarta NoSQL API, that is, it can be executed by other databases that support the communication layer of the Jakarta EE world specification project.

In the code below, we will do our integration with `BookRepository`. What most impresses, certainly, is the ease of using it since we don't have to worry about implementing the interface. An important point is that this interface inherits from `CassandraRepository`, which makes this repository inherit several methods such as` save`, `findById`, among others.

```java
public class App5
{
    public static void main (String [] args)
    {
        try (SeContainer container = SeContainerInitializer.newInstance (). initialize ()) {
            BookRepository repository = container.select (BookRepository.class) .get ();

            Book cleanCode = getBook (1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet ("Java", "OO"));
            Book cleanArchitecture = getBook (2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet ("Good practice"));
            Book effectiveJava = getBook (3L, "Effective Java", "Joshua Bloch", Sets.newHashSet ("Java", "Good practice"));
            Book nosql = getBook (4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet ("NoSQL", "Good practice"));
    
            repository.save (cleanCode);
            repository.save (cleanArchitecture);
            repository.save (effectiveJava);
            repository.save (nosql);
    
            Optional <Book> book = repository.findById (1L);
            System.out.println (book);
    
            repository.deleteById (1L);
    
            System.out.println ("Using method query");
            repository.findAll (). forEach (System.out :: println);
            System.out.println ("Using CQL");
            repository.findAll1 (). forEach (System.out :: println);
            System.out.println ("Using query JNoSQL");
            repository.findAll2 (). forEach (System.out :: println);
    
        }
    }
    
    private static Book getBook (long isbn, String name, String author, Set <String> categories) {
        Book book = new Book ();
        book.setIsbn (isbn);
        book.setName (name);
        book.setAuthor (author);
        book.setCategories (categories);
        return book;
    }
}

```

> The project still has several resources that were not shown here, for example, to perform operations asynchronously. To learn more visit: http://www.jnosql.org/

> The code with every example can be found at: https://github.com/otaviojava/cassandra-java-code to create Cassandra structures, see the chapter “Performing integration with Java”


### Conclusion

With a new home and in an even more vibrant way, the Eclipse JNoSQL project was born under a new direction by Jakarta EE with the Eclipse Foundation. Jakarta NoSQL aims to facilitate the integration between Java and NoSQL with the strategy of dividing the communication and mapping layer. It currently supports more than thirty databases. Many improvements are expected, however, the great benefit of the platform is that it is totally community-oriented. You can get out of your chair and help Jakarta EE yourself right now.