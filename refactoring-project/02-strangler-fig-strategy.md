# 점진적 리팩터링 전략 — Strangler Fig로 레거시 유지하면서 전환

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Strangler Fig 패턴으로 레거시를 유지하면서 새 아키텍처를 도입하는 방법은?
- 하나의 기능(UseCase)부터 Hexagonal로 전환하는 구체적 절차는?
- 테스트 커버리지를 확보한 후 리팩터링해야 하는 이유는?
- 기능 플래그(Feature Flag)로 전환 중에 롤백하는 방법은?
- 리팩터링 완료 기준은 어떻게 정의하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

"빅뱅 전환(전체를 한 번에 바꾸기)"은 실패할 확률이 높다. 3개월의 리팩터링 도중 요구사항이 바뀌면, 레거시와 새 코드가 모두 절반짜리인 상태가 된다.

Strangler Fig는 레거시 코드를 그대로 둔 채, 새 기능을 새 방식으로 구현하고 점진적으로 레거시를 대체한다. 각 단계가 독립적으로 배포 가능하고, 어느 단계에서든 롤백 가능하다.

---

## 😱 흔한 실수 (Before — 빅뱅 전환 시도)

```
빅뱅 전환의 실패 패턴:

Month 1: "전체를 Hexagonal로 전환하겠습니다"
  - 브랜치 생성: feature/hexagonal-refactoring
  - 팀 전체 리팩터링 시작

Month 2: 요구사항 변경 발생
  - 레거시 코드에 새 기능 추가 요청
  - "리팩터링 브랜치와 main 브랜치가 너무 달라져서 머지가 어렵습니다"

Month 3: 포기
  - 리팩터링 브랜치: 절반 완성
  - main 브랜치: 새 기능이 레거시 방식으로 추가됨
  - 결과: "나중에 다시 하기로" → 영원히 미룸

Strangler Fig가 달랐을 점:
  Month 1: "주문 생성 UseCase만 Hexagonal로 전환"
  → 완료, 배포, 운영
  Month 2: 요구사항 변경
  → 새 UseCase를 Hexagonal로 구현 (레거시 방식 안 씀)
  Month 3: "취소 UseCase 전환"
  → 완료, 배포, 운영
  점진적으로 레거시가 새 코드로 대체됨
```

---

## ✨ 올바른 접근 (After — Strangler Fig 단계별 전환)

```
Strangler Fig 전환 단계:

Phase 0: 안전망 구축 (1주)
  핵심 UseCase 통합 테스트 추가
  "이것이 통과하면 기능이 깨지지 않은 것"

Phase 1: Repository 인터페이스 추출 (1일/도메인)
  OrderService → OrderJpaRepository (구체) 의존 제거
  OrderService → OrderRepository (인터페이스) 의존 전환

Phase 2: 도메인 로직 이동 (1주)
  OrderService의 비즈니스 로직 → Order Entity로 이동
  Anemic Order → Rich Domain Model

Phase 3: 외부 연동 Port 추출 (2-3일)
  KakaoPayClient → PaymentPort 인터페이스
  KafkaTemplate → OrderEventPublisher 인터페이스
  KakaoPayAdapter, KafkaEventPublisher 생성

Phase 4: UseCase 인터페이스 도입 (1일)
  OrderController → PlaceOrderUseCase 인터페이스에 의존
  OrderService → PlaceOrderService로 분리

Phase 5: Entity 분리 (선택, 1주)
  Order (@Entity + 비즈니스 로직) → 분리
  Order (순수 Java) + OrderJpaEntity (@Entity)

각 Phase:
  ① 구현 → ② 기존 통합 테스트 통과 확인 → ③ 단위 테스트 추가 → ④ PR → ⑤ 배포
  배포 가능한 단위로 유지
```

---

## 🔬 내부 원리 — 각 Phase 상세

### 1. Phase 0: 통합 테스트 안전망 구축

```java
// 레거시 코드의 핵심 시나리오를 @SpringBootTest로 커버

@SpringBootTest
@Testcontainers
@Transactional
class OrderIntegrationTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Autowired
    private OrderController orderController; // 실제 Spring Bean

    @Autowired
    private OrderRepository orderRepository;

    // 이 테스트들이 리팩터링 내내 통과해야 함
    // = 리팩터링이 기능을 깨지 않았다는 증거

    @Test
    void 주문_생성_성공() {
        PlaceOrderRequest request = PlaceOrderRequest.builder()
            .userId("user-001")
            .items(List.of(new Item("item-1", 2, 5000L)))
            .build();

        ResponseEntity<?> response = orderController.placeOrder(request);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(orderRepository.findAll()).hasSize(1);
    }

    @Test
    void 최소_금액_미달_시_400() {
        PlaceOrderRequest request = PlaceOrderRequest.builder()
            .userId("user-001")
            .items(List.of(new Item("item-1", 1, 500L))) // 500원 = 최소 미달
            .build();

        ResponseEntity<?> response = orderController.placeOrder(request);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        assertThat(orderRepository.findAll()).isEmpty(); // 저장 안 됨
    }

    // 5-10개 핵심 시나리오 커버 (완전한 커버리지 불필요)
}
```

### 2. Phase 1: Repository 인터페이스 추출

```java
// Before: Service가 구체 JPA 클래스에 의존
@Service
public class OrderService {
    @Autowired
    private OrderJpaRepository orderRepository; // 구체 클래스
}

// Step 1: 인터페이스 추출 (order/domain/port/out/)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(Long id);
    List<Order> findByUserId(String userId);
}

// Step 2: 기존 JPA Repository가 인터페이스 구현
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpa; // 내부적으로 Spring Data JPA 사용

    @Override
    public void save(Order order) { jpa.save(order); } // 아직 @Entity Order 사용
    @Override
    public Optional<Order> findById(Long id) { return jpa.findById(id); }
    @Override
    public List<Order> findByUserId(String userId) { return jpa.findByUserId(userId); }
}

// Step 3: Service가 인터페이스에 의존 (자동 DI로 JpaOrderRepository 주입)
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository; // 인터페이스로 변경!
}

// 변경 확인: 기존 통합 테스트 통과 → 배포 가능
// 이익: 이제 InMemoryOrderRepository를 만들 수 있음
```

### 3. Phase 2: 도메인 로직 이동 (Anemic → Rich)

```java
// Before: 비즈니스 로직이 Service에
@Service
public class OrderService {
    public OrderResult placeOrder(PlaceOrderRequest request) {
        // ❌ 총액 계산이 Service에
        BigDecimal total = request.getItems().stream()
            .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        // ❌ 최소 금액 검증이 Service에
        if (total.compareTo(BigDecimal.valueOf(1000)) < 0) {
            throw new MinOrderAmountException("최소 1,000원");
        }

        Order order = new Order();
        order.setStatus("PLACED");
        order.setTotalAmount(total);
        orderRepository.save(order);
        return new OrderResult(order.getId(), total);
    }
}

// Step 1: Order에 비즈니스 메서드 추가 (기존 @Entity 그대로 유지)
@Entity // 아직 @Entity 유지 (Phase 5에서 분리)
public class Order {
    // 기존 필드 유지
    @Id @GeneratedValue private Long id;
    private BigDecimal totalAmount;
    private String status;

    // 새 비즈니스 메서드 추가
    public void place(BigDecimal total) {
        if (total.compareTo(BigDecimal.valueOf(1000)) < 0)
            throw new MinOrderAmountException("최소 1,000원");
        this.totalAmount = total;
        this.status = "PLACED";
    }

    public BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// Step 2: Service에서 비즈니스 로직 제거 (Domain에 위임)
@Service
public class OrderService {
    public OrderResult placeOrder(PlaceOrderRequest request) {
        Order order = new Order();
        BigDecimal total = order.calculateTotal(request.getItems());
        order.place(total); // 비즈니스 로직을 Order에 위임
        orderRepository.save(order);
        return new OrderResult(order.getId(), total);
    }
}

// Step 3: Order 단위 테스트 추가 (이제 가능!)
class OrderTest {
    @Test
    void 최소금액_미달() {
        Order order = new Order();
        assertThatThrownBy(() -> order.place(BigDecimal.valueOf(500)))
            .isInstanceOf(MinOrderAmountException.class);
    }
    // @SpringBootTest 없이 0.001초!
}
```

### 4. Phase 3: 외부 연동 Port 추출

```java
// Before: Service가 결제 SDK 직접 의존
@Service
public class OrderService {
    @Autowired private PaymentClient paymentClient; // 외부 SDK

    public OrderResult placeOrder(...) {
        // ...
        paymentClient.charge(userId, total); // 직접 호출
    }
}

// Step 1: PaymentPort 인터페이스 추출
// order/domain/port/out/PaymentPort.java
public interface PaymentPort {
    PaymentResult charge(String userId, BigDecimal amount);
}

// Step 2: KakaoPayAdapter 생성
// order/adapter/out/payment/KakaoPayAdapter.java
@Component
public class KakaoPayAdapter implements PaymentPort {
    private final PaymentClient client; // SDK는 여기에만

    @Override
    public PaymentResult charge(String userId, BigDecimal amount) {
        KakaoPayResponse res = client.charge(userId, amount);
        return new PaymentResult(res.getTransactionId());
    }
}

// Step 3: Service가 Port에만 의존
@Service
public class OrderService {
    @Autowired private PaymentPort paymentPort; // SDK 모름, Port만 앎

    public OrderResult placeOrder(...) {
        paymentPort.charge(userId, total); // Port를 통해 호출
    }
}

// 이익: 단위 테스트에서 결제 Port를 InMemory로 교체 가능
// InMemoryPaymentPort: public PaymentResult charge(...) { return success(); }
```

### 5. 기능 플래그로 안전하게 전환

```java
// 기능 플래그(Feature Flag)로 레거시와 새 코드를 동시에 운영

@Service
public class OrderService {

    private final LegacyPlaceOrderService legacyService;
    private final PlaceOrderService newService; // 새 구현
    private final FeatureFlags featureFlags;

    public OrderResult placeOrder(PlaceOrderRequest request) {
        if (featureFlags.isEnabled("new-place-order")) {
            // 새 구현 사용 (점진적 롤아웃)
            return newService.placeOrder(PlaceOrderCommand.from(request));
        } else {
            // 레거시 유지
            return legacyService.placeOrder(request);
        }
    }
}

// 롤아웃 전략:
// 1단계: 내부 사용자 1% → 새 구현 사용
// 2단계: 전체 트래픽 10% → 모니터링 (에러율, 레이턴시)
// 3단계: 50%, 100% 점진적 증가
// 문제 발생 시: 플래그 비활성화 → 즉시 레거시로 롤백

// 기능 플래그 설정 (application.yml)
feature-flags:
  new-place-order:
    enabled: true
    rollout-percentage: 50  # 50% 트래픽에만 적용
```

---

## 💻 실전 코드 — Phase별 커밋 전략

```
Week 1 (Phase 0): 안전망 구축
  commit: "test: 주문 생성 핵심 시나리오 통합 테스트 추가"
  commit: "test: 최소 금액 미달 시나리오 통합 테스트"
  commit: "test: 결제 실패 시나리오 통합 테스트"
  → CI 통과 → 배포 가능

Week 2 (Phase 1): Repository 인터페이스
  commit: "refactor: OrderRepository 인터페이스 domain/port/out/으로 이동"
  commit: "refactor: JpaOrderRepository implements OrderRepository 구현"
  commit: "refactor: OrderService가 OrderRepository 인터페이스 의존"
  commit: "test: InMemoryOrderRepository 단위 테스트 추가"
  → CI 통과 (통합 테스트 포함) → 배포 가능

Week 3 (Phase 2): 도메인 로직 이동
  commit: "feat: Order.place() 메서드 추가 (비즈니스 규칙 이동)"
  commit: "refactor: OrderService.placeOrder()에서 Order.place()로 위임"
  commit: "test: OrderTest 단위 테스트 추가 (0.001초)"
  → CI 통과 → 배포 가능

Week 4 (Phase 3): PaymentPort 추출
  commit: "refactor: PaymentPort 인터페이스 추출"
  commit: "refactor: KakaoPayAdapter 생성"
  commit: "refactor: OrderService가 PaymentPort 의존"
  commit: "test: InMemoryPaymentPort로 OrderService 단위 테스트"
  → CI 통과 → 배포 가능

커밋 규칙:
  각 커밋 후 기존 통합 테스트 통과 확인 (반드시)
  "refactor:" 커밋은 기능 변경 없음 (동작 동일)
  "test:" 커밋은 테스트만 추가 (코드 변경 없음)
```

---

## 📊 패턴 비교 — 빅뱅 전환 vs Strangler Fig

```
3개월 후 상태 비교:

빅뱅 전환:
  Month 1: 전체 리팩터링 시작 (기능 개발 중단)
  Month 2: 요구사항 변경 발생 → 충돌
  Month 3: 절반 완성된 리팩터링 + 레거시 + 새 기능 = 혼재
  결과: 모두 불완전, 배포 불가, 팀 사기 저하

Strangler Fig:
  Month 1: Phase 0-1 완료 → 배포 (Repository 인터페이스)
  Month 2: Phase 2 완료 → 배포 (Domain 로직 이동)
            요구사항 변경 → 새 UseCase를 Hexagonal로 구현
  Month 3: Phase 3-4 완료 → 배포 (Port/Adapter)
  결과: 점진적 완성, 각 Phase 배포 가능, 팀 성취감

핵심 차이:
  빅뱅: 완성될 때까지 이익 없음 (위험 집중)
  Strangler Fig: 각 Phase마다 이익 발생 (위험 분산)
```

---

## ⚖️ 트레이드오프

```
Strangler Fig의 비용:
  레거시 코드와 새 코드가 일시적으로 공존 (복잡도)
  기능 플래그 관리 (나중에 제거 필요)
  각 Phase의 설계 결정이 후속 Phase에 영향

Strangler Fig의 이익:
  각 Phase가 독립적으로 배포 가능
  어느 단계에서든 롤백 가능
  비즈니스 개발과 병행 가능
  점진적 테스트 추가로 안전성 확보

언제 빅뱅이 더 나은가:
  코드가 매우 작음 (1,000줄 이하)
  팀이 완전한 재작성 시간을 확보
  비즈니스 요구사항이 안정적 (3개월간 변화 없음)
  대부분의 현실 프로젝트에서는 해당 없음
```

---

## 📌 핵심 정리

```
Strangler Fig 리팩터링 핵심:

단계:
  Phase 0: 통합 테스트 안전망 (핵심 5-10개 시나리오)
  Phase 1: Repository 인터페이스 추출 (1일)
  Phase 2: 도메인 로직 이동 (1주)
  Phase 3: 외부 연동 Port 추출 (2-3일)
  Phase 4: UseCase 인터페이스 도입 (1일)

각 Phase 완료 기준:
  ① 기존 통합 테스트 통과
  ② 새 단위 테스트 추가
  ③ CI 통과 → 배포 가능

기능 플래그:
  레거시와 새 코드 공존 → 점진적 롤아웃
  문제 시 즉시 레거시 롤백

성공 기준:
  단위 테스트 비율: 5% → 70%
  @SpringBootTest 비율: 80% → 20%
  Fat Service: 1,247줄 → 200줄 이하
```

---

## 🤔 생각해볼 문제

**Q1.** Phase 1(Repository 인터페이스 추출) 완료 후 실제로 얻는 이익은 무엇인가? 당장 코드 품질이 좋아지지 않는데 굳이 해야 하는가?

<details>
<summary>해설 보기</summary>

Phase 1 완료의 즉각적인 이익:

1. **InMemory 구현체로 단위 테스트 가능**: `OrderService` 테스트에 `@SpringBootTest`가 불필요해짐 → 단위 테스트 속도 30초 → 0.01초

2. **Port 위치 결정**: Repository 인터페이스가 `domain/port/out/`에 위치 → 이후 Phase의 기준이 됨

3. **검증**: "이 인터페이스가 Domain 레이어에 있어야 한다"는 팀 합의 확인

4. **자신감**: "Phase 1이 배포됐다 → 진짜로 하는구나" 팀 모멘텀

당장의 코드 품질 변화는 작지만, 이후 Phase를 가능하게 하는 기반입니다. 빌딩을 지을 때 기초 공사가 눈에 안 보여도 중요한 것처럼.

</details>

---

**Q2.** 리팩터링 도중 새 기능 요청이 들어왔다. 새 기능을 레거시 방식으로 구현해야 하는가, Hexagonal 방식으로 구현해야 하는가?

<details>
<summary>해설 보기</summary>

**가능하면 Hexagonal 방식으로, 불가능하면 레거시 방식으로 하되 기록합니다.**

결정 기준:
- 새 기능이 리팩터링된 영역과 완전히 독립적인가?
  → 독립적이면 레거시 방식도 허용 (나중에 Strangler Fig로 전환)
  → 연관되어 있으면 Hexagonal 방식 (리팩터링 기반 위에 구현)

레거시 방식으로 구현 시:
```java
// 반드시 기술 부채 표시
// TODO: [ARCH-DEBT] 레거시 방식으로 구현됨 - JIRA-1234 전환 예정
public class NewFeatureService {
    @Autowired
    JpaRepository repo; // 임시 직접 의존
}
```

이 표시가 다음 아키텍처 리뷰에서 전환 아이템이 됩니다.

</details>

---

**Q3.** 기능 플래그를 이용한 점진적 롤아웃에서 "새 구현이 50% 트래픽에 문제없다"는 것을 어떻게 확인하는가?

<details>
<summary>해설 보기</summary>

**에러율, 레이턴시, 비즈니스 메트릭을 모니터링합니다.**

```yaml
# 모니터링 대상
모니터링:
  에러율:
    기준: 새 구현의 5xx 에러율이 레거시의 1.5배 이하
    도구: Datadog, CloudWatch, Grafana
    
  레이턴시:
    기준: P99 레이턴시가 레거시 대비 120% 이하
    
  비즈니스 메트릭:
    주문 완료율 (정상이면 레거시와 동일해야 함)
    결제 성공률 (핵심 지표)
    
롤아웃 단계:
  1%: 내부 테스트 계정
  5%: 1일 모니터링
  20%: 3일 모니터링 (이상 없으면)
  50%: 1주 모니터링
  100%: 레거시 코드 제거 + 기능 플래그 제거
```

가장 중요한 것: 롤아웃 중에 모니터링 대시보드를 팀이 보고 있어야 합니다. 자동 롤백 기준(에러율 임계치 초과 시 플래그 자동 비활성화)을 미리 설정해두면 더 안전합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 레거시 코드 분석](./01-legacy-code-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: 도메인 레이어 분리 ➡️](./03-extracting-domain-layer.md)**

</div>
