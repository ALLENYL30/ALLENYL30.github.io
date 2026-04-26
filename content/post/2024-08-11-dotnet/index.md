+++
author = "yuhao"
title = "Kafka with .NET Core"
date = "2024-08-10"
description = "- ZooKeeper: Download"
tags = [
    ".NET",
    ".NET Core",
    "Kafka",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
# Kafka Quick Start Guide

## Installation

### CentOS Installation

#### Prerequisites

- Kafka: [Download](http://kafka.apache.org/downloads)
- ZooKeeper: [Download](https://zookeeper.apache.org/releases.html)

#### Step 1: Download & Extract

```bash
# Download Kafka
wget https://archive.apache.org/dist/kafka/2.1.1/kafka_2.12-2.1.1.tgz
tar -zxvf kafka_2.12-2.1.1.tgz
mv kafka_2.12-2.1.1 /data/kafka

# Download ZooKeeper
wget https://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.5.8/apache-zookeeper-3.5.8-bin.tar.gz
tar -zxvf apache-zookeeper-3.5.8-bin.tar.gz
mv apache-zookeeper-3.5.8-bin /data/zookeeper
```

#### Step 2: Start ZooKeeper

```bash
cd /data/zookeeper/conf
cp zoo_sample.cfg zoo.cfg

# Modify configuration if needed
vim zoo.cfg

# Service Management
./bin/zkServer.sh start     # Start
./bin/zkServer.sh status    # Check status
./bin/zkServer.sh stop      # Stop
./bin/zkServer.sh restart   # Restart

# Test connection
./bin/zkCli.sh -server localhost:2181
quit
```

#### Step 3: Configure & Start Kafka

```bash
cd /data/kafka/config
cp server.properties server.properties_backup

# Edit critical configurations
vim server.properties

"""
# Cluster-unique broker ID
broker.id=0

# Internal listener
listeners=PLAINTEXT://<internal_ip>:9092

# External advertised address
advertised.listeners=PLAINTEXT://<public_ip>:9092

# Default partitions per topic
num.partitions=3

# ZooKeeper connection
zookeeper.connect=localhost:2181
"""

# Start Kafka
./bin/kafka-server-start.sh config/server.properties &

# Verify process
ps -ef | grep kafka
jps
```

### Docker Installation

```bash
# ZooKeeper
docker pull wurstmeister/zookeeper
docker run -d --name zookeeper -p 2181:2181 wurstmeister/zookeeper

# Kafka
docker run -d --name kafka \
  -p 9092:9092 \
  --link zookeeper \
  -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_ADVERTISED_HOST_NAME=<host_ip> \
  -e KAFKA_ADVERTISED_PORT=9092 \
  wurstmeister/kafka
```

## Core Concepts

![Kafka Architecture](https://api.meowv.com/api/meowv/tool/img?url=https://img2020.cnblogs.com/blog/891843/202009/891843-20200904161605310-1875847528.png)

| Term          | Description                                                                               |
|---------------|-------------------------------------------------------------------------------------------|
| **Broker**    | Kafka server node. Multiple brokers form a cluster.                                       |
| **Topic**     | Logical message category (e.g., `page_views`, `click_streams`).                           |
| **Partition** | Physical subdivision of topics. Each partition maintains an ordered message sequence.     |
| **Segment**   | Physical storage units within partitions.                                                 |
| **Offset**    | Unique sequential ID for messages within a partition.                                     |

### Consumer-Partition Relationship

1. **Consumer > Partitions**: Wasted resources (Kafka prevents concurrent partition access)
2. **Consumer < Partitions**: Single consumer handles multiple partitions (ensure even distribution)
3. Optimal: Partition count = N × Consumers (integer multiple)
4. Order guaranteed **only within partitions**
5. Cluster changes trigger consumer rebalancing

## .NET Integration

### NuGet Package

```powershell
Install-Package Confluent.Kafka
```

### Service Interface

```csharp
public interface IKafkaService
{
    Task PublishAsync<TMessage>(string topic, TMessage message) where TMessage : class;
    Task SubscribeAsync<TMessage>(IEnumerable<string> topics, Action<TMessage> handler, 
                                 CancellationToken cancellationToken) where TMessage : class;
}
```

### Producer Implementation

```csharp
public class KafkaService : IKafkaService
{
    public async Task PublishAsync<TMessage>(string topic, TMessage message) where TMessage : class
    {
        var config = new ProducerConfig { BootstrapServers = "127.0.0.1:9092" };
        
        using var producer = new ProducerBuilder<string, string>(config).Build();
        await producer.ProduceAsync(topic, new Message<string, string>
        {
            Key = Guid.NewGuid().ToString(),
            Value = JsonConvert.SerializeObject(message)
        });
    }
}
```

### Consumer Implementation

```csharp
public class KafkaService : IKafkaService
{
    public async Task SubscribeAsync<TMessage>(IEnumerable<string> topics, Action<TMessage> handler,
                                              CancellationToken cancellationToken) where TMessage : class
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = "127.0.0.1:9092",
            GroupId = "app-consumers",
            EnableAutoCommit = false,
            AutoOffsetReset = AutoOffsetReset.Earliest
        };

        using var consumer = new ConsumerBuilder<Ignore, string>(config)
            .SetErrorHandler((_, e) => Console.WriteLine($"Error: {e.Reason}"))
            .SetPartitionsAssignedHandler((c, partitions) => 
                Console.WriteLine($"Assigned: {string.Join(", ", partitions)}"))
            .Build();
        
        consumer.Subscribe(topics);

        try
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                var result = consumer.Consume(cancellationToken);
                var message = JsonConvert.DeserializeObject<TMessage>(result.Message.Value);
                handler(message);
                consumer.Commit(result);
            }
        }
        catch (OperationCanceledException)
        {
            consumer.Close();
        }
    }
}
```

## Practical Examples

### Producer Console App

```csharp
static async Task Main(string[] args)
{
    var config = new ProducerConfig { BootstrapServers = "localhost:9092" };
    using var producer = new ProducerBuilder<string, string>(config).Build();

    while (true)
    {
        Console.Write("> ");
        var input = Console.ReadLine();
        
        await producer.ProduceAsync("demo-topic", new Message<string, string>
        {
            Key = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds().ToString(),
            Value = input
        });
    }
}
```

### Consumer Console App

```csharp
static void Main(string[] args)
{
    var config = new ConsumerConfig
    {
        BootstrapServers = "localhost:9092",
        GroupId = "console-group",
        AutoOffsetReset = AutoOffsetReset.Earliest
    };

    using var consumer = new ConsumerBuilder<Ignore, string>(config).Build();
    consumer.Subscribe("demo-topic");

    var cts = new CancellationTokenSource();
    Console.CancelKeyPress += (_, e) => cts.Cancel();

    while (!cts.IsCancellationRequested)
    {
        var result = consumer.Consume(cts.Token);
        Console.WriteLine($"Received: {result.Message.Value}");
    }
}
```

---

**Key Takeaways**

1. Always configure `advertised.listeners` for external access
2. Match consumer count to partition count for optimal throughput
3. Use `EnableAutoCommit = false` for at-least-once delivery guarantees
4. Monitor consumer lag in production environments

