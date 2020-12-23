# Realizando integração com Java

Após toda a teoria, instalação, noção de comunicação com CQL e modelagem, finalmente está na hora de juntar tudo isso e colocar dentro de uma aplicação Java. A integração do banco de dados com a linguagem é uma das coisas mais importantes a serem realizadas dentro de uma aplicação e é esse o objetivo deste capítulo. 

## Requisitos mínimos para as demonstrações

Para executar todas as demonstrações será necessário ter o Java 8 instalado, além do Maven superior à versão 3.5.3. No momento em que eu escrevo, o Java se encontra na versão 11, porém, poucos frameworks têm suporte para ele, dessa forma, para manter coerência e facilitar o gerenciamento das variáveis de ambiente, será utilizada a última versão do Java 8. 

Como o objetivo do livro é falar sobre o Cassandra, consideramos que você tenha noção de Java 8 e Maven, além de ter os dois instalados e devidamente configurados no sistema operacional favorito. Você utilizar qualquer IDE com que se sinta confortável.

Para facilitar a comparação entre as ferramentas de integração com Java será utilizado um exemplo simples. Com o intuito de diminuir a complexidade da aplicação e não desviar o foco, os exemplos serão baseados apenas em Java SE, ou seja, executando puramente o bom e velho `public static void main(String[])`.

O nosso exemplo será baseado no mesmo caso de uma livraria.
Teremos uma biblioteca que terá livros e cada livro terá autor e uma categoria. As consultas desejadas são:

* Buscar o livro a partir do ISBN;
* Buscar os livros a partir da respectiva categoria.

Pensando de uma maneira simples, uma modelagem que atenda a todas as queries e distribua as informações seria assim:

```sql
CREATE KEYSPACE IF NOT EXISTS library  WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};
DROP COLUMNFAMILY IF EXISTS library.book;
DROP COLUMNFAMILY IF EXISTS library.category;
DROP TYPE IF EXISTS library.book;

CREATE TYPE IF NOT EXISTS library.book (
    isbn bigint,
    name text,
    author text,
    categories set<text>
);

CREATE COLUMNFAMILY IF NOT EXISTS library.book (
    isbn bigint,
    name text,
    author text,
    categories set<text>,
    PRIMARY KEY (isbn)
);


CREATE COLUMNFAMILY IF NOT EXISTS library.category (
  name text PRIMARY KEY,
  books set<frozen<book>>
);
```

Com esse código criamos a estrutura inicial, ou seja, a família de coluna `livro` para a seguir integrar o Cassandra com o código Java.

## Esqueleto dos projetos exemplos

Como já foi mencionado, os exemplos serão baseados no Java SE, assim, o nosso primeiro passo com o Java é criar o projeto. Utilizaremos o _maven archetype_ para acelerar a criação desse projeto maven. Acesse o terminal e execute o seguinte comando:

```bash
mvn archetype:generate -DgroupId=com.mycompany.app -DartifactId=my-app -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

Um ponto importante é que dentro do `pom.xml` é importante atualizá-lo para ter suporte ao Java 8. Assim, o arquivo ficaria desta forma:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
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
                    <source>8</source>
                    <target>8</target>
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

> Para cada novo projeto uma boa estratégia seria criá-lo utilizando o mesmo esqueleto.

## Utilizando o driver do Cassandra

Para começar a desenvolver o código, o primeiro passo é adicionar a dependência do driver dentro do arquivo que gerencia as dependências do Maven, ou seja, o `pom.xml` dentro da tag `dependencies`:

```xml
<dependency>
    <groupId>com.datastax.cassandra</groupId>
    <artifactId>cassandra-driver-core</artifactId>
    <version>3.6.0</version>
</dependency>
```

> Para facilitar, não ativaremos a segurança e estaremos baseados em uma instalação local ou com o Docker com o seguinte comando: `docker run -d --name casandra-instance -p 9042:9042 cassandra`.

Com a dependência dentro do projeto, o próximo passo é iniciar a comunicação. O `Cluster` é a classe que representa a estrutura de nós do Cassandra. Com ele, é possível utilizar o `try-resource`, dessa maneira, tão logo o código seja encerrado o cluster é encerrado. A interface `Session` representa a conexão com o Cassandra, assim, é a partir dela que se executam os comandos de inserção e atualização dentro do banco de dados.

```java
public class App {

    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "book";
    private static final String[] NAMES = new String[]{"isbn", "name", "author", "categories"};

    public static void main(String[] args) {
        try (Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build()) {

            Session session = cluster.connect();

            Object[] cleanCode = new Object[]{1, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO")};
            Object[] cleanArchitecture = new Object[]{2, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice")};
            Object[] effectiveJava = new Object[]{3, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice")};
            Object[] nosql = new Object[]{4, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice")};

            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, cleanCode));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, cleanArchitecture));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, effectiveJava));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, nosql));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, cleanCode));

            ResultSet resultSet = session.execute(QueryBuilder.select().from(KEYSPACE, COLUMN_FAMILY));
            for (Row row : resultSet) {
                Long isbn = row.getLong("isbn");
                String name = row.getString("name");
                String author = row.getString("author");
                Set<String> categories = row.getSet("categories", String.class);
                System.out.println(String.format(" the result is %s %s %s %s", isbn, name, author, categories));
            }
        }

    }

}
```

Na primeira classe de interação com o Cassandra, é possível perceber o uso intenso do Cassandra Query Language, que explicamos no capítulo 4. Todas as operações com o CQL são facilitadas a partir da classe `QueryBuilder`, que é uma utilitária que contém diversos métodos que facilitam a vida do desenvolvedor para criar CQL para uma aplicação.

Na nossa primeira interação realizamos uma inserção e uma busca de todos os valores dentro do banco de dados, mas esse tipo de busca não é muito comum e tampouco otimizado. Como mencionamos no capítulo de modelagem, o mais recomendado é sempre realizar uma busca pela chave. Assim, na nossa próxima interação realizaremos buscas a partir da chave. Para facilitar um pouco o código, criaremos um `Consumer` para logar o resultado (não será nada sofisticado, apenas o bom e velho `System.out.println`).

```java
public class App2 {

    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "book";
    private static final String[] NAMES = new String[]{"isbn", "name", "author", "categories"};

    public static void main(String[] args) {
        try (Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build()) {

            Session session = cluster.connect();

            Object[] cleanCode = new Object[]{1, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO")};
            Object[] cleanArchitecture = new Object[]{2, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice")};
            Object[] effectiveJava = new Object[]{3, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice")};
            Object[] nosql = new Object[]{4, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice")};

            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, cleanCode));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, cleanArchitecture));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, effectiveJava));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, nosql));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, cleanCode));

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

    private static void deleteById(Session session, Long isbn) {
        session.execute(QueryBuilder.delete().from(KEYSPACE, COLUMN_FAMILY).where(QueryBuilder.eq("isbn", isbn)));

    }

    private static void findById(Session session, long isbn, Consumer<Row> log) {
        ResultSet resultSet = session.execute(QueryBuilder.select().from(KEYSPACE, COLUMN_FAMILY).where(QueryBuilder.eq("isbn", isbn)));
        resultSet.forEach(log);
    }

}
```

Com o código anterior foi possível realizar as operações básicas do CRUD com os tipos triviais no Cassandra. Seguindo e evoluindo o nosso exemplo, realizaremos manipulações com um tipo customizado do Cassandra, ou UDT dentro da família de coluna `Category`.

```java
public class App3 {

    private static final String KEYSPACE = "library";
    private static final String TYPE = "book";
    private static final String COLUMN_FAMILY = "category";
    private static final String[] NAMES = new String[]{"name", "books"};

    public static void main(String[] args) {
        try (Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build()) {

            Session session = cluster.connect();

            UserType userType = session.getCluster().getMetadata().getKeyspace(KEYSPACE).getUserType(TYPE);
            UDTValue cleanCode = getValue(userType, 1, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO", "Good practice", "Design"));
            UDTValue cleanArchitecture = getValue(userType, 2, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("OO", "Good practice"));
            UDTValue effectiveJava = getValue(userType, 3, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "OO", "Good practice"));
            UDTValue nosql = getValue(userType, 4, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));

            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, new Object[]{"Java", Sets.newHashSet(cleanCode, effectiveJava)}));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, new Object[]{"OO", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture)}));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, new Object[]{"Good practice", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture, nosql)}));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, new Object[]{"NoSQL", Sets.newHashSet(nosql)}));

            ResultSet resultSet = session.execute(QueryBuilder.select().from(KEYSPACE, COLUMN_FAMILY));
            for (Row row : resultSet) {
                String name = row.getString("name");
                Set<UDTValue> books = row.getSet("books", UDTValue.class);
                Set<String> logBooks = new HashSet<>();
                for (UDTValue book : books) {
                    long isbn = book.getLong("isbn");
                    String bookName = book.getString("name");
                    String author = book.getString("author");
                    logBooks.add(String.format(" %d %s %s", isbn, bookName, author));
                }
                System.out.println(String.format("The result %s %s", name, logBooks));

            }
        }

    }

    private static UDTValue getValue(UserType userType, long isbn, String name, String author, Set<String> categories) {
        UDTValue udtValue = userType.newValue();
        TypeCodec<Object> textCodec = CodecRegistry.DEFAULT_INSTANCE.codecFor(DataType.text());
        TypeCodec<Object> setCodec = CodecRegistry.DEFAULT_INSTANCE.codecFor(DataType.set(DataType.text()));
        TypeCodec<Object> bigIntCodec = CodecRegistry.DEFAULT_INSTANCE.codecFor(DataType.bigint());
        udtValue.set("isbn", isbn, bigIntCodec);
        udtValue.set("name", name, textCodec);
        udtValue.set("author", author, textCodec);
        udtValue.set("categories", categories, setCodec);
        return udtValue;

    }

}

```

Agora conseguimos realizar a inserção de um tipo customizado do Cassandra o `UDT`, ou seja, além de realizar a inserção também recuperamos um tipo `UDT` dentro do banco de dados Cassandra.

O Driver do Cassandra também possui o conceito de `PreparedStatement` que permite que sejam criadas queries dinamicamente, muito semelhante aos bancos de dados relacionais. 

```java
public class App4 {

    private static final String KEYSPACE = "library";
    private static final String TYPE = "book";
    private static final String COLUMN_FAMILY = "category";
    private static final String[] NAMES = new String[]{"name", "books"};

    public static void main(String[] args) {
        try (Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build()) {

            Session session = cluster.connect();

            UserType userType = session.getCluster().getMetadata().getKeyspace(KEYSPACE).getUserType(TYPE);
            UDTValue cleanCode = getValue(userType, 1, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO", "Good practice", "Design"));
            UDTValue cleanArchitecture = getValue(userType, 2, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("OO", "Good practice"));
            UDTValue effectiveJava = getValue(userType, 3, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "OO", "Good practice"));
            UDTValue nosql = getValue(userType, 4, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));

            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, new Object[]{"Java", Sets.newHashSet(cleanCode, effectiveJava)}));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, new Object[]{"OO", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture)}));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, new Object[]{"Good practice", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture, nosql)}));
            session.execute(QueryBuilder.insertInto(KEYSPACE, COLUMN_FAMILY).values(NAMES, new Object[]{"NoSQL", Sets.newHashSet(nosql)}));

            Consumer<Row> log = row -> {
                String name = row.getString("name");
                Set<UDTValue> books = row.getSet("books", UDTValue.class);
                Set<String> logBooks = new HashSet<>();
                for (UDTValue book : books) {
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

    private static void findById(Session session, String name, Consumer<Row> log) {
        ResultSet resultSet = session.execute(QueryBuilder.select().from(KEYSPACE, COLUMN_FAMILY).where(QueryBuilder.eq("name", name)));
        resultSet.forEach(log);
    }

    private static void deleteById(Session session, String name) {
        session.execute(QueryBuilder.delete().from(KEYSPACE, COLUMN_FAMILY).where(QueryBuilder.eq("name", name)));

    }

    private static UDTValue getValue(UserType userType, long isbn, String name, String author, Set<String> categories) {
        UDTValue udtValue = userType.newValue();
        TypeCodec<Object> textCodec = CodecRegistry.DEFAULT_INSTANCE.codecFor(DataType.text());
        TypeCodec<Object> setCodec = CodecRegistry.DEFAULT_INSTANCE.codecFor(DataType.set(DataType.text()));
        TypeCodec<Object> bigIntCodec = CodecRegistry.DEFAULT_INSTANCE.codecFor(DataType.bigint());
        udtValue.set("isbn", isbn, bigIntCodec);
        udtValue.set("name", name, textCodec);
        udtValue.set("author", author, textCodec);
        udtValue.set("categories", categories, setCodec);
        return udtValue;

    }

}
```

Com isso, realizamos um exemplo completo com o Driver do Cassandra, e completamos todo o ciclo do CRUD dentro da família de coluna `Category`. Utilizar o Driver é muito bom uma vez que temos acesso a todas as queries dentro do CQL, porém, isso acaba dando um grande trabalho, afinal, para cada operação temos que fazer a conversão para o tipo Java a todo o momento. 

Será que existe alguma maneira de diminuir a quantidade de código para trabalhar com o Cassandra? A resposta é sim, no Cassandra também há ferramentas que atuam para converter as entidades em códigos nativos: os Mappers.


## Utilizando o Mapper

Para o nosso primeiro contato, será utilizado o Mapper que é mantido pela mesma empresa que mantém o Driver. De fato, ele é uma camada acima da camada do Driver:


```xml
<dependency>
    <groupId>com.datastax.cassandra</groupId>
    <artifactId>cassandra-driver-mapping</artifactId>
    <version>3.6.0</version>
</dependency>
```

Para o mesmo exemplo que realiza a manipulação da família de coluna `Book`, o primeiro passo é o mapeamento, que nada mais é que uma entidade cujos atributos são anotados. Ao ver o código, é possível perceber que as anotações são bem similares para quem veio do mundo do JPA:

* A anotação `Table` é para indicar que a classe é uma família de coluna.
* `Column` indica que o campo será mapeado para o banco de dados.
* `PartitionKey` indica que aquele atributo desempenha um papel especial dentro da família de coluna, que é a de chave primária.


```java
@Table(name = "book", keyspace = "library")
public class Book {

    @PartitionKey
    @Column
    private Long isbn;

    @Column
    private String name;

    @Column
    private String author;

    @Column
    private Set<String> categories;

//getter and setter
}
```

No primeiro contato com o Mapper, a redução de código é impressionante. O destaque para esse exemplo são as duas novas classes que aparecem: 
o `MappingManager` e o `Mapper`, que servem tanto para gerenciar as instâncias quanto para facilitar a comunicação entre o CQL e um objeto Java, respectivamente.

Agora, vamos ainda mais fundo, ainda precisamos realizar uma busca pela chave e também a criação de queries dinâmicas com o `PreparedStatement`.

```java
public class App {

    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "book";

    public static void main(String[] args) {
        try (Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build()) {

            Session session = cluster.connect();
            MappingManager manager = new MappingManager(session);
            Mapper<Book> mapper = manager.mapper(Book.class);


            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));

            mapper.save(cleanCode);
            mapper.save(cleanArchitecture);
            mapper.save(effectiveJava);
            mapper.save(nosql);

            Result<Book> books = mapper.map(session.execute(QueryBuilder.select().from(KEYSPACE, COLUMN_FAMILY)));
            for (Book book : books) {
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


    public static void main(String[] args) {
        try (Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build()) {

            Session session = cluster.connect();
            MappingManager manager = new MappingManager(session);
            Mapper<Book> mapper = manager.mapper(Book.class);

            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));

            mapper.save(cleanCode);
            mapper.save(cleanArchitecture);
            mapper.save(effectiveJava);
            mapper.save(nosql);


            Book book = mapper.get(1L);
            System.out.println("Book found: " + book);

            mapper.delete(book);

            System.out.println("Book found: " + mapper.get(1L));

            PreparedStatement prepare = session.prepare("select * from library.book where isbn = ?");
            BoundStatement statement = prepare.bind(2L);
            Result<Book> books = mapper.map(session.execute(statement));
            StreamSupport.stream(books.spliterator(), false).forEach(System.out::println);
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

O nosso código cria quatro tipos de livros e os persistem dentro do banco de dados. Em sequida, faz uma busca pela chave e, para finalizar, cria uma query com `PreparedStatement` e a executa sem se preocupar com a conversão para a nossa entidade `Book`. 

Mas não vamos parar por aí. Agora criaremos uma interação com os tipos `UDT` dentro da família de coluna `category`. 


```java
@Table(name = "category", keyspace = "library")
public class Category {

    @PartitionKey
    @Column
    private String name;

    @Frozen
    private Set<BookType> books;
 //getter and setter
}

@UDT(name = "book", keyspace = "library")
public class BookType {

    @Field
    private Long isbn;

    @Field
    private String name;

    @Field
    private String author;

    @Field
    private Set<String> categories;

//getter and setter

}
```

Modelagem criada e pronta para a interação no banco de dados com o tipo `UDT` que será representada pela classe `Category`. Essa classe é bem semelhante à entidade `Book`, que criamos anteriormente, com a diferença na forma como a informação será informada. Um ponto a salientar é que as anotações são bastante intuitivas.

```java
public class App3 {

    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "category";

    public static void main(String[] args) {
        try (Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build()) {

            Session session = cluster.connect();
            MappingManager manager = new MappingManager(session);
            Mapper<Category> mapper = manager.mapper(Category.class);

            BookType cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO"));
            BookType cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice"));
            BookType effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice"));
            BookType nosqlDistilled = getBook(4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));


            Category java = getCategory("Java", Sets.newHashSet(cleanCode, effectiveJava));
            Category oo = getCategory("OO", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture));
            Category goodPractice = getCategory("Good practice", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture, nosqlDistilled));
            Category nosql = getCategory("NoSQL", Sets.newHashSet(nosqlDistilled));

            mapper.save(java);
            mapper.save(oo);
            mapper.save(goodPractice);
            mapper.save(nosql);

            ResultSet resultSet = session.execute(QueryBuilder.select().from(KEYSPACE, COLUMN_FAMILY));
            Result<Category> categories = mapper.map(resultSet);
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

O código é bem semelhante à interação da classe `Book`, a única exceção é o uso da classe `Category.class` como parâmetro dentro do Mapper e, obviamente, serão criadas instâncias do tipo `Category` para a persistência da informação.


> No Mapper, existem métodos com o sufixo `Async` que realizarão operações de maneira assíncrona para o desenvolvedor. O livro não cobrirá todos os recursos do Mapper, então para mais informações consulte a documentação:
https://docs.datastax.com/en/developer/java-driver/3.6/manual/object_mapper/.


Impressionado com o poder do Mapper? Isso não é tudo! Também é possível criar interfaces assessoras que têm o objetivo de ler e escrever a partir do Cassandra sem muito código. Elas são compostas da anotação `Query`, que tem o CQL que será executado quando o método for chamado, como será exibido com uma interface que realizará operações dentro do Book. A cereja do bolo é definir essa interface como assessoras, para isso, é necessário adicionar a anotação `Accessor` nela.

<!--Essa parte "como será exibido com uma interface que realizará operações dentro do Book." não entendi direito. Quem será exibido? Esse "como" é no sentido de "já que" ou "da mesma forma que"? Tente reescrever, para ficar mais claro.-->

```java
@Accessor
public interface BookAccessor {

    @Query("SELECT * FROM library.book")
    Result<Book> getAll();


    @Query("SELECT * FROM library.book where isbn = ?")
    Book findById(long isbn);

    @Query("SELECT * FROM library.book where isbn = :isbn")
    Book findById2(@Param("isbn") long isbn);

}
```

Um ponto importante é que não precisamos nos preocupar com a implementação, tudo será gerenciado pelo Mapper automaticamente sem que o desenvolvedor se preocupe com isso. 

```java
public class App6 {

    public static void main(String[] args) throws Exception {
        try (Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build()) {

            Session session = cluster.connect();
            MappingManager manager = new MappingManager(session);
            BookAccessor bookAccessor = manager.createAccessor(BookAccessor.class);

            Result<Book> all = bookAccessor.getAll();
            StreamSupport.stream(all.spliterator(), false).forEach(System.out::println);

            Book book = bookAccessor.findById(1L);
            Book book2 = bookAccessor.findById(2L);
            System.out.println(book);
            System.out.println(book2);
        }

    }

}
```

### Conclusão

Neste capítulo se obteve o marco da integração entre o Cassandra e um aplicativo Java. Também foi possível ver a diferença de um framework que realiza a comunicação de baixo nível, semelhante ao JDBC, e um Mapper que, ao mesmo tempo que facilita o desenvolvimento, aumenta a responsabilidade para que o desenvolvedor não cometa erros ao esquecer que existe uma quebra de paradigma entre a aplicação e o banco de dados.
