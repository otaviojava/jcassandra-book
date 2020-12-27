# Knowing the CQL

Communication with the database is certainly the most trivial thing in a database.
Cassandra offers its language for communicating with the database, *Cassandra Query language,* or CQL. CQL has a syntax close to SQL, however, in a simpler way since there are no concepts like *joins* (*inner join*, *left join*, among others) within Cassandra's searches. A developer who already knows SQL will have a much smaller learning curve to learn this Cassandra language.
This chapter will aim to talk a little about this language.

> For these operations, we will use the client cqlsh that we saw in the previous chapter.

## Keyspace

Our first stop at CQL is the creation of the largest structure within Cassandra's hierarchy, *keyspace*. It is during the creation of the keyspace that the replication strategy and the replication factor for the data are defined. The creation template is shown below:

```sql
create_keyspace_statement :: = CREATE KEYSPACE [IF NOT EXISTS] keyspace_name WITH options
```

For example, when creating a keyspace of `library`, bookstore in English:

```sql
CREATE KEYSPACE library WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};
// or
CREATE KEYSPACE library
           WITH replication = {'class': 'NetworkTopologyStrategy', 'DC1': 1, 'DC2': 3}
            AND durable_writes = false;
```


Once created, it is possible to check the keyspaces created in Cassandra and for that, the first query will be performed. It is possible to see the similarity with the relational database. We will talk more about seeking information on the following items.


```sql
cqlsh> select * from system_schema.keyspaces where keyspace_name = 'library';

 keyspace_name | replication
--------------- + ---------------------------------- --------------------
       library | {'class': 'SimpleStrategy', 'replication_factor': '3'}

(1 rows)
```

To make any changes to the keyspace created, it is necessary to use the **alter keyspace**. The template is shown below:

```sql
alter_keyspace_statement :: = ALTER KEYSPACE keyspace_name WITH options
```

For example, changing the keyspace created earlier:


```sql
ALTER KEYSPACE Excelsior WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 4};
```

It is also possible to remove the keyspace with the **drop keyspace**, in our example:

```sql
DROP KEYSPACE library;
```

If the user tries to create or remove the same table more than once, an error message is generated. For example, `Keyspace 'library' already exists or ConfigurationException: Cannot drop non-existing keyspace 'library'`. to create or remove respectively. One way to avoid such an error is to make this change only if it makes sense: just create it if it doesn't exist or only remove it if it exists. For this purpose, the syntax supports this condition:


```sql
DROP KEYSPACE IF EXISTS library;
CREATE KEYSPACE IF NOT EXISTS library WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};
```

## Speaker family

The creation of the column family is similar to the creation of the table, that is, it is where the fields are defined, *partition key*, *clustering key*, the index, and more settings. Just use the `CREATE TABLE` template:

```sql
create_table_statement :: = CREATE TABLE [IF NOT EXISTS] table_name
                            '('
                                column_definition
                                (',' column_definition) *
                                [',' PRIMARY KEY '(' primary_key ')']
                            ')' [WITH table_options]
column_definition :: = column_name cql_type [STATIC] [PRIMARY KEY]
primary_key :: = partition_key [',' clustering_columns]
partition_key :: = column_name
                            | '(' column_name (',' column_name) * ')'
clustering_columns :: = column_name (',' column_name) *
table_options :: = COMPACT STORAGE [AND table_options]
                            | CLUSTERING ORDER BY '(' clustering_order ')' [AND table_options]
                            | options
clustering_order :: = column_name (ASC | DESC) (',' column_name (ASC | DESC)) *
```

Since the keyspace `library` was created, the next step is to create the book column `family, book`:


```sql
CREATE TABLE library.book (
    name text PRIMARY KEY,
       year int
);
```

> It is also possible to check if the column family exists before creating, using `IF NOT EXISTS` similar to keyspace.


It is also possible to make changes to the column family, for example, to add or remove a field within it. For this, it is necessary to use the following template:

```sql
alter_table_statement :: = ALTER TABLE table_name alter_table_instruction
alter_table_instruction :: = ADD column_name cql_type (',' column_name cql_type) *
                             | DROP column_name (column_name) *
                             | WITH options
```

For example, add the `author` field:

```sql
ALTER TABLE library.book ADD author text;
```

If you wanted to remove the same field:

```sql
ALTER TABLE library.book DROP author;
```

It is also possible to destroy the newly created structure.

```sql
drop_table_statement :: = DROP TABLE [IF EXISTS] table_name
```

In our example:

```sql
DROP TABLE IF EXISTS library.book;
```

There are cases where the goal is not to destroy the structure, but to remove all content and maintain the structure. For this, it is possible to truncate the column family, with the command `TRUNCATE`.

```sql
truncate_statement :: = TRUNCATE [TABLE] table_name
```


For example:

Creating the `book` column family and inserting two books:

```sql
CREATE TABLE IF NOT EXISTS library.book (
    name text PRIMARY KEY,
       year int
);
INSERT INTO library.book JSON '{"name": "Effective Java", "year": 2001}';
INSERT INTO library.book JSON '{"name": "Clean Code", "year": 2008}';
```

Executing, previously, the query to search the existing values in the database:

```sql
cqlsh> SELECT * FROM library.book;

 name | year
---------------- + ------
     Clean Code | 2008
 Effective Java | 2001

(2 rows)
TRUNCATE library.book;
cqlsh> SELECT * FROM library.book;

 name | year
------ + ------

(0 rows)

```

> You can also use `COLUMNFAMILY` instead of `TABLE`

### Primary key

Within the column family, the primary key (*primary key*) is the single field and all column families **must** define it. It is from this field that the data will be recovered, so there is an initial concern with it. The primary key can consist of more than one field, however, it will have a different concept than the relational bank. It will be divided into two parts:

* The **partition key**: it is the first part of the primary key that is responsible for the distribution of data through the nodes.

* **clustering columns**: this key has the power to define the order within the table. For example, when creating a family of authors we define the name as key and their books as order.


```sql
CREATE TABLE IF NOT EXISTS library.author (
    name text,
    book text,
    year int,
    PRIMARY KEY (name, book)
);
INSERT INTO library.author JSON '{"name": "Robert Cecil Martin", "book": "Clean Code", "year": 2008}';
INSERT INTO library.author JSON '{"name": "Robert Cecil Martin", "book": "Clean Architecture", "year": 2017}';
INSERT INTO library.author JSON '{"name": "Robert Cecil Martin", "book": "Agile Principles, Patterns", "year": 2006}';
```

When executing the query we will have the following result:

```sql
cqlsh> SELECT * FROM library.author;

 name | book | year
--------------------- + ---------------------------- + ------
 Robert Cecil Martin | Agile Principles, Patterns | 2006
 Robert Cecil Martin | Clean Architecture | 2017
 Robert Cecil Martin | Clean Code | 2008

(3 rows)
```

Another option is to remove the table and create again, this time with the order of the books in decreasing order. Like this:

```sql
CREATE TABLE IF NOT EXISTS library.author (
    name text,
    book text,
    year int,
    PRIMARY KEY (name, book)
)
WITH CLUSTERING ORDER BY (book DESC);

INSERT INTO library.author JSON '{"name": "Robert Cecil Martin", "book": "Clean Code", "year": 2008}';
INSERT INTO library.author JSON '{"name": "Robert Cecil Martin", "book": "Clean Architecture", "year": 2017}';
INSERT INTO library.author JSON '{"name": "Robert Cecil Martin", "book": "Agile Principles, Patterns", "year": 2006}';

cqlsh> SELECT * FROM library.author;

 name | book | year
--------------------- + ---------------------------- + ------
 Robert Cecil Martin | Clean Code | 2008
 Robert Cecil Martin | Clean Architecture | 2017
 Robert Cecil Martin | Agile Principles, Patterns | 2006

(3 rows)
```

Performing the search from the key, name of the author of the book:

```sql
cqlsh> SELECT book FROM library.author WHERE name = 'Robert Cecil Martin';

 book
----------------------------
                 Clean Code
         Clean Architecture
 Agile Principles, Patterns

(3 rows)
```

By default, it is not possible to search for a field other than the key, `partition key`. So, in the example, both the search for the book and the year will give the same error message.


```sql
cqlsh> SELECT * FROM library.author WHERE year = 2017;
InvalidRequest: Error from server: code = 2200 [Invalid query] message = "Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the unpredictability performance, use ALLOW FILTERING"
cqlsh> SELECT * FROM library.author WHERE book = 'Clean Architecture';
InvalidRequest: Error from server: code = 2200 [Invalid query] message = "Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the unpredictability performance, use ALLOW FILTERING"
```

As the error message shows, the only way to execute the query is to add the command `ALLOW FILTERING`, however, this will have serious performance consequences. The way Cassandra executes this command is to retrieve all the lines and then filter for those that do not have the condition value. For example, in a family of columns that has a million rows and 98% of them have the query condition, this will be relatively efficient. However, imagine the case where only one line meets the condition. Cassandra will have traversed 999999 elements linearly and inefficiently to just return one.


```sql
cqlsh> SELECT * FROM library.author WHERE year = 2017 ALLOW FILTERING;
 name | book | year
--------------------- + -------------------- + ------
 Robert Cecil Martin | Clean Architecture | 2017
(1 rows)

cqlsh> SELECT * FROM library.author WHERE book = 'Clean Architecture' ALLOW FILTERING;
name | book | year
--------------------- + -------------------- + ------
 Robert Cecil Martin | Clean Architecture | 2017

(1 rows)
```

> If a query is rejected by Cassandra asking for the filter, resist the use of `ALLOW FILTERING`. Check your modeling, your data, and volumetry, and check what you want to do.
> A second option is the use of secondary indexes, which will be discussed below.


### Static types

Some columns can be considered static within the column family. A static column shares the same value for all rows that have the same partition key value. For example, in the author column family scenario (with name and book), imagine that there is now a desire to add the field for “country of residence”. Once the author changes countries, all references must be updated as well, thus creating the country field, `country`, as static, shown below:


Creating the `author` structure now with the static country field:

```sql
CREATE TABLE IF NOT EXISTS library.author (
    name text,
    book text,
    year int,
    country text static,
    PRIMARY KEY (name, book)
);
INSERT INTO library.author JSON '{"name": "Robert Cecil Martin", "book": "Clean Architecture", "year": 2017}';
INSERT INTO library.author JSON '{"name": "Robert Cecil Martin", "book": "Clean Code", "country": "USA", "year": 2008}';
INSERT INTO library.author JSON '{"name": "Joshua Bloch", "book": "Effecive Java", "year": 2001}';
INSERT INTO library.author JSON '{"name": "Joshua Bloch", "book": "JAVA PUZZLERS", "year": 2005, "country": "Brazil"}';
```

Even inserted only once, it was persisted in all fields with the same key.

```sql
cqlsh> SELECT * FROM library.author;

 name | book | country | year
--------------------- + -------------------- + ------- - + ------
        Joshua Bloch | Effecive Java | Brazil | 2001
        Joshua Bloch | JAVA PUZZLERS | Brazil | 2005
 Robert Cecil Martin | Clean Architecture | USA | 2017
 Robert Cecil Martin | Clean Code | USA | 2008

(4 rows)

```


### Secondary index


There is another option to search the table information in addition to the primary key, which is the secondary index. It allows the search for information without generating the `ALLOW FILTERING` error. Your template is shown below:

```sql
index_name :: = re ('[a-zA-Z_0-9] +')
```

For example, to perform a search for the year field, year, in the column family of authors it is possible to execute the following command:

```sql
CREATE INDEX year_index ON library.author (year);
```

Thus, it is finally possible to execute the query from the year:

```sql
cqlsh> SELECT * FROM library.author where year = 2005;

 name | book | country | year
-------------- + --------------- + --------- + ------
 Joshua Bloch | JAVA PUZZLERS | Brazil | 2005

(1 rows)
```

It is also possible to remove the index created with `DROP INDEX`:

```sql
drop_index_statement :: = DROP INDEX [IF EXISTS] index_name
```

In our example:

```sql
DROP INDEX IF EXISTS library.year_index;
```


### How the secondary index works

Secondary indexes are created for a column within a column family and are stored locally for each node.

If a query is based on a secondary index, it will not have the same efficiency as the search for the key and can strongly impact performance. This is one of the reasons why some documentation from Cassandra considers the use of secondary index as an anti-pattern, so it is important to analyze the impact of creating a secondary index.

> An efficient way to use the secondary index is in partnership with the primary key, partition key, so it will be addressed over a range of specific nodes.

There are some good practices when using indexes, which are:

* **Do not** use when there is a high degree of cardinality
* **Do not** use in fields that are updated with a high frequency


## Types in Cassandra

Within a family of columns, each field has a type that defines how the field will be stored in the database. To facilitate understanding, they will be divided into four:

* Native types
* Collection types
* Tuples
* User-defined-type UDT


### Native types

The native types are those that Cassandra already supports and it is not necessary to carry out any modifications or creation within the database.


| Type      | description                                      |
| --------- | ------------------------------------------------ |
| ascii     | ASCII String                                     |
| bigint    | A 64-bit long                                    |
| blob      | A large array of bytes                           |
| boolean   | Has a value of true or false                     |
| counter   | One column counter (long 64-bit)                 |
| date      | A date                                           |
| decimal   | A type of decimal precision                      |
| double    | 64-bit floating point IEEE-754                   |
| duration  | A type of duration with precision in nanoseconds |
| float     | 32-bit floating point IEEE-754                   |
| inet      | An IP address                                    |
| int       | 32-bit int                                       |
| smallint  | 16-bit int                                       |
| text      | A String like UTF8                               |
| time      | A date with precision in nanoseconds             |
| timestamp | A timestamp with millisecond precision           |
| uuid      | unique universal identifier                      |
| varchar   | A String in UTF8                                 |



### Collection types


Within Cassandra, there is support for three types of collections that follow the same line of the Java world. These collections fit perfectly when it is necessary to have a field that has a set of items, for example, the telephone or contact list.

```sql
collection_type :: = MAP '<' cql_type ',' cql_type '>'
                     | SET '<' cql_type '>'
                     | LIST '<' cql_type '>'
```

### List


It is a sequence of items in the order they were added. For example, given a column family of readers, it is possible to have a list of books. Like this:

Creation of the column reader family
```sql
CREATE TABLE IF NOT EXISTS library.reader (
    name text,
    books list <text>,
    PRIMARY KEY (name)
);

INSERT INTO library.reader (name, books) VALUES ('Poliana', ['The Shack', 'The Love', 'Clean Code']);
INSERT INTO library.reader (name, books) VALUES ('David', ['Clean Code', 'Effectove Java', 'Clean Code']);

cqlsh> SELECT * FROM library.reader;

 name | books
--------- + ---------------------------------------- --------
   David | ['Clean Code', 'Effectove Java', 'Clean Code']
 Poliana | ['The Shack', 'The Love', 'Clean Code']

(2 rows)

```

Within the list, it is possible to perform several options, for example, add one or more elements, replace a single item, as shown in the code:

```sql
// repleace
cqlsh> UPDATE library.reader SET books = ['Java EE 8'] WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';

 name | books
--------- + ---------------------------------------- -
   David | ['Java EE 8']

// appending
UPDATE library.reader SET books = books + ['Clean Code'] WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';
 name | books
------- + -----------------------------
 David | ['Java EE 8', 'Clean Code']

// update an element
UPDATE library.reader SET books [1] = 'Clean Architecture' WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';
 name | books
------- + -------------------------------------
 David | ['Java EE 8', 'Clean Architecture']

```

It is also possible to remove elements with `DELETE` and `UPDATE`.

```sql
DELETE books [1] FROM library.reader where name = 'David';
cqlsh> SELECT * FROM library.reader WHERE name = 'David';

 name | books
------- + ---------------
 David | ['Java EE 8']


UPDATE library.reader SET books = books - ['Java EE 8'] WHERE name = 'David';
cqlsh> SELECT * FROM library.reader WHERE name = 'David';
 name | books
------- + -------
 David | null
```

### Set

Set is similar to List, however, it does not allow duplicate values. Thus, the same example mentioned above, but with Set, would be:

Creation of the column family of readers (this time, it is possible to see that the book “Clean Code” by reader “David” will not appear).

```sql
DROP TABLE IF EXISTS library.reader;

CREATE TABLE IF NOT EXISTS library.reader (
    name text,
    books set <text>,
    PRIMARY KEY (name)
);

INSERT INTO library.reader (name, books) VALUES ('Poliana', {'The Shack', 'The Love', 'Clean Code'});
INSERT INTO library.reader (name, books) VALUES ('David', {'Clean Code', 'Effectove Java', 'Clean Code'});

cqlsh> SELECT * FROM library.reader;

 name | books
--------- + ----------------------------------------
   David | {'Clean Code', 'Effectove Java'}
 Poliana | {'Clean Code', 'The Love', 'The Shack'}

(2 rows)

```

Within the Set it is possible to either replace the current collection or add new elements:


```sql
// repleace
cqlsh> UPDATE library.reader SET books = {'Java EE 8'} WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';

 name | books
--------- + ---------------------------------------- -
   David | {'Java EE 8'}

// appending
UPDATE library.reader SET books = books + {'Clean Code'} WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';
 name | books
------- + -----------------------------
 David | {'Java EE 8', 'Clean Code'}

```

> As a recommendation, the `List` type has some limitations and performance problems compared to the `Set`. If you have a choice, always prioritize the `Set`.


### Map

The map follows the line of a data dictionary or a `java.util.Map`, if the reader is from the Java world. Given the previous example from readers, let's assume that you want to add contact information. See the code below.

```sql
DROP TABLE IF EXISTS library.reader;

CREATE TABLE IF NOT EXISTS library.reader (
    name text,
    books set <text>,
    contacts map <text, text>,
    PRIMARY KEY (name)
);
INSERT INTO library.reader (name, books, contacts) VALUES ('Poliana', {'The Shack', 'The Love', 'Clean Code'},
{'email': 'poliana@email.com', 'phone': '+1 55 486848635', 'twitter': 'polianatwitter', 'facebook': 'polianafacebook'});
INSERT INTO library.reader (name, books, contacts) VALUES ('David', {'Clean Code'}, {'email': 'david@email.com'
, 'phone': '+1 55 48684865', 'twitter': 'davidtwitter'});

cqlsh> SELECT * FROM library.reader;

 name | books | contacts
--------- + ---------------------------------------- - + ------------------------------------------------ -------------------------------------------------- ----------------------
   David | {'Clean Code'} | {'email': 'david@email.com', 'phone': '+1 55 48684865', 'twitter': 'davidtwitter'}
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635', 'twitter': 'polianatwitter'}

```

As with previous collections, you can make changes:

```sql
// to update a key element
UPDATE library.reader SET contacts ['twitter'] = 'fakeaccount' WHERE name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';

 name | books | contacts
--------- + ---------------------------------------- - + ------------------------------------------------ -------------------------------------------------- -------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635', 'twitter': 'fakeaccount'}
// to append a new entry
UPDATE library.reader SET contacts = contacts + {'youtube': 'youtubeaccount'} WHERE name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';
 name | books | contacts
--------- + ---------------------------------------- - + ------------------------------------------------ -------------------------------------------------- ------------------------------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635', 'twitter': 'fakeaccount', 'youtube': 'youtubeaccount'}
// remove an element
DELETE contacts ['youtube'] FROM library.reader where name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';
 name | books | contacts
--------- + ---------------------------------------- - + ------------------------------------------------ -------------------------------------------------- -------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635', 'twitter': 'fakeaccount'}
// to remove elements by the key
UPDATE library.reader SET contacts = contacts - {'youtube', 'twitter'} WHERE name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';
 name | books | contacts
--------- + ---------------------------------------- - + ------------------------------------------------ -------------------------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635'}
// to repleace the whole map
UPDATE library.reader SET contacts = {'email': 'just@email.com'} WHERE name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';
 name | books | contacts
--------- + ---------------------------------------- - + -----------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'just@email.com'}

```


### Tuple

A tuple is a combination of key and value. For the Java world, it is similar to `java.util.Map.Entry`. Its creation structure is:

```sql
tuple_type :: = TUPLE '<' cql_type (',' cql_type) * '>'
tuple_literal :: = '(' term (',' term) * ')'
```

For this example, we will use the same column family of readers, however, we will assume that only a single piece of contact information, regardless of which one, is sufficient. Like this:

```sql
DROP TABLE IF EXISTS library.reader;

CREATE TABLE IF NOT EXISTS library.reader (
    name text,
    books set <text>,
    contact tuple <text, text>,
    PRIMARY KEY (name)
);
INSERT INTO library.reader (name, books, contact) VALUES ('Poliana', {'The Shack', 'The Love', 'Clean Code'}, ('email', 'poliana@email.com'));
cqlsh> SELECT * FROM library.reader;

 name | books | contact
--------- + ---------------------------------------- - + --------------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | ('email', 'poliana@email.com')

```

> Unlike collections, it is not possible to replace a single element of the tuple, such as just the key or value. It is necessary to update the entire field.


### User-Defined Types

User-Defined Types, or just UDT, is a data type created by the user. This type is created from a keyspace and follows the same principle as a column family, that is, it will be possible to create, change, and drop a `UDT`.

```sql
create_type_statement :: = CREATE TYPE [IF NOT EXISTS] udt_name
                               '(' field_definition (',' field_definition) * ')'
field_definition :: = identifier cql_type
```

Just like the native type, to be used, it is necessary to define it within a column family. For example, for the information of a library user, the first and last names are required.

```sql
CREATE TYPE IF NOT EXISTS library.name (
    first_name text,
    last_name text,
);
CREATE TABLE IF NOT EXISTS library.user (
    id text,
    name name,
    PRIMARY KEY (id)
);
INSERT INTO library.user (id, name) values ('otaviojava', {first_name: 'Otavio', last_name: 'Santana'});
INSERT INTO library.user (id, name) values ('poliana', {first_name: 'Poliana', last_name: 'Santana'});
cqlsh> SELECT * FROM library.user;

 id | name
------------ + ------------------------------------- ----------
    poliana | {first_name: 'Poliana', last_name: 'Santana'}
 otaviojava | {first_name: 'Otavio', last_name: 'Santana'}
```

It is also possible to change a UDT. For example, given the name type, the middle name field will be added.


```sql
ALTER TYPE library.name ADD middle_name text;
cqlsh> SELECT * FROM library.user;

 id | name
------------ + ------------------------------------- -----------------------------
    poliana | {first_name: 'Poliana', last_name: 'Santana', middle_name: null}
 otaviojava | {first_name: 'Otavio', last_name: 'Santana', middle_name: null}

INSERT INTO library.user (id, name) values ('otaviojava', {first_name: 'Otavio', last_name: 'Santana', middle_name: 'Gonçalves'});
INSERT INTO library.user (id, name) values ('poliana', {first_name: 'Poliana', last_name: 'Santana', middle_name: 'Santos'});

cqlsh> SELECT * FROM library.user;
 id | name
------------ + ------------------------------------- -----------------------------------
    poliana | {first_name: 'Poliana', last_name: 'Santana', middle_name: 'Santos'}
 otaviojava | {first_name: 'Otavio', last_name: 'Santana', middle_name: 'Gonçalves'}
```

It is also possible to remove the UDT as `DROP UDT`.

```sql
drop_type_statement :: = DROP TYPE [IF EXISTS] udt_name
```

For example, when removing the column family name field:

```sql
ALTER COLUMNFAMILY library.user DROP name;
cqlsh> SELECT * FROM library.user;
 id
------------
    poliana
 otaviojava

DROP TYPE IF EXISTS library.name;
```

Cassandra's collections also support UDTs, for this, it is necessary to use the keyword frozen. It is worth noting that the UDT has a similar operation to that of a tuple, that is, it is not possible to change a single element.

```sql
DROP TABLE IF EXISTS library.user;
DROP TABLE IF EXISTS library.user;
CREATE TYPE IF NOT EXISTS library.phone (
    country_code int,
    number text,
);
CREATE TABLE IF NOT EXISTS library.user (
    id text,
    phones set <frozen <phone>>,
    PRIMARY KEY (id)
);
```

## Manipulating information

In this section we will deal with data manipulation, it will be possible to use the famous CRUD: create, retrieve, update, delete information.

### SELECT

`SELECT` is the type of command used to retrieve data information.

```sql
select_statement :: = SELECT [JSON | DISTINCT] (select_clause | '*')
                      FROM table_name
                      [WHERE where_clause]
                      [GROUP BY group_by_clause]
                      [ORDER BY ordering_clause]
                      [PER PARTITION LIMIT (integer | bind_marker)]
                      [LIMIT (integer | bind_marker)]
                      [ALLOW FILTERING]
select_clause :: = selector [AS identifier] (',' selector [AS identifier])
selector :: = column_name
                      | term
                      | CAST '(' selector AS cql_type ')'
                      | function_name '(' [selector (',' selector) *] ')'
                      | COUNT '(' '*' ')'
where_clause :: = relation (AND relation) *
relation :: = column_name operator term
                      '(' column_name (',' column_name) * ')' operator tuple_literal
                      TOKEN '(' column_name (',' column_name) * ')' operator term
operator :: = '=' | '<' | '>' | '<=' | '> =' | '! =' | IN | CONTAINS | CONTAINS KEY
group_by_clause :: = column_name (',' column_name) *
ordering_clause :: = column_name [ASC | DESC] (',' column_name [ASC | DESC]) *
```

For example, in the bookstore, a family of magazine type columns will be added that will have the list of articles and the year of the post.


```sql
DROP COLUMNFAMILY IF EXISTS library.magazine;
CREATE COLUMNFAMILY IF NOT EXISTS library.magazine (
    id text,
    posted_at timestamp,
    articles set <text>,
    pages int,
    PRIMARY KEY (id, posted_at)
);

INSERT INTO library.magazine (id, posted_at, articles, pages) values ('Java Magazine', '2018-01-01', {'Jakarta EE', 'Java 8', 'Cassandra'}, 140);
INSERT INTO library.magazine (id, posted_at, articles, pages) values ('Java Magazine', '2017-01-01', {'Java EE 8', 'Java 7', 'NoSQL'}, 100);
```

Within a query, it is possible to retrieve both all fields and specific columns using the same SQL language standard.

```sql
cqlsh> SELECT * FROM library.magazine;

 id | posted_at | articles | pages
--------------- + --------------------------------- + --------------------------------------- + -------
 Java Magazine | 2017-01-01 00: 00: 00.000000 + 0000 | {'Java 7', 'Java EE 8', 'NoSQL'} | 100
 Java Magazine | 2018-01-01 00: 00: 00.000000 + 0000 | {'Cassandra', 'Jakarta EE', 'Java 8'} | 140


cqlsh> SELECT id, posted_at FROM library.magazine;

 id | posted_at
--------------- + ---------------------------------
 Java Magazine | 2017-01-01 00: 00: 00.000000 + 0000
 Java Magazine | 2018-01-01 00: 00: 00.000000 + 0000

cqlsh> SELECT count (*) FROM library.magazine;

 count
-------
     2

```

Within Cassandra, there are also some value aggregation queries, with the keyword `GROUP BY`, however, to use this feature, the target field must be a primary key.

```sql
cqlsh> SELECT id, max (pages) FROM library.magazine GROUP BY id;

 id | system.max (pages)
--------------- + -------------------
 Java Magazine | 140

cqlsh> SELECT id, min (pages) FROM library.magazine GROUP BY id;

  id | system.min (pages)
 --------------- + -------------------
  Java Magazine | 100


cqlsh> SELECT id, sum (pages) FROM library.magazine GROUP BY id;

 id | system.sum (pages)
--------------- + -------------------
 Java Magazine | 240

```


In the ideal world, all queries are performed based on the key, that is, there will always be the return of a single result. However, in some cases, it is necessary to perform searches using secondary indexes, bringing a large volume of results. One way to avoid this amount is to limit the maximum return value, and Cassandra has this feature. To do this, just use `LIMIT`.

```sql
cqlsh> SELECT * FROM library.magazine LIMIT 1;

 id | posted_at | articles | pages
--------------- + --------------------------------- + ---------------------------------- + -------
 Java Magazine | 2017-01-01 00: 00: 00.000000 + 0000 | {'Java 7', 'Java EE 8', 'NoSQL'} | 100
```

The ordering of the data is done from the `ORDER BY`, however, it is not as powerful as in the relational banks since the ordering is only performed by key type fields.

```sql
 SELECT * FROM library.magazine where id = 'Java Magazine' ORDER BY posted_at DESC;
 id | posted_at | articles | pages
--------------- + --------------------------------- + --------------------------------------- + -------
 Java Magazine | 2018-01-01 00: 00: 00.000000 + 0000 | {'Cassandra', 'Jakarta EE', 'Java 8'} | 140
 Java Magazine | 2017-01-01 00: 00: 00.000000 + 0000 | {'Java 7', 'Java EE 8', 'NoSQL'} | 100


 SELECT * FROM library.magazine where id = 'Java Magazine' ORDER BY posted_at ASC;

  id | posted_at | articles | pages
 --------------- + --------------------------------- + --------------------------------------- + -------
  Java Magazine | 2017-01-01 00: 00: 00.000000 + 0000 | {'Java 7', 'Java EE 8', 'NoSQL'} | 100
  Java Magazine | 2018-01-01 00: 00: 00.000000 + 0000 | {'Cassandra', 'Jakarta EE', 'Java 8'} | 140


```

In a search query, it is also possible to retrieve the values in the `JSON` format.

```sql
cqlsh> SELECT JSON * FROM library.magazine;

 [json]
-------------------------------------------------- -------------------------------------------------- -------------------------------
      {"id": "Java Magazine", "posted_at": "2017-01-01 00: 00: 00.000Z", "articles": ["Java 7", "Java EE 8", "NoSQL"], " pages ": 100}
 {"id": "Java Magazine", "posted_at": "2018-01-01 00: 00: 00.000Z", "articles": ["Cassandra", "Jakarta EE", "Java 8"], "pages ": 140}

cqlsh> SELECT JSON id, pages FROM library.magazine;

 [json]
---------------------------------------
 {"id": "Java Magazine", "pages": 100}
 {"id": "Java Magazine", "pages": 140}

```

> It is important to note that search operations within Cassandra are very limited, so the recommendation is to use the keys, leaving the index as the last option. There is also the feature of using `ALLOW FILTERING`, however, using this feature requires a very high-performance impact.


### INSERT

Within the `INSERT` clause, it is possible to insert a row within the column family.

```sql
insert_statement :: = INSERT INTO table_name (names_values | json_clause)
                      [IF NOT EXISTS]
                      [USING update_parameter (AND update_parameter) *]
names_values :: = names VALUES tuple_literal
json_clause :: = JSON string [DEFAULT (NULL | UNSET)]
names :: = '(' column_name (',' column_name) * ')'
```

To insert the data using the `INSERT` clause, it is possible to use it both in a very similar way to SQL and as JSON. For example, using the previous magazine column family.

```sql
INSERT INTO library.magazine (id, posted_at, articles, pages) values ('Java Magazine', '2017-01-01', {'Java EE 8', 'Java 7', 'NoSQL'}, 100);
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 10, "articles": ["Java EE 8", "Java 7" , "NoSQL"]} ';
```

The key is mandatory and it is from there that it is defined where the information will be sent through the cluster.


```sql
INSERT INTO library.magazine JSON '{"posted_at": "2017-01-01", "pages": 10, "articles": ["Java EE 8", "Java 7", "NoSQL"]}';
InvalidRequest: Error from server: code = 2200 [Invalid query] message = "Invalid null value in condition for column id"
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 10}';
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01"}';
```

Besides the verification of the primary key, there is no other validation, that is, it is possible to have different values with the same key without any problem. If the key does not exist, it will be created, otherwise, it will be overwritten.

```sql
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 112}';
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 121}';
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 90}';
```

It is also possible to define an insertion with TTL, which means the lifetime of a line in seconds.

```sql
TRUNCATE library.magazine;
INSERT INTO library.magazine (id, posted_at) values ('Java Magazine', '2017-01-01') USING TTL 10;
cqlsh> SELECT * FROM library.magazine WHERE id = 'Java Magazine';

 id | posted_at | articles | pages
--------------- + --------------------------------- + ---------- + -------
 Java Magazine | 2017-01-01 00: 00: 00.000000 + 0000 | null | null
// wait 10 seconds
cqlsh> SELECT * FROM library.magazine WHERE id = 'Java Magazine';
 id | posted_at | articles | pages
---- + ----------- + ---------- + -------

(0 rows)
```

### UPDATE

To update, there is the `UPDATE` clause. In general, for Cassandra, it is nothing more than an `INSERT` with `WHERE`. As in `INSERT`, if the information exists, it will be updated, otherwise, it will be overwritten. For example, given a simple family of librarian columns, your creation using the UPDATE clause would look like this:

Creation of the librarian column family, as you can see, a librarian is inserted using the `UPDATE` clause.

```sql
DROP COLUMNFAMILY IF EXISTS library.librarian;

CREATE COLUMNFAMILY IF NOT EXISTS library.librarian (
    id text,
    name text,
    PRIMARY KEY (id)
);

UPDATE library.librarian set name = 'Ivar' where id = 'ivar';
cqlsh> SELECT * FROM library.librarian;

 id | name
------ + ------
 ivar | Ivar

(1 rows)

```

You can also update the record's TTL.

```sql
UPDATE library.librarian USING TTL 12 set name = 'Daniel' where id = 'daniel';
cqlsh> SELECT * FROM library.librarian where id = 'daniel';
 id | name
-------- + --------
 daniel | Daniel
// wait 10 seconds
cqlsh> SELECT * FROM library.librarian where id = 'daniel';

 id | name
---- + ------
```

### DELETE

With the `DELETE` clause, you can remove records within Cassandra, either just one column or the entire record, and for any operation, the primary key and the partition key is required.


```sql
DROP COLUMNFAMILY IF EXISTS library.librarian;

CREATE COLUMNFAMILY IF NOT EXISTS library.librarian (
    id text,
    name text,
    PRIMARY KEY (id)
);

INSERT INTO library.librarian JSON '{"name": "Ivar", "id": "ivar"}';
INSERT INTO library.librarian JSON '{"name": "Daniel Dias", "id": "dani"}';
INSERT INTO library.librarian JSON '{"name": "Gabriela Santana", "id": "gabriela"}';

cqlsh> SELECT * FROM library.librarian;

 id | name
---------- + ------------------
     dani | Daniel Dias
 gabriela | Gabriela Santana
     ivar | Ivar

(3 rows)

DELETE FROM library.librarian where id = 'dani';
cqlsh> SELECT * FROM library.librarian;
 id | name
---------- + ------------------
 gabriela | Gabriela Santana
     ivar | Ivar

(2 rows)
DELETE name FROM library.librarian where id = 'ivar';
cqlsh> SELECT * FROM library.librarian;
 id | name
---------- + ------------------
 gabriela | Gabriela Santana
     ivar | null

(2 rows)

```

### BATCH

`BATCH` allows multiple change operations (INSERT, UPDATE, and DELETE). The operations within a `BATCH` aim to perform several operations in an atomic way, that is, the operation happens or not.

It's important make sure that:

* `BATCH` can only contain INSERT, UPDATE and DELETE;
* `BATCH` commands are not fully supported by the transaction like relational banks;
* All operations belong to _partition key_ to guarantee isolation;
* By default, operations within `BATCH` will be atomic, so operations will eventually be complete or no operations will take place.
* There is a big *trade-off* in the use of `BATCH`: on the one hand, it can save network in the communication between the coordinating node and the other nodes for operations, on the other hand, it can use a node for several operations, hence the importance of know how to use this feature sparingly.

```sql
batch_statement :: = BEGIN [UNLOGGED | COUNTER] BATCH
                            [USING update_parameter (AND update_parameter) *]
                            modification_statement (';' modification_statement) *
                            APPLY BATCH
modification_statement :: = insert_statement | update_statement | delete_statement
```

For example, to perform operations within the column family of librarians.

```sql
BEGIN BATCH
INSERT INTO library.librarian JSON '{"name": "Ivar", "id": "ivar"}';
INSERT INTO library.librarian JSON '{"name": "Daniel Dias", "id": "dani"}';
INSERT INTO library.librarian JSON '{"name": "Gabriela Santana", "id": "gabriela"}';
DELETE FROM library.librarian where id = 'dani';
APPLY BATCH;

cqlsh> SELECT * FROM library.librarian;

 id | name
---------- + ------------------
 gabriela | Gabriela Santana
     ivar | Ivar

(2 rows)

```

### View materialized

Denormalization is the best friend of non-relational databases, and with Cassandra, this is no exception. One of the features that can facilitate this modeling is the creation of materialized views. This feature serves as an ally for denormalization while ensuring the consistency of a column family. An important point is that the materialized view cannot be changed.


```sql
create_materialized_view_statement :: = CREATE MATERIALIZED VIEW [IF NOT EXISTS] view_name AS
                                            select_statement
                                            PRIMARY KEY '(' primary_key ')'
                                            WITH table_options
```

For example, considering the case of the book again and the ISBN (International Standard Book Number) is the code that employees use to borrow a book within the library, it would be possible to create the column family as follows:

```sql
DROP COLUMNFAMILY IF EXISTS library.book;
CREATE COLUMNFAMILY IF NOT EXISTS library.book (
    isbn bigint,
    name text,
    author text,
    PRIMARY KEY (isbn)
);

INSERT INTO library.book JSON '{"isbn": 1, "name": "Clean Code", "author": "Robert Cecil Martin"}';
INSERT INTO library.book JSON '{"isbn": 2, "name": "Effective Java", "author": "Joshua Bloch"}';
INSERT INTO library.book JSON '{"isbn": 3, "name": "The Pragmatic Programmer", "author": "Andy Hunt"}';
```

It is now possible to search smoothly from the ISBN. However, considering that the ISBN is done incrementally, we would like to return the most recent books from the library and for that, a materialized view will be created for ISBN’s greater than 3. Thus:

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS library.recent_book AS
    SELECT * FROM library.book
    WHERE isbn> 3 AND name IS NOT NULL
    PRIMARY KEY (isbn, name);

Warnings: Materialized views are experimental and are not recommended for production use.
```

With the created materialized view, it is possible to perform searches similarly to the column family.

```sql
cqlsh> SELECT * FROM library.recent_book;

 isbn | name | author
------ + ------ + --------

INSERT INTO library.book JSON '{"isbn": 4, "name": "Java EE 8 Cookbook", "author": "Elder Moraes"}';
INSERT INTO library.book JSON '{"isbn": 5, "name": "Best Developer Job Ever!", "Author": "Bruno Souza"}';

cqlsh> SELECT * FROM library.recent_book;
 isbn | name | author
------ + -------------------------- + --------------
    4 | Java EE 8 Cookbook | Elder Moraes
    5 | Best Developer Job Ever! | Bruno Souza

```

It is also possible to make the change or remove the materialized view.


```sql
DROP MATERIALIZED VIEW library.recent_book;
```


> So far, the materialized view resource is on an experimental basis

## Safety


Cassandra has important support for the security feature. Thus, it is possible to define permission, create user, create rules, remove permission, among others. The first step in applying this to the project is to enable the security feature by Cassandra. For that, it is necessary to make a change inside `cassandra.yaml`, inside the `conf` folder: change the line `authenticator: AllowAllAuthenticator` to `authenticator: PasswordAuthenticator`.


If you are using Docker, the solution would be to map the configuration location. Like this:

```bash
docker run --name some-cassandra -p 9042: 9042 -v / my / own / datadir: / var / lib / cassandra -v /path/to/config/cassandra.yaml:/etc/cassandra/cassandra.yaml - d cassandra
// sample
docker run --name some-cassandra -p 9042: 9042 -v / home / otaviojava / data: / var / lib / cassandra -v / home / otaviojava / config: / etc / cassandra -d cassandra
```

Once enabled, when accessing Cassandra again there will be an error message:

```bash
./cqlsh
Connection error: ('Unable to connect to any servers', {'127.0.0.1': AuthenticationFailed ('Remote end requires authentication.',)})
```

This error happens because access authorization is required.

> The default superuser is `cassandra` and the password` cassandra`.

```bash
./cqlsh -u cassandra -p cassandra
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cassandra @ cqlsh>
```

For Docker, access is similar, just search for the container ID and then add the user and password parameters.

```bash
 docker exec -it 66e5f38a3815 cqlsh -u cassandra -p cassandra
```

The whole creation was defined from the ROLES. For example, it is possible to create some users.

The users `ada` and `alice` created.

```sql
CREATE ROLE ada;
CREATE ROLE alice WITH PASSWORD = 'alice' AND LOGIN = true;
cqlsh> LIST ROLES;
 role | super | login | options
----------- + ------- + ------- + ---------
       ada | False | False | {}
     alice | False | True | {}
 cassandr
```

It is now possible to log in with the user `alice`.

```sql
./cqlsh -u alice -p alice
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
alice @ cqlsh>
```

Since `alice` does not have superuser permission, it will not be possible to remove sensitive fields from the system.

```sql
alice @ cqlsh> DROP KEYSPACE IF EXISTS system_auth;
Unauthorized: Error from server: code = 2100 [Unauthorized] message = "Cannot DROP <keyspace system_auth>"
```

It is possible to change user information, for example, to make `alice` super-user permission.

```sql
ALTER ROLE alice WITH SUPERUSER = true;

cassandra @ cqlsh> LIST ROLES;
 role | super | login | options
----------- + ------- + ------- + ---------
       ada | False | False | {}
     alice | True | True | {}
 cassandra | True | True | {}

```

As well as removing user:

```sql
DROP ROLE;
cassandra @ cqlsh> LIST ROLES;
 role | super | login | options
----------- + ------- + ------- + ---------
     alice | True | True | {}
 cassandra | True | True | {}

(2 rows)

```


With the users created, it is now possible, for example, to create rules and define the permissions for each structure within the database. You can see that the options have the same structures as a relational database. The available options are:


* CREATE
* ALTER
* DROP
* SELECT
* MODIFY
* AUTHORIZE
* DESCRIBE
* EXECUTE

```sql
grant_permission_statement :: = GRANT permissions ON resource TO role_name
permissions :: = ALL [PERMISSIONS] | permission [PERMISSION]
permission :: = CREATE | ALTER | DROP | SELECT | MODIFY | AUTHORIZE | DESCRIBE | EXECUTE
resource :: = ALL KEYSPACES
                               | KEYSPACE keyspace_name
                               | [TABLE] table_name
                               | ALL ROLES
                               | ROLE role_name
                               | ALL FUNCTIONS [IN KEYSPACE keyspace_name]
                               | FUNCTION function_name '(' [cql_type (',' cql_type) *] ')'
                               | ALL MBEANS
                               | (MBEAN | MBEANS) string
```


To clarify the security context within Cassandra, imagine the following scenario within the bookstore system:

* The `user` rule can only read within the library keyspace
* The librarian rule can make changes to the library keyspace
* The `manager` rule can create tables within the library keyspace

Based on these rules, the following will be created:

* `ada` as a user
* Mike as a librarian
* `jonh` as manager


```sql
CREATE KEYSPACE IF NOT EXISTS library WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};
// create the role user
CREATE ROLE IF NOT EXISTS user;
GRANT SELECT ON KEYSPACE library TO user;
// create user ada
CREATE ROLE ada WITH PASSWORD = 'ada' AND LOGIN = true;
GRANT user TO ada;
// create the role librarian
CREATE ROLE IF NOT EXISTS librarian;
GRANT MODIFY ON KEYSPACE library TO librarian;
GRANT SELECT ON KEYSPACE library TO librarian;
// create mike
CREATE ROLE mike WITH PASSWORD = 'mike' AND LOGIN = true;
GRANT librarian TO mike;
// create manager
CREATE ROLE IF NOT EXISTS manager;
GRANT ALTER ON KEYSPACE library TO manager;
GRANT CREATE ON KEYSPACE library TO manager;
// create jonh
CREATE ROLE jonh WITH PASSWORD = 'jonh' AND LOGIN = true;
GRANT manager TO jonh;
```

With all users created, the first step is to explore a little bit of security inside Cassandra.

```sql
./cqlsh -u ada -p ada

CREATE COLUMNFAMILY IF NOT EXISTS library.magazine (
    id text,
    pages int,
    PRIMARY KEY (id)
);

Unauthorized: Error from server: code = 2100 [Unauthorized] message = "User ada has no CREATE permission on <keyspace library> or any of its parents"

./cqlsh -u jonh -p jonh

CREATE COLUMNFAMILY IF NOT EXISTS library.magazine (
   id text,
    pages int,
    PRIMARY KEY (id)
);

 ./cqlsh -u mike -p mike

INSERT INTO library.magazine JSON '{"id": "new magazine", "pages": 10}';
SELECT * FROM library.magazine;

 id | pages
-------------- + -------
 new magazine | 10

(1 rows)

./cqlsh -u ada -p ada

SELECT * FROM library.magazine;

 id | pages
-------------- + -------
 new magazine | 10

INSERT INTO library.magazine JSON '{"id": "new magazine", "pages": 10}';
Unauthorized: Error from server: code = 2100 [Unauthorized] message = "User ada has no MODIFY permission on <table library.magazine> or any of its parents"

```
With security enabled within Cassandra, users will be allowed to perform some operations from the `role`. This is a very interesting way of ensuring that only systems or people have read or write access at points that they need.

### Conclusion

In this chapter, it was possible to show that Cassandra Query Language or CQL has many more similarities with SQL besides the name, so that someone who already knows relational databases well and their communication syntax would have a low learning curve when learning the Cassandra.

One of the important points that this chapter explored was the use of security that exists within Cassandra. It is always important to be concerned about security whether with access firewalls or with user and password permission for points in the database.