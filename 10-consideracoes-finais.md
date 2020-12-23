# Considerações finais

Pronto, após todos esses capítulos estamos chegando ao final do livro. O objetivo desta conclusão da série é mostrar as últimas dicas e recomendações para que você esteja livre para desbravar os mares do Cassandra e desenvolver suas próprias experiências.

## Motor de busca

Como já foi discutido no capítulo de modelagem, no mundo ideal todas as queries deverão ser realizadas a partir da partition key, ou seja, além da desnormalização, deve-se evitar a todo custo a utilização dos recursos como `ALLOW INDEXING` ou índice secundário, do contrário, o Cassandra terá impacto em performance. 

Essa duplicação de dados terá a vantagem de performance no momento da leitura, mas também terá o impacto do gerenciamento das informações. 

O motor de buscas contém diversas vantagens com relação ao Cassandra. A primeira delas é o fato de conseguir realizar uma busca em outros campos que não seja o partition key. O segundo ponto é que a precisão com motor de busca trará grandes benefícios para o produto também. 

Imagine que dentro do livro agora se tenha um resumo, porém, o texto está em HTML. Dentro de um motor de busca, a informação quando inserida é tratada para que, quando a busca for realizada, ela seja otimizada. Por exemplo, levando em consideração o processo do Elasticsearch teremos os seguintes passos:


* **Character Filters**: é o primeiro contato da informação. Ele receberá um stream de caracteres e retornará um stream de caracteres. No nosso exemplo, o formato html será removido de forma que o texto ficará da seguinte forma: `Conheça o Clean Code, o livro de boas práticas de programação`.
* **Tokenizer**: o processo de tokenização em que, dado uma stream de caracteres, ele converte em um token, que geralmente resulta em uma palavra. Por exemplo, a descrição do livro se transformaria em tokens como `[“Conheça”,”o”, “Clean”,”Code”,”o”,”livro”,”de”,”boas”,”práticas”,”de”,”programação”]`.
* **Token Filter**: nessa etapa, dada uma sequência de tokens, ele será responsável por remover (remover palavras comuns, as _stop words_, como: “de”, “o”), adicionar (adicionar sinônimos: programação e software) ou modificar tokens (converter as palestras para minúsculo ou remover acentos das palavras).

De modo que, quando for pesquisar, por exemplo, pela palavra “programação” dentro do motor de busca a informação já estará pré-processada como, então não terá de varrer o texto em busca da informação em cada linha. Além do novo horizonte que o motor de busca permite, é possível adicionar sinônimos, adicionar pesos em campos de buscas (caso a palavra esteja no título, ele ter mais relevância do que quando estiver na descrição) e levar em consideração possíveis erros ortográficos que podem realizados pelo usuário de maneira acidental etc.

No mundo do motor de busca existem algumas opções:

* Apache Lucene: quando mencionamos o Hibernate Search foi mencionado um pouco sobre esse projeto. Ele é open source e se encontra dentro da Apache Foundation, sendo uma API de busca e de indexação de documentos. Dentre os grandes cases de experiência do projeto se encontra o Wikipedia.
* Apache Solr: ou simplesmente Solr é também um projeto open source mantido pela Apache Foudation. Dentre as principais features podemos citar pesquisa de texto, indexação em tempo de execução e a possiblidade de executar em Clusters. Uma curiosidade é que o Apache Solr é desenvolvido em cima do Apache Lucene.
* Elasticsearch: o Elasticsearch, assim como o Apache Solr, é um motor de busca baseado no Apache Lucene, porém, não é da Apache Foundation (apesar de a licença ser Apache 2.0). Também tem o recurso de trabalhar o motor de busca como cluster e atualmente é considerado o motor de busca mais popular e utilizado do mundo.

Existem alguns projetos, inclusive, que visam justamente à integração entre o motor de busca com o Cassandra são os casos do Solandra (integração do Cassandra com Solr) e o Elassandra (integração do Cassandra com Elasticsearch). 

## Validando os dados com Bean Validation

Em uma aplicação é muito comum que alguns campos tenham uma validação para garantir a consistência da aplicação. Por exemplo, os campos de e-mails. O Cassandra não possui nenhum recurso de validação, e para preencher essa lacuna uma opção é uso do Bean Validation, que permite colocar constraints em objetos Java a partir de anotações.

O Bean Validation possui um vasto número de anotações nativa que permitem um alto grau de validações, como tamanho mínimo ou tamanho máximo de um text, validação de e-mail, campo obrigatório, dentre outros.


```java
public class User {
@NotNull
@Email
private String email;
}
```

Além de ter várias opções de validação, também é possível criar um novo, de modo que o Bean Validation é uma grande opção caso os dados precisem ter algum tipo de validação para a consistência dos dados.

## Realizando testes de integração com o Cassandra


No mundo da engenharia de software, os testes são uma peça fundamental para a construção de qualquer programa. É a partir deles que é possível garantir que o código atual se encontra estável, que se teste de maneira rápida e sem a interferência humana, além de que tenha maior segurança para refatorar e adicionar novos recursos dentro do sistema. Os testes de integração são um dos tipos de testes de um sistema com os quais se verifica um grupo de componente, por exemplo: a comunicação entre o código Java e o Cassandra. Nesse tópico citarei dois bons exemplos para realizar os testes.

O primeiro deles é o Cassandra Unit, que se comporta de maneira similar ao DBUnit, porém, em vez de um banco relacional, utiliza o Cassandra. Ele levantará um Cassandra embarcado pronto para uso nos seus testes.

Uma grande vantagem de usar tecnologias de build é que nós não precisamos nos preocupar com as dependências. Por exemplo, é possível utilizar o Cassandra Unit integrado com Junit 5.

```xml
<dependency>
    <groupId>org.cassandraunit</groupId>
    <artifactId>cassandra-unit</artifactId>
    <version>3.3.0.2</version>
    <scope>test</scope>
</dependency>
```

É possível perceber que nesse pequeno teste só foi necessário adicionar um método que levanta uma instância embarcada do Cassandra e outra que limpa.

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
    public void test(){
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

`Testcontainers` é uma biblioteca Java que permite a integração do Junit com qualquer imagem docker. Assim, é possível executar o Cassandra através de uma imagem docker. A dependência é algo realmente simples de ser adicionado. Após isso, a diferença com o teste de integração anterior está na dependência e no código de infraestrutura para levantar o Cassandra, desta vez, via contêiner.

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.9.1</version>
    <scope>test</scope>
</dependency>
```

Com a dependência definida o código se mostra bem simples. A classe `GenericContainer` representa um contêiner docker. No seu construtor, há uma `String` que é o nome da imagem que o `testcontainer` buscará no dockerhub. Feito isso, o próximo passo foi expor a porta de conexão com o banco de dados, nesse caso a port `9072`. Como todo teste de integração, queremos utilizar o contêiner tão logo ele esteja disponível, e é por esse motivo que setamos a estratégia padrão que espera o novo contêiner executar a próxima linha apenas quando o banco de dados instanciado esteja de pé. Com o banco de dados pronto, o próximo passo é a realizar o teste necessário.

```java
public class SampleCassandraContainerTest {


    @Test
    public void test(){

        GenericContainer cassandra =
                new GenericContainer("cassandra")
                        .withExposedPorts(9042)
                        .waitingFor(Wait.defaultWaitStrategy());

        cassandra.start();
        Cluster.Builder builder = Cluster.builder();
        builder.addContactPoint(cassandra.getIpAddress()).withPort(cassandra.getFirstMappedPort());
        Cluster cluster = builder.build();
        Session session = cluster.newSession();
        ResultSet resultSet = session.execute("SELECT * FROM system_schema.keyspaces;");
        Assertions.assertNotNull(resultSet);
        cluster.close();
    }
}
```

## Experimentando outros sabores de Cassandra

Um ponto importante é que existem bancos de dados que competem com o Cassandra seja no mesmo tipo de banco de dados seja para resolver, praticamente, os mesmos problemas. É muito importante que o leitor também conheça esses bancos de dados:

* O DataStax DSE: é uma versão corporativa, fechada e paga fornecida pela DataStax. Ele suporta tudo aquilo que o Cassandra faz atualmente e adiciona novos recursos como analytics e motor de busca já integrado. Um outro ponto interessante é que o DSE é um banco de dados _multi-model_, em outras palavras, é um banco de dados NoSQL que suporta mais de um tipo de banco NoSQL (chave-valor, família de coluna e grafos).
* ScillaDB: é um banco de dados cujo foco é a possibilidade de manter total compatibilidade com o Cassandra, porém, com uma performance extremamente superior. Em teoria, é possível pegar uma aplicação em Cassandra e mudar para esse banco de dados sem impacto algum. Ele também tem imagens oficiais dentro do dockerhub. Uma vez que o leitor entrou no mundo Cassandra vale a tentativa.

### Conclusão

Com isso, foram apresentados os conselhos finais e últimas dicas para prosseguir com o Cassandra. Recursos como motor de busca, validação de dados com bean validation e ferramentas para testes de integração são recursos valiosos nos quais é recomendável que você vá muito mais fundo. O objetivo deste capítulo foi de apenas despertar a curiosidade para continuar evoluindo o Cassandra, integrando com outras ferramentas. Espero que este livro tenha contribuído para mostrar como o Cassandra é simples de uso e está a menos de um passo de distância para integrá-lo com aplicações Java.
