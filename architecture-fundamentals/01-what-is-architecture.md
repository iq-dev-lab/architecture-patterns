# 아키텍처의 본질 — 아키텍처는 폴더 구조가 아니다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 소프트웨어 아키텍처란 정확히 무엇인가? 폴더 구조나 기술 스택 선택과 어떻게 다른가?
- Uncle Bob이 "아키텍처는 결정을 미루는 예술"이라고 말한 의도는 무엇인가?
- 좋은 아키텍처가 선택지를 열어두는 것이 왜 중요한가?
- 변경 비용을 낮추는 것이 아키텍처의 목적인 이유는 무엇인가?
- 아키텍처 결정 없이 코드를 짤 때 소프트웨어 엔트로피는 어떻게 증가하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

"우리는 MVC 패턴을 씁니다"라고 말하는 팀과, "도메인 레이어가 어떤 프레임워크도 몰라야 하는 이유, 의존성이 역전될 때 무슨 일이 일어나는지 아는 팀"의 코드는 6개월 후 완전히 다른 모습이 된다.

아키텍처를 모르면 처음에는 빠르게 짤 수 있다. 그러나 기능이 추가될수록, 팀원이 늘어날수록, 시스템이 성장할수록 변경 비용이 기하급수적으로 증가한다. 아키텍처는 이 증가 곡선을 통제하는 도구다.

---

## 😱 흔한 실수 (Before — 아키텍처를 "폴더 구조"로 이해할 때)

```
상황: 새 기능 요청 — "결제 방식을 카카오페이에서 토스페이로 추가해주세요"

아키텍처를 모를 때의 코드 상태:
  OrderService.java (812줄)
    - 주문 생성 로직
    - 결제 처리 로직 (KakaoPay SDK 직접 호출)
    - 재고 차감 로직
    - 이메일 발송 로직 (SMTP 직접 호출)
    - 배송 생성 로직
    - 포인트 적립 로직

변경 시 발생하는 일:
  "카카오페이 → 토스페이 추가"라는 단순한 요청이
  → OrderService 812줄을 전부 이해해야 함
  → KakaoPay 코드가 주문 로직과 뒤섞여서 추출 불가
  → TossPayService 추가했더니 OrderService가 두 SDK를 모두 앎
  → 결제 방식 테스트를 위해 주문, 재고, 이메일까지 Mocking 필요
  → 결제 관련 버그 수정이 주문 로직에 영향을 줄 수 있음

개발자의 체감:
  "이 코드 건드리면 어디서 터질지 모르겠다"
  "리팩터링하고 싶은데 테스트가 없어서 무서워서 못 하겠다"
  "새 기능 추가할 때마다 왜 이렇게 오래 걸리지?"

이것이 아키텍처 부채가 쌓이는 과정:
  Day 1: 빠르게 짬 → 동작함
  Month 3: 기능 추가 → 조금 느림
  Month 6: 기능 추가 → 많이 느림
  Month 12: 기능 추가 → 매우 느림 (엔트로피 임계점)
  Month 18: "전면 재개발" 논의 시작
```

---

## ✨ 올바른 접근 (After — 아키텍처를 "변경 비용 통제"로 이해할 때)

```
같은 요청 — "결제 방식을 토스페이로 추가해주세요"

아키텍처가 있을 때의 코드 상태:
  domain/
    Order.java              ← 순수 비즈니스 규칙, 결제 SDK 모름
    PaymentPort.java        ← 인터페이스 (Port)

  application/
    PlaceOrderUseCase.java  ← 결제 포트에만 의존 (구현체 모름)

  infrastructure/
    KakaoPayAdapter.java    ← PaymentPort 구현 (카카오페이 SDK)
    TossPayAdapter.java     ← PaymentPort 구현 (토스페이 SDK, 신규 추가)

변경 시 발생하는 일:
  "토스페이 추가"
  → TossPayAdapter.java 하나만 새로 작성
  → 기존 Order, PlaceOrderUseCase 코드 변경 없음
  → TossPayAdapter만 단독으로 테스트 가능
  → 다른 기능에 영향 0

개발자의 체감:
  "결제 방식 추가는 Adapter 하나만 새로 만들면 돼"
  "도메인 로직은 어떤 결제 SDK가 들어오든 테스트 가능해"
  "새 팀원도 PaymentPort만 보면 구현 방법을 알 수 있어"

이것이 아키텍처가 선택지를 열어두는 방식:
  초기 설계 시 "어떤 결제 수단을 쓸지"를 결정하지 않아도 됨
  → Port(인터페이스)만 정의
  → 나중에 Adapter를 바꿔 끼우면 됨
  → 변경 비용: O(1) — Adapter 하나만 교체
```

---

## 🔬 내부 원리 — 아키텍처가 변경 비용을 낮추는 메커니즘

### 1. 아키텍처의 정의 — Martin과 Fowler의 관점

```
Martin Fowler:
  "아키텍처는 변경하기 어려운 결정들의 집합이다"
  → 즉, 나중에 바꾸기 힘든 결정들

Uncle Bob (Robert C. Martin):
  "좋은 아키텍처는 결정을 최대한 미룰 수 있게 한다"
  → DB를 MySQL로 할지 PostgreSQL로 할지를 나중에 결정해도 됨
  → 프레임워크를 Spring으로 할지 Quarkus로 할지를 나중에 결정해도 됨
  → 이것이 가능하려면: 도메인이 DB와 프레임워크를 몰라야 함

두 관점의 공통점:
  아키텍처 = 변경 비용을 통제하는 결정
  폴더 구조가 아니라 의존성 방향을 결정하는 것
```

### 2. 소프트웨어 엔트로피 — 아키텍처 없이 코드가 무너지는 과정

```
물리학의 엔트로피: 닫힌 계는 시간이 지날수록 무질서해진다
소프트웨어 엔트로피: 아키텍처 원칙 없이는 코드가 시간이 지날수록 복잡해진다

초기 상태 (Day 1):
  OrderController
      └── OrderService
              └── OrderRepository

3개월 후:
  OrderController
      └── OrderService
              ├── OrderRepository
              ├── KakaoPayClient (결제)
              ├── EmailSender (알림)
              └── InventoryService (재고)

6개월 후:
  OrderController ←──────────── UserController
      └── OrderService ←──────── PaymentService
              ├── OrderRepository     │
              ├── KakaoPayClient ←────┘
              ├── EmailSender
              ├── InventoryService
              └── ShippingService ←── AdminService

문제:
  의존성이 사방으로 퍼짐 → 어디를 바꾸면 어디가 깨질지 예측 불가
  이것을 "Big Ball of Mud (진흙 덩어리)" 패턴이라고 부름

엔트로피를 낮추는 것 = 아키텍처
  의존성의 방향을 단방향으로 강제
  레이어 경계를 명확히 정의
  변경이 전파되는 범위를 예측 가능하게 만듦
```

### 3. 아키텍처가 "선택지를 열어둔다"는 것의 의미

```
Uncle Bob의 핵심 주장:
  "아키텍처는 비즈니스 결정(도메인)을 기술 결정(인프라)으로부터 분리한다"

선택지를 열어두는 예시:

  ① DB 선택을 미룰 수 있다
     Before: OrderService가 JpaRepository에 직접 의존
             → 프로젝트 시작 시 DB를 반드시 결정해야 함
             → 나중에 DB를 바꾸면 Service 코드도 변경 필요

     After:  OrderService가 OrderRepository(인터페이스)에만 의존
             → 초기에는 InMemoryOrderRepository로 개발
             → 나중에 JpaOrderRepository, MongoOrderRepository 선택
             → Service 코드 변경 없음

  ② 프레임워크 선택을 미룰 수 있다
     Before: 도메인 객체에 @Entity, @Column, @Id 붙어있음
             → JPA 없이는 도메인을 테스트할 수 없음
             → JPA 바꾸면 도메인 코드도 바꿔야 함

     After:  도메인 객체는 순수 Java
             → JPA는 Adapter 레이어에만 존재
             → 도메인 테스트: JPA 없이 ms 단위로 실행

  ③ 외부 API 선택을 미룰 수 있다
     Before: Service가 KakaoPayClient를 직접 호출
             → 카카오페이 없으면 Service 테스트 불가
             → 결제 수단 변경 시 Service 수정 필요

     After:  Service가 PaymentPort(인터페이스)에만 의존
             → 테스트: FakePaymentPort 사용
             → 결제 수단 변경: 새 Adapter 추가만 하면 됨

이것이 Uncle Bob이 말한 "결정을 미루는 예술":
  지금 당장 결정하지 않아도 되는 것들을 나중으로 미룸
  = 나중에 더 많은 정보를 가지고 결정할 수 있음
  = 변경 비용을 낮춤
```

### 4. 아키텍처 결정의 두 가지 유형

```
변경하기 어려운 결정 (아키텍처 결정):
  - 의존성 방향 (도메인이 인프라를 알아야 하는가?)
  - 레이어 경계 (어디까지가 도메인이고 어디부터가 인프라인가?)
  - 모듈 경계 (어떤 Bounded Context로 나눌 것인가?)
  - 통신 방식 (동기 API? 비동기 이벤트?)

나중에 바꿀 수 있는 결정 (기술 결정):
  - DB: MySQL → PostgreSQL → MongoDB
  - 메시징: Kafka → RabbitMQ
  - 프레임워크: Spring MVC → Spring WebFlux
  - 외부 API: KakaoPay → TossPay

좋은 아키텍처:
  변경하기 어려운 결정 → 신중하게, 초기에 설계
  나중에 바꿀 수 있는 결정 → 인터페이스로 추상화하여 미룸

나쁜 아키텍처:
  변경하기 어려운 결정 → 무의식적으로 결정됨 (기본값)
  나중에 바꿀 수 있는 결정 → 도메인에 직접 박아넣음
```

---

## 💻 실전 코드 — 아키텍처 결정이 코드에 나타나는 방식

```java
// ❌ Before: 아키텍처 결정이 없을 때
// 도메인(OrderService)이 기술 결정(JPA, Kafka, SMTP)을 직접 알고 있음

@Service
@Transactional
public class OrderService {

    @Autowired
    private OrderJpaRepository orderRepository; // JPA에 직접 의존

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate; // Kafka에 직접 의존

    @Autowired
    private JavaMailSender mailSender; // SMTP에 직접 의존

    public OrderId placeOrder(OrderRequest request) {
        // 비즈니스 로직 + 인프라 로직이 뒤섞임
        Order order = new Order(request.getItems());
        orderRepository.save(order); // JPA

        kafkaTemplate.send("order-placed", order.getId().toString()); // Kafka

        SimpleMailMessage mail = new SimpleMailMessage();
        mail.setTo(request.getEmail());
        mailSender.send(mail); // SMTP

        return order.getId();
    }
}
// 결과: OrderService 테스트에 JPA, Kafka, SMTP 컨테이너가 전부 필요
```

```java
// ✅ After: 아키텍처 결정이 명확할 때
// 도메인은 인터페이스(Port)만 알고, 구현체(Adapter)는 인프라 레이어에

// === domain/ 레이어 — 어떤 기술도 모름 ===

public class Order { // 순수 Java, 어노테이션 없음
    private final OrderId id;
    private final List<OrderLine> lines;
    private OrderStatus status;

    public void place() {
        this.status = OrderStatus.PLACED;
        // 비즈니스 규칙만 존재
    }
}

public interface OrderRepository { // Driven Port
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface OrderEventPublisher { // Driven Port
    void publishOrderPlaced(OrderPlacedEvent event);
}

// === application/ 레이어 — Port에만 의존 ===

public interface PlaceOrderUseCase { // Driving Port (Input Boundary)
    OrderId placeOrder(PlaceOrderCommand command);
}

@Service
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderRepository orderRepository;   // Port
    private final OrderEventPublisher eventPublisher; // Port

    // 생성자 주입 — JPA도 Kafka도 SMTP도 전혀 모름
    public PlaceOrderService(
        OrderRepository orderRepository,
        OrderEventPublisher eventPublisher
    ) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = new Order(command.getItems());
        order.place(); // 비즈니스 규칙 실행

        orderRepository.save(order);
        eventPublisher.publishOrderPlaced(new OrderPlacedEvent(order));

        return order.getId();
    }
}

// === infrastructure/ 레이어 — Port를 구현 (Adapter) ===

@Repository
public class JpaOrderRepository implements OrderRepository {
    private final SpringDataOrderJpaRepository jpa;
    // JPA 세부 구현
}

@Component
public class KafkaOrderEventPublisher implements OrderEventPublisher {
    private final KafkaTemplate<String, String> kafkaTemplate;
    // Kafka 세부 구현
}

// === 테스트 — Spring Context 없이 빠르게 ===

class PlaceOrderServiceTest {

    private final OrderRepository orderRepository =
        new InMemoryOrderRepository();       // Adapter 교체

    private final OrderEventPublisher eventPublisher =
        new InMemoryOrderEventPublisher();   // Adapter 교체

    private final PlaceOrderService service =
        new PlaceOrderService(orderRepository, eventPublisher);

    @Test
    void 주문을_성공적으로_생성한다() {
        // JPA 없음, Kafka 없음, SMTP 없음
        // 수십 ms 안에 실행
        PlaceOrderCommand command = new PlaceOrderCommand(...);
        OrderId orderId = service.placeOrder(command);

        assertThat(orderId).isNotNull();
        assertThat(orderRepository.findById(orderId)).isPresent();
    }
}
```

---

## 📊 패턴 비교 — 아키텍처 유무에 따른 변경 비용 비교

```
기능 추가 시나리오: "결제 수단에 토스페이를 추가해주세요"

=== 아키텍처 없음 (Big Ball of Mud) ===

변경해야 하는 파일:
  OrderService.java      ← KakaoPay 코드가 섞여있어서 수정 필요
  PaymentService.java    ← 두 SDK를 모두 조건 분기로 처리
  OrderController.java   ← 결제 방식 파라미터 추가
  OrderTest.java         ← 모킹 대상이 많아서 수정 필요

리스크:
  기존 카카오페이 결제가 깨질 수 있음
  주문 로직에 의도치 않은 영향

변경 비용: 높음 (O(N) — 관련 파일 전부)

=== 아키텍처 있음 (Hexagonal) ===

변경해야 하는 파일:
  TossPayAdapter.java    ← 신규 파일 하나만 추가

변경하지 않아도 되는 파일:
  Order.java             ← 건드릴 이유 없음
  PlaceOrderService.java ← PaymentPort에만 의존하므로 변경 없음
  OrderController.java   ← 변경 없음

리스크:
  기존 카카오페이 로직에 영향 없음 (별도 Adapter)
  TossPayAdapter만 독립적으로 테스트

변경 비용: 낮음 (O(1) — Adapter 하나만 추가)
```

```
아키텍처 결정이 없을 때의 의존성 그래프:

OrderController ──→ OrderService ──→ OrderJpaRepository ──→ DB
                         │
                         ├──→ KakaoPayClient ──→ 외부 API
                         │
                         ├──→ JavaMailSender ──→ SMTP 서버
                         │
                         └──→ KafkaTemplate ──→ Kafka

문제: OrderService 변경 → 영향 범위 예측 불가
      DB 교체 → Service 코드 변경
      결제 API 교체 → Service 코드 변경
      테스트: DB + 외부API + SMTP + Kafka 전부 필요

아키텍처 결정이 있을 때의 의존성 그래프:

OrderController ──→ PlaceOrderUseCase(I/F) ←── PlaceOrderService
                                                      │
                                        OrderRepository(I/F)
                                              ↑ 구현
                                        JpaOrderRepository ──→ DB

                                        OrderEventPublisher(I/F)
                                              ↑ 구현
                                        KafkaEventPublisher ──→ Kafka

핵심: 모든 화살표가 도메인(PlaceOrderService) 방향을 향함
      DB 교체: JpaOrderRepository만 변경
      Kafka 교체: KafkaEventPublisher만 변경
      Service 테스트: InMemory 구현체로 대체
```

---

## ⚖️ 트레이드오프

```
아키텍처를 도입할 때의 비용:

단기 비용:
  ① 인터페이스 작성 비용 — Port마다 인터페이스 파일 추가
  ② 매핑 비용 — JPA Entity ↔ Domain Entity 변환 코드
  ③ 학습 비용 — 팀원이 Hexagonal/Clean 원칙을 익혀야 함
  ④ 초기 속도 저하 — 단순 CRUD도 구조에 맞게 짜야 함

장기 이익:
  ① 변경 비용 O(1) — Adapter 하나만 교체
  ② 테스트 속도 — Spring Context 없이 ms 단위 테스트
  ③ 신규 팀원 온보딩 — 코드 위치 규칙이 명확해서 빠른 적응
  ④ 기술 부채 방지 — 엔트로피 증가를 구조적으로 차단

아키텍처를 도입하지 말아야 할 때:
  - 프로토타입 / POC — 2주 안에 버릴 코드
  - 단순 CRUD만 존재하는 서비스 — Port/Adapter 오버엔지니어링
  - 팀 전체가 원칙에 동의하지 않은 상태 — 혼재된 구조가 더 나쁨

아키텍처를 반드시 도입해야 할 때:
  - 비즈니스 규칙이 복잡한 도메인
  - 외부 시스템 교체 가능성이 있는 경우
  - 빠른 단위 테스트가 필요한 경우
  - 팀 규모가 3명 이상이고 동시 개발이 필요한 경우
```

---

## 📌 핵심 정리

```
아키텍처의 본질:

아키텍처 = 변경 비용을 통제하는 결정들의 집합
         ≠ 폴더 구조, 기술 스택, 프레임워크 선택

Uncle Bob의 핵심 원칙:
  "좋은 아키텍처는 결정을 최대한 미룰 수 있게 한다"
  → DB, 프레임워크, 외부 API 선택을 나중으로 미룸
  → 도메인이 이들을 모르게 설계
  → 나중에 인터페이스만 구현하면 교체 가능

소프트웨어 엔트로피:
  아키텍처 원칙 없음 → 의존성이 사방으로 증가 → 변경 비용 기하급수 증가
  아키텍처 원칙 있음 → 의존성 방향 단방향 고정 → 변경 비용 O(1) 유지

핵심 공식:
  좋은 아키텍처 = 도메인이 인프라를 모르게 설계
  → 인프라 변경 시 도메인 코드 변경 없음
  → 도메인 테스트에 인프라 불필요
  → 새 인프라 추가 = Adapter 파일 하나 추가
```

---

## 🤔 생각해볼 문제

**Q1.** "아키텍처는 결정을 미루는 예술"이라는 말은 결정을 영원히 미루는 것인가? 어떤 결정은 초기에 반드시 내려야 하고, 어떤 결정은 미룰 수 있는가?

<details>
<summary>해설 보기</summary>

Uncle Bob의 의도는 "결정을 영원히 회피하라"가 아니라 "정보가 더 많아졌을 때 더 나은 결정을 내릴 수 있도록, 불필요하게 일찍 결정하지 마라"는 것입니다.

**초기에 반드시 내려야 하는 결정 (아키텍처 결정):**
- 의존성 방향: 도메인이 인프라를 아느냐 모르느냐
- 레이어 경계: 어디까지가 비즈니스 규칙이고 어디서부터가 기술 구현인가
- 팀 협업 규칙: 어느 레이어에 무엇을 놓을지의 기준

**나중으로 미룰 수 있는 결정 (기술 결정):**
- 어떤 DB를 쓸 것인가 (MySQL? PostgreSQL? MongoDB?)
- 어떤 메시징 시스템을 쓸 것인가 (Kafka? RabbitMQ?)
- 어떤 외부 API를 붙일 것인가 (KakaoPay? TossPay?)

도메인이 `OrderRepository` 인터페이스만 알고 있다면, JPA를 쓸지 MongoDB를 쓸지는 나중에 실제 데이터 특성이 파악된 후에 결정해도 됩니다. 초기에 인터페이스만 정의하고 InMemory 구현체로 개발을 시작할 수 있습니다.

</details>

---

**Q2.** 두 개발자가 논쟁 중이다. A: "우리 서비스는 단순 CRUD라 아키텍처 패턴이 필요 없다." B: "아키텍처 없이 시작하면 나중에 고치기 힘들다." 누가 맞는가?

<details>
<summary>해설 보기</summary>

둘 다 맞고 둘 다 틀렸습니다. 핵심은 **"현재 복잡도"와 "예상 성장"의 균형**입니다.

A가 맞는 상황:
- 실제로 영원히 단순 CRUD로 남을 서비스
- 프로토타입이나 POC
- 2주 내 폐기될 코드

B가 맞는 상황:
- "지금은 CRUD지만 6개월 후 비즈니스 규칙이 복잡해질 예정"
- 외부 시스템(결제, 알림, 배송) 연동이 있는 서비스
- 팀이 3명 이상이고 동시 개발 예정

현실적 조언: 완전한 Hexagonal Architecture를 처음부터 적용하지 않아도 됩니다. 최소한의 원칙 ("도메인 서비스가 JpaRepository에 직접 의존하지 않고, Repository 인터페이스에 의존한다") 하나만 지켜도 나중에 구조 개선이 훨씬 쉬워집니다. 점진적 아키텍처 적용이 현실적입니다.

</details>

---

**Q3.** Spring Boot 프로젝트에서 `@Service` 클래스가 `@Repository`에 직접 의존하는 것과, `Repository` 인터페이스에 의존하는 것의 실제 차이는 무엇인가? 둘 다 Spring이 Autowiring 해주는데 다를 게 있는가?

<details>
<summary>해설 보기</summary>

Spring이 런타임에 Autowiring 해준다는 점에서 실행은 동일합니다. 차이는 **테스트 가능성**과 **변경 비용**입니다.

```java
// Case A: 구체 클래스에 의존
@Service
public class OrderService {
    @Autowired
    private OrderJpaRepository repo; // 구체 클래스
}
// → 테스트 시 JPA + DB 컨테이너 필요
// → OrderJpaRepository를 다른 것으로 교체 시 OrderService 수정 필요

// Case B: 인터페이스에 의존
@Service
public class OrderService {
    private final OrderRepository repo; // 인터페이스
    public OrderService(OrderRepository repo) { this.repo = repo; }
}
// → 테스트 시: new OrderService(new InMemoryOrderRepository())
// → JPA → MongoDB 교체 시: OrderService 코드 변경 없음
```

런타임 동작은 같지만, **테스트 코드에서 InMemory 구현체를 주입할 수 있느냐**가 핵심 차이입니다. Case A는 `@SpringBootTest`가 강제되고, Case B는 순수 Java 테스트가 가능합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 변경 비용과 의존성 ➡️](./02-dependency-and-change-cost.md)**

</div>
