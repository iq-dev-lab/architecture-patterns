# Hexagonal의 실제 비용 — 복잡도와 매핑 비용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 인터페이스 증가로 인한 실제 코드량과 파일 수는 얼마나 늘어나는가?
- 단순 CRUD에 Hexagonal을 적용했을 때 생기는 과잉 복잡도는 구체적으로 어떤 모습인가?
- Domain Entity와 JPA Entity를 분리할 때 발생하는 매핑 비용을 MapStruct로 어떻게 줄이는가?
- 팀 학습 비용과 신규 팀원 온보딩에 미치는 영향은 무엇인가?
- Hexagonal이 적합한 도메인 복잡도 기준은 어떻게 정하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Hexagonal Architecture의 이익만 보면 "항상 써야 한다"고 생각하기 쉽다. 하지만 모든 아키텍처 결정에는 비용이 따른다.

단순 CRUD API에 Port/Adapter/Mapper를 모두 적용하면 10줄짜리 기능이 50줄이 된다. 이 비용이 언제 정당화되고 언제 과잉 엔지니어링인지 판단하는 것이 실무 역량이다.

---

## 😱 흔한 실수 (Before — 단순 기능에 과잉 적용)

```
단순 공지사항 CRUD에 Hexagonal 전면 적용:

파일 목록:
  domain/port/in/
    CreateNoticeUseCase.java       ← 인터페이스
    FindNoticeQuery.java           ← 인터페이스
    UpdateNoticeUseCase.java       ← 인터페이스
    DeleteNoticeUseCase.java       ← 인터페이스
  domain/port/out/
    NoticeSavePort.java            ← 인터페이스
    NoticeQueryPort.java           ← 인터페이스
    NoticeDeletePort.java          ← 인터페이스
  domain/model/
    Notice.java                    ← 도메인 객체 (내용: 제목, 내용, 날짜)
    NoticeId.java                  ← Value Object
  application/service/
    CreateNoticeService.java       ← UseCase 구현
    FindNoticeService.java         ← UseCase 구현
    UpdateNoticeService.java       ← UseCase 구현
    DeleteNoticeService.java       ← UseCase 구현
  adapter/in/web/
    NoticeController.java          ← Driving Adapter
    dto/CreateNoticeRequest.java
    dto/UpdateNoticeRequest.java
    dto/NoticeResponse.java
    mapper/NoticeWebMapper.java
  adapter/out/persistence/
    JpaNoticeRepository.java       ← Driven Adapter
    entity/NoticeJpaEntity.java    ← JPA Entity
    mapper/NoticePersistenceMapper.java

총 파일 수: 18개 (비즈니스 로직: 사실상 없음)

레이어드 아키텍처로 같은 기능:
  NoticeController.java            ← 4개 엔드포인트
  NoticeService.java               ← CRUD 4개 메서드
  NoticeRepository.java            ← JpaRepository 상속

총 파일 수: 3개

결론: 비즈니스 로직이 없는 단순 CRUD에서 Hexagonal은 6배 많은 파일 = 과잉
```

---

## ✨ 올바른 접근 (After — 비용 대비 효과를 판단하여 적용)

```
비용 대비 효과 판단 기준:

Hexagonal이 비용보다 효과가 큰 경우:
  ① 비즈니스 규칙이 복잡한 도메인
     Order: 최소 금액, 할인 정책, 상태 전이, 취소 규칙 → 도메인 테스트 중요
  
  ② 외부 시스템 교체 가능성
     결제: KakaoPay ↔ TossPay → PaymentPort로 교체 비용 최소화

  ③ 빠른 단위 테스트가 중요한 경우
     복잡한 비즈니스 로직 → InMemory로 ms 단위 테스트

Hexagonal이 과잉 엔지니어링인 경우:
  ① 단순 CRUD (공지사항, 설정 관리 등)
     비즈니스 로직 없음 → 도메인 테스트할 것 없음 → 매핑 비용만 증가

  ② 읽기 전용 서비스
     복잡한 쓰기 로직 없음 → Adapter 교체 가능성 낮음

  ③ 2주 이내 폐기 예정 코드
     유지보수 필요 없음 → 아키텍처 투자 ROI 없음

현실적 전략: 선택적 적용
  복잡한 핵심 도메인(주문, 결제): Hexagonal 완전 적용
  단순 지원 기능(공지, 설정): 레이어드 + DIP 개선형
```

---

## 🔬 내부 원리 — 각 비용 항목 분석

### 1. 인터페이스 증가 비용 — 실제 파일 수 비교

```
같은 "주문 기능"을 각 아키텍처로 구현 시 파일 수:

레이어드 아키텍처 (기본):
  OrderController.java        (1)
  OrderService.java           (1)
  OrderRepository.java        (1, JpaRepository 상속)
  Order.java                  (1, @Entity)
  PlaceOrderRequest.java      (1)
  OrderResponse.java          (1)
  총: 6개

레이어드 아키텍처 (DIP 개선):
  OrderController.java        (1)
  PlaceOrderService.java      (1)
  OrderRepository.java        (1, 인터페이스)
  JpaOrderRepository.java     (1, 구현체)
  Order.java                  (1, @Entity 유지)
  PlaceOrderRequest.java      (1)
  OrderResponse.java          (1)
  총: 7개 (+1)

Hexagonal Architecture (기본):
  PlaceOrderUseCase.java      (1, Driving Port)
  PlaceOrderCommand.java      (1, UseCase 입력)
  OrderSavePort.java          (1, Driven Port)
  OrderQueryPort.java         (1, Driven Port)
  PaymentPort.java            (1, Driven Port)
  Order.java                  (1, 순수 도메인)
  OrderId.java                (1, Value Object)
  Money.java                  (1, Value Object)
  PlaceOrderService.java      (1, UseCase 구현)
  OrderController.java        (1, Driving Adapter)
  PlaceOrderRequest.java      (1, HTTP DTO)
  PlaceOrderResponse.java     (1, HTTP DTO)
  OrderWebMapper.java         (1, HTTP 매핑)
  JpaOrderRepository.java     (1, Driven Adapter)
  OrderJpaEntity.java         (1, JPA Entity)
  OrderPersistenceMapper.java (1, JPA 매핑)
  KakaoPayAdapter.java        (1, Driven Adapter)
  InMemoryOrderRepository.java(1, 테스트용)
  총: 18개 (+12)

파일 수 증가: 6개 → 18개 (3배)
하지만: 각 파일의 책임이 명확하고 크기가 작음
```

### 2. 매핑 비용 — Domain Entity ↔ JPA Entity 변환

```java
// 수동 매핑 (가장 명시적, 가장 많은 코드량)
@Component
public class OrderPersistenceMapper {

    public OrderJpaEntity toJpaEntity(Order domain) {
        return OrderJpaEntity.builder()
            .orderId(domain.getId().value())
            .userId(domain.getUserId().value())
            .status(domain.getStatus())
            .paymentTransactionId(domain.getPaymentTransactionId())
            .lines(
                domain.getLines().stream()
                    .map(line -> OrderLineJpaEntity.builder()
                        .itemId(line.getItemId().value())
                        .quantity(line.getQuantity())
                        .unitPrice(line.getUnitPrice().getValue())
                        .build()
                    )
                    .toList()
            )
            .build();
    }

    public Order toDomain(OrderJpaEntity entity) {
        return Order.reconstitute(
            OrderId.of(entity.getOrderId()),
            UserId.of(entity.getUserId()),
            entity.getLines().stream()
                .map(line -> OrderLine.of(
                    ItemId.of(line.getItemId()),
                    line.getQuantity(),
                    Money.of(line.getUnitPrice())
                ))
                .toList(),
            entity.getStatus(),
            entity.getPaymentTransactionId()
        );
    }
    // 약 40-50줄의 매핑 코드
}

// MapStruct로 자동 생성 (매핑 코드 대폭 감소)
@Mapper(
    componentModel = "spring",
    unmappedTargetPolicy = ReportingPolicy.ERROR
)
public interface OrderPersistenceMapper {

    @Mapping(source = "id.value", target = "orderId")
    @Mapping(source = "userId.value", target = "userId")
    OrderJpaEntity toJpaEntity(Order domain);

    @Mapping(source = "orderId", target = "id", qualifiedByName = "toOrderId")
    @Mapping(source = "userId", target = "userId", qualifiedByName = "toUserId")
    Order toDomain(OrderJpaEntity entity);

    @Named("toOrderId")
    default OrderId toOrderId(String value) { return OrderId.of(value); }

    @Named("toUserId")
    default UserId toUserId(String value) { return UserId.of(value); }
}
// 약 15줄로 동일한 매핑 (컴파일 타임 코드 생성)
// MapStruct가 컴파일 시 구현체를 자동 생성
```

### 3. 팀 학습 비용 — 개념별 학습 시간

```
Hexagonal 핵심 개념 학습 비용:

개념 1: Port의 의미 (Driving vs Driven)
  학습 시간: 2-4시간 (문서 읽기 + 코드 예시)
  난이도: 중간

개념 2: Adapter의 역할과 위치
  학습 시간: 1-2시간
  난이도: 낮음 (구체적 코드로 이해)

개념 3: 패키지 구조 규칙
  학습 시간: 30분 (한 번 보면 외워짐)
  난이도: 낮음

개념 4: 의존성 방향 규칙
  학습 시간: 2-4시간 (왜 그래야 하는지 납득 필요)
  난이도: 높음

개념 5: InMemory Adapter 작성
  학습 시간: 1시간 (처음 한 번 따라하면)
  난이도: 낮음

총 학습 비용: 7-11시간 (코드와 함께)

신규 팀원 온보딩:
  레이어드: 반나절 (대부분 이미 알고 있음)
  Hexagonal: 1-2주 (코드 리뷰로 패턴 내재화 필요)

학습 투자 회수:
  레이어드로 6개월 개발 후 리팩터링: 수 주
  Hexagonal 초기 학습: 2주
  → 복잡한 도메인이라면 Hexagonal 학습이 더 경제적
```

### 4. 단순 CRUD vs 복잡한 도메인 — 비용 대비 효과

```
비용 대비 효과 매트릭스:

도메인: 공지사항 CRUD (비즈니스 로직 없음)
  추가 파일 수: +12개
  매핑 코드: +40줄
  도메인 테스트 가능한 로직: 0줄
  결론: 과잉 (Hexagonal 이익 없음, 비용만 있음)

도메인: 이커머스 주문 (복잡한 비즈니스 로직)
  추가 파일 수: +12개
  매핑 코드: +40줄
  도메인 테스트 가능한 로직: 200줄 이상
  도메인 테스트로 절약되는 시간: 매 CI 실행마다 수십 분
  결론: 효과 있음 (비용 < 이익)

도메인: 결제 서비스 (외부 시스템 교체 가능성)
  결제 수단 교체 시 절약: Adapter 교체 O(1), Service 코드 변경 없음
  결론: 효과 있음 (장기적 유지보수 비용 절감)

판단 기준 공식:
  Hexagonal 도입 권장 조건:
    (도메인 비즈니스 로직 양 × 테스트 가치) + (외부 시스템 교체 가능성)
    > (매핑 코드 비용 + 파일 수 증가 비용 + 팀 학습 비용)
```

### 5. 실용적 절충 — 완전 Hexagonal이 아닌 선택적 적용

```
단계적 적용 전략:

Level 0: 레이어드 기본
  적합: 프로토타입, 단순 CRUD
  특징: Controller-Service-Repository, 빠른 개발

Level 1: 레이어드 + DIP
  적합: 비즈니스 로직이 있는 일반적 서비스
  특징: Repository 인터페이스를 Domain에, 외부 API Port 추출
  추가 비용: 최소 (인터페이스만 추가)

Level 2: Hexagonal (핵심 도메인만)
  적합: 복잡한 핵심 도메인 + 단순 지원 기능 혼재
  특징: 주문/결제는 Hexagonal, 공지/설정은 레이어드 Level 1
  장점: 비용을 줄이면서 복잡한 곳의 이익만 얻음

Level 3: Hexagonal (전체)
  적합: 복잡한 도메인, 큰 팀, 장기 프로젝트
  특징: 모든 기능에 Port/Adapter/Mapper
  비용: 가장 높음, 이익도 가장 높음

현실적 권장:
  소규모 팀 + 복잡한 도메인 → Level 2
  중규모 팀 + 복잡한 도메인 → Level 3
  어떤 경우든 단순 CRUD에는 Level 1로 충분
```

---

## 💻 실전 코드 — 비용 절감 기법

```java
// === MapStruct로 매핑 비용 절감 ===
// build.gradle
dependencies {
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
}

@Mapper(componentModel = "spring")
public interface OrderPersistenceMapper {

    // 필드명이 같으면 자동 매핑
    // 다른 경우만 @Mapping 명시
    @Mapping(source = "id.value", target = "orderId")
    @Mapping(source = "userId.value", target = "userId")
    @Mapping(source = "lines", target = "lines")
    OrderJpaEntity toEntity(Order order);

    @Mapping(source = "orderId", target = "id", qualifiedByName = "stringToOrderId")
    @Mapping(source = "userId", target = "userId", qualifiedByName = "stringToUserId")
    Order toDomain(OrderJpaEntity entity);

    @Named("stringToOrderId")
    default OrderId toOrderId(String value) { return OrderId.of(value); }

    @Named("stringToUserId")
    default UserId toUserId(String value) { return UserId.of(value); }

    // 컴파일 타임에 구현체 자동 생성 → 리플렉션 없이 타입 안전 매핑
}

// === 단순 CRUD는 레이어드 Level 1으로 충분 ===
// 공지사항 기능 (비즈니스 로직 없음)
@Service
public class NoticeService {
    private final NoticeRepository noticeRepository; // 인터페이스 (DIP 최소 적용)

    public NoticeResponse createNotice(CreateNoticeRequest request) {
        Notice notice = new Notice(request.getTitle(), request.getContent());
        noticeRepository.save(notice);
        return NoticeResponse.from(notice);
    }
    // 인터페이스로 추상화는 했지만 Port/Adapter 구조 없음
    // 비즈니스 로직이 없어서 도메인 테스트할 것도 없음
    // 이 정도면 충분
}

// === 복잡한 도메인만 Hexagonal 완전 적용 ===
// 동일 프로젝트에서 혼용
@Service
public class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderSavePort orderSavePort; // Hexagonal Driven Port
    private final PaymentPort paymentPort;
    // ... 완전한 Hexagonal 구조
}
```

---

## 📊 패턴 비교 — 도메인 복잡도별 아키텍처 비용 대비 효과

```
도메인별 Hexagonal 적용 ROI:

공지사항 CRUD:
  추가 비용: 파일 +12개, 매핑 코드 +40줄
  이익: 거의 없음 (비즈니스 로직 없어 도메인 테스트 없음)
  판단: 과잉 ❌

이커머스 주문 (할인, 재고, 결제 연동):
  추가 비용: 파일 +15개, 매핑 코드 +60줄
  이익: 도메인 테스트 200줄, 결제 수단 교체 O(1), 테스트 속도 10배 향상
  판단: 효과적 ✅

금융 계약 관리 (매우 복잡한 비즈니스 규칙):
  추가 비용: 파일 +30개, 매핑 코드 +100줄
  이익: 도메인 테스트 500줄, 인프라 교체 시 위험 없음, 팀 협업 용이
  판단: 매우 효과적 ✅✅
```

---

## ⚖️ 트레이드오프

```
Hexagonal Architecture 비용 요약:

구조적 비용:
  파일 수 증가: 레이어드 대비 2-3배
  매핑 코드: Domain ↔ JPA Entity 변환 (MapStruct로 줄일 수 있음)
  인터페이스: Port마다 인터페이스 파일 필요

팀 비용:
  학습 시간: 7-11시간 (처음 적용 시)
  신규 팀원 온보딩: 1-2주 (vs 레이어드 반나절)
  코드 리뷰 기준 수립: 초기 1-2주 논의 필요

이익 (비용을 정당화하는 경우):
  도메인 테스트 속도: 30초 → 0.01초
  인프라 교체 비용: O(N) → O(1)
  비즈니스 로직 변경 영향 범위: 명확히 격리
  새 입력 채널 추가: Driving Adapter 하나만

비용 > 이익인 경우:
  단순 CRUD 서비스 (비즈니스 로직 없음)
  2주 이내 폐기 예정
  팀 전체 동의 없는 상태에서 일방 적용
```

---

## 📌 핵심 정리

```
Hexagonal 비용 vs 이익:

주요 비용:
  파일 수: 레이어드의 2-3배
  매핑 코드: Domain ↔ JPA Entity (MapStruct 권장)
  학습 곡선: 7-11시간 + 2주 온보딩

비용 절감 방법:
  MapStruct: 매핑 코드 자동 생성
  선택적 적용: 복잡한 핵심 도메인만 Hexagonal
  레이어드 Level 1: 단순 기능은 DIP만 적용

적용 판단 기준:
  적합: 복잡한 비즈니스 로직 + 외부 시스템 연동 + 장기 프로젝트
  부적합: 단순 CRUD + 단기 프로젝트 + 팀 학습 여력 없음

현실적 권장:
  "핵심 도메인만 Hexagonal, 나머지는 레이어드 Level 1"
  = 비용 최소화, 핵심 이익 확보
```

---

## 🤔 생각해볼 문제

**Q1.** MapStruct와 수동 매핑 중 어느 것이 더 적합한가? MapStruct를 선택하지 말아야 하는 상황은?

<details>
<summary>해설 보기</summary>

**대부분의 경우 MapStruct가 적합합니다.**

MapStruct 선택: 필드명이 유사하고 반복적인 매핑이 많을 때. 컴파일 타임 검증으로 오타 감지.

수동 매핑 선택:
- 매핑 로직이 복잡해서 MapStruct 어노테이션이 오히려 읽기 어려울 때
- 팀에 MapStruct 경험자가 없어서 학습 비용이 클 때
- 매핑 수가 매우 적어서 수동으로도 충분할 때

MapStruct 주의사항:
```java
// 순환 참조 처리 필요 (Order → OrderLine → Order)
@Mapper(componentModel = "spring")
public interface OrderMapper {
    @Mapping(source = "lines", target = "lines")
    OrderJpaEntity toEntity(Order order);
    // OrderLine에 OrderJpaEntity 역참조가 있으면 무한 루프 가능
    // → @Context 또는 ignore 설정 필요
}
```

</details>

---

**Q2.** "핵심 도메인만 Hexagonal, 나머지는 레이어드"를 동일 프로젝트에서 혼용할 때 팀 내 혼란을 줄이는 방법은?

<details>
<summary>해설 보기</summary>

**명확한 기준과 문서화가 핵심입니다.**

```
1. 기능별 아키텍처 기준 문서화 (ARCHITECTURE.md):
   주문/결제: Hexagonal (이유: 복잡한 비즈니스 로직, 결제 수단 교체 예상)
   공지/설정: 레이어드 Level 1 (이유: 단순 CRUD, 비즈니스 로직 없음)

2. 패키지로 시각화:
   com.example
     ├── order/     ← Hexagonal 구조 (domain/, application/, adapter/)
     ├── notice/    ← 레이어드 구조 (controller/, service/, repository/)
     └── config/    ← 레이어드 구조

3. ArchUnit 규칙 분리:
   order 패키지: Hexagonal 규칙 적용
   notice 패키지: 레이어드 규칙 적용

4. 코드 리뷰 체크리스트:
   order 기능 PR: "Port 인터페이스는 domain/port/에 있는가?" 체크
   notice 기능 PR: "Service가 인터페이스에 의존하는가?" 체크
```

</details>

---

**Q3.** 프로젝트가 레이어드로 시작되어 3년이 지났다. 지금 Hexagonal로 전환하는 것이 합리적인가? 어떤 기준으로 판단하는가?

<details>
<summary>해설 보기</summary>

**전면 전환보다 선택적 부분 전환이 현실적입니다.**

판단 기준:
1. **현재 가장 큰 고통은 무엇인가?**
   - "테스트가 너무 느려서 CI가 1시간" → 핵심 도메인 Hexagonal로 즉시 이익
   - "결제 수단 교체가 너무 힘들어" → PaymentPort 추출만으로 해결

2. **전면 전환 vs 부분 전환**
   - 전면 전환: 2-3주 이상, 기능 개발 중단, 리스크 높음
   - 부분 전환: 기능 추가 시마다 Hexagonal로 작성, 3-6개월에 걸쳐 전환

3. **현실적 접근 (Strangler Fig Pattern)**:
   ```
   새 기능 → Hexagonal로 작성
   기존 기능 → 수정 시 Hexagonal로 리팩터링
   기존 기능 → 수정하지 않는 한 유지
   → 점진적으로 Hexagonal 비율 증가
   ```

   테스트 먼저 작성하고 리팩터링:
   ```
   1. 기존 기능에 통합 테스트 추가 (안전망)
   2. Repository 인터페이스 추출 (DIP 적용)
   3. 단위 테스트가 가능해지면 도메인 로직 이동
   4. 외부 API Port 추출
   ```

</details>

---

<div align="center">

**[⬅️ 이전: 테스트 용이성 향상](./06-testability-with-hexagonal.md)** | **[홈으로 🏠](../README.md)** | **[다음: DDD와 Hexagonal 통합 ➡️](./08-ddd-hexagonal-integration.md)**

</div>
