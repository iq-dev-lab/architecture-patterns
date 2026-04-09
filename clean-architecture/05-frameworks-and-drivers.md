# Frameworks & Drivers 레이어 — 가장 바깥에 위치해야 하는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JPA, Kafka, HTTP, Spring이 가장 바깥에 위치해야 하는 이유는?
- 프레임워크가 교체 가능해야 한다는 주장의 현실적 타당성은?
- `main` 함수가 이 레이어에 위치해야 하는 이유는?
- Frameworks & Drivers에서 안쪽으로 의존성이 향하는 구체적 방법은?
- 이 레이어에서 허용되는 것과 금지되는 것은?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Uncle Bob은 JPA, Spring, Kafka를 "세부 사항(Details)"이라고 부른다. 비즈니스 규칙(Entities, Use Cases)이 핵심이고, 프레임워크는 그것을 실행하기 위한 도구일 뿐이다.

이 관점에서 가장 바깥 레이어에 세부 사항을 모으는 것은 당연하다. 도구가 바뀌어도 핵심은 그대로여야 하기 때문이다. 실무에서 Spring을 완전히 교체하는 일은 드물지만, 이 원칙이 만드는 **테스트 가능성**과 **인프라 독립성**은 매우 현실적인 이익이다.

---

## 😱 흔한 실수 (Before — 프레임워크가 안쪽 레이어에 침투할 때)

```
프레임워크 침투 패턴:

Entities 레이어에 Spring/JPA 침투:
  @Entity                    ← JPA (Frameworks)
  @Table(name = "orders")    ← JPA (Frameworks)
  public class Order {
      @JsonProperty("id")    ← Jackson (Frameworks)
      private String id;
      
      @Autowired             ← Spring (Frameworks)
      private OrderPolicy policy; // Entity에 Spring DI?!
  }

Use Cases에 Kafka 직접 사용:
  public class PlaceOrderInteractor {
      @Autowired
      KafkaTemplate<String, String> kafkaTemplate; // Frameworks가 UseCase에 침투
      
      public void execute(...) {
          kafkaTemplate.send("orders", serialize(order)); // Kafka 직접 호출
      }
  }

결과:
  Entities/UseCase 테스트에 Spring Context 필요
  JPA 버전 업그레이드 시 Entities 수정 필요
  Kafka를 RabbitMQ로 교체 시 UseCase 수정 필요
  "프레임워크 바꾸기 불가능한 구조"
```

---

## ✨ 올바른 접근 (After — 세부 사항을 가장 바깥에 격리)

```
Frameworks & Drivers 레이어에 격리:

  frameworks/
    config/
      KafkaConfig.java          ← Kafka 연결 설정
      JpaConfig.java            ← JPA 설정
      SecurityConfig.java       ← Spring Security 설정
    persistence/
      entity/
        OrderJpaEntity.java     ← @Entity (JPA 전용)
      SpringDataOrderJpa.java   ← JpaRepository 상속 인터페이스
    messaging/
      KafkaMessagePublisher.java ← KafkaTemplate 직접 사용
    http/
      KakaoPayHttpClient.java   ← RestTemplate/WebClient 직접 사용
    main/
      Application.java          ← main() 함수, Spring Boot 시작점

바깥 레이어의 특징:
  JPA @Entity: frameworks/persistence/entity/에만 존재
  KafkaTemplate: frameworks/messaging/에만 존재
  안쪽 레이어(Entities, UseCase)는 이 클래스들을 모름
  = 의존성 규칙 준수
```

---

## 🔬 내부 원리 — Frameworks & Drivers의 구조와 역할

### 1. 왜 프레임워크가 가장 바깥에 있어야 하는가

```
"세부 사항(Details)"의 정의 (Uncle Bob):

비즈니스 규칙과 무관한 것들:
  DB 종류: MySQL, PostgreSQL, MongoDB — 비즈니스 규칙 자체는 아님
  메시징: Kafka, RabbitMQ — 이벤트 전달 방식, 비즈니스 아님
  HTTP 프레임워크: Spring MVC, Quarkus — HTTP 처리 방식, 비즈니스 아님
  외부 라이브러리: Jackson, MapStruct — 변환 도구, 비즈니스 아님

세부 사항의 특성:
  자주 바뀔 수 있음 (DB 버전, 프레임워크 버전)
  비즈니스 규칙을 바꾸지 않아도 기술적 이유로 변경 가능
  교체 가능해야 함 (완전 교체 아니어도 버전 업, 옵션 변경)

따라서:
  가장 자주 바뀔 수 있는 것 → 가장 바깥에
  가장 드물게 바뀌는 것 → 가장 안쪽에 (Entities)
  의존성은 안정적인 것을 향해야 함 (안쪽 방향)
```

### 2. main 함수 — 의존성 조립 지점

```java
// main 함수 = 의존성 그래프 전체를 조립하는 곳
// Clean Architecture에서 main은 Frameworks & Drivers 레이어

@SpringBootApplication
public class OrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}

// 왜 main이 가장 바깥인가?
// main은 모든 구체 클래스를 알아야 함:
//   JpaOrderGateway (Interface Adapters)
//   KakaoPayPaymentGateway (Interface Adapters)
//   PlaceOrderInteractor (Use Cases)
//   Order (Entities)
//   + Spring Boot 자동 설정, JPA 설정, Kafka 설정

// Spring DI Container = main의 책임을 자동화한 것
// Spring이 없었다면 main에서 직접 객체를 생성하고 주입:
//   OrderGateway gateway = new JpaOrderGateway(dataSource);
//   PaymentGateway payment = new KakaoPayPaymentGateway(apiKey);
//   PlaceOrderInteractor useCase = new PlaceOrderInteractor(gateway, payment);
//   OrderController controller = new OrderController(useCase, presenter);

// Spring @Configuration이 이 역할을 담당:
@Configuration
public class OrderConfiguration {

    @Bean
    public OrderGateway orderGateway(SpringDataOrderJpa jpa, OrderGatewayMapper mapper) {
        return new JpaOrderGateway(jpa, mapper); // 구체 클래스 조립
    }

    @Bean
    public PaymentGateway paymentGateway(KakaoPayHttpClient client) {
        return new KakaoPayPaymentGateway(client); // 구체 클래스 조립
    }
    // 내부 레이어(Interactor)는 인터페이스만 알고
    // 여기서 구체 클래스를 주입받음
}
```

### 3. JPA Entity — 가장 바깥에 있어야 하는 이유

```java
// OrderJpaEntity: Frameworks & Drivers 레이어 (또는 Interface Adapters)
// @Entity는 JPA 전용 — 비즈니스 도메인과 무관

@Entity
@Table(name = "orders")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class OrderJpaEntity {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_id", unique = true, nullable = false)
    private String orderId;

    @Column(name = "user_id", nullable = false)
    private String userId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private String status; // Enum String 저장

    @Column(name = "payment_tx_id")
    private String paymentTxId;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLineJpaEntity> lines = new ArrayList<>();
}

// 왜 OrderJpaEntity가 가장 바깥에 있어야 하는가:
// JPA @Entity, @Column, @OneToMany → JPA(Frameworks)에 의존
// JPA 버전 업그레이드, 스키마 변경 → OrderJpaEntity 수정
// 이 변경이 Order(Entities)에 영향주면 안 됨
// → 두 클래스를 분리해서 JPA 변경이 Domain에 파급되지 않게

// Spring Data JPA Repository
interface SpringDataOrderJpa extends JpaRepository<OrderJpaEntity, Long> {
    Optional<OrderJpaEntity> findByOrderId(String orderId);
    List<OrderJpaEntity> findByUserId(String userId);
}
// 이것도 Frameworks 레이어 — JpaRepository는 Spring Data JPA 프레임워크
```

### 4. 프레임워크 교체 가능성 — 현실적 평가

```
Uncle Bob의 주장:
  "좋은 아키텍처는 프레임워크를 세부 사항으로 취급해야 한다"
  "Spring 대신 Guice를 쓸 수 있어야 한다"

현실적 평가:

완전 교체가 드문 이유:
  Spring → Quarkus 교체: 팀 재학습 비용 + 생태계 차이
  MySQL → MongoDB: 데이터 모델 완전 재설계 필요
  HTTP REST → gRPC: API 계약 전면 변경
  → 이런 교체는 아키텍처보다 사업 결정

실제로 자주 일어나는 것들:
  ① 결제 API 교체: KakaoPay → TossPay (→ 가장 현실적 이익)
  ② 알림 서비스 교체: AWS SES → SendGrid
  ③ 메시지 브로커 변경: RabbitMQ → Kafka
  ④ Spring Boot 버전 업그레이드: 2.x → 3.x
  ⑤ JPA → JDBC: 성능 최적화를 위한 부분 교체

이런 교체에서 Clean Architecture의 이익:
  결제 API 교체: KakaoPayPaymentGateway → TossPayPaymentGateway (1개 파일)
  메시지 브로커 변경: KafkaPublisher → RabbitMQPublisher (1개 파일)
  Spring Boot 업그레이드: 내부 레이어 코드 변경 없음

결론:
  "프레임워크 독립적"의 현실적 의미:
  = "외부 시스템 교체 시 내부 비즈니스 코드 변경 없음"
  = "프레임워크 버전 업그레이드가 비즈니스 로직에 영향 없음"
  = "테스트에서 프레임워크 없이 비즈니스 로직 검증 가능"
```

### 5. Frameworks & Drivers에서 안쪽으로 향하는 방법

```
의존성 역전 원리 (DIP) 적용:

Frameworks & Drivers (구현체) → Interface Adapters → Use Cases → Entities

구체적으로:

  [Frameworks] JpaOrderGateway
       │ implements
       ↓
  [Use Cases] OrderGateway (인터페이스)
       ↑ depends on
  [Use Cases] PlaceOrderInteractor

  [Frameworks] KakaoPayPaymentGateway
       │ implements
       ↓
  [Use Cases] PaymentGateway (인터페이스)

  [Frameworks] KafkaOrderEventPublisher
       │ implements
       ↓
  [Use Cases] OrderEventGateway (인터페이스)

→ 모든 Frameworks 구현체가 Use Cases의 인터페이스를 향함
→ Frameworks가 Use Cases를 의존 (안쪽 방향)
→ Use Cases는 Frameworks를 모름
```

---

## 💻 실전 코드 — Frameworks & Drivers 레이어 구조

```java
// === Frameworks & Drivers 레이어 전체 구조 ===

// Spring Boot 설정 (Frameworks)
@SpringBootApplication
@EnableJpaRepositories(basePackages = "com.example.frameworks.persistence")
@EntityScan(basePackages = "com.example.frameworks.persistence.entity")
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}

// JPA 설정 (Frameworks)
@Configuration
@EnableTransactionManagement
public class JpaConfig {
    // DataSource, EntityManagerFactory 등 JPA 설정
}

// Kafka 설정 (Frameworks)
@Configuration
@EnableKafka
public class KafkaConfig {

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

// Kafka 이벤트 발행 구현 (Frameworks → Use Cases 인터페이스 구현)
@Component
@RequiredArgsConstructor
public class KafkaOrderEventPublisher implements OrderEventGateway {

    private final KafkaTemplate<String, String> kafkaTemplate; // Frameworks 의존 OK
    private final ObjectMapper objectMapper;

    @Override // Use Cases의 Gateway 인터페이스 구현
    public void publishOrderPlaced(String orderId) {
        try {
            String payload = objectMapper.writeValueAsString(
                Map.of("orderId", orderId, "timestamp", Instant.now().toString())
            );
            kafkaTemplate.send("order.placed", orderId, payload);
        } catch (JsonProcessingException e) {
            throw new EventPublishException("이벤트 발행 실패", e);
        }
    }
}

// HTTP 클라이언트 (Frameworks)
@Component
public class KakaoPayHttpClient {

    private final RestTemplate restTemplate; // Spring RestTemplate

    public KakaoPayChargeResponse charge(KakaoPayChargeRequest request) {
        return restTemplate.postForObject(
            kakaoPayApiUrl + "/charge",
            request,
            KakaoPayChargeResponse.class
        );
    }
}

// 의존성 조립 설정 (main의 Spring DI 역할)
@Configuration
public class UseCaseConfiguration {

    @Bean
    public PlaceOrderInputBoundary placeOrderInputBoundary(
        OrderGateway orderGateway,    // Interface Adapters의 Gateway 주입
        PaymentGateway paymentGateway,
        OrderEventGateway eventGateway
    ) {
        // Use Cases 레이어의 구체 클래스 생성 + 조립
        return new PlaceOrderInteractor(orderGateway, paymentGateway, eventGateway);
    }
}
```

---

## 📊 패턴 비교 — 프레임워크가 안쪽에 있을 때 vs 바깥에 있을 때

```
JPA가 Entities에 있을 때:
  Order.java: @Entity 있음
  JPA 버전 변경 → Order.java 수정 가능
  Order 단위 테스트: JPA 컨텍스트 필요 (또는 @Entity 어노테이션 처리)
  결과: Entities 레이어가 JPA에 묶임

JPA가 Frameworks에 있을 때:
  Order.java: @Entity 없음, 순수 Java
  JPA 버전 변경 → OrderJpaEntity.java만 수정
  Order 단위 테스트: 순수 Java, JPA 필요 없음
  결과: Entities 레이어가 JPA로부터 완전 독립

변경 비용 비교:
  시나리오: Spring Boot 2.x → 3.x 마이그레이드
  
  프레임워크 Entities 침투 시:
    Order.java (javax.persistence → jakarta.persistence 변경) 수정 필요
    + 모든 @Entity 클래스 수정
    + OrderService에 @Autowired 방식 변경 필요
    
  프레임워크 격리 시:
    OrderJpaEntity.java만 수정 (frameworks/ 패키지)
    Order.java, PlaceOrderInteractor.java: 변경 없음
```

---

## ⚖️ 트레이드오프

```
완전한 Frameworks 격리 비용:
  JPA Entity 분리: 매핑 코드 추가 (MapStruct로 절감 가능)
  설정 파일 증가: @Configuration 클래스들
  의존성 조립 복잡도: Spring DI 설정이 명시적

현실적 절충:
  Entities에 @Entity 허용: 매핑 비용 절감, Entities 오염 허용
  Use Cases에 @Transactional: Spring 의존 허용, 복잡도 절감
  핵심 격리: Kafka, 외부 API → 항상 가장 바깥에 (자주 교체 가능성)

완전 격리가 가장 이익인 경우:
  복잡한 도메인 + 빠른 단위 테스트 필요
  외부 API 교체 가능성 높음
  팀이 매핑 코드 관리 여력 있음

절충이 합리적인 경우:
  팀 규모 작고 빠른 개발 필요
  단순 CRUD가 많음
  최소한 외부 API(결제, 알림)만 Gateway로 추상화
```

---

## 📌 핵심 정리

```
Frameworks & Drivers 레이어 핵심:

위치: 가장 바깥쪽 원 (모든 세부사항 격리)
책임: 프레임워크 설정, DB 설정, 외부 연결, main 함수

포함:
  JPA 설정 + @Entity 클래스
  Spring Boot 자동 설정
  Kafka/RabbitMQ 설정
  외부 HTTP 클라이언트
  main 함수 (의존성 조립)

의존성 방향:
  Frameworks → Interface Adapters → Use Cases → Entities
  (모두 안쪽을 향함)

"세부 사항"의 의미:
  비즈니스 규칙이 아닌 기술적 구현 방식
  자주 바뀔 수 있는 것
  바뀌어도 비즈니스 로직에 영향 없어야 하는 것

현실적 이익:
  외부 API 교체: 1개 파일만 변경
  프레임워크 버전 업그레이드: 내부 코드 무영향
  테스트: 프레임워크 없이 비즈니스 로직 검증
```

---

## 🤔 생각해볼 문제

**Q1.** Spring의 `@Transactional`이 Use Cases 레이어에 있으면 Use Cases가 Spring에 의존하는 것 아닌가? 이것을 Frameworks & Drivers로 내리는 방법은?

<details>
<summary>해설 보기</summary>

맞습니다. `@Transactional`을 Use Cases에 붙이면 Spring 의존이 생깁니다. 이를 순수하게 하려면 Decorator 패턴을 사용합니다.

```java
// Use Cases: 순수 Java (Spring 없음)
public class PlaceOrderInteractor implements PlaceOrderInputBoundary {
    public void execute(...) { /* Spring 없음 */ }
}

// Frameworks: Transactional Decorator
@Configuration
public class TransactionConfig {
    @Bean
    public PlaceOrderInputBoundary placeOrderBoundary(PlaceOrderInteractor interactor) {
        // Spring이 @Transactional 프록시 생성
        return new TransactionalPlaceOrderInteractor(interactor);
    }
}

@Transactional
class TransactionalPlaceOrderInteractor implements PlaceOrderInputBoundary {
    private final PlaceOrderInteractor delegate;
    public void execute(PlaceOrderRequestModel req, PlaceOrderOutputBoundary out) {
        delegate.execute(req, out); // 위임
    }
}
```

실용적 결론: 대부분의 팀은 `@Transactional` Spring 의존을 허용합니다. Decorator 비용 대비 이익이 크지 않아서입니다.

</details>

---

**Q2.** `OrderJpaEntity`와 `Order`(Domain Entity)를 분리하면 DB 스키마와 도메인 모델을 별개로 진화시킬 수 있다. 이것이 실제로 어떤 상황에서 유용한가?

<details>
<summary>해설 보기</summary>

**DB 스키마와 도메인 모델의 독립적 진화가 필요한 상황:**

1. **DB 정규화 vs 도메인 모델**: 도메인에서 `Order`에 `lines` 리스트가 있지만, DB에서는 `orders`와 `order_lines` 두 테이블로 정규화. 매핑으로 연결.

2. **레거시 DB 스키마**: 기존 `ORDER_INFO` 컬럼명을 쓰는 DB가 있지만 도메인에서는 `orderId`로 표현하고 싶을 때. `@Column(name = "ORDER_INFO")`는 JPA Entity에만.

3. **DB 최적화**: 도메인에서 `Money` Value Object를 쓰지만 DB에서는 `DECIMAL(10,2)` 컬럼 하나. Mapper가 변환.

4. **마이크로서비스 이관**: 단일 테이블에서 데이터를 가져오지만 도메인에서 여러 Aggregate로 분리할 때. Mapper가 N:1 또는 1:N 변환.

도메인 모델과 DB 스키마가 항상 1:1이 아니라는 점에서 분리가 유연성을 제공합니다.

</details>

---

**Q3.** `@SpringBootApplication`의 컴포넌트 스캔이 모든 패키지를 스캔하면 Frameworks 레이어와 Entities 레이어의 경계가 코드에서 어떻게 강제되는가?

<details>
<summary>해설 보기</summary>

컴포넌트 스캔은 Bean 등록 메커니즘이고, 패키지 경계는 import 의존성으로 강제됩니다. 두 가지는 독립적입니다.

**ArchUnit으로 import 의존성 강제:**
```java
@Test
void entities_should_not_depend_on_frameworks() {
    noClasses()
        .that().resideInAPackage("..entity..")          // Entities 레이어
        .should().dependOnClassesThat()
        .resideInAPackage("org.springframework..")      // Frameworks
        .check(importedClasses);
}

@Test
void entities_should_not_depend_on_jpa() {
    noClasses()
        .that().resideInAPackage("..entity..")
        .should().dependOnClassesThat()
        .resideInAPackage("jakarta.persistence..")      // JPA (Frameworks)
        .check(importedClasses);
}
```

Spring이 런타임에 모든 Bean을 알아도, 코드 레벨에서 Entities가 Spring을 import하지 않으면 의존성 규칙을 지킨 것입니다. ArchUnit이 컴파일 타임 의존성을 자동으로 검증합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Interface Adapters 레이어](./04-interface-adapters-layer.md)** | **[홈으로 🏠](../README.md)** | **[다음: Clean Architecture 실전 ➡️](./06-clean-architecture-in-practice.md)**

</div>
