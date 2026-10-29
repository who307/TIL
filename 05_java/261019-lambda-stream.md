# Java 람다식 · 함수형 인터페이스 · 메소드 참조 · Stream API 정리 (TIL)

## 1. 람다식 (Lambda Expression)

* **한 줄 비유**: 람다식은 **"이름 없는 짧은 함수"를 쪽지에 적어 건네주는 것**임. 원래는 클래스를 만들고, 메소드를 만들고… 해야 했던 걸 `(a, b) -> a + b` 한 줄로 끝냄.
* **함수형 인터페이스**: 추상 메소드가 **딱 하나**인 인터페이스. 람다식은 이 하나뿐인 메소드의 **몸통을 채우는 것**이라, 메소드가 둘 이상이면 어느 걸 채우는지 알 수 없어서 쓸 수 없음.
* **`@FunctionalInterface`**: "이건 함수형 인터페이스야"라는 표시. 실수로 추상 메소드를 하나 더 추가하면 **컴파일 에러**로 알려줌.

```java
@FunctionalInterface
public interface Calculator {
    int operate(int a, int b);
//  int anotherOperate(int a, int b);   // 추가하면 컴파일 에러
}
```

* **익명 클래스 → 람다식으로 줄이는 과정**
  1. 익명 클래스: `new Calculator() { @Override public int operate(int a, int b) { return a + b; } }`
  2. 인터페이스 이름·메소드 이름은 하나뿐이라 자바가 이미 앎 → 지움 → `(int a, int b) -> { return a + b; }`
  3. 매개변수 타입도 인터페이스를 보면 앎 → 지움 → `(a, b) -> { return a + b; }`
  4. 몸통이 `return` 한 줄이면 `{ }`와 `return`도 지움 → `(a, b) -> a + b`

```java
/* 익명 클래스 */
Calculator anonymousAdd = new Calculator() {
    @Override
    public int operate(int a, int b) {
        return a + b;
    }
};

/* 람다식 */
Calculator lambdaAdd = (a, b) -> a + b;

calculate(10, 19, lambdaAdd);          // 29
calculate(10, 19, (a, b) -> a - b);    // 동작 자체를 인자로 바로 전달
calculate(10, 19, (a, b) -> a * b);

public static int calculate(int first, int second, Calculator calculator) {
    return calculator.operate(first, second);
}
```

* **따라가기**
  * `calculate(10, 19, lambdaAdd)` → `calculator`에 "더하기 쪽지"가 들어감 → `calculator.operate(10, 19)` → `10 + 19` → `29`
  * `calculate(10, 19, (a, b) -> a - b)` → "빼기 쪽지"를 바로 만들어 넘김 → `-9`
  * `calculate(10, 19, (a, b) -> a * b)` → `190`
  * `calculate` 메소드는 한 번만 만들었는데, 넘기는 **동작(쪽지)** 만 바꿔서 여러 계산을 시킴.

* **람다가 쓰는 지역 변수는 다시 대입 금지 (effectively final)**
  * 람다는 만들어진 자리의 지역 변수 값을 **가져가서**, 나중에 실행될 수 있음.
  * 그 사이에 변수가 바뀌면 "만들 때 값을 써야 해? 지금 값을 써야 해?" 헷갈림 → 자바는 람다가 쓰는 지역 변수를 **다시 대입하지 못하게** 막음. `final`을 안 붙여도 실제로 안 바뀌면 되는데, 이를 "사실상 final(effectively final)"이라고 부름.

```java
int bonus = 5;
Calculator bonusCalculator = (a, b) -> a + b + bonus;
// bonus = 10;   // 다시 대입하면 컴파일 에러
calculate(10, 20, bonusCalculator);    // 35
```

* **따라가기**: `bonus = 5`를 람다가 기억 → `10 + 20 + 5` → `35`. `bonus = 10`을 쓰면 컴파일 에러가 나서 주석 처리함.

---

## 2. 자주 사용하는 함수형 인터페이스 (`java.util.function`)

* 매번 `Calculator` 같은 인터페이스를 직접 만들지 않아도, 자바가 **자주 쓰는 모양을 미리 만들어 둠**. 입력과 반환이 **있는지 없는지**로 외우면 쉬움.

| 인터페이스 | 메소드 | 입력 → 반환 | 용도 | 비유 |
| :--- | :--- | :--- | :--- | :--- |
| `Consumer<T>` | `accept(T)` | T → 없음 | 값을 받아서 소비 (출력 등) | 먹기만 하는 사람 |
| `Supplier<T>` | `get()` | 없음 → T | 값을 받지 않고 결과를 만들어 반환 | 주기만 하는 자판기 |
| `Function<T, R>` | `apply(T)` | T → R | 값을 받아 다른 타입의 결과로 변환 | 밀가루 → 빵 기계 |
| `Predicate<T>` | `test(T)` | T → `boolean` | 조건 검사 | O/X 심판 |
| `UnaryOperator<T>` | `apply(T)` | T → T | 입력과 반환 타입이 같은 변환 | 사진 보정 (사진 → 사진) |
| `BiPredicate<T, U>` | `test(T, U)` | T, U → `boolean` | 값 두 개로 조건 검사 | 두 개 비교 심판 |
| `BiFunction<T, U, R>` | `apply(T, U)` | T, U → R | 값 두 개를 받아 결과로 변환 | 재료 두 개 → 요리 |

```java
String input = "   java   ";

Predicate<String> hasText = value -> value != null && !value.trim().isEmpty();
UnaryOperator<String> normalize = value -> value.trim().toUpperCase();
Consumer<String> printer = value -> System.out.println("처리결과: " + value);
Function<String, Integer> length = value -> value.length();
Supplier<String> defaultValue = () -> "GUEST";

hasText.test(input);                          // true
String normalized = normalize.apply(input);   // "JAVA"
printer.accept(normalized);                   // 처리결과: JAVA
length.apply(normalized);                     // 4

// 빈 입력이면 기본값 사용
String blankInput = "   ";
String result = hasText.test(blankInput) ? normalize.apply(blankInput) : defaultValue.get();
printer.accept(result);                       // 처리결과: GUEST
```

* **따라가기**
  1. `hasText.test("   java   ")` → `null` 아님, `trim()` → `"java"`, 비어 있지 않음 → `true`
  2. `normalize.apply(input)` → `trim()` → `"java"` → `toUpperCase()` → `"JAVA"`
  3. `printer.accept("JAVA")` → `처리결과: JAVA` 출력 (반환값 없음)
  4. `length.apply("JAVA")` → `4` (String → Integer, 타입이 바뀜)
  5. `blankInput = "   "` → `hasText.test` → `trim()`하면 `""` → 비어 있음 → `false`
  6. 삼항 연산자에서 `false` → `defaultValue.get()` → `"GUEST"` → `처리결과: GUEST`

---

## 3. 메소드 참조 (Method Reference)

* **개념**: 람다가 **이미 있는 메소드 하나를 그대로 부르기만** 할 때, `클래스::메소드`처럼 더 짧게 쓰는 문법. "이 일은 저 사람한테 맡겨"라고 **이름만 가리키는** 것.

| 형태 | 람다식 | 메소드 참조 |
| :--- | :--- | :--- |
| 클래스::정적메소드 | `value -> Integer.parseInt(value)` | `Integer::parseInt` |
| 객체::인스턴스메소드 | `message -> System.out.println(message)` | `System.out::println` |
| 클래스::인스턴스메소드 | `(left, right) -> left.equals(right)` | `String::equals` |
| 클래스::new (생성자) | `() -> new Account()` | `Account::new` |

* **`String::equals`처럼 클래스 이름 + 인스턴스 메소드인 경우**: 첫 번째 매개변수(`left`)가 메소드를 **부르는 주인**이 되고, 두 번째(`right`)가 **인자**로 들어감.

```java
BiPredicate<String, String> sameTextReference = String::equals;
sameTextReference.test("hello", "world");   // false
```

* **따라가기**: `test("hello", "world")` → `"hello".equals("world")` → `false`

* **생성자 참조**: 같은 `Account::new`라도 **담기는 인터페이스의 입력 개수·타입**에 맞는 생성자가 골라짐.

```java
Supplier<Account> newAccount = Account::new;                         // Account()
Function<String, Account> newAccountWithName = Account::new;         // Account(String)
BiFunction<String, Integer, Account> newAccountFull = Account::new;  // Account(String, int)

newAccount.get();                         // 기본 생성자 호출됨
newAccountWithName.apply("홍길동");        // String 생성자 호출됨
newAccountFull.apply("판다", 10000);       // String, int 생성자 호출됨
```

* **따라가기**
  * `Supplier`는 입력이 **없음** → 매개변수 없는 `Account()` → `newAccount.get()`
  * `Function<String, ...>`은 입력이 **String 하나** → `Account(String)` → `apply("홍길동")`
  * `BiFunction<String, Integer, ...>`은 입력이 **String, Integer 두 개** → `Account(String, int)` → `apply("판다", 10000)` (Integer는 자동으로 int로 풀림)

---

## 4. Stream API 기초

* **한 줄 비유**: Stream은 **공장 컨베이어 벨트**임. 상자(컬렉션)에서 물건을 벨트에 올리면 → **거르는 기계(filter)** → **바꾸는 기계(map)** → **포장 기계(collect)** 를 차례로 지나 결과가 나옴.
* **구성**
  1. **만들기**: `컬렉션.stream()` → 벨트에 올림.
  2. **중간 연산**: `filter`, `map`, `sorted`… → 기계를 원하는 만큼 이어 붙임(파이프라인).
  3. **최종 연산**: `forEach`, `collect`, `count`… → 결과를 꺼냄. **최종 연산을 해야 벨트가 실제로 돌아감**.

* **for/if vs Stream**: "이름이 세 글자인 사람만 골라 대괄호를 붙여 출력"하는 같은 동작을 비교함.

```java
List<String> names = Arrays.asList("홍길동", "유관순", "이순신", "신사임당", "장보고", "강감찬", "이성계");

/* for/if 사용 */
for (String name : names) {
    if (name.length() == 3) {
        String decoratedName = "[" + name + "]";
        System.out.println(decoratedName);
    }
}

/* Stream 사용 */
names.stream()
        .filter(name -> name.length() == 3)    // 거르기
        .map(name -> "[" + name + "]")         // 바꾸기
        .forEach(decoratedName -> System.out.println(decoratedName));
```

* **따라가기 (Stream)**
  1. 7명이 벨트에 올라감.
  2. `filter(길이 == 3)` → "신사임당"(4글자)만 걸러짐 → 6명 통과.
  3. `map("[" + name + "]")` → `[홍길동]`, `[유관순]`, `[이순신]`, `[장보고]`, `[강감찬]`, `[이성계]`
  4. `forEach` → 하나씩 출력. for/if 버전과 결과가 같지만 "거르고 → 바꾸고 → 출력"이 위에서 아래로 바로 읽힘.

* **`collect`로 새 List 만들기**: Stream은 **원본을 바꾸지 않고**, 결과를 새 List로 모아줌.

```java
List<String> leeFamily = names.stream()
        .filter(name -> name.startsWith("이"))
        .sorted(Comparator.reverseOrder())     // 역순 정렬
        .collect(Collectors.toList());

System.out.println(leeFamily);   // [이순신, 이성계]
System.out.println(names);       // 원본은 그대로 유지
```

* **따라가기**
  1. `filter(startsWith("이"))` → "이순신", "이성계"
  2. `sorted(Comparator.reverseOrder())` → 가나다 역순. 둘째 글자 "성" < "순"이므로 역순은 `이순신, 이성계`
  3. `collect(Collectors.toList())` → 새 List `[이순신, 이성계]`
  4. `names`는 처음 7명 그대로.

* **중간 연산 / 최종 연산**

| 구분 | 메소드 | 설명 |
| :--- | :--- | :--- |
| 중간 연산 | `filter()` | 조건에 맞는 요소만 통과 |
| | `map()` | 요소를 다른 값으로 변환 |
| | `sorted()` | 정렬 |
| | `distinct()` | 중복 제거 |
| | `mapToInt()` | `IntStream`으로 변환 (`sum()`, `average()` 사용 가능) |
| 최종 연산 | `forEach()` | 요소마다 동작 실행 |
| | `collect()` | 결과를 컬렉션으로 수집 |
| | `count()` | 남은 요소 개수를 `long`으로 반환 |
| | `anyMatch()` | 하나라도 조건을 만족하면 `true` |
| | `allMatch()` | 모든 요소가 조건을 만족하면 `true` |
| | `findFirst()` | 조건에 맞는 첫 번째 요소를 `Optional`로 반환 |
| | `sum()` | `IntStream`의 합계 |

```java
names.stream().filter(name -> name.startsWith("이")).count();   // 2
names.stream().anyMatch(name -> name.equals("장보고"));          // true
names.stream().allMatch(name -> name.length() == 3);            // false ("신사임당")
```

* **따라가기**
  * `count()` → "이"로 시작하는 사람 → 이순신, 이성계 → `2`
  * `anyMatch(장보고)` → 한 명이라도 있나? → 있음 → `true`
  * `allMatch(길이 == 3)` → 모두 세 글자인가? → "신사임당"에서 실패 → `false`

* **Stream은 한 번만 쓸 수 있음**: 최종 연산을 한 번 하면 벨트가 **다 소비**됨. 같은 Stream에 또 최종 연산을 하면 `IllegalStateException` → 다시 하려면 `names.stream()`으로 **새로** 만들어야 함.

```java
Stream<String> stream = names.stream();
stream.count();                        // 7
stream.forEach(System.out::println);   // IllegalStateException
```

---

## 5. Stream 활용 — 온라인 강의 목록 다루기

* `OnlineCourse(title, category, duration, isFree)` 객체 목록으로 실제 데이터 처리 예제를 연습함. `OnlineCourse::getTitle`, `System.out::println` 같은 **메소드 참조**를 함께 쓰니 파이프라인이 더 읽기 쉬워짐.
* **조건 여러 개 걸기**: `filter`를 여러 번 이어 붙이면 조건을 하나씩 나눠 쓸 수 있음(두 조건을 모두 통과해야 함 = AND).

```java
// Programming 강의 중 200분을 초과하는 강의 제목 출력
courses.stream()
        .filter(c -> "Programming".equals(c.getCategory()))
        .filter(c -> c.getDuration() > 200)
        .map(OnlineCourse::getTitle)
        .forEach(System.out::println);   // Spring 완전 정복, JPA 끝장내기
```

* **따라가기**: 카테고리가 Programming인 강의만 → 그중 200분 초과만 → 강의 객체를 제목 글자로 바꿈 → 출력: `Spring 완전 정복`, `JPA 끝장내기`
  * `"Programming".equals(c.getCategory())`처럼 리터럴 쪽에서 `equals`를 부르면 카테고리가 `null`이어도 에러가 안 남.

* **`findFirst()`와 `Optional`**
  * 조건에 맞는 강의가 **없을 수도** 있음 → 결과를 `null` 대신 **`Optional`(있을 수도, 없을 수도 있는 상자)** 에 담아 돌려줌.
  * `ifPresent(동작)`: 상자 안에 값이 **있을 때만** 동작을 실행함 → `null` 검사를 직접 안 해도 됨.

```java
Optional<OnlineCourse> firstRecommendedCourse = courses.stream()
        .filter(c -> "Programming".equals(c.getCategory()))
        .filter(c -> c.getDuration() > 200)
        .findFirst();

firstRecommendedCourse.ifPresent(c -> System.out.println(c.getTitle()));   // Spring 완전 정복
```

* **따라가기**: 위와 같은 조건 → 처음 찾은 강의 하나(`Spring 완전 정복`)를 `Optional`에 담음 → 값이 있으므로 `ifPresent`가 제목 출력. 조건에 맞는 강의가 없었다면 빈 `Optional`이라 아무것도 출력하지 않음(에러도 없음).

* **정렬 / 중복 제거 / 합계**

```java
// 무료 강의를 수강 시간순으로 정렬
courses.stream()
        .filter(OnlineCourse::isFree)
        .sorted((c1, c2) -> Integer.compare(c1.getDuration(), c2.getDuration()))
        .forEach(System.out::println);   // SQL 첫걸음(80) → 포토샵 마스터(90) → 일러스트레이터 시작하기(110)

// 중복 없는 카테고리 목록
courses.stream()
        .map(OnlineCourse::getCategory)
        .distinct()
        .forEach(System.out::println);   // Programming, Design, Database

// 무료 강의 총 수강 시간
int totalDurationOfFree = courses.stream()
        .filter(OnlineCourse::isFree)
        .mapToInt(OnlineCourse::getDuration)   // Stream<OnlineCourse> → IntStream
        .sum();                                // 280
```

* **따라가기**
  * **정렬**: 무료 강의만 남김 → `Integer.compare(시간1, 시간2)`가 음수면 c1이 앞 → 짧은 순서 → `SQL 첫걸음(80)` → `포토샵 마스터(90)` → `일러스트레이터 시작하기(110)`
  * **중복 제거**: 강의마다 카테고리 글자만 뽑음 → `Programming, Programming, Design, …` → `distinct()`로 겹친 것 제거 → `Programming`, `Design`, `Database`
  * **합계**: 무료 강의만 → 수강 시간 숫자만 뽑아 `IntStream`으로 → `80 + 90 + 110` → `280`. 일반 `Stream`에는 `sum()`이 없어서 `mapToInt`로 숫자 전용 벨트로 바꿔야 함.

* **`Collectors.groupingBy`로 그룹 묶기**: 기준 값(카테고리)을 **key**로, 그 카테고리에 속한 강의들의 **List를 value**로 하는 `Map`을 만들어 줌. 반 학생들을 **번호표 대신 반 이름별로 줄 세우는** 것과 같음.

```java
Map<String, List<OnlineCourse>> courseByCategory = courses.stream()
        .collect(Collectors.groupingBy(OnlineCourse::getCategory));

courseByCategory.forEach((category, courseList) -> {
    System.out.println("[" + category + "]");
    courseList.forEach(c -> System.out.println(" - " + c.getTitle()));
});
```

* **따라가기**
  1. 강의를 하나씩 보며 카테고리별 바구니에 넣음 → `{ "Programming": [Spring 완전 정복, JPA 끝장내기, …], "Design": [포토샵 마스터, …], "Database": [SQL 첫걸음, …] }`
  2. `Map.forEach((key, value) -> ...)` → 카테고리마다 `[Programming]`을 찍고, 그 아래 ` - 강의제목`을 줄줄이 출력.

---

## 느낀 점

* 익명 클래스로 길게 쓰던 코드가 람다식 한 줄로 줄어드는 것을 보고, 람다가 결국 "함수형 인터페이스의 구현 객체"를 짧게 쓰는 방법이라는 점이 이해됨
* `Consumer`, `Supplier`, `Function`, `Predicate`를 입력과 반환 유무로 구분하니 외우기 쉬웠고, 자바스크립트의 콜백 함수처럼 동작 자체를 값으로 넘기는 개념과 비슷하다고 느낌
* 같은 `Account::new`가 대입되는 인터페이스에 따라 다른 생성자를 호출하는 것이 신기했고, 메소드 참조도 결국 함수형 인터페이스의 시그니처에 맞춰 연결된다는 점을 알게 됨
* for/if 중첩 코드를 `filter → map → forEach` 파이프라인으로 바꾸니 "무엇을 하는지"가 위에서 아래로 바로 읽혀서, 자바스크립트의 `filter`/`map` 체이닝과 같은 흐름이라고 느낌
* Stream은 한 번 쓰면 재사용할 수 없고, `findFirst()`는 `Optional`로 반환한다는 점처럼 자바만의 규칙이 있어서 결과 타입을 의식하며 작성해야겠다고 생각함
