# SQL 기본 조회 · 정렬 · 조건 · JOIN · GROUP BY · 서브쿼리 · 집합 연산자 정리 (TIL)

> 실습 환경: MySQL, 메뉴 테이블(`tbl_menu`)과 카테고리 테이블(`tbl_category`) 사용

## 0. 데이터베이스와 SQL이 뭔가

* **데이터베이스(DB)**: 데이터를 **엑셀 표**처럼 정리해서 보관하는 창고. 표 하나를 **테이블**, 가로줄 하나를 **행(row)**, 세로줄 하나를 **열(column, 컬럼)** 이라고 부름.
* **SQL**: DB에게 "이 표에서 이런 데이터 좀 보여줘"라고 **부탁하는 말**. 영어 문장처럼 생겼음.
* **오늘 쓰는 표**
  * `tbl_menu`: 메뉴 코드, 메뉴 이름, 가격, 카테고리 코드, 주문 가능 여부(`Y`/`N`)
  * `tbl_category`: 카테고리 코드, 카테고리 이름, 상위 카테고리 코드(`ref_category_code`)

## 1. SELECT

* **개념**: `SELECT 컬럼명 FROM 테이블명` → "**어느 표(FROM)** 에서 **어떤 열(SELECT)** 을 보여줘."

```sql
-- 단일 열 조회
SELECT menu_name FROM tbl_menu;

-- 여러 열 조회
SELECT
    menu_code,
    menu_name,
    menu_price
FROM
    tbl_menu;

-- 모든 열 조회 (*: 모든 컬럼을 의미하는 와일드카드)
SELECT * FROM tbl_menu;
```

* **한 줄씩 보기**
  * `SELECT menu_name FROM tbl_menu;` → 메뉴 표에서 **이름 열만** 모든 행 보여줌.
  * 여러 열 → 쉼표로 나열한 순서대로 코드·이름·가격 세 열이 나옴. 줄을 나눠 써도 같은 문장임(세미콜론 `;`이 문장의 끝).
  * `SELECT *` → `*`는 "전부"라는 뜻 → 모든 열이 나옴.

* **FROM 없는 SELECT**: 표 없이 계산 결과나 함수 결과만 볼 수도 있음. `AS`는 결과 열에 **별명**을 붙임.

```sql
SELECT 6 + 3;                              -- 9
SELECT NOW();                              -- 현재 날짜와 시간
SELECT CONCAT('홍', '길동') AS name;        -- 홍길동
```

* **실행 결과**: `9` / 지금 시각(예: `2026-10-20 14:00:00`) / 열 이름이 `name`인 칸에 `홍길동` (`CONCAT`은 글자 이어 붙이기)

---

## 2. ORDER BY (정렬)

* **개념**: `SELECT`문의 거의 마지막(`LIMIT`이 있으면 그 바로 앞)에 위치하며, 결과를 기준 열로 줄 세움.

| 키워드 | 의미 | 비고 |
| :--- | :--- | :--- |
| `ASC` | 오름차순 (작은 것 → 큰 것, ㄱ → ㅎ) | 기본값이라 생략 가능 |
| `DESC` | 내림차순 (큰 것 → 작은 것) | 생략 불가 |

```sql
SELECT menu_code, menu_name, menu_price
FROM tbl_menu
ORDER BY
    menu_price DESC,   -- 1차 기준: 가격 내림차순
    menu_name ASC;     -- 2차 기준: 가격이 같으면 이름 오름차순
```

* **따라가기**: 먼저 비싼 순서로 세움 → 가격이 같은 메뉴끼리만 이름 가나다순으로 다시 세움. 키 순서로 줄 세우고, 키가 같으면 이름순으로 세우는 것과 같음.

* **계산한 값으로 정렬**: 별명을 붙인 계산 열로도 정렬할 수 있음.

```sql
SELECT
    menu_code,
    menu_price,
    menu_code * menu_price AS calculated_value
FROM tbl_menu
ORDER BY calculated_value DESC;
```

* **따라가기**: 행마다 `코드 × 가격`을 계산해 `calculated_value` 열을 만듦(예: 코드 2, 가격 9000 → 18000) → 그 값이 큰 순서로 정렬.

* **`FIELD()`로 내가 원하는 순서 정하기**: `FIELD(값, 첫번째, 두번째, ...)`는 값이 목록의 **몇 번째**인지 숫자로 돌려줌. 이 숫자로 정렬하면 원하는 순서가 됨.

```sql
SELECT FIELD('B', 'A', 'B', 'C');   -- 2

-- 'N'을 1순위, 'Y'를 2순위로 정렬
SELECT menu_name, orderable_status
FROM tbl_menu
ORDER BY FIELD(orderable_status, 'N', 'Y');
```

* **따라가기**: `'B'`는 목록 `A, B, C`에서 2번째 → `2`. 메뉴 정렬에서는 `N`인 행은 1, `Y`인 행은 2가 되어 → 숫자 오름차순 → **주문 불가(N) 메뉴가 먼저** 나옴.

* **NULL 위치 정하기**: `NULL`은 "값이 없음"이라는 뜻. `컬럼 IS NULL`은 `NULL`이면 `1`, 아니면 `0`을 돌려줌 → 이걸 1차 정렬 기준으로 두면 `NULL`의 위치를 앞이나 뒤로 보낼 수 있음.

```sql
-- 오름차순 + NULL을 마지막으로
SELECT * FROM tbl_category
ORDER BY
    ref_category_code IS NULL,          -- NULL이 아닌 행(0)이 먼저
    ref_category_code;

-- 내림차순 + NULL을 처음으로
SELECT * FROM tbl_category
ORDER BY
    ref_category_code IS NULL DESC,     -- NULL인 행(1)이 먼저
    ref_category_code DESC;
```

* **따라가기 (첫 번째)**: 1차 기준 → `NULL` 아닌 행(0)이 앞, `NULL`인 행(1)이 뒤 → 2차 기준 → 앞쪽 무리 안에서 코드 오름차순. 결과적으로 "값 있는 것 작은 순 → 맨 끝에 NULL들".

---

## 3. WHERE (조건으로 거르기)

* **개념**: 조건에 맞는 **행만** 골라냄. 체로 거르는 것과 같음.

| 구분 | 연산자 | 예시 |
| :--- | :--- | :--- |
| 비교 | `=`, `<>`(`!=`), `>`, `>=`, `<`, `<=` | `orderable_status <> 'Y'` |
| 논리 | `AND`(`&&`), `OR`(`\|\|`), `NOT` | `orderable_status = 'Y' AND category_code = 10` |
| 범위 | `BETWEEN A AND B` | `menu_price BETWEEN 10000 AND 25000` |
| 패턴 | `LIKE` | `menu_name LIKE '%마늘%'` |
| 목록 | `IN (...)` | `category_code IN (4, 5, 6)` |
| NULL | `IS NULL`, `IS NOT NULL` | `ref_category_code IS NULL` |

* SQL에서 "같다"는 `=` 하나임(자바의 `==`가 아님). `<>`는 "같지 않다".
* 부정은 앞에 `NOT`을 붙임 → `NOT BETWEEN`, `NOT LIKE`, `NOT IN`, `IS NOT NULL`

```sql
-- 주문 가능(Y)하면서 카테고리가 10번인 메뉴
SELECT menu_name, category_code, orderable_status
FROM tbl_menu
WHERE orderable_status = 'Y' AND category_code = 10;

-- 가격이 10,000원 이상 25,000원 이하인 메뉴
SELECT menu_name, menu_price
FROM tbl_menu
WHERE menu_price BETWEEN 10000 AND 25000;
-- WHERE menu_price >= 10000 AND menu_price <= 25000; 와 같음
```

* **따라가기**
  * 첫 번째 → 두 조건을 **모두** 만족하는 행만 남음(`AND`).
  * 두 번째 → `BETWEEN A AND B`는 **A와 B를 포함**함 → 10000원짜리, 25000원짜리도 나옴.

* **AND가 OR보다 먼저 계산됨**: 수학에서 곱셈이 덧셈보다 먼저인 것과 같음. 원하는 순서가 따로 있으면 괄호로 묶어야 함.

```sql
SELECT 1 OR 0 AND 0;     -- 1  → 1 OR (0 AND 0)
SELECT (1 OR 0) AND 0;   -- 0
```

* **따라가기** (MySQL에서 1은 참, 0은 거짓)
  * `1 OR 0 AND 0` → `AND` 먼저: `0 AND 0` = 0 → `1 OR 0` = **1**
  * `(1 OR 0) AND 0` → 괄호 먼저: 1 → `1 AND 0` = **0**

* **LIKE 와일드카드**: `%`는 "**아무 글자가 몇 개든(0개 포함)**"이라는 뜻.

```sql
SELECT menu_name FROM tbl_menu
WHERE menu_name LIKE '%마늘%';   -- '마늘'이 어디든 포함
-- LIKE '마늘%'  → '마늘'로 시작
-- LIKE '%마늘'  → '마늘'로 끝남
```

* **예**: `'%마늘%'` → "마늘빵", "흑마늘라떼", "갈릭마늘" 모두 해당. `'마늘%'` → "마늘빵"만. `'%마늘'` → "갈릭마늘"만.

* **IN / IS NULL**

```sql
-- 카테고리 코드가 4, 5, 6 중 하나인 메뉴
SELECT menu_name, category_code
FROM tbl_menu
WHERE category_code IN (4, 5, 6);

-- 상위 카테고리가 없는(최상위) 카테고리
SELECT category_code, category_name, ref_category_code
FROM tbl_category
WHERE ref_category_code IS NULL;   -- '= NULL'로는 비교할 수 없음
```

* **따라가기**
  * `IN (4, 5, 6)` → `category_code = 4 OR category_code = 5 OR category_code = 6`을 짧게 쓴 것.
  * `NULL`은 "모르는 값"이라 `= NULL`로 비교하면 결과가 참도 거짓도 아닌 `NULL`이 되어 아무 행도 안 나옴 → 반드시 `IS NULL`로 씀.

---

## 4. DISTINCT (중복 제거)

* **개념**: 결과에서 **겹치는 값을 하나만** 남김.

```sql
SELECT DISTINCT category_code
FROM tbl_menu
ORDER BY category_code;
```

* **따라가기**: 메뉴가 20개여도 카테고리 코드가 `4, 4, 5, 10, 10, 10…`처럼 겹치면 → `4, 5, 10…`처럼 종류만 한 번씩 나옴.

* **열이 여러 개면**: 지정한 **모든 열의 조합**이 똑같을 때만 중복으로 봄.

```sql
SELECT DISTINCT
    category_code,
    orderable_status
FROM tbl_menu;   -- (category_code, orderable_status) 쌍이 같은 행만 제거
```

* **예**: `(4, Y)`, `(4, Y)`, `(4, N)` → `(4, Y)`, `(4, N)` 두 줄이 남음. 코드가 같아도 상태가 다르면 다른 쌍임.

---

## 5. LIMIT

* **개념**: 결과 중 **원하는 개수만** 잘라서 가져옴. 정렬과 함께 쓰면 "상위 N개"가 됨.
* **형식**: `LIMIT 개수` 또는 `LIMIT 건너뛸개수, 개수` (건너뛸 개수는 0부터 셈)

```sql
-- 가장 비싼 메뉴 5개
SELECT * FROM tbl_menu
ORDER BY menu_price DESC
LIMIT 5;

-- 가장 비싼 메뉴 1개를 건너뛰고, 2위부터 4개
SELECT * FROM tbl_menu
ORDER BY menu_price DESC
LIMIT 1, 4;
```

* **따라가기**
  * 첫 번째 → 비싼 순 정렬 → 위에서 5줄 → 1~5위.
  * 두 번째 → 비싼 순 정렬 → **1줄 건너뛰고** → 그다음 4줄 → 2~5위. 게시판 2페이지처럼 "몇 개 넘기고 몇 개 보기"에 씀.

---

## 6. JOIN

* **왜 필요한가**: 메뉴 표에는 카테고리 **번호**(`category_code = 4`)만 있고, 카테고리 **이름**("한식")은 카테고리 표에 있음. 두 표를 **같은 번호끼리 이어 붙여야** "김치찌개 - 한식"처럼 함께 볼 수 있음. 학생 명부의 반 번호로 담임 명부를 찾아 담임 이름을 옆에 적는 것과 같음.
* **별칭(Alias)**
  * **컬럼 별칭**: 결과 열 이름을 바꿈. 띄어쓰기나 특수문자가 있으면 따옴표가 필요하고, 간단하면 `AS`와 따옴표를 생략해도 됨.
  * **테이블 별칭**: 표에 짧은 이름을 붙임(`tbl_menu a`). JOIN에서 "이 열이 어느 표 것인지"를 `a.menu_name`처럼 구분하려면 사실상 필수. `AS`는 생략 가능.

```sql
SELECT
    menu_name AS '메뉴 이름',
    menu_price AS price
FROM tbl_menu;

SELECT a.menu_name, a.menu_price
FROM tbl_menu a;
```

| 종류 | 설명 |
| :--- | :--- |
| `INNER JOIN` (`JOIN`) | 두 테이블에서 연결 조건이 일치하는 행만 조회 (교집합) |
| `LEFT JOIN` | 왼쪽 테이블은 모두 보여주고, 오른쪽에 짝이 없으면 `NULL`로 채움 |
| `RIGHT JOIN` | 오른쪽 테이블 기준으로 `LEFT JOIN`과 반대 |
| `SELF JOIN` | 같은 테이블을 두 번 사용해 행과 행 사이의 관계를 연결 |

* **INNER JOIN**: `ON` 뒤에 "어떤 열이 같은 것끼리 이을지" **연결 조건**을 씀.

```sql
-- 메뉴 이름과 카테고리 이름을 함께 조회
SELECT a.menu_name, b.category_name
FROM tbl_menu a
JOIN tbl_category b ON a.category_code = b.category_code;
```

* **따라가기**: 메뉴 "김치찌개"(category_code 4) → 카테고리 표에서 코드 4를 찾음 → "한식" → `김치찌개 | 한식`. 짝이 없는 행은 결과에서 빠짐.

* **LEFT JOIN**: 왼쪽 표의 행은 **짝이 없어도 전부** 남김. 짝이 없으면 오른쪽 칸은 `NULL`.

```sql
SELECT a.category_name, b.menu_name
FROM tbl_category a                -- 기준(왼쪽) 테이블
LEFT JOIN tbl_menu b ON a.category_code = b.category_code;
```

* **따라가기**: 카테고리 "디저트"에 메뉴가 하나도 없어도 → `디저트 | NULL`로 결과에 나옴. INNER JOIN이었다면 "디저트" 줄은 아예 사라짐.

* **SELF JOIN**: `tbl_category`는 `ref_category_code`로 **같은 표 안의 다른 행(상위 카테고리)** 을 가리킴. 같은 표에 별칭을 두 개 붙여 "하위 역할"과 "상위 역할"로 나눠 이음.

```sql
SELECT
    a.category_name AS '하위 카테고리',
    b.category_name AS '상위 카테고리'
FROM tbl_category a                -- 하위 역할
JOIN tbl_category b                -- 상위 역할
    ON a.ref_category_code = b.category_code;
```

* **따라가기**: "한식"(코드 4, 상위 1) → 같은 표에서 코드 1인 행 "식사"를 찾음 → `한식 | 식사`. 상위가 `NULL`인 최상위 카테고리는 짝이 없어 빠짐.

---

## 7. GROUP BY

* **개념**: 같은 값을 가진 행들을 **한 묶음(그룹)** 으로 모으고, 묶음마다 **집계함수**로 계산함. 반별로 학생을 모아 반 평균 점수를 내는 것과 같음.

| 집계함수 | 의미 |
| :--- | :--- |
| `COUNT(*)` | 그룹에 속한 행의 개수 |
| `SUM(컬럼)` | 합계 |
| `AVG(컬럼)` | 평균 |
| `MAX(컬럼)` / `MIN(컬럼)` | 최댓값 / 최솟값 |

```sql
SELECT
    category_code,
    COUNT(*) AS '메뉴 개수',
    SUM(menu_price) AS '가격 총합',
    AVG(menu_price) AS '가격 평균'
FROM tbl_menu
GROUP BY category_code;
```

* **따라가기**: 카테고리 4번 메뉴가 9000원, 10000원, 11000원 3개라면 → `4 | 3 | 30000 | 10000.0000`. 카테고리마다 이런 한 줄씩 나옴.

* **주의**: `GROUP BY`를 쓰면 `SELECT`에는 **`GROUP BY`에 쓴 열과 집계함수만** 올 수 있음. 묶음 하나는 결과에서 한 줄이 되는데, `menu_code`처럼 묶음 안에서 값이 여러 개인 열은 어떤 값을 보여줄지 정할 수 없기 때문임.

* **HAVING**: **그룹**에 거는 조건.
  * `WHERE` → 묶기 **전에** 행을 거름 (집계함수 사용 불가)
  * `HAVING` → 묶은 **후에** 그룹을 거름 (집계함수 사용 가능)

```sql
-- 메뉴가 3개 이상인 카테고리만
SELECT category_code, COUNT(*) AS '메뉴 개수'
FROM tbl_menu
GROUP BY category_code
HAVING COUNT(*) >= 3;
```

* **따라가기**: 카테고리별로 묶어 개수를 셈 → 개수가 3 이상인 그룹만 남김.

* **WITH ROLLUP**: 그룹별 결과에 **소계와 총계** 줄을 더 붙여줌. 소계·총계 줄에서는 묶이지 않은 열이 `NULL`로 나옴.

```sql
-- 카테고리별 합계 + 마지막 행에 전체 총계
SELECT category_code, SUM(menu_price)
FROM tbl_menu
GROUP BY category_code
WITH ROLLUP;

-- 가격별 → 그 안에서 카테고리별로 그룹화, 가격별 소계와 전체 총계까지 출력
SELECT menu_price, category_code, SUM(menu_price)
FROM tbl_menu
GROUP BY menu_price, category_code
WITH ROLLUP;
```

* **따라가기 (첫 번째)**: `4 | 30000`, `5 | 20000`, … 그리고 맨 아래 `NULL | (전체 합)` 한 줄이 추가됨. `NULL`은 "카테고리 구분 없이 전부"라는 뜻.

---

## 8. 서브쿼리 (Subquery)

* **개념**: 쿼리 **안에 들어 있는 또 다른 쿼리**. 안쪽 쿼리의 결과를 바깥 쿼리가 조건이나 표처럼 씀. 작성할 때는 **안쪽 쿼리를 먼저 혼자 실행해 결과를 확인**한 뒤 바깥에 끼워 넣음.

* **WHERE 절 서브쿼리**

```sql
-- '민트미역국'과 같은 카테고리의 메뉴 조회
SELECT menu_name, category_code
FROM tbl_menu
WHERE category_code = (SELECT category_code
                       FROM tbl_menu
                       WHERE menu_name = '민트미역국');
```

* **따라가기**: ① 안쪽: 민트미역국의 카테고리 코드 → 예: `4` ② 바깥: `WHERE category_code = 4` → 4번 카테고리 메뉴 전부.

* **FROM 절 서브쿼리 (파생 테이블)**: 그 자리에서 만들어 쓰는 **임시 표**. 반드시 **별칭**을 붙여야 함.

```sql
-- 가장 많은 메뉴가 포함된 카테고리의 메뉴 수
SELECT MAX(count) AS '최대 메뉴 수'
FROM (SELECT COUNT(*) AS 'count'
      FROM tbl_menu
      GROUP BY category_code) AS count_table;
```

* **따라가기**: ① 안쪽: 카테고리별 메뉴 수 → 예: `3, 2, 5, 1` (임시 표 `count_table`) ② 바깥: 그 표에서 `MAX` → `5`

* **상관 서브쿼리**: 안쪽 쿼리가 **바깥 쿼리의 값**(`a.category_code`)을 씀 → 바깥의 **행마다** 안쪽이 그 행 값으로 다시 계산됨.

```sql
-- 자기 카테고리의 평균 가격보다 비싼 메뉴
SELECT menu_code, menu_name, menu_price
FROM tbl_menu a
WHERE menu_price > (SELECT AVG(menu_price)
                    FROM tbl_menu
                    WHERE category_code = a.category_code);
```

* **따라가기**: 바깥에서 "김치찌개(4번, 11000원)" 행을 볼 때 → 안쪽: 4번 카테고리 평균 → 10000 → `11000 > 10000` → 결과에 남음. 다음 행으로 넘어가면 그 행의 카테고리로 평균을 다시 계산함.

---

## 9. 집합 연산자 (Set Operators)

* **UNION / UNION ALL**: 두 `SELECT` 결과를 **위아래로 이어 붙임**. 두 쿼리의 **열 개수와 순서**가 같아야 함.

| 연산자 | 중복 처리 |
| :--- | :--- |
| `UNION` | 중복 행 제거 |
| `UNION ALL` | 중복 행 포함 (그대로 이어 붙임) |

```sql
-- 카테고리 10번 메뉴 + 9,000원 미만 메뉴
SELECT menu_name, menu_price, category_code
FROM tbl_menu
WHERE category_code = 10
UNION            -- 두 조건을 모두 만족하는 메뉴는 한 번만 나옴
SELECT menu_name, menu_price, category_code
FROM tbl_menu
WHERE menu_price < 9000;
```

* **따라가기**: 10번 메뉴가 A, B, C이고 9000원 미만 메뉴가 C, D라면 → `UNION` → A, B, C, D (C는 한 번만). `UNION ALL`이었다면 A, B, C, C, D.

* **교집합**: 수업에서는 `IN` 서브쿼리로 구현함. (카테고리 10번 **이면서** 9,000원 미만)

```sql
SELECT a.menu_code, a.menu_price, a.category_code
FROM tbl_menu a
WHERE a.category_code = 10
  AND a.menu_code IN (SELECT menu_code
                      FROM tbl_menu
                      WHERE menu_price < 9000);
```

* **따라가기**: 위 예시라면 10번(A, B, C) 중 9000원 미만 목록(C, D)에 **들어 있는 것** → C.

* **차집합**: `LEFT JOIN` 후 오른쪽이 `NULL`인 행만 남기는 방식으로 구현함. (카테고리 10번 메뉴 중 9,000원 미만인 메뉴를 **제외**)

```sql
SELECT a.menu_code, a.menu_price, a.category_code
FROM tbl_menu a
LEFT JOIN (SELECT menu_code
           FROM tbl_menu
           WHERE menu_price < 9000) b ON a.menu_code = b.menu_code
WHERE a.category_code = 10
  AND b.menu_code IS NULL;   -- 짝이 없는(= 9,000원 미만이 아닌) 메뉴만
```

* **따라가기**: 모든 메뉴(왼쪽)에 9000원 미만 목록(오른쪽)을 LEFT JOIN → 9000원 미만인 메뉴는 오른쪽 짝이 있고, 아닌 메뉴는 오른쪽이 `NULL` → 10번이면서 오른쪽이 `NULL`인 것 → A, B.

---

## 📝 오늘의 정리

### SELECT 문 작성 순서

```sql
SELECT    컬럼, 집계함수
FROM      테이블 [JOIN 테이블 ON 조건]
WHERE     행 조건
GROUP BY  그룹 기준 컬럼
HAVING    그룹 조건
ORDER BY  정렬 기준
LIMIT     개수;
```

1. `SELECT`는 작성 순서와 실제 실행 순서(`FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`)가 다름. 표를 고르고 → 행을 거르고 → 묶고 → 그룹을 거르고 → 보여줄 열을 정하고 → 줄 세우고 → 자르는 순서임
2. `WHERE`는 **행**을 거르고, `HAVING`은 `GROUP BY`로 묶인 **그룹**을 거름
3. `INNER JOIN`은 양쪽에 모두 있는 행만, `LEFT JOIN`은 왼쪽 테이블의 행을 모두 남김
4. 서브쿼리와 집합 연산자(`UNION` 등)로 여러 조회 결과를 조합할 수 있음

---

## 느낀 점

* SQL 작성 순서와 실행 순서가 다르다는 것을 알고 나니, `SELECT`에서 지은 별칭을 `WHERE`에서 쓸 수 없는 이유가 이해됨
* JOIN 종류에 따라 결과 행 수가 달라지는 것을 직접 확인하면서, 어떤 테이블을 기준으로 할지 먼저 정해야 한다고 느낌
