# Criando um aplicativo com Java e Spring

O Spring framework é um projeto open source cujo objetivo é facilitar o desenvolvimento Java e hoje se tornou umas das ferramentas mais populares em todo o mundo. A sua história se colide muito com a do Java EE uma vez que ele nasceu para preencher a lacuna que o Java EE não conseguia, além de simplificar diversos pontos, como a parte de segurança. No seu começo, ele era apenas um framework de injeção de dependência, porém, atualmente ele tem diversos subprojetos, dentre os quais podemos citar:

* Spring Batch
* Spring Boot
* Spring security
* Spring LDAP
* Spring XD
* Spring Data
* E muito mais

Neste capítulo, será apresentado um pouco do mundo Spring integrado com o uso do Cassandra.

## Facilitando o acesso aos dados com Spring Data


O Spring Data é um dos vários projetos dentro do guarda-chuva do framework. Este subprojeto tem como maior objetivo facilitar a integração entre Java e os bancos de dados. Existem diversos bancos de dados que são suportados dentro do Spring. Dentre as suas maiores features estão:

* Uma grande abstração _object-mapping_.
* O _query by method_, que se baseia na query dinâmica realizada na interface.
* Fácil integração com outros projetos dentro do Spring, dentre eles, o Spring MVC e o JavaConfig.
* Suporte por auditoria.

Dentro do Spring Data, existe o Spring Data Cassandra, que tem suporte à criação de repositórios, a operações síncronas e assíncronas, a recursos como _query builders_ e uma altíssima abstração do Cassandra - a ponto de tornar dispensável aprender o Cassandra Query Language. Para adentrar no maravilhoso mundo do Spring, o primeiro passo é adicionar a dependência no projeto.


```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-cassandra</artifactId>
    <version>2.2.4.RELEASE</version>
</dependency>
```

Definidas as dependências, o próximo passo é o código de infraestrutura que ativa o Spring e a configuração de conexão com o Cassandra. A classe `Config` tem duas anotações: uma para procurar os componentes dentro de um pacote específico e outra para fazer algo similar aos repositórios Cassandra. Já a classe `CassandraConfig` dispõe as configurações para a conexão com o Cassandra, por exemplo, o keyspace a ser utilizado, configurações do cluster e o Mapper Cassandra Spring.

> O Mapper Cassandra Spring, como os Mappers em geral, tem a responsabilidade de fazer conversão de uma entidade de negócio em Java para o Cassandra ou vice-versa.


```java
@ComponentScan("com.nosqlxp.cassandra")
@EnableCassandraRepositories("com.nosqlxp.cassandra")
public class Config {
}


@Configuration
public class CassandraConfig extends AbstractCassandraConfiguration {

    @Override
    protected String getKeyspaceName() {
        return "library";
    }

    @Bean
    public CassandraClusterFactoryBean cluster() {
        CassandraClusterFactoryBean cluster =
                new CassandraClusterFactoryBean();
        cluster.setContactPoints("127.0.0.1");
        cluster.setPort(9042);
        return cluster;
    }

    @Bean
    public CassandraMappingContext cassandraMapping() {
        BasicCassandraMappingContext mappingContext = new BasicCassandraMappingContext();
        mappingContext.setUserTypeResolver(new SimpleUserTypeResolver(cluster().getObject(), getKeyspaceName()));
        return mappingContext;
    }


}
```

> Para obter mais informações sobre o framework Spring, um bom livro é o _Vire o jogo com Spring_ da Casa do Código: https://www.casadocodigo.com.br/products/livro-spring-framework

Código de configuração ou infraestrutura criado, trabalharemos no primeiro exemplo, que é a leitura e a escrita do livro. A modelagem acontece de maneira simples e intuitiva graças às anotações do Spring Data Cassandra:

* `Table` mapeia a entidade.
* `PrimaryKey` identifica a chave primária dentro da entidade.
* `Column` define os atributos que serão persistidos dentro do Cassandra.


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

    //getter and setter
}
```

Para a manipulação dos dados, existe a classe `CassandraTemplate`, que é um esqueleto para realizar uma operação entre o Cassandra e o objeto mapeado. Funciona de uma maneira bem análoga ao padrão Template Method que define o esqueleto para do algoritmo, porém, para operações no banco de dados com o Cassandra, mapeando as entidades para o banco e vice-versa. Um ponto importante do `CassandraTemplate` é que é possível realizar chamadas CQL, que ele se encarregará de converter para o objeto alvo - nesse caso, o `Book`.

```java
public class App {
    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "book";

    public static void main(String[] args) {

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Config.class)) {

            CassandraTemplate template = ctx.getBean(CassandraTemplate.class);

            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));

            template.insert(cleanCode);
            template.insert(cleanArchitecture);
            template.insert(effectiveJava);
            template.insert(nosql);

            List<Book> books = template.select(QueryBuilder.select().from(KEYSPACE, COLUMN_FAMILY), Book.class);
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

A classe `AnnotationConfigApplicationContext` levanta o contêiner Spring, varrendo as classes anotadas e definidas, em busca das injeções de dependência. Ela permite o uso de _try-resources_, ou seja, tão logo sai do bloco do `try`, a própria JVM se encarregará de chamar e método `close` e fechar o contêiner do Spring para o desenvolvedor.


Para o próximo passo, é possível perceber que não é realizado nenhum contato com o CQL em si, apenas com o template do Cassandra. No código a seguir utilizaremos o `CassandraTemplate` e realizaremos as operações de inserir, recuperar e remover. Apenas para lembrar, não utilizamos o `update`, uma vez que ele funciona como um alias para o `insert`.

```java
public class App2 {


    public static void main(String[] args) {

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Config.class)) {

            CassandraTemplate template = ctx.getBean(CassandraTemplate.class);

            Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO"));
            Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice"));
            Book effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice"));
            Book nosql = getBook(4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));

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

Para a última parte do desafio, que consiste na leitura das categorias do livro, as anotações são as mesmas utilizadas no caso do livro, com exceção do UDT `Book`, que possui as anotações `UserDefinedType` e `CassandraType`, definindo o nome do UDT e as informações para o campo, respectivamente.


```java
@Table
public class Category {

    @PrimaryKey
    private String name;

    @Column
    private Set<BookType> books;

   //getter and setter
}

@UserDefinedType("book")
public class BookType {

    @CassandraType(type = DataType.Name.BIGINT)
    private Long isbn;

    @CassandraType(type = DataType.Name.TEXT)
    private String name;

    @CassandraType(type = DataType.Name.TEXT)
    private String author;

    @CassandraType(type = DataType.Name.SET, typeArguments = DataType.Name.TEXT)
    private Set<String> categories;

    //getter and setter
}
```

Além das anotações do UDT, nada se difere dos dois primeiros caso com relação à consulta pela chave e a persistência do banco de dados. No código a seguir mostraremos uma interação entre as entidades e o tipo UTC. Um aspecto importante é o grande poder que tem o Cassandra e as possiblidades de modelagem sem realizar os famosos _joins_ no SQL. Faremos uma inserção da categoria, que tem um `Set` de tipo de livro, e o tipo de livro que é um UDT terá um `Set` de String.

```java
public class App3 {

    private static final String KEYSPACE = "library";
    private static final String COLUMN_FAMILY = "category";

    public static void main(String[] args) {

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Config.class)) {

            CassandraTemplate template = ctx.getBean(CassandraTemplate.class);

            BookType cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO"));
            BookType cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice"));
            BookType effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice"));
            BookType nosqlDistilled = getBook(4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));


            Category java = getCategory("Java", Sets.newHashSet(cleanCode, effectiveJava));
            Category oo = getCategory("OO", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture));
            Category goodPractice = getCategory("Good practice", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture, nosqlDistilled));
            Category nosql = getCategory("NoSQL", Sets.newHashSet(nosqlDistilled));

            template.insert(java);
            template.insert(oo);
            template.insert(goodPractice);
            template.insert(nosql);

            List<Category> categories = template.select(QueryBuilder.select().from(KEYSPACE, COLUMN_FAMILY), Category.class);
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

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Config.class)) {

            CassandraTemplate template = ctx.getBean(CassandraTemplate.class);

            BookType cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin", Sets.newHashSet("Java", "OO"));
            BookType cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin", Sets.newHashSet("Good practice"));
            BookType effectiveJava = getBook(3L, "Effective Java", "Joshua Bloch", Sets.newHashSet("Java", "Good practice"));
            BookType nosqlDistilled = getBook(4L, "Nosql Distilled", "Martin Fowler", Sets.newHashSet("NoSQL", "Good practice"));


            Category java = getCategory("Java", Sets.newHashSet(cleanCode, effectiveJava));
            Category oo = getCategory("OO", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture));
            Category goodPractice = getCategory("Good practice", Sets.newHashSet(cleanCode, effectiveJava, cleanArchitecture, nosqlDistilled));
            Category nosql = getCategory("NoSQL", Sets.newHashSet(nosqlDistilled));

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

Além da classe template, o Spring Data Cassandra conta com o conceito de **repositórios dinâmicos**, no qual o desenvolvedor cria uma interface e o Spring se responsabilizará da respectiva implementação. A nova interface herdará de `CassandraRepository`, que já possui um grande número de operações para o banco de dados.

Além disso, é possível utilizar o conceito de `query by method`, com o qual, ao utilizar as conversões de busca no nome do método, o Spring fará todo o trabalho pesado. Com esses repositórios, temos uma abstração valiosa que reduz o número de código, gerando uma altíssima produtividade.


```java
@Repository
public interface BookRepository extends CassandraRepository<Book, Long> {

    @Query("select * from book")
    List<Book> findAll();
}


public class App5 {

    public static void main(String[] args) {

        try (AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Config.class)) {

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

> A interface `CassandraRepository` é uma especialização do `CrudRepository` para operações do Cassandra. Já o `CrudRepository` é uma especialização do `Repository`. Essas interfaces fazem parte do Spring Data. Para mais informações: https://docs.spring.io/spring-data/data-commons/docs/2.1.x/reference/html/
>
> O Spring Data Cassandra tem muitos mais recursos, como operações assíncronas que facilitam e muito o dia a dia de quem desenvolve. Para saber mais: https://docs.spring.io/spring-data/cassandra/docs/2.1.2.RELEASE/reference/html/


### Conclusão

O Spring framework é um projeto que trouxe uma grande inovação para o mundo Java. Seus recursos e facilitações fazem com que ele tenha alta popularidade. Dentro da comunicação com o banco de dados, existe o Spring Data com diversas facilitações a ponto de não ser necessário aprender o Cassandra Query Language. Neste capítulo, tivemos uma introdução sobre Spring Data Cassandra e seus recursos. Sua facilidade e a integração com o contêiner do Spring faz com que o Spring Data seja uma ótima solução para aplicativos que já utilizam ou pretendem utilizar o Spring de alguma forma.
