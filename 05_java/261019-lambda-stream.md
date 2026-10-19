# Java 람다식 · 함수형 인터페이스 · 메소드 참조 · Stream API 정리 (TIL)

## 1. 람다식 (Lambda Expression)

* **개념**: 추상 메소드가 **하나뿐인** 함수형 인터페이스의 구현을 간결하게 표현하는 문법임을 학습함. 매개변수와 실행할 동작만 `(매개변수) -> { 실행할 코드 }` 형태로 작성함.


* **함수형 인터페이스**: 추상 메소드가 하나인 인터페이스이며, `@FunctionalInterface`를 붙이면 추상 메소드를 하나 더 추가했을 때 컴파일 에러로 알려줌.

```java
@FunctionalInterface
public interface Calculator {
    int operate(int a, int b);
//  int anotherOperate(int a, int b);   // 추가하면 컴파일 에러
}
```

* **익명 클래스 vs 람다식**: 같은 덧셈 동작을 익명 클래스로 작성하면 `new`, 메소드 선언, `@Override`까지 써야 하지만, 람다식은 한 줄로 끝남.

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

* **람다가 참조하는 지역 변수**: 람다는 선언된 자리의 지역 변수 값을 가져가 나중에 실행될 수 있음. 그 사이 값이 바뀌면 어느 값을 써야 하는지 흐름이 헷갈리므로, 자바는 람다가 참조하는 지역 변수를 **다시 대입하지 못하게** 함 (effectively final).

```java
int bonus = 5;
Calculator bonusCalculator = (a, b) -> a + b + bonus;
// bonus = 10;   // 다시 대입하면 컴파일 에러
calculate(10, 20, bonusCalculator);    // 35
```

---

## 2. 자주 사용하는 함수형 인터페이스 (`java.util.function`)

| 인터페이스 | 메소드 | 입력 → 반환 | 용도 |
| :--- | :--- | :--- | :--- |
| `Consumer<T>` | `accept(T)` | T → 없음 | 값을 받아서 소비 (출력 등) |
| `Supplier<T>` | `get()` | 없음 → T | 값을 받지 않고 결과를 만들어 반환 |
| `Function<T, R>` | `apply(T)` | T → R | 값을 받아 다른 타입의 결과로 변환 |
| `Predicate<T>` | `test(T)` | T → `boolean` | 조건 검사 |
| `UnaryOperator<T>` | `apply(T)` | T → T | 입력과 반환 타입이 같은 변환 |
| `BiPredicate<T, U>` | `test(T, U)` | T, U → `boolean` | 값 두 개로 조건 검사 |
| `BiFunction<T, U, R>` | `apply(T, U)` | T, U → R | 값 두 개를 받아 결과로 변환 |

* 직접 인터페이스를 만들지 않아도 이미 준비된 인터페이스를 조합해 입력값 처리 흐름을 만들 수 있음을 확인함.

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

---

## 3. 메소드 참조 (Method Reference)

* **개념**: 람다가 새로운 동작을 만들지 않고 **이미 존재하는 메소드 하나를 그대로 호출만** 할 때, 그 호출을 `::`로 더 간결하게 표현하는 문법임을 학습함.

| 형태 | 람다식 | 메소드 참조 |
| :--- | :--- | :--- |
| 클래스::정적메소드 | `value -> Integer.parseInt(value)` | `Integer::parseInt` |
| 객체::인스턴스메소드 | `message -> System.out.println(message)` | `System.out::println` |
| 클래스::인스턴스메소드 | `(left, right) -> left.equals(right)` | `String::equals` |
| 클래스::new (생성자) | `() -> new Account()` | `Account::new` |

* **`String::equals`**: 첫 번째 매개변수(`left`)가 메소드를 호출하는 객체가 되고, 두 번째 매개변수(`right`)가 인자로 전달됨.

```java
BiPredicate<String, String> sameTextReference = String::equals;
sameTextReference.test("hello", "world");   // false
```

* **생성자 참조**: 같은 `Account::new`라도 대입되는 함수형 인터페이스의 **매개변수 개수와 타입**에 따라 호출되는 생성자가 달라짐.

```java
Supplier<Account> newAccount = Account::new;                         // Account()
Function<String, Account> newAccountWithName = Account::new;         // Account(String)
BiFunction<String, Integer, Account> newAccountFull = Account::new;  // Account(String, int)

newAccount.get();                         // 기본 생성자 호출됨
newAccountWithName.apply("홍길동");        // String 생성자 호출됨
newAccountFull.apply("판다", 10000);       // String, int 생성자 호출됨
```

---

## 4. Stream API 기초

* **개념**: 컬렉션에 들어있는 여러 데이터를 조건에 따라 **거르고(filter), 바꾸고(map), 모으는(collect)** 작업을 읽기 좋게 만들어주는 도구임을 학습함. `.stream()`으로 요소를 차례로 처리할 Stream을 만들고, 중간 연산을 이어 붙여 파이프라인을 구성한 뒤 최종 연산으로 결과를 얻음.


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

* **`collect`로 새 List 만들기**: Stream은 원본 컬렉션을 바꾸지 않고, 결과를 새로운 List로 수집함.

```java
List<String> leeFamily = names.stream()
        .filter(name -> name.startsWith("이"))
        .sorted(Comparator.reverseOrder())     // 역순 정렬
        .collect(Collectors.toList());

System.out.println(leeFamily);   // [이순신, 이성계]
System.out.println(names);       // 원본은 그대로 유지
```

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

* **Stream은 한 번만 사용 가능**: 최종 연산이 한 번 실행되면 Stream은 소비되므로, 같은 Stream에 다시 최종 연산을 하면 `IllegalStateException`이 발생함. 다시 처리하려면 `names.stream()`으로 새로 만들어야 함.

```java
Stream<String> stream = names.stream();
stream.count();                        // 7
stream.forEach(System.out::println);   // IllegalStateException
```

---

## 5. Stream 활용 — 온라인 강의 목록 다루기

* `OnlineCourse(title, category, duration, isFree)` 객체 목록으로 실제 데이터 처리 예제를 연습함. 람다 대신 `OnlineCourse::getTitle`, `System.out::println` 같은 메소드 참조를 함께 사용하니 파이프라인이 더 읽기 쉬워짐.


* **조건 여러 개 걸기**: `filter`를 여러 번 이어 붙여 조건을 하나씩 나눠 쓸 수 있음.

```java
// Programming 강의 중 200분을 초과하는 강의 제목 출력
courses.stream()
        .filter(c -> "Programming".equals(c.getCategory()))
        .filter(c -> c.getDuration() > 200)
        .map(OnlineCourse::getTitle)
        .forEach(System.out::println);   // Spring 완전 정복, JPA 끝장내기
```

* **`findFirst()`와 `Optional`**: 조건에 맞는 요소가 없을 수도 있으므로 결과를 `Optional`로 감싸 반환함. `ifPresent()`는 값이 있을 때만 전달한 `Consumer`를 실행하므로 `null` 체크를 직접 하지 않아도 됨.

```java
Optional<OnlineCourse> firstRecommendedCourse = courses.stream()
        .filter(c -> "Programming".equals(c.getCategory()))
        .filter(c -> c.getDuration() > 200)
        .findFirst();

firstRecommendedCourse.ifPresent(c -> System.out.println(c.getTitle()));   // Spring 완전 정복
```

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

* **`Collectors.groupingBy`로 그룹 묶기**: 기준 값(카테고리)을 키로, 해당하는 요소 목록을 값으로 하는 `Map`을 만들어줌.

```java
Map<String, List<OnlineCourse>> courseByCategory = courses.stream()
        .collect(Collectors.groupingBy(OnlineCourse::getCategory));

courseByCategory.forEach((category, courseList) -> {
    System.out.println("[" + category + "]");
    courseList.forEach(c -> System.out.println(" - " + c.getTitle()));
});
```

---

## 느낀 점

* 익명 클래스로 길게 쓰던 코드가 람다식 한 줄로 줄어드는 것을 보고, 람다가 결국 "함수형 인터페이스의 구현 객체"를 짧게 쓰는 방법이라는 점이 이해됨.


* `Consumer`, `Supplier`, `Function`, `Predicate`를 입력과 반환 유무로 구분하니 외우기 쉬웠고, 자바스크립트의 콜백 함수처럼 동작 자체를 값으로 넘기는 개념과 비슷하다고 느낌.


* 같은 `Account::new`가 대입되는 인터페이스에 따라 다른 생성자를 호출하는 것이 신기했고, 메소드 참조도 결국 함수형 인터페이스의 시그니처에 맞춰 연결된다는 점을 알게 됨.


* for/if 중첩 코드를 `filter → map → forEach` 파이프라인으로 바꾸니 "무엇을 하는지"가 위에서 아래로 바로 읽혀서, 자바스크립트의 `filter`/`map` 체이닝과 같은 흐름이라고 느낌.


* Stream은 한 번 쓰면 재사용할 수 없고, `findFirst()`는 `Optional`로 반환한다는 점처럼 자바만의 규칙이 있어서 결과 타입을 의식하며 작성해야겠다고 생각함.
