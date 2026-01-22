# kcat Commands for Kafka Testing, Management, and Monitoring

This document contains comprehensive kcat commands for testing, managing, and monitoring your Kafka setup.

**Configuration:**
- Broker: `localhost:9092` (via SSH tunnel)
- Topic: `lab02-bbasavar`

---

## **Testing Commands**

### 1. **Produce Test Messages** (Producer Mode)

```bash
# Send a single message
echo '{"city": "Pittsburgh", "timestamp": "2026-01-21 18:44:34", "temperature_f": 64}' | \
kcat -b localhost:9092 -t lab02-bbasavar -P

# Send multiple messages from a file
kcat -b localhost:9092 -t lab02-bbasavar -P -l test_messages.txt

# Send messages interactively (type messages, press Enter, Ctrl+D to finish)
kcat -b localhost:9092 -t lab02-bbasavar -P
```

### 2. **Consume Messages** (Consumer Mode)

```bash
# Consume 5 messages from earliest offset
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 5

# Consume with offset and timestamp
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 5 -f "%o [%T] %s\n"

# Consume continuously (real-time monitoring)
kcat -b localhost:9092 -t lab02-bbasavar -C -o latest

# Consume from a specific offset
kcat -b localhost:9092 -t lab02-bbasavar -C -o 3 -c 10
```

---

## **Management Commands**

### 3. **List Topics**

```bash
# List all available topics
kcat -b localhost:9092 -L | grep "topic"

# Get detailed broker and topic information
kcat -b localhost:9092 -L
```

### 4. **Describe Topic Metadata**

```bash
# Show detailed topic information (partitions, replicas, etc.)
kcat -b localhost:9092 -L -t lab02-bbasavar

# Show partition information
kcat -b localhost:9092 -L | grep -A 10 "lab02-bbasavar"
```

### 5. **Check Topic Partition Details**

```bash
# Get partition count and replication info
kcat -b localhost:9092 -L -t lab02-bbasavar -J | jq '.topics[0].partitions'
```

---

## **Monitoring Commands**

### 6. **Monitor Message Flow (Real-time)**

```bash
# Watch messages as they arrive (with timestamps)
kcat -b localhost:9092 -t lab02-bbasavar -C -o latest -f "%T [%o] %s\n"

# Monitor with partition info
kcat -b localhost:9092 -t lab02-bbasavar -C -o latest -f "Partition %p, Offset %o: %s\n"
```

### 7. **Check Message Count**

```bash
# Count total messages in topic (from beginning)
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -e | wc -l

# Get last N messages
kcat -b localhost:9092 -t lab02-bbasavar -C -o -5 -c 5
```

### 8. **View Offsets**

```bash
# Show offset information for each partition
kcat -b localhost:9092 -L -t lab02-bbasavar

# Consume with detailed offset info
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 10 -f "Offset: %o | Partition: %p | Key: %k | Value: %s\n"
```

### 9. **Test Producer-Consumer Loop**

```bash
# Terminal 1: Start consumer (waiting for messages)
kcat -b localhost:9092 -t lab02-bbasavar -C -o latest -f "%T [%o] %s\n"

# Terminal 2: Send test message
echo '{"city": "Test", "timestamp": "2026-01-21 19:00:00", "temperature_f": 70}' | \
kcat -b localhost:9092 -t lab02-bbasavar -P
```

---

## **Advanced Monitoring**

### 10. **JSON Formatting**

```bash
# Pretty print JSON messages
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 5 -J | jq '.'

# Filter specific fields
kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 10 -J | jq '.payload.city, .payload.temperature_f'
```

### 11. **Check Broker Health**

```bash
# Verify broker connection and get metadata
kcat -b localhost:9092 -L

# Test connection (should return broker info)
kcat -b localhost:9092 -L -q
```

### 12. **Monitor Specific Partitions**

```bash
# Consume from partition 0 only
kcat -b localhost:9092 -t lab02-bbasavar -C -p 0 -o earliest -c 5
```

---

## **Format String Reference**

Useful format specifiers for `-f` parameter:

| Specifier | Description |
|-----------|-------------|
| `%s` | Message value (payload) |
| `%k` | Message key |
| `%o` | Offset |
| `%p` | Partition number |
| `%T` | Timestamp |
| `%t` | Topic name |
| `%h` | Headers |
| `%S` | Message size |

---

## **Quick Test Script**

Save this as `test_kafka.sh`:

```bash
#!/bin/bash
BROKER="localhost:9092"
TOPIC="lab02-bbasavar"

echo "=== Testing Kafka Connection ==="
kcat -b $BROKER -L

echo -e "\n=== Listing Topics ==="
kcat -b $BROKER -L | grep "topic"

echo -e "\n=== Producing Test Message ==="
echo '{"city": "Test", "timestamp": "'$(date +"%Y-%m-%d %H:%M:%S")'", "temperature_f": 75}' | \
kcat -b $BROKER -t $TOPIC -P

echo -e "\n=== Consuming Last 3 Messages ==="
kcat -b $BROKER -t $TOPIC -C -o -3 -c 3 -f "%o: %s\n"
```

**Usage:**
```bash
chmod +x test_kafka.sh
./test_kafka.sh
```

---

## **Common Command Parameters**

| Parameter | Description |
|-----------|-------------|
| `-b localhost:9092` | Broker address (via SSH tunnel) |
| `-t lab02-bbasavar` | Topic name |
| `-C` | Consumer mode (read messages) |
| `-P` | Producer mode (write messages) |
| `-o earliest` | Start from beginning of topic |
| `-o latest` | Start from latest messages only |
| `-o <number>` | Start from specific offset |
| `-c 5` | Consume exactly N messages then exit |
| `-f "%o: %s\n"` | Custom output format |
| `-L` | List metadata (topics, partitions, brokers) |
| `-J` | Output in JSON format |
| `-p 0` | Specify partition number |
| `-e` | Exit after consuming last message |
| `-q` | Quiet mode (less verbose) |

---

## **Tips**

1. **Always verify SSH tunnel is active** before running commands:
   ```bash
   lsof -i :9092
   ```

2. **Use `-o earliest`** to see all messages from the beginning

3. **Use `-o latest`** to only see new messages arriving after you start consuming

4. **Press Ctrl+C** to stop a running consumer

5. **Use `-J` flag with `jq`** for better JSON formatting:
   ```bash
   kcat -b localhost:9092 -t lab02-bbasavar -C -o earliest -c 5 -J | jq '.'
   ```

6. **Monitor in real-time** by using `-o latest` without `-c` flag:
   ```bash
   kcat -b localhost:9092 -t lab02-bbasavar -C -o latest
   ```
