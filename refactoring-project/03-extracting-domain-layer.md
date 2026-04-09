# 도메인 레이어 분리 — Service에서 도메인 로직 추출

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Service에서 도메인 로직을 추출하여 Domain 객체로 이동하는 구체적 과정은?
- UseCase 인터페이스 도입으로 진입점을 명확히 하는 방법은?
- 도메인 레이어에서 Spring 의존성을 제거하는 단계별 과정은?
- 각 단계별 커밋 전략은 어떻게 짜는가?
- 리팩터링 후 단위 테스트를 어떻게 추가하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Strangler Fig Phase 2의 핵심이다. 레이어드 아키텍처의 Fat Service에서 비즈니스 로직을 Domain 객체로 이동하면 두 가지 즉각적 이익이 생긴다.

첫째, 비즈니스 규칙이 한 곳(Domain)에 집중되어 중복이 없어진다. 둘째, Domain 객체는 순수 Java이므로 Spring 없이 0.001초 단위 테스트가 가능해진다. 이 단계가 Hexagonal Architecture로의 전환에서 가장 실질적인 가치를 만든다.

---

## 😱 흔한 실수 (Before — 로직 이동 방법을 모를 때)

```
실수 1: 도메인 로직을 Service에서 제거만 하고 Domain에 추가 안 함
  Service:
    // Before
    if (total < 1000) throw new MinOrderAmountException();
    // After
    order.validate(total); // Order.validate() 메서드 없음!
  → 컴파일 에러

실수 2: Domain 로직을 이동하면서 Spring 의존성 같이 가져감
  @Entity
  public class Order {
      @Autowired
      private OrderValidator validator; // ❌ Domain에 @Autowired?!
      public void place(BigDecimal total) {
          validator.validate(total); // Domain이 Spring 의존
      }
  }

실수 3: 한 번에 너무 많이 이동
  "OrderService 전체 로직을 Order로 이동하겠습니다"
  → 1,000줄 Domain 클래스 생성
  → 테스트 없이 진행 → 어디서 버그가 났는지 모름

올바른 방식:
  한 번에 하나의 메서드
  이동 → 테스트 추가 → 통합 테스트 통과 → 커밋
```

---

## ✨ 올바른 접근 (After — 메서드 단위 점진적 이동)

```
도메인 로직 이동 원칙:

① 한 번에 하나의 비즈니스 규칙
② 이동 후 즉시 단위 테스트 추가
③ 기존 통합 테스트 통과 확인
④ 커밋 (배포 가능 상태 유지)
⑤ 반복

이동 우선순위:
  먼저: 비즈니스 규칙 검증 (if 조건문)
  다음: 계산 로직 (금액, 상태 결정)
  마지막: 상태 전이 (place, cancel, complete)

이동하지 않는 것:
  외부 API 호출 → Port/Adapter로 (Phase 3)
  Repository 저장 → Application Service에
  이메일/Kafka → Port/Adapter로 (Phase 3)
```

---

## 🔬 내부 원리 — 단계별 도메인 로직 추출

### 1. 값 계산 로직 이동

```java
// Before: OrderService에 총액 계산
@Service
public class OrderService {
    public OrderResult placeOrder(PlaceOrderRequest request) {
        // ❌ 비즈니스 계산이 Service에
        BigDecimal total = request.getItems().stream()
            .map(i -> i.getPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        // ...
    }
}

// Step 1: Order에 calculateTotal() 추가
@Entity // 아직 @Entity 유지 (Phase 5에서 분리)
public class Order {
    // 기존 필드들 유지...

    // 새 메서드 추가 (비즈니스 계산 이동)
    public BigDecimal calculateTotal() {
        return this.items.stream()
            .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// Step 2: Service에서 Order.calculateTotal()로 위임
@Service
public class OrderService {
    public OrderResult placeOrder(PlaceOrderRequest request) {
        Order order = buildOrder(request); // Order 생성
        BigDecimal total = order.calculateTotal(); // Order에 위임
        // ...
    }
}

// Step 3: 단위 테스트 추가 (Order는 이제 테스트 가능!)
class OrderTest {
    @Test
    void 총액_계산_단가x수량_합산() {
        Order order = createOrder(
            orderLine("item-1", 2, BigDecimal.valueOf(5000)),
            orderLine("item-2", 1, BigDecimal.valueOf(3000))
        );

        BigDecimal total = order.calculateTotal();

        assertThat(total).isEqualByComparingTo(BigDecimal.valueOf(13000));
        // @SpringBootTest 없이 0.001초!
    }
}

// 커밋: "refactor: 총액 계산 로직 Order.calculateTotal()로 이동"
// → CI 통과 (통합 테스트 포함) → 배포 가능
```

### 2. 비즈니스 검증 로직 이동

```java
// Before: 최소 금액 검증이 Service에
@Service
public class OrderService {
    public OrderResult placeOrder(...) {
        BigDecimal total = order.calculateTotal();

        // ❌ 비즈니스 검증이 Service에
        if (total.compareTo(BigDecimal.valueOf(1000)) < 0) {
            throw new MinOrderAmountException("최소 1,000원 이상");
        }

        if (request.getItems() == null || request.getItems().isEmpty()) {
            throw new IllegalArgumentException("주문 항목 필요");
        }
    }
}

// Step 1: Order에 place() 메서드 추가 (검증 포함)
public class Order {
    // ...

    // 비즈니스 규칙: place() 호출 시 검증 실행
    public void place() {
        if (this.items == null || this.items.isEmpty())
            throw new IllegalArgumentException("주문 항목이 없습니다");

        BigDecimal total = calculateTotal();
        if (total.compareTo(BigDecimal.valueOf(1000)) < 0)
            throw new MinOrderAmountException("최소 1,000원 이상 주문하세요");

        this.status = "PLACED";
        this.totalAmount = total;
        this.placedAt = LocalDateTime.now();
    }
}

// Step 2: Service에서 Order.place()로 위임
@Service
public class OrderService {
    public OrderResult placeOrder(PlaceOrderRequest request) {
        Order order = buildOrder(request);
        order.place(); // 검증 + 상태 변경을 Order에 위임
        orderRepository.save(order);
        return new OrderResult(order.getId(), order.getTotalAmount());
    }
}

// Step 3: 비즈니스 규칙 단위 테스트
class OrderTest {
    @Test
    void 최소_금액_미달_예외() {
        Order order = createOrder(orderLine("item-1", 1, BigDecimal.valueOf(500)));

        assertThatThrownBy(order::place)
            .isInstanceOf(MinOrderAmountException.class)
            .hasMessageContaining("1,000원");
    }

    @Test
    void 빈_항목_예외() {
        Order order = createOrder(); // 항목 없음
        assertThatThrownBy(order::place)
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void 정상_주문_상태_변경() {
        Order order = createOrder(orderLine("item-1", 2, BigDecimal.valueOf(5000)));

        order.place();

        assertThat(order.getStatus()).isEqualTo("PLACED");
        assertThat(order.getTotalAmount()).isEqualByComparingTo(BigDecimal.valueOf(10000));
        assertThat(order.getPlacedAt()).isNotNull();
    }
}
// 커밋: "refactor: 비즈니스 검증 Order.place()로 이동"
```

### 3. 상태 전이 로직 이동

```java
// Before: 취소 로직이 Service에
@Service
public class OrderService {
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(OrderNotFoundException::new);

        // ❌ 상태 전이 검증이 Service에
        if ("SHIPPED".equals(order.getStatus()) || "DELIVERED".equals(order.getStatus())) {
            throw new CannotCancelException("배송 시작 후 취소 불가");
        }
        order.setStatus("CANCELLED");
        orderRepository.save(order);
    }
}

// Step 1: Order에 cancel() 메서드 추가
public class Order {
    // ...
    public void cancel() {
        if ("SHIPPED".equals(this.status) || "DELIVERED".equals(this.status))
            throw new CannotCancelException("배송 시작 후 취소가 불가합니다");

        this.status = "CANCELLED";
        this.cancelledAt = LocalDateTime.now();
    }
}

// Step 2: Service 간소화
@Service
public class OrderService {
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(OrderNotFoundException::new);
        order.cancel(); // 비즈니스 로직을 Order에 위임
        orderRepository.save(order);
    }
}

// Step 3: 상태 전이 단위 테스트
class OrderTest {
    @Test
    void 배송_후_취소_불가() {
        Order order = createOrderWithStatus("SHIPPED");
        assertThatThrownBy(order::cancel)
            .isInstanceOf(CannotCancelException.class);
    }

    @Test
    void PLACED_상태_취소_가능() {
        Order order = createOrderWithStatus("PLACED");
        order.cancel();
        assertThat(order.getStatus()).isEqualTo("CANCELLED");
    }
}
```

### 4. UseCase 인터페이스 도입

```java
// UseCase 인터페이스 도입으로 진입점 명확화

// Step 1: UseCase 인터페이스 정의 (domain/port/in/)
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

public interface CancelOrderUseCase {
    void cancelOrder(CancelOrderCommand command);
}

// Step 2: Command 객체 정의 (HTTP DTO와 분리)
public record PlaceOrderCommand(
    String userId,
    List<OrderLineCommand> lines
) {
    public record OrderLineCommand(String itemId, int quantity, long unitPrice) {}
}

// Step 3: OrderService → PlaceOrderService 분리 (각 UseCase별)
@Service
@Transactional
@RequiredArgsConstructor
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderRepository orderRepository;
    private final PaymentPort paymentPort;

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command.userId(), mapLines(command.lines()));
        order.place();
        PaymentResult payment = paymentPort.charge(PaymentRequest.of(order));
        order.confirmPayment(payment.transactionId());
        orderRepository.save(order);
        return OrderId.of(order.getId());
    }
}

@Service
@Transactional
@RequiredArgsConstructor
public class CancelOrderService implements CancelOrderUseCase {

    private final OrderRepository orderRepository;

    @Override
    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId())
            .orElseThrow(OrderNotFoundException::new);
        order.cancel();
        orderRepository.save(order);
    }
}

// Step 4: Controller가 UseCase 인터페이스에 의존
@RestController
public class OrderController {
    private final PlaceOrderUseCase placeOrderUseCase; // 인터페이스에만 의존
    private final CancelOrderUseCase cancelOrderUseCase;

    @PostMapping("/api/orders")
    public ResponseEntity<?> placeOrder(@RequestBody PlaceOrderRequest req) {
        PlaceOrderCommand command = PlaceOrderCommand.from(req);
        OrderId id = placeOrderUseCase.placeOrder(command);
        return ResponseEntity.created(...).body(new PlaceOrderResponse(id.value()));
    }
}
```

### 5. Spring 의존성 제거 과정

```java
// Domain에서 Spring 의존성 제거 (단계별)

// Before: Domain Entity에 Spring 어노테이션 혼재
@Entity  // JPA — 나중에 분리
@Table(name = "orders")
@EntityListeners(AuditingEntityListener.class) // Spring Data Auditing
public class Order {
    @CreatedDate  // Spring Data
    private LocalDateTime createdAt;

    @LastModifiedDate  // Spring Data
    private LocalDateTime updatedAt;

    // 비즈니스 메서드들...
}

// Step 1: Spring Data Auditing 제거 → 직접 관리로
public class Order {
    // @CreatedDate, @LastModifiedDate 제거
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public void place() {
        // ...
        this.createdAt = LocalDateTime.now(); // 직접 설정
    }
}

// Step 2: @Table 정보는 JPA Entity 클래스에서 관리
// (Phase 5에서 Domain Entity와 JPA Entity 분리 시 처리)

// Step 3: 도메인에 남기는 허용 절충:
//   @Entity: 절충 허용 (JPA 제약이 도메인 오염 없으면)
//   @Transactional: Application Service에만

// Step 4: ArchUnit 규칙 추가 (Domain의 Spring 의존 감시)
@ArchTest
static final ArchRule domain_no_spring_data_auditing =
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .haveFullyQualifiedName("org.springframework.data.annotation.CreatedDate")
        .orShould().dependOnClassesThat()
        .haveFullyQualifiedName("org.springframework.data.annotation.LastModifiedDate")
        .because("Spring Data Auditing은 Domain 레이어에서 사용하지 않습니다. " +
                 "createdAt/updatedAt은 Domain 메서드에서 직접 설정합니다.");
```

---

## 💻 실전 코드 — 전체 리팩터링 커밋 순서 예시

```java
// === Week 2-3 실제 커밋 로그 예시 ===

// 커밋 1: Order 메서드 추가 (테스트와 함께)
// "refactor: Order.calculateTotal() 총액 계산을 Domain으로 이동"
public class Order {
    public BigDecimal calculateTotal() {
        return items.stream()
            .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
// + OrderTest.java: 총액 계산 단위 테스트 (0.001초)

// 커밋 2: Service에서 해당 로직 위임으로 변경
// "refactor: OrderService.placeOrder()에서 calculateTotal() 위임"
@Service
public class OrderService {
    public OrderResult placeOrder(...) {
        Order order = buildOrder(request);
        BigDecimal total = order.calculateTotal(); // 위임
        // 기존 계산 코드 삭제됨
        order.setTotalAmount(total);
        // ...
    }
}
// ✅ 기존 통합 테스트 통과 → 배포 가능

// 커밋 3: 검증 로직 이동
// "refactor: 최소 주문 금액 검증을 Order.place()로 이동"
public class Order {
    public void place() {
        if (calculateTotal().compareTo(MIN_AMOUNT) < 0)
            throw new MinOrderAmountException();
        this.status = "PLACED";
    }
}
// + OrderTest.java: 최소금액 검증 단위 테스트

// 커밋 4: Service 간소화
// "refactor: OrderService.placeOrder()에서 order.place() 위임"
@Service
public class OrderService {
    public OrderResult placeOrder(...) {
        Order order = buildOrder(request);
        order.place(); // 검증 + 상태 변경 위임
        orderRepository.save(order);
        // ...
    }
}
// ✅ 기존 통합 테스트 통과 → 배포 가능

// 커밋 5: UseCase 인터페이스 도입
// "refactor: PlaceOrderUseCase 인터페이스 추가 + OrderController 의존성 변경"
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}
// OrderController: @Autowired OrderService → @Autowired PlaceOrderUseCase

// 커밋 6: 명확한 Service 이름으로 분리
// "refactor: OrderService.placeOrder() → PlaceOrderService (UseCase별 분리)"
@Service
public class PlaceOrderService implements PlaceOrderUseCase {
    @Override
    public OrderId placeOrder(PlaceOrderCommand command) { ... }
}
// ✅ 기존 통합 테스트 통과 → 배포 가능
// ✅ PlaceOrderService 단위 테스트 추가 (InMemoryOrderRepository 사용)
```

---

## 📊 패턴 비교 — 리팩터링 전후 테스트 비교

```
OrderService 비즈니스 로직 테스트 비교:

=== 리팩터링 전 (로직이 Service에) ===
@SpringBootTest
class OrderServiceTest {
    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepository;

    @Test
    void 최소금액_미달() {
        // Spring Context 로딩 30초...
        PlaceOrderRequest req = /* ... */;
        assertThatThrownBy(() -> orderService.placeOrder(req))
            .isInstanceOf(MinOrderAmountException.class);
    }
}
실행 시간: 30초

=== 리팩터링 후 (로직이 Domain에) ===
class OrderTest {
    @Test
    void 최소금액_미달() {
        Order order = Order.create("user-1", lowAmountLines());
        assertThatThrownBy(order::place)
            .isInstanceOf(MinOrderAmountException.class);
    }
}
실행 시간: 0.001초 (30,000배 빠름)

=== 리팩터링 후 PlaceOrderService 단위 테스트 ===
class PlaceOrderServiceTest {
    InMemoryOrderRepository repo = new InMemoryOrderRepository();
    InMemoryPaymentPort payment = new InMemoryPaymentPort();
    PlaceOrderService service = new PlaceOrderService(repo, payment);

    @Test
    void 결제_실패_시_저장_안_됨() {
        payment.willFail();
        assertThatThrownBy(() -> service.placeOrder(validCommand()))
            .isInstanceOf(PaymentFailedException.class);
        assertThat(repo.count()).isEqualTo(0);
    }
}
실행 시간: 0.01초 (Spring 없음)
```

---

## ⚖️ 트레이드오프

```
도메인 로직 이동의 비용:
  Domain 객체 설계 결정 필요 (어떤 메서드를 어떻게 정의할 것인가)
  기존 Anemic Model에 익숙한 팀원 학습 필요
  정적 팩토리 메서드, 불변 설계 등 고려 사항 증가

도메인 로직 이동의 이익:
  비즈니스 규칙이 한 곳에 집중 → 중복 제거
  순수 Java 단위 테스트 가능 → 30초 → 0.001초
  "Order가 어떤 동작을 하는가"가 코드에서 명확

단계별 커밋의 이익:
  각 커밋이 독립적으로 배포 가능
  잘못됐을 때 특정 커밋으로 롤백
  "무엇을 왜 바꿨는가" 이력이 명확
```

---

## 📌 핵심 정리

```
도메인 레이어 분리 핵심:

이동 순서:
  1. 값 계산 로직 → Domain 메서드 (calculateTotal() 등)
  2. 비즈니스 검증 → Domain 메서드 (place() 내부)
  3. 상태 전이 → Domain 메서드 (cancel(), confirm() 등)
  4. UseCase 인터페이스 도입 → 진입점 명확화
  5. Service 분리 → 각 UseCase별 Service 클래스

각 단계 커밋 전략:
  이동 → 단위 테스트 추가 → 통합 테스트 통과 → 커밋

Spring 의존성 제거:
  @CreatedDate, @LastModifiedDate → 직접 설정
  @Autowired in Domain → 금지 (파라미터 주입으로)
  @Transactional → Application Service에만

성과:
  OrderService 27개 메서드 → PlaceOrderService 1개 + CancelOrderService 1개 등
  테스트: 0.001초 Domain 단위 테스트 가능
  ArchUnit: "Domain에 Spring 없음" 규칙 추가
```

---

## 🤔 생각해볼 문제

**Q1.** 도메인 로직을 Domain 객체로 이동할 때 "이 로직은 Domain인가, Application인가"를 구분하는 기준은?

<details>
<summary>해설 보기</summary>

**"도메인 전문가(비개발자)가 말하는 규칙인가?"로 판단합니다.**

Domain 로직 예시:
- "최소 주문 금액은 1,000원이다" → 비즈니스 정책 → Domain
- "배송 후 취소는 불가하다" → 비즈니스 규칙 → Domain
- "총 금액 = Σ(단가 × 수량)" → 비즈니스 계산 → Domain

Application 로직 예시:
- "주문 생성 시: 재고 확인 → 결제 → 저장 → 알림 순서" → 이 앱의 흐름 → Application
- "결제 실패 시 3번 재시도" → 기술적 처리 → Application

헷갈릴 때 질문:
- "다른 시스템(콜센터 주문, ERP)에서도 이 규칙이 적용되는가?" → YES이면 Domain
- "이 규칙이 변경될 때 도메인 전문가(기획자)가 결정하는가?" → YES이면 Domain

</details>

---

**Q2.** `Order.place()` 메서드가 추가됐는데, 기존에 직접 `order.setStatus("PLACED")`를 호출하는 레거시 코드가 남아 있다. 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**setStatus()를 private으로 만들어 외부에서 호출 불가하게 합니다.**

```java
public class Order {
    private String status; // private 필드

    // public setter 제거!
    // 기존: public void setStatus(String status) { this.status = status; }

    // 비즈니스 메서드를 통해서만 상태 변경 가능
    public void place() {
        // 검증 후
        this.status = "PLACED"; // 내부에서만 설정
    }

    public void cancel() {
        // 검증 후
        this.status = "CANCELLED";
    }
}
```

레거시 코드에서 `order.setStatus("PLACED")`를 호출하면 컴파일 에러 발생 → 강제로 `order.place()`를 쓰게 됨.

단계적 접근:
1. `setStatus()` deprecated 표시 (즉시)
2. 레거시 코드에서 `setStatus()` 호출을 `place()`/`cancel()`로 교체
3. 모두 교체 완료 후 `setStatus()` 제거 (private 또는 삭제)

이 과정에서 "어디에서 order.setStatus()를 호출하는가"를 IDE에서 찾아 전부 교체합니다.

</details>

---

**Q3.** 도메인 로직 이동 후 기존 `@SpringBootTest` 통합 테스트가 실패했다. 어떻게 디버깅하는가?

<details>
<summary>해설 보기</summary>

**리팩터링은 동작 변경이 없어야 합니다. 실패했다면 실수가 있는 것입니다.**

디버깅 순서:
1. **실패한 테스트 시나리오 파악**: 어떤 입력에서 어떤 결과가 달라졌는가
2. **최근 커밋 확인**: `git diff HEAD~1`로 변경 내용 검토
3. **단위 테스트로 재현**: 실패 시나리오를 Domain 단위 테스트로 재현
4. **비즈니스 로직 검토**: Service에 있던 로직이 Domain으로 올바르게 이동됐는가

일반적인 실수들:
```java
// 실수 1: Service에서 로직을 "제거"만 하고 Domain에 "추가"를 빠뜨림
// Service: 검증 코드 삭제됨
// Order: place()가 아직 빈 메서드
public void place() {
    this.status = "PLACED"; // 검증 없이 상태만 변경 → 통합 테스트에서 경계 케이스 실패
}

// 실수 2: 이동 중 로직 변경 (리팩터링 아님)
// Before: total < 1000
// After:  total <= 1000 (실수로 <= 로 변경)
```

황금 규칙: "리팩터링 커밋은 동작을 변경하지 않습니다." 통합 테스트 실패 = 동작 변경 = 버그 or 잘못된 리팩터링.

</details>

---

<div align="center">

**[⬅️ 이전: 점진적 리팩터링 전략](./02-strangler-fig-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인프라 레이어 분리 ➡️](./04-extracting-infrastructure-layer.md)**

</div>
