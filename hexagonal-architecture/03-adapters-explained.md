# 어댑터의 역할 — Driving Adapter와 Driven Adapter

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Driving Adapter(Controller, CLI, Test)가 UseCase를 호출하는 방식의 공통점은 무엇인가?
- Driven Adapter(JpaRepository, KafkaPublisher, InMemoryRepository)가 Port를 구현하는 방식은?
- Adapter 교체 가능성의 실제 의미와 비용은 무엇인가?
- Adapter가 Domain 코드를 절대 포함하면 안 되는 이유는?
- InMemory Adapter는 언제 만들고, 어떤 수준으로 충실하게 구현해야 하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Adapter는 Hexagonal Architecture에서 "외부 세계와 내부 도메인을 연결하는 번역기"다. HTTP 요청을 Command로, JPA Entity를 Domain 객체로 변환한다.

Adapter의 책임이 명확하지 않으면 비즈니스 로직이 Adapter에 스며들기 시작한다. Controller에 `if`문이 생기거나 JpaRepository에 비즈니스 조건이 들어간다. Adapter는 오직 **변환(Translation)**만 해야 한다.

---

## 😱 흔한 실수 (Before — Adapter에 비즈니스 로직이 스며들 때)

```
증상 1: Controller(Driving Adapter)에 비즈니스 로직
  @RestController
  public class OrderController {
      @PostMapping("/orders")
      public ResponseEntity<?> placeOrder(@RequestBody PlaceOrderRequest req) {
          // ❌ 비즈니스 로직이 Controller에
          if (req.getTotalAmount().compareTo(BigDecimal.valueOf(1000)) < 0) {
              return ResponseEntity.badRequest().body("최소 1000원 이상");
          }
          if (req.getItems() == null || req.getItems().isEmpty()) {
              return ResponseEntity.badRequest().body("항목 없음");
          }
          // UseCase 호출
          placeOrderUseCase.placeOrder(PlaceOrderCommand.from(req));
      }
  }
  문제: 최소 금액 검증이 Controller에 있어서 CLI, Batch에서 재사용 불가

증상 2: JpaRepository(Driven Adapter)에 비즈니스 조건
  @Repository
  public class JpaOrderRepository implements OrderRepository {
      public List<Order> findActiveOrders(UserId userId) {
          // ❌ "활성 주문"의 비즈니스 정의가 Adapter에
          return jpa.findByUserIdAndStatusIn(
              userId.value(),
              List.of("PLACED", "PAYMENT_CONFIRMED", "SHIPPING")
              // "활성 주문"의 기준이 바뀌면 Adapter 수정 → 도메인 변경이 인프라에 영향
          );
      }
  }

증상 3: KafkaPublisher(Driven Adapter)에서 메시지 변환 이외의 로직
  @Component
  public class KafkaOrderPublisher implements OrderEventPublisher {
      public void publish(OrderPlacedEvent event) {
          // ❌ 재시도 로직이 아닌 비즈니스 판단이 Adapter에
          if (event.getOrder().isInternational()) {
              kafka.send("international-orders", ...);
          } else {
              kafka.send("domestic-orders", ...);
          }
          // "국제 주문인가"는 도메인 판단이므로 Domain에 있어야 함
      }
  }
```

---

## ✨ 올바른 접근 (After — Adapter는 변환만)

```
Adapter의 유일한 책임: 변환(Translation)

Driving Adapter (Controller):
  HTTP Request → Command (도메인 입력으로 변환)
  Result → HTTP Response (도메인 출력을 HTTP로 변환)
  비즈니스 판단: 전혀 없음 → Domain에 위임

Driven Adapter (JpaRepository):
  Domain Object → JPA Entity (저장을 위한 변환)
  JPA Entity → Domain Object (조회 후 변환)
  비즈니스 조건: 전혀 없음 → Domain Port의 계약만 구현

Driven Adapter (KafkaPublisher):
  Domain Event → Kafka Message (발행을 위한 변환)
  어떤 토픽에 보낼지: Domain Event가 타입으로 결정
  비즈니스 판단: 없음
```

---

## 🔬 내부 원리 — 각 Adapter의 구조

### 1. Driving Adapter — 외부에서 Application Core로

```
Driving Adapter의 역할:
  외부 요청(HTTP, CLI, Test, Batch) → Driving Port(UseCase) 호출

공통 패턴:
  1. 외부 입력을 수신 (HTTP Body, CLI Args, etc.)
  2. Domain Input(Command)으로 변환
  3. Driving Port 호출
  4. 결과(Result)를 외부 형식으로 변환 (HTTP Response, etc.)

HTTP Driving Adapter (Controller):
  @RestController
  public class OrderController {
      private final PlaceOrderUseCase placeOrderUseCase; // Driving Port

      @PostMapping("/api/orders")
      public ResponseEntity<PlaceOrderResponse> placeOrder(
          @Valid @RequestBody PlaceOrderRequest request,
          @AuthenticationPrincipal UserPrincipal principal
      ) {
          // 1. HTTP → Command (변환)
          PlaceOrderCommand command = PlaceOrderCommand.builder()
              .userId(UserId.of(principal.getUserId()))
              .lines(request.getItems().stream()
                  .map(i -> OrderLineCommand.of(i.getItemId(), i.getQuantity()))
                  .toList())
              .build();

          // 2. Driving Port 호출
          OrderId orderId = placeOrderUseCase.placeOrder(command);

          // 3. Result → HTTP Response (변환)
          return ResponseEntity
              .created(URI.create("/api/orders/" + orderId.value()))
              .body(PlaceOrderResponse.of(orderId));
      }
      // 비즈니스 로직: 없음
  }

CLI Driving Adapter:
  @Component
  public class PlaceOrderCLICommand implements CommandLineRunner {
      private final PlaceOrderUseCase placeOrderUseCase;

      @Override
      public void run(String... args) throws Exception {
          // 1. CLI Args → Command (변환)
          PlaceOrderCommand command = parseCLIArgs(args);

          // 2. Driving Port 호출 (HTTP와 동일한 Port!)
          OrderId orderId = placeOrderUseCase.placeOrder(command);

          // 3. Result → Console Output (변환)
          System.out.println("주문 완료: " + orderId.value());
      }
  }

Test Driving Adapter (= 단위 테스트):
  class PlaceOrderServiceTest {
      // Test가 직접 Application Core를 호출 = Test Driving Adapter
      @Test
      void 주문_생성_성공() {
          PlaceOrderCommand command = PlaceOrderCommand.of(userId, lines);
          OrderId orderId = placeOrderUseCase.placeOrder(command); // 직접 호출
          assertThat(orderId).isNotNull();
      }
  }
  // HTTP, CLI 없이 Application Core를 직접 테스트 가능!
```

### 2. Driven Adapter — Application Core에서 외부로

```
Driven Adapter의 역할:
  Driven Port(Repository, PaymentPort) 구현 → 외부 시스템 호출

공통 패턴:
  1. Driven Port 메서드 구현 (implements 선언)
  2. Domain Object → 외부 형식으로 변환 (JPA Entity, HTTP Request 등)
  3. 외부 시스템 호출
  4. 외부 응답 → Domain Object로 변환

JPA Driven Adapter:
  @Repository
  public class JpaOrderRepository implements OrderSavePort, OrderQueryPort {

      private final SpringDataOrderJpaRepository jpa;
      private final OrderMapper mapper; // Domain ↔ JPA Entity 변환

      @Override
      public void save(Order order) {
          // Domain Object → JPA Entity (변환)
          OrderJpaEntity entity = mapper.toJpaEntity(order);
          // JPA 저장
          jpa.save(entity);
      }

      @Override
      public Optional<Order> findById(OrderId id) {
          // JPA 조회
          Optional<OrderJpaEntity> entity = jpa.findById(id.value());
          // JPA Entity → Domain Object (변환)
          return entity.map(mapper::toDomain);
      }
  }

Kafka Driven Adapter:
  @Component
  public class KafkaOrderEventPublisher implements OrderEventPublisher {

      private final KafkaTemplate<String, String> kafkaTemplate;
      private final ObjectMapper objectMapper;

      @Override
      public void publish(OrderPlacedEvent event) {
          try {
              // Domain Event → Kafka Message (변환)
              String payload = objectMapper.writeValueAsString(
                  OrderPlacedMessage.from(event)
              );
              // Kafka 발행
              kafkaTemplate.send("order.placed", event.getOrderId().value(), payload);
          } catch (JsonProcessingException e) {
              throw new EventPublishException("이벤트 직렬화 실패", e);
          }
      }
  }

REST Driven Adapter (외부 API):
  @Component
  public class KakaoPayAdapter implements PaymentPort {

      private final KakaoPayClient client; // 외부 SDK
      private final PaymentMapper mapper;

      @Override
      public PaymentResult charge(PaymentRequest request) {
          // Domain Request → 외부 API 요청 형식 (변환)
          KakaoPayChargeRequest apiRequest = mapper.toKakaoPayRequest(request);
          // 외부 API 호출
          KakaoPayChargeResponse apiResponse = client.charge(apiRequest);
          // 외부 API 응답 → Domain Result (변환)
          return mapper.toPaymentResult(apiResponse);
      }
  }
```

### 3. InMemory Adapter — 테스트용 Driven Adapter

```
InMemory Adapter의 목적:
  테스트 시 실제 인프라(JPA, Kafka, 외부 API) 없이 Driven Port 구현

InMemory Adapter의 충실도 기준:
  Port 계약(인터페이스 시그니처)은 100% 준수
  실제 인프라 동작의 부분적 모사 (완전 동일 X)

InMemoryOrderRepository:
  public class InMemoryOrderRepository
      implements OrderSavePort, OrderQueryPort {

      private final Map<OrderId, Order> store = new LinkedHashMap<>();

      @Override
      public void save(Order order) {
          // 중요: 실제 JPA처럼 "영속성 컨텍스트에 의한 동일 객체" 효과는 없음
          // 하지만 계약(저장 성공)은 이행
          store.put(order.getId(), deepCopy(order)); // 불변성 모사
      }

      @Override
      public Optional<Order> findById(OrderId id) {
          return Optional.ofNullable(store.get(id))
              .map(this::deepCopy); // 다른 참조 반환 모사
      }

      @Override
      public List<Order> findByUserId(UserId userId) {
          return store.values().stream()
              .filter(o -> o.getUserId().equals(userId))
              .map(this::deepCopy)
              .toList();
      }

      // 테스트 헬퍼 메서드 (Port 계약 외)
      public int size() { return store.size(); }
      public boolean contains(OrderId id) { return store.containsKey(id); }
      public void clear() { store.clear(); }

      private Order deepCopy(Order order) {
          // 실제 프로젝트에서는 복사 방식 결정 필요
          return order; // 간단한 경우 참조 공유도 허용
      }
  }

InMemory Adapter가 테스트에서 Mock보다 나은 이유:
  Mock:
    verify(orderRepository).save(any()); // 호출 여부 검증 → 구현 결합
  
  InMemory:
    assertThat(orderRepository.contains(orderId)).isTrue(); // 상태 검증
    → 내부 구현(save vs saveAndFlush)이 바뀌어도 테스트 통과
```

### 4. Adapter 교체 가능성의 실제 의미와 비용

```
교체 가능성의 의미:
  "JPA를 MongoDB로 바꾸려면 JpaOrderRepository를 MongoOrderRepository로 교체"
  "KakaoPay를 TossPay로 바꾸려면 KakaoPayAdapter를 TossPayAdapter로 교체"
  → Application Core(Domain, UseCase) 코드 변경 없음

실제 교체 시나리오:

  KakaoPay → TossPay 교체:
  
  1. TossPayAdapter 새로 작성 (implements PaymentPort)
  2. Spring 빈 교체 (application.yml 프로파일 or @Conditional)
  3. 기존 KakaoPayAdapter는 삭제 or 유지
  
  변경되지 않는 것:
    PlaceOrderService — PaymentPort에만 의존하므로 변경 없음
    Order (Domain) — 변경 없음
    PlaceOrderUseCase — 변경 없음

  실제 비용:
    TossPayAdapter 작성: TossSDK 연동 코드 (실제 비용)
    Spring 설정 변경: @Primary or @ConditionalOnProperty
    TossPayAdapterTest 작성: TossSDK Mock 또는 TestContainers

교체 가능성의 한계 (솔직한 평가):
  "JPA → MongoDB 교체"를 위해:
    JPA Entity → MongoDB Document 스키마 설계
    JPQL → MongoDB Query 변환
    트랜잭션 처리 방식 변경
    → 실제로는 적지 않은 작업량

  "교체가 쉽다"는 것은:
    Application Core(Domain)를 건드리지 않아도 된다는 의미
    Infrastructure 코드는 교체가 필요 (당연)
    완전 무비용 교체가 아님

현실적 이익:
  Domain 로직 검증에 실제 DB/외부API 불필요 → 빠른 테스트
  한 Adapter가 실패해도 다른 Adapter 독립 테스트 가능
  새 기능 추가 시 Domain 코드 변경 없이 새 Adapter 추가 가능
```

---

## 💻 실전 코드 — Adapter 패턴 전체 구조

```java
// === Driving Adapter: Spring Batch (배치 처리) ===
@Component
public class ScheduledOrderProcessingJob {

    private final ProcessPendingOrdersUseCase processUseCase; // Driving Port

    @Scheduled(fixedDelay = 60_000)
    public void processPendingOrders() {
        // Batch 입력 → Command
        ProcessPendingOrdersCommand command = ProcessPendingOrdersCommand.of(
            LocalDateTime.now().minusHours(1) // 1시간 이상 된 대기 주문
        );

        // 동일한 Driving Port 호출 (HTTP Controller와 같은 Port)
        ProcessResult result = processUseCase.process(command);

        log.info("처리 완료: {} 건", result.getProcessedCount());
    }
}

// === Driven Adapter: Redis Cache ===
@Component
@ConditionalOnProperty("feature.cache.enabled", havingValue = "true")
public class RedisOrderQueryAdapter implements OrderQueryPort {

    private final RedisTemplate<String, String> redis;
    private final OrderQueryPort primaryAdapter; // 실제 DB Adapter
    private final ObjectMapper objectMapper;

    @Override
    public Optional<Order> findById(OrderId id) {
        String cacheKey = "order:" + id.value();

        // 캐시 조회
        String cached = redis.opsForValue().get(cacheKey);
        if (cached != null) {
            return Optional.of(deserialize(cached));
        }

        // 캐시 미스 → 실제 DB 조회
        Optional<Order> order = primaryAdapter.findById(id);

        // 캐시 저장
        order.ifPresent(o ->
            redis.opsForValue().set(cacheKey, serialize(o), Duration.ofMinutes(5))
        );

        return order;
    }
    // Domain Order ↔ Redis 직렬화 변환만 담당
}

// === 전체 구성: Spring 빈 설정 ===
@Configuration
public class OrderAdapterConfiguration {

    @Bean
    @Primary // 기본 Adapter
    public OrderSavePort orderSavePort(
        SpringDataOrderJpaRepository jpa, OrderMapper mapper
    ) {
        return new JpaOrderRepository(jpa, mapper);
    }

    @Bean
    @ConditionalOnProperty("payment.provider", havingValue = "kakao")
    public PaymentPort kakaoPayment(KakaoPayClient client) {
        return new KakaoPayAdapter(client);
    }

    @Bean
    @ConditionalOnProperty("payment.provider", havingValue = "toss")
    public PaymentPort tossPayment(TossPayClient client) {
        return new TossPayAdapter(client);
    }
    // application.yml의 payment.provider 값으로 Adapter 교체
}
```

---

## 📊 패턴 비교

```
Adapter 유형별 특성:

                    Driving Adapter         Driven Adapter
────────────────────┼───────────────────────┼─────────────────────────
역할                │ 외부 → Port 호출       │ Port 구현 → 외부 호출
예시                │ Controller, CLI, Test  │ JpaRepository, KafkaPublisher
Port와의 관계       │ Port를 호출하는 쪽     │ Port를 구현하는 쪽
비즈니스 로직       │ 없어야 함              │ 없어야 함
변환 방향           │ 외부입력 → Command     │ Domain → 외부형식
교체 시 영향        │ 다른 Driving Adapter   │ 다른 Driven Adapter
                    │ 에 영향 없음           │ 에 영향 없음
```

---

## ⚖️ 트레이드오프

```
Adapter 패턴의 비용:
  Mapper 코드: Domain ↔ JPA Entity 변환 (보일러플레이트)
  파일 수 증가: 각 외부 시스템마다 Adapter 파일
  InMemory Adapter 관리: 실제 동작과 일치하게 유지 필요

Adapter 패턴의 이익:
  Domain이 인프라를 완전히 모름
  외부 시스템 교체 시 Adapter만 변경
  InMemory Adapter로 빠른 단위 테스트
  각 Adapter 독립 테스트 가능

Mapper 비용 줄이기:
  MapStruct: Annotation 기반 코드 생성으로 매핑 자동화
  @Entity와 Domain Object를 통합: 매핑 불필요 (완전 분리의 장점은 포기)
```

---

## 📌 핵심 정리

```
Adapter = 변환(Translation) 전담 클래스

Driving Adapter:
  외부 입력 → Command → Driving Port 호출 → Response 변환
  비즈니스 로직: 없음
  예: Controller, CLI, Batch Job, Test

Driven Adapter:
  Driven Port 구현 → 외부 시스템 호출
  Domain Object ↔ 외부 형식 변환
  비즈니스 로직: 없음
  예: JpaRepository, KafkaPublisher, KakaoPayAdapter

InMemory Adapter:
  테스트용 Driven Adapter
  Port 계약은 100% 준수
  실제 인프라 없이 빠른 단위 테스트

교체 가능성의 현실:
  "Application Core 변경 없이 Adapter만 교체 가능"
  = 비즈니스 로직 코드는 건드리지 않아도 됨
  = 완전 무비용 교체가 아님 (Adapter 작성은 필요)
```

---

## 🤔 생각해볼 문제

**Q1.** Adapter가 비즈니스 로직을 포함하면 안 되는 이유를 구체적 예시로 설명하면?

<details>
<summary>해설 보기</summary>

Controller에 최소 금액 검증이 있다면, CLI Adapter나 Batch Adapter에도 같은 검증을 복사해야 합니다. 비즈니스 규칙의 중복이 발생하고 하나를 바꾸면 다른 곳도 바꿔야 합니다.

```java
// ❌ Controller(Adapter)에 비즈니스 검증
@PostMapping
public ResponseEntity<?> placeOrder(@RequestBody PlaceOrderRequest req) {
    if (req.getTotal() < 1000) return ResponseEntity.badRequest().body("최소 1000원"); // 비즈니스
    placeOrderUseCase.placeOrder(command);
}

// ❌ CLI(Adapter)에도 같은 검증 중복 필요
public void run(String... args) {
    if (total < 1000) { System.out.println("최소 1000원"); return; } // 중복!
    placeOrderUseCase.placeOrder(command);
}

// ✅ Domain(Order)에 검증
public void place() {
    if (total.isLessThan(Money.of(1000))) throw new MinOrderAmountException();
}
// Controller와 CLI 모두 UseCase를 호출하면 도메인에서 검증 → 중복 없음
```

</details>

---

**Q2.** InMemory Adapter의 구현이 실제 JPA와 100% 동일할 수 없다. 어떤 차이가 문제를 일으킬 수 있는가?

<details>
<summary>해설 보기</summary>

1. **영속성 컨텍스트**: JPA는 같은 트랜잭션 내 동일 ID → 같은 객체 반환. InMemory는 `new` 객체 반환 가능.
2. **LAZY 로딩**: JPA는 연관 관계가 LAZY면 실제 접근 시 DB 쿼리. InMemory는 항상 즉시 로딩.
3. **Cascade**: JPA는 cascade 설정에 따라 자동 처리. InMemory는 수동으로 구현.

해결 전략: InMemory는 비즈니스 로직 테스트에만 사용하고, JPA 관련 동작(@DataJpaTest or TestContainers)은 별도 테스트로 검증합니다.

</details>

---

**Q3.** Adapter 교체를 Spring 설정(@Conditional, @Profile)으로 하는 것이 코드상 어떻게 보이는가?

<details>
<summary>해설 보기</summary>

```java
// 환경별 Adapter 교체
@Configuration
public class PaymentConfiguration {

    // 운영 환경: TossPay
    @Bean
    @Profile("production")
    public PaymentPort productionPayment(TossPayClient client) {
        return new TossPayAdapter(client);
    }

    // 개발 환경: KakaoPay (테스트 모드)
    @Bean
    @Profile("development")
    public PaymentPort developmentPayment(KakaoPayClient client) {
        return new KakaoPayAdapter(client);
    }

    // 테스트 환경: InMemory
    @Bean
    @Profile("test")
    public PaymentPort testPayment() {
        return new InMemoryPaymentAdapter();
    }
}
```

`PlaceOrderService`는 어떤 프로파일에서도 `PaymentPort`에만 의존하므로 코드 변경 없이 다른 구현체가 주입됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: 주도하는 포트 vs 주도받는 포트](./02-driving-vs-driven-ports.md)** | **[홈으로 🏠](../README.md)** | **[다음: 의존성 방향 완전 분석 ➡️](./04-dependency-direction-analysis.md)**

</div>
