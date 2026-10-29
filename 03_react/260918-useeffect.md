# React 리스트 렌더링 · useEffect 정리 (TIL)

## TIL(Today I Learned)

* `map()` 메서드를 활용해 배열 데이터를 JSX 리스트 요소로 변환하고 고유한 `key` 속성을 할당하는 패턴 학습
* 배열 State 조작 시 불변성을 지키기 위해 전개 연산자(`...`)로 요소를 추가하고 `filter()` 메서드로 요소를 제거하는 방식 구현
* 렌더링 이후 콘솔 출력, 타이머 등록 등의 사이드 이펙트(Side Effect)를 다루는 `useEffect` Hook 기본 동작 습득
* 의존성 배열(Dependency Array) 유무에 따른 `useEffect` 실행 시점(Mount, Update) 제어 메커니즘 학습
* 컴포넌트 언마운트(Unmount) 시 타이머를 해제하는 클린업(Cleanup) 함수 구현을 통한 메모리 누수 방지

---

## 1. 리스트 렌더링과 항목 추가·삭제

* **리스트 렌더링**: 배열의 칸마다 `<li>` 같은 JSX를 하나씩 만들어 목록으로 보여주는 것. `map()`으로 "데이터 배열 → JSX 배열"로 바꿈.
* **`key`가 왜 필요한가**: React는 목록이 바뀌었을 때 "어떤 줄이 추가됐고, 어떤 줄이 지워졌는지"를 알아야 그 줄만 고칠 수 있음. `key`는 각 줄의 **출석 번호(이름표)** 임. 겹치지 않는 고유한 값(보통 `id`)을 씀.
* **불변성**: state 배열을 직접 바꾸면(`names.push(...)`) React는 같은 배열이라 생각해서 다시 그리지 않음. 그래서 항상 **새 배열**을 만들어 Setter에 넘김.
  * 추가: `[...기존배열, 새항목]` → 기존 것을 펼쳐 복사하고 뒤에 하나 더 붙인 새 배열.
  * 삭제: `기존배열.filter(조건)` → 조건에 맞는 것만 남긴 새 배열.

```jsx
/**
 * 1. map()을 사용해 배열 데이터를 JSX로 변환할 때 고유한 key 속성을 지정
 * 2. 추가: 기존 배열을 전개 연산자(...names)로 복사 후 새 객체 결합
 * 3. 삭제: filter() 메서드를 통해 지정 ID를 제외한 새 배열 생성
 */

// 객체 배열 리스트 렌더링
const users = [
    { id: 1, name: '판다' },
    { id: 2, name: '코알라' }
];

function UserList() {
    return (
        <ul>
            {users.map((user) => (
                <li key={user.id}>{user.name}</li>
            ))}
        </ul>
    );
}

// 동적 리스트 추가 및 삭제 구현
function NameList() {
    const [inputValue, setInputValue] = useState('');
    const [names, setNames] = useState([{ id: 1, text: '판다' }]);
    const [nextId, setNextId] = useState(2);

    const addHandler = () => {
        if (inputValue.trim() === '') return;
        const newName = { id: nextId, text: inputValue };
        setNames([...names, newName]); // 불변성 유지 (전개 연산자)
        setNextId(nextId + 1);
        setInputValue('');
    };

    const removeHandler = (idToRemove) => {
        // 해당 id 항목을 제외한 새 배열 반환
        const updateNames = names.filter(name => name.id !== idToRemove);
        setNames(updateNames);
    };

    return (
        <>
            <input value={inputValue} onChange={(e) => setInputValue(e.target.value)} />
            <button onClick={addHandler}>추가</button>
            <ul>
                {names.map(name => (
                    <li key={name.id} onDoubleClick={() => removeHandler(name.id)}>
                        {name.text}
                        <button onClick={() => removeHandler(name.id)}>삭제</button>
                    </li>
                ))}
            </ul>
        </>
    );
}

```

* **`UserList`**: `users` 두 칸 → `map` → `[<li key=1>판다</li>, <li key=2>코알라</li>]` → 목록에 "판다", "코알라"가 보임.
* **`NameList` 따라가기**
  1. 처음 → `names = [{ id: 1, text: '판다' }]`, `nextId = 2` → 목록에 "판다" 한 줄.
  2. 입력칸에 "코알라" 입력 → 글자마다 `onChange` → `inputValue`가 "코알라"가 됨.
  3. "추가" 클릭 → `addHandler`
     * 빈 글자인지 검사 → 아님.
     * `newName = { id: 2, text: '코알라' }`
     * `setNames([...names, newName])` → `[{ id: 1, 판다 }, { id: 2, 코알라 }]` (새 배열)
     * `nextId` → 3, 입력칸 비움.
  4. 목록이 "판다", "코알라" 두 줄이 됨.
  5. "판다" 줄의 "삭제" 클릭(또는 더블클릭) → `removeHandler(1)` → `filter(name.id !== 1)` → `[{ id: 2, 코알라 }]` → "코알라"만 남음.
  * `onClick={() => removeHandler(name.id)}`처럼 **화살표 함수로 감싸는** 이유: `onClick={removeHandler(name.id)}`라고 쓰면 화면을 그리는 순간 바로 삭제가 실행돼 버림. 감싸야 "클릭했을 때" 실행됨.

---

## 2. `useEffect`와 의존성 배열

* **사이드 이펙트(Side Effect)**: 화면을 그리는 일 **말고** 하는 부수적인 일. 콘솔 출력, 타이머 시작, 서버에서 데이터 가져오기 등.
* **`useEffect(함수, [의존성])`**: "화면을 다 그린 **다음에** 이 함수를 실행해 줘"라고 맡겨두는 Hook. 그리는 도중에 무거운 일을 하면 화면이 늦게 뜨기 때문에 뒤로 미룸.
* **생명주기 용어**
  * **Mount(마운트)**: 컴포넌트가 화면에 **처음 나타남**.
  * **Update(업데이트)**: state나 props가 바뀌어 **다시 그려짐**.
  * **Unmount(언마운트)**: 컴포넌트가 화면에서 **사라짐**.
* **의존성 배열에 따른 실행 시점**

| 두 번째 인수 | 언제 실행되나 | 비유 |
| :--- | :--- | :--- |
| 없음 | 그릴 때마다 **매번** | 매일 아침 하는 양치 |
| `[]` (빈 배열) | **처음 한 번**(Mount)만 | 입학식 |
| `[count]` | 처음 한 번 + `count`가 **바뀔 때마다** | 키가 자랄 때마다 바지 사기 |

```jsx
/**
 * useEffect(effectCallback, [deps])
 * Side Effect 코드를 렌더링 이후 시점으로 분리
 */

// 1. 의존성 배열 생략: 매 렌더링마다 실행
useEffect(() => {
    console.log('매 렌더링마다 실행됩니다.');
});

// 2. 빈 의존성 배열 []: 첫 렌더링(Mount) 시 1회만 실행
useEffect(() => {
    console.log('컴포넌트가 처음 화면에 나타남 (Mount)');
}, []);

// 3. 특정 변수 지정 [count]: 해당 상태값 변경 시에만 실행
useEffect(() => {
    console.log(`count값이 ${count}로 변경되었습니다.`);
}, [count]);

```

* **따라가기** (`count`라는 state가 있는 컴포넌트라고 가정)
  1. 처음 화면에 나타남 → 세 개 모두 실행 → `매 렌더링마다 실행됩니다.` / `컴포넌트가 처음 화면에 나타남 (Mount)` / `count값이 0로 변경되었습니다.`
  2. `count`가 1로 바뀌어 다시 그림 → 1번(매번)과 3번(`count` 바뀜)만 실행. 2번은 이미 한 번 실행했으므로 건너뜀.
  3. `count` 말고 **다른 state**가 바뀌어 다시 그림 → 1번만 실행. `count`는 그대로라 3번은 건너뜀.

---

## 3. 클린업(Cleanup) 함수와 타이머 정리

* **클린업 함수**: `useEffect` 안에서 **돌려주는(return) 함수**. React가 "뒷정리할 때" 불러줌. 놀고 난 뒤 장난감을 치우는 것과 같음.
* **언제 불리나**
  1. 컴포넌트가 화면에서 **사라지기 직전**(Unmount).
  2. 이펙트를 **다시 실행하기 직전** (이전 것을 먼저 정리).
* **왜 필요한가**: `setInterval`로 만든 타이머는 컴포넌트가 사라져도 **혼자 계속 돌아감**. 정리하지 않으면 보이지도 않는 타이머가 계속 쌓여 메모리를 차지하고(메모리 누수) 엉뚱한 동작을 함.

```jsx
/**
 * Cleanup 함수: useEffect 콜백 내부에서 반환되는 함수
 * 컴포넌트 언마운트(Unmount) 직전이나 이펙트 재실행 직전에 동작
 */

function Timer() {
    useEffect(() => {
        console.log('타이머가 시작되었습니다(Mount)');

        const timerId = setInterval(() => {
            console.log('...1초 경과...');
        }, 1000);

        // 클린업 함수 반환
        return () => {
            console.log('타이머를 정리합니다(Unmount)');
            clearInterval(timerId); // 생성된 타이머 제거
        };
    });

    return <h3>타이머 동작 중입니다.</h3>;
}

```

* **따라가기**
  1. `Timer`가 화면에 나타남 → 그린 뒤 이펙트 실행 → `타이머가 시작되었습니다(Mount)`
  2. `setInterval` → 1초마다 `...1초 경과...` 출력. 타이머 번호를 `timerId`에 저장해 둠.
  3. 부모가 `Timer`를 화면에서 치움(Unmount) → 클린업 실행 → `타이머를 정리합니다(Unmount)` → `clearInterval(timerId)`로 타이머 멈춤.
  4. 더 이상 `...1초 경과...`가 찍히지 않음.
  * 이 예제는 의존성 배열이 없어서 **다시 그려질 때마다** "정리 → 새 타이머 시작"을 반복함. 지금은 state가 없어 다시 그려질 일이 거의 없지만, 타이머를 한 번만 걸고 싶다면 `[]`를 넣는 편이 안전함.

---

## 느낀 점

* `useEffect` 사용 시 의존성 배열(Dependency Array) 유무에 따라 타이밍이 컨트롤되므로, 초기 API 호출은 빈 배열(`[]`)로, 특정 값 변경 감지는 `[state]`로 정확히 지정해야 함을 숙지함
* `setInterval` 같은 지속적인 비동기 리소스나 이벤트를 다룰 때 반환 함수(Cleanup)에서 `clearInterval`을 수행하지 않으면 메모리 누수와 예기치 않은 반복 실행이 발생한다는 점을 깨달음
* 리스트 State 조작 시 원본 배열을 다이렉트로 수정하지 않고 `filter()`나 전개 연산자(`...`)로 새로운 참조 배열을 넘겨 불변성을 지키는 원리를 확실히 익힘
