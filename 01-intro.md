# Conceitos básicos de NoSQL

Os bancos NoSQL são uma classe de tecnologia de persistência que provê um novo mecanismo de armazenamento que vai de encontro com a normalização e os bancos de dados relacionais. Como qualquer outro banco, ele possui os mesmos objetivos: inserir, atualizar, recuperar e deletar informações, porém, com novos conceitos de modelagem e estrutura de armazenamento.  

O termo NoSQL, inicialmente, era relacionado com "não SQL" e posteriormente foi estendido para _Not Only SQL_, ou seja, "não apenas SQL", abrindo o conceito de consciência poliglota (o trabalho de lidar com diversos tipos de bancos para alcançar os objetivos na aplicação). 

Esses bancos têm como principais características velocidade e a alta taxa de escalabilidade, como facilidade de aumentar a quantidade de servidores de banco de dados. Isso impede o gargalo de operações, evita um ponto de falha, além de distribuí-los geograficamente, fazendo com que os dados estejam próximos dos usuários que farão a requisição. 

Bancos NoSQL estão sendo adotados com maior frequência em diversos tipos de aplicações, inclusive para as instituições financeiras. Como consequência, está crescendo também o número de fornecedores para esse tipo de banco de dados.


Atualmente, os bancos de dados NoSQL são classificados em quatro grupos (chave valor, família de coluna, documento e grafos) definidos pelo seu modelo de armazenamento:

> Para conhecer um novo conceito, é muito comum realizar uma comparação com o conhecimento já existente. Dessa maneira, considerando que os bancos de dados relacionais são bem famosos, faremos uma comparação entre eles e NoSQL ao decorrer deste capítulo.


## Chave-valor

<!--Nesta imagem, é Beaty mesmo, ou serua Beauty?-->
![Estrutura de chave-valor {w=70%}](imagens/key-value.png)

Os bancos do tipo chave-valor possuem uma estrutura similar ao `java.util.Map`, ou seja, a informação será recuperada apenas pela chave. 

Esse tipo de banco de dados pode ser utilizado, por exemplo, para gerenciar a sessão do usuário. Outro caso interessante é o DNS, cuja chave é o endereço, por exemplo `www.google.com`, e o valor é o IP desse servidor.

Atualmente existem diversas implementações de banco de dados do tipo chave-valor, dentre os quais os mais famosos são:

* AmazonDynamo
* AmazonS3
* Redis
* Scalaris
* Voldemort

Comparando o banco de dados relacional com o do tipo chave-valor, é possível perceber alguns pontos. Um deles é que a estrutura do chave-valor é bastante simples. 

Não é possível realizar operações como `join` entre os `buckets` e o valor é composto por grande bloco de informação em vez de ser subdivido em colunas como na base de dados relacional.

| Estrutura relacional | Estrutura chave-valor|
| --- | ---|
| Table | Bucket|
| Row | Key/value pair|
| Column | ----|
| Relationship | ----|


```


```

### Família de colunas

![Estrutura família de colunas {w=70%}](imagens/column.png)

Esse modelo se tornou popular através do _paper BigTable_ do Google, com o objetivo de montar um sistema de armazenamento de dados distribuído, e projetado para ter um alto grau de escalabilidade e de volume de dados. Assim como o chave-valor, para realizar uma busca ou recuperar alguma informação dentro do banco de dados é necessário utilizar o campo que funciona como um identificador único que seria semelhante à chave na estrutura chave-valor. Porém, as semelhanças terminam por aí. As informações são agrupadas em colunas: uma unidade da informação que é composta pelo nome e a informação em si.

Esses tipos de bancos de dados são importantes quando se lidam com um alto grau de volume de dados, de modo que seja necessário distribuir as informações entre diversos servidores. Mas vale salientar que a sua operação de leitura é bastante limitada, semelhante ao chave-valor, pois a busca da informação é definida a partir de um campo único ou uma chave. Existem diversos bancos de dados que utilizam essas estruturas, por exemplo:

* Hbase
* Cassandra
* Scylla
* Clouddata
* SimpleDb
* DynamoDB

Dentre os tipos de bancos de dados do tipo família de coluna, o Apache Cassandra é o mais famoso. Assim, caso uma aplicação necessite lidar com um grande volume de dados e com fácil escalabilidade, o Cassandra é certamente uma boa opção.


Ao contrapor o banco do tipo família de coluna com os bancos relacionais, é possível perceber que as operações, em geral, são muito mais rápidas. É mais simples trabalhar com grandes volumes de informações e servidores distribuídos em todo o mundo, porém, isso tem um custo: a leitura desse tipo de banco de dados é bem limitada. 

Por exemplo, não é possível realizar uniões entre família de colunas como no banco relacional. A família de coluna permite que se tenha um número ilimitado de coluna, que por sua vez é composta por nome e a informação, exatamente como mostra a tabela a seguir:

| Estrutura relacional | Estrutura de família de colunas|
| --- | ---|
| Table | Column Family|
| Row | Column|
| Column | nome e valor da Column|
| Relacionamento | Não tem suporte|


## Orientado a documentos

![Estrutura de documentos {w=40%}](imagens/document.png)

Os bancos de dados orientados a documentos têm sua estrutura muito semelhante a um arquivo JSON ou XML. Eles são compostos por um grande número de campos, que são criados em tempo de execução, gerando uma grande flexibilidade, tanto para a leitura como para escrita da informação. 

Eles permitem que seja realizada a leitura da informação por campos que não sejam a chave. Algumas implementações, por exemplo, têm uma altíssima integração com motores de busca. Assim, esse tipo de banco de dados é crucial quando se realiza análise de dados ou logs de um sistema. Existem algumas implementações dos bancos de dados do tipo documento, sendo que o mais famoso é o MongoDB.

* AmazonSimpleDb
* ApacheCouchdb
* MongoDb
* Riak


Ao comparar com uma base relacional, apesar de ser possível realizar uma busca por campos que não sejam o identificador único, os bancos do tipo documentos não têm suporte a relacionamento. Outro ponto é que os bancos do tipo documento, no geral, são _schemeless_.

| Estrutura relacional | Estrutura de documentos|
| --- | ---|
| Table | Collection|
| Row | Document|
| Column | Key/value pair|
| Relationship | --|


## Grafos

![Estrutura de grafos {w=80%}](imagens/graph.png)

Os bancos do tipo grafos são uma estrutura de dados que conecta um conjunto de vértices através de um conjunto de arestas. Os bancos modernos dessa categoria suportam estruturas de grafo multirrelacionais, onde existem diferentes tipos de vértices (representando pessoas, lugares, itens) e diferentes tipos de arestas. Os sistemas de recomendação que acontecem em redes sociais são o maior case para o banco do tipo grafo. Dos tipos de banco de dados mais famosos no mundo NoSQL, o grafo possui uma estrutura distinta com o relacional. 


* Neo4j
* InfoGrid
* Sones
* HyperGraphDB


## Teorema do CAP

![Teorema do CAP {w=60%}](imagens/cap.png)

Um dos grandes desafios dos bancos de dados NoSQL é que eles lidam com a persistência distribuída, ou seja, as informações ficam localizadas em mais de um servidor. Foram criados diversos estudos para ajudar nesse desafio de persistência distribuída, sendo o mais famoso uma teoria criada em 1999, o Teorema do CAP. 

Este teorema afirma que é impossível que o armazenamento de dados distribuído forneça simultaneamente mais de duas das três garantias seguintes:

* *Consistência/Consistency:*: uma garantia de que cada nó em um cluster distribuído retorna a mesma gravação mais recente e bem-sucedida. Consistência refere-se a cada cliente com a mesma visão dos dados.
* *Disponibilidade/Availability*: cada pedido recebe uma resposta (sem erro) - sem garantia de que contém a escrita mais recente.
* *Tolerância à partição/Partition tolerance*: o sistema continua a funcionar e a manter suas garantias de consistência apesar das partições de rede. Os sistemas distribuídos que garantem a tolerância continuam operando mesmo que aconteça alguma falha em um dos nós uma vez que existe, pelo menos, um nó para operar o mesmo trabalho e garantir o funcionamento do sistema.

De uma maneira geral, esse teorema explica que não existe mundo perfeito. Quando se escolhe uma característica, perde-se em outra como consequência. Em um mundo ideal, um banco de dados distribuído conseguiria suportar as três características, porém, na realidade, é importante para o desenvolvedor saber o que ele perderá quando escolher entre um ou outro.

Por exemplo, o Apache Cassandra é AP, ou seja, sua arquitetura focará em tolerância a falha e disponibilidade. Existirão perdas na consistência, assim, em alguns momentos um nó retornará informação desatualizada.
 
Porém, o Cassandra tem o recurso de nível de consistência, de modo que é possível fazer com que algumas requisições ao banco de dados sejam enviadas a todos os nós ao mesmo tempo, garantindo consistência. Vale ressaltar que fazendo isso ele perderá o `A`, de _availability_ do teorema do CAP.

### Conclusão

Este capítulo teve como objetivo dar o pontapé inicial para os bancos de dados não relacionais. Foram discutidos conceitos, os tipos de bancos que existem até o momento e suas estruturas. Com esse novo paradigma de persistência, vêm novas possibilidades e novos desafios para as aplicações. 

Esse tipo de banco de dados veio para enfrentar a nova era das aplicações, na qual velocidade ou o menor tempo de resposta possível são um grande diferencial. Com este capítulo introdutório, você está apto para seguir desbravando os bancos não relacionais, com o Cassandra.
