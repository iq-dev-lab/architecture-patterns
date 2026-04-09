# 엄격한 레이어 vs 느슨한 레이어 — Strict vs Relaxed

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Strict Layer와 Relaxed Layer는 구체적으로 어떻게 다른가?
- Relaxed Layer에서 레이어 건너뛰기를 허용했을 때 어떤 문제가 생기는가?
- 실무에서 두 방식을 어떻게 절충하는가?
- ArchUnit으로 레이어 규칙을 코드로 강제하는 방법은 무엇인가?
- "읽기는 Controller → Repository 직접"이 괜찮다고 생각하는 이유와 그 문제점은?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

"읽기 API는 Service 없이 Controller에서 Repository를 직접 쓰면 안 돼요? 간단한 건데." — 이 말은 레이어드 아키텍처 프로젝트에서 가장 자주 들리는 질문 중 하나다.

이것이 Relaxed Layer다. 허용하면 개발 속도가 빠르지만, 허용 범위가 점점 확장되면서 레이어의 의미가 사라진다. 두 방식의 트레이드오프를 정확히 이해해야 팀이 일관된 기준을 가질 수 있다.

---

## 😱 흔한 실수 (Before — 레이어 규칙을 팀 내 합의 없이 운영할 때)

```
팀 내 혼재 상태:
  개발자 A: "단순 조회는 Controller → Repository 직접 써도 됨"
  개발자 B: "모든 경우에 Service를 통해야 함"
  개발자 C: "빠른 개발을 위해 Controller에서 다 해도 됨"

6개월 후 코드베이스:
  OrderController:
    - 주문 생성: PlaceOrderService 호출
    - 주문 조회: OrderRepository 직접 호출
    - 주문 취소: CancelOrderService 호출
    - 주문 목록: OrderRepository 직접 호출

  ProductController:
    - 전체 조회: ProductRepository 직접 (A의 방식)
    - 단건 조회: ProductService 경유 (B의 방식)
    - 재고 확인: InventoryService + InventoryRepository 혼용 (C의 방식)

결과:
  "새 기능을 어떻게 구현해야 하지?" 매번 판단이 다름
  Repository에 비즈니스 로직이 조금씩 스며들기 시작
  "왜 어떤 건 Service를 쓰고 어떤 건 Repository를 직접 쓰지?" 신규 팀원 혼란
  나중에 비즈니스 로직이 필요해졌을 때 Controller에 이미 있어서 빼기 힘듦
```

---

## ✨ 올바른 접근 (After — 팀 합의로 명확한 규칙을 정하고 강제할 때)

```
명확한 선택과 근거:

Option A: Strict Layer 적용
  규칙: "인접한 레이어만 호출 가능"
  근거: 레이어 경계가 명확해야 장기적 유지보수성 확보
  적합: 복잡한 비즈니스, 장기 프로젝트, 중대형 팀

Option B: Relaxed Layer (부분 허용)
  규칙: "조회 API에 한해 Controller → Repository 직접 허용,
         단 비즈니스 로직은 반드시 Service를 통해야 함"
  근거: 단순 조회는 Service 레이어가 불필요한 보일러플레이트
  적합: CQRS 패턴 적용 시, Query가 압도적으로 많을 때

어느 방식이든 핵심: ArchUnit으로 코드에서 규칙을 강제하라
  → 사람이 코드 리뷰에서 체크하는 것보다 자동 검증이 훨씬 효과적
```

---

## 🔬 내부 원리 — Strict vs Relaxed의 구조적 차이

### 1. Strict Layer — 인접 레이어만 호출 가능

```
구조:
  Presentation → Application → Domain ← Infrastructure

규칙:
  Presentation은 Application만 호출 가능
  Application은 Domain과 Infrastructure만 호출 가능
  Domain은 아무것도 호출하지 않음 (순수)
  Infrastructure는 Domain을 구현

금지 사항:
  Controller → Repository 직접 (Presentation이 Infrastructure 건너뜀)
  Controller → Domain Entity 직접 (Presentation이 Application 건너뜀)
  Application → Controller (하위가 상위를 호출)

장점:
  각 레이어의 역할이 명확히 유지됨
  레이어 경계 위반이 즉시 발견됨 (ArchUnit으로)
  새 기능 추가 시 어디에 코드를 넣을지 명확

단점:
  단순 조회에도 Application Layer를 거쳐야 함
  보일러플레이트 코드 증가 (위임 메서드)
  개발 속도 초기에 느림

적합한 상황:
  비즈니스 로직이 복잡한 서비스
  팀 규모가 크고 일관성이 중요할 때
  장기 운영 예정인 서비스
```

### 2. Relaxed Layer — 모든 하위 레이어 호출 가능

```
구조:
  Presentation → Application, Domain, Infrastructure 모두 접근 가능
  Application → Domain, Infrastructure 모두 접근 가능

일반적 허용 패턴:
  조회(Query): Controller → Repository 직접
  변경(Command): Controller → Service → Repository

이것이 CQRS(Command Query Responsibility Segregation)의 단순 버전:
  Command (쓰기): 비즈니스 로직이 필요 → Service를 통함
  Query (읽기): 단순 조회 → Repository 직접 접근도 허용

장점:
  단순 조회의 보일러플레이트 제거
  개발 속도 향상

단점:
  "단순 조회"의 기준이 모호해짐 → 허용 범위가 점점 확장
  나중에 조회에 비즈니스 로직이 추가될 때 Controller에 있어서 빼기 어려움
  레이어 의미가 흐려짐

적합한 상황:
  Command와 Query가 명확히 분리될 때 (CQRS 적용)
  단순 조회가 압도적으로 많은 서비스
  명확한 팀 내 합의와 ArchUnit 규칙이 있을 때
```

### 3. Relaxed Layer가 무너지는 과정

```
Day 1:
  "단순 조회는 Repository 직접 써도 됨"
  OrderController → OrderRepository.findById() ← OK

Month 2:
  "주문 목록 조회할 때 사용자 권한 체크해야 해"
  OrderController → OrderRepository.findById()
                  + UserRepository.findById() (권한 체크용)
  → Controller에 비즈니스 로직(권한 체크) 추가됨

Month 4:
  "주문 조회 시 쿠폰 적용 여부도 보여줘야 해"
  OrderController → OrderRepository.findById()
                  + UserRepository.findById()
                  + CouponRepository.findByOrderId()
                  + 쿠폰 적용 여부 계산 로직 (Controller에)
  → Controller가 200줄

Month 6:
  "이 조회 로직을 다른 Controller에서도 써야 해"
  → private 메서드로 빼거나 Controller 간 의존 발생
  → "Service가 필요한데 이미 Controller에 로직이 박혀있다"

패턴:
  "단순"이라고 생각했던 조회가 시간이 지나면서 복잡해짐
  그 복잡함이 Controller에 쌓임
  나중에 빼기 어려워짐
```

### 4. ArchUnit으로 레이어 규칙 강제

```java
// Strict Layer 규칙을 ArchUnit으로 강제

@AnalyzeClasses(packages = "com.example")
public class LayeredArchitectureTest {

    JavaClasses importedClasses = new ClassFileImporter()
        .importPackages("com.example");

    @Test
    void presentation_layer_should_not_access_infrastructure_directly() {
        // Controller가 Repository를 직접 호출하는 것 금지
        noClasses()
            .that().resideInAPackage("..controller..")
            .should().dependOnClassesThat()
            .resideInAPackage("..repository..")
            .check(importedClasses);
    }

    @Test
    void presentation_layer_should_not_access_domain_directly() {
        // Controller가 Domain Entity를 직접 생성하는 것 금지
        // (Command 객체를 통해야 함)
        noClasses()
            .that().resideInAPackage("..controller..")
            .should().dependOnClassesThat()
            .resideInAPackage("..domain..")
            .withDescriptionContaining("Service가 아닌 Domain 직접 접근")
            .check(importedClasses);
    }

    @Test
    void domain_layer_should_not_depend_on_infrastructure() {
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..")
            .check(importedClasses);
    }

    @Test
    void domain_layer_should_not_use_spring_annotations() {
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().beAnnotatedWith(Component.class)
            .orShould().beAnnotatedWith(Service.class)
            .orShould().beAnnotatedWith(Repository.class)
            .check(importedClasses);
    }

    // ArchUnit 내장 레이어드 아키텍처 검증
    @Test
    void layered_architecture_should_be_respected() {
        layeredArchitecture()
            .consideringAllDependencies()
            .layer("Presentation").definedBy("..controller..")
            .layer("Application").definedBy("..service..")
            .layer("Domain").definedBy("..domain..")
            .layer("Infrastructure").definedBy("..repository..", "..adapter..")
            .whereLayer("Presentation").mayOnlyBeAccessedByLayers() // 외부에서 접근 불가
            .whereLayer("Application").mayOnlyBeAccessedByLayers("Presentation")
            .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure")
            .whereLayer("Infrastructure").mayOnlyBeAccessedByLayers("Application")
            .check(importedClasses);
    }
}
```

### 5. 실무 절충: 계층화된 Relaxed 규칙

```
실무에서 많이 쓰이는 절충안:

규칙 1: Command (쓰기 작업)
  항상 Controller → UseCase/Service → Domain → Repository
  이유: 비즈니스 로직 필요, 트랜잭션 필요

규칙 2: Query (읽기 작업) — 두 가지 옵션
  Option A (엄격): Controller → QueryService → Repository
    QueryService: 단순히 Repository를 호출하고 DTO로 변환
    장점: 일관성 유지
    단점: 보일러플레이트 QueryService 다수 생성

  Option B (느슨): Controller → QueryFacade → Repository
    QueryFacade: 여러 Repository를 조합하는 얇은 레이어
    장점: 복잡한 조회도 Controller에서 분리
    단점: Service와 QueryFacade의 경계가 모호해질 수 있음

핵심 원칙 (어느 방식이든):
  비즈니스 로직(판단, 계산)은 Domain에
  조율 로직(순서, 조합)은 Application에
  HTTP 로직은 Presentation에
  DB 접근은 Infrastructure에

이 원칙이 지켜진다면 Relaxed여도 큰 문제 없음
이 원칙이 안 지켜지면 Strict여도 의미 없음
```

---

## 💻 실전 코드 — Strict vs Relaxed 구현 비교

```java
// === Strict Layer 예시 ===

// 단순 조회도 Application Layer 통과
@RestController
public class OrderController {

    private final FindOrderUseCase findOrderUseCase; // Application Layer

    @GetMapping("/orders/{id}")
    public OrderResponse findOrder(@PathVariable String id) {
        // Controller는 Application만 호출
        return findOrderUseCase.findById(OrderId.of(id));
    }
}

@Service  // Application Layer
public class FindOrderService implements FindOrderUseCase {

    private final OrderRepository orderRepository; // Domain Port

    @Override
    public OrderResponse findById(OrderId id) {
        return orderRepository.findById(id)
            .map(OrderResponse::from)
            .orElseThrow(OrderNotFoundException::new);
    }
}

// === Relaxed Layer 예시 (CQRS 스타일) ===

@RestController
public class OrderController {

    private final PlaceOrderUseCase placeOrderUseCase; // Command: Service 통과
    private final OrderQueryRepository orderQueryRepository; // Query: Repository 직접

    // Command: Service를 통함
    @PostMapping("/orders")
    public ResponseEntity<PlaceOrderResponse> placeOrder(@RequestBody PlaceOrderRequest req) {
        OrderId id = placeOrderUseCase.placeOrder(PlaceOrderCommand.from(req));
        return ResponseEntity.created(...).body(PlaceOrderResponse.of(id));
    }

    // Query: Repository 직접 (단순 조회)
    @GetMapping("/orders/{id}")
    public OrderSummaryResponse findOrder(@PathVariable String id) {
        return orderQueryRepository.findSummaryById(id)
            .orElseThrow(OrderNotFoundException::new);
    }

    // Query: 목록 조회도 Repository 직접
    @GetMapping("/orders")
    public Page<OrderSummaryResponse> findOrders(Pageable pageable) {
        return orderQueryRepository.findAll(pageable);
    }
}

// Query 전용 Repository (읽기 최적화, 도메인 모델 불필요)
@Repository
public interface OrderQueryRepository extends JpaRepository<OrderJpaEntity, Long> {

    @Query("SELECT new com.example.OrderSummaryResponse(o.id, o.status, o.totalPrice) " +
           "FROM OrderJpaEntity o WHERE o.id = :id")
    Optional<OrderSummaryResponse> findSummaryById(@Param("id") String id);
}
```

---

## 📊 패턴 비교 — Strict vs Relaxed 장단점 비교

```
기능 추가 시나리오: "주문 목록에 쿠폰 할인 여부 표시"

=== Strict Layer ===
변경 위치:
  FindOrderService → CouponRepository 추가 의존
  OrderResponse → couponApplied 필드 추가
  OrderController → 변경 없음

특징:
  Service에 로직이 추가됨 → Service 테스트로 검증
  Controller는 변경 없음 (인터페이스 안 바뀌면)

=== Relaxed Layer (Controller → Repository 직접) ===
변경 위치:
  OrderController → CouponRepository 추가 의존
  OrderController → 쿠폰 적용 여부 계산 로직 추가

문제:
  비즈니스 로직(쿠폰 계산)이 Controller에 추가됨
  Controller가 두 Repository에 의존하게 됨
  다른 Controller에서 같은 로직이 필요하면? 중복 발생

==================================================

ArchUnit 위반 감지 속도:
  Strict + ArchUnit:
    CI에서 즉시 "Controller가 Repository를 직접 의존" 에러 발생
    위반이 코드에 들어올 수 없음

  Relaxed + 팀 합의만:
    코드 리뷰에서 놓칠 수 있음
    "이것도 단순 조회 아닌가요?" 기준이 점점 느슨해짐
    결국 규칙이 유명무실해짐
```

---

## ⚖️ 트레이드오프

```
Strict Layer:
  장점: 레이어 의미가 명확하게 유지됨, ArchUnit으로 자동 강제 용이
  단점: 단순 조회에도 UseCase 클래스 필요 → 보일러플레이트

Relaxed Layer:
  장점: 단순 경우에 빠른 개발, CQRS와 자연스럽게 연결
  단점: "단순"의 기준이 흐릿해짐, 장기적으로 규칙이 무너질 위험

현실적 권장:
  1. 팀이 처음 아키텍처를 적용하는 경우 → Strict
     이유: 규칙이 명확해야 일관성 유지 가능

  2. 팀이 CQRS를 적용하는 경우 → Relaxed (Command/Query 명확 분리)
     이유: Query는 Infrastructure 직접 접근이 CQRS의 의도

  3. 어느 방식이든 ArchUnit으로 강제 → 필수
     이유: 코드 리뷰만으로는 일관성 유지 한계
```

---

## 📌 핵심 정리

```
Strict vs Relaxed Layer 핵심:

Strict Layer:
  인접 레이어만 호출 가능
  Controller → Service → Repository (건너뜀 금지)
  장점: 일관성, 단점: 보일러플레이트

Relaxed Layer:
  모든 하위 레이어 호출 가능
  조회는 Controller → Repository 허용
  장점: 개발 속도, 단점: 허용 범위 확장 위험

실무 핵심:
  어느 방식을 선택하든 ArchUnit으로 규칙을 코드화
  "단순 조회" 범위를 명확히 정의 (문서화)
  비즈니스 로직은 항상 Domain Layer에 → 이 원칙이 더 중요

무너지는 신호:
  Controller에 if 문이 생기기 시작
  Controller가 3개 이상의 Repository에 의존
  조회 로직이 여러 Controller에 중복
```

---

## 🤔 생각해볼 문제

**Q1.** "단순 조회는 Controller → Repository 직접"을 허용했을 때, 시간이 지나면서 이 규칙이 무너지는 이유는 무엇인가? 어떻게 방지할 수 있는가?

<details>
<summary>해설 보기</summary>

무너지는 이유는 **"단순"의 기준이 점차 확장**되기 때문입니다.

처음에는 `findById()` 하나만 직접 접근합니다. 그런데 요구사항이 추가되면:
- "조회 시 권한 체크 추가" → Controller에 Repository 하나 더 추가
- "조회 결과에 쿠폰 정보 포함" → Controller에 또 다른 Repository 추가
- "조회 횟수를 통계에 기록" → Controller에 StatisticsRepository 추가

각각은 "아직 단순한 거잖아요"라고 생각되지만, 결과적으로 Controller가 복잡한 조율 로직을 담게 됩니다.

**방지 방법:**
1. ArchUnit으로 Controller가 의존할 수 있는 Repository를 특정 패턴으로 제한
2. "조회에 Repository가 2개 이상 필요하면 QueryService 도입" 규칙 수립
3. 정기적인 아키텍처 리뷰 (월 1회 의존성 그래프 확인)

</details>

---

**Q2.** Relaxed Layer에서 CQRS(Command Query Responsibility Segregation)를 적용하면 Relaxed의 단점이 어느 정도 해소되는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

CQRS는 **Command(쓰기)와 Query(읽기)의 모델과 경로를 명확히 분리**합니다.

```
Command path (쓰기): Controller → UseCase → Domain → Repository (Strict)
Query path (읽기):   Controller → QueryRepository (Relaxed, 의도적)
```

이 분리가 명확하면:
- "언제 Service를 통하고 언제 직접 접근하는가" 기준이 명확 (Write=Service, Read=직접)
- Query가 복잡해지면 QueryService 추가 (Command path로 승격 아님)
- Command path에는 절대 Relaxed 허용 안 됨 (비즈니스 규칙 보호)

ArchUnit 규칙도 명확하게 설정 가능:
```java
// Command 관련 클래스는 반드시 UseCase를 통해야 함
// Query 관련 클래스는 QueryRepository 직접 접근 허용
```

단, CQRS 없이 "조회는 직접"만 허용하면 기준이 흐릿해집니다. CQRS는 Relaxed를 의미있게 만드는 구조적 지원입니다.

</details>

---

**Q3.** ArchUnit 테스트가 CI에서 실패했는데, 팀원이 "이 경우는 예외로 해야 한다"고 주장한다. 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

ArchUnit은 **예외 처리 메커니즘**을 제공합니다. 그러나 예외를 추가하기 전에 먼저 물어야 합니다.

**먼저 물어야 할 것들:**
1. "왜 이 경우에 레이어 경계를 넘어야 하는가?" (근본 원인 파악)
2. "아키텍처 규칙 대신 다른 방식으로 해결할 수 없는가?" (대안 탐색)
3. "이 예외가 생기면 다른 유사한 케이스도 예외 요청이 올 것인가?" (파급 효과)

**ArchUnit의 예외 처리:**
```java
@Test
void controller_should_not_access_repository_directly() {
    noClasses()
        .that().resideInAPackage("..controller..")
        .should().dependOnClassesThat()
        .resideInAPackage("..repository..")
        .ignoreDependency( // 의도적 예외
            OrderController.class,
            OrderQueryRepository.class // CQRS Query path
        )
        .check(importedClasses);
}
```

예외 추가 시 반드시:
- 코드 주석에 예외 이유 명시
- ADR(Architecture Decision Record)에 기록
- 팀 전체 리뷰 후 결정

예외가 2개 이상 생기면 규칙 자체를 재검토해야 할 신호입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 레이어드 아키텍처의 원칙](./01-layered-principles.md)** | **[홈으로 🏠](../README.md)** | **[다음: 레이어드 아키텍처의 함정 ➡️](./03-layered-architecture-traps.md)**

</div>
