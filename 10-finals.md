# Final considerations

Okay, after all these chapters, we are approaching the end of the book. The purpose of this concluding part of the series is to show the latest tips and suggestions to anyone willing to disassociate Cassandra's mares and develop their next experiences.

## Search engine

As already discussed in the modeling chapter, in the ideal world, all queries should be executed from the partition key, that is, in addition to denormalization, you should avoid using resources like `ALLOW INDEXING`, otherwise Cassandra will have an impact performance. This duplication of data will have a performance advantage at the time of reading, but it will have an impact on information management.

The bus engine has several advantages over Cassandra. The first of these is the fact that you can search fields other than the partition key. The second point is what you need with the search engine for great benefits for the product as well. Imagine that, inside the book now, you have a summary, however, the text is in HTML. Within a search engine, information, when entered, is treated so that, when a search is performed, it is optimized. For example, considering the `Elasticsearch` process, perform the following steps:


* **Character filters**: is the first contact of the information. It receives a character stream and returns a character stream. In our example, the HTML format will be removed in the same way as the captured text as follows: `Know the Clean Code, the book of good programming practices`.
* **Tokenizer**: the tokenization process in which, given a stream of characters, it is converted into a token, which usually results in a word. For example, a book description turns into tokens like "[" Meet "," o "," Clean "," Code "," o "," book "," de "," good "," practices ", ”From”, ”schedule”] `.
* **Token Filter**: in this step, given a sequence of tokens, it will be responsible for removing (removing common words, such as *stop words*, such as: "de", "o"), adding (adding preferences: programming and software ), or modify tokens (convert lectures to lowercase or remove word actions).

So, when using, for example, the word “programming” inside the search engine, the information is already pre-processed, so you won't have to scan or search for information on each line. In addition to the new horizon that the search engine allows, it is possible to add synonyms, add weights in the search fields (if the word is in the title, it will be more relevant than when it is in the description), and consider the possible spelling errors that may be made by the user accidentally, etc.


In the search engine world there are a few options:

* Apache Lucene: when Hibernate Search was mentioned. It is open source and is within the Apache Foundation, being an API for searching and indexing documents. See the great project experience cases that can be found on Wikipedia.
* Apache Solr: or simply Solr is also an open-source project maintained by Apache Foundation. Among the main characteristics, we can mention text searches, indexing at runtime, and the possibility of execution in Clusters. A curiosity that is the Apache Solr is developed on top of Apache Lucene.
* Elasticsearch: Elasticsearch, like Apache Solr, is a search engine based on the Apache Lucene, however, it is not from the Apache Foundation (although licensed as Apache 2.0). It also can work with the search engine as a cluster and is currently considered the most popular and used search engine in the world.

There are even some projects that aim only at the integration between the search engine with Cassandra and the cases of Solandra (integration of Cassandra with Sorl) and Elassandra.

## Validating data with Bean Validation

In an application, it is very common for some fields to have a validation to guarantee the consistency of the application. For example, email fields. Cassandra has no validation feature and to save this gap, one option is to use Bean Validation, which allows you to place restrictions on Java objects from annotations.

Bean Validation has a large number of native annotations that use a high degree of validity, such as minimum or maximum text size, email validation, mandatory field, among others.


```java
public class user {
@NotNull
@Email
private string email;
}
```

In addition to several validation options, it is also possible to create a new one, so that Bean Validation is a great option if the data needs some kind of validation for data consistency.

## Conducting integration tests with Cassandra


In the world of software engineering, testing is a fundamental part of building any program. This guarantees the current code is stable, testing quickly and without human interference, in addition to having greater security to refactor and add new resources within the system. Integration tests are one of the types of tests on a system with which they are checked against a group of components, for example, communication between Java code and Cassandra. In this topic, I will mention two good examples for performing the tests.

The first of these is the Cassandra unit, which offers a similar way to DBUnit, however, instead of a relational database, it uses Cassandra.

A major advantage of using construction technologies is that we don't have to worry about dependencies.

```xml
<dependency>
    <groupId>org.cassandraunit</groupId>
    <artifactId>cassandra-unit</artifactId>
    <version>3.3.0.2</version>
    <scope>test</scope>
</dependency>
```

It is possible to notice that in this small test, it was necessary to add a method that raises an embedded instance of Cassandra and another that cleans.

```java
public class SampleCassandraTest {


    @BeforeAll
    public static void before() throws InterruptedException, IOException, TTransportException {
        EmbeddedCassandraServerHelper.startEmbeddedCassandra();
    }
    
    @AfterAll
    public static void end() {
        EmbeddedCassandraServerHelper.cleanEmbeddedCassandra();
    }
    
    @Test
    public void test() {
        Cluster.Builder builder = Cluster.builder();
        builder.addContactPoint("localhost").withPort(9142);
        Cluster cluster = builder.build();
        Session session = cluster.newSession();
        ResultSet resultSet = session.execute("SELECT * FROM system_schema.keyspaces;");
        Assertions.assertNotNull(resultSet);
        cluster.close();
    }

}

```

TestContainers is a Java library that allows the integration of Junit with any docker image. Thus, it is possible to run Cassandra through a docker image. Dependency is something really simple to add. 

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.9.1</version>
    <scope>test</scope>
</dependency>
```


With the dependency configured, the coding is now very simple to implement. A `GenericContainer` class represents a docker container. In its constructor, there is a `String` which is the name of the image that the `testcontainer` searches for in the `dockerhub`. With that, the next step is to export the connection port to the database, in this case, port `9072`. 

```java
public class SampleCassandraContainerTest {

    @Test
    public void test() {
    
        GenericContainer cassandra =
                new GenericContainer("cassandra")
                        .withExposedPorts(9042)
                        .waitingFor(Wait.defaultWaitStrategy());
    
        cassandra.start();
        Constructor Cluster.Builder = Cluster.builder();
        builder.addContactPoint(cassandra.getIpAddress()).withPort(cassandra.getFirstMappedPort());
        Cluster cluster = builder.build();
        Session session = cluster.newSession();
        ResultSet resultSet = session.execute("SELECT * FROM system_schema.keyspaces;");
        Assertions.assertNotNull(resultSet);
        cluster.close();
    }
}
```

## Experimenting with other flavors of Cassandra

An important point to note is that there are databases that compete with Cassandra either to be the same type of database or to solve, practically, the same problems. It is very important that the reader also knows about these databases:

* DataStax DSE: is a corporate version, closed and paid for, owned by DataStax. It supports everything Cassandra currently does and adds new features like analytics and an already integrated search engine. Another interesting point is that the DSE is a *multi-model* database, in other words, it is a NoSQL database that supports yet another type of NoSQL database (key-value, column family, and graphs).
* ScillaDB: a database which focus is the possibility of maintaining full compatibility with Cassandra, however, with extremely superior performance. In theory, it is possible to take an application in Cassandra and change to this database without any impact. It also has official images inside DockerHub. Once the reader entered Cassandra's world is worth the try.

### Conclusion

With that, the final advice and the last tips to proceed with Cassandra were presented. Features such as search engine, data validation with bean validation, and tools for integration testing are valuable resources in which it is recommended that the reader delve much deeper. The purpose of this chapter was simply to arouse curiosity to continue evolving Cassandra by integrating it with other tools. I hope this book has helped to show how simple Cassandra is to use and prove that it is less than a step away from integrating with Java applications.