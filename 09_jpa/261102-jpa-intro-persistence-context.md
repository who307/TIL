# JDBC 직접 SQL의 문제점 · JPA EntityManager · 영속성 컨텍스트 · 엔티티 생명주기 정리 (TIL)

> 실습 환경: MySQL (`menudb`의 `TBL_MENU`·`TBL_CATEGORY`), Gradle 프로젝트 (`chap00-intro`, `chap01-persistance-context`), Hibernate 7.2 (JPA 구현체), JUnit 5

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| chap00-intro | JDBC로 SQL을 **직접** 다룰 때 생기는 문제 4가지를 테스트로 확인 |
| chap01 section01 | JPA 설정(`persistence.xml`), `EntityManagerFactory`·`EntityManager` 만들기 |
| chap01 section02 | `EntityManager`로 CRUD: `find` / `persist` / 변경 감지 / `remove` |
| chap01 section03 | 엔티티 생명주기: 비영속 · 영속 · 준영속 · 삭제 (`contains`, `detach`, `merge`, `clear`) |

---

## 1. JDBC로 SQL을 직접 다룰 때의 문제점

* 10/22~23에 JDBC로 `Connection` → `Statement` → `ResultSet`을 직접 다뤘음. 오늘은 그 방식의 **불편한 점**을 테스트로 하나씩 확인하고, 그걸 해결하려고 나온 게 **JPA**라는 흐름으로 시작함.

### 테스트 준비: 끝나면 항상 rollback

```java
@BeforeEach
void setConnection() throws SQLException {
    String url = "jdbc:mysql://localhost:3306/menudb";
    String user = "사용자명";
    String password = "비밀번호";

    con = DriverManager.getConnection(url, user, password);
    con.setAutoCommit(false);
}

@AfterEach
void closeConnection() throws SQLException {
    if (con != null && !con.isClosed()) {
        con.rollback();
        con.close();
    }
}
```

* `@BeforeEach` / `@AfterEach` → 테스트 메소드 **하나하나마다** 앞뒤로 실행됨.
* `setAutoCommit(false)` → SQL을 실행해도 바로 확정(commit)하지 않음 → 테스트가 끝나면 `rollback()`으로 **없던 일**로 만듦. 그래서 `INSERT` 테스트를 몇 번 돌려도 DB가 지저분해지지 않음.

### 문제 1. 같은 코드를 계속 반복해서 씀

```java
String query = "SELECT MENU_CODE, MENU_NAME, MENU_PRICE, CATEGORY_CODE, ORDERABLE_STATUS FROM TBL_MENU";

Statement stmt = con.createStatement();
ResultSet rset = stmt.executeQuery(query);

List<Menu> menuList = new ArrayList<>();
while(rset.next()) {
    Menu menu = new Menu();
    menu.setMenuCode(rset.getInt("MENU_CODE"));
    menu.setMenuName(rset.getString("MENU_NAME"));
    menu.setMenuPrice(rset.getInt("MENU_PRICE"));
    menu.setCategoryCode(rset.getInt("CATEGORY_CODE"));
    menu.setOrderableStatus(rset.getString("ORDERABLE_STATUS"));

    menuList.add(menu);
}
```

* **한 줄 비유**: 택배 상자(`ResultSet`)를 받을 때마다 물건을 **하나씩 꺼내서 선반(객체)에 손으로 옮겨 담는** 일. 상자 모양이 같아도 매번 똑같이 꺼내야 함.
* `INSERT`도 마찬가지로 `pstmt.setString(1, menu.getMenuName())`처럼 **객체 → `?` 자리**로 하나씩 옮겨야 함.
* 테이블이 10개면 이 "꺼내서 옮기기" 코드도 10벌 → 개발 시간이 늘고, 컬럼 하나 추가하면 고칠 곳도 여러 군데.

### 문제 2. SQL에 끌려다니는 개발 (SQL 의존)

```java
String query = "SELECT MENU_CODE, MENU_NAME, MENU_PRICE FROM TBL_MENU";
...
menu.setMenuCode(rset.getInt("MENU_CODE"));
menu.setMenuName(rset.getString("MENU_NAME"));
menu.setMenuPrice(rset.getInt("MENU_PRICE"));
```

* 조회 컬럼을 3개로 줄이면 `Menu` 객체의 `categoryCode`는 `0`, `orderableStatus`는 `null`로 남음.
* 객체만 보고는 "**정말 0인지, 조회를 안 해서 0인지**" 알 수 없음 → 결국 **SQL을 열어 봐야** 객체를 믿을 수 있음.
* 화면에 컬럼 하나를 더 보여주고 싶으면 SQL → 매핑 코드 → DTO를 **줄줄이** 고쳐야 함.

### 문제 3. 패러다임 불일치 (객체 ↔ 테이블)

* **한 줄 비유**: 객체는 **"주머니에 물건을 직접 넣어 다니는"** 방식, 테이블은 **"물건 대신 보관함 번호표(FK)만 들고 다니는"** 방식. 둘이 생각하는 방식이 다름.

**상속**
* 객체는 `법인회원 extends 회원`이면 부모 필드를 그냥 같이 씀.
* DB는 상속이 없어서 `회원` 테이블과 `법인회원` 테이블로 **나눠** 저장 → `INSERT INTO 회원 ...`, `INSERT INTO 법인회원 ...` **두 번** 써야 함.
* JPA로 상속을 매핑하면 `entityManager.persist(법인회원)` **한 번**으로 끝남.

**연관관계 / 객체 그래프 탐색**

| 테이블에 맞춘 객체 (`Menu`) | 객체지향에 맞춘 객체 (`MenuAndCategory`) |
| :--- | :--- |
| `private int categoryCode;` (FK 번호만) | `private Category category;` (객체를 통째로) |
| 카테고리 이름이 필요하면 **번호로 다시 조회** | `menu.getCategory().getCategoryName()`으로 **바로 이동** |

```java
String query = "SELECT A.MENU_CODE, A.MENU_NAME, A.MENU_PRICE, B.CATEGORY_CODE, B.CATEGORY_NAME, A.ORDERABLE_STATUS" +
        " FROM TBL_MENU A" +
        " JOIN TBL_CATEGORY B ON (A.CATEGORY_CODE = B.CATEGORY_CODE)";
...
menuAndCategory.setCategory(new Category(rset.getInt("CATEGORY_CODE"), rset.getString("CATEGORY_NAME")));
```

* 객체지향 모델을 쓰려면 개발자가 직접 **JOIN을 쓰고**, `new Category(...)`로 **조립**까지 해야 함.
* 그리고 SQL에 JOIN이 **있었는지 없었는지**에 따라 `getCategory()`가 `null`일 수도 있음 → 객체를 어디까지 따라가도 되는지(객체 그래프 탐색) 믿을 수 없음.

### 문제 4. 같은 행인데 다른 객체 (동일성 보장 X)

```java
String query = "SELECT MENU_CODE, MENU_NAME FROM TBL_MENU WHERE MENU_CODE = 1";
// 같은 쿼리를 두 번 실행해서 menu1, menu2를 각각 new로 만듦

Assertions.assertNotSame(menu1, menu2);                            // 다른 객체
Assertions.assertEquals(menu1.getMenuName(), menu2.getMenuName()); // 내용은 같음
```

* DB에서는 **같은 1번 메뉴**인데, 조회할 때마다 `new Menu()`를 하니까 `menu1 == menu2`는 `false`.
* 자바 컬렉션에서는 같은 걸 두 번 꺼내면 **같은 객체**가 나옴. JDBC로는 그게 안 됨 → 이 문제를 JPA는 아래 **영속성 컨텍스트**로 해결함 (3장 `assertSame` 테스트와 비교).

---

## 2. JPA 시작하기: 설정과 EntityManager

### JPA · Hibernate · ORM

* **ORM** (Object-Relational Mapping): 객체와 테이블을 **연결(매핑)** 해 두면, SQL과 "꺼내서 옮기기" 코드를 프레임워크가 **대신** 만들어 주는 기술.
* **JPA**: 자바 ORM의 **표준 규칙(인터페이스 모음)**. `jakarta.persistence.*` 패키지.
* **Hibernate**: JPA 규칙을 **실제로 구현한** 라이브러리. JPA가 "운전면허 규칙"이면 Hibernate는 "실제 자동차".

```gradle
implementation 'com.mysql:mysql-connector-j:8.0.33'
implementation 'org.hibernate.orm:hibernate-core:7.2.25.Final'
```

* 아직 Spring Boot 없이 **순수 JPA**만으로 실습함 → 설정 파일도 직접 씀.

### persistence.xml: JPA 설정 파일

```xml
<!-- src/main/resources/META-INF/persistence.xml -->
<persistence-unit name="jpatest">

    <class>com.ohgiraffers.section02.crud.Menu</class>
    <class>com.ohgiraffers.section03.entity.Menu</class>

    <properties>
        <!-- 데이터 베이스 연결 정보 -->
        <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="jakarta.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/menudb"/>
        <property name="jakarta.persistence.jdbc.user" value="사용자명"/>
        <property name="jakarta.persistence.jdbc.password" value="비밀번호"/>
        <!-- Hibernate가 실행하는 SQL을 확인하기 위한 설정 -->
        <property name="hibernate.show_sql" value="true"/>
        <property name="hibernate.format_sql" value="true"/>
    </properties>
</persistence-unit>
```

* 위치가 정해져 있음 → **`META-INF/persistence.xml`**. JPA가 이 경로를 찾아 읽음.
* `persistence-unit name="jpatest"` → 설정 묶음의 **이름표**. 코드에서 `Persistence.createEntityManagerFactory("jpatest")`로 이 이름을 불러 씀.
* `<class>` → "이 클래스들이 **엔티티**(테이블과 연결된 클래스)야"라고 등록.
* `show_sql` → Hibernate가 **대신 만든 SQL을 콘솔에 출력**. `format_sql` → 그 SQL을 줄바꿈해서 **보기 좋게** 출력. 내가 SQL을 안 써도 뭐가 실행되는지 눈으로 확인할 수 있음.

### 엔티티: 테이블과 연결된 클래스

```java
@Entity(name = "Section02Menu")
@Table(name = "tbl_menu")
public class Menu {

    @Id
    @Column(name = "menu_code")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int menuCode;

    @Column(name = "menu_name")
    private String menuName;

    @Column(name = "menu_price")
    private int menuPrice;

    @Column(name = "category_code")
    private int categoryCode;

    @Column(name = "orderable_status")
    private String orderableStatus;

    protected Menu() {}

    public Menu(String menuName, int menuPrice, int categoryCode, String orderableStatus) { ... }
    // getter / setter / toString
}
```

| 어노테이션 | 뜻 |
| :--- | :--- |
| `@Entity` | 이 클래스는 JPA가 관리하는 **엔티티** |
| `@Table(name = "tbl_menu")` | 연결할 **테이블 이름** |
| `@Id` | **기본키(PK)** 필드 |
| `@Column(name = "menu_code")` | 연결할 **컬럼 이름** |
| `@GeneratedValue(strategy = IDENTITY)` | PK 값은 **DB가 `AUTO_INCREMENT`로** 만들어 줌 |

* `@Entity(name = "Section02Menu")` → 엔티티 이름은 기본값이 **클래스 이름(`Menu`)**. section02와 section03에 `Menu`가 **두 개** 있어서 이름이 겹치면 안 되므로 따로 이름을 붙임.
* `protected Menu() {}` → JPA는 DB에서 꺼낸 값으로 객체를 만들 때 **기본 생성자로 빈 객체를 먼저 만들고** 값을 채움 → 기본 생성자가 **꼭 있어야** 함. `protected`로 둬서 다른 개발자가 함부로 빈 객체를 만들지 못하게 함.
* 등록용 생성자에 `menuCode`가 없음 → 번호는 DB가 정해 주니까.

### EntityManagerFactory: 딱 하나만

```java
public class EntityManagerFactoryGenerator {

    private static final EntityManagerFactory FACTORY
            = Persistence.createEntityManagerFactory("jpatest");

    private EntityManagerFactoryGenerator() {}

    public static EntityManagerFactory getInstance() {
        return FACTORY;
    }
}
```

* **한 줄 비유**: `EntityManagerFactory`는 **공장**, `EntityManager`는 공장에서 찍어 내는 **작업자**. 공장은 짓는 데 돈이 많이 드니 **하나만** 짓고, 작업자는 일이 생길 때마다 새로 뽑음.
* `Persistence.createEntityManagerFactory("jpatest")` → `persistence.xml`을 읽고, DB 연결과 엔티티 정보를 준비함 → **무거운 작업**.
* `static final` + `private` 생성자 → 10/12에 배운 **싱글톤**. 클래스가 처음 쓰일 때 한 번만 만들어지고, `getInstance()`는 항상 **같은 공장**을 돌려줌.

### EntityManager: 쓸 때마다 새로

```java
public class EntityManagerGenerator {

    private EntityManagerGenerator() {}

    public static EntityManager getInstance() {
        EntityManagerFactory factory = EntityManagerFactoryGenerator.getInstance();
        return factory.createEntityManager();
    }
}
```

* `createEntityManager()` → 부를 때마다 **새** `EntityManager`를 만듦.
* `EntityManager`는 여러 스레드가 **같이 쓰면 안 됨**(thread-safe 아님). 그래서 요청 하나·작업 하나마다 새로 만들어 쓰고 끝나면 `close()`함.
* 이름은 `getInstance()`지만 **싱글톤이 아님**에 주의.

### 테스트로 확인

```java
@Test
@DisplayName("엔티티 매니저 팩토리 싱글톤 인스턴스 확인")
void testIsEntityManagerFactorySingleton() {
    EntityManagerFactory factory1 = EntityManagerFactoryGenerator.getInstance();
    EntityManagerFactory factory2 = EntityManagerFactoryGenerator.getInstance();

    assertSame(factory1, factory2);         // 공장은 같은 것
}

@Test
@DisplayName("엔티티 매니저 스코프 확인")
void testEntityManagerLifeCycle() {
    EntityManager entityManager1 = EntityManagerGenerator.getInstance();
    EntityManager entityManager2 = EntityManagerGenerator.getInstance();

    assertNotSame(entityManager1, entityManager2);   // 작업자는 매번 다른 것
}
```

| 메소드 | 확인하는 것 |
| :--- | :--- |
| `assertSame(a, b)` | `a == b` → **같은 객체**(주소가 같음) |
| `assertNotSame(a, b)` | `a != b` → **다른 객체** |
| `assertEquals(a, b)` | `a.equals(b)` → **내용이 같음** |

> ⚠️ section01의 `EntityManagerGenerator`에 `import javax.swing.text.html.parser.Entity;`가 들어 있음. IDE 자동 import가 엉뚱한 `Entity`를 가져온 것으로, 쓰이지 않으니 지워도 됨. JPA의 `@Entity`는 `jakarta.persistence.Entity`임.

---

## 3. EntityManager로 CRUD

### 영속성 컨텍스트가 뭔가

* **한 줄 비유**: `EntityManager`가 들고 다니는 **작업용 책상**. DB에서 꺼낸 엔티티를 책상 위에 올려 두고(**1차 캐시**), 바뀐 게 있는지 지켜보다가(**변경 감지**), 정리할 때(`flush`) 한꺼번에 DB에 반영함.
* 책상 위에 올라가 있는 상태 = **영속(managed)** 상태.
* `EntityManager`마다 **책상이 따로** 있음.

### 테스트 준비: 트랜잭션 시작 → rollback

```java
@BeforeEach
void setUp() {
    entityManager = EntityManagerGenerator.getInstance();
    transaction = entityManager.getTransaction();
    transaction.begin();
    crud = new EntityManagerCRUD(entityManager);
}

@AfterEach
void tearDown() {
    if (transaction != null && transaction.isActive()) {
        transaction.rollback();
    }
    if (entityManager != null && entityManager.isOpen()) {
        entityManager.close();
    }
}
```

* JPA에서 **데이터를 바꾸는 작업**(`persist`, `remove`, `flush` 등)은 **트랜잭션 안에서만** 할 수 있음.
* `transaction.begin()` → 작업 시작. `rollback()` → 테스트가 끝나면 전부 되돌림 (chap00의 `setAutoCommit(false)` + `rollback()`과 같은 목적).
* 실제 서비스라면 마지막에 `transaction.commit()`으로 **확정**함.

### EntityManagerCRUD

```java
public class EntityManagerCRUD {
    private final EntityManager entityManager;

    public EntityManagerCRUD(EntityManager entityManager){
        this.entityManager = entityManager;
    }

    public Menu findMenuByMenuCode(int menuCode) {
        return entityManager.find(Menu.class, menuCode);
    }

    public void saveMenu(Menu newMenu){
        entityManager.persist(newMenu);
    }

    public Menu modifyMenuName(int menuCode, String menuName){
        Menu foundMenu = entityManager.find(Menu.class, menuCode);
        foundMenu.setMenuName(menuName);
        return foundMenu;
    }

    public void removeMenu(int menuCode) {
        Menu foundMenu = entityManager.find(Menu.class, menuCode);
        if(foundMenu != null){
            entityManager.remove(foundMenu);
        }
    }
}
```

* 어디에도 **SQL 문자열이 없음**. `rset.getInt(...)`로 옮겨 담는 코드도 없음 → chap00의 문제 1·2가 사라짐.

| 하고 싶은 일 | JDBC (chap00) | JPA |
| :--- | :--- | :--- |
| 조회 | `SELECT` 작성 + `ResultSet` → 객체 옮기기 | `find(Menu.class, 1)` |
| 등록 | `INSERT` 작성 + `?`에 하나씩 넣기 | `persist(menu)` |
| 수정 | `UPDATE` 작성 + 실행 | 영속 객체의 **setter만 호출** |
| 삭제 | `DELETE` 작성 + 실행 | `remove(menu)` |

### 조회: find

* `find(엔티티 클래스, PK 값)` → 먼저 **책상(1차 캐시)** 을 봄 → 없으면 DB에 `SELECT`를 날리고, 결과를 책상에 올린 뒤 돌려줌.
* 없는 번호면 예외가 아니라 **`null`** 을 돌려줌.

### 등록: persist

```java
@Test
@DisplayName("새로운 메뉴 저장")
void testSaveMenu() {
    Menu newMenu = new Menu("신메뉴", 35000, 4, "Y");

    crud.saveMenu(newMenu);
    entityManager.flush(); // 영속성 컨텍스트의 변경 내용을 SQL로 DB에 동기화하라고 요청
    int newMenuCode = newMenu.getMenuCode();
    entityManager.clear(); // 영속성 컨텍스트 비우기

    Menu savedMenu = crud.findMenuByMenuCode(newMenuCode);
    assertNotNull(savedMenu);
    assertEquals("신메뉴", savedMenu.getMenuName());
}
```

* **따라가기**
  1. `new Menu(...)` → 그냥 자바 객체. `menuCode`는 아직 `0`.
  2. `persist(newMenu)` → 책상 위에 올림(영속). PK 전략이 `IDENTITY`라 번호를 **DB가 줘야** 하므로 Hibernate가 이때 바로 `INSERT`를 실행하고, 받은 번호를 `newMenu.menuCode`에 채워 넣음.
  3. `flush()` → 책상에 쌓인 변경 내용을 SQL로 DB에 보냄 (commit은 아님).
  4. `clear()` → 책상을 **싹 비움**.
  5. `find(newMenuCode)` → 책상이 비었으니 **DB에서 `SELECT`** → 정말로 DB에 들어갔는지 확인됨.
* `clear()` 없이 `find`를 하면 책상에 있던 `newMenu`를 **그대로** 돌려줘서 DB까지 갔는지 확인이 안 됨. 그래서 `flush` + `clear` 조합을 씀.

| 메소드 | 하는 일 | 비유 |
| :--- | :--- | :--- |
| `flush()` | 책상의 변경 내용을 SQL로 DB에 **보냄** (책상은 그대로) | 메모한 걸 장부에 옮겨 적기 |
| `clear()` | 책상을 **비움** (DB는 그대로) | 책상 위 서류 전부 치우기 |
| `commit()` | `flush` 후 트랜잭션을 **확정** | 장부에 도장 찍기 |
| `rollback()` | 트랜잭션 동안 한 일을 **취소** | 장부 고친 걸 원래대로 |

> 💡 **쓰기 지연**: 원래 JPA는 `persist`를 해도 `INSERT`를 바로 보내지 않고 모아 뒀다가 `flush`/`commit` 때 한꺼번에 보냄. 다만 `IDENTITY` 전략은 **DB에 넣어 봐야 번호를 알 수 있어서** 예외적으로 `persist` 순간에 `INSERT`가 나감.

### 수정: 변경 감지 (dirty checking)

```java
@Test
@DisplayName("영속 상태 메뉴의 이름변경")
void testModifyMenuName() {
    crud.modifyMenuName(1, "변경된 이름");
    entityManager.flush();
    entityManager.clear();

    Menu modifiedMenu = crud.findMenuByMenuCode(1);
    assertEquals("변경된 이름", modifiedMenu.getMenuName());
}
```

* `modifyMenuName`에는 `update()` 같은 메소드 호출이 **없음**. `setMenuName()`만 했을 뿐인데 DB가 바뀜.
* **한 줄 비유**: 책상에 서류를 올릴 때 **복사본(스냅샷)** 을 한 장 같이 떠 둠. `flush` 때 원본과 복사본을 비교해서 **달라진 게 있으면** 알아서 `UPDATE`를 만듦.
* **따라가기**
  1. `find(1)` → `SELECT` → 1번 메뉴를 책상에 올리고 스냅샷 저장
  2. `setMenuName("변경된 이름")` → 객체만 바뀜 (아직 SQL 없음)
  3. `flush()` → 스냅샷과 비교 → 이름이 다름 → `UPDATE tbl_menu SET ... WHERE menu_code=1` 실행
  4. `clear()` 후 `find(1)` → DB에서 다시 `SELECT` → `"변경된 이름"` 확인
* 변경 감지는 **영속 상태**인 객체에만 동작함 (4장의 `detach` 테스트에서 확인).

> ⚠️ `modifyMenuName`은 `find` 결과가 `null`인지 확인하지 않음 → 없는 번호를 넣으면 `foundMenu.setMenuName(...)`에서 `NullPointerException`. `removeMenu`처럼 `null` 확인을 하거나 예외를 던지는 게 안전함.

### 삭제: remove

```java
@Test
@DisplayName("메뉴 삭제")
void testRemoveMenu() {
    crud.removeMenu(1);
    entityManager.flush();
    entityManager.clear();

    assertNull(crud.findMenuByMenuCode(1));
}
```

* `remove()`는 **영속 상태인 객체**만 받을 수 있음 → 그래서 먼저 `find`로 책상에 올린 다음 `remove`함.
* `remove` → "삭제 예정" 표시 → `flush` 때 `DELETE` 실행 → 다시 `find(1)`하면 `null`.

---

## 4. 엔티티 생명주기

```text
            new Menu(...)
                 │
                 ▼
          [ 비영속 (new) ]
                 │ persist()          find()로 DB에서 꺼내도 바로 영속
                 ▼
          [ 영속 (managed) ] ◀──── merge() ──── [ 준영속 (detached) ]
             │        └────── detach() / clear() / close() ──────▲
             │ remove()
             ▼
          [ 삭제 (removed) ] ── flush/commit 시 DELETE
```

| 상태 | 뜻 | 만드는 방법 | `contains()` |
| :--- | :--- | :--- | :--- |
| 비영속 (new/transient) | 만들어졌지만 JPA가 **모르는** 객체 | `new` | `false` |
| 영속 (managed) | 책상 위에서 **관리 중** | `persist()`, `find()`, `merge()`의 반환값 | `true` |
| 준영속 (detached) | 한때 관리됐지만 **책상에서 내려온** 객체 | `detach()`, `clear()`, `close()` | `false` |
| 삭제 (removed) | **삭제 예정** 표시된 객체 | `remove()` | `false` |

* `entityManager.contains(객체)` → 이 객체가 지금 **책상 위에 있나(영속인가)** 를 `true`/`false`로 알려줌. 실습 코드에서는 이걸 `inManaged()` 메소드로 감쌌음.

### 비영속 vs 영속: 값이 같아도 다르다

```java
Menu managedMenu = lifeCycle.findMenuByMenuCode(2);
Menu transientMenu = new Menu(
        managedMenu.getMenuCode(),
        managedMenu.getMenuName(),
        managedMenu.getMenuPrice(),
        managedMenu.getCategoryCode(),
        managedMenu.getOrderableStatus()
);

assertTrue(lifeCycle.inManaged(managedMenu));     // find로 꺼낸 것 → 영속
assertFalse(lifeCycle.inManaged(transientMenu));  // new로 만든 것 → 비영속
```

* `transientMenu`는 2번 메뉴와 **번호·이름·가격까지 똑같지만** `new`로 만들었으니 JPA는 모르는 객체 → 비영속.
* JPA는 "값이 같은가"가 아니라 "**내 책상 위에 있는 그 객체인가**"로 관리 여부를 판단함.

### 동일성 보장: 같은 EntityManager vs 다른 EntityManager

```java
@Test
@DisplayName("같은 EntityManager에서 동일한 엔티티 조회")
void testManagedBySameEntityManager() {
    Menu menu1 = entityManager.find(Menu.class, 3);
    Menu menu2 = entityManager.find(Menu.class, 3);

    assertSame(menu1, menu2);
}

@Test
@DisplayName("서로 다른 EntityManager가 조회한 엔티티 비교")
void testManagedByDifferentEntityManager(){
    EntityManager otherEntityManager = EntityManagerGenerator.getInstance();

    Menu menu1 = entityManager.find(Menu.class, 3);
    Menu menu2 = otherEntityManager.find(Menu.class, 3);

    assertNotSame(menu1, menu2);
}
```

* **같은** `EntityManager`
  1. 첫 번째 `find(3)` → 책상에 없음 → `SELECT` → 책상에 올림
  2. 두 번째 `find(3)` → 책상에 **이미 있음** → SQL 없이 **같은 객체**를 돌려줌 → `assertSame` 통과
  * 콘솔에도 `SELECT`가 **한 번만** 찍힘 (1차 캐시).
  * chap00 문제 4(JDBC는 같은 행인데 다른 객체)가 여기서 해결됨.
* **다른** `EntityManager` → 책상이 **따로**라서 각자 `SELECT` → 다른 객체 → `assertNotSame`.

> ⚠️ 이 테스트에서 만든 `otherEntityManager`는 `close()`하지 않음. `@AfterEach`는 필드의 `entityManager`만 닫으므로, 테스트 안에서 따로 만든 건 직접 `otherEntityManager.close()`해 주는 게 좋음.

### 준영속: detach하면 변경 감지가 꺼진다

```java
@Test
@DisplayName("detach(준영속) 후 변경 감지가 동작하지 않는지 확인")
void testDetachEntity(){
    Menu foundMenu = entityManager.find(Menu.class, 10);
    int originalPrice = foundMenu.getMenuPrice();

    entityManager.detach(foundMenu); // 준영속화
    foundMenu.setMenuPrice(originalPrice + 1000);
    entityManager.flush();

    Menu reloadedMenu = entityManager.find(Menu.class, 10);
    assertEquals(originalPrice, reloadedMenu.getMenuPrice());
    assertFalse(entityManager.contains(foundMenu));
}
```

* **따라가기**
  1. `find(10)` → 영속, 원래 가격 저장
  2. `detach(foundMenu)` → **이 객체만** 책상에서 내려놓음 → 준영속
  3. 가격을 1000원 올려도 JPA는 더 이상 이 객체를 **지켜보지 않음**
  4. `flush()` → 책상 위에 바뀐 게 없으니 `UPDATE` **안 나감**
  5. `find(10)` → 책상에 10번이 없음 → 다시 `SELECT` → **새 객체**, 가격은 원래대로
* `detach(객체)`는 하나만, `clear()`는 책상 **전체**, `close()`는 책상을 아예 **치워 버림**(이후 그 `EntityManager`는 못 씀).

### merge: 준영속 → 다시 영속 (단, 새 객체로)

```java
@Test
@DisplayName("merge가 반환한 새로운 영속 객체 확인")
void testMergeEntity(){
    Menu detachedMenu = entityManager.find(Menu.class, 10);
    entityManager.detach(detachedMenu);
    detachedMenu.setMenuName("병합된 메뉴 이름");

    Menu mergedMenu = entityManager.merge(detachedMenu);
    // detachedMenu를 mergedMenu에 복사한다
    // merge는 전달한 객체를 다시 관리하지 않고 관리되는 객체를 반환한다.
    entityManager.flush();

    assertFalse(entityManager.contains(detachedMenu));
    assertTrue(entityManager.contains(mergedMenu));
    assertNotSame(detachedMenu, mergedMenu);
    assertEquals("병합된 메뉴 이름", mergedMenu.getMenuName());
}
```

* **한 줄 비유**: 책상에서 내려간 서류를 **그대로 다시 올리는 게 아니라**, 책상 위 원본에 내용을 **베껴 적고** 그 원본을 돌려줌.
* **따라가기**
  1. `find(10)` 후 `detach` → `detachedMenu`는 준영속
  2. 이름을 `"병합된 메뉴 이름"`으로 바꿈 (아직 아무 일도 없음)
  3. `merge(detachedMenu)` → 책상에 10번이 없으니 `SELECT`로 **새 영속 객체**를 만듦 → `detachedMenu`의 값을 거기에 **복사** → 그 영속 객체를 반환
  4. `flush()` → 영속 객체의 이름이 DB와 다름 → 변경 감지 → `UPDATE`
  5. `detachedMenu`는 **여전히 준영속**, `mergedMenu`가 영속 → 둘은 다른 객체
* 그래서 `merge` 뒤에는 **반환값(`mergedMenu`)을 써야** 함. `detachedMenu`를 계속 고쳐도 DB에 반영되지 않음.

### clear: 책상 전체 비우기

```java
@Test
@DisplayName("clear 후 같은 엔티티를 다시 조회하면 새 객체가 반환되는지 확인")
void testClearPersistenceContext() {
    Menu firstMenu = entityManager.find(Menu.class, 5);

    entityManager.clear();
    Menu secondMenu = entityManager.find(Menu.class, 6);

    assertFalse(entityManager.contains(firstMenu));
    assertNotSame(firstMenu, secondMenu);
}
```

* `clear()` → 책상 위 엔티티가 **전부 준영속**이 됨 → `contains(firstMenu)`는 `false`.

> ⚠️ 테스트 이름은 "**같은** 엔티티를 다시 조회"인데 코드는 5번을 조회한 뒤 **6번**을 조회함. 5번과 6번은 원래 다른 행이라 `assertNotSame`은 `clear()`를 안 해도 통과함 → `clear()`의 효과를 확인하지 못함. `find(Menu.class, 5)`로 고쳐야 "같은 5번인데 `clear()` 때문에 `SELECT`가 다시 나가고 **새 객체**가 나온다"를 확인할 수 있음. (`clear()`를 빼고 5번을 두 번 조회하면 `assertSame`이 됨)

### remove: 삭제 상태

```java
Menu foundMenu = entityManager.find(Menu.class, 6);

entityManager.remove(foundMenu);   // 삭제 상태
entityManager.flush();             // DELETE 실행
entityManager.clear();

assertNull(entityManager.find(Menu.class, 6));
```

* `remove` → 삭제 상태(아직 DB에는 있음) → `flush` 때 `DELETE` → 이후 `find`는 `null`.
* 테스트 마지막에 `rollback()`되므로 실제 6번 메뉴는 지워지지 않음.

---

## 📝 오늘의 정리

1. JDBC로 SQL을 직접 다루면 ① 조회·등록마다 같은 매핑 코드를 반복하고 ② SQL을 봐야 객체를 믿을 수 있고 ③ 상속·연관관계 같은 객체 구조와 테이블 구조가 맞지 않고 ④ 같은 행을 두 번 조회하면 다른 객체가 나옴
2. JPA는 자바 ORM 표준이고 Hibernate가 그 구현체이며, `META-INF/persistence.xml`의 `persistence-unit`에 DB 접속 정보와 엔티티 클래스를 등록함
3. `@Entity`·`@Table`·`@Id`·`@Column`·`@GeneratedValue(IDENTITY)`로 클래스와 테이블을 연결하고, JPA가 객체를 만들 수 있도록 기본 생성자를 둠
4. `EntityManagerFactory`는 무거우니 싱글톤으로 하나만, `EntityManager`는 작업마다 새로 만들고 닫음
5. `find`·`persist`·`remove`로 조회·등록·삭제하고, 수정은 영속 객체의 setter만 호출하면 `flush` 때 변경 감지로 `UPDATE`가 나감. 데이터 변경은 트랜잭션 안에서 함
6. 엔티티는 비영속 → 영속 → 준영속/삭제 상태를 오가며, 같은 `EntityManager` 안에서는 1차 캐시 덕분에 같은 PK를 조회하면 같은 객체가 나옴(`assertSame`)
7. `detach`된 객체는 변경 감지가 안 되고, `merge`는 넘긴 객체가 아니라 값을 복사한 새 영속 객체를 반환함

---

## 느낀 점

* chap00에서 JDBC 코드의 문제를 먼저 직접 겪고 나서 JPA를 보니, `find` 한 줄과 SQL 없는 CRUD가 왜 반가운지 확실히 느껴졌음
* `update()` 호출 없이 setter만으로 `UPDATE`가 나가는 변경 감지가 가장 신기했음. 대신 그 객체가 지금 영속 상태인지를 항상 의식해야 한다는 점을 `detach` 테스트로 알게 됨
* `merge`가 넘긴 객체를 다시 관리하는 게 아니라 새 객체를 돌려준다는 점은 헷갈리기 쉬워서, 반환값을 받아 써야 한다는 걸 꼭 기억해야겠음
* `flush`와 `clear`를 같이 써야 "DB에 정말 반영됐는지"를 확인할 수 있다는 게 처음엔 이해가 안 됐는데, 1차 캐시 때문에 `clear` 없이는 DB까지 가지 않는다는 걸 알고 나서 납득됨
* `testClearPersistenceContext`처럼 테스트가 통과해도 실제로는 확인하려던 걸 확인하지 못하는 경우가 있다는 것을 보고, 테스트 이름과 코드가 맞는지도 점검해야겠다고 느낌
