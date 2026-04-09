# Spring으로 Hexagonal 구현 — 패키지 구조와 매핑

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hexagonal Architecture를 Spring Boot 프로젝트에 적용할 때 권장 패키지 구조는?
- `@Service`, `@Repository`, `@Controller`가 각각 어느 역할(Port/Adapter)에 매핑되는가?
- JPA Entity와 Domain Entity를 분리할 때 매핑 코드는 어떻게 작성하는가?
- Spring의 의존성 주입(DI)이 Port/Adapter 구조와 어떻게 자연스럽게 맞는가?
- 패키지 구조를 기술(tech) 기준으로 나눌 때와 도메인(domain) 기준으로 나눌 때의 차이는?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Hexagonal Architecture의 개념을 알아도, Spring Boot 프로젝트에서 어떻게 파일을 배치하고 어노테이션을 어디에 붙일지 모르면 구현이 막힌다.

이 문서는 개념을 실제 Spring Boot 코드로 옮기는 구체적인 방법을 다룬다. 패키지 구조, 매핑 코드, Spring DI 설정까지 실무에서 바로 적용 가능한 패턴을 제공한다.

---

## 😱 흔한 실수 (Before — 패키지 구조가 개념과 불일치할 때)

```
혼재된 패키지 구조 (흔한 실수):

com.example
  ├── controller/
  │     OrderController.java
  ├── service/
  │     OrderService.java        ← 도메인 로직 + 인프라 로직 혼재
  ├── repository/
  │     OrderRepository.java     ← JpaRepository 상속 인터페이스 (Infrastructure!)
  ├── entity/
  │     Order.java               ← @Entity + 비즈니스 로직 혼재
  └── dto/
        PlaceOrderRequest.java
        PlaceOrderResponse.java

문제:
  ① repository/ 안의 OrderRepository가 Infrastructure인지 Domain Port인지 불분명
  ② controller/, service/, repository/는 기술 분류 — 도메인 경계 없음
  ③ 새 팀원이 "도메인 로직을 어디에 넣어야 하나요?" 매번 질문
  ④ ArchUnit으로 Hexagonal 규칙 강제하기 어려움 (패키지 구조가 역할을 반영 안 함)
```

---

## ✨ 올바른 접근 (After — 역할이 명확한 패키지 구조)

```
권장 패키지 구조:

com.example
  ├── domain/                           ← Application Core (순수 비즈니스)
  │     ├── model/
  │     │     ├── Order.java            ← Domain Entity
  │     │     ├── OrderLine.java        ← Domain Entity
  │     │     ├── OrderStatus.java      ← Enum
  │     │     └── Money.java            ← Value Object
  │     └── port/
  │           ├── in/                  ← Driving Ports (UseCase 인터페이스)
  │           │     ├── PlaceOrderUseCase.java
  │           │     └── FindOrderQuery.java
  │           └── out/                 ← Driven Ports (Repository, API 인터페이스)
  │                 ├── OrderSavePort.java
  │                 ├── OrderQueryPort.java
  │                 └── PaymentPort.java
  ├── application/                     ← Application Layer (UseCase 구현)
  │     └── service/
  │           └── PlaceOrderService.java
  └── adapter/                         ← Adapters (인프라 연결)
        ├── in/
        │     └── web/                 ← Driving Adapters
        │           ├── OrderController.java
        │           ├── dto/
        │           │     ├── PlaceOrderRequest.java
        │           │     └── PlaceOrderResponse.java
        │           └── mapper/
        │                 └── OrderWebMapper.java
        └── out/
              ├── persistence/         ← Driven Adapters (DB)
              │     ├── JpaOrderRepository.java
              │     ├── entity/
              │     │     └── OrderJpaEntity.java
              │     └── mapper/
              │           └── OrderPersistenceMapper.java
              └── payment/             ← Driven Adapters (외부 API)
                    └── KakaoPayAdapter.java
```

---

## 🔬 내부 원리 — Spring 컴포넌트와 Hexagonal 역할 매핑

### 1. Spring 어노테이션과 Hexagonal 역할 매핑

```
@RestController → Driving Adapter
  역할: HTTP 요청을 Command로 변환하고 UseCase를 호출
  위치: adapter/in/web/
  Spring 역할: HTTP 요청 처리 + 응답 직렬화

@Service (UseCase 구현) → Application Core
  역할: Driving Port(UseCase)를 구현하는 Application Service
  위치: application/service/
  Spring 역할: 트랜잭션 관리 (@Transactional)

interface (Driving Port) → Port (in)
  역할: 외부에서 Application Core로 들어오는 계약
  위치: domain/port/in/
  Spring 역할: 없음 (순수 Java 인터페이스)

interface (Driven Port) → Port (out)
  역할: Application Core에서 외부로 나가는 계약
  위치: domain/port/out/
  Spring 역할: 없음 (순수 Java 인터페이스)

@Repository → Driven Adapter (DB)
  역할: Driven Port(OrderSavePort 등)를 구현
  위치: adapter/out/persistence/
  Spring 역할: JPA 예외 변환, Bean 등록

@Component → Driven Adapter (외부 API, Kafka 등)
  역할: Driven Port(PaymentPort 등)를 구현
  위치: adapter/out/payment/ 등
  Spring 역할: Bean 등록
```

### 2. UseCase 인터페이스 정의와 구현

```java
// === domain/port/in/PlaceOrderUseCase.java ===
// 순수 Java 인터페이스 — Spring 없음
public interface PlaceOrderUseCase {

    /**
     * 주문을 생성합니다.
     * @param command 주문 생성에 필요한 데이터
     * @return 생성된 주문 ID
     * @throws MinOrderAmountException 최소 주문 금액 미달
     * @throws InsufficientStockException 재고 부족
     */
    OrderId placeOrder(PlaceOrderCommand command);
}

// Command: UseCase의 입력 (HTTP DTO와 분리)
public record PlaceOrderCommand(
    UserId userId,
    List<OrderLineCommand> lines
) {
    // 도메인 입력이므로 HTTP 어노테이션(@JsonProperty 등) 없음
    public static PlaceOrderCommand of(UserId userId, List<OrderLineCommand> lines) {
        Objects.requireNonNull(userId, "userId required");
        if (lines == null || lines.isEmpty()) throw new IllegalArgumentException("lines required");
        return new PlaceOrderCommand(userId, lines);
    }
}

// === application/service/PlaceOrderService.java ===
@Service
@Transactional  // Application Layer에서 트랜잭션 경계 정의
@RequiredArgsConstructor
public class PlaceOrderService implements PlaceOrderUseCase {

    // Driven Port만 의존 — 인프라 클래스 없음
    private final OrderSavePort orderSavePort;
    private final OrderQueryPort orderQueryPort;
    private final PaymentPort paymentPort;
    private final OrderEventPublisher eventPublisher;

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        // 도메인 로직 실행
        Order order = Order.create(command.userId(), command.lines());
        order.place();

        // Driven Port 통해 외부 호출
        PaymentResult payment = paymentPort.charge(PaymentRequest.of(order));
        order.confirmPayment(payment.transactionId());

        // 저장 및 이벤트
        orderSavePort.save(order);
        eventPublisher.publish(new OrderPlacedEvent(order));

        return order.getId();
    }
}
```

### 3. JPA Entity와 Domain Entity 분리 및 매핑

```java
// === domain/model/Order.java (Domain Entity) ===
public class Order {
    private final OrderId id;
    private final UserId userId;
    private final List<OrderLine> lines;
    private OrderStatus status;
    private String paymentTransactionId;

    // 정적 팩토리 — 기본 생성자 없음 (불완전 상태 생성 불가)
    public static Order create(UserId userId, List<OrderLine> lines) { ... }

    // 비즈니스 메서드
    public void place() { ... }
    public void confirmPayment(String txId) { ... }
    public Money calculateTotal() { ... }

    // getter만 (setter 없음 — 불변성)
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
}

// === adapter/out/persistence/entity/OrderJpaEntity.java (JPA Entity) ===
@Entity
@Table(name = "orders")
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA 요구 기본 생성자
@Getter
public class OrderJpaEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_id", unique = true, nullable = false)
    private String orderId;

    @Column(name = "user_id", nullable = false)
    private String userId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OrderStatus status;

    @Column(name = "payment_tx_id")
    private String paymentTransactionId;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLineJpaEntity> lines = new ArrayList<>();
}

// === adapter/out/persistence/mapper/OrderPersistenceMapper.java ===
@Component
public class OrderPersistenceMapper {

    // Domain → JPA Entity (저장 시)
    public OrderJpaEntity toJpaEntity(Order domain) {
        OrderJpaEntity entity = new OrderJpaEntity();
        // reflection을 쓰거나 Builder 패턴 활용
        // MapStruct 사용 권장 (자동 생성)
        entity.setOrderId(domain.getId().value());
        entity.setUserId(domain.getUserId().value());
        entity.setStatus(domain.getStatus());
        entity.setPaymentTransactionId(domain.getPaymentTransactionId());
        entity.setLines(
            domain.getLines().stream()
                .map(this::toLineEntity)
                .toList()
        );
        return entity;
    }

    // JPA Entity → Domain (조회 시)
    public Order toDomain(OrderJpaEntity entity) {
        return Order.reconstitute(  // 복원용 정적 메서드 (DB에서 복원)
            OrderId.of(entity.getOrderId()),
            UserId.of(entity.getUserId()),
            entity.getLines().stream().map(this::toLineDomain).toList(),
            entity.getStatus(),
            entity.getPaymentTransactionId()
        );
    }
}

// === adapter/out/persistence/JpaOrderRepository.java ===
@Repository
@RequiredArgsConstructor
public class JpaOrderRepository implements OrderSavePort, OrderQueryPort {

    private final SpringDataOrderJpaRepository jpa;
    private final OrderPersistenceMapper mapper;

    @Override
    public void save(Order order) {
        jpa.save(mapper.toJpaEntity(order));
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findByOrderId(id.value())
            .map(mapper::toDomain);
    }

    @Override
    public List<Order> findByUserId(UserId userId) {
        return jpa.findByUserId(userId.value()).stream()
            .map(mapper::toDomain)
            .toList();
    }
}

// Spring Data JPA 인터페이스 (Infrastructure에만 존재)
interface SpringDataOrderJpaRepository extends JpaRepository<OrderJpaEntity, Long> {
    Optional<OrderJpaEntity> findByOrderId(String orderId);
    List<OrderJpaEntity> findByUserId(String userId);
}
```

### 4. Controller와 Command 매핑

```java
// === adapter/in/web/dto/PlaceOrderRequest.java ===
// HTTP DTO — HTTP 어노테이션만 포함
public record PlaceOrderRequest(
    @NotBlank String userId,
    @NotEmpty @Valid List<OrderItemRequest> items
) {
    public record OrderItemRequest(
        @NotBlank String itemId,
        @Positive int quantity
    ) {}
}

// === adapter/in/web/mapper/OrderWebMapper.java ===
@Component
public class OrderWebMapper {

    // HTTP DTO → Domain Command
    public PlaceOrderCommand toCommand(PlaceOrderRequest request, String principalUserId) {
        return PlaceOrderCommand.of(
            UserId.of(principalUserId),
            request.items().stream()
                .map(i -> OrderLineCommand.of(ItemId.of(i.itemId()), i.quantity()))
                .toList()
        );
    }

    // Domain ID → HTTP Response
    public PlaceOrderResponse toResponse(OrderId orderId) {
        return new PlaceOrderResponse(orderId.value());
    }
}

// === adapter/in/web/OrderController.java ===
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final PlaceOrderUseCase placeOrderUseCase;  // Driving Port만 의존
    private final FindOrderQuery findOrderQuery;
    private final OrderWebMapper mapper;

    @PostMapping
    public ResponseEntity<PlaceOrderResponse> placeOrder(
        @Valid @RequestBody PlaceOrderRequest request,
        @AuthenticationPrincipal UserPrincipal principal
    ) {
        // HTTP → Command 변환
        PlaceOrderCommand command = mapper.toCommand(request, principal.getUserId());
        // UseCase 호출
        OrderId orderId = placeOrderUseCase.placeOrder(command);
        // 결과 → HTTP 응답 변환
        return ResponseEntity
            .created(URI.create("/api/orders/" + orderId.value()))
            .body(mapper.toResponse(orderId));
    }
}
```

### 5. 도메인별 패키지 구조 (Advanced)

```
기술별 구조 (단순, 소규모):
  com.example
    ├── domain/port/in/
    ├── domain/port/out/
    ├── domain/model/
    ├── application/service/
    └── adapter/in|out/

도메인별 구조 (대규모, Bounded Context):
  com.example
    ├── order/                           ← Order Bounded Context
    │     ├── domain/port/in/
    │     ├── domain/port/out/
    │     ├── domain/model/
    │     ├── application/service/
    │     └── adapter/in|out/
    ├── payment/                         ← Payment Bounded Context
    │     ├── domain/port/in/
    │     ...
    └── inventory/                       ← Inventory Bounded Context
          ...

도메인별 구조의 이점:
  각 Bounded Context를 독립 모듈/서비스로 분리 용이
  팀이 Bounded Context 단위로 나뉠 때 코드 소유권 명확
  마이크로서비스 분리 시 패키지 → 독립 서비스로 자연스럽게 이행
```

---

## 💻 실전 코드 — 전체 패키지 구조와 Spring Bean 설정

```java
// === Spring 설정 — Port와 Adapter 연결 ===
@Configuration
public class OrderBeanConfiguration {

    // Driven Adapter를 Driven Port에 바인딩
    // Spring DI가 자동으로 처리 (같은 타입의 Bean이 하나면 자동 주입)

    @Bean
    @ConditionalOnProperty("payment.provider", havingValue = "kakao", matchIfMissing = true)
    public PaymentPort kakaoPaymentPort(KakaoPayClient client) {
        return new KakaoPayAdapter(client);
    }

    @Bean
    @ConditionalOnProperty("payment.provider", havingValue = "toss")
    public PaymentPort tossPaymentPort(TossPayClient client) {
        return new TossPayAdapter(client);
    }
}

// application.yml 설정으로 Adapter 교체
// payment.provider: kakao  → KakaoPayAdapter 주입
// payment.provider: toss   → TossPayAdapter 주입
// PlaceOrderService 코드 변경 없음

// === Value Object 설계 예시 ===
public record OrderId(String value) {
    public OrderId {
        Objects.requireNonNull(value);
        if (value.isBlank()) throw new IllegalArgumentException("OrderId 빈 값 불가");
    }
    public static OrderId newId() { return new OrderId(UUID.randomUUID().toString()); }
    public static OrderId of(String value) { return new OrderId(value); }
}

public record UserId(String value) {
    public UserId {
        Objects.requireNonNull(value);
    }
    public static UserId of(String value) { return new UserId(value); }
}

public record Money(BigDecimal value) {
    public Money {
        Objects.requireNonNull(value);
        if (value.compareTo(BigDecimal.ZERO) < 0) throw new IllegalArgumentException("음수 불가");
    }
    public static Money of(long amount) { return new Money(BigDecimal.valueOf(amount)); }
    public Money add(Money other) { return new Money(this.value.add(other.value)); }
    public boolean isLessThan(Money other) { return this.value.compareTo(other.value) < 0; }
    public static Money ZERO = Money.of(0);
}
```

---

## 📊 패턴 비교 — 기술 기준 vs 도메인 기준 패키지

```
기술 기준 패키지 (전통적 Spring Boot):
  controller/
    OrderController.java
    ProductController.java
  service/
    OrderService.java
    ProductService.java
  repository/
    OrderRepository.java
    ProductRepository.java

문제:
  "주문 기능을 다른 서비스로 분리하려면?" → 여러 패키지에서 Order 관련 파일 수집
  "주문 도메인 경계가 어디인가?" → 패키지 구조에서 파악 불가

도메인 기준 패키지 (Hexagonal 권장):
  order/
    domain/model/Order.java
    domain/port/in/PlaceOrderUseCase.java
    domain/port/out/OrderSavePort.java
    application/service/PlaceOrderService.java
    adapter/in/web/OrderController.java
    adapter/out/persistence/JpaOrderRepository.java
  product/
    domain/model/Product.java
    ...

이점:
  "주문 기능을 다른 서비스로 분리하려면?" → order/ 패키지 전체 이동
  "주문 도메인 경계가 어디인가?" → order/ 패키지가 경계
```

---

## ⚖️ 트레이드오프

```
Domain/JPA Entity 분리 비용:
  Mapper 클래스 (OrderPersistenceMapper) 작성 필요
  파일 수 증가: Order + OrderJpaEntity + OrderPersistenceMapper
  MapStruct 사용으로 일부 자동화 가능

Domain/JPA Entity 통합 절충 허용 조건:
  JPA 제약(@GeneratedValue, LAZY 로딩)이 도메인 메서드에 영향 없을 때
  팀 규모가 작고 빠른 개발이 우선일 때
  단순 CRUD 위주여서 도메인 로직이 많지 않을 때

MapStruct로 매핑 비용 줄이기:
  @Mapper(componentModel = "spring")
  public interface OrderPersistenceMapper {
      OrderJpaEntity toJpaEntity(Order order);
      Order toDomain(OrderJpaEntity entity);
  }
  → 컴파일 타임 코드 생성, 리플렉션 없이 타입 안전 매핑
```

---

## 📌 핵심 정리

```
Spring Boot + Hexagonal 패키지 구조:

domain/port/in/  → Driving Port (UseCase 인터페이스)
domain/port/out/ → Driven Port (Repository, API 인터페이스)
domain/model/    → Domain Entity, Value Object
application/service/ → UseCase 구현 (@Service)
adapter/in/web/  → Driving Adapter (@RestController)
adapter/out/persistence/ → Driven Adapter (@Repository)
adapter/out/*/   → Driven Adapter (외부 API, Kafka 등)

Spring 어노테이션 위치:
  @RestController → adapter/in/web/ (Driving Adapter)
  @Service        → application/service/ (UseCase 구현)
  @Repository     → adapter/out/persistence/ (Driven Adapter)
  @Component      → adapter/out/*/ (기타 Driven Adapter)
  없음            → domain/ (순수 Java)

JPA Entity 분리:
  domain/model/Order.java → 비즈니스 로직, 기본 생성자 없음
  adapter/out/persistence/entity/OrderJpaEntity.java → @Entity, JPA 설정
  adapter/out/persistence/mapper/OrderPersistenceMapper.java → 변환
```

---

## 🤔 생각해볼 문제

**Q1.** `domain/port/out/OrderSavePort.java`와 `domain/port/out/OrderQueryPort.java`를 분리하지 않고 `domain/port/out/OrderRepository.java`로 통합해도 되는가? 어떤 기준으로 결정하는가?

<details>
<summary>해설 보기</summary>

**팀 규모와 복잡도에 따라 선택합니다.**

분리 권장: `PlaceOrderService`가 저장만 하고, `FindOrderService`가 조회만 할 때
- ISP 준수: 각 Service가 필요한 Port만 의존
- 테스트 단순화: Save 테스트에 Query 메서드 구현 불필요

통합 허용: 대부분의 Service가 저장+조회 모두 사용할 때
- 파일 수 감소
- ISP를 약간 포기하지만 실용성 확보

시작점으로 `OrderRepository` 통합 → 나중에 분리가 필요해지면 분리하는 점진적 접근도 유효합니다.

</details>

---

**Q2.** `PlaceOrderCommand`를 record로 정의할 때와 class로 정의할 때의 차이는? Hexagonal에서 Command는 왜 불변이어야 하는가?

<details>
<summary>해설 보기</summary>

**record가 더 적합합니다.** Java 16+의 `record`는 자동으로 불변(final 필드), equals/hashCode/toString을 제공합니다.

Command가 불변이어야 하는 이유:
1. **UseCase 진입 시 데이터가 변경되면 안 됨**: PlaceOrderService가 Command를 처리하는 동안 중간에 값이 바뀌면 예측 불가
2. **Thread Safety**: 동시 요청 처리 시 Command가 불변이면 공유 상태 문제 없음
3. **명시적 입력 계약**: "이 UseCase에 이 데이터가 필요하다"를 불변 객체로 명확하게 표현

```java
// record: 불변, 간결
public record PlaceOrderCommand(UserId userId, List<OrderLineCommand> lines) {}

// class: 가변 위험, 장황
public class PlaceOrderCommand {
    private UserId userId; // setter가 있으면 외부에서 변경 가능
    private List<OrderLineCommand> lines;
    // getter, setter, equals, hashCode, toString 직접 작성
}
```

</details>

---

**Q3.** Spring Boot의 자동 설정과 Hexagonal Architecture는 충돌하는가? `@SpringBootApplication`이 모든 컴포넌트를 스캔하면 패키지 경계가 무너지지 않는가?

<details>
<summary>해설 보기</summary>

**충돌하지 않습니다.** Spring의 컴포넌트 스캔은 Bean 등록 메커니즘이고, Hexagonal의 패키지 경계는 아키텍처 규칙입니다.

Spring DI는 Hexagonal을 지원합니다:
```java
// Spring이 자동으로 올바른 Adapter를 Port에 주입
PlaceOrderService의 OrderSavePort → JpaOrderRepository (자동 주입)
PlaceOrderService의 PaymentPort → KakaoPayAdapter (자동 주입)

// Spring DI 덕분에 PlaceOrderService 코드에 new JpaOrderRepository() 없음
// = 코드 레벨에서 인프라 의존 없음, 런타임에만 주입
```

패키지 경계는 ArchUnit으로 강제합니다:
```java
// Spring의 컴포넌트 스캔은 모든 패키지를 보지만
// ArchUnit은 "domain 패키지 코드가 infrastructure를 import하면 실패"를 강제
// → 런타임 주입(Spring)과 컴파일 타임 의존성 검사(ArchUnit)가 상호보완
```

</details>

---

<div align="center">

**[⬅️ 이전: 의존성 방향 완전 분석](./04-dependency-direction-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: 테스트 용이성 향상 ➡️](./06-testability-with-hexagonal.md)**

</div>
