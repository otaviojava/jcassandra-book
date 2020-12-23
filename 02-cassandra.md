# Cassandra


O Cassandra é um banco de dados NoSQL orientado à família de coluna que nasceu para resolver problemas com aplicações que precisam operar com gigantescas cargas de dados, além de poder escalar com grande facilidade. Ele nasceu no facebook e hoje vem sendo usado intensamente por empresas dos mais variados portes, tais como Netflix, Twitter, Instagram, HP, IBM, dentre muitas outras. 

Um fator importante que vale ser citado é a sua crescente adoção inclusive em mercados mais conservadores, em instituições financeiras e agências governamentais, como a NASA. Este capítulo tem como principal objetivo falar do corpo do Cassandra, suas características e seus conceitos.

## Definição

Focando um pouco mais nos aspectos técnicos do Cassandra, é possível destacar alguns pontos:

* **Tolerante a falhas**: os dados são replicados para vários nós de modo que, caso um nó caia, um outro estará pronto para substituí-lo sem _downtime_. Esse recurso é graças à sua característica _masterless_, que permite que todos os nós possuam o mesmo comportamento, ou seja, ao mesmo tempo que escreve também lê informações do banco de dados. Cada nó troca as informações diretamente entre si. Isso acontece em ciclos contínuos, e cada operação dessa é dada a partir de um aperto de mão, _handshake_, em pares. Portanto, caso um cluster possua quatro nós (para exemplificar, imagine que os nomes das máquinas sejam A, B, C e D), no primeiro ciclo o nó A compartilha informações com B, e C com D; no segundo ciclo, A com C, e D com B, e assim por diante. É a partir desse handshake que o nó descobrirá a localizações dos outros, além das informações do banco de dados em si. Esse protocolo de comunicação que o Cassandra utiliza é o Gossip.

* **Descentralizado**: evitar um calcanhar de Aquiles em uma aplicação é sempre um desafio, que se torna muito maior quando se trata de um banco de dados distribuído. Dados podem ser perdidos, a internet pode ser interrompida ou até mesmo um servidor de banco de dados pode ser destruído fisicamente, dentre outros problemas que podem acontecer - e é por isso que evitar um ponto de falha é algo realmente importante nas aplicações. Para o Cassandra, essa característica vem de modo nativo sem necessidade de realizar nenhuma grande modificação. 

* **Elástico**: um número de máquinas pode crescer ou diminuir de maneira linear sem nenhuma interrupção na aplicação. Desse modo, em dias com um alto grau de processamento, por exemplo, na semana do black friday, podem ser adicionados novos servidores e, com o fim desse período de pico, as informações podem retornar ao número de nós originais. 

## Hierarquia

Sendo o Cassandra um banco de dados NoSQL do tipo família de colunas, ele seguirá a mesma hierarquia explicada anteriormente para esse tipo de banco não relacional. Acima da família de coluna, existe um item, que é o _keyspace_, de modo que a hierarquia do Cassandra ficará da seguinte forma:

* Keyspace
* Column Family
* Column


![A estrutura dentro do Cassandra](imagens/hierarchy.png "A estrutura dentro do Cassandra")

### Keyspace

O keyspace é a estrutura que armazena ou contém uma ou mais família de colunas. Se pensarmos analogamente a uma tecnologia de dados de persistência relacional, seria semelhante a um banco de dados cuja estrutura armazena as tabelas. Outra responsabilidade igualmente importante que o keyspace possui é a definição do fator de réplica. Ele define o número de vezes que a mesma informação será copiada (replicada) entre os servidores. 

Por exemplo, dado um cluster com cinco servidores, e um keyspace com o fator de réplica de quatro, é possível afirmar que nesse cluster a mesma informação será copiada quatro vezes, ou seja, em praticamente todos os nós dentro do data center. Com a criação do fator de réplica vem a metodologia, a definição de como essa informação será copiada entre os nós, chamada de _estratégia de réplica_.

* **SimpleStrategy**: é a estratégia de réplica indicada para caso seja necessário utilizar apenas um único data center com todos os nós.

* **NetworkTopologyStrategy**: essa estratégia é altamente recomendada para os ambientes de produção e é utilizada quando é necessário utilizar mais de um data center. Isso é recomendado por diversas razões, por exemplo, imagine um desastre natural na região onde um dos data centers se encontram, em São Paulo. Isso não será um problema se o desenvolvedor tiver outros dois data centers na manga, ao redor do Brasil, como em Salvador e outro no Rio de Janeiro. Isso é algo muito semelhante que o Netflix faz com os seus dados com data centers ao redor do mundo.


### Column Family e Column

A família de coluna é um contêiner de linhas, sendo que cada linha é composta por uma ou mais colunas. Cada coluna é composta por nome, o valor e o timestamp, que será utilizado para verificar quais informações são as mais atualizadas.


As partes da coluna são:

* **Nome**: que representa o nome da coluna;
* **Valor**: a informação em si que se encontra dentro da coluna;
* **Timestamp**: tão logo seja criada ou alterada, uma coluna tem esse valor atualizado com a data da alteração. Ele serve para informar qual coluna é a mais recente. Por exemplo, durante a comunicação entre dois nós para compartilhar a informação, a partir do protocolo Gossip que informamos anteriormente será esse timestamp que dirá qual coluna é a mais quente, assim, será mantida nos dois bancos de dados.


![A estrutura da Column dentro do Cassandra {w=75%}](imagens/column_cassandra.png)

## Realizando uma operação dentro do Cassandra


No Cassandra, cada nó tem a responsabilidade tanto de escrita como de leitura. Quando é feita uma requisição para o banco de dados são realizados os seguintes passos:

1. O primeiro é a definição do nó que será responsável por gerenciar o pedido do cliente do banco de dados. Esse nó é conhecido como o coordenador.
2. O nó coordenador será o responsável por operacionar a query entre os demais clusters. Por exemplo, quando se busca uma informação a partir do seu respectivo identificador único, o ID, esse nó gerará valor numérico a partir do ID.
3. Esse valor numérico funcionará como um roteador, uma vez que cada nó é responsável por uma range de valores numéricos, assim, o nó coordenador mandará a requisição para o nó responsável por aquele número.

Uma informação importante é que esse range de valores pelos quais cada nó fica responsável é gerenciado automaticamente de acordo com o número de nós de Cassandra disponíveis.


![A partir das chaves serão gerados um hash, partitioner, com o qual é definido qual nó será responsável pela requisição. Cada nó recebe um range de maneira automática a partir do conceito do Vnode. {w=100%}](imagens/coordinator.png)


O particionador tem a responsabilidade de definir como os dados serão distribuídos ao redor dos nós dentro de um data center. De uma maneira geral, ele gerará um valor numérico a partir do ID. Essa configuração é realizada de maneira global, ou seja, todos os clusters deverão utilizar o mesmo tipo de particionador.


![A figura exibe a inserção de três registros. Durante uma inserção, é possível perceber que cada registro tem um campo que o define como identificador único ou chave, de cor amarela (Jim, Carol e a Suzy). A partir dessa chave, o partitioner será responsável por gerar um valor numérico e, em seguida, dizer qual nó é o responsável pela aquela informação. No nosso exemplo, a informação com a chave Jim gerou o valor numérico um e irá para o servidor A.](imagens/partitioner.png)


## Consistência _versus_ disponibilidade

Para cada requisição, é possível configurar o nível de consistência com o qual uma operação será realizada. 

Esse nível define quantos nós precisam responder ao coordenador para indicar que o processo, de escrita ou leitura, foi realizado com sucesso. Essa configuração é um ponto-chave entre o dilema da consistência e a disponibilidade.

É importante salientar que, quanto maior o número de nós utilizado para uma operação, mais possível é garantir um alto grau de consistência.

![Equilíbrio entre consistência e disponibilidade dentro de uma requisição do Cassandra {w=50%}](imagens/consistency_vs_disponibility.png)

## Dentro de um nó Cassandra

Uma vez discutido como o Cassandra funciona em cluster, também é preciso falar como ele funciona internamente e de suas partes. 

Tão logo o Cassandra recebe uma operação de escrita, ele armazena a informação na memória em uma estrutura chamada de `memtable` e também utiliza uma estrutura no disco, chamada `commit log`. Esse commit log recebe cada escrita feita pelo Cassandra e ela permanece mesmo quando o nó está desligado. 

Assim, quando se inicia a escrita, a sequência dentro de um nó Cassandra é:


* Realizar o `logging` dentro do commit log
* Escrever a mesma informação na memória, `memtable`
* Realizar a operação de `flush` a partir do `memtable`
* Armazenar as informações de maneira ordenada dentro do disco com o `SSTables`

Tanto o `memtables` quanto o `SSTable` são armazenados por tabela, de forma organizada e otimizadas para a leitura e escrita, sendo que o `SSTable` tem suas informações dentro do disco.


![A sequência de escrita que acontece dentro de um nó do Cassandra. {w=100%}](imagens/write_sequence.png)

### Operações de escrita


A tabela a seguir mostra o nível de consistência de escrita do mais forte para o mais fraco, sendo que uma escrita mais forte significa maior consistência, ou seja, escrever em mais nós dentro dos datas centers implica na disponibilidade. 

Vale salientar que a operação de escrita em cada nó significa a escrita tanto no `commit log` quanto no `memtable`.

```

```

| Nível de consistência | Descrição                                                    |
| --------------------- | ------------------------------------------------------------ |
| `ALL`                 | A operação de escrita em todos os nós da réplica dentro do cluster |
| `EACH_QUORUM`         | A operação de escrita em quorum de réplicas em cada data center |
| `QUORUM`              | Similar ao anterior, escrita em quorum através de todos os data centers |
| `LOCAL_QUORUM`        | A operação de escrita em quorum dentro do data center do nó coordenador |
| `ONE`                 | Deve acontecer a operação de escrita em, pelo menos, um nó   |
| `TWO`                 | Define a operação de escrita em, pelo menos, dois nós        |
| `THREE`               | Define a operação de escrita em, pelo menos, três nós        |
| `LOCAL_ONE`           | Deve realizar a operação de escrita em, pelo menos, um nó dentro do data center |
| `ANY`                 | Garante a operação de escrita em, pelo menos, um nó.         |

> QUORUM é mais da metade, isto é, a metade mais um. A fórmula é `(fator de réplica/2) + 1`

### Operações de leitura


A tabela a seguir mostra a informação de leitura do mais forte para o mais fraco levando em consideração a consistência. Um ponto importante é que em cada operação de leitura existe o conceito de _read repair_. 

O read repair melhora as consistências dos dados se baseando na simples estratégia de que, após uma requisição, o coordenador pegará as informações mais atualizadas e compartilhará entre todos os nós que participaram da operação de leitura.


Nessa operação, de uma maneira geral, o nó coordenador realiza um _request_ e a partir do nível de consistência define o número de nós que participarão da operação. Depois disso, as informações mais “quentes” são enviadas para o cliente, em seguida, é realizada a operação de _read repair_ de maneira assíncrona.

| Level          | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| `ALL`          | Retorna a resposta depois de todas as réplicas confirmarem o recebimento dos dados |
| `QUORUM`       | Retorna a resposta depois do quorum de todos os data centers confirmarem o recebimento dos dados |
| `LOCAL_QUORUM` | Retorna a resposta depois do quorum do data center do nó coordenador |
| `ONE`          | Retorna a resposta depois de ler do nó mais próximo          |
| `TWO`          | Retorna a resposta depois de ler dos dois nós mais próximos  |
| `THREE`        | Retorna a resposta depois de ler dos três nós mais próximos  |
| `LOCAL_ONE`    | Retorna a resposta depois de ler do nó mais próximo dentro do data center do nó coordenador |
| `SERIAL`       | Permite a leitura do estado atual (e possivelmente não comprometido) dos dados sem propor uma nova adição ou atualização. Se uma leitura `SERIAL` encontrar uma transação não confirmada em andamento, a transação será confirmada como parte da leitura. Semelhante ao `QUORUM`. |
| `LOCAL_SERIAL` | Semelhante ao `SERIAL`, porém, confinado ao data center semelhante ao `LOCAL_QUORUM`. |


### Conclusão

O Apache Cassandra é um banco com uma arquitetura muito intrigante e mostra a plena sintonia do seu funcionamento desde um único nó até o seu funcionamento em conjunto com dezenas ou milhares de nós dentro de diversos data centers ao redor do mundo. Sua forma de distribuir os dados na escrita entre os clusteres a partir da chave faz com que ele foque em disponibilidade e tolerância a falhas. Se pensássemos no teorema do CAP, o Cassandra teria um foco no AP. No próximo capítulo, sairemos um pouco da teoria e aprenderemos como instalar o Cassandra de várias formas.
