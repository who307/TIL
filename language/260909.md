# JavaScript

## TIL(Today I Learned)

* 옵셔널 체이닝 연산자(`?.`)와 null 병합 연산자(`??`)의 개념 및 단축 평가(`&&`, `||`)와의 차이점을 학습
* 생성자 함수(Constructor Function)를 통한 객체 생성 방식과 `new` 키워드의 동작 원리를 학습
* `new.target`을 활용해 `new` 연산자 없이 호출되었을 때를 대비하는 안전한 생성자 함수 패턴을 학습
* `Object.create()`를 이용한 프로토타입 상속과 프로토타입 체인(Prototype Chain)의 동작 방식을 학습
* 프로토타입 체인 상에서의 프로퍼티 검색 우선순위, 프로퍼티 오버라이딩(섀도잉), 그리고 `this` 바인딩 동작 방식을 학습

---

## 핵심 내용(코드)

### 옵셔널 체이닝 연산자 (`?.`)

```javascript
/**
 * 옵셔널 체이닝 연산자(?.)
 * 좌항의 피연산자가 null 또는 undefined인 경우 undefined를 반환하고
 * 그렇지 않으면 우항의 프로퍼티 참조를 이어간다.
 * 객체 속성을 안전하게 읽고 싶을 때 사용한다.
 */

const obj = null;
// let val = obj.value; // TypeError: Cannot read properties of null

const val = obj?.value;
console.log(val); // undefined

// 옵셔널 체이닝 이전에는 논리연산자 &&를 사용한 단축 평가로 확인
// 단, 빈 문자열("")과 같은 Falsy한 값을 false 취급해서 생기는 문제가 발생할 수 있음
const str = "";

// const len = str && str.length; // len에 빈 문자열("")이 담김
const len = str?.length;

console.log(len); // 0

```

### null 병합 연산자 (`??`)

```javascript
/**
 * null 병합 연산자(??)
 * 좌항의 피연산자가 null 또는 undefined인 경우 우항의 피연산자를 반환하고,
 * 그렇지 않으면 좌항의 피연산자를 반환한다.
 */

const test = null ?? "기본 값";
console.log(test); // "기본 값"

// OR 연산자(||)는 빈 문자열("")이나 0 같은 Falsy한 값도 false로 취급하는 문제가 있음
// const value = "" || "기본 값"; // "기본 값"으로 설정됨

const value = "" ?? "기본 값";
console.log(value); // ""

```

### 생성자 함수

```javascript
/**
 * 생성자 함수
 * 동일한 구조의 객체를 여러 개 생성할 때 유용하게 활용
 */

// 객체 리터럴 방식 (중복 발생)
const student1 = {
    name: "판다",
    age: 5,
    getInfo: function () {
        return `${this.name}는 ${this.age}세 입니다.`;
    },
};

// 생성자 함수 방식
function Student(name, age) {
    this.name = name;
    this.age = age;
    this.getInfo = function () {
        return `${this.name}는 ${this.age}세 입니다.`;
    };
}

const student3 = new Student("원숭이", 40);
const student4 = new Student("코끼리", 25);

console.log(student3.getInfo()); // 원숭이는 40세 입니다.
console.log(student4.getInfo()); // 코끼리는 25세 입니다.

console.log("같은 객체인가:", student3 === student4); // false

student3.age = 32;
console.log(student3.getInfo()); // 원숭이는 32세 입니다.
console.log(student4.getInfo()); // 코끼리는 25세 입니다.

```

### 인스턴스 생성 과정과 `new.target`

```javascript
"use strict"; // 엄격 모드

function Student(name, age) {
    console.log(`this가 있는가:`, this !== undefined);
    console.log(`name이 있는가:`, "name" in this);
    this.name = name;
    this.age = age;
    this.getInfo = function () {
        return `${this.name}는 ${this.age}세 입니다.`;
    };
    console.log(`초기화한 값: `, this.name, this.age);
}

const student1 = new Student("호랑이", 10);
console.log(`객체를 반환받았는가:`, typeof student1 === "object");

function Dog(name, age) {
    // new.target: new 키워드로 호출되면 함수 자신을 가리키고, 일반 호출이면 undefined이다.
    if (!new.target) {
        console.log("new 없이 호출했습니다. new를 붙여서 다시 실행합니다.");
        return new Dog(name, age);
    }
    this.name = name;
    this.age = age;
}

const dog = new Dog("바둑이", 3);
console.log(dog);

for (let key in dog) {
    console.log(key);
}

```

### 프로토타입 체인 (Prototype Chain)

```javascript
const user = {
    id: "user",
    activate: true,
    login: function () {
        console.log(`${this.id}님이 로그인 되었습니다.`);
    },
};

// Object.create(): 새로운 빈 객체를 만들고, 그 객체의 프로토타입을 인자로 전달된 객체로 설정
const student = Object.create(user);
student.passion = true;

console.log(student.activate); // true (user 객체에서 상속받음)
console.log(student); // { passion: true }

// student의 프로토타입 확인
console.log(Object.getPrototypeOf(student)); // user 객체

console.log(`자신의 activate:`, Object.hasOwn(student, "activate")); // false
console.log(`자신의 passion:`, Object.hasOwn(student, "passion")); // true

const greedyStudent = Object.create(student);
greedyStudent.greed = true;
greedyStudent.id = "student01"; // 프로퍼티 섀도잉(오버라이딩)

console.log(greedyStudent.activate); // true
console.log(greedyStudent.passion); // true
console.log(greedyStudent.missing); // undefined

// 호출 주체에 따른 this 바인딩
greedyStudent.login(); // student01님이 로그인 되었습니다.

delete greedyStudent.id; // 가려졌던 상속 프로퍼티가 다시 드러남
console.log(greedyStudent.id); // user
greedyStudent.login(); // user님이 로그인 되었습니다.

```

---

## 느낀 점

* 단축 평가 방식(`&&`, `||`) 사용 시 빈 문자열(`""`)이나 `0`이 Falsy 처리되어 예기치 못한 기본값이 할당되는 문제를 `?.`와 `??` 연산자를 활용해 깔끔하게 해결할 수 있음을 체감함
* 생성자 함수 작성 시 `new` 키워드를 누락하면 전역 객체가 오염되거나 에러가 발생할 수 있는데, `new.target`을 활용한 방어적 프로그래밍 방식을 배움
* 자바스크립트의 상속 구조가 클래스 기반 언어와 달리 프로토타입 체인 참조로 이어져 있다는 점을 이해하고, 메서드 호출 시 `this`가 호출하는 시점의 주체를 가리킨다는 메커니즘을 명확히 정리함