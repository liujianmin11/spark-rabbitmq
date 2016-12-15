[![Coverage Status](https://coveralls.io/repos/github/Stratio/RabbitMQ-Receiver/badge.svg?branch=master)]
(https://coveralls.io/github/Stratio/RabbitMQ-Receiver?branch=master)

# RabbitMQ Spark Streaming Receiver

RabbitMQ-Receiver is a library that allows the user to read data with [Apache Spark Streaming](https://spark.apache.org/)
from [RabbitMQ](https://www.rabbitmq.com/).

## Requirements

This library requires Spark 2.0+, Scala 2.11+, RabbitMQ 3.5+

## Using the library

There are two ways of using RabbitMQ-Receiver library:

The first one is to add the next dependency in your pom.xml:

```
<dependency>
  <groupId>com.stratio.receiver</groupId>
  <artifactId>spark-rabbitmq</artifactId>
  <version>LATEST</version>
</dependency>
```

The other one is to clone the full repository and build the project:

```
git clone https://github.com/Stratio/RabbitMQ-Receiver.git
mvn clean install
```

This library includes two implementations for consuming messages from RabbitMQ with Spark Streaming: 
  
  - The basic consumer has been implemented extending the Receiver Actor [Custom Receiver Documentation] 
  (http://spark.apache.org/docs/latest/streaming-custom-receivers.html)
  - The advanced consumer is distributed, the user can consume messages from RabbitMQ with more than one Spark 
  Executor and more than one consumer in one Executor. Is similar to Kafka Direct implemented by Spark [Kafka Direct Approach]
  (http://spark.apache.org/docs/latest/streaming-kafka-integration.html#approach-2-direct-approach-no-receivers)


### Build

`mvn clean package`


### Distributed Approach

This advanced consumer has been implemented extending the Spark InputDStream class. With this approach the user can 
consume message from multiple rabbitMQ clusters or multiple rabbitMQ queues. In addition is possible to parallelize the
 consumption in one node starting more than one consumer, one for each Spark RDD Partition.
 
 
 - One executor with multiple parallelized consumer from one queue
  
 ![Single-Parallelized](./images/singleParalellized.png)
 
 
 - One executor or more with multiple consumers from multiple queues
   
  ![Multiple-Parallelized](./images/multipleParallelized.png)
 
 
 - Cluster consumer
 
  ![Cluster](./images/cluster.png)
  
 
The consumption options establish the number of partitions of the RDD generated by the RabbitMQDStream, each Spark node 
consumes messages and these messages define the data that are included in the partitions. Is not necessary to save the 
data into the Spark Block Manager. The future transformations and actions use the data stored on each executor.
 
When the Streaming starts the time window is divided, to consume and to compute, by default the time for consuming 
messages from rabbitMQ is 0.9 times the Spark Window. It is possible to limit the time in order to have better 
performance, in the configuration the user can choose the "maxReceiveTime" in milliseconds.
In addition is possible to limit the number of consumed messages with the configuration parameter 
"maxMessagesPerPartition"

This receiver has optimized the RDD functions count and countAprox.

Each executor has one connection pool and are reused on each streaming batch window in order to have better 
performance. The actual kafka direct approach implemented by Spark does not have one connection pool, this provoke 
that on each iteration, the RDDs create a new kafka connection.

This consumer has a limitation, the minimum storage level selected for this RabbitMQDStream is MEMORY_ONLY, the user 
can't select NONE, because on each Spark action the RDD will be re-computed
 

#### Scala API

- String
```
val receiverStream = RabbitMQUtils.createDistributedStream[String](sparkStreamingContext, params, distributedKeys)
```
- Generic user Type
```
val receiverStream = RabbitMQUtils.createDistributedStream[R](sparkStreamingContext, params, distributedKeys, Array[Byte] => R))
```


#### Java API

```
JavaReceiverInputDStream receiverStream = RabbitMQUtils.createJavaDistributedStream[R](javaSparkStreamingContext, params, JFunction[Array[Byte], R]);
```

#### Spark Parameters Options 

| Parameter                 | Description                          | Optional                             |
|---------------------------|--------------------------------------|--------------------------------------|
| maxMessagesPerPartition   | Maximum number of messages           | Yes                                  |
| levelParallelism          | Num. of partitions by executor       | Yes  (default: 1)                    |
| MaxReceiveTime            | Max time to receive messages         | Yes  (default: 0) (auto)             |
| rememberDuration          | Remember duration for Spark Dstreams | Yes  (default: 60s)                  |


### Receiver-based Approach

This is a basic consumer, when the Streaming Context starts Spark run one process in one executor for consuming 
messages from RabbitMQ. 
This consumer has one singleton consumer instance for consuming messages asynchronously, on each spark window the 
consumer receives messages and saves the blocks received inside the Spark Block Memory. All the data is replicated to 
other nodes.
The receiver extends one Akka Actor, this makes that the receiver-base approach implementation has the Akka dependency.
 In future versions of Spark the Akka dependency will be removed.

#### Scala API

- String
```
val receiverStream = RabbitMQUtils.createStream[String](sparkStreamingContext, params)
```
- Generic user Type
```
val receiverStream = RabbitMQUtils.createStream[R](sparkStreamingContext, params, Array[Byte] => R)
```


#### Java API

```
JavaReceiverInputDStream receiverStream = RabbitMQUtils.createJavaStream[R](javaSparkStreamingContext, params, JFunction[Array[Byte], R]);
```


### RabbitMQ Parameters Options 

| Parameter                 | Description                  | Optional                             |
|---------------------------|------------------------------|--------------------------------------|
| hosts                     | RabbitMQ hosts               | Yes (default: localhost)             |
| virtualHosts              | RabbitMQ virtual Host        | Yes                                  |
| queueName                 | Queue name                   | Yes                                  |
| exchangeName              | Exchange name                | Yes                                  |
| exchangeType              | Exchange type                | Yes                                  |
| routingKeys               | Routing keys comma separated | Yes                                  |
| userName                  | RabbitMQ username            | Yes                                  |
| password                  | RabbitMQ password            | Yes                                  |
| durable                   | durable                      | Yes (default: true)                  |
| exclusive                 | exclusive                    | Yes (default: false)                 |
| autoDelete                | autoDelete                   | Yes (default: false)                 |
| ackType                   | basic/auto                   | Yes (default: basic)                 |
| fairDispatch              | fairDispatch                 | Yes (default: false)                 |
| prefetchCount             | prefetchCount                | Yes (default: 1)                     |
| storageLevel              | Apache Spark storage level   | Yes (default: MEMORY_ONLY)           |
| x-max-length              | RabbitMQ queue property      | Yes                                  |
| x-message-ttl             | RabbitMQ queue property      | Yes                                  |
| x-expires                 | RabbitMQ queue property      | Yes                                  |
| x-max-length-bytes        | RabbitMQ queue property      | Yes                                  |
| x-dead-letter-exchange    | RabbitMQ queue property      | Yes                                  |
| x-dead-letter-routing-key | RabbitMQ queue property      | Yes                                  |
| x-max-priority            | RabbitMQ queue property      | Yes                                  |


# License #

Licensed to STRATIO (C) under one or more contributor license agreements.
See the NOTICE file distributed with this work for additional information
regarding copyright ownership.  The STRATIO (C) licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
