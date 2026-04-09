# 멀티 모듈 프로젝트 — Gradle로 의존성을 컴파일 시점에 강제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Gradle 멀티 모듈로 레이어 간 의존성을 컴파일 시점에 강제하는 방법은?
- Domain 모듈이 Infrastructure 모듈을 참조할 수 없도록 설정하는 구체적 방법은?
- 멀티 모듈 구성(domain / application / infrastructure / presentation) 예시는?
- 빌드 속도와 모듈 경계 유지의 트레이드오프는?
- 단일 모듈 ArchUnit과 멀티 모듈의 핵심 차이는?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

ArchUnit은 "런타임에 테스트가 실패"하면 규칙 위반을 알려준다. 그러나 멀티 모듈 Gradle에서는 "컴파일 시점에 에러"가 발생한다.

이 차이는 크다. ArchUnit은 개발자가 테스트를 실행해야 검증되지만, Gradle 멀티 모듈은 `import org.springframework.data.jpa.*`를 도메인 모듈에 추가하는 순간 `./gradlew build`가 실패한다. "도메인이 JPA를 모른다"는 규칙을 언어 레벨에서 강제하는 것이다.

---

## 😱 흔한 실수 (Before — 단일 모듈에서 규칙을 팀 합의에만 의존)

```
단일 모듈의 한계:

com.example (단일 모듈)
  ├── domain/
  │   └── Order.java      ← "@Entity를 붙이지 말자" → 합의만
  ├── application/
  │   └── OrderService.java
  └── infrastructure/
      └── JpaOrderRepository.java

문제:
  개발자가 실수로 Order.java에 @Entity 추가
  → 컴파일: 통과 (같은 모듈에 JPA가 있으므로)
  → ArchUnit 테스트 실행: 위반 감지
  → 하지만 ArchUnit을 실행하지 않으면? 감지 못함

  "급하니까 @SpringBootTest에서 직접 JpaRepository 주입"
  → 컴파일: 통과
  → ArchUnit: 실행 안 했으면 모름
  → CI에서 뒤늦게 발견

멀티 모듈이라면:
  domain 모듈에 JPA 의존성 없음
  → Order.java에 import jakarta.persistence.Entity 추가 순간
  → 컴파일 에러: "package jakarta.persistence does not exist"
  → 즉시 발견, ArchUnit 실행 불필요
```

---

## ✨ 올바른 접근 (After — Gradle 모듈로 컴파일 시점 강제)

```
멀티 모듈 구조:

settings.gradle.kts:
  include(":domain")
  include(":application")
  include(":infrastructure")
  include(":presentation")

의존성 방향:
  presentation → application → domain ← infrastructure

Gradle 의존성 설정:
  domain:           의존성 없음 (순수 Java)
  application:      implementation(project(":domain"))
  infrastructure:   implementation(project(":domain"))
                    implementation("org.springframework.data:spring-data-jpa")
  presentation:     implementation(project(":application"))
                    implementation("org.springframework.boot:spring-boot-starter-web")

결과:
  domain 모듈에서 JPA import → 컴파일 에러 (JPA 의존성 없음)
  application 모듈에서 HTTP import → 컴파일 에러 (Web 의존성 없음)
  → 규칙이 언어 레벨에서 강제됨
```

---

## 🔬 내부 원리 — Gradle 멀티 모듈 설정

### 1. 전체 프로젝트 구조

```
ecommerce/                    ← 루트 프로젝트
├── settings.gradle.kts
├── build.gradle.kts          ← 공통 설정
├── domain/                   ← 도메인 모듈
│   ├── build.gradle.kts
│   └── src/main/java/
│       └── com/example/
│           └── order/
│               ├── domain/model/
│               └── domain/port/
├── application/              ← 유스케이스 모듈
│   ├── build.gradle.kts
│   └── src/main/java/
│       └── com/example/
│           └── order/application/
├── infrastructure/           ← 인프라 모듈
│   ├── build.gradle.kts
│   └── src/main/java/
│       └── com/example/
│           └── order/adapter/out/
└── presentation/             ← 표현 모듈
    ├── build.gradle.kts
    └── src/main/java/
        └── com/example/
            └── order/adapter/in/
```

### 2. Gradle 설정 파일

```kotlin
// settings.gradle.kts (루트)
rootProject.name = "ecommerce"

include(":domain")
include(":application")
include(":infrastructure")
include(":presentation")

// build.gradle.kts (루트 공통 설정)
plugins {
    id("org.springframework.boot") version "3.2.0" apply false
    id("io.spring.dependency-management") version "1.1.4" apply false
    kotlin("jvm") version "1.9.21" apply false
}

subprojects {
    apply(plugin = "java")
    apply(plugin = "io.spring.dependency-management")

    group = "com.example"
    version = "0.0.1-SNAPSHOT"

    repositories {
        mavenCentral()
    }

    dependencies {
        // 공통 의존성
        testImplementation("org.junit.jupiter:junit-jupiter")
        testImplementation("org.assertj:assertj-core")
    }
}

// domain/build.gradle.kts
// 의존성: 없음 (순수 Java)
plugins {
    java
}

dependencies {
    // 의도적으로 비어 있음
    // java.util.*, java.time.*, java.math.* 외 의존성 없음
    // JPA, Spring, Kafka 없음 → 이것이 컴파일 시점 강제의 핵심
}

// application/build.gradle.kts
plugins {
    java
}

dependencies {
    implementation(project(":domain"))      // domain만 의존
    // JPA, Web 없음 → application에서 JPA import 불가
    implementation("org.springframework:spring-tx") // @Transactional만 허용 (선택)
}

// infrastructure/build.gradle.kts
plugins {
    java
    id("org.springframework.boot") apply false
    id("io.spring.dependency-management")
}

dependencies {
    implementation(project(":domain"))      // domain 의존
    implementation(project(":application")) // application 의존 (Port 인터페이스 위해)
    
    // 인프라 의존성 (여기에만 있음)
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.kafka:spring-kafka")
    implementation("com.mysql:mysql-connector-j")
}

// presentation/build.gradle.kts
dependencies {
    implementation(project(":application"))  // application만 의존
    // domain은 transitive로 접근 가능하지만 직접 import는 제한 가능
    
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
}
```

### 3. 의존성 위반 시 컴파일 에러 예시

```java
// domain 모듈에서 JPA 사용 시도

// domain/src/main/java/.../Order.java
import jakarta.persistence.Entity; // ← 컴파일 에러 발생!
// 에러: error: package jakarta.persistence does not exist
// 이유: domain/build.gradle.kts에 JPA 의존성 없음

@Entity // ← 이미 컴파일 에러로 이 라인에 도달 불가
public class Order {
}

// application 모듈에서 HTTP 사용 시도

// application/src/main/java/.../PlaceOrderService.java
import org.springframework.web.bind.annotation.RestController; // ← 컴파일 에러!
// 에러: package org.springframework.web.bind.annotation does not exist
// 이유: application/build.gradle.kts에 Web 의존성 없음

// presentation 모듈에서 JPA 직접 사용 시도

// presentation/src/main/java/.../OrderController.java
import com.example.infrastructure.JpaOrderRepository; // ← 가능 (transitive)
// 그러나 ArchUnit으로 추가 제한 가능:
// "presentation은 application만 의존해야 함"
```

### 4. 기능 기반 멀티 모듈 (Feature Module)

```kotlin
// 아키텍처 레이어 기반이 아닌 기능 기반 멀티 모듈

settings.gradle.kts:
  include(":shared")           // 공유 Value Object, 설정
  include(":order")            // Order 기능 전체
  include(":payment")          // Payment 기능 전체
  include(":notification")     // Notification 기능 전체
  include(":app")              // Spring Boot 실행 모듈

// order/build.gradle.kts
dependencies {
    implementation(project(":shared"))      // 공유 컴포넌트만
    // payment, notification 직접 의존 없음 → 이벤트/Port로만 통신
    
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
}

// app/build.gradle.kts (Spring Boot 실행 진입점)
dependencies {
    implementation(project(":order"))
    implementation(project(":payment"))
    implementation(project(":notification"))
    implementation("org.springframework.boot:spring-boot-starter")
}

// 장점:
//   order ↔ payment 직접 import → 컴파일 에러 (모듈 의존성 없음)
//   기능 추가 = 새 모듈 추가, 기존 모듈 변경 없음
//   빌드 시 변경된 모듈만 재컴파일 (빌드 속도 향상)
```

### 5. 빌드 캐시와 속도 최적화

```kotlin
// 멀티 모듈 빌드 최적화 설정

// gradle.properties
org.gradle.caching=true         // Gradle 빌드 캐시 활성화
org.gradle.parallel=true        // 병렬 빌드
org.gradle.configureondemand=true // 필요한 프로젝트만 설정

// settings.gradle.kts
buildCache {
    local {
        isEnabled = true
        directory = File(rootDir, "build-cache")
    }
}

// 빌드 속도 효과:
// 전체 빌드: 60초
// domain 변경 후 재빌드:
//   → domain, application, infrastructure, presentation 모두 재컴파일
//   → 60초 (전체 영향)
// presentation만 변경:
//   → presentation만 재컴파일
//   → 10초 (영향 범위 최소)
// domain 변경이 가장 비쌈 → domain을 가장 안정적으로 유지해야 하는 이유!
```

---

## 💻 실전 코드 — 실제 동작하는 멀티 모듈 설정

```kotlin
// 완전한 settings.gradle.kts 예시

rootProject.name = "order-service"

include(
    ":domain",
    ":application",
    ":infrastructure",
    ":presentation",
    ":bootstrap"     // Spring Boot 시작점 (main 함수)
)

// bootstrap/build.gradle.kts
plugins {
    id("org.springframework.boot")
    id("io.spring.dependency-management")
    java
}

dependencies {
    implementation(project(":presentation"))
    implementation(project(":infrastructure"))
    // bootstrap이 모든 것을 알아서 Spring DI 조립
    // domain, application은 transitive로 포함
}

// Spring Boot의 main 함수 위치
// bootstrap/src/main/java/.../OrderServiceApplication.java

@SpringBootApplication
@ComponentScan(basePackages = ["com.example"])  // 모든 모듈 스캔
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// application.yml (bootstrap/src/main/resources/)
// JPA, Kafka 등 인프라 설정 위치
```

---

## 📊 패턴 비교 — 단일 모듈 vs 멀티 모듈

```
규칙 강제 방식 비교:

                    단일 모듈         멀티 모듈
────────────────────┼───────────────┼────────────────────────
규칙 위반 감지 시점  │ ArchUnit 실행  │ 컴파일 (즉시)
위반 예시           │ 테스트 실패     │ Build 실패
감지 강도           │ 팀이 테스트 실행│ 빌드 자체가 불가
빌드 복잡도         │ 낮음           │ 높음
빌드 속도 (부분 변경)│ 전체 재컴파일  │ 변경 모듈만 재컴파일
초기 설정 비용       │ 없음           │ 1-3일
학습 비용           │ 낮음           │ 중간 (Gradle 이해 필요)
적합한 프로젝트      │ 소규모         │ 중대규모, 명확한 경계 필요
```

---

## ⚖️ 트레이드오프

```
멀티 모듈의 비용:
  초기 설정: Gradle 멀티 모듈 구성 1-3일
  IDE 설정: 모듈 간 탐색 설정 (IntelliJ는 자동 처리)
  팀 학습: Gradle 모듈 개념 이해 필요
  복잡한 테스트: 통합 테스트 시 여러 모듈 의존 설정

멀티 모듈의 이익:
  의존성 규칙이 컴파일 시점에 강제됨
  부분 빌드로 빌드 속도 개선
  모듈별 독립 배포 가능성 (필요 시)

실용적 결론:
  팀 3명 이하 + 빠른 개발 → 단일 모듈 + ArchUnit
  팀 5명 이상 + 경계 침범 문제 → 멀티 모듈 검토
  MSA 분리 계획 → 기능 기반 멀티 모듈
```

---

## 📌 핵심 정리

```
Gradle 멀티 모듈 핵심:

모듈 구조:
  domain: 의존성 없음 (JPA, Spring 없음)
  application: domain만 의존
  infrastructure: domain + JPA + Kafka
  presentation: application + Web

의존성 방향 강제 방식:
  domain에 JPA import → 컴파일 에러 (의존성 없음)
  application에 HTTP import → 컴파일 에러
  = ArchUnit 없이도 규칙 강제!

단일 모듈 vs 멀티 모듈:
  단일 + ArchUnit: 테스트 실행 시 감지
  멀티 모듈: 컴파일 시 즉시 감지 (더 강력)

빌드 최적화:
  변경된 모듈만 재컴파일 (Gradle 캐시)
  domain 변경 = 모든 모듈 재컴파일 (가장 비쌈 → 안정적으로 유지)
```

---

## 🤔 생각해볼 문제

**Q1.** 멀티 모듈에서 `presentation` 모듈이 `domain` 모듈의 클래스를 직접 사용할 수 있는가? (transitive 의존성)

<details>
<summary>해설 보기</summary>

Gradle의 기본 `implementation`은 transitive 의존성을 **런타임에는 포함하지만 컴파일 시점에는 노출하지 않습니다.**

```kotlin
// presentation/build.gradle.kts
dependencies {
    implementation(project(":application"))  // application을 의존
    // application이 implementation(project(":domain"))을 가지므로
    // → presentation에서 domain 클래스를 직접 import하면 컴파일 에러!
}

// 만약 허용하고 싶다면 api() 사용:
// application/build.gradle.kts
dependencies {
    api(project(":domain"))  // api()는 transitive로 노출
}
// → presentation에서 domain 클래스 직접 import 가능

// 결론:
// implementation: transitive 숨김 (엄격한 경계)
// api: transitive 노출 (유연한 경계)
```

일반적으로 `implementation`을 사용해서 경계를 엄격히 유지하고, `controller`가 Domain 객체가 필요하면 Application Layer에서 DTO로 변환해서 전달합니다.

</details>

---

**Q2.** 멀티 모듈 프로젝트에서 통합 테스트(@SpringBootTest)는 어느 모듈에 위치해야 하는가?

<details>
<summary>해설 보기</summary>

**`bootstrap` 또는 별도 `integration-test` 모듈에 위치합니다.**

이유: `@SpringBootTest`는 전체 Spring Context를 로딩하므로 모든 모듈에 접근해야 합니다.

```kotlin
// bootstrap 모듈에 통합 테스트 위치
// bootstrap/src/test/java/.../PlaceOrderIntegrationTest.java
@SpringBootTest
@Testcontainers
class PlaceOrderIntegrationTest {
    @Autowired PlaceOrderUseCase useCase;  // 전체 Context
}

// 또는 별도 통합 테스트 모듈
// integration-test/build.gradle.kts
dependencies {
    testImplementation(project(":bootstrap"))
    testImplementation(project(":domain"))
    testImplementation(project(":application"))
    // 모든 모듈에 접근 가능
}
```

각 레이어 단위 테스트:
- domain 모듈: 순수 Java 단위 테스트 (`domain/src/test/`)
- application 모듈: InMemory Adapter + 단위 테스트 (`application/src/test/`)
- infrastructure 모듈: `@DataJpaTest` + TestContainers (`infrastructure/src/test/`)
- presentation 모듈: `@WebMvcTest` (`presentation/src/test/`)

</details>

---

**Q3.** 기능 기반 멀티 모듈에서 Order 모듈과 Payment 모듈이 서로를 참조해야 할 때 순환 의존성 문제는 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**순환 의존성은 아키텍처 문제를 드러내는 신호입니다.** Gradle은 순환 의존성을 빌드 시 에러로 막습니다.

해결 방법:

1. **이벤트 기반**: Order → `shared/event/OrderPlacedEvent` 발행, Payment가 구독
   ```kotlin
   // order 모듈이 shared/event에 이벤트 정의
   // payment 모듈이 이벤트를 구독 (order에 의존 없이)
   ```

2. **공유 모듈에 인터페이스**: `shared` 모듈에 `PaymentPort` 인터페이스 정의
   ```kotlin
   // shared/build.gradle.kts: 의존성 없음
   // order/build.gradle.kts: implementation(project(":shared"))
   // payment/build.gradle.kts: implementation(project(":shared"))
   // → order와 payment 모두 shared만 의존, 서로는 모름
   ```

3. **경계 재검토**: 순환이 생기면 두 기능의 경계가 잘못 설정된 신호 → BC 재설계

Gradle의 순환 의존성 감지:
```
> Could not resolve project :payment
  Circular dependency between projects: :order -> :payment -> :order
```
이 에러가 아키텍처 문제를 컴파일 시점에 알려주는 것입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Spring Boot 실전 패키지 구조](./02-spring-hexagonal-package.md)** | **[홈으로 🏠](../README.md)** | **[다음: ArchUnit으로 아키텍처 테스트 ➡️](./04-archunit-architecture-tests.md)**

</div>
