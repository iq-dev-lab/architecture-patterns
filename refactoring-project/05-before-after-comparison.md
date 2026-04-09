# 전후 비교 — 의존성·테스트 속도·변경 범위

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 리팩터링 전후 의존성 다이어그램은 어떻게 달라졌는가?
- `@SpringBootTest` 필요 테스트 비율이 전: 70% → 후: 20%로 바뀐 이유는?
- 새 기능 추가 시 변경 파일 범위는 얼마나 줄었는가?
- JPA → MongoDB 교체 시 변경 파일 수는 얼마나 차이나는가?
- 실제 팀이 리팩터링 후 경험한 현실적 장단점은 무엇인가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

리팩터링의 성과를 수치로 측정하지 않으면 "코드가 더 깔끔해졌다"는 주관적 평가만 남는다. 이 문서는 의존성 그래프, 테스트 실행 시간, 변경 파급 범위를 Before/After로 정량 비교한다.

이 비교가 중요한 이유는 두 가지다. 첫째, 리팩터링의 실제 가치를 팀에 명확히 보여줄 수 있다. 둘째, "어느 지표가 좋아졌고 어느 지표는 비용이 됐는가"를 솔직하게 파악해야 다음 프로젝트에서 더 나은 결정을 내릴 수 있다.

---

## 😱 흔한 실수 (Before — 리팩터링 전 상태 기록 없이 시작)

```
리팩터링 전 측정을 빠뜨리면:

"리팩터링 완료했습니다. 훨씬 좋아졌어요!"
→ "얼마나 좋아졌나요?"
→ "... 체감상으로는 많이요"

→ 팀장: "3개월을 썼는데 실제로 뭐가 좋아진 건지 수치로 보여줄 수 있나요?"
→ 측정 안 했기 때문에 증명 불가

리팩터링 시작 전 반드시 측정:
  ① 테스트 실행 시간 (./gradlew test 타이머 측정)
  ② @SpringBootTest 비율 (테스트 파일 grep)
  ③ 핵심 클래스의 의존성 수 (Fan-out)
  ④ 변경 파급 범위 (새 기능 추가 시 수정 파일 수)
  ⑤ 클래스 크기 (wc -l 또는 IntelliJ Metrics)

리팩터링 후 동일 지표 재측정 → 숫자로 비교
```

---

## ✨ 올바른 접근 (After — 수치로 Before/After 비교)

```
핵심 지표 Before/After 요약:

지표                         │ Before    │ After     │ 변화
────────────────────────────┼──────────┼──────────┼──────────────
테스트 전체 실행 시간        │ 43분 12초 │ 4분 18초  │ 10배 빠름
@SpringBootTest 비율        │ 70%       │ 18%       │ -52%p
단위 테스트 비율             │ 8%        │ 72%       │ +64%p
OrderService 줄 수          │ 1,247줄   │ 198줄     │ -84%
OrderService 의존성 수(Ce)  │ 8개       │ 3개       │ -62%
새 기능 추가 시 변경 파일    │ 평균 6개  │ 평균 2-3개│ -50%
JPA→MongoDB 교체 변경 파일  │ 12개      │ 2개       │ -83%
신규 팀원 온보딩 시간        │ 4주       │ 2주       │ -50%
```

---

## 🔬 내부 원리 — 상세 Before/After 비교

### 1. 의존성 다이어그램 비교

```
=== BEFORE: 레이어드 Fat Service ===

  OrderController
        │
        ↓ (직접 의존)
  OrderService (1,247줄, 27개 메서드)
        │
        ├──→ OrderJpaRepository (Spring Data)
        ├──→ UserRepository (JPA)
        ├──→ CouponRepository (JPA)
        ├──→ InventoryRepository (JPA)
        ├──→ PaymentClient (외부 SDK)
        ├──→ KafkaTemplate (Kafka 인프라)
        ├──→ JavaMailSender (Spring 인프라)
        └──→ RedisTemplate (Redis 인프라)
                    ↓
           Order (@Entity, 비즈니스 로직 없음)

팬아웃(Fan-out): OrderService = 8개
팬인(Fan-in): OrderController, AdminController, BatchService = 3개
변경 위험도: Fan-out 8 × Fan-in 3 = 24 (높음)

=== AFTER: Hexagonal Architecture ===

  OrderController
        │
        ↓ (인터페이스 의존)
  PlaceOrderUseCase (인터페이스)
        ↑ implements
  PlaceOrderService (198줄, 1개 UseCase)
        │
        ├──→ OrderSavePort (인터페이스)
        │         ↑ implements
        │    JpaOrderRepository (Adapter)
        │         └──→ Spring Data JPA, MySQL
        │
        ├──→ PaymentPort (인터페이스)
        │         ↑ implements
        │    KakaoPayAdapter (Adapter)
        │         └──→ KakaoPay SDK
        │
        └──→ OrderEventPublisher (인터페이스)
                  ↑ implements
             KafkaOrderEventPublisher (Adapter)
                  └──→ KafkaTemplate, Kafka

  Order (순수 Java, 비즈니스 메서드 15개)

팬아웃(Fan-out): PlaceOrderService = 3개 (Port 인터페이스만)
팬인(Fan-in): OrderController = 1개
변경 위험도: Fan-out 3 × Fan-in 1 = 3 (낮음)

의존성 방향: 모두 PlaceOrderService를 향함
  → PlaceOrderService가 JPA, Kafka를 모름
  → JPA, Kafka가 PlaceOrderService(의 Port)를 모름
```

### 2. 테스트 피라미드 변화

```
=== BEFORE: 역전된 피라미드 ===

       ████████  E2E/통합 (@SpringBootTest)  70%
      ██████      통합 (@DataJpaTest)         22%
    ██            단위 테스트                   8%

테스트 100개 기준 실행 시간:
  @SpringBootTest 70개 × 30초 = 35분
  @DataJpaTest   22개 × 8초  = 2분 56초
  단위 테스트    8개 × 0.1초 = 0.8초
  합계: ~38분

문제:
  PR 1번에 CI 38분 대기 → 하루 2회 PR 가능
  "테스트 실패했는데 원인 찾기 어렵다" (Spring Context 전체)
  "새 기능 테스트 추가가 두렵다" (30초짜리)

=== AFTER: 건강한 피라미드 ===

     ██            E2E (@SpringBootTest)       8%
    ████            통합 (@DataJpaTest)        20%
   ██████████████  단위 테스트                 72%

테스트 100개 기준 실행 시간:
  단위 테스트    72개 × 0.01초 = 0.72초
  @DataJpaTest  20개 × 5초   = 100초
  @SpringBootTest 8개 × 30초 = 240초
  합계: ~341초 (5분 41초)

변화:
  38분 → 5분 41초 (6.7배 빠름)
  "테스트 실패 → 원인이 명확한 단위 테스트" (빠른 피드백)
  "새 기능 테스트: Order 단위 + Service 단위" (0.01초)
  PR당 CI 대기: 38분 → 5분 → 하루 10회 이상 PR 가능
```

### 3. 새 기능 추가 시 변경 범위 비교

```
시나리오: "주문에 쿠폰 적용 기능 추가"

=== BEFORE (레이어드) ===

변경 파일:
  controller/OrderController.java       ← HTTP 파라미터 추가
  service/OrderService.java             ← 쿠폰 적용 비즈니스 로직 추가
  repository/CouponRepository.java      ← (신규 또는 수정)
  entity/Order.java                     ← couponId 필드 추가
  entity/Coupon.java                    ← (신규)
  dto/PlaceOrderRequest.java            ← couponCode 필드 추가
  dto/OrderResponse.java                ← 할인 금액 필드 추가

변경 파일 수: 7개, 4개 패키지
테스트 추가:  @SpringBootTest 1개 (30초)

의도치 않은 영향:
  OrderService.java 수정 → 기존 27개 메서드 컨텍스트에서 작업
  실수로 다른 기능 로직 건드릴 가능성

=== AFTER (Hexagonal) ===

변경 파일:
  order/domain/model/Order.java         ← applyCoupon() 메서드 추가
  order/domain/port/out/CouponPort.java ← (신규 Port)
  order/application/service/PlaceOrderService.java ← 쿠폰 적용 흐름 추가
  order/adapter/in/web/request/PlaceOrderRequest.java ← couponCode 추가
  order/adapter/in/web/mapper/OrderWebMapper.java     ← 매핑 추가

변경 파일 수: 5개, 모두 order/ 패키지 내부
신규 파일:
  coupon/domain/... ← Coupon Bounded Context (별도)
  order/adapter/out/coupon/CouponServiceAdapter.java ← Coupon Port 구현

테스트 추가:
  Order 단위 테스트: applyCoupon() 테스트 (0.001초)
  PlaceOrderService 단위 테스트: InMemoryCouponPort로 (0.01초)
  JpaOrderRepository 통합 테스트: couponId 컬럼 매핑 (5초)

장점:
  PlaceOrderService 수정 → 다른 UseCase 파일에 영향 없음
  Order 도메인 객체 수정 → 단위 테스트로 즉시 검증
  변경 범위가 order/ 패키지에 집중

변경 범위 비교:
  Before: 7개 파일, 4개 패키지 분산
  After:  5개 파일, order/ 패키지 집중 (+ coupon/ 신규)
  → 파일 수 30% 감소 + 변경 집중도 향상
```

### 4. 인프라 교체 시 변경 범위 비교

```
시나리오: "MySQL JPA → MongoDB 전환"

=== BEFORE (레이어드) ===

변경 필요 파일:
  entity/Order.java           ← @Entity 제거, @Document 추가
  entity/OrderLine.java       ← 동일
  entity/Coupon.java          ← 동일
  repository/OrderRepository.java ← JpaRepository → MongoRepository
  repository/OrderLineRepository.java ← 동일
  service/OrderService.java   ← @Transactional 방식 변경 (JPA → Mongo)
  config/JpaConfig.java       ← (삭제)
  config/MongoConfig.java     ← (신규)
  + 모든 JPQL 쿼리를 MongoDB 쿼리로 변환

변경 파일 수: 12개 이상
위험도: OrderService 변경 → 비즈니스 로직 실수 가능

=== AFTER (Hexagonal) ===

변경 필요 파일:
  adapter/out/persistence/MongoOrderRepository.java ← (신규, implements OrderSavePort, OrderQueryPort)
  adapter/out/persistence/entity/OrderDocument.java ← (신규 MongoDB 문서)
  adapter/out/persistence/mapper/OrderMongoMapper.java ← (신규)
  config/MongoConfig.java ← (신규)

삭제 파일:
  adapter/out/persistence/JpaOrderRepository.java
  adapter/out/persistence/entity/OrderJpaEntity.java
  adapter/out/persistence/mapper/OrderPersistenceMapper.java
  config/JpaConfig.java

변경 불필요:
  Order.java (Domain) ← 변경 없음 (MongoDB 모름)
  PlaceOrderService.java ← 변경 없음 (Port만 앎)
  OrderController.java ← 변경 없음 (UseCase만 앎)
  모든 비즈니스 로직 ← 변경 없음

실제 변경 파일 수: 신규 4개 + 삭제 4개 = 영향 8개
비즈니스 로직 변경: 0개
위험도: 낮음 (비즈니스 코드 무관)

변경 범위 비교:
  Before: 12개 변경 (비즈니스 로직 포함)
  After:  2개 신규 + Adapter 교체 (비즈니스 로직 무관)
  → 교체 비용 83% 감소 + 위험도 대폭 감소
```

### 5. 팀 소감과 현실적 주의사항

```
=== 팀 소감 (가상의 리팩터링 팀 피드백) ===

좋았던 점:
  개발자 A: "Domain 단위 테스트가 0.001초라서 TDD를 처음으로 실제로 할 수 있었어요.
             비즈니스 로직 변경할 때 즉각 피드백이 너무 좋아요."

  개발자 B: "결제 수단 변경 요청이 왔을 때 KakaoPayAdapter만 교체했어요.
             PlaceOrderService는 건드리지 않아도 됐고, 테스트도 바로 통과했어요."

  개발자 C: "신규 팀원이 합류했을 때 domain/port/in/과 domain/port/out/을 보여주면서
             '이 인터페이스들이 기능의 입출력 계약이에요'라고 설명했더니
             일주일 만에 적응했어요."

어려웠던 점:
  개발자 D: "Domain Entity와 JPA Entity를 분리해서 Mapper를 작성하는 게
             처음에는 굉장히 번거로웠어요. MapStruct 학습이 필요했어요."

  개발자 E: "Port가 너무 많아지는 것 같아요. 작은 기능에도 Port/Adapter 구조를
             만드니까 파일이 레이어드 때보다 3배 많아졌어요."

  개발자 F: "레거시에서 전환하는 3개월 동안 두 가지 스타일이 혼재해서
             '이 부분은 어느 스타일이에요?' 혼란이 있었어요."

=== 현실적 주의사항 ===

① 팀 전체가 동의하지 않으면 성공 어려움
   "나 혼자 Hexagonal로 바꾸는 중"이면 혼재 상태가 됨
   → 팀 합의 → ADR 작성 → ArchUnit 강제 순서로 진행

② 과잉 적용 방지
   모든 기능에 Port/Adapter를 강제하면 간단한 기능도 복잡해짐
   → 단순 CRUD(공지사항 등)은 레이어드+DIP로 충분

③ 매핑 코드 비용 감수
   Domain Entity ↔ JPA Entity 매핑은 피할 수 없음
   → MapStruct로 자동화, 이 비용이 장기 이익보다 작다고 팀이 동의해야

④ 처음 2-3개월은 개발 속도 감소
   새 구조에 익숙해지는 데 시간이 걸림
   → 팀이 이 기간을 감수할 준비가 됐을 때 도입

⑤ InMemory ≠ 완전한 테스트
   InMemory가 통과해도 JPA Adapter 테스트는 별도 필요
   → 두 가지 테스트 전략을 병행
```

---

## 💻 실전 코드 — 리팩터링 성과 측정 스크립트

```bash
#!/bin/bash
# 리팩터링 전후 측정 스크립트

echo "=== 테스트 실행 시간 측정 ==="
time ./gradlew test --no-build-cache 2>&1 | tail -5

echo ""
echo "=== @SpringBootTest 비율 계산 ==="
TOTAL=$(find src/test -name "*.java" | xargs grep -l "@Test" | wc -l)
SPRINGBOOT=$(find src/test -name "*.java" | xargs grep -l "@SpringBootTest" | wc -l)
echo "전체 테스트 클래스: $TOTAL개"
echo "@SpringBootTest: $SPRINGBOOT개"
echo "비율: $(echo "scale=1; $SPRINGBOOT * 100 / $TOTAL" | bc)%"

echo ""
echo "=== 클래스 크기 (상위 10개) ==="
find src/main -name "*.java" -exec wc -l {} + | \
  sort -rn | head -11 | grep -v "total"

echo ""
echo "=== 핵심 Service 의존성 수 ==="
echo "PlaceOrderService imports:"
grep "^import" src/main/java/**/PlaceOrderService.java | \
  grep -v "java.util\|java.time\|java.math" | wc -l

# 결과 예시 (After):
# 테스트 실행 시간: 4분 18초 (Before: 43분)
# @SpringBootTest 비율: 18% (Before: 70%)
# 최대 클래스: PlaceOrderService.java 198줄 (Before: OrderService.java 1247줄)
# 외부 의존성: 3개 (Before: 8개)
```

---

## 📊 패턴 비교 — Before/After 핵심 수치 요약

```
전체 변화 요약:

테스트 속도:
  Before: 43분 12초
  After:  4분 18초
  개선:   10배 빠름 → PR당 대기 38분 → 4분

테스트 구성:
              Before    After
  단위 테스트   8%       72%
  @DataJpaTest 22%      20%
  @SpringBootTest 70%   8%

코드 구조:
              Before    After
  최대 클래스  1,247줄  198줄
  최대 Fan-out  8개      3개
  Port 인터페이스 없음   15개

변경 비용:
  새 기능 추가: 7개 파일 → 3-5개 파일
  JPA→MongoDB: 12개 변경 → 2개 신규 Adapter
  결제 교체: 전체 Service 수정 → Adapter 1개 교체

비용:
  파일 수: 35개 → 78개 (2.2배 증가)
  초기 학습: 2-4주
  매핑 코드: 신규 추가 (~600줄, MapStruct로 절반 자동화)
```

---

## ⚖️ 트레이드오프

```
리팩터링이 해결한 것:
  ✅ 테스트 실행 시간 10배 단축
  ✅ 비즈니스 로직이 Domain에 집중 → 찾기 쉬움
  ✅ 외부 시스템 교체 비용 대폭 감소
  ✅ 코드 변경 시 의도치 않은 파급 감소

리팩터링이 해결하지 못한/새로 생긴 것:
  ⚠️ 파일 수 2배 이상 증가 → 탐색 비용
  ⚠️ 초기 2-3개월 개발 속도 감소 (새 구조 적응)
  ⚠️ 매핑 코드(Mapper) 관리 필요
  ⚠️ 팀 전체가 Port/Adapter 개념을 이해해야 함

판단 기준:
  "이 비용을 팀이 6개월 후 회수할 수 있는가?"
  
  회수 가능한 경우:
    도메인이 복잡하고 계속 성장 중
    팀이 3명 이상이고 안정적
    테스트 자동화에 투자 의향 있음

  회수 어려운 경우:
    2주 이내 폐기 예정
    단순 CRUD만 존재
    팀이 1-2명이고 빠른 개발이 최우선
```

---

## 📌 핵심 정리

```
리팩터링 전후 핵심 수치:

테스트 속도: 43분 → 4분 (10배)
@SpringBootTest 비율: 70% → 18%
최대 Service 크기: 1,247줄 → 198줄
새 기능 변경 파일: 7개 → 3-5개
인프라 교체 변경: 12개 → 2개

의존성 변화:
  Before: OrderService → JPA, Kafka, 결제SDK, 이메일, Redis (8개)
  After:  PlaceOrderService → Port 인터페이스만 (3개)
  → 의존 방향: 모두 Domain을 향함

현실적 조언:
  팀 합의 없이 시작하지 말 것 (혼재 상태 최악)
  단순 CRUD에 Hexagonal 강제하지 말 것 (과잉)
  InMemory Adapter 테스트 반드시 작성할 것 (핵심 이익)
  MapStruct로 매핑 비용 절감할 것
  월 1회 아키텍처 리뷰로 일관성 유지할 것

리팩터링 성공 기준:
  ① 단위 테스트 비율 70% 이상
  ② CI 실행 5분 이내
  ③ 새 기능 추가 시 기존 UseCase 파일 수정 없음
  ④ 외부 시스템 교체 시 Domain/Application 코드 변경 없음
```

---

## 🤔 생각해볼 문제

**Q1.** "파일 수가 2배 이상 늘어서 오히려 코드 탐색이 어려워졌다"는 팀원의 불만에 어떻게 대응하는가?

<details>
<summary>해설 보기</summary>

**타당한 불만입니다.** 파일 수 증가는 실제 비용이며 부정할 수 없습니다.

대응 방법:

1. **의도 설명**: 파일 수가 늘었지만 각 파일의 책임이 명확해졌음
   - Before: OrderService.java 1,247줄 (무엇이든 다 있음)
   - After: PlaceOrderService.java 198줄 (주문 생성만), Order.java (비즈니스만)
   - "어디에 있는지 모름" → "정해진 위치 결정 트리로 바로 찾음"

2. **IDE 활용법 공유**:
   - Ctrl+Shift+F (전체 검색): "OrderSavePort"로 바로 찾기
   - Ctrl+B (구현체로 이동): Port 인터페이스 → Adapter 즉시 이동
   - IntelliJ의 "Package by Feature" 뷰로 기능별 묶음 탐색

3. **적응 기간 인정**: 새 구조 탐색 패턴이 익숙해지는 데 2-4주 소요 → 그 후에는 더 빠름

4. **수치 비교 제시**: "새 기능 추가 시 파일 찾는 시간"을 Before/After로 측정 → 수치가 반박보다 설득력 있음

</details>

---

**Q2.** 리팩터링 후 테스트 실행 시간이 10배 빨라졌는데, 이것이 실제 팀 생산성에 미치는 영향을 구체적으로 어떻게 측정하는가?

<details>
<summary>해설 보기</summary>

**직접 측정은 어렵지만 간접 지표로 추적할 수 있습니다.**

측정 가능한 지표들:
```
① PR 당 CI 대기 시간:
   Before: 38분 → After: 5분
   하루 PR 횟수: 2회 → 10회 가능 (이론적)

② 테스트 실패 후 수정-재실행 사이클:
   Before: 실패 발견 → 수정 → 38분 대기 → 결과
   After:  실패 발견 → 수정 → 5분 → 결과

③ 로컬 개발 중 테스트 실행 빈도:
   Before: "무거워서 한 번만" → CI에서만 실행
   After:  "가벼워서 자주" → TDD 실천 가능

④ 버그 발견 시점:
   Before: PR 머지 후 CI → 발견 지연
   After:  로컬 개발 중 단위 테스트 → 즉시 발견

⑤ 주관적 개발자 만족도 (설문):
   "테스트 작성이 두려운가?" Before: 70% Yes, After: 20% Yes
```

핵심: 테스트 속도는 "TDD 실천 가능 여부"를 결정합니다. 38분짜리 테스트로는 TDD가 불가능하고, 5분짜리는 가능합니다. 이 차이가 장기적으로 코드 품질과 개발 속도를 좌우합니다.

</details>

---

**Q3.** 이 리팩터링 경험을 바탕으로, 처음부터 새 프로젝트를 Hexagonal로 시작할 때 "레거시 리팩터링과 다르게 주의할 점"은 무엇인가?

<details>
<summary>해설 보기</summary>

**처음부터 시작할 때의 핵심 차이점:**

1. **도메인 설계를 먼저**: 레거시 전환은 기존 DB 스키마에 맞춰야 하지만, 새 프로젝트는 Domain 객체를 먼저 설계하고 DB 스키마가 따라감
   ```java
   // 처음부터: Domain 먼저
   Order order = Order.create(userId, lines);  // Domain 설계
   order.place();  // 비즈니스 메서드 먼저
   // JPA Entity는 나중에 (Domain을 기반으로)
   ```

2. **과잉 설계 경계**: 처음부터 모든 것에 Port/Adapter를 강제하고 싶은 유혹 → 단순 CRUD는 레이어드+DIP로 시작, 복잡해지면 전환

3. **InMemory Adapter 먼저**: 실제 JPA 구현 전에 InMemory Adapter로 비즈니스 로직 개발 → TDD 자연스러운 흐름
   ```
   InMemoryOrderRepository 작성 → PlaceOrderService 구현 → 단위 테스트 작성
   → 비즈니스 로직 완성 → JpaOrderRepository 구현 (나중에)
   ```

4. **ArchUnit 처음부터**: 첫 날부터 ArchUnit 규칙 설정 → 규칙 위반이 쌓이기 전에 차단

5. **팀 합의 문서화**: 첫 주에 ARCHITECTURE.md + ADR-001 작성 → 이후 모든 결정의 기반

레거시 전환의 가장 힘든 점(기존 코드와 공존)이 없으므로 처음부터가 훨씬 쉽습니다. 그러나 과잉 설계의 유혹을 경계하세요.

</details>

---

<div align="center">

**[⬅️ 이전: 인프라 레이어 분리](./04-extracting-infrastructure-layer.md)** | **[홈으로 🏠](../README.md)**

</div>
