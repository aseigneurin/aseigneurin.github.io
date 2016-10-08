---
layout: post
title:  "Kafka Streams - Scaling up or down"
date:   2016-10-07 11:00:00
tags: kafka streams
language: EN
---

Kafka Streams is a new component of the Kafka platform. It is a lightweight library designed to process data from and to Kafka. In this post, I'm not going to go through a full tutorial of Kafka Streams but, instead, see how it behaves as regards to scaling. By scaling, I mean the process of adding or removing nodes to increase or decrease the processing power.

# The theory

Kafka Streams can work as a distributed processing engine and scale horizontally. However, as opposed to Spark or Flink, Kafka Streams does not require setting up a cluster to run the application. Instead, you just start as many instances of the application as you need, and Kafka Streams will rely on Kafka to distribute the load.

To do so, Kafka Streams will register all the instances of your application in the same *consumer group*, and **each instance will take care of some of the partitions of the Kafka topic**. As a consequence, the maximum number of instances of your application you can start is equal to the number of partitions in the topic.

Scaling is then made very easy:

- to scale up: start a new instance of the application and it will take over the responsibility of some partitions from other instances
- to scale down: stop an instance and other instances will take care of the no-longer processed partitions.

# The application

Given the condition described above, I have created a topic named `AUTH_JSON` with 4 partitions:

```
$ .../confluent-3.0.0/bin/kafka-topics --create --topic AUTH_JSON --partitions 4 --replication-factor 1 --zookeeper localhost:2182
```

I have also created the output topic (`AUTH_AVRO`) but the number of partitions has no incidence here.

The consumer application is a standalone Java application. I included the dependencies to the Kafka client and to Kafka Streams using Maven:

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>0.10.0.0</version>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>0.10.0.0</version>
</dependency>
```

The code is pretty simple:

- it reads JSON strings from the `AUTH_JSON` Kafka topic (`builder.stream`)
- it parses the JSON strings into business objects (`flatMapValues`), applies some processing to the objects (first call to `mapValues`) and converts the objects to Avro (second call to `mapValues`)
- it writes the result to the `AUTH_AVRO` Kafka topic (`to`)

```java
public static void main(String[] args) throws SchedulerException {
    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "auth-converter");
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(StreamsConfig.ZOOKEEPER_CONNECT_CONFIG, "localhost:2181");
    props.put(StreamsConfig.KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");

    KStreamBuilder builder = new KStreamBuilder();
    KStream<String, String> kafkaInput = builder.stream("AUTH_JSON");
    kafkaInput.flatMapValues(value -> JsonToAuth.call(value))
            .mapValues(value -> DataProcessMap.call(value))
            .mapValues(a -> AvroSerializer.serialize(a))
            .to(Serdes.String(), Serdes.ByteArray(), "AUTH_AVRO");

    KafkaStreams streams = new KafkaStreams(builder, props);
    streams.start();
}
```

As an addition, I have a Quartz job that prints every second the number of records that have been processed since the launch of the application.

Running separately, a producer sends 2000 records every second (more precisely 20 records every 10 milliseconds) to the `AUTH_JSON` Kafka topic. Since the topic has 4 partitions, that's 500 records per partition every second (5 records per partition every 10 milliseconds).

# Normal run

When we start a first instance of the consumer:

- the application joins the *consumer group* `avro-auth-stream`
- it then takes responsibility for processing the 4 partitions of the topic (`AUTH_JSON-2`, `AUTH_JSON-1`, `AUTH_JSON-3` and `AUTH_JSON-0`).

```
11:00:22,152 org.apache.kafka.common.utils.AppInfoParser - Kafka version : 0.10.0.0
11:00:22,152 org.apache.kafka.common.utils.AppInfoParser - Kafka commitId : b8642491e78c5a13
11:00:22,156 org.apache.kafka.streams.KafkaStreams - Started Kafka Stream process
...
11:00:22,331 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - Successfully joined group avro-auth-stream with generation 1
11:00:22,332 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Setting newly assigned partitions [AUTH_JSON-2, AUTH_JSON-1, AUTH_JSON-3, AUTH_JSON-0] for group avro-auth-stream
...
11:00:22,702 com.seigneurin.Main - Records processed: 0
11:00:23,705 com.seigneurin.Main - Records processed: 0
```

Now, if we start the producer, the consumer starts consuming 2000 records per second:

```
11:00:52,703 com.seigneurin.Main - Records processed: 2292
11:00:53,704 com.seigneurin.Main - Records processed: 4900
11:00:54,706 com.seigneurin.Main - Records processed: 6900
11:00:55,704 com.seigneurin.Main - Records processed: 8900
```

# Scaling up

Now, let's start a second instance of the application (just run the same jar one more time in parallel):

- the new instance joins the same *consumer group* `avro-auth-stream`
- it takes responsibility for processing 2 partitions of the topic (`AUTH_JSON-2` and `AUTH_JSON-3`)
- it immediately starts processing 1000 records per second.

```
11:01:29,546 org.apache.kafka.common.utils.AppInfoParser - Kafka version : 0.10.0.0
11:01:29,546 org.apache.kafka.common.utils.AppInfoParser - Kafka commitId : b8642491e78c5a13
11:01:29,551 org.apache.kafka.streams.KafkaStreams - Started Kafka Stream process
...
11:01:31,402 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - Successfully joined group avro-auth-stream with generation 2
11:01:31,404 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Setting newly assigned partitions [AUTH_JSON-2, AUTH_JSON-3] for group avro-auth-stream
...
11:01:32,085 com.seigneurin.Main - Records processed: 435
11:01:33,086 com.seigneurin.Main - Records processed: 1698
11:01:34,086 com.seigneurin.Main - Records processed: 2698
```

If we look at the first instance, we can see:

- it was processing 2000 records per second
- it stops processing the 4 partitions and switches to processing only 2 of them (`AUTH_JSON-1` and `AUTH_JSON-0`)
- it then only processes 1000 records per second.

```
11:01:29,704 com.seigneurin.Main - Records processed: 76900
11:01:30,707 com.seigneurin.Main - Records processed: 78900
11:01:31,390 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Revoking previously assigned partitions [AUTH_JSON-2, AUTH_JSON-1, AUTH_JSON-3, AUTH_JSON-0] for group avro-auth-stream
...
11:01:31,401 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Setting newly assigned partitions [AUTH_JSON-1, AUTH_JSON-0] for group avro-auth-stream
...
11:01:31,705 com.seigneurin.Main - Records processed: 80582
11:01:32,705 com.seigneurin.Main - Records processed: 81582
11:01:33,702 com.seigneurin.Main - Records processed: 82582
```

**Scaling up is trivial: start a new instance and it will take its share of the load.**

# Scaling down

Now, let's kill one of the instances:

```
11:01:43,087 com.seigneurin.Main - Records processed: 11698

Process finished with exit code 130
```

Here is what happens on the remaining instance:

- before stopping the other instance, it is processing 1000 records per second
- **for 30 seconds, nothing changes**: it keeps processing 1000 records per second
- once the 30 seconds period is over, this instance takes back the responsibility of the 4 partitions (`AUTH_JSON-2`, `AUTH_JSON-1`, `AUTH_JSON-3` and `AUTH_JSON-0`)
- once done, it catches up with the processing of the 30000 records (30 seconds * 1000 records per second) that have not been processed (from 121582 to 158108 below)
- it finally runs smoothly again, processing 2000 records per second.

```
11:01:42,705 com.seigneurin.Main - Records processed: 91582
11:01:43,702 com.seigneurin.Main - Records processed: 92582
11:01:44,704 com.seigneurin.Main - Records processed: 93582
...
11:02:12,705 com.seigneurin.Main - Records processed: 121582
11:02:13,410 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Revoking previously assigned partitions [AUTH_JSON-1, AUTH_JSON-0] for group avro-auth-stream
...
11:02:13,415 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Setting newly assigned partitions [AUTH_JSON-2, AUTH_JSON-1, AUTH_JSON-3, AUTH_JSON-0] for group avro-auth-stream
...
11:02:13,705 com.seigneurin.Main - Records processed: 128171
11:02:14,705 com.seigneurin.Main - Records processed: 158108
11:02:15,704 com.seigneurin.Main - Records processed: 168900
11:02:16,703 com.seigneurin.Main - Records processed: 170900
```
