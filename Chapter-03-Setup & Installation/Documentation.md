# Chapter 3: Kafka Setup & Installation (Dev Environment)

- [3.1 Installing Kafka on Windows (Manual)](#31-installing-kafka-on-windows-manual)
- [3.2 Running Kafka with KRaft Mode](#32-running-kafka-with-kraft-mode)
- [3.3 Installing Kafka UI Tools (AKHQ)](#33-installing-kafka-ui-tools-akhq)
- [3.4 Basic Kafka Configuration](#34-basic-kafka-configuration)
- [3.5 Validating Installation with CLI Tools](#35-validating-installation-with-cli-tools)

---

## 3.1 Installing Kafka on Windows (Manual)

### Introduction

Setting up a Kafka development environment on Windows involves downloading the binary distribution and running the provided scripts from the command line. This method is suitable for a simple, single-broker setup.

### Explanation: Manual Installation Steps

**Prerequisites:**

*   **Java:** Ensure you have a Java Development Kit (JDK), version 17 or newer, installed.
*   **Environment Variables:** Add the JDK's `bin` directory to your system's `Path` environment variable. You can verify this by running `java -version` in Command Prompt.

**Download Kafka:**

1.  Navigate to the [Apache Kafka downloads page](https://kafka.apache.org/downloads).
2.  Download the binary distribution for the latest version. Look for a file named `kafka_2.13-4.0.0.tgz` or similar.

**Extract the Archive:**

1.  Use a tool like 7-Zip or WinRAR to extract the `.tgz` archive.
2.  Place the extracted folder in a convenient location, such as `C:\kafka_4.0.0`. This directory will be your Kafka home.

**Legacy ZooKeeper Setup (for older Kafka versions):**

> **Note:** Kafka 3.0+ recommends using KRaft mode, which is covered in section 3.2. This section is for context on the older setup.

In two separate command prompt windows, navigate to your Kafka home directory.

**Window 1 (ZooKeeper):** Start the ZooKeeper server.
```bash
C:\kafka_4.0.0\> .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
```

**Window 2 (Kafka Broker):** Start the Kafka broker.
```bash
C:\kafka_4.0.0\> .\bin\windows\kafka-server-start.bat .\config\server.properties
```

---

## 3.2 Running Kafka with KRaft Mode

### Introduction

KRaft (Kafka Raft) mode is the modern, ZooKeeper-less architecture for Kafka. It is the recommended setup for all new deployments, including your local development environment.

### Explanation: Detailed KRaft Setup on Windows

This process consolidates the setup steps into a single, cohesive guide.

**1. Generate a Cluster ID:**

Open a command prompt and navigate to your Kafka directory.
```bash
C:\kafka_4.0.0\> .\bin\windows\kafka-storage.bat random-uuid
```
The output is a unique ID (e.g., `M1v6D9P3F2x5L7Y0C8Q1E3J4I2V5S4`). Copy this ID.

**2. Format the Storage Directory:**

Use the generated ID to format Kafka's data directory. This initializes the storage for the new cluster.
```bash
C:\kafka_4.0.0\> .\bin\windows\kafka-storage.bat format -t <your_cluster_id> -c .\config\kraft\server.properties
```
This command must be run only once per cluster.

**3. Start the Kafka Broker:**

Run the server start script, pointing it to the KRaft configuration file.
```bash
C:\kafka_4.0.0\> .\bin\windows\kafka-server-start.bat .\config\kraft\server.properties
```
This command will run the broker and controller roles in a single process. You will see a stream of log messages indicating a successful startup.

---

## 3.3 Installing Kafka UI Tools (AKHQ)

### Introduction

For a development environment, a UI tool provides a powerful way to manage your Kafka cluster, topics, and consumer groups visually. AKHQ is a popular and effective open-source tool for this purpose.

There are two primary methods for setting up AKHQ: using Docker (recommended for its simplicity and isolation) or running it as a standalone application.

### Method 1: Using Docker (Recommended)

This method describes how to run AKHQ as a Docker container and connect it to a Kafka broker that is running on your Windows host machine.

**Prerequisites:**

*   **Docker Desktop:** Ensure Docker Desktop is installed and running on your Windows machine.
*   **Local Kafka Broker:** You must have a Kafka broker already running on your machine, as described in sections 3.1 and 3.4, and listening on `localhost:9092`.

**`docker-compose.yml` Configuration:**

Create a `docker-compose.yml` file in a project folder. This file will define only the AKHQ service.

The `KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS` environment variable is configured to use `host.docker.internal:9092`. This is a special DNS name provided by Docker that allows containers to connect to services running on the host machine.
```yaml
version: '3.7'
services:
  akhq:
    image: tchiotludo/akhq
    environment:
      - KAFKA_CLUSTERS_0_NAME=local-kafka
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=host.docker.internal:9092
    ports:
      - "8080:8080"
```

**Start the UI Tool:**

Open a command prompt in the directory containing the `docker-compose.yml` file.

Run the following command to start the AKHQ container.
```bash
docker-compose up -d
```

**Access AKHQ:**

Open your web browser and navigate to `http://localhost:8080`.

### Method 2: Manual Installation (Without Docker)

This method involves downloading the AKHQ binary and running it as a standalone Java application.

**Prerequisites:**

*   **Java:** You must have a Java Development Kit (JDK), version 11 or newer, installed and configured.
*   **Local Kafka Broker:** A Kafka broker must be running on your machine at `localhost:9092`.

**Download the AKHQ JAR:**

1.  Go to the [AKHQ GitHub Releases page](https://github.com/tchiotludo/akhq/releases).
2.  Download the latest `akhq.jar` file to a directory of your choice (e.g., `C:\tools\akhq`).

**Create a Configuration File:**

In the same directory, create a configuration file named `application.yml`.

This file tells the AKHQ application how to connect to your Kafka cluster.
```yaml
akhq:
  connections:
    local-kafka:
      properties:
        bootstrap.servers: "localhost:9092"

micronaut:
  security:
    enabled: true
    token:
      jwt:
        signatures:
          secret:
            generator:
              secret: "ChangeThisSecretToSomethingLongAndSecure"
```

**Run the Application:**

Open a command prompt and navigate to the directory where you saved the files.

Execute the following command to start AKHQ.
```bash
C:\tools\akhq\> java -Dmicronaut.config.files=application.yml -jar akhq.jar
```

**Access the UI:**

Open your web browser and navigate to `http://localhost:8080`.

### Use Cases

Regardless of the installation method, AKHQ provides a range of features for managing your Kafka cluster:

*   **Topic Management:** View topic configurations, partition counts, and replication factors.
*   **Message Browse:** Search and view the content of messages in a topic.
*   **Consumer Group Management:** Monitor consumer offsets and reset them if needed.

---

## 3.4 Basic Kafka Configuration

### Introduction

Kafka's behavior is controlled by a configuration file, typically `server.properties`. For a dev environment, you only need to be familiar with a few key parameters.

### Explanation: Key Parameters in `server.properties`

The `server.properties` file for a KRaft setup is located at `config\kraft\server.properties`.

```properties
############################# Server Basics #############################

# A unique ID for the broker in the cluster. This must be a positive integer.
broker.id=1

############################ Socket Server Settings ###########################

# The listener that the broker will use to receive client requests.
listeners=PLAINTEXT://:9092

############################# Log Basics #############################

# The directory where Kafka will store its event data.
# This path is relative to the Kafka home directory.
log.dirs=C:/tmp/kafka-logs

# The default number of partitions for new topics if not specified.
num.partitions=1

############################# Log Retention Policy #############################

# How long to retain event data before deleting it (in hours).
# 168 hours = 7 days.
log.retention.hours=168

############################# KRaft Settings #############################

# The unique cluster ID for KRaft.
cluster.id=I8Q6d4P2V5a9b7Y4F1j5s7N3E9R2C8A

# A list of the KRaft quorum controller voters.
controller.quorum.voters=1@localhost:9093
```

---

## 3.5 Validating Installation with CLI Tools

### Introduction

After starting the Kafka broker, you can use the built-in command-line interface (CLI) tools to create a topic and send/receive messages, confirming that your installation is fully functional.

### Explanation: The Validation Flow

Open new command prompt windows for each step, leaving the Kafka broker running in its original window.

**1. Create a Topic:**

This command creates a topic named `my-first-topic`.
```bash
C:\kafka_4.0.0\> .\bin\windows\kafka-topics.bat --create --topic my-first-topic --partitions 1 --replication-factor 1 --bootstrap-server localhost:9092
```

**2. Produce Messages:**

Start a console producer. Any text you type will be sent as a message to the topic.
```bash
C:\kafka_4.0.0\> .\bin\windows\kafka-console-producer.bat --topic my-first-topic --bootstrap-server localhost:9092
```
```
> Hello from the producer!
> This is my second message.
```

**3. Consume Messages:**

Start a console consumer in a new command prompt. The `--from-beginning` flag ensures you read all messages that have been sent so far.
```bash
C:\kafka_4.0.0\> .\bin\windows\kafka-console-consumer.bat --topic my-first-topic --from-beginning --bootstrap-server localhost:9092
```
```
# Output will appear after the consumer connects:
# Hello from the producer!
# This is my second message.
```
As you type more messages in the producer window, they will appear in the consumer window in real time, confirming your local Kafka setup is working.