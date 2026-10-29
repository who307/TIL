# Java 예외처리 · 입출력(IO) · Enum 정리 (TIL)

## 1. 예외처리 (Exception Handling)

* **한 줄 비유**: 예외(Exception)는 프로그램이 달리다 만난 **돌부리**임. 예외처리는 넘어졌을 때 **"어디서 넘어졌는지 알려주고, 털고 일어나 계속 가는 방법"** 을 미리 정해두는 것.
* **처리하지 않으면?**: `Integer.parseInt("abc")`처럼 숫자가 아닌 글자를 숫자로 바꾸려 하면 예외가 생기고, 아무도 처리하지 않으면 프로그램이 그 자리에서 **멈춰버림**(비정상 종료).
* **`throw` vs `throws`**

| 키워드 | 위치 | 의미 | 비유 |
| :--- | :--- | :--- | :--- |
| `throw` | 메소드 내부 | 지금 이 자리에서 예외 객체를 **직접 던짐** | "문제 생겼어요!" 하고 손을 듦 |
| `throws` | 메소드 선언부 | 이 메소드에서 생긴 예외를 **나를 부른 쪽에 떠넘김** | "저는 못 고쳐요, 부른 사람이 처리해 주세요" |

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

* **따라가기**
  * `checkEnoughMoney(1000, 5000)` → `5000 >= 1000` 참 → `상품을 구입하기 위한 돈이 충분합니다.` → 아래로 내려와 `즐거운 쇼핑 하세요~`
  * `checkEnoughMoney(1000, 500)` → 거짓 → `throw new Exception("돈이 부족합니다!!")` → **그 즉시 메소드를 빠져나가** 부른 쪽으로 예외가 날아감. `즐거운 쇼핑 하세요~`는 실행되지 않음.
  * 선언부의 `throws Exception` → "이 메소드는 Exception을 던질 수 있으니 부르는 쪽이 처리하세요"라는 표시.
* **`main`까지 계속 떠넘기면?**: `main`에도 `throws`를 붙이면 예외가 **JVM**(자바 실행기)까지 올라감 → JVM은 오류 내용을 찍고 프로그램을 끝냄 → `main` 마지막의 `"프로그램을 종료합니다."` 같은 줄은 출력되지 않음.

---

## 2. `try-catch-finally`

* **`try`**: 예외가 생길 **수도 있는** 코드를 감쌈. "조심해서 해 봐."
* **`catch`**: `try`에서 예외가 생기면, **타입이 맞는** `catch`가 받아서 처리함. 처리가 끝나면 프로그램은 멈추지 않고 **다음 줄을 계속** 실행함.
* **`finally`**: 예외가 생기든 안 생기든 **항상 마지막에** 실행됨. 파일 닫기처럼 **꼭 해야 하는 뒷정리**를 넣음.

```java
try {
    et.checkEnoughMoney(1000, 500);
} catch (Exception e) {
    System.out.println(e.getMessage());   // 예외 메시지
    e.printStackTrace();                  // 예외가 전달된 호출 경로 출력
}
```

* **따라가기**
  1. `try` 안에서 `checkEnoughMoney(1000, 500)` → 돈이 부족해서 예외가 던져짐.
  2. 던져진 예외(`Exception`)가 `catch (Exception e)`와 타입이 맞음 → `e`에 예외 객체가 담김.
  3. `e.getMessage()` → 던질 때 넣은 메시지 → `돈이 부족합니다!!`
  4. `e.printStackTrace()` → 예외가 **어느 메소드 몇 번째 줄에서 생겨서 어떤 경로로 올라왔는지**를 빨간 글씨로 찍음. 실제 서비스에서는 보통 로깅 도구로 필요한 정보를 기록함.
  5. `catch`가 끝나면 그 아래 코드가 정상적으로 이어서 실행됨.

* **Checked / Unchecked Exception**

| 구분 | 컴파일러 검사 | 예시 |
| :--- | :--- | :--- |
| Checked Exception | `try-catch` 또는 `throws` 선언을 강제함 | `Exception`, `IOException`, `FileNotFoundException` |
| Unchecked Exception | 처리를 강제하지 않으며 주로 실행 중 발견됨 | `NumberFormatException`, `NoSuchElementException` |

* **Checked**: 파일이 없을 수 있는 것처럼 **미리 대비할 수 있는** 상황 → 처리 코드를 안 쓰면 **컴파일이 안 됨**.
* **Unchecked**(`RuntimeException`의 자식들): 대부분 **코드 실수**로 생기는 상황 → 처리를 강제하지 않음. 필요하면 잡을 수는 있음.

```java
Deque<Integer> stack = new ArrayDeque<>();
try {
    stack.pop();                          // 빈 스택 → NoSuchElementException
} catch (NoSuchElementException e) {
    System.out.println("빈 스택에서는 값을 꺼낼 수 없습니다.");
}
```

* **따라가기**: 빈 스택에서 `pop()` → 꺼낼 게 없어서 `NoSuchElementException` → `catch`가 받아서 `빈 스택에서는 값을 꺼낼 수 없습니다.` 출력 → 프로그램은 계속 실행됨.

---

## 3. 사용자 정의 예외 (Custom Exception)

* **왜 만드나**: 모든 문제를 그냥 `Exception`으로 던지면 "무슨 문제인지" 이름만 봐서는 모름. `NotEnoughMoneyException`처럼 **상황이 드러나는 이름**의 예외를 직접 만들면 읽기도 처리하기도 쉬움.
* **만드는 법**: `Exception`을 상속받고, 생성자에서 `super(message)`로 메시지를 부모에게 넘김 → 나중에 `getMessage()`로 꺼낼 수 있음.
* **예외 계층 설계**: 음수 관련 예외를 `NegativeException` 아래로 묶어서, "음수 문제 전부"를 한 번에 잡을 수도 있고 "가격 음수만" 따로 잡을 수도 있게 함.

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

* **따라가기**: `throw new PriceNegativeException("상품 가격은 음수일 수 없습니다.")`
  1. `PriceNegativeException` 생성자 → `super(message)` → `NegativeException` 생성자 → `super(message)` → `Exception`이 메시지를 저장.
  2. 이 예외는 `PriceNegativeException`이면서, 동시에 `NegativeException`이고, `Exception`이기도 함(상속).

* **multi-catch와 catch 순서**
  * `catch (A | B e)`: 처리 방법이 같은 예외 여러 개를 `|`로 묶어 한 번에 잡음.
  * `catch`는 **위에서부터** 차례로 검사함 → **구체적인 예외를 위에, 넓은 예외(`Exception`)를 맨 아래에** 둬야 함. `Exception`을 맨 위에 두면 모든 예외를 다 가로채서 아래 `catch`에는 영원히 도달하지 못함 → 자바가 컴파일 에러로 막음. 그물코가 촘촘한 그물을 먼저, 큰 그물을 나중에 치는 것과 같음.

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

* **따라가기**: 가격 30000, 가진 돈 5000 (이 버전의 `checkEnoughMoney`는 돈이 부족하면 `NotEnoughMoneyException`을 던진다고 가정)
  1. 가격·돈 모두 음수가 아님 → 음수 예외 없음.
  2. `5000 < 30000` → `NotEnoughMoneyException` 던져짐.
  3. 첫 번째 `catch`(음수 예외들) → 타입이 안 맞음 → 건너뜀.
  4. 두 번째 `catch(NotEnoughMoneyException)` → 맞음 → `getSimpleName()`(클래스 짧은 이름) + 메시지 출력 → 예: `NotEnoughMoneyException발생!!가진 돈이 부족합니다.`
  5. 세 번째 `catch`는 실행 안 됨(이미 잡혔으므로).
  6. `finally` → `구매 가능 여부 확인을 마칩니다.` (예외가 없었어도 실행됨)

---

## 4. 오버라이딩 시 예외 선언 규칙

* 부모 메소드가 `throws IOException`이라고 선언했을 때, 자식이 오버라이딩하면서 선언할 수 있는 예외 범위:

| 자식 메소드의 `throws` | 허용 여부 |
| :--- | :--- |
| 선언하지 않음 | O |
| 같은 예외 (`IOException`) | O |
| 더 구체적인 하위 예외 (`FileNotFoundException`) | O |
| 더 넓은 상위 예외 (`Exception`) | X (컴파일 에러) |

* **이유**: `부모 p = new 자식();`처럼 부모 타입으로 자식을 다루는 코드는 **부모의 `throws`만 보고** "`IOException`만 잡으면 되겠네" 하고 `catch`를 씀. 그런데 자식이 더 넓은 `Exception`을 던지면 그 `catch`로 못 잡는 예외가 튀어나옴 → 그래서 자식은 부모보다 **같거나 좁게만** 던질 수 있음. "우산 하나면 된다"고 했는데 우박이 쏟아지면 안 되는 것과 같음.

---

## 5. `File` 클래스

* **개념**: 파일 **자체**를 다루는 클래스(경로, 크기, 만들기, 지우기). 파일 **안의 내용**을 읽고 쓰는 건 스트림이 담당함. 책의 **겉표지 정보**를 보는 것과 **책 내용**을 읽는 것의 차이.

| 메소드 | 기능 |
| :--- | :--- |
| `createNewFile()` | 새 파일 생성 시 `true`, 이미 존재하면 `false` (`IOException` 처리 필요) |
| `length()` | 파일 크기 (byte) |
| `getPath()` / `getAbsolutePath()` | 경로 / 절대 경로 |
| `delete()` | 파일 삭제 성공 시 `true` |

* **경로**: `getPath()`는 내가 적은 그대로(예: `test.txt`), `getAbsolutePath()`는 컴퓨터 맨 위부터의 **전체 주소**(예: `C:\project\test.txt`).

---

## 6. 바이트 스트림 — `FileInputStream` / `FileOutputStream`

* **스트림(Stream)**: 데이터가 흐르는 **빨대**. 파일 → 프로그램으로 빨아들이면 입력(Input), 프로그램 → 파일로 내보내면 출력(Output).
* **바이트 스트림**: **1바이트씩** 읽고 쓰는 빨대. 사진·영상·글자 등 모든 파일을 다룰 수 있음. 다만 한글은 한 글자가 여러 바이트라 1바이트씩 끊어 읽으면 깨질 수 있어서, 글자는 `Reader`/`Writer`가 알맞음.
* **`read()`**: 1바이트를 읽어 `0~255` 숫자로 돌려줌. 더 읽을 게 없으면 **`-1`** 을 돌려줌 → `-1`이 나올 때까지 반복하면 파일 끝까지 읽음.
* **자원 반납**: 다 쓴 스트림은 **반드시 `close()`** 해야 함(빨대를 빼야 다른 프로그램도 파일을 쓸 수 있음). 옛날 방식은 `finally`에서 `null` 확인 + `close()` + 또 `try-catch`라 코드가 길어짐.

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

* **따라가기** (파일 내용이 `abc`)
  1. `new FileInputStream(...)` → 파일에 빨대를 꽂음. 파일이 없으면 `FileNotFoundException`(`IOException`의 자식) → `catch`.
  2. `(value = fin.read()) != -1` → 한 바이트 읽어 `value`에 담고, `-1`이 아니면 반복.
     * 97 → `(char) 97` → `a` 출력 → 98 → `b` → 99 → `c` → 그다음 `-1` → 반복 끝.
  3. `finally` → `fin`이 `null`이 아니면(열렸으면) `close()`. `close()`도 예외를 던질 수 있어서 또 `try-catch`로 감쌈.

* **try-with-resources**: `try ( ... )` 괄호 안에서 스트림을 만들면, 블록이 끝날 때 자바가 **자동으로 `close()`** 해 줌. 위의 긴 `finally`가 필요 없어짐.
* **덮어쓰기 / 이어쓰기**: `new FileOutputStream(path)`는 기존 내용을 **지우고 새로 씀**. 이어 쓰려면 `new FileOutputStream(path, true)`.

```java
try (FileOutputStream fout = new FileOutputStream("testOutputStream.txt")) {
    fout.write(97);                                   // 'a'
    fout.write(new byte[] {98, 99, 100, 101, 10, 102}); // bcde + 개행(10) + f
} catch (IOException e) {
    e.printStackTrace();
}
```

* **따라가기**
  1. `try (...)` 괄호 안에서 출력 스트림 생성 (파일이 없으면 새로 만듦).
  2. `write(97)` → 숫자 97 = 문자 `a`를 씀.
  3. `write(new byte[] {...})` → 98(`b`) 99(`c`) 100(`d`) 101(`e`) 10(줄바꿈) 102(`f`)를 한꺼번에 씀.
  4. 블록이 끝나면서 자동 `close()`.
  5. 파일 내용 → 첫 줄 `abcde`, 둘째 줄 `f`

---

## 7. 문자 스트림 — `FileReader` / `FileWriter`

* **개념**: 바이트가 아니라 **글자(문자) 단위**로 읽고 쓰는 빨대. 한글처럼 여러 바이트로 된 글자도 깨지지 않고 한 글자씩 처리함.
* **문자셋 주의**: 문자셋(charset)은 글자를 숫자로 바꾸는 **규칙표**(UTF-8, MS949 등). `FileReader`/`FileWriter`는 **실행하는 컴퓨터의 기본 규칙표**를 써서, 다른 컴퓨터에서 열면 한글이 깨질 수 있음. 규칙표를 확실히 정하려면 `InputStreamReader`/`OutputStreamWriter`에 `StandardCharsets.UTF_8`을 지정함.

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

* **따라가기**
  * `fw.write(97)` → `a`, `fw.write('A')` → `A`, `char[]` → `apple`, 문자열 → `안녕하세요 반갑습니다` → 파일 내용: `aAapple안녕하세요 반갑습니다`
  * 두 번째 블록 → 바이트 출력 스트림(`FileOutputStream`)을 **UTF-8 규칙으로 글자를 바꿔주는** `OutputStreamWriter`로 감쌈 → 어느 컴퓨터에서 열어도 한글이 깨지지 않는 파일이 됨.

| 구분 | 바이트 스트림 | 문자 스트림 |
| :--- | :--- | :--- |
| 단위 | 1 byte | 문자 (char) |
| 입력 | `InputStream` (`FileInputStream`) | `Reader` (`FileReader`) |
| 출력 | `OutputStream` (`FileOutputStream`) | `Writer` (`FileWriter`) |
| 용도 | 이미지·영상 등 모든 파일 | 텍스트 파일 |

---

## 8. 보조 스트림 (Filter Stream)

* **개념**: 혼자서는 못 쓰고, 기본 스트림을 **감싸서 기능을 더해주는** 스트림. 빨대에 끼우는 **필터**나 **확장 부품**과 같음.
* **`BufferedWriter` / `BufferedReader`**: **버퍼**(임시 바구니)를 달아서 성능을 높임. 물을 한 방울씩 나르지 않고 **양동이에 모았다가 한 번에** 붓는 것과 같음 → 파일에 접근하는 횟수가 줄어 빨라짐.
  * `flush()`: 바구니에 남은 것을 지금 **바로 파일로 쏟음**. try-with-resources가 끝나며 `close()`될 때 `flush()`도 자동으로 됨.
  * `readLine()`: **한 줄씩** 읽어 `String`으로 돌려주고, 더 읽을 줄이 없으면 `null`.

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

* **따라가기**
  1. `FileWriter`를 `BufferedWriter`로 감쌈 → `write("안녕")`, `write("반가워")`는 바로 파일에 가지 않고 버퍼에 쌓임.
  2. 블록 끝 → `close()` → `flush()` → 버퍼 내용이 파일로 → 파일 내용: `안녕반가워` (`newLine()`을 안 했으므로 한 줄)
  3. 읽기 → `readLine()` → `"안녕반가워"` 출력 → 다음 `readLine()` → `null` → 반복 끝.

* **형변환 보조 스트림 (`InputStreamReader` / `OutputStreamWriter`)**: **바이트** 스트림을 **문자** 스트림으로 바꿔 이어주는 **변환 어댑터**. 키보드 입력(`System.in`)은 바이트 스트림이라, 한 줄씩 글자로 읽으려면 `InputStreamReader`로 문자로 바꾸고 `BufferedReader`로 감싸야 함.

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

* **따라가기**: 사용자가 `hello` 엔터 → `System.in`이 바이트로 받음 → `InputStreamReader`가 글자로 바꿈 → `BufferedReader.readLine()`이 한 줄을 통째로 → `value = hello` 출력.

---

## 9. `Properties`

* **개념**: 설정값을 코드 여기저기에 흩어두지 않고 **`key=value` 형태의 설정 파일**로 관리하는 도구. `Map`처럼 key로 value를 찾는데, **파일로 저장(`store`)하고 불러오는(`load`)** 기능이 더 있음. key와 value는 모두 `String`.
* **왜 좋나**: 앱 이름이나 DB 주소가 바뀌어도 **설정 파일만 고치면** 되고 코드를 다시 컴파일할 필요가 없음.

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

* **따라가기**
  1. `prop`에 `app.name=fullstack-course`, `app.version=2.0` 두 쌍을 넣음.
  2. `store(writer, "application settings")` → 파일에 저장. 파일 내용은 대략:
     ```
     #application settings
     #(저장한 날짜)
     app.name=fullstack-course
     app.version=2.0
     ```
  3. 새 `Properties` 객체 `loadedProp`를 만들어 `load(reader)` → 파일 내용을 읽어 다시 key-value로 채움.
  4. `getProperty("app.name")` → `fullstack-course`

---

## 10. 정수 상수의 문제점 → Enum

* **정수 상수 방식**: 인터페이스에 `public static final int`로 과목 번호를 정해 씀. (인터페이스 안의 변수는 자동으로 `public static final`이 붙음)

```java
public interface Subjects {
    int JAVA = 0;  int MYSQL = 1;  int JDBC = 2;
    int HTML = 0;  int CSS = 1;
}
```

* **문제점**
  * `Subjects.JAVA == Subjects.HTML` → 둘 다 `0`이라 **다른 과목인데 같다고** 판단됨.
  * `printSubject(int)`에 `100`처럼 **아무 의미 없는 숫자**도 넣을 수 있음 → 타입 안정성이 없음.
  * `switch`에서 `case JAVA`와 `case HTML`이 같은 값(0)이라 **함께 쓸 수 없음**(중복 case 에러).

---

## 11. 열거 타입 (`enum`)

* **개념**: 서로 관련된 상수들을 하나의 **새로운 타입**으로 묶은 것. 각 상수(`JAVA`, `HTML` 등)는 **그 enum 타입의 객체 하나씩**임. 메뉴판에 적힌 메뉴만 주문할 수 있는 것처럼, 정해둔 값 말고는 넣을 수 없음.

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

* **따라가기**
  * `Subjects.JAVA`와 `Subjects.HTML`은 서로 **다른 객체** → `==` → `false`. 숫자 값이 겹칠 일이 없음.
  * `printSubject(Subjects s)`라고 만들어 두면 → `printSubject(2)`는 숫자라 안 들어감(컴파일 에러) → 정해진 5개 과목만 넣을 수 있음.

* **주요 메소드**

| 메소드 | 기능 | `HTML`의 결과 |
| :--- | :--- | :--- |
| `values()` | 선언된 모든 상수를 배열로 반환 (순회용) | - |
| `ordinal()` | 상수의 선언 위치를 0부터 반환 | `3` |
| `name()` | 상수의 소스 이름을 문자열로 반환 | `"HTML"` |
| `toString()` | 기본적으로 `name()`과 같은 값 반환 | `"HTML"` |

```java
for (Subjects s : Subjects.values()) {
    System.out.println((s.ordinal() + 1) + ". " + s.name());
}
// 1. JAVA  2. MYSQL  3. JDBC  4. HTML  5. CSS
```

* **따라가기**: `values()` → `[JAVA, MYSQL, JDBC, HTML, CSS]` → 하나씩 돌며 `ordinal()`(0부터)에 1을 더해 번호로 씀 → `1. JAVA`, `2. MYSQL`, `3. JDBC`, `4. HTML`, `5. CSS`

---

## 12. Enum 필드 · 생성자 · 메소드

* **개념**: enum도 클래스처럼 **필드, 생성자, 메소드**를 가질 수 있음. 상수 뒤 괄호 안의 값(`GUEST("게스트")`)이 **생성자로 전달**됨. 각 상수가 설명 같은 **부가 정보**를 함께 들고 다닐 수 있음.
* **생성자 특징**
  * 생성자는 **`private`만** 가능 → 밖에서 `new UserRole(...)`로 새 상수를 만들 수 없음.
  * 상수 객체들은 enum을 **처음 사용할 때 한꺼번에, 딱 한 번씩** 만들어지고, 그 뒤로는 이미 만든 것을 재사용함.
  * 예: 생성자에 출력문을 넣고 `UserRole.ADMIN`을 처음 쓰면 → 상수 4개가 모두 만들어지며 출력이 **4번** 찍힘 → 이후 `UserRole.GUEST`를 써도 더 찍히지 않음.

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

* **따라가기**
  1. `UserRole.ADMIN`을 처음 사용 → 상수 4개 생성: `GUEST`는 `description = "게스트"`, … `ADMIN`은 `"관리자"`.
  2. `admin.name()` → 상수 이름 그대로 → `"ADMIN"`
  3. `admin.getDescription()` → 생성자로 받아 저장한 값 → `"관리자"`
  4. `admin.getNameToLowerCase()` → `"ADMIN"`을 소문자로 → `"admin"`
  * `ADMIN("관리자");`의 세미콜론: 상수 목록이 끝나고 필드·메소드가 시작된다는 표시라 빼면 에러.

| 구분 | 정수 상수 | `enum` |
| :--- | :--- | :--- |
| 타입 안정성 | X (`int`라면 아무 값이나 전달 가능) | O (해당 enum 타입만 전달 가능) |
| 값 중복 문제 | 다른 상수끼리 같은 값이면 구분 불가 | 상수마다 별개의 객체 |
| 부가 정보 | 별도 관리 필요 | 필드·메소드로 함께 관리 |
| 순회 | 불가 | `values()`로 가능 |

---

## 느낀 점

* `throw`는 "예외를 던진다", `throws`는 "처리를 호출한 곳에 맡긴다"로 구분하니 예외가 메소드 호출 경로를 따라 위로 전달되는 흐름이 이해됨
* 사용자 정의 예외를 상속 구조로 설계하면서, `catch`를 구체적인 예외부터 작성하고 `Exception`을 마지막에 두는 이유가 상속과 다형성 때문이라는 점을 알게 됨
* 스트림을 매번 `finally`에서 닫던 코드를 try-with-resources로 바꾸니 코드가 훨씬 짧아져서, 앞으로 자원을 다룰 때는 기본적으로 이 방식을 쓰면 되겠다고 생각함
* `BufferedReader(new InputStreamReader(System.in))`처럼 스트림을 겹겹이 감싸는 구조가 처음엔 복잡해 보였지만, 각 보조 스트림이 기능을 하나씩 더해주는 것으로 보니 이해가 쉬워짐
* 정수 상수로는 `JAVA == HTML`이 `true`가 되는 문제를 직접 확인하고 나니, enum이 단순한 상수 모음이 아니라 타입 안정성을 보장하는 **타입**이라는 점이 와닿음
