# Native Query · 인터페이스 프로젝션 · 트랜잭션 롤백 규칙 · cascade·orphanRemoval 정리 (TIL)

> 실습 환경: MySQL (`menudb`의 `tbl_menu`·`tbl_category` + 새로 만든 `practice_order`·`practice_order_line`), Spring Boot 4 + `spring-boot-starter-data-jpa`·`webmvc`·`validation` (Gradle 프로젝트 `chap06-native-query`, `chap07-practical-jpa`), `.http` 파일로 API 호출, JUnit 5 + `@SpringBootTest`
>
> 11/04의 `JpaRepository`·JPQL `@Query`·페이징·`open-in-view: false`, 11/03의 양방향 연관관계·`mappedBy`·연관관계 편의 메소드는 다시 설명하지 않음

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| chap06 | JPQL로는 못 쓰는 SQL(집계 보고서, **윈도 함수**)을 **Native Query**로 직접 실행하고, 결과를 **인터페이스 프로젝션**으로 받기 |
| chap07 ① | 실무형 설정: `ddl-auto: validate`, 테이블은 SQL 스크립트로, DB 접속 정보는 환경 변수로 |
| chap07 ② | `@Transactional`이 **언제 롤백하고 언제 커밋하는지** 테스트로 확인 (런타임/체크 예외, `rollbackFor`, 예외 삼키기, 내부 호출) |
| chap07 ③ | 주문-주문항목에 `cascade = PERSIST` + `orphanRemoval = true` → 부모와 생명주기를 같이하는 자식 |

---

## 1. Native Query: SQL을 그대로 실행 (chap06)

* JPQL은 **엔티티 기준**이라 DB마다 다른 기능(MySQL의 윈도 함수 등)이나, 엔티티로 딱 떨어지지 않는 **집계 결과**를 다루기 어려움. 이럴 때 `nativeQuery = true`로 **진짜 SQL**을 씀.
* **한 줄 비유**: JPQL이 통역사(Hibernate)를 거쳐 말하는 거라면, Native Query는 DB한테 **DB 말(SQL)로 직접** 말하는 것. 통역이 안 되는 표현도 쓸 수 있지만, 상대 DB가 바뀌면 다시 써야 함.

```java
@Query(value = """
    SELECT c.category_code AS categoryCode,
           c.category_name AS categoryName,
           COUNT(m.menu_code) AS menuCount
      FROM tbl_category c
      LEFT JOIN tbl_menu m ON m.category_code = c.category_code
     GROUP BY c.category_code, c.category_name
     ORDER BY menuCount DESC, c.category_code
    """, nativeQuery = true)
List<CategoryMenuCountView> findCategoryMenuCounts();
```

| | JPQL (`@Query`) | Native Query (`nativeQuery = true`) |
| :--- | :--- | :--- |
| 대상 | 엔티티·필드 (`Menu m`, `m.menuName`) | **테이블·컬럼** (`tbl_menu m`, `m.menu_name`) |
| 번역 | Hibernate가 DB에 맞게 SQL로 번역 | **그대로** DB에 보냄 |
| DB 전용 기능 | 제한적 | 윈도 함수 등 DB 기능 다 사용 가능 |
| DB 교체 시 | 그대로 동작 | 쿼리를 다시 써야 할 수 있음 |

* `"""` ... `"""` → Java **텍스트 블록**(15+). 여러 줄 SQL을 `+`로 이어 붙이지 않고 보이는 그대로 쓸 수 있음.
* 카테고리별 메뉴 수에서 **`LEFT JOIN`** 을 쓴 이유 → 메뉴가 하나도 없는 카테고리도 `0`으로 보여 주려고. `INNER JOIN`이면 그런 카테고리는 아예 빠짐.
* `COUNT(*)`가 아니라 **`COUNT(m.menu_code)`** 인 이유 → `LEFT JOIN`에서 짝이 없는 카테고리도 행 1개(메뉴 쪽은 전부 `NULL`)로 나옴. `COUNT(*)`는 그 행까지 세서 `1`이 되고, `COUNT(컬럼)`은 `NULL`을 안 세서 정확히 `0`이 됨 (10월 DB 수업의 집계 함수 내용).

### 인터페이스 프로젝션: 결과를 getter만 있는 인터페이스로 받기

```java
public interface CategoryMenuCountView {
    int getCategoryCode();
    String getCategoryName();
    long getMenuCount();
}
```

* 집계 결과는 `Menu` 엔티티 모양이 아님 → 엔티티로 못 받음. 그렇다고 `Object[]`로 받으면 `row[0]`, `row[1]`처럼 **순서로** 꺼내야 해서 읽기 어려움.
* **인터페이스에 getter만** 선언해 두면, Spring Data가 각 행을 이 인터페이스로 읽을 수 있는 **구현 객체(프록시)를 대신 만들어** 줌.
* 연결 규칙: SQL의 **별칭(`AS categoryCode`)** ↔ getter 이름(`getCategoryCode`)에서 `get`을 뗀 이름. 별칭이 다르면 값이 `null`로 나오거나 에러.
* 컨트롤러에서 그대로 반환하면 Jackson이 **getter를 보고** JSON을 만듦 → `{"categoryCode": 1, "categoryName": "...", "menuCount": 5}`.
* **한 줄 비유**: "이 칸에는 이름, 저 칸에는 개수를 적어 주세요"라는 **빈 양식(인터페이스)** 만 주면, Spring Data가 SQL 결과를 그 양식에 채워 주는 것.

> 💡 `COUNT(...)`나 `RANK()`는 MySQL에서 `BIGINT`로 나옴 → getter도 `long`으로 받는 게 안전함.

---

## 2. 윈도 함수 RANK(): 가격 순위 매기기

```java
@Query(value = """
    SELECT m.menu_code AS menuCode,
           m.menu_name AS menuName,
           m.menu_price AS menuPrice,
           -- RANK의 ORDER BY: 각 메뉴에 몇 위를 붙일지 계산하는 기준
           RANK() OVER (ORDER BY m.menu_price DESC) AS priceRank
      FROM tbl_menu m
      -- 클라이언트에 반환할 순서
     ORDER BY priceRank, m.menu_code
    """, nativeQuery = true)
List<MenuPriceRankView> findMenuPriceRanks();
```

* **윈도 함수** → `GROUP BY`처럼 행을 하나로 **합치지 않고**, 각 행은 그대로 둔 채 옆에 "**전체 중 몇 등**" 같은 계산 값을 붙여 줌. MySQL 8.0부터 지원.
* `OVER (ORDER BY m.menu_price DESC)` → 비싼 순으로 줄 세워서 순위를 계산.
* `ORDER BY`가 **두 번** 나오는 게 헷갈리는 부분:
  * `OVER (...)` 안의 `ORDER BY` → **순위를 매기는 기준**
  * 쿼리 맨 끝의 `ORDER BY` → **결과를 보여 주는 순서**
  * 안쪽만 있고 바깥이 없으면 순위는 맞아도 결과 순서는 보장되지 않음.

### 같은 가격이면? RANK · DENSE_RANK · ROW_NUMBER

| 가격 | `RANK()` | `DENSE_RANK()` | `ROW_NUMBER()` |
| :--- | :---: | :---: | :---: |
| 30000 | 1 | 1 | 1 |
| 30000 | 1 | 1 | 2 |
| 25000 | **3** | **2** | 3 |

* `RANK()` → 공동 순위 다음은 **건너뜀** (1, 1, 3). 올림픽 메달 순위처럼.
* `DENSE_RANK()` → 건너뛰지 않음 (1, 1, 2).
* `ROW_NUMBER()` → 같아도 무조건 다른 번호 (1, 2, 3).

```http
### 카테고리별 메뉴 수 - Native Query(인터페이스 프로젝션)
GET http://localhost:8080/api/v1/menus/reports/category-menu-counts

### 메뉴 가격 순위 - MySQL 윈도 함수
GET http://localhost:8080/api/v1/menus/reports/price-ranks
```

* 서비스는 `@Transactional(readOnly = true)`로 감싸서 Repository 결과를 그대로 반환. 보고서용이라 엔티티를 거치지 않으니 `toResponse` 같은 변환도 필요 없음.
* URL은 `/reports/...`로 묶어서 "**조회 전용 보고서**"라는 걸 드러냄. 두 칸짜리라 `/{menuCode}`와도 안 겹침.

> ⚠️ Native Query 결과(프로젝션)는 **엔티티가 아님** → 영속성 컨텍스트가 관리하지 않으니 값을 바꿔도 변경 감지로 `UPDATE`가 나가지 않음. 조회·보고서 용도로만 씀.

---

## 3. chap07 실무형 설정

```yaml
spring:
  datasource:
    url: ${JPA_DB_URL:jdbc:mysql://localhost:3306/menudb}
    username: ${JPA_DB_USERNAME:who307}
    password: ${JPA_DB_PASSWORD:who307}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```

* `${JPA_DB_URL:기본값}` → **환경 변수** `JPA_DB_URL`이 있으면 그 값, 없으면 `:` 뒤 기본값. 비밀번호 같은 값을 코드에 박지 않고 실행 환경에서 바꿔 넣을 수 있음.
* `ddl-auto: validate` → 시작할 때 **엔티티와 테이블이 맞는지 검사만** 함. 테이블을 만들거나 고치지 않음. 컬럼이 없거나 타입이 다르면 앱이 **시작하지 못함**.
  * 그래서 새 테이블(`practice_order`, `practice_order_line`)은 `sql/01-create-order-tables.sql`로 **직접** 만들어 둠. 기존 `tbl_menu`·`tbl_category`는 건드리지 않음.
  * 11/03의 `create`/`update`는 연습용, 실무에서는 `validate`나 `none` + 스크립트로 관리.
* `format_sql: true` → 콘솔에 찍히는 SQL을 줄바꿈해서 보기 좋게.

### 달라진 코드 스타일

```java
// DTO를 record로
public record OrderResponse(Long id, String customer, List<LineResponse> lines,
                            LocalDateTime createdAt, LocalDateTime updatedAt) {}

// 상태 코드를 어노테이션으로
@PostMapping("/orders")
@ResponseStatus(HttpStatus.CREATED)
public OrderService.OrderResponse create(@Valid @RequestBody CustomerRequest request) { ... }
```

* **`record`** → `private final` 필드 + 순서대로 받는 생성자 + 값 읽는 메소드(`id()`, `customer()`)를 **자동으로** 만들어 주는 문법. 값만 옮기는 DTO에 딱 맞음. getter 이름이 `getId()`가 아니라 **`id()`** 인 점 주의.
* `@ResponseStatus(CREATED)` → `ResponseEntity` 없이 DTO를 바로 반환하면서 상태 코드만 `201`로 지정.
* `Menu`의 PK가 `int menuCode` → **`Integer menuCode`** 로 바뀜. 어제는 "`0`이면 새 엔티티"였는데, 래퍼 타입이면 **`null`이면 새 엔티티**라 의미가 더 분명함.

---

## 4. @Transactional 롤백 규칙 (chap07 ②)

* `@Transactional`은 메소드 자체가 아니라 Spring이 만든 **프록시**가 처리함: 프록시가 트랜잭션 시작 → 진짜 메소드 호출 → **예외가 밖으로 나오는지** 보고 커밋/롤백 결정.
* **한 줄 비유**: 프록시는 문 앞의 **경비원**. 안에서 무슨 일이 있었든, 경비원은 "**문 밖으로 어떤 예외가 들고 나오는지**"만 보고 판단함.

```java
@Transactional
public void failRuntime(String customer) {
    orders.saveAndFlush(new PurchaseOrder(customer));
    throw new IllegalStateException("저장 후 실패");
}

@Transactional
public void failCheckedDefault(String customer) throws IOException {
    orders.saveAndFlush(new PurchaseOrder(customer));
    throw new IOException("체크 예외 기본 규칙 확인");
}

@Transactional(rollbackFor = IOException.class)
public void failCheckedRollback(String customer) throws IOException {
    orders.saveAndFlush(new PurchaseOrder(customer));
    throw new IOException("체크 예외도 롤백");
}
```

* `saveAndFlush` → 저장하고 **바로 `flush`** 해서 `INSERT`를 DB에 보냄. "SQL은 이미 나갔는데 롤백으로 취소된다"는 상황을 확실하게 만들려고 씀.

| 상황 | 결과 | 테스트에서 남은 주문 수 |
| :--- | :--- | :---: |
| `RuntimeException`(언체크) 던짐 | **롤백** | 0 |
| `IOException`(체크 예외) 던짐 | **커밋** ❗ | 1 |
| 체크 예외 + `rollbackFor = IOException.class` | 롤백 | 0 |
| 안에서 `try-catch`로 예외를 잡음 | 커밋 | 1 |
| 같은 클래스 안에서 `@Transactional` 메소드 호출 | 트랜잭션 **적용 안 됨** | 1 |

* Spring 기본 규칙: **`RuntimeException`과 `Error`만 롤백**, 체크 예외는 커밋. "체크 예외 = 호출한 쪽이 처리할 수 있는 예상된 상황"이라고 보기 때문.
* 체크 예외에도 롤백하고 싶으면 `rollbackFor`로 직접 지정.

### 예외를 잡아 버리면

```java
@Transactional
public void swallowFailure(String customer) {
    try {
        orders.saveAndFlush(new PurchaseOrder(customer));
        throw new IllegalStateException("업무 단계 실패");
    } catch (IllegalStateException e) {
        // 프록시 밖으로 전달되지 않는다 → 프록시는 정상 반환을 보므로 커밋
    }
}
```

* 예외가 런타임 예외여도 **메소드 안에서 잡으면** 프록시는 "정상 종료"로 봄 → 커밋. 실패했는데 저장은 남는 상황이 생김.
* 잡아야 한다면 로그만 찍고 끝내지 말고, 다시 던지거나(`throw`) 롤백이 필요한지 따져 봐야 함.

> 💡 잡은 예외가 **다른 `@Transactional` 빈의 메소드**에서 터진 거라면 얘기가 달라짐 → 그쪽 프록시가 이미 트랜잭션에 "롤백만 가능(rollback-only)" 표시를 해 둬서, 바깥에서 커밋하려는 순간 `UnexpectedRollbackException`이 남. 오늘 실습은 같은 메소드 안에서 던지고 잡아서 이 표시가 안 생김.

### 내부 호출(self-invocation)

```java
// @Transactional 없음
public void selfInvocation(String customer) {
    failRuntime(customer);   // = this.failRuntime(...)
}
```

* 컨트롤러·테스트가 `orders.selfInvocation()`을 부르면 프록시를 거치지만, 그 안의 `failRuntime()`은 **`this`(진짜 객체)** 를 직접 부름 → 프록시(경비원)를 **안 거침** → `failRuntime`의 `@Transactional`이 무시됨.
* 그럼 왜 주문이 1건 남나? → `saveAndFlush`는 Spring Data의 `SimpleJpaRepository`에 **자체 `@Transactional`** 이 붙어 있어서, 저장만 따로 트랜잭션을 열고 **바로 커밋**함. 그 뒤 예외가 터져도 되돌릴 트랜잭션이 없음.
* 해결: 트랜잭션이 필요한 메소드를 **다른 빈(클래스)으로 분리**해서 프록시를 거치게 하거나, 바깥 메소드(`selfInvocation`)에 `@Transactional`을 붙임.

---

## 5. 테스트: 클래스에 @Transactional을 안 붙이는 이유

```java
@SpringBootTest
abstract class FixtureSupport {
    @Autowired protected OrderService orders;
    @Autowired protected OrderRepository orderRepository;
    @Autowired protected EntityManagerFactory factory;
    ...
    protected String prefix;

    @BeforeEach
    void prepareOwnFixtures() {
        prefix = "J7" + UUID.randomUUID().toString()
                .replace("-", "").substring(0, 16) + "-";
        ...
    }

    @AfterEach
    void cleanupOwnFixtures() {
        for (PurchaseOrder order : orderRepository.findByCustomerStartingWith(prefix)) {
            orders.delete(order.getId());
        }
        ...
    }

    protected long ownOrderCount() {
        return orderRepository.countByCustomerStartingWith(prefix);
    }
}
```

* 11/04 테스트는 클래스에 `@Transactional` → 끝나면 전부 롤백. 그런데 오늘은 **서비스가 실제로 커밋했는지/롤백했는지**를 보는 게 목적이라, 테스트가 트랜잭션을 쥐고 있으면 서비스 트랜잭션이 테스트 트랜잭션에 **합쳐져서** 결과를 확인할 수 없음.
* 그래서 롤백 없이 **진짜로 커밋**시키고, 대신 정리를 직접 함:
  * 테스트마다 **UUID로 만든 고유 접두어**(`J7xxxx-`)를 고객명·메뉴명에 붙임 → 공용 DB에 있는 기존 데이터나 다른 테스트 데이터와 안 섞임.
  * 검증은 `countByCustomerStartingWith(prefix)` → **내 테스트가 만든 주문만** 셈 (쿼리 메소드 `StartingWith` = `LIKE '값%'`).
  * `@AfterEach`에서 그 접두어로 찾은 주문과 직접 만든 메뉴만 지움.
* `abstract class` + 상속 → 테스트 클래스(`TransactionTests`, `OrderLifecycleTests`)마다 같은 준비·정리 코드를 반복하지 않음.

---

## 6. cascade + orphanRemoval: 부모와 생명주기를 같이하는 자식 (chap07 ③)

```java
@Entity
@Table(name = "practice_order")
public class PurchaseOrder extends BaseTimeEntity {
    ...
    @OneToMany(mappedBy = "order", cascade = CascadeType.PERSIST, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();

    public void addLine(OrderLine line) {
        lines.add(line);
        line.assignOrder(this);
    }

    public void removeLine(Long lineId) {
        OrderLine line = lines.stream().filter(item -> item.getId().equals(lineId))
                .findFirst().orElseThrow(() -> new NoSuchElementException("주문항목 없음"));
        lines.remove(line);
        line.assignOrder(null);   // 메모리의 자식 참조도 끊는다
    }
}
```

```java
@Entity
@Table(name = "practice_order_line")
public class OrderLine {
    ...
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private PurchaseOrder order;

    public OrderLine(String menuName, int quantity) {
        if (quantity < 1) throw new IllegalArgumentException("수량은 1 이상이어야 합니다.");
        ...
    }

    void assignOrder(PurchaseOrder order) { this.order = order; }   // package-private
}
```

* **한 줄 비유**: 주문서(부모)와 주문서에 적힌 **줄(자식)**. 줄은 주문서 없이는 의미가 없음 → 주문서를 만들면 줄도 같이 생기고, 줄을 지우거나 주문서를 버리면 줄도 같이 사라짐.

| 설정 | 하는 일 | 오늘 확인한 동작 |
| :--- | :--- | :--- |
| `cascade = PERSIST` | 부모를 저장하면 **자식도 같이 저장** | `orders.saveAndFlush(order)` 한 번에 주문 + 항목 2개 `INSERT` |
| `orphanRemoval = true` | 부모 컬렉션에서 **빠진 자식**(고아)을 `DELETE` | `lines.remove(line)` → 해당 항목 `DELETE` |
| (`orphanRemoval`) | 부모를 삭제하면 **자식도 삭제** | `orders.delete(order)` → 항목들도 `DELETE` |

* `cascade`에 `REMOVE`를 안 넣었는데도 부모 삭제 시 자식이 지워짐 → `orphanRemoval = true`는 "부모가 사라지면 자식도 고아"로 보고 삭제까지 해 줌.
* `removeLine`에는 `delete`나 `remove` 호출이 **없음** → 컬렉션에서 빼기만 하면 끝. 11/02의 변경 감지처럼, 컬렉션 변화도 `flush` 때 SQL로 바뀜.
* `assignOrder`는 `public`이 아니라 **package-private** → 같은 패키지의 `PurchaseOrder.addLine()`만 부를 수 있음. 바깥에서 자식 쪽만 바꿔서 양쪽이 어긋나는 걸 막음 (11/03 연관관계 편의 메소드 연장).
* 수량 검증을 **엔티티 생성자**에서 함 → 수량 0짜리 항목은 객체부터 만들어지지 않음.

> ⚠️ cascade·orphanRemoval은 **자식이 그 부모에만 속할 때만** 씀. 주문항목은 이 주문에만 속하니 OK. 반대로 `Menu`→`Category`처럼 여러 곳에서 같이 쓰는 대상에 걸면, 메뉴 하나 지울 때 카테고리까지 지워지는 사고가 남. 그래서 주문항목도 `Menu` 엔티티를 참조하지 않고 `menuName` 문자열만 복사해 둠 → 항목을 지워도 **메뉴는 안전**.

### 생명주기 테스트

```java
@Test
void cascadePersistAndOrphanRemoval() {
    OrderService.OrderResponse created = orders.create(prefix + "학생");
    assertEquals(2, orders.find(created.id()).lines().size());

    Long removedId = created.lines().get(0).id();
    orders.removeLine(created.id(), removedId);
    assertEquals(1, orders.find(created.id()).lines().size());

    EntityManager em = factory.createEntityManager();
    try {
        assertNull(em.find(OrderLine.class, removedId));
    } finally { em.close(); }
}
```

* `factory.createEntityManager()` → **새 영속성 컨텍스트**(빈 1차 캐시)를 직접 만들어 DB에서 다시 조회. 서비스 트랜잭션은 이미 끝났으니 1차 캐시 덕에 "있는 것처럼" 보이는 착각 없이 **DB에 진짜 없는지** 확인.
* 직접 만든 `EntityManager`는 Spring이 관리하지 않으므로 `finally`에서 **꼭 `close()`**.
* `deletingParentAlsoDeletesOwnedLines` → 부모 삭제 후 항목 번호들로 `em.find`가 전부 `null`이고, 내 주문 수가 0인지 확인.

```http
@base = http://localhost:8080/api/v1/practice
@orderId = 8
@lineId = 1

### 주문 생성: 응답 id/lines[0].id를 위 변수에 복사
POST {{base}}/orders
Content-Type: application/json

{"customer":"홍길동"}

### 항목 하나 제거
DELETE {{base}}/orders/{{orderId}}/lines/{{lineId}}

### 부모 제거 → 이후 조회는 404
DELETE {{base}}/orders/{{orderId}}
```

* `.http` 파일 맨 위 `@이름 = 값` → **변수**. 요청에서 `{{이름}}`으로 꺼내 씀. 응답에서 나온 번호만 바꿔 주면 아래 요청 URL이 다 따라 바뀜.
* 항목 삭제 URL이 `/orders/{id}/lines/{lineId}` → "**이 주문의** 항목"이라는 소속을 경로로 표현.

> 📌 chap07의 `BaseTimeEntity`(JPA Auditing, `@CreatedDate`·`@LastModifiedDate`)와 `MenuSpecifications`(동적 검색, `Specification`)는 아직 `TODO` 상태 → 다음 수업 내용.

---

## 📝 오늘의 정리

1. JPQL로 표현하기 어려운 집계·DB 전용 기능은 `nativeQuery = true`로 SQL을 직접 실행하고, 테이블·컬럼 이름을 씀
2. Native Query 결과는 getter만 있는 **인터페이스 프로젝션**으로 받으며, SQL 별칭과 getter 이름(`get` 뺀 부분)이 맞아야 함
3. `LEFT JOIN` + `COUNT(컬럼)`으로 짝이 없는 행도 `0`으로 셀 수 있고, `RANK() OVER (ORDER BY ...)`로 행을 합치지 않고 순위를 붙임 (공동 순위 뒤는 건너뜀)
4. `ddl-auto: validate`는 엔티티와 테이블이 맞는지 검사만 하고, 테이블은 SQL 스크립트로 관리하며, 접속 정보는 `${환경변수:기본값}`으로 뺌
5. `@Transactional`은 프록시가 처리하며 기본적으로 **런타임 예외만 롤백**, 체크 예외는 커밋. 체크 예외도 롤백하려면 `rollbackFor`
6. 메소드 안에서 예외를 잡으면 커밋되고, 같은 클래스 안에서 부르는 `@Transactional` 메소드는 프록시를 안 거쳐서 트랜잭션이 적용되지 않음
7. 커밋/롤백 결과를 검증하는 테스트는 클래스에 `@Transactional`을 빼고, UUID 접두어로 내 데이터만 세고 `@AfterEach`에서 정리함
8. `cascade = PERSIST`는 부모 저장 시 자식도 저장, `orphanRemoval = true`는 컬렉션에서 빠진 자식과 삭제된 부모의 자식을 `DELETE`. 한 부모에만 속하는 자식에만 씀

---

## 느낀 점

* 어제는 JPQL만으로 다 될 것 같았는데, 카테고리별 개수나 순위처럼 보고서성 조회는 결국 SQL을 직접 쓰는 게 자연스럽다는 걸 느낌. 10월에 배운 `LEFT JOIN`·`COUNT`가 다시 쓰여서 반가웠음
* 인터페이스에 getter만 적어 두면 결과가 채워지는 게 쿼리 메소드만큼 신기했는데, 별칭 하나만 틀려도 안 맞는다는 점은 조심해야겠음
* 체크 예외는 롤백이 안 된다는 게 가장 의외였음. `@Transactional`만 붙이면 무조건 안전하다고 생각했는데, 예외 종류와 잡는 위치에 따라 결과가 달라진다는 걸 테스트로 직접 확인하니 확실히 기억에 남음
* 내부 호출에서 트랜잭션이 무시되는 건 "프록시가 문 앞에서 처리한다"는 그림을 떠올리니 이해가 됐음
* `removeLine`에 삭제 코드가 없는데 `DELETE`가 나가는 걸 보고, JPA는 결국 "객체를 바꾸면 SQL은 알아서"라는 방향이라는 게 다시 와닿음
