# 포켓헙 (PocketHub)

> Git 같은 파일 관리 시스템과 일정(Task) 관리 시스템을 합친 협업 서비스

- **주제:** 코드·파일 버전 관리와 Task 관리를 하나로 합친 통합 협업 플랫폼
- **기획 배경:** 개발 프로젝트에서 코드는 GitHub, 작업은 Trello·Jira처럼 서로 다른 서비스로 관리하는 경우가 많아, 작업 현황과 코드 변경 내역을 함께 파악하기 번거로움
- **주요 사용자**
  - 협업 및 일정 관리가 필요한 사용자
  - 문서나 프로그램을 공유하고 싶은 사용자
  - 영어 사용에 익숙하지 않은 사용자 (Git 명령어 대신 한글 GUI 제공)
- **개발 기간:** 2026.11.02 ~ (진행 중)
- **팀 구성:** 5명 (아래 [팀 구성](#팀-구성) 참고)
- **내 역할:** Git·GitHub 담당(브랜치 규칙, PR·충돌 관리) / 도메인 기능 **PR 생성·승인, PR 리뷰** 풀스택 개발
- **저장소:** [github.com/PocketHub-SeSAC/PocketHub](https://github.com/PocketHub-SeSAC/PocketHub)
- **배포 주소:** <!-- 배포 후 추가 -->

### 기대 효과

- 코드 관리와 Task 관리를 한 플랫폼에서 할 수 있어 여러 서비스를 오가는 번거로움이 줄어듦
- 직관적인 GUI 덕분에 Git 명령어에 익숙하지 않은 초보 개발자도 버전 관리를 쉽게 사용
- 프로젝트별 작업 진행 상황과 코드 변경 이력을 한곳에서 확인
- 개발 도구의 진입 장벽을 낮추고 쉽게 협업을 시도해 볼 수 있음

---

## 팀 구성

| 역할 | 담당자 | 담당 내용 |
| :--- | :--- | :--- |
| 팀장 | 팀장 | 일정 관리, 회의 진행, 업무 조율 |
| DB·ERD | 팀원1 | DB 구조 및 테이블 설계 관리 |
| **Git·GitHub** | **나** | 브랜치 규칙, PR 및 충돌 관리 |
| 문서 | 팀원2 | 회의록, 요구사항 및 진행 상황 정리 |
| UI·디자인 가이드, 발표 | 팀원3 | 화면 디자인과 공통 스타일 관리, 발표 자료·시연 준비 |

* 위 공통 역할과 별도로, 각자 도메인 기능 하나 이상을 맡아 **화면 + 서버를 모두 개발(풀스택)** 하는 방식
* 내 도메인 기능: **PR(Pull Request) 생성·승인, PR 리뷰 작성** (화면 + API)

---

## 기술 스택

| 영역 | 기술 |
| :--- | :--- |
| Frontend | Next.js 16 (App Router), React 19, JavaScript, Playwright (E2E 테스트) |
| Backend | Java 21, Spring Boot 3.5, Spring Web, Spring Data JPA, Validation, Spring Security, Lombok, Gradle |
| Database | MySQL 8 (`db/schema.sql`을 직접 실행, `ddl-auto: none`) |
| 협업 도구 | Git·GitHub, Notion, Postman |
| 개발 도구 | IntelliJ IDEA, VS Code |

* 노션 개발 환경 문서에는 Spring Boot 4.x, Next.js 14/15, jjwt로 적혀 있지만, **저장소 기준 실제 버전은 Spring Boot 3.5.16, Next.js 16.4.6**이고 JWT 라이브러리는 아직 추가되지 않음 (11/12 기준)

---

## 주요 기능

| 기능 | 설명 |
| :--- | :--- |
| 회원가입 / 로그인 | |
| 팀 생성 | |
| 파일 업로드 / 다운로드 | |
| Task 생성 / 관리 | |
| 파일과 Task 연결 | |
| PR 생성 / 승인 ⭐ 내 담당 | |
| 리뷰 작성 ⭐ 내 담당 | |

---

## 프로젝트 구조

```text
PocketHub/
├── backend/                   Spring Boot
│   ├── db/schema.sql          MySQL에서 직접 실행 (앱이 자동 실행하지 않음)
│   └── src/main/java/com/pockethub
│       ├── common/            response · exception · config · auth · validation
│       └── member · team · file · commit · task · pr · review
│            └── controller · service · repository · domain · dto
├── frontend/                  Next.js (App Router), 백엔드 없이 목업 데이터로 동작
│   ├── app/                   주소별 화면 (/auth, /me, /teams/[teamId]/files·commits·tasks·pulls)
│   ├── components/ services/ mocks/ hooks/ utils/
│   └── e2e/                   Playwright 테스트
└── docs/convention.md         팀 컨벤션
```

* 프론트는 `NEXT_PUBLIC_USE_MOCK=true`면 브라우저 안 목업 서버, `false`면 실제 API로 요청함. 화면은 `services/*.js` 함수만 부르기 때문에 전환해도 화면 코드를 고치지 않음

---

## 아키텍처

<!-- 프론트 ↔ API 서버 ↔ DB 구조도, 인증 흐름(Access/Refresh Token) 등 -->

---

## ERD

테이블 (`backend/db/schema.sql` 기준): `user`, `repository`, `collaborator`, `task`, `pr`, `commit`, `file`, `review`, `refresh_token`

<!-- ERD 이미지 또는 mermaid erDiagram -->

---

## API 명세

| Method | URL | 설명 | 인증 |
| :--- | :--- | :--- | :--- |
| | | | |

---

## 협업 규칙

자세한 내용: [docs/convention.md](https://github.com/PocketHub-SeSAC/PocketHub/blob/main/docs/convention.md)

| 항목 | 규칙 |
| :--- | :--- |
| 브랜치 | `main`(배포) · `dev`(통합) · `feature/` · `fix/` · `hotfix/`, 이름은 `타입/기능명` (영문 소문자·하이픈) |
| 작업 흐름 | `dev`에서 브랜치 생성 → push → `dev` 대상 PR → 1명 이상 승인 후 병합. `main`·`dev` 직접 push 금지, 강제 push 금지 |
| 커밋 | `타입: 한글 내용` (feat, fix, docs, style, refactor, test, chore, perf) |
| PR 제목 | `[타입] 작업 내용` |
| 보안 | `.env` 등 실제 환경변수와 개인 설정 파일은 올리지 않음 |
| 프론트 코드 | 컴포넌트 PascalCase, 폴더 kebab-case, 작은따옴표, 세미콜론, 들여쓰기 2칸 |

---

## 개발 기록

날짜별 작업 내용은 이 폴더에 `YYMMDD-주제.md` 파일로 추가합니다.

| 날짜 | 파일명 | 내용 |
| :--- | :--- | :--- |
| `26.11.02` | | 팀 구성·역할 분담, 프로젝트 기획 |
| `26.11.11` | | 저장소 생성, 백엔드 초기 세팅(폴더 구조·DB 스키마·입력 검증 상수), 프론트 목업 추가 |
| `26.11.12` | | 프론트 Next.js 이전, Playwright E2E 테스트, 팀 컨벤션 문서 추가 |

---

## 트러블슈팅

<!-- 문제 → 원인 → 해결 → 결과 -->

---

## 회고

