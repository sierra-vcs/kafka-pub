# Kafka ACL Permissions - Complete Reference

## All Available Operations

### 1. **Topic Operations**

| Operation           | Description                                   | Use Case                             |
| ------------------- | --------------------------------------------- | ------------------------------------ |
| **Read**            | Read messages from topic                      | Consumers reading messages           |
| **Write**           | Produce messages to topic                     | Producers sending messages           |
| **Create**          | Create topics                                 | Applications that auto-create topics |
| **Delete**          | Delete topics                                 | Admin tools, cleanup jobs            |
| **Describe**        | View topic metadata (partitions, configs)     | All clients need this to connect     |
| **DescribeConfigs** | View topic configuration details              | Monitoring, admin tools              |
| **Alter**           | Modify topic settings (partitions, retention) | Changing topic configuration         |
| **AlterConfigs**    | Modify detailed topic configs                 | Advanced admin operations            |

### 2. **Consumer Group Operations**

| Operation    | Description              | Use Case                  |
| ------------ | ------------------------ | ------------------------- |
| **Read**     | Read from consumer group | Consumers joining a group |
| **Describe** | View group metadata      | Monitoring consumer lag   |
| **Delete**   | Delete consumer group    | Cleanup, reset offsets    |

### 3. **Cluster Operations**

| Operation           | Description                     | Use Case                    |
| ------------------- | ------------------------------- | --------------------------- |
| **Create**          | Create topics on cluster        | Auto-topic creation         |
| **Alter**           | Modify cluster configs          | Changing broker settings    |
| **AlterConfigs**    | Modify detailed cluster configs | Advanced cluster management |
| **ClusterAction**   | Perform cluster-level actions   | Administrative operations   |
| **Describe**        | View cluster metadata           | Monitoring, discovery       |
| **DescribeConfigs** | View cluster configuration      | Monitoring tools            |
| **IdempotentWrite** | Enable idempotent producer      | Exactly-once semantics      |

### 4. **Transactional Operations**

| Operation    | Description               | Use Case                |
| ------------ | ------------------------- | ----------------------- |
| **Describe** | View transaction state    | Transactional producers |
| **Write**    | Write transaction markers | Transactional producers |

### 5. **Special Operations**

| Operation | Description    | Use Case                    |
| --------- | -------------- | --------------------------- |
| **All**   | All operations | Full access (use sparingly) |

---

## Common Permission Patterns

### 1. **Basic Producer (Write Only)**

```powershell
# Write to specific topic
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:producer1 `
  --operation Write `
  --operation Describe `
  --topic orders
```

### 2. **Idempotent Producer (Exactly-Once)**

```powershell
# Topic-level permissions
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:producer1 `
  --operation Write `
  --operation Describe `
  --topic orders

# Cluster-level permission for idempotence
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:producer1 `
  --operation IdempotentWrite `
  --cluster
```

### 3. **Basic Consumer (Read Only)**

```powershell
# Read from topic
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:consumer1 `
  --operation Read `
  --operation Describe `
  --topic orders

# Read from consumer group
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:consumer1 `
  --operation Read `
  --group my-consumer-group
```

### 4. **Producer with Auto-Create Topics**

```powershell
# Create topics
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:app-service `
  --operation Create `
  --operation Describe `
  --cluster

# Write to topics
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:app-service `
  --operation Write `
  --operation Describe `
  --topic app- `
  --resource-pattern-type prefixed
```

### 5. **Topic Administrator**

```powershell
# Cluster-level permissions
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:topic-admin `
  --operation Create `
  --operation Delete `
  --operation Describe `
  --operation Alter `
  --cluster

# All operations on all topics
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:topic-admin `
  --operation All `
  --topic '*'
```

### 6. **Consumer Group Administrator**

```powershell
# Manage all consumer groups
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:group-admin `
  --operation Describe `
  --operation Delete `
  --group '*'
```

### 7. **Monitoring/Read-Only User**

```powershell
# Describe topics
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:monitor `
  --operation Describe `
  --operation DescribeConfigs `
  --topic '*'

# Describe consumer groups
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:monitor `
  --operation Describe `
  --group '*'

# Describe cluster
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:monitor `
  --operation Describe `
  --operation DescribeConfigs `
  --cluster
```

### 8. **Transactional Producer**

```powershell
# Topic permissions
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:transactional-producer `
  --operation Write `
  --operation Describe `
  --topic orders

# Transactional ID
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:transactional-producer `
  --operation Write `
  --operation Describe `
  --transactional-id my-transactional-id

# IdempotentWrite
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:transactional-producer `
  --operation IdempotentWrite `
  --cluster
```

### 9. **Kafka Streams Application**

```powershell
# Read input topics
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:streams-app `
  --operation Read `
  --operation Describe `
  --topic input-topic

# Write output topics
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:streams-app `
  --operation Write `
  --operation Describe `
  --topic output-topic

# Create internal topics
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:streams-app `
  --operation Create `
  --cluster

# Write to internal topics (with prefix)
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:streams-app `
  --operation All `
  --topic streams-app- `
  --resource-pattern-type prefixed

# Consumer group
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:streams-app `
  --operation Read `
  --group streams-app-group

# IdempotentWrite
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:streams-app `
  --operation IdempotentWrite `
  --cluster
```

### 10. **Kafka Connect Worker**

```powershell
# Create topics for offsets, configs, status
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:connect-worker `
  --operation Create `
  --cluster

# Read/Write internal topics
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:connect-worker `
  --operation All `
  --topic connect- `
  --resource-pattern-type prefixed

# Consumer group
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:connect-worker `
  --operation Read `
  --group connect-cluster
```

---

## Resource Pattern Types

| Pattern Type | Description           | Example                                                                           |
| ------------ | --------------------- | --------------------------------------------------------------------------------- |
| **LITERAL**  | Exact match (default) | `--topic orders` matches only "orders"                                            |
| **PREFIXED** | Prefix match          | `--topic app- --resource-pattern-type prefixed` matches "app-orders", "app-users" |
| **ANY**      | Matches all resources | Rarely used                                                                       |

---

## Permission Types

| Type      | Description                                              |
| --------- | -------------------------------------------------------- |
| **ALLOW** | Grant permission (default)                               |
| **DENY**  | Explicitly deny permission (takes precedence over ALLOW) |

```powershell
# Allow
.\bin\windows\kafka-acls.bat --add --allow-principal User:user1 ...

# Deny
.\bin\windows\kafka-acls.bat --add --deny-principal User:user1 ...
```

---

## Host-Based ACLs

You can restrict access by IP address:

```powershell
# Allow only from specific IP
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:producer1 `
  --allow-host 192.168.1.100 `
  --operation Write `
  --topic orders

# Allow from any host (default)
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --add --allow-principal User:producer1 `
  --allow-host '*' `
  --operation Write `
  --topic orders
```

---

## Viewing ACLs

```powershell
# List all ACLs
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --list

# List ACLs for specific user
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --list `
  --principal User:producer1

# List ACLs for specific topic
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --list `
  --topic orders

# List ACLs for consumer group
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --list `
  --group my-group
```

---

## Removing ACLs

```powershell
# Remove specific ACL
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --remove `
  --allow-principal User:producer1 `
  --operation Write `
  --topic orders

# Remove all ACLs for a user
.\bin\windows\kafka-acls.bat --bootstrap-server localhost:9092 `
  --command-config admin.properties `
  --remove `
  --principal User:producer1 `
  --force
```

---

## Quick Reference Matrix

| User Type               | Topic Read | Topic Write | Topic Create | Topic Delete | Group Read | Cluster IdempotentWrite |
| ----------------------- | ---------- | ----------- | ------------ | ------------ | ---------- | ----------------------- |
| **Basic Producer**      | ❌         | ✅          | ❌           | ❌           | ❌         | ❌                      |
| **Idempotent Producer** | ❌         | ✅          | ❌           | ❌           | ❌         | ✅                      |
| **Basic Consumer**      | ✅         | ❌          | ❌           | ❌           | ✅         | ❌                      |
| **App Service**         | ❌         | ✅          | ✅           | ❌           | ❌         | ✅                      |
| **Topic Admin**         | ✅         | ✅          | ✅           | ✅           | ❌         | ❌                      |
| **Monitor/Read-Only**   | Describe   | Describe    | ❌           | ❌           | Describe   | ❌                      |
| **Super Admin**         | ✅         | ✅          | ✅           | ✅           | ✅         | ✅                      |
