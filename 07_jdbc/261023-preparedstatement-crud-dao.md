# JDBC PreparedStatement · SQL Injection · CRUD · DAO 정리 (TIL)

> 실습 환경: MySQL (`menudb`의 `TBL_MENU`·`TBL_CATEGORY`, `employeedb`의 `EMPLOYEE`), Gradle 프로젝트

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| chap02-statements | `JDBCTemplate`에 `close` 오버로딩 추가, `Statement` → `PreparedStatement`, XML로 쿼리 분리, SQL Injection 비교 |
| chap03-crud | `executeUpdate()`로 `INSERT` / `UPDATE` / `DELETE` |
| chap04-dao | 모든 코드가 `main`에 몰린 문제 → **DAO**로 DB 접근 코드 분리 |

---

## 1. JDBCTemplate에 close 오버로딩 추가

* `Connection`뿐 아니라 `Statement`, `ResultSet`도 DB 자원이므로 닫아야 함.
* 같은 이름 `close()`를 매개변수 타입만 다르게 **오버로딩**함. `PreparedStatement`는 `Statement`의 자식이라 `close(Statement)` 하나로 같이 처리됨.

```java
public static void close(Statement stmt) {
    try {
        if (stmt != null && !stmt.isClosed()) stmt.close();
    } catch (SQLException e) {
        e.printStackTrace();
    }
}

public static void close(ResultSet rset) {
    try {
        if (rset != null && !rset.isClosed()) rset.close();
    } catch (SQLException e) {
        e.printStackTrace();
    }
}
```

> 💡 닫는 순서는 생성의 역순: `close(rset)` → `close(pstmt)` → `close(con)`

---

## 2. Statement vs PreparedStatement

### Statement: 문자열 이어붙이기

* 사용자 입력값을 SQL 문자열에 직접 붙여서 실행함. 작은따옴표 처리가 번거롭고 **SQL Injection에 취약**함.

```java
String query = "SELECT MENU_NAME, MENU_PRICE FROM TBL_MENU WHERE MENU_CODE = '" + menuCode + "'";
rset = stmt.executeQuery(query);
```

### PreparedStatement: 위치 홀더 `?`

* SQL을 **미리 준비(컴파일)** 해두고, `?` 자리에 값만 나중에 채워 넣음.
* `setString(순번, 값)`, `setInt(순번, 값)`. **순번은 1부터 시작**함.
* 쿼리는 `prepareStatement(query)` 때 전달하므로 `executeQuery()`에는 인자를 넣지 않음.

```java
String query = "SELECT MENU_NAME, MENU_PRICE FROM TBL_MENU WHERE MENU_CODE = ?";

pstmt = con.prepareStatement(query);
pstmt.setString(1, menuCode);   // 첫 번째 ? 에 값 바인딩
rset = pstmt.executeQuery();
```

### LIKE 검색

* `?`를 따옴표 안에 넣을 수 없으므로 `CONCAT(?, '%')`으로 패턴을 만듦.

```java
String query = "SELECT * FROM TBL_MENU WHERE MENU_NAME LIKE CONCAT(?, '%')";
pstmt.setString(1, menuName);
```

### 조회 결과를 DTO 리스트로 담기

* `ResultSet`의 한 행 = DTO 객체 하나. `while(rset.next())`로 돌면서 리스트에 담음.

```java
List<MenuDTO> menuList = new ArrayList<>();

while (rset.next()) {
    MenuDTO row = new MenuDTO();
    row.setMenuCode(rset.getInt("MENU_CODE"));
    row.setMenuName(rset.getString("MENU_NAME"));
    row.setMenuPrice(rset.getInt("MENU_PRICE"));
    row.setCategoryCode(rset.getInt("CATEGORY_CODE"));
    row.setOrderableStatus(rset.getString("ORDERABLE_STATUS"));
    menuList.add(row);
}
```

* 날짜 컬럼은 `rset.getDate("HIRE_DATE")`(`java.sql.Date`), 실수는 `rset.getDouble("BONUS")`처럼 타입에 맞는 getter를 사용함.

---

## 3. 쿼리를 XML 파일로 분리

* SQL을 자바 코드에서 분리해 XML에 `key`로 저장하고, `Properties.loadFromXML()`로 읽어옴.
* 쿼리를 바꿀 때 자바 코드를 건드리지 않아도 됨.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <entry key="selectEmpByFamilyName">
        SELECT E.*
          FROM employeedb.EMPLOYEE E
         WHERE E.EMP_NAME LIKE CONCAT(?, '%')
    </entry>
</properties>
```

```java
Properties prop = new Properties();

try (FileInputStream queryStream = new FileInputStream("src/main/java/com/who307/mapper/menu-query.xml")) {
    prop.loadFromXML(queryStream);
    String query = prop.getProperty("selectEmpByFamilyName");

    pstmt = con.prepareStatement(query);
    // ...
}
```

---

## 4. SQL Injection

* 입력값에 SQL 구문을 끼워 넣어 **쿼리의 의미 자체를 바꿔버리는 공격**.

```java
String empId = "210";
String empName = "' OR 1=1 AND EMP_ID = '205";
```

### Statement: 공격 성공 ❌

```java
String query = "SELECT * FROM employeedb.EMPLOYEE WHERE EMP_ID = '" + empId + "' AND EMP_NAME = '" + empName + "'";
```

* 실제로 만들어지는 쿼리:

```sql
SELECT * FROM employeedb.EMPLOYEE
WHERE EMP_ID = '210' AND EMP_NAME = '' OR 1=1 AND EMP_ID = '205'
```

* `AND`가 `OR`보다 먼저 계산되므로 `(EMP_ID='210' AND EMP_NAME='') OR (1=1 AND EMP_ID='205')`가 됨. 이름을 몰라도 **205번 사원으로 "로그인"됨**.

### PreparedStatement: 공격 실패 ✅

```java
String query = "SELECT * FROM employeedb.EMPLOYEE WHERE EMP_ID = ? AND EMP_NAME = ?";
pstmt.setString(1, empId);
pstmt.setString(2, empName);
```

* `setString`으로 넣은 값은 SQL 구문이 아니라 **하나의 문자열 값**으로만 취급됨(따옴표가 escape 처리됨).
* 그래서 이름이 `' OR 1=1 AND EMP_ID = '205`인 사원을 찾는 쿼리가 되고, 결과는 "회원 정보가 없습니다."

> ⚠️ 사용자 입력이 들어가는 쿼리는 **항상 PreparedStatement**를 사용함.

---

## 5. CRUD: executeUpdate()

* `INSERT` / `UPDATE` / `DELETE`(DML)는 `executeUpdate()`로 실행함.
* 반환값은 **영향받은 행(row)의 개수(`int`)**. `result > 0`이면 성공으로 판단함.

```xml
<entry key="insertMenu">
    INSERT INTO TBL_MENU (MENU_NAME, MENU_PRICE, CATEGORY_CODE, ORDERABLE_STATUS)
    VALUES (?, ?, ?, ?)
</entry>
<entry key="updateMenu">
    UPDATE TBL_MENU
       SET MENU_NAME = ?
         , MENU_PRICE = ?
     WHERE MENU_CODE = ?
</entry>
<entry key="deleteMenu">
    DELETE FROM TBL_MENU
     WHERE MENU_CODE = ?
</entry>
```

### INSERT: 입력값을 DTO에 담아 바인딩

```java
MenuDTO newMenu = new MenuDTO();
newMenu.setName(menuName);
newMenu.setPrice(menuPrice);
newMenu.setCategoryCode(categoryCode);
newMenu.setOrderableStatus(orderableStatus);

pstmt = con.prepareStatement(prop.getProperty("insertMenu"));
pstmt.setString(1, newMenu.getName());
pstmt.setInt(2, newMenu.getPrice());
pstmt.setInt(3, newMenu.getCategoryCode());
pstmt.setString(4, newMenu.getOrderableStatus());

int result = pstmt.executeUpdate();

if (result > 0) System.out.println("메뉴 등록 성공!");
else            System.out.println("메뉴 등록 실패!");
```

### UPDATE / DELETE

* `?`의 **순서**에 맞게 바인딩하는 것이 핵심. `UPDATE`는 `SET` 값들 → `WHERE` 조건 순서임.

```java
// UPDATE: SET MENU_NAME = ?(1), MENU_PRICE = ?(2) WHERE MENU_CODE = ?(3)
pstmt.setString(1, changeMenu.getName());
pstmt.setInt(2, changeMenu.getPrice());
pstmt.setInt(3, changeMenu.getCode());

// DELETE: WHERE MENU_CODE = ?(1)
pstmt.setInt(1, deleteMenu.getCode());
```

> 💡 `Scanner`에서 `nextInt()` 다음에 `nextLine()`을 쓰면 남아있는 엔터(`\n`)가 먼저 읽힘. 중간에 `sc.nextLine()`을 한 번 넣어서 버퍼를 비움.

---

## 6. DAO (Data Access Object)

### 문제: 모든 코드가 main에 몰려 있음

* 카테고리 조회 + 사용자 입력 + 값 가공 + 메뉴 등록이 하나의 `main` 안에 섞여 있음.
* 카테고리 코드를 `switch`로 하드코딩(`"한식" → 4`)해서 DB 데이터가 바뀌면 코드도 바꿔야 함.
* 같은 쿼리를 다른 곳에서 쓰려면 코드를 통째로 복사해야 함.

### 해결: DB 접근 코드를 DAO로 분리

* **DAO**: DB에 접근해서 CRUD를 수행하는 메소드들을 모아둔 클래스.
* 쿼리 XML은 **생성자에서 한 번만** 읽어둠.
* `Connection`은 DAO가 만들지 않고 **매개변수로 전달받음**. DAO는 자신이 만든 `PreparedStatement`, `ResultSet`만 닫음.

```java
public class MenuDAO {

    private Properties prop = new Properties();

    public MenuDAO() {
        try (FileInputStream queryStream = new FileInputStream("src/main/java/com/who307/mapper/menu-query.xml")) {
            prop.loadFromXML(queryStream);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /* 1. 하위 카테고리 조회 */
    public List<CategoryDTO> selectAllCategory(Connection con) {
        PreparedStatement pstmt = null;
        ResultSet rset = null;
        List<CategoryDTO> categoryList = new ArrayList<>();

        try {
            pstmt = con.prepareStatement(prop.getProperty("selectAllCategoryList"));
            rset = pstmt.executeQuery();

            while (rset.next()) {
                CategoryDTO categoryDTO = new CategoryDTO();
                categoryDTO.setCategoryCode(rset.getInt("category_code"));
                categoryDTO.setCategoryName(rset.getString("category_name"));
                categoryDTO.setRefCategoryCode(rset.getInt("ref_category_code"));
                categoryList.add(categoryDTO);
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            close(rset);
            close(pstmt);      // con은 호출한 쪽에서 닫음
        }
        return categoryList;
    }

    /* 2. 신규 메뉴 등록 */
    public int insertNewMenu(Connection con, MenuDTO newMenu) {
        PreparedStatement pstmt = null;
        int result = 0;

        try {
            pstmt = con.prepareStatement(prop.getProperty("insertMenu"));
            pstmt.setString(1, newMenu.getMenuName());
            pstmt.setInt(2, newMenu.getMenuPrice());
            pstmt.setInt(3, newMenu.getCategoryCode());
            pstmt.setString(4, newMenu.getOrderableStatus());

            result = pstmt.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            close(pstmt);
        }
        return result;
    }
}
```

### 실행부: 입력과 흐름만 담당

```java
Connection con = getConnection();
MenuDAO menuDAO = new MenuDAO();

// 1. 카테고리 조회 → DB에서 가져온 목록을 보여주므로 하드코딩 switch가 필요 없음
List<CategoryDTO> categoryList = menuDAO.selectAllCategory(con);
for (CategoryDTO c : categoryList) {
    System.out.println(c.getCategoryCode() + " : " + c.getCategoryName());
}

// 2. 입력값 가공
String orderableStatus = answer.equals("예") ? "Y" : "N";

// 3. DAO 호출
MenuDTO newMenu = new MenuDTO(menuName, menuPrice, categoryCode, orderableStatus);
int result = menuDAO.insertNewMenu(con, newMenu);

close(con);
```

| 구분 | 역할 |
| :--- | :--- |
| `Application` (run) | 사용자 입력, 값 가공, 흐름 제어, Connection 생성·종료 |
| `MenuDAO` | 쿼리 실행, 결과를 DTO로 변환 |
| `MenuDTO` / `CategoryDTO` | 계층 간에 데이터를 담아 전달 |
| `menu-query.xml` | SQL 보관 |

> 💡 `CategoryDTO.refCategoryCode`는 NULL이 들어올 수 있어 `Integer`로 선언함. 다만 `rset.getInt()`는 NULL을 `0`으로 반환하므로 NULL을 그대로 받으려면 `rset.getObject("ref_category_code", Integer.class)`를 사용함.

---

## 📝 오늘의 정리

1. 사용한 `ResultSet` → `Statement` → `Connection` 순서로 닫고, `close()`는 오버로딩으로 통일한다.
2. 사용자 입력이 들어가는 쿼리는 `?` 위치 홀더를 쓰는 **PreparedStatement**로 작성한다. SQL Injection을 막을 수 있다.
3. SQL은 **XML 파일로 분리**하고 `Properties.loadFromXML()`로 key를 통해 꺼내 쓴다.
4. `SELECT`는 `executeQuery()`(`ResultSet`), DML은 `executeUpdate()`(영향받은 행 수)로 실행한다.
5. DB 접근 코드는 **DAO**로 분리하고, `Connection`은 호출하는 쪽에서 만들어 넘기고 닫는다.
