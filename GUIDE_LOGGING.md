# Hướng Dẫn Chi Tiết về Logging trong Dự Án

## Mục Lục
1. [Tổng Quan về Logging](#tổng-quan-về-logging)
2. [Các Nơi Ghi Logs](#các-nơi-ghi-logs)
3. [Cấu Hình Logging](#cấu-hình-logging)
4. [Logging Pipeline Behavior](#logging-pipeline-behavior)
5. [Structured Logging](#structured-logging)
6. [OpenTelemetry Logging](#opentelemetry-logging)
7. [Log Aggregation và Storage](#log-aggregation-và-storage)
8. [Best Practices](#best-practices)

---

## Tổng Quan về Logging

Dự án sử dụng **SLF4J** (Simple Logging Facade for Java) với **Logback** làm implementation mặc định của Spring Boot. Logs được thu thập và gửi đến **OpenTelemetry Collector**, sau đó được lưu trữ trong **Loki** và **Elasticsearch**.

### Kiến Trúc Logging

```
┌─────────────────────────────────────────────────────────────┐
│  Application Services                                       │
│  (Flight, Passenger, Booking)                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ SLF4J Logger                                         │  │
│  │ - LogPipelineBehavior                                │  │
│  │ - CustomProblemDetailsHandler                        │  │
│  │ - PersistMessageProcessor                            │  │
│  └──────────────────┬───────────────────────────────────┘  │
│                     │                                      │
│                     ▼                                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ OpenTelemetry SDK                                    │  │
│  │ - OtlpGrpcLogRecordExporter                          │  │
│  └──────────────────┬───────────────────────────────────┘  │
└──────────────────────┼──────────────────────────────────────┘
                       │ OTLP (gRPC/HTTP)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  OpenTelemetry Collector                                     │
│  - Receives logs via OTLP                                   │
│  - Processes and batches logs                                │
│  - Exports to multiple destinations                          │
└──────────┬───────────────────────┬─────────────────────────┘
           │                       │
           ▼                       ▼
┌─────────────────────┐  ┌─────────────────────┐
│  Loki                │  │  Elasticsearch     │
│  (Log Aggregation)   │  │  (Search & Index)  │
└─────────────────────┘  └─────────────────────┘
           │                       │
           └───────────┬───────────┘
                       ▼
              ┌─────────────────┐
              │  Grafana/Kibana  │
              │  (Visualization) │
              └─────────────────┘
```

---

## Các Nơi Ghi Logs

### 1. **LogPipelineBehavior** - Logging Tự Động cho Mọi Request

**Vị trí**: `buildingblocks/mediator/behaviors/LogPipelineBehavior.java`

Đây là **pipeline behavior** tự động log mọi request đi qua Mediator:

```java
public class LogPipelineBehavior<TRequest extends IRequest<TResponse>, TResponse>
        implements IPipelineBehavior<TRequest, TResponse> {

    private static final Logger logger = LoggerFactory.getLogger(LogPipelineBehavior.class);

    @Override
    public TResponse handle(TRequest request, RequestHandlerDelegate<TResponse> next) {
        long startTime = System.currentTimeMillis();

        // Log request bắt đầu
        logger.info("Handling request of type: {}", request.getClass().getSimpleName());
        logger.debug("Request details: {}", request);

        TResponse response;

        try {
            response = next.handle();
        } catch (Exception ex) {
            // Log lỗi
            logger.error(
                    "Error occurred while handling request of type {}",
                    request.getClass().getSimpleName(),
                    ex);
            throw ex;
        } finally {
            long endTime = System.currentTimeMillis();
            long executionTime = endTime - startTime;

            // Log thời gian xử lý
            logger.info(
                    "Request of type {} handled in {} ms", 
                    request.getClass().getSimpleName(), 
                    executionTime);
        }

        logger.debug("Response details: {}", response);
        return response;
    }
}
```

**Logs được ghi**:
- ✅ **INFO**: Bắt đầu xử lý request (tên request type)
- ✅ **DEBUG**: Chi tiết request (chỉ khi log level = DEBUG)
- ✅ **ERROR**: Lỗi xảy ra (kèm exception)
- ✅ **INFO**: Thời gian xử lý (milliseconds)
- ✅ **DEBUG**: Chi tiết response (chỉ khi log level = DEBUG)

**Ví dụ log output**:
```
INFO  - Handling request of type: CreateBookingCommand
DEBUG - Request details: CreateBookingCommand[id=..., flightId=..., passengerId=...]
INFO  - Request of type CreateBookingCommand handled in 245 ms
DEBUG - Response details: BookingDto[id=..., ...]
```

---

### 2. **CustomProblemDetailsHandler** - Logging Lỗi HTTP

**Vị trí**: `buildingblocks/problemdetails/CustomProblemDetailsHandler.java`

Log tất cả exceptions được xử lý bởi global exception handler:

```java
@RestControllerAdvice
public class CustomProblemDetailsHandler extends ResponseEntityExceptionHandler {

    private final Logger logger = LoggerFactory.getLogger(CustomProblemDetailsHandler.class);

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleAllExceptions(Exception ex, WebRequest request) {
        // ... xử lý exception ...
        
        // Log structured error details
        logger.atError()
            .addKeyValue("details", JsonConverterUtils.serializeObject(details))
            .log("An error occurred while processing the request.");
        
        return ResponseEntity.status(problemDetail.getStatus()).body(problemDetail);
    }
}
```

**Logs được ghi**:
- ✅ **ERROR**: Structured log với key-value pairs
- ✅ Chi tiết exception (status, title, detail)
- ✅ Timestamp và exception type

**Ví dụ log output**:
```
ERROR - An error occurred while processing the request. details={"detail":"Flight not found","title":"FlightNotFoundException","status":404}
```

---

### 3. **PersistMessageProcessorImpl** - Logging Outbox Pattern

**Vị trí**: `buildingblocks/outboxprocessor/PersistMessageProcessorImpl.java`

Log các message được xử lý từ outbox pattern:

```java
public class PersistMessageProcessorImpl implements PersistMessageProcessor {
    
    private final Logger logger;

    // Log khi xử lý message từ outbox
    logger.info("Message with id: {} and delivery type: {} processed from the persistence message store.",
        message.getId(), message.getDeliveryType().toString());

    // Log khi xử lý internal command
    logger.info("InternalCommand with id: {} and delivery type: {} processed from the persistence message store.",
        message.getId(), message.getDeliveryType().toString());

    // Log khi lưu message vào outbox
    logger.info("Message with id: {} and delivery type: {} saved in persistence message store.",
        persistMessageEntity.getId(), deliveryType.toString());
}
```

**Logs được ghi**:
- ✅ **INFO**: Message được xử lý từ outbox
- ✅ **INFO**: Internal command được xử lý
- ✅ **INFO**: Message được lưu vào outbox

---

### 4. **PersistMessageBackgroundJob** - Logging Background Job

**Vị trí**: `buildingblocks/outboxprocessor/PersistMessageBackgroundJob.java`

Log lỗi trong background job xử lý outbox messages:

```java
public class PersistMessageBackgroundJob {
    
    private final Logger logger;

    @Scheduled(fixedDelay = 5000)
    public void processPersistMessages() {
        try {
            persistMessageProcessor.process();
        } catch (Exception ex) {
            logger.error("Error in persistent message processing", ex);
        }
    }
}
```

**Logs được ghi**:
- ✅ **ERROR**: Lỗi khi xử lý outbox messages

---

### 5. **TransactionPipelineBehavior** - Logging Transactions

**Vị trí**: `buildingblocks/mediator/behaviors/TransactionPipelineBehavior.java`

Log kết quả transaction:

```java
public class TransactionPipelineBehavior<TRequest extends IRequest<TResponse>, TResponse>
        implements IPipelineBehavior<TRequest, TResponse> {
    
    private final Logger logger;

    // Log khi commit thành công
    logger.info("Transaction successfully committed.");

    // Log khi rollback
    logger.error("Transaction is rolled back.", ex);
}
```

**Logs được ghi**:
- ✅ **INFO**: Transaction commit thành công
- ✅ **ERROR**: Transaction rollback

---

### 6. **FlightUpdatedListener** - Logging Event Listeners

**Vị trí**: `services/passenger/listeners/FlightUpdatedListener.java`

Log khi nhận được events từ RabbitMQ:

```java
@Component
public class FlightUpdatedListener implements MessageHandler<FlightUpdated> {

    private final Logger logger;

    @Override
    public void onMessage(FlightUpdated flightUpdated) {
        logger.info("Do other processing after update flight in passenger service for this flight: {}", 
            new KeyValuePair("flight_updated_event", JsonConverterUtils.serializeObject(flightUpdated)));
    }
}
```

**Logs được ghi**:
- ✅ **INFO**: Structured log với event data

---

### 7. **Data Seeders** - Logging Data Initialization

**Vị trí**: 
- `services/flight/data/jpa/seeds/FlightDataSeeder.java`
- `services/passenger/data/jpa/seeds/PassengerDataSeeder.java`

Log quá trình seed data:

```java
public class FlightDataSeeder {
    
    private final Logger logger;

    public void seed() {
        try {
            logger.info("Data seeder is started.");
            // ... seed data ...
            logger.info("Data seeder is finished.");
        } catch (Exception ex) {
            logger.error(ex.getMessage(), ex);
        }
    }
}
```

**Logs được ghi**:
- ✅ **INFO**: Bắt đầu seed data
- ✅ **INFO**: Hoàn thành seed data
- ✅ **ERROR**: Lỗi khi seed data

---

### 8. **RabbitMQ Configuration** - Logging Message Handling

**Vị trí**: `buildingblocks/rabbitmq/RabbitmqConfiguration.java`

Log khi không tìm thấy handler cho message:

```java
@Configuration
public class RabbitmqConfiguration {
    
    private final Logger logger;

    // Log warning khi không có handler
    logger.warn("No handler found for message type: {}", messageType.getTypeName());
}
```

**Logs được ghi**:
- ✅ **WARN**: Không tìm thấy handler cho message type

---

### 9. **Flyway Configuration** - Logging Database Migrations

**Vị trí**: `buildingblocks/flyway/FlywayConfiguration.java`

Log quá trình migration database:

```java
@Configuration
public class FlywayConfiguration {
    
    public FlywayMigrationStrategy flywayMigrationStrategy(Logger logger) {
        return flyway -> {
            if (!flywayProperties.isEnabled()) {
                logger.info("Flyway migrations are disabled.");
                return;
            }
            
            logger.info("Starting Flyway migration...");
            flyway.migrate();
            logger.info("Flyway migration completed successfully!");
        };
    }
}
```

**Logs được ghi**:
- ✅ **INFO**: Flyway migrations disabled
- ✅ **INFO**: Bắt đầu migration
- ✅ **INFO**: Migration hoàn thành

---

### 10. **CustomAuthenticationEntryPoint** - Logging Authentication Errors

**Vị trí**: `buildingblocks/keycloak/CustomAuthenticationEntryPoint.java`

Log lỗi authentication:

```java
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    
    private final Logger logger;

    @Override
    public void commence(HttpServletRequest request, ...) {
        // ... tạo problem detail ...
        
        // Log structured error
        logger.atError()
            .addKeyValue("details", JsonConverterUtils.serializeObject(problemDetail))
            .log("An error occurred while processing the request.");
    }
}
```

**Logs được ghi**:
- ✅ **ERROR**: Structured log với authentication error details

---

## Cấu Hình Logging

### 1. Application Configuration (`application.yml`)

Mỗi service có cấu hình logging riêng:

```yaml
# Flight Service
logging:
  name: flight-logger
  level:
    root: INFO
    io.grpc: DEBUG  # Enable gRPC debug logging (optional)

# Passenger Service
logging:
  name: passenger-logger
  level:
    root: INFO

# Booking Service
logging:
  name: booking-logger
  level:
    root: INFO
    io.grpc: DEBUG
```

### 2. LoggerConfiguration Bean

**Vị trí**: `buildingblocks/logger/LoggerConfiguration.java`

Tạo default logger bean:

```java
@Configuration
public class LoggerConfiguration {

    @Value("${logging.name:default-logger}")
    private String defaultLoggerName;

    @Bean
    public Logger defaultLogger() {
        return LoggerFactory.getLogger(defaultLoggerName);
    }
}
```

**Cách sử dụng**:
```java
@Service
public class MyService {
    
    private final Logger logger;
    
    public MyService(Logger logger) {
        this.logger = logger;  // Injected từ LoggerConfiguration
    }
}
```

### 3. Log Levels

| Level | Mô Tả | Khi Nào Sử Dụng |
|-------|-------|----------------|
| **TRACE** | Chi tiết nhất | Debugging rất chi tiết |
| **DEBUG** | Thông tin debug | Development, troubleshooting |
| **INFO** | Thông tin chung | Business events, request handling |
| **WARN** | Cảnh báo | Vấn đề không nghiêm trọng |
| **ERROR** | Lỗi | Exceptions, failures |

---

## Logging Pipeline Behavior

### Cách Hoạt Động

1. **Mọi request** đi qua Mediator đều được log tự động
2. **LogPipelineBehavior** được đăng ký trong pipeline
3. Logs được ghi ở các điểm:
   - Trước khi xử lý request
   - Sau khi xử lý request
   - Khi có exception

### Cấu Hình Pipeline

**Vị trí**: `buildingblocks/mediator/MediatorConfiguration.java`

```java
@Configuration
public class MediatorConfiguration {
    
    @Bean
    <TRequest extends IRequest<TResponse>, TResponse>
    LogPipelineBehavior<TRequest, TResponse> logPipelineBehavior() {
        return new LogPipelineBehavior<>();
    }
}
```

### Enable/Disable Logging Pipeline

**Vị trí**: `buildingblocks/mediator/MediatorProperties.java`

```java
@ConfigurationProperties(prefix = "mediator")
public class MediatorProperties {
    private boolean enabledLogPipeline = true;
    
    // Có thể disable qua application.yml
    // mediator.enabled-log-pipeline=false
}
```

---

## Structured Logging

Dự án sử dụng **structured logging** với key-value pairs để dễ query và filter.

### Ví Dụ Structured Logging

```java
// Sử dụng KeyValuePair
logger.info("Processing flight update: {}", 
    new KeyValuePair("flight_updated_event", JsonConverterUtils.serializeObject(flightUpdated)));

// Sử dụng addKeyValue (SLF4J 2.x)
logger.atError()
    .addKeyValue("details", JsonConverterUtils.serializeObject(details))
    .addKeyValue("requestId", requestId)
    .log("An error occurred while processing the request.");
```

### Lợi Ích Structured Logging

- ✅ **Dễ query**: Filter theo key-value
- ✅ **Dễ parse**: JSON format
- ✅ **Dễ aggregate**: Group by fields
- ✅ **Better observability**: Tích hợp với Grafana/Kibana

---

## OpenTelemetry Logging

### Cấu Hình OpenTelemetry

**Vị trí**: `buildingblocks/otel/collector/OtelCollectorConfiguration.java`

```java
@Configuration
public class OtelCollectorConfiguration {
    
    @Bean
    public OpenTelemetry openTelemetry(Resource resource, OtelCollectorOptions options) {
        // Configure log exporter
        OtlpGrpcLogRecordExporter logExporter = OtlpGrpcLogRecordExporter.builder()
                .setEndpoint(options.getEndpoint())  // http://localhost:4317
                .build();
        
        // Set up logger provider
        SdkLoggerProvider sdkLoggerProvider = SdkLoggerProvider.builder()
                .setResource(resource)
                .addLogRecordProcessor(
                    BatchLogRecordProcessor.builder(logExporter).build()
                )
                .build();
        
        return OpenTelemetrySdk.builder()
                .setLoggerProvider(sdkLoggerProvider)
                .buildAndRegisterGlobal();
    }
}
```

### Application Configuration

```yaml
spring:
  otel:
    collector:
      service-name: booking
      service-version: 1.0.0
      endpoint: http://localhost:4317  # OpenTelemetry Collector endpoint
```

---

## Log Aggregation và Storage

### 1. OpenTelemetry Collector

**Vị trí**: `deployments/configs/otel-collector-config.yaml`

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:  # Batch logs for better performance

exporters:
  otlphttp/loki:
    endpoint: "http://loki:3100/otlp"
    tls:
      insecure: true
      
  elasticsearch:
    endpoint: "http://elasticsearch:9200"

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/loki, elasticsearch]
```

### 2. Loki Configuration

**Vị trí**: `deployments/configs/loki-config.yaml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /tmp/loki

schema_config:
  configs:
    - from: 2020-05-15
      store: tsdb
      object_store: filesystem
      schema: v13

storage_config:
  filesystem:
    directory: /tmp/loki/chunks

limits_config:
  allow_structured_metadata: true  # Required for OTLP
```

### 3. Elasticsearch + Kibana

Logs cũng được gửi đến Elasticsearch để:
- ✅ Full-text search
- ✅ Advanced querying
- ✅ Visualization trong Kibana

---

## Best Practices

### 1. Sử Dụng Đúng Log Level

```java
// ✅ ĐÚNG
logger.debug("Detailed request data: {}", request);  // Chỉ khi cần debug
logger.info("User {} created booking {}", userId, bookingId);  // Business events
logger.warn("Rate limit approaching: {} requests", count);  // Warnings
logger.error("Failed to process payment", ex);  // Errors với exception

// ❌ SAI
logger.info("Debug data: {}", hugeObject);  // Không nên log debug ở INFO
logger.error("User logged in");  // Không phải error
```

### 2. Structured Logging

```java
// ✅ ĐÚNG - Structured logging
logger.info("Booking created: {}", 
    new KeyValuePair("bookingId", bookingId),
    new KeyValuePair("userId", userId),
    new KeyValuePair("amount", amount));

// ❌ SAI - Unstructured
logger.info("Booking created: bookingId=" + bookingId + ", userId=" + userId);
```

### 3. Không Log Sensitive Data

```java
// ❌ SAI - Log sensitive data
logger.info("User login: username={}, password={}", username, password);

// ✅ ĐÚNG - Không log sensitive data
logger.info("User login attempt: username={}", username);
```

### 4. Log Exception Đúng Cách

```java
// ✅ ĐÚNG - Log exception với context
logger.error("Failed to process booking: {}", bookingId, ex);

// ❌ SAI - Chỉ log message
logger.error("Failed to process booking: " + ex.getMessage());
```

### 5. Performance Logging

```java
// ✅ ĐÚNG - Log execution time
long startTime = System.currentTimeMillis();
// ... process ...
long duration = System.currentTimeMillis() - startTime;
logger.info("Processed in {} ms", duration);
```

### 6. Context Information

```java
// ✅ ĐÚNG - Include context
logger.info("Processing request: {}", 
    new KeyValuePair("requestId", requestId),
    new KeyValuePair("userId", userId),
    new KeyValuePair("endpoint", request.getRequestURI()));
```

---

## Tóm Tắt

### Checklist Logging

- [x] **LogPipelineBehavior**: Tự động log mọi request
- [x] **CustomProblemDetailsHandler**: Log exceptions
- [x] **Outbox Pattern**: Log message processing
- [x] **Event Listeners**: Log event handling
- [x] **Data Seeders**: Log data initialization
- [x] **OpenTelemetry**: Collect và export logs
- [x] **Loki**: Log aggregation
- [x] **Elasticsearch**: Log storage và search

### Log Flow Summary

```
Application Code
    ↓
SLF4J Logger
    ↓
OpenTelemetry SDK
    ↓
OTLP Exporter (gRPC/HTTP)
    ↓
OpenTelemetry Collector
    ↓
    ├─→ Loki (Log Aggregation)
    └─→ Elasticsearch (Storage)
         ↓
    Grafana/Kibana (Visualization)
```

---

## Tài Liệu Tham Khảo

- [SLF4J Documentation](http://www.slf4j.org/manual.html)
- [OpenTelemetry Logging](https://opentelemetry.io/docs/specs/otel/logs/)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)

---

**Lưu ý**: Tất cả logs trong dự án đều được tự động thu thập và gửi đến OpenTelemetry Collector. Không cần cấu hình thêm để enable logging pipeline - nó đã được enable mặc định.
