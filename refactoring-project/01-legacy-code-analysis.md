# 레거시 코드 분석 — 의존성 그래프로 문제 시각화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Controller에 비즈니스 로직, Service에 SQL 직접 호출이 있는 실제 코드의 문제를 어떻게 진단하는가?
- 의존성 그래프로 문제 구조를 어떻게 시각화하는가?
- 변경 시 파급 범위를 어떻게 측정하는가?
- 테스트 커버리지를 측정하고 리팩터링 시작 기준을 어떻게 판단하는가?
- "이 코드는 리팩터링이 필요하다"는 진단 기준은 무엇인가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

리팩터링을 시작하기 전에 현재 상태를 정확히 진단해야 한다. "코드가 지저분하다"는 느낌만으로 리팩터링을 시작하면 무엇을 바꿔야 하는지, 어디서 시작해야 하는지, 얼마나 걸리는지 알 수 없다.

의존성 그래프, 테스트 커버리지, 변경 파급 범위를 측정하면 "어느 코드가 가장 큰 문제인가"와 "어느 순서로 리팩터링하면 되는가"가 명확해진다.

---

## 😱 흔한 실수 (Before — 진단 없이 리팩터링 시작)

```
진단 없는 리팩터링의 실패 패턴:

"전체를 한 번에 Hexagonal로 바꾸겠습니다"
→ 3개월 작업
→ 중간에 요구사항 변경 → 리팩터링 코드와 레거시 코드 혼재
→ 완성 못하고 "리팩터링 브랜치" 영원히 머지 못함

"중요한 것부터 바꾸겠습니다"
→ "중요한 것"의 기준이 개인 판단
→ 정작 문제가 큰 곳은 그대로
→ 효과 없는 리팩터링

올바른 시작:
  1. 현재 상태 측정 (의존성, 커버리지, 변경 파급)
  2. 문제가 가장 큰 곳 식별
  3. 그곳부터 점진적으로 시작
```

---

## ✨ 올바른 접근 (After — 측정 후 우선순위 결정)

```
진단 → 우선순위 → 점진적 리팩터링:

진단 도구:
  ① 의존성 그래프: 누가 누구를 얼마나 의존하는가
  ② 코드 메트릭: 클래스 크기, 메서드 수, 복잡도
  ③ 테스트 커버리지: 어느 코드가 테스트 없이 노출됐는가
  ④ 변경 이력: git log로 가장 자주 변경되는 파일

우선순위 결정 기준:
  (변경 빈도) × (테스트 없음) × (의존성 많음) = 위험도
  → 위험도 높은 곳부터 리팩터링

리팩터링 기준 달성 여부:
  핵심 시나리오 통합 테스트 커버리지 > 80%
  → 안전망이 생겼으니 리팩터링 시작 가능
```

---

## 🔬 내부 원리 — 진단 방법

### 1. 레거시 코드의 전형적 문제 패턴

```java
// 실제 레거시 코드 예시: 3가지 문제가 혼재

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @Autowired
    private JdbcTemplate jdbcTemplate; // ❌ Controller가 DB를 직접 접근

    @PostMapping
    public ResponseEntity<?> placeOrder(@RequestBody PlaceOrderRequest request) {

        // ❌ Controller에 비즈니스 검증 (레이어 위반)
        if (request.getTotalAmount() < 1000) {
            return ResponseEntity.badRequest().body("최소 1,000원 이상");
        }
        if (request.getItems() == null || request.getItems().isEmpty()) {
            return ResponseEntity.badRequest().body("항목을 선택해주세요");
        }

        // ❌ Controller가 DB를 직접 조회
        Integer stock = jdbcTemplate.queryForObject(
            "SELECT stock FROM items WHERE id = ?",
            Integer.class, request.getItems().get(0).getItemId()
        );
        if (stock < request.getItems().get(0).getQuantity()) {
            return ResponseEntity.badRequest().body("재고 부족");
        }

        OrderResult result = orderService.placeOrder(request);
        return ResponseEntity.ok(result);
    }
}

@Service
@Transactional
public class OrderService {

    @Autowired private OrderRepository orderRepository;
    @Autowired private UserRepository userRepository;
    @Autowired private CouponRepository couponRepository;
    @Autowired private InventoryRepository inventoryRepository;
    @Autowired private PaymentClient paymentClient; // 외부 SDK 직접 의존
    @Autowired private KafkaTemplate<String, String> kafkaTemplate; // 인프라 직접
    @Autowired private JavaMailSender mailSender;  // 인프라 직접
    @Autowired private RedisTemplate<String, String> redisTemplate; // 캐시 직접

    public OrderResult placeOrder(PlaceOrderRequest request) { // ← HTTP DTO 직접 수신

        // ❌ 비즈니스 로직이 Service에 (도메인 객체에 있어야 함)
        BigDecimal total = BigDecimal.ZERO;
        for (PlaceOrderRequest.Item item : request.getItems()) {
            total = total.add(item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())));
        }

        // ❌ 쿠폰 적용 로직이 Service에
        if (request.getCouponCode() != null) {
            Coupon coupon = couponRepository.findByCode(request.getCouponCode());
            total = coupon.apply(total);
        }

        // JPA Entity 직접 생성 (Anemic Model)
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setTotalAmount(total);
        order.setStatus("PLACED");
        orderRepository.save(order);

        // ❌ 트랜잭션 내에서 외부 API 직접 호출
        paymentClient.charge(request.getUserId(), total);

        // ❌ 트랜잭션 내에서 Kafka 발행
        kafkaTemplate.send("order-placed", order.getId().toString());

        // ❌ 트랜잭션 내에서 이메일 발송
        SimpleMailMessage mail = new SimpleMailMessage();
        mail.setTo(request.getUserEmail());
        mailSender.send(mail);

        return new OrderResult(order.getId(), total);
    }

    // OrderService에 관련 없는 기능들도 추가됨
    public Page<Order> findOrders(String userId, Pageable pageable) { ... }
    public void cancelOrder(Long orderId) { ... }
    public BigDecimal calculateShippingFee(List<OrderItem> items) { ... } // ← 비즈니스 로직
    public void updateOrderStatus(Long orderId, String status) { ... }
    // ... (총 27개 메서드, 1,247줄)
}

// ❌ Anemic Domain Model: 비즈니스 로직 없음
@Entity
public class Order {
    @Id @GeneratedValue private Long id;
    private String userId;
    private BigDecimal totalAmount;
    private String status;
    private LocalDateTime createdAt;

    // getter/setter만 있음, 비즈니스 메서드 없음
}
```

### 2. 의존성 그래프 시각화

```
레거시 코드의 의존성 그래프:

OrderController ──→ OrderService ──→ OrderRepository (JPA)
     │                    │         UserRepository (JPA)
     │                    │         CouponRepository (JPA)
     ↓                    │         InventoryRepository (JPA)
JdbcTemplate              │         PaymentClient (외부 SDK)
(DB 직접)                  │         KafkaTemplate (인프라)
                          │         JavaMailSender (인프라)
                          │         RedisTemplate (캐시)
                          ↓
                      Order (@Entity, Anemic)

의존성 문제:
  OrderService의 의존성 수: 8개 (너무 많음)
  Fan-in(OrderService를 아는 것): OrderController만
  Fan-out(OrderService가 아는 것): 8개

Fan-out이 높을수록 변경 시 영향받는 것이 많음
= OrderService가 변경의 진원지 → 모든 의존 대상이 영향

IntelliJ에서 의존성 그래프 보기:
  Analyze → Dependency Matrix (모듈 수준)
  Analyze → Show Diagram (클래스 수준)
  → "OrderService가 연결된 선이 몇 개인가" 시각적 확인
```

### 3. 코드 메트릭 측정

```
측정 도구: IntelliJ Metrics, SonarQube, Checkstyle

핵심 메트릭과 기준:

① 클래스 크기 (Lines of Code)
  경고: 300줄 이상
  위험: 500줄 이상
  레거시 OrderService: 1,247줄 → 즉시 분리 대상

② 메서드 수 (Method Count per Class)
  경고: 15개 이상
  위험: 25개 이상
  레거시 OrderService: 27개 메서드 → 여러 책임 혼재

③ 순환 복잡도 (Cyclomatic Complexity per Method)
  경고: 10 이상
  위험: 20 이상
  placeOrder() 메서드: if문 8개, 복잡도 = 9 → 단위 테스트 어려움

④ 결합도 (Efferent Coupling - Ce)
  경고: 7 이상
  위험: 10 이상
  OrderService의 Ce: 8 → 8개 외부 의존

⑤ 응집도 (Lack of Cohesion in Methods - LCOM)
  경고: 0.5 이상 (0은 완전 응집, 1은 완전 비응집)
  OrderService: LCOM ≈ 0.8 → 메서드들이 서로 무관한 필드를 사용
```

### 4. 테스트 커버리지 측정 및 리팩터링 시작 기준

```
테스트 커버리지 측정 (JaCoCo):

// build.gradle.kts
plugins {
    id("jacoco")
}

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}

// ./gradlew test jacocoTestReport
// → build/reports/jacoco/test/html/index.html 확인

레거시 코드의 전형적 커버리지:
  OrderController: 0% (수동 테스트만)
  OrderService: 15% (@SpringBootTest 느린 테스트 몇 개)
  Order (@Entity): 0% (Anemic이라 테스트할 로직 없음)
  전체: 12%

리팩터링 시작 기준:
  핵심 비즈니스 시나리오 통합 테스트 커버리지 > 80%
  → "이 테스트들이 통과하면 리팩터링이 기능을 깨지 않은 것"

통합 테스트 추가 방법:
  @SpringBootTest + @Transactional(롤백) + Testcontainers
  → 핵심 UseCase 5-10개를 E2E로 먼저 커버
  → 이것이 리팩터링의 안전망
```

### 5. 변경 이력으로 위험도 측정

```bash
# git log로 자주 변경된 파일 찾기

# 최근 6개월 동안 가장 많이 변경된 파일 10개
git log --since="6 months ago" --name-only --format="" | \
  sort | uniq -c | sort -rn | head -10

결과 예시:
  145 src/main/java/.../OrderService.java  ← 가장 자주 변경!
   78 src/main/java/.../OrderController.java
   45 src/main/java/.../PaymentService.java
   32 src/main/java/.../Order.java
   ...

위험도 계산:
  OrderService: 145회 변경 × 테스트 15% × 의존성 8개 = 높은 위험도
  → OrderService가 1순위 리팩터링 대상

# 두 파일이 함께 변경되는 빈도 (논리적 결합도)
git log --name-only --format="" | awk 'NF {print}' | \
  paste - - | sort | uniq -c | sort -rn | head -10

# 같이 변경되는 파일 쌍이 많으면 → 경계가 잘못된 신호
```

---

## 💻 실전 코드 — 진단 보고서 예시

```
레거시 진단 보고서 (OrderService 기준):

==============================
문제 1: Fat Service (1,247줄, 27개 메서드)
심각도: 높음
증거:
  - 주문 생성/취소/조회/배송료 계산 모두 한 클래스
  - 결합도(Ce) 8: JPA 4개 + 외부 API 3개 + 캐시 1개
리팩터링: 비즈니스 로직 → Domain 객체로, 외부 연동 → Port/Adapter로

문제 2: Anemic Domain Model
심각도: 중간
증거:
  - Order.java: getter/setter만, 비즈니스 메서드 0개
  - 최소 금액 계산, 상태 전이 로직이 Service에 분산
리팩터링: Order에 place(), cancel(), calculateTotal() 추가

문제 3: HTTP DTO 침투
심각도: 중간
증거:
  - OrderService.placeOrder(PlaceOrderRequest request)
  - PlaceOrderRequest가 Service 레이어에서 사용됨
리팩터링: PlaceOrderCommand로 변환 후 Service에 전달

문제 4: 트랜잭션 내 외부 API/이벤트 호출
심각도: 높음
증거:
  - paymentClient.charge() + kafkaTemplate.send() + mailSender.send() 모두 @Transactional 내
  - 결제 성공 후 DB 롤백 시 결제 취소 불가
리팩터링: Outbox Pattern으로 이벤트 분리

문제 5: Controller에 비즈니스 검증 + DB 직접 접근
심각도: 높음
증거:
  - if (request.getTotalAmount() < 1000) → Controller에 비즈니스 규칙
  - jdbcTemplate.queryForObject() → Controller가 DB 직접 접근
리팩터링: 비즈니스 검증 → Domain, DB 접근 → UseCase를 통해

우선순위:
  1순위: OrderService 분해 (위험도 가장 높음)
  2순위: Domain 로직 이동 (Anemic 해소)
  3순위: Port/Adapter 분리 (외부 연동 격리)
  4순위: Outbox Pattern 도입 (트랜잭션 안전성)
==============================
```

---

## 📊 패턴 비교 — 진단 지표별 비교

```
지표              │ 레거시 상태     │ 목표 (리팩터링 후)
──────────────────┼───────────────┼──────────────────
OrderService 줄수 │ 1,247줄       │ ~200줄 (분해 후)
메서드 수         │ 27개          │ ~6개
결합도(Ce)        │ 8             │ ~3 (Port만 의존)
테스트 커버리지   │ 12%           │ 70%+
@SpringBootTest 비율│ 80%          │ 20%
단위 테스트 비율  │ 5%            │ 70%
변경 파급 범위    │ 전체          │ 해당 레이어만
```

---

## ⚖️ 트레이드오프

```
진단에 시간을 쓰는 비용 vs 이익:

비용: 진단 및 계획 수립 시간 (1-3일)
이익:
  무작위 리팩터링 대신 고효율 순서로 작업
  "어디까지 왔는가" 측정 가능 (지표 변화 추적)
  팀원에게 "왜 이 순서로 작업하는가" 설명 가능
  리팩터링 완료 기준이 명확 (커버리지 X%, 메트릭 Y 이하)

진단 없이 시작하면:
  "어디서부터?"로 시간 낭비
  중요하지 않은 곳을 리팩터링
  "다 된 건가?" 기준 없음
  리팩터링이 무한정 늘어남
```

---

## 📌 핵심 정리

```
레거시 코드 진단 핵심:

진단 4가지 도구:
  ① 의존성 그래프: Fan-out 높은 클래스 → 변경 위험도 높음
  ② 코드 메트릭: 300줄/15메서드 이상 클래스 → 분리 대상
  ③ 테스트 커버리지: 80% 미만 → 안전망 먼저 구축
  ④ 변경 이력: 자주 변경 + 테스트 없음 = 최우선 리팩터링

리팩터링 시작 조건:
  핵심 시나리오 통합 테스트 80% 커버
  → "이것이 통과하면 기능이 깨지지 않은 것"

레거시 코드의 전형적 문제:
  Fat Service (1,000줄+, 의존성 8개+)
  Anemic Domain Model (Entity에 로직 없음)
  HTTP DTO 침투 (Service가 Controller DTO를 직접 사용)
  트랜잭션 오염 (결제/이메일이 @Transactional 안에)
  Controller에 비즈니스 로직

우선순위 공식:
  (변경 빈도) × (테스트 없음) × (의존성 수) = 위험도
  → 위험도 높은 곳부터 리팩터링
```

---

## 🤔 생각해볼 문제

**Q1.** 테스트가 전혀 없는 레거시 코드를 리팩터링해야 하는데, 테스트를 먼저 추가하는 데 시간이 너무 많이 걸린다. 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

**최소한의 안전망(핵심 시나리오 통합 테스트)만 먼저 추가합니다.**

우선순위:
1. `@SpringBootTest` + Testcontainers로 핵심 UseCase 5개만 E2E 커버 (1-2일)
2. 이 5개 테스트가 통과하면 리팩터링 시작
3. 리팩터링하면서 단위 테스트 추가 (각 단계별로)

"완벽한 커버리지 후 리팩터링"은 현실적으로 불가능합니다. 핵심 흐름 5개만 통합 테스트로 커버하고 시작하세요. 리팩터링을 진행하면서 각 레이어를 분리할 때 단위 테스트를 추가합니다.

</details>

---

**Q2.** OrderService가 1,247줄인데 한 번에 분해하면 너무 큰 변경이다. 어떤 기준으로 조각을 나눠서 진행하는가?

<details>
<summary>해설 보기</summary>

**"하나의 커밋 = 하나의 책임 이동"으로 나눕니다.**

```
Week 1: 비즈니스 로직 Domain으로 이동
  커밋 1: "refactor: Order.place() 메서드 추가 + OrderService에서 제거"
  커밋 2: "refactor: Order.calculateTotal() 이동"
  커밋 3: "refactor: Order.cancel() 이동"

Week 2: Port 추출
  커밋 4: "refactor: OrderRepository 인터페이스 domain/port/out/으로 이동"
  커밋 5: "refactor: PaymentPort 인터페이스 추출 + KakaoPayAdapter 생성"

Week 3: UseCase 인터페이스 도입
  커밋 6: "refactor: PlaceOrderUseCase 인터페이스 도입"
  커밋 7: "refactor: OrderService → PlaceOrderService + FindOrderService 분리"
```

각 커밋 후 기존 통합 테스트가 통과하면 성공. 실패하면 이전 커밋으로 롤백 가능한 크기로 유지합니다.

</details>

---

**Q3.** 진단 결과 "전체를 다시 써야 한다"는 결론이 나왔다. 이때 새로 쓰는 것과 점진적 리팩터링 중 어느 것이 더 적합한가?

<details>
<summary>해설 보기</summary>

**대부분의 경우 점진적 리팩터링이 더 안전합니다.** "전체를 새로 쓰자"는 유혹은 "제2의 시스템 효과(Second System Effect)"로 알려진 함정입니다.

새로 쓰는 것이 적합한 경우:
- 코드가 작고(3,000줄 이하) 비즈니스 요구사항이 완전히 이해됨
- 기존 코드에 숨겨진 비즈니스 규칙이 없을 만큼 단순함
- 팀이 충분한 시간과 여유를 확보함

점진적 리팩터링이 더 나은 경우 (대부분):
- 기존 코드에 숨겨진 비즈니스 규칙이 많음
- 비즈니스가 계속 변화 중
- 새로 쓰는 동안 기존 시스템을 유지해야 함

Joel Spolsky가 말했습니다: "코드를 처음부터 다시 쓰는 것은 소프트웨어 회사가 저지를 수 있는 가장 큰 실수다."

Strangler Fig(점진적 교체)가 더 현실적이고 안전합니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 코드 가이드라인 수립](../package-structure/05-code-guidelines.md)** | **[홈으로 🏠](../README.md)** | **[다음: 점진적 리팩터링 전략 ➡️](./02-strangler-fig-strategy.md)**

</div>
