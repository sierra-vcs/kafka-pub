# 📘 Apache Kafka for .NET Developers – Structured Learning Path

## 🟢 [Chapter 1: Getting Started with Apache Kafka](./Chapter-01-Introduction/)

- 1.1 Introduction to Kafka and Event Streaming
- 1.2 Kafka in Modern Architecture
- 1.3 Kafka Ecosystem Overview
- 1.4 Kafka vs RabbitMQ vs MQTT vs Azure Event Hubs

## 🟢 [Chapter 2: Kafka Core Concepts & Architecture](./Chapter-02-Core%20&%20Architecture/)

- 2.1 Kafka Brokers and Clusters
- 2.2 Topics and Partitions
- 2.3 Producers and Consumers
- 2.4 Replication and In-Sync Replicas
- 2.5 Zookeeper vs KRaft

## 🟢 [Chapter 3: Kafka Setup & Installation (Dev Environment)](./Chapter-03-Setup%20&%20Installation/)

- 3.1 Installing Kafka on Windows (Manual + WSL)
- 3.2 Installing Kafka on Linux (Debian/Ubuntu/CentOS)
- 3.3 Installing Kafka on macOS (Homebrew or Manual)
- 3.4 Setting up Kafka using Docker Compose
- 3.5 Installing Kafka UI Tools (Kafdrop, kafka-ui, AKHQ)
- 3.6 Basic Kafka Configuration (Ports, Data Dir, Log Retention)
- 3.7 Running Kafka with KRaft Mode
- 3.8 Validating Installation with CLI Tools

## 🟢 [Chapter 4: Kafka with .NET – Producing & Consuming Messages](./Chapter-04-%20Kafka%20with%20.NET/)

- 4.1 Installing Confluent.Kafka NuGet Package
- 4.2 Creating a Kafka Producer in .NET
- 4.3 Creating a Kafka Consumer in .NET
- 4.4 Serializing JSON Messages
- 4.5 Basic Producer/Consumer Configurations
- 4.6 Error Handling and Retry Logic

## 🟡 [Chapter 5: Topics, Partitions & Message Flow](./Chapter-05-Topics,Partitions%20&%20Message%20Flow/)

- 5.1 Creating Topics with CLI and .NET
- 5.2 Partitioning Strategies (Round-Robin, Key-Based)
- 5.3 Message Ordering and Keys
- 5.4 Replication Factor and Durability
- 5.5 Log Retention Policies
- 5.6 Log Compaction Explained

## 🟡 [Chapter 6: Consumer Groups & Offset Handling](./Chapter-06-Consumer%20Groups/)

- 6.1 Understanding Consumer Groups
- 6.2 Auto vs Manual Offset Commit
- 6.3 Offset Storage and Recovery
- 6.4 Consumer Rebalancing
- 6.5 Best Practices in Scalable Consumer Design

## 🟡 [Chapter 7: Kafka Message Delivery Semantics](./Chapter-07-Kafka%20Message%20Delivery%20Semantics/)

- 7.1 At-Most-Once, At-Least-Once, Exactly-Once
- 7.2 Idempotent Producers
- 7.3 Kafka Transactions
- 7.4 Dead Letter Topics in .NET
- 7.5 Error Handling Patterns

## 🟡 [Chapter 8: Schema Management and Serialization](./Chapter-08-Schema%20Management%20and%20Serialization/)

- 8.1 Introduction to Schema Registry
- 8.2 Avro, Protobuf, JSON Schema Overview
- 8.3 Integrating Schema Registry with .NET
- 8.4 Handling Schema Evolution
- 8.5 Configuring Compatibility Settings

## 🔵 [Chapter 9: Kafka Connect – Data Integration](./Chapter-09-Kafka%20Connect/)

- 9.1 Kafka Connect Architecture
- 9.2 Installing Kafka Connect
- 9.3 Source vs Sink Connectors
- 9.4 Configuring Connectors (e.g., JDBC, S3, Elasticsearch)
- 9.5 Running Standalone vs Distributed Mode
- 9.6 Custom Connector Overview

## 🔵 [Chapter 10: Kafka Cluster Setup & High Availability](./Chapter-10-Kafka%20Cluster%20Setup/)

- 10.1 Multi-Broker Cluster Setup (Zookeeper + KRaft)
- 10.2 Understanding Controller and Leader Election
- 10.3 Configuring Replication and ISR
- 10.4 Broker Failure Recovery
- 10.5 Partition Assignment Strategies

## 🔵 [Chapter 11: Kafka Stream Processing](./Chapter-11-Kafka%20Stream%20Processing/)

- 11.1 Kafka Streams Overview
- 11.2 KStreams vs KTables
- 11.3 Windowing, Aggregation, and Joins
- 11.4 Stream Processing in .NET (Streamiz.Kafka.Net)
- 11.5 Introduction to ksqlDB

## 🔴 [Chapter 12: Kafka Security Essentials](./Chapter-12-Kafka%20Security%20Essentials/)

- 12.1 SSL Encryption Configuration
- 12.2 SASL Authentication (PLAIN, SCRAM, OAUTH)
- 12.3 ACLs for Topic-Level Authorization
- 12.4 Securing .NET Kafka Clients
- 12.5 Vault & Secret Management

## 🔴 [Chapter 13: Kafka Observability & Operations](./Chapter-13-Kafka%20Observability/)

- 13.1 Monitoring Kafka Metrics
- 13.2 Prometheus + Grafana Setup
- 13.3 Kafka Logging Strategies
- 13.4 Alerting for Lag, Broker Health
- 13.5 Consumer Lag Detection and Analysis

---

### Additional to learn

## 🔴 Chapter 14: DevOps, CI/CD & Testing Kafka Apps

- 14.1 Docker Compose for Local Dev
- 14.2 Integration Tests with Kafka (TestContainers, Local Kafka)
- 14.3 Unit Testing Kafka Producers and Consumers
- 14.4 GitHub Actions Pipeline for Kafka Projects
- 14.5 Automating Topic Creation and Config Checks

## 🔴 Chapter 15: Event-Driven Microservices with Kafka in .NET

- 15.1 Microservices Event Flow Design
- 15.2 Building Domain Events in .NET
- 15.3 Saga & Orchestration Patterns
- 15.4 Distributed Tracing with OpenTelemetry
- 15.5 Deployment, Scaling, and Observability in Real Scenarios
