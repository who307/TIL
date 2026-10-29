# Java 변수 종류 · 오버로딩 · static/final · 싱글톤 정리 (TIL)

## 1. 변수의 종류 (Kinds of Variables)

* 변수는 **어디에 선언했는지**에 따라 이름과 수명(언제 생기고 언제 사라지는지)이 다름.

| 종류 | 선언 위치 | 생기는 때 → 사라지는 때 | 비유 |
| :--- | :--- | :--- | :--- |
| **인스턴스 변수** | 클래스 안, 메소드 밖 (`static` 없음) | 객체를 `new`로 만들 때 → 객체가 버려질 때 | 학생마다 가진 **개인 사물함** |
| **클래스(static) 변수** | 클래스 안, 메소드 밖 (`static` 있음) | 프로그램이 클래스를 처음 쓸 때 → 프로그램 끝날 때 | 반 전체가 같이 쓰는 **교실 칠판** |
| **지역 변수** | 메소드나 `{ }` 블록 안 | 그 줄이 실행될 때 → 블록이 끝날 때 | 수업 시간에만 쓰는 **연습장** |
| **매개변수** | 메소드 괄호 안 | 메소드가 불릴 때 → 메소드가 끝날 때 | 자판기에 넣은 **동전** |

```java
public class Student {
    static int studentCount = 0;   // 클래스 변수: 모든 학생이 공유
    String name;                   // 인스턴스 변수: 학생마다 따로

    public void study(int hours) { // hours: 매개변수
        int pages = hours * 10;    // pages: 지역 변수
        System.out.println(name + "이(가) " + pages + "쪽 공부함");
    }                              // 여기서 hours, pages는 사라짐
}
```

* **따라가기**: `Student s = new Student(); s.name = "판다"; s.study(2);`
  1. `new Student()` → 인스턴스 변수 `name`이 생김.
  2. `s.study(2)` → 매개변수 `hours = 2`가 생김 → 지역 변수 `pages = 20`이 생김.
  3. `판다이(가) 20쪽 공부함` 출력.
  4. 메소드가 끝나면 `hours`, `pages`는 사라짐. `name`은 객체가 있는 한 남아 있음.

---

## 2. 메소드 오버로딩 (Method Overloading)과 매개변수

### 오버로딩 규칙

* **오버로딩**: **같은 이름**의 메소드를 매개변수만 다르게 해서 **여러 개** 만드는 것. "더하기"라는 이름 하나로 정수 더하기, 실수 더하기, 세 수 더하기를 모두 처리함.
* **구분 기준**: 매개변수의 **개수**, **타입**, **순서** 중 하나라도 달라야 함.
* **구분 기준이 아닌 것**: 반환형만 다른 것, 매개변수 **이름**만 다른 것 → 자바가 어느 걸 부를지 알 수 없어서 에러.

```java
public int add(int a, int b) { return a + b; }            // ① 정수 2개
public int add(int a, int b, int c) { return a + b + c; } // ② 개수가 다름
public double add(double a, double b) { return a + b; }   // ③ 타입이 다름
// public double add(int a, int b) { ... }  // ①과 매개변수가 같아서 에러
```

* **따라가기**: 자바는 넘긴 값을 보고 맞는 메소드를 고름.
  * `add(1, 2)` → 정수 2개 → ① → `3`
  * `add(1, 2, 3)` → 정수 3개 → ② → `6`
  * `add(1.5, 2.5)` → 실수 2개 → ③ → `4.0`

### 매개변수로 넘길 수 있는 것

* **기본 자료형** (`int` 등): 값이 **복사**되어 넘어감 → 메소드 안에서 바꿔도 원래 변수는 그대로임. 숙제를 복사해서 친구에게 주면, 친구가 낙서해도 내 숙제는 멀쩡한 것과 같음.
* **배열·객체** (참조 자료형): 값이 있는 **위치(주소)** 가 복사되어 넘어감 → 메소드 안에서 내용을 바꾸면 원래 것도 바뀜. 사물함 **열쇠**를 복사해서 준 것과 같음.
* **가변 인자 (`int... nums`)**: 개수가 정해지지 않은 값을 받음. 메소드 안에서는 **배열**처럼 씀. `sum(1, 2)`, `sum(1, 2, 3, 4)` 모두 가능.

---

## 3. `static` 키워드와 `final` 키워드

### `static`: 모두가 함께 쓰는 것

* `static`이 붙은 변수·메소드는 **객체가 아니라 클래스에 하나만** 있음. 객체를 만들지 않아도 `클래스명.이름`으로 바로 씀.
* `Student` 예시: 학생 수를 세는 `studentCount`를 `static`으로 두면, 학생을 몇 명 만들든 **칠판 하나**에 숫자가 누적됨.

```java
public class Student {
    static int studentCount = 0;
    String name;

    public Student(String name) {
        this.name = name;
        studentCount++;   // 학생이 만들어질 때마다 공용 칠판 숫자 +1
    }
}
```

* **따라가기**: `new Student("판다"); new Student("코알라");` → 각 학생은 `name`을 따로 가지지만, `Student.studentCount`는 하나뿐이라 `2`가 됨.

### `final`: 바꿀 수 없게 고정

| 붙는 곳 | 의미 |
| :--- | :--- |
| 변수 | 값을 한 번만 넣을 수 있음 (상수) |
| 메소드 | 자식 클래스가 **재정의(오버라이딩) 못 함** |
| 클래스 | 다른 클래스가 **상속 못 함** (예: `String`) |

* `MathConstant` 예시: 원주율처럼 **모두가 쓰는데 절대 바뀌면 안 되는 값**은 `public static final double PI = 3.14159;`처럼 `static`(하나만) + `final`(고정)을 함께 씀. 쓸 때는 `MathConstant.PI`.

---

## 4. 싱글톤 패턴 (Singleton Pattern)

* **싱글톤**: 어떤 클래스의 객체를 **프로그램 전체에서 딱 하나만** 만들고, 모두가 그 하나를 같이 쓰게 하는 설계 방법. 학교에 **교장 선생님은 한 명**뿐인 것과 같음.
* **왜 쓰나**: 설정 정보나 프린터 관리자처럼 여러 개 있으면 오히려 헷갈리거나 메모리를 낭비하는 객체가 있음.
* **만드는 요령 3단계**
  1. 생성자를 `private`으로 막음 → 밖에서 `new`를 못 함.
  2. 클래스 안에 자기 자신을 담는 `private static` 변수를 둠.
  3. 그 하나를 꺼내주는 `public static getInstance()` 메소드를 만듦.

### Eager vs Lazy 초기화

| 방식 | 객체를 만드는 때 | 장점 | 단점 |
| :--- | :--- | :--- | :--- |
| **Eager (이른 초기화)** | 클래스를 처음 쓸 때 **미리** | 코드가 단순함 | 한 번도 안 써도 만들어져 있음 |
| **Lazy (게으른 초기화)** | `getInstance()`를 **처음 부를 때** | 필요할 때만 만듦 | 처음 부를 때 "없으면 만들기" 검사가 필요함 |

```java
public class EagerSingleton {
    private static EagerSingleton eager = new EagerSingleton(); // 미리 만들어 둠

    private EagerSingleton() {}

    public static EagerSingleton getInstance() {
        return eager;
    }
}

public class LazySingleton {
    private static LazySingleton lazy;   // 아직 null

    private LazySingleton() {}

    public static LazySingleton getInstance() {
        if (lazy == null) {               // 처음 부를 때만
            lazy = new LazySingleton();   // 그때 만듦
        }
        return lazy;
    }
}
```

* **따라가기: `LazySingleton`**
  1. `LazySingleton.getInstance()` 첫 호출 → `lazy`가 `null` → 새로 만들어 담고 돌려줌.
  2. 두 번째 호출 → `lazy`가 이미 있음 → `if`를 건너뛰고 **같은 객체**를 돌려줌.
  3. `LazySingleton.getInstance() == LazySingleton.getInstance()` → `true`
  4. `new LazySingleton()`을 밖에서 쓰면 → 생성자가 `private`이라 컴파일 에러.

---

## 느낀 점

* 변수의 선언 위치에 따른 종류와 생명주기를 구분하고, 다형성을 지원하는 메소드 오버로딩의 규칙을 명확히 정리함
* `static`과 `final`을 통해 메모리 공유와 불변성 제어를 구현하는 자바의 핵심 제어자를 다뤄봄
* 싱글톤 패턴의 두 가지 구현 방식(Eager, Lazy)을 통해 객체 생성 제어와 메모리 효율성 최적화 기법을 체득함
