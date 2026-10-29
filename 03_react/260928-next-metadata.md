# Next.js Provider · Zustand · TanStack Query 정리 (TIL)

> 레이아웃·내비게이션·Suspense·이미지·폰트·메타데이터·서버/클라이언트 컴포넌트는 [260924-next-layout.md](./260924-next-layout.md)에 정리함. 이 문서는 그 위에 새로 배운 **전역 Provider, Hydration, 상태 관리** 부분만 다룸.

## 1. 루트 레이아웃에 Provider 주입하기

* **Provider**: 앱 전체에 어떤 기능(예: TanStack Query의 캐시 보관함)을 **나눠주는 공급자 컴포넌트**. 건물 전체에 물을 공급하는 **수도 본관**과 같음. 본관에 연결된 방(컴포넌트)이라면 어디서든 수도꼭지를 틀어 쓸 수 있음.
* **왜 루트 레이아웃에?**: `RootLayout`은 모든 페이지를 감싸는 가장 바깥 틀이므로, 여기서 `{children}`을 `<Providers>`로 감싸면 **모든 페이지**가 그 기능을 쓸 수 있음.
* **왜 `Providers`를 따로 만드나**: Provider는 보통 브라우저에서 state를 들고 있어야 해서 `"use client"`가 필요함. 그런데 `RootLayout`은 서버 컴포넌트로 두는 게 좋음 → 클라이언트 부분만 `Providers.js` 파일로 떼어내고, 레이아웃은 그걸 가져다 감싸기만 함.

```jsx
// 루트 레이아웃 예시 (layout.js)
import "./globals.css";
import { Providers } from "./Providers";

export default function RootLayout({ children }) {
    return (
        <html lang="en">
            <body>
                <Providers>{children}</Providers>
            </body>
        </html>
    );
}
```

* **한 줄씩 보기**
  * `import "./globals.css"` → 사이트 전체에 적용할 기본 스타일을 불러옴.
  * `import { Providers } from "./Providers"` → 같은 폴더의 `Providers.js`에서 공급자 컴포넌트를 가져옴.
  * `<html><body>` → 모든 페이지의 가장 바깥 뼈대.
  * `<Providers>{children}</Providers>` → 지금 보여줄 페이지(`children`)를 Provider로 감쌈 → 그 안의 어떤 컴포넌트든 `useQuery` 같은 기능을 쓸 수 있게 됨.

---

## 2. Hydration Mismatch 막기

* **Hydration(하이드레이션)**: Next.js는 서버에서 HTML을 먼저 만들어 보내고(빠르게 보이게), 그다음 브라우저에서 자바스크립트가 그 HTML에 **이벤트와 state를 연결해 살아 움직이게** 함. 말린 미역(HTML)에 물을 부어 불리는 것과 같아서 "수분 공급(hydration)"이라고 부름.
* **Mismatch(불일치) 문제**: 이 연결을 할 때 React는 "서버가 만든 화면"과 "브라우저가 처음 만든 화면"이 **똑같다고 가정**함. 그런데 `localStorage`는 **브라우저에만** 있음.
  * 서버: `localStorage`를 못 읽음 → 기본 테마 `"light"`로 그림.
  * 브라우저: `localStorage`에 저장된 `"dark"`를 읽음 → `"dark"`로 그림.
  * → 두 화면이 달라서 **Hydration Error** 경고가 남.
* **`isMounted` 패턴**: "브라우저에 완전히 올라온 **뒤에만** 저장소 값을 쓰는 화면을 그린다."
  1. 처음엔 `isMounted = false` → 서버와 브라우저 모두 `로딩 중...`으로 **똑같이** 그림 → 불일치 없음.
  2. `useEffect`는 브라우저에서 화면이 그려진 **뒤에만** 실행됨 → 여기서 `isMounted = true`.
  3. 다시 그리기 → 이제 저장소 값을 쓴 진짜 화면을 그림.

```jsx
// Hydration 에러 해결 예시 (page - persist.js)
"use client";
import { usePersistStore } from "@/store/usePersistStore";
import { useEffect, useState } from "react";

export default function PersistPage() {
    const { theme, toggleTheme } = usePersistStore();
    const [isMounted, setIsMounted] = useState(false);

    useEffect(() => {
        setIsMounted(true);
    }, []);

    if (!isMounted) return <div>로딩 중...</div>;

    return (
        <div style={{ backgroundColor: theme === "dark" ? "#333" : "#fff" }}>
            <p>현재 테마: {theme}</p>
            <button onClick={toggleTheme}>테마 변경</button>
        </div>
    );
}
```

* **따라가기** (`localStorage`에 `"dark"`가 저장돼 있음)
  1. 서버 → `isMounted`는 `false` → `<div>로딩 중...</div>`
  2. 브라우저 첫 그리기 → 역시 `false` → `<div>로딩 중...</div>` → 서버와 같으므로 하이드레이션 성공.
  3. 이펙트 → `setIsMounted(true)` → 다시 그리기.
  4. `theme`은 `"dark"` → 배경 `#333`(어두운 회색), `현재 테마: dark` 표시.
  5. "테마 변경" 클릭 → `toggleTheme` → `"light"`로 바뀌고 저장소에도 저장됨 → 배경 `#fff`(흰색).

---

## 3. 전역 상태 관리: Zustand

### Props Drilling 문제와 Zustand 도입

* **Props Drilling**: `Home`(할아버지)의 데이터를 `GrandChild`(손자)가 쓰려면, 중간의 `Child`(아빠)는 그 데이터를 **쓰지도 않으면서** props로 받아서 넘겨줘야 함. 드릴로 층층이 구멍을 뚫듯 전달해야 해서 붙은 이름임. 단계가 깊어질수록 중간 컴포넌트가 지저분해짐.
* **Zustand**: 데이터를 컴포넌트 밖의 **중앙 창고(Store)** 하나에 두고, 필요한 컴포넌트가 **창고에서 직접** 꺼내 쓰게 하는 라이브러리. 반 전체가 쓰는 **교실 게시판**과 같음. 쪽지를 친구에서 친구로 돌리지 않고 게시판에 붙여두면 누구나 직접 봄.
* **`create((set) => ({ ... }))`**: 창고를 만드는 함수.
  * 객체 안에 **데이터**(`count`, `text`)와 **데이터를 바꾸는 함수(액션)**(`increase` 등)를 함께 넣음.
  * `set(...)`: 창고 데이터를 바꾸는 도구. 바뀌면 그 데이터를 쓰는 컴포넌트만 다시 그려짐.
  * `set((state) => ...)`: 지금 창고 상태(`state`)를 받아 새 값을 계산할 때.
  * `set({ text: value })`: 값을 바로 넣을 때.
* **쓰는 법**: 컴포넌트에서 `const { count, increase } = useStore();`처럼 꺼내 씀. `Child`를 거칠 필요가 없음.

```jsx
// Zustand 스토어 정의 (useStore.js)
import { create } from "zustand";

export const useStore = create((set) => ({
    count: 0,
    text: "",
    increase: () => set((state) => ({ count: state.count + 1 })),
    decrease: () => set((state) => ({ count: state.count - 1 })),
    setText: (value) => set({ text: value }),
}));
```

* **따라가기**
  1. 처음 창고 → `{ count: 0, text: "" }`
  2. 어느 컴포넌트에서 `increase()` → `state.count`(0) + 1 → `{ count: 1, text: "" }` → `count`를 쓰는 컴포넌트들이 다시 그려짐.
  3. `decrease()` → `{ count: 0, ... }`
  4. `setText("안녕")` → `{ count: 0, text: "안녕" }`. `set`은 넘긴 칸만 바꾸고 나머지 칸(`count`)은 그대로 둠.

### 도메인별 Store 분리와 불변성

* **관심사 분리**: 창고 하나에 모든 걸 넣으면 커지고 복잡해짐 → `UI Store`(모달·사이드바), `Cart Store`(장바구니), `User Store`(로그인 사용자), `Persist Store`(저장해 둘 설정)처럼 **주제별 창고**로 나눔. 냉장고·옷장·책장을 따로 두는 것과 같음.
* **불변성**: 배열을 `state.items.push(item)`처럼 직접 고치면 Zustand가 바뀐 줄 모름 → 항상 **새 배열**을 만들어 `set`에 넘김.

```jsx
// 장바구니 스토어 및 불변성 관리 (useCartStore.js)
import { create } from "zustand";

export const useCartStore = create((set) => ({
    items: [],
    addItem: (item) =>
        set((state) => ({
            items: [...state.items, item], // 불변성 유지
        })),
    removeItem: (id) =>
        set((state) => ({
            items: state.items.filter((item) => item.id !== id),
        })),
    clearCart: () => set({ items: [] }),
}));
```

* **따라가기**
  1. 처음 → `items: []`
  2. `addItem({ id: 1, name: "사과" })` → `[...[], 사과]` → `[사과]`
  3. `addItem({ id: 2, name: "우유" })` → `[...[사과], 우유]` → `[사과, 우유]`
  4. `removeItem(1)` → `filter(item.id !== 1)` → `[우유]`
  5. `clearCart()` → `[]`

### Persist 미들웨어와 비동기 액션

* **Persist 미들웨어**: 창고 내용을 **자동으로 `localStorage`에 저장**하고, 다시 접속하면 꺼내서 **복원**해 주는 추가 기능. 게임의 자동 저장과 같음. (Zustand 기본 창고는 메모리에만 있어서 새로고침하면 사라짐)
* **비동기 액션**: 창고의 액션 함수 안에서 `async/await`로 서버 요청까지 처리할 수 있음. 컴포넌트는 `fetchUser()`만 부르면 되고, 로딩 상태·결과 저장은 창고가 알아서 함.

```jsx
// 비동기 액션을 갖는 Zustand 스토어 (useUserStore.js)
import { create } from "zustand";

export const useUserStore = create((set) => ({
    user: null,
    loading: false,
    fetchUser: async () => {
        set({ loading: true });
        try {
            const res = await fetch("https://jsonplaceholder.typicode.com/users/1");
            const data = await res.json();
            set({ user: data });
        } catch (error) {
            console.error("데이터 가져오기 실패", error);
        } finally {
            set({ loading: false });
        }
    },
}));
```

* **따라가기: `fetchUser()` 호출**
  1. `set({ loading: true })` → 화면에 "불러오는 중"을 보여줄 수 있음.
  2. `await fetch(...)` → 1번 사용자 정보 요청 → `await res.json()`으로 객체 변환.
  3. `set({ user: data })` → 창고에 사용자 저장 → `user`를 쓰는 컴포넌트들이 다시 그려짐.
  4. 실패하면 `catch`에서 콘솔에 오류 출력.
  5. `finally` → 성공이든 실패든 `set({ loading: false })`

---

## 4. 비동기 데이터 페칭과 서버 상태 관리

### `fetch` API 요청을 한곳에 모으기

* **API 모듈화**: 게시글 관련 요청(`목록`, `등록`, `수정`, `삭제`)을 `postApi.js` 한 파일에 모아둠. 주소가 바뀌거나 에러 처리를 고칠 때 **한 곳만** 고치면 됨. 전화번호부를 한 권에 모아두는 것과 같음.
* **HTTP 메서드**: 서버에 "무엇을 할지" 알려주는 동사.

| 메서드 | 뜻 | 여기서 |
| :--- | :--- | :--- |
| GET | 가져와 | 게시글 목록 |
| POST | 새로 만들어 | 게시글 등록 |
| PATCH | 일부만 고쳐 | 제목 수정 |
| DELETE | 지워 | 게시글 삭제 |

* **`JSON.stringify(객체)`**: 자바스크립트 객체를 서버에 보낼 수 있는 **JSON 글자**로 바꿈. `headers`의 `"Content-Type": "application/json"`은 "보내는 내용이 JSON이에요"라고 알려주는 표시.

```javascript
// postApi.js 예시
const BASE_URL = "http://localhost:4000/posts";

export const postApi = {
    getPosts: async () => {
        const response = await fetch(BASE_URL);
        if (!response.ok) throw new Error("게시글 목록을 가져오지 못했습니다");
        return response.json();
    },
    createPost: async (newPost) => {
        const response = await fetch(BASE_URL, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(newPost),
        });
        if (!response.ok) throw new Error("게시글 등록 실패");
        return response.json();
    },
    updatePost: async (id, updatedTitle) => {
        const response = await fetch(`${BASE_URL}/${id}`, {
            method: "PATCH",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ title: updatedTitle }),
        });
        if (!response.ok) throw new Error("게시글 수정 실패");
        return response.json();
    },
    delete: async (id) => {
        const response = await fetch(`${BASE_URL}/${id}`, { method: "DELETE" });
        if (!response.ok) throw new Error("게시글 삭제 실패");
        return true;
    },
};
```

* **한 줄씩 보기**
  * `getPosts` → `GET /posts` → 성공하면 게시글 배열을 돌려줌. 실패(`!response.ok`)면 에러를 던져서 부른 쪽이 알 수 있게 함.
  * `createPost({ title: "첫 글" })` → `POST /posts`, 본문 `{"title":"첫 글"}` → 서버가 만든 게시글(번호 포함)을 돌려줌.
  * `updatePost(3, "새 제목")` → `PATCH /posts/3`, 본문 `{"title":"새 제목"}` → 3번 글의 제목만 바뀜.
  * `delete(3)` → `DELETE /posts/3` → 성공하면 `true`.
  * 모든 함수에 `response.ok` 검사를 넣은 이유: `fetch`는 404·500이어도 실패로 치지 않기 때문임.

### TanStack Query로 서버 데이터 관리하기

* **서버 상태(Server State)**: 내 앱이 아니라 **서버에 진짜 원본이 있는 데이터**(사용자 목록, 게시글). 언제든 다른 사람이 바꿀 수 있어서, 가져오기·로딩·에러·다시 가져오기·저장(캐싱)까지 신경 쓸 게 많음.
* **TanStack Query(React Query)**: 이 일을 대신해 주는 라이브러리. **도서관 사서**처럼 한 번 빌려온 책(데이터)을 보관해 두고(캐시), 같은 책을 또 찾으면 보관본을 먼저 주고, 필요하면 최신본을 다시 받아옴.
* **`useQuery({ queryKey, queryFn })`**
  * `queryKey: ["users"]`: 데이터의 **이름표**. 같은 이름표면 같은 캐시를 같이 씀.
  * `queryFn: fetchUsers`: 데이터를 실제로 가져오는 함수.
* **돌려주는 상태값**

| 값 | 뜻 |
| :--- | :--- |
| `data` | 가져온 데이터 |
| `isLoading` | **처음** 가져오는 중 (보여줄 데이터가 아직 없음) |
| `isError` / `error` | 실패했는지 / 무슨 에러인지 |
| `isFetching` | 가져오는 중 (보관본을 보여주면서 뒤에서 최신본을 받는 중일 때도 `true`) |
| `refetch` | 지금 다시 가져오라는 함수 |

```jsx
// React Query 활용 예시 (page-query.js)
'use client'

import { fetchUsers } from "@/api/userApi";
import { useQuery } from "@tanstack/react-query";

export default function QueryPage() {
    const { data, isLoading, isError, error, refetch, isFetching } = useQuery({
        queryKey: ["users"],
        queryFn: fetchUsers,
    });

    if (isLoading) return <h1>데이터를 불러오는중...</h1>;
    if (isError) return <p>오류 발생: {error.message}</p>;

    return (
        <>
            <h1>사용자 목록</h1>
            {isFetching && <p>최신 데이터 확인중...</p>}
            {data.map((user) => (
                <div key={user.id}>
                    <strong>{user.name}</strong>
                    <p>{user.email}</p>
                </div>
            ))}
            <button onClick={() => refetch()}>새로고침</button>
        </>
    );
}
```

* **따라가기**
  1. 처음 접속 → 캐시에 `["users"]`가 없음 → `fetchUsers` 실행, `isLoading`이 `true` → `데이터를 불러오는중...`
  2. 데이터 도착 → 캐시에 저장, `isLoading`이 `false` → `사용자 목록`과 이름·이메일 카드들이 보임.
  3. "새로고침" 클릭 → `refetch()` → 기존 목록은 **그대로 보여주면서** 뒤에서 다시 요청 → `isFetching`이 `true` → `최신 데이터 확인중...` 문구가 잠깐 뜸 → 새 데이터로 바뀜.
  4. 다른 페이지에 갔다가 돌아오면 → 캐시에 있는 목록을 **바로** 보여줌 (다시 처음부터 로딩하지 않음).
  5. 요청이 실패하면 → `isError`가 `true` → `오류 발생: (에러 메시지)`
  * 이 페이지가 동작하려면 1번에서 만든 `<Providers>` 안에 TanStack Query의 공급자(`QueryClientProvider`)가 들어 있어야 함.

---

## 5. 상태 관리 방법 비교

| 종류 | 도구 | 무엇을 담나 | 비유 |
| :--- | :--- | :--- | :--- |
| **Local State** | `useState` | 한 컴포넌트 안에서만 쓰는 데이터 (입력칸 값, 열림/닫힘) | 내 책상 서랍 |
| **Global State** | `Zustand` | 여러 화면이 함께 쓰는 **내 앱의** 데이터 (로그인 정보, 모달 상태, 장바구니) | 교실 게시판 |
| **Server State** | `TanStack Query` | **서버에 원본이 있는** 데이터 (게시글, 사용자 목록) — 캐싱·자동 갱신 | 도서관 사서 |

---

## 느낀 점

* Props Drilling 구조를 Zustand 전역 스토어로 전환함으로써 컴포넌트 결합도를 낮추고 코드의 가독성 및 유지보수성을 크게 향상시킬 수 있음을 확인함
* Client-side 저장소(`localStorage`) 연동 시 발생하는 SSR Hydration Mismatch 문제를 `isMounted` 패턴을 통해 안정적으로 제어하는 방법을 체득함
* `fetch` 기반 REST API 연동을 모듈화하고, 단순 Client State와 Async/Server State의 구분을 명확히 함으로써 프론트엔드 아키텍처 관점에서의 데이터 흐름 제어 능력을 다질 수 있었음
* TanStack Query를 도입하여 로딩/에러 상태, 백그라운드 동기화(`isFetching`), 수동 갱신(`refetch`) 처리를 선언적이고 효율적으로 작성하는 실무 패턴을 파악함
