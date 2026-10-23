# JDBC Connection · JDBCTemplate · Statement 정리 (TIL)

> 실습 환경: MySQL (`menudb`), Gradle 프로젝트, `mysql-connector-j` 드라이버

## 1. JDBC란?

* **JDBC (Java DataBase Connectivity)**: 자바 애플리케이션에서 DB에 접속해 SQL을 실행할 수 있게 해주는 **표준 API** (`java.sql` 패키지).
* 자바는 **인터페이스**(`Connection`, `Statement`, `ResultSet` 등)만 정의하고, 실제 구현체는 각 DB 벤더가 제공하는 **JDBC 드라이버**가 담당함.
* 그래서 DB를 바꿔도 드라이버와 접속 정보만 바꾸면 같은 코드로 동작함.

| 객체 | 역할 |
| :--- | :--- |
| `DriverManager` | 드라이버를 관리하고 DB 접속 정보로 `Connection`을 만들어 줌 |
| `Connection` | 자바 애플리케이션과 DB 사이의 **연결 통로** |
| `Statement` | 연결 통로를 통해 **SQL을 전달·실행**하는 객체 |
| `ResultSet` | `SELECT` 실행 결과(행 집합)를 담는 객체 |

---

## 2. 프로젝트 설정 (Gradle)

* `build.gradle`의 `dependencies`에 MySQL 드라이버를 추가함. ([mvnrepository](https://mvnrepository.com/artifact/com.mysql/mysql-connector-j)에서 검색)

```groovy
dependencies {
    // Source: https://mvnrepository.com/artifact/com.mysql/mysql-connector-j
    implementation 'com.mysql:mysql-connector-j:9.7.0'
}
```

---

## 3. Connection 직접 생성

* **흐름**: ① 드라이버 클래스 로딩 → ② `DriverManager.getConnection(url, user, password)` → ③ 사용 후 `close()`
* `Class.forName()`은 `ClassNotFoundException`, `getConnection()`은 `SQLException`을 던지므로 예외 처리가 필요함.
* 연결은 DB 자원이므로 `finally`에서 **반드시 닫아줌**.

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

> ⚠️ 접속 정보를 코드에 하드코딩하면 정보가 바뀔 때마다 코드를 수정해야 하고, 비밀번호가 소스에 그대로 노출됨.

---

## 4. 접속 정보 분리 + JDBCTemplate

### properties 파일로 접속 정보 분리

* `key=value` 형태로 접속 정보를 외부 파일에 작성하고, `Properties`로 읽어옴.

```properties
# config/jdbc-config.properties
driver=com.mysql.cj.jdbc.Driver
url=jdbc:mysql://localhost/menudb
user=사용자명
password=비밀번호
```

> 💡 비밀번호가 들어있는 설정 파일은 `.gitignore`에 추가해서 원격 저장소에 올라가지 않게 관리하는 것이 좋음.

### JDBCTemplate

* Connection을 **생성하고 닫는 반복 코드**를 `static` 메소드로 모아둔 유틸 클래스.
* `DriverManager.getConnection(url, prop)`처럼 `Properties`를 통째로 넘기면 `user`, `password` 키를 알아서 읽어감.
* `FileReader`는 try-with-resources로 열어서 자동으로 닫히게 함.

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

### static import로 사용

* `import static`을 쓰면 클래스명 없이 메소드를 바로 호출할 수 있음.

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

---

## 5. Statement로 SELECT 실행

* `con.createStatement()`로 `Statement`를 만들고, `executeQuery(sql)`로 `SELECT`를 실행하면 `ResultSet`이 반환됨.
* `ResultSet`은 커서 방식으로 동작함. `next()`가 다음 행으로 이동하면서 행이 있으면 `true`를 반환하므로 `while`로 순회함.
* `getString("컬럼명")`, `getInt("컬럼명")`처럼 **컬럼 타입에 맞는 getter**로 값을 꺼냄.

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

| 메소드 | 용도 | 반환값 |
| :--- | :--- | :--- |
| `executeQuery(sql)` | `SELECT` | `ResultSet` |
| `executeUpdate(sql)` | `INSERT` / `UPDATE` / `DELETE` | 영향받은 행 수 (`int`) |

> ⚠️ `ResultSet`, `Statement`도 `Connection`과 마찬가지로 DB 자원이므로 사용 후 닫아야 함. → 다음 단계로 `JDBCTemplate`에 `close(Statement)`, `close(ResultSet)` 오버로딩 메소드를 추가해서 정리할 예정.

---

## 📝 오늘의 정리

1. JDBC는 자바 표준 **인터페이스**, 실제 동작은 벤더의 **드라이버**가 담당한다.
2. `DriverManager` → `Connection` → `Statement` → `ResultSet` 순서로 객체를 얻는다.
3. 접속 정보는 **properties 파일로 분리**하고, 연결 생성/해제는 **JDBCTemplate**으로 재사용한다.
4. `ResultSet`은 `next()`로 한 행씩 이동하며 컬럼 타입에 맞는 getter로 값을 꺼낸다.
5. DB 자원은 **생성의 역순으로 반드시 close** 한다.

---

## 느낀 점

* SQL로만 다루던 DB를 자바 코드에서 직접 연결하고 조회해보니, 지금까지 배운 Java와 SQL이 하나로 이어지는 느낌이 들었음.
* 접속 정보를 properties로 분리하는 이유가 코드 수정 편의뿐 아니라 비밀번호 같은 민감 정보를 저장소에 올리지 않기 위해서라는 점을 실감함.
