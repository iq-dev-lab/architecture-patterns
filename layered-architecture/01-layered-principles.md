# 레이어드 아키텍처의 원칙 — Presentation / Business / Data Access

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Presentation / Business / Data Access 각 레이어의 책임과 경계는 무엇인가?
- 레이어 간 의존성이 항상 아래 방향이어야 하는 이유는 무엇인가?
- 상위 레이어가 하위 레이어의 구체 클래스에 의존하면 안 되는 이유는 무엇인가?
- 레이어 경계를 지키지 않으면 실무에서 어떤 일이 발생하는가?
- 4-레이어(Presentation / Application / Domain / Infrastructure)는 3-레이어와 어떻게 다른가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

레이어드 아키텍처는 Spring Boot 프로젝트에서 가장 많이 쓰이는 구조다. 하지만 "Controller-Service-Repository를 나눴다"와 "레이어드 아키텍처를 올바르게 적용했다"는 완전히 다르다.

레이어의 본질은 **변경 이유(Reason to Change)에 따른 분리**다. 각 레이어가 왜 존재하는지, 어떤 경계를 가져야 하는지 이해하지 못하면 레이어를 나눠도 Fat Service와 의존성 혼재가 발생한다.

---

## 😱 흔한 실수 (Before — 레이어를 "폴더 분류"로만 이해할 때)

```
전형적인 Spring Boot 프로젝트 구조 (형태만 레이어드):

controller/
  OrderController.java    ← HTTP 처리 + 일부 비즈니스 로직
service/
  OrderService.java       ← 비즈니스 로직 + DB 쿼리 + 외부 API 호출
repository/
  OrderRepository.java    ← JpaRepository 상속 (Spring Data)
entity/
  Order.java              ← @Entity + 비즈니스 메서드 혼재

레이어 경계 위반 사례:
  ① Controller가 Repository를 직접 호출
     @RestController
     public class OrderController {
         @Autowired
         OrderRepository repository; // ← Service 건너뜀!
         // "조회는 간단하니까 직접 쓰자"
     }

  ② Service가 HttpServletRequest를 파라미터로 받음
     public OrderResult placeOrder(HttpServletRequest request) {
         String token = request.getHeader("Authorization"); // ← HTTP 개념이 Business 레이어에!
     }

  ③ Entity에 비즈니스 로직이 없고, Service에 모두 집중
     // Order.java → 필드와 getter/setter만 (Anemic Domain Model)
     // OrderService.java → 1,000줄 (Fat Service)

결과:
  레이어가 있지만 실질적인 분리 효과가 없음
  어느 파일에 코드를 넣어야 할지 팀 내 기준이 없음
  Controller 수정 → Service 영향 → Repository 영향 (전파 무제한)
```

---

## ✨ 올바른 접근 (After — 각 레이어의 책임과 경계를 명확히 할 때)

```
올바른 레이어드 아키텍처:

각 레이어의 유일한 변경 이유:
  Presentation Layer:
    변경 이유: API 스펙 변경, HTTP 프로토콜 변경, 인증 방식 변경
    알아야 하는 것: HTTP 요청/응답, DTO, 인증/인가

  Business (Application/Domain) Layer:
    변경 이유: 비즈니스 규칙 변경
    알아야 하는 것: 비즈니스 로직만 (HTTP도, JPA도 모름)

  Data Access (Infrastructure) Layer:
    변경 이유: DB 변경, 외부 API 변경
    알아야 하는 것: DB 접근, 외부 API 연동

경계 규칙:
  ① 상위 레이어 → 하위 레이어 의존 (단방향)
  ② 하위 레이어는 상위 레이어를 모름
  ③ 레이어를 건너뛰는 직접 호출 금지 (Strict Layer 기준)
  ④ HTTP 관련 클래스(HttpServletRequest 등)는 Presentation에만 존재
```

---

## 🔬 내부 원리 — 레이어드 아키텍처의 구조와 원칙

### 1. 3-레이어 vs 4-레이어 구조

```
전통적 3-레이어 (N-tier):
  ┌──────────────────────────────┐
  │   Presentation Layer         │ ← HTTP Controller, View
  ├──────────────────────────────┤
  │   Business Logic Layer       │ ← Service (비즈니스 + 도메인)
  ├──────────────────────────────┤
  │   Data Access Layer          │ ← Repository, DAO
  └──────────────────────────────┘

문제: Business Layer가 너무 많은 것을 담당
  - 비즈니스 규칙 (도메인 로직)
  - 유스케이스 조율 (여러 도메인 객체 협력)
  → Business Layer가 비대해짐

현대적 4-레이어 (DDD 기반):
  ┌──────────────────────────────┐
  │   Presentation Layer         │ ← Controller, View, API
  ├──────────────────────────────┤
  │   Application Layer          │ ← UseCase, 비즈니스 흐름 조율
  ├──────────────────────────────┤
  │   Domain Layer               │ ← Entity, Domain Service, VO
  ├──────────────────────────────┤
  │   Infrastructure Layer       │ ← Repository, External API, DB
  └──────────────────────────────┘

차이:
  Business Layer를 Application + Domain으로 분리
  Application: 유스케이스 흐름 조율 (도메인 객체들의 협력 조율)
  Domain: 순수 비즈니스 규칙 (Order, User, Money 등 도메인 객체)
```

### 2. 각 레이어의 책임과 경계

```
Presentation Layer:
  책임:
    - HTTP 요청 파싱 (URL, Header, Body → DTO)
    - 인증/인가 처리 (JWT 검증 등)
    - 입력 데이터 기본 유효성 검증 (형식 검증)
    - UseCase/Service 호출
    - 결과를 HTTP 응답으로 변환

  알면 안 되는 것:
    - 비즈니스 규칙 ("주문 금액이 1만원 이상이어야 한다")
    - DB 접근 방법
    - 외부 API 호출 방법

  포함되는 클래스:
    @RestController, @Controller
    Request/Response DTO
    @ControllerAdvice (예외 처리)
    Filter, Interceptor (인증/인가)

Application Layer:
  책임:
    - 유스케이스 흐름 조율 ("주문을 생성하려면 재고 확인 → 결제 → 저장 순서")
    - 트랜잭션 경계 정의
    - 도메인 객체들의 협력 조율
    - 도메인 이벤트 발행

  알면 안 되는 것:
    - HTTP 요청 형식 (HttpServletRequest 절대 금지)
    - DB 접근 구체 방법 (JPA 구체 클래스)
    - 외부 API 구체 클라이언트

  포함되는 클래스:
    @Service (UseCase 구현)
    Command/Query 객체 (Input)
    Result 객체 (Output)

Domain Layer:
  책임:
    - 비즈니스 규칙 구현 (순수 Java)
    - Entity, Value Object, Domain Service
    - Repository 인터페이스 정의 (Port)

  알면 안 되는 것:
    - Spring Framework (@Autowired, @Transactional 등)
    - JPA (@Entity 어노테이션 가능하나 이상적으론 분리)
    - HTTP
    - Kafka, Redis 등 인프라

  포함되는 클래스:
    도메인 Entity (Order, User, Product)
    Value Object (Money, Address, OrderId)
    Domain Service (두 Aggregate가 협력해야 하는 로직)
    Repository 인터페이스

Infrastructure Layer:
  책임:
    - DB 접근 구현 (JPA, JDBC, MongoDB)
    - 외부 API 클라이언트 구현
    - 메시지 발행/구독 구현
    - 캐시 구현

  알면 안 되는 것:
    - 비즈니스 규칙
    - HTTP 요청 처리

  포함되는 클래스:
    @Repository (JPA Repository 구현체)
    JpaEntity + Mapper
    외부 API Client (RestTemplate, WebClient)
    Kafka Producer/Consumer
```

### 3. 의존성 방향 원칙

```
올바른 의존 방향:
  Presentation → Application → Domain ← Infrastructure

도식:
  ┌─────────────┐
  │ Presentation│ ──depends on──→
  └─────────────┘
                  ┌─────────────┐
                  │ Application │ ──depends on──→
                  └─────────────┘
                                  ┌─────────────┐
                                  │   Domain    │ ←──implements──
                                  └─────────────┘
                                                  ┌──────────────┐
                                                  │Infrastructure│
                                                  └──────────────┘

핵심: Infrastructure가 Domain을 의존 (구현)
      Domain은 Infrastructure를 모름
      = DIP 적용

잘못된 의존 방향:
  Domain → Infrastructure (도메인이 JPA를 직접 의존)
  Presentation → Domain 건너뛰고 Infrastructure 직접 접근
  Infrastructure → Application (하위가 상위를 앎)

레이어 건너뛰기 (Layer Skipping):
  Controller → Repository 직접 (Service 건너뜀)
  → Strict Layer에서는 금지
  → Relaxed Layer에서는 허용 (그러나 남용 위험)
```

### 4. 레이어 경계를 침범하는 대표적 패턴들

```
침범 패턴 1: HTTP 개념이 Business Layer로 유입

  // ❌ Service에 HttpServletRequest 파라미터
  @Service
  public class OrderService {
      public OrderResult placeOrder(HttpServletRequest request) {
          String token = request.getHeader("Authorization");
          // HTTP 개념이 Business Layer에 침투
          // → OrderService 테스트에 MockHttpServletRequest 필요
      }
  }

  올바른 방법:
  // ✅ Controller가 Command로 변환
  @RestController
  public class OrderController {
      public ResponseEntity<?> placeOrder(
          @RequestBody PlaceOrderRequest request,
          @AuthenticationPrincipal UserPrincipal principal // Spring Security가 처리
      ) {
          PlaceOrderCommand command = PlaceOrderCommand.of(request, principal.getUserId());
          return ResponseEntity.ok(placeOrderUseCase.placeOrder(command));
      }
  }

침범 패턴 2: 비즈니스 로직이 Controller에 있음

  // ❌
  @RestController
  public class OrderController {
      public ResponseEntity<?> placeOrder(@RequestBody OrderRequest request) {
          if (request.getTotalPrice().compareTo(BigDecimal.valueOf(1000)) < 0) {
              return ResponseEntity.badRequest().body("최소 주문 금액 미달");
              // 비즈니스 규칙이 Controller에!
          }
          // ...
      }
  }

침범 패턴 3: Repository가 비즈니스 로직을 포함

  // ❌
  @Repository
  public class OrderRepository {
      public List<Order> findValidOrders() {
          // "유효한 주문"의 정의가 Repository에 있음
          return jpa.findByStatusNotAndCreatedAtAfter(
              OrderStatus.CANCELLED,
              LocalDateTime.now().minusDays(30)
          );
          // "유효한 주문"의 기준이 바뀌면 Repository를 수정해야 함
      }
  }
```

### 5. DTO의 레이어 이동 원칙

```
DTO는 레이어 경계에서 변환되어야 한다:

올바른 DTO 흐름:

  [Client] ──HTTP Body──→ [Controller]
                               │ PlaceOrderRequest (Presentation DTO)
                               │ → Command 변환
                               ↓
                         [Application]
                               │ PlaceOrderCommand (Application Input)
                               │ → 도메인 객체 생성
                               ↓
                           [Domain]
                               │ Order (도메인 객체, DTO 아님)
                               ↓
                         [Application]
                               │ OrderResult (Application Output)
                               ↓
                         [Controller]
                               │ → OrderResponse 변환
                               ↓
  [Client] ←──HTTP Body── [Controller]

잘못된 DTO 흐름 (DTO 침투):

  [Controller]
      │ PlaceOrderRequest (Controller DTO)
      │ 그대로 Service까지 전달
      ↓
  [Service]
      │ PlaceOrderRequest 그대로 사용
      ↓
  [Repository]
      │ 심한 경우 Repository까지 전달

문제:
  Presentation DTO가 Domain까지 내려오면
  → HTTP 요청 스펙 변경 시 Service, Repository까지 수정
  → 레이어 분리 효과 없음
```

---

## 💻 실전 코드 — 올바른 레이어드 아키텍처 구현

```java
// === Presentation Layer ===

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final PlaceOrderUseCase placeOrderUseCase; // Application Layer만 의존

    @PostMapping
    public ResponseEntity<PlaceOrderResponse> placeOrder(
        @Valid @RequestBody PlaceOrderRequest request,
        @AuthenticationPrincipal UserPrincipal principal
    ) {
        // Presentation 책임: HTTP → Command 변환
        PlaceOrderCommand command = PlaceOrderCommand.builder()
            .userId(UserId.of(principal.getUserId()))
            .items(request.getItems().stream()
                .map(item -> OrderItemCommand.of(item.getItemId(), item.getQuantity()))
                .toList())
            .build();

        // Application Layer에 위임
        OrderId orderId = placeOrderUseCase.placeOrder(command);

        // Application 결과 → HTTP 응답 변환
        return ResponseEntity
            .created(URI.create("/api/orders/" + orderId.value()))
            .body(PlaceOrderResponse.of(orderId));
    }
}

// === Application Layer ===

public interface PlaceOrderUseCase { // Input Port (Driving Port)
    OrderId placeOrder(PlaceOrderCommand command);
}

@Service
@Transactional
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderRepository orderRepository;    // Domain Port
    private final PaymentPort paymentPort;            // Domain Port
    private final OrderEventPublisher eventPublisher; // Domain Port

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        // Application 책임: 유스케이스 흐름 조율
        User user = userQueryPort.findById(command.getUserId());

        Order order = Order.create(
            command.getUserId(),
            command.getItems().stream()
                .map(item -> OrderLine.of(item.getItemId(), item.getQuantity()))
                .toList()
        );

        order.validate(); // Domain 로직에 위임

        PaymentResult payment = paymentPort.charge(
            PaymentRequest.of(order, user.getPaymentMethod())
        );

        order.confirmPayment(payment.getTransactionId());
        orderRepository.save(order);
        eventPublisher.publish(new OrderPlacedEvent(order));

        return order.getId();
    }
}

// === Domain Layer ===

public class Order { // 순수 Java, 어노테이션 없음

    private final OrderId id;
    private final UserId userId;
    private final List<OrderLine> lines;
    private OrderStatus status;
    private String paymentTransactionId;

    // 정적 팩토리 메서드: 불완전한 Order 생성 불가
    public static Order create(UserId userId, List<OrderLine> lines) {
        Objects.requireNonNull(userId, "userId는 필수입니다");
        if (lines == null || lines.isEmpty()) {
            throw new IllegalArgumentException("주문 항목이 없습니다");
        }
        return new Order(OrderId.newId(), userId, lines, OrderStatus.DRAFT);
    }

    // 비즈니스 규칙: Domain의 책임
    public void validate() {
        if (calculateTotal().isLessThan(Money.of(1_000))) {
            throw new MinOrderAmountException(Money.of(1_000));
        }
    }

    public void confirmPayment(String transactionId) {
        this.paymentTransactionId = transactionId;
        this.status = OrderStatus.PAYMENT_CONFIRMED;
    }

    public Money calculateTotal() {
        return lines.stream()
            .map(OrderLine::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Domain Port (인터페이스) — Domain Layer에 위치
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// === Infrastructure Layer ===

@Repository
public class JpaOrderRepository implements OrderRepository {

    private final SpringDataOrderJpaRepository jpa;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toJpaEntity(order);
        jpa.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value())
            .map(mapper::toDomain);
    }
}
```

---

## 📊 패턴 비교 — 3-레이어 vs 4-레이어 책임 비교

```
같은 "주문 생성" 기능에서의 책임 위치:

              3-레이어 (전통)          4-레이어 (DDD 기반)
─────────────┼───────────────────────┼──────────────────────────
비즈니스 규칙  │ OrderService          │ Order (Domain Entity)
(최소 금액 등) │ (Service에 모두)      │
─────────────┼───────────────────────┼──────────────────────────
유스케이스 조율│ OrderService          │ PlaceOrderService
(흐름 제어)   │ (Service에 모두)      │ (Application Layer)
─────────────┼───────────────────────┼──────────────────────────
DB 접근       │ OrderJpaRepository    │ JpaOrderRepository
              │ (Service가 직접 호출)  │ (OrderRepository 인터페이스 구현)
─────────────┼───────────────────────┼──────────────────────────
외부 API 접근  │ OrderService가        │ KakaoPayAdapter
              │ SDK 직접 호출         │ (PaymentPort 인터페이스 구현)
─────────────┼───────────────────────┼──────────────────────────
테스트 용이성  │ @SpringBootTest 필요   │ Domain 순수 Java 테스트 가능
              │ (모든 의존성 필요)     │ Application은 Port Mock으로
```

---

## ⚖️ 트레이드오프

```
레이어드 아키텍처의 본질적 한계:
  의존성이 여전히 DB 방향으로 흐름
  → Application Layer가 Infrastructure를 의존
  → 이것은 레이어드의 구조적 한계
  → 이 문제를 해결하려면 Hexagonal Architecture 필요

레이어드 아키텍처가 잘 작동하는 조건:
  - 각 레이어의 책임이 명확히 지켜질 때
  - 레이어 건너뛰기가 없을 때
  - 비즈니스 로직이 Domain Layer에 있을 때 (4-레이어 기준)

레이어드 아키텍처의 실패 조건:
  - Service가 모든 것을 담당할 때 (Fat Service)
  - HTTP 개념이 Business Layer로 유입될 때
  - Repository가 비즈니스 로직을 담을 때
```

---

## 📌 핵심 정리

```
레이어드 아키텍처 핵심 원칙:

3개 (또는 4개) 레이어, 각 레이어의 변경 이유가 다름:
  Presentation: API 스펙, HTTP 프로토콜
  Application:  비즈니스 흐름 (유스케이스)
  Domain:       비즈니스 규칙
  Infrastructure: DB, 외부 API

의존성 방향: 위에서 아래 (단방향)
  Presentation → Application → Domain ← Infrastructure

DTO는 레이어 경계에서 변환:
  HTTP Request → Command (Presentation 경계)
  Domain Object → Result (Application 경계)
  Result → HTTP Response (Presentation 경계)

레이어 경계 위반의 신호:
  Service 메서드 파라미터에 HttpServletRequest
  Controller가 Repository 직접 호출
  Entity에 비즈니스 로직이 없고 Service에 모두 집중
```

---

## 🤔 생각해볼 문제

**Q1.** 4-레이어 구조에서 Application Layer와 Domain Layer의 차이는 무엇인가? "비즈니스 로직"은 어느 레이어에 위치해야 하는가?

<details>
<summary>해설 보기</summary>

두 레이어의 핵심 차이는 **"누가 판단하는가"** 입니다.

**Domain Layer의 비즈니스 로직:**
- "주문 금액은 최소 1,000원 이상이어야 한다" → `Order.validate()`
- "취소는 배송 시작 전에만 가능하다" → `Order.cancel()`
- 도메인 전문가가 말하는 규칙들, 기술과 무관한 순수 비즈니스 규칙

**Application Layer의 로직:**
- "주문 생성 시 재고 확인 → 결제 → 저장 → 이벤트 발행 순서로"
- 여러 도메인 객체들의 협력을 조율
- 트랜잭션 경계 결정
- UseCase = "어떤 순서로 도메인을 조율하는가"

```java
// Domain: 순수 비즈니스 판단
public class Order {
    public void cancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new CannotCancelShippedOrderException();
        }
        this.status = OrderStatus.CANCELLED;
    }
}

// Application: 취소 흐름 조율
public class CancelOrderService {
    public void cancelOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(OrderNotFoundException::new);

        order.cancel(); // Domain에 판단 위임

        orderRepository.save(order);
        notificationPort.notifyCancellation(order); // 외부 알림
        eventPublisher.publish(new OrderCancelledEvent(order));
    }
}
```

</details>

---

**Q2.** `@Transactional`을 Application Layer에 붙여야 하는가, Domain Layer에 붙여야 하는가? 그 이유는?

<details>
<summary>해설 보기</summary>

**Application Layer의 UseCase 메서드**에 붙여야 합니다.

이유:
1. **트랜잭션 경계 = 유스케이스 경계**: "주문 생성"이라는 유스케이스가 하나의 트랜잭션으로 실행되어야 함. 이것은 Application Layer의 결정.

2. **Domain Layer는 Spring을 모름**: Domain Layer에 `@Transactional`을 붙이면 Spring에 의존하게 됨. Domain은 순수 Java여야 함.

3. **Infrastructure Layer의 트랜잭션**: `@Repository` 구현체에 붙이는 것은 개별 DB 접근에 대한 것이므로 UseCase 전체를 묶는 효과가 없음.

```java
// ✅ Application Layer
@Service
@Transactional  // UseCase 전체가 하나의 트랜잭션
public class PlaceOrderService implements PlaceOrderUseCase {
    public OrderId placeOrder(PlaceOrderCommand command) {
        // 여러 Repository 호출이 하나의 트랜잭션으로
    }
}

// ❌ Domain Layer — Spring 의존 발생
public class Order {
    @Transactional  // Domain이 Spring을 알아야 함
    public void place() { ... }
}
```

단, 외부 API 호출(결제, 이메일)은 트랜잭션 범위 밖에서 처리하거나 도메인 이벤트로 분리해야 트랜잭션 롤백 시 문제가 없습니다.

</details>

---

**Q3.** Spring Data JPA의 `JpaRepository`를 직접 상속한 인터페이스(`OrderJpaRepository extends JpaRepository<Order, Long>`)를 Service에서 `@Autowired`하는 것은 레이어드 아키텍처 원칙에서 허용되는가?

<details>
<summary>해설 보기</summary>

**형식적으로는 허용**되지만, **원칙적으로는 레이어 의존성 방향을 위반**합니다.

```java
// 이 구조:
@Service
public class OrderService {
    @Autowired
    OrderJpaRepository repository; // JpaRepository<Order, Long> 상속
}

// 의존성 방향:
// Service(Business) → JpaRepository(Infrastructure) ← Spring Data JPA
```

문제점:
- Service(Application/Business)가 Infrastructure(`JpaRepository`)에 직접 의존
- JPA를 MongoDB로 교체하면 Service 코드 수정 필요
- Service 단위 테스트에 JPA 컨텍스트 필요

올바른 방향:
```java
// Domain Port 인터페이스
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// Infrastructure: JpaRepository → Domain Port 구현
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpa; // 여기에만 JPA
}

// Service: Domain Port에만 의존
@Service
public class OrderService {
    private final OrderRepository orderRepository; // Infrastructure 몰라도 됨
}
```

레이어드 아키텍처에서도 이 분리를 적용하면 Hexagonal의 방향으로 자연스럽게 이행됩니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 아키텍처 선택 기준](../architecture-fundamentals/05-architecture-selection-criteria.md)** | **[홈으로 🏠](../README.md)** | **[다음: 엄격한 레이어 vs 느슨한 레이어 ➡️](./02-strict-vs-relaxed-layers.md)**

</div>
