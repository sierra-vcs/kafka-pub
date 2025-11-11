# Chapter 11: Kafka Stream Processing

- [11.1 Kafka Streams Overview](#111-kafka-streams-overview)
- [11.2 KStreams vs KTables](#112-kstreams-vs-ktables)
- [11.3 Windowing, Aggregation, and Joins](#113-windowing-aggregation-and-joins)
- [11.4 Stream Processing in .NET (Streamiz.Kafka.Net)](#114-stream-processing-in-net-streamizkafkanet)
- [11.5 Introduction to ksqlDB](#115-introduction-to-ksqldb)

## 11.1 Kafka Streams Overview

### Introduction

Kafka Streams is a client-side library for building applications and microservices that process and analyze data stored in Kafka. Unlike other stream processing frameworks, Kafka Streams is not a separate cluster technology; it is simply a Java library that you can embed in your own applications. This makes it lightweight, easy to deploy, and fully integrated with the Kafka ecosystem.

### Explanation

Kafka Streams is designed to simplify real-time data processing. It allows you to transform, filter, aggregate, and join data from multiple Kafka topics in a continuous, stream-oriented fashion.

- **Client-side Library**: Kafka Streams is a library, not a cluster. This means your application runs as a standard Java application or a microservice. This simplifies deployment and management, as you don't need a dedicated stream processing cluster. You can deploy your Kafka Streams application like any other application, for example, on a VM, in a container, or on Kubernetes.

- **Stateless vs. Stateful Processing**:

  - **Stateless Processing**: This involves transforming a message without any knowledge of previous messages. Examples include filtering out specific events or changing the format of a message.
  - **Stateful Processing**: This is the most powerful feature. It involves operations that require context from previous messages, such as aggregations (e.g., counting a number of events per minute) or joins (e.g., joining a stream of user clicks with a database of user profiles). Kafka Streams handles the management of this state by storing it in local, fault-tolerant **state stores** on the application's machine. These state stores are backed up to internal Kafka topics, ensuring the state is durable and can be recovered if the application fails.

- **High-Level DSL (Domain Specific Language)**: The library provides a high-level, functional API that abstracts away the low-level complexities of Kafka's Producer and Consumer APIs. Instead of dealing with consumer groups, offsets, and brokers directly, you write code that looks like a data processing pipeline. You can use methods like `filter()`, `map()`, `groupBy()`, and `join()` to define your stream processing logic.

- **Automatic Scalability and Fault Tolerance**:
  - **Scalability**: You can scale a Kafka Streams application by running multiple instances of it. All instances of the same application use the same Kafka consumer group ID. Kafka automatically distributes the topic partitions among the running instances, ensuring that the workload is spread evenly.
  - **Fault Tolerance**: If an instance of your application fails, Kafka detects this and automatically reassigns its partitions to the remaining healthy instances. The new instances can recover the state of the failed instance from the internal Kafka state store topics, ensuring processing can resume without data loss.

### Real-World Analogy

Imagine you are a chef in a busy restaurant. You receive ingredients (data) continuously and in real time.

- Kafka is the kitchen's main pantry and delivery system. All ingredients are stored and transported via this system.
- A traditional stream processing framework (like a separate cluster) is like a large, dedicated food processing factory off-site. You send ingredients there, and they send back the finished products. It's powerful but requires a separate infrastructure.
- Kafka Streams is like having a sophisticated set of food processors and blenders right on your kitchen counter. You can write simple instructions (`filter()`, `map()`) to process ingredients as they arrive. The processors (the application instances) are on your counter, not in a remote factory. If one breaks, you just pull another one from your cupboard, plug it in, and it seamlessly picks up where the old one left off because all the recipes and intermediate ingredients are saved in a backup notebook (the internal Kafka topics).

## 11.2 KStreams vs KTables

### Introduction

In Kafka Streams, the two fundamental data abstractions you'll work with are KStreams and KTables. While both are used to represent data, they model it in different ways and are suitable for different types of processing tasks. Understanding the difference between these two concepts is crucial for building effective stream processing applications.

### Explanation

#### KStream (Stream)

- **Concept**: A KStream represents an unbounded, ever-growing stream of records, where each record is an independent event. Think of it as a continuous log of immutable facts. When a new record arrives, it is processed, and then the application moves on.
- **Data Model**: A KStream models data as a sequence of events. Each record is a new entry with a unique timestamp.
- **Immutability**: Each record in a KStream is an event that "happened." It doesn't modify a previous record; it's just another record in the sequence. For example, if a user clicks a button, that's an event. If they click it again, that's a new, separate event.
- **Operations**: KStreams are ideal for stateless transformations like filtering, mapping, and key-value transformations. They are also used for stateful operations like joining streams or performing windowed aggregations (e.g., counting events in a 5-minute window).
- **Analogy**: A KStream is like a continuous Twitter feed 🐦. Every tweet is a new, separate event. You can read the tweets in order, filter them for certain keywords, or count the number of tweets per hour, but a new tweet doesn't "update" a previous one.

#### KTable (Table)

- **Concept**: A KTable represents a changelog stream, where each record represents an update to a key. It models data as a compact, materialized view of a database table. When a new record with an existing key arrives, it overwrites the previous value for that key.
- **Data Model**: A KTable models data as a table with a primary key. Each record is a key-value pair, and the latest value for a given key is considered the current state.
- **Mutability**: Records in a KTable are mutable from a logical perspective. The last event for a key is the "source of truth." This makes KTables suitable for representing the state of things that change over time, like a user's current account balance or a product's stock level.
- **Operations**: KTables are primarily used for stateful operations, particularly for aggregations (e.g., summing up a user's total purchases) and for joining a stream to a table (e.g., enriching a stream of orders with product information from a product table).
- **Analogy**: A KTable is like a user database 💻. Each record represents a user, and a new record for that user updates their information, such as their email address or current login status. You don't care about their past email addresses, only their current one.

### Summary Table

| Feature                | KStream                                         | KTable                                          |
| ---------------------- | ----------------------------------------------- | ----------------------------------------------- |
| **Concept**            | Event Stream                                    | Changelog / Database Table                      |
| **Data Model**         | A series of immutable events                    | A materialized view with mutable values per key |
| **What it Represents** | "What happened?"                                | "What is the current state?"                    |
| **Example Data**       | Clicks, sensor readings, transactions           | User profiles, stock levels, account balances   |
| **Common Use Case**    | Filtering, stateless transformations, windowing | Aggregations, joins with other streams/tables   |

### KStreams to KTables & Vice Versa

You can easily convert between these two abstractions:

- **Stream to Table**: You can transform a KStream into a KTable using an aggregation. For example, you can take a stream of transactions and group them by user ID to create a KTable that stores the total amount spent by each user.
- **Table to Stream**: You can transform a KTable into a KStream. When a value in the KTable is updated, that update is emitted as a new event in the resulting KStream. This is useful for capturing changes in state and reacting to them.

## 11.3 Windowing, Aggregation, and Joins

### Introduction

Windowing, aggregation, and joins are the core concepts that allow Kafka Streams to perform complex, stateful stream processing. They enable you to analyze data not just as a continuous flow of individual events but as a cohesive series of related records over time.

### Windowing

Windowing is the process of grouping records of a stream into finite, time-based groups for performing stateful operations. Since a stream is unbounded, you can't just group all records forever; you need to define a time window.

- **Tumbling Windows**: These are fixed-size, non-overlapping, and continuous time intervals. Each record belongs to exactly one window. They are ideal for calculating metrics over a regular time period, like counting website clicks every 5 minutes.
- **Hopping Windows**: These are fixed-size, but they overlap. A new window "hops" forward at a specified interval, which is smaller than the window's size. This is useful for calculating moving averages, such as tracking the average temperature over the last 10 minutes, with a new average calculated every minute.
- **Session Windows**: These are not fixed-size. They group records that are separated by a specified "inactivity gap" or "session timeout." If a new record arrives within the gap, it extends the current session. If it arrives after the gap, it starts a new session. This is perfect for analyzing user behavior, where a user's clicks are grouped into a "session."

### Aggregation

Aggregation is a stateful operation that takes a stream of records and combines them into a single result. It involves transforming a KStream into a KTable by reducing a set of events into a single, summary record per key.

- **How it Works**: Aggregations are typically performed on a windowed stream after it has been grouped by a key. Common aggregation functions include `count()`, `sum()`, `min()`, `max()`, and `reduce()`.
- **Example**: You can take a KStream of transactions, group them by user ID, apply a tumbling window of 1 hour, and then use `sum()` to calculate the total amount spent by each user within that hour. The output would be a KTable with the user ID as the key and the total spend as the value.

### Joins

Joins in Kafka Streams are used to combine data from two different streams or tables based on a common key. This is a crucial operation for enriching data. For a join to work, the data in both sources must be co-partitioned, meaning records with the same key are on the same partition in the respective topics. Kafka Streams handles this automatically by re-partitioning the data if needed.

#### Types of Joins

- **Stream-Stream Join (KStream to KStream)**:

  This is a windowed join. It combines two streams based on a shared key and a time window. Since streams are unbounded, Kafka Streams uses an internal state store to buffer records from both streams. A record from the left stream is joined with a record from the right stream if they have the same key and arrive within the defined join window.

  - **Use Case**: Joining a stream of user clicks with a stream of ad impressions to see which ads were clicked. The join window ensures you only link clicks and impressions that happened close to each other in time.

- **Stream-Table Join (KStream to KTable)**:

  This is the most common join type. It is a non-windowed join. It combines an event from a KStream with the latest state of a KTable. This is an extremely efficient operation as it simply performs a lookup in the local state store that backs the KTable.

  - **Use Case**: Enriching a stream of customer transactions (KStream) with up-to-date customer profile information (KTable) to add details like a customer's loyalty status or address to each transaction.

- **Table-Table Join (KTable to KTable)**:

  This type of join combines two changelog streams (KTables). The resulting KTable is updated whenever a change occurs in either of the two input tables.

  - **Use Case**: Joining a table of product information with a table of inventory counts to produce a final table that has a complete, up-to-date view of all products and their stock levels.

### Analogy: The Assembly Line

Imagine an assembly line where packages (records) move on two different conveyor belts (topics).

- **Windowing** is like a worker who grabs all packages that arrive in a specific 5-minute interval to put them into a single crate.
- **Aggregation** is like that same worker counting all the items in that crate to produce a summary sheet with a total count.
- **Joins** are the most complex.
  - **Stream-Stream Join**: Imagine a package on the left conveyor belt that needs a label from a package on the right conveyor belt. A worker at a station waits for both packages to arrive within a 10-second window before combining them.
  - **Stream-Table Join**: This is much more efficient. One conveyor belt carries new packages (the KStream), and there's a big reference manual on a table next to it (the KTable). As each new package arrives, the worker looks up its details in the manual and writes the information on the package. The worker doesn't need to wait for a matching package on another belt; they just need to look up the current state in the manual. This is why stream-table joins are so common and fast.

## 11.4 Stream Processing in .NET (Streamiz.Kafka.Net)

### Introduction

While Kafka Streams is a Java library, the .NET ecosystem has a powerful and mature client for building stream processing applications: Streamiz.Kafka.Net. This library is a port of the official Kafka Streams API, bringing its high-level, declarative, and stateful processing capabilities to .NET developers. It allows you to build microservices that can filter, transform, aggregate, and join data streams with the same elegant API as its Java counterpart.

### Key Concepts

Streamiz.Kafka.Net mirrors the core concepts of Kafka Streams, providing a familiar API for developers.

- **Topology**: The heart of a Streamiz.Kafka.Net application is a Topology object. This object defines your stream processing graph, or the flow of data from input topics to output topics. You use a StreamBuilder to construct this topology by chaining together methods like Stream, Filter, Map, GroupBy, Join, and To.
- **KStream and KTable**: Just like in the Java version, Streamiz.Kafka.Net provides the KStream and KTable abstractions.
  - **KStream<TKey, TValue>**: Represents an unbounded stream of events. Used for stateless transformations and windowed stateful operations.
  - **IKTable<TKey, TValue>**: Represents a changelog stream, modeling a materialized view or the current state of data. Used for aggregations and joins.
- **State Stores**: For stateful operations like aggregations and joins, Streamiz.Kafka.Net uses local state stores. These are materialized views of your data that are managed on the application's local machine. The library automatically handles the fault tolerance of these state stores by backing them up to internal Kafka topics. If your application instance fails, another instance can recover its state and resume processing.

### Building a Simple Stream Processing App in .NET

Here is a simple example of a Streamiz.Kafka.Net application written in C#. This application will:

1. Read a stream of sensor readings from a topic.
2. Filter out any readings below a certain threshold.
3. Group the remaining readings by the sensor ID.
4. Count the number of valid readings for each sensor in a 1-minute tumbling window.
5. Publish the results to a new topic.

```csharp
using Streamiz.Kafka.Net;
using Streamiz.Kafka.Net.SerDes;
using Streamiz.Kafka.Net.Stream;
using Streamiz.Kafka.Net.Table;
using System.Threading.Tasks;

public class Program
{
    public static async Task Main(string[] args)
    {
        // 1. Configure the Streamiz application
        var config = new StreamConfig<StringSerDes, StringSerDes>();
        config.ApplicationId = "sensor-data-processor-app";
        config.BootstrapServers = "localhost:9092";
        config.AutoOffsetReset = AutoOffsetReset.Earliest;

        // 2. Build the stream processing topology
        var builder = new StreamBuilder();
        builder.Stream<string, string>("sensor-readings-topic")
               // Step A: Filter out readings below 10
               .Filter((key, value) => double.Parse(value) >= 10.0)
               // Step B: Group by the sensor ID
               .GroupByKey()
               // Step C: Count the readings in a 1-minute tumbling window
               .WindowedBy(TumblingWindowOptions.Of(TimeSpan.FromMinutes(1)))
               .Count(InMemoryWindows<string, long>.As("sensor-counts"))
               // Step D: Convert the result back to a stream and send it to a new topic
               .ToStream((key, value) => $"{key.Key}_{key.Window.Start}_{key.Window.End}", (key, value) => value.ToString())
               .To("sensor-counts-topic", new StringSerDes(), new StringSerDes());

        // 3. Create and start the Streamiz application
        var topology = builder.Build();
        var stream = new KafkaStream(topology, config);

        // Start the stream processing and wait for a cancellation signal
        await stream.StartAsync();
    }
}
```

This code snippet demonstrates how the fluent API allows you to define a complex data pipeline in a clear, declarative way. Streamiz.Kafka.Net handles all the underlying complexities of Kafka, including consumer groups, offset management, and fault-tolerant state management.

### Analogy: The Automated Report Generator

Think of a small .NET microservice that uses Streamiz.Kafka.Net as a robotic assistant in a mailroom.

- The mailroom is your Kafka cluster.
- The incoming mailbags are your input topics (sensor-readings-topic).
- The robotic assistant is your Streamiz.Kafka.Net application instance.
- The assistant is programmed with a flowchart (your Topology). The flowchart says: "First, throw away any junk mail (Filter). Then, sort the remaining mail into bins by sender (GroupBy). Every hour, count the number of letters in each bin (WindowedBy/Count). Finally, package up a summary report for each bin and send it to the 'Reports' mailbox (To)."

The bins are the local state stores that the assistant uses to hold the mail temporarily before counting it. The assistant has a backup logbook (the internal Kafka topics) to remember the exact state of each bin, so if it ever breaks down, a new assistant can instantly take over and continue the job without losing a single letter.

## 11.5 Introduction to ksqlDB

### Introduction

ksqlDB is an open-source, distributed stream processing database for Apache Kafka. It allows you to write real-time stream processing applications using a familiar SQL-like syntax, which is a powerful alternative to writing code in a programming language like Java or C#. Under the hood, ksqlDB is built on top of the Kafka Streams library, which means it inherits all the benefits of scalability, fault tolerance, and performance.

### How ksqlDB Works

ksqlDB operates as a separate server or cluster that connects to your existing Kafka cluster. It acts as a database for streams of events, providing a simplified interface for developers and data analysts who may not be familiar with traditional programming.

- **SQL-Based Interface**: The most significant feature of ksqlDB is its declarative SQL syntax. You can use standard SQL commands like `CREATE STREAM`, `CREATE TABLE`, `SELECT`, and `JOIN` to define your data pipelines. This makes it accessible to a much broader audience, including data analysts and business intelligence professionals.
- **Streams and Tables**: ksqlDB uses the same core abstractions as Kafka Streams:
  - **Streams**: These are similar to Kafka Streams' KStream and represent an unbounded, append-only series of events.
  - **Tables**: These are similar to Kafka Streams' KTable and represent a materialized view of a stream's latest state, like a database table with an updating primary key.
- You can use DDL (Data Definition Language) statements to define schemas for your Kafka topics as either streams or tables. For example:

  ```sql
  -- Create a stream over a Kafka topic
  CREATE STREAM user_activity (user_id INT, action VARCHAR, timestamp BIGINT)
  WITH (kafka_topic='user-activity-topic', value_format='json', partitions=1);

  -- Create a table from the stream to store the latest action for each user
  CREATE TABLE latest_user_action AS
  SELECT user_id, LATEST_BY_OFFSET(action) AS latest_action
  FROM user_activity
  GROUP BY user_id;
  ```

- **Continuous Queries**: ksqlDB queries are not one-off, point-in-time queries like in a traditional database. They are continuous queries that run indefinitely. As new data arrives in the source topic, the query processes it and pushes updates to the result stream or table in real time.
- **Push and Pull Queries**: ksqlDB supports two types of queries to retrieve data:
  - **Push Queries**: These are continuous queries that push results to the client as they are computed. This is ideal for building real-time dashboards or notifications.
  - **Pull Queries**: These are similar to traditional database queries. They fetch the current state of a materialized view (a TABLE) at a specific point in time and then terminate. This is great for an application that needs to look up a single value on demand.

### ksqlDB vs. Kafka Streams

Choosing between ksqlDB and Kafka Streams is a matter of trading flexibility for ease of use.

| Feature         | ksqlDB                                                     | Kafka Streams                                                                           |
| --------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Interface**   | SQL-based declarative syntax                               | Java/Scala-based library API                                                            |
| **Ease of Use** | Very easy to learn and use for SQL experts.                | Requires knowledge of Java/Scala.                                                       |
| **Flexibility** | Less flexible. Limited to what the SQL syntax can express. | Highly flexible. Can implement any custom logic.                                        |
| **Use Cases**   | Stream ETL, real-time analytics, materialized views.       | Complex business logic, custom stateful operations, integrations with external systems. |
| **Development** | Minimal coding required. Fast to prototype.                | Full-fledged application development.                                                   |

### Analogy: The Automated Robot Assistant

If Kafka Streams is a powerful, customizable robot arm that you have to program with code, then ksqlDB is a user-friendly control panel for that robot arm.

- The Kafka Streams library is the code that controls the robot arm's every movement and function. You can program it to do anything you can imagine, but it requires deep technical knowledge.
- The ksqlDB control panel is the simplified SQL interface. You can just type commands like "Move box A to shelf B," and the control panel translates that into the complex code the robot arm needs. You can do a lot of common tasks very easily and quickly, but you can't program a completely custom, novel action that isn't supported by the control panel's commands.
