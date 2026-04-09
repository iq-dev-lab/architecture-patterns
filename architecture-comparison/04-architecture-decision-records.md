# 아키텍처 결정 기록(ADR) — 결정을 문서화하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ADR(Architecture Decision Record)이 왜 필요한가? "코드가 문서다"로는 부족한 이유는?
- ADR 템플릿(Context / Decision / Consequences)의 각 섹션이 담아야 할 내용은?
- 팀이 아키텍처 결정에 합의하는 프로세스는 어떻게 구성하는가?
- "왜 Repository 인터페이스를 Domain 레이어에 놓는가"를 ADR로 작성하면 어떻게 되는가?
- ADR을 언제 작성하고, 어디에 저장하고, 어떻게 관리하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

아키텍처 결정은 가장 비싼 결정이다. 한 번 내리면 수개월 동안 팀 전체가 그 결정의 영향을 받는다. 그런데 대부분의 팀은 이 결정을 회의실 화이트보드나 Slack 메시지로 남기고, 6개월 후 "왜 이렇게 됐더라?"를 기억하지 못한다.

ADR은 이 문제를 해결한다. 결정 자체가 아니라 **왜 그 결정을 내렸는가**, **어떤 대안을 검토했는가**, **어떤 트레이드오프를 수용했는가**를 기록한다. 이것이 신규 팀원 온보딩, 규칙 위반 방지, 의사결정 재검토 모두에 활용된다.

---

## 😱 흔한 실수 (Before — 결정이 기록되지 않을 때)

```
기록 없는 아키텍처 결정의 결과:

6개월 후 신규 팀원:
  "왜 OrderRepository 인터페이스가 domain 패키지에 있어요?
   infrastructure에 있어야 하지 않나요?"
  → 당시 결정한 팀원이 없거나 이유를 기억 못함
  → 신규 팀원이 "이해가 안 가서" infrastructure에 새로운 인터페이스 생성
  → 두 스타일 혼재, ArchUnit 규칙 위반

1년 후 규칙 위반:
  "왜 이 코드가 ArchUnit 테스트를 실패시키지?"
  → "규칙이 왜 있는지 모르니까 예외로 처리하면 되지 않나요?"
  → 이유를 모르면 규칙을 쉽게 우회

팀 합의 없는 결정:
  개발자 A: "@Transactional은 UseCase에 붙여야 해"
  개발자 B: "아니, Repository에 붙여야 해"
  개발자 C: "Service에 붙이는 게 맞아"
  → 셋 다 다른 방식으로 코딩
  → 코드베이스에 3가지 스타일 혼재

ADR이 있었다면:
  "ADR-002: @Transactional은 UseCase 레이어에 붙인다"
  → 이유, 대안, 트레이드오프가 명시됨
  → 신규 팀원도 문서를 보고 이해
  → 위반 시 "ADR-002를 보세요"로 대화 시작
```

---

## ✨ 올바른 접근 (After — ADR로 결정을 관리할 때)

```
ADR의 역할:

신규 팀원 온보딩:
  "왜 우리 코드가 이렇게 구성됐는가"를 ADR 디렉토리에서 읽음
  "아, ADR-001에 그 이유가 있네요" → 질문 시간 절감

규칙 위반 방지:
  ArchUnit 규칙이 있고 ADR로 이유가 문서화됨
  → 위반 시 단순히 "ArchUnit이 싫어해"가 아니라
  → "ADR-003의 이유로 이 규칙이 있습니다"

결정 재검토:
  상황이 바뀌면 ADR의 상태를 DEPRECATED로 변경
  새 ADR을 작성해서 이전 결정을 왜 바꾸는지 기록
  → 결정의 이력이 남음

팀 합의:
  ADR 초안 작성 → 팀 리뷰 (PR 형태) → 승인 → 병합
  → 팀이 결정에 참여하고 동의한 기록이 남음
```

---

## 🔬 내부 원리 — ADR의 구조와 프로세스

### 1. ADR 템플릿 — 핵심 섹션

```markdown
# ADR-{번호}: {결정 제목}

## 상태
[제안됨 | 승인됨 | 폐기됨 | 대체됨 by ADR-{번호}]
승인 날짜: YYYY-MM-DD
결정자: {팀 또는 담당자}

## 맥락 (Context)
왜 이 결정이 필요했는가?
  - 어떤 문제를 해결하려고 했는가?
  - 제약 조건은 무엇이었는가?
  - 현재 상황의 어떤 점이 문제였는가?

## 결정 (Decision)
무엇을 결정했는가? (능동태, 직접적으로)
  "우리는 X를 선택한다"
  "Y를 하지 않는다"

왜 이 결정을 내렸는가?
  - 주요 이유
  - 채택한 트레이드오프

## 검토한 대안 (Alternatives Considered)
| 대안 | 장점 | 단점 | 선택하지 않은 이유 |
|------|------|------|-------------------|
| 대안 A | ... | ... | ... |
| 대안 B | ... | ... | ... |

## 결과 (Consequences)

긍정적 결과:
  - 이 결정으로 얻는 것

부정적 결과 (수용된 트레이드오프):
  - 이 결정으로 포기하는 것
  - 추가로 관리해야 하는 것

## 관련 문서
  - 관련 ADR: ADR-{번호}
  - 참고 자료: 링크
```

### 2. 실제 ADR 예시 — Repository 인터페이스 위치 결정

```markdown
# ADR-001: Repository 인터페이스를 Domain 레이어에 위치

## 상태
승인됨 (2024-04-01, 팀 전체 합의)

## 맥락

현재 OrderService가 OrderJpaRepository(Spring Data JPA 인터페이스)를
직접 의존하고 있습니다. 이 구조에서는:

1. OrderService 단위 테스트에 JPA 컨텍스트가 필요하여 @SpringBootTest를
   사용해야 합니다. 현재 단위 테스트 100개 기준 CI 실행 시간이 43분입니다.

2. JPA를 MongoDB로 교체할 경우 OrderService 코드 수정이 필요합니다.
   현재 결제 DB를 MongoDB로 전환하는 계획이 Q3에 예정되어 있습니다.

3. Repository 인터페이스의 위치에 대해 팀 내 기준이 없어 일부는
   repository/ 패키지에, 일부는 infrastructure/ 패키지에 두고 있습니다.

두 가지 옵션을 검토했습니다:
- 옵션 A: 인터페이스를 infrastructure/ 패키지에 위치 (현재 방식의 정규화)
- 옵션 B: 인터페이스를 domain/ 패키지에 위치 (DIP 완전 적용)

## 결정

Repository 인터페이스(OrderRepository, PaymentRepository 등)를
domain/port/out/ 패키지에 위치시킨다.

JPA 구현체(JpaOrderRepository)는 infrastructure/persistence/ 패키지에 위치한다.

이유:
  DIP(의존성 역전 원칙) 완전 적용:
    OrderService(domain)가 OrderRepository(domain)에 의존
    JpaOrderRepository(infrastructure)가 OrderRepository(domain)를 구현
    = infrastructure가 domain을 의존 (올바른 방향)

  테스트 속도 개선:
    InMemoryOrderRepository를 OrderRepository 인터페이스로 구현하여
    OrderService 단위 테스트에서 JPA 없이 실행 가능
    목표: @SpringBootTest 비율 80% → 10%, CI 시간 43분 → 8분

## 검토한 대안

| 대안 | 장점 | 단점 | 선택하지 않은 이유 |
|------|------|------|-------------------|
| 인터페이스를 infrastructure에 | 기존 Spring 관행과 일치, 학습 비용 낮음 | OrderService(domain)가 infrastructure에 의존 → 의존성 규칙 위반, JPA 변경 시 domain 영향 | DIP 미적용으로 MongoDB 전환 시 domain 코드 수정 필요 |
| 인터페이스를 domain에 (채택) | DIP 완전 적용, 단위 테스트 가능 | infrastructure 관행에서 벗어남, 팀 학습 필요 | Q3 MongoDB 전환 계획 감안 시 장기 이익이 단기 비용을 상회 |

## 결과

긍정적 결과:
  - OrderService 단위 테스트를 InMemory 구현체로 실행 가능
  - CI 실행 시간 43분 → 목표 8분 (2주 후 측정 예정)
  - Q3 MongoDB 전환 시 JpaOrderRepository만 MongoOrderRepository로 교체
    (OrderService 코드 변경 없음)

부정적 결과 (수용된 트레이드오프):
  - 인터페이스가 domain/port/out/ 패키지에 있어 초기에 어색함
  - InMemoryOrderRepository 작성 및 유지 필요 (초기 1회 작업)
  - 신규 팀원 온보딩 시 "왜 Repository가 domain에?" 설명 필요
    → 이 ADR을 참고하도록 안내

## 이행 계획
1주 차: OrderRepository 인터페이스 domain/port/out/ 로 이동 + ArchUnit 규칙 추가
2주 차: JpaOrderRepository 분리 + InMemoryOrderRepository 작성
3주 차: OrderService 단위 테스트로 교체 + CI 시간 측정

## 관련 문서
  - ADR-002: @Transactional 위치 결정
  - ArchUnit 규칙: src/test/java/.../ArchitectureTest.java
  - 참고: Hexagonal Architecture 가이드 (ARCHITECTURE.md)
```

### 3. ADR 프로세스 — 팀 합의 방법

```
ADR 작성 → 리뷰 → 승인 → 구현 프로세스:

Step 1: 문제 인식 (1-2일)
  "이 결정을 내려야 한다"를 팀에 공유
  채널: Slack, 이슈 트래커, 회의 아젠다
  형식: "OrderRepository 위치 결정 필요: domain vs infrastructure"

Step 2: ADR 초안 작성 (2-4시간)
  담당자(또는 지원자)가 ADR 초안 작성
  맥락, 옵션들, 초기 의견 포함
  저장: docs/adr/YYYY-MM-DD-{제목}.md

Step 3: 팀 리뷰 (2-5일)
  GitHub PR로 ADR 파일 제출
  팀원 댓글로 대안 의견, 추가 고려사항 제시
  반드시 모든 의견을 ADR에 반영 (검토한 대안 섹션)

Step 4: 동기 합의 (30-60분)
  PR 댓글만으로 결론 안 날 때 회의 소집
  회의 결과를 ADR에 반영
  참석자 목록을 ADR의 결정자 필드에 기록

Step 5: 승인 및 병합
  팀 과반수 또는 아키텍처 리드 승인
  상태를 "승인됨"으로 변경
  날짜와 결정자 기록

Step 6: 구현
  ADR에 이행 계획 추가
  ArchUnit 규칙 변경이 필요하면 동시 PR

ADR 재검토 (상황 변화 시):
  새 ADR 작성: "ADR-007: ADR-001 대체 — Repository 인터페이스를 application으로 이동"
  기존 ADR 상태를 "대체됨 by ADR-007"로 변경
  기존 ADR은 삭제하지 않음 (결정의 이력 보존)
```

### 4. ADR 저장소 구조와 관리

```
docs/adr/ 디렉토리 구조:

docs/
└── adr/
    ├── README.md                          ← ADR 목록 및 사용 가이드
    ├── 0001-use-hexagonal-architecture.md
    ├── 0002-repository-in-domain-layer.md
    ├── 0003-transactional-in-usecase.md
    ├── 0004-entity-jpa-separation.md
    ├── 0005-testing-strategy.md
    └── 0006-archunit-enforcement.md

README.md (ADR 인덱스):
  | 번호 | 제목 | 상태 | 날짜 |
  |------|------|------|------|
  | 0001 | Hexagonal Architecture 채택 | 승인됨 | 2024-01-15 |
  | 0002 | Repository 인터페이스를 Domain에 | 승인됨 | 2024-02-01 |
  | 0003 | @Transactional 위치 | 승인됨 | 2024-02-01 |
  | 0004 | Entity/JPA Entity 분리 | 폐기됨 | 2024-03-01 |
  | 0005 | 테스트 전략 (70/20/5 피라미드) | 승인됨 | 2024-03-15 |

네이밍 규칙:
  {4자리 번호}-{동사}-{명사}.md
  예: 0001-adopt-hexagonal-architecture.md
      0002-place-repository-interface-in-domain.md

번호 관리:
  순차적으로 부여, 취소하지 않음
  (폐기돼도 번호는 유지)
```

### 5. ADR이 ArchUnit과 연결되는 방법

```java
// ADR-002의 결정을 ArchUnit으로 자동 강제

@AnalyzeClasses(packages = "com.example")
public class Adr002RepositoryInDomainTest {

    /**
     * ADR-002: Repository 인터페이스를 Domain 레이어에 위치
     * 
     * 이유: DIP 완전 적용, 단위 테스트 가능, MongoDB 전환 대비
     * 링크: docs/adr/0002-repository-in-domain-layer.md
     */
    @Test
    @DisplayName("ADR-002: Repository 인터페이스는 domain/port/out 패키지에 위치해야 함")
    void repository_interfaces_must_reside_in_domain_port_out() {
        classes()
            .that().haveNameMatching(".*Repository")
            .and().areInterfaces()
            .should().resideInAPackage("..domain.port.out..")
            .because("ADR-002: DIP 적용을 위해 Repository 인터페이스는 domain 레이어에 위치")
            .check(classes);
    }

    /**
     * ADR-002의 결과: infrastructure가 domain을 의존
     */
    @Test
    @DisplayName("ADR-002: JpaRepository 구현체는 domain Port를 구현해야 함")
    void jpa_repositories_must_implement_domain_ports() {
        classes()
            .that().haveNameMatching("Jpa.*Repository")
            .and().areNotInterfaces()
            .should().implement(classesFrom("..domain.port.out.."))
            .because("ADR-002: DIP 적용 구조 강제")
            .check(classes);
    }
}

// ADR-002 위반 시 CI 에러 메시지:
// "FAILED: ADR-002: Repository 인터페이스는 domain/port/out 패키지에 위치해야 함"
// "com.example.infrastructure.OrderRepository가 domain.port.out에 없습니다"
// "참고: docs/adr/0002-repository-in-domain-layer.md"
```

---

## 💻 실전 코드 — 추가 ADR 예시

```markdown
# ADR-003: @Transactional은 UseCase 레이어에 붙인다

## 상태
승인됨 (2024-02-01)

## 맥락
트랜잭션 경계를 어느 레이어에서 정의할지 논의가 필요했습니다.
현재 Repository, Service 등 여러 곳에 @Transactional이 혼재합니다.

## 결정
@Transactional은 UseCase 레이어(Service 클래스)에 붙인다.
Repository 구현체에는 @Transactional을 붙이지 않는다.
(단, 읽기 전용 최적화를 위한 @Transactional(readOnly = true)는 허용)

이유:
  트랜잭션 경계 = UseCase(유스케이스) 경계가 가장 자연스러운 매핑
  "주문 생성" UseCase 전체가 하나의 트랜잭션으로 실행되어야 함
  Repository에 붙이면 여러 Repository 호출이 각각 다른 트랜잭션

## 결과
긍정적:
  트랜잭션 경계가 UseCase로 명확화
  여러 Repository 호출이 하나의 트랜잭션으로 묶임

부정적:
  UseCase 레이어가 Spring(@Transactional)에 의존하게 됨
  → 이 의존을 수용하는 절충을 팀이 동의함

---

# ADR-004: Domain Entity와 JPA Entity를 분리하지 않는다 (절충)

## 상태
승인됨 (2024-02-15)
대체됨 by ADR-008 (2024-09-01, Payment 도메인에서 분리 필요 발생)

## 맥락
Domain Entity(@Entity 없는 순수 Java)와 JPA Entity(@Entity 있는 클래스)를
분리할지 여부를 결정해야 합니다.

## 결정
현재 단계에서는 @Entity 분리를 하지 않는다.
Order.java에 @Entity를 유지하며, JPA 어노테이션 외 비즈니스 로직은 모두 포함한다.

이유:
  현재 JPA 제약(LAZY, 기본 생성자)이 비즈니스 메서드를 방해하지 않음
  매핑 코드(Mapper) 없이 개발 속도 유지

재검토 조건:
  "JPA LAZY 로딩이 비즈니스 메서드에서 예외를 발생시키기 시작할 때"
  "도메인 로직이 JPA 제약으로 우회 코드를 필요로 할 때"

## 결과
부정적 (수용된 트레이드오프):
  Domain Entity에 @Entity가 있어 Entities 레이어 순수성이 완전하지 않음
  → ADR-008에서 Payment 도메인 분리 결정 후 이 ADR이 대체됨
```

---

## 📊 패턴 비교 — ADR이 있을 때 vs 없을 때

```
신규 팀원 질문: "왜 Repository 인터페이스가 domain 패키지에 있나요?"

=== ADR 없음 ===
담당자: "음... 원래 그렇게 설계됐어요"
신규 팀원: "언제부터요? 왜요?"
담당자: "당시 결정한 사람에게 물어봐야 할 것 같아요"
신규 팀원: "그 분이 퇴사하셨는데..."
결과: 이유 모름 → 임의로 infrastructure에 새 Repository 생성 → 혼재

=== ADR 있음 ===
신규 팀원: "왜 Repository 인터페이스가 domain 패키지에 있나요?"
담당자: "docs/adr/0002 보세요"
신규 팀원: (문서 확인) "아, DIP 적용과 MongoDB 전환 때문이었군요"
결과: 이해 → 같은 패턴으로 새 Repository 작성

아키텍처 변경 의사결정 속도 비교:
  ADR 없음: "왜 그렇게 됐는지 모르니 바꾸기 무서움" → 결정 지연
  ADR 있음: "ADR-002의 조건이 바뀌었으니 ADR-008로 대체하자" → 근거 있는 변경
```

---

## ⚖️ 트레이드오프

```
ADR 작성의 비용:
  초안 작성: 1-2시간 (처음 익숙해지기 전)
  리뷰 사이클: 2-5일
  관리: 상태 업데이트, 대체 ADR 연결

ADR 없을 때의 비용:
  신규 팀원 온보딩: "왜 이렇게 됐나요?" 반복 설명 (회당 30분-1시간)
  아키텍처 규칙 위반: 이유 모르고 우회 → 혼재
  의사결정 재검토: "이 결정이 맞나?" 근거 없이 논쟁

ROI 계산:
  ADR 작성 비용: 초안 2시간 + 리뷰 3일
  온보딩 절감: 팀원 1명당 2-4시간 (6개월 기준 팀원 3명 = 6-12시간)
  아키텍처 혼재 예방: 수정 비용 수십 시간
  → 대부분의 팀에서 ADR ROI는 양수

ADR 작성 최소화 방법:
  모든 결정에 ADR 작성은 과잉
  ADR 작성 기준:
    ① 팀 내 논쟁이 있었던 결정
    ② 한 번 결정하면 뒤집기 어려운 결정
    ③ 새 팀원이 "왜?" 물을 것 같은 결정
    → 이 3가지 중 하나에 해당하면 ADR 작성
```

---

## 📌 핵심 정리

```
ADR 핵심:

목적:
  "왜 이 결정을 내렸는가"를 기록 (결정 자체가 아닌 이유)
  신규 팀원 이해 + 규칙 위반 방지 + 의사결정 이력

ADR 템플릿 핵심 섹션:
  맥락(Context): 왜 이 결정이 필요했는가?
  결정(Decision): 무엇을 선택했는가? 왜?
  대안(Alternatives): 어떤 옵션을 검토했는가?
  결과(Consequences): 얻은 것과 포기한 것은?

관리 원칙:
  저장: docs/adr/ 디렉토리 (코드와 같은 저장소)
  리뷰: PR 형태로 팀 검토
  불변: 한 번 승인된 ADR은 수정 없이 새 ADR로 대체
  ArchUnit 연결: ADR의 결정을 코드 규칙으로 자동 강제

ADR 작성 기준:
  팀 논쟁이 있었던 결정
  뒤집기 어려운 결정
  신규 팀원이 "왜?"를 물을 것 같은 결정
```

---

## 🤔 생각해볼 문제

**Q1.** ADR의 내용이 길어지면 실제로 팀원들이 읽지 않게 된다. 효과적인 ADR의 적정 분량과 구조는?

<details>
<summary>해설 보기</summary>

**1-2페이지가 이상적입니다.** 읽는 데 10분을 넘기면 실제로 읽히지 않습니다.

**효과적인 ADR의 특성:**

1. **첫 단락에 결론**: "우리는 Repository 인터페이스를 domain 패키지에 둔다" — 첫 줄에 결론
2. **맥락은 압축**: 세 문장 이내로 "왜 이 결정이 필요했는가"
3. **대안은 표로**: 장황한 설명 대신 비교표
4. **결과는 솔직하게**: 긍정적 결과만이 아니라 수용한 트레이드오프도 명시

피해야 할 것:
- 결정을 정당화하는 긴 설득 글
- 이미 알고 있는 배경 설명 (팀이 다 아는 것)
- 구현 세부사항 (코드는 코드에)

신호: "이 ADR을 읽는 데 10분이 넘는다" → 너무 김, 핵심만 남기고 상세 내용은 링크로

</details>

---

**Q2.** ADR에서 결정이 틀렸다는 것을 나중에 알았을 때 어떻게 처리하는가? 기존 ADR을 수정해도 되는가?

<details>
<summary>해설 보기</summary>

**기존 ADR을 수정하지 않고, 새 ADR로 대체합니다.**

이유: ADR은 결정의 이력(History)입니다. "당시 상황에서 왜 그 결정을 내렸는가"도 가치 있는 정보입니다. 수정하면 이 이력이 사라집니다.

올바른 처리:
```markdown
# ADR-001 (기존)
## 상태
대체됨 by ADR-007 (2024-09-01)
이유: Q3 MongoDB 전환 계획 취소로 ADR-001의 전제 조건 변경

---

# ADR-007: Repository 인터페이스를 Application 레이어로 이동 (ADR-001 대체)

## 상태
승인됨 (2024-09-01)

## 맥락
ADR-001에서 MongoDB 전환 대비로 domain에 Repository를 뒀으나,
Q3 전환 계획이 취소됐습니다. 동시에 팀이 Hexagonal 대신 레이어드+DIP로
단순화하기로 결정했습니다.

## 결정
Repository 인터페이스를 application/port/ 로 이동
...
```

불변성 원칙: 승인된 ADR의 내용은 변경하지 않는다. 상태 필드만 업데이트하고 새 ADR로 대체한다.

</details>

---

**Q3.** Woowacourse 미션처럼 개인 프로젝트나 소규모(1-2인) 팀에서도 ADR이 필요한가?

<details>
<summary>해설 보기</summary>

**소규모에서는 간소화된 형태로 충분합니다.**

1인 프로젝트의 경우:
- 미래의 자신이 신규 팀원
- "왜 이렇게 설계했지?"를 6개월 후에 묻게 됨
- README.md의 "Architecture Decisions" 섹션으로 간소화 가능

```markdown
## Architecture Decisions

### Repository 인터페이스를 domain 패키지에 두는 이유
DIP 적용과 InMemory 테스트를 위해 domain/port/out/ 에 위치.
JpaRepository 구현체는 infrastructure/에 분리.
(ADR-001 참고: 단위 테스트 속도 10배 목표)
```

2인 팀:
- 슬랙이나 PR 댓글에 결정 이유를 남기는 것만으로도 80% 효과
- 결정을 코드에 주석으로: `// ADR-001: DIP를 위해 domain에 위치`

ADR 완전 형식이 필요한 시점:
- 팀 3명 이상
- 6개월 이상 지속 프로젝트
- 아키텍처 규칙 위반이 반복될 때

결론: 형식보다 "왜"를 기록하는 습관이 중요합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 혼합 아키텍처](./03-mixed-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: MSA와 아키텍처 패턴 ➡️](./05-msa-and-architecture-patterns.md)**

</div>
