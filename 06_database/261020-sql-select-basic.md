# SQL 기본 조회 · 정렬 · 조건 · JOIN · GROUP BY · 서브쿼리 · 집합 연산자 정리 (TIL)

> 실습 환경: MySQL, 메뉴 테이블(`tbl_menu`)과 카테고리 테이블(`tbl_category`) 사용

## 1. SELECT

* **개념**: `SELECT 컬럼명 FROM 테이블명` 형태로 테이블에서 원하는 열(컬럼)의 데이터를 조회함을 학습함.

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

* **FROM 없는 SELECT**: 테이블 없이 연산 결과나 함수 결과만 확인할 수도 있음. `AS`로 결과 컬럼에 별칭을 붙일 수 있음.

```sql
SELECT 6 + 3;                              -- 9
SELECT NOW();                              -- 현재 날짜와 시간
SELECT CONCAT('홍', '길동') AS name;        -- 홍길동
```

---

## 2. ORDER BY (정렬)

* **개념**: `SELECT`문의 **가장 마지막**에 위치하며, 결과 집합을 기준 컬럼으로 정렬함.

| 키워드 | 의미 | 비고 |
| :--- | :--- | :--- |
| `ASC` | 오름차순 | 기본값이라 생략 가능 |
| `DESC` | 내림차순 | 생략 불가 |

```sql
SELECT menu_code, menu_name, menu_price
FROM tbl_menu
ORDER BY
    menu_price DESC,   -- 1차 기준: 가격 내림차순
    menu_name ASC;     -- 2차 기준: 가격이 같으면 이름 오름차순
```

* **연산 결과로 정렬**: 별칭을 붙인 계산 컬럼으로도 정렬할 수 있음.

```sql
SELECT
    menu_code,
    menu_price,
    menu_code * menu_price AS calculated_value
FROM tbl_menu
ORDER BY calculated_value DESC;
```

* **`FIELD()`로 원하는 순서 지정**: `FIELD(값, 첫번째, 두번째, ...)`는 값이 목록의 **몇 번째**에 있는지 숫자로 반환함. 이를 정렬 기준으로 쓰면 원하는 순서대로 정렬할 수 있음.

```sql
SELECT FIELD('B', 'A', 'B', 'C');   -- 2

-- 'N'을 1순위, 'Y'를 2순위로 정렬
SELECT menu_name, orderable_status
FROM tbl_menu
ORDER BY FIELD(orderable_status, 'N', 'Y');
```

* **NULL 정렬 위치 제어**: `컬럼 IS NULL`은 NULL이면 `1`, 아니면 `0`을 반환함. 이 값을 1차 기준으로 두면 NULL의 위치를 조절할 수 있음.

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

---

## 3. WHERE (조건 필터링)

* **개념**: 특정 조건에 맞는 **행만** 골라내는 역할을 함.

| 구분 | 연산자 | 예시 |
| :--- | :--- | :--- |
| 비교 | `=`, `<>`(`!=`), `>`, `>=`, `<`, `<=` | `orderable_status <> 'Y'` |
| 논리 | `AND`(`&&`), `OR`(`\|\|`), `NOT` | `orderable_status = 'Y' AND category_code = 10` |
| 범위 | `BETWEEN A AND B` | `menu_price BETWEEN 10000 AND 25000` |
| 패턴 | `LIKE` | `menu_name LIKE '%마늘%'` |
| 목록 | `IN (...)` | `category_code IN (4, 5, 6)` |
| NULL | `IS NULL`, `IS NOT NULL` | `ref_category_code IS NULL` |

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

* **AND가 OR보다 우선순위가 높음**: 의도한 순서가 있다면 괄호로 묶어야 함.

```sql
SELECT 1 OR 0 AND 0;     -- 1  → 1 OR (0 AND 0)
SELECT (1 OR 0) AND 0;   -- 0
```

* **LIKE 와일드카드**: `%`는 글자가 **없거나 여러 개** 있어도 된다는 의미임.

```sql
SELECT menu_name FROM tbl_menu
WHERE menu_name LIKE '%마늘%';   -- '마늘'이 어디든 포함
-- LIKE '마늘%'  → '마늘'로 시작
-- LIKE '%마늘'  → '마늘'로 끝남
```

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

---

## 4. DISTINCT (중복 제거)

* **개념**: 중복된 값을 제거하고 **고유한 값**의 목록만 조회함.

```sql
SELECT DISTINCT category_code
FROM tbl_menu
ORDER BY category_code;
```

* **열이 여러 개인 경우**: 지정한 **모든 열의 조합**이 똑같을 때만 중복으로 간주함.

```sql
SELECT DISTINCT
    category_code,
    orderable_status
FROM tbl_menu;   -- (category_code, orderable_status) 쌍이 같은 행만 제거
```

---

## 5. LIMIT

* **개념**: 결과 중에서 원하는 개수만큼만 잘라서 가져옴. 정렬과 함께 쓰면 "상위 N개" 조회가 가능함.
* **형식**: `LIMIT 개수` 또는 `LIMIT 시작위치(건너뛸 개수), 개수` (시작 위치는 0부터)

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

---

## 6. JOIN

* **별칭(Alias)**
  * **컬럼 별칭**: 띄어쓰기나 특수문자가 있으면 따옴표가 필요하고, 간단하면 `AS`와 따옴표를 생략할 수 있음.
  * **테이블 별칭**: 여러 테이블을 다루는 JOIN에서 어느 테이블의 컬럼인지 구분하기 위해 사실상 필수임. `AS`는 생략 가능.

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

* **INNER JOIN**: `ON` 뒤에 같은 행을 찾을 **연결 조건**을 작성함.

```sql
-- 메뉴 이름과 카테고리 이름을 함께 조회
SELECT a.menu_name, b.category_name
FROM tbl_menu a
JOIN tbl_category b ON a.category_code = b.category_code;
```

* **LEFT JOIN**: 메뉴가 하나도 없는 카테고리도 결과에 남고, 메뉴 이름은 `NULL`로 표시됨.

```sql
SELECT a.category_name, b.menu_name
FROM tbl_category a                -- 기준(왼쪽) 테이블
LEFT JOIN tbl_menu b ON a.category_code = b.category_code;
```

* **SELF JOIN**: `tbl_category`는 `ref_category_code`로 자신의 상위 카테고리를 가리킴. 같은 테이블에 별칭을 두 개 붙여 하위/상위 역할을 나눠 연결함.

```sql
SELECT
    a.category_name AS '하위 카테고리',
    b.category_name AS '상위 카테고리'
FROM tbl_category a                -- 하위 역할
JOIN tbl_category b                -- 상위 역할
    ON a.ref_category_code = b.category_code;
```

---

## 7. GROUP BY

* **개념**: 지정한 컬럼의 값이 같은 행들을 **하나의 그룹**으로 묶고, 그룹마다 집계함수로 결과를 계산함.

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

* **주의**: `GROUP BY`를 사용하면 `SELECT`에는 **`GROUP BY`에 사용한 컬럼과 집계함수만** 올 수 있음. (`menu_code`처럼 그룹 안에서 값이 여러 개인 컬럼은 하나로 정할 수 없기 때문)

* **HAVING**: 그룹에 대한 조건 필터링임.
  * `WHERE` → 그룹화 **전** 행을 거름
  * `HAVING` → 그룹화 **후** 그룹을 거름 (집계함수 사용 가능)

```sql
-- 메뉴가 3개 이상인 카테고리만
SELECT category_code, COUNT(*) AS '메뉴 개수'
FROM tbl_menu
GROUP BY category_code
HAVING COUNT(*) >= 3;
```

* **WITH ROLLUP**: 그룹별 집계 결과와 함께 **소계와 총계** 행을 추가로 보여줌. 소계/총계 행은 묶이지 않은 컬럼이 `NULL`로 표시됨.

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

---

## 8. 서브쿼리 (Subquery)

* **개념**: 쿼리 안에 들어있는 또 다른 쿼리임. 서브쿼리 결과를 메인쿼리가 조건이나 테이블처럼 사용함. 먼저 서브쿼리를 단독으로 실행해 결과를 확인한 뒤 메인쿼리에 넣는 순서로 작성함.

* **WHERE 절 서브쿼리**

```sql
-- '민트미역국'과 같은 카테고리의 메뉴 조회
SELECT menu_name, category_code
FROM tbl_menu
WHERE category_code = (SELECT category_code
                       FROM tbl_menu
                       WHERE menu_name = '민트미역국');
```

* **FROM 절 서브쿼리 (파생 테이블)**: 즉석에서 만들어 쓰는 임시 테이블처럼 동작함. 파생 테이블에는 **반드시 별칭**이 있어야 함.

```sql
-- 가장 많은 메뉴가 포함된 카테고리의 메뉴 수
SELECT MAX(count) AS '최대 메뉴 수'
FROM (SELECT COUNT(*) AS 'count'
      FROM tbl_menu
      GROUP BY category_code) AS count_table;
```

* **상관 서브쿼리**: 서브쿼리가 메인쿼리의 값(`a.category_code`)을 참조함. 메인쿼리의 **행마다** 서브쿼리가 그 행의 값으로 다시 계산됨.

```sql
-- 자기 카테고리의 평균 가격보다 비싼 메뉴
SELECT menu_code, menu_name, menu_price
FROM tbl_menu a
WHERE menu_price > (SELECT AVG(menu_price)
                    FROM tbl_menu
                    WHERE category_code = a.category_code);
```

---

## 9. 집합 연산자 (Set Operators)

* **UNION / UNION ALL**: 두 `SELECT` 결과를 위아래로 합침. 두 쿼리의 **컬럼 개수와 순서**가 맞아야 함.

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

* **교집합**: 수업에서는 `IN` 서브쿼리로 구현함. (카테고리 10번 **이면서** 9,000원 미만)

```sql
SELECT a.menu_code, a.menu_price, a.category_code
FROM tbl_menu a
WHERE a.category_code = 10
  AND a.menu_code IN (SELECT menu_code
                      FROM tbl_menu
                      WHERE menu_price < 9000);
```

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

---

## 📝 정리: SELECT 문 작성 순서

```sql
SELECT    컬럼, 집계함수
FROM      테이블 [JOIN 테이블 ON 조건]
WHERE     행 조건
GROUP BY  그룹 기준 컬럼
HAVING    그룹 조건
ORDER BY  정렬 기준
LIMIT     개수;
```
