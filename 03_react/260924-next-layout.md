# Next.js & React 핵심 기술 정리 (TIL)

## 1. Next.js (App Router) 주요 기술

### 루트 및 중첩 레이아웃 (Layout Composition)

* **`RootLayout`**: 애플리케이션 최상위 단에서 `<html>` 및 `<body>` 태그를 정의하고, 전역 폰트 및 스타일을 주입하는 구조를 작성함.
* **`Nested Layout`**: `/dashboard`와 같은 특정 경로 하위에만 적용되는 독립 레이아웃(`DashboardLayout`)을 구성함.
* **`{children}` 주입**: 레이아웃 컴포넌트 내부에 `{children}` 프로퍼티를 배치하여 상위 틀을 재렌더링하지 않고 하위 페이지 컴포넌트만 동적으로 교체하는 원리를 체득함.

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

---

### 내비게이션 & 동적 라우팅 (`next/navigation`, `next/link`)

* **`usePathname()`**: 클라이언트 사이드에서 현재 브라우저의 URL 경로명(Pathname)을 실시간으로 가져오는 Hook 활용법을 습득함.
* **`<Link>` 컴포넌트**: 브라우저의 전체 페이지 새로고침(Full Reload) 없이 클라이언트 단에서 빠른 SPA 방식의 페이지 이동을 구현함.
* **Active Link 구현**: `usePathname()`의 반환값과 `<Link>`의 `href` 경로를 비교(`isActive`)하여 현재 활성화된 메뉴에 동적 인라인 스타일을 적용함.
* **`useParams()`**: Dynamic Routes 경로(`/menu/[menuCode]`)의 가변 파라미터 값을 클라이언트 컴포넌트에서 추출하는 방식을 체득함.
* **`useSearchParams()`**: URL 쿼리 스트링(`?menuName=value`)을 분석 및 추출하여 검색 조건에 따른 데이터 필터링을 구현하는 기법을 파악함.

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

---

### 스트리밍 및 Suspense 처리 (`<Suspense>`)

* **`useSearchParams` 바운더리 격리**: App Router 환경에서 `useSearchParams()`를 호출하는 클라이언트 컴포넌트는 SSR 및 빌드 시점 오류를 방지하기 위해 반드시 `<Suspense>` 바운더리로 감싸야 함을 깨달음.
* **로딩 Fallback UI**: 데이터 파싱 및 로딩 중일 때 표시할 대체 UI(`fallback`)를 지정하여 사용자 경험(UX)을 향상시키는 구조를 습득함.

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

---

### 이미지 최적화 (`next/image`)

* **`<Image/>` 컴포넌트**: 자동 이미지 리사이징, WebP/AVIF 최적 포맷 변환, 레이아웃 시프트(CLS) 예방 기능을 파악함.
* **`priority` 속성**: LCP(Largest Contentful Paint)에 해당하는 주요 이미지(로고 등)에 부여하여 사전 로딩(Preload) 처리하는 최적화 기법을 이해함.

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

---

### Google Fonts 최적화 (`next/font/google`)

* **빌드 시점 최적화**: 외부 폰트 요청으로 인한 FOUT/FOIT(폰트 깜빡임) 현상을 방지하도록 빌드 타임에 폰트를 자체 호스팅 변환함을 확인함.
* **CSS 변수 연결**: `variable: "--font-geist-sans"` 옵션으로 CSS 변수를 생성하고 `<html>` 태그 클래스에 주입하여 CSS Module에서 `var(--font-geist-sans)` 형태로 활용하는 구조를 체득함.

---

### 서버 사이드 메타데이터 (Metadata API)

* **`export const metadata`**: `layout.js` 또는 `page.js` 파일 내에서 메타데이터 객체를 export 하여 `title`, `description` 등 SEO 메타 태그를 서버 측에서 자동 생성하는 방식을 습득함.

---

### 스타일 스코핑 (CSS Modules)

* **`*.module.css`**: 클래스 이름 충돌을 원천 차단하기 위해 고유한 지역 스코프 클래스명을 생성·관리함.
* **`color-mix()`, CSS 변수, 미디어 쿼리**: 다크 모드(`@media (prefers-color-scheme: dark)`) 및 반응형 해상도 대응을 CSS 변수 기반으로 구현하는 패턴을 파악함.

---

## 2. React 핵심 기술 & 패턴

### 서버 컴포넌트(RSC) vs 클라이언트 컴포넌트(RCC)

* **`"use client"` 지시어**: 파일 최상단에 선언하여 클라이언트 경계(Client Boundary)를 지정하는 역할을 이해함.
* **클라이언트 컴포넌트 적용 대상**: 브라우저 API, React Hook(`useState`, `useEffect`, `useParams`, `useSearchParams` 등), 이벤트 리스너를 사용하는 컴포넌트에 필수 선언해야 함을 깨달음.
* **기본 서버 컴포넌트**: `"use client"` 선언이 없는 컴포넌트(`Dashboard`, `RootLayout`, `Header`)는 서버에서만 실행되어 번들 용량을 극적으로 축소할 수 있음을 확인함.

---

### React Hooks 및 데이터 바인딩 패턴

* **`useState`**: 컴포넌트 내부의 로컬 상태(단일 객체, 목록 배열 등)를 관리하는 기초를 다짐.
* **`useEffect`와 의존성 배열**: URL 변경 사항(`menuCode`, `menuName`)을 감지하여 최신 비동기 데이터로 상태를 동적 갱신하는 라이프사이클 처리 방식을 습득함.
* **`usePathname` / `useParams` / `useSearchParams**`: 라우터 상태 및 URL 정보 변화에 반응하는 UI rendering 지원 Next.js 전용 클라이언트 Hook을 숙지함.

---

### 컴포넌트 합성 및 구조 분리 (Component Composition & Boundary Splitting)

* **`Layout Pattern`**: `<Header/>`, `<Navbar/>`와 같은 공통 틀을 레이아웃 컴포넌트에 배치하고, 개별 비즈니스 뷰는 `{children}`을 통해 외부에서 주입받는 유연한 컴포넌트 구조 패턴을 구현함.
* **Suspense Wrapper 패턴**: Client Component 내에서 `useSearchParams`를 안전하게 사용하기 위해, 비즈니스 로직 컴포넌트(`MenuSearchResultContent`)와 Suspense 래퍼 컴포넌트(`MenuSearchResult`)를 분리 설계하는 아키텍처를 체득함.

---

## 느낀 점

* Next.js App Router의 핵심은 **서버 컴포넌트 중심 설계**와 **클라이언트 경계의 명확한 분리**에 있음을 확인함.
* 상단 메뉴 및 레이아웃 틀은 서버 컴포넌트나 공통 Layout으로 유지하고, URL 상태에 따라 스타일 변경이 필요한 내비게이션 요소만 `"use client"`와 `usePathname`을 결합하여 최소한의 범위만 클라이언트 사이드로 동작시키는 기법의 효율성을 체득함.
* Dynamic Route의 `useParams`와 Query String의 `useSearchParams`를 결합하여 RESTful한 라우팅을 손쉽게 구축할 수 있음을 깨달았으며, 특히 `useSearchParams` 적용 시 클라이언트 바운더리 오류 방지를 위한 `<Suspense>` 래퍼 컴포넌트 분리 패턴의 중요성을 파악함.
* 단순 `<img>` 및 표준 웹 폰트 대신 `<Image/>` 컴포넌트와 `next/font/google`을 활용하여 CLS 차단 및 폰트 로딩 성능까지 고려하는 최신 웹 프론트엔드 최적화 기법을 파악함.