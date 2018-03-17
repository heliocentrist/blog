---
layout: post
title:  "SQL queries on Apache Kafka"
date:   2018-03-16 12:18:00
categories: post
---

{% newthought 'Apache Kafka' %} is a good example of a great product that knows both its strengths and its limits. One feature that Confluent, the developers of Kafka, apparently do not want to support is random access to the messages in the topics, or search queries on those messages. This is an exercise that Kafka’s documentation leaves to the reader.

To Confluent’s credit they supply us with excellent libraries and frameworks, such as KSQL or Kafka Streams, that we can use to build our own queryable stores, backed by Kafka topics. However, if you deal with Kafka, there comes a time when you just wish that you could run a simple, boring SQL query to find the subset of data that results in some weird behaviour or that your boss wants to see right this minute.

I'm glad to tell you that it is in fact possible, and you don't have to write a single line of code to search, filter, slice and dice the Kafka topics, as long as they contain JSON messages. You can even use your favourite JDBC client to perform those SQL queries.<!--more-->

Now, with enough buildup, I can explain that I am talking about a tool called Apache Drill. It's a SQL query engine that can read data—and deduce schemas—from a number of various backends, such as Hadoop, MongoDB, Hive or just a file system. As I found out recently, Drill has a plugin that allows us to use Kafka as backend for its queries. In this post I will show how to set up this plugin and what capabilities it gives you.

In this post I'm assuming that you're using macOS. The commands should also work on Linux without too much hassle.

<hr class="inliner">

## Index

* Table of contents
{:toc}

<hr class="inliner">

## Preparing a Kafka cluster

{% newthought 'If you have read this far into the post' %}, I assume that you know what Kafka is. On the off chance that you don't, Kafka is a popular streaming platform built around the idea of a distributed log{% sidenote 'One' 'Distributed streaming what? Find a more detailed explanation here: [kafka.apache.org/intro](https://kafka.apache.org/intro)' %}.If you already have a cluster that you can play with, then you can skip this section, otherwise read on to find out how to start up a Kafka cluster and push some data into it.

### Quick installation

The easiest way to run a Kafka broker is to use a distribution that Confluent themselves provide, aptly named the Confluent Platform.

```sh
wget http://packages.confluent.io/archive/4.0/confluent-oss-4.0.0-2.11.tar.gz
tar -xvzf confluent-oss-4.0.0-2.11.tar.gz
cd confluent-4.0.0
./bin/confluent start
```

{% marginfigure 'mf-id-confluentservices' 'assets/img/kafkasql/confluent-services.png' 'UP is good.' %}You should see a bunch of messages reporting that the necessary services are "UP". This means that we can proceed by pushing some messages into our new cluster. We will not use most of these services in this exercise, but for simplicity's sake we start the Confluent Platform in its default configuration.

### Producing some test messages

Run the Kafka producer for a topic called `drilltest`:

```sh
./bin/kafka-console-producer --broker-list localhost:9092 --topic drilltest
```

Then, when you're in the `>` prompt you can enter some JSON messages, for example:

```json
{ "a": 123, "b": "drilling" }
{ "a": 321, "b": "not anymore", "c": "testing" }
```

You will probably see an error complaining that a LEADER is not AVAILABLE. Disregard it, this is only Kafka complaining that it doesn't know the topic. But the topic will be auto-created for us.

## Preparing Apache Drill

{% newthought 'Now that we have a Kafka cluster' %} that we'd like to explore, let's proceed with installing and configuring Apache Drill.

### Installation

Run the following commands to download, unpack and run Drill:

```shell
wget http://apache.mirrors.hoobly.com/drill/drill-1.12.0/apache-drill-1.12.0.tar.gz
tar -xvzf apache-drill-1.12.0.tar.gz
cd apache-drill-1.12.0
./bin/drill-embedded
```

Drill will take several seconds to start up, and then you should see a quirky welcome message{% sidenote 'OneAndHalf' 'I got: *the only truly happy people are children, the creative minority and drill users*' %} and a prompt that looks like this: `0: jdbc:drill:zk=local>`. We can test the Drill's query capabilities by trying it out on a filesystem backend.

First, prepare a file to query:

```sh
echo '[{"a": 1}, {"a": 2}, {"a": 3, "b": "dfs test"}]' > /tmp/dfstest.json
```

Then, run a query in Drill:

```sql
SELECT * FROM dfs.tmp.`dfstest.json`;
```

You should see the data from the JSON file laid out neatly in columns:
{% maincolumn 'assets/img/kafkasql/dfstest.png' '' %}

If you do, that means that Drill is properly installed! Note that it has guessed what a schema for our data could look like. Now we can configure Drill to read from Kafka.

### Pointing Drill at Kafka

For this part we are going to use the web UI of Apache Drill{% sidenote 'Two' 'I encourage you to look around in the Drill web UI, it has a few tools you might find useful' %}. Open [localhost:8047/storage](localhost:8047/storage) in your browser, find *Kafka* in the list of *Disabled Storage Plugins* and click *Update* next to it. You'll get to the *Configuration* page for the Kafka plugin. Enter the following into the big text field:

```json
{
  "type": "kafka",
  "kafkaConsumerProps": {
    "key.deserializer": "org.apache.kafka.common.serialization.ByteArrayDeserializer",
    "auto.offset.reset": "earliest",
    "bootstrap.servers": "localhost:9092",
    "enable.auto.commit": "true",
    "group.id": "drill-query-consumer-1",
    "value.deserializer": "org.apache.kafka.common.serialization.ByteArrayDeserializer",
    "session.timeout.ms": "30000"
  },
  "enabled": true
}
```

Then click *Update* and *Back*. That's all it takes! Finally we can unleash the Drill's queries on that `drilltest` Kafka topic that we've created earlier.

### Finally, SQL queries on a topic

In the Drill prompt enter:

```sql
USE kafka;
SHOW tables;
```

At this point you should see all Kafka topics that currently exist in the cluster, including our `drilltest`. Proceed with:

```sql
SELECT * FROM drilltest;
```

You should see all the messages that you've sent to your topic earlier:

{% maincolumn 'assets/img/kafkasql/drilltest1.png' '' %}

Let's try something more sophisticated:

```sql
SELECT * FROM drilltest WHERE a = 123;
```

{% maincolumn 'assets/img/kafkasql/drilltest2.png' '' %}

Note that next to the data from JSON the table also includes metadata columns like `kafkaPartitionId`, `kafkaMessageOffset` and `kafkaMsgTimestamp`. I've found those immensely useful, because they allow you to query messages based on their time of arrival or find out something about your data. For example, this is how we find out the earliest and the latest timestamp of all messages in the topic.

```sql
SELECT kafkaPartitionId,
    from_unixtime(MIN(kafkaMsgTimestamp)/1000) AS minKafkaTS,
    from_unixtime(MAX(kafkaMsgTimestamp)/1000) AS maxKafkaTS
    FROM drilltest GROUP BY kafkaPartitionId;
```

{% maincolumn 'assets/img/kafkasql/partitions.png' '' %}

## Exploring Kafka topics from DBeaver

{% newthought 'In the beginning of the post' %} I promised that you can use a JDBC client to run SQL queries on Kafka topics, and now it's time to fulfil the promise. My client of choice is [DBeaver](https://dbeaver.jkiss.org/). If you'd like to follow along, you can download it here: [dbeaver.jkiss.org/download](https://dbeaver.jkiss.org/download/). You should keep Drill running for this exercise.

Once DBeaver is installed, run it and click the leftmost button in the menu bar to create a new connection. You will see the list of drivers available in your system. Expand the *Hadoop* entry and you will find the *Apache Drill* driver. Select it and click *Next*. You'll get to the *Connection settings* screen. Now you just need to set the connection details, specifically the JDBC URL. Click *Edit Driver Settings* and in that screen change the *URL Template* to ```jdbc:drill:drillbit=localhost```. Click *OK* to go back and make sure that your change has been applied. You can test the connection to make sure everything is tied together, and then click *Finish*{% sidenote 'Three' 'If you have errors while configuring the driver, please ensure that you have the latest versions of both Drill and DBeaver. I am using Drill 1.12.0 and DBeaver 5.0.0.' %}.

This is it! Now you have a connection to Drill, and you can expand the ```kafka``` schema to find the topics (which are called *Tables* in this view). DBeaver allows you to treat those topics like if they are database tables, so you can view the data and run queries on it.

Isn't it beautiful:
{% maincolumn 'assets/img/kafkasql/dbeaver.png' '' %}

## Limitations and further reading

{% newthought 'In the Drill release I am using' %} the Kafka plugin is included, and you can find its source code and documentation [on Github](https://github.com/apache/drill/tree/master/contrib/storage-kafka). However, at the moment the  plugin only works with JSON messages. The developers do have a plan to support Avro in one of the future releases.

You might find that a query on a big busy Kafka topic can be rather slow. This is not Drill's fault, it is just a consequence of how Kafka is designed. For this reason I wouldn't recommend using Drill queries on Kafka in hot paths in production, this feature is better suited for development and ad-hoc support scenarios.

For the purposes of this tutorial we have started Drill in the simplest embedded mode. If you like what the tool has to offer, you can find out how to run it in a cluster setup in the [Drill Documentation](https://drill.apache.org/docs/install-drill-introduction/).

In this tutorial I was using the following versions of tools:

<div class="table-wrapper">
<br/>
  <table class="table-alpha" id="newspaper-tone">
    <tbody>
      <tr>
        <td class="text">Apache Drill</td>
        <td class="number">1.12.0</td>
      </tr>
      <tr>
        <td class="text">Apache Kafka</td>
        <td class="number">1.0.0-cp1</td>
      </tr>
      <tr>
        <td class="text">Confluent Platform</td>
        <td class="number">4.0.0</td>
      </tr>
      <tr>
        <td class="text">DBeaver</td>
        <td class="number">5.0.0</td>
      </tr>
    </tbody>
  </table>
</div>
