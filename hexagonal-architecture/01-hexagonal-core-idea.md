# Hexagonal Architecture의 핵심 아이디어 — 내부와 외부의 분리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Alistair Cockburn이 Hexagonal Architecture를 만든 원래 의도는 무엇인가?
- 애플리케이션을 "내부(도메인)"와 "외부(인프라)"로 분리하는 것이 왜 중요한가?
- Port가 "안과 밖의 계약(Contract)"이라는 말의 정확한 의미는?
- 레이어드 아키텍처와 Hexagonal Architecture의 핵심적인 구조 차이는 무엇인가?
- "육각형"이라는 이름은 어디서 왔고 어떤 의미인가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

레이어드 아키텍처의 문제는 결국 하나로 귀결된다 — **도메인이 인프라를 알고 있다는 것**. DB가 바뀌면 Service 코드가 바뀌고, 외부 API가 바뀌면 비즈니스 로직이 영향받는다.

Hexagonal Architecture는 이 방향을 완전히 뒤집는다. 도메인은 아무것도 모른다. 인프라가 도메인을 구현한다. 이 단순한 원칙이 테스트, 유지보수, 외부 시스템 교체 모든 측면을 바꾼다.

---

## 😱 흔한 실수 (Before — Hexagonal을 "폴더 구조"로 이해할 때)

```
흔한 오해:
  "Hexagonal Architecture = application/domain/infrastructure 폴더 구조"

그 결과:
  infrastructure/
    KakaoPayClient.java
  domain/
    OrderService.java
      @Autowired KakaoPayClient kakaoPayClient; // 여전히 인프라에 직접 의존!
  application/
    OrderController.java

폴더는 나눴지만 의존성은 여전히 인프라 방향
= 레이어드 아키텍처에 폴더 이름만 바꾼 것

진짜 Hexagonal의 핵심은 폴더 구조가 아니라:
  "도메인이 무엇도 모르는가?"
  "외부 시스템이 바뀌어도 도메인이 변경되지 않는가?"
  "도메인을 Spring 없이, DB 없이 테스트할 수 있는가?"

이 세 질문에 Yes가 나와야 Hexagonal Architecture
```

---

## ✨ 올바른 접근 (After — Hexagonal의 진짜 의미)

```
Hexagonal Architecture의 핵심 아이디어:

"애플리케이션은 외부 세계와 Port를 통해서만 통신한다"

내부 (도메인, Application Core):
  비즈니스 규칙, 도메인 로직
  어떤 프레임워크도 모름 (Spring X)
  어떤 DB도 모름 (JPA X)
  어떤 외부 API도 모름 (KakaoPay SDK X)

외부 (인프라, Adapters):
  HTTP Controller (Driving Adapter)
  JPA Repository (Driven Adapter)
  Kafka Publisher (Driven Adapter)
  KakaoPay Client (Driven Adapter)

Port (경계, Contract):
  내부와 외부 사이의 인터페이스
  Driving Port: 외부 → 내부로 들어오는 계약 (UseCase 인터페이스)
  Driven Port: 내부 → 외부로 나가는 계약 (Repository, EventPublisher 인터페이스)

결과:
  외부 시스템 전체 교체 → Adapter만 교체, 도메인 코드 변경 없음
  도메인 테스트 → Spring X, JPA X, 외부 API X → ms 단위로 실행
```

---

## 🔬 내부 원리 — Hexagonal Architecture의 구조

### 1. Alistair Cockburn의 원래 의도

```
2005년 Alistair Cockburn이 제안:
  "Ports and Adapters Architecture"가 원래 이름
  "Hexagonal Architecture"는 육각형 다이어그램에서 나온 별칭

핵심 동기:
  "애플리케이션이 사람, 자동화된 테스트, 배치 프로그램, 다른 애플리케이션
   중 어느 것에 의해서도 동일하게 구동될 수 있어야 한다.
   그리고 최종 실행 환경과 독립적으로 개발 및 테스트할 수 있어야 한다."
   — Alistair Cockburn

이 목표를 위한 구조:
  애플리케이션 코어(도메인)가 외부 환경을 모르게
  → 테스트 시: TestAdapter
  → 운영 시: 실제 JPA Adapter, Kafka Adapter
  → 어떤 Adapter를 꽂아도 도메인은 동일하게 동작

"육각형"의 의미:
  실제 6개의 포트가 있다는 게 아님
  육각형의 각 면이 하나의 Port를 나타냄
  → "여러 개의 포트가 있을 수 있다"는 시각적 표현
  → 사각형(레이어드)처럼 위아래 방향이 아닌, 모든 방향에서 포트 가능
```

### 2. 내부와 외부의 분리 — 구체적인 경계

```
내부 (Application Core):
┌────────────────────────────────────────────────┐
│                                                │
│  Domain Layer:                                 │
│    Order (Entity)                              │
│    Money (Value Object)                        │
│    OrderStatus (Enum)                          │
│    OrderDomainService                          │
│                                                │
│  Application Layer:                            │
│    PlaceOrderUseCase (Driving Port)            │
│    PlaceOrderService (UseCase 구현)             │
│    OrderRepository (Driven Port)               │
│    PaymentPort (Driven Port)                   │
│    OrderEventPublisher (Driven Port)           │
│                                                │
└────────────────────────────────────────────────┘

경계 (Port):
  Driving Port: PlaceOrderUseCase (외부 → 내부)
  Driven Port:  OrderRepository, PaymentPort (내부 → 외부)

외부 (Adapters):
  Driving Adapter: OrderController (HTTP → PlaceOrderUseCase 호출)
  Driven Adapter:  JpaOrderRepository (OrderRepository 구현)
  Driven Adapter:  KakaoPayAdapter (PaymentPort 구현)
  Driven Adapter:  KafkaEventPublisher (OrderEventPublisher 구현)
  Driven Adapter:  InMemoryOrderRepository (테스트용)
```

### 3. Port가 "계약(Contract)"인 이유

```
Port = 인터페이스 (Java)
     = 내부와 외부 사이의 계약

계약이란:
  "이 메서드를 이 시그니처로 구현하면, 내부가 올바르게 동작한다"

예시: OrderRepository Port

  public interface OrderRepository {
      void save(Order order);                    // 계약: 저장 성공 or 예외
      Optional<Order> findById(OrderId id);      // 계약: empty or Order, null 없음
  }

  이 계약을 지키는 구현체라면 어떤 것이든:
    JpaOrderRepository → MySQL DB에 저장
    MongoOrderRepository → MongoDB에 저장
    InMemoryOrderRepository → 메모리에 저장 (테스트용)
    CachedOrderRepository → Redis 캐시 + DB

  도메인(PlaceOrderService)은 어떤 구현체인지 모름
  → 계약(인터페이스)만 알면 됨

계약이 깨지면 (LSP 위반):
  InMemoryOrderRepository.findById() → null 반환 (계약: Optional)
  → PlaceOrderService에서 NullPointerException
  → 계약 위반이 런타임 오류로 나타남
```

### 4. 레이어드 vs Hexagonal 구조 비교

```
=== 레이어드 아키텍처 ===

      [HTTP Client]
           │
    [Presentation Layer]
    OrderController
           │ depends on
    [Business Layer]
    OrderService ──depends on──→ OrderJpaRepository (인프라!)
                 ──depends on──→ KakaoPayClient (인프라!)
           │
    [Data Access Layer]
    OrderJpaRepository
           │
         [DB]

의존성이 인프라 방향으로 흐름
OrderService가 JPA와 KakaoPay SDK를 직접 앎

=== Hexagonal Architecture ===

[HTTP Client]                        [DB]
     │                                 │
[OrderController]          [JpaOrderRepository]
(Driving Adapter)          (Driven Adapter)
     │ calls                    │ implements
     ↓                          │
[PlaceOrderUseCase] ←──── [PlaceOrderService] ────→ [OrderRepository]
(Driving Port)        implements (Domain)     uses   (Driven Port)
                                                         │ implements
                                              [KakaoPayAdapter] ── [KakaoPayClient]
                                              (Driven Adapter)

핵심 차이:
  레이어드: 도메인(Service)이 인프라(JPA, KakaoPay)를 직접 의존
  Hexagonal: 인프라(Adapter)가 도메인(Port 인터페이스)을 구현
             도메인은 아무 인프라도 모름

모든 의존성이 도메인(Application Core)을 향함:
  Controller → UseCase ← Service
  Service → OrderRepository ← JpaOrderRepository
  Service → PaymentPort ← KakaoPayAdapter
```

### 5. 왜 "육각형"이고 왜 여러 포트가 가능한가

```
레이어드 (사각형 구조):
  위 → 아래 방향만 존재
  HTTP 위, DB 아래 — 방향이 고정

  문제:
    CLI로 애플리케이션을 구동하려면? → 별도 구조 필요
    배치 작업으로 구동하려면? → 별도 구조 필요
    테스트로 구동하려면? → @SpringBootTest 필요

Hexagonal (육각형 구조):
  어떤 방향에서도 Port를 통해 접근 가능
  
       [HTTP REST API]          [GraphQL]
           │                        │
    [OrderRestController]   [OrderGraphQLController]
           │                        │
           └──────→ [PlaceOrderUseCase] ←──────┘
                           │ (Application Core)
           ┌───────────────┼───────────────────┐
           │               │                   │
    [OrderRepository]  [PaymentPort]   [EventPublisher]
           │               │                   │
    [JpaAdapter]    [KakaoPayAdapter]  [KafkaAdapter]
    [MongoAdapter]  [TossPayAdapter]   [InMemoryAdapter] ← 테스트용

  HTTP, GraphQL, CLI, 배치 — 어떤 방식으로 구동해도 Application Core는 동일
  JPA, MongoDB — 어떤 DB를 써도 Application Core는 동일
  KakaoPay, TossPay — 어떤 결제 수단을 써도 Application Core는 동일

  = "교체 가능성"을 극대화한 구조
```

---

## 💻 실전 코드 — Hexagonal의 핵심 구조

```java
// === Application Core (내부) ===

// Driving Port (외부 → 내부)
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

// Driven Port (내부 → 외부)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentPort {
    PaymentResult charge(PaymentRequest request);
}

// Application Core 구현 — 인프라를 전혀 모름
@Service
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderRepository orderRepository;  // Port만 앎
    private final PaymentPort paymentPort;           // Port만 앎

    public PlaceOrderService(OrderRepository orderRepository, PaymentPort paymentPort) {
        this.orderRepository = orderRepository;
        this.paymentPort = paymentPort;
    }

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.getUserId(), command.getLines());
        order.place();

        PaymentResult payment = paymentPort.charge(PaymentRequest.of(order));
        order.confirmPayment(payment.getTransactionId());

        orderRepository.save(order);
        return order.getId();
    }
    // JPA 없음, KakaoPay SDK 없음, Spring 없음 → 순수 Java
}

// === Adapters (외부) ===

// Driving Adapter: HTTP 요청을 UseCase로 변환
@RestController
public class OrderController {
    private final PlaceOrderUseCase placeOrderUseCase; // Port만 앎

    @PostMapping("/api/orders")
    public ResponseEntity<PlaceOrderResponse> placeOrder(@RequestBody PlaceOrderRequest req) {
        PlaceOrderCommand command = PlaceOrderCommand.from(req);
        OrderId id = placeOrderUseCase.placeOrder(command);
        return ResponseEntity.created(...).body(PlaceOrderResponse.of(id));
    }
}

// Driven Adapter: OrderRepository Port 구현
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final SpringDataOrderJpaRepository jpa;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        jpa.save(mapper.toJpaEntity(order));
    }
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value()).map(mapper::toDomain);
    }
}

// Driven Adapter: PaymentPort 구현
@Component
public class KakaoPayAdapter implements PaymentPort {
    private final KakaoPayClient client;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        KakaoPayResponse res = client.charge(KakaoPayRequest.of(request));
        return PaymentResult.success(res.getTransactionId());
    }
}

// 테스트용 Adapter (InMemory)
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<OrderId, Order> store = new HashMap<>();
    @Override public void save(Order order) { store.put(order.getId(), order); }
    @Override public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id));
    }
}

// === 단위 테스트 (Spring/JPA/KakaoPay 없음) ===
class PlaceOrderServiceTest {
    InMemoryOrderRepository orderRepository = new InMemoryOrderRepository();
    PaymentPort paymentPort = req -> PaymentResult.success("tx-001");

    PlaceOrderService sut = new PlaceOrderService(orderRepository, paymentPort);

    @Test
    void 주문_생성_성공() {
        OrderId id = sut.placeOrder(validCommand());
        assertThat(orderRepository.findById(id)).isPresent();
        // Spring 없음, JPA 없음, 실행 시간: ~0.01초
    }
}
```

---

## 📊 패턴 비교 — 레이어드 vs Hexagonal 핵심 비교

```
핵심 차이 요약:

관점                │ 레이어드 아키텍처         │ Hexagonal Architecture
────────────────────┼──────────────────────────┼────────────────────────────
의존성 방향          │ 도메인 → 인프라           │ 인프라 → 도메인 (역전)
도메인이 아는 것      │ JPA, Kafka, HTTP 등      │ 아무것도 모름 (Port만)
외부 시스템 교체     │ Service 코드 수정 필요    │ Adapter 교체만으로 가능
단위 테스트          │ @SpringBootTest 의존     │ 순수 Java, ms 단위
새 입력 방식 추가    │ Controller 수정           │ 새 Driving Adapter 추가만
테스트 전략         │ Mock이 구현에 결합         │ InMemory Adapter로 격리
```

---

## ⚖️ 트레이드오프

```
Hexagonal Architecture의 비용:
  Port 인터페이스 파일 증가
  Domain Entity ↔ JPA Entity 매핑 코드 필요
  팀 학습 곡선 (Port/Adapter/Driving/Driven 개념)
  초기 개발 속도 약간 감소

Hexagonal Architecture의 이익:
  도메인 코드의 인프라 독립성 완전 달성
  단위 테스트 속도 극적 향상 (30초 → 0.01초)
  외부 시스템 교체 용이성 (Adapter만 교체)
  여러 입력 채널 지원 (HTTP, GraphQL, CLI, 배치)

레이어드와 Hexagonal 중 선택:
  단순 CRUD → 레이어드 (DIP 개선형으로 충분)
  복잡한 비즈니스 + 외부 시스템 연동 → Hexagonal 권장
  외부 시스템 교체 가능성 높음 → Hexagonal 필수
```

---

## 📌 핵심 정리

```
Hexagonal Architecture의 핵심:

Alistair Cockburn의 의도:
  "외부 환경과 무관하게 도메인을 개발하고 테스트할 수 있게"

구조:
  내부 (Application Core): 도메인, UseCase, Port 인터페이스
  외부 (Adapters): HTTP Controller, JPA Repository, Kafka Publisher
  경계 (Port): 내부와 외부 사이의 계약 (인터페이스)

의존성 방향:
  모든 화살표가 Application Core(내부)를 향함
  인프라가 도메인 인터페이스를 구현 = DIP 완전 적용

핵심 질문 (Hexagonal 확인용):
  "도메인이 Spring을 아는가?" → NO여야 함
  "도메인이 JPA를 아는가?" → NO여야 함
  "DB 없이 도메인 테스트 가능한가?" → YES여야 함

레이어드와의 결정적 차이:
  레이어드: 인프라가 아래에 있고 도메인이 위에서 내려다봄 (의존)
  Hexagonal: 도메인이 중앙에 있고 인프라가 사방에서 감싸며 구현
```

---

## 🤔 생각해볼 문제

**Q1.** Hexagonal Architecture에서 "도메인이 인프라를 모른다"면, 도메인은 어떻게 DB에 데이터를 저장하는가? 저장 동작은 누가 트리거하는가?

<details>
<summary>해설 보기</summary>

도메인(Order)은 저장을 직접 하지 않습니다. **Application Layer(PlaceOrderService)가 OrderRepository Port를 통해 트리거합니다.**

```java
// Domain: 저장 방법 모름, 비즈니스 로직만
public class Order {
    public void place() {
        // 비즈니스 규칙 실행 — 저장 책임 없음
        this.status = OrderStatus.PLACED;
    }
}

// Application: 언제 저장할지 결정
public class PlaceOrderService {
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(...);
        order.place();                           // Domain에 로직 위임
        orderRepository.save(order);            // Application이 저장 트리거
        return order.getId();
    }
}

// Infrastructure: 어떻게 저장할지 결정
public class JpaOrderRepository implements OrderRepository {
    public void save(Order order) {
        jpa.save(mapper.toJpaEntity(order));    // JPA로 실제 저장
    }
}
```

책임 분리:
- **도메인**: 비즈니스 규칙 (무엇을)
- **Application**: 언제 저장할지 결정 (언제)
- **Infrastructure**: 어떻게 저장할지 구현 (어떻게)

</details>

---

**Q2.** "Port"라는 이름이 왜 적절한가? 소프트웨어 포트와 USB 포트의 유사점은 무엇인가?

<details>
<summary>해설 보기</summary>

USB 포트와의 유사점이 정확합니다.

**USB 포트:**
- 규격(인터페이스)이 정해져 있음
- 어떤 장치(마우스, 키보드, 외장하드)를 연결해도 포트 규격만 맞으면 동작
- 포트 내부(컴퓨터)는 어떤 장치가 연결됐는지 모름 (그냥 USB 장치)
- 장치 교체 시 컴퓨터 내부 변경 없음

**소프트웨어 Port:**
- 인터페이스가 정해져 있음 (`OrderRepository.save(Order)`)
- 어떤 Adapter(JPA, MongoDB, InMemory)를 연결해도 Port 계약만 맞으면 동작
- Application Core(도메인)는 어떤 Adapter인지 모름 (그냥 `OrderRepository`)
- Adapter 교체 시 Application Core 변경 없음

Cockburn이 "Ports and Adapters"라고 명명한 이유가 이 유사성입니다. Port는 규격(계약), Adapter는 그 규격에 맞게 외부 기술을 연결하는 어댑터.

</details>

---

**Q3.** Hexagonal Architecture에서 도메인 객체(Order)에 `@Entity` 어노테이션을 붙여도 되는가? 이것이 "도메인이 인프라를 모른다"는 원칙과 충돌하는가?

<details>
<summary>해설 보기</summary>

**이론적으로는 충돌하고, 현실적으로는 절충합니다.**

이론적 완전 분리:
```java
// Domain (순수 Java — @Entity 없음)
public class Order {
    private final OrderId id;
    // JPA 어노테이션 없음
}

// Infrastructure (JPA Entity)
@Entity @Table(name = "orders")
public class OrderJpaEntity { ... }
```
→ 도메인이 JPA를 완전히 모름, 순수 비즈니스 로직만

현실적 절충 (작은 프로젝트):
```java
@Entity  // JPA 어노테이션은 있지만
public class Order {
    // JPA 제약(@GeneratedValue, LAZY 로딩)이 비즈니스 메서드를 오염하지 않도록 주의
    public void place() { /* 순수 비즈니스 로직 */ }
}
```
→ 완전한 분리는 아니지만, 실용적인 타협

**판단 기준:**
- JPA 제약(기본 생성자, LAZY 로딩 예외)이 비즈니스 메서드에 영향을 준다면 → 분리 필요
- 영향이 없다면 → 절충 허용

DDD 원칙 상으로는 분리를 권장하지만, 분리 비용(매핑 코드)이 크므로 팀 상황에 따라 결정하세요.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 레이어드 아키텍처 개선](../layered-architecture/05-improving-with-dip.md)** | **[홈으로 🏠](../README.md)** | **[다음: 주도하는 포트 vs 주도받는 포트 ➡️](./02-driving-vs-driven-ports.md)**

</div>
