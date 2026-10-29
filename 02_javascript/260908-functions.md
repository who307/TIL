# JavaScript 함수 · 호이스팅 · 콜백 정리 (TIL)

## TIL(Today I Learned)

* 함수 선언문과 함수 표현식의 구조 및 호이스팅(Hoisting) 동작 방식 차이를 학습
* 매개변수(parameter)와 인수(argument)의 개념 및 매개변수 기본값 설정 방식을 학습
* return 키워드의 역할과 조건문과 함께 사용하는 얼리 리턴(Early Return) 패턴을 학습
* 자바스크립트 함수가 일급 객체로서 갖는 특징(변수 할당, 객체 프로퍼티 저장, 인자 전달 및 반환)을 학습
* 고차 함수와 콜백 함수(Callback Function)의 개념 및 활용 방식을 학습
* 스코프(Scope)의 개념과 함수 정의 위치에 따라 상위 스코프가 결정되는 렉시컬 스코프(Lexical Scope)를 학습
* var, let, const 키워드의 스코프 차이(함수 vs 블록), 재선언·재할당, 호이스팅 특징을 학습

---

## 1. 함수, 매개변수 기본값, 얼리 리턴

* **한 줄 비유**: 함수는 **자판기**임. 돈과 버튼(입력)을 넣으면 정해진 작업을 하고 음료(결과)를 내놓음.
* **용어**
  * **매개변수(parameter)**: 함수를 만들 때 정해두는 **입력 칸 이름** (`name`).
  * **인수(argument)**: 함수를 부를 때 실제로 **넣는 값** (`"유관순"`).
  * **`return`**: 결과를 함수 밖으로 돌려주고, 그 즉시 함수를 **끝냄**.
* **매개변수 기본값**: 인수를 안 넣었을 때 대신 쓸 값을 `name = "아무개"`처럼 미리 정해둠.
* **얼리 리턴(Early Return)**: 조건이 안 맞으면 **맨 앞에서 바로 `return`** 해서 끝내버리는 방식. "키 120cm 미만은 탑승 불가"를 입구에서 먼저 걸러내는 놀이기구 줄과 같음. `if-else`로 깊게 감싸지 않아도 돼서 읽기 쉬움.

```javascript
// 매개변수 기본값 설정
function hi(name = "아무개") {
    return `${name} 안녕?`;
}

console.log(hi());        // "아무개 안녕?"
console.log(hi("유관순")); // "유관순 안녕?"

// 얼리 리턴 (Early Return)
function registerUser(nickname) {
    if (nickname.length < 2) {
        console.log("닉네임이 너무 짧습니다");
        return; // 조건 미달 시 즉시 종료
    }
    console.log(`${nickname}님 환영합니다~`);
}

registerUser("홍");
registerUser("홍길동");

```

* **한 줄씩 따라가기**
  * `hi()` → 인수가 없음 → `name`에 기본값 `"아무개"`가 들어감 → `아무개 안녕?`
  * `hi("유관순")` → `name`에 `"유관순"`이 들어감 → `유관순 안녕?`
  * `registerUser("홍")` → `"홍".length`는 1 → `1 < 2` 참 → `닉네임이 너무 짧습니다` 출력 → `return`으로 **즉시 종료**. 아래 환영 문구는 실행되지 않음.
  * `registerUser("홍길동")` → 길이 3 → 조건 거짓 → `if`를 건너뜀 → `홍길동님 환영합니다~` 출력.

---

## 2. 함수 선언문 vs 함수 표현식, 호이스팅

* **함수 선언문**: `function hello() {}`처럼 `function` 키워드로 시작하는 방식.
* **함수 표현식**: `const hi = function () {}`처럼 함수를 **변수에 담는** 방식.
* **호이스팅(Hoisting)**: 자바스크립트가 코드를 실행하기 **전에** 선언들을 먼저 훑어서 준비해 두는 동작. 마치 선언이 맨 위로 "끌어올려진(hoist)" 것처럼 보여서 붙은 이름임.
  * **함수 선언문** → 함수 전체가 미리 준비됨 → **선언보다 위에서 호출해도 동작**함.
  * **함수 표현식(`const`)** → 변수 이름만 준비되고 값은 그 줄에 도착해야 들어감 → **선언보다 위에서 호출하면 `ReferenceError`**.

```javascript
// 함수 선언문: 호이스팅으로 인해 선언부 이전 호출 가능
console.log(hello("홍길동"));

function hello(name) {
    return `${name} 안녕?`;
}

// 함수 표현식: 선언 전 호출 불가 (ReferenceError)
const hi = function (name) {
    return `${name}님 안녕하세요!`;
};

console.log(hi("판다"));

```

* **한 줄씩 보기**
  * `console.log(hello("홍길동"))` → `hello`는 아래에 있지만 선언문이라 미리 준비돼 있음 → `홍길동 안녕?`
  * `const hi = function ...` → 이 줄이 실행되는 순간 `hi`에 함수가 담김.
  * `console.log(hi("판다"))` → 담긴 뒤에 호출했으므로 정상 → `판다님 안녕하세요!`
  * 만약 `hi("판다")`를 `const hi` 줄보다 위로 옮기면 → 아직 값이 없는 상태라 `ReferenceError`가 남.

---

## 3. 일급 객체, 고차 함수, 콜백 함수

* **일급 객체**: 함수를 숫자나 문자열 같은 **보통 값처럼** 다룰 수 있다는 뜻. 변수에 담기, 객체에 넣기, 다른 함수에 **넘기기**, 함수에서 **돌려주기**가 모두 가능함.
* **콜백 함수(Callback)**: 다른 함수에 **인수로 넘겨지는 함수**. "다 되면 이걸 불러줘"라고 맡겨두는 함수임. 식당에서 진동벨을 맡기는 것과 같음.
* **고차 함수(Higher-order Function)**: 콜백 함수를 **받거나** 함수를 **돌려주는** 함수.

```javascript
// 고차 함수: 콜백 함수를 전달받아 실행 시점과 방식을 제어
function calculator(calculateCallback, a, b) {
    console.log("계산을 시작합니다.");
    return calculateCallback(a, b);
}

function add(a, b) {
    return a + b;
}

function multiply(a, b) {
    return a * b;
}

console.log(calculator(add, 9, 8));      // 17
console.log(calculator(multiply, 9, 8)); // 72

// 실용 예제: 배열 정렬 콜백
const numbers = [3, 10, 1, 6, 9];
numbers.sort((a, b) => a - b);
console.log(numbers); // [1, 3, 6, 9, 10]

```

* **따라가기: `calculator(add, 9, 8)`**
  1. `calculateCallback`에 **함수 `add` 자체**가 들어감 (괄호 없이 넘겼으므로 실행된 게 아니라 함수가 통째로 전달됨).
  2. `계산을 시작합니다.` 출력.
  3. `calculateCallback(9, 8)` → 사실상 `add(9, 8)` → `17`을 돌려받아 그대로 `return` → `17` 출력.
  * `calculator(multiply, 9, 8)`도 같은 흐름 → `multiply(9, 8)` → `72`. **계산기 함수는 그대로인데 넘기는 콜백만 바꿔서** 다른 계산을 시킨 것임.
* **`numbers.sort((a, b) => a - b)`**
  * `sort`는 두 값을 골라 콜백에 넘기고, 돌아온 값으로 순서를 정함.
  * `a - b`가 **음수**면 a를 앞에, **양수**면 b를 앞에 둠 → 작은 수가 앞으로 → **오름차순** → `[1, 3, 6, 9, 10]`
  * 콜백 없이 `sort()`만 쓰면 숫자를 **글자로 바꿔서** 비교하므로 `10`이 `3`보다 앞에 오는 문제가 생김.

---

## 4. 렉시컬 스코프 (Lexical Scope)

* **스코프(Scope)**: 변수를 **찾을 수 있는 범위**. 방 안에서 만든 변수는 방 밖에서 볼 수 없음.
* **렉시컬 스코프**: 함수가 변수를 찾을 때, 함수가 **호출된 곳**이 아니라 **코드에 적힌(정의된) 곳**을 기준으로 바깥을 찾아 나간다는 규칙. 내 고향은 지금 여행 중인 도시가 아니라 **태어난 곳**인 것과 같음.

```javascript
const label = "바깥";

function printLabel() {
    // 함수가 호출된 위치가 아니라 정의된 위치의 상위 스코프 참조
    console.log(label);
}

function run() {
    const label = "run 안쪽";
    printLabel();
}

run(); // "바깥"

```

* **따라가기**
  1. `run()` 실행 → `run` 안에 `label = "run 안쪽"`이 생김.
  2. `run` 안에서 `printLabel()` 호출.
  3. `printLabel` 안에는 `label`이 없음 → 바깥으로 찾으러 감.
  4. 이때 "바깥"은 `run`이 아니라 `printLabel`이 **적혀 있는 곳**(맨 바깥)임 → `label = "바깥"`을 찾음.
  5. 출력 → `바깥`

---

## 5. var vs let/const 스코프 비교

* **함수 스코프 (`var`)**: `{ }` 블록을 무시하고 **함수 전체**가 하나의 방임.
* **블록 스코프 (`let`, `const`)**: `if`, `for`의 `{ }` 하나하나가 **각각의 방**임.

```javascript
// var: 함수 스코프, 재선언 허용, 호이스팅 시 undefined 초기화
function compareVar() {
    var i = 100;
    for (var i = 0; i < 3; i++) {}
    console.log("var 밖", i); // 3
}

// let / const: 블록 스코프, 재선언 불가
function compareLet() {
    let i = 100;
    for (let i = 0; i < 3; i++) {}
    console.log("let 밖", i); // 100
}

// const 객체: 재할당은 불가능하지만 내부 프로퍼티 수정은 가능
const student = { name: "판다", age: 5 };
student.name = "코알라";
console.log(student.name); // "코알라"

```

* **따라가기: `compareVar`**
  1. `var i = 100` → 함수 방에 `i` 하나 생김(100).
  2. `for (var i = 0; ...)` → `var`는 블록을 무시하므로 **같은 `i`를 다시 씀** → 0으로 덮어씀.
  3. 반복하며 0 → 1 → 2 → 3. 3에서 조건 `3 < 3`이 거짓이라 멈춤.
  4. 출력 → `var 밖 3`. 원래 100은 사라짐(변수 오염).
* **따라가기: `compareLet`**
  1. `let i = 100` → 함수 방의 `i`(100).
  2. `for (let i = 0; ...)` → `for` 블록 안에 **새로운 `i`** 가 따로 생김.
  3. 블록 안의 `i`가 3까지 바뀌어도 바깥 `i`와는 다른 변수임.
  4. 출력 → `let 밖 100`
* `student.name = "코알라"` → `const`는 재할당만 막고 내부 수정은 허용 → `코알라`

---

## 느낀 점

* 함수 선언문과 표현식의 호이스팅 차이를 다루며 예측 가능하고 안전한 코드를 위해 함수 표현식을 활용하는 가치를 이해함
* 얼리 리턴(Early Return)을 활용하여 불필요한 중첩 들여쓰기를 줄이고 가독성 높은 로직을 작성하는 방식을 익힘
* 자바스크립트 함수가 일급 객체로 취급된다는 점과 콜백 함수를 고차 함수에 주입하여 유연한 로직을 구현하는 패턴을 익힘
* 렉시컬 스코프 원리를 통해 변수 참조가 '호출 위치'가 아닌 '정의 위치' 기준임을 명확히 파악함
* var의 함수 스코프 남용 시 발생하는 변수 오염 문제와 let/const의 블록 스코프 보장 장점을 확인함
