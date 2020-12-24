# Final considerations

Okay, after all these chapters we are reaching the end of the book. The purpose of this conclusion of the series is to show the latest tips and suggestions to anyone who is willing to disassociate Cassandra's mares and develop their next experiences.

## Search engine

As already discussed in the modeling chapter, in the ideal world, all queries should be executed from the partition key, that is, in addition to denormalization, you should avoid using resources like ALLOW INDEXING`, otherwise Cassandra will have an impact performance. This duplication of data will have the performance advantage at the time of reading, but it will have the impact of information management.

The bus engine has several advantages over Cassandra. The first of these is the fact that you can conduct a search in fields other than the partition key. The second point is what you need with the search engine for great benefits for the product as well. Imagine that, inside the book now, you have a summary, however, or the text is in HTML. Within a search engine, information when entered is treated so that, when a search is performed, it is optimized. For example, considering the Elasticsearch process, perform the following steps:


* **Character filters**: is the first contact of the information. It receives a character stream and returns a character stream. In our example, the html format will be removed in the same way as the captured text as follows: `Know the Clean Code, the book of good programming practices`.
* **Tokenizer**: the tokenization process in which, given a stream of characters, it is converted into a token, which usually results in a word. For example, a book description turns into tokens like "[" Meet "," o "," Clean "," Code "," o "," book "," de "," good "," practices ", ”From”, ”schedule”] `.
* **Token Filter**: in this step, given a sequence of tokens, he will be responsible for removing (removing common words, such as _stop words_, such as: "de", "o"), adding (adding preferences: programming and software ) or modify tokens (convert lectures to lowercase or remove word actions).

So, when using, for example, the word “programming” inside the search engine, the information is already pre-processed as, so you won't have to scan or search for information on each line. In addition to the new horizon that the search engine allows, it is possible to add synonyms, add weights in the search fields (if the word is in the title, it will be more relevant than when it is in the description) and consider the possible spelling errors that may be made by the user accidentally etc.


In the search engine world there are a few options:

* Apache Lucene: when Hibernate Search was mentioned, it was mentioned a little about this project. It is open source and is within the Apache Foundation, being an API for searching and indexing documents. See the great project experience cases that can be found on Wikipedia.
* Apache Sorl: or simply Sorl is also an open source project maintained by Apache Foundation. Among the main characteristics, we can mention text searches, indexing at runtime and possibility of execution in Clusters. A curiosity that is the Apache Sorl is developed on top of Apache Lucene.
* Elasticsearch: Elasticsearch, like Apache Sorl, is a search engine based on the Apache Lucene, however, it is not from the Apache Foundation (although licensed as Apache 2.0). It also has the ability to work with the search engine as a cluster and is currently considered the most popular and used search engine in the world.

There are even some projects that aim only at the integration between the search engine with Cassandra and the cases of Solandra (integration of Cassandra with Sorl) and Elassandra.

## Validating data with Bean Validation

In an application it is very common for some fields to have a validation to guarantee the consistency of the application. For example, email fields. Cassandra has no validation feature and to save this gap, one option is to use Bean Validation, which allows you to place restrictions on Java objects from annotations.

Bean Validation has a large number of native annotations that use a high degree of validity, such as minimum or maximum text size, email validation, mandatory field, among others.


```java
public class user {
@Not null
@The e-mail
private string email;
}
```

In addition to several validation options, it is also possible to create a new one, so that Bean Validation is a great option if the data needs some kind of validation for data consistency.

## Conducting integration tests with Cassandra


In the world of software engineering, testing is a fundamental part of building any program. It is from them that it is possible to guarantee the current code if it is stable, that it tests quickly and without human interference, in addition to having greater security to refactor and add new resources within the system. Integration tests are one of the types of tests on a system with which they are checked against a group of components, for example: a communication between Java code and Cassandra. In this topic, I will quote two good examples for performing the tests.

The first of these is the Cassandra unit, which offers a similar way to DBUnit, however, instead of a relational bank, it uses Cassandra. He lifted an onboard Cassandra ready for use in his tests.

A major advantage of using construction technologies is that we don't have to worry about dependencies. For example, it is possible to use the integrated Cassandra Unit with June 5th.

```xml
<dependency>
    <groupId>org.cassandraunit</groupId>
    <artifactId>cassandra-unit</artifactId>
    <version>3.3.0.2</version>
    <scope>test</scope>
</dependency>
```

It is possible to notice that in this small test it was necessary to add a method that raises an embedded instance of Cassandra and another that cleans.

```java
public class SampleCassandraTest {


    @Before everything
    public static void before () throws InterruptedException, IOException, TTransportException {
        EmbeddedCassandraServerHelper.startEmbeddedCassandra ();
    }
    
    @After all
    public static void end () {
        EmbeddedCassandraServerHelper.cleanEmbeddedCassandra ();
    }
    
    @Test
    public nullity test () {
        Constructor Cluster.Builder = Cluster.builder ();
        builder.addContactPoint ("localhost"). withPort (9142);
        Cluster cluster = builder.build ();
        Session session = cluster.newSession ();
        ResultSet resultSet = session.execute ("SELECT * FROM system_schema.keyspaces;");
        Assertions.assertNotNull (resultSet);
        cluster.close ();
    }

}

```

The test containers are a Java library that allows the integration of Junit with any docker image. Thus, it is possible to run Cassandra through a docker. Dependency is something really simple to add. After that, the difference with the previous integration test is dependent and without infrastructure code to lift Cassandra, this time, via container.

```xml
<dependency>
    <groupId> org.testcontainers </groupId>
    <artifactId> testcontainers </artifactId>
    <version> 1.9.1 </version>
    <scope> test </scope>
</dependency>
```


With a defined dependency or code, it proves to be very simple. A `GenericContainer` class represents a container docker. In its constructor, there is a `String` which is the name of the image that the testcontainer searches for in the dockerhub. That done, the next step was to export the connection port to the database, in this case, the port `9072`. How to do the integration test, use the container as soon as it is available, and this is the reason that will define the default strategy you expect or the new container to execute the next line only when the instantiated database is on foot. With the database ready, the next step is to perform the necessary test.

```java
public class SampleCassandraContainerTest {

    @Test
    public nullity test () {
    
        GenericContainer cassandra =
                new GenericContainer ("cassandra")
                        .withExposedPorts (9042)
                        .waitingFor (Wait.defaultWaitStrategy ());
    
        cassandra.start ();
        Constructor Cluster.Builder = Cluster.builder ();
        builder.addContactPoint (cassandra.getIpAddress ()). withPort (cassandra.getFirstMappedPort ());
        Cluster cluster = builder.build ();
        Session session = cluster.newSession ();
        ResultSet resultSet = session.execute ("SELECT * FROM system_schema.keyspaces;");
        Assertions.assertNotNull (resultSet);
        cluster.close ();
    }
}
```

## Experimenting with other flavors of Cassandra

An important point is that there are databases that compete with Cassandra to be the same type of database that is to solve, practically, the same problems. It is very important that the reader also knows these databases:

* DataStax DSE: is a corporate version, closed and paid for by DataStax. It supports everything Cassandra currently does and adds new features like analytics and an already integrated search engine. Another interesting point is that the DSE is a _multi-model_ database, in other words, it is a NoSQL database that supports yet another type of NoSQL database (key-value, column family and graphs).
* ScillaDB: a database whose focus is the possibility of maintaining full compatibility with Cassandra, however, with an extremely superior performance. In theory, it is possible to take an application in Cassandra and change this database without any impact. It also has official images inside the dockerhub. Once the reader entered the world Cassandra is worth the try.

### Conclusion

With that, the final advice and the last tips to proceed with Cassandra were presented. Features such as search engine, data validation with bean validation and tools for integration testing are valuable resources in which it is recommended that the reader go much deeper. The purpose of this chapter was just to arouse curiosity to continue evolving Cassandra, integrating with other tools. I hope this book has helped to show how simple Cassandra is to use and is less than a step away from integrating with Java applications.