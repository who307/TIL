# JPA Auditing(생성·수정 시간 자동 기록) · Specification 동적 검색 정리 (TIL)

> 실습 환경: MySQL (`menudb`의 `tbl_menu`·`tbl_category`·`practice_order`), Spring Boot 4 + Spring Data JPA (Gradle 프로젝트 `chap07-practical-jpa`), JUnit 5 + `@SpringBootTest`
>
> 11/05에 정리한 chap07 설정(`ddl-auto: validate`·환경 변수·`record` DTO), 트랜잭션 롤백 규칙, `FixtureSupport`(클래스에 `@Transactional` 없이 테스트), `cascade`·`orphanRemoval`은 다시 설명하지 않음

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| chap07 ④ Auditing | `BaseTimeEntity` + `@EnableJpaAuditing`으로 `createdAt`·`updatedAt` **자동 기록** |
| chap07 ⑤ 동적 검색 | `Specification` + `JpaSpecificationExecutor`로 "**있는 조건만**" 골라 붙이는 검색 + 페이징 |

---

## 1. JPA Auditing: 생성·수정 시간 자동 기록

* 주문을 만들 때 `createdAt = LocalDateTime.now()`, 고칠 때마다 `updatedAt = now()`를 **매번 직접** 쓰면 빠뜨리기 쉬움. 엔티티가 10개면 10군데.
* **Auditing**(감사, 기록) → "언제 만들어졌고 언제 바뀌었는지"를 Spring Data JPA가 **자동으로** 채워 주는 기능.
* **한 줄 비유**: 서류를 낼 때마다 직접 날짜를 쓰는 대신, 창구에 **자동 날짜 도장**을 달아 두는 것. 처음 접수할 때 "접수일" 도장, 고칠 때마다 "수정일" 도장이 찍힘.

### 1) 켜기: @EnableJpaAuditing

```java
@EnableJpaAuditing
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

* 이걸 안 붙이면 아래 어노테이션을 다 달아도 **아무 값도 안 들어감**(`null`). 스위치 역할.

### 2) 공통 부모 클래스: BaseTimeEntity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTimeEntity {
    @CreatedDate
    private LocalDateTime createdAt;
    @LastModifiedDate
    private LocalDateTime updatedAt;

    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

| 어노테이션 | 뜻 |
| :--- | :--- |
| `@MappedSuperclass` | 나는 **테이블이 없는** 부모. 내 필드는 상속받은 **자식 엔티티의 테이블 컬럼**으로 들어감 |
| `@EntityListeners(AuditingEntityListener.class)` | 저장·수정 **직전**에 Spring의 감시자(리스너)가 끼어들어 날짜를 채움 |
| `@CreatedDate` | 처음 저장(`INSERT`)될 때 한 번 기록 |
| `@LastModifiedDate` | 저장될 때 + 수정(`UPDATE`)될 때마다 기록 |

* `abstract` → `BaseTimeEntity` 혼자서는 쓸 일이 없으니 `new`로 못 만들게 막음.
* setter가 없음 → 개발자가 날짜를 직접 바꿀 수 없고 리스너만 채움.
* 11/03의 `@Embedded`는 "**부품을 필드로** 끼워 넣기", `@MappedSuperclass`는 "**부모에게 물려받기**". 둘 다 별도 테이블 없이 컬럼만 합쳐진다는 점은 같음.

### 3) 자식 엔티티는 상속만

```java
@Entity
@Table(name = "practice_order")
public class PurchaseOrder extends BaseTimeEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false)
    private String customer;
    ...
}
```

```sql
CREATE TABLE practice_order (
    id BIGINT NOT NULL AUTO_INCREMENT,
    customer VARCHAR(255) NOT NULL,
    created_at DATETIME(6),
    updated_at DATETIME(6),
    PRIMARY KEY (id)
)
```

* `extends BaseTimeEntity` 한 줄로 `createdAt`·`updatedAt`이 `practice_order` 테이블의 `created_at`·`updated_at` 컬럼에 연결됨 (카멜 → 스네이크 이름 변환은 Spring Boot 기본 전략, 11/03 내용).
* `ddl-auto: validate`라서 이 두 컬럼이 테이블에 **미리 있어야** 앱이 뜸 → SQL 스크립트에 넣어 둠.
* `DATETIME(6)` → 소수점 아래 **6자리(마이크로초)** 까지 저장. 그냥 `DATETIME`이면 초 단위라 아래 테스트처럼 짧은 간격의 수정을 구분하지 못함.

### 언제 값이 채워지나: flush 시점

```java
@Transactional
public OrderResponse rename(Long id, String customer) {
    PurchaseOrder order = findEntity(id);
    order.changeCustomer(customer);
    orders.flush();
    return toResponse(order);
}
```

* 리스너는 **JPA가 실제로 `INSERT`/`UPDATE`를 하기 직전**(`@PrePersist`·`@PreUpdate` 시점)에 날짜를 채움.
* 수정은 변경 감지로 `UPDATE`가 나가는데, 그건 보통 **트랜잭션이 끝날 때**(= 메소드가 `return`한 뒤) 일어남.
* `flush()` 없이 `toResponse(order)`를 하면 → 아직 `UPDATE` 전이라 `updatedAt`이 **예전 값**인 채로 응답이 만들어짐.
* 그래서 `flush()`로 `UPDATE`를 먼저 보내 → 그 직전에 리스너가 `updatedAt`을 새로 채움 → 그다음 응답을 만듦.
* **따라가기**
  1. `findEntity(id)` → `SELECT` (영속 상태, `updatedAt` = 예전 시각)
  2. `changeCustomer(...)` → 객체만 바뀜
  3. `flush()` → 변경 감지 → 리스너가 `updatedAt = now()` → `UPDATE ... SET customer=?, updated_at=?`
  4. `toResponse(order)` → 새 `updatedAt`이 담긴 응답

### 테스트

```java
class AuditingTests extends FixtureSupport {
    @Test
    void auditingKeepsCreationAndUpdatesModificationTime() throws InterruptedException {
        // DB 저장 정밀도에 맞춘 재조회 값을 기준으로 비교한다.
        Long id = orders.create(prefix + "수정 전").id();
        OrderService.OrderResponse created = orders.find(id);
        assertNotNull(created.createdAt());
        assertNotNull(created.updatedAt());

        Thread.sleep(30); // DATETIME(6)의 저장 정밀도보다 확실하게 구분한다.
        OrderService.OrderResponse updated = orders.rename(created.id(), prefix + "수정 후");

        assertEquals(created.createdAt(), updated.createdAt());
        assertTrue(updated.updatedAt().isAfter(created.updatedAt()));
    }
}
```

| 검사 | 뜻 |
| :--- | :--- |
| `createdAt`·`updatedAt`이 `null`이 아님 | 저장할 때 **둘 다** 채워짐 (처음 저장도 "수정"으로 보고 `updatedAt`도 같이 채움) |
| 수정 전후 `createdAt`이 같음 | 생성일은 **처음 한 번만** 기록되고 안 바뀜 |
| 수정 후 `updatedAt`이 더 나중 | 수정할 때마다 **새로** 기록됨 |

* `create()` 결과를 바로 쓰지 않고 `find(id)`로 **다시 조회**한 값을 기준으로 비교함.
  * 자바의 `LocalDateTime.now()`는 나노초(9자리)까지 있는데 DB는 마이크로초(6자리)까지만 저장함 → 메모리 값과 DB 값이 끝자리에서 달라질 수 있음.
  * DB에서 다시 읽은 값끼리 비교해야 `assertEquals(createdAt)`가 흔들리지 않음.
* `Thread.sleep(30)` → 생성과 수정이 같은 순간에 일어나서 `isAfter`가 `false`가 되는 일을 막으려고 30ms 쉼.
* 클래스에 `@Transactional`이 없으니(11/05 `FixtureSupport`) 서비스 메소드마다 **실제로 커밋**되고, 그 결과를 새 트랜잭션으로 다시 조회해서 확인하는 구조.

> 💡 `@CreatedDate` 필드에 `@Column(updatable = false)`를 같이 붙이면, 실수로 값이 바뀌어도 `UPDATE` 문에 **아예 포함되지 않아서** 생성일이 더 확실하게 보호됨.

> 💡 같은 방식으로 `@CreatedBy`·`@LastModifiedBy`를 쓰면 "**누가**" 만들고 고쳤는지도 자동 기록할 수 있음. 이때는 현재 로그인 사용자를 알려 주는 `AuditorAware` Bean이 추가로 필요함 (Spring Security와 연결).

---

## 2. Specification: 동적 검색

### 왜 필요한가

* 메뉴 검색 화면에 **이름 / 최소 가격 / 최대 가격 / 카테고리** 4개의 검색칸이 있음. 사용자는 아무 칸이나 **비워 둘 수 있음**.
* 11/04의 쿼리 메소드로 하려면 조합마다 메소드가 필요함 → `findByMenuNameContaining`, `findByMenuNameContainingAndMenuPriceGreaterThanEqual`, … 조건 4개면 조합이 **16가지**.
* JPQL 문자열을 `if`로 이어 붙이는 방법도 있지만, `" AND "` 공백 하나 빠뜨리면 실행해 봐야 에러를 앎.
* **Specification** → 조건 하나하나를 **자바 객체**로 만들어서, 값이 **있는 조건만** 목록에 담고 마지막에 `AND`로 묶음.
* **한 줄 비유**: 메뉴를 고르는 **체크리스트**. 손님이 체크한 칸만 골라 주방에 넘기고, 체크 안 한 칸은 아예 안 넘김.

### 1) Repository에 JpaSpecificationExecutor 추가

```java
public interface MenuRepository extends JpaRepository<Menu, Integer>, JpaSpecificationExecutor<Menu> {
}
```

* `JpaSpecificationExecutor<Menu>`를 같이 상속하면 `findAll(Specification<Menu> spec, Pageable pageable)` 같은 메소드가 **추가로** 생김 → 조건 객체를 넘겨서 조회.
* 페이징도 그대로 됨 → `Page`와 전체 개수 쿼리까지 자동 (11/04 페이징 내용).

### 2) 조건 만들기: MenuSpecifications

```java
public final class MenuSpecifications {
    private MenuSpecifications() {}

    public static Specification<Menu> search(String keyword, Integer minPrice,
                                             Integer maxPrice, Integer categoryCode) {
        return (root, query, cb) -> {
            ArrayList<Predicate> conditions = new ArrayList<>(); // 검색 조건을 담기 위한 목록

            if (keyword != null && !keyword.isBlank()) {
                String pattern = "%" + escapeLike(keyword) + "%";
                conditions.add(cb.like(root.get("menuName"), pattern, '!'));
            }
            if (minPrice != null) {
                conditions.add(cb.greaterThanOrEqualTo(root.get("menuPrice"), minPrice));
            }
            if (maxPrice != null) {
                conditions.add(cb.lessThanOrEqualTo(root.get("menuPrice"), maxPrice));
            }
            if (categoryCode != null) {
                conditions.add(cb.equal(root.get("category").get("categoryCode"), categoryCode));
            }

            return cb.and(conditions.toArray(new Predicate[0]));
        };
    }

    // LIKE에서 %, _를 검색 패턴이 아닌 입력 문자로 취급하기 위한 처리
    static String escapeLike(String keyword) {
        return keyword.replace("!", "!!").replace("%", "!%").replace("_", "!_");
    }
}
```

* `Specification<Menu>`는 메소드가 하나뿐인 함수형 인터페이스 → **람다** `(root, query, cb) -> { ... }`로 구현 (10/19 람다 내용).

| 람다 매개변수 | 타입 | 역할 | 비유 |
| :--- | :--- | :--- | :--- |
| `root` | `Root<Menu>` | 조회 대상 `Menu`. `root.get("menuName")`으로 **필드**를 가리킴 | 검색할 **물건** |
| `query` | `CriteriaQuery<?>` | 쿼리 전체 (정렬·중복 제거 등) | 주문서 **전체** |
| `cb` | `CriteriaBuilder` | `like`, `equal`, `>=`, `and` 같은 **조건을 만드는 도구** | 조건을 찍어 내는 **도장 세트** |

* `Predicate` → 조건 **하나**(`menuPrice >= 2000` 같은 것)를 나타내는 객체.
* `root.get("필드명")` → JPQL처럼 **엔티티 필드 이름**을 씀 (`menu_price` 아님).
* `root.get("category").get("categoryCode")` → 연관 엔티티의 필드까지 따라감. `categoryCode`는 카테고리의 PK라서 JOIN 없이 `tbl_menu.category_code`(FK)로 바로 비교됨.

### 3) 있는 조건만 담고, 마지막에 AND

* **따라가기**: `keyword="마늘", minPrice=2000, maxPrice=null, categoryCode=null`
  1. `keyword` 있음 → `menuName LIKE '%마늘%'` 추가
  2. `minPrice` 있음 → `menuPrice >= 2000` 추가
  3. `maxPrice`, `categoryCode`는 `null` → **건너뜀**
  4. `cb.and(조건 2개)` → `WHERE menu_name LIKE ? AND menu_price >= ?`
* 아무 조건도 없으면 → `cb.and()`에 **빈 배열** → "항상 참" → `WHERE` 없이 **전체** 조회.
* `conditions.toArray(new Predicate[0])` → `cb.and(...)`가 가변 인자(`Predicate...`)라서 `List`를 **배열로** 바꿔 넘김. `new Predicate[0]`은 "`Predicate` 타입 배열로 만들어 줘"라는 표시 (크기는 알아서 맞춰짐).
* `MenuSpecifications`는 `final` 클래스 + `private` 생성자 → 객체를 만들 일이 없는 **정적 메소드 모음**(유틸 클래스).

### 4) LIKE 이스케이프

* 사용자가 검색창에 `%`를 입력하면? `LIKE '%%%'` → `%`가 "아무 글자나"라는 **패턴 기호**로 해석돼서 **전체 메뉴**가 검색됨. `_`도 "아무 글자 1개"라는 기호.
* `escapeLike` → 입력 안의 `%`, `_` 앞에 `!`를 붙여 `!%`, `!_`로 바꿈.
* `cb.like(..., pattern, '!')` → 세 번째 인자 `'!'`가 **이스케이프 문자** → "`!` 바로 뒤 글자는 기호가 아니라 **글자 그대로**"라는 뜻.
* `!` 자체도 입력될 수 있으니 **먼저** `!` → `!!`로 바꾸고 나서 `%`, `_`를 바꿈 (순서가 바뀌면 방금 붙인 `!`가 또 바뀜).

| 입력 | 바꾼 패턴 | 의미 |
| :--- | :--- | :--- |
| `마늘` | `%마늘%` | "마늘"이 들어간 메뉴 |
| `50%` | `%50!%%` | "50%"라는 **글자**가 들어간 메뉴 |
| `%` | `%!%%` | "%" 글자가 들어간 메뉴 (전체 아님) |

### 5) 서비스: 입력 검증 + 조회 + DTO 변환

```java
@Transactional(readOnly = true)
public Page<MenuResponse> search(String keyword, Integer minPrice,
                                 Integer maxPrice, Integer categoryCode, Pageable pageable) {
    if ((minPrice != null && minPrice < 0) || (maxPrice != null && maxPrice < 0)
            || (minPrice != null && maxPrice != null && minPrice > maxPrice)) {
        throw new IllegalArgumentException("가격 범위를 확인하세요.");
    }
    return menus.findAll(MenuSpecifications.search(keyword, minPrice, maxPrice, categoryCode),
            pageable).map(MenuResponse::from);
}
```

```java
public record MenuResponse(Integer menuCode, String menuName, int menuPrice,
                           int categoryCode, String categoryName) {
    public static MenuResponse from(Menu menu) {
        return new MenuResponse(menu.getMenuCode(), menu.getMenuName(), menu.getMenuPrice(),
                menu.getCategory().getCategoryCode(),
                menu.getCategory().getCategoryName());
    }
}
```

* 음수 가격, 최소 > 최대 같은 **말이 안 되는 범위**는 DB에 가기 전에 막음 → `IllegalArgumentException` (`api-test.http`에서 `400` 응답 확인).
* 가격 매개변수가 `int`가 아니라 **`Integer`** → "입력 안 함"을 `null`로 표현하기 위해서. `int`면 안 보낸 값이 `0`이 되어 "0원 이상" 조건과 구분이 안 됨.
* `Page.map(MenuResponse::from)` → 페이지 정보는 유지하고 내용만 DTO로 (11/04 내용). `from`은 `record` 안에 둔 **정적 팩토리 메소드**.

### 6) 컨트롤러: 선택 파라미터

```java
@GetMapping("/menus")
public PageResponse<MenuService.MenuResponse> search(
        @RequestParam(required = false) String keyword,
        @RequestParam(required = false) Integer minPrice,
        @RequestParam(required = false) Integer maxPrice,
        @RequestParam(required = false) Integer categoryCode,
        @PageableDefault(size = 5, sort = "menuCode") Pageable pageable) {
    return PageResponse.from(menus.search(keyword, minPrice, maxPrice, categoryCode, pageable));
}
```

```http
### 이름만
GET {{base}}/menus?keyword=마늘&page=0&size=5&sort=menuCode,asc

### 이름 + 최소 가격
GET {{base}}/menus?keyword=마늘&minPrice=2000&page=0&size=5

### 모든 조건
GET {{base}}/menus?keyword=마늘&minPrice=2000&maxPrice=30000&categoryCode={{categoryCode}}&page=0&size=5

### 잘못된 가격 범위: 400
GET {{base}}/menus?minPrice=5000&maxPrice=1000
```

* `@RequestParam(required = false)` → 그 파라미터가 **없어도 400이 아니라 `null`** 로 들어옴. 기본값은 `required = true`라서 이걸 빼면 검색칸을 비울 수 없음.
* URL 하나(`/menus`)로 조건 조합 16가지를 모두 처리함.
* `{{base}}` → `.http` 파일의 **변수**. 파일 위쪽에 `@base = http://localhost:8080/api/v1/practice`처럼 정의해 두고 재사용.

### 테스트

```java
class DynamicSearchTests extends FixtureSupport {
    @Test
    void dynamicConditionsAndPaging() {
        // 픽스처: prefix+"마늘빵"(2500), prefix+"마늘크림빵"(4500), prefix+"아메리카노"(3000)
        var page = menus.search(prefix + "마늘", 2000, 3000, category.getCategoryCode(),
                PageRequest.of(0, 1, Sort.by("menuCode")));
        assertEquals(1, page.getTotalElements());
        assertEquals(prefix + "마늘빵", page.getContent().get(0).menuName());

        assertEquals(3, menus.search(prefix, null, null, null,
                PageRequest.of(0, 2)).getTotalElements());
        assertEquals(2, menus.search(prefix, null, null, null,
                PageRequest.of(0, 2)).getContent().size());

        assertThrows(IllegalArgumentException.class, () -> menus.search(
                null, 5000, 1000, null, PageRequest.of(0, 5)));
    }
}
```

| 검색 | 걸리는 메뉴 | 확인 |
| :--- | :--- | :--- |
| 이름 `prefix+"마늘"` + 2000~3000원 + 카테고리 | 마늘빵(2500)만 (마늘크림빵 4500은 가격 초과) | 조건 **4개 모두** AND로 적용 |
| 이름 `prefix`만, 페이지 크기 2 | 전체 3개 중 2개 | `totalElements`=3, `content.size()`=2 → **페이징**도 같이 적용 |
| 최소 5000 > 최대 1000 | - | 서비스에서 **예외** |

* 검색어에 테스트마다 다른 `prefix`(UUID)를 붙여서, DB에 원래 있던 "마늘" 메뉴와 섞이지 않고 **내가 만든 3개만** 대상으로 확인함 (11/05 `FixtureSupport`).

> ⚠️ 파일 이름은 `DynamicsSearchTests.java`인데 안의 클래스 이름은 `DynamicSearchTests`임. `public`이 아닌 클래스라 컴파일은 되지만, 파일과 클래스 이름이 다르면 IDE에서 찾기 헷갈림 → 제공 코드처럼 `DynamicSearchTests.java`로 맞추는 게 좋음.

> ⚠️ `MenuResponse.from`에서 `menu.getCategory().getCategoryName()`을 꺼냄 → `category`가 `LAZY`라서 11/04에 본 **N+1**이 여기서도 생김. Specification에서도 `root.fetch("category")`로 fetch join을 걸 수 있는데, 페이징 때 실행되는 **개수 쿼리에는 fetch를 걸면 안 되므로** `if (query.getResultType() != Long.class) root.fetch("category");`처럼 구분해서 넣음.

---

## 📝 오늘의 정리

1. `@EnableJpaAuditing`으로 Auditing을 켜고, `@MappedSuperclass` + `@EntityListeners(AuditingEntityListener.class)` 부모 클래스에 `@CreatedDate`·`@LastModifiedDate`를 두면 상속받은 엔티티의 생성·수정 시간이 자동 기록됨
2. 날짜는 `INSERT`/`UPDATE` 직전에 채워지므로, 수정 직후 응답에 새 `updatedAt`을 담으려면 `flush()`로 `UPDATE`를 먼저 보냄
3. `DATETIME(6)`은 마이크로초까지만 저장하므로, 시간 비교 테스트는 DB에서 다시 읽은 값끼리 비교함
4. `JpaSpecificationExecutor`를 상속하면 `findAll(Specification, Pageable)`로 조건 객체를 넘겨 조회할 수 있음
5. Specification 람다에서 값이 있는 조건만 `Predicate` 목록에 담고 `cb.and(...)`로 묶으면, 조건 조합마다 메소드를 만들지 않고 동적 검색이 됨. 조건이 없으면 전체 조회
6. LIKE 검색어의 `%`·`_`는 이스케이프해서 글자 그대로 검색되게 하고, 선택 파라미터는 `Integer` + `@RequestParam(required = false)`로 받아 `null`로 "입력 안 함"을 표현함

---

## 느낀 점

* 생성일·수정일을 서비스마다 직접 넣던 걸 상속 한 줄로 끝낼 수 있어서, 공통 컬럼은 부모 클래스로 빼야겠다는 생각이 들었음
* `rename`의 `flush()`가 왜 있는지 처음엔 몰랐는데, 날짜가 `UPDATE` 직전에 채워진다는 걸 알고 나니 없으면 응답에 옛날 시간이 나간다는 게 이해됨
* 검색 조건 4개를 쿼리 메소드로 만들면 16개가 필요하다는 계산을 해 보고 Specification이 왜 필요한지 확실히 느낌
* 검색창에 `%` 하나만 넣어도 전체가 조회될 수 있다는 건 생각도 못 했는데, 사용자 입력은 SQL 문법 기호까지 의심해야 한다는 걸 배움
