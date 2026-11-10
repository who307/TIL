# Spring Security 2: 보호된 API·MockMvc 보안 테스트 · JWT 개념 · JwtEncoder/JwtDecoder · AuthenticationManager로 로그인하고 Access Token 발급 (TIL)

> 실습 환경: Spring Boot 4 + `spring-boot-starter-security`·`spring-boot-starter-security-oauth2-resource-server`(JWT 발급·검증용 Nimbus 포함)·`data-jpa`·`validation`·`webmvc`, MySQL `securitydb`, 테스트는 H2 인메모리 DB
>
> 오전엔 11/06에 만들던 `chap01-security-basic`의 컨트롤러와 보안 흐름 테스트를 끝냈고, 오후엔 `chap02-jwt-authentication`에서 **로그인하면 JWT를 발급**하는 부분까지 작성함. 받은 토큰으로 API를 호출하는 **검증 쪽 연결**(`oauth2ResourceServer`)은 다음 시간에 진행

## 0. 오늘의 흐름

| 단계 | 내용 |
| :--- | :--- |
| chap01 마무리 | `/api/users/me`(내 정보), `/api/admin/dashboard`(관리자) 컨트롤러, `Authentication`으로 로그인 사용자 꺼내기 |
| 보안 테스트 | `MockMvc` + `httpBasic(...)`으로 가입·401·200·403 흐름 검증 |
| JWT 개념 | 세션 vs 토큰, JWT 구조(Header·Payload·Signature), Claim |
| JWT 도구 | `SecretKey`, `JwtEncoder`(발급), `JwtDecoder`(검증), `JwtAuthenticationConverter` |
| 로그인 | `AuthenticationManager`로 아이디·비밀번호 확인 → `JwtService`로 Access Token 발급 → `TokenResponse` |

---

## 1. chap01 마무리: 보호된 API 만들기

### 로그인한 사용자 정보 꺼내기: `Authentication`

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserAccountService userAccountService;

    public UserController(UserAccountService userAccountService) {
        this.userAccountService = userAccountService;
    }

    @GetMapping("/me")
    public ResponseEntity<UserResponse> findMyProfile(Authentication authentication) {
        return ResponseEntity.ok(userAccountService.findByUserId(authentication.getName()));
    }
}
```

* 컨트롤러 메소드 파라미터에 **`Authentication`** 을 적으면, Security 필터가 인증을 끝내고 만들어 둔 **"인증된 사용자 정보"** 를 Spring이 넣어 줌.
* `authentication.getName()` → 로그인한 **아이디**(`UserDetails`의 username). 
* 그래서 `/me`처럼 "**내** 정보"를 조회할 때 URL에 아이디를 받을 필요가 없음. 클라이언트가 `/api/users/user02`처럼 남의 아이디를 넣어 조회하는 일을 원천 차단.

```java
// UserAccountService
public UserResponse findByUserId(String userId) {
    UserAccount user = userAccountRepository.findByUserId(userId)
            .orElseThrow(() -> new IllegalArgumentException("회원을 찾을 수 없습니다."));
    return UserResponse.from(user);
}
```

### 관리자 API

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

    @GetMapping("/dashboard")
    public ResponseEntity<Map<String, String>> dashboard() {
        return ResponseEntity.ok(Map.of("message", "관리자 페이지입니다."));
    }
}
```

* 컨트롤러 안에는 **권한 검사 코드가 하나도 없음**. `SecurityConfig`의 `.requestMatchers("/api/admin/**").hasRole("ADMIN")`이 컨트롤러 도착 전에 이미 걸러 줌.

### 지난 시간 오타 정리

11/06 느낀 점에 적었던 컴파일 오류를 먼저 고침.

| 위치 | 틀린 코드 | 고친 코드 |
| :--- | :--- | :--- |
| `CustomUserDetailService` | `() - new UsernameNotFoundException(...)` | `() -> new ...` (람다 화살표) |
| `SecurityConfig` | `.requestMatchers((HttpMethod.POST, ...)` | 괄호 하나 제거 |
| `SecurityConfig` | `"api/admin/**"` | `"/api/admin/**"` (앞 `/` 빠지면 매칭 안 됨) |
| `AuthController` | `package ...securitybasic.user;` | `...user.controller;` (폴더와 패키지 일치) |

---

## 2. MockMvc로 보안 흐름 테스트

서버를 띄우고 `.http`로 하나씩 눌러 보는 대신, **401·403·200이 의도대로 나오는지** 테스트 코드로 고정함.

```java
@SpringBootTest
@AutoConfigureMockMvc          // Boot 4: org.springframework.boot.webmvc.test.autoconfigure 패키지
@ActiveProfiles("test")        // application-test.yml (H2) 사용
@Transactional                 // 테스트마다 롤백
class SecurityFlowTests {

    @Autowired private MockMvc mockMvc;
    @Autowired private UserAccountRepository userAccountRepository;
    @Autowired private PasswordEncoder passwordEncoder;

    @Test
    @DisplayName("인증 정보 없이 보호된 API를 요청하면 401을 반환한다")
    void authenticationRequiredTest() throws Exception {
        mockMvc.perform(get("/api/users/me"))
                .andExpect(status().isUnauthorized());
    }

    @Test
    @DisplayName("USER 권한으로 관리자 API를 요청하면 403을 반환한다")
    void authorizationDeniedTest() throws Exception {
        UserAccount user = saveUser(UserRole.USER);

        mockMvc.perform(get("/api/admin/dashboard")
                        .with(httpBasic(user.getUserId(), "pass1234")))
                .andExpect(status().isForbidden());
    }

    private UserAccount saveUser(UserRole role) {
        String suffix = UUID.randomUUID().toString().substring(0, 8);
        return userAccountRepository.save(new UserAccount(
                "user-" + suffix,
                passwordEncoder.encode("pass1234"),   // DB엔 해시로 저장
                "테스트 사용자",
                suffix + "@test.com",
                role));
    }
}
```

| 테스트 | 요청 | 기대 결과 |
| :--- | :--- | :--- |
| 회원 가입 | `POST /api/auth/signup` | `201`, 응답에 `password` 없음, DB엔 해시 저장(`matches` true) |
| 인증 없음 | `GET /api/users/me` | `401` |
| 정상 로그인 | `GET /api/users/me` + `httpBasic` | `200`, `userId` 일치 |
| 권한 부족 | `USER`로 `GET /api/admin/dashboard` | `403` |
| 관리자 | `ADMIN`으로 `GET /api/admin/dashboard` | `200`, `"관리자 페이지입니다."` |

* **`.with(httpBasic(아이디, 비밀번호))`** → `spring-security-test`가 주는 도우미. `Authorization: Basic ...` 헤더를 대신 만들어 붙여 줌.
* `jsonPath("$.password").doesNotExist()` → 응답 JSON에 비밀번호 필드가 **아예 없는지** 확인. 응답 DTO 분리가 제대로 됐는지 테스트로 보장.
* 아이디에 `UUID` 앞 8자리를 붙여서 **테스트끼리 아이디가 겹치지 않게** 함 (중복 가입 409 방지).
* 회원 가입 테스트의 JSON 본문은 **텍스트 블록 + `formatted()`** 로 만듦 (11/05 Native Query에서 쓴 `"""` 문법).

---

## 3. JWT 개념

### 세션 vs 토큰

| | 세션 방식 | 토큰(JWT) 방식 |
| :--- | :--- | :--- |
| 로그인 정보 저장 | **서버 메모리**에 세션 저장, 클라이언트는 세션 ID(쿠키)만 가짐 | 서버는 저장 안 함, **토큰 안에 정보가 들어 있음** |
| 요청할 때 | 쿠키의 세션 ID로 서버가 세션을 찾음 | `Authorization: Bearer 토큰`을 보내면 서버가 **서명만 확인** |
| 서버 여러 대 | 세션 공유가 필요해서 번거로움 | 같은 키만 있으면 어느 서버든 검증 가능 |
| 설정 | | `SessionCreationPolicy.STATELESS` (세션 안 만듦) |

* **한 줄 비유**: 세션은 "**보관소 번호표**"(물건은 서버가 들고 있음), JWT는 "**도장 찍힌 출입증**"(정보가 출입증에 적혀 있고, 서버는 도장이 진짜인지만 봄).
* HTTP Basic(11/06)은 매 요청마다 **비밀번호**를 보냈지만, JWT는 **로그인 때 한 번만** 비밀번호를 보내고 그 뒤엔 토큰만 보냄.

### JWT 구조

```text
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiJ1c2VyMDEiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXX0 . 서명값
└──── Header ──────┘   └──────────────────── Payload(Claims) ───────────────────┘   └ Signature ┘
```

| 부분 | 들어 있는 것 | 오늘 코드 |
| :--- | :--- | :--- |
| **Header** | 서명 알고리즘 | `HS256` |
| **Payload** | **Claim**(토큰에 담는 정보 한 조각씩) | `iss`, `iat`, `exp`, `sub`, `authorities` |
| **Signature** | Header + Payload를 **비밀 키로 서명**한 값 | `SecretKey`로 HMAC-SHA256 |

* Header·Payload는 **Base64URL 인코딩일 뿐** → jwt.io에 붙여 넣으면 누구나 내용을 읽을 수 있음. **비밀번호 같은 민감 정보는 절대 넣지 않음**.
* 대신 내용을 한 글자라도 바꾸면 **서명이 맞지 않게** 되어 서버가 거부함 → JWT는 "**숨기는**" 게 아니라 "**위조를 막는**" 기술.

| Claim | 뜻 |
| :--- | :--- |
| `iss` (issuer) | 발급자 |
| `iat` (issued at) | 발급 시각 |
| `exp` (expires at) | 만료 시각 → 지나면 검증 실패 |
| `sub` (subject) | 토큰 주인 (여기선 아이디) |
| `authorities` | 직접 만든 Claim, 권한 목록 (`["ROLE_USER"]`) |

---

## 4. JWT 도구 등록: `JwtInfrastructureConfig`

### 설정값

```yaml
security:
  jwt:
    secret: ${JWT_SECRET:c3ByaW5nLXNlY3VyaXR5LWp3dC1jbGFzcy1zZWNyZXQta2V5LTEyMzQ=}
    expiration-seconds: 3600
```

* `${JWT_SECRET:기본값}` → 환경 변수 `JWT_SECRET`이 있으면 그 값, 없으면 `:` 뒤의 기본값 (11/05 환경 변수 설정과 같은 문법). 실무에선 키를 코드·Git에 남기지 않고 환경 변수로만 줌.
* 기본값은 `spring-security-jwt-class-secret-key-1234`를 Base64로 바꾼 것 (41바이트). **HS256 키는 최소 256비트(32바이트)** 이상이어야 함.
* `expiration-seconds: 3600` → Access Token 유효 시간 1시간.

### Bean 4개

```java
@Configuration
public class JwtInfrastructureConfig {

    // 발급과 검증이 같은 비밀 키를 사용하도록 설정값을 SecretKey로 바꾼다
    @Bean
    public SecretKey jwtSecretKey(@Value("${security.jwt.secret}") String secret) {
        byte[] keyBytes = Base64.getDecoder().decode(secret);
        return new SecretKeySpec(keyBytes, "HmacSHA256");
    }

    // 로그인 성공 후 Claim에 서명해 JWT 문자열을 만드는 도구
    @Bean
    public JwtEncoder jwtEncoder(SecretKey secretKey) {
        return new NimbusJwtEncoder(new ImmutableSecret<SecurityContext>(secretKey));
    }

    // Bearer 요청으로 들어온 JWT의 서명과 만료 등을 확인하는 도구
    @Bean
    public JwtDecoder jwtDecoder(SecretKey secretKey) {
        return NimbusJwtDecoder.withSecretKey(secretKey)
                .macAlgorithm(MacAlgorithm.HS256)
                .build();
    }

    // 토큰의 authorities Claim을 Security 권한으로 바꾸는 규칙
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthoritiesClaimName("authorities");
        authoritiesConverter.setAuthorityPrefix("");

        JwtAuthenticationConverter authenticationConverter = new JwtAuthenticationConverter();
        authenticationConverter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return authenticationConverter;
    }
}
```

| Bean | 언제 쓰임 | 역할 |
| :--- | :--- | :--- |
| `SecretKey` | 발급·검증 둘 다 | **같은 키 하나**로 서명하고 확인 (HS256 = 대칭키 방식) |
| `JwtEncoder` | 로그인 성공 시 | Claim + Header → 서명된 JWT 문자열 |
| `JwtDecoder` | API 요청 시 | 서명이 맞는지, 만료됐는지 확인 → 실패하면 401 |
| `JwtAuthenticationConverter` | API 요청 시 | 검증된 JWT → `Authentication` (권한 포함) |

* **`setAuthoritiesClaimName("authorities")`** → 기본값은 `scope`/`scp` Claim을 읽으므로, 우리가 넣은 `authorities` Claim을 읽으라고 알려 줌.
* **`setAuthorityPrefix("")`** → 기본으로는 권한 앞에 `SCOPE_`를 붙여 `SCOPE_ROLE_USER`가 됨. 토큰에 이미 `ROLE_USER`가 들어 있으니 접두사를 비워야 `hasRole("ADMIN")`(= `ROLE_ADMIN`)과 맞음.

---

## 5. 로그인하고 Access Token 발급

### 전체 흐름

```text
POST /api/auth/login  {"userId":"user01","password":"pass1234"}
  │
  ▼ AuthController.login (@Valid LoginRequest)
  ▼ AuthService.login
  │   ① UsernamePasswordAuthenticationToken.unauthenticated(아이디, 비밀번호)
  │   ② authenticationManager.authenticate(...)
  │        └ ProviderManager → DaoAuthenticationProvider
  │             ├ CustomUserDetailsService.loadUserByUsername → DB 조회
  │             └ PasswordEncoder.matches(입력값, 저장된 해시)
  │        실패 → AuthenticationException (토큰 발급 안 됨)
  │        성공 → 인증된 Authentication (아이디 + ROLE_USER)
  │   ③ jwtService.createAccessToken(authentication)
  ▼
200 {"tokenType":"Bearer","accessToken":"eyJ...","expiresIn":3600}
```

* 11/06엔 `BasicAuthenticationFilter`가 **필터 안에서** 인증을 했다면, 오늘은 로그인 API에서 **내가 직접** `AuthenticationManager`를 불러 인증을 시킴. 검증 로직(`DaoAuthenticationProvider`)은 그대로 재사용.

### AuthenticationManager Bean

```java
@Bean
public AuthenticationManager authenticationManager(AuthenticationProvider authenticationProvider) {
    return new ProviderManager(authenticationProvider);
}
```

* `AuthenticationManager` = 인증 요청을 받는 **창구**, `ProviderManager`는 그 구현체로 등록된 **`AuthenticationProvider`(실제 검사 담당자)** 들에게 일을 넘김.
* 11/06에 만든 `DaoAuthenticationProvider`(회원 조회 + 비밀번호 비교)를 그대로 넣어서 서비스에서 주입받아 쓸 수 있게 Bean으로 꺼내 둠.

### 요청·응답 DTO

```java
public record LoginRequest(
        @NotBlank String userId,
        @NotBlank String password
) {}

public record TokenResponse(
        String tokenType,    // "Bearer"
        String accessToken,
        long expiresIn       // 초 단위
) {}
```

* 처음에 `LoginRequest`를 `auth` 폴더에 두고 패키지를 `auth.auth.dto`로 잘못 적었다가, `auth/dto` 폴더로 옮기고 패키지를 `...auth.dto`로 맞춤 (`TokenResponse`도 같이 수정).

### AuthService

```java
@Service
public class AuthService {

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;

    public AuthService(AuthenticationManager authenticationManager, JwtService jwtService) {
        this.authenticationManager = authenticationManager;
        this.jwtService = jwtService;
    }

    public TokenResponse login(LoginRequest request) {
        // 아이디·비밀번호를 Provider에 전달, 실패하면 예외가 나서 아래로 내려가지 않음
        Authentication authentication = authenticationManager.authenticate(
                UsernamePasswordAuthenticationToken.unauthenticated(
                        request.userId(), request.password()));

        // 검증을 통과한 Authentication의 아이디와 권한으로 토큰 생성
        String accessToken = jwtService.createAccessToken(authentication);
        return new TokenResponse("Bearer", accessToken, jwtService.getExpirationSeconds());
    }
}
```

* **`UsernamePasswordAuthenticationToken.unauthenticated(...)`** → "아직 확인 안 된" 아이디·비밀번호 묶음. 이걸 `authenticate`에 넣으면 확인을 마친 **`authenticated` 상태의 `Authentication`** 이 돌아옴.
* 토큰 생성은 반드시 `authenticate` **다음 줄**에 있음 → 비밀번호가 틀리면 예외로 빠져나가서 토큰이 절대 만들어지지 않음.

### JwtService: Claim 만들고 서명

```java
@Service
public class JwtService {

    private final JwtEncoder jwtEncoder;
    private final long expirationSeconds;

    public JwtService(JwtEncoder jwtEncoder,
                      @Value("${security.jwt.expiration-seconds}") long expirationSeconds) {
        this.jwtEncoder = jwtEncoder;
        this.expirationSeconds = expirationSeconds;
    }

    public String createAccessToken(Authentication authentication) {
        Instant issuedAt = Instant.now();
        List<String> authorities = authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)     // "ROLE_USER"
                .toList();

        // Payload
        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("chap02-jwt-authentication")
                .issuedAt(issuedAt)
                .expiresAt(issuedAt.plusSeconds(expirationSeconds))
                .subject(authentication.getName())       // 토큰 주인 = 아이디
                .claim("authorities", authorities)
                .build();

        // Header: 서명 방식
        JwsHeader header = JwsHeader.with(MacAlgorithm.HS256).build();

        return jwtEncoder.encode(JwtEncoderParameters.from(header, claims))
                .getTokenValue();
    }

    public long getExpirationSeconds() {
        return expirationSeconds;
    }
}
```

* `JwsHeader.with(MacAlgorithm.HS256)` → `JwtDecoder`의 `.macAlgorithm(MacAlgorithm.HS256)`과 **알고리즘을 맞춰야** 검증이 됨.
* 권한은 `getAuthorities()`를 문자열 리스트로 바꿔 Claim에 넣음 → 나중에 4장의 `JwtAuthenticationConverter`가 이 Claim을 다시 권한으로 되돌림 (**넣는 쪽과 꺼내는 쪽의 Claim 이름이 같아야 함**).

### 컨트롤러

```java
@PostMapping("/login")
public ResponseEntity<TokenResponse> login(@Valid @RequestBody LoginRequest request) {
    return ResponseEntity.ok(authService.login(request));
}
```

### 로그인 실패 처리

```java
@ExceptionHandler(AuthenticationException.class)
public ResponseEntity<Map<String, String>> handleAuthentication(AuthenticationException exception) {
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(Map.of("message", "아이디 또는 비밀번호가 올바르지 않습니다"));
}
```

* 아이디가 없는 경우(`UsernameNotFoundException`)와 비밀번호가 틀린 경우(`BadCredentialsException`)를 **같은 메시지**로 응답 → "이 아이디는 존재한다"는 정보를 공격자에게 흘리지 않기 위함. (실제로 `DaoAuthenticationProvider`도 기본으로 아이디 없음을 `BadCredentialsException`으로 감춰서 던짐)

> ⚠️ 지금 코드는 `AuthenticationException`을 **`javax.security.sasl.AuthenticationException`** 으로 import하고 있음. IntelliJ 자동 import가 이름만 같은 다른 클래스를 골라 준 것이고, Spring Security가 던지는 건 **`org.springframework.security.core.AuthenticationException`** 이라 이 핸들러에 걸리지 않음 → 로그인 실패가 의도한 401 메시지로 응답되지 않음. import를 바꿔야 함.

---

## 6. 아직 남은 것 (다음 시간)

오늘 `chap02`의 `SecurityConfig`는 아직 11/06 설정 그대로(`httpBasic`)라 두 가지가 비어 있음.

| 문제 | 증상 | 해결 (chap03 완성본 기준) |
| :--- | :--- | :--- |
| `/api/auth/login`이 `permitAll`에 없음 | 로그인하려는데 "먼저 로그인하라"며 **401** | `.requestMatchers(HttpMethod.POST, "/api/auth/signup", "/api/auth/login").permitAll()` |
| JWT 검증 필터가 연결 안 됨 | `Authorization: Bearer 토큰`을 보내도 인증 안 됨 | `.httpBasic(...)` 대신 `.oauth2ResourceServer(rs -> rs.jwt(jwt -> jwt.jwtAuthenticationConverter(converter)))` |

* `JwtDecoder`·`JwtAuthenticationConverter` Bean은 이미 만들어 뒀으니, `oauth2ResourceServer`만 켜면 `BearerTokenAuthenticationFilter`가 헤더의 토큰을 꺼내 → `JwtDecoder`로 검증 → `JwtAuthenticationConverter`로 `Authentication`을 만들어 줌. 그러면 `/me`의 `authentication.getName()`은 토큰의 `sub`가 됨.

```http
### 로그인 - 응답의 accessToken을 전역 변수에 저장
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{ "userId": "user01", "password": "pass1234" }

> {% client.global.set("accessToken", response.body.accessToken); %}

### Bearer 토큰으로 내 정보 요청
GET http://localhost:8080/api/users/me
Authorization: Bearer {{accessToken}}
```

* IntelliJ `.http`의 **응답 핸들러 스크립트**(`> {% ... %}`)로 로그인 응답의 토큰을 `accessToken` 변수에 저장해 두면, 아래 요청에서 `{{accessToken}}`으로 복사 없이 바로 씀.

---

## 📝 오늘의 정리

1. 컨트롤러 파라미터의 `Authentication`으로 로그인한 사용자를 꺼내고, `getName()`(아이디)으로 "내 정보"를 조회함. 권한 검사는 컨트롤러가 아니라 `SecurityConfig`가 담당함
2. `MockMvc` + `.with(httpBasic(...))`로 401(인증 없음)·403(권한 부족)·200 흐름과 "응답에 비밀번호 없음"을 테스트로 고정함
3. JWT는 Header·Payload(Claim)·Signature로 이루어지고, 내용은 누구나 읽을 수 있지만 서명 덕분에 위조할 수 없음. 그래서 민감 정보는 넣지 않음
4. 같은 `SecretKey`로 `JwtEncoder`는 서명(발급), `JwtDecoder`는 서명·만료 확인(검증)을 함. `JwtAuthenticationConverter`는 `authorities` Claim을 접두사 없이 권한으로 바꾸도록 설정함
5. 로그인은 `AuthenticationManager`(`ProviderManager` → `DaoAuthenticationProvider`)로 아이디·비밀번호를 확인한 **뒤에만** `JwtService`가 Access Token을 만들고 `TokenResponse("Bearer", 토큰, 만료초)`로 응답함
6. 로그인 실패는 아이디 없음/비밀번호 틀림을 구분하지 않고 같은 401 메시지로 응답함 (import는 Spring Security의 `AuthenticationException`이어야 함)
7. 다음 시간: `/api/auth/login` 열어 두기 + `oauth2ResourceServer().jwt()`로 Bearer 토큰 검증 연결

---

## 느낀 점

* HTTP Basic은 요청마다 비밀번호를 보냈는데, JWT는 로그인 때 한 번 확인하고 그 결과를 "서명된 출입증"으로 바꿔 준다는 점에서 왜 API 서버에서 토큰을 쓰는지 이해됨
* JWT 내용이 Base64라서 누구나 읽을 수 있다는 게 처음엔 이상했는데, 목적이 숨기는 게 아니라 "바뀌지 않았음을 증명"하는 거라고 생각하니 납득됨
* 발급 쪽(`authorities` Claim, `ROLE_` 포함)과 검증 쪽(`setAuthoritiesClaimName`, `setAuthorityPrefix("")`)이 서로 짝이 맞아야 한다는 점, `HS256`도 양쪽이 같아야 한다는 점에서 "보내는 쪽과 받는 쪽의 약속"이 중요하다고 느낌
* 로그인 API를 만들어 놓고도 `permitAll`에 안 넣으면 로그인 자체가 막힌다는 게 재밌었음. 보안 설정은 "잠그는 것"만큼 "**무엇을 열어 둘지**"도 빠짐없이 챙겨야 함
* 자동 import가 `javax.security.sasl.AuthenticationException`을 골라 준 것처럼, 이름이 같은 클래스는 import 줄까지 꼭 확인해야겠음
