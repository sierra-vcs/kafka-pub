# Chapter 9: Kafka Connect – Data Integration

- [9.1 Kafka Connect Architecture](#91-kafka-connect-architecture)
- [9.2 Installing Kafka Connect](#92-installing-kafka-connect)
- [9.3 Source vs Sink Connectors](#93-source-vs-sink-connectors)
- [9.4 Configuring Connectors (e.g., JDBC, S3, Elasticsearch)](#94-configuring-connectors-eg-jdbc-s3-elasticsearch)
- [9.5 Running Standalone vs Distributed Mode](#95-running-standalone-vs-distributed-mode)
- [9.6 Custom Connector Overview](#96-custom-connector-overview)

## 9.1 Kafka Connect Architecture

### Introduction

Kafka Connect is a crucial component of the Apache Kafka ecosystem, designed to simplify and automate the process of moving data between Kafka and other systems. It acts as a scalable and reliable framework for streaming data from external sources into Kafka (Source connectors) and from Kafka to external destinations (Sink connectors). Instead of writing custom, low-level code to interact with Kafka's Producer and Consumer APIs, you can use Kafka Connect's pre-built or custom-developed connectors to handle the data integration pipeline.

### Explanation

The architecture of Kafka Connect is built around a few key components that work together to manage the data flow:

- **Connectors**: These are the heart of Kafka Connect. A connector is a high-level abstraction that defines how data is moved. It's responsible for the overall data-copying task. There are two main types:

  - **Source Connectors**: These ingest data from an external system (like a database, file system, or message queue) and write it to a Kafka topic.
  - **Sink Connectors**: These export data from a Kafka topic to an external system (like a data warehouse, search index, or another database).

- **Tasks**: A connector doesn't do the actual work of moving data itself. Instead, it breaks the work down into a set of smaller, parallelizable units called tasks. A single connector instance can have multiple tasks running simultaneously to scale the data movement. This allows Kafka Connect to distribute the workload across multiple machines (workers) for fault tolerance and high throughput.

- **Workers**: These are the runtime processes where the connectors and tasks execute. A worker is a single JVM process that hosts the tasks. There are two modes of operation for workers:

  - **Standalone Mode**: A single worker runs all connectors and their tasks on a single machine. This is ideal for development, testing, and smaller-scale, non-critical deployments.
  - **Distributed Mode**: Multiple workers run on different machines and form a cluster. They automatically coordinate, distribute tasks, and handle fault tolerance. If a worker fails, its tasks are automatically reassigned to other healthy workers. This is the recommended mode for production environments.

- **Converters**: Data in Kafka is stored as bytes. Converters are responsible for translating data between the format used by the external system (e.g., JSON, Avro, Protobuf) and the internal byte format used by Kafka Connect. They are configured per connector and handle both key and value data. For example, a `JsonConverter` would serialize structured data into JSON bytes and deserialize JSON bytes back into structured data.

- **Transforms (Single Message Transforms - SMTs)**: These are optional, lightweight transformations that can modify, filter, or route individual messages as they pass through the connector. For instance, you could use an SMT to add a new field to a message, rename an existing field, or filter out messages that don't meet certain criteria. They are often used to make small, simple changes without needing to build a custom connector.

## 9.2 Installing Kafka Connect

### Introduction

Installing Kafka Connect isn't a separate process from installing Apache Kafka itself. Kafka Connect is included as part of the Apache Kafka distribution. The installation process primarily involves setting up the Kafka environment and then configuring the Connect workers and the connectors you want to use.

### Explanation

The installation process can be broken down into these key steps:

1. **Download and Extract Apache Kafka**:

   - First, download the official Apache Kafka binary distribution. You can find the latest version on the Apache Kafka website. Once downloaded, extract the archive to a directory of your choice. This directory will be referred to as `$KAFKA_HOME` in this documentation.

2. **Ensure Java is Installed**:

   - Kafka and Kafka Connect are JVM-based applications, so you must have a compatible version of Java installed on your machine. Apache Kafka requires a recent version of the Java Development Kit (JDK), typically Java 11 or later. You can check your Java version by running `java -version` in your terminal.

3. **Start Kafka and Zookeeper (or use KRaft mode)**:

   - Before you can run Kafka Connect, you need a running Kafka cluster. Since Kafka 2.8, you can run Kafka in KRaft (Kafka Raft) mode, which removes the dependency on ZooKeeper. For older versions or if you prefer, you would need to start ZooKeeper first and then the Kafka brokers.

   - **With ZooKeeper (older versions)**:

     ```bash
     $KAFKA_HOME/bin/zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties
     $KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties
     ```

   - **With KRaft mode (newer versions)**:
     ```bash
     $KAFKA_HOME/bin/kafka-storage.sh random-uuid
     $KAFKA_HOME/bin/kafka-storage.sh format -t <your_cluster_id> -c $KAFKA_HOME/config/kraft/server.properties
     $KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/kraft/server.properties
     ```

4. **Install Connector Plugins**:

   - The core functionality of Kafka Connect comes from its plugins, which are the connectors, converters, and transforms. These plugins are typically packaged as JAR files. You install a connector by:
     - Downloading the connector's JAR files (e.g., from Confluent Hub or Maven Central).
     - Placing the JAR files into a designated directory on your system.
     - Configuring the `plugin.path` property in your Kafka Connect worker configuration file to point to this directory. The worker will automatically discover and load the plugins from this path when it starts. This is a critical step for making connectors available to your Kafka Connect instance.

5. **Configure and Run Kafka Connect**:
   - Now you can configure and start your Kafka Connect worker. You'll use a properties file to configure the worker.
     - **Standalone Mode**: Use the `connect-standalone.properties` file. This is great for local development and testing. You specify the Kafka brokers, converters, and the file where offsets are stored.
       ```bash
       $KAFKA_HOME/bin/connect-standalone.sh $KAFKA_HOME/config/connect-standalone.properties <your_connector_config.properties>
       ```
     - **Distributed Mode**: Use the `connect-distributed.properties` file. This is for production, as it enables a fault-tolerant cluster. You configure topics for storing connector configurations, offsets, and status information.
       ```bash
       $KAFKA_HOME/bin/connect-distributed.sh $KAFKA_HOME/config/connect-distributed.properties
       ```

### Code Example (Configuration File Snippet)

While the main installation process involves bash scripts, the configuration is defined in property files. Here is a simplified example of what a `connect-standalone.properties` file might look like, which is crucial for making the installation work.

```properties
# A list of Kafka brokers used to connect to the Kafka cluster
bootstrap.servers=localhost:9092

# The converters to use for keys and values
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter

# Configure the location where connector plugins are stored
plugin.path=/usr/local/share/kafka/plugins

# The file to store offsets. This is for standalone mode only
offset.storage.file.filename=/tmp/connect.offsets
```

### Pros and Cons of Manual Installation

- **Pros**:

  - **Full Control**: You have complete control over the versions of Kafka and its components, as well as the exact configuration.
  - **Customization**: It's easy to add custom connector plugins and fine-tune settings for your specific environment.

- **Cons**:
  - **Complexity**: The process can be more involved and error-prone, especially for beginners.
  - **Maintenance**: You are responsible for managing all the components (Kafka, Zookeeper, Connect workers), which can be complex in a production environment.
  - **Lack of Automation**: You need to manually run scripts and manage configuration files. For production, you would typically use a containerized approach (like Docker or Kubernetes) to automate this.

[How To Run Kafka Connect in Standalone and Distributed Mode Examples ](https://www.youtube.com/watch?v=OZ4Yne_rK3M)

## 9.3 Source vs Sink Connectors

### Introduction

In the Kafka Connect framework, data movement is handled by specialized components called connectors. These connectors are fundamentally categorized into two types based on the direction of data flow: Source Connectors and Sink Connectors. Understanding this distinction is crucial to building effective data pipelines.

### Explanation

The core difference between Source and Sink connectors lies in their role in the data flow relative to Apache Kafka.

1. **Source Connectors**

   A Source Connector is responsible for ingesting data from an external system and publishing it to one or more Kafka topics. Its purpose is to get data into Kafka. It acts as a data producer, continuously monitoring a source system for new or changed data and streaming that data into Kafka.

   **How it works**:

   - A Source Connector defines the logic to read data from a specific source. For example, a JDBC Source Connector might poll a relational database table for new rows.
   - The connector breaks the work into smaller tasks. Each task is an independent unit that fetches data from the source.
   - The data read by the tasks is then converted into a Kafka-compatible format (using a Converter) and published as messages to a specified Kafka topic.
   - Source connectors typically use an offset mechanism to keep track of what data has already been read from the source. This ensures that in case of a failure, the connector can resume from where it left off without duplicating data.

   **Example Use Cases**:

   - Reading log files from a directory and streaming new log entries to a Kafka topic.
   - Polling a database table for new records (e.g., new user signups) and sending them to Kafka.
   - Capturing change data from a database (using techniques like CDC) and streaming it into Kafka for real-time analytics.

2. **Sink Connectors**

   A Sink Connector is the opposite of a Source Connector. It consumes data from one or more Kafka topics and writes it to an external system. Its purpose is to get data out of Kafka. It acts as a data consumer, subscribing to topics and writing the message content to a destination system.

   **How it works**:

   - A Sink Connector subscribes to a list of Kafka topics.
   - The connector creates tasks that consume messages from the assigned partitions of those topics.
   - The messages are deserialized from Kafka's byte format back into a structured format using a Converter.
   - The tasks then write the data to the destination system. For example, an Elasticsearch Sink Connector would index the messages into an Elasticsearch cluster.
   - Similar to the producer-consumer model, Sink Connectors manage consumer group offsets to track which messages have been successfully processed and written to the destination.

   **Example Use Cases**:

   - Writing messages from a Kafka topic to a file on HDFS or S3 for long-term storage.
   - Indexing events from a Kafka topic into an Elasticsearch cluster for search and analytics.
   - Loading data from Kafka into a data warehouse like Snowflake or a relational database for business intelligence.

### Summary Table

| Feature       | Source Connector                                       | Sink Connector                                                |
| ------------- | ------------------------------------------------------ | ------------------------------------------------------------- |
| Data Flow     | From an external system to Kafka                       | From Kafka to an external system                              |
| Role          | Acts as a Producer                                     | Acts as a Consumer                                            |
| Core Function | Ingests data from a source                             | Exports data to a destination                                 |
| Tracking      | Uses offsets to track data read from the source        | Uses consumer group offsets to track data consumed from Kafka |
| Analogy       | A delivery driver picking up packages from a warehouse | A postman delivering mail to various houses                   |

### Real-World Analogy

Imagine a company's internal communication system (Kafka) and its interactions with the outside world.

- A Source Connector is like an automated email scanner. It constantly monitors a shared email inbox (the external system) for new emails (data). When a new email arrives, it automatically pulls the content, categorizes it, and posts it on the company's internal message board (Kafka topic). Its job is to bring information into the company's central system.

- A Sink Connector is like an automated report generator. It constantly monitors specific channels on the internal message board (Kafka topics) for new posts. When it sees a post that needs to be archived or shared externally, it automatically takes the information and writes it into a formal company report (the external system, like a PDF file or a database). Its job is to move information out of the company's central system.

## 9.4 Configuring Connectors (e.g., JDBC, S3, Elasticsearch)

### Introduction

Configuring a Kafka Connect connector is the process of defining its behavior, specifying its connection details, and setting up the data pipeline. This is typically done through a configuration file (often in .properties or .json format) that you provide to the Kafka Connect worker. The key to effective configuration is understanding the common properties that apply to most connectors, as well as the specific properties required by each connector type.

### Explanation

All connectors have some common properties that are mandatory, and then a set of connector-specific properties.

#### Common Properties

- `name`: A unique name for the connector instance. This is used by Kafka Connect's REST API and for logging.
- `connector.class`: The fully qualified class name of the connector to be used (e.g., `io.confluent.connect.jdbc.JdbcSourceConnector`).
- `tasks.max`: The maximum number of tasks to create for this connector. Kafka Connect will try to run as many tasks as possible up to this limit to parallelize the work.
- `topics` (for Sink Connectors): A comma-separated list of Kafka topics to consume from.
- `topic.prefix` (for Source Connectors): A prefix to add to the topic names that the connector will create.

Now let's look at specific configuration examples for popular connectors.

#### JDBC Source Connector

This connector reads data from a relational database using the Java Database Connectivity (JDBC) API and streams it to Kafka.

**Key Configuration Properties**:

- `connection.url`: The JDBC connection string to the database. For a SQL Server database, this might look like `jdbc:sqlserver://localhost:1433;databaseName=MyDatabase`.
- `connection.user` & `connection.password`: The credentials for the database connection.
- `table.whitelist`: A comma-separated list of tables to include for ingestion. You can also use `table.blacklist` to exclude tables.
- `mode`: The data capture mode. This is a critical setting.
  - `bulk`: Performs a full-table copy every time it runs.
  - `incrementing`: Uses a strictly incrementing column (e.g., an auto-incrementing ID) to find new rows. This won't detect updates or deletions.
  - `timestamp`: Uses a timestamp column to detect new and modified rows.
  - `timestamp+incrementing`: The most robust option, using both a timestamp and an incrementing column to detect all changes.
- `topic.prefix`: The topic name will be `topic.prefix + table_name`.

**Example .properties File for a SQL Server Source Connector**:

```properties
name=jdbc-source-connector
connector.class=io.confluent.connect.jdbc.JdbcSourceConnector
tasks.max=1

# Database connection details
connection.url=jdbc:sqlserver://localhost:1433;databaseName=MyApp
connection.user=sa
connection.password=Password123

# Data capture mode
mode=timestamp+incrementing
incrementing.column.name=id
timestamp.column.name=last_modified_date

# Tables to track and topic naming
table.whitelist=users,products
topic.prefix=my-app-db-
```

#### S3 Sink Connector

This connector reads data from Kafka topics and writes it to files in an Amazon S3 bucket.

**Key Configuration Properties**:

- `s3.region`: The AWS region of the S3 bucket (e.g., `us-east-1`).
- `s3.bucket.name`: The name of the S3 bucket where data will be stored.
- `topics`: The Kafka topics to read from.
- `flush.size`: The number of records that should be written to a single S3 file before closing it and starting a new one. This is a key performance setting.
- `format.class`: The format of the output files (e.g., `io.confluent.connect.s3.format.json.JsonFormat` for JSON).
- `storage.class`: The storage mechanism, which is `io.confluent.connect.s3.storage.S3Storage` for S3.

**Example .properties File for an S3 Sink Connector**:

```properties
name=s3-sink-connector
connector.class=io.confluent.connect.s3.S3SinkConnector
tasks.max=1

# Kafka and S3 details
topics=user-events,product-updates
s3.region=us-east-1
s3.bucket.name=my-company-data-lake

# Data format and file flushing
flush.size=10000
format.class=io.confluent.connect.s3.format.json.JsonFormat
storage.class=io.confluent.connect.s3.storage.S3Storage
```

#### Elasticsearch Sink Connector

This connector takes data from Kafka topics and indexes it into an Elasticsearch cluster.

**Key Configuration Properties**:

- `connection.url`: The URL of the Elasticsearch instance (e.g., `http://localhost:9200`).
- `topics`: A comma-separated list of Kafka topics to index.
- `topics.regex`: You can also use a regular expression to match topics (e.g., `topics.regex=^my-app-logs-.*`).
- `key.ignore`: Set to true if you don't want the message key to be used in Elasticsearch.
- `schema.ignore`: Set to true if you are using schemaless JSON data and don't want the connector to fail on a missing schema.
- `type.name`: The type name to use for the indexed documents (for older Elasticsearch versions).

**Example .json File for an Elasticsearch Sink Connector**:

```json
{
  "name": "elasticsearch-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": 1,
    "topics": "user-profiles",
    "key.ignore": "true",
    "schema.ignore": "true",
    "connection.url": "http://localhost:9200",
    "type.name": "user_profile"
  }
}
```

### Real-World Analogy

Configuring a Kafka connector is like programming a smart home automation device.

- The common properties (`name`, `connector.class`, `tasks.max`) are like the basic setup: giving the device a name ("Living Room Lights"), telling it what kind of device it is ("Smart Light Switch"), and specifying how many individual lights it can control (`tasks.max`).
- The connector-specific properties are the detailed rules you set. For a JDBC Source Connector, this is telling it which database to connect to, what your password is, and the specific tables to monitor. For a S3 Sink Connector, you're telling it which S3 bucket to use and how many messages to bundle into a single file before uploading.

## 9.5 Running Standalone vs Distributed Mode

### Introduction

Kafka Connect offers two primary modes of operation for its workers: Standalone and Distributed. The choice between these modes is critical and depends entirely on your use case, operational requirements, and the scale of your data pipelines. Standalone is for single-machine, simple use cases, while Distributed is for production-grade, fault-tolerant, and scalable environments.

### Explanation

#### Standalone Mode

In standalone mode, a single Kafka Connect worker process runs all connectors and their tasks. This is the simplest way to run Kafka Connect.

**Characteristics**:

- **Single Process**: A single Java Virtual Machine (JVM) process. If this process fails, all connectors and tasks running on it will stop.
- **Configuration**: All connector configurations are defined in .properties files, and you start the worker by passing the worker configuration file and each connector's configuration file as arguments to the startup script.
- **State Management**: All state, including connector offsets (the last message processed), is stored locally on the file system. This means if the machine crashes, you might lose the most recent offsets, leading to data re-processing.
- **No Fault Tolerance**: There is no built-in mechanism to restart a failed connector or rebalance tasks to another worker. If the worker goes down, it stays down.
- **No Scalability**: You cannot scale out by adding more workers to handle more tasks. All work is confined to a single machine.

**Use Cases**:

- **Development and Testing**: This is the most common use case. It is simple to set up on a local machine to test a connector or a small data pipeline.
- **Lightweight, Single-Agent Environments**: For scenarios where you need to run a simple, non-critical connector on a single machine, such as a log file shipper running on an edge server.

#### Distributed Mode

In distributed mode, multiple Kafka Connect workers form a cluster. This is the recommended mode for production environments.

**Characteristics**:

- **Clustered Processes**: Multiple worker processes run on different machines and coordinate with each other. They use Kafka itself to store and manage the state of the cluster, including connector configurations, task assignments, and offsets.
- **Configuration**: You first start the distributed worker processes with a generic configuration file. Then, you manage and configure individual connectors and tasks using Kafka Connect's REST API. This allows for dynamic, on-the-fly changes without restarting the worker processes.
- **Fault Tolerance**: If a worker in the cluster fails, the remaining workers automatically rebalance the tasks that were running on the failed worker and redistribute them among themselves. This ensures the data pipeline continues to run with minimal interruption.
- **Scalability**: You can scale out the cluster simply by adding new worker nodes. The cluster will automatically detect the new worker and rebalance tasks to utilize the new capacity. Similarly, you can scale in by removing workers.
- **Shared State**: The state of all connectors and tasks is stored in Kafka topics. This provides a highly available and fault-tolerant storage mechanism, ensuring no state is lost if a worker fails.

**Use Cases**:

- **Production Deployments**: This is the standard for any production data pipeline that requires high availability, scalability, and resilience.
- **Large-Scale Data Integration**: When you have a high volume of data to move or a large number of connectors to manage, distributed mode is the only practical option.

### Summary Comparison

| Feature          | Standalone Mode                        | Distributed Mode                                     |
| ---------------- | -------------------------------------- | ---------------------------------------------------- |
| Processes        | One worker process                     | Multiple worker processes in a cluster               |
| Configuration    | File-based (.properties files)         | REST API-based                                       |
| State Storage    | Local file system                      | Kafka topics (highly available)                      |
| Fault Tolerance  | None                                   | High (tasks are automatically rebalanced on failure) |
| Scalability      | Not scalable                           | Horizontally scalable                                |
| Primary Use Case | Development, testing, simple use cases | Production, high-volume data pipelines               |

### Real-World Analogy

Think of the difference as managing a small, local catering business versus a large, national food delivery service.

- Standalone Mode is the local caterer. All tasks—taking orders, cooking, and delivering—are handled by a single person (the worker). It's simple and easy to start, perfect for a small party (a simple data pipeline). But if that one person gets sick or their car breaks down, the entire operation shuts down (no fault tolerance). You can't hire more people to help with a single event (no scalability).

- Distributed Mode is the national food delivery service. Orders are managed by a central system (Kafka), and they are assigned to a fleet of delivery drivers (the workers). If one driver's car breaks down, the central system automatically reassigns their deliveries to other available drivers (fault tolerance). If the number of orders spikes, the service can easily hire more drivers to handle the load (scalability). All the delivery status information is tracked in a central database (Kafka) for reliability.

## 9.6 Custom Connector Overview

### Introduction

While Kafka Connect provides a rich set of pre-built connectors for popular data systems, there are times when you need to integrate with a unique or proprietary system. In such cases, the Kafka Connect framework allows you to develop your own custom connectors. Writing a custom connector involves implementing the core Kafka Connect APIs to define how data is read from or written to your specific system.

### Explanation

The core principle of a custom connector is to adhere to the Kafka Connect API, which is primarily a Java-based framework. This means that a custom connector is a Java application packaged as a JAR file that implements specific interfaces provided by the Kafka Connect API.

A custom connector typically consists of two main parts, just like the built-in connectors: a Connector class and a Task class.

#### The Connector Class

This is the high-level class that defines the data transfer job. It’s responsible for:

- **Parsing configuration**: It reads the properties provided by the user in the connector configuration file.
- **Defining ConfigDef**: It specifies the configuration properties it expects from the user and whether they are required or optional.
- **Breaking the job into tasks**: Based on the configuration and the source/sink system, it determines how to partition the work and creates a list of Task configurations. For a source connector, this might involve assigning different database tables or log files to different tasks. For a sink connector, it might involve assigning a set of Kafka topic partitions to each task.
- **Lifecycle methods**: It has `start()` and `stop()` methods to handle the connector's lifecycle.

#### The Task Class

This is where the actual data movement logic resides. A Task is an independent unit of work that runs on a Connect worker.

- **Data transfer logic**: It contains the code to read from (for a source task) or write to (for a sink task) the external system. For example, a custom source task would use a client library to connect to the external system and read data.
- **Offset management**: For a source task, it must keep track of its read position (offset) in the external system. This allows it to resume from where it left off in case of failure.
- **Conversion**: It uses the configured Converter to transform data into the Kafka Connect format (for source tasks) or from the Kafka Connect format (for sink tasks).

### The Challenge for .NET Developers

A key point for .NET developers is that there is no official Kafka Connect API for .NET. The framework is tightly integrated with the Java ecosystem. While you can write producers and consumers in .NET using libraries like Confluent.Kafka, you cannot write a Kafka Connect connector itself in C#. The options for a .NET developer needing a custom data integration are:

- **Develop the Connector in Java**: The most common and direct approach is to have a Java developer create the connector. This is the recommended path as it leverages the native Kafka Connect framework.
- **Use a different approach**: If developing in Java is not an option, you would need to create a standalone .NET application that uses the Confluent.Kafka client library. This application would act as a custom producer or consumer, and you would be responsible for all the logic that Kafka Connect provides out-of-the-box, such as:

  - Offset management and fault tolerance.
  - Error handling.
  - Scalability and distribution (e.g., managing multiple instances of your .NET app).
  - Configuration management.

  This method is viable for simple tasks but is more complex and less robust than a native Kafka Connect solution for production environments.

### Real-World Analogy

Building a custom Kafka Connect connector is like designing a new type of power adapter for a global electrical system.

- The Kafka Connect framework is the standardized electrical outlet found in every country.
- A built-in connector is a standard plug that fits one of these outlets (e.g., a US plug for a US outlet).
- A custom connector is a new, specialized plug you've designed to connect a unique device (your external system) to the standard outlet. The Kafka Connect API is the blueprint for how this plug must be constructed to fit into the outlet and carry power correctly.

The .NET challenge is like a manufacturer telling you the only tools to build the new plug are in a specific metric-based toolkit (Java), and your American-standard toolkit (.NET) won't work. While you could build your own separate power supply for your device, it's a lot more work and won't have the benefits of the standardized, plug-and-play system.
