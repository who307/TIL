# REST API 응답 설계 · Bean Validation 에러 코드 · Swagger 문서화 정리 (TIL)

> 실습 환경: Spring Boot 4 + Spring Web MVC (`chap08-rest-api`), Bean Validation, springdoc-openapi 3 (Swagger UI), Lombok, IntelliJ HTTP Client (`api-test.http`)

## 0. REST API가 뭔가

* **한 줄 비유**: 도서관에서 책을 다룰 때 "**무엇을**(책 번호)"과 "**어떻게**(빌리기·반납·등록·폐기)"를 따로 말하는 것과 같음. 주소(URI)는 **무엇을**, HTTP 메소드는 **어떻게**를 맡음.
* **REST 규칙 몇 가지**
  * 주소에는 **명사(자원)** 를 씀. 동사는 쓰지 않음 → `/users` (O), `/getUsers`, `/deleteUser` (X)
  * 하는 일은 **HTTP 메소드**로 구분함.
  * 여러 개를 뜻하므로 **복수형**(`users`)을 씀.
  * 주소 앞에 `/api/v1`처럼 **버전**을 붙여 두면, 나중에 규칙을 바꿀 때 `/api/v2`를 새로 열고 기존 사용자는 `v1`을 계속 쓸 수 있음.

| 하고 싶은 일 | 메소드 + 주소 | 성공 응답 |
| :--- | :--- | :--- |
| 회원 목록 조회 / 검색 | `GET /api/v1/response/users?status=active&name=고릴라` | `200 OK` + 목록 |
| 회원 한 명 조회 | `GET /api/v1/response/users/1` | `200 OK` + 회원 |
| 회원 등록 | `POST /api/v1/response/users` | `201 Created` + `Location` 헤더 |
| 회원 수정 | `PUT /api/v1/response/users/1` | `204 No Content` |
| 회원 삭제 | `DELETE /api/v1/response/users/1` | `204 No Content` |

* **query string vs path variable**
  * `/users/1` (path variable) → "**1번 회원**"처럼 자원 **하나를 콕 집을 때**.
  * `/users?status=active` (query string) → "활성 회원**만 골라서**"처럼 목록을 **거르거나 검색할 때**. 안 붙이면 전체 목록.

---

## 1. 응답 설계 심화: ResponseEntity

### 요청 DTO와 응답 DTO를 나누는 이유

* **한 줄 비유**: 은행 창구에 내는 **신청서**(비밀번호 칸 있음)와 은행이 돌려주는 **통장 사본**(비밀번호 칸 없음)은 다른 종이임.
* `UserRequestDTO` → 클라이언트가 **보내는** 값: `id`, `pwd`, `name`, `status`
* `UserResponseDTO` → 서버가 **돌려주는** 값: `no`, `id`, `name`, `enrollDate`, `status` → **비밀번호가 없음**
* 하나의 DTO를 같이 쓰면, 실수로 `pwd`까지 응답 JSON에 담겨 **비밀번호가 밖으로 새는** 일이 생길 수 있음. 아예 응답용 클래스에 `pwd` 칸을 만들지 않으면 그럴 일이 없음.

```java
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@ToString(exclude = "pwd")
public class UserRequestDTO {

    /* 클라이언트가 회원 등록·수정 요청으로 보내는 값만 담는다. */
    private String id;
    private String pwd;
    private String name;
    private String status;
}
```

* `@ToString(exclude = "pwd")` → Lombok이 만들어 주는 `toString()`에서 `pwd`를 **빼라**는 뜻. 디버깅하려고 `System.out.println(request)`나 로그를 찍어도 `UserRequestDTO(id=user04, name=신규회원, status=active)`처럼 **비밀번호가 찍히지 않음**. 로그 파일은 여러 사람이 보므로 비밀번호가 남으면 위험함.
* `@NoArgsConstructor` → Jackson이 JSON을 DTO로 바꿀 때 **빈 객체를 먼저 만들고** setter로 값을 채우기 때문에 필요함.

### 응답 모양을 통일하는 ResponseMessage

```java
@AllArgsConstructor
@Getter
@Setter
@ToString
public class ResponseMessage {

    private int httpStatus;
    private String message;
    private Map<String, Object> results;
}
```

* **한 줄 비유**: 회사에서 보내는 공문은 항상 **"상태 / 제목 / 내용"** 칸이 같은 양식임. 받는 쪽은 매번 어디에 뭐가 있는지 고민할 필요가 없음.
* 모든 조회 응답이 `{"httpStatus":200, "message":"조회 성공", "results":{...}}` 모양이 되므로, 프론트엔드는 항상 `results` 안만 보면 됨.
* `results`를 `Map`으로 둔 이유 → 목록일 때는 `"users"`, 한 명일 때는 `"user"`처럼 **이름표를 붙여** 담을 수 있음.

### 회원 목록은 어디에 저장되나

```java
@RestController
@RequestMapping("/api/v1/response/users")
public class ResponseEntityTestController {

    private final List<UserResponseDTO> users;
    private int nextUserNo = 4;

    public ResponseEntityTestController() {
        users = new ArrayList<>();

        users.add(new UserResponseDTO(1, "user01", "고릴라", new Date(), "active"));
        users.add(new UserResponseDTO(2, "user02", "호랑이", new Date(), "active"));
        users.add(new UserResponseDTO(3, "user03", "원숭이", new Date(), "inactive"));
    }
    ...
}
```

* 아직 DB를 연결하지 않아서 **Controller 안의 `List`를 임시 DB처럼** 씀.
* Controller는 스프링 Bean이고 기본 scope가 **singleton**(10/26 TIL)이라 객체가 **딱 하나**만 만들어짐 → 요청이 여러 번 와도 같은 `users` 리스트를 씀 → `POST`로 추가한 회원이 다음 `GET`에서도 보임.
* 서버를 껐다 켜면 생성자가 다시 실행되어 **처음 3명으로 돌아감**(메모리에만 있으니까).
* `nextUserNo = 4` → 다음에 등록될 회원 번호. 등록할 때마다 1씩 늘림.

### 목록 조회 + 조건 검색: 200 OK

```java
@GetMapping
public ResponseEntity<ResponseMessage> findUserByConditions(
        @RequestParam(required = false) String name,
        @RequestParam(required = false) String status) {

    List<UserResponseDTO> foundUsers = users.stream()
            .filter(user -> {
                if (status != null && !user.getStatus().equals(status)) {
                    return false;
                }
                if (name != null && !user.getName().contains(name)) {
                    return false;
                }
                return true;
            })
            .collect(Collectors.toList());

    Map<String, Object> responseMap = new HashMap<>();
    responseMap.put("users", foundUsers);

    ResponseMessage responseMessage = new ResponseMessage(
            HttpStatus.OK.value(),
            "조회 성공",
            responseMap
    );
    return new ResponseEntity<>(responseMessage, HttpStatus.OK);
}
```

* **필터의 규칙**: "조건이 **주어졌는데** 안 맞으면 탈락, 나머지는 통과"
  * `status != null && ...` → `status`를 **안 보냈으면**(`null`) 이 검사는 건너뜀 → 상태와 상관없이 통과.
  * `name`은 `equals`가 아니라 `contains` → **일부만 맞아도** 통과(검색창 느낌). `name=원`이면 `원숭이`가 걸림.
* **따라가기** (`GET /api/v1/response/users?status=active&name=고릴라`)
  1. `status = "active"`, `name = "고릴라"`
  2. 1번 고릴라 → 상태 `active` 맞음 → 이름에 `고릴라` 포함 → **통과**
  3. 2번 호랑이 → 상태 맞음 → 이름에 `고릴라` 없음 → **탈락**
  4. 3번 원숭이 → 상태 `inactive`라 첫 번째 검사에서 **탈락**
  5. 응답 `200 OK` → `{"httpStatus":200,"message":"조회 성공","results":{"users":[{"no":1,"id":"user01","name":"고릴라",...}]}}`
* **따라가기** (`GET /api/v1/response/users`) → 둘 다 `null` → 모든 검사를 건너뜀 → **3명 모두** 응답.
* `new ResponseEntity<>(body, HttpStatus.OK)` → 응답 상자를 **생성자로 직접** 만드는 방법. `HttpStatus.OK.value()`는 상태를 숫자 `200`으로 꺼낸 것.

### 단건 조회: ok().body()

```java
@GetMapping("/{userNo}")
public ResponseEntity<ResponseMessage> findUserByNo(@PathVariable int userNo) {

    UserResponseDTO foundUser = users.stream()
            .filter(user -> user.getNo() == userNo)
            .findFirst()
            .get();

    Map<String, Object> responseMap = new HashMap<>();
    responseMap.put("user", foundUser);

    ResponseMessage responseMessage = new ResponseMessage(
            HttpStatus.OK.value(),"조회 성공", responseMap
    );

    return ResponseEntity
            .ok()
            .body(responseMessage);
}
```

* `.findFirst()` → 조건에 맞는 첫 번째를 `Optional`(**있을 수도 없을 수도 있는 상자**)에 담아 줌 → `.get()`으로 꺼냄.
* `ResponseEntity.ok().body(...)` → **빌더(builder) 방식**. "상태는 200으로 하고(`ok()`), 내용은 이걸로(`body()`)"를 한 줄씩 이어서 조립함. `ResponseEntity.ok(responseMessage)`로 줄여 써도 같음.

> ⚠️ `.get()`은 상자가 **비어 있으면** `NoSuchElementException`을 던짐. `GET /users/999`처럼 없는 번호를 보내면 처리하는 핸들러가 없어서 **`500 Internal Server Error`** 가 나감. "없다"는 서버 고장이 아니라 `404`가 맞음 → 아래 Swagger 실습(3장)에서는 `orElseThrow`로 고쳐 씀.

### 등록: 201 Created + Location 헤더

```java
@PostMapping
public ResponseEntity<Void> registUser(@RequestBody UserRequestDTO request) {

    UserResponseDTO newUser = new UserResponseDTO(
            nextUserNo++,
            request.getId(),
            request.getName(),
            new Date(),
            request.getStatus()
    );
    users.add(newUser);
    // 상태를 201로 만들고, 전달한 URI를 Location header에 넣는다.
    return ResponseEntity.created(URI.create("api/v1/response/users/"
            + newUser.getNo())).build();
}
```

* **한 줄 비유**: 택배를 맡기면 "접수 완료(201)" 도장과 함께 **송장 번호(Location)** 를 줌. 나중에 그 번호로 택배를 찾아볼 수 있음.
* `ResponseEntity<Void>` → **body가 없는** 응답이라는 뜻. `Void`는 "아무것도 없음"을 나타내는 타입.
* `ResponseEntity.created(uri)` → 상태를 **201 Created**로 정하고, 응답 헤더 `Location`에 **새로 만든 자원의 주소**를 넣어 줌.
* `.build()` → body 없이 응답 상자를 **완성**. (body가 있으면 `.body(...)`로 끝냄)
* **따라가기** (`POST` + `{"id":"user04","pwd":"pass04","name":"신규회원","status":"active"}`)
  1. JSON → `UserRequestDTO`로 변환 (`pwd`도 들어오지만 아래에서 쓰지 않음)
  2. `nextUserNo++` → 지금 값 `4`를 쓰고 나서 `5`로 늘림 (후위 증가)
  3. `new Date()` → 가입일은 **서버가 지금 시각**으로 정함. 클라이언트가 마음대로 정하지 못하게.
  4. 응답 → `201 Created`, 헤더 `Location: api/v1/response/users/4`, body 없음
  5. 클라이언트는 `Location`을 보고 `GET /api/v1/response/users/4`로 방금 등록한 회원을 조회할 수 있음.

> ⚠️ 실습 코드의 `"api/v1/response/users/"`는 맨 앞 `/`가 빠져 있어서 **상대 주소**가 됨. 브라우저는 상대 주소를 "지금 요청한 주소 기준"으로 해석하므로 `/api/v1/response/api/v1/response/users/4`처럼 엉뚱한 곳을 가리킬 수 있음. `"/api/v1/response/users/"`처럼 `/`로 시작해야 함. (Swagger 실습 코드는 `/`가 붙어 있음)

### 수정·삭제: 204 No Content

```java
@PutMapping("/{userNo}")
public ResponseEntity<Void> modifyUser(
        @PathVariable int userNo,
        @RequestBody UserRequestDTO modifyInfo) {

    UserResponseDTO foundUser = users.stream()
            .filter(user -> user.getNo() == userNo)
            .findFirst()
            .get();

    foundUser.setId(modifyInfo.getId());
    foundUser.setName(modifyInfo.getName());
    foundUser.setStatus(modifyInfo.getStatus());

    // noContent()가 상태를 204로 정하고, body가 없으므로 build()로 완성
    return ResponseEntity.noContent().build();
}

@DeleteMapping("/{userNo}")
public ResponseEntity<Void> removeUser(@PathVariable int userNo) {

    UserResponseDTO foundUser = users.stream()
            .filter(user -> user.getNo() == userNo)
            .findFirst()
            .get();

    users.remove(foundUser);

    return ResponseEntity.noContent().build();
}
```

* **204 No Content** → "잘 처리했어요. 그런데 **돌려줄 내용은 없어요**." 수정·삭제는 결과물을 다시 보여줄 필요가 없는 경우가 많아서 자주 씀.
* **따라가기** (`PUT /users/1` + `{"id":"user01","pwd":"changed","name":"수정회원","status":"active"}`)
  1. 1번 고릴라를 찾음 → `foundUser`는 리스트 안 객체를 **가리키는 리모컨**(참조)
  2. setter로 이름을 `수정회원`으로 바꿈 → 리스트 안의 **그 객체 자체가** 바뀜 (따로 다시 넣을 필요 없음)
  3. 응답 `204`, body 없음 → 다시 `GET /users/1` 하면 `"name":"수정회원"`
* **PUT vs PATCH**: `PUT`은 자원을 **통째로 바꾸는** 요청이라 `id`·`name`·`status`를 모두 보냄. 일부만 바꾸고 싶을 때는 보통 `PATCH`를 씀. 위 코드에서 `name`을 빼고 보내면 `name`이 `null`로 바뀌어 버림.
* **따라가기** (`DELETE /users/1`) → 1번을 찾아 `users.remove(foundUser)` → `204` → 이제 목록에 2명만 남음.

### ResponseEntity 만드는 방법 모음

| 코드 | 상태 | body |
| :--- | :--- | :--- |
| `new ResponseEntity<>(body, HttpStatus.OK)` | 200 | 있음 |
| `ResponseEntity.ok(body)` / `ResponseEntity.ok().body(body)` | 200 | 있음 |
| `ResponseEntity.status(HttpStatus.NOT_FOUND).body(body)` | 원하는 상태 | 있음 |
| `ResponseEntity.created(uri).build()` | 201 + `Location` | 없음 |
| `ResponseEntity.noContent().build()` | 204 | 없음 |

---

## 2. Bean Validation 심화: 검증 실패 → 에러 코드

* 10/28에 `@Valid` + `@NotBlank`로 요청 값을 검사하는 기본을 배웠음. 오늘은 **검사 종류를 늘리고**, 어떤 검사에 걸렸는지에 따라 **에러 코드를 다르게** 돌려줌.

### 검사 규칙이 붙은 UserDTO

```java
@ToString(exclude = "pwd")
public class UserDTO {

    private Integer no;

    @NotBlank(message = "아이디는 반드시 입력되어야 합니다.")
    private String id;

    @NotBlank(message = "비밀번호는 반드시 입력되어야 합니다.")
    @Size(max = 10, message="비밀번호는 길이 10을 초과할 수 없습니다.")
    private String pwd;

    @NotBlank(message = "이름은 반드시 입력되어야 합니다.")
    @Size(max = 10, message="이름은 길이 10을 초과할 수 없습니다.")
    private String name;

    @NotNull(message="가입일은 반드시 입력되어야 합니다.")
    @Past(message="가입일은 현재보다 과거 날짜가 입력 되어야 합니다.")
    private Date enrollDate;
}
```

| 어노테이션 | 통과 못 하는 값 | 주로 쓰는 곳 |
| :--- | :--- | :--- |
| `@NotNull` | `null`만 막음 (`""`, `" "`는 통과) | 날짜, 숫자, 객체 |
| `@NotBlank` | `null`, `""`, `"   "`(공백만) 모두 막음 | 문자열 |
| `@Size(min, max)` | 길이가 범위를 벗어남 (`max = 10` → 11글자부터 실패) | 문자열, 컬렉션 |
| `@Past` | 지금보다 **미래이거나 지금**인 날짜 | 가입일, 생년월일 |

* 한 필드에 **여러 개를 겹쳐** 붙일 수 있음. `pwd`는 "비어 있으면 안 되고(`@NotBlank`), 10글자를 넘으면 안 됨(`@Size`)".
* `enrollDate`에 `@NotBlank`가 아니라 `@NotNull`인 이유 → `@NotBlank`는 **문자열 전용**이라 `Date`에는 쓸 수 없음.
* `@Size`는 `null`을 **통과**시킴 (검사할 길이가 없으니까). 그래서 "필수"까지 원하면 `@NotBlank`와 함께 씀.

### Controller: @Valid만 붙이면 끝

```java
@RestController
@RequestMapping("/api/v1/validation")
public class ValidTestController {

    @PostMapping("/users")
    public ResponseEntity<Void> registUser(@Valid @RequestBody UserDTO user) {
        return ResponseEntity
                .created(URI.create("/api/v1/validation/users/1"))
                .build();
    }

    @GetMapping("/users/{userNo}")
    public ResponseEntity<Void> findUserByNo(@PathVariable int userNo) {

        if(userNo != 1) {
            throw new UserNotFoundException(userNo + "번 회원 정보를 찾을 수 없습니다.");
        }

        return ResponseEntity.ok().build();
    }
}
```

* `registUser` 본문에는 검사 코드(`if`)가 **한 줄도 없음**. `@Valid`가 붙어 있어서 검사에 실패하면 본문이 실행되기 **전에** `MethodArgumentNotValidException`이 터짐 → 본문까지 왔다면 이미 **검사를 통과한 값**임.
* `findUserByNo` → 1번만 있다고 가정. 그 외 번호면 직접 만든 `UserNotFoundException`(`RuntimeException`의 자식)을 던짐.

### ExceptionController: 어떤 검사에 걸렸는지 보고 에러 코드 정하기

```java
@RestControllerAdvice
public class ExceptionController {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> methodValidException(MethodArgumentNotValidException e) {

        String code = "";
        String description = "";

        FieldError fieldError = e.getBindingResult().getFieldError();

        String detail = fieldError != null ? fieldError.getDefaultMessage() : "요청 값을 확인해주세요.";
        String bindResultCode = fieldError != null ? fieldError.getCode() : "";

        switch (bindResultCode) {
            case "NotBlank", "NotNull":
                code = "ERROR_CODE_00001";
                description = "필수 값이 누락되었거나 공백입니다.";
                break;
            case "Size":
                code = "ERROR_CODE_00002";
                description = "글자 수를 확인해주세요";
                break;
            default:
                code = "ERROR_CODE_99999";
                description = "요청 값 검증에 실패했습니다.";
        }

        ErrorResponse errorResponse = new ErrorResponse(code, description, detail);

        return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserException( UserNotFoundException e ) {
        String code = "ERROR_CODE_0000";
        String description = "회원 정보 조회 실패";
        String detail = e.getMessage();

        ErrorResponse errorResponse = new ErrorResponse(code, description, detail);

        return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
    }
}
```

* **한 줄 비유**: 시험지 채점표에 "몇 번 문제에서, **어떤 규칙** 때문에 틀렸는지"가 적혀 있음. 그 규칙 이름을 보고 **에러 코드 번호**를 붙여 줌.
* **꺼내는 값 정리**

| 코드 | 뜻 | 예 |
| :--- | :--- | :--- |
| `e.getBindingResult()` | 검사 결과 **채점표** 전체 | |
| `.getFieldError()` | 틀린 필드 오류 **하나** (없으면 `null`) | `id` 필드 오류 |
| `fieldError.getCode()` | 걸린 **규칙 이름** (어노테이션 이름) | `"NotBlank"`, `"Size"` |
| `fieldError.getDefaultMessage()` | 어노테이션에 적은 `message` | `"아이디는 반드시 입력되어야 합니다."` |

* `fieldError != null ? A : B` → 삼항 연산자. 혹시 필드 오류가 없을 때 `null.getCode()`로 **또 다른 예외가 터지지 않게** 막음.
* `case "NotBlank", "NotNull":` → Java 14부터 `case` 하나에 값을 **쉼표로 여러 개** 쓸 수 있음. 둘 중 하나면 같은 코드로 처리.
* `ErrorResponse(code, description, detail)` → `code`는 프로그램이 읽는 **번호**, `description`은 분류 설명, `detail`은 **구체적인 이유**.
* **왜 에러 코드를 따로 두나**: 프론트엔드가 `"아이디는 반드시..."` 같은 문장을 비교하면, 문장이 조금만 바뀌어도 코드가 깨짐. `ERROR_CODE_00001`처럼 **바뀌지 않는 번호**로 판단하면 안전함.

### 따라가기

* **정상 등록** (`{"id":"user01","pwd":"pass01","name":"홍길동","enrollDate":"2024-01-01T00:00:00.000+00:00"}`)
  1. 모든 검사 통과 (`pass01`은 6글자, 가입일은 과거) → 본문 실행
  2. `201 Created`, `Location: /api/v1/validation/users/1`
* **`id` 누락** (`{"pwd":"pass01","name":"홍길동","enrollDate":"2024-01-01..."}`)
  1. `id`가 `null` → `@NotBlank` 실패 → `MethodArgumentNotValidException`
  2. `getCode()` → `"NotBlank"` → `case "NotBlank", "NotNull"`
  3. `400 Bad Request` → `{"code":"ERROR_CODE_00001","description":"필수 값이 누락되었거나 공백입니다.","detail":"아이디는 반드시 입력되어야 합니다."}`
* **비밀번호 17글자** (`"pwd":"pass0111111111111"`, `"name":"안녕"`)
  1. `pwd`는 비어 있지 않지만 10글자 초과 → `@Size` 실패
  2. `getCode()` → `"Size"` → `400` → `{"code":"ERROR_CODE_00002","description":"글자 수를 확인해주세요","detail":"비밀번호는 길이 10을 초과할 수 없습니다."}`
  * `api-test.http`의 8번 요청은 제목이 "이름 공백으로 400"이지만, 실제로는 이름 `안녕`은 정상이고 **비밀번호 길이** 때문에 실패하는 요청임. 이름 공백을 시험하려면 `"name": "   "`을 보내야 `ERROR_CODE_00001`이 나옴.
* **없는 회원 조회** (`GET /api/v1/validation/users/999`)
  1. `999 != 1` → `UserNotFoundException("999번 회원 정보를 찾을 수 없습니다.")`
  2. `handleUserException` → `404 Not Found` → `{"code":"ERROR_CODE_0000","description":"회원 정보 조회 실패","detail":"999번 회원 정보를 찾을 수 없습니다."}`

> ⚠️ `getFieldError()`는 틀린 필드 중 **하나만** 돌려줌. `id`도 없고 `pwd`도 너무 길면 둘 중 하나만 응답에 담김(어느 쪽이 먼저일지도 보장되지 않음). 모든 오류를 알려주고 싶으면 10/28처럼 `getFieldErrors()`(복수형)로 **목록 전체**를 꺼내야 함.

---

## 3. Swagger (springdoc-openapi): API 문서 자동 만들기

### Swagger가 뭔가

* **한 줄 비유**: 식당의 **메뉴판**. 손님(프론트엔드 개발자)이 주방(백엔드 코드)에 들어가지 않아도, 메뉴판만 보고 "무엇을, 어떻게 주문하면, 뭐가 나오는지" 알 수 있음. 게다가 메뉴판에서 **바로 주문(요청 테스트)** 까지 할 수 있음.
* **OpenAPI**: REST API를 설명하는 **표준 양식**(JSON/YAML). **Swagger UI**: 그 양식을 **보기 좋은 웹 화면**으로 그려 주는 도구.
* **springdoc-openapi**: Spring Controller의 `@GetMapping`, `@RequestParam`, DTO 등을 읽어서 OpenAPI 문서를 **자동으로** 만들어 주는 라이브러리.
* 문서를 손으로 따로 쓰면 코드가 바뀔 때 문서를 깜빡하고 안 고치는 일이 생김. 코드에서 바로 만들면 **코드와 문서가 항상 같음**.

### 설치와 설정

```gradle
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:3.0.3'
```

```properties
# 이 패키지 아래 Controller만 문서에 보여준다
springdoc.packages-to-scan=com.ohgiraffers.restapi.section03
```

* 의존성만 넣고 서버를 켜면 바로 사용할 수 있음.

| 주소 | 내용 |
| :--- | :--- |
| `http://localhost:8080/swagger-ui/index.html` | Swagger UI 화면 (메뉴판) |
| `http://localhost:8080/v3/api-docs` | 화면이 읽는 OpenAPI **원본 JSON** |

* `packages-to-scan` → 설정하지 않으면 프로젝트의 **모든 Controller**가 문서에 나옴. 여기서는 `section03`만 지정해서 Swagger 실습용 API만 보이게 함 (section01·02 API는 동작은 하지만 문서에는 안 나옴).

### SwaggerConfig: 문서의 표지

```java
@Configuration
public class SwaggerConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI().info(swaggerInfo());
    }

    private Info swaggerInfo() {
        return new Info()
                .title("Ohgiraffers API")
                .description("SpringBoot Swagger 연동 테스트")
                .version("1.0.0");
    }
}
```

* `OpenAPI` 객체를 `@Bean`으로 등록하면 springdoc이 그걸 가져다 **문서 맨 위 정보**로 씀.
* `Info` → 책 표지처럼 **제목·설명·버전**을 적음. Swagger UI 맨 위에 `Ohgiraffers API 1.0.0`이 크게 보임.

### Controller에 설명 붙이기

```java
@RestController
@RequestMapping("/api/v1/swagger/users")
@Tag(name="user 관련 api", description = "회원 조회 등록 수정 삭제 API")
public class SwaggerTestController {

    @Operation(summary = "회원 목록 조회/검색",
            description = "전체회원을 조회하거나 name, status query string으로 회원을 검색한다")
    @ApiResponses({ @ApiResponse(responseCode = "200", description = "회원 목록 조회 성공") })
    @GetMapping
    public ResponseEntity<ResponseMessage> findUserByConditions(
            @RequestParam(required = false) String name,
            @RequestParam(required = false) String status) { ... }

    @ApiResponses({@ApiResponse(responseCode = "200", description = "회원 단건 조회 성공"),
                    @ApiResponse(responseCode = "404", description = "회원 정보 없음")})
    @GetMapping("/{userNo}")
    public ResponseEntity<ResponseMessage> findUserByNo(@PathVariable int userNo) { ... }
    ...
}
```

| 어노테이션 | 붙이는 곳 | 하는 일 |
| :--- | :--- | :--- |
| `@Tag(name, description)` | Controller 클래스 | API들을 **한 묶음(그룹)** 으로 보여줌. 메뉴판의 "식사 / 음료" 같은 칸 |
| `@Operation(summary, description)` | 핸들러 메소드 | API 한 개의 **한 줄 제목**과 자세한 설명 |
| `@ApiResponses` / `@ApiResponse(responseCode, description)` | 핸들러 메소드 | "이 API는 **200이면 이런 뜻, 404면 이런 뜻**"이라고 가능한 응답을 적어 둠 |
| `@Schema(description, example)` | DTO 클래스·필드 | DTO와 각 필드의 **설명과 예시 값** |

* 어노테이션을 **안 붙여도** 주소·메소드·파라미터·DTO 모양은 자동으로 문서에 나옴. 위 어노테이션들은 **사람이 읽을 설명을 덧붙이는** 용도임. (실습에서 `POST`·`PUT`·`DELETE`에는 `@Operation`이 없지만 문서에는 메소드 이름으로 보임)
* Swagger UI에서 API를 펼치고 **Try it out** → 값 입력 → **Execute**를 누르면 실제로 요청이 가고 응답 코드·헤더·body를 볼 수 있음. `api-test.http` 없이도 테스트 가능.

### DTO에 설명과 예시 붙이기: @Schema

```java
@ToString(exclude = "pwd")
@Schema(description = "회원 등록.수정 요청")
public class UserRequestDTO {

    @Schema(description = "회원ID", example = "user04")
    private String id;
    private String pwd;
    @Schema(description = "회원 이름", example = "신규 회원")
    private String name;
    private String status;
}
```

* Swagger UI에서 `POST`의 **Try it out**을 누르면 요청 body 칸에 `{"id":"user04","pwd":"string","name":"신규 회원","status":"string"}`처럼 `example` 값이 **미리 채워짐**. `example`이 없는 필드는 타입 이름(`string`)이 들어감.

### 없는 회원은 404로: orElseThrow

```java
private UserResponseDTO findUser(int userNo) {
    return users.stream()
            .filter(user -> user.getNo() == userNo)
            .findFirst()
            .orElseThrow(() -> new SwaggerUserNotFoundException(
                    userNo + "번 회원 정보를 찾을 수 없습니다."));
}
```

* section01에서는 조회·수정·삭제마다 같은 `stream().filter().findFirst().get()`을 **세 번 반복**했음 → 여기서는 `findUser()` 하나로 모아서 재사용함.
* `.get()` 대신 **`.orElseThrow(...)`** → 상자가 비어 있으면 **내가 정한 예외**를 던짐. `() -> new ...` 람다는 "**비어 있을 때만** 예외 객체를 만들어라"라는 뜻.

```java
@RestControllerAdvice(basePackageClasses = SwaggerTestController.class)
public class SwaggerExceptionHandler {

    @ExceptionHandler(SwaggerUserNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleUserNotFound(
            SwaggerUserNotFoundException exception) {

        Map<String, Object> error = new LinkedHashMap<>();
        error.put("status", HttpStatus.NOT_FOUND.value());
        error.put("error", HttpStatus.NOT_FOUND.name());
        error.put("message", exception.getMessage());

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

* `basePackageClasses = SwaggerTestController.class` → "이 클래스가 있는 **패키지(`section03.swagger`)의 Controller에서 난 예외만** 맡겠다"는 뜻. 범위를 좁혀서 다른 section의 예외 처리와 섞이지 않게 함. (section02의 `ExceptionController`는 범위를 안 정해서 **모든 Controller**에 적용됨)
* `LinkedHashMap` → 넣은 **순서대로** JSON에 나옴 (`HashMap`은 순서가 뒤죽박죽일 수 있음).
* `HttpStatus.NOT_FOUND.value()` → `404`, `.name()` → `"NOT_FOUND"`
* **따라가기** (`GET /api/v1/swagger/users/999`)
  1. `findUser(999)` → 필터에 맞는 회원이 없음 → 빈 `Optional` → `orElseThrow`가 `SwaggerUserNotFoundException` 던짐
  2. `SwaggerExceptionHandler.handleUserNotFound` 실행
  3. `404 Not Found` → `{"status":404,"error":"NOT_FOUND","message":"999번 회원 정보를 찾을 수 없습니다."}`
  * section01의 같은 요청은 `500`이었는데, 여기서는 의미에 맞는 `404`가 나감. Swagger의 `@ApiResponse(responseCode = "404", ...)` 설명과도 실제 동작이 일치하게 됨.

---

## 📝 오늘의 정리

1. REST API는 주소에 **명사(자원)**, HTTP 메소드에 **행동**을 맡기고, 자원 하나는 path variable, 검색·필터는 query string으로 표현함
2. 요청 DTO와 응답 DTO를 나눠 비밀번호가 응답에 섞이지 않게 하고, `@ToString(exclude = "pwd")`로 로그에도 남지 않게 함
3. 조회는 `200 OK`(`ok()`), 등록은 `201 Created` + `Location` 헤더(`created(uri).build()`), 수정·삭제는 `204 No Content`(`noContent().build()`)로 응답함
4. `@NotNull`·`@NotBlank`·`@Size`·`@Past`로 검사하고, `getFieldError().getCode()`로 걸린 규칙 이름을 꺼내 `ERROR_CODE_xxxxx` 같은 고정 에러 코드로 바꿔 응답함
5. springdoc-openapi를 넣으면 `/swagger-ui/index.html`에 API 문서가 자동으로 생기고, `@Tag`·`@Operation`·`@ApiResponse`·`@Schema`로 설명과 예시를 덧붙이며, Try it out으로 바로 테스트할 수 있음
6. `Optional.get()` 대신 `orElseThrow()`로 사용자 정의 예외를 던져야 "없음"이 `500`이 아니라 `404`로 응답됨

---

## 느낀 점

* 같은 기능이라도 `200`, `201`, `204`처럼 상황에 맞는 상태 코드와 `Location` 헤더를 돌려주니, 응답 body를 열어보지 않아도 결과를 알 수 있는 API가 된다는 점이 인상 깊었음
* 요청 DTO와 응답 DTO를 나누는 게 처음엔 클래스만 늘어나는 것 같았는데, 응답에 비밀번호 칸 자체가 없으면 실수로 새어 나갈 일도 없다는 설명을 듣고 납득됨
* 에러 메시지 문장 대신 고정된 에러 코드를 주는 이유가 프론트엔드가 코드로 분기하기 쉽게 하려는 것임을 알게 됨. 다만 `getFieldError()`는 오류를 하나만 준다는 점은 기억해 둬야겠음
* `.get()` 하나 때문에 "없는 회원"이 서버 오류(500)로 나가는 것을 보고, `orElseThrow`로 의도한 예외를 던지는 습관이 중요하다는 것을 느낌
* Swagger는 의존성 하나로 문서와 테스트 화면이 생겨서 신기했음. 팀 프로젝트에서 프론트엔드와 API를 맞출 때 `api-test.http`보다 훨씬 편하게 쓸 수 있을 것 같음
