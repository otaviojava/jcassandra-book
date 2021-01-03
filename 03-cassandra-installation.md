# Cassandra installation

In the previous chapters, we discussed the concepts of non-relational databases, their comparisons with the relational ones, and we dissected the internal workings of Cassandra.

In this chapter, we will have a more practical view: we will show the installation process, configuration, how to create Cassandra instances inside a container with Docker, as well as how to create a cluster simply with docker-compose.

## Downloading Cassandra

Before starting the installation, it is important to talk about Cassandra's prerequisites. As it is a database made in Java, for installation, you must have installed a Java 8 implementation (OpenJDK, Azul, Oracle HotSpot, etc.).

In this first installation moment, no other client will be used besides `cqlsh`, Cassandra's native communicator, so it is necessary that the computer also has the latest version of Python 2.7. `cqlsh` is a communication bridge with the database that allows interaction with an instance via Shell lines through CQL, the Cassandra Query Language.


* Download Cassandra from the project website: http://cassandra.apache.org/download/


  * Unzip the file, which will have a name similar to `tar -xvf apache-cassandra-3.11.9-bin.tar.gz`

* With the files unzipped, the next step is to enter the `bin` folder and start Apache Cassandra. To do this, run the command `sh cassandra -f`.


Once the database instance is up, the next step will be to test communication with it. For this, the client mentioned above, will be used. Thus, to execute it, it is necessary to run the `./cqlsh` command inside the `bin` folder.

```bash
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> SHOW VERSION
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
cqlsh>
```


## Settings within the yaml file

To run a single node, the default values are sufficient, however, to run in a cluster with more than one node, some changes are important. The main settings are found in the `conf` folder, in the `cassandra.yaml` file. The following stand out:


* `cluster_name`: the name of the cluster.
* `seeds`: The IPs, the Internet Protocol (the computer address) of the seed nodes separated by commas.
* `listen_address`: The IPs of the seed nodes, or the instances of Cassandra that will be used as a reference in a startup, separated by commas. These instances will be responsible for "training" new servers newly arrived in the cluster. Thus, it will be these nodes that will be in charge of sending all the necessary information so that new nodes can work within the cluster.
* `listen_interface`: Tells Cassandra which interface to use, consequently, which address to use. It is necessary to modify the `listen_address` or this configuration, however, not both.



## Simplifying installation with containers


One way to install Cassandra is through a container with Docker. In a general view, a container is an isolated environment and Docker technology uses the linux kernel and resources, for example, Cgroups and namespaces, to segregate processes, so that they can be executed independently.

The purpose of containers is to create such independence: the ability to run multiple processes and applications in isolation, make better use of the infrastructure, maintain security between the containers executed, in addition to the ease of creation and maintenance.

Once Docker is installed (https://docs.docker.com/install/), run the following command on the console:

```bash
docker run -d --name casandra-instance -p 9042:9042 cassandra
```

This command downloads and runs[ https://store.docker.com/images/cassandra](https://store.docker.com/images/cassandra), Cassandra's official image, directly from the *docker hub*.

To run `cqlsh` inside Docker:

* List the containers running on the machine with the `docker ps` command

```bash
$ docker ps
CONTAINER ID IMAGE
7373093f921a cassandra
```

* Each container created has a unique identifier or ID. The purpose of the previous command was to list the existing containers and their respective Ids. Once you have found the Cassandra container ID,  run the command for `docker exec -it CONTAINER_ID cqlsh`.

```bash
$ docker exec -it 7373093f921a cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4] Use HELP for help.
cqlsh> SHOW VERSION
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
```

By default in Docker, all files are created inside the container. Thus, to extract the volume of data out of the container, it is necessary to map the `/var/lib/cassandra` path, for example:

```bash
docker run --name some-cassandra -p 9042: 9042 -v /my/own/datadir:/var/lib/cassandra -d cassandra
```

## Creating the first cluster with Docker Compose

Following the line of the Docker and container, to execute a cluster, it is necessary to have many containers. One of the tools that allow the execution of multiple containers is Docker Compose.

Compose is a tool to run multiple containers, which is done in a very simple way with a YAML configuration file. Thus, with a single command, it is possible to run many containers. The following file shows a simple configuration using three nodes in the cluster.

The entire description of containers, configuration of each, and how they interrelate is made from a file of extension YML, which by convention has the name `docker-compose.yml`. This file contains the configuration of three Cassandra nodes from Docker images 

> Make sure you replace the volume directory with your own.

```yaml

version: '3.2'

services:

    db-01:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-01
          seeds: db-01, db-02, db-03
        volumes:
          - /home/otaviojava/Environment/nosql/db1:/var/lib/cassandra
    
    db-02:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-02
          seeds: db-01, db-02, db-03
        volumes:
          - /home/otaviojava/Environment/nosql/db2:/var/lib/cassandra
    
    db-03:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-03
          seeds: db-01, db-02, db-03
        volumes:
          - /home/otaviojava/Environment/nosql/db3:/var/lib/cassandra

networks:
    cassandranet:

```


With the `docker-compose.yml` file created, the next steps are very simple:

1. To start the containers: `docker-compose -f docker-compose.yml up -d`
2. To stop and remove the containers: `docker-compose -f docker-compose.yml down`


> For this example, three clusters are being built. If you wanted to run locally, make sure you have enough memory for that.

One possibility is to decrease the memory consumption of the clusters, for example, by starting three nodes and making each node have a maximum of 1 GB of memory.

. Cassandra's cluster configuration file starting each 1 gigabyte node
```bash
version: '3.2'

services:

    db-01:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-01
          seeds: db-01, db-02, db-03
          JVM_OPTS: -Xms1G -Xmx1G
        volumes:
          - /home/otaviojava/Environment/nosql/db1:/var/lib/cassandra
    
    db-02:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-02
          seeds: db-01, db-02, db-03
          JVM_OPTS: -Xms1G -Xmx1G
        volumes:
          - /home/otaviojava/Environment/nosql/db2:/var/lib/cassandra
    
    db-03:
        image: "cassandra"
        networks:
          - cassandranet
        environment:
          broadcast_address: db-03
          seeds: db-01, db-02, db-03
          JVM_OPTS: -Xms1G -Xmx1G
        volumes:
          - /home/otaviojava/Environment/nosql/db3:/var/lib/cassandra

networks:
    cassandranet:

```

### Conclusion

The installation and configuration of Cassandra, whether in a cluster using a container such as Docker or not, proved to be something very simple when compared to a similar cluster configuration within a relational database.

An important point is that the popularity of Docker is not in vain: its ease of execution and configuration for a node or clusters is fascinating, especially for developers. Perhaps this is the reason why Docker is currently considered the greatest tool when it comes to DevOps.

In the next scenes, it we'll discuss how to communicate with Cassandra, something that will be extremely simple if the reader is already used to relational databases.