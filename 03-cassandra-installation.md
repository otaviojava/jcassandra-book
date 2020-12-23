# Instalação do Cassandra

Nos capítulos anteriores, foram abordados os conceitos dos bancos de dados não relacionais, suas comparações com o relacional, e dissecamos o funcionamento interno do Cassandra. 

Neste capítulo, teremos uma visão mais prática: mostraremos como funciona o processo de instalação, configuração, como criar instâncias do Cassandra dentro um contêiner com Docker, além de como criar um cluster de uma maneira simples com `docker-compose`.

## Realizando download do Cassandra

Antes de iniciar a instalação é importante falar dos pré-requisitos do Cassandra. Como ele é um banco de dados feito em Java, para a instalação é necessário que você tenha instalada alguma implementação do Java 8 (OpenJDK, Azul, Oracle HotSpot etc.).

Nesse primeiro momento de instalação não será utilizado nenhum outro cliente além do `cqlsh`, o comunicador nativo do Cassandra, assim, é necessário que o computador também tenha a última versão do Python 2.7. O `cqlsh` é uma ponte de comunicação com o banco de dados que permite a interação com uma instância via linhas Shell através do CQL, o Cassandra Query Language.


* Realize o download do Cassandra no site do projeto: http://cassandra.apache.org/download/
* Descompacte o arquivo, que terá nome semelhante a `tar -xvf apache-cassandra-3.6-bin.tar.gz`.
* Com os arquivos descompactados, o próximo passo é entrar na pasta `bin` e iniciar o Apache Cassandra. Para isso, execute o comando `cassandra -f`.


Uma vez com a instância de banco de dados de pé, o próximo passo será testar a comunicação com ela. Para isso, será utilizado o `client`, citado anteriormente. Assim, para o executar, é necessário rodar o comando `./cqlsh` dentro da pasta `bin`.

```bash
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> SHOW VERSION
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
cqlsh>
```

## Configurações dentro do arquivo yaml

Para executar um único nó, os valores padrões são suficientes, porém, para rodar em um cluster com mais de um nó, algumas mudanças são importantes. As principais configurações se encontram dentro da pasta `conf`, no arquivo `cassandra.yaml`. Destacam-se:

* `cluster_name`: o nome do cluster.
* `seeds`: os IPs, Internet Protocol (o endereço do computador), dos nós sementes separados por vírgulas.
* `listen_address`: os IPs dos nós sementes, ou as instâncias de Cassandra que serão utilizados como referência em uma startup, separados por vírgulas. Essas instâncias terão como responsabilidade “treinar” os novos servidores recém-chegados no cluster. Assim, serão esses nós que serão encarregados de enviar todas as informações necessárias para que o nó calouro consiga trabalhar dentro do cluster.
* `listen_interface`: informa para o Cassandra qual interface utilizar, consequentemente, qual endereço para o uso. É necessário modificar o `listen_address` ou essa configuração, porém, não os dois.


## Simplificando a instalação com contêineres

Uma maneira de se instalar o Cassandra é através de contêiner com o Docker. Em uma visão geral, um contêiner é um ambiente isolado. A tecnologia Docker utiliza o kernel linux e recursos, por exemplo Cgroups e namespaces, para segregar processos, de modo que eles possam ser executados de maneira independente. 

O objetivo dos contêineres é criar tal independência: a habilidade de executar diversos processos e aplicativos de maneira isolada, utilizar melhor a infraestrutura, manter a segurança entre os contêineres executados, além da facilidade de criação e manutenção.

> Se você tiver interesse em se aprofundar em Docker, acesse: https://www.casadocodigo.com.br/products/livro-docker

Uma vez instalado o Docker (https://docs.docker.com/install/), basta executar o seguinte comando no console:

```bash
docker run -d --name casandra-instance -p 9042:9042 cassandra
```

Esse comando baixa e executa https://store.docker.com/images/cassandra, a imagem oficial do Cassandra, diretamente do _docker hub_.

Para executar o `cqlsh` dentro do Docker:

* Liste os contêineres sendo executados na máquina com o comando `docker ps`.

```bash
$ docker ps
CONTAINER ID        IMAGE
7373093f921a        cassandra
```

* Cada contêiner criado possui um identificador único ou ID. O objetivo do comando anterior foi listar os contêineres existentes e seus respectivos IDs. Uma vez com o ID do contêiner do Cassandra encontrado, basta executar o comando para `docker exec -it CONTAINER_ID cqlsh`.


```bash
$ docker exec -it 7373093f921a cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4] Use HELP for help.
cqlsh> SHOW VERSION
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
```

Por padrão do Docker, todos os arquivos são criados dentro do contêiner. Assim, para extrair o volume de dados para fora do contêiner, é necessário mapear o caminho `/var/lib/cassandra`, por exemplo:

```bash
docker run --name some-cassandra -p 9042:9042 -v /my/own/datadir:/var/lib/cassandra -d cassandra
```

## Criando o primeiro cluster com Docker Compose

Seguindo a linha do Docker e contêiner, para que se execute um cluster é necessário ter muitos contêineres. Umas das ferramentas que permite a execução de múltiplos contêineres é o Docker Compose.

Isso é feito de uma maneira bastante simples com um arquivo YAML de configuração. Dessa forma, com um único comando é possível executar muitos contêineres. O arquivo a seguir mostra uma simples configuração utilizando três nós no cluster.

Toda a descrição de contêineres, configuração de cada um e como eles se inter-relacionam é feita a partir de um arquivo de extensão yml, que por convenção tem como nome `docker-compose.yml`. Esse arquivo contém a configuração de três nós Cassandra a partir de imagens Dockers.

```yaml

version:  '3.2'

services:

    db-01:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-01
          seeds: db-01,db-02,db-03
        volumes:
          - /home/otaviojava/Environment/nosql/db1:/var/lib/cassandra

    db-02:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-02
          seeds: db-01,db-02,db-03
        volumes:
          - /home/otaviojava/Environment/nosql/db2:/var/lib/cassandra

    db-03:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-03
          seeds: db-01,db-02,db-03
        volumes:
          - /home/otaviojava/Environment/nosql/db3:/var/lib/cassandra

networks:
    cassandranet:
```

Com o arquivo `docker-compose.yml` criado, os próximos passos são muito simples:

1. Para iniciar os contêineres: `docker-compose -f docker-compose.yml up -d`
2. Para parar e remover os contêineres: `docker-compose -f docker-compose.yml down`


> Para esse exemplo, estão sendo levantados três clusters. Caso queria rodar localmente verifique se você terá memória suficiente para isso.

Uma possibilidade é diminuir o consumo de memória dos clusters, por exemplo, iniciando três nós e fazendo com que cada um nó tenha no máximo 1 GB de memória. 

**Arquivo de configuração de cluster de Cassandra levantando cada nó com 1 gigabyte**

```bash
version:  '3.2'

services:

    db-01:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-01
          seeds: db-01,db-02,db-03
          JVM_OPTS: -Xms1G -Xmx1G
        volumes:
          - /home/otaviojava/Environment/nosql/db1:/var/lib/cassandra

    db-02:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-02
          seeds: db-01,db-02,db-03
          JVM_OPTS: -Xms1G -Xmx1G
        volumes:
          - /home/otaviojava/Environment/nosql/db2:/var/lib/cassandra

    db-03:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-03
          seeds: db-01,db-02,db-03
          JVM_OPTS: -Xms1G -Xmx1G
        volumes:
          - /home/otaviojava/Environment/nosql/db3:/var/lib/cassandra

networks:
    cassandranet:
```

### Conclusão

A instalação e a configuração do Cassandra, seja em cluster utilizando contêiner como Docker ou não, mostraram-se algo realmente muito simples se compararmos a uma configuração de cluster semelhante dentro de um banco de dados relacional. 

Um ponto importante é que a popularidade do Docker não é em vão: a sua facilidade de execução e de configuração para um nó ou clusters é realmente muito interessante principalmente para desenvolvedores. Talvez, esse seja o motivo por que atualmente o Docker é considerado a maior ferramenta quando o assunto é DevOps. 

Nas próximas cenas, será discutido como realizar a comunicação com o Cassandra, algo que será extremamente simples caso você já esteja acostumado com os bancos relacionais.
