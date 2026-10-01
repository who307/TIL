# JavaScript

## TIL(Today I Learned)
* 함수 선언문과 함수 표현식의 구조 및 호이스팅(Hoisting) 동작 방식 차이를 학습
* 매개변수(parameter)와 인수(argument)의 개념 및 매개변수 기본값 설정 방식을 학습
* return 키워드의 역할과 조건문과 함께 사용하는 얼리 리턴(Early Return) 패턴을 학습
* 자바스크립트 함수가 일급 객체로서 갖는 특징(변수 할당, 객체 프로퍼티 저장, 인자 전달 및 반환)을 학습
* 고차 함수와 콜백 함수(Callback Function)의 개념 및 활용 방식을 학습
* 스코프(Scope)의 개념과 함수 정의 위치에 따라 상위 스코프가 결정되는 렉시컬 스코프(Lexical Scope)를 학습
* var, let, const 키워드의 스코프 차이(함수 vs 블록), 재선언·재할당, 호이스팅 특징을 학습

## 핵심 내용(코드)

### 함수 선언과 매개변수 기본값 / 얼리 리턴

```javascript
// 매개변수 기본값 설정[cite: 9]
function hi(name = "아무개") {
    return `${name} 안녕?`;
}

console.log(hi());        // "아무개 안녕?"[cite: 9]
console.log(hi("유관순")); // "유관순 안녕?"[cite: 9]

// 얼리 리턴 (Early Return)[cite: 10]
function registerUser(nickname) {
    if (nickname.length < 2) {
        console.log("닉네임이 너무 짧습니다");
        return; // 조건 미달 시 즉시 종료[cite: 10]
    }
    console.log(`${nickname}님 환영합니다~`);
}

registerUser("홍");
registerUser("홍길동");

```

### 함수 표현식과 호이스팅

```javascript
// 함수 선언문: 호이스팅으로 인해 선언부 이전 호출 가능[cite: 11]
console.log(hello("홍길동"));

function hello(name) {
    return `${name} 안녕?`;
}

// 함수 표현식: 선언 전 호출 불가 (ReferenceError)[cite: 11]
const hi = function (name) {
    return `${name}님 안녕하세요!`;
};

console.log(hi("판다"));

```

### 일급 객체와 콜백 함수

```javascript
// 고차 함수: 콜백 함수를 전달받아 실행 시점과 방식을 제어[cite: 13]
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

console.log(calculator(add, 9, 8));      // 17[cite: 13]
console.log(calculator(multiply, 9, 8)); // 72[cite: 13]

// 실용 예제: 배열 정렬 콜백[cite: 13]
const numbers = [3, 10, 1, 6, 9];
numbers.sort((a, b) => a - b);
console.log(numbers); // [1, 3, 6, 9, 10][cite: 13]

```

### 렉시컬 스코프 (Lexical Scope)

```javascript
const label = "바깥";

function printLabel() {
    // 함수가 호출된 위치가 아니라 정의된 위치의 상위 스코프 참조[cite: 16]
    console.log(label);
}

function run() {
    const label = "run 안쪽";
    printLabel();
}

run(); // "바깥"[cite: 16]

```

### 변수 선언 키워드 비교 (var, let, const)

```javascript
// var: 함수 스코프, 재선언 허용, 호이스팅 시 undefined 초기화[cite: 17]
function compareVar() {
    var i = 100;
    for (var i = 0; i < 3; i++) {}
    console.log("var 밖", i); // 3[cite: 17]
}

// let / const: 블록 스코프, 재선언 불가[cite: 17]
function compareLet() {
    let i = 100;
    for (let i = 0; i < 3; i++) {}
    console.log("let 밖", i); // 100[cite: 17]
}

// const 객체: 재할당은 불가능하지만 내부 프로퍼티 수정은 가능[cite: 17]
const student = { name: "판다", age: 5 };
student.name = "코알라";
console.log(student.name); // "코알라"[cite: 17]

```

## 느낀 점
* 함수 선언문과 표현식의 호이스팅 차이를 다루며 예측 가능하고 안전한 코드를 위해 함수 표현식을 활용하는 가치를 이해함
* 얼리 리턴(Early Return)을 활용하여 불필요한 중첩 들여쓰기를 줄이고 가독성 높은 로직을 작성하는 방식을 익힘
* 자바스크립트 함수가 일급 객체로 취급된다는 점과 콜백 함수를 고차 함수에 주입하여 유연한 로직을 구현하는 패턴을 익힘
* 렉시컬 스코프 원리를 통해 변수 참조가 '호출 위치'가 아닌 '정의 위치' 기준임을 명확히 파악함
* var의 함수 스코프 남용 시 발생하는 변수 오염 문제와 let/const의 블록 스코프 보장 장점을 확인함