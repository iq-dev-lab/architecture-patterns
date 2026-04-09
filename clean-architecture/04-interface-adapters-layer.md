# Interface Adapters 레이어 — 변환의 레이어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Controller가 HTTP Request를 Use Case Input으로 변환하는 역할이란 정확히 무엇인가?
- Presenter가 Use Case Output을 ViewModel로 변환하는 역할은 어디서 필요한가?
- Gateway가 Repository 인터페이스를 구현하는 방식은 Hexagonal Driven Adapter와 어떻게 같은가?
- 이 레이어에서 Spring MVC(`@RestController`, `@Repository`)가 허용되는 이유는?
- Interface Adapters에서 비즈니스 로직이 있으면 안 되는 이유는?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Interface Adapters 레이어는 두 세계 — Use Cases의 순수 비즈니스 세계와 Frameworks의 기술 세계 — 를 연결하는 번역기다.

이 레이어에서 절대로 일어나면 안 되는 것은 **비즈니스 판단**이다. "주문 금액이 최소 금액 이상인가?"는 Entities가 판단한다. "HTTP 응답 코드가 201인가 400인가?"는 비즈니스 규칙이 아니라 표현 방식이다. Interface Adapters는 이 경계에서 형식만 변환한다.

---

## 😱 흔한 실수 (Before — Interface Adapters에 비즈니스 로직이 있을 때)

```
Controller에 비즈니스 판단:
  @RestController
  public class OrderController {
      public ResponseEntity<?> placeOrder(@RequestBody PlaceOrderRequest req) {
          // ❌ 비즈니스 규칙이 Controller에
          if (req.getTotalAmount() < 1000) {
              return ResponseEntity.badRequest().body("최소 1000원 이상");
          }
          if (req.getItems().isEmpty()) {
              return ResponseEntity.badRequest().body("항목 없음");
          }
          placeOrderInputBoundary.execute(mapToRequest(req), presenter);
          return ResponseEntity.created(...).body(presenter.getViewModel());
      }
  }
  문제: 최소 금액 규칙이 Controller에 → CLI 채널에서 재사용 불가

Gateway에 비즈니스 로직:
  @Repository
  public class JpaOrderGateway implements OrderGateway {
      public List<Order> findActiveOrders(String userId) {
          // ❌ "활성 주문"의 비즈니스 정의가 Gateway에
          return jpa.findByUserIdAndStatusIn(userId,
              List.of("PLACED", "PAYMENT_CONFIRMED", "SHIPPING"));
          // "활성 주문" 기준이 바뀌면 Gateway 수정 → Entities가 정의해야 할 규칙
      }
  }
```

---

## ✨ 올바른 접근 (After — Interface Adapters는 변환만)

```
올바른 Interface Adapters 책임:

Controller: HTTP Request → Request Model 변환만
  @RestController
  public class OrderController {
      public ResponseEntity<?> placeOrder(@RequestBody PlaceOrderRequest req) {
          // HTTP DTO → Request Model (변환만, 비즈니스 판단 없음)
          PlaceOrderRequestModel requestModel = mapper.toRequestModel(req, userId);
          // UseCase에 위임 (비즈니스 판단은 UseCase → Entities)
          placeOrderBoundary.execute(requestModel, presenter);
          return ResponseEntity.created(...).body(presenter.getViewModel());
      }
  }

Presenter: Response Model → ViewModel 변환만
  public class OrderPresenter implements PlaceOrderOutputBoundary {
      public void present(PlaceOrderResponseModel response) {
          // Response Model → ViewModel (변환만, 비즈니스 판단 없음)
          this.viewModel = new OrderViewModel(
              response.orderId(),
              "/api/orders/" + response.orderId(), // URI 형식
              response.totalAmount()
          );
      }
  }

Gateway: Domain Entity → JPA Entity 변환 + DB 접근
  public class JpaOrderGateway implements OrderGateway {
      public void save(Order order) {
          // Domain → JPA Entity 변환 + JPA save
          jpa.save(mapper.toJpaEntity(order));
      }
  }
```

---

## 🔬 내부 원리 — Interface Adapters 구성 요소

### 1. Controller — Driving Adapter (HTTP → UseCase)

```java
// Controller = Driving Adapter
// HTTP 세계와 Use Cases 세계를 연결하는 번역기

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    // Use Cases 레이어의 인터페이스에만 의존
    private final PlaceOrderInputBoundary placeOrderInputBoundary;
    private final FindOrderQuery findOrderQuery;
    private final OrderPresenter orderPresenter; // Presenter (Output Boundary 구현)

    @PostMapping
    public ResponseEntity<OrderViewModel> placeOrder(
        @Valid @RequestBody PlaceOrderRequest httpRequest,
        @AuthenticationPrincipal UserPrincipal principal
    ) {
        // 1. HTTP DTO → Request Model (형식 변환)
        PlaceOrderRequestModel requestModel = PlaceOrderRequestModel.builder()
            .userId(principal.getUserId())
            .items(httpRequest.items().stream()
                .map(i -> new OrderLineRequestModel(i.itemId(), i.quantity(), i.unitPrice()))
                .toList())
            .build();

        // 2. UseCase 실행 (비즈니스 로직은 UseCase → Entities에서)
        placeOrderInputBoundary.execute(requestModel, orderPresenter);

        // 3. Presenter가 만든 ViewModel → HTTP Response
        OrderViewModel viewModel = orderPresenter.getViewModel();
        return ResponseEntity
            .created(URI.create(viewModel.selfLink()))
            .body(viewModel);
    }

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDetailViewModel> getOrder(
        @PathVariable String orderId
    ) {
        // Query UseCase는 단순 반환 (Presenter 없이)
        OrderDetailModel detail = findOrderQuery.findById(OrderId.of(orderId));
        return ResponseEntity.ok(OrderDetailViewModel.from(detail));
    }
}
```

### 2. Presenter — Output Boundary 구현 (UseCase Output → ViewModel)

```java
// Presenter = Output Boundary 구현 (선택적 패턴)
// UseCase의 Response Model을 HTTP ViewModel로 변환

@Component
@Scope("request") // 요청 범위로 Thread Safety 해결
public class OrderPresenter implements PlaceOrderOutputBoundary {

    private OrderViewModel viewModel;

    @Override
    public void present(PlaceOrderResponseModel responseModel) {
        // Response Model → ViewModel (HTTP 표현 형식으로 변환)
        this.viewModel = new OrderViewModel(
            responseModel.orderId(),
            String.format("/api/orders/%s", responseModel.orderId()), // URI 형식
            responseModel.totalAmount(),
            LocalDateTime.now().format(DateTimeFormatter.ISO_DATE_TIME)
        );
    }

    public OrderViewModel getViewModel() {
        if (viewModel == null) throw new IllegalStateException("present() 먼저 호출 필요");
        return viewModel;
    }
}

// ViewModel: HTTP Response 형식 (Interface Adapters 레이어)
public record OrderViewModel(
    String orderId,
    @JsonProperty("_links") Map<String, String> links,
    String totalAmount,
    String createdAt
) {
    // 정적 팩토리 (Presenter가 사용)
    public static OrderViewModel of(String orderId, String selfLink, ...) { ... }
    public String selfLink() { return links.get("self"); }
}
```

### 3. Gateway — Driven Adapter (UseCase Output Port → DB/API)

```java
// Gateway = Driven Adapter
// Use Cases의 Gateway 인터페이스 구현 + DB/외부 API 연동

@Repository
@RequiredArgsConstructor
public class JpaOrderGateway implements OrderGateway {

    private final SpringDataOrderJpaRepository jpa; // Frameworks 레이어
    private final OrderGatewayMapper mapper;         // 매핑 책임

    @Override
    public void save(Order order) {
        // Domain Entity → JPA Entity 변환 (Interface Adapters 책임)
        OrderJpaEntity entity = mapper.toJpaEntity(order);
        jpa.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findByOrderId(id.value())
            .map(entity -> mapper.toDomain(entity)); // JPA Entity → Domain 변환
    }

    @Override
    public List<Order> findByUserId(UserId userId) {
        return jpa.findByUserId(userId.value()).stream()
            .map(mapper::toDomain)
            .toList();
    }
}

// 외부 API Gateway
@Component
@RequiredArgsConstructor
public class KakaoPayPaymentGateway implements PaymentGateway {

    private final KakaoPayHttpClient client; // Frameworks 레이어의 HTTP 클라이언트

    @Override
    public PaymentResponseModel charge(PaymentRequestModel request) {
        // Use Cases Request Model → 외부 API 요청 형식 변환
        KakaoPayChargeRequest apiRequest = KakaoPayChargeRequest.builder()
            .partnerOrderId(request.orderId())
            .totalAmount(request.amount())
            .build();

        // 외부 API 호출
        KakaoPayChargeResponse apiResponse = client.charge(apiRequest);

        // 외부 API 응답 → Use Cases Response Model 변환
        return new PaymentResponseModel(
            apiResponse.getTransactionId(),
            "SUCCESS".equals(apiResponse.getResultCode())
        );
    }
}
```

### 4. Spring MVC가 Interface Adapters에서 허용되는 이유

```
의존성 규칙 관점에서의 허용 이유:

Spring MVC (@RestController, @RequestMapping 등):
  위치: Interface Adapters 레이어
  이유: Controller = Interface Adapters의 일부
       Spring MVC는 HTTP 요청을 메서드에 매핑하는 프레임워크
       이 레이어가 HTTP 세계와 연결하는 것 자체가 역할
  
  허용되는 Spring 어노테이션:
    @RestController, @RequestMapping, @GetMapping, @PostMapping
    @RequestBody, @PathVariable, @Valid
    @AuthenticationPrincipal (Spring Security)

Spring Data JPA (@Repository):
  위치: Interface Adapters 레이어 (Gateway 구현)
  이유: Gateway가 JPA로 구현하는 것 자체가 역할
  
  허용되는 Spring/JPA 어노테이션:
    @Repository
    @Transactional (트랜잭션은 UseCase 또는 Gateway)

허용되지 않는 곳:
  Entities 레이어: @Entity, @Component, @JsonProperty → 금지
  Use Cases 레이어: @RestController, @Repository → 금지
                    @Transactional → 허용 (절충)

요약:
  "이 레이어가 본래 하는 일"에 필요한 프레임워크 어노테이션은 허용
  "이 레이어가 몰라야 할 것"에 관한 어노테이션은 금지
```

### 5. 매핑 책임 — Interface Adapters의 핵심 코드

```java
// 매핑은 Interface Adapters의 핵심 책임
// Mapper 클래스로 분리 (MapStruct 또는 수동)

@Component
public class OrderGatewayMapper {

    // Domain → JPA Entity
    public OrderJpaEntity toJpaEntity(Order domain) {
        OrderJpaEntity entity = new OrderJpaEntity();
        entity.setOrderId(domain.getId().value());
        entity.setUserId(domain.getUserId().value());
        entity.setStatus(domain.getStatus().name());
        entity.setPaymentTxId(domain.getPaymentTransactionId());
        entity.setLines(
            domain.getLines().stream()
                .map(line -> {
                    OrderLineJpaEntity lineEntity = new OrderLineJpaEntity();
                    lineEntity.setItemId(line.getItemId().value());
                    lineEntity.setQuantity(line.getQuantity());
                    lineEntity.setUnitPrice(line.getUnitPrice().getValue());
                    return lineEntity;
                })
                .toList()
        );
        return entity;
    }

    // JPA Entity → Domain (reconstitute 사용)
    public Order toDomain(OrderJpaEntity entity) {
        return Order.reconstitute(
            OrderId.of(entity.getOrderId()),
            UserId.of(entity.getUserId()),
            entity.getLines().stream()
                .map(l -> OrderLine.of(
                    ItemId.of(l.getItemId()),
                    l.getQuantity(),
                    Money.of(l.getUnitPrice())
                ))
                .toList(),
            OrderStatus.valueOf(entity.getStatus()),
            entity.getPaymentTxId()
        );
    }
}
```

---

## 📊 패턴 비교 — Interface Adapters 구성 요소와 Hexagonal Adapter 매핑

```
Clean Architecture Interface Adapters  │  Hexagonal Architecture Adapters
───────────────────────────────────────┼────────────────────────────────────
Controller                             │  Driving Adapter
Presenter                              │  (없음 — Hexagonal은 직접 반환)
Gateway (Repository 구현)              │  Driven Adapter
Order Mapper (Domain ↔ JPA)            │  Persistence Mapper
Request Model Mapper (HTTP ↔ UC)       │  Web Mapper

공통점:
  두 방식 모두 "변환"이 핵심 책임
  비즈니스 로직 없음
  Domain 객체를 보호하기 위한 외부 형식 변환

차이점:
  Presenter: Clean만 있음 (UseCase가 직접 반환 안 함)
  Gateway: Hexagonal의 Driven Adapter와 동일한 역할, 이름만 다름
```

---

## ⚖️ 트레이드오프

```
Presenter 패턴의 트레이드오프:
  UseCase가 표현 형식을 완전히 모름 → 더 순수한 분리
  Presenter 상태 관리 복잡 → @Scope("request") 또는 ThreadLocal
  코드 흐름 파악 어려움 → 직접 반환이 더 직관적

대안 (대부분의 팀 선택):
  UseCase가 Response Model 직접 반환
  Controller가 Response Model → ViewModel 변환 (Controller의 추가 책임)
  단순하지만 Controller에 변환 로직이 일부 들어감

Gateway 매핑 비용:
  Domain ↔ JPA Entity 매핑 코드 추가
  MapStruct로 자동화 가능
  분리하지 않으면 @Entity가 Domain에 침투 → Entities 레이어 오염
```

---

## 📌 핵심 정리

```
Interface Adapters 레이어 핵심:

역할: 내부(Use Cases, Entities) ↔ 외부(Frameworks) 형식 변환

구성 요소:
  Controller: HTTP Request → Request Model (Driving)
  Presenter:  Response Model → ViewModel (Output Boundary 구현)
  Gateway:    OrderGateway 구현 → JPA/외부 API (Driven)
  Mapper:     Domain ↔ JPA Entity, HTTP DTO ↔ Request Model

허용되는 것:
  Spring MVC 어노테이션 (@RestController, @Repository)
  JPA 접근 (Gateway 구현체에서)
  외부 API 호출 (Gateway 구현체에서)

금지되는 것:
  비즈니스 판단 (최소 금액 검증 등 → Entities로)
  Domain Model을 오염시키는 변환 (JPA Entity가 Domain까지 전달 → 금지)

Hexagonal과의 관계:
  Gateway = Driven Adapter (이름만 다름, 역할 동일)
  Controller = Driving Adapter (동일)
  Presenter = Hexagonal에 없는 개념 (선택적 적용)
```

---

## 🤔 생각해볼 문제

**Q1.** Controller가 HTTP Request를 Request Model로 변환할 때, 유효성 검증(`@Valid`, `@NotBlank`)은 어디서 해야 하는가?

<details>
<summary>해설 보기</summary>

**두 레이어에서 다른 종류의 검증을 합니다.**

Interface Adapters(Controller)의 검증: **형식 검증**
```java
public record PlaceOrderRequest(
    @NotBlank String userId,          // 빈 문자열 금지 (형식)
    @NotEmpty List<OrderItemDto> items // 빈 목록 금지 (형식)
) {}
// HTTP 입력이 올바른 형식인가? → Controller가 담당
```

Entities의 검증: **비즈니스 규칙 검증**
```java
public class Order {
    public static Order create(...) {
        if (lines.isEmpty()) throw new IllegalArgumentException("..."); // 비즈니스
    }
    public void place() {
        if (total.isLessThan(minAmount)) throw new MinOrderAmountException(); // 비즈니스
    }
}
// 비즈니스 규칙을 만족하는가? → Entities가 담당
```

겹치는 것처럼 보여도:
- Controller의 `@NotEmpty`는 "HTTP 요청에 items 필드가 있는가" (형식)
- Entities의 검증은 "주문 항목이 비즈니스 규칙을 만족하는가" (비즈니스)

</details>

---

**Q2.** Gateway와 Repository의 차이는 무엇인가? DDD의 Repository와 Clean Architecture의 Gateway는 같은 개념인가?

<details>
<summary>해설 보기</summary>

**역할은 같고 이름과 위치가 다릅니다.**

DDD Repository: Aggregate를 저장/조회하는 추상화 (인터페이스)
Clean Architecture Gateway: UseCase의 Output Port로서 외부 시스템 접근 추상화

```
DDD:
  OrderRepository (인터페이스, Domain Layer)
  JpaOrderRepository (구현체, Infrastructure Layer)

Clean Architecture:
  OrderGateway (인터페이스, Use Cases Layer)
  JpaOrderGateway (구현체, Interface Adapters Layer)
```

차이:
- 이름: Repository vs Gateway
- 위치: Domain Layer vs Use Cases Layer (인터페이스)
- 범위: Gateway는 DB뿐만 아니라 외부 API도 포함 (PaymentGateway, NotificationGateway)

본질은 같습니다: "도메인/비즈니스 코드가 인프라를 모르도록 추상화하는 인터페이스"

</details>

---

**Q3.** `@Scope("request")`로 Presenter의 Thread Safety 문제를 해결하면 성능 이슈가 생기지 않는가?

<details>
<summary>해설 보기</summary>

`@Scope("request")`는 요청마다 새 Presenter 인스턴스를 생성하므로 Thread Safety 문제를 해결하지만, Bean 생성 비용이 있습니다.

**대안들:**

1. **ThreadLocal 사용**: Singleton Presenter + ThreadLocal 내부 상태
```java
private final ThreadLocal<OrderViewModel> viewModelHolder = new ThreadLocal<>();
```

2. **메서드 반환값 사용 (가장 단순)**:
```java
// Presenter 없이 Controller가 직접 변환
PlaceOrderResponseModel result = useCase.execute(request); // 직접 반환
OrderViewModel viewModel = OrderViewModelMapper.from(result); // Controller 변환
return ResponseEntity.created(...).body(viewModel);
```
이 방식이 대부분의 팀에서 채택됩니다. Presenter 패턴의 복잡성 없이 동일한 분리 효과를 얻습니다.

실용적 결론: Presenter 패턴이 강제적으로 필요한 경우가 아니라면 직접 반환을 사용하세요.

</details>

---

<div align="center">

**[⬅️ 이전: Use Cases 레이어](./03-use-cases-layer.md)** | **[홈으로 🏠](../README.md)** | **[다음: Frameworks & Drivers 레이어 ➡️](./05-frameworks-and-drivers.md)**

</div>
