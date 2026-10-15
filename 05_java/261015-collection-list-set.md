# Java 컬렉션 프레임워크 · List · Stack/Queue · 와일드카드 · Set 정리 (TIL)

## 1. 컬렉션 프레임워크 (Collection Framework)

* **개념**: 여러 개의 데이터(객체)를 효과적으로 관리하기 위해 자바에서 제공하는 클래스들의 묶음으로, 크게 `List`, `Set`, `Map` 인터페이스로 분류됨을 이해함.


* **`Collections` 클래스**: 컬렉션을 다루는 데 유용한 static 메소드(`sort()` 등)를 제공하는 유틸리티 클래스이며, `Collection` 인터페이스와는 다르다는 점을 구분함.

| 구분 | 순서(인덱스) | 중복 허용 | 대표 구현 클래스 |
| :--- | :--- | :--- | :--- |
| `List` | O | O | `ArrayList`, `LinkedList` |
| `Set` | X | X | `HashSet`, `LinkedHashSet`, `TreeSet` |

---

## 2. `ArrayList`

* **개념**: `List` 인터페이스의 가장 대표적인 구현 클래스로, 내부적으로 배열을 이용하며 순서(인덱스)가 있고 중복 저장을 허용함을 학습함.


* **제네릭의 필요성**: 제네릭을 지정하지 않으면 요소가 `Object`로 취급되어 서로 다른 타입이 섞여 들어가고, 값을 꺼낼 때마다 형변환이 필요함. `List<String>`처럼 타입을 지정하면 다른 타입 추가 시 컴파일 에러로 막아줌.

```java
List alist = new ArrayList();          // 제네릭 X → Object로 취급
alist.add("apple");
alist.add(123);
String fruit = (String) alist.get(0);  // 매번 형변환 필요

List<String> stringList = new ArrayList<>();
// stringList.add(123);                // 컴파일 에러
```

* **주요 메소드**

| 메소드 | 기능 |
| :--- | :--- |
| `add(값)` | 마지막에 값 추가 |
| `get(인덱스)` | 해당 위치의 값 조회 |
| `set(인덱스, 값)` | 해당 위치의 값 수정 |
| `remove(인덱스 또는 값)` | 해당 위치 또는 해당 값 삭제 |
| `size()` | 현재 저장된 요소 개수 |

* **정렬**: `Collections.sort(list)`로 오름차순, `list.sort(Comparator.reverseOrder())`로 내림차순 정렬함.

---

## 3. 객체 정렬 — `Comparable` / `Comparator`

* **`Comparable` (기본 정렬 기준)**: `BookDTO`가 `Comparable<BookDTO>`를 구현하고 `compareTo()`를 오버라이딩하여 책 번호 오름차순을 기본 정렬 기준으로 정함. `Collections.sort(bookList)`는 이 기준을 사용함.
  * 음수 반환 → 자리 유지 / 양수 반환 → 자리 바꿈 / 0 → 자리 유지


* **`Comparator` (별도 정렬 기준)**: 기본값과 다른 기준으로 정렬하고 싶을 때 `Comparator<BookDTO>`를 구현한 `AscendingPrice` 클래스를 만들어 `list.sort()`에 전달함.

```java
// BookDTO - 기본 정렬 기준: 번호 오름차순
@Override
public int compareTo(BookDTO o) {
    return Integer.compare(this.number, o.getNumber());
}

// AscendingPrice - 별도 정렬 기준: 가격 오름차순
@Override
public int compare(BookDTO o1, BookDTO o2) {
    return Integer.compare(o1.getPrice(), o2.getPrice());
}

Collections.sort(bookList);              // 번호순
bookList.sort(new AscendingPrice());     // 가격순
```

| 구분 | `Comparable` | `Comparator` |
| :--- | :--- | :--- |
| 위치 | 정렬 대상 클래스 내부에서 구현 | 별도 클래스로 분리해 구현 |
| 메소드 | `compareTo(T o)` | `compare(T o1, T o2)` |
| 용도 | 기본 정렬 기준 | 기본과 다른 기준으로 정렬 |

---

## 4. `LinkedList`

* **개념**: `ArrayList`가 내부적으로 배열을 사용하는 것과 달리, **이중 연결 리스트** 방식으로 각 요소가 자신의 앞/뒤 요소만 기억하며 데이터를 관리함을 학습함.


* **특징**: 중간에 데이터를 추가/삭제할 때 앞뒤 연결만 바꾸면 되므로 빠르지만, 특정 인덱스를 찾으려면 처음부터 순서대로 따라가야 하므로 조회가 느림. 추가/삭제가 많다는 이유만으로 항상 `ArrayList`보다 빠른 것은 아님을 파악함.

| 구분 | `ArrayList` | `LinkedList` |
| :--- | :--- | :--- |
| 내부 구조 | 배열 | 이중 연결 리스트 |
| 인덱스 조회 | 빠름 | 느림 (순차 탐색) |
| 중간 추가/삭제 | 요소 이동 필요 | 연결만 변경 |

---

## 5. Stack과 Queue — `ArrayDeque`

* **Stack (LIFO)**: 후입선출 자료구조로, 브라우저의 '뒤로 가기'나 '실행 취소' 기능에 쓰임. 오래된 `Stack` 클래스 대신 `Deque` 인터페이스의 구현체인 `ArrayDeque`를 스택처럼 사용하는 방식이 일반적임을 학습함.
  * `push()` 추가 / `peek()` 꼭대기 확인 / `pop()` 꼭대기 꺼내고 제거
  * 비어있을 때 `pop()`을 호출하면 예외가 발생하므로 `isEmpty()`로 확인 후 꺼냄.


* **Queue (FIFO)**: 선입선출 자료구조로, `Queue`는 인터페이스이므로 구현 클래스인 `ArrayDeque`로 객체를 생성함.
  * `offer()` 맨 뒤에 추가 / `peek()` 맨 앞 확인 / `poll()` 맨 앞 꺼내고 제거
  * 비어있을 때 `poll()`은 예외 대신 `null`을 반환함.

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2); stack.push(3);
stack.pop();                 // 3 (마지막에 넣은 값)

Queue<String> queue = new ArrayDeque<>();
queue.offer("first"); queue.offer("second");
queue.poll();                // "first" (먼저 넣은 값)
```

| 구분 | Stack | Queue |
| :--- | :--- | :--- |
| 방식 | 후입선출 (LIFO) | 선입선출 (FIFO) |
| 추가 | `push()` | `offer()` |
| 확인 | `peek()` | `peek()` |
| 꺼내기 | `pop()` (비면 예외) | `poll()` (비면 `null`) |

---

## 6. 제네릭 와일드카드

* **개념**: 호출할 때 들어오는 정확한 요소 타입은 달라도, 메소드가 허용할 타입의 범위를 정하는 문법임을 학습함.


* **제네릭은 상속 관계가 적용되지 않음**: `Integer`가 `Number`의 자식이어도 `List<Integer>`를 `List<Number>`에 대입할 수 없으므로 와일드카드가 필요함.

| 문법 | 의미 | 활용 |
| :--- | :--- | :--- |
| `<?>` | 모든 타입 허용 | 요소를 `Object`로 읽기만 할 때 (출력 등) |
| `<? extends Number>` | `Number`와 그 하위 타입만 허용 (상한 제한) | 값을 꺼내 `Number`로 사용할 때 (합계 등) |
| `<? super Integer>` | `Integer`와 그 상위 타입만 허용 (하한 제한) | `Integer` 값을 넣을 때 |

```java
private static double sum(List<? extends Number> list) {
    double total = 0;
    for (Number number : list) total += number.doubleValue();
    return total;
}
sum(integerList);   // List<Integer> OK
sum(doubleList);    // List<Double>  OK

private static void addInteger(List<? super Integer> list) {
    list.add(100);  // List<Number>, List<Object>에도 안전하게 추가 가능
}
```

---

## 7. `Set`

* **개념**: 순서를 보장하지 않고 중복을 허용하지 않는 컬렉션으로, 같은 값을 여러 번 `add()`해도 하나만 저장됨을 `ArrayList`와 비교하며 확인함.


* **순회 방법**: 인덱스가 없으므로 `get()` 대신 향상된 for문 또는 `Iterator`로 순회함.
  * `iterator()`: 컬렉션을 순회할 수 있는 `Iterator` 객체 반환
  * `hasNext()`: 다음에 꺼낼 요소가 있으면 `true` / `next()`: 다음 요소를 꺼냄

```java
Set<String> hset = new HashSet<>();
hset.add("java"); hset.add("html"); hset.add("css"); hset.add("java");
// [css, java, html] → 순서 보장 X, "java" 중복 제거

Iterator<String> iter = hset.iterator();
while (iter.hasNext()) {
    System.out.println(iter.next());
}
```

| 구현 클래스 | 중복 | 순서 |
| :--- | :--- | :--- |
| `HashSet` | 불가 | 보장 X |
| `LinkedHashSet` | 불가 | 입력된 순서 유지 |
| `TreeSet` | 불가 | 저장할 때마다 자동 오름차순 정렬 |

---

## 느낀 점

* 같은 "여러 데이터 저장"이라도 순서·중복 허용 여부에 따라 `List`와 `Set`을 골라 써야 한다는 기준이 생김.


* `ArrayList`와 `LinkedList`의 내부 구조 차이를 이해하니, "추가/삭제가 많으면 무조건 `LinkedList`"가 아니라 조회 비용까지 함께 따져야 한다는 점을 알게 됨.


* `Comparable`(기본 기준)과 `Comparator`(별도 기준)를 나눠 구현하면서, 정렬 기준을 클래스 밖으로 분리하면 대상 클래스를 수정하지 않고도 여러 기준으로 정렬할 수 있다는 장점을 체감함.


* 어제 배운 인터페이스 다형성이 `List<String> list = new ArrayList<>()`, `Queue<String> q = new ArrayDeque<>()`처럼 컬렉션 선언 전반에 그대로 쓰이고 있음을 확인함.
