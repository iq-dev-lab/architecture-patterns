<div align="center">

# 🏛️ Architecture Patterns Deep Dive

**"레이어드 아키텍처를 쓰는 것과, 왜 의존성이 안으로만 향해야 하는지 아는 것은 다르다"**

<br/>

> *"Controller-Service-Repository 폴더를 만들었다 — 와 — 의존성이 역전될 때 무슨 일이 일어나는지, 도메인 레이어가 왜 어떤 프레임워크도 몰라야 하는지 아는 것의 차이를 만드는 레포"*

아키텍처는 폴더 구조가 아닙니다.  
**변경이 어디서 시작되어 어디로 전파되는가** — 그 결정의 집합입니다.

Hexagonal Architecture의 Port와 Adapter가 왜 인프라를 교체 가능하게 만드는지, Clean Architecture의 의존성 규칙이 왜 Entities에서 바깥으로 향하지 않는지, DDD Aggregate가 어느 레이어에 위치해야 하는지,  
**왜 이렇게 설계됐는가** 라는 질문으로 아키텍처 패턴의 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-17-007396?style=flat-square&logo=openjdk&logoColor=white)](https://openjdk.org/)
[![Spring](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://spring.io/projects/spring-boot)
[![ArchUnit](https://img.shields.io/badge/ArchUnit-1.x-FF6B35?style=flat-square)](https://www.archunit.org/)
[![Docs](https://img.shields.io/badge/Docs-39개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

아키텍처에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떤 구조를 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "Controller-Service-Repository 구조를 쓰세요" | 이 구조에서 의존성이 DB 방향으로 흐르는 이유, 도메인이 영속성에 오염되는 경로, 단위 테스트가 DB 없이 안 되는 원인 |
| "Hexagonal Architecture를 쓰면 됩니다" | Port가 내부와 외부의 계약(Contract)인 이유, Driving Port vs Driven Port의 차이, Adapter 교체가 실제로 무엇을 의미하는지 |
| "Clean Architecture는 4개 레이어입니다" | 의존성 규칙(Dependency Rule)이 왜 안쪽만 향해야 하는지, UseCase의 Input/Output Boundary가 DTO와 다른 이유, Spring에서 엄격하게 구현 시 생기는 현실적 타협 |
| "DDD와 아키텍처를 통합하세요" | Aggregate가 Hexagonal의 Domain Layer에 위치해야 하는 이유, Repository Port가 Aggregate 경계를 지키는 방법, Domain Event가 Driven Port를 통해 발행되는 구조 |
| "ArchUnit으로 아키텍처를 강제하세요" | 도메인 레이어가 Spring 어노테이션을 쓰지 못하도록 테스트 코드로 규칙 강제, CI에서 의존성 위반을 자동 감지하는 방법 |
| "점진적으로 리팩터링하면 됩니다" | Strangler Fig 패턴으로 레거시를 유지하면서 Hexagonal로 전환하는 경로, 하나의 기능부터 시작하는 실전 절차 |
| 이론 나열 | 실행 가능한 Spring Boot 코드 + ArchUnit 테스트 + Before/After 의존성 다이어그램 + Docker Compose 환경 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Fundamentals](https://img.shields.io/badge/🔹_Fundamentals-아키텍처의_본질-4A90D9?style=for-the-badge)](./architecture-fundamentals/01-what-is-architecture.md)
[![Layered](https://img.shields.io/badge/🔹_Layered-레이어드_아키텍처_원칙-4A90D9?style=for-the-badge)](./layered-architecture/01-layered-principles.md)
[![Hexagonal](https://img.shields.io/badge/🔹_Hexagonal-Port와_Adapter_핵심-4A90D9?style=for-the-badge)](./hexagonal-architecture/01-hexagonal-core-idea.md)
[![Clean](https://img.shields.io/badge/🔹_Clean-Uncle_Bob의_4원칙-4A90D9?style=for-the-badge)](./clean-architecture/01-clean-architecture-overview.md)
[![Comparison](https://img.shields.io/badge/🔹_Comparison-패턴_비교_분석-4A90D9?style=for-the-badge)](./architecture-comparison/01-layered-vs-hexagonal-vs-clean.md)
[![Package](https://img.shields.io/badge/🔹_Package-패키지_구조_원칙-4A90D9?style=for-the-badge)](./package-structure/01-package-structure-principles.md)
[![Refactoring](https://img.shields.io/badge/🔹_Refactoring-레거시_코드_분석-4A90D9?style=for-the-badge)](./refactoring-project/01-legacy-code-analysis.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: 아키텍처가 존재하는 이유

> **핵심 질문:** 아키텍처란 무엇인가? 의존성은 왜 변경 전파의 경로인가? 좋은 아키텍처는 어떻게 변경 비용을 낮추는가?

<details>
<summary><b>아키텍처의 본질부터 전통적 레이어드 아키텍처의 문제까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 아키텍처의 본질 — 아키텍처는 폴더 구조가 아니다](./architecture-fundamentals/01-what-is-architecture.md) | Uncle Bob의 "아키텍처는 결정을 미루는 예술", 좋은 아키텍처가 선택지를 열어두는 이유, 변경 비용을 낮추는 것이 아키텍처의 목적인 이유, 소프트웨어 엔트로피가 아키텍처 결정 없이 증가하는 과정 |
| [02. 변경 비용과 의존성 — 의존성이 변경 전파의 경로인 이유](./architecture-fundamentals/02-dependency-and-change-cost.md) | 의존하는 쪽이 의존받는 쪽 변경에 영향받는 원리, Ripple Effect(파급 효과)를 설계로 통제하는 방법, 의존성 그래프로 변경 범위를 예측하는 방법, 의존성 수가 줄어들수록 변경 비용이 낮아지는 이유 |
| [03. SOLID 원칙과 아키텍처 — SRP/OCP/DIP가 패턴의 이유](./architecture-fundamentals/03-solid-and-architecture.md) | SRP(단일 책임)가 레이어 분리의 이유, OCP(개방-폐쇄)가 Adapter 패턴의 이유, DIP(의존성 역전)가 Hexagonal/Clean Architecture의 핵심인 이유, 5원칙이 각 아키텍처 패턴에서 구체적으로 나타나는 지점 |
| [04. 전통적 레이어드 아키텍처의 문제 — 의존성이 DB 방향으로 흐르는 이유](./architecture-fundamentals/04-layered-architecture-problems.md) | Controller → Service → Repository 구조가 직관적이지만 의존성이 인프라 방향으로 흐르는 문제, 도메인이 영속성에 오염되는 이유, Spring Context 없이 단위 테스트가 어려워지는 원인, 의존성 역전으로 이 문제를 해결하는 방향 |
| [05. 아키텍처 선택 기준 — 언제 Hexagonal이고 언제 레이어드인가](./architecture-fundamentals/05-architecture-selection-criteria.md) | 도메인 복잡도(단순 CRUD vs 복잡한 비즈니스), 팀 규모, 외부 시스템 교체 가능성 요구, 테스트 자동화 수준에 따른 선택 가이드, 과잉 엔지니어링의 위험, 점진적 아키텍처 개선 전략 |

</details>

<br/>

### 🔹 Chapter 2: 레이어드 아키텍처 완전 분해

> **핵심 질문:** 레이어드 아키텍처의 레이어 간 의존성 규칙은 무엇인가? Fat Service 문제는 왜 생기고, DIP로 어떻게 개선하는가?

<details>
<summary><b>레이어드 원칙부터 DIP 기반 개선까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 레이어드 아키텍처의 원칙 — Presentation / Business / Data Access](./layered-architecture/01-layered-principles.md) | 각 레이어의 책임과 경계, 레이어 간 의존성이 항상 아래 방향이어야 하는 이유, 상위 레이어가 하위 레이어의 구체 클래스에 의존하지 않아야 하는 이유, 실무에서 레이어 경계를 지키지 않을 때 발생하는 일 |
| [02. 엄격한 레이어 vs 느슨한 레이어 — Strict vs Relaxed](./layered-architecture/02-strict-vs-relaxed-layers.md) | Strict Layer(인접 레이어만 호출 가능) vs Relaxed Layer(모든 하위 레이어 호출 가능)의 차이, 각각의 장단점, Relaxed 구조에서 레이어 의미가 흐려지는 문제, 실무에서의 타협점 |
| [03. 레이어드 아키텍처의 함정 — Fat Service와 DTO 침투](./layered-architecture/03-layered-architecture-traps.md) | Presentation이 DTO를 Domain까지 그대로 내려보내는 문제, Service 레이어에 모든 비즈니스 로직이 집중되는 Fat Service 문제, 레이어 간 순환 의존성이 생기는 경로, @Transactional 남용이 Service를 비대하게 만드는 이유 |
| [04. 레이어드 아키텍처에서 테스트 — Spring Context가 항상 필요한 이유](./layered-architecture/04-testing-in-layered.md) | Service가 JpaRepository에 직접 의존할 때 단위 테스트에 DB가 필요해지는 이유, @SpringBootTest vs @DataJpaTest vs Mockito의 트레이드오프, Mocking의 한계(내부 구현에 결합되는 문제), 아키텍처 변경이 테스트 속도에 미치는 영향 |
| [05. 레이어드 아키텍처 개선 — DIP로 의존성 역전](./layered-architecture/05-improving-with-dip.md) | Repository 인터페이스를 Business 레이어로 올리는 이유, DIP 적용 전후 의존성 다이어그램 비교, 인터페이스 도입 후 InMemory 구현체로 빠른 단위 테스트가 가능해지는 이유, Hexagonal Architecture로 자연스럽게 연결되는 방향 |

</details>

<br/>

### 🔹 Chapter 3: Hexagonal Architecture (Ports & Adapters)

> **핵심 질문:** Port는 무엇이고 Adapter는 무엇인가? 왜 모든 의존성이 도메인을 향하는가? DDD Aggregate는 어느 위치에 놓이는가?

<details>
<summary><b>Hexagonal 핵심 아이디어부터 DDD 통합까지 (8개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Hexagonal Architecture의 핵심 아이디어 — 내부와 외부의 분리](./hexagonal-architecture/01-hexagonal-core-idea.md) | Alistair Cockburn의 원래 의도, 애플리케이션을 "내부(도메인)"와 "외부(인프라)"로 분리하는 이유, 포트가 안과 밖의 계약(Contract)인 이유, 레이어드 아키텍처와의 핵심 차이점 |
| [02. 주도하는 포트 vs 주도받는 포트 — Driving vs Driven](./hexagonal-architecture/02-driving-vs-driven-ports.md) | Driving Port(사용자 → 애플리케이션, UseCase 인터페이스)가 애플리케이션의 진입점인 이유, Driven Port(애플리케이션 → 인프라, Repository/MessagePublisher 인터페이스)가 도메인 레이어에 위치해야 하는 이유, 포트가 인터페이스인 이유 |
| [03. 어댑터의 역할 — Driving Adapter와 Driven Adapter](./hexagonal-architecture/03-adapters-explained.md) | Driving Adapter(HTTP Controller, CLI, Test가 UseCase를 호출하는 방식), Driven Adapter(JpaRepository, KafkaPublisher, InMemoryRepository가 Port를 구현하는 방식), 어댑터 교체 가능성의 실제 의미와 비용 |
| [04. 의존성 방향 완전 분석 — 왜 모든 화살표가 도메인을 향하는가](./hexagonal-architecture/04-dependency-direction-analysis.md) | 모든 의존성이 도메인 방향(안)으로 향하는 이유, 인프라(JPA, Kafka, HTTP)가 도메인에 의존하는 이유, DIP가 의존성 방향을 역전시키는 방법, 레이어드 아키텍처와 Hexagonal의 의존성 다이어그램 비교 |
| [05. Spring으로 Hexagonal 구현 — 패키지 구조와 매핑](./hexagonal-architecture/05-hexagonal-with-spring.md) | 패키지 구조(application/domain/infrastructure) 설계, UseCase 인터페이스 정의, @Service가 UseCase를 구현하는 방식, @Repository가 Port를 구현하는 방식, @Controller가 UseCase를 호출하는 방식, JPA Entity와 Domain Entity 분리 필요성 |
| [06. 테스트 용이성 향상 — InMemory Adapter로 빠른 단위 테스트](./hexagonal-architecture/06-testability-with-hexagonal.md) | Port 인터페이스 덕분에 InMemory Adapter로 빠른 단위 테스트가 가능한 이유, Spring Context 없이 도메인 로직을 테스트하는 방법, 어댑터별 독립 테스트 전략, @SpringBootTest 의존 비율이 줄어드는 과정 |
| [07. Hexagonal의 실제 비용 — 복잡도와 매핑 비용](./hexagonal-architecture/07-hexagonal-real-costs.md) | 인터페이스 증가로 인한 코드량, 단순 CRUD에 적용 시 과잉 복잡도 문제, Domain Entity와 JPA Entity 분리 시 발생하는 매핑 비용, 팀 학습 비용, Hexagonal이 적합한 도메인 복잡도 기준 |
| [08. DDD와 Hexagonal 통합 — Aggregate, Repository Port, Domain Event](./hexagonal-architecture/08-ddd-hexagonal-integration.md) | Aggregate가 도메인 내부에 위치해야 하는 이유, Repository Port가 Aggregate 경계를 지키는 방법, Domain Event가 Driven Port(MessagePublisher)를 통해 발행되는 구조, Bounded Context 경계와 Hexagonal 포트의 관계 |

</details>

<br/>

### 🔹 Chapter 4: Clean Architecture

> **핵심 질문:** Uncle Bob의 4개 동심원은 무엇이고 의존성 규칙이란 무엇인가? Hexagonal과 어떤 점이 같고 다른가?

<details>
<summary><b>Uncle Bob의 4원칙부터 Spring 실전 구현까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Uncle Bob의 Clean Architecture 개요 — 4개 동심원과 의존성 규칙](./clean-architecture/01-clean-architecture-overview.md) | Entities / Use Cases / Interface Adapters / Frameworks & Drivers 4개 레이어의 책임, 의존성 규칙(Dependency Rule)이 안쪽만 향해야 하는 이유, Hexagonal Architecture와의 공통점과 차이점, Clean Architecture가 해결하려는 구체적 문제 |
| [02. Entities 레이어 — 비즈니스 규칙의 핵심](./clean-architecture/02-entities-layer.md) | Entities가 어떤 외부 변경에도 영향받지 않아야 하는 이유, 기업 수준 비즈니스 규칙과 애플리케이션 수준 비즈니스 규칙의 차이, DDD Entity / Value Object / Aggregate와의 매핑, Entities 레이어에서 Spring 어노테이션이 없어야 하는 이유 |
| [03. Use Cases 레이어 — 애플리케이션별 비즈니스 규칙](./clean-architecture/03-use-cases-layer.md) | UseCase가 애플리케이션 고유의 비즈니스 규칙을 담는 이유, 입출력 포트(Input/Output Boundary)가 인터페이스인 이유, Request Model / Response Model이 외부 DTO와 분리되어야 하는 이유, Command와 Query 분리(CQRS)와의 연결 |
| [04. Interface Adapters 레이어 — 변환의 레이어](./clean-architecture/04-interface-adapters-layer.md) | Controller가 HTTP Request를 Use Case Input으로 변환하는 역할, Presenter가 Use Case Output을 ViewModel로 변환하는 역할, Gateway가 Repository 인터페이스를 구현하는 역할, 이 레이어에서 프레임워크(Spring MVC)가 허용되는 이유 |
| [05. Frameworks & Drivers 레이어 — 가장 바깥에 위치해야 하는 이유](./clean-architecture/05-frameworks-and-drivers.md) | JPA, Kafka, HTTP, Spring이 가장 바깥에 위치하는 이유, 프레임워크가 교체 가능해야 하는 이유(현실적 타당성 포함), main 함수가 이 레이어에 위치하는 이유, Frameworks & Drivers에서 안쪽으로 의존성이 향하는 방법 |
| [06. Clean Architecture 실전 — Spring Boot에서의 현실적 구현](./clean-architecture/06-clean-architecture-in-practice.md) | Spring Boot에서 모든 레이어를 엄격히 분리한 패키지 구조 예시, Entity에 JPA 어노테이션 허용 여부의 현실적 타협, UseCase 레이어에서 @Transactional 허용 여부 판단 기준, 팀 합의가 필요한 결정들과 ADR 작성 예시 |

</details>

<br/>

### 🔹 Chapter 5: 아키텍처 패턴 비교와 선택

> **핵심 질문:** Layered / Hexagonal / Clean은 어떤 기준으로 비교하는가? 우리 팀 상황에는 어느 패턴이 맞는가?

<details>
<summary><b>3가지 패턴 비교부터 MSA와의 통합까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Layered vs Hexagonal vs Clean 비교 — 의존성·테스트·복잡도](./architecture-comparison/01-layered-vs-hexagonal-vs-clean.md) | 의존성 방향, 테스트 용이성(Spring Context 필요 비율), 코드 복잡도, 팀 학습 비용, 도메인 순수성 관점에서의 비교표, 각 패턴이 해결하는 문제와 해결하지 못하는 문제 |
| [02. 패턴 선택 기준 — 도메인 복잡도와 팀 상황에 따른 가이드](./architecture-comparison/02-pattern-selection-guide.md) | 단순 CRUD vs 복잡한 비즈니스 규칙에 따른 선택, 팀 규모와 학습 곡선 고려, 외부 시스템 교체 가능성 요구 여부, 테스트 자동화 성숙도에 따른 선택 가이드, 과잉 엔지니어링의 징후 |
| [03. 혼합 아키텍처 — Layered 기본 + 핵심 도메인에만 Hexagonal 적용](./architecture-comparison/03-mixed-architecture.md) | 레이어드를 기본으로 하되 핵심 도메인에만 Hexagonal 적용하는 전략, 점진적 리팩터링 경로(Strangler Fig 방식), 혼합 아키텍처에서 일관성을 유지하는 규칙, 혼합 구조를 ArchUnit으로 강제하는 방법 |
| [04. 아키텍처 결정 기록(ADR) — 결정을 문서화하는 방법](./architecture-comparison/04-architecture-decision-records.md) | ADR(Architecture Decision Record)의 필요성, ADR 템플릿(Context / Decision / Consequences), 팀이 아키텍처 결정에 합의하는 프로세스, 실제 ADR 예시("왜 Repository 인터페이스를 도메인 레이어에 놓는가") |
| [05. MSA와 아키텍처 패턴 — Bounded Context 안에서의 내부 아키텍처](./architecture-comparison/05-msa-and-architecture-patterns.md) | 서비스 내부에 적용하는 아키텍처 패턴, 서비스 크기(Nano/Micro/Modular Monolith)에 따른 아키텍처 선택, DDD Bounded Context와 아키텍처 패턴의 조합, 서비스 간 통신(API/Event)이 Adapter로 추상화되는 방식 |

</details>

<br/>

### 🔹 Chapter 6: 패키지 구조와 모듈화

> **핵심 질문:** 레이어 기반 vs 기능 기반 패키지 구조는 어떤 차이인가? Gradle 멀티 모듈과 ArchUnit으로 의존성을 강제하는 방법은?

<details>
<summary><b>패키지 구조 원칙부터 ArchUnit 아키텍처 테스트까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 패키지 구조의 원칙 — 레이어 기반 vs 기능 기반 vs 컴포넌트 기반](./package-structure/01-package-structure-principles.md) | 레이어 기반(presentation/service/repository) vs 기능 기반(order/payment/user) vs 컴포넌트 기반 구조의 차이, 각각에서 기능 추가 시 변경되는 파일 범위, 응집도와 결합도 관점에서의 비교, 규모에 따른 적합한 구조 |
| [02. Spring Boot 실전 패키지 구조 — Hexagonal 기반 권장 구조](./package-structure/02-spring-hexagonal-package.md) | Hexagonal Architecture 기반 패키지 구조 상세 예시(application/domain/infrastructure 내부 세부 패키지), UseCase / Port / Adapter / Domain 각 파일의 위치 결정 기준, 실제 Order 도메인 예시로 전체 구조 설명 |
| [03. 멀티 모듈 프로젝트 — Gradle로 의존성을 컴파일 시점에 강제](./package-structure/03-multi-module-project.md) | Gradle 멀티 모듈로 레이어 간 의존성을 컴파일 시점에 강제하는 방법, domain 모듈이 infrastructure 모듈을 참조할 수 없도록 설정, 멀티 모듈 구성(domain / application / infrastructure / presentation), 빌드 속도와 모듈 경계 유지의 트레이드오프 |
| [04. ArchUnit으로 아키텍처 테스트 — 의존성 규칙을 코드로 강제](./package-structure/04-archunit-architecture-tests.md) | ArchUnit 기본 문법과 설정, "도메인 레이어는 Spring 어노테이션을 사용하지 않는다" 규칙 코드, "infrastructure는 domain에만 의존한다" 규칙 코드, CI 파이프라인에서 아키텍처 규칙 자동 검증, 규칙 위반 발생 시 에러 메시지 해석 |
| [05. 코드 가이드라인 수립 — 팀이 아키텍처 원칙을 유지하는 방법](./package-structure/05-code-guidelines.md) | 신규 기능 추가 시 어느 레이어에 코드를 작성할지 결정 트리, 코드 리뷰에서 아키텍처 원칙을 체크하는 기준, 온보딩 시 아키텍처 원칙을 전달하는 방법, 규칙이 너무 엄격해서 생산성을 낮출 때 완화하는 기준 |

</details>

<br/>

### 🔹 Chapter 7: 실전 프로젝트 — 아키텍처 리팩터링

> **핵심 질문:** 레거시 레이어드 아키텍처를 Hexagonal로 전환하는 실전 경로는 무엇인가? 리팩터링 전후 테스트 속도와 변경 범위는 어떻게 달라지는가?

<details>
<summary><b>레거시 분석부터 전후 비교까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 레거시 코드 분석 — 의존성 그래프로 문제 시각화](./refactoring-project/01-legacy-code-analysis.md) | Controller에 비즈니스 로직, Service에 SQL 직접 호출, Entity에 비즈니스 로직 혼재하는 실제 코드 예시, 의존성 그래프로 문제 구조 시각화, 변경 시 파급 범위 측정, 테스트 커버리지 측정 후 리팩터링 시작 기준 판단 |
| [02. 점진적 리팩터링 전략 — Strangler Fig로 레거시 유지하면서 전환](./refactoring-project/02-strangler-fig-strategy.md) | Strangler Fig 패턴으로 레거시를 유지하면서 새 아키텍처를 도입하는 방법, 하나의 기능(UseCase)부터 Hexagonal로 전환하는 절차, 테스트 커버리지 확보 후 리팩터링하는 이유, 기능 플래그로 전환 중 롤백하는 방법 |
| [03. 도메인 레이어 분리 — Service에서 도메인 로직 추출](./refactoring-project/03-extracting-domain-layer.md) | Service에서 도메인 로직을 추출하여 Domain Object(Entity, Value Object)로 이동하는 과정, UseCase 인터페이스 도입으로 진입점을 명확히 하는 방법, 도메인 레이어에서 Spring 의존성 제거 과정, 각 단계별 커밋 전략 |
| [04. 인프라 레이어 분리 — Repository와 Kafka를 Port로 추상화](./refactoring-project/04-extracting-infrastructure-layer.md) | Repository 인터페이스를 도메인으로 이동시키는 과정, JPA Repository를 Driven Adapter로 이동하는 방법, Kafka 발행을 Port(MessagePublisher)로 추상화하는 방법, 분리 후 InMemory Adapter로 단위 테스트 작성 |
| [05. 전후 비교 — 의존성·테스트 속도·변경 범위](./refactoring-project/05-before-after-comparison.md) | 리팩터링 전후 의존성 다이어그램 비교, @SpringBootTest 필요 테스트 비율 변화(전: 70% / 후: 20%), 새 기능 추가 시 변경 파일 범위 비교, 인프라 교체(JPA → MongoDB) 시 변경 범위 비교, 팀 소감과 실제 적용 시 주의사항 |

</details>

---

## 🔑 핵심 코드 스니펫 치트시트

```java
// ❌ Before: 의존성이 인프라 방향으로 (레이어드 아키텍처)
@Service
class OrderService {
    @Autowired OrderJpaRepository repo;  // 인프라에 직접 의존
    @Autowired KafkaTemplate kafka;      // 인프라에 직접 의존
    // → 단위 테스트에 DB, Kafka 필요
}

// ✅ After: 의존성이 도메인 방향으로 (Hexagonal Architecture)
// === Domain Layer ===
public class Order { /* 순수 Java, 어노테이션 없음 */ }
public interface OrderRepository { /* Driven Port */ }
public interface OrderEventPublisher { /* Driven Port */ }

// === Application Layer ===
public interface PlaceOrderUseCase { /* Driving Port (Input Boundary) */ }
@Service
class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderRepository repo;         // Port에만 의존
    private final OrderEventPublisher publisher; // Port에만 의존
    // JPA도 Kafka도 전혀 모름
}

// === Infrastructure Layer (Adapter) ===
@Repository
class JpaOrderRepository implements OrderRepository { /* Driven Adapter */ }
@Component
class KafkaOrderEventPublisher implements OrderEventPublisher { /* Driven Adapter */ }

// === 단위 테스트 (Spring Context 없음) ===
class PlaceOrderServiceTest {
    OrderRepository repo = new InMemoryOrderRepository();       // Adapter 교체
    OrderEventPublisher pub = new InMemoryOrderEventPublisher();
    PlaceOrderService service = new PlaceOrderService(repo, pub);
    // 수십 ms 안에 실행되는 빠른 테스트
}
```

```java
// ArchUnit으로 아키텍처 규칙 강제
@AnalyzeClasses(packages = "com.example")
class ArchitectureTest {

    @ArchTest
    static final ArchRule domainShouldNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..")
            .because("ADR 참고: Domain은 인프라를 몰라야 합니다");

    @ArchTest
    static final ArchRule domainShouldNotUseSpringAnnotations =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().beAnnotatedWith(Component.class)
            .orShould().beAnnotatedWith(Service.class)
            .because("ADR 참고: Domain 레이어는 Spring 어노테이션을 사용하지 않습니다");
}
```

```
의존성 방향 핵심 요약:

=== 레이어드 (문제) ===
Controller → Service → JpaRepository → DB
                 ↓
             KafkaTemplate → Kafka
도메인(Service)이 인프라(JPA, Kafka)를 알고 있음

=== Hexagonal (해결) ===
모든 화살표가 Domain(중앙)을 향함
Controller → UseCase(Interface) ← PlaceOrderService
                                       ↓ 의존
                            OrderRepository(Interface)
                                 ↑ 구현
                            JpaOrderRepository → DB
JPA 변경 → JpaOrderRepository만 변경
도메인 테스트에 DB 불필요
```

---

## 🚧 실험 환경

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: architecture_demo
    ports:
      - "3306:3306"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      # ※ localhost 사용 시 호스트 머신 → Kafka 접속만 가능
      # 컨테이너 간 통신이 필요하면 PLAINTEXT://kafka:9092 로 변경하세요

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 아키텍처 결정이 중요한가** | 아키텍처 원칙 없이 짤 때 실무에서 마주치는 문제 |
| 😱 **흔한 실수** | Before — 원칙 없이 짜는 코드와 그 결과 (의존성 다이어그램 포함) |
| ✨ **올바른 접근** | After — 아키텍처 원칙 적용 코드와 패키지 구조 |
| 🔬 **내부 원리** | 왜 이 구조가 변경 비용을 낮추는가 — 의존성 분석과 Ripple Effect 비교 |
| 💻 **실전 코드** | Spring Boot 기반, 패키지 구조 포함, ArchUnit 테스트 포함 |
| 📊 **패턴 비교** | Layered vs Hexagonal vs Clean, 아키텍처별 의존성 다이어그램 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "Service가 JpaRepository를 직접 쓰는 게 왜 문제인지 모른다" — 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch1-04  전통적 레이어드 아키텍처의 문제 → 의존성이 DB 방향으로 흐르는 이유
Day 2  Ch2-04  레이어드에서 테스트 → Spring Context가 항상 필요한 이유
       Ch2-05  DIP로 의존성 역전 → Repository 인터페이스를 도메인으로 올리기
Day 3  Ch3-01  Hexagonal 핵심 아이디어 → Port와 Adapter의 의미
       Ch3-06  테스트 용이성 향상 → InMemory Adapter로 단위 테스트
```

</details>

<details>
<summary><b>🟡 "Hexagonal Architecture를 제대로 이해하고 Spring에서 구현하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  아키텍처의 본질 → 변경 비용과 의존성
       Ch1-03  SOLID 원칙 → DIP가 Hexagonal의 핵심인 이유
Day 2  Ch3-01  Hexagonal 핵심 아이디어
       Ch3-02  Driving vs Driven Port 구분
Day 3  Ch3-03  어댑터의 역할
       Ch3-04  의존성 방향 완전 분석
Day 4  Ch3-05  Spring으로 Hexagonal 구현 (패키지 구조)
       Ch3-06  테스트 용이성 향상
Day 5  Ch4-01  Clean Architecture 개요 → Hexagonal과 비교
       Ch4-03  Use Cases 레이어 → Input/Output Boundary
Day 6  Ch6-02  Spring Boot 실전 패키지 구조
       Ch6-04  ArchUnit으로 아키텍처 테스트
Day 7  Ch7-05  전후 비교 → 리팩터링 결과 확인
```

</details>

<details>
<summary><b>🔴 "레거시 코드를 Hexagonal로 전환하는 실전 경로가 필요하다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — 아키텍처가 존재하는 이유
        → 의존성 그래프 직접 그려보기, 변경 파급 범위 측정

2주차  Chapter 2 전체 — 레이어드 아키텍처 완전 분해
        → 현재 코드에서 레이어 경계 위반 찾기, DIP 적용해보기

3주차  Chapter 3 전체 — Hexagonal Architecture
        → 하나의 UseCase를 Hexagonal로 직접 구현, InMemory Adapter 테스트 작성

4주차  Chapter 4 전체 — Clean Architecture
        → Hexagonal vs Clean 구조 비교 직접 구현, ADR 작성해보기

5주차  Chapter 5 전체 — 패턴 비교와 선택
        → 팀 상황에 맞는 아키텍처 선택 기준 수립, ADR 팀 리뷰

6주차  Chapter 6 전체 — 패키지 구조와 모듈화
        → ArchUnit 테스트 CI에 추가, 멀티 모듈 설정 실습

7주차  Chapter 7 전체 — 실전 리팩터링
        → 실제 레거시 코드에 Strangler Fig 패턴 적용, 전후 테스트 속도 측정
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | DI 컨테이너, AOP, Bean 생명주기 | Ch3-05(Spring으로 Hexagonal 구현), Ch6-02(패키지 구조와 Bean 등록) |
| [ddd-deep-dive](https://github.com/dev-book-lab/ddd-deep-dive) | Domain Model, Aggregate, Repository | Ch3-08(DDD+Hexagonal 통합), Ch4-02(Entities와 DDD Entity 매핑) |
| [unit-testing](https://github.com/dev-book-lab/unit-testing) | 단위 테스트, 테스트 대역, Mock | Ch3-06(InMemory Adapter 테스트), Ch7-02(리팩터링 전 테스트 커버리지 확보) |
| [redis-deep-dive](https://github.com/dev-book-lab/redis-deep-dive) | Redis 내부 구조, 캐싱 패턴 | Ch3-03(Driven Adapter 패턴 — 외부 시스템 추상화), Ch5-05(MSA 서비스 간 이벤트 통신) |

> 💡 이 레포는 **아키텍처 패턴의 원리와 Spring 실전 구현**에 집중합니다. DDD 개념(Aggregate, Bounded Context)은 ddd-deep-dive에서, Spring DI/AOP의 기술적 구현은 spring-core-deep-dive에서 선행 학습하면 이 레포의 내용이 더 깊이 연결됩니다.

---

## 🙏 Reference

- [Clean Architecture — Robert C. Martin (Uncle Bob)](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164)
- [Get Your Hands Dirty on Clean Architecture — Tom Hombergs](https://leanpub.com/get-your-hands-dirty-on-clean-architecture) — Spring 실전 구현 최고 참고서
- [Hexagonal Architecture 원문 — Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [PresentationDomainDataLayering — Martin Fowler](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
- [Making Architecture Matter — Martin Fowler](https://youtu.be/DngAZyWMGR0)
- [ArchUnit 공식 문서](https://www.archunit.org/)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/) — 분산 시스템에서의 아키텍처 결정

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"아키텍처는 폴더 구조가 아니다 — 변경이 전파되는 방향을 결정하는 것이다"*

</div>
