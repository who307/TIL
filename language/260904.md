# JavaScript

## TIL(Today I Learned)
- 변수(variable)의 개념과 var, let, const 키워드의 차이를 학습
- 변수명 작성 규칙과 camelCase 표기법을 학습
- 자바스크립트는 동적 타입 언어이며, Number·String·Boolean·null·undefined 등 기본 타입을 학습
- 템플릿 리터럴을 사용한 문자열 표현 방법을 학습
- 암묵적 타입 변환(문자열, 숫자, 불린 타입 변환)의 동작 방식을 학습
- Truthy/Falsy 값의 개념을 학습

## 핵심 내용(코드)

### 변수 선언(var, let, const)

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

### 변수명 규칙

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

### 데이터 타입

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

### 템플릿 리터럴

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

### 암묵적 타입 변환

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

### Truthy / Falsy 값

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

## 느낀 점
- var, let, const의 차이와 각 키워드가 도입된 이유를 이해
- 자바스크립트가 동적 타입 언어라는 개념과 각 데이터 타입의 특징을 배움
- 템플릿 리터럴을 사용하면 문자열 연결보다 가독성 좋은 코드를 작성할 수 있다는 점을 알게됨
- 암묵적 타입 변환의 동작 방식을 이해하면서 예상치 못한 버그를 방지하려면 타입을 명확히 다루는 습관이 필요하다고 느낌
- Truthy/Falsy 개념을 알고 나니 조건문 작성 시 값의 참·거짓 평가를 더 정확히 예측할 수 있게 됨
