# Spring Data JPA CRUD · 쿼리 메소드 · JPQL · 페이징 · fetch join 정리 (TIL)

> 실습 환경: MySQL (`menudb`의 `tbl_menu`·`tbl_category`), Spring Boot 4 + `spring-boot-starter-data-jpa`·`webmvc`·`validation` (Gradle 프로젝트 `chap04-spring-data-jpa-crud`, `chap05-spring-data-jpa-query`), `.http` 파일로 API 호출, JUnit 5 + `@SpringBootTest`
>
> 11/02의 영속성 컨텍스트·변경 감지, 11/03의 `@Transactional`·`@ManyToOne`·`LAZY`, 10/30의 `ResponseEntity`·`@Valid`·`orElseThrow` 404 처리는 다시 설명하지 않음

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| chap04 | `EntityManager`를 직접 쓰던 Repository → **`JpaRepository` 인터페이스**만 선언. 메뉴 CRUD를 REST API(`/api/v1/menus`)로 완성 |
| chap05 ① | **쿼리 메소드**: 메소드 이름만으로 조회 쿼리 만들기 (`findByMenuPriceGreaterThanEqual...`) |
| chap05 ② | **JPQL** + `@Query`: 엔티티 기준 쿼리 직접 쓰기 (`LIKE` 검색) |
| chap05 ③ | **페이징**(`Pageable`·`Page`) + **fetch join**(`JOIN FETCH`)으로 N+1 줄이기 |

---

## 1. Spring Data JPA: Repository를 인터페이스로

* 어제까지는 Repository 클래스에 `@PersistenceContext EntityManager`를 두고 `persist`·`find`를 **직접** 불렀음. 오늘은 인터페이스만 선언하면 **구현 클래스를 Spring이 대신 만들어** 줌.

```java
// JpaRepository<관리할 엔티티, 엔티티 식별자의 타입>
public interface MenuRepository extends JpaRepository<Menu, Integer> {
}

public interface CategoryRepository extends JpaRepository<Category, Integer> {
}
```

* **한 줄 비유**: 어제는 창고 직원(Repository)을 **직접 교육**해서 썼다면, 오늘은 "메뉴 창고 담당 하나 주세요"라고 **신청서(인터페이스)** 만 내면 Spring이 기본 업무를 다 아는 직원을 **배치해 주는** 것.
* `<Menu, Integer>` → `Menu` 엔티티를 다루고, PK(`menuCode`) 타입은 `Integer`. PK가 `int`여도 제네릭에는 기본형을 못 쓰니 **래퍼 타입**(`Integer`)을 적음.
* `@Repository`를 안 붙여도 됨 → `JpaRepository`를 상속한 인터페이스는 Spring Data가 찾아서 Bean으로 등록함.

### 공짜로 생기는 메소드

| 메소드 | 하는 일 | 어제 방식으로 치면 |
| :--- | :--- | :--- |
| `save(entity)` | 새 엔티티면 저장, 아니면 병합 | `persist` / `merge` |
| `findById(id)` | PK로 1건 조회 → **`Optional<Menu>`** | `find` (없으면 `null`) |
| `findAll()` | 전체 조회 | JPQL `SELECT m FROM Menu m` |
| `delete(entity)` | 삭제 | `remove` |
| `count()`, `existsById(id)` | 개수, 존재 여부 | - |

* `findById`는 `null` 대신 **`Optional`** 을 돌려줌 → `orElseThrow`로 바로 404 예외를 던질 수 있음 (10/30 내용).
* `save()`가 "새 것"인지 판단하는 기준 → PK 값. `int menuCode`가 **`0`이면 새 엔티티**로 보고 `persist`, 값이 있으면 `merge`.

---

## 2. chap04: Spring Data JPA로 메뉴 CRUD API

### Menu 엔티티: setter 대신 update 메소드

```java
@Entity
@Table(name = "tbl_menu")
public class Menu {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "menu_code")
    private int menuCode;
    ...
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_code", nullable = false)
    private Category category;
    ...
    protected Menu() {}

    public Menu(String menuName, int menuPrice, Category category, String orderableStatus) { ... }

    public void update(String menuName, int menuPrice, Category category, String orderableStatus) {
        this.menuName = menuName;
        this.menuPrice = menuPrice;
        this.category = category;
        this.orderableStatus = orderableStatus;
    }
    // getter만
}
```

* setter를 하나씩 열어 두지 않고, **"메뉴 정보를 수정한다"는 의미의 메소드 하나**만 둠 → 어디서 값을 바꾸는지 찾기 쉽고, 필드를 제멋대로 하나만 바꾸는 일을 막음.
* `Category`도 getter만 있고 생성자는 `protected` → 이 실습에서 카테고리는 **조회만** 함.

### MenuService

```java
@Service
public class MenuService {

    private final MenuRepository menuRepository;
    private final CategoryRepository categoryRepository;
    ...
    private Menu findMenuEntity(int menuCode) {
        return menuRepository.findById(menuCode)
                .orElseThrow(() -> new ResourceNotFoundException("메뉴를 찾을 수 없습니다."));
    }

    private Category findCategory(int categoryCode) {
        return categoryRepository.findById(categoryCode)
                .orElseThrow(() -> new ResourceNotFoundException("카테고리를 찾을 수 없습니다."));
    }

    private MenuResponse toResponse(Menu menu) {
        Category category = menu.getCategory();
        return new MenuResponse(
                menu.getMenuCode(), menu.getMenuName(), menu.getMenuPrice(),
                category.getCategoryCode(), category.getCategoryName(),
                menu.getOrderableStatus()
        );
    }

    @Transactional(readOnly = true)
    public MenuResponse findMenu(int menuCode) {
        return toResponse(findMenuEntity(menuCode));
    }

    @Transactional(readOnly = true)
    public List<MenuResponse> findMenus() {
        return menuRepository.findAll().stream()
                .map(this::toResponse)
                .toList();
    }

    @Transactional
    public MenuResponse createMenu(MenuRequest request) {
        Category category = findCategory(request.getCategoryCode());
        Menu menu = new Menu(request.getMenuName(), request.getMenuPrice(),
                category, request.getOrderableStatus());

        return toResponse(menuRepository.save(menu));
    }

    @Transactional
    public MenuResponse updateMenu(int menuCode, MenuRequest request) {
        Menu menu = findMenuEntity(menuCode);
        Category category = findCategory(request.getCategoryCode());

        menu.update(request.getMenuName(), request.getMenuPrice(),
                category, request.getOrderableStatus());

        return toResponse(menu);
    }

    @Transactional
    public void deleteMenu(int menuCode) {
        Menu menu = findMenuEntity(menuCode);
        menuRepository.delete(menu);
    }
}
```

| 작업 | 흐름 | 포인트 |
| :--- | :--- | :--- |
| 단건 조회 | `findById` → 없으면 404 예외 → DTO 변환 | `Optional` + `orElseThrow` |
| 전체 조회 | `findAll()` → `stream().map(toResponse)` | 엔티티 목록을 DTO 목록으로 |
| 등록 | 카테고리 조회 → `new Menu` → `save` | 요청에는 `categoryCode`(번호)만 오니까 **카테고리 엔티티를 먼저 찾아서** 연결 |
| 수정 | 메뉴 조회 → `menu.update(...)` | **`save()`를 안 부름** → 영속 상태 엔티티라 트랜잭션 끝날 때 **변경 감지**로 `UPDATE` |
| 삭제 | 메뉴 조회 → `delete` | 없는 번호는 404 |

* 수정에서 `save()`가 없는 게 핵심. 11/02에 배운 변경 감지가 Spring Data JPA에서도 그대로 동작함.
* **엔티티는 서비스 밖으로 안 내보냄** → `toResponse`로 DTO로 바꿔서 반환. `menu.getCategory()`는 `LAZY` 프록시라서, 트랜잭션이 살아 있는 **서비스 안에서** 카테고리 이름을 꺼내 둬야 함 (아래 `open-in-view: false`와 연결).

### open-in-view: false

```yaml
spring:
  jpa:
    show-sql: true
    open-in-view: false
    hibernate:
      ddl-auto: none
```

* **OSIV**(Open Session In View) → 영속성 컨텍스트를 **HTTP 요청이 끝날 때까지** 열어 두는 기능. Spring Boot는 기본이 `true`.
  * `true`면 컨트롤러·JSON 변환 중에도 `LAZY` 로딩이 됨 → 편하지만 DB 연결을 응답이 끝날 때까지 붙잡고 있고, 어디서 SQL이 나가는지 알기 어려움.
  * `false`면 영속성 컨텍스트가 **서비스의 `@Transactional`이 끝날 때 닫힘** → 그 뒤 `LAZY` 값을 꺼내면 `LazyInitializationException`.
* 그래서 `false`로 두고 **서비스 안에서 DTO로 다 바꿔서** 내보내는 구조를 씀. 실습의 `toResponse`가 그 역할.

### 요청 DTO 검증

```java
public class MenuRequest {

    @NotBlank(message = "메뉴 이름은 필수입니다.")
    private String menuName;

    @Positive(message = "메뉴 가격은 0보다 커야 합니다.")
    private int menuPrice;

    @Positive(message = "카테고리 코드는 0보다 커야 합니다.")
    private int categoryCode;

    @Pattern(regexp = "Y|N", message = "주문 가능 상태는 Y 또는 N이어야 합니다.")
    @NotBlank
    private String orderableStatus;
    ...
}
```

* `@Positive` → 0보다 큰 수만 허용 (0, 음수 거부). `int`는 값이 없으면 `0`이 되므로, 가격·카테고리를 아예 안 보내도 `@Positive`에 걸림.
* 오늘 `orderableStatus`에 `@NotBlank`를 **추가**함. 이유: `@Pattern`은 값이 **`null`이면 검사하지 않고 통과**시킴 → 필드를 빼고 보내면 `null`이 그대로 들어가 DB의 `NOT NULL`에서 터짐. `@NotBlank`로 "**있어야 하고**" + `@Pattern`으로 "**Y나 N이어야**" 둘 다 검사.

> 💡 Bean Validation 어노테이션은 대부분 `null`을 "검사 대상 아님"으로 보고 통과시킴 (`@Size`, `@Pattern`, `@Email` 등). 필수값이면 `@NotNull`/`@NotBlank`를 **같이** 붙여야 함.

### 컨트롤러

```java
@RestController
@RequestMapping("/api/v1/menus")
public class MenuController {
    ...
    @GetMapping("/{menuCode}")   // 200 + 메뉴
    @GetMapping                  // 200 + 메뉴 목록
    @PostMapping                 // 201 + 등록된 메뉴   (@Valid @RequestBody MenuRequest)
    @PutMapping("/{menuCode}")   // 200 + 수정된 메뉴   (@Valid @RequestBody MenuRequest)
    @DeleteMapping("/{menuCode}")// 204
}
```

* 10/30에 배운 REST URI·`ResponseEntity` 규칙 그대로. 이번엔 데이터가 리스트가 아니라 **진짜 DB**에 저장됨.
* 등록은 `ResponseEntity.status(HttpStatus.CREATED).body(...)` → `201` + **등록된 메뉴(번호 포함)** 를 본문으로 돌려줌. 클라이언트가 새 `menuCode`를 바로 알 수 있음.

### .http 파일로 API 호출

```http
### 메뉴 등록
POST http://localhost:8080/api/v1/menus
Content-Type: application/json

{
  "menuName": "까르보나라",
  "menuPrice": 14000,
  "categoryCode": 4,
  "orderableStatus": "Y"
}
```

* IntelliJ에서 `.http` 파일의 요청 옆 ▶ 버튼으로 바로 보냄. `###`가 요청 하나의 구분선.
* 등록 응답의 `menuCode`를 보고 수정·삭제 URL의 번호를 바꿔서 순서대로 실행.

> ⚠️ chap04의 `api-test.http` 수정 요청은 `"menuPrice": "16000"`처럼 **문자열**로 보냄. Jackson이 숫자 문자열을 `int`로 바꿔 줘서 동작은 하지만, 원래 의도대로라면 `16000`(숫자)이 맞음.

> ⚠️ chap04 컨트롤러는 `@RequestMapping("api/v1/menus")`로 앞의 `/`가 빠져 있음. Spring이 알아서 붙여 줘서 동작은 같지만, chap05처럼 `"/api/v1/menus"`로 맞춰 쓰는 게 깔끔함.

### CRUD 통합 테스트

```java
@SpringBootTest
@Transactional
class MenuServiceTests {

    @Autowired
    private MenuService menuService;

    @PersistenceContext
    private EntityManager entityManager;

    @Test
    void menuCrudTest() {
        MenuResponse created = menuService.createMenu(
                new MenuRequest("돈까스 스파게티", 15000, 4, "Y"));
        assertTrue(created.getMenuCode() > 0);

        MenuResponse found = menuService.findMenu(created.getMenuCode());
        assertEquals("돈까스 스파게티", found.getMenuName());

        MenuResponse updated = menuService.updateMenu(
                created.getMenuCode(),
                new MenuRequest("치즈 돈까스 스파게티", 17000, 4, "Y"));
        entityManager.flush();
        assertEquals(17000, updated.getMenuPrice());

        menuService.deleteMenu(created.getMenuCode());
        entityManager.flush();
        entityManager.clear();

        assertThrows(ResourceNotFoundException.class,
                () -> menuService.findMenu(created.getMenuCode()));
    }
}
```

* 등록 → 조회 → 수정 → 삭제를 **한 테스트에서 이어서** 확인. 테스트의 `@Transactional` 덕분에 서비스 트랜잭션이 전부 하나로 묶이고 끝나면 rollback.
* 테스트가 트랜잭션을 쥐고 있어서 서비스 메소드가 끝나도 commit이 안 됨 → `UPDATE`/`DELETE`가 실제로 나가는지 보려고 중간에 `flush()`를 직접 부름.
* 마지막 `clear()` → 1차 캐시를 비워야 `findById`가 DB까지 가서 "정말 지워졌는지" 확인함.

---

## 3. 쿼리 메소드: 이름으로 쿼리 만들기 (chap05)

* `findById`·`findAll`만으로는 "2만 원 이상 메뉴"를 못 찾음. Spring Data JPA는 **메소드 이름을 규칙대로 지으면** 그 이름을 읽어서 쿼리를 만들어 줌.

```java
// 20,000원 이상의 메뉴를 비싼 순서로 보여 주세요
List<Menu> findByMenuPriceGreaterThanEqualOrderByMenuPriceDesc(int minPrice);
```

* **이름 쪼개 보기**

| 조각 | 뜻 |
| :--- | :--- |
| `findBy` | 조회 시작, 뒤에 조건이 옴 |
| `MenuPrice` | 엔티티의 **필드** `menuPrice` (컬럼명 `menu_price`가 아님) |
| `GreaterThanEqual` | `>=` |
| `OrderBy` `MenuPrice` `Desc` | `menuPrice` 내림차순 정렬 |

* 만들어지는 JPQL (개념상): `SELECT m FROM Menu m WHERE m.menuPrice >= ?1 ORDER BY m.menuPrice DESC`
* **한 줄 비유**: 주문서에 **정해진 양식대로** "가격이 / 이상인 / 가격 내림차순"이라고 쓰면, 주방(Spring Data)이 알아서 요리(SQL)를 만들어 주는 것.
* 필드 이름을 틀리면(`findByPrice...`) **앱이 시작할 때** 바로 에러가 남 → 실행 중에 터지는 것보다 빨리 알 수 있음.

### 자주 쓰는 키워드

| 키워드 | 예시 | JPQL 조건 |
| :--- | :--- | :--- |
| `And` / `Or` | `findByMenuNameAndOrderableStatus` | `... = ? AND ... = ?` |
| `GreaterThan` / `LessThanEqual` | `findByMenuPriceLessThanEqual` | `>` / `<=` |
| `Between` | `findByMenuPriceBetween(a, b)` | `BETWEEN a AND b` |
| `Containing` | `findByMenuNameContaining` | `LIKE '%값%'` |
| `StartingWith` | `findByMenuNameStartingWith` | `LIKE '값%'` |
| `In` | `findByCategoryCategoryCodeIn` | `IN (...)` |
| `OrderBy...Asc/Desc` | `...OrderByMenuCodeAsc` | `ORDER BY` |

* 조건이 2~3개를 넘으면 이름이 너무 길어짐 → 그때는 아래 JPQL로 직접 씀.

```java
@Test
@DisplayName("쿼리 메소드로 최소 가격 이상의 메뉴 조회")
void findMenusByMinimumPriceTest() {
    List<MenuResponse> menus = menuService.findMenusByMinimumPrice(20000);

    assertFalse(menus.isEmpty());
    assertTrue(menus.stream().allMatch(menu -> menu.getMenuPrice() >= 20000));
}
```

* `allMatch(조건)` → 스트림의 **모든** 요소가 조건을 만족하면 `true` (10/19 Stream 내용).

---

## 4. JPQL과 @Query

```java
// JPQL: 테이블과 컬럼이 아니라, 엔티티와 필드를 대상으로 조회하는 JPA 쿼리 언어
@Query("SELECT m FROM Menu m WHERE m.menuName LIKE CONCAT('%', :keyword, '%') ORDER BY m.menuCode")
List<Menu> searchByMenuName(@Param("keyword") String keyword);
```

### SQL vs JPQL

| | SQL | JPQL |
| :--- | :--- | :--- |
| 대상 | **테이블**·컬럼 (`tbl_menu`, `menu_name`) | **엔티티**·필드 (`Menu`, `menuName`) |
| 결과 | 행(row) | 엔티티 객체 |
| `SELECT` 절 | `SELECT *` 또는 컬럼 나열 | `SELECT m` (별칭 = 엔티티 통째로) |
| DB 종류 | DB마다 문법이 조금씩 다름 | Hibernate가 **DB에 맞는 SQL로 번역** |

* `FROM Menu m` → `Menu`는 **엔티티 이름**(기본값은 클래스 이름). 대소문자도 구분함. `FROM tbl_menu`라고 쓰면 에러.
* `m.menuName` → 엔티티의 **필드**. 별칭 `m`을 꼭 붙여야 함.

### 파라미터 바인딩

* `:keyword` → **이름 기반 파라미터**. `@Param("keyword")`가 붙은 메소드 인자 값이 여기 들어감.
* `CONCAT('%', :keyword, '%')` → 검색어 앞뒤에 `%`를 붙여 "**포함**" 검색. `"마늘"` → `LIKE '%마늘%'`.
* 문자열을 `+`로 이어 붙여 쿼리를 만들지 않고 파라미터로 넘기므로, 10/23에 본 `PreparedStatement`의 `?`처럼 **SQL Injection을 막아 줌**.
* `@Query`가 붙으면 메소드 이름은 **규칙과 상관없음** → `searchByMenuName`처럼 아무 이름이나 지어도 됨. (사실 이 조건은 쿼리 메소드 `findByMenuNameContainingOrderByMenuCode`로도 만들 수 있음)

| | 쿼리 메소드 | `@Query` JPQL |
| :--- | :--- | :--- |
| 쓰는 법 | 메소드 이름만 | 쿼리 직접 작성 |
| 좋은 점 | 짧은 조건은 제일 빠름 | 복잡한 조건·JOIN·fetch join 가능 |
| 단점 | 조건이 많으면 이름이 끝없이 길어짐 | 쿼리 문자열 오타 주의 (그래도 시작할 때 문법 검사함) |

---

## 5. 페이징: Pageable과 Page

* **한 줄 비유**: 메뉴 100개를 한 번에 주는 게 아니라 **책처럼 한 쪽에 5개씩** 나눠서 "**몇 쪽** 주세요"라고 요청하는 것.

```java
// Repository
@Query(value = "SELECT m FROM Menu m JOIN FETCH m.category",
       countQuery = "SELECT COUNT(m) FROM Menu m")
Page<Menu> findPageWithCategory(Pageable pageable);
```

```java
// Controller
@GetMapping("/page")
public ResponseEntity<PageResponse<MenuResponse>> findMenuPage(
        @PageableDefault(size = 5, sort = "menuCode") Pageable pageable
) {
    return ResponseEntity.ok(menuService.findMenuPage(pageable));
}
```

```http
GET http://localhost:8080/api/v1/menus/page?page=0&size=5&sort=menuCode,asc
```

### Pageable: "몇 쪽을, 몇 개씩, 어떤 순서로"

| 요청 파라미터 | 뜻 | 기본값 (`@PageableDefault`) |
| :--- | :--- | :--- |
| `page` | 몇 번째 페이지 (**0부터 시작**) | `0` |
| `size` | 한 페이지에 몇 개 | `5` |
| `sort` | `필드,방향` (`menuCode,asc`) | `menuCode` 오름차순 |

* 컨트롤러 매개변수에 `Pageable`을 두면 Spring이 쿼리 스트링 `page`·`size`·`sort`를 읽어서 **자동으로 만들어 줌**. 파라미터가 없으면 `@PageableDefault` 값을 씀.
* 테스트처럼 직접 만들 때는 `PageRequest.of(0, 5, Sort.by("menuCode").ascending())`.
* `sort`에도 **엔티티 필드 이름**(`menuCode`)을 씀.
* 페이지가 **0부터**라는 점 주의 → 화면의 "1페이지"는 `page=0`.

### Page: 내용 + 페이지 정보

* `Page<Menu>`를 반환 타입으로 두면 Spring Data가 쿼리를 **두 번** 실행함.
  1. 내용 쿼리: `... LIMIT 5 OFFSET 0` → 이번 페이지의 메뉴 5개
  2. 개수 쿼리: `SELECT COUNT(...)` → 전체 몇 개인지 (전체 페이지 수 계산용)

| `Page` 메소드 | 뜻 |
| :--- | :--- |
| `getContent()` | 이번 페이지의 데이터 목록 |
| `getNumber()` | 현재 페이지 번호 (0부터) |
| `getSize()` | 페이지 크기 |
| `getTotalElements()` | 전체 데이터 개수 |
| `getTotalPages()` | 전체 페이지 수 |

* `countQuery`를 따로 쓴 이유 → 내용 쿼리에 `JOIN FETCH`가 있으면 Spring Data가 그걸로 개수 쿼리를 **자동으로 만들 수 없어서**(fetch join은 COUNT와 같이 못 씀) 에러가 남. 개수만 세면 되니 JOIN도 필요 없는 가벼운 쿼리를 직접 지정함.

### Page.map()과 PageResponse

```java
@Transactional(readOnly = true)
public PageResponse<MenuResponse> findMenuPage(Pageable pageable) {
    Page<MenuResponse> responsePage = menuRepository
            .findPageWithCategory(pageable)   // Page<Menu>
            .map(this::toResponse);           // Page<MenuResponse>

    return PageResponse.from(responsePage);
}
```

```java
public class PageResponse<T> {

    private final List<T> content;
    private final int page;
    private final int size;
    private final long totalElements;
    private final int totalPages;

    private PageResponse(...) { ... }

    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
                page.getContent(), page.getNumber(), page.getSize(),
                page.getTotalElements(), page.getTotalPages()
        );
    }
    // getter
}
```

* `Page.map()` → 스트림의 `map`처럼 내용물을 바꾸지만, 결과가 다시 **`Page`** 라서 페이지 번호·전체 개수 정보가 **그대로 유지**됨. `Menu` 엔티티만 `MenuResponse`로 바뀜.
* `PageResponse<T>` → `Page`를 그대로 JSON으로 내보내지 않고 **필요한 5개 값만** 담은 응답 DTO.
  * `Page` 구현체(`PageImpl`)를 그대로 내보내면 내부 정보까지 잔뜩 나가고, 버전이 바뀌면 JSON 모양도 바뀔 수 있음 → 응답 모양을 **내가 정해서** 고정.
  * `<T>` 제네릭 → 메뉴든 카테고리든 어떤 목록에도 재사용 가능.
  * 생성자는 `private`, 만드는 건 `from(page)` **정적 팩토리 메소드**로만 → "`Page`에서 변환한다"는 뜻이 이름에 드러남.

```json
{
  "content": [ { "menuCode": 1, "menuName": "...", "categoryName": "..." }, ... ],
  "page": 0,
  "size": 5,
  "totalElements": 21,
  "totalPages": 5
}
```

---

## 6. N+1 문제와 fetch join

### N+1 문제

* chap04의 `findMenus()`는 `findAll()` 후 메뉴마다 `toResponse`에서 `category.getCategoryName()`을 부름. `category`는 `LAZY` 프록시라서…
  1. `SELECT ... FROM tbl_menu` → 메뉴 N개 (쿼리 **1번**)
  2. 메뉴마다 카테고리 이름을 꺼낼 때 `SELECT ... FROM tbl_category WHERE category_code=?` (최대 **N번**)
* 처음 1번 + 연관 객체 N번 → **N+1 문제**. 메뉴가 100개면 쿼리가 최대 101번.
* 같은 카테고리는 1차 캐시에 있으니 다시 조회하지 않음 → 실제로는 "**서로 다른 카테고리 개수**"만큼 추가로 나감. 그래도 데이터가 많아지면 쿼리 수가 같이 늘어나는 건 똑같음.
* **한 줄 비유**: 반 학생 30명 명단을 받은 뒤, 각 학생의 담임 이름을 알려고 **교무실을 30번 왕복**하는 것.

> 💡 `EAGER`로 바꿔도 해결되지 않음. `findAll()`은 JPQL이라 일단 메뉴만 조회한 뒤, `EAGER`를 채우려고 카테고리를 **바로 이어서** N번 조회함 → 시점만 앞당겨질 뿐 쿼리 수는 같음.

### fetch join

```java
@Query(value = "SELECT m FROM Menu m JOIN FETCH m.category",
       countQuery = "SELECT COUNT(m) FROM Menu m")
Page<Menu> findPageWithCategory(Pageable pageable);
```

* `JOIN FETCH m.category` → 메뉴를 조회할 때 연결된 카테고리를 **JOIN해서 한 번에** 가져와 `category` 필드에 **진짜 객체**를 채워 넣음 (프록시 아님).
* 결과: 메뉴 + 카테고리를 **SQL 1번**으로 (페이징이라 개수 쿼리까지 2번). `toResponse`에서 `getCategoryName()`을 불러도 추가 SQL이 없음.
* **한 줄 비유**: 명단을 받을 때 **담임 이름까지 적힌 명단**을 받아 오는 것.

| | 그냥 `JOIN` | `JOIN FETCH` |
| :--- | :--- | :--- |
| 목적 | 조건 걸기 (`WHERE c.categoryName = ...`) | 연관 객체를 **함께 로딩** |
| `m.category` 상태 | 여전히 `LAZY` 프록시 | 실제 객체가 채워짐 |
| 추가 SQL | 꺼낼 때 또 나감 | 안 나감 |

```java
@Test
@DisplayName("fetch join과 페이징으로 메뉴와 카테고리 조회")
void findMenuPageTest() {
    PageRequest pageable = PageRequest.of(0, 5, Sort.by("menuCode").ascending());

    PageResponse<MenuResponse> page = menuService.findMenuPage(pageable);

    assertFalse(page.getContent().isEmpty());
    assertTrue(page.getContent().size() <= 5);
    assertTrue(page.getTotalElements() > 0);
    assertTrue(page.getContent().stream()
            .allMatch(menu -> menu.getCategoryName() != null));
}
```

* 콘솔에서 `tbl_menu`와 `tbl_category`를 JOIN한 `SELECT` 1번 + `COUNT` 1번만 찍히는지 확인.

> ⚠️ 이 실습은 `@ManyToOne`(메뉴 → 카테고리 **1개**)라서 fetch join + 페이징이 안전함. 반대로 `@OneToMany` **컬렉션**(카테고리 → 메뉴 목록)을 fetch join하면서 페이징하면, JOIN 결과 행 수가 늘어나서 DB에서 `LIMIT`을 못 걸고 Hibernate가 **전부 메모리로 가져와서** 자름 (경고 로그가 남음). 컬렉션은 fetch join과 페이징을 같이 쓰지 않음.

> ⚠️ `findMenusByMinimumPrice`, `searchMenusByName`, chap04의 `findMenus`는 fetch join이 없어서 위의 N+1이 그대로 남아 있음. 메뉴 수가 늘어나면 `@Query`에 `JOIN FETCH m.category`를 넣거나, 쿼리 메소드에 `@EntityGraph(attributePaths = "category")`를 붙여 같은 효과를 낼 수 있음.

### URL 겹침?

* `/api/v1/menus/page`와 `/api/v1/menus/{menuCode}`는 둘 다 경로 한 칸짜리라 겹쳐 보이지만, Spring은 **고정 문자열(`page`)을 변수(`{menuCode}`)보다 먼저** 매칭함 → `/page`는 페이징 API로 감.
* `/search/price`, `/search/name`은 두 칸짜리라 `{menuCode}`와 애초에 안 겹침.

---

## 📝 오늘의 정리

1. `JpaRepository<엔티티, PK타입>`을 상속한 인터페이스만 만들면 `save`·`findById`·`findAll`·`delete` 같은 기본 메소드가 생기고, `findById`는 `Optional`을 돌려줘서 `orElseThrow`로 404 처리를 함
2. 수정은 `save()` 없이 영속 엔티티의 `update` 메소드만 호출하면 변경 감지로 반영되고, 엔티티는 서비스 안에서 DTO로 바꿔서 내보냄. `open-in-view: false`면 트랜잭션 밖에서는 `LAZY` 로딩이 안 됨
3. `@Pattern` 등은 `null`을 통과시키므로 필수값에는 `@NotBlank`/`@NotNull`을 같이 붙임
4. 쿼리 메소드는 `findBy` + 필드명 + 키워드(`GreaterThanEqual`·`OrderBy...Desc` 등) 이름 규칙으로 쿼리를 자동 생성함
5. JPQL은 테이블이 아니라 엔티티·필드 기준으로 쓰는 쿼리이고, `@Query` + `:이름` + `@Param`으로 파라미터를 바인딩함
6. `Pageable`(page는 0부터, size, sort)로 요청하고 `Page`로 내용과 전체 개수를 받으며, `Page.map()`으로 페이지 정보를 유지한 채 DTO로 바꾸고 `PageResponse`로 응답 모양을 고정함
7. `LAZY` 연관 객체를 목록마다 꺼내면 N+1 문제가 생기고, `JOIN FETCH`로 연관 객체를 한 번에 가져와 해결함. fetch join + 페이징에는 `countQuery`를 따로 지정함

---

## 느낀 점

* 어제까지 직접 `persist`·`find`를 부르던 Repository가 인터페이스 한 줄로 끝나는 걸 보니 허무할 정도였지만, 그 안에서 결국 `EntityManager`가 같은 일을 한다는 걸 알고 있으니 덜 마법처럼 느껴졌음
* 수정 메소드에 `save()`가 없는 게 처음엔 빠뜨린 줄 알았는데, 11/02에 배운 변경 감지 덕분이라는 걸 바로 연결할 수 있었음
* 쿼리 메소드 이름이 문장처럼 읽히는 게 신기했지만, 조건이 늘면 이름이 너무 길어져서 JPQL과 적당히 나눠 써야겠다고 느낌
* `LAZY` 하나 때문에 목록 조회에서 쿼리가 메뉴 수만큼 늘어날 수 있다는 N+1 문제가 가장 기억에 남음. 콘솔의 SQL 개수를 항상 확인하는 습관을 들여야겠음
* `@Pattern`만으로는 값이 없을 때 막지 못한다는 걸 알고, 검증 어노테이션이 `null`을 어떻게 다루는지 따로 챙겨 봐야겠다고 생각함
