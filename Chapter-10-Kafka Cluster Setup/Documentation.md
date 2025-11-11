# Chapter 10: Kafka Cluster Setup & High Availability

- [10.1 Multi-Broker Cluster Setup (Zookeeper + KRaft)](#101-multi-broker-cluster-setup-zookeeper--kraft)
- [10.2 Understanding Controller and Leader Election](#102-understanding-controller-and-leader-election)
- [10.3 Configuring Replication and ISR](#103-configuring-replication-and-isr)
- [10.4 Broker Failure Recovery](#104-broker-failure-recovery)
- [10.5 Partition Assignment Strategies](#105-partition-assignment-strategies)

## 10.1 Multi-Broker Cluster Setup (Zookeeper + KRaft)

### Introduction

To achieve high availability, scalability, and fault tolerance, a Kafka cluster is typically configured with multiple brokers. A multi-broker setup ensures that if one broker fails, others can take over its responsibilities, and the data remains accessible. This section covers the two main architectures for setting up a multi-broker cluster: the traditional ZooKeeper-based method and the modern, ZooKeeper-less KRaft (Kafka Raft) mode.

### ZooKeeper-based Cluster

#### Explanation

For many years, Apache Kafka relied on Apache ZooKeeper for its cluster metadata management. ZooKeeper served as the central point for storing critical information about the Kafka cluster's state, including broker registration, topic configurations, and partition leader elections. In this architecture, ZooKeeper is a separate, distributed system that must be installed and managed independently of the Kafka brokers.

#### How it Works

1. **Start ZooKeeper Ensemble**: You must first start a ZooKeeper ensemble (a cluster of at least three nodes for fault tolerance). Kafka brokers connect to this ensemble.
2. **Broker Registration**: When a Kafka broker starts, it registers itself with ZooKeeper. This allows other brokers to discover its presence. Each broker is assigned a unique `broker.id`.
3. **Metadata Storage**: ZooKeeper stores all the cluster metadata. If a broker goes down, ZooKeeper notifies the other brokers, triggering a new leader election for the affected partitions.
4. **Configuration**: The `server.properties` file for each broker needs to specify the `zookeeper.connect` property, which points to the ZooKeeper ensemble.

#### Pros & Cons

- **Pros**: It's a well-established and battle-tested architecture. Many existing production clusters still use this mode.
- **Cons**: It adds an external dependency, increasing operational complexity. You have to manage and monitor two separate distributed systems (Kafka and ZooKeeper), which can be a burden.

### KRaft-based Cluster

#### Explanation

The Kafka Raft (KRaft) protocol is a major architectural shift that removes the dependency on ZooKeeper. Kafka itself now handles its own cluster metadata, using a Raft-based consensus algorithm. In a KRaft cluster, some Kafka brokers also take on the role of a Controller, which manages the cluster state. This simplifies the architecture and improves scalability.

#### How it Works

1. **Generate a Cluster ID**: You first generate a unique Cluster UUID using the `kafka-storage.sh` tool. This UUID is shared by all brokers in the cluster.
2. **Controller Quorum**: A set of brokers are configured to act as KRaft Controllers. They form a quorum (e.g., three or five controllers) and use the Raft protocol to elect a leader among themselves. This leader manages all the cluster metadata.
3. **Brokers**: The other brokers (or the same brokers in a shared-role setup) connect to the controller quorum to receive metadata updates.
4. **Configuration**: The `server.properties` file for each broker includes a `process.roles` property (either `broker`, `controller`, or `broker,controller`) and the `controller.quorum.voters` property, which lists the ID and address of all the controller-eligible brokers.

#### Pros & Cons

- **Pros**:
  - **Simpler Architecture**: No external dependency on ZooKeeper. This reduces the number of systems to manage.
  - **Improved Scalability**: It handles larger clusters more efficiently by separating the metadata plane from the data plane.
  - **Faster Leader Election**: The new consensus protocol allows for faster failover and recovery.
- **Cons**: It's a newer protocol, although it's now production-ready in recent Kafka versions. Migrating from a ZooKeeper-based cluster to a KRaft-based one can be a complex process.

### Summary Table

| Feature                    | ZooKeeper-based Cluster                   | KRaft-based Cluster                     |
| -------------------------- | ----------------------------------------- | --------------------------------------- |
| **Metadata Management**    | External (Apache ZooKeeper)               | Internal (Kafka Raft protocol)          |
| **Architecture**           | Requires two separate clusters            | Single, self-contained cluster          |
| **Operational Complexity** | High                                      | Low                                     |
| **Scalability**            | Can be limited by ZooKeeper               | Highly scalable                         |
| **Fault Tolerance**        | Leader election is dependent on ZooKeeper | Leader election is handled within Kafka |
| **Recommended for**        | Legacy deployments                        | All new deployments                     |

### Real-World Analogy

Think of running a school:

- **ZooKeeper-based setup**: This is like having two principals: one for students and one for teachers. The student principal (Kafka) manages the classrooms (partitions) and assignments (data), but all decisions about who is in charge (leader election) and how the school is organized (metadata) are made by the teacher principal (ZooKeeper). If the teacher principal is unavailable, the student principal can't make new decisions.
- **KRaft-based setup**: This is like having one principal who manages everything. The principal (KRaft Controller) is still part of the school staff, but they handle all the administrative decisions for the entire school, including who leads which classroom. This makes the whole system more self-sufficient and efficient.

## 10.2 Understanding Controller and Leader Election

### Introduction

In a multi-broker Kafka cluster, the concepts of a Controller and a Partition Leader are fundamental to its operation, fault tolerance, and high availability. While both are "leaders" in their own right, they serve very different purposes. The Controller manages the cluster's administrative state, while a Partition Leader handles all data read and write operations for a specific partition.

### The Kafka Controller

The Kafka Controller is a single, active broker within the cluster that has a special administrative role. Its primary responsibility is to act as the central authority for all cluster-level metadata and administrative tasks.

#### Role of the Controller

- **Partition Leader Election**: The Controller is responsible for initiating and managing the election of a new leader for a partition when the current leader fails.
- **Cluster State Management**: It tracks which brokers are online, which topics exist, and the location of all partition replicas.
- **Administrative Tasks**: It handles administrative operations like creating, deleting, and reassigning topics and partitions.
- **In-Sync Replicas (ISR) Management**: It keeps track of the replicas that are fully caught up with the partition leader.

#### Controller Election Process

- **In a ZooKeeper-based cluster**: The election process is managed by ZooKeeper's "ephemeral node" feature. Each broker attempts to create a unique, ephemeral ZNode (a node that exists only as long as its creator's session is active) called `/controller`. Only the first broker to successfully create this ZNode becomes the active Controller. If the active Controller fails, its ZNode is deleted, triggering a watch by the other brokers to start a new election.

- **In a KRaft-based cluster**: The Controller election is handled by the Raft consensus protocol. The Controller-eligible brokers form a quorum and use a voting process to elect a leader among themselves. This process is faster and doesn't rely on an external system.

### The Partition Leader

A Partition Leader is the replica for a given partition that is designated to handle all client-facing requests. This is the broker that producers send messages to and from which consumers fetch messages.

#### Role of the Partition Leader

- **Read/Write Operations**: All data reads and writes for a specific partition are routed through its leader. This simplifies client interactions, as they only need to know the location of the leader, not all of its replicas.
- **Replication Coordination**: The leader is responsible for accepting new messages from producers and then ensuring that its followers replicate those messages to stay in-sync.

#### Partition Leader Election Process

1. **Trigger**: A partition leader election is triggered by the Controller whenever the current leader of a partition fails, becomes unresponsive, or is shut down in a controlled manner.
2. **Eligibility**: The Controller selects a new leader from the list of In-Sync Replicas (ISR). The ISR list contains all the replicas that are fully caught up with the current leader's log. This is a critical step because it guarantees that the new leader has all the committed data, preventing any data loss.
3. **Selection**: The Controller typically selects the "preferred replica" as the new leader, if it's in the ISR. The preferred replica is the first one in the replica list for a partition, designed to ensure an even distribution of leadership across the brokers. If the preferred replica isn't available, another replica from the ISR is chosen.
4. **Update**: Once a new leader is chosen, the Controller updates the metadata for that partition and notifies all other brokers and clients of the change.

### Analogy: The School Hierarchy

Think of the Kafka cluster as a school.

- The Kafka Controller is the principal of the school. There is only one active principal at any time. The principal handles all high-level administrative tasks: hiring new teachers (creating new topics), assigning classes to teachers (assigning partitions to brokers), and, most importantly, choosing a new lead teacher for a class if the current one is absent (partition leader election).
- A Partition Leader is the lead teacher of a specific class. Every class (partition) has a single lead teacher who is responsible for teaching the students (handling read/write requests).
- The Followers are the assistant teachers in each class. They sit in on the lessons and keep their own notes to stay up-to-date. If the lead teacher gets sick, the principal (Controller) will choose one of the assistant teachers (followers in the ISR) to become the new lead teacher.
- The ISR is the group of assistant teachers who have attended every lesson and have notes that are fully consistent with the lead teacher's. The principal will only promote an assistant teacher from this group to ensure the class's curriculum remains complete.

## 10.3 Configuring Replication and ISR

### Introduction

Data durability and high availability are core strengths of Kafka, and they are achieved through its replication mechanism. Understanding how to configure the Replication Factor and In-Sync Replicas (ISR) is essential for balancing data safety, performance, and resource usage in a production environment. These settings determine how many copies of your data exist and how many of those copies must be available for a write to be considered successful.

### Replication Factor

The Replication Factor is a topic-level setting that defines the number of copies (replicas) of each partition. Each replica is stored on a different broker in the cluster.

#### Purpose

- **High Availability**: If a broker fails, a replica of its partition exists on another broker, so data remains accessible.
- **Data Durability**: Replicating data ensures that it is not lost if a broker crashes.

#### Configuration

The replication factor is set when a new topic is created using the kafka-topics command. For example, to create a topic with a replication factor of 3:

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic my-replicated-topic --partitions 3 --replication-factor 3
```

The recommended replication factor for production is 3. This provides a good balance between fault tolerance and resource overhead. It allows you to lose up to two brokers without data loss.

You can change the replication factor of an existing topic using the kafka-reassign-partitions tool, but this is a more involved process.

### In-Sync Replicas (ISR)

The In-Sync Replicas (ISR) is a dynamic list of replicas for a given partition that are "in sync" with the partition's leader. A replica is considered "in sync" if it has fully caught up with the leader's log and has not fallen behind by a significant amount.

#### Purpose

- **Data Consistency**: Only replicas in the ISR are considered for a new leader election if the current leader fails. This ensures that the newly elected leader has a complete and up-to-date copy of the data, preventing data loss.
- **Write Guarantees**: When a producer sends a message with `acks=all`, it waits for the message to be written to all replicas in the ISR. This provides the highest level of data durability.

#### Configuration

The `min.insync.replicas` property is a topic-level configuration that defines the minimum number of in-sync replicas required for a write to succeed. If the number of in-sync replicas falls below this threshold, producers will receive an error, and no new data can be written to that partition.

It is a crucial setting that works in tandem with the producer's `acks` configuration. A typical production setup uses `replication.factor=3` and `min.insync.replicas=2`.

```bash
kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name my-replicated-topic --alter --add-config min.insync.replicas=2
```

**Note**: `min.insync.replicas` must always be less than or equal to the topic's `replication.factor`.

### Working Together: Producer acks

The producer's `acks` (acknowledgments) setting determines how many acknowledgments a producer must receive from brokers before considering a message successfully sent.

- `acks=0`: Producer does not wait for any acknowledgment. This is a high-throughput but low-durability setting, as messages can be lost.
- `acks=1`: Producer waits for the leader replica to write the message to its log. The leader does not wait for followers to replicate the data before sending an acknowledgment. This is a good balance for many use cases.
- `acks=all (-1)`: Producer waits for the message to be written to the leader and all replicas in the ISR. This, in combination with `min.insync.replicas`, provides the strongest durability guarantee. This is the recommended setting for critical data.

### Real-World Analogy

Think of Kafka replication as taking notes in a very important class.

- The Replication Factor is the number of students taking notes. A replication factor of 3 means there are three students: a lead student and two assistant students.
- The Leader is the lead student. The professor (producer) only lectures to this student, and all the other students (followers) must copy their notes.
- The Followers are the assistant students. They are constantly trying to keep their notes up-to-date by copying from the leader's notes.
- The In-Sync Replicas (ISR) is the group of students who are all diligently taking notes and are completely caught up with the professor's lecture. If the lead student suddenly gets sick and leaves, the teacher knows that any student from the ISR group has a complete set of notes and can be promoted to be the new lead student without losing any information.
- The `min.insync.replicas` setting is like a rule the professor enforces: "I will not continue my lecture until at least two of you (the leader and one assistant) have confirmed that you've written down the last point." This ensures that the lecture progresses only when the key information is safely recorded by a sufficient number of students, preventing the loss of vital notes.

## 10.4 Broker Failure Recovery

### Introduction

A key advantage of Kafka's distributed architecture is its ability to automatically recover from a broker failure without data loss or service disruption. The recovery process is managed by the Kafka Controller and is heavily dependent on the concepts of replication and In-Sync Replicas (ISR). Understanding this process is crucial for operating a reliable Kafka cluster.

### The Failure and Recovery Process

The recovery process can be broken down into a series of automatic steps that happen in a matter of seconds.

1. **Failure Detection**: When a broker fails (e.g., due to a hardware failure, network issue, or process crash), its connection to the Kafka cluster is lost. The other brokers and, most importantly, the Controller, detect this failure.

   - In a ZooKeeper-based cluster, the broker's ephemeral node in ZooKeeper is automatically deleted, which triggers a watch event that the Controller receives.
   - In a KRaft-based cluster, the Controller detects the failure directly through network timeouts and internal health checks.

2. **Controller Action**: Upon detecting the failure, the Controller takes over. It identifies all the partitions that had a replica on the failed broker. For each partition, it determines if the failed broker was the partition leader.

   - If the failed broker was a follower for a partition, the partition's data remains available because its leader is still active. The Controller simply removes the failed broker from the replica list for that partition.
   - If the failed broker was the leader for a partition, a new leader election is immediately triggered for that partition.

3. **Partition Leader Re-election**: This is the most critical step. The Controller selects a new leader for each affected partition from its list of In-Sync Replicas (ISR).

   The ISR list contains all the replicas that are fully caught up with the former leader. By choosing from this list, Kafka guarantees that no committed messages are lost during the failover.

   The new leader takes over all responsibility for accepting reads and writes for that partition.

4. **Client Notification**: The Kafka Controller sends out a metadata update to all brokers and connected clients (producers and consumers).

   Producers and consumers receive this update and automatically discover the new partition leader. They seamlessly start sending requests to the new leader without any manual intervention. This is why client libraries like Confluent.Kafka are so powerful—they handle this entire process transparently to your application.

5. **Broker Recovery**: When the failed broker eventually comes back online, it automatically rejoins the cluster.

   It checks the metadata and starts to act as a follower for all the partitions for which it holds a replica.

   The broker begins to fetch messages from the new leaders to catch up and rejoin the ISR. Once it's fully caught up, it is re-added to the ISR list by the Controller, and it can once again be considered for future leader elections.

### Analogy: The Automated Supply Chain

Imagine a high-tech warehouse with multiple conveyor belts (brokers) moving products (messages).

- **Normal Operation**: One main conveyor belt (the partition leader) receives all the new products from the factory (producer). Several other secondary conveyor belts (followers) run parallel to the main one, copying every product as it passes. The secondary belts that are moving at the same speed and have all the products are the ISR.

- **Broker Failure**: Suddenly, the main conveyor belt breaks down (broker failure).

- **Recovery**: Instantly, the central computer (the Controller) notices the breakdown. It finds the next-fastest, fully up-to-date secondary conveyor belt from the ISR group and makes it the new main belt.

- **Seamless Transition**: The factory automatically starts sending new products to this new main belt. The original customers (consumers) are automatically rerouted to the new belt as well. The old, broken belt can be fixed and brought back online. When it returns, it will start to copy from the new main belt until it's fully caught up.

## 10.5 Partition Assignment Strategies

### Introduction

When a group of consumers subscribe to a topic, Kafka needs a way to distribute the partitions of that topic among the consumers. This is handled by a partition assignment strategy. This strategy determines which consumer in a consumer group is responsible for fetching messages from which partition. The correct strategy is vital for ensuring even workload distribution, high throughput, and preventing "hot spots" where one consumer is overloaded while others are idle.

### The Consumer Group and Rebalancing

First, let's briefly define a consumer group. A consumer group is a set of consumers that cooperate to consume data from one or more topics. When a consumer joins or leaves a group, or when a topic's partitions are changed, the group triggers a rebalance. A rebalance is the process of reassigning partitions among the consumers in the group. All consumers in the group stop consuming, the partitions are re-assigned according to the chosen strategy, and then consumption resumes with the new assignments.

### Common Partition Assignment Strategies

Kafka's client libraries, including the Confluent.Kafka .NET client, provide a few built-in strategies. You can configure the strategy using the `partition.assignment.strategy` property in your consumer configuration.

1. **RangeAssignor**

   - **How it works**: This is the default strategy. It works on a per-topic basis. For each topic, it sorts the partitions numerically and the consumers lexicographically. It then divides the partitions into roughly equal "ranges" and assigns each range to a consumer.

   - **Example**:

     Topic with 10 partitions (0-9) and a consumer group with 3 consumers (C1, C2, C3).

     - C1 gets partitions 0, 1, 2, 3.
     - C2 gets partitions 4, 5, 6.
     - C3 gets partitions 7, 8, 9.

     This strategy can lead to an uneven distribution of partitions if a consumer is assigned more partitions than others, especially with multiple topics.

   - **Pros**: Simple and predictable.
   - **Cons**: Can result in an unequal distribution of partitions among consumers, potentially causing an imbalance in workload, especially if the number of partitions isn't a clean multiple of the number of consumers.

2. **RoundRobinAssignor**

   - **How it works**: This strategy works on a per-consumer basis across all subscribed topics. It sorts the partitions from all subscribed topics and then assigns them to consumers in a round-robin fashion.

   - **Example**:

     Topic A has 3 partitions (0, 1, 2). Topic B has 2 partitions (0, 1).

     Consumer group with 3 consumers (C1, C2, C3).

     - C1 gets partition A-0, C2 gets A-1, C3 gets A-2.
     - Then C1 gets partition B-0, C2 gets B-1, C3 gets B-2. Oh wait, B-2 doesn't exist so it will be evenly distributed among C1 and C2.

     This strategy tends to result in a more even distribution of partitions.

   - **Pros**: Ensures a more balanced distribution of partitions, leading to a more even workload across consumers.
   - **Cons**: Can be less predictable than the RangeAssignor.

3. **StickyAssignor (Recommended)**

   - **How it works**: This is the most modern and recommended strategy. The "sticky" part means it tries to keep the assignment of partitions to consumers as stable as possible during a rebalance. It also aims for a perfectly balanced assignment. It minimizes the number of partition movements during a rebalance.

   - **Example**:

     A consumer group has 3 consumers (C1, C2, C3) and a topic with 9 partitions (0-8).

     - **Initial Assignment**: C1 gets partitions 0, 1, 2. C2 gets 3, 4, 5. C3 gets 6, 7, 8.

     Now, consumer C1 leaves the group.

     - A rebalance is triggered. The StickyAssignor tries to re-assign C1's partitions (0, 1, 2) to C2 and C3 while keeping C2 and C3's existing assignments stable. The new assignment will likely be C2 getting 3, 4, 5, 0, 1, and C3 getting 6, 7, 8, 2. This minimized the number of partitions that had to be moved.

   - **Pros**:
     - Minimizes Partition Movement: This is the biggest advantage. It reduces the overhead and latency associated with rebalancing.
     - Perfectly Balanced: It aims for a perfectly even distribution of partitions across consumers.
   - **Cons**: The logic is more complex than the other strategies.

### The .NET Consumer Configuration

In the Confluent.Kafka .NET client, you would set the partition assignment strategy in your `ConsumerConfig` object:

```csharp
using Confluent.Kafka;
// ...

var config = new ConsumerConfig
{
    GroupId = "my-consumer-group",
    BootstrapServers = "localhost:9092",
    // Set the partition assignment strategy
    PartitionAssignmentStrategy = PartitionAssignmentStrategy.Sticky,
    AutoOffsetReset = AutoOffsetReset.Earliest
};

using (var consumer = new ConsumerBuilder<string, string>(config).Build())
{
    // ...
}
```

### Real-World Analogy

Think of partition assignment as organizing a team of couriers (consumers) to deliver packages (messages) from different postal codes (partitions).

- **RangeAssignor**: This is like assigning the couriers by postal code. Courier A gets postal codes 1000-1010, Courier B gets 1011-1020, and so on. It's a simple and logical approach, but if one postal code area is much larger or busier than others, that courier will be overworked.
- **RoundRobinAssignor**: This is like giving the first package to Courier A, the second to Courier B, the third to C, and so on. This ensures that everyone gets a fair share of the packages, regardless of which postal code they're from.
- **StickyAssignor**: This is the best approach. It's like a smart dispatcher who not only makes sure the workload is balanced but also tries to avoid disrupting a courier's route. If a new courier joins the team, the dispatcher will give them a fair share of the new packages without taking existing routes away from the other couriers unless absolutely necessary. This minimizes the chaos and ensures deliveries continue smoothly.
