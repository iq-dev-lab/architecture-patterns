# ArchUnit으로 아키텍처 테스트 — 의존성 규칙을 코드로 강제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ArchUnit 기본 문법과 설정 방법은?
- "도메인 레이어는 Spring 어노테이션을 사용하지 않는다" 규칙을 코드로 어떻게 작성하는가?
- "infrastructure는 domain에만 의존한다" 규칙 코드는?
- CI 파이프라인에서 아키텍처 규칙을 자동 검증하는 방법은?
- ArchUnit 위반 에러 메시지를 어떻게 해석하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

팀이 아키텍처 규칙에 합의해도, 코드 리뷰에서 모든 위반을 잡기는 어렵다. "급하게 개발하다 보니" 실수가 생기고, 그 실수가 누적되면 처음에 의도한 아키텍처가 무너진다.

ArchUnit은 아키텍처 규칙을 테스트 코드로 작성해서 CI 파이프라인이 자동으로 검증하게 만든다. 인간의 코드 리뷰 대신 기계가 검증하므로, 규칙이 팀 합의가 아닌 자동화된 강제 수단이 된다.

---

## 😱 흔한 실수 (Before — 코드 리뷰에만 의존할 때)

```
코드 리뷰 기반 규칙 적용의 한계:

리뷰어: "도메인에 @Entity 붙이지 말자고 합의했잖아요?"
개발자: "아, 맞다. 급해서 깜빡했어요. 나중에 분리할게요."

6개월 후:
  Order.java: @Entity 있음
  Payment.java: @Entity 없음
  User.java: @Entity 있음
  → 세 가지 다른 스타일 혼재

리뷰어가 놓치는 이유:
  "이번 PR은 비즈니스 로직이 핵심이라 아키텍처 규칙까지 꼼꼼히 못 봤어요"
  "PR이 300줄인데 어디가 위반인지 파악하기 어려워요"
  리뷰어도 규칙을 완전히 기억하지 못하는 경우

ArchUnit이 있었다면:
  CI에서 즉시 에러:
  "Architecture Violation: Order (com.example.domain.Order) 
   uses @Entity which belongs to package jakarta.persistence.
   Rule: Domain classes must not use JPA annotations"
  → PR 머지 전에 자동 차단
```

---

## ✨ 올바른 접근 (After — ArchUnit으로 CI 자동 검증)

```
ArchUnit 적용 효과:

코드 리뷰 역할 변화:
  Before: "이 코드가 아키텍처 규칙을 지키는가?" → 리뷰어가 수동 체크
  After:  "이 코드가 비즈니스 로직을 올바르게 구현하는가?" → 리뷰어가 집중 가능
  아키텍처 규칙 체크 → CI 자동화로 위임

ArchUnit 에러 메시지의 가치:
  "com.example.domain.Order uses @Entity" 에러가 나오면
  개발자: "아 Domain이 JPA를 모르게 해야 한다는 규칙이구나"
  → 에러 메시지 자체가 교육 효과
  → 신규 팀원이 실수하면 CI가 가르쳐줌
```

---

## 🔬 내부 원리 — ArchUnit 설정과 핵심 규칙

### 1. ArchUnit 기본 설정

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("com.tngtech.archunit:archunit-junit5:1.2.1")
}

// ArchUnit의 기본 사용 방식

// 방법 1: @AnalyzeClasses 어노테이션 (권장)
@AnalyzeClasses(
    packages = ["com.example"],          // 분석할 패키지
    importOptions = [ImportOption.DoNotIncludeTests::class]  // 테스트 코드 제외
)
class ArchitectureTest {
    @ArchTest
    val domain_should_not_use_spring: ArchRule = ...
}

// 방법 2: 테스트 메서드에서 직접 생성
class ArchitectureTest {
    val classes: JavaClasses = ClassFileImporter()
        .withImportOption(ImportOption.Companion.DoNotIncludeTests())
        .importPackages("com.example")

    @Test
    fun `domain should not use spring`() {
        val rule = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("org.springframework..")
        rule.check(classes)
    }
}
```

### 2. 핵심 규칙 코드 모음

```java
@AnalyzeClasses(packages = "com.example", importOptions = ImportOption.DoNotIncludeTests.class)
public class HexagonalArchitectureTest {

    // ============================================================
    // 규칙 1: Domain이 Spring을 모름
    // ============================================================
    @ArchTest
    static final ArchRule domain_should_not_use_spring =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("org.springframework..")
            .because("Domain 레이어는 Spring 프레임워크를 몰라야 합니다. " +
                     "ADR-002 참고: docs/adr/0002-domain-purity.md");

    // ============================================================
    // 규칙 2: Domain이 JPA를 모름
    // ============================================================
    @ArchTest
    static final ArchRule domain_should_not_use_jpa =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("jakarta.persistence..", "javax.persistence..")
            .because("Domain Entity는 JPA @Entity가 아닙니다. " +
                     "JPA Entity는 adapter/out/persistence/entity/에 위치합니다.");

    // ============================================================
    // 규칙 3: Domain이 Adapter를 모름 (의존성 방향)
    // ============================================================
    @ArchTest
    static final ArchRule domain_should_not_depend_on_adapter =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..adapter..")
            .because("Domain은 Adapter를 몰라야 합니다. 의존성은 Adapter → Domain입니다.");

    // ============================================================
    // 규칙 4: Application이 Infrastructure를 모름
    // ============================================================
    @ArchTest
    static final ArchRule application_should_not_depend_on_infrastructure =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..adapter.out..", "..infrastructure..")
            .because("Application 레이어는 구체 인프라 구현을 몰라야 합니다.");

    // ============================================================
    // 규칙 5: Controller가 Domain을 직접 생성하면 안 됨
    // ============================================================
    @ArchTest
    static final ArchRule controller_should_not_access_domain_directly =
        noClasses()
            .that().resideInAPackage("..adapter.in.web..")
            .should().dependOnClassesThat()
            .resideInAPackage("..domain.model..")
            .because("Controller는 UseCase(Port)를 통해서만 도메인에 접근합니다. " +
                     "Controller → UseCase → Service → Domain 순서를 지키세요.");

    // ============================================================
    // 규칙 6: Port 인터페이스 위치 강제
    // ============================================================
    @ArchTest
    static final ArchRule ports_should_reside_in_domain_port =
        classes()
            .that().haveNameMatching(".*(UseCase|Port|Query)")
            .and().areInterfaces()
            .should().resideInAPackage("..domain.port..")
            .because("UseCase 인터페이스와 Port 인터페이스는 domain/port/ 패키지에 위치합니다.");

    // ============================================================
    // 규칙 7: JPA Entity는 adapter/out/persistence/entity/에만
    // ============================================================
    @ArchTest
    static final ArchRule jpa_entities_should_reside_in_persistence_entity =
        classes()
            .that().areAnnotatedWith("jakarta.persistence.Entity")
            .should().resideInAPackage("..adapter.out.persistence.entity..")
            .because("JPA @Entity 클래스는 adapter/out/persistence/entity/에만 위치합니다. " +
                     "Domain 객체(Order.java)와 JPA Entity(OrderJpaEntity.java)는 분리됩니다.");

    // ============================================================
    // 규칙 8: @Service 어노테이션은 application/service/에만
    // ============================================================
    @ArchTest
    static final ArchRule services_should_reside_in_application_service =
        classes()
            .that().areAnnotatedWith(Service.class)
            .and().haveNameNotMatching(".*Test.*")
            .should().resideInAPackage("..application.service..")
            .because("@Service 클래스(UseCase 구현체)는 application/service/에 위치합니다.");

    // ============================================================
    // 규칙 9: Feature 간 직접 의존 금지 (도메인 경계)
    // ============================================================
    @ArchTest
    static final ArchRule order_should_not_depend_on_payment_domain =
        noClasses()
            .that().resideInAPackage("..order..")
            .should().dependOnClassesThat()
            .resideInAPackage("..payment.domain..")
            .because("Order와 Payment는 독립적인 Bounded Context입니다. " +
                     "PaymentPort 인터페이스나 이벤트를 통해서만 통신하세요.");

    // ============================================================
    // 규칙 10: Adapter에 비즈니스 로직 금지
    // ============================================================
    @ArchTest
    static final ArchRule adapters_should_not_contain_business_logic =
        noClasses()
            .that().resideInAPackage("..adapter..")
            .should().dependOnClassesThat()
            .haveSimpleNameContaining("Policy")  // 비즈니스 정책 클래스
            .orShould().dependOnClassesThat()
            .haveSimpleNameContaining("Rule")    // 비즈니스 규칙 클래스
            .because("Adapter는 변환(Translation)만 담당합니다. " +
                     "비즈니스 로직은 Domain 또는 Application에 위치합니다.");
}
```

### 3. 레이어드 아키텍처 내장 검증

```java
// ArchUnit 내장 레이어드 아키텍처 검증 API

@AnalyzeClasses(packages = "com.example")
public class LayeredArchTest {

    @ArchTest
    static final ArchRule hexagonal_architecture =
        layeredArchitecture()
            .consideringAllDependencies()
            .layer("Domain").definedBy("..domain..")
            .layer("Application").definedBy("..application..")
            .layer("Adapter").definedBy("..adapter..")
            .whereLayer("Domain").mayNotAccessAnyLayer()     // Domain은 아무것도 의존 X
            .whereLayer("Application").mayOnlyAccessLayers("Domain")
            .whereLayer("Adapter").mayOnlyAccessLayers("Domain", "Application");

    // 예외 처리 (절충 허용 시)
    @ArchTest
    static final ArchRule hexagonal_with_exceptions =
        layeredArchitecture()
            .consideringAllDependencies()
            .layer("Domain").definedBy("..domain..")
            .layer("Application").definedBy("..application..")
            .whereLayer("Domain").mayNotAccessAnyLayer()
            .whereLayer("Application").mayOnlyAccessLayers("Domain")
            // 예외: @Transactional (Spring 의존 허용)
            .ignoreDependency(
                describedAs("@Transactional in Application"),
                resideInAPackage("..application.."),
                areAssignableTo(org.springframework.transaction.annotation.Transactional.class)
            );
}
```

### 4. CI 파이프라인 통합

```yaml
# GitHub Actions 설정 (.github/workflows/architecture-test.yml)

name: Architecture Tests

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

jobs:
  architecture-test:
    name: ArchUnit Architecture Tests
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Gradle packages
        uses: actions/cache@v3
        with:
          path: ~/.gradle/caches
          key: gradle-${{ hashFiles('**/*.gradle.kts') }}

      - name: Run Architecture Tests
        run: ./gradlew test --tests "*ArchTest*" --tests "*Architecture*"

      - name: Upload Test Report
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: architecture-test-report
          path: build/reports/tests/

# 별도 Job으로 분리하면:
# - 아키텍처 위반은 비즈니스 테스트 실패와 별도로 즉시 발견
# - PR 체크에서 "Architecture Tests ❌" 명확히 표시
```

### 5. 에러 메시지 해석 방법

```
ArchUnit 에러 메시지 예시:

java.lang.AssertionError: Architecture Violation [Priority: MEDIUM]
Rule 'classes that reside in a package '..domain..' should not depend
on classes that reside in a package 'org.springframework..'
was violated (2 times):
  Field <com.example.order.domain.model.Order.status>
  has type <org.springframework.data.annotation.ReadOnlyProperty>
  in (Order.java:15)
  
  Constructor <com.example.order.domain.service.OrderValidator.<init>()>
  calls method <org.springframework.util.Assert.notNull(...)>
  in (OrderValidator.java:22)

해석 방법:

1. 위반 규칙 파악: "domain이 org.springframework..를 의존하면 안 됨"
2. 위반 위치: Order.java:15, OrderValidator.java:22
3. 위반 내용:
   - Order.status 필드가 Spring의 @ReadOnlyProperty를 사용
   - OrderValidator 생성자에서 Spring의 Assert를 사용

수정 방법:
   - Order.status의 @ReadOnlyProperty 제거 (사용 의도 파악 후)
   - OrderValidator에서 Spring Assert 대신 Java 표준으로:
     Assert.notNull(x) → Objects.requireNonNull(x)

because() 메시지의 역할:
  에러 메시지에 "ADR-002 참고: docs/adr/0002.md"를 추가하면
  → 개발자가 "왜 이 규칙이 있는가"를 즉시 찾을 수 있음
  → 에러 메시지가 가이드가 됨
```

---

## 💻 실전 코드 — 전체 ArchUnit 설정 파일

```java
// 프로젝트에 바로 사용 가능한 전체 ArchUnit 설정

@AnalyzeClasses(
    packages = "com.example",
    importOptions = ImportOption.DoNotIncludeTests.class
)
public class ProjectArchitectureTest {

    @ArchTest
    static final ArchRule domain_no_spring =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("org.springframework..")
            .because("ADR-002: domain 레이어는 Spring을 몰라야 합니다.");

    @ArchTest
    static final ArchRule domain_no_jpa =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("jakarta.persistence..", "javax.persistence..")
            .because("ADR-004: Domain Entity는 JPA @Entity가 아닙니다.");

    @ArchTest
    static final ArchRule domain_no_adapter =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..adapter..")
            .because("의존성은 Adapter → Domain 방향이어야 합니다.");

    @ArchTest
    static final ArchRule application_no_infrastructure =
        noClasses().that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..adapter.out..", "..infrastructure..")
            .because("Application은 구체 인프라를 몰라야 합니다.");

    @ArchTest
    static final ArchRule ports_in_domain =
        classes().that().haveNameMatching(".*(UseCase|Port|Query)")
            .and().areInterfaces()
            .should().resideInAPackage("..domain.port..")
            .because("Port 인터페이스는 domain/port/ 패키지에 위치합니다.");

    @ArchTest
    static final ArchRule jpa_entities_in_adapter =
        classes().that().areAnnotatedWith("jakarta.persistence.Entity")
            .should().resideInAPackage("..adapter.out.persistence.entity..")
            .because("@Entity 클래스는 adapter/out/persistence/entity/에만 위치합니다.");

    @ArchTest
    static final ArchRule services_in_application =
        classes().that().areAnnotatedWith(Service.class)
            .should().resideInAPackage("..application.service..")
            .because("@Service(UseCase 구현체)는 application/service/에 위치합니다.");

    // 위반 시 CI 에러 메시지 예시:
    // Architecture Violation: Order uses @Entity (jakarta.persistence.Entity)
    // → adapter/out/persistence/entity/ 로 이동하거나 @Entity 제거 필요
    // → ADR-004 참고: docs/adr/0004-entity-separation.md
}
```

---

## 📊 패턴 비교 — ArchUnit 규칙 복잡도별 예시

```
단순 규칙 (패키지 의존성):
  noClasses().that().resideInAPackage("..domain..")
  .should().dependOnClassesThat().resideInAPackage("org.springframework..")
  → "이 패키지의 클래스는 저 패키지를 import하면 안 됨"

중간 규칙 (어노테이션 + 위치):
  classes().that().areAnnotatedWith(Entity.class)
  .should().resideInAPackage("..adapter.out.persistence.entity..")
  → "@Entity 클래스는 특정 패키지에만 있어야 함"

복잡 규칙 (이름 패턴 + 구현 관계):
  classes().that().haveNameMatching(".*(UseCase|Port)")
  .and().areInterfaces()
  .should().resideInAPackage("..domain.port..")
  → "이름이 UseCase/Port로 끝나는 인터페이스는 domain/port/에만"

레이어 전체 규칙:
  layeredArchitecture()
  .layer("Domain").definedBy("..domain..")
  .whereLayer("Domain").mayNotAccessAnyLayer()
  → "Domain 레이어는 어떤 레이어도 의존하지 않음"
```

---

## ⚖️ 트레이드오프

```
ArchUnit의 비용:
  테스트 코드 작성 시간 (처음: 2-4시간, 이후 규칙 추가: 10-30분)
  빌드 시간 증가: ArchUnit 테스트 자체 실행 시간 (보통 10-30초)
  규칙이 너무 엄격하면 예외 관리 필요

ArchUnit의 이익:
  코드 리뷰에서 아키텍처 체크 부담 감소
  신규 팀원 실수 자동 차단
  규칙 위반 즉시 발견 (PR 단계에서)
  에러 메시지가 가이드 역할 (because() 활용)

예외 처리 전략:
  의도적 예외: ignoreDependency() 또는 @ArchIgnore
  모든 예외는 이유를 코드 주석으로 명시
  예외가 3개 이상 → 규칙 자체를 재검토 신호
```

---

## 📌 핵심 정리

```
ArchUnit 핵심:

설정:
  testImplementation("com.tngtech.archunit:archunit-junit5:1.2.1")
  @AnalyzeClasses(packages = "com.example")

핵심 규칙 패턴:
  noClasses().that().[위치].should().[위반 행위]  ← 금지 규칙
  classes().that().[조건].should().[위치]         ← 위치 강제 규칙
  layeredArchitecture()...                        ← 전체 레이어 규칙

because() 활용:
  규칙마다 "왜 이 규칙이 있는가" + ADR 링크 추가
  에러 메시지가 가이드 역할 → 신규 팀원 교육 효과

CI 통합:
  PR마다 Architecture Test 별도 Job으로 실행
  위반 즉시 PR 차단 → 인간 코드 리뷰 이전에 자동 감지

코드 리뷰어 역할 변화:
  Before: "아키텍처 규칙 지켰는가?" 체크 포함
  After: "비즈니스 로직과 설계"에만 집중
  → ArchUnit이 규칙 체크를 담당
```

---

## 🤔 생각해볼 문제

**Q1.** ArchUnit 테스트가 CI에서 통과했는데도 런타임에 아키텍처 위반 문제가 발생할 수 있는가?

<details>
<summary>해설 보기</summary>

**네, 가능합니다.** ArchUnit은 **정적 분석**(컴파일된 바이트코드 분석)이므로 런타임에만 나타나는 문제는 감지하지 못합니다.

ArchUnit이 감지하지 못하는 것:
1. **리플렉션으로 우회**: `Class.forName("JpaRepository")` 같은 동적 로딩
2. **AOP 프록시**: Spring이 런타임에 생성하는 프록시 의존성
3. **Spring Bean 이름으로 주입**: `@Qualifier("jpaOrderRepository")`로 구체 클래스 주입
4. **설정 클래스에서 new 직접 생성**: `@Bean` 메서드에서 `new JpaOrderRepository()` 직접

이를 보완하기 위해:
- 통합 테스트(`@SpringBootTest`)로 실제 Bean 주입 검증
- 아키텍처 결정을 Gradle 멀티 모듈로도 강제 (컴파일 시점)
- 두 가지를 함께 사용

</details>

---

**Q2.** ArchUnit 규칙이 너무 많아지면 어떤 문제가 생기고, 어떻게 관리하는가?

<details>
<summary>해설 보기</summary>

**규칙이 너무 많아지면 "어떤 규칙을 지켜야 하는지 모르는" 상황이 됩니다.**

문제들:
1. 빌드 시간 증가 (규칙 50개 이상이면 30-60초 추가)
2. 예외가 쌓이면 규칙의 신뢰도 하락
3. 신규 팀원이 규칙 목록을 이해하기 어려움

관리 전략:
```java
// 규칙을 핵심/선택으로 분류
public class CoreArchRules {
    // 절대 예외 없는 핵심 규칙 (3-5개만)
    @ArchTest static ArchRule domain_no_spring = ...;
    @ArchTest static ArchRule domain_no_jpa = ...;
    @ArchTest static ArchRule domain_no_adapter = ...;
}

public class AdditionalArchRules {
    // 팀 합의에 따른 추가 규칙
    @ArchTest static ArchRule jpa_entity_location = ...;
}
```

핵심 원칙: 절대 지켜야 하는 규칙 3-5개를 정하고, 나머지는 가이드라인으로 운영.

</details>

---

**Q3.** ArchUnit이 `@ArchIgnore`를 지원한다. 이것을 남용하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

`@ArchIgnore`는 특정 클래스나 규칙 위반을 무시하게 만듭니다. 남용하면 **"규칙이 있지만 지키지 않아도 되는 상태"**가 됩니다.

```java
// ArchIgnore 남용 예시
@ArchIgnore // "나중에 고칠게요" → 6개월째 그대로
@Entity
public class Order {
    @Autowired // @ArchIgnore로 규칙 무시
    private OrderService service; // 도메인이 서비스를 의존 (심각한 위반)
}
```

대안적 접근:
1. `@ArchIgnore` 대신 `ignoreDependency()`를 규칙에 명시적으로 추가 → 이유를 코드에 남김
2. `@ArchIgnore`를 사용할 때 이유를 주석으로 필수 작성 + 만료 날짜 명시
3. `@ArchIgnore`가 3개 이상 생기면 → 이 클래스를 리팩터링 대상 목록에 추가

규칙: `@ArchIgnore`를 추가하려면 팀 리뷰를 반드시 거치도록 PR 가이드라인에 명시.

</details>

---

<div align="center">

**[⬅️ 이전: 멀티 모듈 프로젝트](./03-multi-module-project.md)** | **[홈으로 🏠](../README.md)** | **[다음: 코드 가이드라인 수립 ➡️](./05-code-guidelines.md)**

</div>
