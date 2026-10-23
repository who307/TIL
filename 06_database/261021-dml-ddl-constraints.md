# SQL DML · DDL · 제약조건 · 형변환 정리 (TIL)

> 실습 환경: MySQL (`menudb`), `tbl_menu` 컬럼 순서는 `menu_code`, `menu_name`, `menu_price`, `category_code`, `orderable_status`

## 1. SQL 명령어 분류

| 분류 | 의미 | 명령어 |
| :--- | :--- | :--- |
| **DQL** (Data Query Language) | 데이터 조회 | `SELECT` |
| **DML** (Data Manipulation Language) | 데이터 추가·수정·삭제 | `INSERT`, `UPDATE`, `DELETE`, `REPLACE` |
| **DDL** (Data Definition Language) | 테이블 구조 정의·변경 | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |

---

## 2. DML

### INSERT

* **개념**: 테이블에 **새로운 행**을 추가함.
* 컬럼을 지정하지 않으면 테이블의 **모든 컬럼 순서대로** 값을 넣어야 함. `AUTO_INCREMENT` 컬럼(`menu_code`)에는 `NULL`을 주면 번호가 자동으로 채워짐.

```sql
-- 모든 컬럼에 순서대로 값 입력
INSERT INTO tbl_menu VALUES (NULL, '순대국밥', 9000, 4, 'Y');

-- 넣을 컬럼을 지정 (지정한 순서대로 값을 맞추면 됨)
INSERT INTO tbl_menu (menu_name, menu_price, orderable_status, category_code)
VALUES ('초콜릿', 5000, 'Y', 7);

-- 한 번에 여러 행 추가
INSERT INTO tbl_menu
VALUES
    (NULL, '바닐라아이스크림', 2000, 12, 'Y'),
    (NULL, '딸기아이스크림', 1000, 11, 'Y'),
    (NULL, '녹차아이스크림', 1700, 8, 'Y');
```

### UPDATE

* **개념**: 이미 존재하는 행의 컬럼 값을 수정함. `SET`에 **바꿀 내용**, `WHERE`에 **바꿀 대상**을 작성함.
* 실행 전에 같은 `WHERE` 조건으로 `SELECT` 해서 대상을 먼저 확인하는 습관을 들임.

```sql
-- 1. 대상 확인
SELECT menu_code, menu_name, category_code
FROM tbl_menu
WHERE menu_name = '순대국밥';

-- 2. 수정
UPDATE tbl_menu
SET category_code = 7     -- 바꿀 내용
WHERE menu_code = 22;     -- 바꿀 대상
```

> ⚠️ `WHERE`를 빠뜨리면 테이블의 **모든 행**이 수정됨.

### DELETE

* **개념**: 테이블에서 특정 행을 삭제함.

```sql
SELECT * FROM tbl_menu WHERE menu_code = 22;   -- 대상 확인

DELETE FROM tbl_menu
WHERE menu_code = 22;
```

> ⚠️ `UPDATE`와 마찬가지로 `WHERE`가 없으면 **모든 행**이 삭제됨.

### REPLACE

* **개념**: `INSERT` 시 `PRIMARY KEY`나 `UNIQUE` 값이 충돌할 수 있을 때 사용함. 같은 키의 데이터가 있으면 **기존 행을 삭제(DELETE)한 뒤 새 행을 INSERT** 하여 덮어씀.

```sql
-- menu_code 17이 이미 있으면 PRIMARY KEY 중복 에러 발생
INSERT INTO tbl_menu VALUES (17, '참기름', 5000, 10, 'Y');

-- 기존 17번 행을 지우고 새 데이터로 저장
REPLACE INTO tbl_menu VALUES (17, '참기름', 5000, 10, 'Y');
```

---

## 3. DDL

### CREATE TABLE

```sql
CREATE TABLE 테이블명 (
    컬럼명 데이터타입(길이),
    컬럼명 데이터타입(길이),
    ...
) ENGINE = INNODB;
```

* `IF NOT EXISTS`: 같은 이름의 테이블이 **없을 때만** 생성함 (이미 있어도 에러가 나지 않음).
* `DESCRIBE 테이블명`: 테이블의 컬럼 구조(타입, NULL 허용, 키, 기본값 등)를 확인함.

```sql
CREATE TABLE IF NOT EXISTS tb1 (
    pk INT,
    fk INT,
    col1 VARCHAR(255)
) ENGINE = INNODB;

DESCRIBE tb1;
```

### AUTO_INCREMENT

* **개념**: `INSERT` 시 `PRIMARY KEY` 컬럼에 번호를 **자동으로** 발생시켜 저장함. 해당 컬럼에 `NULL`을 넣으면 다음 번호가 채워짐.

```sql
CREATE TABLE IF NOT EXISTS tb2 (
    pk INT AUTO_INCREMENT PRIMARY KEY,
    fk INT,
    col1 VARCHAR(255)
) ENGINE = INNODB;

INSERT INTO tb2 VALUES (NULL, 10, 'Y');   -- pk = 1
INSERT INTO tb2 VALUES (NULL, 20, 'N');   -- pk = 2
```

### ALTER

* **개념**: 이미 만든 테이블의 구조를 추가·변경·삭제할 때 사용함.

| 구문 | 역할 |
| :--- | :--- |
| `ADD 컬럼 타입` | 새 컬럼 추가 |
| `DROP COLUMN 컬럼` | 컬럼 삭제 |
| `CHANGE COLUMN 기존이름 새이름 타입` | **이름까지 포함**해 컬럼 정의 변경 |
| `MODIFY 컬럼 타입` | **이름은 유지**하고 정의(타입 등)만 변경 |

```sql
ALTER TABLE tb2 ADD col2 INT;                         -- 컬럼 추가
ALTER TABLE tb2 DROP COLUMN col2;                     -- 컬럼 삭제
ALTER TABLE tb2 CHANGE COLUMN fk change_fk BIGINT;    -- 이름 + 타입 변경
ALTER TABLE tb2 MODIFY change_fk INT;                 -- 타입만 변경
```

### DROP / TRUNCATE

* **DROP TABLE**: 테이블 **자체(구조 + 데이터)** 를 삭제함. `IF EXISTS`로 테이블이 있을 때만 삭제할 수 있음.
* **TRUNCATE TABLE**: 테이블 **구조는 남기고 데이터만** 비워 초기화함. `AUTO_INCREMENT` 번호도 초기화되어 다시 1부터 시작함.

```sql
DROP TABLE IF EXISTS tb3;

INSERT INTO tb4 VALUES (NULL, 10, 'Y'), (NULL, 20, 'N');   -- pk 1, 2
TRUNCATE TABLE tb4;
INSERT INTO tb4 VALUES (NULL, 10, 'Y'), (NULL, 20, 'N');   -- 다시 pk 1, 2
```

| 구분 | 삭제 대상 | 구조 | AUTO_INCREMENT |
| :--- | :--- | :--- | :--- |
| `DELETE` (DML) | `WHERE` 조건에 맞는 행 | 유지 | 유지 |
| `TRUNCATE` (DDL) | 모든 행 | 유지 | 초기화 |
| `DROP` (DDL) | 테이블 전체 | 삭제 | - |

---

## 4. 제약조건 (Constraints)

* **개념**: 테이블에 저장될 데이터에 규칙을 걸어 **잘못된 값이 들어오지 못하게** 막음. 위반하면 `INSERT`/`UPDATE`가 에러로 거부됨.

| 제약조건 | 의미 |
| :--- | :--- |
| `NOT NULL` | `NULL` 허용 안 함 |
| `UNIQUE` | 중복 값 허용 안 함 |
| `PRIMARY KEY` | 행을 구분하는 식별자 (`NOT NULL` + `UNIQUE`) |
| `FOREIGN KEY` | 다른 테이블의 값을 참조해 관계를 맺음 |
| `CHECK` | 들어올 수 있는 값의 범위나 조건을 지정 |
| `DEFAULT` | 값을 주지 않았을 때 채워질 기본값 |

### NOT NULL

```sql
CREATE TABLE IF NOT EXISTS user_notnull (
    user_no INT NOT NULL,
    user_id VARCHAR(255) NOT NULL,
    user_pwd VARCHAR(255) NOT NULL,
    gender VARCHAR(3) NOT NULL
) ENGINE = INNODB;

INSERT INTO user_notnull VALUES (1, 'nuser01', 'pass01', '남');   -- 성공
INSERT INTO user_notnull VALUES (2, 'nuser02', NULL, '여');       -- 에러: user_pwd는 NULL 불가
```

### UNIQUE

* 제약조건은 컬럼 옆에 쓰는 **컬럼 레벨**, 컬럼 정의 아래에 따로 쓰는 **테이블 레벨** 두 가지 방식으로 지정할 수 있음.

```sql
CREATE TABLE IF NOT EXISTS user_unique (
    user_no INT NOT NULL UNIQUE,     -- 컬럼 레벨
    user_id VARCHAR(255) NOT NULL,
    user_pwd VARCHAR(255) NOT NULL,
    gender VARCHAR(3),
    UNIQUE (user_id)                 -- 테이블 레벨
) ENGINE = INNODB;

INSERT INTO user_unique VALUES (1, 'nuser01', 'pass01', '남');   -- 성공
INSERT INTO user_unique VALUES (1, 'nuser02', 'pass02', '여');   -- 에러: user_no 1 중복
INSERT INTO user_unique VALUES (3, 'nuser03', 'pass01', '남');   -- 성공 (pwd는 UNIQUE가 아님)
```

### PRIMARY KEY

* 테이블에서 **한 행을 구분하는 식별자** 역할을 하며, `NOT NULL` + `UNIQUE`의 의미를 가짐.

```sql
CREATE TABLE IF NOT EXISTS user_primarykey (
    user_no INT,
    user_id VARCHAR(255) NOT NULL,
    user_pwd VARCHAR(255) NOT NULL,
    gender VARCHAR(3),
    PRIMARY KEY (user_no)
) ENGINE = INNODB;

INSERT INTO user_primarykey VALUES (1, 'nuser01', 'pass01', '남');   -- 성공
INSERT INTO user_primarykey VALUES (1, 'nuser02', 'pass02', '여');   -- 에러: PK 중복
INSERT INTO user_primarykey VALUES (2, 'nuser02', 'pass02', '여');   -- 성공
```

### FOREIGN KEY

* 두 테이블을 연결해 관계를 맺어줌. 참조하는 테이블(부모)에 **존재하는 값만** 사용할 수 있고, 그 외에는 `NULL`만 허용됨.

```sql
-- 부모 테이블
CREATE TABLE IF NOT EXISTS user_grade (
    grade_code INT PRIMARY KEY,
    grade_name VARCHAR(255) NOT NULL
) ENGINE = INNODB;

INSERT INTO user_grade VALUES (10, '일반회원'), (20, '우수회원'), (30, '특별회원');

-- 자식 테이블
CREATE TABLE IF NOT EXISTS user_foreignkey1 (
    user_no INT PRIMARY KEY,
    grade_code INT,
    FOREIGN KEY (grade_code) REFERENCES user_grade (grade_code)
) ENGINE = INNODB;

INSERT INTO user_foreignkey1 VALUES (1, 10);   -- 성공
INSERT INTO user_foreignkey1 VALUES (2, 50);   -- 에러: user_grade에 50이 없음
INSERT INTO user_foreignkey1 VALUES (2, 20);   -- 성공
```

### ON UPDATE / ON DELETE

* 부모 테이블의 값이 **바뀌거나 삭제될 때** 자식 테이블의 값을 어떻게 처리할지 정하는 옵션임.

| 옵션 | 동작 |
| :--- | :--- |
| (지정 안 함) | 참조 중인 자식이 있으면 부모 값 변경·삭제를 **막음** |
| `SET NULL` | 부모가 바뀌거나 사라지면 자식의 값을 `NULL`로 변경 |
| `CASCADE` | 부모가 바뀌면 자식도 따라 바뀌고, 부모가 삭제되면 자식도 함께 삭제 |

```sql
CREATE TABLE IF NOT EXISTS user_foreignkey2 (
    user_no INT PRIMARY KEY,
    grade_code INT,
    FOREIGN KEY (grade_code) REFERENCES user_grade (grade_code)
        ON UPDATE SET NULL
        ON DELETE SET NULL
) ENGINE = INNODB;

INSERT INTO user_foreignkey2 VALUES (1, 20), (2, 20), (3, 30);

-- 옵션 없이 grade_code 10을 참조하던 user_foreignkey1을 먼저 삭제해야
-- 부모의 10을 수정할 수 있음
DROP TABLE IF EXISTS user_foreignkey1;

UPDATE user_grade
SET grade_code = 40
WHERE grade_code = 10;
```

### CHECK

* 컬럼에 들어올 수 있는 값의 **조건**을 직접 지정함.

```sql
CREATE TABLE IF NOT EXISTS user_check (
    user_no INT AUTO_INCREMENT PRIMARY KEY,
    user_name VARCHAR(255) NOT NULL,
    gender VARCHAR(3) CHECK (gender IN ('남', '여')),
    age INT CHECK (age >= 19)
) ENGINE = INNODB;

INSERT INTO user_check VALUES (NULL, '홍길동', '남', 25);   -- 성공
INSERT INTO user_check VALUES (NULL, '판다', '여', 5);      -- 에러: age >= 19 위반
INSERT INTO user_check VALUES (NULL, '판다', '여자', 20);   -- 에러: gender 조건 위반
```

### DEFAULT

* `INSERT` 시 값을 주지 않거나 `DEFAULT`를 넣으면 지정한 기본값이 채워짐. 함수 결과를 기본값으로 쓸 때는 `DEFAULT (current_date)`처럼 괄호로 감쌈.

```sql
CREATE TABLE IF NOT EXISTS tbl_country (
    country_code INT AUTO_INCREMENT PRIMARY KEY,
    country_name VARCHAR(255) DEFAULT '한국',
    add_day DATE DEFAULT (current_date),
    add_time TIME DEFAULT (current_time)
) ENGINE = INNODB;

INSERT INTO tbl_country VALUES (NULL, DEFAULT, DEFAULT, DEFAULT);  -- 한국, 오늘 날짜, 현재 시간
INSERT INTO tbl_country (country_name) VALUES ('미국');             -- 미국, 오늘 날짜, 현재 시간
```

---

## 5. 형변환 (CAST / CONVERT)

### 명시적 형변환

* `CAST(값 AS 타입)` 또는 `CONVERT(값, 타입)`으로 값을 원하는 타입으로 바꿈.
* `SIGNED INTEGER`: 부호가 있는 정수(음수, 0, 양수). `INTEGER`는 생략 가능.

```sql
SELECT AVG(menu_price) AS '변환 전 평균' FROM tbl_menu;   -- 소수점이 붙은 값

SELECT CAST(AVG(menu_price) AS SIGNED INTEGER) AS '메뉴 평균 가격' FROM tbl_menu;
SELECT CONVERT(AVG(menu_price), SIGNED INTEGER) AS '메뉴 평균 가격' FROM tbl_menu;

SELECT CAST('2026-10-21' AS DATE);   -- 문자열 → 날짜

-- 숫자 → 문자로 바꾼 뒤 '원'을 붙임
SELECT
    menu_name,
    CONCAT(CAST(menu_price AS CHAR(5)), '원') AS '메뉴가격'
FROM tbl_menu;
```

### 암시적 형변환

* 타입이 다른 값끼리 연산하면 MySQL이 **자동으로** 타입을 맞춰 계산함.

```sql
SELECT '1' + '2';   -- 3   : 문자열이 숫자로 변환되어 더해짐
SELECT 3 > 'MAY';   -- 1   : 숫자로 바꿀 수 없는 'MAY'는 0으로 취급되어 3 > 0 → 참
```

> 암시적 형변환은 의도치 않은 결과를 낼 수 있으므로, 타입이 다른 값은 `CAST`로 명시적으로 바꿔 쓰는 편이 안전함.

---

## 📝 오늘의 정리

1. SQL은 조회(DQL), 데이터 조작(DML), 구조 정의(DDL)로 나뉜다.
2. `UPDATE`·`DELETE`는 실행 전에 같은 `WHERE`로 `SELECT` 해서 대상을 먼저 확인한다.
3. `DELETE`는 행을 지우고, `TRUNCATE`는 테이블을 초기화하며(`AUTO_INCREMENT`도 리셋), `DROP`은 테이블 자체를 삭제한다.
4. 제약조건(`NOT NULL`·`UNIQUE`·`PRIMARY KEY`·`FOREIGN KEY`·`CHECK`·`DEFAULT`)으로 잘못된 데이터가 들어오는 것을 DB 단에서 막는다.
5. 타입이 다른 값은 암시적 형변환에 맡기지 말고 `CAST`/`CONVERT`로 명시적으로 바꾼다.

---

## 느낀 점

* `WHERE` 없이 `UPDATE`·`DELETE`를 실행하면 전체 행이 바뀐다는 점이 가장 무섭게 느껴졌고, 먼저 `SELECT`로 확인하는 습관을 들여야겠다고 생각함.
* 제약조건을 걸어두면 애플리케이션 코드에서 실수해도 DB가 한 번 더 막아준다는 점에서 테이블 설계 단계가 중요하다고 느낌.
