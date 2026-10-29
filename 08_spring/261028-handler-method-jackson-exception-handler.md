# 핸들러 메소드 매개변수 · Jackson JSON 변환 · 전역 예외 처리 정리 (TIL)

> 실습 환경: Spring Boot 4 + Spring Web MVC (`chap02-handler-method` ~ `chap04-exception-handler`), Jackson 3, Bean Validation, IntelliJ HTTP Client (`api-test.http`)

## 1. 요청 데이터를 받는 핸들러 메소드 매개변수

* **HTTP 요청은 택배 상자**임. 정보가 상자의 **여러 곳**에 붙어 올 수 있고, 어디서 꺼낼지에 따라 쓰는 어노테이션이 다름.
  * 송장 주소 → URL 경로 / 주소 뒤 메모 → query string / 상자 안 내용물 → 본문(body) / 상자 겉 스티커 → 헤더 / 단골 손님 표시 → 쿠키

| 어노테이션 | 꺼내는 위치 | 예시 |
| :--- | :--- | :--- |
| `@RequestParam` | query string | `?keyword=coffee` |
| `@PathVariable` | URL 경로 | `/menus/5` |
| `@RequestBody` | 요청 본문(body) | JSON, 문자열 |
| `@RequestHeader` | 요청 헤더 | `User-Agent`, `Content-Type` |
| `@CookieValue` | 쿠키 | `Cookie: clientId=client-001` |

### @RequestParam Map: query string 전체 받기

* 파라미터 이름을 하나하나 정하지 않고 **`Map<String, String>`으로 전부** 받을 수 있음.
* 검색 조건처럼 어떤 파라미터가 올지 그때그때 다를 때, 무엇이 들어왔는지 확인하기 편함.

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

* **따라가기**: `?` 뒤의 `이름=값` 4쌍이 `parameters`에 담김 → `{name=latte, minPrice=3000, maxPrice=6000, orderableStatus=Y}` → 응답 `{"message":"검색 조건 전체 확인","parameter":{"name":"latte","minPrice":"3000",...}}`

> 💡 `Map`으로 받으면 값이 전부 `String`이라서, 숫자로 쓰려면 직접 변환해야 함. (`"3000"` → `Integer.parseInt("3000")`)

### @PathVariable 이름 지정

* `@PathVariable("menuCode")`처럼 `{}` 안의 이름을 직접 적을 수도 있음. 생략하면 매개변수 이름과 같은 경로 변수를 찾음.

```java
@GetMapping("/{menuCode}")
public ResponseEntity<Map<String, Object>> findMenu(@PathVariable("menuCode") int menuCode) { ... }
```

### @RequestBody: JSON 본문 → DTO

* 요청 **본문**에 담긴 JSON을 읽어서, **이름이 같은 DTO 필드**에 채워 객체로 만들어 줌. 택배 상자를 열어 물건을 **정리함 칸에 이름대로** 넣어주는 것과 같음.

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

* **따라가기**
  1. `Content-Type: application/json` → "상자 안 내용물은 JSON이에요"라는 스티커.
  2. 스프링(Jackson)이 본문을 읽어 `MenuDTO`를 만들고 `name`, `price`, `categoryCode`, `orderableStatus`에 값을 채움.
  3. 응답 → `201 Created`, `{"message":"메뉴 등록 요청 성공","menu":{"name":"아이스 아메리카노","price":3000,...}}`

| 상황 | 결과 |
| :--- | :--- |
| 본문이 비었거나 JSON 형식이 잘못됨 | `400 Bad Request` |
| 지원하지 않는 `Content-Type` | `415 Unsupported Media Type` |

* 매개변수 타입을 **`String`** 으로 두면 변환하지 않고 **본문 글자 그대로** 받음.

### @RequestHeader · @CookieValue

* **헤더**: 요청에 붙는 **부가 정보 스티커**. 어떤 브라우저인지(`User-Agent`), 내용물 형식이 뭔지(`Content-Type`) 등.
* **쿠키**: 서버가 브라우저에 맡겨둔 **작은 메모**. 다음 요청 때 브라우저가 자동으로 같이 보냄(로그인 유지 등에 씀).
* `@RequestHeader`는 지정한 헤더 값을 받음. **기본이 필수(`required = true`)** 라서 없을 수도 있으면 `required = false`를 줘야 함.
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

* **따라가기**
  * `readRawBody` → 본문을 JSON으로 바꾸지 않고 **글자 그대로** `body`에, `Content-Type` 헤더 값을 `contentType`에 받음.
  * `requestInfo`
    1. `user-agent` 헤더 → `"IntelliJ HTTP Client"`
    2. 헤더 전체 → `headers` Map (몇 개인지 세서 `headerCount`)
    3. `Cookie: clientId=client-001` → `clientId = "client-001"`
    4. 응답 → `{"message":"요청 헤더와 쿠키 확인","userAgent":"IntelliJ HTTP Client","clientId":"client-001","headerCount":(헤더 개수)}`
  * 쿠키를 안 보냈다면 → `required = false`라 `clientId`는 `null`.

> 💡 HTTP 헤더 이름은 대소문자를 구분하지 않아서 `"content-type"`으로 적어도 `Content-Type` 헤더를 찾음.

---

## 2. Jackson: Java 객체 ↔ JSON 변환

* **한 줄 비유**: Jackson은 **번역기**임. 자바 객체는 서버 안에서만 통하는 말이고, JSON은 브라우저·앱 등 누구나 읽는 **공용어**임.
* **직렬화(Serialization)**: Java 객체 → JSON 글자 (응답할 때). 물건을 상자에 **포장**하는 것.
* **역직렬화(Deserialization)**: JSON 글자 → Java 객체 (`@RequestBody`로 받을 때). 상자를 **풀어서** 물건을 꺼내는 것.
* Spring Boot에는 Jackson이 기본으로 들어 있어서, `@RestController`의 반환값과 `@RequestBody`를 **알아서** 변환해 줌.

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

* **따라가기**
  * `findMembers()` → `List<MemberDTO>` → Jackson이 `[ {회원1}, {회원2}, ... ]` 배열로 바꿈.
  * `findMembersWithWrapper()` → `Map` 하나에 메시지·개수·목록을 담음 → Map은 `{ }`, 그 안의 List는 `[ ]` → 위 JSON처럼 **객체 안에 배열**이 들어간 모양. 받는 쪽이 "몇 명인지", "성공했는지"를 한눈에 알 수 있음.

### @JsonProperty · @JsonFormat

* **`@JsonProperty("이름")`**: 자바 필드 이름과 **JSON 키 이름을 다르게** 연결함. 내보낼 때(직렬화)와 받을 때(역직렬화) **둘 다** 적용됨. 집에서는 "막내"라고 부르고 학교에서는 "민준"이라고 부르는 것처럼.
* **`@JsonFormat(pattern = "...")`**: 날짜를 **정한 글자 모양**으로 주고받음. 받은 글자가 그 모양이 아니면 변환 실패(400).

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

* **따라가기 (역직렬화)**
  1. Jackson이 **기본 생성자**로 빈 `MemberDTO`를 만듦.
  2. JSON 키 `"memberNo"` → `@JsonProperty("memberNo")`가 붙은 자바 필드 `no`에 `5`를 넣음.
  3. `"name"`, `"age"` → 이름이 같은 필드에 그대로.
  4. `"enrollDate": "2026-05-27"` → `yyyy-MM-dd` 모양이 맞음 → `LocalDate` 2026-05-27로 변환. `"2026/05/27"`처럼 모양이 다르면 실패 → 400.
* **직렬화(응답)** 때는 반대로 `no` 필드가 `"memberNo"` 키로 나감.

> ⚠️ `@JsonProperty("memberNo")`를 붙이면 요청 JSON도 `"no"`가 아니라 `"memberNo"`로 보내야 값이 들어감.

### ObjectMapper 직접 사용

* **`ObjectMapper`**: Jackson의 **번역기 본체**. 보통은 스프링이 알아서 쓰지만, 코드에서 직접 JSON 글자를 만들고 싶을 때 씀.
* Spring Boot가 날짜 처리 등 공통 설정을 해서 **bean으로 등록**해 두므로, 직접 `new`하지 않고 **생성자 주입**으로 받아 씀.

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

* **따라가기**: 스프링이 `JacksonController`를 만들 때 `ObjectMapper` bean을 꽂아줌 → `writeValueAsString(memberList)` → `"[{\"memberNo\":1,\"name\":...}, ...]"` 같은 **JSON 글자 한 줄**이 만들어짐.

> 💡 Spring Boot 4는 Jackson 3을 사용해서 `ObjectMapper`는 `tools.jackson.databind` 패키지에 있음. 반면 `@JsonProperty`·`@JsonFormat` 같은 어노테이션은 그대로 `com.fasterxml.jackson.annotation` 패키지를 씀.

> 💡 Jackson 3의 `writeValueAsString()`은 unchecked 예외를 던져서 `try-catch` 없이 호출할 수 있음.

---

## 3. 전역 예외 처리: @RestControllerAdvice + @ExceptionHandler

* **문제**: 스프링도 일부 오류(잘못된 JSON → 400 등)는 기본으로 처리해 주지만, 내가 만든 예외는 모두 `500`으로 나가고 **응답 모양도 제각각**이 됨. 받는 쪽(프론트엔드)은 오류마다 다른 모양을 해석해야 해서 힘듦.
* **해결**: 상황에 맞는 **상태 코드**와 **똑같은 모양의 JSON 실패 응답**을 주기 위해, 예외 처리를 **한 곳(전역 예외 처리기)** 에 모음. 학교의 모든 사고 신고를 **보건실 한 곳**에서 받아 같은 양식으로 기록하는 것과 같음.

| 구성 요소 | 역할 |
| :--- | :--- |
| `@RestControllerAdvice` | 모든 Controller에서 발생한 예외를 공통으로 처리하고, 반환값을 응답 본문(JSON)으로 보냄 |
| `@ExceptionHandler(예외.class)` | 해당 예외 타입이 발생했을 때 실행할 메소드 지정 |
| `ErrorResponse` | 실패 응답의 공통 JSON 구조 |
| 사용자 정의 예외 | `RuntimeException`을 상속해 상황(회원 없음, 잘못된 요청)을 이름으로 구분 |

### 실패 응답 DTO와 사용자 정의 예외

* `ErrorResponse`: 모든 실패 응답이 따르는 **공통 양식**(언제, 몇 번 오류, 무슨 오류, 설명, 어느 주소).
* `MemberNotFoundException`: "회원 없음" 상황을 이름으로 드러내는 예외. `RuntimeException`(unchecked)을 상속해서 `throws`를 줄줄이 달지 않아도 됨.

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

* `'T'`처럼 작은따옴표로 감싼 글자는 패턴이 아니라 **그 글자 그대로** 찍으라는 뜻 → `2026-10-28T14:20:31`

### Controller: 예외만 던진다

* Controller는 문제가 생기면 **예외를 던지기만** 하고, 응답을 어떻게 만들지는 신경 쓰지 않음.

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

* **따라가기: `GET /api/v1/members/999`**
  1. 회원 목록을 Stream으로 돌며 번호가 999인 회원을 찾음 → 없음 → `findFirst()`는 **빈 Optional**.
  2. `orElseThrow(...)` → 비어 있으므로 `MemberNotFoundException("999번 회원을 찾을 수 없습니다.")`를 던짐.
  3. Controller는 여기서 끝. 예외는 아래 `GlobalExceptionHandler`로 넘어감.
* **따라가기: 이미 있는 번호로 `POST`**: `anyMatch`로 같은 번호가 있는지 검사 → `true` → `InvalidMemberRequestException("이미 등록된 회원입니다.")`를 던짐.

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

* **따라가기 (위의 999번 조회 예외가 도착하면)**
  1. 스프링이 던져진 `MemberNotFoundException`에 맞는 `@ExceptionHandler`를 찾음 → `handleMemberNotFound` 실행.
  2. `HttpStatus.NOT_FOUND.value()` → `404`, `getReasonPhrase()` → `"Not Found"`, `exception.getMessage()` → `"999번 회원을 찾을 수 없습니다."`
  3. `extractPath(request)` → 요청 설명 `"uri=/api/v1/members/999"`에서 `"uri="`를 지움 → `"/api/v1/members/999"`
  4. 이 넷으로 `ErrorResponse`를 만들어 `404`와 함께 응답 → 아래 JSON처럼 나감.
  * 전혀 예상 못 한 예외(예: `NullPointerException`)가 나면 → 더 구체적인 핸들러가 없으므로 `Exception.class` 핸들러가 받음 → 원인은 `log.error`로 **서버 기록장**에만 남기고, 사용자에게는 일반 메시지와 `500`만 보냄.

| 예외 | 발생 상황 | 응답 |
| :--- | :--- | :--- |
| `MemberNotFoundException` | 없는 회원 번호 조회 | `404 Not Found` |
| `MethodArgumentNotValidException` | `@Valid` 검증 실패 (이름 빈 값 등) | `400 Bad Request` |
| `InvalidMemberRequestException` | 중복 회원 번호 등록 | `400 Bad Request` |
| `HttpMessageNotReadableException` | JSON 형식·타입 불일치 (`"age": "스무 살"`) | `400 Bad Request` |
| `Exception` | 그 외 예상하지 못한 오류 | `500 Internal Server Error` |

> 💡 여러 핸들러가 해당될 때는 **가장 구체적인 예외 타입**의 핸들러가 선택됨. 그래서 `Exception.class` 핸들러는 마지막 안전망 역할을 함.

> ⚠️ `@ExceptionHandler(Exception.class)`가 있으면 원래 스프링이 400으로 처리해 주던 `HttpMessageNotReadableException` 같은 예외도 **이 핸들러가 가로채서 500으로** 응답하게 됨. 표처럼 400을 돌려주려면 `HttpMessageNotReadableException`, `InvalidMemberRequestException`에 대한 `@ExceptionHandler`도 **따로** 만들어 두어야 함.

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

* **한 줄 비유**: 놀이공원 입구의 **키 재는 막대**임. 서비스 로직(놀이기구)에 들어가기 전에 입구에서 미리 "이름이 비었나? 나이가 음수인가?"를 검사해서 돌려보냄.
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

* 필드마다 규칙과 **실패했을 때 보여줄 메시지**를 함께 적어둠.
* Controller 매개변수에 **`@Valid`** 를 붙여야 검사가 실행됨(막대를 세워 두기만 하고 입구에서 재지 않으면 소용없음). 실패하면 메소드 본문이 실행되기 **전에** `MethodArgumentNotValidException`이 발생함.
* 핸들러에서 `getBindingResult().getFieldErrors()`로 실패한 필드들의 `message`를 꺼내 응답에 담음.

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

* **한 줄씩 보기**
  * `getBindingResult().getFieldErrors()` → 검사에 걸린 **필드 오류 목록**.
  * `.stream().findFirst()` → 그중 첫 번째 하나만 고름(여러 개 걸렸을 때 어느 것이 첫 번째일지는 정해져 있지 않음).
  * `.map(error -> error.getDefaultMessage())` → 그 오류에 적힌 메시지(어노테이션의 `message`)를 꺼냄.
  * `.orElse(...)` → 혹시 필드 오류가 없으면 기본 문구.
  * 그 메시지로 `400` `ErrorResponse`를 만들어 응답.

```http
### 이름 누락 → 400, "회원 이름은 비어 있을 수 없습니다."
POST http://localhost:8080/api/v1/members
Content-Type: application/json

{
  "memberNo": 5,
  "name": "",
  "age": 22
}
```

* **따라가기**
  1. JSON을 `MemberDTO`로 변환(역직렬화) → 성공.
  2. `@Valid` 검사 → `name`이 `""` → `@NotBlank` 위반.
  3. Controller 본문은 실행되지 않고 `MethodArgumentNotValidException` 발생 → 위 핸들러 → `400 Bad Request` + 걸린 필드의 메시지.

> 💡 `"age": "스무 살"`처럼 타입 자체가 안 맞으면 검증 단계까지 가지 못하고 JSON 변환 단계에서 `HttpMessageNotReadableException`이 발생함. 그래서 두 예외를 따로 처리함.

* **순서 정리**: 요청 도착 → ① JSON → DTO 변환 (실패 시 `HttpMessageNotReadableException`) → ② `@Valid` 검사 (실패 시 `MethodArgumentNotValidException`) → ③ Controller 본문 실행 (여기서 던지면 `MemberNotFoundException` 등).

---

## 📝 오늘의 정리

1. query string은 `@RequestParam`(전체는 `Map`), 경로 값은 `@PathVariable`, 본문은 `@RequestBody`, 헤더·쿠키는 `@RequestHeader`·`@CookieValue`로 받음
2. Jackson이 DTO·`Map`은 JSON 객체로, `List`는 JSON 배열로 자동 변환하고, `@RequestBody`로 받은 JSON은 DTO로 역직렬화함
3. `@JsonProperty`로 JSON 키 이름을, `@JsonFormat`으로 날짜 형식을 지정하고, 필요하면 주입받은 `ObjectMapper`로 직접 변환함
4. `@RestControllerAdvice` + `@ExceptionHandler`로 예외를 한 곳에서 처리해 상황별 상태 코드(404/400/500)와 통일된 `ErrorResponse`를 응답함
5. `spring-boot-starter-validation`의 `@NotBlank`·`@Min`과 `@Valid`로 요청 값을 검증하고, 실패 메시지를 `MethodArgumentNotValidException`에서 꺼내 응답함

---

## 느낀 점

* 요청 데이터가 query string, 경로, 본문, 헤더, 쿠키 중 어디에 있는지에 따라 어노테이션이 나뉘어 있어서, HTTP 요청 구조를 다시 정리하는 계기가 됨
* `List`와 `Map`을 반환하기만 해도 JSON 배열과 객체로 바뀌는 걸 보니 Jackson이 뒤에서 많은 일을 해주고 있다는 게 느껴짐. `@JsonProperty`를 붙이면 요청 JSON 키도 바뀐다는 점은 주의해야겠음
* Controller는 예외만 던지고 응답 형식은 `GlobalExceptionHandler`가 정하니까, 각 메소드에 `try-catch`를 넣지 않아도 실패 응답이 일정하게 나와서 깔끔했음
* 같은 400이라도 검증 실패, 중복 등록, JSON 타입 불일치처럼 원인이 다르다는 걸 `api-test.http`로 하나씩 보내보며 확인할 수 있었음
