# Implementing a Synchronous Request-Reply Pattern with Apache Kafka and Java

Apache Kafka is designed for high-throughput asynchronous communication, but some use cases require synchronous interactions where a client waits for a response after sending a request. In this article, we’ll explore how to implement a synchronous request-reply pattern using Kafka and Java, complete with correlation IDs and callback handling.

## 1. Introduction to the Request-Reply Pattern
The request-reply pattern enables two-way communication between services in distributed systems. While Kafka primarily excels at one-way streaming, we can implement this pattern using:
* Dedicated request and reply topics
* Correlation IDs to match requests with responses
* Consumer polling with timeout handling
* Synchronization primitives to bridge asynchronous operations

## 2. High-Level Design
Our implementation consists of three main components:
* Kafka Sync Client: Manages request sending and response waiting
* Reply Consumer: Handles response consumption and callback matching
* Response Future: Synchronization mechanism for request/response pairing
## Flow 

```
(Request Flow: Client → Request Topic → Service → Reply Topic → Client)
```

## 3. Key Components Explained
Kafka Sync Client: The core client class providing synchronous communication

```
import com.shr.kafka.async.callback.reply.ReplyConsumer;
import com.shr.kafka.async.callback.response.ResponseFuture;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.header.Headers;
import org.apache.kafka.common.header.internals.RecordHeaders;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.Properties;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

public class KafkaSyncClient<K, V> implements AutoCloseable {
private final Producer<K, V> producer;
private final ReplyConsumer<K, V> replyConsumer;
private final String requestTopic;
private final String replyTopic;

    public KafkaSyncClient(Properties producerProps, Properties consumerProps,
                           String requestTopic, String replyTopic) {
        this.producer = new KafkaProducer<>(producerProps);
        this.replyConsumer = new ReplyConsumer<>(consumerProps, replyTopic);
        this.requestTopic = requestTopic;
        this.replyTopic = replyTopic;
    }

    public V sendAndWait(K key, V request, Duration timeout) throws Exception {
        String correlationId = UUID.randomUUID().toString();
        Headers headers = new RecordHeaders()
                .add("CORRELATION_ID", correlationId.getBytes(StandardCharsets.UTF_8))
                .add("REPLY_TOPIC", replyTopic.getBytes(StandardCharsets.UTF_8));

        ProducerRecord<K, V> record = new ProducerRecord<>(
                requestTopic, null, key, request, headers
        );

        ResponseFuture<V> future = replyConsumer.registerCallback(correlationId);
        producer.send(record);
        return future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
    }

    public void close() {
        producer.close();
        replyConsumer.close();
    }
}
```

### Key features:
* Generates unique correlation IDs for each request
* Attaches metadata in message headers
* Manages producer and consumer lifecycle
* Implements timeout handling

### 3.2 Reply Consumer
The response handler running in a dedicated thread:

```
import com.shr.kafka.async.callback.response.ResponseFuture;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.header.Header;

import java.time.Duration;
import java.util.Collections;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ConcurrentHashMap;

public class ReplyConsumer<K, V> {
private final KafkaConsumer<K, V> consumer;
private final Map<String, ResponseFuture<V>> pendingRequests = new ConcurrentHashMap<>();
private volatile boolean running = true;

    public ReplyConsumer(Properties props, String replyTopic) {
        this.consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList(replyTopic));
        new Thread(this::pollLoop).start();
    }

    public ResponseFuture<V> registerCallback(String correlationId) {
        ResponseFuture<V> future = new ResponseFuture<>();
        pendingRequests.put(correlationId, future);
        return future;
    }

    private void pollLoop() {
        while (running) {
            ConsumerRecords<K, V> records = consumer.poll(Duration.ofMillis(100));

            for (ConsumerRecord<K, V> record : records) {
                Header correlationHeader = record.headers().lastHeader("CORRELATION_ID");

                if (correlationHeader != null) {
                    String correlationId = new String(correlationHeader.value());
                    ResponseFuture<V> future = pendingRequests.remove(correlationId);

                    if (future != null) {
                        future.complete(record.value());
                    }
                }
            }
        }
        consumer.close();
    }
    
    public void close() {
        running = false;
    }
}
```

### Key responsibilities:
* Continuous polling of reply topic
* Correlation ID matching using concurrent map
* Completing futures with received responses
* Thread-safe request management

### 3.3 Response Future
Synchronization primitive for bridging asynchronous operations:

```
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class ResponseFuture<V> {
private final CountDownLatch latch = new CountDownLatch(1);
private V response;
private Exception exception;

    public V get(long timeout, TimeUnit unit) throws Exception {
        if (!latch.await(timeout, unit)) {
            throw new TimeoutException("Response not received within timeout");
        }
        if (exception != null) throw exception;
        return response;
    }

    public void complete(V response) {
        this.response = response;
        latch.countDown();
    }

    public void fail(Exception ex) {
        this.exception = ex;
        latch.countDown();
    }
}
```

## Key characteristics:
* CountDownLatch-based synchronization 
* Timeout handling for system resilience 
* Thread-safe completion mechanism

## 4. Implementation Benefits
Synchronous Semantics: Provides familiar blocking interface for callers
* Correlation Handling: Ensures precise request-response matching
* Timeout Management: Prevents indefinite blocking 
* Resource Efficiency: Single consumer instance handles all responses 
* Scalability: Concurrent map supports high request volumes

## 5. Limitations and Considerations
* Consumer Polling Latency: Maximum delay equals poll interval (100ms in example)
* Memory Management: Unbounded concurrent map could cause memory pressure 
* Error Handling: Needs additional error propagation mechanism 
* Consumer Rebalancing: Requires handling during consumer group changes 

Message Ordering: Does not guarantee request processing order

## 6. Usage Example
   import com.shr.kafka.async.callback.client.KafkaSyncClient;
   import org.apache.kafka.common.serialization.StringDeserializer;
   import org.apache.kafka.common.serialization.StringSerializer;

```
import java.time.Duration;
import java.util.Properties;

public class KafkaAsyncCallbackApplication {
public static void main(String[] args) {
Properties producerProps = new Properties();
producerProps.put("bootstrap.servers", "localhost:9092");
producerProps.put("key.serializer", StringSerializer.class.getName());
producerProps.put("value.serializer", StringSerializer.class.getName());

Properties consumerProps = new Properties();
consumerProps.put("bootstrap.servers", "localhost:9092");
consumerProps.put("group.id", "response-consumer-group");
consumerProps.put("key.deserializer", StringDeserializer.class.getName());
consumerProps.put("value.deserializer", StringDeserializer.class.getName());

try (KafkaSyncClient<String, String> client = new KafkaSyncClient<>(
producerProps,
consumerProps,
"requests",
"responses"
)) {
String response = null;
try {
response = client.sendAndWait("request-key",
"payload", Duration.ofSeconds(30));
} catch (Exception e) {
throw new RuntimeException(e);
}
System.out.println("Received response: " + response);
}
}
}
```

## 7. Production-Ready Enhancements
* Map Eviction: Add TTL for pending requests 
* Error Handling: Track and handle producer errors 
* Metrics: Add monitoring for pending requests and timeouts 
* Serialization: Support multiple serialization formats 
* Retry Logic: Implement retry mechanism for failed requests

## 8. Conclusion
This implementation demonstrates how to build synchronous communication on top of Kafka's asynchronous messaging system. While suitable for many use cases, carefully consider your specific requirements around latency, throughput, and error handling before adopting this pattern.
For further improvement, consider:
* Adding dead-letter queue support
* Implementing request prioritization 
* Adding tracing headers for distributed tracing 
* Supporting multiple response types

The complete code implementation provides a foundation you can extend based on your application's specific needs.
