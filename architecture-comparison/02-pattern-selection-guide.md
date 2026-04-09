# 패턴 선택 기준 — 도메인 복잡도와 팀 상황에 따른 가이드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 단순 CRUD와 복잡한 비즈니스 규칙 기준으로 어떤 패턴을 선택해야 하는가?
- 팀 규모와 학습 곡선을 고려한 현실적 선택 기준은?
- 외부 시스템 교체 가능성 요구가 패턴 선택에 어떤 영향을 주는가?
- 테스트 자동화 성숙도가 낮을 때 어떤 패턴이 더 유리한가?
- 과잉 엔지니어링의 구체적인 징후는 무엇인가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

"어느 아키텍처가 더 좋은가"에는 답이 없다. 상황에 따라 다르다. 이 문서는 "우리 상황에 어떤 것이 맞는가"를 결정하는 구체적인 기준을 제공한다.

---

## 😱 흔한 실수 (Before — 기준 없이 선택할 때)

```
기준 없는 선택의 결과:

과잉 엔지니어링:
  사내 공지사항 CRUD (기능: 등록/수정/삭제/조회)
  → Hexagonal 완전 적용
  → Port 8개, Adapter 10개, Mapper 4개
  → "글 하나 등록하는 데 파일 20개?"

과소 엔지니어링:
  이커머스 주문/결제/배송 서비스 (복잡한 비즈니스 규칙)
  → 기본 레이어드
  → 6개월 후 OrderService 1200줄
  → "이 코드 건드리면 배포 못 하겠다"
```

---

## ✨ 올바른 접근 (After — 4가지 기준으로 결정)

```
패턴 선택 기준 4가지:

기준 1: 도메인 복잡도
  낮음: 단순 CRUD, 비즈니스 규칙 5개 이하 → 레이어드(DIP 개선)
  중간: 복잡한 규칙, 상태 전이, 외부 연동 2-4개 → Hexagonal 시작
  높음: 도메인 전문 지식, 외부 연동 5+개 → Hexagonal or Clean

기준 2: 외부 시스템 교체 가능성
  없음: 외부 연동 없거나 교체 가능성 없음 → 레이어드
  있음: 결제/알림/검색 교체 예상 → Hexagonal (Port로 추상화)

기준 3: 테스트 자동화 목표
  없음 or 낮음: 수동 테스트 위주 → 레이어드 (복잡도 줄이기)
  높음: CI 5분 이내, TDD → Hexagonal 필수

기준 4: 팀 역량
  Hexagonal 경험 없음 → 레이어드+DIP 시작 후 점진적
  Hexagonal 경험 있음 → 처음부터 Hexagonal
  Clean Architecture 경험 있음 → 대규모 팀에서 Clean
```

---

## 🔬 내부 원리 — 기준별 상세 분석

### 1. 도메인 복잡도 측정 방법

```
복잡도 측정 체크리스트:

낮음 (레이어드 충분):
  □ CRUD 작업이 90% 이상
  □ 상태 전이 없거나 2단계 이하 (예: 활성/비활성)
  □ 비즈니스 검증: "빈 값 확인", "최대 길이 확인" 수준
  □ 외부 연동: 없거나 교체 가능성 0%
  □ 도메인 전문 지식 불필요 (개발자가 보면 즉시 이해)

중간 (Hexagonal 권장):
  □ 복잡한 상태 전이 (주문: DRAFT→PLACED→PAYMENT→SHIPPING→DELIVERED)
  □ 비즈니스 규칙 10개 이상 (최소 금액, 할인 정책, 재고 확인 등)
  □ 외부 연동 2-4개 (결제, 알림, 재고)
  □ 도메인 전문 지식 필요 (할인 계산 방식, 반품 정책 등)

높음 (Hexagonal or Clean 필수):
  □ 복잡한 도메인 규칙 (보험, 금융, 의료)
  □ 외부 연동 5개 이상
  □ 도메인 전문가 협업 (DDD 필요)
  □ 잦은 비즈니스 규칙 변경 (주 1-2회)
  □ 팀이 6명 이상이고 동시 개발

수치 기준:
  비즈니스 메서드(if문 포함)가 Entity당 5개 이하 → 낮음
  Entity당 5-15개 → 중간
  Entity당 15개 이상 → 높음
```

### 2. 외부 시스템 교체 가능성 평가

```
교체 가능성 높음 → Hexagonal의 Port 추상화 권장:

자주 교체되는 것들:
  결제 수단: KakaoPay ↔ TossPay ↔ NaverPay
    → PaymentPort 추상화 ROI 높음

  SMS/이메일 발송: AWS SES ↔ 네이버 클라우드 ↔ SendGrid
    → NotificationPort 추상화 ROI 높음

  검색 엔진: MySQL FULLTEXT ↔ Elasticsearch
    → SearchPort 추상화 ROI 높음

교체 가능성 낮음 → 추상화 과잉:
  사내 레거시 시스템 (절대 안 바뀜)
    → 직접 의존도 OK

  회사 표준 메시지 브로커 (Kafka, 전사 표준)
    → KafkaTemplate 직접 사용도 허용

판단 공식:
  "이 외부 시스템이 3년 내 교체될 가능성이 30% 이상인가?"
  YES → Port 추상화
  NO  → 직접 사용 허용
```

### 3. 테스트 자동화 성숙도와 패턴 선택

```
테스트 자동화 성숙도 Level별 권장 패턴:

Level 0: 테스트 없음
  "QA팀이 수동 테스트"
  → 레이어드 기본 (복잡도를 낮추는 것이 우선)
  → Hexagonal 도입 전에 테스트 문화 먼저 구축

Level 1: E2E 테스트만 (@SpringBootTest)
  → 레이어드+DIP (일부 단위 테스트 가능하게)
  → Hexagonal 도입 시 InMemory 패턴 바로 적용

Level 2: 단위 + 통합 테스트 혼재
  → Hexagonal 도입 (InMemory로 단위 테스트 비율 높이기)
  → 목표: 단위 70%, 통합 25%, E2E 5%

Level 3: TDD, 단위 70%+
  → Hexagonal or Clean 완전 적용
  → 아키텍처가 테스트 속도를 최대화하는 방향

주의:
  Hexagonal 도입이 테스트 속도를 자동으로 개선하지 않음
  → InMemory Adapter를 실제로 작성하고 단위 테스트를 짜야 이익
  → "Hexagonal 구조만 갖추고 @SpringBootTest를 계속 쓰면" 이익 없음
```

### 4. 과잉 엔지니어링의 5가지 징후

```
징후 1: 비즈니스 메서드 없는 Port
  public interface NoticeRepository {
      void save(Notice notice);
      Optional<Notice> findById(Long id);
  }
  // Notice는 제목, 내용, 날짜만 있는 단순 CRUD
  // 비즈니스 규칙 없음 → Port 추상화 이익 없음
  → 레이어드 DIP 수준으로 충분

징후 2: 매핑 코드 > 비즈니스 코드
  // 매핑:
  public Notice toDomain(NoticeJpaEntity entity) { ... } // 30줄
  public NoticeJpaEntity toEntity(Notice notice) { ... } // 30줄
  // 비즈니스 로직:
  public void activate() { this.active = true; } // 1줄

징후 3: 팀이 "새 기능 어디에 넣어요?" 자주 물어봄
  → 아키텍처가 복잡해서 위치 기준이 불명확
  → 단순화 필요

징후 4: 단일 Adapter가 있는 Port
  PaymentPort ← KakaoPayAdapter (유일한 구현체, 교체 계획 없음)
  → 실제 교체 가능성 없는 Port는 과잉

징후 5: 인터페이스가 클래스보다 많음
  PlaceOrderUseCase (인터페이스) + PlaceOrderService (구현체)
  CancelOrderUseCase (인터페이스) + CancelOrderService (구현체)
  // UseCase 인터페이스가 단 하나의 구현체만 있고 교체 계획 없음
  → 단순화: UseCase 인터페이스 제거, Service 직접 호출
```

---

## 💻 실전 코드 — 복잡도 수준별 구현 예시

```java
// === 낮은 복잡도: 레이어드 DIP ===
// 공지사항 CRUD

@Service
public class NoticeService {
    private final NoticeRepository noticeRepository; // 인터페이스 의존 (DIP 최소 적용)

    public NoticeResponse createNotice(CreateNoticeRequest req) {
        Notice notice = new Notice(req.getTitle(), req.getContent());
        noticeRepository.save(notice);
        return NoticeResponse.from(notice);
    }
    // 비즈니스 로직: 거의 없음 → Hexagonal 불필요
}

// === 중간 복잡도: Hexagonal 시작 ===
// 이커머스 주문 (핵심 도메인만)

@Service
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderSavePort orderSavePort; // Driven Port
    private final PaymentPort paymentPort;     // Driven Port (교체 예상)

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.userId(), command.lines());
        order.place(); // 복잡한 비즈니스 규칙 (Domain에)

        PaymentResult payment = paymentPort.charge(PaymentRequest.of(order));
        order.confirmPayment(payment.transactionId());

        orderSavePort.save(order);
        return order.getId();
    }
    // 결제 수단 교체: PaymentPort 구현체만 교체
}

// === 높은 복잡도: Clean Architecture ===
// 보험 계약 (매우 복잡한 도메인)

@Service @Transactional
public class CreateContractInteractor implements CreateContractInputBoundary {

    private final ContractGateway contractGateway;
    private final RiskAssessmentGateway riskGateway;
    private final PremiumCalculationGateway premiumGateway;
    // ...

    public void execute(CreateContractRequestModel req, CreateContractOutputBoundary out) {
        // 복잡한 보험 도메인 로직 조율
        RiskScore risk = riskGateway.assess(RiskRequest.from(req));
        InsuranceContract contract = InsuranceContract.create(req, risk);
        contract.calculatePremium(premiumGateway.getPremiumTable());
        contract.validateUnderwritingRules();
        contractGateway.save(contract);
        out.present(CreateContractResponseModel.from(contract));
    }
}
```

---

## 📊 패턴 비교 — 상황별 권장 패턴

```
도메인 복잡도 × 팀 규모 × 외부 연동 조합별 권장:

                    외부 연동 없음       외부 연동 1-3개    외부 연동 4개+
                  ┌─────────────────┬────────────────┬──────────────────┐
도메인 낮음(CRUD) │ 레이어드 기본    │ 레이어드+DIP    │ 레이어드+DIP     │
                  ├─────────────────┼────────────────┼──────────────────┤
도메인 중간       │ 레이어드+DIP     │ Hexagonal 시작  │ Hexagonal        │
                  ├─────────────────┼────────────────┼──────────────────┤
도메인 높음       │ Hexagonal        │ Hexagonal       │ Hexagonal/Clean  │
                  └─────────────────┴────────────────┴──────────────────┘
```

---

## ⚖️ 트레이드오프

```
선택 오류의 비용:

레이어드를 써야 할 곳에 Hexagonal:
  단순 CRUD에 18개 파일
  팀원 혼란, 개발 속도 50% 감소
  "너무 복잡하다"는 팀 불만 → 규칙 무시 → 혼재

Hexagonal을 써야 할 곳에 레이어드:
  6개월 후 Fat Service
  테스트 없음 → 배포 공포
  신규 기능 추가 시간 점점 증가

현실적 접근:
  처음에는 레이어드+DIP 시작
  → 복잡도가 임계점을 넘으면 점진적 Hexagonal 전환
  → 핵심 도메인부터 선택적 적용
  → 전체 전환보다 부분 적용이 현실적
```

---

## 📌 핵심 정리

```
패턴 선택 4가지 기준:

① 도메인 복잡도
  낮음 → 레이어드 / 중간 → Hexagonal / 높음 → Hexagonal or Clean

② 외부 시스템 교체 가능성
  30%+ → Hexagonal (Port 추상화)
  낮음 → 레이어드로 충분

③ 테스트 자동화 목표
  CI 5분 이내 목표 → Hexagonal 필수
  수동 테스트 위주 → 패턴보다 문화 먼저

④ 팀 역량
  Hexagonal 경험 없음 → 레이어드+DIP 시작
  경험 있음 → 처음부터 Hexagonal

과잉 엔지니어링 방지:
  "두 번째 구현체가 생길 때 인터페이스 추출"
  "교체 가능성 30% 미만이면 Port 불필요"
  "비즈니스 메서드 없는 Entity에 Hexagonal 불필요"
```

---

## 🤔 생각해볼 문제

**Q1.** "도메인 복잡도가 높아지면 Hexagonal로 전환"은 언제 해야 하는가? 구체적인 임계점 신호는?

<details>
<summary>해설 보기</summary>

**임계점 신호 5가지:**

1. Service 클래스가 300줄을 초과하기 시작할 때
2. "@SpringBootTest 없이 Service 테스트가 불가능하다"는 말이 나올 때
3. "외부 API를 바꾸려면 Service 코드를 고쳐야 해서 영향 범위를 모르겠다"
4. 새 기능 추가 시 기존 테스트가 5개 이상 깨질 때
5. "이 코드 건드리면 어디서 터질지 모르겠다"는 말이 2명 이상에게서 나올 때

이 신호들이 보이면 전면 전환보다 핵심 Pain Point(결제, 재고 등)부터 부분 전환을 시작합니다.

</details>

---

**Q2.** 스타트업처럼 빠른 개발이 필요한 환경에서 Hexagonal Architecture를 선택하면 정말로 불리한가?

<details>
<summary>해설 보기</summary>

**초기에는 불리하지만 6개월 이후부터는 역전됩니다.**

초기(1-3개월): 레이어드가 30-50% 빠름 (파일 수 적음, 학습 불필요)

6개월 후: Hexagonal이 같은 속도
- 레이어드: Fat Service 시작, 테스트 없음, 기능 추가 느려짐
- Hexagonal: 구조 안정, InMemory 테스트, 기능 추가 일정

1년 후: Hexagonal이 빠름
- 레이어드: "이 코드 리팩터링해야 하는데 시간이 없다" 루프
- Hexagonal: 비즈니스 규칙이 Domain에 명확, 새 기능 = Adapter 추가

스타트업 현실적 선택:
- MVP(2주): 레이어드 기본 (버릴 코드)
- 2차(2개월): 레이어드+DIP (최소한의 구조)
- PMF 이후: 핵심 도메인부터 Hexagonal 전환

</details>

---

**Q3.** 팀의 절반은 Hexagonal 경험이 있고 절반은 없을 때 어떻게 아키텍처를 도입하는가?

<details>
<summary>해설 보기</summary>

**페어 프로그래밍 + 점진적 도입 + ArchUnit 강제 순서로 합니다.**

1. **ArchUnit 먼저 설정** (1일): "domain 패키지가 infrastructure를 import하면 빌드 실패" → 규칙을 코드로 먼저 강제

2. **예제 모듈 생성** (1주): 경험자가 Order 도메인을 Hexagonal로 구현한 레퍼런스 코드 작성

3. **페어 프로그래밍** (2주): 경험자 + 미경험자 짝을 이루어 Hexagonal 기능 함께 개발

4. **코드 리뷰 체크리스트** (지속): "Port가 domain 패키지에 있는가?", "Adapter에 비즈니스 로직이 없는가?"

5. **레트로스펙티브** (격주): "무엇이 이해가 안 됐나?" 피드백으로 지식 격차 해소

가장 중요한 것: ArchUnit 자동화 검증. 사람이 코드 리뷰에서 놓치는 것을 CI가 잡아줍니다.

</details>

---

<div align="center">

**[⬅️ 이전: Layered vs Hexagonal vs Clean 비교](./01-layered-vs-hexagonal-vs-clean.md)** | **[홈으로 🏠](../README.md)** | **[다음: 혼합 아키텍처 ➡️](./03-mixed-architecture.md)**

</div>
