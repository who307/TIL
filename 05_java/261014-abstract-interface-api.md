# Java 추상 클래스 · 인터페이스 · 주요 API 정리 (TIL)

## 1. 추상 클래스 (Abstract Class)

* **개념**: 직접 객체를 생성할 수 없으며, 자식 클래스들의 공통 기능을 정의하는 미완성 부모 클래스임을 이해함.


* **특징**: `abstract` 키워드로 선언하고, 일반 메소드와 몸체가 없는 추상 메소드를 함께 가질 수 있으며, 추상 메소드는 자식 클래스에서 반드시 오버라이딩해야 함을 학습함.


* **다형성 적용**: `Player` 추상 클래스를 `Warrior`, `Wizard`가 상속받아 `attack()`을 각자 구현하고, `Player[]` 배열로 묶어 하나의 반복문에서 다형적으로 호출하는 구조를 체득함.

```java
public abstract class Player {
    public abstract void attack();          // 추상 메소드: 자식이 반드시 구현
    public void levelUp() { System.out.println("레벨업!!"); } // 일반 메소드
}

Player[] players = { new Warrior(), new Wizard() };
for (Player player : players) {
    player.attack();   // 실제 객체의 오버라이딩된 메소드 실행
    player.levelUp();
}
```

---

## 2. 인터페이스 (Interface)

* **개념**: 클래스가 반드시 구현해야 할 기능의 규칙(약속)을 정의하는 타입으로, `interface` 키워드로 선언하고 `implements`로 구현함을 학습함.


* **특징**: 인터페이스의 메소드에는 `public abstract`가 자동으로 적용되며, 인터페이스 타입으로 구현 객체를 참조할 수 있고(다형성), 여러 인터페이스를 동시에 구현할 수 있음을 파악함.


* **USB 장치 예시**: `IConnectable` 인터페이스를 `Keyboard`, `Mouse`가 구현하고, `Computer`는 구체 클래스가 아닌 `IConnectable` 타입만 받아 어떤 장치든 연결/해제할 수 있는 구조를 이해함.

```java
public interface IConnectable {
    void connect();      // public abstract 자동 적용
    void disconnect();
}

public void connectDevice(IConnectable device) {
    device.connect();    // Keyboard든 Mouse든 동일하게 처리
}
```

| 구분 | 추상 클래스 | 인터페이스 |
| :--- | :--- | :--- |
| 의미 | 공통 속성과 기능을 물려주는 미완성 부모 클래스 | 구현 클래스가 지켜야 할 기능의 약속 |
| 키워드 | `abstract class` / `extends` | `interface` / `implements` |
| 다중 상속·구현 | 단일 상속만 가능 | 여러 개 동시 구현 가능 |

---

## 3. `Object` 클래스 — `toString()` / `equals()` / `hashCode()`

* **API의 개념**: 프로그램을 만들 때 사용할 수 있도록 미리 제공되는 클래스와 메소드의 사용 규칙임을 이해함.


* **`Object`**: 자바의 최상위 클래스로, 모든 클래스가 자동으로 상속하므로 그 메소드를 오버라이딩하여 사용할 수 있음을 학습함.


* **`toString()`**: 기본 출력 형식은 `클래스명@16진수 해시값`이라 의미가 없으므로, `Book` 클래스에서 필드값을 보여주도록 오버라이딩함.


* **동일성 vs 동등성**: `==`는 같은 객체인지(동일성)를 비교하고, `equals()`를 오버라이딩하면 필드 내용이 같은지(동등성)를 비교할 수 있음을 체득함. `equals()`를 재정의할 때는 `hashCode()`도 함께 재정의함.

```java
Book book2 = new Book(2, "목민심서", "정약용", 30000);
Book book3 = new Book(2, "목민심서", "정약용", 30000);

book2 == book3;        // false (서로 다른 객체)
book2.equals(book3);   // true  (필드 값이 같음 - 오버라이딩 결과)
```

---

## 4. `String` 클래스

* **주요 메소드**: 상품 데이터 문자열을 가공하며 아래 메소드들의 동작을 익힘.

| 메소드 | 기능 |
| :--- | :--- |
| `trim()` | 문자열 앞/뒤 공백 제거 |
| `toUpperCase()` / `toLowerCase()` | 대/소문자 변환 |
| `indexOf(str)` | 특정 문자의 위치(인덱스) 반환 |
| `substring(start, end)` | `start`부터 `end` 직전까지 잘라 새 문자열 반환 |
| `replace(a, b)` | 문자열 내의 `a`를 `b`로 대체한 새 문자열 반환 |
| `split(구분자)` | 구분자를 기준으로 잘라 배열로 반환 (구분자를 **정규식**으로 해석) |

* **문자열 생성 방식**: 리터럴은 String Constant Pool의 같은 문자열을 재사용하고, `new String()`은 매번 새 객체를 생성하므로 문자열 비교는 `equals()`를 사용하는 것이 안전함을 파악함.


* **null 안전 비교**: `"java".equals(input)`처럼 리터럴 쪽에서 호출하면 `input`이 `null`이어도 `NullPointerException`이 발생하지 않음.


* **불변(immutable)**: `+=` 등으로 문자열을 변경하면 기존 문자열이 수정되는 게 아니라 새 문자열 객체가 생성되고 변수가 새 객체를 가리키게 됨을 이해함.

```java
"report.final.pdf".split("\\.");   // "." 은 정규식에서 모든 문자 → 이스케이프 필요
"100/홍길동/서울//교육부".split("/"); // 빈 문자열("")도 배열 요소로 포함됨
```

* **이스케이프 문자**: `\n`(줄바꿈), `\t`(탭), `\"`, `\'`, `\\` 등 역슬래시로 시작하는 특수 문자를 정리함.

---

## 5. `StringBuilder`

* **String vs StringBuilder**: `String`은 불변이라 반복 연결 시 매번 새 객체가 생겨 비효율적이고, `StringBuilder`는 가변 객체로 `append()`로 내부 내용을 누적하므로 반복적인 문자열 조립에 적합함을 `System.nanoTime()`으로 작업 시간을 비교하며 확인함.


* **주요 메소드**: `append()`(추가), `delete(start, end)`(구간 삭제), `insert(index, value)`(삽입), `reverse()`(뒤집기), `toString()`(String 변환)을 익힘.


* **메소드 체이닝**: `append()`는 자기 자신을 반환하므로 연속 호출이 가능함.

```java
StringBuilder query = new StringBuilder();
query.append("SELECT ")
     .append("* ")
     .append("FROM product ")
     .append("WHERE price > ")
     .append(10000);
String sql = query.toString();
```

---

## 6. Wrapper 클래스

* **개념**: 기본 자료형의 값을 객체로 감싸는 클래스로, 문자열과 기본 자료형 간의 변환 기능을 제공하고 객체가 필요한 곳에서 기본 자료형을 사용할 수 있게 함을 학습함.


* **파싱(Parsing)**: `Double.parseDouble("175.5")`처럼 `parse자료형()` static 메소드로 문자열을 기본 자료형으로 변환하여 BMI를 계산함.


* **박싱 / 언박싱**: 기본 자료형 → Wrapper 객체(Boxing), Wrapper 객체 → 기본 자료형(Unboxing)이 자동으로 일어남(Auto-Boxing/Unboxing).


* **주의점**
  * Wrapper는 `null`을 가질 수 있으며, `null`을 언박싱하면 `NullPointerException`이 발생함.
  * Wrapper 값 비교는 `==`가 아닌 `equals()`를 사용함 (`==`는 내부 캐시 최적화에 따라 결과가 달라짐, 예: `Integer` 128 이상).


* **숫자 → 문자열**: `String.valueOf(값)` 또는 `값 + ""`.

---

## 7. 날짜/시간 API (`java.time`)

* **주요 클래스**: `LocalDate`(날짜), `LocalTime`(시간), `LocalDateTime`(날짜+시간), 시간대가 필요한 경우 `ZonedDateTime`과 `ZoneId`를 사용함을 학습함.


* **생성 및 조회**: `of()`로 특정 시각, `now()`로 현재 시각을 생성하고 `getYear()`, `getMonthValue()`, `getDayOfWeek()` 등으로 값을 꺼냄.


* **불변 객체**: `plusXxx()` / `minusXxx()`는 원본을 바꾸지 않고 새 객체를 반환하므로 반환값을 변수에 저장해야 함을 체득함.


* **비교**: `isBefore()`, `isAfter()`, `isEqual()`.


* **포맷팅 / 파싱**: `DateTimeFormatter.ofPattern()`으로 패턴을 만들어 `format()`(객체 → 문자열), `parse()`(문자열 → 객체)로 변환함.

```java
LocalDateTime trainDeparture = LocalDateTime.of(LocalDate.of(2026, 12, 25), LocalTime.of(14, 30));
LocalDateTime deadline = trainDeparture.minusMinutes(20);   // 새 객체 반환

DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy년 MM월 dd일 HH시 mm분 출발");
String ticket = trainDeparture.format(formatter);           // LocalDateTime → String

LocalDate parsed = LocalDate.parse("20261225", DateTimeFormatter.ofPattern("yyyyMMdd")); // String → LocalDate
```

---

## 느낀 점

* 추상 클래스와 인터페이스를 직접 구현해보며, "공통 기능을 물려주는 부모"와 "지켜야 할 기능의 약속"이라는 역할 차이를 명확히 구분할 수 있게 됨.


* `Computer`가 구체 클래스가 아닌 인터페이스 타입에 의존하도록 설계하면, 새로운 장치가 추가되어도 기존 코드를 수정할 필요가 없다는 확장성의 장점을 체감함.


* `==`(동일성)와 `equals()`(동등성)의 차이가 `Object`, `String`, Wrapper 전반에 공통으로 적용되는 핵심 개념임을 정리함.


* `String`과 `java.time` 객체가 불변이라는 특성을 이해하고, 반복적인 문자열 조립에는 `StringBuilder`를 사용하는 이유를 성능 비교로 확인함.
