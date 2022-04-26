# Creating an application with Java and Spring


The Spring framework is an open-source project whose objective is to facilitate Java development and today it has become one of the most popular tools worldwide. Its history clashes a lot with that of Java EE since it was born to fill the gap that Java EE could not, besides simplifying several points, such as the security part. In its beginning it was just a dependency injection framework, however, it currently has several subprojects, among which we can mention:

* Spring Batch
* Spring Boot
* Spring security
* Spring LDAP
* Spring XD
* Spring Data
* And much more


In this chapter, a little of the Spring world integrated with the use of Cassandra will be presented.

## Making data access easier with Spring Data


Spring Data is one of several projects within the framework's umbrella. This subproject has the main objective of facilitating the integration between Java and the databases. Several databases are supported within Spring. Among its greatest features are:

* A great abstraction _object-mapping_;
* The _query by method_, which is based on the dynamic query performed on the interface;
* Easy integration with other projects within Spring, including Spring MVC and JavaConfig;
* Audit support.

Within Spring Data, there is Spring Data Cassandra, which supports the creation of repositories, synchronous and asynchronous operations, resources such as *query builders,* and a very high abstraction from Cassandra - to the point of making learning Cassandra Query Language unnecessary. To get into the wonderful world of Spring, the first step is to add dependency to the project.


```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-cassandra</artifactId>
    <version>3.1.2</version>
</dependency>
```

Once the dependencies are defined, the next step is the infrastructure code that activates Spring and the connection configuration with Cassandra. The `Config` class has two annotations: one to search for components within a specific package and another to do something similar to the Cassandra repositories. The `CassandraConfig` class has the settings for connecting to Cassandra, for example, the keyspace to be used, cluster settings, and the Mapper Cassandra Spring.

> Mapper Cassandra Spring, like Mappers in general, has the responsibility of converting a business entity in Java to Cassandra or vice versa.


```java
@Configuration
@EnableCassandraRepositories(basePackages = "otaviojava.github.io.cassandra")
public class CassandraConfig extends AbstractCassandraConfiguration {

    @Override
    protected String getKeyspaceName() {
        return "library";
    }

    @Override
    public String getContactPoints() {
        return "localhost";
    }

}

```

Configuration code and infrastructure created, we will work on the first example, which is reading and writing the book. Modeling happens simply and intuitively thanks to the Spring Data Cassandra annotations:

* `Table` maps the entity.
* `PrimaryKey` identifies the primary key within the entity.
* `Column` defines the attributes that will be persisted within Cassandra.

```java
@Table
public class Book {

    @PrimaryKey
    private Long isbn;
    
    @Column
    private String name;
    
    @Column
    private String author;
    
    @Column
    private Set<String> categories;
    
    // getter and setter
}
```



For data manipulation, there is the `CassandraTemplate` class, which is a skeleton to operate Cassandra and the mapped object. It works in a very analogous way to the Template Method pattern that defines the skeleton for the algorithm, however, for operations in the database with Cassandra, mapping the entities to the database and vice versa. An important point of `CassandraTemplate` is that it is possible to make CQL calls, which will be in charge of converting to the target object - in this case, `Book`.

```java
public class App {
    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "book";

    public static void main(String[] args) {

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(CassandraConfig.class)) {

            CassandraTemplate template = ctx.getBean(CassandraTemplate.class);

            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Set.of("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Set.of("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            template.insert(cleanCode);
            template.insert(cleanArchitecture);
            template.insert(effectiveJava);
            template.insert(nosql);

            List<Book> books = template.select(QueryBuilder.selectFrom(KEYSPACE, COLUMN_FAMILY).all().build(), Book.class);
            System.out.println(books);


        }
    }

    private static Book getBook(long isbn, String name, String author, Set<String> categories) {
        Book book = new Book();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        book.setCategories(categories);
        return book;
    }
}
```

The `AnnotationConfigApplicationContext` class raises the Spring container, scanning the annotated and defined classes, looking for dependency injections. It allows the use of _try-resources_, that is, as soon as it leaves the `try` block, the JVM itself will take care of calling and the` close` method and closing the Spring container for the developer.

```java
public class App2 {


    public static void main(String[] args) {

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(CassandraConfig.class)) {

            CassandraTemplate template = ctx.getBean(CassandraTemplate.class);

            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Set.of("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Set.of("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            template.insert(cleanCode);
            template.insert(cleanArchitecture);
            template.insert(effectiveJava);
            template.insert(nosql);

            Book book = template.selectOneById(1L, Book.class);
            System.out.println(book);
            template.deleteById(1L, Book.class);

        }

    }

    private static Book getBook(long isbn, String name, String author, Set<String> categories) {
        Book book = new Book();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        book.setCategories(categories);
        return book;
    }

}
```




For the next step, it is possible to notice that no contact is made with the CQL itself, only with the Cassandra template. In the following code, we will use `CassandraTemplate` and perform the operations of inserting, retrieving, and removing. Just to remember, we do not use `update`, since it works as an alias for insert.

For the last part of the challenge, which consists of reading the book categories, the annotations are the same used in the case of the book, except for the UDT `Book`, which has the annotations `UserDefinedType` and `CassandraType`, defining the name UDT and field information, respectively.

```java
@Table
public class Category {
    @PrimaryKey
    private String name;
    @Column
    private Set<BookType> books;
   // getter and setter
}
@UserDefinedType("book")
public class BookType {
    @CassandraType(type = CassandraType.Name.BIGINT)
    private Long isbn;
    @CassandraType(type = CassandraType.Name.TEXT)
    private String name;
    @CassandraType(type = CassandraType.Name.TEXT)
    private String author;
    @CassandraType(type = CassandraType.Name.SET, typeArguments = CassandraType.Name.TEXT)
    private Set<String> categories;
    // getter and setter
}
```

In addition to the UDT notes, nothing differs from the first two cases regarding the query by the key and the persistence of the database. In the code below we will show an interaction between the entities and the UTC type. An important thing is the great power that Cassandra has and the possibilities of modeling without making the famous *joins* in SQL. We will make an insertion of the category, which has a book type `Set`, and the book type which is a UDT will have a String `Set`.

```java
public class App3 {

    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "category";

    public static void main(String[] args) {

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(CassandraConfig.class)) {

            CassandraTemplate template = ctx.getBean(CassandraTemplate.class);

            BookType cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO"));
            BookType cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Set.of("Good practice"));
            BookType effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Set.of("Java", "Good practice"));
            BookType nosqlDistilled = getBook(4L, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            Category java = getCategory("Java", Set.of(cleanCode, effectiveJava));
            Category oo = getCategory("OO", Set.of(cleanCode, effectiveJava, cleanArchitecture));
            Category goodPractice = getCategory("Good practice", Set.of(cleanCode, effectiveJava, cleanArchitecture, nosqlDistilled));
            Category nosql = getCategory("NoSQL", Set.of(nosqlDistilled));

            template.insert(java);
            template.insert(oo);
            template.insert(goodPractice);
            template.insert(nosql);

            List<Category> categories = template.select(QueryBuilder.selectFrom(KEYSPACE, COLUMN_FAMILY).all().build(), Category.class);
            System.out.println(categories);
        }

    }

    private static Category getCategory(String name, Set<BookType> books) {
        Category category = new Category();
        category.setName(name);
        category.setBooks(books);
        return category;
    }

    private static BookType getBook(long isbn, String name, String author, Set<String> categories) {
        BookType book = new BookType();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        book.setCategories(categories);
        return book;
    }

}

public class App4 {


    public static void main(String[] args) {

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(CassandraConfig.class)) {

            CassandraTemplate template = ctx.getBean(CassandraTemplate.class);

            BookType cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO"));
            BookType cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Set.of("Good practice"));
            BookType effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Set.of("Java", "Good practice"));
            BookType nosqlDistilled = getBook(4L, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            Category java = getCategory("Java", Set.of(cleanCode, effectiveJava));
            Category oo = getCategory("OO", Set.of(cleanCode, effectiveJava, cleanArchitecture));
            Category goodPractice = getCategory("Good practice", Set.of(cleanCode, effectiveJava, cleanArchitecture, nosqlDistilled));
            Category nosql = getCategory("NoSQL", Set.of(nosqlDistilled));

            template.insert(java);
            template.insert(oo);
            template.insert(goodPractice);
            template.insert(nosql);

            Category category = template.selectOneById("Java", Category.class);
            System.out.println(category);
            template.deleteById("Java", Category.class);

        }

    }

    private static Category getCategory(String name, Set<BookType> books) {
        Category category = new Category();
        category.setName(name);
        category.setBooks(books);
        return category;
    }

    private static BookType getBook(long isbn, String name, String author, Set<String> categories) {
        BookType book = new BookType();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        book.setCategories(categories);
        return book;
    }

}
```

In addition to the template class, Spring Data Cassandra has the concept of **dynamic repositories**, in which the developer creates an interface and Spring will be responsible for the respective implementation. The new interface will inherit from `CassandraRepository`, which already has a large number of operations for the database.

Also, it is possible to use the concept of `query by method`, with which, when using search conversions in the method name, Spring will do all the heavy lifting. With these repositories, we have a valuable abstraction that reduces the number of code, generating very high productivity.


```java
@Repository
public interface BookRepository extends CassandraRepository<Book, Long> {

    @Query("select * from book")
    List<Book> findAll();
}


public class App5 {

    public static void main(String [] args) {
    
        try(AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Config.class)) {
    
            BookRepository repository = ctx.getBean(BookRepository.class);
            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));
    
            repository.insert(cleanCode);
            repository.insert(cleanArchitecture);
            repository.insert(effectiveJava);
            repository.insert(nosql);
    
            List<Book> books = repository.findAll();
            System.out.println(books);
    
            Optional<Book> book = repository.findById(1L);
            System.out.println(book);


        }
    }


    private static Book getBook(long isbn, String name, String author, Set<String> categories) {
        Book book = new Book();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        book.setCategories(categories);
        return book;
    }
}

```

> The `CassandraRepository` interface is a specialization of `CrudRepository` for Cassandra operations. `CrudRepository` is a specialization of Repository. These interfaces are part of Spring Data. For more information:[ ](https://docs.spring.io/spring-data/data-commons/docs/2.1.x/reference/html/): https://docs.spring.io/spring-data/data-commons/docs/2.1.x/reference/html/
>
> Spring Data Cassandra has many more features, such as asynchronous operations that make the day to day of the developer much easier. To learn more: [https://docs.spring.io/spring-data/cassandra/docs/3.1.2/reference/html/#reference](https://docs.spring.io/spring-data/cassandra/docs/3.1.2/reference/html/#reference)


### Conclusion

The Spring framework is a project that brought great innovation to the Java world. Its features and facilities make it highly popular. Within the communication with the database, there is Spring Data with several facilitations to the point that it is not necessary to learn Cassandra Query Language. In this chapter, we had an introduction to Spring Data Cassandra and its resources. Its ease and integration with the Spring container make Spring Data a great solution for applications that already use or intend to use Spring in some way.