# React

## TIL(Today I Learned)

* `useState` Hook을 사용하여 함수형 컴포넌트 내부에서 상태(State)를 선언하고, Setter 함수 호출을 통해 컴포넌트를 리렌더링시키는 동작 원리를 학습
* 상태 변경 시 발생하는 일괄 처리(Batching) 특성과, 최신 이전 상태값을 보장하는 함수형 업데이트(`prev => prev + 1`)의 작성법을 습득
* `onChange` 이벤트와 `e.target.value`를 이용한 기본적인 입력 상태 관리 방식을 이해
* 객체 형태의 State, 전개 연산자(`...`), 계산된 속성명(`[name]: value`)을 활용하여 다중 `input` 상태를 하나의 핸들러로 통합 관리하는 효율적인 패턴을 학습
* 부모 컴포넌트에서 정의한 이벤트 핸들러 함수를 자식 컴포넌트의 Props로 전달하여 실행하는 이벤트 전달 방식을 체득



---

## 핵심 내용(코드)

### `useState` 기본 사용법 및 리렌더링

```jsx
/**
 * useState Hook: [상태 값, 상태 변경 함수] = useState(초기값)[cite: 29]
 * - props: 부모가 전달하며 자식이 직접 수정할 수 없음[cite: 29]
 * - state: 컴포넌트 내부에서 관리하며 Setter 함수로 변경[cite: 29]
 * - Setter 함수가 호출되면 컴포넌트 함수가 다시 실행되면서 새로고침 없이 리렌더링됨[cite: 29]
 */

const { useState } = React;[cite: 29]

function MessageManager() {
    const [message, setMessage] = useState('초기 상태 메세지입니다.');[cite: 29]
    const [textColor, setTextColor] = useState('black');[cite: 29]

    const handleEnter = () => {
        setMessage('안녕하세요! 환영합니다.');[cite: 29]
        setTextColor('blue');[cite: 29]
    };

    const handleLeave = () => {
        setMessage('안녕히 가세요! 다음에 또 만나요~');[cite: 29]
        setTextColor('red');[cite: 29]
    };

    return (
        <>
            <h2 style={{ color: textColor }}>{message}</h2>[cite: 29]
            <button onClick={handleEnter}>입장</button>[cite: 29]
            <button onClick={handleLeave}>퇴장</button>[cite: 29]
        </>
    );
}

```

### 일반 업데이트 vs 함수형 업데이트

```jsx
/**
 * 함수형 업데이트
 * - Setter 함수의 인자로 콜백 함수를 전달 (prev => prev + 1)[cite: 29]
 * - 콜백의 첫 번째 매개변수(prev)에는 최신 이전 상태값이 보장되어 동기적 연속 연산 시 안전함[cite: 29]
 */

function Counter() {
    const [number, setNumber] = useState(0);[cite: 29]

    // 이전 상태값을 안전하게 보장하여 2 증가
    const increaseByTwo = () => {
        setNumber((prev) => prev + 1);[cite: 29]
        setNumber((prev) => prev + 1);[cite: 29]
    };

    return (
        <>
            <h2>현재 숫자: {number}</h2>[cite: 29]
            <button onClick={() => setNumber(number + 1)}>+1</button>[cite: 29]
            
            {/* 현재 렌더링 시점의 number를 동일하게 참조하므로 의도한 +2가 아닌 +1로 동작 */}
            <button onClick={() => {
                setNumber(number + 1);[cite: 29]
                setNumber(number + 1);[cite: 29]
            }}>+2 (의도대로 안됨)</button>[cite: 29]
            
            <button onClick={increaseByTwo}>+2 (함수형 업데이트)</button>[cite: 29]
        </>
    );
}

```

### 객체 State를 활용한 효율적인 다중 Input 관리

```jsx
/**
 * 하나의 객체 State로 여러 input 관리
 * - input의 name 속성과 state 객체의 key 이름을 동일하게 설정[cite: 30]
 * - 기존 객체를 전개 연산자(...form)로 복사 후 [name]: value로 변경된 값만 덮어씀[cite: 30]
 */

function SignupFormAdvanced() {
    const [form, setForm] = useState({
        username: '',
        email: ''
    });[cite: 30]

    const { username, email } = form;[cite: 30]

    const handleChange = (e) => {
        const { name, value } = e.target;[cite: 30]
        setForm({
            ...form,        // 기존 form 객체 복사 (불변성 유지)[cite: 30]
            [name]: value   // 변경된 input의 name을 Key로 사용하여 덮어씀[cite: 30]
        });
    };

    return (
        <>
            <input
                type="text"
                name="username"
                value={username}
                onChange={handleChange}
            />[cite: 30]
            <input
                type="email"
                name="email"
                value={email}
                onChange={handleChange}
            />[cite: 30]
        </>
    );
}

```

### Props를 통한 이벤트 핸들러 전달

```jsx
/**
 * 부모 컴포넌트에서 함수를 작성하고 자식 컴포넌트의 Props(onClick)로 넘겨 이벤트 연동[cite: 30]
 */

function CustomButton({ onClick, children }) {
    const basicStyleButton = {
        backgroundColor: "black",
        color: "white",
        border: "none"
    };[cite: 30]

    return (
        <button style={basicStyleButton} onClick={onClick}>
            {children}
        </button>
    );[cite: 30]
}

function App() {
    const showWelcomeMessage = () => {
        alert("우리 사이트에 오신 것을 환영합니다.");[cite: 30]
    };

    return (
        <CustomButton onClick={showWelcomeMessage}>
            <strong>환영 메세지</strong>
        </CustomButton>
    );[cite: 30]
}

```

---

## 느낀 점

* `useState`의 Setter 함수를 호출할 때 이전 State 참조 오류가 발생하는 이유가 React의 배치(Batching) 처리 때문임을 알게 되었고, 연속적인 연산 시에는 반드시 콜백 형태의 함수형 업데이트(`prev => prev + 1`)를 써야 함을 깨달음
* 폼 입력을 처리할 때 입력 요소마다 `state`를 개별적으로 만드는 것보다 객체 `state`와 `[name]: value` 패턴을 결합하는 것이 코드의 중복을 획기적으로 줄여준다는 점을 확인 함
* 자식 컴포넌트 제작 시 이벤트 핸들러와 `children` 속성을 적절히 커스텀 Props로 넘겨 재사용성이 뛰어난 UI 부품을 구성하는 방식을 익힘