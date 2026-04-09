# Layered vs Hexagonal vs Clean 비교 — 의존성·테스트·복잡도

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 의존성 방향, 테스트 용이성, 코드 복잡도 관점에서 3가지 패턴은 어떻게 다른가?
- 팀 학습 비용과 신규 팀원 온보딩 시간은 각 패턴에서 얼마나 차이나는가?
- 도메인 순수성(프레임워크 독립성) 측면에서 각 패턴의 수준은?
- 각 패턴이 해결하는 문제와 해결하지 못하는 문제는 무엇인가?
- 실무에서 "Hexagonal과 Clean의 차이를 모르겠다"는 말이 맞는 경우는?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

세 패턴을 각각 이해하는 것과, 세 패턴의 차이를 비교해서 "우리 상황에 어떤 것이 맞는가"를 판단하는 것은 다르다.

이 문서는 이론적 설명이 아닌 **비교 관점**에서 각 패턴의 특성을 정리한다. 팀이 아키텍처를 선택할 때 이 비교표를 기준으로 논의할 수 있도록 구성됐다.

---

## 😱 흔한 실수 (Before — 패턴을 비교 없이 선택할 때)

```
잘못된 선택 이유들:
  "Hexagonal이 요즘 유행이니까 → 도입"
  "Clean Architecture가 더 이론적으로 완벽하니까 → 도입"
  "레이어드는 구식이니까 → 무조건 탈피"

결과:
  단순 CRUD 서비스에 Hexagonal 전면 적용
  → 파일 18개, 비즈니스 로직 0줄 → 과잉

  복잡한 금융 도메인에 기본 레이어드
  → 테스트 없음, 1000줄 Service, 배포 공포증

  팀이 Clean Architecture 개념 이해 없이 적용
  → "Presenter가 뭐예요?" → 혼재된 코드베이스

올바른 선택:
  팀 규모, 도메인 복잡도, 외부 시스템 교체 가능성, 테스트 자동화 목표를 기준으로
  각 패턴의 비용-이익을 비교해서 결정
```

---

## ✨ 올바른 접근 (After — 비교 기준으로 결정)

```
비교 기준 5가지:

① 의존성 방향 (얼마나 철저히 역전되는가?)
② 테스트 용이성 (Spring Context 없이 테스트 가능한 비율)
③ 코드 복잡도 (파일 수, 개념 수, 학습 난이도)
④ 도메인 순수성 (프레임워크가 도메인에 얼마나 침투하는가?)
⑤ 변경 비용 (외부 시스템 교체, 비즈니스 로직 변경 시 영향 범위)

이 5가지 기준으로 3 패턴을 비교하면
팀 상황에 맞는 선택이 가능
```

---

## 🔬 내부 원리 — 5가지 기준 상세 비교

### 1. 의존성 방향 비교

```
=== 레이어드 아키텍처 ===
의존성 방향: 위 → 아래 (인프라 방향)

  Controller → Service → JpaRepository → DB
                 │
                 └→ KakaoPayClient (외부 API)

Service가 JPA와 외부 API를 직접 의존
= 의존성이 인프라 방향으로 흐름
= DB 변경 → Service 영향 가능

개선형(DIP 적용):
  Controller → Service → OrderRepository(인터페이스)
                              ↑ 구현
                         JpaOrderRepository

부분적 역전 (Repository만)

=== Hexagonal Architecture ===
의존성 방향: 모두 Application Core를 향함

  Controller → UseCase(인터페이스) ← PlaceOrderService
                                           │
                    OrderRepository(인터페이스) ← JpaOrderRepository
                    PaymentPort(인터페이스)     ← KakaoPayAdapter

모든 인프라 클래스가 도메인 인터페이스를 구현
= 의존성 역전 완전 적용

=== Clean Architecture ===
의존성 방향: 의존성 규칙 (안쪽만)

  Controller → Input Boundary ← Interactor
  Interactor → OrderGateway(인터페이스) ← JpaOrderGateway
  Interactor → Entity (직접 의존, 같은 레이어 또는 안쪽)

Hexagonal과 동일하게 의존성 역전 완전 적용
차이: 레이어 이름과 추가 개념(Presenter) 유무
```

### 2. 테스트 용이성 비교

```
테스트 시나리오: "최소 주문 금액 검증 로직 단위 테스트"

=== 레이어드 아키텍처 ===
코드: OrderService.placeOrder() 안에 검증 로직

테스트 방법:
  방법 A: @SpringBootTest → 전체 Context 로딩 (30초+)
  방법 B: @ExtendWith(MockitoExtension) + 5개 Mock 설정

  @Test
  void 최소금액_미달() {
      given(userService.findById(any())).willReturn(mockUser()); // Mock 필요
      given(inventoryService.check(any())).willReturn(true);     // Mock 필요
      // ... 5개 Mock 설정 후
      assertThatThrownBy(() -> service.placeOrder(command)).isInstanceOf(...)
  }
  → Mock이 구현 세부사항에 결합

=== Hexagonal Architecture ===
코드: Order.place() 안에 검증 로직 (Domain Entity)

테스트 방법:
  @Test
  void 최소금액_미달() {
      Order order = Order.create(userId, lowAmountLines());
      assertThatThrownBy(order::place)
          .isInstanceOf(MinOrderAmountException.class);
  }
  → 순수 Java, Spring 없음, 0.001초

=== Clean Architecture ===
코드: Order Entity.place() 안에 검증 로직 (Entities 레이어)
→ Hexagonal과 동일하게 순수 Java 단위 테스트 가능

테스트 전략 비교:

                    레이어드      레이어드(개선)  Hexagonal   Clean
@SpringBootTest    80%          50%            5%          5%
@DataJpaTest       10%          20%            15%         15%
단위 테스트(순수)    5%           20%            70%         70%
기타               5%           10%            10%         10%

CI 실행 시간 (100개 테스트):
  레이어드:          ~43분
  레이어드(개선):    ~20분
  Hexagonal:       ~5분
  Clean:           ~5분
```

### 3. 코드 복잡도 비교

```
같은 "주문 생성" 기능 파일 수 비교:

레이어드 (기본):
  OrderController.java
  OrderService.java
  OrderRepository.java (JpaRepository 상속)
  Order.java (@Entity)
  PlaceOrderRequest.java
  OrderResponse.java
  총: 6개

레이어드 (DIP 개선):
  + OrderRepository.java (인터페이스)
  + JpaOrderRepository.java (구현체)
  총: 8개 (+2)

Hexagonal:
  + PlaceOrderUseCase.java (Driving Port)
  + PlaceOrderCommand.java
  + OrderSavePort.java, OrderQueryPort.java (Driven Port)
  + PaymentPort.java
  + PlaceOrderService.java (UseCase 구현)
  + OrderController.java (Driving Adapter — HTTP 변환)
  + PlaceOrderRequest.java, PlaceOrderResponse.java (HTTP DTO)
  + OrderWebMapper.java
  + JpaOrderRepository.java (Driven Adapter)
  + OrderJpaEntity.java
  + OrderPersistenceMapper.java
  + KakaoPayAdapter.java
  총: ~18개 (+12)

Clean Architecture:
  위 Hexagonal 파일들 +
  + PlaceOrderInputBoundary.java
  + PlaceOrderOutputBoundary.java
  + PlaceOrderRequestModel.java, PlaceOrderResponseModel.java
  + OrderPresenter.java (선택)
  + PaymentGateway.java, PaymentRequestModel.java, PaymentResponseModel.java
  총: ~24개 (+6)

파일 수 비교:
  레이어드 기본: 6개
  레이어드 개선: 8개
  Hexagonal: 18개 (레이어드의 3배)
  Clean: 24개 (레이어드의 4배)
```

### 4. 도메인 순수성 비교

```
"Order 클래스에 JPA 어노테이션이 있는가?" 로 판단

레이어드 (기본):
  @Entity 있음, @Column 있음, @OneToMany 있음
  기본 생성자 필요 (protected)
  → 도메인이 JPA에 완전히 의존

레이어드 (DIP 개선):
  Repository 인터페이스 도입
  그러나 Order.java에는 여전히 @Entity 가능
  → 도메인이 인터페이스로 인프라를 모르지만 @Entity는 여전히 있을 수 있음

Hexagonal:
  이상적: Order.java에 @Entity 없음, 순수 Java
  현실적: @Entity 절충 허용 (팀 선택)
  → 도메인 순수성 목표, 절충 가능

Clean Architecture:
  이상적: Entities 레이어에 어떤 프레임워크도 없음
  현실적: @Transactional 절충 허용
  → 도메인 순수성 최고 수준 목표

순수성 수준 (높음 → 낮음):
  Clean Architecture (이론) > Hexagonal > 레이어드(DIP) > 레이어드(기본)
```

### 5. 각 패턴이 해결하는 문제와 해결 못하는 문제

```
레이어드 아키텍처:

해결하는 문제:
  - 책임 분리(Controller/Service/Repository)
  - 팀 내 코드 위치 규칙 수립
  - 기본적인 코드 구조 제공

해결 못하는 문제:
  - 도메인이 인프라에 의존하는 구조적 문제
  - Spring Context 없는 빠른 단위 테스트
  - 외부 시스템 교체 비용

Hexagonal Architecture:

해결하는 문제:
  - 의존성 역전으로 인프라 독립성 확보
  - InMemory Adapter로 빠른 단위 테스트 가능
  - 외부 시스템 교체를 Adapter 교체로 국한

해결 못하는 문제:
  - 코드량 증가와 매핑 복잡도
  - 초기 팀 학습 비용
  - 단순 CRUD에서의 과잉 복잡도

Clean Architecture:

해결하는 문제:
  - Hexagonal의 문제 + Presenter를 통한 UI 완전 분리
  - 레이어 명칭과 역할의 명확한 이론적 기반
  - 엄격한 의존성 규칙 강제

해결 못하는 문제:
  - Hexagonal보다 더 높은 복잡도
  - Presenter 패턴의 Thread Safety 문제
  - "이론과 실무 사이의 간극" (대부분 팀이 절충 선택)
```

---

## 💻 실전 코드 — 동일 기능의 3패턴 구현 비교

```java
// === 같은 기능: "주문 금액 검증" 위치 비교 ===

// 레이어드: Service에 있음
@Service
public class OrderService {
    public void placeOrder(PlaceOrderRequest req) {
        if (req.getTotal() < 1000) // ← Service에 비즈니스 규칙
            throw new MinOrderAmountException();
        orderRepository.save(new Order(req));
    }
}

// Hexagonal: Domain Entity에 있음
public class Order {
    public void place() {
        if (this.calculateTotal().isLessThan(Money.of(1000)))
            throw new MinOrderAmountException(); // ← Domain에 비즈니스 규칙
        this.status = OrderStatus.PLACED;
    }
}
// PlaceOrderService: order.place() 호출만 (비즈니스 규칙 없음)

// Clean Architecture: Entity에 있음 (Hexagonal과 동일 위치)
public class Order {
    public void place() {
        if (this.calculateTotal().isLessThan(Money.of(1000)))
            throw new MinOrderAmountException(); // ← Entity에 비즈니스 규칙
    }
}
// Interactor: order.place() 호출만 (비즈니스 규칙 없음)

// 비교 결론:
// Hexagonal과 Clean은 비즈니스 로직 위치가 동일 → 테스트 방식도 동일
// 차이는 주변 구조(Presenter, Gateway vs Port, Adapter 이름)
```

---

## 📊 패턴 비교 전체 요약표

```
5가지 기준 비교표:

기준                     │ 레이어드     │ 레이어드+DIP │ Hexagonal    │ Clean
─────────────────────────┼────────────┼─────────────┼─────────────┼──────────────
의존성 방향              │ 인프라 방향  │ 부분 역전    │ 완전 역전    │ 완전 역전
Spring없는 단위 테스트   │ 불가 (5%)   │ 가능 (20%)  │ 가능 (70%)  │ 가능 (70%)
CI 실행 시간 (100개)     │ ~43분       │ ~20분       │ ~5분        │ ~5분
파일 수 (주문 기능)      │ 6개         │ 8개         │ 18개        │ 24개
도메인 순수성            │ 낮음        │ 중간         │ 높음        │ 가장 높음
외부 시스템 교체 비용     │ O(N)        │ O(1) 일부   │ O(1)        │ O(1)
팀 학습 비용             │ 낮음        │ 낮음         │ 중간         │ 높음
신규 팀원 온보딩         │ 반나절      │ 1-3일       │ 1-2주       │ 2-4주
적합한 도메인 복잡도     │ 단순 CRUD   │ 일반 서비스  │ 복잡한 도메인│ 복잡+대규모
```

---

## ⚖️ 트레이드오프

```
패턴 선택의 핵심 트레이드오프:

단순성 vs 순수성:
  레이어드: 단순 (학습 쉬움, 빠른 시작) / 인프라 의존
  Hexagonal: 중간 / 도메인 순수성 높음
  Clean: 복잡 (개념 많음) / 도메인 순수성 가장 높음

개발 속도 vs 장기 유지보수:
  레이어드: 초기 빠름 / 장기 유지보수 어려움 (Fat Service 등)
  Hexagonal/Clean: 초기 느림 / 장기 유지보수 용이

팀 학습 비용 vs 코드 품질:
  레이어드: 학습 없이 시작 가능 / 기술 부채 누적
  Hexagonal: 2주 학습 / 지속적 품질 유지

실용적 결론:
  아키텍처 패턴 선택보다 중요한 것:
  1. 팀 전체가 선택한 패턴을 이해하고 있는가?
  2. ArchUnit으로 규칙이 강제되는가?
  3. 도메인 복잡도에 맞는 패턴을 선택했는가?
```

---

## 📌 핵심 정리

```
3패턴 비교 핵심 요약:

레이어드: "빠른 시작, 점진적 개선 시작점"
  의존성이 인프라 방향 / 빠른 CI 불가 / 단순 구조

레이어드+DIP: "현실적 개선, 과도기"
  Repository 인터페이스 역전 / 일부 빠른 테스트 / 절충

Hexagonal: "도메인 보호, 테스트 속도"
  모든 의존성 역전 / Port/Adapter 명확한 역할 / 학습 필요

Clean: "엄격한 레이어, 대규모 팀"
  Hexagonal + Presenter / 더 많은 파일 / 높은 학습 비용

공통점 (Hexagonal ≈ Clean):
  의존성 역전: 동일
  테스트 속도: 동일
  도메인 순수성: 거의 동일

차이점:
  Presenter 패턴 유무
  용어 (Port/Adapter vs Input/Output Boundary, Gateway)
  파일 수 (Hexagonal < Clean)
```

---

## 🤔 생각해볼 문제

**Q1.** "Hexagonal과 Clean Architecture의 실제 차이를 모르겠다"는 말이 어떤 경우에 맞는 말인가?

<details>
<summary>해설 보기</summary>

**대부분의 실무 팀에서 맞는 말입니다.** 이유:

실무에서 Clean Architecture를 적용하는 팀은 보통:
- Presenter 패턴을 생략하고 UseCase가 직접 반환
- Gateway와 Port/Adapter를 같은 개념으로 취급
- Input/Output Boundary를 Hexagonal의 Driving/Driven Port와 동일하게 구현

이 절충들을 적용하면 두 패턴은 사실상 동일해집니다.

진짜 차이가 있는 경우:
- Presenter 패턴을 완전히 구현할 때
- Clean Architecture의 레이어 이름(Interactor, Gateway)을 엄격히 사용할 때
- Uncle Bob의 Request/Response Model을 HTTP DTO와 완전히 분리할 때

결론: 두 패턴의 차이보다 "의존성 방향과 도메인 순수성"이라는 공통 원칙이 더 중요합니다.

</details>

---

**Q2.** 레이어드 아키텍처에서 Hexagonal로 이행할 때 "어느 순간 Hexagonal이 됐다"고 볼 수 있는 기준은 무엇인가?

<details>
<summary>해설 보기</summary>

**3가지 조건이 모두 충족됐을 때:**

1. **모든 의존성이 도메인 방향**: Service/Interactor가 JPA, Kafka, 외부 API를 직접 import하지 않음
2. **Domain/Application Core를 Spring 없이 테스트 가능**: InMemory 구현체로 단위 테스트가 70%+ 가능
3. **외부 시스템 교체가 Adapter 파일 교체로 완결**: PaymentPort를 구현하는 새 Adapter만 추가하면 됨

패키지 이름(port/, adapter/)은 부차적입니다. 위 3조건을 코드로 검증하는 ArchUnit 테스트가 통과한다면 Hexagonal Architecture입니다.

</details>

---

**Q3.** 팀에 레이어드 아키텍처 경험자만 있을 때 Hexagonal을 도입하면 발생하는 구체적인 문제와 해결 방법은?

<details>
<summary>해설 보기</summary>

**구체적 문제들:**

1. **포트 위치 혼동**: Repository 인터페이스를 infrastructure/ 패키지에 넣음 → DIP 미적용
2. **Adapter에 비즈니스 로직 추가**: Controller에 if문 → 도메인 규칙 중복
3. **InMemory Adapter 미작성**: 테스트는 여전히 @SpringBootTest → 속도 개선 없음
4. **패키지 이름은 hexagonal이지만 import는 레이어드**: 형식만 Hexagonal, 실질은 레이어드

**해결 방법:**

1. **ArchUnit 먼저**: 이론 학습보다 "domain 패키지에서 JPA import가 있으면 빌드 실패" 규칙 먼저 설정
2. **페어 프로그래밍**: 첫 Hexagonal 기능은 경험자(또는 문서)와 함께 작성
3. **InMemory Adapter 템플릿**: 팀 공용 InMemory 구현 패턴을 먼저 제공
4. **점진적 도입**: 새 기능 하나에 Hexagonal 적용, 팀 리뷰로 개념 내재화

</details>

---

<div align="center">

**[⬅️ 이전 챕터: Clean Architecture 실전](../clean-architecture/06-clean-architecture-in-practice.md)** | **[홈으로 🏠](../README.md)** | **[다음: 패턴 선택 기준 ➡️](./02-pattern-selection-guide.md)**

</div>
