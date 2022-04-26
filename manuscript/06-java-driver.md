# Performing integration with Java

After all the theory, installation, notion of communication with CQL and modeling, it's finally time to put it all together and put it inside a Java application. The integration of the database with the language is one of the most important things to be done within an application and that is the objective of this chapter.

## Minimum requirements for the demos

To run all the demos, you will need to have Java 8 installed, in addition to Maven above version 3.5.3. At the time of writing, Java is in version 11, however, few frameworks have support for it, so to maintain coherence and facilitate the management of environment variables, the latest version of Java 8 will be used.

As the purpose of the book is to talk about Cassandra, we consider that you have a notion of Java 8 and Maven, in addition to having both installed and properly configured in your favorite operating system. You use any IDE you feel comfortable with.

To facilitate the comparison between the Java integration tools, a simple example will be used. To reduce the complexity of the application and not divert the focus, the examples will be based only on Java SE, that is, running purely the good old `public static void main(String[])`.

Our example will be based on the same case as a bookstore:

We will have a library that will have books and each book will have an author and a category. The desired queries are:

* Search for the book from the ISBN;
* Search for books from the respective category.

Thinking simply, a model that meets all queries and distributes information would look like this:

```sql
CREATE KEYSPACE IF NOT EXISTS library WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};
DROP COLUMNFAMILY IF EXISTS library.book;
DROP COLUMNFAMILY IF EXISTS library.category;
DROP TYPE IF EXISTS library.book;
CREATE TYPE IF NOT EXISTS library.book(
    isbn bigint,
    name text,
    author text,
    categories set <text>
);
CREATE COLUMNFAMILY IF NOT EXISTS library.book(
    isbn bigint,
    name text,
    author text,
    categories set <text>,
    PRIMARY KEY(isbn)
);
CREATE COLUMNFAMILY IF NOT EXISTS library.category(
  name text PRIMARY KEY,
  books set <frozen <book>>
);

```




With this code, we created the initial structure, that is, the book column family to then integrate Cassandra with Java code.

## Skeleton of the example projects

As already mentioned, the examples will be based on Java SE, so our first step with Java is to create the project. We will use the *maven archetype* to speed up the creation of this maven project. Access the terminal and run the following command:


```bash
mvn archetype:generate -DgroupId=com.mycompany.app -DartifactId=my-app -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

An important point is that inside `pom.xml`, it is important to update it to support Java 11. Thus, the file would look like this:

```xml
<project xmlns = "http://maven.apache.org/POM/4.0.0" xmlns: xsi = "http://www.w3.org/2001/XMLSchema-instance"
         xsi: schemaLocation = "http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.mycompany.app</groupId>
    <artifactId>my-app</artifactId>
    <packaging>jar</packaging>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
   </properties>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>11</source>
                    <target>11</target>
               </configuration>
           </plugin>
       </plugins>
   </build>
    <name>my-app</name>
    <url>http://maven.apache.org</url>
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>3.8.1</version>
            <scope>test</scope>
       </dependency>
   </dependencies>
</project>
```



> For each new project, a good strategy would be to create it using the same skeleton.

## Using the Cassandra driver

To start developing the code, the first step is to add the driver dependency within the file that manages the Maven dependencies, that is, the `pom.xml` inside the `dependencies` tag:

```xml
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-core</artifactId>
    <version>${data.stax.version}</version>
</dependency>
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-query-builder</artifactId>
    <version>${data.stax.version}</version>
</dependency>
```

> To make it easier, we will not activate security and will be based on a local installation or with Docker with the following command: `docker run -d --name casandra-instance -p 9042:9042 cassandra`.

With the dependency within the project, the next step is to start communication. The `CqlSession` interface represents the connection to Cassandra, thus, it is from there that the insertion and update commands are executed within the database.

```java
public class App {

    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "book";

    public static void main(String[] args) {
        try (CqlSession session = CqlSession.builder().build()) {

            Map<String, Term> cleanCode = createInsertQuery(new Object[]{1, "Clean Code", "Robert Cecil Martin",
                    Set.of("Java", "OO")});
            Map<String, Term> cleanArchitecture = createInsertQuery(new Object[]{2, "Clean Architecture",
                    "Robert Cecil Martin",
                    Set.of("Good practice")});
            Map<String, Term> effectiveJava = createInsertQuery(new Object[]{3, "Effective Java", "Joshua Bloch",
                    Set.of("Java", "Good practice")});
            Map<String, Term> nosql = createInsertQuery(new Object[]{4, "Nosql Distilled", "Martin Fowler",
                    Set.of("NoSQL", "Good practice")});

            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(cleanCode).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(cleanArchitecture).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(effectiveJava).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(nosql).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(cleanCode).build());

            ResultSet resultSet = session.execute(QueryBuilder.selectFrom(KEYSPACE, COLUMN_FAMILY).all().build());
            for (Row row : resultSet) {
                Long isbn = row.getLong("isbn");
                String name = row.getString("name");
                String author = row.getString("author");
                Set<String> categories = row.getSet("categories", String.class);
                System.out.println(String.format(" the result is %s %s %s %s", isbn, name, author, categories));
            }
        }

    }

    private static Map<String, Term>  createInsertQuery(Object[] parameters) {
        return Map.of("isbn", literal(parameters[0]), "name", literal(parameters[1]),
                "author", literal(parameters[2]),
                "categories", literal(parameters[3]));

    }

}
```




In the first class of interaction with Cassandra, it is possible to notice the intense use of Cassandra Query Language, which we explained in chapter 4. All operations with CQL are facilitated from the `QueryBuilder` class, which is a utility that contains several methods that make life easier for the developer to create CQL for an application.


In our first interaction, we performed an insertion and a search of all values within the database, but this type of search is not very common and neither is it optimized. As we mentioned in the modeling chapter, the most recommended is always to perform a search for the key. Thus, in our next interaction, we will perform searches from the key. To make the code a little easier, we will create a `Consumer` to log the result(it will not be anything sophisticated, just the good old `System.out.println`).


```java
public class App2 {

    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "book";

    public static void main(String[] args) {
        try (CqlSession session = CqlSession.builder().build()) {

            Map<String, Term> cleanCode = createInsertQuery(new Object[]{1, "Clean Code", "Robert Cecil Martin",
                    Set.of("Java", "OO")});
            Map<String, Term> cleanArchitecture = createInsertQuery(new Object[]{2, "Clean Architecture",
                    "Robert Cecil Martin",
                    Set.of("Good practice")});
            Map<String, Term> effectiveJava = createInsertQuery(new Object[]{3, "Effective Java", "Joshua Bloch",
                    Set.of("Java", "Good practice")});
            Map<String, Term> nosql = createInsertQuery(new Object[]{4, "Nosql Distilled", "Martin Fowler",
                    Set.of("NoSQL", "Good practice")});

            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(cleanCode).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(cleanArchitecture).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(effectiveJava).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(nosql).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(cleanCode).build());

            Consumer<Row> log = row -> {
                Long isbn = row.getLong("isbn");
                String name = row.getString("name");
                String author = row.getString("author");
                Set<String> categories = row.getSet("categories", String.class);
                System.out.println(String.format(" the result is %s %s %s %s", isbn, name, author, categories));
            };

            findById(session,1L, log);

            deleteById(session, 1L);

            PreparedStatement prepare = session.prepare("select * from library.book where isbn = ?");
            BoundStatement statement = prepare.bind(2L);
            ResultSet resultSet = session.execute(statement);
            resultSet.forEach(log);

        }

    }

    private static void deleteById(CqlSession session, Long isbn) {
        session.execute(QueryBuilder.deleteFrom(KEYSPACE, COLUMN_FAMILY)
                .where(Relation.column("isbn").isEqualTo(QueryBuilder.literal(isbn))).build());

    }

    private static void findById(CqlSession session, long isbn, Consumer<Row> log) {
        ResultSet resultSet = session.execute(QueryBuilder.selectFrom(KEYSPACE, COLUMN_FAMILY)
                .all().where(Relation.column("isbn").isEqualTo(QueryBuilder.literal(isbn))).build());
        resultSet.forEach(log);
    }

    private static Map<String, Term> createInsertQuery(Object[] parameters) {
        return Map.of("isbn", literal(parameters[0]), "name", literal(parameters[1]),
                "author", literal(parameters[2]),
                "categories", literal(parameters[3]));

    }

}
```

With the previous code, it was possible to perform the basic operations of CRUD with trivial types in Cassandra. Following and evolving our example, we will perform manipulations with a custom Cassandra type, or UDT within the `Category` column family.

```java
public class App3 {

    private static final String KEYSPACE = "library";
    private static final String TYPE = "book";
    private static final String COLUMN_FAMILY = "category";

    public static void main(String[] args) {
        try (CqlSession session = CqlSession.builder().build()) {

            UserDefinedType userType =
                    session.getMetadata()
                            .getKeyspace(KEYSPACE)
                            .flatMap(ks -> ks.getUserDefinedType(TYPE))
                            .orElseThrow(() -> new IllegalArgumentException("Missing UDT definition"));

            UdtValue cleanCode = getValue(userType, 1, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO", "Good practice", "Design"));
            UdtValue cleanArchitecture = getValue(userType, 2, "Clean Architecture", "Robert Cecil Martin", Set.of("OO", "Good practice"));
            UdtValue effectiveJava = getValue(userType, 3, "Effective Java", "Joshua Bloch", Set.of("Java", "OO", "Good practice"));
            UdtValue nosql = getValue(userType, 4, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY)
                    .values(createCondition("Java", Set.of(cleanCode, effectiveJava))).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY)
                    .values(createCondition("OO", Set.of(cleanCode, effectiveJava, cleanArchitecture))).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY)
                    .values(createCondition("Good practice",
                            Set.of(cleanCode, effectiveJava, cleanArchitecture, nosql))).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY)
                    .values(createCondition("NoSQL", Set.of(nosql))).build());

            ResultSet resultSet = session.execute(QueryBuilder.selectFrom(KEYSPACE, COLUMN_FAMILY).all().build());
            for (Row row : resultSet) {
                String name = row.getString("name");
                Set<UdtValue> books = row.getSet("books", UdtValue.class);
                Set<String> logBooks = new HashSet<>();
                for (UdtValue book : books) {
                    long isbn = book.getLong("isbn");
                    String bookName = book.getString("name");
                    String author = book.getString("author");
                    logBooks.add(String.format(" %d %s %s", isbn, bookName, author));
                }
                System.out.println(String.format("The result %s %s", name, logBooks));

            }
        }

    }

    private static UdtValue getValue(UserDefinedType userType, long isbn, String name, String author, Set<String> categories) {
        UdtValue udtValue = userType.newValue();

        TypeCodec<Object> bigIntCodec = CodecRegistry.DEFAULT.codecFor(userType.getFieldTypes().get(0));
        TypeCodec<Object> textCodec = CodecRegistry.DEFAULT.codecFor(userType.getFieldTypes().get(1));
        TypeCodec<Object> setCodec = CodecRegistry.DEFAULT.codecFor(userType.getFieldTypes().get(3));
        udtValue.set("isbn", isbn, bigIntCodec);
        udtValue.set("name", name, textCodec);
        udtValue.set("author", author, textCodec);
        udtValue.set("categories", categories, setCodec);
        return udtValue;

    }

    private static Map<String, Term> createCondition(String name, Set<UdtValue> books) {
        return Map.of("name", QueryBuilder.literal(name), "books", QueryBuilder.literal(books));
    }

}

```


Now we are able to insert a custom Cassandra type `UDT`, that is, in addition to performing the insertion we also recover a `UDT` type within the Cassandra database.


The Cassandra Driver also has the concept of `PreparedStatement` which allows queries to be created dynamically, very similar to relational databases.

```java
public class App4 {

    private static final String KEYSPACE = "library";
    private static final String TYPE = "book";
    private static final String COLUMN_FAMILY = "category";

    public static void main(String[] args) {
        try (CqlSession session = CqlSession.builder().build()) {

            UserDefinedType userType =
                    session.getMetadata()
                            .getKeyspace(KEYSPACE)
                            .flatMap(ks -> ks.getUserDefinedType(TYPE))
                            .orElseThrow(() -> new IllegalArgumentException("Missing UDT definition"));

            UdtValue cleanCode = getValue(userType, 1, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO", "Good practice", "Design"));
            UdtValue cleanArchitecture = getValue(userType, 2, "Clean Architecture", "Robert Cecil Martin", Set.of("OO", "Good practice"));
            UdtValue effectiveJava = getValue(userType, 3, "Effective Java", "Joshua Bloch", Set.of("Java", "OO", "Good practice"));
            UdtValue nosql = getValue(userType, 4, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY)
                    .values(createCondition("Java", Set.of(cleanCode, effectiveJava))).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY)
                    .values(createCondition("OO", Set.of(cleanCode, effectiveJava, cleanArchitecture))).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY)
                    .values(createCondition("Good practice",
                            Set.of(cleanCode, effectiveJava, cleanArchitecture, nosql))).build());
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY)
                    .values(createCondition("NoSQL", Set.of(nosql))).build());

            Consumer<Row> log = row -> {
                String name = row.getString("name");
                Set<UdtValue> books = row.getSet("books", UdtValue.class);
                Set<String> logBooks = new HashSet<>();
                for (UdtValue book : books) {
                    long isbn = book.getLong("isbn");
                    String bookName = book.getString("name");
                    String author = book.getString("author");
                    logBooks.add(String.format(" %d %s %s", isbn, bookName, author));
                }
                System.out.println(String.format("The result %s %s", name, logBooks));
            };

            findById(session, "OO", log);
            findById(session, "Good practice", log);
            deleteById(session, "OO");

            PreparedStatement prepare = session.prepare("select * from library.category where name = ?");
            BoundStatement statement = prepare.bind("Java");
            ResultSet resultSet = session.execute(statement);
            resultSet.forEach(log);
        }

    }

    private static void findById(CqlSession session, String name, Consumer<Row> log) {
        ResultSet resultSet = session.execute(QueryBuilder.selectFrom(KEYSPACE, COLUMN_FAMILY)
                .all().where(Relation.column("name").isEqualTo(QueryBuilder.literal(name))).build());
        resultSet.forEach(log);
    }

    private static void deleteById(CqlSession session, String name) {
        session.execute(QueryBuilder.deleteFrom(KEYSPACE, COLUMN_FAMILY)
                .where(Relation.column("name").isEqualTo(QueryBuilder.literal(name))).build());

    }

    private static UdtValue getValue(UserDefinedType userType, long isbn, String name, String author, Set<String> categories) {
        UdtValue udtValue = userType.newValue();
        TypeCodec<Object> bigIntCodec = CodecRegistry.DEFAULT.codecFor(userType.getFieldTypes().get(0));
        TypeCodec<Object> textCodec = CodecRegistry.DEFAULT.codecFor(userType.getFieldTypes().get(1));
        TypeCodec<Object> setCodec = CodecRegistry.DEFAULT.codecFor(userType.getFieldTypes().get(3));
        udtValue.set("isbn", isbn, bigIntCodec);
        udtValue.set("name", name, textCodec);
        udtValue.set("author", author, textCodec);
        udtValue.set("categories", categories, setCodec);
        return udtValue;
    }

    private static Map<String, Term> createCondition(String name, Set<UdtValue> books) {
        return Map.of("name", QueryBuilder.literal(name), "books", QueryBuilder.literal(books));
    }
}
```


With that, we performed a complete example with the Cassandra Driver, and completed the entire CRUD cycle within the `Category` column family. Using the Driver is very good since we have access to all queries within the CQL, however, this ends up performing great, after all, for each operation, we have to convert to the Java type at all times. 

Is there any way to decrease the amount of code to work with Cassandra? The answer is yes, Cassandra also has tools that work to convert entities into native codes: Mappers.


## Using the Mapper

For our first contact, the Mapper will be used, which is maintained by the same company that maintains the Driver. It is a layer above the Driver layer.

The mapper generates the boilerplate to execute queries and convert the results into application-level objects.

It is published as two artifacts:

- the `java-driver-mapper-processor`module is **only needed in the compile classpath**, your application doesn’t need to depend on it at runtime.
- the `java-driver-mapper-runtime` module is a regular runtime dependency.

The mapper’s annotation processor hooks into the Java compiler, and generates additional source files from your annotated classes before the main compilation happens. It is contained in the `java-driver-mapper-processor` artifact.

As a reminder, there is also a `java-driver-mapper-runtime` artifact, which contains the annotations and a few utility classes. This one is a regular dependency, and it is required at runtime.


```xml
<dependencies>
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-mapper-runtime</artifactId>
    <version>${data.stax.version}</version>
</dependency>
...
<dependencies>

<build>
<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.8.1</version>
        <configuration>
            <source>11</source>
            <target>11</target>
            <annotationProcessorPaths>
                <path>
                    <groupId>com.datastax.oss</groupId>
                    <artifactId>java-driver-mapper-processor</artifactId>
                    <version>${data.stax.version}</version>
                </path>
            </annotationProcessorPaths>
        </configuration>
    </plugin>
</plugins>
</build>
```

For the same example that performs the manipulation of the `Book` column family, the first step is mapping, which is nothing more than an entity whose attributes are noted. When you see the code, you can see that the annotations are very similar for those who came from the JPA world:

* The annotation `Entity` is to indicate that the class is a column family.
* `PartitionKey` indicates that that attribute plays a special role within the column family which is the primary key.


```java
@Entity
public class Book {

    @PartitionKey
    private Long isbn;

    private String name;

    private String author;

    private Set<String> categories;

    //getters and setters
}
```

In the first contact with Mapper, the code reduction is impressive. The highlight for this example are the two new classes that appear:

`BookDao` which serve both to manage instances and to facilitate communication between CQL and a Java object respectively.

Now, let's delve even deeper, we still need to do a search for the key and also create dynamic queries with `BookDao`.

```java
@Dao
public interface BookDao {

    @Select
    Book findById(Long id);

    @Insert
    void save(Book book);

    @Delete
    void delete(Book book);

    @Query("SELECT * FROM library.book")
    PagingIterable<Book> map();

    @Insert
    CompletableFuture<Book> saveAsync(Book book);

    @Select
    CompletableFuture<Book> findByIdAsync(Long id);
}


@Mapper
public interface InventoryMapper {
    @DaoFactory
    BookDao getBookDao(@DaoKeyspace CqlIdentifier keyspace);

    @DaoFactory
    CategoryDao getCategoryDao(@DaoKeyspace CqlIdentifier keyspace);
}


public class App {

    private static final String KEYSPACE = "library";

    public static void main(String[] args) {
        try (CqlSession session = CqlSession.builder().build()) {

            InventoryMapper inventoryMapper = new InventoryMapperBuilder(session).build();
            BookDao mapper = inventoryMapper.getBookDao(CqlIdentifier.fromCql(KEYSPACE));
            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Set.of("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Set.of("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            mapper.save(cleanCode);
            mapper.save(cleanArchitecture);
            mapper.save(effectiveJava);
            mapper.save(nosql);

            for (Book book : mapper.map()) {
                System.out.println("The result: " + book);
            }
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

public class App2 {

    private static final String KEYSPACE = "library";

    public static void main(String[] args) {
        try (CqlSession session = CqlSession.builder().build()) {

            InventoryMapper inventoryMapper = new InventoryMapperBuilder(session).build();
            BookDao mapper = inventoryMapper.getBookDao(CqlIdentifier.fromCql(KEYSPACE));

            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Set.of("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Set.of("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            mapper.save(cleanCode);
            mapper.save(cleanArchitecture);
            mapper.save(effectiveJava);
            mapper.save(nosql);

            Book book = mapper.findById(1L);
            System.out.println("Book found: " + book);

            mapper.delete(book);
            System.out.println("Book found: " + mapper.findById(1L));

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

Our code creates four types of books and persists within the database. Then, search for the key and, finally, create a query with `BookDao` and execute it without worrying about the conversion to our `Book` entity.

But we will not stop there. We will now create an interaction with the `UDT` types within the `category` column family.


```java
@Entity
public class Category {

    @PartitionKey
    private String name;

    private Set<BookType> books;

    //getters and setters

}

@Entity
public class BookType {

    private Long isbn;

    private String name;

    private String author;

    private Set<String> categories;

    //getters and setters
}

@Dao
public interface CategoryDao {

    @Select
    Category findById(String id);

    @Insert
    void save(Category book);

    @Delete
    void delete(Category book);

    @Query("SELECT * FROM library.category")
    PagingIterable<Category> map();

}

```


Modeling created and ready for interaction in the database with the `UDT` type that will be represented by the `Category` class, this class is very similar to the `Book` entity, which we created earlier. A point to note is that the notes are quite intuitive.

```java
public class App3 {

    private static final String KEYSPACE = "library";

    public static void main(String[] args) {
        try (CqlSession session = CqlSession.builder().build()) {

            InventoryMapper inventoryMapper = new InventoryMapperBuilder(session).build();
            CategoryDao mapper = inventoryMapper.getCategoryDao(CqlIdentifier.fromCql(KEYSPACE));

            BookType cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Set.of("Java", "OO"));
            BookType cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Set.of("Good practice"));
            BookType effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Set.of("Java", "Good practice"));
            BookType nosqlDistilled = getBook(4L, "Nosql Distilled", "Martin Fowler", Set.of("NoSQL", "Good practice"));

            Category java = getCategory("Java", Sets.newHashSet(cleanCode, effectiveJava));
            Category oo = getCategory("OO", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture));
            Category goodPractice = getCategory("Good practice", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture, nosqlDistilled));
            Category nosql = getCategory("NoSQL", Sets.newHashSet(nosqlDistilled));

            mapper.save(java);
            mapper.save(oo);
            mapper.save(goodPractice);
            mapper.save(nosql);

            PagingIterable<Category> categories = mapper.map();
            StreamSupport.stream(categories.spliterator(), false).forEach(System.out::println);
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

The code is very similar to the interaction of the Book class, the only exception is the use of the `Category.class` as a parameter inside the `mapper`, and instances of type `Category` will be created for the persistence of the information.



> You can use this mapper integrated with Quarkus, take look at this link: [https://quarkus.io/guides/cassandra](https://quarkus.io/guides/cassandra)

### Conclusion

In this chapter, the integration framework between Cassandra and a Java application was obtained. It was also possible to see the difference between a framework that performs low-level communication, similar to JDBC, and a Mapper that, while facilitating development, increases responsibility so that the developer does not make mistakes when forgetting that there is a break paradigm between the application and the database.