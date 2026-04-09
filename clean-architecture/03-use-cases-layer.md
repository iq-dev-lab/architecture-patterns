# Use Cases 레이어 — 애플리케이션별 비즈니스 규칙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- UseCase가 애플리케이션 고유의 비즈니스 규칙을 담는 이유는?
- Input Boundary / Output Boundary가 인터페이스인 이유는?
- Request Model / Response Model이 외부 HTTP DTO와 분리되어야 하는 이유는?
- CQRS(Command Query Responsibility Segregation)와 Use Cases 레이어의 연결은?
- UseCase Interactor가 Entities와 Gateway(Repository)를 어떻게 조율하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Use Cases 레이어는 Entities(기업 핵심 규칙)와 Interface Adapters(기술 변환) 사이에서 "이 애플리케이션이 무엇을 하는가"를 정의한다. HTTP가 바뀌어도, DB가 바뀌어도, 이 레이어의 비즈니스 흐름은 그대로여야 한다.

Input/Output Boundary를 인터페이스로 정의하는 것이 DIP를 Use Cases 레이어에서 실현하는 방법이다. 이를 이해해야 "왜 Controller가 UseCase를 호출하면서도 UseCase가 Controller를 모를 수 있는가"가 명확해진다.

---

## 😱 흔한 실수 (Before — HTTP 개념이 Use Cases에 침투할 때)

```
Use Cases에 HTTP 형식이 침투:

@Service
public class PlaceOrderInteractor {

    // ❌ HTTP Request 객체가 Use Cases 레이어에
    public OrderId execute(HttpServletRequest httpRequest) {
        String userId = httpRequest.getHeader("X-User-Id");
        String body = IOUtils.toString(httpRequest.getInputStream());
        // HTTP를 파싱하는 로직이 UseCase에
    }

    // ❌ HTTP Response 객체가 Use Cases 레이어에
    public ResponseEntity<OrderResponse> execute(PlaceOrderCommand cmd) {
        Order order = ...;
        return ResponseEntity.created(URI.create("/orders/" + order.getId()))
            .body(new OrderResponse(order.getId().value())); // HTTP 형식이 UseCase에
    }
}

문제:
  "웹 → CLI로 채널 추가" 시 UseCase 수정 필요
  UseCase 테스트에 HttpServletRequest 생성 필요
  UseCase가 HTTP 결과 코드(201 Created)를 결정 → 책임 혼재
  "왜 UseCase가 URI를 알아야 하지?"
```

---

## ✨ 올바른 접근 (After — Use Cases는 Entities 조율, 형식 변환 없음)

```
UseCase 레이어의 순수한 역할:

PlaceOrderInteractor가 아는 것:
  Entities (Order, OrderLine, Money)
  Input Boundary (자신이 구현하는 인터페이스)
  Output Boundary (Presenter에게 위임하는 인터페이스)
  OrderGateway (DB 접근 추상화, Output Port)
  PaymentGateway (결제 추상화, Output Port)

PlaceOrderInteractor가 모르는 것:
  HTTP (HttpServletRequest, ResponseEntity)
  JPA (@Entity, @Transactional 세부사항)
  Kafka, Redis
  JSON 직렬화

결과:
  UseCase 테스트: 순수 Java, Spring 없음, ~0.01초
  HTTP → CLI → Batch: UseCase 코드 변경 없음
  JPA → MongoDB: Gateway 교체만, UseCase 변경 없음
```

---

## 🔬 내부 원리 — Use Cases 레이어 구조

### 1. Input Boundary — UseCase의 진입점 계약

```java
// Input Boundary: UseCase가 외부에 제공하는 계약
// Use Cases 레이어에 위치

public interface PlaceOrderInputBoundary {
    /**
     * 주문을 생성합니다.
     * @param requestModel HTTP, CLI, Batch 어디서 호출하든 동일한 입력 형식
     * @param outputBoundary 결과를 어떻게 표현할지는 호출자가 결정
     */
    void execute(PlaceOrderRequestModel requestModel,
                 PlaceOrderOutputBoundary outputBoundary);
}

// Request Model: HTTP DTO와 분리된 UseCase 전용 입력 형식
// Use Cases 레이어에 위치
public record PlaceOrderRequestModel(
    String userId,
    List<OrderLineRequestModel> items
    // HTTP @JsonProperty, @NotBlank 없음 → 순수 데이터 구조
) {
    // 필요하다면 기본적 유효성 검증
    public PlaceOrderRequestModel {
        Objects.requireNonNull(userId, "userId required");
        if (items == null || items.isEmpty())
            throw new IllegalArgumentException("items required");
    }
}

public record OrderLineRequestModel(String itemId, int quantity, long unitPrice) {}

// 왜 인터페이스인가?
// Controller, CLI, Test, Batch 중 어느 것이 UseCase를 구현하는 게 아님
// → UseCase(Interactor)가 Input Boundary를 구현
// → Controller가 Input Boundary를 호출 (의존 방향: Controller → Input Boundary)
// → UseCase Interactor가 교체되어도 Controller는 Input Boundary에만 의존하므로 변경 없음
```

### 2. Output Boundary — 결과를 외부로 전달하는 계약

```java
// Output Boundary: UseCase가 결과를 전달하는 계약
// Use Cases 레이어에 위치

public interface PlaceOrderOutputBoundary {
    void present(PlaceOrderResponseModel responseModel);
}

// Response Model: HTTP Response와 분리된 UseCase 전용 출력 형식
// Use Cases 레이어에 위치
public record PlaceOrderResponseModel(
    String orderId,
    String totalAmountFormatted  // UseCase가 계산한 값
    // HTTP Location 헤더 URL 없음 → HTTP 형식 침투 없음
) {}

// 왜 콜백(Output Boundary) 방식인가?
// UseCase가 결과를 직접 반환하면 → 반환 타입에 형식 결정이 포함됨
// Output Boundary를 통하면 → UseCase는 "무슨 데이터"만 결정
//                             → Presenter가 "어떻게 표현"을 결정

// 직접 반환 방식과 비교:
// 직접 반환: OrderId execute(request) → Controller가 변환
// Output Boundary: void execute(request, boundary) → Presenter가 변환
// 실용적: 대부분의 팀은 직접 반환을 선택 (단순성 우선)
```

### 3. UseCase Interactor — Entities와 Gateway 조율

```java
// UseCase Interactor: Input Boundary 구현
// Use Cases 레이어에 위치, Spring 어노테이션 최소화

public class PlaceOrderInteractor implements PlaceOrderInputBoundary {

    // Output Port (Gateway 인터페이스) — Use Cases 레이어에 위치
    private final OrderGateway orderGateway;
    private final PaymentGateway paymentGateway;
    private final NotificationGateway notificationGateway;

    public PlaceOrderInteractor(
        OrderGateway orderGateway,
        PaymentGateway paymentGateway,
        NotificationGateway notificationGateway
    ) {
        this.orderGateway = orderGateway;
        this.paymentGateway = paymentGateway;
        this.notificationGateway = notificationGateway;
    }

    @Override
    public void execute(
        PlaceOrderRequestModel requestModel,
        PlaceOrderOutputBoundary outputBoundary
    ) {
        // 1. Request Model → Entities 생성
        UserId userId = UserId.of(requestModel.userId());
        List<OrderLine> lines = requestModel.items().stream()
            .map(i -> OrderLine.of(
                ItemId.of(i.itemId()),
                i.quantity(),
                Money.of(i.unitPrice())
            ))
            .toList();

        // 2. Entities 비즈니스 로직 실행
        Order order = Order.create(userId, lines);
        order.place(); // Entities 레이어의 기업 수준 규칙 실행

        // 3. Gateway를 통해 결제 처리 (Output Port)
        PaymentRequestModel paymentRequest = new PaymentRequestModel(
            order.getId().value(),
            order.calculateTotal().getValue()
        );
        PaymentResponseModel paymentResult = paymentGateway.charge(paymentRequest);
        order.confirmPayment(paymentResult.transactionId());

        // 4. Gateway를 통해 저장
        orderGateway.save(order);

        // 5. Output Boundary를 통해 결과 전달
        PlaceOrderResponseModel responseModel = new PlaceOrderResponseModel(
            order.getId().value(),
            order.calculateTotal().toString()
        );
        outputBoundary.present(responseModel);

        // 6. 부가 알림 (비동기 가능)
        notificationGateway.notifyOrderPlaced(order.getId().value());
    }
}
```

### 4. Gateway 인터페이스 — Use Cases의 Output Port

```java
// Gateway = Output Port = Output Boundary (Repository/API)
// Use Cases 레이어에 위치 (인터페이스)

public interface OrderGateway {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentGateway {
    PaymentResponseModel charge(PaymentRequestModel request);
    void refund(String transactionId, long amount);
}

public interface NotificationGateway {
    void notifyOrderPlaced(String orderId);
}

// 왜 인터페이스가 Use Cases 레이어에 있어야 하는가?
// 인터페이스가 Interface Adapters에 있으면:
//   PlaceOrderInteractor(Use Cases) → OrderGateway(Interface Adapters) 의존
//   = Use Cases → Interface Adapters (의존성 규칙 위반!)
//
// 인터페이스가 Use Cases에 있으면:
//   PlaceOrderInteractor(Use Cases) → OrderGateway(Use Cases) 의존 (같은 레이어 내)
//   JpaOrderGateway(Interface Adapters) → OrderGateway(Use Cases) 구현 (안쪽 의존)
//   = 의존성 규칙 준수!
```

### 5. CQRS와 Use Cases 레이어의 연결

```
CQRS (Command Query Responsibility Segregation)의 Use Cases 적용:

Command Use Cases (상태 변경):
  PlaceOrderInputBoundary    → 주문 생성
  CancelOrderInputBoundary   → 주문 취소
  UpdateOrderInputBoundary   → 주문 수정
  
  특징:
    Output Boundary로 결과 전달
    트랜잭션 필요 (Application Service에 @Transactional)
    복잡한 비즈니스 흐름 (재고 확인 → 결제 → 저장 → 알림)

Query Use Cases (상태 조회):
  FindOrderQuery             → 주문 조회
  FindOrdersByUserQuery      → 사용자별 주문 목록

  특징:
    직접 반환 (단순한 조회 결과)
    트랜잭션 불필요 (읽기 전용)
    복잡한 흐름 없음 (조회만)
    Query 전용 Gateway 사용 가능 (도메인 모델 불필요한 경우)

Query UseCase는 단순하게:
  // Query UseCase는 Presenter 없이 직접 반환도 허용
  public interface FindOrderQuery {
      OrderDetail findById(OrderId id);
      Page<OrderSummary> findByUser(UserId userId, Pageable pageable);
  }

  // 복잡한 쿼리는 도메인 모델 없이 Query 전용 Repository 직접 사용도 허용
  public class FindOrderQueryHandler implements FindOrderQuery {
      private final OrderQueryRepository queryRepository; // 읽기 최적화
      public OrderDetail findById(OrderId id) {
          return queryRepository.findDetailById(id.value())
              .orElseThrow(OrderNotFoundException::new);
      }
  }
```

---

## 💻 실전 코드 — Use Cases 레이어 전체 예시

```java
// === Use Cases 레이어 전체 구조 ===

// Input Boundary
public interface PlaceOrderInputBoundary {
    void execute(PlaceOrderRequestModel request, PlaceOrderOutputBoundary output);
}

// Request/Response Model (외부 DTO와 분리)
public record PlaceOrderRequestModel(String userId, List<OrderLineRequestModel> items) {}
public record OrderLineRequestModel(String itemId, int quantity, long unitPrice) {}
public record PlaceOrderResponseModel(String orderId, String totalAmount) {}

// Output Boundary
public interface PlaceOrderOutputBoundary {
    void present(PlaceOrderResponseModel response);
}

// Gateway 인터페이스 (Output Port)
public interface OrderGateway {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentGateway {
    PaymentResponseModel charge(PaymentRequestModel request);
}

public record PaymentRequestModel(String orderId, long amount) {}
public record PaymentResponseModel(String transactionId, boolean success) {}

// UseCase Interactor (핵심 구현)
public class PlaceOrderInteractor implements PlaceOrderInputBoundary {

    private final OrderGateway orderGateway;
    private final PaymentGateway paymentGateway;

    @Override
    public void execute(PlaceOrderRequestModel request, PlaceOrderOutputBoundary output) {
        // Entities 생성 및 비즈니스 로직 실행
        Order order = Order.create(
            UserId.of(request.userId()),
            request.items().stream()
                .map(i -> OrderLine.of(ItemId.of(i.itemId()), i.quantity(), Money.of(i.unitPrice())))
                .toList()
        );
        order.place(); // Entities의 기업 수준 규칙

        // Gateway를 통한 외부 시스템 연동
        var payment = paymentGateway.charge(
            new PaymentRequestModel(order.getId().value(), order.calculateTotal().getValue())
        );
        if (!payment.success()) throw new PaymentFailedException(payment.transactionId());
        order.confirmPayment(payment.transactionId());

        // 저장
        orderGateway.save(order);

        // Output Boundary로 결과 전달
        output.present(new PlaceOrderResponseModel(
            order.getId().value(),
            order.calculateTotal().format()
        ));
    }
}

// === 단위 테스트 (Spring, JPA, Kafka 없음) ===
class PlaceOrderInteractorTest {

    private final List<Order> savedOrders = new ArrayList<>();
    private final OrderGateway orderGateway = new OrderGateway() {
        public void save(Order o) { savedOrders.add(o); }
        public Optional<Order> findById(OrderId id) { return Optional.empty(); }
    };
    private final PaymentGateway paymentGateway =
        req -> new PaymentResponseModel("tx-001", true);

    private PlaceOrderResponseModel capturedResponse;
    private final PlaceOrderOutputBoundary outputBoundary =
        response -> capturedResponse = response;

    private final PlaceOrderInteractor sut =
        new PlaceOrderInteractor(orderGateway, paymentGateway);

    @Test
    void 정상_주문_생성() {
        var request = new PlaceOrderRequestModel("user-1",
            List.of(new OrderLineRequestModel("item-1", 2, 5000)));

        sut.execute(request, outputBoundary);

        assertThat(savedOrders).hasSize(1);
        assertThat(capturedResponse.orderId()).isNotNull();
    }
}
```

---

## 📊 패턴 비교 — Use Cases 직접 반환 vs Output Boundary 방식

```
직접 반환 방식 (대부분의 팀, Hexagonal 스타일):
  public interface PlaceOrderUseCase {
      PlaceOrderResult execute(PlaceOrderCommand command);
  }
  장점: 직관적, 코드 흐름 이해 쉬움
  단점: 반환 타입에 표현 형식이 섞일 수 있음

Output Boundary 방식 (Clean Architecture 원본):
  public interface PlaceOrderInputBoundary {
      void execute(PlaceOrderRequestModel req, PlaceOrderOutputBoundary out);
  }
  장점: UseCase가 표현 형식을 완전히 모름, Presenter로 완전 분리
  단점: Presenter 상태 관리 복잡, 코드 흐름 추적 어려움

실용적 권장:
  Command UseCase: 직접 반환 (간단한 ID나 Result 반환)
  복잡한 View 조합: Output Boundary + Presenter 고려
```

---

## ⚖️ 트레이드오프

```
Output Boundary(Presenter) 패턴 비용:
  Presenter 클래스 추가
  Presenter 내부 상태 (Thread Safety 주의)
  코드 흐름: Controller → Interactor → Presenter → Controller (복잡)

직접 반환 방식의 단순성:
  반환값으로 결과 전달
  Controller가 직접 변환
  대부분의 REST API에서 충분

Query UseCase 특별 취급:
  Command와 Query를 분리 → Query는 단순 반환
  Query는 도메인 모델 없이 읽기 전용 Repository 직접 사용도 허용
  CQRS로 읽기/쓰기 최적화
```

---

## 📌 핵심 정리

```
Use Cases 레이어 핵심:

책임: 애플리케이션 고유 비즈니스 흐름 조율
위치: Entities(안쪽)와 Interface Adapters(바깥) 사이

포함 요소:
  Input Boundary (인터페이스) — UseCase 진입점 계약
  Request Model — HTTP DTO와 분리된 입력 형식
  UseCase Interactor — Entities 조율 + Gateway 사용
  Output Boundary (인터페이스) — 결과 전달 계약
  Response Model — HTTP DTO와 분리된 출력 형식
  Gateway 인터페이스 — DB/외부 API 추상화 (Output Port)

UseCase가 모르는 것:
  HTTP (어떤 채널에서 호출해도 동일)
  JPA (Gateway 인터페이스만 앎)
  Kafka, Redis (NotificationGateway 인터페이스만 앎)

CQRS 적용:
  Command UseCase: 복잡한 흐름, @Transactional, Output Boundary
  Query UseCase: 단순 반환, 읽기 최적화 Repository 직접 사용
```

---

## 🤔 생각해볼 문제

**Q1.** UseCase Interactor가 Gateway 인터페이스에만 의존하는데, `@Transactional`은 누가 담당해야 하는가?

<details>
<summary>해설 보기</summary>

**Use Cases 레이어의 Interactor에 `@Transactional`을 붙이는 것이 일반적입니다.**

트랜잭션 경계 = UseCase 경계입니다. "주문 생성"이라는 UseCase가 하나의 트랜잭션으로 실행되어야 합니다.

```java
// Spring 의존 허용 절충
@Service
@Transactional
public class PlaceOrderInteractor implements PlaceOrderInputBoundary {
    // UseCase = 트랜잭션 단위
}

// 완전 순수 추구 시 (Decorator 패턴)
// PlaceOrderInteractor: 순수 Java
// TransactionalPlaceOrderInteractor: PlaceOrderInteractor를 감싸며 @Transactional 추가
```

실용적으로는 `PlaceOrderInteractor`에 직접 `@Transactional`을 붙이는 것이 일반적이며, Spring 의존성이 생기지만 복잡도를 감수할 이유가 없습니다.

</details>

---

**Q2.** `PlaceOrderRequestModel`과 HTTP Controller의 `PlaceOrderRequest`를 분리하면 어떤 실제 이익이 생기는가? 같아 보이는데 왜 나눠야 하는가?

<details>
<summary>해설 보기</summary>

지금 당장은 같아 보여도 **두 가지 다른 변경 이유**를 가집니다.

`PlaceOrderRequest` (HTTP DTO) 변경 이유:
- API 클라이언트 요구로 필드명 변경 (`userId` → `customerId`)
- JSON 직렬화 형식 변경 (`@JsonProperty`)
- HTTP 검증 어노테이션 추가 (`@NotBlank`, `@Size`)

`PlaceOrderRequestModel` (UseCase Input) 변경 이유:
- UseCase가 새로운 데이터를 필요로 할 때 (비즈니스 이유)

분리하지 않으면: API 스펙 변경 → UseCase 코드도 수정 (두 레이어가 결합)
분리하면: API 스펙 변경 → HTTP DTO만 수정, UseCase는 변경 없음

실용적으로 처음에는 같을 수 있어도 분리된 구조를 유지하면 나중에 다른 채널(CLI, Batch)을 추가할 때 UseCase를 재사용할 수 있습니다.

</details>

---

**Q3.** UseCase가 두 개의 Gateway를 호출해야 하는데 첫 번째(결제)가 성공하고 두 번째(저장)가 실패했다면 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**트랜잭션과 보상 트랜잭션(Compensating Transaction)으로 처리합니다.**

단순한 경우 (@Transactional):
```java
@Transactional
public void execute(...) {
    paymentGateway.charge(request);  // 성공
    orderGateway.save(order);        // 실패 → 예외 발생
    // @Transactional이 롤백 → 하지만 결제는 이미 완료됨!
}
```

문제: 결제(외부 API)는 트랜잭션 롤백 불가

올바른 처리:
1. **Outbox Pattern**: 결제 요청을 DB에 저장 → 결제 완료 이벤트 구독 → 순서 보장
2. **Saga Pattern**: 결제 성공 후 저장 실패 → 결제 취소(환불) 보상 트랜잭션 실행
3. **DB 저장 먼저**: 저장 성공 → 결제 시도 (저장 실패 시 결제 안 함, 재시도)

```java
@Transactional
public void execute(...) {
    orderGateway.save(order);           // DB 저장 (롤백 가능)
    eventStore.save(new PaymentRequired(order)); // Outbox: 결제 요청 예약
    // 트랜잭션 커밋 후: Outbox 폴링 → 결제 처리 → 결제 완료 이벤트
}
```

</details>

---

<div align="center">

**[⬅️ 이전: Entities 레이어](./02-entities-layer.md)** | **[홈으로 🏠](../README.md)** | **[다음: Interface Adapters 레이어 ➡️](./04-interface-adapters-layer.md)**

</div>
