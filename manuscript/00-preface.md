# The new era of applications comes with new databases

The business world grows exponentially with several opportunities and companies competing with each other. The dispute between customers is something that is no longer defined by the company's location, but by its applications' response time. Several studies reveal that millisecond improvements are enough to acquire new customers and retain existing ones, in addition to overcoming the competition.

In this battle for milliseconds, several paradigms and frameworks have birthed, however, all this improvement would be useless if the persistence of information was still the bottleneck.

With the need for better performance, NoSQL databases emerged to facilitate performance and information distribution. Along with this concept, Apache Cassandra was born, the elastic NoSQL database, fault-tolerant with a high level of performance, achieving success in the largest companies in the world such as Netflix, GitHub, eBay among others. Cassandra is a non-relational database developed by Facebook, and today is an open-source column family type NoSQL database within the Apache Foundation. In general, the objective of this book is to talk about Cassandra, its concepts, and its applicability with Java. To cover all of the concepts, the book was sectioned into a total of nine chapters.

The scope of the first chapter is the introduction to non-relational databases. The chapter addresses the challenges of distributed databases, their limitations with the CAP theorem, which is NoSQL and its types, as well as the comparison between NoSQL and the already consolidated relational database.

In the second chapter, we will cover Cassandra's concepts, such as its hierarchy, reading, and writing, its functioning in the node, and its orchestration within a cluster. We will see different ways of installing Cassandra - manual installation, Docker, and clustering with docker-compose.

Using queries is very common within a database. In the relational world, the developer is familiar with SQL. Cassandra has its own query language: Cassandra Query Language or CQL, and note that, one of its biggest differentials is the similarity with SQL. Learning how to create keyspace, column family, perform CRUD operations (creation, recovery, update and delete data) that the developer loves so much are topics in this chapter.

There are several similarities between Cassandra's communication language, CQL, and SQL. However, the databases similarities end there. It is worth remembering that conceptually,taking into account the CAP theorem, the relational database and Cassandra are in different aspects. Thus, performing the modeling of Cassandra in the same way that it takes place within the relational database will have a high-performance impact, not to mention that you'd be trying to use an airplane, as if it were a car. The topics of this session are the explanation and tips to perform the modeling in Cassandra and some simple principles for daily modeling.

After all of Cassandra's communication concepts and modeling tips, the next step is the integration with Java. Within this communication between Java and Cassandra, the first point to be covered is how to implement it with the DataStax Driver whose API is similar to the JDBC driver. An important point within the DataStax Driver is that once the developer is familiar with CQL, all communication takes place in a very fluid way. Business are generally very object-oriented, the content covers the impacts of Mappers, frameworks whose objective is to carry out the mapping between Cassandra and the objects of a business entity, positively as productivity and negatively with possible performance problems. Among these mappers, we will use DataStax Mapper, Spring Data Cassandra, Hibernate OGM Cassandra, and Eclipse JNoSQL, and to facilitate the comparison between them, the same example will be used, which is based on a solution for a library.



To access the source code of the book: [ https://github.com/otaviojava/cassandra-java-code](https://github.com/otaviojava/cassandra-java-code)

## Target audience / prerequisites

Java developers, enthusiastic and familiar with NoSQL technology and who want to learn or deepen their knowledge about Cassandra.

## About the author
 
Otavio Santana is a passionate software engineer with a focus on Java technology. He has experience in multilingual persistence and high-performance applications in areas such as finance, social networks, and e-commerce. He works on several Java specifications as part of the expert group or as a specification leader and is an executive member of the Java Community Process. He works on several open-source projects from both the Apache and the Eclipse Foundation such as Apache Tamaya, Eclipse JNoSQL, Eclipse MicroProfile, and Jakarta EE in addition to being a present member of the community helping the JUG and in several conferences around the world such as JavaOne, OracleCode, Devoxx, Qcon among others. Due to his efforts for the community and open-source, he received several recognition awards such as the JCP Outstanding Award, Member of the year and innovative JSR, Duke’s Choice Award, Java Champion, Oracle Groundbreaker, among others.

### About the Reviewer

Karina M. Varela is a Senior Technical Marketing Manager at Red Hat and an expert on the matter of Business Automation. She brings a solid background in development, architecting delivering, and troubleshooting critical software in enterprise environments of different 