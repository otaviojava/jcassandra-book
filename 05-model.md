# Modelando sua aplicação com Cassandra


Dentro da Ciência de Dados, a modelagem certamente é um dos pontos mais enigmáticos e mais desafiadores. Um erro nesse ponto significará impacto de performance tanto na leitura quanto na escrita de informações no banco de dados. No mundo relacional, quem desenvolve já está acostumado com o conceito de normalização - que é um conjunto de regras que visa, principalmente, à organização dentro do banco de dados relacional para evitar a redundância de dados. 

O ponto a ser discutido é que, quando surgiram os bancos relacionais, eles apresentavam o foco de evitar a redundância, afinal, era um grande desafio, já que o preço de armazenamento era algo realmente muito caro. Por exemplo, em 1980 um servidor que armazenava vinte e seis megabytes tinha um custo de quase cinco mil dólares, atualmente, um terabyte custa menos de cinquenta dólares.

![Um HD com 5MB em 1956 fabricado pela IBM. Fonte:  https://thenextweb.com/shareables/2011/12/26/this-is-what-a-5mb-hard-drive-looked-like-is-1956-required-a-forklift/ {w=40%}](imagens/ibm_history_server.png)

A consequência da normalização é que, com uma grande complexidade dos dados e a variedade, para manter os dados não duplicados é necessário realizar uma alta quantidade de joins, a união de várias tabelas, e por consequência há um aumento no tempo de resposta dessas queries. O desafio atual se encontra no tempo de resposta e não mais no armazenamento, como era anteriormente. 

O objetivo deste capítulo é demonstrar dicas e motivações para realizar uma modelagem dentro do Cassandra.

## Dicas de modelagem

Diferente dos bancos relacionais em que existem as regras de normalização, dentro do Cassandra os princípios de modelagem são definidos muito mais pelo contexto da aplicação ou volumetria. Ou seja, um mesmo sistema de e-commerce, por exemplo, pode ser modelado de maneira diferente a depender dos recursos e volumetria de cada um desses negócios. Como a maioria dos desenvolvedores começa com relacional, as primeiras dicas serão justamente entender quais *não* são os objetivos da modelagem com o Cassandra:

* Minimizar o número de leituras: dentro do Cassandra a escrita é relativamente barata e de maneira eficiente, diferente da leitura; assim, priorize a escrita e tente ler as informações pelo identificador único o máximo possível.
* Normalização ou minimizar dados duplicados: desnormalização e duplicação de dados são as melhores amigas para uma modelagem dentro do Cassandra. Facilitar o máximo possível a leitura é sempre a melhor estratégia dentro do Cassandra.
* Emular: não tente emular ou simular de alguma maneira os bancos relacionais dentro do Cassandra. O resultado será semelhante a usar um garfo como faca.

Um bom primeiro passo para começar a modelagem é saber de quais queries a aplicação precisa. Assim, salientando, criar famílias de colunas que satisfaçam aquela requisição apenas em uma query é o grande objetivo. Existem vários casos de modelagem dentro do mundo Cassandra. Para o próximo tópico, o próximo passo será demonstrar alguns exemplos de modelagem.

### Casos um para um

Imagine o seguinte cenário: cada livro (ISBN, nome e ano) tem o seu autor (nome e país).

A primeira pergunta para a modelagem: quais queries queremos suportar?

Para esse caso, gostaríamos de buscar os livros a partir do ISBN uma vez que nesse caso não existe motivo para que se exista uma família de coluna para autores.

```sql
CREATE KEYSPACE IF NOT EXISTS library  WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};

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

No primeiro exemplo, houve a criação do tipo autor do qual ele faz parte.

```sql
INSERT INTO library.book (isbn, name, year, author) values (1,'Clean Code', 2008, {name: 'Robert Cecil Martin', country: 'USA'});
INSERT INTO library.book (isbn, name, year, author) values (2,'Clean Architecture', 2017, {name: 'Robert Cecil Martin', country: 'USA'});
INSERT INTO library.book (isbn, name, year, author) values (3,'Agile Principles, Patterns, and Practices in C#', 2002, {name: 'Robert Cecil Martin', country: 'USA'});
INSERT INTO library.book (isbn, name, year, author) values (4,'Java EE 8 Cookbook', 2018, {name: 'Elder Moraes', country: 'Brazil'});
INSERT INTO library.book (isbn, name, year, author) values (5,'Effective Java', 2001, {name: 'Joshua Bloch', country: 'USA'});
INSERT INTO library.book (isbn, name, year, author) values (6,'Java Puzzlers: Traps, Pitfalls, and Corner Cases', 2005, {name: 'Joshua Bloch', country: 'USA'});
```

No cenário é possível ver o impacto na duplicação das informações, no caso dos autores. Porém, a mudança de nome e nacionalidade de um autor tende a ser algo extremamente raro.

### Casos um para N

Seguindo com o exemplo anterior, na relação livro-autor, este exemplo ampliará o número de autores, afinal, existem livros que possuem mais de um escritor. Com isso, existirá uma relação de um livro para N autores. Porém, como o objetivo da leitura não mudou, a modelagem não modificará.

```sql
CREATE KEYSPACE IF NOT EXISTS library  WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};

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
    authors set<frozen<author>>,
    PRIMARY KEY (isbn)
);
//INSERT
INSERT INTO library.book (isbn, name, year, authors) values (1,'Design Patterns: Elements of Reusable Object-Oriented Software', 1994,
{{name: 'Erich Gamma', country: 'Switzerland'}, {name: 'John Vlissides', country: 'USA'}});
INSERT INTO library.book (isbn, name, year, authors) values (2,'The Pragmatic Programmer', 1999,
{{name: 'Andy Hunt', country: 'Switzerland'}, {name: 'Dave Thomas', country: 'England'}});
```

O impacto segue semelhante ao anterior: um alto grau de dados duplicados. Isso será de grande incômodo para quem vier do mundo relacional, porém, é importante salientar: **desnormalização será a melhor amiga da modelagem no Cassandra**.

### Casos N para N

Para o último exemplo, o objetivo é criar uma relação N para N. Seguindo o escopo de negócio de uma biblioteca, é possível pensar na relação entre revistas e artigos porque uma revista possui N artigos da mesma forma que um artigo pode estar em N revistas. Independente disso, no Cassandra a pergunta é a mesma: quais queries o banco de dados deseja suportar? Nesse cenário existirão duas:

* Buscar a revista pelo ISSN
* Buscar o artigo pelo título


```sql
CREATE KEYSPACE IF NOT EXISTS library  WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};

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
    author frozen<author>
);

CREATE TYPE IF NOT EXISTS library.article (
    name text,
    year int,
    author frozen<author>
);

CREATE COLUMNFAMILY IF NOT EXISTS library.magazine (
   issn bigint,
   name text,
   editor author,
   articles set<frozen<article>>,
   PRIMARY KEY (issn)
);

CREATE COLUMNFAMILY IF NOT EXISTS library.article (
   title text,
   year int,
   author author,
   magazines set<frozen<magazine>>,
   PRIMARY KEY (title, year)
);
//INSERT
INSERT INTO library.magazine (issn, name, editor, articles) values (1, 'Java Magazine', {name: 'Java Editor', country: 'USA'}, {{name: 'Jakarta EE', year: 2018, author: {name: 'Elder Moraes', country: 'Brazil'}},
{name: 'Cloud and Docker', year: 2018, author: {name: 'Bruno Souza', country: 'Brazil'}}});
```

Nesse caso, é bastante simples uma vez que ambas as entidades, revistas e artigos, não necessitam realizar alteração.

### Conclusão

Com isso foram apresentados os conceitos de modelagem dentro de um banco de dados Cassandra, demonstrando como ele é diferente de uma base de dados relacional. O núcleo de seu conceito é justamente escrever ao máximo, uma vez que tende a ser uma operação barata, para justamente diminuir o número de operações na leitura. No mundo ideal é possível criar queries e consultas sem o uso de índices, seja ele como secundário ou com o recurso de `ALLOW FILTERING`, porém, existirá um trabalho gigantesco para gerenciar a duplicação de dados que ocorrerá.
