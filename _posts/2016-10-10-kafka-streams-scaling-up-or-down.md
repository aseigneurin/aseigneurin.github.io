---
layout: post
title:  "Kafka Streams - Scaling up or down"
date:   2016-10-10 11:00:00
tags: kafka streams
language: EN
---

Kafka Streams is a new component of the Kafka platform. It is a library designed as a lightweight library to process data from and to Kafka. In this post, I'm not going to go through a full tutorial of Kafka Streams but, instead, see how it behaves as regards to scaling. By scaling, I mean how Kafka Streams behaves when you add or remove nodes.

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

I have also created the output topic - `AUTH_AVRO` - but the number of partitions has no incidence here.

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

- it reads from the JSON strings from the `AUTH_JSON` Kafka topic (`builder.stream`)
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

Running separately, a producer sends 2000 records every second (more precisely 20 records every 10 milliseconds) to the `AUTH_JSON` Kafka topic.

# Normal run


```
2016-10-07 23:00:22,152 org.apache.kafka.common.utils.AppInfoParser - Kafka version : 0.10.0.0
2016-10-07 23:00:22,152 org.apache.kafka.common.utils.AppInfoParser - Kafka commitId : b8642491e78c5a13
2016-10-07 23:00:22,156 org.apache.kafka.streams.KafkaStreams - Started Kafka Stream process
2016-10-07 23:00:22,157 org.apache.kafka.streams.processor.internals.StreamThread - Starting stream thread [StreamThread-1]
2016-10-07 23:00:22,302 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - Discovered coordinator localhost:9093 (id: 2147483647 rack: null) for group avro-auth-stream-5.
2016-10-07 23:00:22,302 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Revoking previously assigned partitions [] for group avro-auth-stream-5
2016-10-07 23:00:22,302 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - (Re-)joining group avro-auth-stream-5
2016-10-07 23:00:22,322 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:00:22,327 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:00:22,327 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:00:22,331 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - Successfully joined group avro-auth-stream-5 with generation 1
2016-10-07 23:00:22,332 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Setting newly assigned partitions [AUTH_JSON-2, AUTH_JSON-1, AUTH_JSON-3, AUTH_JSON-0] for group avro-auth-stream-5
2016-10-07 23:00:22,347 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_0
2016-10-07 23:00:22,355 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_1
2016-10-07 23:00:22,357 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_2
2016-10-07 23:00:22,359 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_3
2016-10-07 23:00:22,702 com.capitalone.labs.Main - Records processed: 0
2016-10-07 23:00:23,705 com.capitalone.labs.Main - Records processed: 0
```

```
2016-10-07 23:00:52,703 com.capitalone.labs.Main - Records processed: 2292
2016-10-07 23:00:53,704 com.capitalone.labs.Main - Records processed: 4900
2016-10-07 23:00:54,706 com.capitalone.labs.Main - Records processed: 6900
2016-10-07 23:00:55,704 com.capitalone.labs.Main - Records processed: 8900
```

# Scaling up

```
2016-10-07 23:01:29,704 com.capitalone.labs.Main - Records processed: 76900
2016-10-07 23:01:30,707 com.capitalone.labs.Main - Records processed: 78900
2016-10-07 23:01:31,390 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Revoking previously assigned partitions [AUTH_JSON-2, AUTH_JSON-1, AUTH_JSON-3, AUTH_JSON-0] for group avro-auth-stream-5
2016-10-07 23:01:31,394 org.apache.kafka.streams.processor.internals.StreamThread - Removing a task 0_0
2016-10-07 23:01:31,394 org.apache.kafka.streams.processor.internals.StreamThread - Removing a task 0_1
2016-10-07 23:01:31,395 org.apache.kafka.streams.processor.internals.StreamThread - Removing a task 0_2
2016-10-07 23:01:31,395 org.apache.kafka.streams.processor.internals.StreamThread - Removing a task 0_3
2016-10-07 23:01:31,395 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - (Re-)joining group avro-auth-stream-5
2016-10-07 23:01:31,397 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:01:31,398 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:01:31,398 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:01:31,400 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - Successfully joined group avro-auth-stream-5 with generation 2
2016-10-07 23:01:31,401 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Setting newly assigned partitions [AUTH_JSON-1, AUTH_JSON-0] for group avro-auth-stream-5
2016-10-07 23:01:31,402 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_0
2016-10-07 23:01:31,403 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_1
2016-10-07 23:01:31,705 com.capitalone.labs.Main - Records processed: 80582
2016-10-07 23:01:32,705 com.capitalone.labs.Main - Records processed: 81582
2016-10-07 23:01:33,702 com.capitalone.labs.Main - Records processed: 82582
```


```
2016-10-07 23:01:29,546 org.apache.kafka.common.utils.AppInfoParser - Kafka version : 0.10.0.0
2016-10-07 23:01:29,546 org.apache.kafka.common.utils.AppInfoParser - Kafka commitId : b8642491e78c5a13
2016-10-07 23:01:29,551 org.apache.kafka.streams.KafkaStreams - Started Kafka Stream process
2016-10-07 23:01:29,551 org.apache.kafka.streams.processor.internals.StreamThread - Starting stream thread [StreamThread-1]
2016-10-07 23:01:29,696 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - Discovered coordinator localhost:9093 (id: 2147483647 rack: null) for group avro-auth-stream-5.
2016-10-07 23:01:29,696 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Revoking previously assigned partitions [] for group avro-auth-stream-5
2016-10-07 23:01:29,696 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - (Re-)joining group avro-auth-stream-5
2016-10-07 23:01:30,082 com.capitalone.labs.Main - Records processed: 0
2016-10-07 23:01:31,086 com.capitalone.labs.Main - Records processed: 0
2016-10-07 23:01:31,402 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - Successfully joined group avro-auth-stream-5 with generation 2
2016-10-07 23:01:31,404 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Setting newly assigned partitions [AUTH_JSON-2, AUTH_JSON-3] for group avro-auth-stream-5
2016-10-07 23:01:31,419 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_2
2016-10-07 23:01:31,426 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_3
2016-10-07 23:01:32,085 com.capitalone.labs.Main - Records processed: 435
2016-10-07 23:01:33,086 com.capitalone.labs.Main - Records processed: 1698
2016-10-07 23:01:34,086 com.capitalone.labs.Main - Records processed: 2698
```

# Scaling down

```
2016-10-07 23:01:43,087 com.capitalone.labs.Main - Records processed: 11698

Process finished with exit code 130

```

```
2016-10-07 23:01:42,705 com.capitalone.labs.Main - Records processed: 91582
2016-10-07 23:01:43,702 com.capitalone.labs.Main - Records processed: 92582
2016-10-07 23:01:44,704 com.capitalone.labs.Main - Records processed: 93582
...
2016-10-07 23:02:12,705 com.capitalone.labs.Main - Records processed: 121582
2016-10-07 23:02:13,410 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Revoking previously assigned partitions [AUTH_JSON-1, AUTH_JSON-0] for group avro-auth-stream-5
2016-10-07 23:02:13,412 org.apache.kafka.streams.processor.internals.StreamThread - Removing a task 0_0
2016-10-07 23:02:13,412 org.apache.kafka.streams.processor.internals.StreamThread - Removing a task 0_1
2016-10-07 23:02:13,413 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - (Re-)joining group avro-auth-stream-5
2016-10-07 23:02:13,414 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:02:13,414 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:02:13,414 org.apache.kafka.streams.processor.internals.StreamPartitionAssignor - Completed validating internal topics in partition assignor.
2016-10-07 23:02:13,415 org.apache.kafka.clients.consumer.internals.AbstractCoordinator - Successfully joined group avro-auth-stream-5 with generation 3
2016-10-07 23:02:13,415 org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Setting newly assigned partitions [AUTH_JSON-2, AUTH_JSON-1, AUTH_JSON-3, AUTH_JSON-0] for group avro-auth-stream-5
2016-10-07 23:02:13,415 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_0
2016-10-07 23:02:13,416 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_1
2016-10-07 23:02:13,417 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_2
2016-10-07 23:02:13,417 org.apache.kafka.streams.processor.internals.StreamTask - Creating restoration consumer client for stream task #0_3
2016-10-07 23:02:13,705 com.capitalone.labs.Main - Records processed: 128171
2016-10-07 23:02:14,705 com.capitalone.labs.Main - Records processed: 158108
2016-10-07 23:02:15,704 com.capitalone.labs.Main - Records processed: 168900
2016-10-07 23:02:16,703 com.capitalone.labs.Main - Records processed: 170900
```
