# 테스트 용이성 향상 — InMemory Adapter로 빠른 단위 테스트

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Port 인터페이스 덕분에 InMemory Adapter로 빠른 단위 테스트가 가능한 정확한 이유는?
- Spring Context 없이 도메인 로직을 테스트하는 구체적인 방법은?
- Adapter(JPA, Kafka, 외부 API)를 각각 독립적으로 테스트하는 전략은?
- Hexagonal Architecture에서 `@SpringBootTest` 의존 비율이 어떻게 줄어드는가?
- 도메인 단위 테스트, Adapter 통합 테스트, E2E 테스트를 각각 어떻게 구성하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Hexagonal Architecture의 가장 즉각적이고 측정 가능한 이익이 테스트 속도다. 레이어드 아키텍처에서 100개 테스트가 50분 걸리던 것이 Hexagonal로 전환하면 5분으로 줄어든다.

이것은 추상적 이론이 아니다 — Port 인터페이스가 있으면 InMemory 구현체를 만들 수 있고, 그 InMemory 구현체로 Spring/JPA/Kafka 없이 비즈니스 로직을 테스트할 수 있다. 이 메커니즘을 이해하면 어떤 도메인에도 동일하게 적용할 수 있다.

---

## 😱 흔한 실수 (Before — Hexagonal이지만 여전히 느린 테스트)

```
패키지 구조는 Hexagonal인데 테스트는 여전히 느린 경우:

실수 1: Port 인터페이스가 있어도 @SpringBootTest 사용
  @SpringBootTest
  class PlaceOrderServiceTest {
      @Autowired PlaceOrderUseCase placeOrderUseCase; // InMemory 안 씀
      // 30초 이상
  }

실수 2: InMemory Adapter 대신 Mock 남용
  @ExtendWith(MockitoExtension.class)
  class PlaceOrderServiceTest {
      @Mock OrderSavePort orderSavePort;   // 행위 검증에 결합
      @Mock PaymentPort paymentPort;
      @InjectMocks PlaceOrderService service;

      @Test
      void 성공() {
          service.placeOrder(command);
          verify(orderSavePort, times(1)).save(any()); // 구현 결합
      }
  }

실수 3: 테스트에서 Port 경계를 무시하고 구현체 직접 사용
  class PlaceOrderServiceTest {
      @Autowired JpaOrderRepository jpaRepo; // InMemory 대신 실제 JPA
      // @DataJpaTest or @SpringBootTest 필요
  }

결과:
  Hexagonal 구조를 갖췄지만 테스트 이익을 얻지 못함
  "Hexagonal 해봤는데 별로 빠르지 않던데요"
```

---

## ✨ 올바른 접근 (After — InMemory Adapter + 계층별 테스트 전략)

```
테스트 전략 전체 그림:

Layer 1: Domain 단위 테스트 (가장 많음, 가장 빠름)
  대상: Order, Money, OrderLine 등 Domain 객체
  실행 시간: ~0.001초/테스트
  Spring/JPA/외부 API: 없음

Layer 2: UseCase 단위 테스트 (많음, 빠름)
  대상: PlaceOrderService, CancelOrderService
  InMemory Adapter: OrderSavePort, PaymentPort 등 InMemory 구현
  실행 시간: ~0.01초/테스트
  Spring/JPA/외부 API: 없음

Layer 3: Adapter 통합 테스트 (중간)
  대상: JpaOrderRepository, KakaoPayAdapter
  @DataJpaTest or TestContainers (DB Adapter)
  MockWebServer (외부 API Adapter)
  실행 시간: ~5초/테스트

Layer 4: E2E 테스트 (소수, 느림)
  대상: 전체 플로우 (HTTP → Domain → DB)
  @SpringBootTest + TestContainers
  실행 시간: ~30초/테스트

비율 목표:
  Domain 단위: 50%
  UseCase 단위: 30%
  Adapter 통합: 15%
  E2E: 5%
```

---

## 🔬 내부 원리 — 테스트 가능성의 메커니즘

### 1. Domain 단위 테스트 — 순수 Java, 0ms

```java
// Order 도메인 테스트 (Spring 없음, JPA 없음, 어노테이션 없음)
class OrderTest {

    @Test
    void 주문_생성_성공() {
        // given
        UserId userId = UserId.of("user-001");
        List<OrderLine> lines = List.of(
            OrderLine.of(ItemId.of("item-1"), 2, Money.of(5_000))
        );

        // when
        Order order = Order.create(userId, lines);

        // then
        assertThat(order.getId()).isNotNull();
        assertThat(order.getUserId()).isEqualTo(userId);
        assertThat(order.getStatus()).isEqualTo(OrderStatus.DRAFT);
    }

    @Test
    void 최소_금액_미달_시_주문_불가() {
        // 총 금액 = 500원 (최소 1,000원 필요)
        Order order = Order.create(
            UserId.of("user-001"),
            List.of(OrderLine.of(ItemId.of("item-1"), 1, Money.of(500)))
        );

        assertThatThrownBy(order::place)
            .isInstanceOf(MinOrderAmountException.class)
            .hasMessageContaining("1,000");
    }

    @Test
    void 빈_항목으로_주문_생성_불가() {
        assertThatThrownBy(() ->
            Order.create(UserId.of("user-001"), Collections.emptyList())
        ).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void 결제_확인_후_상태_변경() {
        Order order = Order.create(UserId.of("user-001"), validLines());
        order.place();

        order.confirmPayment("tx-001");

        assertThat(order.getStatus()).isEqualTo(OrderStatus.PAYMENT_CONFIRMED);
        assertThat(order.getPaymentTransactionId()).isEqualTo("tx-001");
    }
    // 실행 시간: ~0.001초, Spring 없음, DB 없음
}
```

### 2. UseCase 단위 테스트 — InMemory Adapter 활용

```java
// PlaceOrderService 테스트 (Spring/JPA/Kafka 없음)
class PlaceOrderServiceTest {

    // InMemory Adapter 인스턴스
    private final InMemoryOrderRepository orderRepository = new InMemoryOrderRepository();
    private final InMemoryPaymentPort paymentPort = new InMemoryPaymentPort();
    private final InMemoryOrderEventPublisher eventPublisher = new InMemoryOrderEventPublisher();

    // 테스트 대상 (직접 생성, Spring DI 없음)
    private final PlaceOrderService sut = new PlaceOrderService(
        orderRepository, paymentPort, eventPublisher
    );

    @BeforeEach
    void setUp() {
        orderRepository.clear();
        paymentPort.reset();
        eventPublisher.clear();
    }

    @Test
    void 정상_주문_생성() {
        // given
        PlaceOrderCommand command = PlaceOrderCommand.of(
            UserId.of("user-001"),
            List.of(OrderLineCommand.of(ItemId.of("item-1"), 2))
        );

        // when
        OrderId orderId = sut.placeOrder(command);

        // then: 상태(State) 기반 검증 — 구현 무관
        assertThat(orderId).isNotNull();
        assertThat(orderRepository.findById(orderId)).isPresent();
        assertThat(orderRepository.findById(orderId).get().getStatus())
            .isEqualTo(OrderStatus.PAYMENT_CONFIRMED);
        assertThat(paymentPort.chargeCallCount()).isEqualTo(1);
        assertThat(eventPublisher.publishedEvents()).hasSize(1);
        assertThat(eventPublisher.publishedEvents().get(0))
            .isInstanceOf(OrderPlacedEvent.class);
    }

    @Test
    void 결제_실패_시_주문_저장_안_됨() {
        paymentPort.willFail(new PaymentFailedException("카드 한도 초과"));

        assertThatThrownBy(() -> sut.placeOrder(validCommand()))
            .isInstanceOf(PaymentFailedException.class);

        // 결제 실패 시 주문이 저장되지 않아야 함
        assertThat(orderRepository.count()).isEqualTo(0);
    }

    @Test
    void 최소_금액_미달_시_결제_호출_안_됨() {
        PlaceOrderCommand lowAmountCommand = PlaceOrderCommand.of(
            UserId.of("user-001"),
            List.of(OrderLineCommand.of(ItemId.of("item-1"), 1, Money.of(500)))
        );

        assertThatThrownBy(() -> sut.placeOrder(lowAmountCommand))
            .isInstanceOf(MinOrderAmountException.class);

        // 비즈니스 규칙 위반 시 결제 API가 호출되면 안 됨
        assertThat(paymentPort.chargeCallCount()).isEqualTo(0);
    }
    // 실행 시간: ~0.01초/테스트, Spring/JPA/Kafka 없음
}

// === InMemory Adapter 구현 ===

class InMemoryOrderRepository implements OrderSavePort, OrderQueryPort {
    private final Map<OrderId, Order> store = new LinkedHashMap<>();

    @Override
    public void save(Order order) { store.put(order.getId(), order); }

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<Order> findByUserId(UserId userId) {
        return store.values().stream()
            .filter(o -> o.getUserId().equals(userId))
            .toList();
    }

    // 테스트 헬퍼
    public int count() { return store.size(); }
    public void clear() { store.clear(); }
}

class InMemoryPaymentPort implements PaymentPort {
    private int chargeCount = 0;
    private RuntimeException failWith = null;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        if (failWith != null) throw failWith;
        chargeCount++;
        return PaymentResult.success("tx-" + UUID.randomUUID());
    }

    public void willFail(RuntimeException ex) { this.failWith = ex; }
    public int chargeCallCount() { return chargeCount; }
    public void reset() { chargeCount = 0; failWith = null; }
}

class InMemoryOrderEventPublisher implements OrderEventPublisher {
    private final List<Object> events = new ArrayList<>();

    @Override
    public void publish(OrderPlacedEvent event) { events.add(event); }

    public List<Object> publishedEvents() { return Collections.unmodifiableList(events); }
    public void clear() { events.clear(); }
}
```

### 3. Adapter 통합 테스트 — 각 Adapter 독립 검증

```java
// JPA Adapter 통합 테스트 (@DataJpaTest)
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class JpaOrderRepositoryTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb").withUsername("test").withPassword("test");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired SpringDataOrderJpaRepository jpaRepo;
    private JpaOrderRepository repository;

    @BeforeEach
    void setUp() {
        repository = new JpaOrderRepository(jpaRepo, new OrderPersistenceMapper());
    }

    @Test
    void 저장_후_동일_ID로_조회() {
        Order order = Order.create(UserId.of("u1"), validLines());
        repository.save(order);

        Optional<Order> found = repository.findById(order.getId());

        assertThat(found).isPresent();
        assertThat(found.get().getUserId()).isEqualTo(UserId.of("u1"));
        assertThat(found.get().getStatus()).isEqualTo(OrderStatus.DRAFT);
    }

    @Test
    void 없는_ID_조회_시_empty_반환() {
        Optional<Order> found = repository.findById(OrderId.of("non-exist"));
        assertThat(found).isEmpty(); // null 반환 아님 (LSP 준수 검증)
    }
}

// KakaoPay Adapter 통합 테스트 (MockWebServer)
class KakaoPayAdapterTest {

    private MockWebServer mockWebServer;
    private KakaoPayAdapter adapter;

    @BeforeEach
    void setUp() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();

        KakaoPayClient client = new KakaoPayClient(
            mockWebServer.url("/").toString()
        );
        adapter = new KakaoPayAdapter(client);
    }

    @AfterEach
    void tearDown() throws IOException {
        mockWebServer.shutdown();
    }

    @Test
    void 결제_성공_시_트랜잭션_ID_반환() throws InterruptedException {
        mockWebServer.enqueue(new MockResponse()
            .setBody("""{"transactionId": "kakao-tx-001", "status": "SUCCESS"}""")
            .addHeader("Content-Type", "application/json")
        );

        PaymentResult result = adapter.charge(
            PaymentRequest.of(Money.of(10_000), "payment-key-001")
        );

        assertThat(result.getTransactionId()).isEqualTo("kakao-tx-001");

        RecordedRequest request = mockWebServer.takeRequest();
        assertThat(request.getMethod()).isEqualTo("POST");
        assertThat(request.getPath()).isEqualTo("/v1/payment/charge");
    }

    @Test
    void 외부_API_오류_시_PaymentFailedException_발생() {
        mockWebServer.enqueue(new MockResponse().setResponseCode(500));

        assertThatThrownBy(() ->
            adapter.charge(PaymentRequest.of(Money.of(10_000), "key"))
        ).isInstanceOf(PaymentFailedException.class);
    }
}
```

### 4. Controller 테스트 — @WebMvcTest

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean PlaceOrderUseCase placeOrderUseCase; // Driving Port만 Mock
    @MockBean FindOrderQuery findOrderQuery;

    @Test
    void POST_주문_생성_201_반환() throws Exception {
        given(placeOrderUseCase.placeOrder(any()))
            .willReturn(OrderId.of("order-001"));

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .header("Authorization", "Bearer test-token")
                .content("""
                    {
                        "userId": "user-001",
                        "items": [{"itemId": "item-1", "quantity": 2}]
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", containsString("/api/orders/order-001")))
            .andExpect(jsonPath("$.orderId").value("order-001"));
    }

    @Test
    void 유효성_검증_실패_시_400_반환() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"userId": "", "items": []}""")) // 빈 값
            .andExpect(status().isBadRequest());

        verify(placeOrderUseCase, never()).placeOrder(any()); // UseCase 호출 안 됨
    }
    // 실행 시간: ~2초, JPA/Kafka/외부API 없음
}
```

---

## 💻 실전 코드 — 계층별 테스트 파일 구조

```
테스트 파일 구조:

src/test/java/com/example/
  ├── domain/                          ← Domain 단위 테스트
  │     ├── OrderTest.java             (0.001초)
  │     ├── MoneyTest.java             (0.001초)
  │     └── OrderLineTest.java         (0.001초)
  ├── application/                     ← UseCase 단위 테스트 (InMemory)
  │     ├── PlaceOrderServiceTest.java  (0.01초)
  │     ├── CancelOrderServiceTest.java (0.01초)
  │     └── support/                   ← 테스트 지원 클래스
  │           ├── InMemoryOrderRepository.java
  │           ├── InMemoryPaymentPort.java
  │           └── InMemoryOrderEventPublisher.java
  ├── adapter/                         ← Adapter 통합 테스트
  │     ├── persistence/
  │     │     └── JpaOrderRepositoryTest.java   (5-10초, TestContainers)
  │     ├── payment/
  │     │     └── KakaoPayAdapterTest.java       (2-3초, MockWebServer)
  │     └── web/
  │           └── OrderControllerTest.java       (2-3초, @WebMvcTest)
  └── e2e/                             ← E2E 테스트 (소수)
        └── PlaceOrderE2ETest.java     (30초, @SpringBootTest)

// 공용 테스트 픽스처
class OrderFixture {
    public static PlaceOrderCommand validCommand() {
        return PlaceOrderCommand.of(
            UserId.of("user-001"),
            List.of(OrderLineCommand.of(ItemId.of("item-1"), 2))
        );
    }

    public static Order validOrder() {
        return Order.create(UserId.of("user-001"),
            List.of(OrderLine.of(ItemId.of("item-1"), 2, Money.of(5_000))));
    }
}
```

---

## 📊 패턴 비교 — 레이어드 vs Hexagonal 테스트 속도

```
100개 테스트 기준 실행 시간 비교:

=== 레이어드 아키텍처 ===
  @SpringBootTest:  85개 × 30초 = 2,550초 (42분 30초)
  @DataJpaTest:     10개 × 8초  = 80초
  단위 테스트:        5개 × 0.1초 = 0.5초
  총계: 2,630초 (43분 50초)

=== Hexagonal Architecture ===
  Domain 단위:      50개 × 0.001초 = 0.05초
  UseCase 단위:     30개 × 0.01초  = 0.3초
  Adapter 통합:     15개 × 5초     = 75초
  E2E (@SpringBootTest): 5개 × 30초 = 150초
  총계: 225초 (3분 45초)

속도 향상: 43분 50초 → 3분 45초 = 약 12배 빠름

개발 사이클에 미치는 영향:
  레이어드: 코드 수정 → 테스트 실행 (43분 대기) → 피드백
  Hexagonal: 코드 수정 → 테스트 실행 (4분 이내) → 피드백
  → TDD 실천 가능 여부가 달라짐
  → CI 파이프라인 시간 단축 → PR 피드백 빨라짐
```

---

## ⚖️ 트레이드오프

```
InMemory Adapter의 관리 비용:
  InMemoryOrderRepository, InMemoryPaymentPort 등 직접 작성 필요
  실제 DB 동작과 완전히 일치하지 않음 (LAZY 로딩, 트랜잭션 등)
  → InMemory 테스트 통과 후 Adapter 통합 테스트도 필요

테스트 격리 수준 결정:
  UseCase 테스트에서 InMemory가 적합한 경우:
    비즈니스 로직 검증 (최소 금액, 상태 전이)
    결제 실패 시나리오 (실제 외부 API 없이)

  Adapter 통합 테스트가 필요한 경우:
    JPA 쿼리 결과 검증
    DB 제약 조건 검증
    외부 API 연동 검증 (MockWebServer)

적절한 균형:
  비즈니스 로직 → InMemory로 빠르게 검증 (70%)
  인프라 연동 → Adapter 테스트로 검증 (25%)
  전체 플로우 → E2E로 검증 (5%)
```

---

## 📌 핵심 정리

```
Hexagonal 테스트 전략:

1. Domain 단위 테스트 (50%)
   대상: Order, Money 등 순수 도메인 객체
   방법: new Order(), 순수 Java
   속도: ~0.001초

2. UseCase 단위 테스트 (30%)
   대상: PlaceOrderService 등 UseCase 구현
   방법: InMemory Adapter 주입, 직접 new PlaceOrderService(...)
   속도: ~0.01초

3. Adapter 통합 테스트 (15%)
   대상: JpaOrderRepository, KakaoPayAdapter
   방법: @DataJpaTest, TestContainers, MockWebServer
   속도: ~5초

4. E2E 테스트 (5%)
   대상: HTTP → Domain → DB 전체 플로우
   방법: @SpringBootTest + TestContainers
   속도: ~30초

핵심 원칙:
  UseCase 테스트는 InMemory Adapter로 → Spring 없이
  상태(State) 기반 검증 → 구현 변경에 견고한 테스트
  행위(Behavior) 검증은 외부 시스템 호출 여부 확인에만
```

---

## 🤔 생각해볼 문제

**Q1.** `verify(orderRepository, times(1)).save(any())` 같은 행위 기반 검증과 `assertThat(repository.findById(id)).isPresent()` 같은 상태 기반 검증 중 어느 것이 더 견고한 테스트인가?

<details>
<summary>해설 보기</summary>

**상태 기반 검증이 더 견고합니다.**

행위 기반 검증의 문제:
```java
verify(orderRepository, times(1)).save(any());
// PlaceOrderService 내부 구현이
// repository.save(order) → repository.saveAll(List.of(order))로 바뀌면
// 비즈니스 동작은 동일하지만 테스트 실패!
// → 테스트가 구현 세부사항에 결합됨
```

상태 기반 검증의 장점:
```java
assertThat(repository.findById(orderId)).isPresent();
assertThat(repository.findById(orderId).get().getStatus()).isEqualTo(OrderStatus.PLACED);
// "주문이 올바른 상태로 저장됐는가"를 검증
// save() vs saveAll() 여부에 무관 → 구현 변경에 견고
```

행위 검증이 적합한 경우: 외부 시스템 호출 여부 확인
```java
// "최소 금액 미달 시 결제 API가 호출되지 않아야 한다" → 행위 검증 적합
assertThat(paymentPort.chargeCallCount()).isEqualTo(0);
// "결제가 호출됐는가"라는 부작용(Side Effect)이 중요한 경우
```

</details>

---

**Q2.** `@SpringBootTest`의 비율을 5%로 줄이면 충분한가? 나머지 95% 테스트가 통과해도 실제 운영에서 문제가 발생할 수 있는 경우는?

<details>
<summary>해설 보기</summary>

**네, 실제 운영 문제가 발생할 수 있습니다.** InMemory 테스트가 놓치는 것들:

1. **DB 제약 조건**: InMemory는 DB의 unique constraint, foreign key를 모름
   → InMemory 테스트 통과, DB에서 ConstraintViolationException 발생

2. **LAZY 로딩 예외**: InMemory는 항상 즉시 로딩
   → InMemory 통과, JPA에서 LazyInitializationException 발생

3. **트랜잭션 격리 수준**: InMemory는 트랜잭션 개념 없음
   → 동시성 문제가 InMemory 테스트에서 재현 안 됨

4. **실제 외부 API 응답 형식**: MockWebServer는 미리 설정한 응답만
   → 실제 KakaoPay API 응답 형식이 다르면 발견 못함

방지 전략:
- DB 관련: @DataJpaTest or TestContainers로 Adapter 통합 테스트 필수
- 외부 API: 실제 Sandbox 환경으로 주기적 통합 테스트
- E2E: 핵심 Critical Path는 @SpringBootTest로 검증

</details>

---

**Q3.** TDD(Test-Driven Development)를 Hexagonal Architecture와 함께 적용하면 어떤 순서로 테스트를 작성해야 하는가?

<details>
<summary>해설 보기</summary>

**안에서 밖으로(Inside-Out)** 방향이 자연스럽습니다.

```
1단계: Domain 테스트 먼저 (Red-Green-Refactor)
   // Red: 실패하는 테스트
   @Test void 최소_금액_미달_시_예외() {
       Order order = Order.create(userId, lowAmountLines());
       assertThatThrownBy(order::place)
           .isInstanceOf(MinOrderAmountException.class);
   }
   // Green: Order.place() 구현
   // Refactor: 코드 개선

2단계: UseCase 테스트 (InMemory Adapter 사용)
   // Red: 실패하는 UseCase 테스트
   @Test void 주문_생성_성공() {
       OrderId id = sut.placeOrder(validCommand());
       assertThat(orderRepository.findById(id)).isPresent();
   }
   // Green: PlaceOrderService 구현
   // Refactor

3단계: Adapter 테스트 (마지막에)
   // Red: JpaOrderRepository 테스트
   @Test void DB_저장_후_조회() { ... }
   // Green: JpaOrderRepository 구현
   // Refactor

4단계: E2E 테스트 (가장 마지막)
   // 전체 플로우 검증

이 순서의 이점:
  Domain 로직을 가장 먼저 설계 → 비즈니스 규칙이 명확해짐
  InMemory로 빠른 피드백 → TDD 리듬 유지 가능
  Adapter는 Port 계약이 확정된 후 구현
```

</details>

---

<div align="center">

**[⬅️ 이전: Spring으로 Hexagonal 구현](./05-hexagonal-with-spring.md)** | **[홈으로 🏠](../README.md)** | **[다음: Hexagonal의 실제 비용 ➡️](./07-hexagonal-real-costs.md)**

</div>
