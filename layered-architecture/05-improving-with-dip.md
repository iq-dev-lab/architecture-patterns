# 레이어드 아키텍처 개선 — DIP로 의존성 역전

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Repository 인터페이스를 Business 레이어로 올린다는 것은 구체적으로 어떤 의미인가?
- DIP 적용 전후 의존성 다이어그램은 어떻게 달라지는가?
- 인터페이스 도입 후 InMemory 구현체로 빠른 단위 테스트가 가능해지는 이유는?
- 이 개선이 Hexagonal Architecture와 어떻게 자연스럽게 연결되는가?
- 기존 레이어드 코드에 DIP를 점진적으로 적용하는 순서는?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

레이어드 아키텍처의 구조적 문제(의존성이 DB 방향으로 흐름)를 Hexagonal Architecture로 전환하지 않고도 부분적으로 해결하는 가장 현실적인 방법이 DIP 적용이다.

Repository 인터페이스 하나를 올바르게 위치시키는 것만으로도 Service의 단위 테스트가 가능해지고, 이후 Hexagonal Architecture로의 전환 경로가 자연스럽게 열린다. 이것이 "점진적 아키텍처 개선"의 시작점이다.

---

## 😱 흔한 실수 (Before — DIP 없는 레이어드 아키텍처)

```
전형적인 구조 (DIP 없음):

  [Presentation]   OrderController
                       │ @Autowired
                       ↓
  [Business]       OrderService
                       │ @Autowired
                       ↓
  [Data Access]    OrderJpaRepository (implements JpaRepository<Order, Long>)
                       │
                       ↓
                   DB

의존성 방향: 위에서 아래로, DB 방향으로 흐름

문제:
  ① Service가 JpaRepository(인프라)를 직접 앎
     → JPA → MongoDB 교체 시 Service 코드 수정 필요
  
  ② Service 단위 테스트에 JPA 필요
     @ExtendWith(MockitoExtension.class)
     class OrderServiceTest {
         @Mock
         OrderJpaRepository repository; // JPA 구현체를 Mock
         // → Service가 JPA 메서드(saveAndFlush, findByIdWithLock 등)에 결합됨
     }

  ③ 인터페이스가 없어서 InMemory 구현체를 만들 수 없음
     → 빠른 단위 테스트 불가능
```

---

## ✨ 올바른 접근 (After — DIP 적용 후)

```
DIP 적용 후 구조:

  [Presentation]   OrderController
                       │ depends on
                       ↓
  [Business]       OrderService
                       │ depends on (인터페이스에만)
                       ↓
                   OrderRepository ← [Business Layer에 위치]
                       ↑ implements
  [Data Access]    JpaOrderRepository
                       │
                       ↓
                   DB

의존성 방향: 인프라가 Business Layer의 인터페이스를 구현
            = 의존성 역전!

이익:
  ① Service가 OrderRepository(인터페이스)만 앎
     → JPA → MongoDB 교체: JpaOrderRepository만 교체, Service 변경 없음

  ② Service 단위 테스트에 InMemory 구현체 사용 가능
     OrderRepository repo = new InMemoryOrderRepository();
     OrderService service = new OrderService(repo, ...);
     // JPA 없이 테스트 가능!

  ③ 이 구조가 Hexagonal Architecture의 Driven Port와 동일
     → Hexagonal로의 자연스러운 전환 경로
```

---

## 🔬 내부 원리 — DIP가 의존성 방향을 역전시키는 메커니즘

### 1. DIP 적용 전후 의존성 다이어그램

```
=== DIP 적용 전 ===

  OrderService ──depends on──→ OrderJpaRepository (구체 클래스)
                                        │
                                        ↓
                                    Spring Data JPA
                                        │
                                        ↓
                                       DB

화살표 방향: Business → Infrastructure (인프라 방향)
문제: 인프라가 바뀌면 Business도 바꿔야 함

=== DIP 적용 후 ===

  OrderService ──depends on──→ OrderRepository (인터페이스) ← Business Layer에 위치
                                        ↑
                               implements
                                        │
                               JpaOrderRepository (구체 클래스) ← Infrastructure Layer
                                        │
                                        ↓
                                    Spring Data JPA
                                        │
                                        ↓
                                       DB

화살표 방향:
  Business → OrderRepository (인터페이스) ← Infrastructure
  = 인프라가 Business의 인터페이스에 의존 (역전!)

핵심: OrderRepository 인터페이스가 어느 레이어에 있느냐
  잘못된 위치: Infrastructure Layer
    → OrderService(Business)가 Infrastructure Layer에 의존
    → DIP 미적용
  
  올바른 위치: Business Layer (또는 Domain Layer)
    → Infrastructure Layer가 Business Layer의 인터페이스를 구현
    → DIP 적용
```

### 2. Repository 인터페이스를 Business Layer로 올리는 방법

```
Step 1: 인터페이스 정의 (Business/Domain Layer에)

  // domain/ 패키지에 위치
  public interface OrderRepository {
      void save(Order order);
      Optional<Order> findById(OrderId id);
      List<Order> findByUserId(UserId userId);
  }

Step 2: 구현체를 Infrastructure Layer에

  // infrastructure/ 패키지에 위치
  @Repository
  public class JpaOrderRepository implements OrderRepository {

      private final SpringDataOrderJpaRepository jpa; // Spring Data JPA
      private final OrderMapper mapper;

      @Override
      public void save(Order order) {
          OrderJpaEntity entity = mapper.toJpaEntity(order);
          jpa.save(entity);
      }

      @Override
      public Optional<Order> findById(OrderId id) {
          return jpa.findById(id.value()).map(mapper::toDomain);
      }
  }

  // Spring Data JPA Repository는 Infrastructure에만 존재
  interface SpringDataOrderJpaRepository extends JpaRepository<OrderJpaEntity, Long> {}

Step 3: Service는 인터페이스에만 의존

  // business/ 또는 application/ 패키지
  @Service
  public class OrderService {

      private final OrderRepository orderRepository; // 인터페이스!

      // 생성자 주입 — 인터페이스에만 의존
      public OrderService(OrderRepository orderRepository) {
          this.orderRepository = orderRepository;
      }
  }

  // Spring이 자동으로 JpaOrderRepository를 주입
  // (OrderRepository 인터페이스의 구현체)
```

### 3. InMemory 구현체로 빠른 단위 테스트

```java
// 테스트용 InMemory 구현체 (OrderRepository 인터페이스 구현)

public class InMemoryOrderRepository implements OrderRepository {

    private final Map<OrderId, Order> store = new HashMap<>();

    @Override
    public void save(Order order) {
        store.put(order.getId(), order);
    }

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

    // 테스트용 헬퍼
    public int count() { return store.size(); }
    public void clear() { store.clear(); }
}

// 단위 테스트
class OrderServiceTest {

    private InMemoryOrderRepository orderRepository = new InMemoryOrderRepository();
    private OrderService service = new OrderService(orderRepository);

    @Test
    void 주문_저장_후_조회() {
        // Spring 없음, JPA 없음, DB 없음
        // 실행 시간: ~0.01초
        Order order = Order.create(UserId.of("user-1"), lines);
        service.saveOrder(order);

        Optional<Order> found = orderRepository.findById(order.getId());
        assertThat(found).isPresent();
        assertThat(found.get().getUserId()).isEqualTo(UserId.of("user-1"));
    }

    @Test
    void 없는_주문_조회_시_예외() {
        assertThatThrownBy(() -> service.findOrder(OrderId.of("non-exist")))
            .isInstanceOf(OrderNotFoundException.class);
    }
}
```

### 4. 외부 의존성도 인터페이스로 추출

```java
// Repository 뿐만 아니라 외부 API도 인터페이스로

// Before: Service가 SDK를 직접 의존
@Service
public class OrderService {
    @Autowired KakaoPayClient kakaoPayClient; // SDK 구체 클래스
    @Autowired JavaMailSender mailSender;      // Spring 인프라 클래스
}

// After: 인터페이스(Port)로 추상화
// Business Layer에 인터페이스 정의
public interface PaymentPort {
    PaymentResult charge(PaymentRequest request);
}

public interface NotificationPort {
    void notifyOrderPlaced(Order order);
}

// Infrastructure Layer에 구현체
@Component
public class KakaoPayAdapter implements PaymentPort {
    private final KakaoPayClient client; // SDK는 여기에만

    @Override
    public PaymentResult charge(PaymentRequest request) {
        KakaoPayResponse response = client.charge(
            KakaoPayRequest.of(request.getAmount(), request.getPaymentKey())
        );
        return PaymentResult.of(response.getTransactionId());
    }
}

// Service: 인터페이스에만 의존
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentPort paymentPort;
    private final NotificationPort notificationPort;

    // 어떤 구현체가 주입되는지 Service는 모름
    // KakaoPay → TossPay 교체: KakaoPayAdapter → TossPayAdapter, Service 변경 없음
}
```

### 5. Hexagonal Architecture로의 자연스러운 연결

```
DIP 적용 레이어드 → Hexagonal Architecture 연결:

DIP 적용 레이어드에서 이미 가지고 있는 것:
  ✅ OrderRepository (Business Layer 인터페이스)
     = Hexagonal의 Driven Port (OrderSavePort, OrderQueryPort)
  
  ✅ JpaOrderRepository (Infrastructure 구현체)
     = Hexagonal의 Driven Adapter
  
  ✅ PaymentPort (Business Layer 인터페이스)
     = Hexagonal의 Driven Port
  
  ✅ KakaoPayAdapter (Infrastructure 구현체)
     = Hexagonal의 Driven Adapter

추가로 도입하면 Hexagonal 완성:
  UseCase 인터페이스 (Driving Port)
    public interface PlaceOrderUseCase {
        OrderId placeOrder(PlaceOrderCommand command);
    }
  → Controller가 UseCase 인터페이스에 의존

  Controller (Driving Adapter)
    OrderController → PlaceOrderUseCase (인터페이스)

이것이 Hexagonal Architecture의 구조 그대로:
  Driving Adapter → Driving Port → Application Service → Driven Port ← Driven Adapter

DIP를 레이어드에 적용하는 것 = Hexagonal로 가는 중간 단계
```

---

## 💻 실전 코드 — 점진적 DIP 적용 예시

```java
// Phase 1: Repository 인터페이스를 Business Layer로 (1-2시간)

// Before
@Service
public class OrderService {
    @Autowired
    OrderJpaRepository orderRepository; // ← 구체 클래스
}

// After: 인터페이스 도입
// domain/OrderRepository.java (새 파일)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// infrastructure/JpaOrderRepository.java (기존 코드 수정)
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpa;
    @Override public void save(Order order) { jpa.save(toEntity(order)); }
    @Override public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value()).map(this::toDomain);
    }
}

// application/OrderService.java
@Service
public class OrderService {
    private final OrderRepository orderRepository; // ← 인터페이스로 변경
}
// 완료! Spring이 JpaOrderRepository를 자동 주입

// Phase 2: 외부 API 인터페이스 추출 (의존성마다 반복)

// Before
@Service
public class OrderService {
    @Autowired KakaoPayClient kakaoPayClient;
}

// After
// domain/PaymentPort.java
public interface PaymentPort {
    PaymentResult charge(PaymentRequest request);
}

// infrastructure/KakaoPayAdapter.java
@Component
public class KakaoPayAdapter implements PaymentPort {
    private final KakaoPayClient client;
    @Override public PaymentResult charge(PaymentRequest req) { ... }
}

// application/OrderService.java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentPort paymentPort; // ← KakaoPay 직접 의존 제거
}

// Phase 3: UseCase 인터페이스 도입 (선택사항, Hexagonal로의 연결)

// application/PlaceOrderUseCase.java (새 파일)
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

// OrderService가 이를 구현
@Service
public class OrderService implements PlaceOrderUseCase {
    @Override
    public OrderId placeOrder(PlaceOrderCommand command) { ... }
}

// Controller: UseCase 인터페이스에만 의존
@RestController
public class OrderController {
    private final PlaceOrderUseCase placeOrderUseCase; // OrderService 직접 X
}
```

---

## 📊 패턴 비교 — DIP 적용 전후

```
=== DIP 적용 전 ===
의존성 그래프:
  OrderController → OrderService → OrderJpaRepository → DB
                                → KakaoPayClient → 외부 API
                                → JavaMailSender → SMTP

변경 시나리오: JPA → MongoDB 교체
  변경 파일:
    OrderJpaRepository → MongoOrderRepository (새 파일)
    OrderService (직접 의존하므로 수정 필요)
    관련 테스트 전부

=== DIP 적용 후 ===
의존성 그래프:
  OrderController → OrderService → OrderRepository (I/F) ← JpaOrderRepository → DB
                                → PaymentPort (I/F) ← KakaoPayAdapter → 외부 API
                                → NotificationPort (I/F) ← SmtpAdapter → SMTP

변경 시나리오: JPA → MongoDB 교체
  변경 파일:
    MongoOrderRepository (새 파일, OrderRepository 구현)
    Spring 설정에서 빈 교체 or @Profile 사용
    OrderService: 변경 없음!

변경 비용: O(N) → O(1)
테스트 속도: @SpringBootTest(30초) → 순수 Java(0.01초)
```

---

## ⚖️ 트레이드오프

```
DIP 적용 비용:
  인터페이스 파일 추가 (OrderRepository, PaymentPort 등)
  Infrastructure에 매핑/변환 코드 추가 (Domain ↔ JpaEntity)
  팀원들이 "왜 인터페이스가 필요한가?" 이해 필요

DIP 적용 이익:
  Service 단위 테스트 가능 (InMemory 사용)
  인프라 교체 시 Service 코드 변경 없음
  Hexagonal Architecture로의 자연스러운 전환 경로

DIP가 불필요한 경우:
  구현체가 단 하나이고 절대 교체될 일 없을 때
  2주짜리 프로토타입
  팀이 DIP 이익을 이해하지 못하는 상태에서 강제 적용
```

---

## 📌 핵심 정리

```
DIP로 레이어드 아키텍처 개선:

핵심 변화:
  Before: OrderService → OrderJpaRepository (구체 클래스)
  After:  OrderService → OrderRepository (인터페이스) ← JpaOrderRepository

인터페이스 위치:
  Business/Domain Layer에 위치 (매우 중요)
  Infrastructure Layer에 두면 DIP 효과 없음

즉각적인 이익:
  단위 테스트: InMemoryOrderRepository로 JPA 없이 테스트
  인프라 교체: Adapter 교체만으로 Service 코드 변경 없음

Hexagonal Architecture와의 연결:
  OrderRepository 인터페이스 = Driven Port
  JpaOrderRepository 구현체 = Driven Adapter
  → 이미 Hexagonal의 절반

점진적 적용 순서:
  1. Repository 인터페이스를 Domain Layer로
  2. 외부 의존성(결제, 알림) Port로 추상화
  3. UseCase 인터페이스 도입 (Hexagonal 완성)
```

---

## 🤔 생각해볼 문제

**Q1.** `OrderRepository` 인터페이스를 Infrastructure Layer에 두는 것과 Business(Domain) Layer에 두는 것은 어떤 차이가 있는가?

<details>
<summary>해설 보기</summary>

**인터페이스가 어느 레이어에 위치하느냐가 DIP의 핵심입니다.**

```
인터페이스가 Infrastructure Layer에 있으면:
  OrderService (Business) → OrderRepository (Infrastructure)
  의존성 방향: Business → Infrastructure (DIP 미적용)
  → Service가 Infrastructure Layer를 알아야 함

인터페이스가 Business/Domain Layer에 있으면:
  OrderService (Business) → OrderRepository (Business/Domain)
  JpaOrderRepository (Infrastructure) → OrderRepository (Business/Domain) 구현
  의존성 방향: Infrastructure → Business (DIP 적용!)
  → Infrastructure가 Business의 계약을 구현
```

Spring Data JPA의 `JpaRepository<Order, Long>`을 직접 상속한 인터페이스는 Infrastructure Layer에 속합니다. `@Autowired OrderJpaRepository`처럼 이것에 의존하면 DIP가 적용되지 않습니다.

</details>

---

**Q2.** DIP를 적용했을 때 Domain Entity(`Order`)와 JPA Entity(`OrderJpaEntity`)를 분리해야 하는가? 분리하지 않으면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**이상적으로는 분리해야 하지만, 비용이 크므로 점진적으로 접근하는 것이 현실적입니다.**

분리하지 않을 때의 문제:
```java
@Entity // JPA 어노테이션이 Domain 객체에
public class Order {
    @Id @GeneratedValue
    private Long id;
    
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderLine> lines; // LAZY 로딩 — 도메인 메서드에서 예외 가능
    
    protected Order() {} // JPA 요구 — 불완전한 객체 생성 허용
}
```

분리했을 때:
```java
// Domain (순수 Java)
public class Order {
    private final OrderId id;
    private final List<OrderLine> lines; // 항상 완전히 로딩됨
    // 기본 생성자 없음 — 불완전한 상태 생성 불가
}

// Infrastructure (JPA Entity)
@Entity @Table(name = "orders")
public class OrderJpaEntity {
    @Id private Long id;
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderLineJpaEntity> lines;
    protected OrderJpaEntity() {} // JPA 요구사항
}
```

**현실적 전략:**
- 처음엔 `@Entity`가 붙은 채로 DIP 적용 시작
- 도메인 로직이 JPA 제약(`LAZY`, 기본 생성자)에 걸리기 시작하면 분리
- 분리 전에 테스트 커버리지 충분히 확보

</details>

---

**Q3.** DIP를 적용한 레이어드 아키텍처와 Hexagonal Architecture의 실질적 차이는 무엇인가? DIP만 적용해도 Hexagonal과 동일한 것 아닌가?

<details>
<summary>해설 보기</summary>

DIP 적용 레이어드 ≈ Hexagonal의 85%, 하지만 차이가 있습니다.

**같은 점:**
- Driven Port: `OrderRepository` 인터페이스 → 동일
- Driven Adapter: `JpaOrderRepository` → 동일
- 의존성 역전: Infrastructure → Business → 동일

**다른 점:**

1. **Driving Port(UseCase 인터페이스)**:
   - DIP 적용 레이어드: Controller가 Service 구체 클래스에 직접 의존
   - Hexagonal: Controller → `PlaceOrderUseCase` 인터페이스에만 의존
   → Controller 테스트 시 UseCase Mock이 간단해짐

2. **명시적 구조**:
   - DIP 적용 레이어드: 팀 내 합의로 인터페이스 위치 관리
   - Hexagonal: application/domain/infrastructure 패키지 구조로 명확
   → 신규 팀원이 "어디에 코드를 넣어야 하는가"를 패키지 구조로 이해

3. **ArchUnit 규칙**:
   - Hexagonal: 패키지 기반으로 명확한 규칙 설정 가능
   - DIP 적용 레이어드: 규칙이 암묵적 합의에 의존

결론: DIP 적용은 Hexagonal로 가는 중간 단계입니다. UseCase 인터페이스 추가와 패키지 구조 정리를 하면 Hexagonal이 됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: 레이어드 아키텍처에서 테스트](./04-testing-in-layered.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: Hexagonal Architecture ➡️](../hexagonal-architecture/01-hexagonal-core-idea.md)**

</div>
