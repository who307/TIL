# React

## TIL(Today I Learned)

* `map()` 메서드를 활용해 배열 데이터를 JSX 리스트 요소로 변환하고 고유한 `key` 속성을 할당하는 패턴 학습
* 배열 State 조작 시 불변성을 지키기 위해 전개 연산자(`...`)로 요소를 추가하고 `filter()` 메서드로 요소를 제거하는 방식 구현
* 렌더링 이후 콘솔 출력, 타이머 등록 등의 사이드 이펙트(Side Effect)를 다루는 `useEffect` Hook 기본 동작 습득
* 의존성 배열(Dependency Array) 유무에 따른 `useEffect` 실행 시점(Mount, Update) 제어 메커니즘 학습
* 컴포넌트 언마운트(Unmount) 시 타이머를 해제하는 클린업(Cleanup) 함수 구현을 통한 메모리 누수 방지



---

## 핵심 내용(코드)

### 리스트 렌더링 및 동적 항목 추가·삭제 (`map`, `spread`, `filter`)

```jsx
/**
 * 1. map()을 사용해 배열 데이터를 JSX로 변환할 때 고유한 key 속성을 지정[cite: 24]
 * 2. 추가: 기존 배열을 전개 연산자(...names)로 복사 후 새 객체 결합[cite: 25]
 * 3. 삭제: filter() 메서드를 통해 지정 ID를 제외한 새 배열 생성[cite: 25]
 */

// 객체 배열 리스트 렌더링[cite: 24]
const users = [
    { id: 1, name: '판다' },
    { id: 2, name: '코알라' }
];

function UserList() {
    return (
        <ul>
            {users.map((user) => (
                <li key={user.id}>{user.name}</li>[cite: 24]
            ))}
        </ul>
    );
}

// 동적 리스트 추가 및 삭제 구현[cite: 25]
function NameList() {
    const [inputValue, setInputValue] = useState('');[cite: 25]
    const [names, setNames] = useState([{ id: 1, text: '판다' }]);[cite: 25]
    const [nextId, setNextId] = useState(2);[cite: 25]

    const addHandler = () => {
        if (inputValue.trim() === '') return;[cite: 25]
        const newName = { id: nextId, text: inputValue };[cite: 25]
        setNames([...names, newName]); // 불변성 유지 (전개 연산자)[cite: 25]
        setNextId(nextId + 1);[cite: 25]
        setInputValue('');[cite: 25]
    };

    const removeHandler = (idToRemove) => {
        // 해당 id 항목을 제외한 새 배열 반환[cite: 25]
        const updateNames = names.filter(name => name.id !== idToRemove);[cite: 25]
        setNames(updateNames);[cite: 25]
    };

    return (
        <>
            <input value={inputValue} onChange={(e) => setInputValue(e.target.value)} />[cite: 25]
            <button onClick={addHandler}>추가</button>[cite: 25]
            <ul>
                {names.map(name => (
                    <li key={name.id} onDoubleClick={() => removeHandler(name.id)}>[cite: 25]
                        {name.text}
                        <button onClick={() => removeHandler(name.id)}>삭제</button>[cite: 25]
                    </li>
                ))}
            </ul>
        </>
    );
}

```

### `useEffect` 생명주기 및 의존성 배열 조건 처리

```jsx
/**
 * useEffect(effectCallback, [deps])
 * Side Effect 코드를 렌더링 이후 시점으로 분리[cite: 26]
 */

// 1. 의존성 배열 생략: 매 렌더링마다 실행[cite: 28]
useEffect(() => {
    console.log('매 렌더링마다 실행됩니다.');[cite: 28]
});

// 2. 빈 의존성 배열 []: 첫 렌더링(Mount) 시 1회만 실행[cite: 27, 28]
useEffect(() => {
    console.log('컴포넌트가 처음 화면에 나타남 (Mount)');[cite: 28]
}, []);

// 3. 특정 변수 지정 [count]: 해당 상태값 변경 시에만 실행[cite: 28]
useEffect(() => {
    console.log(`count값이 ${count}로 변경되었습니다.`);[cite: 28]
}, [count]);

```

### 타이머 해제 및 클린업(Cleanup) 함수

```jsx
/**
 * Cleanup 함수: useEffect 콜백 내부에서 반환되는 함수
 * 컴포넌트 언마운트(Unmount) 직전이나 이펙트 재실행 직전에 동작[cite: 28]
 */

function Timer() {
    useEffect(() => {
        console.log('타이머가 시작되었습니다(Mount)');[cite: 28]

        const timerId = setInterval(() => {
            console.log('...1초 경과...');[cite: 28]
        }, 1000);

        // 클린업 함수 반환[cite: 28]
        return () => {
            console.log('타이머를 정리합니다(Unmount)');[cite: 28]
            clearInterval(timerId); // 생성된 타이머 제거[cite: 28]
        };
    });

    return <h3>타이머 동작 중입니다.</h3>;[cite: 28]
}

```

---

## 느낀 점

* `useEffect` 사용 시 의존성 배열(Dependency Array) 유무에 따라 타이밍이 컨트롤되므로, 초기 API 호출은 빈 배열(`[]`)로, 특정 값 변경 감지는 `[state]`로 정확히 지정해야 함을 숙지함
* `setInterval` 같은 지속적인 비동기 리소스나 이벤트를 다룰 때 반환 함수(Cleanup)에서 `clearInterval`을 수행하지 않으면 메모리 누수와 예기치 않은 반복 실행이 발생한다는 점을 깨달음
* 리스트 State 조작 시 원본 배열을 다이렉트로 수정하지 않고 `filter()`나 전개 연산자(`...`)로 새로운 참조 배열을 넘겨 불변성을 지키는 원리를 확실히 익힘