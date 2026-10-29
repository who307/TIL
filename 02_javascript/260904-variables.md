# JavaScript 변수 · 데이터 타입 · 형변환 정리 (TIL)

## TIL(Today I Learned)

* 변수(variable)의 개념과 var, let, const 키워드의 차이를 학습
* 변수명 작성 규칙과 camelCase 표기법을 학습
* 자바스크립트는 동적 타입 언어이며, Number·String·Boolean·null·undefined 등 기본 타입을 학습
* 템플릿 리터럴을 사용한 문자열 표현 방법을 학습
* 암묵적 타입 변환(문자열, 숫자, 불린 타입 변환)의 동작 방식을 학습
* Truthy/Falsy 값의 개념을 학습

---

## 1. 변수 선언 (var, let, const)

* **한 줄 비유**: 변수는 **이름표가 붙은 상자**임. 상자(메모리)에 값을 넣어두고, 나중에 이름표(변수명)로 다시 꺼내 씀.
* **선언과 할당**
  * **선언**: 상자를 만들고 이름표를 붙이는 것 → `var number;`
  * **할당**: 상자에 값을 넣는 것 → `number = 307;`
  * **재할당**: 상자 안의 값을 다른 값으로 바꾸는 것
* **키워드 3개의 차이**

| 키워드 | 재할당 | 특징 |
| :--- | :--- | :--- |
| `var` | 가능 | 예전(ES5) 방식. 같은 이름으로 또 선언해도 에러가 안 나서 실수하기 쉬움 |
| `let` | 가능 | ES6에서 추가. 값이 바뀌어야 하는 변수에 씀 |
| `const` | **불가** | ES6에서 추가. 한 번 넣은 값을 바꿀 수 없음. 선언할 때 반드시 값을 넣어야 함 |

```javascript
// var: ES5까지 사용했던 유일한 키워드
// 단점을 보완하기 위해 ES6에서 새로운 키워드인 let, const를 도입
var number;
number = 307;
console.log(number);

let greeting = "Hello, Node.js";
console.log(greeting);

greeting = "welcome!";
console.log(greeting);

// const: 재할당 금지
// 반드시 선언과 동시에 초기화해야 한다.
const num = 1;
// num = 2; // 재할당 불가
console.log(num);
```

* **한 줄씩 보기**
  * `var number;` → `number`라는 상자를 만듦. 아직 값이 없어서 안에는 `undefined`가 들어 있음.
  * `number = 307;` → 상자에 307을 넣음. `console.log(number)` → `307` 출력.
  * `let greeting = "Hello, Node.js";` → 상자를 만들면서 바로 값을 넣음(선언 + 할당). 출력 → `Hello, Node.js`
  * `greeting = "welcome!";` → `let`이라 값을 바꿀 수 있음. 출력 → `welcome!`
  * `const num = 1;` → 값 1로 고정된 상자. `num = 2`를 실행하면 `TypeError`가 나므로 주석 처리해 둠. 출력 → `1`
* **언제 무엇을 쓰나**: 기본은 `const`, 값이 바뀌어야 하면 `let`. `var`는 쓰지 않음.

---

## 2. 변수명 규칙

* **꼭 지켜야 하는 규칙** (어기면 에러)
  1. 문자, 숫자, `_`, `$`만 쓸 수 있음.
  2. **숫자로 시작할 수 없음** → `1age` 불가.
  3. **예약어**(자바스크립트가 이미 쓰는 단어: `if`, `for`, `const` 등)는 이름으로 쓸 수 없음.
* **지키면 좋은 관례**: **카멜케이스(camelCase)** — 첫 단어는 소문자, 다음 단어부터 첫 글자를 대문자로 씀. 낙타 등의 혹처럼 울퉁불퉁해서 붙은 이름임. 예: `user` + `name` → `userName`

```javascript
// 변수 이름에는 문자, 숫자, _, $ 등을 사용할 수 있다.
// 변수 이름은 숫자로 시작할 수 없다.
// 카멜케이스(camelCase)를 사용하는 것이 일반적이다.

// const 1age = 1;   // 숫자로 시작 불가
// const if = true;  // 예약어 사용 금지

const userName = "tiger";
const userAge = 5;

console.log(userName);
console.log(userAge);
```

* **실행 결과**: `tiger`, `5`가 차례로 출력됨. 주석 처리한 두 줄은 실제로 실행하면 `SyntaxError`가 남.

---

## 3. 데이터 타입

* **데이터 타입**: 상자에 들어 있는 값의 **종류**. 숫자인지, 글자인지, 참/거짓인지에 따라 할 수 있는 일이 다름.
* **동적 타입 언어**: 자바스크립트는 상자에 "숫자 전용" 같은 딱지를 붙이지 않음. 같은 변수에 숫자를 넣었다가 글자를 넣어도 됨. 값의 타입은 **실행할 때** 정해짐.
* `typeof 값` → 그 값의 타입 이름을 문자열로 알려줌.

```javascript
// 1. 숫자(Number)
const age = 30;
console.log(age);
console.log(typeof age);

// 2. 문자열(String)
const string = "문자열";
console.log(string);
console.log(typeof string);

// 3. 불린(Boolean)
const boolean = true;
console.log(boolean);
console.log(typeof boolean);

// 4. null: 값이 없음을 의도적으로 명시할 때 사용
const address = null;
console.log(address);
console.log(typeof address);

// 5. undefined: 변수에 값이 할당되지 않았을 때 자동으로 할당되는 값
let salary;
console.log(salary);
console.log(typeof salary);
```

* **실행 결과**

| 변수 | 출력 | `typeof` 결과 | 설명 |
| :--- | :--- | :--- | :--- |
| `age` | `30` | `"number"` | 정수·소수 모두 Number 하나로 다룸 |
| `string` | `문자열` | `"string"` | 따옴표로 감싼 글자 |
| `boolean` | `true` | `"boolean"` | 참(`true`) 또는 거짓(`false`) 두 가지뿐 |
| `address` | `null` | `"object"` | "값이 없다"고 **내가 일부러** 넣은 값. `"object"`로 나오는 건 자바스크립트 초창기의 버그가 그대로 남은 것임 |
| `salary` | `undefined` | `"undefined"` | 값을 넣은 적이 없어서 **자동으로** 들어간 값 |

* **null vs undefined**: 빈 의자에 "자리 비움" 쪽지를 일부러 올려둔 것이 `null`, 아직 아무도 의자를 정하지 않은 것이 `undefined`임.

---

## 4. 템플릿 리터럴 (Template Literal)

* **뜻**: 따옴표 대신 **백틱(`` ` ``)** 으로 문자열을 감싸고, 그 안에 `${변수}`를 넣어 값을 끼워 넣는 문법.
* **왜 필요한가**: `+`로 이어 붙이면 띄어쓰기와 따옴표를 하나하나 신경 써야 해서 읽기 어렵고 실수가 잦음.

```javascript
const lastName = "문";
const firstName = "건후";

// 기존 문자열 연결 방식
console.log("제 이름은 " + lastName + firstName + " 입니다");

// 템플릿 리터럴 사용(표현식 삽입, ${})
console.log(`제 이름은 ${lastName}${firstName} 입니다`);

// 백틱을 사용하면 줄바꿈이 허용되고 모든 공백이 그대로 적용된다.
const str = `안녕하세요.
반갑습니다.`;
console.log(str);
```

* **한 줄씩 보기**
  * `"제 이름은 " + lastName + firstName + " 입니다"` → 조각 4개를 순서대로 이어 붙임 → `제 이름은 문건후 입니다`
  * `` `제 이름은 ${lastName}${firstName} 입니다` `` → `${}` 자리에 변수 값이 들어감 → 결과는 위와 똑같지만 문장 모양 그대로 읽힘.
  * `` const str = `안녕하세요.(엔터)반갑습니다.` `` → 백틱 안에서는 엔터를 친 그대로 줄바꿈이 됨 → 두 줄로 출력됨.

---

## 5. 암묵적 타입 변환

* **뜻**: 서로 다른 타입끼리 계산하면, 자바스크립트가 **알아서 한쪽 타입을 바꿔서** 계산하는 것. "암묵적"은 내가 시키지 않았는데 몰래 한다는 뜻임.
* **규칙 요약**
  1. `+`인데 한쪽이라도 **문자열**이면 → 나머지도 문자열로 바꿔서 **이어 붙임**.
  2. `-`, `*`, `/`, `%`는 문자열을 이어 붙이는 기능이 없음 → 문자열을 **숫자로 바꿔서** 계산함.
  3. 숫자로 바꿀 수 없는 문자열(`"js"`)은 `NaN`(Not a Number, "숫자가 아님")이 됨.

```javascript
// 문자열 타입으로 변환
console.log(10 + "20");        // "1020" (+는 문자열이 끼어 있으면 연결 연산자로 동작)
console.log(typeof (10 + "20"));
console.log(true + "");        // "true"
console.log(null + "");        // "null"
console.log(1 + "");           // "1"

// 숫자 타입으로 변환
console.log(10 - "5");
console.log(10 * "5");
console.log(10 / "5");
console.log(10 % "js");        // NaN - 숫자 변환 불가로 연산 수행 불가

// 숫자와 문자열 비교 시 문자열을 숫자로 변환한 뒤 비교
console.log(10 > "5");

// 단항 연산자(+)를 이용한 숫자 변환
console.log(+"");
console.log(+true);
console.log(+false);
```

* **한 줄씩 따라가기**
  * `10 + "20"` → `"20"`이 문자열 → 10을 `"10"`으로 바꿈 → `"10" + "20"` → `"1020"`. `typeof` → `"string"`
  * `true + ""`, `null + ""`, `1 + ""` → 빈 문자열과 더하면 그 값이 글자로 바뀜 → `"true"`, `"null"`, `"1"`
  * `10 - "5"` → `"5"`를 숫자 5로 → `5`. 같은 방식으로 `10 * "5"` → `50`, `10 / "5"` → `2`
  * `10 % "js"` → `"js"`는 숫자로 못 바꿈 → `NaN`
  * `10 > "5"` → `"5"`를 숫자 5로 바꿔 비교 → `true`
  * `+""` → 빈 문자열은 숫자 `0`. `+true` → `1`, `+false` → `0`

---

## 6. Truthy / Falsy

* **뜻**: `if` 같은 조건문에 `true`/`false`가 아닌 값을 넣으면, 자바스크립트가 그 값을 참이나 거짓으로 **취급**함.
  * **Falsy**: 거짓으로 취급되는 값. 아래 6개뿐임.
  * **Truthy**: Falsy가 아닌 **나머지 모든 값** (예: `"0"`, `"false"`, `[]`, `{}`도 Truthy).

```text
- Falsy 값(false로 평가되는 값)
false
0 (숫자 0)
"" (빈 문자열)
null
undefined
NaN (Not a Number)
```

```javascript
if (10 > 5) console.log("참 입니다.");
if ("") console.log("Falsy");
if (0) console.log(0);
```

* **한 줄씩 보기**
  * `if (10 > 5)` → `true` → `참 입니다.` 출력.
  * `if ("")` → 빈 문자열은 Falsy → 거짓으로 취급 → **아무것도 출력 안 됨**.
  * `if (0)` → 숫자 0은 Falsy → **아무것도 출력 안 됨**.

---

## 느낀 점

* var, let, const의 차이와 각 키워드가 도입된 이유를 이해함
* 자바스크립트가 동적 타입 언어라는 개념과 각 데이터 타입의 특징을 배움
* 템플릿 리터럴을 사용하면 문자열 연결보다 가독성 좋은 코드를 작성할 수 있다는 점을 알게 됨
* 암묵적 타입 변환의 동작 방식을 이해하면서 예상치 못한 버그를 방지하려면 타입을 명확히 다루는 습관이 필요하다고 느낌
* Truthy/Falsy 개념을 알고 나니 조건문 작성 시 값의 참·거짓 평가를 더 정확히 예측할 수 있게 됨
