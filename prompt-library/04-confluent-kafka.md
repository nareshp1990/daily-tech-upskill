# Confluent Kafka — Daily Prompts

> Tech stack: Confluent Kafka (managed), Spring Kafka, Avro, Confluent Schema Registry

---

## Producer Implementation

### Create a Kafka producer
```
Implement a Kafka producer in Spring Boot for [event, e.g., "OrderCreated"].

Requirements:
- KafkaTemplate with Avro serialization (Confluent Schema Registry)
- Avro schema (.avsc) for the event with [list fields]
- ProducerConfig: acks=all, retries=3, idempotence=true, linger.ms=5
- Partition key strategy: [describe — e.g., by userId for ordering]
- Headers: correlationId, eventType, timestamp, source
- Success/failure callbacks with logging
- Transactional outbox pattern (optional: produce after DB commit)
- application.yml with Confluent Cloud/Confluent Platform connection

Show: Avro schema, config class, producer service, and example usage.
```

### Transactional outbox pattern
```
Implement the Transactional Outbox pattern for [use case] with Spring Boot + MySQL + Kafka.

Problem: I need to save to DB AND publish to Kafka atomically.

Show:
1. Outbox table DDL (id, aggregate_type, aggregate_id, event_type, payload JSON,
   created_at, published_at)
2. Service: save entity + outbox record in same @Transactional
3. Outbox poller: @Scheduled job reads unpublished rows, publishes to Kafka,
   marks as published
4. Distributed lock (ShedLock) so only one pod polls
5. Cleanup: purge published records older than [X] days
6. Idempotency on consumer side (in case of duplicate publish)

Show all code files.
```

---

## Consumer Implementation

### Create a Kafka consumer
```
Implement a Kafka consumer in Spring Boot for topic [topic_name].

Requirements:
- @KafkaListener with Avro deserialization
- Consumer group: [group-id]
- Concurrency: [N] threads
- Manual ack (AckMode.MANUAL_IMMEDIATE) for at-least-once
- Error handling:
  • Deserialization error → log and skip (ErrorHandlingDeserializer)
  • Business logic error → retry 3x with backoff, then send to DLT
  • Non-retryable errors (validation) → straight to DLT
- Dead Letter Topic (DLT) with original headers preserved
- Idempotent processing (check processed_events table before handling)
- Structured logging: topic, partition, offset, key, correlationId

Show: config, listener class, error handler, and DLT consumer.
```

### Handle poison pill messages
```
My Kafka consumer is failing on a malformed message (poison pill) and restarting in a loop.

Topic: [topic], Consumer group: [group], Error: [paste error]

Walk me through:
1. Immediate fix: skip the bad message (seek past it)
   - Using kafka-consumer-groups CLI to reset offset
   - Or using ErrorHandlingDeserializer to route to DLT
2. Prevention: configure ErrorHandlingDeserializer in Spring Kafka
3. DLT setup for capturing bad messages
4. Monitoring: alert when DLT has messages
5. Tooling: how to inspect the bad message (kafkacat/kcat, Confluent CLI)

Show the configuration changes needed.
```

### Consumer lag troubleshooting
```
My Kafka consumer is lagging behind. Consumer group: [group], topic: [topic].
Current lag: [X messages], normal lag: [Y].

Help me diagnose:
1. Check consumer lag: kafka-consumer-groups --describe (show command)
2. Common causes:
   - Slow processing (DB calls, external API, blocking I/O)
   - Too few partitions for consumer instances
   - Rebalancing storms
   - GC pauses (max.poll.interval.ms timeout)
3. Fix strategies:
   - Increase partitions + consumers
   - Batch processing
   - Async processing within consumer
   - Tune: max.poll.records, max.poll.interval.ms, fetch.min.bytes
4. How to verify: metrics to monitor in Confluent Cloud/Control Center

Show the specific config changes with before/after values.
```

---

## Schema Management

### Design an Avro schema
```
Design an Avro schema for [event] with these fields:
[list fields with types]

Requirements:
- Namespace: com.company.events.[domain]
- Use logical types: timestamp-millis for dates, decimal for money
- Default values for optional fields (for backward compatibility)
- Include metadata fields: eventId (UUID), eventType, timestamp, version, source
- Follow Confluent Schema Registry compatibility rules (BACKWARD by default)

Show:
- .avsc file
- How to register with Schema Registry (Maven plugin or curl)
- Compatibility check command
```

### Schema evolution
```
I need to evolve the Avro schema for [event] on topic [topic].

Current schema: [paste or describe current fields]
Change needed: [add field / remove field / rename / change type]

Confluent Schema Registry compatibility mode: [BACKWARD / FORWARD / FULL]

Walk me through:
1. Is this change compatible under [mode]? Why or why not?
2. If not, what's the migration strategy? (new topic? new version field?)
3. The new .avsc with proper defaults for backward compatibility
4. How to test compatibility before deploying:
   curl -X POST schema-registry/compatibility/subjects/[subject]/versions/latest
5. Consumer-side handling of both old and new schema versions
```

---

## Topic Management

### Create/configure a topic
```
I need a Kafka topic for [use case, e.g., "order events"].

Help me choose:
- Topic name: following convention [domain].[entity].[event] (e.g., orders.order.created)
- Partitions: [expected throughput] messages/sec, [N] consumer instances
  → calculate optimal partition count
- Replication factor: 3 (Confluent Cloud default)
- Retention: [time-based vs size-based, e.g., 7 days vs 1GB]
- Cleanup policy: delete vs compact (explain when to use each)
- min.insync.replicas: 2
- Compression: lz4 or zstd

Show: kafka-topics CLI command and Spring Boot NewTopic @Bean.
```

---

## Confluent-Specific

### Confluent Cloud connection
```
Generate Spring Boot application.yml for connecting to Confluent Cloud.

Include:
- Bootstrap servers (from Confluent Cloud cluster settings)
- SASL_SSL authentication (API key + secret)
- Schema Registry URL with basic auth
- Producer config (acks, retries, idempotence)
- Consumer config (group-id, auto-offset-reset, isolation-level)
- Separate profiles: dev (local Confluent Platform via Docker) and prod (Confluent Cloud)
- Sensitive values referenced from environment variables or Azure Key Vault

Show both profiles.
```

---

## Monitoring & Operations

### Kafka health check
```
Create a Kafka health check for my Spring Boot app running on AKS.

Requirements:
- Custom HealthIndicator that verifies:
  • Can reach Kafka brokers
  • Can reach Schema Registry
  • Consumer group is assigned partitions
  • Consumer lag is below threshold
- Integrate with Kubernetes readiness probe (/actuator/health/readiness)
- Don't mark unhealthy during rebalance (temporary state)

Show the HealthIndicator and actuator configuration.
```

---

## Quick One-Liners

```
Show me how to consume from a specific offset in Spring Kafka (seek to offset [X]
on partition [Y] of topic [Z]).
```

```
How do I test a @KafkaListener with EmbeddedKafka? Show minimal test class.
```

```
Write a Spring Kafka interceptor that adds traceId and spanId headers to every produced message.
```

```
What's the difference between KafkaTemplate.send() and KafkaTemplate.sendDefault()?
When should I use each?
```

```
Show me how to replay messages from a DLT back to the original topic using Spring Kafka.
```

```
Generate a docker-compose.yml for a local Confluent Platform (broker, schema-registry,
control-center, connect) for development.
```

---

*Kafka is easy to write to, hard to consume correctly. Always plan for poison pills, lag, and rebalances.*
