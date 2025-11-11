# Chapter 4: Kafka with .NET – Producing & Consuming Messages

- [4.1 Installing Confluent.Kafka NuGet Package](#41-installing-confluentkafka-nuget-package)
- [4.2 Creating a Kafka Producer in .NET](#42-creating-a-kafka-producer-in-net)
- [4.3 Creating a Kafka Consumer in .NET](#43-creating-a-kafka-consumer-in-net)
- [4.4 Serializing JSON Messages](#44-serializing-json-messages)
- [4.5 Basic Producer/Consumer Configurations](#45-basic-producerconsumer-configurations)
- [4.6 Error Handling and Retry Logic](#46-error-handling-and-retry-logic)

---

## 4.1 Installing Confluent.Kafka NuGet Package

### Introduction

The Confluent.Kafka library is the official and widely-used C# client for Apache Kafka. It provides a robust, high-performance API that wraps the native librdkafka C library, making it the standard choice for .NET applications.

### Explanation: Installation Methods

There are two common ways to install the package into your .NET project.

**Using the NuGet Package Manager (Visual Studio):**

- In Visual Studio, right-click on your project in the Solution Explorer and select **"Manage NuGet Packages..."**.
- Go to the **"Browse"** tab and search for `Confluent.Kafka`.
- Select the package and click **Install**.

**Using the .NET CLI:**
Open a terminal (Command Prompt, PowerShell, or dotnet CLI) in your project's directory and run:

```bash
dotnet add package Confluent.Kafka
```

---

## 4.2 Creating a Kafka Producer in .NET

### Introduction

A producer is an application that publishes events (messages) to a Kafka topic. In a .NET application, you use the `IProducer` interface from the Confluent.Kafka library to create and send messages.

### Explanation & Code Example

The following console application demonstrates how to configure and use a producer to send a simple string message.

```csharp
using Confluent.Kafka;

var config = new ProducerConfig
{
    // Replace "localhost:9092" with your actual broker addresses.
    BootstrapServers = "localhost:9092"
};

using var producer = new ProducerBuilder<Null, string>(config).Build();

try
{
    var deliveryReport = await producer.ProduceAsync("my-first-topic", new Message<Null, string>
    {
        Value = "Hello, Kafka from .NET!"
    });

    Console.WriteLine($"Message sent to partition: {deliveryReport.Partition}, offset: {deliveryReport.Offset}");
}
catch (ProduceException<Null, string> e)
{
    Console.WriteLine($"Delivery failed: {e.Error.Reason}");
}

// Ensure all buffered messages are sent.
producer.Flush(TimeSpan.FromSeconds(10));
```

---

## 4.3 Creating a Kafka Consumer in .NET

### Introduction

A consumer is an application that subscribes to a Kafka topic and reads the events. Consumers are part of a **consumer group**, which allows multiple consumers to read a topic's partitions in parallel, distributing the workload.

### Explanation & Code Example

```csharp
using Confluent.Kafka;

var config = new ConsumerConfig
{
    BootstrapServers = "localhost:9092",
    GroupId = "my-consumer-group",
    AutoOffsetReset = AutoOffsetReset.Earliest
};

using var consumer = new ConsumerBuilder<Null, string>(config).Build();
consumer.Subscribe("my-first-topic");

Console.WriteLine("Consumer started. Press Ctrl+C to exit.");

try
{
    while (true)
    {
        var consumeResult = consumer.Consume();
        Console.WriteLine($"Consumed message '{consumeResult.Message.Value}' at: '{consumeResult.TopicPartitionOffset}'.");
        consumer.Commit(consumeResult);
    }
}
catch (OperationCanceledException)
{
    // Triggered when Ctrl+C is pressed
}
finally
{
    consumer.Close();
}
```

---

## 4.4 Serializing JSON Messages

### Introduction

In real-world applications, you often send and receive complex data structures, not just strings. Before sending, objects must be serialized. JSON is the most common format.

### Explanation & Code Example

**Define a C# model:**

```csharp
public class Order
{
    public int OrderId { get; set; }
    public string ProductName { get; set; }
    public decimal Price { get; set; }
}
```

**Producer serializing JSON:**

```csharp
using System.Text.Json;

using var producer = new ProducerBuilder<Null, string>(config).Build();

var order = new Order { OrderId = 123, ProductName = "Laptop", Price = 1200.50m };
string jsonOrder = JsonSerializer.Serialize(order);

await producer.ProduceAsync("my-orders-topic", new Message<Null, string>
{
    Value = jsonOrder
});
```

**Consumer deserializing JSON:**

```csharp
using System.Text.Json;

using var consumer = new ConsumerBuilder<Null, string>(config).Build();
consumer.Subscribe("my-orders-topic");

while (true)
{
    var consumeResult = consumer.Consume();
    string jsonOrder = consumeResult.Message.Value;

    var order = JsonSerializer.Deserialize<Order>(jsonOrder);

    Console.WriteLine($"Received order for product: {order.ProductName}, ID: {order.OrderId}");
    consumer.Commit(consumeResult);
}
```

---

## 4.5 Basic Producer/Consumer Configurations

### Producer Configurations

- **Acks** (`Acks.None`, `Acks.Leader`, `Acks.All`)

  - `None (0)` → Fastest, no guarantee of delivery.
  - `Leader (1)` → Leader broker acknowledges, moderate reliability.
  - `All (-1)` → Leader + all ISRs acknowledge, strongest guarantee.

- **LingerMs** → Time producer waits before sending batch.
- **BatchSize** → Max size of a batch of messages.

### Consumer Configurations

- **GroupId** → Required, defines consumer group.
- **AutoOffsetReset** (`Earliest`, `Latest`) → Where to start if no offset.
- **EnableAutoCommit** → If `false`, must commit manually for exactly-once guarantees.

---

## 4.6 Error Handling and Retry Logic

### Introduction

In distributed systems, network issues and transient failures are expected. Proper retry and error handling are essential.

### Producer Error Handling

```csharp
try
{
    var deliveryReport = await producer.ProduceAsync("my-topic", new Message<Null, string>
    {
        Value = "test message"
    });

    if (deliveryReport.Status != PersistenceStatus.Persisted)
    {
        Console.WriteLine($"Message delivery failed: {deliveryReport.Status}");
    }
    else
    {
        Console.WriteLine($"Message delivered to {deliveryReport.TopicPartitionOffset}");
    }
}
catch (ProduceException<Null, string> e)
{
    Console.WriteLine($"Permanent produce error: {e.Error.Reason}");
}
```

### Consumer Error Handling

```csharp
using var consumer = new ConsumerBuilder<Null, string>(config).Build();
consumer.Subscribe("my-topic");

while (true)
{
    try
    {
        var consumeResult = consumer.Consume(TimeSpan.FromSeconds(1));

        if (consumeResult == null) continue;

        if (consumeResult.IsPartitionEOF)
        {
            Console.WriteLine($"End of partition {consumeResult.TopicPartition} reached.");
            continue;
        }

        Console.WriteLine($"Consumed message '{consumeResult.Message.Value}'");
    }
    catch (ConsumeException e)
    {
        Console.WriteLine($"Consume error: {e.Error.Reason}");
    }
    catch (OperationCanceledException)
    {
        break;
    }
}
```
