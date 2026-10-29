# Java 추상 클래스 · 인터페이스 · 주요 API 정리 (TIL)

## 1. 추상 클래스 (Abstract Class)

* **한 줄 비유**: 추상 클래스는 **빈칸이 있는 학습지**임. 공통 문제는 이미 풀려 있고(일반 메소드), 빈칸(추상 메소드)은 학생(자식 클래스)마다 **반드시 직접 채워야** 함.
* **특징**
  1. `abstract class`로 선언하며, **직접 `new`로 객체를 만들 수 없음**.
  2. 몸통이 있는 **일반 메소드**와 몸통이 없는 **추상 메소드**를 함께 가질 수 있음.
  3. 자식은 추상 메소드를 **반드시 오버라이딩**해야 함. 안 하면 컴파일 에러.
* **다형성 적용**: `Warrior`(전사), `Wizard`(마법사)가 `Player`를 상속받아 `attack()`을 각자 방식으로 채우고, `Player[]` 배열 하나에 담아 반복문 하나로 모두 공격시킴.

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

* **따라가기** (전사는 `"검으로 공격!"`, 마법사는 `"불꽃 마법 공격!"`을 출력하도록 구현했다고 가정)
  1. `players` 배열 → 0번 칸에 전사, 1번 칸에 마법사. 배열 타입은 둘 다 담을 수 있는 부모 `Player`.
  2. 첫 바퀴(전사) → `attack()` → 전사가 채운 버전 → `검으로 공격!` → `levelUp()` → 부모 것 그대로 → `레벨업!!`
  3. 두 번째 바퀴(마법사) → `불꽃 마법 공격!` → `레벨업!!`
  * 반복문 코드는 하나인데 객체마다 다르게 동작함 → 다형성.

---

## 2. 인터페이스 (Interface)

* **한 줄 비유**: 인터페이스는 **콘센트 규격**임. "220V, 구멍 두 개"라는 약속만 정해두면, 어느 회사가 만든 전자제품이든 그 규격만 지키면 꽂아 쓸 수 있음.
* **특징**
  1. `interface`로 선언하고, 클래스는 `implements`로 **약속을 지키겠다고(구현) 선언**함.
  2. 인터페이스에 적은 메소드는 자동으로 `public abstract`(공개 + 몸통 없음)가 됨. 구현 클래스가 몸통을 채워야 함.
  3. 인터페이스 타입 변수로 구현 객체를 담을 수 있음(다형성).
  4. 클래스는 부모를 하나만 `extends`할 수 있지만, 인터페이스는 **여러 개를 동시에** `implements`할 수 있음.
* **USB 장치 예시**: `Keyboard`, `Mouse`가 `IConnectable`(연결 가능) 약속을 지킴 → `Computer`는 구체적인 장치 이름을 몰라도 `IConnectable`만 받아서 연결·해제함.

```java
public interface IConnectable {
    void connect();      // public abstract 자동 적용
    void disconnect();
}

public void connectDevice(IConnectable device) {
    device.connect();    // Keyboard든 Mouse든 동일하게 처리
}
```

* **따라가기**
  * `connectDevice(new Keyboard())` → `device`에 키보드가 들어감 → 키보드가 구현한 `connect()` 실행 (예: `키보드 연결됨`).
  * `connectDevice(new Mouse())` → 마우스가 구현한 `connect()` 실행 (예: `마우스 연결됨`).
  * 나중에 `Speaker`가 `IConnectable`을 구현하면 → `Computer` 코드는 **한 줄도 안 고치고** 스피커도 연결 가능.

| 구분 | 추상 클래스 | 인터페이스 |
| :--- | :--- | :--- |
| 의미 | 공통 속성과 기능을 물려주는 미완성 부모 클래스 | 구현 클래스가 지켜야 할 기능의 약속 |
| 키워드 | `abstract class` / `extends` | `interface` / `implements` |
| 다중 상속·구현 | 단일 상속만 가능 | 여러 개 동시 구현 가능 |

---

## 3. `Object` 클래스 — `toString()` / `equals()` / `hashCode()`

* **API**: 자바가 **미리 만들어 둔 클래스와 메소드 모음**과 그 사용법. 공구 상자에 들어 있는 망치·드라이버처럼 가져다 쓰기만 하면 됨.
* **`Object`**: 자바의 **모든 클래스의 조상**. 아무것도 `extends`하지 않아도 자동으로 `Object`를 상속함. 그래서 모든 객체가 `toString()`, `equals()`, `hashCode()`를 가지고 있음.
* **`toString()`**: 객체를 글자로 바꿔주는 메소드. `System.out.println(객체)`를 하면 자동으로 불림.
  * 기본 모양은 `Book@1b6d3586`처럼 `클래스명@해시값`이라 무슨 책인지 알 수 없음 → `Book`에서 오버라이딩해서 `Book[번호=2, 제목=목민심서, ...]`처럼 필드를 보여주게 함.
* **동일성 vs 동등성**
  * **동일성 (`==`)**: **같은 물건**인가? (메모리 위치가 같은가)
  * **동등성 (`equals()`)**: **내용이 같은**가? → `Object`의 기본 `equals()`는 `==`와 똑같이 동작하므로, 내용 비교를 하려면 **오버라이딩**해야 함.
  * 쌍둥이 비유: 쌍둥이는 생김새(내용)가 같아도 **서로 다른 사람**임. `==`는 "같은 사람이야?", `equals()`는 "생김새가 같아?"를 묻는 것.
* **`hashCode()`**: 객체를 빨리 찾기 위한 **번호표**. `equals()`가 `true`인 두 객체는 `hashCode()`도 같아야 한다는 규칙이 있어서, `equals()`를 재정의하면 `hashCode()`도 **함께** 재정의함. (안 그러면 `HashSet` 같은 곳에서 같은 책이 두 번 들어가는 문제가 생김)

```java
Book book2 = new Book(2, "목민심서", "정약용", 30000);
Book book3 = new Book(2, "목민심서", "정약용", 30000);

book2 == book3;        // false (서로 다른 객체)
book2.equals(book3);   // true  (필드 값이 같음 - 오버라이딩 결과)
```

* **따라가기**
  1. `new`를 두 번 했으므로 메모리에 책 객체가 **2개** 따로 생김.
  2. `book2 == book3` → 서로 다른 위치 → `false`
  3. `book2.equals(book3)` → `Book`에서 "번호·제목·저자·가격이 모두 같으면 같다"고 오버라이딩해 둠 → 네 값이 모두 같음 → `true`

---

## 4. `String` 클래스

* **주요 메소드**: 상품 데이터 문자열을 가공하며 아래 메소드들의 동작을 익힘.

| 메소드 | 기능 | 예 → 결과 |
| :--- | :--- | :--- |
| `trim()` | 문자열 앞/뒤 공백 제거 | `"  hi  "` → `"hi"` |
| `toUpperCase()` / `toLowerCase()` | 대/소문자 변환 | `"Java"` → `"JAVA"` / `"java"` |
| `indexOf(str)` | 특정 문자의 위치(인덱스) 반환, 없으면 `-1` | `"hello".indexOf("l")` → `2` |
| `substring(start, end)` | `start`부터 `end` 직전까지 잘라 새 문자열 반환 | `"hello".substring(1, 3)` → `"el"` |
| `replace(a, b)` | 문자열 내의 `a`를 `b`로 대체한 새 문자열 반환 | `"a-b".replace("-", "+")` → `"a+b"` |
| `split(구분자)` | 구분자를 기준으로 잘라 배열로 반환 (구분자를 **정규식**으로 해석) | `"a,b".split(",")` → `["a", "b"]` |

* **문자열 만드는 두 방법**
  * **리터럴 `"java"`**: 자바가 문자열을 **String Constant Pool**(문자열 공용 보관함)에 한 번만 저장하고, 같은 글자가 또 나오면 **그걸 재사용**함 → `"java" == "java"`는 `true`.
  * **`new String("java")`**: 보관함과 상관없이 **매번 새 객체**를 만듦 → `new String("java") == "java"`는 `false`.
  * 그래서 문자열 내용 비교는 **항상 `equals()`** 를 씀.
* **null 안전 비교**: `input.equals("java")`는 `input`이 `null`이면 `NullPointerException`(없는 것에 일을 시킨 에러)이 남. `"java".equals(input)`처럼 **확실히 있는 리터럴 쪽에서** 부르면 `input`이 `null`이어도 그냥 `false`가 나옴.
* **불변(immutable)**: `String`은 한 번 만들면 **절대 안 바뀜**. `s += "!"`를 하면 원래 문자열이 고쳐지는 게 아니라 **새 문자열이 만들어지고** `s`가 새 것을 가리키게 됨.

```java
"report.final.pdf".split("\\.");   // "." 은 정규식에서 모든 문자 → 이스케이프 필요
"100/홍길동/서울//교육부".split("/"); // 빈 문자열("")도 배열 요소로 포함됨
```

* **따라가기**
  * `split(".")`라고 쓰면 정규식에서 `.`은 "아무 글자 하나"라는 뜻이라 모든 글자가 구분자가 되어 빈 배열이 나옴. `\\.`처럼 역슬래시로 "진짜 점"이라고 알려줘야 함 → `["report", "final", "pdf"]`
  * `"100/홍길동/서울//교육부".split("/")` → `//` 사이의 빈칸도 하나의 칸으로 셈 → `["100", "홍길동", "서울", "", "교육부"]` (5칸)
* **이스케이프 문자**: 역슬래시(`\`)로 시작하는 특수 문자. `\n`(줄바꿈), `\t`(탭), `\"`(큰따옴표 자체), `\'`(작은따옴표 자체), `\\`(역슬래시 자체).

---

## 5. `StringBuilder`

* **왜 필요한가**: `String`은 불변이라 `+=`를 1000번 하면 **새 문자열이 1000개** 만들어짐 → 느리고 메모리 낭비. `System.nanoTime()`(아주 정밀한 시계)으로 시간을 재 보면 차이가 확연함.
* **`StringBuilder`**: **고칠 수 있는(가변) 문자열**. 하나의 공책에 계속 이어서 적는 것과 같아서, 반복해서 문자열을 조립할 때 빠름.
* **주요 메소드**: `append()`(뒤에 추가), `delete(start, end)`(구간 삭제), `insert(index, value)`(중간 삽입), `reverse()`(뒤집기), `toString()`(완성된 `String`으로 바꾸기).
* **메소드 체이닝**: `append()`는 **자기 자신을 돌려주므로** `.append().append()`처럼 이어서 쓸 수 있음.

```java
StringBuilder query = new StringBuilder();
query.append("SELECT ")
     .append("* ")
     .append("FROM product ")
     .append("WHERE price > ")
     .append(10000);
String sql = query.toString();
```

* **따라가기** (`query` 안의 내용 변화)
  1. 빈 상태 `""`
  2. `"SELECT "` → `"SELECT * "` → `"SELECT * FROM product "` → `"SELECT * FROM product WHERE price > "`
  3. `.append(10000)` → 숫자도 글자로 붙음 → `"SELECT * FROM product WHERE price > 10000"`
  4. `toString()` → 완성된 `String`으로 꺼내 `sql`에 담음. 이 과정에서 새 객체는 거의 만들어지지 않음.

---

## 6. Wrapper 클래스

* **한 줄 비유**: 기본 자료형(`int`, `double`…)을 **포장지로 감싸서 객체로 만든 것**. 객체만 받는 곳(예: `ArrayList`)에 숫자를 넣으려면 포장이 필요함.

| 기본 자료형 | Wrapper 클래스 |
| :--- | :--- |
| `int` | `Integer` |
| `double` | `Double` |
| `char` | `Character` |
| `boolean` | `Boolean` |

* **파싱(Parsing)**: 글자를 숫자로 바꾸는 것. `Double.parseDouble("175.5")` → `175.5`. Scanner나 파일에서 읽은 값은 글자라서, 키·몸무게를 숫자로 바꿔야 BMI를 계산할 수 있음.
* **박싱 / 언박싱**: 기본값 → 포장(Boxing), 포장 → 기본값(Unboxing). 자바가 **알아서** 해 줌(Auto-Boxing/Unboxing). `Integer a = 10;`, `int b = a;` 모두 그냥 됨.
* **주의점**
  * Wrapper는 객체라 `null`이 될 수 있음. `null`을 언박싱하면(`int x = (Integer) null;`) `NullPointerException`.
  * Wrapper끼리 `==`로 비교하면 안 됨. 자바는 **-128~127** 범위의 `Integer`만 미리 만들어 두고 재사용(캐시)해서, 그 범위 안에서는 `==`가 우연히 `true`, 128 이상이면 `false`가 나옴. 값 비교는 `equals()`를 씀.
* **숫자 → 문자열**: `String.valueOf(값)` 또는 `값 + ""`.

---

## 7. 날짜/시간 API (`java.time`)

| 클래스 | 담는 것 | 예 |
| :--- | :--- | :--- |
| `LocalDate` | 날짜 | 2026-12-25 |
| `LocalTime` | 시간 | 14:30 |
| `LocalDateTime` | 날짜 + 시간 | 2026-12-25T14:30 |
| `ZonedDateTime` + `ZoneId` | 날짜 + 시간 + **시간대** | 서울 시간, 뉴욕 시간 |

* **만들기**: `of(...)`로 원하는 시각, `now()`로 지금 시각.
* **꺼내기**: `getYear()`(연), `getMonthValue()`(월, **1~12** 그대로), `getDayOfWeek()`(요일).
* **불변**: `plusDays()`, `minusMinutes()` 같은 메소드는 원본을 고치지 않고 **새 객체를 돌려줌** → 반드시 변수에 받아야 함. `d.plusDays(1);`만 쓰면 아무 변화가 없음.
* **비교**: `isBefore()`(이전인가), `isAfter()`(이후인가), `isEqual()`(같은가).
* **포맷팅 / 파싱**: `DateTimeFormatter.ofPattern("모양")`으로 날짜 글자 모양을 정하고, `format()`(날짜 → 글자), `parse()`(글자 → 날짜)로 바꿈.
  * 패턴 문자: `yyyy`(연 4자리), `MM`(월 2자리), `dd`(일 2자리), `HH`(0~23시), `mm`(분). `MM`(월)과 `mm`(분)은 대소문자로 구분하니 조심.

```java
LocalDateTime trainDeparture = LocalDateTime.of(LocalDate.of(2026, 12, 25), LocalTime.of(14, 30));
LocalDateTime deadline = trainDeparture.minusMinutes(20);   // 새 객체 반환

DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy년 MM월 dd일 HH시 mm분 출발");
String ticket = trainDeparture.format(formatter);           // LocalDateTime → String

LocalDate parsed = LocalDate.parse("20261225", DateTimeFormatter.ofPattern("yyyyMMdd")); // String → LocalDate
```

* **따라가기**
  * `trainDeparture` → 2026-12-25 날짜와 14:30 시간을 합침 → `2026-12-25T14:30`
  * `minusMinutes(20)` → 20분 전 새 객체 → `deadline = 2026-12-25T14:10`. `trainDeparture`는 그대로 14:30.
  * `format(formatter)` → `"2026년 12월 25일 14시 30분 출발"`
  * `LocalDate.parse("20261225", yyyyMMdd)` → 앞 4자리 연, 2자리 월, 2자리 일로 읽음 → `2026-12-25` 날짜 객체.

---

## 느낀 점

* 추상 클래스와 인터페이스를 직접 구현해보며, "공통 기능을 물려주는 부모"와 "지켜야 할 기능의 약속"이라는 역할 차이를 명확히 구분할 수 있게 됨
* `Computer`가 구체 클래스가 아닌 인터페이스 타입에 의존하도록 설계하면, 새로운 장치가 추가되어도 기존 코드를 수정할 필요가 없다는 확장성의 장점을 체감함
* `==`(동일성)와 `equals()`(동등성)의 차이가 `Object`, `String`, Wrapper 전반에 공통으로 적용되는 핵심 개념임을 정리함
* `String`과 `java.time` 객체가 불변이라는 특성을 이해하고, 반복적인 문자열 조립에는 `StringBuilder`를 사용하는 이유를 성능 비교로 확인함
