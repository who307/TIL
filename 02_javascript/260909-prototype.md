# JavaScript 옵셔널 체이닝 · 생성자 함수 · 프로토타입 정리 (TIL)

## TIL(Today I Learned)

* 옵셔널 체이닝 연산자(`?.`)와 null 병합 연산자(`??`)의 개념 및 단축 평가(`&&`, `||`)와의 차이점을 학습
* 생성자 함수(Constructor Function)를 통한 객체 생성 방식과 `new` 키워드의 동작 원리를 학습
* `new.target`을 활용해 `new` 연산자 없이 호출되었을 때를 대비하는 안전한 생성자 함수 패턴을 학습
* `Object.create()`를 이용한 프로토타입 상속과 프로토타입 체인(Prototype Chain)의 동작 방식을 학습
* 프로토타입 체인 상에서의 프로퍼티 검색 우선순위, 프로퍼티 오버라이딩(섀도잉), 그리고 `this` 바인딩 동작 방식을 학습

---

## 1. 옵셔널 체이닝 연산자 (`?.`)

* **한 줄 비유**: 상자를 열기 전에 **"상자가 있긴 해?"** 하고 먼저 확인하는 것. 상자가 없으면 열지 않고 "없음(`undefined`)"이라고만 말함.
* **왜 필요한가**: `null`이나 `undefined`에서 `.value`처럼 프로퍼티를 꺼내려 하면 `TypeError`가 나면서 프로그램이 멈춤.
* **동작**: `A?.B`
  1. A가 `null` 또는 `undefined`인가?
  2. 그렇다 → B를 찾지 않고 바로 `undefined`를 돌려줌.
  3. 아니다 → 평소처럼 `A.B`를 꺼냄.
* **`&&`와의 차이**: `str && str.length`는 `str`이 Falsy(빈 문자열, 0 등)이기만 해도 멈춤. `?.`는 **오직 `null`/`undefined`일 때만** 멈춤.

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

* **한 줄씩 보기**
  * `obj.value` → `obj`가 `null`이라 꺼낼 수 없음 → `TypeError` → 주석 처리.
  * `obj?.value` → `obj`가 `null`이므로 1~2단계에서 멈춤 → `undefined`
  * `str && str.length` → `""`는 Falsy → `&&`는 왼쪽이 Falsy면 왼쪽 값을 그대로 돌려줌 → `len`에 `""`가 들어감. 우리가 원한 건 길이 0이었는데 엉뚱한 값이 됨.
  * `str?.length` → `""`는 `null`/`undefined`가 아님 → 정상적으로 `.length`를 꺼냄 → `0`

---

## 2. null 병합 연산자 (`??`)

* **한 줄 비유**: "값이 **아예 없을 때만** 기본값을 쓸게"라는 규칙.
* **동작**: `A ?? B`
  1. A가 `null` 또는 `undefined`인가?
  2. 그렇다 → B(기본값)를 돌려줌.
  3. 아니다 → A를 그대로 돌려줌. A가 `""`이나 `0`이어도 A를 씀.
* **`||`와의 차이**: `A || B`는 A가 Falsy이기만 하면 B를 씀. 그래서 사용자가 일부러 0이나 빈 문자열을 넣어도 기본값으로 바뀌어 버림.

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

* **한 줄씩 보기**
  * `null ?? "기본 값"` → 왼쪽이 `null` → 오른쪽 사용 → `기본 값`
  * `"" || "기본 값"` → `""`는 Falsy → 오른쪽 사용 → `기본 값` (원래 값 `""`가 사라짐)
  * `"" ?? "기본 값"` → `""`는 `null`/`undefined`가 아님 → 왼쪽 그대로 → `""`

---

## 3. 생성자 함수 (Constructor Function)

* **한 줄 비유**: 생성자 함수는 **붕어빵 틀**임. 틀 하나로 팥붕, 슈붕을 계속 찍어낼 수 있음. 찍혀 나온 붕어빵 하나하나를 **인스턴스**라고 부름.
* **왜 필요한가**: 객체 리터럴(`{ }`)로 학생을 100명 만들면 같은 구조를 100번 복사해서 써야 함. 생성자 함수는 구조를 한 번만 쓰고 값만 바꿔 찍어냄.
* **규칙**: 함수 이름을 **대문자로 시작**하고(`Student`), 부를 때 앞에 **`new`** 를 붙임.

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

* **한 줄씩 보기**
  * `student1` → 한 명을 직접 손으로 만든 방식. 다른 학생을 만들려면 이 덩어리를 또 써야 함.
  * `new Student("원숭이", 40)` → 틀에 "원숭이", 40을 넣고 찍음 → `{ name: "원숭이", age: 40, getInfo }`
  * `student3.getInfo()` → `원숭이는 40세 입니다.` / `student4.getInfo()` → `코끼리는 25세 입니다.`
  * `student3 === student4` → 같은 틀에서 나왔어도 **서로 다른 붕어빵** → `false`
  * `student3.age = 32` → `student3`만 바뀜 → `원숭이는 32세 입니다.` / `student4`는 그대로 `25세`

---

## 4. `new`가 하는 일과 `new.target`

* **`new Student(...)`를 쓰면 일어나는 일 (4단계)**
  1. 빈 객체 `{}`를 새로 만듦.
  2. 함수 안의 `this`가 그 빈 객체를 가리키게 함.
  3. 함수 본문을 실행 → `this.name = ...`으로 빈 객체에 칸을 채움.
  4. 다 채운 객체를 자동으로 돌려줌 (`return`을 안 써도 됨).
* **`new`를 깜빡하면?**: 위 단계가 일어나지 않음. 엄격 모드(`"use strict"`)에서는 `this`가 `undefined`라 `this.name = ...`에서 에러가 남. 엄격 모드가 아니면 `this`가 전역 객체가 되어 전역 변수를 더럽힘.
* **`new.target`**: 함수가 `new`로 불렸으면 그 함수 자신, 그냥 불렸으면 `undefined`가 들어 있는 값. 이걸로 "new를 깜빡했는지" 검사할 수 있음.

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

* **실행 결과 따라가기**
  * `this가 있는가: true` → 1~2단계에서 빈 객체가 만들어져 `this`가 가리키고 있음.
  * `name이 있는가: false` → 아직 3단계 전이라 빈 객체임.
  * `초기화한 값:  호랑이 10` → 3단계에서 칸이 채워짐.
  * `객체를 반환받았는가: true` → 4단계에서 객체가 돌아옴.
  * `new Dog("바둑이", 3)` → `new`로 불렀으니 `new.target`이 `Dog` → `if`를 건너뛰고 정상 생성.
  * `console.log(dog)` → `Dog { name: '바둑이', age: 3 }`
  * `for...in` → 키를 하나씩 → `name`, `age`
  * 만약 `Dog("바둑이", 3)`처럼 `new` 없이 부르면 → `new.target`이 `undefined` → 안내 문구를 찍고 **스스로 `new Dog(...)`를 다시 불러** 올바른 객체를 돌려줌.

---

## 5. 프로토타입 체인 (Prototype Chain)

* **한 줄 비유**: 모르는 단어가 있으면 **내 공책 → 형 공책 → 아빠 공책** 순서로 찾아보는 것. 이렇게 이어진 공책 줄을 **프로토타입 체인**이라고 함.
* **프로토타입(prototype)**: 객체가 "모르면 물어볼" **부모 객체**. 자기한테 없는 프로퍼티는 부모에게서 빌려 씀(상속).
* **`Object.create(부모)`**: 빈 객체를 새로 만들고, 그 객체의 부모(프로토타입)를 `부모`로 정해줌.
* **찾는 순서**: 내 칸에 있나? → 없으면 부모 → 없으면 부모의 부모 → … → 끝까지 없으면 `undefined`
* **섀도잉(Shadowing)**: 부모에게 있는 이름과 **같은 이름을 내가 가지면**, 내 것이 부모 것을 가림. 그림자처럼 덮는다고 해서 붙은 이름임.

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

* **체인 구조**: `greedyStudent` → (부모) `student` → (부모) `user`

| 객체 | 자기가 직접 가진 칸 |
| :--- | :--- |
| `user` | `id: "user"`, `activate: true`, `login` |
| `student` | `passion: true` |
| `greedyStudent` | `greed: true`, `id: "student01"` |

* **한 줄씩 따라가기**
  * `student.activate` → `student`에 없음 → 부모 `user`에서 찾음 → `true`
  * `console.log(student)` → **자기가 직접 가진 칸만** 보여줌 → `{ passion: true }`
  * `Object.getPrototypeOf(student)` → 부모를 알려줌 → `user` 객체
  * `Object.hasOwn(student, "activate")` → "**직접** 가지고 있니?" → 빌려 쓰는 것이라 `false`. `passion`은 직접 가짐 → `true`
  * `greedyStudent.activate` → 나 → `student` → `user`까지 올라가 찾음 → `true`
  * `greedyStudent.passion` → 나 → `student`에서 찾음 → `true`
  * `greedyStudent.missing` → 끝까지 없음 → `undefined`
  * `greedyStudent.login()` → `login`은 `user`에서 빌려왔지만, **부른 주체가 `greedyStudent`** 이므로 `this`는 `greedyStudent` → `this.id`는 내 칸의 `"student01"` → `student01님이 로그인 되었습니다.`
  * `delete greedyStudent.id` → 내 `id`가 사라짐 → 가려져 있던 `user`의 `id`가 다시 보임 → `user`
  * 다시 `login()` → `this.id`를 찾아 올라가 `user`의 `"user"` → `user님이 로그인 되었습니다.`

---

## 느낀 점

* 단축 평가 방식(`&&`, `||`) 사용 시 빈 문자열(`""`)이나 `0`이 Falsy 처리되어 예기치 못한 기본값이 할당되는 문제를 `?.`와 `??` 연산자를 활용해 깔끔하게 해결할 수 있음을 체감함
* 생성자 함수 작성 시 `new` 키워드를 누락하면 전역 객체가 오염되거나 에러가 발생할 수 있는데, `new.target`을 활용한 방어적 프로그래밍 방식을 배움
* 자바스크립트의 상속 구조가 클래스 기반 언어와 달리 프로토타입 체인 참조로 이어져 있다는 점을 이해하고, 메서드 호출 시 `this`가 호출하는 시점의 주체를 가리킨다는 메커니즘을 명확히 정리함
