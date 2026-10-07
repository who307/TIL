# Java 핵심 기술 정리 (TIL)

## 1. 콘솔 입력 처리 (Scanner)

* `java.util.Scanner` 클래스를 `import`하여 콘솔 키보드 입력을 동적으로 수신하는 기법을 습득함.


* `nextInt()`, `nextDouble()`, `next()`, `nextLine()` 등 입력받고자 하는 데이터 타입에 적합한 메소드를 호출하여 변수에 저장하는 흐름을 파악함.


* `nextLine()` 호출 시 기존 입력 버퍼에 남아있는 개행 문자(`\n`)로 인해 입력을 건너뛰는 현상을 이해하고, 개행 처리 버퍼 비우기 패턴을 체득함.



---

## 2. 제어문 (Conditional, Looping, Branching)

### 조건문 (Conditional)

* `if`, `else if`, `else` 구문과 `switch-case` 구문을 활용해 조건식의 평가 결과(`boolean`)에 따라 실행 흐름을 제어하는 구조를 체득함.


* `switch` 구문에서 조건에 일치하는 `case` 실행 후 `break`문이 없을 경우 다음 `case`로 연속 진행되는 Fall-through 특성을 다루는 기법을 습득함.



### 반복문 (Looping)

* 반복 횟수가 명확할 때 사용하는 `for`문과 조건식 평가에 따라 반복을 수행하는 `while`문의 구조적 차이를 이해함.


* 조건식 평가 전 최소 1회의 블록 실행을 보장하는 `do-while`문의 구동 메커니즘을 파악함.



### 분기문 (Branching)

* 반복문 전체를 즉시 탈출하는 `break`문과, 현재 진행 중인 회차를 중단하고 다음 반복 회차로 넘어가는 `continue`문의 차이점을 구별함.



---

## 3. 배열 (Array & Dimensional Array & Array Copy)

### 1차원 배열 (Array)

* 동일한 자료형의 여러 데이터를 연속된 메모리 공간에 할당하고, 인덱스(0부터 시작)를 통해 요소에 직접 접근하는 구조를 체득함.


* `length` 필드를 활용해 배열의 크기를 파악하고 `for`문 또는 향상된 `for`문(Enhanced for)으로 배열 전체 요소를 순회하는 패턴을 습득함.



### 다차원 배열 (Dimensional Array)

* 행과 열의 2차원 구조를 갖는 배열을 선언·할당하고, 중첩 `for`문을 활용해 다차원 데이터 집합을 제어하는 방식을 파악함.


* 가변 배열 구조와 같이 행마다 열의 길이가 다르게 구성되는 메모리 참조 방식을 이해함.



### 배열 복사 (Array Copy)

* **얕은 복사 (Shallow Copy)**: 배열 변수가 가지고 있는 메모리 주소(Reference) 값만 복사하여, 두 변수가 동일한 배열 객체를 공유하도록 만드는 특성을 확인함.


* **깊은 복사 (Deep Copy)**: `System.arraycopy()`, `Arrays.copyOf()`, `clone()` 메소드 또는 직접 순회를 이용하여 새로운 메모리 공간에 원본 요소를 복사함으로써 객체의 독립성을 보장하는 기법을 체득함.



---

## 핵심 코드 예시

```java
// 1. Scanner 콘솔 입력 예시 (section03/scanner)
import java.util.Scanner;[cite: 29]

public class Application1 {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);[cite: 29]
        
        System.out.print("나이를 입력하세요: ");[cite: 29]
        int age = sc.nextInt(); // 정수 입력[cite: 29]
        
        sc.nextLine(); // 버퍼의 개행 문자 처리[cite: 29]
        
        System.out.print("이름을 입력하세요: ");[cite: 29]
        String name = sc.nextLine(); // 문자열 입력[cite: 29]
    }
}

```

```java
// 2. 깊은 복사(Deep Copy) 예시 (section03/copy)
public class Application2 {
    public static void main(String[] args) {
        int[] origin = {1, 2, 3, 4, 5};[cite: 30]
        
        // clone() 이용 깊은 복사[cite: 30]
        int[] copy1 = origin.clone();[cite: 30]
        
        // System.arraycopy() 이용 깊은 복사[cite: 30]
        int[] copy2 = new int[origin.length];[cite: 30]
        System.arraycopy(origin, 0, copy2, 0, origin.length);[cite: 30]
        
        copy1[0] = 99; // origin 데이터에 영향 없음[cite: 30]
    }
}

```

---

## 느낀 점

* `Scanner` 클래스를 통해 하드코딩된 리터럴이 아닌 사용자의 실제 키보드 입력을 받아 동적으로 작동하는 다채로운 프로그램을 작성하는 방식을 습득함.


* 조건문과 반복문, 분기문(`break`, `continue`)을 조합하여 프로그램의 제어 흐름(Control Flow)을 효율적으로 구성하는 논리력을 강화함.


* 배열의 메모리 참조 개념을 정리하면서, 얕은 복사 사용 시 발생할 수 있는 데이터 오염(Side Effect) 위험성을 인지함.


* 데이터의 독립성을 안전하게 유지하기 위한 깊은 복사(`System.arraycopy`, `clone()`) 적용의 필요성과 사용법을 명확히 체득함.