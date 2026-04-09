# 레이어드 아키텍처의 함정 — Fat Service와 DTO 침투

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- DTO 침투란 무엇이고, Presentation DTO가 Domain까지 내려갔을 때 어떤 문제가 생기는가?
- Fat Service는 어떤 과정을 거쳐 만들어지는가? 레이어드 구조의 어떤 특성이 이를 유발하는가?
- 레이어 간 순환 의존성은 어떻게 생기는가?
- `@Transactional` 남용이 Service를 비대하게 만드는 이유는 무엇인가?
- Anemic Domain Model이란 무엇이고 왜 문제인가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

레이어드 아키텍처를 "올바르게" 사용하더라도 시간이 지나면서 반드시 마주치는 함정들이 있다. DTO 침투와 Fat Service는 규칙을 위반해서 생기는 게 아니라, **레이어드 아키텍처의 구조적 특성** 때문에 자연스럽게 발생한다.

이 함정들을 이해해야 "단순히 레이어드 아키텍처를 더 잘 쓰는 것"으로 해결이 되는지, 아니면 Hexagonal Architecture로의 전환이 필요한지 판단할 수 있다.

---

## 😱 흔한 실수 (Before — 함정에 빠진 전형적인 코드)

```
프로젝트 출시 후 1년, 다음 증상이 나타남:

증상 1: DTO 침투
  PlaceOrderRequest (Controller DTO)가
  → OrderService.placeOrder(PlaceOrderRequest request)에 그대로 전달
  → OrderRepository.save(PlaceOrderRequest request)까지 내려옴
  
  결과:
    API 스펙(JSON 필드명) 변경 → Service, Repository 수정 필요
    PlaceOrderRequest에 HTTP 관련 어노테이션(@JsonProperty 등)이
    Domain 로직 실행 중에도 메모리에 올라와 있음

증상 2: Fat Service
  OrderService.java: 1,247줄
    placeOrder() - 89줄
    cancelOrder() - 67줄
    findOrdersByUser() - 45줄
    calculateShippingFee() - 112줄 ← 비즈니스 로직
    applyDiscountCoupon() - 98줄  ← 비즈니스 로직
    checkInventory() - 76줄      ← 비즈니스 로직
    sendOrderEmail() - 55줄      ← 인프라 로직
    publishOrderEvent() - 34줄   ← 인프라 로직
    updateUserPoint() - 48줄     ← 다른 도메인 로직
    ... (27개 메서드)

  문제: 주문, 할인, 포인트, 알림, 재고 — 모든 것이 OrderService에

증상 3: Anemic Domain Model
  Order.java:
    private Long id;
    private Long userId;
    private OrderStatus status;
    private BigDecimal totalPrice;
    // getter/setter만 있음
    // 비즈니스 로직 없음 → 모두 OrderService에

  결과: Order는 데이터 컨테이너, OrderService가 모든 것을 판단
```

---

## ✨ 올바른 접근 (After — 함정을 인식하고 책임을 분산할 때)

```
함정 해결 방향:

DTO 침투 해결:
  각 레이어 경계에서 변환 객체(Command, Result) 사용
  Controller DTO → Command (Application Input)
  Domain → Result (Application Output)
  Result → Response DTO (Controller Output)

Fat Service 해결:
  비즈니스 로직 → Domain 객체(Order, User)로 이동
  Service = 도메인 객체들의 협력 조율만
  외부 의존성 → Port/Adapter 또는 별도 Service로 분리

Anemic Domain Model 해결:
  도메인 객체에 비즈니스 메서드 추가
  Order.place() → 주문 가능 여부 판단은 Order가
  Order.cancel() → 취소 가능 여부 판단은 Order가
  Order.calculateTotal() → 금액 계산은 Order가
```

---

## 🔬 내부 원리 — 각 함정이 발생하는 메커니즘

### 1. DTO 침투 — 경계가 없는 객체의 이동

```
DTO 침투의 전파 경로:

시작: Controller에서 DTO를 그대로 Service로 전달
  @RestController
  public class OrderController {
      public ResponseEntity<?> placeOrder(@RequestBody PlaceOrderRequest request) {
          orderService.placeOrder(request); // ← DTO 그대로 전달
      }
  }

Service가 DTO를 받아서 Repository까지 전달:
  @Service
  public class OrderService {
      public void placeOrder(PlaceOrderRequest request) {  // ← Presentation DTO 수신
          Order order = new Order();
          order.setUserId(request.getUserId());
          order.setItems(request.getItems());
          orderRepository.save(order);
      }
  }

발생하는 문제들:
  ① API 스펙 변경 파급
    PlaceOrderRequest.userId → customerId 로 필드명 변경
    → Controller 수정 + Service 수정 (DTO를 직접 사용하는 모든 곳)
    → API 스펙 변경이 비즈니스 레이어까지 파급

  ② HTTP 개념이 Business Layer로 유입
    PlaceOrderRequest에 @JsonProperty, @NotNull 같은 HTTP/검증 어노테이션
    → 비즈니스 로직 실행 중에 HTTP 관련 객체가 존재
    → Service 테스트 시 HTTP DTO 객체 생성 필요

  ③ 테스트 불편
    OrderServiceTest:
      PlaceOrderRequest request = new PlaceOrderRequest();
      request.setUserId("user-1");
      request.setItems(List.of(...));
      // HTTP DTO를 Service 테스트에서 직접 생성 → 어색함

올바른 해결:
  PlaceOrderRequest (Controller DTO)
    → 변환 →
  PlaceOrderCommand (Application Input — HTTP 개념 없음)
    → Service 사용
```

### 2. Fat Service — 비즈니스 로직이 Service에 집중되는 구조적 이유

```
레이어드 아키텍처에서 Fat Service가 필연적인 이유:

레이어드 구조의 암묵적 규칙:
  "Controller는 HTTP 처리만"
  "Repository는 DB 접근만"
  → 그 외 모든 것 = Service의 책임

시간에 따른 Service 비대 과정:

  Sprint 1: 주문 생성 (30줄)
    public void placeOrder() {
        // 주문 생성 로직 30줄
    }

  Sprint 3: 할인 쿠폰 적용 (30줄 추가)
    public void placeOrder() {
        // 주문 생성 로직 30줄
        // 할인 쿠폰 적용 로직 30줄
    }

  Sprint 5: 재고 확인 추가 (20줄 추가)
    public void placeOrder() {
        // 주문 생성 로직 30줄
        // 할인 쿠폰 적용 로직 30줄
        // 재고 확인 로직 20줄
    }

  Sprint 8: 포인트 적립 추가 (25줄 추가)
    public void placeOrder() {
        // 주문 생성 로직 30줄
        // 할인 쿠폰 적용 로직 30줄
        // 재고 확인 로직 20줄
        // 포인트 적립 로직 25줄
    }
  
  → 1년 후: placeOrder() 단일 메서드 200줄

Fat Service의 증상:
  ① 메서드 길이: 단일 메서드 50줄 이상
  ② 의존성 수: 생성자/필드에 의존성 5개 이상
  ③ Private 메서드 수: 비즈니스 로직을 숨기는 private 메서드 다수
  ④ 클래스 크기: 500줄 이상

근본 원인: 비즈니스 로직이 Domain 객체가 아닌 Service에 있음
해결 방향:
  할인 계산 → DiscountPolicy.apply(order)
  재고 확인 → Inventory.check(items)
  포인트 적립 → Point.earn(user, amount)
  → Service는 이들을 조율하는 역할만
```

### 3. Anemic Domain Model — 빈약한 도메인 객체

```
Anemic Domain Model의 특징:

도메인 객체(Order.java):
  public class Order {
      private Long id;
      private Long userId;
      private List<OrderLine> items;
      private BigDecimal totalPrice;
      private String status;

      // getter만 있음
      public Long getId() { return id; }
      public Long getUserId() { return userId; }
      public BigDecimal getTotalPrice() { return totalPrice; }
      // ... (setter도 있음)

      // 비즈니스 메서드 없음!
  }

Service(OrderService.java):
  public void placeOrder(PlaceOrderCommand command) {
      Order order = new Order();
      order.setUserId(command.getUserId());
      order.setItems(command.getItems());

      // 비즈니스 규칙이 Service에 있음
      BigDecimal total = BigDecimal.ZERO;
      for (OrderLine item : command.getItems()) {
          total = total.add(item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())));
      }
      order.setTotalPrice(total);

      if (total.compareTo(BigDecimal.valueOf(1000)) < 0) {
          throw new MinOrderAmountException("최소 1,000원 이상");
      }

      order.setStatus("PLACED");
      orderRepository.save(order);
  }

문제:
  ① 비즈니스 규칙(최소 금액 검증, 금액 계산)이 Service에 있음
     → 같은 규칙이 다른 Service에 중복될 수 있음
     → OrderService.placeOrder()에도 있고
        AdminService.forcePlace()에도 있고
        BatchService.scheduledOrder()에도 있고...

  ② Order 객체는 단순 데이터 컨테이너 (setter 허용)
     → Order를 불완전한 상태로 만들 수 있음
     → order.setStatus("INVALID_STATUS") 가능

풍부한 도메인 모델 (Rich Domain Model):
  public class Order {
      // setter 없음, 불변성
      private final OrderId id;
      private final UserId userId;
      private final List<OrderLine> lines;
      private OrderStatus status;

      // 비즈니스 메서드가 Order에 있음
      public static Order create(UserId userId, List<OrderLine> lines) {
          // 생성 시 유효성 검증
          if (lines.isEmpty()) throw new IllegalArgumentException("항목 없음");
          return new Order(OrderId.newId(), userId, lines, OrderStatus.DRAFT);
      }

      public Money calculateTotal() {
          return lines.stream()
              .map(OrderLine::getSubtotal)
              .reduce(Money.ZERO, Money::add);
      }

      public void place() {
          if (this.calculateTotal().isLessThan(Money.of(1_000))) {
              throw new MinOrderAmountException(Money.of(1_000));
          }
          this.status = OrderStatus.PLACED;
      }
      // → 비즈니스 규칙이 Order 자신에게 있음
  }

  Service는 조율만:
  public void placeOrder(PlaceOrderCommand command) {
      Order order = Order.create(command.getUserId(), command.getLines());
      order.place(); // 비즈니스 규칙은 Order에 위임
      orderRepository.save(order);
  }
```

### 4. 순환 의존성 발생 경로

```
레이어드 아키텍처에서 순환 의존성이 생기는 전형적 경로:

시작: OrderService와 UserService가 서로 필요
  OrderService → UserService (주문 시 사용자 조회)
  UserService → OrderService (사용자 탈퇴 시 주문 처리)
  → 순환!

Spring이 해결?: @Lazy 어노테이션
  @Autowired @Lazy private OrderService orderService;
  → 순환을 숨기는 것이지 해결이 아님

올바른 해결:
  방법 1: 공통 의존 분리
    Order의 사용자 정보 조회: UserQueryPort (인터페이스) → UserRepository 직접
    → OrderService가 UserService를 호출하지 않아도 됨

  방법 2: 도메인 이벤트
    UserService가 OrderService를 직접 호출 대신
    UserWithdrawnEvent 발행 → OrderEventHandler가 구독하여 주문 처리
    → 순환 의존 제거 + 느슨한 결합

  방법 3: 책임 재분배
    "사용자 탈퇴 처리"라는 유스케이스를 별도 UseCase로
    WithdrawUserUseCase가 UserRepository + OrderRepository 모두 사용
    → OrderService, UserService 모두 이 UseCase를 모름
```

### 5. `@Transactional` 남용이 Service를 비대하게 만드는 이유

```
@Transactional이 Service 비대에 기여하는 방식:

  @Service
  @Transactional  // 클래스 레벨 @Transactional
  public class OrderService {

      public void placeOrder(...) {
          // JPA 트랜잭션 내에서 실행
          orderRepository.save(order);
          // 여기서 KakaoPay API 호출 (외부 API는 트랜잭션 롤백 불가)
          kakaoPayClient.charge(...);
          // 여기서 이메일 발송 (SMTP도 롤백 불가)
          mailSender.send(...);
          // 여기서 Kafka 발행 (비동기, 롤백 불가)
          kafkaTemplate.send(...);
      }
  }

문제 1: 트랜잭션이 롤백 불가능한 작업을 감쌈
  JPA save는 롤백 가능
  KakaoPay 결제, 이메일, Kafka는 롤백 불가
  → 트랜잭션이 실패하면 DB는 롤백되지만 결제/이메일은 이미 처리됨

문제 2: 트랜잭션 범위가 불필요하게 큼
  Service 메서드 진입부터 종료까지 DB 커넥션 점유
  외부 API 응답 기다리는 동안 (수백 ms) DB 커넥션 점유
  → 커넥션 풀 고갈 가능성

문제 3: @Transactional이 있어서 Service에 외부 로직도 넣게 됨
  "어차피 @Transactional 있으니까 여기서 다 처리하면 되겠다"
  → Service에 이질적 로직 집중

올바른 @Transactional 사용:
  @Transactional
  public OrderId placeOrder(PlaceOrderCommand command) {
      // 트랜잭션 내: DB 작업만
      Order order = Order.create(...);
      order.place();
      orderRepository.save(order);
      // 이벤트 저장 (outbox pattern)
      eventStore.save(new OrderPlacedEvent(order));
      return order.getId();
  }
  // 트랜잭션 완료 후: 외부 API, 이메일은 별도 처리
```

---

## 💻 실전 코드 — 함정 해결 코드 비교

```java
// ❌ Before: DTO 침투 + Fat Service + Anemic Domain

@Service @Transactional
public class OrderService {

    @Autowired private OrderRepository orderRepository;
    @Autowired private UserService userService;
    @Autowired private CouponRepository couponRepository;
    @Autowired private InventoryRepository inventoryRepository;
    @Autowired private KakaoPayClient kakaoPayClient;
    @Autowired private JavaMailSender mailSender;
    @Autowired private KafkaTemplate<String, String> kafka;

    // DTO 침투 + Fat Service 집약
    public Long placeOrder(PlaceOrderRequest request) { // ← Presentation DTO 직접 수신

        // 사용자 조회
        User user = userService.findById(request.getUserId());

        // 재고 확인 (비즈니스 로직이 Service에)
        for (PlaceOrderRequest.Item item : request.getItems()) {
            Inventory inv = inventoryRepository.findByItemId(item.getItemId());
            if (inv.getStock() < item.getQuantity()) {
                throw new InsufficientStockException(item.getItemId());
            }
        }

        // 금액 계산 (비즈니스 로직이 Service에)
        BigDecimal total = request.getItems().stream()
            .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        // 최소 금액 검증 (비즈니스 로직이 Service에)
        if (total.compareTo(BigDecimal.valueOf(1000)) < 0) {
            throw new MinOrderAmountException("최소 1,000원 이상");
        }

        // 쿠폰 적용
        if (request.getCouponCode() != null) {
            Coupon coupon = couponRepository.findByCode(request.getCouponCode());
            total = coupon.apply(total);
        }

        // Order 생성 (Anemic)
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setTotalPrice(total);
        order.setStatus("PLACED");
        orderRepository.save(order);

        // 결제 (트랜잭션 내에서 외부 API)
        kakaoPayClient.charge(user.getPaymentKey(), total);

        // 이메일 (트랜잭션 내에서 SMTP)
        SimpleMailMessage mail = new SimpleMailMessage();
        mailSender.send(mail);

        // 이벤트 (트랜잭션 내에서 Kafka)
        kafka.send("order-placed", order.getId().toString());

        return order.getId();
    }
}
```

```java
// ✅ After: 책임 분산, 풍부한 도메인 모델, 트랜잭션 올바른 범위

// === Domain Layer (풍부한 도메인 모델) ===

public class Order {
    private final OrderId id;
    private final UserId userId;
    private final List<OrderLine> lines;
    private OrderStatus status;

    public static Order create(UserId userId, List<OrderLine> lines) {
        Objects.requireNonNull(userId);
        if (lines == null || lines.isEmpty())
            throw new IllegalArgumentException("주문 항목 없음");
        return new Order(OrderId.newId(), userId, List.copyOf(lines), OrderStatus.DRAFT);
    }

    public void place(DiscountPolicy discountPolicy) {
        Money rawTotal = calculateRawTotal();
        Money discountedTotal = discountPolicy.apply(rawTotal); // 할인 정책 주입

        if (discountedTotal.isLessThan(Money.of(1_000)))
            throw new MinOrderAmountException(Money.of(1_000));

        this.status = OrderStatus.PLACED;
    }

    public Money calculateRawTotal() {
        return lines.stream().map(OrderLine::getSubtotal).reduce(Money.ZERO, Money::add);
    }
}

// === Application Layer (조율만, 외부 작업 분리) ===

@Service
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderRepository orderRepository;
    private final InventoryPort inventoryPort;
    private final DiscountPolicyPort discountPolicyPort;
    private final OrderEventStore eventStore; // Outbox pattern

    @Override
    @Transactional // DB 작업 범위만
    public OrderId placeOrder(PlaceOrderCommand command) {
        // 재고 확인 (Port를 통해)
        inventoryPort.validateStock(command.getLines());

        // 할인 정책 조회
        DiscountPolicy policy = discountPolicyPort.findPolicy(command.getCouponCode());

        // 도메인 로직: Order에 위임
        Order order = Order.create(command.getUserId(), command.getLines());
        order.place(policy); // 비즈니스 규칙은 Order가

        // DB 작업
        orderRepository.save(order);
        eventStore.save(new OrderPlacedEvent(order)); // Outbox: 트랜잭션 내 저장

        return order.getId();
        // 트랜잭션 커밋 후: Outbox 폴링으로 결제, 이메일, Kafka 처리
    }
}
```

---

## 📊 패턴 비교 — Fat Service vs 책임 분산

```
주문 생성 시 "최소 금액 검증 로직 변경" 요청:

=== Fat Service ===
변경 위치: OrderService.placeOrder() 메서드 내부
문제:
  OrderService 전체 테스트 재실행
  같은 로직이 AdminService.forceOrder()에도 있다면? 그쪽도 수정

=== 책임 분산 (Rich Domain Model) ===
변경 위치: Order.place() 메서드만
이익:
  OrderTest만 재실행 (빠른 단위 테스트)
  OrderService, AdminService 등 Order를 사용하는 모든 곳에 자동 반영
  → 비즈니스 규칙은 한 곳에만 존재

테스트 시간 비교:
  Fat Service 검증: @SpringBootTest → 30초
  Rich Domain 검증: 순수 Java 단위 테스트 → 0.01초
```

---

## ⚖️ 트레이드오프

```
Rich Domain Model의 비용:
  초기 설계 시 도메인 객체에 어떤 메서드를 넣을지 결정 필요
  도메인 객체가 복잡해질 수 있음 (너무 많은 메서드)
  서비스 간 공유 로직이 어느 도메인에 위치하는지 판단 필요

DTO 레이어 변환의 비용:
  각 레이어 경계마다 변환 코드 (Command, Result, Response)
  매핑 클래스(Mapper)가 증가

트레이드오프 정리:
  Fat Service → 단기 개발 속도 ↑, 장기 유지보수 ↓
  Rich Domain → 단기 설계 비용 ↑, 장기 유지보수 ↑

  DTO 침투 → 단기 코드량 ↓, 장기 변경 비용 ↑
  레이어 변환 → 단기 코드량 ↑, 장기 변경 비용 ↓
```

---

## 📌 핵심 정리

```
레이어드 아키텍처 3대 함정:

① DTO 침투
  증상: Controller DTO가 Service, Repository 파라미터로 직접 전달
  결과: API 스펙 변경이 Business Layer까지 파급
  해결: 레이어 경계에서 Command/Result로 변환

② Fat Service (+ Anemic Domain Model)
  증상: Service가 500줄 이상, 비즈니스 로직이 모두 Service에
  결과: 비즈니스 규칙 중복, 테스트 어려움
  해결: 비즈니스 로직을 Domain 객체로 이동 (Rich Domain Model)

③ @Transactional 남용
  증상: 외부 API, 이메일, Kafka가 트랜잭션 내에서 실행
  결과: 롤백 불가 작업의 일관성 문제, DB 커넥션 점유
  해결: 트랜잭션 = DB 작업만, 외부 작업 = Outbox 패턴

이 세 가지 함정은 레이어드 아키텍처의 구조적 특성에서 비롯됨
→ 레이어드 내에서 개선 가능하지만 근본 해결은 Hexagonal로
```

---

## 🤔 생각해볼 문제

**Q1.** Anemic Domain Model을 "나쁘다"고 하는데, Martin Fowler는 이를 "anti-pattern"으로 분류했다. 하지만 실제로 많은 Spring Boot 프로젝트에서 Anemic Domain Model을 사용한다. 모든 경우에 Rich Domain Model이 더 좋은가?

<details>
<summary>해설 보기</summary>

**항상 Rich Domain Model이 더 좋은 것은 아닙니다.**

Anemic Domain Model이 합리적인 경우:
- 단순 CRUD 서비스 — 비즈니스 규칙이 거의 없을 때
- 스크립트 지향 트랜잭션 스크립트 패턴이 더 직관적인 경우
- 팀이 DDD에 익숙하지 않아서 Rich Domain Model 설계가 오히려 더 복잡해질 때
- 도메인 객체가 여러 Bounded Context에서 공유되어 비즈니스 메서드 추가가 어려울 때

Anemic이 문제가 되는 경우:
- 같은 비즈니스 규칙(금액 계산, 상태 전이 검증)이 여러 Service에 중복
- 도메인 객체를 불완전한 상태로 만들 수 있는 setter 노출
- 테스트하고 싶은 비즈니스 규칙이 항상 Service를 통해야 해서 느린 테스트

판단 기준:
- "비즈니스 규칙이 여러 Service에 중복 등장하기 시작했는가?" → Rich Domain 전환 신호
- "도메인 객체를 불완전한 상태로 만들 수 있는 setter가 공개되어 있는가?" → 위험 신호

</details>

---

**Q2.** Service 간 순환 의존성을 `@Lazy`로 해결하면 안 되는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

`@Lazy`는 **순환 의존성의 증상을 숨길 뿐, 근본 문제를 해결하지 않습니다.**

```java
@Service
public class OrderService {
    @Autowired @Lazy
    private UserService userService; // 순환을 @Lazy로 회피
}
```

`@Lazy`의 문제점:
1. **설계 문제를 숨김**: OrderService와 UserService가 서로 필요하다는 것은 두 서비스의 경계가 잘못 정의됐다는 신호
2. **런타임 오류 가능성**: 지연 초기화된 빈이 실제 사용 시점에 프록시 관련 문제 발생 가능
3. **테스트 복잡도**: `@Lazy` 빈을 Mock할 때 추가 설정 필요

근본 해결:
```java
// 순환의 이유 파악:
// OrderService.placeOrder() → userService.findById() (사용자 정보 필요)
// UserService.withdraw() → orderService.cancelAll() (탈퇴 시 주문 취소)

// 해결 1: UserQueryPort로 UserService 직접 의존 대신 Repository 사용
public class OrderService {
    private final UserRepository userRepository; // UserService 대신 Repository
}

// 해결 2: 도메인 이벤트로 순환 제거
// UserService: UserWithdrawnEvent 발행
// OrderEventHandler: UserWithdrawnEvent 구독 → 주문 취소 처리
```

순환 의존성이 생겼다면 두 서비스의 **경계(Boundary) 설계를 재검토**해야 합니다.

</details>

---

**Q3.** Outbox Pattern이란 무엇이고, 트랜잭션 내에서 Kafka를 직접 호출하는 것과 어떻게 다른가?

<details>
<summary>해설 보기</summary>

**Outbox Pattern**: 이벤트 발행을 외부 시스템에 직접 보내는 대신, DB의 `event_outbox` 테이블에 같은 트랜잭션으로 저장하고, 별도 폴링/CDC로 발행하는 패턴입니다.

```java
// ❌ 트랜잭션 내 Kafka 직접 호출
@Transactional
public OrderId placeOrder(...) {
    orderRepository.save(order);         // DB 트랜잭션 내
    kafkaTemplate.send("order", event);  // Kafka는 트랜잭션 밖
    // DB 커밋 성공 + Kafka 발행 실패 → 이벤트 유실!
    // 또는 DB 커밋 실패 + Kafka 발행 성공 → 이벤트 중복!
}

// ✅ Outbox Pattern
@Transactional
public OrderId placeOrder(...) {
    orderRepository.save(order);
    // 이벤트를 DB에 저장 (같은 트랜잭션)
    outboxRepository.save(new OutboxEvent("order-placed", serialize(event)));
    // DB 커밋 → outbox 테이블도 같이 커밋 → 원자성 보장
}

// 별도 스케줄러 or Debezium CDC:
// outbox 테이블 폴링 → 미발행 이벤트 발견 → Kafka 발행 → 발행 완료 표시
```

핵심 차이:
- 직접 호출: DB 커밋과 Kafka 발행이 **원자적이지 않음** → 일관성 위험
- Outbox Pattern: DB 저장과 이벤트 저장이 **같은 트랜잭션** → 원자성 보장, 이벤트 유실 없음

</details>

---

<div align="center">

**[⬅️ 이전: 엄격한 레이어 vs 느슨한 레이어](./02-strict-vs-relaxed-layers.md)** | **[홈으로 🏠](../README.md)** | **[다음: 레이어드 아키텍처에서 테스트 ➡️](./04-testing-in-layered.md)**

</div>
