# 핸들러 메소드 매개변수 · Jackson JSON 변환 · 전역 예외 처리 정리 (TIL)

> 실습 환경: Spring Boot 4 + Spring Web MVC (`chap02-handler-method` ~ `chap04-exception-handler`), Jackson 3, Bean Validation, IntelliJ HTTP Client (`api-test.http`)

## 1. 요청 데이터를 받는 핸들러 메소드 매개변수

| 어노테이션 | 꺼내는 위치 | 예시 |
| :--- | :--- | :--- |
| `@RequestParam` | query string | `?keyword=coffee` |
| `@PathVariable` | URL 경로 | `/menus/5` |
| `@RequestBody` | 요청 본문(body) | JSON, 문자열 |
| `@RequestHeader` | 요청 헤더 | `User-Agent`, `Content-Type` |
| `@CookieValue` | 쿠키 | `Cookie: clientId=client-001` |

### @RequestParam Map: query string 전체 받기

* 파라미터 이름을 하나하나 지정하지 않고 **`Map<String, String>`으로 전부** 받을 수 있음.
* 검색 조건처럼 어떤 파라미터가 올지 유동적일 때 확인용으로 편함.

```java
@GetMapping("/search")
public ResponseEntity<Map<String, Object>> searchMenus(
        @RequestParam Map<String, String> parameters) {

    Map<String, Object> response = new LinkedHashMap<>();
    response.put("message", "검색 조건 전체 확인");
    response.put("parameter", parameters);

    return ResponseEntity.ok(response);
}
```

```http
GET http://localhost:8080/api/v1/menus/search?name=latte&minPrice=3000&maxPrice=6000&orderableStatus=Y
```

> 💡 `Map`으로 받으면 값이 전부 `String`이라서, 숫자로 쓰려면 직접 변환해야 함.

### @PathVariable 이름 지정

* `@PathVariable("menuCode")`처럼 `{}` 안의 이름을 직접 적을 수도 있음. 생략하면 매개변수 이름과 같은 경로 변수를 찾음.

```java
@GetMapping("/{menuCode}")
public ResponseEntity<Map<String, Object>> findMenu(@PathVariable("menuCode") int menuCode) { ... }
```

### @RequestBody: JSON 본문 → DTO

* `application/json` 본문을 **프로퍼티 이름이 같은 DTO 필드**에 채워서 넘겨줌.

```java
@PostMapping
public ResponseEntity<Map<String, Object>> createMenu(@RequestBody MenuDTO menu) {

    Map<String, Object> response = new LinkedHashMap<>();
    response.put("message", "메뉴 등록 요청 성공");
    response.put("menu", menu);

    // 등록 요청이므로 201 Created
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

```http
POST http://localhost:8080/api/v1/menus
Content-Type: application/json

{
  "name": "아이스 아메리카노",
  "price": 3000,
  "categoryCode": 2,
  "orderableStatus": "Y"
}
```

| 상황 | 결과 |
| :--- | :--- |
| 본문이 비었거나 JSON 형식이 잘못됨 | `400 Bad Request` |
| 지원하지 않는 `Content-Type` | `415 Unsupported Media Type` |

* 매개변수 타입을 **`String`** 으로 두면 변환 없이 **본문 그대로** 받음.

### @RequestHeader · @CookieValue

* `@RequestHeader`는 지정한 헤더 값을 받음. **기본이 필수(`required = true`)** 라서 없을 수 있으면 `required = false`를 줘야 함.
* `@RequestHeader Map<String, String>`으로 헤더 전체를 받을 수도 있음.
* `@CookieValue`는 이름에 해당하는 쿠키 값을 받음.

```java
@PostMapping("/raw")
public ResponseEntity<Map<String, Object>> readRawBody(
        @RequestBody String body,                          // 본문 문자열 그대로
        @RequestHeader("content-type") String contentType) { ... }

@GetMapping("request-info")
public ResponseEntity<Map<String, Object>> requestInfo(
        @RequestHeader(value = "user-agent", required = false) String userAgent,
        @RequestHeader Map<String, String> headers,
        @CookieValue(value = "clientId", required = false) String clientId) {

    Map<String, Object> response = new LinkedHashMap<>();
    response.put("message", "요청 헤더와 쿠키 확인");
    response.put("userAgent", userAgent);
    response.put("clientId", clientId);
    response.put("headerCount", headers.size());

    return ResponseEntity.ok(response);
}
```

```http
GET http://localhost:8080/api/v1/menus/request-info
User-Agent: IntelliJ HTTP Client
Cookie: clientId=client-001
```

> 💡 HTTP 헤더 이름은 대소문자를 구분하지 않아서 `"content-type"`으로 적어도 `Content-Type` 헤더를 찾음.

---

## 2. Jackson: Java 객체 ↔ JSON 변환

* **직렬화(Serialization)**: Java 객체 → JSON (응답할 때)
* **역직렬화(Deserialization)**: JSON → Java 객체 (`@RequestBody`로 받을 때)
* Spring Boot는 Jackson을 기본으로 포함하고 있어서 `@RestController`의 반환값과 `@RequestBody`를 자동으로 변환해줌.

| Java | JSON |
| :--- | :--- |
| DTO / `Map` | 객체 `{ }` |
| `List` | 배열 `[ ]` |
| `Map` 안의 `List` | 객체 안에 중첩된 배열 |

```java
/* List는 JSON 배열로, 각 MemberDTO는 배열 안의 JSON 객체로 직렬화 */
@GetMapping
public ResponseEntity<List<MemberDTO>> findMembers() {
    return ResponseEntity.ok(memberList);
}

/* message + count + data 구조로 감싸서 응답 */
@GetMapping("/response-wrapper")
public ResponseEntity<Map<String, Object>> findMembersWithWrapper() {

    Map<String, Object> response = new LinkedHashMap<>();
    response.put("message", "회원 목록 조회 성공");
    response.put("count", memberList.size());
    response.put("data", memberList);

    return ResponseEntity.ok(response);
}
```

```json
{
  "message": "회원 목록 조회 성공",
  "count": 4,
  "data": [
    { "memberNo": 1, "name": "...", "age": 18, "enrollDate": "2026-04-21" }
  ]
}
```

### @JsonProperty · @JsonFormat

* `@JsonProperty("이름")`: Java 필드명과 **JSON 키 이름을 다르게** 연결. 직렬화·역직렬화 둘 다 적용됨.
* `@JsonFormat(pattern = "...")`: 날짜 타입을 **지정한 문자열 형식**으로 주고받음. 형식이 다르면 역직렬화 실패(400).

```java
public class MemberDTO {

    @JsonProperty("memberNo")   // Java의 no ↔ JSON의 memberNo
    private int no;
    private String name;
    private int age;

    @JsonFormat(pattern = "yyyy-MM-dd")   // LocalDate ↔ "2026-05-27"
    private LocalDate enrollDate;

    public MemberDTO() {}   // 역직렬화할 때 기본 생성자로 객체를 만든 뒤 값을 채움
    ...
}
```

```http
POST http://localhost:8080/api/v1/members
Content-Type: application/json

{
  "memberNo": 5,
  "name": "하재경",
  "age": 18,
  "enrollDate": "2026-05-27"
}
```

> ⚠️ `@JsonProperty("memberNo")`를 붙이면 요청 JSON도 `"no"`가 아니라 `"memberNo"`로 보내야 값이 들어감.

### ObjectMapper 직접 사용

* `ObjectMapper`: Jackson의 JSON 변환기. Spring Boot가 날짜 모듈 등 공통 설정을 적용해서 bean으로 등록해두므로 **생성자 주입**으로 받아 씀.

```java
private final ObjectMapper objectMapper;

public JacksonController(ObjectMapper objectMapper) {
    this.objectMapper = objectMapper;
}

@GetMapping("/object-mapper")
public ResponseEntity<Map<String, Object>> objectMapperTest() {

    String jsonString = objectMapper.writeValueAsString(memberList);   // 객체 → JSON 문자열
    ...
}
```

> 💡 Spring Boot 4는 Jackson 3을 사용해서 `ObjectMapper`는 `tools.jackson.databind` 패키지에 있음. 반면 `@JsonProperty`·`@JsonFormat` 같은 어노테이션은 그대로 `com.fasterxml.jackson.annotation` 패키지를 씀.

> 💡 Jackson 3의 `writeValueAsString()`은 unchecked 예외를 던져서 `try-catch` 없이 호출할 수 있음.

---

## 3. 전역 예외 처리: @RestControllerAdvice + @ExceptionHandler

* 예외를 처리하지 않으면 모든 오류가 `500`으로 나가고 응답 형식도 제각각이 됨.
* 상황에 맞는 **상태 코드**와 **통일된 JSON 실패 응답**을 주기 위해 예외 처리를 한 곳에 모음.

| 구성 요소 | 역할 |
| :--- | :--- |
| `@RestControllerAdvice` | 모든 Controller에서 발생한 예외를 공통으로 처리하고, 반환값을 응답 본문(JSON)으로 보냄 |
| `@ExceptionHandler(예외.class)` | 해당 예외 타입이 발생했을 때 실행할 메소드 지정 |
| `ErrorResponse` | 실패 응답의 공통 JSON 구조 |
| 사용자 정의 예외 | `RuntimeException`을 상속해 상황(회원 없음, 잘못된 요청)을 이름으로 구분 |

### 실패 응답 DTO와 사용자 정의 예외

```java
public class ErrorResponse {

    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private final LocalDateTime timestamp;
    private final int status;      // 404
    private final String error;    // "Not Found"
    private final String message;  // "999번 회원을 찾을 수 없습니다."
    private final String path;     // "/api/v1/members/999"

    public ErrorResponse(int status, String error, String message, String path) {
        this.timestamp = LocalDateTime.now();
        ...
    }
    // getter
}
```

```java
public class MemberNotFoundException extends RuntimeException {
    public MemberNotFoundException(String message) {
        super(message);
    }
}
```

### Controller: 예외만 던진다

```java
@GetMapping("/{memberNo}")
public ResponseEntity<MemberDTO> findMember(@PathVariable int memberNo) {

    MemberDTO foundMember = members.stream()
            .filter(member -> member.getNo() == memberNo)
            .findFirst()
            .orElseThrow(() -> new MemberNotFoundException(memberNo + "번 회원을 찾을 수 없습니다."));

    return ResponseEntity.ok(foundMember);
}

@PostMapping
public ResponseEntity<Map<String, Object>> createMember(@Valid @RequestBody MemberDTO member) {

    boolean duplicatedMemberNo = members.stream()
            .anyMatch(savedMember -> savedMember.getNo() == member.getNo());

    if (duplicatedMemberNo) {
        throw new InvalidMemberRequestException("이미 등록된 회원입니다.");
    }
    members.add(member);
    ...
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

### GlobalExceptionHandler: 예외 → 상태 코드 + JSON

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    /* 조회 대상이 없으면 500이 아니라 404 */
    @ExceptionHandler(MemberNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleMemberNotFound(
            MemberNotFoundException exception, WebRequest request) {

        ErrorResponse response = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                HttpStatus.NOT_FOUND.getReasonPhrase(),
                exception.getMessage(),
                extractPath(request)
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
    }

    /* 처리하지 못한 나머지 예외는 원인을 로그에 남기고 500 */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception exception, WebRequest request) {

        log.error("처리하지 못한 예외가 발생했습니다", exception);

        ErrorResponse response = new ErrorResponse(
                HttpStatus.INTERNAL_SERVER_ERROR.value(),
                HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase(),
                "서버 내부 오류가 발생했습니다",
                extractPath(request)
        );
        return ResponseEntity.internalServerError().body(response);
    }

    /* "uri=/api/v1/members/999" → "/api/v1/members/999" */
    private String extractPath(WebRequest request) {
        return request.getDescription(false).replace("uri=", "");
    }
}
```

| 예외 | 발생 상황 | 응답 |
| :--- | :--- | :--- |
| `MemberNotFoundException` | 없는 회원 번호 조회 | `404 Not Found` |
| `MethodArgumentNotValidException` | `@Valid` 검증 실패 (이름 빈 값 등) | `400 Bad Request` |
| `InvalidMemberRequestException` | 중복 회원 번호 등록 | `400 Bad Request` |
| `HttpMessageNotReadableException` | JSON 형식·타입 불일치 (`"age": "스무 살"`) | `400 Bad Request` |
| `Exception` | 그 외 예상하지 못한 오류 | `500 Internal Server Error` |

> 💡 여러 핸들러가 해당될 때는 **가장 구체적인 예외 타입**의 핸들러가 선택됨. 그래서 `Exception.class` 핸들러는 마지막 안전망 역할을 함.

> ⚠️ 500 응답에는 내부 정보를 숨기고 일반 메시지만 보내고, **실제 원인은 `log.error()`로 서버 로그에** 남김. `System.out` 대신 Logger를 쓰면 로그 레벨·시간·클래스 정보가 함께 기록됨.

```json
{
  "timestamp": "2026-10-28T14:20:31",
  "status": 404,
  "error": "Not Found",
  "message": "999번 회원을 찾을 수 없습니다.",
  "path": "/api/v1/members/999"
}
```

---

## 4. Bean Validation: @Valid로 요청 값 검증

* `spring-boot-starter-validation` 의존성을 추가하면 DTO 필드에 **검증 어노테이션**을 붙일 수 있음.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

| 어노테이션 | 검사 내용 |
| :--- | :--- |
| `@NotBlank` | `null`, 빈 문자열, 공백만 있는 문자열 불가 |
| `@Min(value)` | 지정한 값 이상이어야 함 |

```java
public class MemberDTO {

    @Min(value = 1, message = "회원 번호는 1 이상이어야 합니다")
    private int no;

    @NotBlank(message = "회원 이름은 비어 있을 수 없습니다.")
    private String name;

    @Min(value = 0, message = "회원 나이는 0 이상이어야 합니다.")
    private int age;
    ...
}
```

* Controller 매개변수에 **`@Valid`** 를 붙여야 검증이 실행됨. 실패하면 메소드 본문이 실행되기 전에 `MethodArgumentNotValidException`이 발생함.
* 핸들러에서 `getBindingResult().getFieldErrors()`로 실패한 필드의 `message`를 꺼내 응답에 담음.

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleMethodArgumentNotValid(
        MethodArgumentNotValidException exception, WebRequest request) {

    String message = exception.getBindingResult()
            .getFieldErrors()
            .stream()
            .findFirst()
            .map(error -> error.getDefaultMessage())   // 어노테이션에 적은 message
            .orElse("요청 값이 올바르지 않습니다");

    ErrorResponse response = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            HttpStatus.BAD_REQUEST.getReasonPhrase(),
            message,
            extractPath(request)
    );
    return ResponseEntity.badRequest().body(response);
}
```

```http
### 이름 누락 → 400, "회원 이름은 비어 있을 수 없습니다."
POST http://localhost:8080/api/v1/members
Content-Type: application/json

{
  "no": 5,
  "name": "",
  "age": 22
}
```

> 💡 `"age": "스무 살"`처럼 타입 자체가 안 맞으면 검증 단계까지 가지 못하고 JSON 변환 단계에서 `HttpMessageNotReadableException`이 발생함. 그래서 두 예외를 따로 처리함.

---

## 📝 오늘의 정리

1. query string은 `@RequestParam`(전체는 `Map`), 경로 값은 `@PathVariable`, 본문은 `@RequestBody`, 헤더·쿠키는 `@RequestHeader`·`@CookieValue`로 받는다.
2. Jackson이 DTO·`Map`은 JSON 객체로, `List`는 JSON 배열로 자동 변환하고, `@RequestBody`로 받은 JSON은 DTO로 역직렬화한다.
3. `@JsonProperty`로 JSON 키 이름을, `@JsonFormat`으로 날짜 형식을 지정하고, 필요하면 주입받은 `ObjectMapper`로 직접 변환한다.
4. `@RestControllerAdvice` + `@ExceptionHandler`로 예외를 한 곳에서 처리해 상황별 상태 코드(404/400/500)와 통일된 `ErrorResponse`를 응답한다.
5. `spring-boot-starter-validation`의 `@NotBlank`·`@Min`과 `@Valid`로 요청 값을 검증하고, 실패 메시지를 `MethodArgumentNotValidException`에서 꺼내 응답한다.

---

## 느낀 점

* 요청 데이터가 query string, 경로, 본문, 헤더, 쿠키 중 어디에 있는지에 따라 어노테이션이 나뉘어 있어서, HTTP 요청 구조를 다시 정리하는 계기가 됨.
* `List`와 `Map`을 반환하기만 해도 JSON 배열과 객체로 바뀌는 걸 보니 Jackson이 뒤에서 많은 일을 해주고 있다는 게 느껴짐. `@JsonProperty`를 붙이면 요청 JSON 키도 바뀐다는 점은 주의해야겠음.
* Controller는 예외만 던지고 응답 형식은 `GlobalExceptionHandler`가 정하니까, 각 메소드에 `try-catch`를 넣지 않아도 실패 응답이 일정하게 나와서 깔끔했음.
* 같은 400이라도 검증 실패, 중복 등록, JSON 타입 불일치처럼 원인이 다르다는 걸 `api-test.http`로 하나씩 보내보며 확인할 수 있었음.
