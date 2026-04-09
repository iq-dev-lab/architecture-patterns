# 패키지 구조의 원칙 — 레이어 기반 vs 기능 기반 vs 컴포넌트 기반

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 레이어 기반(presentation/service/repository) 구조의 근본적인 문제는?
- 기능 기반(order/payment/user) 구조가 응집도 측면에서 왜 더 좋은가?
- 컴포넌트 기반 구조는 기능 기반과 어떻게 다른가?
- 기능 추가 시 변경되는 파일 범위로 구조를 평가하는 방법은?
- 팀 규모와 도메인 복잡도에 따라 어떤 구조를 선택해야 하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

패키지 구조는 팀의 일상적인 작업 효율을 직접 결정한다. "주문 기능을 추가하려면 어느 파일을 열어야 하는가?"라는 질문에 대한 답이 패키지 구조에 달려 있다.

레이어 기반 구조는 Spring Boot의 기본 관행이어서 가장 많이 쓰이지만, 도메인 복잡도가 올라갈수록 "기능과 관련된 파일들이 여러 패키지에 흩어진다"는 문제가 발생한다. 이 문제를 해결하는 것이 기능 기반, 컴포넌트 기반 구조다.

---

## 😱 흔한 실수 (Before — 레이어 기반 구조의 함정)

```
전형적인 Spring Boot 레이어 기반 구조:

com.example
├── controller/
│   ├── OrderController.java
│   ├── PaymentController.java
│   └── UserController.java
├── service/
│   ├── OrderService.java
│   ├── PaymentService.java
│   └── UserService.java
├── repository/
│   ├── OrderRepository.java
│   ├── PaymentRepository.java
│   └── UserRepository.java
└── entity/
    ├── Order.java
    ├── Payment.java
    └── User.java

문제 1: 기능 추가 시 여러 패키지를 동시에 열어야 함
  "주문에 쿠폰 적용 기능 추가"
  → controller/ 파일 수정
  → service/ 파일 수정
  → repository/ 파일 수정 (또는 신규)
  → entity/ 파일 수정
  → 4개 패키지에 걸쳐 작업

문제 2: 패키지가 도메인 경계를 표현 못함
  "Order 기능이 어디서 시작해서 어디서 끝나는가?"
  → 코드를 보기 전에 알 수 없음
  → 신규 팀원이 Order 기능 전체를 이해하려면 4개 패키지를 탐색

문제 3: 모듈화 불가
  "Order 기능을 별도 서비스로 분리하려면?"
  → 어느 파일이 Order 관련인지 파악 후 선별 필요
  → controller/에서 OrderController만, service/에서 OrderService만...

결론:
  레이어 기반 = 기술(레이어)로 분류
  기능(Order, Payment) 경계가 코드 구조에 없음
```

---

## ✨ 올바른 접근 (After — 응집도 높은 구조 선택)

```
구조 선택 기준:
  변경할 때 함께 바뀌는 것들은 같은 패키지에
  = 높은 응집도 (High Cohesion)
  변경할 때 서로 영향 없는 것들은 다른 패키지에
  = 낮은 결합도 (Low Coupling)

"주문 기능 추가 시 변경 파일이 얼마나 한 곳에 모여 있는가?"
  레이어 기반: 4개 패키지에 분산
  기능 기반:  1개 패키지 (order/) 내에서 완결
  → 기능 기반이 응집도 높음

선택 가이드:
  소규모(3명 이하) + 단순 도메인: 레이어 기반 (단순)
  중규모(3-10명) + 중간 복잡도: 기능 기반 권장
  대규모(10명+) + 복잡한 도메인: 기능 기반 + ArchUnit 경계 강제
```

---

## 🔬 내부 원리 — 3가지 구조 상세 비교

### 1. 레이어 기반 구조 (Package by Layer)

```
구조:
  com.example
  ├── controller/
  ├── service/
  ├── repository/
  └── entity/

특성:
  기술적 관심사(레이어)로 분류
  같은 기능의 파일들이 여러 패키지에 분산
  
장점:
  Spring Boot 기본 관행 → 신규 팀원 즉시 이해
  레이어별 어노테이션 규칙이 명확
  작은 프로젝트(파일 수 30개 이하)에서 충분

단점:
  도메인 경계가 코드 구조에 없음
  기능 추가 시 여러 패키지 동시 수정
  도메인 간 의존성 파악 어려움
  모듈화/MSA 분리 어려움

적합한 상황:
  팀 2-3명, 프로젝트 초기
  비즈니스 도메인이 단순한 경우
  빠른 MVP 개발
```

### 2. 기능 기반 구조 (Package by Feature)

```
구조:
  com.example
  ├── order/
  │   ├── OrderController.java
  │   ├── OrderService.java
  │   ├── OrderRepository.java
  │   └── Order.java
  ├── payment/
  │   ├── PaymentController.java
  │   ├── PaymentService.java
  │   ├── PaymentRepository.java
  │   └── Payment.java
  └── user/
      ├── UserController.java
      └── ...

특성:
  도메인 기능으로 분류
  같은 기능의 파일이 하나의 패키지에 집중

장점:
  도메인 경계가 패키지 구조로 시각화됨
  기능 추가 시 한 패키지 내에서 완결
  MSA 분리 시 패키지 = 서비스로 이동
  신규 팀원이 기능별로 코드 탐색 가능

단점:
  패키지 내 레이어 구분이 없어 섞일 수 있음
  팀 합의 없으면 패키지 내 구조가 제각각
  도메인 간 의존성을 명시적으로 관리해야 함

적합한 상황:
  중규모 이상 팀
  도메인이 명확히 구분된 경우
  MSA 전환 계획이 있는 경우
```

### 3. 컴포넌트 기반 구조 (Package by Component)

```
"컴포넌트 = 기능 + 아키텍처 레이어가 결합된 패키지"
Simon Brown(C4 모델 저자)이 제안

구조:
  com.example
  ├── ordercomponent/
  │   ├── api/           ← 컴포넌트 외부 공개 API
  │   │   └── OrderComponent.java (인터페이스)
  │   └── internal/      ← 컴포넌트 내부 구현 (외부 접근 금지)
  │       ├── OrderController.java
  │       ├── OrderService.java
  │       ├── OrderRepository.java
  │       └── Order.java
  └── paymentcomponent/
      ├── api/
      └── internal/

특성:
  기능 기반 + 내부/외부 경계를 명시적으로 구분
  다른 컴포넌트는 api/만 접근, internal/은 접근 금지
  = 모듈 경계를 패키지로 강제

차이점 (기능 기반 vs 컴포넌트 기반):
  기능 기반: order/ 내부가 모두 공개 (경계 없음)
  컴포넌트 기반: order/api/만 공개, order/internal/은 private

장점:
  컴포넌트 경계가 명확 (ArchUnit으로 강제 가능)
  대규모 팀에서 실수로 내부에 접근하는 것 차단

단점:
  학습 비용 높음 (api/internal 구분 익숙지 않음)
  작은 프로젝트에서 과잉

적합한 상황:
  대규모 팀 (10명+)
  컴포넌트 경계가 자주 침범되는 문제가 있을 때
```

### 4. 기능 추가 시 변경 파일 범위 비교

```
시나리오: "주문에 쿠폰 적용 기능 추가"

=== 레이어 기반 ===
변경/신규 파일:
  controller/OrderController.java (수정)
  service/OrderService.java (수정)
  service/CouponService.java (신규 또는 수정)
  repository/CouponRepository.java (신규)
  entity/Coupon.java (신규)
  entity/Order.java (수정 - couponId 필드 추가)

총 변경: 6개 파일, 4개 패키지에 분산

=== 기능 기반 ===
변경/신규 파일:
  order/OrderController.java (수정)
  order/PlaceOrderService.java (수정)
  coupon/ (신규 패키지)
    coupon/Coupon.java
    coupon/CouponRepository.java
    coupon/CouponService.java

총 변경: 5개 파일, order/ + coupon/ 2개 패키지

차이:
  레이어 기반: 변경이 4개의 다른 역할 패키지에 분산
  기능 기반:  새 기능(coupon)은 자체 패키지, 기존 수정은 order/ 내부
  → 기능 기반이 파급 범위 더 명확

=== 컴포넌트 기반 ===
변경/신규 파일:
  ordercomponent/internal/ (수정)
  couponcomponent/api/ (신규, 외부 공개 API)
  couponcomponent/internal/ (신규, 내부 구현)

총 변경: 컴포넌트 경계가 명확, 실수로 coupon 내부에 접근 불가
```

### 5. 기능 기반 + Hexagonal 결합 (실무 권장)

```
실무에서 가장 많이 쓰이는 구조:
  기능(Feature) 기반 + 내부에 Hexagonal 레이어

com.example
├── order/                     ← Feature 경계
│   ├── domain/                ← Entities 레이어 (순수 Java)
│   │   ├── model/
│   │   └── port/
│   ├── application/           ← Use Cases 레이어
│   └── adapter/               ← Adapters 레이어
│       ├── in/web/
│       └── out/persistence/
├── payment/                   ← Feature 경계
│   ├── domain/
│   ├── application/
│   └── adapter/
└── shared/                    ← 공유 컴포넌트
    └── domain/                ← 공유 Value Object

이 구조의 특성:
  외부(Feature): 도메인 경계 시각화
  내부(Hexagonal): 의존성 규칙 강제
  → 두 가지 이익을 모두 얻음
```

---

## 💻 실전 코드 — 구조별 ArchUnit 검증 방법

```java
// 기능 기반 + Hexagonal 구조에서의 ArchUnit 규칙

@AnalyzeClasses(packages = "com.example")
public class PackageStructureTest {

    JavaClasses classes = new ClassFileImporter().importPackages("com.example");

    // 규칙 1: 기능 간 직접 내부 의존 금지 (기능 경계 보호)
    @Test
    void features_should_not_directly_depend_on_each_other_internals() {
        noClasses()
            .that().resideInAPackage("..order..")
            .should().dependOnClassesThat()
            .resideInAPackage("..payment.domain..")  // payment 내부 직접 접근 금지
            .check(classes);
    }

    // 규칙 2: 각 Feature의 domain이 adapter를 모름 (Hexagonal 규칙)
    @Test
    void domain_should_not_depend_on_adapter() {
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..adapter..")
            .check(classes);
    }

    // 규칙 3: 레이어 기반을 쓰더라도 Service가 다른 Feature의 Service를 직접 의존하면 안 됨
    @Test
    void services_should_not_directly_depend_on_other_feature_services() {
        // 이 규칙은 기능 기반이 아닌 레이어 기반 구조에서 도메인 경계를 지키기 위함
        noClasses()
            .that().haveNameMatching(".*Service")
            .and().resideInAPackage("..order..")
            .should().dependOnClassesThat()
            .haveNameMatching(".*Service")
            .and().resideInAPackage("..payment..")
            .check(classes);
    }
}
```

---

## 📊 패턴 비교 — 3가지 구조 비교표

```
기준                    │ 레이어 기반      │ 기능 기반       │ 컴포넌트 기반
────────────────────────┼────────────────┼───────────────┼──────────────────
도메인 경계 시각화       │ 없음            │ 패키지로 표현   │ 패키지 + 내/외부 구분
기능 추가 시 파일 범위   │ 여러 패키지 분산 │ 1-2 패키지 내  │ 1-2 컴포넌트 내
MSA 분리 용이성         │ 어려움          │ 쉬움           │ 쉬움
경계 강제 방법          │ 팀 합의만        │ ArchUnit       │ ArchUnit + 접근제어
학습 비용              │ 낮음            │ 중간            │ 높음
적합 팀 규모            │ 소규모          │ 중간            │ 대규모
```

---

## ⚖️ 트레이드오프

```
레이어 기반의 현실적 장점:
  Spring Boot 문서, 튜토리얼이 모두 이 구조
  모든 팀원이 즉시 이해
  30개 이하 파일에서 충분

기능 기반 전환 비용:
  "이 파일 어디 있더라?" 탐색 방식 변화 (익숙해지면 더 빠름)
  팀 합의 필요 (각 Feature의 내부 구조 기준)
  기존 프로젝트 리팩터링 비용

기능 기반이 이익을 주기 시작하는 시점:
  Feature가 5개 이상일 때
  팀원이 3명 이상일 때
  같은 Feature를 여러 명이 동시에 개발할 때
```

---

## 📌 핵심 정리

```
패키지 구조 원칙:

"함께 변경되는 것은 함께 있어야 한다" (Common Closure Principle)

레이어 기반: 기술(레이어)로 분류 → Feature가 여러 패키지에 분산
기능 기반:  도메인(Feature)으로 분류 → Feature가 한 패키지에 집중
컴포넌트 기반: 기능 + 내/외부 경계 명시 → 가장 강한 경계 보호

선택 기준:
  파일 수 30개 이하 → 레이어 기반
  Feature 5개 이상 → 기능 기반
  대규모 팀 + 경계 침범 문제 → 컴포넌트 기반

실무 권장:
  기능 기반 + 내부에 Hexagonal 레이어
  = 도메인 경계 시각화 + 의존성 방향 규칙
```

---

## 🤔 생각해볼 문제

**Q1.** 레이어 기반 구조에서 기능 기반으로 전환할 때 가장 어려운 점은 무엇인가?

<details>
<summary>해설 보기</summary>

**가장 어려운 점: 도메인 경계를 어디로 그을지 결정하는 것입니다.**

레이어 기반에서는 OrderService에 주문, 결제, 재고 로직이 혼재되어 있는 경우가 많습니다. 기능 기반으로 전환하려면 "이 로직은 order 기능인가, payment 기능인가"를 결정해야 합니다.

실용적 접근:
1. 데이터 소유권으로 경계 결정 — "이 데이터를 누가 생성하고 관리하는가"
2. 변경 이유로 경계 결정 — "이 두 로직은 같은 이유로 변경되는가"
3. 팀 소유권으로 경계 결정 — "이 기능을 어느 팀/개발자가 담당하는가"

처음에는 경계가 잘못될 수 있습니다. 시간이 지나면서 리팩터링하며 조정하는 것이 자연스럽습니다.

</details>

---

**Q2.** 기능 기반 구조에서 Order가 User 정보를 필요로 할 때, user/ 패키지를 직접 import해도 되는가?

<details>
<summary>해설 보기</summary>

**구체 클래스는 금지, 인터페이스 또는 공유 DTO는 허용입니다.**

```java
// ❌ 금지: order가 user 내부 구체 클래스 직접 import
import com.example.user.UserService;
import com.example.user.User;

// ✅ 허용 방법 1: 공유 Value Object (shared 패키지)
import com.example.shared.UserId;  // 원시 타입 래퍼

// ✅ 허용 방법 2: Port 추상화
// order/domain/port/out/UserQueryPort.java (order 패키지 내부)
public interface UserQueryPort {
    boolean isUserActive(UserId userId);
}
// user 패키지가 이 인터페이스를 구현

// ✅ 허용 방법 3: 이벤트 기반 통신
// order가 UserDeactivatedEvent를 구독
// → user.User 직접 import 없음
```

원칙: Feature 간 통신은 공유 Value Object, Port 인터페이스, 이벤트 중 하나를 통해서만.

</details>

---

**Q3.** 컴포넌트 기반 구조에서 `internal/` 패키지 접근을 Java의 접근 제한자로는 막을 수 없는데, 어떻게 강제하는가?

<details>
<summary>해설 보기</summary>

**Java 패키지는 `public`으로 선언하면 어디서든 접근 가능**합니다. 컴포넌트 경계는 Java 컴파일러가 아니라 ArchUnit으로 강제합니다.

```java
@Test
void components_should_only_access_api_packages() {
    noClasses()
        .that().resideOutsideOfPackage("..ordercomponent..")  // order 외부에서
        .should().dependOnClassesThat()
        .resideInAPackage("..ordercomponent.internal..")  // internal 접근 금지
        .check(classes);
}
```

Gradle 멀티 모듈로 더 강하게 강제:
- `order-api` 모듈: 공개 인터페이스만
- `order-impl` 모듈: 내부 구현, `api` 모듈에만 의존
- 다른 모듈은 `order-api`만 `compile` 의존성 가능

→ 컴파일 시점에 `internal` 클래스를 다른 모듈에서 볼 수 없음 (멀티 모듈의 핵심 이익).

</details>

---

<div align="center">

**[⬅️ 이전 챕터: MSA와 아키텍처 패턴](../architecture-comparison/05-msa-and-architecture-patterns.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Boot 실전 패키지 구조 ➡️](./02-spring-hexagonal-package.md)**

</div>
