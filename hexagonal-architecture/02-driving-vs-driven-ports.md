# 주도하는 포트 vs 주도받는 포트 — Driving vs Driven

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Driving Port(주도하는 포트)와 Driven Port(주도받는 포트)는 어떻게 다른가?
- UseCase 인터페이스가 왜 Driving Port인가?
- Repository/MessagePublisher 인터페이스가 왜 Driven Port이고, 왜 Domain Layer에 위치해야 하는가?
- 두 종류의 Port 모두 인터페이스인데 방향이 왜 다른가?
- "주도한다(Drive)"는 표현은 어떤 의미인가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Hexagonal Architecture를 처음 배울 때 가장 혼란스러운 부분이 Port의 방향이다. "UseCase도 인터페이스고 Repository도 인터페이스인데, 왜 하나는 Driving Port고 다른 하나는 Driven Port인가?"

이 차이를 이해해야 Port를 어디에 놓을지, Adapter를 어느 쪽에서 구현할지 명확해진다. 방향을 잘못 이해하면 Controller가 UseCase를 구현하거나, Repository 인터페이스를 Infrastructure Layer에 두는 실수가 발생한다.

---

## 😱 흔한 실수 (Before — Driving/Driven을 구분하지 못할 때)

```
혼동 상황:
  "Port는 다 인터페이스잖아요. 어떤 게 Driving이고 어떤 게 Driven인지 모르겠어요"

잘못된 구현:
  ① Driven Port를 Infrastructure Layer에 위치
    infrastructure/
      OrderRepository.java (인터페이스)  ← 잘못된 위치
      JpaOrderRepository.java (구현체)

    결과:
      Application(Service)이 Infrastructure Layer의 인터페이스에 의존
      = 의존성이 여전히 인프라 방향 (DIP 미적용)

  ② Driving Port 방향 혼동
    OrderController implements PlaceOrderUseCase  ← Controller가 UseCase를 구현?!
    PlaceOrderService.execute(controller)         ← Service가 Controller를 호출?

  ③ UseCase마다 Repository를 직접 주입
    @RestController
    public class OrderController {
        @Autowired OrderRepository orderRepository; // ← Controller가 Driven Port에 직접 접근
        // Application Core 건너뜀
    }
```

---

## ✨ 올바른 접근 (After — Driving/Driven 명확히 구분)

```
구분 기준: "누가 애플리케이션을 주도하는가?"

Driving Side (주도하는 쪽):
  외부 액터(사람, 자동화 시스템)가 애플리케이션을 호출
  → HTTP 클라이언트가 Controller를 호출
  → Controller가 UseCase(Driving Port)를 호출
  → 애플리케이션을 "주도"하는 쪽 = Driving

  Driving Port: PlaceOrderUseCase (인터페이스)
    → 외부에서 애플리케이션으로 들어오는 진입점
    → Application Core가 정의, Adapter(Controller)가 호출

  Driving Adapter: OrderController
    → Driving Port를 호출하는 쪽

Driven Side (주도받는 쪽):
  애플리케이션이 외부 인프라를 호출
  → Application Core가 OrderRepository(Driven Port)를 호출
  → JpaOrderRepository(Driven Adapter)가 DB에 저장
  → 애플리케이션에게 "주도받는" 쪽 = Driven

  Driven Port: OrderRepository (인터페이스)
    → 애플리케이션에서 외부 인프라로 나가는 계약
    → Application Core가 정의, Adapter(JpaOrderRepository)가 구현

  Driven Adapter: JpaOrderRepository
    → Driven Port를 구현하는 쪽
```

---

## 🔬 내부 원리

### 1. Driving Port — 애플리케이션의 진입점

```
Driving Port의 본질:
  애플리케이션 코어가 외부에 제공하는 API (인터페이스)
  "이 방법으로 나를 사용할 수 있다"는 선언

UseCase 인터페이스가 Driving Port인 이유:
  PlaceOrderUseCase.placeOrder(command)
  → 이것이 애플리케이션이 외부에 제공하는 기능
  → 어떤 Adapter(HTTP, GraphQL, CLI, Test)가 호출해도 동일하게 동작

  // Driving Port 정의 (Application Core에 위치)
  public interface PlaceOrderUseCase {  // Driving Port
      OrderId placeOrder(PlaceOrderCommand command);
  }

  // Application Core가 구현
  @Service
  public class PlaceOrderService implements PlaceOrderUseCase {
      @Override
      public OrderId placeOrder(PlaceOrderCommand command) { ... }
  }

  // Driving Adapter들이 호출
  OrderController.placeOrder()        → useCase.placeOrder(command) // HTTP
  OrderCLICommand.execute()           → useCase.placeOrder(command) // CLI
  PlaceOrderServiceTest.test()        → useCase.placeOrder(command) // Test
  OrderBatchJob.run()                 → useCase.placeOrder(command) // Batch

같은 Driving Port, 다른 Driving Adapter:
  → 어떤 Adapter가 호출해도 Application Core는 동일
  → 테스트 시 Test Adapter로 직접 호출 가능 → Spring 없이 테스트!
```

### 2. Driven Port — 애플리케이션이 외부에 요청하는 계약

```
Driven Port의 본질:
  애플리케이션 코어가 인프라에게 요청하는 API (인터페이스)
  "너는 이 방법으로 나를 도와주면 된다"는 선언

Driven Port가 Domain Layer에 위치해야 하는 이유:

  잘못된 위치 (Infrastructure Layer):
    domain/PlaceOrderService → infrastructure/OrderRepository (인터페이스) 의존
    = Service(domain)이 Infrastructure Layer에 의존
    = DIP 미적용, 의존성이 인프라 방향

  올바른 위치 (Domain Layer):
    domain/PlaceOrderService → domain/OrderRepository (인터페이스) 의존
    infrastructure/JpaOrderRepository → domain/OrderRepository 구현
    = 인프라가 Domain Layer에 의존 (DIP!)

  // Driven Port 정의 (Domain/Application Layer에 위치)
  public interface OrderRepository {  // Driven Port
      void save(Order order);
      Optional<Order> findById(OrderId id);
  }

  // Driven Adapter가 구현 (Infrastructure Layer에 위치)
  @Repository
  public class JpaOrderRepository implements OrderRepository {
      @Override public void save(Order order) { ... }
      @Override public Optional<Order> findById(OrderId id) { ... }
  }

  // Application Core가 사용 (Port만 알고 Adapter는 모름)
  @Service
  public class PlaceOrderService {
      private final OrderRepository orderRepository; // Driven Port만 의존
      // JpaOrderRepository를 전혀 모름
  }
```

### 3. 두 Port의 구조적 차이

```
Driving Port vs Driven Port 비교:

              Driving Port              │  Driven Port
──────────────────────────────────────┼─────────────────────────────────────
정의 위치      Application Core          │  Application Core (Domain Layer)
구현 위치      Application Core          │  Infrastructure Layer (Adapter)
호출 방향      외부(Adapter) → Core       │  Core → 외부(Adapter)
역할           "이걸로 나를 사용해"        │  "이걸로 나를 도와줘"
예시           PlaceOrderUseCase         │  OrderRepository
               FindOrderQuery           │  PaymentPort
               CancelOrderUseCase       │  EventPublisher
구현자         PlaceOrderService         │  JpaOrderRepository
                                        │  KakaoPayAdapter
                                        │  InMemoryOrderRepository (테스트)
호출자         OrderController           │  PlaceOrderService
               OrderCLICommand          │
               PlaceOrderServiceTest    │

공통점: 둘 다 인터페이스 (계약, Contract)
차이점: 누가 구현하고 누가 호출하는가의 방향이 반대
```

### 4. Input Boundary / Output Boundary (Clean Architecture와의 연결)

```
Uncle Bob(Clean Architecture)의 용어와 연결:

Driving Port = Input Boundary
  "UseCase의 입력 경계"
  외부에서 UseCase로 들어오는 데이터와 호출 방법의 계약

Driven Port = Output Boundary / Gateway
  "UseCase의 출력 경계" 또는 "게이트웨이"
  UseCase에서 외부(DB, API)로 나가는 데이터와 호출 방법의 계약

// Input Boundary (Driving Port)
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command); // Input
}

// Output Boundary / Gateway (Driven Port)
public interface OrderRepository {
    void save(Order order); // Output (저장 요청)
}

// Request Model (Input Boundary와 함께)
public record PlaceOrderCommand(
    UserId userId,
    List<OrderLineCommand> lines
) {} // HTTP DTO와 분리된 Application Input

// Response Model (Output Boundary와 함께)
public record OrderCreatedResult(
    OrderId orderId,
    Money totalAmount
) {} // HTTP Response DTO와 분리된 Application Output
```

### 5. 여러 Driving Port / Driven Port 설계

```
하나의 애플리케이션에 여러 Port 존재:

Driving Ports (Use Cases):
  PlaceOrderUseCase      → 주문 생성
  CancelOrderUseCase     → 주문 취소
  FindOrderQuery         → 주문 조회 (Query는 별도 분리)
  FindOrdersByUserQuery  → 사용자별 주문 목록

Driven Ports (Infrastructure 계약):
  OrderSavePort          → 주문 저장
  OrderQueryPort         → 주문 조회
  PaymentPort            → 결제 처리
  NotificationPort       → 알림 발송
  OrderEventPublisher    → 도메인 이벤트 발행

ISP(인터페이스 분리 원칙) 적용:
  거대한 OrderRepository 대신 역할별 분리
  PlaceOrderService: OrderSavePort만 의존 (조회 메서드 불필요)
  FindOrderService:  OrderQueryPort만 의존 (저장 메서드 불필요)
  → 각 UseCase는 필요한 Port만 의존
```

---

## 💻 실전 코드 — Driving/Driven Port 전체 구현

```java
// === Driving Ports (Application Core) ===

public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

public interface CancelOrderUseCase {
    void cancelOrder(CancelOrderCommand command);
}

// Query는 별도 인터페이스 (CQRS 스타일)
public interface FindOrderQuery {
    OrderDetail findById(OrderId id);
    Page<OrderSummary> findByUser(UserId userId, Pageable pageable);
}

// === Driven Ports (Domain Layer) ===

public interface OrderSavePort {       // PlaceOrderService용
    void save(Order order);
}

public interface OrderQueryPort {      // FindOrderService용
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
}

public interface PaymentPort {
    PaymentResult charge(PaymentRequest request);
    void refund(String transactionId, Money amount);
}

public interface OrderNotificationPort {
    void notifyOrderPlaced(Order order);
    void notifyOrderCancelled(Order order);
}

// === Application Core 구현 ===

@Service
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderSavePort orderSavePort;          // Driven Port만
    private final PaymentPort paymentPort;               // Driven Port만
    private final OrderNotificationPort notificationPort; // Driven Port만

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.getUserId(), command.getLines());
        order.place();
        PaymentResult payment = paymentPort.charge(PaymentRequest.of(order));
        order.confirmPayment(payment.getTransactionId());
        orderSavePort.save(order);
        notificationPort.notifyOrderPlaced(order);
        return order.getId();
    }
}

// === Driving Adapters ===

@RestController
public class OrderController {
    private final PlaceOrderUseCase placeOrderUseCase; // Driving Port 호출
    private final FindOrderQuery findOrderQuery;         // Query Port 호출

    @PostMapping("/api/orders")
    public ResponseEntity<PlaceOrderResponse> placeOrder(
        @RequestBody PlaceOrderRequest request
    ) {
        OrderId id = placeOrderUseCase.placeOrder(PlaceOrderCommand.from(request));
        return ResponseEntity.created(URI.create("/api/orders/" + id.value()))
            .body(PlaceOrderResponse.of(id));
    }

    @GetMapping("/api/orders/{id}")
    public OrderDetailResponse findOrder(@PathVariable String id) {
        return OrderDetailResponse.from(findOrderQuery.findById(OrderId.of(id)));
    }
}

// === Driven Adapters ===

@Repository
public class JpaOrderRepository implements OrderSavePort, OrderQueryPort {

    private final SpringDataJpaOrderRepository jpa;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        jpa.save(mapper.toEntity(order));
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value()).map(mapper::toDomain);
    }

    @Override
    public List<Order> findByUserId(UserId userId) {
        return jpa.findByUserId(userId.value()).stream()
            .map(mapper::toDomain).toList();
    }
}

@Component
public class KakaoPayAdapter implements PaymentPort {
    private final KakaoPayClient client;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        KakaoResponse res = client.charge(...);
        return PaymentResult.success(res.getTxId());
    }

    @Override
    public void refund(String transactionId, Money amount) {
        client.cancel(transactionId, amount.getValue());
    }
}
```

---

## 📊 패턴 비교

```
Driving Port / Driven Port 위치와 의존성 방향:

                    Application Core
     ┌──────────────────────────────────────────┐
     │                                          │
     │  [PlaceOrderUseCase] ← [PlaceOrderService] → [OrderSavePort]
     │  (Driving Port)        implements          (Driven Port)
     │                                          │
     └──────────────────────────────────────────┘
              ↑ calls                                    ↑ implements
     [OrderController]                       [JpaOrderRepository]
     (Driving Adapter)                       (Driven Adapter)

방향 정리:
  Driving: 외부 → Core (Controller → UseCase 인터페이스 → Service)
  Driven:  Core → 외부 (Service → Port 인터페이스 ← Repository)
```

---

## ⚖️ 트레이드오프

```
UseCase 인터페이스(Driving Port)를 항상 만들 필요가 있는가?

Driving Port(UseCase 인터페이스) 없는 경우:
  Controller → PlaceOrderService (구체 클래스 직접 의존)
  장점: 파일 수 감소
  단점: Controller 테스트 시 PlaceOrderService Mock 복잡

Driving Port(UseCase 인터페이스) 있는 경우:
  Controller → PlaceOrderUseCase (인터페이스)
  장점: Controller 단독 테스트 가능 (@WebMvcTest + @MockBean 간단)
  단점: 파일 수 증가

실용적 결론:
  Driven Port(Repository, PaymentPort): 항상 필요 (DIP 필수)
  Driving Port(UseCase): 팀 선택 (Controller 테스트 편의 여부)
  최소한 Driven Port는 반드시 Domain Layer에 위치해야 함
```

---

## 📌 핵심 정리

```
Driving vs Driven Port:

Driving Port (UseCase 인터페이스):
  위치: Application Core
  방향: 외부 Adapter → Core (Controller가 호출)
  구현: Application Core (PlaceOrderService)
  의미: "이 방법으로 애플리케이션을 사용하세요"

Driven Port (Repository, PaymentPort 인터페이스):
  위치: Domain/Application Layer (Core 안에)
  방향: Core → 외부 Adapter (Service가 사용)
  구현: Infrastructure Adapter (JpaOrderRepository)
  의미: "이 계약대로 나를 도와주세요"

핵심 규칙:
  Driven Port는 반드시 Domain Layer에 위치
  = Infrastructure가 Domain을 의존 (DIP)
  = Domain이 Infrastructure를 모름

헷갈릴 때 질문:
  "이 인터페이스를 구현하는 쪽이 Infrastructure인가?" → Driven Port
  "이 인터페이스를 호출하는 쪽이 Infrastructure(Controller)인가?" → Driving Port
```

---

## 🤔 생각해볼 문제

**Q1.** Driven Port인 `OrderRepository`를 Infrastructure Layer에 두면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**의존성 방향이 역전되지 않아 DIP 효과가 사라집니다.**

```
잘못된 구조:
  PlaceOrderService (domain)
       ↓ depends on
  OrderRepository (infrastructure) ← 잘못된 위치
       ↑ implements
  JpaOrderRepository (infrastructure)

PlaceOrderService가 infrastructure 패키지를 의존
→ 인프라 패키지 변경이 도메인에 영향
→ 의존성이 여전히 인프라 방향 (DIP 미적용)

올바른 구조:
  PlaceOrderService (domain)
       ↓ depends on
  OrderRepository (domain) ← 올바른 위치
       ↑ implements
  JpaOrderRepository (infrastructure)

인프라가 domain 패키지를 의존
→ 도메인은 인프라를 모름 (DIP 적용!)
```

</details>

---

**Q2.** Driving Port(UseCase)가 없이 Controller가 Service 구체 클래스를 직접 의존하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

실용적으로는 Driven Port보다 덜 심각하지만 두 가지 문제가 있습니다.

1. **Controller 단위 테스트가 복잡해짐**: `@WebMvcTest`에서 `@MockBean PlaceOrderService` 대신 구체 클래스를 Mock해야 하는데, 구체 클래스는 의존성이 많아 설정이 복잡.

2. **Application Core의 경계가 모호해짐**: UseCase 인터페이스가 없으면 "이 Service가 어떤 기능을 제공하는가"를 코드에서 명시적으로 알기 어려움. 메서드 수가 많아지면 어떤 게 진입점인지 모호.

현실적으로 작은 프로젝트에서는 Driving Port 없이 Service 직접 의존도 허용됩니다. 단, Driven Port(Repository, PaymentPort)는 항상 도메인에 위치해야 합니다.

</details>

---

**Q3.** `PlaceOrderService`가 `OrderSavePort`와 `OrderQueryPort`를 분리해서 의존하는 것(ISP 적용)과 `OrderRepository` 하나에 모두 담는 것은 어떤 실질적 차이를 만드는가?

<details>
<summary>해설 보기</summary>

**ISP 적용이 테스트를 더 단순하게 만들고 의존성을 명확히 합니다.**

```java
// ISP 미적용: PlaceOrderService가 조회 메서드도 알아야 함
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
    Page<Order> findAll(Pageable p);
    // ... 10개 메서드
}

// PlaceOrderService 테스트: 10개 메서드 중 save만 필요해도
// InMemory나 Mock에 10개 모두 구현 or 설정 필요

// ISP 적용: PlaceOrderService가 save만 필요
public interface OrderSavePort {
    void save(Order order);
}

// PlaceOrderService 테스트: save만 구현하면 됨
OrderSavePort savePort = order -> savedOrders.add(order); // 람다 한 줄
```

실질적 차이:
- 테스트 코드 단순화 (필요한 메서드만 구현)
- "PlaceOrderService는 주문 저장만 하고 조회는 하지 않는다"를 코드로 명시
- `OrderRepository`가 바뀌어도 `PlaceOrderService`는 `OrderSavePort`만 지키면 됨

단, 작은 프로젝트에서는 ISP보다 단일 인터페이스가 단순해서 더 나을 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Hexagonal 핵심 아이디어](./01-hexagonal-core-idea.md)** | **[홈으로 🏠](../README.md)** | **[다음: 어댑터의 역할 ➡️](./03-adapters-explained.md)**

</div>
