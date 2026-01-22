# Lab 2: Kafka Insights and Deliverables

## Deliverable 1: SSH Tunnel, Topics, and Offsets

### Establishing SSH Tunnel

To establish a secure SSH tunnel to the Kafka server, use the following command:

```bash
ssh -L 9092:localhost:<remote_port> <user>@<remote_server> -NTf
```

**Parameters:**
- `-L 9092:localhost:<remote_port>`: Creates a local port forward from localhost:9092 to the remote Kafka port
- `-N`: No remote command execution
- `-T`: Disable pseudo-terminal allocation
- `-f`: Run in background

**Verification:**
```bash
lsof -i :9092  # Should show an ssh process
```

### Concepts: Topics and Offsets

#### **Topics**

A **topic** in Kafka is a category or feed name to which messages are published. Think of it as a named stream of data.

**Key characteristics:**
- Topics are partitioned for scalability and parallelism
- Multiple producers can write to the same topic
- Multiple consumers can read from the same topic
- Topics are immutable - messages are appended, never modified or deleted (within retention period)
- Each topic can have multiple partitions, allowing parallel processing

**Example:** In our lab, we created topic `lab02-bbasavar` to store city temperature data. Each student has their own topic to avoid message collisions.

#### **Offsets**

An **offset** is a unique, sequential identifier assigned to each message within a partition of a topic. It acts as a position marker in the message stream.

**Key characteristics:**
- Offsets start from 0 and increment sequentially
- Offsets are per-partition (not global across the topic)
- Offsets are immutable - once assigned, they never change
- Kafka tracks the last committed offset for each consumer group

**Example:** If a topic has messages with offsets 0, 1, 2, 3, 4, 5:
- Offset 0 is the first message
- Offset 5 is the sixth message
- The next message will be offset 6

#### **Message Continuity Through Offsets**

Offsets ensure message continuity when a consumer disconnects through the following mechanism:

1. **Offset Committing**: When a consumer reads a message, it commits the offset to Kafka, indicating "I've processed up to this point."

2. **Offset Storage**: Kafka stores the last committed offset for each consumer group in a special topic called `__consumer_offsets`.

3. **Resume on Reconnect**: When a consumer reconnects:
   - Kafka checks the last committed offset for that consumer group
   - The consumer resumes reading from the next unread message (last committed offset + 1)
   - This ensures no messages are lost and no messages are processed twice

**Example Scenario:**
```
Consumer reads messages: 0, 1, 2, 3, 4
Consumer commits offset: 4
Consumer disconnects (network issue, crash, etc.)
Consumer reconnects
Kafka resumes from offset: 5 (next unread message)
```

**Benefits:**
- **No message loss**: Even if a consumer crashes, it can resume from where it left off
- **No duplicate processing**: Messages already processed won't be re-read
- **Fault tolerance**: System can handle consumer failures gracefully
- **Scalability**: Multiple consumers in a group can process different partitions in parallel

---

## Deliverable 2: Producer and Consumer Implementation

### Producer Implementation

**Code:**
```python
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: dumps(v).encode('utf-8')
)

# Send messages
for i in range(10):
    data = cities[randint(0, len(cities)-1)]
    producer.send(topic=topic, value=data)
    sleep(1)

producer.flush()  # Ensure all messages are sent
```

**Key points:**
- `value_serializer`: Converts Python dicts to JSON bytes (Kafka requirement)
- `producer.send()`: Asynchronous - queues messages but doesn't wait
- `producer.flush()`: Blocks until all queued messages are sent

### Consumer Implementation

**Code:**
```python
consumer = KafkaConsumer(
    topic,
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',
    enable_auto_commit=True,
    auto_commit_interval_ms=1000
)

for message in consumer:
    message_str = message.value.decode('utf-8')
    message_dict = loads(message_str)
    print(message_dict)
```

### Tradeoffs of `auto_offset_reset` Values

The `auto_offset_reset` parameter controls where a consumer starts reading when there is no stored offset for the consumer group.

#### **1. `auto_offset_reset='earliest'`**

**Behavior:** Start reading from the beginning of the topic (offset 0).

**Use cases:**
- Processing all historical data
- Initial data load
- Testing and development
- When you need to see all messages

**Tradeoffs:**
- ✅ **Pros**: 
  - Reads all available messages
  - Useful for batch processing
  - Good for initial setup
- ❌ **Cons**: 
  - May process very old messages
  - Can be slow if topic has millions of messages
  - May reprocess data if consumer group is reset

**Example:** In our lab, we used `'earliest'` to read all 10 messages we sent, even if the consumer started after the producer finished.

#### **2. `auto_offset_reset='latest'`**

**Behavior:** Start reading only new messages that arrive after the consumer starts.

**Use cases:**
- Real-time streaming applications
- Live data processing
- When you only care about new data
- Production systems processing current events

**Tradeoffs:**
- ✅ **Pros**: 
  - Only processes new messages
  - Fast startup (no historical data)
  - Efficient for real-time systems
- ❌ **Cons**: 
  - Misses historical messages
  - Not suitable for batch processing
  - May miss data if consumer starts before producer

**Example:** If you set `'latest'` and start the consumer before the producer, you'll see all new messages. But if you start after, you might miss some messages if they were sent before the consumer connected.

#### **3. `auto_offset_reset='none'`**

**Behavior:** Throw an error if no offset is found for the consumer group.

**Use cases:**
- Strict data processing requirements
- When you want explicit control over offset management
- Debugging offset issues
- Ensuring no accidental data processing

**Tradeoffs:**
- ✅ **Pros**: 
  - Explicit error handling
  - Prevents accidental data processing
  - Forces you to manage offsets manually
- ❌ **Cons**: 
  - Requires manual offset management
  - Consumer will fail if no offset exists
  - More complex to implement

**Example:** If you use `'none'` and there's no stored offset, the consumer will raise an exception instead of starting to read.

### Recommendation

- **Development/Testing**: Use `'earliest'` to see all data
- **Production (Real-time)**: Use `'latest'` for current data only
- **Production (Batch)**: Use `'earliest'` or manage offsets manually
- **Critical Systems**: Use `'none'` with explicit offset management

---

## Deliverable 3: Using kcat CLI Tool

### Installation

**macOS:**
```bash
brew install kcat
```

**Ubuntu/Debian:**
```bash
sudo apt-get install kcat
```

### Listing All Topics

**Command:**
```bash
kcat -b localhost:9092 -L
```

**Output:** Lists all topics with their partition information, including:
- Topic names
- Number of partitions
- Leader broker information
- Replica information

**Example output:**
```
Metadata for all topics (from broker 1: localhost:9092/1):
 1 brokers:
  broker 1 at localhost:9092 (controller)
 249 topics:
  topic "lab02-bbasavar" with 1 partitions:
    partition 0, leader 1, replicas: 1, isrs: 1
  topic "movielog1" with 2 partitions:
    partition 0, leader 1, replicas: 1, isrs: 1
    partition 1, leader 1, replicas: 1, isrs: 1
```

**Use cases:**
- Discovering available topics
- Verifying topic creation
- Checking partition configuration
- Monitoring topic metadata

### Consuming Messages

#### **Basic Consumption**

**Command:**
```bash
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 5
```

**Parameters:**
- `-b localhost:9092`: Broker address
- `-t lab02-bbasavar`: Topic name
- `-C`: Consumer mode
- `-o earliest`: Start from beginning
- `-c 5`: Consume 5 messages then exit

#### **Consuming with Offset Display**

**Command:**
```bash
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 5 -f "%o: %s\n"
```

**Format string:**
- `%o`: Offset number
- `%s`: Message content
- `\n`: Newline

**Example output:**
```
0: {"city": "Pittsburgh", "timestamp": "2026-01-22 15:09:20", "temperature_f": 64}
1: {"city": "New York", "timestamp": "2026-01-22 15:09:20", "temperature_f": 72}
2: {"city": "Los Angeles", "timestamp": "2026-01-22 15:09:20", "temperature_f": 78}
3: {"city": "Chicago", "timestamp": "2026-01-22 15:09:20", "temperature_f": 58}
4: {"city": "Mumbai", "timestamp": "2026-01-22 15:09:20", "temperature_f": 84}
```

#### **Consuming from Specific Partition**

**Command:**
```bash
kcat -b localhost:9092 -t movielog1 -C -o earliest -c 5 -p 0
```

**Parameters:**
- `-p 0`: Read only from partition 0

**Use case:** When a topic has multiple partitions and you want to read from a specific one.

#### **Consuming Latest Messages**

**Command:**
```bash
kcat -b localhost:9092 -t lab02-bbasavar -C -o latest -c 10
```

**Use case:** Read only new messages arriving after the command starts.

### Producing Messages

**Command:**
```bash
echo '{"city": "Seattle", "temperature_f": 65}' | kcat -b localhost:9092 -t lab02-bbasavar -P
```

**Parameters:**
- `-P`: Producer mode
- Input from stdin or file

### Advanced kcat Usage

#### **Reading from File**

**Command:**
```bash
kcat -b localhost:9092 -t lab02-bbasavar -P -l messages.json
```

**Parameters:**
- `-l`: Read messages from file (one per line)

#### **JSON Formatting**

**Command:**
```bash
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 5 -J
```

**Parameters:**
- `-J`: Output in JSON format with metadata

#### **Key-Value Separation**

**Command:**
```bash
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 5 -K:
```

**Parameters:**
- `-K:`: Use `:` as key-value separator

### Monitoring and Management

**Key kcat commands for monitoring:**

1. **List topics:** `kcat -b localhost:9092 -L`
2. **Consume with metadata:** `kcat -b localhost:9092 -t <topic> -C -J`
3. **Check topic offsets:** `kcat -b localhost:9092 -t <topic> -L`
4. **Produce test messages:** `echo "message" | kcat -b localhost:9092 -t <topic> -P`

### Benefits of kcat

- **Quick debugging**: Fast way to inspect Kafka topics
- **No code required**: CLI tool for quick checks
- **Flexible**: Supports various output formats
- **Lightweight**: Simple tool for basic operations
- **Useful for testing**: Verify producer/consumer behavior

### Example: Exploring Movie Log Streams

For the group project, you can explore movielog streams:

```bash
# List all topics
kcat -b localhost:9092 -L | grep movielog

# Read from movielog1
kcat -b localhost:9092 -t movielog1 -C -o latest -c 10

# Read with offset display
kcat -b localhost:9092 -t movielog2 -C -o earliest -c 5 -f "%o: %s\n"
```

This helps understand the data structure before implementing your group project consumer.

---

## Summary

1. **Topics and Offsets**: Topics are named streams of data, and offsets are position markers that enable message continuity through offset committing and resumption.

2. **auto_offset_reset Tradeoffs**: 
   - `'earliest'`: All data, but may be slow
   - `'latest'`: Only new data, but misses history
   - `'none'`: Explicit control, but requires manual management

3. **kcat Usage**: Powerful CLI tool for listing topics, consuming/producing messages, and monitoring Kafka clusters without writing code.
