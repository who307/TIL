# Java 예외처리 · 입출력(IO) · Enum 정리 (TIL)

## 1. 예외처리 (Exception Handling)

* **개념**: 프로그램 실행 중 발생하는 문제를 감지하고, 그 문제를 안내·기록·복구할 방법을 정하는 것임을 학습함. `Integer.parseInt("abc")`처럼 처리되지 않은 예외가 발생하면 프로그램이 그 자리에서 비정상 종료됨을 확인함.


* **`throw` vs `throws`**

| 키워드 | 위치 | 의미 |
| :--- | :--- | :--- |
| `throw` | 메소드 내부 | 현재 위치에서 예외 객체를 직접 던짐 |
| `throws` | 메소드 선언부 | 이 메소드가 처리하지 않은 예외를 호출한 곳으로 위임함 |

```java
public void checkEnoughMoney(int price, int money) throws Exception {
    if (money >= price) {
        System.out.println("상품을 구입하기 위한 돈이 충분합니다.");
    } else {
        throw new Exception("돈이 부족합니다!!");   // 예외 발생
    }
    // throw가 실행되면 실행 흐름이 끝나므로 이 줄까지 내려오지 않음
    System.out.println("즐거운 쇼핑 하세요~");
}
```

* **`main`에서 `throws`로 계속 위임하면** 결국 JVM까지 예외가 넘어가 프로그램이 종료되므로, `"프로그램을 종료합니다."`가 출력되지 않음을 확인함.

---

## 2. `try-catch-finally`

* **`try`**: 예외가 발생할 수 있는 코드를 감쌈.
* **`catch`**: 전달된 예외 타입과 일치할 때 실행되며, 처리 후 프로그램은 정상적으로 다음 줄을 이어서 실행함.
* **`finally`**: 예외 발생 여부와 관계없이 항상 실행되므로, 반드시 해야 하는 마무리 작업(자원 반납 등)을 작성함.

```java
try {
    et.checkEnoughMoney(1000, 500);
} catch (Exception e) {
    System.out.println(e.getMessage());   // 예외 메시지
    e.printStackTrace();                  // 예외가 전달된 호출 경로 출력
}
```

* **`printStackTrace()`**: 예외가 어디서 발생해 어떤 경로로 전달됐는지 확인할 수 있음. 실제 서비스에서는 보통 로깅 도구로 필요한 정보를 기록함.


* **Checked / Unchecked Exception**

| 구분 | 컴파일러 검사 | 예시 |
| :--- | :--- | :--- |
| Checked Exception | `try-catch` 또는 `throws` 선언을 강제함 | `Exception`, `IOException`, `FileNotFoundException` |
| Unchecked Exception | 처리를 강제하지 않으며 주로 실행 중 발견됨 | `NumberFormatException`, `NoSuchElementException` |

```java
Deque<Integer> stack = new ArrayDeque<>();
try {
    stack.pop();                          // 빈 스택 → NoSuchElementException
} catch (NoSuchElementException e) {
    System.out.println("빈 스택에서는 값을 꺼낼 수 없습니다.");
}
```

---

## 3. 사용자 정의 예외 (Custom Exception)

* **개념**: `Exception`을 상속받아 상황에 맞는 이름의 예외 클래스를 직접 만들 수 있음. 생성자에서 `super(message)`로 메시지를 부모에게 전달하면 `getMessage()`로 꺼낼 수 있음을 학습함.


* **예외 계층 설계**: 음수 관련 예외를 `NegativeException`으로 묶고, 그 아래에 가격/금액 예외를 두어 상속 구조로 분류함.

```
Exception
 ├─ NegativeException
 │   ├─ PriceNegativeException   (상품 가격이 음수)
 │   └─ MoneyNegativeException   (가진 돈이 음수)
 └─ NotEnoughMoneyException      (돈이 부족)
```

```java
public class NegativeException extends Exception {
    public NegativeException(String message) {
        super(message);
    }
}

public class PriceNegativeException extends NegativeException {
    public PriceNegativeException(String message) {
        super(message);
    }
}
```

* **multi-catch와 catch 순서**: `|`로 여러 예외를 한 `catch`에서 처리할 수 있음. `Exception`은 모든 예외의 부모이므로 반드시 **마지막**에 작성해야 함 (앞에 두면 아래 `catch`에 도달할 수 없어 컴파일 에러).

```java
try {
    et.checkEnoughMoney(30000, 5000);
} catch (PriceNegativeException | MoneyNegativeException e) {
    System.out.println(e.getClass().getSimpleName() + "발생!!" + e.getMessage());
} catch (NotEnoughMoneyException e) {
    System.out.println(e.getClass().getSimpleName() + "발생!!" + e.getMessage());
} catch (Exception e) {
    e.printStackTrace();                  // 앞에서 분류하지 못한 예외 처리
} finally {
    System.out.println("구매 가능 여부 확인을 마칩니다.");
}
```

---

## 4. 오버라이딩 시 예외 선언 규칙

* 부모 메소드가 `throws IOException`을 선언했을 때, 자식이 오버라이딩하며 선언할 수 있는 예외 범위를 확인함.

| 자식 메소드의 `throws` | 허용 여부 |
| :--- | :--- |
| 선언하지 않음 | O |
| 같은 예외 (`IOException`) | O |
| 더 구체적인 하위 예외 (`FileNotFoundException`) | O |
| 더 넓은 상위 예외 (`Exception`) | X (컴파일 에러) |

* **이유**: 부모 타입으로 자식 객체를 다루는 코드는 부모의 `throws`만 보고 예외 처리를 작성하므로, 자식이 더 넓은 예외를 던지면 그 처리를 벗어나게 됨.

---

## 5. `File` 클래스

* **개념**: 파일 자체(경로, 크기, 생성/삭제)를 다루는 클래스이며, 파일 내용을 읽고 쓰는 것은 스트림이 담당함.

| 메소드 | 기능 |
| :--- | :--- |
| `createNewFile()` | 새 파일 생성 시 `true`, 이미 존재하면 `false` (`IOException` 처리 필요) |
| `length()` | 파일 크기 (byte) |
| `getPath()` / `getAbsolutePath()` | 경로 / 절대 경로 |
| `delete()` | 파일 삭제 성공 시 `true` |

---

## 6. 바이트 스트림 — `FileInputStream` / `FileOutputStream`

* **개념**: 데이터를 **1바이트 단위**로 읽고 쓰는 스트림으로, 이미지·영상·텍스트 등 모든 파일을 다룰 수 있음. 다만 텍스트를 글자 단위로 처리할 때는 `Reader`/`Writer` 계열이 알맞음.


* **`read()`**: 1바이트씩 읽어 `0~255` 정수로 반환하고, 더 이상 읽을 데이터가 없으면 `-1`을 반환함.


* **자원 반납**: 사용이 끝난 스트림은 반드시 `close()`로 닫아야 하므로, 기존 방식은 `finally`에서 `null` 체크 후 다시 `try-catch`로 감싸야 해서 코드가 길어짐.

```java
FileInputStream fin = null;
try {
    fin = new FileInputStream("testInputStream.txt");
    int value;
    while ((value = fin.read()) != -1) {
        System.out.println((char) value);
    }
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fin != null) {
        try { fin.close(); } catch (IOException e) { e.printStackTrace(); }
    }
}
```

* **try-with-resources**: `try ( )` 괄호 안에서 스트림을 생성하면 블록이 끝날 때 자동으로 `close()`가 호출됨.


* **덮어쓰기 / 이어쓰기**: `new FileOutputStream(path)`는 기본이 덮어쓰기이며, 이어쓰려면 `new FileOutputStream(path, true)`처럼 append 옵션을 줌.

```java
try (FileOutputStream fout = new FileOutputStream("testOutputStream.txt")) {
    fout.write(97);                                   // 'a'
    fout.write(new byte[] {98, 99, 100, 101, 10, 102}); // bcde + 개행(10) + f
} catch (IOException e) {
    e.printStackTrace();
}
```

---

## 7. 문자 스트림 — `FileReader` / `FileWriter`

* **개념**: 바이트가 아닌 **문자 단위**로 텍스트를 읽고 쓰는 스트림으로, 한글처럼 여러 바이트로 이루어진 글자도 깨지지 않고 한 글자씩 처리됨.


* **문자셋 주의**: `FileReader`/`FileWriter`는 실행 환경의 기본 문자셋을 사용하므로, 문자셋을 명시하려면 `InputStreamReader`/`OutputStreamWriter`에 `StandardCharsets.UTF_8`을 지정함.

```java
try (FileWriter fw = new FileWriter("testWriter.txt")) {
    fw.write(97);                                   // 'a'
    fw.write('A');
    fw.write(new char[] {'a', 'p', 'p', 'l', 'e'});
    fw.write("안녕하세요 반갑습니다");                 // 문자열도 바로 쓰기 가능
}

try (Writer writer = new OutputStreamWriter(
        new FileOutputStream("testWriterUtf8.txt"), StandardCharsets.UTF_8)) {
    writer.write("UTF-8로 저장한 한글 텍스트");
}
```

| 구분 | 바이트 스트림 | 문자 스트림 |
| :--- | :--- | :--- |
| 단위 | 1 byte | 문자 (char) |
| 입력 | `InputStream` (`FileInputStream`) | `Reader` (`FileReader`) |
| 출력 | `OutputStream` (`FileOutputStream`) | `Writer` (`FileWriter`) |
| 용도 | 이미지·영상 등 모든 파일 | 텍스트 파일 |

---

## 8. 보조 스트림 (Filter Stream)

* **개념**: 혼자서는 사용할 수 없고, 기본 스트림을 **감싸서** 기능을 추가하는 스트림임을 학습함.


* **`BufferedWriter` / `BufferedReader`**: 데이터를 모아두는 공간(버퍼)을 거쳐 입출력 횟수를 줄여 성능을 높임.
  * `flush()`: 버퍼에 남은 내용을 즉시 파일로 밀어냄. try-with-resources가 끝나며 `close()`가 호출될 때 `flush()`도 함께 처리됨.
  * `readLine()`: 한 줄씩 읽어 `String`으로 반환하고, 더 읽을 줄이 없으면 `null`을 반환함.

```java
try (BufferedWriter bw = new BufferedWriter(new FileWriter("testBuffered.txt"))) {
    bw.write("안녕");
    bw.write("반가워");
}   // close() 시 flush()도 처리됨

try (BufferedReader br = new BufferedReader(new FileReader("testBuffered.txt"))) {
    String temp;
    while ((temp = br.readLine()) != null) {
        System.out.println(temp);
    }
}
```

* **형변환 보조 스트림 (`InputStreamReader` / `OutputStreamWriter`)**: 바이트 기반 스트림을 문자 기반 스트림으로 연결함. 키보드 입력(`System.in`)은 바이트 단위 `InputStream`이므로 `InputStreamReader`로 문자로 바꾸고, `BufferedReader`로 감싸 `readLine()`을 사용함.

```java
try (BufferedReader br = new BufferedReader(new InputStreamReader(System.in))) {
    System.out.print("문자열 입력 : ");
    String value = br.readLine();
    System.out.println("value = " + value);
}
```

```
System.in (byte) → InputStreamReader (byte → char) → BufferedReader (버퍼 + readLine)
```

---

## 9. `Properties`

* **개념**: 설정값을 코드에 흩어 놓지 않고 `key=value` 형태의 설정 파일로 관리하기 위한 도구로, `Map`과 비슷한 key-value 구조에 **저장(`store`)과 로딩(`load`)** 기능이 더해짐. key와 value 모두 `String`이어야 함.

| 메소드 | 기능 |
| :--- | :--- |
| `setProperty(key, value)` | 값 저장 |
| `getProperty(key)` | 값 조회 |
| `store(writer, 주석)` | 파일로 저장 |
| `load(reader)` | 파일에서 불러오기 |

```java
Properties prop = new Properties();
prop.setProperty("app.name", "fullstack-course");
prop.setProperty("app.version", "2.0");

try (Writer writer = new OutputStreamWriter(new FileOutputStream(filePath), StandardCharsets.UTF_8)) {
    prop.store(writer, "application settings");
}

Properties loadedProp = new Properties();
try (Reader reader = new InputStreamReader(new FileInputStream(filePath), StandardCharsets.UTF_8)) {
    loadedProp.load(reader);
}
System.out.println(loadedProp.getProperty("app.name"));   // fullstack-course
```

---

## 10. 정수 상수의 문제점 → Enum

* **정수 상수 방식**: 인터페이스에 `public static final int`로 과목 번호를 선언해 사용함.

```java
public interface Subjects {
    int JAVA = 0;  int MYSQL = 1;  int JDBC = 2;
    int HTML = 0;  int CSS = 1;
}
```

* **문제점**
  * `Subjects.JAVA == Subjects.HTML`이 둘 다 `0`이라 **다른 과목인데 같다고 판단**됨.
  * `printSubject(int)`에 `100`처럼 의미 없는 값도 전달 가능해 타입 안정성이 없음.
  * `switch`에서 `case JAVA`와 `case HTML`이 같은 값이라 함께 쓸 수 없음.

---

## 11. 열거 타입 (`enum`)

* **개념**: 서로 관련된 상수들을 하나의 **타입**으로 묶어 정의하는 것으로, 각 상수는 그 enum 타입의 객체임을 학습함.

```java
public enum Subjects {
    JAVA, MYSQL, JDBC, HTML, CSS
}

Subjects subject1 = Subjects.JAVA;
Subjects subject2 = Subjects.HTML;
System.out.println(subject1 == subject2);   // false → 다른 과목으로 구분됨

// printSubject(2);                         // 컴파일 에러: Subjects 타입만 전달 가능
printSubject(Subjects.HTML);
```

* **주요 메소드**

| 메소드 | 기능 |
| :--- | :--- |
| `values()` | 선언된 모든 상수를 배열로 반환 (순회용) |
| `ordinal()` | 상수의 선언 위치를 0부터 반환 |
| `name()` | 상수의 소스 이름을 문자열로 반환 |
| `toString()` | 기본적으로 `name()`과 같은 값 반환 |

```java
for (Subjects s : Subjects.values()) {
    System.out.println((s.ordinal() + 1) + ". " + s.name());
}
// 1. JAVA  2. MYSQL  3. JDBC  4. HTML  5. CSS
```

---

## 12. Enum 필드 · 생성자 · 메소드

* **개념**: enum도 클래스처럼 필드, 생성자, 메소드를 가질 수 있음. 상수 뒤 괄호의 값이 생성자로 전달됨.


* **생성자 특징**: 생성자는 `private`만 가능하여 외부에서 `new`로 만들 수 없음. 상수 객체들은 enum이 처음 사용될 때 **한 번씩만** 미리 생성되고, 이후에는 이미 만들어진 상수를 참조함. (`UserRole.ADMIN`을 처음 사용할 때 생성자 출력이 4번 찍히고, `UserRole.GUEST`를 다시 사용해도 더 찍히지 않음)

```java
public enum UserRole {
    GUEST("게스트"),
    CONSUMER("구매자"),
    PRODUCER("판매자"),
    ADMIN("관리자");          // 필드/메소드가 있으면 상수 목록 끝에 ; 필수

    private final String description;

    private UserRole(String description) {
        this.description = description;
    }

    public String getDescription() {
        return description;
    }

    public String getNameToLowerCase() {
        return this.name().toLowerCase();
    }
}

UserRole admin = UserRole.ADMIN;
admin.name();                // "ADMIN"
admin.getDescription();      // "관리자"
admin.getNameToLowerCase();  // "admin"
```

| 구분 | 정수 상수 | `enum` |
| :--- | :--- | :--- |
| 타입 안정성 | X (`int`라면 아무 값이나 전달 가능) | O (해당 enum 타입만 전달 가능) |
| 값 중복 문제 | 다른 상수끼리 같은 값이면 구분 불가 | 상수마다 별개의 객체 |
| 부가 정보 | 별도 관리 필요 | 필드·메소드로 함께 관리 |
| 순회 | 불가 | `values()`로 가능 |

---

## 느낀 점

* `throw`는 "예외를 던진다", `throws`는 "처리를 호출한 곳에 맡긴다"로 구분하니 예외가 메소드 호출 경로를 따라 위로 전달되는 흐름이 이해됨.


* 사용자 정의 예외를 상속 구조로 설계하면서, `catch`를 구체적인 예외부터 작성하고 `Exception`을 마지막에 두는 이유가 상속과 다형성 때문이라는 점을 알게 됨.


* 스트림을 매번 `finally`에서 닫던 코드를 try-with-resources로 바꾸니 코드가 훨씬 짧아져서, 앞으로 자원을 다룰 때는 기본적으로 이 방식을 쓰면 되겠다고 생각함.


* `BufferedReader(new InputStreamReader(System.in))`처럼 스트림을 겹겹이 감싸는 구조가 처음엔 복잡해 보였지만, 각 보조 스트림이 기능을 하나씩 더해주는 것으로 보니 이해가 쉬워짐.


* 정수 상수로는 `JAVA == HTML`이 `true`가 되는 문제를 직접 확인하고 나니, enum이 단순한 상수 모음이 아니라 타입 안정성을 보장하는 **타입**이라는 점이 와닿음.
