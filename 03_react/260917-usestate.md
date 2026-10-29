# React useState · 상태 관리 정리 (TIL)

## TIL(Today I Learned)

* `useState` Hook을 사용하여 함수형 컴포넌트 내부에서 상태(State)를 선언하고, Setter 함수 호출을 통해 컴포넌트를 리렌더링시키는 동작 원리를 학습
* 상태 변경 시 발생하는 일괄 처리(Batching) 특성과, 최신 이전 상태값을 보장하는 함수형 업데이트(`prev => prev + 1`)의 작성법을 습득
* `onChange` 이벤트와 `e.target.value`를 이용한 기본적인 입력 상태 관리 방식을 이해
* 객체 형태의 State, 전개 연산자(`...`), 계산된 속성명(`[name]: value`)을 활용하여 다중 `input` 상태를 하나의 핸들러로 통합 관리하는 효율적인 패턴을 학습
* 부모 컴포넌트에서 정의한 이벤트 핸들러 함수를 자식 컴포넌트의 Props로 전달하여 실행하는 이벤트 전달 방식을 체득

---

## 1. `useState` 기본 사용법과 리렌더링

* **한 줄 비유**: state는 컴포넌트가 들고 있는 **자기만의 메모장**임. 메모 내용이 바뀌면 React가 "화면 다시 그려야겠다" 하고 알아서 다시 그려줌.
* **왜 일반 변수로는 안 되나**: `let message = "..."`를 바꿔도 React는 그 사실을 모름 → 화면이 그대로임. 게다가 컴포넌트 함수가 다시 실행될 때마다 `let`은 처음 값으로 돌아감.
* **`useState`의 모양**: `const [값, 바꾸는함수] = useState(처음값);`
  * `값`: 지금 메모에 적힌 내용.
  * `바꾸는함수`(Setter): 메모를 고치는 **유일한 방법**. 이걸 불러야 React가 변화를 알아챔.
* **props vs state**: props는 부모가 준 **선물**(못 고침), state는 내가 가진 **메모장**(Setter로 고침).
* **리렌더링(re-rendering) 순서**
  1. Setter 호출 → React에 "값을 이걸로 바꿔줘" 예약.
  2. React가 컴포넌트 **함수를 처음부터 다시 실행**함.
  3. 이번 실행에서 `useState`는 **바뀐 값**을 돌려줌.
  4. 새로 만들어진 화면과 이전 화면을 비교해 다른 부분만 고침. 페이지 새로고침은 없음.

```jsx
/**
 * useState Hook: [상태 값, 상태 변경 함수] = useState(초기값)
 * - props: 부모가 전달하며 자식이 직접 수정할 수 없음
 * - state: 컴포넌트 내부에서 관리하며 Setter 함수로 변경
 * - Setter 함수가 호출되면 컴포넌트 함수가 다시 실행되면서 새로고침 없이 리렌더링됨
 */

const { useState } = React;

function MessageManager() {
    const [message, setMessage] = useState('초기 상태 메세지입니다.');
    const [textColor, setTextColor] = useState('black');

    const handleEnter = () => {
        setMessage('안녕하세요! 환영합니다.');
        setTextColor('blue');
    };

    const handleLeave = () => {
        setMessage('안녕히 가세요! 다음에 또 만나요~');
        setTextColor('red');
    };

    return (
        <>
            <h2 style={{ color: textColor }}>{message}</h2>
            <button onClick={handleEnter}>입장</button>
            <button onClick={handleLeave}>퇴장</button>
        </>
    );
}

```

* **따라가기**
  1. 처음 그릴 때 → `message`는 `'초기 상태 메세지입니다.'`, `textColor`는 `'black'` → 검은 글씨로 초기 메시지가 보임.
  2. "입장" 클릭 → `handleEnter` → Setter 두 개 호출.
  3. React가 두 변경을 **한 번에 모아서**(Batching) 컴포넌트를 한 번만 다시 실행 → `message`는 `'안녕하세요! 환영합니다.'`, `textColor`는 `'blue'` → 파란 글씨로 바뀜.
  4. "퇴장" 클릭 → 같은 흐름으로 빨간 글씨 `'안녕히 가세요! 다음에 또 만나요~'`

---

## 2. 일반 업데이트 vs 함수형 업데이트

* **핵심**: 한 번 그려진 화면 안에서 `number`는 **그 순간 찍어둔 사진(스냅샷)** 처럼 고정된 값임. Setter를 불러도 지금 실행 중인 코드의 `number`가 바로 바뀌지 않고, **다음 렌더링**에서야 새 값이 됨.
* **함수형 업데이트**: Setter에 값 대신 **함수**를 넘기는 방식 → `setNumber(prev => prev + 1)`. React가 대기 중인 변경을 차례로 처리하면서, 매번 **바로 앞 단계의 결과**를 `prev`로 넣어줌.

```jsx
/**
 * 함수형 업데이트
 * - Setter 함수의 인자로 콜백 함수를 전달 (prev => prev + 1)
 * - 콜백의 첫 번째 매개변수(prev)에는 최신 이전 상태값이 보장되어 동기적 연속 연산 시 안전함
 */

function Counter() {
    const [number, setNumber] = useState(0);

    // 이전 상태값을 안전하게 보장하여 2 증가
    const increaseByTwo = () => {
        setNumber((prev) => prev + 1);
        setNumber((prev) => prev + 1);
    };

    return (
        <>
            <h2>현재 숫자: {number}</h2>
            <button onClick={() => setNumber(number + 1)}>+1</button>

            {/* 현재 렌더링 시점의 number를 동일하게 참조하므로 의도한 +2가 아닌 +1로 동작 */}
            <button onClick={() => {
                setNumber(number + 1);
                setNumber(number + 1);
            }}>+2 (의도대로 안됨)</button>

            <button onClick={increaseByTwo}>+2 (함수형 업데이트)</button>
        </>
    );
}

```

* **따라가기: "+2 (의도대로 안됨)" (지금 `number`는 0)**
  1. 첫 번째 `setNumber(number + 1)` → `number`는 스냅샷 0 → "1로 바꿔줘" 예약.
  2. 두 번째 `setNumber(number + 1)` → `number`는 **여전히 0** → 또 "1로 바꿔줘" 예약.
  3. 다시 그리면 → 마지막 예약대로 `1`. **+1만 됨**.
* **따라가기: "+2 (함수형 업데이트)" (지금 `number`는 0)**
  1. 첫 번째 `prev => prev + 1` 예약, 두 번째도 예약.
  2. React가 차례로 처리: 첫 번째는 `prev = 0` → `1`, 두 번째는 앞 결과를 받아 `prev = 1` → `2`.
  3. 다시 그리면 → `2`. **의도대로 +2**.
* **정리**: 이전 값을 바탕으로 새 값을 계산할 때는 함수형 업데이트를 씀.

---

## 3. 객체 state로 여러 입력칸 관리하기

* **제어 컴포넌트**: `<input value={state} onChange={...}>`처럼 입력칸의 값을 **state가 쥐고 있는** 방식. 글자를 칠 때마다 `onChange` → state 변경 → 다시 그림 → 입력칸에 새 값 표시.
* **문제**: 입력칸이 10개면 state도 10개, 핸들러도 10개가 필요함.
* **해결**: state를 객체 하나로 만들고, 입력칸의 `name` 속성을 **객체의 키 이름과 똑같이** 맞춤. 그러면 핸들러 하나로 "이름이 `name`인 칸만 바꿔줘"라고 처리할 수 있음.
* **불변성**: state 객체를 직접 고치면(`form.email = ...`) React가 바뀐 줄 모름. 반드시 **새 객체**를 만들어 Setter에 넘김.
* **계산된 속성명 `[name]: value`**: 대괄호 안의 **변수 값**이 키 이름이 됨. `name`이 `"email"`이면 `{ email: value }`와 같음.

```jsx
/**
 * 하나의 객체 State로 여러 input 관리
 * - input의 name 속성과 state 객체의 key 이름을 동일하게 설정
 * - 기존 객체를 전개 연산자(...form)로 복사 후 [name]: value로 변경된 값만 덮어씀
 */

function SignupFormAdvanced() {
    const [form, setForm] = useState({
        username: '',
        email: ''
    });

    const { username, email } = form;

    const handleChange = (e) => {
        const { name, value } = e.target;
        setForm({
            ...form,        // 기존 form 객체 복사 (불변성 유지)
            [name]: value   // 변경된 input의 name을 Key로 사용하여 덮어씀
        });
    };

    return (
        <>
            <input
                type="text"
                name="username"
                value={username}
                onChange={handleChange}
            />
            <input
                type="email"
                name="email"
                value={email}
                onChange={handleChange}
            />
        </>
    );
}

```

* **따라가기: 이메일 칸에 `a`를 입력** (지금 `form`은 `{ username: "panda", email: "" }`)
  1. 이메일 칸에서 `onChange` → `handleChange(e)`
  2. `e.target`은 이메일 입력칸 → `name = "email"`, `value = "a"`
  3. `...form` → `{ username: "panda", email: "" }`를 펼쳐 새 객체에 복사.
  4. `[name]: value` → `email: "a"`로 덮어씀 → 새 객체 `{ username: "panda", email: "a" }`
  5. `setForm(새 객체)` → 다시 그림 → 이메일 칸에 `a`가 보임. `username`은 그대로 유지됨.

---

## 4. 이벤트 핸들러를 props로 넘기기

* 부모가 "버튼을 누르면 할 일"을 함수로 만들어 **props로 자식 버튼에게 넘김**. 자식은 그 함수를 자기 `<button>`의 `onClick`에 그대로 연결만 함.
* **장점**: 버튼 모양(자식)과 버튼이 하는 일(부모)을 나눠서, 같은 버튼 컴포넌트를 여러 곳에서 다른 동작으로 재사용할 수 있음.

```jsx
/**
 * 부모 컴포넌트에서 함수를 작성하고 자식 컴포넌트의 Props(onClick)로 넘겨 이벤트 연동
 */

function CustomButton({ onClick, children }) {
    const basicStyleButton = {
        backgroundColor: "black",
        color: "white",
        border: "none"
    };

    return (
        <button style={basicStyleButton} onClick={onClick}>
            {children}
        </button>
    );
}

function App() {
    const showWelcomeMessage = () => {
        alert("우리 사이트에 오신 것을 환영합니다.");
    };

    return (
        <CustomButton onClick={showWelcomeMessage}>
            <strong>환영 메세지</strong>
        </CustomButton>
    );
}

```

* **따라가기**
  1. `App`이 `CustomButton`에게 `onClick={showWelcomeMessage}`와 children(`<strong>환영 메세지</strong>`)을 넘김.
  2. `CustomButton`은 검은 배경·흰 글씨 버튼을 만들고, 안에 children을 넣고, `onClick`에 받은 함수를 연결.
  3. 화면에 굵은 글씨 "환영 메세지" 버튼이 보임.
  4. 클릭 → 연결된 `showWelcomeMessage` 실행 → `우리 사이트에 오신 것을 환영합니다.` 알림창.

---

## 느낀 점

* `setNumber(number + 1)`을 연속 호출해도 한 번만 증가하는 이유가 그 렌더 시점의 `number` 값(스냅샷)을 그대로 쓰기 때문임을 알게 되었고, 이전 값을 바탕으로 계산할 때는 콜백 형태의 함수형 업데이트(`prev => prev + 1`)를 써야 함을 깨달음
* 폼 입력을 처리할 때 입력 요소마다 `state`를 개별적으로 만드는 것보다 객체 `state`와 `[name]: value` 패턴을 결합하는 것이 코드의 중복을 획기적으로 줄여준다는 점을 확인함
* 자식 컴포넌트 제작 시 이벤트 핸들러와 `children` 속성을 적절히 커스텀 Props로 넘겨 재사용성이 뛰어난 UI 부품을 구성하는 방식을 익힘
