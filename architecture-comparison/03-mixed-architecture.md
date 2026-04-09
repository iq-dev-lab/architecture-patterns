# 혼합 아키텍처 — Layered 기본 + 핵심 도메인에만 Hexagonal 적용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 레이어드를 기본으로 하되 핵심 도메인에만 Hexagonal을 적용하는 전략은?
- Strangler Fig 방식으로 기존 코드를 점진적으로 리팩터링하는 구체적 순서는?
- 혼합 아키텍처에서 일관성을 유지하는 규칙은 무엇인가?
- 혼합 구조를 ArchUnit으로 어떻게 강제하는가?
- 혼합 아키텍처가 "최악의 혼합"이 되지 않으려면 무엇이 필요한가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

현실에서 대부분의 프로젝트는 "처음부터 완전한 Hexagonal"이 아니다. 기존 레이어드 코드베이스가 있고, 복잡도가 증가하면서 점진적으로 개선해야 한다.

이 전환이 잘못 되면 레이어드도 Hexagonal도 아닌 혼재 상태가 된다. 혼합 아키텍처를 성공적으로 적용하려면 명확한 규칙과 자동화된 강제 수단이 필요하다.

---

## 😱 흔한 실수 (Before — 규칙 없는 혼합)

```
"일부는 Hexagonal, 일부는 레이어드"의 나쁜 사례:

order/
  OrderController.java
  PlaceOrderService.java  ← Port/Adapter 있음 (Hexagonal)
  
product/
  ProductController.java
  ProductService.java     ← Repository 직접 의존 (레이어드)
  
user/
  UserController.java
  UserService.java        ← 일부는 Port 있고 일부는 직접 의존 (혼재)

문제:
  신규 팀원: "주문은 Port 써야 하고 상품은 Repository 직접 쓰면 되나요?"
  "왜 어떤 건 Hexagonal이고 어떤 건 레이어드인지 모르겠어요"
  명문화된 규칙 없음 → 개인 판단 → 계속 혼재
  ArchUnit으로 규칙 강제 불가 (기준이 없어서)
```

---

## ✨ 올바른 접근 (After — 명확한 규칙의 혼합)

```
올바른 혼합 전략:

규칙 1: 기능별 아키텍처 분류를 문서화
  핵심 도메인 (Hexagonal): 주문, 결제, 재고
  이유: 복잡한 비즈니스 규칙, 외부 시스템 교체 예상
  
  지원 기능 (레이어드+DIP): 공지, 설정, 로그
  이유: 단순 CRUD, 비즈니스 규칙 없음

규칙 2: 각 분류별 ArchUnit 규칙 적용
  order/ 패키지: Hexagonal 규칙 (domain이 infrastructure 모름)
  notice/ 패키지: 레이어드 규칙 (Service가 인터페이스에 의존)

규칙 3: 경계를 넘나드는 의존성 관리
  주문 도메인 ↔ 공지 도메인: 직접 의존 금지
  인터페이스 or 이벤트를 통해서만 통신

규칙 4: 신규 기능은 반드시 패턴 지정
  PR 체크리스트에 "이 기능은 Hexagonal인가, 레이어드인가?" 포함
```

---

## 🔬 내부 원리 — 혼합 아키텍처 전략

### 1. Strangler Fig 패턴 — 점진적 전환 방법

```
Strangler Fig 패턴의 의미:
  무화과나무(Fig)는 다른 나무를 감싸면서 천천히 교체함
  기존 코드를 한 번에 교체하지 않고, 새 기능을 새 방식으로 작성하면서
  점진적으로 기존 코드를 새 방식으로 교체

단계별 적용:

Phase 1: 안전망 구축 (1주)
  기존 코드에 통합 테스트 추가 (현재 동작 기록)
  @SpringBootTest로 핵심 시나리오 5-10개 커버
  "이 테스트들이 통과하면 리팩터링 성공"

Phase 2: Repository 인터페이스 추출 (1-2일/도메인)
  Before: OrderService → OrderJpaRepository (구체 클래스)
  After:  OrderService → OrderRepository (인터페이스)
          JpaOrderRepository implements OrderRepository

  가장 빠른 이익: Service 단위 테스트 가능
  리스크: 매우 낮음 (Spring DI가 자동 연결)

Phase 3: 외부 API 인터페이스 추출 (2-3일/Port)
  Before: OrderService → KakaoPayClient (SDK 직접)
  After:  OrderService → PaymentPort (인터페이스)
          KakaoPayAdapter implements PaymentPort

  이익: 결제 수단 교체 가능, 단위 테스트 가능
  
Phase 4: Domain 로직 이동 (1주/도메인)
  Before: OrderService에 비즈니스 검증 로직
  After:  Order.place()에 비즈니스 로직
          OrderService는 Order.place() 호출만

Phase 5: UseCase 인터페이스 도입 (1-2일)
  OrderController → PlaceOrderUseCase (인터페이스)
  이익: Controller 단위 테스트 가능

Phase 6: Entity와 JPA Entity 분리 (선택, 1주)
  Order (순수 Java) + OrderJpaEntity (@Entity)
  필요한 경우에만 (JPA 제약이 도메인을 오염시킬 때)

각 Phase는 독립적 완료 후 배포 가능
→ "전체를 한 번에 바꾸자"는 실패 확률 높음
→ Phase 단위로 PR → 리뷰 → 머지 → 배포 사이클
```

### 2. 혼합 아키텍처 패키지 구조

```
com.example
├── order/                          ← Hexagonal (핵심 도메인)
│   ├── domain/
│   │   ├── model/Order.java
│   │   └── port/
│   │       ├── in/PlaceOrderUseCase.java
│   │       └── out/OrderSavePort.java
│   ├── application/PlaceOrderService.java
│   └── adapter/
│       ├── in/web/OrderController.java
│       └── out/persistence/JpaOrderRepository.java
│
├── payment/                        ← Hexagonal (핵심 도메인, 외부 API 교체 예상)
│   ├── domain/port/out/PaymentPort.java
│   └── adapter/out/KakaoPayAdapter.java
│
├── notice/                         ← 레이어드+DIP (단순 CRUD)
│   ├── controller/NoticeController.java
│   ├── service/NoticeService.java  ← 인터페이스에 의존 (DIP 적용)
│   └── repository/
│       ├── NoticeRepository.java   ← 인터페이스
│       └── JpaNoticeRepository.java ← 구현체
│
└── config/                         ← 지원 기능 (레이어드 기본 허용)
    └── service/ConfigService.java

ARCHITECTURE.md에 명시:
  order/, payment/ → Hexagonal 규칙 적용
  notice/, config/ → 레이어드+DIP 규칙 적용
  새 기능 추가 시: PR 설명에 "적용 패턴: Hexagonal/레이어드" 명시
```

### 3. 혼합 아키텍처 ArchUnit 규칙

```java
// 기능별 다른 규칙을 ArchUnit으로 강제

@AnalyzeClasses(packages = "com.example")
public class MixedArchitectureTest {

    JavaClasses classes = new ClassFileImporter().importPackages("com.example");

    // Hexagonal 기능 (order, payment)에 대한 규칙
    @Test
    void hexagonal_features_domain_should_not_depend_on_infrastructure() {
        noClasses()
            .that().resideInAnyPackage("..order.domain..", "..payment.domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..adapter..", "..infrastructure..",
                "org.springframework..", "jakarta.persistence..")
            .check(classes);
    }

    @Test
    void hexagonal_features_port_should_be_in_domain() {
        classes()
            .that().haveNameMatching(".*(UseCase|Port|Gateway)")
            .and().areInterfaces()
            .and().resideInAnyPackage("..order..", "..payment..")
            .should().resideInAnyPackage("..domain..")
            .check(classes);
    }

    // 레이어드 기능 (notice, config)에 대한 규칙
    @Test
    void layered_features_service_should_depend_on_repository_interface() {
        classes()
            .that().resideInAnyPackage("..notice.service..", "..config.service..")
            .and().areAnnotatedWith(Service.class)
            .should().onlyDependOnClassesThat()
            .resideInAnyPackage(
                "..notice.repository..", "..config.repository..", // 인터페이스만
                "java..", "org.springframework.."
            )
            .check(classes);
        // 주의: JpaRepository 직접 의존 금지 (인터페이스에만)
    }

    // 도메인 간 직접 의존 금지 (경계 유지)
    @Test
    void features_should_not_directly_depend_on_each_other() {
        // order가 notice의 구체 클래스를 직접 의존하면 안 됨
        noClasses()
            .that().resideInAPackage("..order..")
            .should().dependOnClassesThat()
            .resideInAPackage("..notice..")
            .check(classes);
    }
}
```

### 4. 혼합 아키텍처에서 도메인 간 통신

```java
// Hexagonal 도메인과 레이어드 도메인 간 통신 방법

// 방법 1: 이벤트 기반 통신 (권장 — 결합 최소화)
// order/adapter/in/event/NoticeCreatedEventConsumer.java

@Component
public class OrderNoticeConsumer {
    private final PlaceOrderUseCase useCase;

    @EventListener(NoticeCreatedEvent.class) // Spring 이벤트 수신
    public void handle(NoticeCreatedEvent event) {
        // Notice 도메인의 이벤트를 Order 도메인이 처리
        // Notice의 구체 클래스에 의존하지 않음 (이벤트만)
    }
}

// 방법 2: Facade/Port를 통한 통신
// order/domain/port/out/NoticePort.java (Hexagonal 쪽에 Port 정의)
public interface NoticePort {
    List<NoticeId> findActiveNotices();
}

// notice/adapter/out/NoticePortAdapter.java (레이어드 쪽이 Port 구현)
@Component
public class NoticePortAdapter implements NoticePort {
    private final NoticeRepository noticeRepository; // 레이어드 Repository

    @Override
    public List<NoticeId> findActiveNotices() {
        return noticeRepository.findActive().stream()
            .map(n -> NoticeId.of(n.getId()))
            .toList();
    }
}

// 방법 3: 공유 DTO (가장 단순, 결합 허용 시)
// shared/dto/NoticeDto.java
// order와 notice 둘 다 사용 가능한 공유 데이터 구조
```

### 5. "최악의 혼합"이 되지 않으려면

```
최악의 혼합 패턴 (피해야 할 것):

① Hexagonal 구조인데 의존성은 레이어드
  // domain 패키지에 있지만 JPA 직접 의존
  package com.example.order.domain;
  import org.springframework.data.jpa.repository.JpaRepository; // ❌
  public interface OrderRepository extends JpaRepository<Order, Long> {}
  // 패키지 이름만 domain, 실제로는 인프라 의존

② Port는 있지만 Adapter에서 비즈니스 로직
  @Component
  public class KakaoPayAdapter implements PaymentPort {
      public PaymentResult charge(PaymentRequest req) {
          if (req.getAmount() < 100) // ❌ 비즈니스 규칙이 Adapter에
              throw new MinPaymentException();
      }
  }

③ 팀 합의 없이 자기 파트만 스타일 적용
  개발자 A: order 파트를 Hexagonal로 혼자 바꿈
  개발자 B: payment 파트를 레이어드로 유지
  → 두 스타일이 섞인 상태, 아무도 전체를 이해 못함

최악의 혼합을 방지하는 3가지:
  ① 명확한 규칙 문서화 (ARCHITECTURE.md)
  ② ArchUnit으로 자동 강제 (사람 코드리뷰에 의존 X)
  ③ 팀 전체가 규칙에 동의한 상태에서 시작
```

---

## 💻 실전 코드 — 혼합 아키텍처 전환 예시

```java
// Phase 2 예시: Repository 인터페이스 추출 (1시간 작업)

// Before
@Service
public class OrderService {
    @Autowired
    OrderJpaRepository orderRepository; // 구체 JPA 클래스 직접 의존
}

// Step 1: 인터페이스 추출 (order/domain/port/out/OrderSavePort.java 신규)
public interface OrderSavePort {
    void save(Order order);
}

public interface OrderQueryPort {
    Optional<Order> findById(OrderId id);
}

// Step 2: 기존 JpaRepository가 인터페이스 구현
@Repository
public class JpaOrderRepository implements OrderSavePort, OrderQueryPort {
    private final OrderJpaRepository jpa;

    @Override
    public void save(Order order) { jpa.save(toEntity(order)); }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value()).map(this::toDomain);
    }
}

// Step 3: Service가 인터페이스에만 의존
@Service
public class OrderService {
    private final OrderSavePort orderSavePort;   // 인터페이스로 변경
    private final OrderQueryPort orderQueryPort; // 인터페이스로 변경

    // 이제 InMemory 구현체로 단위 테스트 가능!
}

// Phase 2 단위 테스트 (Phase 2 완료 직후 작성)
class OrderServiceTest {
    InMemoryOrderRepository repository = new InMemoryOrderRepository();
    OrderService service = new OrderService(repository, repository); // InMemory!

    @Test
    void 주문_저장_성공() {
        // @SpringBootTest 없이 0.01초 실행
        service.createOrder(validCommand());
        assertThat(repository.count()).isEqualTo(1);
    }
}
```

---

## 📊 패턴 비교 — 전환 전략별 비교

```
전환 방법 비교:

빅뱅 전환 (한 번에 전체 교체):
  기간: 2-4주
  위험도: 높음 (기능 개발 중단, 실수 시 대규모 롤백)
  결과: 실패 확률 높음 (완료 못하고 혼재 상태)
  권장: 팀 전체가 경험 있고, 2주 이상의 리팩터링 시간 확보 시만

Strangler Fig (점진적 교체):
  기간: 3-6개월 (기능 개발과 병행)
  위험도: 낮음 (Phase 단위 배포 가능)
  결과: 성공 확률 높음 (각 Phase가 독립적으로 가치 있음)
  권장: 대부분의 경우

선택적 적용 (핵심 도메인만):
  기간: 1-2개월 (핵심 도메인에만 집중)
  위험도: 낮음
  결과: 비용 최소화로 최대 이익
  권장: 팀 학습 비용이 우선순위가 낮을 때
```

---

## ⚖️ 트레이드오프

```
혼합 아키텍처의 장단점:

장점:
  핵심 도메인만 Hexagonal 적용 → 비용 최소화
  점진적 전환 → 기능 개발 병행 가능
  과잉 엔지니어링 방지 (단순 기능은 레이어드 유지)

단점:
  팀이 두 가지 패턴을 모두 알아야 함
  신규 팀원 온보딩 복잡 ("이 기능은 어느 패턴인가요?")
  ArchUnit 규칙이 복잡해짐 (기능별 다른 규칙)

성공 조건:
  ARCHITECTURE.md로 각 기능별 패턴 명시
  ArchUnit으로 자동 강제 (필수)
  팀 전체 동의
```

---

## 📌 핵심 정리

```
혼합 아키텍처 핵심:

전략:
  핵심 도메인(주문, 결제) → Hexagonal
  지원 기능(공지, 설정) → 레이어드+DIP
  신규 기능은 도입 전에 패턴 지정

점진적 전환 (Strangler Fig):
  Phase 1: 통합 테스트 안전망
  Phase 2: Repository 인터페이스 추출 (1일)
  Phase 3: 외부 API Port 추출 (2-3일)
  Phase 4: 도메인 로직 Entity로 이동 (1주)
  Phase 5: UseCase 인터페이스 도입 (1일)

최악의 혼합 방지:
  문서화 (ARCHITECTURE.md) + ArchUnit + 팀 합의

혼합 성공 기준:
  신규 팀원이 "이 기능은 어느 패턴?"을 문서에서 찾을 수 있음
  ArchUnit이 패턴 위반을 자동으로 차단
  각 Phase 완료 후 배포 가능
```

---

## 🤔 생각해볼 문제

**Q1.** "레이어드+DIP는 Hexagonal의 80%다"라는 말의 의미는? 어떤 20%가 다른가?

<details>
<summary>해설 보기</summary>

레이어드+DIP(Driven Port만 적용)와 완전한 Hexagonal의 차이:

**레이어드+DIP가 가진 것 (80%)**:
- Repository 인터페이스 도입 (Driven Port)
- 외부 API 인터페이스 추출 (Driven Port)
- InMemory Adapter로 단위 테스트 가능

**Hexagonal이 추가로 가진 것 (나머지 20%)**:
- UseCase 인터페이스 (Driving Port): Controller → UseCase 인터페이스에 의존 → Controller 단위 테스트 단순화
- 패키지 구조 명확화: domain/port/in, domain/port/out, adapter/in, adapter/out
- ArchUnit 규칙이 패키지 구조로 명확하게 적용 가능

실용적으로: 레이어드+DIP로 시작해서 필요할 때 UseCase 인터페이스를 추가하면 완전한 Hexagonal이 됩니다. 두 단계가 자연스럽게 이어집니다.

</details>

---

**Q2.** Strangler Fig 방식에서 Phase 2(Repository 인터페이스 추출) 후 기존 테스트가 깨진다면 어떻게 대응하는가?

<details>
<summary>해설 보기</summary>

**Phase 2는 리팩터링이므로 외부 동작이 변하지 않아야 합니다.** 테스트가 깨진다면:

1. **인터페이스가 잘못 정의된 경우**: JpaRepository의 구체 메서드(`saveAndFlush`, `findByIdWithLock`)를 그대로 사용하던 테스트가 인터페이스에 없는 메서드를 찾아서 실패
   → 인터페이스에 필요한 메서드를 추가

2. **Mock이 구체 클래스를 직접 Mock하던 경우**: `@Mock OrderJpaRepository`가 인터페이스 변경으로 깨짐
   → `@Mock OrderRepository`(인터페이스)로 변경

3. **@DataJpaTest가 인터페이스를 주입 못 하는 경우**: 
   → `@DataJpaTest`에 `@Import(JpaOrderRepository.class)` 추가

Phase 1(통합 테스트 안전망)이 충분하다면, Phase 2 후 통합 테스트가 모두 통과하면 리팩터링 성공입니다.

</details>

---

**Q3.** 혼합 아키텍처에서 Hexagonal 도메인과 레이어드 도메인이 서로 데이터를 공유해야 할 때(예: 주문이 공지사항 ID를 참조) 어떻게 설계하는가?

<details>
<summary>해설 보기</summary>

**공유 ID + 인터페이스 통신**으로 결합을 최소화합니다.

```java
// 방법 1: 원시값(ID)만 공유 (가장 느슨한 결합)
// Order(Hexagonal)가 Notice의 ID만 알고, 구체 클래스는 모름
public class Order {
    private final Long pinnedNoticeId; // Long 타입, Notice 구체 클래스 X
}

// 방법 2: 공유 인터페이스 (shared 패키지)
// shared/domain/NoticeId.java (Value Object)
public record NoticeId(Long value) {}

// order/domain이 NoticeId를 알고 Notice 구체 클래스는 모름
public class Order {
    private final NoticeId pinnedNoticeId; // NoticeId만 앎
}

// 방법 3: Port를 통한 조회
// order/domain/port/out/NoticeQueryPort.java
public interface NoticeQueryPort {
    boolean isNoticeActive(Long noticeId);
}

// notice가 이 Port를 구현 (레이어드 → Hexagonal Port 구현)
@Component
public class NoticeQueryPortAdapter implements NoticeQueryPort {
    private final NoticeRepository noticeRepository; // 레이어드 Repository
    @Override
    public boolean isNoticeActive(Long noticeId) {
        return noticeRepository.findById(noticeId)
            .map(Notice::isActive).orElse(false);
    }
}
```

원칙: **Hexagonal 도메인이 레이어드 도메인의 구체 클래스(Notice.java, NoticeService.java)를 직접 import하지 않아야 합니다.**

</details>

---

<div align="center">

**[⬅️ 이전: 패턴 선택 기준](./02-pattern-selection-guide.md)** | **[홈으로 🏠](../README.md)** | **[다음: 아키텍처 결정 기록(ADR) ➡️](./04-architecture-decision-records.md)**

</div>
