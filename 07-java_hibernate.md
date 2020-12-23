# Criando um aplicativo com Hibernate

No mundo relacional existem diversos frameworks ORM que facilitam a integração entre a aplicação e o objeto relacional, dentre eles, o mais famoso no mundo Java é o Hibernate. Trata-se de um ORM Java open source criado por Gavin King e, atualmente, é desenvolvido pela RedHat. 

No mundo Java, existe um processo de padronização, que é a JSR, Java Speficication Request, regido pelo JCP, Java Community Process (no momento que eu escrevo este capítulo existe um processo de migração do processo do JCP para Eclipse Foundation com um novo nome, que será discutido no capítulo sobre Jakarta). As especificações garantem diversos benefícios, dentre eles diversas empresas, instituições acadêmicas e membros da comunidade contribuindo nos projetos, totalmente orientado à comunidade. E o principal: o bloqueio do _vendor lock-in_, assim, o desenvolvedor não tem o risco de o projeto ser descontinuado por uma empresa, uma vez que poderá ser substituída por outras. A especificação entre o mundo Java e o banco relacional é o JPA, que faz parte da plataforma Java EE, agora Jakarta EE.

Assim como o Spring, o Hibernate possui diversos subprojetos para facilitar o mundo de desenvolvimento mais focado no mundo de persistência de dados, por exemplo, Hibernate Search, Hibernate Validator, dentre outras coisas. Neste capítulo, falemos um pouco mais sobre o Hibernate e sua relação com o mundo não relacional com o Hibernate OGM.

## O que é Hibernate OGM?

Com o objetivo de facilitar a integração e diminuir a curva de aprendizagem para aprender bancos de dados não relacionais, nasceu o Hibernate OGM. Sua abordagem é bastante simples: utilizar a API JPA, que os desenvolvedores Java já conhecem, com bancos de dados NoSQL. Atualmente Hibernate OGM tem suporte para diversos projetos:

* Cassandra
* CouchDB
* EhCache
* Apache Ignite
* Redis

### Exemplo prático com Hibernate OGM

Com uma breve introdução da história do Hibernate e da importância desse projeto no mundo Java, seguiremos para a parte prática. Sua configuração é definida em duas partes: 

1. A inserção de dependências, a partir do Maven.
2. A adição das configurações do banco de dados a partir do arquivo `persistence.xml` dentro da pasta `META-INF`.


```xml
<dependency>
    <groupId>org.hibernate.ogm</groupId>
    <artifactId>hibernate-ogm-cassandra</artifactId>
    <version>5.1.0.Final</version>
</dependency>
<dependency>
    <groupId>org.jboss.logging</groupId>
    <artifactId>jboss-logging</artifactId>
    <version>3.3.0.Final</version>
</dependency>
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-search-orm</artifactId>
    <version>5.6.1.Final</version>
</dependency>
```

A configuração entre o banco de dados e o framework Java é realizada a partir do arquivo `persistence.xml`, seguindo a linha da configuração do JPA, ou seja, não existe nenhuma novidade para quem já conhece essa especificação.

* `hibernate.ogm.datastore.provider`: define qual implementação o Hibernate utilizará, nesse caso, a implementação que utiliza o Cassandra.
* `hibernate.ogm.datastore.database`: para o Cassandra é o `keyspace`, ou seja, para esse exemplo será o `library`.
* `hibernate.search.default.directory_provider` e `hibernate.search.default.indexBase`: uma das dependências existentes no Hibernate OGM Cassandra é o Hiberante Search, que é a parte do Hibernate que oferece o `full-search` para os objetos gerenciados pelo Hibernate. É possível realizar buscas utilizando o Lucene ou Elasticsearch.


```xml
<?xml version="1.0" encoding="utf-8"?>
<persistence xmlns="http://java.sun.com/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://java.sun.com/xml/ns/persistence http://java.sun.com/xml/ns/persistence/persistence_2_0.xsd"
             version="2.0">

    <persistence-unit name="hibernate">
        <provider>org.hibernate.ogm.jpa.HibernateOgmPersistence</provider>
        <class>com.nosqlxp.cassandra.Book</class>
        <properties>
            <property name="hibernate.ogm.datastore.provider" value="org.hibernate.ogm.datastore.cassandra.impl.CassandraDatastoreProvider"/>
            <property name="hibernate.ogm.datastore.host" value="localhost:9042"/>
            <property name="hibernate.ogm.datastore.create_database" value="true"/>
            <property name="hibernate.ogm.datastore.database" value="library"/>
            <property name="hibernate.search.default.directory_provider" value="filesystem"/>
            <property name="hibernate.search.default.indexBase" value="/tmp/lucene/data"/>
        </properties>
    </persistence-unit>
</persistence>
```

Com a parte de infraestrutura pronta dentro do projeto, o próximo passo é a modelagem. Toda anotação do modelo é feita utilizando as anotações do JPA, o que facilita para quem já conhece a API de persistência relacional dentro do mundo Java.


```java
@Entity(name = "book")
@Indexed
@Analyzer(impl = StandardAnalyzer.class)
public class Book {

    @Id
    @DocumentId
    private Long isbn;

    @Column
    @Field(analyze = Analyze.NO)
    private String name;

    @Column
    @Field
    private String author;

    @Column
    @Field
    private String category;

   //getter and setter
}

```

Para atender ao requisito de buscar tanto pela chave quanto pela categoria, utilizaremos o recurso de motor de busca com o Apache Lucene através do Hibernate Search. Esse recurso é realmente muito interessante, principalmente, para permitir a busca de campos que não sejam a chave. Porém, isso incrementa a dificuldade do projeto, uma vez que existirá o desafio para sincronizar as informações dentro do banco de dados e do motor de busca.


```java
public class App {


    public static void main(String[] args) {
        EntityManagerFactory managerFactory = Persistence.createEntityManagerFactory("hibernate");
        EntityManager manager = managerFactory.createEntityManager();
        manager.getTransaction().begin();

        Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin");
        Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin");
        Book agile = getBook(3L, "Agile Principles, Patterns, and Practices in C#", "Robert Cecil Martin");
        Book effectiveJava = getBook(4L, "Effective Java", "Joshua Bloch");
        Book javaConcurrency = getBook(5L, "Java Concurrency", "Robert Cecil Martin");

        manager.merge(cleanCode);
        manager.merge(cleanArchitecture);
        manager.merge(agile);
        manager.merge(effectiveJava);
        manager.merge(javaConcurrency);
        manager.getTransaction().commit();

        Book book = manager.find(Book.class, 1L);
        System.out.println("book: " + book);
        managerFactory.close();
    }


    private static Book getBook(long isbn, String name, String author) {
        Book book = new Book();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        return book;
    }

}
```

Como esperado, toda operação acontece pelo `EntityManager`. A informação só vai definitivamente para o banco de dados quando se realiza o commit da transação. Como? O ponto é que, mesmo o Cassandra não tendo transação de maneira nativa no banco de dados, o Hibernate acaba simulando esse comportamento, que pode ser perigoso, principalmente, em ambientes distribuídos. 

O recurso de JPQL, Java Persistence Query Language, é uma query de consulta criada para o JPA e também está disponível dentro do Hibernate OGM, tudo isso graças ao Hibernate Search, que permite a busca por campos além da partition key. Existe a contrapartida: esse campo não poderá será analisado dentro da busca, ou seja, dentro da anotação `Field` o atributo `analizy` precisará ser definido como `Analyze.NO` (verifique como o campo `name` foi anotado dentro da classe).


```java
public class App2 {


    public static void main(String[] args) {
        EntityManagerFactory managerFactory = Persistence.createEntityManagerFactory("hibernate");
        EntityManager manager = managerFactory.createEntityManager();
        manager.getTransaction().begin();
        String name = "Clean Code";
        Book cleanCode = getBook(1L, "Clean Code", "Robert Cecil Martin");
        Book cleanArchitecture = getBook(2L, "Clean Architecture", "Robert Cecil Martin");
        Book agile = getBook(3L, "Agile Principles, Patterns, and Practices in C#", "Robert Cecil Martin");
        Book effectiveJava = getBook(4L, "Effective Java", "Joshua Bloch");
        Book javaConcurrency = getBook(5L, "Java Concurrency", "Robert Cecil Martin");

        manager.merge(cleanCode);
        manager.merge(cleanArchitecture);
        manager.merge(agile);
        manager.merge(effectiveJava);
        manager.merge(javaConcurrency);
        manager.getTransaction().commit();

        Query query = manager.createQuery("select b from book b where name = :name");
        query.setParameter("name", name);
        List<Book> books = query.getResultList();
        System.out.println("books: " + books);
        managerFactory.close();
    }


    private static Book getBook(long isbn, String name, String author) {
        Book book = new Book();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        return book;
    }

}
```

Com o motor de busca ativado no projeto, é possível realizar pesquisas utilizando todos os recursos do Lucene. Por exemplo, realizar busca do tipo `term`, que procura uma palavra dentro do texto, além de definir analisadores, tokens etc. Como mostra o código a seguir, o `FullTextEntityManager` é uma especialização do `EntityManager` com recursos do motor de busca. Isso faz com que a busca seja possível para campos que não sejam ID, além de oferecer recursos bem poderosos.

```java

public class App3 {


    public static void main(String[] args) {
        EntityManagerFactory managerFactory = Persistence.createEntityManagerFactory("hibernate");
        EntityManager manager = managerFactory.createEntityManager();
        manager.getTransaction().begin();
        manager.merge(getBook(1L, "Clean Code", "Robert Cecil Martin"));
        manager.merge(getBook(2L, "Clean Architecture", "Robert Cecil Martin"));
        manager.merge(getBook(3L, "Agile Principles, Patterns, and Practices in C#", "Robert Cecil Martin"));
        manager.merge(getBook(4L, "Effective Java", "Joshua Bloch"));
        manager.merge(getBook(5L, "Java Concurrency", "Robert Cecil Martin"));
        manager.getTransaction().commit();
        FullTextEntityManager fullTextEntityManager = Search.getFullTextEntityManager(manager);

        QueryBuilder qb = fullTextEntityManager.getSearchFactory()
                .buildQueryBuilder().forEntity(Book.class).get();
        org.apache.lucene.search.Query query = qb
                .keyword()
                .onFields("name", "author")
                .matching("Robert")
                .createQuery();

        Query persistenceQuery =  fullTextEntityManager.createFullTextQuery(query, Book.class);
        List<Book> result = persistenceQuery.getResultList();
        System.out.println(result);

        manager.close();
    }


    private static Book getBook(long isbn, String name, String author) {
        Book book = new Book();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        return book;
    }

}
```

Ainda resta um desafio: uma vez que não conseguimos representar um Set de UDT dentro do JPA, como faremos a busca pela categoria? A resposta surge utilizando os recursos do Hibernate Search. O que faremos é adicionar um novo campo, o `category`. Esse campo será uma `String` e conterá as categorias separadas por vírgula. Depois, todo o trabalho será realizado pelo motor de busca. Com isso, será necessário realizar uma mudança dentro da família de coluna para adicionar o novo campo.

```sql
ALTER COLUMNFAMILY library.book ADD category text;
```

Com o campo criado, basta alimentar o campo, que o Hibernate Search integrado com o OGM Cassandra se encarregará de fazer o trabalho pesado da indexação do campo e tratá-lo em conjunto com o Apache Lucene.


```java
public class App4 {


    public static void main(String[] args) {
        EntityManagerFactory managerFactory = Persistence.createEntityManagerFactory("hibernate");
        EntityManager manager = managerFactory.createEntityManager();
        manager.getTransaction().begin();
        manager.merge(getBook(1L, "Clean Code", "Robert Cecil Martin", "Java,OO"));
        manager.merge(getBook(2L, "Clean Architecture", "Robert Cecil Martin", "Good practice"));
        manager.merge(getBook(3L, "Agile Principles, Patterns, and Practices in C#", "Robert Cecil Martin", "Good practice"));
        manager.merge(getBook(4L, "Effective Java", "Joshua Bloch", "Java, Good practice"));
        manager.merge(getBook(5L, "Java Concurrency", "Robert Cecil Martin", "Java,OO"));
        manager.merge(getBook(6L, "Nosql Distilled", "Martin Fowler", "Java,OO"));
        manager.getTransaction().commit();

        FullTextEntityManager fullTextEntityManager = Search.getFullTextEntityManager(manager);

        QueryBuilder builder = fullTextEntityManager.getSearchFactory().buildQueryBuilder().forEntity(Book.class).get();
        org.apache.lucene.search.Query luceneQuery = builder.keyword().onFields("category").matching("Java").createQuery();

        Query query = fullTextEntityManager.createFullTextQuery(luceneQuery, Book.class);
        List<Book> result = query.getResultList();
        System.out.println(result);
        managerFactory.close();

    }


    private static Book getBook(long isbn, String name, String author, String category) {
        Book book = new Book();
        book.setIsbn(isbn);
        book.setName(name);
        book.setAuthor(author);
        book.setCategory(category);
        return book;
    }

}
```

### Conclusão

Utilizar uma API que o desenvolvedor já conhece para navegar em um novo paradigma no mundo da persistência é uma grande estratégia da qual o Hibernate tira o proveito. Neste capítulo vimos como utilizar a API JPA para trabalhar com o Apache Cassandra, o que diminui e muito a curva de aprendizagem para quem é experiente em Java. Porém, essa facilidade cobra o seu preço, uma vez que o JPA foi feito para o banco de dados relacional. Existem diversas lacunas que API tende a não cobrir no mundo NoSQL, por exemplo, operações de maneira assíncrona, assim como definições de nível de consistência existente no Cassandra. No próximo capítulo veremos também uma especificação do mundo Java, porém, uma API criada especificamente para a o mundo não relacional.
