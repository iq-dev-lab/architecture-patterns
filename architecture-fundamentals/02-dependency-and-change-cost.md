# 변경 비용과 의존성 — 의존성이 변경 전파의 경로인 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 의존하는 쪽이 왜 의존받는 쪽의 변경에 영향을 받는가?
- Ripple Effect(파급 효과)란 무엇이고, 설계로 어떻게 통제하는가?
- 의존성 그래프를 그리면 변경 범위를 어떻게 예측할 수 있는가?
- 의존성 수가 줄어들수록 변경 비용이 낮아지는 이유는 무엇인가?
- Fan-out이 높은 클래스가 왜 아키텍처의 핵심 리스크인가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

"이 코드 건드리면 어디서 터질지 모르겠다"는 말의 원인은 항상 같다 — 의존성이 예측 불가능하게 퍼져있기 때문이다.

의존성은 변경이 전파되는 물리적 경로다. `A → B` 의존 관계가 있을 때, B가 변경되면 A도 영향을 받는다. 이 경로를 이해하고 설계하지 않으면, 코드베이스가 성장할수록 변경 하나가 연쇄적으로 수십 개 파일에 영향을 준다.

---

## 😱 흔한 실수 (Before — 의존성을 의식하지 않고 짤 때)

```
상황: "UserService에서 User의 이메일 검증 방식을 변경해주세요"

의존성을 모를 때의 코드 상태:
  UserRepository → User (JPA Entity, 이메일 필드 포함)
  UserService → UserRepository, EmailSender
  OrderService → UserService (주문 시 사용자 조회)
  PointService → UserService (포인트 적립 시 사용자 조회)
  AdminService → UserService (관리자 기능)
  UserController → UserService
  AdminController → AdminService → UserService
  NotificationService → UserService

"User 이메일 검증 방식 변경" 요청의 실제 영향:
  UserService 수정
  → UserService를 사용하는 OrderService 테스트 깨짐
  → PointService 테스트 깨짐
  → AdminService 테스트 깨짐
  → NotificationService 테스트 깨짐
  → 전체 테스트 스위트의 40%가 빨간불

개발자 체감:
  "이메일 검증 로직 하나 바꿨는데 왜 주문 테스트가 깨지지?"
  "이거 다 고치다가 퇴근 못 하겠다..."
  "Pull Request 올렸더니 리뷰어가 영향 범위 정리해오라고 한다"

이것이 Fan-out이 높을 때 생기는 문제:
  UserService가 너무 많은 곳에서 의존됨 (Fan-in이 높음)
  → UserService 변경 → 모든 의존자가 영향받음
  → 변경을 무서워하게 됨 → 코드를 건드리지 않게 됨 → 부채 누적
```

---

## ✨ 올바른 접근 (After — 의존성을 의식적으로 설계할 때)

```
같은 요청 — "User 이메일 검증 방식 변경"

의존성을 설계했을 때의 구조:

  domain/
    User.java             ← 이메일 검증 로직 (순수 도메인)
    UserRepository.java   ← 인터페이스

  application/
    FindUserUseCase.java  ← User 조회 전용
    UpdateUserUseCase.java ← User 수정 전용

"User 이메일 검증 방식 변경"의 실제 영향:
  User.java 내부의 validateEmail() 메서드만 수정
  → User.java를 직접 사용하는 테스트만 재확인
  → OrderService는 UserRepository를 통해 User를 조회하지만
     User의 이메일 검증 방식이 바뀌어도 OrderService 코드 변경 없음
  → 영향 범위: User.java 1개 + UserTest.java 1개

핵심 차이:
  Before: UserService라는 거대한 허브에 모든 서비스가 연결
          → 허브 변경 = 전체 영향

  After:  도메인(User)에 비즈니스 규칙, UseCase는 흐름만 담당
          → 비즈니스 규칙 변경 = 도메인만 영향
          → 각 UseCase는 필요한 Port만 의존하므로 독립적
```

---

## 🔬 내부 원리 — 의존성이 변경을 전파하는 메커니즘

### 1. 의존성의 정의와 방향

```
의존성(Dependency):
  A가 B를 사용하면 A는 B에 의존한다
  표기: A → B (A가 B에 의존)

의존성의 의미:
  A → B 관계에서:
    B가 변경되면 → A도 영향을 받을 수 있다
    A가 변경되면 → B는 영향을 받지 않는다

  즉, 의존성 화살표의 방향이 변경 전파의 방향이다

코드에서 의존성이 생기는 방법:
  ① 필드로 사용    class A { B b; }
  ② 파라미터 사용  void method(B b) {}
  ③ 반환 타입      B method() {}
  ④ 상속           class A extends B {}
  ⑤ 인터페이스 구현 class A implements B {}
  ⑥ 로컬 변수      void method() { B b = new B(); }
  ⑦ 정적 메서드    B.staticMethod();

모든 경우에서: B가 바뀌면 A 코드를 확인해야 한다
```

### 2. Ripple Effect — 의존성 그래프에서 변경이 전파되는 방식

```
의존성 그래프 예시:

  Controller ──→ Service ──→ Repository ──→ DB Schema
                    │
                    ├──→ ExternalApiClient ──→ 외부 API
                    │
                    └──→ CacheClient ──→ Redis

Repository.findById() 시그니처 변경 (Long id → UserId id):
  Repository 변경
    → Service가 Repository를 호출하는 코드 수정 필요
    → Controller가 Service를 호출하는 코드 수정 필요 (ID 타입 변환)
    → 관련 테스트 전부 수정

Ripple Effect (파급 효과):
  변경 1개 → 의존성 체인을 따라 파급 전파
  의존성 체인이 길수록 → 파급 범위가 넓어짐

파급 범위 계산:
  직접 의존: A → B (B 변경 시 A 확인 필요)
  간접 의존: A → B → C (C 변경 시 B, A 모두 확인 필요)
  이행적 의존: 의존성 체인의 모든 노드에 파급 가능

파급 범위를 줄이는 방법:
  ① 의존성 수 줄이기 — 필요 이상으로 의존하지 않기
  ② 의존성 방향 통제 — 단방향으로 유지
  ③ 인터페이스 도입 — 구현 변경이 의존자에게 전파되지 않도록
```

### 3. Fan-in과 Fan-out — 변경 리스크 측정

```
Fan-out: 어떤 클래스가 의존하는 클래스의 수
         = 내가 변경될 때 확인해야 하는 클래스 수

Fan-in:  어떤 클래스에 의존하는 클래스의 수
         = 내가 변경될 때 영향받는 클래스 수

예시:

OrderService.java:
  의존하는 것들 (Fan-out):
    OrderRepository
    UserService
    PaymentService
    NotificationService
    InventoryService
  Fan-out = 5

UserService.java를 의존하는 것들 (UserService의 Fan-in):
  OrderService
  PointService
  AdminService
  NotificationService
  UserController
  AdminController
  Fan-in = 6

리스크 분석:
  OrderService: Fan-out 5 → OrderService 수정 시 5개 의존성 확인
  UserService: Fan-in 6 → UserService 변경 시 6개 의존자 영향

  Fan-out이 높은 클래스 → 변경하기 어려움 (많은 의존성이 있음)
  Fan-in이 높은 클래스  → 변경 영향이 큼 (많은 곳에서 의존함)

아키텍처적 해결:
  Fan-out 줄이기:
    OrderService가 UserService에 직접 의존 X
    → OrderRepository를 통해 User 정보 조회
    또는 UserPort(인터페이스)에만 의존

  Fan-in이 높은 클래스 안정화:
    Interface로 추상화 → 구현 변경이 의존자에게 전파되지 않음
    UserRepository(인터페이스): Fan-in 높아도 OK
    → 인터페이스 시그니처만 유지하면 구현 변경 자유
```

### 4. 안정성 원칙 (Stable Dependencies Principle)

```
Uncle Bob의 안정성 원칙:
  "불안정한 것이 안정적인 것에 의존해야 한다"
  "변경이 잦은 것이 변경이 적은 것에 의존해야 한다"

안정성(Stability) 측정:
  안정성 = Fan-in / (Fan-in + Fan-out)
  0에 가까울수록 불안정, 1에 가까울수록 안정

예시:
  OrderRepository(인터페이스):
    Fan-in:  OrderService, OrderQueryService, AdminService = 3
    Fan-out: 0 (인터페이스는 아무것도 의존하지 않음)
    안정성 = 3 / (3 + 0) = 1.0 → 매우 안정적 ✓

  OrderService:
    Fan-in:  OrderController = 1
    Fan-out: OrderRepository, UserRepository, PaymentPort = 3
    안정성 = 1 / (1 + 3) = 0.25 → 불안정 (자주 바뀜)

올바른 의존 방향:
  OrderService(불안정, 0.25) → OrderRepository(안정, 1.0) ✓

잘못된 의존 방향:
  OrderRepository(안정) → OrderService(불안정) ✗
  → 안정적인 것이 불안정한 것에 의존 = 안정적인 것도 자주 바뀌어야 함

도메인이 인프라를 모르게 설계하는 이유:
  인프라(JPA, Kafka, REST Client) → 불안정 (자주 교체 가능)
  도메인(비즈니스 규칙) → 안정 (비즈니스 규칙은 잘 바뀌지 않음)

  잘못된 방향: 도메인 → 인프라 (안정이 불안정에 의존)
  올바른 방향: 인프라 → 도메인 (불안정이 안정에 의존)
```

### 5. 의존성 역전 — Ripple Effect를 차단하는 방법

```
문제 상황:
  OrderService → JpaOrderRepository

  JpaOrderRepository의 findByIdWithLock() 메서드 시그니처 변경
  → OrderService가 직접 의존하므로 OrderService 코드 수정 필요
  → OrderService를 사용하는 모든 테스트 재확인

의존성 역전 적용:
  Step 1: 인터페이스 추출
    interface OrderRepository {
        Optional<Order> findById(OrderId id);
    }

  Step 2: 구현체가 인터페이스를 구현
    class JpaOrderRepository implements OrderRepository {
        // 내부 구현: JPA 사용
    }

  Step 3: Service는 인터페이스에만 의존
    OrderService → OrderRepository(인터페이스)
                        ↑ 구현
                   JpaOrderRepository

  JpaOrderRepository 내부 변경 후:
  → OrderRepository 인터페이스 시그니처가 그대로면
  → OrderService는 변경할 필요 없음
  → Ripple Effect 차단!

이것이 DIP(Dependency Inversion Principle)의 핵심:
  "구체 클래스가 아닌 추상(인터페이스)에 의존하라"
  → 구현이 바뀌어도 의존자는 변경 없음
  → Ripple Effect를 인터페이스에서 차단
```

---

## 💻 실전 코드 — 의존성 그래프 분석과 개선

```java
// ❌ Before: 의존성이 콘크리트 클래스 방향으로 (Ripple Effect 높음)

@Service
public class OrderService {

    // 구체 클래스에 직접 의존 — Fan-out 높음
    @Autowired private OrderJpaRepository orderRepository;   // 구체 JPA
    @Autowired private UserService userService;              // 구체 Service
    @Autowired private KakaoPayClient kakaoPayClient;       // 구체 외부 API
    @Autowired private EmailSenderService emailSender;       // 구체 이메일

    public OrderResult placeOrder(Long userId, List<Long> itemIds) {
        // UserService 내부 구현에 의존
        User user = userService.findById(userId);

        // KakaoPayClient 메서드에 직접 의존
        PaymentResult payment = kakaoPayClient.charge(
            user.getPaymentKey(), calculateTotal(itemIds)
        );

        // OrderJpaRepository JPA 메서드에 의존
        Order order = new Order(user, itemIds, payment.getTransactionId());
        orderRepository.saveAndFlush(order); // JPA 구체 메서드

        // EmailSenderService 구현에 의존
        emailSender.sendOrderConfirmation(user.getEmail(), order);

        return new OrderResult(order.getId());
    }
}

// 문제:
// KakaoPayClient → TossPayClient 교체 시 OrderService 수정
// orderRepository.saveAndFlush → save 변경 시 OrderService 수정
// emailSender 구현 변경 시 OrderService 수정
// OrderService 테스트: User, KakaoPay, JPA, Email 전부 Mock 필요
```

```java
// ✅ After: 인터페이스에만 의존 — 의존성 역전으로 Ripple Effect 차단

// === Domain Ports (안정적 인터페이스) ===

public interface OrderRepository {          // Stable
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentPort {              // Stable
    PaymentResult charge(PaymentRequest request);
}

public interface OrderNotificationPort {    // Stable
    void notifyOrderPlaced(Order order);
}

public interface UserQueryPort {            // Stable
    User findById(UserId id);
}

// === Application Service (인터페이스에만 의존) ===

@Service
public class PlaceOrderService implements PlaceOrderUseCase {

    // 모두 인터페이스 — Fan-out은 있지만 안정적 추상에 의존
    private final OrderRepository orderRepository;
    private final PaymentPort paymentPort;
    private final OrderNotificationPort notificationPort;
    private final UserQueryPort userQueryPort;

    public PlaceOrderService(
        OrderRepository orderRepository,
        PaymentPort paymentPort,
        OrderNotificationPort notificationPort,
        UserQueryPort userQueryPort
    ) {
        this.orderRepository = orderRepository;
        this.paymentPort = paymentPort;
        this.notificationPort = notificationPort;
        this.userQueryPort = userQueryPort;
    }

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        User user = userQueryPort.findById(command.getUserId());

        Order order = Order.create(user, command.getItems()); // 도메인 규칙
        order.validate(); // 비즈니스 검증

        PaymentResult payment = paymentPort.charge(
            PaymentRequest.of(order, user.getPaymentMethod())
        );

        order.confirmPayment(payment);
        orderRepository.save(order);
        notificationPort.notifyOrderPlaced(order);

        return order.getId();
    }
}

// Ripple Effect 분석:
// KakaoPayAdapter → TossPayAdapter 교체:
//   PaymentPort 인터페이스 시그니처 유지 → PlaceOrderService 변경 없음
//
// JPA → MongoDB 교체:
//   OrderRepository 인터페이스 시그니처 유지 → PlaceOrderService 변경 없음
//
// PlaceOrderService 테스트 (인터페이스만 있으면 됨):
class PlaceOrderServiceTest {
    PaymentPort paymentPort = command -> PaymentResult.success("tx-001");
    OrderRepository repository = new InMemoryOrderRepository();
    OrderNotificationPort notification = order -> {}; // No-op
    UserQueryPort userQuery = id -> User.fixture(id);

    PlaceOrderService sut = new PlaceOrderService(
        repository, paymentPort, notification, userQuery
    );
    // Spring Context 없음, DB 없음, Kafka 없음
    // 테스트 실행 시간: ~10ms
}
```

---

## 📊 패턴 비교 — 의존성 설계 방식에 따른 변경 비용 비교

```
시나리오: "결제 실패 시 재시도 로직 추가"

=== 의존성 설계 없음 ===

OrderService (812줄):
  kakaoPayClient.charge() 실패 처리 코드가
  주문 생성 로직, 재고 차감, 이메일 발송 코드와 뒤섞여있음

변경 과정:
  1. 812줄 중 결제 관련 코드를 전부 찾아야 함
  2. 재시도 로직이 다른 코드와 영향을 주는지 확인
  3. 테스트: OrderService 전체 다시 테스트 (KakaoPay + JPA + Email 전부 Mock)
  4. 예상치 못한 side effect 발생 가능성 높음

변경 비용: 높음, 영향 범위: 예측 어려움

=== 의존성 설계 있음 (Hexagonal) ===

PaymentPort (인터페이스):
  charge(request) → 결제 처리

KakaoPayAdapter (PaymentPort 구현):
  charge() 안에 재시도 로직 추가 (RetryTemplate 또는 resilience4j)

변경 과정:
  1. KakaoPayAdapter.charge() 에만 재시도 로직 추가
  2. PaymentPort 인터페이스 시그니처 변경 없음
  3. PlaceOrderService 코드 변경 없음
  4. 테스트: KakaoPayAdapterTest만 추가 (Spring 없이 단위 테스트)

변경 비용: 낮음, 영향 범위: KakaoPayAdapter 1개
```

```
의존성 방향에 따른 변경 전파 비교:

=== 구체 클래스 의존 (변경이 퍼짐) ===

변경: JpaOrderRepository.findById() 반환 타입 변경

  JpaOrderRepository (변경됨)
       ↑ 의존
  OrderService (수정 필요)
       ↑ 의존
  OrderController (수정 필요)
       ↑ 의존
  관련 테스트 전부 (수정 필요)

변경 1개 → 파급 4개 이상

=== 인터페이스 의존 (변경이 차단됨) ===

변경: JpaOrderRepository 내부 구현 변경

  JpaOrderRepository (변경됨) → OrderRepository(인터페이스) 구현
  OrderRepository 인터페이스 (변경 없음) ← OrderService 의존

  OrderService (변경 없음!) ← 인터페이스 시그니처 유지
  OrderController (변경 없음!)

변경 1개 → 파급 0개 (인터페이스에서 차단)
```

---

## ⚖️ 트레이드오프

```
인터페이스 도입의 비용과 이익:

비용:
  ① 파일 수 증가 — 인터페이스 파일 + 구현 클래스 파일 별도
  ② 간접 참조 증가 — IDE에서 "구현체로 이동"이 한 단계 더 필요
  ③ 매핑 비용 — 인터페이스 메서드 시그니처 설계 노력

이익:
  ① Ripple Effect 차단 — 구현 변경이 의존자에게 전파 안 됨
  ② 테스트 격리 — InMemory 구현체로 단위 테스트 가능
  ③ 병렬 개발 — 인터페이스 정의 후 구현과 호출을 병렬로 개발

인터페이스가 필요 없는 경우:
  - 구현체가 하나뿐이고 교체 가능성이 없을 때
  - 프레임워크 제공 클래스 (Spring의 JdbcTemplate 등)
  - 값 객체 (Value Object) — 변경 방식이 정해진 경우

인터페이스가 반드시 필요한 경우:
  - 외부 시스템 연동 (결제, 알림, 외부 API)
  - 영속성 계층 (DB 교체 가능성)
  - 단위 테스트에서 교체가 필요한 모든 의존성
```

---

## 📌 핵심 정리

```
의존성과 변경 비용의 핵심:

의존성 = 변경이 전파되는 경로
  A → B 관계: B 변경 → A에 파급 가능
  의존성 체인이 길수록 → 파급 범위 넓어짐

Fan-out / Fan-in 분석:
  Fan-out 높음 → 변경 시 많은 의존성 확인 필요
  Fan-in 높음  → 변경 시 많은 의존자에게 파급

Ripple Effect 차단 방법:
  구체 클래스 의존 → 인터페이스 의존으로 변경
  인터페이스 시그니처가 유지되는 한 구현 변경이 전파되지 않음

안정성 원칙:
  불안정한 것(자주 바뀌는 구현체)이
  안정적인 것(인터페이스, 도메인)에 의존해야 함
  → 인프라(JPA, Kafka) → 도메인(Port 인터페이스) 방향이 올바름

핵심 공식:
  변경 비용 ∝ 의존성 수 × 의존성 체인 길이
  아키텍처 = 이 두 값을 최소화하는 설계
```

---

## 🤔 생각해볼 문제

**Q1.** `OrderService`가 `UserService`에 의존하는 것과, `UserRepository`에 직접 의존하는 것의 차이는 무엇인가? 어느 쪽이 의존성 관점에서 더 나은가?

<details>
<summary>해설 보기</summary>

일반적으로 **`UserRepository`(인터페이스)에 의존하는 것이 더 낫습니다.** 이유는 다음과 같습니다.

`OrderService → UserService` 의존 시:
- UserService의 모든 비즈니스 로직 변경이 OrderService에 영향
- OrderService 테스트에 UserService Mock 필요 (UserService가 복잡할수록 Mock도 복잡)
- 두 서비스 간 순환 의존성 위험 (UserService가 OrderService도 필요하게 되면?)

`OrderService → UserQueryPort(인터페이스)` 의존 시:
- OrderService가 필요한 사용자 조회 계약만 알면 됨
- UserService 내부 구현 변경이 OrderService에 전파되지 않음
- 테스트: `UserQueryPort` 람다 하나로 대체 가능

```java
// 더 나은 방식
public interface UserQueryPort {
    User findById(UserId id);
}

// OrderService는 User 조회 계약(UserQueryPort)에만 의존
// UserService의 다른 메서드(createUser, updatePassword 등)는 전혀 모름
```

</details>

---

**Q2.** 인터페이스를 도입하면 파일 수가 늘어난다. 그렇다면 모든 클래스에 인터페이스를 만들어야 하는가? 어떤 기준으로 인터페이스를 만들지 결정하는가?

<details>
<summary>해설 보기</summary>

**모든 클래스에 인터페이스를 만드는 것은 과잉입니다.** 인터페이스는 두 가지 목적 중 하나가 있을 때만 만듭니다.

**인터페이스를 만드는 기준:**
1. **교체 가능성** — 구현체를 다른 것으로 교체할 가능성이 있는가?
   - 결제 수단 (KakaoPay ↔ TossPay) → 인터페이스 필요
   - DB (JPA ↔ MongoDB) → 인터페이스 필요
   - 알림 (Email ↔ SMS) → 인터페이스 필요

2. **테스트 격리** — 단위 테스트에서 이 의존성을 교체해야 하는가?
   - 외부 API 호출 → 테스트에서 Fake로 교체 필요 → 인터페이스 필요
   - 순수 계산 로직 (Money 클래스의 add()) → 교체 불필요 → 인터페이스 불필요

**인터페이스가 불필요한 경우:**
- 변경 가능성이 없고 테스트에서 교체할 일도 없는 유틸리티 클래스
- 값 객체 (Value Object) — Money, Address 등
- Spring Framework 내부 클래스들 (이미 인터페이스로 추상화됨)

</details>

---

**Q3.** 순환 의존성(Circular Dependency)은 왜 위험한가? A → B → C → A 형태의 의존성이 있으면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

순환 의존성은 **"이 중 어느 것도 독립적으로 변경할 수 없다"** 상태를 만듭니다.

```
A → B → C → A (순환)

A를 변경하면: B에 파급 → C에 파급 → 다시 A에 파급 (무한 루프)
B를 변경하면: C에 파급 → A에 파급 → 다시 B에 파급

결과:
  ① 어느 하나를 단독으로 컴파일하거나 배포할 수 없음
  ② A 테스트 시 B, C가 전부 필요 → 사실상 통합 테스트만 가능
  ③ A, B, C 중 어느 하나가 깨지면 셋 다 영향
```

순환 의존성을 끊는 방법:
1. **의존성 역전** — C → A 방향을 `C → APort(인터페이스) ← A`로 변경
2. **공통 모듈 추출** — A와 C가 모두 의존하는 공통 인터페이스를 별도 모듈로 분리
3. **이벤트 기반** — C가 A를 직접 호출하는 대신 이벤트를 발행, A가 구독

Spring에서는 `@Lazy`로 순환 의존성을 임시 해결할 수 있지만, 이는 문제를 숨기는 것이지 해결이 아닙니다. ArchUnit으로 순환 의존성을 CI에서 자동 감지하는 것이 권장됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: 아키텍처의 본질](./01-what-is-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: SOLID 원칙과 아키텍처 ➡️](./03-solid-and-architecture.md)**

</div>
