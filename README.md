# Running Kafka Streams applications against a containerized Kafka cluster with Confluent Open Source

----
**Table of Contents**

* [What we want to do](#what-we-want-to-do)
* [Terminology](#Terminology)
* [Requirements](#Requirements)
* [Start a containerized Kafka cluster](#Start-cluster)
    * [Step 1: Clone this repository](#clone-repo)
    * [Step 1.5: Mac and Windows users only -- start Docker Machine](#start-docker-machine)
    * [Step 2: Start Kafka cluster](#start-kafka-cluster)
    * [Done: Kafka cluster is ready!](#cluster-ready)
* [Run Confluent demo applications for Kafka Streams API](#run-demos)
    * [Clone the examples repository](#clone-example-repo)
    * [Build and package the Kafka Streams API examples](#package-examples)
    * [Run the WordCount demo application](#run-wordcount)
    * [Write your own Kafka Streams application](#write-app)
* [Stop the containerized Kafka cluster](#Stop-cluster)
* [Where to go from here](#next-steps)
* [Appendix](#appendix)
    * [Helpful docker commands](#helpful-docker)
----


<a name="what-we-want-to-do"></a>
# What we want to do

The [Apache Kafka](http://kafka.apache.org/) project includes the
[Kafka Streams API](http://docs.confluent.io/current/streams/index.html), which is a Java library for building
applications in Java/Scala/Clojure/... that process and analyze data stored in Kafka.  The cool part about the Kafka
Streams API is that it makes your applications
[highly scalable, elastic, stateful, and fault-tolerant](http://docs.confluent.io/3.2.0/streams/introduction.html)
-- and all this without requiring any processing cluster.  This document helps you to more quickly and more conveniently
develop Kafka Streams applications on your laptop, build server, etc. by running and integrating with a containerized
Kafka cluster.

With the code and instructions in this repository, we will:

1. Start a containerized, single-node Kafka cluster on your host machine, e.g. your Mac laptop, using
   [Docker Compose](https://docs.docker.com/compose/) and Confluent's
   [Docker images for Confluent Platform](https://github.com/confluentinc/cp-docker-images); more precisely, we use
   [Confluent Open Source](https://www.confluent.io/download/) version 3.2.0 with Apache Kafka 0.10.2.0, i.e. the latest
   versions as of March 2017.
2. Build and package the [Confluent demo applications](https://github.com/confluentinc/examples) for the Kafka Streams
   API on your host machine.
3. Run one of the demo applications (here: WordCount) on your host machine against the containerized Kafka cluster.
4. You can interactively enter the input data for the WordCount application.

**"Why should I do this?"** A local development setup such as the above is very useful when doing iterative development,
when you don't have access to a Kafka cluster from your laptop, when you would like to experiment with failure
scenarios, and for many more reasons.  Did we already mention that it's also a heck of a lot of fun?

**"How much time do I need?"** The expected time to complete this walkthrough is 10 minutes, excluding the time to
[install prerequisites](#Requirements) such as Docker.


<a name="Terminology"></a>
# Terminology

* The **host machine** is the machine that is running Docker; for example, your Mac laptop.
* On Mac OS and Windows OS, the [**Docker Machine**](https://docs.docker.com/machine/overview/)
  is the VM in which your Docker containers run.


<a name="Requirements"></a>
# Requirements

Your host machine must have the following software installed:

* Docker 17.03.1-ce (2017-03-27) or later.

    ```shell
    $ docker --version
    Docker version 17.03.1-ce, build c6d412e

    $ docker-compose --version
    docker-compose version 1.11.2, build dfed245

    # Mac and Windows users only
    $ docker-machine --version
    docker-machine version 0.10.0, build 76ed2a6
    ```

* The following software is required to build and run the
  [Confluent demo applications](https://github.com/confluentinc/examples) for the Kafka Streams API.
  There's no container provided for this part because, arguably, most users are developing Kafka Streams applications
  directly on their host machines, e.g. via an IDE on their Mac laptops.
    * git
    * Maven 3
    * Java JDK 8+


<a name="Start-cluster"></a>
# Start a containerized Kafka cluster

Here, we essentially follow the
[Confluent Docker Quickstart](http://docs.confluent.io/3.2.0/cp-docker-images/docs/quickstart.html) in the Confluent
documentation.  If you run into problems, Confluent's Docker Quickstart has troubleshooting tips available.


<a name="clone-repo"></a>
## Step 1: Clone this repository

Clone this repository:

```bash
# Clone this repository to `$HOME/kafka-streams-docker` aka `~/kafka-streams-docker`.
$ git clone https://github.com/miguno/kafka-streams-docker.git ~/kafka-streams-docker
```

<a name="start-docker-machine"></a>
## Step 1.5: Mac and Windows users only -- start Docker Machine

Create a VM with 6GB of memory as our Docker Machine:

```bash
# Create a VirtualBox VM with ~6GB of memory to serve as our Docker Machine.
$ docker-machine create --driver virtualbox --virtualbox-memory 6000 confluent
```
Run `docker-machine ls` to verify that the Docker Machine is running correctly.
The command's output should be similar to:

```bash
$ docker-machine ls
NAME        ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
confluent   *        virtualbox   Running   tcp://192.168.99.100:2376           v17.03.1-ce
```

Now configure your terminal to attach it to the new Docker Machine named `confluent`:

```bash
$ eval $(docker-machine env confluent)
```

> **Important**: Whenever you run Docker commands from a terminal window, then this terminal must be attached to the
> Docker Machine first via `eval $(docker-machine env confluent)`.  Keep this in mind when opening new terminal
> windows!


<a name="start-kafka-cluster"></a>
## Step 2: Start the Kafka cluster

Next, we start a containerized Kafka cluster (1 broker) with a ZooKeeper ensemble (1 node) in the background.

```bash
# Change into the `kafka-streams-docker` directory from step 1,
# which is where `docker-compose.yml` resides.
$ cd ~/kafka-streams-docker
```

> **Additional command for Mac and Windows users:**
>
> ```bash
> # Make the Docker Machine's IP address available via the DOCKER_MACHINE_IP
> # environment variable, which is used by docker-compose.yml.
> $ export DOCKER_MACHINE_IP=`docker-machine ip confluent`
> ```

```bash
# Start the cluster
$ docker-compose up -d
```

Verify that the containers are running:

```bash
$ docker-compose ps

# You should see the following:
          Name                       Command            State   Ports
---------------------------------------------------------------------
streamsdocker_kafka_1       /etc/confluent/docker/run   Up
streamsdocker_zookeeper_1   /etc/confluent/docker/run   Up
```

Verify that the ZooKeeper node is healthy:

```bash
$ docker-compose logs zookeeper | grep -i "binding to port"

# You should see a line similar to:
zookeeper_1  | [2017-03-31 20:00:57,076] INFO binding to port 0.0.0.0/0.0.0.0:32181 (org.apache.zookeeper.server.NIOServerCnxnFactory)
```

Verify that the Kafka broker is healthy:

```bash
$ docker-compose logs kafka | grep -i "started (kafka.server.KafkaServer)"

# You should see a line similar to:
kafka_1      | [2017-03-31 20:00:58,910] INFO [Kafka Server 1], started (kafka.server.KafkaServer)
```


<a name="cluster-ready"></a>
## Done: Kafka cluster is ready!

At this point, the Kafka cluster is up and running.  To recap, we have now available to us:

| Service        | Container name | Access on Mac/Windows hosts | Access on Linux hosts | Kafka parameter     |
|----------------|----------------|-----------------------------|-----------------------|---------------------|
| Kafka broker   | kafka          | `$DOCKER_MACHINE_IP:29092`  | `localhost:29092`     | `bootstrap.servers` |
| ZooKeeper node | zookeeper      | `$DOCKER_MACHINE_IP:32181`  | `localhost:32181`     | `zookeeper.connect` |


Note: The Kafka broker and the ZooKeeper node are both accessible from *other containers* via the `localhost:PORT`
setting in the column "Access on Linux hosts" above.


<a name="run-demos"></a>
# Run Confluent demo applications for Kafka Streams API


<a name="clone-examples-repo"></a>
## Clone the examples repository

Clone the repository that contains the Confluent demo applications:

```bash
# Clone the examples repository to `$HOME/examples` aka `~/examples`.
$ git clone https://github.com/confluentinc/examples.git ~/examples
```


<a name="package-examples"></a>
## Build and package the Kafka Streams API examples

Next, we must build and package the examples into a so-called "fat" jar:

```bash
# Change into the directory that contains the examples for the Kafka Streams API
$ cd ~/examples/kafka-streams

# We want to use examples that work with Confluent 3.2.x
$ git checkout 3.2.x

# Build and package the examples.
$ mvn -D skipTests=true clean package

>>> Creates ~/examples/kafka-streams/target/streams-examples-3.2.0-standalone.jar
```

Now we can run any of the Kafka Streams API examples.  Each example such as
the [WordCountLambdaExample](https://github.com/confluentinc/examples/blob/3.2.x/kafka-streams/src/main/java/io/confluent/examples/streams/WordCountLambdaExample.java)
ships with instructions how to use it.  The only parts in the instructions of an example that we need to modify are
**where to find the Kafka brokers** aka Kafka's `bootstrap.servers` parameter because the instructions in the examples
assume `localhost:9092` by default.

* **Mac and Windows users**: `bootstrap.servers` is `$DOCKER_MACHINE_IP:29092`
* **Linux users**: `bootstrap.servers` is `$localhost:29092`

All examples allow us to override the `bootstrap.servers` parameter via a CLI argument, and for *most* examples you do
so by providing the `bootstrap.servers` parameter as the *first* CLI argument (for examples demonstrating Kafka's
[Interactive Queries](http://docs.confluent.io/3.2.0/streams/developer-guide.html#interactive-queries) feature, it is
the *second* CLI argument).


<a name="run-wordcount"></a>
## Run the WordCount demo application

Let's test-drive the aforementioned [WordCountLambdaExample](https://github.com/confluentinc/examples/blob/3.2.x/kafka-streams/src/main/java/io/confluent/examples/streams/WordCountLambdaExample.java)
application.  The "steps" below refer to the steps in the example's instructions.

Step 1: We can skip step 1 in the example's instructions because Kafka and ZooKeeper are already running (see above).

Step 2: Create the input and output topics used by the WordCount application.

```bash
# If you haven't done so, change into the `kafka-streams-docker` directory
# from step 1, which is where `docker-compose.yml` resides.
$ cd ~/kafka-streams-docker

# Create the application's input topic "TextLinesTopic".
$ docker-compose exec kafka kafka-topics \
    --create --topic TextLinesTopic \
    --zookeeper localhost:32181 --partitions 1 --replication-factor 1

# Create the application's output topic "WordsWithCountsTopic".
$ docker-compose exec kafka kafka-topics \
    --create --topic WordsWithCountsTopic \
    --zookeeper localhost:32181 --partitions 1 --replication-factor 1
```

> **Tip:** If you have [Confluent Open Source](https://www.confluent.io/download/) installed locally on your host
> machine, then you can also run the Kafka CLI commands such as `kafka-topics` and `kafka-console-producer` in this
> section directly from your host machine, rather than indirectly via `docker-compose exec ...` from inside the `kafka`
> container.  For example, with Confluent Open Source available locally, the first `docker-compose exec` command above
> that executes `kafka-topics` could also be run directly on the host machine as:
>
> ```bash
> $ /path/to/confluent-3.2.0/bin/kafka-topics \
>     --create --topic TextLinesTopic \
>     --zookeeper localhost:32181 --partitions 1 --replication-factor 1
> ```
>
> This direct approach is particularly helpful if, for example, you want to ingest some local data into Kafka during
> iterative development or for testing and debugging.

You can verify that topics were created successfully with `kafka-topics --list` or `kafka-topics --describe`:

```bash
$ docker-compose exec kafka kafka-topics --describe --topic TextLinesTopic --zookeeper localhost:32181
Topic:TextLinesTopic    PartitionCount:1    ReplicationFactor:1    Configs:
        Topic: TextLinesTopic    Partition: 0    Leader: 1    Replicas: 1    Isr: 1
```

Step 3: Start the WordCount application either in your IDE or on the command line.  In this document, we use the
command line.

**Mac and Windows users:**

```bash
# Start the WordCount application
$ export DOCKER_MACHINE_IP=`docker-machine ip confluent`
$ java -cp ~/examples/kafka-streams/target/streams-examples-3.2.0-standalone.jar \
           io.confluent.examples.streams.WordCountLambdaExample $DOCKER_MACHINE_IP:29092
```

**Linux users:**

```bash
# Start the WordCount application
$ java -cp ~/examples/kafka-streams/target/streams-examples-3.2.0-standalone.jar \
           io.confluent.examples.streams.WordCountLambdaExample localhost:29092
```

The application will continue to run in the terminal until you stop it via `Ctrl-C` -- but don't stop it just yet
because we are not done yet with this example!

Step 4: Write some input data to the source topic "TextLinesTopic", e.g. via `kafka-console-producer`.
The already running WordCount application (step 3) will automatically process this input data
and write the results to the output topic "WordsWithCountsTopic".

```bash
# Tip: Use a new terminal for the following commands!
# (Mac and Windows users: ensure the new terminal is attached to Docker Machine, see above)

# Start the console producer.
$ docker-compose exec kafka kafka-console-producer \
    --broker-list localhost:29092 \
    --topic TextLinesTopic
```

The console producer will start up and wait for your input (unfortunately it does not show a proper prompt to indicate
this).  Any text lines you enter now will be turned into Kafka messages and sent to the input topic "TextLinesTopic".
Let's enter some input data:

```bash
hello kafka streams<ENTER>
all streams lead to kafka<ENTER>
join kafka summit<ENTER>
```

This will send 3 messages to the input topic: message keys are `null`, and message values are the textlines,
e.g. "hello kafka streams".  If you want to, you can terminate the console producer now via `Ctrl-C`.
Alternatively, you can keep it running and enter more input data later.

Step 5: Inspect the resulting data in the output topic, e.g. via `kafka-console-consumer`.

```bash
# Tip: Use a new terminal for the following commands!
# (Mac and Windows users: ensure the new terminal is attached to Docker Machine, see above)

# Start the console consumer.
$ docker-compose exec kafka kafka-console-consumer \
    --bootstrap-server localhost:29092 --new-consumer \
    --topic WordsWithCountsTopic --from-beginning \
    --property print.key=true \
    --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```

You should see output data similar to below.

```
hello    1
kafka    1
streams  1
all      1
streams  2
lead     1
to       1
join     1
kafka    3
summit   1
```

> Note: The exact output sequence will depend on how fast you type the above sentences.  If you type them slowly,
> you are likely to get each count update, e.g., "kafka 1", "kafka 2", "kafka 3".  If you type them quickly, you
> are likely to get fewer count updates, e.g., just "kafka 3".  This is because the commit interval is set to
> 10 seconds, and anything typed within that interval will be "compacted" in memory (cf.
> [record caches in the DSL](http://docs.confluent.io/3.2.0/streams/developer-guide.html#record-caches-in-the-dsl)).

You can stop the console consumer at any time with `Ctrl-C`.  Alternatively, you can keep it running and
enter more input data via the console producer that runs in your other terminal.

Once you are done with your experiments, you can stop the WordCount application via `Ctrl-C` as well as
the console producer/consumer.


<a name="write-app"></a>
## Write your own Kafka Streams application

If by now you want to write your own Kafka Streams application, head over to the
[Confluent documentation](http://docs.confluent.io/current/) and read the chapter on
the [Kafka Streams API](http://docs.confluent.io/current/streams/).


<a name="Stop-cluster"></a>
# Stop the containerized Kafka cluster

To shutdown the Kafka cluster:

```shell
# Stop and remove containers.
# Careful: THIS STEP WILL RESULT IN LOSING ALL THE DATA THAT IS STORED IN KAFKA AND ZOOKEEPER.
$ docker-compose down
```

> Tip: If you want to preserve the containers including any of their local data such as Kafka topics, you must use
> `docker-compose stop` (rather than `down`), and subsequently `docker-compose start` (rather than `up`) to re-start the
> same cluster again.

**Mac and Windows users only:**: If you also want to shutdown the Docker Machine:

```shell
# Option 1: Gracefully stop it but don't throw it away.
# Re-use by restarting with `docker-machine start confluent`.
$ docker-machine stop confluent

# Option 2: Stop it and throw it away.
# Start from scratch via `docker-machine create ...` (see above).
$ docker-machine rm -f confluent
```


<a name="next-steps"></a>
# Where to go from here

Hopefully you enjoyed this quick walkthrough!

As next steps, you may want to:

* [Download Confluent Open Source](https://www.confluent.io/download/)
* Want to write your own Kafka Streams application now?
  Read the [Confluent documentation](http://docs.confluent.io/current/) for the
  [Kafka Streams API](http://docs.confluent.io/current/streams/) to get started!
* Further information on [using Docker with Confluent](http://docs.confluent.io/current/cp-docker-images/docs/)
* Browse through further [Confluent demo applications](https://github.com/confluentinc/examples) for the Kafka Streams API
* Join our Confluent Community Slack at https://confluentcommunity.slack.com/, notably the `#streams` channel
  (you need to [register a free account](https://slackpass.io/confluentcommunity) first)


<a name="appendix"></a>
# Appendix

<a name="helpful-docker"></a>
## Helpful docker commands

> **IMPORTANT:** `docker-compose` commands must be run from the directory in which `docker-compose.yml` resides.

Show running containers:

```bash
$ docker-compose ps
$ docker ps

# Example:
$ docker-compose ps
Name                       Command            State   Ports
---------------------------------------------------------------------
streamsdocker_kafka_1       /etc/confluent/docker/run   Up
streamsdocker_zookeeper_1   /etc/confluent/docker/run   Up

$ docker ps
CONTAINER ID        IMAGE                             COMMAND                  CREATED             STATUS              PORTS               NAMES
fd33621e20a8        confluentinc/cp-kafka:3.2.0       "/etc/confluent/do..."   11 minutes ago      Up 11 minutes                           streamsdocker_kafka_1
b84974d3df15        confluentinc/cp-zookeeper:3.2.0   "/etc/confluent/do..."   11 minutes ago      Up 11 minutes                           streamsdocker_zookeeper_1
```

Show ALL containers, including those that are not running:

```bash
$ docker-compose ps -a
$ docker ps -a
```

Log into the Docker Machine named "confluent":

```bash
$ docker-machine ssh confluent
```

Log into a running container by opening a shell:

```bash
$ docker-compose exec <container id or name> /bin/bash
$ docker exec -ti <container id or name> /bin/bash

# Example:
$ docker-compose exec kafka /bin/bash
```

Show the logs of a running container:

```bash
# Print logs and exit
$ docker-compose logs <container id or name>
$ docker logs <container id or name>

# Print logs continuously until you stop the command via `Ctrl-C` (think: `tail -f`)
$ docker-compose logs -f <container id or name>
$ docker logs -f <container id or name>

# Example:
$ docker-compose logs -f kafka
```
