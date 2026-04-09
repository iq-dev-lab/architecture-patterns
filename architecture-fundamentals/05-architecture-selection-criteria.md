# 아키텍처 선택 기준 — 언제 Hexagonal이고 언제 레이어드인가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 도메인 복잡도에 따라 어떤 아키텍처를 선택해야 하는가?
- 팀 규모와 학습 곡선을 고려한 현실적인 아키텍처 선택 기준은 무엇인가?
- 과잉 엔지니어링의 징후는 무엇이고, 어떻게 피하는가?
- 레이어드 아키텍처에서 Hexagonal로 점진적으로 이행하는 전략은 무엇인가?
- 테스트 자동화 수준이 아키텍처 선택에 어떤 영향을 주는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

"어떤 아키텍처가 더 좋은가?"라는 질문에는 정답이 없다. 상황에 따라 다르다.

Hexagonal Architecture를 모든 서비스에 적용하면 단순 CRUD에 과잉 복잡도가 생긴다. 반대로 레이어드 아키텍처만 고집하면 복잡한 도메인에서 기술 부채가 쌓인다.

아키텍처 선택의 핵심은 현재 도메인의 복잡도, 팀의 역량, 변경 예상 방향을 종합해서 **변경 비용을 최소화하는 결정**을 내리는 것이다.

---

## 😱 흔한 실수 (Before — 아키텍처 선택 기준이 없을 때)

```
실수 패턴 1: 무조건 레이어드
  "Spring Boot 시작하면 Controller-Service-Repository지"
  → 3년 후 1,000줄 Service, 테스트 없음, 아무도 건드리기 싫은 코드

실수 패턴 2: 무조건 Hexagonal
  "요즘 Hexagonal이 대세니까 다 이걸로 하자"
  → 3개 엔티티짜리 내부 어드민에 Port/Adapter 10개
  → 팀원들 "어디에 코드 넣어야 하지?" 혼란
  → 개발 속도 70% 감소

실수 패턴 3: 중간에 갑자기 전환
  "우리 코드가 너무 복잡해졌으니 이번 스프린트에 전부 Hexagonal로 바꾸자"
  → 2주 동안 기능 개발 없이 리팩터링만
  → 완성하지 못하고 중간에 포기
  → 레이어드와 Hexagonal이 혼재된 최악의 상태

실수 패턴 4: 아키텍처가 팀 문서에만 존재
  → 실제 코드는 정의된 아키텍처를 따르지 않음
  → 새 기능 추가할 때마다 개인 판단으로 위치 결정
  → 코드베이스가 아키텍처가 아닌 개인 스타일의 혼합
```

---

## ✨ 올바른 접근 (After — 상황에 맞는 아키텍처 선택)

```
올바른 선택 프레임워크:

Step 1: 도메인 복잡도 평가
  낮음: CRUD 위주, 비즈니스 규칙 5개 이하
  중간: 복잡한 비즈니스 규칙, 외부 시스템 연동 2-3개
  높음: 도메인 전문 지식이 필요, 외부 시스템 5개 이상, 잦은 요구사항 변화

Step 2: 팀 역량 평가
  레이어드 숙련도, Hexagonal 경험 유무, 학습 시간 확보 가능 여부

Step 3: 변경 예상 방향
  외부 시스템 교체 가능성, 비즈니스 규칙 변화 빈도, 서비스 규모 성장 예상

선택 결과:
  도메인 낮음 + 팀 소규모 + 단기 프로젝트 → 레이어드 (개선된 형태)
  도메인 중간 + 팀 성장 중 + 장기 프로젝트 → 레이어드 → 점진적 Hexagonal
  도메인 높음 + 팀 경험 있음 + 장기 프로젝트 → Hexagonal or Clean
```

---

## 🔬 내부 원리 — 아키텍처 선택 결정 요소

### 1. 도메인 복잡도 — 가장 중요한 기준

```
도메인 복잡도 측정 기준:

낮음 (레이어드 적합):
  - CRUD 작업이 대부분
  - 비즈니스 규칙: "5개 이하의 단순 조건"
  - 외부 시스템 연동: 없음 or 1개
  - 도메인 전문 지식 불필요
  - 요구사항 변화: 드문 편

  예시 서비스:
    사내 게시판 / 공지사항 시스템
    단순 주소록 관리 서비스
    내부 팀 일정 관리 도구

중간 (레이어드 개선 or Hexagonal 시작):
  - 복잡한 비즈니스 규칙 존재 (할인 정책, 권한 체계 등)
  - 외부 시스템 연동: 2-4개 (결제, 알림, 배송 등)
  - 도메인 특화 용어와 규칙이 있음
  - 요구사항 변화: 월 1-2회

  예시 서비스:
    이커머스 주문/결제 서비스
    예약 관리 시스템
    구독 서비스 관리

높음 (Hexagonal or Clean 적합):
  - 복잡한 비즈니스 도메인 (금융, 의료, 물류)
  - 외부 시스템 연동: 5개 이상
  - 도메인 전문가와 협업 필수 (DDD가 필요한 수준)
  - 요구사항 변화: 주 1-2회 이상

  예시 서비스:
    보험 계약 관리 시스템
    병원 EMR 시스템
    복합 물류 관리 플랫폼
```

### 2. 팀 역량과 학습 곡선

```
아키텍처별 팀 학습 비용:

레이어드 아키텍처:
  학습 곡선: 낮음
  → Spring Boot 기본 강의에서 배움
  → 대부분의 개발자가 이미 알고 있음
  → 신규 팀원 온보딩: 1-2일

Hexagonal Architecture:
  학습 곡선: 중간
  → Port / Adapter / Driving / Driven 개념 학습 필요
  → 패키지 구조 규칙 내재화 필요
  → 신규 팀원 온보딩: 1-2주 (코드 리뷰 병행 시)

Clean Architecture:
  학습 곡선: 높음
  → 4개 레이어의 의미와 의존성 규칙 숙달 필요
  → Input/Output Boundary, Presenter 개념 익숙해지기까지
  → 신규 팀원 온보딩: 2-4주

팀 규모별 권장:

  1-2명 (스타트업 초기):
    → 레이어드 개선형 (DIP 적용)
    → 빠른 개발 속도 우선
    → 팀이 커지기 시작할 때 점진적 전환

  3-5명 (소규모 팀):
    → Hexagonal 시작 권장 (핵심 도메인부터)
    → 팀 전체가 동일한 패턴 사용 중요
    → 코드 리뷰에서 아키텍처 원칙 검증

  6명 이상 (중규모 팀):
    → Hexagonal or Clean (규칙 없으면 혼돈)
    → ArchUnit으로 아키텍처 규칙 자동 강제
    → 모듈 경계가 팀 경계와 일치하도록
```

### 3. 외부 시스템 교체 가능성 — Hexagonal 필요성의 핵심 지표

```
외부 시스템 교체 가능성 평가:

교체 가능성 높음 → Hexagonal 강력 권장:
  결제 수단 (카카오페이 → 토스페이 → 네이버페이)
  SMS/이메일 발송 (AWS SES → 네이버 클라우드)
  검색 엔진 (MySQL FULLTEXT → Elasticsearch)
  메시지 큐 (RabbitMQ → Kafka)

교체 가능성 낮음 → 레이어드로 충분:
  사내 전용 레거시 시스템 (절대 안 바뀜)
  단일 DB만 사용하는 단순 서비스
  외부 연동이 전혀 없는 내부 도구

Hexagonal의 핵심 가치:
  "외부 시스템이 바뀌어도 도메인 코드는 변하지 않는다"
  외부 시스템 교체 가능성이 낮다면 이 가치가 감소

테스트 관점에서의 판단:
  "외부 API 없이 비즈니스 로직을 테스트하고 싶은가?"
  → Yes → Port/Adapter 구조 필요 (Hexagonal)
  → No  → 레이어드 + Mockito로 충분
```

### 4. 과잉 엔지니어링의 징후와 피하는 방법

```
과잉 엔지니어링의 신호:

코드 신호:
  ① 3개 엔티티짜리 서비스에 Port 12개
  ② UseCase 인터페이스가 하나의 구현체만 존재하고 교체 계획 없음
  ③ 매핑 코드(Entity ↔ Domain)가 비즈니스 코드보다 많음
  ④ 팀원이 "새 기능 어디에 넣어야 해요?" 자주 물어봄

팀 신호:
  ① 아키텍처 설명에 30분 이상 필요
  ② 신규 팀원이 첫 주에 코드를 거의 못 건드림
  ③ 간단한 CRUD 추가에 5개 파일 이상 수정 필요

과잉 엔지니어링을 피하는 원칙 (YAGNI):
  "You Aren't Gonna Need It"
  → 지금 필요하지 않은 추상화를 추가하지 마라
  → 두 번째 구현체가 생길 때 인터페이스 추출
  → 교체 가능성이 명확할 때만 Port 설계

현실적 절충:
  모든 레이어에 Hexagonal 적용 X
  → 핵심 도메인 (결제, 주문, 사용자)에만 적용
  → 단순 조회 API는 레이어드로 충분
  → 점진적으로 확장
```

### 5. 점진적 아키텍처 이행 전략

```
레이어드 → Hexagonal 점진적 전환:

Phase 0: 현재 상태 파악 (1주)
  - 의존성 그래프 그리기 (Mermaid or 화이트보드)
  - 가장 복잡한 Service 파악 (줄 수 기준)
  - 테스트 커버리지 측정 (Before 기준선)

Phase 1: Repository 인터페이스를 도메인으로 (1-2일)
  Before: OrderService → OrderJpaRepository
  After:  OrderService → OrderRepository(I/F)
          JpaOrderRepository implements OrderRepository

  리스크: 낮음 (Spring DI로 투명하게 전환)
  이익: OrderService 단위 테스트 가능해짐

Phase 2: 외부 의존성 Port로 추상화 (2-3일/Port)
  Before: OrderService → KakaoPayClient
  After:  OrderService → PaymentPort(I/F)
          KakaoPayAdapter implements PaymentPort

  리스크: 중간 (Adapter 매핑 코드 추가)
  이익: 외부 API 없이 단위 테스트 가능

Phase 3: Entity와 Domain Model 분리 (1주)
  Before: Order (@Entity + 비즈니스 로직)
  After:  Order (순수 Java) + OrderJpaEntity (@Entity)
          + OrderMapper (변환)

  리스크: 높음 (전체 코드 영향) — 테스트 먼저!
  이익: 도메인이 JPA를 완전히 모름

Phase 4: UseCase 인터페이스 도입 (1-2일)
  Before: OrderController → OrderService (직접)
  After:  OrderController → PlaceOrderUseCase(I/F)
          PlaceOrderService implements PlaceOrderUseCase

  리스크: 낮음
  이익: Controller 테스트 시 UseCase Mock 간단

전환 시 주의사항:
  ① 한 번에 하나의 Phase만 진행
  ② 각 Phase 완료 후 테스트 통과 확인
  ③ 기능 개발과 병행 (Strangler Fig 방식)
  ④ PR 단위를 작게 유지 (리뷰 용이)
```

---

## 💻 실전 코드 — 도메인 복잡도별 아키텍처 선택 예시

```java
// === 낮은 복잡도: 레이어드 (개선형) 충분 ===
// 사내 공지사항 서비스 — CRUD, 외부 연동 없음

@RestController
class NoticeController {
    @GetMapping("/notices/{id}")
    NoticeResponse findById(@PathVariable Long id) {
        return noticeService.findById(id);
    }
}

@Service
public class NoticeService {
    private final NoticeRepository noticeRepository; // 인터페이스 의존 (최소 개선)
    // 외부 연동 없으므로 Port 불필요
}

// 단순하지만 최소한의 원칙(인터페이스)은 지킴


// === 중간 복잡도: Hexagonal 시작 ===
// 이커머스 주문 서비스 — 복잡한 비즈니스 규칙, 결제/알림 연동

// domain/
public class Order {
    public void place(OrderPolicy policy) { /* 복잡한 비즈니스 규칙 */ }
    public void cancel(CancelPolicy policy) { /* 복잡한 취소 정책 */ }
    public void applyDiscount(DiscountPolicy discount) { /* 할인 정책 */ }
}
public interface OrderRepository { /* Driven Port */ }
public interface PaymentPort { /* Driven Port */ }
public interface NotificationPort { /* Driven Port */ }

// application/
@Service
public class PlaceOrderService implements PlaceOrderUseCase {
    // 비즈니스 로직 조율 (Port에만 의존)
}

// infrastructure/
@Repository class JpaOrderRepository implements OrderRepository { /* Adapter */ }
@Component class KakaoPayAdapter implements PaymentPort { /* Adapter */ }
@Component class SlackNotificationAdapter implements NotificationPort { /* Adapter */ }


// === 높은 복잡도: Clean Architecture ===
// 보험 계약 관리 — 매우 복잡한 도메인 규칙, 다수 외부 시스템

// Entities (가장 안쪽 — 아무것도 모름)
public class InsuranceContract {
    private final ContractId id;
    private final Premium premium;
    private final CoverageScope scope;
    // 순수 보험 비즈니스 규칙 (계산, 검증)
    public Premium calculatePremium(RiskFactor factor) { /* ... */ }
}

// Use Cases
public interface CreateContractUseCase {
    ContractId createContract(CreateContractCommand command);
}
public interface CreateContractOutputPort { // Output Boundary
    void present(ContractCreatedResponse response);
}

@Service
public class CreateContractInteractor implements CreateContractUseCase {
    private final ContractRepository contractRepository; // Gateway
    private final RiskAssessmentPort riskPort;           // Gateway
    private final CreateContractOutputPort outputPort;   // Output Boundary
    // Use Case 로직
}

// Interface Adapters
@RestController
public class ContractController {
    // HTTP Request → CreateContractCommand 변환
    // CreateContractUseCase 호출
    // Response를 ViewModel로 변환 (Presenter)
}

@Repository
public class JpaContractRepository implements ContractRepository { /* Gateway */ }
```

---

## 📊 패턴 비교 — 상황별 아키텍처 선택표

```
도메인 복잡도 × 외부 연동 수에 따른 권장 아키텍처:

              외부 연동 없음   외부 연동 1-3개    외부 연동 4개 이상
            ┌──────────────┬────────────────┬──────────────────┐
  복잡도 낮음 │  레이어드     │  레이어드 개선형  │  레이어드 개선형   │
  (단순 CRUD) │  (기본형)     │  (Port 일부)    │  or Hexagonal 시작│
            ├──────────────┼────────────────┼──────────────────┤
  복잡도 중간 │  레이어드 개선  │  Hexagonal      │  Hexagonal       │
  (비즈니스   │  (DIP 적용)   │  (핵심 도메인)   │  (전체 도메인)    │
  규칙 있음)  │              │                │                  │
            ├──────────────┼────────────────┼──────────────────┤
  복잡도 높음 │  Hexagonal   │  Hexagonal      │  Clean           │
  (도메인 전문│  or Clean    │  or Clean       │  Architecture    │
  지식 필요)  │              │                │                  │
            └──────────────┴────────────────┴──────────────────┘

아키텍처별 핵심 지표 비교:

지표                      │ 레이어드  │ 레이어드 개선 │ Hexagonal │ Clean
─────────────────────────┼──────────┼─────────────┼───────────┼──────────
단위 테스트 속도           │ 느림      │ 중간         │ 빠름      │ 빠름
외부 시스템 교체 용이성    │ 어려움    │ 중간         │ 쉬움      │ 쉬움
코드베이스 복잡도          │ 낮음      │ 중간         │ 중간      │ 높음
팀 학습 비용              │ 낮음      │ 낮음         │ 중간      │ 높음
신규 팀원 온보딩           │ 1-2일    │ 3-5일        │ 1-2주     │ 2-4주
초기 개발 속도            │ 빠름      │ 중간         │ 중간      │ 느림
6개월 후 유지보수 용이성   │ 낮음      │ 중간         │ 높음      │ 높음
```

---

## ⚖️ 트레이드오프

```
각 아키텍처의 핵심 트레이드오프:

레이어드 (기본형):
  장점: 빠른 개발 속도, 낮은 학습 비용, 모든 팀원이 알고 있음
  단점: 도메인 복잡도가 높아지면 빠르게 기술 부채가 쌓임
  적합: 단순 CRUD, 단기 프로젝트, 소규모 팀

레이어드 (개선형, DIP 적용):
  장점: 학습 비용 낮으면서 테스트 가능성 향상
  단점: Hexagonal만큼 도메인 순수성 보장 못 함
  적합: 팀이 Hexagonal로 전환 중인 과도기

Hexagonal:
  장점: 도메인 순수성, 빠른 단위 테스트, 외부 시스템 교체 용이
  단점: 코드 파일 수 증가, 매핑 코드 추가, 학습 필요
  적합: 복잡한 비즈니스 도메인, 외부 시스템 연동 많은 서비스

Clean Architecture:
  장점: 레이어 경계 엄격, 팀 대규모에서 일관성 유지
  단점: 가장 높은 학습 비용, 가장 많은 코드량
  적합: 매우 복잡한 도메인, 대규모 팀, 장기 프로젝트

현실적 조언:
  "처음부터 Clean Architecture를 완벽하게 적용하기보다
   핵심 원칙(DIP, SRP)을 지키면서 점진적으로 개선하는 것이
   대부분의 팀에서 현실적이다"
```

---

## 📌 핵심 정리

```
아키텍처 선택 결정 기준 요약:

도메인 복잡도 (가장 중요):
  낮음 → 레이어드 (개선형이면 충분)
  중간 → Hexagonal 시작 (핵심 도메인부터)
  높음 → Hexagonal or Clean

팀 역량:
  Hexagonal 경험 없음 → 점진적 전환 (Phase 단위)
  경험 있음 → 처음부터 Hexagonal 적용

외부 시스템 교체 가능성:
  높음 → Hexagonal (Port/Adapter)
  낮음 → 레이어드 개선형 충분

과잉 엔지니어링 방지:
  YAGNI 원칙 → 지금 필요한 추상화만
  두 번째 구현체가 생길 때 인터페이스 추출

점진적 이행 순서:
  1. Repository 인터페이스 도메인으로
  2. 외부 의존성 Port 추상화
  3. Entity와 Domain Model 분리
  4. UseCase 인터페이스 도입
  → 각 단계마다 테스트 통과 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 현재 레이어드 아키텍처로 운영 중인 서비스가 있다. 모든 Service가 500줄 이상이고 @SpringBootTest 없이는 테스트가 불가능하다. 이 서비스를 Hexagonal로 전환하려면 가장 먼저 무엇을 해야 하는가?

<details>
<summary>해설 보기</summary>

**가장 먼저 해야 할 것: 테스트 커버리지 확보 (전환 전)**

테스트 없이 리팩터링하는 것은 "눈 감고 수술"입니다. 전환 전에:

1. **현재 동작을 @SpringBootTest 기반의 통합 테스트로 먼저 커버**
   - 완벽할 필요 없음, 핵심 시나리오 5-10개
   - 이것이 리팩터링 후에도 동작하는지 검증하는 안전망

2. **의존성 그래프를 그려서 가장 많이 의존받는 클래스 파악**
   - 이 클래스를 먼저 인터페이스로 추상화하면 영향 범위 최대

3. **가장 단순한 외부 의존성부터 Port로 추출**
   - 이메일 발송 (인터페이스 추출 쉬움)
   - 결제 SDK (두 번째)
   - 나중에 Repository

4. **Strangler Fig 패턴 적용**
   - 기존 코드 유지하면서 새 기능은 Hexagonal로 작성
   - 점진적으로 기존 코드를 Hexagonal로 전환

"전체를 한 번에 바꾸자"는 계획은 대부분 실패합니다. 2-3주가 걸리다 포기하면 혼재 상태가 더 나빠집니다.

</details>

---

**Q2.** 팀원 중 한 명이 "Hexagonal Architecture는 너무 복잡해서 우리 팀에는 안 맞는다"고 한다. 어떻게 설득할 것인가? 또는 설득하지 않아야 하는 경우는 언제인가?

<details>
<summary>해설 보기</summary>

**설득이 필요한 경우와 불필요한 경우를 먼저 구분합니다.**

**설득하지 않아야 하는 경우:**
- 실제로 도메인이 단순하고 CRUD 위주일 때
- 프로젝트 기간이 3개월 이내일 때
- 팀 전체가 동의하지 않는 상태에서 혼자 적용하는 것은 더 나쁨

**설득할 수 있는 방법:**
1. **테스트 속도로 설득하기**
   - "@SpringBootTest 30초 vs 순수 Java 테스트 0.01초"
   - 100개 테스트 기준 42분 vs 1초의 차이를 직접 보여줌

2. **구체적인 사례로 설득하기**
   - "지난 달에 결제 수단 바꿀 때 Service 코드 얼마나 건드렸나?"
   - Hexagonal이었다면 Adapter 파일 하나만 추가했을 것

3. **전체가 아닌 부분 적용 제안**
   - "결제 연동 부분만 Port/Adapter로 해보자"
   - 작은 성공 경험 후 팀이 스스로 확장

**팀 동의 없이 혼자 적용하면:**
- 다른 팀원이 이해 못 하고 기존 방식으로 계속 코드 추가
- 레이어드 + Hexagonal 혼재 → 최악의 상태
- 팀 합의가 없으면 아키텍처 원칙이 코드에서 사라짐

결론: 기술적 설득보다 팀 전체의 공감이 더 중요합니다.

</details>

---

**Q3.** Woowacourse 우테코에서 진행 중인 Janggi(장기) 미션과 같은 프로젝트에서는 어떤 아키텍처가 적합한가? 교육 목적 코드와 실제 서비스 코드의 아키텍처 선택 기준이 다른 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

**교육 목적 코드에서의 아키텍처:**

Janggi(장기) 미션 같은 경우:
- 외부 시스템 없음 (DB, API 등)
- 콘솔 I/O 또는 간단한 UI
- 도메인 로직이 핵심 (기물 이동 규칙, 체크 판단 등)

이 경우 적합한 구조:
```
domain/
  Board.java
  Piece.java (+ 하위 기물들)
  Position.java
  GameState.java

application/
  JanggiGame.java (게임 진행 조율)

view/
  InputView.java
  OutputView.java
```

이것은 레이어드와 비슷하지만 핵심 원칙이 있습니다:
- **도메인이 View를 모름** (JanggiGame이 InputView를 직접 호출 X)
- **도메인이 UI 프레임워크를 모름** (순수 Java)
- **도메인 단위 테스트 가능** (Board, Piece 등 Spring 없이 테스트)

**교육 vs 실제 서비스의 차이:**
- 교육: 원칙 학습이 목적 → 가능한 단순한 구조에서 원칙 적용
- 실제 서비스: 비즈니스 가치 전달이 목적 → 도메인 복잡도에 맞는 구조

교육 목적에서도 핵심 원칙 (DIP, SRP)은 동일하게 적용됩니다. 다만 Port/Adapter를 구체적으로 만들 필요는 없고, "도메인이 인프라를 모르게"라는 원칙을 지키는 것이 핵심입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 전통적 레이어드 아키텍처의 문제](./04-layered-architecture-problems.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 레이어드 아키텍처 완전 분해 ➡️](../layered-architecture/01-layered-principles.md)**

</div>
