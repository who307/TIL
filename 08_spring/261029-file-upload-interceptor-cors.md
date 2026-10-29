# 파일 업로드 · Interceptor · React 연동(CORS) 정리 (TIL)

> 실습 환경: Spring Boot 4 + Spring Web MVC (`chap05-file-upload` ~ `chap07-react-integration`), Next.js 16 + React 19 (`react-client`), IntelliJ HTTP Client (`api-test.http`)

## 1. 파일 업로드 (MultipartFile)

### multipart/form-data가 뭔가

* **한 줄 비유**: 지금까지 보낸 JSON 요청은 **편지 한 장**이었음. 파일 업로드는 편지와 함께 **사진, 문서 같은 물건까지 한 상자에 넣어 보내는 택배**임.
* **`multipart/form-data`**: 요청 본문을 **여러 칸(part)으로 나눠** 글자와 파일을 함께 보내는 형식. "multi(여러) + part(조각)"이라는 뜻.
* 칸과 칸 사이는 **경계선(boundary)** 이라는 약속된 글자로 나눔. 택배 상자 안의 **칸막이**와 같음.

```http
### 1. 단일 파일 업로드
POST http://localhost:8080/api/v1/files/single
Content-Type: multipart/form-data; boundary=singleBoundary

--singleBoundary
Content-Disposition: form-data; name="description"

단일 파일 설명입니다.
--singleBoundary
Content-Disposition: form-data; name="file"; filename="sample.txt"
Content-Type: text/plain

single file upload sample
--singleBoundary--
```

* **한 줄씩 보기**
  * `Content-Type: multipart/form-data; boundary=singleBoundary` → "이 상자는 칸이 나뉜 택배이고, 칸막이 이름은 `singleBoundary`야."
  * `--singleBoundary` → 칸막이. 여기서부터 새 칸이 시작됨.
  * 첫 번째 칸: `name="description"` → 이름이 `description`인 **글자 칸**, 내용은 `단일 파일 설명입니다.`
  * 두 번째 칸: `name="file"; filename="sample.txt"` → 이름이 `file`인 **파일 칸**. 원래 파일 이름은 `sample.txt`, 종류는 `text/plain`(일반 글자 파일), 파일 내용은 `single file upload sample`
  * `--singleBoundary--` → 칸막이 뒤에 `--`가 하나 더 붙으면 **상자의 끝**.
  * 브라우저에서 `<input type="file">`로 파일을 고르면 이 모양은 브라우저가 알아서 만들어 줌. 여기서는 연습을 위해 직접 적은 것임.

### 업로드 크기 제한과 저장 위치 설정

```properties
# 파일 하나의 최대 업로드 크기
spring.servlet.multipart.max-file-size=10MB

# multipart 요청 전체의 최대 크기
spring.servlet.multipart.max-request-size=50MB

# 업로드 파일이 저장될 프로젝트 루트 기준 디렉터리
file.upload-dir=uploads
```

* **따라가기**
  * `max-file-size=10MB` → 파일 **한 개**가 10MB를 넘으면 거절.
  * `max-request-size=50MB` → 한 번에 보내는 **상자 전체**가 50MB를 넘으면 거절. (10MB 파일 6개를 한꺼번에 보내면 60MB라 거절됨)
  * `file.upload-dir=uploads` → 스프링이 정한 설정이 아니라 **내가 만든 설정 이름**. 아래 Controller에서 `@Value`로 꺼내 씀.
* **왜 제한하나**: 제한이 없으면 누군가 아주 큰 파일을 보내 **서버 저장 공간과 메모리를 가득 채울** 수 있음.

### 업로드 결과를 담는 FileDTO

* 응답으로 **파일 자체**를 돌려주는 게 아니라, "어떤 이름으로, 어디에, 얼마만 한 크기로 저장했는지" 같은 **파일의 정보(메타 정보)** 만 JSON으로 돌려줌. 택배를 맡기면 받는 **접수증**과 같음.

| 필드 | 뜻 | 예 |
| :--- | :--- | :--- |
| `originFileName` | 사용자가 올린 원래 파일 이름 | `sample.txt` |
| `savedFileName` | 서버에 실제로 저장한 이름 (UUID) | `3f2a…9c.txt` |
| `filePath` | 업로드 폴더 기준 상대 경로 | `single/3f2a…9c.txt` |
| `fileDescription` | 함께 보낸 설명 | `단일 파일 설명입니다.` |
| `fileSize` | 파일 크기 (byte) | `25` |
| `contentType` | 파일 종류(MIME 타입) | `text/plain`, `image/png` |

### 단일 파일 업로드 Controller

* **`MultipartFile`**: 요청 상자에서 꺼낸 **파일 한 개**를 담는 스프링 객체. 파일 이름(`getOriginalFilename()`), 크기(`getSize()`), 종류(`getContentType()`)를 알 수 있고, `transferTo(경로)`로 디스크에 저장함.
* `@RequestParam("file")` → 상자에서 **이름이 `file`인 칸**을 꺼냄. 요청을 보낼 때 쓴 칸 이름(`name="file"`)과 **반드시 같아야** 연결됨.

```java
@RestController
@RequestMapping("/api/v1/files")
public class FileUploadController {

    private final Path uploadRoot;

    public FileUploadController(@Value("${file.upload-dir}") String uploadDir) {
        // 설정값을 정규화된 절대 경로로 바꾸고 모든 업로드의 저장 기준으로 사용
        this.uploadRoot = Paths.get(uploadDir).toAbsolutePath().normalize();
    }

    @PostMapping("/single")
    public ResponseEntity<Map<String, Object>> uploadSingleFile(
            /* FormData의 "file" key와 이름이 같아야 연결된다 */
            @RequestParam("file") MultipartFile file,
            @RequestParam(value="description", required = false) String description) throws IOException {

        FileDTO uploadFile = saveFile(file, "single", description);

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("message", "단일 파일 업로드 성공!");
        response.put("file", uploadFile);

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

* **따라가기**
  1. 서버가 켜질 때 생성자 실행 → `@Value("${file.upload-dir}")`로 `"uploads"`를 받음.
  2. `Paths.get("uploads")` → 경로 객체 → `.toAbsolutePath()` → 컴퓨터 맨 위부터의 전체 주소(예: `D:/lecture/.../chap05-file-upload/uploads`) → `.normalize()` → `a/./b`, `a/x/../b` 같은 군더더기를 정리. 이걸 **모든 업로드의 기준 폴더**(`uploadRoot`)로 기억함.
  3. 단일 업로드 요청 도착 → `file`에는 `sample.txt`, `description`에는 `"단일 파일 설명입니다."`가 들어감. 설명은 `required = false`라 안 보내도 됨.
  4. `saveFile(file, "single", description)` → 실제 저장 (아래에서 자세히).
  5. 응답 → `201 Created`, `{"message":"단일 파일 업로드 성공!","file":{FileDTO 내용}}`

### 실제 저장 과정: saveFile()

* 저장할 때 신경 쓸 것 세 가지
  1. **빈 파일**은 받지 않음.
  2. 이름이 같은 파일이 와도 **덮어쓰지 않도록** 저장 이름을 새로 만듦(UUID).
  3. `../../` 같은 이름으로 **업로드 폴더 밖으로 빠져나가 저장하는 공격**을 막음(경로 조작 공격, Path Traversal).

```java
private FileDTO saveFile(MultipartFile file, String directoryName, String description) throws IOException {

    validateFile(file);

    /* uploads/single 또는 uploads/multiple 처럼 요청 종류별 하위 디렉토리를 만든다 */
    Path uploadPath = uploadRoot.resolve(directoryName).normalize();
    Files.createDirectories(uploadPath);

    // 원본파일 이름 검증
    String originFileName = StringUtils.cleanPath(file.getOriginalFilename());
    if(originFileName.contains("..")) {
        throw new IllegalArgumentException("파일명에 올바르지 않은 경로가 포함되어 있습니다.");
    }

    originFileName = Paths.get(originFileName).getFileName().toString();
    String extension = extractExtension(originFileName);

    // UUID를 사용하여 중복되지 않는 문자를 만든 뒤 , 원본 확장자를 붙여 파일 저장명을 완성한다.
    String savedFilName = UUID.randomUUID().toString().replace("-", "") + extension;

    // 실제 디스크에 쓸 위치
    Path targetPath = uploadPath.resolve(savedFilName).normalize();

    if (!targetPath.startsWith(uploadPath)) {
        throw new IllegalArgumentException("저장할 파일 경로가 올바르지 않습니다.");
    }

    // 실제 파일을 만든다
    file.transferTo(targetPath);

    // 절대 경로 대신 업로드 루트 기준 상대 경로만 응답하여 서버 내부 경로 노출을 줄인다.
    String relativePath = uploadRoot.relativize(targetPath).toString().replace('\\', '/');

    return new FileDTO(
            originFileName,
            savedFilName,
            relativePath,
            description,
            file.getSize(),
            file.getContentType()
    );
}
```

* **따라가기** (`sample.txt`, `directoryName = "single"`)
  1. `validateFile(file)` → 파일이 없거나 비었거나, 원래 이름이 없으면 `IllegalArgumentException`을 던지고 멈춤.
  2. `uploadRoot.resolve("single")` → 기준 폴더 아래 `uploads/single` 경로를 만듦. `resolve`는 "이 폴더 안의 ○○"라는 뜻.
  3. `Files.createDirectories(uploadPath)` → 그 폴더가 없으면 **만들고**, 있으면 그냥 넘어감.
  4. `StringUtils.cleanPath(...)` → 파일 이름의 `\`를 `/`로 바꾸는 등 경로 모양을 정리 → `"sample.txt"`
  5. 이름에 `..`이 들어 있으면 → "위 폴더로 올라가기"를 노리는 이름이라 거절.
  6. `Paths.get(originFileName).getFileName()` → 혹시 `folder/sample.txt`처럼 폴더가 붙어 와도 **파일 이름만** 남김 → `"sample.txt"`
  7. `extractExtension("sample.txt")` → 마지막 점부터 끝까지 → `".txt"`. 점이 없으면 `""`.
  8. `UUID.randomUUID()` → 전 세계에서 겹칠 일이 거의 없는 **무작위 번호표**(예: `3f2a8b1c-...`) → `-`를 지우고 `.txt`를 붙임 → `3f2a8b1c…9c.txt`. 친구 둘이 똑같이 `sample.txt`를 올려도 서로 덮어쓰지 않음.
  9. `uploadPath.resolve(저장이름).normalize()` → 실제로 쓸 위치 `uploads/single/3f2a…9c.txt`
  10. `targetPath.startsWith(uploadPath)` → 최종 위치가 **정말 `uploads/single` 안에 있는지** 한 번 더 확인함. 밖이면 거절. (이중 잠금)
  11. `file.transferTo(targetPath)` → 이때 **진짜 파일이 디스크에 생김**.
  12. `uploadRoot.relativize(targetPath)` → 기준 폴더에서 본 **상대 경로**만 남김 → `single\3f2a…9c.txt` → `\`를 `/`로 바꿔 → `single/3f2a…9c.txt`. 응답에 `D:/lecture/...` 같은 **서버 내부 주소를 드러내지 않기** 위해서임.
  13. 정보들을 `FileDTO`에 담아 돌려줌.

```java
// 확장자 추출
private String extractExtension(String originFileName) {
    int lastDotIndex = originFileName.lastIndexOf(".");

    if (lastDotIndex == -1) {
        return "";
    }

    return originFileName.substring(lastDotIndex);
}
```

* **따라가기**: `"report.final.pdf"` → 마지막 점의 위치 12 → `substring(12)` → `".pdf"` / `"README"` → 점이 없어 `-1` → `""`

### 다중 파일 업로드와 실패 시 정리

* 여러 파일은 `List<MultipartFile>`로 받음. 요청에서 같은 칸 이름(`name="files"`)으로 파일을 여러 개 보내면 리스트에 차례로 담김.
* **문제 상황**: 파일 3개 중 2개는 저장했는데 3번째에서 실패하면, 사용자는 "실패"를 받는데 서버에는 파일 2개가 **반쯤 남아 쓰레기가 됨**.
* **해결**: 중간에 실패하면 **이미 저장한 파일을 지우고**(정리), 원래 예외를 다시 던짐. 레고를 쌓다 무너지면 쌓던 블록을 치우고 "실패했어요"라고 알리는 것과 같음.

```java
@PostMapping("/multiple")
public ResponseEntity<Map<String, Object>> uploadMultipleFiles(
        @RequestParam("files") List<MultipartFile> files,
        @RequestParam(value = "description", required = false) String description) throws IOException{

    if(files == null || files.isEmpty()) {
        throw new IllegalArgumentException("업로드할 파일 목록이 비어 있습니다.");
    }

    List<FileDTO> uploadedFiles = new ArrayList<>();

    try {
        for (MultipartFile file : files) {
            uploadedFiles.add(saveFile(file, "multiple", description));
        }
    } catch (IOException | RuntimeException e) {
        // 중간 실패 시 앞서 저장한 파일을 정리한 뒤 원래 예외를 다시 던진다
        cleanupSavedFiles(uploadedFiles, e);
        throw e;    // 업로드가 실패했다는 사실을 다시 전달
    }
    Map<String, Object> response = new LinkedHashMap<>();
    response.put("message", "다중 파일 업로드 성공");
    response.put("count", uploadedFiles.size());
    response.put("files", uploadedFiles);

    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}
```

```java
// 다중 파일 업로드 중 중간 실패 시, 이미 저장된 파일을 정리하는 메소드
private void cleanupSavedFiles(List<FileDTO> uploadedFiles,
                               Exception originalException) {
    for (FileDTO uploadedFile : uploadedFiles) {
        Path savedPath = uploadRoot.resolve(uploadedFile.getFilePath()).normalize();
        try {
            if (savedPath.startsWith(uploadRoot)) {
                Files.deleteIfExists(savedPath);
            }
        } catch (IOException cleanupException) {
            // 정리 실패가 최초 저장 실패의 원인을 덮어쓰지 않도록 부가 정보로 남긴다
            originalException.addSuppressed(cleanupException);
        }
    }
}
```

* **따라가기 (성공)**: `sample1.txt`, `sample2.txt` → 둘 다 `uploads/multiple/`에 UUID 이름으로 저장 → `201`, `{"message":"다중 파일 업로드 성공","count":2,"files":[{…},{…}]}`
* **따라가기 (2번째에서 실패)**
  1. 1번째 저장 성공 → `uploadedFiles`에 1개.
  2. 2번째가 빈 파일 → `saveFile` 안에서 `IllegalArgumentException`(RuntimeException의 자식) → `catch`로 감.
  3. `cleanupSavedFiles` → 리스트에 있는 1번째 파일의 경로를 다시 만들고, 업로드 폴더 안이 맞으면 `Files.deleteIfExists`로 지움.
  4. 지우다가 또 실패하면 → 그 예외는 `addSuppressed`로 **원래 예외에 덧붙여만** 둠. "왜 업로드가 실패했는지"라는 진짜 원인이 지우기 실패 메시지에 **가려지지 않게** 하려는 것임.
  5. `throw e` → 원래 예외를 다시 던져서 업로드가 실패했음을 알림.

---

## 2. Interceptor (인터셉터)

### 인터셉터가 뭔가

* **한 줄 비유**: 인터셉터는 교실 문 앞의 **선도부**임. 학생(요청)이 교실(Controller)에 들어가기 **전에** 명찰을 확인하고, 수업이 **끝난 뒤에도** 기록을 남길 수 있음.
* **왜 쓰나**: "모든 API의 처리 시간 재기", "관리자 API는 키 확인하기" 같은 일을 Controller 메소드마다 복사해 넣으면 중복이 생기고 빠뜨리기 쉬움 → **한 곳에서** 여러 URL에 한꺼번에 적용함.
* **`HandlerInterceptor`의 세 시점**

| 메소드 | 언제 | 비유 |
| :--- | :--- | :--- |
| `preHandle` | Controller **실행 전** | 교실 들어가기 전 명찰 확인 |
| `postHandle` | Controller가 **정상적으로 끝난 뒤** (예외가 나면 호출 안 됨) | 수업 끝나고 바로 |
| `afterCompletion` | 응답까지 **모두 끝난 뒤** (예외가 나도 호출됨) | 하교 기록 |

* **`preHandle`의 반환값이 중요함**
  * `true` → "통과" → 다음 인터셉터나 Controller로 넘어감.
  * `false` → "출입 금지" → Controller를 **실행하지 않고** 여기서 끝냄. 이때 응답은 인터셉터가 직접 만들어야 함.

### StopwatchInterceptor: 처리 시간 재기

```java
@Component
public class StopwatchInterceptor implements HandlerInterceptor {

    private static final String START_TIME = "startTime";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        long startTime = System.nanoTime();
        request.setAttribute(START_TIME, startTime);

        System.out.println("[Stopwatch] preHandle "
                + request.getMethod() + " " + request.getRequestURI());

        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {

        Object startTimeAttribute = request.getAttribute(START_TIME);

        if (startTimeAttribute instanceof Long startTime) {
            long elapsedTime = (System.nanoTime() - startTime) / 1_000_000;

            System.out.println("[Stopwatch] afterCompletion "
                + request.getMethod() + " " + request.getRequestURI() + " " + elapsedTime +"ms");
        }
    }
}
```

* **왜 시작 시각을 `request`에 넣나**: 인터셉터 bean은 **singleton**이라 모든 요청이 같은 객체를 씀. 필드에 시작 시각을 저장하면 동시에 들어온 요청끼리 값을 덮어씀. `request.setAttribute`로 **요청마다 따로 있는 가방**에 넣어두면 안전함.
* **따라가기: `GET /api/v1/menus`**
  1. `preHandle` → `System.nanoTime()`(아주 정밀한 시계)으로 시작 시각을 재서 요청 가방에 `startTime`으로 넣음 → `[Stopwatch] preHandle GET /api/v1/menus` 출력 → `true`로 통과.
  2. Controller 실행 → `/menus 핸들러 메소드 동작` 출력 → 시간이 보이도록 일부러 `Thread.sleep(300)`(0.3초 멈춤) → 메뉴 목록 응답.
  3. `afterCompletion` → 가방에서 시작 시각을 꺼냄 → `instanceof Long startTime`으로 "Long이 맞으면 `startTime`이라는 이름으로 바로 꺼내 쓰기"(패턴 매칭).
  4. `(지금 - 시작) / 1_000_000` → 나노초(10억분의 1초)를 밀리초(1000분의 1초)로 바꿈 → `[Stopwatch] afterCompletion GET /api/v1/menus 302ms`처럼 출력. (`1_000_000`의 `_`는 읽기 쉽게 끊어 쓴 것일 뿐 값은 1000000)

### ApiKeyInterceptor: 관리자 API 막기

* 관리자 API는 요청 헤더에 **`X-API-KEY: secret-key`** 라는 열쇠가 있어야 통과시킴.

```java
@Component
public class ApiKeyInterceptor implements HandlerInterceptor {

    private static final String API_KEY_HEADER = "X-API-KEY";
    private static final String VALID_API_KEY = "secret-key";

    private final ObjectMapper objectMapper;

    public ApiKeyInterceptor(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        String apiKey = request.getHeader(API_KEY_HEADER);

        if(VALID_API_KEY.equals(apiKey)) {
            System.out.println("[ApiKey] authorized "
                    + request.getMethod() + " " + request.getRequestURI());
            return true;
        }

        System.out.println("[ApiKey] unauthorized "
                + request.getMethod() + " " + request.getRequestURI());

        // false로 Controller를 막을 것이므로 상태,형식,본문을 여기서 직접 작성
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");

        Map<String, Object> errorResponse = new LinkedHashMap<>();
        errorResponse.put("status", HttpStatus.UNAUTHORIZED.value());
        errorResponse.put("error", HttpStatus.UNAUTHORIZED.name());
        errorResponse.put("message", "관리자 API를 호출하려면 X-API-KEY 헤더가 필요합니다.");
        errorResponse.put("path", request.getRequestURI());

        // ObjectMapper로 오류 정보를 JSON문자열로 직렬화한다.
        // getWriter()로 얻은 writer가 그 문자열을 HTTP응답 본문에 쓴다
        response.getWriter().write(objectMapper.writeValueAsString(errorResponse));

        return false;
    }
}
```

* **따라가기 (열쇠 없음)**
  1. `request.getHeader("X-API-KEY")` → 헤더가 없으니 `null`.
  2. `"secret-key".equals(null)` → `false` (리터럴 쪽에서 `equals`를 불러서 `null`이어도 에러가 안 남).
  3. `[ApiKey] unauthorized POST /api/v1/admin/menus` 출력.
  4. Controller가 실행되지 않으므로 응답을 **직접** 채움: 상태 코드 `401`(인증 필요), 형식 `application/json`, 글자 규칙 UTF-8(한글 깨짐 방지).
  5. 오류 정보 Map → `ObjectMapper`로 JSON 글자로 바꿈 → `response.getWriter().write(...)`로 응답 본문에 씀.
  6. `return false` → Controller로 가지 않고 끝.
  7. 받은 응답 → `401` `{"status":401,"error":"UNAUTHORIZED","message":"관리자 API를 호출하려면 X-API-KEY 헤더가 필요합니다.","path":"/api/v1/admin/menus"}`
* **따라가기 (열쇠 있음)**: 헤더 값이 `secret-key` → `[ApiKey] authorized ...` → `true` → Controller가 메뉴를 등록하고 `201` 응답.
* `ObjectMapper`는 Spring Boot가 bean으로 만들어 둔 것을 **생성자 주입**으로 받음 (Jackson 3이라 `tools.jackson.databind` 패키지).

### 인터셉터 등록: WebMvcConfigurer

* 인터셉터 클래스를 만들기만 하면 동작하지 않음 → **`WebMvcConfigurer`의 `addInterceptors()`** 에서 "어느 URL에, 어떤 순서로" 적용할지 **등록**해야 함.

```java
@Configuration
public class WebConfiguration implements WebMvcConfigurer {

    private final StopwatchInterceptor stopwatchInterceptor;
    private final ApiKeyInterceptor apiKeyInterceptor;

    public WebConfiguration(StopwatchInterceptor stopwatchInterceptor, ApiKeyInterceptor apiKeyInterceptor) {
        this.stopwatchInterceptor = stopwatchInterceptor;
        this.apiKeyInterceptor = apiKeyInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(stopwatchInterceptor)
                .addPathPatterns("/api/v1/**")
                .order(1);

        registry.addInterceptor(apiKeyInterceptor)
                .addPathPatterns("/api/v1/admin/**")
                .order(2);
    }
}
```

* **한 줄씩 보기**
  * `addPathPatterns("/api/v1/**")` → `**`는 "그 아래 **모든** 경로". `/api/v1/menus`, `/api/v1/menus/1`, `/api/v1/admin/menus` 모두 해당.
  * `addPathPatterns("/api/v1/admin/**")` → 관리자 경로에만 적용.
  * `order(1)`, `order(2)` → 숫자가 **작은 것부터** `preHandle`이 실행됨.

| 요청 | Stopwatch | ApiKey | 결과 |
| :--- | :--- | :--- | :--- |
| `GET /api/v1/menus` | O | X | 200, 시간 측정 |
| `GET /api/v1/menus/1` | O | X | 200, 시간 측정 |
| `POST /api/v1/admin/menus` (키 없음) | O | O → `false` | 401, Controller 실행 안 됨 |
| `POST /api/v1/admin/menus` (키 있음) | O | O → `true` | 201 |

* **키 없이 관리자 API를 부르면 콘솔 출력 순서**
  1. `[Stopwatch] preHandle POST /api/v1/admin/menus` (order 1, `true`)
  2. `[ApiKey] unauthorized POST /api/v1/admin/menus` (order 2, `false` → 여기서 멈춤)
  3. `[Stopwatch] afterCompletion POST /api/v1/admin/menus 1ms` → `preHandle`이 `true`였던 인터셉터는 뒤에서 `false`가 나와도 `afterCompletion`이 불림. 그래서 막힌 요청도 시간이 기록됨.

---

## 3. React와 Spring 연동 (CORS)

### 서버 두 개가 따로 돈다

* **백엔드(Spring)**: `http://localhost:8080` — 메뉴 데이터를 JSON으로 주고받음.
* **프론트엔드(Next.js)**: `http://localhost:3000` — 화면을 그리고, 버튼을 누르면 8080에 요청을 보냄.
* 식당으로 치면 **홀(3000)** 과 **주방(8080)** 이 다른 건물에 있고, 홀 직원이 주문서를 들고 주방에 가는 구조임.

### CORS가 뭔가

* **출처(Origin)**: `프로토콜 + 주소 + 포트` 묶음. `http://localhost:3000`과 `http://localhost:8080`은 포트가 달라서 **다른 출처**임.
* **같은 출처 정책(Same-Origin Policy)**: 브라우저는 보안을 위해, 한 웹 페이지가 **다른 출처의 서버**에서 받은 응답을 자바스크립트가 마음대로 읽지 못하게 막음. 남의 집 우편함을 함부로 열지 못하게 하는 규칙과 같음.
* **CORS (Cross-Origin Resource Sharing)**: 서버가 응답에 **"3000번에서 온 요청은 읽어도 돼"라는 허가증**(`Access-Control-Allow-Origin` 헤더)을 붙여주는 약속. 허가증이 없으면 서버는 응답을 보냈어도 **브라우저가 자바스크립트에게 전달하지 않고** 콘솔에 CORS 오류를 띄움.
* **Preflight(사전 요청)**: `Content-Type: application/json` 같은 요청을 보내기 전에, 브라우저가 먼저 **`OPTIONS` 메소드로 "이런 요청 보내도 돼요?"라고 물어봄**. 서버가 허락해야 진짜 `POST`를 보냄. 그래서 허용 메소드에 `OPTIONS`가 들어감.
* `api-test.http`나 Postman은 브라우저가 아니라서 CORS 검사를 하지 않음 → 거기서는 잘 되는데 화면에서만 실패하면 CORS를 의심해야 함.

### Spring에서 CORS 허용하기

```java
@Configuration
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
       registry.addMapping("/api/**")
               .allowedOrigins("http://localhost:3000")
               .allowedMethods("GET", "POST", "OPTIONS")
               .allowedHeaders("Content-Type");
    }
}
```

* **한 줄씩 보기**
  * `addMapping("/api/**")` → `/api`로 시작하는 모든 주소에 CORS 규칙을 적용.
  * `allowedOrigins("http://localhost:3000")` → 이 출처에서 온 요청만 허락. 아무 데서나 허락하는 `*`보다 안전함.
  * `allowedMethods("GET", "POST", "OPTIONS")` → 조회·등록과 사전 요청만 허락. (`PUT`, `DELETE`는 아직 막혀 있음)
  * `allowedHeaders("Content-Type")` → 요청에 붙여도 되는 헤더. JSON을 보내려면 `Content-Type`이 필요함.
  * 인터셉터 등록과 같은 `WebMvcConfigurer`를 쓰지만, 이번에는 `addCorsMappings()`를 오버라이딩함.

### 메뉴 API (요청 DTO와 응답 DTO 분리)

* **`MenuRequest`**: 클라이언트가 **보내는** 값(이름, 가격)만 담음. 번호는 서버가 정하므로 없음.
* **`MenuResponse`**: 서버가 **돌려주는** 값(번호, 이름, 가격). 필드가 `final`이라 만든 뒤 바뀌지 않음.
* **왜 나누나**: 하나의 DTO로 같이 쓰면 사용자가 `code`를 마음대로 보내 번호를 조작할 수 있음. 들어오는 문과 나가는 문을 따로 두는 것과 같음.

```java
@RestController
@RequestMapping("/api/v1/menus")
public class MenuController {

    private final List<MenuResponse> menus = new ArrayList<>();
    private int nextCode = 3;

    public MenuController() {
        menus.add(new MenuResponse(1, "아메리카노", 3000));
        menus.add(new MenuResponse(2, "카페라떼", 4000));
    }

    @GetMapping
    public ResponseEntity<List<MenuResponse>> findAllMenus() {
        return ResponseEntity.ok(menus);
    }

    @PostMapping
    public ResponseEntity<MenuResponse> registMenu (
        @RequestBody MenuRequest request) {

        MenuResponse createMenu = new MenuResponse(
                nextCode++,
                request.getName(),
                request.getPrice()
        );
        menus.add(createMenu);

        return ResponseEntity.status(HttpStatus.CREATED).body(createMenu);
    }
}
```

* **따라가기: `POST {"name":"바닐라라떼","price":4500}`**
  1. JSON → `MenuRequest`(`name`, `price`)로 변환.
  2. `nextCode++` → 지금 값 3을 **먼저 쓰고** 그다음 4로 올림(후위 증가) → 새 메뉴 번호 3.
  3. `new MenuResponse(3, "바닐라라떼", 4500)` → 목록에 추가.
  4. 응답 → `201` `{"code":3,"name":"바닐라라떼","price":4500}`. 다음 등록은 번호 4.
  * DB 없이 메모리 리스트를 쓰므로 서버를 껐다 켜면 처음 2개로 돌아감.

### Next.js 화면에서 API 호출하기

* **환경 변수**: 서버 주소를 코드에 박지 않고 `react-client/.env.local` 파일에 `NEXT_PUBLIC_API_BASE_URL`로 적어둠. 이름이 **`NEXT_PUBLIC_`으로 시작해야** 브라우저 코드에서 `process.env.NEXT_PUBLIC_API_BASE_URL`로 읽을 수 있음.

```jsx
"use client";

import { useState, useEffect } from "react";

const API_BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL;

export default function Home() {
  const [menus, setMenus] = useState([]);
  const [name, setName] = useState("");
  const [price, setPrice] = useState("");
  const [error, setError] = useState("");
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
      async function loadMenus() {
          try{
              const response = await fetch(`${API_BASE_URL}/api/v1/menus`);
              if(!response.ok) {
                  throw new Error("메뉴를 불러오지 못했습니다.");
              }
              const data = await response.json();
              setMenus(data);
          } catch (error) {
            setError(error.message);
          } finally {
              setIsLoading(false);
          }
      }
      loadMenus();
  }, []);

  async function handleSubmit(event) {
      event.preventDefault(); // form 기본 새로고침 막기
      setError("");

      try{
          const response = await fetch(`${API_BASE_URL}/api/v1/menus`,{
              method:"POST",
              headers: {"Content-Type": "application/json"},
              body: JSON.stringify({name, price: Number(price)})
          })
          if(!response.ok){
              throw new Error("메뉴를 등록하지 못했습니다");
          }
          const createMenu = await response.json();
          // 서버가 돌려준 새 메뉴를 기존 배열 뒤에 추가한 새 배열로 상태를 갱신한다
          setMenus((currentMenus) =>[...currentMenus, createMenu]);
          setName("");
          setPrice("");
      } catch (error) {
          setError(error.message);
      }
  }

  if (isLoading) return <p>메뉴를 불러오는 중입니다.</p>;

  return (
    <main>
      <h1>메뉴 관리</h1>

        {/* error가 빈 문자열이 아닐 때만 오류 문장 렌더링 */}
        {error && <p>{error}</p>}

      <ul>
        {menus.map((menu) => (
          <li key={menu.code}>
            {menu.name} - {menu.price.toLocaleString()}원
          </li>
        ))}
      </ul>

      <form onSubmit={handleSubmit}>
        <input
          value={name}
          onChange={(event) => setName(event.target.value)}
          placeholder="메뉴명"
          required
        />
        <input
          type="number"
          value={price}
          onChange={(event) => setPrice(event.target.value)}
          placeholder="가격"
          min="0"
          required
        />
        <button type="submit">등록</button>
      </form>
    </main>
  );
}
```

* **따라가기: 처음 화면을 열 때**
  1. `isLoading`이 `true` → `메뉴를 불러오는 중입니다.`
  2. 화면이 그려진 뒤 `useEffect`(빈 배열이라 한 번만) → `fetch("http://localhost:8080/api/v1/menus")`
  3. 브라우저가 요청을 보내면서 `Origin: http://localhost:3000`을 붙임 → Spring이 CORS 설정을 보고 응답에 허가증을 붙임 → 브라우저가 응답을 자바스크립트에 넘겨줌.
  4. `response.json()` → `[{code:1,…},{code:2,…}]` → `setMenus`
  5. `finally` → `setIsLoading(false)` → 다시 그리기 → `아메리카노 - 3,000원`, `카페라떼 - 4,000원` (`toLocaleString()`이 천 단위 쉼표를 찍어줌)
* **따라가기: "바닐라라떼", "4500" 입력 후 등록**
  1. `event.preventDefault()` → `<form>`이 페이지를 새로고침하지 않게 막음. 이전 오류 문구도 지움.
  2. 입력칸의 값은 항상 **글자**라서 `Number(price)`로 숫자 4500으로 바꿈 → `JSON.stringify({name, price})` → `{"name":"바닐라라떼","price":4500}`
  3. `Content-Type: application/json`이라 브라우저가 먼저 **`OPTIONS` 사전 요청** → 서버가 허락 → 진짜 `POST`를 보냄.
  4. 서버가 돌려준 `{code:3,…}`을 `setMenus(현재목록 => [...현재목록, 새메뉴])`로 **새 배열**을 만들어 넣음(불변성) → 목록 맨 아래에 `바닐라라떼 - 4,500원`이 생김.
  5. 입력칸 두 개를 비움.
  * `required`, `min="0"` → 빈칸이거나 음수면 브라우저가 제출 자체를 막아줌(서버에 가기 전 1차 검사).

> ⚠️ 실습 코드에서는 `handleSubmit`의 `catch`가 `catch {`로 되어 있어서, 안에서 쓰는 `error`가 예외 객체가 아니라 **state 변수 `error`(빈 문자열)** 를 가리킴. 그러면 `"".message`는 `undefined`라 오류 문구가 표시되지 않음. 위 코드처럼 `catch (error)`로 예외를 받아야 `메뉴를 등록하지 못했습니다`가 화면에 나옴. 또 같은 이름이 겹쳐 헷갈리지 않게 예외 변수 이름을 `err`처럼 다르게 짓는 편이 안전함.

---

## 📝 오늘의 정리

1. 파일은 `multipart/form-data`로 보내고, 서버는 `MultipartFile`로 받아 `transferTo()`로 저장함
2. 저장할 때는 빈 파일 검사, UUID 저장 이름, `..`·`startsWith` 경로 검사로 덮어쓰기와 경로 조작 공격을 막고, 응답에는 상대 경로만 보냄
3. 다중 업로드 중 실패하면 이미 저장한 파일을 지우고 원래 예외를 다시 던짐 (`addSuppressed`로 원인 보존)
4. 인터셉터는 `preHandle`(실행 전, `false`면 차단) · `postHandle` · `afterCompletion`(항상 마지막) 시점에 공통 작업을 끼워 넣고, `addInterceptors()`에서 URL 패턴과 순서를 등록함
5. 포트가 다른 React(3000)에서 Spring(8080)을 부르려면 서버에서 `addCorsMappings()`로 출처·메소드·헤더를 허용해야 하고, JSON `POST`는 `OPTIONS` 사전 요청을 먼저 거침

---

## 느낀 점

* 파일 하나 저장하는 데도 빈 파일, 이름 중복, `../` 경로 조작, 내부 경로 노출까지 신경 써야 한다는 점에서, 사용자가 보낸 값은 파일 이름조차 믿으면 안 된다는 것을 실감함
* 다중 업로드 실패 시 이미 저장한 파일을 정리하고 원래 예외를 다시 던지는 흐름을 보며, "실패했을 때 서버에 무엇이 남는가"까지 생각해야 한다는 것을 배움
* 인터셉터로 시간 측정과 API 키 검사를 Controller 밖으로 빼내니, Controller는 메뉴 로직만 남아 훨씬 깔끔해짐. `preHandle`이 `false`면 응답을 직접 만들어야 한다는 점도 기억해야겠음
* `api-test.http`에서는 잘 되던 요청이 브라우저에서는 CORS로 막힌다는 것을 알고 나니, CORS가 서버가 아니라 브라우저의 보안 규칙이라는 점이 이해됨
* React에서 배운 `useEffect`, `useState`, 불변성 갱신이 실제 Spring API와 연결되어 동작하는 것을 보니 지금까지 배운 프론트엔드와 백엔드가 하나로 이어지는 느낌이 들었음
