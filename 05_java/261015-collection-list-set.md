# Java 컬렉션 프레임워크 · List · Stack/Queue · 와일드카드 · Set 정리 (TIL)

## 1. 컬렉션 프레임워크 (Collection Framework)

* **한 줄 비유**: 배열이 **크기가 정해진 달걀판**이라면, 컬렉션은 **넣는 만큼 늘어나는 가방**들임. 가방마다 성격이 다름(순서대로 넣는 가방, 같은 물건은 하나만 넣는 가방 등).
* **컬렉션 프레임워크**: 여러 데이터를 담고 꺼내는 클래스들을 자바가 미리 만들어 둔 묶음. 크게 `List`, `Set`, `Map` 인터페이스로 나뉨.
* **`Collections` vs `Collection`**: 이름이 비슷하지만 다름.
  * `Collection`(s 없음): `List`, `Set`의 부모 **인터페이스**.
  * `Collections`(s 있음): 정렬(`sort()`) 같은 편리한 `static` 메소드를 모아둔 **도구 클래스**.

| 구분 | 순서(인덱스) | 중복 허용 | 대표 구현 클래스 |
| :--- | :--- | :--- | :--- |
| `List` | O | O | `ArrayList`, `LinkedList` |
| `Set` | X | X | `HashSet`, `LinkedHashSet`, `TreeSet` |

---

## 2. `ArrayList`

* **개념**: `List`의 가장 대표적인 구현 클래스. 안쪽에 **배열**을 가지고 있다가, 칸이 모자라면 더 큰 배열을 새로 만들어 옮겨 담음. 순서(0번부터)가 있고 같은 값을 여러 번 넣을 수 있음.
* **제네릭(Generic) `<타입>`**: 가방에 **"이 가방엔 String만"** 이라고 라벨을 붙이는 것.
  * 라벨이 없으면: 모든 걸 `Object`로 받아서 문자열과 숫자가 뒤섞이고, 꺼낼 때마다 `(String)`처럼 형변환해야 함. 잘못 꺼내면 실행 중에 에러가 남.
  * 라벨이 있으면: 다른 타입을 넣으려는 순간 **컴파일 에러**로 막아줌. 꺼낼 때 형변환도 필요 없음.

```java
List alist = new ArrayList();          // 제네릭 X → Object로 취급
alist.add("apple");
alist.add(123);
String fruit = (String) alist.get(0);  // 매번 형변환 필요

List<String> stringList = new ArrayList<>();
// stringList.add(123);                // 컴파일 에러
```

* **한 줄씩 보기**
  * `List alist = new ArrayList();` → 라벨 없는 가방.
  * `add("apple")`, `add(123)` → 글자와 숫자가 같이 들어감 → `["apple", 123]`
  * `(String) alist.get(0)` → 꺼낸 값이 `Object`라 `String`으로 바꿔야 씀. `(String) alist.get(1)`이었다면 숫자를 글자로 바꾸려다 `ClassCastException`.
  * `List<String> stringList` → String 전용 가방. `<>` 안을 비워도 왼쪽을 보고 자바가 알아서 채움.
  * `stringList.add(123)` → 정수는 못 넣음 → 컴파일 에러(실행 전에 미리 잡힘).
* `List<String> list = new ArrayList<>()`처럼 **왼쪽은 인터페이스(`List`)** 로 쓰는 게 보통임. 나중에 `LinkedList`로 바꿔도 다른 코드를 안 고쳐도 됨.

* **주요 메소드**

| 메소드 | 기능 |
| :--- | :--- |
| `add(값)` | 마지막에 값 추가 |
| `get(인덱스)` | 해당 위치의 값 조회 |
| `set(인덱스, 값)` | 해당 위치의 값 수정 |
| `remove(인덱스 또는 값)` | 해당 위치 또는 해당 값 삭제 (뒤의 값들이 한 칸씩 당겨짐) |
| `size()` | 현재 저장된 요소 개수 (배열의 `length`와 같은 역할) |

* **정렬**: `Collections.sort(list)` → 오름차순, `list.sort(Comparator.reverseOrder())` → 내림차순.

---

## 3. 객체 정렬 — `Comparable` / `Comparator`

* **문제**: 숫자나 글자는 크기 비교 규칙이 있지만, `BookDTO`(책) 객체는 "어떤 책이 더 작은지" 자바가 모름 → **비교 규칙을 알려줘야** 정렬할 수 있음.
* **비교 메소드의 약속**: 두 값을 비교해서
  * **음수**를 돌려주면 → 앞의 것(`this` 또는 `o1`)이 **앞에** 와야 함
  * **양수**를 돌려주면 → 앞의 것이 **뒤에** 와야 함
  * **0**을 돌려주면 → 둘의 순서가 **같다**고 봄
  * `Integer.compare(a, b)`는 a<b면 음수, a>b면 양수, 같으면 0을 돌려줘서 이 약속에 딱 맞음.
* **`Comparable` (기본 정렬 기준)**: 책 클래스 **안에** `compareTo()`를 만들어 "책은 원래 번호순이야"라고 정해둠. `Collections.sort(bookList)`는 이 기준을 씀.
* **`Comparator` (별도 정렬 기준)**: 책 클래스는 그대로 두고, **바깥에** "가격순 비교기"(`AscendingPrice`)를 따로 만들어 `list.sort()`에 넘김. 필요할 때마다 비교기만 바꿔 끼우면 됨.

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

* **따라가기** (책 A: 번호 3·가격 1만, 책 B: 번호 1·가격 3만)
  * `A.compareTo(B)` → `Integer.compare(3, 1)` → 양수 → A가 B **뒤로** → 번호순 결과 `[B, A]`
  * `compare(A, B)` (가격 비교기) → `Integer.compare(10000, 30000)` → 음수 → A가 **앞으로** → 가격순 결과 `[A, B]`

| 구분 | `Comparable` | `Comparator` |
| :--- | :--- | :--- |
| 위치 | 정렬 대상 클래스 내부에서 구현 | 별도 클래스로 분리해 구현 |
| 메소드 | `compareTo(T o)` | `compare(T o1, T o2)` |
| 용도 | 기본 정렬 기준 | 기본과 다른 기준으로 정렬 |

---

## 4. `LinkedList`

* **개념**: `ArrayList`가 배열(붙어 있는 칸)을 쓰는 것과 달리, **이중 연결 리스트**로 데이터를 관리함. 각 칸이 **앞 칸과 뒤 칸이 누구인지만** 기억하고 손을 잡고 있는 **기차놀이**와 같음.
* **장단점**
  * **중간에 넣기/빼기**: 앞뒤 사람의 손만 바꿔 잡으면 됨 → 빠름. (`ArrayList`는 뒤의 칸들을 전부 한 칸씩 밀거나 당겨야 함)
  * **번호로 찾기**: "7번째 사람"을 찾으려면 맨 앞부터 한 명씩 세어 가야 함 → 느림. (`ArrayList`는 번호로 바로 감)
  * 그래서 "추가/삭제가 많으면 무조건 `LinkedList`"는 아님. 넣을 **위치를 찾는 데** 시간이 걸리면 결국 느려짐.

| 구분 | `ArrayList` | `LinkedList` |
| :--- | :--- | :--- |
| 내부 구조 | 배열 | 이중 연결 리스트 |
| 인덱스 조회 | 빠름 | 느림 (순차 탐색) |
| 중간 추가/삭제 | 요소 이동 필요 | 연결만 변경 |

---

## 5. Stack과 Queue — `ArrayDeque`

* **Stack (LIFO, 후입선출)**: **나중에 넣은 것이 먼저 나옴**. 접시를 쌓으면 맨 위(마지막에 올린) 접시부터 꺼내는 것과 같음. 브라우저 '뒤로 가기', '실행 취소(Ctrl+Z)'에 쓰임.
  * 옛날 `Stack` 클래스 대신 `Deque` 인터페이스의 구현체인 **`ArrayDeque`** 를 스택처럼 쓰는 게 일반적임.
  * `push()`: 맨 위에 올림 / `peek()`: 맨 위를 **보기만** 함 / `pop()`: 맨 위를 **꺼내고 없앰**
  * 비어 있을 때 `pop()`을 하면 예외가 나므로 `isEmpty()`로 먼저 확인함.
* **Queue (FIFO, 선입선출)**: **먼저 넣은 것이 먼저 나옴**. 매점 줄과 같음. `Queue`는 인터페이스라 구현 클래스 `ArrayDeque`로 만듦.
  * `offer()`: 줄 맨 뒤에 섬 / `peek()`: 맨 앞을 보기만 함 / `poll()`: 맨 앞을 꺼내고 없앰
  * 비어 있을 때 `poll()`은 예외 대신 `null`을 돌려줌.

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2); stack.push(3);
stack.pop();                 // 3 (마지막에 넣은 값)

Queue<String> queue = new ArrayDeque<>();
queue.offer("first"); queue.offer("second");
queue.poll();                // "first" (먼저 넣은 값)
```

* **따라가기**
  * 스택: `push(1)` → `[1]` → `push(2)` → `[1, 2]` → `push(3)` → `[1, 2, 3]` (3이 맨 위) → `pop()` → `3`을 꺼냄 → `[1, 2]`
  * 큐: `offer("first")` → `[first]` → `offer("second")` → `[first, second]` → `poll()` → 맨 앞 `"first"`를 꺼냄 → `[second]`

| 구분 | Stack | Queue |
| :--- | :--- | :--- |
| 방식 | 후입선출 (LIFO) | 선입선출 (FIFO) |
| 추가 | `push()` | `offer()` |
| 확인 | `peek()` | `peek()` |
| 꺼내기 | `pop()` (비면 예외) | `poll()` (비면 `null`) |

---

## 6. 제네릭 와일드카드

* **문제**: `Integer`는 `Number`의 자식이지만, `List<Integer>`는 `List<Number>`의 자식이 **아님**. 그래서 `sum(List<Number> list)`에 `List<Integer>`를 넘길 수 없음. "과일 바구니"를 받는 곳에 "사과 바구니"를 못 주는 셈.
* **와일드카드 `?`**: "정확한 타입은 몰라도 **이 범위 안이면** 받을게"라고 범위를 정하는 문법.

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

* **따라가기**
  * `sum(integerList)` (값 `[1, 2, 3]`) → 무엇이 들었든 **최소한 `Number`** 임이 보장됨 → `doubleValue()`로 꺼내 더함 → `6.0`
  * `sum(doubleList)` (값 `[1.5, 2.5]`) → `4.0`
  * `addInteger(numberList)` → 리스트가 `Integer`이거나 그 **부모** 타입을 담는 리스트임이 보장됨 → `Integer` 100을 넣어도 항상 안전함.
  * 반대로 `<? extends Number>` 리스트에는 값을 **넣을 수 없음**. 실제로 `List<Double>`일 수도 있는데 `Integer`를 넣으면 안 되기 때문임. → "꺼낼 땐 extends, 넣을 땐 super"

---

## 7. `Set`

* **개념**: **순서가 없고 중복을 허용하지 않는** 가방. 같은 값을 여러 번 `add()`해도 하나만 남음. 출석부에 같은 이름을 두 번 적지 않는 것과 같음.
* **순회**: 번호(인덱스)가 없어서 `get(i)`를 못 씀 → 향상된 for문이나 `Iterator`로 하나씩 꺼냄.
  * `iterator()`: 하나씩 꺼내주는 **안내원(Iterator)** 을 받음.
  * `hasNext()`: 다음에 꺼낼 게 남았나? → `true`/`false`
  * `next()`: 다음 것을 꺼냄.

```java
Set<String> hset = new HashSet<>();
hset.add("java"); hset.add("html"); hset.add("css"); hset.add("java");
// [css, java, html] → 순서 보장 X, "java" 중복 제거

Iterator<String> iter = hset.iterator();
while (iter.hasNext()) {
    System.out.println(iter.next());
}
```

* **따라가기**
  1. `"java"`, `"html"`, `"css"`를 넣음 → 3개.
  2. `"java"`를 또 넣음 → 이미 있으므로 무시 → 여전히 3개.
  3. 출력 순서는 넣은 순서가 아니라 `HashSet` 내부 규칙(해시값)에 따름 → 예: `[css, java, html]`
  4. `while (iter.hasNext())` → 남았으면 `next()`로 꺼내 출력 → 3번 돌고 남은 게 없으면 끝.

| 구현 클래스 | 중복 | 순서 |
| :--- | :--- | :--- |
| `HashSet` | 불가 | 보장 X |
| `LinkedHashSet` | 불가 | 입력된 순서 유지 |
| `TreeSet` | 불가 | 저장할 때마다 자동 오름차순 정렬 |

---

## 느낀 점

* 같은 "여러 데이터 저장"이라도 순서·중복 허용 여부에 따라 `List`와 `Set`을 골라 써야 한다는 기준이 생김
* `ArrayList`와 `LinkedList`의 내부 구조 차이를 이해하니, "추가/삭제가 많으면 무조건 `LinkedList`"가 아니라 조회 비용까지 함께 따져야 한다는 점을 알게 됨
* `Comparable`(기본 기준)과 `Comparator`(별도 기준)를 나눠 구현하면서, 정렬 기준을 클래스 밖으로 분리하면 대상 클래스를 수정하지 않고도 여러 기준으로 정렬할 수 있다는 장점을 체감함
* 어제 배운 인터페이스 다형성이 `List<String> list = new ArrayList<>()`, `Queue<String> q = new ArrayDeque<>()`처럼 컬렉션 선언 전반에 그대로 쓰이고 있음을 확인함
