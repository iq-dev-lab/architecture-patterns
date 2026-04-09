# Entities 레이어 — 비즈니스 규칙의 핵심

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Entities가 어떤 외부 변경에도 영향받지 않아야 하는 이유는?
- 기업 수준 비즈니스 규칙과 애플리케이션 수준 비즈니스 규칙의 차이는?
- DDD의 Entity / Value Object / Aggregate와 Clean Architecture Entities의 매핑 관계는?
- Entities 레이어에서 Spring 어노테이션(`@Entity`, `@Component`)이 없어야 하는 이유는?
- Entities는 가장 자주 변경되어야 하는가, 가장 드물게 변경되어야 하는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

Clean Architecture에서 Entities는 가장 안쪽 원이다. 이 레이어가 흔들리면 모든 것이 흔들린다. 반대로 이 레이어가 안정적이면 프레임워크가 바뀌어도, DB가 바뀌어도, 비즈니스 핵심은 그대로다.

Entities를 "JPA Entity 클래스"와 혼동하는 실수가 가장 많다. Clean Architecture의 Entity는 `@Entity` 어노테이션이 붙은 JPA 클래스가 아니라, **비즈니스 규칙을 캡슐화한 순수 Java 클래스**다.

---

## 😱 흔한 실수 (Before — JPA Entity와 Clean Entity를 혼동할 때)

```
혼동 상황:
  "Entities 레이어? 그거 JPA @Entity 붙이는 곳 아닌가요?"

그 결과 (entity/ 패키지):
  @Entity
  @Table(name = "orders")
  public class Order {
      @Id @GeneratedValue
      private Long id;

      @Column(name = "total_price")
      private BigDecimal totalPrice;

      // getter/setter만 있음 (Anemic Domain Model)
      // 비즈니스 로직 없음 → Use Cases 레이어에 다 있음
  }

문제:
  ① Order.java가 JPA(@Entity, @Column)에 의존
     → JPA 버전 업그레이드 시 Order.java 영향받음
     → Entities 레이어가 Frameworks 레이어에 의존 = 의존성 규칙 위반

  ② Order에 비즈니스 로직이 없음 (Anemic)
     → "최소 금액 검증" 같은 기업 핵심 규칙이 Use Cases에 있음
     → Entities 레이어의 존재 이유가 없어짐

  ③ Entities 레이어가 자주 변경됨
     → DB 스키마 변경 → Entity 수정 → Entities 레이어가 불안정
     → 가장 안쪽인 Entities가 가장 불안정한 역설
```

---

## ✨ 올바른 접근 (After — Entities는 순수한 비즈니스 규칙 캡슐)

```
올바른 Entities 레이어:

  entities/
    Order.java           ← 순수 Java, 비즈니스 로직 포함
    OrderLine.java       ← 값 객체 또는 내부 Entity
    Money.java           ← Value Object
    OrderId.java         ← Value Object (Type-safe ID)
    OrderStatus.java     ← Enum (상태 전이 규칙)
    OrderDomainService.java ← 두 Aggregate 간 조율이 필요한 로직

  특징:
    import에 Spring, JPA, Jackson 없음 (java.util.* 만 있음)
    모든 비즈니스 규칙이 여기에 캡슐화
    DB 스키마 변경 → 이 레이어 영향 없음
    Spring 버전 업 → 이 레이어 영향 없음
    가장 안정적 → 가장 드물게 변경됨

  Entities의 변경 이유 (오직 하나):
    "기업의 핵심 비즈니스 정책이 변경될 때"
    예: "최소 주문 금액 정책이 1,000원 → 5,000원으로 변경"
```

---

## 🔬 내부 원리 — Entities 레이어의 설계 원칙

### 1. 기업 수준 vs 애플리케이션 수준 비즈니스 규칙

```
기업 수준 비즈니스 규칙 (Entities에 위치):
  "어떤 소프트웨어 시스템을 써도 적용되는 규칙"
  "비즈니스 자체의 규칙, 기술과 무관"

  예시:
    주문 금액 = Σ(단가 × 수량)
    취소는 배송 시작 전에만 가능
    할인은 정가에 대해서만 적용 (이미 할인된 가격에 추가 할인 불가)
    포인트는 실제 결제금액의 1%만 적립 (취소분 제외)

  이 규칙들은:
    웹 쇼핑몰에서도 동일
    콜센터 주문 시스템에서도 동일
    배치 주문 시스템에서도 동일
    = 기업의 핵심 정책, 기술과 무관

애플리케이션 수준 비즈니스 규칙 (Use Cases에 위치):
  "이 특정 애플리케이션에서의 흐름"
  "다른 애플리케이션에서는 다를 수 있음"

  예시:
    "웹 쇼핑몰 주문 생성 흐름: 재고 확인 → 쿠폰 적용 → 결제 → 저장 → 알림"
    "콜센터 주문: 재고 확인 → 상담원 승인 → 수동 결제 처리"

  이 흐름들은:
    시스템마다 다름
    비즈니스 확장 시 달라질 수 있음
    = 이 애플리케이션 고유의 규칙
```

### 2. Entities 레이어와 DDD 개념의 매핑

```
DDD 개념 → Clean Architecture Entities 매핑:

DDD Domain Entity → Clean Architecture Entity
  Order (Aggregate Root):
    - 상태를 가지며 시간이 지남에 따라 변함
    - 비즈니스 ID(OrderId)로 동일성 판단
    - 비즈니스 규칙 캡슐화
    Clean에서: Entities 레이어의 핵심

DDD Value Object → Clean Architecture Entities 내부
  Money, Address, OrderId:
    - 상태는 있지만 동일성은 값으로 판단
    - 불변(Immutable)
    Clean에서: Entities 레이어 내 Value Object로 구현

DDD Aggregate → Clean Architecture Entities 내부 구조
  Order Aggregate:
    - Order (Root) + OrderLine (내부 Entity)
    - 외부는 Order를 통해서만 내부 접근
    Clean에서: Entities 레이어 내 클래스 그룹

DDD Domain Service → Clean Architecture Entities 또는 Use Cases
  두 Aggregate 간 비즈니스 규칙:
    Clean에서: 규칙이 기업 수준 → Entities 레이어
               규칙이 흐름 조율 → Use Cases 레이어

DDD Repository (인터페이스) → Clean Architecture Use Cases (Output Port)
  Clean에서: Use Cases 레이어의 Output Boundary(Gateway 인터페이스)
```

### 3. Entities에 Spring 어노테이션이 없어야 하는 이유

```
Spring 어노테이션이 Entities에 있으면:

@Entity 문제:
  Order.java에 @Entity가 있으면 JPA가 없는 환경에서 Order 사용 불가
  클래스패스에 javax.persistence/jakarta.persistence 필요
  JPA 버전 업그레이드 시 Order.java 수정 가능성

@Component / @Service 문제:
  Order.java에 @Component면 Spring 컨테이너에 의존
  단위 테스트에 Spring Context 필요 → 빠른 테스트 불가

@Transactional 문제:
  Entities에 @Transactional은 비즈니스 로직이 트랜잭션 경계를 결정한다는 의미
  트랜잭션은 Use Cases 레이어의 결정이어야 함

@JsonProperty 문제:
  Entities에 JSON 어노테이션은 HTTP 표현이 도메인을 오염
  HTTP 스펙 변경 시 Entities 수정 필요 → 의존성 규칙 위반

대안:
  @Entity → JPA Entity 분리 (adapter/out/persistence/entity/)
  @Transactional → Use Cases 레이어에
  @JsonProperty → HTTP DTO 클래스에
  @Component → Use Cases가 인스턴스 생성 또는 DI를 Frameworks 레이어에서

Entities가 import할 수 있는 것:
  java.util.*, java.time.*, java.math.* (Java 표준)
  같은 Entities 레이어 내 다른 클래스
  없음 (가장 이상적)
```

### 4. Entities는 왜 가장 드물게 변경되어야 하는가

```
Uncle Bob의 역설:
  가장 안쪽 레이어 = 가장 많은 것에 의존받음 (Fan-in 높음)
  = 변경 시 가장 많은 곳에 영향
  = 따라서 가장 안정적이어야 함

안정성 원칙과의 연결:
  Fan-in이 높은 클래스 → 의존받는 것이 많음 → 안정적이어야 함
  
  Entities의 Fan-in:
    Use Cases가 Entities를 의존
    Interface Adapters가 Entities를 의존
    → Entities가 변경 시 모든 레이어에 파급

  따라서:
    Entities 변경 이유: "핵심 비즈니스 정책 변경" 하나만
    DB 스키마 변경 → Entities 변경 없음 (JPA Entity 변경)
    API 스펙 변경 → Entities 변경 없음 (DTO 변경)
    프레임워크 변경 → Entities 변경 없음 (Frameworks 레이어 변경)

변경 빈도 비교 (높음 → 낮음):
  Frameworks: DB 버전, 프레임워크 버전, 외부 API → 자주 변경
  Interface Adapters: API 스펙 변경 → 중간
  Use Cases: 새 기능, 흐름 변경 → 중간
  Entities: 핵심 비즈니스 정책 → 드물게 변경 (가장 안정적)
```

### 5. 불변성과 캡슐화 — 올바른 Entities 설계

```java
// 올바른 Entity 설계 원칙

public class Order {

    // ① 불변 필드 (생성 후 변경 불가)
    private final OrderId id;
    private final UserId userId;
    private final List<OrderLine> lines; // 방어적 복사

    // ② 가변 필드 (비즈니스 이벤트로만 변경)
    private OrderStatus status;
    private String paymentTransactionId;

    // ③ 기본 생성자 없음 → 불완전한 객체 생성 불가
    private Order(OrderId id, UserId userId, List<OrderLine> lines, OrderStatus status) {
        this.id = id;
        this.userId = userId;
        this.lines = List.copyOf(lines); // 불변 복사
        this.status = status;
    }

    // ④ 정적 팩토리 메서드 → 유효성 검증 후 생성
    public static Order create(UserId userId, List<OrderLine> lines) {
        Objects.requireNonNull(userId, "userId 필수");
        if (lines == null || lines.isEmpty())
            throw new IllegalArgumentException("주문 항목 없음");
        return new Order(OrderId.newId(), userId, lines, OrderStatus.DRAFT);
    }

    // ⑤ DB 복원용 팩토리 (Use Cases → Gateway → 복원)
    public static Order reconstitute(OrderId id, UserId userId,
                                      List<OrderLine> lines, OrderStatus status) {
        return new Order(id, userId, lines, status);
    }

    // ⑥ 비즈니스 메서드 → 기업 수준 규칙 캡슐화
    public void place() {
        if (this.status != OrderStatus.DRAFT)
            throw new IllegalStateException("DRAFT 상태만 주문 가능");
        if (calculateTotal().isLessThan(Money.of(1_000)))
            throw new MinOrderAmountException(Money.of(1_000));
        this.status = OrderStatus.PLACED;
    }

    public void cancel() {
        if (this.status == OrderStatus.SHIPPED || this.status == OrderStatus.DELIVERED)
            throw new CannotCancelException("배송 시작 후 취소 불가");
        this.status = OrderStatus.CANCELLED;
    }

    public Money calculateTotal() {
        return lines.stream()
            .map(OrderLine::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // ⑦ getter만 (setter 없음) → 외부에서 직접 상태 변경 불가
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public List<OrderLine> getLines() { return lines; } // 이미 불변 복사
}
```

---

## 📊 패턴 비교 — Entities 레이어 설계 방식 비교

```
설계 방식별 Entities 특성:

                    Anemic Model       Rich Domain Model
                    (잘못된 방식)       (올바른 방식)
──────────────────┼─────────────────┼───────────────────────
비즈니스 로직      │ Use Cases에 있음  │ Entities에 있음
불변성             │ setter 존재       │ 불변 필드 + 비즈니스 메서드
Spring 어노테이션  │ @Entity 있음      │ 없음 (순수 Java)
기본 생성자        │ 있음 (JPA 요구)   │ 없음 (불완전 생성 방지)
단위 테스트        │ 로직 없어서 불필요 │ 순수 Java로 빠르게 테스트
변경 빈도          │ DB 스키마마다 변경 │ 비즈니스 정책 변경 시만
```

---

## ⚖️ 트레이드오프

```
순수 Entities 설계의 비용:
  JPA Entity 분리 필요 → 매핑 코드 추가
  기본 생성자 없음 → JPA와 직접 사용 시 설정 복잡
  팀이 Anemic Model에 익숙하면 Rich Domain Model 학습 필요

순수 Entities 설계의 이익:
  비즈니스 규칙이 한 곳에만 존재 → 중복 없음
  순수 Java 단위 테스트 → 0.001초 실행
  기술 변경(JPA, Spring 버전)이 Entities에 영향 없음

현실적 절충:
  @Entity 완전 제거가 어려우면:
    @Entity는 허용, @JsonProperty 같은 HTTP 어노테이션은 금지
    JPA 제약(기본 생성자, LAZY)이 비즈니스 메서드를 오염하지 않도록 주의
  핵심: 어노테이션이 있어도 비즈니스 로직은 반드시 Entity에 있어야 함
```

---

## 📌 핵심 정리

```
Clean Architecture Entities 핵심:

위치: 가장 안쪽 원 (모든 의존성이 안으로 향함)
책임: 기업 수준 비즈니스 규칙 캡슐화
변경 이유: 핵심 비즈니스 정책 변경 시만 (가장 드물게)

올바른 Entities:
  순수 Java (Spring, JPA, Jackson 없음)
  비즈니스 메서드 있음 (Rich Domain Model)
  불변 필드 + 정적 팩토리 메서드
  기본 생성자 없음

Entities ≠ JPA @Entity:
  JPA @Entity = Infrastructure 관심사 (Frameworks 레이어)
  Clean Entity = 비즈니스 규칙 캡슐 (Entities 레이어)
  → JPA Entity는 Interface Adapters 또는 Frameworks에 분리

DDD와의 연결:
  Domain Entity/VO/Aggregate = Clean Architecture Entities
  Repository 인터페이스 = Use Cases Output Boundary (Gateway 인터페이스)
```

---

## 🤔 생각해볼 문제

**Q1.** `@Entity`를 Entities 레이어에서 완전히 제거해야 하는 경우와 절충으로 허용해도 되는 경우의 판단 기준은?

<details>
<summary>해설 보기</summary>

**판단 기준: JPA 제약이 비즈니스 메서드를 오염시키는가?**

제거 필요: JPA 제약이 도메인 설계를 침해할 때
```java
// LAZY 로딩이 비즈니스 메서드를 깨뜨림
@Entity
public class Order {
    @OneToMany(fetch = LAZY)
    private List<OrderLine> lines;

    public Money calculateTotal() {
        return lines.stream()... // @Transactional 밖이면 LazyInitializationException!
    }
}
```

허용 가능: JPA 어노테이션이 비즈니스 로직에 영향 없을 때
```java
@Entity // 어노테이션은 있지만
public class Order {
    @Id @Column(name = "order_id")
    private String id; // 필드만 영향받음, 비즈니스 메서드는 독립적

    public void place() { /* JPA와 무관한 순수 비즈니스 로직 */ }
}
```

실용적 기준: 팀이 매핑 비용을 감당할 수 있고 JPA 제약 문제가 없다면 절충 허용. 문제가 발생하기 시작하면 분리.

</details>

---

**Q2.** `OrderId`를 `String`이나 `Long`이 아닌 별도 Value Object 타입으로 만드는 이유는?

<details>
<summary>해설 보기</summary>

**타입 안전성(Type Safety)과 비즈니스 의미 표현입니다.**

```java
// Long을 ID로 쓸 때의 문제
void transfer(Long fromAccountId, Long toAccountId, Long amount) {
    // 컴파일 에러 없이 파라미터 순서를 바꿔 전달 가능
    // transfer(toId, fromId, amount) → 버그지만 컴파일 통과
}

// Value Object 사용
void transfer(AccountId from, AccountId to, Money amount) {
    // transfer(to, from, amount) → 타입이 같아서 여전히 통과되지만
    // transfer(amount, from, to) → 컴파일 에러! (Money는 AccountId 아님)
}
```

추가 이익:
- 유효성 검증: `OrderId.of(null)` → 생성자에서 즉시 예외
- 의미 표현: `findById(OrderId id)` vs `findById(String id)` — 어떤 ID인지 명확
- 리팩터링 안전성: ID 타입(String → UUID 형식 강제) 변경 시 컴파일 에러로 누락 발견

단점: 파일 수 증가. 모든 ID에 Value Object를 만드는 것보다 핵심 엔티티에만 적용하는 것이 실용적.

</details>

---

**Q3.** Entities 레이어의 Order 클래스와 Interface Adapters 레이어의 OrderJpaEntity 클래스가 분리됐을 때, DB 복원(reconstitute) 시 `Order`의 `private` 생성자를 어떻게 호출하는가?

<details>
<summary>해설 보기</summary>

**별도의 복원용 정적 팩토리 메서드(reconstitute)를 제공합니다.**

```java
// Entities 레이어
public class Order {
    private Order(OrderId id, UserId userId, ...) { ... } // private

    // 신규 생성용 (검증 포함)
    public static Order create(UserId userId, List<OrderLine> lines) {
        validateLines(lines);
        return new Order(OrderId.newId(), userId, lines, OrderStatus.DRAFT);
    }

    // DB 복원용 (검증 생략 — DB에 있으면 이미 유효함)
    public static Order reconstitute(OrderId id, UserId userId,
                                     List<OrderLine> lines, OrderStatus status,
                                     String paymentTxId) {
        return new Order(id, userId, lines, status); // 복원
    }
}

// Interface Adapters 레이어 (Gateway)
public class JpaOrderGateway implements OrderGateway {
    public Optional<Order> findById(OrderId id) {
        return jpa.findByOrderId(id.value())
            .map(entity -> Order.reconstitute( // reconstitute 사용
                OrderId.of(entity.getOrderId()),
                UserId.of(entity.getUserId()),
                entity.getLines().stream().map(this::toLine).toList(),
                entity.getStatus(),
                entity.getPaymentTxId()
            ));
    }
}
```

`create`는 신규 생성 시 비즈니스 검증을 수행하고, `reconstitute`는 이미 유효한 DB 데이터를 복원할 때 사용합니다. 이 두 팩토리 메서드의 분리가 도메인 의도를 명확히 표현합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Clean Architecture 개요](./01-clean-architecture-overview.md)** | **[홈으로 🏠](../README.md)** | **[다음: Use Cases 레이어 ➡️](./03-use-cases-layer.md)**

</div>
