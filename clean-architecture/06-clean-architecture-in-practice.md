# Clean Architecture 실전 — Spring Boot에서의 현실적 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Spring Boot에서 4개 레이어를 엄격히 분리한 패키지 구조는 어떻게 생겼는가?
- Entity에 JPA 어노테이션을 허용할지 말지를 어떤 기준으로 결정하는가?
- UseCase 레이어에서 `@Transactional`을 허용하는 이유와 그 절충점은?
- 팀이 반드시 합의해야 하는 결정들은 무엇이고, ADR로 어떻게 문서화하는가?
- Hexagonal Architecture와 비교해 Clean Architecture를 선택해야 하는 상황은?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Clean Architecture 이론을 알아도 Spring Boot 프로젝트에서 실제로 어떻게 파일을 배치하고 어디에 어떤 어노테이션을 붙일지 모르면 구현이 막힌다.

더 중요한 것은 "이론을 100% 따를 것인가"의 결정이다. 팀이 사전에 합의 없이 각자 다른 해석으로 코드를 작성하면, Clean Architecture 구조를 갖췄지만 혼재된 코드가 된다. 이 문서는 그 합의 지점을 명확히 한다.

---

## 😱 흔한 실수 (Before — 팀 합의 없이 각자 다른 해석)

```
팀 내 혼재:
  개발자 A: "Entities에 @Entity 붙여도 돼, 어차피 JPA 안 쓸 일 없잖아"
  개발자 B: "Use Cases가 Spring 몰라야 하니까 @Transactional 못 씀"
  개발자 C: "Presenter 패턴은 너무 복잡해서 직접 반환으로"
  개발자 D: "Gateway 이름 안 써도 돼, 그냥 Repository로"

6개월 후 코드베이스:
  Order.java: @Entity 있는 버전, 없는 버전 혼재
  UseCase: @Transactional 있는 것, 없는 것 혼재
  Presenter: 있는 곳, 없는 곳 혼재
  Gateway 이름: JpaOrderGateway, JpaOrderRepository, OrderRepositoryImpl 혼재

결과:
  "이 파일은 어느 레이어 스타일이지?"
  신규 팀원 온보딩 시 "어느 것이 표준이에요?" 혼란
  ArchUnit 규칙 작성이 어려움 (기준이 없어서)
```

---

## ✨ 올바른 접근 (After — 팀 합의 + 문서화 + ArchUnit 강제)

```
팀이 사전에 합의해야 할 결정 목록:

① @Entity 분리 여부
   결정: "Domain Entity와 JPA Entity를 분리한다"
   이유: JPA 제약(기본 생성자, LAZY)이 도메인 설계를 오염시키기 시작할 때
   절충: 매핑 코드(Mapper) 추가

② @Transactional 위치
   결정: "Use Cases 레이어(Interactor)에 붙인다"
   이유: 트랜잭션 경계 = UseCase 경계가 자연스러운 매핑
   절충: Use Cases가 Spring에 의존하게 됨

③ Presenter 패턴 사용 여부
   결정: "Query UseCase는 직접 반환, Command UseCase는 선택적으로 Output Boundary"
   이유: 대부분의 REST API에서 Presenter 복잡도가 이익보다 큼
   절충: UseCase가 반환 타입을 가짐 (표현 형식에 약간 의존)

④ 패키지 구조 기준
   결정: "기능(Feature) 기준으로 분리, 각 기능 내부에 4개 레이어"
   이유: 기능별 응집도가 레이어별 응집도보다 실용적

⑤ 테스트 전략
   결정: "Domain 단위 50%, UseCase 단위 30%, Gateway 통합 15%, E2E 5%"

각 결정을 ADR(Architecture Decision Record)로 문서화
```

---

## 🔬 내부 원리 — 실전 패키지 구조와 결정 포인트

### 1. 권장 패키지 구조

```
com.example
├── order/                                ← Order 기능 (Bounded Context)
│   ├── entity/                           ← Entities 레이어
│   │   ├── Order.java                    ← Domain Entity (순수 Java)
│   │   ├── OrderLine.java               ← 내부 Entity
│   │   ├── OrderStatus.java             ← Enum + 전이 규칙
│   │   ├── Money.java                   ← Value Object
│   │   └── OrderId.java                 ← Value Object (Type-safe ID)
│   │
│   ├── usecase/                          ← Use Cases 레이어
│   │   ├── PlaceOrderInputBoundary.java  ← Input Port (인터페이스)
│   │   ├── PlaceOrderRequestModel.java  ← UseCase 전용 입력
│   │   ├── PlaceOrderResponseModel.java ← UseCase 전용 출력
│   │   ├── PlaceOrderOutputBoundary.java ← Output Port (인터페이스)
│   │   ├── PlaceOrderInteractor.java    ← UseCase 구현 (@Service, @Transactional)
│   │   ├── FindOrderQuery.java          ← Query UseCase 인터페이스
│   │   ├── FindOrderInteractor.java     ← Query UseCase 구현
│   │   ├── gateway/                     ← Gateway 인터페이스 (Output Port)
│   │   │   ├── OrderGateway.java
│   │   │   └── PaymentGateway.java
│   │   └── model/                       ← Gateway 입출력 모델
│   │       ├── PaymentRequestModel.java
│   │       └── PaymentResponseModel.java
│   │
│   ├── adapter/                          ← Interface Adapters 레이어
│   │   ├── in/
│   │   │   ├── web/
│   │   │   │   ├── OrderController.java  ← HTTP Controller
│   │   │   │   ├── request/
│   │   │   │   │   └── PlaceOrderRequest.java  ← HTTP DTO
│   │   │   │   ├── response/
│   │   │   │   │   └── OrderViewModel.java     ← HTTP Response DTO
│   │   │   │   └── mapper/
│   │   │   │       └── OrderWebMapper.java     ← HTTP ↔ UseCase 모델 변환
│   │   │   └── event/
│   │   │       └── OrderEventConsumer.java     ← Kafka Consumer (Driving Adapter)
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── JpaOrderGateway.java         ← OrderGateway 구현
│   │       │   └── mapper/
│   │       │       └── OrderGatewayMapper.java  ← Domain ↔ JPA 변환
│   │       └── payment/
│   │           └── KakaoPayPaymentGateway.java  ← PaymentGateway 구현
│   │
│   └── framework/                        ← Frameworks & Drivers 레이어
│       ├── persistence/
│       │   ├── entity/
│       │   │   └── OrderJpaEntity.java   ← @Entity (JPA 전용)
│       │   └── SpringDataOrderJpa.java  ← JpaRepository 상속
│       └── config/
│           └── OrderConfiguration.java  ← Spring Bean 설정
│
└── shared/                               ← 공유 컴포넌트
    ├── domain/                           ← 공유 Value Object
    │   └── Money.java
    └── infrastructure/
        └── KafkaConfig.java
```

### 2. @Entity 분리 여부 결정 가이드

```
결정 트리:

Q1: JPA의 기본 생성자가 도메인 설계를 방해하는가?
  YES → 분리 필요
    (Order가 기본 생성자 없어야 하는 비즈니스 이유가 있을 때)
  NO → 절충 가능

Q2: LAZY 로딩이 비즈니스 메서드를 깨뜨리는가?
  YES → 분리 필요
    (calculateTotal()에서 LAZY 컬렉션 접근 → LazyInitializationException)
  NO → 절충 가능

Q3: @Column, @Table이 도메인 클래스의 가독성을 해치는가?
  심각하게 YES → 분리 권장
  NO → 절충 허용

분리 결정 시:
  // Domain Entity (entity/ 패키지)
  public class Order { /* @Entity 없음, 기본 생성자 없음 */ }
  
  // JPA Entity (framework/persistence/entity/ 패키지)
  @Entity @Table(name = "orders")
  public class OrderJpaEntity { /* @Entity, 기본 생성자 protected */ }
  
  // Mapper (adapter/out/persistence/mapper/ 패키지)
  @Component
  public class OrderGatewayMapper { /* 양방향 변환 */ }

절충 결정 시:
  // @Entity 허용하되 비즈니스 메서드는 반드시 유지
  @Entity @Table(name = "orders")
  public class Order {
      protected Order() {} // JPA 요구, private하게는 불가능
      
      public void place() { /* 비즈니스 메서드 유지 */ }
  }
  // 허용 조건: JPA 제약이 비즈니스 메서드를 깨뜨리지 않을 때
```

### 3. ADR(Architecture Decision Record) 예시

```markdown
# ADR-001: Repository 인터페이스를 Use Cases 레이어에 위치

## 상태
승인됨 (2024-03-15, 팀 전체 합의)

## 맥락
주문 기능 개발 중 OrderRepository 인터페이스를 어느 레이어에 위치시킬지
논의가 필요했습니다. 두 가지 옵션이 제안됨:
  A. 기존 방식: repository/ 패키지 (Interface Adapters 또는 별도 패키지)
  B. Clean Architecture 방식: usecase/gateway/ 패키지 (Use Cases 레이어)

## 결정
옵션 B 선택: OrderGateway 인터페이스를 usecase/gateway/ 패키지에 위치

## 이유
1. 의존성 방향: PlaceOrderInteractor(Use Cases) → OrderGateway(Interface Adapters)면
   Use Cases가 Interface Adapters에 의존 = 의존성 규칙 위반
   
2. Gateway를 Use Cases에 두면:
   PlaceOrderInteractor(Use Cases) → OrderGateway(Use Cases) = 같은 레이어
   JpaOrderGateway(Interface Adapters) → OrderGateway(Use Cases) = 안쪽 의존 ✓
   
3. 테스트 이익: In-Memory Gateway로 UseCase 단위 테스트 가능

## 결과
긍정적:
  - 의존성 규칙 준수 가능
  - UseCase 단위 테스트 속도 향상

부정적:
  - Gateway 인터페이스 파일이 usecase/ 패키지에 있어 처음엔 어색함
  - 신규 팀원 설명 필요

## 관련 결정
ADR-002: @Transactional 위치 결정
ADR-003: Domain Entity와 JPA Entity 분리 여부
```

### 4. ArchUnit으로 결정 강제

```java
@AnalyzeClasses(packages = "com.example")
public class CleanArchitectureTest {

    JavaClasses classes = new ClassFileImporter().importPackages("com.example");

    // 규칙 1: Entities가 어떤 외부도 의존하지 않음
    @Test
    void entities_have_no_outward_dependencies() {
        noClasses()
            .that().resideInAPackage("..entity..")
            .should().dependOnClassesThat()
            .resideInAnyPackage(
                "org.springframework..",
                "jakarta.persistence..",
                "com.fasterxml.jackson.."
            )
            .check(classes);
    }

    // 규칙 2: Use Cases가 Frameworks에 의존하지 않음
    @Test
    void use_cases_do_not_depend_on_frameworks() {
        noClasses()
            .that().resideInAPackage("..usecase..")
            .and().areNotAnnotatedWith(Transactional.class) // @Transactional 예외
            .should().dependOnClassesThat()
            .resideInAnyPackage(
                "..adapter..",
                "..framework..",
                "org.springframework.web..",
                "jakarta.persistence.."
            )
            .check(classes);
    }

    // 규칙 3: Gateway 인터페이스가 Use Cases 레이어에 위치
    @Test
    void gateway_interfaces_reside_in_usecase_layer() {
        classes()
            .that().haveNameMatching(".*Gateway")
            .and().areInterfaces()
            .should().resideInAPackage("..usecase.gateway..")
            .check(classes);
    }

    // 규칙 4: Adapter가 Entities를 직접 생성하지 않음 (Gateway를 통해서만)
    @Test
    void adapters_do_not_create_entities_directly() {
        noClasses()
            .that().resideInAPackage("..adapter.in..")
            .should().dependOnClassesThat()
            .resideInAPackage("..entity..")
            .check(classes);
        // Controller가 Order를 직접 생성하면 안 됨
        // Request Model → Interactor → Order 생성
    }
}
```

---

## 💻 실전 코드 — 합의된 결정이 반영된 전체 예시

```java
// === 합의된 결정 반영 코드 예시 ===

// 결정①: @Entity 분리 (도메인 설계 보호)
// entity/Order.java
public class Order {
    private final OrderId id;
    private final List<OrderLine> lines;
    private OrderStatus status;

    // 기본 생성자 없음 → 불완전 생성 불가
    private Order(OrderId id, List<OrderLine> lines, OrderStatus status) { ... }
    public static Order create(UserId userId, List<OrderLine> lines) { ... }
    public void place() { /* 비즈니스 규칙 */ }
}

// framework/persistence/entity/OrderJpaEntity.java
@Entity @Table(name = "orders")
@NoArgsConstructor(access = AccessLevel.PROTECTED) // JPA 요구
public class OrderJpaEntity {
    @Id @GeneratedValue private Long id;
    @Column(name = "order_id", unique = true) private String orderId;
    // ... JPA 매핑 전용 필드
}

// 결정②: @Transactional은 Use Cases에 (Spring 의존 절충 허용)
// usecase/PlaceOrderInteractor.java
@Service                  // Spring 의존 허용 (절충)
@Transactional            // Use Cases 레이어 트랜잭션 경계
public class PlaceOrderInteractor implements PlaceOrderInputBoundary {

    private final OrderGateway orderGateway;   // Gateway 인터페이스
    private final PaymentGateway paymentGateway;

    @Override
    public void execute(PlaceOrderRequestModel req, PlaceOrderOutputBoundary out) {
        Order order = Order.create(UserId.of(req.userId()), mapLines(req.items()));
        order.place();

        PaymentResponseModel payment = paymentGateway.charge(
            new PaymentRequestModel(order.getId().value(), order.calculateTotal().getValue())
        );
        order.confirmPayment(payment.transactionId());

        orderGateway.save(order);

        out.present(new PlaceOrderResponseModel(
            order.getId().value(),
            order.calculateTotal().toString()
        ));
    }
}

// 결정③: Command UseCase는 Output Boundary, Query UseCase는 직접 반환
// usecase/FindOrderInteractor.java (Query - 직접 반환)
@Service
public class FindOrderInteractor implements FindOrderQuery {

    private final OrderGateway orderGateway;

    @Override
    public OrderDetail findById(String orderId) {
        Order order = orderGateway.findById(OrderId.of(orderId))
            .orElseThrow(OrderNotFoundException::new);
        return OrderDetail.from(order); // 직접 반환 (Presenter 없음)
    }
}
```

---

## 📊 패턴 비교 — Clean Architecture vs Hexagonal 선택 기준

```
Clean Architecture 선택이 더 적합한 경우:
  ✅ 팀이 엄격한 레이어 분리를 원할 때
  ✅ 여러 UI 채널(Web, Mobile, CLI)을 지원해야 할 때
  ✅ Presenter 패턴으로 View 변환을 완전히 격리해야 할 때
  ✅ Uncle Bob의 이론적 기반에 따른 명확한 네이밍을 원할 때

Hexagonal Architecture가 더 적합한 경우:
  ✅ REST API 위주의 서비스 (Presenter 패턴 불필요)
  ✅ 팀이 Port/Adapter 개념에 더 친숙할 때
  ✅ 개념 단순성이 중요할 때 (Port/Adapter가 더 직관적)
  ✅ 실무 예제와 오픈소스가 더 많음

대부분의 실무:
  두 패턴을 혼합 (Hexagonal + Clean 아이디어)
  Driving Port(UseCase 인터페이스) + Driven Port(Gateway 인터페이스)
  Presenter는 생략하고 직접 반환
  의존성 방향은 Clean Architecture 원칙 준수
```

---

## ⚖️ 트레이드오프

```
Clean Architecture 완전 적용 비용:
  파일 수: Hexagonal보다 더 많음 (Presenter, Request/Response Model 추가)
  개념 복잡도: Interactor, Gateway, Input/Output Boundary 등 용어 많음
  Presenter 패턴: Thread Safety 고려 필요, 코드 흐름 복잡

현실적 절충의 결과:
  Presenter 제거 → "Clean Architecture 80% + Hexagonal 방식 혼합"
  @Transactional 허용 → "Use Cases에 Spring 약간 의존"
  @Entity 허용 → "Entities에 JPA 약간 의존"
  
  이 절충들이 쌓이면 결국 "Hexagonal Architecture"와 유사해짐
  → 처음부터 Hexagonal을 선택해도 될 수 있음
  
팀 합의가 없으면 아키텍처가 없는 것과 같음:
  아키텍처 패턴 선택보다 팀 합의와 ArchUnit 강제가 더 중요
```

---

## 📌 핵심 정리

```
Clean Architecture 실전 적용 핵심:

패키지 구조:
  feature/entity/ → Entities (순수 Java)
  feature/usecase/ → Use Cases (Gateway 인터페이스 포함)
  feature/adapter/in|out/ → Interface Adapters
  feature/framework/ → Frameworks & Drivers (@Entity, JPA 설정)

팀 합의 결정 사항:
  ① @Entity 분리 여부 (JPA 제약이 도메인 오염 시 분리)
  ② @Transactional 위치 (Use Cases 허용 권장)
  ③ Presenter 사용 여부 (대부분 생략, 직접 반환)
  ④ 테스트 전략 (Domain 단위 50% 목표)

ADR로 문서화:
  결정, 이유, 결과를 기록
  ArchUnit으로 결정을 코드로 강제

Hexagonal vs Clean 선택:
  두 방식의 차이보다 팀 합의와 일관성이 중요
  실무에서 두 패턴의 혼합이 일반적
```

---

## 🤔 생각해볼 문제

**Q1.** Clean Architecture를 처음 적용하는 팀이 첫 번째 ADR(Architecture Decision Record)로 작성해야 할 가장 중요한 결정은 무엇인가?

<details>
<summary>해설 보기</summary>

**가장 먼저 결정해야 할 것: "Gateway 인터페이스를 어느 레이어에 위치시킬 것인가"**

이 결정이 의존성 방향 전체를 결정하기 때문입니다:
- Use Cases에 위치: 의존성 규칙 완전 준수 가능
- Interface Adapters에 위치: Use Cases → Interface Adapters 의존 발생

두 번째 중요 결정: "@Entity 분리 여부"
- 분리: 더 순수하지만 매핑 비용
- 절충: 빠른 개발이지만 Entities 오염

이 두 결정이 나머지 결정들의 방향을 잡아줍니다. 나머지(Presenter, @Transactional)는 첫 두 결정 이후에 자연스럽게 따라옵니다.

</details>

---

**Q2.** Clean Architecture를 6개월 적용한 팀이 "너무 복잡하다"고 느낀다. 어떻게 진단하고 개선하는가?

<details>
<summary>해설 보기</summary>

**진단 체크리스트:**

1. **단순 CRUD에도 완전 적용했는가?** → 공지사항 같은 단순 기능에 모든 레이어를 적용하면 오버헤드가 큼. 핵심 도메인만 적용.

2. **Presenter 패턴을 모든 곳에 적용했는가?** → 생략하고 직접 반환으로 단순화.

3. **Request/Response Model이 HTTP DTO와 완전히 다른가?** → 처음엔 같아도 됨. 달라질 때 분리.

4. **팀이 개념(Interactor, Gateway, Input/Output Boundary)에 익숙한가?** → 용어를 Hexagonal의 Port/Adapter로 통일하는 것도 방법.

**개선 방향:**
- 단순 기능: 레이어드 Level 1으로 단순화
- 복잡한 핵심 도메인: Clean Architecture 유지
- Presenter: 생략하고 직접 반환
- 결과적으로 "Hexagonal + Clean 혼합" 형태로 수렴

아키텍처는 목표(변경 비용 최소화, 테스트 용이성)를 위한 수단. 복잡도가 이익보다 크면 단순화가 옳습니다.

</details>

---

**Q3.** Woowacourse 환경처럼 외부 DB 없이 콘솔 I/O만 있는 미션에서 Clean Architecture를 적용하면 어떤 형태가 되는가?

<details>
<summary>해설 보기</summary>

**콘솔 프로그램의 Clean Architecture:**

```
Entities:
  Order, Board (Janggi), Piece 등 순수 도메인

Use Cases:
  PlaceOrderUseCase / PlaceMoveUseCase 등
  Gateway 인터페이스: OrderRepository (인메모리)

Interface Adapters:
  ConsoleController → UseCase 호출
  ConsolePresenter → ViewModel(문자열 출력 형식)
  InMemoryOrderGateway → 메모리 저장

Frameworks:
  main() → 의존성 조립
  InputView / OutputView (콘솔 I/O)
```

실제 적용 형태:
```java
// main (Frameworks & Drivers)
public class Application {
    public static void main(String[] args) {
        InMemoryOrderRepository repo = new InMemoryOrderRepository();
        PlaceOrderUseCase useCase = new PlaceOrderInteractor(repo);
        ConsoleController controller = new ConsoleController(useCase, new InputView());
        controller.run();
    }
}
```

Woowacourse 미션에서의 현실:
- Entities + Use Cases + 얇은 Adapter로 충분
- "도메인이 InputView를 모른다"는 핵심 원칙만 지켜도 Clean Architecture의 80% 이익
- Presenter 패턴까지 완전 적용하면 과잉 (콘솔 출력이 View의 전부이므로)

</details>

---

<div align="center">

**[⬅️ 이전: Frameworks & Drivers 레이어](./05-frameworks-and-drivers.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 아키텍처 패턴 비교와 선택 ➡️](../architecture-comparison/01-layered-vs-hexagonal-vs-clean.md)**

</div>
