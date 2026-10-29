# JDBC PreparedStatement · SQL Injection · CRUD · DAO 정리 (TIL)

> 실습 환경: MySQL (`menudb`의 `TBL_MENU`·`TBL_CATEGORY`, `employeedb`의 `EMPLOYEE`), Gradle 프로젝트

## 1. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| chap02-statements | `JDBCTemplate`에 `close` 오버로딩 추가, `Statement` → `PreparedStatement`, XML로 쿼리 분리, SQL Injection 비교 |
| chap03-crud | `executeUpdate()`로 `INSERT` / `UPDATE` / `DELETE` |
| chap04-dao | 모든 코드가 `main`에 몰린 문제 → **DAO**로 DB 접근 코드 분리 |

---

## 2. JDBCTemplate에 close 오버로딩 추가

* `Connection`뿐 아니라 `Statement`, `ResultSet`도 DB 자원이라 다 쓰면 닫아야 함.
* 같은 이름 `close()`를 매개변수 타입만 다르게 **오버로딩**함 → 무엇을 넘기든 `close(그것)` 하나로 닫을 수 있음.
* `PreparedStatement`는 `Statement`의 **자식**이라서 `close(Statement)`에 그대로 들어감 → 따로 만들 필요 없음.

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

* **따라가기**: `close(rset)` → 매개변수가 `ResultSet`인 버전 실행 / `close(pstmt)` → `PreparedStatement`는 `Statement`이기도 하므로 `Statement` 버전 실행 / `close(con)` → 어제 만든 `Connection` 버전 실행. 자바가 넘긴 값의 타입을 보고 알맞은 메소드를 고름.

> 💡 닫는 순서는 생성의 역순: `close(rset)` → `close(pstmt)` → `close(con)`

---

## 3. Statement vs PreparedStatement

### Statement: 문자열 이어붙이기

* 사용자 입력값을 SQL 문자열에 `+`로 **직접 이어 붙여서** 실행함.
  * 작은따옴표(`'`)를 하나하나 맞춰 쓰기 번거롭고 실수하기 쉬움.
  * 입력값이 SQL의 일부가 되어버려서 **SQL Injection 공격에 취약**함(5번에서 자세히).

```java
String query = "SELECT MENU_NAME, MENU_PRICE FROM TBL_MENU WHERE MENU_CODE = '" + menuCode + "'";
rset = stmt.executeQuery(query);
```

* **따라가기**: `menuCode`가 `"3"`이면 → `... WHERE MENU_CODE = '3'`이라는 문장이 만들어져 실행됨.

### PreparedStatement: 위치 홀더 `?`

* **한 줄 비유**: **빈칸이 있는 신청서 양식**임. 양식(SQL)은 미리 인쇄해두고, 빈칸(`?`)에는 나중에 **값만** 적어 넣음. 빈칸에 뭘 적든 양식의 문장 자체는 바뀌지 않음.
* SQL을 **미리 준비(컴파일)** 해 두고, `?` 자리에 값만 나중에 채움.
* `setString(순번, 값)`, `setInt(순번, 값)` → **몇 번째 `?`** 에 어떤 타입의 값을 넣을지 정함. **순번은 1부터** 시작함(배열처럼 0이 아님).
* SQL은 `prepareStatement(query)`할 때 이미 넘겼으므로, `executeQuery()`에는 **아무것도 넣지 않음**.

```java
String query = "SELECT MENU_NAME, MENU_PRICE FROM TBL_MENU WHERE MENU_CODE = ?";

pstmt = con.prepareStatement(query);
pstmt.setString(1, menuCode);   // 첫 번째 ? 에 값 바인딩
rset = pstmt.executeQuery();
```

* **따라가기**
  1. `prepareStatement(query)` → `?`가 하나 있는 SQL 양식을 DB에 보낼 준비.
  2. `setString(1, "3")` → 1번째 `?`에 `'3'`을 채움(따옴표는 알아서 처리됨).
  3. `executeQuery()` → 실행 → 3번 메뉴의 이름·가격이 담긴 `ResultSet`.

### LIKE 검색

* `?`는 **값 하나 전체**를 대신하는 자리라서 `'?%'`처럼 따옴표 안에 넣을 수 없음 → `CONCAT(?, '%')`로 "입력값 + %"를 SQL 쪽에서 이어 붙임.

```java
String query = "SELECT * FROM TBL_MENU WHERE MENU_NAME LIKE CONCAT(?, '%')";
pstmt.setString(1, menuName);
```

* **따라가기**: `menuName`이 `"김치"`면 → `CONCAT('김치', '%')` → `'김치%'` → "김치"로 시작하는 메뉴(김치찌개, 김치볶음밥…) 조회.

### 조회 결과를 DTO 리스트로 담기

* `ResultSet`의 **한 줄 = DTO 객체 하나**. `while (rset.next())`로 줄마다 DTO를 만들어 값을 채우고 리스트에 넣음. 시험지 한 장 한 장을 학생 카드에 옮겨 적어 파일철에 꽂는 것과 같음.

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

* **따라가기** (결과가 2줄이라면)
  1. `next()` → 1번째 줄 → 빈 `MenuDTO` 생성 → 코드·이름·가격·카테고리·주문가능 값을 꺼내 setter로 채움 → 리스트에 추가 → `menuList` 크기 1.
  2. `next()` → 2번째 줄 → 새 `MenuDTO`로 같은 작업 → 크기 2.
  3. `next()` → `false` → 끝. 이제 `menuList`만 있으면 DB 없이도 메뉴 2개 정보를 다룰 수 있음.
* 날짜 열은 `rset.getDate("HIRE_DATE")`(`java.sql.Date` 타입), 소수 열은 `rset.getDouble("BONUS")`처럼 타입에 맞는 getter를 씀.

---

## 4. 쿼리를 XML 파일로 분리

* SQL을 자바 코드에서 빼내 **XML 파일에 `key`(이름표)를 붙여** 저장하고, `Properties.loadFromXML()`로 읽어옴.
* **장점**: 쿼리를 고칠 때 자바 코드를 건드리지 않아도 됨. 긴 SQL이 자바 코드 사이에 섞이지 않아 읽기 쉬움.

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

* **따라가기**
  1. XML의 `<entry key="selectEmpByFamilyName">` 안에 SQL 문장이 들어 있음. `<!DOCTYPE ...>`은 "이 파일은 Properties용 XML 형식"이라는 표시.
  2. `loadFromXML(queryStream)` → XML을 읽어 `key → SQL` 쌍으로 `prop`에 채움.
  3. `getProperty("selectEmpByFamilyName")` → 그 SQL 문장을 꺼냄.
  4. 꺼낸 SQL로 `PreparedStatement`를 만들고 `?`에 성씨(예: `"김"`)를 넣으면 → 이름이 "김"으로 시작하는 사원 조회.

---

## 5. SQL Injection

* **SQL Injection**: 입력칸에 **SQL 문장 조각**을 몰래 넣어서 **쿼리의 의미 자체를 바꿔버리는 공격**. 신청서 이름 칸에 "홍길동, 그리고 금고 문 열어 주세요"라고 적었더니 직원이 그 문장까지 명령으로 알아듣는 것과 같음.

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

* **따라가기**
  1. 입력값의 첫 `'`가 이름 칸의 따옴표를 **닫아버림** → `EMP_NAME = ''`
  2. 그 뒤의 `OR 1=1 AND EMP_ID = '205`가 **SQL 문장의 일부**가 되어버림.
  3. `AND`가 `OR`보다 먼저 계산되므로 → `(EMP_ID='210' AND EMP_NAME='') OR (1=1 AND EMP_ID='205')`
  4. 앞부분은 이름이 빈 210번 사원이 없어 거짓이지만, 뒷부분 `1=1`(항상 참) `AND EMP_ID='205'` → **205번 사원이 조회됨**.
  5. 비밀번호(이름)를 몰라도 **205번 사원으로 "로그인"** 되는 셈.

### PreparedStatement: 공격 실패 ✅

```java
String query = "SELECT * FROM employeedb.EMPLOYEE WHERE EMP_ID = ? AND EMP_NAME = ?";
pstmt.setString(1, empId);
pstmt.setString(2, empName);
```

* `setString`으로 넣은 값은 SQL 명령이 아니라 **그냥 글자 하나의 덩어리**로만 취급됨(안의 따옴표도 escape 처리되어 "진짜 따옴표 글자"가 됨).
* 그래서 "이름이 `' OR 1=1 AND EMP_ID = '205`라는 **이상한 글자**인 사원"을 찾는 쿼리가 되고 → 그런 사원은 없으니 결과는 "회원 정보가 없습니다."

> ⚠️ 사용자 입력이 들어가는 쿼리는 **항상 PreparedStatement**를 사용함.

---

## 6. CRUD: executeUpdate()

* **CRUD**: Create(추가), Read(조회), Update(수정), Delete(삭제) — 데이터를 다루는 네 가지 기본 동작.
* `INSERT` / `UPDATE` / `DELETE`(DML)는 `executeUpdate()`로 실행함.
* 돌려주는 값은 **영향받은 행(줄)의 개수(`int`)**. 1줄이 추가·수정·삭제됐으면 `1`, 해당하는 줄이 없었으면 `0` → `result > 0`이면 성공으로 판단함.

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
newMenu.setMenuName(menuName);
newMenu.setMenuPrice(menuPrice);
newMenu.setCategoryCode(categoryCode);
newMenu.setOrderableStatus(orderableStatus);

pstmt = con.prepareStatement(prop.getProperty("insertMenu"));
pstmt.setString(1, newMenu.getMenuName());
pstmt.setInt(2, newMenu.getMenuPrice());
pstmt.setInt(3, newMenu.getCategoryCode());
pstmt.setString(4, newMenu.getOrderableStatus());

int result = pstmt.executeUpdate();

if (result > 0) System.out.println("메뉴 등록 성공!");
else            System.out.println("메뉴 등록 실패!");
```

* **따라가기** (입력: 이름 "마라탕", 가격 12000, 카테고리 4, 주문가능 "Y")
  1. 사용자 입력값 4개를 DTO 하나에 담음.
  2. XML에서 `insertMenu` SQL을 꺼내 `PreparedStatement`를 만듦.
  3. `?` 4개를 순서대로 채움 → `VALUES ('마라탕', 12000, 4, 'Y')`
  4. `executeUpdate()` → 1줄 추가 → `result = 1` → `메뉴 등록 성공!`

### UPDATE / DELETE

* `?`의 **순서**대로 값을 맞추는 게 핵심. `UPDATE`는 `SET`의 값들이 먼저, `WHERE` 조건이 마지막임.

```java
// UPDATE: SET MENU_NAME = ?(1), MENU_PRICE = ?(2) WHERE MENU_CODE = ?(3)
pstmt.setString(1, changeMenu.getMenuName());
pstmt.setInt(2, changeMenu.getMenuPrice());
pstmt.setInt(3, changeMenu.getMenuCode());

// DELETE: WHERE MENU_CODE = ?(1)
pstmt.setInt(1, deleteMenu.getMenuCode());
```

* **따라가기**
  * UPDATE (25번 메뉴를 "로제마라탕", 13000원으로) → 1번 `?` ← 이름, 2번 `?` ← 가격, 3번 `?` ← 25 → 25번 줄만 바뀌고 `result = 1`. 25번이 없으면 `0`.
  * DELETE (25번) → 1번 `?` ← 25 → 25번 줄 삭제.

> 💡 `Scanner`에서 `nextInt()` 다음에 `nextLine()`을 쓰면 남아있는 엔터(`\n`)가 먼저 읽힘. 중간에 `sc.nextLine()`을 한 번 넣어서 버퍼를 비움.

---

## 7. DAO (Data Access Object)

### 문제: 모든 코드가 main에 몰려 있음

* 카테고리 조회 + 사용자 입력 + 값 가공 + 메뉴 등록이 하나의 `main` 안에 뒤섞여 있음 → 길고 읽기 어려움.
* 카테고리 코드를 `switch`로 하드코딩(`"한식" → 4`)해서, DB의 카테고리가 바뀌면 **자바 코드도 고쳐야** 함.
* 같은 쿼리를 다른 곳에서 쓰려면 코드를 통째로 **복사**해야 함.

### 해결: DB 접근 코드를 DAO로 분리

* **DAO**: DB에 접근해서 CRUD를 하는 메소드들만 모아둔 **DB 전담 직원** 클래스. `main`은 "이 메뉴 등록해 줘"라고 시키기만 하고, SQL을 어떻게 실행하는지는 DAO가 알아서 함.
* **규칙**
  * 쿼리 XML은 DAO **생성자에서 한 번만** 읽어둠 → 메소드마다 파일을 다시 읽지 않음.
  * `Connection`은 DAO가 만들지 않고 **매개변수로 받음** → 여러 DAO 메소드를 같은 연결로 이어서 쓸 수 있음.
  * DAO는 **자기가 만든** `PreparedStatement`, `ResultSet`만 닫음. `Connection`은 만든 쪽(호출한 쪽)이 닫음.

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
                categoryDTO.setRefCategoryCode(rset.getObject("ref_category_code", Integer.class));
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

* **따라가기**
  * `new MenuDAO()` → 생성자에서 쿼리 XML을 읽어 `prop`에 저장해 둠.
  * `selectAllCategory(con)` → 받은 연결로 카테고리 조회 SQL 실행 → 줄마다 `CategoryDTO`를 만들어 리스트에 담음 → `rset`, `pstmt`만 닫고 리스트를 돌려줌.
  * `insertNewMenu(con, newMenu)` → DTO의 값 4개를 `?`에 채워 실행 → 영향받은 줄 수(성공하면 1)를 돌려줌.

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

* **따라가기**
  1. 연결을 하나 만들고 DAO를 준비.
  2. DAO에게 카테고리 목록을 받아 `4 : 한식`, `5 : 중식` … 처럼 화면에 보여줌 → 사용자는 번호를 보고 고름. DB에 카테고리가 추가돼도 자바 코드는 그대로.
  3. 사용자가 "예"라고 답했으면 `"Y"`, 아니면 `"N"`으로 바꿈(DB에 맞는 값으로 가공).
  4. 입력값을 DTO에 담아 DAO에게 등록을 맡김 → 결과 숫자를 받음.
  5. 마지막에 **연결을 만든 쪽(실행부)** 이 `close(con)`.

| 구분 | 역할 |
| :--- | :--- |
| `Application` (run) | 사용자 입력, 값 가공, 흐름 제어, Connection 생성·종료 |
| `MenuDAO` | 쿼리 실행, 결과를 DTO로 변환 |
| `MenuDTO` / `CategoryDTO` | 계층 간에 데이터를 담아 전달 |
| `menu-query.xml` | SQL 보관 |

> 💡 `CategoryDTO.refCategoryCode`는 NULL이 들어올 수 있어 `Integer`로 선언함. `rset.getInt()`는 NULL을 `0`으로 바꿔 돌려주기 때문에, 위 코드처럼 `rset.getObject("ref_category_code", Integer.class)`로 받아 최상위 카테고리의 NULL을 그대로 유지함.

---

## 📝 오늘의 정리

1. 사용한 `ResultSet` → `Statement` → `Connection` 순서로 닫고, `close()`는 오버로딩으로 통일함
2. 사용자 입력이 들어가는 쿼리는 `?` 위치 홀더를 쓰는 **PreparedStatement**로 작성함. SQL Injection을 막을 수 있음
3. SQL은 **XML 파일로 분리**하고 `Properties.loadFromXML()`로 key를 통해 꺼내 씀
4. `SELECT`는 `executeQuery()`(`ResultSet`), DML은 `executeUpdate()`(영향받은 행 수)로 실행함
5. DB 접근 코드는 **DAO**로 분리하고, `Connection`은 호출하는 쪽에서 만들어 넘기고 닫음

---

## 느낀 점

* 같은 입력값으로 Statement와 PreparedStatement를 비교해보니 SQL Injection이 얼마나 쉽게 일어나는지 체감했고, 사용자 입력이 들어가는 쿼리는 무조건 PreparedStatement를 써야겠다고 느낌
* `main` 하나에 모든 코드를 넣었을 때보다 DAO로 나누니 각 클래스의 역할이 분명해졌고, 다른 기능을 추가할 때도 DAO 메소드만 늘리면 된다는 점이 좋았음
