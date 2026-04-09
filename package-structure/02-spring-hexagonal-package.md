# Spring Boot 실전 패키지 구조 — Hexagonal 기반 권장 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hexagonal Architecture 기반 Spring Boot 프로젝트의 표준 패키지 구조는?
- UseCase / Port / Adapter / Domain 각 파일의 정확한 위치 결정 기준은?
- 실제 Order 도메인으로 전체 구조를 어떻게 설명하는가?
- "어디에 놓아야 하나?" 판단을 위한 체크리스트는?
- 여러 팀이 사용하는 shared 패키지는 어떻게 관리하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

아키텍처 개념(Port, Adapter)을 알아도 "새 파일을 어디에 만드는가"의 기준이 없으면 팀마다 다른 위치에 파일을 생성하고, 결국 일관성이 깨진다.

이 문서는 Hexagonal Architecture를 Spring Boot 프로젝트에서 실제로 구현할 때의 표준 패키지 구조와 "파일 위치 결정 트리"를 제공한다. 이것을 ARCHITECTURE.md에 팀 표준으로 등록하면 신규 팀원도 바로 따를 수 있다.

---

## 😱 흔한 실수 (Before — Hexagonal 개념은 알지만 구조가 제각각)

```
팀 내 혼재:
  개발자 A: order/domain/OrderRepository.java (인터페이스)
  개발자 B: order/port/out/OrderRepository.java (인터페이스)
  개발자 C: order/application/port/OrderRepository.java (인터페이스)

세 가지 패턴이 혼재하는 이유:
  Hexagonal 개념은 이해하지만 "Spring Boot에서 어떤 패키지에?"의 기준 없음
  각자의 경험과 참조한 예제가 다름
  팀 표준 문서 없음

결과:
  "이 인터페이스가 Driving Port인지 Driven Port인지 패키지만 봐서 모름"
  ArchUnit 규칙 작성이 어려움 (패키지 경로가 통일되지 않아서)
  신규 팀원이 "어느 예제를 따라야 하나요?" 혼란
```

---

## ✨ 올바른 접근 (After — 팀 표준 패키지 구조)

```
권장 패키지 구조 (기능 기반 + Hexagonal):

com.example
├── {feature}/                       ← 기능별 패키지 (order, payment 등)
│   ├── domain/                      ← Entities + Ports
│   │   ├── model/                   ← Domain Entity, Value Object
│   │   │   ├── {Feature}.java       ← Aggregate Root
│   │   │   ├── {Feature}Status.java ← 상태 Enum
│   │   │   └── {Value}.java         ← Value Object
│   │   └── port/
│   │       ├── in/                  ← Driving Port (UseCase 인터페이스)
│   │       │   └── {Action}UseCase.java
│   │       └── out/                 ← Driven Port (Repository, API 인터페이스)
│   │           ├── {Feature}SavePort.java
│   │           ├── {Feature}QueryPort.java
│   │           └── {Service}Port.java
│   ├── application/                 ← Use Cases 구현
│   │   └── service/
│   │       └── {Action}Service.java ← @Service, UseCase 구현
│   └── adapter/                     ← Adapters
│       ├── in/
│       │   └── web/                 ← HTTP Driving Adapter
│       │       ├── {Feature}Controller.java
│       │       ├── request/         ← HTTP 요청 DTO
│       │       ├── response/        ← HTTP 응답 DTO
│       │       └── mapper/          ← HTTP ↔ UseCase 모델 변환
│       └── out/
│           ├── persistence/         ← DB Driven Adapter
│           │   ├── Jpa{Feature}Repository.java
│           │   ├── entity/          ← JPA Entity
│           │   └── mapper/          ← Domain ↔ JPA Entity 변환
│           ├── {service}/           ← 외부 API Driven Adapter
│           │   └── {Service}Adapter.java
│           └── messaging/           ← 메시지 Driven Adapter
│               └── {Feature}EventPublisher.java
└── shared/                          ← 공유 컴포넌트
    ├── domain/                      ← 공유 Value Object
    │   ├── UserId.java
    │   └── Money.java
    └── infrastructure/              ← 공유 인프라 설정
        ├── KafkaConfig.java
        └── JpaConfig.java

파일 위치 결정 기준:
  "비즈니스 규칙인가?" → domain/model/
  "외부에서 내 기능을 어떻게 호출하는가?" → domain/port/in/
  "내 기능이 외부를 어떻게 부르는가?" → domain/port/out/
  "비즈니스 흐름 조율인가?" → application/service/
  "HTTP 요청 처리인가?" → adapter/in/web/
  "DB 접근인가?" → adapter/out/persistence/
  "외부 API 호출인가?" → adapter/out/{service}/
```

---

## 🔬 내부 원리 — Order 도메인 전체 구조 예시

### 1. Domain 레이어 파일 예시

```java
// === domain/model/ ===

// order/domain/model/Order.java (Aggregate Root)
public class Order {
    private final OrderId id;
    private final UserId userId;
    private final List<OrderLine> lines;
    private OrderStatus status;
    
    public static Order create(UserId userId, List<OrderLine> lines) { ... }
    public static Order reconstitute(OrderId id, ...) { ... }
    public void place() { ... }
    public void cancel() { ... }
    public Money calculateTotal() { ... }
}

// order/domain/model/OrderLine.java (내부 Entity)
public class OrderLine {
    private final ItemId itemId;
    private final int quantity;
    private final Money unitPrice;
    public static OrderLine of(ItemId itemId, int quantity, Money unitPrice) { ... }
    public Money getSubtotal() { ... }
}

// order/domain/model/OrderStatus.java (Enum)
public enum OrderStatus {
    DRAFT, PLACED, PAYMENT_CONFIRMED, SHIPPING, DELIVERED, CANCELLED;

    public boolean canTransitionTo(OrderStatus next) {
        // 상태 전이 규칙도 Domain에
        return switch (this) {
            case DRAFT -> next == PLACED;
            case PLACED -> next == PAYMENT_CONFIRMED || next == CANCELLED;
            default -> false;
        };
    }
}

// === domain/port/in/ ===

// order/domain/port/in/PlaceOrderUseCase.java (Driving Port)
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

// order/domain/port/in/CancelOrderUseCase.java (Driving Port)
public interface CancelOrderUseCase {
    void cancelOrder(CancelOrderCommand command);
}

// order/domain/port/in/FindOrderQuery.java (Query Driving Port)
public interface FindOrderQuery {
    OrderDetail findById(OrderId id);
    Page<OrderSummary> findByUser(UserId userId, Pageable pageable);
}

// Command 객체 (UseCase 입력)
public record PlaceOrderCommand(
    UserId userId,
    List<OrderLineCommand> lines
) {
    public record OrderLineCommand(ItemId itemId, int quantity, Money unitPrice) {}
}

// === domain/port/out/ ===

// order/domain/port/out/OrderSavePort.java (Driven Port - 쓰기)
public interface OrderSavePort {
    void save(Order order);
}

// order/domain/port/out/OrderQueryPort.java (Driven Port - 읽기)
public interface OrderQueryPort {
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
}

// order/domain/port/out/PaymentPort.java (Driven Port - 외부 API)
public interface PaymentPort {
    PaymentResult charge(PaymentRequest request);
    void refund(String transactionId, Money amount);
}

// order/domain/port/out/OrderEventPublisher.java (Driven Port - 이벤트)
public interface OrderEventPublisher {
    void publishOrderPlaced(OrderPlacedEvent event);
    void publishOrderCancelled(OrderCancelledEvent event);
}
```

### 2. Application 레이어 파일 예시

```java
// === application/service/ ===

// order/application/service/PlaceOrderService.java
@Service
@Transactional
@RequiredArgsConstructor
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderSavePort orderSavePort;      // domain/port/out/
    private final PaymentPort paymentPort;           // domain/port/out/
    private final OrderEventPublisher eventPublisher; // domain/port/out/

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        List<OrderLine> lines = command.lines().stream()
            .map(l -> OrderLine.of(l.itemId(), l.quantity(), l.unitPrice()))
            .toList();

        Order order = Order.create(command.userId(), lines);
        order.place();

        PaymentResult payment = paymentPort.charge(PaymentRequest.of(order));
        order.confirmPayment(payment.transactionId());

        orderSavePort.save(order);
        eventPublisher.publishOrderPlaced(OrderPlacedEvent.from(order));

        return order.getId();
    }
}

// order/application/service/FindOrderService.java
@Service
@RequiredArgsConstructor
public class FindOrderService implements FindOrderQuery {

    private final OrderQueryPort orderQueryPort;

    @Override
    public OrderDetail findById(OrderId id) {
        return orderQueryPort.findById(id)
            .map(OrderDetail::from)
            .orElseThrow(OrderNotFoundException::new);
    }
}
```

### 3. Adapter 레이어 파일 예시

```java
// === adapter/in/web/ ===

// order/adapter/in/web/OrderController.java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final PlaceOrderUseCase placeOrderUseCase;
    private final FindOrderQuery findOrderQuery;
    private final OrderWebMapper mapper;

    @PostMapping
    public ResponseEntity<PlaceOrderResponse> placeOrder(
        @Valid @RequestBody PlaceOrderRequest request,
        @AuthenticationPrincipal UserPrincipal principal
    ) {
        PlaceOrderCommand command = mapper.toCommand(request, principal.getUserId());
        OrderId orderId = placeOrderUseCase.placeOrder(command);
        return ResponseEntity.created(URI.create("/api/orders/" + orderId.value()))
            .body(mapper.toResponse(orderId));
    }
}

// order/adapter/in/web/request/PlaceOrderRequest.java
public record PlaceOrderRequest(
    @NotEmpty List<OrderItemRequest> items
) {
    public record OrderItemRequest(
        @NotBlank String itemId,
        @Positive int quantity,
        @Positive long unitPrice
    ) {}
}

// order/adapter/in/web/mapper/OrderWebMapper.java
@Component
public class OrderWebMapper {
    public PlaceOrderCommand toCommand(PlaceOrderRequest req, String userId) {
        return new PlaceOrderCommand(
            UserId.of(userId),
            req.items().stream()
                .map(i -> new PlaceOrderCommand.OrderLineCommand(
                    ItemId.of(i.itemId()), i.quantity(), Money.of(i.unitPrice())))
                .toList()
        );
    }
    public PlaceOrderResponse toResponse(OrderId id) {
        return new PlaceOrderResponse(id.value());
    }
}

// === adapter/out/persistence/ ===

// order/adapter/out/persistence/JpaOrderRepository.java
@Repository
@RequiredArgsConstructor
public class JpaOrderRepository implements OrderSavePort, OrderQueryPort {

    private final SpringDataOrderJpa jpa;
    private final OrderPersistenceMapper mapper;

    @Override public void save(Order order) { jpa.save(mapper.toEntity(order)); }
    @Override public Optional<Order> findById(OrderId id) {
        return jpa.findByOrderId(id.value()).map(mapper::toDomain);
    }
}

// order/adapter/out/persistence/entity/OrderJpaEntity.java
@Entity @Table(name = "orders")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderJpaEntity {
    @Id @GeneratedValue private Long id;
    @Column(name = "order_id") private String orderId;
    @Column(name = "user_id") private String userId;
    @Enumerated(EnumType.STRING) private OrderStatus status;
}

// === adapter/out/payment/ ===

// order/adapter/out/payment/KakaoPayAdapter.java
@Component
@RequiredArgsConstructor
public class KakaoPayAdapter implements PaymentPort {
    private final KakaoPayClient client;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        KakaoPayResponse res = client.charge(KakaoPayRequest.from(request));
        return PaymentResult.of(res.getTransactionId());
    }
}
```

### 4. 파일 위치 결정 트리

```
새 파일을 어디에 만들어야 하는가?

① 이 코드가 비즈니스 규칙인가? (도메인 로직, 금액 계산, 상태 전이 등)
   → YES: {feature}/domain/model/ (Entity, Value Object, Enum)

② 이 코드가 "외부에서 내 기능을 어떻게 사용하는가"의 계약인가?
   → YES: {feature}/domain/port/in/ (UseCase 인터페이스)

③ 이 코드가 "내 기능이 외부를 어떻게 사용하는가"의 계약인가?
   → YES: {feature}/domain/port/out/ (Repository, API, Event 인터페이스)

④ 이 코드가 비즈니스 흐름 조율(UseCase 구현)인가?
   → YES: {feature}/application/service/ (@Service + @Transactional)

⑤ 이 코드가 HTTP 요청 처리인가?
   → YES: {feature}/adapter/in/web/ (@RestController, DTO, Mapper)

⑥ 이 코드가 DB 접근 구현인가?
   → YES: {feature}/adapter/out/persistence/ (JPA 구현체, @Entity, Mapper)

⑦ 이 코드가 외부 API 연동인가?
   → YES: {feature}/adapter/out/{service}/ (HTTP 클라이언트 Adapter)

⑧ 이 코드가 이벤트 발행/구독인가?
   → YES (발행): {feature}/adapter/out/messaging/
   → YES (구독): {feature}/adapter/in/event/

⑨ 여러 Feature가 공유하는가?
   → YES: shared/domain/ (Value Object) 또는 shared/infrastructure/ (설정)
```

---

## 💻 실전 코드 — 파일 위치 결정 트리 요약 코드

```java
// 새 파일 위치를 판단하는 실전 코드 예시

// Q: "Order의 최소 주문 금액 검증 로직 → 어디에?"
// A: "비즈니스 규칙이다" → {feature}/domain/model/Order.java
public class Order {
    public void place() {
        if (calculateTotal().isLessThan(Money.of(1_000)))
            throw new MinOrderAmountException(); // 비즈니스 규칙 → Domain
    }
}

// Q: "OrderController.java → 어디에?"
// A: "HTTP 요청 처리" → {feature}/adapter/in/web/OrderController.java
@RestController
public class OrderController {
    private final PlaceOrderUseCase useCase; // UseCase(Port)에만 의존
}

// Q: "PlaceOrderUseCase.java(인터페이스) → 어디에?"
// A: "외부에서 내 기능을 어떻게 호출하는가의 계약" → {feature}/domain/port/in/
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

// Q: "JpaOrderRepository.java → 어디에?"
// A: "DB 접근 구현" → {feature}/adapter/out/persistence/
@Repository
public class JpaOrderRepository implements OrderSavePort {
    // SpringDataOrderJpa (JPA 인프라)를 Port로 감쌈
}

// Q: "Money.java(여러 Feature 공유) → 어디에?"
// A: "여러 Feature 공유 Value Object" → shared/domain/Money.java
public record Money(BigDecimal value) {
    public boolean isLessThan(Money other) { ... }
    public Money add(Money other) { ... }
}
```

---

## 📊 패턴 비교 — 같은 파일이 어느 패키지에 있어야 하는지

```
파일 유형별 위치 비교:

파일 유형                    │ 위치
────────────────────────────┼────────────────────────────────────────
Order.java (도메인 객체)    │ order/domain/model/Order.java
OrderStatus.java (Enum)     │ order/domain/model/OrderStatus.java
Money.java (공유 VO)        │ shared/domain/Money.java
PlaceOrderUseCase.java      │ order/domain/port/in/PlaceOrderUseCase.java
OrderRepository.java (인터페이스) │ order/domain/port/out/OrderSavePort.java
PaymentPort.java (인터페이스) │ order/domain/port/out/PaymentPort.java
PlaceOrderService.java      │ order/application/service/PlaceOrderService.java
OrderController.java        │ order/adapter/in/web/OrderController.java
PlaceOrderRequest.java (DTO)│ order/adapter/in/web/request/PlaceOrderRequest.java
JpaOrderRepository.java     │ order/adapter/out/persistence/JpaOrderRepository.java
OrderJpaEntity.java         │ order/adapter/out/persistence/entity/OrderJpaEntity.java
KakaoPayAdapter.java        │ order/adapter/out/payment/KakaoPayAdapter.java
InMemoryOrderRepository.java│ (테스트) order/adapter/out/persistence/ 또는
                            │          test/support/InMemoryOrderRepository.java
```

---

## ⚖️ 트레이드오프

```
기능 기반 + Hexagonal 구조의 비용:
  파일 수 증가: 기능당 평균 15-20개 파일
  패키지 깊이: 4-5 단계 (feature/domain/port/in/UseCase.java)
  IDE 탐색: 폴더가 많아서 처음에 어색함

이익:
  "이 파일 어디?" → 위치 결정 트리로 즉시 찾음
  ArchUnit 규칙이 패키지 구조로 명확하게 설정 가능
  MSA 분리 시 feature/ 폴더 전체 이동으로 완결

IDE 설정으로 완화:
  IntelliJ "Compact Middle Packages" 옵션으로 패키지 트리 압축
  기능별 파일 탐색 → File Structure(Ctrl+F12)로 내부 구조 파악
```

---

## 📌 핵심 정리

```
Hexagonal 기반 Spring Boot 패키지 구조:

{feature}/domain/model/      → 도메인 객체 (순수 Java)
{feature}/domain/port/in/    → Driving Port (UseCase 인터페이스)
{feature}/domain/port/out/   → Driven Port (Repository, API 인터페이스)
{feature}/application/service/ → UseCase 구현 (@Service)
{feature}/adapter/in/web/    → HTTP Adapter (@RestController)
{feature}/adapter/out/persistence/ → DB Adapter (@Repository)
{feature}/adapter/out/{서비스}/ → 외부 API Adapter
shared/domain/               → 공유 Value Object

위치 결정 핵심 질문:
  "비즈니스 규칙인가?" → domain/model/
  "계약 정의인가?" → domain/port/in/ or out/
  "흐름 조율인가?" → application/service/
  "변환인가?" → adapter/
```

---

## 🤔 생각해볼 문제

**Q1.** `OrderQueryPort`와 `OrderSavePort`를 하나의 `OrderRepository` 인터페이스로 통합해도 되는가? 어떤 기준으로 결정하는가?

<details>
<summary>해설 보기</summary>

**팀 상황과 복잡도에 따라 결정합니다.**

분리 권장: ISP(인터페이스 분리 원칙) 적용 시
- `PlaceOrderService`는 `OrderSavePort`만 필요 (조회 메서드 불필요)
- 테스트 시 필요한 메서드만 구현하면 됨

통합 허용: 단순한 경우
- 대부분의 Service가 저장+조회를 모두 사용
- 파일 수 최소화 우선

결정 기준:
- "InMemory 구현체를 만들 때 불필요한 메서드가 10개 이상인가?" → 분리
- "UseCase별로 필요한 메서드가 명확히 다른가?" → 분리
- 그렇지 않으면 통합도 허용

</details>

---

**Q2.** `PlaceOrderCommand`는 `domain/port/in/` 패키지에 있는가, 별도 위치에 있는가?

<details>
<summary>해설 보기</summary>

Command 객체의 위치는 팀마다 다르지만 다음 두 방식이 일반적입니다.

```
방법 1: UseCase 인터페이스와 같은 패키지 (권장)
  order/domain/port/in/PlaceOrderUseCase.java
  order/domain/port/in/PlaceOrderCommand.java   ← 같은 위치

방법 2: application/service/ 패키지 내부
  order/application/service/PlaceOrderService.java
  order/application/service/PlaceOrderCommand.java

방법 3: 별도 command/ 패키지
  order/application/command/PlaceOrderCommand.java
```

권장은 방법 1 — UseCase 인터페이스와 그 입력을 같은 위치에 두면 "이 UseCase에 어떤 입력이 필요한가"가 명확합니다. Command는 UseCase 계약의 일부이므로 `port/in/`에 있는 것이 논리적입니다.

</details>

---

**Q3.** `InMemoryOrderRepository`(테스트용 Adapter)는 어디에 위치해야 하는가? `src/main`인가 `src/test`인가?

<details>
<summary>해설 보기</summary>

**`src/test`가 권장이지만, 여러 테스트에서 재사용한다면 별도 모듈도 고려합니다.**

```
방법 1: src/test (기본, 단일 모듈)
  src/test/java/com/example/order/support/InMemoryOrderRepository.java
  테스트 코드에서만 사용, 프로덕션 빌드에 포함되지 않음

방법 2: 별도 테스트 지원 모듈 (멀티 모듈)
  order-test-support/src/main/InMemoryOrderRepository.java
  다른 모듈에서 testImplementation으로 의존

방법 3: src/main (과잉, 피해야 함)
  프로덕션 코드에 테스트 용도 클래스가 포함됨 → 피해야 함
```

단일 모듈 프로젝트: `src/test`의 공용 패키지(`support/`)에 위치
멀티 모듈 프로젝트: 테스트 지원 전용 모듈(`-test-support`)로 분리

</details>

---

<div align="center">

**[⬅️ 이전: 패키지 구조의 원칙](./01-package-structure-principles.md)** | **[홈으로 🏠](../README.md)** | **[다음: 멀티 모듈 프로젝트 ➡️](./03-multi-module-project.md)**

</div>
