# Modeling your application with Cassandra


Within data science, modeling is certainly one of the most enigmatic and most challenging points. An error at this point will mean a performance impact on both reading and writing information in the database. In the relational world, those who develop it are already accustomed to the concept of normalization - which is a set of rules that mainly aim at organizing within the relational database to avoid data redundancy. The point to be discussed is that, when relational databases emerged, they were focused on avoiding redundancy, after all, it was a big challenge since the price of storage was something very expensive. For example, in 1980 a server that stored twenty-six megabytes had a cost of almost five thousand dollars, today, a terabyte costs less than fifty dollars.

![A 5MB hard drive in 1956 manufactured by IBM.](imagens/ibm_history_server.png "A 5MB hard drive in 1956 manufactured by IBM.")

The consequence of normalization is that, with great complexity of data and variety, to avoid data replication, it is necessary to perform a high amount of joins, the union of several tables, and consequently, there is an increase in the response time of these queries. The current challenge is in response time and no longer in storage, as it was before. The purpose of this chapter is to demonstrate tips and motivations for modeling inside Cassandra.

## Modeling tips

Unlike relational databases in which normalization rules exist, within Cassandra, the modeling principles are defined much more by the context of the application or data volume. In other words, the same e-commerce system can be modeled differently depending on the resources and size of each of these businesses. As most developers start with relational, the first tips will be precisely to understand what *are* not the goals of modeling with Cassandra:

* Minimize the number of readings: within Cassandra, writing is relatively inexpensive and efficient, different from reading; therefore, prioritize writing and try to read the information by the unique identifier as much as possible.
* Normalization or minimizing duplicate data: denormalization and duplication of data are the best friends for modeling within Cassandra. Making reading as easy as possible is always the best strategy within Cassandra.
* Emulate: don't try to emulate or in any way simulate the relational databases within Cassandra. The result will be similar to trying to use a fork as if it was a knife.

A good first step to start modeling is to know what queries the application needs. Thus, stressing, creating families of columns that satisfy that request in just one query is the big goal. There are several modeling cases within the Cassandra world, for the next topic, the next step will be to demonstrate some modeling examples.

### One-to-one cases

Imagine the following scenario:
Each book (ISBN, name, and year) has its author (name and country).

The first question for modeling:
What queries do we want to support?
For this case, we would like to search for books by its ISBN since in this case there is no reason to have a column family for authors.

```sql
CREATE KEYSPACE IF NOT EXISTS library WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

DROP COLUMNFAMILY IF EXISTS library.book;
DROP TYPE IF EXISTS library.author;

CREATE TYPE IF NOT EXISTS library.author (
    name text,
    country text
);

CREATE COLUMNFAMILY IF NOT EXISTS library.book (
    isbn bigint,
    name text,
    year int,
    author author,
    PRIMARY KEY (isbn)
);
```

In the first example, there was the creation of the author type of which he is part of.

```sql
INSERT INTO library.book (isbn, name, year, author) values (1, 'Clean Code', 2008, {name: 'Robert Cecil Martin', country: 'USA'});
INSERT INTO library.book (isbn, name, year, author) values (2, 'Clean Architecture', 2017, {name: 'Robert Cecil Martin', country: 'USA'});
INSERT INTO library.book (isbn, name, year, author) values (3, 'Agile Principles, Patterns, and Practices in C #', 2002, {name: 'Robert Cecil Martin', country: 'USA'});
INSERT INTO library.book (isbn, name, year, author) values (4, 'Java EE 8 Cookbook', 2018, {name: 'Elder Moraes', country: 'Brazil'});
INSERT INTO library.book (isbn, name, year, author) values (5, 'Effective Java', 2001, {name: 'Joshua Bloch', country: 'USA'});
INSERT INTO library.book (isbn, name, year, author) values (6, 'Java Puzzlers: Traps, Pitfalls, and Corner Cases', 2005, {name: 'Joshua Bloch', country: 'USA'});
```

In the scenario, it is possible to see the impacts on the duplication of information, in the case of the authors. However, changing an author's name and nationality tends to be extremely rare.

### Cases one to N

Following the previous example, in the book-author relationship, this example will increase the number of authors, after all, some books have more than one writer. With that, there will be a list of books for N authors. However, as the purpose of the reading has not changed, the modeling will not change.

```sql
CREATE KEYSPACE IF NOT EXISTS library WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

DROP COLUMNFAMILY IF EXISTS library.book;
DROP TYPE IF EXISTS library.author;

CREATE TYPE IF NOT EXISTS library.author (
    name text,
    country text
);

CREATE COLUMNFAMILY IF NOT EXISTS library.book (
    isbn bigint,
    name text,
    year int,
    authors set <frozen <author>>,
    PRIMARY KEY (isbn)
);
// INSERT
INSERT INTO library.book (isbn, name, year, authors) values (1, 'Design Patterns: Elements of Reusable Object-Oriented Software', 1994,
{{name: 'Erich Gamma', country: 'Switzerland'}, {name: 'John Vlissides', country: 'USA'}});
INSERT INTO library.book (isbn, name, year, authors) values (2, 'The Pragmatic Programmer', 1999,
{{name: 'Andy Hunt', country: 'Switzerland'}, {name: 'Dave Thomas', country: 'England'}});
```



The impact remains similar to the previous one: a high degree of duplicate data. This will be of great inconvenience for those coming from the relational world, however, it is important to note: **denormalization will be Cassandra's modeling best friend**.

### Cases N to N

In this last example, the goal is to create an N to N relationship. Following the business scope of a library, it is possible to think about the relationship between magazines and articles because a magazine has N articles in the same way that an article can be in N magazines. Regardless, in Cassandra the question is the same: what queries does the database want to support? In this scenario there will be two:

* Search the magazine by ISSN
* Search the article by title


```sql
CREATE KEYSPACE IF NOT EXISTS library WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

DROP COLUMNFAMILY IF EXISTS library.magazine;
DROP COLUMNFAMILY IF EXISTS library.article;
DROP TYPE IF EXISTS library.author;
DROP TYPE IF EXISTS library.magazine;
DROP TYPE IF EXISTS library.article;

CREATE TYPE IF NOT EXISTS library.author (
    name text,
    country text
);

CREATE TYPE IF NOT EXISTS library.magazine (
    issn bigint,
    name text,
    year int,
    author frozen <author>
);

CREATE TYPE IF NOT EXISTS library.article (
    name text,
    year int,
    author frozen <author>
);

CREATE COLUMNFAMILY IF NOT EXISTS library.magazine (
   issn bigint,
   name text,
   editor author,
   articles set <frozen <article>>,
   PRIMARY KEY (issn)
);

CREATE COLUMNFAMILY IF NOT EXISTS library.article (
   title text,
   year int,
   author author,
   magazines set <frozen <magazine>>,
   PRIMARY KEY (title, year)
);
// INSERT
INSERT INTO library.magazine (issn, name, editor, articles) values (1, 'Java Magazine', {name: 'Java Editor', country: 'USA'}, {{name: 'Jakarta EE', year: 2018 , author: {name: 'Elder Moraes', country: 'Brazil'}},
{name: 'Cloud and Docker', year: 2018, author: {name: 'Bruno Souza', country: 'Brazil'}}});
```

In this case, it is quite simple since both entities, magazines, and articles, do not need to make changes.

### Conclusion
With that, modeling concepts were presented within a Cassandra database, demonstrating how it is different from a relational database. Its core concept is precisely to write as much as possible since it tends to be a cheap operation to precisely decrease the number of operations in reading. In the ideal world, it is possible to create queries without using indexes either as a secondary or with the `ALLOW FILTERING` feature. However, there will be a huge job to manage the duplication of data that will exist.