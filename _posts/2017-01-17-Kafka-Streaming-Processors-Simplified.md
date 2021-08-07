---
layout: post
title:  Simplified Kafka Streaming Processors For Sorting
date:   2017-01-17 12:00:00
categories: Java
#preview: ...
disqus_id: 18
---

After a week of poking and prodding at the Kafka streams API and reading through tons of docs and confusing examples I have finally distilled it down to its simplest form and I think I can help all the people, like me, out there struggling to understand how to make this powerful tool work in the real world.

So lets jump right in shall we?

I want to make a message sorter for dogstatd JSON objects that are flowing through the Kafka system. This adds some complexity from all the examples on the interwebs because we need to deserialize JSON, do something with the data, and then reserialize it and push it down to other topics. All the examples relating to JSON are confusing and over burdened with custom objects... this will be much simpler, I promise. To do sorting, I want to use the `filter` method and the `branch` method, so in this example we'll be using both of those methods and I'll show you how to use them for message sorting.

_by the way, `Serdes` is a term for "Serializer" - "Deserializer". Its a fancy streaming serialization scheme... you should read up on it if you are unfamiliar with it: [https://en.wikipedia.org/wiki/SerDes](https://en.wikipedia.org/wiki/SerDes)_

First things first, lets get a running Kafka locally. I'm a docker fan so I suggest using the `spotify/kafka` container to do the magic for you. The nice thing about the `spotify/kafka` container is that it is self contained with zookeeper and Kafka in the same container.

~~~
docker run -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=`docker-machine ip \`docker-machine active\`` --env ADVERTISED_PORT=9092 spotify/kafka
~~~

Once we have our kafka going we want to set some environment variables for working with the container

~~~
export KAFKA=`docker-machine ip \`docker-machine active\``:9092
export ZOOKEEPER=`docker-machine ip \`docker-machine active\``:2181
~~~

Now that we have that done. Let's look at the code.

You can get the java project here if you just want to play around: [https://github.com/parabuzzle/KafkaStreamsPlayground](https://github.com/parabuzzle/KafkaStreamsPlayground)

I'm assuming that you don't want to setup an entire dogstatsd implementation just to try this stuff out, so I built a simple mocker class and publisher to push randomized objects into Kafka so you can just shove stuff in and play with the the streaming processors without much ramp up on producers and connectors.

You can use that by running the `StatsProducer` class from your IDE or if you compile the jars down, you can just use the `kafka-run-class.sh` script like this:

~~~
bin/kafka-run-class.sh com.mikeheijmans.kafka.playground.mock.StatsProducer
~~~

That will get data flowing to the `test` topic for you.

To run the streaming processors you just run the `MessageSorter` class from your IDE or run the class like this:

~~~
bin/kafka-run-class.sh com.mikeheijmans.kafka.playground.MessageSorter
~~~

That will copy all messages that have a hostname of `mockedhost1.mikeheijmans.com` to a topic called `firsthost` and it will also send a copy of messages that are of a metric type of `gauge` to a `gauges` topic and of type `counter` to a `counters` topic.

<img class="img-fluid" src="/img/kafka-streams.png"/>

So let's see how all this works.

I've tried to do a good job of commenting the code to explain to the beginners out there on what's happening so rather than try to explain every line here, I'll just let the comments do the talking:

~~~java
/**
 * This class sorts messages into different topics based on the contents of the message
 *
 *
 * MIT License
 *
 * Copyright (c) 2017 Michael Heijmans <mikeheijmans.com>
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in all
 * copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
 * SOFTWARE.
 */

package com.mikeheijmans.kafka.playground;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.Serializer;
import org.apache.kafka.common.serialization.Deserializer;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KStreamBuilder;
import org.apache.kafka.streams.kstream.Predicate;
import org.apache.kafka.connect.json.JsonDeserializer;
import org.apache.kafka.connect.json.JsonSerializer;

import com.fasterxml.jackson.databind.JsonNode;

import java.util.Properties;

public class MessageSorter {

  // The topic we want the source data to come from
  public static final String MAIN_TOPIC = "test";

  public static void main(String[] args) throws Exception {
    Properties props = new Properties();

    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "message-sorter");

    // Change this to YOUR kafka server or just set the KAFKA ENV variable!
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, System.getenv("KAFKA"));
    // Change this to YOUR zookeeper server or just set the ZOOKEEPER ENV variable!
    props.put(StreamsConfig.ZOOKEEPER_CONNECT_CONFIG, System.getenv("ZOOKEEPER"));

    props.put(StreamsConfig.KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    props.put(StreamsConfig.VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());

    // Reset to the earliest position so we can reprocess the messages for the demo
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

    // load a simple json serializer
    final Serializer<JsonNode> jsonSerializer = new JsonSerializer();
    // load a simple json deserializer
    final Deserializer<JsonNode> jsonDeserializer = new JsonDeserializer();
    // use the simple json serializer and deserialzer we just made and load a Serde for streaming data
    final Serde<JsonNode> jsonSerde = Serdes.serdeFrom(jsonSerializer, jsonDeserializer);

    // Setup a string Serde for the key portion of the messages
    final Serde<String> stringSerde = Serdes.String();

    // Setup a builder for the streams
    KStreamBuilder builder = new KStreamBuilder();

    // Just so we know everything is working.. let's report that we are about to start working
    System.out.println("Starting Sorting Job");

    // Lets start a source and set it to deserialize the key as a String and the value as a JSON object.
    //
    // We know that all of our messages are keyed with a string and the payload is a string representation of
    //   a JSON object. By assigning our jsonSerde to the value, Kafka, will automatically deserialize the value
    //   as a JSON object so we can navigate the data in the JSON tree without any extra work.
    KStream<String, JsonNode> source = builder.stream(stringSerde, jsonSerde, MAIN_TOPIC);


    // Predicates... these are fun to figure out...
    // Think of Predicates as simple matchers that know how to traverse the data and return a boolean
    //
    // At this point we want to setup a few Predicates to use for sorting

    // This Predicate is checking to see if the Hostname is mockedhost1.mikeheijmans.com
    Predicate<String, JsonNode> isFirstHost = (k, v) ->
      v.path("host")                           // Read the value of "host" from the json object
      .asText()                                // Turn it into a String -> This is a gotcha for newbies like me...
      .equals("mockedhost1.mikeheijmans.com"); // Return a boolean of what we want

    // This Predicate is checking if the metrics object is of the type "Counter"
    Predicate<String, JsonNode> isCounter = (k, v) ->
      v.path("type")       // Read the value of "type" from the json object
      .asText()            // Turn it into a String -> This is a gotcha for newbies like me...
      .equals("counter");  // Return a boolean of what we want

    // This Predicate is checking if the metrics object is of the type "Gauge"
    Predicate<String, JsonNode> isGauge = (k, v) ->
      v.path("type")       // Read the value of "type" from the json object
      .asText()            // Turn it into a String -> This is a gotcha for newbies like me...
      .equals("gauge");    // Return a boolean of what we want


    // Simple filtering processor
    // We setup another stream called firstHost to receive messages that match the filter with the
    //   the "isFirstHost" predicate
    KStream<String, JsonNode> firstHost = source.filter(isFirstHost);

    // Take all messages coming into the firstHost steam and sink them to the 'firsthost' topic
    //   we are re-serializing the object back into a string using our jsonSerde on the value field.
    firstHost.to(stringSerde, jsonSerde, "firsthost");

    // Branching processor
    // Here we setup a branching filter to branch gauges and counters into seperate streams
    //   Everything that matches the first Predicate is put into index 0 of the array and
    //   everything that matches the second Predicate is put into the index of 1 of the array
    //   and so on for multiple branching Predicates
    KStream<String, JsonNode>[] metricTypes = source.branch(isGauge, isCounter);

    // Take all messages coming into the metricTypes streams and sink them to the proper topics
    // In this case, metricTypes[0] is filled with gauge events and metricTypes[1] is filled with counter
    //   events. We sink them to appropriately named topics. And again, we use the stringSerde for the key
    //   and the jsonSerde for the value so that the json object gets turned back to a string properly on egress.
    metricTypes[0].to(stringSerde, jsonSerde, "gauges");
    metricTypes[1].to(stringSerde, jsonSerde, "counters");

    // Now lets use that builder and the props we set to setup a streams object
    KafkaStreams streams = new KafkaStreams(builder, props);

    // Start the streaming processor
    streams.start();

    // Gracefully shutdown on an interrupt
    Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
  }

}
~~~

That's essentially it. If the comments are unclear or you have questions, feel free to leave a comment or DM on twitter and I'd be happy to help make it clearer.

Other than that, go grab the example code and have fun! [https://github.com/parabuzzle/KafkaStreamsPlayground](https://github.com/parabuzzle/KafkaStreamsPlayground)
