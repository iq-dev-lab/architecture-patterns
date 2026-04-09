# 전통적 레이어드 아키텍처의 문제 — 의존성이 DB 방향으로 흐르는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Controller → Service → Repository 구조에서 의존성이 왜 DB 방향으로 흐르는가?
- 도메인이 영속성에 오염된다는 것은 무슨 의미인가?
- 레이어드 아키텍처에서 왜 Spring Context 없이 단위 테스트가 어려운가?
- Fat Service 문제는 어떻게 생겨나고, 왜 레이어드 구조가 이를 유발하는가?
- 의존성 역전으로 레이어드 아키텍처의 문제를 어떻게 해결하는 방향을 잡는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

레이어드 아키텍처는 직관적이다. Controller → Service → Repository라는 3단 구조는 처음 Spring을 배울 때 가장 먼저 접하는 패턴이다.

그러나 이 구조의 핵심 문제는 **의존성이 DB(인프라) 방향으로 흐른다**는 것이다. 도메인(비즈니스 규칙)이 인프라(JPA, DB 스키마)를 알게 되고, 결과적으로 "비즈니스 로직을 테스트하려면 DB가 필요한" 상황이 만들어진다. 이 문제를 이해해야 Hexagonal/Clean Architecture가 왜 이 방향을 역전시키는지 납득할 수 있다.

---

## 😱 흔한 실수 (Before — 레이어드 아키텍처의 일반적인 사용 방식)

```
6개월된 Spring Boot 프로젝트의 전형적인 상태:

structure:
  controller/
    OrderController.java
  service/
    OrderService.java  ← 현재 823줄
  repository/
    OrderRepository.java (JpaRepository 상속)
  entity/
    Order.java (@Entity 붙어있음)

OrderService.java (823줄) 내부:
  - 주문 생성 로직
  - 주문 조회 로직
  - 결제 연동 (KakaoPay SDK 직접 호출)
  - 재고 차감 (InventoryService 직접 호출)
  - 이메일 발송 (JavaMailSender 직접 사용)
  - 포인트 적립 (PointService 직접 호출)
  - 배송 생성 (ShippingService 직접 호출)
  - 어드민 기능 (상태 강제 변경 등)
  - 통계용 조회 로직

문제들:
  ① "주문 생성 로직만 단위 테스트하고 싶다"
     → OrderService 생성에 7개 의존성 전부 Mock 필요
     → Mock 설정 코드가 테스트 코드보다 길어짐

  ② "결제 방식을 토스페이로 바꿔주세요"
     → OrderService 823줄 중 어디에 결제 코드가 있는지 찾아야 함
     → KakaoPay SDK 코드가 주문 로직과 뒤섞여있어 추출 어려움

  ③ "@Entity Order 클래스에 비즈니스 로직도 있어요"
     → Order.java에 @Entity + @Column + 비즈니스 메서드 혼재
     → JPA 없이 Order 도메인 테스트 불가

  ④ "레이어드 구조인데 왜 Controller에서 Repository를 직접 쓰죠?"
     → Relaxed Layer로 되어있어서 어느 레이어든 하위 레이어 자유 접근
     → 실질적으로 레이어 의미가 없어짐
```

---

## ✨ 올바른 접근 (After — 문제를 인식하고 방향을 잡을 때)

```
레이어드 아키텍처의 문제를 이해한 후의 설계 방향:

인식해야 할 핵심:
  "의존성이 DB 방향으로 흐른다" = 도메인이 인프라를 안다
  이것을 역전시키는 것이 Hexagonal/Clean Architecture의 핵심

즉각 적용 가능한 개선 (레이어드 내에서):
  ① Repository 인터페이스를 Service 레이어로 올리기
     OrderService → OrderRepository (JPA 직접 X, 인터페이스만)

  ② Entity와 Domain Model 분리 시작
     Order.java (순수 비즈니스 로직)
     OrderJpaEntity.java (@Entity, 영속성 매핑)

  ③ 외부 의존성 인터페이스 추출
     PaymentPort, NotificationPort로 KakaoPay, EmailSender 추상화

이 방향이 Hexagonal Architecture로 자연스럽게 이어짐:
  (자세한 내용은 Chapter 3에서 다룸)
```

---

## 🔬 내부 원리 — 레이어드 아키텍처의 구조적 문제

### 1. 의존성이 DB 방향으로 흐르는 구조

```
전통적 레이어드 아키텍처의 의존성 방향:

  Controller ──→ Service ──→ Repository ──→ DB Schema
  (Presentation)  (Business)  (Data Access)

화살표(→)의 의미: "~를 알고 있다", "~에 의존한다"

무슨 일이 일어나는가:
  Service가 Repository를 알고 있음 (JpaRepository 구현체를)
  = Service(도메인)가 JPA(인프라)를 알고 있음
  = 비즈니스 규칙이 영속성 기술에 묶임

DB Schema가 바뀌면:
  Repository 변경 → Service 변경 필요 → Controller 변경 가능성
  DB 변경이 비즈니스 로직까지 전파

이것이 Uncle Bob이 말한 "아키텍처 역전":
  도메인(비즈니스 규칙)이 인프라(JPA, DB)에 의존
  = 더 중요한 것이 덜 중요한 것에 의존
  = 비즈니스 가치는 JPA보다 중요하지만, JPA에 종속됨

올바른 방향 (Hexagonal):
  Service ──→ Repository(인터페이스)
                   ↑ 구현
              JpaRepository(인프라)

  = 인프라가 도메인 인터페이스에 의존 (역전!)
  = 도메인이 JPA를 모름
```

### 2. 도메인이 영속성에 오염되는 메커니즘

```
전형적인 JPA Entity + 도메인 혼재:

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id")
    private Long userId; // userId인가 User 객체인가?

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<OrderLine> lines; // LAZY 로딩 - 비즈니스 로직에서 N+1 유발 가능

    @Column(name = "status")
    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    // JPA 요구사항: 기본 생성자 필수
    protected Order() {} // 비즈니스 규칙상 기본 생성자가 없어야 할 수도 있음

    // 비즈니스 로직 (이 클래스에 있는 게 맞는가?)
    public void place() {
        if (this.lines == null || this.lines.isEmpty()) {
            throw new IllegalStateException("주문 항목이 없습니다");
        }
        this.status = OrderStatus.PLACED;
    }

    // JPA 편의 메서드
    public void addLine(OrderLine line) {
        this.lines.add(line);
        line.setOrder(this); // 양방향 관계 설정 (JPA 요구사항)
    }
}

문제 1: JPA 제약이 비즈니스 규칙을 오염
  - 기본 생성자 강제 → 불완전한 Order 객체 생성 가능
  - LAZY 로딩 → @Transactional 범위 밖에서 LazyInitializationException
  - 양방향 관계 → 비즈니스 로직에서 setOrder() 같은 JPA 전용 메서드 호출

문제 2: JPA 없이는 Order를 테스트할 수 없음
  Order order = new Order(); // JPA Entity인데 new로 생성?
  // Spring Context 없어도 되지만, JPA 어노테이션이 있으면
  // DB 연결 없이 객체 그래프 탐색 시 예외 가능

문제 3: DB 스키마 변경이 비즈니스 객체를 변경
  DB 컬럼명 변경: @Column(name = "user_id") → (name = "customer_id")
  → Order.java 수정 = 비즈니스 객체가 DB 스키마 변경에 영향받음
```

### 3. Fat Service — 레이어드 구조가 비대한 Service를 만드는 이유

```
레이어드 아키텍처에서 Service 레이어가 비대해지는 구조적 이유:

레이어드 아키텍처의 규칙:
  Controller는 View/API 처리만
  Repository는 DB 접근만
  → 그 사이의 "모든 것"이 Service로 집중

Service가 담당하게 되는 것들:
  ① 비즈니스 로직 (마땅히 있어야 함)
  ② 외부 API 호출 (KakaoPay, SMS, Email)
  ③ 트랜잭션 조율 (여러 Repository 호출 순서)
  ④ 유효성 검증 (Controller에서 못한 것)
  ⑤ DTO 변환 (Entity → 응답 DTO)
  ⑥ 캐싱 로직 (Redis 직접 호출)
  ⑦ 이벤트 발행 (Kafka 직접 호출)

결과:
  OrderService: 823줄
  PaymentService: 612줄
  UserService: 1,043줄 (가장 많이 의존받는 서비스)

Fat Service의 특징:
  - 메서드 간 private 메서드 공유 (유사 도메인 로직 파편화)
  - @Transactional 남용 (필요 이상으로 큰 트랜잭션 범위)
  - 테스트를 위한 Mock 수가 계속 증가
  - 새 기능 추가 시 항상 OrderService를 건드려야 함

근본 원인:
  비즈니스 규칙이 도메인 객체(Order, User)에 있지 않고
  Service 레이어에 집중됨
  → Domain Model이 빈약한 "Anemic Domain Model" 상태
```

### 4. 테스트가 어려운 이유 — 의존성 그래프와 테스트의 관계

```
테스트하고 싶은 것: "주문 금액이 최소 주문 금액보다 낮으면 주문 실패"

테스트 코드 작성 시도:
  OrderService service = new OrderService(
      ???  // OrderJpaRepository - JPA 필요
      ???  // UserService - UserJpaRepository 필요 (재귀적)
      ???  // KakaoPayClient - 외부 API
      ???  // JavaMailSender - SMTP 서버
      ???  // InventoryService - InventoryJpaRepository 필요
      ???  // KafkaTemplate - Kafka 브로커
  );

문제: 단순한 비즈니스 규칙 하나를 테스트하려면
  JPA(DB) + 외부 결제 API + 이메일 서버 + Kafka 전부 필요

실제 선택지:
  ① @SpringBootTest 사용 → 전체 컨텍스트 로딩 (30초 이상)
  ② Mockito로 모두 Mock → Mock 설정 코드 > 실제 테스트 코드
  ③ 테스트 안 씀 → 부채 누적

Mockito Mock 지옥의 예시:
  @ExtendWith(MockitoExtension.class)
  class OrderServiceTest {

      @Mock OrderJpaRepository orderRepository;
      @Mock UserService userService;
      @Mock KakaoPayClient kakaoPayClient;
      @Mock JavaMailSender mailSender;
      @Mock InventoryService inventoryService;
      @Mock KafkaTemplate kafkaTemplate;
      @Mock CacheManager cacheManager;

      @InjectMocks OrderService orderService;

      @Test
      void 최소주문금액_미달_시_실패() {
          // given
          given(userService.findById(anyLong()))
              .willReturn(new User(1L, "test@test.com"));
          given(inventoryService.check(anyList()))
              .willReturn(true);
          // 테스트와 무관한 Mock 설정이 계속 추가됨...

          // when & then
          // 실제 테스트하려는 로직: 금액 검증 단 하나
      }
  }

의존성이 도메인 객체에 있을 때 (Hexagonal):
  @Test
  void 최소주문금액_미달_시_실패() {
      Order order = Order.create(items, userId);
      assertThatThrownBy(() -> order.place(minOrderAmount))
          .isInstanceOf(MinOrderAmountViolationException.class);
  }
  // Mock 없음, Spring 없음, DB 없음 — 10ms 실행
```

### 5. 레이어드 아키텍처의 올바른 사용법 (개선 방향)

```
레이어드 아키텍처 자체가 나쁜 것이 아님
문제는 "구현 방식"

개선 방향 1: Repository 인터페이스를 Business 레이어로

  Before:
    Service → OrderJpaRepository (구체 클래스)

  After:
    domain/OrderRepository (인터페이스) ← Service 의존
    infrastructure/JpaOrderRepository implements OrderRepository

개선 방향 2: Entity와 Domain Model 분리

  Before:
    Order (@Entity + 비즈니스 메서드 혼재)

  After:
    domain/Order (순수 Java, 비즈니스 메서드)
    infrastructure/OrderJpaEntity (@Entity, 영속성 매핑)
    infrastructure/OrderMapper (Order ↔ OrderJpaEntity 변환)

개선 방향 3: 외부 의존성 인터페이스 추출

  Before:
    OrderService → KakaoPayClient (구체 클래스)
    OrderService → JavaMailSender (구체 클래스)

  After:
    domain/PaymentPort (인터페이스)
    domain/NotificationPort (인터페이스)
    infrastructure/KakaoPayAdapter implements PaymentPort
    infrastructure/SmtpNotificationAdapter implements NotificationPort

이 세 가지 개선이 모이면 = Hexagonal Architecture
```

---

## 💻 실전 코드 — 문제 있는 코드와 개선 방향

```java
// ❌ Before: 전형적인 레이어드 아키텍처 (문제 집약)

@Entity
@Table(name = "orders")
public class Order { // JPA Entity + 도메인 모델 혼재
    @Id @GeneratedValue
    private Long id;

    @Column(name = "total_price")
    private BigDecimal totalPrice; // DB 컬럼명에 의존

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<OrderLine> lines;

    protected Order() {} // JPA 요구 기본 생성자 → 불완전 객체 허용

    // 비즈니스 로직이 @Entity 클래스에 있음
    public void place() {
        if (lines.isEmpty()) throw new IllegalStateException("항목 없음");
        // lines가 LAZY라면 여기서 LazyInitializationException 가능
    }
}

@Service
@Transactional
public class OrderService { // Fat Service

    @Autowired private OrderJpaRepository orderRepository;  // 구체 JPA
    @Autowired private UserService userService;             // 서비스 간 의존
    @Autowired private KakaoPayClient kakaoPayClient;      // 외부 SDK
    @Autowired private JavaMailSender mailSender;           // 인프라
    @Autowired private InventoryService inventoryService;  // 서비스 간 의존
    @Autowired private KafkaTemplate<String, String> kafka; // 인프라

    // 823줄 중 일부
    public OrderResult placeOrder(OrderRequest request) {
        User user = userService.findById(request.getUserId()); // 서비스 직접 호출
        inventoryService.decreaseStock(request.getItemIds());  // 서비스 직접 호출

        Order order = new Order();
        // ... 주문 생성

        orderRepository.saveAndFlush(order); // JPA 구체 메서드

        kakaoPayClient.charge(user.getPaymentKey(), order.getTotal()); // SDK 직접

        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(user.getEmail());
        mailSender.send(message); // SMTP 직접

        kafka.send("order-placed", order.getId().toString()); // Kafka 직접

        return new OrderResult(order.getId());
    }
}
```

```java
// ✅ After: 레이어드 아키텍처 개선 방향 (Hexagonal로의 첫 단계)

// === domain/ ===

// 순수 도메인 모델 (JPA 어노테이션 없음)
public class Order {
    private final OrderId id;
    private final List<OrderLine> lines;
    private OrderStatus status;

    // 완전한 상태의 객체만 생성 가능 (기본 생성자 없음)
    public static Order create(List<OrderLine> lines) {
        if (lines == null || lines.isEmpty()) {
            throw new IllegalArgumentException("주문 항목이 없습니다");
        }
        return new Order(OrderId.newId(), lines, OrderStatus.DRAFT);
    }

    public void place(Money minOrderAmount) {
        if (this.calculateTotal().isLessThan(minOrderAmount)) {
            throw new MinOrderAmountViolationException(minOrderAmount);
        }
        this.status = OrderStatus.PLACED;
    }

    // JPA 없이 순수 Java로 테스트 가능!
}

// Driven Port (인터페이스)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentPort {
    PaymentResult charge(PaymentRequest request);
}

public interface OrderEventPublisher {
    void publishOrderPlaced(Order order);
}

// === application/ ===

@Service
public class PlaceOrderService implements PlaceOrderUseCase {

    // 모두 인터페이스에 의존 (구체 클래스 없음)
    private final OrderRepository orderRepository;
    private final PaymentPort paymentPort;
    private final OrderEventPublisher eventPublisher;

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.getLines());
        order.place(command.getMinOrderAmount()); // 비즈니스 규칙은 도메인에

        orderRepository.save(order);
        paymentPort.charge(PaymentRequest.of(order));
        eventPublisher.publishOrderPlaced(order);

        return order.getId();
    }
}

// === 단위 테스트 (Spring 없음, DB 없음) ===

class PlaceOrderServiceTest {

    // 람다로 간단하게 Port 구현
    private final List<Order> savedOrders = new ArrayList<>();
    private final OrderRepository orderRepository = order -> savedOrders.add(order);
    private final PaymentPort paymentPort = req -> PaymentResult.success("tx-001");
    private final OrderEventPublisher eventPublisher = order -> {};

    private final PlaceOrderService sut = new PlaceOrderService(
        orderRepository, paymentPort, eventPublisher
    );

    @Test
    void 최소주문금액_미달_시_주문_실패() {
        PlaceOrderCommand command = PlaceOrderCommand.builder()
            .lines(List.of(OrderLine.of(item, 1)))
            .minOrderAmount(Money.of(10_000))
            .build();

        // 총 금액이 최소 주문 금액보다 낮은 경우
        assertThatThrownBy(() -> sut.placeOrder(command))
            .isInstanceOf(MinOrderAmountViolationException.class);
    }
    // Mock 없음, @SpringBootTest 없음, 10ms 실행
}
```

---

## 📊 패턴 비교 — 레이어드 아키텍처 vs 개선된 구조

```
"새 결제 수단 추가" 시나리오에서의 비교:

=== 전통적 레이어드 ===
변경 위치:
  OrderService.java
    + else if (paymentType == TOSS) { tossPayClient.charge(); }
  OrderService 테스트
    + tossPayClient Mock 추가

리스크:
  OrderService에 손댔으므로 기존 카카오페이 로직 영향 가능
  테스트: 전체 OrderService Mock 재설정

=== 개선된 구조 (Hexagonal 방향) ===
변경 위치:
  TossPayAdapter.java (신규 파일 추가만)

리스크:
  PlaceOrderService 변경 없음
  기존 KakaoPayAdapter 변경 없음
  TossPayAdapterTest 단독으로 추가

==================================================

테스트 속도 비교:

전통적 레이어드 (OrderService):
  @SpringBootTest 필요: 약 25초
  Mockito Mock 전체 설정: 약 0.5초
  실제 테스트 로직: 약 0.01초
  전체: 약 25.5초/테스트

개선된 구조:
  Spring Context 없음: 0초
  Mock 설정 없음: 0초
  실제 테스트 로직: 약 0.01초
  전체: 약 0.01초/테스트

100개 테스트 기준:
  레이어드: 2,550초 (42분)
  개선된 구조: 1초
```

---

## ⚖️ 트레이드오프

```
레이어드 아키텍처가 여전히 유효한 경우:
  ① 단순 CRUD 서비스 — 비즈니스 규칙이 거의 없는 경우
  ② 팀 전체가 Hexagonal을 학습할 시간이 없는 경우
  ③ 빠른 프로토타입 — 2주 내 검증 후 폐기할 코드

레이어드 아키텍처를 개선해야 하는 신호:
  ① @SpringBootTest 없이 Service 단위 테스트가 불가능해짐
  ② Service 클래스가 500줄을 초과함
  ③ "이 코드 건드리면 어디서 터질지 모르겠다"는 말이 나올 때
  ④ 외부 API(결제, 알림) 교체 시 Service 코드 수정이 필요할 때

점진적 개선 전략:
  1단계: Repository 인터페이스를 도메인으로 올리기 (1일)
  2단계: 외부 의존성(결제, 알림) Port로 추상화 (2-3일)
  3단계: Entity와 Domain Model 분리 (1주)
  4단계: UseCase 인터페이스 도입 (1-2일)
  → 이 과정이 Hexagonal Architecture로의 자연스러운 전환
```

---

## 📌 핵심 정리

```
레이어드 아키텍처의 핵심 문제:

의존성 방향:
  Controller → Service → Repository → DB
  = 도메인(Service)이 인프라(JPA)를 알고 있음
  = DIP 위반: 고수준이 저수준에 의존

결과:
  ① DB 변경 → Service 코드 영향 가능
  ② 도메인 테스트에 DB 필요 → @SpringBootTest 강제
  ③ Fat Service — 모든 책임이 Service로 집중
  ④ @Entity 클래스에 비즈니스 로직 혼재

해결 방향 (Hexagonal로):
  Repository 인터페이스를 도메인으로
  외부 의존성을 Port로 추상화
  Entity와 Domain Model 분리
  = 의존성이 도메인을 향하도록 역전

이것이 Hexagonal과 Clean Architecture가 존재하는 이유
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional`을 Service 메서드에 붙이는 것이 레이어드 아키텍처의 어떤 문제를 심화시키는가?

<details>
<summary>해설 보기</summary>

`@Transactional`은 두 가지 방식으로 문제를 심화시킵니다.

**① 트랜잭션이 Service를 인프라에 결합시킴:**
```java
@Service
@Transactional // Spring의 트랜잭션 관리에 의존
public class OrderService {
    // Service가 JPA 트랜잭션 생명주기에 의존
    // = Spring 없이 OrderService 테스트 불가 (트랜잭션 처리 안 됨)
}
```

**② 트랜잭션 범위가 지나치게 커짐:**
```java
@Transactional
public OrderResult placeOrder(OrderRequest request) {
    // 주문 저장 (JPA 트랜잭션)
    // 결제 API 호출 (외부 API — 롤백 불가!)
    // 이메일 발송 (SMTP — 롤백 불가!)
    // Kafka 발행 (비동기 — 롤백 불가!)
}
// 트랜잭션이 롤백 불가능한 외부 작업들을 감싸는 잘못된 설계
```

개선된 설계에서는 트랜잭션을 도메인 작업에만 제한하고, 외부 API 호출은 트랜잭션 밖에서 처리합니다 (예: 도메인 이벤트 방식).

</details>

---

**Q2.** JPA의 `@OneToMany(fetch = FetchType.LAZY)`는 도메인 로직에 어떤 문제를 일으키는가?

<details>
<summary>해설 보기</summary>

LAZY 로딩은 **"도메인이 JPA의 프록시 메커니즘을 알아야 한다"** 는 결합을 만듭니다.

```java
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderLine> lines;

    // 도메인 로직
    public Money calculateTotal() {
        return lines.stream() // ← @Transactional 밖이면 예외!
            .map(OrderLine::getPrice)
            .reduce(Money.ZERO, Money::add);
    }
}
```

문제:
1. `calculateTotal()`은 순수 비즈니스 로직이지만, JPA 세션이 열려있어야 동작
2. `@Transactional` 범위 밖에서 호출 시 `LazyInitializationException`
3. 이를 해결하기 위해 `EAGER`로 바꾸면 N+1 문제 발생 가능

Domain Model과 JPA Entity를 분리하면:
- `Order` (도메인): `List<OrderLine> lines` — 완전히 로딩된 컬렉션
- `OrderJpaEntity`: LAZY 로딩 설정 — 영속성 최적화
- `OrderMapper`: JPA Entity → Domain Model 변환 시 필요한 것만 로딩

</details>

---

**Q3.** 레이어드 아키텍처에서 Service가 다른 Service를 호출하는 것(`OrderService → UserService`)은 어떤 문제를 만드는가? 어떻게 개선할 수 있는가?

<details>
<summary>해설 보기</summary>

서비스 간 직접 의존은 **순환 의존성**과 **테스트 복잡도** 문제를 만듭니다.

```java
// 문제 상황
@Service class OrderService {
    @Autowired UserService userService; // UserService에 의존
}
@Service class UserService {
    @Autowired OrderService orderService; // OrderService에도 의존?! → 순환!
}
```

순환이 아니어도 문제:
```java
// OrderService 테스트에 UserService Mock이 필요
// UserService Mock 내부가 복잡하면 테스트 설정도 복잡해짐
```

**개선 방법 1: Repository를 직접 사용**
```java
@Service class OrderService {
    // UserService 대신 UserRepository(인터페이스)에만 의존
    private final UserQueryPort userQueryPort; // 필요한 조회만
}
```

**개선 방법 2: 도메인 이벤트**
```java
// OrderService가 UserService를 직접 호출하는 대신
// 도메인 이벤트를 발행하고 UserService가 구독
eventPublisher.publish(new OrderPlacedEvent(order));
// UserService는 이 이벤트를 구독하여 필요한 작업 수행
```

서비스 간 의존은 결합도를 높이고 테스트를 어렵게 만듭니다. 가능하면 Repository를 통한 데이터 접근이나 이벤트 기반 통신으로 대체하는 것이 좋습니다.

</details>

---

<div align="center">

**[⬅️ 이전: SOLID 원칙과 아키텍처](./03-solid-and-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: 아키텍처 선택 기준 ➡️](./05-architecture-selection-criteria.md)**

</div>
