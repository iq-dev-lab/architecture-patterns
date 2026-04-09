# 인프라 레이어 분리 — Repository와 Kafka를 Port로 추상화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Repository 인터페이스를 Infrastructure에서 Domain으로 이동시키는 정확한 절차는?
- JPA Repository 구현체를 Driven Adapter로 재구성하는 방법은?
- Kafka 발행을 `OrderEventPublisher` Port로 추상화하는 방법은?
- 분리 완료 후 InMemory Adapter로 단위 테스트를 작성하는 방법은?
- 인프라 분리 전후 단위 테스트 실행 시간은 얼마나 달라지는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

도메인 레이어 분리(Ch7-03)가 "비즈니스 로직을 Domain으로 가져오는 것"이었다면, 인프라 레이어 분리는 "인프라(JPA, Kafka)를 Domain으로부터 밀어내는 것"이다.

이 두 작업이 완료되면 Domain과 Application Service는 JPA, Kafka, Spring을 전혀 모르게 된다. 그 결과 Application Service 단위 테스트에서 InMemory 구현체를 주입하면 Spring Context, DB, Kafka 없이 0.01초 테스트가 가능해진다. 이것이 Hexagonal Architecture의 핵심 이익이다.

---

## 😱 흔한 실수 (Before — 인터페이스 위치가 잘못됐을 때)

```
인터페이스를 Infrastructure에 두는 실수:

// 잘못된 구조
com.example
  ├── service/
  │   └── OrderService.java         ← Application
  └── repository/
      ├── OrderRepository.java      ← ❌ 인터페이스가 infrastructure에
      └── JpaOrderRepository.java   ← 구현체

OrderService → OrderRepository(인터페이스) → infrastructure 패키지
= Application이 Infrastructure를 의존 (의존성 방향 위반!)

올바른 구조:
com.example
  ├── order/
  │   ├── domain/port/out/
  │   │   └── OrderSavePort.java    ← ✅ 인터페이스가 domain에
  │   ├── application/service/
  │   │   └── PlaceOrderService.java
  │   └── adapter/out/persistence/
  │       └── JpaOrderRepository.java ← 구현체 (domain Port 구현)

PlaceOrderService(application) → OrderSavePort(domain)
JpaOrderRepository(adapter) → OrderSavePort(domain) 구현
= 모든 의존성이 domain 방향 ✓
```

---

## ✨ 올바른 접근 (After — 인터페이스를 domain으로, 구현체를 adapter로)

```
인프라 분리 후 전체 구조:

Domain Port (인터페이스):
  order/domain/port/out/OrderSavePort.java
  order/domain/port/out/OrderQueryPort.java
  order/domain/port/out/OrderEventPublisher.java
  order/domain/port/out/PaymentPort.java

Driven Adapter (구현체):
  order/adapter/out/persistence/JpaOrderRepository.java
    → implements OrderSavePort, OrderQueryPort
  order/adapter/out/messaging/KafkaOrderEventPublisher.java
    → implements OrderEventPublisher
  order/adapter/out/payment/KakaoPayAdapter.java
    → implements PaymentPort

테스트용 InMemory Adapter:
  test/support/InMemoryOrderRepository.java
    → implements OrderSavePort, OrderQueryPort
  test/support/InMemoryOrderEventPublisher.java
    → implements OrderEventPublisher

결과:
  PlaceOrderService는 Port 인터페이스만 앎 (JPA, Kafka 모름)
  PlaceOrderService 단위 테스트: InMemory 주입 → Spring 없이 실행
```

---

## 🔬 내부 원리 — 단계별 인프라 분리

### 1. Repository 인터페이스를 Domain으로 이동

```java
// Phase 1: Repository 인터페이스 정의 (domain/port/out/)

// order/domain/port/out/OrderSavePort.java
public interface OrderSavePort {
    void save(Order order);
}

// order/domain/port/out/OrderQueryPort.java
public interface OrderQueryPort {
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
    boolean existsById(OrderId id);
}

// 인터페이스 분리 원칙(ISP) 적용:
// PlaceOrderService는 save()만 필요 → OrderSavePort만 의존
// FindOrderService는 find*()만 필요 → OrderQueryPort만 의존
// 두 Port를 통합하면 모든 Service가 불필요한 메서드도 알게 됨

// 의존성 방향 확인:
// PlaceOrderService.java:
//   import com.example.order.domain.port.out.OrderSavePort; ← domain 내부
// JpaOrderRepository.java:
//   import com.example.order.domain.port.out.OrderSavePort; ← adapter가 domain 참조
//   → adapter → domain (올바른 방향)
```

### 2. JPA Repository를 Driven Adapter로 재구성

```java
// Before: Spring Data JPA 인터페이스를 Service가 직접 사용
public interface OrderJpaRepository extends JpaRepository<Order, Long> {
    Optional<Order> findByOrderId(String orderId);
    List<Order> findByUserId(String userId);
}
// OrderService가 이 인터페이스를 직접 @Autowired

// After: JPA 구현체가 Domain Port를 구현

// order/adapter/out/persistence/entity/OrderJpaEntity.java
@Entity
@Table(name = "orders")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class OrderJpaEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_id", unique = true, nullable = false)
    private String orderId;

    @Column(name = "user_id", nullable = false)
    private String userId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private String status;

    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount;

    @Column(name = "payment_tx_id")
    private String paymentTxId;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
}

// order/adapter/out/persistence/SpringDataOrderJpa.java (Spring Data 내부용)
interface SpringDataOrderJpa extends JpaRepository<OrderJpaEntity, Long> {
    Optional<OrderJpaEntity> findByOrderId(String orderId);
    List<OrderJpaEntity> findByUserId(String userId);
}

// order/adapter/out/persistence/JpaOrderRepository.java
// Domain Port 구현체 (핵심!)
@Repository
@RequiredArgsConstructor
public class JpaOrderRepository implements OrderSavePort, OrderQueryPort {

    private final SpringDataOrderJpa jpa;
    private final OrderPersistenceMapper mapper;

    // OrderSavePort 구현
    @Override
    public void save(Order order) {
        // Domain Entity → JPA Entity 변환 후 저장
        OrderJpaEntity entity = mapper.toJpaEntity(order);
        jpa.save(entity);
    }

    // OrderQueryPort 구현
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findByOrderId(id.value())
            .map(mapper::toDomain);  // JPA Entity → Domain Entity 변환
    }

    @Override
    public List<Order> findByUserId(UserId userId) {
        return jpa.findByUserId(userId.value()).stream()
            .map(mapper::toDomain)
            .toList();
    }

    @Override
    public boolean existsById(OrderId id) {
        return jpa.findByOrderId(id.value()).isPresent();
    }
}

// order/adapter/out/persistence/mapper/OrderPersistenceMapper.java
@Component
public class OrderPersistenceMapper {

    public OrderJpaEntity toJpaEntity(Order domain) {
        OrderJpaEntity entity = new OrderJpaEntity();
        // 리플렉션 또는 빌더 패턴으로 필드 설정
        // (MapStruct를 사용하면 자동 생성)
        setField(entity, "orderId", domain.getId().value());
        setField(entity, "userId", domain.getUserId().value());
        setField(entity, "status", domain.getStatus().name());
        setField(entity, "totalAmount", domain.calculateTotal().getValue());
        setField(entity, "paymentTxId", domain.getPaymentTransactionId());
        return entity;
    }

    public Order toDomain(OrderJpaEntity entity) {
        return Order.reconstitute(
            OrderId.of(entity.getOrderId()),
            UserId.of(entity.getUserId()),
            OrderStatus.valueOf(entity.getStatus()),
            entity.getPaymentTxId()
        );
    }
}
```

### 3. Kafka 발행을 Port로 추상화

```java
// Before: OrderService가 KafkaTemplate을 직접 의존
@Service
public class OrderService {
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate; // ❌ Kafka 직접 의존

    public void placeOrder(...) {
        // ...
        kafkaTemplate.send("order-placed",
            order.getId().toString(),
            serialize(order)); // 직접 발행
    }
}

// After: Port 추상화

// Step 1: 이벤트 클래스 정의 (domain/event/)
public record OrderPlacedEvent(
    OrderId orderId,
    UserId userId,
    BigDecimal totalAmount,
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

// Step 2: Port 인터페이스 정의 (domain/port/out/)
public interface OrderEventPublisher {
    void publish(OrderPlacedEvent event);
    void publish(OrderCancelledEvent event);
}

// Step 3: Kafka Adapter 구현 (adapter/out/messaging/)
@Component
@RequiredArgsConstructor
public class KafkaOrderEventPublisher implements OrderEventPublisher {

    private final KafkaTemplate<String, String> kafkaTemplate; // Kafka는 Adapter에만
    private final ObjectMapper objectMapper;

    @Override
    public void publish(OrderPlacedEvent event) {
        try {
            String payload = objectMapper.writeValueAsString(
                OrderPlacedMessage.from(event) // 이벤트 → Kafka 메시지 변환
            );
            kafkaTemplate.send(
                "order.placed",           // 토픽
                event.orderId().value(),  // 키
                payload                   // 값
            );
        } catch (JsonProcessingException e) {
            throw new EventPublishException("OrderPlacedEvent 발행 실패", e);
        }
    }

    @Override
    public void publish(OrderCancelledEvent event) {
        try {
            String payload = objectMapper.writeValueAsString(
                OrderCancelledMessage.from(event)
            );
            kafkaTemplate.send("order.cancelled", event.orderId().value(), payload);
        } catch (JsonProcessingException e) {
            throw new EventPublishException("OrderCancelledEvent 발행 실패", e);
        }
    }
}

// Step 4: Kafka 메시지 형식 (adapter/out/messaging/)
// Kafka 페이로드 형식은 Adapter 레이어의 관심사
public record OrderPlacedMessage(
    String orderId,
    String userId,
    String totalAmount,
    String occurredAt
) {
    public static OrderPlacedMessage from(OrderPlacedEvent event) {
        return new OrderPlacedMessage(
            event.orderId().value(),
            event.userId().value(),
            event.totalAmount().toPlainString(),
            event.occurredAt().toString()
        );
    }
}

// Step 5: PlaceOrderService가 Port를 통해 발행
@Service
@Transactional
@RequiredArgsConstructor
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderSavePort orderSavePort;
    private final OrderEventPublisher eventPublisher; // Kafka 모름, Port만 앎
    private final PaymentPort paymentPort;

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.userId(), mapLines(command.lines()));
        order.place();

        PaymentResult payment = paymentPort.charge(PaymentRequest.of(order));
        order.confirmPayment(payment.transactionId());

        orderSavePort.save(order);
        eventPublisher.publish(OrderPlacedEvent.from(order)); // Port를 통해 발행

        return order.getId();
    }
}
```

### 4. InMemory Adapter 작성 및 단위 테스트

```java
// test/support/InMemoryOrderRepository.java
// OrderSavePort + OrderQueryPort 구현 (테스트 전용)
public class InMemoryOrderRepository implements OrderSavePort, OrderQueryPort {

    private final Map<String, Order> store = new LinkedHashMap<>();

    @Override
    public void save(Order order) {
        store.put(order.getId().value(), order);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id.value()));
    }

    @Override
    public List<Order> findByUserId(UserId userId) {
        return store.values().stream()
            .filter(o -> o.getUserId().equals(userId))
            .toList();
    }

    @Override
    public boolean existsById(OrderId id) {
        return store.containsKey(id.value());
    }

    // 테스트 헬퍼 메서드
    public int count() { return store.size(); }
    public void clear() { store.clear(); }
    public List<Order> findAll() { return new ArrayList<>(store.values()); }
}

// test/support/InMemoryOrderEventPublisher.java
public class InMemoryOrderEventPublisher implements OrderEventPublisher {

    private final List<OrderPlacedEvent> placedEvents = new ArrayList<>();
    private final List<OrderCancelledEvent> cancelledEvents = new ArrayList<>();

    @Override
    public void publish(OrderPlacedEvent event) {
        placedEvents.add(event);
    }

    @Override
    public void publish(OrderCancelledEvent event) {
        cancelledEvents.add(event);
    }

    public List<OrderPlacedEvent> getPlacedEvents() {
        return Collections.unmodifiableList(placedEvents);
    }
    public void clear() {
        placedEvents.clear();
        cancelledEvents.clear();
    }
}

// test/support/InMemoryPaymentPort.java
public class InMemoryPaymentPort implements PaymentPort {

    private int chargeCallCount = 0;
    private RuntimeException failWith = null;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        chargeCallCount++;
        if (failWith != null) throw failWith;
        return PaymentResult.success("tx-" + UUID.randomUUID());
    }

    public void willFail(RuntimeException ex) { this.failWith = ex; }
    public int chargeCallCount() { return chargeCallCount; }
    public void reset() { chargeCallCount = 0; failWith = null; }
}

// === PlaceOrderService 단위 테스트 (Spring 없음, 0.01초) ===
class PlaceOrderServiceTest {

    // InMemory Adapter 주입
    private final InMemoryOrderRepository orderRepository = new InMemoryOrderRepository();
    private final InMemoryOrderEventPublisher eventPublisher = new InMemoryOrderEventPublisher();
    private final InMemoryPaymentPort paymentPort = new InMemoryPaymentPort();

    // 테스트 대상 (Spring 없이 직접 생성)
    private final PlaceOrderService sut = new PlaceOrderService(
        orderRepository, eventPublisher, paymentPort
    );

    @BeforeEach
    void setUp() {
        orderRepository.clear();
        eventPublisher.clear();
        paymentPort.reset();
    }

    @Test
    void 정상_주문_생성_저장_이벤트_발행() {
        PlaceOrderCommand command = validCommand();

        OrderId orderId = sut.placeOrder(command);

        // 상태 기반 검증 (구현 세부사항 무관)
        assertThat(orderId).isNotNull();
        assertThat(orderRepository.findById(orderId)).isPresent();
        assertThat(orderRepository.findById(orderId).get().getStatus())
            .isEqualTo(OrderStatus.PAYMENT_CONFIRMED);

        // 이벤트 발행 검증
        assertThat(eventPublisher.getPlacedEvents()).hasSize(1);
        assertThat(eventPublisher.getPlacedEvents().get(0).orderId())
            .isEqualTo(orderId);

        // 결제 호출 검증
        assertThat(paymentPort.chargeCallCount()).isEqualTo(1);
    }

    @Test
    void 결제_실패_시_주문_저장_안_됨_이벤트_발행_안_됨() {
        paymentPort.willFail(new PaymentFailedException("카드 한도 초과"));

        assertThatThrownBy(() -> sut.placeOrder(validCommand()))
            .isInstanceOf(PaymentFailedException.class);

        // 결제 실패 시 저장 안 됨
        assertThat(orderRepository.count()).isEqualTo(0);
        // 결제 실패 시 이벤트 발행 안 됨
        assertThat(eventPublisher.getPlacedEvents()).isEmpty();
    }

    @Test
    void 최소_금액_미달_시_결제_API_호출_안_됨() {
        PlaceOrderCommand lowAmountCommand = PlaceOrderCommand.of(
            UserId.of("user-001"),
            List.of(PlaceOrderCommand.OrderLineCommand.of(
                ItemId.of("item-1"), 1, Money.of(500) // 최소 미달
            ))
        );

        assertThatThrownBy(() -> sut.placeOrder(lowAmountCommand))
            .isInstanceOf(MinOrderAmountException.class);

        // 비즈니스 규칙 위반 시 외부 API(결제) 호출 안 됨
        assertThat(paymentPort.chargeCallCount()).isEqualTo(0);
        assertThat(orderRepository.count()).isEqualTo(0);
    }

    private PlaceOrderCommand validCommand() {
        return PlaceOrderCommand.of(
            UserId.of("user-001"),
            List.of(PlaceOrderCommand.OrderLineCommand.of(
                ItemId.of("item-1"), 2, Money.of(5000)
            ))
        );
    }
}
// 실행 시간: ~0.01초 (Spring Context, DB, Kafka 없음)
```

### 5. JPA Adapter 통합 테스트 (독립적 검증)

```java
// JpaOrderRepository Adapter 독립 통합 테스트
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class JpaOrderRepositoryTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    SpringDataOrderJpa springDataJpa;

    private JpaOrderRepository sut;

    @BeforeEach
    void setUp() {
        sut = new JpaOrderRepository(springDataJpa, new OrderPersistenceMapper());
    }

    @Test
    void 저장_후_동일_ID로_조회_성공() {
        Order order = Order.create(UserId.of("u1"),
            List.of(OrderLine.of(ItemId.of("i1"), 2, Money.of(5000))));
        order.place();

        sut.save(order);
        Optional<Order> found = sut.findById(order.getId());

        assertThat(found).isPresent();
        assertThat(found.get().getUserId()).isEqualTo(UserId.of("u1"));
        assertThat(found.get().getStatus()).isEqualTo(OrderStatus.PLACED);
    }

    @Test
    void 없는_ID_조회_시_empty_반환() {
        Optional<Order> found = sut.findById(OrderId.of("non-exist"));
        assertThat(found).isEmpty();
    }
}
// 이 테스트는 Adapter 자체를 검증 (JPA 매핑, 쿼리 정확성)
// PlaceOrderService 단위 테스트와는 독립적
```

---

## 💻 실전 코드 — MapStruct로 매핑 코드 자동화

```java
// MapStruct 사용 시 매핑 코드 대폭 감소

// build.gradle.kts
dependencies {
    implementation("org.mapstruct:mapstruct:1.5.5.Final")
    annotationProcessor("org.mapstruct:mapstruct-processor:1.5.5.Final")
}

// OrderPersistenceMapper.java (MapStruct 버전)
@Mapper(componentModel = "spring",
        unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface OrderPersistenceMapper {

    @Mapping(source = "id.value", target = "orderId")
    @Mapping(source = "userId.value", target = "userId")
    @Mapping(source = "status", target = "status")  // Enum → String 자동
    OrderJpaEntity toJpaEntity(Order domain);

    @Mapping(source = "orderId", target = "id", qualifiedByName = "toOrderId")
    @Mapping(source = "userId", target = "userId", qualifiedByName = "toUserId")
    Order toDomain(OrderJpaEntity entity);

    @Named("toOrderId")
    default OrderId toOrderId(String value) { return OrderId.of(value); }

    @Named("toUserId")
    default UserId toUserId(String value) { return UserId.of(value); }
}
// 컴파일 시 구현체 자동 생성 → 매핑 코드 수동 작성 불필요
```

---

## 📊 패턴 비교 — 인프라 분리 전후 의존성 비교

```
인프라 분리 전 (의존성 방향):

  OrderController
       ↓
  OrderService ────→ OrderJpaRepository (Spring Data)
       │         ────→ KafkaTemplate (Kafka SDK)
       │         ────→ KakaoPayClient (외부 SDK)
       │         ────→ JavaMailSender (Spring)
       │         ────→ RedisTemplate (Spring)
       ↓
  Order (@Entity, Anemic)

  문제: OrderService의 Fan-out = 6개 (테스트 시 6개 Mock 필요)

인프라 분리 후 (의존성 방향):

  OrderController → PlaceOrderUseCase (Port 인터페이스)
                          ↓ implements
                   PlaceOrderService
                          │
              ┌───────────┼───────────┐
              ↓           ↓           ↓
        OrderSavePort  PaymentPort  OrderEventPublisher
        (인터페이스)   (인터페이스)  (인터페이스)
              ↑           ↑           ↑
     JpaOrderRepository KakaoPayAdapter KafkaPublisher
     (Adapter)         (Adapter)       (Adapter)

  장점: PlaceOrderService의 Fan-out = 3개 (Port 인터페이스만)
        단위 테스트: InMemory 3개로 대체 가능

테스트 비교:
  분리 전: @SpringBootTest + Mock 6개 = 30초
  분리 후: InMemory 3개 주입 = 0.01초 (3,000배 빠름)
```

---

## ⚖️ 트레이드오프

```
인프라 분리의 비용:
  매핑 코드: Domain ↔ JPA Entity 변환 (MapStruct로 절감)
  파일 수 증가: Port 인터페이스 + Adapter 클래스 + Mapper 클래스
  초기 설계 결정: 어떤 단위로 Port를 나눌 것인가 (Save/Query 분리 vs 통합)

인프라 분리의 이익:
  단위 테스트 속도: 30초 → 0.01초 (3,000배)
  외부 시스템 교체: Adapter 파일 하나만 변경
  테스트 커버리지: InMemory로 다양한 시나리오 테스트 가능 (결제 실패, DB 에러 등)
  Port 계약 명확화: "PlaceOrderService는 save()와 publish()만 필요" 문서화됨

InMemory 테스트의 한계:
  실제 DB 제약(unique, foreign key) 검증 안 됨
  → JPA Adapter 통합 테스트(@DataJpaTest)로 별도 검증 필수
  LAZY 로딩 예외 재현 안 됨
  → 트랜잭션 경계 테스트는 통합 테스트에서
```

---

## 📌 핵심 정리

```
인프라 레이어 분리 핵심:

Repository 이동:
  인터페이스: Infrastructure → domain/port/out/ (DIP 적용)
  구현체: 기존 위치 → adapter/out/persistence/ (Driven Adapter)
  Domain ↔ JPA Entity 변환: OrderPersistenceMapper

Kafka 추상화:
  KafkaTemplate 직접 의존 → OrderEventPublisher(Port) 인터페이스
  KafkaOrderEventPublisher: Port 구현 Adapter (Kafka SDK는 여기만)
  이벤트 형식 변환 (OrderPlacedEvent → Kafka 메시지): Adapter 책임

InMemory Adapter:
  OrderSavePort/OrderQueryPort 구현: Map<String, Order> 저장
  OrderEventPublisher 구현: List<Event> 수집
  PaymentPort 구현: 성공/실패 설정 가능 (willFail())

단위 테스트 결과:
  PlaceOrderService 테스트: InMemory 3개 주입 → 0.01초
  JpaOrderRepository 테스트: @DataJpaTest → 5-10초 (독립 검증)
  → 각 관심사를 독립적으로 빠르게 테스트 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `OrderSavePort`와 `OrderQueryPort`를 분리하지 않고 `OrderRepository` 하나로 통합해도 되는가? ISP 관점에서 어떻게 판단하는가?

<details>
<summary>해설 보기</summary>

**ISP(인터페이스 분리 원칙) 관점에서는 분리가 권장되지만, 실용적으로는 통합도 허용됩니다.**

분리 권장 경우:
```java
// PlaceOrderService: save()만 필요
class PlaceOrderService {
    private final OrderSavePort savePort; // OrderQueryPort 불필요
}

// FindOrderService: find*()만 필요
class FindOrderService {
    private final OrderQueryPort queryPort; // OrderSavePort 불필요
}
// → ISP 완전 적용, 각 Service가 필요한 Port만 의존
```

통합 허용 경우:
```java
// 대부분의 UseCase가 저장+조회 모두 필요
// 또는 팀이 단순성을 우선시
public interface OrderRepository extends OrderSavePort, OrderQueryPort {
    // 두 인터페이스를 통합
}
```

결정 기준:
- "InMemory 구현 시 불필요한 메서드가 많은가?" → 5개 이상이면 분리
- "각 UseCase가 명확히 다른 메서드만 사용하는가?" → 분리
- "단순성이 우선이고 UseCase가 몇 개 없는가?" → 통합 허용

</details>

---

**Q2.** `@Transactional`이 `PlaceOrderService`에 있는데, Kafka 이벤트 발행도 같은 트랜잭션에 포함된다. DB 저장 성공 + Kafka 발행 실패 시 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**DB 저장이 롤백되지 않는 불일치 문제가 생깁니다.** Kafka는 트랜잭션 롤백 대상이 아니기 때문입니다.

```java
@Transactional
public OrderId placeOrder(...) {
    orderSavePort.save(order);  // DB 저장 성공
    eventPublisher.publish(event); // Kafka 발행 실패 → 예외 발생
    // @Transactional 롤백: DB 저장 롤백됨
    // 그러나 만약 Kafka가 발행 성공 후 예외를 냈다면?
    // → DB는 롤백됐지만 Kafka 메시지는 이미 발행됨 → 불일치
}
```

해결: **Outbox Pattern**
```java
@Transactional
public OrderId placeOrder(...) {
    orderSavePort.save(order);

    // Kafka 직접 발행 대신 OutboxEvent DB에 저장 (같은 트랜잭션)
    outboxRepository.save(OutboxEvent.from(OrderPlacedEvent.from(order)));
    // 트랜잭션 커밋 → DB 저장 + Outbox 저장이 원자적

    // 별도 스케줄러가 Outbox를 읽어 Kafka에 발행 (트랜잭션 외부)
}
```

Outbox Pattern의 이익: DB 저장 성공 = Outbox 저장 성공 (원자적). Kafka 발행은 Eventually Consistent하게 처리.

</details>

---

**Q3.** InMemory Adapter 테스트가 통과했는데 실제 JPA Adapter에서 오류가 발생했다. 어떤 경우에 이런 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**InMemory는 JPA의 제약을 재현하지 않으므로 다음 경우에 불일치가 발생합니다.**

1. **유니크 제약**: InMemory는 중복 저장을 허용하지만 DB는 `UNIQUE` 제약 위반
   ```java
   // InMemory: 같은 orderId로 두 번 save() → 덮어씀
   // JPA: ConstraintViolationException 발생
   ```

2. **NULL 제약**: InMemory는 null 저장 가능, DB는 `NOT NULL` 제약
   ```java
   // Order에 userId가 null일 때
   // InMemory: 저장됨
   // JPA: DataIntegrityViolationException
   ```

3. **N+1 문제**: InMemory는 컬렉션을 즉시 반환, JPA는 LAZY 로딩
   ```java
   // order.getLines()
   // InMemory: 즉시 반환
   // JPA(@Transactional 밖): LazyInitializationException
   ```

방지 방법:
- `@DataJpaTest` + Testcontainers로 JPA Adapter 독립 테스트 필수
- 중요 비즈니스 시나리오는 `@SpringBootTest` E2E 테스트로도 검증
- InMemory 테스트와 JPA 테스트를 상호보완적으로 활용

</details>

---

<div align="center">

**[⬅️ 이전: 도메인 레이어 분리](./03-extracting-domain-layer.md)** | **[홈으로 🏠](../README.md)** | **[다음: 전후 비교 ➡️](./05-before-after-comparison.md)**

</div>
