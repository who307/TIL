# SQL DML · DDL · 제약조건 · 형변환 정리 (TIL)

> 실습 환경: MySQL (`menudb`), `tbl_menu` 컬럼 순서는 `menu_code`, `menu_name`, `menu_price`, `category_code`, `orderable_status`

## 1. SQL 명령어 분류

* SQL 명령어는 **무엇을 다루느냐**에 따라 나뉨. 집에 비유하면 DQL은 **방 안을 둘러보기**, DML은 **가구 넣고 빼기**, DDL은 **방 자체를 만들고 허물기**임.

| 분류 | 의미 | 명령어 |
| :--- | :--- | :--- |
| **DQL** (Data Query Language) | 데이터 조회 | `SELECT` |
| **DML** (Data Manipulation Language) | 데이터 추가·수정·삭제 | `INSERT`, `UPDATE`, `DELETE`, `REPLACE` |
| **DDL** (Data Definition Language) | 테이블 구조 정의·변경 | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |

---

## 2. DML

### INSERT

* **개념**: 표에 **새 행(줄)** 을 추가함.
* 열 이름을 안 쓰면 표의 **모든 열에 순서대로** 값을 넣어야 함. 열 이름을 쓰면 그 열들만, **쓴 순서대로** 값을 맞추면 됨.
* `AUTO_INCREMENT` 열(`menu_code`)에 `NULL`을 넣으면 DB가 **다음 번호를 자동으로** 채워줌.

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

* **따라가기**
  * 첫 번째 → 열 5개 순서(코드, 이름, 가격, 카테고리, 주문가능)대로 → 코드는 자동 번호(예: 22), `순대국밥 | 9000 | 4 | Y`
  * 두 번째 → 열을 내가 정한 순서(이름, 가격, **주문가능, 카테고리**)로 썼으니 값도 그 순서 → `초콜릿, 5000, 'Y', 7`. 코드는 안 썼으므로 자동 번호.
  * 세 번째 → 괄호 묶음을 쉼표로 이어 **3줄을 한 번에** 추가.

### UPDATE

* **개념**: 이미 있는 행의 값을 **고침**. `SET`에 **무엇을 어떻게 바꿀지**, `WHERE`에 **어느 행을 바꿀지**를 씀.
* 실행 전에 같은 `WHERE` 조건으로 `SELECT`를 먼저 해서 **대상이 맞는지 확인**하는 습관을 들임.

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

* **따라가기**: ① 순대국밥을 조회 → 코드가 22, 카테고리 4임을 확인 ② 22번 행만 카테고리를 7로 바꿈.

> ⚠️ `WHERE`를 빠뜨리면 테이블의 **모든 행**이 수정됨. 모든 메뉴의 카테고리가 7이 되어버림.

### DELETE

* **개념**: 표에서 행을 **지움**.

```sql
SELECT * FROM tbl_menu WHERE menu_code = 22;   -- 대상 확인

DELETE FROM tbl_menu
WHERE menu_code = 22;
```

* **따라가기**: 22번 행을 먼저 눈으로 확인 → 22번 행 삭제.

> ⚠️ `UPDATE`와 마찬가지로 `WHERE`가 없으면 **모든 행**이 삭제됨.

### REPLACE

* **개념**: `INSERT`하려는데 `PRIMARY KEY`나 `UNIQUE` 값이 이미 있으면 에러가 남. `REPLACE`는 같은 키의 행이 있으면 **기존 행을 지우고(DELETE) 새 행을 넣음(INSERT)** → 덮어쓰기 효과.

```sql
-- menu_code 17이 이미 있으면 PRIMARY KEY 중복 에러 발생
INSERT INTO tbl_menu VALUES (17, '참기름', 5000, 10, 'Y');

-- 기존 17번 행을 지우고 새 데이터로 저장
REPLACE INTO tbl_menu VALUES (17, '참기름', 5000, 10, 'Y');
```

* **따라가기**: 17번이 이미 있음 → `INSERT`는 "번호 중복!" 에러 → `REPLACE`는 기존 17번을 지우고 `17 | 참기름 | 5000 | 10 | Y`를 새로 넣음. 17번이 없었다면 그냥 `INSERT`처럼 추가됨.

---

## 3. DDL

### CREATE TABLE

* **개념**: 새 **표를 만듦**. 열 이름과 그 열에 담을 **자료형**(숫자 `INT`, 글자 `VARCHAR(최대길이)`, 날짜 `DATE` 등)을 정함.

```sql
CREATE TABLE 테이블명 (
    컬럼명 데이터타입(길이),
    컬럼명 데이터타입(길이),
    ...
) ENGINE = INNODB;
```

* `ENGINE = INNODB`: 표를 저장·관리하는 방식(저장 엔진). InnoDB는 외래키, 트랜잭션을 지원하는 MySQL의 기본 엔진임.
* `IF NOT EXISTS`: 같은 이름의 표가 **없을 때만** 만듦 → 이미 있어도 에러 없이 넘어감.
* `DESCRIBE 테이블명`: 표의 **설계도**(열 이름, 타입, NULL 허용, 키, 기본값)를 보여줌.

```sql
CREATE TABLE IF NOT EXISTS tb1 (
    pk INT,
    fk INT,
    col1 VARCHAR(255)
) ENGINE = INNODB;

DESCRIBE tb1;
```

* **따라가기**: 정수 열 `pk`, `fk`, 글자 최대 255자 열 `col1`을 가진 `tb1` 표를 만듦 → `DESCRIBE`로 세 열의 정보 확인.

### AUTO_INCREMENT

* **개념**: 새 행을 넣을 때 `PRIMARY KEY` 열에 번호를 **1, 2, 3… 자동으로** 붙여줌. 은행 번호표 기계와 같음. 그 열에 `NULL`을 넣으면 다음 번호가 들어감.

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

* **개념**: 이미 만든 표의 **구조(열)** 를 추가·변경·삭제함. 방에 창문을 새로 내거나 막는 리모델링과 같음.

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

* **따라가기** (`tb2`의 열 변화)
  1. 처음 → `pk, fk, col1`
  2. `ADD col2 INT` → `pk, fk, col1, col2`
  3. `DROP COLUMN col2` → `pk, fk, col1`
  4. `CHANGE COLUMN fk change_fk BIGINT` → `fk`의 이름을 `change_fk`로, 타입을 큰 정수 `BIGINT`로 → `pk, change_fk, col1`
  5. `MODIFY change_fk INT` → 이름은 그대로, 타입만 `INT`로.

### DROP / TRUNCATE

* **DROP TABLE**: 표 **자체(구조 + 데이터)** 를 없앰. 방을 통째로 허무는 것. `IF EXISTS`를 붙이면 표가 있을 때만 지움.
* **TRUNCATE TABLE**: 표 **구조는 남기고 데이터만** 싹 비움. 방은 두고 가구만 다 빼는 것. `AUTO_INCREMENT` 번호표도 **1부터 다시** 시작함.

```sql
DROP TABLE IF EXISTS tb3;

INSERT INTO tb4 VALUES (NULL, 10, 'Y'), (NULL, 20, 'N');   -- pk 1, 2
TRUNCATE TABLE tb4;
INSERT INTO tb4 VALUES (NULL, 10, 'Y'), (NULL, 20, 'N');   -- 다시 pk 1, 2
```

* **따라가기**: `tb4`에 두 줄 넣음 → pk 1, 2 → `TRUNCATE`로 비움 → 다시 두 줄 넣으면 번호가 3, 4가 아니라 **1, 2**부터.

| 구분 | 삭제 대상 | 구조 | AUTO_INCREMENT |
| :--- | :--- | :--- | :--- |
| `DELETE` (DML) | `WHERE` 조건에 맞는 행 | 유지 | 유지 |
| `TRUNCATE` (DDL) | 모든 행 | 유지 | 초기화 |
| `DROP` (DDL) | 테이블 전체 | 삭제 | - |

---

## 4. 제약조건 (Constraints)

* **개념**: 표에 들어올 데이터에 **규칙**을 걸어서 **잘못된 값을 DB가 직접 막게** 함. 놀이기구 입구의 "키 120cm 이상" 표지판과 같음. 규칙을 어기면 `INSERT`/`UPDATE`가 에러로 거부됨.

| 제약조건 | 의미 |
| :--- | :--- |
| `NOT NULL` | `NULL` 허용 안 함 |
| `UNIQUE` | 중복 값 허용 안 함 |
| `PRIMARY KEY` | 행을 구분하는 식별자 (`NOT NULL` + `UNIQUE`) |
| `FOREIGN KEY` | 다른 테이블의 값을 참조해 관계를 맺음 |
| `CHECK` | 들어올 수 있는 값의 범위나 조건을 지정 |
| `DEFAULT` | 값을 주지 않았을 때 채워질 기본값 |

### NOT NULL

* 그 열은 **반드시 값이 있어야** 함. 회원가입에서 "필수 입력" 칸과 같음.

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

* 그 열에 **같은 값이 두 번** 들어올 수 없음. 아이디 중복 금지와 같음.
* 제약조건을 쓰는 위치는 두 가지: 열 바로 옆에 쓰는 **컬럼 레벨**, 열 정의를 다 쓴 뒤 아래에 따로 쓰는 **테이블 레벨**.

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

* **따라가기**: 1번 성공 → 또 `user_no = 1` → 중복이라 에러 → `user_no = 3`, 비밀번호 `pass01`은 1번과 같지만 비밀번호 열엔 UNIQUE가 없어서 성공.

### PRIMARY KEY

* 표에서 **한 행을 콕 집어 구분하는 대표 번호**. 학생의 **학번**과 같음. `NOT NULL` + `UNIQUE`의 의미를 함께 가짐(비어 있으면 안 되고, 겹치면 안 됨). 표 하나에 하나만 둠.

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

* 두 표를 **연결**해줌. 자식 표의 열 값은 **부모 표에 실제로 있는 값만** 쓸 수 있음(또는 열이 NULL을 허용하면 `NULL`). 학생 명부의 "반 번호"가 실제로 있는 반이어야 하는 것과 같음. 없는 7반을 적을 수 없음.

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

* **따라가기**: 부모 등급은 10, 20, 30뿐 → 10은 있음(성공) → 50은 없음(에러) → 20은 있음(성공).

### ON UPDATE / ON DELETE

* 부모 값이 **바뀌거나 지워질 때** 그걸 가리키던 자식 값을 어떻게 할지 정하는 옵션.

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

* **따라가기**
  1. `user_foreignkey2`는 `SET NULL` 옵션으로 만들고 20, 20, 30을 가리키는 회원 3명을 넣음.
  2. 부모의 10을 40으로 바꾸려는데, 옵션 없이 만든 `user_foreignkey1`이 10을 가리키고 있어서 **막힘** → 그 표를 먼저 지움.
  3. `UPDATE`로 10 → 40 변경 성공.
  4. 만약 20을 바꿨다면 → `user_foreignkey2`에서 20을 가리키던 1, 2번 회원의 `grade_code`가 `NULL`로 바뀜.

### CHECK

* 열에 들어올 값의 **조건**을 직접 정함.

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

* **따라가기**: 성별은 '남'/'여'만, 나이는 19 이상만 → 25세 남(성공) → 5세(나이 위반) → '여자'(성별 목록에 없음).

### DEFAULT

* 값을 안 주거나 `DEFAULT`라고 쓰면 **미리 정한 기본값**이 들어감. 함수 결과(오늘 날짜 등)를 기본값으로 쓸 때는 `DEFAULT (current_date)`처럼 **괄호**로 감쌈.

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

* **따라가기**: 첫 줄 → 모든 칸에 `DEFAULT` → `1 | 한국 | 2026-10-21 | 14:00:00` / 둘째 줄 → 나라 이름만 줌 → 나머지는 기본값 → `2 | 미국 | 2026-10-21 | 14:00:01`

---

## 5. 형변환 (CAST / CONVERT)

### 명시적 형변환

* `CAST(값 AS 타입)` 또는 `CONVERT(값, 타입)`으로 값을 **내가 원하는 타입으로** 바꿈.
* `SIGNED INTEGER`: 부호(+, -)가 있는 정수(음수, 0, 양수). `INTEGER`는 생략 가능.

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

* **따라가기**
  * `AVG(menu_price)` → 예: `10833.3333`
  * `CAST(... AS SIGNED INTEGER)` / `CONVERT(..., SIGNED INTEGER)` → 정수로 → `10833` (MySQL은 반올림함)
  * `CAST('2026-10-21' AS DATE)` → 글자를 날짜 타입으로.
  * `CAST(menu_price AS CHAR(5))` → 가격 9000을 글자 `'9000'`으로 → `CONCAT`으로 `'원'`을 붙임 → `9000원`
  * ⚠️ `CHAR(5)`는 **최대 5글자**라서, 가격이 100000원처럼 6자리 이상이면 `10000`까지만 남고 **잘림**. 큰 값이 있을 수 있으면 길이를 넉넉하게 줘야 함.

### 암시적 형변환

* 타입이 다른 값끼리 계산하면 MySQL이 **몰래 알아서** 타입을 맞춰 계산함.

```sql
SELECT '1' + '2';   -- 3   : 문자열이 숫자로 변환되어 더해짐
SELECT 3 > 'MAY';   -- 1   : 숫자로 바꿀 수 없는 'MAY'는 0으로 취급되어 3 > 0 → 참
```

* **따라가기**
  * `'1' + '2'` → SQL의 `+`는 **숫자 더하기만** 함 → 둘 다 숫자로 바꿈 → `3` (자바스크립트처럼 `'12'`가 되지 않음)
  * `3 > 'MAY'` → `'MAY'`를 숫자로 바꾸려는데 못 바꿈 → `0` → `3 > 0` → 참 → `1`

> 암시적 형변환은 의도치 않은 결과를 낼 수 있으므로, 타입이 다른 값은 `CAST`로 명시적으로 바꿔 쓰는 편이 안전함.

---

## 📝 오늘의 정리

1. SQL은 조회(DQL), 데이터 조작(DML), 구조 정의(DDL)로 나뉨
2. `UPDATE`·`DELETE`는 실행 전에 같은 `WHERE`로 `SELECT` 해서 대상을 먼저 확인함
3. `DELETE`는 행을 지우고, `TRUNCATE`는 테이블을 초기화하며(`AUTO_INCREMENT`도 리셋), `DROP`은 테이블 자체를 삭제함
4. 제약조건(`NOT NULL`·`UNIQUE`·`PRIMARY KEY`·`FOREIGN KEY`·`CHECK`·`DEFAULT`)으로 잘못된 데이터가 들어오는 것을 DB 단에서 막음
5. 타입이 다른 값은 암시적 형변환에 맡기지 말고 `CAST`/`CONVERT`로 명시적으로 바꿈

---

## 느낀 점

* `WHERE` 없이 `UPDATE`·`DELETE`를 실행하면 전체 행이 바뀐다는 점이 가장 무섭게 느껴졌고, 먼저 `SELECT`로 확인하는 습관을 들여야겠다고 생각함
* 제약조건을 걸어두면 애플리케이션 코드에서 실수해도 DB가 한 번 더 막아준다는 점에서 테이블 설계 단계가 중요하다고 느낌
