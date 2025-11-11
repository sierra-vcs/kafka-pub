# Chapter 5: Topics, Partitions & Message Flow

- [5.1 Creating Topics with CLI and .NET](#51-creating-topics-with-cli-and-net)
- [5.2 Partitioning Strategies (Round-Robin, Key-Based)](#52-partitioning-strategies-round-robin-key-based)
- [5.3 Message Ordering and Keys](#53-message-ordering-and-keys)
- [5.4 Replication Factor and Durability](#54-replication-factor-and-durability)
- [5.5 Log Retention Policies](#55-log-retention-policies)
- [5.6 Log Compaction Explained](#56-log-compaction-explained)

---

## 5.1 Creating Topics with CLI and .NET

### Explanation

A Kafka topic is a named feed where events are stored. It acts as a category for related messages. Before you can send or receive messages, a topic must be created. You can do this with command-line tools for simple tasks or programmatically within your code.

### Creating a Topic with the Command Line (CLI)

This is the fastest way to create a topic using the `kafka-topics` script.

```bash
./bin/windows/kafka-topics.bat --create \
  --topic <topic-name> \
  --bootstrap-server <bootstrap-server> \
  --partitions <num-partitions> \
  --replication-factor <num-replicas>
```

**Explanation of Settings:**

- `--topic`: The name of the feed you are creating. Example: `user-activity-events`.
- `--bootstrap-server`: The network address of a Kafka broker that your command can connect to.
- `--partitions`: The number of divisions your topic will have. More partitions allow multiple consumers to read in parallel.
- `--replication-factor`: The number of copies of your messages. A factor of `3` provides high durability by storing replicas across machines.

### Creating a Topic with .NET AdminClient

You can also create topics programmatically using `IAdminClient` from Confluent.Kafka.

```csharp
using Confluent.Kafka.Admin;
using Confluent.Kafka;

var config = new AdminClientConfig { BootstrapServers = "localhost:9092" };
using var adminClient = new AdminClientBuilder(config).Build();

var topicName = "new-user-topic";
var topicSettings = new TopicSpecification
{
    Name = topicName,
    NumPartitions = 5,
    ReplicationFactor = 1
};

try
{
    await adminClient.CreateTopicsAsync(new[] { topicSettings });
    Console.WriteLine($"Topic '{topicName}' created successfully.");
}
catch (CreateTopicsException e)
{
    Console.WriteLine($"Failed to create topic: {e.Results[0].Error.Reason}");
}
```

---

## 5.2 Partitioning Strategies (Round-Robin, Key-Based)

### Explanation

A topic is divided into partitions. A producer must decide which partition to send a message to. The two common strategies are **round-robin** and **key-based**.

### Round-Robin Partitioning

- **How it works**: When no key is provided, Kafka distributes messages evenly across partitions.
- **When to use**: Best for balanced distribution where message order isn’t important.

### Key-Based Partitioning

- **How it works**: Kafka hashes the message key (e.g., user ID) and always assigns the same key to the same partition.
- **When to use**: Ensures message order for related events (e.g., all events for a single bank account).

### C# Example

```csharp
// IProducer<string, string> - first type is key, second is value.
using var producer = new ProducerBuilder<string, string>(config).Build();

// Key-based partitioning ensures order per user.
await producer.ProduceAsync("my-topic", new Message<string, string>
{
    Key = "user-123",
    Value = "First message from user 123"
});

await producer.ProduceAsync("my-topic", new Message<string, string>
{
    Key = "user-123",
    Value = "Second message from user 123"
});

// Different key → likely different partition.
await producer.ProduceAsync("my-topic", new Message<string, string>
{
    Key = "user-456",
    Value = "Message from user 456"
});
```

---

## 5.3 Message Ordering and Keys

### Explanation

- **Order within a partition**: Messages are always read in the same order they were written.
- **No global order**: Across partitions, Kafka does not guarantee order.
- **Keys for order**: By assigning a key, you ensure all related messages go to the same partition, maintaining correct order for that key.

---

## 5.4 Replication Factor and Durability

### Explanation

- **Replication factor**: Defines how many copies of each partition exist (e.g., `3` for high availability).
- **Leader & Followers**: One partition replica is the **Leader**. Others are **Followers**. Producers and consumers interact only with the Leader.
- **In-Sync Replicas (ISR)**: Followers fully caught up with the Leader.
- **Durability with acks**:
  - `acks=1`: Leader confirms only.
  - `acks=all`: Waits for all ISRs → safest against data loss.

---

## 5.5 Log Retention Policies

### Explanation

Kafka is not a permanent database — it deletes old messages based on **retention policies**.

### Time-Based Retention

- Config: `log.retention.hours`
- Example: `168` → messages deleted after 7 days.

### Size-Based Retention

- Config: `log.retention.bytes`
- Deletes oldest messages once partition log exceeds configured size.

### C# Example (Custom Retention)

```csharp
var adminConfig = new AdminClientConfig { BootstrapServers = "localhost:9092" };
using var adminClient = new AdminClientBuilder(adminConfig).Build();

var topicSpec = new TopicSpecification
{
    Name = "24-hour-retention-topic",
    NumPartitions = 1,
    ReplicationFactor = 1,
    Configs = new Dictionary<string, string>
    {
        { "retention.ms", "86400000" } // 24 hours
    }
};

await adminClient.CreateTopicsAsync(new[] { topicSpec });
```

---

## 5.6 Log Compaction Explained

### Explanation

Log compaction keeps **only the latest message per key** instead of deleting messages by time/size. It’s like a key-value store.

- **Before compaction**: Multiple values for the same key exist.
- **After compaction**: Only the latest value per key remains.

### Example

**Before Compaction:**
| Offset | Key | Value |
|--------|-------|--------------------|
| 0 | Alice | Address: 123 Main |
| 1 | Bob | Address: 456 Elm |
| 2 | Alice | Address: 789 Oak |

**After Compaction:**
| Offset | Key | Value |
|--------|-------|--------------------|
| 1 | Bob | Address: 456 Elm |
| 2 | Alice | Address: 789 Oak |

### When to Use

- **CDC (Change Data Capture)** → keep only the latest row state.
- **State Stores** → stream apps storing latest state of an object.

### C# Example

```csharp
var adminConfig = new AdminClientConfig { BootstrapServers = "localhost:9092" };
using var adminClient = new AdminClientBuilder(adminConfig).Build();

var topicSpec = new TopicSpecification
{
    Name = "state-store-topic",
    NumPartitions = 1,
    ReplicationFactor = 1,
    Configs = new Dictionary<string, string>
    {
        { "cleanup.policy", "compact" }
    }
};

await adminClient.CreateTopicsAsync(new[] { topicSpec });
```
