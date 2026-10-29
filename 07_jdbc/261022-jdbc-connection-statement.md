# JDBC Connection · JDBCTemplate · Statement 정리 (TIL)

> 실습 환경: MySQL (`menudb`), Gradle 프로젝트, `mysql-connector-j` 드라이버

## 1. JDBC란?

* **한 줄 비유**: 자바 프로그램과 DB는 **서로 다른 말**을 씀. JDBC는 둘 사이의 **통역사 규칙**임. 자바는 "통역사는 이렇게 일해야 한다"는 규칙(인터페이스)만 정하고, 실제 통역사(드라이버)는 MySQL, Oracle 같은 DB 회사가 각자 보내줌.
* **JDBC (Java DataBase Connectivity)**: 자바에서 DB에 접속해 SQL을 실행하게 해주는 **표준 API** (`java.sql` 패키지).
* 자바는 **인터페이스**(`Connection`, `Statement`, `ResultSet` 등)만 정의하고, 진짜 동작은 DB 회사가 만든 **JDBC 드라이버**가 담당함.
* 그래서 DB를 MySQL에서 Oracle로 바꿔도 **드라이버와 접속 정보만** 바꾸면 자바 코드는 거의 그대로 동작함.

| 객체 | 역할 | 비유 |
| :--- | :--- | :--- |
| `DriverManager` | 드라이버를 관리하고 DB 접속 정보로 `Connection`을 만들어 줌 | 전화 교환원 |
| `Connection` | 자바 애플리케이션과 DB 사이의 **연결 통로** | 연결된 전화선 |
| `Statement` | 연결 통로를 통해 **SQL을 전달·실행**하는 객체 | 전화로 하는 말 |
| `ResultSet` | `SELECT` 실행 결과(행 집합)를 담는 객체 | 받아 적은 대답 메모 |

---

## 2. 프로젝트 설정 (Gradle)

* **Gradle**: 프로젝트에 필요한 외부 라이브러리를 **자동으로 내려받아 연결**해 주는 도구. 장보기 목록(`build.gradle`)에 적어두면 대신 장을 봐 옴.
* `build.gradle`의 `dependencies`에 MySQL 드라이버를 추가함. ([mvnrepository](https://mvnrepository.com/artifact/com.mysql/mysql-connector-j)에서 검색)

```groovy
dependencies {
    // Source: https://mvnrepository.com/artifact/com.mysql/mysql-connector-j
    implementation 'com.mysql:mysql-connector-j:9.7.0'
}
```

* **한 줄씩 보기**: `implementation '그룹:이름:버전'` → "MySQL 회사(`com.mysql`)의 `mysql-connector-j` 9.7.0 버전을 가져와서 내 프로젝트에서 쓸게."

---

## 3. Connection 직접 만들기

* **흐름**: ① 드라이버 클래스 불러오기 → ② `DriverManager.getConnection(주소, 아이디, 비밀번호)`로 연결 → ③ 다 쓰면 `close()`
* `Class.forName()`은 `ClassNotFoundException`(드라이버를 못 찾음), `getConnection()`은 `SQLException`(DB 접속 실패)을 던질 수 있어서 **예외 처리**가 필요함.
* 연결은 DB 쪽 자원(동시에 받을 수 있는 연결 수가 정해져 있음)이라 `finally`에서 **반드시 닫음**. 통화가 끝나면 전화를 끊어야 다른 사람이 전화할 수 있는 것과 같음.

```java
Connection con = null;

try {
    // 1. 사용할 JDBC Driver 클래스 로딩
    Class.forName("com.mysql.cj.jdbc.Driver");

    // 2. DriverManager.getConnection(url, user, password)
    con = DriverManager.getConnection("jdbc:mysql://localhost/menudb", "user", "password");

    System.out.println(con);

} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (SQLException e) {
    e.printStackTrace();
} finally {
    // 3. 자원 반납
    if (con != null) {
        try {
            if (!con.isClosed()) con.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

* **한 줄씩 보기**
  * `Class.forName("com.mysql.cj.jdbc.Driver")` → MySQL 드라이버 클래스를 메모리에 불러옴 → 드라이버가 스스로 `DriverManager`에 "저 왔어요" 하고 등록함.
  * `"jdbc:mysql://localhost/menudb"` → 접속 주소. "JDBC로, MySQL에, 내 컴퓨터(`localhost`)의, `menudb` 데이터베이스".
  * `getConnection(주소, "user", "password")` → 아이디·비밀번호로 접속 → 성공하면 `Connection` 객체를 돌려줌.
  * `System.out.println(con)` → 연결 객체 정보가 찍힘(예: `com.mysql.cj.jdbc.ConnectionImpl@1a2b3c`) → 연결 성공 확인.
  * `finally` → 연결이 만들어졌고(`null` 아님) 아직 안 닫혔으면 `close()`. `close()`도 `SQLException`을 던질 수 있어서 `try-catch`로 감쌈.

> ⚠️ 접속 정보를 코드에 하드코딩(직접 적어 넣기)하면 정보가 바뀔 때마다 코드를 고쳐야 하고, 비밀번호가 소스 코드에 그대로 노출됨.

---

## 4. 접속 정보 분리 + JDBCTemplate

### properties 파일로 접속 정보 분리

* 접속 정보를 `key=value` 모양으로 **별도 파일**에 적고, `Properties`로 읽어옴. 비밀번호를 공책 대신 **금고(설정 파일)** 에 따로 넣어두는 것.

```properties
# config/jdbc-config.properties
driver=com.mysql.cj.jdbc.Driver
url=jdbc:mysql://localhost/menudb
user=사용자명
password=비밀번호
```

> 💡 비밀번호가 들어있는 설정 파일은 `.gitignore`에 추가해서 원격 저장소에 올라가지 않게 관리하는 것이 좋음.

### JDBCTemplate

* **JDBCTemplate**: Connection을 **만들고 닫는 반복 코드**를 `static` 메소드로 모아둔 **도우미 클래스**. 매번 위의 긴 코드를 쓰지 않고 `getConnection()` 한 줄로 끝냄.
* `DriverManager.getConnection(url, prop)`처럼 `Properties`를 통째로 넘기면, 그 안의 `user`, `password` 키를 **알아서** 꺼내 씀.
* 설정 파일을 읽는 `FileReader`는 try-with-resources로 열어서 자동으로 닫히게 함.

```java
public class JDBCTemplate {

    public static Connection getConnection() {
        Connection con = null;
        Properties prop = new Properties();

        try (FileReader reader = new FileReader("src/main/java/com/who307/config/jdbc-config.properties")) {
            prop.load(reader);

            String driver = prop.getProperty("driver");
            String url = prop.getProperty("url");

            Class.forName(driver);
            con = DriverManager.getConnection(url, prop);   // user, password는 prop에서 읽음

        } catch (IOException | ClassNotFoundException | SQLException e) {
            e.printStackTrace();
        }
        return con;
    }

    public static void close(Connection con) {
        try {
            if (con != null && !con.isClosed()) con.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

* **따라가기: `getConnection()`**
  1. 설정 파일을 열어 `prop.load(reader)` → `driver`, `url`, `user`, `password` 네 쌍이 `prop`에 채워짐.
  2. `driver`, `url` 값을 꺼냄.
  3. `Class.forName(driver)` → 드라이버 불러오기.
  4. `getConnection(url, prop)` → `prop` 안의 `user`, `password`로 접속.
  5. 블록이 끝나며 `FileReader` 자동 닫힘 → 연결 객체를 돌려줌. (실패하면 오류를 찍고 `null`을 돌려줌)
  * `catch (IOException | ClassNotFoundException | SQLException e)` → 파일 못 읽음, 드라이버 없음, 접속 실패 세 가지를 한 번에 잡음(multi-catch).
* **`close(con)`**: `null`이 아니고 안 닫혔을 때만 닫음 → 부르는 쪽에서는 걱정 없이 `close(con)`만 쓰면 됨.

### static import로 사용

* `import static`을 쓰면 `JDBCTemplate.getConnection()` 대신 **클래스 이름 없이** `getConnection()`으로 바로 부를 수 있음.

```java
import static com.who307.common.JDBCTemplate.close;
import static com.who307.common.JDBCTemplate.getConnection;

public class Application {
    public static void main(String[] args) {
        Connection con = getConnection();
        System.out.println("con = " + con);
        close(con);
    }
}
```

* **따라가기**: 연결 얻기 → `con = com.mysql.cj.jdbc.ConnectionImpl@...` 출력 → 닫기. 세 줄로 끝남.

---

## 5. Statement로 SELECT 실행하기

* **순서**
  1. `con.createStatement()` → SQL을 실어 나를 `Statement`를 만듦.
  2. `stmt.executeQuery("SELECT ...")` → SQL을 DB에 보내고, 결과표를 담은 `ResultSet`을 받음.
  3. `ResultSet`을 한 줄씩 읽음.
* **ResultSet은 커서(손가락) 방식**: 처음에 손가락이 **첫 줄 바로 위**에 있음. `next()`를 부를 때마다 손가락이 **한 줄 아래로** 내려가고, 줄이 있으면 `true`, 더 없으면 `false`를 돌려줌 → `while (rset.next())`로 끝까지 읽음.
* **값 꺼내기**: 지금 손가락이 가리키는 줄에서 `getString("컬럼명")`, `getInt("컬럼명")`처럼 **열 타입에 맞는 getter**로 꺼냄.

```java
Statement stmt = null;
ResultSet rset = null;

Connection con = getConnection();

try {
    stmt = con.createStatement();
    rset = stmt.executeQuery("SELECT MENU_NAME, MENU_PRICE FROM TBL_MENU");

    while (rset.next()) {
        System.out.println(rset.getString("MENU_NAME") + ", "
                + rset.getInt("MENU_PRICE"));
    }
} catch (SQLException e) {
    e.printStackTrace();
} finally {
    // 사용한 자원은 생성의 역순으로 닫음: ResultSet → Statement → Connection
}
```

* **따라가기** (표에 `열무김치라떼 4500`, `우럭스무디 5000`, … 이 있다고 가정)
  1. `executeQuery(...)` → 메뉴 이름·가격 두 열짜리 결과표를 받음. 손가락은 첫 줄 위.
  2. `next()` → 1번째 줄로 이동, `true` → `열무김치라떼, 4500` 출력.
  3. `next()` → 2번째 줄, `true` → `우럭스무디, 5000` 출력.
  4. … 마지막 줄 다음 `next()` → `false` → 반복 끝.
  5. `finally`에서 만든 순서의 **반대로** 닫아야 함: `ResultSet` → `Statement` → `Connection`. (책을 쌓았으면 위에서부터 치우는 것과 같음)

| 메소드 | 용도 | 반환값 |
| :--- | :--- | :--- |
| `executeQuery(sql)` | `SELECT` | `ResultSet` |
| `executeUpdate(sql)` | `INSERT` / `UPDATE` / `DELETE` | 영향받은 행 수 (`int`) |

> ⚠️ `ResultSet`, `Statement`도 `Connection`과 마찬가지로 DB 자원이므로 사용 후 닫아야 함. → 다음 단계로 `JDBCTemplate`에 `close(Statement)`, `close(ResultSet)` 오버로딩 메소드를 추가해서 정리할 예정.

---

## 📝 오늘의 정리

1. JDBC는 자바 표준 **인터페이스**, 실제 동작은 벤더의 **드라이버**가 담당함
2. `DriverManager` → `Connection` → `Statement` → `ResultSet` 순서로 객체를 얻음
3. 접속 정보는 **properties 파일로 분리**하고, 연결 생성/해제는 **JDBCTemplate**으로 재사용함
4. `ResultSet`은 `next()`로 한 행씩 이동하며 컬럼 타입에 맞는 getter로 값을 꺼냄
5. DB 자원은 **생성의 역순으로 반드시 close** 함

---

## 느낀 점

* SQL로만 다루던 DB를 자바 코드에서 직접 연결하고 조회해보니, 지금까지 배운 Java와 SQL이 하나로 이어지는 느낌이 들었음
* 접속 정보를 properties로 분리하는 이유가 코드 수정 편의뿐 아니라 비밀번호 같은 민감 정보를 저장소에 올리지 않기 위해서라는 점을 실감함
