# Conhecendo o CQL

A comunicação é, certamente, a tarefa mais trivial em um banco de dados.
O Cassandra oferece sua própria linguagem para comunicação com o banco de dados, o _Cassandra Query language_, ou CQL. O CQL possui uma sintaxe próxima ao SQL, porém, mais simples, já que não existem conceitos como _joins_ (_inner join_, _left join_, dentre outros) dentro das buscas do Cassandra. Quem já conhece o SQL terá uma curva de aprendizagem muito menor para aprender essa linguagem do Cassandra.


> Para essas operações utilizaremos o client `cqlsh` que vimos no capítulo anterior.

## Keyspace

A nossa primeira parada no CQL é a criação da maior estrutura dentro da hierarquia do Cassandra, o _keyspace_. É durante a criação do keyspace que é definida a estratégia de replicação e o fator de réplica para os dados. O template da criação é mostrado a seguir:

```sql
create_keyspace_statement ::=  CREATE KEYSPACE [ IF NOT EXISTS ] keyspace_name WITH options
```

Por exemplo, ao se criar um keyspace de `library`, livraria em inglês:

```sql
CREATE KEYSPACE library  WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};
//or
CREATE KEYSPACE library
           WITH replication = {'class': 'NetworkTopologyStrategy', 'DC1' : 1, 'DC2' : 3}
            AND durable_writes = false;
```


Realizada a criação, é possível verificar os keyspaces criados no Cassandra e para isso será realizada a primeira `query`. É possível perceber a semelhança com o banco de dados relacional. Falaremos mais sobre buscar informações nos itens a seguir.


```sql
cqlsh> select * from system_schema.keyspaces where keyspace_name= 'library';

 keyspace_name |replication
---------------+------------------------------------------------------
       library |{'class': 'SimpleStrategy', 'replication_factor': '3'}

(1 rows)
```

Para realizar alguma alteração do keyspace criado, é necessário utilizar o *alter keyspace*. O template é mostrado a seguir:

```sql
alter_keyspace_statement ::=  ALTER KEYSPACE keyspace_name WITH options
```

Por exemplo, alterando o keyspace criado anteriormente:


```sql
ALTER KEYSPACE Excelsior WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 4};
```

Também é possível remover o keyspace com o *drop keyspace*, no nosso exemplo:

```sql
DROP KEYSPACE library;
```

Caso o usuário tente criar ou remover mais de uma vez a mesma tabela, uma mensagem de erro é gerada. Por exemplo, `Keyspace 'library' already exists` ou `ConfigurationException: Cannot drop non existing keyspace 'library'.` para criar ou remover, respectivamente. Uma maneira de evitar tal erro é realizar essa alteração apenas caso realmente faça sentido: só crie caso ela não exista ou só remova caso ela exista. Com esse objetivo, a sintaxe suporta essa condição:


```sql
DROP KEYSPACE IF EXISTS library;
CREATE KEYSPACE IF NOT EXISTS library  WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};
```

## Familia de colunas

A criação da família de coluna é semelhante à criação da tabela, ou seja, é nela que são definidos os campos, _partition key_, _clustering key_, o índice e mais configurações. Basta utilizar o template `CREATE TABLE`:

```sql
create_table_statement ::=  CREATE TABLE [ IF NOT EXISTS ] table_name
                            '('
                                column_definition
                                ( ',' column_definition )*
                                [ ',' PRIMARY KEY '(' primary_key ')' ]
                            ')' [ WITH table_options ]
column_definition      ::=  column_name cql_type [ STATIC ] [ PRIMARY KEY]
primary_key            ::=  partition_key [ ',' clustering_columns ]
partition_key          ::=  column_name
                            | '(' column_name ( ',' column_name )* ')'
clustering_columns     ::=  column_name ( ',' column_name )*
table_options          ::=  COMPACT STORAGE [ AND table_options ]
                            | CLUSTERING ORDER BY '(' clustering_order ')' [ AND table_options ]
                            | options
clustering_order       ::=  column_name (ASC | DESC) ( ',' column_name (ASC | DESC) )*
```

Dado que foi criado o keyspace `library`, o próximo passo é criar a família de coluna de livros, `book`:


```sql
CREATE TABLE library.book (
    name text PRIMARY KEY,
       year int
);
```

> É possível também verificar se a família de coluna existe antes de criar, utilizando o `IF NOT EXISTS` semelhante ao keyspace.


Também é possível realizar alterações na família de coluna, por exemplo, adicionar ou remover um campo dentro dela. Para isso, é necessário utilizar o seguinte template:

```sql
alter_table_statement   ::=  ALTER TABLE table_name alter_table_instruction
alter_table_instruction ::=  ADD column_name cql_type ( ',' column_name cql_type )*
                             | DROP column_name ( column_name )*
                             | WITH options
```

Por exemplo, adicionar o campo `author`:

```sql
ALTER TABLE library.book ADD author text;
```

Caso queria remover o mesmo campo:

```sql
ALTER TABLE library.book DROP author;
```

Também é possível destruir a estrutura recém-criada.

```sql
drop_table_statement ::=  DROP TABLE [ IF EXISTS ] table_name
```

No nosso exemplo:

```sql
DROP TABLE IF EXISTS library.book;
```

Existem casos em que o objetivo não destruir a estrutura, mas remover todo o conteúdo e manter a estrutura. Para isso, é possível truncar a família de coluna, com o comando `TRUNCATE`.

```sql
truncate_statement ::=  TRUNCATE [ TABLE ] table_name
```


Por exemplo, para criar a família de coluna `book` e inserir dois livros: 

```sql
CREATE TABLE IF NOT EXISTS library.book (
    name text PRIMARY KEY,
       year int
);
INSERT INTO library.book  JSON '{"name": "Effective Java", "year": 2001}';
INSERT INTO library.book  JSON '{"name": "Clean Code", "year": 2008}';
```

Executando, previamente, a query para buscar os valores existentes no banco de dados:

```sql
cqlsh> SELECT * FROM library.book;

 name           | year
----------------+------
     Clean Code | 2008
 Effective Java | 2001

(2 rows)
TRUNCATE library.book;
cqlsh> SELECT * FROM library.book;

 name | year
------+------

(0 rows)

```

> É possível utilizar também `COLUMNFAMILY` em vez de `TABLE`.

### Chave primária

Dentro da família de coluna, a chave primária (_primary key_) é o campo único e todas as famílias de colunas *devem* defini-la. É a partir desse campo que os dados serão recuperados, por isso, existe uma preocupação inicial com ele. A chave primária pode ser constituída por mais de um campo, porém, ela terá um conceito diferente do banco relacional. Ela será dividida em duas partes:

* O **partition key**: é a primeira parte da chave primária que tem como responsabilidade a distribuição dos dados através dos nós.

* O **clustering columns**: essa chave tem o poder de definir a ordem dentro da tabela. Por exemplo, ao criar uma família de autores definimos o nome como chave e seus livros como ordem.


```sql
CREATE TABLE IF NOT EXISTS library.author (
    name text,
    book text,
    year int,
    PRIMARY KEY (name, book)
);
INSERT INTO library.author  JSON '{"name": "Robert Cecil Martin", "book": "Clean Code", "year": 2008}';
INSERT INTO library.author  JSON '{"name": "Robert Cecil Martin", "book": "Clean Architecture", "year": 2017}';
INSERT INTO library.author  JSON '{"name": "Robert Cecil Martin", "book": "Agile Principles, Patterns", "year": 2006}';
```

Ao executar a query teremos o seguinte resultado:

```sql
cqlsh> SELECT * FROM library.author;

 name                | book                       | year
---------------------+----------------------------+------
 Robert Cecil Martin | Agile Principles, Patterns | 2006
 Robert Cecil Martin |         Clean Architecture | 2017
 Robert Cecil Martin |                 Clean Code | 2008

(3 rows)
```

Uma outra opção é remover a tabela e criar novamente, dessa vez com a ordem dos livros de maneira decrescente. Assim:

```sql
CREATE TABLE IF NOT EXISTS library.author (
    name text,
    book text,
    year int,
    PRIMARY KEY (name, book)
)
WITH CLUSTERING ORDER BY (book DESC);

INSERT INTO library.author  JSON '{"name": "Robert Cecil Martin", "book": "Clean Code", "year": 2008}';
INSERT INTO library.author  JSON '{"name": "Robert Cecil Martin", "book": "Clean Architecture", "year": 2017}';
INSERT INTO library.author  JSON '{"name": "Robert Cecil Martin", "book": "Agile Principles, Patterns", "year": 2006}';

cqlsh> SELECT * FROM library.author;

 name                | book                       | year
---------------------+----------------------------+------
 Robert Cecil Martin |                 Clean Code | 2008
 Robert Cecil Martin |         Clean Architecture | 2017
 Robert Cecil Martin | Agile Principles, Patterns | 2006

(3 rows)
```

Realizando a busca a partir da chave, nome do autor do livro:

```sql
cqlsh> SELECT book FROM library.author WHERE name = 'Robert Cecil Martin';

 book
----------------------------
                 Clean Code
         Clean Architecture
 Agile Principles, Patterns

(3 rows)
```

Por padrão, não é possível realizar a busca por um campo que não seja a chave, `partition key`. Assim, no exemplo, tanto a busca pelo livro e pelo ano dará a mesma mensagem de erro.


```sql
cqlsh> SELECT * FROM library.author WHERE year =2017;
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
cqlsh> SELECT * FROM library.author WHERE book  ='Clean Architecture';
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
```

Como mostra a mensagem de erro, a única maneira de executar a query é adicionando o comando `ALLOW FILTERING`, porém, isso terá sérias consequências de performance. A maneira como o Cassandra executa esse comando é recuperando todas as linhas e então filtrando por aqueles que não têm o valor da condição. Por exemplo, em uma família de colunas que tenha um milhão de linhas e 98% delas tenha a condição da query, isso será relativamente eficiente. Porém, imagine o caso em que apenas uma linha atenda à condição. O Cassandra terá percorrido de maneira linear e ineficiente 999999 elementos para apenas retornar um.


```sql
cqlsh> SELECT * FROM library.author WHERE year =2017 ALLOW FILTERING;
 name                | book               | year
---------------------+--------------------+------
 Robert Cecil Martin | Clean Architecture | 2017
(1 rows)

cqlsh> SELECT * FROM library.author WHERE book  ='Clean Architecture' ALLOW FILTERING;
name                | book               | year
---------------------+--------------------+------
 Robert Cecil Martin | Clean Architecture | 2017

(1 rows)
```

> Se uma query é rejeitada pelo Cassandra pedindo o filtro, resista ao uso do `ALLOW FILTERING`. Verifique sua modelagem, seus dados e volumetria e veja o que você realmente quer fazer.
> Uma segunda opção é o uso de índices secundários que será abordado a seguir.


### Tipos estáticos

Algumas colunas podem ser consideradas estáticas dentro da família de coluna. Uma coluna estática compartilha o mesmo valor para todas as linhas que tenham o mesmo valor de `partition key`. Por exemplo, no cenário de família de coluna autor (com nome e livro), imagine que agora existe o desejo de se adicionar o campo para país de residência. Uma vez que o autor mude de país é importante que todas as suas referências também sejam atualizadas, assim, será criado o campo país, `country`, como estático, exibido a seguir:


Criando a estrutura de `author` agora com o campo estático para país:

```sql
CREATE TABLE IF NOT EXISTS library.author (
    name text,
    book text,
    year int,
    country text static,
    PRIMARY KEY (name, book)
);
INSERT INTO library.author  JSON '{"name": "Robert Cecil Martin", "book": "Clean Architecture", "year": 2017}';
INSERT INTO library.author  JSON '{"name": "Robert Cecil Martin", "book": "Clean Code", "country": "USA", "year": 2008}';
INSERT INTO library.author  JSON '{"name": "Joshua Bloch", "book": "Effecive Java", "year": 2001}';
INSERT INTO library.author  JSON '{"name": "Joshua Bloch", "book": "JAVA PUZZLERS", "year": 2005, "country": "Brazil"}';
```

Mesmo inserido uma única vez, ele foi persistido em todos os campos com a mesma chave.

```sql
cqlsh> SELECT * FROM library.author;

 name                | book               | country | year
---------------------+--------------------+---------+------
        Joshua Bloch |      Effecive Java |  Brazil | 2001
        Joshua Bloch |      JAVA PUZZLERS |  Brazil | 2005
 Robert Cecil Martin | Clean Architecture |     USA | 2017
 Robert Cecil Martin |         Clean Code |     USA | 2008

(4 rows)

```


### Índice secundário


Existe uma outra opção para buscar as informações da tabela além da chave primária, que é o índice secundário. Ele permite a busca de informação sem que gere o erro do `ALLOW FILTERING`. O seu template é exibido a seguir:

```sql
index_name ::=  re('[a-zA-Z_0-9]+')
```

Por exemplo, para realizar uma busca pelo campo de ano, `year`, na família de coluna de autores é possível executar o seguinte comando:

```sql
CREATE INDEX year_index ON library.author (year);
```

Assim, finalmente é possível executar a query a partir do ano:

```sql
cqlsh> SELECT * FROM library.author where year = 2005;

 name         | book          | country | year
--------------+---------------+---------+------
 Joshua Bloch | JAVA PUZZLERS |  Brazil | 2005

(1 rows)
```

Também é possível remover o índice criado com o `DROP INDEX`:

```sql
drop_index_statement ::=  DROP INDEX [ IF EXISTS ] index_name
```

No nosso exemplo:

```sql
DROP  INDEX IF EXISTS library.year_index;
```


### Como funciona o índice secundário

Os índices secundários são criados para uma coluna dentro de uma família de coluna e são armazenados localmente para cada nó.

Se uma query é baseada em índice secundário, ela não terá a mesma eficiência como a busca pela chave e pode impactar fortemente a performance. Esse é um dos motivos pelos quais algumas documentações do Cassandra considera o uso de índice secundário como anti-pattern, de modo que é importante analisar o impacto da criação de um índice secundário.

> Uma maneira eficiente de usar o índice secundário é em parceria com a chave primária, `partition key`, assim ele será endereçado em um range de nós específicos.

Existem algumas boas práticas ao se utilizar índices, que são:

* **Não** utilizar quando existe um alto grau de cardinalidade;
* **Não** utilizar em campos que são atualizados com uma alta frequência.


## Tipos no Cassandra

Dentro de uma família de colunas cada campo tem um tipo que define como o campo será armazenado no banco de dados. Para facilitar o entendimento, eles serão divididos em quatro:

* Tipos nativos
* Tipos de coleção
* Tuplas
* User-defined-type, UDT


### Tipos nativos

Os tipos nativos são aqueles aos quais o Cassandra já tem suporte e não é necessário realizar nenhuma modificação ou criação dentro do banco de dados.


| Tipo | descrição|
| --- | ---|
|ascii|ASCII String|
|bigint|Um long de 64-bit|
|blob|Um grande array de bytes|
|boolean|Tem o valor true ou falso|
|counter|Uma coluna contadora (long 64-bit)|
|date|Uma data|
|decimal|Um tipo de precisão decimal|
|double|64-bit IEEE-754 de ponto flutuante|
|duration|Um tipo de duração com precisão em nanossegundos|
|float|32-bit IEEE-754 de ponto flutuante|
|inet|Um endereço IP|
|int|32-bit int|
|smallint|16-bit int|
|text| Uma String como UTF8|
|time|Uma data com precisão em nanossegundos|
|timestamp|Um timestamp com precisão em milissegundos|
|uuid| único identificador universal|
|varchar|Uma String em UTF8|



### Tipos de coleção


Dentro do Cassandra existe suporte para três tipos de coleções que seguem a mesma linha do mundo Java. Essas coleções se encaixam perfeitamente quando é necessário ter um campo que possui conjunto de itens, por exemplo, a lista de telefones ou de contatos.

```sql
collection_type ::=  MAP '<' cql_type ',' cql_type '>'
                     | SET '<' cql_type '>'
                     | LIST '<' cql_type '>'
```

### List


É a uma sequência de itens na ordem que em que foram adicionados. Por exemplo, dada uma família de coluna de leitores é possível ter uma lista livros. Assim:

Criação da família de coluna de leitores
```sql
CREATE TABLE IF NOT EXISTS library.reader (
    name text,
    books list<text>,
    PRIMARY KEY (name)
);

INSERT INTO library.reader (name, books) VALUES ('Poliana', ['The Shack','The Love','Clean Code']);
INSERT INTO library.reader (name, books) VALUES ('David', ['Clean Code','Effectove Java','Clean Code']);

cqlsh> SELECT * FROM library.reader;

 name    | books
---------+------------------------------------------------
   David | ['Clean Code', 'Effectove Java', 'Clean Code']
 Poliana |        ['The Shack', 'The Love', 'Clean Code']

(2 rows)

```

Dentro da lista é possível realizar diversas opções, por exemplo, adicionar um ou mais elementos, substituir um único item, como mostra o código:

```sql
//repleace
cqlsh> UPDATE library.reader SET books = [ 'Java EE 8'] WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';

 name    | books
---------+-----------------------------------------
   David |                           ['Java EE 8']

//appending
UPDATE library.reader SET books = books + [ 'Clean Code'] WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';
 name  | books
-------+-----------------------------
 David | ['Java EE 8', 'Clean Code']

//update an element
UPDATE library.reader SET books[1]= 'Clean Architecture' WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';
 name  | books
-------+-------------------------------------
 David | ['Java EE 8', 'Clean Architecture']

```

Também é possível remover os elementos com o `DELETE` e com o `UPDATE`.

```sql
DELETE books[1] FROM library.reader where name = 'David';
cqlsh> SELECT * FROM library.reader WHERE name= 'David';

 name  | books
-------+---------------
 David | ['Java EE 8']


UPDATE library.reader SET books = books - [ 'Java EE 8' ] WHERE name = 'David';
cqlsh> SELECT * FROM library.reader WHERE name= 'David';
 name  | books
-------+-------
 David |  null
```

### Set

O Set é similar ao List, porém, ele não permite valores duplicados. Assim, o mesmo exemplo citado anteriormente, porém com Set, ficaria:

Criação da família de coluna de leitores (dessa vez, é possível ver que o livro “Clean Code” do leitor “David” não aparecerá).

```sql
DROP TABLE IF EXISTS library.reader;

CREATE TABLE IF NOT EXISTS library.reader (
    name text,
    books set<text>,
    PRIMARY KEY (name)
);

INSERT INTO library.reader (name, books) VALUES ('Poliana', {'The Shack','The Love','Clean Code'});
INSERT INTO library.reader (name, books) VALUES ('David', {'Clean Code','Effectove Java','Clean Code'});

cqlsh> SELECT * FROM library.reader;

 name    | books
---------+----------------------------------------
   David |        {'Clean Code', 'Effectove Java'}
 Poliana | {'Clean Code', 'The Love', 'The Shack'}

(2 rows)

```

Dentro do Set é possível tanto substituir a corrente coleção como adicionar novos elementos:


```sql
//repleace
cqlsh> UPDATE library.reader SET books = { 'Java EE 8'} WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';

 name    | books
---------+-----------------------------------------
   David |                           {'Java EE 8'}

//appending
UPDATE library.reader SET books = books + { 'Clean Code'} WHERE name = 'David';
cqlsh> SELECT * FROM library.reader where name = 'David';
 name  | books
-------+-----------------------------
 David | {'Java EE 8', 'Clean Code'}

```

> Como recomendação, o tipo `List` tem algumas limitações e problemas de performance comparado ao `Set`. Caso possa escolher, sempre priorize o `Set`.


### Map

O mapa segue a linha de um dicionário de dados ou de um `java.util.Map`, caso o leitor seja do mundo Java. Dado o exemplo anterior de leitores, vamos supor que se deseje adicionar informações de contato. Veja o código a seguir.

```sql
DROP TABLE IF EXISTS library.reader;

CREATE TABLE IF NOT EXISTS library.reader (
    name text,
    books set<text>,
    contacts map<text,text>,
    PRIMARY KEY (name)
);
INSERT INTO library.reader (name, books, contacts) VALUES ('Poliana', {'The Shack','The Love','Clean Code'},
{'email': 'poliana@email.com','phone': '+1 55 486848635', 'twitter': 'polianatwitter', 'facebook': 'polianafacebook'});
INSERT INTO library.reader (name, books, contacts) VALUES ('David', {'Clean Code'}, {'email': 'david@email.com'
,'phone': '+1 55 48684865', 'twitter': 'davidtwitter'});

cqlsh> SELECT * FROM library.reader;

 name    | books                                   | contacts
---------+-----------------------------------------+------------------------------------------------------------------------------------------------------------------------
   David |                          {'Clean Code'} |                                     {'email': 'david@email.com', 'phone': '+1 55 48684865', 'twitter': 'davidtwitter'}
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635', 'twitter': 'polianatwitter'}

```

Assim como as coleções anteriores, é possível realizar alterações:

```sql
//to update a key element
UPDATE library.reader SET contacts['twitter'] = 'fakeaccount' WHERE name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';

 name    | books                                   | contacts
---------+-----------------------------------------+---------------------------------------------------------------------------------------------------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635', 'twitter': 'fakeaccount'}
//to append a new entry
UPDATE library.reader SET contacts = contacts+ {'youtube':'youtubeaccount'} WHERE name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';
 name    | books                                   | contacts
---------+-----------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635', 'twitter': 'fakeaccount', 'youtube': 'youtubeaccount'}
//remove an element
DELETE contacts['youtube'] FROM  library.reader where  name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';
 name    | books                                   | contacts
---------+-----------------------------------------+---------------------------------------------------------------------------------------------------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635', 'twitter': 'fakeaccount'}
//to remove elements by the key
UPDATE library.reader SET contacts = contacts - {'youtube','twitter'} WHERE name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';
 name    | books                                   | contacts
---------+-----------------------------------------+-------------------------------------------------------------------------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'poliana@email.com', 'facebook': 'polianafacebook', 'phone': '+1 55 486848635'}
// to repleace the whole map
UPDATE library.reader SET contacts= {'email': 'just@email.com'} WHERE name = 'Poliana';
cqlsh> SELECT * from library.reader where name = 'Poliana';
 name    | books                                   | contacts
---------+-----------------------------------------+-----------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | {'email': 'just@email.com'}

```


### Tupla

Uma tupla é uma combinação de chave e valor. Para o mundo Java, é semelhante ao `java.util.Map.Entry`. Sua estrutura de criação é:

```sql
tuple_type    ::=  TUPLE '<' cql_type ( ',' cql_type )* '>'
tuple_literal ::=  '(' term ( ',' term )* ')'
```

Para esse exemplo, vamos utilizar a mesma família de coluna de leitores, porém, assumiremos que apenas uma única informação de contato, independente de qual seja, é suficiente. Assim:

```sql
DROP TABLE IF EXISTS library.reader;

CREATE TABLE IF NOT EXISTS library.reader (
    name text,
    books set<text>,
    contact tuple<text,text>,
    PRIMARY KEY (name)
);
INSERT INTO library.reader (name, books, contact) VALUES ('Poliana', {'The Shack','The Love','Clean Code'}, ('email', 'poliana@email.com'));
cqlsh> SELECT * FROM library.reader;

 name    | books                                   | contact
---------+-----------------------------------------+--------------------------------
 Poliana | {'Clean Code', 'The Love', 'The Shack'} | ('email', 'poliana@email.com')

```

> Diferente das coleções, não é possível substituir um único elemento da tupla, como apenas a chave ou o valor. É necessário atualizar todo o campo.


### User-Defined Types

O User-Defined Types, ou apenas UDT, é um tipo de dados criado pelo usuário. Esse tipo é criado a partir de um keyspace e segue o mesmo princípio de uma família de colunas, ou seja, será possível criar, alterar e dropar um UDT.

```sql
create_type_statement ::=  CREATE TYPE [ IF NOT EXISTS ] udt_name
                               '(' field_definition ( ',' field_definition )* ')'
field_definition      ::=  identifier cql_type
```

Assim como o tipo nativo, para ser utilizado, é necessário defini-lo dentro de uma família de coluna. Por exemplo, para informação de um usuário da biblioteca são necessários o primeiro e o último nome.

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

 id         | name
------------+-----------------------------------------------
    poliana | {first_name: 'Poliana', last_name: 'Santana'}
 otaviojava |  {first_name: 'Otavio', last_name: 'Santana'}
 
```


Também é possível alterar um UDT. Por exemplo, dado o tipo nome, será adicionado o campo nome do meio.


```sql
ALTER TYPE library.name ADD middle_name text;
cqlsh> SELECT * FROM library.user;

 id         | name
------------+------------------------------------------------------------------
    poliana | {first_name: 'Poliana', last_name: 'Santana', middle_name: null}
 otaviojava |  {first_name: 'Otavio', last_name: 'Santana', middle_name: null}

INSERT INTO library.user (id, name) values ('otaviojava', {first_name: 'Otavio', last_name: 'Santana', middle_name: 'Gonçalves'});
INSERT INTO library.user (id, name) values ('poliana', {first_name: 'Poliana', last_name: 'Santana', middle_name: 'Santos'});

cqlsh> SELECT * FROM library.user;
 id         | name
------------+------------------------------------------------------------------------
    poliana |   {first_name: 'Poliana', last_name: 'Santana', middle_name: 'Santos'}
 otaviojava | {first_name: 'Otavio', last_name: 'Santana', middle_name: 'Gonçalves'}
```

Também é possível remover remover o UDT como `DROP UDT`.

```sql
drop_type_statement ::=  DROP TYPE [ IF EXISTS ] udt_name

```

Por exemplo, ao remover o campo nome da família de colunas:

```sql
ALTER COLUMNFAMILY library.user DROP name;
cqlsh> SELECT * FROM library.user;
 id
------------
    poliana
 otaviojava

DROP TYPE IF EXISTS library.name;
```

As coleções do Cassandra também têm suporte aos UDTs, para isso é necessário utilizar o keyword `frozen`. Vale salientar que o UDT tem o seu funcionamento semelhante ao de uma tupla, ou seja, não é possível alterar um único elemento.

```sql
DROP TABLE IF EXISTS library.user;
DROP TABLE IF EXISTS library.user;
CREATE TYPE IF NOT EXISTS library.phone (
    country_code int,
    number text,
);
CREATE TABLE IF NOT EXISTS library.user (
    id text,
    phones set<frozen<phone>>,
    PRIMARY KEY (id)
);
```

```

```

## Manipulando informação

Nesta seção abordaremos a manipulação de dados, será possível utilizar o famoso CRUD: criar, recuperar, atualizar, deletar as informações.

### SELECT

O `SELECT` é o tipo de comando utilizado para recuperar as informações dos dados.

```sql
select_statement ::=  SELECT [ JSON | DISTINCT ] ( select_clause | '*' )
                      FROM table_name
                      [ WHERE where_clause ]
                      [ GROUP BY group_by_clause ]
                      [ ORDER BY ordering_clause ]
                      [ PER PARTITION LIMIT (integer | bind_marker) ]
                      [ LIMIT (integer | bind_marker) ]
                      [ ALLOW FILTERING ]
select_clause    ::=  selector [ AS identifier ] ( ',' selector [ AS identifier ] )
selector         ::=  column_name
                      | term
                      | CAST '(' selector AS cql_type ')'
                      | function_name '(' [ selector ( ',' selector )* ] ')'
                      | COUNT '(' '*' ')'
where_clause     ::=  relation ( AND relation )*
relation         ::=  column_name operator term
                      '(' column_name ( ',' column_name )* ')' operator tuple_literal
                      TOKEN '(' column_name ( ',' column_name )* ')' operator term
operator         ::=  '=' | '<' | '>' | '<=' | '>=' | '!=' | IN | CONTAINS | CONTAINS KEY
group_by_clause  ::=  column_name ( ',' column_name )*
ordering_clause  ::=  column_name [ ASC | DESC ] ( ',' column_name [ ASC | DESC ] )*
```

Por exemplo, na livraria será adicionada uma família de colunas do tipo revista que terá a lista de artigos e o ano do post.


```sql
DROP COLUMNFAMILY IF EXISTS library.magazine;
CREATE COLUMNFAMILY IF NOT EXISTS library.magazine (
    id text,
    posted_at timestamp,
    articles set<text>,
    pages int,
    PRIMARY KEY (id, posted_at)
);

INSERT INTO library.magazine (id, posted_at, articles, pages) values ('Java Magazine', '2018-01-01',{'Jakarta EE', 'Java 8', 'Cassandra'}, 140);
INSERT INTO library.magazine (id, posted_at, articles, pages) values ('Java Magazine', '2017-01-01',{'Java EE 8', 'Java 7', 'NoSQL'}, 100);
```

Dentro de uma query é possível recuperar tanto todos os campos quanto colunas específicas utilizando o mesmo padrão da linguagem SQL.

```sql
cqlsh> SELECT * FROM library.magazine;

 id            | posted_at                       | articles                              | pages
---------------+---------------------------------+---------------------------------------+-------
 Java Magazine | 2017-01-01 00:00:00.000000+0000 |      {'Java 7', 'Java EE 8', 'NoSQL'} |   100
 Java Magazine | 2018-01-01 00:00:00.000000+0000 | {'Cassandra', 'Jakarta EE', 'Java 8'} |   140


cqlsh> SELECT id, posted_at FROM library.magazine;

 id            | posted_at
---------------+---------------------------------
 Java Magazine | 2017-01-01 00:00:00.000000+0000
 Java Magazine | 2018-01-01 00:00:00.000000+0000

cqlsh> SELECT count(*) FROM library.magazine;

 count
-------
     2

```

Dentro do Cassandra também existem algumas queries de agregação de valores, com a keyword `GROUP BY`, porém, para utilizar esse recurso é necessário que o campo alvo seja uma chave primária.

```sql
cqlsh> SELECT id, max(pages) FROM library.magazine GROUP BY id;

 id            | system.max(pages)
---------------+-------------------
 Java Magazine |               140

cqlsh> SELECT id, min(pages) FROM library.magazine GROUP BY id;

  id            | system.min(pages)
 ---------------+-------------------
  Java Magazine |               100


cqlsh> SELECT id, sum(pages) FROM library.magazine GROUP BY id;

 id            | system.sum(pages)
---------------+-------------------
 Java Magazine |               240

```

No mundo ideal, todas as queries são realizas a partir da chave, ou seja, sempre haverá o retorno de um único resultado. Porém, em alguns casos, é necessário realizar buscas utilizando índices secundários, trazendo um grande volume de resultados. Uma maneira de evitar essa quantidade é limitando o valor máximo do retorno e o Cassandra tem esse recurso. Para isso, basta utilizar o `LIMIT`.

```sql
cqlsh> SELECT * FROM library.magazine LIMIT 1;

 id            | posted_at                       | articles                         | pages
---------------+---------------------------------+----------------------------------+-------
 Java Magazine | 2017-01-01 00:00:00.000000+0000 | {'Java 7', 'Java EE 8', 'NoSQL'} |   100
```

A ordenação dos dados é feita a partir do `ORDER BY`, porém, ele não é tão poderoso quanto nos bancos relacionais uma vez que a ordenação só é realizada pelos campos do tipo chave.

```sql
 SELECT * FROM library.magazine where id = 'Java Magazine' ORDER BY posted_at DESC;
 id            | posted_at                       | articles                              | pages
---------------+---------------------------------+---------------------------------------+-------
 Java Magazine | 2018-01-01 00:00:00.000000+0000 | {'Cassandra', 'Jakarta EE', 'Java 8'} |   140
 Java Magazine | 2017-01-01 00:00:00.000000+0000 |      {'Java 7', 'Java EE 8', 'NoSQL'} |   100


 SELECT * FROM library.magazine where id = 'Java Magazine' ORDER BY posted_at ASC;

  id            | posted_at                       | articles                              | pages
 ---------------+---------------------------------+---------------------------------------+-------
  Java Magazine | 2017-01-01 00:00:00.000000+0000 |      {'Java 7', 'Java EE 8', 'NoSQL'} |   100
  Java Magazine | 2018-01-01 00:00:00.000000+0000 | {'Cassandra', 'Jakarta EE', 'Java 8'} |   140


```

Em uma query de busca também é possível recuperar os valores no formato *JSON*.

```sql
cqlsh> SELECT JSON * FROM library.magazine;

 [json]
-----------------------------------------------------------------------------------------------------------------------------------
      {"id": "Java Magazine", "posted_at": "2017-01-01 00:00:00.000Z", "articles": ["Java 7", "Java EE 8", "NoSQL"], "pages": 100}
 {"id": "Java Magazine", "posted_at": "2018-01-01 00:00:00.000Z", "articles": ["Cassandra", "Jakarta EE", "Java 8"], "pages": 140}

cqlsh> SELECT JSON id, pages FROM library.magazine;

 [json]
---------------------------------------
 {"id": "Java Magazine", "pages": 100}
 {"id": "Java Magazine", "pages": 140}

```

> É importante salientar que operações de buscas dentro do Cassandra são bem limitadas, de modo que a recomendação é fazer a partir das chaves, deixando o índice em último caso. Existe também o recurso do uso do `ALLOW FILTERING`, porém, utilizar esse recurso demanda um altíssimo impacto de performance.


### INSERT

Dentro da cláusula `INSERT` é possível inserir uma linha dentro da família de coluna.

```sql
insert_statement ::=  INSERT INTO table_name ( names_values | json_clause )
                      [ IF NOT EXISTS ]
                      [ USING update_parameter ( AND update_parameter )* ]
names_values     ::=  names VALUES tuple_literal
json_clause      ::=  JSON string [ DEFAULT ( NULL | UNSET ) ]
names            ::=  '(' column_name ( ',' column_name )* ')'
```

Para inserir os dados utilizando a cláusula `INSERT` é possível utilizá-la tanto de uma forma bem semelhante ao SQL e quanto como JSON. Por exemplo, utilizando a família de coluna `magazine` anterior.

```sql
INSERT INTO library.magazine (id, posted_at, articles, pages) values ('Java Magazine', '2017-01-01',{'Java EE 8', 'Java 7', 'NoSQL'}, 100);
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 10, "articles": ["Java EE 8", "Java 7", "NoSQL"]}';
```

A chave é obrigatória e é a partir dela que é definido onde a informação será enviada através do cluster.


```sql
INSERT INTO library.magazine JSON '{"posted_at": "2017-01-01", "pages": 10, "articles": ["Java EE 8", "Java 7", "NoSQL"]}';
InvalidRequest: Error from server: code=2200 [Invalid query] message="Invalid null value in condition for column id"
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 10}';
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01"}';
```

Além da verificação da chave primária, não existe nenhuma outra validação, ou seja, é possível diferentes valores com a mesma chave sem nenhum problema. Basicamente, se a chave não existir ela será criada, do contrário, ela será sobrescrita.

```sql
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 112}';
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 121}';
INSERT INTO library.magazine JSON '{"id": "Java Magazine", "posted_at": "2017-01-01", "pages": 90}';
```

Também é possível definir uma inserção com TTL, que significa o tempo de vida de uma linha em segundos.

```sql
TRUNCATE library.magazine ;
INSERT INTO library.magazine (id, posted_at) values ('Java Magazine', '2017-01-01') USING TTL 10;
cqlsh> SELECT * FROM library.magazine WHERE id = 'Java Magazine';

 id            | posted_at                       | articles | pages
---------------+---------------------------------+----------+-------
 Java Magazine | 2017-01-01 00:00:00.000000+0000 |     null |  null
//wait 10 seconds
cqlsh> SELECT * FROM library.magazine WHERE id = 'Java Magazine';
 id | posted_at | articles | pages
----+-----------+----------+-------

(0 rows)
```

### UPDATE

Para atualizar, existe a cláusula `UPDATE`. De uma maneira geral, para o Cassandra, nada mais é que um `INSERT` com o `WHERE`. Assim como no `INSERT`, caso a informação exista, será atualizada, do contrário, será sobrescrita. Por exemplo, dada uma simples família de colunas de bibliotecário, sua criação utilizando a cláusula `UPDATE` ficaria assim:

Criação da família de coluna bibliotecário, como é possível ver é inserido um bibliotecário mesmo utilizando a cláusula `UPDATE`.

```sql
DROP COLUMNFAMILY IF EXISTS library.librarian;

CREATE COLUMNFAMILY IF NOT EXISTS library.librarian (
    id text,
    name text,
    PRIMARY KEY (id)
);

UPDATE library.librarian set name = 'Ivar' where id = 'ivar';
cqlsh> SELECT * FROM library.librarian;

 id   | name
------+------
 ivar | Ivar

(1 rows)

```

Também é possível atualizar o TTL do registro.

```sql
UPDATE library.librarian USING TTL 12 set name = 'Daniel' where id = 'daniel';
cqlsh> SELECT * FROM library.librarian where id = 'daniel';
 id     | name
--------+--------
 daniel | Daniel
//wait 10 seconds
cqlsh> SELECT * FROM library.librarian where id = 'daniel';

 id | name
----+------
```

### DELETE

Com a cláusula `DELETE` se pode remover registros dentro do Cassandra, seja apenas uma coluna ou todo o registro, sendo que para qualquer operação é necessária a chave primária, a partition key.


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

 id       | name
----------+------------------
     dani |      Daniel Dias
 gabriela | Gabriela Santana
     ivar |             Ivar

(3 rows)

DELETE FROM library.librarian where id = 'dani';
cqlsh> SELECT * FROM library.librarian;
 id       | name
----------+------------------
 gabriela | Gabriela Santana
     ivar |             Ivar

(2 rows)
DELETE name FROM library.librarian where id = 'ivar';
cqlsh> SELECT * FROM library.librarian;
 id       | name
----------+------------------
 gabriela | Gabriela Santana
     ivar |             null

(2 rows)

```

### BATCH

O `BATCH` permite realizar múltiplas operações de alteração (INSERT, UPDATE e DELETE). As operações dentro de um `BATCH` têm como objetivo realizar diversas operações de maneira atômica, ou seja, ou a operação acontece ou não.

É importante salientar que:

* Os `BATCH` podem conter apenas INSERT, UPDATE e DELETE;
* Os comandos `BATCH` não têm total suporte à transação como os bancos relacionais;
* Todas as operações pertencem a _partition key_ para garantir isolamento;
* Por padrão as operações dentro do `BATCH` serão atômicas, assim as operações serão eventualmente completas ou nenhuma operação acontecerá.
* Existe um grande _trade-off_ no uso do `BATCH`: de um lado, pode economizar rede na comunicação entre nó coordenador e os outros nós para as operações, do outro lado, pode sobreutilizar um nó para diversas operações, daí a importância de saber usar esse recurso com parcimônia.

```sql
batch_statement        ::=  BEGIN [ UNLOGGED | COUNTER ] BATCH
                            [ USING update_parameter ( AND update_parameter )* ]
                            modification_statement ( ';' modification_statement )*
                            APPLY BATCH
modification_statement ::=  insert_statement | update_statement | delete_statement
```

Por exemplo, para realizar operações dentro da família de coluna dos bibliotecários.

```sql
BEGIN BATCH
INSERT INTO library.librarian JSON '{"name": "Ivar", "id": "ivar"}';
INSERT INTO library.librarian JSON '{"name": "Daniel Dias", "id": "dani"}';
INSERT INTO library.librarian JSON '{"name": "Gabriela Santana", "id": "gabriela"}';
DELETE FROM library.librarian where id = 'dani';
APPLY BATCH;

cqlsh> SELECT * FROM library.librarian;

 id       | name
----------+------------------
 gabriela | Gabriela Santana
     ivar |             Ivar

(2 rows)

```

### View materializada

A desnormalização é a melhor amiga dos bancos de dados do tipo não relacionais, e com o Cassandra isso não é uma exceção. Um dos recursos que podem facilitar nessa modelagem é a criação de views materializadas. Isso serve como um aliado para a desnormalização, ao mesmo tempo que garante a consistência de uma família de coluna. Um ponto importante é que a view materializada não pode sofrer nenhuma alteração.


```sql
create_materialized_view_statement ::=  CREATE MATERIALIZED VIEW [ IF NOT EXISTS ] view_name AS
                                            select_statement
                                            PRIMARY KEY '(' primary_key ')'
                                            WITH table_options
```

Por exemplo, considerando novamente o caso do livro e que o ISBN (International Standard Book Number) é o código que os funcionários utilizam para emprestar um livro dentro da biblioteca, seria possível criar a família de coluna da seguinte forma:

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

Agora é possível realizar buscas de maneira tranquila a partir do ISBN. Porém, considerando que o ISBN é feito de maneira incremental, gostaríamos de retornar os livros mais recentes da biblioteca e, para isso, será criada uma view materializada para ISBN maiores que 3. Assim:

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS library.recent_book AS
    SELECT * FROM library.book
    WHERE isbn > 3 AND name IS NOT NULL
    PRIMARY KEY (isbn, name);

Warnings : Materialized views are experimental and are not recommended for production use.
```

Com a view materializada criada é possível realizar buscas de maneira semelhante ao que se faz a família de colunas.

```sql
cqlsh> SELECT * FROM library.recent_book;

 isbn | name | author
------+------+--------

INSERT INTO library.book JSON '{"isbn": 4, "name": "Java EE 8 Cookbook", "author": "Elder Moraes"}';
INSERT INTO library.book JSON '{"isbn": 5, "name": "Best Developer Job Ever!", "author": "Bruno Souza"}';

cqlsh> SELECT * FROM library.recent_book;
 isbn | name                     | author
------+--------------------------+--------------
    4 |       Java EE 8 Cookbook | Elder Moraes
    5 | Best Developer Job Ever! |  Bruno Souza

```

Também é possível realizar a alteração ou remover a view materializada.


```sql
DROP MATERIALIZED VIEW library.recent_book;
```


> Até o momento o recurso de view materializada se encontra em caráter experimental.

## Segurança


O Cassandra tem um importante suporte ao recurso de segurança. Assim, é possível definir permissão, criar usuário, criar regras, remover permissão dentre outros. O primeiro passo para aplicarmos isso no projeto é habilitar o recurso de segurança pelo Cassandra. Para isso, é necessário realizar uma mudança dentro do `cassandra.yaml` dentro da pasta `conf`: trocar a linha `authenticator: AllowAllAuthenticator` para `authenticator: PasswordAuthenticator`.
Para habilitar o gerenciamento de permissão do Cassandra também é necessário alterar a linha `authorizer: AllowAllAuthorizer` para `authorizer: CassandraAuthorizer`.


Caso esteja utilizando Docker, a solução seria mapear o local de configuração. Assim:

```bash
docker run --name some-cassandra -p 9042:9042 -v /my/own/datadir:/var/lib/cassandra -v  /path/to/config/cassandra.yaml:/etc/cassandra/cassandra.yaml -d cassandra
//sample
docker run --name some-cassandra -p 9042:9042 -v /home/otaviojava/data:/var/lib/cassandra -v /home/otaviojava/config:/etc/cassandra -d cassandra
```

Feita a habilitação, quando acessar Cassandra novamente haverá uma mensagem de erro:

```bash
./cqlsh
Connection error: ('Unable to connect to any servers', {'127.0.0.1': AuthenticationFailed('Remote end requires authentication.',)})
```

Esse erro acontece porque é necessário ter autorização de acesso.

> O superusuário padrão é `cassandra` e a senha `cassandra`.

```bash
./cqlsh -u cassandra -p cassandra
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cassandra@cqlsh>
```

Para Docker, o acesso é semelhante, basta procurar o ID do contêiner e depois adicionar os parâmetros de usuário e senha.

```bash
 docker exec -it 66e5f38a3815 cqlsh -u cassandra -p cassandra
```

Toda a criação foi definida a partir dos `ROLES`. Por exemplo, é possível criar alguns usuários.

As usuárias `ada` e `alice` criadas:
```sql
CREATE ROLE ada;
CREATE ROLE alice WITH PASSWORD = 'alice' AND LOGIN = true;
cqlsh> LIST ROLES;
 role      | super | login | options
-----------+-------+-------+---------
       ada | False | False |        {}
     alice | False |  True |        {}
 cassandr
```

Agora é possível, realizar o login com a usuária `alice`.

```sql
./cqlsh -u alice -p alice
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
alice@cqlsh>
```

Uma vez que `alice` não tem permissão de superusuário, não será possível remover campos sensíveis do sistema.

```sql
alice@cqlsh> DROP KEYSPACE IF EXISTS system_auth;
Unauthorized: Error from server: code=2100 [Unauthorized] message="Cannot DROP <keyspace system_auth>"
```

É possível alterar a informação de usuário, por exemplo, para fazer com que `alice` tenha permissão de superusuário.

```sql
ALTER ROLE alice WITH SUPERUSER = true;

cassandra@cqlsh> LIST ROLES;
 role      | super | login | options
-----------+-------+-------+---------
       ada | False | False |        {}
     alice |  True |  True |        {}
 cassandra |  True |  True |        {}

```

Assim como remover usuário:

```sql
DROP ROLE ada;
cassandra@cqlsh> LIST ROLES;
 role      | super | login | options
-----------+-------+-------+---------
     alice |  True |  True |        {}
 cassandra |  True |  True |        {}

(2 rows)

```


Com os usuários criados, agora é possível, por exemplo, criar regras e definir as permissões para cada estrutura dentro do banco de dados. Veja que as opções possuem as mesmas estruturas de um banco de dados relacional. As opções disponíveis são:


* CREATE
* ALTER
* DROP
* SELECT
* MODIFY
* AUTHORIZE
* DESCRIBE
* EXECUTE

```sql
grant_permission_statement ::=  GRANT permissions ON resource TO role_name
permissions                ::=  ALL [ PERMISSIONS ] | permission [ PERMISSION ]
permission                 ::=  CREATE | ALTER | DROP | SELECT | MODIFY | AUTHORIZE | DESCRIBE | EXECUTE
resource                   ::=  ALL KEYSPACES
                               | KEYSPACE keyspace_name
                               | [ TABLE ] table_name
                               | ALL ROLES
                               | ROLE role_name
                               | ALL FUNCTIONS [ IN KEYSPACE keyspace_name ]
                               | FUNCTION function_name '(' [ cql_type ( ',' cql_type )* ] ')'
                               | ALL MBEANS
                               | ( MBEAN | MBEANS ) string
```


Para elucidar o contexto de segurança dentro do Cassandra, imagine o seguinte cenário dentro do sistema de livraria:

* A regra de `user` apenas pode ler dentro do keyspace da biblioteca;
* A regra de `librarian` pode realizar alterações no keyspace da biblioteca;
* A regra de `manager` pode criar tabelas dentro do keyspace da biblioteca.

Com base nessas regras serão criados:

* `ada` como usuária;
* `mike` como bibliotecário;
* `jonh` como manager.


```sql
CREATE KEYSPACE IF NOT EXISTS library  WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};
//create the role user
CREATE ROLE IF NOT EXISTS user;
GRANT SELECT ON KEYSPACE library TO user;
//create user ada
CREATE ROLE ada WITH PASSWORD = 'ada' AND LOGIN = true;
GRANT user TO ada;
//create the role librarian
CREATE ROLE IF NOT EXISTS librarian;
GRANT MODIFY ON KEYSPACE library TO librarian;
GRANT SELECT ON KEYSPACE library TO librarian;
//create mike
CREATE ROLE mike WITH PASSWORD = 'mike' AND LOGIN = true;
GRANT librarian TO mike;
//create manager
CREATE ROLE IF NOT EXISTS manager;
GRANT ALTER ON KEYSPACE library TO manager;
GRANT CREATE ON KEYSPACE library TO manager;
//create jonh
CREATE ROLE jonh WITH PASSWORD = 'jonh' AND LOGIN = true;
GRANT manager TO jonh;
```

Com todos os usuários criados, o primeiro passo é explorar um pouco da segurança dentro do Cassandra.

```sql
./cqlsh -u ada -p ada

CREATE COLUMNFAMILY IF NOT EXISTS library.magazine (
    id text,
    pages int,
    PRIMARY KEY (id)
);

Unauthorized: Error from server: code=2100 [Unauthorized] message="User ada has no CREATE permission on <keyspace library> or any of its parents"

./cqlsh -u jonh -p jonh

CREATE COLUMNFAMILY IF NOT EXISTS library.magazine (
   id text,
    pages int,
    PRIMARY KEY (id)
);

 ./cqlsh -u mike -p mike

INSERT INTO library.magazine JSON '{"id": "new magazine", "pages": 10}';
SELECT * FROM library.magazine;

 id           | pages
--------------+-------
 new magazine |    10

(1 rows)

./cqlsh -u ada -p ada

SELECT * FROM library.magazine ;

 id           | pages
--------------+-------
 new magazine |    10

INSERT INTO library.magazine JSON '{"id": "new magazine", "pages": 10}';
Unauthorized: Error from server: code=2100 [Unauthorized] message="User ada has no MODIFY permission on <table library.magazine> or any of its parents"

```

Com a segurança ativada dentro do Cassandra, os usuários terão permissão para realizar algumas operações a partir do `role`. Essa é uma maneira bem interessante de garantir que apenas sistemas ou pessoas tenham acesso de leitura ou escrita em pontos de que elas realmente precisam.

### Conclusão

Neste capítulo foi possível mostrar que o Cassandra Query Language ou CQL tem muitas mais semelhanças com o SQL além do nome, de modo que alguém que já conhece bem os bancos de dados relacionais e sua sintaxe de comunicação terá uma baixa curva de aprendizagem ao aprender o Cassandra. 

Um dos pontos importantes que este capítulo explorou foi o uso de segurança que existe dentro do Cassandra. É sempre importante se preocupar com a segurança seja com firewalls de acesso seja com permissão de usuário e senha para pontos do banco de dados.
