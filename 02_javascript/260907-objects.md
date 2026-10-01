# JavaScript

## TIL(Today I Learned)
* 객체(Object)의 개념과 키-값(key-value) 쌍으로 구성된 프로퍼티 및 메서드 작성법을 학습
* 마침표 표기법과 대괄호 표기법의 차이 및 변수/특수문자 키 접근법을 학습
* 객체 프로퍼티의 값 수정, 추가, delete 연산자를 통한 삭제, in 연산자 및 for...in 순회법을 학습
* if-else 및 switch 문을 활용한 조건 분기 처리 방법을 학습
* for, while, do-while 반복문 및 break, continue 키워드를 통한 제어 흐름을 학습

## 핵심 내용(코드)

### 객체 선언과 프로퍼티 접근법

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

### 프로퍼티 동적 조작 및 순회

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

### 조건문 (if-else, switch)

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

### 반복문 (for, while) 및 제어문 (break, continue)

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

## 느낀 점
* 마침표 표기법과 대괄호 표기법의 쓰임새를 구분하면서, 동적 키 접근이나 특수문자 키 처리 시 대괄호 표기법이 꼭 필요하다는 점을 명확히 이해함
* `const`로 선언된 객체라 할지라도 객체 자체의 재할당만 금지될 뿐, 내부 프로퍼티의 값 변경·추가·삭제는 자유롭게 가능하다는 특징을 배움
* 반복문 내에서 `continue`와 `break`를 적절히 제어하여 불필요한 연산을 줄이고 코드 흐름을 효과적으로 관리하는 방법을 익힘
* 조건문과 반복문을 다루면서 로직 조건식 설정에 따라 코드 동작이 어떻게 달라지는지 흐름을 파악하는 기본기를 쌓음