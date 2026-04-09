# MSA와 아키텍처 패턴 — Bounded Context 안에서의 내부 아키텍처

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 서비스 내부 아키텍처(Layered/Hexagonal/Clean)와 서비스 간 아키텍처(MSA)는 어떤 관계인가?
- 서비스 크기(Nano/Micro/Modular Monolith)에 따라 내부 아키텍처를 어떻게 선택하는가?
- DDD Bounded Context가 아키텍처 패턴 선택에 어떻게 영향을 주는가?
- 서비스 간 통신(REST API, 이벤트)이 Hexagonal의 Adapter로 어떻게 추상화되는가?
- 모노리스에서 MSA로 전환할 때 내부 아키텍처가 어떤 역할을 하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

MSA를 도입한다고 해서 내부 아키텍처 문제가 자동으로 해결되지 않는다. 서비스를 잘게 나눠도 각 서비스 내부가 레이어드의 Fat Service 구조라면, 작은 서비스들 각각이 유지보수 불가능한 상태가 된다.

반대로 모노리스라도 내부를 Bounded Context 기준으로 잘 분리하고 Hexagonal을 적용하면, 나중에 MSA로 전환할 때 각 Bounded Context를 독립 서비스로 분리하는 것이 자연스러워진다. **내부 아키텍처가 MSA 전환 비용을 결정한다.**

---

## 😱 흔한 실수 (Before — MSA와 내부 아키텍처를 혼동할 때)

```
실수 1: MSA = 자동으로 좋은 아키텍처라는 착각

  서비스를 OrderService, PaymentService, UserService로 분리
  각 서비스 내부:
    OrderService 내부:
      OrderController → OrderService → OrderRepository (레이어드 기본)
      OrderService: 600줄 Fat Service, 단위 테스트 없음
  
  결과:
    서비스는 분리됐지만 각각이 모놀리스의 문제를 그대로 복제
    분산 Fat Service의 탄생: 더 복잡하고 더 나쁨

실수 2: 내부 경계 없이 MSA 전환 시도

  모노리스 OrderService에 주문, 결제, 재고, 알림 로직이 모두 있음
  "이걸 4개 서비스로 나누자"
  → 경계가 없으니 어디서 잘라야 할지 모름
  → 잘못 자르면 두 서비스가 항상 같이 배포됨 (분산 모놀리스)

올바른 순서:
  내부 Bounded Context 분리 먼저
  → 경계가 명확해지면 MSA 전환이 자연스러움
```

---

## ✨ 올바른 접근 (After — 서비스 크기별 내부 아키텍처 전략)

```
서비스 크기와 내부 아키텍처 매핑:

Nano Service (단일 기능, 단일 UC):
  "결제 처리만 하는 서비스"
  내부: 레이어드+DIP 충분
  이유: 도메인이 단순, 하나의 UseCase만 존재

Micro Service (하나의 Bounded Context):
  "주문 관리 서비스" (주문 생성, 수정, 취소, 조회)
  내부: Hexagonal 권장
  이유: 여러 UseCase, 외부 서비스 연동, 단위 테스트 중요

Modular Monolith (여러 Bounded Context, 단일 배포):
  "전체 이커머스 시스템" (주문 + 결제 + 재고 + 알림)
  내부: 각 모듈에 Hexagonal + 모듈 간 API 경계
  이유: 나중에 MSA 전환 시 모듈 = 서비스로 매핑

전환 경로:
  Modular Monolith → 트래픽 증가 시 특정 모듈만 MSA로 분리
  내부 Hexagonal 구조가 있으면 분리 비용이 낮음
```

---

## 🔬 내부 원리 — MSA + 내부 아키텍처 통합

### 1. DDD Bounded Context와 아키텍처 패턴의 관계

```
DDD Bounded Context: "용어와 모델이 일관되게 적용되는 경계"

이커머스 시스템의 Bounded Context:
  ┌─────────────────────────────────────────────────────────────┐
  │                     이커머스 시스템                           │
  │ ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐  │
  │ │   Order BC       │  │   Payment BC    │  │ Inventory BC│  │
  │ │  Order, OrderLine│  │ Payment, Refund │  │ Stock, Item │  │
  │ │  "주문" 개념     │  │ "결제" 개념     │  │ "재고" 개념 │  │
  │ └─────────────────┘  └─────────────────┘  └─────────────┘  │
  │                                                              │
  │ ┌─────────────────┐  ┌─────────────────┐                   │
  │ │   User BC        │  │ Notification BC │                   │
  │ │ User, Address    │  │ Email, SMS      │                   │
  │ └─────────────────┘  └─────────────────┘                   │
  └─────────────────────────────────────────────────────────────┘

각 BC 내부 아키텍처 선택:
  Order BC: 복잡한 상태 전이, 할인 정책 → Hexagonal
  Payment BC: 외부 결제 API 연동, 교체 가능성 → Hexagonal (Port로 추상화)
  Inventory BC: 재고 관리, 동시성 이슈 → Hexagonal (복잡한 비즈니스)
  User BC: 단순 CRUD + 인증 → 레이어드+DIP
  Notification BC: 발송 수단 교체 예상 → 레이어드+DIP + NotificationPort

BC와 서비스의 매핑 (MSA에서):
  원칙: "하나의 BC = 하나의 서비스" (이상적)
  현실: "하나의 서비스에 여러 BC" (Modular Monolith)
        → 나중에 BC별로 서비스 분리

BC 경계 = 서비스 경계:
  내부 아키텍처가 BC 경계를 잘 분리해뒀다면
  MSA 전환 시 BC 폴더를 독립 서비스로 이동만 하면 됨
```

### 2. 서비스 간 통신이 Hexagonal Adapter로 추상화되는 방식

```java
// MSA에서 Order Service가 Payment Service를 호출해야 할 때

// ❌ 잘못된 방식: Order 도메인이 Payment 서비스를 직접 알음
@Service
public class PlaceOrderService {
    @Autowired
    RestTemplate restTemplate; // HTTP 클라이언트 직접 주입

    public OrderId placeOrder(PlaceOrderCommand cmd) {
        Order order = Order.create(cmd.userId(), cmd.lines());
        order.place();

        // Payment Service 직접 호출 — Order 도메인이 HTTP를 앎
        PaymentResponse response = restTemplate.postForObject(
            "http://payment-service/v1/payments",
            new PaymentRequest(order.getId().value(), order.calculateTotal()),
            PaymentResponse.class
        );
        order.confirmPayment(response.getTransactionId());
        orderRepository.save(order);
        return order.getId();
    }
}

// ✅ 올바른 방식: Hexagonal Port로 추상화

// Order 도메인 내부의 Driven Port (어떤 서비스인지 모름)
// order/domain/port/out/PaymentPort.java
public interface PaymentPort {
    PaymentResult charge(PaymentRequest request);
}

// Order 도메인 내부의 Application Service
@Service
public class PlaceOrderService implements PlaceOrderUseCase {
    private final PaymentPort paymentPort; // Port만 앎

    public OrderId placeOrder(PlaceOrderCommand cmd) {
        Order order = Order.create(cmd.userId(), cmd.lines());
        order.place();

        // Port를 통해 결제 (HTTP인지 이벤트인지 모름)
        PaymentResult result = paymentPort.charge(PaymentRequest.of(order));
        order.confirmPayment(result.transactionId());
        orderSavePort.save(order);
        return order.getId();
    }
}

// Driven Adapter: Payment Service HTTP 호출 (Infrastructure)
// order/adapter/out/payment/PaymentServiceAdapter.java
@Component
public class PaymentServiceAdapter implements PaymentPort {

    private final PaymentServiceClient client; // HTTP 클라이언트

    @Override
    public PaymentResult charge(PaymentRequest request) {
        // Order 도메인 모델 → Payment Service API 형식 변환 (Anti-Corruption Layer)
        PaymentApiRequest apiRequest = PaymentApiRequest.builder()
            .partnerOrderId(request.orderId())
            .totalAmount(request.amount().getValue())
            .build();

        PaymentApiResponse response = client.charge(apiRequest);

        // Payment Service 응답 → Order 도메인 모델 변환
        return PaymentResult.of(response.getTransactionId(), response.isSuccess());
    }
}

// Driven Adapter: 이벤트 기반으로 교체 (동기 → 비동기 전환 시)
// order/adapter/out/payment/EventBasedPaymentAdapter.java
@Component
@ConditionalOnProperty("payment.mode", havingValue = "async")
public class EventBasedPaymentAdapter implements PaymentPort {

    private final KafkaTemplate<String, String> kafkaTemplate;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        // Kafka 이벤트 발행 (비동기)
        kafkaTemplate.send("payment.requested",
            request.orderId(),
            serialize(PaymentRequestedEvent.from(request))
        );
        // 비동기의 경우 즉시 반환 (결과는 나중에 이벤트로)
        return PaymentResult.pending(request.orderId());
    }
}

// 동기 → 비동기 전환: PlaceOrderService 코드 변경 없음!
// application.yml: payment.mode: async → EventBasedPaymentAdapter 자동 선택
```

### 3. 모노리스에서 MSA로 전환 — 내부 아키텍처의 역할

```
전환 시나리오: 모노리스 이커머스 → 결제 서비스 분리

=== 내부 아키텍처가 없을 때 (레이어드 기본) ===

OrderService.java (1200줄):
  placeOrder() {
      // 주문 로직
      order.setStatus("PLACED");
      // 결제 로직 (직접 코드)
      BigDecimal total = calculateTotal(items);
      kakaoPayClient.charge(userId, total);
      // 재고 로직 (직접 코드)
      inventoryRepository.decreaseStock(items);
      // 알림 로직
      emailSender.sendOrderConfirmation(userId, order);
      orderRepository.save(order);
  }

결제 서비스 분리 시도:
  문제 1: 결제 로직이 OrderService 중간에 섞여 있음 → 어디서 잘라야 하는지 모름
  문제 2: 결제, 재고, 알림 로직이 하나의 트랜잭션 → 분리 시 분산 트랜잭션 필요
  문제 3: 테스트 없음 → 분리 후 동작 보장 없음
  결과: 수개월의 대규모 리팩터링 필요

=== 내부 아키텍처가 있을 때 (Hexagonal) ===

PlaceOrderService.java:
  placeOrder() {
      Order order = Order.create(userId, lines);
      order.place(); // 주문 로직: Order 도메인 내부

      // 결제: PaymentPort 통해 호출
      PaymentResult payment = paymentPort.charge(PaymentRequest.of(order));
      order.confirmPayment(payment.transactionId());

      // 재고: InventoryPort 통해 호출
      inventoryPort.decreaseStock(order.getLines());

      // 이벤트: 알림은 OrderPlacedEvent 구독
      orderSavePort.save(order);
      eventPublisher.publish(new OrderPlacedEvent(order));
  }

결제 서비스 분리 시:
  Step 1: PaymentPort 구현체를 PaymentService HTTP 클라이언트로 교체
          (KakaoPayAdapter → PaymentServiceAdapter)
  Step 2: PlaceOrderService 코드 변경 없음
  Step 3: 단위 테스트가 이미 있으므로 분리 후 동작 검증 가능

전환 비용: 수개월 → 수주
```

### 4. 서비스 크기별 내부 아키텍처 선택

```
Nano Service (단일 UseCase):
  예: "이미지 리사이징 서비스", "단가 계산 서비스"
  도메인 복잡도: 최소
  외부 연동: 1개 이하
  
  권장 구조:
    레이어드+DIP 충분
    UseCase 인터페이스도 불필요
    Controller → Service → Repository(인터페이스)
    테스트: 단위 테스트 20%, E2E 80%

Micro Service (단일 Bounded Context):
  예: "주문 서비스", "결제 서비스"
  도메인 복잡도: 중간-높음
  외부 연동: 2-5개

  권장 구조:
    Hexagonal 완전 적용
    각 외부 서비스 연동: 별도 Port와 Adapter
    테스트: 단위 70%, 통합 25%, E2E 5%

Modular Monolith (여러 Bounded Context):
  예: "스타트업 초기 이커머스 전체"
  모듈: Order, Payment, Inventory, User, Notification
  
  권장 구조:
    각 모듈: 독립 패키지 + Hexagonal 내부
    모듈 간: 인터페이스 또는 이벤트만 통신 (직접 import 금지)
    나중에 MSA 전환: 모듈 → 독립 서비스로 분리

  패키지 구조:
    com.example
      order/ ← Hexagonal
      payment/ ← Hexagonal
      inventory/ ← Hexagonal
      user/ ← 레이어드+DIP (단순 CRUD)
      notification/ ← 레이어드+DIP (이벤트 수신 후 발송)
    
    모듈 간 통신:
      order → payment: PaymentPort (Hexagonal Driven Port)
      payment → notification: OrderPaidEvent (Spring 이벤트 또는 Kafka)
      모듈 간 직접 클래스 import: ArchUnit으로 금지
```

### 5. 이벤트 기반 통신과 Adapter 추상화

```java
// 이벤트 기반 MSA에서 Order Service → Notification Service 통신

// Order Service 내부: 이벤트 발행 Port
// order/domain/port/out/OrderEventPublisher.java
public interface OrderEventPublisher {
    void publishOrderPlaced(OrderPlacedEvent event);
}

// Order Service Application Service: Port만 사용
@Service @Transactional
public class PlaceOrderService implements PlaceOrderUseCase {

    private final OrderEventPublisher eventPublisher; // Port만 앎

    public OrderId placeOrder(PlaceOrderCommand cmd) {
        // ...
        orderSavePort.save(order);
        // 어떤 방식으로 발행될지 모름 (Kafka? Spring 이벤트? RabbitMQ?)
        eventPublisher.publishOrderPlaced(OrderPlacedEvent.from(order));
        return order.getId();
    }
}

// Driven Adapter: Kafka로 발행 (Production)
@Component
@Profile("!test")
public class KafkaOrderEventPublisher implements OrderEventPublisher {

    private final KafkaTemplate<String, String> kafkaTemplate;

    @Override
    public void publishOrderPlaced(OrderPlacedEvent event) {
        String payload = serialize(OrderPlacedMessage.from(event));
        kafkaTemplate.send("order.placed", event.orderId().value(), payload);
    }
}

// Driven Adapter: InMemory (테스트용)
@Component
@Profile("test")
public class InMemoryOrderEventPublisher implements OrderEventPublisher {

    private final List<OrderPlacedEvent> publishedEvents = new ArrayList<>();

    @Override
    public void publishOrderPlaced(OrderPlacedEvent event) {
        publishedEvents.add(event);
    }

    public List<OrderPlacedEvent> getPublishedEvents() { return publishedEvents; }
}

// Notification Service: 이벤트 수신
@Component
public class OrderPlacedEventConsumer {

    private final SendOrderNotificationUseCase notificationUseCase;

    @KafkaListener(topics = "order.placed")
    public void handle(OrderPlacedMessage message) {
        // Driving Adapter: Kafka Consumer가 Notification UseCase 호출
        notificationUseCase.sendConfirmation(SendNotificationCommand.from(message));
    }
}

// 교체 가능성:
// Kafka → RabbitMQ: KafkaOrderEventPublisher → RabbitMQOrderEventPublisher (1파일)
// PlaceOrderService: 변경 없음
// Notification Service: Consumer만 변경
```

---

## 💻 실전 코드 — Modular Monolith 내부 구조

```java
// Modular Monolith: 모듈 간 경계를 ArchUnit으로 강제

@AnalyzeClasses(packages = "com.example")
public class ModularMonolithArchTest {

    /**
     * 모듈 간 직접 내부 클래스 import 금지
     * 모든 통신은 각 모듈이 노출한 Port 또는 이벤트를 통해
     */
    @Test
    void order_module_should_not_directly_import_payment_internals() {
        noClasses()
            .that().resideInAPackage("..order..")
            .should().dependOnClassesThat()
            .resideInAPackage("..payment.domain..")  // payment 도메인 내부 직접 접근 금지
            .check(classes);
        // order가 payment의 Payment.java를 직접 import하면 안 됨
        // PaymentPort(인터페이스)를 통해서만 통신 가능
    }

    @Test
    void modules_should_communicate_only_through_ports_or_events() {
        // 허용: order → PaymentPort (인터페이스)
        // 허용: order → OrderPlacedEvent (이벤트 DTO, shared 패키지)
        // 금지: order → Payment (구체 클래스)
        // 금지: order → PaymentService (다른 모듈의 Service)
        
        noClasses()
            .that().resideInAPackage("..order..")
            .should().dependOnClassesThat()
            .resideInAnyPackage(
                "..payment.application..",     // 다른 모듈 Application
                "..payment.adapter..",          // 다른 모듈 Adapter
                "..payment.domain.model.."      // 다른 모듈 Domain Model
            )
            .check(classes);
    }
}

// 각 모듈이 노출하는 공개 API (Facade 패턴)
// payment/PaymentModuleApi.java (다른 모듈에서 접근 가능한 유일한 진입점)
public interface PaymentModuleApi {
    PaymentResult charge(String orderId, long amount);
    void refund(String transactionId, long amount);
}

// order 모듈의 PaymentPort는 PaymentModuleApi를 래핑
// order/adapter/out/payment/PaymentModuleAdapter.java
@Component
public class PaymentModuleAdapter implements PaymentPort {
    private final PaymentModuleApi paymentApi; // 모듈 공개 API만 사용

    @Override
    public PaymentResult charge(PaymentRequest request) {
        return paymentApi.charge(request.orderId(), request.amount().getValue());
    }
}
```

---

## 📊 패턴 비교 — 서비스 크기별 내부 아키텍처 선택

```
서비스 크기         │ 내부 아키텍처   │ 이유
───────────────────┼───────────────┼─────────────────────────────────
Nano Service        │ 레이어드+DIP   │ 도메인 단순, 빠른 개발 우선
(단일 UC)           │               │
                   │               │
Micro Service       │ Hexagonal     │ 여러 UseCase, 외부 연동 교체 예상
(단일 BC)           │               │ 단위 테스트 속도 중요
                   │               │
Modular Monolith    │ 모듈별 선택    │ 핵심 BC: Hexagonal
(여러 BC, 단일 배포) │               │ 지원 BC: 레이어드+DIP
                   │               │ 모듈 간: Port/이벤트 통신
                   │               │
MSA                 │ 서비스별 선택  │ 각 서비스 = Micro Service 선택

공통 원칙:
  BC 경계 명확하게 → 나중에 서비스 분리 용이
  서비스 간 통신: Port 추상화 → 동기↔비동기 전환 비용 최소화
```

---

## ⚖️ 트레이드오프

```
MSA + Hexagonal의 복잡도:
  각 서비스마다 Port/Adapter 구조
  서비스 간 통신마다 Anti-Corruption Layer (번역 계층)
  분산 트랜잭션 처리 (Saga, Outbox Pattern)

MSA + 레이어드의 단순성:
  서비스 내부 단순
  그러나 서비스가 많아지면 각각이 Fat Service 위험

Modular Monolith 장점:
  MSA의 서비스 분리 장점 (경계 명확)
  분산 시스템의 복잡도 없음 (네트워크, 분산 트랜잭션)
  나중에 MSA로 전환 가능 (경계가 명확하므로)

현실적 권장:
  초기: Modular Monolith (경계 명확히 + Hexagonal)
  성장 후: 트래픽이 집중되는 모듈만 MSA 분리
  결국: 대부분의 성공적 MSA는 Modular Monolith에서 진화
```

---

## 📌 핵심 정리

```
MSA + 내부 아키텍처 핵심:

MSA != 좋은 아키텍처 자동 보장:
  서비스를 나눠도 내부가 Fat Service면 분산 Fat Service
  내부 아키텍처(Hexagonal)가 MSA 전환 비용을 결정

BC와 서비스 매핑:
  Bounded Context = 이상적인 서비스 경계
  BC 내부: Hexagonal (복잡한 BC) or 레이어드+DIP (단순 BC)

서비스 간 통신 추상화:
  Driven Port(PaymentPort)로 다른 서비스 통신 추상화
  Adapter 교체로 동기(REST) ↔ 비동기(Kafka) 전환 가능

전환 경로:
  모노리스 → Modular Monolith → 필요 시 MSA 분리
  내부 Hexagonal 구조가 있으면 모듈 → 서비스 이동이 자연스러움

Modular Monolith 내부 규칙:
  모듈 간 직접 클래스 import 금지 (ArchUnit 강제)
  통신: Port 인터페이스 or 이벤트만 허용
  각 모듈이 공개 API(Facade) 노출
```

---

## 🤔 생각해볼 문제

**Q1.** "서비스 크기가 작을수록 더 좋다"는 Nano Service 사고방식의 문제점은 무엇인가?

<details>
<summary>해설 보기</summary>

**Nano Service(과도한 분해)의 문제:**

1. **분산 트랜잭션 폭발**: 주문 생성 = 주문서비스 + 결제서비스 + 재고서비스 + 알림서비스가 모두 연관 → 하나라도 실패 시 롤백이 복잡 (Saga 패턴 필요)

2. **네트워크 오버헤드**: 서비스 하나가 처리할 것을 3개 서비스가 HTTP로 주고받음 → 레이턴시 증가, 장애 포인트 증가

3. **로컬 디버깅 불가**: 로컬 개발 환경에서 10개 서비스를 동시 실행 → Docker Compose가 거대해짐

4. **배포 조율 복잡**: 같이 배포해야 하는 서비스들이 생김 → 분산 모놀리스로 역행

**적절한 서비스 크기 기준:**
- "팀이 독립적으로 배포할 수 있는가?" → 독립 배포 가능 단위가 서비스 경계
- "하나의 팀이 소유할 수 있는 크기인가?" → 팀이 이해하고 운영할 수 있어야 함
- "변경 이유가 단일한가?" → 같이 바뀌는 것들은 같은 서비스에

</details>

---

**Q2.** 모노리스에서 MSA로 전환할 때 "어떤 BC를 먼저 분리해야 하는가"의 기준은?

<details>
<summary>해설 보기</summary>

**트래픽 + 변경 빈도 + 독립 배포 필요성 기준으로 결정합니다.**

우선 분리 대상 (하나라도 해당하면):
1. **독립 스케일 필요**: "결제 처리가 병목 → 결제 서비스만 증설하고 싶다"
2. **다른 팀이 소유**: "결제팀과 주문팀이 분리됐다 → 배포 독립이 필요"
3. **빠른 변경 주기**: "알림 서비스는 매주 바뀌지만 다른 서비스와 독립적"
4. **기술 스택 분리**: "ML 기반 추천 엔진은 Python이 필요"

나중에 분리해도 되는 것:
- 모노리스에서도 성능 충분한 서비스
- 다른 BC와 강하게 결합된 서비스 (분리 전 경계 정리 필요)
- 팀이 단일 배포를 선호하는 서비스

**내부 Hexagonal 구조가 있다면**: 결정한 BC를 분리할 때 해당 패키지를 독립 서비스로 이동 + Port 구현체를 HTTP/이벤트 Adapter로 교체하면 됩니다.

</details>

---

**Q3.** 서비스 간 통신을 동기(REST)에서 비동기(Kafka)로 전환할 때 Hexagonal의 Port 추상화가 어떻게 도움이 되는가? 완전히 무비용 전환인가?

<details>
<summary>해설 보기</summary>

**Port 추상화 덕분에 Application 코드 변경 없이 Adapter만 교체됩니다.** 단, 완전 무비용은 아닙니다.

**변경 없는 것 (Port 덕분에):**
```java
// PlaceOrderService: 변경 없음
PaymentResult result = paymentPort.charge(request); // Port 호출은 동일
```

**변경이 필요한 것:**
1. **새 Adapter 작성**: `EventBasedPaymentAdapter` (Kafka 발행 로직)
2. **비동기 결과 처리**: 동기는 즉시 결과 반환, 비동기는 나중에 이벤트로
   → `PlaceOrderService`가 "결과를 기다리는 방식"이 변함 (이것은 비즈니스 플로우 변경)
3. **에러 처리**: 동기는 예외로, 비동기는 실패 이벤트로
4. **Outbox Pattern 추가**: 이벤트 유실 방지를 위해

**결론**: 인프라 코드(Adapter)는 교체로 완결되지만, 비동기 처리 방식이 비즈니스 플로우에 영향을 주면 Application 코드도 일부 수정이 필요합니다. "완전 무비용"은 순수 인프라 교체(KakaoPay → TossPay)에 해당하고, 동기↔비동기 전환은 "최소 비용"에 해당합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 아키텍처 결정 기록(ADR)](./04-architecture-decision-records.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 패키지 구조 ➡️](../package-structure/01-package-by-layer-vs-feature.md)**

</div>
