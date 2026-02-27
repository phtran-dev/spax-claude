# SPEC-06: Queue Abstraction + Redis Streams Implementation

## Mục tiêu
Implement `IngestQueue` interface abstraction và Redis Streams implementation. Abstraction cho phép swap sang WAL implementation sau này chỉ bằng đổi config, không sửa business logic.

## Dependencies
- SPEC-01 (project setup, dependency spring-boot-starter-data-redis)

## Files cần tạo

### 1. `src/main/java/com/spax/queue/IngestMessage.java`

```java
public record IngestMessage(
    String filePath,      // absolute path trên temp disk
    String tenantCode,    // tenant để route đúng schema
    Instant receivedAt    // thời điểm nhận file
) {
    // Serialize/deserialize thủ công cho Redis (Map<String, String>)
    public Map<String, String> toMap() {
        return Map.of(
            "filePath", filePath,
            "tenantCode", tenantCode,
            "receivedAt", receivedAt.toString()
        );
    }

    public static IngestMessage fromMap(Map<String, String> map) {
        return new IngestMessage(
            map.get("filePath"),
            map.get("tenantCode"),
            Instant.parse(map.get("receivedAt"))
        );
    }
}
```

### 2. `src/main/java/com/spax/queue/IngestQueue.java`

```java
public interface IngestQueue {
    /**
     * Publish one message to the queue.
     * Non-blocking, must be fast.
     */
    void publish(IngestMessage message);

    /**
     * Consume batch messages. Blocks until at least 1 message available or timeout.
     * Handler receives batch, must process all. Queue ACKs automatically on success.
     * On handler exception: messages remain unACKed, will be redelivered.
     *
     * @param batchSize max messages per call
     * @param handler   process the batch (must be safe to retry)
     */
    void consume(int batchSize, Consumer<List<IngestMessage>> handler);

    /**
     * Get pending (unprocessed) message count for monitoring.
     */
    long getPendingCount(String tenantCode);
}
```

### 3. `src/main/java/com/spax/queue/RedisStreamIngestQueue.java`

Redis Streams implementation:

```java
@Component
@ConditionalOnProperty(name = "spax.queue.type", havingValue = "redis", matchIfMissing = true)
public class RedisStreamIngestQueue implements IngestQueue {

    private static final String CONSUMER_GROUP = "indexer-group";
    private static final String CONSUMER_NAME = "consumer-" + ManagementFactory.getRuntimeMXBean().getName();

    @Autowired private RedisTemplate<String, String> redisTemplate;
    @Autowired private StringRedisTemplate stringRedisTemplate;

    private StreamOperations<String, String, String> streamOps() {
        return stringRedisTemplate.opsForStream();
    }

    private String streamKey(String tenantCode) {
        return "spax:ingest:" + tenantCode;
    }

    @Override
    public void publish(IngestMessage message) {
        String key = streamKey(message.tenantCode());
        MapRecord<String, String, String> record = StreamRecords.newRecord()
            .ofMap(message.toMap())
            .withStreamKey(key);
        streamOps().add(record);
    }

    @Override
    public void consume(int batchSize, Consumer<List<IngestMessage>> handler) {
        // This method is called in a loop by IndexingConsumer.
        // Reads from ALL tenant streams (need to query which tenants exist).
        // Simplified: reads from a single "all tenants" stream or uses a known tenant list.

        // In practice: IndexingConsumer will call this per tenant.
        // For now, implement a single-tenant version:
        throw new UnsupportedOperationException("Use consumeForTenant() instead");
    }

    /**
     * Consume messages for a specific tenant's stream.
     * Creates consumer group if not exists.
     */
    public void consumeForTenant(String tenantCode, int batchSize, Consumer<List<IngestMessage>> handler) {
        String key = streamKey(tenantCode);
        ensureConsumerGroup(key);

        // Read pending messages first (from previous crashed consumer)
        List<MapRecord<String, String, String>> pending = streamOps().read(
            Consumer.from(CONSUMER_GROUP, CONSUMER_NAME),
            StreamReadOptions.empty().count(batchSize).block(Duration.ofSeconds(2)),
            StreamOffset.create(key, ReadOffset.lastConsumed())
        );

        if (pending == null || pending.isEmpty()) return;

        List<IngestMessage> messages = pending.stream()
            .map(r -> IngestMessage.fromMap(r.getValue()))
            .toList();

        // Call handler — if it throws, messages won't be ACKed
        handler.accept(messages);

        // ACK all processed messages
        RecordId[] ids = pending.stream()
            .map(MapRecord::getId)
            .toArray(RecordId[]::new);
        streamOps().acknowledge(key, CONSUMER_GROUP, ids);
    }

    private void ensureConsumerGroup(String key) {
        try {
            streamOps().createGroup(key, ReadOffset.from("0"), CONSUMER_GROUP);
        } catch (Exception e) {
            // Group already exists — ignore
            if (!e.getMessage().contains("BUSYGROUP")) throw e;
        }
    }

    @Override
    public long getPendingCount(String tenantCode) {
        PendingMessagesSummary summary = streamOps()
            .pending(streamKey(tenantCode), CONSUMER_GROUP);
        return summary != null ? summary.getTotalPendingMessages() : 0;
    }
}
```

### 4. `src/main/java/com/spax/queue/WalIngestQueue.java`

Placeholder implementation (future):

```java
@Component
@ConditionalOnProperty(name = "spax.queue.type", havingValue = "wal")
public class WalIngestQueue implements IngestQueue {
    // TODO: implement file-based WAL queue
    // For now, fail fast to signal this is not ready
    @Override
    public void publish(IngestMessage message) {
        throw new UnsupportedOperationException("WAL queue not yet implemented");
    }

    @Override
    public void consume(int batchSize, Consumer<List<IngestMessage>> handler) {
        throw new UnsupportedOperationException("WAL queue not yet implemented");
    }

    @Override
    public long getPendingCount(String tenantCode) { return 0; }
}
```

### 5. `src/main/java/com/spax/config/RedisConfig.java`

```java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new StringRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }
}
```

## Lưu ý quan trọng
- `CONSUMER_NAME` phải unique per JVM instance — dùng PID từ `ManagementFactory`
- Consumer group phải được tạo trước khi consume. `ensureConsumerGroup()` idempotent.
- `ReadOffset.lastConsumed()` = `">"` trong Redis Streams — chỉ lấy messages chưa delivered đến group này
- Khi consumer crash, messages ở trạng thái PENDING. Next startup đọc lại với `ReadOffset.from("0")` của pending list → không mất message
- Handler phải idempotent (có thể gọi lại nếu crash sau process nhưng trước ACK)

## Kiểm tra thành công
- Publish 10 messages → pending count = 10
- Start consumer → consume 10 messages → handler được gọi → pending count = 0
- Publish 5 → consumer crash trước ACK → restart → messages được redelivered
