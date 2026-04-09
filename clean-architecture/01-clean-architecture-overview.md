# Uncle Bob의 Clean Architecture 개요 — 4개 동심원과 의존성 규칙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Entities / Use Cases / Interface Adapters / Frameworks & Drivers 4개 레이어의 책임은?
- 의존성 규칙(Dependency Rule)이 "안쪽만 향해야 한다"는 말의 정확한 의미는?
- Hexagonal Architecture와 Clean Architecture의 공통점과 핵심 차이점은?
- Clean Architecture가 해결하려는 구체적인 문제는 무엇인가?
- "프레임워크 독립성"이 현실적으로 어떤 의미를 갖는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Robert C. Martin(Uncle Bob)의 Clean Architecture(2017)는 Hexagonal Architecture, Onion Architecture, BCE Architecture 등 여러 아키텍처 패턴의 아이디어를 통합한 것이다.

핵심은 하나다 — **의존성 규칙(Dependency Rule)**: 소스 코드 의존성은 반드시 안쪽으로만(고수준 정책 방향으로) 향해야 한다. 이 단순한 규칙이 테스트 가능성, 인프라 독립성, 유지보수성을 모두 가능하게 한다.

---

## 😱 흔한 실수 (Before — Clean Architecture를 "4개 폴더"로 이해할 때)

```
흔한 오해:
  "Clean Architecture = entities/, usecases/, adapters/, frameworks/ 폴더 만들기"

그 결과:
  entities/
    Order.java          ← @Entity + JPA 어노테이션 있음
  usecases/
    PlaceOrderUseCase.java ← KafkaTemplate 직접 의존
  adapters/
    OrderController.java
  frameworks/
    OrderJpaRepository.java

폴더는 나눴지만 의존성 방향은 여전히 안쪽→바깥쪽:
  Order.java에 @Entity (Entities → Frameworks 의존)
  PlaceOrderUseCase에 KafkaTemplate (Use Cases → Frameworks 의존)
  = 의존성 규칙 위반!

Clean Architecture의 핵심은 폴더 이름이 아니라:
  "Entities 레이어에 JPA 어노테이션이 있는가?" → NO여야 함
  "Use Cases 레이어에 KafkaTemplate이 있는가?" → NO여야 함
  "안쪽 레이어가 바깥 레이어를 import하는가?" → NO여야 함
```

---

## ✨ 올바른 접근 (After — 의존성 규칙의 진짜 의미)

```
4개 동심원과 의존성 방향:

         ┌──────────────────────────────────────┐
         │ Frameworks & Drivers (가장 바깥)       │
         │  JPA, Kafka, Spring MVC, HTTP        │
         │  ┌──────────────────────────────┐    │
         │  │ Interface Adapters            │    │
         │  │  Controller, Presenter,       │    │
         │  │  Gateway(Repository 구현)      │    │
         │  │  ┌──────────────────────┐    │    │
         │  │  │ Use Cases             │    │    │
         │  │  │  Application 비즈니스 │    │    │
         │  │  │  규칙, UseCase 구현   │    │    │
         │  │  │  ┌──────────────┐    │    │    │
         │  │  │  │  Entities    │    │    │    │
         │  │  │  │  기업 수준    │    │    │    │
         │  │  │  │  비즈니스    │    │    │    │
         │  │  │  │  규칙       │    │    │    │
         │  │  │  └──────────────┘    │    │    │
         │  │  └──────────────────────┘    │    │
         │  └──────────────────────────────┘    │
         └──────────────────────────────────────┘

의존성 규칙:
  모든 의존성은 → 안쪽으로만 향함
  바깥 레이어는 안쪽 레이어를 알 수 있음
  안쪽 레이어는 바깥 레이어를 모름

결과:
  Entities: 아무것도 모름 (가장 안정적)
  Use Cases: Entities만 앎
  Interface Adapters: Use Cases, Entities 앎
  Frameworks: 모든 것을 알 수 있음 (가장 불안정)
```

---

## 🔬 내부 원리 — Clean Architecture 구조 분석

### 1. 4개 레이어의 책임

```
레이어 1: Entities (가장 안쪽)
  책임: 기업 수준의 비즈니스 규칙
  의미: "여러 애플리케이션에서 공유될 수 있는 핵심 비즈니스 규칙"
  포함:
    - Domain Entity (Order, User, Product)
    - Value Object (Money, Address)
    - 기업 핵심 비즈니스 규칙 (도메인 로직)
  알면 안 되는 것: 모든 외부 (Spring, JPA, HTTP, 애플리케이션 특정 로직)
  변경 이유: 기업의 핵심 비즈니스 정책이 바뀔 때만

레이어 2: Use Cases (두 번째)
  책임: 애플리케이션 수준의 비즈니스 규칙
  의미: "이 애플리케이션 고유의 흐름과 규칙"
  포함:
    - UseCase Interactor (PlaceOrderInteractor)
    - Input Port / Output Port (Input/Output Boundary)
    - Request Model / Response Model (외부 DTO와 분리된 내부 모델)
  알면 안 되는 것: 프레임워크, DB, UI
  변경 이유: 이 애플리케이션의 비즈니스 규칙이 바뀔 때

레이어 3: Interface Adapters (세 번째)
  책임: 내부↔외부 형식 변환 (번역기)
  포함:
    - Controller (HTTP Request → Use Case Input)
    - Presenter (Use Case Output → View Model)
    - Gateway/Repository 구현체 (Domain Port → DB 접근)
  알 수 있는 것: Use Cases, Entities (안쪽 레이어)
  알면 안 되는 것: Frameworks의 세부 구현

레이어 4: Frameworks & Drivers (가장 바깥)
  책임: 프레임워크와 외부 시스템 구체 구현
  포함:
    - JPA 설정, Kafka 설정
    - Spring Boot 자동 설정
    - HTTP 서버 설정
    - DB 드라이버
    - main 함수 (의존성 조립 지점)
  특징: "세부 사항(Details)"에 해당
         비즈니스 로직 없음, 설정과 연결만 담당
```

### 2. 의존성 규칙 (Dependency Rule)

```
Uncle Bob의 핵심 규칙:
  "소스 코드 의존성은 오직 안쪽으로만 향해야 한다
   (안쪽) Entities ← Use Cases ← Interface Adapters ← Frameworks"

이것이 의미하는 것:
  Entities의 import문에: 다른 레이어의 클래스 없음
  Use Cases의 import문에: Entities 클래스만
  Interface Adapters의 import문에: Use Cases + Entities 클래스만
  Frameworks의 import문에: 모든 안쪽 레이어 가능

왜 안쪽만 향해야 하는가?
  안쪽 레이어 = 더 중요한 것 (비즈니스 규칙)
  바깥 레이어 = 덜 중요한 것 (기술 세부사항)
  
  더 중요한 것이 덜 중요한 것에 의존하면:
  → 기술이 바뀔 때 비즈니스 로직도 바뀌어야 함
  → 프레임워크가 업그레이드되면 도메인도 영향받음
  
  덜 중요한 것이 더 중요한 것에 의존하면:
  → 기술이 바뀌어도 비즈니스 로직은 그대로
  → 프레임워크가 교체되어도 도메인 코드는 변경 없음

경계를 넘나드는 데이터:
  경계를 넘을 때는 항상 가장 안쪽 레이어에 편리한 형태로
  Controller → UseCase: Request Model (UseCase가 편한 형식)
  UseCase → Presenter: Response Model (UseCase가 편한 형식)
  = 외부 세계의 형식(HTTP JSON)이 내부를 오염시키지 않음
```

### 3. Hexagonal과 Clean Architecture의 비교

```
공통점:
  ① 의존성 방향: 모두 도메인/비즈니스 로직 방향으로
  ② 도메인 순수성: 비즈니스 로직이 프레임워크를 모름
  ③ 인터페이스 기반: 경계를 인터페이스로 정의
  ④ 테스트 용이성: 인프라 없이 비즈니스 로직 단위 테스트

차이점:
  구조 표현:
    Hexagonal: 육각형 (여러 Port가 대등하게 존재)
    Clean:     동심원 (레이어의 계층이 명확)

  Presenter 개념:
    Hexagonal: 없음 (Output은 DTO로 직접 반환)
    Clean:     있음 (UseCase → Output Port → Presenter → ViewModel)
    → Clean의 Output Boundary + Presenter가 더 엄격한 분리

  Request/Response Model:
    Hexagonal: Command/Result 객체 (팀마다 다름)
    Clean:     Request Model / Response Model 명시적 정의
    → Clean이 명칭과 역할이 더 명확

  복잡도:
    Hexagonal: 상대적으로 단순 (Port/Adapter 개념)
    Clean:     상대적으로 복잡 (Presenter, Gateway, Interactor 등 추가 개념)

  Uncle Bob이 추가한 개념:
    Interactor: UseCase의 구현체 이름 (Hexagonal의 Application Service)
    Presenter: Output Boundary를 통해 ViewModel 생성 (Hexagonal에 없음)
    Gateway:   Repository의 Interface Adapter 이름 (Hexagonal의 Driven Adapter)
```

### 4. Clean Architecture가 해결하는 문제

```
Uncle Bob이 제시한 5가지 독립성:

① 프레임워크 독립적 (Independent of Frameworks)
  Spring → Quarkus 교체 시 도메인 코드 변경 없음
  현실: 완전 교체는 드물지만, 프레임워크 업그레이드가 비즈니스 로직에 영향 없음

② 테스트 가능 (Testable)
  UI, DB, 외부 서버 없이 비즈니스 규칙 테스트 가능
  현실: 가장 즉각적이고 측정 가능한 이익

③ UI 독립적 (Independent of the UI)
  UI가 REST API → GraphQL → CLI로 바뀌어도 UseCase 코드 변경 없음
  현실: API 스펙 변경이 비즈니스 로직에 영향 없음

④ DB 독립적 (Independent of the Database)
  MySQL → PostgreSQL → MongoDB 교체 시 비즈니스 로직 변경 없음
  현실: DB 교체는 드물지만 DB 버전 업그레이드, 쿼리 최적화 시 영향 없음

⑤ 외부 시스템 독립적 (Independent of any external agency)
  KakaoPay → TossPay, AWS SES → 네이버 클라우드 교체 시 비즈니스 로직 무관
  현실: 외부 API 교체가 가장 자주 발생하는 케이스 → 가장 현실적 이익
```

### 5. 경계를 넘는 흐름 — Input부터 Output까지

```
HTTP Request → Entities 처리 → HTTP Response 흐름:

1. HTTP Request 도착
   (Frameworks & Drivers: HTTP 서버가 수신)

2. Controller가 Request 파싱
   (Interface Adapters: HTTP JSON → Request Model 변환)

3. UseCase 호출
   (Use Cases: Input Port 통해 UseCase Interactor 실행)

4. Entities 조작
   (Use Cases → Entities: 도메인 로직 실행)

5. DB 접근 (필요 시)
   (Use Cases → Output Port(인터페이스) → Gateway(구현체) → DB)

6. UseCase 결과 전달
   (Use Cases: Response Model → Output Port → Presenter 호출)

7. Presenter가 ViewModel 생성
   (Interface Adapters: Response Model → ViewModel)

8. View에 ViewModel 전달
   (Interface Adapters → Frameworks: HTTP Response 생성)

핵심: 각 경계를 넘을 때 항상 내부에 편한 형태로 변환
      HTTP 형식이 Use Cases까지 침투하지 않음
```

---

## 💻 실전 코드 — Clean Architecture 핵심 구조

```java
// === Entities (가장 안쪽) ===
// 어떤 프레임워크도 모름
package com.example.entity;

public class Order {                           // 순수 Java
    private final OrderId id;
    private final List<OrderLine> lines;
    private OrderStatus status;

    public void place() { /* 기업 수준 비즈니스 규칙 */ }
    public Money calculateTotal() { /* ... */ }
}

// === Use Cases ===
// Input Boundary (Driving Port)
package com.example.usecase;

public interface PlaceOrderInputBoundary {
    void execute(PlaceOrderRequestModel requestModel,
                 PlaceOrderOutputBoundary outputBoundary);
}

// Request Model (HTTP DTO와 분리)
public record PlaceOrderRequestModel(
    String userId,
    List<OrderLineRequestModel> lines
) {}

// Output Boundary (Output Port)
public interface PlaceOrderOutputBoundary {
    void present(PlaceOrderResponseModel responseModel);
}

// Response Model (HTTP DTO와 분리)
public record PlaceOrderResponseModel(
    String orderId,
    String totalAmount
) {}

// UseCase Interactor
public class PlaceOrderInteractor implements PlaceOrderInputBoundary {

    private final OrderGateway orderGateway;      // Output Port (Gateway)
    private final PaymentGateway paymentGateway;  // Output Port (Gateway)

    @Override
    public void execute(PlaceOrderRequestModel request,
                        PlaceOrderOutputBoundary outputBoundary) {
        // Entities 조작
        Order order = Order.create(
            UserId.of(request.userId()),
            mapLines(request.lines())
        );
        order.place();

        // Gateway 통해 저장
        orderGateway.save(order);

        // Output Boundary 통해 결과 전달 (직접 반환하지 않음)
        PlaceOrderResponseModel response = new PlaceOrderResponseModel(
            order.getId().value(),
            order.calculateTotal().toString()
        );
        outputBoundary.present(response);
    }
}

// === Interface Adapters ===

// Controller (Driving Adapter)
@RestController
public class OrderController {
    private final PlaceOrderInputBoundary useCase;
    private final OrderPresenter presenter; // Presenter 주입

    @PostMapping("/api/orders")
    public ResponseEntity<OrderViewModel> placeOrder(
        @RequestBody PlaceOrderHttpRequest httpRequest
    ) {
        // HTTP Request → Request Model 변환
        PlaceOrderRequestModel requestModel = new PlaceOrderRequestModel(
            httpRequest.userId(),
            httpRequest.lines().stream()
                .map(l -> new OrderLineRequestModel(l.itemId(), l.quantity()))
                .toList()
        );

        // UseCase 실행 (결과는 Presenter가 받음)
        useCase.execute(requestModel, presenter);

        // Presenter가 만든 ViewModel을 HTTP Response로
        return ResponseEntity.created(...).body(presenter.getViewModel());
    }
}

// Presenter (Output Boundary 구현)
@Component
public class OrderPresenter implements PlaceOrderOutputBoundary {
    private OrderViewModel viewModel;

    @Override
    public void present(PlaceOrderResponseModel responseModel) {
        // Response Model → ViewModel (HTTP 표현 형식으로 변환)
        this.viewModel = new OrderViewModel(
            responseModel.orderId(),
            "/api/orders/" + responseModel.orderId(),
            responseModel.totalAmount()
        );
    }

    public OrderViewModel getViewModel() { return viewModel; }
}

// Gateway (Output Port 구현체)
@Repository
public class JpaOrderGateway implements OrderGateway {
    private final SpringDataOrderJpa jpa;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        jpa.save(mapper.toJpaEntity(order));
    }
}
```

---

## 📊 패턴 비교 — Clean Architecture vs Hexagonal Architecture

```
개념 매핑:

Clean Architecture          │  Hexagonal Architecture
───────────────────────────┼──────────────────────────────
Entities                    │  Domain Model
Use Cases                   │  Application Core
Input Boundary              │  Driving Port (UseCase 인터페이스)
Output Boundary             │  Driven Port (Repository 인터페이스)
UseCase Interactor          │  Application Service
Controller                  │  Driving Adapter
Presenter                   │  (없음 — Hexagonal에서는 직접 반환)
Gateway                     │  Driven Adapter
Frameworks & Drivers        │  Infrastructure

핵심 차이:
  Presenter 개념: Clean만 있음
    UseCase가 결과를 "직접 반환"하지 않고 Output Boundary를 통해 Presenter에게 전달
    → UseCase가 HTTP Response 형식을 전혀 모름 (더 엄격한 분리)
    → 대신 복잡도 증가 (Presenter 클래스 필요, 상태 공유 문제)

  실용성 비교:
    대부분의 팀: Output Boundary만 쓰고 직접 반환 (Hexagonal 스타일)
    Presenter 패턴: View가 복잡한 경우 (여러 UI, MVC 패턴 강조 시)
```

---

## ⚖️ 트레이드오프

```
Clean Architecture의 비용:
  Presenter 패턴: UseCase가 직접 반환하지 않고 Presenter 콜백 방식
                → 코드 흐름 파악이 어려워짐
                → 상태 공유 (Presenter가 ViewModel을 내부에 저장) 문제
  Request/Response Model: HTTP DTO와 완전 분리된 모델 추가
  Gateway 개념: Repository 구현체에 별도 이름과 역할 부여
  학습 비용: Hexagonal보다 높음 (더 많은 개념)

Clean Architecture의 이익:
  가장 엄격한 레이어 분리 (Presenter가 UI 변환 완전 격리)
  의존성 규칙이 명확하고 검증 가능 (ArchUnit으로 강제)
  대규모 팀에서 레이어별 책임 명확
  Uncle Bob의 체계적 이론적 기반

실용적 절충 (대부분의 팀):
  Presenter 패턴은 생략 (직접 반환)
  나머지 구조(의존성 규칙, Request/Response Model 분리)는 유지
  = Hexagonal + Clean 혼합 형태
```

---

## 📌 핵심 정리

```
Clean Architecture 핵심:

4개 동심원:
  Entities: 기업 핵심 비즈니스 규칙 (가장 안정적)
  Use Cases: 애플리케이션별 규칙 (흐름 조율)
  Interface Adapters: 형식 변환 (Controller, Presenter, Gateway)
  Frameworks & Drivers: 기술 세부사항 (JPA, Kafka, HTTP)

의존성 규칙:
  모든 소스 코드 의존성은 안쪽 방향만
  안쪽 레이어는 바깥 레이어를 import하면 안 됨
  = 기술이 비즈니스를 침범하지 않음

Hexagonal과의 관계:
  같은 목표: 도메인 순수성, 인프라 독립성
  Clean이 추가한 것: Presenter 패턴, 명확한 레이어 이름
  실용 현장: 두 패턴을 혼합해서 사용하는 경우 많음
```

---

## 🤔 생각해볼 문제

**Q1.** Clean Architecture의 Use Cases 레이어가 "Application 수준 비즈니스 규칙"을 담고, Entities가 "기업 수준 비즈니스 규칙"을 담는다는 차이는 구체적으로 무엇인가?

<details>
<summary>해설 보기</summary>

Uncle Bob의 구분:

**기업 수준 (Entities)**: "이 비즈니스가 어떤 소프트웨어 시스템을 사용하든 관계없이 적용되는 규칙"
- 주문 금액 = 각 항목의 (단가 × 수량) 합계
- 취소는 배송 시작 전에만 가능
- 이 규칙들은 ERP를 써도, 자체 개발 시스템을 써도 동일하게 적용

**애플리케이션 수준 (Use Cases)**: "이 특정 애플리케이션에서 사용자가 수행하는 흐름"
- "웹 쇼핑몰에서 주문 생성 시: 재고 확인 → 결제 → 저장 → 이메일 발송"
- 이 흐름은 다른 시스템(콜센터 주문 시스템)에서는 다를 수 있음

```java
// Entities: 어떤 시스템에서도 동일한 규칙
public class Order {
    public Money calculateTotal() { /* 항상 단가×수량 합계 */ }
    public void cancel() { /* 항상 배송 전만 가능 */ }
}

// Use Cases: 이 애플리케이션의 특정 흐름
public class PlaceOrderInteractor {
    public void execute(...) {
        // 웹 쇼핑몰 흐름: 재고확인 → 결제 → 이메일
        // 콜센터 시스템이라면 다른 흐름일 수 있음
    }
}
```

</details>

---

**Q2.** Presenter 패턴에서 UseCase가 Output Boundary를 통해 Presenter를 호출하는 방식의 문제점은 무엇인가? 실무에서 왜 대부분 생략하는가?

<details>
<summary>해설 보기</summary>

**Presenter 패턴의 실용적 문제:**

1. **상태 공유 문제**: Presenter가 ViewModel을 내부 상태로 저장 → Thread Safety 이슈
```java
@Component // Spring 기본은 Singleton
public class OrderPresenter implements PlaceOrderOutputBoundary {
    private OrderViewModel viewModel; // 싱글톤이면 동시 요청 시 상태 충돌!
}
```

2. **코드 흐름 파악 어려움**: Controller → UseCase → Presenter (콜백) → Controller가 getViewModel() 흐름이 직관적이지 않음

3. **직접 반환으로 동일 효과**: UseCase가 Response Model을 반환하고 Controller가 ViewModel로 변환하면 같은 분리 효과를 더 단순하게 달성

```java
// Presenter 없이 동일 효과
@RestController
public class OrderController {
    public ResponseEntity<OrderViewModel> placeOrder(...) {
        PlaceOrderResponseModel result = useCase.execute(requestModel);
        return ResponseEntity.ok(OrderViewModelMapper.from(result)); // Controller가 변환
    }
}
```

대부분의 실무 팀은 Presenter를 생략하고 Controller에서 직접 변환합니다. 이것이 Hexagonal Architecture의 방식이며, Clean Architecture의 Presenter는 UI가 복잡한 전통적 MVC 앱에서 더 적합합니다.

</details>

---

**Q3.** "프레임워크 독립적"이어야 한다는 Uncle Bob의 주장이 실무에서 얼마나 현실적인가? Spring을 Quarkus로 교체하는 일이 실제로 일어나는가?

<details>
<summary>해설 보기</summary>

**현실적으로 프레임워크 완전 교체는 드뭅니다.** Uncle Bob의 주장을 문자 그대로 받아들이면 과잉 엔지니어링이 될 수 있습니다.

**그러나 실제 이익은 다른 곳에서 옵니다:**

1. **프레임워크 버전 업그레이드**: Spring Boot 2 → 3 마이그레이션 시, 도메인 코드가 Spring에 의존하지 않으면 영향 최소화
2. **테스트에서 프레임워크 제거**: `@SpringBootTest` 없이 도메인 테스트 → 12배 빠른 테스트 스위트
3. **부분 교체**: HTTP 서버를 Spring MVC → Spring WebFlux로 변경 시, 도메인 코드 변경 없음
4. **외부 API 교체**: KakaoPay → TossPay (이게 실제로 가장 자주 일어남)

결론: "프레임워크 독립적"의 실용적 의미는 "프레임워크를 완전히 교체할 수 있다"가 아니라 "프레임워크 세부사항이 비즈니스 로직에 침투하지 않는다"입니다. 이 제한적 독립성이 가장 현실적이고 즉각적인 이익입니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: DDD와 Hexagonal 통합](../hexagonal-architecture/08-ddd-hexagonal-integration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Entities 레이어 ➡️](./02-entities-layer.md)**

</div>
