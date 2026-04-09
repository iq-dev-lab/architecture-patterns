# DDD와 Hexagonal 통합 — Aggregate, Repository Port, Domain Event

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Aggregate가 Application Core(도메인)에 위치해야 하는 이유는 무엇인가?
- Repository Port가 Aggregate 경계를 지키는 구체적인 방법은?
- Domain Event가 Driven Port(EventPublisher)를 통해 발행되는 구조는 어떻게 작동하는가?
- Bounded Context의 경계와 Hexagonal Architecture의 Port 경계는 어떻게 관계를 맺는가?
- DDD의 Aggregate Root 규칙이 Hexagonal의 Port 설계에 어떤 영향을 주는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Hexagonal Architecture와 DDD는 자연스러운 파트너다. DDD가 도메인을 어떻게 구조화할지 알려준다면, Hexagonal은 그 도메인을 인프라로부터 어떻게 보호할지 알려준다.

이 둘을 합치면 강력한 시너지가 생긴다 — Aggregate가 Application Core에 캡슐화되고, Repository Port가 Aggregate 경계를 지키고, Domain Event가 느슨하게 결합된 방식으로 컨텍스트 간 통신을 담당한다.

---

## 😱 흔한 실수 (Before — DDD와 Hexagonal을 결합하지 못할 때)

```
흔한 실수 1: Repository가 개별 Entity에 접근
  // ❌ Aggregate 내부 Entity에 직접 Repository
  public interface OrderLineRepository {
      void save(OrderLine orderLine); // OrderLine은 Aggregate 내부
  }

  // OrderService에서:
  orderRepository.save(order);
  orderLineRepository.save(newLine); // Aggregate 내부를 외부에서 조작

  문제: Aggregate 일관성 보장 불가
       OrderLine 저장 성공 + Order 저장 실패 = 불일치 상태

흔한 실수 2: Domain Event를 Service에서 직접 발행
  @Service
  public class PlaceOrderService {
      @Autowired KafkaTemplate kafka; // ❌ 인프라 직접 의존

      public OrderId placeOrder(PlaceOrderCommand cmd) {
          Order order = Order.create(...);
          orderRepository.save(order);
          kafka.send("order.placed", serialize(order)); // ❌ Service가 Kafka 알아야 함
      }
  }

흔한 실수 3: Bounded Context 경계 무시
  // Order Bounded Context의 Service가
  // Payment Bounded Context의 도메인 객체를 직접 사용
  @Service
  public class PlaceOrderService {
      @Autowired PaymentService paymentService; // ❌ 다른 BC 직접 의존
      public OrderId placeOrder(PlaceOrderCommand cmd) {
          PaymentTransaction tx = paymentService.charge(...);
          // Order BC가 Payment BC의 내부 객체를 앎
      }
  }
```

---

## ✨ 올바른 접근 (After — DDD + Hexagonal 통합)

```
올바른 DDD + Hexagonal 통합:

Aggregate:
  Application Core(domain)에 위치
  Aggregate Root(Order)를 통해서만 내부 접근
  불변식(Invariant) = 도메인 비즈니스 규칙

Repository Port:
  Aggregate Root 단위로 정의
  OrderRepository: Order 전체를 저장/조회
  OrderLineRepository: 없음 (OrderLine은 Order를 통해)

Domain Event:
  Aggregate가 이벤트를 생성 (Order.place() → OrderPlacedEvent)
  Application Service가 EventPublisher Port를 통해 발행
  OrderEventPublisher(Driven Port) → KafkaAdapter(구현체)

Bounded Context:
  Order BC: OrderController → PlaceOrderUseCase → Order → OrderRepository
  Payment BC: PaymentController → ChargeUseCase → Payment → PaymentRepository
  BC 간 통신: PaymentPort(인터페이스) → PaymentBC Adapter 또는 Domain Event
```

---

## 🔬 내부 원리

### 1. Aggregate와 Aggregate Root — Application Core의 핵심

```
DDD Aggregate 정의:
  데이터 변경의 단위로 취급되는 연관 객체의 클러스터
  Aggregate Root: 클러스터의 진입점, 외부에서 접근 가능한 유일한 객체

Order Aggregate 구조:
  ┌─────────────────────────────────┐
  │        Order (Aggregate Root)   │
  │         - id: OrderId           │
  │         - userId: UserId        │
  │         - status: OrderStatus   │
  │         - lines: List<OrderLine>│
  │         - events: List<Event>   │
  │                                 │
  │   OrderLine (내부 Entity)        │
  │    - itemId: ItemId             │
  │    - quantity: int              │
  │    - unitPrice: Money           │
  └─────────────────────────────────┘

규칙:
  OrderLine은 Order를 통해서만 접근 가능
  OrderLine을 직접 수정하는 외부 코드 없음
  Order.addLine(), Order.removeLine() 등 Aggregate Root 메서드를 통해서만 변경

Hexagonal에서 Aggregate 위치:
  domain/model/Order.java       ← Aggregate Root (Application Core 안)
  domain/model/OrderLine.java   ← 내부 Entity (외부에서 직접 접근 불가)
```

### 2. Repository Port가 Aggregate 경계를 지키는 방법

```java
// Aggregate Root 단위 Repository Port
// domain/port/out/OrderRepository.java

public interface OrderRepository {
    // Aggregate Root 단위로 저장/조회
    void save(Order order);                    // 전체 Aggregate 저장
    Optional<Order> findById(OrderId id);      // 전체 Aggregate 로드

    // 내부 Entity(OrderLine)에 대한 직접 메서드 없음
    // ❌ void saveOrderLine(OrderLine line) — 이런 메서드 없어야 함
}

// Aggregate 일관성을 지키는 방식:
@Service
@Transactional
public class AddItemToOrderService {

    private final OrderRepository orderRepository;

    public void addItem(AddItemCommand command) {
        // 1. Aggregate 전체 로드
        Order order = orderRepository.findById(command.getOrderId())
            .orElseThrow(OrderNotFoundException::new);

        // 2. Aggregate Root를 통해 내부 상태 변경
        order.addLine(OrderLine.of(command.getItemId(), command.getQuantity()));

        // 3. Aggregate 전체 저장 (내부 일관성 보장)
        orderRepository.save(order);
        // OrderLine 저장 → Order와 같은 트랜잭션, 일관성 보장
    }
}

// ❌ 잘못된 패턴 (Aggregate 경계 무시):
public class AddItemToOrderService {
    private final OrderLineRepository orderLineRepository; // ← Aggregate 내부에 직접 접근

    public void addItem(AddItemCommand command) {
        OrderLine line = new OrderLine(command.getOrderId(), command.getItemId(), command.getQuantity());
        orderLineRepository.save(line); // ← Order 상태 변경 없이 OrderLine만 저장
        // 문제: Order.status가 CANCELLED인데 OrderLine을 추가해도 막을 방법 없음
    }
}
```

### 3. Domain Event — Aggregate에서 이벤트 생성, Port를 통해 발행

```java
// Domain Event 정의 (domain/event/)
public record OrderPlacedEvent(
    OrderId orderId,
    UserId userId,
    Money totalAmount,
    LocalDateTime occurredAt
) {
    public static OrderPlacedEvent from(Order order) {
        return new OrderPlacedEvent(
            order.getId(),
            order.getUserId(),
            order.calculateTotal(),
            LocalDateTime.now()
        );
    }
}

// Aggregate가 이벤트를 내부에 수집
public class Order {
    private final List<Object> domainEvents = new ArrayList<>();

    public void place() {
        // 비즈니스 규칙 실행
        validateMinAmount();
        this.status = OrderStatus.PLACED;

        // 이벤트 수집 (Kafka 모름, 이벤트만 생성)
        this.domainEvents.add(OrderPlacedEvent.from(this));
    }

    public List<Object> pullDomainEvents() {
        List<Object> events = List.copyOf(this.domainEvents);
        this.domainEvents.clear();
        return events;
    }
}

// Driven Port (domain/port/out/)
public interface OrderEventPublisher {
    void publish(OrderPlacedEvent event);
    void publish(OrderCancelledEvent event);
}

// Application Service가 이벤트를 발행
@Service
@Transactional
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderRepository orderRepository;
    private final OrderEventPublisher eventPublisher; // Kafka 모름, Port만 앎

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.getUserId(), command.getLines());
        order.place(); // 이벤트가 Order 내부에 수집됨

        orderRepository.save(order);

        // 이벤트 발행 (Port를 통해)
        order.pullDomainEvents().forEach(event -> {
            if (event instanceof OrderPlacedEvent e) eventPublisher.publish(e);
        });

        return order.getId();
    }
}

// Driven Adapter: Kafka로 실제 발행
@Component
public class KafkaOrderEventPublisher implements OrderEventPublisher {
    private final KafkaTemplate<String, String> kafka;
    private final ObjectMapper objectMapper;

    @Override
    public void publish(OrderPlacedEvent event) {
        try {
            String payload = objectMapper.writeValueAsString(
                OrderPlacedMessage.from(event)
            );
            kafka.send("order.placed", event.orderId().value(), payload);
        } catch (JsonProcessingException e) {
            throw new EventPublishException("이벤트 발행 실패", e);
        }
    }
}
```

### 4. Bounded Context 경계와 Hexagonal Port

```
Bounded Context 간 통신 방식:

동기 통신 (Anti-Corruption Layer):
  Order BC → PaymentPort(인터페이스) → PaymentBCAdapter
  
  Order BC가 Payment BC를 직접 모르고 PaymentPort(계약)만 앎
  PaymentBCAdapter가 Payment BC의 API를 호출하고 Order BC 모델로 변환

  public interface PaymentPort {          // Order BC의 Driven Port
      PaymentResult charge(PaymentRequest request); // Order BC의 개념
  }

  @Component
  public class PaymentServiceAdapter implements PaymentPort { // Order BC 어댑터
      private final PaymentServiceClient client; // Payment BC의 HTTP 클라이언트

      @Override
      public PaymentResult charge(PaymentRequest request) {
          // Order BC 모델 → Payment BC API 요청 변환 (Anti-Corruption Layer)
          PaymentApiRequest apiRequest = translate(request);
          PaymentApiResponse apiResponse = client.charge(apiRequest);
          // Payment BC API 응답 → Order BC 모델 변환
          return translate(apiResponse);
      }
  }

비동기 통신 (Domain Event):
  Order BC → OrderPlacedEvent 발행 (Kafka)
  Payment BC → OrderPlacedEvent 구독 → 결제 처리 시작

  @Component
  public class OrderPlacedEventConsumer { // Payment BC 내부
      @KafkaListener(topics = "order.placed")
      public void handle(OrderPlacedMessage message) {
          // Order BC 이벤트 → Payment BC 처리
          ChargeOrderCommand command = ChargeOrderCommand.from(message);
          chargeOrderUseCase.charge(command);
      }
  }

BC 간 통신 선택 기준:
  동기 (PaymentPort): 즉시 응답이 필요할 때 (결제 승인 여부를 즉시 알아야 할 때)
  비동기 (Domain Event): 느슨한 결합, 결과를 나중에 알아도 될 때
```

### 5. DDD + Hexagonal 전체 구조

```
Order Bounded Context의 전체 구조:

Domain Layer:
  Order (Aggregate Root)
    - 비즈니스 규칙: place(), cancel(), addLine()
    - 내부 Entity: OrderLine
    - 이벤트 수집: domainEvents
  Money (Value Object)
  OrderId (Value Object)
  OrderStatus (Enum)

Port Layer (in):
  PlaceOrderUseCase   → 주문 생성 진입점
  CancelOrderUseCase  → 주문 취소 진입점
  FindOrderQuery      → 주문 조회 진입점

Port Layer (out):
  OrderSavePort       → Aggregate 저장 계약
  OrderQueryPort      → Aggregate 조회 계약
  PaymentPort         → 결제 처리 계약 (Payment BC 통신)
  OrderEventPublisher → Domain Event 발행 계약

Application Layer:
  PlaceOrderService   → PlaceOrderUseCase 구현
  CancelOrderService  → CancelOrderUseCase 구현

Adapter Layer (in):
  OrderController     → HTTP Driving Adapter
  OrderEventConsumer  → Kafka Driving Adapter (다른 BC 이벤트 수신)

Adapter Layer (out):
  JpaOrderRepository  → OrderSavePort, OrderQueryPort 구현
  PaymentServiceAdapter → PaymentPort 구현 (Payment BC Anti-Corruption Layer)
  KafkaEventPublisher → OrderEventPublisher 구현
  InMemoryOrderRepository → 테스트용
```

---

## 💻 실전 코드 — DDD + Hexagonal 통합 전체 예시

```java
// === Aggregate Root (Application Core) ===
public class Order {
    private final OrderId id;
    private final UserId userId;
    private final List<OrderLine> lines;
    private OrderStatus status;
    private final List<Object> domainEvents = new ArrayList<>();

    // 정적 팩토리 (불완전 객체 생성 불가)
    public static Order create(UserId userId, List<OrderLine> lines) {
        Objects.requireNonNull(userId);
        if (lines == null || lines.isEmpty())
            throw new IllegalArgumentException("주문 항목 필요");
        return new Order(OrderId.newId(), userId, new ArrayList<>(lines), OrderStatus.DRAFT);
    }

    // DB 복원용 정적 팩토리
    public static Order reconstitute(OrderId id, UserId userId,
        List<OrderLine> lines, OrderStatus status) {
        return new Order(id, userId, lines, status);
    }

    // 비즈니스 메서드 (Aggregate Root 통해서만 상태 변경)
    public void place() {
        if (this.status != OrderStatus.DRAFT)
            throw new IllegalStateException("DRAFT 상태만 주문 가능");
        if (calculateTotal().isLessThan(Money.of(1_000)))
            throw new MinOrderAmountException(Money.of(1_000));

        this.status = OrderStatus.PLACED;
        this.domainEvents.add(OrderPlacedEvent.from(this)); // 이벤트 수집
    }

    public void addLine(OrderLine line) {
        if (this.status != OrderStatus.DRAFT)
            throw new IllegalStateException("DRAFT 상태만 항목 추가 가능");
        this.lines.add(line);
        // Aggregate 내부 일관성 유지 — 직접 OrderLine 저장 없음
    }

    public Money calculateTotal() {
        return lines.stream().map(OrderLine::getSubtotal).reduce(Money.ZERO, Money::add);
    }

    public List<Object> pullDomainEvents() {
        List<Object> events = List.copyOf(domainEvents);
        domainEvents.clear();
        return events;
    }

    // getter (setter 없음)
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public List<OrderLine> getLines() { return Collections.unmodifiableList(lines); }
}

// === Application Service (UseCase 조율) ===
@Service
@Transactional
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderSavePort orderSavePort;
    private final PaymentPort paymentPort;
    private final OrderEventPublisher eventPublisher;

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        // Aggregate 생성 (불완전 상태 불가)
        Order order = Order.create(command.userId(), command.lines());

        // Aggregate의 비즈니스 메서드 호출
        order.place();

        // Port를 통해 결제 (Payment BC와 통신)
        PaymentResult payment = paymentPort.charge(
            PaymentRequest.of(order.getId(), order.calculateTotal())
        );

        // Aggregate Root를 통해 상태 변경
        order.confirmPayment(payment.transactionId());

        // Aggregate 전체 저장 (내부 일관성 보장)
        orderSavePort.save(order);

        // Domain Event 발행 (Port를 통해, Kafka 모름)
        order.pullDomainEvents().forEach(event -> {
            if (event instanceof OrderPlacedEvent e) eventPublisher.publish(e);
        });

        return order.getId();
    }
}
```

---

## 📊 패턴 비교 — DDD 없이 vs DDD + Hexagonal

```
비즈니스 규칙 변경 시나리오: "주문 최소 금액 1,000원 → 5,000원"

=== DDD 없이 (Anemic Model + Service) ===
변경 위치: PlaceOrderService.placeOrder() 내부 조건문
          AdminService.forcePlace() 내부 조건문 (중복 규칙)
          BatchService.scheduledOrder() 내부 조건문 (또 다른 중복)
리스크: 3곳 이상 수정, 하나라도 빠지면 불일치

=== DDD + Hexagonal ===
변경 위치: Order.place() 내부 validateMinAmount() 단 하나
이유: 비즈니스 규칙이 Aggregate에 캡슐화
결과: PlaceOrderService, AdminService, BatchService 모두 변경 없음
     → Order.place()를 호출하는 모든 경로에 자동 반영
```

---

## ⚖️ 트레이드오프

```
DDD + Hexagonal 통합의 비용:
  Aggregate 경계 설계 난이도 높음 (어디까지가 Aggregate인가?)
  Domain Event 인프라 (Outbox Pattern or CDC) 필요
  Bounded Context 경계 결정 (팀 합의 필요)

DDD + Hexagonal 통합의 이익:
  비즈니스 규칙이 한 곳(Aggregate)에 캡슐화 → 중복 없음
  BC 간 느슨한 결합 (Domain Event 또는 Anti-Corruption Layer)
  각 BC를 독립적으로 배포/확장 가능

Aggregate 크기 설계 원칙:
  작게 유지 (Small Aggregate): 트랜잭션 범위 = Aggregate 경계
  한 트랜잭션에서 하나의 Aggregate만 변경 (원칙)
  여러 Aggregate 변경 필요 → Domain Event로 Eventually Consistent
```

---

## 📌 핵심 정리

```
DDD + Hexagonal 통합 핵심:

Aggregate:
  Application Core(domain)에 위치
  Aggregate Root를 통해서만 내부 접근
  비즈니스 규칙 = Aggregate가 캡슐화

Repository Port:
  Aggregate Root 단위로 설계 (OrderLine용 없음)
  Aggregate 전체 저장 → 내부 일관성 보장

Domain Event:
  Aggregate가 이벤트 생성 (Kafka 모름)
  Application Service가 EventPublisher Port로 발행
  Driven Adapter(KafkaPublisher)가 실제 발행

Bounded Context 통신:
  동기: PaymentPort(Anti-Corruption Layer) → 즉시 응답 필요 시
  비동기: Domain Event → 느슨한 결합, Eventually Consistent

DDD + Hexagonal 시너지:
  DDD → 도메인을 어떻게 구조화할지
  Hexagonal → 도메인을 인프라에서 어떻게 보호할지
  합치면: 비즈니스 규칙 캡슐화 + 인프라 독립성 + 테스트 용이성
```

---

## 🤔 생각해볼 문제

**Q1.** Aggregate Root 규칙("Aggregate 내부 Entity에 직접 Repository를 만들지 않는다")을 지키면 성능 문제가 생길 수 있다. `Order` 전체를 로드해야 `OrderLine` 하나를 추가할 수 있다면 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**Aggregate 설계를 재검토하거나 읽기 모델을 분리합니다.**

방법 1: Aggregate 크기 재검토
```java
// Order에 OrderLine이 1,000개라면 Order 전체 로드가 부담
// → OrderLine이 실제로 Order와 함께 변경되어야 하는가?
// → 독립적으로 관리 가능하면 별도 Aggregate 고려
```

방법 2: CQRS로 읽기 분리
```java
// 쓰기: Order Aggregate 전체 로드 (일관성 필요)
Order order = orderRepository.findById(id); // 전체 로드
order.addLine(line);
orderRepository.save(order);

// 읽기: 별도 Query 모델 (최적화된 조회)
List<OrderLineView> lines = orderLineQueryRepository.findByOrderId(id);
// Order 전체 없이 OrderLine만 빠르게 조회
```

방법 3: Lazy Loading (JPA)
```java
// @OneToMany(fetch = FetchType.LAZY)
// addLine() 호출 시에만 lines 로드
// 다른 작업에서는 lines 로드 없이 Order 처리
```

성능과 Aggregate 설계 일관성의 트레이드오프 — 팀의 도메인 이해에 따라 결정합니다.

</details>

---

**Q2.** Domain Event를 Aggregate 내부에서 수집하는 방식(domainEvents 리스트)의 장단점은? 대안적 방법은?

<details>
<summary>해설 보기</summary>

**내부 수집 방식의 특징:**

장점:
- Aggregate가 어떤 이벤트를 발생시키는지 코드로 명시됨
- Application Service에서 일관성 있게 처리 가능
- 테스트에서 `pullDomainEvents()`로 이벤트 발생 검증 용이

단점:
- Aggregate가 이벤트 리스트를 상태로 갖게 됨 (약간의 상태 복잡도)
- `pullDomainEvents()` 호출을 잊으면 이벤트 유실

**대안 1: Spring의 ApplicationEventPublisher**
```java
@Service
public class PlaceOrderService {
    @Autowired ApplicationEventPublisher eventPublisher; // Spring 의존
    public OrderId placeOrder(...) {
        order.place();
        eventPublisher.publishEvent(OrderPlacedEvent.from(order)); // 즉시 발행
    }
}
// 단점: Service가 Spring에 의존 (Pure Domain 아님)
```

**대안 2: 반환값으로 이벤트 전달**
```java
public record OrderPlacedResult(Order order, List<DomainEvent> events) {}
public OrderPlacedResult place() {
    // ...
    return new OrderPlacedResult(this, List.of(OrderPlacedEvent.from(this)));
}
// 장점: 순수 함수형, 단점: 반환 타입이 복잡해짐
```

내부 수집 방식(domainEvents 리스트)이 DDD 커뮤니티에서 가장 널리 쓰이는 패턴입니다.

</details>

---

**Q3.** Order BC와 Payment BC가 있을 때, 주문 생성과 결제를 "하나의 트랜잭션"으로 처리하고 싶다면 어떻게 해야 하는가? 두 BC가 독립적이어야 하는 원칙과 충돌하지 않는가?

<details>
<summary>해설 보기</summary>

**두 BC가 독립적이어야 한다면 "하나의 트랜잭션"은 포기하고 Eventually Consistent 방식을 택합니다.**

```
Saga Pattern (분산 트랜잭션 없이 일관성):

1. Order BC: 주문 생성 → OrderStatus = PENDING_PAYMENT
2. Order BC: OrderPlacedEvent 발행 (Kafka)
3. Payment BC: OrderPlacedEvent 수신 → 결제 처리 시작
4. Payment BC: 결제 성공 → PaymentSucceededEvent 발행
5. Order BC: PaymentSucceededEvent 수신 → OrderStatus = PAYMENT_CONFIRMED

실패 처리:
4'. Payment BC: 결제 실패 → PaymentFailedEvent 발행
5'. Order BC: PaymentFailedEvent 수신 → OrderStatus = CANCELLED
```

"하나의 트랜잭션"이 필요하다면:
- 두 Aggregate가 실제로 같은 BC 안에 있어야 하는 것인지 경계를 재검토
- 또는 동기 PaymentPort로 결제 API 호출 후 결과를 Order에 즉시 반영 (강한 결합 수용)

결론: "두 BC의 독립성 유지"와 "하나의 트랜잭션"은 양립하기 어렵습니다. Saga/Choreography 패턴으로 Eventually Consistent하게 처리하는 것이 DDD+Hexagonal의 권장 방향입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Hexagonal의 실제 비용](./07-hexagonal-real-costs.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Clean Architecture ➡️](../clean-architecture/01-clean-architecture-overview.md)**

</div>
