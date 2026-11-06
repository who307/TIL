# Spring Security 시작: 인증·인가 · SecurityFilterChain · PasswordEncoder(BCrypt) · UserDetailsService · HTTP Basic 정리 (TIL)

> 실습 환경: MySQL (`securitydb`의 `tbl_user_account`), Spring Boot 4 + `spring-boot-starter-security`·`data-jpa`·`validation`·`webmvc` (Gradle 프로젝트 `chap01-security-basic`), 테스트는 H2 인메모리 DB, `.http` 파일로 API 호출
>
> 오늘은 회원 가입 서비스·로그인 사용자 조회·보안 설정까지 작성함. 컨트롤러(`/api/auth/signup`, `/api/users/me`, `/api/admin/dashboard`)와 테스트는 다음 시간에 이어서 진행

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| 개념 | **인증**(누구냐) vs **인가**(무엇을 할 수 있냐), 401 vs 403 |
| 준비 | `securitydb`, `UserAccount` 엔티티, `UserRole` enum, `SignupRequest`·`UserResponse` record |
| 회원 가입 | 중복 검사 + **`PasswordEncoder`로 비밀번호 해시**해서 저장 |
| 사용자 조회 | **`UserDetailsService`** 구현: 아이디로 회원을 찾아 Security가 읽는 `UserDetails`로 바꿔 줌 |
| 보안 설정 | **`SecurityFilterChain`**: CSRF 끄기, 세션 안 쓰기(STATELESS), URL별 권한, HTTP Basic 로그인 |

---

## 1. 인증과 인가

* **한 줄 비유**: 회사 건물에 들어갈 때
  * **인증**(Authentication) = 출입문에서 **사원증을 확인**하는 것 → "당신 누구세요?"
  * **인가**(Authorization) = 들어온 뒤 **서버실 문을 열 권한**이 있는지 확인하는 것 → "당신 여기 들어와도 돼요?"
* 순서가 있음: 인증이 먼저, 인가는 그다음.

| 상태 코드 | 뜻 | 실습 예 |
| :--- | :--- | :--- |
| `401 Unauthorized` | **인증 실패**: 누군지 모름 (로그인 안 함, 비밀번호 틀림) | 아이디·비밀번호 없이 `GET /api/users/me` |
| `403 Forbidden` | **인가 실패**: 누군지는 알지만 **권한이 없음** | `USER` 권한으로 `GET /api/admin/dashboard` |

* 이름이 `Unauthorized`라서 헷갈리지만 401은 "**인증**이 안 됨"이라는 뜻.

### Spring Security는 필터

```text
요청 ──▶ [ Security 필터들 ] ──▶ DispatcherServlet ──▶ Controller
           ├ 인증: 아이디/비밀번호 확인        (실패 → 401)
           └ 인가: 이 URL에 접근할 권한 확인   (실패 → 403)
```

* Spring Security는 **컨트롤러에 도착하기 전**에 요청을 검사하는 **필터 묶음**(Filter Chain). 10/29의 Interceptor보다 더 앞, DispatcherServlet 바깥에서 동작함.
* 의존성만 추가해도 **모든 URL이 잠김** → 기본 로그인 화면과 콘솔에 임시 비밀번호가 찍힘. 그래서 아래 `SecurityConfig`로 "무엇을 열고 무엇을 잠글지" 직접 정함.

---

## 2. 준비: DB와 회원 엔티티

### application.yml

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/securitydb?createDatabaseIfNotExist=true&characterEncoding=UTF-8&serverTimezone=Asia/Seoul
    username: 사용자명
    password: 비밀번호
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.springframework.security: debug
```

* `createDatabaseIfNotExist=true` → `securitydb`가 없으면 MySQL 드라이버가 **DB를 만들어 줌** (해당 계정에 권한이 있을 때). 수업에서는 `sql/01_securitydb.sql`로 DB 생성 + 권한 부여(`GRANT ALL PRIVILEGES ON securitydb.*`)를 먼저 실행함.
* `ddl-auto: update` → 연습용이라 엔티티를 보고 `tbl_user_account`를 자동으로 만들게 둠 (11/03 `ddl-auto` 표 참고, 실무에선 `validate`/`none`).
* `logging.level.org.springframework.security: debug` → 요청마다 **어떤 Security 필터를 거쳤고 왜 막혔는지** 콘솔에 자세히 찍힘. 401/403 원인 찾을 때 유용.

### 테스트용 설정: H2 인메모리 DB

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:securitydb;MODE=MySQL;DB_CLOSE_DELAY=-1
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
```

```gradle
testRuntimeOnly 'com.h2database:h2'
```

* **H2** → 자바로 만든 가벼운 DB. `mem:` → 디스크가 아니라 **메모리**에만 만들어서, 테스트가 끝나면 사라짐 → MySQL 데이터를 더럽히지 않음.
* `MODE=MySQL` → MySQL 문법에 최대한 맞춰서 동작.
* `DB_CLOSE_DELAY=-1` → 연결이 잠깐 다 끊겨도 앱이 살아 있는 동안 DB를 **지우지 않음**.
* `application-test.yml` → `test` **프로필**용 설정. 테스트 클래스에 `@ActiveProfiles("test")`를 붙이면 기본 `application.yml` 위에 이 값이 **덮어씌워짐**.

### UserAccount 엔티티

```java
@Entity
@Table(name = "tbl_user_account")
public class UserAccount {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String userId;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false, length = 50)
    private String userName;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private UserRole role;

    @Column(nullable = false)
    private boolean enabled;

    protected UserAccount() {}

    public UserAccount(String userId, String password, String userName,
                       String email, UserRole role) {
        ...
        this.enabled = true;
    }
    // getter만
}
```

```java
public enum UserRole {
    USER,
    ADMIN
}
```

* `id`(DB용 번호)와 `userId`(로그인 아이디)를 **따로** 둠 → 로그인 아이디는 사람이 정하는 값이라 PK로 쓰지 않음.
* `userId`, `email`에 `unique = true` → DB 차원에서도 중복 가입을 막음.
* `password`는 길이를 따로 안 정함(기본 255) → 아래 BCrypt 해시는 **60자**라서 `length = 50` 같은 걸 붙이면 저장이 안 됨.
* `enabled` → 계정 **사용 가능 여부**. 가입하면 `true`. 나중에 정지시키면 `false` → Security가 로그인 자체를 막음.
* `role`은 11/03 배운 대로 `EnumType.STRING`으로 저장 (`"USER"`).

### 요청·응답 DTO: record + 검증

```java
public record SignupRequest(
        @NotBlank @Size(max = 50) String userId,
        @NotBlank @Size(min = 8, max = 100) String password,
        @NotBlank @Size(max = 50) String userName,
        @NotBlank @Email @Size(max = 100) String email
) {}
```

```java
public record UserResponse(Long id, String userId, String userName, String email, String role) {
    public static UserResponse from(UserAccount user) {
        return new UserResponse(user.getId(), user.getUserId(), user.getUserName(),
                user.getEmail(), user.getRole().name());
    }
}
```

* `record`의 각 필드에 검증 어노테이션을 바로 붙일 수 있음. `@Size` 최대 길이를 엔티티의 `length`와 **맞춰 둠** → DB에서 터지기 전에 400으로 막음.
* `@Email` → 이메일 형식 검사. `@Email`도 `null`은 통과시키니 `@NotBlank`와 같이 씀 (11/04 내용).
* **응답에 `password`가 없음** → 해시라도 비밀번호 관련 값은 절대 밖으로 내보내지 않음.

### Repository: 쿼리 메소드

```java
public interface UserAccountRepository extends JpaRepository<UserAccount, Long> {

    Optional<UserAccount> findByUserId(String userId);

    boolean existsByUserId(String userId);

    boolean existsByEmail(String email);
}
```

* `existsBy...` → 있으면 `true`/없으면 `false`. 엔티티를 통째로 가져오지 않고 **있는지만** 확인해서 가벼움 (11/04 쿼리 메소드 규칙).

---

## 3. 회원 가입: PasswordEncoder로 비밀번호 해시

### 비밀번호는 왜 그대로 저장하면 안 되나

* DB가 털리거나 직원이 DB를 열어 보면 **모든 회원 비밀번호가 그대로** 보임. 사람들은 여러 사이트에 같은 비밀번호를 쓰니 피해가 다른 곳까지 번짐.
* 그래서 **해시**(hash)로 바꿔서 저장함.
  * **해시** = 원래 값을 알아볼 수 없는 문자열로 바꾸는 **한 방향** 변환. 해시 값에서 원래 비밀번호를 **거꾸로 계산할 수 없음**.
  * **한 줄 비유**: 고기를 **갈아서 다진 고기**로 만드는 것. 다진 고기에서 원래 고깃덩어리로 되돌릴 수는 없지만, 같은 고기를 같은 방식으로 갈면 같은 결과인지 **비교**는 할 수 있음.
* 로그인할 때는 입력한 비밀번호를 같은 방식으로 해시해서 **저장된 해시와 비교**함.

### BCryptPasswordEncoder

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

| 메소드 | 하는 일 |
| :--- | :--- |
| `encode(원래 비밀번호)` | 해시를 만듦 → DB에 저장 |
| `matches(입력값, 저장된 해시)` | 입력값이 그 해시와 맞는지 `true`/`false` |

* `PasswordEncoder` → "해시 만들기 + 비교하기" 규칙(인터페이스). `BCryptPasswordEncoder` → 그 규칙을 **BCrypt 알고리즘**으로 구현한 클래스.
* **BCrypt의 특징**
  * **솔트(salt)**: 해시할 때마다 랜덤 값을 섞음 → 같은 `pass1234`도 `encode`할 때마다 **결과가 다름**. 그래서 "자주 쓰는 비밀번호의 해시 목록"을 미리 만들어 두고 맞춰 보는 공격이 안 통함.
  * 솔트는 결과 문자열 안에 같이 들어 있어서 `matches`가 그걸 꺼내 같은 방식으로 비교함.
  * 일부러 **느리게** 계산함 → 비밀번호를 하나하나 대입해 보는 공격이 오래 걸림.
* 결과 예: `$2a$10$N9qo8uLOickgx2ZMRZoMye...` (60자). `$2a$` = BCrypt, `10` = 계산 강도.
* `@Bean`으로 등록해 두면 서비스와 Security 양쪽에서 **같은 인코더**를 주입받아 씀.

### UserAccountService.signup

```java
@Service
public class UserAccountService {

    private final UserAccountRepository userAccountRepository;
    private final PasswordEncoder passwordEncoder;
    ...
    @Transactional
    public UserResponse signup(SignupRequest request) {
        if (userAccountRepository.existsByUserId(request.userId())) {
            throw new IllegalStateException("이미 사용 중인 아이디입니다.");
        }
        if (userAccountRepository.existsByEmail(request.email())) {
            throw new IllegalStateException("이미 사용 중인 이메일입니다.");
        }

        UserAccount user = new UserAccount(
                request.userId(),
                passwordEncoder.encode(request.password()),
                request.userName(),
                request.email(),
                UserRole.USER
        );

        return UserResponse.from(userAccountRepository.save(user));
    }
}
```

* **따라가기**
  1. 아이디 중복? → 예외
  2. 이메일 중복? → 예외
  3. `passwordEncoder.encode("pass1234")` → `$2a$10$...` 해시
  4. 권한은 요청에서 받지 않고 서버가 **`UserRole.USER`로 고정** → 가입 요청에 `"role": "ADMIN"`을 끼워 넣어도 관리자가 될 수 없음
  5. 저장 후 비밀번호 없는 `UserResponse`로 반환
* 미리 `exists`로 확인하는 건 **친절한 에러 메시지**용. 동시에 두 명이 같은 아이디로 가입하는 순간까지는 못 막으니, 최종 방어는 DB의 `unique` 제약조건이 맡음.

> ⚠️ 중복일 때 `IllegalStateException`을 던지는데, `GlobalExceptionHandler`는 **`IllegalArgumentException`** 만 `409 Conflict`로 바꿔 줌 → 지금 코드로는 중복 가입이 의도한 409로 응답되지 않음. 다음 챕터 코드처럼 `IllegalArgumentException`으로 맞추거나, 핸들러에 `IllegalStateException`을 추가해야 함.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, String>> handleIllegalArgument(IllegalArgumentException exception) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(Map.of("message", exception.getMessage()));
    }
}
```

---

## 4. UserDetailsService: Security에게 회원 정보 알려 주기

* Security는 우리 `UserAccount` 엔티티를 **모름**. 로그인 요청이 오면 "이 아이디의 비밀번호 해시랑 권한 좀 줘"라고 **정해진 창구**에 물어봄. 그 창구가 `UserDetailsService`.
* **한 줄 비유**: Security는 **경비원**, `UserDetailsService`는 **인사팀**. 경비원은 사원 명부가 어디 있는지 모르니, 아이디를 주면 인사팀이 **표준 양식의 사원 카드**(`UserDetails`)를 만들어 건네줌.

| 이름 | 역할 |
| :--- | :--- |
| `UserDetailsService` | 아이디를 받아 사용자 정보를 돌려주는 **인터페이스**. 메소드는 `loadUserByUsername` 하나 |
| `UserDetails` | Security가 읽는 사용자 정보 **공통 규격** (아이디, 비밀번호 해시, 권한, 사용 가능 여부 등) |
| `User` | `UserDetails`를 Security가 미리 구현해 둔 클래스. 빌더로 쉽게 만듦 |

```java
@Service
public class CustomUserDetailService implements UserDetailsService {

    private final UserAccountRepository userAccountRepository;
    ...
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        UserAccount user = userAccountRepository.findByUserId(username)
                .orElseThrow(() -> new UsernameNotFoundException("회원을 찾을 수 없습니다."));

        return User.withUsername(user.getUserId())
                .password(user.getPassword())
                .roles(user.getRole().name())
                .disabled(!user.isEnabled())
                .build();
    }
}
```

* `username` → Security 용어로 "로그인 아이디". 우리 엔티티에서는 `userId`.
* 회원이 없으면 `UsernameNotFoundException` (Security가 정한 예외).
* `.password(...)` → DB의 **해시**를 그대로 넘김. 비교는 이 메소드가 아니라 아래 `DaoAuthenticationProvider`가 `PasswordEncoder.matches`로 함.
* `.roles("USER")` → Security 안에서는 권한 이름 앞에 **`ROLE_`이 자동으로 붙어** `ROLE_USER`가 됨. 그래서 enum은 `USER`/`ADMIN`으로만 써도 됨.
* `.disabled(!user.isEnabled())` → `enabled = false`인 계정은 비밀번호가 맞아도 로그인 실패.

> 💡 아이디가 없을 때와 비밀번호가 틀릴 때 Security는 둘 다 똑같이 **401(아이디 또는 비밀번호가 틀림)** 으로 응답함. "회원을 찾을 수 없습니다"를 그대로 보여 주면 공격자가 **어떤 아이디가 가입돼 있는지** 알아낼 수 있어서, 기본적으로 이 차이를 숨김.

---

## 5. SecurityConfig: 무엇을 열고 무엇을 잠글지

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationProvider authenticationProvider(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder
    ) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return provider;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http,
                                                   AuthenticationProvider authenticationProvider) {
        http.csrf(AbstractHttpConfigurer::disable)
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(authorize -> authorize
                        .requestMatchers(HttpMethod.POST, "/api/auth/signup").permitAll()
                        .requestMatchers("/api/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated())
                .httpBasic(Customizer.withDefaults())
                .authenticationProvider(authenticationProvider);

        return http.build();
    }
}
```

### AuthenticationProvider: 실제로 비밀번호를 확인하는 곳

* `DaoAuthenticationProvider` → "DB(DAO)에서 사용자를 꺼내 비밀번호를 비교하는" 인증 담당.
  1. `userDetailsService.loadUserByUsername(아이디)` → `UserDetails` 받기
  2. `passwordEncoder.matches(입력 비밀번호, 저장된 해시)` → 맞으면 인증 성공
* 생성자에 `UserDetailsService`, setter로 `PasswordEncoder`를 넣어 **우리 인사팀 + 우리 해시 방식**을 쓰게 연결함.
* `@Bean` 메소드의 매개변수(`UserDetailsService`, `PasswordEncoder`)는 Spring이 컨테이너에서 찾아 **자동 주입**함 → `CustomUserDetailService`(`@Service`)와 위의 `passwordEncoder()` Bean이 들어감.

### SecurityFilterChain 한 줄씩

| 설정 | 뜻 | 왜 |
| :--- | :--- | :--- |
| `csrf(disable)` | CSRF 보호 끄기 | CSRF는 **브라우저 쿠키(세션)** 로 자동 로그인되는 걸 악용하는 공격. 이 API는 쿠키 세션을 안 쓰고 요청마다 인증 정보를 직접 보내므로 끔 |
| `sessionCreationPolicy(STATELESS)` | 서버에 로그인 **세션을 만들지 않음** | 요청마다 스스로 인증하는 REST API 방식. 서버가 "누가 로그인했는지" 기억하지 않음 |
| `requestMatchers(POST, "/api/auth/signup").permitAll()` | 회원 가입은 **누구나** | 가입하려는 사람은 아직 계정이 없으니까 |
| `requestMatchers("/api/admin/**").hasRole("ADMIN")` | `/api/admin/` 아래는 **ADMIN만** | `hasRole("ADMIN")` = 권한 `ROLE_ADMIN` 확인 (`ROLE_`은 자동) |
| `anyRequest().authenticated()` | 나머지는 **로그인만 하면** | 권한 상관없이 인증된 사용자 |
| `httpBasic(...)` | **HTTP Basic** 방식으로 로그인 | 아래 6장 |
| `authenticationProvider(...)` | 위에서 만든 인증 담당 사용 | |

* `requestMatchers`는 **위에서부터 차례로** 검사하고 **처음 맞는 규칙**을 적용함 → 구체적인 규칙을 위에, `anyRequest()`는 **맨 마지막**에. `anyRequest()`를 먼저 쓰면 아래 규칙은 영영 안 쓰임.
* `/**` → 그 아래 **모든 하위 경로** (`/api/admin/dashboard`, `/api/admin/users/1` …).
* `HttpMethod.POST`를 붙이면 **POST 요청만** 열림 → 같은 URL이라도 `GET`은 여전히 인증 필요.
* `AbstractHttpConfigurer::disable` → `csrf -> csrf.disable()`을 메소드 참조로 줄여 쓴 것.

> ⚠️ 오늘 작성한 코드에는 아직 고칠 곳이 남아 있음. 실행 전에 맞춰야 함.
> * `SecurityConfig`: `.requestMatchers((HttpMethod.POST, "/api/auth/signup")`처럼 여는 괄호가 **하나 더** 있어서 컴파일 에러 → `.requestMatchers(HttpMethod.POST, "/api/auth/signup")`
> * `SecurityConfig`: `"api/admin/**"`에 앞의 `/`가 빠짐. Security의 경로 규칙은 실제 요청 경로(`/api/admin/...`)와 비교하므로 `"/api/admin/**"`로 써야 함. 그렇지 않으면 관리자 경로가 의도대로 막히지 않을 수 있음
> * `CustomUserDetailService`: `orElseThrow(() - new ...)`에서 람다 화살표의 `>`가 빠짐 → `() -> new UsernameNotFoundException(...)`
> * `UserAccountService`가 `securitybasic/service` 패키지에 있음. 다른 회원 관련 클래스는 `user/` 아래(`user/domain`, `user/dto` …)라 다음 챕터처럼 `user/service`로 모으면 구조가 일정해짐

---

## 6. HTTP Basic 인증

* **HTTP Basic** → 요청할 때마다 헤더에 `아이디:비밀번호`를 **Base64로 인코딩**해서 같이 보내는 가장 단순한 로그인 방식.

```text
Authorization: Basic dXNlcjAxOnBhc3MxMjM0
                     └ "user01:pass1234"를 Base64로 바꾼 값
```

```http
### 인증 없이 내 정보 요청 - 401 예상
GET http://localhost:8080/api/users/me

### HTTP Basic 인증으로 내 정보 요청
GET http://localhost:8080/api/users/me
Authorization: Basic user01 pass1234

### USER 권한으로 관리자 API 요청 - 403 예상
GET http://localhost:8080/api/admin/dashboard
Authorization: Basic user01 pass1234
```

* IntelliJ `.http` 파일에서는 `Basic 아이디 비밀번호`처럼 **띄어 쓰기만** 하면 알아서 Base64로 바꿔서 보내 줌.
* `STATELESS`라서 로그인 상태가 남지 않음 → **매 요청마다** 이 헤더를 보내야 함.

### 요청 한 번의 흐름

```text
GET /api/users/me  +  Authorization: Basic (user01:pass1234)
  │
  ▼ BasicAuthenticationFilter: 헤더에서 아이디·비밀번호 꺼냄
  ▼ DaoAuthenticationProvider
  │    ├ CustomUserDetailService.loadUserByUsername("user01") → DB 조회 → UserDetails
  │    └ passwordEncoder.matches("pass1234", "$2a$10$...")
  │         ├ 실패 → 401
  │         └ 성공 → SecurityContext에 "user01, ROLE_USER" 저장
  ▼ AuthorizationFilter: requestMatchers 규칙 확인
  │    ├ /api/users/me → authenticated() → 통과
  │    └ /api/admin/** → hasRole("ADMIN") → ROLE_USER라서 403
  ▼ Controller
```

| 요청 | 결과 | 이유 |
| :--- | :--- | :--- |
| `POST /api/auth/signup` (인증 없음) | 통과 | `permitAll()` |
| `GET /api/users/me` (인증 없음) | **401** | 누군지 모름 |
| `GET /api/users/me` (user01) | 통과 | `authenticated()` |
| `GET /api/admin/dashboard` (user01) | **403** | 인증은 됐지만 `ROLE_USER` |
| `GET /api/admin/dashboard` (ADMIN 계정) | 통과 | `ROLE_ADMIN` |

> ⚠️ Base64는 **암호화가 아니라 단순 변환**이라 누구나 원래 값으로 되돌릴 수 있음. 그래서 HTTP Basic은 반드시 **HTTPS** 위에서만 써야 하고, 매 요청마다 비밀번호를 보내는 것 자체도 부담이라 실무 API에서는 잘 쓰지 않음 → 다음 챕터에서 **JWT 토큰** 방식으로 바꿈.

---

## 📝 오늘의 정리

1. 인증은 "누구인지 확인"(실패 시 401), 인가는 "권한이 있는지 확인"(실패 시 403)이고, Spring Security는 컨트롤러 앞의 필터 묶음에서 이 둘을 처리함
2. 비밀번호는 `PasswordEncoder`(BCrypt)로 해시해서 저장하고, 로그인 때는 `matches`로 비교함. BCrypt는 솔트 때문에 같은 비밀번호도 매번 다른 해시가 나옴
3. 회원 가입은 중복 검사 → 비밀번호 해시 → 권한은 서버가 `USER`로 고정 → 비밀번호 없는 응답 DTO 반환 순서로 처리함
4. `UserDetailsService.loadUserByUsername`이 아이디로 회원을 찾아 `User.withUsername(...).password(...).roles(...).build()`로 Security 규격의 `UserDetails`를 돌려줌. `roles("USER")`는 `ROLE_USER`가 됨
5. `DaoAuthenticationProvider`가 `UserDetailsService`와 `PasswordEncoder`로 실제 아이디·비밀번호를 검증함
6. `SecurityFilterChain`에서 CSRF 끄기, `STATELESS`, `requestMatchers`(위에서부터 처음 맞는 규칙)·`permitAll`·`hasRole`·`anyRequest().authenticated()`, `httpBasic`을 설정함
7. HTTP Basic은 매 요청마다 `아이디:비밀번호`를 Base64로 보내는 방식이라 HTTPS가 필수이고, 다음은 JWT로 넘어감

---

## 느낀 점

* 401과 403의 차이를 "사원증 확인"과 "서버실 출입 권한"으로 나눠 생각하니 헷갈리지 않게 됐음
* 같은 비밀번호를 `encode`할 때마다 결과가 달라지는데도 `matches`로 비교가 된다는 게 신기했고, 솔트가 해시 문자열 안에 같이 들어 있다는 걸 알고 이해됨
* Security가 내 엔티티를 모르기 때문에 `UserDetailsService`라는 창구로 연결해 준다는 구조가, 11/04에 `JpaRepository` 인터페이스만 만들면 Spring이 채워 주던 것처럼 "**정해진 규격에 맞춰 끼워 넣는**" 방식이라는 점에서 비슷하게 느껴졌음
* 가입 요청에서 권한을 받지 않고 서버가 정한다는 것처럼, 보안은 기능보다 "클라이언트가 보낸 값을 어디까지 믿을지"를 먼저 생각해야 한다는 걸 느낌
* 오늘 코드에 괄호·화살표 오타가 남아 있어서, 다음 시간 컨트롤러를 붙이기 전에 먼저 컴파일부터 확인해야겠음
