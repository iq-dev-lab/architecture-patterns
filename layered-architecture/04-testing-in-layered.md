# 레이어드 아키텍처에서 테스트 — Spring Context가 항상 필요한 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Service가 JpaRepository에 직접 의존할 때 단위 테스트에 DB가 필요해지는 정확한 이유는?
- `@SpringBootTest`, `@DataJpaTest`, `Mockito` 각각의 트레이드오프는 무엇인가?
- Mocking의 한계 — "내부 구현에 결합된 Mock"이 어떤 문제를 일으키는가?
- 아키텍처 변경이 테스트 속도에 미치는 실제 영향은 어느 정도인가?
- 테스트 피라미드와 레이어드 아키텍처의 관계는 무엇인가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

"테스트를 짜면 되는 거 아닌가요?" — 물론이다. 하지만 아키텍처가 테스트 비용을 결정한다.

Service가 JpaRepository에 직접 의존하면, 그 Service의 단위 테스트는 JPA와 DB 없이는 불가능하다. 100개 테스트가 모두 `@SpringBootTest`라면 전체 테스트 스위트 실행에 5분이 걸린다. 이것이 아키텍처 결정이 테스트에 미치는 직접적 영향이다.

테스트가 느리면 → 자주 실행하지 않게 됨 → 문제를 늦게 발견 → 수정 비용 증가.

---

## 😱 흔한 실수 (Before — 테스트 없음 또는 전부 통합 테스트)

```
상황 1: 테스트 자체가 없는 경우
  이유: "OrderService 테스트 짜려면 @SpringBootTest해야 하는데
         너무 느려서 그냥 수동 테스트로 대체"
  결과:
    기능 추가 시마다 전체 수동 테스트
    회귀 버그를 배포 후에 발견
    리팩터링이 무서워서 안 함

상황 2: 모두 @SpringBootTest
  @SpringBootTest
  class OrderServiceTest {
      @Autowired OrderService orderService;

      @Test
      void 주문_생성_테스트() {
          // 전체 Spring Context 로딩 (30초)
          // DB 연결 필요
          // Kafka 연결 필요
          // 외부 API Mock 서버 필요
          // ...
      }
  }
  
  결과:
    테스트 100개 × 30초 = 50분
    CI 파이프라인이 1시간 걸림
    개발자가 Push 전에 테스트를 실행하지 않게 됨

상황 3: Mock 지옥
  @ExtendWith(MockitoExtension.class)
  class OrderServiceTest {
      @Mock OrderJpaRepository orderRepository;
      @Mock UserService userService;
      @Mock KakaoPayClient paymentClient;
      @Mock JavaMailSender mailSender;
      @Mock KafkaTemplate kafkaTemplate;
      @Mock InventoryService inventoryService;
      @Mock CouponRepository couponRepository;

      @InjectMocks OrderService orderService;

      @BeforeEach
      void setUp() {
          // 모든 Mock 설정 (30줄)
          given(userService.findById(anyLong())).willReturn(mockUser());
          given(inventoryService.checkStock(anyList())).willReturn(true);
          given(paymentClient.charge(any())).willReturn(mockPaymentResult());
          // ...
      }

      @Test
      void 최소금액_미달_테스트() {
          // 실제 테스트: 3줄
          // Mock 설정: 30줄
          // 테스트가 구현 세부사항에 결합됨
      }
  }
```

---

## ✨ 올바른 접근 (After — 아키텍처 개선으로 테스트 속도와 격리성 확보)

```
테스트 피라미드 목표:
  단위 테스트 (빠름, 많음): 70%
  통합 테스트 (중간): 20%
  E2E 테스트 (느림, 적음): 10%

아키텍처 개선 후 달성 가능한 것:
  Domain 단위 테스트: Spring 없음, DB 없음, ~0.01초
  Application 단위 테스트: InMemory Adapter, ~0.05초
  Adapter 통합 테스트: @DataJpaTest 또는 TestContainers, ~5초
  E2E 테스트: @SpringBootTest + TestContainers, ~30초

100개 테스트 기준:
  레이어드 (@SpringBootTest): 100 × 30초 = 50분
  개선된 구조: (70 × 0.01초) + (20 × 5초) + (10 × 30초) = ~401초 ≈ 7분
```

---

## 🔬 내부 원리 — 테스트 어려움의 메커니즘

### 1. 의존성 그래프가 테스트 비용을 결정하는 방법

```
의존성과 테스트 비용의 관계:

테스트하려는 것: OrderService.placeOrder()의 "최소 금액 미달 시 예외 발생"

테스트 비용 = 의존성을 준비하는 비용

OrderService의 의존성 그래프:
  OrderService
    ├── OrderJpaRepository (JPA 필요)
    │     └── DataSource (DB 연결 필요)
    ├── UserService
    │     └── UserJpaRepository (JPA 필요)
    ├── KakaoPayClient (외부 API)
    ├── JavaMailSender (SMTP 서버)
    ├── KafkaTemplate (Kafka 브로커)
    └── CouponRepository (JPA 필요)

최소 금액 검증 하나를 테스트하기 위해:
  JPA + DB + 외부 API + SMTP + Kafka 전부 필요하거나
  전부 Mock해야 함

반면, 비즈니스 로직이 Domain 객체에 있으면:
  Order.place() 테스트:
    Order order = Order.create(userId, lines); // 순수 Java
    assertThatThrownBy(() -> order.place())
        .isInstanceOf(MinOrderAmountException.class);
  의존성: 없음, 실행 시간: ~0.01초
```

### 2. 세 가지 테스트 접근 방식의 트레이드오프

```
방법 1: @SpringBootTest

@SpringBootTest
class OrderServiceTest {
    @Autowired OrderService orderService;
}

장점:
  실제 Spring Context, 실제 Bean 주입
  통합 환경을 가장 잘 반영
  설정이 간단 (Spring이 다 해줌)

단점:
  시작 시간: 20-40초 (전체 컨텍스트 로딩)
  DB 필요 (H2 또는 TestContainers)
  외부 의존성 필요 (Kafka, Redis 등)
  단위 테스트로 부적합 (너무 느림)

적합한 경우:
  E2E 시나리오 테스트
  Controller → Service → Repository 전체 흐름 검증
  실제 환경과 유사한 통합 테스트

방법 2: @DataJpaTest

@DataJpaTest
class OrderRepositoryTest {
    @Autowired TestEntityManager em;
    @Autowired OrderJpaRepository repository;
}

장점:
  JPA 레이어만 로딩 (전체보다 빠름, ~5-10초)
  H2 인메모리 DB 자동 설정
  Repository 계층 단독 테스트에 적합

단점:
  Service, Controller는 로딩하지 않음
  오직 JPA Repository 테스트에 한정
  H2와 실제 DB의 방언 차이 (MySQL 전용 쿼리 등)

적합한 경우:
  JPA Repository의 커스텀 쿼리 검증
  @Query 결과 검증
  연관 관계 설정 검증

방법 3: Mockito (@ExtendWith(MockitoExtension.class))

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository orderRepository;
    @InjectMocks OrderService orderService;
}

장점:
  Spring Context 없음 → 빠름 (~0.1초)
  의존성을 모두 Mock으로 제어 가능

단점:
  Mock이 실제 구현체와 다를 수 있음 (구현 세부사항 결합)
  의존성이 많으면 Mock 설정 코드가 방대해짐
  "의도한 대로 Mock했지만 실제는 다르게 동작" 위험

적합한 경우:
  의존성이 적은 Service의 흐름 테스트
  외부 시스템(API, 이메일) 격리 테스트
```

### 3. Mock의 한계 — 내부 구현에 결합되는 문제

```
Mock이 내부 구현에 결합되는 예시:

@Test
void 주문_생성_성공() {
    // given
    given(orderRepository.save(any(Order.class)))
        .willReturn(savedOrder);
    given(paymentPort.charge(any()))
        .willReturn(PaymentResult.success("tx-001"));

    // when
    orderService.placeOrder(command);

    // then
    verify(orderRepository, times(1)).save(any(Order.class)); // ← 구현 세부사항 검증
    verify(paymentPort, times(1)).charge(any());              // ← 구현 세부사항 검증
}

문제:
  이 테스트는 "주문이 성공적으로 생성됐는가?"가 아니라
  "orderRepository.save()가 정확히 1번 호출됐는가?"를 검증
  
  만약 내부 구현이 바뀌어서:
    save() → saveAll(List.of(order))로 변경
    → 비즈니스 동작은 동일하지만 테스트 실패!
    → 테스트가 구현 세부사항에 결합됨

올바른 테스트 (상태 기반 검증):
  InMemoryOrderRepository 사용:

  @Test
  void 주문_생성_성공() {
      // given
      InMemoryOrderRepository repository = new InMemoryOrderRepository();
      PaymentPort paymentPort = cmd -> PaymentResult.success("tx-001");
      PlaceOrderService service = new PlaceOrderService(repository, paymentPort, ...);

      // when
      OrderId orderId = service.placeOrder(command);

      // then: 상태(State)로 검증 — 구현 세부사항 무관
      assertThat(repository.findById(orderId)).isPresent(); // 저장됐는가?
      assertThat(repository.findById(orderId).get().getStatus())
          .isEqualTo(OrderStatus.PLACED); // 올바른 상태인가?
  }

  내부 구현(save vs saveAll)이 바뀌어도 테스트는 통과
  "주문이 저장됐는가"라는 비즈니스 결과를 검증하기 때문
```

### 4. TestContainers — 실제 DB로 빠른 통합 테스트

```java
// TestContainers: 테스트 전용 Docker 컨테이너 실행

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderJpaRepositoryTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    JpaOrderRepository orderRepository;

    @Test
    void 주문_저장_후_조회() {
        // H2 방언 문제 없음, 실제 MySQL로 테스트
        Order order = Order.create(UserId.of("user-1"), lines);
        orderRepository.save(order);

        Optional<Order> found = orderRepository.findById(order.getId());
        assertThat(found).isPresent();
    }
}
// 장점: 실제 DB 방언, H2 차이 없음
// 비용: 컨테이너 시작 시간 (~5초), @DataJpaTest보다 느림
// 최초 실행 후 컨테이너 재사용으로 속도 개선 가능 (Singleton Pattern)
```

### 5. 아키텍처별 테스트 전략 매핑

```
레이어드 아키텍처에서의 테스트 전략:

  Presentation Layer (Controller):
    사용: @WebMvcTest (Controller만 로딩)
    속도: ~2-3초
    범위: HTTP 요청/응답, 파라미터 바인딩, 예외 처리

  Application Layer (Service):
    레이어드: @SpringBootTest 또는 Mockito (구현에 결합)
    개선 후: 순수 Java + InMemory Adapter (~0.01초)
    범위: 유스케이스 흐름, 도메인 조율

  Domain Layer (Entity, Domain Service):
    레이어드: 방법 없음 (Entity에 로직이 없음, Anemic)
    개선 후: 순수 Java 단위 테스트 (~0.01초)
    범위: 비즈니스 규칙, 상태 전이

  Infrastructure Layer (Repository):
    사용: @DataJpaTest 또는 TestContainers
    속도: 5-10초
    범위: 쿼리 결과, 매핑, DB 제약조건
```

---

## 💻 실전 코드 — 테스트 접근 방식 비교

```java
// === @SpringBootTest (통합 테스트) ===
@SpringBootTest
@Transactional
class OrderServiceIntegrationTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepository;

    @Test
    void 주문_생성_통합_테스트() {
        // 장점: 실제 환경과 동일
        // 단점: 30초 이상, DB/Kafka/외부 API 필요
        PlaceOrderCommand command = ...;
        OrderId orderId = orderService.placeOrder(command);
        assertThat(orderRepository.findById(orderId)).isPresent();
    }
}

// === @DataJpaTest (Repository 테스트) ===
@DataJpaTest
class JpaOrderRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired OrderJpaRepository jpaRepository;
    JpaOrderRepository repository;

    @BeforeEach
    void setUp() { repository = new JpaOrderRepository(jpaRepository, new OrderMapper()); }

    @Test
    void 저장_후_조회() {
        // 장점: JPA 레이어 격리 테스트
        // 단점: 5-10초, JPA 관련만 테스트 가능
        Order order = Order.create(UserId.of("u1"), lines);
        repository.save(order);
        assertThat(repository.findById(order.getId())).isPresent();
    }
}

// === 순수 Java 단위 테스트 (도메인/애플리케이션) ===
class PlaceOrderServiceTest {

    // InMemory 구현체 — Port를 구현하는 테스트 전용 클래스
    private final List<Order> savedOrders = new ArrayList<>();
    private final List<OutboxEvent> events = new ArrayList<>();

    private final OrderRepository repository = new OrderRepository() {
        public void save(Order o) { savedOrders.add(o); }
        public Optional<Order> findById(OrderId id) {
            return savedOrders.stream().filter(o -> o.getId().equals(id)).findFirst();
        }
    };

    private final PaymentPort paymentPort =
        req -> PaymentResult.success("tx-" + UUID.randomUUID());

    private final PlaceOrderService sut =
        new PlaceOrderService(repository, paymentPort);

    @Test
    void 주문_생성_성공() {
        // 0.01초, Spring/DB/Kafka 없음
        OrderId id = sut.placeOrder(validCommand());
        assertThat(savedOrders).hasSize(1);
        assertThat(savedOrders.get(0).getStatus()).isEqualTo(OrderStatus.PLACED);
    }

    @Test
    void 최소금액_미달_시_예외() {
        PlaceOrderCommand cmd = commandWithTotal(Money.of(500));
        assertThatThrownBy(() -> sut.placeOrder(cmd))
            .isInstanceOf(MinOrderAmountException.class);
        assertThat(savedOrders).isEmpty(); // 저장되지 않아야 함
    }

    @Test
    void 결제_실패_시_예외() {
        PaymentPort failingPayment = req -> { throw new PaymentFailedException(); };
        PlaceOrderService service = new PlaceOrderService(repository, failingPayment);

        assertThatThrownBy(() -> service.placeOrder(validCommand()))
            .isInstanceOf(PaymentFailedException.class);
        assertThat(savedOrders).isEmpty(); // 결제 실패 시 저장되지 않아야 함
    }
}

// === @WebMvcTest (Controller 테스트) ===
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean PlaceOrderUseCase placeOrderUseCase; // UseCase만 Mock

    @Test
    void 주문_생성_API_200_반환() throws Exception {
        // 장점: Controller 레이어만 테스트 (2-3초)
        // HTTP 요청/응답 검증에 집중
        given(placeOrderUseCase.placeOrder(any()))
            .willReturn(OrderId.of("order-001"));

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"userId": "user-1", "items": [...]}"""))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.orderId").value("order-001"));
    }
}
```

---

## 📊 패턴 비교 — 레이어드 vs 개선된 구조 테스트 속도

```
시나리오: 100개 테스트 스위트 실행 시간

=== 레이어드 아키텍처 (전형적) ===
  @SpringBootTest: 80개 × 30초 = 2,400초 (40분)
  @DataJpaTest:    15개 × 8초  = 120초
  단위 테스트:       5개 × 0.1초 = 0.5초
  총계: ~2,520초 (42분)

=== 개선된 구조 (Hexagonal 방향) ===
  순수 Java 단위:   70개 × 0.01초 = 0.7초
  @DataJpaTest:    20개 × 8초   = 160초
  @WebMvcTest:      5개 × 2초   = 10초
  @SpringBootTest:  5개 × 30초  = 150초
  총계: ~321초 (5분 21초)

차이: 42분 → 5분, 약 8배 단축

테스트가 빠를 때의 효과:
  개발자가 Push 전에 테스트 실행 (42분이면 실행 안 함)
  CI 파이프라인 단축 (PR 피드백이 5분 vs 42분)
  TDD 가능성 (테스트 먼저 짜는 것이 현실적)
```

---

## ⚖️ 트레이드오프

```
테스트 속도 vs 실제 환경 반영:
  순수 Java 단위 테스트: 빠름, 격리됨, 하지만 실제 DB/인프라와 다를 수 있음
  @SpringBootTest 통합 테스트: 느림, 실제 환경에 가까움

현실적 전략:
  비즈니스 규칙 검증 → 순수 Java 단위 테스트 (빠름이 중요)
  DB 쿼리 검증 → @DataJpaTest 또는 TestContainers
  API 계약 검증 → @WebMvcTest
  중요 E2E 시나리오 → @SpringBootTest (소수만)

InMemory Adapter의 한계:
  InMemoryOrderRepository는 실제 JPA의 트랜잭션, flush, lazy loading과 다름
  → InMemory로 통과해도 실제 DB에서 실패할 수 있음
  → InMemory + @DataJpaTest 두 가지 모두 필요
```

---

## 📌 핵심 정리

```
레이어드 아키텍처에서 테스트 어려움의 원인:

의존성 그래프 = 테스트 준비 비용
  Service → JpaRepository → DB → 테스트에 DB 필요
  Service → ExternalAPI → 테스트에 Mock 또는 서버 필요

세 가지 접근 방식:
  @SpringBootTest: 실제 환경 반영, 느림 (30초+)
  @DataJpaTest:    Repository 격리, 중간 (5-10초)
  Mockito:         빠름, 구현 결합 위험

Mock의 핵심 함정:
  행위(호출 횟수) 검증 → 구현 세부사항 결합 → 취약한 테스트
  상태(결과 값) 검증 → 구현 무관 → 견고한 테스트

아키텍처 개선 효과:
  Domain → 순수 Java 단위 테스트 (0.01초)
  Application → InMemory Adapter (0.05초)
  100개 기준: 42분 → 5분
```

---

## 🤔 생각해볼 문제

**Q1.** Mock(Mockito) 기반 테스트와 InMemory 구현체 기반 테스트는 어떤 차이가 있는가? 언제 Mock을 사용하고 언제 InMemory를 사용해야 하는가?

<details>
<summary>해설 보기</summary>

핵심 차이는 **"구현에 결합되는가"** 입니다.

```java
// Mock 기반: 행위 검증 (구현 결합)
given(orderRepository.save(any())).willReturn(savedOrder);
// ...
verify(orderRepository).save(any()); // save()가 호출됐는가? → 구현 세부사항

// InMemory 기반: 상태 검증 (구현 독립)
InMemoryOrderRepository repo = new InMemoryOrderRepository();
// ...
assertThat(repo.findById(orderId)).isPresent(); // 저장 결과가 있는가? → 비즈니스 결과
```

**Mock을 사용해야 할 때:**
- 외부 시스템(결제 API, 이메일 서버)처럼 실제 구현이 없거나 만들기 어려울 때
- 특정 실패 시나리오를 재현할 때 (`given(api.call()).willThrow(...)`)

**InMemory를 사용해야 할 때:**
- Repository, EventPublisher 등 도메인 Port를 테스트할 때
- 상태 기반 검증이 자연스러운 경우
- 테스트가 구현 변경에 깨지지 않아야 할 때

</details>

---

**Q2.** `@DataJpaTest`가 H2 인메모리 DB를 사용할 때의 위험성은 무엇인가? 어떻게 해결할 수 있는가?

<details>
<summary>해설 보기</summary>

H2와 실제 DB(MySQL, PostgreSQL)의 차이로 **"테스트는 통과하지만 운영에서 실패"** 가 발생합니다.

대표적인 H2 차이점:
```sql
-- MySQL 전용 함수 (H2에서 실패)
SELECT DATE_FORMAT(created_at, '%Y-%m') FROM orders;

-- MySQL 전용 힌트 (H2에서 실패)
SELECT * FROM orders USE INDEX (idx_user_id);

-- 대소문자 구분 (MySQL 기본: 구분 없음, H2: 구분 있음)
SELECT * FROM ORDERS; -- MySQL에서 동작, H2에서도 동작하지만 차이 있음
```

해결 방법 1: TestContainers (권장)
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class OrderRepositoryTest {
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");
    // 실제 MySQL로 테스트 → H2 방언 차이 없음
}
```

해결 방법 2: H2 방언을 MySQL 호환 모드로
```yaml
spring.datasource.url: jdbc:h2:mem:testdb;MODE=MYSQL
```
완전한 호환은 아니지만 대부분의 차이 해소 가능.

</details>

---

**Q3.** 테스트 피라미드에서 단위 테스트:통합 테스트:E2E 테스트의 비율은 보통 70:20:10으로 권장된다. 레이어드 아키텍처에서 이 비율을 유지하기 어려운 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

레이어드 아키텍처에서 **단위 테스트로 검증할 수 있는 비즈니스 로직이 없기 때문**입니다.

```
레이어드 아키텍처의 현실:
  Domain 객체 (Order): getter/setter만 → 단위 테스트할 로직 없음
  Service: JpaRepository 의존 → 단위 테스트에 DB 필요
  Repository: JPA 기능 → @DataJpaTest 필요

  결과: "단위 테스트"가 실질적으로 @SpringBootTest 통합 테스트
  비율이 역전: E2E 80%, 통합 15%, 단위 5%

개선된 구조에서의 비율:
  Domain 객체: 순수 Java → 단위 테스트 70% 달성 가능
  Application: InMemory Adapter → 단위 테스트로 분류 가능
  Infrastructure: @DataJpaTest → 통합 테스트 20%
  E2E: @SpringBootTest 소수 → E2E 10%
```

핵심: 테스트 피라미드는 단위 테스트가 **단독으로 실행 가능한 빠른 테스트**를 의미합니다. 비즈니스 로직이 Domain 객체에 있어야 단위 테스트가 의미있어집니다.

</details>

---

<div align="center">

**[⬅️ 이전: 레이어드 아키텍처의 함정](./03-layered-architecture-traps.md)** | **[홈으로 🏠](../README.md)** | **[다음: 레이어드 아키텍처 개선 — DIP로 의존성 역전 ➡️](./05-improving-with-dip.md)**

</div>
