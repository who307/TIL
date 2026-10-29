# Next.js App Router · 레이아웃 정리 (TIL)

## 0. Next.js와 App Router가 뭔가

* **한 줄 비유**: React가 **레고 블록**이라면, Next.js는 블록으로 집을 짓기 쉽게 **설계도·배관·전기까지 깔아둔 집터**임.
* **App Router**: `app` 폴더 안의 **폴더 구조가 곧 주소(URL)** 가 되는 방식.
  * `app/page.js` → `/`
  * `app/about/page.js` → `/about`
  * `app/dashboard/page.js` → `/dashboard`
* **특별한 파일 이름**
  * `page.js`: 그 주소에서 보여줄 **페이지 내용**.
  * `layout.js`: 그 주소와 하위 주소들이 **함께 쓰는 틀**(머리말, 메뉴 등).

---

## 1. 루트 레이아웃과 중첩 레이아웃

* **한 줄 비유**: 레이아웃은 **액자**, 페이지는 **액자 속 그림**임. 그림(페이지)만 바꿔 끼우고 액자(레이아웃)는 그대로 둠.
* **`RootLayout`** (`app/layout.js`): 모든 페이지를 감싸는 **가장 바깥 액자**. `<html>`, `<body>` 태그와 전역 글꼴·스타일을 여기서 정함.
* **중첩 레이아웃(Nested Layout)** (`app/dashboard/layout.js`): `/dashboard`와 그 아래 주소에만 씌우는 **안쪽 액자**. 바깥 액자 안에 한 번 더 끼워짐.
* **`{children}`**: 액자 안에서 **그림이 들어갈 자리**. Next.js가 지금 주소에 맞는 `page.js`(또는 더 안쪽 레이아웃)를 여기에 넣어줌.
* **장점**: 다른 페이지로 이동해도 레이아웃은 **다시 그려지지 않고** `children` 자리만 바뀜 → 메뉴가 깜빡이지 않고, 레이아웃 안의 state도 유지됨.

```jsx
// 중첩 레이아웃 예시 (layout_2.js)
export default function DashboardLayout({ children }) {
    return (
        <>
            <h2>대시보드 메뉴</h2>
            {children} {/* 하위 page.js가 들어오는 영역 */}
        </>
    );
}

```

* **따라가기: `/dashboard`에 접속**
  1. `RootLayout`이 `<html><body>`를 그리고, 자기 `children` 자리에 `DashboardLayout`을 넣음.
  2. `DashboardLayout`이 `<h2>대시보드 메뉴</h2>`를 그리고, 자기 `children` 자리에 `app/dashboard/page.js`의 내용을 넣음.
  3. 결과: `RootLayout > DashboardLayout > page` 순서로 겹쳐진 화면.
  * `/dashboard/settings`로 이동하면 → `대시보드 메뉴`는 그대로 두고 아래 `page` 부분만 settings 페이지로 바뀜.

---

## 2. 내비게이션과 동적 라우팅

| 도구 | 하는 일 | 비유 |
| :--- | :--- | :--- |
| `<Link href="...">` | 페이지 이동. 전체 새로고침 없이 바뀐 부분만 갈아 끼움 | 책장을 넘기는 것 (책을 새로 사지 않음) |
| `usePathname()` | 지금 주소의 경로(`/about`)를 알려줌 | "지금 몇 쪽이야?" |
| `useParams()` | 동적 경로 `[menuCode]` 자리에 실제로 들어온 값을 알려줌 | 사물함 번호표 읽기 |
| `useSearchParams()` | 주소 뒤 `?menuName=김밥` 부분(쿼리 스트링)을 읽어줌 | 주문서에 적힌 옵션 읽기 |

* **SPA(Single Page Application)**: 페이지를 통째로 새로 받지 않고, 한 페이지 안에서 필요한 부분만 바꿔 보여주는 방식. `<a>` 태그는 새로고침이 일어나지만 `<Link>`는 일어나지 않음.
* **`'use client'`**: 이 파일의 컴포넌트는 **브라우저에서도 실행**된다는 표시. `usePathname`, `useState` 같은 Hook은 브라우저에서만 동작하므로 반드시 필요함.

```jsx
// Navbar.jsx
'use client';
import Link from "next/link";
import { usePathname } from "next/navigation";

export default function Navbar() {
    const pathname = usePathname(); // 현재 URL 경로 추적
    const isActive = (path) => pathname === path;

    return (
        <Link href="/about" style={isActive("/about") ? { color: 'red' } : undefined}>
            소개
        </Link>
    );
}

```

* **따라가기 (Active Link: 지금 있는 메뉴만 색칠하기)**
  1. `/about`에 있을 때 → `pathname`은 `"/about"` → `isActive("/about")` → `true` → 스타일 `{ color: 'red' }` → "소개" 링크가 빨간색.
  2. `/`에 있을 때 → `pathname`은 `"/"` → `false` → 스타일 `undefined`(아무것도 안 줌) → 기본 색.
  3. "소개"를 클릭하면 → 새로고침 없이 `/about` 페이지로 바뀌고 → `pathname`이 바뀌어 Navbar가 다시 그려지면서 빨간색이 됨.

* **동적 라우트(Dynamic Route)**: 폴더 이름을 `[menuCode]`처럼 **대괄호**로 만들면, 그 자리에 어떤 값이 와도 같은 `page.js`가 보여짐.
  * `/menu/1`, `/menu/2`, `/menu/35` → 모두 `app1/menu/[menuCode]/page.js` 하나가 처리.

```jsx
// Dynamic Route 파라미터 추출 예시 (app1/menu/[menuCode]/page.js)
'use client';

import { getMenubyMenuCode } from "@/lib/MenuAPI";
import { useParams } from "next/navigation";
import { useEffect, useState } from "react";

export default function MenuDetail() {
    const { menuCode } = useParams(); // Dynamic Route 파라미터 추출
    const [menu, setMenu] = useState();

    useEffect(() => {
        setMenu(getMenubyMenuCode(menuCode));
    }, [menuCode]);

    return (
        menu && (
            <>
                <h1>{menu.menuName} 상세 페이지!</h1>
                <h3>메뉴 가격: {menu.menuPrice}</h3>
                <h3>메뉴 종류: {menu.categoryName}</h3>
                <h3>메뉴 설명: {menu.detail.description}</h3>
                <img src={menu.detail.image} style={{maxWidth: 500}} alt={menu.menuName}/>
            </>
        )
    );
}

```

* **따라가기: `/menu/3` 접속**
  1. `useParams()` → `{ menuCode: "3" }` → 구조 분해로 `menuCode = "3"` (주소에서 온 값이라 **문자열**임).
  2. 처음 그리기 → `menu`는 `undefined` → `menu && (...)`에서 `undefined`가 되어 **아무것도 안 그림**. (데이터가 없는데 `menu.menuName`을 읽으면 에러가 나므로 막아둔 것)
  3. 그린 뒤 이펙트 → `getMenubyMenuCode("3")`로 3번 메뉴 데이터를 찾아 `setMenu`.
  4. 다시 그리기 → `menu`가 있으므로 이름·가격·종류·설명·사진이 보임.
  5. `/menu/5`로 이동하면 → `menuCode`가 바뀜 → 의존성 배열 `[menuCode]` 때문에 이펙트가 다시 실행되어 5번 메뉴로 바뀜.
  * `@/lib/MenuAPI`의 `@`는 프로젝트 기준 경로를 줄여 쓰는 별명임. `../../../lib`처럼 점점이 길게 쓰지 않아도 됨.

---

## 3. Suspense로 감싸기

* **`<Suspense fallback={대체화면}>`**: 안쪽 컴포넌트가 **아직 준비가 안 됐을 때** 대신 보여줄 화면(`fallback`)을 정해두는 상자. 식당에서 음식이 나오기 전에 "조리 중" 팻말을 세워두는 것과 같음.
* **왜 `useSearchParams`에 필요한가**: Next.js는 페이지를 **미리 서버(빌드 시점)에서 만들어 두려고** 함. 그런데 `?menuName=...` 같은 쿼리 스트링은 **사용자가 실제로 접속해야** 알 수 있음. 그래서 그 부분을 `Suspense`로 감싸서 "여기는 나중에 브라우저에서 채울게"라고 표시해야 빌드 오류가 나지 않음.
* **패턴**: 실제 내용 컴포넌트(`MenuSearchResultContent`)와 그걸 감싸는 껍데기(`MenuSearchResult`)를 **둘로 나눔**. 페이지로 내보내는 건 껍데기 쪽임.

```jsx
// Query Params 및 Suspense 활용 예시 (app1/menu/search/page.js)
'use client';

import MenuItem from "@/item/MenuItem";
import { searchMenu } from "@/lib/MenuAPI";
import { useSearchParams } from "next/navigation";
import { Suspense, useEffect, useState } from "react";

function MenuSearchResultContent() {
    const [menuList, setMenuList] = useState([]);
    const searchParam = useSearchParams(); // 쿼리 스트링 추출
    const menuName = searchParam.get("menuName");

    useEffect(() => {
        setMenuList(searchMenu(menuName));
    }, [menuName]);

    return (
        <>
            <h1>검색 결과!!!</h1>
            <div>
                {menuList.map(menu => <MenuItem key={menu.menuCode} menu={menu} />)}
            </div>
        </>
    );
}

export default function MenuSearchResult() {
    return (
        <Suspense fallback={<h1>검색 조건을 확인하는 중입니다.</h1>}>
            <MenuSearchResultContent />
        </Suspense>
    );
}

```

* **따라가기: `/menu/search?menuName=김밥` 접속**
  1. 아직 쿼리 스트링을 읽기 전 → `Suspense`가 `검색 조건을 확인하는 중입니다.`를 보여줌.
  2. 브라우저에서 `MenuSearchResultContent` 실행 → `searchParam.get("menuName")` → `"김밥"`
  3. 이펙트 → `searchMenu("김밥")` → 이름에 김밥이 들어간 메뉴 배열 → `setMenuList`
  4. 다시 그리기 → `검색 결과!!!` 아래에 `MenuItem` 카드들이 나열됨.

---

## 4. 이미지 최적화 (`next/image`)

* **`<Image />`**: `<img>`를 업그레이드한 컴포넌트. 자동으로 해주는 일이 많음.
  1. 화면 크기에 맞게 **이미지 크기를 줄여서** 보냄 (휴대폰에 큰 사진을 보내지 않음).
  2. 브라우저가 지원하면 더 가벼운 형식(WebP/AVIF)으로 **바꿔서** 보냄.
  3. `width`, `height`로 자리를 **미리 잡아둠** → 사진이 늦게 떠도 글자가 갑자기 밀려나지 않음(CLS, 레이아웃 흔들림 방지).
* **`priority`**: 화면에 가장 먼저 크게 보이는 중요한 이미지(LCP, 가장 큰 요소)를 **미리 불러오라는** 표시. 로고나 첫 화면 대표 사진에 붙임.
  * ⚠️ **Next.js 16부터 `priority`는 deprecated(앞으로 없어질 예정)** 되었고, 같은 역할을 하는 `preload` 속성을 쓰도록 바뀜.

```jsx
// page_5.js
import Image from "next/image";

<Image
  src="/next.svg"
  alt="Next.js logo"
  width={100}
  height={20}
  priority // LCP 최적화 우선순위 설정
/>

```

* **한 줄씩 보기**
  * `src="/next.svg"` → `public` 폴더의 `next.svg` 파일.
  * `alt` → 이미지가 안 뜨거나 화면 낭독기가 읽을 때 쓰는 설명.
  * `width={100}`, `height={20}` → 100×20 크기의 자리를 미리 확보.
  * `priority` → 페이지가 열릴 때 이 이미지를 먼저 불러옴. (Next.js 16 이후라면 `preload`로 바꿔 씀)

---

## 5. Google Fonts 최적화 (`next/font/google`)

* **문제**: 구글 폰트를 인터넷에서 그때그때 받아오면, 받기 전엔 기본 글꼴로 보였다가 갑자기 바뀌거나(FOUT) 글자가 잠깐 안 보이는(FOIT) **깜빡임**이 생김.
* **해결**: `next/font/google`은 **빌드할 때** 폰트 파일을 미리 내려받아 내 사이트 파일에 함께 넣어둠(자체 호스팅). 외부 요청이 없어서 깜빡임이 줄어듦.
* **CSS 변수 연결**: `variable: "--font-geist-sans"` 옵션을 주면 `--font-geist-sans`라는 CSS 변수가 생김 → `<html>`의 class에 넣어두면 → 어느 CSS 파일에서든 `font-family: var(--font-geist-sans)`로 쓸 수 있음.

---

## 6. 메타데이터 (Metadata API)

* **메타데이터**: 페이지에 대한 **이름표 정보**. 브라우저 탭 제목(`title`), 검색 결과에 보이는 설명(`description`) 등.
* **`export const metadata = { title: "...", description: "..." }`**: `layout.js`나 `page.js`에서 이 객체를 내보내기만 하면, Next.js가 서버에서 `<head>` 안에 `<title>`, `<meta>` 태그를 자동으로 만들어 줌. 검색 엔진(SEO)이 페이지 내용을 잘 알아보게 됨.

---

## 7. CSS Modules

* **문제**: 일반 CSS의 `.title`은 사이트 **전체**에 적용돼서, 다른 파일의 `.title`과 이름이 겹치면 스타일이 섞임.
* **`이름.module.css`**: 파일 이름에 `.module`을 붙이면, Next.js가 클래스 이름을 `title_abc123`처럼 **그 파일 전용 이름**으로 바꿔줌 → 겹칠 일이 없음. 같은 반에 "민준"이 둘이면 "1번 민준", "2번 민준"으로 부르는 것과 같음.
* **다크 모드·반응형**: CSS 변수(`--background` 등)에 색을 담아두고, `@media (prefers-color-scheme: dark)`(사용자 컴퓨터가 다크 모드일 때) 안에서 변수 값만 바꾸면 사이트 전체 색이 한 번에 바뀜. `color-mix()`로 두 색을 섞어 중간색을 만들 수도 있음.

---

## 8. 서버 컴포넌트 vs 클라이언트 컴포넌트

* **서버 컴포넌트(RSC, 기본값)**: **서버에서만** 실행되어 완성된 HTML만 브라우저로 보냄. 컴포넌트 코드(자바스크립트)는 브라우저에 보내지 않음 → 받는 파일이 작아짐(번들 용량 감소).
  * 예: `Dashboard`, `RootLayout`, `Header`처럼 움직이지 않는 틀.
* **클라이언트 컴포넌트(RCC)**: 파일 맨 위에 **`"use client"`** 를 쓰면 브라우저에서도 실행됨. 이 선 아래로는 "브라우저 구역"이라는 경계(Client Boundary)가 생김.
  * 꼭 필요한 경우: `useState`·`useEffect`·`useParams`·`useSearchParams` 같은 Hook, 클릭 같은 이벤트, `window`·`localStorage` 같은 브라우저 기능을 쓸 때.
* **설계 요령**: 기본은 서버 컴포넌트로 두고, **움직여야 하는 작은 부분만** 클라이언트 컴포넌트로 떼어냄. 예: 레이아웃은 서버, 그 안의 Navbar(현재 메뉴 색칠)만 클라이언트.

---

## 9. React Hook과 컴포넌트 합성 정리

* **`useState`**: 메뉴 하나(객체), 메뉴 목록(배열) 같은 컴포넌트 안의 데이터를 기억함.
* **`useEffect` + 의존성 배열**: `menuCode`, `menuName`처럼 **주소에서 온 값이 바뀔 때마다** 데이터를 다시 가져오게 함.
* **`usePathname` / `useParams` / `useSearchParams`**: 주소 정보를 읽는 Next.js 전용 클라이언트 Hook.
* **레이아웃 패턴**: `<Header/>`, `<Navbar/>` 같은 공통 틀은 레이아웃에 두고, 페이지마다 다른 내용은 `{children}`으로 받아 끼움.
* **Suspense 래퍼 패턴**: `useSearchParams`를 쓰는 내용 컴포넌트와 그걸 감싸는 `Suspense` 컴포넌트를 나눠서 설계함.

---

## 느낀 점

* Next.js App Router의 핵심은 **서버 컴포넌트 중심 설계**와 **클라이언트 경계의 명확한 분리**에 있음을 확인함
* 상단 메뉴 및 레이아웃 틀은 서버 컴포넌트나 공통 Layout으로 유지하고, URL 상태에 따라 스타일 변경이 필요한 내비게이션 요소만 `"use client"`와 `usePathname`을 결합하여 최소한의 범위만 클라이언트 사이드로 동작시키는 기법의 효율성을 체득함
* Dynamic Route의 `useParams`와 Query String의 `useSearchParams`를 결합하여 RESTful한 라우팅을 손쉽게 구축할 수 있음을 깨달았으며, 특히 `useSearchParams` 적용 시 클라이언트 바운더리 오류 방지를 위한 `<Suspense>` 래퍼 컴포넌트 분리 패턴의 중요성을 파악함
* 단순 `<img>` 및 표준 웹 폰트 대신 `<Image/>` 컴포넌트와 `next/font/google`을 활용하여 CLS 차단 및 폰트 로딩 성능까지 고려하는 최신 웹 프론트엔드 최적화 기법을 파악함
