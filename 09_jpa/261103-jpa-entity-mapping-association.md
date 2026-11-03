# Spring Boot JPA 전환 · 엔티티 매핑(@Column·@Enumerated·@Embedded) · 연관관계 매핑(N:1·1:N·양방향) 정리 (TIL)

> 실습 환경: MySQL (`menudb`), Spring Boot 4 + `spring-boot-starter-data-jpa` (Gradle 프로젝트 `chap02-mapping`, `chap03-association-mapping`), JUnit 5 + `@SpringBootTest`
>
> 11/02에 정리한 `@Entity`·`@Id`·`@GeneratedValue` 기본, `EntityManager` CRUD, 영속성 컨텍스트·생명주기는 여기서 다시 설명하지 않음

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| 전환 | 순수 JPA(`persistence.xml`, 직접 만든 `EntityManager`) → **Spring Boot**가 대신 설정·주입·트랜잭션 처리 |
| chap02 section01 | `@Column` 속성(`nullable`·`unique`·`length`), `@Enumerated`, `ddl-auto`로 테이블 자동 생성 |
| chap02 section02 | `@Embeddable`·`@Embedded`로 **값 객체**(`Price`)를 엔티티 컬럼에 포함 |
| chap03 section01 | `@ManyToOne` + `@JoinColumn`: 메뉴 → 카테고리 (N:1), 지연 로딩(`LAZY`) |
| chap03 section02 | `@OneToMany` 단방향: 카테고리 → 메뉴 목록 (1:N) |
| chap03 section03 | 양방향 연관관계, `mappedBy`, **연관관계의 주인**, 연관관계 편의 메소드 |

---

## 1. 순수 JPA → Spring Boot JPA

* 어제는 `persistence.xml`, `EntityManagerFactoryGenerator`, `transaction.begin()`을 **전부 직접** 만들었음. 오늘부터는 Spring Boot가 그 일을 대신 함.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
runtimeOnly 'com.mysql:mysql-connector-j'
testImplementation 'org.springframework.boot:spring-boot-starter-data-jpa-test'
```

| 하는 일 | 어제 (순수 JPA) | 오늘 (Spring Boot) |
| :--- | :--- | :--- |
| 설정 파일 | `META-INF/persistence.xml` | `application.yml` |
| 엔티티 등록 | `<class>`로 하나씩 | `@SpringBootApplication` 패키지 아래를 **자동 스캔** |
| `EntityManagerFactory` | 싱글톤 클래스를 직접 만듦 | Spring이 만들어서 Bean으로 관리 |
| `EntityManager` 얻기 | `EntityManagerGenerator.getInstance()` | `@PersistenceContext`로 **주입** |
| 트랜잭션 | `getTransaction().begin()` / `commit()` / `rollback()` | 메소드에 `@Transactional` |

### application.yml

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/menudb
    username: 사용자명
    password: 비밀번호
  jpa:
    show-sql: true
    database: mysql
    hibernate:
      ddl-auto: create   # chap02는 create, chap03는 none
    properties:
      hibernate:
        format_sql: true
```

* `show-sql`·`format_sql` → 어제 `persistence.xml`의 `hibernate.show_sql`·`format_sql`과 같은 역할.
* `ddl-auto` → 엔티티를 보고 **테이블을 자동으로 만들지** 정하는 설정 (2장에서 자세히).
* chap03에는 `naming.physical-strategy: CamelCaseToUnderscoresNamingStrategy`도 있음 → `@Column(name=...)`을 안 붙인 필드는 `menuName` → `menu_name`처럼 **카멜 표기를 언더스코어로** 바꿔서 컬럼 이름으로 씀. (Spring Boot의 기본값이기도 하고, chap03 엔티티는 모든 필드에 `name`을 적어서 차이가 드러나진 않음)

### 계층 구조: DTO → Service → Repository

```java
@Repository
public class MemberRepository {

    @PersistenceContext
    private EntityManager entityManager;

    public Member save(Member member) {
        entityManager.persist(member);
        return member;
    }
}
```

```java
@Service
public class MemberRegistService {

    private final MemberRepository memberRepository;

    @Autowired
    public MemberRegistService(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }

    @Transactional
    public Member registMember(MemberRegistDTO newMember) {
        Member member = new Member(
                newMember.getMemberId(),
                ...
                newMember.getStatus()
        );
        return memberRepository.save(member);
    }
}
```

* **한 줄 비유**: DTO는 **주문서**, Service는 주문서를 보고 **물건(엔티티)을 조립하는 직원**, Repository는 조립된 물건을 **창고(DB)에 넣는 직원**.
* `@PersistenceContext` → 어제 배운 대로 `EntityManager`는 여러 스레드가 같이 쓰면 안 되는데, Spring은 **트랜잭션마다 알맞은 `EntityManager`로 연결해 주는 대리 객체(프록시)** 를 넣어 줌 → 필드에 하나만 두고 써도 안전함.
* `@Transactional` → 메소드 시작 때 트랜잭션 시작, 정상 종료면 `commit`, 예외면 `rollback`을 Spring이 대신 함. 어제 테스트의 `begin()`·`rollback()` 코드가 사라짐.
* `@Transactional(readOnly = true)` → 조회 전용. "데이터를 안 바꾼다"고 알려 주면 Hibernate가 변경 감지용 스냅샷을 만들지 않아서 조금 더 가벼워짐. chap03의 `findMenu`·`findCategory`에 붙어 있음.
* `BookRegistService`에는 생성자에 `@Autowired`가 없음 → 생성자가 **하나뿐이면** 생략해도 주입됨 (10/26 내용).

### 테스트: @SpringBootTest + @Transactional

```java
@SpringBootTest
@Transactional
public class MemberRegistServiceTests {

    @Autowired
    private MemberRegistService memberRegistService;

    @Test
    @Commit
    void testRegistMember() { ... }
}
```

| 어노테이션 | 하는 일 |
| :--- | :--- |
| `@SpringBootTest` | 실제 앱처럼 Spring 컨테이너를 띄워서 Bean을 `@Autowired`로 받아 쓸 수 있게 함 |
| 테스트 클래스의 `@Transactional` | 테스트 메소드 하나하나를 트랜잭션으로 감싸고, **끝나면 자동 rollback** |
| `@Commit` | 이 테스트만 rollback 대신 **commit** → DB에 실제로 남음 |

* 서비스 메소드에도 `@Transactional`이 있지만, 테스트가 이미 트랜잭션을 열어 뒀으므로 **그 트랜잭션에 참여**함 → 테스트가 끝날 때 한꺼번에 rollback됨.
* `testRegistMember`는 `@Commit`이 붙어 있어서 DB에서 `tbl_member`에 회원이 남은 걸 직접 확인할 수 있음.

---

## 2. 엔티티 매핑 상세 (chap02 section01)

### Member 엔티티

```java
@Entity
@Table(name="tbl_member")
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_no")
    private int memberNo;
    @Column(name="member_id", nullable = false, unique = true, length = 50)
    private String memberId;
    @Column(name="member_pwd", nullable = false, length = 100)
    private String memberPwd;
    @Column(name="member_name", nullable = false, length = 50)
    private String memberName;
    @Column(name="phone")
    private String phone;
    @Column(name="address")
    private String address;
    @Column(name="enroll_date", nullable = false)
    private LocalDateTime enrollDate;

    @Enumerated(EnumType.STRING)
    @Column(name="member_role", nullable = false)
    private MemberRole memberRole;
    @Column(name="status", nullable = false, length = 1)
    private String status;

    protected Member() {}
    // 등록용 생성자 + getter만 (setter 없음)
}
```

```java
public enum MemberRole {
    ROLE_MEMBER,
    ROLE_ADMIN
}
```

### @Column 속성

| 속성 | 뜻 | 기본값 | 테이블에 생기는 것 |
| :--- | :--- | :--- | :--- |
| `nullable = false` | 빈 값 금지 | `true` | `NOT NULL` |
| `unique = true` | 중복 금지 | `false` | `UNIQUE` 제약조건 |
| `length = 50` | 문자열 최대 길이 | `255` | `varchar(50)` |

* 이 속성들은 주로 `ddl-auto`로 **테이블을 만들 때** 쓰임. 이미 있는 테이블에 값을 넣을 때 JPA가 미리 검사해 주는 게 아니라, 제약조건 위반은 **DB가** 막음.
* `phone`, `address`처럼 아무 속성이 없으면 → `varchar(255)`, `NULL` 허용.
* `length`는 **문자열(`String`)에만** 의미가 있음.
* `LocalDateTime` → 날짜+시간 컬럼(`datetime`)으로 바로 매핑됨. 별도 변환 코드가 필요 없음.

### @Enumerated: enum을 어떻게 저장할까

| 방식 | 저장되는 값 | 문제 |
| :--- | :--- | :--- |
| `EnumType.ORDINAL` (기본값) | 선언 **순서 번호** (`ROLE_MEMBER` → `0`, `ROLE_ADMIN` → `1`) | enum 중간에 값을 추가하거나 순서를 바꾸면 **기존 데이터의 의미가 바뀜** |
| `EnumType.STRING` | **이름 그대로** (`"ROLE_MEMBER"`) | 공간을 조금 더 씀 (그 대신 안전) |

* **한 줄 비유**: ORDINAL은 "**앞에서 몇 번째 사람**"으로 기억하는 것 → 줄 순서가 바뀌면 엉뚱한 사람이 됨. STRING은 "**이름**"으로 기억하는 것.
* 예: `ROLE_MEMBER` 앞에 `ROLE_GUEST`를 추가하면, ORDINAL로 저장된 기존 `0`(원래 회원)이 **손님**으로 읽힘.
* 그래서 실습처럼 **항상 `EnumType.STRING`** 을 씀.

### ddl-auto: 테이블 자동 생성

| 값 | 하는 일 |
| :--- | :--- |
| `create` | 시작할 때 엔티티 테이블을 **지우고(DROP) 새로 만듦** |
| `create-drop` | `create` + 앱이 **끝날 때** 다시 지움 |
| `update` | 엔티티와 테이블을 비교해서 **추가된 것만** 반영 (삭제는 안 함) |
| `validate` | 엔티티와 테이블이 맞는지 **검사만**, 다르면 실행 실패 |
| `none` | 아무것도 안 함 |

* chap02는 `create` → `tbl_member`, `tbl_book`이 없어도 엔티티를 보고 Hibernate가 `CREATE TABLE`을 실행함 (콘솔에 DDL이 찍힘).
* chap03는 `none` → `menudb`에 **이미 있는** `tbl_menu`·`tbl_category`를 그대로 사용.

> ⚠️ `create`는 실행할 때마다 테이블을 **DROP**함. `@Commit`으로 남긴 회원도 다음 실행 때 사라짐. 실제 서비스나 같이 쓰는 DB에서는 절대 `create`/`create-drop`/`update`를 쓰면 안 되고, `validate`나 `none`을 씀.

### 저장 테스트

```java
MemberRegistDTO newMember = new MemberRegistDTO(
        "user01", "pass01", "홍길동", "010-1234-5678", "서울시 종로구",
        LocalDateTime.now(), MemberRole.ROLE_MEMBER, "Y"
);

Member savedMember = memberRegistService.registMember(newMember);

assertTrue(savedMember.getMemberNo() > 0);
assertEquals("홍길동", savedMember.getMemberName());
assertEquals(MemberRole.ROLE_MEMBER, savedMember.getMemberRole());
```

* `memberNo > 0` → `IDENTITY` 전략이라 `persist` 순간 `INSERT`가 나가고 DB가 만든 번호가 채워졌는지 확인 (어제 정리한 내용).
* `Member`에는 **setter가 없음** → 만들 때 생성자로만 값을 넣고, 아무 데서나 값을 바꾸지 못하게 막음.

---

## 3. 임베디드 타입: @Embeddable · @Embedded (chap02 section02)

* **한 줄 비유**: 가격 관련 값 3개(정가·할인율·판매가)를 **"가격"이라는 한 봉투**에 묶어서 다루는 것. 테이블을 따로 만드는 게 아니라, 봉투 안 내용물이 **책 테이블의 컬럼으로 펼쳐져** 저장됨.

```java
@Embeddable
public class Price {

    @Column(name="regular_price", nullable = false)
    private int regularPrice;

    @Column(name="dicount_rate", nullable = false)
    private double discountRate;

    @Column(name="sell_price", nullable = false)
    private int sellPrice;

    protected Price() {}

    public Price(int regularPrice, double discountRate) {
        validatePrice(regularPrice);
        validateDiscountRate(discountRate);
        this.regularPrice = regularPrice;
        this.discountRate = discountRate;
        this.sellPrice = calculateSellPrice(regularPrice, discountRate);
    }

    private void validatePrice(int regularPrice) {
        if (regularPrice < 0) {
            throw new IllegalArgumentException("가격은 음수일 수 없습니다.");
        }
    }

    private void validateDiscountRate(double discountRate) {
        if (discountRate < 0 || discountRate > 1) {
            throw new IllegalArgumentException("할인율은 0 이상 1 이하여야 합니다.");
        }
    }

    private int calculateSellPrice(int regularPrice, double discountRate) {
        return (int) (regularPrice - (regularPrice * discountRate));
    }
    // getter만
}
```

```java
@Entity
@Table(name="tbl_book")
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name="book_no")
    private int bookNo;
    @Column(name="book_title")
    private String bookTitle;
    @Column(name="author")
    private String author;
    @Column(name="publisher")
    private String publisher;
    @Column(name="published_date")
    private LocalDate publishedDate;

    @Embedded
    private Price price;
    ...
}
```

| 어노테이션 | 붙이는 곳 | 뜻 |
| :--- | :--- | :--- |
| `@Embeddable` | 값 객체 클래스 (`Price`) | "나는 다른 엔티티 안에 **끼워 넣어지는** 부품" |
| `@Embedded` | 엔티티의 필드 (`Book.price`) | "이 필드는 끼워 넣은 부품이니 **안의 필드를 컬럼으로 펼쳐라**" |

* 만들어지는 `tbl_book` 컬럼: `book_no`, `book_title`, `author`, `publisher`, `published_date`, **`regular_price`, `dicount_rate`, `sell_price`** → 테이블은 하나, 자바 쪽에서만 묶여 있음.
* `Price`는 `@Id`가 없음 → 혼자서는 저장·조회할 수 없고, 항상 `Book`에 **딸려서** 저장됨.
* `@Embeddable`도 JPA가 객체를 만들어야 하므로 `protected` 기본 생성자가 필요함.

### 값 객체로 묶으면 좋은 점

* **검증을 한곳에서**: 음수 가격, 1을 넘는 할인율은 `new Price(...)` 단계에서 바로 예외 → 잘못된 `Price`는 **애초에 만들어지지 않음**.
* **계산 규칙도 한곳에서**: 판매가는 생성자 안에서 정가·할인율로 계산 → 서비스마다 계산식을 따로 쓰지 않아도 됨.
* **setter가 없음** → 만들어진 뒤 값이 바뀌지 않음(불변). 가격을 바꾸려면 `new Price(...)`로 **새로** 만들어 끼움.
* `Book` 클래스가 가격 계산 코드로 지저분해지지 않음.

### 테스트

```java
@Test
@DisplayName("임베디드 가격을 포함한 도서 저장 확인")
void testRegistBookWithEmbeddedPrice() {
    BookRegistDTO newBook = new BookRegistDTO(
            "자바 ORM 표준 JPA 프로그래밍", "김영한", "에이콘",
            LocalDate.now(), 43000, 0.1
    );

    Book savedBook = bookRegistService.registBook(newBook);

    assertTrue(savedBook.getBookNo() > 0);
    assertEquals(38700, savedBook.getPrice().getSellPrice());
}

@Test
@DisplayName("할인율은 0 이상 1 이하만 허용")
void testInvalidDiscountRate() {
    assertThrows(
            IllegalArgumentException.class,
            () -> new Price(43000, 1.1)
    );
}
```

* `43000 - (43000 × 0.1) = 38700` → 판매가가 생성자에서 자동 계산됐는지 확인.
* `assertThrows(예외 클래스, 실행할 코드)` → 람다 안의 코드를 실행했을 때 **그 예외가 터지면 통과**. 할인율 `1.1`은 1을 넘으므로 `IllegalArgumentException`.
* `BookRegistService`는 DTO의 `regularPrice`, `discountRate`를 꺼내 `new Price(...)`로 조립해서 `Book`에 넣음 → DTO는 평평한 값, 엔티티는 묶인 값.

> ⚠️ `Price`의 `@Column(name="dicount_rate")`는 `discount`의 **오타**. `ddl-auto: create`라서 오타 그대로 `dicount_rate` 컬럼이 만들어짐. 직접 SQL로 조회할 때 헷갈리니 `discount_rate`로 고치는 게 좋음.

> ⚠️ 실습 원본 `Book`의 `bookNo`에는 `@Column(name="book_no", nullable = false, length = 100)`처럼 `length`가 붙어 있는데, `int` 필드라 `length`는 **아무 효과가 없음**(문자열 전용). 또 쓰지 않는 `import org.springframework.stereotype.Component;`도 있음 → 둘 다 지워도 됨.

---

## 4. 연관관계 매핑이 필요한 이유

* 어제 chap00 문제 3(패러다임 불일치)에서 본 것처럼, 테이블은 **FK 번호**(`category_code`)로 연결하고 객체는 **참조**(`Category category`)로 연결함.
* 오늘 이걸 JPA가 대신 연결해 줌 → JOIN을 직접 쓰거나 `new Category(...)`로 조립하지 않아도 `menu.getCategory().getCategoryName()`이 됨.

| | 테이블 | 객체 |
| :--- | :--- | :--- |
| 연결 수단 | FK 컬럼 (`tbl_menu.category_code`) | 참조 필드 (`Menu.category`, `Category.menuList`) |
| 방향 | FK 하나로 **양쪽 다** JOIN 가능 | 필드가 있는 쪽에서만 이동 가능 (**방향이 있음**) |

* 그래서 객체에서는 "누가 누구를 들고 있게 할지(방향)"와 "몇 대 몇인지(다중성)"를 정해 줘야 함.

```text
tbl_category (1) ───< tbl_menu (N)
  category_code          menu_code
  category_name          menu_name
                         category_code (FK)
```

* 카테고리 하나에 메뉴 여러 개 → 메뉴 입장에서는 **N:1**(`@ManyToOne`), 카테고리 입장에서는 **1:N**(`@OneToMany`).

---

## 5. N:1 단방향: @ManyToOne (chap03 section01)

```java
@Entity(name = "Section01Menu")
@Table(name = "tbl_menu")
public class Menu {

    @Id
    @Column(name = "menu_code")
    private int menuCode;
    ...
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name="category_code", nullable = false)
    private Category category;
    ...
}
```

* `private int categoryCode` 대신 **`private Category category`** → 번호가 아니라 카테고리 객체를 들고 있음.
* `@ManyToOne` → "메뉴(Many) 여러 개가 카테고리(One) 하나를 가리킴".
* `@JoinColumn(name="category_code")` → 이 연결에 쓰는 **FK 컬럼 이름**. JPA는 `category.getCategoryCode()` 값을 이 컬럼에 넣고, 조회할 때는 이 컬럼으로 카테고리를 찾음.
* `Category` 쪽에는 `Menu`에 대한 필드가 **없음** → 메뉴 → 카테고리 한 방향만 가능 (**단방향**).

### 즉시 로딩(EAGER) vs 지연 로딩(LAZY)

* **한 줄 비유**: EAGER는 메뉴판을 받을 때 **카테고리 설명서까지 같이** 받아 오는 것. LAZY는 일단 **"카테고리 4번" 쪽지만** 받아 두고, 설명서가 필요해지는 **그 순간** 가지러 가는 것.

| | EAGER (즉시) | LAZY (지연) |
| :--- | :--- | :--- |
| `find(Menu)` 시점 | 메뉴 + 카테고리를 **JOIN으로 같이** 조회 | **메뉴만** 조회, `category`에는 **가짜 객체(프록시)** |
| 카테고리 조회 시점 | 이미 가져와 있음 | `getCategoryName()`처럼 **실제 값을 쓸 때** `SELECT` |
| 기본값 | `@ManyToOne`, `@OneToOne` | `@OneToMany`, `@ManyToMany` |

* 실무에서는 필요 없는 JOIN이 계속 붙는 걸 막으려고 **전부 `LAZY`로 두는 것**을 기본으로 함. 실습에서도 `@ManyToOne`의 기본값(EAGER)을 일부러 `LAZY`로 바꿨음.
* 프록시는 `categoryCode`(PK)는 이미 알고 있음 → `getCategoryCode()`만 부르면 SQL이 안 나가고, 그 외 값을 꺼낼 때 SQL이 나감.

### 조회 테스트

```java
@Test
@DisplayName("N:1 연관관계로 메뉴에서 카테고리 조회")
void manyToOneFindTest() {
    int menuCode = 10;

    Menu foundMenu = manyToOneService.findMenu(menuCode);
    Category category = foundMenu.getCategory();

    assertNotNull(foundMenu);
    assertNotNull(category.getCategoryName());
}
```

* **따라가기**
  1. `findMenu(10)` → `SELECT ... FROM tbl_menu WHERE menu_code=10` (카테고리는 조회 안 함)
  2. `getCategory()` → 프록시를 돌려줌 (아직 SQL 없음)
  3. `getCategoryName()` → 이제 진짜 값이 필요 → `SELECT ... FROM tbl_category WHERE category_code=?`
* 콘솔에서 `SELECT`가 **두 번, 따로** 찍히는 걸로 LAZY를 확인할 수 있음.

> ⚠️ 3단계가 되는 건 테스트 클래스에 `@Transactional`이 있어서 영속성 컨텍스트가 **테스트 끝까지 열려 있기** 때문. 서비스 메소드의 트랜잭션이 끝난 **바깥**(예: 트랜잭션 없는 컨트롤러)에서 프록시의 값을 꺼내면 `LazyInitializationException`이 남. 그래서 보통 서비스 안에서 **DTO로 바꿔서** 내보냄.

> ⚠️ 이 테스트는 `foundMenu.getCategory()`를 먼저 부르고 나서 `assertNotNull(foundMenu)`를 검사함 → `foundMenu`가 `null`이면 검사 전에 `NullPointerException`이 남. 제공된 완성 코드처럼 `assertNotNull(category)`로 검사하거나, `foundMenu` 검사를 `getCategory()`보다 **위로** 올리는 게 맞음.

### 등록 테스트: 기존 카테고리와 연결해서 저장

```java
@Transactional
public Menu registMenu(MenuDTO menuInfo) {
    Category category = manyToOneRepository.findCategory(menuInfo.getCategoryCode());

    if (category == null) {
        throw new IllegalArgumentException("존재하지 않는 카테고리입니다");
    }

    Menu menu = new Menu(
            menuInfo.getMenuCode(),
            menuInfo.getMenuName(),
            menuInfo.getMenuPrice(),
            category,
            menuInfo.getOrderableStatus()
    );

    return manyToOneRepository.save(menu);
}
```

```java
MenuDTO menuInfo = new MenuDTO(9001, "돈까스 스파게티", 15000, 4, "Y");

Menu savedMenu = manyToOneService.registMenu(menuInfo);
entityManager.flush();

assertEquals(9001, savedMenu.getMenuCode());
assertEquals(4, savedMenu.getCategory().getCategoryCode());
```

* DTO에는 `categoryCode`(번호)만 있음 → 서비스가 번호로 **카테고리 엔티티를 먼저 찾아서** `Menu`에 넣음. 없는 번호면 예외.
* `INSERT` 때 JPA가 `category.getCategoryCode()`(= 4)를 꺼내서 `category_code` 컬럼에 넣어 줌.
* **따라가기**
  1. `findCategory(4)` → `SELECT ... FROM tbl_category`
  2. `persist(menu)` → 이 `Menu`는 `@GeneratedValue` 없이 `menuCode`(9001)를 **직접** 넣음 → 번호를 DB에 물어볼 필요가 없으니 `INSERT`를 **바로 보내지 않고** 모아 둠 (쓰기 지연)
  3. `entityManager.flush()` → 이때 `INSERT INTO tbl_menu (..., category_code, ...) VALUES (9001, ..., 4, ...)` 실행
  4. 테스트 종료 → rollback
* 어제는 `IDENTITY`라서 `persist` 순간 `INSERT`가 나갔는데, 오늘은 PK를 직접 넣으니 **원래의 쓰기 지연**대로 `flush` 때 나가는 걸 볼 수 있음.

---

## 6. 1:N 단방향: @OneToMany (chap03 section02)

```java
@Entity(name = "Section02Category")
@Table(name = "tbl_category")
public class Category {

    @Id
    @Column(name = "category_code")
    private int categoryCode;
    ...
    @OneToMany(fetch = FetchType.EAGER)
    @JoinColumn(name="category_code", insertable = false, updatable = false)
    private List<Menu> menuList;
    ...
}
```

```java
@Entity(name = "Section02Menu")
@Table(name = "tbl_menu")
public class Menu {
    ...
    @Column(name = "category_code")
    private int categoryCode;   // 객체가 아니라 번호
    ...
}
```

* 이번엔 반대로 **카테고리 → 메뉴 목록** 방향만 있음. `Menu`는 카테고리를 객체가 아니라 `int categoryCode`로 들고 있음.
* `@JoinColumn(name="category_code")` → 1:N에서는 FK가 **상대 테이블(`tbl_menu`)** 에 있음. "`tbl_menu`의 `category_code`가 내 PK와 같은 메뉴들을 모아라"라는 뜻.
* `insertable = false, updatable = false` → 이 연관관계로는 `tbl_menu.category_code`를 **읽기만** 하고, `INSERT`/`UPDATE`에는 쓰지 않게 막음.
  * 같은 `category_code` 컬럼을 `Menu.categoryCode`도 관리하고 있어서, 두 군데서 같은 컬럼을 바꾸려 하면 충돌함.
  * 또 1:N 단방향은 FK가 **남의 테이블**에 있어서, 카테고리를 저장할 때 메뉴 쪽에 `UPDATE`를 **추가로** 날리게 됨 → 실습에서는 조회만 하도록 막아 둠.
* `fetch = FetchType.EAGER` → `@OneToMany`의 기본은 LAZY지만 여기서는 일부러 EAGER로 바꿔서, 카테고리를 조회할 때 **메뉴 목록까지 JOIN으로 한 번에** 가져오는 걸 확인함.

```java
@Test
@DisplayName("1:N 연관관계로 카테고리에서 메뉴 목록 조회")
void oneToManyFindTest() {
    int categoryCode = 4;

    Category category = oneToManyService.findCategory(categoryCode);
    List<Menu> menuList = category.getMenuList();

    System.out.println(category.getMenuList());
    assertNotNull(category);
    assertFalse(menuList.isEmpty());
}
```

* `find(Category, 4)` 한 번에 4번 카테고리 + 그 카테고리의 메뉴들이 같이 조회됨 → `menuList`가 비어 있지 않음.

---

## 7. 양방향 연관관계 (chap03 section03)

* section01(메뉴 → 카테고리)과 section02(카테고리 → 메뉴)를 **합친 것**. 양쪽 모두 상대를 들고 있음.

```java
// Menu: FK를 가진 쪽 → 연관관계의 주인
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "category_code", nullable = false)
private Category category;
```

```java
// Category: 주인이 아닌 쪽
@OneToMany(mappedBy = "category")
private List<Menu> menuList = new ArrayList<>();
```

### 연관관계의 주인과 mappedBy

* 테이블에는 FK가 `tbl_menu.category_code` **하나뿐**인데, 객체에는 `Menu.category`와 `Category.menuList` **두 개**가 있음 → 둘이 다른 값을 가지면 "**어느 쪽을 믿고 FK를 바꿀지**" 정해야 함.
* **연관관계의 주인** = FK 값을 **실제로 바꿀 권한**이 있는 쪽. 보통 **FK가 있는 테이블의 엔티티**(N 쪽, `Menu`)를 주인으로 정함.
* **한 줄 비유**: 자동차 키(FK)는 **하나**. 키를 가진 사람(`Menu.category`)만 차를 움직일 수 있고, 다른 사람(`Category.menuList`)은 차가 어디 있는지 **보기만** 할 수 있음.
* `mappedBy = "category"` → "나는 주인이 아니고, 주인은 `Menu`의 **`category` 필드**야"라는 표시.
  * DB 컬럼명(`category_code`)이 아니라 반대편 **자바 필드 이름**을 적음.
  * `mappedBy`가 붙은 쪽은 **읽기 전용** → `category.getMenuList().add(menu)`만 해서는 DB의 FK가 바뀌지 않음. `menu.setCategory(category)`를 해야 `UPDATE`/`INSERT`에 반영됨.

| | `Menu.category` | `Category.menuList` |
| :--- | :--- | :--- |
| 어노테이션 | `@ManyToOne` + `@JoinColumn` | `@OneToMany(mappedBy = "category")` |
| 역할 | **주인** (FK 등록·수정) | 조회용 (읽기 전용) |
| 값을 바꾸면 | DB에 반영됨 | DB에 반영 **안 됨** |

### 양쪽에서 조회

```java
@Test
@DisplayName("주인이 아닌 카테고리에서 메뉴 목록 조회")
void biDirectionFindTest2() {
    int categoryCode = 4;

    Category foundCategory = biDirectionService.findCategory(categoryCode);
    var menuList = foundCategory.getMenuList();

    assertEquals(categoryCode, foundCategory.getCategoryCode());
    assertFalse(menuList.isEmpty());
    assertSame(foundCategory, menuList.get(0).getCategory());
}
```

* 주인이 아닌 쪽에서도 **조회는 됨** → `getMenuList()`로 4번 카테고리의 메뉴들을 가져옴 (`@OneToMany` 기본값 LAZY라서 이때 `SELECT`).
* `menuList.get(0).getCategory()` → 메뉴가 가리키는 카테고리는 4번 → 같은 영속성 컨텍스트에 이미 있는 `foundCategory`와 **같은 객체** → `assertSame` 통과. 메뉴 → 카테고리 → 메뉴 → … 로 **양쪽을 오갈 수 있음**.

### 연관관계 편의 메소드

```java
public void addMenu(Menu menu) {
    menuList.add(menu);       // 카테고리의 메뉴 목록에 추가
    menu.setCategory(this);   // 메뉴에도 카테고리를 설정 (주인 쪽)
}
```

```java
@Test
@DisplayName("연관관계 편의 메소드로 객체 양쪽을 함께 연결")
void addMenuTest() {
    Category category = new Category(9002, "퓨전", null);
    Menu menu = new Menu(9002, "돈까스 스파게티", 15000, null, "Y");

    category.addMenu(menu);

    assertTrue(category.getMenuList().contains(menu));
    assertSame(category, menu.getCategory());
}
```

* 양방향이면 자바 객체에서도 **양쪽을 다** 맞춰 줘야 함.
  * `menuList.add(menu)`만 하면 → 객체끼리는 연결된 것 같지만 주인 쪽이 비어서 **DB에 FK가 안 들어감**.
  * `menu.setCategory(category)`만 하면 → DB는 맞지만, 같은 트랜잭션 안에서 `category.getMenuList()`에는 **아직 이 메뉴가 없음**.
* 그래서 한쪽을 바꿀 때 **반대쪽도 같이** 바꿔 주는 메소드를 하나 만들어 두고, 항상 그 메소드로 연결함 → 실수로 한쪽만 바꾸는 일을 막음.
* 이 테스트는 DB 없이 **순수 자바 객체**만으로 양쪽이 연결됐는지 확인함.
* `menuList`를 `new ArrayList<>()`로 **미리 초기화**해 둔 이유 → `new Category(...)`로 막 만든 객체에서 `addMenu`를 부를 때 `menuList`가 `null`이면 `NullPointerException`이 나기 때문.

> 💡 지금 `addMenu`는 메뉴가 **원래 다른 카테고리에 속해 있었을 때** 그 카테고리의 목록에서 빼 주지 않음. 메뉴의 카테고리를 옮기는 경우까지 생각하면 `menu.getCategory()`가 있을 때 기존 목록에서 `remove(menu)`를 먼저 해 주는 게 더 안전함.

### toString 무한 호출 주의

* `Menu.toString()`은 `category`를 출력하고, `Category.toString()`은 `menuList`를 **출력하지 않음**.
* 만약 양쪽 다 상대를 출력하면 → `Menu.toString()` → `Category.toString()` → `Menu.toString()` → … 끝없이 서로를 불러서 `StackOverflowError`가 남.
* 양방향에서는 `toString`(그리고 나중에 JSON 변환)에서 **한쪽은 상대를 빼야** 함.

---

## 8. 연관관계 정리

| | N:1 단방향 (section01) | 1:N 단방향 (section02) | 양방향 (section03) |
| :--- | :--- | :--- | :--- |
| `Menu` | `@ManyToOne Category category` | `int categoryCode` | `@ManyToOne Category category` (주인) |
| `Category` | 메뉴 필드 없음 | `@OneToMany @JoinColumn List<Menu>` | `@OneToMany(mappedBy) List<Menu>` |
| 이동 가능 방향 | 메뉴 → 카테고리 | 카테고리 → 메뉴들 | 양쪽 |
| FK 관리 | `Menu` | `Category` (실습에선 `insertable/updatable=false`로 막음) | `Menu` (주인) |

* 기본은 **N:1 단방향(`@ManyToOne`)으로 설계**하고, 반대 방향 조회가 꼭 필요할 때 `mappedBy`로 **양방향을 추가**하는 순서가 자연스러움. 양방향을 추가해도 테이블은 전혀 바뀌지 않음.

> ⚠️ 실습 코드에 쓰지 않는 import가 몇 군데 있음: `OneToManyRepository`의 `EnableAspectJAutoProxy`, `BiDirectionRepository`의 `jakarta.persistence.Entity`, section01 `MenuDTO`의 `jakarta.persistence.*`. DTO는 엔티티가 아니므로 JPA import가 필요 없음.

---

## 📝 오늘의 정리

1. Spring Boot에서는 `application.yml`에 DB·JPA 설정을 하고, `EntityManager`는 `@PersistenceContext`로 주입받으며, 트랜잭션은 `@Transactional`로 처리함. 테스트 클래스의 `@Transactional`은 끝나면 자동 rollback, `@Commit`은 commit
2. `@Column`의 `nullable`·`unique`·`length`는 테이블 생성 시 `NOT NULL`·`UNIQUE`·`varchar(n)`이 되고, `ddl-auto`가 `create`면 실행마다 테이블을 지우고 새로 만듦 (운영 DB에서는 `validate`/`none`)
3. enum은 순서가 바뀌어도 의미가 유지되도록 `@Enumerated(EnumType.STRING)`으로 이름을 저장함
4. `@Embeddable` 값 객체를 `@Embedded`로 넣으면 별도 테이블 없이 엔티티 테이블의 컬럼으로 펼쳐지고, 검증·계산 규칙을 값 객체 안에 모을 수 있음
5. `@ManyToOne` + `@JoinColumn`으로 FK 번호 대신 객체를 참조하고, `LAZY`면 실제 값을 쓸 때 `SELECT`가 나가며 트랜잭션 밖에서 쓰면 `LazyInitializationException`이 남
6. `@OneToMany` 단방향은 FK가 상대 테이블에 있어서 `@JoinColumn`이 상대 테이블 컬럼을 가리킴
7. 양방향에서는 FK를 가진 `Menu.category`가 연관관계의 주인이고, 반대편은 `mappedBy = "필드명"`으로 읽기 전용이 됨. 객체 양쪽을 맞추기 위해 연관관계 편의 메소드를 쓰고, `toString`은 한쪽만 상대를 출력하게 함

---

## 느낀 점

* 어제 직접 만들던 `EntityManagerFactory`, `begin()`·`rollback()`이 Spring Boot에서 설정 몇 줄과 어노테이션으로 바뀌는 걸 보니, 어제 그 과정을 먼저 손으로 해 본 덕분에 Spring이 뒤에서 무슨 일을 해 주는지 그려졌음
* `EnumType.ORDINAL`이 기본값인데 순서가 바뀌면 기존 데이터 의미가 통째로 바뀐다는 점이 무서웠음. enum은 항상 `STRING`으로 저장해야겠음
* `Price`처럼 값 객체 안에 검증과 계산을 넣으니 잘못된 가격이 아예 만들어질 수 없다는 게 좋았음
* `mappedBy` 쪽에 아무리 `add`해도 DB에 반영되지 않는다는 게 처음엔 이상했는데, FK는 하나뿐이라 주인을 한쪽으로 정해야 한다는 설명으로 이해됨
* LAZY 로딩이 테스트에서는 잘 되는데 트랜잭션 밖에서는 예외가 난다는 걸 알게 되어, 앞으로 컨트롤러로 엔티티를 그대로 넘기지 말고 서비스에서 DTO로 바꿔야겠다고 느낌
