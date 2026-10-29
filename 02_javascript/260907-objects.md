# JavaScript 객체 · 조건문 · 반복문 정리 (TIL)

## TIL(Today I Learned)

* 객체(Object)의 개념과 키-값(key-value) 쌍으로 구성된 프로퍼티 및 메서드 작성법을 학습
* 마침표 표기법과 대괄호 표기법의 차이 및 변수/특수문자 키 접근법을 학습
* 객체 프로퍼티의 값 수정, 추가, delete 연산자를 통한 삭제, in 연산자 및 for...in 순회법을 학습
* if-else 및 switch 문을 활용한 조건 분기 처리 방법을 학습
* for, while, do-while 반복문 및 break, continue 키워드를 통한 제어 흐름을 학습

---

## 1. 객체 선언과 프로퍼티 접근

* **한 줄 비유**: 객체는 **칸마다 이름표가 붙은 서랍장**임. "이름" 칸에는 "뽀삐", "나이" 칸에는 5를 넣어두는 식.
* **용어**
  * **프로퍼티(property)**: 서랍 한 칸. `키: 값` 한 쌍으로 이루어짐 (`name: "판다"`).
  * **키(key)**: 서랍 칸의 이름표 (`name`).
  * **값(value)**: 칸 안에 든 물건 (`"판다"`).
  * **메서드(method)**: 값으로 **함수**가 들어 있는 프로퍼티. 객체가 할 수 있는 "행동"임 (`eat`).
* **`this`**: 메서드 안에서 쓰면 "**지금 이 메서드를 부른 객체 자신**"을 가리킴. `dog.eat()`이면 `this`는 `dog`임.
* **꺼내는 방법 2가지**
  * **마침표 표기법**: `dog.name` → 짧고 읽기 쉬움.
  * **대괄호 표기법**: `dog["name"]` → 키를 **문자열**로 씀. 키가 변수에 들어 있거나, `-` 같은 특수문자·숫자로 된 키일 때는 이 방법만 가능함.

```javascript
// 객체 생성 및 메서드 정의
const student = {
    name: "판다",
    age: 5,
    getInfo: function() {
        return `${this.name}는 ${this.age}세 입니다.`;
    }
};

const dog = {
    name: "뽀삐",
    eat: function (food) {
        console.log(`${this.name}는 ${food}를 맛있게 먹어요.`);
    }
};

// 1. 마침표 표기법
console.log(dog.name);
dog.eat("두부");

// 2. 대괄호 표기법 (프로퍼티 키를 문자열로 지정)
console.log(dog["name"]);
dog["eat"]("고구마");

// 키가 변수에 담겨 있거나 특수문자/숫자인 경우 대괄호 표기법 필수 사용
const prop = "name";
console.log(dog.prop); // undefined ("prop"이라는 프로퍼티 키를 찾으려고 함)
console.log(dog[prop]); // dog["name"] -> "뽀삐"

const obj = {
    "dash-key": "dash-value",
    0: 1
};
console.log(obj["dash-key"]);
console.log(obj[0]);

```

* **한 줄씩 보기**
  * `student`의 `getInfo` → 호출하면 `this.name`(판다), `this.age`(5)를 꺼내 `판다는 5세 입니다.`를 돌려줌.
  * `dog.name` → `뽀삐` 출력.
  * `dog.eat("두부")` → `eat` 함수가 실행되고 `food`에 "두부"가 들어감 → `뽀삐는 두부를 맛있게 먹어요.`
  * `dog["name"]`, `dog["eat"]("고구마")` → 마침표 대신 대괄호로 꺼냈을 뿐 결과는 같음 → `뽀삐`, `뽀삐는 고구마를 맛있게 먹어요.`
  * `const prop = "name";` → 변수 `prop`에 문자열 `"name"`을 넣음.
  * `dog.prop` → 마침표 뒤의 글자는 **그대로 키 이름**으로 읽힘 → `"prop"`이라는 칸을 찾는데 없음 → `undefined`
  * `dog[prop]` → 대괄호 안의 `prop`은 **변수**로 읽힘 → 변수 안의 값 `"name"`으로 바뀜 → `dog["name"]` → `뽀삐`
  * `obj["dash-key"]` → `obj.dash-key`라고 쓰면 `obj.dash 빼기 key`로 해석돼서 안 됨 → 대괄호로만 꺼낼 수 있음 → `dash-value`
  * `obj[0]` → 숫자 키도 대괄호로 꺼냄 → `1`

---

## 2. 프로퍼티 추가·수정·삭제와 순회

* 객체는 만든 뒤에도 칸을 **바꾸고, 새로 만들고, 없앨** 수 있음.
* **`const` 객체인데 왜 바뀌나?**: `const`가 막는 것은 **서랍장 자체를 다른 서랍장으로 바꾸는 것(재할당)** 뿐임. 서랍 안의 물건을 바꾸는 것은 막지 않음.

```javascript
const dog = {
    name: "뽀삐"
};

// 이미 존재하는 프로퍼티 값 갱신
dog.name = "감자";

// 새로운 프로퍼티 동적 추가
dog.age = 5;
console.log(dog); // { name: '감자', age: 5 }

// delete 연산자를 사용한 프로퍼티 삭제
delete dog.age;
console.log(dog); // { name: '감자' }

// const 객체는 내부 값 변경/추가/삭제는 가능하나 객체 자체 재할당은 불가
// dog = { name: "초코" }; // Error

// in 연산자: 객체 내 프로퍼티 존재 여부를 boolean 값으로 반환
console.log("name" in dog);   // true
console.log("weight" in dog); // false

// for...in 문: 객체의 프로퍼티 키를 순회
for (const key in dog) {
    console.log(`키: ${key}`);
    console.log(`값: ${dog[key]}`);
}

```

* **한 줄씩 따라가기** (`dog`의 상태 변화)
  1. 처음 → `{ name: "뽀삐" }`
  2. `dog.name = "감자"` → 이미 있는 칸이라 값만 바뀜 → `{ name: "감자" }`
  3. `dog.age = 5` → 없던 칸이라 **새로 생김** → `{ name: "감자", age: 5 }`
  4. `delete dog.age` → `age` 칸이 통째로 사라짐 → `{ name: "감자" }`
  5. `dog = { name: "초코" }` → 서랍장 자체를 바꾸려는 것이라 `TypeError` → 주석 처리함.
  6. `"name" in dog` → `name` 칸이 있음 → `true` / `"weight" in dog` → 없음 → `false`
  7. `for (const key in dog)` → 칸의 **키를 하나씩** `key`에 넣으며 반복함. 지금은 칸이 `name` 하나라 1번만 돔 → `키: name`, `값: 감자` 출력. 값은 `dog[key]`처럼 **대괄호**로 꺼내야 함 (key가 변수이므로).

---

## 3. 조건문 (if-else, switch)

* **조건문**: "만약 ~라면 이걸 하고, 아니면 저걸 해"를 코드로 쓰는 것. 갈림길에서 표지판을 보고 길을 고르는 것과 같음.
* **`if / else if / else`**: 위에서부터 차례로 조건을 검사하다가 **처음으로 참인 곳 하나만** 실행하고 나머지는 건너뜀. 점수 구간처럼 **범위**를 나눌 때 좋음.
* **`switch`**: 값 하나를 여러 `case`와 **똑같은지(===)** 비교함. 메뉴 번호처럼 **정해진 값 중 하나**를 고를 때 좋음.
  * `break`: 해당 `case`를 실행한 뒤 `switch`를 빠져나감. 빼먹으면 아래 `case`까지 줄줄이 실행됨.
  * `default`: 어느 `case`에도 해당하지 않을 때 실행됨.

```javascript
// if-else if-else 문: 범위 조건 처리
let score = 75;

if (score >= 90) {
    console.log("성적이 매우 우수합니다.");
} else if (score >= 70) {
    console.log("성적이 보통입니다.");
} else if (score >= 50) {
    console.log("성적이 부족합니다.");
} else {
    console.log("공부가 많이 필요합니다.");
}

// switch 문: 특정 값에 따른 분기 처리
const fruit = "바나나";

switch (fruit) {
    case "사과":
        console.log("선택한 과일은 사과이다.");
        break;
    case "바나나":
        console.log("선택한 과일은 바나나이다.");
        break;
    default:
        console.log("알 수 없는 과일입니다.");
}

```

* **따라가 보기 (`score = 75`)**
  1. `75 >= 90` → 거짓 → 건너뜀.
  2. `75 >= 70` → **참** → `성적이 보통입니다.` 출력.
  3. 이미 참인 곳을 찾았으므로 `>= 50`, `else`는 검사하지 않고 끝남.
* **따라가 보기 (`fruit = "바나나"`)**
  1. `"바나나" === "사과"` → 아님.
  2. `"바나나" === "바나나"` → 맞음 → `선택한 과일은 바나나이다.` 출력.
  3. `break` → `switch` 밖으로 나감. `default`는 실행되지 않음.

---

## 4. 반복문과 break, continue

* **반복문**: 같은 일을 여러 번 시키는 문법. "줄넘기 10번 해"를 코드로 쓰는 것.
* **`for (시작; 조건; 증감)`**: 반복 횟수가 정해져 있을 때 씀.
  1. **시작**: `let i = 1` → 처음 한 번만 실행.
  2. **조건**: `i <= 5` → 참이면 `{ }` 안을 실행, 거짓이면 반복 끝.
  3. **증감**: `i++` → 한 바퀴 돌 때마다 i를 1 늘림. 다시 2번으로.
* **`continue`**: 이번 바퀴의 **나머지를 건너뛰고** 다음 바퀴로 감.
* **`break`**: 반복문을 **완전히 끝냄**.
* **`while (조건)`**: 조건이 참인 동안 계속 반복함. 횟수보다 "언제 멈출지"가 중요할 때 씀.
* **`do { } while (조건)`**: 먼저 한 번 실행한 **뒤에** 조건을 검사함 → 조건이 처음부터 거짓이어도 **최소 1번은 실행**됨.

```javascript
// for 문 기본
for (let i = 1; i <= 5; i++) {
    console.log(`반복 횟수 : ${i}`);
}

// continue: 이번 회차를 건너뛰고 다음 반복 진행
for (let i = 1; i <= 10; i++) {
    if (i % 2 === 0) continue; // 홀수만 출력
    console.log(i);
}

// break: 반복문을 완전히 종료
for (let i = 1; i <= 10; i++) {
    if (i % 2 === 0) break; // 짝수를 만나면 루프 종료
    console.log(i);
}

// while 문
let count = 3;
while (count > 0) {
    console.log(count);
    count--;
}

// do-while 문: 조건 검사 전 최소 한 번은 실행
let number = 1;
do {
    console.log(number);
    number++;
} while (number <= 3);

```

* **실행 결과 따라가기**
  * 첫 번째 `for` → i가 1, 2, 3, 4, 5일 때 출력 → `반복 횟수 : 1` ~ `반복 횟수 : 5`. i가 6이 되면 `6 <= 5`가 거짓이라 끝남.
  * `continue` 예제 → `i % 2`는 i를 2로 나눈 나머지. 짝수면 나머지 0 → `continue`로 출력을 건너뜀 → `1 3 5 7 9`만 출력.
  * `break` 예제 → i=1: 홀수라 `1` 출력 → i=2: 짝수라 `break` → 반복문 즉시 종료. 출력은 `1` 하나뿐임.
  * `while` → count=3 출력 후 2로 → 2 출력 후 1로 → 1 출력 후 0으로 → `0 > 0` 거짓이라 끝 → `3 2 1`
  * `do-while` → number=1 출력 후 2로 → `2 <= 3` 참 → 2 출력 후 3으로 → 참 → 3 출력 후 4로 → `4 <= 3` 거짓이라 끝 → `1 2 3`

---

## 느낀 점

* 마침표 표기법과 대괄호 표기법의 쓰임새를 구분하면서, 동적 키 접근이나 특수문자 키 처리 시 대괄호 표기법이 꼭 필요하다는 점을 명확히 이해함
* `const`로 선언된 객체라 할지라도 객체 자체의 재할당만 금지될 뿐, 내부 프로퍼티의 값 변경·추가·삭제는 자유롭게 가능하다는 특징을 배움
* 반복문 내에서 `continue`와 `break`를 적절히 제어하여 불필요한 연산을 줄이고 코드 흐름을 효과적으로 관리하는 방법을 익힘
* 조건문과 반복문을 다루면서 로직 조건식 설정에 따라 코드 동작이 어떻게 달라지는지 흐름을 파악하는 기본기를 쌓음
