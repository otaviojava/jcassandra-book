# A nova era das aplicações vem com novos bancos de dados


O mundo dos negócios cresce de maneira exponencial com várias oportunidades e empresas concorrendo entre si. A disputa entre clientes é algo que não se define mais pela localização da empresa, mas pela usualidade e, sobretudo, o tempo de resposta. Existem diversos estudos que explicam que melhorias de milissegundos são o suficiente para adquirir novos clientes, fidelizar os já existentes, além de desbancar a concorrência. 

Nessa batalha pelos milissegundos, nasceram diversos paradigmas e frameworks, porém, toda essa melhoria seria inútil se a persistência da informação continuasse lenta. Com a necessidade de uma melhor performance, surgiram os bancos de dados NoSQL para facilitar o desempenho e a distribuição da informação. 

Junto a esse conceito, nasceu o Apache Cassandra, o banco de dados NoSQL elástico, tolerante a falhas e com um alto grau de performance, tendo cases de sucessos nas maiores empresas do mundo, como o Netflix, GitHub, eBay, dentre outros. O Cassandra é um banco de dados não relacional originado pelo Facebook, e hoje é um banco de dados NoSQL do tipo família de coluna open source dentro da Apache Foundation.

De uma maneira geral, o objetivo do livro é abordar o Cassandra, seus conceitos e sua aplicabilidade com o Java. Para cobrir todos esses conceitos, o livro foi desenhado no total de dez capítulos.

O escopo do inicial é a introdução dos bancos de dados não relacionais. Serão abordados os desafios dos bancos de dados distribuídos, suas limitações com o teorema do CAP, o que é NoSQL e seus tipos, além da comparação entre o NoSQL e o já consolidado banco de dados relacional.

Em seguida, abordaremos os conceitos do Cassandra, como a sua hierarquia, leitura e escrita, seu funcionamento no nó e sua orquestração dentro de um Cluster. Veremos sua instalação manual, Docker e Cluster com `docker-compose`.

A realização de consultas é muito comum dentro de um banco de dados. No mundo relacional o desenvolvedor está familiarizado com o SQL. O Cassandra tem a sua própria linguagem de consulta: o Cassandra Query Language ou CQL. Você vai aprender como criar keyspace, família de coluna, realizar as operações CRUD (criação, recuperação, atualização e deleção dos dados) que os desenvolvedores tanto amam.

Existem diversas similaridades entre a linguagem da comunicação do Cassandra, o CQL, e o SQL. Vale lembrar que conceitualmente o banco relacional e o Cassandra estão em diferentes espectros, levando em consideração o teorema do CAP. Realizar a modelagem do Cassandra da mesma maneira que se realiza dentro do banco relacional terá um alto impacto de performance.

Após todos os conceitos de comunicação do Cassandra e dicas de modelagem, o próximo passo é a integração com o Java. Dentro disso, o primeiro ponto a ser exibido é o Driver do DataStax, cuja API é similar ao driver JDBC. Um ponto importante é que uma vez o desenvolvedor tenha familiaridade com o CQL toda a comunicação acontece de maneira bastante fluida. 

No dia a dia, os negócios se comportam com orientação a objetos, de modo que também é importante que nos debrucemos sobre os impactos dos Mappers, frameworks cujo objetivo é realizar o mapeamento entre o Cassandra e os objetos de uma entidade de negócio, seja de maneira positiva, com a produtividade, ou de maneira negativa, com possíveis problemas de performance. Dentre esses Mappers utilizaremos o DataStax Mapper, Spring Data Cassandra, Hibernate OGM Cassandra e o Eclipse JNoSQL. E para facilitar a comparação entre eles será utilizado o mesmo exemplo que se baseia em uma solução para uma biblioteca.

Para acessar o código-fonte do livro: 

https://github.com/otaviojava/cassandra-java-code

## Público-alvo/pré-requisitos

Desenvolvedores Java, entusiastas e familiarizados com a tecnologia NoSQL e que desejam aprender ou aprofundar os conhecimentos no Cassandra.
