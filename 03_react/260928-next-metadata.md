# Next.js & React 핵심 기술 정리 (TIL)

## 1. Next.js (App Router) 주요 기술

### 루트 및 중첩 레이아웃 (Layout Composition)

* **`RootLayout`**: 애플리케이션 최상위 단에서 `<html>` 및 `<body>` 태그를 정의하고, 전역 폰트, 메타데이터, Provider Wrapper를 주입하는 구조를 작성함.
* **`Nested Layout`**: `/dashboard`와 같은 특정 경로 하위에만 적용되는 독립 레이아웃(`DashboardLayout`)을 구성함.
* **`{children}` 주입**: 레이아웃 컴포넌트 내부에 `{children}` 프로퍼티를 배치하여 상위 틀을 재렌더링하지 않고 하위 페이지 컴포넌트만 동적으로 교체하는 원리를 체득함.

```jsx
// 루트 레이아웃 예시 (layout.js)
import { Geist, Geist_Mono } from "next/font/google";
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

---

### 내비게이션 & 동적 라우팅 (`next/navigation`, `next/link`)

* **`usePathname()`**: 클라이언트 사이드에서 현재 브라우저의 URL 경로명(Pathname)을 실시간으로 가져오는 Hook 활용법을 습득함.
* **`<Link>` 컴포넌트**: 브라우저의 전체 페이지 새로고침(Full Reload) 없이 클라이언트 단에서 빠른 SPA 방식의 페이지 이동을 구현함.
* **Active Link 구현**: `usePathname()`의 반환값과 `<Link>`의 `href` 경로를 비교(`isActive`)하여 현재 활성화된 메뉴에 동적 인라인 스타일을 적용함.
* **`useParams()`**: Dynamic Routes 경로(`/menu/[menuCode]`)의 가변 파라미터 값을 클라이언트 컴포넌트에서 추출하는 방식을 체득함.
* **`useSearchParams()`**: URL 쿼리 스트링(`?menuName=value`)을 분석 및 추출하여 검색 조건에 따른 데이터 필터링을 구현하는 기법을 파악함.

---

### 스트리밍 및 Suspense 처리 (`<Suspense>`)

* **`useSearchParams` 바운더리 격리**: App Router 환경에서 `useSearchParams()`를 호출하는 클라이언트 컴포넌트는 SSR 및 빌드 시점 오류를 방지하기 위해 반드시 `<Suspense>` 바운더리로 감싸야 함을 깨달음.
* **로딩 Fallback UI**: 데이터 파싱 및 로딩 중일 때 표시할 대체 UI(`fallback`)를 지정하여 사용자 경험(UX)을 향상시키는 구조를 습득함.

---

### 이미지 및 Google Fonts 최적화 (`next/image`, `next/font/google`)

* **`<Image/>` 컴포넌트**: 자동 이미지 리사이징, WebP/AVIF 최적 포맷 변환, 레이아웃 시프트(CLS) 예방 기능 및 `priority`를 통한 LCP 최적화를 체득함.
* **Google Fonts 최적화**: 빌드 타임에 폰트를 자체 호스팅 변환하여 FOUT/FOIT 현상을 방지하고, CSS 변수(`variable`)를 생성해 전역 스타일에 주입하는 패턴을 파악함.

---

### 서버 사이드 메타데이터 (Metadata API)

* **`export const metadata`**: `layout.js` 또는 `page.js` 파일 내에서 메타데이터 객체를 export 하여 `title`, `description` 등 SEO 메타 태그를 서버 측에서 자동 생성하는 방식을 습득함.

---

### 하이드레이션(Hydration) Mismatch 방지 패턴

* **브라우저 전용 데이터 이슈**: `localStorage`나 쿠키 등 클라이언트 전용 데이터를 초기 상태로 사용할 때 SSR 결과물과 CSR 결과물이 일치하지 않아 발생하는 Hydration Error를 방지하는 패턴을 파악함.
* **`isMounted` 패턴**: `useEffect`를 이용해 컴포넌트가 브라우저에 마운트된 이후에만 로컬 스토리지 연동 UI를 렌더링하도록 처리함.

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

---

## 2. 전역 상태 관리 (Zustand)

### Props Drilling 문제점과 Zustand 도입

* **Props Drilling 문제**: 상위 컴포넌트(`Home`)에서 최하위 컴포넌트(`GrandChild`)까지 상태를 전달하기 위해 중간 컴포넌트(`Child`)가 불필요하게 Props를 교징하는 구조적 한계를 확인함.
* **Zustand 도입**: 중앙 집중식 Store를 구축하여 중간 컴포넌트의 가공 및 전달 과정을 제거하고, 필요한 컴포넌트에서만 상태를 직접 구독하도록 구조를 개선함.

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

---

### 도메인별 Store 분리 & 불변성 관리

* **관심사 분리(Separation of Concerns)**: 단일 스토어에 모든 상태를 모아두지 않고, `UI Store`, `Cart Store`, `User Store`, `Persist Store` 등으로 역할에 따라 분리하여 모듈화함.
* **불변성(Immutability) 유지**: 배열 및 객체 상태 변경 시 Spread 연산자(`[...state.items, item]`)나 `filter` 메서드를 통해 기존 상태를 직접 변경하지 않고 새로운 객체를 생성하여 세팅함.

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

---

### Zustand Persist 미들웨어 및 비동기 액션

* **Persist Middleware**: `persist` 미들웨어를 활용해 스토어 상태 변경 시 자동으로 `localStorage`에 상태를 동기화하고 재접속 시 복원하는 기능을 구성함.
* **Async Action**: 스토어 내부 액션 함수에서 `async/await`를 직접 사용해 API 호출, 로딩 상태 변경(`loading`), 결과 데이터 업데이트(`user`)를 일관되게 처리함.

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

---

## 3. 비동기 데이터 페칭 & 서버 상태 관리

### RESTful API 연동 캡슐화 (`fetch` API)

* **API 모듈화**: `postApi.js`, `userApi.js`로 RESTful API 요청 함수를 캡슐화하여 HTTP 요청 방식(GET, POST, PATCH, DELETE) 및 JSON 데이터 직렬화(`JSON.stringify`)를 한곳에서 관리함.
* **예외 처리**: `response.ok` 검사를 수행하여 비정상 응답 시 명시적인 `Error`를 throw 하는 안전한 예외 구조를 수립함.

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

---

### TanStack Query (React Query) 기반 서버 상태 관리

* **`useQuery` Hook**: 서버 상태를 캐싱, 추적, 갱신하기 위해 `queryKey`와 `queryFn`을 정의함.
* **상태 변수 활용**: `isLoading`(초기 로딩), `isError`(에러 유무), `isFetching`(백그라운드 재요청 중) 등의 라이프사이클 상태를 활용해 로딩/에러 UI 처리 및 새로고침(`refetch`)을 손쉽게 작성함.

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

---

## 4. React 핵심 기술 & 패턴

### 서버 컴포넌트(RSC) vs 클라이언트 컴포넌트(RCC)

* **`"use client"` 지시어**: 파일 최상단에 선언하여 클라이언트 경계(Client Boundary)를 지정하는 역할을 이해함.
* **클라이언트 컴포넌트 적용 대상**: 브라우저 API, React Hook(`useState`, `useEffect`, `useQuery`, Zustand Store 등), 이벤트 리스너를 사용하는 컴포넌트에 필수 선언해야 함을 깨달음.
* **기본 서버 컴포넌트**: `"use client"` 선언이 없는 컴포넌트는 서버에서만 실행되어 클라이언트 자바스크립트 번들 용량을 줄일 수 있음을 확인함.

---

### 상태 관리 라이브러리 및 패턴 비교

1. **Local State (`useState`)**: 컴포넌트 내부 및 CRUD 페이지 내 단순 UI 로컬 데이터 관리.
2. **Global State (`Zustand`)**: 로그인 사용자 정보, UI 상태(모달/사이드바), 장바구니 등 앱 전역에서 동기적으로 참조되는 클라이언트 상태 관리.
3. **Server State (`TanStack Query`)**: 서버 비동기 데이터의 자동 캐싱, 무효화(Invalidation), 백그라운드 동기화 관리.

---

## 느낀 점

* Props Drilling 구조를 Zustand 전역 스토어로 전환함으로써 컴포넌트 결합도를 낮추고 코드의 가독성 및 유지보수성을 크게 향상시킬 수 있음을 확인했습니다.
* Client-side 저장소(`localStorage`) 연동 시 발생하는 SSR Hydration Mismatch 문제를 `isMounted` 패턴을 통해 안정적으로 제어하는 방법을 체득했습니다.
* `fetch` 기반 REST API 연동을 모듈화하고, 단순 Client State와 Async/Server State의 구분을 명확히 함으로써 프론트엔드 아키텍처 관점에서의 데이터 흐름 제어 능력을 다질 수 있었습니다.
* TanStack Query를 도입하여 로딩/에러 상태, 백그라운드 동기화(`isFetching`), 수동 갱신(`refetch`) 처리를 선언적이고 효율적으로 작성하는 실무 패턴을 파악했습니다.