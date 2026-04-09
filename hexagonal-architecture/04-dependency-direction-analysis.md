# 의존성 방향 완전 분석 — 왜 모든 화살표가 도메인을 향하는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hexagonal Architecture에서 모든 의존성이 도메인 방향으로 향해야 하는 정확한 이유는?
- JPA, Kafka, HTTP 같은 인프라가 도메인에 의존한다는 것은 코드에서 어떻게 나타나는가?
- DIP가 의존성 방향을 역전시키는 메커니즘을 단계별로 설명하면?
- 레이어드와 Hexagonal의 의존성 다이어그램은 어떻게 다른가?
- 의존성 방향이 잘못됐을 때 코드에서 나타나는 구체적인 신호는?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

"모든 화살표가 도메인을 향한다"는 규칙이 Hexagonal Architecture의 핵심이다. 이 방향이 하나라도 역전되면 — 도메인이 JPA를 알게 되거나 UseCase가 Kafka를 직접 호출하면 — Hexagonal의 핵심 이익(테스트 용이성, 인프라 독립성)이 무너진다.

이 문서는 의존성 방향이 왜 그래야 하는지를 "변경 비용"의 관점에서 분석한다.

---

## 😱 흔한 실수 (Before — 의존성 방향을 혼동할 때)

```
코드는 Hexagonal처럼 보이지만 의존성 방향이 잘못된 경우:

실수 1: Domain이 인프라 어노테이션을 사용
  // domain/Order.java
  @Entity  // JPA 어노테이션 — OK (절충 허용)
  @Table(name = "orders")  // DB 스키마 의존 — 주의
  public class Order {
      @JsonProperty("order_id")  // Jackson 어노테이션 — ❌ 심각
      private OrderId id;
      // HTTP 직렬화 라이브러리가 도메인에 침투
  }

실수 2: Application Service가 Kafka를 직접 의존
  @Service
  public class PlaceOrderService implements PlaceOrderUseCase {
      @Autowired
      KafkaTemplate<String, String> kafkaTemplate; // ❌ 인프라 직접 의존

      public OrderId placeOrder(PlaceOrderCommand command) {
          // ...
          kafkaTemplate.send("order-placed", serialize(order)); // Kafka 직접 호출
      }
  }
  // PlaceOrderService 테스트에 Kafka 브로커 필요

실수 3: Driven Port 인터페이스가 Infrastructure Layer에 위치
  // infrastructure/OrderRepository.java (❌ 잘못된 위치)
  public interface OrderRepository {
      void save(Order order);
  }
  // PlaceOrderService(domain) → OrderRepository(infrastructure)
  // = domain이 infrastructure에 의존 → DIP 미적용
```

---

## ✨ 올바른 접근 (After — 의존성 방향이 모두 도메인을 향할 때)

```
올바른 의존성 방향:

  [HTTP] → [Controller(Driving Adapter)] → [UseCase(Driving Port)] ← [PlaceOrderService]
                                                                              │
                                                          [OrderRepository(Driven Port)] ← [JpaAdapter]
                                                          [PaymentPort(Driven Port)]     ← [KakaoPayAdapter]
                                                          [EventPublisher(Driven Port)]  ← [KafkaAdapter]

모든 화살표(→, ←)가 Application Core(PlaceOrderService, Port 인터페이스)를 향함

확인 방법:
  "JPA가 도메인을 import하는가?" → YES (JpaAdapter implements OrderRepository(domain))
  "도메인이 JPA를 import하는가?" → NO (PlaceOrderService는 JPA 클래스 없음)
  = 의존성 방향 올바름
```

---

## 🔬 내부 원리 — 의존성 방향 분석

### 1. 변경 비용과 의존성 방향의 관계

```
핵심 원리: 변경이 자주 일어나는 것이 변경이 드문 것에 의존해야 한다

변경 빈도 비교:
  도메인(비즈니스 규칙): 상대적으로 안정적 (비즈니스 핵심)
  인프라(JPA, Kafka, 외부 API): 상대적으로 불안정 (기술 교체 가능)

올바른 방향: 불안정한 것 → 안정적인 것
  인프라(JPA, Kafka) → 도메인(Port 인터페이스)
  = 인프라가 바뀌어도 도메인은 안 바뀜

잘못된 방향: 안정적인 것 → 불안정한 것
  도메인(PlaceOrderService) → 인프라(JpaRepository)
  = JPA가 바뀌면 도메인도 바꿔야 함

Uncle Bob의 Stable Dependencies Principle:
  "의존성은 안정적인 방향으로 흘러야 한다"
  도메인 = 가장 안정적인 것 → 모든 의존성이 도메인을 향해야 함
```

### 2. 레이어드 vs Hexagonal 의존성 다이어그램 비교

```
=== 레이어드 아키텍처의 의존성 다이어그램 ===

  OrderController ──→ OrderService ──→ OrderJpaRepository
                           │                   │
                           ↓                   ↓
                    KakaoPayClient          DB Schema
                           │
                           ↓
                     JavaMailSender

화살표(→) 방향: 위에서 아래로, 인프라 방향
  OrderService가 모든 인프라를 직접 앎
  인프라 변경 → Service 영향
  Service 테스트 = 모든 인프라 필요

=== Hexagonal Architecture의 의존성 다이어그램 ===

  [OrderController]     [JpaOrderRepository]   [KakaoPayAdapter]
  (Driving Adapter)     (Driven Adapter)        (Driven Adapter)
       │ calls               │ implements             │ implements
       ↓                     │                        │
  [PlaceOrderUseCase]         ↓                       ↓
  (Driving Port)    ← [PlaceOrderService] → [OrderRepository]
                     implements  (Core)     (Driven Port)  [PaymentPort]
                                            (Driven Port)

화살표 방향을 정리하면:
  모든 화살표의 HEAD(화살촉)가 Application Core를 향함

  Driving Adapter → Core: Controller가 Core를 호출
  Driven Adapter → Core: JpaAdapter가 Core의 인터페이스를 구현
  Core → Driven Port: Service가 인터페이스를 통해 외부 요청

  ※ Core → Driven Port는 Core에서 나가는 화살표처럼 보이지만
     Port 인터페이스 자체가 Core에 위치하므로 Core 안에서의 참조
```

### 3. DIP가 의존성 방향을 역전시키는 단계별 분석

```
Step 0: 자연스러운 방향 (DIP 없음)
  "Service는 Repository를 사용한다"
  → Service → Repository (Service가 Repository를 의존)
  → Service가 JPA 구현체를 직접 앎

  PlaceOrderService ──depends on──→ JpaOrderRepository
  방향: Service → 인프라 (자연스럽지만 DIP 위반)

Step 1: 인터페이스 추출 (추상화 도입)
  "OrderRepository 인터페이스를 만든다"
  PlaceOrderService ──depends on──→ OrderRepository (인터페이스)
  JpaOrderRepository ──implements──→ OrderRepository (인터페이스)

Step 2: 인터페이스 위치 결정 (핵심!)
  잘못된 위치: infrastructure 패키지
    PlaceOrderService(domain) → OrderRepository(infrastructure) ← JpaOrderRepository
    = domain → infrastructure (DIP 미적용)

  올바른 위치: domain 패키지
    JpaOrderRepository(infrastructure) → OrderRepository(domain) ← PlaceOrderService(domain)
    = infrastructure → domain (DIP 적용!)

Step 3: 결과 — 의존성 역전 완성
  Before: PlaceOrderService ──→ JpaOrderRepository
          (Service가 인프라 의존)
  
  After:  JpaOrderRepository ──→ OrderRepository ←── PlaceOrderService
          (인프라가 도메인 의존, Service는 인터페이스만 의존)

  "방향이 역전됨": JpaOrderRepository → 도메인 방향으로
  이것이 Dependency INVERSION (역전)
```

### 4. 각 컴포넌트의 import 문으로 의존성 방향 확인

```java
// 의존성 방향을 import 문으로 확인하는 방법

// domain/Order.java — 올바른 경우
package com.example.domain;

// import에 인프라가 없어야 함
import java.util.List;        // Java 표준 — OK
import java.util.UUID;        // Java 표준 — OK
// ❌ 절대 없어야 하는 import:
// import javax.persistence.*;  (JPA)
// import com.fasterxml.jackson.*;  (Jackson)
// import org.springframework.*;  (Spring)
// import com.kakaopay.*;  (외부 SDK)

// domain/PlaceOrderService.java — 올바른 경우
package com.example.domain.application;

import com.example.domain.Order;             // 도메인 — OK
import com.example.domain.OrderRepository;   // 도메인 Port — OK
import com.example.domain.PaymentPort;       // 도메인 Port — OK
// ❌ 절대 없어야 하는 import:
// import org.springframework.data.jpa.*;  (JPA)
// import org.apache.kafka.*;  (Kafka)
// import com.kakaopay.*;  (외부 SDK)

// infrastructure/JpaOrderRepository.java — 올바른 경우
package com.example.infrastructure.persistence;

import com.example.domain.Order;             // 도메인 — OK (인프라가 도메인 import)
import com.example.domain.OrderRepository;   // 도메인 Port — OK (구현하기 위해)
import org.springframework.data.jpa.*;       // JPA — OK (인프라는 인프라 알아도 됨)
import javax.persistence.*;                  // JPA — OK

// 이 import 패턴이 올바른 의존성 방향의 코드 증거:
// 인프라(JpaOrderRepository)가 도메인(OrderRepository)을 import
// 도메인은 인프라를 import하지 않음
```

### 5. 의존성 방향 위반 탐지 — ArchUnit 자동 검증

```java
@AnalyzeClasses(packages = "com.example")
public class HexagonalArchitectureTest {

    JavaClasses classes = new ClassFileImporter()
        .importPackages("com.example");

    // 핵심 규칙 1: 도메인은 인프라를 모름
    @Test
    void domain_should_not_depend_on_infrastructure() {
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..")
            .check(classes);
    }

    // 핵심 규칙 2: 도메인은 Spring을 모름
    @Test
    void domain_should_not_use_spring_framework() {
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("org.springframework..")
            .check(classes);
    }

    // 핵심 규칙 3: 도메인은 JPA를 모름
    @Test
    void domain_should_not_use_jpa() {
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("javax.persistence..")
            .orShould().dependOnClassesThat()
            .resideInAPackage("jakarta.persistence..")
            .check(classes);
    }

    // 핵심 규칙 4: Driven Adapter는 Driven Port를 구현해야 함
    @Test
    void driven_adapters_should_implement_driven_ports() {
        classes()
            .that().resideInAPackage("..infrastructure..")
            .and().implement(classesFrom("..domain.."))
            .should().resideInAPackage("..infrastructure..")
            .check(classes);
    }

    // 핵심 규칙 5: 애플리케이션 전체 의존성 규칙
    @Test
    void hexagonal_dependency_rules() {
        layeredArchitecture()
            .consideringAllDependencies()
            .layer("Domain").definedBy("..domain..")
            .layer("Application").definedBy("..application..")
            .layer("Infrastructure").definedBy("..infrastructure..")
            .whereLayer("Domain").mayNotAccessAnyLayer()     // 도메인은 아무것도 의존 X
            .whereLayer("Application").mayOnlyAccessLayers("Domain")  // App은 Domain만
            .whereLayer("Infrastructure").mayOnlyAccessLayers("Domain", "Application")
            .check(classes);
    }
}
```

---

## 💻 실전 코드 — 의존성 방향 올바른 전체 구조

```java
// === 올바른 의존성 방향 전체 예시 ===

// [1] domain/ — 아무것도 의존하지 않음
package com.example.domain;

public class Order {                              // 순수 Java
    private final OrderId id;
    private final List<OrderLine> lines;
    public void place() { /* 비즈니스 로직만 */ }
}

public interface OrderRepository {               // Driven Port (domain에 위치)
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentPort {                   // Driven Port (domain에 위치)
    PaymentResult charge(PaymentRequest req);
}

// [2] application/ — domain만 의존
package com.example.application;

import com.example.domain.*;                     // domain만 import

public interface PlaceOrderUseCase {             // Driving Port
    OrderId placeOrder(PlaceOrderCommand cmd);
}

@Service
public class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderRepository repo;          // domain Port
    private final PaymentPort payment;           // domain Port

    @Override
    public OrderId placeOrder(PlaceOrderCommand cmd) {
        Order order = Order.create(cmd.getUserId(), cmd.getLines());
        order.place();
        PaymentResult result = payment.charge(PaymentRequest.of(order));
        order.confirmPayment(result.getTransactionId());
        repo.save(order);
        return order.getId();
    }
    // import에 JPA, Kafka, HTTP 없음 → 의존성 방향 올바름
}

// [3] infrastructure/ — domain과 application을 의존 (구현)
package com.example.infrastructure;

import com.example.domain.*;                     // domain import — OK
import org.springframework.data.jpa.*;           // JPA import — OK (인프라니까)

@Repository
public class JpaOrderRepository implements OrderRepository {  // domain Port 구현
    private final SpringDataOrderJpa jpa;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        jpa.save(mapper.toEntity(order));        // Domain → JPA Entity 변환
    }
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value()).map(mapper::toDomain); // JPA → Domain 변환
    }
}

// Driving Adapter — application Port 사용
@RestController
public class OrderController {
    private final PlaceOrderUseCase useCase;     // application Port

    @PostMapping("/orders")
    public ResponseEntity<?> place(@RequestBody PlaceOrderRequest req) {
        OrderId id = useCase.placeOrder(PlaceOrderCommand.from(req));
        return ResponseEntity.created(...).body(PlaceOrderResponse.of(id));
    }
}
```

---

## 📊 패턴 비교 — 의존성 방향에 따른 변경 영향 비교

```
인프라 변경 시나리오: "MySQL → PostgreSQL 교체"

=== 레이어드 (의존성이 인프라 방향) ===
  PlaceOrderService → OrderJpaRepository (MySQL 전용 쿼리 포함)
  
  변경 필요:
    JpaOrderRepository: MySQL 방언 → PostgreSQL 방언
    PlaceOrderService: JpaRepository 메서드 시그니처 변경 시 수정 필요
    OrderServiceTest: Mock 재설정

=== Hexagonal (의존성이 도메인 방향) ===
  PlaceOrderService → OrderRepository (인터페이스)
  JpaOrderRepository (MySQL) → PostgreSQL 교체

  변경 필요:
    JpaOrderRepository: MySQL → PostgreSQL 구현 변경
  변경 불필요:
    PlaceOrderService: OrderRepository 인터페이스 시그니처 유지 → 변경 없음
    Domain 테스트: InMemory 사용 → 변경 없음

==================================================

의존성 방향 위반 신호 vs 올바른 신호:

위반 신호:
  domain 패키지에서 import org.springframework.data.jpa 발견
  PlaceOrderService 생성자에 KafkaTemplate 파라미터
  Order 클래스에 @JsonProperty 어노테이션

올바른 신호:
  PlaceOrderService의 import: domain 패키지만
  JpaOrderRepository의 import: domain 패키지 + javax.persistence
  ArchUnit 테스트 모두 GREEN
```

---

## ⚖️ 트레이드오프

```
의존성 방향 엄격하게 지킬 때:
  비용: Domain Entity와 JPA Entity 분리 → 매핑 코드 증가
        Spring 어노테이션을 Domain에서 제거 → 설정 코드 추가
  이익: 도메인 순수성 완전 보장
        어떤 인프라도 교체 시 도메인 불변
        순수 Java 단위 테스트 100% 가능

현실적 절충:
  @Entity는 Domain에 허용 (절충): 매핑 비용 절감, 일부 JPA 의존 허용
  @Transactional은 Application Layer: Spring 의존이지만 필수적
  @Valid는 Controller에만: Bean Validation은 Presentation 관심사

절충의 기준:
  "이 어노테이션 때문에 테스트가 어려워지는가?"
    → YES → 제거 또는 분리
  "이 어노테이션이 도메인 로직을 오염시키는가?"
    → YES → 반드시 분리
```

---

## 📌 핵심 정리

```
Hexagonal 의존성 방향 핵심:

규칙: 모든 의존성이 Application Core(도메인)를 향함
  Driving Adapter → Core (Controller가 Core를 호출)
  Driven Adapter → Core (JpaAdapter가 Core 인터페이스 구현)
  Core → Driven Port → (Port가 Core 안에 있으므로 Core 내부 참조)

확인 방법:
  domain 패키지의 import에 인프라 클래스가 없으면 올바름
  ArchUnit으로 자동 검증

DIP 역전 메커니즘:
  자연스러운 방향: Service → JpaRepository
  인터페이스 추출: Service → OrderRepository(interface)
  위치 결정: OrderRepository를 domain에 → JpaRepository가 domain 의존
  결과: JpaRepository → domain (역전 완성!)

위반 신호:
  domain 클래스에 @JsonProperty, @KafkaListener 등 인프라 어노테이션
  PlaceOrderService 생성자에 KafkaTemplate, RestTemplate 직접 주입
  domain 패키지의 Driven Port 인터페이스가 infrastructure에 위치
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional`을 `PlaceOrderService`에 붙이면 도메인이 Spring에 의존하는 것 아닌가? 이것은 의존성 방향 위반인가?

<details>
<summary>해설 보기</summary>

**실용적으로는 허용하지만 이론적으로는 타협입니다.**

순수 Hexagonal 관점에서 `@Transactional`은 Spring 의존이므로 도메인이 Spring을 알게 됩니다. 이를 피하는 방법:

```java
// 완전 순수: Application Layer에 @Transactional, Domain은 모름
// domain/PlaceOrderService.java — @Transactional 없음
public class PlaceOrderService implements PlaceOrderUseCase {
    public OrderId placeOrder(PlaceOrderCommand cmd) { ... }
}

// infrastructure/TransactionalPlaceOrderService.java — Decorator 패턴
@Service
@Transactional
public class TransactionalPlaceOrderService implements PlaceOrderUseCase {
    private final PlaceOrderService delegate;
    @Override
    public OrderId placeOrder(PlaceOrderCommand cmd) {
        return delegate.placeOrder(cmd);
    }
}
```

그러나 이 분리의 비용(파일 수 증가, 복잡도)이 실익보다 크므로 대부분의 팀은 `@Transactional`을 Application Service에 직접 붙이는 것을 허용합니다. ArchUnit 규칙에서 `org.springframework.transaction` 예외 처리로 절충합니다.

</details>

---

**Q2.** Hexagonal Architecture에서 Domain Entity에 `@Entity`를 붙이는 것(절충)과 분리하는 것의 실제 트레이드오프는?

<details>
<summary>해설 보기</summary>

```
@Entity 통합 (절충):
  장점: 매핑 코드(Mapper) 불필요, 파일 수 감소
  단점:
    JPA 제약 침투: protected 기본 생성자 필요 → 불완전 객체 생성 가능
    LAZY 로딩 예외: @Transactional 범위 밖에서 도메인 메서드 호출 시 위험
    DB 스키마 변경이 도메인 객체에 직접 영향

Domain/JPA Entity 분리:
  장점: 도메인이 JPA를 완전히 모름, 기본 생성자 불필요, LAZY 문제 없음
  단점:
    Mapper 클래스 필요 (MapStruct로 자동화 가능)
    파일 수 증가 (Order + OrderJpaEntity + OrderMapper)

판단 기준:
  "JPA 제약(기본 생성자, LAZY)이 도메인 메서드를 깨뜨리는가?" → 분리 필요
  "도메인이 단순해서 JPA 제약이 문제가 없는가?" → 절충 허용
```

</details>

---

**Q3.** ArchUnit이 "도메인이 인프라를 의존하지 않는다" 규칙을 CI에서 자동 검증하면 어떤 장점이 있는가?

<details>
<summary>해설 보기</summary>

**코드 리뷰로는 놓칠 수 있는 의존성 위반을 자동으로 차단합니다.**

구체적 장점:
1. **무의식적 위반 차단**: "급해서 Service에 KafkaTemplate 잠깐 주입"이 PR 단계에서 차단됨
2. **신규 팀원 가이드**: "왜 이게 틀렸나요?" 설명 없이 ArchUnit 에러 메시지가 설명
3. **리팩터링 안전망**: 대규모 패키지 이동 시 의존성 방향이 깨지면 즉시 감지
4. **팀 합의 코드화**: "도메인에 Spring 쓰지 말자"는 구두 합의가 테스트로 강제됨

```java
// CI에서 이 테스트 실패 시:
// "FAILED: domain_should_not_use_spring_framework"
// "com.example.domain.PlaceOrderService imports org.springframework.kafka..."
// 개발자가 어느 클래스에서 어떤 의존성이 생겼는지 즉시 파악
```

</details>

---

<div align="center">

**[⬅️ 이전: 어댑터의 역할](./03-adapters-explained.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring으로 Hexagonal 구현 ➡️](./05-hexagonal-with-spring.md)**

</div>
