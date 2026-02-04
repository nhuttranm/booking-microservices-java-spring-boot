# Hướng Dẫn Chi Tiết về gRPC và Luồng Xử Lý

## Mục Lục
1. [Tổng Quan về gRPC](#tổng-quan-về-grpc)
2. [Kiến Trúc gRPC trong Dự Án](#kiến-trúc-grpc-trong-dự-án)
3. [Cài Đặt và Cấu Hình](#cài-đặt-và-cấu-hình)
4. [Tạo gRPC Service (Server)](#tạo-grpc-service-server)
5. [Tạo gRPC Client](#tạo-grpc-client)
6. [Luồng Xử Lý Chi Tiết](#luồng-xử-lý-chi-tiết)
7. [Mapping Dữ Liệu](#mapping-dữ-liệu)
8. [Best Practices](#best-practices)

---

## Tổng Quan về gRPC

### gRPC là gì?
- **gRPC** (gRPC Remote Procedure Calls) là một framework RPC mã nguồn mở của Google
- Sử dụng **Protocol Buffers (protobuf)** làm ngôn ngữ định nghĩa interface
- Hỗ trợ nhiều ngôn ngữ: Java, C++, Python, Go, C#, v.v.
- Hiệu suất cao hơn REST/JSON nhờ binary serialization

### Ưu Điểm của gRPC
- ✅ **Hiệu suất cao**: Binary protocol, nhanh hơn JSON
- ✅ **Type-safe**: Code generation từ .proto files
- ✅ **Streaming**: Hỗ trợ unary, server streaming, client streaming, bidirectional streaming
- ✅ **HTTP/2**: Multiplexing, header compression
- ✅ **Code Generation**: Tự động generate code từ .proto

### Khi Nào Sử Dụng gRPC?
- ✅ Giao tiếp **nội bộ** giữa các microservices (internal communication)
- ✅ Cần hiệu suất cao và độ trễ thấp
- ✅ Cần type-safe contracts
- ❌ Không phù hợp cho public API (khó debug, không có browser support)

---

## Kiến Trúc gRPC trong Dự Án

### Sơ Đồ Kiến Trúc

```
┌─────────────────┐
│  Booking Service │
│   (gRPC Client)  │
└────────┬─────────┘
         │ gRPC Calls
         │
    ┌────┴────┬──────────────┐
    │         │              │
    ▼         ▼              ▼
┌────────┐ ┌──────────┐  ┌──────────┐
│ Flight │ │Passenger │  │  Other   │
│Service │ │ Service  │  │ Services │
│(Server)│ │(Server)  │  │          │
└────────┘ └──────────┘  └──────────┘
```

### Ports Configuration

| Service | HTTP Port | gRPC Port |
|---------|-----------|-----------|
| Flight Service | 8082 | 9092 |
| Passenger Service | 8083 | 9093 |
| Booking Service | 8084 | 9094 |

### Luồng Giao Tiếp

1. **Booking Service** (Client) gọi **Flight Service** và **Passenger Service** (Servers)
2. Sử dụng **Blocking Stub** cho synchronous calls
3. Tất cả services đều có thể expose gRPC server

---

## Cài Đặt và Cấu Hình

### 1. Dependencies trong `pom.xml`

#### Dependencies cho gRPC Core
```xml
<!-- gRPC Core -->
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.57.2</version>
</dependency>

<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.57.2</version>
</dependency>

<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.57.2</version>
</dependency>

<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty</artifactId>
    <version>1.57.2</version>
</dependency>
```

#### Dependencies cho Spring Boot Integration
```xml
<!-- Spring Boot gRPC Integration -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>

<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>

<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-client-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
```

### 2. Maven Plugin cho Protobuf

```xml
<build>
    <extensions>
        <!-- OS Detection Plugin -->
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.0</version>
        </extension>
    </extensions>
    
    <plugins>
        <!-- Protobuf Maven Plugin -->
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>
                    com.google.protobuf:protoc:3.24.2:exe:${os.detected.classifier}
                </protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>
                    io.grpc:protoc-gen-grpc-java:1.57.2:exe:${os.detected.classifier}
                </pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>        <!-- Generate protobuf classes -->
                        <goal>compile-custom</goal> <!-- Generate gRPC service classes -->
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 3. Cấu Hình trong `application.yml`

#### Server Configuration (Flight Service, Passenger Service)
```yaml
grpc:
  server:
    port: 9092  # Cho Flight Service
    # port: 9093  # Cho Passenger Service
```

#### Client Configuration (Booking Service)
```yaml
grpc:
  server:
    port: 9094  # gRPC server port của Booking Service
  client:
    flight-service:
      address: "static://localhost:9092"
    passenger-service:
      address: "static://localhost:9093"
```

---

## Tạo gRPC Service (Server)

### Bước 1: Định Nghĩa Proto File

Tạo file `src/main/proto/flight.proto`:

```protobuf
syntax = "proto3";

package flight;
import "google/protobuf/timestamp.proto";

// Định nghĩa Service
service FlightService {
  rpc GetById (GetByIdRequestDto) returns (FlightResponseDto);
  rpc GetAvailableSeats (GetAvailableSeatsRequestDto) returns (GetAvailableSeatsResponseDto);
  rpc ReserveSeat (ReserveSeatRequestDto) returns (SeatResponseDto);
}

// Request Messages
message GetByIdRequestDto {
  string Id = 1;
}

message GetAvailableSeatsRequestDto {
  string FlightId = 1;
}

message ReserveSeatRequestDto {
  string FlightId = 1;
  string SeatNumber = 2;
}

// Response Messages
message FlightResponseDto {
  string Id = 1;
  string FlightNumber = 2;
  string AircraftId = 3;
  string DepartureAirportId = 4;
  google.protobuf.Timestamp DepartureDate = 5;
  google.protobuf.Timestamp ArriveDate = 6;
  string ArriveAirportId = 7;
  double DurationMinutes = 8;
  google.protobuf.Timestamp FlightDate = 9;
  FlightStatus Status = 10;
  double Price = 11;
}

message GetAvailableSeatsResponseDto {
  repeated SeatResponseDto SeatsDto = 1;
}

message SeatResponseDto {
  string Id = 1;
  string SeatNumber = 2;
  SeatType SeatType = 3;
  SeatClass SeatClass = 4;
  string FlightId = 5;
}

// Enums
enum FlightStatus {
  FLIGHT_STATUS_FLYING = 0;
  FLIGHT_STATUS_DELAY = 1;
  FLIGHT_STATUS_CANCELED = 2;
  FLIGHT_STATUS_COMPLETED = 3;
}

enum SeatType {
  SEAT_TYPE_WINDOW = 0;
  SEAT_TYPE_MIDDLE = 1;
  SEAT_TYPE_AISLE = 2;
}

enum SeatClass {
  SEAT_CLASS_FIRST_CLASS = 0;
  SEAT_CLASS_BUSINESS = 1;
  SEAT_CLASS_ECONOMY = 2;
}
```

### Bước 2: Build Proto Files

Chạy Maven để generate Java classes:

```bash
mvn clean compile
```

Sau khi build, các class được generate tại:
- `target/generated-sources/protobuf/java/` - Protobuf messages
- `target/generated-sources/protobuf/grpc-java/` - gRPC service classes

### Bước 3: Implement gRPC Service

Tạo class `FlightServiceGrpcImpl.java`:

```java
package io.bookingmicroservices.flight.grpcserver;

import buildingblocks.mediator.abstractions.IMediator;
import flight.Flight;
import flight.FlightServiceGrpc;
import io.bookingmicroservices.flight.flights.dtos.FlightDto;
import io.bookingmicroservices.flight.flights.features.getflightbyid.GetFlightByIdQuery;
import io.bookingmicroservices.flight.seats.dtos.SeatDto;
import io.bookingmicroservices.flight.seats.features.getavailableseats.GetAvailableSeatsQuery;
import io.bookingmicroservices.flight.seats.features.reserveseat.ReserveSeatCommand;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import java.util.List;
import java.util.UUID;
import static io.bookingmicroservices.flight.flights.features.Mappings.toFlightResponseDtoGrpc;
import static io.bookingmicroservices.flight.seats.features.Mappings.toSeatResponseDtoGrpc;

@GrpcService  // Annotation này tự động register service với Spring
public class FlightServiceGrpcImpl extends FlightServiceGrpc.FlightServiceImplBase {
  
  private final IMediator mediator;

  public FlightServiceGrpcImpl(IMediator mediator) {
    this.mediator = mediator;
  }

  @Override
  public void getById(
      Flight.GetByIdRequestDto request, 
      StreamObserver<Flight.FlightResponseDto> responseObserver) {
    
    // 1. Convert gRPC request sang domain query
    GetFlightByIdQuery query = new GetFlightByIdQuery(UUID.fromString(request.getId()));
    
    // 2. Gọi business logic qua Mediator
    FlightDto result = mediator.send(query);
    
    // 3. Convert domain DTO sang gRPC response
    Flight.FlightResponseDto flightResponseDtoGrpc = toFlightResponseDtoGrpc(result);
    
    // 4. Gửi response về client
    responseObserver.onNext(flightResponseDtoGrpc);
    responseObserver.onCompleted();
  }

  @Override
  public void getAvailableSeats(
      Flight.GetAvailableSeatsRequestDto request, 
      StreamObserver<Flight.GetAvailableSeatsResponseDto> responseObserver) {
    
    List<SeatDto> result = mediator.send(
        new GetAvailableSeatsQuery(UUID.fromString(request.getFlightId()))
    );
    
    // Convert list sang gRPC response
    List<Flight.SeatResponseDto> seatsResponseDtoGrpc = 
        result.stream()
            .map(Mappings::toSeatResponseDtoGrpc)
            .toList();
    
    Flight.GetAvailableSeatsResponseDto response = 
        Flight.GetAvailableSeatsResponseDto.newBuilder()
            .addAllSeatsDto(seatsResponseDtoGrpc)
            .build();
    
    responseObserver.onNext(response);
    responseObserver.onCompleted();
  }

  @Override
  public void reserveSeat(
      Flight.ReserveSeatRequestDto request, 
      StreamObserver<Flight.SeatResponseDto> responseObserver) {
    
    ReserveSeatCommand command = new ReserveSeatCommand(
        request.getSeatNumber(), 
        UUID.fromString(request.getFlightId())
    );
    
    SeatDto result = mediator.send(command);
    Flight.SeatResponseDto seatResponseDtoGrpc = toSeatResponseDtoGrpc(result);
    
    responseObserver.onNext(seatResponseDtoGrpc);
    responseObserver.onCompleted();
  }
}
```

### Giải Thích:

1. **@GrpcService**: Annotation tự động đăng ký service với Spring Boot gRPC server
2. **Extends FlightServiceGrpc.FlightServiceImplBase**: Base class được generate từ .proto
3. **StreamObserver**: Callback interface để gửi response về client
4. **onNext()**: Gửi response data
5. **onCompleted()**: Báo hiệu hoàn thành request

---

## Tạo gRPC Client

### Bước 1: Copy Proto Files

Để Booking Service có thể gọi Flight Service, cần copy proto files:

```
booking-service/
  src/main/proto/
    flight.proto      (copy từ flight-service)
    passenger.proto   (copy từ passenger-service)
```

### Bước 2: Build Proto Files

Chạy `mvn clean compile` để generate client stubs.

### Bước 3: Cấu Hình gRPC Client

Tạo `GrpcClientsConfiguration.java`:

```java
package io.bookingmicroservices.booking.grpcclient;

import flight.FlightServiceGrpc;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import passenger.PassengerServiceGrpc;
import net.devh.boot.grpc.client.config.GrpcChannelsProperties;

@Configuration
public class GrpcClientsConfiguration {

    private final GrpcChannelsProperties grpcChannelsProperties;

    public GrpcClientsConfiguration(GrpcChannelsProperties grpcChannelsProperties) {
        this.grpcChannelsProperties = grpcChannelsProperties;
    }

    @Bean
    public FlightServiceGrpc.FlightServiceBlockingStub flightServiceBlockingStub() {
        // Lấy cấu hình từ application.yml
        var channelProperties = this.grpcChannelsProperties.getChannel("flight-service");

        // Tạo ManagedChannel
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress(
                channelProperties.getAddress().getHost(), 
                channelProperties.getAddress().getPort()
            )
            .usePlaintext()  // Không dùng TLS (chỉ cho dev)
            .build();

        // Tạo Blocking Stub (synchronous calls)
        return FlightServiceGrpc.newBlockingStub(channel);
    }

    @Bean
    public PassengerServiceGrpc.PassengerServiceBlockingStub passengerServiceBlockingStub() {
        var channelProperties = this.grpcChannelsProperties.getChannel("passenger-service");

        ManagedChannel channel = ManagedChannelBuilder
            .forAddress(
                channelProperties.getAddress().getHost(), 
                channelProperties.getAddress().getPort()
            )
            .usePlaintext()
            .build();

        return PassengerServiceGrpc.newBlockingStub(channel);
    }
}
```

### Bước 4: Sử Dụng gRPC Client

Trong Command Handler:

```java
@Service
public class CreateBookingCommandHandler 
    implements ICommandHandler<CreateBookingCommand, BookingDto> {

    private final FlightServiceGrpc.FlightServiceBlockingStub flightServiceBlockingStub;
    private final PassengerServiceGrpc.PassengerServiceBlockingStub passengerServiceBlockingStub;
    private final BookingRepository bookingRepository;

    public CreateBookingCommandHandler(
            FlightServiceGrpc.FlightServiceBlockingStub flightServiceBlockingStub,
            PassengerServiceGrpc.PassengerServiceBlockingStub passengerServiceBlockingStub,
            BookingRepository bookingRepository) {
        this.flightServiceBlockingStub = flightServiceBlockingStub;
        this.passengerServiceBlockingStub = passengerServiceBlockingStub;
        this.bookingRepository = bookingRepository;
    }

    @Override
    public BookingDto handle(CreateBookingCommand command) {
        
        // 1. Gọi Flight Service để lấy thông tin chuyến bay
        Flight.GetByIdRequestDto flightRequest = 
            Flight.GetByIdRequestDto.newBuilder()
                .setId(command.flightId().toString())
                .build();
        
        Flight.FlightResponseDto flight = 
            flightServiceBlockingStub.getById(flightRequest);
        
        if (flight == null) {
            throw new FlightNotFoundException();
        }

        // 2. Gọi Passenger Service để lấy thông tin hành khách
        Passenger.PassengerRequestDto passengerRequest = 
            Passenger.PassengerRequestDto.newBuilder()
                .setId(command.passengerId().toString())
                .build();
        
        Passenger.PassengerResponseDto passenger = 
            passengerServiceBlockingStub.getById(passengerRequest);
        
        if (passenger == null) {
            throw new PassengerNotFoundException();
        }

        // 3. Gọi Flight Service để lấy ghế trống
        Flight.GetAvailableSeatsRequestDto seatsRequest = 
            Flight.GetAvailableSeatsRequestDto.newBuilder()
                .setFlightId(command.flightId().toString())
                .build();
        
        Flight.GetAvailableSeatsResponseDto availableSeats = 
            flightServiceBlockingStub.getAvailableSeats(seatsRequest);
        
        Flight.SeatResponseDto emptySeat = availableSeats
            .getSeatsDtoList()
            .stream()
            .findAny()
            .orElse(null);
        
        if (emptySeat == null) {
            throw new SeatNumberIsNotAvailableException();
        }

        // 4. Tạo booking entity
        Booking booking = Booking.create(
            new BookingId(command.id()),
            new PassengerInfo(passenger.getName()),
            new Trip(
                flight.getFlightNumber(),
                UUID.fromString(flight.getAircraftId()),
                UUID.fromString(flight.getDepartureAirportId()),
                UUID.fromString(flight.getArriveAirportId()),
                ProtobufUtils.toLocalDateTime(flight.getFlightDate()),
                BigDecimal.ONE,
                command.description(),
                emptySeat.getSeatNumber()
            )
        );

        // 5. Reserve seat qua gRPC
        Flight.ReserveSeatRequestDto reserveRequest = 
            Flight.ReserveSeatRequestDto.newBuilder()
                .setFlightId(flight.getId())
                .setSeatNumber(emptySeat.getSeatNumber())
                .build();
        
        flightServiceBlockingStub.reserveSeat(reserveRequest);

        // 6. Lưu booking vào database
        BookingEntity bookingEntity = Mappings.toBookingEntity(booking);
        BookingEntity createdBooking = bookingRepository.save(bookingEntity);

        return toBookingDto(createdBooking);
    }
}
```

---

## Luồng Xử Lý Chi Tiết

### Luồng Tạo Booking (End-to-End)

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. HTTP Request                                                  │
│    POST /api/v1/booking                                          │
│    Body: { passengerId, flightId, description }                 │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. CreateBookingController                                       │
│    - Nhận CreateBookingRequestDto                               │
│    - Convert sang CreateBookingCommand                          │
│    - Gọi mediator.send(command)                                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Mediator Pipeline                                             │
│    - ValidationPipelineBehavior                                  │
│    - LoggingPipelineBehavior                                    │
│    - ErrorHandlingPipelineBehavior                              │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. CreateBookingCommandHandler                                   │
│                                                                  │
│    ┌──────────────────────────────────────────────────────┐    │
│    │ 4.1. Gọi Flight Service (gRPC)                       │    │
│    │      flightServiceBlockingStub.getById()            │    │
│    │      └─> FlightServiceGrpcImpl.getById()            │    │
│    │          └─> GetFlightByIdQueryHandler              │    │
│    │              └─> FlightRepository.findById()         │    │
│    └──────────────────────────────────────────────────────┘    │
│                                                                  │
│    ┌──────────────────────────────────────────────────────┐    │
│    │ 4.2. Gọi Passenger Service (gRPC)                   │    │
│    │      passengerServiceBlockingStub.getById()          │    │
│    │      └─> PassengerServiceGrpcImpl.getById()        │    │
│    │          └─> GetPassengerByIdQueryHandler           │    │
│    │              └─> PassengerRepository.findById()     │    │
│    └──────────────────────────────────────────────────────┘    │
│                                                                  │
│    ┌──────────────────────────────────────────────────────┐    │
│    │ 4.3. Gọi Flight Service để lấy ghế trống (gRPC)     │    │
│    │      flightServiceBlockingStub.getAvailableSeats()   │    │
│    │      └─> FlightServiceGrpcImpl.getAvailableSeats()  │    │
│    │          └─> GetAvailableSeatsQueryHandler           │    │
│    │              └─> SeatRepository.findAvailableSeats() │    │
│    └──────────────────────────────────────────────────────┘    │
│                                                                  │
│    ┌──────────────────────────────────────────────────────┐    │
│    │ 4.4. Tạo Booking Domain Object                       │    │
│    │      Booking.create(...)                              │    │
│    └──────────────────────────────────────────────────────┘    │
│                                                                  │
│    ┌──────────────────────────────────────────────────────┐    │
│    │ 4.5. Reserve Seat qua gRPC                           │    │
│    │      flightServiceBlockingStub.reserveSeat()        │    │
│    │      └─> FlightServiceGrpcImpl.reserveSeat()       │    │
│    │          └─> ReserveSeatCommandHandler             │    │
│    │              └─> SeatRepository.update()             │    │
│    └──────────────────────────────────────────────────────┘    │
│                                                                  │
│    ┌──────────────────────────────────────────────────────┐    │
│    │ 4.6. Lưu Booking vào PostgreSQL                      │    │
│    │      bookingRepository.save(bookingEntity)          │    │
│    └──────────────────────────────────────────────────────┘    │
│                                                                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Return BookingDto                                             │
│    - Convert BookingEntity sang BookingDto                      │
│    - Return về Controller                                       │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. HTTP Response                                                 │
│    200 OK                                                        │
│    Body: BookingDto                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Sequence Diagram

```
Client          Booking Service        Flight Service        Passenger Service
  │                    │                      │                      │
  │──POST /booking───>│                      │                      │
  │                    │                      │                      │
  │                    │──getById()──────────>│                      │
  │                    │<──FlightResponse─────│                      │
  │                    │                      │                      │
  │                    │──getById()─────────────────────────────────>│
  │                    │<──PassengerResponse─────────────────────────│
  │                    │                      │                      │
  │                    │──getAvailableSeats()>│                      │
  │                    │<──SeatsResponse─────│                      │
  │                    │                      │                      │
  │                    │──reserveSeat()──────>│                      │
  │                    │<──SeatResponse───────│                      │
  │                    │                      │                      │
  │                    │──save()──────────────│                      │
  │                    │                      │                      │
  │<──200 OK───────────│                      │                      │
```

---

## Mapping Dữ Liệu

### 1. Java Domain Object → gRPC Message

```java
public static flight.Flight.FlightResponseDto toFlightResponseDtoGrpc(FlightDto flightDto) {
    return flight.Flight.FlightResponseDto.newBuilder()
        .setId(flightDto.id().toString())
        .setFlightNumber(flightDto.flightNumber())
        .setAircraftId(flightDto.aircraftId().toString())
        .setDepartureAirportId(flightDto.departureAirportId().toString())
        .setArriveAirportId(flightDto.arriveAirportId().toString())
        .setDepartureDate(ProtobufUtils.toProtobufTimestamp(flightDto.departureDate()))
        .setArriveDate(ProtobufUtils.toProtobufTimestamp(flightDto.arriveDate()))
        .setDurationMinutes(flightDto.durationMinutes().doubleValue())
        .setFlightDate(ProtobufUtils.toProtobufTimestamp(flightDto.flightDate()))
        .setStatus(toFlightStatusGrpc(flightDto.status()))
        .setPrice(flightDto.price().doubleValue())
        .build();
}
```

### 2. gRPC Message → Java Domain Object

```java
// Trong CreateBookingCommandHandler
Flight.FlightResponseDto flight = flightServiceBlockingStub.getById(request);

// Convert protobuf timestamp sang LocalDateTime
LocalDateTime flightDate = ProtobufUtils.toLocalDateTime(flight.getFlightDate());

// Convert string UUID sang UUID
UUID aircraftId = UUID.fromString(flight.getAircraftId());
```

### 3. ProtobufUtils - Helper Class

```java
package buildingblocks.utils.protobuf;

import com.google.protobuf.Timestamp;
import java.time.LocalDateTime;
import java.time.ZoneOffset;

public final class ProtobufUtils {
    
    // Java LocalDateTime → Protobuf Timestamp
    public static Timestamp toProtobufTimestamp(LocalDateTime localDateTime) {
        return Timestamps.fromMillis(
            localDateTime.toInstant(ZoneOffset.UTC).toEpochMilli()
        );
    }
    
    // Protobuf Timestamp → Java LocalDateTime
    public static LocalDateTime toLocalDateTime(Timestamp timestamp) {
        Instant instant = Instant.ofEpochSecond(
            timestamp.getSeconds(), 
            timestamp.getNanos()
        );
        return LocalDateTime.ofInstant(instant, ZoneOffset.UTC);
    }
}
```

### 4. Enum Mapping

```java
public static flight.Flight.FlightStatus toFlightStatusGrpc(FlightStatus flightStatus) {
    return switch (flightStatus) {
        case Flying -> flight.Flight.FlightStatus.FLIGHT_STATUS_FLYING;
        case Delay -> flight.Flight.FlightStatus.FLIGHT_STATUS_DELAY;
        case Completed -> flight.Flight.FlightStatus.FLIGHT_STATUS_COMPLETED;
        case Canceled -> flight.Flight.FlightStatus.FLIGHT_STATUS_CANCELED;
    };
}
```

---

## Best Practices

### 1. Error Handling

```java
@Override
public void getById(
    Flight.GetByIdRequestDto request, 
    StreamObserver<Flight.FlightResponseDto> responseObserver) {
    
    try {
        FlightDto result = mediator.send(
            new GetFlightByIdQuery(UUID.fromString(request.getId()))
        );
        
        if (result == null) {
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription("Flight not found")
                    .asRuntimeException()
            );
            return;
        }
        
        Flight.FlightResponseDto response = toFlightResponseDtoGrpc(result);
        responseObserver.onNext(response);
        responseObserver.onCompleted();
        
    } catch (Exception e) {
        responseObserver.onError(
            Status.INTERNAL
                .withDescription(e.getMessage())
                .withCause(e)
                .asRuntimeException()
        );
    }
}
```

### 2. Request Validation

```java
@Override
public void getById(
    Flight.GetByIdRequestDto request, 
    StreamObserver<Flight.FlightResponseDto> responseObserver) {
    
    // Validate request
    if (request.getId() == null || request.getId().isEmpty()) {
        responseObserver.onError(
            Status.INVALID_ARGUMENT
                .withDescription("Id is required")
                .asRuntimeException()
        );
        return;
    }
    
    // Validate UUID format
    try {
        UUID.fromString(request.getId());
    } catch (IllegalArgumentException e) {
        responseObserver.onError(
            Status.INVALID_ARGUMENT
                .withDescription("Invalid UUID format")
                .asRuntimeException()
        );
        return;
    }
    
    // Process request...
}
```

### 3. Timeout Configuration

```java
@Bean
public FlightServiceGrpc.FlightServiceBlockingStub flightServiceBlockingStub() {
    ManagedChannel channel = ManagedChannelBuilder
        .forAddress("localhost", 9092)
        .usePlaintext()
        .build();
    
    return FlightServiceGrpc.newBlockingStub(channel)
        .withDeadlineAfter(5, TimeUnit.SECONDS);  // 5 seconds timeout
}
```

### 4. Retry Configuration

```java
@Bean
public FlightServiceGrpc.FlightServiceBlockingStub flightServiceBlockingStub() {
    ManagedChannel channel = ManagedChannelBuilder
        .forAddress("localhost", 9092)
        .usePlaintext()
        .build();
    
    return FlightServiceGrpc.newBlockingStub(channel)
        .withRetry(RetryPolicy.builder()
            .maxAttempts(3)
            .initialBackoff(Duration.ofMillis(100))
            .maxBackoff(Duration.ofSeconds(1))
            .backoffMultiplier(2.0)
            .build());
}
```

### 5. Logging

```yaml
logging:
  level:
    io.grpc: DEBUG  # Enable gRPC debug logging
```

### 6. Proto File Best Practices

- ✅ Sử dụng `proto3` syntax
- ✅ Đặt tên message rõ ràng (RequestDto, ResponseDto)
- ✅ Sử dụng `repeated` cho lists
- ✅ Sử dụng `google.protobuf.Timestamp` cho dates
- ✅ Định nghĩa enums rõ ràng
- ✅ Version proto files khi có breaking changes

---

## Tóm Tắt

### Checklist Tạo gRPC Service

- [ ] 1. Tạo `.proto` file định nghĩa service và messages
- [ ] 2. Thêm dependencies vào `pom.xml`
- [ ] 3. Cấu hình `protobuf-maven-plugin`
- [ ] 4. Chạy `mvn clean compile` để generate code
- [ ] 5. Implement service class extends `*ServiceImplBase`
- [ ] 6. Thêm `@GrpcService` annotation
- [ ] 7. Cấu hình port trong `application.yml`
- [ ] 8. Tạo mapping methods (DTO ↔ gRPC)

### Checklist Tạo gRPC Client

- [ ] 1. Copy `.proto` files từ service
- [ ] 2. Build để generate client stubs
- [ ] 3. Cấu hình client channels trong `application.yml`
- [ ] 4. Tạo `@Configuration` class với `@Bean` methods
- [ ] 5. Inject stubs vào handlers/services
- [ ] 6. Sử dụng stubs để gọi remote methods
- [ ] 7. Handle errors và timeouts

---

## Tài Liệu Tham Khảo

- [gRPC Official Documentation](https://grpc.io/docs/)
- [Protocol Buffers Guide](https://developers.google.com/protocol-buffers)
- [grpc-spring-boot-starter](https://github.com/yidongnan/grpc-spring-boot-starter)
- [gRPC Java Tutorial](https://grpc.io/docs/languages/java/)

---

**Lưu ý**: Hướng dẫn này dựa trên dự án booking-microservices-java-spring-boot. Các ví dụ và code snippets đều từ codebase thực tế của dự án.
