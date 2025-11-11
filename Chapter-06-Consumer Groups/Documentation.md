# Chapter 6: Consumer Groups & Offset Handling

- [6.1 Understanding Consumer Groups](#61-understanding-consumer-groups)
- [6.2 Auto vs Manual Offset Commit](#62-auto-vs-manual-offset-commit)
- [6.3 Offset Storage and Recovery](#63-offset-storage-and-recovery)
- [6.4 Consumer Rebalancing](#64-consumer-rebalancing)
- [6.5 Best Practices in Scalable Consumer Design](#65-best-practices-in-scalable-consumer-design)

---

## 6.1 Understanding Consumer Groups

![](../assets/image3.png)

A **Consumer Group** is a way to scale out the consumption of messages from a Kafka topic. It's a collection of one or more consumers that share a common `group.id`. The primary purpose is to ensure that a message from a topic is processed by only one consumer within that group.

- **Load Balancing**: When multiple consumers with the same `group.id` subscribe to a topic, Kafka automatically distributes the topic's partitions among them. Each partition is assigned to a single consumer in the group, preventing duplicate processing.
- **Scalability**: By adding more consumers to a group (up to the number of partitions), you can increase throughput and processing speed. If a topic has 10 partitions, you can have up to 10 consumers in a group, each handling one partition. Any consumers beyond this number will be idle.
- **Fault Tolerance**: If a consumer in a group fails, Kafka detects this and automatically reassigns its partitions to other active consumers in the same group.

---

## 6.2 Auto vs Manual Offset Commit

An **offset** is a unique, sequential identifier for each message within a partition. It acts as a bookmark, telling the consumer where it left off. **Offset committing** is the process of saving this bookmark to Kafka so that if the consumer stops and restarts, it knows where to begin reading again.

### a) Auto Commit

This is the default and simplest method. The Kafka client library automatically commits the offsets of the last consumed messages at a regular, configurable interval (e.g., every 5 seconds).

- **Pros**: Simple to configure and use; requires minimal code.
- **Cons**: Can lead to **data loss or duplicate processing**. If a consumer crashes after receiving messages but before the next auto-commit interval, those messages will be re-processed on restart. Conversely, if the consumer receives messages, auto-commits the offset, and then crashes before processing them, those messages are effectively lost (skipped) on restart.

### b) Manual Commit

In this approach, you disable auto-commits and explicitly tell the Kafka client when to commit offsets. This gives you fine-grained control over when a message is considered "successfully processed."

- **Pros**: Provides **at-least-once** or **exactly-once** processing guarantees. You commit the offset only after you are certain the message has been processed successfully (e.g., after it has been written to a database).
- **Cons**: More complex to implement and manage.

Here's a .NET code example showing a manual commit with the Confluent Kafka client:

```csharp


    using Confluent.Kafka;

    public class ManualCommitConsumer
    {
        public void Consume()
        {
            var config = new ConsumerConfig
            {
                GroupId = "my-consumer-group",
                BootstrapServers = "localhost:9092",
                EnableAutoCommit = false // Disable auto commit
            };

            using var consumer = new ConsumerBuilder<Ignore, string>(config).Build();
            consumer.Subscribe("my-topic");

            try
            {
                while (true)
                {
                    var consumeResult = consumer.Consume();

                    // Process the message
                    Console.WriteLine($"Received: {consumeResult.Message.Value} at {consumeResult.Offset}");

                    // Simulate processing
                    Thread.Sleep(100);

                    // Commit manually after success
                    consumer.Commit(consumeResult);
                }
            }
            catch (OperationCanceledException)
            {
                // graceful shutdown
            }
            finally
            {
                consumer.Close();
            }
        }
    }
```

---

## 6.3 Offset Storage and Recovery

KKafka stores consumer offsets in a special internal topic called **`__consumer_offsets`**. When a consumer commits an offset, it's essentially sending a message to this topic. Kafka's brokers handle the storage and management of this topic.

- **Storage**: The **`__consumer_offsets`** topic is compacted, meaning old offset messages for the same consumer group and partition are eventually replaced by newer ones. This prevents the topic from growing indefinitely.
- **Recovery**: When a consumer starts up, it reads from the **`__consumer_offsets`** topic to find its last committed offset for each of its assigned partitions. It then begins consuming messages from the offset _after_ the committed one. This mechanism is crucial for fault tolerance and ensures that the consumer can resume its work from where it left off, even after a restart.

---

## 6.4 Consumer Rebalancing

**Consumer Rebalancing** is the process by which Kafka dynamically re-distributes partition ownership among consumers in a group. This happens whenever the membership of a consumer group changes.

### Triggers

- A new consumer joins the group.
- A consumer leaves gracefully.
- A consumer fails (no heartbeats within `session.timeout.ms`).

### Process

1. Consumer joins/leaves.
2. Group Coordinator detects the change.
3. All consumers stop consuming.
4. Partitions are reassigned.
5. Consumers resume with new assignments.

⚠️ This creates a **short pause** ("stop-the-world"). Applications must be designed to handle it gracefully.

---

## 6.5 Best Practices in Scalable Consumer Design

- **Manual Offset Commits**: Use for critical apps. Commit only after processing is done.
- **Idempotent Consumers**: Make sure re-processing the same message has no side effects.
- **Partition Planning**: Have at least as many partitions as you expect consumer instances at peak load.
- **Monitor Consumer Lag**: Track lag metrics to know if consumers are falling behind.
- **Graceful Shutdown**: On shutdown, finish processing in-flight messages, commit final offsets, and close properly.

---

## Real-World Analogy

Imagine a **group of librarians** sorting books:

- The Kafka topic = conveyor belt of books.
- Partitions = different lanes of the conveyor.
- Consumer group = team of librarians.
- Consumers = individual librarians.
- Offset = sticky note marking last shelved book.
- Auto-commit = librarian updates sticky note every 5 minutes (may cause errors).
- Manual commit = librarian updates sticky note only after shelving each book (reliable).

---
