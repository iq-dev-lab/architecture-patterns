# SOLID 원칙과 아키텍처 — SRP/OCP/DIP가 패턴의 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- SRP(단일 책임)가 레이어 분리의 이유인 것은 무슨 의미인가?
- OCP(개방-폐쇄)가 Adapter 패턴의 이유는 무엇인가?
- DIP(의존성 역전)가 Hexagonal과 Clean Architecture의 핵심인 이유는 무엇인가?
- SOLID 5원칙이 각 아키텍처 패턴에서 구체적으로 어디에 나타나는가?
- ISP(인터페이스 분리)가 Port 설계와 어떻게 연결되는가?

---

## 🔍 왜 이 아키텍처 결정이 중요한가

SOLID 원칙을 클래스 레벨의 OOP 원칙으로만 알고 있다면, 아키텍처 패턴이 왜 그렇게 설계됐는지 이해하기 어렵다.

Hexagonal Architecture의 Port가 왜 인터페이스인가? → ISP + DIP  
레이어를 왜 나누는가? → SRP  
새 결제 수단을 추가할 때 기존 코드를 수정하지 않아도 되는 이유? → OCP  
도메인이 JPA를 몰라야 하는 이유? → DIP

SOLID를 아키텍처 수준에서 이해하면, 어떤 아키텍처 패턴을 봐도 "왜 이렇게 설계됐는가"를 스스로 유도할 수 있다.

---

## 😱 흔한 실수 (Before — SOLID를 클래스 레벨로만 알 때)

```
상황: 코드 리뷰에서 "이건 SRP 위반이에요"라는 피드백을 받음

SOLID를 클래스 레벨로만 알 때의 이해:
  SRP: "클래스는 하나의 책임만 가져야 한다"
       → OrderService를 OrderCreationService, OrderQueryService로 쪼개자
       → 수많은 작은 클래스들이 생겨남
       → 오히려 파악하기 어려워짐

  OCP: "확장에는 열려있고 수정에는 닫혀있어야 한다"
       → 추상 클래스를 만들고 상속으로 확장하면 되는 것 아닌가?

  DIP: "추상에 의존하라"
       → 모든 클래스에 인터페이스 만들면 되는 것 아닌가?

결과:
  원칙을 알지만 어디에 적용해야 하는지 모름
  클래스 수만 늘어나고 아키텍처는 개선되지 않음
  "SOLID 했는데 왜 테스트가 어렵지?"

아키텍처 수준에서 SOLID를 이해하지 못할 때:
  레이어드 아키텍처를 쓰지만 레이어가 왜 있는지 모름
  Hexagonal Architecture를 보고 "Port가 왜 필요하지?" 의문
  Clean Architecture의 4개 동심원 분리 이유를 설명할 수 없음
```

---

## ✨ 올바른 접근 (After — SOLID를 아키텍처 수준에서 이해할 때)

```
SOLID 5원칙을 아키텍처 패턴과 연결하면:

SRP (단일 책임 원칙)
  → 레이어 분리의 이유
  → 도메인 레이어: 비즈니스 규칙 변경 이유
     인프라 레이어: 기술 스택 변경 이유
     = 변경 이유가 다른 것들을 분리 → 레이어

OCP (개방-폐쇄 원칙)
  → Adapter 패턴의 이유
  → "새 결제 수단 추가 = 새 Adapter 클래스 추가"
  → 기존 Port(인터페이스)와 PlaceOrderService는 수정 없음
  = 확장(Adapter 추가)에는 열림, 기존 코드 수정에는 닫힘

LSP (리스코프 치환 원칙)
  → Adapter 교체 가능성의 이유
  → JpaOrderRepository를 MongoOrderRepository로 교체해도
     OrderRepository 인터페이스 계약을 만족하면 동작해야 함

ISP (인터페이스 분리 원칙)
  → Port 설계의 이유
  → OrderRepository: save(), findById() — 주문에 필요한 것만
  → "모든 것을 아는 거대한 Repository 인터페이스" 금지

DIP (의존성 역전 원칙)
  → Hexagonal/Clean Architecture의 핵심 원리
  → 도메인(고수준)이 인프라(저수준)를 모르게 함
  → 인프라가 도메인 인터페이스를 구현하는 방향으로 역전
```

---

## 🔬 내부 원리 — 각 원칙이 아키텍처에 나타나는 방식

### 1. SRP — 레이어 분리의 이유

```
Uncle Bob의 SRP 재정의 (Clean Architecture):
  "하나의 모듈은 오직 하나의 액터(Actor)에 대해서만 책임져야 한다"
  액터 = 변경을 요청하는 사람/조직

클래스 수준의 SRP:
  "OrderService가 너무 많은 일을 한다 → 나눠라"
  → 단순히 클래스를 쪼개는 것

아키텍처 수준의 SRP:
  "이 모듈이 변경되는 이유는 무엇인가?"
  → 비즈니스 팀이 비즈니스 규칙을 바꾸면 → 도메인 레이어 변경
  → DBA가 DB 스키마를 바꾸면 → 인프라 레이어 변경
  → 프론트엔드 팀이 API 스펙을 바꾸면 → Controller 레이어 변경

  변경 이유가 다른 것들 = 다른 레이어로 분리해야 함

레이어드 아키텍처가 SRP를 실현하는 방식:
  Presentation Layer (Web, Controller)
    변경 이유: API 스펙 변경, UI 요구사항 변경

  Business Layer (UseCase, Domain Service)
    변경 이유: 비즈니스 규칙 변경

  Data Access Layer (Repository, Adapter)
    변경 이유: DB 종류 변경, 외부 API 변경

SRP 위반의 신호:
  "주문 서비스 수정했는데 왜 결제 관련 테스트가 깨지지?"
  → 두 가지 다른 책임이 같은 클래스에 있음
```

### 2. OCP — Adapter 패턴의 이유

```
OCP 정의:
  "소프트웨어 개체(클래스, 모듈, 함수)는
   확장에는 열려있어야 하고, 수정에는 닫혀있어야 한다"

어떻게 동시에 열림과 닫힘이 가능한가?
  답: 인터페이스(Port)로 확장 지점을 정의

Adapter 패턴이 OCP를 실현하는 방식:

  PaymentPort (인터페이스) ← 이것은 수정에 닫혀있음
                                (한번 정의되면 계약 유지)
       ↑ 구현
  KakaoPayAdapter          ← 기존 코드 (수정하지 않음)
  TossPayAdapter           ← 신규 추가 (확장에 열려있음)
  NaverPayAdapter          ← 신규 추가 (확장에 열려있음)

PlaceOrderService:
  PaymentPort에만 의존 → 새 결제 수단 추가 시 수정 없음

OCP가 아닐 때 (수정에도 열려있는 상황):

  OrderService.placeOrder():
    if (paymentType == KAKAO) {
        kakaoPayClient.charge(...);
    } else if (paymentType == TOSS) {     // ← 수정
        tossPayClient.charge(...);
    } else if (paymentType == NAVER) {    // ← 또 수정
        naverPayClient.charge(...);
    }

  새 결제 수단 추가마다 OrderService를 수정해야 함
  → 기존에 잘 돌던 카카오페이 로직이 건드려짐
  → OCP 위반

OCP를 지킬 때:
  새 결제 수단 = NaverPayAdapter 클래스 파일만 추가
  OrderService 코드는 단 한 줄도 수정하지 않음
```

### 3. LSP — Adapter 교체 가능성의 이유

```
LSP 정의:
  "하위 타입은 언제나 상위 타입으로 대체될 수 있어야 한다"

아키텍처에서의 LSP:
  OrderRepository 인터페이스를 구현하는 모든 클래스는
  OrderRepository를 사용하는 코드를 깨지 않고 교체 가능해야 함

올바른 LSP 준수:
  interface OrderRepository {
      void save(Order order);  // 저장 후 예외 없으면 성공 보장
      Optional<Order> findById(OrderId id);  // 없으면 empty 반환
  }

  JpaOrderRepository implements OrderRepository {
      // save: JPA로 저장, 성공 보장
      // findById: JPA로 조회, 없으면 Optional.empty()
  }

  MongoOrderRepository implements OrderRepository {
      // save: MongoDB로 저장, 성공 보장
      // findById: MongoDB로 조회, 없으면 Optional.empty()
  }

  PlaceOrderService는 두 구현체 중 어느 것을 주입받아도 동일하게 동작

LSP 위반의 신호:
  interface OrderRepository {
      Optional<Order> findById(OrderId id);
  }

  // 위반: null 반환 (계약: Optional, 실제: null)
  class InMemoryOrderRepository implements OrderRepository {
      public Optional<Order> findById(OrderId id) {
          Order order = store.get(id);
          return order;  // ← null 가능! LSP 위반
      }
  }

  // 위반: 예외 발생 방식이 다름
  class CachedOrderRepository implements OrderRepository {
      public void save(Order order) {
          throw new UnsupportedOperationException("읽기 전용");
          // ← 계약에 없는 예외 발생, LSP 위반
      }
  }
```

### 4. ISP — Port 설계의 이유

```
ISP 정의:
  "클라이언트가 사용하지 않는 메서드에 의존하면 안 된다"
  → 인터페이스는 사용하는 쪽에 맞게 분리해야 한다

Port 설계에서 ISP가 나타나는 방식:

ISP 위반 (거대한 단일 인터페이스):
  interface OrderFacade {
      void save(Order order);
      Optional<Order> findById(OrderId id);
      List<Order> findByUserId(UserId userId);
      List<Order> findByStatus(OrderStatus status);
      void delete(OrderId id);
      Page<Order> findAll(Pageable pageable);
      void bulkUpdate(List<Order> orders);
      List<Order> findByDateRange(LocalDate from, LocalDate to);
  }

  문제:
    PlaceOrderService는 save()와 findById()만 필요
    그런데 위 인터페이스 전체에 의존 = 사용하지 않는 메서드에 의존
    OrderFacade 변경 시 PlaceOrderService도 영향

ISP 준수 (사용하는 쪽에 맞게 분리):
  // PlaceOrderService가 사용하는 Port
  interface OrderSavePort {
      void save(Order order);
  }

  // FindOrderService가 사용하는 Port
  interface OrderQueryPort {
      Optional<Order> findById(OrderId id);
      List<Order> findByUserId(UserId userId);
  }

  // AdminService가 사용하는 Port
  interface OrderAdminPort {
      Page<Order> findAll(Pageable pageable);
      void bulkUpdate(List<Order> orders);
  }

  JpaOrderRepository implements OrderSavePort, OrderQueryPort, OrderAdminPort {
      // 하나의 구현 클래스가 여러 Port를 구현
  }

  이익:
    PlaceOrderService는 OrderSavePort만 의존
    → OrderQueryPort, OrderAdminPort 변경에 영향 없음
    → 각 UseCase가 필요한 Port만 알면 됨
```

### 5. DIP — Hexagonal/Clean Architecture의 핵심

```
DIP 정의:
  "고수준 모듈이 저수준 모듈에 의존해서는 안 된다
   둘 다 추상에 의존해야 한다
   추상은 세부 사항에 의존해서는 안 된다
   세부 사항이 추상에 의존해야 한다"

고수준 vs 저수준:
  고수준: 비즈니스 규칙, 정책, 핵심 도메인 로직
  저수준: 구체적인 구현 세부 사항 (JPA, Kafka, HTTP, SMTP)

DIP 없이 (전통적 레이어드):
  도메인(고수준) → JPA(저수준)
  OrderService → JpaOrderRepository

  문제:
    고수준이 저수준에 의존 = DIP 위반
    JPA가 바뀌면 도메인도 바꿔야 함
    도메인 테스트에 JPA 필요

DIP 적용 (Hexagonal/Clean):
  도메인(고수준) → OrderRepository 인터페이스(추상)
  JpaOrderRepository(저수준) → OrderRepository 인터페이스(추상) 구현

  의존 방향:
    OrderService → OrderRepository(I/F) ← JpaOrderRepository

  결과:
    고수준(도메인)이 저수준(JPA)을 모름
    저수준(JPA)이 고수준(도메인 인터페이스)을 구현 = 의존성 역전!
    JPA → MongoDB 교체: 도메인 코드 변경 없음

DIP가 "역전"인 이유:
  자연스러운 의존 방향: 도메인 → 인프라 (사용하는 쪽이 사용받는 쪽에 의존)
  DIP 적용 후:         인프라 → 도메인 (구현이 계약에 의존)
  방향이 뒤집힘 = 역전(Inversion)
```

---

## 💻 실전 코드 — SOLID 5원칙의 아키텍처 구현

```java
// SOLID 5원칙이 모두 적용된 Hexagonal Architecture 예시

// === SRP: 레이어 분리 ===
// 각 클래스는 하나의 변경 이유만 가짐

// Domain: 비즈니스 규칙 변경 이유
public class Order {
    private final OrderId id;
    private final List<OrderLine> lines;
    private OrderStatus status;

    public Money calculateTotal() { /* 비즈니스 규칙 */ }
    public void place() { /* 비즈니스 규칙 */ }
    public void cancel() { /* 비즈니스 규칙 */ }
}

// Infrastructure: 기술 스택 변경 이유
@Entity @Table(name = "orders")
public class OrderJpaEntity {
    @Id private Long id;
    @OneToMany private List<OrderLineJpaEntity> lines;
    @Enumerated private OrderStatus status;
}

// Presentation: API 스펙 변경 이유
public record PlaceOrderRequest(
    String userId,
    List<String> itemIds
) {}

// === OCP: Adapter로 확장, 기존 코드 수정 없음 ===

// Port (수정에 닫힘)
public interface PaymentPort {
    PaymentResult charge(PaymentRequest request);
}

// 기존 Adapter (수정하지 않음)
@Component
public class KakaoPayAdapter implements PaymentPort {
    @Override
    public PaymentResult charge(PaymentRequest request) { /* ... */ }
}

// 신규 확장 (새 파일만 추가)
@Component
public class TossPayAdapter implements PaymentPort {
    @Override
    public PaymentResult charge(PaymentRequest request) { /* ... */ }
}

// PlaceOrderService: 변경 없음 (PaymentPort에만 의존)

// === LSP: 모든 구현체가 계약을 준수 ===

public interface OrderRepository {
    void save(Order order);                        // 성공 or 예외
    Optional<Order> findById(OrderId id);          // empty or Order, null 없음
}

// LSP 준수: 계약을 정확히 지킴
@Repository
public class JpaOrderRepository implements OrderRepository {
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value())
            .map(OrderMapper::toDomain); // null 반환 없음
    }
}

// LSP 준수: InMemory도 동일 계약
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<OrderId, Order> store = new HashMap<>();

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id)); // null 반환 없음
    }
}

// === ISP: 사용하는 쪽에 맞게 Port 분리 ===

// PlaceOrderService가 필요한 것만
public interface OrderSavePort {
    void save(Order order);
}

// FindOrderService가 필요한 것만
public interface OrderQueryPort {
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
}

// 하나의 Adapter가 여러 Port 구현
@Repository
public class JpaOrderRepository
    implements OrderSavePort, OrderQueryPort {
    // ...
}

// PlaceOrderService는 OrderSavePort만 의존
// → OrderQueryPort 변경에 영향 없음
@Service
public class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderSavePort orderSavePort;      // ISP
    private final PaymentPort paymentPort;          // ISP
    private final UserQueryPort userQueryPort;      // ISP
    private final OrderEventPublisher eventPublisher; // ISP
}

// === DIP: 도메인이 인프라를 모름 ===

// 도메인(고수준) → 인터페이스(추상)에만 의존
@Service
public class PlaceOrderService {
    private final OrderSavePort orderSavePort;  // 추상
    private final PaymentPort paymentPort;      // 추상

    // 어디에도 JPA, Kafka, HTTP 클래스가 없음
    // = 도메인이 저수준 세부사항을 모름 = DIP 준수
}

// 인프라(저수준) → 도메인 인터페이스(추상) 구현
@Repository
public class JpaOrderRepository implements OrderSavePort { // DIP
    // JPA가 도메인 계약을 구현 = 저수준이 추상에 의존
}
```

---

## 📊 패턴 비교 — SOLID와 아키텍처 패턴의 연결

```
SOLID 원칙이 각 아키텍처 패턴에서 나타나는 위치:

원칙   │ 레이어드 아키텍처        │ Hexagonal            │ Clean Architecture
───────┼─────────────────────────┼──────────────────────┼────────────────────────
SRP    │ Presentation / Business │ Domain / Application │ Entities / Use Cases /
       │ / Data Access 레이어 분리│ / Infrastructure 분리 │ Interface Adapters 분리
───────┼─────────────────────────┼──────────────────────┼────────────────────────
OCP    │ Repository 인터페이스로  │ Adapter 추가로 기능   │ Gateway / Presenter
       │ DB 교체 (일부 적용)     │ 확장 (완전 적용)     │ 교체 (완전 적용)
───────┼─────────────────────────┼──────────────────────┼────────────────────────
LSP    │ 암묵적 (인터페이스 없음) │ Port 구현체 계약 준수 │ Gateway 계약 준수
       │ → LSP 검증 어려움       │                      │
───────┼─────────────────────────┼──────────────────────┼────────────────────────
ISP    │ 적용 적음               │ 사용 목적별 Port 분리 │ Input/Output Boundary
       │ (Repository 비대 흔함)  │ (OrderSavePort 등)   │ 로 UseCase별 분리
───────┼─────────────────────────┼──────────────────────┼────────────────────────
DIP    │ 부분 적용               │ 핵심 원리             │ 핵심 원리
       │ (Repository 인터페이스)  │ 모든 Port가 DIP      │ 의존성 규칙 = DIP
       │ Service는 JPA 직접 의존 │                      │

결론:
  레이어드: SRP 중심 (책임 분리), DIP 부분 적용
  Hexagonal: DIP + ISP 중심 (Port로 인프라 역전)
  Clean: DIP + SRP 중심 (동심원 경계가 SRP + 의존성 규칙이 DIP)
```

---

## ⚖️ 트레이드오프

```
SOLID를 아키텍처 수준에 완전히 적용할 때의 비용:

SRP 완전 적용:
  장점: 변경 이유가 명확히 분리됨
  비용: 클래스/레이어 수 증가, 파악해야 할 파일이 많아짐

OCP 완전 적용:
  장점: 기존 코드 수정 없이 확장 가능
  비용: 확장 지점을 설계 시 미리 예측해야 함 (YAGNI와 충돌 가능)

ISP 완전 적용:
  장점: UseCase별 의존성 최소화
  비용: 인터페이스 파일 수 증가, 하나의 Adapter가 여러 인터페이스 구현

DIP 완전 적용:
  장점: 도메인이 인프라를 모름, 인프라 교체 자유
  비용: 인터페이스 + 구현 클래스 + 매핑 코드 추가

실용적 접근:
  처음부터 모든 SOLID를 완전히 적용할 필요 없음
  핵심: DIP (도메인이 인프라를 모르게)
       SRP (변경 이유가 다른 것들을 분리)
  이 두 가지만 지켜도 아키텍처의 80% 이익을 얻음

과잉 적용의 신호:
  "이 클래스에는 메서드가 하나뿐인데 왜 인터페이스가 있지?"
  "이 Port는 save() 하나뿐인데 ISP를 위해 인터페이스로 분리했다"
  → 실제 교체 가능성이 없는 곳에 인터페이스 추가는 과잉
```

---

## 📌 핵심 정리

```
SOLID와 아키텍처 패턴의 연결 지도:

SRP → 레이어 분리
  "변경 이유가 다른 것들을 다른 레이어에"
  도메인(비즈니스 규칙) / 인프라(기술 구현) / 프레젠테이션(API)

OCP → Adapter 패턴
  "새 기능 추가 = 새 Adapter 클래스 추가"
  "기존 Port 인터페이스와 UseCase 코드는 수정 없음"

LSP → Adapter 교체 가능성
  "모든 Adapter는 Port 계약을 완전히 준수해야 한다"
  "JpaAdapter ↔ MongoAdapter 교체 시 UseCase 코드 동일"

ISP → Port 세분화
  "UseCase가 사용하지 않는 메서드에 의존하지 않게"
  "OrderSavePort / OrderQueryPort / OrderAdminPort 분리"

DIP → Hexagonal/Clean의 핵심
  "도메인(고수준)이 인프라(저수준)를 모르게"
  "인프라가 도메인 인터페이스를 구현 = 의존성 역전"
  "도메인 테스트에 인프라 불필요"
```

---

## 🤔 생각해볼 문제

**Q1.** `@Service` 클래스가 `@Repository`(인터페이스)에 의존하는 것은 DIP를 지키는가? `JpaRepository`(Spring Data)를 직접 상속한 인터페이스에 의존하는 것은 DIP를 지키는가?

<details>
<summary>해설 보기</summary>

**`@Repository` 인터페이스에 의존하는 것이 DIP를 더 잘 지킵니다.**

```java
// Case A: Spring Data JpaRepository 직접 의존
public interface OrderJpaRepository extends JpaRepository<Order, Long> {}

@Service
public class OrderService {
    @Autowired OrderJpaRepository repository; // ← JPA에 결합!
}
// 문제: Spring Data JPA의 JpaRepository에 의존 = JPA에 의존
// DIP 부분 위반: 추상(인터페이스)에 의존하지만, 그 추상이 JPA를 알고 있음

// Case B: 도메인 Repository 인터페이스에 의존
public interface OrderRepository { // 순수 도메인 인터페이스
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

@Repository
class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpa; // JPA는 여기에만
}

@Service
public class OrderService {
    private final OrderRepository repository; // 도메인 인터페이스에만 의존
}
// DIP 완전 준수: Service가 JPA를 전혀 모름
```

Case A는 "추상에 의존"하지만, 그 추상(JpaRepository)이 JPA 세부 사항을 포함하므로 완전한 DIP라고 보기 어렵습니다.

</details>

---

**Q2.** OCP를 지키기 위해 모든 변경 지점을 미리 인터페이스로 추상화해야 하는가? "예측하기 전에 인터페이스를 만들지 마라"는 YAGNI 원칙과 OCP는 충돌하는가?

<details>
<summary>해설 보기</summary>

**OCP와 YAGNI는 충돌하지 않습니다.** 핵심은 "실제 확장 지점"과 "가상의 확장 지점"을 구분하는 것입니다.

**OCP를 적용해야 하는 경우 (실제 확장 지점):**
- 외부 시스템 연동 (결제, 알림, 배송) — 나중에 바뀔 가능성 높음
- 환경별 다른 구현이 필요한 경우 (테스트 환경: InMemory, 운영: JPA)

**OCP를 적용하지 않아도 되는 경우 (가상의 확장 지점):**
- "나중에 바꿀 수도 있을 것 같은" 불명확한 예측
- 실제로 구현체가 하나뿐이고 교체 계획이 없는 경우

**실용적 접근 (Sandro Mancuso 방식):**
- 처음에는 구체 클래스로 시작
- 실제로 두 번째 구현체가 필요해질 때 인터페이스 추출
- 단, 외부 시스템 연동은 처음부터 인터페이스로 — 테스트를 위해서라도

결론: "외부 시스템이나 교체 가능성이 명확한 것"에만 OCP 적용. 불명확한 미래를 위한 추상화는 YAGNI 위반.

</details>

---

**Q3.** ISP를 지키기 위해 `OrderSavePort`, `OrderQueryPort`, `OrderAdminPort`로 나눴다면, 구현체인 `JpaOrderRepository`는 세 인터페이스를 모두 구현한다. 이것이 SRP 위반인가?

<details>
<summary>해설 보기</summary>

**SRP 위반이 아닙니다.** SRP는 "변경 이유"로 판단합니다.

`JpaOrderRepository`의 변경 이유는 단 하나입니다: **"JPA를 사용한 주문 영속성 구현이 바뀔 때"**

세 인터페이스를 구현한다는 것은:
- "읽기/쓰기/관리 기능을 모두 JPA로 구현한다"는 것
- 변경 이유는 여전히 하나 (JPA 구현 방식 변경)

SRP는 "메서드 수"가 아니라 "변경 이유"로 판단합니다. `JpaOrderRepository`가 save(), findById(), findAll() 등을 모두 구현해도, 그 변경 이유가 "JPA 기반 주문 영속성" 하나라면 SRP 위반이 아닙니다.

만약 SRP 위반이 되려면:
```java
class JpaOrderRepository {
    // JPA 저장 로직 + Kafka 이벤트 발행 로직이 같이 있을 때
    // → 변경 이유가 2개: JPA 변경 + Kafka 변경
    // → 이것이 SRP 위반
}
```

</details>

---

<div align="center">

**[⬅️ 이전: 변경 비용과 의존성](./02-dependency-and-change-cost.md)** | **[홈으로 🏠](../README.md)** | **[다음: 전통적 레이어드 아키텍처의 문제 ➡️](./04-layered-architecture-problems.md)**

</div>
