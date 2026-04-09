# 코드 가이드라인 수립 — 팀이 아키텍처 원칙을 유지하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 신규 기능 추가 시 어느 레이어에 코드를 작성할지 결정하는 트리는?
- 코드 리뷰에서 아키텍처 원칙을 체크하는 기준은?
- 신규 팀원 온보딩 시 아키텍처 원칙을 전달하는 방법은?
- 규칙이 너무 엄격해서 생산성을 낮출 때 완화하는 기준은?
- 팀이 아키텍처 원칙을 지속적으로 유지하는 "지속가능한 방법"은?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

좋은 아키텍처는 팀이 이해하고 실천할 수 있어야 유지된다. 이론적으로 완벽한 아키텍처도 팀이 따르지 못하면 의미가 없다.

가이드라인은 "이론을 실천으로 연결하는 도구"다. 신규 기능을 추가할 때, 코드 리뷰를 할 때, 신규 팀원이 합류했을 때 — 각 상황에서 팀이 같은 기준으로 판단할 수 있게 한다.

---

## 😱 흔한 실수 (Before — 가이드라인 없이 팀이 각자 판단)

```
가이드라인 없는 팀의 상황:

신규 기능 추가 시:
  개발자 A: "이 비즈니스 로직은 Service에 넣겠습니다"
  개발자 B: "저는 Domain 객체에 넣습니다"
  개발자 C: "Controller에서 처리했습니다 (간단한 거라서)"
  → 같은 팀, 다른 기준, 혼재된 코드

코드 리뷰 시:
  리뷰어: "이 if문은 Service에 있으면 안 되는 것 같은데..."
  개발자: "네? 지금까지 Service에 이런 거 많이 있었는데요?"
  → 기준 불명확으로 리뷰어-개발자 논쟁 발생

신규 팀원 합류 시:
  "여기는 어떤 아키텍처를 쓰나요?"
  "팀마다 달라요. 코드 보면서 파악하세요."
  → 1달 후 혼재된 방식으로 코드 작성
```

---

## ✨ 올바른 접근 (After — 문서화된 가이드라인 + 자동화 + 주기적 리뷰)

```
팀 가이드라인 구성 요소:

1. 파일 위치 결정 트리 (어디에 코드를 넣는가?)
2. 코드 리뷰 체크리스트 (어떤 기준으로 리뷰하는가?)
3. 온보딩 가이드 (신규 팀원이 첫 1주 동안 배워야 할 것)
4. 가이드라인 완화 기준 (언제 예외를 허용하는가?)
5. 주기적 아키텍처 리뷰 (어떻게 가이드라인을 발전시키는가?)

자동화로 보완:
  ArchUnit → 규칙 위반 자동 차단 (기계)
  PR 템플릿 → 아키텍처 체크리스트 포함 (사람 + 프로세스)
  온보딩 문서 → 신규 팀원 자습 가능 (문서)
```

---

## 🔬 내부 원리 — 가이드라인 구성 요소

### 1. 파일 위치 결정 트리

```
새 코드를 작성할 때 다음 질문에 답해서 위치를 결정합니다:

================================
질문 1: 이 코드가 비즈니스 규칙인가?
  "도메인 전문가(비개발자)가 말하는 규칙인가?"
  "기술(JPA, HTTP)과 무관하게 항상 적용되는가?"
================================
  YES → {feature}/domain/model/에 Entity, VO, Enum으로

================================
질문 2: 외부에서 내 기능을 어떻게 호출하는가의 계약인가?
================================
  YES → {feature}/domain/port/in/에 UseCase 인터페이스

================================
질문 3: 내 기능이 외부를 어떻게 쓰는가의 계약인가?
  "Repository, 외부 API, 이벤트 발행 인터페이스인가?"
================================
  YES → {feature}/domain/port/out/에 Port 인터페이스

================================
질문 4: 비즈니스 흐름(여러 Domain 객체의 협력)을 조율하는가?
  "재고 확인 → 결제 → 저장 같은 순서 결정인가?"
  "@Transactional이 필요한가?"
================================
  YES → {feature}/application/service/에 @Service 클래스

================================
질문 5: HTTP 요청/응답 처리인가?
  "REST API 엔드포인트인가?"
  "HTTP DTO인가?"
  "HTTP ↔ UseCase 모델 변환인가?"
================================
  YES → {feature}/adapter/in/web/에 Controller, DTO, Mapper

================================
질문 6: DB 접근 구현인가?
  "JPA Repository 구현인가?"
  "JPA Entity 클래스인가?"
  "Domain ↔ JPA 변환 Mapper인가?"
================================
  YES → {feature}/adapter/out/persistence/

================================
질문 7: 외부 API 연동인가?
  "결제 API, SMS, 이메일 클라이언트인가?"
================================
  YES → {feature}/adapter/out/{서비스명}/

================================
질문 8: 여러 Feature가 공유하는 개념인가?
  "여러 Feature가 같은 Value Object를 쓰는가?"
================================
  YES → shared/domain/ (공유 Value Object)

결정을 못 하겠으면:
  "이 코드가 바뀔 때 함께 바뀌는 것은 무엇인가?"
  → 함께 바뀌는 것끼리 같은 위치에
```

### 2. 코드 리뷰 체크리스트

```markdown
# PR 아키텍처 체크리스트

## 필수 체크 (ArchUnit으로 자동화된 것은 skip 가능)

### 의존성 방향
- [ ] Domain 코드에 Spring, JPA, Kafka import가 없는가?
- [ ] Application 코드에 HTTP, JPA 구체 클래스 import가 없는가?
- [ ] Controller가 Domain 객체를 직접 생성하지 않는가?
      (Port → Service → Domain 순서를 통하는가?)

### 비즈니스 로직 위치
- [ ] 비즈니스 규칙(if문, 계산)이 Domain 또는 Application에 있는가?
- [ ] Controller에 비즈니스 판단이 없는가?
      (입력 형식 검증(@Valid)은 OK, 비즈니스 검증은 Domain으로)
- [ ] JPA Repository 또는 Adapter에 비즈니스 조건이 없는가?
      ("활성 주문" 같은 비즈니스 정의가 쿼리에 있으면 Domain으로 이동)

### 테스트
- [ ] Domain 비즈니스 로직이 순수 Java 단위 테스트로 검증되는가?
- [ ] Application UseCase가 InMemory Adapter로 테스트되는가?
- [ ] @SpringBootTest 비율이 20% 미만인가?

### 파일 위치
- [ ] Port 인터페이스가 domain/port/에 있는가?
- [ ] JPA Entity(@Entity)가 adapter/out/persistence/entity/에 있는가?
- [ ] UseCase 구현체가 application/service/에 있는가?

## 권장 체크 (규칙 아닌 가이드라인)
- [ ] Command/Result 객체가 HTTP DTO와 분리되는가?
- [ ] Domain 객체에 setter가 없는가? (불변성)
- [ ] 새 Feature가 기존 Feature를 직접 import하지 않는가?
```

### 3. 신규 팀원 온보딩 가이드

```markdown
# 온보딩 Week 1: 아키텍처 이해

## Day 1-2: 읽기
1. ARCHITECTURE.md 읽기 (30분)
   → 우리 팀이 사용하는 패턴과 그 이유
2. ADR 목록 읽기 (docs/adr/) (1시간)
   → 주요 결정과 이유
3. ArchUnit 규칙 코드 읽기 (30분)
   → 자동으로 강제되는 규칙 목록

## Day 3: 코드 탐색
1. Order 기능 전체 코드 탐색 (2시간)
   order/domain/model/Order.java → 비즈니스 규칙 위치 확인
   order/domain/port/in/ → UseCase 계약 확인
   order/application/service/ → 흐름 조율 확인
   order/adapter/in/web/ → HTTP 변환 확인
   order/adapter/out/persistence/ → DB 접근 확인

2. 단위 테스트 실행 (30분)
   ./gradlew test --tests "*.order.*"
   → 테스트 속도와 구조 파악

## Day 4-5: 첫 번째 기능
1. 작은 기능 구현 (멘토와 함께)
   → 파일 위치 결정 트리를 따라 위치 선정
   → ArchUnit 통과 확인
   → 코드 리뷰 체크리스트 적용

## 자주 묻는 질문 (FAQ)
Q: "Repository 인터페이스가 왜 domain/port/out/에 있나요?"
A: ADR-002 참고. DIP 적용으로 Service가 인프라를 모르게 하기 위함.

Q: "@Transactional을 Service에 붙이는데 Application이 Spring을 아는 거 아닌가요?"
A: ADR-003 참고. 실용적 절충으로 허용. 완전 분리는 비용이 이익보다 큼.

Q: "도메인 로직과 Application 로직의 차이가 헷갈려요"
A: 도메인 = "최소 주문 금액 규칙" (항상 적용)
   Application = "주문 생성 시 재고→결제→저장 순서" (이 앱에서만)
```

### 4. 가이드라인 완화 기준

```
언제 예외를 허용하는가:

원칙: "규칙을 어기기 전에 두 가지를 먼저 확인하라"
  ① 이 규칙이 왜 있는지 이해했는가?
  ② 규칙을 지키는 더 나은 방법은 없는가?

완화 허용 케이스:

케이스 1: 단순 CRUD 기능에 Hexagonal 완전 적용이 과잉
  원칙: Port/Adapter 완전 분리
  완화: "비즈니스 로직이 없는 기능(공지사항 등)은 레이어드+DIP 허용"
  기록: ADR에 "어떤 기능에 완화가 허용되는가" 명시

케이스 2: @Transactional이 Application에 있어 Spring 의존
  원칙: Application은 Spring을 모름
  완화: "Application에 @Transactional 허용 (Decorator 비용이 이익보다 큼)"
  기록: ADR-003에 명시됨

케이스 3: Domain Entity에 @Entity 허용
  원칙: Domain이 JPA를 모름
  완화: "JPA 제약이 도메인을 오염시키지 않을 때 절충 허용"
  조건: "LAZY 예외, 기본 생성자 이슈가 없어야 함"
  기록: ADR-004에 명시됨

케이스 4: 외부 라이브러리(Lombok)의 Domain 사용
  원칙: Domain에 외부 라이브러리 없음
  완화: "Lombok @Getter, @Builder는 허용 (코드 품질 개선 도구)"
  ArchUnit 예외: org.projectlombok.. 패키지를 제외 목록에 추가

완화 프로세스:
  1. 팀에 제안 (Slack 또는 회의)
  2. 다수 동의 후 ADR에 기록
  3. ArchUnit 예외 코드 추가 (이유 주석 포함)
  4. ARCHITECTURE.md 업데이트
```

### 5. 주기적 아키텍처 리뷰

```
아키텍처가 시간이 지나면서 나빠지는 것을 방지하는 방법:

월 1회 아키텍처 리뷰:
  시간: 30-60분
  참여: 팀 전체 (의무)
  도구: ArchUnit 리포트, 의존성 그래프

리뷰 체크포인트:
  ① 최근 ArchUnit 예외가 추가됐는가?
     → 예외 이유가 타당한지, 제거 계획이 있는지

  ② 특정 클래스의 의존성 수가 10개를 넘는가?
     → Fat Service 조짐 → 분리 계획

  ③ 새 기능의 파일 위치가 가이드라인과 일치하는가?
     → 벗어난 경우 가이드라인 업데이트 필요 여부 검토

  ④ 테스트 피라미드 비율은 유지되는가?
     → @SpringBootTest 비율 모니터링 (20% 이하 목표)

  ⑤ ADR이 최신 상태인가?
     → 변경된 결정이 ADR에 반영됐는가?

의존성 시각화 도구:
  IntelliJ → Analyze → Dependency Matrix
  Gradle: ./gradlew dependencies
  ArchUnit: 의존성 그래프 HTML 리포트 생성 가능

결과물:
  회의 후 1주 이내 ADR 업데이트 또는 ArchUnit 규칙 수정 완료
  "나중에 고치자" → 다음 달 아키텍처 리뷰 아젠다에 등록
```

---

## 💻 실전 코드 — ARCHITECTURE.md 예시

```markdown
# ARCHITECTURE.md

## 아키텍처 패턴

우리 팀은 **기능 기반 패키지 구조 + Hexagonal Architecture**를 사용합니다.

### 패키지 구조

각 Feature(order, payment 등)는 다음 내부 구조를 가집니다:

```
{feature}/
├── domain/
│   ├── model/          ← 순수 비즈니스 규칙 (Spring/JPA 없음)
│   └── port/
│       ├── in/         ← UseCase 인터페이스 (Driving Port)
│       └── out/        ← Repository, API 인터페이스 (Driven Port)
├── application/
│   └── service/        ← @Service, @Transactional (UseCase 구현)
└── adapter/
    ├── in/web/         ← @RestController, DTO, Mapper
    └── out/
        ├── persistence/← @Repository, @Entity, JPA Mapper
        └── {service}/  ← 외부 API Adapter
```

### 핵심 규칙 (ArchUnit으로 강제됨)

1. **Domain은 Spring, JPA, HTTP를 import하지 않는다**
2. **Domain/Application은 Adapter를 import하지 않는다**
3. **Port 인터페이스는 domain/port/에 위치한다**
4. **@Entity 클래스는 adapter/out/persistence/entity/에만 위치한다**

### 허용된 절충 (ADR에 이유 기록됨)

| 절충 | 이유 | ADR |
|------|------|-----|
| Application에 @Transactional | Spring 의존 허용 (복잡도 감소) | ADR-003 |
| Domain에 @Entity 허용 | JPA 제약이 도메인 오염 없을 때 | ADR-004 |
| CRUD 기능은 레이어드+DIP | Hexagonal 과잉 방지 | ADR-005 |

### 새 기능 추가 시 결정 트리

[파일 위치 결정 트리 참고](./docs/dev-guide/file-location-decision-tree.md)

### 코드 리뷰 체크리스트

[PR 체크리스트 참고](.github/pull_request_template.md)
```

---

## 📊 패턴 비교 — 가이드라인 적용 효과

```
가이드라인 없을 때 vs 있을 때:

신규 팀원 온보딩 (첫 기능 구현까지):
  없음: 2-4주 (코드 탐색 + 질문 + 시행착오)
  있음: 1주 (문서 읽기 + 결정 트리 따라 구현)

코드 리뷰 시간 (PR당):
  없음: 60분 (아키텍처 논쟁 포함)
  있음: 30분 (비즈니스 로직에 집중)

아키텍처 위반 발견 시점:
  없음: 배포 후 또는 나중에
  있음: PR 단계 (ArchUnit + 체크리스트)
```

---

## ⚖️ 트레이드오프

```
가이드라인 수립 비용:
  초기 작성: ARCHITECTURE.md + 결정 트리 + 온보딩 가이드 (1-2일)
  유지 관리: 월 1회 아키텍처 리뷰 (30-60분)
  업데이트: 결정 변경 시 문서 동기화 (30분-1시간)

가이드라인이 없을 때의 비용:
  신규 팀원 온보딩: 반복 질문에 답변 (팀원 시간 소모)
  코드 리뷰 논쟁: 기준 불명확으로 논쟁 (시간 소모)
  아키텍처 부패: 일관성 없는 코드 (장기 유지보수 비용)

문서가 코드와 달라지는 문제 방지:
  "코드 변경 시 ARCHITECTURE.md 업데이트" → PR 체크리스트에 포함
  ADR 변경 → ADR 파일 업데이트 (코드와 함께 커밋)
```

---

## 📌 핵심 정리

```
팀 아키텍처 가이드라인 핵심 구성요소:

① 파일 위치 결정 트리 (가장 중요)
   "새 코드를 어디에 넣는가" 질문 순서

② PR 체크리스트
   의존성 방향, 비즈니스 로직 위치, 테스트 비율

③ 신규 팀원 온보딩 가이드
   Day 1-5 학습 로드맵 + FAQ

④ 완화 기준
   예외 허용 조건과 기록 프로세스

⑤ 주기적 아키텍처 리뷰 (월 1회)
   의존성 그래프 점검, ArchUnit 예외 검토, ADR 동기화

자동화 + 문서화 + 리뷰의 조합:
  ArchUnit → 위반 자동 차단 (기계)
  PR 체크리스트 → 인간이 놓친 것 (프로세스)
  월 리뷰 → 장기적 일관성 (문화)
```

---

## 🤔 생각해볼 문제

**Q1.** 가이드라인을 만들었는데 팀원 중 한 명이 "이 규칙이 우리 상황에 맞지 않는다"고 주장한다. 어떻게 대응하는가?

<details>
<summary>해설 보기</summary>

**두 가지 경우로 나눠서 대응합니다.**

경우 1: 타당한 근거가 있는 주장
- 그 의견을 ADR 초안으로 문서화
- 팀 전체 리뷰 (비동기 PR 또는 회의)
- 다수 동의 시 가이드라인 업데이트

경우 2: 선호도 차이인 경우
- "이 규칙이 있는 이유(ADR)"를 공유
- 대안을 제시하게 함: "이 규칙을 바꾸면 어떤 이익이 있는가?"
- 이익이 비용보다 크면 변경, 그렇지 않으면 기존 유지

중요한 것: 모든 의견을 경청하되, 개인 선호도가 아닌 **팀 전체의 이익**을 기준으로 결정. 한 사람의 반대로 합의된 규칙이 깨지면 가이드라인의 신뢰도가 무너집니다.

</details>

---

**Q2.** 긴급 배포 상황에서 아키텍처 규칙을 어기는 "기술 부채 코드"를 작성해야 할 때 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**작성은 허용하되, 반드시 추적 가능하게 합니다.**

```java
// TODO: [ARCH-DEBT] 긴급 배포로 인한 임시 코드
// 이유: 2024-12-15 결제 오류 긴급 수정
// 해결 기한: 2025-01-15
// 이슈: JIRA-1234
@ArchIgnore
@Service
public class EmergencyOrderService {
    @Autowired
    JpaOrderRepository jpaRepository; // ← 임시: Port 없이 직접 의존
}
```

프로세스:
1. `// TODO: [ARCH-DEBT]` 주석 필수
2. 해결 기한 명시
3. 이슈 트래커(Jira, GitHub Issues)에 티켓 생성
4. 다음 아키텍처 리뷰에서 해결 여부 확인

기한이 지나도 해결 안 됐으면:
- 아키텍처 리뷰에서 "기술 부채 해소" 아이템으로 우선순위 상향
- "나중에"가 영원히 안 오는 것을 막는 것이 핵심

</details>

---

**Q3.** 아키텍처 가이드라인을 만들었는데 6개월 후 팀원 중 아무도 읽지 않는 "죽은 문서"가 됐다. 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

**문서를 "참조 도구"가 아닌 "일상 워크플로"에 포함시켜야 합니다.**

죽은 문서의 원인: 문서가 있지만 사용할 필요가 없음 (기억으로 해결 가능한 수준)

방지 방법:

1. **PR 템플릿에 링크**: 
   ```markdown
   ## 아키텍처 체크
   - [ ] 파일 위치 결정 트리 확인: [링크]
   ```

2. **온보딩에 "문서 읽기" 포함**: Day 1 필수 항목으로 구조화

3. **가이드라인이 답을 제공해야 함**: 
   "Controller에 비즈니스 로직을 넣으면 안 되는 이유"를 설명하는 문서는 살아있음
   "어디에 코드를 넣는가"라는 질문의 답을 주는 문서는 살아있음

4. **월 리뷰에서 문서 동기화**: "이 가이드라인이 현재 코드를 반영하는가?" 체크

5. **팀원이 기여할 수 있게**: 가이드라인이 PR을 통해 업데이트됨 → 팀원이 "내 것"으로 느낌

가장 중요한 것: **가이드라인이 일상 작업에 도움이 되어야** 팀원이 읽습니다. "어디에 파일을 만드는가"가 실제로 매일 결정해야 하는 것이라면, 그 가이드라인은 살아있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: ArchUnit으로 아키텍처 테스트](./04-archunit-architecture-tests.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 실전 프로젝트 ➡️](../refactoring-project/01-legacy-code-analysis.md)**

</div>
