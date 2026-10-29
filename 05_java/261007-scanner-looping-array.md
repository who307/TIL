# Java Scanner · 제어문 · 배열 정리 (TIL)

## 1. 콘솔 입력 처리 (Scanner)

* **한 줄 비유**: `Scanner`는 키보드로 친 글자를 받아 적는 **받아쓰기 담당**임. 지금까지는 값이 코드에 고정돼 있었지만, Scanner를 쓰면 실행할 때마다 사용자가 값을 정할 수 있음.
* **쓰는 순서**
  1. `import java.util.Scanner;` → Scanner가 들어 있는 패키지를 불러옴.
  2. `Scanner sc = new Scanner(System.in);` → `System.in`(키보드)에서 읽는 Scanner 객체를 만듦.
  3. 원하는 자료형에 맞는 메소드로 읽음.

| 메소드 | 읽는 것 |
| :--- | :--- |
| `nextInt()` | 정수 |
| `nextDouble()` | 실수 |
| `next()` | 띄어쓰기 전까지의 단어 하나 |
| `nextLine()` | 엔터 전까지의 **한 줄 전체** |

* **`nextLine()` 건너뛰기 문제**
  * 키보드로 `20`을 치고 엔터를 누르면 실제로 들어가는 건 `20\n`(`\n`은 엔터)임.
  * `nextInt()`는 숫자 `20`만 가져가고 `\n`은 **입력 대기실(버퍼)에 남겨둠**.
  * 다음 `nextLine()`은 "엔터 전까지"를 읽는데, 남아 있던 `\n`을 보자마자 **빈 줄을 읽고 끝나버림** → 이름을 입력할 기회가 사라짐.
  * **해결**: 중간에 `sc.nextLine();`을 한 번 더 불러서 남은 `\n`을 **버림**.

---

## 2. 제어문 (조건문, 반복문, 분기문)

### 조건문

* **`if / else if / else`**: 위에서부터 조건(`boolean`)을 검사해 처음으로 참인 블록 하나만 실행함. 범위 조건에 좋음(점수 90 이상, 80 이상…).
* **`switch-case`**: 값 하나가 어느 `case`와 **같은지** 보고 그 자리로 점프함.
  * **Fall-through**: `case` 끝에 `break`가 없으면 아래 `case`들까지 **줄줄이 이어서 실행**됨. 미끄럼틀에서 멈추지 않고 끝까지 내려가는 것과 같음. 보통은 `break`를 꼭 쓰고, 여러 `case`가 같은 일을 할 때만 일부러 생략함.

### 반복문

| 반복문 | 언제 쓰나 | 특징 |
| :--- | :--- | :--- |
| `for (시작; 조건; 증감)` | 몇 번 돌지 **정해져 있을 때** | `for (int i = 0; i < 5; i++)` → 5번 |
| `while (조건)` | 몇 번일지 모르고 **조건이 참인 동안** | 조건이 처음부터 거짓이면 0번 |
| `do { } while (조건);` | **최소 1번은** 꼭 실행해야 할 때 | 먼저 실행하고 나중에 검사 |

### 분기문

* **`break`**: 반복문을 **완전히 빠져나감**. "그만!"
* **`continue`**: 이번 바퀴의 남은 부분을 건너뛰고 **다음 바퀴로**. "이번 건 패스!"

---

## 3. 배열 (Array)

### 1차원 배열

* **한 줄 비유**: 배열은 **같은 크기의 칸이 일렬로 붙은 사물함**임. 같은 자료형만 넣을 수 있고, 칸 번호(인덱스)는 **0부터** 시작함.
* **만들기**: `int[] arr = new int[5];` → 정수 5칸짜리 사물함. 처음엔 모든 칸이 0. 또는 `int[] arr = {1, 2, 3};`처럼 값을 바로 넣음.
* **꺼내기**: `arr[0]` → 첫 칸, `arr[arr.length - 1]` → 마지막 칸. 없는 번호(`arr[5]`)를 쓰면 `ArrayIndexOutOfBoundsException` 에러.
* **`length`**: 칸의 개수. 반복문에서 `i < arr.length`로 끝을 정함.
* **향상된 for문**: `for (int n : arr)` → 번호 없이 칸의 값을 하나씩 꺼냄.

### 다차원 배열

* **2차원 배열**: 배열 안에 배열이 들어 있는 것. **행(가로줄)과 열(세로줄)이 있는 표**나 **아파트(층, 호수)** 와 같음.
  * `int[][] apt = new int[3][4];` → 3층, 층마다 4호.
  * `apt[1][2]` → 2층(인덱스 1)의 3호(인덱스 2).
  * 중첩 `for`문으로 순회: 바깥 for가 층, 안쪽 for가 호수를 돎.
* **가변 배열**: 층마다 호수 개수가 다른 배열. `int[][] arr = new int[3][];`로 층만 정하고, `arr[0] = new int[2]; arr[1] = new int[5];`처럼 따로 만듦.

### 배열 복사

* **얕은 복사(Shallow Copy)**: `int[] copy = origin;` → 사물함을 새로 만든 게 아니라, **같은 사물함의 열쇠를 하나 더 복사한 것**. 한쪽에서 칸을 바꾸면 다른 쪽에서도 바뀐 게 보임.
* **깊은 복사(Deep Copy)**: **새 사물함을 따로 만들고 내용물을 옮겨 담음**. 한쪽을 바꿔도 다른 쪽은 그대로임.
  * 방법: `clone()`, `System.arraycopy()`, `Arrays.copyOf()`, 또는 for문으로 한 칸씩 복사.

---

## 핵심 코드 예시

```java
// 1. Scanner 콘솔 입력 예시 (section03/scanner)
import java.util.Scanner;

public class Application1 {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        System.out.print("나이를 입력하세요: ");
        int age = sc.nextInt(); // 정수 입력

        sc.nextLine(); // 버퍼의 개행 문자 처리

        System.out.print("이름을 입력하세요: ");
        String name = sc.nextLine(); // 문자열 입력
    }
}

```

* **따라가기** (사용자가 `20` 엔터, `홍길동` 엔터를 입력)
  1. `나이를 입력하세요: ` 출력 (`print`라 줄바꿈 없이 커서가 옆에 있음).
  2. `sc.nextInt()` → 버퍼의 `20\n`에서 `20`만 가져감 → `age = 20`. 버퍼에는 `\n`이 남음.
  3. `sc.nextLine()` → 남은 `\n`을 읽어서 **버림**. 이제 버퍼가 깨끗함.
  4. `이름을 입력하세요: ` 출력.
  5. `sc.nextLine()` → `홍길동\n`에서 엔터 전까지 → `name = "홍길동"`.
  * 3번 줄이 없었다면 → 5번의 `nextLine()`이 남은 `\n`을 읽어 `name`이 빈 문자열 `""`이 되고, 사용자는 이름을 칠 기회도 없이 프로그램이 넘어감.

```java
// 2. 깊은 복사(Deep Copy) 예시 (section03/copy)
public class Application2 {
    public static void main(String[] args) {
        int[] origin = {1, 2, 3, 4, 5};

        // clone() 이용 깊은 복사
        int[] copy1 = origin.clone();

        // System.arraycopy() 이용 깊은 복사
        int[] copy2 = new int[origin.length];
        System.arraycopy(origin, 0, copy2, 0, origin.length);

        copy1[0] = 99; // origin 데이터에 영향 없음
    }
}

```

* **한 줄씩 보기**
  * `int[] origin = {1, 2, 3, 4, 5};` → 5칸 사물함.
  * `origin.clone()` → 똑같은 내용의 **새 사물함**을 만들어 돌려줌 → `copy1 = {1, 2, 3, 4, 5}`
  * `new int[origin.length]` → 빈 5칸 사물함(전부 0)을 먼저 만듦.
  * `System.arraycopy(origin, 0, copy2, 0, origin.length)` → 순서대로 "`origin`의 **0번 칸부터**, `copy2`의 **0번 칸부터**, **5개**를 복사해" → `copy2 = {1, 2, 3, 4, 5}`
  * `copy1[0] = 99;` → `copy1 = {99, 2, 3, 4, 5}`. 다른 사물함이므로 `origin`은 여전히 `{1, 2, 3, 4, 5}`.
  * 만약 `int[] copy3 = origin;`(얕은 복사) 후 `copy3[0] = 99`였다면 → `origin[0]`도 99가 됨.

---

## 느낀 점

* `Scanner` 클래스를 통해 하드코딩된 리터럴이 아닌 사용자의 실제 키보드 입력을 받아 동적으로 작동하는 다채로운 프로그램을 작성하는 방식을 습득함
* 조건문과 반복문, 분기문(`break`, `continue`)을 조합하여 프로그램의 제어 흐름(Control Flow)을 효율적으로 구성하는 논리력을 강화함
* 배열의 메모리 참조 개념을 정리하면서, 얕은 복사 사용 시 발생할 수 있는 데이터 오염(Side Effect) 위험성을 인지함
* 데이터의 독립성을 안전하게 유지하기 위한 깊은 복사(`System.arraycopy`, `clone()`) 적용의 필요성과 사용법을 명확히 체득함
